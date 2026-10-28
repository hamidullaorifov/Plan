# Domain 3 – Deployment (24%)

What this domain tests:
- Preparing application artifacts to be deployed (packaging, dependencies, container images, config)
- Testing applications in development environments (SAM local, stages, mocks)
- Automating deployment testing (CI/CD, test events, environments)
- Deploying code using AWS CI/CD services (CodePipeline, CodeBuild, CodeDeploy, SAM, CloudFormation, Beanstalk; deployment strategies and rollback)

---

## 1. Deployment strategies (concepts)

| Strategy | How | Downtime | Rollback | Cost |
|---|---|---|---|---|
| **All at once / in-place** | Update all instances simultaneously | **Yes** | Redeploy (slow) | Cheapest |
| **Rolling** | Update in batches | No (reduced capacity) | Redeploy | No extra |
| **Rolling with additional batch** | Launch a new batch first, then roll | No (full capacity) | Redeploy | Small extra |
| **Immutable** | Deploy to brand-new instances, then swap | No | **Fast** (terminate new instances) | High during deploy |
| **Blue/Green** | Full new environment; shift traffic (DNS/LB) | No | **Fastest** (switch back) | Double during deploy |
| **Canary** | Small % of traffic first, then the rest | No | Fast | Low |
| **Linear** | Shift traffic in equal increments over time | No | Fast | Low |
| **Traffic splitting (A/B)** | % of traffic to the new version for evaluation | No | Fast | Medium |

---

## 2. AWS Elastic Beanstalk

### 2.1 Concepts
- A **PaaS** that handles capacity provisioning, load balancing, auto scaling, and health monitoring. **You manage only the code.** Elastic Beanstalk itself is free; you pay for the underlying resources (EC2, ALB, RDS).
- **Application** → **application versions** (code bundles in S3) → **environments** (running instances of one version).
- **Environment tiers:**
  - **Web server tier:** ALB/NLB + Auto Scaling group.
  - **Worker tier:** processes **SQS** messages via the `sqsd` daemon, which POSTs messages to your app on localhost. Use it for long-running background jobs. A **`cron.yaml`** file defines periodic tasks.
- Environment types: **single instance** (dev) or **load balanced / high availability** (prod).
- Platforms: Java, .NET, Node.js, Python, PHP, Ruby, Go, Docker (single container or ECS-based multi-container), and custom platforms (Packer).

### 2.2 Deployment policies (very high yield)

| Policy | Downtime | Deploy time | Capacity during deploy | Rollback | Notes |
|---|---|---|---|---|---|
| **All at once** | **Yes** | Fastest | Zero | Manual redeploy | Dev environments |
| **Rolling** | No | Slower | **Reduced** by batch size | Manual redeploy | Both versions run simultaneously |
| **Rolling with additional batch** | No | Slower | **Full** capacity | Manual redeploy | Small extra cost; good for prod |
| **Immutable** | No | Slowest | Full (double during deploy) | **Fast**: terminate new ASG | New instances in a **temporary ASG**; safest in-place option |
| **Traffic splitting** | No | Slow | Full | **Fast**, automatic | **Canary testing**: % of traffic to new instances for an evaluation period |
| **Blue/Green** | No | Varies | Full | **Swap URL back** | **Not a built-in policy**: create a second environment and use **"Swap environment URLs"** (CNAME swap) or Route 53 weighted records |

### 2.3 Configuration
- **`.ebextensions/`** folder in the source bundle root, containing YAML/JSON files ending in **`.config`**. It can set `option_settings`, install packages, create files, run `commands` / `container_commands`, and define extra resources (CloudFormation syntax).
  - `commands`: run **before** the app and web server are set up.
  - `container_commands`: run **after** the app is extracted, before it is deployed. **`leader_only: true`** runs on a single instance (e.g., DB migrations).
- **Configuration precedence** (highest first): settings applied directly to the environment (console/CLI/API) > **saved configurations** > `.ebextensions` > default values.
- **`Procfile`:** commands to start the app processes. **`Buildfile`:** build commands. **`.platform/`** hooks and nginx config on Amazon Linux 2/2023.
- **`env.yaml`:** environment manifest (name, solution stack, links).
- **EB CLI:** `eb init`, `eb create`, `eb deploy`, `eb status`, `eb logs`, `eb config`, `eb swap`, `eb terminate`.
- **Application version lifecycle policy:** max **1,000 application versions**. Configure a lifecycle policy (by count or age) to delete old versions, with an option to keep source bundles in S3.
- **Database:** an RDS DB created *inside* an EB environment is tied to that environment's lifecycle (deleted with it). **For production, create RDS separately** and pass the connection string via env vars. To decouple an existing one: take a snapshot, enable deletion protection, create a new environment without RDS, point it at the DB, then terminate the old environment.
- **HTTPS:** load-balancer listener config via `.ebextensions` (with an ACM certificate). Redirect HTTP to HTTPS in the app or via ALB rules.
- **X-Ray:** enable via console option or `.ebextensions` (`XRayEnabled: true`) and give the instance profile X-Ray permissions.
- Under the hood, EB uses **CloudFormation**.
- **Cloning** environments and **migrating platform versions**: create a new environment and swap URLs.
- Health: basic vs **enhanced health reporting**.

---

## 3. AWS CloudFormation

### 3.1 Template anatomy
```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Sample
Metadata: {}
Parameters:            # Inputs (Type, Default, AllowedValues, NoEcho, AWS::SSM::Parameter::Value<String>)
  Env: { Type: String, AllowedValues: [dev, prod] }
Mappings:              # Static lookup tables (e.g., AMI per Region)
  RegionMap: { us-east-1: { AMI: ami-123 } }
Conditions:            # Conditional creation
  IsProd: !Equals [!Ref Env, prod]
Transform: AWS::Serverless-2016-10-31   # Macros (SAM, AWS::Include)
Resources:             # The ONLY required section
  MyBucket:
    Type: AWS::S3::Bucket
    Condition: IsProd
    DeletionPolicy: Retain
Outputs:               # Values to display/export
  BucketName:
    Value: !Ref MyBucket
    Export: { Name: !Sub "${AWS::StackName}-BucketName" }
```
- **Resources is the only mandatory section.**
- Resource type format: `AWS::<Service>::<Resource>` (e.g., `AWS::DynamoDB::Table`).

### 3.2 Intrinsic functions (know what each returns)
| Function | Purpose |
|---|---|
| `!Ref` | Parameter value, or resource's **physical ID** (often the name/ID) |
| `!GetAtt Resource.Attribute` | A resource attribute (e.g., `MyFn.Arn`, `MyBucket.DomainName`) |
| `!Sub` | String substitution: `!Sub "arn:aws:s3:::${BucketName}/*"` |
| `!Join [delim, [list]]` | Join values |
| `!Split`, `!Select` | Split a string / select by index |
| `!FindInMap [Map, TopKey, SecondKey]` | Look up a value in Mappings |
| `!ImportValue` | Import an **exported** output from another stack (**cross-stack reference**) |
| `!If`, `!Equals`, `!And`, `!Or`, `!Not` | Conditions |
| `!GetAZs`, `!Cidr`, `!Base64` | AZ list, CIDR blocks, encode user data |

**Pseudo parameters:** `AWS::Region`, `AWS::AccountId`, `AWS::StackName`, `AWS::StackId`, `AWS::Partition`, `AWS::URLSuffix`, `AWS::NoValue` (removes a property when used with `!If`).

### 3.3 Stacks and updates
- **Change sets:** preview what an update will change (add, modify, **replace**) before executing.
- **Update behaviors:** no interruption, some interruption, or **replacement** (new physical resource, e.g., changing a DynamoDB table name).
- **Rollback:** on create failure, the stack rolls back (deletes resources) by default. Use `--disable-rollback` / `--on-failure DO_NOTHING` to troubleshoot. On update failure, it rolls back to the last known good state. A stack stuck in `UPDATE_ROLLBACK_FAILED` needs `ContinueUpdateRollback` (after fixing the cause, optionally skipping resources).
- **Stack policies:** JSON that protects resources from updates (e.g., deny updates to a prod DB).
- **Termination protection:** prevents stack deletion.
- **Drift detection:** finds manual changes made outside CloudFormation.
- **Capabilities:** `CAPABILITY_IAM`, `CAPABILITY_NAMED_IAM` (when creating IAM resources; NAMED if they have custom names), and `CAPABILITY_AUTO_EXPAND` (macros, nested stacks with transforms such as SAM). Missing them causes an `InsufficientCapabilities` error.
- **Service role:** CloudFormation can use a role you pass (`iam:PassRole`), so users don't need direct resource permissions.

### 3.4 Resource attributes
- **`DeletionPolicy`:** `Delete` (default for most), **`Retain`** (keep the resource), **`Snapshot`** (RDS, EBS, ElastiCache, Redshift, Neptune, DocumentDB: take a final snapshot). Note: **S3 buckets must be empty to be deleted**; use a custom resource to empty them or `Retain`.
- **`UpdateReplacePolicy`:** what happens to the old resource when it is replaced.
- **`DependsOn`:** explicit ordering (`!Ref`/`!GetAtt` create implicit dependencies).
- **`CreationPolicy`** + **`cfn-signal`:** wait for the EC2 instance or ASG to signal success (e.g., after `cfn-init` finishes). On timeout or failure, the stack rolls back.
- **`WaitCondition`** + `WaitConditionHandle`: wait for external signals (older; CreationPolicy is preferred for EC2/ASG).
- **`UpdatePolicy`:** ASG rolling update / replacing update, Lambda alias CodeDeploy (`CodeDeployLambdaAliasUpdate`).
- **Helper scripts:** `cfn-init` (reads `AWS::CloudFormation::Init` metadata to install packages, files and services), `cfn-signal`, `cfn-hup` (detects metadata changes), `cfn-get-metadata`. Logs are in `/var/log/cfn-init.log` and `/var/log/cloud-init-output.log`.

### 3.5 Multi-stack patterns
- **Cross-stack references:** `Outputs` → `Export` → `!ImportValue`. You **cannot delete** a stack whose exports are imported by another stack. Export names must be unique per Region.
- **Nested stacks:** `AWS::CloudFormation::Stack` resource for **reusable components** (e.g., a standard ALB template). Update via the root stack.
  - Cross-stack = shared resources with independent lifecycles (e.g., a VPC used by many apps). Nested = reuse of a template, managed as one unit.
- **StackSets:** deploy stacks across **multiple accounts and Regions** in a single operation (with AWS Organizations).
- **Custom resources:** `Custom::MyThing` / `AWS::CloudFormation::CustomResource` backed by **Lambda** (or SNS), for unsupported resources or custom logic (e.g., empty an S3 bucket before deletion, look up an AMI). The Lambda **must send a response** to the pre-signed S3 `ResponseURL`, otherwise the stack hangs until it times out.
- **Modules, macros** (`Transform`), **registry** (third-party resource types), **Hooks** (proactive compliance checks).
- **CloudFormation Guard** (cfn-guard) for policy-as-code; **cfn-lint** for template linting.

### 3.6 Lambda in CloudFormation
- `Code` property: `ZipFile` (inline code, small), or `S3Bucket` + `S3Key` (+ `S3ObjectVersion`), or `ImageUri`.
- **Gotcha:** if you upload new code to the same S3 key, CloudFormation **won't detect the change**. Change the `S3Key`, use `S3ObjectVersion` (bucket versioning), or use `aws cloudformation package` / `sam package`, which uploads with a content hash.
- `aws cloudformation package` uploads local artifacts to S3 and rewrites the template. `aws cloudformation deploy` creates or updates the stack through a change set.

---

## 4. AWS SAM (Serverless Application Model)

- An extension of CloudFormation for serverless apps. **`Transform: AWS::Serverless-2016-10-31`** is required.
- **Resource types:** `AWS::Serverless::Function`, `::Api` (REST), `::HttpApi`, `::SimpleTable` (DynamoDB with a primary key only), `::LayerVersion`, `::Application` (nested app from SAR), `::StateMachine`, `::Connector` (simplified permissions between resources), `::GraphQLApi`.
- **`Globals` section:** shared properties (Runtime, Timeout, MemorySize, Environment, Tracing) for all functions and APIs.
- **Events** on a function: `Api`, `HttpApi`, `S3`, `SQS`, `SNS`, `Kinesis`, `DynamoDB`, `Schedule`/`ScheduleV2`, `EventBridgeRule`, `CloudWatchLogs`, `Cognito`, and more. SAM creates the trigger and permissions.
- **Policy templates:** `DynamoDBCrudPolicy`, `DynamoDBReadPolicy`, `S3ReadPolicy`, `S3CrudPolicy`, `SQSPollerPolicy`, `SQSSendMessagePolicy`, `SNSPublishMessagePolicy`, `LambdaInvokePolicy`, `SSMParameterReadPolicy`, `KMSDecryptPolicy`, `AWSSecretsManagerGetSecretValuePolicy`.

```yaml
Transform: AWS::Serverless-2016-10-31
Globals:
  Function: { Runtime: python3.12, Timeout: 10, Tracing: Active }
Resources:
  OrdersFn:
    Type: AWS::Serverless::Function
    Properties:
      Handler: app.handler
      CodeUri: src/
      AutoPublishAlias: live
      DeploymentPreference:
        Type: Canary10Percent5Minutes
        Alarms: [!Ref ErrorsAlarm]
        Hooks: { PreTraffic: !Ref PreTrafficFn, PostTraffic: !Ref PostTrafficFn }
      Policies:
        - DynamoDBCrudPolicy: { TableName: !Ref OrdersTable }
      Events:
        Api: { Type: Api, Properties: { Path: /orders, Method: post } }
  OrdersTable:
    Type: AWS::Serverless::SimpleTable
```

- **SAM + CodeDeploy (safe Lambda deployments):**
  - **`AutoPublishAlias`** publishes a new version on each change and points the alias at it.
  - **`DeploymentPreference.Type`:** `Canary10Percent5Minutes`, `Canary10Percent10Minutes`, `Canary10Percent15Minutes`, `Canary10Percent30Minutes`, `Linear10PercentEvery1Minute`, `Linear10PercentEvery2Minutes`, `Linear10PercentEvery3Minutes`, `Linear10PercentEvery10Minutes`, `AllAtOnce`.
  - **Alarms** trigger an automatic **rollback**. **Hooks:** **PreTraffic** / **PostTraffic** Lambda functions run validation tests.
- **SAM CLI commands:**
  | Command | Purpose |
  |---|---|
  | `sam init` | Scaffold a project from templates |
  | `sam build` | Resolve dependencies and build into `.aws-sam/build` |
  | `sam local invoke` | Run a function locally in Docker with an event (`-e event.json`) |
  | `sam local start-api` | Run a local API Gateway emulator |
  | `sam local start-lambda` | Local Lambda endpoint (for SDK tests) |
  | `sam local generate-event` | Generate sample events (s3, sqs, apigateway...) |
  | `sam validate` | Validate the template |
  | `sam package` | Upload artifacts to S3 and output the packaged template (implicit in `deploy`) |
  | `sam deploy --guided` | Deploy (creates `samconfig.toml`) |
  | `sam sync --watch` | **SAM Accelerate**: fast sync of code changes to the cloud for development |
  | `sam logs`, `sam traces` | Fetch CloudWatch logs and X-Ray traces |
  | `sam pipeline init` | Generate CI/CD pipeline config |
  | `sam delete` | Delete the stack |
- SAM templates **are** CloudFormation templates, so you can mix in any `AWS::` resources.

---

## 5. AWS CDK (Cloud Development Kit)
- Define infrastructure in **TypeScript, JavaScript, Python, Java, C#, or Go**. It **synthesizes to CloudFormation**.
- **Constructs:**
  - **L1** (`Cfn*`, e.g., `CfnBucket`): 1:1 with CloudFormation resources.
  - **L2** (e.g., `s3.Bucket`): intent-based with sensible defaults and helper methods like `bucket.grantRead(fn)`.
  - **L3 (patterns)** (e.g., `ApiGatewayToLambda`, `ecsPatterns.ApplicationLoadBalancedFargateService`): multiple resources.
- Structure: **App → Stacks → Constructs**.
- **Commands:** `cdk init`, **`cdk bootstrap`** (creates the CDKToolkit stack with an S3 bucket and ECR repo for assets, **once per account/Region**; deploy fails without it), `cdk synth` (emit the CloudFormation template), `cdk diff`, `cdk deploy`, `cdk destroy`, `cdk watch` / hotswap (dev).
- Testing: the **assertions** module (fine-grained `Template.fromStack(...).hasResourceProperties(...)` and snapshot tests).
- CDK vs SAM: both produce CloudFormation. SAM is YAML and serverless-focused with local testing. CDK is general-purpose code. You can use `sam local` on CDK-synthesized templates.

---

## 6. AWS Code* CI/CD services

### 6.1 Source
- **AWS CodeCommit:** managed Git. **AWS closed it to new customers in July 2024**, though it still appears in older exam questions. Know that it supports IAM auth, Git credentials, SSH, `git-remote-codecommit`, triggers (SNS/Lambda), and EventBridge notifications.
- Modern source: **GitHub, GitHub Enterprise, GitLab, Bitbucket via CodeConnections** (formerly CodeStar Connections), or **S3**, or **ECR** in CodePipeline.

### 6.2 AWS CodeBuild
- Fully managed build service. Pay per build minute. Scales automatically with no build servers to manage.
- **`buildspec.yml` at the root** of the source (or override the name or path in the project settings).

```yaml
version: 0.2
env:
  variables: { NODE_ENV: production }
  parameter-store: { DB_URL: /myapp/prod/db-url }
  secrets-manager: { API_KEY: prod/api:key }
  exported-variables: [IMAGE_TAG]
phases:
  install:
    runtime-versions: { nodejs: 20 }
    commands: [npm ci]
  pre_build:
    commands:
      - aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $REPO
  build:
    commands: [npm test, npm run build, docker build -t $REPO:$IMAGE_TAG .]
  post_build:
    commands: [docker push $REPO:$IMAGE_TAG]
reports:
  unit-tests: { files: [junit.xml], file-format: JUNITXML }
artifacts:
  files: ['**/*']
  base-directory: dist
cache:
  paths: ['/root/.npm/**/*']
```
- **Phases:** `install` → `pre_build` → `build` → `post_build`. `finally` blocks are possible.
- **Artifacts** go to S3 (encrypted with KMS). **Cache** to S3 or local (Docker layer, source, custom) speeds up builds.
- **Environment:** managed images (Amazon Linux, Ubuntu, Windows) or custom Docker images (ECR/Docker Hub). Compute types include Lambda compute for fast short builds.
- **Docker builds** require **privileged mode** on the project.
- **VPC access:** configure VPC, subnets and security groups to reach private resources (e.g., RDS for integration tests).
- **Service role:** permissions for CodeBuild to access S3, ECR, CloudWatch Logs, Parameter Store, and so on.
- **Logs** go to CloudWatch Logs and/or S3. **Metrics** in CloudWatch. **Build notifications** via EventBridge.
- **Test reports** (JUnit, Cucumber, NUnit, and others) and **code coverage reports**.
- **Run locally** with the CodeBuild local agent (Docker) to debug the buildspec.
- **Timeout:** default 60 min (configurable). A build in **queued** status times out separately.
- **Batch builds** (matrix/graph builds) for parallel builds.
- Common env vars: `CODEBUILD_RESOLVED_SOURCE_VERSION` (commit ID), `CODEBUILD_BUILD_ID`, `AWS_REGION`.

### 6.3 AWS CodeDeploy
- Automates deployments to **EC2/on-premises**, **Lambda**, and **ECS**.
- **EC2/on-premises requires the CodeDeploy agent** on the instances. Instances need an **instance profile** with S3 read access (for the revision). Target instances are selected by **tags** or **ASG**.
- Concepts: **Application** → **Deployment group** (targets + config + alarms + rollback) → **Revision** (app files + `appspec.yml`, in S3 or GitHub).
- **Deployment types:** **in-place** (EC2/on-prem only) or **blue/green** (EC2 with a new ASG; Lambda; ECS).

**Deployment configurations**
| Platform | Configs |
|---|---|
| EC2/on-prem | `CodeDeployDefault.AllAtOnce`, **`HalfAtATime`**, **`OneAtATime`**, or custom (min healthy hosts) |
| Lambda | `LambdaCanary10Percent5Minutes` (and 10/15/30), `LambdaLinear10PercentEvery1Minute` (and 2/3/10), `LambdaAllAtOnce` |
| ECS | `ECSCanary10Percent5Minutes`, `ECSLinear10PercentEvery1Minutes`, `ECSAllAtOnce` |

**`appspec.yml` for EC2/on-prem** (must be at the **root** of the revision):
```yaml
version: 0.0
os: linux
files:
  - source: /
    destination: /var/www/html
permissions: []
hooks:
  ApplicationStop:   [{ location: scripts/stop.sh, timeout: 60, runas: root }]
  BeforeInstall:     [{ location: scripts/before_install.sh }]
  AfterInstall:      [{ location: scripts/after_install.sh }]
  ApplicationStart:  [{ location: scripts/start.sh }]
  ValidateService:   [{ location: scripts/validate.sh }]
```

**EC2 in-place hook order (memorize):**
`ApplicationStop` → `DownloadBundle`\* → `BeforeInstall` → `Install`\* → `AfterInstall` → `ApplicationStart` → `ValidateService`
(\* = reserved for the CodeDeploy agent; you can't script these.)
With a load balancer: `BeforeBlockTraffic` → `BlockTraffic`\* → `AfterBlockTraffic` at the start, and `BeforeAllowTraffic` → `AllowTraffic`\* → `AfterAllowTraffic` at the end.

**Lambda appspec:** `Resources` (function name, alias, `CurrentVersion`, `TargetVersion`) and hooks **`BeforeAllowTraffic`** and **`AfterAllowTraffic`** (Lambda validation functions).

**ECS appspec:** `TaskDefinition`, `ContainerName`, `ContainerPort`; hooks `BeforeInstall`, `AfterInstall`, `AfterAllowTestTraffic`, `BeforeAllowTraffic`, `AfterAllowTraffic`. Uses an ALB with production and test listeners.

**Rollbacks:**
- Automatic rollback on **deployment failure** or when a **CloudWatch alarm** triggers.
- A rollback **redeploys the last known good revision as a new deployment** (new deployment ID), not a "restore".
- If `ApplicationStop` fails because the previous revision's script is broken, deploy with `--ignore-application-stop-failures`.

**Troubleshooting:**
- `InvalidSignatureException`: the instance clock is out of sync.
- Deployment stuck or failing: check the agent is running (`/var/log/aws/codedeploy-agent/`), the IAM instance profile, and the appspec location and format.
- Too few healthy instances: `HEALTH_CONSTRAINTS` error.

### 6.4 AWS CodePipeline
- Orchestrates CI/CD as **stages** (Source, Build, Test, Deploy, Approval, Invoke), each with **actions** (sequential via `runOrder`, or parallel).
- **Artifacts** pass between stages through the **S3 artifact store** (encrypted with KMS; cross-Region actions need an artifact bucket in each Region).
- **Action providers:** Source (GitHub/GitLab/Bitbucket via CodeConnections, CodeCommit, S3, ECR), Build (CodeBuild, Jenkins), Test (CodeBuild, third-party), Deploy (**CloudFormation**, CodeDeploy, Elastic Beanstalk, ECS, S3, Service Catalog, AppConfig), **Manual approval** (with SNS notification; the approver needs `codepipeline:PutApprovalResult`), Invoke (**Lambda**, Step Functions).
- **CloudFormation deploy action modes:** `CREATE_UPDATE`, `DELETE_ONLY`, `REPLACE_ON_FAILURE`, **`CHANGE_SET_REPLACE` + `CHANGE_SET_EXECUTE`** (create a change set, add manual approval, then execute). Template parameter overrides and config files.
- **Triggers:** EventBridge on source changes (recommended over polling), webhooks, Git tags/branches/file-path filters (V2 pipelines).
- **State changes** emit **EventBridge** events. Use them to notify (SNS) or trigger automation on failure.
- **Pipeline types:** V1, and **V2** (pipeline-level variables, triggers with Git filters, execution modes: queued, superseded, parallel).
- Troubleshooting: a failed action shows in the console. Check the **pipeline service role** permissions (e.g., to call CodeBuild, read S3, pass roles).
- **Cross-account deployments:** pipeline in a tools account, customer managed KMS key for artifacts, and cross-account roles.

### 6.5 Other developer tools
- **AWS CodeArtifact:** managed artifact repository for **npm, PyPI, Maven/Gradle, NuGet, Swift, RubyGems, Cargo, generic**. Uses **upstream repositories** and **external connections** (proxy and cache public repos like npmjs). Authenticate with `aws codeartifact login --tool npm|pip|twine` (token valid up to 12 h). Use **domains** to share across accounts. EventBridge events on package version changes.
- **Amazon CodeGuru:** Reviewer (automated code reviews, security detectors, secrets detection) and Profiler (runtime performance and cost recommendations for Java/Python apps).
- **Amazon Q Developer** (formerly CodeWhisperer): AI coding assistant and security scanning.
- **AWS CodeStar:** discontinued. **AWS Cloud9:** closed to new customers (2024).
- **AWS CodeCatalyst:** unified dev service (spaces, projects, workflows).

---

## 7. Containers: ECS, ECR, Fargate, and others

### 7.1 Amazon ECR
- Private and public Docker/OCI image registries. **Image scanning** (basic or enhanced with Inspector), lifecycle policies (expire untagged/old images), **tag immutability**, cross-Region and cross-account replication, KMS encryption.
- **Login:**
  ```bash
  aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <acct>.dkr.ecr.us-east-1.amazonaws.com
  docker build -t myapp . && docker tag myapp:latest <repo-uri>:latest && docker push <repo-uri>:latest
  ```
- Pull permission issues: check the IAM policy (`ecr:GetAuthorizationToken`, `ecr:BatchGetImage`, `ecr:GetDownloadUrlForLayer`) or the repository policy.

### 7.2 Amazon ECS
- **Cluster:** logical grouping. **Launch types / capacity:** **EC2** (you manage instances running the **ECS agent**), **Fargate** (serverless; no instances to manage; you specify CPU and memory per task), and ECS Managed Instances.
- **Task definition:** JSON blueprint containing images, CPU/memory, port mappings, env vars, **secrets**, log configuration (`awslogs` driver to CloudWatch), volumes, network mode (`awsvpc` required for Fargate; each task gets an ENI), and **IAM roles**:
  - **Task role:** permissions **your application code** uses (e.g., call S3 or DynamoDB).
  - **Task execution role:** permissions the **ECS agent** uses to **pull images from ECR, send logs to CloudWatch, and fetch secrets** from Secrets Manager or Parameter Store.
  - **EC2 instance profile** (EC2 launch type): the agent registers with the cluster.
- **Service:** keeps the desired count of tasks running, integrates with ALB/NLB, and supports **Service Auto Scaling** (target tracking on CPU, memory, or ALB request count).
- **Deployment types:** **rolling update** (`minimumHealthyPercent` / `maximumPercent`), **blue/green via CodeDeploy**, native ECS blue/green, and the external controller. **Deployment circuit breaker** auto-rolls back failed deployments.
- **Task placement (EC2 launch type only):**
  - Strategies: **`binpack`** (fill instances by CPU or memory to minimize cost), **`spread`** (across AZs or instances for HA), **`random`**.
  - Constraints: **`distinctInstance`** (one task per instance), **`memberOf`** (cluster query language expression, e.g., `attribute:ecs.instance-type =~ t3.*`).
- **Dynamic port mapping** with ALB on the EC2 launch type (host port 0) runs multiple copies of the same task per instance.
- **Data volumes:** EFS (shared, persistent, works with Fargate), bind mounts, Docker volumes (EC2). Fargate gets ephemeral storage (20 GB default, up to 200 GB).
- **ECS Exec:** shell into running containers (SSM).
- **Service Connect / Cloud Map:** service discovery.

### 7.3 Other container services
- **AWS Fargate:** serverless compute for ECS and EKS.
- **Amazon EKS:** managed Kubernetes; pods get IAM permissions via **IRSA** or **EKS Pod Identity**.
- **AWS App Runner:** fully managed service for web apps from source code or a container image. **Simplest** option: automatic build, deploy, scale, and HTTPS.
- **AWS Copilot CLI:** tool to build and deploy containerized apps on ECS/Fargate/App Runner.

---

## 8. Testing in development environments
- **API Gateway stages** (dev/test/prod) with **stage variables** pointing to Lambda **aliases**.
- **Lambda aliases + versions** for dev/test/prod in the same function.
- **Mock integrations** (API Gateway MOCK) for front-end development before the backend exists.
- **`sam local`** for local testing in Docker. **`sam local generate-event`** for test payloads. Console test events (shareable test events stored in EventBridge Schemas).
- **Separate accounts per environment** (AWS Organizations), separate CloudFormation stacks per environment (parameterized templates).
- **Integration tests in CodeBuild**, with test reports. Manual approval before prod.
- **Feature flags** with **AWS AppConfig** to decouple deploy from release.
- **Amazon Q Developer / CodeGuru** for code review and test generation.

---

## 9. Deployment exam traps
- "**Zero downtime** and **fastest rollback** on Elastic Beanstalk": **Blue/Green (swap URLs)**, or **Immutable** if the question asks for a deployment policy.
- "Beanstalk deploy with **full capacity** and minimal cost": **Rolling with additional batch**.
- "Beanstalk: canary-style evaluation of a new version": **Traffic splitting**.
- "Run a DB migration once per deployment on Beanstalk": **`container_commands` with `leader_only: true`**.
- "Gradually shift Lambda traffic with auto rollback": **SAM `AutoPublishAlias` + `DeploymentPreference` + CloudWatch alarms** (CodeDeploy).
- "CodeDeploy hook order": `ApplicationStop → BeforeInstall → AfterInstall → ApplicationStart → ValidateService`.
- "CodeDeploy can't deploy to EC2": **the agent isn't installed or running**, or the instance profile can't read S3.
- "CloudFormation doesn't update Lambda code": **change the S3 key or object version** (or use `package`).
- "Reuse VPC outputs across stacks": **Outputs Export + `!ImportValue`**.
- "Deploy the same stack to many accounts and Regions": **StackSets**.
- "Preview changes before update": **change sets**.
- "Keep RDS data when the stack is deleted": **`DeletionPolicy: Snapshot`** (or `Retain`).
- "Custom provisioning logic in CloudFormation": **Lambda-backed custom resource**.
- "Store build secrets securely in CodeBuild": **`parameter-store` / `secrets-manager` in `buildspec.yml`**, not plaintext `variables`.
- "CodeBuild needs to reach a private RDS": configure the **VPC** in the project.
- "Speed up CodeBuild dependency downloads": **cache** (S3 or local).
- "Manual approval before prod": **CodePipeline manual approval action + SNS**.
- "ECS container can't pull from ECR or write logs": **task execution role**. "App in the container can't access DynamoDB": **task role**.
- "Pack tasks to minimize EC2 instances": **binpack**. "Max availability": **spread** across AZs.
