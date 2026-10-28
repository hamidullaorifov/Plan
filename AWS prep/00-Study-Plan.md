# AWS Certified Developer – Associate (DVA-C02) – Preparation Plan

> Check the details on the official exam page before you book: https://aws.amazon.com/certification/certified-developer-associate/
> AWS changes exam versions, prices and question formats from time to time.

## 1. Exam facts

| Item | Detail |
|---|---|
| Exam code | DVA-C02 |
| Questions | 65 (50 scored + 15 unscored, and you can't tell which are which) |
| Question types | Multiple choice (1 correct) and multiple response (2+ correct) |
| Duration | 130 minutes (non-native English speakers can request +30 min "ESL accommodation" **before** booking) |
| Passing score | 720 / 1000 (scaled, compensatory: you don't need to pass each domain separately) |
| Cost | 150 USD (associate tier) |
| Delivery | Pearson VUE test center or online proctored |
| Validity | 3 years |
| Wrong answers | Not penalized, so **never leave a question blank** |

### Domain weighting

| # | Domain | Weight | Study file |
|---|---|---|---|
| 1 | Development with AWS Services | 32% | [01-Development-with-AWS-Services.md](01-Development-with-AWS-Services.md) |
| 2 | Security | 26% | [02-Security.md](02-Security.md) |
| 3 | Deployment | 24% | [03-Deployment.md](03-Deployment.md) |
| 4 | Troubleshooting and Optimization | 18% | [04-Troubleshooting-and-Optimization.md](04-Troubleshooting-and-Optimization.md) |

**New to AWS? Start with the beginner guide.** It explains each service in plain language with analogies, diagrams and examples:
1. [Part 1: Cloud basics, networking, compute, storage, databases](00-Beginner-Guide/01-Cloud-Basics-Networking-Compute-Storage-Databases.md)
2. [Part 2: Integration, APIs, security](00-Beginner-Guide/02-Integration-APIs-Security.md)
3. [Part 3: Dev tools, deployment, monitoring, example architectures, glossary](00-Beginner-Guide/03-DevTools-Monitoring-Architecture.md)

Supporting material:
- [05-Cheat-Sheet.md](05-Cheat-Sheet.md): numbers, limits, keyword-to-service mapping, common traps
- [06-Practice-Questions.md](06-Practice-Questions.md): scenario questions with explanations

## 2. Service priority (by how often each appears on the exam)

**Tier 1 (expect several questions on each, so know them deeply):**
Lambda, DynamoDB, API Gateway, S3, IAM, Cognito, KMS, SQS, SNS, CloudFormation, SAM, CodeDeploy, CodeBuild, CodePipeline, Elastic Beanstalk, CloudWatch (Metrics/Logs/Alarms), X-Ray

**Tier 2 (expect 1–3 questions each):**
Kinesis Data Streams / Firehose, Step Functions, EventBridge, ElastiCache, Secrets Manager, SSM Parameter Store, ECS/ECR/Fargate, CDK, STS, CloudTrail, CloudFront, AppSync, Amplify, RDS Proxy

**Tier 3 (recognize what each one does):**
CodeArtifact, CodeGuru, AppConfig, SES, OpenSearch, Aurora Serverless, App Runner, EKS, Route 53, VPC basics, EC2 basics (user data, metadata, instance profiles), AWS Copilot CLI

## 3. Eight-week study schedule (about 1–2 hours/day)

Shorten to 4–5 weeks if you already use AWS daily. **If you are new to AWS, add Week 0 below, which makes it a 10-week plan.** Follow each reading block with hands-on work in your own account (Free Tier) and set a **billing alarm/budget first**.

### Week 0 (1–2 weeks, beginners only): Understand the services
- [ ] Read the beginner guide [Part 1](00-Beginner-Guide/01-Cloud-Basics-Networking-Compute-Storage-Databases.md): cloud concepts, Regions/AZs, VPC, EC2, Lambda, containers, S3, RDS, DynamoDB, ElastiCache
- [ ] Read [Part 2](00-Beginner-Guide/02-Integration-APIs-Security.md): API Gateway, SQS, SNS, EventBridge, Kinesis, Step Functions, IAM, Cognito, KMS, Secrets Manager
- [ ] Read [Part 3](00-Beginner-Guide/03-DevTools-Monitoring-Architecture.md): CI/CD, CloudFormation, SAM, CDK, CodeBuild/Deploy/Pipeline, CloudWatch, X-Ray, CloudTrail
- [ ] Do the 10 "first hands-on steps" at the end of Part 3
- [ ] Optional: the free AWS Skill Builder course "AWS Cloud Practitioner Essentials" for extra fundamentals

### Week 1: Foundations and IAM
- [ ] Create an AWS account, enable MFA on root, create an admin IAM Identity Center user, and set an AWS Budget
- [ ] IAM: users, groups, roles, policies (identity vs resource-based), policy evaluation logic, trust policies
- [ ] STS: AssumeRole, GetSessionToken (MFA), cross-account access
- [ ] AWS CLI and SDK: profiles, credential provider chain, `--dry-run`, `--query`, pagination, `aws sts get-caller-identity`
- [ ] EC2 basics: instance profile, user data, IMDSv2, security groups
- **Lab:** Create a role that EC2 assumes to read S3 without access keys.

### Week 2: Storage and databases
- [ ] S3: storage classes, versioning, lifecycle, encryption (SSE-S3/SSE-KMS/DSSE-KMS/SSE-C/client-side), presigned URLs, CORS, event notifications, multipart upload, Transfer Acceleration, performance per prefix
- [ ] DynamoDB, part 1: keys, RCU/WCU calculation, on-demand vs provisioned, LSI vs GSI, Query vs Scan
- [ ] DynamoDB, part 2: Streams, TTL, DAX, transactions, batch ops, conditional writes, optimistic locking, PartiQL, expressions
- **Lab:** Create a table with a GSI; write items with the SDK; run Query vs Scan; enable Streams and process them with Lambda.

### Week 3: Serverless compute
- [ ] Lambda: invocation models (sync/async/event source mapping), concurrency (reserved/provisioned), versions and aliases, layers, env vars, destinations, DLQ, VPC access, /tmp, container images, limits, cold starts, SnapStart
- [ ] API Gateway: REST vs HTTP vs WebSocket, integration types (proxy/non-proxy), mapping templates, stages and stage variables, canary, caching, throttling, usage plans/API keys, authorizers, CORS, error codes
- **Lab:** Build API Gateway → Lambda → DynamoDB CRUD. Add a Cognito authorizer later in week 5.

### Week 4: Integration and messaging
- [ ] SQS: standard vs FIFO, visibility timeout, long polling, DLQ, delay queues, extended client
- [ ] SNS: topics, fan-out to SQS, filter policies, FIFO topics
- [ ] EventBridge: buses, rules, patterns, Scheduler, archive/replay
- [ ] Kinesis Data Streams (shards, partition keys, KCL) vs Firehose
- [ ] Step Functions: Standard vs Express, states, Retry/Catch, callback pattern
- [ ] ElastiCache: Redis/Valkey vs Memcached, lazy loading vs write-through
- **Lab:** SNS → 2 SQS queues fan-out; Lambda consumer with a DLQ; Step Functions workflow with Retry/Catch.

### Week 5: Security deep dive
- [ ] Cognito: user pools vs identity pools, JWT tokens, hosted UI, Lambda triggers
- [ ] KMS: envelope encryption, `GenerateDataKey`, key policies, grants, quotas, rotation, multi-region keys
- [ ] Secrets Manager vs Parameter Store
- [ ] Encryption in transit and at rest across services; ACM certificates
- **Lab:** Encrypt or decrypt data with KMS from the CLI; store a DB secret with rotation; read it from Lambda.

### Week 6: Deployment and CI/CD
- [ ] CloudFormation: template anatomy, intrinsic functions, change sets, nested stacks, cross-stack exports, StackSets, DeletionPolicy, custom resources, drift
- [ ] SAM: template, CLI commands, `AutoPublishAlias`, `DeploymentPreference`, policy templates, `sam local`
- [ ] CDK: constructs (L1/L2/L3), bootstrap, synth, deploy
- [ ] Elastic Beanstalk: environments, deployment policies, `.ebextensions`, worker tier
- [ ] CodeBuild (`buildspec.yml`), CodeDeploy (`appspec.yml` + hooks), CodePipeline, CodeArtifact
- [ ] ECS/ECR/Fargate: task definitions, task role vs execution role
- **Lab:** Deploy a SAM app with canary traffic shifting; build a CodePipeline (GitHub → CodeBuild → CloudFormation/SAM deploy).

### Week 7: Monitoring, troubleshooting and optimization
- [ ] CloudWatch: metrics, custom/high-resolution metrics, EMF, alarms, Logs, metric filters, Logs Insights, subscription filters, agent
- [ ] X-Ray: SDK and daemon, segments, subsegments, annotations vs metadata, sampling, Lambda/ECS/Beanstalk integration; OpenTelemetry (ADOT)
- [ ] CloudTrail; reading error codes (4xx/5xx); throttling and exponential backoff
- [ ] Optimization: caching layers, Lambda memory tuning, DynamoDB key design, S3 performance
- **Lab:** Instrument a Lambda with X-Ray, add annotations, filter traces; create a metric filter and an alarm on log errors.

### Week 8: Exam simulation
- [ ] Take the **AWS official practice question set** (free on AWS Skill Builder)
- [ ] Take 3–5 full-length practice exams (timed, 130 min). Target **≥ 80%** consistently before booking.
- [ ] For every wrong **and every guessed** question, write down why each wrong option is wrong
- [ ] Re-read [05-Cheat-Sheet.md](05-Cheat-Sheet.md) and the FAQs for Lambda, DynamoDB, SQS and API Gateway
- [ ] Book the exam and do a systems check if taking it online

## 4. Recommended resources

**Official (free or low cost)**
- Exam guide PDF (on the official exam page): read the **in-scope / out-of-scope services list**
- AWS Skill Builder: "Exam Prep Standard Course: AWS Certified Developer – Associate", the official practice question set, and the official pretest
- AWS service FAQs (very high yield): Lambda, DynamoDB, SQS, SNS, API Gateway, S3, KMS, Cognito, Elastic Beanstalk, CodeDeploy
- Developer guides: "Best practices for working with AWS Lambda functions", "Best practices for designing and architecting with DynamoDB"
- Whitepapers: *Practicing Continuous Integration and Continuous Delivery on AWS*, *Serverless Applications Lens (Well-Architected)*, *Overview of Deployment Options on AWS*

**Popular third-party courses and tests**
- Video courses: Stéphane Maarek (Udemy), Adrian Cantrill, Neal Davis (Digital Cloud Training)
- Practice exams: Tutorials Dojo (Jon Bonso) is widely regarded as closest to the real exam's difficulty; Maarek/Stephane practice exams

## 5. Exam-day strategy

1. **Read the last sentence first.** It tells you what is being asked: "MOST cost-effective", "LEAST operational overhead", "MOST secure", "with the FEWEST code changes".
2. **Underline the constraints** (serverless, no code change, real-time, ordering, exactly-once, < 1 ms latency, 15+ minutes).
3. **Eliminate first.** Usually 2 options are obviously wrong, such as a service that doesn't exist, the wrong service category, or something that violates a constraint.
4. **Prefer managed/serverless and native features** over custom code when the question says "least operational overhead".
5. **Never store credentials** in code or env vars in plain text. An answer that uses IAM roles, Secrets Manager or Parameter Store is almost always better.
6. **Flag and move on.** Aim for ~1.5 min/question on the first pass (about 100 min), then review flagged questions.
7. Answer every question; there is no penalty for guessing.

## 6. Progress tracker

| Week | Topics done | Labs done | Practice score |
|---|---|---|---|
| 1 | | | |
| 2 | | | |
| 3 | | | |
| 4 | | | |
| 5 | | | |
| 6 | | | |
| 7 | | | |
| 8 | | | |
