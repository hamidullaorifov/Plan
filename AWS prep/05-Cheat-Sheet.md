# DVA-C02 Cheat Sheet – Numbers, Keywords, Traps

Review this the day before the exam.

## 1. Numbers to memorize

### Lambda
| Item | Value |
|---|---|
| Memory | 128 MB – 10,240 MB (CPU scales with memory; ~1 vCPU at 1,769 MB) |
| Timeout | default 3 s, **max 900 s (15 min)** |
| /tmp | 512 MB – 10,240 MB |
| Env vars | **4 KB** total |
| Deployment package | 50 MB zipped (direct), **250 MB unzipped** (incl. layers), container image **10 GB** |
| Layers | **5** per function |
| Payload | **6 MB** sync; async 256 KB (raised to 1 MB in 2025) |
| Default concurrency | **1,000** per Region (soft) |
| Async retries | **2** (0–2), event max age up to 6 h |
| SQS trigger | visibility timeout **≥ 6× function timeout** |

### DynamoDB
| Item | Value |
|---|---|
| Item size | **400 KB** |
| 1 RCU | 1 strongly consistent read/s **≤ 4 KB** (or 2 eventually consistent) |
| 1 WCU | 1 write/s **≤ 1 KB** |
| Transactions | **2×** capacity; up to **100** items |
| BatchWriteItem | **25** items / 16 MB |
| BatchGetItem | **100** items / 16 MB |
| Query/Scan page | **1 MB** |
| LSI | **5**, created at table creation only, 10 GB per partition key value |
| GSI | **20** (default), any time, eventual consistency only |
| Streams retention | **24 h** |
| DAX default TTL | 5 min |
| PITR | 35 days |

### S3
| Item | Value |
|---|---|
| Max object | **5 TB** |
| Single PUT | **5 GB** |
| Multipart | recommended **> 100 MB**, required **> 5 GB**; up to 10,000 parts |
| Request rate | **3,500** PUT/COPY/POST/DELETE, **5,500** GET/HEAD per second **per prefix** |
| Presigned URL | CLI default 3,600 s; max 7 days (IAM user) |
| Consistency | Strong read-after-write |

### SQS / SNS / Kinesis
| Item | Value |
|---|---|
| SQS message size | **256 KB** (1 MiB since 2025); Extended Client up to 2 GB via S3 |
| SQS retention | 1 min – **14 days**, default **4 days** |
| Visibility timeout | default **30 s**, max **12 h** |
| Long polling | up to **20 s** |
| Delay queue | up to **15 min** |
| ReceiveMessage | up to **10** messages |
| FIFO throughput | **300 msg/s**, **3,000** with batching (more in high-throughput mode) |
| FIFO dedup window | **5 min** |
| Kinesis shard in | **1 MB/s or 1,000 records/s** |
| Kinesis shard out | **2 MB/s** shared; 2 MB/s **per consumer** with enhanced fan-out |
| Kinesis retention | 24 h default – 365 days |

### API Gateway
| Item | Value |
|---|---|
| Integration timeout | **29 s** (can be raised for Regional/private REST APIs) |
| Account throttle | **10,000 rps**, burst **5,000** |
| Cache TTL | default **300 s**, max **3,600 s** |
| Cache size | 0.5 GB – 237 GB |
| Lambda authorizer cache | default 300 s, max 3,600 s |

### Other
| Item | Value |
|---|---|
| KMS Encrypt API | **4 KB** max data (use envelope encryption beyond) |
| KMS key deletion wait | 7–30 days |
| STS AssumeRole duration | 15 min – 12 h (default 1 h) |
| Parameter Store | Standard **4 KB** (10,000 free), Advanced **8 KB** |
| Secrets Manager | 64 KB |
| Step Functions | Standard **1 year**, Express **5 min** |
| CloudWatch high resolution | **1 s**; alarms 10 s / 30 s |
| EC2 monitoring | basic 5 min, detailed 1 min |
| CloudWatch dimensions | 30 per metric |
| CloudTrail event history | **90 days** |
| X-Ray daemon | **UDP 2000** |
| X-Ray default sampling | **1 req/s + 5%** |
| Beanstalk app versions | **1,000** |
| Cognito refresh token | default 30 days |

---

## 2. Keyword → answer mapping

| If the question says… | Think… |
|---|---|
| "decouple", "buffer", "absorb spikes" | **SQS** |
| "fan-out", "multiple subscribers" | **SNS → SQS** |
| "order", "exactly once", "no duplicates" | **SQS FIFO** (MessageGroupId, dedup ID) |
| "real-time streaming", "replay", "ordered per key", "clickstream" | **Kinesis Data Streams** |
| "load streaming data into S3/Redshift/OpenSearch", "near real-time", "no code" | **Data Firehose** |
| "event bus", "SaaS events", "schedule", "route events by content" | **EventBridge** |
| "orchestrate", "workflow", "retry/catch", "human approval" | **Step Functions** (`.waitForTaskToken`) |
| "longer than 15 minutes" | **Not Lambda** → ECS/Fargate, Step Functions, Batch |
| "microsecond latency DynamoDB" | **DAX** |
| "cache DB query results / sessions", "leaderboard" | **ElastiCache (Redis/Valkey)** |
| "user sign-up / sign-in", "JWT" | **Cognito user pools** |
| "temporary AWS credentials for app users", "guest access" | **Cognito identity pools** |
| "rotate DB credentials" | **Secrets Manager** |
| "store config hierarchically", "cheapest" | **SSM Parameter Store** |
| "encrypt > 4 KB with KMS" | **Envelope encryption** (`GenerateDataKey`) |
| "GraphQL", "real-time subscriptions", "offline sync" | **AppSync** |
| "trace requests across microservices", "latency bottleneck" | **X-Ray** |
| "audit API calls", "who did what" | **CloudTrail** |
| "memory metric on EC2" | **CloudWatch agent** |
| "deploy serverless app", "local testing" | **SAM** (`sam local`) |
| "IaC in a programming language" | **CDK** |
| "multi-account / multi-Region stacks" | **CloudFormation StackSets** |
| "preview infrastructure changes" | **Change sets** |
| "shift Lambda traffic gradually with rollback" | **CodeDeploy via SAM `DeploymentPreference`** |
| "upload directly from browser to private S3" | **Presigned URL** |
| "browser calls API on another domain" | **CORS** |
| "per-client rate limiting and quotas" | **API Gateway usage plans + API keys** |
| "custom auth with third-party tokens" | **Lambda authorizer** |
| "Lambda needs many DB connections" | **RDS Proxy** |
| "Lambda in VPC needs internet" | **NAT Gateway** (or VPC endpoints for AWS services) |
| "process every message once despite retries" | **Idempotency** (DynamoDB conditional write with a request ID) |
| "least operational overhead" | **Managed/serverless native feature** |
| "simplest way to run a container web app" | **App Runner** (or Elastic Beanstalk) |
| "store npm/pip/maven packages privately" | **CodeArtifact** |
| "feature flags / dynamic config without redeploy" | **AWS AppConfig** |

---

## 3. Classic traps
1. **FilterExpression doesn't reduce RCU** consumed; only key conditions do.
2. **GSIs never support strongly consistent reads.** **LSIs must be created with the table.**
3. **Lambda aliases can't point to aliases.** **Provisioned concurrency can't be set on `$LATEST`.**
4. **Reserved concurrency = 0** throttles all invocations (kill switch).
5. **The SQS DLQ for a Lambda event source mapping is set on the queue**, not on the function. A **Lambda DLQ / destinations only apply to async invocations.**
6. **Kinesis: one KCL worker per shard max**; more workers than shards = idle workers.
7. **API Gateway changes require redeploying to a stage.**
8. **Lambda proxy integration: the function must return CORS headers and the `{statusCode, body}` format.**
9. **SSE-C requires HTTPS**; keys aren't stored by AWS.
10. **KMS key policy must allow the account** for IAM policies to work.
11. **`iam:PassRole`** is required to assign roles to services.
12. **CloudFormation won't redeploy Lambda code from the same S3 key** unless the key or version changes.
13. **CodeDeploy rollback = a new deployment** of the last good revision.
14. **`.ebextensions` files must end in `.config`** and live at the bundle root.
15. **`buildspec.yml` and `appspec.yml` go in the root** of the source/revision.
16. **ECS task execution role** (pull image, logs, secrets) vs **task role** (app permissions).
17. **X-Ray annotations are indexed; metadata is not.**
18. **CloudWatch doesn't collect EC2 memory by default.**
19. **Metric filters aren't retroactive.**
20. **Multi-AZ RDS = HA, not read scaling.** Read replicas = read scaling.
21. **Cognito user pools = authN; identity pools = AWS credentials.**
22. **Don't retry 4xx errors (except 429/throttling).**
23. **API keys are not an authorization mechanism.**
24. **CloudFront / Cognito custom domain ACM certificates must be in us-east-1.**
25. **SQS standard can deliver duplicates and out of order**, so design for idempotency.

---

## 4. Useful CLI commands
```bash
aws sts get-caller-identity
aws sts decode-authorization-message --encoded-message <msg>
aws ec2 run-instances ... --dry-run
aws s3 presign s3://bucket/key --expires-in 600
aws s3 cp big.zip s3://bucket/ --storage-class STANDARD_IA
aws lambda invoke --function-name fn --payload file://e.json --cli-binary-format raw-in-base64-out out.json
aws lambda invoke --function-name fn --invocation-type Event ...
aws lambda publish-version --function-name fn
aws lambda create-alias --function-name fn --name prod --function-version 3 --routing-config AdditionalVersionWeights={"4"=0.1}
aws lambda put-function-concurrency --function-name fn --reserved-concurrent-executions 50
aws lambda add-permission --function-name fn --statement-id s3 --action lambda:InvokeFunction --principal s3.amazonaws.com --source-arn arn:aws:s3:::bucket
aws dynamodb query --table-name T --key-condition-expression "pk = :p" --expression-attribute-values '{":p":{"S":"USER#1"}}'
aws sqs receive-message --queue-url URL --wait-time-seconds 20 --max-number-of-messages 10
aws kms encrypt --key-id alias/my-key --plaintext fileb://secret.txt
aws kms generate-data-key --key-id alias/my-key --key-spec AES_256
aws ssm get-parameters-by-path --path /myapp/prod --with-decryption --recursive
aws secretsmanager get-secret-value --secret-id prod/db
aws cloudformation package --template-file t.yaml --s3-bucket b --output-template-file out.yaml
aws cloudformation deploy --template-file out.yaml --stack-name s --capabilities CAPABILITY_IAM
aws ecr get-login-password | docker login --username AWS --password-stdin <acct>.dkr.ecr.<region>.amazonaws.com
aws cloudwatch put-metric-data --namespace MyApp --metric-name Orders --value 1 --storage-resolution 1
aws cloudwatch set-alarm-state --alarm-name A --state-value ALARM --state-reason test
aws logs tail /aws/lambda/fn --follow
aws codeartifact login --tool npm --domain d --repository r
sam build && sam deploy --guided
sam local invoke Fn -e events/event.json
cdk bootstrap && cdk synth && cdk deploy
eb init && eb create && eb deploy && eb swap
```

---

## 5. Important headers and env vars
| Name | Context |
|---|---|
| `x-amz-server-side-encryption: AES256 \| aws:kms \| aws:kms:dsse` | S3 SSE |
| `x-amz-server-side-encryption-customer-key` | S3 SSE-C |
| `x-api-key` | API Gateway API key |
| `Cache-Control: max-age=0` | Bypass API Gateway cache |
| `X-Amzn-Trace-Id` | X-Ray trace propagation |
| `Authorization` | Cognito JWT / SigV4 |
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`, `AWS_REGION`, `AWS_PROFILE` | SDK credentials/config |
| `_X_AMZN_TRACE_ID`, `AWS_XRAY_DAEMON_ADDRESS`, `AWS_XRAY_CONTEXT_MISSING` | X-Ray |
| `AWS_LAMBDA_FUNCTION_NAME`, `AWS_LAMBDA_FUNCTION_VERSION`, `AWS_LAMBDA_LOG_GROUP_NAME` | Lambda reserved env vars |
| `CODEBUILD_RESOLVED_SOURCE_VERSION` | CodeBuild commit ID |
