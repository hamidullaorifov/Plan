# Domain 4 – Troubleshooting and Optimization (18%)

What this domain tests:
- Assisting in a root cause analysis (logs, metrics, traces, error codes, debugging service integrations)
- Instrumenting code for observability (structured logging, custom metrics, tracing, alerts)
- Optimizing applications by using AWS services and features (caching, concurrency, messaging, resource sizing)

---

## 1. Amazon CloudWatch

### 1.1 Metrics
- Metrics are grouped by **namespace** (e.g., `AWS/EC2`, `AWS/Lambda`, or a custom one like `MyApp`) and identified by **dimensions** (key/value pairs, **up to 30 per metric**).
- **EC2 default metrics:** CPU, network, disk (instance store), status checks. **Basic monitoring = 5-minute** intervals (free). **Detailed monitoring = 1-minute** (paid). Faster scaling reactions need detailed monitoring.
- **Not available by default on EC2: memory usage, disk space utilization, process counts.** Use the **CloudWatch agent** (unified agent) or a custom metric.
- **Custom metrics:** the **`PutMetricData`** API.
  - **Standard resolution:** 1 minute.
  - **High resolution:** **1 second** (`StorageResolution = 1`). Retrieve at 1, 5, 10, 30, 60 s periods.
  - Timestamps are accepted up to **2 weeks in the past and 2 hours in the future**.
  - Use `StatisticValues` or `Values`/`Counts` arrays to batch, reducing API calls and cost.
- **Embedded Metric Format (EMF):** write **structured JSON logs** that CloudWatch automatically **extracts as metrics**. Ideal for Lambda: no `PutMetricData` calls, no latency, and it works asynchronously. Powertools Metrics uses EMF.
- **Metric math**, **anomaly detection** (ML band), **Metrics Insights** (SQL-like queries), **metric streams** (to Firehose: Datadog, Splunk, S3).
- Retention: 1-second data for 3 h, 60-second for 15 days, 5-minute for 63 days, 1-hour for 455 days (15 months).

### 1.2 Alarms
- Watch a single metric or math expression. **States: `OK`, `ALARM`, `INSUFFICIENT_DATA`.**
- Settings: **period**, **evaluation periods**, **datapoints to alarm** (M out of N), threshold, **treatment of missing data** (missing, notBreaching, breaching, ignore).
- **High-resolution alarms:** periods of 10 s or 30 s (on high-resolution metrics).
- **Actions:** **SNS** notification, **EC2 actions** (stop, terminate, reboot, recover), **Auto Scaling** policies, Systems Manager OpsItems/incidents, Lambda.
- **Composite alarms** combine multiple alarms with AND/OR to **reduce alarm noise**.
- Test with `aws cloudwatch set-alarm-state`.
- Alarms on Lambda `Errors` or API Gateway `5XXError` drive **CodeDeploy/SAM rollbacks**.

### 1.3 CloudWatch Logs
- **Log groups** (one per app or function, e.g., `/aws/lambda/my-fn`) contain **log streams** (one per instance or execution environment).
- **Retention:** never expire by default. Configure 1 day – 10 years to control cost.
- Log events can be **encrypted with KMS** (associate the key at the log group level).
- Sources: SDK `PutLogEvents`, **CloudWatch agent** (EC2/on-prem), Lambda (automatic, with execution role permissions `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`), ECS (`awslogs` log driver), Elastic Beanstalk, API Gateway, CloudTrail, VPC Flow Logs, Route 53.
- **Metric filters:** create metrics from log patterns (e.g., count `"ERROR"` occurrences), then alarm on them. They apply **only to new log events**, not historical data.
- **CloudWatch Logs Insights:** interactive **query language** to search and analyze logs (`fields @timestamp, @message | filter @message like /ERROR/ | stats count() by bin(5m)`). Queries across multiple log groups; saved queries.
- **Subscription filters:** stream logs in real time to **Kinesis Data Streams, Data Firehose, Lambda, or OpenSearch**. Cross-account log aggregation goes through destinations.
- **Export to S3:** the `CreateExportTask` batch export (can take up to 12 h; not real time). For near real time use subscription filters to Firehose.
- **Live Tail:** stream logs interactively for debugging.
- **Data protection policies:** mask sensitive data (PII).
- **Log classes:** Standard vs **Infrequent Access** (cheaper; fewer features).

### 1.4 CloudWatch agent
- Collects **system-level metrics** (memory, disk, swap, netstat, processes) and **logs** from EC2 and on-premises servers.
- Configuration can be stored in **SSM Parameter Store** for fleet-wide use.
- Requires an IAM role with `CloudWatchAgentServerPolicy`.
- Supports StatsD and collectd for custom app metrics.
- The **old CloudWatch Logs agent** is deprecated in favor of the unified agent.

### 1.5 Other CloudWatch features
- **CloudWatch Synthetics canaries:** scripts (Node.js/Python, Puppeteer/Selenium) that monitor endpoints and APIs on a schedule, check availability and latency, and take screenshots.
- **CloudWatch RUM:** real user monitoring for web apps.
- **CloudWatch Evidently:** feature flags and A/B experiments (being retired; use **AppConfig** for feature flags).
- **Application Signals / ServiceLens:** APM view correlating metrics, logs and traces (X-Ray/OpenTelemetry). SLOs.
- **Container Insights** (ECS/EKS), **Lambda Insights** (Lambda system-level metrics via an extension: memory, CPU, cold starts).
- **Contributor Insights:** top-N contributors (e.g., top IPs, heavy DynamoDB keys, i.e., hot partitions).
- **Dashboards:** cross-account, cross-Region.

---

## 2. AWS X-Ray (distributed tracing)

### 2.1 Concepts
- Traces requests end to end across services. Builds a **service map** (visual graph) showing latency, errors (4xx), faults (5xx), and throttles (429) per node.
- **Trace:** an end-to-end request, identified by a **trace ID** that propagates in the **`X-Amzn-Trace-Id`** HTTP header.
- **Segment:** the work done by one service (e.g., your app on EC2).
- **Subsegment:** granular work inside a segment (e.g., a DynamoDB call, an HTTP call, a SQL query, or a custom block of code).
- **Annotations:** **key/value pairs that are INDEXED**. Use them for **filter expressions** to search traces (e.g., `annotation.customer_tier = "premium"`). Up to 50 per trace.
- **Metadata:** key/value data that is **NOT indexed**. Use it to store extra info you don't need to search.
- **Sampling:** controls how many requests are traced to reduce cost.
  - **Default rule: the first request each second (reservoir = 1) plus 5% of additional requests.**
  - Custom **sampling rules** can be defined in the console (no code change or redeploy needed) by service, host, path, method, and so on.
- **Groups:** filter expressions that create separate service maps and metrics.
- **X-Ray Insights:** anomaly detection.

### 2.2 How to instrument
1. **Code:** use the **X-Ray SDK** (or, increasingly, **AWS Distro for OpenTelemetry (ADOT)**, which AWS recommends for new instrumentation) to instrument AWS SDK calls, HTTP calls, and SQL.
2. **X-Ray daemon / agent:** the SDK sends segment data over **UDP port 2000** to the daemon, which batches and uploads it to the X-Ray API. Configure the address with the **`AWS_XRAY_DAEMON_ADDRESS`** env var.
3. **IAM permissions:** `xray:PutTraceSegments`, `xray:PutTelemetryRecords` (managed policy **`AWSXRayDaemonWriteAccess`**). Sampling rules need `xray:GetSamplingRules`, `xray:GetSamplingTargets`.

### 2.3 X-Ray by platform
| Platform | How to enable |
|---|---|
| **Lambda** | Enable **Active tracing** (console, SAM `Tracing: Active`). The daemon is built in. The execution role needs `AWSXRayDaemonWriteAccess`. Instrument the SDK in code for subsegments. Env vars: `_X_AMZN_TRACE_ID`, `AWS_XRAY_CONTEXT_MISSING`, `AWS_XRAY_DAEMON_ADDRESS`. |
| **EC2 / on-prem** | Install and run the **X-Ray daemon** (user data script); instance role with permissions |
| **Elastic Beanstalk** | Enable the X-Ray daemon option in the console or `.ebextensions` (`option_settings: aws:elasticbeanstalk:xray: XRayEnabled: true`); instance profile permissions |
| **ECS** | Run the X-Ray daemon as a **sidecar container** in the task definition (EC2 or Fargate), or as a **daemon service** (one per EC2 host). Map UDP 2000. Task role needs permissions. |
| **API Gateway** | Enable X-Ray tracing on the **stage** |
| Others | SNS, SQS (trace header propagation), Step Functions, App Runner, AppSync, ALB (adds the trace ID header only; doesn't send segments) |

- **Cross-account tracing:** the daemon assumes a role in the central account.
- **APIs for reading traces:** `GetTraceSummaries` (IDs and annotations matching a filter), **`BatchGetTraces`** (full traces by ID), `GetServiceGraph`, `GetTraceGraph`.
- **Troubleshooting "no traces showing up":** the daemon isn't running, IAM permissions are missing, active tracing isn't enabled, or sampling excluded the requests.

### 2.4 CloudWatch vs X-Ray vs CloudTrail
| | CloudWatch | X-Ray | CloudTrail |
|---|---|---|---|
| Answers | **How is it performing?** (metrics, logs, alarms) | **Where is the latency or error in a distributed request?** (traces) | **Who did what, when?** (API call auditing) |
| Example | CPU > 80% alarm | Slow DynamoDB call inside a microservice chain | Who deleted an S3 bucket |

---

## 3. AWS CloudTrail
- Records **API calls** (console, CLI, SDK, services) for governance, compliance and auditing. Enabled by default; **Event history shows 90 days** of management events.
- **Trails** deliver events to **S3** (long-term) and optionally CloudWatch Logs. They can cover all Regions and the whole organization.
- **Event types:** **management events** (control plane, e.g., `CreateBucket`, `RunInstances`; logged by default), **data events** (high volume, e.g., S3 `GetObject`/`PutObject`, Lambda `Invoke`, DynamoDB item operations; **not logged by default**, extra cost), **Insights events** (unusual API activity).
- **CloudTrail Lake:** SQL queries over event data.
- **EventBridge integration:** react to API calls (e.g., a rule on `DeleteTable` sends an SNS alert).
- Delivery delay is about 5–15 min (not real time).

---

## 4. Troubleshooting by error and symptom

### 4.1 HTTP and SDK errors
| Error | Meaning | Action |
|---|---|---|
| **400** `ValidationException` / bad request | Invalid parameters | Fix the request |
| **403** `AccessDenied` / `UnauthorizedOperation` | IAM, resource policy, KMS key policy, SCP, or explicit deny | Check policies; decode the message; Policy Simulator; CloudTrail |
| **404** | Resource not found (wrong Region, wrong name) | Check Region and identifiers |
| **429** / `ThrottlingException` / `TooManyRequestsException` / `ProvisionedThroughputExceededException` / `RequestLimitExceeded` | **Throttled** | **Exponential backoff with jitter**; request a quota increase; batch; cache; reduce calls |
| **500** | Internal server error | Retry with backoff |
| **502** (API Gateway) | Bad Lambda proxy response format, or Lambda error or permission issue | Fix the response format `{statusCode, headers, body}`; check Lambda logs |
| **503** | Service unavailable | Retry with backoff |
| **504** (API Gateway) | Integration timeout (> 29 s) | Optimize the backend; go async (SQS/Step Functions); request a higher timeout (Regional REST APIs) |

- **AWS SDKs retry automatically** on throttling and 5xx (standard or adaptive retry mode, with configurable max attempts). **Don't retry non-throttling 4xx errors.**
- **Exponential backoff with jitter:** wait = random(0, min(cap, base × 2^attempt)). This prevents the thundering herd problem.

### 4.2 Lambda troubleshooting
| Symptom | Likely cause and fix |
|---|---|
| `Task timed out after X seconds` | Increase the **timeout**; check downstream latency; a VPC Lambda can't reach the internet or an AWS service (**needs a NAT gateway or VPC endpoint**) |
| Slow / CPU-bound | **Increase memory** (CPU scales with it). Use **AWS Lambda Power Tuning** (Step Functions-based tool) or **AWS Compute Optimizer** to pick the best memory size. |
| Cold start latency | **Provisioned concurrency**, **SnapStart** (Java/Python/.NET), smaller package, init outside the handler, avoid VPC-heavy init |
| `TooManyRequestsException` (429) | Concurrency limit reached. Request a limit increase; set **reserved concurrency**; or put **SQS** in front as a buffer |
| `Runtime.ImportModuleError` / handler not found | Wrong handler name or missing dependency in the package or layer |
| Out of memory (`Runtime exited with error: signal: killed`) | Increase memory; check `Max Memory Used` in the `REPORT` log line |
| `AccessDeniedException` calling a service | **Execution role** is missing a permission |
| Trigger not invoking (e.g., S3) | Lambda **resource-based policy** missing for the service principal; event notification misconfigured |
| Async events lost | Configure an **on-failure destination / DLQ**; check the `AsyncEventsDropped` metric |
| SQS messages processed multiple times | Visibility timeout < function timeout (set it **≥ 6×**); make processing idempotent; use partial batch responses |
| Kinesis/DynamoDB stream stuck | A poison record blocks the shard: use **bisect on error**, max retry attempts, max record age, an on-failure destination. Watch the **`IteratorAge`** metric. |
| Can't connect to RDS / too many connections | Use **RDS Proxy**; initialize the connection outside the handler |
| `/tmp` full | Increase ephemeral storage (up to 10 GB) or clean up |

**Key Lambda CloudWatch metrics:** `Invocations`, `Errors`, `Throttles`, **`Duration`**, `ConcurrentExecutions`, `UnreservedConcurrentExecutions`, **`IteratorAge`** (stream lag), `DeadLetterErrors`, `DestinationDeliveryFailures`, `ProvisionedConcurrencySpilloverInvocations`, `AsyncEventAge`, `AsyncEventsDropped`.
**`REPORT` log line:** `Duration`, `Billed Duration`, `Memory Size`, `Max Memory Used`, `Init Duration` (cold start).

### 4.3 DynamoDB troubleshooting
| Symptom | Fix |
|---|---|
| `ProvisionedThroughputExceededException` | Backoff (SDK); better partition key / write sharding; increase capacity or auto scaling; on-demand; DAX for reads |
| Throttling even though total capacity isn't consumed | **Hot partition**: a poorly distributed key. Use **CloudWatch Contributor Insights** to find hot keys. |
| Base table writes throttled because of a GSI | GSI under-provisioned; raise GSI WCU |
| Scan is slow or expensive | Use Query with a proper key or GSI; parallel scan; smaller page size; avoid scans in hot paths |
| `ConditionalCheckFailedException` | Expected with optimistic locking: re-read and retry |
| `ItemCollectionSizeLimitExceededException` | LSI item collection > 10 GB: redesign keys or use a GSI |
| Reads return stale data | Use `ConsistentRead=true` (not on GSIs) |

Key metrics: `ConsumedReadCapacityUnits`, `ConsumedWriteCapacityUnits`, `ReadThrottleEvents`, `WriteThrottleEvents`, `ThrottledRequests`, `SuccessfulRequestLatency`, `SystemErrors`, `UserErrors`.

### 4.4 Other service troubleshooting
- **SQS:** messages reprocessed → visibility timeout too short / consumer not deleting. Many empty receives → enable **long polling**. Messages stuck in the DLQ → inspect, fix, **redrive**. Growing backlog → scale consumers on `ApproximateNumberOfMessagesVisible` / `ApproximateAgeOfOldestMessage`.
- **Kinesis:** `ProvisionedThroughputExceededException` → better partition key, **split shards**, backoff, batching (KPL). Consumer lag → enhanced fan-out, more shards or workers, `GetRecords.IteratorAgeMilliseconds`.
- **API Gateway:** `Latency` vs `IntegrationLatency`. The difference is API Gateway overhead; `IntegrationLatency` is backend time. Enable **execution logging** and **access logging**. Missing CORS headers → enable CORS / return headers from the Lambda proxy.
- **S3:** 503 `SlowDown` → spread across prefixes, backoff. Slow uploads → multipart and Transfer Acceleration. `AccessDenied` on SSE-KMS objects → missing `kms:Decrypt` / `kms:GenerateDataKey`.
- **ECS:** task stops immediately → check **stopped reason**, container logs (CloudWatch `awslogs`), image pull errors (execution role, ECR auth, no route to ECR from a private subnet: NAT or VPC endpoints).
- **EC2 / Beanstalk:** Beanstalk health degraded → `eb logs`, enhanced health, instance logs in `/var/log`.
- **CloudFormation:** check stack **Events** for the first `CREATE_FAILED` reason. `cfn-init` logs on the instance.
- **CodeDeploy:** agent logs `/var/log/aws/codedeploy-agent/codedeploy-agent.log`; deployment logs per lifecycle event.

---

## 5. Instrumenting code for observability

- **Structured logging (JSON)** with correlation IDs (request ID, trace ID) so logs can be queried in Logs Insights. Use log levels and avoid logging secrets or PII.
- **Lambda advanced logging controls:** JSON log format, log level filtering (application and system), custom log group.
- **Custom metrics** for business KPIs (orders/min) via **EMF** (preferred in Lambda) or `PutMetricData`.
- **Tracing** with X-Ray / OpenTelemetry. Add **annotations** for searchable business fields.
- **Health checks** (ALB target groups, Route 53, Beanstalk) and **Synthetics canaries**.
- **Alerts:** CloudWatch alarms → SNS → email, Slack (via Amazon Q Developer in chat applications, formerly AWS Chatbot), PagerDuty.
- **Powertools for AWS Lambda** (Python, TypeScript, Java, .NET): Logger, Metrics (EMF), Tracer, Idempotency, Parameters, Batch processing.
- Dashboards for golden signals: latency, traffic, errors, saturation.

---

## 6. Optimization

### 6.1 Performance optimization
| Goal | Approach |
|---|---|
| Faster reads from DynamoDB | **DAX** (microseconds), good key design, Query instead of Scan, ProjectionExpression |
| Reduce DB load for repeated queries | **ElastiCache** (lazy loading + TTL) |
| Reduce API latency and backend load | **API Gateway caching**, **CloudFront** |
| Global static content | **CloudFront** (+ S3 with OAC) |
| Faster S3 uploads | **Multipart upload**, **Transfer Acceleration** |
| Faster S3 downloads | **Byte-range fetches** (parallel), CloudFront |
| Higher S3 request rate | Spread across **prefixes** |
| Lambda CPU-bound | **More memory**, arm64 (Graviton) |
| Lambda cold starts | Provisioned concurrency, SnapStart, reuse connections |
| Lambda + RDS connections | **RDS Proxy** |
| Long synchronous work in an API | Make it async: API Gateway → SQS / Step Functions; return 202 and poll or WebSocket |
| Decouple and absorb spikes | **SQS** as a buffer |
| Parallel processing | SNS fan-out, Step Functions Parallel/Map, Kinesis shards, SQS + multiple consumers |
| Kinesis throughput | More shards, enhanced fan-out, KPL aggregation |
| Reduce chattiness | **Batch APIs**: `SendMessageBatch`, `BatchWriteItem`, `PutRecords`, `PutMetricData` batching |

### 6.2 Cost optimization for developers
- Lambda: right-size memory (Power Tuning / Compute Optimizer), **arm64**, avoid idle waiting (use Step Functions **wait states** instead of sleeping in Lambda), event filtering on event source mappings.
- SQS **long polling** (fewer empty receives). Batch requests.
- DynamoDB: on-demand for spiky, provisioned + auto scaling for steady; TTL to delete old data for free; avoid scans; project only needed attributes into GSIs.
- S3 lifecycle policies and storage classes; Intelligent-Tiering.
- CloudWatch Logs **retention** settings; Infrequent Access log class.
- KMS **Bucket Keys** and data key caching.
- Use **Step Functions Express** for high-volume short workflows.
- Spot instances for fault-tolerant workloads; Fargate Spot.
- HTTP API instead of REST API when advanced features aren't needed.

### 6.3 Concurrency and messaging choices
- Use **reserved concurrency** to protect downstream systems (e.g., a DB that can handle only 50 connections: reserved concurrency = 50).
- Choose SQS FIFO **only** when ordering or exactly-once processing is required. Standard queues have far higher throughput.
- Use **message group IDs** in FIFO to parallelize while keeping per-entity order (e.g., per customer).
- Use **EventBridge** for routing many event types to many targets with content filtering. **SNS** for high-throughput fan-out.

---

## 7. Troubleshooting and optimization exam traps
- "Find which microservice is slow" → **X-Ray**. "Who deleted the resource" → **CloudTrail**. "Alert on high error rate" → **CloudWatch alarm**.
- "Search traces by customer ID" → **X-Ray annotations** (not metadata).
- "Monitor EC2 memory" → **CloudWatch agent** (custom metric).
- "Custom metric every second" → **high-resolution** custom metric (`StorageResolution=1`).
- "Lambda emits custom metrics without extra API calls or latency" → **Embedded Metric Format**.
- "Count ERROR lines in logs and alarm" → **metric filter + alarm**.
- "Stream logs in real time to another system or account" → **subscription filter** (Kinesis/Firehose/Lambda).
- "Query logs ad hoc" → **CloudWatch Logs Insights**.
- "X-Ray not showing data from EC2" → **daemon not running or missing IAM permissions**.
- "X-Ray in ECS" → **sidecar container** running the daemon.
- "Reduce X-Ray cost" → **sampling rules**.
- "Lambda function slow and CPU-heavy" → **increase memory**.
- "API Gateway 504" → backend > 29 s → make it **asynchronous**.
- "API Gateway 502 with Lambda proxy" → **response format** wrong.
- "Throttling errors from any AWS API" → **exponential backoff + jitter**.
- "Hot partition" → better partition key or **write sharding**; use **Contributor Insights** to find it.
