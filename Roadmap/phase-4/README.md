# Phase 4: Cloud, DevOps, Observability and Security → Middle Checkpoint

> **Months 9–12 (Aug 2 – Oct 31, 2027)**
> **Goal:** own a feature all the way to production, and be able to debug it there.
> ← [Main roadmap](../level-up.md) · [← Phase 3](../phase-3/README.md) · [Phase 5 →](../phase-5/README.md)

---

## How to study

Same loop: **Read → Run → Explain → Check → Apply**. Phase 4 adds one rule: **deploy for real.** Theory about containers, pipelines and telemetry only sticks once you've watched your own code fail in a real environment and found out why from logs and traces. Use a free or low-cost cloud tier, set a **budget alert first**, and tear down resources after each lab.

---

## Topics

| # | Guide | Weeks | Effort |
|---|---|---|---|
| 1 | [Containers: Dockerfiles, SDK container publishing, Compose](01-containers.md) | W1 | 6–8 h |
| 2 | [Aspire: local orchestration and service defaults](02-aspire.md) | W2 | 5–6 h |
| 3 | [Kubernetes fundamentals for application developers](03-kubernetes.md) | W3 | 8–10 h |
| 4 | [CI/CD: pipelines, branching, feature flags, deployments](04-ci-cd.md) | W4–W5 | 10–12 h |
| 5 | [Cloud platforms: Azure and AWS for .NET developers](05-cloud-azure-aws.md) | W6 | 8–10 h |
| 6 | [Infrastructure as code: Bicep and Terraform](06-infrastructure-as-code.md) | W7 | 6–8 h |
| 7 | [Observability: OpenTelemetry, logs, metrics, traces, SLOs](07-observability.md) | W8–W9 | 10–12 h |
| 8 | [Application security: OWASP, secrets, supply chain](08-security.md) | W10–W11 | 10–12 h |
| 9 | [Production readiness: load testing, incidents, runbooks](09-production-readiness.md) | W12 | 5–6 h |
| 10 | [Labs](10-labs.md) | All weeks | ~40 h |

---

## Week-by-week plan

| Week | Dates | Theory (Mon/Wed/Thu) | Tue: algorithms / system design | Sat: labs |
|---|---|---|---|---|
| W1 | Aug 2 | 01 Containers | System design: file storage | Lab 1: production-grade image |
| W2 | Aug 9 | 02 Aspire | Mixed ×2 | Lab 2: Aspire AppHost |
| W3 | Aug 16 | 03 Kubernetes | System design: news feed | Lab 3: deploy to a local cluster |
| W4 | Aug 23 | 04 CI/CD, sections 1–4 | Mixed ×2 | Lab 4: full pipeline |
| W5 | Aug 30 | 04 CI/CD, sections 5–8 | System design: job scheduler | Lab 5: feature flag and canary |
| W6 | Sep 6 | 05 Cloud | Mixed ×2 | Lab 6: deploy to the cloud |
| W7 | Sep 13 | 06 IaC | System design: payment system | Lab 7: IaC for the lab environment |
| W8 | Sep 20 | 07 Observability, sections 1–5 | Mixed ×2 | Lab 8: end-to-end trace |
| W9 | Sep 27 | 07 Observability, sections 6–9 | System design: metrics platform | Lab 9: SLO and alert |
| W10 | Oct 4 | 08 Security, sections 1–5 | Mixed ×2 | Lab 10: break and fix (OWASP) |
| W11 | Oct 11 | 08 Security, sections 6–10 | Mock system design interview | Lab 11: supply-chain hardening |
| W12 | Oct 18 | 09 Production readiness | Mock coding interview | Lab 12: load test and game day |
| W13 | Oct 25 | **Middle checkpoint review** | Mock behavioral interview | Write-ups, brag document, promotion conversation |

> **At work during this phase:** join at least one incident investigation, add one dashboard or alert that didn't exist before, and own one deployment end to end.

---

## Toolbox additions

| Tool | Purpose |
|---|---|
| Docker Desktop (with Kubernetes enabled) or kind / k3d | Containers and a local cluster |
| `kubectl`, Helm | Kubernetes CLI and packaging |
| Azure CLI (`az`) + Azure Developer CLI (`azd`), or AWS CLI | Cloud deployments |
| Bicep (via `az bicep`) or Terraform / OpenTofu | Infrastructure as code |
| Aspire CLI / templates | Local orchestration and the dashboard |
| Grafana + Prometheus + Tempo + Loki (Docker), or the Aspire dashboard | Observability backends |
| k6 | Load testing |
| Trivy | Image and dependency scanning |
| `dotnet-monitor` | Diagnostics (dumps, traces, counters) inside containers |

---

## ✅ Middle checkpoint (end of October 2027)

- [ ] I shipped at least 3 features end to end at work with minimal guidance.
- [ ] I investigated and resolved at least 1 production issue using logs, traces or metrics.
- [ ] I can deploy a containerized .NET API to the cloud through a pipeline I built, with secrets in a vault and telemetry in a dashboard.
- [ ] I can explain the OWASP risks most relevant to ASP.NET Core and how my code prevents them.
- [ ] Optional: one cloud certification (an Azure developer associate exam — check the current exam code on Microsoft Learn — or AWS Certified Developer – Associate).
- [ ] I passed a mock interview (coding, system design and behavioral) run by a Middle or Senior engineer.
- [ ] I had the promotion conversation with my lead, or started applying for Middle roles.
