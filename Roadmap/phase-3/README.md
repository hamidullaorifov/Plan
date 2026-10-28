# Phase 3: Architecture and Distributed Systems

> **Months 6–9 (May 3 – Jul 31, 2027)**
> **Goal:** understand *why* codebases are structured the way they are, and make and defend design decisions, both inside one service and across services connected by messages.
> ← [Main roadmap](../level-up.md) · [← Phase 2](../phase-2/README.md) · [Phase 4 →](../phase-4/README.md)

---

## How to study

Use the same loop: **Read → Run → Explain → Check → Apply**. Phase 3 adds one habit: **argue both sides.** For every pattern you learn (Clean Architecture, CQRS, microservices, sagas), write down when it *helps* and when it's *overkill*. Interviewers at Middle level care less about whether you know a pattern and more about whether you know *when not to use it*.

---

## Topics

| # | Guide | Weeks | Effort |
|---|---|---|---|
| 1 | [Design principles and patterns in .NET](01-design-principles-and-patterns.md) | W1 | 6–8 h |
| 2 | [Application architecture: layered, Clean, vertical slices, CQRS](02-application-architecture.md) | W2 | 6–8 h |
| 3 | [Modular monoliths and microservices](03-modular-monolith-and-microservices.md) | W3 | 6–8 h |
| 4 | [Domain-driven design: tactical and strategic](04-domain-driven-design.md) | W4–W5 | 10–12 h |
| 5 | [Messaging fundamentals and brokers](05-messaging.md) | W6–W7 | 10–12 h |
| 6 | [Outbox, inbox and sagas](06-outbox-inbox-sagas.md) | W8–W9 | 10–12 h |
| 7 | [Resilience and background jobs](07-resilience-and-background-jobs.md) | W10 | 6–8 h |
| 8 | [Distributed systems theory for practitioners](08-distributed-systems-theory.md) | W11 | 6–8 h |
| 9 | [ADRs, design docs and code review](09-adrs-design-docs-code-review.md) | W12 | 4–6 h |
| 10 | [Labs](10-labs.md) | All weeks | ~35 h |

---

## Week-by-week plan

| Week | Dates | Theory (Mon/Wed/Thu) | Tue: algorithms / system design | Sat: labs |
|---|---|---|---|---|
| W1 | May 3 | 01 Principles and patterns | Graphs ×2 | Lab 1: refactoring kata |
| W2 | May 10 | 02 Application architecture | System design: URL shortener | Lab 2: Clean vs vertical slice |
| W3 | May 17 | 03 Modular monolith | 1-D DP ×2 | Lab 3: module boundaries and architecture tests |
| W4 | May 24 | 04 DDD, tactical | System design: rate limiter | Lab 4: aggregate design |
| W5 | May 31 | 04 DDD, strategic | 2-D DP ×2 | Lab 5: event storming |
| W6 | Jun 7 | 05 Messaging, sections 1–6 | System design: notification service | Lab 6: RabbitMQ delivery semantics |
| W7 | Jun 14 | 05 Messaging, sections 7–10 | Intervals ×2 | Lab 6 continued: DLQ and prefetch |
| W8 | Jun 21 | 06 Outbox and inbox | System design: ticket booking | Lab 7: outbox under failure |
| W9 | Jun 28 | 06 Sagas | Greedy ×2 | Lab 8: saga with timeout and compensation |
| W10 | Jul 5 | 07 Resilience and jobs | System design: chat | Lab 9: resilience under a flaky dependency |
| W11 | Jul 12 | 08 Distributed theory | Mixed ×2 | Lab 10: distributed lock pitfalls |
| W12 | Jul 19 | 09 ADRs, docs, reviews | Mixed ×2 | Lab 11: write an ADR and a design doc |
| W13 | Jul 26 | Review the self-checks | Mock system design interview | Lab write-ups, exit check |

> **At work during this phase:** write one design doc or ADR, even for a small feature, and get it reviewed. Review your peers' PRs every week with substantive comments.

---

## Toolbox additions

| Tool | Purpose |
|---|---|
| RabbitMQ (`rabbitmq:4-management` image) | A local broker with a management UI on port 15672 |
| Mailpit (`axllent/mailpit` image) | A fake SMTP server with a web UI, for notification experiments |
| NetArchTest.Rules or ArchUnitNET | Architecture tests |
| Mermaid (in Markdown) | Diagrams in design docs and ADRs |
| Excalidraw or Miro | Event storming and whiteboarding |

---

## Phase 3 exit criteria

- [ ] I can explain why the Outbox pattern exists and what breaks without it.
- [ ] I can design an idempotent consumer and explain at-least-once delivery.
- [ ] I can argue for a modular monolith vs microservices in a given context.
- [ ] I can explain a saga with compensation and a timeout, and choose between orchestration and choreography.
- [ ] I wrote a design doc or ADR at work and got it reviewed.
- [ ] I review peers' PRs regularly with substantive comments, not just "LGTM".
- [ ] All labs have a `NOTES.md` with measurements and conclusions.
