# 10. Labs

> Phase 4 · All weeks · [← Phase index](README.md)

These labs take one small system all the way to the cloud. Use any **lab system** you built in Phases 2–3: an API, a worker, PostgreSQL, Redis and a broker. Keep each lab's `NOTES.md` with what you did, the measurements, and what broke.

> **Before any cloud lab:** create a **budget alert**, use the cheapest tiers, tag everything with `env=lab`, and delete the resource group or stack at the end of each session.

| Lab | Guide | Time |
|---|---|---|
| [Lab 1: Production-grade image](#lab-1-production-grade-image) | [01](01-containers.md) | 3 h |
| [Lab 2: Aspire AppHost](#lab-2-aspire-apphost) | [02](02-aspire.md) | 3 h |
| [Lab 3: Deploy to a local cluster](#lab-3-deploy-to-a-local-cluster) | [03](03-kubernetes.md) | 4 h |
| [Lab 4: Full pipeline](#lab-4-full-pipeline) | [04](04-ci-cd.md) | 5 h |
| [Lab 5: Feature flag and canary](#lab-5-feature-flag-and-canary) | [04](04-ci-cd.md) | 2 h |
| [Lab 6: Deploy to the cloud](#lab-6-deploy-to-the-cloud) | [05](05-cloud-azure-aws.md) | 4 h |
| [Lab 7: IaC for the lab environment](#lab-7-iac-for-the-lab-environment) | [06](06-infrastructure-as-code.md) | 4 h |
| [Lab 8: End-to-end trace](#lab-8-end-to-end-trace) | [07](07-observability.md) | 3 h |
| [Lab 9: SLO and alert](#lab-9-slo-and-alert) | [07](07-observability.md) | 2 h |
| [Lab 10: Break and fix (OWASP)](#lab-10-break-and-fix-owasp) | [08](08-security.md) | 4 h |
| [Lab 11: Supply-chain hardening](#lab-11-supply-chain-hardening) | [08](08-security.md) | 2 h |
| [Lab 12: Load test and game day](#lab-12-load-test-and-game-day) | [09](09-production-readiness.md) | 4 h |

---

## Lab 1: Production-grade image

1. Write the multi-stage, non-root, chiseled Dockerfile from [guide 1](01-containers.md) for your lab API.
2. Build three variants: a single-stage SDK image, the multi-stage `aspnet:10.0` image and the multi-stage chiseled image. Record the size of each.
3. Build with the SDK (`/t:PublishContainer`) as well and compare.
4. Change one line of C#, and time the rebuild with good vs bad layer ordering.
5. Scan all variants with Trivy and compare the CVE counts.

**Deliverable:** a table of size, rebuild time and CVEs per variant.

---

## Lab 2: Aspire AppHost

1. Add an AppHost and ServiceDefaults to your lab system: API, worker, PostgreSQL, Redis and RabbitMQ, with `WithReference` and `WaitFor`.
2. Replace hard-coded connection strings with Aspire's injected configuration.
3. Make one HTTP call between two projects via service discovery.
4. Write one `Aspire.Hosting.Testing` smoke test.

**Deliverable:** a dashboard screenshot showing all resources healthy and one cross-service trace.

---

## Lab 3: Deploy to a local cluster

1. Enable Kubernetes in Docker Desktop (or use kind/k3d). Load your image into the cluster.
2. Apply the Deployment, Service, ConfigMap, Secret and HPA from [guide 3](03-kubernetes.md). Use a separate in-cluster PostgreSQL (a Helm chart or a simple StatefulSet) or a database on the host.
3. **Break it four ways and diagnose each** with `describe` and `logs --previous`: a wrong image tag, a missing secret key, a liveness probe pointing at a database-dependent endpoint (then stop the database), and a 64 Mi memory limit.
4. Do a rolling update during a load test from the Phase 1 LoadGen tool. Record the failed requests with and without a readiness probe.

**Deliverable:** a table of failure → symptom → diagnostic command → fix.

---

## Lab 4: Full pipeline

1. Implement the [guide 4](04-ci-cd.md) pipeline: build, test (with coverage), image publish to GHCR, Trivy scan, deploy to staging and an approval-gated deploy to production (two Container Apps, two ECS services, or two namespaces in a cluster).
2. Use **OIDC** to authenticate to the cloud, with no stored cloud secrets.
3. Add branch protection requiring the CI check, and try to merge a failing PR.
4. Add an EF migration bundle step that runs before the deployment.

**Deliverable:** the workflow file and a screenshot of a successful run, including the approval step.

---

## Lab 5: Feature flag and canary

1. Add `Microsoft.FeatureManagement` and put a new endpoint behavior behind a percentage flag (backed by Azure App Configuration if you're on Azure).
2. Deploy with the flag at 0%, then roll out 10% → 50% → 100% while watching the error rate.
3. If your platform supports traffic splitting (Container Apps revisions, ingress weights), also run a **canary** of a new revision at 10% and roll it back.

**Deliverable:** a timeline of the rollout steps with the metrics at each step.

---

## Lab 6: Deploy to the cloud

1. Deploy the lab system: Container Apps (or ECS Fargate), managed PostgreSQL, managed Redis, a broker (Service Bus or SQS/SNS) and Key Vault (or Secrets Manager).
2. Connect the app to storage, the broker and (if supported) the database through **managed identity or IAM roles**, with no passwords in configuration.
3. Move every remaining secret to the vault and confirm none appear in files, images or pipeline logs.
4. After a few days, review the cost breakdown.

**Deliverable:** a network and identity diagram (what's public or private, which identity accesses what), and the cost summary.

---

## Lab 7: IaC for the lab environment

1. Express Lab 6's environment in Bicep **or** Terraform: the app, logs, database, cache, broker, vault and RBAC/IAM grants.
2. Destroy everything and recreate it from code. How long did it take?
3. Change a setting manually in the portal or console, and detect the drift with `what-if` or `plan`.
4. Add a plan step to the pipeline that posts the plan as a PR comment.

**Deliverable:** the IaC code, the timing of the rebuild from scratch and the drift output.

---

## Lab 8: End-to-end trace

1. Configure OpenTelemetry for the API and the worker (or use Aspire's ServiceDefaults), exporting OTLP to the Aspire dashboard locally and to your cloud backend or Grafana when deployed.
2. Propagate trace context through the broker (manually, as in [guide 7](07-observability.md), or via your messaging library).
3. Produce **one trace** that shows HTTP → database → outbox relay → broker → worker → email.
4. Add a business counter (`shop.orders.placed`) and a histogram (`shop.checkout.duration`), and build a RED dashboard.

**Deliverable:** a screenshot of the full trace and of the dashboard.

---

## Lab 9: SLO and alert

1. Define an SLO for one endpoint (for example, 99.5% of `POST /api/orders` succeed within 500 ms over 30 days).
2. Create a fast burn-rate alert and a slow burn-rate alert, each linking to a runbook.
3. Inject failures with a flag or a chaos strategy until the fast-burn alert fires. Follow your own runbook.

**Deliverable:** the SLO definition, the alert rules, the runbook and the notes from the drill.

---

## Lab 10: Break and fix (OWASP)

1. In a **local** copy of your lab API, introduce: an IDOR on `GET /orders/{id}`, mass assignment on an update endpoint, SQL injection via `FromSqlRaw`, an SSRF-prone "import from URL" endpoint, and verbose errors in Production.
2. Exploit each one with Bruno or curl, and write down the exploit request.
3. Fix each one and add an automated test proving the fix (two users for the IDOR test; a malicious payload for injection).
4. Add the security headers, HSTS, explicit CORS and persisted Data Protection keys.

**Deliverable:** a table of vulnerability → exploit → fix → test.

---

## Lab 11: Supply-chain hardening

1. Enable NuGet audit (transitive mode), central package management, lock files with locked-mode restores in CI, and package source mapping.
2. Add Dependabot or Renovate for NuGet, Docker and GitHub Actions; pin actions to commit SHAs.
3. Generate an SBOM for your API.
4. Add a deliberately vulnerable old package version and confirm the build fails. Then remove it.

**Deliverable:** the configuration files and the failing-build screenshot.

---

## Lab 12: Load test and game day

1. Write a k6 script with load, spike and soak profiles for your deployed lab system (staging only).
2. Find one bottleneck (a slow query, connection-pool exhaustion, GC pressure or autoscaling lag), fix it, and re-run.
3. **Game day:** while load runs, kill pods, block the database for 60 s and make the payments mock return 503s. Check the probes, breakers, fallbacks, alerts and runbooks.
4. Write a mini postmortem for the worst surprise you found.

**Deliverable:** before/after load-test numbers and the postmortem.
