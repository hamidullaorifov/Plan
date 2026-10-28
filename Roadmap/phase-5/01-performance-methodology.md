# 1. Performance Methodology and Profiling

> Phase 5 · Weeks 1–2 · [← Phase index](README.md)

**Why this matters:** Most "optimizations" are guesses, and most guesses are wrong. The skill that distinguishes a Strong Middle developer is **methodology**: define the goal, measure, find the actual bottleneck, change one thing, and prove the improvement. The tools differ by level (whole system, process, method), and choosing the right one saves days.

---

## 1.1 The loop

```
1. Define the goal       "p95 of GET /events < 150 ms at 500 RPS" (not "make it faster")
2. Measure the baseline  under realistic load and data
3. Find the bottleneck   with the RIGHT tool for the level (1.2)
4. Form one hypothesis   "the JSON serialization allocates 40 MB/s"
5. Change one thing
6. Measure again         same conditions; compare distributions, not single runs
7. Keep or revert        record the result (even negative results)
```

> **Amdahl's law:** speeding up a part that takes 5% of the time by 10× gains less than 5% overall. **Find where the time actually goes first.**

---

## 1.2 Choosing the tool by level

| Level | Question | Tool |
|---|---|---|
| **System** | which service or dependency is slow? | traces (OpenTelemetry), RED dashboards, APM (Phase 4) |
| **Load** | how does it behave at N requests per second? | k6, NBomber |
| **Process (live)** | is it GC, the thread pool, lock contention or exceptions? | `dotnet-counters`, OTel runtime metrics |
| **Process (CPU)** | which methods burn CPU? | `dotnet-trace` + PerfView/speedscope, dotTrace, the Visual Studio profiler |
| **Process (memory)** | what allocates, and what retains? | `dotnet-trace` (allocation sampling), `dotnet-gcdump`, dotMemory, the VS memory profiler |
| **Method (micro)** | which of two implementations is faster or allocates less? | **BenchmarkDotNet** |
| **Database** | which queries are expensive? | `pg_stat_statements`, Query Store, `EXPLAIN ANALYZE` (Phase 2) |

**Typical order:** traces tell you *where* (the database? the CPU in service X?) → counters tell you *what kind* (CPU? GC? waiting?) → a profiler tells you *which code* → BenchmarkDotNet validates the fix in isolation → a load test proves it at the system level.

---

## 1.3 Reading `dotnet-counters`

```powershell
dotnet-counters monitor -n Shop.Api --counters System.Runtime,Microsoft.AspNetCore.Hosting
```

| Pattern | Likely cause | Next step |
|---|---|---|
| High CPU, high request rate | real work; maybe inefficient code | CPU profile |
| **High latency, low CPU, growing thread-pool queue** | thread-pool starvation (sync-over-async) | dump the stacks (`dotnet-stack report -p <pid>`), find the blocking calls |
| High "% time in GC", frequent Gen2 GCs | allocation pressure, LOH churn | allocation profile ([guide 2](02-high-performance-code.md)) |
| Exception count in the thousands per second | exceptions as control flow | find the throw sites in the trace |
| Monotonically growing working set | a leak | heap snapshots (Phase 1, guide 2) |
| Low CPU, low queue, high latency | waiting on I/O: the database, HTTP | traces (the dependency is slow) |

---

## 1.4 CPU profiling with `dotnet-trace`

```powershell
dotnet-trace collect -n Shop.Api --profile cpu-sampling --duration 00:00:30 --format Speedscope
# Open the .speedscope.json at https://www.speedscope.app, or the .nettrace in PerfView / Visual Studio
```

**Reading a flame graph:**

- The **width** of a frame is the share of samples (time) that included it. Look for **wide plateaus**.
- The **top edge** shows where the CPU is actually spent (self time).
- Use the **left-heavy** or "sandwich" view to aggregate by function across call paths.
- Typical surprises: JSON serialization, regex, logging (string formatting at disabled levels), LINQ in hot loops, `ToList()` copies, exception throwing, reflection-based mappers.

Profile **in Release**, under **realistic load**, after warm-up. Tiered compilation and dynamic PGO mean the first seconds don't represent steady state.

---

## 1.5 Allocation profiling

```powershell
dotnet-trace collect -n Shop.Api --profile gc-verbose --duration 00:00:30
```

Open it in PerfView ("GC Heap Alloc Ignore Free (Coarse Sampling) Stacks") or the Visual Studio ".NET Object Allocation" tool to see **which call stacks allocate the most bytes**. Fix the top 1–3; it's usually a few hot spots.

---

## 1.6 BenchmarkDotNet, beyond the basics

```csharp
[MemoryDiagnoser]
[HideColumns("Error", "StdDev")]
public class SlugBenchmarks
{
    [Params(16, 256)]
    public int Length { get; set; }

    private string _input = "";

    [GlobalSetup]
    public void Setup() => _input = new string('A', Length) + " Jazz Night 2028!";

    [Benchmark(Baseline = true)]
    public string Naive() => SlugNaive.Create(_input);

    [Benchmark]
    public string SpanBased() => SlugFast.Create(_input);
}
```

```powershell
dotnet run -c Release -- --filter *Slug*                       # one runtime
dotnet run -c Release -f net10.0 -- --filter *Slug* --runtimes net10.0 net12.0   # compare runtimes (needs multi-targeting)
```

| Feature | Use |
|---|---|
| `[Params]` | test several input sizes; performance often changes shape with N |
| `[GlobalSetup]` | prepare inputs outside the measured code |
| `[MemoryDiagnoser]` | allocated bytes per operation, Gen0/1/2 collections |
| `[DisassemblyDiagnoser]` | see the JIT's assembly (bounds checks, inlining, vectorization) |
| `Baseline = true` + the `Ratio` column | relative comparison |
| `--runtimes` | compare .NET versions (useful for upgrade write-ups) |

**Pitfalls:** dead-code elimination (always *return* or *consume* the result), measuring setup inside the benchmark, benchmarking in Debug or under a debugger, micro-results that don't matter at the system level.

---

## 1.7 Reporting results

```markdown
## Perf: GET /api/events (2027-11-20)
Goal: p95 < 150 ms @ 500 RPS (staging, 2 replicas, 1.2 M events)

| Change                          | RPS  | p50   | p95    | Alloc/req | CPU  |
|---------------------------------|------|-------|--------|-----------|------|
| Baseline                        | 500  | 48 ms | 410 ms | 380 KB    | 82 % |
| + projection instead of Include | 500  | 22 ms | 160 ms | 120 KB    | 61 % |
| + STJ source generation         | 500  | 20 ms | 138 ms | 96 KB     | 55 % |

Conclusion: goal met. The remaining p95 is DB time on page 50+ → keyset pagination next (ticket SHOP-981).
```

Distributions (p50, p95, p99) and allocations matter more than averages. Always note the **environment and data volume**.

---

## Interview questions

<details><summary><b>1. A service is slow. Walk me through your investigation.</b></summary>

Define what "slow" means (which endpoint, which percentile, since when). Check the traces to see whether the time is spent in a dependency or in-process. If in-process, check the counters (CPU, GC, thread-pool queue, exceptions) to classify the problem. Then profile CPU or allocations, form one hypothesis, fix it, validate with BenchmarkDotNet and a load test, and report before/after numbers.
</details>

<details><summary><b>2. What can a flame graph tell you?</b></summary>

Where the CPU time goes: wide frames are expensive, and top-edge frames are self time. It shows unexpected hotspots such as serialization, regex, logging or exceptions, and which call paths lead to them.
</details>

<details><summary><b>3. Why might a micro-benchmark improvement not show up in production?</b></summary>

The optimized code may not be on the critical path (Amdahl's law). The real bottleneck may be I/O. Production data sizes and concurrency may differ. Or the benchmark may be flawed (dead-code elimination, measuring setup). Always confirm improvements with system-level metrics.
</details>

---

## Exercises

1. Profile a slow endpoint (at work or in a lab) end to end: trace → counters → CPU profile → fix → before/after table ([Lab 1](11-labs.md#lab-1-profile-a-slow-endpoint)).
2. Reproduce thread-pool starvation (Phase 1, Lab 4) and diagnose it from counters plus `dotnet-stack` alone, as if you didn't know the cause.
3. Benchmark one utility at work with `[Params]` across three input sizes and `[MemoryDiagnoser]`.

## Self-check

- [ ] I follow the goal → measure → hypothesis → change → re-measure loop.
- [ ] I pick the right tool for the system, process, method and database levels.
- [ ] I can read counters, flame graphs and allocation profiles.
- [ ] I report percentiles and allocations with the environment and data volume.

## Further reading

- Microsoft Learn: *.NET diagnostics tools* (dotnet-counters, dotnet-trace, dotnet-stack).
- Stephen Toub, the annual *"Performance Improvements in .NET N"* posts.
- *Pro .NET Benchmarking* (Andrey Akinshin, the author of BenchmarkDotNet).
- Brendan Gregg, *Systems Performance*, the methodology chapters.
