# Apache Flink — Realistic Scenarios at Staff-Engineer Depth

> A practical, opinionated reference for running Apache Flink at MAANG scale — true streaming with sub-second latency, exactly-once across heterogeneous sinks, stateful pipelines holding TBs of state, CEP for fraud detection, and operator drills that recover from full cluster loss in minutes. Written for engineers who already understand "Flink is a stream processor" and now have to defend Flink vs Spark vs Kafka Streams in design reviews, debug a checkpoint timeout in production, and explain why the new feature requested by product needs a savepoint migration.

> Companion to `sparkScenariosAtScale.md`, `snsSqsEventBridgeAtScale.md`, `eventPlatformsAtScale.md`, `druidScenariosAtScale.md`, `dynamoDBScenariosAtScale.md`, `awsS3ScenariosAtScale.md`. Flink owns the "true streaming + heavy state + exactly-once" niche of the stack.

---

## 0. The Staff-Level Frame

Flink earns its complexity. The cost of operating Flink — JobManagers, TaskManagers, checkpoints, savepoints, state backend tuning, watermark theory, exactly-once with two-phase commit — is real. The benefit is unique: nothing else gives you **per-event processing with strict event-time semantics, exactly-once guarantees through transactional sinks, and TBs of keyed state with sub-100ms latency**.

At staff level the questions are:

1. **Is the workload genuinely streaming-first?** (Sub-second latency, per-event semantics, continuous computation that runs forever.)
2. **Does it need event-time semantics?** (Out-of-order events, watermarks, late events handled correctly.)
3. **Is the state pattern tractable?** (Per-key state, bounded by watermark or TTL; not unbounded growth.)
4. **What's the failure / recovery contract?** (Exactly-once across which sinks? RTO measured in seconds, minutes, or hours?)
5. **What's the operational maturity?** (Flink isn't a click-and-go service. Savepoints, version upgrades, schema migrations, on-call expertise required.)
6. **Spark Structured Streaming vs Flink vs Kafka Streams vs Materialize?** (Each has its slot.)

The mistake everyone makes: choosing Flink because it's "the most powerful" without measuring whether the use case actually justifies the complexity. A nightly batch job dressed as streaming is not a Flink workload.

---

## 1. Mental Model — What Flink Is and Isn't

### 1.1 What Flink is

A **distributed stateful stream processor** with:
- **True per-event processing**: each event flows through operators independently; not batched.
- **Event-time semantics**: watermarks track logical progress; out-of-order events handled correctly.
- **Exactly-once state guarantees** via the Chandy-Lamport algorithm (asynchronous distributed snapshots = checkpoints).
- **Exactly-once end-to-end** via Two-Phase Commit Sinks (Kafka, Iceberg, Delta, JDBC with care).
- **State backends**: heap (in-memory) or RocksDB (disk-backed; can hold TBs).
- **Rich state APIs**: ValueState, ListState, MapState, ReducingState, AggregatingState; per-key, per-window, per-operator.
- **Windowing**: tumbling, sliding, session, custom triggers, allowed lateness, side outputs.
- **CEP library**: complex event pattern detection.
- **Flink SQL**: SQL over streams, stream-table duality.
- **Connectors**: Kafka, Kinesis, JDBC, file systems, Iceberg, Hudi, Cassandra, Elasticsearch, OpenSearch, Hive, MongoDB, etc.

Used at: Alibaba (originated heavy use), Uber, Netflix, Lyft, Stripe, Pinterest, ByteDance, Yelp, eBay, etc.

### 1.2 What Flink isn't

- **Not a database.** State is queryable via Queryable State (deprecated in 1.18) or by building a sink. Don't query Flink for OLTP.
- **Not a batch engine first.** Flink can do batch, but Spark is better at it.
- **Not micro-batch.** That's Spark Structured Streaming.
- **Not Kafka Streams.** KStreams is JVM-only, deeply Kafka-coupled, simpler. Flink is its own runtime.
- **Not free.** JobManager + TaskManager + state backend + checkpoint storage. Operationally heavier than Spark Streaming.
- **Not "streaming bolted on."** Streaming is the core; batch is the limit case (bounded streams).

### 1.3 The execution model

```
Job Graph (logical plan)
   ▼
Execution Graph (physical plan)
   ▼
Tasks (parallel instances of operators)
   ▼
TaskManagers (workers; each runs N task slots)
   ▼
JobManager (driver; schedules, manages state, checkpoints)

Each operator: parallelism N → N parallel tasks.
Tasks send records via network buffers (chained ops within same TaskManager skip network).

State per task:
  - Operator state (small; broadcast or list).
  - Keyed state (large; partitioned by key).

Checkpoint:
  - JobManager triggers periodic snapshot.
  - Each task snapshots its state asynchronously.
  - Checkpoints written to durable storage (S3/HDFS).
  - On failure: restore latest complete checkpoint.
```

### 1.4 The state pattern

```
Keyed state (most common):
  Stream is keyBy("user_id").
  Each parallel task owns a subset of keys.
  Per-key state stored locally (RocksDB or heap).
  
Operator state (less common):
  Per-task; not partitioned by key.
  Used for connectors (e.g., Kafka offsets) and broadcast state.

Broadcast state:
  Small data broadcast to all parallel tasks.
  Used for control / configuration / small lookup.
```

Most production Flink jobs are keyed-state heavy. Designing the key strategy = designing the application.

### 1.5 Watermarks — the heart of event-time

```
Event time: when the event happened.
Processing time: when Flink saw it.

Watermark W(t): "no events with event_time < t will arrive."
   Flink uses W to decide when to emit window results.

Strategies:
  - Bounded out-of-orderness: max event-time seen - X seconds.
  - Punctuated: per-record watermark (rare).
  - Monotonic: always-increasing event time.

Trade:
  - Tighter watermark (X small) = lower latency, more late events dropped.
  - Looser watermark (X large) = higher latency, fewer drops.
```

A staff-level skill: defending the watermark choice in design review. "5 seconds of out-of-orderness" must be backed by data on actual event arrival distribution.

### 1.6 Quick reference

| Aspect | Value / shape |
|---|---|
| **Latency** | ms to sub-second |
| **Throughput** | 100K to 10M+ events/sec/job |
| **State size** | GB (heap) to TBs (RocksDB) |
| **Checkpoint interval** | 30s to 5 min typical |
| **Checkpoint storage** | S3 / HDFS / GCS |
| **Recovery time** | Seconds to minutes (depending on state size) |
| **Cluster managers** | Kubernetes (operator), YARN, standalone, AWS Kinesis Data Analytics, Confluent Flink |
| **Languages** | Java/Scala (full); Python (PyFlink, growing); SQL (Flink SQL) |
| **Cost shape** | TaskManager EC2 + S3 checkpoint storage + cross-AZ |

### 1.7 Deployment options

| Option | Trade |
|---|---|
| **Self-managed K8s + Flink Operator** | Most control; ops heavy |
| **Confluent Cloud Flink (managed)** | Best Kafka integration; vendor-locked |
| **AWS Kinesis Data Analytics (now Managed Flink)** | AWS-managed; less customizable |
| **Ververica Platform** | Commercial; ex-Flink core team |
| **Self on YARN** | Mature; Hadoop-shaped |
| **Standalone** | Demos only |

For most teams: K8s + Flink Operator (controllable, modern) or AWS Managed Flink (less ops).

---

## 2. Scenario 1 — Real-Time Fraud Detection (CEP)

### 2.1 The problem

Credit card transactions stream. Detect:
- 3 transactions from same card in different countries within 60 seconds.
- 10 transactions in 2 minutes.
- A test transaction (small amount) followed by a large transaction within 10 minutes.

Latency budget: alert within 1 second of triggering event.

### 2.2 Why Flink fits

- Per-event semantics ✓
- Stateful by-card ✓
- Time-windowed pattern detection ✓
- Sub-second latency ✓

This is *the* canonical Flink use case.

### 2.3 The CEP pattern

```java
DataStream<Transaction> txns = env.fromSource(...);

Pattern<Transaction, ?> pattern = Pattern.<Transaction>begin("first")
    .where(SimpleCondition.of(t -> t.amount < 1.0))
    .next("second")
    .where(SimpleCondition.of(t -> t.amount > 1000.0))
    .within(Time.minutes(10));

PatternStream<Transaction> patternStream = CEP.pattern(
    txns.keyBy(Transaction::getCardId),
    pattern);

DataStream<Alert> alerts = patternStream.select(
    pat -> {
        Transaction first = pat.get("first").get(0);
        Transaction second = pat.get("second").get(0);
        return new Alert(first.cardId, "test+big pattern");
    });
```

Flink's CEP library (FlinkCEP) handles state, timeouts, partial-match progression. For each card, an NFA tracks possible matches.

### 2.4 The state size

```
N active cards (currently mid-pattern): each holds partial-match state.
  e.g., 100M cards × 0.1% mid-pattern × ~1 KB = 100 MB state.

Larger time windows / patterns = larger state.
```

For TBs of state: RocksDB backend mandatory.

### 2.5 Process function alternative

For complex custom patterns not expressible in CEP DSL:

```java
DataStream<Alert> alerts = txns
    .keyBy(Transaction::getCardId)
    .process(new KeyedProcessFunction<>() {
        ValueState<List<Transaction>> recentState;
        
        @Override
        public void processElement(Transaction t, Context ctx, Collector<Alert> out) {
            List<Transaction> recent = recentState.value();
            recent.add(t);
            // remove old
            recent.removeIf(prev -> ctx.timestamp() - prev.timestamp > 60000);
            
            // check pattern
            Set<String> countries = recent.stream().map(Transaction::getCountry).collect(toSet());
            if (countries.size() >= 3) {
                out.collect(new Alert(t.getCardId(), "multi-country"));
            }
            recentState.update(recent);
        }
    });
```

KeyedProcessFunction gives full control: per-key state, timers, side outputs, custom logic. The lower-level escape hatch when CEP DSL is too rigid.

### 2.6 Trade-offs and alternatives

| Approach | Trade |
|---|---|
| **Flink CEP** | Native; declarative; powerful |
| **Flink ProcessFunction** | Custom; full control; more code |
| **Spark Streaming + windows** | Workable; less expressive for sequences |
| **Custom rules engine** | Bespoke; engineering cost |
| **Esper / Drools (CEP)** | Single-node; doesn't scale |

### 2.7 What I'd actually do

For real-time fraud detection at MAANG scale:
- Flink with FlinkCEP for declarative patterns.
- Process functions for custom rules.
- RocksDB state backend.
- Sink to alerting service (Kafka → on-call notification system).
- Tuned watermark (event time) so detection is consistent across late-arriving payment events.

---

## 3. Scenario 2 — Stream-Stream Join (Orders + Payments)

### 3.1 The problem

Two Kafka topics:
- `orders` — order placed.
- `payments` — payment processed.

Match: same `order_id`, payment within 30 minutes of order. Output: enriched record. Cardinality: 1M orders/sec, 1.2M payments/sec at peak.

### 3.2 The stream-stream join

```java
DataStream<Order> orders = env.fromSource(ordersSource);
DataStream<Payment> payments = env.fromSource(paymentsSource);

DataStream<EnrichedOrder> joined = orders
    .keyBy(Order::getId)
    .intervalJoin(payments.keyBy(Payment::getOrderId))
    .between(Time.minutes(-5), Time.minutes(30))    // payment within [-5min, 30min] of order
    .process(new ProcessJoinFunction<Order, Payment, EnrichedOrder>() {
        @Override
        public void processElement(Order o, Payment p, Context ctx, Collector<EnrichedOrder> out) {
            out.collect(new EnrichedOrder(o, p));
        }
    });
```

### 3.3 The state

```
For each order: stored until window expires (max 30 min after order).
For each payment: stored until [-5 min, +30 min] window passes.

State size at 1M orders/sec × 30 min = 1.8B orders in state at peak.
At 1 KB each: ~1.8 TB state.

RocksDB mandatory. Significant disk needed on TaskManagers.
```

### 3.4 Watermarks

The join uses event time. Both streams must emit watermarks. The join's progress is bounded by the *minimum* of both watermarks — slowest stream gates the join.

Mitigations for slow stream:
- Idleness detection: if a stream has no events for X seconds, treat its watermark as "advanced."
- Out-of-order handling: accept late events with side output.

### 3.5 Outer joins / unmatched handling

Flink's IntervalJoin is inner. For outer (orders without payments / payments without orders):
- KeyedCoProcessFunction with explicit state and timers.
- Emit "unmatched" via side output after timeout.

```java
public void processElement1(Order o, Context ctx, Collector<EnrichedOrder> out) {
    orderState.update(o);
    ctx.timerService().registerEventTimeTimer(o.timestamp() + 30 * 60 * 1000);
}

public void processElement2(Payment p, Context ctx, Collector<EnrichedOrder> out) {
    Order o = orderState.value();
    if (o != null) {
        out.collect(new EnrichedOrder(o, p));
    } else {
        paymentState.update(p);
    }
}

public void onTimer(long ts, OnTimerContext ctx, Collector<EnrichedOrder> out) {
    Order o = orderState.value();
    if (o != null) {
        // unmatched after window
        ctx.output(unmatchedTag, o);
        orderState.clear();
    }
}
```

### 3.6 Trade-offs

| Approach | Trade |
|---|---|
| **Flink interval join** | Native; bounded state; inner only |
| **CoProcessFunction** | Full control; outer joins |
| **Spark stream-stream join** | Works; micro-batch latency |
| **Materialize / RisingWave** | Real-time SQL; smaller state scale |
| **Kafka Streams join** | JVM only; KSQL-friendly |

### 3.7 What I'd actually do

For stream-stream joins at scale: Flink interval join (with side outputs for unmatched), RocksDB state, S3 checkpoints. For complex outer logic: CoProcessFunction.

---

## 4. Scenario 3 — Real-Time Windowed Aggregation

### 4.1 The problem

Compute per-minute count of events per user. 10M events/sec aggregate. Sub-second latency for the latest minute.

### 4.2 The aggregation

```java
DataStream<Event> events = env.fromSource(...);

DataStream<MinuteCount> counts = events
    .keyBy(Event::getUserId)
    .window(TumblingEventTimeWindows.of(Time.minutes(1)))
    .aggregate(new CountAggregator());
```

`TumblingEventTimeWindows`: 1-minute non-overlapping. Each user × each minute → one count.

### 4.3 The watermark contract

```
Watermark advances → window closes → aggregate emits.

If watermark = bounded out-of-order 30 sec:
  Window for [12:00, 12:01) emits when watermark crosses 12:01:30.
  Latency: at least 30 sec.

Tighter watermark: lower latency; more dropped late events.
```

For ms-latency: tight watermark + side output for late events for reconciliation.

### 4.4 Allowed lateness

```java
.window(TumblingEventTimeWindows.of(Time.minutes(1)))
.allowedLateness(Time.seconds(30))
.sideOutputLateData(lateTag)
```

- After watermark passes, window remains open for `allowedLateness`.
- Late events update the window's result (re-emits).
- After lateness expires: events go to side output.

### 4.5 State size

```
Active windows × users:
  At any moment, ~2 active 1-min windows (current + lateness).
  100M users × 2 × ~32 bytes = ~6 GB state.

Manageable in heap; comfortable in RocksDB.
```

### 4.6 Output

```java
counts.addSink(KafkaSink.<MinuteCount>builder()
    .setBootstrapServers(...)
    .setRecordSerializer(...)
    .setDeliveryGuarantee(DeliveryGuarantee.EXACTLY_ONCE)
    .build());
```

Exactly-once Kafka sink uses Kafka transactions.

### 4.7 Trade-offs

| Approach | Trade |
|---|---|
| **Flink windowed aggregate** | Real-time; event-time correct |
| **Spark Structured Streaming** | Similar; micro-batch |
| **Druid streaming ingestion** | Aggregates as side effect of OLAP |
| **Materialize SQL** | Declarative; smaller scale |
| **ClickHouse + Kafka engine** | Eventual; cheaper |

### 4.8 What I'd actually do

For per-user real-time aggregates: Flink with event-time windows + allowed lateness. Sink to Kafka or downstream service. For dashboard display: feed to Druid for pre-aggregated query layer.

---

## 5. Scenario 4 — Sessionization with Custom Logic

### 5.1 The problem

User sessions: events from same user; new session if 30-minute gap. Output session metrics: duration, page count, conversion within session.

### 5.2 Session windows

```java
DataStream<SessionMetrics> sessions = events
    .keyBy(Event::getUserId)
    .window(EventTimeSessionWindows.withGap(Time.minutes(30)))
    .aggregate(new SessionAggregator());
```

Session window dynamically merges events into sessions based on inter-event gap.

### 5.3 The state

```
Active sessions per user.
At any moment: ~1 active session per user.
100M users × 1 KB session state = 100 GB.

RocksDB mandatory.
```

### 5.4 Custom session triggers

For non-standard sessions ("session ends on logout event"):

```java
events.keyBy(Event::getUserId)
    .process(new KeyedProcessFunction<>() {
        ValueState<Session> sessionState;
        
        public void processElement(Event e, Context ctx, Collector<SessionMetrics> out) {
            Session s = sessionState.value();
            if (s == null) s = new Session(e);
            else s.add(e);
            
            if (e.type == LOGOUT || timeoutExpired(s, ctx)) {
                out.collect(s.metrics());
                sessionState.clear();
                ctx.timerService().deleteEventTimeTimer(s.timeoutTime);
            } else {
                sessionState.update(s);
                ctx.timerService().registerEventTimeTimer(e.timestamp + 30 * 60_000);
            }
        }
        
        public void onTimer(long ts, OnTimerContext ctx, Collector<SessionMetrics> out) {
            Session s = sessionState.value();
            if (s != null && s.lastEventTime + 30 * 60_000 == ts) {
                out.collect(s.metrics());
                sessionState.clear();
            }
        }
    });
```

This is the canonical Flink low-level pattern: state + timers + custom logic.

### 5.5 Trade-offs

| Approach | Trade |
|---|---|
| **Flink session windows** | Standard; declarative |
| **Flink ProcessFunction** | Custom logic; more code |
| **Spark session_window** | Works; micro-batch |
| **Batch sessionization** | Cheaper; latency hours |

### 5.6 What I'd actually do

For sessionization with custom logic: Flink ProcessFunction with state + timers. For standard inactivity-based: session windows.

---

## 6. Scenario 5 — CDC Processing (Flink CDC Connector)

### 6.1 The problem

Postgres → real-time replicate to Iceberg warehouse. Apply inserts, updates, deletes.

### 6.2 Flink CDC

```sql
-- Source: Postgres
CREATE TABLE users_cdc (
    id INT,
    name STRING,
    email STRING,
    PRIMARY KEY (id) NOT ENFORCED
) WITH (
    'connector' = 'postgres-cdc',
    'hostname' = 'pg.host',
    'database-name' = 'mydb',
    'schema-name' = 'public',
    'table-name' = 'users',
    'slot.name' = 'flink_cdc_slot'
);

-- Sink: Iceberg
CREATE TABLE users_warehouse (
    id INT, name STRING, email STRING,
    PRIMARY KEY (id) NOT ENFORCED
) WITH ('connector' = 'iceberg', ...);

INSERT INTO users_warehouse SELECT * FROM users_cdc;
```

Flink CDC includes Debezium internally; reads Postgres WAL. Outputs change stream with op (INSERT/UPDATE/DELETE).

### 6.3 The streaming MERGE

Iceberg's V2 spec supports row-level deletes. Flink upserts via Iceberg sink with primary-key awareness.

```
On INSERT: append.
On UPDATE: emit delete (old row) + insert (new row); Iceberg compaction merges.
On DELETE: emit delete marker.
```

### 6.4 vs Spark CDC

```
Spark CDC: foreachBatch + MERGE; micro-batch (~minutes).
Flink CDC: per-event; sub-second; native upsert support.

For low-latency CDC: Flink wins.
For batch-friendly: Spark.
```

### 6.5 Initial snapshot + incremental

Flink CDC handles "snapshot-then-stream":
- Reads full table once (initial snapshot).
- Switches to log-based incremental.
- No gap; uses LSN markers.

### 6.6 Schema evolution

Flink CDC propagates schema changes from source. Iceberg sink applies them. Limited support for arbitrary changes; safer with backward-compatible.

### 6.7 Trade-offs

| Approach | Trade |
|---|---|
| **Flink CDC** | Real-time; incremental |
| **Debezium → Kafka → Flink/Spark** | Decoupled; more moving parts |
| **DMS** | Managed; less rich |
| **AWS Glue / Spark batch** | Cheaper; latency |

### 6.8 What I'd actually do

For real-time CDC at MAANG: Flink CDC native source + Iceberg sink. For decoupled: Debezium → Kafka → Flink (more flexible; engineering cost).

---

## 7. Scenario 6 — Real-Time Enrichment with Reference Data

### 7.1 The problem

Stream of events. Enrich with user attributes (from a database). Reference data: 10M users × 1 KB; updates ~1K/sec.

### 7.2 Pattern A: Broadcast state

For small reference data:

```java
MapStateDescriptor<String, User> userDescriptor = new MapStateDescriptor<>(
    "users", String.class, User.class);

BroadcastStream<User> broadcastUsers = userUpdates.broadcast(userDescriptor);

DataStream<EnrichedEvent> enriched = events
    .connect(broadcastUsers)
    .process(new BroadcastProcessFunction<>() {
        public void processElement(Event e, ReadOnlyContext ctx, Collector<EnrichedEvent> out) {
            User u = ctx.getBroadcastState(userDescriptor).get(e.userId);
            out.collect(new EnrichedEvent(e, u));
        }
        public void processBroadcastElement(User u, Context ctx, Collector<EnrichedEvent> out) {
            ctx.getBroadcastState(userDescriptor).put(u.id, u);
        }
    });
```

- Reference data broadcast to all parallel tasks.
- Updates propagated as broadcast.
- Limit: ~GB scale (broadcast state lives in heap of every task).

### 7.3 Pattern B: Async I/O for lookups

For larger reference data not broadcastable:

```java
DataStream<EnrichedEvent> enriched = AsyncDataStream
    .unorderedWait(
        events,
        new AsyncFunction<Event, EnrichedEvent>() {
            public void asyncInvoke(Event e, ResultFuture<EnrichedEvent> result) {
                redisClient.getAsync(e.userId).thenAccept(user -> {
                    result.complete(Collections.singleton(new EnrichedEvent(e, user)));
                });
            }
        },
        100, TimeUnit.MILLISECONDS, // timeout
        100); // max concurrent
```

- Async I/O lets Flink keep CPU busy while external lookup is in flight.
- 100 concurrent lookups per task: 100× single-threaded throughput.
- Reference data lives in Redis / DynamoDB / Cassandra.

### 7.4 Pattern C: Lookup join (Flink SQL)

```sql
-- Flink SQL temporal lookup join
SELECT e.*, u.name, u.country
FROM events e
JOIN users FOR SYSTEM_TIME AS OF e.proctime AS u
  ON e.user_id = u.id
```

`FOR SYSTEM_TIME AS OF` is the temporal join. The `users` table can be:
- JDBC (with caching).
- HBase / Hive / Redis / DynamoDB.
- Versioned table (point-in-time).

### 7.5 Trade-offs

| Approach | Trade |
|---|---|
| **Broadcast state** | Smallest data; in-memory; refreshed |
| **Async I/O** | Large data; external store; latency |
| **Lookup join (SQL)** | Declarative; depends on connector |
| **Stream-stream join** | If reference is itself a stream |

### 7.6 What I'd actually do

- Reference < 100 MB: broadcast state.
- Reference 100 MB – 100 GB: async I/O against Redis / DynamoDB.
- Reference > 100 GB: only the relevant subset; or batch-precompute.
- For SQL pipelines: lookup join.

---

## 8. Scenario 7 — Exactly-Once with Two-Phase Commit Sinks

### 8.1 The problem

Pipeline: Kafka → Flink → Kafka. Must guarantee exactly-once: each input event produces exactly one output event, regardless of failures.

### 8.2 The mechanics

```
Flink TwoPhaseCommitSinkFunction:
  Pre-commit phase: write data to sink in pending state (e.g., Kafka uncommitted transaction).
  Checkpoint barrier: included with the pending state.
  Commit phase: when checkpoint completes globally, sink commits transaction.
  Failure: rollback uncommitted; restore from last successful checkpoint.

Kafka 0.11+ supports transactions. Flink Kafka sink uses them.
```

### 8.3 Configuration

```java
KafkaSink<String> sink = KafkaSink.<String>builder()
    .setBootstrapServers(brokers)
    .setRecordSerializer(...)
    .setDeliveryGuarantee(DeliveryGuarantee.EXACTLY_ONCE)
    .setTransactionalIdPrefix("my-job-")
    .build();

env.enableCheckpointing(60_000); // 1-minute checkpoints
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
```

### 8.4 The cost

Exactly-once is not free:
- Kafka transactions add ~100ms latency per commit.
- Checkpoint interval bounds end-to-end latency: events committed only at checkpoint boundary.
- 1-minute checkpoint = up to 1-minute downstream visibility lag.

For lower latency: shorter checkpoint interval (e.g., 10 sec) but higher checkpoint overhead.

### 8.5 Sink compatibility

```
Kafka: transactional sink; exactly-once.
Iceberg: transactional sink; exactly-once.
Delta Lake: transactional via Iceberg-like commit.
JDBC: TwoPhaseCommit extension; databases must support 2PC (XA).
Cassandra / Elasticsearch: idempotent at-least-once; not transactional.
HTTP / generic: at-least-once; idempotency required at consumer.
```

For non-transactional sinks: at-least-once + idempotency keys.

### 8.6 The "consumer side of the chain" gotcha

Exactly-once Flink → Kafka means Kafka has the records exactly once. But:
- Downstream consumer must use `read_committed` isolation level (Kafka 0.11+).
- Otherwise: reads uncommitted records → duplicates if Flink retries.

Document this as part of the pipeline contract.

### 8.7 Trade-offs

| Guarantee | Latency | Setup |
|---|---|---|
| **At-most-once** | Lowest | Simplest |
| **At-least-once** | Low | Standard; idempotent consumer |
| **Exactly-once** | Higher (checkpoint-bound) | Transactional sink + checkpoint |

### 8.8 What I'd actually do

For financial / billing / critical data: exactly-once with Kafka transactions, 30-second checkpoints. For analytics / metrics: at-least-once with idempotent consumer.

---

## 9. Scenario 8 — Stateful Checkpointing at Scale

### 9.1 The problem

Job has 10 TB of keyed state. Checkpoints take longer than checkpoint interval → backpressure → OOM.

### 9.2 The state backend choice

```
HashMapStateBackend (heap):
  - State in JVM heap.
  - Fast access; bounded by available heap.
  - Synchronous checkpoint.
  - Best for < 100s of GB state.

EmbeddedRocksDBStateBackend:
  - State in RocksDB on local disk.
  - Larger state (TBs).
  - Asynchronous checkpoints (snapshot of SSTables).
  - Incremental checkpoints (only changed data).
```

For 10 TB: RocksDB mandatory.

### 9.3 Incremental checkpoints

```yaml
state.backend: rocksdb
state.backend.incremental: true
```

Flink only persists *changed* SST files since last checkpoint. For mostly-stable state, checkpoint size shrinks 10–100×.

### 9.4 Asynchronous snapshots

```
Synchronous: barrier → snapshot now → resume processing.
   Stops processing during snapshot. Bad for big state.

Asynchronous: barrier → spawn snapshot thread → resume processing.
   Snapshot writes in background; processing continues.
   Default in modern Flink.
```

### 9.5 Checkpoint storage

```
Local disk (RocksDB): staging.
Persisted to: S3, HDFS, GCS.

Throughput:
  10 TB / 60 sec = 170 GB/sec.
  Far beyond S3's per-bucket limits without prefix-spreading.

Mitigation: parallel paths + prefix randomization for very large states.
```

### 9.6 Checkpoint timeouts

```yaml
execution.checkpointing.interval: 60s
execution.checkpointing.timeout: 30min   # large state needs longer
execution.checkpointing.min-pause: 30s   # space between checkpoints
```

For 10 TB state: timeout 30 min; interval 1 min; min-pause 1 min. Tune for actual snapshot duration.

### 9.7 Savepoints — manual snapshots

```bash
flink savepoint <jobId> s3://savepoints/my-job-2026-04-28
```

Savepoint = full self-contained snapshot. Used for:
- Job upgrades (stop → savepoint → start with new code from savepoint).
- Code changes (state migration).
- Retention beyond checkpoint cycle.

Treat savepoints as version-controlled production state.

### 9.8 Recovery time

```
Restore from incremental checkpoint:
  Download from S3 + replay since checkpoint barrier.
  
For 10 TB state: 10 TB / 1 GB/sec download = 3 hours.
Recovery time = bounded by network throughput.

Mitigation:
  - Local recovery: TaskManagers cache state on local disk.
  - Cluster failover with shared state: avoid full re-download if possible.
```

### 9.9 Trade-offs

| Approach | Trade |
|---|---|
| **Heap state** | Fast; small |
| **RocksDB + incremental** | Big state; standard |
| **RocksDB + async snapshots** | Concurrent processing during checkpoint |
| **External state (queryable)** | Decouple; complex |

### 9.10 What I'd actually do

For multi-TB state at scale:
- RocksDB with incremental checkpoints.
- 1-min checkpoint interval.
- S3 with prefixed paths for storage.
- Savepoints daily for upgrade safety.
- Local recovery enabled.

---

## 10. Scenario 9 — Late Events and Side Outputs

### 10.1 The problem

Events arrive out of order. After window closes, late events still arrive. Need:
- Window result for "on-time" events.
- Late events captured separately for reconciliation.

### 10.2 Side outputs

```java
OutputTag<Event> lateTag = new OutputTag<Event>("late") {};

SingleOutputStreamOperator<Result> windowed = events
    .keyBy(Event::getUserId)
    .window(TumblingEventTimeWindows.of(Time.minutes(1)))
    .allowedLateness(Time.seconds(30))
    .sideOutputLateData(lateTag)
    .aggregate(new MyAggregator());

DataStream<Event> lateEvents = windowed.getSideOutput(lateTag);

// Send late events to a separate sink for reconciliation
lateEvents.addSink(...);
```

### 10.3 The reconciliation flow

```
Main sink: aggregated minute results, "as of" tight watermark.
Late side output: late events stored to S3 / Iceberg.
Daily batch job: merge late events into a "corrected" version of the table.
```

This is the **lambda architecture** within Flink: stream gives fast results; batch reconciliation corrects.

### 10.4 Other side output uses

```java
// Routing different event types
public void processElement(Event e, Context ctx, Collector<Event> out) {
    if (e.type == ERROR) {
        ctx.output(errorTag, e);
    } else if (e.type == AUDIT) {
        ctx.output(auditTag, e);
    } else {
        out.collect(e);
    }
}
```

Side outputs are the universal "second output" mechanism. Used for:
- Late events.
- Errors / dead-letter.
- Different downstream targets.
- Audit / debug.

### 10.5 Trade-offs

| Approach | Trade |
|---|---|
| **Side outputs** | Native; clean separation |
| **Multiple sinks via splits** | Less clean; older API |
| **Filter + multiple downstreams** | Duplicates the stream; expensive |

### 10.6 What I'd actually do

Side outputs for any "secondary output" need: late events, errors, special routing. Don't filter and split.

---

## 11. Scenario 10 — Backfill / Reprocess from Kafka

### 11.1 The problem

Bug in pipeline; need to reprocess last 7 days of Kafka data with fixed code.

### 11.2 The strategy

```
1. Take savepoint of current job.
2. Stop job.
3. Modify code (fix bug).
4. Start NEW job from savepoint OR from 7-day-ago Kafka offset.
5. New job runs as fast as possible to catch up (event time).
6. Eventually catches present; continues steady-state.
```

### 11.3 The "rewind from offset" mode

```java
KafkaSource<Event> source = KafkaSource.<Event>builder()
    .setStartingOffsets(OffsetsInitializer.timestamp(7daysAgoMs))
    .build();
```

Or for already-running job: use savepoint.

### 11.4 Catch-up performance

```
During backfill: job processes faster than real-time (no idle time between events).
Limited by: Flink throughput, sink throughput, Kafka read throughput.

For 1M events/sec sustained × 7 days × historical = 600B events.
At 5M events/sec catch-up rate: ~33 hours of replay.

Plan: provision more parallelism for catch-up; scale back after caught up.
```

### 11.5 Watermark behavior in catch-up

Watermarks advance from old to current. Operators see large jumps. Windows fire many at once. Plan for sink burst.

### 11.6 The "reprocess and merge" pattern

If you can't stop the live job:
- Run a parallel "reprocess" job from Kafka offset.
- Output to a *new* Iceberg table or Kafka topic.
- After validation: replace live data with reprocessed.

### 11.7 Trade-offs

| Approach | Trade |
|---|---|
| **Stop + savepoint + restart** | Clean; downtime |
| **Reprocess job in parallel** | No downtime; more complex |
| **Spark batch reprocess** | Sometimes simpler; loses Flink state |

### 11.8 What I'd actually do

For backfills: stop + savepoint + new code + restart. For zero-downtime: parallel reprocess job + merge.

---

## 12. Scenario 11 — Multi-Tenancy

### 12.1 The problem

Shared Flink platform serving 50 internal teams. Each team's job:
- Different topology.
- Different state size.
- Different SLA.
- Must not interfere with others.

### 12.2 Isolation models

```
1. Per-tenant cluster:
   Strongest isolation; ops cost (50 clusters).

2. Shared cluster, separate jobs:
   Each team submits a job; jobs run independently.
   Failure in one job doesn't affect others.
   But: shared TaskManager pool → resource contention.

3. Per-tenant TaskManager pool:
   Jobs allocated to dedicated TM pools.
   Resource isolation.
   Operational middle-ground.
```

For 50 teams: option 2 with K8s namespaces and CPU/memory quotas.

### 12.3 K8s Operator multi-tenancy

```yaml
# Per-tenant FlinkDeployment
apiVersion: flink.apache.org/v1beta1
kind: FlinkDeployment
metadata:
  namespace: team-a
  name: orders-pipeline
spec:
  taskManager:
    resource:
      memory: "8Gi"
      cpu: 4
  ...
```

Each tenant's namespace + ResourceQuota = enforced isolation.

### 12.4 Resource quotas

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  namespace: team-a
spec:
  hard:
    requests.cpu: "100"
    requests.memory: "400Gi"
```

Tenant cannot exceed allocated resources. K8s enforces.

### 12.5 Per-tenant savepoints / checkpoints

S3 prefix per tenant; per-tenant retention. Audit by namespace.

### 12.6 What I'd actually do

For platform Flink:
- K8s + Flink Operator.
- One namespace per team.
- Resource quotas enforced.
- Self-service deploy via team's own CD pipeline.
- Centralized monitoring + alerting.

---

## 13. Scenario 12 — Schema Evolution / State Migration

### 13.1 The problem

Add a field to the state object (e.g., new metric per session). Existing state has the old shape.

### 13.2 State serializers

```java
// Old:
public class SessionState {
    public long count;
    public long lastEvent;
}

// New (added a field):
public class SessionState {
    public long count;
    public long lastEvent;
    public Set<String> distinctPages = new HashSet<>();  // NEW
}
```

If using POJO serializer: Flink can evolve POJOs (adding/removing fields). Documented compatibility rules.

If using Avro / Protobuf: schema evolution rules apply (backward-compatible).

If using Kryo: limited evolution; likely needs migration.

### 13.3 The migration via savepoint

```
1. Take savepoint of running job.
2. Stop job.
3. Deploy new code (with new state shape; serializer must handle migration).
4. Start from savepoint.
5. State automatically migrates per serializer rules.
```

### 13.4 Custom state migrators

For non-trivial migrations (rename field, restructure):

```java
class MyTypeSerializerSnapshot extends TypeSerializerSnapshot<MyState> {
    public TypeSerializerSchemaCompatibility<MyState> resolveSchemaCompatibility(...) {
        // logic to migrate or reject
    }
}
```

Complex but supported. Flink calls migration during state restore.

### 13.5 The "flat-out replace" approach

When migration is too complex:
- Run new job in parallel from offset.
- Old job continues serving until new is verified.
- Switch traffic.
- Discard old state.

### 13.6 Trade-offs

| Approach | Trade |
|---|---|
| **POJO evolution** | Simplest; supported field changes |
| **Avro/Protobuf with rules** | Stricter; clearer compatibility |
| **Custom migration** | Universal; engineering cost |
| **Parallel-run replace** | Cleanest; double-cost during transition |

### 13.7 What I'd actually do

For state schema evolution: Avro for serialization (clear compatibility rules); savepoint + restart for migration. For incompatible changes: parallel-run replace.

---

## 14. Scenario 13 — Real-Time ML Feature Computation

### 14.1 The problem

ML model needs features computed in real-time:
- "Number of transactions in last 24 hours."
- "Average transaction amount last 7 days."
- "Time since last login."

Latency: ms to feature serving for inference.

### 14.2 The Flink pipeline

```java
DataStream<Transaction> txns = env.fromSource(...);

DataStream<Features> features = txns
    .keyBy(Transaction::getUserId)
    .process(new KeyedProcessFunction<>() {
        ValueState<Long> last24hCount;
        ValueState<Double> avg7d;
        ListState<Transaction> recent;
        // ... etc
        
        public void processElement(Transaction t, Context ctx, Collector<Features> out) {
            // Update sliding window aggregates
            // ...
            out.collect(new Features(t.userId, last24hCount.value(), avg7d.value(), ...));
        }
    });

features.addSink(redisSink);  // for online serving
```

Features computed per event; written to Redis / DynamoDB for inference layer to read.

### 14.3 Feast / feature store integration

```
Flink computes features → writes to online store (Redis/DDB).
ML model serving reads from online store.
Same features periodically materialized to offline store (Iceberg) for training.
```

Feast (open-source feature store) supports this pattern. Flink as the computation engine; Feast as the catalog and serving abstraction.

### 14.4 Sliding window aggregates

For "transactions in last 24h":

```java
.window(SlidingEventTimeWindows.of(Time.hours(24), Time.minutes(5)))
.aggregate(new CountAggregator())
```

24-hour window, slide 5 min. Per user. State: 24h / 5min = 288 active windows per user.

For 10M users × 288 windows: ~3B state entries → big RocksDB.

Alternative: maintain a sliding count incrementally with timestamp tombstones. Lower state, more code.

### 14.5 Trade-offs

| Approach | Trade |
|---|---|
| **Flink real-time features** | Sub-second; complex |
| **Spark batch features** | Cheap; hour-stale |
| **Hybrid (Flink + Spark)** | Best of both; pipeline ops |
| **Snowflake / BigQuery** | Cheap; minutes-stale |

### 14.6 What I'd actually do

Hybrid:
- Flink computes critical real-time features (e.g., last-hour aggregates).
- Spark batch computes historical features (e.g., last-90-day averages).
- Both materialize to feature store (online + offline).
- ML serving reads from online store.

---

## 15. Scenario 14 — Real-Time Anomaly Detection

### 15.1 The problem

Detect anomalies in metrics: sudden spike, drop, deviation from baseline. Real-time; alert within seconds.

### 15.2 The pattern

```java
DataStream<Metric> metrics = env.fromSource(...);

DataStream<Alert> alerts = metrics
    .keyBy(Metric::getName)
    .window(TumblingEventTimeWindows.of(Time.minutes(1)))
    .aggregate(new StatsAggregator())  // mean, stddev per minute
    .keyBy(Stats::getMetricName)
    .process(new AnomalyDetector());   // compare current vs historical

alerts.addSink(alertSink);
```

`AnomalyDetector` keeps historical stats per metric; compares incoming.

### 15.3 ML-based detection

For sophisticated anomaly detection: stream features to a model serving layer.

```java
// Async I/O to model serving
AsyncDataStream.unorderedWait(metrics, new ModelInferFunction(), 100, MS);
```

Model returns "anomaly score"; high scores trigger alerts.

### 15.4 Stateful baseline

```
Baseline = exponentially-weighted moving average of past values.
On each event: update EWMA.
Anomaly if |current - EWMA| > k × stddev.

State per metric: just two doubles. Cheap.
```

### 15.5 Trade-offs

| Approach | Trade |
|---|---|
| **Flink stateful detection** | Real-time; in-process |
| **Streaming + external model** | Best ML; async overhead |
| **Druid + alerting** | Eventual; query-based |
| **Custom rules engine** | Bespoke |

### 15.6 What I'd actually do

For real-time anomaly: Flink with EWMA + statistical thresholds for simple metrics. ML inference via async I/O for sophisticated.

---

## 16. Scenario 15 — Pattern Detection with CEP

### 16.1 The problem

Detect user journeys: "viewed product → added to cart → did NOT purchase within 30 min."

### 16.2 CEP "not-followed-by" pattern

```java
Pattern<Event, ?> abandoned = Pattern.<Event>begin("view")
    .where(SimpleCondition.of(e -> e.type == VIEW))
    .followedBy("addCart")
    .where(SimpleCondition.of(e -> e.type == ADD_CART))
    .notFollowedBy("purchase")
    .where(SimpleCondition.of(e -> e.type == PURCHASE))
    .within(Time.minutes(30));
```

Triggers when 30 min pass without a purchase after a cart add.

### 16.3 Use cases

- Cart abandonment.
- Session anomalies.
- Fraud sequences.
- Workflow detection.

### 16.4 State considerations

CEP NFA grows with active partial matches. For 100M users with 5% mid-pattern: 5M partial matches × 1 KB = 5 GB state.

For very large CEP state: aggressive within() / cleanup.

### 16.5 Trade-offs vs alternatives

| Approach | Trade |
|---|---|
| **Flink CEP** | Native; declarative |
| **ProcessFunction** | Custom; more code |
| **Stream-stream join** | For "same time bucket" matches |
| **Spark Streaming + complex SQL** | Workable; less expressive |

### 16.6 What I'd actually do

For declarative pattern detection: Flink CEP. For ad-hoc patterns: ProcessFunction.

---

## 17. Scenario 16 — Time-Series Real-Time Aggregations

### 17.1 The problem

Real-time per-host metrics: 1 min, 5 min, 1 hour aggregates. 100K hosts × 1000 metric types.

### 17.2 The aggregation

```java
DataStream<Metric> metrics = env.fromSource(...);

DataStream<Aggregate> oneMinAgg = metrics
    .keyBy(m -> Tuple.of(m.host, m.metricName))
    .window(TumblingEventTimeWindows.of(Time.minutes(1)))
    .aggregate(new MetricAggregator());  // sum, avg, p95

// Output to Druid / Pinot / Prometheus
```

For multiple resolutions: parallel windowed aggregations.

### 17.3 State size

```
100K hosts × 1000 metrics × 3 resolutions = 300M keys.
State per key: ~100 bytes.
Total: 30 GB state.

RocksDB; manageable.
```

### 17.4 Output cardinality vs Druid

Flink computes; Druid serves. Flink output cardinality matches per-host-per-metric; Druid ingests this stream.

```
Flink produces 1M aggregates/min.
Druid ingests; serves dashboards.
```

### 17.5 Trade-offs vs Druid-native

```
Pure Druid streaming ingestion: Druid does its own aggregation.
   Simpler; couples processing to storage.

Flink + Druid: Flink computes; Druid serves.
   More flexible (Flink can do CEP, joins, etc.); more pipeline.
```

For pure simple aggregation: Druid alone. For pre-processing / enrichment / joining: Flink → Druid.

### 17.6 What I'd actually do

Flink for any non-trivial real-time logic; Druid as the query layer. For simple count / sum: Druid alone.

---

## 18. Scenario 17 — Flink SQL for Real-Time Analytics

### 18.1 The problem

Stream + SQL. Analysts write SQL; gets translated to Flink job.

### 18.2 Flink SQL

```sql
CREATE TABLE events (
    user_id STRING, event_type STRING, event_time TIMESTAMP(3),
    WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
) WITH ('connector' = 'kafka', ...);

CREATE TABLE results (
    minute TIMESTAMP, count BIGINT,
    PRIMARY KEY (minute) NOT ENFORCED
) WITH ('connector' = 'jdbc', ...);

INSERT INTO results
SELECT
    TUMBLE_START(event_time, INTERVAL '1' MINUTE) as minute,
    COUNT(*) as count
FROM events
GROUP BY TUMBLE(event_time, INTERVAL '1' MINUTE);
```

### 18.3 Stream-table duality

```
Stream: append-only events.
Table: a snapshot of events; updates over time.

Flink SQL handles both:
  - Append-only sources.
  - Upsert sources (CDC).
  - Versioned tables (point-in-time).
```

### 18.4 Limitations

Compared to mature SQL engines:
- Optimizer less mature than Calcite-on-batch.
- Some functions not supported.
- Joins expensive in state.
- Complex nested subqueries: limited.

### 18.5 Use cases

- Stream ETL (Kafka in, Iceberg out).
- Real-time KPIs.
- Stream-table joins for enrichment.
- Materialized views.

### 18.6 Trade-offs

| Approach | Trade |
|---|---|
| **Flink SQL** | Declarative; managed pipeline |
| **Flink DataStream (Java/Scala)** | Full power; more code |
| **PyFlink** | Python; less complete |
| **Materialize / RisingWave** | SQL streaming + materialized views |

### 18.7 What I'd actually do

For SQL-friendly streaming: Flink SQL. For complex custom logic: DataStream API.

---

## 19. Scenario 18 — Auto-Scaling with Reactive Mode

### 19.1 The problem

Job's load varies 10× between peak and trough. Don't want to over-provision.

### 19.2 Reactive mode

```yaml
jobmanager.scheduler: adaptive
execution.checkpointing.interval: 60s
```

Reactive scheduler:
- Adjusts parallelism based on available task slots.
- Add TaskManagers → job rescales (with brief restart).
- Remove TaskManagers → job rescales.

### 19.3 K8s integration

```yaml
# Flink Operator with reactive
spec:
  flinkVersion: v1_18
  image: flink:1.18
  taskManager:
    replicas: 4
  jobManager:
    replicas: 1
  flinkConfiguration:
    jobmanager.scheduler: adaptive
    execution.checkpointing.interval: 60s
    
# K8s HPA scales TaskManagers based on backlog metric
```

### 19.4 The trade-off

Reactive rescaling = brief job restart (savepoint → restart). 30-second-ish pauses. Acceptable for many workloads.

For zero-downtime: don't reactive-scale; provision for peak.

### 19.5 Trade-offs

| Approach | Trade |
|---|---|
| **Reactive mode** | Cost-efficient; brief downtime |
| **Fixed parallelism** | Stable; over-provision |
| **Manual rescale** | Tightly controlled; manual |

### 19.6 What I'd actually do

For elastic batch-like streaming workloads: reactive mode.
For latency-critical: fixed parallelism, sized for peak.

---

## 20. Scenario 19 — Multi-Sink Pipelines (Side Outputs / Splits)

### 20.1 The problem

Single pipeline; multiple outputs:
- Main metrics → Druid.
- Errors → S3 + alert.
- Debug events → Elasticsearch.
- Audit → Iceberg.

### 20.2 The side-output pattern

```java
OutputTag<Error> errorTag = new OutputTag<>("error") {};
OutputTag<Audit> auditTag = new OutputTag<>("audit") {};

SingleOutputStreamOperator<Metric> mainStream = events.process(new ProcessFunction<>() {
    public void processElement(Event e, Context ctx, Collector<Metric> out) {
        if (e.type == ERROR) {
            ctx.output(errorTag, new Error(e));
        } else if (shouldAudit(e)) {
            ctx.output(auditTag, new Audit(e));
        }
        out.collect(toMetric(e));
    }
});

mainStream.addSink(druidSink);
mainStream.getSideOutput(errorTag).addSink(s3Sink);
mainStream.getSideOutput(auditTag).addSink(icebergSink);
```

One pass over the data; multiple outputs.

### 20.3 Why not multiple consumers from Kafka

Could just have separate jobs each reading the same Kafka topic. Pros:
- Independent jobs; simpler isolation.
- Each tuned for its sink.

Cons:
- Multiple reads of same Kafka data (cost).
- Multiple state copies if each computes overlapping things.

For small pipelines: separate. For unified processing: side outputs.

### 20.4 Trade-offs

| Approach | Trade |
|---|---|
| **Single Flink job + side outputs** | One pass; coupled |
| **Multiple Flink jobs** | Independent; multi-read |
| **Routing via Kafka topics** | Decoupled; latency |

### 20.5 What I'd actually do

For tightly-related sinks: single job + side outputs. For independent processing: separate jobs.

---

## 21. Performance and Scaling Deep Dive

### 21.1 Parallelism

```
operator.parallelism = N parallel tasks.
Total task slots = sum of TaskManager slots.
Job parallelism cannot exceed total slots.

Set per-operator:
  source.setParallelism(10);
  expensiveOp.setParallelism(100);
  sink.setParallelism(10);
```

Match parallelism to operator's work and shuffle constraints.

### 21.2 Backpressure

```
Detection:
  - Flink UI → Backpressure tab.
  - JVM thread sampling: fraction of time blocked on output buffer.
  
Causes:
  - Slow sink.
  - GC pauses.
  - Network saturation.
  - State access slowness (RocksDB).
  
Mitigations:
  - More parallelism on the bottlenecked operator.
  - Bigger network buffers.
  - Tune RocksDB (memory, compaction).
  - Faster sink (more partitions, async).
```

### 21.3 Network buffers

```yaml
taskmanager.network.memory.fraction: 0.2  # 20% of TaskManager memory
taskmanager.network.memory.max: 4gb
```

For high-throughput pipelines: bigger buffers.

### 21.4 RocksDB tuning

```yaml
state.backend.rocksdb.memory.managed: true   # Flink manages RocksDB memory
state.backend.rocksdb.thread.num: 4          # background compaction threads
state.backend.rocksdb.predefined-options: SPINNING_DISK_OPTIMIZED  # or FLASH
```

For SSD: FLASH options. For NVMe: FLASH_OPTIMIZED. Compaction threads tuned for I/O bandwidth.

### 21.5 Watermark debugging

```
Stuck watermark = jobs hang on windows.
Cause:
  - One source not advancing watermark (idle Kafka partition).
  - One operator blocked.
  
Mitigation: idleness detection.
WatermarkStrategy.<Event>forBoundedOutOfOrderness(Duration.ofSeconds(5))
    .withIdleness(Duration.ofMinutes(1));
```

A staff-level Flink skill: knowing how watermarks propagate, where they get stuck, and how to detect.

### 21.6 Async I/O

For external lookups, async I/O is critical:

```
Sync lookup: 10ms × 1000 events/sec = 10 sec/sec → backpressure.
Async lookup with capacity 100: 100× concurrent → 100K events/sec capable.
```

Always use async I/O for lookups in hot paths.

---

## 22. Operational Lifecycle

### 22.1 Day-1 setup

```
- K8s cluster + Flink Operator (or YARN, etc.).
- S3 (or HDFS) for checkpoints + savepoints.
- Metadata store (ZooKeeper for HA, or K8s native).
- Monitoring: JobManager + TaskManager metrics → Prometheus.
- Alerting: backpressure, checkpoint failures, restarts.
```

### 22.2 Upgrades

```
Job upgrade:
  1. Take savepoint of running job.
  2. Stop job.
  3. Deploy new code.
  4. Start from savepoint.
  
Flink runtime upgrade:
  - Same approach but for cluster.
  - Test on staging first.
```

### 22.3 Common operational issues

- **Checkpoint timeout**: state too big, sink slow, network slow.
- **OOM on TaskManager**: state in heap exceeded; switch to RocksDB.
- **Backpressure cascading**: slow operator → upstream OOM.
- **Watermark stuck**: idle source → no window emits.
- **Kafka broker hiccup**: source restarts; transactional sink may have aborted txns to clean up.

### 22.4 Disaster recovery

```
Savepoint: durable, S3-backed.
On region failure: restart job in another region from savepoint.
RTO: minutes (state download time).
RPO: depends on checkpoint interval; typically 1-5 min.
```

For zero-downtime: hot-standby cluster in another region; periodic savepoint sync.

### 22.5 Multi-region

Flink natively single-region. For multi-region:
- Active-passive: savepoint replicated; standby restarts on failure.
- Active-active: separate jobs in each region; consume Kafka cross-region (read replicas) or have per-region Kafka and reconcile downstream.

---

## 23. Anti-Patterns — Staff-Level Red Flags

### 23.1 Using Flink for batch-only jobs

Spark is better for batch. Flink's batch execution is a special case of streaming; less optimized.

### 23.2 Heap state for big state

Heap doesn't scale beyond 100s of GB; long GC; OOM. RocksDB for production.

### 23.3 No watermark idleness

Idle source → watermark stuck → job hangs. Always set idleness.

### 23.4 Synchronous external lookup

10ms-per-event sync lookup = 100 events/sec/task. Async I/O is mandatory.

### 23.5 Default checkpoint interval (off)

No checkpoints = no recovery. Always enable checkpointing.

### 23.6 No DLQ for sink failures

Sink fails → job restarts → infinite loop. Configure DLQ via side output.

### 23.7 Mixing batch and streaming code paths

Cleaner: separate jobs.

### 23.8 Ignoring backpressure

Backpressure indicator = something slow. Profile; don't ignore.

### 23.9 No savepoint discipline

Code change without savepoint → state lost. Always savepoint before stopping for an upgrade.

### 23.10 Treating Flink like Spark

Flink is per-event. Spark is micro-batch. Different mental model.

### 23.11 KeyedState across all keys (no keyBy)

Operator state instead of keyed; doesn't scale. Always keyBy.

### 23.12 Unbounded state without TTL or watermark

State grows forever. Configure TTL or rely on watermark cleanup.

```java
StateTtlConfig ttlConfig = StateTtlConfig.newBuilder(Time.days(7))
    .setUpdateType(UpdateType.OnCreateAndWrite)
    .build();
ValueStateDescriptor<MyState> desc = new ValueStateDescriptor<>("my-state", MyState.class);
desc.enableTimeToLive(ttlConfig);
```

### 23.13 Side input as a stream-stream join

For large reference data, broadcast or async I/O — not stream-stream join (state explodes).

### 23.14 Ignoring exactly-once vs at-least-once

Defaulting to exactly-once when at-least-once + idempotent suffices = unnecessary latency cost.

### 23.15 Custom serializers without snapshot

Custom Kryo serializers without TypeSerializerSnapshot → fragile across versions. Use POJO/Avro/Protobuf.

### 23.16 Single global parallelism

Different operators have different needs. Tune per-operator.

### 23.17 Tiny checkpoint intervals (< 5s)

Checkpoint overhead dominates. Min 10s; typical 30-60s.

### 23.18 No operator chaining

Chained ops avoid network/serialization. Default chains; only break for specific reasons.

### 23.19 Heap dump / state inspection in production

Querying running state is hard; queryable state is deprecated. Build a sink for what you need to observe.

### 23.20 Not testing at scale

Local mode hides distributed issues. Test on a real cluster.

---

## 24. Decision Framework

### 24.1 Step 0 — Is Flink the right tool?

```
Yes:
  - Sub-second latency.
  - Per-event semantics.
  - Event-time correctness needed.
  - Stateful processing.
  - Exactly-once across heterogeneous sinks.

Maybe:
  - Streaming with relaxed latency: Spark Structured Streaming.
  - Kafka-only ecosystem: Kafka Streams.
  - SQL-only: Materialize / RisingWave.

No:
  - Pure batch.
  - Sub-100K events/sec without state: Lambda + SQS works.
  - Team without Flink expertise.
```

### 24.2 Step 1 — Pick deployment

K8s + Operator / managed service / YARN.

### 24.3 Step 2 — State design

Keys, sizes, TTLs, backend (heap vs RocksDB).

### 24.4 Step 3 — Watermark strategy

Bounded out-of-orderness; idleness; allowed lateness.

### 24.5 Step 4 — Sink semantics

Exactly-once via 2PC, or at-least-once + idempotent.

### 24.6 Step 5 — Operational lifecycle

Savepoints. Upgrade plan. DR drill. Monitoring.

### 24.7 Step 6 — Validate against alternatives

Spark, Kafka Streams, Materialize. Defend choice.

---

## 25. Mental Models a Staff Engineer Carries

1. **Flink is true streaming, not micro-batch.** Per-event; sub-second.

2. **Watermarks are the heart.** Tightness controls latency vs correctness trade.

3. **State is the work.** Designing keys = designing the application.

4. **RocksDB for big state.** Heap for small.

5. **Async I/O for lookups.** Sync = throughput cap.

6. **Checkpoints are durable; savepoints are version-controlled.**

7. **Exactly-once requires transactional sinks.** Otherwise at-least-once + idempotency.

8. **Side outputs for any second sink / late events / errors.**

9. **TTL or watermark cleanup for state.** Otherwise it grows forever.

10. **Backpressure points to the slow operator.** Profile; tune.

11. **Allowed lateness extends window for late events.** Side output beyond.

12. **Reactive mode for elastic; fixed for latency-critical.**

13. **CEP for declarative patterns.** ProcessFunction for custom.

14. **Stream-stream join needs bounded interval.** Otherwise state grows.

15. **Flink CDC for real-time database replication.**

16. **Multi-tenancy via K8s namespaces.** Resource quotas enforce.

17. **Flink SQL for declarative; DataStream for power.**

18. **Local recovery saves time.** Don't always re-download from S3.

19. **Watermark idleness for sparse sources.** Else watermarks stuck.

20. **Don't query Flink state.** Build a sink to externalize.

21. **Spark vs Flink: per-shape.** Defend.

22. **Operational maturity is mandatory.** Flink is not click-and-go.

23. **Schema evolution via savepoint + new code.** POJO/Avro for simple changes.

24. **Backfill via savepoint + reset offsets.** Or parallel reprocess.

25. **Chained operators avoid serialization.** Don't break unless necessary.

26. **Fault tolerance is part of the design.** Not an afterthought.

27. **Sub-region DR via savepoints.** RTO minutes; RPO checkpoint interval.

28. **Boring is a feature.** A Flink job running for years with no surprises is the goal.

---

## 26. Closing Notes

Flink is the precision instrument of the streaming world. Its complexity is the price of its capabilities: ms-latency, true event time, exactly-once, TBs of state, complex pattern detection.

Used right, Flink is unmatched:
- Real-time fraud detection.
- Stream-stream joins at hyperscale.
- ML feature computation in milliseconds.
- CDC pipelines with sub-second freshness.
- Stateful workflows that survive failures.

Used wrong, it's a beast:
- Batch jobs awkwardly running on a streaming engine.
- Unbounded state growth.
- Watermark-stuck pipelines.
- Checkpoint storms.
- Cluster operations team consuming half the engineering hours.

The staff-level skill is choosing Flink only when the workload's shape fits, then designing the watermark, state, sinks, savepoints, and operational discipline correctly from day 1. Most "Flink is hard" complaints come from teams that didn't internalize the model.

A boring Flink job that runs for years, processes 10M events/sec, recovers from failures in seconds, and never wakes anyone up is one of the highest engineering accomplishments in the streaming domain. Everything in this document is in service of that goal.

> Companion docs:
> - `sparkScenariosAtScale.md` — when batch / micro-batch is right.
> - `eventPlatformsAtScale.md` — Kafka / MSK feeding Flink.
> - `druidScenariosAtScale.md` — analytics layer Flink feeds.
> - `dynamoDBScenariosAtScale.md` — async-I/O lookup target.
> - `awsS3ScenariosAtScale.md` — checkpoint / savepoint storage.