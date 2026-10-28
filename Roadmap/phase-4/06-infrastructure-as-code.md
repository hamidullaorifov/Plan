# 6. Infrastructure as Code: Bicep and Terraform

> Phase 4 · Week 7 · [← Phase index](README.md)

**Why this matters:** Clicking resources together in a portal is unrepeatable, unreviewable and undocumented. Infrastructure as code (IaC) makes environments reproducible, puts changes through pull requests, and lets you rebuild after a disaster. You don't need to be a platform engineer, but you should be able to read and modify the IaC for *your* service.

---

## 6.1 Core ideas

| Idea | Meaning |
|---|---|
| **Declarative** | describe the *desired state*; the tool works out create, update and delete |
| **Idempotent** | applying the same definition twice changes nothing the second time |
| **Plan / what-if** | preview the changes before applying them. **Review the plan like a code diff** |
| **State** | Terraform tracks deployed resources in a state file; Bicep/ARM uses Azure itself as the state |
| **Drift** | someone changed a resource manually, so reality no longer matches the code. Detect it and fix it in code |
| **Modules** | reusable building blocks (a "container app with standard settings") |

| Tool | Scope | Notes |
|---|---|---|
| **Bicep** | Azure only | first-class Azure support, no state file, concise syntax |
| **Terraform / OpenTofu** | multi-cloud and SaaS (GitHub, Datadog, …) | huge ecosystem; you manage remote state and locking. OpenTofu is the open-source fork |
| **Pulumi** | multi-cloud | IaC in C# or TypeScript |
| AWS CDK / CloudFormation | AWS | CDK lets you write IaC in C# |

---

## 6.2 Bicep: a container app with a managed identity

```bicep
// infra/main.bicep
param location string = resourceGroup().location
param appName string
param image string

resource logs 'Microsoft.OperationalInsights/workspaces@2023-09-01' = {
  name: '${appName}-logs'
  location: location
  properties: {
    sku: { name: 'PerGB2018' }
    retentionInDays: 30
  }
}

resource env 'Microsoft.App/managedEnvironments@2024-03-01' = {
  name: '${appName}-env'
  location: location
  properties: {
    appLogsConfiguration: {
      destination: 'log-analytics'
      logAnalyticsConfiguration: {
        customerId: logs.properties.customerId
        sharedKey: logs.listKeys().primarySharedKey
      }
    }
  }
}

resource app 'Microsoft.App/containerApps@2024-03-01' = {
  name: appName
  location: location
  identity: { type: 'SystemAssigned' }
  properties: {
    managedEnvironmentId: env.id
    configuration: {
      ingress: {
        external: true
        targetPort: 8080
      }
    }
    template: {
      containers: [
        {
          name: 'api'
          image: image
          resources: { cpu: json('0.5'), memory: '1Gi' }
          env: [
            { name: 'ASPNETCORE_ENVIRONMENT', value: 'Production' }
          ]
          probes: [
            { type: 'Liveness', httpGet: { path: '/health/live', port: 8080 } }
            { type: 'Readiness', httpGet: { path: '/health/ready', port: 8080 } }
          ]
        }
      ]
      scale: { minReplicas: 1, maxReplicas: 5 }
    }
  }
}

output principalId string = app.identity.principalId   // grant RBAC roles to this identity
output url string = 'https://${app.properties.configuration.ingress.fqdn}'
```

```powershell
az group create -n rg-shop-dev -l westeurope
az deployment group what-if -g rg-shop-dev -f infra/main.bicep -p appName=shop-dev image=ghcr.io/acme/shop-api:1.0.42
az deployment group create  -g rg-shop-dev -f infra/main.bicep -p appName=shop-dev image=ghcr.io/acme/shop-api:1.0.42
```

> API versions (`@2024-03-01`) change over time. The Bicep VS Code extension suggests the latest ones. Use **Azure Verified Modules** (AVM) for production-grade building blocks instead of writing everything yourself.

---

## 6.3 Terraform: the same idea

```hcl
# infra/main.tf
terraform {
  required_providers {
    azurerm = { source = "hashicorp/azurerm", version = "~> 4.0" }
  }
  backend "azurerm" {}          # remote state in a storage account (configured at init)
}

provider "azurerm" {
  features {}
}

variable "app_name" { type = string }
variable "image"    { type = string }
variable "location" {
  type    = string
  default = "westeurope"
}

resource "azurerm_resource_group" "rg" {
  name     = "rg-${var.app_name}"
  location = var.location
}

resource "azurerm_log_analytics_workspace" "logs" {
  name                = "${var.app_name}-logs"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
}

resource "azurerm_container_app_environment" "env" {
  name                       = "${var.app_name}-env"
  location                   = azurerm_resource_group.rg.location
  resource_group_name        = azurerm_resource_group.rg.name
  log_analytics_workspace_id = azurerm_log_analytics_workspace.logs.id
}

resource "azurerm_container_app" "app" {
  name                         = var.app_name
  container_app_environment_id = azurerm_container_app_environment.env.id
  resource_group_name          = azurerm_resource_group.rg.name
  revision_mode                = "Single"

  identity { type = "SystemAssigned" }

  template {
    min_replicas = 1
    max_replicas = 5
    container {
      name   = "api"
      image  = var.image
      cpu    = 0.5
      memory = "1Gi"
    }
  }

  ingress {
    external_enabled = true
    target_port      = 8080
    traffic_weight {
      percentage      = 100
      latest_revision = true
    }
  }
}

output "url" { value = azurerm_container_app.app.ingress[0].fqdn }
```

```powershell
terraform init -backend-config="backend.hcl"
terraform fmt -check; terraform validate
terraform plan -out tfplan -var "app_name=shop-dev" -var "image=ghcr.io/acme/shop-api:1.0.42"
terraform apply tfplan
```

### Terraform state rules

- **Remote state with locking** (an Azure Storage blob lease, S3 with locking), never a local file in a team.
- **The state contains secrets.** Restrict access to it and encrypt it.
- **One state per environment** (or workspaces), so a dev `apply` can't touch prod.
- Never edit state by hand. Use `terraform import`, `terraform state mv` and `moved` blocks.

---

## 6.4 IaC in the pipeline

```
PR opened  → fmt/validate → plan (posted as a PR comment) → human review of the plan
PR merged  → apply to dev → apply to staging → (approval) → apply to prod
```

- The pipeline authenticates via **OIDC** with an identity that can manage *only* that environment.
- Run security scanners on the IaC (Checkov, tfsec/Trivy config scanning, the Bicep linter) to catch public storage, missing encryption and open firewall rules.
- **Separate infrastructure changes from app deployments**, or at least from image-tag-only changes. Most deployments should only change the image.

---

## 6.5 What a developer should own

| Usually yours | Usually the platform team's |
|---|---|
| the app's container app or ECS service definition, scaling, probes, environment variables | networking (VNets/VPCs, private DNS), shared clusters |
| the app's database, queue, topic and cache *instances* (using approved modules) | landing zones, policies, identity foundations |
| RBAC/IAM grants for the app's identity | organization-wide security baselines |
| dashboards and alerts for the service | the shared observability platform |

---

## Interview questions

<details><summary><b>1. Why infrastructure as code?</b></summary>

Environments become reproducible, reviewable through PRs, versioned and documented by the code itself. Changes are previewed with plan or what-if, drift is detectable, and disaster recovery becomes "re-apply" instead of guesswork.
</details>

<details><summary><b>2. What's Terraform state, and how do you handle it in a team?</b></summary>

State maps the configuration to real resources. Store it remotely with locking, restrict and encrypt it (it contains secrets), keep one state per environment, and change it only through Terraform commands or `moved` blocks.
</details>

<details><summary><b>3. Bicep or Terraform?</b></summary>

Bicep is Azure-only, has no state file and gets day-zero support for new Azure features. Terraform is multi-cloud and covers SaaS providers, at the cost of state management. Choose based on your cloud footprint and the team's existing skills.
</details>

---

## Exercises

1. Write the Bicep *or* Terraform for your lab environment: app, logs, database and vault. Deploy it, destroy it and redeploy it ([Lab 7](10-labs.md#lab-7-iac-for-the-lab-environment)).
2. Change a setting manually in the portal, then detect the drift with `what-if` or `plan`.
3. Add an IaC plan step to your pipeline that posts the plan to the PR.
4. Run a configuration scanner on your IaC and fix one finding.

## Self-check

- [ ] I can read and modify Bicep or Terraform for my service.
- [ ] I always review a plan or what-if before applying.
- [ ] I understand Terraform state handling, or Bicep's stateless model.
- [ ] IaC changes go through PRs and a pipeline with OIDC.

## Further reading

- Microsoft Learn: *Bicep documentation*, *Azure Verified Modules*.
- HashiCorp: *Terraform tutorials* (the Azure or AWS track); the OpenTofu docs.
- *Terraform: Up & Running* (Yevgeniy Brikman).
