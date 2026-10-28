# 3. Modular Monoliths and Microservices

> Phase 3 · Week 3 · [← Phase index](README.md)

**Why this matters:** Many teams adopted microservices too early and paid with complexity. The industry has moved toward **modular monoliths first, extracting services only when there's a concrete reason.** You'll be asked to argue this trade-off in interviews, and you'll face it in real projects.

---

## 3.1 The spectrum

```
Big ball of mud → Layered monolith → MODULAR MONOLITH → Microservices
  (no boundaries)    (technical layers)   (business modules,       (separately deployed,
                                           one deployable)          separately owned data)
```

The real question isn't "monolith or microservices?". It's **"where are the boundaries, and how strongly do we enforce them?"** Good boundaries matter more than the deployment model.

---

## 3.2 The modular monolith

**One deployable unit**, divided into **business modules** (Catalog, Booking, Payments, Notifications). Each module:

- owns its **data** (its own schema or tables, and nobody else queries them),
- exposes a **small public API** (contracts and integration events),
- hides everything else (`internal`), and
- can later be extracted into a service with limited effort.

### Layout

```
src/
├─ Shop.Host/                         Program.cs: composes the modules; one process
├─ Modules/
│  ├─ Catalog/
│  │  ├─ Shop.Catalog.Contracts/      public: ICatalogApi, DTOs, integration events
│  │  └─ Shop.Catalog/                internal: domain, features, CatalogDbContext (schema "catalog")
│  ├─ Booking/
│  │  ├─ Shop.Booking.Contracts/
│  │  └─ Shop.Booking/                references Shop.Catalog.Contracts ONLY
│  └─ Payments/ …
└─ Shared/
   └─ Shop.Shared/                    truly shared kernel: Result, Error, IClock, integration-event base types
```

**The key rule:** a module references **only other modules' `.Contracts` projects**, never their internals.

### Wiring the modules

```csharp
// Shared/Shop.Shared/IModule.cs
public interface IModule
{
    void Register(IServiceCollection services, IConfiguration configuration);
    void MapEndpoints(IEndpointRouteBuilder app);
}

// Modules/Catalog/Shop.Catalog/CatalogModule.cs
public sealed class CatalogModule : IModule
{
    public void Register(IServiceCollection services, IConfiguration configuration)
    {
        services.AddDbContext<CatalogDbContext>(o => o.UseNpgsql(configuration.GetConnectionString("shop")));
        services.AddScoped<ICatalogApi, CatalogApi>();     // the public, in-process API
    }

    public void MapEndpoints(IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/catalog").WithTags("Catalog");
        SearchProducts.Map(group);
        GetProduct.Map(group);
    }
}

// Shop.Host/Program.cs
IModule[] modules = [new CatalogModule(), new BookingModule(), new PaymentsModule()];
foreach (var module in modules) module.Register(builder.Services, builder.Configuration);
var app = builder.Build();
foreach (var module in modules) module.MapEndpoints(app);
```

### Data per module

```csharp
internal sealed class CatalogDbContext(DbContextOptions<CatalogDbContext> options) : DbContext(options)
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.HasDefaultSchema("catalog");          // each module owns a schema
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(CatalogDbContext).Assembly);
    }
}
```

- **No cross-module joins or foreign keys.** If Booking needs a product name, it either asks `ICatalogApi` or keeps its **own copy**, updated from Catalog's integration events.
- Each module has its own migrations history (`MigrationsHistoryTable("__ef_migrations", "catalog")`).

### How modules communicate

| Style | Use for | Example |
|---|---|---|
| **Synchronous in-process call** through a contracts interface | queries needed *now* | Booking calls `ICatalogApi.GetPriceAsync(productId)` |
| **Integration events** (in-process bus or broker) | reacting to something that happened | Payments publishes `PaymentCaptured`; Booking confirms the order; Notifications sends an email |

Integration events are **public contracts**: versioned, stable and minimal. Domain events stay **inside** a module.

### Enforce boundaries with architecture tests

Discipline erodes; tests don't. With NetArchTest.Rules:

```csharp
public class ModuleBoundaryTests
{
    [Fact]
    public void Booking_does_not_depend_on_catalog_internals()
    {
        var result = Types.InAssembly(typeof(BookingModule).Assembly)
            .ShouldNot()
            .HaveDependencyOn("Shop.Catalog")            // the internals namespace
            .GetResult();

        Assert.True(result.IsSuccessful, string.Join(", ", result.FailingTypeNames ?? []));
    }

    [Fact]
    public void Domain_does_not_depend_on_ef_core()
    {
        var result = Types.InAssembly(typeof(BookingModule).Assembly)
            .That().ResideInNamespace("Shop.Booking.Domain")
            .ShouldNot().HaveDependencyOn("Microsoft.EntityFrameworkCore")
            .GetResult();

        Assert.True(result.IsSuccessful);
    }
}
```

> `HaveDependencyOn("Shop.Catalog")` matches by namespace prefix, so make sure `Shop.Catalog.Contracts` isn't caught by the rule. Either use distinct root namespaces (`Shop.Catalog.Internal`), or use `HaveDependencyOnAny` with explicit internal namespaces. Run the test once with a deliberate violation to confirm it actually fails.

---

## 3.3 Microservices: what you buy and what you pay

| You buy | You pay |
|---|---|
| **Independent deployment** per team | network calls instead of method calls: latency, partial failure, retries, timeouts |
| **Independent scaling** of hot components | **distributed data**: no joins, no ACID transactions across services, so you need sagas and eventual consistency |
| Fault isolation (if designed well) | **operational load**: N pipelines, N dashboards, service discovery, versioned APIs |
| Technology freedom per service | **observability is mandatory**: distributed tracing, correlation IDs |
| Team autonomy (Conway's law working for you) | integration testing and local development get much harder |

### The fallacies of distributed computing (Deutsch)

The network is reliable · latency is zero · bandwidth is infinite · the network is secure · topology doesn't change · there is one administrator · transport cost is zero · the network is homogeneous.

**Each fallacy is a production incident waiting to happen.**

### Signals you might actually need microservices

- Several teams block each other on one deployment pipeline.
- One component has a radically different scaling profile or availability requirement (for example, a ticket-sale spike).
- One component needs different technology or a regulatory boundary (PCI scope for payments).
- **And** you already have CI/CD automation, observability and on-call maturity.

### Signals you don't

- One team, or a few developers.
- Boundaries aren't stable yet (a new product, a domain still being discovered).
- "Netflix does it", or "it's on my CV wishlist".

---

## 3.4 Microservice rules if you do go there

1. **A database per service.** A shared database means a distributed monolith: all the costs and none of the benefits.
2. **Communicate via APIs or events, never through each other's tables.**
3. **Prefer asynchronous messaging** between services, so that a failure in one doesn't cascade synchronously.
4. **Design for failure:** timeouts, retries with idempotency, circuit breakers ([guide 7](07-resilience-and-background-jobs.md)).
5. **Version APIs and messages**, and evolve them in backward-compatible ways.
6. **Put an API gateway or BFF at the edge** (YARP in .NET) to handle auth, routing and aggregation for clients.
7. **Trace everything** with OpenTelemetry (Phase 4).

---

## 3.5 From modular monolith to services: the strangler fig

1. Make sure the module's boundary is clean (architecture tests pass; no shared tables).
2. Turn its in-process contract into an HTTP/gRPC API and its in-process events into broker messages, **while still running in the monolith**.
3. Deploy the module as a separate service. Route traffic to it (YARP or a gateway); the monolith calls the new API.
4. Move its data to a separate database (it already had its own schema).
5. Remove the module's code from the monolith.

The same approach migrates a legacy .NET Framework app piece by piece to modern .NET (Phase 5 upgrade playbook).

---

## Interview questions

<details><summary><b>1. What is a modular monolith, and why start there?</b></summary>

A single deployable application split into business modules with enforced boundaries, each owning its data and exposing a small public API. You get most of the design benefits of services (clear ownership, low coupling) without network, data-consistency and operational costs. Later extraction is straightforward because the boundaries already exist.
</details>

<details><summary><b>2. How do modules communicate in a modular monolith?</b></summary>

Through public contracts: synchronous in-process interfaces for queries, and integration events (in-process or via a broker) for reactions. Never through another module's internal classes or tables.
</details>

<details><summary><b>3. Why is a shared database between microservices bad?</b></summary>

It couples services through the schema: one team's migration breaks another's service, and nothing can be deployed or scaled independently. It's a "distributed monolith": network costs without autonomy.
</details>

<details><summary><b>4. When would you split a service out?</b></summary>

When there's a concrete driver: independent team ownership and deployment cadence, different scaling or availability needs, or a regulatory or security boundary. You also need the operational maturity (CI/CD, observability, on-call) to run more services.
</details>

<details><summary><b>5. How do you keep module boundaries from eroding?</b></summary>

Use `internal` by default, contracts-only project references, a schema per module and architecture tests in CI that fail on forbidden dependencies, plus code review.
</details>

---

## Exercises

1. Sketch module boundaries for your current product at work: modules, data ownership, contracts and events. Where do today's boundaries leak?
2. Build a two-module skeleton with architecture tests, and prove the test fails when you add a forbidden reference ([Lab 3](10-labs.md#lab-3-module-boundaries-and-architecture-tests)).
3. Write a one-page "monolith vs microservices" recommendation for a fictional company of 3 teams and 12 developers.

## Self-check

- [ ] I can design a modular monolith layout with contracts, data ownership and communication rules.
- [ ] I can write architecture tests that enforce boundaries.
- [ ] I can list what microservices buy and cost, and the signals for each choice.
- [ ] I can explain the strangler-fig extraction path.

## Further reading

- Kamil Grzybek, *Modular Monolith with DDD* (GitHub sample and blog series).
- Sam Newman, *Monolith to Microservices*.
- Martin Fowler, *"MonolithFirst"* and *"StranglerFigApplication"* (articles).
- Microsoft Learn: *.NET Microservices: Architecture for Containerized .NET Applications* (free e-book).
