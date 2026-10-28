# Phase 1: C# and Runtime Depth, Plus Testing

> **Months 1–3 (Nov 16, 2026 – Jan 31, 2027)**
> **Goal:** Move from *using* .NET to *understanding* it, and be able to pass any C# fundamentals interview round.
> ← Back to the [main roadmap](../level-up.md)

---

## How to study each topic

Use the same loop for every guide:

1. **Read** the guide. Budget 1–2 hours per section, and don't skim the code.
2. **Run** every code sample. Change it, break it and watch what happens. Paste snippets into [SharpLab](https://sharplab.io) to see the lowered C#, the IL and the JIT assembly. It shows what the compiler *really* generates.
3. **Explain** the topic in your own words. Write a 5–10 sentence summary in a `notes/` folder (the Feynman technique). If you can't explain it simply, go back to step 1.
4. **Check** yourself by answering the interview questions *without* looking. Only then open the model answers.
5. **Apply** the topic to the TicketHub capstone in the same week.

---

## Topics

| # | Guide | Week | Effort |
|---|---|---|---|
| 1 | [The C# type system](01-type-system.md) | W1 | 6–8 h |
| 2 | [Memory, the GC and resource management](02-memory-and-gc.md) | W2 | 6–8 h |
| 3 | [Async, await and concurrency](03-async-and-concurrency.md) | W3–W4 | 12–16 h |
| 4 | [Collections and LINQ](04-collections-and-linq.md) | W5 | 6–8 h |
| 5 | [Modern C# (12 → 15)](05-modern-csharp.md) | W6 | 5–6 h |
| 6 | [Generic host: DI, configuration, options, logging, background services](06-generic-host-di-config-logging.md) | W8 | 6–8 h |
| 7 | [Error handling: exceptions vs results](07-error-handling.md) | W9 | 4–5 h |
| 8 | [Testing: xUnit v3, test doubles, good tests](08-testing.md) | W10 | 6–8 h |
| 9 | [Labs, algorithms and capstone TicketHub v0.1](09-labs-and-capstone.md) | All weeks | ~40 h total |

---

## Week-by-week plan

| Week | Dates | Mon/Wed/Thu (theory, ~3 h) | Tue (algorithms, 1 h) | Sat (capstone/labs, 3–4 h) |
|---|---|---|---|---|
| W1 | Nov 16 | 01 Type system | Arrays and hashing ×2 | Solution setup, `Directory.Build.props`, CI |
| W2 | Nov 23 | 02 Memory and GC | Arrays and hashing ×2 | Lab: memory-leak hunt; `Money` and `SeatId` value objects |
| W3 | Nov 30 | 03 Async, sections 1–8 | Two pointers ×2 | Lab: deadlock repro |
| W4 | Dec 7 | 03 Async, sections 9–16 | Two pointers ×2 | Lab: thread-pool starvation; first 5 benchmarks |
| W5 | Dec 14 | 04 Collections and LINQ | Sliding window ×2 | `Event` aggregate: hold seats |
| W6 | Dec 21 | 05 Modern C# (holiday-light week) | Stack ×2 | C# 15 playground (unions, closed hierarchies) |
| W7 | Dec 28 | **Buffer / rest / catch-up** | (none) | (none) |
| W8 | Jan 4 | 06 Host, DI, config, logging | Stack ×2 | Benchmarks 6–10 |
| W9 | Jan 11 | 07 Error handling | Binary search ×2 | Result type through the domain; error catalog |
| W10 | Jan 18 | 08 Testing | Binary search ×2 | Full domain test suite and coverage report |
| W11 | Jan 25 | Review all self-checks | Linked list ×3 | TicketHub v0.1 release, benchmark write-up, exit check |

---

## Toolbox for Phase 1

| Tool | Install | Used for |
|---|---|---|
| .NET 10 SDK (LTS) | <https://dotnet.microsoft.com/download> | The capstone target |
| .NET 11 SDK (GA at .NET Conf, Nov 10–13) | Same page | The C# 15 playground |
| IDE | Rider, Visual Studio, or VS Code with C# Dev Kit | Writing and debugging code |
| SharpLab | <https://sharplab.io> (browser) | Seeing lowered C#, IL and assembly |
| ILSpy | `winget install icsharpcode.ILSpy` | Decompiling compiler-generated code |
| BenchmarkDotNet | NuGet package | Micro-benchmarks |
| Diagnostics tools | `dotnet tool install -g dotnet-counters`<br>`dotnet tool install -g dotnet-gcdump`<br>`dotnet tool install -g dotnet-dump`<br>`dotnet tool install -g dotnet-trace` | Live metrics, heap snapshots, dumps, traces |

---

## Phase 1 exit criteria

- [ ] I can explain what happens when `await` reaches an incomplete `Task`.
- [ ] I can explain the 3 DI lifetimes and demonstrate a captive dependency bug.
- [ ] I can explain GC generations and why allocations on the LOH (large object heap) hurt.
- [ ] TicketHub has nullable reference types enabled and zero warnings (warnings are treated as errors).
- [ ] 10 benchmark experiments are documented.
- [ ] ~25–35 algorithm problems are solved in C#.
- [ ] TicketHub v0.1 is tagged, CI is green, and domain logic has more than 80% coverage.
