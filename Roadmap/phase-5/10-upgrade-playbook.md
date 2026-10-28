# 10. Upgrade Playbook: .NET 12 and Legacy Modernization

> Phase 5 · Weeks 19–20 · [← Phase index](README.md)

**Why this matters:** .NET 12 (the next LTS) is expected in November 2027, and .NET 10 support ends in November 2028, so every .NET shop will migrate during 2028. On top of that, a lot of revenue still runs on .NET Framework. The developer who can **plan and lead upgrades safely** becomes indispensable. It's also a strong blog post and a strong interview story.

---

## 10.1 Know the support calendar

| Version | Type | End of support |
|---|---|---|
| .NET 10 | LTS | **Nov 2028** |
| .NET 11 | STS (24 months) | ~Nov 2028 |
| .NET 12 | LTS (expected Nov 2027) | ~Nov 2030 |
| .NET Framework 4.8.x | part of Windows | supported with Windows, but **no new features** |

Upgrade **within the first 6 months** after an LTS ships. That leaves time for surprises before the old version expires. Track the official support policy page and the monthly Patch Tuesday updates.

---

## 10.2 Upgrading .NET 10 → .NET 12: step by step

### 1. Prepare

- [ ] Read **"What's new in .NET 12"** and the **breaking changes** list (runtime, ASP.NET Core, EF Core, SDK, containers).
- [ ] Make an inventory: projects, target frameworks, NuGet packages (and whether each supports .NET 12), CI images, container base images, hosting platforms.
- [ ] Capture a **baseline**: test pass rate, BenchmarkDotNet results for hot paths, and a load test (p95, allocations, memory).

### 2. Upgrade the build

```json
// global.json
{ "sdk": { "version": "12.0.100", "rollForward": "latestFeature" } }
```

```xml
<!-- Directory.Build.props -->
<TargetFramework>net12.0</TargetFramework>
```

```powershell
dotnet list package --outdated                         # (or: dotnet package list --outdated)
# update packages, Microsoft.* packages first (ASP.NET Core, EF Core, Extensions), then the rest
dotnet build -warnaserror                               # new analyzers and obsoletions show up here
```

- **The new C# version** becomes the default. Watch for new keywords colliding with identifiers (as `field` did in C# 14), and new warnings.
- **Libraries** consumed by older apps can multi-target during the transition: `<TargetFrameworks>net10.0;net12.0</TargetFrameworks>`.

### 3. Fix and verify

- Fix the breaking changes and obsoletion warnings; don't suppress them blindly.
- Run the **full test suite**, including integration tests with Testcontainers.
- Re-run the **benchmarks and load tests** against the baseline, and compare runtimes with BenchmarkDotNet's `--runtimes`. Most upgrades bring free performance, but verify it, because regressions happen.
- Update the **CI images**, the **container base images** (`aspnet:12.0-…`) and any SDK-dependent tooling (dotnet-ef, global tools).

### 4. Roll out

- Deploy to staging and **soak** it (hours to days) while watching GC, memory, CPU and error rates.
- Roll out to production gradually (a canary or a percentage), with a tested rollback (the previous image).
- Upgrade **one service at a time** in a multi-service system. Start with low-risk services.

### 5. Share

- Write the upgrade notes: what broke, what you fixed, the performance numbers. That's an internal wiki page plus a blog post.

---

## 10.3 Legacy modernization: .NET Framework → modern .NET

### Assess first

| Question | Why it matters |
|---|---|
| Which app types are involved? (ASP.NET MVC/Web API, WebForms, WCF, Windows services, WinForms/WPF) | each has a different migration path |
| Which dependencies are Framework-only? (`System.Web`, old NuGet packages, COM, native DLLs) | blockers |
| How good is the test coverage? | without tests, you add characterization tests first |
| How big, and how critical? | big-bang vs incremental |

Tools such as **GitHub Copilot app modernization for .NET** and Microsoft's upgrade guidance can analyze projects and automate parts of the upgrade. Check which tools are current, and treat their output like any AI-generated change: review it and test it.

### Migration paths by app type

| App type | Path |
|---|---|
| **Class libraries** | retarget to `netstandard2.0` (shared by both) or multi-target, to unblock everything else |
| **ASP.NET MVC / Web API** | **incremental** migration: a new ASP.NET Core app in front with **YARP** proxying unmigrated routes to the old app; move endpoints one by one; **System.Web adapters** help share session and auth during the transition |
| **WebForms** | there's no direct port: rewrite the UI (Razor Pages, Blazor or an SPA), page by page, behind the same proxy |
| **WCF services** | **CoreWCF** (keeps contracts, fastest path) or re-expose as REST or gRPC |
| **Windows services** | the Generic Host worker service (`BackgroundService`) with `UseWindowsService()` or containers |
| **EF6** | EF6 runs on modern .NET (a quick win); migrate to EF Core later, context by context |
| **Configuration** | `web.config`/`app.config` → `appsettings.json` + environment variables + vault |
| **Authentication** | OWIN/Forms → ASP.NET Core authentication (cookies/OIDC); plan cookie or ticket compatibility during a side-by-side migration |
| **WinForms / WPF** | supported on modern .NET (Windows-only); retarget and modernize gradually |

### The strangler-fig migration, in practice

```
          ┌────────── YARP (new ASP.NET Core app) ──────────┐
Client ──▶│ /api/orders/*   → handled here (migrated)        │
          │ everything else → proxied to legacy app (IIS)    │
          └──────────────────────────────────────────────────┘
```

1. Put the proxy in front, with no behavior change. Deploy and verify it.
2. Migrate one route group at a time, with **characterization tests** that compare old and new responses (or shadow traffic).
3. Move shared concerns (auth, session) using adapters, or by externalizing them (Redis, an OIDC provider).
4. When nothing is proxied any more, retire the legacy app.

### Risk management

- **Characterization tests** before touching legacy behavior: capture what it does *now*, quirks included.
- **Feature parity checklists** per page or endpoint, signed off by the product owner.
- **Shadow traffic or dual-run** comparisons for critical calculations (pricing, tax).
- **Small, reversible steps.** Every step should be deployable and reversible.

---

## Interview questions

<details><summary><b>1. How would you plan an upgrade from .NET 10 to .NET 12 for 15 services?</b></summary>

Read the breaking changes, inventory the projects and packages, and baseline the tests and performance. Upgrade shared libraries first (multi-targeting if needed), then services from lowest to highest risk. For each one: bump the SDK and TFM, update packages, fix warnings, run all tests and benchmarks, soak in staging, and roll out gradually with a rollback. Document the issues for the next team.
</details>

<details><summary><b>2. How do you migrate a large ASP.NET MVC (.NET Framework) app without a big-bang rewrite?</b></summary>

Strangler fig: put a new ASP.NET Core app with YARP in front, proxy all routes to the legacy app, migrate route groups incrementally (with characterization tests), share auth and session via adapters or external stores, and retire the legacy app when nothing is proxied any more.
</details>

<details><summary><b>3. What's your strategy for WCF services?</b></summary>

CoreWCF when you need to keep SOAP contracts and existing clients. Otherwise, re-expose the operations as REST or gRPC, and migrate the clients over time with a compatibility period.
</details>

---

## Exercises

1. Upgrade a multi-project solution (one of your labs) to the newest .NET version available to you, following the checklist, with benchmark comparisons across runtimes ([Lab 9](11-labs.md#lab-9-upgrade-to-net-12)).
2. Write a one-page upgrade plan for your product at work: inventory, risks, order and rollback.
3. Prototype the YARP strangler setup: a small legacy-style app behind a new ASP.NET Core proxy, with one route migrated.

## Self-check

- [ ] I can run an LTS upgrade end to end with baselines, staged rollout and rollback.
- [ ] I know the migration paths for MVC/Web API, WebForms, WCF, Windows services and EF6.
- [ ] I can explain and set up a strangler-fig migration with YARP.

## Further reading

- Microsoft Learn: *Breaking changes in .NET* (per version), *Upgrade from ASP.NET Framework to ASP.NET Core* (incremental migration, System.Web adapters), *YARP documentation*, *CoreWCF*.
- The .NET support policy page (dotnet.microsoft.com).
