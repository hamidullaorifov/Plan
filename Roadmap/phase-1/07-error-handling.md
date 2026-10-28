# 7. Error Handling: Exceptions vs Results

> Phase 1 · Week 9 · [← Phase index](README.md)

**Why this matters:** How a codebase handles failure decides whether incidents are a 5-minute fix ("the log says exactly what failed") or a 5-hour hunt ("an exception was swallowed somewhere"). Middle developers are expected to choose deliberately between **exceptions** and **result types**, and to put error handling at the right layer.

---

## 7.1 Two kinds of failure

| | **Expected (domain) outcomes** | **Unexpected (exceptional) failures** |
|---|---|---|
| Examples | validation failed, seat already taken, insufficient funds, not found, event not on sale | database unreachable, a null where it's impossible, a bug, disk full, a timeout to a dependency |
| Who handles it | the caller: show a message, return 400/404/409, offer alternatives | usually **nobody locally**: a global handler logs it, returns 500 and alerts |
| How often | regularly, as part of normal business flow | rarely |
| Best mechanism | **a return value**: `Result`, a union, `TryX(out)` or `null` for "not found" | **an exception** |

> **Rule:** use exceptions for things that are *exceptional*. If a failure is part of the business rules, it belongs in the method signature.

---

## 7.2 Exceptions done right

### Throwing

```csharp
public void ChangeCapacity(int capacity)
{
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(capacity);           // .NET 8 guard helpers
    ObjectDisposedException.ThrowIf(_disposed, this);
    if (Status == VenueStatus.Archived)
        throw new InvalidOperationException("An archived venue can't be modified.");
    // ...
}
```

| Exception | Meaning |
|---|---|
| `ArgumentException` / `ArgumentNullException` / `ArgumentOutOfRangeException` | the **caller passed bad input**: a contract violation and a caller bug |
| `InvalidOperationException` | the object is in the **wrong state** for this call |
| `NotSupportedException` | the operation is never supported by this type |
| `ObjectDisposedException` | the object was used after `Dispose` |
| `UnreachableException` (.NET 7) | "this code can't run". Use it in impossible `switch` defaults |
| `TimeoutException`, `OperationCanceledException` | time and cancellation |
| A custom `XyzException` | **only** when callers need to catch *this specific* condition and react differently |

### Catching

```csharp
try
{
    await _gateway.ChargeAsync(order, ct);
}
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.TooManyRequests)  // exception filter
{
    _logger.LogWarning(ex, "Payment provider throttled order {OrderId}; scheduling retry", order.Id);
    await _retryQueue.EnqueueAsync(order.Id, ct);
}
```

- **Catch only what you can handle** (retry, fall back, translate). Otherwise, let the exception bubble up.
- **Exception filters (`when`)** decide *before* the stack unwinds, which preserves the debugging context and keeps the code clean.
- **Never swallow exceptions** (`catch { }`). If you truly must ignore one, catch the specific type and add a comment explaining why.
- `catch (Exception)` belongs only at **boundaries**: the global handler, the top of a background-worker loop, or a message handler.

### Rethrowing

```csharp
catch (SqlException ex)
{
    _logger.LogError(ex, "…");
    throw;                                  // ✅ keeps the original stack trace
    // throw ex;                            // ❌ resets the stack trace to this line (analyzer CA2200)
    // throw new RepositoryException("Saving order failed", ex); // ✅ wrap with context: keep it as InnerException
}
```

### Cost

Throwing is *orders of magnitude* more expensive than returning a value: it captures a stack trace and unwinds frames. A `try` block that doesn't throw costs essentially nothing. Exception handling got significantly faster in .NET 9, but don't use exceptions for control flow in hot paths, such as validating every item in a loop of 100,000.

### Exceptions in async code

| Situation | Behavior |
|---|---|
| `await` on a faulted task | rethrows the **first** inner exception |
| `await Task.WhenAll(...)` with several failures | throws the first; **all** of them are in `whenAllTask.Exception!.InnerExceptions` |
| `async void` throws | crashes the process (see [guide 3](03-async-and-concurrency.md#38-async-void-only-for-event-handlers)) |
| A task faults and nobody awaits it | the exception is "unobserved". It's raised on `TaskScheduler.UnobservedTaskException` at a later GC, and is effectively lost |

---

## 7.3 The Result pattern

### A minimal, production-ready implementation (C# 14 / `net10.0`)

```csharp
namespace TicketHub.Domain.Common;

public sealed record Error(string Code, string Message);

public class Result
{
    protected Result(bool isSuccess, Error? error)
    {
        if (isSuccess == error is not null)
            throw new ArgumentException("A success has no error; a failure must have one.", nameof(error));
        IsSuccess = isSuccess;
        Error = error;
    }

    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public Error? Error { get; }

    public static Result Success() => new(true, null);
    public static Result Failure(Error error) => new(false, error);

    public static implicit operator Result(Error error) => Failure(error);
}

public sealed class Result<T> : Result
{
    private readonly T? _value;

    private Result(T value) : base(true, null) => _value = value;
    private Result(Error error) : base(false, error) { }

    public T Value => IsSuccess
        ? _value!
        : throw new InvalidOperationException($"Cannot read Value of a failed result ({Error!.Code}).");

    public static Result<T> Success(T value) => new(value);
    public static new Result<T> Failure(Error error) => new(error);

    public static implicit operator Result<T>(T value) => Success(value);
    public static implicit operator Result<T>(Error error) => Failure(error);

    public TOut Match<TOut>(Func<T, TOut> onSuccess, Func<Error, TOut> onFailure) =>
        IsSuccess ? onSuccess(_value!) : onFailure(Error!);
}
```

> The implicit conversions make the call sites clean (`return hold;` / `return EventErrors.NotOnSale(Id);`). Note that C# doesn't apply user-defined conversions when `T` is an interface type. In that case, call `Result<T>.Success(...)` explicitly.

### An error catalog: one place per aggregate

```csharp
public static class EventErrors
{
    public static Error NotDraft(Guid id) =>
        new("Event.NotDraft", $"Event {id} can only be published from Draft status.");

    public static Error NotOnSale(Guid id) =>
        new("Event.NotOnSale", $"Event {id} is not on sale.");

    public static Error InvalidSeatCount(int max) =>
        new("Event.InvalidSeatCount", $"You can hold between 1 and {max} distinct seats.");

    public static Error SeatsUnavailable(IEnumerable<SeatId> seats) =>
        new("Event.SeatsUnavailable", $"Seats unavailable: {string.Join(", ", seats)}.");
}
```

- **Stable codes** (`Event.SeatsUnavailable`) are what front ends, tests and monitoring rely on. Never parse error messages.
- Error messages must **not leak internals** such as SQL, stack traces or other users' data.

### Using it

```csharp
// Domain
public Result<SeatHold> HoldSeats(IReadOnlyCollection<SeatId> seats, Guid customerId, TimeProvider clock)
{
    if (Status != EventStatus.Published) return EventErrors.NotOnSale(Id);
    if (seats.Count is 0 or > MaxSeatsPerHold) return EventErrors.InvalidSeatCount(MaxSeatsPerHold);
    // ...
    return hold;
}

// API layer (Phase 2): map error codes to HTTP status codes and ProblemDetails
IResult ToHttp(Result<SeatHold> result) => result.Match(
    hold  => Results.Ok(hold),
    error => error.Code switch
    {
        "Event.NotOnSale"        => Results.Problem(error.Message, statusCode: StatusCodes.Status409Conflict),
        "Event.SeatsUnavailable" => Results.Problem(error.Message, statusCode: StatusCodes.Status409Conflict),
        "Event.InvalidSeatCount" => Results.Problem(error.Message, statusCode: StatusCodes.Status400BadRequest),
        _                        => Results.Problem(error.Message, statusCode: StatusCodes.Status400BadRequest),
    });
```

### The C# 15 alternative: unions (when you target `net11.0`)

```csharp
public sealed record SeatsHeld(SeatHold Hold);
public sealed record SeatsUnavailable(IReadOnlyList<SeatId> Seats);
public sealed record EventNotOnSale(Guid EventId);
public sealed record InvalidSeatCount(int Max);

public union HoldSeatsResult(SeatsHeld, SeatsUnavailable, EventNotOnSale, InvalidSeatCount);
```

The compiler checks that every caller's `switch` handles all four outcomes, which is stronger than string error codes. The trade-off is one union type per operation. Libraries like **ErrorOr**, **FluentResults** and **OneOf** exist too. Know their names, but a small in-house `Result` (or unions) is usually enough.

---

## 7.4 Decision guide

| Situation | Use |
|---|---|
| Invalid argument from a *programmer* (null, negative count) | **throw** `ArgumentException` (a caller bug; fail fast) |
| Invalid input from a *user* or API client | **validation** → `Result` / ProblemDetails 400 |
| A business rule says no (seat taken, not on sale) | **`Result`** or a union |
| "Not found" for a lookup | `null` (`Task<Order?>`) or `Result` with a NotFound error |
| A parse that may fail, in a hot path | the **Try-pattern**: `bool TryParse(string s, [NotNullWhen(true)] out T? value)` |
| Infrastructure failure (database down, timeout) | **exception**, caught by the global handler (500) or a resilience policy (retry) |
| An "impossible" state | **throw** `UnreachableException` / `InvalidOperationException` |

**Don't overdo `Result`.** Wrapping every infrastructure call in `Result` turns your code into "if failure return failure" boilerplate and hides real bugs. Use results for *domain* outcomes and exceptions for *exceptional* failures.

---

## 7.5 Where to handle errors in a layered app

```
API / endpoint layer   → maps Result errors to 4xx ProblemDetails; global IExceptionHandler → 500 + traceId
Application layer      → orchestrates; returns Result; lets infrastructure exceptions bubble
Domain layer           → enforces invariants; returns Result for business rules; throws for programmer errors
Infrastructure layer   → translates provider exceptions only when it adds meaning (e.g., unique-constraint → DuplicateError)
```

- **Log an exception once**, where it's finally handled (usually the global handler), with the full exception object.
- **Don't leak details:** a 500 response contains a generic message and a **trace ID**, never the stack trace (an OWASP finding).
- You'll implement `IExceptionHandler` and `ProblemDetails` in Phase 2.

---

## Interview questions

<details><summary><b>1. When do you throw an exception and when do you return a result?</b></summary>

Throw for exceptional or unexpected failures and for programmer errors (contract violations). Return a result for expected domain outcomes that the caller must handle, such as validation, business rule violations or not found. That puts them in the signature and avoids exceptions as control flow.
</details>

<details><summary><b>2. <code>throw;</code> vs <code>throw ex;</code>?</b></summary>

`throw;` rethrows and preserves the original stack trace. `throw ex;` resets the stack trace to the current line, so the origin is lost. To add context, wrap the exception and pass the original as `innerException`.
</details>

<details><summary><b>3. What are exception filters, and why use them?</b></summary>

`catch (X ex) when (condition)`. The filter is evaluated before the stack unwinds, so if it's false, the exception continues as if it had never been caught. That preserves the debugging context. It's useful for status-code-specific handling.
</details>

<details><summary><b>4. How do exceptions behave with Task.WhenAll?</b></summary>

`await` rethrows the first exception. All of them are available in the returned task's `Exception.InnerExceptions` (an `AggregateException`).
</details>

<details><summary><b>5. Where should exceptions be logged in a layered application?</b></summary>

Once, at the point where they're handled, typically a global exception handler or a worker loop boundary. Logging at every layer creates duplicate noise.
</details>

<details><summary><b>6. What are the downsides of the Result pattern?</b></summary>

Boilerplate and propagation code, the risk of ignoring a returned result, and over-use for infrastructure failures that should stay exceptions. With C# 15 unions, exhaustiveness is enforced by the compiler.
</details>

---

## Exercises

1. **Audit:** find three `catch (Exception)` blocks at work. For each, decide whether it should catch something narrower, rethrow, or move to a boundary.
2. **Implement:** add `Result`, `Result<T>` and `Error` to TicketHub.Domain, plus `EventErrors`, and write tests for `Match` and the implicit conversions.
3. **Union version:** in the `net11.0` playground, rewrite `HoldSeats` to return a union, and compare the call sites with the `Result` version.
4. **Perf intuition:** benchmark validating 100,000 items by throwing on invalid ones versus returning `Result.Failure`, with 10% of the items invalid.
5. **`WhenAll`:** write a test proving that only the first exception is thrown and that all of them can be read from the task.

## Self-check

- [ ] I can classify a failure as expected or exceptional and choose the right mechanism.
- [ ] I use guard helpers, filters and `throw;` correctly, and I never swallow exceptions.
- [ ] I have a `Result` type and an error catalog in my capstone.
- [ ] I know where each layer handles, translates and logs errors.

## Further reading

- Microsoft Learn: *Best practices for exceptions*, *Exception filters*, *Handle errors in ASP.NET Core*.
- Vladimir Khorikov, *"Functional C#: Handling failures, input errors"* (blog).
- Microsoft Learn: *Union types* (C# 15).
