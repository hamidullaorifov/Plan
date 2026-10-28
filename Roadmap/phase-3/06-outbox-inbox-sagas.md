# 6. Outbox, Inbox and Sagas

> Phase 3 · Weeks 8–9 · [← Phase index](README.md)
> Week 8 covers the outbox and inbox (6.1–6.5); week 9 covers sagas (6.6–6.9).

**Why this matters:** "Save to the database, then publish a message" looks harmless, but it loses or invents events under failure. The **outbox**, the **inbox** and **sagas** are the standard answers, and Middle-level interviews ask about them: *"Why does the Outbox pattern exist? What breaks without it?"*

---

## 6.1 The dual-write problem

```csharp
// ❌ Two independent writes, with no atomicity between them
db.Orders.Add(order);
await db.SaveChangesAsync(ct);                 // (1) committed
await bus.PublishAsync(new OrderPlaced(order.Id), ct);   // (2) the process can crash right before this
```

| Failure point | Result |
|---|---|
| Crash after (1), before (2) | the order exists, but **no event**. Downstream (email, inventory) never hears about it |
| Publish first, then the DB commit fails | a **phantom event**: the email says "thanks for your order" for an order that doesn't exist |
| Publish times out but actually succeeded | a retry publishes a **duplicate** |

You can't make a database and a broker commit atomically without distributed transactions (2PC), which most brokers don't support and which hurt availability.

---

## 6.2 The transactional outbox

**Idea:** write the event **into the same database, in the same transaction** as the business change. A separate **relay** reads unpublished rows and publishes them, at least once.

```
┌──────────── one DB transaction ───────────┐
│ UPDATE orders …                            │
│ INSERT INTO outbox (id, type, payload, …)  │
└────────────────────────────────────────────┘
          │ (later, asynchronously)
          ▼
   Outbox relay ──publish──▶ Broker ──▶ Consumers (idempotent!)
          │
          └─ marks the row processed_at = now()
```

**Guarantees:** no lost events (they're committed with the data) and no phantom events (they're rolled back with the data). Duplicates are still possible if the relay publishes and then crashes before marking the row, so **consumers must be idempotent**.

---

## 6.3 Implementing an outbox with EF Core

### The outbox row

```csharp
public sealed class OutboxMessage
{
    public Guid Id { get; init; }
    public required string Type { get; init; }           // a stable name, e.g. "shop.orders.order-cancelled.v1"
    public required string Payload { get; init; }        // JSON (jsonb on PostgreSQL)
    public DateTimeOffset OccurredAt { get; init; }
    public DateTimeOffset? ProcessedAt { get; set; }
    public int Attempts { get; set; }
    public string? LastError { get; set; }
}
```

> **Use stable type names, not CLR names.** `typeof(OrderCancelled).FullName` breaks the day you rename or move the class while old rows are still unpublished. Keep a small registry that maps each event type to a versioned name.

### An interceptor that turns domain events into outbox rows

```csharp
using System.Text.Json;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Diagnostics;

public sealed class DomainEventsToOutboxInterceptor(IEventTypeRegistry registry, TimeProvider clock)
    : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData, InterceptionResult<int> result, CancellationToken cancellationToken = default)
    {
        if (eventData.Context is { } db)
        {
            var aggregates = db.ChangeTracker.Entries<IHasDomainEvents>()
                .Select(e => e.Entity)
                .Where(a => a.DomainEvents.Count > 0)
                .ToList();

            foreach (var aggregate in aggregates)
            {
                foreach (var domainEvent in aggregate.DomainEvents)
                {
                    db.Set<OutboxMessage>().Add(new OutboxMessage
                    {
                        Id = Guid.CreateVersion7(),
                        Type = registry.NameOf(domainEvent.GetType()),
                        Payload = JsonSerializer.Serialize(domainEvent, domainEvent.GetType()),
                        OccurredAt = clock.GetUtcNow(),
                    });
                }
                aggregate.ClearDomainEvents();
            }
        }
        return base.SavingChangesAsync(eventData, result, cancellationToken);
    }
}

// Registration
builder.Services.AddScoped<DomainEventsToOutboxInterceptor>();
builder.Services.AddDbContext<ShopDbContext>((sp, options) => options
    .UseNpgsql(builder.Configuration.GetConnectionString("shop"))
    .AddInterceptors(sp.GetRequiredService<DomainEventsToOutboxInterceptor>()));
```

Now `SaveChangesAsync` atomically commits the aggregate **and** its events, with no extra code in handlers.

### The relay: safe with several app instances

PostgreSQL's `FOR UPDATE SKIP LOCKED` lets several relay instances work in parallel without picking the same rows:

```csharp
public sealed class OutboxRelay(
    IServiceScopeFactory scopes, IMessagePublisher publisher, TimeProvider clock, ILogger<OutboxRelay> logger)
    : BackgroundService
{
    private const int BatchSize = 50;
    private const int MaxAttempts = 10;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(1), clock);
        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            try
            {
                await PublishBatchAsync(stoppingToken);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                logger.LogError(ex, "Outbox relay iteration failed");
            }
        }
    }

    private async Task PublishBatchAsync(CancellationToken ct)
    {
        await using var scope = scopes.CreateAsyncScope();
        var db = scope.ServiceProvider.GetRequiredService<ShopDbContext>();
        await using var tx = await db.Database.BeginTransactionAsync(ct);

        // Table and column names follow your naming convention (snake_case shown here)
        var batch = await db.OutboxMessages
            .FromSql($"""
                SELECT * FROM outbox_messages
                WHERE processed_at IS NULL AND attempts < {MaxAttempts}
                ORDER BY occurred_at
                LIMIT {BatchSize}
                FOR UPDATE SKIP LOCKED
                """)
            .ToListAsync(ct);

        foreach (var message in batch)
        {
            try
            {
                await publisher.PublishAsync(message.Type, message.Payload, messageId: message.Id, ct);
                message.ProcessedAt = clock.GetUtcNow();
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                message.Attempts++;
                message.LastError = ex.Message;
                logger.LogWarning(ex, "Publishing outbox message {MessageId} failed", message.Id);
            }
        }

        await db.SaveChangesAsync(ct);
        await tx.CommitAsync(ct);
    }
}
```

### Operational details

| Concern | Practice |
|---|---|
| **Message ID** | publish with the outbox row's `Id` as the message ID, so consumers can deduplicate |
| **Latency** | polling every 1 s is usually fine; use `LISTEN/NOTIFY` (PostgreSQL) or a CDC tool (Debezium) for near-real-time delivery |
| **Ordering** | `ORDER BY occurred_at` gives rough order; strict per-aggregate order needs partitioning by aggregate ID |
| **Poison rows** | cap attempts, alert on rows with `attempts >= MaxAttempts`, and inspect them like a DLQ |
| **Cleanup** | delete or archive processed rows older than N days in a scheduled job |
| **Indexes** | a partial index on `occurred_at WHERE processed_at IS NULL` |

> MassTransit, Wolverine and NServiceBus include production-grade outboxes. Build one yourself **once** to understand it ([Lab 7](10-labs.md#lab-7-outbox-under-failure)), then prefer a proven implementation.

---

## 6.4 The inbox: an idempotent consumer done right

```csharp
public sealed class ProcessedMessage
{
    public Guid MessageId { get; init; }
    public required string Consumer { get; init; }       // primary key: (MessageId, Consumer)
    public DateTimeOffset ProcessedAt { get; init; }
}

public sealed class AwardLoyaltyPointsHandler(ShopDbContext db, TimeProvider clock)
{
    public async Task HandleAsync(Guid messageId, OrderPaid evt, CancellationToken ct)
    {
        await using var tx = await db.Database.BeginTransactionAsync(ct);

        db.ProcessedMessages.Add(new ProcessedMessage
        {
            MessageId = messageId,
            Consumer = nameof(AwardLoyaltyPointsHandler),
            ProcessedAt = clock.GetUtcNow(),
        });

        var account = await db.LoyaltyAccounts.SingleAsync(a => a.CustomerId == evt.CustomerId, ct);
        account.Award(points: (int)evt.Total, reason: $"Order {evt.OrderId}");

        try
        {
            await db.SaveChangesAsync(ct);                // the inbox row and the side effect commit together
            await tx.CommitAsync(ct);
        }
        catch (DbUpdateException ex) when (ex.InnerException is Npgsql.PostgresException { SqlState: Npgsql.PostgresErrorCodes.UniqueViolation })
        {
            // A duplicate delivery: the inbox row already exists, so the work was done before. Ack and move on.
        }
    }
}
```

**Side effects outside your database** (email, a payment provider) can't share your transaction. Use the provider's **idempotency key** (the message ID), or accept and design for rare duplicates. A duplicate "your tickets" email is annoying; a duplicate charge isn't acceptable.

---

## 6.5 Outbox + inbox together

```
Service A: [business change + outbox row] ──relay──▶ broker ──▶ Service B: [inbox row + business change]
              one transaction                                       one transaction
```

The end-to-end result is **effectively-once processing** of a business fact across two services, without distributed transactions.

---

## 6.6 Sagas: long-running business transactions

**Problem:** "place an order" spans several services (Booking holds seats → Payments charges the card → Booking confirms → Notifications sends tickets). There's no ACID transaction across them.

**A saga** is a sequence of local transactions. Each step publishes an event or command that triggers the next. When a step fails, **compensating actions** semantically undo the earlier steps.

| Step | Action | Compensation |
|---|---|---|
| 1 | Hold seats (Booking) | Release seats |
| 2 | Charge card (Payments) | Refund |
| 3 | Confirm order (Booking) | n/a (the final step) |
| 4 | Send tickets (Notifications) | n/a (idempotent; retry until it succeeds) |

> Compensation isn't rollback. The charge *happened*, and a refund is a new business action that customers and accountants can see. Design compensations with domain experts.

---

## 6.7 Choreography vs orchestration

| | Choreography | Orchestration |
|---|---|---|
| How | each service reacts to events and emits new ones; there's no central brain | a **saga orchestrator** (a state machine) sends commands and waits for replies |
| Pros | loose coupling, simple for 2–3 steps | the flow is explicit in one place; easier to monitor, change and handle timeouts |
| Cons | the flow is implicit and hard to follow and debug; cyclic dependencies creep in | the orchestrator can become a "god service" that's coupled to every participant |
| Use when | short flows with few participants | 4+ steps, timeouts, complex compensation, business visibility needed |

### An orchestrator as a persisted state machine

```csharp
public enum CheckoutState { AwaitingPayment, Confirmed, Cancelled }

public sealed class CheckoutSaga
{
    public Guid OrderId { get; init; }                      // correlation ID
    public CheckoutState State { get; private set; } = CheckoutState.AwaitingPayment;
    public DateTimeOffset PaymentDeadline { get; init; }
    public uint Version { get; private set; }               // optimistic concurrency token

    // Each handler returns the commands to send. The caller saves the saga plus outbox rows in ONE transaction.
    public IReadOnlyList<object> On(PaymentCaptured evt) => State switch
    {
        CheckoutState.AwaitingPayment => Transition(CheckoutState.Confirmed, new ConfirmOrder(OrderId)),
        _ => [],                                             // a duplicate or late event: ignore it (idempotent)
    };

    public IReadOnlyList<object> On(PaymentFailed evt) => State switch
    {
        CheckoutState.AwaitingPayment => Transition(CheckoutState.Cancelled, new ReleaseSeats(OrderId), new CancelOrder(OrderId, evt.Reason)),
        _ => [],
    };

    public IReadOnlyList<object> OnTimeout(DateTimeOffset now) =>
        State == CheckoutState.AwaitingPayment && now >= PaymentDeadline
            ? Transition(CheckoutState.Cancelled, new ReleaseSeats(OrderId), new CancelOrder(OrderId, "Payment timeout"))
            : [];

    private object[] Transition(CheckoutState next, params object[] commands)
    {
        State = next;
        return commands;
    }
}
```

**What makes a saga robust:**

1. **State is persisted** and loaded by correlation ID for every incoming message.
2. **Transitions are idempotent.** Duplicate or late messages are ignored based on the current state.
3. **Concurrency is controlled** with a version or row version on the saga row, because two messages for the same saga may arrive at the same time.
4. **Commands go out through the outbox**, in the same transaction as the saga's state change.
5. **Timeouts** are scheduled messages (Service Bus scheduled delivery, a delayed retry) or a periodic job that scans for expired deadlines.

### Tooling

| Option | Notes |
|---|---|
| MassTransit state machines | a rich DSL, persistence and timeouts (check the v9 license) |
| Wolverine sagas | simple handler-style sagas (MIT) |
| NServiceBus sagas | mature, commercial |
| **Durable workflow engines**: Azure Durable Functions, Temporal (.NET SDK), Dapr Workflow | write the saga as ordinary async code; the engine persists state and replays it. Great for long-running processes |

---

## 6.8 Eventual consistency in the user experience

- Return `202 Accepted` with a status URL, or show a **"Processing payment…"** state instead of pretending the operation finished.
- Push updates (SSE or SignalR) or let the client poll the status endpoint.
- **Read-your-writes:** after a command, read from the primary (not a lagging replica or cache) for that user.

---

## 6.9 Common mistakes

- An outbox relay that runs inside the request ("publish right after SaveChanges, in the same request"). That's the dual write again.
- Consumers that aren't idempotent "because duplicates are rare". They aren't rare during incidents, which is exactly when they hurt most.
- Sagas without timeouts, so seats stay held forever when a payment message is lost.
- Compensations that can fail with no retry or alert.
- Huge events containing the entire aggregate, coupling every consumer to your internal model.

---

## Interview questions

<details><summary><b>1. Why does the Outbox pattern exist? What breaks without it?</b></summary>

Writing to the database and publishing to a broker are two separate writes with no shared transaction. A crash between them loses the event, and publishing first can create phantom events. The outbox writes the event in the same database transaction as the business change, and a relay publishes it at least once, so nothing is lost or invented. Consumers still need to deduplicate.
</details>

<details><summary><b>2. How do you make the outbox relay safe with several app instances?</b></summary>

Claim rows with row locks that skip already-locked rows (`FOR UPDATE SKIP LOCKED` on PostgreSQL, `READPAST`/`UPDLOCK` hints on SQL Server) inside a transaction, process the batch, mark the rows processed and commit. Or run a single relay instance through leader election.
</details>

<details><summary><b>3. What's the inbox pattern?</b></summary>

The consumer records each processed message ID (per consumer) in a table with a unique constraint, in the same transaction as its side effects. A duplicate delivery violates the constraint, and the consumer skips it.
</details>

<details><summary><b>4. Saga: orchestration vs choreography?</b></summary>

In choreography, services react to each other's events, which is decoupled but makes the flow implicit. In orchestration, a central state machine sends commands and handles replies, timeouts and compensation, so the flow is explicit and observable at the cost of a central coordinator. Use choreography for short flows and orchestration for complex ones.
</details>

<details><summary><b>5. Why not use a distributed transaction (2PC)?</b></summary>

Most brokers and cloud databases don't support it. It also blocks participants while the coordinator is unavailable, which hurts availability and latency, and it couples services tightly. Sagas trade atomicity for availability and use compensation instead.
</details>

---

## Exercises

1. Build the outbox with an EF interceptor and a relay. Kill the process between commit and publish, and prove no event is lost ([Lab 7](10-labs.md#lab-7-outbox-under-failure)).
2. Deliver the same message twice to an inbox-protected consumer and prove the side effect happens once.
3. Implement the checkout saga with a simulated payment service that fails 30% of the time and sometimes never answers. Verify compensation and the timeout ([Lab 8](10-labs.md#lab-8-saga-with-timeout-and-compensation)).
4. Draw the checkout flow as choreography and as orchestration, and list two failure scenarios for each.

## Self-check

- [ ] I can explain the dual-write problem and how an outbox solves it.
- [ ] I can implement an outbox (interceptor plus relay) and an inbox.
- [ ] I can design a saga with compensations, idempotent transitions, concurrency control and timeouts.
- [ ] I can choose between choreography and orchestration and justify the choice.

## Further reading

- Chris Richardson, *Microservices Patterns*, the chapters on sagas and transactional messaging (also microservices.io).
- The MassTransit and Wolverine documentation pages on their transactional outboxes.
- Microsoft Learn: *Saga distributed transactions pattern* (Azure Architecture Center).
