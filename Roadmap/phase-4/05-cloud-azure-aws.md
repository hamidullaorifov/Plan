# 5. Cloud Platforms: Azure and AWS for .NET Developers

> Phase 4 · Week 6 · [← Phase index](README.md)

**Why this matters:** Job postings expect "experience with Azure or AWS". In practice, that means choosing the right compute and data services, connecting to them **without passwords** (managed identities and IAM roles), keeping secrets in a vault, and knowing roughly what things cost. Go deep on **one** provider (the one your market scan from Phase 0 favored) and learn the other's names.

---

## 5.1 Service map: the same concepts, different names

| Need | Azure | AWS |
|---|---|---|
| Containers without managing a cluster | **Azure Container Apps** | **ECS on Fargate**, App Runner |
| Managed Kubernetes | AKS | EKS |
| PaaS web hosting | App Service | Elastic Beanstalk |
| Functions / serverless | Azure Functions | AWS Lambda |
| PostgreSQL | Azure Database for PostgreSQL – Flexible Server | Amazon RDS / Aurora PostgreSQL |
| SQL Server | Azure SQL Database | RDS for SQL Server |
| Document / NoSQL | Cosmos DB | DynamoDB |
| Redis | Azure Managed Redis / Azure Cache for Redis | ElastiCache |
| Queues / pub-sub | Service Bus, Event Grid, Event Hubs | SQS, SNS, EventBridge, Kinesis/MSK |
| Object storage | Blob Storage | S3 |
| Secrets | **Key Vault** | **Secrets Manager**, SSM Parameter Store |
| App configuration and feature flags | App Configuration | AppConfig |
| Identity for workloads | **Managed Identity** / workload identity | **IAM roles** (task roles, IRSA) |
| User identity | Microsoft Entra ID (External ID for customers) | Cognito, IAM Identity Center |
| Monitoring | Azure Monitor / Application Insights | CloudWatch, X-Ray |
| Container registry | ACR | ECR |
| Infrastructure as code | Bicep, ARM, Terraform | CloudFormation, CDK, Terraform |

---

## 5.2 Choosing compute

| Option | Choose it when | Avoid it when |
|---|---|---|
| **Container Apps / ECS Fargate** | the default for containerized APIs and workers: scaling (including to zero), revisions and traffic splitting, no cluster to manage | you need low-level Kubernetes features |
| **App Service** | simple web apps without containers; deployment slots | unusual runtimes; heavy background processing |
| **AKS / EKS** | a platform team exists; many services; you need Kubernetes features (operators, a service mesh) | a small team without Kubernetes skills: the operational cost dominates |
| **Functions / Lambda** | event-driven glue, spiky low-volume workloads, schedules | long-running, latency-critical or chatty workloads (cold starts, execution limits) |

> **Default recommendation for a team of 3–15 developers:** containers on Container Apps or ECS Fargate, managed PostgreSQL, managed Redis, a managed broker and a vault. Move to Kubernetes when a platform team and real needs appear.

---

## 5.3 Identity without secrets

**The best secret is no secret.** Instead of connection strings with passwords, give the *workload* an identity and grant that identity access.

### Azure: managed identity + `DefaultAzureCredential`

```csharp
// Packages: Azure.Identity, Azure.Storage.Blobs, Azure.Messaging.ServiceBus
using Azure.Core;
using Azure.Identity;

TokenCredential credential = builder.Environment.IsDevelopment()
    ? new DefaultAzureCredential()                 // dev: your az / Visual Studio login
    : new ManagedIdentityCredential(ManagedIdentityId.SystemAssigned);  // prod: explicit, no credential-chain surprises

builder.Services.AddSingleton(new BlobServiceClient(new Uri("https://shopstorage.blob.core.windows.net"), credential));
builder.Services.AddSingleton(new ServiceBusClient("shop-prod.servicebus.windows.net", credential));
```

Then grant **RBAC roles** to the app's identity: *Storage Blob Data Contributor* on the storage account, *Azure Service Bus Data Sender* on the namespace, and so on. Grant the smallest scope and role that works.

> `DefaultAzureCredential` tries a chain of credential sources. That's convenient locally, but in production it can be slow or pick an unexpected identity. Use the specific credential in production.

### AWS: IAM roles

On ECS/Fargate, attach a **task role**; on EKS, use **IAM Roles for Service Accounts (IRSA)** or EKS Pod Identity. The AWS SDK for .NET picks up credentials automatically, with no keys in configuration:

```csharp
// Package: AWSSDK.Extensions.NETCore.Setup, AWSSDK.S3
builder.Services.AddDefaultAWSOptions(builder.Configuration.GetAWSOptions());
builder.Services.AddAWSService<IAmazonS3>();
```

Write **least-privilege IAM policies**: specific actions on specific resource ARNs, never `"Action": "*"`.

---

## 5.4 Secrets and configuration

```csharp
// Azure Key Vault as a configuration source. Package: Azure.Extensions.AspNetCore.Configuration.Secrets
if (!builder.Environment.IsDevelopment())
{
    builder.Configuration.AddAzureKeyVault(
        new Uri(builder.Configuration["KeyVault:Uri"]!),
        new ManagedIdentityCredential(ManagedIdentityId.SystemAssigned));
}
// The Key Vault secret "ConnectionStrings--Shop" becomes the configuration key "ConnectionStrings:Shop"
```

| Practice | Why |
|---|---|
| Secrets in a vault; configuration in App Configuration or environment variables | separation; auditing; rotation |
| **Passwordless database connections** where supported (Entra authentication for Azure PostgreSQL/SQL, IAM authentication for RDS) | nothing to leak or rotate |
| Rotate any remaining secrets automatically | leaked keys expire |
| Separate vaults (or at least separate access) per environment | a dev compromise can't read prod |
| Never print configuration at startup | it leaks secrets into logs |

---

## 5.5 Networking essentials

- **Virtual networks:** put the app, database, cache and broker in private networks. Databases should **not** have public endpoints in production.
- **Private endpoints / PrivateLink:** reach PaaS services over private IPs.
- **Ingress:** expose only the API through a gateway or load balancer (Front Door / Application Gateway; CloudFront / ALB), with TLS and a WAF.
- **Egress:** restrict outbound traffic where possible. It's an important defense against SSRF and data exfiltration.

You don't need to be a network engineer, but you must be able to read a diagram and say what's publicly reachable and what isn't.

---

## 5.6 Cost awareness

| Habit | Example |
|---|---|
| **Budget alerts on day one** | an alert at 50%, 80% and 100% of the monthly budget |
| Know the cost drivers | compute (always-on replicas), databases (vCores, storage, IOPS), egress, logging volume |
| Scale to zero for dev and test | Container Apps minimum replicas = 0; stop dev databases at night |
| **Tag everything** | `env`, `team`, `service`, for cost reports |
| Watch telemetry ingestion | verbose logging and 100% trace sampling can cost more than the compute |
| Clean up | delete lab resource groups or stacks after each session |

---

## 5.7 Reliability basics in the cloud

- **Availability zones:** run at least 2–3 replicas spread across zones for production APIs; use zone-redundant database options.
- **Backups and restore tests:** point-in-time restore exists, but test a restore at least once.
- **Region failure** is rare. A multi-region design is expensive, so decide based on business requirements (RTO/RPO), not fashion.
- **Know the service limits:** request size, connection counts, throughput units, Lambda timeouts, Service Bus message size.

---

## Interview questions

<details><summary><b>1. How does your app authenticate to a database or storage in the cloud without passwords?</b></summary>

The workload gets an identity (an Azure managed identity, or an AWS IAM role for ECS/EKS). The SDK obtains short-lived tokens for it automatically, and access is granted through RBAC roles or IAM policies on specific resources. Nothing is stored in configuration, and nothing needs rotating.
</details>

<details><summary><b>2. Container Apps or Fargate vs Kubernetes: how do you choose?</b></summary>

Managed container platforms remove cluster operations and suit most small and medium teams. Kubernetes gives more control and features but needs operational skills. Choose it when a platform team exists or a concrete need for its features does.
</details>

<details><summary><b>3. Where do secrets live, and how does the app read them?</b></summary>

In a vault (Key Vault or Secrets Manager), accessed via the workload's identity, either loaded into configuration at startup or fetched through the SDK. Never in source control, images or pipeline logs. Where possible, avoid secrets entirely with passwordless authentication.
</details>

<details><summary><b>4. Name three common cloud cost traps.</b></summary>

Always-on, oversized compute and databases in non-production environments; telemetry ingestion (verbose logs, full trace sampling); and data egress or cross-zone traffic. Also forgotten lab resources. Use budget alerts, tags, scale-to-zero and retention policies.
</details>

---

## Exercises

1. Deploy a lab API plus PostgreSQL to Container Apps (or ECS Fargate + RDS), connected through managed identity or IAM authentication ([Lab 6](10-labs.md#lab-6-deploy-to-the-cloud)).
2. Move one secret into Key Vault or Secrets Manager, read it through configuration, and confirm it isn't in any file or pipeline log.
3. Create a budget alert, then look at the cost breakdown after a week. Which resource cost the most, and why?
4. Draw your deployment's network: what's public, what's private, and why.

## Self-check

- [ ] I can map services between Azure and AWS and pick compute for a given workload.
- [ ] I connect to cloud resources with managed identities or IAM roles and least privilege.
- [ ] I keep secrets in a vault and prefer passwordless authentication.
- [ ] I can explain a basic network layout and the main cost drivers.

## Further reading

- Microsoft Learn: *Azure Container Apps overview*, *Managed identities for Azure resources*, *Azure Key Vault configuration provider*, *Azure Architecture Center*.
- AWS docs: *Amazon ECS on Fargate*, *IAM roles for tasks*, *AWS Well-Architected Framework*.
