# 4. EF Core 10 in Depth (plus Dapper)

> Phase 2 · Weeks 5–6 · [← Phase index](README.md)

**Why this matters:** EF Core is in almost every .NET backend. Used well, it's productive and fast. Used carelessly, it produces N+1 queries, memory leaks through the change tracker, lost updates and migrations that take production down. Interviewers probe EF Core heavily, because it shows whether you understand what happens *underneath*.

---

## 4.1 What a `DbContext` really is

| Role | Meaning |
|---|---|
| **Unit of work** | collects changes and writes them in **one transaction** on `SaveChanges` |
| **Identity map** | within one context, one database row maps to one object instance |
| **Change tracker** | remembers original values to detect what changed |
| **Query translator** | turns LINQ expression trees into SQL |

```csharp
builder.Services.AddDbContextPool<TicketHubDbContext>(options => options
    .UseNpgsql(builder.Configuration.GetConnectionString("tickethub"),
        npgsql => npgsql.EnableRetryOnFailure())      // retries transient errors such as failovers
    .UseSnakeCaseNamingConvention());                 // package EFCore.NamingConventions (optional)
```

- **Scoped lifetime**, one per request. **It isn't thread-safe:** no `Task.WhenAll` over one context.
- `AddDbContextPool` reuses instances for less allocation. The context must be stateless apart from options.
- For background services, Blazor Server or parallel work, use `IDbContextFactory<T>` and `CreateDbContextAsync()`, then dispose the context yourself.

---

## 4.2 Modeling

```csharp
public sealed class TicketHubDbContext(DbContextOptions<TicketHubDbContext> options) : DbContext(options)
{
    public DbSet<Event> Events => Set<Event>();
    public DbSet<EventSeat> EventSeats => Set<EventSeat>();
    public DbSet<Order> Orders => Set<Order>();

    protected override void OnModelCreating(ModelBuilder modelBuilder) =>
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(TicketHubDbContext).Assembly);
}

public sealed class EventSeatConfiguration : IEntityTypeConfiguration<EventSeat>
{
    public void Configure(EntityTypeBuilder<EventSeat> b)
    {
        b.HasKey(s => new { s.EventId, s.Code });
        b.Property(s => s.Code).HasMaxLength(32);
        b.Property(s => s.Status).HasConversion<string>().HasMaxLength(16);   // enums as readable strings
        b.Property(s => s.Version).IsRowVersion();                            // Npgsql maps uint + row version to xmin
        b.HasIndex(s => new { s.EventId, s.Status });
        b.HasIndex(s => s.HoldExpiresAt).HasFilter("status = 'Held'");        // partial index; the filter is raw SQL, so use the real column name
        b.HasOne<Event>().WithMany().HasForeignKey(s => s.EventId);
    }
}
```

**Modeling rules:**

- Use **Fluent API configuration classes** (`IEntityTypeConfiguration<T>`) rather than attributes on domain classes. That keeps the domain persistence-ignorant.
- **Always set max lengths.** Otherwise strings become `text` or `nvarchar(max)`, which hurts indexing and memory grants.
- **Rich domain models work.** EF can use private setters, backing fields and private parameterless constructors, so you don't need public setters everywhere.
- **Value objects** (`Money`, `Address`) map as **complex types**: `b.ComplexProperty(o => o.Total)`. EF 10 adds optional complex types, struct support, and mapping complex types to **JSON columns** (`.ToJson()`). Prefer complex types over the older *owned entities* for value objects.
- **Keys:** use `Guid.CreateVersion7()` (time-ordered) for GUID keys, or database identity columns.

---

## 4.3 The change tracker

| State | Meaning on `SaveChanges` |
|---|---|
| `Added` | INSERT |
| `Unchanged` | nothing |
| `Modified` | UPDATE (only the changed columns) |
| `Deleted` | DELETE |
| `Detached` | not tracked |

```csharp
var seat = await db.EventSeats.SingleAsync(s => s.EventId == id && s.Code == "A1", ct); // tracked, Unchanged
seat.Hold(holdId, expiresAt);                                                          // DetectChanges → Modified
await db.SaveChangesAsync(ct);                                                         // UPDATE ... SET status, hold_id, ...
```

- **Read-only queries: use `AsNoTracking()`** (or `UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking)` as the default for query-heavy contexts). It saves memory and CPU.
- `db.Update(entity)` marks **every** property as modified. Prefer loading the entity and changing what you need, or use `ExecuteUpdate`.
- **A long-lived context** (for example, one per background job that processes 100,000 rows) keeps every entity in memory. Process in batches with a new context per batch, or call `ChangeTracker.Clear()`.

---

## 4.4 Querying well

### Projection is the default for reads

```csharp
// ✅ Selects only the needed columns, untracked, one round-trip
var orders = await db.Orders
    .Where(o => o.CustomerId == customerId)
    .OrderByDescending(o => o.CreatedAt)
    .Select(o => new MyOrderDto(o.Id, o.Total, o.Event.Name, o.Event.StartsAt, o.Status))
    .Take(20)
    .ToListAsync(ct);
```

### N+1: the classic

```csharp
// ❌ 1 query for the orders + 1 query PER order (or the same thing through lazy loading)
var list = await db.Orders.Where(o => o.CustomerId == customerId).ToListAsync(ct);
foreach (var o in list)
{
    var evt = await db.Events.FindAsync([o.EventId], ct);   // a round-trip per iteration
    // ...
}
```

**How to detect it:** log the SQL in development and look for the same statement repeated N times:

```csharp
options.LogTo(Console.WriteLine, LogLevel.Information)
       .EnableSensitiveDataLogging();   // DEVELOPMENT ONLY: it logs parameter values
```

Phase 4 uses OpenTelemetry traces, which show N+1 as a ladder of identical database spans.

**Fix it** with a projection (as above), `Include` (for loading aggregates to modify them) or a batched `WHERE id IN (...)` lookup.

### Loading related data

| Strategy | Code | Use when |
|---|---|---|
| Eager | `.Include(o => o.Lines).ThenInclude(l => l.Seat)` | you'll **modify** the aggregate |
| Filtered include | `.Include(e => e.Seats.Where(s => s.Status == SeatStatus.Available))` | you only need part of a collection |
| Projection | `.Select(...)` | **read models**: the most common case |
| Explicit | `await db.Entry(order).Collection(o => o.Lines).LoadAsync(ct)` | conditional loading |
| Lazy | proxies or `ILazyLoader` | ⚠️ avoid in web APIs: hidden N+1, and it fails after the context is disposed |

### Cartesian explosion and split queries

Including two collections (`.Include(e => e.Seats).Include(e => e.Orders)`) in one SQL statement multiplies the rows (seats × orders). Use **`.AsSplitQuery()`** to issue one query per collection. That trades a JOIN for extra round-trips.

### Know your SQL

```csharp
var query = db.Events.Where(e => e.StartsAt > now).OrderBy(e => e.StartsAt);
Console.WriteLine(query.ToQueryString());   // see the SQL without running it
```

**EF 10 notes:**

- `ids.Contains(x.Id)` now translates to a padded list of scalar parameters (good for plan caching).
- The new `LeftJoin`/`RightJoin` LINQ operators translate to SQL joins.
- Inlined constants are redacted from logs by default.

---

## 4.5 Writing efficiently

```csharp
// Bulk operations run directly in SQL, without loading or tracking entities (EF 7+)
int released = await db.EventSeats
    .Where(s => s.Status == SeatStatus.Held && s.HoldExpiresAt <= now)
    .ExecuteUpdateAsync(s => s
        .SetProperty(x => x.Status, SeatStatus.Available)
        .SetProperty(x => x.HoldId, (Guid?)null)
        .SetProperty(x => x.HoldExpiresAt, (DateTimeOffset?)null), ct);

await db.Orders.Where(o => o.Status == OrderStatus.Abandoned && o.CreatedAt < cutoff).ExecuteDeleteAsync(ct);
```

- `ExecuteUpdate` and `ExecuteDelete` **bypass the change tracker and SaveChanges interceptors**. Audit fields and domain events won't fire, and tracked entities in memory become stale.
- In EF 10, the `ExecuteUpdateAsync` setter can be a regular lambda with `if` statements, which makes dynamic updates easy.
- `SaveChanges` **batches** statements automatically. Adding 1,000 entities doesn't mean 1,000 round-trips.

---

## 4.6 Optimistic concurrency: the TicketHub seat hold

```csharp
public enum SeatStatus { Available, Held, Sold }

public sealed class EventSeat
{
    private EventSeat() { }                                  // for EF

    public EventSeat(Guid eventId, string code, string category)
    {
        EventId = eventId;
        Code = code;
        Category = category;
    }

    public Guid EventId { get; private set; }
    public string Code { get; private set; } = string.Empty;
    public string Category { get; private set; } = string.Empty;
    public SeatStatus Status { get; private set; }
    public Guid? HoldId { get; private set; }
    public DateTimeOffset? HoldExpiresAt { get; private set; }
    public uint Version { get; private set; }               // xmin concurrency token

    public bool IsAvailable(DateTimeOffset now) =>
        Status == SeatStatus.Available || (Status == SeatStatus.Held && HoldExpiresAt <= now);

    public void Hold(Guid holdId, DateTimeOffset expiresAt)
    {
        Status = SeatStatus.Held;
        HoldId = holdId;
        HoldExpiresAt = expiresAt;
    }
}
```

```csharp
public async Task<Result<HoldDto>> HoldAsync(Guid eventId, string[] codes, Guid customerId, CancellationToken ct)
{
    var now = clock.GetUtcNow();
    var distinct = codes.Distinct(StringComparer.OrdinalIgnoreCase).ToArray();

    var seats = await db.EventSeats
        .Where(s => s.EventId == eventId && distinct.Contains(s.Code))
        .ToListAsync(ct);                                        // tracked: we're going to modify them

    if (seats.Count != distinct.Length || seats.Any(s => !s.IsAvailable(now)))
        return EventErrors.SeatsUnavailable(distinct);

    var holdId = Guid.CreateVersion7();
    var expiresAt = now + TimeSpan.FromMinutes(10);
    foreach (var seat in seats)
        seat.Hold(holdId, expiresAt);

    try
    {
        await db.SaveChangesAsync(ct);                           // UPDATE ... WHERE event_id=@e AND code=@c AND xmin=@v
    }
    catch (DbUpdateConcurrencyException)
    {
        return EventErrors.SeatsUnavailable(distinct);           // someone else changed a seat between our read and write
    }

    return new HoldDto(holdId, eventId, distinct, expiresAt);
}
```

**How it works:**

1. EF adds `AND xmin = @original` to each `UPDATE`.
2. If another transaction changed the row in the meantime, the database affects 0 rows.
3. EF then throws `DbUpdateConcurrencyException`, and the whole `SaveChanges` transaction rolls back, so either all seats are held or none.

On SQL Server, the equivalent is `[Timestamp] public byte[] RowVersion { get; set; }`.

**Alternative: an atomic conditional update.** It needs no read first and no exception:

```csharp
await using var tx = await db.Database.BeginTransactionAsync(ct);
int updated = await db.EventSeats
    .Where(s => s.EventId == eventId && distinct.Contains(s.Code)
             && (s.Status == SeatStatus.Available || (s.Status == SeatStatus.Held && s.HoldExpiresAt <= now)))
    .ExecuteUpdateAsync(s => s
        .SetProperty(x => x.Status, SeatStatus.Held)
        .SetProperty(x => x.HoldId, holdId)
        .SetProperty(x => x.HoldExpiresAt, expiresAt), ct);

if (updated != distinct.Length)
{
    await tx.RollbackAsync(ct);                  // partial success isn't allowed
    return EventErrors.SeatsUnavailable(distinct);
}
await tx.CommitAsync(ct);
```

> **Prove it with a test.** You'll fire 20 concurrent hold requests for the same seat and assert exactly **one** 201 response and **nineteen** 409s ([guide 9](09-integration-testing.md)).

---

## 4.7 Transactions

- **Each `SaveChanges` is a transaction.** You don't need an explicit transaction for one unit of work.
- Use an explicit transaction for **several `SaveChanges` calls or raw SQL** that must be atomic.
- With **retrying execution strategies** (`EnableRetryOnFailure`), wrap explicit transactions in the strategy so the *whole block* is retried:

```csharp
var strategy = db.Database.CreateExecutionStrategy();
await strategy.ExecuteAsync(async () =>
{
    await using var tx = await db.Database.BeginTransactionAsync(ct);
    // ... several operations ...
    await db.SaveChangesAsync(ct);
    await tx.CommitAsync(ct);
});
```

- **Never** hold a transaction open across an HTTP call to another service. That's a distributed-transaction smell; Phase 3 covers outboxes and sagas instead.

---

## 4.8 Migrations

```powershell
dotnet ef migrations add AddEventSeats -p src/TicketHub.Api
dotnet ef database update -p src/TicketHub.Api                              # local development
dotnet ef migrations script --idempotent -o artifacts/migrate.sql           # for review and DBA pipelines
dotnet ef migrations bundle --self-contained -r linux-x64 -o artifacts/efbundle   # a single executable for CD
```

| Rule | Why |
|---|---|
| **One migration per PR**, reviewed like code. Read the generated SQL | migrations can drop columns or rebuild tables |
| Resolve conflicts by **removing your migration, merging main, and re-adding it** | merge conflicts in the model snapshot are dangerous |
| **Don't call `Database.Migrate()` at startup** in multi-instance production | several instances race, and a failed migration crash-loops the app. Use a pipeline step, a Kubernetes init container, or a bundle |
| **Expand → migrate → contract** for zero-downtime changes | e.g., add the new column (nullable) → deploy code that writes both → backfill → switch reads → drop the old column in a later release |
| Data migrations: idempotent SQL in the migration (`migrationBuilder.Sql(...)`) or a separate job | keep them re-runnable |

---

## 4.9 Interceptors and query filters

```csharp
public interface IAuditable
{
    DateTimeOffset CreatedAt { get; set; }
    DateTimeOffset? UpdatedAt { get; set; }
}

public sealed class AuditInterceptor(TimeProvider clock) : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData, InterceptionResult<int> result, CancellationToken cancellationToken = default)
    {
        if (eventData.Context is not null)
        {
            var now = clock.GetUtcNow();
            foreach (var entry in eventData.Context.ChangeTracker.Entries<IAuditable>())
            {
                if (entry.State == EntityState.Added) entry.Entity.CreatedAt = now;
                if (entry.State == EntityState.Modified) entry.Entity.UpdatedAt = now;
            }
        }
        return base.SavingChangesAsync(eventData, result, cancellationToken);
    }
}

builder.Services.AddSingleton<AuditInterceptor>();
builder.Services.AddDbContext<TicketHubDbContext>((sp, options) => options
    .UseNpgsql(connectionString)
    .AddInterceptors(sp.GetRequiredService<AuditInterceptor>()));
```

**Named query filters (EF 10)** for soft delete and multi-tenancy:

```csharp
modelBuilder.Entity<Event>()
    .HasQueryFilter("SoftDelete", e => !e.IsDeleted)
    .HasQueryFilter("Tenant", e => e.TenantId == _tenantId);       // _tenantId is a field on the DbContext

var includingDeleted = await db.Events.IgnoreQueryFilters(["SoftDelete"]).ToListAsync(ct);
```

---

## 4.10 Performance checklist

- [ ] Read queries use **projections** or `AsNoTracking`.
- [ ] There's no N+1 (check the SQL log or traces).
- [ ] Every `WHERE`/`ORDER BY` column on a hot path has an index, defined with `HasIndex` and checked with `EXPLAIN`.
- [ ] Pagination is applied **in SQL** (`Skip`/`Take` or keyset) and never after `ToList()`.
- [ ] There's no client-side evaluation of filters (`AsEnumerable()` too early).
- [ ] Bulk changes use `ExecuteUpdate`/`ExecuteDelete`.
- [ ] Hot single-row lookups use a context pool and, if they're profiled as hot, compiled queries (`EF.CompileAsyncQuery`).
- [ ] Contexts are short-lived, and batch jobs clear or recreate them.

---

## 4.11 Dapper and raw SQL

Use **Dapper** for complex reporting queries, hand-tuned hot paths, or when you want exactly the SQL you wrote:

```csharp
await using var connection = new NpgsqlConnection(connectionString);
var revenue = await connection.QueryAsync<EventRevenue>(
    """
    SELECT e.name AS EventName, SUM(o.total) AS Revenue
    FROM orders o JOIN events e ON e.id = o.event_id
    WHERE o.status = 'Paid' AND o.created_at >= @since
    GROUP BY e.name ORDER BY Revenue DESC LIMIT @top
    """,
    new { since, top = 10 });                 // ALWAYS parameterized: never concatenate user input
```

**Raw SQL in EF Core:**

- `FromSql($"... WHERE id = {id}")` is **safe**: the interpolation becomes parameters.
- `FromSqlRaw("..." + input)` is **dangerous**: it's SQL injection. EF 10 ships an analyzer that warns about concatenation in raw SQL APIs.

---

## Interview questions

<details><summary><b>1. What is the change tracker, and when would you disable it?</b></summary>

It tracks loaded entities and their original values, so `SaveChanges` can generate UPDATE, INSERT and DELETE statements and keep one instance per row (the identity map). Disable it (`AsNoTracking` or projections) for read-only queries, to save memory and CPU.
</details>

<details><summary><b>2. What is N+1 and how do you fix it?</b></summary>

One query for a list, then one more query per item, through lazy loading or loops. Fix it with a projection (a JOIN in one query), `Include` when you need the full aggregate, or a batched `IN` query. Detect it with SQL logging or traces.
</details>

<details><summary><b>3. How does optimistic concurrency work in EF Core?</b></summary>

A concurrency token (`rowversion` on SQL Server, `xmin` on PostgreSQL, or a version column) is added to the `UPDATE`'s `WHERE` clause. If 0 rows are affected, EF throws `DbUpdateConcurrencyException`, and the app resolves it by returning a conflict, retrying, or merging.
</details>

<details><summary><b>4. Split query vs single query?</b></summary>

A single query with several collection Includes produces a Cartesian product (duplicated data). A split query issues one SQL query per collection: less duplicated data, but more round-trips and possible inconsistency without a transaction. Choose per query, based on the data's shape.
</details>

<details><summary><b>5. Why not run migrations at application startup?</b></summary>

Several instances race to apply them, a long migration blocks startup and health checks, and a failure leads to a crash loop. Run migrations as a deliberate pipeline step (a bundle, a script, or an init container).
</details>

<details><summary><b>6. What do <code>ExecuteUpdate</code> and <code>ExecuteDelete</code> skip?</b></summary>

The change tracker and SaveChanges interceptors. So no audit fields set by interceptors and no domain events, and tracked entities in memory become stale.
</details>

---

## Exercises

1. Turn on SQL logging and implement "my orders" three ways: N+1, `Include`, and a projection. Compare the SQL, the round-trips and the timings.
2. Implement the seat hold with **both** optimistic concurrency and the atomic conditional update. Write down which one you prefer and why (it's material for an ADR in Phase 3).
3. Create a migration that renames a column using the **expand/contract** approach across two releases.
4. Add `AuditInterceptor` with `TimeProvider` and test it with `FakeTimeProvider` against a real PostgreSQL in Testcontainers.
5. Write one reporting query in Dapper and compare it with the EF LINQ version, for readability and for the SQL it generates.

## Self-check

- [ ] I can explain the unit of work, the identity map and the change tracker.
- [ ] I project by default and can spot and fix N+1 queries.
- [ ] I implemented optimistic concurrency and can handle conflicts properly.
- [ ] I know the safe migration workflow, including expand/contract.
- [ ] I know when to use `ExecuteUpdate` and Dapper, and their trade-offs.

## Further reading

- Microsoft Learn: *EF Core performance* (the whole section), *Handling concurrency conflicts*, *Migrations overview*, *Complex types*, *What's new in EF Core 10*.
- *Entity Framework Core in Action* (Jon P Smith).
- The Npgsql EF Core provider docs: *Concurrency tokens (xmin)*.
