# 7. Observability: OpenTelemetry, Logs, Metrics, Traces and SLOs

> Phase 4 · Weeks 8–9 · [← Phase index](README.md)
> Week 8 covers sections 7.1–7.5 (instrumentation); week 9 covers 7.6–7.9 (SLOs, alerting, diagnostics).

**Why this matters:** "Investigated and resolved a production issue using logs, traces or metrics" is a Middle checkpoint item. You can't debug a distributed system with a debugger. You debug it with **telemetry you added before the incident**. OpenTelemetry (OTel) is now the standard, and .NET has first-class support for it.

---

## 7.1 The three signals

| Signal | Answers | .NET API | Example |
|---|---|---|---|
| **Logs** | *what* happened, in detail | `ILogger` | "Payment for order 42 declined: insufficient funds" |
| **Metrics** | *how much / how often*: cheap aggregates over time | `System.Diagnostics.Metrics` (`Meter`) | request rate, error rate, p95 latency, queue depth |
| **Traces** | *where the time went* across services | `System.Diagnostics.Activity` (`ActivitySource`) | HTTP → database → broker → worker, with timings for each span |

**They connect:** a log line carries the `TraceId` and `SpanId` of the current activity, so you can jump from an alert (metric) → a slow trace → the exact error log.

---

## 7.2 Setting up OpenTelemetry in ASP.NET Core

```csharp
// Packages: OpenTelemetry.Extensions.Hosting, OpenTelemetry.Instrumentation.AspNetCore,
//           OpenTelemetry.Instrumentation.Http, OpenTelemetry.Instrumentation.Runtime,
//           OpenTelemetry.Exporter.OpenTelemetryProtocol
using OpenTelemetry.Logs;
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

builder.Logging.AddOpenTelemetry(logging =>
{
    logging.IncludeFormattedMessage = true;
    logging.IncludeScopes = true;
});

builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource
        .AddService(serviceName: "shop-api", serviceVersion: "1.4.2")
        .AddAttributes([new("deployment.environment.name", builder.Environment.EnvironmentName)]))
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddSource("Shop.*"))                  // your own ActivitySources
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()
        .AddMeter("Shop.*"))                   // your own Meters
    .UseOtlpExporter();                        // all signals → OTEL_EXPORTER_OTLP_ENDPOINT
```

- **OTLP** is the vendor-neutral protocol. Point it at the Aspire dashboard locally, or at an OpenTelemetry Collector, Grafana (Tempo/Loki/Prometheus), Azure Monitor or Datadog in production, **without code changes**.
- Configuration through standard environment variables: `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_SERVICE_NAME`, `OTEL_RESOURCE_ATTRIBUTES`.
- **Database spans:** Npgsql has built-in tracing (`AddNpgsql()` from Npgsql.OpenTelemetry), SQL Server via `SqlClient` instrumentation, and EF Core via its instrumentation package. Many Azure SDKs and RabbitMQ.Client 7 emit activities you can enable by source name.
- With Aspire, `AddServiceDefaults()` configures most of this for you ([guide 2](02-aspire.md)).

---

## 7.3 Custom traces and metrics

```csharp
using System.Diagnostics;
using System.Diagnostics.Metrics;

public sealed class CheckoutTelemetry
{
    public static readonly ActivitySource Source = new("Shop.Checkout");

    private readonly Counter<long> _ordersPlaced;
    private readonly Histogram<double> _checkoutDuration;

    public CheckoutTelemetry(IMeterFactory meterFactory)            // DI-friendly and testable
    {
        var meter = meterFactory.Create("Shop.Checkout");
        _ordersPlaced = meter.CreateCounter<long>("shop.orders.placed", unit: "{order}");
        _checkoutDuration = meter.CreateHistogram<double>("shop.checkout.duration", unit: "s");
    }

    public void OrderPlaced(string paymentMethod) =>
        _ordersPlaced.Add(1, new KeyValuePair<string, object?>("payment.method", paymentMethod));

    public void CheckoutCompleted(TimeSpan elapsed, string outcome) =>
        _checkoutDuration.Record(elapsed.TotalSeconds, new KeyValuePair<string, object?>("outcome", outcome));
}

// In the handler
using var activity = CheckoutTelemetry.Source.StartActivity("Checkout.PlaceOrder");
activity?.SetTag("order.id", order.Id);
activity?.SetTag("order.line_count", order.Lines.Count);
// on failure:
activity?.SetStatus(ActivityStatusCode.Error, error.Code);
```

**Naming and cardinality rules:**

- Use dotted, lowercase names that follow OTel semantic conventions where they exist (`http.server.request.duration`).
- **Never put unbounded values in metric tags** (user IDs, order IDs, raw URLs). Each unique combination creates a new time series, and costs and memory explode. IDs belong on **spans and logs**, not on metrics.
- Prefer **histograms** for durations, so you can compute percentiles. Averages hide tail latency.

---

## 7.4 Propagating context through messages

HTTP calls propagate `traceparent` automatically. **Messaging doesn't unless you (or your library) do it**:

```csharp
using OpenTelemetry;
using OpenTelemetry.Context.Propagation;

// Producer: inject the current context into the message headers
var headers = new Dictionary<string, string>();
Propagators.DefaultTextMapPropagator.Inject(
    new PropagationContext(Activity.Current?.Context ?? default, Baggage.Current),
    headers,
    (carrier, key, value) => carrier[key] = value);
// ...attach the headers to the message (RabbitMQ BasicProperties.Headers, ServiceBusMessage.ApplicationProperties)

// Consumer: extract the context and start a consumer span linked to the producer's trace
var parent = Propagators.DefaultTextMapPropagator.Extract(
    default,
    receivedHeaders,
    (carrier, key) => carrier.TryGetValue(key, out var value) ? [value] : []);
Baggage.Current = parent.Baggage;
using var activity = CheckoutTelemetry.Source.StartActivity(
    "order-paid process", ActivityKind.Consumer, parent.ActivityContext);
```

Result: **one trace** from `POST /checkout` → database → outbox relay → broker → worker → email, which is the single most useful debugging view in a distributed system. Messaging libraries (MassTransit, Wolverine, the Azure SDKs) usually handle this for you.

---

## 7.5 Logging strategy in production

| Rule | Why |
|---|---|
| Structured logs with message templates (Phase 1, guide 6) | queryable fields |
| `Information` for business events; `Warning` for handled anomalies; `Error` for failed operations | signal over noise |
| **No PII or secrets** (or mask them deliberately) | GDPR, security |
| Log exceptions **once**, at the boundary that handles them | no duplicate noise |
| Correlation: `TraceId`/`SpanId` on every log (automatic with OTel logging) | jump from a trace to its logs |
| Sampling and retention policies | cost control |
| Health-check and static-file noise filtered out | cost and readability |

---

## 7.6 SLIs, SLOs and error budgets

| Term | Meaning | Example |
|---|---|---|
| **SLI** (indicator) | a measured ratio of good events to all events | the share of `POST /checkout` requests answered successfully in under 500 ms |
| **SLO** (objective) | the target for the SLI over a window | 99.5% over 30 days |
| **Error budget** | the allowed failure: 1 − SLO | 0.5% of requests ≈ 3.6 h of full outage per 30 days |
| **SLA** (agreement) | a contractual promise with penalties, looser than the SLO | 99.0% |

| SLO | Allowed downtime per 30 days |
|---|---|
| 99% | 7.2 h |
| 99.5% | 3.6 h |
| 99.9% | 43.2 min |
| 99.99% | 4.3 min |

**Why bother?** SLOs turn reliability into a **decision tool**. When the error budget is healthy, ship faster. When it's burning, prioritize reliability work. They also tell you what to alert on.

---

## 7.7 Alerting that people trust

- **Alert on symptoms users feel** (error rate, latency, SLO burn rate), not on causes (CPU at 80%). Use cause-level metrics for diagnosis on dashboards.
- **Burn-rate alerts:** page when the error budget is being consumed fast (for example, 2% of the monthly budget in 1 hour). Open a ticket when it's burning slowly (10% in 3 days).
- Every page must be **actionable** and link to a **runbook** ([guide 9](09-production-readiness.md)).
- Track noisy alerts and delete or tune them. **Alert fatigue makes people miss real incidents.**

### A minimal service dashboard ("RED" + saturation)

| Panel | Metric |
|---|---|
| **R**ate | requests/second per endpoint |
| **E**rrors | the share of 5xx responses (and of business failures) |
| **D**uration | p50, p95 and p99 latency per endpoint |
| Saturation | CPU, memory, GC pause time, thread-pool queue length, DB connection pool usage, queue depth |
| Dependencies | latency and error rate per downstream (database, payments, broker) |

---

## 7.8 Diagnostics in production

| Need | Tool |
|---|---|
| Live counters (GC, thread pool, exceptions) | `dotnet-counters`, or the same runtime metrics through OTel |
| A CPU profile of a slow pod | `dotnet-trace` / **`dotnet-monitor`** (a sidecar exposing an HTTP API for traces, dumps and logs) |
| A memory leak | `dotnet-gcdump` / `dotnet-dump` (Phase 1, guide 2), collected through `dotnet-monitor` in containers |
| Crash analysis | `DOTNET_DbgEnableMiniDump=1` (or .NET 11's in-process crash reports) + `dotnet-dump analyze` |
| "Why is this request slow?" | **its trace**: find the widest span |

---

## 7.9 An incident investigation flow

1. **Alert:** the checkout error rate SLO is burning fast.
2. **Dashboard:** errors began at 14:05, only on `POST /checkout`; payments-provider latency jumped at the same time.
3. **Traces:** filter failed checkout traces. The `payments.charge` span times out at 2 s, and the circuit breaker opened at 14:07.
4. **Logs** for one trace ID: `TaskCanceledException` from the payments `HttpClient`; the provider's status page shows an incident.
5. **Mitigate:** enable the kill switch to show "pay later"; communicate the status.
6. **Follow-up:** write a postmortem ([guide 9](09-production-readiness.md)), tune the timeout, and add an alert on the provider's latency.

---

## Interview questions

<details><summary><b>1. Logs vs metrics vs traces?</b></summary>

Logs are detailed discrete events. Metrics are cheap numeric aggregates over time, used for alerting and trends. Traces show a request's path and timings across services. They're correlated through trace IDs: alert on metrics, find the trace, read the logs.
</details>

<details><summary><b>2. How do you trace a request across a message broker?</b></summary>

Inject the W3C trace context (`traceparent` plus baggage) into the message headers when publishing, then extract it in the consumer and start a consumer span with the extracted parent context. Many messaging libraries do this automatically.
</details>

<details><summary><b>3. What's an SLO, and what's an error budget used for?</b></summary>

An SLO is the target reliability for a user-facing indicator over a window (for example, 99.5% of checkouts succeed within 500 ms over 30 days). The error budget (1 − SLO) quantifies acceptable unreliability: while it's available, the team ships features; when it's exhausted, reliability work takes priority. Burn-rate alerts are based on it.
</details>

<details><summary><b>4. Why not put user IDs in metric tags?</b></summary>

High cardinality: every unique value creates a separate time series, which explodes storage, memory and cost and slows queries. Put IDs on spans and logs instead.
</details>

<details><summary><b>5. What should you alert on?</b></summary>

User-facing symptoms: error rate, latency and SLO burn rate, with actionable runbooks. Cause metrics (CPU, memory) go on dashboards for diagnosis, or get alerts only when they reliably predict user impact.
</details>

---

## Exercises

1. Add OpenTelemetry to a lab API and worker, send it to the Aspire dashboard or a Grafana stack, and produce **one trace** that spans HTTP → database → broker → worker ([Lab 8](10-labs.md#lab-8-end-to-end-trace)).
2. Add one business counter and one histogram, and build a RED dashboard.
3. Define one SLO for a lab endpoint, add a burn-rate alert, and trigger it with injected failures ([Lab 9](10-labs.md#lab-9-slo-and-alert)).
4. At work: find one service without a dashboard or alert, and add one.

## Self-check

- [ ] I can configure OpenTelemetry tracing, metrics and logs with OTLP export.
- [ ] I can create custom spans and metrics with sensible names and low-cardinality tags.
- [ ] I can propagate trace context through messaging.
- [ ] I can define SLIs, SLOs and burn-rate alerts, and build a RED dashboard.
- [ ] I can follow an investigation from an alert to a trace to logs.

## Further reading

- Microsoft Learn: *.NET observability with OpenTelemetry*, *Create metrics*, *Distributed tracing instrumentation*.
- opentelemetry.io: *.NET getting started*, *semantic conventions*.
- Google, *Site Reliability Engineering* and *The Site Reliability Workbook*, the chapters on SLOs and alerting (free online).
- *Observability Engineering* (Majors, Fong-Jones, Miranda).
