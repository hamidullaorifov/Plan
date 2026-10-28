# 1. Design Principles and Patterns in .NET

> Phase 3 · Week 1 · [← Phase index](README.md)

**Why this matters:** Principles are the vocabulary of design discussions and code reviews. At Middle level you're expected to *apply* them to real code ("this class has two reasons to change") and to recognize when following one blindly makes things worse.

---

## 1.1 Coupling and cohesion: the two forces behind every principle

| | Meaning | You want | Smell |
|---|---|---|---|
| **Coupling** | how much a change in A forces a change in B | **low** | changing the payment provider touches 14 files |
| **Cohesion** | how strongly the things inside one unit belong together | **high** | a `Utils` class with date math, email sending and PDF rendering |

Almost every principle below is a specific way to **lower coupling** or **raise cohesion**.

---

## 1.2 SOLID, with real code

### S: Single Responsibility (*one reason to change*)

"Reason to change" means **one stakeholder or one axis of change**, not "one method".

```csharp
// ❌ Three reasons to change: pricing rules (business), PDF layout (design), email delivery (infrastructure)
public sealed class InvoiceService
{
    public decimal CalculateTotal(Order order) { /* discounts, taxes */ return 0; }
    public byte[] RenderPdf(Order order) { /* layout */ return []; }
    public Task EmailAsync(Order order, byte[] pdf) { /* SMTP */ return Task.CompletedTask; }
}

// ✅ Each piece changes for one reason; an orchestrator composes them
public sealed class InvoicePricing { public decimal CalculateTotal(Order order) => 0; }
public interface IInvoiceRenderer { byte[] Render(Invoice invoice); }
public interface IInvoiceSender { Task SendAsync(Invoice invoice, byte[] document, CancellationToken ct); }
```

### O: Open/Closed (*open for extension, closed for modification*)

Add new behavior by adding code, not by editing a growing `switch`:

```csharp
// ❌ Every new payment method edits this switch (and its tests)
decimal Fee(PaymentMethod method, decimal amount) => method switch
{
    PaymentMethod.Card => amount * 0.029m,
    PaymentMethod.PayPal => amount * 0.034m,
    _ => throw new UnreachableException(),
};

// ✅ Strategy: a new method is a new class plus one registration
public interface IFeePolicy { string Method { get; } decimal Fee(decimal amount); }
public sealed class CardFee : IFeePolicy { public string Method => "card"; public decimal Fee(decimal a) => a * 0.029m; }
public sealed class PayPalFee : IFeePolicy { public string Method => "paypal"; public decimal Fee(decimal a) => a * 0.034m; }

public sealed class FeeCalculator(IEnumerable<IFeePolicy> policies)
{
    private readonly Dictionary<string, IFeePolicy> _byMethod = policies.ToDictionary(p => p.Method);
    public decimal Fee(string method, decimal amount) => _byMethod[method].Fee(amount);
}
```

> **Don't over-apply OCP.** A `switch` over a closed set that rarely changes is *fine*, and with C# 15 unions or closed hierarchies the compiler even checks it for exhaustiveness. Introduce the strategy when the set actually grows or varies by deployment.

### L: Liskov Substitution (*subtypes must honor the base type's contract*)

```csharp
// ❌ Callers of IList<T> expect Add to work; ReadOnlyCollection<T> throws NotSupportedException
IList<string> names = new List<string> { "a" }.AsReadOnly();
names.Add("b"); // NotSupportedException at runtime: a classic LSP violation in the BCL itself

// ✅ Accept the narrowest contract you need: IReadOnlyList<T> when you don't mutate
```

Typical LSP smells: overrides that throw `NotSupportedException`, overrides that strengthen preconditions ("this subtype rejects nulls the base accepted"), and `if (x is SpecialSubtype)` checks in callers.

### I: Interface Segregation (*clients shouldn't depend on methods they don't use*)

```csharp
// ❌ A "fat" repository: the reporting job depends on Delete and Update it never calls
public interface IOrderRepository
{
    Task<Order?> GetAsync(Guid id, CancellationToken ct);
    Task AddAsync(Order order, CancellationToken ct);
    Task DeleteAsync(Guid id, CancellationToken ct);
    Task<IReadOnlyList<OrderSummary>> SearchAsync(OrderQuery query, CancellationToken ct);
}

// ✅ Split by client need: the write side and the read side evolve independently (a seed of CQRS)
public interface IOrderStore { Task<Order?> GetAsync(Guid id, CancellationToken ct); Task AddAsync(Order order, CancellationToken ct); }
public interface IOrderSearch { Task<IReadOnlyList<OrderSummary>> SearchAsync(OrderQuery query, CancellationToken ct); }
```

### D: Dependency Inversion (*depend on abstractions owned by the higher-level policy*)

The important half is **who owns the interface**. The domain or application layer defines `IPaymentGateway` in *its* terms. The infrastructure layer implements it. The dependency arrow points *inward*, toward the business rules.

```csharp
// Application layer owns the abstraction, in business language
public interface IPaymentGateway
{
    Task<PaymentResult> ChargeAsync(Money amount, PaymentToken token, string idempotencyKey, CancellationToken ct);
}

// Infrastructure adapts a vendor SDK to it (see the Adapter pattern below)
public sealed class StripePaymentGateway(StripeClient stripe) : IPaymentGateway { /* ... */ }
```

> **DIP ≠ "an interface for every class".** Interfaces with exactly one implementation that will never be swapped or faked are noise. Introduce an abstraction at **boundaries**: I/O, third-party systems, time, randomness, and policies that vary.

---

## 1.3 Other principles worth naming

| Principle | Meaning | Watch out for |
|---|---|---|
| **Composition over inheritance** | build behavior by combining objects | deep base-class hierarchies (`BaseService<T> : BaseRepository<T> : …`) |
| **DRY** | one *knowledge* lives in one place | DRY on *code that looks similar* but changes for different reasons creates coupling |
| **"Duplication is cheaper than the wrong abstraction"** (Sandi Metz) | wait for the **rule of three** before abstracting | premature generic frameworks inside business code |
| **KISS / YAGNI** | build the simplest thing that meets today's needs | "we might need multi-tenancy one day" |
| **Tell, don't ask** | ask objects to do things instead of pulling their data out | `if (order.Status == X && order.Lines.Count > 0) order.Status = Y;` outside the `Order` |
| **Law of Demeter** | talk to your immediate collaborators | `order.Customer.Address.Country.VatRate` chains |

---

## 1.4 Patterns you'll see in every .NET codebase

| Pattern | Where .NET uses it | Write it yourself when… |
|---|---|---|
| **Strategy** | `IComparer<T>`, `IEqualityComparer<T>`, keyed services | an algorithm varies by type, tenant or configuration |
| **Decorator** | `DelegatingHandler`, `IChatClient` middleware, caching repositories | you add cross-cutting behavior (caching, logging, retries) without touching the core |
| **Adapter** | `IHttpClientFactory` wrapping handlers, `ILogger` providers | you wrap a vendor SDK behind your own interface |
| **Factory** | `IHttpClientFactory`, `IDbContextFactory<T>`, `ILoggerFactory` | creation needs runtime data or has a complex lifecycle |
| **Chain of responsibility / pipeline** | ASP.NET Core middleware, `DelegatingHandler`, MediatR behaviors | a request passes through ordered, independent steps |
| **Observer** | C# events, `IObservable<T>`, `IOptionsMonitor.OnChange` | many listeners react to one source |
| **Options** | `IOptions<T>` | strongly typed configuration |
| **Specification** | (libraries such as Ardalis.Specification) | reusable, composable query or validation rules |

### Decorator, two ways

```csharp
public sealed class CachedOrderStore(IOrderStore inner, HybridCache cache) : IOrderStore
{
    public async Task<Order?> GetAsync(Guid id, CancellationToken ct) =>
        await cache.GetOrCreateAsync($"order:{id}", async token => await inner.GetAsync(id, token), cancellationToken: ct);

    public Task AddAsync(Order order, CancellationToken ct) => inner.AddAsync(order, ct);
}

// Manual registration: explicit, no library
builder.Services.AddScoped<SqlOrderStore>();
builder.Services.AddScoped<IOrderStore>(sp =>
    new CachedOrderStore(sp.GetRequiredService<SqlOrderStore>(), sp.GetRequiredService<HybridCache>()));

// With Scrutor (MIT): register the core, then decorate
builder.Services.AddScoped<IOrderStore, SqlOrderStore>();
builder.Services.Decorate<IOrderStore, CachedOrderStore>();
```

### Pipeline: an outgoing HTTP handler

```csharp
public sealed class CorrelationIdHandler(IHttpContextAccessor accessor) : DelegatingHandler
{
    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken ct)
    {
        if (accessor.HttpContext?.TraceIdentifier is { } id)
            request.Headers.TryAddWithoutValidation("X-Correlation-Id", id);
        return base.SendAsync(request, ct);
    }
}

builder.Services.AddHttpContextAccessor();
builder.Services.AddTransient<CorrelationIdHandler>();
builder.Services.AddHttpClient<PaymentsClient>().AddHttpMessageHandler<CorrelationIdHandler>();
```

### Adapter: keep vendor types out of your core

```csharp
// The vendor SDK's types (StripeException, ChargeCreateOptions, …) never leave this class
public sealed class StripePaymentGateway(StripeClient stripe, ILogger<StripePaymentGateway> logger) : IPaymentGateway
{
    public async Task<PaymentResult> ChargeAsync(Money amount, PaymentToken token, string idempotencyKey, CancellationToken ct)
    {
        try
        {
            // map Money/PaymentToken to the vendor request, call it with the idempotency key, map the response back
            return PaymentResult.Succeeded("ch_123");
        }
        catch (Exception ex) when (ex is not OperationCanceledException)
        {
            logger.LogWarning(ex, "Payment provider rejected charge");
            return PaymentResult.Failed("provider_error");
        }
    }
}
```

---

## 1.5 Code smells and the refactorings that fix them

| Smell | Symptom | Refactoring |
|---|---|---|
| **Long method** | > 30–40 lines, comments separating "sections" | Extract Method, named after *intent* |
| **God class** | 2,000-line `OrderService` | Extract Class by responsibility |
| **Primitive obsession** | `string email`, `decimal amount, string currency` everywhere | Introduce value objects (`Email`, `Money`) |
| **Long parameter list** | 6+ parameters | Introduce Parameter Object or a record |
| **Feature envy** | a method uses another class's data more than its own | Move Method to where the data lives |
| **Shotgun surgery** | one change touches many classes | Move related code together (higher cohesion) |
| **Switch on type** | `if (x is A) … else if (x is B)` repeated in many places | Polymorphism, or one exhaustive `switch` in one place |
| **Speculative generality** | `IRepository<T>` + `IGenericService<T>` with one user | Inline the abstraction |

**Safe refactoring loop:** make sure tests cover the behavior → take one small step → run the tests → commit. Use the IDE's automated refactorings (Rename, Extract, Move), which are safer than hand edits.

---

## Interview questions

<details><summary><b>1. Explain SRP with an example from your work.</b></summary>

A class should have one reason to change, meaning one stakeholder or axis of change. Example: an `InvoiceService` that calculates totals, renders PDFs and sends emails changes when pricing rules, the design or the email provider change. Splitting it isolates each change and makes testing simpler.
</details>

<details><summary><b>2. What's the difference between Dependency Inversion and Dependency Injection?</b></summary>

DIP is a design principle: high-level policy shouldn't depend on low-level details, and both depend on abstractions owned by the policy. DI is a technique, passing dependencies in from outside (usually via a container), which is one way to implement DIP.
</details>

<details><summary><b>3. Give an LSP violation in .NET itself.</b></summary>

`ReadOnlyCollection<T>` implements `IList<T>` but throws `NotSupportedException` from `Add`. Arrays implementing `IList<T>` behave the same way. Code written against `IList<T>` can't safely call `Add`. Prefer `IReadOnlyList<T>` when you don't mutate.
</details>

<details><summary><b>4. When does DRY hurt?</b></summary>

When you merge code that looks the same but represents different knowledge with different reasons to change. The shared abstraction then couples unrelated features and grows flags and parameters. Wait for the rule of three, and duplicate when the reasons to change differ.
</details>

<details><summary><b>5. Name three patterns ASP.NET Core uses internally.</b></summary>

Chain of responsibility (the middleware pipeline and `DelegatingHandler`), Factory (`IHttpClientFactory`, `ILoggerFactory`), Options, Decorator (handlers wrapping handlers) and Observer (`IOptionsMonitor.OnChange`).
</details>

---

## Exercises

1. **SOLID audit:** pick one large class at work. List its reasons to change, and sketch (don't implement yet) how you'd split it.
2. **Decorator:** add caching to a repository using a decorator, first with manual registration and then with Scrutor. Write a test that proves the inner store is called once for two reads.
3. **Strategy:** replace a growing `switch` with keyed services (`AddKeyedScoped<IFeePolicy, CardFee>("card")`) and resolve the strategy with `[FromKeyedServices]` or `IServiceProvider.GetRequiredKeyedService`.
4. **Refactoring kata:** do the Gilded Rose kata in C# ([Lab 1](10-labs.md#lab-1-refactoring-kata)).

## Self-check

- [ ] I can explain each SOLID principle with a real example and a case where applying it would be overkill.
- [ ] I can implement Strategy, Decorator and Adapter in idiomatic .NET with DI.
- [ ] I can name five code smells and the refactorings that fix them.

## Further reading

- *Refactoring*, 2nd ed. (Martin Fowler), the catalog of smells and refactorings.
- *Dependency Injection Principles, Practices, and Patterns* (van Deursen and Seemann), the chapters on DIP and decorators.
- Sandi Metz, *"The Wrong Abstraction"* (blog post).
