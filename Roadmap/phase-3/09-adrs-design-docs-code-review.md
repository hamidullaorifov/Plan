# 9. ADRs, Design Docs and Code Review

> Phase 3 · Week 12 · [← Phase index](README.md)

**Why this matters:** At Middle level, your influence grows through **writing** and **reviewing**, not just coding. A clear design doc prevents weeks of rework. An ADR stops the same debate from happening every six months. A thoughtful review catches the bug and also teaches the author. These are exactly the "senior behaviors" promotion committees look for.

---

## 9.1 Architecture Decision Records (ADRs)

An ADR is a **short, immutable record of one significant decision**: the context, the choice and its consequences.

**Write one when** a decision is hard to reverse, affects several teams or modules, or people will later ask "why did we do it this way?". Examples: choosing a message broker, moving to a modular monolith, adopting the Result pattern, picking PostgreSQL over SQL Server.

### Template (MADR-style)

```markdown
# 0007. Use a transactional outbox for integration events

- Status: Accepted            <!-- Proposed | Accepted | Deprecated | Superseded by 00NN -->
- Date: 2027-06-24
- Deciders: Anna (lead), Marco, you

## Context
Order events are published right after SaveChanges. In the last quarter, two incidents lost
OrderPaid events when pods restarted during deployment, so customers didn't receive tickets.

## Decision drivers
- No lost events; at-least-once delivery is acceptable
- No new infrastructure this quarter
- Must work with several app instances

## Considered options
1. Keep publishing after SaveChanges and add retries
2. Transactional outbox table + background relay (FOR UPDATE SKIP LOCKED)
3. CDC with Debezium

## Decision
Option 2. Retries (1) don't cover crashes between commit and publish; CDC (3) needs Kafka Connect,
which we don't operate.

## Consequences
- + Events are committed atomically with business data
- − Up to ~1 s extra latency; consumers must deduplicate (inbox table)
- − New operational task: monitor the outbox backlog and clean up processed rows
```

**Practices:**

- Store ADRs in the repository (`docs/adr/0007-….md`) and review them in PRs like code.
- **Never edit an accepted ADR's decision.** Write a new one that *supersedes* it. History is the point.
- Keep them short: one or two pages. An ADR is a decision record, not a design document.

---

## 9.2 Design documents

**Write one when** the work takes more than about a week, crosses team or module boundaries, touches data migrations or security, or carries real risk. **Skip it** for small, well-understood changes.

### Template

```markdown
# Design: Seat holds with automatic expiry

## 1. Context and problem
What's happening today, why it hurts (data, incidents, user complaints).

## 2. Goals / Non-goals
- Goal: no double booking under 1,000 concurrent buyers
- Non-goal: dynamic pricing (separate project)

## 3. Proposal
Overview diagram, then details: API changes, data model, flows, state machine.

## 4. Alternatives considered
At least two, with why they were rejected.

## 5. Risks and mitigations
Concurrency, performance, migrations, backward compatibility.

## 6. Rollout plan
Feature flag, migration steps, backfill, rollback strategy.

## 7. Observability
Metrics, logs, traces, alerts, dashboards. How we'll know it works in production.

## 8. Security and privacy
Authorization, PII, abuse cases (a bot holding every seat).

## 9. Open questions
Things reviewers should help decide.
```

### Diagrams as code (Mermaid)

```mermaid
sequenceDiagram
    participant C as Client
    participant A as Booking API
    participant DB as PostgreSQL
    participant R as Outbox relay
    participant B as Broker
    C->>A: POST /holds (seats)
    A->>DB: UPDATE seats ... WHERE version = @v ; INSERT outbox
    DB-->>A: committed
    A-->>C: 201 Created (holdId, expiresAt)
    R->>DB: SELECT ... FOR UPDATE SKIP LOCKED
    R->>B: publish SeatsHeld
```

### Running a design review

1. Share the doc **2–3 days before** the meeting. Reviewers comment asynchronously first.
2. Use the meeting only for **open questions and disagreements**. Don't read the doc aloud.
3. Record the decisions in the doc (and in an ADR if they're significant).
4. **Disagree and commit:** once decided, everyone supports the decision, and new information reopens it formally.

---

## 9.3 Code review: as a reviewer

**Purpose, in order:** correctness and security → maintainability → knowledge sharing → consistency. Style nitpicks are last, and **automate them** (formatters, analyzers, `.editorconfig`).

### Checklist

| Area | Questions |
|---|---|
| **Correctness** | Does it do what the ticket says? Edge cases: empty, null, max, concurrent, duplicate? |
| **Tests** | Do the tests prove the behavior? Would they fail if the code were wrong? |
| **Design** | Is this the right place for the code? Does it follow existing patterns? Any needless abstraction? |
| **Data** | Are migrations backward-compatible? Indexes for new queries? N+1? Transactions? |
| **Security** | Authorization on every endpoint and resource (IDOR)? Input validation? Secrets? PII in logs? |
| **Resilience** | Timeouts, cancellation, idempotency, retries in one layer only? |
| **Observability** | Can we debug this in production: logs, metrics, traces? |
| **Compatibility** | API or message contract changes that break consumers? |
| **AI-generated code** | Do the packages actually exist and are they maintained? Are the APIs current? Is there a subtle "almost right" logic error? Are there tests that pin the behavior? |

### How to comment (Conventional Comments)

```
issue (blocking): this query runs once per order line → N+1. Use Include or a projection.
suggestion: consider TimeProvider here so the expiry logic is testable.
question: what happens if the payment webhook arrives before the hold is saved?
nitpick (non-blocking): naming, `HoldSeatsAsync` matches the other handlers.
praise: really clear test names, I learned the rules just by reading them.
```

- Comment on the **code**, not the person ("this method…", not "you…").
- **Explain why**, and link to docs or ADRs.
- **Approve with minor comments** when the remaining points are non-blocking. Don't hold PRs hostage over nits.
- **Review within one working day.** Slow reviews are one of the biggest drags on team throughput.

---

## 9.4 Code review: as an author

1. **Keep PRs small.** Under ~400 changed lines gets a real review; 2,000 lines gets "LGTM".
2. **Self-review first.** Read your own diff in the PR UI before requesting a review; you'll find 30% of the issues.
3. **Write a good description:**

```markdown
## What
Adds seat-hold expiry via a background job (every minute).

## Why
Holds currently never expire → seats stay blocked after abandoned checkouts (TICKET-412).

## How
- `ExpiredHoldsJob` releases holds where expires_at < now (batch of 500, SKIP LOCKED)
- New index `ix_seat_holds_expires_at` (migration 20270612_AddHoldExpiryIndex)

## Testing
- Unit: expiry boundaries with FakeTimeProvider
- Integration: Testcontainers, 10k holds released in < 2 s

## Risks / rollout
Behind flag `HoldExpiry`; safe to roll back (no destructive migration).
```

4. **Respond to every comment.** Fix it, or explain and agree. Don't silently ignore feedback.
5. **Split refactorings from behavior changes** into separate PRs (or at least separate commits).

---

## Interview questions

<details><summary><b>1. What is an ADR, and why keep them?</b></summary>

A short, immutable record of a significant decision: its context, the options, the decision and the consequences. It preserves the *why* for future team members, prevents re-litigating settled decisions, and documents trade-offs. A new ADR supersedes an old one, and old ones are never edited.
</details>

<details><summary><b>2. What goes into a good design doc?</b></summary>

The problem and context, goals and non-goals, the proposal (API, data, flows), the alternatives considered, risks, the rollout and rollback plan, observability, security, and open questions. It should be short enough that people actually read it.
</details>

<details><summary><b>3. What do you look for when reviewing a PR?</b></summary>

Correctness and edge cases, tests that would catch a regression, design fit, data and migration safety, security (especially authorization and IDOR), resilience, observability and contract compatibility. Formatting is left to automated tools.
</details>

<details><summary><b>4. How do you handle disagreement in code review?</b></summary>

Make the reasoning explicit, separate blocking issues from preferences, refer to agreed standards or ADRs, and talk synchronously if a thread goes back and forth more than twice. If it's a team-wide question, capture the outcome as a guideline or ADR.
</details>

---

## Exercises

1. Write three ADRs for decisions already made in your codebase at work: reverse-engineer the context and the consequences.
2. Write a design doc for a medium feature at work (or one from the labs) using the template, and ask a senior to review it.
3. Do 10 code reviews over the next weeks using the checklist and Conventional Comments. Track how many real issues you found.
4. Take your last large PR and split it, retrospectively, into 3–4 reviewable PRs. Write down how you'd sequence them.

## Self-check

- [ ] I can write an ADR and know when one is needed.
- [ ] I can write a design doc that covers rollout, observability and security, and run a review.
- [ ] I review with a checklist, label my comments, and turn reviews around within a day.
- [ ] My own PRs are small, self-reviewed and well described.

## Further reading

- Michael Nygard, *"Documenting Architecture Decisions"* (the original ADR post), and the MADR template project.
- Google Engineering Practices: *Code Review Developer Guide* (free online).
- conventionalcomments.org.
- Simon Brown, *The C4 model* (c4model.com) for diagrams.
