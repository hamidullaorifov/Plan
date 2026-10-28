# 7. Cross-Cutting Concerns and Real Time

> Phase 2 · Week 10 · [← Phase index](README.md)
> Covers rate limiting, CORS, health checks, compression, HttpClient, SSE, SignalR and gRPC.

**Why this matters:** These features decide how your API behaves under abuse, failure and scale: whether a bot can take you down, whether Kubernetes restarts you correctly, whether outgoing calls exhaust sockets, and whether users see seat changes live.

---

## 7.1 Rate limiting

```csharp
using System.Threading.RateLimiting;

builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;

    // Per-user token bucket for authenticated users; per-IP for anonymous ones
    options.AddPolicy("per-user", httpContext =>
    {
        string partitionKey = httpContext.User.FindFirst("sub")?.Value
            ?? httpContext.Connection.RemoteIpAddress?.ToString()
            ?? "anonymous";

        return RateLimitPartition.GetTokenBucketLimiter(partitionKey, _ => new TokenBucketRateLimiterOptions
        {
            TokenLimit = 20,                                   // burst
            TokensPerPeriod = 10,                              // sustained rate
            ReplenishmentPeriod = TimeSpan.FromSeconds(1),
            QueueLimit = 0,
        });
    });

    // A strict limit on the expensive, abuse-prone endpoint
    options.AddFixedWindowLimiter("holds", o => { o.PermitLimit = 5; o.Window = TimeSpan.FromSeconds(10); });

    options.OnRejected = (ctx, _) =>
    {
        if (ctx.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
            ctx.HttpContext.Response.Headers.RetryAfter = ((int)retryAfter.TotalSeconds).ToString(CultureInfo.InvariantCulture);
        return ValueTask.CompletedTask;
    };
});

app.UseRateLimiter();                                   // after UseAuthentication, so the user is known
app.MapGroup("/api").RequireRateLimiting("per-user");
app.MapPost("/api/events/{id:guid}/holds", HoldSeats).RequireRateLimiting("holds");
```

| Algorithm | Behavior | Good for |
|---|---|---|
| Fixed window | N requests per window | simple caps; bursty at window edges |
| Sliding window | smooths the window edges | fairer caps |
| **Token bucket** | allows bursts up to a bucket size, then a steady rate | most APIs |
| Concurrency | at most N requests *in flight* | protecting expensive resources |

> The built-in limiter is **per instance**. With 3 instances, a limit of "10/s" really means about 30/s. For global limits, use an API gateway (Azure API Management, an ingress, YARP) or a Redis-based limiter. Behind proxies, partition by the real client IP (forwarded headers).

---

## 7.2 CORS

**CORS** is a **browser** rule that controls which *origins* (scheme + host + port) can call your API from JavaScript. It isn't an authentication mechanism, and servers and curl ignore it.

```csharp
builder.Services.AddCors(o => o.AddPolicy("web", p => p
    .WithOrigins("https://app.tickethub.dev", "http://localhost:5173")   // explicit list
    .WithMethods("GET", "POST", "PUT", "DELETE")
    .WithHeaders("Authorization", "Content-Type", "Idempotency-Key")
    .WithExposedHeaders("Location", "ETag")));

app.UseCors("web");
```

- **Preflight:** for non-simple requests (JSON bodies, `Authorization` headers), the browser first sends `OPTIONS`. The preflight carries no token, so CORS must run **before** authentication.
- ❌ Never combine `AllowAnyOrigin()` with `AllowCredentials()`. The framework forbids it because it would let any site make authenticated calls.
- With a **BFF on the same origin**, you often don't need CORS at all.

---

## 7.3 Health checks

```csharp
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"])
    .AddNpgSql(builder.Configuration.GetConnectionString("tickethub")!, tags: ["ready"])   // package AspNetCore.HealthChecks.NpgSql
    .AddRedis(builder.Configuration.GetConnectionString("redis")!, tags: ["ready"]);       // package AspNetCore.HealthChecks.Redis

app.MapHealthChecks("/health/live",  new HealthCheckOptions { Predicate = c => c.Tags.Contains("live") }).AllowAnonymous();
app.MapHealthChecks("/health/ready", new HealthCheckOptions { Predicate = c => c.Tags.Contains("ready") }).AllowAnonymous();
```

| Probe | Question | If it fails… | Should it check dependencies? |
|---|---|---|---|
| **Liveness** | is the process stuck? | the orchestrator **restarts** the container | ❌ **No.** If the database goes down, restarting every instance doesn't help and causes cascading restarts |
| **Readiness** | can it serve traffic right now? | it's **removed from load balancing** until it's ready | ✅ yes (database, cache) |
| **Startup** | has it finished starting? | liveness is delayed until it has | for slow starts |

Don't expose detailed health output (versions, connection strings, error messages) publicly.

---

## 7.4 Response compression

```csharp
builder.Services.AddResponseCompression(o => o.EnableForHttps = true);   // Brotli and Gzip
app.UseResponseCompression();
```

- It's useful for large JSON responses. Often the reverse proxy or CDN compresses instead, so don't do it twice.
- `EnableForHttps` has a theoretical BREACH risk when responses mix secrets with attacker-controlled input. That's rare for JSON APIs, but know the name.

---

## 7.5 Outgoing HTTP: `IHttpClientFactory` and resilience

```csharp
builder.Services.AddHttpClient<PaymentGatewayClient>(client =>
    {
        client.BaseAddress = new Uri(builder.Configuration["Payments:BaseUrl"]!);
        client.Timeout = Timeout.InfiniteTimeSpan;             // let the resilience pipeline own timeouts
    })
    .AddStandardResilienceHandler();                           // package Microsoft.Extensions.Http.Resilience

public sealed class PaymentGatewayClient(HttpClient http)
{
    public async Task<PaymentIntent> CreateIntentAsync(CreateIntent request, CancellationToken ct)
    {
        using var response = await http.PostAsJsonAsync("/v1/intents", request, ct);
        response.EnsureSuccessStatusCode();
        return (await response.Content.ReadFromJsonAsync<PaymentIntent>(ct))!;
    }
}
```

- **Never `new HttpClient()` per request:** it exhausts sockets. A long-lived static `HttpClient` doesn't see DNS changes. The factory pools handlers and rotates them.
- `AddStandardResilienceHandler()` adds rate limiting, an overall timeout, **retries with exponential backoff and jitter**, a **circuit breaker** and a per-attempt timeout. Tune it per dependency. **Beware retrying non-idempotent POSTs:** send an idempotency key. Phase 3 covers resilience in depth.

---

## 7.6 Real time: Server-Sent Events (SSE)

SSE is a one-way server → client stream over plain HTTP. It's simple, proxy-friendly, and the browser reconnects automatically (`EventSource`). .NET 10 has built-in support:

```csharp
app.MapGet("/api/events/{id:guid}/seat-updates", (Guid id, SeatUpdateFeed feed, CancellationToken ct) =>
    TypedResults.ServerSentEvents(feed.SubscribeAsync(id, ct), eventType: "seat"));
```

```csharp
public sealed record SeatChanged(string Code, string Status);

public sealed class SeatUpdateFeed
{
    // Simplified: a production version fans out per event and removes subscribers on disconnect.
    private readonly Channel<(Guid EventId, SeatChanged Change)> _channel = Channel.CreateUnbounded<(Guid, SeatChanged)>();

    public ValueTask PublishAsync(Guid eventId, SeatChanged change) => _channel.Writer.WriteAsync((eventId, change));

    public async IAsyncEnumerable<SeatChanged> SubscribeAsync(Guid eventId, [EnumeratorCancellation] CancellationToken ct)
    {
        await foreach (var (id, change) in _channel.Reader.ReadAllAsync(ct))
            if (id == eventId) yield return change;
    }
}
```

> This toy feed delivers each message to only *one* reader. Real fan-out needs a channel per subscriber, or a backplane (Redis pub/sub) once you have multiple instances. Building that properly is a good stretch exercise.

---

## 7.7 Real time: SignalR

SignalR is **bidirectional** (WebSockets, falling back to SSE or long polling) and provides groups, typed hubs and client SDKs for JS, .NET, Java and Swift.

```csharp
public interface ISeatClient { Task SeatChanged(string code, string status); }

public sealed class SeatHub : Hub<ISeatClient>
{
    public Task Watch(Guid eventId) => Groups.AddToGroupAsync(Context.ConnectionId, $"event-{eventId}");
}

builder.Services.AddSignalR();               // multiple instances: .AddStackExchangeRedis(redisConnection) backplane
app.MapHub<SeatHub>("/hubs/seats");

// Push from anywhere, for example after a successful hold:
public sealed class SeatNotifier(IHubContext<SeatHub, ISeatClient> hub)
{
    public Task NotifyAsync(Guid eventId, string code, string status) =>
        hub.Clients.Group($"event-{eventId}").SeatChanged(code, status);
}
```

| Choose | When |
|---|---|
| Polling | updates are rare, or it's the simplest thing that works |
| **SSE** | one-way server → client notifications (live seat map, progress) |
| **SignalR** | two-way interaction, groups, many client platforms |
| gRPC streaming | service-to-service streaming |

---

## 7.8 gRPC (service-to-service)

gRPC is contract-first (`.proto`), uses binary Protobuf over HTTP/2, and supports streaming and generated clients. It's faster and more strongly typed than JSON for **internal** calls. It isn't usable directly from browsers without gRPC-Web.

```proto
syntax = "proto3";
option csharp_namespace = "TicketHub.Pricing";

service PricingService {
  rpc GetPrice (GetPriceRequest) returns (GetPriceReply);
}
message GetPriceRequest { string event_id = 1; string category = 2; }
message GetPriceReply  { string currency = 1; int64 amount_minor = 2; }   // money in minor units
```

```csharp
builder.Services.AddGrpc();
app.MapGrpcService<PricingGrpcService>();
```

Know when to use it. TicketHub doesn't need it yet, but it's a common interview question.

---

## Interview questions

<details><summary><b>1. Liveness vs readiness probes: what goes into each?</b></summary>

Liveness checks only that the process can respond (no dependencies), so the orchestrator restarts genuinely stuck processes. Readiness includes critical dependencies, so an instance is taken out of rotation until it can serve traffic. Checking dependencies in liveness causes cascading restarts during an outage.
</details>

<details><summary><b>2. Why use <code>IHttpClientFactory</code>?</b></summary>

It pools and rotates `HttpMessageHandler`s, which avoids socket exhaustion from per-request clients and the stale DNS of singleton clients. It also centralizes configuration and lets you add resilience handlers and `DelegatingHandler`s (auth, logging).
</details>

<details><summary><b>3. What is CORS and what isn't it?</b></summary>

A browser-enforced policy that lets a server declare which origins may call it from scripts, using preflight `OPTIONS` requests for non-simple requests. It isn't authentication or authorization: non-browser clients ignore it.
</details>

<details><summary><b>4. SSE vs WebSockets/SignalR?</b></summary>

SSE is one-way server → client over HTTP, simple, with auto-reconnect. WebSockets/SignalR are bidirectional, with groups and richer client SDKs. Choose SSE for notifications and SignalR for interactive scenarios.
</details>

<details><summary><b>5. How does rate limiting behave with multiple instances?</b></summary>

The in-process limiter counts per instance, so the effective limit is multiplied by the instance count. Use a gateway or a distributed (Redis) limiter for global limits.
</details>

---

## Exercises

1. Add the `per-user` and `holds` rate limiting policies. Write a small load script, verify the 429 responses and the `Retry-After` header, and check that anonymous users are partitioned by IP.
2. Add liveness and readiness checks, stop the Postgres container, and observe that liveness stays healthy while readiness fails.
3. Add a typed `PaymentGatewayClient` with the standard resilience handler. Use a fake server that fails 50% of the time and watch the retries in the logs.
4. Build the live seat map with SSE: after a hold succeeds, publish `SeatChanged`, and consume it with a 10-line HTML page that uses `EventSource`.

## Self-check

- [ ] I can configure rate limiting, CORS, health checks and compression, and explain their placement.
- [ ] I use `IHttpClientFactory` with resilience and know the retry and idempotency caveat.
- [ ] I can choose between polling, SSE, SignalR and gRPC.

## Further reading

- Microsoft Learn: *Rate limiting middleware*, *Enable CORS*, *Health checks*, *IHttpClientFactory*, *Build resilient HTTP apps*, *SignalR overview*, *gRPC on .NET*.
