# Domain 1 – Development with AWS Services (32%)

What this domain tests:
- Writing code for applications hosted on AWS (architectural patterns, event-driven, microservices, choreography vs orchestration, fan-out, idempotency, stateless design, loose coupling, retries with backoff)
- Writing code for AWS Lambda
- Using data stores in application development (DynamoDB, S3, caching, RDS)

---

## 1. Architectural patterns you must recognize

| Pattern | Meaning | AWS implementation |
|---|---|---|
| **Loose coupling** | Components communicate through a buffer so one failing or slow part doesn't break the other | SQS between producer/consumer |
| **Fan-out** | One message goes to many consumers | SNS → multiple SQS queues; EventBridge rule with multiple targets |
| **Event-driven** | React to state changes rather than polling | S3 events, DynamoDB Streams, EventBridge |
| **Orchestration** | A central coordinator controls the steps | Step Functions |
| **Choreography** | Services react to each other's events, with no central controller | EventBridge / SNS |
| **Stateless** | No session data on the compute node | Store session in ElastiCache/DynamoDB; Lambda is stateless |
| **Idempotency** | Processing the same request twice has the same effect as once | Idempotency key + DynamoDB conditional write; Powertools for Lambda idempotency utility |
| **Synchronous vs asynchronous** | The caller waits for the result, or fires and forgets | API Gateway → Lambda (sync); S3 → Lambda (async); SQS (async) |
| **Retries with exponential backoff and jitter** | Increasing waits between retries, with randomness | Built into AWS SDKs. Retry on **5xx and throttling (429, `ThrottlingException`, `ProvisionedThroughputExceededException`)**, but **not** on other 4xx errors. |

---

## 2. AWS Lambda (most important service on the exam)

### 2.1 Basics
- Runs code in response to events, billed per request and per GB-second (1 ms granularity).
- **Memory: 128 MB – 10,240 MB.** CPU and network scale proportionally with memory (at 1,769 MB = 1 vCPU). **To make a CPU-bound function faster, increase memory.**
- **Timeout: default 3 s, max 900 s (15 min).** Longer jobs belong in Step Functions, ECS/Fargate, or AWS Batch.
- **Ephemeral storage `/tmp`: 512 MB default, up to 10,240 MB.**
- Architectures: x86_64 or **arm64 (Graviton2)**, which gives up to ~34% better price-performance.
- Runtimes: Node.js, Python, Java, .NET, Ruby, custom runtime (`provided.al2023`) via the Runtime API, and container images (up to 10 GB).

### 2.2 Handler and execution environment
- `handler(event, context)`: `event` holds the input; `context` holds runtime info (`aws_request_id`, `get_remaining_time_in_millis()`, function name, memory limit, log group).
- **Code outside the handler runs once per execution environment (on cold start) and is reused on warm invocations.** Initialize SDK clients and DB connections **outside** the handler.
- Cold start = download code, start runtime, run init code. Ways to reduce it:
  - **Provisioned concurrency** keeps pre-initialized environments ready (costs money).
  - **Lambda SnapStart** (Java, Python, .NET) snapshots the initialized environment.
  - Smaller packages, fewer dependencies, lighter frameworks.
- Lifecycle phases: Init → Invoke → Shutdown.

### 2.3 Invocation models (very high yield)

| Model | Who calls | Retries | Error handling | Examples |
|---|---|---|---|---|
| **Synchronous** (`RequestResponse`) | Caller waits for the response | **Caller** is responsible | Error returned to caller | API Gateway, ALB, Cognito triggers, SDK `Invoke`, CloudFront (Lambda@Edge) |
| **Asynchronous** (`Event`) | Event queued internally by Lambda | **2 retries by default** (configurable 0–2), with event max age up to 6 h | **DLQ** (SQS/SNS) or, better, **Destinations** (on-success / on-failure: SQS, SNS, Lambda, EventBridge, S3) | S3, SNS, EventBridge, CloudWatch Logs, CodeCommit, SES, CloudFormation |
| **Event source mapping** (poller) | Lambda **polls** the source | Depends on source (see below) | Stream: retries until success or record expiry; use bisect-on-error, max retry attempts, max record age, on-failure destination. SQS: message returns to the queue after the visibility timeout, then goes to the **queue's** DLQ | SQS, Kinesis Data Streams, DynamoDB Streams, MSK, self-managed Kafka, MQ, DocumentDB |

Key points:
- Async invocations can be **processed more than once**, so make handlers idempotent.
- `aws lambda invoke --invocation-type Event` makes an async call; the default `RequestResponse` is sync. `DryRun` only validates parameters and permissions.
- **Destinations vs DLQ:** Destinations are newer, support success *and* failure, and send richer records. AWS recommends Destinations for async.
- For **SQS** event source mappings, the DLQ is configured **on the SQS queue** (redrive policy), not on Lambda.
- **SQS + Lambda:** set the queue **visibility timeout ≥ 6× the function timeout**. Use `ReportBatchItemFailures` (partial batch response) so only failed messages are retried.
- **Kinesis/DynamoDB Streams + Lambda:** records are processed **in order per shard**. A failing batch **blocks the shard** until it succeeds or expires. Configure `BisectBatchOnFunctionError`, `MaximumRetryAttempts`, `MaximumRecordAgeInSeconds`, an on-failure destination, and `ParallelizationFactor` (up to 10 concurrent batches per shard).
- Event filtering on event source mappings lets you invoke only for matching records, which reduces cost.

### 2.4 Permissions
- **Execution role** (identity-based): what the function **can do** (e.g., `dynamodb:PutItem`). Logging needs `AWSLambdaBasicExecutionRole` (CloudWatch Logs). Pollers need read permissions on the source (e.g., `AWSLambdaSQSQueueExecutionRole`, `AWSLambdaKinesisExecutionRole`). VPC needs `AWSLambdaVPCAccessExecutionRole` (ENI management).
- **Resource-based policy**: **who can invoke** the function (e.g., allow `s3.amazonaws.com`, `apigateway.amazonaws.com`, or another account). Added with `aws lambda add-permission`.

### 2.5 Concurrency and scaling
- Concurrency ≈ requests per second × average duration (s).
- **Default account limit: 1,000 concurrent executions per Region** (soft limit; request an increase).
- Each function scales by up to 1,000 concurrent executions every 10 seconds.
- Exceeding concurrency causes **throttling**: sync callers get **429 `TooManyRequestsException`**; async events are retried automatically for up to 6 h.
- **Reserved concurrency**: guarantees **and caps** concurrency for a function. Setting it to **0 stops all invocations** (an emergency kill switch). It protects other functions from one noisy function and protects downstream DBs.
- **Provisioned concurrency**: pre-initialized environments with no cold starts. It is configured on a **version or alias** (not `$LATEST`) and can be managed by Application Auto Scaling.

### 2.6 Versions and aliases
- `$LATEST` is mutable. **Publishing a version** creates an **immutable** snapshot (code + configuration) with its own ARN (`...:function:my-fn:3`).
- An **alias** is a mutable pointer to a version (`...:function:my-fn:prod`). Aliases **cannot point to other aliases**.
- **Weighted aliases** split traffic between two versions (e.g., 90% v1, 10% v2) for canary releases. CodeDeploy/SAM automate this.
- Point event sources and API Gateway stage variables at **aliases**, so you never have to reconfigure triggers.

### 2.7 Configuration
- **Environment variables:** key/value pairs, **4 KB total**, encrypted at rest with KMS. You can use a customer managed key and encryption helpers for in-transit protection. Don't put secrets here in plaintext; use Secrets Manager or Parameter Store (and the **AWS Parameters and Secrets Lambda Extension** for caching).
- **Layers:** share libraries, custom runtimes, or config. **Up to 5 layers per function.** Total unzipped size (function + layers) **≤ 250 MB**. Contents are extracted to `/opt`.
- **Deployment package limits:** 50 MB zipped (direct upload), 250 MB unzipped. Larger code goes in S3 (still 250 MB unzipped) or a **container image (up to 10 GB, stored in ECR)**.
- **Payload limits:** sync request/response **6 MB**. Async event: historically **256 KB**, raised to 1 MB in 2025. Many exam questions still use 256 KB.
- Lambda **in a VPC** reaches private resources (RDS, ElastiCache) through Hyperplane ENIs. **To reach the internet from a VPC Lambda you need a NAT Gateway** in a public subnet. Alternatively, use **VPC endpoints** for AWS services such as S3 and DynamoDB.
- Connecting Lambda to RDS at scale exhausts DB connections. Fix it with **RDS Proxy** (connection pooling, IAM auth, faster failover).
- **Lambda function URLs:** a dedicated HTTPS endpoint without API Gateway (auth type `AWS_IAM` or `NONE`; CORS supported; response streaming).
- **Lambda@Edge vs CloudFront Functions:** CloudFront Functions are JavaScript only, sub-millisecond, viewer request/response only, and very cheap. Lambda@Edge supports Node/Python, runs up to 5 s (viewer) / 30 s (origin), can make network calls, and handles all four triggers.
- **Lambda extensions** run alongside the function (monitoring, secrets caching). **Powertools for AWS Lambda** provides structured logging, metrics (EMF), tracing, and idempotency.

### 2.8 Testing Lambda
- `sam local invoke`, `sam local start-api`, test events in the console, unit tests with mocked SDK calls.

---

## 3. Amazon DynamoDB (second most important)

### 3.1 Core concepts
- Fully managed NoSQL key-value and document DB with single-digit millisecond latency at any scale.
- Table → items (rows) → attributes. **Max item size: 400 KB.** Store large objects in S3 and keep a pointer in DynamoDB.
- **Primary key:**
  - **Partition key** only (simple): must be unique, with high cardinality.
  - **Partition key + sort key** (composite): the combination must be unique. Items with the same partition key are stored together and sorted by the sort key.
- **Good partition key** = high cardinality and evenly distributed access (user_id, order_id). A **bad** one is status or date, which creates **hot partitions** and throttling.
  - Fix hot keys with **write sharding** (append a random or calculated suffix: `2024-01-01#7`).

### 3.2 Capacity modes
- **On-demand:** pay per request, with no capacity planning. Best for unpredictable or spiky workloads and new apps. More expensive per request.
- **Provisioned:** specify RCU/WCU, optionally with **auto scaling**. Cheaper for predictable traffic. Bursts are absorbed briefly by burst capacity (up to 300 s of unused capacity).
- Exceeding capacity raises **`ProvisionedThroughputExceededException`**. Fix it with exponential backoff (the SDK does this automatically), better key distribution, DAX, or more capacity.

### 3.3 Capacity calculations (always on the exam)

**WCU:** 1 WCU = **1 write/second for an item up to 1 KB**. Round item size **up** to the next KB.
**RCU:** 1 RCU = **1 strongly consistent read/second for an item up to 4 KB**, or **2 eventually consistent reads/second** of up to 4 KB. Round item size **up** to the next 4 KB.
**Transactional** reads/writes cost **2×**.

Worked examples:
1. 10 writes/s, 2.5 KB items: 2.5 rounds up to 3 KB, so 3 × 10 = **30 WCU**
2. 6 writes/s, 4.5 KB items: 5 KB per write, so 5 × 6 = **30 WCU**
3. 10 strongly consistent reads/s, 6 KB items: ⌈6/4⌉ = 2, so 2 × 10 = **20 RCU**
4. 10 eventually consistent reads/s, 6 KB items: 20 / 2 = **10 RCU**
5. 16 eventually consistent reads/s, 12 KB items: ⌈12/4⌉ = 3, so 3 × 16 = 48, and 48 / 2 = **24 RCU**
6. 5 transactional writes/s, 3 KB items: 3 × 5 × 2 = **30 WCU**
7. 4 transactional reads/s, 5 KB items: ⌈5/4⌉ = 2, so 2 × 4 × 2 = **16 RCU**

### 3.4 Read consistency
- **Eventually consistent** (default): may return stale data briefly; costs half.
- **Strongly consistent**: set `ConsistentRead=true` on `GetItem`, `Query` or `Scan`. **Not supported on GSIs.**

### 3.5 Secondary indexes

| | LSI (Local Secondary Index) | GSI (Global Secondary Index) |
|---|---|---|
| Key | **Same partition key**, different sort key | **Any** partition key and optional sort key |
| When created | **Only at table creation** | Any time |
| Limit | 5 per table | 20 per table (default) |
| Capacity | Uses the **table's** RCU/WCU | Has **its own** RCU/WCU |
| Consistency | Strong or eventual | **Eventual only** |
| Size limit | 10 GB per partition key value (item collection) | None |
| Throttling impact | n/a | **If a GSI is throttled on writes, the base table writes are throttled too.** Provision GSI WCU ≥ table WCU. |

- **Projections:** `KEYS_ONLY`, `INCLUDE` (selected attributes), `ALL`.

### 3.6 Reading data
- **GetItem:** one item by full primary key. Most efficient.
- **Query:** items with the **same partition key**, optionally narrowed by a **sort key condition** (`=, <, <=, >, >=, BETWEEN, begins_with`). Efficient.
- **Scan:** reads the **entire table**. Expensive and slow. It returns up to **1 MB per call** (same for Query), so paginate with `LastEvaluatedKey` → `ExclusiveStartKey`.
  - Reduce Scan impact with a smaller `Limit`, or speed it up with **parallel scan** (`Segment`, `TotalSegments`), which consumes more capacity.
- **FilterExpression** is applied **after** reading. **It does not reduce consumed RCU**, only the data returned.
- **ProjectionExpression:** return only specific attributes. It also doesn't reduce RCU, but reduces network payload.
- **KeyConditionExpression:** for Query on keys.
- **ExpressionAttributeNames** (`#name`) handle reserved words. **ExpressionAttributeValues** (`:val`) are placeholders.
- **PartiQL:** SQL-compatible syntax for DynamoDB.

### 3.7 Writing data
- `PutItem` (create or replace), `UpdateItem` (modify attributes; supports **atomic counters**: `SET #c = #c + :inc`), `DeleteItem`.
- **Conditional writes:** `ConditionExpression` (e.g., `attribute_not_exists(pk)` to avoid overwriting). Failure raises `ConditionalCheckFailedException`.
- **Optimistic locking:** keep a `version` attribute and update only `IF version = :expected`. The DynamoDBMapper / Enhanced Client `@DynamoDbVersionAttribute` does this for you.
- **Batch operations** (not atomic; each item succeeds or fails independently, so retry `UnprocessedItems` / `UnprocessedKeys` with backoff):
  - `BatchWriteItem`: up to **25** put/delete requests, 16 MB. **No updates.**
  - `BatchGetItem`: up to **100** items, 16 MB.
- **Transactions** (ACID, all-or-nothing): `TransactWriteItems` / `TransactGetItems`. Up to **100** items (4 MB), across multiple tables. They cost **2×** capacity. Use a `ClientRequestToken` for idempotency.

### 3.8 DynamoDB Streams
- An ordered log of item-level changes (create/update/delete), **retained 24 hours**.
- **StreamViewType:** `KEYS_ONLY`, `NEW_IMAGE`, `OLD_IMAGE`, `NEW_AND_OLD_IMAGES`.
- Consumed by **Lambda** (event source mapping) or the KCL with the DynamoDB Streams Kinesis Adapter.
- Use cases: trigger workflows (send a welcome email on a new user), replicate, aggregate, audit, sync to OpenSearch.
- **Global Tables** (multi-Region, multi-active replication) require Streams with `NEW_AND_OLD_IMAGES`.
- Alternatively, **Kinesis Data Streams for DynamoDB** gives longer retention and more consumers.

### 3.9 TTL
- Set a Number attribute to an **epoch timestamp in seconds**. DynamoDB deletes expired items **at no WCU cost**, typically within a few days.
- Expired items may still appear in reads until deleted, so filter them out. Deletions appear in Streams (useful for archiving to S3).

### 3.10 DAX (DynamoDB Accelerator)
- An in-memory, write-through cache **in front of DynamoDB**, with **microsecond** latency. It is API-compatible, so only minimal code changes are needed (swap the client).
- **Item cache** (GetItem/BatchGetItem) and **query cache** (Query/Scan). Default TTL **5 min**.
- Use it for read-heavy, hot keys, and eventually consistent reads. **Not useful for strongly consistent reads** (those pass through) or write-heavy apps.
- DAX vs ElastiCache: DAX for DynamoDB item/query caching. ElastiCache for **computed/aggregated results**, sessions, and caching for other databases.

### 3.11 Other DynamoDB features
- **Backups:** on-demand, **PITR** (point-in-time recovery, up to 35 days), export to S3.
- **Encryption at rest** is always on (AWS owned key, AWS managed key, or customer managed key).
- **Fine-grained access control:** IAM condition `dynamodb:LeadingKeys` restricts users (e.g., Cognito identities) to items whose partition key equals their ID: `${cognito-identity.amazonaws.com:sub}`.
- **Single-table design:** multiple entity types in one table with overloaded keys (`PK=USER#123`, `SK=ORDER#456`).
- Error codes: `ProvisionedThroughputExceededException`, `ConditionalCheckFailedException`, `TransactionCanceledException`, `ValidationException`, `ItemCollectionSizeLimitExceededException` (LSI 10 GB).

---

## 4. Amazon S3 (development aspects)

- Objects up to **5 TB**. A **single PUT is max 5 GB**. **Multipart upload is recommended for > 100 MB and required for > 5 GB**; parts upload in parallel and failed parts can be retried. Use a lifecycle rule to **abort incomplete multipart uploads**.
- **Strong read-after-write consistency** for all operations (since Dec 2020).
- **Performance:** **3,500 PUT/COPY/POST/DELETE and 5,500 GET/HEAD requests per second per prefix.** Spread keys across prefixes to scale.
- **S3 Transfer Acceleration:** faster long-distance uploads via CloudFront edge locations (bucket name must be DNS-compliant, no dots).
- **Byte-range fetches:** parallel downloads, or fetch only part of an object (e.g., a file header).
- **Presigned URLs:** grant temporary access to a private object for GET or PUT, **with the permissions of the IAM principal that generated them**. CLI default expiry 3,600 s; max 7 days when signed with IAM user credentials. If signed with a role's temporary credentials, the URL expires when the credentials expire.
  - Use case: let users upload directly to S3 from a browser without passing through your servers.
- **CORS:** configure on the **bucket being requested** (the target) to allow other origins (`AllowedOrigins`, `AllowedMethods`, `AllowedHeaders`, `ExposeHeaders`, `MaxAgeSeconds`).
- **Event notifications:** on `s3:ObjectCreated:*`, `ObjectRemoved`, etc., sent to **Lambda, SQS, SNS** (destination resource policies must allow S3), or **EventBridge** (more filtering, more targets, archive/replay).
- **Versioning:** protects against overwrite or deletion (deletes add a delete marker). **MFA Delete** can be enabled only by the root user via CLI.
- **Storage classes:** Standard, Intelligent-Tiering, Standard-IA, One Zone-IA, Glacier Instant Retrieval, Glacier Flexible Retrieval (Expedited 1–5 min / Standard 3–5 h / Bulk 5–12 h), Glacier Deep Archive (Standard 12 h / Bulk 48 h), and Express One Zone (single-digit ms, directory buckets).
- **Lifecycle rules:** transition or expire objects and old versions.
- **Static website hosting:** endpoint is `http://bucket.s3-website-region.amazonaws.com`. Use **CloudFront + OAC** for HTTPS and a private bucket.
- **Object Lock** (WORM: governance/compliance modes, legal hold) and **Glacier Vault Lock**.
- **Encryption:** see [02-Security.md](02-Security.md#5-encryption-in-s3).
- **S3 Object Lambda:** transform objects on GET (e.g., redact PII, resize) using a Lambda function.

---

## 5. Messaging and streaming

### 5.1 Amazon SQS (Simple Queue Service)

| Feature | Standard | FIFO |
|---|---|---|
| Throughput | Nearly unlimited | **300 msg/s** without batching, **3,000** with batching (high-throughput mode supports much more) |
| Ordering | Best effort | **Strict order within a `MessageGroupId`** |
| Delivery | **At-least-once** (duplicates possible) | **Exactly-once processing** (dedup within a 5-min window) |
| Name | any | **Must end with `.fifo`** |

- **Message size: up to 256 KB** (raised to 1 MiB in 2025). For larger payloads use the **SQS Extended Client Library** (Java/Python), which stores the payload in S3 and sends a pointer (up to 2 GB).
- **Retention:** 1 min – 14 days (**default 4 days**).
- **Visibility timeout:** after a consumer receives a message, it is hidden from others for this time (**default 30 s**, 0 s – 12 h). If the message isn't deleted in time, it becomes visible again and **may be processed twice**.
  - Long-running processing: call **`ChangeMessageVisibility`** to extend it.
  - The consumer **must call `DeleteMessage`** after successful processing.
- **Long polling:** `WaitTimeSeconds` 1–20 s (queue attribute `ReceiveMessageWaitTimeSeconds`). It **reduces empty responses and cost**, so always prefer it over short polling.
- **`ReceiveMessage` returns up to 10 messages** per call (`MaxNumberOfMessages`).
- **Delay queue:** 0–15 min delay before messages become visible (queue-level `DelaySeconds`, or per-message timers for standard queues only).
- **Dead-letter queue (DLQ):** after `maxReceiveCount` failed receives, the message moves to the DLQ. A FIFO queue's DLQ must also be FIFO. Use **DLQ redrive** to move messages back to the source.
- **FIFO deduplication:** `MessageDeduplicationId` or **content-based deduplication** (SHA-256 of the body).
- **FIFO `MessageGroupId`:** ordering is per group; different groups can be consumed in parallel (one consumer per group at a time).
- Scaling consumers: an EC2 Auto Scaling group based on the **`ApproximateNumberOfMessagesVisible`** metric (or backlog per instance).
- **Security:** SSE (SQS-managed SSE-SQS or KMS), access policies (resource-based, e.g., allow SNS/S3 to send), HTTPS.
- Batch APIs: `SendMessageBatch`, `DeleteMessageBatch`, `ChangeMessageVisibilityBatch` (up to 10) reduce cost.

### 5.2 Amazon SNS (Simple Notification Service)
- Pub/sub: a publisher sends to a **topic**, which pushes to all **subscribers**: SQS, Lambda, HTTP/S, email, SMS, mobile push, Firehose.
- **Fan-out pattern:** SNS → multiple SQS queues for parallel, independent, durable processing. The SQS queue policy must allow `sns.amazonaws.com` to `SendMessage`.
- **Message filtering:** a JSON **filter policy on the subscription** (by message attributes or body), so each subscriber receives only relevant messages.
- **FIFO topics:** ordering and dedup; subscribers must be SQS FIFO (or standard) queues.
- Up to 12.5 million subscriptions per topic and 100,000 topics per account.
- Message delivery retries for HTTP endpoints follow a delivery policy. Subscription-level DLQ (SQS).

### 5.3 Amazon EventBridge
- Serverless event bus. Event sources: **AWS services (default bus), custom apps (custom bus), SaaS partners (partner bus)**.
- **Rules** match **event patterns** (JSON) or schedules and route to **targets** (Lambda, SQS, SNS, Step Functions, Kinesis, API destinations (HTTP), ECS tasks, other buses/accounts, and more). Up to 5 targets per rule.
- **Input transformer** reshapes events before sending.
- **Archive and replay** events. **Schema registry** discovers schemas and generates code bindings.
- **EventBridge Scheduler:** cron/rate/one-time schedules at scale (replaces CloudWatch Events scheduled rules).
- **EventBridge Pipes:** point-to-point source → filter → enrich → target (e.g., SQS → Step Functions).
- Cross-account: put events onto another account's bus (resource policy on the bus).
- EventBridge was formerly CloudWatch Events; the same API is still used.

### 5.4 Amazon Kinesis

**Kinesis Data Streams (KDS)**: real-time streaming with replay.
- Capacity in **shards**: per shard **1 MB/s or 1,000 records/s IN**, **2 MB/s OUT** (shared among standard consumers). **Enhanced fan-out** gives **2 MB/s per consumer per shard** with push (`SubscribeToShard`) and lower latency.
- Modes: **provisioned** (you manage shards; split for more capacity, merge to reduce) or **on-demand**.
- **Retention:** 24 h default, up to 365 days. **Replay** is possible, unlike SQS.
- **Partition key** determines the shard. Ordering is guaranteed **within a shard**. Hot partition keys cause **`ProvisionedThroughputExceededException`**. Fix it with a better partition key (high cardinality), retries with backoff, or more shards (split).
- Record max size: 1 MB.
- **Producers:** SDK `PutRecord`/`PutRecords` (batching reduces cost), **KPL** (Kinesis Producer Library: batching, aggregation, async), Kinesis Agent.
- **Consumers:** Lambda, the **KCL** (Kinesis Client Library), Firehose, Managed Service for Apache Flink.
  - **KCL:** **each shard is processed by at most one KCL worker**, but one worker can process many shards. It uses a **DynamoDB table** for checkpointing and leases. Max useful workers = number of shards.

**Amazon Data Firehose** (formerly Kinesis Data Firehose): load streaming data into destinations.
- **Near real-time** (buffering by size or time, e.g., 1–15 MB / 0–900 s). **Fully managed, auto scaling, no replay.**
- Destinations: **S3, Redshift (via S3), OpenSearch, Splunk**, HTTP endpoints, Datadog, and others.
- Optional **Lambda transformation**, format conversion (JSON to Parquet/ORC), and compression. Failed records go to an S3 backup.

**SQS vs SNS vs Kinesis**

| | SQS | SNS | Kinesis Data Streams |
|---|---|---|---|
| Model | Pull (queue) | Push (pub/sub) | Pull (stream) |
| Consumers | Each message is processed by one consumer, then deleted | All subscribers get a copy | Many consumers read the same data |
| Persistence | Up to 14 days | None (no persistence) | Up to 365 days, replayable |
| Ordering | FIFO queues | FIFO topics | Per shard |
| Use case | Decoupling, buffering work | Notifications, fan-out | Real-time analytics, logs, clickstream, ordered big data |

### 5.5 AWS Step Functions
- Serverless **orchestration** of workflows defined in **Amazon States Language (ASL)**, which is JSON.
- **State types:** `Task` (do work: Lambda or 200+ direct AWS SDK integrations), `Choice` (branching), `Parallel`, `Map` (iterate; **Distributed Map** for large-scale S3 data), `Wait`, `Pass`, `Succeed`, `Fail`.
- **Error handling:** `Retry` (ErrorEquals, IntervalSeconds, MaxAttempts, BackoffRate) and `Catch` (fallback state). Predefined errors: `States.ALL`, `States.Timeout`, `States.TaskFailed`, `States.Permissions`.
- **Integration patterns:**
  - Request/Response (default): call the service and continue.
  - **Run a job (`.sync`)**: wait for the job to complete (e.g., ECS task, Glue job).
  - **Wait for callback (`.waitForTaskToken`)**: pause until an external system calls `SendTaskSuccess`/`SendTaskFailure` with the token. Used for **human approval** and long external processes.
- **Standard vs Express:**

| | Standard | Express |
|---|---|---|
| Max duration | **1 year** | **5 minutes** |
| Execution semantics | **Exactly-once** | **At-least-once** (async) / at-most-once (sync) |
| Pricing | Per state transition | Per execution, duration and memory |
| Use case | Long-running, auditable, non-idempotent | High-volume event processing (IoT, streaming) |
| History | Viewable in the console for 90 days | CloudWatch Logs |

- Input/output processing: `InputPath`, `Parameters`, `ResultSelector`, `ResultPath`, `OutputPath`. Newer workflows use **JSONata** with `Arguments`/`Output`.

### 5.6 Amazon MQ
- Managed ActiveMQ/RabbitMQ for **migrating existing apps** that use standard protocols (AMQP, MQTT, STOMP, OpenWire, JMS) **without rewriting** them for SQS/SNS.

---

## 6. Caching

### 6.1 Amazon ElastiCache
- Managed in-memory cache with **Redis OSS / Valkey** or **Memcached**.
- Requires **application code changes** (unlike DAX).

| Redis / Valkey | Memcached |
|---|---|
| Replication, Multi-AZ with auto-failover | No replication |
| Persistence (AOF/snapshots), backup/restore | No persistence |
| Rich data types: sorted sets (leaderboards), lists, hashes, pub/sub | Simple key/value |
| Read replicas | Multi-threaded, sharding |
| IAM auth, Redis AUTH, encryption | SASL auth |

**Caching strategies (very commonly tested):**
- **Lazy loading (cache-aside):** read from the cache; on a **miss**, read from the DB and write to the cache.
  - Only requested data is cached, and node failure isn't fatal.
  - Cache misses cost 3 round trips, and data can become **stale**.
- **Write-through:** write to the DB **and** the cache on every write.
  - Data is never stale, but every write is slower and the cache fills with unread data (**cache churn**). Missing data persists until written.
  - Usually combined with lazy loading.
- **TTL:** add expiry to limit staleness. Combine with either strategy.
- **Eviction:** LRU when memory is full. Scale up or out if evictions are high.
- **Session store:** store web sessions in ElastiCache (or DynamoDB) so apps are stateless behind a load balancer.

### 6.2 Other caching layers
- **CloudFront:** edge caching for static and dynamic content. Cache key = path + selected headers, cookies and query strings (via cache policies). Invalidations (`/*`) cost money; prefer **versioned file names**.
- **API Gateway cache:** see section 7.
- **DAX:** for DynamoDB.
- **In-Lambda caching:** variables outside the handler persist across warm invocations.

---

## 7. Amazon API Gateway

### 7.1 API types

| | REST API | HTTP API | WebSocket API |
|---|---|---|---|
| Features | Full: API keys, usage plans, caching, request validation, mapping templates (VTL), WAF, private endpoints, X-Ray | Simpler, **cheaper (~70%)**, lower latency; JWT authorizers, Lambda/HTTP proxy | Two-way, stateful (chat, games, dashboards) |
| Endpoint types | Edge-optimized (default, via CloudFront), Regional, Private (VPC endpoint) | Regional | Regional |

- WebSocket routes: `$connect`, `$disconnect`, `$default`, custom routes. The backend sends messages to clients using `@connections` with the connection ID (callback URL).

### 7.2 Integration types
- **Lambda proxy (`AWS_PROXY`):** the full request is passed to Lambda as-is. **Lambda must return a specific format** `{statusCode, headers, body}`. A malformed response produces a **502 Bad Gateway**.
- **Lambda / AWS (non-proxy, `AWS`):** you configure **mapping templates** (Velocity Template Language, VTL) to transform the request and response. Can call any AWS service directly (e.g., put to SQS, Step Functions `StartExecution`, DynamoDB) **without Lambda**.
- **HTTP proxy (`HTTP_PROXY`):** pass through to an HTTP backend.
- **HTTP (non-proxy):** with mapping templates.
- **MOCK:** returns a response without a backend, for testing and development.
- **VPC Link:** reach private resources (NLB for REST API; ALB/NLB/Cloud Map for HTTP API).

### 7.3 Stages and deployments
- Changes take effect **only after you deploy** to a **stage** (dev, test, prod).
- **Stage variables:** like environment variables for stages. Use them to point to a Lambda **alias** (`arn:...:function:my-fn:${stageVariables.lambdaAlias}`). You must grant API Gateway permission to invoke each alias.
- **Canary deployments:** send a percentage of stage traffic to a canary release, then promote it.
- Stage settings: logging, X-Ray tracing, throttling, caching, client certificates.
- Export or import with **OpenAPI 3 / Swagger** definitions.

### 7.4 Throttling and quotas
- Default **account-level limit: 10,000 requests/second, burst 5,000** per Region (soft).
- Stage and method-level throttling; **usage plans + API keys** give per-client throttling and quotas (e.g., for monetized APIs). The client sends the key in the **`x-api-key` header**. API keys are for **identification and metering, not authorization**.
- Exceeding a limit returns **429 Too Many Requests**. Clients should retry with backoff.

### 7.5 Caching (REST APIs only)
- Per stage. **Default TTL 300 s**, max 3,600 s, 0 disables it. Size 0.5 GB – 237 GB. Can be overridden per method. Encryption optional.
- Clients can bypass the cache with the header **`Cache-Control: max-age=0`** if authorized (`execute-api:InvalidateCache`). Enable "Require authorization" so anyone can't flush your cache.
- Flush the entire cache from the console or API.

### 7.6 Security and authorization
- **IAM authorization (SigV4):** for AWS users, roles and services (`execute-api:Invoke`). Combine with a **resource policy** for cross-account access, IP restrictions, or VPC endpoint restrictions.
- **Cognito user pool authorizer:** validates the JWT (ID or access token in the `Authorization` header). No custom code needed. Authentication only; authorization logic lives in the backend.
- **Lambda authorizer** (custom authorizer):
  - **TOKEN** type: bearer token (JWT, OAuth).
  - **REQUEST** type: headers, query strings, stage variables, context.
  - Returns an **IAM policy**. The result can be **cached** (default 300 s, up to 1 h).
  - Use it for third-party identity providers (Auth0, custom OAuth).
- **HTTP API:** JWT authorizer (OIDC/OAuth 2.0), Lambda authorizer, IAM.
- **Resource policies:** restrict by source IP, VPC endpoint, or account.
- **mTLS** with custom domain names. **AWS WAF** for REST APIs.

### 7.7 CORS
- Needed when a browser on domain A calls an API on domain B. Enable it on the resource. This adds an **OPTIONS** preflight method and `Access-Control-Allow-Origin/Methods/Headers` response headers.
- With **Lambda proxy integration**, **your Lambda must return the CORS headers itself**.

### 7.8 Error codes to know
| Code | Meaning |
|---|---|
| 400 | Bad request (validation failed) |
| 403 | Access denied, WAF block, or wrong path/stage |
| 429 | **Throttled** (rate or quota exceeded) |
| 502 | **Bad gateway**: usually an incompatible Lambda proxy response or Lambda error |
| 503 | Service unavailable |
| 504 | **Integration timeout**: backend took longer than **29 s** (default max integration timeout). Regional/private REST APIs can request a higher limit. |

### 7.9 Other
- **Request validation** (REST): validate body against a JSON schema or required params before calling the backend. Invalid requests get a 400.
- **Gateway responses** customize error responses (e.g., add CORS headers to 4xx/5xx).
- CloudWatch metrics: `Count`, `Latency` (total), `IntegrationLatency` (backend only), `4XXError`, `5XXError`, `CacheHitCount`, `CacheMissCount`.
- Logging: execution logs and access logs (custom format).

---

## 8. Other developer-relevant services

### AWS AppSync
- Managed **GraphQL** (and pub/sub) API. Resolvers connect to DynamoDB, Lambda, RDS (Aurora), OpenSearch, and HTTP.
- **Real-time subscriptions** over WebSockets, **offline sync** for mobile (with Amplify DataStore). Replaced Cognito Sync for user data sync.
- Auth: API key, IAM, Cognito user pools, OIDC, Lambda.

### AWS Amplify
- Framework + hosting for full-stack web/mobile apps (React, Vue, Next.js, iOS, Android). Amplify Hosting gives CI/CD from Git. Amplify Gen 2 is TypeScript code-first and uses CDK under the hood.
- Libraries for Auth (Cognito), Data (AppSync/DynamoDB), Storage (S3).

### Amazon RDS / Aurora (developer view)
- Read replicas (async; scale reads; the app must use a separate endpoint) vs **Multi-AZ** (sync standby for **high availability**, not for scaling reads).
- **RDS Proxy:** connection pooling for Lambda, IAM auth, Secrets Manager integration, reduced failover time.
- **IAM DB authentication** (MySQL/PostgreSQL): a 15-minute auth token instead of a password.
- **Aurora Serverless v2:** auto-scaling capacity. Aurora has **reader** and **writer** endpoints.

### AWS SDK and CLI essentials
- **Credential provider chain** (typical order for the SDK/CLI):
  1. Explicit credentials in code (avoid)
  2. **Environment variables** (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`)
  3. Shared credentials/config files (`~/.aws/credentials`, `~/.aws/config`), including SSO and assume-role profiles
  4. Container credentials (ECS task role)
  5. **EC2 instance profile** (IMDS)
- **Best practice: use IAM roles** (instance profile, task role, Lambda execution role). **Never** hard-code keys or commit them.
- Region resolution: `AWS_REGION`/`AWS_DEFAULT_REGION` env var, config file, or the SDK client parameter. Some SDKs require an explicit Region.
- CLI tips:
  - `--dry-run` (EC2): checks permissions without running the action (`DryRunOperation` vs `UnauthorizedOperation`).
  - `aws sts decode-authorization-message --encoded-message ...` decodes an encoded authorization failure message (requires `sts:DecodeAuthorizationMessage`).
  - `aws sts get-caller-identity`: who am I.
  - `--query` (JMESPath), `--output json|table|text`.
  - **Pagination:** `--max-items`, `--starting-token`, `--page-size`. Use `--page-size` to avoid timeouts on large lists.
  - `--profile`, `aws configure`, `aws configure sso`.
- **SigV4 signing** is handled automatically by the SDK/CLI. For raw HTTP, sign with headers or a query string (presigned).
- **EC2 instance metadata:** `http://169.254.169.254/latest/meta-data/` (IAM role credentials at `/iam/security-credentials/<role>`). **IMDSv2** requires a session token (PUT to `/latest/api/token`). User data is at `/latest/user-data`.

### Other services to recognize
- **Amazon SES:** send email (transactional or marketing) from apps.
- **AWS AppConfig** (part of Systems Manager): feature flags and dynamic configuration with validation and gradual deployment and rollback.
- **Amazon OpenSearch Service:** full-text search and log analytics. Often fed from DynamoDB Streams.
- **Amazon Cognito Sync:** deprecated, replaced by AppSync.
- **AWS Serverless Application Repository (SAR):** share and deploy SAM apps.
