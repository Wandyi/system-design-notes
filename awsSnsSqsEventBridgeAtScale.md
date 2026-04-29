# AWS SNS, SQS, EventBridge — Realistic Scenarios at Staff-Engineer Depth

> A practical reference for the three pillars of AWS messaging: when each is right, when each is wrong, and how they combine for production-grade pub/sub, fanout, queueing, and event-driven architectures at MAANG-level scale (millions of events/sec, multi-region, multi-account, SLA-bound). Every scenario is one I've shipped, broken, fixed, or watched a peer break — written for staff engineers who are past "what is a queue" and need decision-grade depth.

> Companion to `awsS3ScenariosAtScale.md`, `lambdaStepFunctionsAtScale.md`, `eventPlatformsAtScale.md`. Together they describe the AWS-native event/serverless surface.

---

## 0. The Staff-Level Frame

When a junior engineer hits an async problem, they pick whichever of the three they know. When a senior engineer hits it, they pick the one with the right primitive. A staff engineer:

1. **Picks the primitive that matches the *correctness contract*** — at-most-once, at-least-once, exactly-once-processing, ordered, ordered-per-key, fanout, point-to-point.
2. **Plans for the operational lifecycle** — DLQs, replay, schema evolution, multi-region, observability, cost.
3. **Knows the limits and the failure modes** — visibility timeout pathology, FIFO group hot-spotting, EventBridge throttling, SNS HTTP retry storms.
4. **Sees the cost shape clearly** — per-event cost, fanout multiplier, cross-account/cross-region overhead.
5. **Designs for evolution** — a 5-year-old architecture has had 50 schema changes; the messaging layer must absorb them without every consumer breaking.

The three services overlap heavily. Choose well; the wrong primitive at scale costs months of re-architecting.

---

## 1. Mental Models — SNS vs SQS vs EventBridge

### 1.1 The one-line rule

```
SQS         = a durable buffer between a producer and ONE consumer
              (or one consumer group). Pull-based.
SNS         = pub/sub fanout: one publish, many subscribers. Push-based.
EventBridge = SNS-with-filtering, schemas, archive, replay, cross-account,
              SaaS partner sources. Push-based, richer.
```

### 1.2 The combination patterns

```
                   ┌────► SQS (consumer A)
SNS ────► fanout ──┤
                   └────► SQS (consumer B)

This is the "SNS+SQS fanout" pattern — the canonical way to do pub/sub-with-buffering.
SNS pushes; each SQS queue absorbs the burst for its own consumer.

EventBridge ─► rule 1 ─► Lambda
            ├► rule 2 ─► SQS
            ├► rule 3 ─► Step Functions
            └► rule 4 ─► another EventBridge bus (cross-account)

EventBridge replaces SNS in modern stacks: same fanout, plus filtering by content,
schema registry, archive/replay, cross-account routing, SaaS sources.
```

### 1.3 When to use each

| Use case | Best fit |
|---|---|
| Producer wants buffer; one consumer | **SQS** |
| Many consumers want the same event | **SNS** or **EventBridge** |
| Consumers need filtering by content | **EventBridge** (or SNS message filter) |
| Cross-account event routing | **EventBridge** (native) |
| SaaS source ingestion (Stripe, Datadog, Zendesk) | **EventBridge partner buses** |
| Event archive + replay | **EventBridge** |
| Schema registry / discovery | **EventBridge** |
| Strict ordering per key | **SQS FIFO** or **Kinesis** |
| Mobile push notifications | **SNS** (only one with mobile push) |
| SMS / email push | **SNS** for SMS (also Pinpoint); **SES** for email; not EventBridge |
| Cron / one-shot scheduling | **EventBridge Scheduler** |
| DB / log → events without code | **EventBridge Pipes** |
| Streaming analytics with replay | **Kinesis / MSK** (not these three) |

### 1.4 The migration arc

Most teams' arc:
1. **Year 0**: SQS for queueing; SNS for fanout. Simple.
2. **Year 1**: SNS+SQS fanout topology. Subscriptions explode.
3. **Year 2**: Move SNS → EventBridge for content-based routing and cross-account.
4. **Year 3**: EventBridge becomes the *event backbone*; SNS retained for mobile push and SES; SQS retained for queues.

The endgame is *EventBridge as the bus, SQS as the buffer, SNS for what only SNS does*.

### 1.5 Quick reference: limits and pricing (us-east-1, current at writing)

| Aspect | SNS | SQS | EventBridge |
|---|---|---|---|
| **Cost (publish)** | $0.50/M | n/a (only on receive/send) | $1/M custom events; $0 AWS-source on default bus |
| **Cost (delivery)** | $0.06–$2/M depending on protocol | $0.40/M (Std), $0.50/M (FIFO) | included with rule match |
| **Max message size** | 256 KB | 256 KB (extended via S3 to 2 GB) | 256 KB |
| **Throughput** | 100K+ TPS (Std); 300/3K TPS (FIFO) | unlimited (Std); 300/3K/70K TPS (FIFO) | 10K events/sec/account default; raisable |
| **Ordering** | best-effort (Std); strict (FIFO) | none (Std); strict (FIFO) | best-effort |
| **Delivery semantics** | at-least-once | at-least-once (Std); exactly-once-processing (FIFO) | at-least-once |
| **Retention** | 1 hour delivery retry | 1 min – 14 days | rule match instant; archive separately |
| **Filtering** | message attribute filter | none | rich content-based filter |
| **Schema** | none | none | schema registry |
| **Replay** | no | no (DLQ + redrive) | yes (from archive) |
| **DLQ** | yes (per subscription) | yes | yes (per target) |
| **Cross-account** | yes (subscription) | yes (resource policy) | yes (native) |
| **Cross-region** | manual (subscriber pulls) | manual | EventBridge global endpoints |
| **Scheduling** | no | delay queues (15 min max) | EventBridge Scheduler (any future time) |

---

## 2. Scenario 1 — Order Processing Pipeline

### 2.1 The problem

E-commerce. User places order. Multiple downstream actions:
- Inventory: reserve stock.
- Payment: charge card.
- Email: confirmation.
- Shipping: schedule.
- Analytics: record event.
- Loyalty: credit points.

At peak: 10K orders/sec. Each downstream is independently scaled and operated.

### 2.2 The naive approach

Order service synchronously calls each downstream:

```
OrderService → Inventory → Payment → Email → Shipping → Analytics → Loyalty
```

Issues:
- Latency = sum of all downstreams.
- Any downstream failure = order fails (or cascading retries).
- Tight coupling: change Loyalty schema → Order service deploys.
- Order service holds state during minutes-long flow.

### 2.3 The SNS+SQS fanout approach (classic)

```
OrderService → publish "OrderPlaced" → SNS topic
                                        │
                                        ├─► SQS_inventory  → InventoryService
                                        ├─► SQS_payment    → PaymentService
                                        ├─► SQS_email      → EmailService
                                        ├─► SQS_shipping   → ShippingService
                                        ├─► SQS_analytics  → AnalyticsService
                                        └─► SQS_loyalty    → LoyaltyService
```

Each consumer:
- Has its own SQS queue (buffering, isolation).
- Pulls at its own pace.
- Failures contained (DLQ per queue).
- No back-pressure to producer.

**Why SNS+SQS, not direct SNS→Lambda?**
- SQS buffers bursts: 100K orders in 1 second don't overload your Lambda concurrency or downstream.
- DLQ per queue: each consumer has its own redrive policy.
- Replay: messages in queue survive consumer restarts.
- Decoupled scaling: Lambda autoscales pulling from queue; queue absorbs the rate mismatch.

This pattern is the **default modern choice for fanout-with-buffering**.

### 2.4 The EventBridge alternative

```
OrderService → put "OrderPlaced" event → EventBridge bus
                                          │
                                          ├─► rule 1: filter type=Order
                                          │   └─► SQS_inventory  (Lambda)
                                          ├─► rule 2: filter type=Order && payment_method exists
                                          │   └─► SQS_payment
                                          ├─► rule 3: filter type=Order && country=US
                                          │   └─► SQS_email
                                          └─► ... etc
```

Benefits over SNS:
- **Content-based filtering** at routing layer (no consumer-side filtering).
- **Schema registry** (catch-all "what events does this bus emit" via Schema Discovery).
- **Archive + replay**: replay last 7 days of orders if downstream had a bug.
- **Cross-account targets**: route order events to a partner-owned target without sharing IAM.
- **Multiple targets per rule**: 5 targets per rule by default.

The **trade-off**: EventBridge $1/M custom events vs SNS $0.50/M publish. At 10K events/sec × 30M/month = $30/month of EventBridge vs $15 of SNS. Negligible at small scale; meaningful at billions/month.

### 2.5 What about FIFO ordering?

For "OrderCancelled" and "OrderUpdated" events on the same order — must arrive in order. Otherwise downstream sees "cancel" before "update" and acts wrong.

Options:

| Option | Trade |
|---|---|
| **SNS FIFO + SQS FIFO** | Strict order; but lower throughput (300 TPS default per topic; 3K with high-throughput) |
| **EventBridge with order_id partition key + downstream serialization** | Best-effort; downstream sequences by order_id |
| **Kinesis / MSK** | Strict ordering per shard/partition; more ops cost |
| **Application-level versioning** | Each event has version; consumer rejects stale; needs idempotency |

For 10K orders/sec, FIFO 300 TPS is too slow. **High-throughput FIFO** (per group ID; 3K transactions/sec per group; 70K TPS aggregate) works if `MessageGroupId = order_id` (each order's events serialized; orders parallel).

```
SNS FIFO Topic → SQS FIFO Queue
Producer publishes with MessageGroupId=order_id, MessageDeduplicationId=event_id
Per-order strict ordering; across orders parallel.
```

**Staff insight**: FIFO is for the *minority* of events that need ordering. Use Standard for the rest. Don't blanket-FIFO; you'll hit throughput limits and pay extra per message.

### 2.6 Trade-offs and alternatives

| Approach | Throughput | Filtering | Replay | Order | Cost | Coupling |
|---|---|---|---|---|---|---|
| **Sync RPC chain** | order-rate | n/a | n/a | strict | n/a | tight |
| **SNS + SQS fanout** | high | basic (attr) | no | best-effort | low | loose |
| **EventBridge + SQS** | high | rich content | yes (archive) | best-effort | medium | loose |
| **SNS FIFO + SQS FIFO** | 3-70K TPS | basic | no | strict | medium-high | loose |
| **Kafka / Kinesis** | very high | partition | replay | per-partition | high | loose |
| **Step Functions saga** | medium | n/a | yes | sequenced | medium | choreographed |

### 2.7 What I'd actually do

Modern build at MAANG scale:
- Order service emits events to **EventBridge custom bus** (`order-events`).
- Each downstream owns an **SQS queue + Lambda/Fargate consumer**.
- EventBridge rules route by content (order type, region, customer tier).
- DLQ on every Lambda/SQS combo.
- 7-day archive on the bus for replay.
- For ordered events (cancel/update on same order): **SNS FIFO + SQS FIFO** with `MessageGroupId=order_id` — separate from the main bus.
- Saga orchestration via Step Functions when the multi-step is *workflow-like* (see §6).

---

## 3. Scenario 2 — Burst Buffering (SQS as Shock Absorber)

### 3.1 The problem

Image processing service. User uploads → must thumbnail. Normal load: 100/sec. Burst (Black Friday, viral content): 50K/sec for 10 minutes. The processing tier can do 1000/sec sustained.

### 3.2 The naive approach

Synchronous: API → Processor. At burst, API server queues internally → eventually OOMs or times out. Users see errors.

### 3.3 The SQS buffer

```
Upload request → S3 (object stored) → S3 ObjectCreated event → SQS queue
                                                                ▼
                                                  Lambda / ECS / EKS workers
                                                  Pull from queue at sustainable rate
                                                  Process; mark done
```

The queue absorbs the burst:
- 50K/sec arrival × 10 min × 60 sec = 30M backlog.
- Workers process at 1000/sec sustained → 30,000 sec = 8.3 hours to drain.
- During the burst: queue depth grows; worker count constant.
- After burst: queue drains; users see eventual completion.

### 3.4 What "absorb" actually means

```
SQS Standard:
  - Effectively unlimited throughput.
  - 14 days max retention (configurable).
  - Visibility timeout: lock window for in-flight messages (default 30s, max 12h).
  - Each message ~$0.40/M.
```

The queue is the buffer. The producer never blocks. The consumer never overloads.

### 3.5 Visibility timeout pathology

```
Visibility timeout = 30s (default).
Worker pulls message; processing takes 35s; visibility expires; another worker pulls;
both process; double processing.
```

**Mitigations:**
- Set visibility timeout > p99 processing time (with 2× margin).
- For long jobs: extend visibility (`ChangeMessageVisibility`) periodically in worker.
- Make processing **idempotent** (processed?: skip).
- Use a deduplication key in downstream state to detect dupes.

This is the #1 cause of "duplicate processing" bugs in SQS-based systems.

### 3.6 Long polling vs short polling

```
Short polling (default):
  Worker calls ReceiveMessage(WaitTimeSeconds=0).
  Returns immediately even if queue empty.
  Worker spins, calls again. Wastes API calls = wastes money.

Long polling:
  Worker calls ReceiveMessage(WaitTimeSeconds=20).
  Waits up to 20 seconds for messages.
  Returns sooner if any arrive.
  10× cheaper at low queue depth.
```

**Always set WaitTimeSeconds=20** unless you have a specific reason. Forgotten setting = 10× wasted SQS API calls.

### 3.7 Backpressure to upstream

Sometimes you don't want unbounded queueing. Several strategies:

```
1. Reject at API if queue depth > threshold (CloudWatch alarm + Lambda check).
2. SQS source-side throttling (publishing rate-limited at API tier).
3. Visibility-timeout extension model: workers pace themselves.
4. Use Lambda's "maximumConcurrency" to bound consumption rate.
```

For "we'd rather drop than backlog 4 hours" workloads: bound the queue or shed at admission.

### 3.8 SQS at extreme scale

```
At 50K msgs/sec sustained on a single queue:
  - Throughput is fine (SQS Standard scales).
  - LIST / receive bursts may face short throttling if not warmed up.
  - DLQ same throughput considerations.
  - Consumer fleet must scale to match.
```

At MAANG-scale ingestion (>500K msgs/sec): consider **partition by key into N queues**. Each queue has its own consumer fleet. Avoids any per-queue API ceiling.

### 3.9 Trade-offs and alternatives

| Approach | Trade |
|---|---|
| **SQS Standard** | Cheap, scalable, 14-day retention, at-least-once |
| **SQS FIFO** | 3K–70K TPS, exactly-once-processing, ordered |
| **Kinesis Data Streams** | Replay, 7-day retention default, ordered per shard |
| **MSK (Kafka)** | Highest throughput, ordered, expensive ops |
| **DynamoDB-as-queue** | Don't (anti-pattern, but seen in practice) |

### 3.10 What I'd actually do

For burst buffering: **SQS Standard** every time. Set:
- WaitTimeSeconds=20.
- VisibilityTimeout=2× p99 processing.
- DLQ with redrive after 5 attempts.
- Idempotent consumer.
- Alarm on queue depth > N (catches stuck consumers).
- Alarm on age-of-oldest-message > T (catches consumer falling behind).

---

## 4. Scenario 3 — Massive Fanout to Many Consumers

### 4.1 The problem

Platform event: "UserSignedUp." 50 internal services need to know: profile, billing, email, recommendations, fraud, analytics, audit, marketing, etc. New services added monthly. Producer shouldn't know about consumers.

### 4.2 SNS topic with N subscriptions

```
Producer → SNS topic ─┬─► SQS queue 1 → Consumer 1
                      ├─► SQS queue 2 → Consumer 2
                      ├─► Lambda 3
                      └─► HTTPS endpoint 4
```

- One publish, N delivery.
- Subscribers added without producer code change.
- SNS retries failed deliveries (HTTPS, mobile).

Practical limits:
- 12.5M subscriptions per topic (effectively unlimited for typical use).
- 256 KB max message.
- Filter policies per subscription (basic attribute filtering).

### 4.3 SNS message filtering

```json
// Subscription filter policy
{
  "event_type": ["user.signed_up", "user.upgraded"],
  "country": ["US", "CA"],
  "premium": [true]
}
```

If message attributes match → delivered. If not → filtered out (no charge for delivery).

Limitation: filter on **message attributes**, not on payload body. So producer must enrich attributes for filterable fields.

### 4.4 EventBridge for richer routing

```
EventBridge custom bus event:
  source: "user-service"
  detail-type: "UserSignedUp"
  detail: { user_id, country, plan, age, ... }

Rule 1:
  pattern: { source: ["user-service"], detail-type: ["UserSignedUp"] }
  target: SQS profile-queue

Rule 2:
  pattern: { source: ["user-service"], detail: { country: ["US"], plan: ["premium"] } }
  target: Lambda special-onboarding

Rule 3:
  pattern: { detail: { age: [{ numeric: [">=", 65] }] } }
  target: Lambda senior-discount
```

EventBridge filters on the **full payload** with rich operators (numeric comparisons, prefix, exists, anything-but, $or, $and, content-based suffix/prefix). SNS can't do this without consumer-side logic.

### 4.5 The fanout cost arithmetic

```
SNS standard topic, 1M messages/sec, fanned to 50 SQS queues:
  Publish: 1M × $0.50/M = $0.50
  Delivery to SQS: 50M × $0.06 = $3.00
  SQS receive: 50M × $0.40/M = $20.00 (assuming each consumed once)
  Total per second: ~$23.50/sec → $61M/month at sustained 1M/sec

Reduce by:
  - Batching at SQS receive (10× per call, same cost).
  - Using EventBridge with content filters → fewer deliveries.
  - Sharding to fewer high-volume queues + filtering at consumer.
```

At MAANG scale, fanout cost is real. Filtering reduces cost; batching reduces per-message overhead.

### 4.6 SNS → SQS or SNS → Lambda directly?

```
SNS → SQS → Lambda:
  + Buffer absorbs Lambda concurrency limits.
  + Replay if consumer broken.
  + DLQ at SQS layer.
  + Lambda batching from SQS (10 messages/invocation).
  - Extra hop, extra cost (SNS delivery fee + SQS receive fee).

SNS → Lambda directly:
  + Lower latency.
  + Cheaper per delivery (no SQS in middle).
  - Lambda concurrency limits hit during bursts → throttles → SNS retries → eventual loss.
  - DLQ at SNS subscription level (less granular than SQS).
  - No replay window beyond SNS retry policy (1 hour total).
```

For low volume / known-reliable consumer: SNS → Lambda. For production-grade burst-tolerance: SNS → SQS → Lambda.

### 4.7 Trade-offs and alternatives

| Approach | Use when |
|---|---|
| **SNS + SQS fanout** | Many consumers, simple attr filtering |
| **EventBridge + targets** | Rich filtering, cross-account, archive |
| **Kafka + consumer groups** | High throughput, replay-as-feature, log-based |
| **DynamoDB Streams + Lambda** | DB-driven event sourcing |
| **Direct service-to-service** | <5 consumers, simple |

### 4.8 What I'd actually do

For modern fanout: **EventBridge custom bus** with content-based rules per consumer. Consumers' targets are SQS+Lambda for buffering. Archive enabled for replay. Schema registry tracking what events exist.

For very-high-throughput simple fanout: SNS+SQS retains a place (lower per-event cost, simpler ops).

---

## 5. Scenario 4 — Strict Ordering (FIFO Use Cases)

### 5.1 The problem

Financial system. Account balance updates: "deposit $100", "withdraw $50". Must process in order or the user briefly goes overdrawn (and the system might reject). Trade events for the same security must serialize. Ledger entries must apply in correct order.

### 5.2 FIFO mechanics

```
SQS FIFO queue:
  - MessageGroupId: messages with same group serialize; across groups parallel.
  - MessageDeduplicationId (or content-based dedup): drops dupes within 5-min window.
  - Throughput: 300 TPS default per group; 3K with high-throughput; 70K aggregate.
  - Visibility timeout: still applies; in-flight blocks others in group.

SNS FIFO topic → SQS FIFO queue:
  - Same group/dedup semantics.
  - Subscribers must be SQS FIFO (only).
```

### 5.3 Choosing the MessageGroupId

The group id is the **unit of serialization**.

```
Wrong: MessageGroupId = "all" (everything serial; throughput = 300 TPS).
Right: MessageGroupId = account_id (per-account serial; across-account parallel).
```

Group cardinality determines parallelism:
- 1 group → 300 TPS (1 worker effectively).
- 1000 groups → 300K TPS theoretical (300 per group × 1000).
- 100K accounts → near-unlimited parallelism.

### 5.4 Hot group / hot account

If account "platform_fees" is touched on every transaction, that group's throughput caps at 300 TPS regardless of how many workers.

Mitigations:
- **Sharded counters** in DynamoDB / database (same pattern as in transactions doc).
- **Async aggregation** (write to per-shard sub-account; sum on read).
- **High-throughput FIFO mode** — 3K TPS per group instead of 300.
- **Avoid the hot group** by design.

### 5.5 Deduplication

```
Two patterns:
1. Content-based dedup: SHA-256 of body; SQS deduplicates within 5-min window.
2. Explicit dedup ID: producer provides idempotency key.

5-min window: messages with same dedup id within 5 min → second silently dropped.
```

This is **exactly-once-processing**, not exactly-once-delivery. The message is delivered once to *one consumer*. If you process and the consumer crashes before deleting → another consumer pulls the same message after visibility timeout. Still need idempotency at consumer.

### 5.6 The visibility timeout in FIFO

```
Worker A pulls msg M from group G at t=0; visibility=30s.
While A is processing: NO OTHER message in group G can be received.
If A crashes; M reappears at t=30; another worker pulls.

Practical effect: in-group serialization is enforced even across worker failures.
```

This is what makes FIFO usable: ordering survives consumer crashes.

### 5.7 SNS FIFO + SQS FIFO end-to-end

```
Producer → SNS FIFO Topic (publishes with group_id, dedup_id)
                              │
                              └─► SQS FIFO Queue (same group, dedup honored)
                                    └─► Consumer (worker per group)
```

Subscribers to SNS FIFO must be SQS FIFO (no Lambda direct, no HTTPS, no email).

### 5.8 When NOT to use FIFO

- Throughput needs > 70K TPS aggregate even with high cardinality groups.
- Order doesn't actually matter; engineers reach for FIFO defensively.
- Cost-sensitive (FIFO is ~25% more expensive than Standard).
- Cross-region (SQS FIFO is regional; you'd manually replicate).

For most events, **Standard + idempotency + per-key sequencing in app** beats FIFO. Reach for FIFO only when ordering is a hard correctness requirement.

### 5.9 Trade-offs and alternatives

| Approach | Order | Throughput | Replay | Cost |
|---|---|---|---|---|
| **SQS FIFO** | per-group strict | 70K TPS | no | medium |
| **SQS Standard + app-level seq** | best-effort | unlimited | no | low |
| **Kinesis (per-partition order)** | partition strict | 1K records/sec/shard, scales | yes (7d) | medium |
| **MSK (Kafka)** | partition strict | very high | unlimited | high ops |
| **Spanner / Timestamp ordering** | global strict | bounded | yes | very high |

### 5.10 What I'd actually do

For ordered-per-key workloads at MAANG scale:
- **SQS FIFO** if 70K TPS aggregate is enough and you need AWS-native simplicity.
- **MSK / Kinesis** if you need replay, higher throughput, or stream processing semantics.
- **App-level versioning + Standard SQS** for most "should be ordered" cases that aren't strict.

---

## 6. Scenario 5 — Workflow Orchestration (Saga, Step Functions)

### 6.1 The problem

Customer onboarding flow: 8 steps, mix of synchronous (call APIs) and asynchronous (wait for callback). Must handle failures, retries, compensation. Needs visibility for ops.

### 6.2 Choreography (event-driven choreography)

```
ServiceA → emit "Step1Done" → EventBridge → ServiceB consumes, does step 2, emits "Step2Done" → ...
```

- No central coordinator.
- Each service independent.
- Hard to debug: workflow logic scattered across services and event logs.
- Hard to change: adding step requires multiple service updates.
- Failure handling: each service publishes failure events; subscribers compensate. Complex.

Good for: high-volume, simple flows, loosely-coupled services with independent teams.

### 6.3 Orchestration (Step Functions)

```
Step Functions state machine:
  State 1: invoke ServiceA (Task)
  State 2: invoke ServiceB (Task)
  State 3: wait for callback (WaitForTaskToken)
  State 4: choice (Choice)
  ...
  Catch: on failure → compensating actions.

Step Functions persists state durably; workflow survives crashes.
Visual representation; easy to debug.
Built-in retry, error handling, parallel branches.
```

- One central place for the workflow.
- Visible: each execution has a UI graph.
- Cost: $25/M state transitions for Standard workflows; $0.000025/state for Express (high-volume).
- Limits: 1-year max execution duration (Standard); 5-minute (Express).

For complex multi-step workflows: orchestration wins on operability.

### 6.4 The hybrid pattern

```
Step Functions orchestrates workflow.
Each step is async via SNS/SQS/EventBridge.
Step Functions waits via callback token (WaitForTaskToken pattern):
  - Service A starts; emits event with TaskToken; processes async; calls SendTaskSuccess(token).
```

This combines orchestration (workflow visibility) with async events (no blocking).

### 6.5 Choreography vs orchestration trade

| Aspect | Choreography | Orchestration |
|---|---|---|
| **Coupling** | Lowest | Medium (state machine knows services) |
| **Visibility** | Hard (across services) | Easy (one execution) |
| **Debug** | Painful | Straightforward |
| **Adding step** | Many services touched | One state machine |
| **Cost** | Per-event (EventBridge $1/M) | Per-transition (~$25/M) |
| **Fit** | Simple flows, many independent teams | Complex flows, audit needs |

### 6.6 The EventBridge Pipes pattern (2022+)

```
Source → Pipe → (optional filter) → (optional enrich) → Target

E.g.:
  DynamoDB stream → Pipe → filter (only INSERT events) → enrich (Lambda) → EventBridge bus
```

Pipes replace boilerplate Lambda code that just shuffled events between sources and sinks. For source-to-target with light transformation, pipes are cheaper than equivalent Lambda code.

### 6.7 Step Functions Standard vs Express

```
Standard:
  - 1-year execution.
  - Detailed history / visibility.
  - $25/M state transitions.
  - Best for long-running, audit-needed workflows.

Express:
  - 5-min max execution.
  - $1/M starts + $0.000004/GB-sec memory.
  - 100K starts/sec.
  - Best for high-volume short-duration workflows (e.g., per-request fan-out).
```

For event-per-request orchestration at high RPS: Express. For long-running business processes: Standard.

### 6.8 Trade-offs and alternatives

| Approach | When |
|---|---|
| **Choreography (EventBridge)** | Loose coupling, simple flows |
| **Step Functions Standard** | Complex, audit-heavy, long-running |
| **Step Functions Express** | High-volume, short, traceable |
| **Temporal / Cadence** | Code-first orchestration; richer features |
| **AWS-managed Camunda** | Enterprise BPMN; legacy-friendly |
| **SQS chain (manual)** | Simplest; no visibility |

### 6.9 What I'd actually do

For a complex onboarding flow at scale:
- **Step Functions Standard** as orchestrator.
- Each step a Task: Lambda for sync, SNS/SQS+callback token for async.
- DLQ on each Task for failures.
- CloudWatch alarms on workflow stuck > N hours.
- For high-volume per-request flows: Express.

For simple event chains: **EventBridge choreography** is fine.

---

## 7. Scenario 6 — SaaS Webhook Delivery to Customers

### 7.1 The problem

Your SaaS sends webhooks to customers when events occur ("invoice paid", "subscription updated"). Hundreds of customers; each receives 100s of events/day. Customer endpoints occasionally:
- Return 5xx (their bug).
- Time out (their system slow).
- Are temporarily down for maintenance.

You must deliver eventually with retry; not lose events; respect rate limits per customer; not let one bad customer back up the entire system.

### 7.2 The naive approach

```
Internal event → Lambda calls customer URL.
Failure → Lambda retries. Many failures → Lambda runs hot.
Lambda has no per-customer isolation; one customer's slowness blocks others.
```

### 7.3 The SQS-per-customer pattern (sometimes)

```
Internal event → Lambda decides webhook target → SQS queue (per-customer)
                                                    ▼
                                                  Worker pool sends webhook
```

- Per-customer queue: isolation. One slow customer's queue grows; doesn't block others.
- Worker per customer: bounded concurrency.

But: thousands of queues becomes operational nightmare.

### 7.4 The single-queue + per-customer concurrency pattern

```
Internal event → SQS (single queue) with body containing customer_id
                  ▼
              Lambda consumes; fetches customer's webhook URL; sends.
              Lambda set with maximumConcurrency to bound aggregate.
              Per-customer rate limiting in Lambda (token bucket via DynamoDB / Redis).
```

Better — but the slow-customer problem remains: Lambda invocations holding for that customer slow-burn the concurrency budget.

### 7.5 The dispatch tier pattern (production-grade)

```
1. Event → main bus (EventBridge).
2. Dispatch service: looks up subscribed customers; writes message to per-customer-shard SQS queue.
   - 32 SQS queues (sharded by customer_id hash).
3. Per-shard consumer pool: pulls from queue; sends webhook.
   - Each consumer has rate limit per customer (token bucket).
   - Slow customer hits its bucket; back-pressure on its messages only.
4. Per-customer DLQ; retry strategy.
5. After N retries: move to "dead" state; mark customer as unhealthy; circuit break further deliveries.
```

This is what Stripe, Shopify, Twilio, and other webhook-heavy SaaS actually do. Variants:

- **EventBridge → multiple SQS queues sharded**: AWS Eventbridge "destination" or "rule" sharding.
- **Dedicated workers per shard**: ECS/EKS service per shard.
- **Centralized rate-limit store**: Redis with per-customer token bucket.

### 7.6 Idempotency for webhook delivery

The customer endpoint is at-least-once recipient. Your webhook payload must include:
- **Idempotency key** (unique event id; customer can dedup).
- **Signed payload** (HMAC) so customer verifies authenticity.
- **Timestamp** so customer can reject very old retries.

Document this clearly in your API. Customers who don't dedup will see double-charges, double-emails, etc.

### 7.7 SNS HTTP/HTTPS subscriptions — the alternative

```
SNS topic → HTTPS subscription with customer URL.
SNS retries: 3 immediate, then with exponential backoff up to 1 hour total.
After 1 hour: dead-letter to a DLQ if configured.
```

Limitations:
- 1-hour total retry window; many real customer outages last longer.
- No per-customer rate limiting.
- No per-customer isolation.
- Single shared retry pool.

For reliable webhook delivery at scale, SNS HTTP subscriptions are too simplistic. Custom dispatch tier is the production answer.

### 7.8 Trade-offs and alternatives

| Approach | Trade |
|---|---|
| **SNS HTTP subscription** | Simplest; weak retry/isolation |
| **EventBridge API destination** | Newer; rate limit + retry + bucket pattern; not infinitely flexible |
| **Custom dispatch tier (SQS shards + workers)** | Most flexible; ops cost |
| **Managed services (Hookdeck, Svix)** | Outsourced; per-event cost |

### 7.9 What I'd actually do

For mid-scale (M events/day): **EventBridge API destinations** — added 2022; provides retries, throttling, archiving, OAuth/auth headers. Good enough for most webhook delivery now.

For mega-scale (B events/day): **custom dispatch tier** with sharded SQS queues, per-customer rate limit, dead-letter circuit breaker. Engineering investment but operationally controllable.

---

## 8. Scenario 7 — DLQ and Poison Pill Handling

### 8.1 The problem

A small fraction of messages cause consumer to crash or reject (malformed JSON, missing fields, business-rule violation). Without handling, they re-deliver forever, burning Lambda invocations and clogging the queue.

### 8.2 The DLQ pattern

```
Main queue (SQS):
  receiveCount on each message.
  If receiveCount > maxReceiveCount: redrive to DLQ.
  
DLQ: SQS queue with the same message + original failure context.
```

```yaml
# CloudFormation
MainQueue:
  RedrivePolicy:
    deadLetterTargetArn: !GetAtt DLQ.Arn
    maxReceiveCount: 5
```

After 5 failures, the message lands in DLQ. Main queue continues processing healthy messages.

### 8.3 What goes into the DLQ

Different consumers contribute different DLQ patterns:

```
SQS DLQ:           messages that exceeded maxReceiveCount.
SNS DLQ:           messages that failed all delivery retries to a subscriber.
Lambda DLQ:        Lambda invocation errors after retries (deprecated; use destinations).
Lambda destination: more flexible than DLQ; on success or failure.
EventBridge DLQ:   per-target; events that target couldn't accept after retries.
Step Functions:    failure path within state machine.
```

Each must be configured. Forgetting DLQ on any of these = silent data loss.

### 8.4 DLQ ops discipline

A DLQ that nobody reads is just a slow leak. Required:
- **Alarm** on DLQ depth > 0 (or threshold).
- **Daily report**: top error reasons; counts.
- **Replay tool**: restore from DLQ to main queue after fix.
- **Retention**: DLQ retention 14 days; if not investigated, escalate.

### 8.5 Replay strategies

```
1. Manual: receive from DLQ, reprocess, delete.
2. Scripted: redrive policy on DLQ → another queue with new consumer.
3. Filtered: redrive only messages of certain type or after fix tag.
```

AWS added **SQS redrive task** (2022) to automate moving messages back from DLQ to source queue.

### 8.6 Poison pill detection

A specific category: messages that *will never* succeed (corrupted, schema-mismatch). Should be detected and shed immediately, not retry 5 times.

```
Consumer logic:
  try: process(msg)
  catch (UnrecoverableError):
    move to DLQ immediately (or to "poison" bucket); don't increment receive count.
  catch (TransientError):
    return without deleting; let visibility timeout retry.
```

The trick: distinguish recoverable from unrecoverable errors at the consumer.

### 8.7 The cascading retry trap

Default Lambda + SQS:
```
Lambda fails on message → SQS retries → Lambda fails → ...
maxReceiveCount=5 → message in DLQ after 5 invocations.

Each invocation duration counted; if avg 30s, 150s of "wasted" Lambda time per poison pill.
At 1M poison-pill events: 150M Lambda-seconds = significant $$$.
```

**Mitigation**: shorten visibility timeout slightly so retries happen quickly, *or* use Lambda destinations to short-circuit on specific error classes.

### 8.8 Trade-offs and alternatives

| Approach | Trade |
|---|---|
| **Standard DLQ + alarm** | Defaults; works; needs ops |
| **Lambda destinations** | Per-invocation routing of success/failure; richer than DLQ |
| **Per-error-class DLQs** | Multiple DLQs by error type; granular triage |
| **Step Functions with explicit error states** | Best visibility; cost |

### 8.9 What I'd actually do

For every async consumer at scale:
- DLQ on the source queue with maxReceiveCount=5.
- Alarm on DLQ depth.
- Per-error-class handling in consumer code:
  - Schema errors → poison DLQ immediately.
  - Transient errors → retry path.
  - Business-rule errors → separate "manual review" queue.
- Daily DLQ triage report to a Slack channel.
- Quarterly DLQ replay drill (verify replay tooling actually works).

---

## 9. Scenario 8 — Cross-Account Event Routing

### 9.1 The problem

Multiple AWS accounts (per business unit, per environment, per security domain). Events generated in Account A must reach consumers in Account B (and C, D, ...). Cannot share IAM users.

### 9.2 EventBridge cross-account (the modern way)

```
Account A: event bus "shared-events"
  Permission: allow Account B to PutEvents.
  
Account A producer → puts event → bus
  Bus rule: target = Account B's bus (cross-account).
  Account B's bus rule: target = its own SQS or Lambda.
```

```yaml
# Account A's bus permission policy
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowAccountBPutEvents",
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::222222222222:root" },
    "Action": "events:PutEvents",
    "Resource": "arn:aws:events:us-east-1:111111111111:event-bus/shared-events"
  }]
}
```

EventBridge handles cross-account natively — the rule's target can be another account's bus, Lambda, SQS, etc.

### 9.3 SNS cross-account

```
Account A publishes to SNS topic.
Topic policy allows Account B's principal (or root) to subscribe.
Account B subscribes its SQS queue (queue policy allows topic to send).
```

Two policies must align: the SNS topic's resource policy allowing B to subscribe, and the SQS queue in B allowing the SNS topic to send.

### 9.4 SQS cross-account

```
Account A's queue has resource policy allowing Account B to send/receive.
Account B's IAM allows the action against that queue.
```

Both sides must allow.

### 9.5 The "events as the integration contract"

A common pattern at large orgs:

```
Centralized event bus (in a "hub" account):
  All teams publish to it.
  All teams subscribe via rules.
  Schema registry shared.
  Archive shared.
  Replay shared.

Each team owns its events as products.
Teams subscribe to other teams' events without coordination.
```

This is the "event mesh" pattern. EventBridge is the AWS-native tool for it.

### 9.6 Permission boundary considerations

Cross-account = expanding trust boundary. Mitigations:
- Use **distinct buses per domain** (don't put all events on one bus).
- Use **resource policies + SCPs** to restrict what can publish.
- **Event encryption** with KMS; cross-account KMS permissions required (the same gotcha as S3).
- **Audit trail**: CloudTrail captures PutEvents calls.

### 9.7 Trade-offs and alternatives

| Approach | Trade |
|---|---|
| **EventBridge cross-account bus** | Native; rich; modern |
| **SNS cross-account subscription** | Mature; limited filtering |
| **SQS cross-account** | Direct point-to-point |
| **Direct API call** | Tightest coupling; not "events" |

### 9.8 What I'd actually do

For multi-account orgs:
- **Hub-and-spoke EventBridge**: each account has a bus; central hub bus for shared events.
- **Schema registry** in the hub.
- **Cross-account replication** via rules.
- IAM and SCP guardrails.
- Cost-allocation tags on events (which producer, which domain).

---

## 10. Scenario 9 — Cross-Region Event Distribution

### 10.1 The problem

Multi-region active-active service. Events generated in us-east-1 must reach consumers in eu-west-1 and ap-southeast-1. Local consumers should consume locally for latency. DR: lose a region, others continue.

### 10.2 EventBridge global endpoints (2021+)

```
Global endpoint: front of two buses (primary + secondary region).
  Producer publishes to global endpoint.
  Routes to primary; replicates to secondary.
  On primary failure: failover to secondary.

Replication: cross-region async (1-2 sec lag typical).
```

This is the AWS-native multi-region event solution. Provides:
- **Primary/standby** active-passive.
- **Auto-failover** based on health.
- **Replay** from archive on either side.

For active-active where both regions independently produce: it's still primary+backup, not full active-active. For independent multi-region producers: replicate to both buses; consumers subscribe locally.

### 10.3 SNS cross-region (manual)

```
Producer → SNS in region A.
Lambda subscriber in region A: PutEvents to region B's bus or topic.
Lambda subscriber in region B: PutEvents to region A's bus or topic.

This is "manual mirroring" — works but you operate it.
```

### 10.4 The cost of cross-region

```
Cross-region data transfer: $0.02/GB.
Per-event (1KB body): negligible.
Per-million events: $20 (1GB-equivalent).

But: fanout multiplier. 1 event × N regions × M consumers per region = high.
```

Plan replication carefully — don't replicate every event everywhere.

### 10.5 Region-pinned events

Most events are region-local. Only a subset benefits from global distribution:
- Identity changes (user account events).
- Cross-region reconciliation (financial, inventory).
- Global monitoring / metrics.
- Configuration / feature flag events.

For region-local events: keep them local. For globally-relevant events: replicate.

### 10.6 Trade-offs and alternatives

| Approach | Trade |
|---|---|
| **EventBridge global endpoints** | Native, primary-standby, auto-failover |
| **Manual mirroring (Lambda)** | Flexible, you own ops |
| **Kinesis cross-region replication** | Stream-based; replay on each side |
| **DynamoDB global tables + streams** | Data-driven event distribution |
| **Custom replication service** | Full control; engineering cost |

### 10.7 What I'd actually do

For multi-region at MAANG scale:
- Identify the **few event classes that need global distribution**.
- Use **EventBridge global endpoints** for those.
- Keep region-local events region-local.
- Cross-region replication via DynamoDB Global Tables for state events.
- Periodic DR drill: kill a region, verify failover.

---

## 11. Scenario 10 — Scheduled Jobs and Cron Replacement

### 11.1 The problem

Replace cron daemon and operator-managed schedules with a cloud-native solution. Schedule:
- Recurring (every 5 min, daily at 02:00, weekly).
- One-shot (in 30 minutes; at 2026-04-30 12:00).
- Per-customer (each tenant has different schedules).
- Massive scale (10M+ scheduled events per day).

### 11.2 The legacy: EventBridge Rules with schedule

```yaml
EventBridge Rule:
  ScheduleExpression: rate(5 minutes)         # or cron(0 2 * * ? *)
  Targets:
    - Arn: !GetAtt MyLambda.Arn
```

Limits:
- 300 rules per region per account historically.
- Schedule expression is per-rule; for many one-shot or per-customer schedules, you'd burn rules fast.

### 11.3 EventBridge Scheduler (2022+)

The new dedicated service:

- **One-shot or recurring**.
- **Per-target retry** + DLQ.
- **Time zones** (no UTC pain).
- **Up to 14M schedules per region per account** (paginated).
- **At-most-once or at-least-once** semantics.
- **Targets**: 200+ AWS services.
- **Cost**: $1.00/M schedules.

```yaml
Schedule:
  ScheduleExpression: at(2026-04-30T12:00:00)
  Target:
    Arn: !GetAtt MyLambda.Arn
    Input: '{ "tenant_id": "abc" }'
```

For "schedule per tenant per task," create a schedule per tenant. Each is independent.

### 11.4 Use cases

```
Daily ETL at 02:00:               EventBridge Scheduler (cron).
Per-customer report delivery:    Scheduler — one schedule per customer.
Trial expiry in 14 days:         Scheduler — one-shot.
Reminder emails:                  Scheduler — one-shot per email.
Cleanup jobs:                     Scheduler — recurring.
```

Replaces:
- Cron daemons (operator hassle).
- DB-polling pattern ("every minute, find rows where due_at < now()").
- Manual SQS delay queues (15-min max delay).
- Step Functions Wait (lower-level for similar things).

### 11.5 The DB-polling alternative

Some teams still use:

```
Every minute, Lambda polls: SELECT * FROM jobs WHERE due_at < NOW() AND status='SCHEDULED';
```

Issues:
- Database load proportional to number of scheduled jobs.
- 1-minute polling = ±60s precision.
- Hot table.

EventBridge Scheduler is the cloud-native replacement.

### 11.6 SQS delay queues for short delays

```
SQS delay: up to 15 minutes per-queue or per-message.
Use case: retry after 5 minutes; throttle bucket reset; rate limit.
```

For < 15 minute delays: SQS DelaySeconds is simpler and cheaper than Scheduler.

### 11.7 Trade-offs and alternatives

| Approach | Best for |
|---|---|
| **EventBridge Scheduler** | Recurring + one-shot + many schedules + future times |
| **EventBridge Rule (schedule)** | Few simple recurring schedules |
| **SQS Delay (≤15 min)** | Short async delays |
| **Step Functions Wait** | Wait inside a workflow |
| **DB polling** | Legacy; avoid for new |
| **DynamoDB TTL** | Auto-delete records; not "fire event" |

### 11.8 What I'd actually do

For new scheduled-jobs needs:
- Recurring infra schedules → **EventBridge Rule** (simple cron).
- Per-customer / one-shot → **EventBridge Scheduler**.
- Short delay (retry, etc.) → **SQS delay**.
- DB polling → migrate to Scheduler.

---

## 12. Scenario 11 — Real-Time Analytics Ingestion

### 12.1 The problem

Click events, telemetry, app events: 10M+ events/sec. Must ingest, route to multiple consumers (warehouse, real-time dashboard, ML feature store, anomaly detection).

### 12.2 Why not SNS / SQS / EventBridge here

```
SNS:        scales fine for fanout but no replay; per-event cost adds up.
SQS:        scales fine but no replay; consumer must handle all history.
EventBridge: $1/M custom events × 10M/sec = expensive at this volume.
```

For 10M+ events/sec: **Kinesis Data Streams** or **MSK (Kafka)** is the answer.

### 12.3 Kinesis Data Streams

```
Stream with N shards.
Shard: 1 MB/sec or 1000 records/sec input; 2 MB/sec output.
Producers: PutRecord(s) with partition key.
Consumers: Lambda, KCL, Firehose.
Retention: 1-365 days.
Replay: by sequence number / timestamp.
```

For 10M events/sec at 1 KB each: 10 GB/sec → 10,000 shards. Cost dominates.

Alternative: **on-demand mode** (auto-scales) — easier ops, but less predictable cost.

### 12.4 MSK (managed Kafka)

```
Kafka topic with partitions.
Higher throughput per broker; replay; consumer groups; mature ecosystem.
```

For >1M events/sec sustained, MSK often wins on cost-per-event vs Kinesis. But operational complexity higher.

### 12.5 The hybrid: Kinesis + EventBridge

```
Kinesis stream → consumer Lambda → EventBridge bus
                    └─► filters / enriches / fans out
```

Use Kinesis for high-volume stream; emit a downsampled / important subset to EventBridge for downstream subscribers that don't need every record.

### 12.6 Where SNS/SQS/EventBridge still fit

- **Low-volume control events** (config changes, deployments): EventBridge.
- **Per-user notifications** (mobile push): SNS.
- **Async work derived from analytics** (e.g., "user crossed engagement threshold"): EventBridge.

The three services aren't replaced by Kinesis — they coexist with it.

### 12.7 Trade-offs and alternatives

| Approach | Volume | Replay | Cost | Ops |
|---|---|---|---|---|
| **EventBridge** | <100K/sec | yes (archive) | $1/M | low |
| **Kinesis** | 1M+/sec | yes | shard $$ | low |
| **MSK / Kafka** | 10M+/sec | yes | high | high |
| **Firehose → S3** | very high | replay-via-S3 | low | low |
| **SQS** | unlimited | no | low | low |

### 12.8 What I'd actually do

For 10M+ events/sec analytics ingestion:
- **Kinesis Firehose** → S3 (Parquet) for warehouse.
- **Kinesis Data Streams** for real-time consumers (anomaly, dashboard).
- **EventBridge** for the *important subset* (alerts, downstream workflows).
- SNS/SQS for control-plane around it (deployment events, ops triggers).

---

## 13. Scenario 12 — Database CDC Pipeline

### 13.1 The problem

Operational database (Postgres / DynamoDB / Aurora) is source of truth. Need to fan out changes to: search index (OpenSearch), data warehouse (Redshift/Snowflake), cache invalidation, downstream microservices.

### 13.2 The patterns

```
Postgres + Debezium → MSK / Kinesis → consumers.
DynamoDB Streams → Lambda / EventBridge Pipes.
Aurora → DMS → S3 / Kinesis.
RDS Activity Streams → Kinesis.
```

### 13.3 EventBridge Pipes for DynamoDB

```
DynamoDB Stream → EventBridge Pipe →
  filter (only INSERT) → enrich (Lambda transforms) → target (EventBridge bus)
```

Pipes (2022+) replace boilerplate Lambda code that just shuffled stream events.

### 13.4 The dual-write problem

Producer writes to DB and emits event. If event publish fails after DB commit → divergence.

Solutions:
- **Outbox pattern** (covered in transactions doc): write event to DB in same txn; CDC reads it.
- **Native CDC**: DynamoDB Streams, Aurora binlog; no explicit publish.

DB-native CDC + EventBridge Pipes is the cleanest modern approach: zero application code, zero divergence.

### 13.5 Schema evolution

A change to the DB schema → CDC pipeline must keep working. Mitigations:
- Versioned event schema (envelope: {version, payload}).
- Schema registry (EventBridge Schema Registry).
- Backward-compatible changes only (nullable fields, no removed fields).

### 13.6 Trade-offs and alternatives

| Approach | Trade |
|---|---|
| **DynamoDB Streams + Pipes** | Native; cheapest; minimal code |
| **Postgres + Debezium → MSK** | Most powerful; ops-heavy |
| **DMS** | Managed; works for many DBs; less rich |
| **App-level outbox + EventBridge** | Decoupled from DB tech; transactional |

### 13.7 What I'd actually do

For DynamoDB-backed services: **Streams + Pipes + EventBridge bus**. For Postgres at scale: **Debezium → MSK** (or Pipes if low-volume). For app-level events: **outbox + EventBridge**.

---

## 14. Scenario 13 — Mobile Push Notifications

### 14.1 The problem

Send push notifications to mobile apps. iOS APNS, Android FCM. Per-user; reliable; scaled to millions.

### 14.2 SNS Mobile Push

SNS has unique support for mobile push:

```
Create platform application (APNS or FCM with credentials).
Each device registration → endpoint ARN (one per device).
Publish to endpoint → platform-specific delivery.
Or: subscribe endpoints to a topic → publish to topic → fanout to all devices.
```

Limits:
- 256 KB max message.
- 100M+ endpoints per platform application.
- Throughput: gradually scaling.

### 14.3 The user-targeted flow

```
User opens app → registers device token with backend.
Backend creates SNS endpoint; stores (user_id → endpoint_arn).
Notification needed → backend looks up endpoints → publishes to each.
```

Two patterns:

**Per-endpoint publish**: 1M notifications = 1M Publish API calls. Direct, simple, scales.

**Topic per user**: each user has SNS topic; devices subscribe; publish to topic fans out to all devices. Used for users with many devices.

**Topic per segment**: subscribe to topics like "all-android" or "premium-users". Publish once → fanout to thousands. Powerful for marketing, broadcast.

### 14.4 Direct vs Pinpoint

```
SNS Mobile Push:
  - Bare-metal mobile push.
  - You manage endpoint lifecycle, segmentation, scheduling.

Amazon Pinpoint:
  - Engagement platform on top of SNS.
  - Segments, campaigns, scheduling, A/B test, deep analytics.
  - Higher cost; more features.
```

For one-shot transactional notifications: SNS direct. For marketing campaigns: Pinpoint.

### 14.5 Trade-offs and alternatives

| Approach | Trade |
|---|---|
| **SNS Mobile Push** | Cheapest; you build retry, segmentation |
| **Pinpoint** | Managed campaign tooling |
| **Firebase Cloud Messaging** | Direct (no AWS) |
| **3rd-party (OneSignal, Braze)** | Best engagement features |

### 14.6 What I'd actually do

Transactional pushes (order shipped, message received): **SNS direct**.
Engagement / marketing: **Pinpoint** if you're in AWS, or 3rd-party if engagement features matter more than infra.

---

## 15. Scenario 14 — Email and SMS

### 15.1 SNS for SMS

```
SNS direct SMS:
  Publish to phone number.
  Worldwide delivery.
  Per-message cost ($0.00645/SMS in US, varies globally).
  No bulk feature.
```

For small-volume transactional SMS: SNS works. For high-volume marketing: dedicated provider (Twilio, Pinpoint).

### 15.2 Why not SNS for Email

SNS supports email subscription but only for raw notifications (e.g., subscribe yourself for alerts). For real email:

- **SES** (Simple Email Service) is the right service.
- Reputation management, bounce handling, DKIM, click tracking.
- $0.10/1K emails sent.
- Integration with EventBridge: SES bounces / complaints emit events.

### 15.3 The right architecture

```
Internal "SendNotification" event → EventBridge.
Rule: type=email → Lambda → SES.
Rule: type=sms → Lambda → SNS SMS.
Rule: type=push → Lambda → SNS Mobile Push.
Rule: type=in-app → Lambda → DynamoDB store + websocket.

SES bounce/complaint event → EventBridge → mark user "do not email".
```

EventBridge is the glue; SNS and SES are protocol-specific delivery mechanisms.

### 15.4 What I'd actually do

Multi-channel notification at scale:
- Notification API → EventBridge.
- Per-channel Lambda + SES/SNS/in-app.
- Per-user opt-out tracked centrally.
- Bounce handling via SES → EventBridge.
- Marketing tools (Pinpoint or 3rd-party) layered on top.

---

## 16. Scenario 15 — Slow Consumer / Hot Partition Protection

### 16.1 The problem

One slow consumer (or one hot tenant on a shared queue) consumes all the consumer-side throughput, starving everyone else.

### 16.2 The patterns

**Per-tenant queues**: most isolating, most operationally heavy.

**Sharded queues**: hash tenant_id → 32 queues; each consumer pulls from one shard. Bounded "blast radius" when one tenant misbehaves.

**Concurrency limits per tenant**: at consumer (Lambda concurrency, semaphore), enforce per-tenant max in-flight.

**Time-based throttling**: token bucket per tenant; rate-limit requests.

**Fairness scheduling**: consumer pulls from all tenants' queues round-robin, not greedily.

### 16.3 The Lambda concurrency limit

```
Lambda function → SQS event source.
maximumConcurrency on event source: per-source cap.
ReservedConcurrentExecutions: per-function cap.
```

If the queue gets one slow tenant's message: it can fill concurrency budget. Mitigations:
- Per-tenant Lambda? Operational nightmare.
- Multiple Lambdas with multiple queues, sharded.
- One Lambda with internal per-tenant rate limit.

### 16.4 The "shovel" pattern

```
SQS queue → Lambda receives in small batch → routes to per-tenant ECS tasks.
ECS task per tenant (or shard) processes; bounded concurrency per shard.
```

The Lambda is just a fast shovel; tenant-isolated work runs on long-lived workers.

### 16.5 Trade-offs and alternatives

| Approach | Isolation | Ops cost |
|---|---|---|
| **Single queue + concurrency limit** | weak | low |
| **Sharded queues** | medium | medium |
| **Per-tenant queues** | strong | high (1000s of queues) |
| **Per-tenant ECS service** | strongest | very high |

### 16.6 What I'd actually do

For multi-tenant async processing:
- **N sharded queues** by tenant_id hash (typical N = 16–64).
- Per-shard consumer fleet.
- Per-tenant rate limit at consumer (Redis/DynamoDB token bucket).
- Alarm on per-shard backlog imbalance.
- Auto-rebalance / migrate tenants between shards if persistent skew.

---

## 17. Scenario 16 — Schema Evolution

### 17.1 The problem

A producer changes event schema. Consumers must keep working. Across 50 consumers and 7 years of running, schemas evolve.

### 17.2 Backward-compatible changes only

Allowed:
- Add optional field.
- Add new event type.
- Widen field type (smaller int → bigger int).

Not allowed (without versioning):
- Remove field.
- Rename field.
- Change semantics.
- Change field type incompatibly.

### 17.3 Versioned schemas

```json
{
  "schema_version": "v2",
  "type": "user.signed_up",
  "data": { ... }
}
```

Consumers check version; route to v1 or v2 handler. Default to forward-compatible parsing (ignore unknown fields).

### 17.4 EventBridge Schema Registry

```
Schema discovery: enable on bus → discovered events become entries.
Schemas auto-versioned with API description.
SDK code generation: get strongly-typed client for events.
```

For org-wide event-mesh discipline: schema registry is the audit + governance layer.

### 17.5 Schema-on-read vs schema-on-write

```
Schema-on-write (Avro, Protobuf): producer encodes per schema; consumer decodes; mismatched fail.
Schema-on-read (JSON, dynamic): consumer parses what it knows; ignores rest.
```

JSON + JSON Schema validation is the typical SNS/SQS/EventBridge pattern. Strict schema enforcement (Avro) more typical in Kafka.

### 17.6 Trade-offs and alternatives

| Approach | Trade |
|---|---|
| **JSON + version field** | Simple; ad-hoc |
| **EventBridge Schema Registry** | Discovered + governed; AWS-coupled |
| **Confluent Schema Registry (Avro)** | Strict; Kafka-coupled |
| **Custom schema registry** | Full control; engineering cost |

### 17.7 What I'd actually do

- **JSON envelope with schema_version field** as the universal first step.
- **EventBridge Schema Registry** for discoverability.
- **CI rule**: producer can't deploy schema-breaking change.
- **Consumer parsing**: lenient, ignore unknown fields, version-aware.

---

## 18. Scenario 17 — Event Archive and Replay

### 18.1 The problem

A bug in a downstream consumer drops events for 6 hours before detection. Need to replay missed events.

### 18.2 EventBridge Archive

```
Archive on a bus:
  Stores events for N days (configurable).
  Replay: time range + filter pattern → re-emits to current rules.
```

```bash
aws events start-replay \
  --replay-name fix-consumer-bug \
  --event-source-arn arn:aws:events:...:event-bus/main \
  --event-start-time "2026-04-28T00:00:00Z" \
  --event-end-time "2026-04-28T06:00:00Z" \
  --destination Arn=arn:aws:events:...:event-bus/main,FilterArns=[...]
```

Powerful: can fix bugs by re-running the events through fixed code.

### 18.3 Considerations

- Replay re-emits to all matching rules; if you only want one consumer, filter at rule level or use a temporary rule.
- Idempotency at consumer is mandatory (replays = doubles).
- Cost: archive is per-GB stored.

### 18.4 SNS / SQS replay alternatives

SNS: no built-in archive. Mitigation: subscribe an SQS queue with long retention as "archive"; replay from it.

SQS: no native archive. DLQ + redrive is the closest, but messages are removed once delivered.

For replay-heavy needs, EventBridge wins. Or move to Kinesis/Kafka where replay is native.

### 18.5 Trade-offs and alternatives

| Approach | Replay |
|---|---|
| **EventBridge Archive** | Built-in; time + filter |
| **SQS-as-archive** | Manual; up to 14 days |
| **Kinesis** | Built-in; up to 365 days |
| **Kafka / MSK** | Built-in; unbounded with retention |
| **S3 + custom replay** | Most flexible; engineer it |

### 18.6 What I'd actually do

For audit / compliance / debug:
- **EventBridge Archive enabled on every production bus** (7–30 days).
- **Replay tooling** documented and rehearsed.
- **Idempotent consumers** mandatory.
- For higher-volume or longer retention: Kinesis.

---

## 19. Scenario 18 — SaaS Partner Integration (EventBridge Partner Sources)

### 19.1 The problem

SaaS tools (Stripe, Zendesk, Datadog, GitHub, PagerDuty, etc.) emit events. Want to consume them in your AWS account without writing webhook receivers.

### 19.2 EventBridge Partner Buses

```
SaaS partner publishes to a partner event bus in your AWS account.
Your rules route partner events to your targets.
```

Partners: 50+ at this point. Stripe, Twilio, Shopify, Datadog, GitHub, PagerDuty, Zendesk, Auth0, etc.

### 19.3 Why this matters

Without it: build webhook receivers (HTTP endpoints, retries, signature verification, etc.). With it: AWS handles ingestion; you handle business logic.

```
Stripe → Partner Bus → Rule → Lambda (process invoice paid)
       → Rule → SNS Topic (notify ops)
       → Rule → SQS (queue for analytics)
```

### 19.4 Filtering

Standard EventBridge content filters apply. Filter by Stripe event type, customer, amount, etc.

### 19.5 Cross-account partner buses

Partner buses can be cross-account: route from your central account to spoke accounts.

### 19.6 Trade-offs

| Approach | Trade |
|---|---|
| **EventBridge partner bus** | Zero infra; partner-specific |
| **Custom webhook receiver** | Universal; engineering cost |
| **3rd-party (Hookdeck, Pipedream)** | Outsourced; vendor cost |

### 19.7 What I'd actually do

For supported partners: **partner bus**. Free integration. For unsupported: custom webhook receiver behind API Gateway → SQS → Lambda.

---

## 20. Scenario 19 — Async API / Job Submission

### 20.1 The problem

User submits a long-running job ("export 1M rows", "transcode video", "run report"). API can't block. Must accept, run async, notify on completion.

### 20.2 The pattern

```
1. POST /jobs → API server validates → writes to SQS + DynamoDB (job_id, status=PENDING).
2. Returns job_id immediately.
3. Worker pulls from SQS → runs job → updates DynamoDB (status=COMPLETE, result_url).
4. EventBridge event "JobCompleted" → notification (email, push, websocket).
5. Client polls GET /jobs/{id} OR receives push.
```

Standard async API pattern.

### 20.3 The notification subsystem

```
On completion:
  - EventBridge event with job_id + user_id + result.
  - Subscribers:
    - Lambda → SES email
    - Lambda → Mobile push
    - WebSocket service → push to active connection (if any)
    - Internal analytics
```

### 20.4 Trade-offs and alternatives

| Approach | Trade |
|---|---|
| **SQS + worker + DynamoDB status** | Standard; well-understood |
| **Step Functions** | Visual, retries, but per-step cost |
| **Lambda + Lambda + ...** | Simpler at low volume |
| **EKS jobs + queue** | For long-running compute |

### 20.5 What I'd actually do

Standard pattern: **SQS → ECS/Lambda worker → DynamoDB status + EventBridge completion event → notification fan-out**.

---

## 21. Scenario 20 — Migrating From Synchronous Monolith to Events

### 21.1 The problem

Existing synchronous monolith. Slow, brittle, can't scale individual workflows. Migrate to event-driven without big-bang rewrite.

### 21.2 The strangler pattern

```
1. Identify event boundaries: places where current code calls another module.
2. Replace direct call with: emit event + listen for response (or fire-and-forget).
3. Run side-by-side: legacy code path still works; new code path also processes events.
4. Gradually shift load to event path.
5. Remove legacy code.
```

Each migration is independent; no big-bang.

### 21.3 The dual-write trap (revisited)

When transitioning, you might write to both: old DB + new event. Synchronization issues. Fix: outbox pattern.

### 21.4 Choreography vs orchestration during migration

For migration, **orchestration** wins:
- Visible state: see exactly which step is happening.
- Easy to add new steps.
- Easy to compensate on failure.

Use **Step Functions** as the orchestrator for migrated workflows; Step Functions calls Lambdas / events as needed.

### 21.5 What I'd actually do

For a monolith → events migration:
- Catalog the workflows.
- One workflow at a time; Step Functions orchestrates each.
- Outbox pattern in monolith DB for events.
- EventBridge bus as the new mesh.
- Quarter-by-quarter migration with metrics on adoption.

---

## 22. Performance and Scaling Deep Dive

### 22.1 Throughput limits

| Service | Per-account default | Burst allowed | Raise via |
|---|---|---|---|
| **SNS Standard publish** | 100K+ TPS | yes | support |
| **SNS FIFO publish** | 300 TPS | 3K (high-throughput); 1M (max) | support |
| **SQS Standard** | unlimited | n/a | n/a |
| **SQS FIFO** | 300 / 3K (high-throughput) / 70K | yes | support |
| **EventBridge PutEvents** | 10K events/sec | yes | support |
| **EventBridge rule invocations** | 18,750 invocations/sec/rule | yes | support |
| **EventBridge Scheduler** | 14M schedules/region | yes | support |
| **Kinesis shard** | 1K records/sec write; 2 MB/sec read | n/a | add shards |

### 22.2 Burst tactics

- **SQS**: built for unlimited; just publish.
- **SNS Standard**: 100K+ TPS; AWS auto-scales; for sustained > 100K request capacity ahead.
- **EventBridge**: contact AWS for limit increase; default 10K is low.

### 22.3 Latency

```
SQS:           ~100-200 ms p50; can be ms with long polling on warm consumer.
SNS:           ~100 ms publish; delivery latency depends on subscriber.
EventBridge:   ~500 ms p50 from PutEvents to rule target invocation.
                Higher for cross-region.
                Cold-start spikes possible.
```

EventBridge is **slower than SNS/SQS** because of the rule-matching engine. For latency-critical event paths, SNS direct or SQS direct is faster.

### 22.4 Batching

```
SNS PublishBatch:    up to 10 messages per call.
SQS SendMessageBatch: up to 10 messages.
SQS ReceiveMessage:  up to 10 messages.
EventBridge PutEvents: up to 10 events per call.
```

Batching reduces request count → reduces cost AND latency overhead.

---

## 23. Cost Engineering

### 23.1 The cost model

```
SNS:  $0.50/M publish + $0.06-2/M delivery (varies by protocol).
SQS:  $0.40/M (Std), $0.50/M (FIFO).
EventBridge: $1/M custom events; AWS-source events on default bus free.
```

### 23.2 Where costs surprise

```
1. Forgotten DLQs collecting messages forever → no $$$ but messy ops.
2. EventBridge rules: each rule with no filter matches all events × delivery cost.
3. SNS HTTP/HTTPS deliveries: $0.60/M (10× SQS deliveries).
4. SNS retry storm to bad endpoint: each failed retry counted.
5. SQS short-polling: 10× expected request count.
6. Cross-region replication of high-volume event streams.
7. Long polling without sleep: idle consumer burns $$ in API calls.
```

### 23.3 Optimizations

- **Long polling** always.
- **Batch sends and receives**.
- **Filter at the source** (EventBridge rule patterns; SNS message attributes) — don't deliver to consumer that filters out.
- **VPC endpoints** for SQS / SNS / EventBridge → save NAT Gateway costs.
- **Compress payloads** if large; or store body in S3 + send pointer.
- **Dedup at source** for SQS FIFO content-based dedup; don't pay for duplicates.

### 23.4 Cost monitoring

CloudWatch metrics, AWS Cost Explorer, S3 Storage Lens-equivalent for messaging (just normal cost dashboard). No "Storage Lens" for messaging; build dashboards.

---

## 24. Security Considerations

### 24.1 Encryption

```
SNS:        SSE with KMS (AWS-managed or customer-managed).
SQS:        SSE with KMS.
EventBridge: SSE with KMS for event content.
```

All three support KMS. Cross-account use requires KMS key permissions to align.

### 24.2 Access control

- **Resource policies** on topics/queues/buses for cross-account.
- **IAM** for in-account access.
- **Conditions** on policies (source IP, VPC endpoint, etc.).
- **VPC endpoints** to keep traffic private.

### 24.3 Audit

CloudTrail logs all management API calls (CreateTopic, etc.). For data events:
- SQS: not natively logged; consumer logs.
- SNS: not natively logged at publish/deliver level.
- EventBridge: PutEvents logged.

For sensitive event flows: log at consumer for audit trail.

### 24.4 PII / sensitive data

```
Don't put unmasked PII in event body unless:
  - End-to-end encrypted (you control key).
  - Compliance allows.
  - Bus has restricted access.

Better pattern: event has reference (user_id); fetch sensitive data via API with auth.
```

---

## 25. Anti-Patterns — Staff-Level Red Flags

### 25.1 Using SNS for queueing

SNS is push, not pull. No retention. If subscriber missed delivery (1-hour retry), gone. Use SNS for fanout to durable subscribers (SQS), not as a queue itself.

### 25.2 Using SQS for fanout

One queue can't be consumed by multiple independent groups. Each receive *removes* the message. Use SNS or EventBridge for fanout, with SQS per consumer.

### 25.3 EventBridge for high-throughput streaming

10K events/sec/account default; expensive at $1/M. For high-volume telemetry: Kinesis.

### 25.4 No DLQ

Silent message loss. Always configure.

### 25.5 No idempotency at consumer

At-least-once means dupes. Always design for them.

### 25.6 Visibility timeout < processing time

Double-processing. Set to 2× p99.

### 25.7 maxReceiveCount=1

A single transient failure → DLQ. Should be 3-10 typically.

### 25.8 Polling instead of long polling

10× wasted SQS calls. Always WaitTimeSeconds=20.

### 25.9 Single MessageGroupId in FIFO

Throughput capped at 300 TPS. Use group_id = entity_id for parallelism.

### 25.10 EventBridge rules without filters

Every event matches → costly. Filter as tightly as possible.

### 25.11 Cross-region without considering cost

Per-event cross-region transfer × fanout multiplier × volume. Plan deliberately.

### 25.12 Not testing replay

Replay tools that have never been used don't work. Practice.

### 25.13 SNS HTTPS to flaky endpoints

Built-in retries are limited (1 hr). Use EventBridge API destinations or custom dispatch.

### 25.14 Lambda + SQS without concurrency cap

One slow message → unbounded Lambda invocations × concurrency limit hit → outage.

### 25.15 EventBridge buses sharing across many domains

A single "central bus" for everything → coupling. Use bus-per-domain.

---

## 26. The Decision Framework

### 26.1 Step 0 — What's the contract?

- Pull or push? → SQS pull; SNS/EventBridge push.
- Order? → FIFO if strictly ordered per key.
- Replay? → EventBridge or stream service.
- Filtering? → EventBridge.
- Cross-account/region? → EventBridge native.
- Mobile push, SMS, email? → SNS / Pinpoint / SES.

### 26.2 Step 1 — Pick the right primitive

Apply §1.3 quick-rule.

### 26.3 Step 2 — Add buffering

Most async paths benefit from SQS between push source and pull consumer.

### 26.4 Step 3 — Plan for failure

DLQ on every async path. Retry policy. Idempotency at consumer. Replay tooling for EventBridge.

### 26.5 Step 4 — Plan for evolution

Schema versioning. Schema registry. Backward-compatible changes only.

### 26.6 Step 5 — Plan operational lifecycle

CloudWatch alarms on queue depth, age-of-oldest, DLQ count, errors. Dashboards. On-call playbooks.

### 26.7 Step 6 — Cost model

Estimate per-event cost. Identify fanout multipliers. Apply filtering early.

### 26.8 Step 7 — Security model

KMS encryption. Resource policies. VPC endpoints. Audit logs.

---

## 27. Mental Models a Staff Engineer Carries

1. **SQS = buffer; SNS = fanout; EventBridge = bus with filters and history.** Different tools.

2. **Always at-least-once.** Design idempotent consumers. No exceptions.

3. **Visibility timeout > processing time.** Else, doubles.

4. **DLQ + alarm + replay tool** for every async path.

5. **FIFO is for ordering correctness, not by default.** Standard + idempotency wins for most.

6. **MessageGroupId is the parallelism unit.** Pick wisely.

7. **EventBridge for cross-account / cross-domain.** SNS is in-account.

8. **EventBridge replaces SNS in modern stacks.** Except for mobile push, SMS.

9. **SNS+SQS fanout is the workhorse pattern.** Buffered, isolated, replay-able.

10. **Filter as early as possible.** EventBridge rule beats consumer-side filter.

11. **Long polling, always.** Saves 10× cost.

12. **Batch publishes and receives.** 10× efficient.

13. **SQS as queue; not as DB or cache.** Use right tool.

14. **EventBridge Pipes for source-to-target without code.** Cheaper than equivalent Lambda.

15. **EventBridge Scheduler for cron and one-shot.** Replace polling.

16. **Step Functions for workflows; events for fan-out.** Don't blur.

17. **Cross-region replication is opt-in.** Don't replicate everything everywhere.

18. **Schema evolution is forever.** Version events; backward-compat only.

19. **Replay is a feature.** EventBridge Archive on every prod bus.

20. **Cost compounds with fanout.** 1M × 50 subscribers × $0.50/M deliveries adds up.

21. **VPC endpoints save NAT Gateway costs.** Free; enable always.

22. **Shard for tenant isolation.** Per-tenant queues are nightmare; sharded queues are right.

23. **SNS HTTPS subscriptions are weak retry.** Custom dispatch tier for serious webhooks.

24. **Mobile push is SNS-only.** Pinpoint adds segmentation.

25. **SQS FIFO ≠ Kafka.** 70K TPS aggregate ceiling. For higher: Kinesis/MSK.

26. **EventBridge latency is higher than SNS.** For latency-sensitive, SNS direct.

27. **Boring is a feature.** A 10M-event/sec architecture that quietly works is the goal.

---

## 28. Closing Notes

The three services exist because no single primitive covers all async patterns. Staff-level mastery means knowing:

- The **shape** of each service.
- The **failure modes** (visibility timeout pathology, FIFO group hot-spotting, EventBridge throttling, SNS retry storms).
- The **cost shape** at your scale.
- The **right combinations**: SNS+SQS for fanout-with-buffer, EventBridge+SQS+Lambda for filtered async work, EventBridge+Step Functions for orchestrated workflow.

Most outages I've seen in event-driven AWS architectures come from a small set of issues: missing DLQs, non-idempotent consumers, wrong visibility timeout, blanket FIFO, unbounded Lambda concurrency. Avoid those and you'll have boring, scalable, debuggable async infrastructure.

The art is choosing precisely. The mistake is using only the one you know.

> Companion docs:
> - `awsS3ScenariosAtScale.md` — S3 patterns including event triggers.
> - `lambdaStepFunctionsAtScale.md` — compute layer that processes events.
> - `eventPlatformsAtScale.md` — broader event platforms (Kafka, Pulsar).
> - `databaseTransactionScenarios.md` — outbox pattern integration.
> - `statelessSystemsAtMAANGScale.md` — stateless tiers consuming events.

The four together describe the AWS-native event-driven mesh.