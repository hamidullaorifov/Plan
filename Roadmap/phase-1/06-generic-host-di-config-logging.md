# 6. Generic Host: DI, Configuration, Options, Logging and Background Services

> Phase 1 · Week 8 · [← Phase index](README.md)

**Why this matters:** Every modern .NET app (ASP.NET Core, worker services, Azure Functions, console tools built on the host) is assembled the same way. Lifetime bugs, configuration mistakes and noisy logs are daily sources of production issues. *"Explain the 3 DI lifetimes and the captive dependency problem"* is a near-certain interview question.

---

## 6.1 The Generic Host

The host wires together **dependency injection, configuration, logging, environment and application lifetime** (startup and graceful shutdown).

```csharp
// Worker / console app (.NET 7+)
var builder = Host.CreateApplicationBuilder(args);
builder.Services.AddHostedService<CleanupWorker>();
using var host = builder.Build();
await host.RunAsync();

// Web app: same concepts plus HTTP
var webBuilder = WebApplication.CreateBuilder(args);
webBuilder.Services.AddScoped<IOrderService, OrderService>();
var app = webBuilder.Build();
app.MapGet("/health", () => "ok");
app.Run();
```

What `CreateApplicationBuilder` / `CreateBuilder` set up for you:

| Concern | Default |
|---|---|
| Configuration | `appsettings.json` → `appsettings.{Environment}.json` → user secrets (Development only) → environment variables → command-line arguments (**later sources win**) |
| Logging | console, debug and EventSource providers; levels taken from `Logging:LogLevel` |
| DI | `Microsoft.Extensions.DependencyInjection`, with scope validation turned on in Development |
| Environment | from `DOTNET_ENVIRONMENT` / `ASPNETCORE_ENVIRONMENT`; `Production` if unset |
| Lifetime | Ctrl+C or SIGTERM triggers a graceful shutdown that stops hosted services |

---

## 6.2 Dependency injection

### Why DI exists

A class declares *what it needs* through its constructor, and the container decides *which implementation* and *how long it lives*. This gives you loose coupling, testability (you can pass fakes) and a single place where the application is composed.

```csharp
builder.Services.AddSingleton(TimeProvider.System);
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.AddTransient<IPriceCalculator, PriceCalculator>();
builder.Services.AddScoped<OrderService>();                       // concrete type, no interface

builder.Services.AddSingleton<INotificationChannel, EmailChannel>(); // multiple implementations:
builder.Services.AddSingleton<INotificationChannel, SmsChannel>();   // inject IEnumerable<INotificationChannel>

builder.Services.AddKeyedSingleton<IPaymentGateway, StripeGateway>("stripe");   // keyed services (.NET 8)
builder.Services.AddKeyedSingleton<IPaymentGateway, AdyenGateway>("adyen");

public sealed class CheckoutService([FromKeyedServices("stripe")] IPaymentGateway gateway) { /* ... */ }
```

### The three lifetimes

| Lifetime | One instance per… | Typical use | ⚠️ Watch out |
|---|---|---|---|
| **Singleton** | application | stateless services, caches, `HttpClient` handlers, `TimeProvider`, configuration | must be **thread-safe**; must not depend on scoped services |
| **Scoped** | **scope**, which in ASP.NET Core means **one HTTP request** | `DbContext`, unit of work, per-request user context | don't resolve it from the root provider; don't use it after the scope is disposed |
| **Transient** | every resolution | lightweight stateless helpers | disposable transients resolved from the root are kept until **app shutdown**, which is a leak |

### Captive dependency: the lifetime bug

```csharp
builder.Services.AddSingleton<PriceCache>();          // lives forever
builder.Services.AddScoped<AppDbContext>();           // should live for one request

public sealed class PriceCache(AppDbContext db) { /* ... */ }  // ❌ the singleton captures a scoped DbContext
```

What goes wrong:

1. The first request creates `PriceCache` and a `DbContext`. That one `DbContext` is now **captured forever**.
2. Every later request uses the *same* context concurrently, and `DbContext` isn't thread-safe, so you get exceptions.
3. Its change tracker grows forever, which is a memory leak, and it holds stale data.

**Detect it:** scope validation throws *"Cannot consume scoped service … from singleton …"*. It's on in Development by default. Turn it on everywhere:

```csharp
builder.Host.UseDefaultServiceProvider(options =>
{
    options.ValidateScopes = true;   // catches captive dependencies
    options.ValidateOnBuild = true;  // checks at startup that every registration can be constructed
});
```

**Fix it:** align the lifetimes (make `PriceCache` scoped), or create a scope explicitly when a singleton truly needs scoped work:

```csharp
public sealed class PriceCache(IServiceScopeFactory scopeFactory)
{
    public async Task RefreshAsync(CancellationToken ct)
    {
        await using var scope = scopeFactory.CreateAsyncScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        // ... use db; it's disposed with the scope
    }
}
```

**The general rule:** a service may depend only on services with an **equal or longer** lifetime.

| Consumer → dependency | Singleton | Scoped | Transient |
|---|---|---|---|
| **Singleton** | ✅ | ❌ captive | ⚠️ the transient effectively becomes a singleton |
| **Scoped** | ✅ | ✅ | ✅ |
| **Transient** | ✅ | ✅ (within its scope) | ✅ |

### More DI rules

- **The container disposes what it creates.** Never dispose an injected dependency yourself.
- **Avoid the service locator:** don't inject `IServiceProvider` into business classes just to call `GetService`. It hides dependencies. The exceptions are factories and the `IServiceScopeFactory` scope pattern shown above.
- **Watch constructor size.** More than about 5 dependencies is a sign the class does too much.
- **Typed HTTP clients:** register them with `builder.Services.AddHttpClient<GitHubClient>(c => c.BaseAddress = new("https://api.github.com"))`. Handlers are pooled, which prevents socket exhaustion (more in Phase 2).
- **`TryAdd*`** (`TryAddSingleton`, …) registers a service only if it isn't registered yet. Library authors use it so apps can override their defaults.

---

## 6.3 Configuration

```json
// appsettings.json: non-secret defaults, committed to Git
{
  "Ticketing": {
    "HoldDurationMinutes": 10,
    "MaxSeatsPerHold": 10
  },
  "Smtp": {
    "Host": "localhost",
    "Port": 1025,
    "From": "noreply@tickethub.local"
  }
}
```

| Source | Use for | Key format |
|---|---|---|
| `appsettings.json` | defaults | `"Smtp": { "Port": 1025 }` |
| `appsettings.Production.json` | per-environment overrides (no secrets) | same |
| **User secrets** (`dotnet user-secrets set "Smtp:Password" "..."`) | **local development secrets** | `Smtp:Password` |
| **Environment variables** | containers, CI and cloud settings | `Smtp__Password` (a **double underscore** replaces `:`) |
| Command line | one-off overrides | `--Smtp:Port=2525` |
| Key Vault / Secrets Manager / App Configuration | production secrets and feature flags | Phase 4 |

> ❌ **Never commit secrets** to `appsettings*.json`. Configuration files end up in Git history, container images and logs.

---

## 6.4 The options pattern

Bind a configuration section to a **strongly-typed class**, validate it, and fail fast at startup:

```csharp
using System.ComponentModel.DataAnnotations;

public sealed class SmtpOptions
{
    public const string SectionName = "Smtp";

    [Required] public required string Host { get; init; }
    [Range(1, 65535)] public int Port { get; init; } = 25;
    [Required, EmailAddress] public required string From { get; init; }
    public string? Password { get; init; }
}

builder.Services.AddOptions<SmtpOptions>()
    .BindConfiguration(SmtpOptions.SectionName)
    .ValidateDataAnnotations()             // [Required], [Range], ...
    .Validate(o => o.Port != 25 || o.Host == "localhost", "Port 25 is allowed only for local relays.")
    .ValidateOnStart();                    // fail at startup, not on the first email at 3 AM
```

> In worker or console apps, `ValidateDataAnnotations` needs the `Microsoft.Extensions.Options.DataAnnotations` package. ASP.NET Core includes it. For AOT-friendly validation, use the `[OptionsValidator]` source generator (.NET 8+). .NET 11 adds **asynchronous** startup validation (`IAsyncStartupValidator`).

### Which interface to inject

| Interface | Lifetime | Reloads on config change? | Use when |
|---|---|---|---|
| `IOptions<T>` | singleton | ❌ (read once) | most cases: settings that don't change at runtime |
| `IOptionsSnapshot<T>` | **scoped** | ✅ per request/scope | per-request consistent values in web apps. **Can't be injected into singletons** |
| `IOptionsMonitor<T>` | singleton | ✅ live: `CurrentValue` plus `OnChange` | singletons or background services that must react to changes |

```csharp
public sealed class EmailSender(IOptions<SmtpOptions> options, ILogger<EmailSender> logger) : IEmailSender
{
    private readonly SmtpOptions _smtp = options.Value;
    // ...
}
```

**Why options instead of `IConfiguration["Smtp:Host"]` everywhere?** You get type safety, validation, testability (`Options.Create(new SmtpOptions { ... })`), discoverability, and no magic strings scattered through the code.

---

## 6.5 Logging

### Structured logging: templates, not interpolation

```csharp
// ✅ A message template: OrderId and Total become queryable fields in Seq, Elastic, Application Insights, ...
logger.LogInformation("Order {OrderId} placed with total {Total}", order.Id, order.Total);

// ❌ Interpolation: one opaque string; allocates even when the level is disabled; analyzer CA2254
logger.LogInformation($"Order {order.Id} placed with total {order.Total}");
```

### Log levels: use them deliberately

| Level | Use for | Production default |
|---|---|---|
| `Trace` | very detailed internals, possibly sensitive | off |
| `Debug` | developer diagnostics | off |
| `Information` | business events: "Order placed", "Payment captured" | **on** |
| `Warning` | unexpected but handled: a retry, a fallback, slow upstream | on |
| `Error` | the current operation failed | on, and alerts on rate |
| `Critical` | the app can't continue: database unreachable at startup, data corruption | on, and pages someone |

```json
"Logging": {
  "LogLevel": {
    "Default": "Information",
    "Microsoft.AspNetCore": "Warning",
    "Microsoft.EntityFrameworkCore.Database.Command": "Warning"
  }
}
```

### High-performance logging with the source generator

```csharp
internal static partial class Log
{
    [LoggerMessage(Level = LogLevel.Information, Message = "Order {OrderId} placed by customer {CustomerId}")]
    public static partial void OrderPlaced(this ILogger logger, Guid orderId, Guid customerId);

    [LoggerMessage(Level = LogLevel.Warning, Message = "Seat hold {HoldId} expired before payment")]
    public static partial void HoldExpired(this ILogger logger, Guid holdId);
}

logger.OrderPlaced(order.Id, order.CustomerId);   // no boxing, no allocation when the level is disabled
```

### Scopes: attach context to every log line inside a block

```csharp
using (logger.BeginScope(new Dictionary<string, object> { ["OrderId"] = order.Id, ["TenantId"] = tenantId }))
{
    logger.LogInformation("Charging card");     // carries OrderId and TenantId
    await gateway.ChargeAsync(order, ct);
    logger.LogInformation("Card charged");
}
```

### Logging rules

- **Log exceptions with the exception overload:** `logger.LogError(ex, "Payment failed for order {OrderId}", id)`. Never just log `ex.Message`.
- **Log once, at the boundary that handles the error.** Log-and-rethrow at every layer produces five copies of the same error.
- **Never log secrets or PII:** passwords, tokens, card numbers, full emails or addresses (GDPR).
- **Use correlation IDs.** OpenTelemetry trace and span IDs are attached automatically in Phase 4.

---

## 6.6 Background services

```csharp
public sealed class ExpiredHoldsCleanupWorker(
    IServiceScopeFactory scopeFactory,
    TimeProvider clock,
    ILogger<ExpiredHoldsCleanupWorker> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromMinutes(1), clock);
        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            try
            {
                await using var scope = scopeFactory.CreateAsyncScope();             // a new scope per iteration
                var holds = scope.ServiceProvider.GetRequiredService<ISeatHoldRepository>();
                int released = await holds.ReleaseExpiredAsync(clock.GetUtcNow(), stoppingToken);
                if (released > 0)
                    logger.LogInformation("Released {Count} expired seat holds", released);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                logger.LogError(ex, "Expired-holds cleanup failed; will retry next tick");  // don't kill the worker
            }
        }
    }
}

builder.Services.AddHostedService<ExpiredHoldsCleanupWorker>();
```

The queue consumer that pairs with the `EmailQueue` from [guide 3](03-async-and-concurrency.md#313-async-streams-and-channelt):

```csharp
public sealed class EmailSenderWorker(EmailQueue queue, IServiceScopeFactory scopes, ILogger<EmailSenderWorker> logger)
    : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var message in queue.ReadAllAsync(stoppingToken))
        {
            try
            {
                await using var scope = scopes.CreateAsyncScope();
                var sender = scope.ServiceProvider.GetRequiredService<IEmailSender>();
                await sender.SendAsync(message, stoppingToken);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                logger.LogError(ex, "Failed to send email {MessageId}", message.Id);
            }
        }
    }
}
```

**Key facts:**

- `BackgroundService` is registered as a **singleton**, so create a **scope** to use scoped services such as `DbContext`.
- **An unhandled exception in `ExecuteAsync` stops the whole host** (the default since .NET 6, `BackgroundServiceExceptionBehavior.StopHost`). Catch errors per iteration.
- **Honor `stoppingToken`** for graceful shutdown. The host waits up to `HostOptions.ShutdownTimeout`, which is 30 seconds by default.
- Before .NET 10, the synchronous part of `ExecuteAsync` (before its first `await`) blocked host startup. .NET 10 runs `ExecuteAsync` as a background task. Either way, don't put long synchronous work at the start.
- An in-memory queue **loses messages on restart or crash**. For durable work, use a broker or an outbox (Phase 3).

---

## Interview questions

<details><summary><b>1. Explain singleton, scoped and transient. Which would you use for a DbContext and why?</b></summary>

Singleton means one instance per app, scoped means one per scope (one per request in ASP.NET Core), and transient means a new instance per resolution. Register `DbContext` as scoped: it isn't thread-safe and it tracks changes for one unit of work, which naturally matches a request.
</details>

<details><summary><b>2. What is a captive dependency? How do you detect and fix one?</b></summary>

It's a longer-lived service holding a shorter-lived one, such as a singleton holding a scoped `DbContext`. The short-lived service is effectively promoted to the longer lifetime, which causes concurrency and staleness bugs. Detect it with `ValidateScopes` and `ValidateOnBuild`. Fix it by aligning lifetimes or creating a scope through `IServiceScopeFactory`.
</details>

<details><summary><b>3. IOptions vs IOptionsSnapshot vs IOptionsMonitor?</b></summary>

`IOptions` is a singleton read once. `IOptionsSnapshot` is scoped and re-read per scope, so it can't go into singletons. `IOptionsMonitor` is a singleton with live `CurrentValue` and `OnChange`. Add `ValidateOnStart` so bad configuration fails fast.
</details>

<details><summary><b>4. What is the configuration precedence in ASP.NET Core?</b></summary>

`appsettings.json` < `appsettings.{Env}.json` < user secrets (Development) < environment variables < command line. Later sources override earlier ones. Environment variables use `__` as the separator.
</details>

<details><summary><b>5. Why use message templates instead of string interpolation in logs?</b></summary>

Templates keep the parameters as structured fields, so you can query `OrderId = X`. They also avoid formatting and allocating when the level is disabled, and they keep one stable message template for grouping. The source generator (`[LoggerMessage]`) is even faster.
</details>

<details><summary><b>6. How do you use a DbContext inside a BackgroundService?</b></summary>

Inject `IServiceScopeFactory`, create an async scope per unit of work, resolve the `DbContext` from the scope, and dispose the scope afterwards. Never inject `DbContext` directly into the hosted service, because hosted services are singletons.
</details>

<details><summary><b>7. What happens if a BackgroundService throws an unhandled exception?</b></summary>

Since .NET 6, the default `StopHost` behavior logs the error and stops the application. Catch exceptions per iteration, filtering out `OperationCanceledException` during shutdown.
</details>

---

## Exercises

1. **Captive dependency demo:** in a minimal API, register a singleton that depends on a scoped service. Run it in `Production` with validation off and observe the bug, for example by printing the scoped instance's hash code per request. Then turn on `ValidateScopes` and watch it fail at startup.
2. **Options:** add `TicketingOptions` (`HoldDurationMinutes`, `MaxSeatsPerHold`) with validation and `ValidateOnStart`, and prove that a bad value stops startup.
3. **Overrides:** override `Ticketing__MaxSeatsPerHold` through an environment variable and through the command line, and confirm the precedence.
4. **Logging:** convert 10 interpolated log calls at work to templates or `[LoggerMessage]`, and enable analyzers CA2254 and CA1848.
5. **Worker:** build the `ExpiredHoldsCleanupWorker` with a fake repository. Use `FakeTimeProvider` to test that it runs once per tick (see [guide 8](08-testing.md)).

## Self-check

- [ ] I can explain the three lifetimes and the captive dependency, and I know how to detect and fix it.
- [ ] I know the configuration source order and never commit secrets.
- [ ] I can bind, validate and fail fast with options, and I know which options interface to inject.
- [ ] I write structured logs at the right levels, with no PII.
- [ ] I can write a robust `BackgroundService` with scopes, error handling and cancellation.

## Further reading

- Microsoft Learn: *Dependency injection in .NET* and its guidelines, *Configuration in .NET*, *Options pattern*, *Logging in .NET*, *Compile-time logging source generation*, *Worker services*.
- *Dependency Injection Principles, Practices, and Patterns* (van Deursen and Seemann), chapters 1–4 and 8 (lifetime management).
- Andrew Lock's blog (*.NET Escapades*): his posts on options validation and DI scopes.
