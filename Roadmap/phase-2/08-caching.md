# 8. Caching: Output Caching, HybridCache and Redis

> Phase 2 · Week 11 · [← Phase index](README.md)

**Why this matters:** Caching is the cheapest way to make a read-heavy system fast, and the easiest way to serve **wrong data**. "There are only two hard things in computer science: cache invalidation and naming things." A Middle developer chooses the right cache layer and has an explicit invalidation strategy.

---

## 8.1 Cache layers

```
Browser cache ─► CDN ─► Reverse proxy ─► Output cache (whole HTTP responses) ─► HybridCache (objects: L1 memory + L2 Redis) ─► Database
```

| Layer | Caches | Invalidation | Good for |
|---|---|---|---|
| HTTP caching headers (`Cache-Control`, `ETag`) | responses at the client or CDN | TTL, or revalidation with `If-None-Match` → 304 | public, mostly static content |
| **Output caching** | full responses on the server | TTL plus **tags** (`EvictByTagAsync`) | anonymous GETs such as the event catalog |
| **In-memory (`IMemoryCache`)** | objects in one process | TTL or removal; **per instance** | tiny, hot, rarely-changing data |
| **Distributed (Redis)** | serialized objects shared by every instance | TTL or removal | data shared across instances; survives deploys |
| **HybridCache** | L1 memory + L2 distributed, with stampede protection and tags | TTL plus `RemoveAsync` / `RemoveByTagAsync` | **the default choice for object caching in .NET 9+** |

---

## 8.2 Cache-aside, and what can go wrong

**Cache-aside** (lazy loading):

1. Read from the cache.
2. On a miss, load from the database and store the result in the cache with a TTL.
3. On a write, update the database, then **invalidate** (remove) the cache entry.

| Problem | Description | Mitigation |
|---|---|---|
| **Stale data** | the cache still holds the old value after an update | invalidate on write; short TTLs; tags |
| **Cache stampede** (thundering herd) | a hot key expires, and 1,000 requests hit the database at once | single-flight per key (**HybridCache does this**), early refresh, TTL jitter |
| **Cache penetration** | repeated requests for keys that don't exist always miss | cache "not found" briefly; validate IDs |
| **Large values** | serializing big graphs is slow and memory-heavy | cache DTOs or projections, not entities |
| **Caching per-user data in a shared cache** | data leaks between users | include the user or tenant in the key, or don't cache it |
| **Cache treated as the source of truth** | lost writes when the cache is evicted | the database is the truth; the cache is disposable |

> **What *not* to cache in TicketHub:** **seat availability during a sale.** It changes every second, and a stale "Available" leads to failed holds. Cache the *catalog* (event details, venues, prices), and serve availability from the database (indexed) or push it with SSE.

---

## 8.3 Output caching

```csharp
builder.Services.AddOutputCache(options =>
{
    options.AddBasePolicy(b => b.Expire(TimeSpan.FromSeconds(30)));
    options.AddPolicy("catalog", b => b.Expire(TimeSpan.FromMinutes(5)).Tag("catalog").SetVaryByQuery("page", "city", "from"));
});
// Shared across instances: builder.Services.AddStackExchangeRedisOutputCache(o => o.Configuration = redisConnection);

app.UseOutputCache();   // after UseCors and UseAuthentication

app.MapGet("/api/events", ListEvents).CacheOutput("catalog").AllowAnonymous();

// After publishing or updating an event:
await outputCacheStore.EvictByTagAsync("catalog", ct);   // IOutputCacheStore from DI
```

- By default, it only caches `GET`/`HEAD` 200 responses for **unauthenticated** requests without cookies. That's a safe default, so don't override it carelessly.
- It's the cheapest win for public catalog endpoints.

---

## 8.4 HybridCache (object caching done right)

```csharp
// Packages: Microsoft.Extensions.Caching.Hybrid, Microsoft.Extensions.Caching.StackExchangeRedis
builder.Services.AddStackExchangeRedisCache(o => o.Configuration = builder.Configuration.GetConnectionString("redis"));
builder.Services.AddHybridCache(o =>
{
    o.DefaultEntryOptions = new HybridCacheEntryOptions
    {
        Expiration = TimeSpan.FromMinutes(10),            // L2 (Redis)
        LocalCacheExpiration = TimeSpan.FromMinutes(1),   // L1 (memory); keep it shorter to limit cross-instance staleness
    };
});
```

```csharp
public sealed class EventCatalog(HybridCache cache, TicketHubDbContext db)
{
    public ValueTask<EventDetailsDto?> GetAsync(Guid id, CancellationToken ct) =>
        cache.GetOrCreateAsync(
            $"event:{id}",                                                    // namespaced, stable key
            async token => await db.Events.AsNoTracking()
                .Where(e => e.Id == id)
                .Select(e => new EventDetailsDto(e.Id, e.Name, e.VenueName, e.StartsAt))
                .FirstOrDefaultAsync(token),
            tags: ["catalog", $"event:{id}"],
            cancellationToken: ct);

    public async Task InvalidateAsync(Guid id, CancellationToken ct)
    {
        await cache.RemoveAsync($"event:{id}", ct);
        await cache.RemoveByTagAsync("catalog", ct);                        // lists that include this event
    }
}
```

**What HybridCache gives you over `IDistributedCache`:**

- **Stampede protection:** concurrent misses for the same key run the factory *once*.
- **Two levels:** memory first, then Redis, then the database.
- **Tag-based invalidation.**
- A clean async API with serialization handled for you.

> **L1 and multiple instances:** when instance A invalidates, instance B's **memory** copy lives until `LocalCacheExpiration`. Keep L1 short for data that changes, or use only L2 for those keys.

---

## 8.5 Redis beyond caching

| Use | Redis feature | Notes |
|---|---|---|
| Cache | strings with TTL | eviction policy `allkeys-lru` for pure caches |
| Rate limiting across instances | `INCR` + `EXPIRE`, or a sorted set (sliding window) | also offered by gateways |
| SignalR backplane / pub-sub | `PUBLISH`/`SUBSCRIBE` | fire-and-forget: no durability |
| Leaderboards, counters | sorted sets, `INCRBY` | |
| Distributed locks | `SET key value NX PX 30000` | ⚠️ see below |
| Queues / streams | Redis Streams | possible, but prefer a real broker for business messages (Phase 3) |

### Distributed locks: be careful

```csharp
// Acquire: only if the key doesn't exist (NX), with an expiry so a crashed holder can't keep it forever
bool acquired = await redis.StringSetAsync($"lock:report:{id}", ownerToken, TimeSpan.FromSeconds(30), When.NotExists);
```

**Pitfalls:**

1. A GC pause or network delay can outlast the TTL, so **two holders** think they own the lock.
2. Releasing it must check the owner token (atomically, with a Lua script), or you might delete someone else's lock.
3. Redlock across nodes is debated; see Martin Kleppmann's "How to do distributed locking".

**Rule:** use Redis locks for **efficiency** (avoiding duplicate work). For **correctness** (preventing double booking), rely on **database constraints, optimistic concurrency or fencing tokens**, as TicketHub does.

---

## Interview questions

<details><summary><b>1. Explain cache-aside and its main problems.</b></summary>

Read from the cache; on a miss, load from the database and populate the cache; invalidate on write. The main problems are staleness, stampedes on hot keys, penetration by missing keys, consistency across instances, and accidentally caching per-user data in shared keys.
</details>

<details><summary><b>2. What is a cache stampede and how do you prevent it?</b></summary>

Many concurrent misses for the same expired key overload the database. Prevent it with single-flight (one loader per key; HybridCache does this), early or background refresh, and TTL jitter.
</details>

<details><summary><b>3. Output caching vs HybridCache?</b></summary>

Output caching stores whole HTTP responses (best for anonymous GETs) and invalidates by tag. HybridCache stores objects or DTOs used by any code, with L1 and L2 levels and stampede protection. They're often used together.
</details>

<details><summary><b>4. Can a Redis lock guarantee that two workers never do the same thing?</b></summary>

No. Pauses and clock or network issues can let the TTL expire while the holder is still working. Use locks for efficiency, and enforce correctness with database constraints, idempotency or fencing tokens.
</details>

---

## Exercises

1. Add output caching to `GET /api/events` with tag invalidation on publish or update. Measure the latency with a cold vs a warm cache.
2. Implement `EventCatalog` with HybridCache over Redis. Prove stampede protection: 100 concurrent requests for an uncached key should run one database query (count it with logging).
3. Write an ADR (an early one for Phase 3): "What we cache and what we never cache in TicketHub, and why."
4. Implement a Redis lock with owner-token release (a Lua script) and write down a scenario in which it fails.

## Self-check

- [ ] I can choose a cache layer per use case and define its invalidation.
- [ ] I implemented HybridCache with tags and understand L1/L2 staleness.
- [ ] I know why seat availability isn't cached, and why Redis locks don't guarantee correctness.

## Further reading

- Microsoft Learn: *Output caching middleware*, *HybridCache library*, *Distributed caching*.
- Martin Kleppmann, *"How to do distributed locking"* (blog).
- Redis docs: *Eviction policies*, *Distributed locks*.
