# Event-Driven Platforms at Scale: Kafka, Pulsar, Redis, SQS

Staff-level playbook for designing event/messaging backbones at scale. Covers when to pick each system, realistic **combinations** in production, how to **autoscale** the stateful parts, and how to handle **multi-system transactions** (the hardest problem).

---

## Table of Contents

1. The Four Systems — Honest Characterization
2. Picking One: Decision Matrix
3. Combination Patterns (Kafka + Redis, Kafka + SQS, Pulsar + Redis, etc.)
4. Scenario Catalog (10 realistic scenarios)
5. Autoscaling the Cluster
6. Multi-Transaction / Multi-System Atomicity
7. Cross-Cutting: Ordering, Idempotency, Delivery Semantics
8. Corner Cases at Scale
9. Decision Cheat Sheet

---

## 1. The Four Systems — Honest Characterization

### Apache Kafka
- **Model**: partitioned log. Producers write to partitions; consumers pull from offsets; retention is time/size-bound (can be infinite with tiered storage).
- **Ordering**: per-partition.
- **Delivery**: at-least-once by default; exactly-once within Kafka transactions.
- **Throughput**: millions of messages/sec per cluster.
- **Pain**: stateful brokers — scaling is painful; partition rebalances are disruptive; cluster ops are an ongoing tax. KRaft (ZooKeeper-less) improved this; still not trivial.
- **Best for**: event sourcing, CDC, streaming ETL, high-throughput pub-sub where consumers need replay.

### Apache Pulsar
- **Model**: brokers (stateless compute) + BookKeeper (storage) + ZooKeeper (metadata). Topics have partitions but can also be non-partitioned.
- **Ordering**: per-partition.
- **Delivery**: at-least-once; effectively-once with deduplication; exactly-once with transactions.
- **Subscriptions**: exclusive, shared (queue-like), failover, key-shared (ordered per key without full partition lock). This is Pulsar's killer feature.
- **Tiered storage**: hot on BookKeeper, cold on S3/GCS, transparent to consumers.
- **Geo-replication**: built-in, first class.
- **Multi-tenancy**: namespaces + tenants with quotas and policies.
- **Pain**: more moving parts; smaller community; operational learning curve.
- **Best for**: multi-tenant platforms, geo-distributed events, workloads that want queue-semantics and stream-semantics from the same broker.

### Redis (Streams / Pub/Sub)
- **Model**:
  - **Pub/Sub**: at-most-once, no persistence, fire-and-forget fan-out.
  - **Streams** (since 5.0): append-only log with consumer groups, pending-entry-lists (PEL), XCLAIM for work stealing, XADD/XREAD.
- **Ordering**: per-stream.
- **Delivery**: Streams — at-least-once. Pub/Sub — at-most-once.
- **Throughput**: very high (in-memory), typically 100K–1M ops/sec/shard.
- **Retention**: memory-bound; `MAXLEN` or `MINID` trimming.
- **Pain**: no built-in durability beyond memory + AOF/RDB; cluster mode adds slot sharding complexity; large streams eat RAM.
- **Best for**: low-latency ephemeral events, cache-adjacent messaging, in-memory work queues, pub-sub to web clients, short-lived task fanout.

### AWS SQS
- **Model**: managed distributed queue. Standard (at-least-once, best-effort ordering) or FIFO (exactly-once, strict ordering per group).
- **Delivery**:
  - **Standard**: at-least-once; unordered; unlimited throughput.
  - **FIFO**: exactly-once dedupe (5 min window); ordered per MessageGroupId; 3000 msg/s per API (300 msg/s per group with batching).
- **Retention**: up to 14 days.
- **Fanout**: SNS → multiple SQS queues is the canonical fanout pattern.
- **Pain**: no native replay, no topic semantics, 256 KB message limit (with extended client for S3 pointers up to 2 GB).
- **Best for**: AWS-native async processing, per-consumer backlogs, serverless consumers (Lambda), simple durability without operating a cluster.

### Quick orthogonal comparison

| Dimension | Kafka | Pulsar | Redis Streams | SQS |
|---|---|---|---|---|
| Unit of parallelism | Partition | Partition (or key-shared) | Consumer | Receiver |
| Managed option | MSK / Confluent / WarpStream | StreamNative / DataStax | ElastiCache / Upstash | Fully managed |
| Replay | Yes (retention) | Yes (tiered) | Yes (stream length) | No (per-message redrive only) |
| Per-consumer backlog | Consumer group offsets | Subscription cursor | Consumer group PEL | Native (queue) |
| Ops cost | High (stateful) | Medium | Low–Medium | None |
| Latency (median) | 5–15 ms | 5–15 ms | < 1 ms | 10–50 ms |
| Throughput ceiling | 10M+ msg/s | Similar | 1M ops/s per shard | Per-queue ~100K/s (Standard) |
| Cost at scale | Efficient | Efficient | Expensive (RAM) | Expensive per message |
| Ordering primitive | Partition | Key-shared subscription | Stream | FIFO group |

---

## 2. Picking One: Decision Matrix

| Requirement | Pick |
|---|---|
| Event sourcing / CDC / replay long history | Kafka or Pulsar |
| Multi-tenant platform with geo-replication | Pulsar |
| Need queue + stream semantics in same broker | Pulsar (key-shared sub) |
| Serverless AWS consumers, simple per-service queues | SQS |
| Sub-millisecond fanout, ephemeral | Redis Pub/Sub |
| Sub-millisecond fanout with at-least-once | Redis Streams |
| Team can't run stateful clusters | SQS or a managed Kafka/Pulsar |
| Petabyte-scale log with cheap cold storage | Pulsar tiered / Kafka tiered storage / WarpStream |
| Minimum ops burden | SQS > Redis managed > Pulsar managed > Kafka managed |
| Polyglot clients across clouds | Kafka (biggest ecosystem) |
| Strict FIFO per-entity (e.g., per-user order events) | Kafka partitioned by user_id, Pulsar key-shared, or SQS FIFO |

---

## 3. Combination Patterns

Real systems rarely use one. Here are the seven combinations that show up again and again.

### Combo 1: Kafka (durable log) + Redis (low-latency fanout)

**When**: ingestion must be durable and replayable, but downstream consumers need sub-ms delivery (web UIs, real-time dashboards).

```
Producers → Kafka (durable, partitioned)
              │
              ├── Stream processor → sinks (ETL)
              └── Fanout service → Redis Pub/Sub → WebSocket servers → users
```

- Kafka keeps the source of truth; Redis is the delivery optimizer.
- Redis fan-out is at-most-once; UI clients tolerate occasional misses or request replay via Kafka on reconnect.
- Redis Streams instead if you need reliable per-client delivery (at-least-once with ack).

**Corner cases**
- **Reconnect replay**: client reconnects to WebSocket, must not miss events. Pass last-seen event ID; server reads from Kafka offset, not Redis.
- **Fanout service crash**: Redis is volatile; new fanout instance boots and starts from latest Kafka offset to avoid replaying history to clients.

### Combo 2: Kafka (ingress) + SQS (per-service backlog)

**When**: you want Kafka's throughput at the edge, but downstream microservices want simple per-service retry semantics without learning Kafka.

```
Events → Kafka → Bridge service → SNS → SQS(svc_a), SQS(svc_b), SQS(svc_c)
                                            ↓
                                          Lambda/worker per service
```

- Each service owns its SQS queue, its DLQ, its redrive policy.
- Kafka is shielded from consumer misbehavior (no poison pill ever stalls Kafka).
- Bridge service is the translation layer: Kafka consumer → SNS publisher.

**Corner cases**
- **Bridge is a single failure point**: run 3+ instances in a consumer group; health-check + auto-heal; size so one instance can handle full load briefly.
- **Backpressure into Kafka**: if SQS fails, bridge stops committing offsets; Kafka builds up; monitor consumer lag and SQS `SendMessage` errors independently.
- **256 KB SQS limit** vs Kafka (default 1 MB, tunable): use SQS Extended Client (stores payload in S3, sends pointer).

### Combo 3: Pulsar + Redis

**When**: multi-tenant platform (Pulsar namespaces = tenants) with real-time dashboards (Redis for per-session cache / Pub/Sub).

```
Tenant producers → Pulsar (tenant/namespace) → processors → Redis (per-tenant view cache)
                                                             │
                                                             └── Clients
```

- Pulsar's key-shared subscription keeps per-entity ordering without dedicated partitions per entity.
- Redis stores the "latest view" per entity; clients subscribe to a Redis key with pub/sub for invalidation.

**Corner cases**
- **Redis per-tenant isolation**: use logical DBs or key prefixes; at scale shard Redis per tenant or tenant-bucket.
- **Tiered storage lag**: consumers reading from cold tier hit S3 latency; isolate cold-tier-heavy consumers so they don't starve hot consumers.

### Combo 4: SQS (ingress) + Kafka (analytics)

**When**: operational work is simple (queue + Lambda), but analytics wants replay + retention + stream processing.

```
Producers → SQS → worker (process + publish) → Kafka → Flink/Spark → warehouse
```

- Operational service keeps its simple SQS+Lambda pattern.
- Worker double-writes: processes the job and publishes an event to Kafka for analytics.
- Analytics never blocks operations.

**Corner cases — this combo has the dual-write problem** (see §6). Use the outbox pattern if Kafka publish failure would cause permanent divergence.

### Combo 5: Kafka + Pulsar (migration or bridge)

**When**: you inherit both (mergers, multi-team decisions), or migrating between them.

```
Legacy → Kafka ──┐
                 ├── MirrorMaker / Pulsar IO source → Pulsar (new platform)
New →  Pulsar ───┘
```

- Use Pulsar IO connectors or Kafka MirrorMaker 2 for bridging.
- Maintain schema registry alignment (Confluent SR ↔ Pulsar SR).
- Migration direction: usually new system consumes from old via bridge; cut over producer-by-producer.

**Corner cases**
- **Offset/message ID translation**: Kafka offsets vs Pulsar MessageIds are different; track cross-system checkpoints for replay.
- **Double-delivery during cutover**: idempotent consumers on both sides or a window where both are read with dedupe.

### Combo 6: SQS FIFO + Redis (per-entity serialization)

**When**: you need strict per-entity ordering but SQS FIFO's 300 msg/s per group hits a limit.

```
Producers → Redis sorted set (per-entity queue, ordered by timestamp)
            │
            └── Worker pops per-entity with distributed lock → processes
```

- Redis gives unlimited per-entity throughput with ordered processing.
- Durability is the tradeoff; pair with AOF (`appendfsync everysec`) and replication, or log messages to durable store first.

**Corner cases**
- **Lock starvation**: entity lock held too long; use lease-based locks (Redlock or equivalent) with expiration.
- **Redis failover mid-processing**: pending entity work may replay; idempotency handles it.

### Combo 7: Three-tier (Kafka + Pulsar + Redis)

**When**: mega-scale platforms (e.g., Uber, LinkedIn, Netflix) with distinct needs per tier.

```
Edge ingestion → Kafka (cheap, massive throughput, retention)
                     │
                     ├── Stream processor
                     │
                     └── Pulsar (multi-tenant routing, geo-replicated)
                                │
                                └── Redis (hot data, sub-ms delivery to clients)
```

- Each tier does what it's best at.
- Rare for most teams — only justified when the scale and diversity of use cases warrants the operational complexity.

---

## 4. Scenario Catalog

### Scenario 1: IoT telemetry at 10M events/sec

**Requirements**: 1M devices, 10 events/s each, lossy OK, analytics + alerting + storage.

**Design**
- **Ingress**: MQTT broker (HiveMQ / EMQX) → Kafka (partitioned by device_id hash mod N).
- **Alerting path**: Flink consumer on Kafka → rules engine → SNS/PagerDuty.
- **Storage**: Kafka Connect → S3 (Parquet, partitioned by day/device_id_bucket).
- **Dashboards**: Flink aggregations → Redis (per-device latest).

**Why Kafka**: retention + throughput + partition-level parallelism matches device-centric sharding.

**Corner cases**
- **Hot partition** from one misbehaving device firing 1000 msg/s. Partition by `(device_id_bucket, device_id)` to spread; cap per-device rate at ingress.
- **Schema evolution**: 5 firmware versions in the field. Use Avro + Schema Registry with backward compatibility.

### Scenario 2: Order processing (saga across services)

**Requirements**: order.created → reserve inventory → charge payment → create shipment → confirm. Strict per-order ordering. Compensation on failure.

**Design**
- **Event store**: Kafka, partitioned by order_id (guarantees ordering per order).
- **Services**: each consumes a subset of event types, emits follow-on events.
- **State machine** per order tracked in a DB; event stream is the source of truth.
- **Compensation events** reverse prior steps.

**Alternative**: Step Functions (§4 of `lambdaStepFunctionsAtScale.md`) if you're in AWS and want orchestration rather than choreography. Kafka-native sagas trade simplicity for operational visibility.

**Corner cases**
- **Out-of-order delivery to a consumer**: only in bug scenarios (partition reassignment race). Detect via event sequence numbers; reject and alert.
- **Duplicate processing**: idempotent services keyed on `(order_id, event_type)`.

### Scenario 3: CDC pipeline (DB → Kafka → consumers)

**Requirements**: Postgres changes flow to search index, cache invalidation, analytics warehouse.

**Design**
- **Debezium** (Kafka Connect source) reads WAL → Kafka topic per table, partitioned by PK.
- **Consumers**: search indexer, cache invalidator, BigQuery sink.
- **Schema registry** for evolving table schemas.

**Corner cases**
- **Snapshot-and-stream transition**: initial snapshot is huge; Debezium locks risks. Use incremental snapshots (Debezium 1.6+).
- **DDL changes**: column drops break downstream. Gate schema migrations on consumer readiness; use compat mode on SR.
- **Long transactions in source DB**: WAL position doesn't advance; consumer sees nothing; eventually catches up in a burst. Monitor WAL lag.

### Scenario 4: Notifications platform (email/push/SMS)

**Requirements**: 100M users, personalized notifications, strict per-user rate limits, multiple channels with different SLAs.

**Design**
- **Event ingress**: services publish `notification.requested` to Kafka, partitioned by user_id.
- **Router**: consumes, applies preferences + rate limits (Redis), emits `notification.dispatched` per channel.
- **Per-channel queue**: SQS per channel (email, push, sms) → worker → provider.
- **Rate limiting**: Redis token bucket keyed on `(user_id, channel)` with Lua script for atomicity.

**Why this combo**: Kafka's ordering per user → Redis for per-user rate state → SQS per channel for per-provider backoff isolation.

**Corner cases**
- **Provider outage**: SendGrid down → SQS backs up → DLQ after redrive max → retry queue with delay. Don't let email failures poison push queue.
- **Time-sensitive vs time-insensitive**: separate priority queues; high-priority has dedicated workers.
- **User unsubscribes mid-flight**: dispatch check at send time (not at enqueue time); Redis preferences cache read per message.

### Scenario 5: Real-time financial tick data

**Requirements**: sub-ms latency to traders, loss-tolerance (next tick replaces previous), market-hours-bound.

**Design**
- **Feed handler → Redis Pub/Sub → trader WebSocket servers**.
- **Durable log**: feed handler also writes to Kafka for compliance + replay.
- **Redis Pub/Sub** (not Streams): at-most-once is fine; old ticks are obsolete.

**Corner cases**
- **Redis Pub/Sub doesn't buffer**: slow subscribers are dropped (`client-output-buffer-limit`). This is by design; trading apps handle it.
- **Re-connection replay**: trader reconnects → reads Kafka from last acked tick_id → then subscribes to Redis for live.

### Scenario 6: Multi-tenant SaaS event platform

**Requirements**: 1000s of tenants with wildly varying volumes (from 10 events/day to 100K/s).

**Design**
- **Pulsar**: each tenant = Pulsar tenant, each app = namespace. Quotas + backlog limits per namespace.
- **Key-shared subscriptions** for per-entity ordering without partition explosion.
- **Tiered storage** so idle tenants don't hold hot storage; instant replay even for old events.

**Why Pulsar over Kafka here**: Kafka's per-topic broker placement makes 10,000 topics expensive; Pulsar abstracts storage from broker.

**Corner cases**
- **Noisy tenant eats bookie bandwidth**: namespace rate limits + backlog size limits (`subscription-expiration-time`).
- **Per-tenant billing**: Pulsar stats exposed per namespace → directly attributable.

### Scenario 7: Stream processing with exactly-once (Kafka Streams / Flink)

**Requirements**: consume, process, emit results atomically; no duplicates, no loss.

**Design**
- **Kafka Streams** with EOS v2 (exactly-once semantics v2): transactional producer + `read_committed` isolation.
- **Flink**: 2PC commits to Kafka; checkpoints align with Kafka transaction boundaries.

**Corner cases** — see §6 in depth.

### Scenario 8: Log aggregation (logs → search + archive)

**Requirements**: 10 TB/day of logs, search recent, archive old.

**Design**
- **Agents (Vector/Fluentbit) → Kafka** (compressed, short retention ~3 days).
- **Consumers**: OpenSearch ingest (hot search), S3 archive (cold), anomaly detection.
- **Backpressure**: Kafka retention absorbs bursts; agents don't block apps.

**Corner cases**
- **Cardinality explosion in search**: drop high-cardinality fields at agent; avoid indexing them.
- **Topic per team vs shared topic**: shared topic with tenant tag = less overhead; per-team topic = isolation. Usually tenant tag wins operationally.

### Scenario 9: Task queue for background jobs

**Requirements**: 1M jobs/day, variable priorities, retries, delayed execution.

**Design**
- **Redis Streams with consumer groups** (simple, fast) or **SQS** (zero ops).
- For heavy features (scheduling, priority, rate limits): **Temporal / Celery / BullMQ** on top of Redis.

**Trade-off**: don't build Celery on Kafka — Kafka's mental model (partitioned log, offset commit) doesn't match job queue (per-item ack, retry with backoff).

### Scenario 10: Geo-distributed event replication

**Requirements**: events produced in US, consumed globally, sub-second lag.

**Design**
- **Pulsar geo-replication**: enabled at namespace level, async between clusters.
- **Kafka MirrorMaker 2**: configures cross-cluster replication with topic mapping.
- **Disaster recovery**: event consumers in DR region consume from local cluster, which is a replica.

**Corner cases**
- **Offset translation across clusters** (Kafka MM2): use translated offsets or use `__consumer_offsets` replication.
- **Duplicate on failover**: both regions see the same event; consumers must dedupe.
- **Write conflicts**: true multi-master is hard; usually designate one region per event stream as primary.

---

## 5. Autoscaling the Cluster

Autoscaling event platforms has three dimensions: **producers**, **consumers**, and **broker/storage**. The first two are relatively easy; the third is the hard problem because stateful.

### Kafka autoscaling

**Consumers (easy)**
- Run consumer groups on Kubernetes behind HPA.
- **Scaling signal**: consumer lag (not CPU). Use **KEDA** with the `kafka` scaler: `lagThreshold` (e.g., 1000 messages) triggers HPA.
- **Upper bound**: concurrent consumers per group ≤ partition count. Scaling past that does nothing. Plan partitions for 2–5× current consumer count.
- **Rebalance pain**: every scale-up/down triggers consumer group rebalance. With cooperative rebalancing (Kafka 2.4+), only affected partitions pause, not the whole group. Use it.

**Producers**: stateless, HPA on CPU/RPS is fine.

**Brokers (hard — stateful)**
- **Vertical scaling** is usually easier: bigger instance types, more disk, more memory.
- **Horizontal scaling**: add brokers, then **partition reassignment** to move data. This is days of background data movement at scale.
  - Tools: `kafka-reassign-partitions.sh`, **Cruise Control** (LinkedIn) for continuous rebalancing, or managed options (MSK Connect handles this).
- **Tiered storage** (Kafka 3.6+): offload older segments to S3 → broker disk stays small → scaling is mostly about compute.
- **WarpStream / AutoMQ**: newer "Kafka-compatible" systems that use S3 for storage; brokers are stateless, scale like a web service. Radically changes the ops model.

**Capacity signals to watch**
- Broker CPU
- Under-replicated partitions (indicator of I/O or network saturation)
- Request latency P99
- Disk usage / rate of growth
- Partition count per broker
- Network-in / network-out

**KEDA example for consumer autoscaling**
```yaml
triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka:9092
      consumerGroup: my-group
      topic: my-topic
      lagThreshold: "1000"
      offsetResetPolicy: latest
```

### Pulsar autoscaling

Easier than Kafka because compute (brokers) and storage (BookKeeper) are decoupled.

- **Brokers**: stateless compute, scale on CPU/memory with HPA. Rebalancing of topic ownership is fast (metadata move, not data move).
- **Bookies**: stateful; scale by adding nodes; BookKeeper's ensemble placement handles new bookies without rebalancing (new ledgers use new bookies).
- **Functions / IO**: stateless, scale like any K8s workload.
- **Subscription auto-scaling** via key-shared subscription adjusts consumer count dynamically.

### Redis autoscaling

- **Vertical**: bigger instance; easy with managed (ElastiCache). Downtime-less with replica promotion.
- **Horizontal (Cluster mode)**: add shards; resharding moves slots. Disruptive to clients if not using cluster-aware client.
- **Read scaling**: add read replicas; read-from-replica clients.
- **Consumer-side for Streams**: KEDA `redis-streams` scaler on PEL length (`pendingEntriesCount`).

### SQS autoscaling

- **Queue side**: zero ops, infinite throughput for standard queues.
- **Consumer side**:
  - **Lambda**: event source mapping auto-scales up to `MaximumConcurrency`.
  - **ECS/K8s**: KEDA `aws-sqs-queue` scaler on `ApproximateNumberOfMessagesVisible`.
- **FIFO**: capped at 3000 msg/s (with batching) per queue. To scale, shard across multiple queues by routing key.

### Autoscaling signals — use the right metric

| System | Scale consumers on | Don't use |
|---|---|---|
| Kafka | Consumer lag | CPU (usually I/O-bound) |
| Pulsar | Backlog size per sub | CPU |
| Redis Streams | PEL length | CPU |
| SQS | Messages visible | CPU |

### Predictive vs reactive

- For predictable patterns (nightly jobs, business hours): **scheduled scaling** beats reactive.
- For spikes: keep a warm minimum 2× steady-state to absorb the first surge while reactive scaling kicks in.
- For absolute worst case: pre-provision for known events (sales, launches) rather than trusting autoscaling.

### Broker autoscaling — when to even try

Most teams should **not** autoscale brokers. Instead:
- Size for peak + 2× headroom.
- Manual scale-out quarterly based on capacity trends.
- Alert at 70% utilization so there's time for human-driven capacity planning.
- Revisit with tiered storage / S3-native systems where stateless brokers allow true elasticity.

---

## 6. Multi-Transaction / Multi-System Atomicity

This is the hardest problem in event-driven systems: "I need to update a DB **and** publish an event, atomically, across N systems."

### The Dual-Write Problem

```
// Anti-pattern
db.update(user)         // commits to Postgres
kafka.publish(event)    // publishes to Kafka
```

Failure modes:
- DB commit succeeds, Kafka publish fails → consumers never see event → divergence.
- Kafka publish succeeds, DB commit fails → consumers see an event that doesn't match DB → false facts.
- Crash between the two → either failure mode, randomly.

**Partial retry** doesn't save you: on retry you don't know which side succeeded.

### Solution 1: Transactional Outbox Pattern (the default answer)

Write DB update + event row to the same DB transaction. A separate process reads the outbox table and publishes to the broker, marking rows as sent.

```sql
BEGIN;
UPDATE users SET email = 'new@x.com' WHERE id = 1;
INSERT INTO outbox (aggregate_id, event_type, payload, created_at)
  VALUES (1, 'user.updated', '{...}', now());
COMMIT;
```

A poller (or **Debezium CDC**) reads the outbox, publishes to Kafka, deletes/marks rows.

**Properties**
- DB is the single commit point — true atomicity.
- At-least-once publication (poller may retry after crash).
- Consumers must be idempotent (always true in event systems anyway).

**Corner cases**
- **Outbox table hot**: millions of rows/day.
  - Delete/archive aggressively; partition by day; use DB that tolerates high churn.
- **Ordering**: if consumers need per-aggregate ordering, poller reads in aggregate-id order; Kafka partition key = aggregate-id.
- **Poller lag**: events delayed when poller is slow/down.
  - Alarm on outbox age; pager duty on lag. Redundant pollers with leader election (DB row-lock).

This is the #1 pattern in production. Simple, robust, works across DBs and brokers.

### Solution 2: Kafka Transactions (broker-only atomicity)

If your "dual write" is actually Kafka+Kafka (read from A, write to B, commit offset), Kafka transactions give exactly-once semantics.

```java
producer.initTransactions();
producer.beginTransaction();
producer.send(topicB, record);                            // write to B
producer.sendOffsetsToTransaction(offsets, consumerGroup); // commit consumer offset for A
producer.commitTransaction();
```

- Consumers must use `isolation.level=read_committed` to skip aborted transaction messages.
- This **does not** extend to external systems (your DB).
- Use case: stream processing pipelines (Kafka Streams EOS, Flink with Kafka sink).

**Corner cases**
- **Zombie producer**: old instance comes back after failover; `transactional.id` fences it via epoch bump.
- **Hanging transactions**: transaction timeout (`transaction.timeout.ms`). Tune based on worst-case processing time.

### Solution 3: Pulsar Transactions

Similar to Kafka; spans multiple topics + subscriptions within Pulsar. Same external-system caveat.

### Solution 4: Saga (no atomicity, just compensations)

When atomicity isn't achievable (truly distributed, external systems), use a saga:
- Forward path: step 1 → step 2 → … → step N (each its own local transaction).
- On failure at step K: run compensations for steps 1..K-1 in reverse.

**Choreography saga**: each service emits events triggering the next.
- Pros: no central coordinator.
- Cons: harder to observe; spaghetti at 10+ services.

**Orchestration saga**: a coordinator (Step Functions, Temporal, Kafka Streams) drives each step.
- Pros: single source of truth for the workflow state.
- Cons: coordinator is a critical component.

**Corner cases**
- **Compensation idempotency**: refund twice → user double-refunded. Key compensations on `(saga_id, step)`.
- **Lost compensation**: process crashes between step success and compensation ack. Durable saga state + replay on recovery.
- **Semantic lock**: inventory reserved but not yet confirmed; other orders can't reserve it. TTL-based release if saga stalls.

### Solution 5: Two-Phase Commit (2PC) — generally avoid

XA-style distributed transactions across DB + broker. Kafka supports XA-like via transactions, but spanning to external systems requires coordinators (JTA / JTS) that don't scale well and are brittle.

**When it's tolerable**: very small number of systems, all within a tightly-coupled stack (e.g., two DBs in same cluster). At messaging scale, 2PC is a trap.

### Solution 6: Event Sourcing (eliminate the dual write)

Instead of "update DB + publish event", the **event is the state change**. Only one write: to the event log.

- Current state derived by folding events.
- No dual-write problem because there's only one write.
- Materialized views (DB, search index) are downstream projections, eventually consistent.

**When to use**: domains with audit requirements, complex temporal logic, natural event-centric modeling (finance, retail orders). Not a universal solution — adds complexity in reporting and query.

### Solution 7: Idempotent Receiver + At-Least-Once

The cheapest "atomicity" at scale: don't try for exactly-once. Publish at least once, make consumers idempotent.

- Consumer writes `(message_id → status)` in DB transaction with the business write.
- Duplicate delivery → `ON CONFLICT DO NOTHING`.
- No dual write because it's read-from-broker, write-to-DB-once.

Most practitioners end up here. Exactly-once is a limited-scope guarantee; idempotency is everywhere.

### Cross-system atomicity cheat sheet

| Scenario | Pattern |
|---|---|
| DB + Kafka | Outbox (Debezium CDC) |
| Kafka + Kafka stream processing | Kafka Transactions (EOS) |
| Pulsar + Pulsar | Pulsar Transactions |
| DB + External API | Saga with orchestrator (Temporal, SFN) |
| Multi-DB | Event sourcing or saga |
| Consumer processing with side effects | Idempotent consumer + at-least-once |
| Complex multi-service workflow | Orchestration saga (Temporal/SFN) |

### Multi-item "transaction" within a single Kafka transaction (the original question)

If the question is "I want to publish N messages atomically — either all visible or none":

```java
producer.initTransactions();  // once
// ... later ...
producer.beginTransaction();
for (record : records) {
    producer.send(record);
}
producer.commitTransaction();   // all-or-nothing
```

- Consumers with `isolation.level=read_committed` see all or none.
- Aborted transactions are invisible to committed consumers.

**Caveats**
- Transactions add latency (commit markers, coordinator roundtrip). Batch multiple messages per transaction, don't use a transaction per message.
- `transactional.id` must be unique per producer instance across restarts; coordinate via service registry or stable ID (instance index in k8s StatefulSet).
- Compacted topics + transactions: compaction may remove aborted messages' tombstones in unexpected orders; validate on your use case.

### Example: outbox pattern end-to-end (concrete)

```sql
-- Schema
CREATE TABLE orders (id UUID PRIMARY KEY, status TEXT, ...);
CREATE TABLE outbox (
  id BIGSERIAL PRIMARY KEY,
  aggregate_type TEXT,
  aggregate_id UUID,
  event_type TEXT,
  payload JSONB,
  created_at TIMESTAMPTZ DEFAULT now(),
  published_at TIMESTAMPTZ
);
CREATE INDEX ON outbox (published_at, id) WHERE published_at IS NULL;
```

```text
-- Application transaction
BEGIN;
  INSERT INTO orders VALUES ('uuid', 'placed', ...);
  INSERT INTO outbox (aggregate_type, aggregate_id, event_type, payload)
    VALUES ('order', 'uuid', 'order.placed', '{...}');
COMMIT;
```

Two options to ship events:

**Option A: Debezium CDC on outbox table** (preferred at scale)
- Debezium reads WAL → Kafka. No polling. Sub-second latency.
- Use Outbox Event Router SMT to shape event.

**Option B: Poller**
```text
loop:
  SELECT id, payload FROM outbox
    WHERE published_at IS NULL
    ORDER BY id
    LIMIT 100
    FOR UPDATE SKIP LOCKED;
  FOR each row: publish to Kafka.
  UPDATE outbox SET published_at = now() WHERE id IN (...);
```

- `FOR UPDATE SKIP LOCKED` lets multiple pollers run concurrently without contention.
- `ORDER BY id` preserves per-aggregate ordering if partitioned by aggregate_id.

---

## 7. Cross-Cutting: Ordering, Idempotency, Delivery Semantics

### Delivery semantics

| Semantic | Guarantee | System examples |
|---|---|---|
| At-most-once | Never duplicate; may lose | Redis Pub/Sub, Kafka with `acks=0` |
| At-least-once | Never lose; may duplicate | SQS Standard, Kafka default, Pulsar default |
| Exactly-once | Never lose, never duplicate | Kafka transactions (within Kafka), Pulsar transactions, SQS FIFO (within 5-min dedupe window) |

"Exactly-once" is scope-bounded — always ask "exactly-once between what and what?"

### Ordering guarantees

- Kafka: per-partition.
- Pulsar: per-partition; per-key via key-shared subscription (without single-consumer bottleneck).
- Redis Streams: per-stream.
- SQS FIFO: per MessageGroupId.
- SQS Standard: no ordering.

**Global ordering** is usually a smell. If you truly need it, you have one partition / one producer / one consumer — and no scale. Re-examine whether you really need global order or per-entity order.

### Idempotency patterns

- **Natural idempotency**: `SET user.email = 'x'` is naturally idempotent.
- **Business-key dedupe**: dedupe table keyed on `(event_id)` or `(aggregate_id, sequence)`.
- **Version-based**: only apply if `version > current_version`; reject stale updates.
- **Operation log**: write append-only log of applied operations; skip seen ones.

### Schema evolution

- **Backward-compatible** (new readers read old data): add optional fields.
- **Forward-compatible** (old readers read new data): add fields with defaults, don't reorder.
- **Full-compat**: both.
- Use a registry (Confluent Schema Registry, Pulsar Schema Registry, Apicurio) with compat mode.

---

## 8. Corner Cases at Scale

- **Zombie consumer after GC pause**: long pause → session expired → another consumer took the partition → zombie processes events as "live". Kafka fencing via `member.id` + transaction epochs; consumer must check before external side effects.
- **Compaction deleting still-needed data**: Kafka log compaction keeps latest per key. If your consumer is slow and compaction is aggressive, intermediate states disappear. Tune `min.compaction.lag.ms` and consumer SLAs.
- **Rebalance storm**: deploys + autoscale cause continuous rebalances, stalling consumption. Use cooperative rebalancing; scale with minimum frequency; static group membership for long-running consumers.
- **Large messages**: default 1 MB in Kafka. Don't raise it — instead, store payload in S3 and send a pointer. Large messages kill batching efficiency.
- **Unbounded fanout**: one event triggers 10 consumers, each emits 10 events, each triggers 10 consumers… amplification. Cap fanout depth; trace event lineage.
- **Topic explosion**: teams create one topic per thing → 10K topics → broker metadata overhead. Consolidate by type + use headers for routing.
- **Offset committed but processing not complete** (at-least-once gap): auto-commit with long processing → crash → committed but not processed → message lost. Use manual commit after processing finishes, or transactional outbox for side effects.
- **Reprocessing poison**: DLQ redrive without fixing the bug re-poisons the queue. Rate-limit redrive; gate on metric.
- **SQS retention expiry**: 14-day max. If a message ages out, it's gone. Monitor `ApproximateAgeOfOldestMessage`; archive before expiry if meaningful.
- **Kafka consumer memory**: fetch.max.bytes × concurrent partitions → OOM on consumer. Tune per consumer instance.
- **Cross-region replication loops** (MirrorMaker two-way without tagging): event in us-east replicated to eu-west, replicated back. Topic naming or header-based loop prevention.

---

## 9. Decision Cheat Sheet

| Problem | Solution |
|---|---|
| Millions of events/sec, long retention, replay | Kafka |
| Multi-tenant with geo-replication and queue+stream semantics | Pulsar |
| Sub-ms fanout, ephemeral, to many clients | Redis Pub/Sub |
| Sub-ms ordered work queue, in-memory | Redis Streams |
| Simple per-service async queue in AWS | SQS |
| Strict per-entity ordering | Kafka / Pulsar partitioned by entity, or SQS FIFO per group |
| DB + broker atomicity | Outbox pattern (Debezium CDC preferred) |
| Kafka-to-Kafka exactly-once | Kafka transactions |
| Distributed workflow with failures | Saga via Temporal / Step Functions / Kafka choreography |
| Autoscale consumers | KEDA on lag/backlog metrics |
| Autoscale brokers | Prefer manual + headroom; or use S3-backed Kafka variants |
| Cross-region replication | MirrorMaker 2 (Kafka) / built-in (Pulsar) |
| Hot partition | Add salt to partition key; accept ordering loss within the salted space |
| Poison pill | DLQ with `maxReceiveCount`; `BisectBatchOnFunctionError` for Lambda |
| Rebalance storm | Cooperative rebalancing + static membership + careful scale cadence |
| Runaway cost | Retention tuning + tiered storage + monitor topics without consumers |

---

## 10. The Staff-Level Truths

1. **Pick the messaging system for your blast-radius model, not just throughput.** Kafka stateful clusters make ops expensive; SQS makes per-consumer isolation cheap; Pulsar gives you multi-tenancy; Redis gives you latency. Throughput is rarely the deciding factor.
2. **Combinations are normal.** Nobody at scale runs a single messaging system. Layer them by concern: durable log + per-consumer queues + low-latency fanout.
3. **The dual-write problem is the most common distributed-systems bug.** Outbox pattern should be the default; don't publish from app code after DB commit.
4. **Exactly-once is a scope, not a guarantee.** Kafka EOS is exactly-once within Kafka. Your external side effects are still at-least-once. Plan idempotency accordingly.
5. **Consumer lag is the right autoscaling signal.** CPU is noise; lag is reality. KEDA is the Kubernetes-native standard.
6. **Broker autoscaling is a trap** for traditional Kafka. Either oversize + manual plan, or use S3-backed variants. Pulsar's stateless brokers are the exception.
7. **Per-partition ordering is what you actually need.** Global ordering is almost always a design smell.
8. **DLQ everything; monitor DLQ as a first-class metric.** Unattended DLQs silently lose data for quarters.
9. **Schema registry with compat checks is non-negotiable** at any scale where teams evolve independently.
10. **Kafka transactions ≠ DB transactions.** Don't hand-wave that equivalence; outbox or saga is the real answer for cross-system atomicity.