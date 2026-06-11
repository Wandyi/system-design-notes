# Staff-Level AWS Lambda Design Patterns

The patterns below are organized by the forcing function — what physics of Lambda (limits, billing, concurrency model) makes the pattern necessary. Staff-level thinking is less about "what's the pattern" and more about "why does this pattern exist, and where does it break."

---

## 0. The Three Mental Models You Reason From

Every Lambda design decision collapses to one of these.

| Model | What it actually means | Pattern consequence |
|---|---|---|
| **Lambda is a function, not a server** | No long-lived state, no graceful shutdown guarantees, container reuse is a *cache* not a contract | Push state out (DDB, Redis, S3); make init lazy & idempotent |
| **You pay for wall-clock × memory** | Sleeping/blocking on I/O burns money the same as compute | Never `time.Sleep` in Lambda; never long-poll a queue from a Lambda; never call another Lambda synchronously and wait |
| **Concurrency is the unit of scale, not RPS** | 1000 RPS × 10ms = 10 concurrent. 1000 RPS × 1s = 1000 concurrent (= account cap by default) | Latency reduction *is* a scaling lever; reserved/provisioned concurrency is a capacity decision |

If you internalize nothing else, internalize the second row. Most Lambda anti-patterns are someone treating it like a server.

---

## 1. Invocation Model Patterns (Pick Your Failure Mode)

Lambda has three invocation models and they fail very differently. Choose based on **what failure you can tolerate**.

```
┌─────────────┬──────────────────┬─────────────────┬──────────────────────┐
│ Model       │ Caller waits?    │ Retry semantics │ Failure mode         │
├─────────────┼──────────────────┼─────────────────┼──────────────────────┤
│ Sync        │ Yes (≤ 29s API)  │ Caller's job    │ 5xx to caller        │
│ Async       │ No (returns 202) │ 2 auto retries  │ Silent → DLQ/onFail  │
│ Poll-based  │ N/A (poller)     │ Until visibility│ Stuck messages       │
│ (SQS/Kinesis│                  │ timeout / age   │ DLQ or stream halt   │
│  /DDB)      │                  │                 │                      │
└─────────────┴──────────────────┴─────────────────┴──────────────────────┘
```

**Staff-level corollaries:**

- **Never use sync for >1s work behind API Gateway.** API GW has a 29s hard cap and you pay for the Lambda wall-clock holding the connection. Use the **Accepted-202 pattern**: return a job ID immediately, process async, expose `GET /jobs/{id}`.
- **Async invocation silently truncates >256 KB payloads.** Always pair async with a DLQ + `onFailure` destination, and validate size at the edge.
- **Poll-based invocations don't count against your function's invocation rate the same way.** Lambda's service polls SQS/Kinesis and batches. You don't pay per message, you pay per batch.

---

## 2. Event-Source Fan-Out & Fan-In Patterns

This is where most architecture interview questions live.

### 2a. SNS Fan-Out (broadcast, low-coupling)

```
Producer → SNS Topic ──┬─► SQS A → Lambda A
                       ├─► SQS B → Lambda B
                       └─► Firehose → S3
```

- Always put **SQS between SNS and Lambda**, never SNS → Lambda directly at scale. Reason: Lambda async retry is only 2 attempts with exponential backoff that you don't control; SQS gives you visibility timeout, redrive, and a real DLQ.
- One topic with **filter policies** > many topics. Cheaper, easier IAM.

### 2b. EventBridge (semantic routing)

Use EventBridge when **the producer doesn't know its consumers**. SNS is for known broadcast; EventBridge is for content-based routing across teams/domains. Cost is ~$1/M events vs SNS ~$0.50/M — pay the premium for the routing decoupling.

### 2c. SQS → Lambda (the workhorse pattern)

The most important knobs:

| Knob | Default | Why you change it |
|---|---|---|
| Batch size | 10 | Larger = fewer invocations = lower cost; but one poison message kills the whole batch |
| Batch window | 0s | Wait up to N seconds to fill a batch (cost vs latency) |
| `ReportBatchItemFailures` | off | **Always turn this on.** Let Lambda return per-message failures so a single bad message doesn't reprocess the whole batch |
| Maximum concurrency (event-source mapping) | unbounded | Critical for protecting downstream — see §4 |
| Visibility timeout | 30s | Must be ≥ 6× Lambda timeout (AWS guidance) |

### 2d. Fan-In: aggregating from many sources

The naive fan-in mistake: many Lambdas writing to one row in DynamoDB → contention, throttling.

Staff fix: **fan-in via Kinesis Data Streams with a partition key matching the aggregation key**, then a single Lambda per shard does ordered aggregation. Or write to a per-source sub-record and aggregate on read.

---

## 3. Stream Processing Patterns (Kinesis, DDB Streams, MSK)

The hidden trap: **stream Lambdas are ordered per shard**. One poison message blocks the shard until expiration (up to 7 days). Three patterns to manage this:

1. **`BisectBatchOnFunctionError`** — Lambda halves the batch on failure to isolate the poison message. Cheap, built-in. Always on.
2. **`MaximumRetryAttempts` + `OnFailure` destination** — bound retries, divert poison to SQS DLQ, **don't block the shard**.
3. **`ParallelizationFactor`** (1–10 per shard) — multiple concurrent Lambda invocations per shard, but **ordering is preserved per partition key**. Free throughput multiplier when ordering allows.

**Tumbling windows** (`TumblingWindowInSeconds`) are the lesser-known feature for stateful streaming aggregation without external state. Lambda keeps an in-flight state object across batches within the window. Use for top-K, rolling sums, etc. — avoids dragging in Redis for simple windowing.

---

## 4. Concurrency Control Patterns (the failure-isolation tier)

Staff candidates get this question: *"Your Lambda hammers an upstream Postgres with 1000 connections during a spike. Fix it."* There are four levers — know all four:

| Lever | Where it lives | What it bounds |
|---|---|---|
| **Reserved concurrency** | Function setting | Hard cap on simultaneous executions. Throttles excess invocations. |
| **Provisioned concurrency** | Function/version | Pre-warmed instances (paid hourly) — eliminates cold start, doesn't cap |
| **Event-source mapping max concurrency** | SQS trigger | Caps SQS-driven concurrency without throttling other invokers |
| **RDS Proxy / per-region pool** | Architectural | Multiplexes Lambda connections onto a small DB pool |

The right answer is usually **two of these together**: reserved concurrency to protect downstream + RDS Proxy to multiplex.

**Subtlety:** reserved concurrency is *also* an isolation mechanism. Setting it to 100 on Function A doesn't just cap A — it reserves 100 slots, so a runaway Function B can't starve A out of the account's 1000-concurrent budget.

---

## 5. Cold Start Mitigation Patterns

In priority order — apply the simplest one that works:

1. **Right-size memory.** CPU scales linearly with memory. A 256 MB Lambda doing JSON parsing may be slower *and more expensive* than a 1024 MB one. Use Lambda Power Tuning.
2. **Stay out of VPC unless you need a private resource.** VPC ENI attach adds ~100ms (much better than the old 10s, but still meaningful). Use VPC endpoints for S3/DDB to keep traffic private without putting the Lambda in VPC.
3. **Lazy-init inside the handler for non-critical deps.** Init code runs *before* the timer starts in some runtimes, but counts in others — read your runtime docs.
4. **SnapStart for Java/.NET/Python.** Snapshots a warm process — ~80% cold start reduction. Free.
5. **Provisioned concurrency.** Last resort because it's hourly billing — you're paying for a server you said you didn't want.
6. **Container images >250MB** have *worse* cold starts than zip until first warm hit. Use AWS-optimized base images.

**Anti-pattern:** "ping the Lambda every 5 minutes to keep it warm." It only warms one container; doesn't help concurrent traffic; and it's a code smell that you should be using provisioned concurrency.

---

## 6. State Patterns (Lambda is Stateless — Live With It)

### 6a. The execution context cache

`/tmp` (up to 10 GB), globals, and connection objects persist across invocations on the same container. Use for:

- **DB connection reuse** — initialize the client *outside* the handler.
- **Secret caching** — fetch once, cache 5–15 min with a TTL (never indefinitely; rotations break you).
- **Read-only data** like ML model weights, GeoIP tables — load once at init.

**Trap:** if you write to a *mutable global*, you've created a non-deterministic bug. Container reuse is unpredictable; treat any mutable global as a race condition.

### 6b. Distributed state — choose by access pattern

| State shape | Store | Why |
|---|---|---|
| Per-request small KV (1ms reads) | DynamoDB | Single-digit ms, scales infinitely, IAM-native |
| Hot counters / locks | DAX or Redis (ElastiCache/MemoryDB) | DDB write contention on hot keys; Redis SETNX for locks |
| Large blobs | S3 with presigned URLs | Don't pass >100 KB through Lambda payloads |
| Workflow state | Step Functions | Don't reinvent a state machine in Lambda + DDB |

### 6c. Idempotency

Async retries + at-least-once delivery means **every Lambda processing external events must be idempotent**. The canonical pattern:

```
1. Receive event with unique key (message ID, idempotency key header)
2. Conditional PutItem to DynamoDB { pk: key, status: IN_PROGRESS } 
   with attribute_not_exists(pk) — fails if duplicate
3. Do work
4. UpdateItem to COMPLETED with TTL (e.g., 24h)
```

Use the **Powertools for Lambda** `@idempotent` decorator (Python/TS/Java) — it implements exactly this pattern. Don't roll your own.

---

## 7. Orchestration Patterns (When You Need More Than One Lambda)

This is the **most-misused area at staff level**. The decision tree:

```
Need to call N Lambdas in sequence?
├── < 5 steps, total < 15 min, no human waits? → ONE Lambda. Stop overengineering.
├── Branching, parallel, retries, > 15 min? → Step Functions Standard
├── High-volume (>1M/day), short, idempotent? → Step Functions Express
└── Need callbacks/human approval/long waits? → Step Functions w/ Task Token
```

### Anti-pattern: Lambda chaining (Lambda invokes Lambda invokes Lambda)

- You pay 2× for the period both are running.
- Errors are opaque (each Lambda has its own retry).
- No visibility into the chain.

**Use Step Functions, EventBridge, or a queue instead.** Lambda invoking Lambda is a code smell unless it's a *very* short utility call.

### Saga pattern

Distributed transaction across multiple services with compensating actions. Step Functions is the canonical implementation — each step has a `Catch` to its compensating Lambda. Beats trying to coordinate it in app code.

### Distributed Map (Step Functions)

For embarrassingly parallel work over S3 buckets / massive datasets — up to 10,000 child executions. Replaces the old "Lambda fans out to Lambdas via SQS" pattern with native orchestration, retries, and result aggregation.

---

## 8. Data Movement Patterns

### Pattern: S3 pointers, not payloads

Lambda payload limits (6 MB sync, 256 KB async, 256 KB between Step Function states) bite hardest in data pipelines. Pattern:

```
Producer → uploads blob to S3 → emits event with {bucket, key, etag}
Consumer Lambda → reads from S3 → processes → writes result back to S3
```

Always pass the pointer, not the data. Even when the data fits, this gives you:
- Free retry (event has the same pointer)
- Free debugging (raw input preserved)
- Free replay (re-trigger from S3 event log)

### Pattern: S3 event-driven pipeline with idempotency keys

S3 → SNS → SQS → Lambda. **Not** S3 → Lambda directly, because:
- S3 → Lambda has no batching, no DLQ control, no replay.
- One bad object can trigger a Lambda retry storm.

Add an idempotency record keyed by `{bucket}/{key}/{etag}`.

### Pattern: Response streaming for large outputs

Lambda Response Streaming (via Function URLs or invoke mode) lifts the 6 MB response cap to 20 MB and lets you start sending bytes before the function finishes. Use for LLM streaming, large file downloads, server-sent events. Saves both latency and the "double the memory to buffer" cost.

---

## 9. Observability Patterns

### Structured logging — non-negotiable at scale

JSON to stdout, with correlation IDs propagated from the caller. CloudWatch Logs Insights queries depend on JSON structure. Use Powertools `Logger`.

### Correlation IDs across async boundaries

When SQS/SNS/EventBridge sit between functions, the X-Ray trace breaks unless you propagate `traceparent` headers in the message attributes. Lambda Powertools `Tracer` does this — wire it in.

### Embedded Metric Format (EMF)

Emit business metrics directly in log lines using CloudWatch EMF. No extra API call (= no extra latency, no extra cost). Pattern:

```json
{"_aws": {"CloudWatchMetrics": [{"Namespace":"orders","Metrics":[{"Name":"placed"}]}]}, "placed": 1}
```

You get a CloudWatch metric for free. Beats the anti-pattern of calling `PutMetricData` per invocation.

### The three alarms every prod Lambda needs

1. **Error rate** > X% over 5 min
2. **Duration p99** > Y% of timeout (you're about to timeout)
3. **Throttles** > 0 (you're hitting a concurrency limit you didn't know about)

---

## 10. Security Patterns

- **One IAM role per function.** Never share a "lambda-execution-role". Least-privilege is per-function.
- **Resource-based policies** to grant invoke rights — never put an access key into Lambda env vars.
- **Secrets via Secrets Manager / SSM Parameter Store**, fetched at init and cached. Use Lambda extensions (Parameters and Secrets Lambda Extension) to avoid in-handler fetch latency.
- **Environment variables are visible in the console**, so encrypt sensitive ones with a CMK or just don't put them there.
- **VPC endpoints (Gateway for S3/DDB, Interface for everything else)** so a VPC Lambda doesn't need a NAT gateway — NAT is the #1 surprise bill in serverless setups (~$32/mo per AZ + data).

---

## 11. Cost Patterns (where staff thinking shows)

Lambda billing = `requests + GB-seconds + provisioned-concurrency-hours`. The leverage points:

| Lever | When it pays off |
|---|---|
| **Right-size memory** (Power Tuning) | Always run this. 30–50% savings is typical. |
| **ARM/Graviton** | ~20% cheaper, ~20% faster for most workloads. Free win if your runtime supports it. |
| **Drop sync invocations of internal Lambdas** | You're paying 2× for both functions' wall-clock. Use EventBridge or SQS. |
| **Batch SQS aggressively** | 10× batch = 10× fewer invocations = 10× fewer billed requests. |
| **Tiered Step Functions** | Express for hot path (millions of executions), Standard for low-volume + audit-required. Mixing is fine. |
| **Reduce log volume** | At very high scale, CloudWatch Logs ingestion ($0.50/GB) exceeds Lambda compute cost. Log at INFO, ship to S3 via subscription filter, query with Athena. |

**The "is Lambda still the right answer?" inflection point:** when a function runs >50% of the day at steady load, Fargate or EC2 is cheaper. Lambda's premium is paid for *idle*. Burst-y or low-volume → Lambda wins. Steady high load → containers.

---

## 12. The Anti-Patterns (Staff Should Reject These on Sight)

1. **Lambda calling Lambda synchronously and waiting.** Double-billed; opaque errors. Use queues/Step Functions.
2. **A Lambda that runs >5 minutes regularly.** You're paying for sleep. Probably wants Step Functions or Fargate.
3. **Polling a queue from inside a Lambda.** The event-source mapping does this for free, properly.
4. **Holding a DB connection across invocations without RDS Proxy** at scale. Connection exhaustion is the #1 outage cause for serverless+RDBMS.
5. **Pinging your own Lambda to keep it warm.** Use provisioned concurrency or SnapStart.
6. **Catching all exceptions and returning 200.** Hides failures from Lambda's retry/DLQ machinery — you've blinded the platform.
7. **One giant Lambda doing many event types.** Mixed concerns + harder IAM scoping + worse cold starts (bigger package). Split per event source.
8. **Step Functions Standard for high-throughput trivial workflows.** ~$25/M state transitions × millions = real money. Express is 10–100× cheaper.
9. **Logging the full event at INFO.** PII risk + log cost. Log keys, not payloads.
10. **Using Lambda as a cron with EventBridge** for jobs that take > 10 min or have dependencies. Step Functions Schedule is the right tool.

---

## 13. The "When NOT to Use Lambda" Heuristic

Staff-level answer is always *"and here's where this pattern stops working"*:

- **Sustained CPU-bound workloads** > 10 min — Fargate/Batch.
- **Workloads needing >10 GB memory or GPU** — Fargate/EC2/SageMaker.
- **Sub-10ms p99 latency requirements** at low traffic — cold starts kill you; use a warm container (Fargate, ECS).
- **Workloads with stable, high RPS where idle cost is zero relevance** — ECS/EKS is cheaper at steady state.
- **Stateful protocols** (WebSocket fan-out at high scale, persistent gRPC streams) — Lambda can do API GW WebSockets but it's awkward beyond modest scale.
- **Strict ordering across high-cardinality keys** — Lambda+Kinesis works only as well as your partition key distribution.

---

## TL;DR Heuristics for an Interview

1. *"What's the failure mode?"* — every pattern choice should start here.
2. *"Where's the idempotency boundary?"* — at-least-once is the default; design for it.
3. *"Who owns the concurrency budget?"* — reserved + event-source mapping limits + downstream protection.
4. *"Are you paying for wall-clock you don't need?"* — the canonical Lambda cost smell.
5. *"Does this need to be Lambda at all?"* — the strongest staff signal is willingness to say "no."
