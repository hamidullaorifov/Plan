# 4. Domain-Driven Design: Tactical and Strategic

> Phase 3 · Weeks 4–5 · [← Phase index](README.md)
> Week 4 covers the tactical patterns (4.1–4.7); week 5 covers strategic design (4.8–4.11).

**Why this matters:** DDD gives you a vocabulary (aggregate, bounded context, ubiquitous language) that architects and senior engineers use daily. Even if your team doesn't "do DDD", these ideas decide **where business rules live** and **where system boundaries go**.

---

## 4.1 When DDD is worth it

| Use DDD's tactical patterns when… | Skip them when… |
|---|---|
| business rules are complex and change often (pricing, booking, insurance, compliance) | the app is mostly CRUD forms over tables |
| domain experts and developers misunderstand each other | logic is simple and stable |
| bugs come from violated invariants ("overbooked", "negative stock") | the main challenge is integration or performance |

Strategic DDD (boundaries and language) is useful **almost everywhere**. The tactical patterns are worth their cost only in the complex parts, the **core domain**.

---

## 4.2 Entities and value objects

| | Entity | Value object |
|---|---|---|
| Identity | **has an ID**; two entities with equal fields are still different | **no identity**; equal if all values are equal |
| Mutability | changes over its lifecycle | **immutable**; "changing" means replacing |
| Examples | `Order`, `Customer`, `Event` | `Money`, `Email`, `Address`, `DateRange`, `SeatId` |

### Strongly typed IDs: stop mixing up GUIDs

```csharp
public readonly record struct OrderId(Guid Value)
{
    public static OrderId New() => new(Guid.CreateVersion7());   // time-ordered GUIDs index better
}
public readonly record struct CustomerId(Guid Value);

// The compiler now catches Cancel(customerId) passed where an OrderId is expected
public Result Cancel(OrderId orderId, CustomerId requestedBy);

// EF Core mapping
builder.Property(o => o.Id).HasConversion(id => id.Value, value => new OrderId(value));
```

### Value objects remove primitive obsession

```csharp
public sealed record Email
{
    public string Value { get; }

    private Email(string value) => Value = value;

    public static Result<Email> Create(string? input)
    {
        if (string.IsNullOrWhiteSpace(input) || !input.Contains('@'))
            return new Error("Email.Invalid", "Email address is invalid.");
        return new Email(input.Trim().ToLowerInvariant());
    }

    public override string ToString() => Value;
}
```

An `Email` instance is **always valid**. You validate once, at creation, instead of everywhere it's used.

---

## 4.3 Aggregates: the most important tactical pattern

An **aggregate** is a cluster of objects treated as **one unit for data changes**, with one entry point, the **aggregate root**. It is the **consistency boundary**: every invariant inside it holds after every transaction.

```csharp
public sealed class Order : AggregateRoot<OrderId>
{
    public const int MaxLines = 20;
    private readonly List<OrderLine> _lines = [];

    private Order(OrderId id, CustomerId customerId, DateTimeOffset placedAt) : base(id)
    {
        CustomerId = customerId;
        PlacedAt = placedAt;
    }

    public CustomerId CustomerId { get; }
    public DateTimeOffset PlacedAt { get; }
    public OrderStatus Status { get; private set; } = OrderStatus.Pending;
    public IReadOnlyList<OrderLine> Lines => _lines;
    public Money Total => _lines.Aggregate(Money.Zero("EUR"), (sum, l) => sum + l.Subtotal);

    public static Order Place(CustomerId customerId, DateTimeOffset now) =>
        new(OrderId.New(), customerId, now);

    public Result AddLine(ProductId productId, int quantity, Money unitPrice)
    {
        if (Status != OrderStatus.Pending) return OrderErrors.NotEditable(Id);
        if (quantity <= 0) return OrderErrors.InvalidQuantity;
        if (_lines.Count >= MaxLines) return OrderErrors.TooManyLines(MaxLines);

        var existing = _lines.Find(l => l.ProductId == productId);
        if (existing is not null) existing.Increase(quantity);   // OrderLine is internal to the aggregate
        else _lines.Add(new OrderLine(productId, quantity, unitPrice));
        return Result.Success();
    }

    public Result Cancel(string reason, DateTimeOffset now)
    {
        if (Status is OrderStatus.Shipped or OrderStatus.Cancelled) return OrderErrors.CannotCancel(Id, Status);
        Status = OrderStatus.Cancelled;
        Raise(new OrderCancelled(Id, reason, now));
        return Result.Success();
    }
}
```

### Aggregate design rules (Vaughn Vernon)

1. **Protect true invariants inside the boundary.** Ask: "what must be consistent *immediately*, in the same transaction?"
2. **Design small aggregates.** Large aggregates cause lock contention and load too much data.
3. **Reference other aggregates by ID only** (`CustomerId`, not `Customer`). There are no navigation properties across aggregates.
4. **Use eventual consistency between aggregates.** Modify one aggregate per transaction where practical, and update others via domain events.

> **Example:** "an order's total equals the sum of its lines" is an invariant *inside* `Order`. "The customer's loyalty points update when an order is paid" is a rule *between* aggregates, handled by an event, eventually consistent.

### Encapsulation in practice

- **No public setters.** State changes only through methods that express intent: `Cancel`, `AddLine`, `Confirm`.
- Expose collections as `IReadOnlyList<T>` backed by a private `List<T>`. EF Core maps the backing field automatically when it follows the naming convention, or via `Navigation(o => o.Lines).HasField("_lines")`.
- Use a **private constructor plus a static factory**, so an aggregate can't exist in an invalid state.

---

## 4.4 Domain events

A domain event records that **something meaningful happened in the domain**, named in the past tense: `OrderPlaced`, `OrderCancelled`, `SeatsHeld`.

```csharp
public interface IDomainEvent { DateTimeOffset OccurredAt { get; } }
public sealed record OrderCancelled(OrderId OrderId, string Reason, DateTimeOffset OccurredAt) : IDomainEvent;

public interface IHasDomainEvents
{
    IReadOnlyList<IDomainEvent> DomainEvents { get; }
    void ClearDomainEvents();
}

public abstract class AggregateRoot<TId>(TId id) : IHasDomainEvents
{
    private readonly List<IDomainEvent> _domainEvents = [];

    public TId Id { get; } = id;
    public IReadOnlyList<IDomainEvent> DomainEvents => _domainEvents;

    protected void Raise(IDomainEvent domainEvent) => _domainEvents.Add(domainEvent);
    public void ClearDomainEvents() => _domainEvents.Clear();
}
```

**When do events get dispatched?**

| Option | Behavior | Risk |
|---|---|---|
| **Before `SaveChanges`**, in the same transaction | handlers' changes commit atomically with the aggregate | handlers run inside the transaction, so keep them fast and in-process |
| **After `SaveChanges`** | handlers see committed data | if the process crashes in between, the event is **lost** |
| **Into an outbox table**, in the same transaction | durable; published later, at least once | slight delay; consumers must be idempotent. **The production answer**, covered in [guide 6](06-outbox-inbox-sagas.md) |

**Domain events vs integration events:**

- *Domain* events are internal to a module, rich and may contain domain types.
- *Integration* events cross module or service boundaries. They're public, versioned, minimal and serializable. They're often *derived* from domain events.

---

## 4.5 Repositories, domain services and factories

| Building block | Role | Guideline |
|---|---|---|
| **Repository** | loads and saves **whole aggregates** | one per aggregate root; no repository for `OrderLine` |
| **Domain service** | domain logic that doesn't belong to one entity | stateless, named in domain language: `PricingService`, `SeatAllocationPolicy` |
| **Factory** | complex creation | a static method on the root, or a separate class when creation needs services |
| **Application service / handler** | orchestrates a use case: load → call the domain → save → publish | contains **no** business rules |

```csharp
// Application handler: orchestration only. The rule ("can't cancel shipped orders") lives in Order.Cancel
public sealed class CancelOrderHandler(ShopDbContext db, TimeProvider clock)
{
    public async Task<Result> HandleAsync(OrderId id, CustomerId requestedBy, string reason, CancellationToken ct)
    {
        var order = await db.Orders.Include(o => o.Lines).SingleOrDefaultAsync(o => o.Id == id, ct);
        if (order is null || order.CustomerId != requestedBy) return OrderErrors.NotFound(id);

        var result = order.Cancel(reason, clock.GetUtcNow());
        if (result.IsFailure) return result;

        await db.SaveChangesAsync(ct);   // the interceptor writes domain events to the outbox (guide 6)
        return Result.Success();
    }
}
```

---

## 4.6 Rich vs anemic domain model

- **Anemic model:** entities are bags of getters and setters, and services hold all the logic. It's fine for CRUD, but in complex domains, rules get duplicated across services and invariants are easy to bypass.
- **Rich model:** behavior lives with the data, and invariants are enforced in one place.

**Use a rich model where the rules are complex, and simple CRUD (vertical-slice projections) elsewhere.** It's not all-or-nothing.

---

## 4.7 Persisting a rich model with EF Core

```csharp
internal sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.HasKey(o => o.Id);
        builder.Property(o => o.Id).HasConversion(id => id.Value, v => new OrderId(v));
        builder.Property(o => o.CustomerId).HasConversion(id => id.Value, v => new CustomerId(v));
        builder.Property(o => o.Status).HasConversion<string>().HasMaxLength(20);
        builder.Ignore(o => o.Total);                              // computed, not stored
        builder.Ignore(o => o.DomainEvents);

        builder.OwnsMany(o => o.Lines, line =>                     // lines live and die with the order
        {
            line.WithOwner().HasForeignKey("OrderId");
            line.Property(l => l.ProductId).HasConversion(id => id.Value, v => new ProductId(v));
            line.OwnsOne(l => l.UnitPrice);                         // Money stored as columns on the line row
        });
        builder.Navigation(o => o.Lines).HasField("_lines");
    }
}
```

> **Trade-off to accept:** EF Core needs some concessions, such as a private parameterless constructor or a constructor it can bind to, and backing fields. Keep them small. Don't let persistence concerns leak into method names or invariants.
>
> **Owned vs complex types:** owned types (`OwnsOne`/`OwnsMany`) must be classes. For a struct value object such as a `readonly record struct Money`, use a complex type (`ComplexProperty`) on the root entity, or map it with a value converter. Check the EF Core 10 docs for complex-type support inside collections.

---

## 4.8 Strategic design: ubiquitous language and bounded contexts

**Ubiquitous language:** the *same* words, with the *same* meaning, used by domain experts, in conversations, in tickets **and in the code**. If the business says "hold" and the code says `Reservation`, a translation layer lives in everyone's head, and bugs live there too.

**Bounded context:** the boundary within which one model and one language are consistent. The same word can mean different things in different contexts:

| Word | Sales context | Fulfilment context | Billing context |
|---|---|---|---|
| **Customer** | a lead with preferences and a segment | a delivery address and contact | a billing account, tax ID and payment terms |
| **Order** | a cart and offer | a pick list and shipment | an invoice line |

Trying to build **one** `Customer` class for everyone produces a giant, fragile model. **Each context gets its own model.** In a modular monolith or microservices, a bounded context is usually a module or a service.

---

## 4.9 Subdomains: where to invest

| Subdomain | Meaning | Strategy |
|---|---|---|
| **Core** | what makes the business different (for example, dynamic seat pricing) | invest heavily: rich model, best developers, DDD tactics |
| **Supporting** | needed but not differentiating (venue admin) | simpler design; CRUD is fine |
| **Generic** | solved problems (auth, email, payments) | **buy or reuse**: Keycloak, SendGrid, Stripe |

---

## 4.10 Context maps: how contexts relate

| Relationship | Meaning |
|---|---|
| **Shared kernel** | two contexts share a small piece of model; it requires coordination |
| **Customer / supplier** | the upstream team considers the downstream team's needs |
| **Conformist** | the downstream simply adopts the upstream model (often with big vendors) |
| **Anti-corruption layer (ACL)** | the downstream **translates** the upstream model into its own, to protect itself; essential for legacy systems and vendor APIs |
| **Open host service / published language** | the upstream offers a stable, documented API or event schema for many consumers |

The **ACL** is the one you'll implement most often: an adapter that maps a legacy or vendor model into your domain language, so their quirks don't spread through your code.

---

## 4.11 Event storming: discovering the domain

A workshop technique (Alberto Brandolini) using sticky notes on a long wall or a Miro board:

1. **Orange: domain events** in the past tense, placed on a timeline ("Seats held", "Payment captured", "Hold expired").
2. **Hotspots (pink):** questions, conflicts, unknowns.
3. **Blue: commands** that cause events ("Hold seats"). **Yellow: actors.** **Purple: policies** ("whenever a payment is captured, confirm the order").
4. **Pale yellow: aggregates** that accept commands and emit events.
5. Draw **boundaries** around clusters of language. These become candidate bounded contexts.

**Outcome:** a shared understanding, an initial ubiquitous language, aggregate candidates and context boundaries, usually in hours instead of weeks of documents.

---

## Interview questions

<details><summary><b>1. Entity vs value object?</b></summary>

An entity has identity and a lifecycle, so equality is by ID. A value object has no identity, is immutable and is equal when all its values are equal (`Money`, `Email`). Value objects validate themselves at creation and remove primitive obsession.
</details>

<details><summary><b>2. What is an aggregate, and how big should it be?</b></summary>

It's a consistency boundary with one root that enforces its invariants within each transaction. Keep it as small as the true invariants allow, reference other aggregates by ID, and use eventual consistency (domain events) between aggregates.
</details>

<details><summary><b>3. Where should domain events be dispatched?</b></summary>

Ideally into a transactional outbox in the same transaction as the aggregate change, and published at least once afterward. Dispatching after commit without an outbox can lose events on a crash. Dispatching in-process before commit couples handlers to the transaction.
</details>

<details><summary><b>4. What is a bounded context? Give an example.</b></summary>

A boundary within which a model and its language are consistent. "Customer" means a lead in Sales, a delivery address in Fulfilment and a billing account in Billing. Each context has its own model and integrates through explicit contracts.
</details>

<details><summary><b>5. What's an anti-corruption layer?</b></summary>

A translation layer that converts an upstream or legacy model into your context's model, so the external model's concepts and quirks don't leak into your domain.
</details>

<details><summary><b>6. Anemic or rich model?</b></summary>

Use a rich model in complex core domains, where invariants and behavior belong together. An anemic or transaction-script approach is fine for simple CRUD. Mixing them per context is normal.
</details>

---

## Exercises

1. **Aggregate design:** for a booking domain, decide the aggregates and their invariants: event, seat inventory, hold, order, payment. Justify each boundary ([Lab 4](10-labs.md#lab-4-aggregate-design)).
2. **Strongly typed IDs:** introduce them in a lab project with EF Core value converters and confirm that mixing two ID types no longer compiles.
3. **Event storming:** run a 90-minute session on a domain from work, alone or with colleagues ([Lab 5](10-labs.md#lab-5-event-storming)).
4. **Language audit:** list 10 business terms from your product at work and the class names used for them. Where do they differ?

## Self-check

- [ ] I can design an aggregate with private state, intent-revealing methods and invariants.
- [ ] I know the four aggregate design rules and can apply them.
- [ ] I can explain domain vs integration events and the dispatch options.
- [ ] I can explain bounded contexts, subdomains and context-map relationships with examples.

## Further reading

- *Learning Domain-Driven Design* (Vlad Khononov): the most practical modern introduction.
- *Implementing Domain-Driven Design* (Vaughn Vernon), and his *"Effective Aggregate Design"* essays.
- *Domain-Driven Design* (Eric Evans): the original; read parts II and IV.
- Alberto Brandolini, *Introducing EventStorming*.
