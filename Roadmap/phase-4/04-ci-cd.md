# 4. CI/CD: Pipelines, Branching, Feature Flags and Deployments

> Phase 4 · Weeks 4–5 · [← Phase index](README.md)
> Week 4 covers sections 4.1–4.4; week 5 covers 4.5–4.8.

**Why this matters:** "It works on my machine" isn't a deliverable. A Middle developer can change the pipeline, understand why a build failed, ship safely behind a feature flag, and roll back without panic. Fast, reliable delivery is one of the strongest predictors of team performance (the DORA research).

---

## 4.1 Continuous integration, continuous delivery and continuous deployment

| Term | Meaning |
|---|---|
| **Continuous integration** | everyone merges small changes to `main` at least daily; every push is built and tested automatically |
| **Continuous delivery** | every green `main` build *can* be released to production with one click |
| **Continuous deployment** | every green `main` build *is* released automatically |

**The DORA metrics** measure delivery performance: **deployment frequency**, **lead time for changes**, **change failure rate** and **time to restore service**. Small batches improve all four.

---

## 4.2 Branching strategies

| Strategy | How | Fits |
|---|---|---|
| **Trunk-based development** | short-lived branches (hours to 2 days) merged to `main`; unfinished work hidden behind **feature flags** | most web and API teams; enables CI/CD |
| **GitHub flow** | a feature branch → PR → merge to `main` → deploy | simple, close to trunk-based |
| **GitFlow** | long-lived `develop`, `release/*` and `hotfix/*` branches | versioned products with scheduled releases; heavy for services |

> Long-lived feature branches cause merge hell and late integration bugs. Prefer small PRs and flags.

**Versioning:** Semantic Versioning (`MAJOR.MINOR.PATCH`) for libraries and public APIs. For services, a build-based version (`1.4.2+sha.abc1234`) that's traceable to a commit is usually enough. Tools such as MinVer and Nerdbank.GitVersioning derive versions from Git tags.

---

## 4.3 A complete GitHub Actions pipeline

```yaml
# .github/workflows/ci-cd.yml
name: ci-cd

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read
  packages: write        # push to GHCR
  id-token: write        # OIDC login to the cloud (no stored cloud secrets)

env:
  IMAGE: ghcr.io/${{ github.repository_owner }}/shop-api

jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'
      - run: dotnet restore
      - run: dotnet build --no-restore -c Release      # TreatWarningsAsErrors + NuGet audit fail here
      - run: dotnet test --no-build -c Release --coverage --coverage-output-format cobertura
      - uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: '**/TestResults/*.cobertura.xml'

  image:
    needs: build-test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    outputs:
      tag: ${{ steps.meta.outputs.tag }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'
      - id: meta
        run: echo "tag=1.0.${{ github.run_number }}-${GITHUB_SHA::7}" >> "$GITHUB_OUTPUT"
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - run: >
          dotnet publish src/Shop.Api -c Release /t:PublishContainer
          -p ContainerRegistry=ghcr.io
          -p ContainerRepository=${{ github.repository_owner }}/shop-api
          -p ContainerImageTag=${{ steps.meta.outputs.tag }}
      - name: Scan image
        uses: aquasecurity/trivy-action@master     # pin to a released version or commit SHA in real pipelines
        with:
          image-ref: ${{ env.IMAGE }}:${{ steps.meta.outputs.tag }}
          severity: CRITICAL,HIGH
          exit-code: '1'
          ignore-unfixed: true

  deploy-staging:
    needs: image
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: azure/login@v2
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - run: >
          az containerapp update --name shop-api-staging --resource-group rg-shop-staging
          --image ${{ env.IMAGE }}:${{ needs.image.outputs.tag }}
      - run: curl --fail --retry 10 --retry-delay 5 https://staging.shop.example/health/ready

  deploy-prod:
    needs: [image, deploy-staging]
    runs-on: ubuntu-latest
    environment: production          # requires reviewer approval (configured in repository settings)
    steps:
      - uses: azure/login@v2
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - run: >
          az containerapp update --name shop-api --resource-group rg-shop-prod
          --image ${{ env.IMAGE }}:${{ needs.image.outputs.tag }}
```

**What this demonstrates:**

- **Build once, deploy the same artifact everywhere.** Staging and production run the identical image tag, never a rebuild.
- **OIDC federation** (`id-token: write` plus `azure/login`) instead of long-lived cloud secrets in GitHub. On AWS, use `aws-actions/configure-aws-credentials` with `role-to-assume`.
- **Environments with protection rules** give you manual approval, deployment history and environment-scoped secrets.
- **Quality gates:** warnings as errors, tests, coverage artifacts, an image scan and a smoke test after deployment.

> Pin third-party actions to a **full commit SHA** (or at least a released major version) and keep them updated with Dependabot. Actions are supply-chain code that runs with your permissions.

---

## 4.4 Pipeline hygiene

| Practice | Why |
|---|---|
| A PR build under ~10 minutes | slow CI kills small PRs and trunk-based development |
| Cache NuGet packages (`actions/setup-dotnet` caching, or `actions/cache` on `~/.nuget/packages`) | faster restores |
| Run unit tests and integration tests (Testcontainers) in parallel jobs | speed |
| Required status checks plus branch protection on `main` | nothing merges red |
| Fix or quarantine flaky tests immediately | otherwise people learn to ignore red builds |
| Pipelines as code, reviewed like code | auditability |

---

## 4.5 Feature flags

**Deploying** (code in production) and **releasing** (users see the feature) are separate decisions.

```csharp
// Package: Microsoft.FeatureManagement.AspNetCore
builder.Services.AddFeatureManagement();

app.MapPost("/api/checkout", async (CheckoutRequest req, IFeatureManager features, CheckoutService checkout, CancellationToken ct) =>
    await features.IsEnabledAsync("NewCheckout")
        ? await checkout.NewFlowAsync(req, ct)
        : await checkout.LegacyFlowAsync(req, ct));
```

```json
{
  "FeatureManagement": {
    "NewCheckout": {
      "EnabledFor": [
        { "Name": "Microsoft.Percentage", "Parameters": { "Value": 10 } }
      ]
    }
  }
}
```

| Flag type | Lifetime | Example |
|---|---|---|
| **Release toggle** | days to weeks; **remove after rollout** | `NewCheckout` |
| **Ops toggle / kill switch** | long-lived | `DisableRecommendations` during an incident |
| **Experiment** | weeks | A/B pricing page |
| **Permission / entitlement** | long-lived | premium features |

**Rules:**

- Store flags centrally (Azure App Configuration, LaunchDarkly, Unleash) to change them without redeploying.
- **Test both paths.**
- **Delete release flags** within weeks. Stale flags are technical debt and hidden complexity.
- Give every flag an owner and an expiry date.

---

## 4.6 Database migrations in pipelines

```powershell
# Build a self-contained migration executable from your EF Core model
dotnet ef migrations bundle -p src/Shop.Api -o efbundle --self-contained -r linux-x64

# Run it in the pipeline (or as a Kubernetes Job) BEFORE deploying the new app version
./efbundle --connection "$env:SHOP_DB_CONNECTION"
```

Alternatives: idempotent SQL scripts (`dotnet ef migrations script --idempotent`) reviewed by a DBA, or tools such as DbUp and Flyway.

### Expand → migrate → contract (zero-downtime schema changes)

Renaming `customers.name` to `full_name` in one step breaks old pods during a rolling update. Instead:

1. **Expand:** add `full_name` (nullable); the new code writes **both** columns and reads `full_name`, falling back to `name`.
2. **Migrate:** backfill `full_name` in batches.
3. **Contract:** in a *later* release, stop using `name`; then drop it.

**Never** combine "drop a column" with the release that stops using it.

---

## 4.7 Deployment strategies

| Strategy | How | Rollback | Cost |
|---|---|---|---|
| **Rolling** (the Kubernetes default) | replace pods gradually | roll forward or undo | low |
| **Blue/green** | run the new version beside the old; switch all traffic at once | switch back instantly | 2× resources during the switch |
| **Canary** | route 1% → 10% → 50% → 100% of traffic to the new version, watching metrics | route back to 0% | needs traffic splitting (Container Apps revisions, a service mesh, ingress weights) |
| **Feature-flag rollout** | deploy dark; enable for a percentage of users | turn the flag off | needs flag discipline |

**Automated rollback triggers:** error rate, p95 latency or a failing SLO burn rate during a canary step ([guide 7](07-observability.md)).

---

## 4.8 Release checklist (for a Middle developer)

- [ ] Backward-compatible API, message and schema changes (or a coordinated plan).
- [ ] Migrations tested on a copy of production-like data; rollback plan written down.
- [ ] Feature flag in place for risky changes; default **off**.
- [ ] Dashboards and alerts ready *before* the release.
- [ ] Smoke test after deployment, automated.
- [ ] Release notes and changelog updated.
- [ ] You know how to roll back, and you've practiced it.

---

## Interview questions

<details><summary><b>1. Continuous delivery vs continuous deployment?</b></summary>

With continuous delivery, every green build *can* go to production, with a manual decision. With continuous deployment, every green build *does* go automatically. Both require CI: small, frequently merged changes that are built and tested automatically.
</details>

<details><summary><b>2. Why "build once, deploy many"?</b></summary>

Rebuilding per environment can produce a different artifact (other dependency versions or flags) from the one that was tested. Promoting the same immutable image or package through the environments guarantees that what you tested is what you run.
</details>

<details><summary><b>3. How do you change a database column without downtime?</b></summary>

Expand → migrate → contract. Add the new column and write to both; backfill; switch reads; remove the old column in a later release. Each step stays compatible with both the old and the new app versions running during a rolling deployment.
</details>

<details><summary><b>4. What are feature flags for, and what's their risk?</b></summary>

They separate deployment from release (dark launches, percentage rollouts, kill switches, experiments). The risks are combinatorial complexity, untested code paths and stale flags. Mitigate them with owners, expiry dates, tests for both paths and prompt removal.
</details>

<details><summary><b>5. Canary vs blue/green?</b></summary>

Blue/green switches all traffic at once between two full environments, with instant rollback and double resources. A canary shifts traffic gradually while watching metrics, so the blast radius is limited, but it needs traffic splitting and good telemetry.
</details>

---

## Exercises

1. Build the pipeline above for a lab API (adapt the cloud steps to your provider) ([Lab 4](10-labs.md#lab-4-full-pipeline)).
2. Add a branch protection rule requiring the CI check, and try to merge a red PR.
3. Put a new endpoint behind a percentage flag, deploy it, and roll it out to 10% → 100% ([Lab 5](10-labs.md#lab-5-feature-flag-and-canary)).
4. Perform an expand-migrate-contract column rename across three PRs on a lab project.

## Self-check

- [ ] I can write a CI/CD pipeline with build, test, image, scan and gated deployments using OIDC.
- [ ] I can explain trunk-based development and feature flags, and I remove stale flags.
- [ ] I run migrations as a separate step and design backward-compatible schema changes.
- [ ] I can choose between rolling, blue/green and canary deployments.

## Further reading

- *Accelerate* (Forsgren, Humble, Kim): the research behind the DORA metrics.
- *Continuous Delivery* (Humble and Farley).
- GitHub Docs: *Security hardening for GitHub Actions*, *About security hardening with OpenID Connect*.
- Microsoft Learn: *.NET feature management*, *EF Core migration bundles*.
