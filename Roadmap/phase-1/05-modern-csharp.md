# 5. Modern C# (12 → 15)

> Phase 1 · Week 6 · [← Phase index](README.md)

**Why this matters:** Modern C# removes boilerplate and makes illegal states unrepresentable. Interviewers notice whether your code looks like it's from 2015 or from 2026. Teams adopting .NET 10 and 11 expect you to read *and* review these features.

### Which language version you get

The language version is **tied to the target framework** by default:

| Target framework | Default C# | Status (Oct 2026) |
|---|---|---|
| `net8.0` | C# 12 | out of support on Nov 10, 2026 |
| `net9.0` | C# 13 | out of support on Nov 10, 2026 |
| `net10.0` (LTS) | **C# 14** | **use this for production and the capstone** |
| `net11.0` (STS) | **C# 15** | GA in Nov 2026; use it in a playground project |

> Don't set `<LangVersion>latest</LangVersion>` to use features that are newer than your runtime. Some features need runtime support.

---

## 5.1 Pattern matching

```csharp
decimal CalculateDiscount(Order order) => order switch
{
    { Total: >= 1_000m, Customer.IsVip: true } => 0.15m,          // property pattern + nested member
    { Total: >= 1_000m }                       => 0.10m,          // relational pattern
    { Items.Count: > 10 }                      => 0.05m,
    { Customer: null }                         => throw new ArgumentException("Order has no customer."),
    _                                          => 0m,
};

string Classify(object value) => value switch
{
    null                    => "null",
    int n when n < 0        => "negative int",
    int and (0 or 1)        => "bit",                 // logical patterns: and / or / not
    int                     => "int",
    string { Length: 0 }    => "empty string",
    string s                => $"string({s.Length})",
    not null                => value.GetType().Name,
};

// List patterns (C# 11)
string Describe(int[] values) => values switch
{
    []                      => "empty",
    [var single]            => $"one: {single}",
    [var first, .., var last] => $"from {first} to {last}",
};
```

**Exhaustiveness:** for enums, the compiler warns (CS8509/CS8524) when a `switch` expression misses cases. Unions and `closed` hierarchies in C# 15 make *type* switches exhaustive too, as shown in section 5.8.

---

## 5.2 Object creation: `init`, `required` and primary constructors

```csharp
public sealed class CreateEventRequest
{
    public required string Name { get; init; }           // the caller MUST set it in the initializer
    public required DateTimeOffset StartsAt { get; init; }
    public string? Description { get; init; }            // optional
}

var request = new CreateEventRequest { Name = "Jazz Night", StartsAt = DateTimeOffset.UtcNow.AddDays(30) };
// new CreateEventRequest { Name = "x" };  // CS9035: required member 'StartsAt' must be set
```

### Primary constructors for classes (C# 12)

```csharp
public sealed class OrderService(IOrderRepository repository, TimeProvider clock, ILogger<OrderService> logger)
{
    public async Task<Order> PlaceAsync(Cart cart, CancellationToken ct)
    {
        var order = Order.Create(cart, clock.GetUtcNow());
        await repository.AddAsync(order, ct);
        logger.LogInformation("Order {OrderId} placed", order.Id);
        return order;
    }
}
```

> ⚠️ **The pitfall:** primary-constructor parameters are **captured variables, not `readonly` fields**. Code inside the class can reassign them (`repository = null!;` compiles). If you also copy a parameter into a field or property, you end up with two copies of the state, and the compiler warns with **CS9124**.
>
> **Team guideline:** use primary constructors for DI-style classes and *never* assign to the parameters. When immutability matters, declare an explicit field: `private readonly IClock _clock = clock;`.

---

## 5.3 Collections

```csharp
int[] a = [1, 2, 3];                                 // collection expression (C# 12)
List<int> b = [.. a, 4, 5];                          // spread
IReadOnlyList<string> empty = [];                    // works for interfaces as well
ReadOnlySpan<byte> bytes = [0x01, 0x02];             // no heap allocation

// C# 15: collection-expression arguments, with(...) as the FIRST element
List<string> names = [with(capacity: a.Length * 2), "x", "y"];
HashSet<string> tags = [with(StringComparer.OrdinalIgnoreCase), "Jazz", "JAZZ", "jazz"]; // 1 element

// params collections (C# 13): no array allocation for callers
static int Sum(params ReadOnlySpan<int> values)
{
    int total = 0;
    foreach (var v in values) total += v;
    return total;
}
Sum(1, 2, 3);
```

---

## 5.4 Less ceremony

```csharp
global using System.Collections.Frozen;           // global using (usually in GlobalUsings.cs)
using Point = (int X, int Y);                     // alias any type, including tuples (C# 12)

namespace TicketHub.Domain;                       // file-scoped namespace (C# 10); usings must come first

var json = """
    {
      "name": "Jazz Night",
      "price": 49.90
    }
    """;                                           // raw string literal (C# 11): no escaping

ReadOnlySpan<byte> header = "HTTP/1.1"u8;         // UTF-8 literal

var parse = (string s, int fallback = 0) => int.TryParse(s, out var n) ? n : fallback; // lambda default param (C# 12)
```

---

## 5.5 Generic math (C# 11 / .NET 7+)

Static abstract interface members let you write *one* algorithm for every numeric type:

```csharp
using System.Numerics;

static T Average<T>(ReadOnlySpan<T> values) where T : INumber<T>
{
    if (values.IsEmpty) throw new ArgumentException("No values.", nameof(values));
    T sum = T.Zero;
    foreach (var v in values) sum += v;
    return sum / T.CreateChecked(values.Length);
}

Average<int>([1, 2, 3]);          // 2
Average<decimal>([1.5m, 2.5m]);   // 2.0
```

You won't write this every day, but you'll *read* it in modern libraries, and it shows up in interviews as "what are static abstract members?".

---

## 5.6 C# 14 (.NET 10): the features you'll use in production

### The `field` keyword: validation without a hand-written backing field

```csharp
public sealed class Venue
{
    public required string Name
    {
        get;
        set => field = string.IsNullOrWhiteSpace(value)
            ? throw new ArgumentException("Venue name is required.", nameof(value))
            : value.Trim();
    }

    public required int Capacity
    {
        get;
        init => field = value > 0 ? value : throw new ArgumentOutOfRangeException(nameof(value));
    }
}
```

### Extension members: extension properties and static extensions

```csharp
public static class OrderExtensions
{
    extension(IEnumerable<Order> orders)                   // instance-style extensions
    {
        public bool IsEmpty => !orders.Any();               // extension property
        public decimal Revenue => orders.Sum(o => o.Total);
        public IEnumerable<Order> PaidOnly() => orders.Where(o => o.Status == OrderStatus.Paid);
    }

    extension(Order)                                        // static-style extensions on the type
    {
        public static Order Empty(Guid customerId) => new(Guid.NewGuid(), customerId, []);
    }
}

// usage
if (!orders.IsEmpty) Console.WriteLine(orders.PaidOnly().Revenue);
var draft = Order.Empty(customerId);
```

### Null-conditional assignment

```csharp
customer?.LastSeenAt = clock.GetUtcNow();   // assigns only if customer isn't null; the right side isn't evaluated otherwise
settings?.RetryCount += 1;                  // compound assignment works too (++ and -- don't)
```

### Smaller C# 14 features

- `nameof(List<>)` → `"List"`: unbound generic types are allowed in `nameof`.
- **Implicit `Span` conversions:** arrays convert to `Span<T>`/`ReadOnlySpan<T>` naturally, including as extension-method receivers.
- **Modifiers on simple lambda parameters:** `(text, out result) => int.TryParse(text, out result)`.
- **Partial constructors and events** (mostly for source generators).
- **User-defined compound assignment operators** (`+=` defined separately from `+`) for performance-sensitive types.

---

## 5.7 C# 15 (.NET 11): learn now, adopt when your team targets `net11.0`

### Union types

A union is a value that is **exactly one of** a fixed set of *case types*. The compiler provides implicit conversions from each case type and checks that `switch` expressions are exhaustive.

```csharp
public sealed record SeatsHeld(Guid HoldId, DateTimeOffset ExpiresAt);
public sealed record SeatsUnavailable(IReadOnlyList<string> SeatCodes);
public sealed record EventNotOnSale(Guid EventId);

public union HoldSeatsResult(SeatsHeld, SeatsUnavailable, EventNotOnSale);

public HoldSeatsResult HoldSeats(/* ... */)
{
    if (Status != EventStatus.Published) return new EventNotOnSale(Id);   // implicit conversion
    // ...
    return new SeatsHeld(holdId, expiresAt);
}

// The compiler checks every case is handled, so no default arm is needed
IResult ToHttp(HoldSeatsResult result) => result switch
{
    SeatsHeld held          => Results.Ok(held),
    SeatsUnavailable taken  => Results.Conflict(taken),
    EventNotOnSale missing  => Results.NotFound(missing),
};
```

**Why it matters:** expected outcomes become part of the method *signature*. That's a type-safe alternative to exceptions for control flow and to hand-rolled `Result` classes (see [guide 7](07-error-handling.md)).

### Closed hierarchies

`closed` on a class (or record) means it can only be derived from **inside the same assembly**, so the compiler knows every direct descendant:

```csharp
public closed record class PaymentState;
public sealed record class Pending : PaymentState;
public sealed record class Authorized(string AuthCode) : PaymentState;
public sealed record class Captured(decimal Amount) : PaymentState;
public sealed record class Failed(string Reason) : PaymentState;

string Describe(PaymentState state) => state switch
{
    Pending            => "Waiting for the payment provider",
    Authorized(var c)  => $"Authorized ({c})",
    Captured(var amt)  => $"Captured {amt:0.00}",
    Failed(var why)    => $"Failed: {why}",
    // no warning: every direct descendant is handled
};
```

- A `closed` class is implicitly `abstract`, and it can't be combined with `sealed`, `static` or an explicit `abstract`.
- Closedness isn't transitive. Mark intermediate classes `closed` too if you want exhaustive checks further down the hierarchy.

| Use a **union** when… | Use a **closed hierarchy** when… |
|---|---|
| the cases are independent types, possibly existing ones or types from other libraries | the cases share a base type, shared members or behavior |
| you're modeling "one of these results" | you're modeling states or variants of one concept (a state machine) |

### Other C# 15 features

```csharp
// Extension indexers
public static class SequenceIndexer
{
    extension(IEnumerable<int> sequence)
    {
        public int this[int index] => sequence.ElementAt(index);
    }
}

// Labeled break / continue: no more boolean flags or goto in nested loops
outer: for (int row = 0; row < grid.Height; row++)
{
    for (int col = 0; col < grid.Width; col++)
    {
        if (grid[row, col].IsBlocked) continue outer;
        if (grid[row, col].IsGoal) break outer;
    }
}
```

- **Memory safety (preview):** this begins tying `unsafe` to the operations that actually *access* memory, such as dereferencing a pointer. It requires `LangVersion=preview`. Just be aware it exists.

---

## 5.8 Team adoption guidelines

| Feature | Adopt? |
|---|---|
| File-scoped namespaces, global usings, collection expressions, raw strings, pattern matching | ✅ always |
| `required` / `init` for DTOs and options | ✅ |
| Records for DTOs, messages and value objects | ✅ |
| Primary constructors | ✅ for DI classes, but never reassign the parameters |
| `field` keyword | ✅ for validated properties |
| Extension members | ✅ but keep them discoverable, and don't hide business logic in extensions |
| Unions / closed hierarchies (C# 15) | ✅ for result types and state machines, once the team targets `net11.0`+ |
| Generic math, `ref struct` tricks, `unsafe` | only in libraries or hot paths, after measuring |

Write the conventions down in `.editorconfig` so the IDE and the build enforce them. That's better than a wiki page nobody reads.

---

## Interview questions

<details><summary><b>1. What are the pitfalls of primary constructors on classes?</b></summary>

The parameters are captured, mutable variables rather than `readonly` fields. They can be reassigned, and copying them into a field or property as well creates two copies of the state (warning CS9124). Use an explicit `readonly` field when immutability matters.
</details>

<details><summary><b>2. <code>required</code> vs a constructor parameter?</b></summary>

`required` forces callers to set a member in an object initializer, which keeps initializer syntax readable for DTOs. A constructor is better when the invariants span several members and need validation in one place.
</details>

<details><summary><b>3. How do C# 15 unions differ from a Result&lt;T&gt; class?</b></summary>

Unions are a language feature. They provide implicit conversions from every case and **compiler-checked exhaustive switches**, and the outcomes appear directly in the signature. A hand-written `Result<T>` usually models only success or failure with an `Error` object, and nothing forces callers to handle every case.
</details>

<details><summary><b>4. When would you use a closed hierarchy instead of a union?</b></summary>

When the variants are conceptually subtypes of one thing with shared members, such as payment states. Use a union when the cases are independent, possibly pre-existing types.
</details>

<details><summary><b>5. What is the <code>field</code> keyword?</b></summary>

It refers to the compiler-synthesized backing field inside a property accessor. You can add validation or normalization to an auto-property without declaring a private field yourself.
</details>

---

## Exercises

1. **Refactor:** take a 2015-style class from work (constructor-assigned fields, `if`/`else` type checks, `string.Format`) and modernize it. Discuss the diff with a colleague.
2. **Pattern matching:** write a ticket-price calculator using one `switch` expression that covers age group × day of week × VIP status.
3. **C# 15 playground (`net11.0`):** model the TicketHub `HoldSeats` outcome as a union and `PaymentState` as a closed hierarchy. Delete one case and observe the exhaustiveness warning.
4. **Extension members:** add extension properties `IsSoldOut` and `AvailableSeatCount` on `Event` in a separate static class. Do you think they belong on the type itself instead? Write down why.

## Self-check

- [ ] I can read and write switch expressions with property, relational, logical and list patterns.
- [ ] I know the primary-constructor pitfall and my team's guideline for it.
- [ ] I can use the C# 14 `field` keyword, extension members and null-conditional assignment.
- [ ] I can explain C# 15 unions vs closed hierarchies and when to use each.

## Further reading

- Microsoft Learn: *What's new in C# 12*, *13*, *14* and *15*; *Pattern matching overview*; *Union types*; *Closed modifier*.
- The `dotnet/csharplang` repository on GitHub, for the design notes behind each feature.
