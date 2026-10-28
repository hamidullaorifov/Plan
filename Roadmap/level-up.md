# .NET Level-Up Roadmap: Junior+ → Middle → Strong Middle

> Prepared: **October 26, 2026** · Horizon: **18 months (Nov 2026 → Apr 2028)**
> Assumed budget: **8–10 h/week** outside work, plus using your day job on purpose.

---

## TL;DR

1. **The .NET platform is stable and well liked.** About 27% of developers use C#, a figure that has stayed flat since 2021. ASP.NET Core has the highest **admiration score (58%)** of the 10 most-used web frameworks. .NET is a mature enterprise stack, and deep skills keep their value for a long time.
2. **There is an opportunity right now.** **.NET 8 and .NET 9 both lose support on Nov 10, 2026.** Every company still on them has to move to **.NET 10 (LTS, supported until Nov 2028)**. If you volunteer to own that migration at work, you get visibility fast.
3. **AI raised the bar for juniors instead of removing it.** More than 70% of developers use AI every day. The most common AI problem is *"plausible but wrong code"* (69%). Employers now pay for people who can **verify, debug, design and own** systems, more than for people who only type code.
4. **Target profile:** a backend .NET engineer who owns a feature end to end: API, database, messaging, deployment and observability. This person uses AI as a power tool and can prove the result is correct.
5. **The plan:** 5 phases, one portfolio project that grows through every phase, measurable exit criteria, a **Middle checkpoint at month 12** and a **Strong Middle checkpoint at month 18**.

---

## 1. Market Analysis (October 2026)

### 1.1 Platform status

| Version | Released | Type | Support ends | What it means for you |
|---|---|---|---|---|
| .NET 8 | Nov 2023 | LTS | **Nov 10, 2026** | Support ends in 2 weeks, so companies everywhere have migration work |
| .NET 9 | Nov 2024 | STS | **Nov 10, 2026** | Same |
| .NET 10 | Nov 2025 | LTS | Nov 14, 2028 | **The production baseline for the next 2 years. Learn this one deeply** |
| .NET 11 | Nov 2026 (RC2 now; GA at .NET Conf, Nov 10–13) | STS (24 months) | ~Nov 2028 | Learn the C# 15 features and use them in your own projects |
| .NET 12 | Expected Nov 2027 | LTS | ~Nov 2030 | The next big enterprise upgrade wave, which lands at your month 12 |

**Where Microsoft is investing (.NET 10/11):**
- **Language (C# 15):** union types, closed hierarchies, extension indexers and collection-expression arguments. These make domain modeling more functional and type-safe ("make illegal states unrepresentable").
- **Runtime:** *Runtime Async*, which moves async support into the runtime for cleaner stack traces and lower overhead. Also ongoing JIT and Native AOT gains.
- **Tooling:** xUnit v3 with Microsoft.Testing.Platform is now the default test template. OpenTelemetry is built into more libraries. Aspire is integrated with `dotnet watch`. File-based apps (`dotnet run app.cs`) are supported, and the SDK can publish container images directly.
- **Security and supply chain:** TLS hardening, post-quantum crypto (ML-KEM) APIs, and NuGet `--min-publish-age-hours`, which protects against freshly published malicious packages.

### 1.2 What the industry uses (Stack Overflow Developer Survey 2026)

| Area | Data | Takeaway for a .NET developer |
|---|---|---|
| Language | C# **27.1%** usage (27–28% every year since 2021), 50% admired | A large, stable enterprise market with slow churn, so deep skills pay off |
| Web | ASP.NET Core **19.9%** usage, **58% admired**. Legacy ASP.NET (.NET Framework) is still **13.9%** | Modern ASP.NET Core is the core skill. Migrating legacy ASP.NET to modern .NET is a long-running source of jobs |
| Databases | **PostgreSQL 57.9%** (#1 and growing every year), Redis 28.3%, SQL Server 27.1% | Know SQL Server (common in enterprise .NET) **and** PostgreSQL. Redis is a must |
| Containers and cloud | Docker **59.1%**, AWS 41.8%, Kubernetes 26.0%, Azure 25.1%, Terraform 18.3% | Docker is required. Azure is the natural fit for .NET, but AWS is bigger overall, so learn cloud *concepts* and go deep on one provider |
| Testing | Playwright 24.9%, xUnit 10.7% (53% admired), Testcontainers 7.8% (55% admired) | Integration tests against real dependencies (Testcontainers) are the modern standard |
| Identity | Auth0 29.7%, Okta 28.7%, Keycloak 18.6% | You need OAuth2/OIDC. Teams rarely write their own authentication now |

### 1.3 AI changed what "junior" means

| Signal | 2025 → 2026 |
|---|---|
| Daily AI use | 47% → **>70%** of respondents |
| Coding assistants and agents | Now the **#1 AI use case (66%)**. Leading coding agents: Claude Code (66%), GitHub Copilot (59%) |
| Most common AI uses | Generating code in a familiar area (69%), debugging (64%), writing tests (58%) |
| AI mistakes that cost the most time | **Plausible but wrong code (69%)**, hard-to-review generated code (54%), missing context (49%), hallucinated package or config (45%), outdated APIs (34%), security/privacy issues (24%) |
| When developers trust AI | 48% trust it only when they can **easily check the answer** |
| Fear | 17% avoid AI because they worry about **losing their skills** |

**Implication:** AI now handles much of the routine junior work: CRUD, boilerplate and simple tests. These skills are still scarce and get people promoted:
1. **Verification.** Read code critically and write tests that catch "almost right" code.
2. **Context.** Know the domain, the system and the constraints that AI can't see.
3. **Debugging in production.** Work with logs, traces, dumps and query plans.
4. **Design judgment.** Choose simple over clever and understand the trade-offs.
5. **Security.** AI-generated code and hallucinated packages create new attack surface, yet only 23% of respondents used security tools.
6. **Building AI into products.** RAG, agents and MCP now have first-class .NET libraries.

### 1.4 An honest read of the market

- **Hiring for entry-level roles has been tight since 2023.** AI puts the "junior who only writes CRUD" profile most at risk. Demand at **Middle level is much healthier**, so moving from Junior+ to Middle is the most valuable step you can take now.
- **.NET demand is concentrated** in enterprise, fintech/banking, insurance, healthcare, government, e-commerce and outsourcing/consulting. These sectors value reliability, security and long-lived systems, so fundamentals beat hype.
- **Region matters a lot.** Salaries and stack choices (Azure vs AWS, SQL Server vs PostgreSQL, Angular vs React) vary by country. Do the market scan in Phase 0 to fit this plan to your local market.

### 1.5 Forecast for 2026–2028

| Trend | Confidence | Action |
|---|---|---|
| .NET 10 LTS stays the enterprise baseline until .NET 12 is adopted (2028) | High | Master .NET 10 and follow what 11/12 add |
| Upgrade and modernization work (.NET 8/9 → 10, .NET Framework → modern .NET) | High | Become the "migration person" |
| AI features inside business apps (RAG, agents, tool calling via MCP) | High | Learn `Microsoft.Extensions.AI` and one vector store |
| AI-assisted development becomes the default, so review and verification skills gain value | High | Build a disciplined AI workflow (Section 7) |
| PostgreSQL keeps gaining ground in .NET shops | Medium-High | Learn PostgreSQL alongside SQL Server |
| Containers, OpenTelemetry and Aspire become the cloud-native default (Aspire for local development) | Medium-High | Use Aspire in the capstone project |
| Domain modeling gets more functional through C# unions and closed hierarchies | Medium | Learn Result types and discriminated unions now |
| The microservices hype cools and modular monoliths become the preferred starting point | Medium | Learn the modular monolith first and microservices second |
| Blazor/MAUI growth | Low-Medium (Blazor ~7% usage) | Optional. Only learn them if your target employers use them |

---

## 2. Target Profile: What Each Level Looks Like

| Competency | Junior+ (now, typical) | **Middle (month 12)** | **Strong Middle (month 18)** |
|---|---|---|---|
| Scope | Tasks with clear specs | Features end to end with little guidance | Ambiguous features and small epics; splits work for others |
| C#/.NET | Uses language features | Knows *how* they work (async, GC, LINQ, DI) | Diagnoses runtime issues (memory, threads, performance) |
| Data | Writes EF queries | Designs schema and indexes, reads query plans, handles transactions | Tunes queries and chooses the right data store |
| Architecture | Follows existing patterns | Explains *why* the patterns exist and makes local design decisions | Proposes designs with trade-offs and writes ADRs |
| Testing | Writes unit tests when asked | Owns the test strategy for their feature (unit + integration) | Shapes the team's testing approach |
| Production | Rarely touches it | Reads logs and traces; fixes incidents with help | Leads incident investigations and adds observability proactively |
| DevOps | Uses the existing pipeline | Changes CI/CD, Dockerfiles and configs | Designs pipelines and the environment strategy |
| AI | Uses chat for answers | Uses agents productively and verifies their output | Sets team AI conventions (instruction files, review rules) |
| Collaboration | Asks for help | Reviews peers' code meaningfully and estimates reliably | Mentors juniors and drives technical discussions |

---

## 3. The Roadmap

### Phase 0: Baseline (Weeks 1–2, Nov 1–14, 2026)

- [ ] **Self-assessment:** score yourself 0–3 on every row in Section 2 and write down your **top 3 gaps**.
- [ ] **Market scan:** collect **30 Middle .NET job postings** in your target market (your country or remote). Tally the required skills in a spreadsheet, then adjust the priorities in Phases 2–4 (Azure vs AWS, SQL Server vs PostgreSQL, Angular vs React).
- [ ] **Talk to your lead:** ask *"What would I need to show to be promoted to Middle in 12 months?"* Write the answer down and agree on 1–2 stretch responsibilities.
- [ ] **Set up your tools:** .NET 10 SDK (and .NET 11 after Nov 10), Docker or Podman, Rider / Visual Studio / VS Code with C# Dev Kit, Git, a GitHub account and an AI assistant.
- [ ] **Create the capstone repo** (Section 4) with a README.
- [ ] **Start a "brag document":** a running log of what you did, its impact and the skills you used. You will use it for performance reviews and your CV.

---

### Phase 1: C# and Runtime Depth, Plus Testing (Months 1–3, Nov 2026 – Jan 2027)

> **Detailed study guides:** [phase-1/README.md](phase-1/README.md) contains 9 topic guides with explanations, code, interview Q&A, exercises and labs.

**Goal:** Move from *using* .NET to *understanding* it, and be able to pass any C# fundamentals interview round.

**Must learn**
- **Type system:** value vs reference types, struct pitfalls, boxing, `record` / `record struct`, `readonly struct`, `ref struct`. Enable **nullable reference types** everywhere and treat their warnings as errors.
- **Memory:** stack vs heap, GC generations, LOH/POH, `IDisposable` / `using` / finalizers, `Span<T>` / `Memory<T>`, `ArrayPool<T>`, and why allocations matter.
- **Async and concurrency:** how the async state machine works, `Task` vs `ValueTask`, `ConfigureAwait`, sync-over-async deadlocks, `CancellationToken` everywhere, `IAsyncEnumerable`, `Channel<T>`, `SemaphoreSlim`, `Parallel.ForEachAsync`, `System.Threading.Lock`, `Interlocked`. Read what **Runtime Async** in .NET 11 changes.
- **Collections and LINQ:** the complexity of `List` / `Dictionary` / `HashSet` / `FrozenDictionary` / immutable collections, deferred execution, `IEnumerable` vs `IQueryable`, and the multiple-enumeration bug.
- **Modern C# (12–15):** primary constructors, collection expressions, the `field` keyword, extension members, pattern matching, required members, and **union types and closed hierarchies (C# 15)**.
- **Generic host basics:** DI lifetimes (singleton/scoped/transient and the captive dependency bug), the Options pattern (`IOptions` / `IOptionsSnapshot` / `IOptionsMonitor` with validation), configuration providers, structured logging with `ILogger`, `BackgroundService`.
- **Error handling:** exceptions vs the Result pattern, and when to use each.

**Testing**
- xUnit v3, NSubstitute or Moq, the AAA pattern, and the kinds of test doubles (stub, mock, fake).
- Test *observable behavior* rather than implementation details (Khorikov).
- Assertion libraries: Shouldly or AwesomeAssertions. *FluentAssertions v8+ uses a commercial license.*
- Treat coverage as a signal, not a goal. Optionally try mutation testing with Stryker.NET.

**Practice**
- Write **10 BenchmarkDotNet experiments** in a `/lab` folder: boxing cost, `List` vs `Span`, `string.Concat` vs `StringBuilder`, LINQ vs loops, `async` overhead, `FrozenDictionary` lookups, and so on. Record the results in markdown.
- Reproduce a sync-over-async deadlock in a console app, then fix it.
- Solve ~25 easy/medium algorithm problems in C#.

**Capstone: TicketHub v0.1**
- A domain library (Events, Venues, Seats, Orders) with a rich domain model, enforced invariants and Result/union types. Domain logic has more than 80% unit-test coverage.
- GitHub Actions builds and runs the tests on every push.

**Exit criteria**
- [ ] I can explain what happens when `await` reaches an incomplete `Task`.
- [ ] I can explain the 3 DI lifetimes and demonstrate a captive dependency bug.
- [ ] I can explain GC generations and why allocations on the large object heap (LOH) hurt.
- [ ] The capstone has nullable reference types enabled and zero warnings.
- [ ] 10 benchmark experiments are documented.

---

### Phase 2: ASP.NET Core 10 and Data (Months 3–6, Feb – Apr 2027)

> **Detailed study guides:** [phase-2/README.md](phase-2/README.md) contains 9 topic guides plus 10 labs, with explanations, code, interview Q&A and exercises.

**Goal:** Build a production-grade API that you can defend line by line.

**ASP.NET Core**
- The request pipeline and middleware order, Kestrel, routing. Know both **Minimal APIs and Controllers**, because many enterprise codebases use controllers.
- Model binding and validation (built-in Minimal API validation in .NET 10, or FluentValidation), `ProblemDetails`, and global error handling with `IExceptionHandler`.
- OpenAPI with the built-in `Microsoft.AspNetCore.OpenApi` (Swashbuckle is no longer in the templates), plus Scalar or Swagger UI.
- **Authentication and authorization:** cookies vs JWT bearer, OAuth2/OIDC flows (authorization code + PKCE, client credentials), policies and requirements, resource-based authorization. Integrate with **Keycloak** (free, runs locally) or Entra ID / Auth0.
- **API design:** resource modeling, status codes, pagination and filtering, idempotency keys, versioning (`Asp.Versioning`), ETags and optimistic concurrency.
- **Cross-cutting concerns:** rate limiting, output caching, `HybridCache`, CORS, health checks, response compression.
- Basics of SignalR, Server-Sent Events (SSE) and gRPC.

**Data**
- **SQL fundamentals** on both SQL Server and PostgreSQL: joins, window functions, CTEs, indexes (clustered, non-clustered, covering, column order in composite indexes), **reading execution plans**, isolation levels and the anomalies they prevent, locking and deadlocks.
- **EF Core 10:** `DbContext` lifetime, the change tracker, `AsNoTracking`, loading strategies (and why lazy loading is dangerous), **N+1 detection**, split queries, projecting to DTOs, compiled queries, `ExecuteUpdate`/`ExecuteDelete`, migrations in a team and in CI/CD, optimistic concurrency (`rowversion`/`xmin`), transactions, interceptors, complex types and JSON columns, global query filters (soft delete, multi-tenancy).
- **Dapper** for hot paths and reporting queries.
- **Redis:** cache-aside, TTLs, invalidation, and distributed locks with their pitfalls.

**Testing**
- Integration tests with `WebApplicationFactory`, **Testcontainers** (PostgreSQL, Redis) and Respawn to reset the database between tests.

**Practice**
- Find a slow endpoint at work or in the capstone. Log the SQL it runs, fix the N+1 query or missing index, and record the before/after numbers.
- Read one ASP.NET Core source file per week on GitHub, for example how authentication and routing middleware work.

**Capstone: TicketHub v0.2**
- A REST API to browse events, **reserve seats** (with optimistic concurrency so two users can't book the same seat), check out, and view "my orders".
- PostgreSQL with EF Core, Redis caching for the event catalog, Keycloak with roles (customer/organizer/admin), OpenAPI, ProblemDetails, rate limiting and health checks.
- Integration tests run in CI using Testcontainers.

**Exit criteria**
- [ ] I can explain where authentication, CORS and exception handling go in the pipeline, and why order matters.
- [ ] I can read an execution plan and explain why a query does a scan.
- [ ] I can explain OAuth2 authorization code + PKCE step by step.
- [ ] A concurrent integration test proves that seat reservation is race-safe.
- [ ] I shipped at least one non-trivial feature at work end to end: design, PR, deploy and verification in production.

---

### Phase 3: Architecture and Distributed Systems (Months 6–9, May – Jul 2027)

> **Detailed study guides:** [phase-3/README.md](phase-3/README.md) contains 9 topic guides plus 11 labs, with explanations, code, interview Q&A and exercises.

**Goal:** Understand *why* codebases are structured the way they are, and make and defend design decisions.

**Design and architecture**
- SOLID, coupling and cohesion, composition over inheritance, all applied to real code rather than memorized definitions.
- Layered / Clean / Onion vs **Vertical Slice** architecture: the trade-offs, and when Clean Architecture is overkill.
- **Modular monolith:** module boundaries, internal APIs, a schema per module, and boundaries enforced by architecture tests (NetArchTest / ArchUnitNET).
- **DDD:**
  - Tactical: entities, value objects, aggregates and their consistency boundaries, domain events.
  - Strategic (introduction): bounded contexts, ubiquitous language, context maps.
- **CQRS** in its light form: separate read and write models, not necessarily separate databases.
- *Note: MediatR, AutoMapper and MassTransit v9+ moved to commercial licenses in 2025.* Know them because they are in many codebases, but check the license before adopting them, and be able to write a simple dispatcher or mapper yourself (or use alternatives such as Mapperly or Wolverine).
- Patterns you will see in .NET code: Strategy, Decorator (Scrutor), Factory, Adapter, Pipeline / Chain of Responsibility.
- **ADRs (Architecture Decision Records).**

**Distributed systems**
- **Messaging:** queues vs topics, at-least-once delivery, **idempotent consumers**, ordering, poison messages and dead-letter queues (DLQ), retries with backoff.
- **Brokers:** RabbitMQ (locally), Azure Service Bus or AWS SQS/SNS, and Kafka concepts (partitions, consumer groups, offsets).
- **Patterns:** Transactional **Outbox/Inbox**, **Saga** (orchestration vs choreography), compensation, eventual consistency.
- **Resilience:** timeouts, retries, circuit breakers and bulkheads with `Microsoft.Extensions.Http.Resilience` (Polly v8).
- **Background jobs:** `BackgroundService`, Hangfire or Quartz.NET.
- **Theory:** CAP/PACELC, consistency models, and the pitfalls of distributed locks and clocks. Read the DDIA chapters on replication, transactions and distributed-systems failures.

**Capstone: TicketHub v0.3**
- Refactor into a **modular monolith** with Catalog, Booking, Payments and Notifications modules.
- Payments is a separate process: a simulated payment provider with webhooks, and **idempotent** webhook handling.
- Domain events go through the **Outbox** to RabbitMQ, and the Notifications module sends emails (Mailpit locally).
- Seat holds expire through a background job. A **Saga** covers "reserve → pay → confirm", releasing the seats on timeout or failure.
- Architecture tests enforce the module boundaries. At least 3 ADRs are written.

**Exit criteria**
- [ ] I can explain why the Outbox pattern exists and what breaks without it.
- [ ] I can design an idempotent consumer and explain at-least-once delivery.
- [ ] I can argue for a modular monolith vs microservices in a given context.
- [ ] I wrote a design doc or ADR at work (even for a small feature) and got it reviewed.
- [ ] I review peers' PRs regularly with substantive comments, not just "LGTM".

---

### Phase 4: Cloud, DevOps, Observability and Security (Months 9–12, Aug – Oct 2027) → **Middle checkpoint**

> **Detailed study guides:** [phase-4/README.md](phase-4/README.md) contains 9 topic guides plus 12 labs, with explanations, code, interview Q&A and exercises.

**Goal:** Own a feature all the way to production, and be able to debug it there.

**Containers and orchestration**
- Dockerfiles: multi-stage builds, non-root users, small and chiseled images. SDK container publishing with `dotnet publish /t:PublishContainer`. Docker Compose.
- **Aspire** (formerly .NET Aspire): the AppHost, service defaults, the dashboard and integrations. It runs the whole capstone locally with one command.
- **Kubernetes fundamentals:** pods, deployments, services, ingress, ConfigMaps and Secrets, probes, resource limits, HPA, and Helm basics. Learn enough to deploy and debug, not to administer a cluster.

**CI/CD**
- GitHub Actions or Azure DevOps: build, test, coverage, static analysis, container build and push, environment promotion and approvals.
- Trunk-based development vs GitFlow, semantic versioning, feature flags (`Microsoft.FeatureManagement`), and the ideas behind blue/green and canary releases.
- Database migrations in pipelines (EF migration bundles or idempotent scripts).

**Cloud (go deep on one provider, based on your market scan)**
- **Azure:** App Service, Container Apps, Functions, Azure SQL / PostgreSQL Flexible Server, Service Bus, Storage, Key Vault, Managed Identity, App Configuration, Application Insights, Entra ID.
- **AWS:** ECS/Fargate, Lambda, RDS, SQS/SNS, S3, Secrets Manager, IAM roles, CloudWatch.
- Infrastructure as code basics: **Bicep** for Azure, or **Terraform/OpenTofu**.
- Cost awareness: know what your resources cost.

**Observability**
- **OpenTelemetry:** traces, metrics and logs, with correlation across HTTP calls *and* messaging.
- Structured logging (Serilog or `ILogger`), disciplined log levels, and no PII in logs.
- Backends: the Aspire dashboard locally, then Grafana + Prometheus + Tempo/Loki, Application Insights or Datadog.
- **Diagnostics:** `dotnet-counters`, `dotnet-trace`, `dotnet-dump`, `dotnet-gcdump`. Analyze at least one memory dump.
- SLIs and SLOs, and alerting on symptoms rather than causes.

**Security**
- The **OWASP Top 10** mapped to ASP.NET Core: broken access control (IDOR), injection, SSRF, misconfiguration, vulnerable components.
- **Secrets management:** never keep secrets in `appsettings.json`. Use Key Vault or Secrets Manager with managed identities or IAM roles.
- **Supply chain:** NuGet audit, `dotnet list package --vulnerable`, Dependabot or Renovate. **Check that packages suggested by AI actually exist and are maintained**, because 45% of developers lost time to hallucinated packages or config.
- The Data Protection API, HTTPS/HSTS, security headers, CSRF protection, input validation and output encoding.

**Capstone: TicketHub v1.0**
- The Aspire AppHost starts everything locally with one command.
- **GitHub Actions** pipeline: build → unit and integration tests → security scan → container push → **deploy to the cloud** (Azure Container Apps or AWS ECS, on a low-cost tier) on every merge to `main`.
- **One OpenTelemetry trace** shows the full path: HTTP → database → RabbitMQ → Notifications.
- A **k6 load test** documents throughput and p95 latency. Find and fix one bottleneck.
- Secrets live in Key Vault or Secrets Manager. A public demo URL and a polished README with an architecture diagram.

**✅ Middle checkpoint (Month 12, ~end of Oct 2027)**
- [ ] I shipped at least 3 features end to end at work with minimal guidance.
- [ ] I investigated and resolved at least 1 production issue using logs, traces or metrics.
- [ ] TicketHub v1.0 is public, deployed and documented.
- [ ] Optional: one cloud certification (Azure developer associate, checking the current exam code on Microsoft Learn, or AWS Certified Developer – Associate).
- [ ] I passed a mock interview (Section 8) run by a Middle or Senior engineer.
- [ ] I had the promotion conversation with my lead, or started applying for Middle roles.

---

### Phase 5: Performance, AI-Enabled Apps and Senior Behaviors (Months 12–18, Nov 2027 – Apr 2028)

> **Detailed study guides:** [phase-5/README.md](phase-5/README.md) contains 10 topic guides plus 10 labs, with explanations, code, interview Q&A and exercises.

**Goal:** Become the person the team goes to for hard problems, and stand out by building AI into applications.

**Performance**
- Measure first: BenchmarkDotNet, profilers (dotTrace / dotMemory, the Visual Studio profiler, PerfView), `dotnet-counters`.
- Reduce allocations: pooling, `Span`, `System.IO.Pipelines`, and **source generators** (System.Text.Json, logging, regex).
- **Native AOT and trimming:** learn when they pay off (serverless, CLIs, cold-start-sensitive services).
- Database performance at scale: indexing strategy, Query Store, connection pooling, read replicas, cache layers.
- Read Stephen Toub's yearly *"Performance Improvements in .NET X"* post.

**AI-enabled applications in .NET**
- `Microsoft.Extensions.AI` (`IChatClient`, `IEmbeddingGenerator`): structured output, tool/function calling, streaming.
- **RAG:** chunking, embeddings, vector search (**pgvector** on PostgreSQL, Azure AI Search or Qdrant), hybrid search, re-ranking and citations.
- **Agents:** Microsoft Agent Framework (the successor to Semantic Kernel and AutoGen), multi-step workflows and human-in-the-loop steps.
- **MCP:** build an MCP server in C# with the official ModelContextProtocol SDK to expose your app's capabilities to AI agents.
- **Evaluation and operations:** `Microsoft.Extensions.AI.Evaluation`, tracking cost and latency, OpenTelemetry for LLM calls, and local models via Ollama for development.
- **AI security:** prompt injection, data exfiltration through tools, least privilege for tool calls, PII handling and output validation.

**Senior behaviors (practice them deliberately)**
- Write design docs for medium-sized features and run a design review.
- Break an epic into tasks for other people and estimate with ranges.
- Mentor a junior or intern, and give a knowledge-sharing talk (for example, "What we learned migrating to .NET 10").
- Join the on-call rotation and write a blameless post-mortem.
- Drive one technical improvement with measurable impact: CI speed, flaky tests, observability or dependency hygiene.

**Capstone: TicketHub v2.0**
- **Upgrade to .NET 12 LTS** (Nov 2027) and write a blog post about the migration.
- **Natural-language event search**, for example "jazz concerts next weekend under $50", built with RAG on pgvector and `Microsoft.Extensions.AI`.
- An **MCP server** that exposes "search events" and "hold seats" tools, with proper authorization and a human confirmation step.
- **A performance pass** that improves p95 latency, with a documented before/after report.
- **Native AOT** for one small service (the Notifications worker), with startup time and memory measured.

**✅ Strong Middle checkpoint (Month 18, ~end of Apr 2028)**
- [ ] I led a feature or epic that involved at least 2 other developers.
- [ ] At least 2 of my design docs or ADRs were accepted by the team.
- [ ] I am the go-to person on the team for one area (performance, data, CI/CD, AI integration or migrations).
- [ ] I published 2–4 technical blog posts or gave internal talks.
- [ ] TicketHub v2.0 is done, with AI features and a performance report.

---

### Continuous tracks (all 18 months)

| Track | Weekly dose | Goal |
|---|---|---|
| Algorithms and data structures | 2–3 problems in C# | ~130–150 problems by month 12 (the NeetCode 150 list is a good spine) so you can pass live-coding rounds |
| System design | 1 case every 2 weeks, starting at month 6 | Design a URL shortener, rate limiter, notification system, ticket booking system and chat at Middle depth |
| Reading code | 30 min | The ASP.NET Core, EF Core and runtime repos, and the `dotnet/eShop` reference app |
| Writing and English | As you go | PR descriptions, design docs and blog posts. Writing is a promotion skill |
| Git | As you go | Interactive rebase, `bisect`, `cherry-pick`, `reflog`, and resolving conflicts with confidence |
| AI workflow | Daily | See Section 7 |

---

## 4. Capstone Project: TicketHub

The capstone is a ticket-booking platform. The domain is easy to explain, and it teaches **concurrency** (seat races), **consistency** (reserve and pay), **idempotency** (payment webhooks), **caching**, **messaging** and **AI search**. Each of these gives you a strong interview story.

| Version | Phase | Highlights |
|---|---|---|
| v0.1 | 1 | Domain model, union/Result types, unit tests, CI |
| v0.2 | 2 | REST API, EF Core + PostgreSQL, Redis, Keycloak, integration tests |
| v0.3 | 3 | Modular monolith, Outbox, RabbitMQ, Saga, architecture tests, ADRs |
| v1.0 | 4 | Aspire, Docker, CI/CD to the cloud, OpenTelemetry, k6 load test |
| v2.0 | 5 | .NET 12, RAG search, MCP server, performance report, Native AOT |

**Repo quality bar:** a clear README (what it is, how to run it, an architecture diagram), meaningful commits, PRs with descriptions (even when you work alone), green CI, `docs/adr/` and a `CHANGELOG.md`.

---

## 5. Use Your Day Job (the Biggest Multiplier)

You have about 10 learning hours a week outside work and about 40 at work. Make work your main learning engine:

1. **Own the .NET 10 migration now.** .NET 8 and 9 lose support on **Nov 10, 2026**. Propose a plan:
   - List every project and its target framework (TFM).
   - Bump `global.json` and the TFMs.
   - Update packages and review the official breaking-change notes.
   - Update CI agents and container base images.
   - Run a regression test, then roll out in stages.

   Use AI-assisted upgrade tooling (such as GitHub Copilot app modernization) if your company allows it. Write up what you did, because it makes a perfect interview story.
2. **Volunteer for the work others avoid:** flaky tests, slow queries, gaps in logging, vulnerable dependencies, slow CI.
3. **Join incident investigations.** Shadow first, then lead.
4. **Review seniors' PRs** and ask "why" questions in the comments. You learn how they reason.
5. **Before each task**, write a 5-line plan and ask a senior to review it. It's cheap mentorship.
6. **Keep the brag document up to date:** date, what you did, impact (with numbers) and skills used.
7. **Have a quarterly 1:1** with your lead to review your progress against the Middle criteria.

---

## 6. Weekly Schedule Template (8–10 hours)

| Day | Time | Activity |
|---|---|---|
| Mon | 1 h | Theory: a chapter from the current phase's book or course, with notes |
| Tue | 1 h | Algorithms (2 problems) |
| Wed | 1 h | Apply that week's theory in `/lab` experiments |
| Thu | 1 h | Read code, the .NET blog or deep-dive articles |
| Sat | 3–4 h | Capstone increment: a feature with tests and a PR with a description |
| Sun | 30 min | Weekly review: update the tracker and brag doc, plan next week |

**Rules:** finish every month with something shipped (a capstone release tag and changelog entry). Take one light week roughly every 8 weeks to avoid burnout.

---

## 7. Using AI Without Stunting Your Growth

Overusing AI and avoiding it are both career risks. Follow these rules:

1. **Separate learning mode from shipping mode.** When you learn a new concept, write the first version yourself and then ask AI to critique it. For familiar work, let AI draft and you review.
2. **Never merge code you can't explain.** If you can't defend every line in a code review, you don't own it.
3. **Treat tests as your verification tool.** Let AI generate test scaffolding, but write the key assertions yourself.
4. **Check packages and APIs.** Confirm that suggested NuGet packages exist, are maintained and have an acceptable license. Check APIs against the current docs, because AI often suggests **outdated APIs (34%)**.
5. **Give AI context.** Learn *context engineering*: repo instruction files (`.github/copilot-instructions.md`, `AGENTS.md`), short specs and pointers to the relevant files. **Missing context causes 49% of AI failures.**
6. **Use agents freely** for scaffolding, test generation, refactoring, explaining unfamiliar code and summarizing logs. **Be extra careful** with security, concurrency, money and data migrations.
7. **Take a monthly "no-AI hour":** build a small feature or solve a problem with AI turned off to check your fundamentals.

---

## 8. Interview Prep for Middle .NET Roles

### Questions you must answer confidently

**C# and runtime**
- Value vs reference types; where structs actually live; boxing.
- `class` vs `struct` vs `record` vs `record struct`; how record equality works.
- GC generations, the LOH, `IDisposable` vs finalizers, the dispose pattern.
- How `async`/`await` works internally; `Task` vs `ValueTask`; `ConfigureAwait(false)`; why `async void` is dangerous; deadlocks.
- Thread safety: `lock`, `Interlocked`, `SemaphoreSlim`, concurrent collections.
- `IEnumerable` vs `IQueryable`; deferred execution; `yield`; multiple enumeration.
- Delegates, events and closures (the captured loop variable bug).
- Generics: constraints, covariance and contravariance.
- `throw` vs `throw ex`; exception filters; what exceptions cost.

**ASP.NET Core**
- The middleware pipeline and its order; filters vs middleware.
- DI lifetimes and captive dependencies; using scoped services from a singleton (`IServiceScopeFactory`).
- Authentication vs authorization; JWT structure and validation; refresh tokens; OAuth2 flows.
- The Options pattern variants; configuration precedence.
- Global exception handling and `ProblemDetails`; health checks.

**Data**
- Tracking vs no-tracking; N+1; loading strategies; projections.
- Index types; covering indexes; why an index isn't used; reading a plan.
- Isolation levels and the anomalies they prevent; deadlocks; optimistic vs pessimistic concurrency.
- Why distributed transactions are avoided, and what to use instead (Outbox, Saga).

**Architecture and distributed systems**
- SOLID, illustrated with examples from *your* code.
- Clean vs Vertical Slice architecture; modular monolith vs microservices.
- Idempotency, at-least-once delivery, the Outbox pattern, retries and circuit breakers.
- Caching strategies and invalidation; REST semantics (PUT vs PATCH, idempotent methods, status codes).

### Formats to expect
- **Live coding:** an easy or medium algorithm, or "build an endpoint with validation and tests".
- **Code review exercise:** find the bugs, performance problems and security issues in a snippet. This format is increasingly common now that AI writes so much code.
- **Debugging task:** find why some code is failing or slow.
- **Take-home:** a small API. Show tests, a README and clean commits.
- **System design (Middle depth):** a booking or notification system. Cover data model, database choice, caching, queues, idempotency, failure modes and scaling reads.

### Behavioral questions (STAR format): prepare 6 stories
1. The hardest bug you've fixed.
2. A production incident you handled.
3. A disagreement with a teammate and how you resolved it.
4. Something you improved without being asked (for example, the .NET 10 migration).
5. A mistake you made and what changed afterward.
6. A feature you owned end to end.

---

## 9. What Not to Spend Time on (for Now)

- ❌ Chasing every new library, or watching tutorials without building anything.
- ❌ Microservices for a one-person project before you understand a modular monolith.
- ❌ Deep Blazor/MAUI/WPF, *unless* your target jobs require them (Blazor has about 7% usage).
- ❌ Legacy technology (WebForms, WCF server, Remoting), unless you deliberately target modernization roles. In that case, learn just enough to migrate *away* from it.
- ❌ Collecting certifications without projects. At Middle level they carry little weight on their own.
- ❌ Kubernetes administration internals. A backend developer only needs the fundamentals.
- ❌ Memorizing pattern definitions. Learn patterns by refactoring real code.

---

## 10. Specialization After Middle (Choose Around Month 12)

| Track | Choose it if you… | Extra skills | Market signal |
|---|---|---|---|
| **Cloud-native backend / platform** | Like infrastructure and reliability | Kubernetes, Terraform, SRE practices, cost optimization | Docker 59%, Kubernetes 26% usage |
| **AI-enabled application engineer** | Like building products with AI | RAG, agents, MCP, evaluations, AI security | The fastest-moving area, and few .NET developers do it well |
| **High-load / performance** | Like low-level work | Profiling, memory, concurrency, database internals | Fintech, trading, gaming backends |
| **Full-stack .NET + TypeScript** | Like UI work | React or Angular + TypeScript (43.8% usage) | Many .NET job postings also require a front-end framework |
| **Modernization specialist** | Like legacy puzzles | .NET Framework → modern .NET, WCF → gRPC/REST, splitting monoliths | Legacy ASP.NET still has 13.9% usage; steady consulting demand |
| **Security-focused backend** | Like breaking things | AppSec, threat modeling, OIDC, supply chain | Only 23% use security tools, and AI-related security issues cost time for 24% of developers |

---

## 11. Resources (Curated)

**Books, in phase order**
- **Phase 1:** *C# in Depth* (Jon Skeet), *Concurrency in C# Cookbook* (Stephen Cleary), *Pro .NET Memory Management* (Konrad Kokosa, selected chapters), *Unit Testing Principles, Practices, and Patterns* (Vladimir Khorikov).
- **Phase 2:** *ASP.NET Core in Action* (Andrew Lock), *Entity Framework Core in Action* (Jon P Smith), *Dependency Injection Principles, Practices, and Patterns* (van Deursen & Seemann).
- **Phase 3:** *Learning Domain-Driven Design* (Vlad Khononov), *Designing Data-Intensive Applications* (Martin Kleppmann), *Fundamentals of Software Architecture* (Richards & Ford).
- **Phases 4–5:** *Release It!* (Michael Nygard), *System Design Interview, Vol. 1* (Alex Xu), *The Pragmatic Programmer*.

**Official sources**
- Microsoft Learn: .NET documentation.
- The .NET Blog: release posts and Stephen Toub's performance posts.
- **.NET Conf 2026 (Nov 10–13):** watch the .NET 11 launch sessions.
- The `dotnet/eShop` reference application and the ASP.NET Core / EF Core / runtime source on GitHub.

**Community (pick 2–3; don't try to follow everything)**
- Andrew Lock: ASP.NET Core internals.
- Milan Jovanović: architecture, EF Core, modular monoliths.
- Nick Chapsas: modern C# and performance.
- Steve "Ardalis" Smith: Clean Architecture and DDD.
- Stephen Cleary: async.
- Jimmy Bogard: messaging and distributed systems.

---

## 12. Progress Tracker

| Checkpoint | Target date | Key evidence | Status |
|---|---|---|---|
| Phase 0: Baseline | 2026-11-14 | Self-assessment, 30-posting market scan, written expectations from your lead | ☐ |
| Phase 1: C# and runtime | 2027-01-31 | TicketHub v0.1, 10 benchmarks, ~35 algorithm problems | ☐ |
| Phase 2: ASP.NET Core and data | 2027-04-30 | TicketHub v0.2, 1 feature shipped end to end at work | ☐ |
| Phase 3: Architecture | 2027-07-31 | TicketHub v0.3, ADRs, a design doc at work | ☐ |
| **Middle checkpoint** | **2027-10-31** | TicketHub v1.0 deployed, a production incident handled, a mock interview passed | ☐ |
| **Strong Middle checkpoint** | **2028-04-30** | TicketHub v2.0 (AI and performance), a feature led, talks or posts published | ☐ |

---

## Sources

- .NET support policy (updated Oct 25, 2026): https://dotnet.microsoft.com/en-us/platform/support/policy/dotnet-core
- What's new in .NET 11 (RC2): https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-11/overview
- Stack Overflow Developer Survey 2026, Technology: https://survey.stackoverflow.co/2026/technology/data
- Stack Overflow Developer Survey 2026, AI: https://survey.stackoverflow.co/2026/ai
- Stack Overflow Developer Survey 2025, AI (used for the trend comparison): https://survey.stackoverflow.co/2025/ai

> **Caveat:** The survey data is global and skews toward Stack Overflow users. Job-posting volumes, salaries and the Azure vs AWS split vary by region, so calibrate this plan with the Phase 0 market scan.
