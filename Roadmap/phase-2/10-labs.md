# 10. Labs

> Phase 2 · All weeks · [← Phase index](README.md)

These labs make the theory visible. Keep each lab in your `dotnet-levelup/labs` repository with a short `NOTES.md`: what you did, what you measured, what surprised you, and the rule you'll apply at work. The code examples in the guides use a ticket-booking domain (events, seats, holds, orders). Reuse it in your lab projects, or substitute your own domain.

| Lab | Guide | Time |
|---|---|---|
| [Lab 1: Pipeline order](#lab-1-pipeline-order) | [01](01-aspnetcore-pipeline-and-endpoints.md) | 2 h |
| [Lab 2: Errors and OpenAPI](#lab-2-errors-and-openapi) | [02](02-validation-errors-openapi.md) | 2 h |
| [Lab 3: SQL plans, isolation and deadlocks](#lab-3-sql-plans-isolation-and-deadlocks) | [03](03-sql-fundamentals.md) | 6 h |
| [Lab 4: N+1 hunt](#lab-4-n1-hunt) | [04](04-ef-core.md) | 3 h |
| [Lab 5: Optimistic concurrency race](#lab-5-optimistic-concurrency-race) | [04](04-ef-core.md), [09](09-integration-testing.md) | 3 h |
| [Lab 6: Token flow](#lab-6-token-flow) | [05](05-auth.md) | 3 h |
| [Lab 7: Idempotent POST](#lab-7-idempotent-post) | [06](06-api-design.md) | 2 h |
| [Lab 8: Rate limiting and health checks](#lab-8-rate-limiting-and-health-checks) | [07](07-cross-cutting-and-realtime.md) | 2 h |
| [Lab 9: Cache stampede](#lab-9-cache-stampede) | [08](08-caching.md) | 2 h |
| [Lab 10: Work, find and fix a slow query](#lab-10-work-find-and-fix-a-slow-query) | all | ongoing |

**Shared local infrastructure** (`labs/docker-compose.yml`):

```yaml
services:
  postgres:
    image: postgres:17
    environment:
      POSTGRES_USER: lab
      POSTGRES_PASSWORD: lab          # local development only
      POSTGRES_DB: lab
    ports: ["5432:5432"]
  redis:
    image: redis:7
    ports: ["6379:6379"]
  keycloak:
    image: quay.io/keycloak/keycloak:26.0
    command: start-dev
    environment:
      KC_BOOTSTRAP_ADMIN_USERNAME: admin
      KC_BOOTSTRAP_ADMIN_PASSWORD: admin   # local development only
    ports: ["8080:8080"]
```

> Use the newest stable image tags when you start; the versions above are just a known baseline.

---

## Lab 1: Pipeline order

1. Create `dotnet new web -n PipelineLab -f net10.0`. Add `UseExceptionHandler`, `UseCors`, `UseAuthentication`, `UseAuthorization`, and an endpoint that throws.
2. **Break it on purpose**, one change at a time, and write down the symptom:
   - Put `UseCors` *after* `UseAuthorization`. Call the API from a browser page on another origin and look at the preflight failure in the browser's network tab.
   - Put `UseExceptionHandler` *last*. Throw from an endpoint, and see that the response isn't a `ProblemDetails`.
   - Remove `UseAuthentication` and keep `[Authorize]`. Watch every request get 401, even with a valid token.
3. Add a tiny custom middleware that logs `context.Request.Path` and the elapsed time, and place it first. Explain why "first" means it also sees the time spent in everything after it.

**Deliverable:** a table of "misordering → symptom → fix".

---

## Lab 2: Errors and OpenAPI

1. Add global exception handling with `IExceptionHandler` and `AddProblemDetails`.
2. Add validation to a `POST` endpoint and confirm that invalid input returns `400` with a `ValidationProblemDetails` body listing every field error.
3. Make sure a `500` response contains a `traceId` but **no stack trace** when running with `ASPNETCORE_ENVIRONMENT=Production`.
4. Add OpenAPI plus Scalar, then annotate your endpoints with `Produces`/`ProducesProblem` until every response code appears in the generated document.

**Deliverable:** screenshots or saved responses for the 400, 404, 409 and 500 cases.

---

## Lab 3: SQL plans, isolation and deadlocks

1. Create an `orders` table and generate **500,000+ rows** with `generate_series` (PostgreSQL) so that plans are realistic.
2. Write three queries: orders for one customer, sorted by date, with paging; total revenue per event for last month; and a search by email prefix.
3. For each one: capture `EXPLAIN (ANALYZE, BUFFERS)` → add or change an index → capture it again. Record rows read, timings and plan nodes (Seq Scan vs Index Scan vs Index Only Scan).
4. **Lost update:** in two `psql` sessions, both read a seat's status and both write "Held". Show that both "succeed". Then fix it with a conditional `UPDATE … WHERE status = 'Available'` (check the affected row count) and with `SELECT … FOR UPDATE`.
5. **Deadlock:** in two sessions, lock rows A→B and B→A in opposite order. Capture PostgreSQL's deadlock error and explain how consistent lock ordering prevents it.

**Deliverable:** a before/after table for the three queries and a written explanation of the lost-update and deadlock experiments.

---

## Lab 4: N+1 hunt

1. With EF Core and PostgreSQL, model `Customer → Orders → OrderLines`. Seed 200 customers with about 10 orders each.
2. Implement "orders with their lines" three ways: lazy loading or a loop of queries (N+1), `Include`/`ThenInclude`, and a projection with `Select` into DTOs.
3. Turn on SQL logging (`LogTo(Console.WriteLine, LogLevel.Information)`) and count the queries. Time each version with `Stopwatch` across 10 runs.
4. Try `AsSplitQuery()` with the `Include` version and compare the SQL and the timings.

**Deliverable:** a table of query count, rows transferred and time per approach, plus the rule you'll use at work.

---

## Lab 5: Optimistic concurrency race

1. Add a concurrency token (`xmin` on PostgreSQL or `rowversion` on SQL Server) to a `Seat` entity.
2. Write an integration test with Testcontainers that fires **20 concurrent** "hold seat A1" requests through `WebApplicationFactory` and asserts that **exactly one** succeeds and the rest get `409 Conflict`.
3. Run it in a loop 50 times. Then remove the concurrency token and watch the test fail (two holds succeed). Put it back.

**Deliverable:** the test, and a short note on why in-memory providers can't prove this.

---

## Lab 6: Token flow

1. In Keycloak, create a realm, a public client with PKCE, an API audience and a test user with a role.
2. Use Bruno or Postman's OAuth 2.0 helper to get a token with the authorization code + PKCE flow. Decode it on a **local** tool and identify `iss`, `aud`, `exp`, `sub` and the role claims.
3. Call your API with the token. Then change one thing at a time (expire it, change the audience, remove the role) and record the 401 vs 403 you get and why.
4. Implement resource-based authorization: a user can read only **their own** orders. Prove that user A gets `404` (or `403`, per your API design) for user B's order ID.

**Deliverable:** a sequence diagram of the PKCE flow in your own words, plus the 401/403 table.

---

## Lab 7: Idempotent POST

1. Implement `POST /orders` that requires an `Idempotency-Key` header.
2. Store the key, a request hash and the response in a table with a **unique constraint** on the key.
3. Send the same request twice → the same response, one order. Send the same key with a different body → `422` (or `409`). Send 10 concurrent identical requests → still exactly one order.

**Deliverable:** the endpoint, the tests and a note on how long you keep keys and why.

---

## Lab 8: Rate limiting and health checks

1. Add a fixed-window rate limiter partitioned by user (or by IP for anonymous users) to one endpoint. Hammer it with the Phase 1 LoadGen tool and check for `429` responses and a `Retry-After` header.
2. Add `/health/live` (process only) and `/health/ready` (database plus Redis). Stop the Postgres container and check that *ready* fails while *live* still passes. Explain why an orchestrator needs both.

**Deliverable:** the configuration and the observed responses.

---

## Lab 9: Cache stampede

1. Put a slow (500 ms) database read behind `HybridCache` with Redis as L2.
2. Fire **100 concurrent requests** for the same key on a cold cache. Count the database calls (log them, or use a counter). HybridCache should run the factory **once**.
3. Repeat with a naive `IDistributedCache` get-then-set and count again.
4. Invalidate by tag after an update and confirm the next read is fresh.

**Deliverable:** the counts for both approaches and the invalidation result.

---

## Lab 10: Work, find and fix a slow query

1. Find a slow endpoint at work (APM, logs or user complaints).
2. Capture its SQL and execution plan. Fix the N+1 query, add the missing index or project fewer columns.
3. Measure before and after, in production or a production-like environment, and record the numbers in your brag document.

> This is the most valuable lab in Phase 2, because it becomes a real interview story: *"I cut p95 latency of X from 1.2 s to 90 ms by…"*
