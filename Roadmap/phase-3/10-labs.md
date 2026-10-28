# 10. Labs

> Phase 3 · All weeks · [← Phase index](README.md)

Keep each lab in your `dotnet-levelup/labs` repository with a `NOTES.md`: what you did, what you measured, what surprised you, and the rule you'll apply at work.

| Lab | Guide | Time |
|---|---|---|
| [Lab 1: Refactoring kata](#lab-1-refactoring-kata) | [01](01-design-principles-and-patterns.md) | 3 h |
| [Lab 2: Clean vs vertical slice](#lab-2-clean-vs-vertical-slice) | [02](02-application-architecture.md) | 4 h |
| [Lab 3: Module boundaries and architecture tests](#lab-3-module-boundaries-and-architecture-tests) | [03](03-modular-monolith-and-microservices.md) | 3 h |
| [Lab 4: Aggregate design](#lab-4-aggregate-design) | [04](04-domain-driven-design.md) | 3 h |
| [Lab 5: Event storming](#lab-5-event-storming) | [04](04-domain-driven-design.md) | 2 h |
| [Lab 6: RabbitMQ delivery semantics](#lab-6-rabbitmq-delivery-semantics) | [05](05-messaging.md) | 5 h |
| [Lab 7: Outbox under failure](#lab-7-outbox-under-failure) | [06](06-outbox-inbox-sagas.md) | 5 h |
| [Lab 8: Saga with timeout and compensation](#lab-8-saga-with-timeout-and-compensation) | [06](06-outbox-inbox-sagas.md) | 5 h |
| [Lab 9: Resilience under a flaky dependency](#lab-9-resilience-under-a-flaky-dependency) | [07](07-resilience-and-background-jobs.md) | 3 h |
| [Lab 10: Distributed lock pitfalls](#lab-10-distributed-lock-pitfalls) | [08](08-distributed-systems-theory.md) | 2 h |
| [Lab 11: ADR and design doc](#lab-11-adr-and-design-doc) | [09](09-adrs-design-docs-code-review.md) | ongoing |

**Local infrastructure** (`labs/docker-compose.yml`, additions to Phase 2):

```yaml
services:
  rabbitmq:
    image: rabbitmq:4-management
    ports: ["5672:5672", "15672:15672"]   # UI at http://localhost:15672 (guest/guest locally)
  mailpit:
    image: axllent/mailpit
    ports: ["1025:1025", "8025:8025"]     # SMTP on 1025, UI at http://localhost:8025
```

---

## Lab 1: Refactoring kata

1. Clone the **Gilded Rose refactoring kata** (Emily Bache's repository on GitHub has a C# version).
2. **Write characterization tests first:** cover the existing behavior, including its weirdness, until you can refactor without fear (aim for 100% branch coverage of `UpdateQuality`).
3. Refactor in small steps (Extract Method → Strategy per item type, or a `switch` over a closed set), running the tests after every step and committing often.
4. Add the new "Conjured" item feature *after* the refactoring.

**Deliverable:** the commit history (small steps), plus notes on which smells you found and which refactorings fixed them.

---

## Lab 2: Clean vs vertical slice

Implement the same feature twice: **"cancel an order with a reason; only the owner can cancel; not after shipping; publish `OrderCancelled`"**.

1. **Clean-style:** `Domain`, `Application`, `Infrastructure` and `Api` projects; a repository interface and handler.
2. **Vertical slice:** one `Features/Orders/CancelOrder.cs` file plus a rich `Order` entity.
3. Then apply the same **change request** to both: "cancellation is allowed up to 24 h before the event; afterward it's denied".

**Deliverable:** a table of files and projects touched per change, test effort and lines of code, plus your recommendation for a 5-developer team.

---

## Lab 3: Module boundaries and architecture tests

1. Create a modular-monolith skeleton with **Catalog** and **Booking** modules, each with a `.Contracts` project and an internal project with its own `DbContext` and schema.
2. Booking needs product names: implement them once through `ICatalogApi` (a synchronous call) and once through a local copy updated from a `ProductRenamed` integration event.
3. Write architecture tests (NetArchTest or ArchUnitNET): Booking must not reference Catalog internals; domain namespaces must not reference EF Core.
4. **Violate a rule on purpose** (reference a Catalog internal class) and confirm the test fails with a clear message.

**Deliverable:** the skeleton, the tests, and a comparison of the two data-access approaches (coupling, consistency, performance).

---

## Lab 4: Aggregate design

For a booking domain (events, venues, seats, holds, orders, payments, refunds):

1. List the **invariants**. Which must be true *immediately*, and which can be eventually consistent?
2. Draw the aggregates and their boundaries. For each one, write its root, its internal entities and value objects, the commands it accepts and the events it raises.
3. Implement one aggregate (for example, `Order`) with private state, factory methods, `Result` returns, domain events and strongly typed IDs. Unit-test every invariant.
4. Map it with EF Core (owned types, backing fields, value converters) and verify the round trip with an integration test.

**Deliverable:** the aggregate diagram, the code, the tests and a short rationale for each boundary.

---

## Lab 5: Event storming

1. Pick a process from work (or the booking flow). Gather 1–4 people, or do it alone on Miro or Excalidraw.
2. 30 minutes: orange domain events on a timeline. 20 minutes: hotspots. 30 minutes: commands, actors, policies and aggregates. 10 minutes: draw candidate bounded contexts.
3. Photograph or export the board.

**Deliverable:** the board, a glossary of 10–15 ubiquitous-language terms, and 3 candidate bounded contexts with one sentence each.

---

## Lab 6: RabbitMQ delivery semantics

1. Write a producer (1,000 messages with unique IDs) and a consumer that "processes" each message for 50 ms and acks **after** processing.
2. **Crash test:** kill the consumer at around 50% progress, restart it and count the processed IDs. Expect some processed twice (at-least-once delivery).
3. Make the consumer idempotent with a processed-IDs table (PostgreSQL), and repeat. Expect zero double-processing.
4. **Poison test:** make every 100th message throw. Configure a dead-letter exchange, `nack` with `requeue: false`, and inspect the DLQ in the management UI. "Fix the bug" and redrive the dead letters.
5. **Prefetch test:** run two consumers (one slow at 200 ms, one fast at 20 ms) with prefetch 1, 10 and 100. Measure the total time and the distribution of messages.

**Deliverable:** a table of the duplicate counts, DLQ behavior and prefetch results, and your conclusions.

---

## Lab 7: Outbox under failure

1. Implement the outbox from [guide 6](06-outbox-inbox-sagas.md): the interceptor, the `outbox_messages` table and a relay with `FOR UPDATE SKIP LOCKED`, publishing to RabbitMQ.
2. **Baseline failure:** without an outbox, publish right after `SaveChanges`, add `Environment.FailFast` between the commit and the publish (behind a flag), and show that the event is lost.
3. **With the outbox:** crash at the same point, restart, and show that the relay publishes the event.
4. **Duplicate test:** crash the relay *after* publishing but *before* marking the row processed. Show the duplicate on the broker, and the consumer's inbox deduplicating it.
5. Run **two relay instances** and confirm that no message is published twice due to concurrent claiming (count the publishes per ID).

**Deliverable:** logs and counts for each scenario, plus a sequence diagram of the outbox flow.

---

## Lab 8: Saga with timeout and compensation

1. Build three small services (they can be three console apps): **Booking** (holds and confirms seats), **Payments** (fails 30% of the time and never answers 10% of the time), and a **Checkout orchestrator** with persisted saga state.
2. Implement the state machine from [guide 6](06-outbox-inbox-sagas.md): `AwaitingPayment → Confirmed | Cancelled`, with compensation (`ReleaseSeats`) and a 2-minute payment timeout (use a periodic scan or scheduled messages).
3. Send duplicate and late messages on purpose (`PaymentCaptured` after the timeout cancelled the order). The saga must ignore them, and you should log an alert for "payment captured on a cancelled order → refund needed".
4. Run 200 checkouts and check the invariant: **every order ends Confirmed or Cancelled, and no seat stays held**.

**Deliverable:** the state diagram, the invariant check results, and the list of edge cases you discovered.

---

## Lab 9: Resilience under a flaky dependency

1. Create a fake dependency API: 50% `503` responses, 10% responses delayed by 3 s, and `429` with `Retry-After: 2` on bursts.
2. Call it from a client API **without** resilience and measure the success rate, p95 latency and threads in use (`dotnet-counters`) under load from the Phase 1 LoadGen tool.
3. Add `AddStandardResilienceHandler` with tuned timeouts. Measure again.
4. **Retry storm:** add retries in three layers and count the calls the dependency receives per user request.
5. Stop the dependency completely and observe the circuit breaker opening (log its state changes with `OnOpened`/`OnClosed` callbacks).

**Deliverable:** a before/after table and a one-paragraph recommendation for your team's `HttpClient` defaults.

---

## Lab 10: Distributed lock pitfalls

1. Implement a naive Redis lock (`SET key token NX PX 2000`) protecting writes to a PostgreSQL row.
2. In worker A, take the lock and then `await Task.Delay(3000)` (simulating a GC pause) before writing. Meanwhile, worker B takes the expired lock and writes. Show that both writes happen.
3. Add **fencing**: each lock grant increments a counter (Redis `INCR`) returned as a token, and the write does `UPDATE … SET value = @v, last_token = @t WHERE id = @id AND last_token < @t`. Show that A's stale write is rejected.
4. Replace the whole thing with a conditional update or optimistic concurrency and no lock at all. Compare the complexity.

**Deliverable:** the code for the three versions and a note on when a lock is merely an optimization and when correctness requires more.

---

## Lab 11: ADR and design doc

1. Write **three ADRs** for decisions already made in your codebase at work (reverse-engineer their context and consequences).
2. Write **one design doc** for a real upcoming feature at work using the [guide 9](09-adrs-design-docs-code-review.md) template, and get it reviewed by a senior.
3. Do **10 code reviews** with the checklist and Conventional Comments, and track the issues you found.

**Deliverable:** links or copies (sanitized of anything confidential), plus a short reflection on what changed after the reviews.
