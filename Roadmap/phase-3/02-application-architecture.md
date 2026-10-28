# 2. Application Architecture: Layered, Clean, Vertical Slices and CQRS

> Phase 3 · Week 2 · [← Phase index](README.md)

**Why this matters:** "Which architecture do you use and why?" is a standard interview question. A Middle developer must explain the **dependency rule**, compare Clean Architecture with vertical slices honestly, and avoid ceremony that adds files without adding value.

---

## 2.1 What architecture is for

Architecture is about **managing change**: making the likely changes cheap and isolating the risky parts (database, vendors, frameworks). Every style answers two questions:

1. **How is code grouped?** By technical layer (controllers, services, repositories) or by feature (place order, cancel order)?
2. **Which way do dependencies point?** Toward the database, or toward the business rules?

---

## 2.2 Classic layered (N-tier)

```
API / UI  →  Business logic  →  Data access  →  Database
```

- **Pros:** simple and familiar; fine for CRUD.
- **Cons:** business logic depends on data access, so the domain is shaped by the database. A feature touches every layer (shotgun surgery), and "services" grow into god classes.

---

## 2.3 Clean / Onion / Hexagonal (ports and adapters)

The three names describe the same idea: **dependencies point inward, toward the domain.**

```
          ┌───────────────────────────────┐
          │ Infrastructure (EF, HTTP, MQ) │ ← adapters implement ports
          │  ┌─────────────────────────┐  │
          │  │ Application (use cases) │  │ ← orchestrates; defines ports (interfaces)
          │  │  ┌───────────────────┐  │  │
          │  │  │ Domain (entities, │  │  │ ← pure C#, no framework references
          │  │  │ rules, events)    │  │  │
          │  │  └───────────────────┘  │  │
          │  └─────────────────────────┘  │
          │ API (endpoints) composes it all│
          └───────────────────────────────┘
```

**Typical solution layout:**

```
src/
├─ Shop.Domain/            entities, value objects, domain events, domain errors (no NuGet dependencies)
├─ Shop.Application/       use-case handlers, ports (IOrderStore, IPaymentGateway), DTOs, validation
├─ Shop.Infrastructure/    EF Core DbContext, repositories, payment adapter, email adapter
└─ Shop.Api/               endpoints, DI composition (Program.cs), auth, ProblemDetails
```

| Pros | Cons |
|---|---|
| The domain is testable in isolation, with no database | many projects and interfaces, even for simple CRUD |
| Infrastructure is swappable (in theory) and contained (in practice) | a simple feature touches 4 projects |
| Clear rules that architecture tests can enforce | teams often add "pass-through" layers that do nothing |

> **When Clean Architecture is overkill:** CRUD-heavy apps with little business logic, small internal tools, and prototypes. Repositories that only wrap `DbContext` add indirection without benefit, because `DbContext` already *is* a unit of work and repository.

---

## 2.4 Vertical slice architecture

**Group by feature, not by layer.** Each slice owns everything it needs: request, validation, handler, data access and response.

```
src/Shop.Api/
├─ Features/
│  ├─ Orders/
│  │  ├─ PlaceOrder.cs          request + validator + handler + endpoint mapping
│  │  ├─ CancelOrder.cs
│  │  └─ GetMyOrders.cs         query with a projection straight from DbContext
│  └─ Catalog/
│     └─ SearchProducts.cs
├─ Domain/                       shared rich entities (Order, Product) where rules live
└─ Infrastructure/               ShopDbContext, configurations, adapters
```

```csharp
// Features/Orders/GetMyOrders.cs: a read slice doesn't need a repository or a domain model
public static class GetMyOrders
{
    public sealed record Item(Guid Id, DateTimeOffset PlacedAt, decimal Total, string Status);

    public static void Map(IEndpointRouteBuilder app) =>
        app.MapGet("/api/me/orders", HandleAsync).RequireAuthorization();

    private static async Task<Ok<List<Item>>> HandleAsync(ClaimsPrincipal user, ShopDbContext db, CancellationToken ct)
    {
        var customerId = user.GetCustomerId();
        var items = await db.Orders
            .AsNoTracking()
            .Where(o => o.CustomerId == customerId)
            .OrderByDescending(o => o.PlacedAt)
            .Select(o => new Item(o.Id, o.PlacedAt, o.Total, o.Status.ToString()))
            .ToListAsync(ct);
        return TypedResults.Ok(items);
    }
}
```

| Pros | Cons |
|---|---|
| A change usually touches **one folder** | risk of duplicated logic across slices if there's no shared domain |
| Each slice picks the simplest approach (EF projection, Dapper, a rich domain call) | needs discipline: "shared" code must be truly shared |
| Easy to delete a feature | less familiar to teams trained on layers |

### The pragmatic combination

Most modern .NET teams use **vertical slices for organization, plus a rich domain model where the business rules are complex.** Write slices call the domain (`order.Cancel(reason)`); read slices project directly from the database. Use Clean-style ports only at the real boundaries (payment, email, external APIs).

---

## 2.5 CQRS, the light version

**Command Query Responsibility Segregation** separates **writes** (commands that change state through the domain model) from **reads** (queries that return DTOs, optimized for the screen).

| | Commands | Queries |
|---|---|---|
| Purpose | change state, enforce invariants | return data |
| Model | rich domain aggregates, tracked by EF | DTO projections, `AsNoTracking`, Dapper, SQL views |
| Returns | an ID, a `Result`, or nothing | data |
| Scale | correctness | speed |

**Light CQRS needs only one database.** Heavier variants add read replicas, denormalized read tables updated by events, or a separate read store (Elasticsearch, Redis). Each step adds **eventual consistency**, so only take it when you've measured the need.

> **Common mistake:** CQRS combined with event sourcing and a separate read database for a CRUD admin panel. CQRS is a spectrum. Start at the light end.

---

## 2.6 The mediator question

**MediatR** popularized `_mediator.Send(new PlaceOrder(...))` with pipeline behaviors for validation, logging and transactions. Since 2025, **MediatR (and AutoMapper) use commercial licenses** for new versions. You'll still meet it in many codebases, so understand it, and check the license before adding it to new projects.

**Do you need a mediator at all?** Usually not. Injecting the handler directly is explicit, "Go to Definition" works, and there's no reflection:

```csharp
public interface ICommandHandler<in TCommand, TResult>
{
    Task<TResult> HandleAsync(TCommand command, CancellationToken ct);
}

public sealed record PlaceOrder(Guid CustomerId, IReadOnlyList<OrderLineDto> Lines);

public sealed class PlaceOrderHandler(ShopDbContext db, TimeProvider clock)
    : ICommandHandler<PlaceOrder, Result<Guid>>
{
    public async Task<Result<Guid>> HandleAsync(PlaceOrder command, CancellationToken ct)
    {
        var order = Order.Place(command.CustomerId, command.Lines, clock.GetUtcNow());
        if (order.IsFailure) return order.Error!;
        db.Orders.Add(order.Value);
        await db.SaveChangesAsync(ct);
        return order.Value.Id;
    }
}

// Endpoint: inject the handler directly
app.MapPost("/api/orders", async (PlaceOrder cmd, ICommandHandler<PlaceOrder, Result<Guid>> handler, CancellationToken ct) =>
    (await handler.HandleAsync(cmd, ct)).Match(
        id => Results.Created($"/api/orders/{id}", new { id }),
        error => error.ToProblem()));
```

**Cross-cutting behavior without a mediator** uses an open-generic decorator (Scrutor):

```csharp
public sealed class LoggingCommandHandler<TCommand, TResult>(
    ICommandHandler<TCommand, TResult> inner,
    ILogger<LoggingCommandHandler<TCommand, TResult>> logger) : ICommandHandler<TCommand, TResult>
{
    public async Task<TResult> HandleAsync(TCommand command, CancellationToken ct)
    {
        var started = Stopwatch.GetTimestamp();
        try
        {
            return await inner.HandleAsync(command, ct);
        }
        finally
        {
            logger.LogInformation("Handled {Command} in {ElapsedMs} ms",
                typeof(TCommand).Name, Stopwatch.GetElapsedTime(started).TotalMilliseconds);
        }
    }
}

builder.Services.Scan(scan => scan.FromAssemblyOf<PlaceOrderHandler>()
    .AddClasses(c => c.AssignableTo(typeof(ICommandHandler<,>)))
    .AsImplementedInterfaces()
    .WithScopedLifetime());
builder.Services.Decorate(typeof(ICommandHandler<,>), typeof(LoggingCommandHandler<,>));
```

Alternatives if you want a mediator: **Wolverine** (MIT; also a message bus) or source-generated mediators (check each license).

---

## 2.7 Mapping between layers

| Approach | Pros | Cons |
|---|---|---|
| **Manual mapping** (constructor or extension method) | explicit, fast, refactor-safe, easy to debug | some typing |
| **Mapperly** (source generator, Apache 2.0) | compile-time generated code, no reflection, errors at build time | another tool to learn |
| AutoMapper | widespread in older code | runtime reflection, mapping errors found at runtime, commercial license for new versions |

```csharp
// Manual, recommended for most code
public static class OrderMappings
{
    public static OrderDto ToDto(this Order order) =>
        new(order.Id, order.PlacedAt, order.Total.Amount, order.Total.Currency, order.Status.ToString());
}

// Mapperly
[Mapper]
public static partial class OrderMapper
{
    public static partial OrderDto ToDto(Order order);
}
```

> In read slices, **project in the query** (`.Select(o => new OrderDto(...))`) instead of loading entities and mapping them afterward. EF Core then only selects the needed columns.

---

## 2.8 Where does each concern go?

| Concern | Place |
|---|---|
| Input shape validation (required, length, format) | API boundary (endpoint validation) |
| Business invariants ("max 10 seats", "can't cancel a shipped order") | **Domain** |
| Authorization ("only the owner can cancel") | endpoint policy plus a resource check in the handler |
| Transactions | the handler / unit of work (`SaveChangesAsync` once per command) |
| Mapping to DTOs | endpoint or query projection |
| Logging, metrics, retries | decorators, middleware, `HttpClient` handlers |
| Calls to external systems | infrastructure adapters behind ports |

---

## Interview questions

<details><summary><b>1. What is the dependency rule in Clean Architecture?</b></summary>

Source-code dependencies point inward only. The domain knows nothing about the application, infrastructure or UI layers, and the application layer defines interfaces (ports) that infrastructure implements. Business rules then don't change when frameworks or databases do.
</details>

<details><summary><b>2. Clean Architecture vs vertical slices?</b></summary>

Clean groups by technical layer and protects the domain with inward dependencies. It suits complex domains, but it's ceremony-heavy. Vertical slices group by feature, so one change usually touches one folder and each slice chooses its own approach. A common combination is slices for organization plus a rich domain for complex rules.
</details>

<details><summary><b>3. Is CQRS the same as event sourcing? Do you need two databases?</b></summary>

No to both. CQRS separates read and write models. Event sourcing stores state as a sequence of events. They're often combined but independent. Light CQRS uses one database, with tracked aggregates for writes and DTO projections for reads.
</details>

<details><summary><b>4. Do you need MediatR?</b></summary>

No. Direct handler injection is simpler and explicit, and decorators give the same cross-cutting behavior. A mediator decouples endpoints from handlers and gives a uniform pipeline, but it adds indirection, and new MediatR versions require a commercial license.
</details>

<details><summary><b>5. Should a repository wrap DbContext?</b></summary>

Often not. `DbContext` already implements the unit-of-work and repository patterns. Add an abstraction when you need to hide persistence from a rich domain, to enforce aggregate boundaries (one repository per aggregate root), or when you have several storage technologies. Avoid generic `IRepository<T>` wrappers that just re-expose `IQueryable`.
</details>

---

## Exercises

1. Implement the same feature ("cancel order with a reason, only by the owner, not after shipping") in a Clean-style solution and in a vertical-slice solution ([Lab 2](10-labs.md#lab-2-clean-vs-vertical-slice)). Count the files touched and write down which one you'd pick for your team.
2. Replace a MediatR pipeline behavior at work (or in a sample) with a Scrutor decorator, and compare readability.
3. Find a read endpoint at work that loads entities and maps them in memory. Rewrite it as a projection and compare the SQL.

## Self-check

- [ ] I can draw Clean Architecture and explain the dependency rule.
- [ ] I can compare layered, Clean and vertical slices, with pros, cons and when to use each.
- [ ] I can explain light CQRS and when to go further.
- [ ] I can implement handlers and cross-cutting decorators without a mediator library.

## Further reading

- Jimmy Bogard, *"Vertical Slice Architecture"* (blog post and talks).
- Robert C. Martin, *Clean Architecture*, the parts on the dependency rule and boundaries.
- Microsoft Learn: *Architect modern web applications with ASP.NET Core and Azure* (free e-book), the chapters on common architectures.
- The `dotnet/eShop` reference app on GitHub.
