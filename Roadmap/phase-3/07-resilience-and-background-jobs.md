# 7. Resilience and Background Jobs

> Phase 3 · Week 10 · [← Phase index](README.md)

**Why this matters:** Every call over a network eventually fails or hangs. Without timeouts and sensible retries, one slow dependency takes down the whole system (cascading failure). With *badly* designed retries, you make the outage worse (retry storms). Background jobs have their own traps: duplicate runs across instances and lost work on restart.

---

## 7.1 Failure modes you must design for

| Failure | Example | Right response |
|---|---|---|
| **Transient** | a dropped connection, 503 during a deployment, 429 throttling | retry with back-off |
| **Slow** | a dependency answering in 30 s instead of 200 ms | **timeout** (then retry or fail fast) |
| **Persistent outage** | the payment provider is down for 20 minutes | **circuit breaker**: stop calling it; fail fast or fall back |
| **Overload** | too many concurrent calls to a fragile service | **bulkhead / concurrency limit**, load shedding |
| **Permanent** | 400 Bad Request, 404, validation failure | **don't retry**; fix the input |

> **Timeouts come first.** A missing timeout turns a slow dependency into exhausted threads, connection pools and memory in *your* service. `HttpClient`'s default timeout is 100 seconds, which is far too long for most service-to-service calls.

---

## 7.2 The resilience strategies

| Strategy | What it does | Key settings |
|---|---|---|
| **Timeout** | cancels an attempt or the whole operation | per-attempt *and* total timeout |
| **Retry** | tries again after transient failures | max attempts, **exponential back-off + jitter**, which failures to retry |
| **Circuit breaker** | after many failures, "opens" and fails fast for a period, then lets a probe through ("half-open") | failure ratio, sampling window, minimum throughput, break duration |
| **Bulkhead / concurrency limiter** | limits concurrent calls so one dependency can't consume every resource | permit limit, queue limit |
| **Rate limiter (client side)** | respects a provider's quota | tokens per period |
| **Fallback** | returns a degraded answer (cached data, a default) | what's acceptable to the business |
| **Hedging** | sends a second request if the first is slow, and uses whichever answers first | only for idempotent reads |

**Why jitter?** Without it, 1,000 clients that failed at the same moment all retry at exactly 1 s, 2 s and 4 s, sending synchronized waves at a recovering service. Jitter spreads them out.

---

## 7.3 HTTP resilience in .NET: `Microsoft.Extensions.Http.Resilience`

```csharp
// Package: Microsoft.Extensions.Http.Resilience (built on Polly v8)
builder.Services.AddHttpClient<PaymentsClient>(client =>
    {
        client.BaseAddress = new Uri(builder.Configuration["Payments:BaseUrl"]!);
    })
    .AddStandardResilienceHandler(options =>
    {
        options.AttemptTimeout.Timeout = TimeSpan.FromSeconds(2);
        options.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(10);
        options.Retry.MaxRetryAttempts = 2;
        options.CircuitBreaker.SamplingDuration = TimeSpan.FromSeconds(30);   // must be ≥ 2 × the attempt timeout
        options.CircuitBreaker.FailureRatio = 0.2;
        options.CircuitBreaker.BreakDuration = TimeSpan.FromSeconds(15);
    });
```

The **standard pipeline**, from outermost to innermost:

```
rate limiter → total timeout → retry (exponential + jitter, honors Retry-After) → circuit breaker → attempt timeout → HTTP call
```

It retries transient HTTP failures (5xx, 408, 429, `HttpRequestException`, attempt timeouts) by default. For read-heavy, latency-sensitive calls there's `AddStandardHedgingHandler()`.

### ⚠️ Only retry what's safe to repeat

- `GET`, `PUT` and `DELETE` are idempotent by HTTP semantics (if the server implements them correctly).
- `POST` **isn't**, unless you send an **idempotency key** that the server honors ([Phase 2, guide 6](../phase-2/06-api-design.md)).
- Payment charges without idempotency keys + automatic retries = double charges.

You can disable retries for unsafe methods:

```csharp
.AddStandardResilienceHandler(options => options.Retry.DisableForUnsafeHttpMethods());
```

---

## 7.4 Non-HTTP resilience with Polly v8 pipelines

```csharp
using Npgsql;
using Polly;
using Polly.Retry;

builder.Services.AddResiliencePipeline("reporting-db", pipeline => pipeline
    .AddRetry(new RetryStrategyOptions
    {
        MaxRetryAttempts = 3,
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = true,
        Delay = TimeSpan.FromMilliseconds(200),
        ShouldHandle = new PredicateBuilder().Handle<NpgsqlException>(ex => ex.IsTransient),
    })
    .AddTimeout(TimeSpan.FromSeconds(5)));

public sealed class ReportService(ResiliencePipelineProvider<string> pipelines, NpgsqlDataSource dataSource)
{
    private readonly ResiliencePipeline _pipeline = pipelines.GetPipeline("reporting-db");

    public async Task<long> CountOrdersAsync(CancellationToken ct) =>
        await _pipeline.ExecuteAsync(async token =>
        {
            await using var cmd = dataSource.CreateCommand("SELECT count(*) FROM orders");
            return (long)(await cmd.ExecuteScalarAsync(token))!;
        }, ct);
}
```

### EF Core: use the execution strategy, not a hand-made retry loop

```csharp
options.UseNpgsql(connectionString, npgsql => npgsql.EnableRetryOnFailure(maxRetryCount: 3));

// With retries enabled, a user-started transaction must run inside the strategy,
// so the WHOLE unit of work is retried, not just one statement
var strategy = db.Database.CreateExecutionStrategy();
await strategy.ExecuteAsync(async () =>
{
    await using var tx = await db.Database.BeginTransactionAsync(ct);
    // ... several SaveChangesAsync calls ...
    await tx.CommitAsync(ct);
});
```

---

## 7.5 Resilience design rules

1. **Retry in one layer only.** If the gateway retries 3×, the service 3× and the client 3×, one failure becomes **27** calls. That's a retry storm.
2. **Every timeout must be shorter than the caller's timeout.** Otherwise the caller gives up while you're still retrying.
3. **Honor `Retry-After`** on 429 and 503 responses (the standard handler does).
4. **One circuit breaker per dependency** (per client or host), so one bad dependency doesn't trip others.
5. **Fall back deliberately.** Show cached prices with a "may be outdated" label, queue the email for later, or degrade a recommendation widget to "popular items". Agree on this with product owners.
6. **Observe it.** Log and measure retries, breaker state changes and timeouts. A breaker that's open silently is an outage nobody sees.
7. **Test it.** Polly v8 includes chaos strategies (`AddChaosFault`, `AddChaosLatency`, `AddChaosOutcome`) to inject failures in test environments.

---

## 7.6 Background jobs: choosing a tool

| Option | Persistence | Multi-instance safe? | Best for |
|---|---|---|---|
| `BackgroundService` + `PeriodicTimer` | ❌ in memory | ❌ every instance runs it (unless you add a lock or row claiming) | simple loops: cache refresh, the outbox relay with `SKIP LOCKED` |
| **Hangfire** | ✅ (SQL Server, PostgreSQL, Redis via extensions) | ✅ jobs are claimed from storage | fire-and-forget, delayed and recurring jobs with retries and a dashboard |
| **Quartz.NET** (Apache 2.0) | ✅ (ADO.NET job store) | ✅ clustering | complex schedules (calendars, misfire policies) |
| Broker delayed messages (Service Bus scheduled messages, delayed retries) | ✅ in the broker | ✅ | timeouts in sagas, "do X in 15 minutes" |
| Cloud schedulers (Azure Functions timer, AWS EventBridge Scheduler) | ✅ | ✅ (one trigger) | cron-like jobs outside your app |

### Hangfire in a minute

```csharp
// Packages: Hangfire.AspNetCore + a storage provider (e.g. Hangfire.PostgreSql); see its docs for storage options
builder.Services.AddHangfire(config => config.UsePostgreSqlStorage(/* connection configuration */));
builder.Services.AddHangfireServer();

var app = builder.Build();

app.Services.GetRequiredService<IRecurringJobManager>()
    .AddOrUpdate<ExpiredHoldsJob>("expired-holds", job => job.RunAsync(CancellationToken.None), Cron.Minutely());

// Fire-and-forget from a request; persisted, so it survives a restart
app.MapPost("/api/reports", (IBackgroundJobClient jobs) =>
{
    var jobId = jobs.Enqueue<ReportJob>(job => job.GenerateAsync(CancellationToken.None));
    return Results.Accepted($"/api/reports/{jobId}");
});
```

Hangfire replaces `CancellationToken.None` with its own shutdown token at execution time.

### Job design rules

- **Idempotent:** jobs get retried; running twice must be safe.
- **Small and resumable:** process in batches and record progress, so a restart doesn't redo two hours of work.
- **Bounded:** a timeout per job, and alerts on repeated failures.
- **No unprotected request-scoped data:** pass IDs and reload data inside the job, rather than serializing entities.
- **Single-run guarantee** where it matters: storage-based claiming (Hangfire/Quartz), `SKIP LOCKED` row claiming, or a distributed lock **with a fencing token** ([guide 8](08-distributed-systems-theory.md)).

---

## Interview questions

<details><summary><b>1. Which resilience strategies do you apply to an outgoing HTTP call, and in what order?</b></summary>

Use a total timeout, a retry with exponential back-off and jitter (only for transient errors and idempotent requests), a circuit breaker and a per-attempt timeout. In .NET, `AddStandardResilienceHandler` provides exactly this pipeline, plus a rate limiter.
</details>

<details><summary><b>2. What's a retry storm, and how do you avoid it?</b></summary>

Retries multiplying across layers, or many clients retrying in sync, overload a recovering service. Retry in one layer only, add jitter, cap attempts, honor `Retry-After` and use circuit breakers.
</details>

<details><summary><b>3. How does a circuit breaker work?</b></summary>

It's closed while calls flow normally. When failures exceed a threshold within a sampling window, it **opens** and fails fast for the break duration. Then it goes **half-open** and lets a probe through: success closes it, failure opens it again. It protects both the caller's resources and the struggling dependency.
</details>

<details><summary><b>4. Why is retrying a POST dangerous?</b></summary>

`POST` isn't idempotent, so a retry after a timeout can repeat a side effect that already succeeded (a double charge or a duplicate order). Retry only with an idempotency key that the server deduplicates.
</details>

<details><summary><b>5. How do you make sure a recurring job runs on only one of five instances?</b></summary>

Use a job scheduler with persistent storage-based claiming (Hangfire, Quartz clustering), claim work rows with `SKIP LOCKED`, use an external scheduler that triggers once, or use a distributed lock or lease with fencing. Make the job idempotent anyway.
</details>

---

## Exercises

1. Build a fake "flaky payments" API that fails 50% of the time and adds 3 s of latency 10% of the time. Call it with and without `AddStandardResilienceHandler`, and measure the success rate and p95 latency ([Lab 9](10-labs.md#lab-9-resilience-under-a-flaky-dependency)).
2. Nest retries on purpose (a client retry plus a handler retry plus a gateway retry). Count the calls the dependency receives for one user request.
3. Use Polly chaos strategies in a test environment to inject latency, and verify that your timeouts fire.
4. Move a `BackgroundService` job at work, or in a lab, to Hangfire with retries, and observe it surviving an app restart.

## Self-check

- [ ] I always set timeouts, and I know why they come first.
- [ ] I can configure the standard resilience handler and explain each strategy.
- [ ] I retry only transient failures of idempotent operations, in one layer.
- [ ] I can choose between `BackgroundService`, Hangfire, Quartz and broker-scheduled messages.

## Further reading

- *Release It!*, 2nd ed. (Michael Nygard): stability patterns and anti-patterns.
- Microsoft Learn: *Build resilient HTTP apps* (Microsoft.Extensions.Http.Resilience).
- The Polly documentation (pollydocs.org), including the chaos engineering section.
- AWS Architecture Blog: *"Exponential Backoff And Jitter"*.
