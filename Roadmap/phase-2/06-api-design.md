# 6. API Design: Resources, Status Codes, Pagination, Idempotency, Versioning and Concurrency

> Phase 2 · Week 9 · [← Phase index](README.md)

**Why this matters:** You can refactor code, but a published API contract is used by other teams, mobile apps and partners for years. Good API design reduces support load, prevents data corruption (with idempotency and concurrency control) and lets you evolve the API without breaking clients.

---

## 6.1 Resource modeling

| ✅ Do | ❌ Avoid |
|---|---|
| Nouns for resources: `/api/events`, `/api/events/{id}/seats`, `/api/orders/{id}` | verbs in paths: `/api/getEvents`, `/api/createOrder` |
| Plural collections | mixing `/event/1` and `/events` |
| Nesting only for real containment (one level, two at most) | `/api/venues/1/events/2/seats/3/holds/4` |
| **Actions as sub-resources** when CRUD doesn't fit: `POST /api/holds/{id}/checkout`, `POST /api/events/{id}/publish` | `PUT /api/events/1 { "status": "Published" }` for business transitions with rules |
| Consistent casing (camelCase JSON) and ISO 8601 UTC times (`2027-03-01T19:30:00Z`) | mixed formats and local times without an offset |

**TicketHub resources:**

```
GET    /api/events?from=2027-03-01&city=Berlin&page=1     list (public)
GET    /api/events/{id}                                   detail (public)
GET    /api/events/{id}/seats?status=Available            the seat map (public)
POST   /api/events                                        create (organizer)
POST   /api/events/{id}/publish                           state transition (organizer)
POST   /api/events/{id}/holds                             hold seats (customer) → 201 + Location: /api/holds/{holdId}
GET    /api/holds/{id}                                    the hold and its expiry (owner)
DELETE /api/holds/{id}                                    release a hold (owner)
POST   /api/holds/{id}/checkout                           create an order from the hold (owner, Idempotency-Key)
GET    /api/me/orders?cursor=...                          my orders (owner)
```

---

## 6.2 HTTP methods and their semantics

| Method | Safe (no side effects) | Idempotent (repeating it has the same effect) | Typical success response |
|---|---|---|---|
| GET | ✅ | ✅ | 200 |
| HEAD / OPTIONS | ✅ | ✅ | 200 / 204 |
| PUT (full replace) | ❌ | ✅ | 200 / 204 |
| DELETE | ❌ | ✅ (deleting twice → the resource is still gone; 404 or 204 on the second call) | 204 |
| PATCH (partial) | ❌ | ❌ by definition (it can be made idempotent) | 200 / 204 |
| POST (create or action) | ❌ | ❌ → **make it idempotent with idempotency keys** | 201 + `Location` / 200 / 202 |

**Retries happen everywhere:** mobile networks, proxies, Polly policies and users double-clicking. Any non-idempotent operation that moves money or inventory **must** be made idempotent.

---

## 6.3 Pagination, filtering and sorting

**Offset pagination**, for small or admin lists:

```http
GET /api/events?page=2&pageSize=20&sort=startsAt
```

```json
{ "items": [ ... ], "page": 2, "pageSize": 20, "totalCount": 431 }
```

**Cursor (keyset) pagination**, for large or growing lists:

```http
GET /api/me/orders?limit=20&cursor=eyJjcmVhdGVkQXQiOiIyMDI3LTAzLTAxVDE5OjMwOjAwWiIsImlkIjoiLi4uIn0
```

```json
{ "items": [ ... ], "nextCursor": "eyJjcmVhdGVkQXQiOi..." }   // null when there are no more pages
```

```csharp
public sealed record OrderCursor(DateTimeOffset CreatedAt, Guid Id)
{
    public string Encode() => Base64Url.EncodeToString(JsonSerializer.SerializeToUtf8Bytes(this));
    public static OrderCursor? Decode(string? value) =>
        string.IsNullOrEmpty(value) ? null : JsonSerializer.Deserialize<OrderCursor>(Base64Url.DecodeFromChars(value));
}

var query = db.Orders.AsNoTracking().Where(o => o.CustomerId == userId);
if (OrderCursor.Decode(cursor) is { } c)
    query = query.Where(o => EF.Functions.LessThan(               // Npgsql row-value comparison: (created_at, id) < (@c1, @c2)
        ValueTuple.Create(o.CreatedAt, o.Id),
        ValueTuple.Create(c.CreatedAt, c.Id)));

var page = await query
    .OrderByDescending(o => o.CreatedAt).ThenByDescending(o => o.Id)
    .Take(limit + 1)                                   // fetch one extra row to know whether there's a next page
    .Select(o => new MyOrderDto(o.Id, o.Total, o.CreatedAt, o.Status))
    .ToListAsync(ct);

string? next = page.Count > limit ? new OrderCursor(page[limit - 1].CreatedAt, page[limit - 1].Id).Encode() : null;
return TypedResults.Ok(new CursorPage<MyOrderDto>(page.Take(limit).ToList(), next));
```

> `Base64Url` lives in `System.Buffers.Text` (.NET 9+). Treat cursors as **opaque**: clients must not parse them, so you can change the format later. The row-value comparison is Npgsql-specific. On other providers, write `created_at < @c OR (created_at = @c AND <tie-breaker> < @id)` with a tie-breaker the provider can compare, such as a `bigint` identity column.

**Rules:**

- **Cap `pageSize`** (for example at 100), or a client will ask for a million rows.
- **Whitelist** the sort and filter fields. Never pass a user-supplied field name into SQL.
- Make the ordering deterministic with a tie-breaker (`id`).

---

## 6.4 Idempotency keys

The client generates a unique key per *logical operation* and sends it with the request. If the request is retried, the server returns the **original result** instead of doing the work again.

```http
POST /api/holds/7b0e.../checkout
Idempotency-Key: 5f1c3a9e-2d8b-4c1e-9a7f-1b2c3d4e5f60
```

**Algorithm** (store the keys in your database, in the same transaction as the work):

1. Look up `(customer_id, idempotency_key)`.
   - If found and **completed**, return the stored status code and body.
   - If found and **in progress**, return **409** ("request in progress").
   - If the stored **request hash differs**, return **422**: the same key was reused with a different payload.
2. Insert the key with status *in progress*. A **unique constraint** on `(customer_id, key)` resolves races between duplicates.
3. Do the work. Store the response (status and body) and mark the key *completed*, **in the same transaction** as the work.
4. Expire keys after, for example, 24 hours.

```sql
CREATE TABLE idempotency_keys (
    customer_id   uuid         NOT NULL,
    key           varchar(64)  NOT NULL,
    request_hash  char(64)     NOT NULL,
    status_code   int          NULL,
    response_body jsonb        NULL,
    created_at    timestamptz  NOT NULL,
    PRIMARY KEY (customer_id, key)
);
```

> Stripe's API documentation on idempotent requests is the classic reference. Read it.

---

## 6.5 Optimistic concurrency over HTTP: ETags

This prevents lost updates *between clients*, for example two organizers editing the same event:

```http
GET /api/events/42            → 200, ETag: "W/\"1876\""         (the row version)
PUT /api/events/42            If-Match: "W/\"1876\""             → 200, a new ETag
PUT /api/events/42            If-Match: "W/\"1876\""  (stale)    → 412 Precondition Failed
PUT /api/events/42            (no If-Match on an endpoint that requires one) → 428 Precondition Required
```

Implementation: map the `If-Match` value to the entity's concurrency token (`xmin`/`rowversion`) and let EF's concurrency check do the work. Turn `DbUpdateConcurrencyException` into a 412.

---

## 6.6 Long-running operations

For work that takes more than a few seconds (report generation, bulk imports):

```http
POST /api/reports        → 202 Accepted, Location: /api/reports/jobs/91
GET  /api/reports/jobs/91 → 200 { "status": "Running", "progress": 40 }
GET  /api/reports/jobs/91 → 303 See Other, Location: /api/reports/91   (when it's done)
```

Do the work in a background worker or queue (Phase 3), not inside the request.

---

## 6.7 Versioning and evolution

**Additive changes are not breaking:** a new optional request field, a new response field, or a new endpoint. Clients must **ignore unknown fields** (System.Text.Json does by default).

**Breaking changes:** removing or renaming fields, changing types or meanings, a new *required* input, or changed status codes.

| Strategy | Example | Notes |
|---|---|---|
| URL segment | `/api/v2/events` | the most explicit and cache-friendly; the most common choice |
| Query string | `?api-version=2.0` | Azure-style |
| Header | `Api-Version: 2.0` | cleaner URLs, harder to test in a browser |

```csharp
// Package: Asp.Versioning.Http
builder.Services.AddApiVersioning(o =>
{
    o.DefaultApiVersion = new ApiVersion(1, 0);
    o.ReportApiVersions = true;                       // api-supported-versions response header
    o.ApiVersionReader = new UrlSegmentApiVersionReader();
});

var versions = app.NewApiVersionSet().HasApiVersion(new ApiVersion(1)).HasApiVersion(new ApiVersion(2)).Build();
app.MapGroup("/api/v{version:apiVersion}/events").WithApiVersionSet(versions);
```

**Deprecation:** announce it, add `Deprecation` and `Sunset` response headers, monitor usage, then remove. **Prefer evolving one version over creating new ones.**

---

## 6.8 API design review checklist

- [ ] Resources are nouns; actions are explicit sub-resources; nesting is at most 2 levels deep.
- [ ] Correct methods and status codes; `201` + `Location` on create.
- [ ] Every error is ProblemDetails with a stable `code`.
- [ ] Pagination is capped, ordering is deterministic, and sort/filter fields are whitelisted.
- [ ] Money and inventory operations take an `Idempotency-Key`.
- [ ] Concurrent edits are protected with ETags or version checks.
- [ ] Times are UTC ISO 8601; money uses `decimal` plus a currency, never `double`.
- [ ] The OpenAPI document is accurate; breaking changes are detected in CI.
- [ ] The authorization rules are documented for each endpoint.

---

## Interview questions

<details><summary><b>1. PUT vs PATCH vs POST?</b></summary>

PUT replaces the resource fully and is idempotent. PATCH partially updates it and isn't idempotent by definition. POST creates a resource or performs an action and isn't idempotent. Make POST idempotent with idempotency keys when it matters.
</details>

<details><summary><b>2. How do you implement an idempotency key?</b></summary>

Store `(caller, key, request hash, response)` with a unique constraint. On a repeat, return the stored response. If the payload differs, reject with 422. If the original is still in progress, return 409. Record the key in the same transaction as the work, and expire keys after a while.
</details>

<details><summary><b>3. Offset vs cursor pagination?</b></summary>

Offset is simple and supports random access, but it gets slower with depth and is unstable under inserts. A cursor (keyset) is constant cost and stable, and is best for feeds and large tables, but it has no random access. Cap the page size and use a deterministic order either way.
</details>

<details><summary><b>4. What is a breaking change? How do you version an API?</b></summary>

A breaking change removes or renames fields, changes types or semantics, adds required inputs or changes status codes. Additive changes aren't breaking. Version through the URL segment (most common), the query string or a header. Deprecate with `Deprecation`/`Sunset` headers and usage monitoring.
</details>

<details><summary><b>5. How do ETags prevent lost updates?</b></summary>

The server returns a version tag. The client sends it back in `If-Match`, and the server applies the update only if the version still matches. Otherwise it returns 412, and the client must re-read and merge.
</details>

---

## Exercises

1. Design the full TicketHub API surface on paper first: resources, methods, status codes and errors. Review it against the checklist with a colleague.
2. Implement cursor pagination for `/api/me/orders` and test it with 10,000 orders: no duplicates and no gaps.
3. Implement idempotent checkout and prove with a test that sending the same key twice creates one order and returns the same body.
4. Add ETag/`If-Match` to `PUT /api/events/{id}` and test both the 412 and the 428 paths.

## Self-check

- [ ] I can model resources and choose methods and status codes confidently.
- [ ] I implemented cursor pagination and idempotency keys.
- [ ] I know the strategies for versioning and evolving an API without breaking clients.

## Further reading

- *API Design Patterns* (JJ Geewax) or the Microsoft REST API Guidelines (GitHub: `microsoft/api-guidelines`).
- Stripe API docs: *Idempotent requests*, *Pagination*.
- RFC 9110 (HTTP semantics), the sections on methods and status codes.
