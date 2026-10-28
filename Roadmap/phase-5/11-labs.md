# 11. Labs

> Phase 5 · All weeks · [← Phase index](README.md)

These labs build the evidence for the Strong Middle checkpoint: **measured** performance work, AI features that are **evaluated and secured**, an upgrade story, and published writing. Use the **lab system** you built in Phases 2–4 (an API, a worker, PostgreSQL, Redis, a broker, Aspire and a pipeline). Keep each lab's `NOTES.md` with what you did, the before/after numbers, and what broke.

> **Before any AI lab:** set a **spending cap** on your provider account, keep API keys in user secrets or a vault (never in Git), and prefer a local model (Ollama) while developing.

| Lab | Guide | Time |
|---|---|---|
| [Lab 1: Profile a slow endpoint](#lab-1-profile-a-slow-endpoint) | [01](01-performance-methodology.md) | 4 h |
| [Lab 2: Allocation diet](#lab-2-allocation-diet) | [02](02-high-performance-code.md) | 5 h |
| [Lab 3: AOT worker](#lab-3-aot-worker) | [03](03-native-aot-and-startup.md) | 3 h |
| [Lab 4: 10× data volume](#lab-4-10-data-volume) | [04](04-database-performance.md) | 6 h |
| [Lab 5: Chat with tools](#lab-5-chat-with-tools) | [05](05-ai-foundations.md) | 5 h |
| [Lab 6: RAG over your docs](#lab-6-rag-over-your-docs) | [06](06-rag.md) | 8 h |
| [Lab 7: MCP server](#lab-7-mcp-server) | [07](07-agents-and-mcp.md) | 6 h |
| [Lab 8: Evaluation suite and red-team](#lab-8-evaluation-suite-and-red-team) | [08](08-ai-quality-security-ops.md) | 6 h |
| [Lab 9: Upgrade to .NET 12](#lab-9-upgrade-to-net-12) | [10](10-upgrade-playbook.md) | 5 h |
| [Lab 10: Share what you learned](#lab-10-share-what-you-learned) | [09](09-senior-behaviors.md) | 10 h+ |

---

## Lab 1: Profile a slow endpoint

1. Pick an endpoint in your lab system, and make it realistically slow if needed: an N+1 query, a sync-over-async call, a large JSON payload, or a regex in a loop.
2. Load-test it (k6 or NBomber) and record p50/p95/p99, throughput and the error rate. That's your **baseline**.
3. Diagnose top-down: the trace (Aspire dashboard or Jaeger) → `dotnet-counters` (GC, thread pool, exceptions) → `dotnet-trace` CPU profile (open it in PerfView or speedscope).
4. Fix **one** cause at a time, and re-measure after each fix.

**Deliverable:** a before/after table per fix, plus a flame-graph screenshot of the main hotspot before and after.

---

## Lab 2: Allocation diet

1. Pick the hottest path from Lab 1. Measure allocated bytes per request with `dotnet-counters` and a `[MemoryDiagnoser]` benchmark of the core method.
2. Reduce allocations by at least 50% using the [guide 2](02-high-performance-code.md) techniques: `Span<T>`, `ArrayPool<T>`, avoiding LINQ and closures in the hot loop, `FrozenDictionary`, `SearchValues`, source-generated JSON and `[LoggerMessage]`.
3. Keep the code readable. For each change, note the allocation saved and the complexity added.
4. Prove the end-to-end effect with the load test: Gen0 collections per second, p99 latency, CPU.

**Deliverable:** BenchmarkDotNet tables (before/after) plus the load-test comparison, and a note on which changes were *not* worth it.

---

## Lab 3: AOT worker

1. Build a small notification worker (or a minimal API) that reads messages, calls one HTTP endpoint and logs the result.
2. Publish it twice: the normal JIT build and `PublishAot=true` (with source-generated JSON and options validation).
3. Fix every trimming and AOT warning without suppressing them blindly.
4. Compare startup time, working-set memory, binary or image size and steady-state throughput.

**Deliverable:** a comparison table and a short recommendation: when would you choose AOT for this kind of service, and when wouldn't you?

---

## Lab 4: 10× data volume

1. Seed the lab database with 10× more data (millions of rows) using a fast bulk method (`COPY`, or a batched generator).
2. Enable `pg_stat_statements` (or Query Store on SQL Server), run your load test, and list the top 5 queries by **total** time.
3. Fix them using `EXPLAIN (ANALYZE, BUFFERS)`: missing or composite indexes, covering indexes, projections, keyset pagination, compiled queries or split queries.
4. Check connection-pool behavior under load (pool size, waits), and try a read replica or caching for one read-heavy endpoint if your setup allows it.

**Deliverable:** for each query, the plan before and after, the timing before and after, and the change made.

---

## Lab 5: Chat with tools

1. Run a local model with Ollama. Build a console chat with history and **streaming** using `IChatClient`.
2. Add a `ChatClientBuilder` pipeline: function invocation, logging, OpenTelemetry and distributed caching.
3. Add 2–3 read-only tools over your lab API (for example, search and details) using `AIFunctionFactory.Create`, and watch the tool calls in the trace.
4. Get **structured output** for one task (for example, extract `{ date, city, maxPrice }` from a free-text query).
5. Switch the provider (Ollama → Azure OpenAI or OpenAI) by changing only the registration code. Write unit tests for your tool logic using a fake `IChatClient`.

**Deliverable:** the repo, a trace screenshot that shows the tool calls, and a note comparing the local and hosted models on quality and latency.

---

## Lab 6: RAG over your docs

1. Choose a document set you know: your team's docs, or a subset of the .NET docs (50–200 pages).
2. Build the **ingestion** pipeline: parse → chunk (try two sizes) → embed → store in PostgreSQL with `pgvector` (an HNSW index), including source metadata.
3. Build the **query** pipeline: embed the question → retrieve the top-k → a grounded prompt → an answer **with citations** and an "I don't know" path.
4. Add **hybrid search** (full-text + vector) and compare it with vector-only on 20 test questions.
5. Add a **permission filter** (for example, per-team documents) and prove that one user can't retrieve another team's chunks.

**Deliverable:** a retrieval quality table (hit rate on 20 questions, per chunk size and per search type), and a demo answering 5 questions with citations.

---

## Lab 7: MCP server

1. Build an MCP server over **stdio** with 2–3 **read-only** tools that wrap your lab API (or the RAG search from Lab 6).
2. Test it with the MCP Inspector, then use it from GitHub Copilot in VS Code via `.vscode/mcp.json`.
3. Add one side-effecting tool with a two-step **propose → confirm** design, and try to make the model skip the confirmation.
4. Move the server to **Streamable HTTP** behind JWT authentication, so tools run with the calling user's permissions.
5. Log every tool call (user, tool, arguments with PII masked, result status).

**Deliverable:** the repo, a Copilot chat screenshot that uses your tools, and notes on what you did to prevent unsafe tool use.

---

## Lab 8: Evaluation suite and red-team

1. Build a **golden dataset** of 40 cases for your Lab 6 or Lab 7 feature: normal questions, edge cases, out-of-scope questions and adversarial inputs, each with the expected facts, sources or tool calls.
2. Run `Microsoft.Extensions.AI.Evaluation` quality evaluators plus one **custom evaluator** (for example, "contains a valid citation"), with response caching and an HTML report.
3. Change one prompt line or the model, re-run, and compare the reports.
4. **Red-team** the feature with the [guide 8](08-ai-quality-security-ops.md) checklist: direct and indirect prompt injection, cross-user data, tool abuse, huge inputs and unsafe output rendering.
5. Fix at least two issues, and add each attack as a regression case in the golden set.
6. Add a cost dashboard (tokens → currency per day) and a per-user daily budget.

**Deliverable:** the two evaluation reports, a red-team findings table (attack → result → fix), and the cost dashboard screenshot.

---

## Lab 9: Upgrade to .NET 12

1. Take a multi-project lab solution (API + worker + tests) and capture the **baseline**: test results, BenchmarkDotNet numbers and a load test.
2. Upgrade it to the newest .NET version available to you (.NET 12 once it ships in November 2027; until then, practice with .NET 11) by following the [guide 10](10-upgrade-playbook.md) checklist: `global.json`, TFM, packages, analyzers, container images and CI.
3. Record every breaking change and warning you hit, and how you fixed it.
4. Compare performance across runtimes with BenchmarkDotNet (`--runtimes net10.0 net12.0`) and the load test.
5. Prototype a **YARP strangler** setup: a small legacy-style app behind a new ASP.NET Core proxy, with one route group migrated.

**Deliverable:** an upgrade report (issues, fixes, performance before/after) that you could publish as a blog post.

---

## Lab 10: Share what you learned

1. Turn your best 2–4 labs into **blog posts** or **internal talks**, using the outline from [guide 9](09-senior-behaviors.md). Good candidates are Lab 2 (allocation diet), Lab 4 (10× data), Lab 8 (red-teaming an AI feature) and Lab 9 (the upgrade report).
2. Every post must include real numbers, code and at least one "what didn't work" section.
3. Ask one experienced colleague to review a draft before you publish.
4. Add each post or talk to your brag document, with the audience size or reactions.

**Deliverable:** published links, or recordings or slides of the talks.
