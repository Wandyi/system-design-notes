# AWS Lambda & Step Functions at Production Scale

Staff-level scenarios, patterns, and corner cases for building scalable applications with Lambda and Step Functions. 
Focuses on what breaks at scale, where the service boundaries are sharp, and how to design around them.

---

## Table of Contents

1. Mental Models & Service Limits You Must Know
2. Scenario 1: Event-Driven Async API Backend
3. Scenario 2: High-Fanout Event Processing (SNS/SQS/EventBridge)
4. Scenario 3: Stream Processing (Kinesis / DynamoDB Streams / MSK)
5. Scenario 4: Long-Running Workflows (Order, Saga)
6. Scenario 5: Massive Parallel Processing (Distributed Map)
7. Scenario 6: Human-in-the-Loop Workflows (Approval, Callback)
8. Scenario 7: File / Media Processing Pipeline
9. Scenario 8: ETL & Analytics Pipelines
10. Scenario 9: ML Inference & Batch Scoring
11. Scenario 10: Scheduled / Cron Workflows
12. Scenario 11: Serverless SaaS Multi-Tenant Platform
13. Cross-Cutting: Idempotency, DLQs, Observability, Cost, Security
14. Corner Cases & Anti-Patterns
15. When NOT to Use Lambda / Step Functions
16. Decision Matrix

---

## 1. Mental Models & Service Limits You Must Know

### Lambda — the numbers that shape design

| Limit | Value | Design implication |
|---|---|---|
| Max execution duration | 15 min | Anything longer → Step Functions, Fargate, or split work |
| Memory | 128 MB – 10,240 MB | CPU scales linearly with memory; "right-size for speed" |
| Payload (sync invocation) | 6 MB | Use S3 pointers for larger |
| Payload (async invocation) | 256 KB | Silently truncated — test this! |
| Payload (response streaming) | 20 MB | Newer Function URLs / invoke modes |
| `/tmp` | 512 MB default, up to 10 GB | Container reuse caches this |
| Concurrent executions (account) | 1000 default (soft) | Request increase early, don't wait for incident |
| Concurrency scaling rate | 1000/min per region (bursts higher) | Flash traffic can throttle |
| Environment variables | 4 KB total | Use Secrets Manager / SSM for large config |
| Deployment package (zipped) | 50 MB direct, 250 MB S3 | Containers up to 10 GB for ML payloads |

### Step Functions — the numbers that shape design

| Limit | Value | Design implication |
|---|---|---|
| Execution history | 25,000 events | Long workflows must shard or use Express |
| Max execution duration (Standard) | 1 year | Suitable for very long-running waits |
| Max execution duration (Express) | 5 min | Only for short high-volume workflows |
| State transition payload | 256 KB | **The single biggest footgun** — use S3 pointers |
| Task token timeout | 1 year | Callback pattern for human approval |
| Started executions (Standard) | 2000/s per account | Bursts: region limit, then throttle |
| Started executions (Express) | Nearly unlimited | Billed per request + duration |
| Distributed Map | 10,000 concurrent child executions | Big-data fanout |
| Workflow types pricing | Standard: per transition ($0.025/1K); Express: per request + duration | 10–100× cost difference |

**Standard vs Express — pick wrong, pay a lot.**

- **Standard**: exactly-once semantics, full audit, 1-year duration, priced per state transition. Use for workflows with business value per execution (orders, onboardings, sagas).
- **Express**: at-least-once semantics, 5-min duration, priced like Lambda (per invocation + duration). Use for high-volume event processing, streaming ETL, idempotent work where ~$0.025/1K transitions × millions would be a bill.

---

## 2. Scenario 1: Event-Driven Async API Backend

### Pattern

Client POSTs → API Gateway → Lambda → writes to DB + publishes event → returns 202 Accepted with job ID. Downstream consumers process asynchronously.

```
Client → API GW → Lambda (validate, enqueue) → SQS → Worker Lambda → downstream
                                    ↓
                                 DynamoDB (job state)
```

### Scalability design

- **API-tier Lambda**: 128 MB (cheap), pure I/O-bound. Provisioned concurrency if TTFT matters for logged-in users.
- **Worker Lambda**: tuned memory for its actual workload. SQS batch size 10 (default) is rarely optimal — size based on per-message latency.
- **DynamoDB on-demand** for job state to absorb spikes without capacity planning.

### Corner cases

- **Cold start on the sync path**: API Gateway → Lambda cold start is 200–800 ms (1–3 s for Java/VPC). Users feel it on the first hit.
  - Mitigation: provisioned concurrency for the API tier. Or SnapStart for Java (~80% cold-start reduction).
- **VPC-attached API Lambda doubles cold-start pain**: the ENI attach has improved but still adds ~100ms.
  - If the Lambda only needs to hit AWS APIs, pull it out of VPC and use VPC endpoints instead.
- **API Gateway 29-second hard limit**: sync Lambda that runs longer = API times out even though Lambda keeps running.
  - Design for async from day one. 202-Accepted pattern, job status polling, or WebSockets for push.
- **429 on burst from cold account**: Lambda scales 1000 concurrent/min. A legitimate spike to 5000 concurrent in 30s gets throttled.
  - Reserve concurrency explicitly; request limit increase; add SQS in front as a shock absorber.
- **Silent truncation on async invocation**: if your payload goes over 256 KB via async invoke, it fails silently to DLQ (if configured) or is dropped.
  - Validate payload size at the edge; always configure DLQ on async invocations.

---

## 3. Scenario 2: High-Fanout Event Processing (SNS/SQS/EventBridge)

### Pattern: SNS → multiple SQS → multiple Lambda consumers

Order placed → SNS topic → fanned out to `inventory-queue`, `email-queue`, `fraud-queue`, `analytics-queue`. Each queue has its own Lambda consumer with independent scaling and retry policy.

### Why SNS → SQS → Lambda, not SNS → Lambda directly

- **Durability**: SQS persists; SNS → Lambda direct loses messages if Lambda is down or throttled.
- **Backpressure**: SQS queue depth smooths bursts; Lambda scales to consume.
- **Per-consumer isolation**: one consumer misbehaving doesn't affect others.
- **Per-consumer DLQ**: SQS DLQ per queue with configurable max receives.

### Scalability

- **Lambda scales with SQS** by increasing concurrent pollers (up to `maximum concurrency` on the event source mapping).
- **Batch size + batch window**: batch size 10 is wrong for most workloads. For low-latency workloads, size 1 with `MaximumBatchingWindowInSeconds` 0. For high-throughput, size 100 + a 5s window lets Lambda amortize.
- **Partial batch response**: return `batchItemFailures` so only failed messages are re-delivered. Without this, a single bad message retries the whole batch repeatedly — poison-pill amplification.

### Corner cases

- **Retry storms on downstream failure**: DB is slow → Lambda times out → SQS re-delivers → more Lambdas → DB slower.
  - Use `MaximumConcurrency` on the event source mapping (up to 1000) to cap Lambda concurrency per queue.
  - Set `ReservedConcurrentExecutions` to isolate this Lambda's noisy failures from the rest of the account.
- **Poison pill**: one bad message that always fails.
  - Visibility timeout must be > Lambda timeout × 6 (for 3 retries at least).
  - `maxReceiveCount` on the redrive policy sends to DLQ after N failures. Always set this; default is infinite retry.
- **FIFO queue + Lambda = low concurrency**: FIFO queues preserve order per MessageGroupId. Concurrency is capped by group count.
  - If you don't truly need ordering, use standard SQS. If you do, shard by a high-cardinality key (e.g., user_id) and route across many groups.
- **Visibility timeout race**: Lambda takes longer than visibility timeout → message re-delivered while still being processed → double-processing.
  - Set visibility timeout to ≥ 6× Lambda timeout. And make the handler idempotent.
- **EventBridge vs SNS**: EventBridge adds schema registry, rule-based routing, content filtering, cross-account, archive/replay. SNS is cheaper and lower latency. Rule of thumb: EventBridge for business events (with schemas), SNS for high-volume pub-sub.
- **EventBridge Pipes** collapses a common pipeline (source → filter → transform → target) into one construct. Good for simple passthroughs; still use Step Functions for anything branching.

---

## 4. Scenario 3: Stream Processing (Kinesis / DynamoDB Streams / MSK)

### Pattern

Kinesis shard → Lambda consumer processes records in order per shard → writes to sink.

```
Producer → Kinesis (N shards) → Lambda (1 consumer per shard) → S3/DDB/...
```

### Scalability

- **Shard count = parallelism**. 1000 records/sec/shard ingest, 2 MB/sec/shard consume (enhanced fan-out doubles this).
- **Parallelization factor**: 1–10 Lambdas per shard, preserving per-partition-key ordering inside a shard. This is the main lever for scaling consumers without resharding.
- **Enhanced fan-out**: dedicated throughput per consumer (2 MB/sec/shard), avoids consumer contention. ~$0.015/shard-hour extra.

### Corner cases

- **Per-shard sequential**: Lambda processes one batch per shard at a time. A slow record blocks everything on that shard.
  - Use `BisectBatchOnFunctionError` + `MaximumRetryAttempts` + DLQ. Tolerates poison pills without stalling the shard forever.
  - `MaximumRecordAgeInSeconds` rejects records older than N, preventing infinite catch-up on a bad consumer.
- **Hot shard**: traffic skewed to one partition key.
  - Re-shard with a composite partition key (append a random suffix). Accept loss of strict ordering.
- **DynamoDB Streams quota**: 2 readers per stream. Beyond that: fan out via Lambda → SNS → multiple downstream, or use Kinesis Data Streams for DynamoDB.
- **Iterator age alarm**: the P0 metric for streaming. If `IteratorAge` is rising, your consumer can't keep up → scale consumer or reshard producer.
- **At-least-once delivery**: every consumer must be idempotent. Use a record checkpoint in DDB keyed by `(shard_id, sequence_number)`.
- **Re-drive after incident**: consumer was down 30 min; restarting causes Lambda to catch up, potentially 30× spike to the sink.
  - Rate limit at the consumer (sleep between batches, or use reserved concurrency to cap). Don't overload your DB for "catching up" faster than its capacity.
- **Error handler splitting batches**: with `BisectBatchOnFunctionError`, Lambda halves the batch and retries. Can expand retries exponentially. Combine with max attempts + DLQ.

---

## 5. Scenario 4: Long-Running Workflows (Order, Saga)

### Pattern: distributed saga

Order placement spans inventory, payment, fulfillment, shipping. Each has its own service; if any fails, compensating actions must unwind prior steps.

```
[Reserve Inventory] → [Charge Payment] → [Create Shipment] → [Notify Customer]
       ↓ fail             ↓ fail               ↓ fail
 (no compensation)   [Release Inventory]  [Refund Payment] + [Release Inventory]
```

### Implementation

- **Step Functions Standard workflow**: durable, exact-once semantics, 1-year timeout.
- **Each Task** is either a Lambda invoke, an SDK integration (direct to DynamoDB, SNS, SQS), or an HTTP call via EventBridge/API destinations.
- **Catch blocks** trigger compensating branches.
- **Saga pattern**: maintain a "compensations" list as state progresses; on failure, iterate backwards.

### Why Step Functions over orchestrating in code

- **Durability for free**: every transition persisted. Lambda crash doesn't lose progress.
- **Visibility**: execution history is the debugger — click-through state timeline.
- **Versioned workflows**: new executions use new definition; running executions finish on old. No in-place edits to long-running logic.
- **Native retries + backoff** per task without handwritten code.
- **Task tokens** for async callbacks (payment provider webhooks) — no polling.

### Corner cases

- **25,000 history event quota**: a workflow with a big loop + many retries blows through this. Execution fails with `ExecutionHistoryLimitExceeded`.
  - Use **nested workflows**: orchestrator invokes child Step Functions for each sub-saga. Child history resets the count.
  - Or switch to Express for high-iteration sub-workflows.
- **256 KB state payload**: passing huge items between states fails.
  - Never pass raw blobs through state. Write to S3 / DynamoDB, pass only the pointer.
  - `ResultSelector` + `ResultPath` filter outputs aggressively before the next state sees them.
- **Non-idempotent compensation**: refund already processed from a prior failed compensation.
  - Compensation handlers must be idempotent — keyed on `(saga_id, step)`. Record completion in a durable store.
- **Task token leaked / lost**: if the code that receives the token crashes before calling `SendTaskSuccess/Failure`, the token can be orphaned.
  - `HeartbeatSeconds` + `TimeoutSeconds` on the task — Step Functions fails the task if heartbeat stops.
- **Rolling workflow changes**: you redeploy the workflow, but 10K running executions are on the old version.
  - Step Functions pins each execution to its definition at start. Rollouts are additive: deploy new version, start new executions on it, old ones finish.
  - For urgent bug fixes in a running execution: stop + restart from a checkpoint. Design state schema for resumability.
- **Lambda timeout < workflow expectation**: a task Lambda times out at 15 min on a cold DB → Step Functions marks task failed and retries.
  - Always set Step Functions task `TimeoutSeconds` ≤ Lambda timeout minus buffer. Set `HeartbeatSeconds` for long-running tasks so stuck ones fail fast.
- **Waiting on external system forever**: the callback never comes.
  - `TimeoutSeconds` on `waitForTaskToken`. After timeout, execute a compensation path.

---

## 6. Scenario 5: Massive Parallel Processing (Distributed Map)

### Pattern

Process 10M S3 objects through a common pipeline. Each item independent. Need throughput, not ordering.

### Old way (Inline Map)
- Up to 40 concurrent iterations. Hits history quota fast. Can't do millions.

### New way: Distributed Map (since 2022)
- Up to 10,000 concurrent child executions, each a full Step Functions sub-execution (its own history).
- Source: S3 objects, S3 inventory report, CSV, JSON Lines.
- Per-item failure tolerance: `ToleratedFailurePercentage` / `ToleratedFailureCount`.
- Results aggregated back to S3.

### Corner cases

- **10K downstream concurrency is a hammer**: if your Map's child invokes a Lambda that calls RDS, you'll nuke the DB.
  - `MaxConcurrency` on the Map caps concurrent children. Start at 100, raise with DB capacity.
- **Cost**: 10M child executions × Express ≈ acceptable; × Standard ≈ $250K. Pick Express for Map children unless you need Standard semantics per item.
- **Input data ordering**: Map doesn't guarantee order. If you need ordered side effects, you need another mechanism (sequencing in the sink).
- **Partial failures**: 9.99M succeeded, 10K failed. Without tolerance config, the whole Map fails.
  - Set `ToleratedFailurePercentage: 1` (or similar). Failed items listed in the result manifest for follow-up.
- **Input too large for Map dataset**: Distributed Map supports reading directly from S3 for datasets > payload limits.

---

## 7. Scenario 6: Human-in-the-Loop Workflows (Approval, Callback)

### Pattern

Loan application: validate → auto-decision or → route to human reviewer → [wait up to 72h for approve/reject] → finalize.

### Implementation: `.waitForTaskToken`

```
Task State (PublishToSNS.waitForTaskToken):
  - Publishes message with task_token to SNS (email, UI, queue)
  - Execution pauses, no cost accrued while waiting
  - Reviewer clicks approve → backend calls SendTaskSuccess(token, payload)
  - Workflow resumes with reviewer's payload
```

### Corner cases

- **Token forgotten in transit**: reviewer's email sat in spam; reviewer goes on vacation.
  - `TimeoutSeconds` on the task (72h + grace). On timeout, auto-escalate (catch → escalation path).
- **Duplicate approval**: reviewer clicks twice / two reviewers click before UI updates.
  - Use task tokens that are single-use (Step Functions enforces this); second call errors. Surface "already decided" cleanly.
- **Reviewer approves but backend crashes before `SendTaskSuccess`**: token now dangling.
  - Store `(approval_id → task_token)` mapping durably at the moment of creation, not on approval. Reviewer's approval is persisted first; a background reconciler replays `SendTaskSuccess` if it hasn't happened.
- **Security of the token**: anyone with the token can resume the workflow as "approved".
  - Never put tokens in URLs or emails directly. Map from a short, authenticated, single-use approval ID → token, stored server-side. User follows link to your auth gate, which produces the `SendTaskSuccess` call.
- **Payment webhook pattern**: payment provider might retry webhooks → same token called twice. Idempotent on your side (first call wins; subsequent calls get 200).

---

## 8. Scenario 7: File / Media Processing Pipeline

### Pattern

User uploads video to S3 → triggers processing:
1. Extract metadata + thumbnail
2. Transcode to multiple bitrates (fanned out in parallel)
3. Run content moderation
4. Generate captions
5. Publish to CDN

### Implementation

- **Trigger**: S3 event → EventBridge (not direct Lambda) for better fan-out and filtering.
- **Orchestration**: Step Functions Standard workflow per upload.
- **Transcoding**: not Lambda (15-min limit + not GPU-bound on Lambda). Step Functions invokes MediaConvert / Elemental via SDK integration.
- **Moderation**: Rekognition video analysis via SDK integration; `.sync` pattern waits for result.
- **CPU-bound thumbnail extraction**: fits in Lambda (short jobs); use 3 GB memory for speed.

### Corner cases

- **Direct S3 → Lambda scaling issue**: bulk upload of 100K files triggers 100K Lambdas instantly, melts downstream.
  - S3 → SQS → Lambda with reserved concurrency. Or S3 inventory + Distributed Map for bulk.
- **Large files exceed Lambda `/tmp`**: video file is 5 GB.
  - Never download to `/tmp`; stream-process from S3 or delegate to MediaConvert/Batch.
- **S3 eventual consistency on cross-region replication**: processor in us-east-2 triggers before file is replicated from us-east-1.
  - Process in the region where the object lives. If cross-region, confirm existence with `HEAD` or use replication-completion events.
- **Duplicate S3 events**: S3 may deliver event twice (at-least-once).
  - Idempotent processor keyed on S3 object version ID.
- **Missed S3 event**: rare but possible. Reconciliation pipeline: periodically list bucket, compare with processed manifest, enqueue misses.
- **Overwrite-in-place invalidates cached derivatives**: user uploads new version of same key.
  - Enable S3 versioning; process per version ID; CDN cache-key includes version.

---

## 9. Scenario 8: ETL & Analytics Pipelines

### Pattern

Nightly: ingest CSV/Parquet from vendors → validate → transform → load to warehouse → publish metrics.

### Choice: Lambda + Step Functions vs Glue / EMR / Airflow

- **Lambda + SFN**: small-to-medium data (< 100 GB/run), latency-sensitive, sparse schedule. Serverless cost advantage.
- **Glue / EMR / Airflow**: heavy Spark transforms, terabytes, complex DAGs with Python scripts.

The line shifts every year; often teams use SFN as the orchestrator and Glue / EMR as compute steps invoked by SFN.

### Pattern

```
SFN Standard
├── Pass: Collect run context
├── Task: StartGlueJob (sync) — extract
├── Choice: if record count = 0 → Succeed
├── Distributed Map:
│    for each partition:
│        ├── Lambda: validate
│        └── Task: StartGlueJob — transform
├── Task: StartRedshiftQuery — load
├── Parallel:
│    ├── Task: UpdateDashboardCache (Lambda)
│    └── Task: PublishMetrics (Lambda)
└── Task: SNS.Publish (notify on complete)
```

### Corner cases

- **Overnight runs colliding with business hours**: if job runs long (backfill), it's still going when users arrive.
  - Alarm on run duration > P95 × 1.5. Hard limit via SFN `TimeoutSeconds`.
- **Idempotent reruns**: same file processed twice.
  - Staging table with upsert keys or `INSERT ... ON CONFLICT`. Never append blindly.
- **Late-arriving data**: partition boundary skew.
  - Reprocess windows: every nightly run also re-validates last 48h of data.
- **Vendor schema change breaks parser**: silent data drift.
  - Schema validation as a step; fail loudly. Version-pin expected schemas; alert on drift.
- **Step Functions history quota on Map children**: 10M-row transform.
  - Distributed Map with Express children. The 25K history limit is per execution, so each child starts fresh.
- **Cost blowout on Distributed Map with Standard workflows**: see §6.

---

## 10. Scenario 9: ML Inference & Batch Scoring

### Online inference
- **Small models** (< 500 MB, CPU-based): Lambda is great. Container image for scikit-learn / lightweight PyTorch.
- **Medium models** (GPU needed): Lambda can't do GPU. Use SageMaker Serverless or Fargate behind API Gateway.
- **Cold start on inference Lambda**: SnapStart (Java) or provisioned concurrency (others).

### Batch scoring pipeline (Step Functions)

```
SFN
├── Query eligible population (Lambda → DB)
├── Distributed Map (Express children):
│    ├── Lambda: score single record
│    └── Task: DynamoDB.PutItem score
└── Task: Publish batch summary
```

### Corner cases

- **Model weights in deployment package** bloat cold start.
  - Use container image (lazy-load weights from EFS-mounted storage or S3, cache in `/tmp`).
- **SageMaker Serverless has its own cold start** (5–60s for first invoke).
  - Provisioned concurrency ($). Evaluate whether latency matters for your UX.
- **Model version drift across concurrent Lambdas**: some containers still running old weights after update.
  - Version model artifact; Lambda fetches the active version from Parameter Store on init; alias update triggers reload by forcing new containers (publish new Lambda version).
- **Prediction cost at scale**: batch with Distributed Map will invoke 10M × container cold-start overhead.
  - Larger batch size per child (process 1K records per child) amortizes the init cost.

---

## 11. Scenario 10: Scheduled / Cron Workflows

### Options

- **EventBridge Scheduler** (preferred): one-time or recurring, at-least-once, cron expressions, targets include Lambda and Step Functions directly. Replaces older `EventBridge Rules + schedule`.
- **Step Functions as the scheduler** isn't idiomatic; use EventBridge Scheduler → SFN for scheduled workflows.

### Corner cases

- **Daylight savings / timezone bugs**: old cron expressions assumed UTC; business expected America/New_York.
  - EventBridge Scheduler supports timezones in cron expressions. Use them explicitly.
- **Concurrent runs of a "should be singleton" schedule**: overlapping triggers due to retries or SFN lock miss.
  - Use a global idempotency key (e.g., `daily-etl-2026-04-22`) checked at the start of the workflow; exit if already running.
- **Missed executions on scheduler downtime**: rare, but at-least-once doesn't mean always-on-time.
  - Reconciliation: at startup, check "was yesterday's job run?" and catch up if not.
- **Bursts of scheduled jobs all firing at 00:00**: thundering herd.
  - Jitter schedules across a window; different schedules shouldn't all start on the top of the hour.

---

## 12. Scenario 11: Serverless SaaS Multi-Tenant Platform

### Pattern

Per-tenant isolation across many small customers. Lambda runs tenant code / data paths; Step Functions orchestrate tenant workflows.

### Isolation models

- **Pool (shared)**: one Lambda, tenant ID in request. Cheapest, lowest blast-radius isolation. Risk of noisy neighbor.
- **Silo (dedicated)**: one Lambda (or account) per tenant. Best isolation, highest cost + ops burden. For enterprise tiers.
- **Bridge (hybrid)**: pooled Lambda for small tenants, dedicated for big. Most common.

### Corner cases

- **Reserved concurrency per tenant in pooled model**: impossible natively. A runaway tenant consumes all Lambda concurrency.
  - Per-tenant SQS + `MaximumConcurrency` on event source mapping = per-tenant concurrency cap.
  - Or route big tenants to their own Lambda alias with reserved concurrency.
- **Cross-tenant data leak via warm containers**: `/tmp` or process globals retain prior tenant's data.
  - Never cache tenant data in `/tmp` or module-level variables. Use scoped, tenant-keyed caches with TTL.
- **IAM per tenant**: dynamic policies are painful. 
  - Attribute-based access control (ABAC): IAM policy uses `${aws:PrincipalTag/TenantId}`. Lambda assumes role with tenant tag. Scales without policy explosion.
- **Per-tenant billing / observability**: can't tell how much tenant X cost.
  - Tag every resource with `TenantId`; emit metrics with tenant dimension (bounded cardinality).
- **Deployment across many tenants**: schema migration, config rollout.
  - Use Step Functions Distributed Map to run per-tenant operations in parallel with concurrency cap.

---

## 13. Cross-Cutting Concerns

### Idempotency — mandatory, not optional

Lambda event sources are at-least-once. Every handler must be idempotent.

- **Powertools Idempotency** (Python/TS/Java): wraps handler, caches result in DDB keyed by idempotency key, returns cached response on replay.
- **Manual**: dedupe table in DDB with TTL. Write `(key, status=in_progress)` with conditional put; if exists, return existing. Key choices:
  - For API: `(user_id, client_request_id)` from header.
  - For SQS: `(queue, message_id)`.
  - For Kinesis: `(stream, shard, sequence_number)`.
  - For S3: `(bucket, key, version_id)`.
- **Idempotency keys must have TTL** (matches business retry window). Infinite storage is expensive and useless.

### DLQs & failure handling

- **Async Lambda invocation DLQ** (on the function config): captures events that failed all retries. Use SQS DLQ, not SNS (visibility into failed messages matters).
- **Event source mapping DLQ**: for SQS/Kinesis/DDB Streams. Separate DLQ per source.
- **Step Functions DLQ**: not a thing natively. Use `Catch` to write failed executions to SQS for triage.
- **DLQ redrive**: SQS supports native redrive (re-send all DLQ messages back to source queue). Use after fix. Cap redrive rate to avoid a new failure storm.

### Observability

- **Structured logs** with JSON; lambda-powertools Logger is the standard.
- **Correlation IDs** passed through every service hop. Propagate via `X-Correlation-Id` header or Step Functions `$$.Execution.Id`.
- **Metrics**: CloudWatch EMF (Embedded Metric Format) to emit metrics inline in logs; cheaper than PutMetricData per-call.
- **Tracing**: X-Ray or OpenTelemetry; SFN and Lambda both support tracing natively. Tail-based sampling for costs.
- **SFN observability is unique**: Execution history is the source of truth; Map Run observability shows child fan-out; CloudWatch Metrics for `ExecutionsFailed`, `ExecutionTime`, `ExecutionsThrottled`.

### Cost control

- **Right-size memory**: Lambda CPU scales with memory. Often 1024 MB runs 2× faster than 512 MB for the same cost. Use Lambda Power Tuning to find the sweet spot.
- **Graviton (arm64)**: 20% cheaper flat. Requires arm64-compatible deps.
- **Express over Standard** for high-volume simple workflows.
- **SDK integrations over Lambda** in SFN: direct integrations (DDB, SNS, SQS, StepFunctions, ECS, etc.) skip Lambda compute cost + latency. Often 50% cheaper for simple glue.
- **Provisioned concurrency only where measured**: it's a fixed cost 24/7. Unless TTFT matters, on-demand is cheaper.
- **Minimize state payload** in SFN: large payloads charged per transition.
- **Reserved concurrency != cost savings**: it caps, doesn't pre-pay.

### Security

- **Least privilege IAM per function**: auto-generated permissive roles are the #1 audit finding. Use IAM Access Analyzer to generate scoped policies.
- **Secrets**: Secrets Manager / Parameter Store. Lambda caches via extension (`AWS_LAMBDA_PARAMS_AND_SECRETS_EXTENSION`) — avoids fetching per invocation.
- **Signing**: Lambda code signing for supply-chain protection; Step Functions workflow policies.
- **VPC only when needed**: if function only hits AWS APIs, use VPC endpoints and keep Lambda out of VPC for faster cold starts.
- **VPC ENI exhaustion**: high-concurrency VPC Lambda can exhaust subnet IPs.
  - Use large subnets (/20+) dedicated to Lambda; ENI sharing across functions is automatic in modern Lambda.
- **Resource policies on SFN**: restrict `StartExecution` to specific principals; prevents privilege escalation via StepFunctions.

### Deployment

- **SAM / CDK / Terraform / Serverless Framework**: all fine; pick one and stick with it.
- **Versions + Aliases**: every deploy publishes a new version; alias (e.g., `live`) points to current. API Gateway stages point to alias.
- **Traffic shifting** with CodeDeploy: 10% → 50% → 100% with CloudWatch Alarm rollback. Blue/green for Lambda without downtime.
- **Infrastructure + workflow in same repo**: SFN ASL (JSON/YAML) lives next to Lambda code; deploy together.
- **Workflow versioning** in SFN: deploy new definition, new executions use new; running executions finish on old. No in-place mutation of running executions.
- **Pre-prod SFN testing**: Step Functions Local, plus `TestState` API (introduced 2023) to test single states against real AWS services.

---

## 14. Corner Cases & Anti-Patterns

### Cold starts

- **VPC + Java + large JAR** = 3-5s cold start. Solutions in priority:
  1. SnapStart (Java/Python/.NET) — 80% reduction.
  2. Exit VPC if possible.
  3. Provisioned concurrency for steady baseline.
  4. Smaller deployment package; lazy-load dependencies.
- **SigV4 init overhead**: building the SDK client is part of cold start. Do it at module level so container reuse amortizes it.
- **Invisible cold starts on scale-up**: traffic 2× → Lambda scales → half of requests hit new containers.
  - Monitor `Init Duration` as a percentile, not average. Provisioned concurrency for the baseline + on-demand for burst.

### Connection pooling

- **Classic mistake**: each Lambda invocation opens a new DB connection. 1000 concurrent Lambdas × 10 connections = 10K connections, RDS rejects.
  - **RDS Proxy** fronts connections, pool-shared across Lambdas.
  - **DynamoDB** or **HTTP-based stores** avoid this entirely; they're Lambda-native.
  - In-container connection reuse is fine for warm invokes, but concurrent cold starts still hammer the DB.

### Function URLs vs API Gateway

- **Function URLs**: free, simpler, no custom domain without CloudFront, limited auth, no rate limiting.
- **API Gateway REST**: full-featured, 10K RPS default, usage plans, API keys, expensive for high QPS.
- **API Gateway HTTP API**: 70% cheaper, less features, higher throughput. Usually the right choice for new APIs.
- **ALB → Lambda**: integrates with existing VPC infra; good when you need WAF + target groups. More infra than Gateway.

### Step Functions anti-patterns

- **Using Lambda to call AWS APIs inside SFN**: DDB PutItem via Lambda = 2× latency + Lambda cost. Use direct SDK integration.
- **Giant single workflow**: one SFN doing everything. 25K event limit + debuggability nightmare. Split into sub-workflows.
- **Imperative Lambda doing orchestration** because "SFN is complex": you'll reinvent retries, catch, parallel, visibility. Bite the bullet.
- **State payload accumulation**: each state appends to the payload; by state 20 it's 200 KB.
  - Aggressively filter with `OutputPath`, `ResultPath`, `ResultSelector`. Offload to S3 if persistence matters.

### SQS / Lambda nuances

- **Standard queue delivery is at-least-once**, not exactly once. Without idempotency, duplicates corrupt state.
- **Visibility timeout must exceed Lambda timeout** by at least 6× for safe retries.
- **Max 120K messages in-flight** per queue. Slow consumer → queue backs up → SQS rejects new messages.
  - Monitor `ApproximateAgeOfOldestMessage`; scale consumer concurrency.

### The "runaway Lambda" incident

A Lambda bug causes infinite loop / self-invocation. Charges pile up in minutes.

- **Account-level budget alarm** at daily thresholds.
- **Reserved concurrency = 0** is the kill switch — instantly disables a function.
- **Avoid self-invocation patterns** without termination condition; use SFN for retry orchestration.

### Throttling behavior

- **Sync invocation throttled**: caller gets 429, handles retry.
- **Async invocation throttled**: Lambda retries internally for 6 hours (!) with exponential backoff.
- **Event source throttled**: SQS re-delivers after visibility timeout; Kinesis stalls the shard.

Understand each: the "same" throttle feels different at each integration.

---

## 15. When NOT to Use Lambda / Step Functions

### Not Lambda

- **Steady-state high QPS** where EC2/Fargate is 3–10× cheaper at full utilization.
- **Sub-10ms latency** requirements — cold starts + invoke overhead + handler init kill this.
- **> 15-minute execution** — Step Functions, Fargate, or Batch.
- **Heavy GPU** — SageMaker, ECS, EKS.
- **Long-lived connections** (WebSocket server, gRPC streaming) — API Gateway WebSocket is a workaround but not ideal.

### Not Step Functions

- **Simple linear flows with 2-3 steps**: a single Lambda with sequential code is simpler and cheaper.
- **Very high volume simple events**: EventBridge Pipes + Lambda is often simpler than SFN Express.
- **Workflows requiring rich programmatic logic**: Step Functions ASL can express loops/conditions but is clunky. Consider Temporal / Airflow for complex DAGs with heavy code.
- **Workflows where history > 25K events with no natural shard boundary**: Express Workflows or different orchestrator.

---

## 16. Decision Matrix

| Problem | Service combination |
|---|---|
| Sync API, low latency | API Gateway HTTP API + Lambda (+ provisioned concurrency / SnapStart) |
| Fire-and-forget processing | SNS → SQS → Lambda with DLQ |
| Fan-out to many consumers | SNS → multiple SQS → Lambda per consumer |
| Event-driven business logic | EventBridge (schemas, filtering) → consumers |
| Stream processing | Kinesis / MSK → Lambda with parallelization factor + partial batch response |
| DynamoDB change data capture | DynamoDB Streams → Lambda (+ idempotency) or → Kinesis for more consumers |
| Multi-step business workflow | Step Functions Standard with SDK integrations + Catch |
| Saga / distributed transaction | Step Functions Standard + compensation branches |
| Millions of parallel jobs | Step Functions Distributed Map (Express children) |
| Human approval | Step Functions + `.waitForTaskToken` + per-approval ID → token mapping |
| Scheduled cron | EventBridge Scheduler → Lambda or SFN |
| File processing pipeline | S3 event → EventBridge → SFN (orchestrate) + MediaConvert/Lambda (compute) |
| ETL | SFN + Glue jobs + Distributed Map for partitioned transforms |
| Batch ML scoring | SFN Distributed Map + Lambda (CPU) or SageMaker Batch Transform |
| Low-latency inference | SageMaker Serverless / Fargate (not Lambda for GPU) |
| Multi-tenant SaaS | Pooled Lambda + per-tenant SQS + ABAC IAM; silo tier via dedicated Lambdas |
| Need to kill a runaway function | Reserved concurrency = 0 |

---

## 17. The Hard-Won Truths

1. **Idempotency is a property of your handler, not the service.** Every Lambda you ship must tolerate replay.
2. **Cold start is a design variable, not a metric to ignore.** Measure Init Duration P99; budget for it; use SnapStart / provisioned concurrency where it matters.
3. **Use SDK integrations in Step Functions ruthlessly.** If Lambda is only making an AWS API call, cut it out.
4. **Pass pointers, not payloads, through Step Functions.** 256 KB sneaks up on you.
5. **DLQ everything asynchronous.** A lost message is a lost customer.
6. **Set `MaximumConcurrency` on every event source mapping** that feeds a non-infinitely-scalable downstream (RDS, external API).
7. **Standard vs Express is a 10-100× cost choice.** Don't pick without doing the math.
8. **Reserved concurrency = 0 is your incident kill switch.** Document it in runbooks.
9. **Lambda Power Tuning is free money** — run it quarterly on your top functions.
10. **Graviton + right-sized memory** is the cheapest 20–40% win in any serverless budget.