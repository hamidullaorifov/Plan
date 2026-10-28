# 5. Messaging Fundamentals and Brokers

> Phase 3 · Weeks 6–7 · [← Phase index](README.md)
> Week 6 covers sections 5.1–5.6 (concepts); week 7 covers 5.7–5.10 (brokers and libraries).

**Why this matters:** As soon as two parts of a system communicate asynchronously, you face duplicates, ordering, poison messages and lost messages. These are the topics behind "design an idempotent consumer" and "what happens if the consumer crashes?". Messaging is also where many production incidents begin.

---

## 5.1 Why asynchronous messaging

| Benefit | Example |
|---|---|
| **Temporal decoupling:** the receiver doesn't have to be up when the sender sends | the email service is down for 5 minutes, and no orders fail |
| **Load leveling:** a queue absorbs spikes | a ticket sale opens; 50,000 confirmations queue up and are processed at a steady rate |
| **Fan-out:** one event, many independent reactions | `OrderPaid` → send tickets, update analytics, award loyalty points |
| **Failure isolation:** one slow consumer doesn't block the sender | the analytics pipeline is slow; checkout isn't affected |

The cost is **eventual consistency**, plus everything in this guide: duplicates, ordering and operational complexity.

---

## 5.2 Message types and anatomy

| Type | Intent | Naming | Receivers |
|---|---|---|---|
| **Command** | "please do X"; it can be rejected | imperative: `SendTicketEmail`, `CapturePayment` | exactly **one** logical handler |
| **Event** | "X happened"; a fact, it can't be rejected | past tense: `OrderPaid`, `SeatsReleased` | **zero to many** subscribers |
| **Query (request/reply)** | "tell me X" | rarely over a broker; prefer HTTP/gRPC | one |

```jsonc
{
  "messageId": "0b9e1c42-6f0a-4c5b-9d43-2c1a7f5b8e01",
  "type": "shop.orders.order-paid.v1",
  "occurredAt": "2027-06-10T18:42:11Z",
  "correlationId": "a7f3…",            // ties together the whole business flow
  "causationId": "5d1e…",              // the ID of the message that caused this one
  "traceparent": "00-4bf9…-00f0…-01",  // W3C trace context (Phase 4)
  "data": { "orderId": "…", "customerId": "…", "total": 99.80, "currency": "EUR" }
}
```

**Event payload design:**

- **Notification events** are thin (an ID plus the change). Consumers call back for details, which couples them in time.
- **Event-carried state transfer** includes the data consumers need, so they're autonomous. The trade-off is larger messages and schema coupling.

Choose deliberately. Default to including what typical consumers need, and **never** include secrets or unnecessary PII.

---

## 5.3 Queues, topics and consumers

```
Queue (point-to-point):         Topic / pub-sub:
Producer ─▶ [ queue ] ─▶ C1      Producer ─▶ (topic) ─┬▶ [sub A] ─▶ Email service
                      └▶ C2                           ├▶ [sub B] ─▶ Analytics
 (competing consumers: each                           └▶ [sub C] ─▶ Loyalty
  message goes to ONE consumer)   (each subscription gets its OWN copy)
```

- **Competing consumers:** several instances read the same queue to scale out. Each message is processed by one of them.
- **Subscriptions:** each subscribing *service* gets its own queue or subscription, so it receives every event independently.

---

## 5.4 Delivery guarantees

| Guarantee | Meaning | How it arises |
|---|---|---|
| At-most-once | may be lost, never duplicated | ack before processing |
| **At-least-once** | never lost, **may be duplicated** | ack after processing; a crash before the ack means **redelivery** |
| "Exactly-once" | marketing, in general | impossible end to end across independent systems |

**Effectively-once = at-least-once delivery + idempotent processing.** This is the standard design. Assume **every message can arrive twice** (and late, and out of order).

### Acknowledgement mechanics

1. The broker hands the message to a consumer and **locks** it (a visibility timeout in SQS, a peek-lock in Service Bus, an unacked delivery in RabbitMQ).
2. The consumer processes it and **acks/completes**, and the broker deletes it.
3. If the consumer crashes or the lock expires, the broker **redelivers** the message, possibly to another instance.

> **The lock-expiry trap:** if processing takes longer than the lock duration, the message is redelivered **while the first consumer is still working on it**. You get duplicate processing even with no crash. Keep processing short, renew locks, or size the timeout properly.

---

## 5.5 Idempotent consumers

| Technique | How | Good for |
|---|---|---|
| **Natural idempotency** | the operation is idempotent by nature: `SET status = 'Paid'`, upsert by key | state updates |
| **Deduplication (inbox) table** | store `(MessageId, Consumer)` with a unique constraint in the **same transaction** as the side effect; skip if it exists | database side effects ([guide 6](06-outbox-inbox-sagas.md)) |
| **Conditional update / version check** | `UPDATE … WHERE version = @expected` | out-of-order protection |
| **Idempotency key at the external system** | pass the message ID as the provider's idempotency key | payments, emails (if the provider supports it) |

```csharp
// Natural idempotency plus an out-of-order guard: only move forward
var rows = await db.Orders
    .Where(o => o.Id == evt.OrderId && o.Status == OrderStatus.Pending)
    .ExecuteUpdateAsync(s => s.SetProperty(o => o.Status, OrderStatus.Paid), ct);
// rows == 0 → already paid (a duplicate) or in another state: log it and ack
```

---

## 5.6 Ordering, poison messages and retries

### Ordering

- Most brokers order **within a queue or partition**, not globally. With competing consumers, even a single queue's messages are **processed** out of order.
- Need per-entity order? Use **partition keys** (Kafka), **sessions** (Service Bus, keyed by `OrderId`) or **message groups** (SQS FIFO).
- Better still, **design for disorder**: version numbers, timestamps from the source and state machines that ignore stale transitions.

### Retries

| Kind | Use for |
|---|---|
| **Immediate retries** (1–3, in-process) | short transient glitches |
| **Delayed retries** (back-off: 10 s, 1 min, 10 min) | a downstream service that's down |
| **Dead-letter queue (DLQ)** after N attempts | **poison messages** that will never succeed (bad data, bugs) |

**DLQ discipline:** alert on DLQ depth, inspect messages, fix the cause, then **redrive** them (replay to the main queue). A DLQ nobody looks at is a black hole for lost business.

### Schema evolution

- **Additive changes only** within a version: add optional fields, never rename or remove.
- Consumers are **tolerant readers**: ignore unknown fields and supply defaults for missing ones.
- For breaking changes, publish a **new type/version** (`order-paid.v2`) alongside the old one until all consumers migrate.

---

## 5.7 Broker landscape

| Broker | Model | Strengths | Watch out for |
|---|---|---|---|
| **RabbitMQ** | queues + exchanges (routing) | flexible routing, mature, great locally; quorum queues for durability | ordering only per queue; you operate it (or use a managed offering) |
| **Azure Service Bus** | queues, topics + subscriptions | sessions (ordered per key), DLQ, scheduled messages, duplicate detection, transactions within a namespace | Azure-only; message size limits by tier |
| **AWS SQS / SNS** | SQS queues; SNS topics fanning out to SQS | serverless, near-infinite scale, FIFO queues with dedup | visibility timeout tuning; SNS+SQS wiring for pub/sub |
| **Apache Kafka** (or Azure Event Hubs, Amazon MSK) | **a partitioned, replicated log** | huge throughput, **replay** (retention), ordering per partition key, stream processing | it's a log, not a queue: no per-message ack/DLQ out of the box; consumer groups and offsets need care |

> **Queue vs log:** a queue *removes* messages once they're consumed. A log *keeps* them for a retention period, and each consumer group tracks its own **offset**. With a log you can replay history and add new consumers later.

### RabbitMQ concepts

- **Exchanges** receive messages and route them to **queues** through **bindings**:
  - `direct`: exact routing-key match.
  - `topic`: patterns such as `orders.*.paid`.
  - `fanout`: every bound queue.
- **Acks** with `autoAck: false`: ack after processing; `nack`/`reject` to requeue or dead-letter.
- **Prefetch** (`BasicQos`) limits unacked messages per consumer. Too high and one consumer hoards the work; too low and throughput drops.
- **Dead-letter exchange (DLX):** configure it on the queue; rejected or expired messages go there.
- Use **quorum queues** for durable, replicated queues in production.

---

## 5.8 Code: RabbitMQ.Client 7 (async API)

```csharp
using System.Text.Json;
using RabbitMQ.Client;
using RabbitMQ.Client.Events;

var factory = new ConnectionFactory { HostName = "localhost" };
await using var connection = await factory.CreateConnectionAsync();
await using var channel = await connection.CreateChannelAsync();

await channel.QueueDeclareAsync(queue: "order-paid", durable: true, exclusive: false, autoDelete: false);

// Publish
var evt = new OrderPaid(Guid.NewGuid(), Guid.NewGuid(), 99.80m, "EUR");
var props = new BasicProperties
{
    MessageId = Guid.NewGuid().ToString(),
    ContentType = "application/json",
    Persistent = true,                      // survives a broker restart (with a durable queue)
    Type = "shop.orders.order-paid.v1",
};
await channel.BasicPublishAsync(exchange: "", routingKey: "order-paid", mandatory: true,
    basicProperties: props, body: JsonSerializer.SerializeToUtf8Bytes(evt));

// Consume
await channel.BasicQosAsync(prefetchSize: 0, prefetchCount: 10, global: false);
var consumer = new AsyncEventingBasicConsumer(channel);
consumer.ReceivedAsync += async (_, ea) =>
{
    try
    {
        var message = JsonSerializer.Deserialize<OrderPaid>(ea.Body.Span)!;
        await HandleAsync(message, ea.BasicProperties.MessageId);       // must be idempotent
        await channel.BasicAckAsync(ea.DeliveryTag, multiple: false);
    }
    catch (Exception)
    {
        // requeue: false sends it to the DLX, if configured, instead of a hot retry loop
        await channel.BasicNackAsync(ea.DeliveryTag, multiple: false, requeue: false);
    }
};
await channel.BasicConsumeAsync(queue: "order-paid", autoAck: false, consumer: consumer);

public sealed record OrderPaid(Guid OrderId, Guid CustomerId, decimal Total, string Currency);
```

## 5.9 Code: Azure Service Bus

```csharp
using Azure.Identity;
using Azure.Messaging.ServiceBus;

await using var client = new ServiceBusClient("shop-prod.servicebus.windows.net", new DefaultAzureCredential());

// Publish to a topic
ServiceBusSender sender = client.CreateSender("orders");
await sender.SendMessageAsync(new ServiceBusMessage(BinaryData.FromObjectAsJson(evt))
{
    MessageId = evt.OrderId.ToString(),        // enables broker-side duplicate detection, if turned on
    Subject = "order-paid.v1",
    SessionId = evt.OrderId.ToString(),        // ordered processing per order (session-enabled subscription)
});

// Consume a subscription
ServiceBusProcessor processor = client.CreateProcessor("orders", "email-service",
    new ServiceBusProcessorOptions { AutoCompleteMessages = false, MaxConcurrentCalls = 8 });

processor.ProcessMessageAsync += async args =>
{
    var paid = args.Message.Body.ToObjectFromJson<OrderPaid>();
    await HandleAsync(paid!, args.Message.MessageId);
    await args.CompleteMessageAsync(args.Message);   // after MaxDeliveryCount failures → the DLQ automatically
};
processor.ProcessErrorAsync += args =>
{
    Console.Error.WriteLine(args.Exception);
    return Task.CompletedTask;
};
await processor.StartProcessingAsync();
```

> With sessions, use `CreateSessionProcessor` instead. Sessions also need the subscription to be **session-enabled**.

---

## 5.10 Libraries vs raw SDKs

A messaging framework typically gives you serialization, retries and delayed redelivery, DLQ handling, an outbox/inbox, sagas, topology setup and OpenTelemetry tracing.

| Option | License | Notes |
|---|---|---|
| Raw SDK (RabbitMQ.Client, Azure.Messaging.ServiceBus, AWSSDK.SQS, Confluent.Kafka) | open source | full control; you build retries, outbox, inbox and tracing yourself. **Do it once to learn** |
| **MassTransit** | v8: Apache 2.0; **v9+: commercial** | very popular; transport-agnostic; outbox and sagas included |
| **Wolverine** | MIT | messaging plus a mediator plus a durable outbox; works with Marten/EF Core |
| NServiceBus | commercial | mature, with strong tooling |
| Rebus | open source | a lightweight service bus |

> **Rule:** understand the raw mechanics first (this guide plus the labs). Then pick a library for production, after checking its license and maintenance status.

---

## Interview questions

<details><summary><b>1. What does "at-least-once delivery" mean for your code?</b></summary>

Every message can be delivered more than once (after a crash before the ack, a lock expiry or a retry), so consumers must be idempotent. Effectively-once processing is at-least-once delivery plus idempotency.
</details>

<details><summary><b>2. How do you design an idempotent consumer?</b></summary>

Prefer naturally idempotent operations (set state, upsert). Otherwise record the processed message ID in an inbox table with a unique constraint, in the same transaction as the side effect, and skip duplicates. For external calls, pass the message ID as an idempotency key. Guard against out-of-order messages with versions or state checks.
</details>

<details><summary><b>3. Queue vs topic? Queue vs log?</b></summary>

A queue delivers each message to one of the competing consumers. A topic delivers a copy to every subscription. A queue deletes messages after they're acknowledged. A log such as Kafka retains them, and each consumer group tracks its offset, so you can replay history and add consumers later.
</details>

<details><summary><b>4. What's a poison message, and how do you handle it?</b></summary>

A message that fails every time (bad data or a bug). After a bounded number of retries with back-off, move it to a dead-letter queue, alert on DLQ depth, fix the cause and redrive it. Never retry it forever.
</details>

<details><summary><b>5. How do you guarantee ordering per order ID?</b></summary>

Use a partition key (Kafka), sessions (Service Bus) or message groups (SQS FIFO) keyed by the order ID, so messages for one order are processed sequentially. Better still, make consumers tolerant of disorder through state machines and versions.
</details>

---

## Exercises

1. Run RabbitMQ locally with the management UI. Publish 1,000 messages, kill the consumer halfway through and observe redelivery and duplicates ([Lab 6](10-labs.md#lab-6-rabbitmq-delivery-semantics)).
2. Configure a DLX, make every 10th message fail, and redrive the dead letters after "fixing" the bug.
3. Measure throughput with a prefetch of 1, 10 and 100, using two consumers with uneven processing times.
4. Write a schema-evolution test: a v1 consumer reads a v1.1 message with an extra field, and nothing breaks.

## Self-check

- [ ] I can explain commands vs events, queues vs topics and queues vs logs.
- [ ] I can explain at-least-once delivery, acks, locks and redelivery, and design idempotent consumers.
- [ ] I know how to handle ordering, retries, DLQs and schema evolution.
- [ ] I can write a raw RabbitMQ or Service Bus producer and consumer.

## Further reading

- *Enterprise Integration Patterns* (Hohpe and Woolf), the classic pattern catalog (also online).
- *Designing Data-Intensive Applications* (Kleppmann), the chapter on stream processing.
- The RabbitMQ tutorials (rabbitmq.com) and the Azure Service Bus documentation on sessions and dead-lettering.
