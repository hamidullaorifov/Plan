# Phase 2: ASP.NET Core 10 and Data

> **Months 3–6 (Feb 1 – Apr 30, 2027)**
> **Goal:** build a production-grade API that you can defend line by line: HTTP, SQL, EF Core, auth, caching and tests.
> ← [Main roadmap](../level-up.md) · [Phase 1](../phase-1/README.md) · [Phase 3 →](../phase-3/README.md)

---

## How to study

Use the same loop as in Phase 1: **Read → Run → Explain → Check → Apply**. One extra rule applies in this phase: **look at the wire.** For every topic, inspect the actual HTTP traffic (with `.http` files or Bruno/Postman) and the actual SQL (with EF logging and `EXPLAIN`). Juniors guess what the framework does. Middle developers check.

---

## Topics

| # | Guide | Weeks | Effort |
|---|---|---|---|
| 1 | [Request pipeline, routing, Minimal APIs and controllers](01-aspnetcore-pipeline-and-endpoints.md) | W1 | 6–8 h |
| 2 | [Validation, errors (ProblemDetails) and OpenAPI](02-validation-errors-openapi.md) | W2 | 5–6 h |
| 3 | [SQL fundamentals: indexes, plans, transactions](03-sql-fundamentals.md) | W3–W4 | 10–12 h |
| 4 | [EF Core 10 in depth (plus Dapper)](04-ef-core.md) | W5–W6 | 12–14 h |
| 5 | [Authentication and authorization: JWT, OAuth2/OIDC, policies](05-auth.md) | W7–W8 | 10–12 h |
| 6 | [API design: resources, status codes, pagination, idempotency, versioning, concurrency](06-api-design.md) | W9 | 5–6 h |
| 7 | [Cross-cutting concerns and real time: rate limiting, CORS, health, HttpClient, SSE, SignalR, gRPC](07-cross-cutting-and-realtime.md) | W10 | 6–8 h |
| 8 | [Caching: output caching, HybridCache, Redis](08-caching.md) | W11 | 5–6 h |
| 9 | [Integration testing: WebApplicationFactory, Testcontainers, Respawn](09-integration-testing.md) | W12 | 6–8 h |
| 10 | [Labs](10-labs.md) | All weeks | ~30 h |

---

## Week-by-week plan

| Week | Dates | Theory (Mon/Wed/Thu) | Tue: algorithms | Sat: labs and practice |
|---|---|---|---|---|
| W1 | Feb 1 | 01 Pipeline and endpoints | Trees ×2 | Lab 1: pipeline order |
| W2 | Feb 8 | 02 Validation, errors, OpenAPI | Trees ×2 | Lab 2: errors and OpenAPI |
| W3 | Feb 15 | 03 SQL, sections 1–5 | Trees ×2 | Lab 3: plans and indexes |
| W4 | Feb 22 | 03 SQL, sections 6–9 | Tries/heaps ×2 | Lab 3: isolation and deadlocks |
| W5 | Mar 1 | 04 EF Core, sections 1–6 | Heaps ×2 | Lab 4: N+1 hunt |
| W6 | Mar 8 | 04 EF Core, sections 7–12 | Backtracking ×2 | Lab 5: optimistic concurrency race |
| W7 | Mar 15 | 05 Auth, sections 1–5 | Backtracking ×2 | Lab 6: token flow (Keycloak + PKCE) |
| W8 | Mar 22 | 05 Auth, sections 6–10 | Graphs ×2 | Lab 6: resource-based authorization |
| W9 | Mar 29 | 06 API design | Graphs ×2 | Lab 7: idempotent POST |
| W10 | Apr 5 | 07 Cross-cutting | Graphs ×2 | Lab 8: rate limiting and health checks |
| W11 | Apr 12 | 08 Caching | 1-D DP ×2 | Lab 9: cache stampede |
| W12 | Apr 19 | 09 Integration testing | 1-D DP ×2 | Testcontainers suite for your lab API |
| W13 | Apr 26 | Review the self-checks | Mixed ×3 | Lab write-ups, exit check |

> **At work during this phase:** ship one non-trivial feature end to end, and find and fix one slow query with before/after numbers.

---

## Toolbox additions

| Tool | Purpose |
|---|---|
| Docker Desktop or Podman | Postgres, Redis and Keycloak locally; Testcontainers |
| `dotnet tool install -g dotnet-ef` | EF Core migrations |
| A database client: DBeaver, Azure Data Studio or `psql` | Run SQL and read `EXPLAIN` output |
| `.http` files (VS / VS Code / Rider) or Bruno | Request collections stored in Git |
| <https://jwt.io> or `jwt.ms` (on **test** tokens only) | Inspect JWTs. Never paste production tokens into a website |

---

## Phase 2 exit criteria

- [ ] I can explain where authentication, CORS and exception handling go in the pipeline, and why the order matters.
- [ ] I can read an execution plan and explain why a query does a scan.
- [ ] I can explain OAuth2 authorization code + PKCE step by step.
- [ ] A repeated concurrent integration test proves that an update is race-safe (Lab 5).
- [ ] I shipped at least one non-trivial feature at work end to end: design, PR, deploy and verification in production.
- [ ] All labs have a `NOTES.md` with measurements and conclusions.
