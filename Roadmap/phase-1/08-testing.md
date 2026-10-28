# 8. Testing: xUnit v3, Test Doubles and Good Tests

> Phase 1 · Week 10 · [← Phase index](README.md)

**Why this matters:** AI now writes a lot of code, and the #1 AI problem is *"plausible but wrong code"*. **Tests are your verification tool.** A Middle developer owns the test strategy for their features, writes tests that catch real regressions, and avoids tests that break on every refactor.

---

## 8.1 What makes a test *good*: the four pillars

(From Vladimir Khorikov, *Unit Testing Principles, Practices, and Patterns*.)

| Pillar | Question | How to achieve it |
|---|---|---|
| **Protection against regressions** | Does it fail when behavior breaks? | test meaningful code paths, edge cases and invariants |
| **Resistance to refactoring** | Does it stay green when I restructure code without changing behavior? | test **observable behavior** (outputs, state, side effects visible to callers), not implementation details |
| **Fast feedback** | Does it run in milliseconds? | keep the domain free of I/O; use real infrastructure only in integration tests |
| **Maintainability** | Is it easy to read and change? | short Arrange, one behavior per test, intention-revealing names |

> **The most common junior mistake** is to mock every dependency and verify every internal call. Those tests pass when the code is broken and fail when the code is refactored correctly, which means they hurt both of the first two pillars.

---

## 8.2 Kinds of tests

| Kind | Scope | Speed | Phase |
|---|---|---|---|
| **Unit** | one *unit of behavior* (a class or cluster of classes) in memory | ms | **1** |
| **Integration** | your code plus real infrastructure (database, broker) in containers | 100 ms – s | 2 |
| **End-to-end / API** | the deployed system over HTTP/UI | s – min | 2–4 |
| **Architecture** | rules about dependencies between layers and modules | ms | 3 |
| **Performance / load** | throughput and latency under load (k6) | min | 4 |

**Mock only unmanaged dependencies:**

- **Managed dependencies** are things that only *your* app talks to, such as your own database. Use the **real** one in integration tests instead of mocking it.
- **Unmanaged dependencies** have side effects that are visible outside your app: an SMTP server, a payment provider, a message bus that other systems consume. **Mock** them, because the interaction *is* the observable behavior.

---

## 8.3 Setting up xUnit v3

```powershell
# Install the xUnit v3 templates (the .NET 11 SDK's built-in test templates also target xUnit v3)
dotnet new install xunit.v3.templates
dotnet new xunit3 -n TicketHub.Domain.Tests -o tests/TicketHub.Domain.Tests -f net10.0   # defaults to net8.0 without -f
dotnet sln add tests/TicketHub.Domain.Tests
dotnet add tests/TicketHub.Domain.Tests reference src/TicketHub.Domain
dotnet add tests/TicketHub.Domain.Tests package Shouldly
dotnet add tests/TicketHub.Domain.Tests package NSubstitute
dotnet add tests/TicketHub.Domain.Tests package Microsoft.Extensions.TimeProvider.Testing
dotnet test
```

> The template also creates a `global.json` with `"test": { "runner": "Microsoft.Testing.Platform" }`, which makes `dotnet test` (in the .NET 10 SDK) use **Microsoft.Testing.Platform (MTP)** instead of VSTest. Keep that file.

**What's new in xUnit v3 compared with v2:**

- Test projects are **executables**, so you can run one directly with `dotnet run`.
- They support **Microsoft.Testing.Platform**.
- `TestContext.Current` gives you `CancellationToken`, test output and test state from anywhere.
- `IAsyncLifetime` now uses `ValueTask`.
- Tests can be skipped dynamically with `Assert.Skip(...)`.
- Assembly-level fixtures are available.

### Anatomy of a test

```csharp
public class MoneyTests
{
    [Fact]
    public void Adding_money_in_the_same_currency_sums_the_amounts()
    {
        // Arrange
        var a = new Money(10.50m, "EUR");
        var b = new Money(4.25m, "EUR");

        // Act
        var total = a + b;

        // Assert
        total.ShouldBe(new Money(14.75m, "EUR"));
    }

    [Fact]
    public void Adding_money_in_different_currencies_throws()
    {
        var eur = new Money(10m, "EUR");
        var usd = new Money(10m, "USD");

        Should.Throw<InvalidOperationException>(() => eur + usd);
    }

    [Theory]
    [InlineData(-0.01)]
    [InlineData(-100)]
    public void Negative_amounts_are_rejected(decimal amount)
    {
        Should.Throw<ArgumentOutOfRangeException>(() => new Money(amount, "EUR"));
    }
}
```

### Naming

Name a test after the **behavior** in business language: `Holding_a_seat_that_is_already_held_fails`. The `Method_Scenario_ExpectedResult` style is also common. The rule is that a failing test name tells you **what broke** without opening the code.

### `[Theory]` with typed data

```csharp
public static TheoryData<int, bool> SeatCounts => new()
{
    { 0, false },
    { 1, true },
    { 10, true },
    { 11, false },
};

[Theory]
[MemberData(nameof(SeatCounts))]
public void Seat_count_must_be_between_1_and_10(int count, bool expectedSuccess)
{
    // ...
}
```

### Lifecycle and shared context

| Need | xUnit mechanism |
|---|---|
| Setup per test | the **constructor** (xUnit creates a *new class instance for every test*) |
| Teardown per test | `IDisposable` / `IAsyncDisposable` |
| Async setup per test | `IAsyncLifetime` (`ValueTask InitializeAsync()` + `DisposeAsync()`) |
| Expensive setup shared by one class (for example, a database container) | `IClassFixture<TFixture>` |
| Shared by several classes | `[CollectionDefinition]` + `ICollectionFixture<TFixture>` |
| Shared by the whole assembly (v3) | `[assembly: AssemblyFixture(typeof(TFixture))]` |
| Cancellation in async tests | `TestContext.Current.CancellationToken` |

> **Parallelism:** test *classes* (collections) run in parallel by default, and the tests inside one class run sequentially. Never share mutable static state between tests.

---

## 8.4 Test doubles

| Double | What it does | Example |
|---|---|---|
| **Dummy** | is passed in but never used | `null!` or `Substitute.For<ILogger<T>>()` |
| **Stub** | returns canned answers (the **input** to the system under test) | `repository.FindAsync(id).Returns(customer)` |
| **Spy / mock** | records calls so you can **verify outgoing interactions** | `emailSender.Received(1).SendAsync(...)` |
| **Fake** | a working, lightweight implementation | an in-memory repository, `FakeTimeProvider` |

**Rule:** *stub the queries, mock the commands.* Verify only interactions that are part of observable behavior (an email sent, a payment charged), and **never assert on stubs**.

### NSubstitute in practice

```csharp
public sealed class PlaceOrderHandler(IOrderRepository orders, IEmailSender email, TimeProvider clock)
{
    public async Task<Result<Guid>> HandleAsync(PlaceOrder command, CancellationToken ct)
    {
        if (command.Lines.Count == 0) return new Error("Order.Empty", "An order needs at least one line.");

        var order = Order.Create(command.CustomerId, command.Lines, clock.GetUtcNow());
        await orders.AddAsync(order, ct);
        await email.SendAsync(new OrderConfirmation(order.Id, command.CustomerEmail), ct);
        return order.Id;
    }
}

public class PlaceOrderHandlerTests
{
    private readonly IOrderRepository _orders = Substitute.For<IOrderRepository>();
    private readonly IEmailSender _email = Substitute.For<IEmailSender>();
    private readonly FakeTimeProvider _clock = new(new DateTimeOffset(2027, 1, 10, 12, 0, 0, TimeSpan.Zero));

    private PlaceOrderHandler CreateSut() => new(_orders, _email, _clock);

    [Fact]
    public async Task Placing_an_order_sends_a_confirmation_email()
    {
        var command = new PlaceOrder(Guid.NewGuid(), "ann@example.com", [new OrderLine("A-1", 49.90m)]);

        var result = await CreateSut().HandleAsync(command, TestContext.Current.CancellationToken);

        result.IsSuccess.ShouldBeTrue();
        await _email.Received(1).SendAsync(
            Arg.Is<OrderConfirmation>(c => c.OrderId == result.Value && c.To == "ann@example.com"),
            Arg.Any<CancellationToken>());
    }

    [Fact]
    public async Task An_empty_order_is_rejected_and_nothing_is_sent()
    {
        var command = new PlaceOrder(Guid.NewGuid(), "ann@example.com", []);

        var result = await CreateSut().HandleAsync(command, TestContext.Current.CancellationToken);

        result.Error!.Code.ShouldBe("Order.Empty");
        await _email.DidNotReceiveWithAnyArgs().SendAsync(default!, default);
    }
}
```

> In real code, prefer a **fake** in-memory `IOrderRepository` over a mock. Then you can assert on *state* ("the order exists") rather than on calls.

### Time: never use `DateTime.Now` in testable code

```csharp
// Production: inject TimeProvider (.NET 8+) and register TimeProvider.System as a singleton
public bool IsExpired(TimeProvider clock) => clock.GetUtcNow() >= ExpiresAt;

// Test: Microsoft.Extensions.TimeProvider.Testing
var clock = new FakeTimeProvider(new DateTimeOffset(2027, 1, 10, 18, 0, 0, TimeSpan.Zero));
var hold = CreateHold(expiresAt: clock.GetUtcNow().AddMinutes(10));
clock.Advance(TimeSpan.FromMinutes(10));
hold.IsExpired(clock).ShouldBeTrue();
```

`FakeTimeProvider` also drives `PeriodicTimer`, `Task.Delay(…, timeProvider)` and `CancellationTokenSource` timeouts. Your tests never sleep.

---

## 8.5 Assertion libraries

| Library | Style | License |
|---|---|---|
| xUnit `Assert` | `Assert.Equal(expected, actual)` | Apache 2.0 |
| **Shouldly** | `actual.ShouldBe(expected)`; clear failure messages | BSD |
| **AwesomeAssertions** | `actual.Should().Be(expected)`; a community fork of FluentAssertions | Apache 2.0 |
| FluentAssertions v8+ | same style as AwesomeAssertions | **commercial** for commercial use. Check your company's license before adding it |

Pick one per solution and be consistent.

---

## 8.6 Testing async code

```csharp
[Fact]
public async Task Cancelled_request_stops_loading()             // ✅ async Task, never async void
{
    using var cts = new CancellationTokenSource();
    await cts.CancelAsync();

    // ThrowsAnyAsync also accepts derived types such as TaskCanceledException; ThrowsAsync requires the exact type
    await Assert.ThrowsAnyAsync<OperationCanceledException>(
        () => _service.BuildReportAsync(Guid.NewGuid(), cts.Token));
}
```

- Never use `Thread.Sleep` or real `Task.Delay` to "wait for something". Use `FakeTimeProvider` or await the actual task.
- Use `TaskCompletionSource<T>` to control *when* a fake dependency completes (to test ordering, timeouts and concurrency).

---

## 8.7 What to test in a domain model (Phase 1 focus)

For each aggregate, such as `Event`, cover:

1. **Every invariant:** "a hold can't have more than 10 seats", "a cancelled event can't be booked".
2. **Every state transition:** `Draft → Published → Cancelled`, including the **illegal** transitions.
3. **Boundary values:** 0, 1, max, max+1, the exact expiry instant.
4. **Each error code** in the error catalog. At least one test per code.
5. **Time-based behavior** with `FakeTimeProvider`.

Optional: **property-based testing** (FsCheck, CsCheck) generates hundreds of random inputs to check properties such as "a hold followed by an expiry always frees every seat".

---

## 8.8 Coverage and mutation testing

```powershell
# Microsoft.Testing.Platform projects (the xUnit v3 template default); verified with the .NET 10 SDK
dotnet add tests/TicketHub.Domain.Tests package Microsoft.Testing.Extensions.CodeCoverage
dotnet test --coverage --coverage-output-format cobertura     # writes TestResults/<guid>.cobertura.xml

# (Older VSTest-based projects with coverlet.collector use: dotnet test --collect:"XPlat Code Coverage")

# HTML report
dotnet tool install -g dotnet-reportgenerator-globaltool
reportgenerator -reports:"**/*.cobertura.xml" -targetdir:"coveragereport" -reporttypes:Html
```

- **Coverage tells you what is *not* tested, not what *is* tested well.** 100% coverage with weak assertions is worthless.
- **Mutation testing** checks test quality. Stryker changes your code (for example, `>` becomes `>=`) and checks that a test fails. Surviving mutants reveal weak tests.

```powershell
dotnet tool install -g dotnet-stryker
cd tests/TicketHub.Domain.Tests
dotnet stryker
```

> If Stryker doesn't discover your tests, check the Stryker.NET docs for its current Microsoft.Testing.Platform support and the recommended project settings.

---

## 8.9 Anti-patterns

| Anti-pattern | Why it's bad | Instead |
|---|---|---|
| Testing private methods (through reflection or by making them `internal`) | couples tests to the implementation | test through the public behavior |
| Mocking everything, verifying every call | brittle; passes when broken | stub queries, mock only unmanaged commands, prefer fakes |
| `DateTime.Now`, `Guid.NewGuid()` or `Random` in assertions | non-deterministic | inject `TimeProvider`; assert on properties rather than generated values |
| Logic in tests (`if`, loops, calculations) | the test itself can be wrong | hard-code the expected values |
| Shared mutable state, order-dependent tests | flaky; fails in parallel | a fresh SUT per test |
| One giant test asserting 15 things | unclear failures | one *behavior* per test (several asserts about *that* behavior are fine) |
| Mocking types you don't own (`HttpClient`, `DbContext`) | mocks drift from real behavior | wrap them behind your own interface, or use integration tests |
| Ignoring flaky tests | erodes trust in the suite | fix them or delete them; treat flakiness as a bug |

---

## Interview questions

<details><summary><b>1. What makes a good unit test?</b></summary>

It protects against regressions, resists refactoring (it tests observable behavior), gives fast feedback and is easy to maintain. It's deterministic, independent and named after the behavior it checks.
</details>

<details><summary><b>2. Mock vs stub vs fake?</b></summary>

A stub provides canned inputs (queries). A mock verifies outgoing interactions (commands). A fake is a lightweight working implementation, such as an in-memory repository or `FakeTimeProvider`. Never verify calls on stubs.
</details>

<details><summary><b>3. What should you mock, and what shouldn't you?</b></summary>

Mock unmanaged out-of-process dependencies whose side effects other systems can see (email, payment, a public message bus). Don't mock managed dependencies like your own database: use the real one in integration tests. Don't mock domain classes.
</details>

<details><summary><b>4. How do you test time-dependent code?</b></summary>

Inject `TimeProvider` (`TimeProvider.System` in production) and use `FakeTimeProvider` in tests to set and advance time deterministically. It also controls timers and delays.
</details>

<details><summary><b>5. Is 100% code coverage a good goal?</b></summary>

No. Coverage measures executed lines, not meaningful assertions. Use it to find untested areas, and use mutation testing to evaluate test quality. Focus coverage on domain and business logic.
</details>

<details><summary><b>6. How do you share an expensive setup (like a database container) between tests in xUnit?</b></summary>

With `IClassFixture<T>` for a single class, `ICollectionFixture<T>` for several classes, or an assembly fixture in v3. Put async setup in `IAsyncLifetime`.
</details>

---

## Exercises

1. **Domain suite:** write the full TicketHub domain test suite following the checklist in section 8.7 (see [guide 9](09-labs-and-capstone.md)).
2. **Refactor resistance:** take a mock-heavy test at work, rewrite it as a state-based test with a fake, then refactor the production code and see which test survives.
3. **Mutation testing:** run Stryker on your domain project, kill at least 5 surviving mutants with better tests, and record the score before and after.
4. **Time:** test seat-hold expiry exactly at the boundary (1 tick before and exactly at `ExpiresAt`) using `FakeTimeProvider`.
5. **Flaky test clinic:** write a deliberately flaky test (shared static state or `DateTime.Now`), run the suite in parallel to see it fail, then fix it.

## Self-check

- [ ] I can explain the four pillars and the managed vs unmanaged distinction.
- [ ] I can set up xUnit v3 with theories, fixtures and `TestContext.Current.CancellationToken`.
- [ ] I use stubs, mocks and fakes appropriately and never assert on stubs.
- [ ] I test time with `FakeTimeProvider`, and my tests never sleep.
- [ ] I can generate a coverage report and run Stryker.

## Further reading

- *Unit Testing Principles, Practices, and Patterns* (Vladimir Khorikov). Read it all; chapters 1–8 are essential.
- The xUnit v3 documentation: *What's new in v3*, *Shared context*.
- Microsoft Learn: *Unit testing best practices for .NET*, *TimeProvider*, *Testing with FakeTimeProvider*.
