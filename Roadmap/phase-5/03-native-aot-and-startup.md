# 3. Native AOT, Trimming and Startup

> Phase 5 · Week 5 · [← Phase index](README.md)

**Why this matters:** Some workloads care about **startup time and memory** more than peak throughput: serverless functions, scale-to-zero containers, CLIs, sidecars and dense container hosting. Native AOT compiles your app ahead of time into a self-contained native executable. It starts in milliseconds and uses much less memory, but it constrains which libraries and features you can use.

---

## 3.1 The options

| Option | What it does | Startup | Peak throughput | Compatibility |
|---|---|---|---|---|
| **JIT (default)** | IL compiled at runtime; tiered compilation + dynamic PGO | slowest | **best long-term** (PGO) | everything |
| **ReadyToRun** (`PublishReadyToRun`) | pre-compiles IL to native code, keeping the JIT for re-optimization | faster | ≈ JIT after warm-up | everything; larger files |
| **Trimming** (`PublishTrimmed`) | removes unused code from a self-contained app | — | — | reflection-heavy code can break |
| **Native AOT** (`PublishAot`) | a fully native executable, no JIT, trimmed | **fastest** (ms) | good, sometimes slightly lower than PGO'd JIT | **restricted** (3.3) |

---

## 3.2 Publishing a Native AOT minimal API

```powershell
dotnet new webapiaot -n Shop.Notifications -f net10.0
```

```xml
<PropertyGroup>
  <PublishAot>true</PublishAot>
  <InvariantGlobalization>true</InvariantGlobalization>   <!-- if you don't need culture data -->
</PropertyGroup>
```

```csharp
var builder = WebApplication.CreateSlimBuilder(args);     // fewer default features, AOT-friendly

builder.Services.ConfigureHttpJsonOptions(o =>
    o.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default));

var app = builder.Build();

app.MapPost("/notifications", (NotificationRequest request) =>
    TypedResults.Accepted($"/notifications/{Guid.NewGuid()}"));

app.Run();

public sealed record NotificationRequest(string To, string Template);

[JsonSerializable(typeof(NotificationRequest))]
internal partial class AppJsonContext : JsonSerializerContext;
```

```powershell
dotnet publish -c Release -r linux-x64        # needs the platform's native toolchain; a Docker build stage works well
```

Minimal APIs use the **Request Delegate Generator** under AOT: route handlers are compiled at build time instead of being generated with reflection at runtime.

---

## 3.3 What breaks, and how you find out

The compiler reports **trim and AOT warnings** (`IL2026`, `IL2070`, `IL3050`, …) for code that relies on reflection or runtime code generation. **Treat them as errors**: a warning means it may fail at runtime.

| Feature | AOT status (verify for your version) |
|---|---|
| Minimal APIs, gRPC, health checks, SignalR (JSON protocol with source generation), Kestrel | ✅ supported |
| **MVC controllers, Razor Pages, Blazor Server** | ❌ not supported |
| System.Text.Json | ✅ **with source generation only** |
| EF Core | ⚠️ partial or experimental (precompiled queries and compiled models); check the current docs. Dapper with its AOT generator, or raw ADO.NET, are safer |
| Reflection-based libraries (some mappers, DI scanning, older serializers) | ❌ or ⚠️: look for AOT-compatible versions or source-generator alternatives |
| `dynamic`, `Assembly.LoadFrom`, `Reflection.Emit` | ❌ |

**Libraries mark compatibility** with `<IsAotCompatible>true</IsAotCompatible>`. Test your dependency graph early with a throwaway `PublishAot` build.

---

## 3.4 Measuring the payoff

```powershell
# Startup: time until the first successful response
Measure-Command { Start-Process ./Shop.Notifications; do { Start-Sleep -Milliseconds 10 } until ((Invoke-WebRequest http://localhost:8080/health -SkipHttpErrorCheck).StatusCode -eq 200) }

# Memory under load: watch the working set (docker stats, kubectl top, or dotnet-counters for JIT builds)
```

| Measure | Why |
|---|---|
| Time to the first successful request | cold start for scale-to-zero and serverless |
| Working set at idle and under load | how many replicas fit per node; cost |
| Image size | pull time during scale-out |
| Throughput and p99 at steady state | confirms no regression compared with JIT + PGO |

A typical result for a small API: startup drops from hundreds of milliseconds to tens, and memory drops by roughly half or more. Peak throughput is similar or a bit lower. **Measure your own app**; these are indications only.

---

## 3.5 When to use which

| Workload | Recommendation |
|---|---|
| A long-running, high-throughput API | JIT (optionally ReadyToRun for faster warm-up) |
| Scale-to-zero containers, Azure Functions / AWS Lambda | Native AOT if the dependencies allow it; otherwise ReadyToRun |
| A CLI tool (`dotnet tool`) or sidecar | Native AOT |
| An app using MVC, EF Core-heavy features or reflection-based libraries | JIT; don't fight the framework |

---

## Interview questions

<details><summary><b>1. What is Native AOT, and what do you trade off?</b></summary>

Ahead-of-time compilation to a native, self-contained, trimmed executable with no JIT. You gain very fast startup, lower memory and smaller deployments. You lose reflection, dynamic code generation and some frameworks (MVC, parts of EF Core), and sometimes a little peak throughput compared with dynamic PGO.
</details>

<details><summary><b>2. How do you know whether your app is AOT-compatible?</b></summary>

Publish with `PublishAot` and treat every trim and AOT warning (IL2xxx/IL3xxx) as an error. Check that dependencies declare `IsAotCompatible`, replace reflection with source generators, and run the integration tests against the AOT binary.
</details>

<details><summary><b>3. ReadyToRun vs Native AOT?</b></summary>

ReadyToRun pre-compiles code but keeps the JIT and full compatibility: startup improves and steady-state stays the same. Native AOT removes the JIT entirely for the best startup and memory, at the cost of compatibility constraints.
</details>

---

## Exercises

1. Build a small notification worker or API as Native AOT. Compare startup, memory and image size with the JIT version ([Lab 3](11-labs.md#lab-3-aot-worker)).
2. Add a reflection-based library on purpose, read the trim warnings, and replace it with a source-generated alternative.
3. Try ReadyToRun on a larger API and measure the time to first request.

## Self-check

- [ ] I can publish a minimal API as Native AOT and fix trim and AOT warnings.
- [ ] I know which frameworks and libraries are AOT-compatible.
- [ ] I can measure startup, memory and throughput, and choose JIT, ReadyToRun or AOT per workload.

## Further reading

- Microsoft Learn: *Native AOT deployment*, *ASP.NET Core support for Native AOT*, *Prepare .NET libraries for trimming*.
- The .NET blog posts on AOT improvements in each release.
