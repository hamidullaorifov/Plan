# 3. SQL Fundamentals: Indexes, Execution Plans and Transactions

> Phase 2 · Weeks 3–4 · [← Phase index](README.md)
> Examples are for **PostgreSQL** (TicketHub) with **SQL Server** equivalents, because many .NET jobs use SQL Server.

**Why this matters:** Most "the API is slow" incidents are database problems: a missing index, an N+1 query, a lock wait or a bad plan. An ORM doesn't remove the need to understand SQL. It makes it *easier to write bad SQL without noticing*.

---

## 3.1 A practice schema

```sql
-- PostgreSQL
CREATE TABLE events (
    id          uuid PRIMARY KEY,
    name        varchar(200) NOT NULL,
    venue_name  varchar(200) NOT NULL,
    starts_at   timestamptz  NOT NULL,
    status      varchar(16)  NOT NULL,
    created_at  timestamptz  NOT NULL DEFAULT now()
);

CREATE TABLE event_seats (
    event_id         uuid        NOT NULL REFERENCES events(id),
    code             varchar(32) NOT NULL,
    category         varchar(16) NOT NULL,
    status           varchar(16) NOT NULL,           -- Available | Held | Sold
    hold_id          uuid        NULL,
    hold_expires_at  timestamptz NULL,
    PRIMARY KEY (event_id, code)
);

CREATE TABLE orders (
    id           uuid PRIMARY KEY,
    customer_id  uuid           NOT NULL,
    event_id     uuid           NOT NULL REFERENCES events(id),
    total        numeric(10,2)  NOT NULL,
    status       varchar(16)    NOT NULL,
    created_at   timestamptz    NOT NULL
);
```

Generate realistic volume for the exercises (for example 1,000 events × 2,000 seats and 500,000 orders), using `generate_series` in PostgreSQL or a small C# seeder. **Plans only become interesting with data.**

---

## 3.2 Joins, aggregation, CTEs and window functions

```sql
-- Revenue per event in the last 30 days, top 10
SELECT e.name, SUM(o.total) AS revenue, COUNT(*) AS orders
FROM orders o
JOIN events e ON e.id = o.event_id
WHERE o.status = 'Paid' AND o.created_at >= now() - interval '30 days'
GROUP BY e.name
ORDER BY revenue DESC
LIMIT 10;                                   -- SQL Server: SELECT TOP (10) ...

-- Events with no orders: LEFT JOIN plus an IS NULL check (an "anti-join")
SELECT e.id, e.name
FROM events e
LEFT JOIN orders o ON o.event_id = e.id
WHERE o.id IS NULL;

-- Window functions: rank each customer's orders and compute a running total
SELECT customer_id, id, total,
       ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC)  AS recency_rank,
       SUM(total)   OVER (PARTITION BY customer_id ORDER BY created_at)       AS running_total
FROM orders;

-- CTE: the latest order per customer
WITH ranked AS (
    SELECT o.*, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC) AS rn
    FROM orders o
)
SELECT * FROM ranked WHERE rn = 1;
```

| Join | Returns |
|---|---|
| `INNER JOIN` | only matching rows |
| `LEFT JOIN` | all left rows; the right side is NULL when there's no match |
| `RIGHT JOIN` / `FULL JOIN` | the mirror image / everything from both sides |
| `CROSS JOIN` | the Cartesian product (rarely what you want) |

> `WHERE` filters rows *before* grouping, and `HAVING` filters groups *after* it. `COUNT(*)` counts rows, while `COUNT(col)` skips NULLs.

---

## 3.3 How indexes work

A **B-tree index** is a sorted structure. It turns "scan every row" (O(n)) into "walk the tree" (O(log n)) for equality and range predicates, and it can return rows **already sorted**.

| Concept | PostgreSQL | SQL Server |
|---|---|---|
| Table storage | a **heap**; every index is secondary | usually a **clustered index**: the table *is* the B-tree, ordered by the clustering key (by default the PK) |
| Secondary index | `CREATE INDEX` | non-clustered index (points to the clustering key) |
| Covering index | `CREATE INDEX ... INCLUDE (cols)` → **index-only scan** | `INCLUDE (cols)` → avoids a **key lookup** |
| Filtered or partial index | `CREATE INDEX ... WHERE status = 'Held'` | `CREATE INDEX ... WHERE status = 'Held'` |

### Composite index column order

The index on `(event_id, status)` is sorted by `event_id` first, then `status` within each `event_id`.

| Query predicate | Can it use `(event_id, status)`? |
|---|---|
| `event_id = X` | ✅ (leftmost prefix) |
| `event_id = X AND status = 'Available'` | ✅ the best case |
| `status = 'Available'` alone | ❌ usually not; the leftmost column is missing |
| `event_id = X ORDER BY status` | ✅ rows come back already ordered |

**Rule of thumb:** put **equality columns first**, then **range or sort columns**. For example, `(customer_id, created_at DESC)` serves "a customer's latest orders".

```sql
-- Serves: WHERE customer_id = $1 ORDER BY created_at DESC LIMIT 20
CREATE INDEX ix_orders_customer_created ON orders (customer_id, created_at DESC) INCLUDE (total, status);

-- Serves the expired-holds cleanup job, and stays tiny because only held seats are indexed
CREATE INDEX ix_event_seats_hold_expiry ON event_seats (hold_expires_at) WHERE status = 'Held';
```

### Indexes aren't free

- Every `INSERT`, `UPDATE` and `DELETE` must update every index. Write-heavy tables need fewer indexes.
- Indexes use memory and disk, so unused indexes are pure cost. Find them with `pg_stat_user_indexes` or `sys.dm_db_index_usage_stats`.
- **Foreign keys aren't indexed automatically** in PostgreSQL or SQL Server. Index FK columns that you join or filter on.

### Sargability: why an index gets ignored

| ❌ Non-sargable (can't seek) | ✅ Sargable rewrite |
|---|---|
| `WHERE date(created_at) = '2027-03-01'` | `WHERE created_at >= '2027-03-01' AND created_at < '2027-03-02'` |
| `WHERE lower(email) = 'ann@x.com'` | an expression index on `lower(email)`, or a case-insensitive collation |
| `WHERE name LIKE '%jazz%'` | full-text search (`tsvector`) or a trigram index (`pg_trgm`) |
| `WHERE total + 0 > 100` | `WHERE total > 100` |
| comparing `varchar` to `nvarchar` (SQL Server implicit conversion) | match the parameter types to the column type |

---

## 3.4 Reading execution plans

```sql
-- PostgreSQL: actual run with timing and buffer usage
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, total FROM orders WHERE customer_id = '…' ORDER BY created_at DESC LIMIT 20;
```

```
-- SQL Server: in SSMS / Azure Data Studio, enable "Include Actual Execution Plan", or:
SET STATISTICS IO, TIME ON;
```

| Node (PostgreSQL) | Meaning | Good or bad? |
|---|---|---|
| `Seq Scan` | reads the whole table | fine for small tables or when most rows are needed; bad for selective queries |
| `Index Scan` | walks the index, then fetches the rows | good for selective queries |
| `Index Only Scan` | everything comes from the index (covering) | best |
| `Bitmap Heap Scan` | collects matches from an index, then reads pages in order | good for medium selectivity |
| `Nested Loop` | for each outer row, look up the inner side | great when the outer side is small *and* the inner side is indexed |
| `Hash Join` | builds a hash table from one side | good for large unsorted inputs |
| `Sort` with `external merge` | the sort spilled to disk | add an index for that order, or increase `work_mem` |

**What to look at, in order:**

1. **Actual vs estimated rows.** A large mismatch means stale statistics or a hard-to-estimate predicate. Run `ANALYZE` (PostgreSQL) or `UPDATE STATISTICS` (SQL Server).
2. **The node with the highest actual time.** Optimize that one.
3. **Rows removed by the filter.** Reading 1 million rows to return 20 means a missing or wrong index.
4. **Buffers** (pages read). This measures I/O, which is more stable than timing.

SQL Server equivalents: *Clustered Index Scan* means a table scan; *Index Seek* is good; *Key Lookup* means the index isn't covering (add `INCLUDE` columns).

---

## 3.5 Pagination: OFFSET vs keyset

```sql
-- OFFSET: simple, but page 5,000 still reads and discards 100,000 rows
SELECT * FROM orders WHERE customer_id = $1 ORDER BY created_at DESC, id DESC OFFSET 100000 LIMIT 20;

-- Keyset (seek): constant cost regardless of depth; the client sends the last (created_at, id) it saw
SELECT * FROM orders
WHERE customer_id = $1 AND (created_at, id) < ($2, $3)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

Use offset pagination for admin screens where the total is small, and keyset (cursor) pagination for feeds and large tables. API design for both is covered in [guide 6](06-api-design.md).

---

## 3.6 Transactions and ACID

| Property | Meaning |
|---|---|
| **A**tomicity | all statements commit, or none do |
| **C**onsistency | constraints (PK, FK, CHECK, UNIQUE) always hold |
| **I**solation | concurrent transactions don't see each other's partial work (to a degree set by the **isolation level**) |
| **D**urability | once committed, the data survives a crash (thanks to the WAL / transaction log) |

### Isolation levels and anomalies

| Level | Dirty read | Non-repeatable read | Phantom | Lost update / write skew |
|---|---|---|---|---|
| Read Uncommitted | possible (SQL Server only; PostgreSQL treats it as Read Committed) | possible | possible | possible |
| **Read Committed** (default in both) | ❌ | possible | possible | possible |
| Repeatable Read | ❌ | ❌ | possible in SQL Server; ❌ in PostgreSQL | PostgreSQL: lost updates are detected (serialization error) |
| Serializable | ❌ | ❌ | ❌ | ❌ (PostgreSQL SSI aborts conflicting transactions, so you **must retry**) |

- **PostgreSQL** uses **MVCC**: readers never block writers. Each statement in Read Committed sees a snapshot of committed data.
- **SQL Server** Read Committed uses **locks** by default, so readers can block writers. Many teams enable **RCSI** (`READ_COMMITTED_SNAPSHOT ON`) to get MVCC-like behavior. Know whether yours does.

### The lost update: the TicketHub problem

```
T1: SELECT status FROM event_seats WHERE code='A1'   → 'Available'
T2: SELECT status FROM event_seats WHERE code='A1'   → 'Available'
T1: UPDATE event_seats SET status='Held', hold_id=h1 WHERE code='A1'; COMMIT
T2: UPDATE event_seats SET status='Held', hold_id=h2 WHERE code='A1'; COMMIT   ← overwrites T1: double booking
```

**Three fixes** (you'll implement the first and the second in [guide 4](04-ef-core.md)):

1. **Optimistic concurrency:** `UPDATE ... WHERE code='A1' AND version = @originalVersion`. If 0 rows are affected, someone else changed the row, so return a conflict (409).
2. **An atomic conditional update:** `UPDATE ... SET status='Held' WHERE code='A1' AND status='Available'`, then check the affected row count.
3. **Pessimistic locking:** `SELECT ... FOR UPDATE` (PostgreSQL) or `WITH (UPDLOCK, ROWLOCK)` (SQL Server) inside a transaction. This is correct but holds locks, so keep the transaction short.

---

## 3.7 Locks and deadlocks

A **deadlock** happens when T1 holds a lock on A and waits for B, while T2 holds B and waits for A. The database kills one transaction (the victim) with an error: PostgreSQL `40P01`, SQL Server `1205`.

**Prevention:**

- Access rows **in a consistent order**, for example sorting seat codes before updating them.
- Keep transactions **short**: no HTTP calls or user think-time inside a transaction.
- Use proper indexes, so updates lock fewer rows or pages.
- **Retry** on deadlock or serialization errors. EF Core's execution strategies help (see [guide 4](04-ef-core.md)).

---

## 3.8 Constraints are your last line of defense

```sql
ALTER TABLE orders ADD CONSTRAINT ck_orders_total CHECK (total >= 0);
CREATE UNIQUE INDEX ux_payments_provider_event ON payments (provider_event_id);   -- makes webhook processing idempotent
```

Application checks can race. **Unique constraints can't.** Whenever a rule says "only one", enforce it with a unique index *as well as* in code.

---

## 3.9 PostgreSQL vs SQL Server cheat sheet

| Topic | PostgreSQL | SQL Server |
|---|---|---|
| Row limit | `LIMIT n OFFSET m` | `OFFSET m ROWS FETCH NEXT n ROWS ONLY` / `TOP (n)` |
| Auto ID | `GENERATED ALWAYS AS IDENTITY` | `IDENTITY(1,1)` |
| UUID | `uuid` | `uniqueidentifier` (random GUIDs as a clustered key fragment the table; prefer sequential GUIDs such as `Guid.CreateVersion7()`) |
| Strings | `text` / `varchar` (UTF-8) | `nvarchar` (UTF-16) / `varchar` |
| Date/time with offset | `timestamptz` | `datetimeoffset` |
| JSON | `jsonb` (indexable with GIN) | `json` type (SQL Server 2025) / `nvarchar(max)` |
| Upsert | `INSERT ... ON CONFLICT DO UPDATE` | `MERGE` (use with care) |
| Optimistic concurrency token | the system column `xmin` | `rowversion` |
| Plan | `EXPLAIN (ANALYZE, BUFFERS)` | actual execution plan, Query Store |

> **`Guid.CreateVersion7()` (.NET 9+)** creates time-ordered GUIDs, which make much friendlier primary keys for B-tree indexes than random `Guid.NewGuid()` values.

---

## Interview questions

<details><summary><b>1. How do you decide the column order of a composite index?</b></summary>

Equality predicates first, then range or sort columns, following the leftmost-prefix rule. Also consider selectivity, and add `INCLUDE` columns to cover the query and avoid key lookups or heap fetches.
</details>

<details><summary><b>2. A query does a full scan even though there's an index. Why?</b></summary>

Possible causes: the predicate isn't sargable (a function on the column, a leading wildcard, an implicit conversion); low selectivity, so a scan really is cheaper; stale statistics; the leftmost index column isn't in the predicate; or a parameter-sniffing mismatch. Check actual vs estimated rows in the plan.
</details>

<details><summary><b>3. Explain the isolation levels and the default in PostgreSQL and SQL Server.</b></summary>

Read Uncommitted < Read Committed < Repeatable Read < Serializable; higher levels prevent more anomalies (dirty reads, non-repeatable reads, phantoms, write skew) at a higher cost. Both default to Read Committed. PostgreSQL uses MVCC snapshots. SQL Server uses locking unless RCSI is enabled. Serializable in PostgreSQL can abort transactions, so callers must retry.
</details>

<details><summary><b>4. How do you prevent two users from booking the same seat?</b></summary>

Use optimistic concurrency (a version column checked in the `UPDATE`'s `WHERE`), an atomic conditional `UPDATE ... WHERE status = 'Available'` with a row-count check, or pessimistic `SELECT ... FOR UPDATE`. Back it up with a unique constraint where possible, and test it with concurrent requests.
</details>

<details><summary><b>5. What causes deadlocks and how do you mitigate them?</b></summary>

Cyclic lock waits between transactions. Mitigate them with a consistent access order, short transactions, good indexes and retries on deadlock errors.
</details>

<details><summary><b>6. OFFSET vs keyset pagination?</b></summary>

OFFSET is simple and allows random page access, but its cost grows with depth and it's unstable under concurrent inserts. Keyset pagination uses a `WHERE (sort_key, id) < (last values)` predicate. Its cost is constant and it's stable, but it has no random page access.
</details>

---

## Exercises (SQL lab)

1. Load the practice schema with at least 500,000 orders. For each of these queries, capture the plan **before** and **after** adding an index, and record the timings and buffers:
   - a customer's latest 20 orders;
   - the available seats for an event;
   - the expired holds (`status = 'Held' AND hold_expires_at < now()`).
2. Rewrite three non-sargable predicates and show the plan change.
3. Reproduce a **lost update** with two `psql` sessions, then fix it twice: with a version column and with a conditional `UPDATE`.
4. Reproduce a **deadlock** with two sessions that update two rows in opposite order. Read the error, then fix it by ordering the updates.
5. Write a window-function query that returns each event's top 3 customers by spend.

## Self-check

- [ ] I can write joins, `GROUP BY`/`HAVING`, CTEs and window functions without looking them up.
- [ ] I can design a composite and covering index for a given query and verify it in the plan.
- [ ] I can read `EXPLAIN ANALYZE` output and spot the expensive node.
- [ ] I can explain the isolation levels, the lost-update problem and three ways to prevent it.

## Further reading

- *Use The Index, Luke!* (Markus Winand, free online). The best resource on indexes.
- PostgreSQL docs: *Using EXPLAIN*, *Transaction Isolation*, *Explicit Locking*.
- Microsoft Learn: *SQL Server index architecture and design guide*, *Transaction locking and row versioning guide*.
- *Designing Data-Intensive Applications*, chapter 7 (Transactions).
