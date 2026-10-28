# 4. Database Performance at Scale

> Phase 5 · Weeks 6–8 · [← Phase index](README.md)

**Why this matters:** In most business applications, the database is the bottleneck, and the hardest component to scale. Phase 2 taught you indexes, plans and EF Core basics. Here you apply them **at 10–100× data volume**: finding the worst queries in production, pagination that stays fast at page 5,000, connection management and when to add replicas, caches or partitions.

---

## 4.1 Find the expensive queries first

### PostgreSQL: `pg_stat_statements`

```sql
-- Enable once (shared_preload_libraries = 'pg_stat_statements'), then:
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SELECT query,
       calls,
       round(total_exec_time::numeric, 0)  AS total_ms,
       round(mean_exec_time::numeric, 2)   AS mean_ms,
       rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC        -- total time = frequency × cost: the best target list
LIMIT 20;
```

### SQL Server: Query Store

Query Store records plans and runtime statistics per query. Use the "Top Resource Consuming Queries" and "Regressed Queries" reports, and **force the last good plan** when a plan regression hits production.

**Prioritize by total time** (calls × mean). A 5 ms query called 2 million times a day usually matters more than a 3 s report run twice.

---

## 4.2 Indexing strategy at scale

| Principle | Example |
|---|---|
| Index for the **access patterns**, not for every column | `(customer_id, created_at DESC)` serves "my latest orders" |
| **Composite order:** equality columns first, then range or sort columns | `WHERE status = 'Pending' AND created_at > …` → `(status, created_at)` |
| **Covering indexes** avoid table lookups | PostgreSQL `INCLUDE (total)`; SQL Server `INCLUDE` |
| **Partial / filtered indexes** for hot subsets | `CREATE INDEX … ON outbox_messages (occurred_at) WHERE processed_at IS NULL` |
| Every index slows writes and uses memory | drop unused indexes (`pg_stat_user_indexes.idx_scan = 0`) |
| Keep statistics current | autovacuum/analyze; update statistics after big loads |

---

## 4.3 Pagination that doesn't degrade

```csharp
// ❌ OFFSET pagination: page 5,000 reads and discards 100,000 rows first
var page = await db.Orders.OrderByDescending(o => o.CreatedAt).ThenByDescending(o => o.Id)
    .Skip((pageNumber - 1) * 20).Take(20).ToListAsync(ct);

// ✅ Keyset (seek) pagination: an index seek from the last seen key, constant cost at any depth
// (Id here is a bigint key used as the tiebreaker for equal timestamps)
var next = await db.Orders
    .Where(o => o.CreatedAt < cursor.CreatedAt || (o.CreatedAt == cursor.CreatedAt && o.Id < cursor.Id))
    .OrderByDescending(o => o.CreatedAt).ThenByDescending(o => o.Id)
    .Take(20)
    .Select(o => new OrderRow(o.Id, o.CreatedAt, o.Total))
    .ToListAsync(ct);
// Return an opaque cursor (base64 of CreatedAt+Id) to the client; index on (created_at DESC, id DESC)
```

Keyset pagination can't jump to "page 4,812", but feeds, exports and APIs rarely need that.

---

## 4.4 EF Core at scale

| Technique | When | How |
|---|---|---|
| **Projections** | always, for reads | `.Select(o => new Dto(...))`: only the needed columns, no tracking |
| `AsNoTracking()` | read-only queries | skips change-tracking cost |
| **Split queries** | `Include` of several collections (cartesian explosion) | `.AsSplitQuery()` |
| **Bulk operations** | updating or deleting many rows | `ExecuteUpdateAsync` / `ExecuteDeleteAsync`: one SQL statement, no loading |
| **Batching** | many inserts | EF batches automatically; tune `MaxBatchSize`; use `COPY` (Npgsql binary import) or `SqlBulkCopy` for huge loads |
| **Compiled queries** | very hot, simple queries | `EF.CompileAsyncQuery(...)` removes query-compilation overhead |
| **Compiled models** | large models with slow startup | `dotnet ef dbcontext optimize` |
| `DbContext` pooling | high request rates | `AddDbContextPool<T>()` |
| Raw SQL / Dapper | complex reporting, vendor-specific features | `FromSql`, `SqlQuery<T>`, Dapper, for the hot 5% |

```csharp
private static readonly Func<ShopDbContext, Guid, CancellationToken, Task<EventSummary?>> GetEventSummary =
    EF.CompileAsyncQuery((ShopDbContext db, Guid id, CancellationToken ct) =>
        db.Events.Where(e => e.Id == id)
                 .Select(e => new EventSummary(e.Id, e.Name, e.StartsAt))
                 .SingleOrDefault());

var summary = await GetEventSummary(db, id, ct);
```

---

## 4.5 Connections: the hidden bottleneck

- Each PostgreSQL connection is a **process** on the server; hundreds of them waste memory. SQL Server handles more, but pools still saturate.
- **Pool size × replicas ≤ what the database can handle.** Ten pods × 100 pool size = 1,000 potential connections.
- **Hold connections briefly:** short transactions, no external HTTP calls inside a transaction, async all the way.
- Use **PgBouncer** (transaction pooling) in front of PostgreSQL at scale (be aware of its limitations with prepared statements and session state), or Npgsql's multiplexing where it fits.
- Watch the pool: connection wait time and timeouts (Npgsql and SqlClient both emit metrics).

---

## 4.6 Scaling reads

| Technique | Trade-off |
|---|---|
| **Caching** (HybridCache, Redis; Phase 2) | staleness and invalidation complexity; the biggest win for read-heavy catalogs |
| **Read replicas** | replication lag means stale reads (Phase 3, guide 8). Route only lag-tolerant queries to them |
| **Materialized views / denormalized read tables** | refresh cost; eventual consistency. Great for dashboards and search pages |
| **A dedicated search engine** (Elasticsearch, OpenSearch, Azure AI Search) | another system to sync (via outbox events) |

```csharp
// Read-replica routing with two contexts: explicit and easy to reason about
builder.Services.AddDbContextPool<ShopDbContext>(o => o.UseNpgsql(cs.Primary));
builder.Services.AddDbContextPool<ShopReadDbContext>(o => o.UseNpgsql(cs.Replica)
    .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking));
```

---

## 4.7 Scaling writes and big tables

- **Partitioning** (PostgreSQL declarative partitioning, SQL Server partitioned tables): split huge tables by time or tenant. Queries that filter on the partition key touch only the relevant partitions, and old partitions can be dropped cheaply (data retention).
- **Archiving:** move cold data (orders older than two years) to cheaper storage.
- **Queue-based load leveling:** accept writes into a queue and process them at a sustainable rate (ticket-sale spikes).
- **Sharding** (by tenant or customer) is the last resort: cross-shard queries and transactions get hard. Managed options (Citus, Cosmos DB, Azure SQL Hyperscale) reduce the operational burden.

---

## 4.8 Concurrency at scale

- **Hot rows** (one counter row for "tickets sold") serialize all writers. Shard the counter (N rows, summed on read), or use queue-based allocation.
- Prefer **optimistic concurrency** with short retries for low-contention updates; use **pessimistic locks** (`FOR UPDATE`) briefly, and only for genuinely contended resources.
- **Deadlocks:** access rows and tables in a consistent order, keep transactions short, add the right indexes (scans lock more), and retry deadlock victims.

---

## Interview questions

<details><summary><b>1. How do you find which queries to optimize in production?</b></summary>

Use database-level statistics (`pg_stat_statements`, Query Store), sorted by total time (calls × mean). Correlate them with the APM traces of slow endpoints, then inspect the plans with `EXPLAIN ANALYZE` or the actual execution plans.
</details>

<details><summary><b>2. Why is OFFSET pagination slow on deep pages, and what's the alternative?</b></summary>

The database has to read and discard every skipped row, so the cost grows with the page number. Keyset pagination seeks from the last key using an index, so the cost is constant at any depth. It uses a cursor instead of page numbers.
</details>

<details><summary><b>3. Your API scales to 30 pods and the database falls over. Why?</b></summary>

Likely connection exhaustion (pool size × pods), plus more concurrent queries than the database can handle. Cap the pools, add a connection pooler, shorten transactions, cache reads, move lag-tolerant reads to replicas, and level writes through a queue. Scaling the stateless tier doesn't scale the database.
</details>

<details><summary><b>4. When would you use a read replica, and what's the risk?</b></summary>

For read-heavy, lag-tolerant queries (reports, catalog browsing). The risk is replication lag, which causes stale reads and read-your-writes violations. Keep user-specific "just written" reads on the primary.
</details>

---

## Exercises

1. Load your lab database with 10× more data (millions of rows). Find the top 5 queries by total time, and fix them with indexes, projections or keyset pagination ([Lab 4](11-labs.md#lab-4-10-data-volume)).
2. Replace an `OFFSET` endpoint with keyset pagination and measure page 1 vs page 5,000 before and after.
3. Compare a compiled query with a regular one in BenchmarkDotNet against a real PostgreSQL (Testcontainers).
4. Simulate connection exhaustion (a small pool, a slow query, many concurrent requests), observe the errors, and fix it.

## Self-check

- [ ] I can find the most expensive queries with database statistics.
- [ ] I design indexes for access patterns and use covering and partial indexes.
- [ ] I use keyset pagination, projections, bulk operations and compiled queries appropriately.
- [ ] I understand connection-pool limits and how to scale reads and writes.

## Further reading

- *Use The Index, Luke* (use-the-index-luke.com) by Markus Winand.
- PostgreSQL docs: *pg_stat_statements*, *Table Partitioning*, *Using EXPLAIN*.
- Microsoft Learn: *EF Core performance* (efficient querying, efficient updating, advanced topics).
