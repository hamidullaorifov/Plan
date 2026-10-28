# DVA-C02 Practice Questions (with explanations)

Answer each question before opening the explanation. Write down **why each wrong option is wrong**; that habit improves exam scores the most.

---

### Q1 (Development – DynamoDB capacity)
An application reads 20 items per second from DynamoDB. Each item is 9 KB, and reads must be strongly consistent. How many RCUs are required?
- A. 40
- B. 60
- C. 30
- D. 45

<details><summary>Answer</summary>

**B. 60.** 9 KB rounds up to 12 KB = 3 × 4 KB units. 3 RCU × 20 reads = 60. With eventually consistent reads it would be 30.
</details>

---

### Q2 (Development – DynamoDB capacity)
An application writes 12 items per second, each 1.5 KB, using `TransactWriteItems`. How many WCUs are needed?
- A. 12
- B. 24
- C. 36
- D. 48

<details><summary>Answer</summary>

**D. 48.** 1.5 KB rounds up to 2 KB, so 2 WCU per write × 12 = 24. Transactions double it to **48**.
</details>

---

### Q3 (Development – Lambda + SQS)
A Lambda function processes messages from an SQS standard queue. Function timeout is 2 minutes. Some messages are processed twice. What should the developer do **first**?
- A. Switch to an SQS FIFO queue
- B. Set the queue visibility timeout to at least 12 minutes
- C. Increase the Lambda reserved concurrency
- D. Enable long polling on the queue

<details><summary>Answer</summary>

**B.** AWS recommends a visibility timeout of at least **6× the function timeout**. If the visibility timeout is shorter than processing time, messages become visible again and are reprocessed. FIFO (A) adds complexity and lower throughput; you should still make processing idempotent.
</details>

---

### Q4 (Development – API Gateway)
A REST API uses Lambda proxy integration. Clients receive **502 Bad Gateway**, while the Lambda logs show the function completed successfully. What is the most likely cause?
- A. The API Gateway integration timeout was exceeded
- B. The Lambda function returned a response in an incorrect format
- C. The client is being throttled
- D. The Lambda function lacks an execution role

<details><summary>Answer</summary>

**B.** With proxy integration, Lambda must return `{ "statusCode": ..., "headers": ..., "body": "string" }`. A malformed response causes a 502. A timeout would be 504 (A); throttling is 429 (C).
</details>

---

### Q5 (Security – Envelope encryption)
A developer needs to encrypt 50 MB files using a KMS customer managed key. What is the correct approach?
- A. Call `kms:Encrypt` with the file
- B. Call `kms:GenerateDataKey`, encrypt the file locally with the plaintext key, store the encrypted data key with the file
- C. Split the file into 4 KB chunks and call `kms:Encrypt` on each
- D. Use `kms:ReEncrypt`

<details><summary>Answer</summary>

**B.** This is envelope encryption. `Encrypt` supports only 4 KB. Chunking (C) is inefficient and would exhaust KMS quotas.
</details>

---

### Q6 (Security – Cognito)
A mobile app must let users sign in with Google and then upload photos directly to their own folder in an S3 bucket. Which solution requires the LEAST custom code?
- A. Cognito user pool with a Lambda that generates presigned URLs
- B. Cognito identity pool federated with Google, and an IAM role policy using `${cognito-identity.amazonaws.com:sub}` in the S3 resource ARN
- C. IAM users for each mobile user
- D. Store an IAM access key in the mobile app

<details><summary>Answer</summary>

**B.** Identity pools exchange the Google identity for temporary AWS credentials. The policy variable restricts each user to their own prefix. Options C and D are insecure or don't scale.
</details>

---

### Q7 (Deployment – Elastic Beanstalk)
A company wants to deploy a new version to a production Elastic Beanstalk environment with **no reduction in capacity**, **minimal additional cost**, and does not need instant rollback. Which deployment policy fits?
- A. All at once
- B. Rolling
- C. Rolling with additional batch
- D. Immutable

<details><summary>Answer</summary>

**C.** It maintains full capacity by launching an extra batch first, at a small extra cost. Rolling (B) reduces capacity. Immutable (D) doubles instances temporarily, which costs more, though it gives faster rollback.
</details>

---

### Q8 (Deployment – CodeDeploy)
Which is the correct order of CodeDeploy lifecycle hooks for an EC2 in-place deployment?
- A. BeforeInstall → ApplicationStop → AfterInstall → ApplicationStart → ValidateService
- B. ApplicationStop → BeforeInstall → AfterInstall → ApplicationStart → ValidateService
- C. ApplicationStop → AfterInstall → BeforeInstall → ValidateService → ApplicationStart
- D. ValidateService → ApplicationStop → BeforeInstall → AfterInstall → ApplicationStart

<details><summary>Answer</summary>

**B.** The DownloadBundle and Install steps (run by the agent) occur between ApplicationStop/BeforeInstall and BeforeInstall/AfterInstall respectively.
</details>

---

### Q9 (Deployment – SAM)
A team uses SAM to deploy Lambda functions. They want 10% of traffic sent to the new version for 5 minutes, then 100%, with automatic rollback if errors spike. What should they configure?
- A. API Gateway canary on the stage
- B. `AutoPublishAlias` and `DeploymentPreference: Type: Canary10Percent5Minutes` with CloudWatch `Alarms`
- C. Two Lambda functions behind Route 53 weighted records
- D. Lambda reserved concurrency of 10%

<details><summary>Answer</summary>

**B.** SAM uses CodeDeploy to shift alias traffic between versions and rolls back on alarm.
</details>

---

### Q10 (Troubleshooting – X-Ray)
A developer wants to search X-Ray traces for requests from a specific `customerId`. How should the value be recorded?
- A. As metadata
- B. As an annotation
- C. As a subsegment name
- D. In a CloudWatch log

<details><summary>Answer</summary>

**B.** Annotations are indexed and usable in filter expressions. Metadata is not indexed.
</details>

---

### Q11 (Troubleshooting – CloudWatch)
An operations team needs to monitor **memory utilization** of EC2 instances and alarm when it exceeds 80%. What should the developer do?
- A. Enable detailed monitoring
- B. Install the CloudWatch agent to publish memory metrics, then create an alarm
- C. Use the `MemoryUtilization` default metric in the AWS/EC2 namespace
- D. Enable CloudTrail data events

<details><summary>Answer</summary>

**B.** Memory isn't a default EC2 metric because the hypervisor can't see it. Detailed monitoring (A) only changes frequency.
</details>

---

### Q12 (Development – Lambda concurrency)
A Lambda function writes to a legacy database that can handle a maximum of 30 concurrent connections. During spikes, the database is overwhelmed. What is the simplest fix?
- A. Set reserved concurrency for the function to 30
- B. Enable provisioned concurrency of 30
- C. Increase the function memory
- D. Increase the function timeout

<details><summary>Answer</summary>

**A.** Reserved concurrency caps concurrent executions. Provisioned concurrency (B) pre-warms environments but doesn't cap them. Adding SQS as a buffer in front would also help if invocations are async.
</details>

---

### Q13 (Security – IAM)
A developer receives an `AccessDenied` error when creating a Lambda function and assigning an existing execution role, although they have `lambda:CreateFunction`. What permission is missing?
- A. `lambda:InvokeFunction`
- B. `iam:CreateRole`
- C. `iam:PassRole`
- D. `sts:AssumeRole`

<details><summary>Answer</summary>

**C.** Passing a role to a service requires `iam:PassRole`.
</details>

---

### Q14 (Development – S3)
Users must upload files up to 2 GB from a browser directly to a private S3 bucket without exposing AWS credentials. What is the best approach?
- A. Make the bucket public for writes
- B. Generate presigned URLs (or presigned multipart upload URLs) from a backend
- C. Embed IAM user keys in JavaScript
- D. Use S3 Transfer Acceleration only

<details><summary>Answer</summary>

**B.** Presigned URLs grant temporary, scoped access. For large files, presign each multipart part. Transfer Acceleration (D) improves speed but doesn't solve authorization.
</details>

---

### Q15 (Development – Caching)
An application reads product data from RDS. Data changes rarely, and occasional staleness of up to 10 minutes is acceptable. The team wants to reduce DB load with minimal memory waste. Which strategy?
- A. Write-through caching without TTL
- B. Lazy loading with a TTL of 10 minutes
- C. DAX
- D. RDS Multi-AZ

<details><summary>Answer</summary>

**B.** Lazy loading caches only requested data, and the TTL bounds staleness. DAX (C) is only for DynamoDB. Multi-AZ (D) provides HA, not performance.
</details>

---

### Q16 (Deployment – CloudFormation)
A developer updated the Lambda zip file in S3 under the same key and ran a stack update, but the function code didn't change. Why?
- A. CloudFormation doesn't support Lambda
- B. The template didn't change, so CloudFormation detected no update. Change the S3 key or `S3ObjectVersion`.
- C. The function needs to be deleted first
- D. The stack needs `CAPABILITY_IAM`

<details><summary>Answer</summary>

**B.** CloudFormation compares template properties. Use versioned keys, `S3ObjectVersion`, or `aws cloudformation package` / `sam deploy`.
</details>

---

### Q17 (Security – Secrets)
A Lambda function needs a database password that must be **rotated automatically every 30 days**. Where should it be stored?
- A. Lambda environment variable
- B. SSM Parameter Store `String`
- C. AWS Secrets Manager with rotation enabled
- D. S3 object encrypted with SSE-S3

<details><summary>Answer</summary>

**C.** Secrets Manager provides native rotation.
</details>

---

### Q18 (Troubleshooting – Kinesis)
A Kinesis Data Streams producer receives `ProvisionedThroughputExceededException` errors although total throughput is well below the stream's aggregate capacity. What is the most likely cause?
- A. Retention period too short
- B. Poorly distributed partition key causing a hot shard
- C. Too many consumers
- D. Enhanced fan-out is disabled

<details><summary>Answer</summary>

**B.** Use a higher-cardinality partition key, and add retries with backoff.
</details>

---

### Q19 (Development – Step Functions)
A workflow must pause until a human approves an order via email, which can take several days. Which feature should be used?
- A. Express workflow with a Wait state
- B. Standard workflow with a Task using `.waitForTaskToken`
- C. Lambda with a 15-minute sleep loop
- D. SQS delay queue

<details><summary>Answer</summary>

**B.** The callback pattern. The approval handler calls `SendTaskSuccess` with the token. Express workflows (A) run a maximum of 5 minutes.
</details>

---

### Q20 (Deployment – ECS)
An ECS task on Fargate fails to start with an error pulling the image from a private ECR repository. Which role needs ECR permissions?
- A. Task role
- B. Task execution role
- C. EC2 instance profile
- D. CodeDeploy service role

<details><summary>Answer</summary>

**B.** The execution role is used by the ECS agent and Fargate to pull images, push logs, and fetch secrets. In a private subnet, also check for a NAT gateway or ECR VPC endpoints.
</details>

---

### Q21 (Development – DynamoDB)
A developer must prevent two users from overwriting each other's changes to the same DynamoDB item. What should be implemented?
- A. Strongly consistent reads
- B. Optimistic locking with a version attribute and conditional writes
- C. DAX
- D. BatchWriteItem

<details><summary>Answer</summary>

**B.** Read the version, then update with `ConditionExpression: version = :expected` and increment it. On `ConditionalCheckFailedException`, re-read and retry.
</details>

---

### Q22 (Troubleshooting – Lambda performance)
A CPU-intensive Lambda function configured with 512 MB takes 8 seconds per run. What is the most effective way to reduce duration?
- A. Increase the timeout
- B. Increase the memory allocation
- C. Add a Lambda layer
- D. Enable provisioned concurrency

<details><summary>Answer</summary>

**B.** CPU is allocated proportionally to memory. Provisioned concurrency (D) only removes cold starts.
</details>

---

### Q23 (Development – Messaging)
An order service must send each order event to inventory, billing, and shipping services, each processing independently and durably even if one is down. Which design?
- A. One SQS queue polled by all three services
- B. SNS topic with three SQS queue subscriptions
- C. Kinesis Data Firehose
- D. Direct synchronous HTTP calls

<details><summary>Answer</summary>

**B.** The fan-out pattern. With a single queue (A), each message goes to only one consumer.
</details>

---

### Q24 (Security – S3 encryption)
A security policy requires that all objects uploaded to a bucket are encrypted with SSE-KMS. Uploads without it must be rejected. What should the developer do?
- A. Enable default encryption only
- B. Add a bucket policy that denies `s3:PutObject` if `s3:x-amz-server-side-encryption` is not `aws:kms`
- C. Enable versioning
- D. Use an ACL

<details><summary>Answer</summary>

**B.** Default encryption (A) encrypts requests that don't specify encryption, but doesn't **reject** them. The explicit deny in the bucket policy enforces it.
</details>

---

### Q25 (Troubleshooting – API Gateway)
An API Gateway endpoint integrated with Lambda returns **504** errors for some requests that run complex reports. What should the developer do?
- A. Increase Lambda memory to 10 GB only
- B. Change to an asynchronous pattern: API Gateway puts a job on SQS (or starts Step Functions) and returns 202, and the client polls for results
- C. Increase API Gateway throttling limits
- D. Enable API Gateway caching

<details><summary>Answer</summary>

**B.** 504 means the integration exceeded the 29 s timeout. Long-running work should be asynchronous. More memory (A) might help partially but doesn't guarantee completion within 29 s.
</details>

---

### Q26 (Deployment – CodeBuild)
A CodeBuild project needs a third-party API key during the build. What is the most secure way to provide it?
- A. Hard-code it in `buildspec.yml` under `env.variables`
- B. Store it in Secrets Manager and reference it under `env.secrets-manager` in `buildspec.yml`
- C. Commit it in a `.env` file
- D. Pass it as a plaintext environment variable in the project

<details><summary>Answer</summary>

**B.** The CodeBuild service role needs `secretsmanager:GetSecretValue`.
</details>

---

### Q27 (Development – Lambda async)
A Lambda function invoked asynchronously by S3 sometimes fails. The team wants failed events captured along with details, and also successful results sent to another function. What should they use?
- A. DLQ only
- B. Lambda destinations (on-failure: SQS, on-success: Lambda)
- C. Increase retries to 10
- D. SQS redrive policy

<details><summary>Answer</summary>

**B.** Destinations support both success and failure. A DLQ (A) handles only failures. Async retries max out at 2 (C).
</details>

---

### Q28 (Troubleshooting – Logs)
A team wants an alarm whenever the string `PaymentFailed` appears more than 5 times in 1 minute in application logs in CloudWatch Logs. What should they create?
- A. CloudTrail trail
- B. Metric filter on the log group plus a CloudWatch alarm on the resulting metric
- C. Logs Insights saved query
- D. X-Ray group

<details><summary>Answer</summary>

**B.**
</details>

---

### Q29 (Development – DynamoDB Streams)
When a new user record is inserted into DynamoDB, a welcome email must be sent. Which approach requires the least code and infrastructure?
- A. Scan the table every minute
- B. Enable DynamoDB Streams and trigger a Lambda function that sends email via SES
- C. Use DAX
- D. Use Kinesis Firehose

<details><summary>Answer</summary>

**B.**
</details>

---

### Q30 (Security – STS/MFA)
An IAM policy requires MFA for deleting objects (`aws:MultiFactorAuthPresent`). A developer using the CLI with access keys gets AccessDenied. What should they do?
- A. Call `sts:GetSessionToken` with the MFA serial number and token code, and use the temporary credentials
- B. Call `sts:AssumeRoleWithWebIdentity`
- C. Rotate the access keys
- D. Use the root account

<details><summary>Answer</summary>

**A.**
</details>

---

## Scoring guide
- **27–30:** ready, so take full-length timed practice exams.
- **21–26:** review the domain files for the questions you missed.
- **< 21:** go through the 8-week plan more slowly and do the labs.

## Next steps
- Official AWS Skill Builder practice question set (free) and the official practice exam.
- 3–5 full-length timed third-party practice exams (target ≥ 80%).
