# Apache Spark — Realistic Scenarios at Staff-Engineer Depth

> A practical, opinionated reference for running Apache Spark at MAANG scale — petabyte-day ETL, streaming pipelines feeding ML, hundreds-of-table joins, gnarly skew, and clusters running for hours that need to *just work*. Written for engineers past "Spark runs distributed jobs" who now need to size executors for a 10 TB join, defend Spark vs Flink in a design review, debug a job stuck in shuffle for 6 hours, and make Spark cost 50% less without breaking the SLA.

> Companion to `awsS3ScenariosAtScale.md`, `dynamoDBScenariosAtScale.md`, `druidScenariosAtScale.md`, `eventPlatformsAtScale.md`, `logProcessingAndAggregation.md`. Spark is the workhorse compute layer; this doc is about driving it well.

---

## 0. The Staff-Level Frame

A Spark job that works on 10 GB of test data and fails on 10 TB of production data is not a bug — it's a sign the engineer didn't internalize Spark's execution model. The shift from senior to staff is asking, before writing a line of code:

1. **What's the data shape?** Skew, cardinality, file count, format.
2. **What's the join shape?** Broadcast-eligible? Sort-merge? Shuffle hash? Skewed?
3. **What's the resource shape?** Executor memory, cores, parallelism, shuffle partitions.
4. **What's the storage and I/O shape?** S3 vs HDFS, Parquet vs CSV, partition pruning, predicate pushdown.
5. **What's the failure shape?** Fault tolerance, retry, output committer, idempotency.
6. **What's the cost shape?** Spot vs on-demand, autoscaling, shuffle data on disk vs S3.
7. **What's the operational shape?** Cluster manager, deployment, alerting, profile.

The staff-level mistake to avoid: treating Spark like Pandas with a bigger budget. Pandas runs on one machine; Spark runs on hundreds. The mental model — RDDs/DataFrames as immutable, lazily-evaluated, stage-partitioned plans — is the entire game.

---

## 1. Mental Model — What Spark Is and Isn't

### 1.1 What Spark is

A **distributed in-memory data processing engine** with:
- **Lazy DAG execution**: build a logical plan; execute on action.
- **Catalyst optimizer**: rewrites logical plans (predicate pushdown, projection pruning, join reordering, AQE).
- **Tungsten execution engine**: off-heap memory, vectorized columnar processing, whole-stage code generation.
- **DataFrames + Datasets + RDDs**: three APIs at different abstraction levels.
- **Structured Streaming**: event-time, watermarked, exactly-once micro-batch (or continuous) streaming.
- **MLlib + GraphX + Spark SQL**: domain-specific libraries.
- **Pluggable storage** (Parquet, ORC, Avro, JSON, JDBC, Iceberg, Delta, Hudi).
- **Pluggable cluster managers** (Standalone, YARN, Kubernetes, Mesos).

### 1.2 What Spark isn't

- **Not a database.** It's a compute engine. Catalogs (Hive, Glue, Iceberg) provide table metadata.
- **Not low-latency.** Single queries: seconds to minutes. Streaming micro-batches: 100s of ms minimum.
- **Not a streaming-first engine.** Flink is. Spark Structured Streaming is "streaming bolted onto batch"; very capable but architecturally micro-batch-first.
- **Not free.** Cluster, shuffle storage, cross-AZ, data egress all bill.
- **Not point-and-shoot.** Default settings are wrong for big jobs. Sizing executors, partitions, shuffles is the work.
- **Not for transactional workloads.** Spark + Delta does ACID over Parquet, but for OLTP-style use, you want a real DB.

### 1.3 The execution model

```
Driver
  ├─► creates SparkSession
  ├─► builds DAG (logical plan)
  ├─► Catalyst optimizes (logical → physical)
  ├─► AQE adapts plan during execution
  ├─► sends tasks to executors
  └─► collects results

Executors (workers)
  ├─► run tasks (reading data, transformation, writing output)
  ├─► hold cached RDDs / DataFrames in memory
  ├─► shuffle data between stages

Stages
  ├─► sequence of transformations without shuffle
  ├─► boundaries are shuffles (groupBy, join, repartition)
  └─► one stage's output is shuffled + becomes next stage's input

Tasks
  ├─► one task per partition
  └─► smallest unit of work
```

The shuffle is the most-expensive operation. Most performance work is about minimizing or optimizing shuffles.

### 1.4 The deployment landscape

| Platform | Trade |
|---|---|
| **EMR** | AWS-managed; cheap for bursty; cluster boot time 10-15 min |
| **Databricks** | Best DX; notebooks + jobs + Photon engine; expensive |
| **EKS Spark Operator** | K8s-native; ops cost; auto-scaling |
| **Glue** | Serverless; small to mid jobs; cold start; limited config |
| **Self-managed (YARN/Mesos)** | Full control; ops heavy |
| **Synapse / Dataproc / HDInsight** | Cloud-vendor variants |

For most: EMR + Spark on YARN, or Databricks on AWS, or EKS for ops-mature teams.

### 1.5 Quick reference

| Aspect | Value / shape |
|---|---|
| **Min job latency** | 10s (driver+task overhead) |
| **Max useful job size** | PB-day on a large cluster |
| **Default shuffle partitions** | 200 (almost always wrong for production) |
| **Executor memory** | 4-32 GB typical; 8-16 GB sweet spot |
| **Executor cores** | 4-5 sweet spot; 1-8 range |
| **Driver memory** | 4-32 GB; bumps for big driver-side ops |
| **Storage formats** | Parquet > ORC > Avro > JSON > CSV |
| **Table formats** | Iceberg / Delta / Hudi (modern); Hive (legacy) |
| **Join types** | Broadcast (small), sort-merge (large), shuffle hash (rare) |
| **Cost shape** | EC2 (executor) dominates; S3 list+get; cross-AZ shuffle |

---

## 2. Scenario 1 — Petabyte-Day ETL Pipeline

### 2.1 The problem

Daily ETL: ingest yesterday's clickstream (~5 TB compressed Parquet), enrich with user dimension (10 GB), session attribution, write back partitioned to S3 for warehouse + ML feature store consumers. SLA: completes in 4 hours.

### 2.2 The naive (broken) version

```python
df = spark.read.parquet("s3://raw/clickstream/")  # all data
users = spark.read.parquet("s3://dim/users/")
joined = df.join(users, "user_id")
sessions = sessionize(joined)
sessions.write.parquet("s3://warehouse/sessions/")
```

What goes wrong:
- Reads everything (no date filter).
- Default 200 shuffle partitions for a 5 TB shuffle → 25 GB per partition → OOM.
- No partition pruning on output; one giant write.
- No bucketing → rejoins re-shuffle.
- Default executor memory 4 GB → OOM on first day's data.

### 2.3 The production version

```python
# 1. Filter at source (partition pruning)
df = (spark.read
      .parquet("s3://raw/clickstream/")
      .filter(F.col("date") == "2026-04-28"))    # prunes Hive-partitioned

# 2. Project only needed columns (column pruning)
df = df.select("user_id", "event_time", "page", "device", "geo")

# 3. Tune shuffle partitions for the data volume
spark.conf.set("spark.sql.shuffle.partitions", "2000")  # 2.5 GB / partition target

# 4. Broadcast join the small dimension
users = spark.read.parquet("s3://dim/users/")
joined = df.join(F.broadcast(users), "user_id")

# 5. Sessionize (window function over user_id)
window = Window.partitionBy("user_id").orderBy("event_time")
sessions = joined.withColumn("session_gap",
    F.col("event_time") - F.lag("event_time").over(window))

# 6. Write partitioned by date and hour for downstream pruning
(sessions
 .repartition("date", "hour")    # one file per partition target
 .write
 .partitionBy("date", "hour")
 .mode("overwrite")
 .parquet("s3://warehouse/sessions/"))
```

### 2.4 Why each change matters

```
Filter at source: 5 TB → 1 TB (pre-Catalyst pushdown).
Project columns: 1 TB → 100 GB (10× column pruning).
Broadcast small table: avoid shuffle (saves the 100 GB shuffle for the join).
Tune shuffle partitions: 200 → 2000 (avoid OOM on 25 GB partitions).
Repartition before write: avoid 100K tiny output files OR one huge file.
Partitioned output: downstream queries prune to one date+hour.
```

### 2.5 The shuffle math

```
Shuffle = data movement between stages.
Each shuffle:
  - Reads input partition.
  - Sorts/hashes.
  - Writes shuffle output to local disk (or S3 with shuffle service).
  - Next stage reads.

For 100 GB shuffle:
  partitions = 200 → 500 MB / partition → fits in memory, fast.
  partitions = 200 with 10 TB input → 50 GB / partition → spills to disk → 10× slower or OOM.
  
Rule of thumb: target 100-500 MB per shuffle partition.
spark.sql.shuffle.partitions = total shuffle bytes / 200 MB.
```

### 2.6 Adaptive Query Execution (AQE) — Spark 3.0+

```python
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
```

AQE adapts the plan during execution:
- **Coalesce small partitions**: 2000 partitions but most are tiny → coalesce to 100s.
- **Switch join strategies**: planned sort-merge but turns out one side is tiny → switch to broadcast at runtime.
- **Skew-join optimization**: detect skewed keys; split them across more tasks.

Always enable AQE for production. It's the single biggest performance lever in modern Spark.

### 2.7 Cluster sizing

```
Goal: 5 TB compressed Parquet (50 TB uncompressed) processed in 4 hours.

Throughput target:
  50 TB / 4 hr = 12.5 TB/hr = 3.5 GB/sec aggregate.

Per-executor throughput (ballpark, 4 cores, 16 GB):
  ~50 MB/sec sustained processing.
  3.5 GB/sec / 50 MB = 70 executors.

Plus headroom + redundancy: ~100 executors.

Cluster (r5.4xlarge: 16 cores, 128 GB):
  4 executors per node (4 cores, 24 GB executor memory each, leaving room for OS/YARN/driver).
  100 executors / 4 = 25 nodes.

Cost (us-east-1):
  25 × r5.4xlarge ($1.008/hr) × 4 hr = $100.80/job/day.
  Spot at 70% discount: ~$30.
```

### 2.8 Trade-offs and alternatives

| Approach | Trade |
|---|---|
| **Spark on EMR** | Bursty cluster; fast ramp; AWS-managed |
| **Spark on Databricks** | Best DX; expensive; vendor-locked |
| **Spark on K8s** | Best multi-tenancy; ops heavy |
| **BigQuery / Snowflake** | Less code; per-query cost can dominate |
| **Trino / Athena** | Better for ad-hoc; less for heavy ETL |
| **Hadoop MapReduce** | Don't (legacy) |

### 2.9 What I'd actually do

For petabyte-day ETL:
- **Spark on EMR with Spot fleet** for batch.
- **Iceberg or Delta Lake** as table format (schema evolution, time travel, atomic writes).
- **AQE enabled, shuffle partitions tuned** to data volume.
- **Partition + Z-order / cluster output** for downstream pruning.
- **Auto-recovery** on Spot eviction (EMR managed).

---

## 3. Scenario 2 — Structured Streaming Pipeline

### 3.1 The problem

Real-time pipeline: Kafka → Spark Structured Streaming → enriched output to Iceberg + downstream Kafka. 1M events/sec. End-to-end latency target: < 5 sec.

### 3.2 The pipeline

```python
df = (spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "broker:9092")
      .option("subscribe", "events")
      .option("startingOffsets", "latest")
      .option("maxOffsetsPerTrigger", 1000000)   # rate limit
      .load())

events = df.selectExpr("CAST(value AS STRING) as json").select(
    F.from_json("json", schema).alias("e")).select("e.*")

# Enrich with broadcast lookup (refreshed periodically)
users = spark.read.parquet("s3://dim/users/")
enriched = events.join(F.broadcast(users), "user_id")

# Write to Iceberg (using Spark Streaming + Iceberg sink)
query = (enriched.writeStream
        .format("iceberg")
        .option("path", "s3://warehouse/events_iceberg")
        .option("checkpointLocation", "s3://checkpoints/events_pipe")
        .trigger(processingTime="30 seconds")
        .start())
```

### 3.3 The mental model: micro-batch

```
Spark Structured Streaming = micro-batch.
Every trigger interval: read new offsets → process as a batch → write atomically → commit offsets.
Default trigger: as fast as possible.
With trigger("30 seconds"): batch every 30 seconds.

Continuous mode (experimental): true per-record processing; very limited features.
```

For most production: micro-batch. Latency floor ~100s of ms. For ms-latency: Flink.

### 3.4 Watermarking

```python
events_with_watermark = events.withWatermark("event_time", "5 minutes")
```

- Watermark = "all events with time < watermark have arrived."
- Used for: stateful aggregations, joins, windows.
- 5-minute watermark = state is kept for 5 min after window close; late events dropped.

```python
windowed = (events_with_watermark
    .groupBy(F.window("event_time", "1 minute"), "geo")
    .count())
```

### 3.5 Stateful streaming

State stored in:
- **HDFS-backed state store** (default).
- **RocksDB state store** (better for large state).

For 1M events/sec with 1-min window and 100K geos: 100K state entries × 1KB = 100 MB. Fits in memory.

For sessionization with 10M concurrent sessions × 10 KB each = 100 GB state — needs RocksDB and careful tuning.

### 3.6 Exactly-once semantics

Spark Streaming provides exactly-once **end-to-end** if:
- Source is replayable (Kafka, Kinesis with checkpoints).
- Sink is idempotent or transactional (Iceberg, Delta, JDBC with epoch IDs, transactions).

Checkpoint + offsets in source = at-least-once consumption. Atomic commit at sink = exactly-once-effective.

### 3.7 Failure recovery

```
On driver/executor failure:
  - Restart from last checkpoint.
  - Re-read from last committed offsets.
  - Re-process the batch (sink dedups via atomic commit).
  - Continues.

Kept-state for window/session: restored from checkpoint.
```

Checkpoint location is critical; on S3 use S3 Express or local HDFS for low-latency.

### 3.8 Trade-offs vs alternatives

| Approach | Trade |
|---|---|
| **Spark Structured Streaming** | Reuses batch code; micro-batch; sub-second hard |
| **Flink** | True streaming; ms latency; richer state APIs |
| **Kafka Streams** | Kafka-native; JVM only; no batch reuse |
| **Beam** | Multi-runner; abstraction layer |
| **Materialize / RisingWave** | Real-time SQL; newer |

For most: Spark Structured Streaming if you also have batch in Spark; Flink if streaming is the *primary* workload.

### 3.9 What I'd actually do

For Kafka → enrichment → sink:
- Spark Structured Streaming with 30s trigger.
- AQE + RocksDB state store.
- Watermark = 2× expected late-arrival window.
- Sink to Iceberg (atomic, schema-evolvable, time-travel).
- Checkpoint to S3 Express One Zone (or HDFS in EMR).

For sub-second latency: Flink (or accept the tradeoff).

---

## 4. Scenario 3 — Joining Wide Tables (Broadcast vs Shuffle)

### 4.1 The problem

Join a 100 TB clickstream fact table with:
- A 5 GB user dimension.
- A 50 GB session dimension (medium-sized).
- A 100 TB product dimension (huge).

### 4.2 Broadcast join (small dimension)

```python
clicks.join(F.broadcast(users), "user_id")
```

- Spark sends `users` to every executor.
- Each executor does in-memory hash lookup.
- No shuffle on `clicks`.
- Throughput limited by `clicks` scan speed.

```
Threshold: spark.sql.autoBroadcastJoinThreshold (default 10 MB compressed).
For 5 GB user dim: increase to 1 GB:
  spark.conf.set("spark.sql.autoBroadcastJoinThreshold", 1024*1024*1024)

Practical limit: ~5 GB per broadcast (driver/executor memory pressure).
```

### 4.3 Sort-merge join (medium-large dimensions)

```python
clicks.join(sessions, "session_id")  # default sort-merge
```

- Spark shuffles both tables on `session_id`.
- Sorts each side; merges.
- O(N log N) per side.

```
Cost: shuffle 100 TB clicks + 50 GB sessions.
   Almost the entire cost is shuffling clicks.
```

### 4.4 Shuffle hash join (alternative)

```
Spark may pick shuffle-hash join when one side fits in memory of each shuffle partition.
Hash table built; smaller side hashed; bigger side probed.
Less memory-friendly than sort-merge for skewed data.
```

Rarely manually selected; AQE picks.

### 4.5 Pre-bucketed join (the trick for repeated joins)

```python
# Write tables with bucketing
(clicks.write
 .bucketBy(1024, "session_id")
 .sortBy("session_id")
 .saveAsTable("clicks_bucketed"))
(sessions.write
 .bucketBy(1024, "session_id")
 .saveAsTable("sessions_bucketed"))

# Subsequent joins skip the shuffle
clicks_bucketed.join(sessions_bucketed, "session_id")  # bucketed → no shuffle
```

For tables joined repeatedly on the same key: bucketing eliminates shuffle. Costs once at write; saves on every join.

Caveat: bucketing works with Hive metastore + Spark; Iceberg's hidden partitioning is a different model.

### 4.6 Star join (multiple dimensions)

```python
fact.join(F.broadcast(users), "user_id") \
    .join(F.broadcast(geos), "geo_id") \
    .join(F.broadcast(products), "product_id")
```

For data warehouse star schema: broadcast all dimensions; one scan of the fact table.

If fact is partition-pruned + dimensions are small: this is the optimal pattern.

### 4.7 The 100 TB × 100 TB join

```
Truly large × large join:
  Sort-merge with proper bucketing/partitioning.
  Pre-shuffle sort.
  Many shuffle partitions (10K+).
  
Or: rethink the design.
  - Can you pre-aggregate one side?
  - Can you partition both by date and join only same-date?
  - Can you use a stream-table join in Flink instead?
```

A pure 100 TB × 100 TB join in Spark is doable but expensive ($1000s+ in compute). Consider whether the join is necessary at that volume.

### 4.8 Trade-offs

| Join Type | Best for | Trade |
|---|---|---|
| **Broadcast** | < ~5 GB inner | Best throughput; OOM if too big |
| **Sort-merge** | Large × large | Standard; 2× shuffle |
| **Shuffle-hash** | One side fits in partition memory | AQE picks; rare manually |
| **Bucketed (pre-shuffle)** | Repeated same-key joins | Pre-pay shuffle once |
| **Skew join** | Skewed keys | AQE handles; or manual salting |

### 4.9 What I'd actually do

- Always check if any side qualifies for broadcast (raise threshold to 1 GB; verify with EXPLAIN).
- For star schema: broadcast dimensions.
- For repeated same-key joins: bucket.
- Enable AQE skew-join.
- For 100 TB × 100 TB: question requirements; partition by date if possible.

---

## 5. Scenario 4 — Handling Data Skew

### 5.1 The problem

Join on user_id. 0.1% of users (celebrities, bots) account for 50% of events. A handful of partitions take 10× longer → straggler tasks → job times out.

### 5.2 Detection

```
Spark UI → Stages → look at task duration distribution.
If max task time >> median: skew.
"Shuffle Read Size" per task: skewed values stand out.
```

Modern AQE (3.0+) detects and reports this automatically.

### 5.3 AQE skew-join optimization

```python
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionFactor", "5")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes", "256MB")
```

AQE detects skewed partitions at runtime; splits them into multiple tasks; re-replicates the other side.

### 5.4 Manual salting

For pre-AQE or extreme skew, salt the join key:

```python
N = 100  # salt cardinality
fact_salted = fact.withColumn("salt", (F.rand() * N).cast("int"))
fact_salted = fact_salted.withColumn("user_salted", F.concat("user_id", F.lit("#"), "salt"))

dim_replicated = dim.withColumn("salt", F.explode(F.array(*[F.lit(i) for i in range(N)])))
dim_replicated = dim_replicated.withColumn("user_salted", F.concat("user_id", F.lit("#"), "salt"))

joined = fact_salted.join(dim_replicated, "user_salted")
```

- Splits each user's events across N synthetic keys.
- Replicates the dimension N times (one row per salt).
- Result: joins distribute across N×original partitions.

Cost: N× storage for the dimension during the join. Worth it for severe skew.

### 5.5 Pre-aggregation

If the join is followed by aggregation: aggregate before join.

```python
# Bad: join then aggregate (skew amplifies)
fact.join(dim, "user_id").groupBy("country").agg(...)

# Good: aggregate per-user first, then join
per_user = fact.groupBy("user_id").agg(F.count("*").alias("events"))
per_user.join(dim, "user_id").groupBy("country").agg(F.sum("events"))
```

Reduces fact size before the skewed join.

### 5.6 Separate hot keys

For known hot keys (top 100 users):

```python
hot = fact.filter(F.col("user_id").isin(hot_users))
cold = fact.filter(~F.col("user_id").isin(hot_users))

hot_joined = hot.join(F.broadcast(dim_filtered_to_hot), "user_id")
cold_joined = cold.join(dim, "user_id")

result = hot_joined.union(cold_joined)
```

Hot keys join via broadcast; cold via shuffle. Best for very-known-skew distributions.

### 5.7 Trade-offs

| Approach | Trade |
|---|---|
| **AQE skew-join** | Automatic; modern Spark |
| **Manual salting** | Universal; more code |
| **Hot/cold split** | Targeted; needs known hot keys |
| **Pre-aggregate** | Best when followed by aggregate |
| **Repartitioning** | Doesn't fix; just moves skew |

### 5.8 What I'd actually do

Always: enable AQE. Profile after every job for skew. For severe skew not handled by AQE: hot/cold split + broadcast of hot dim subset.

---

## 6. Scenario 5 — Sessionization / User Journey

### 6.1 The problem

User events. Define a session as: same user, gap < 30 min between events. Compute session duration, page count per session, conversion within session.

### 6.2 Window-function approach

```python
window = Window.partitionBy("user_id").orderBy("event_time")

events = events.withColumn("prev_time", F.lag("event_time").over(window))
events = events.withColumn("gap_minutes", 
    (F.col("event_time") - F.col("prev_time")) / 60)
events = events.withColumn("new_session_flag", 
    F.when(F.col("gap_minutes") > 30, 1).otherwise(0))
events = events.withColumn("session_id",
    F.sum("new_session_flag").over(window))
```

`session_id` is now a per-user running session counter.

### 6.3 The shuffle cost

```
partitionBy("user_id") shuffles all events to the same partition per user.
For 1B events / 100M users: 10 events/user average.
Shuffle: 1B events × 200 bytes = 200 GB.

For 5K shuffle partitions: 40 MB / partition.
Per-user state: ~negligible (10 events).
```

Manageable. If users are skewed (some have millions of events): apply skew handling from §5.

### 6.4 Streaming sessionization

For real-time sessions:

```python
# Spark Structured Streaming with watermark and custom session window
sessions = (events.withWatermark("event_time", "10 minutes")
            .groupBy("user_id", F.session_window("event_time", "30 minutes"))
            .agg(F.count("*").alias("event_count"),
                 F.first("session_id").alias("session_start_id")))
```

Spark 3.2+ has native `session_window`. State stored in RocksDB.

### 6.5 Trade-offs

| Approach | Trade |
|---|---|
| **Spark batch (window)** | Best for daily sessionization |
| **Spark streaming session_window** | Real-time; state size matters |
| **Flink event-time sessions** | Best streaming; richer features |
| **Custom in app** | Don't (reinventing) |

### 6.6 What I'd actually do

Daily sessionization: batch Spark with window functions. Real-time: Flink for true session windows; Spark Structured Streaming if already in Spark stack.

---

## 7. Scenario 6 — Slowly Changing Dimensions (SCD)

### 7.1 The problem

User table. Each user has attributes (plan, country, segment) that change over time. Need to know "what was their plan on 2026-01-15?"

### 7.2 SCD Type 2 with Iceberg / Delta

```
Each row has:
  user_id
  attributes
  valid_from
  valid_to (nullable; null = current)

For a user updated:
  Old row: valid_to = update timestamp.
  New row: valid_from = update timestamp, valid_to = null.
```

```python
# MERGE INTO with Iceberg / Delta
spark.sql("""
MERGE INTO users_history t
USING staged_updates s
  ON t.user_id = s.user_id AND t.valid_to IS NULL
WHEN MATCHED AND s.attributes != t.attributes THEN
  UPDATE SET t.valid_to = current_timestamp()
WHEN NOT MATCHED THEN
  INSERT (user_id, attributes, valid_from, valid_to)
  VALUES (s.user_id, s.attributes, current_timestamp(), null)
""")
```

This is non-trivial in pure Parquet/Hive; Iceberg/Delta MERGE makes it tractable.

### 7.3 Querying SCD2

```sql
SELECT * FROM events e
JOIN users_history u
  ON e.user_id = u.user_id
  AND e.event_time BETWEEN u.valid_from AND COALESCE(u.valid_to, current_timestamp())
```

Range join on time; AQE / proper bucketing helps.

### 7.4 Trade-offs

| SCD Type | Description | Trade |
|---|---|---|
| **Type 1** | Overwrite | Simplest; loses history |
| **Type 2** | Versioned rows | History; complex queries |
| **Type 3** | Limited history (one prior value) | Compromise |
| **Type 4 (Iceberg time travel)** | Snapshot table at time T | Native Iceberg/Delta |

Iceberg/Delta time travel is essentially free Type-4 SCD via snapshots. For "what did the table look like on date X": just `AS OF` query.

### 7.5 What I'd actually do

Type 1 for low-importance attributes; Type 2 (with Iceberg MERGE) for tracked attributes; Iceberg time travel for arbitrary point-in-time queries.

---

## 8. Scenario 7 — Deduplication at Scale

### 8.1 The problem

100 TB of event data with potential duplicates (replays, retries). Need to deduplicate by `event_id`.

### 8.2 The naive approach

```python
df.dropDuplicates(["event_id"])
```

Issues at scale:
- Shuffles all data on `event_id`.
- Keeps only one row per key in memory.
- For 100 TB: 100 TB shuffle. Slow but works.

### 8.3 The optimization for time-windowed dedup

If duplicates always occur within a small time window (e.g., dedup within 24 hours):

```python
windowed = (df.withColumn("date", F.to_date("event_time"))
            .repartition("date"))
deduped = windowed.dropDuplicates(["event_id"])
```

Process per-date; smaller shuffles per partition.

### 8.4 Streaming deduplication

```python
deduped = events.withWatermark("event_time", "1 hour") \
                .dropDuplicates(["event_id", "event_time"])
```

State store keeps `event_id`s for the watermark duration. After watermark passes, IDs forgotten.

### 8.5 Probabilistic dedup with Bloom filter

For approximate dedup (acceptable false-negative rate):

```python
bloom = build_bloom_filter(historical_event_ids)
deduped = events.filter(~bloom.contains("event_id"))
```

Cheap; small false-positive rate (drops some non-duplicates).

### 8.6 Iceberg / Delta MERGE for upsert dedup

```sql
MERGE INTO target t
USING incoming i
  ON t.event_id = i.event_id
WHEN NOT MATCHED THEN INSERT *
```

Built-in; atomic; per-batch dedup.

### 8.7 Trade-offs

| Approach | Trade |
|---|---|
| **dropDuplicates** | Universal; full shuffle |
| **Time-windowed** | Faster; assumes locality |
| **Streaming + watermark** | Real-time; state-bound |
| **Bloom filter** | Cheap; approximate |
| **Iceberg MERGE** | Atomic upsert |

### 8.8 What I'd actually do

For batch: time-windowed dedup if events have time locality. For streaming: watermarked dropDuplicates. For incremental ingestion: Iceberg MERGE.

---

## 9. Scenario 8 — Backfill and Reprocessing

### 9.1 The problem

Bug in last 30 days of pipeline. Need to reprocess. Source data unchanged; output needs full rebuild.

### 9.2 The backfill plan

```
1. Stop streaming pipeline (or fork to non-prod output).
2. Run batch Spark job for affected date range.
3. Write to staging path.
4. Validate (counts, samples, hash compare with old where possible).
5. Atomic swap (Iceberg snapshot replace, or rename in S3).
6. Resume streaming.
```

### 9.3 Idempotent partitioned writes

```python
# Iceberg / Delta: REPLACE WHERE
spark.sql("""
REPLACE INTO events
SELECT * FROM new_data
WHERE date BETWEEN '2026-04-01' AND '2026-04-28'
""")
```

Atomically replaces only the affected date range. Other dates unaffected.

### 9.4 Parallelizing backfills

```python
# Spawn one Spark job per date in parallel (across cluster slots)
for d in dates_to_backfill:
    submit_spark_job(date=d)  # via Airflow / orchestrator
```

For 30 days × 1 hour each = 30 hours sequential. Parallel: 4 hours if 8 jobs concurrently.

### 9.5 Resource isolation

Backfill jobs are heavy; don't starve regular jobs. On YARN/K8s: separate queue / namespace with capped resources.

### 9.6 Validation

```python
# Row counts per partition
old.groupBy("date").count().orderBy("date")
new.groupBy("date").count().orderBy("date")

# Aggregate checksums
old.groupBy("date").agg(F.sum(F.hash("*")).alias("h")).orderBy("date")
new.groupBy("date").agg(F.sum(F.hash("*")).alias("h")).orderBy("date")
```

Mismatches at partition level isolate the bug.

### 9.7 What I'd actually do

For backfills:
- Iceberg/Delta REPLACE WHERE for atomic partition replacement.
- Parallelize per-date jobs via Airflow.
- Validation + manual approval before final swap.
- Time-bounded to avoid runaway cost.

---

## 10. Scenario 9 — Stateful Streaming (Sessionization, Counting Windows)

### 10.1 The problem

Real-time per-user 1-hour rolling counts: how many events per user in last hour. Need to update as events arrive; state across millions of users.

### 10.2 The implementation

```python
events_with_watermark = events.withWatermark("event_time", "10 minutes")

windowed = (events_with_watermark
    .groupBy("user_id", F.window("event_time", "1 hour", "10 minutes"))
    .agg(F.count("*").alias("event_count")))
```

- Sliding window: 1-hour size, 10-minute slide.
- Each event is in 6 windows (1 hour / 10 min).
- State: per-user × per-window; 6× event count.

### 10.3 State size

```
1M users × 6 windows × 1 KB / window = 6 GB state.
On HDFS state store: fits, but slow.
On RocksDB state store: fast and large-state friendly.
```

```python
spark.conf.set("spark.sql.streaming.stateStore.providerClass",
    "org.apache.spark.sql.execution.streaming.state.RocksDBStateStoreProvider")
```

### 10.4 State expiry

Watermark is the GC mechanism. State entries with end time before watermark are deleted. Without watermark: state grows unboundedly.

### 10.5 Stateful with arbitrary logic (mapGroupsWithState)

For custom session logic / fraud detection:

```python
def updateUser(user_id, events, state):
    if state.exists:
        s = state.get
    else:
        s = UserState(events_count=0, ...)
    for e in events:
        s.events_count += 1
        # custom logic
    state.update(s)
    return s

result = events.groupByKey(_.user_id).mapGroupsWithState(updateUser)
```

Full per-key state; user-defined logic. Requires Scala / Java for full power; PySpark has limited support.

### 10.6 Trade-offs

| Approach | Trade |
|---|---|
| **Window aggregation** | Standard; declarative |
| **mapGroupsWithState** | Full custom logic; Scala only really |
| **flatMapGroupsWithState** | Like mapGroups but with timeouts |
| **Flink** | Better state APIs; richer event-time handling |

### 10.7 What I'd actually do

For window aggregations: declarative with watermark. For complex stateful logic: consider Flink. State store: RocksDB.

---

## 11. Scenario 10 — CDC Processing (Database → Spark → Sink)

### 11.1 The problem

Postgres source → Debezium → Kafka → Spark → Iceberg warehouse. Apply inserts, updates, deletes.

### 11.2 The pipeline

```python
cdc = (spark.readStream.format("kafka")
       .option("subscribe", "postgres.public.users")
       .load())

# Parse Debezium envelope
parsed = cdc.select(F.from_json("value", debezium_schema).alias("e")) \
            .selectExpr("e.op as op", "e.before.* as before", "e.after.* as after")

# Apply CDC to Iceberg
def applyCDC(batch_df, batch_id):
    batch_df.createOrReplaceTempView("incoming")
    spark.sql("""
        MERGE INTO users t
        USING (SELECT * FROM incoming) s
          ON t.user_id = s.user_id
        WHEN MATCHED AND s.op = 'd' THEN DELETE
        WHEN MATCHED AND s.op IN ('u', 'c') THEN UPDATE SET *
        WHEN NOT MATCHED AND s.op IN ('c', 'u') THEN INSERT *
    """)

query = parsed.writeStream.foreachBatch(applyCDC).start()
```

### 11.3 Considerations

```
Idempotency: MERGE is idempotent (re-applying same CDC events is safe).
Out-of-order: handle via CDC sequence numbers (LSN, txn id).
Schema evolution: Iceberg handles new columns; deletions of columns require care.
Volume: at high CDC rate, MERGE on huge target table is expensive.
   Mitigation: filter target by partition keys present in batch.
```

### 11.4 The 1-hour vs micro-batch choice

```
1-hour batches: cheap; less frequent MERGE; up-to-1-hour staleness.
30-second micro-batches: real-time; many small MERGEs; fragmentation.

For typical analytics: 5-15 minute batches strike a balance.
```

### 11.5 Trade-offs

| Approach | Trade |
|---|---|
| **Spark + foreachBatch + Iceberg MERGE** | Standard; flexible |
| **Flink CDC connector** | Lower latency; ms |
| **Hudi DeltaStreamer** | Hudi-specific; managed |
| **DMS to S3 (no transformation)** | Cheapest; raw landing |

### 11.6 What I'd actually do

For CDC at MAANG scale:
- Debezium → Kafka.
- Spark Structured Streaming with foreachBatch + Iceberg MERGE.
- 5-minute batches.
- Iceberg compaction nightly.
- Source-of-truth in Iceberg; downstream BI consumes.

---

## 12. Scenario 11 — Schema Evolution with Delta / Iceberg

### 12.1 The problem

Add a column to a table. Producers add the column; consumers read with a new schema. How to evolve safely?

### 12.2 Delta Lake / Iceberg schema evolution

```python
# Iceberg / Delta supports:
spark.sql("ALTER TABLE events ADD COLUMN device_model STRING")

# Or auto-merge schema on write:
df.write.mode("append").option("mergeSchema", "true").save("...")
```

Both formats handle:
- Add column (nullable).
- Drop column.
- Rename column (with constraints).
- Reorder columns.
- Type promotion (int → long).

Old Parquet files are read with the new schema; missing columns return null.

### 12.3 Backward compatibility

When consumers read newly-added columns from old files:
- Iceberg: returns null (consistent).
- Delta: returns null.
- Plain Parquet: schema mismatch error unless mergeSchema enabled — and then can be slow.

For schema-stable production: Iceberg/Delta is mandatory.

### 12.4 Forward compatibility

When consumers expect a new column on old data: use defaults.

```sql
ALTER TABLE events ALTER COLUMN device_model SET DEFAULT 'unknown'
```

Iceberg supports column defaults.

### 12.5 Schema enforcement

Write to a Delta/Iceberg table with mismatched schema fails by default. Mitigates dirty data. For tolerant ingestion: opt-in mergeSchema; for strict pipelines: fail-fast.

### 12.6 Trade-offs

| Approach | Trade |
|---|---|
| **Iceberg / Delta** | Native evolution; metadata-versioned |
| **Hive + manual recompute** | Painful; legacy |
| **Avro with registry** | Stream-friendly; doesn't help with batch tables |
| **Raw Parquet** | Fragile; mergeSchema slow |

### 12.7 What I'd actually do

Iceberg as the table format. Schema evolution via ALTER TABLE. mergeSchema only for known-safe streaming ingestion.

---

## 13. Scenario 12 — Time-Series Aggregations

### 13.1 The problem

Hourly metric rollups over a year of data. Window functions, percentiles, trend computations.

### 13.2 The aggregation

```python
hourly = (events
    .withColumn("hour", F.date_trunc("hour", "event_time"))
    .groupBy("hour", "service")
    .agg(F.count("*").alias("count"),
         F.expr("percentile_approx(latency, 0.95)").alias("p95"),
         F.sum("revenue").alias("revenue")))
```

### 13.3 percentile_approx vs percentile

```
percentile: exact; sorts the data; expensive.
percentile_approx: t-digest based; approximate; fast.

For dashboards: approx.
For SLO measurement: exact (or larger sample).
```

### 13.4 Lag/lead for trend

```python
window = Window.partitionBy("service").orderBy("hour")
hourly_with_trend = hourly.withColumn(
    "prev_hour_count", F.lag("count").over(window))
hourly_with_trend = hourly_with_trend.withColumn(
    "trend", F.col("count") / F.col("prev_hour_count") - 1)
```

### 13.5 Materializing rolling windows

```python
# 7-day rolling average
window_7d = Window.partitionBy("service") \
    .orderBy(F.unix_timestamp("hour")) \
    .rangeBetween(-7*86400, 0)

hourly_with_rolling = hourly.withColumn(
    "count_7d_avg", F.avg("count").over(window_7d))
```

### 13.6 Storage for time-series Spark output

Common pattern: Spark batch produces hourly aggregates → write to Druid for dashboards (best path) or Iceberg + Athena for ad-hoc.

### 13.7 What I'd actually do

For batch time-series aggregation: Spark with window functions. For sub-second dashboard: Spark output → Druid. Iceberg for warehouse / ad-hoc.

---

## 14. Scenario 13 — Graph Processing

### 14.1 The problem

Social graph: 1B users, 100B edges. Compute PageRank, community detection, k-hop neighborhoods.

### 14.2 GraphFrames

```python
from graphframes import GraphFrame

vertices = users  # df with 'id' column
edges = follows   # df with 'src', 'dst' columns

g = GraphFrame(vertices, edges)

# PageRank
ranks = g.pageRank(resetProbability=0.15, maxIter=10)

# Connected components
cc = g.connectedComponents()

# Triangle count
tc = g.triangleCount()
```

### 14.3 Performance

For 100B edges: each PageRank iteration is a shuffle of all edges. 10 iterations = 10× shuffle of 100B edges = expensive.

```
Optimization:
  - Partition edges by source vertex.
  - Cache vertices (low cardinality) in memory.
  - Reduce iterations by tuning convergence threshold.
```

### 14.4 Alternatives

| Approach | Trade |
|---|---|
| **Spark GraphFrames** | Spark-native; Java/Python |
| **Apache GraphX** | Spark legacy; Scala |
| **Neptune** | Managed graph DB; smaller scale |
| **JanusGraph** | Self-managed; scaling |
| **Pregel-style custom** | Bespoke; max performance |
| **Custom MapReduce-style** | For specific algorithms (PageRank in raw Spark) |

For very large graphs: pure Spark with carefully crafted custom algorithms often outperforms GraphFrames.

### 14.5 What I'd actually do

For graph processing at MAANG: Spark with custom message-passing for known algorithms (PageRank, BFS, etc.). GraphFrames for prototyping. Neptune for online queries (not analytics).

---

## 15. Scenario 14 — Data Quality Checks

### 15.1 The problem

Pipeline outputs need to meet quality criteria: row counts within X% of yesterday, no negative revenues, schema match, primary key uniqueness.

### 15.2 Deequ / Great Expectations / Custom

```python
from pydeequ.checks import Check, CheckLevel
from pydeequ.verification import VerificationSuite

check = Check(spark, CheckLevel.Error, "events_quality") \
    .hasSize(lambda x: x > 1_000_000) \
    .isUnique("event_id") \
    .isNonNegative("revenue") \
    .hasMin("event_time", lambda x: x >= yesterday)

result = VerificationSuite(spark) \
    .onData(events) \
    .addCheck(check) \
    .run()
```

Deequ (Amazon-built): native Spark library for data quality. Computes metrics, runs constraints, generates reports.

### 15.3 The data-quality pipeline

```
Raw ingest → DQ check → if pass: promote to clean → if fail: alert + freeze.

Layered checks:
  1. Schema match (file load fails fast).
  2. Row count delta (today vs yesterday ± 5%).
  3. Per-column nulls / distributions.
  4. Cross-table referential (if applicable).
  5. Business rules (revenue > 0, etc.).
```

### 15.4 Profiling

Periodically profile data:

```python
from pydeequ.profiles import ColumnProfilerRunner

profile = ColumnProfilerRunner(spark) \
    .onData(events) \
    .run()

# Auto-detects: distinctness, completeness, statistics per column.
```

Detect drift; surface anomalies before they break dashboards.

### 15.5 Trade-offs

| Approach | Trade |
|---|---|
| **Deequ** | Spark-native; mature |
| **Great Expectations** | Polyglot; DOps-friendly |
| **dbt tests** | SQL-warehouse; not Spark-native |
| **Custom asserts** | Light; ad-hoc |

### 15.6 What I'd actually do

For Spark pipeline DQ:
- Deequ checks at each stage.
- Profiling for drift detection.
- Failed checks → alert + halt pipeline (don't promote bad data).

---

## 16. Scenario 15 — Spark SQL for Ad-Hoc Analytics

### 16.1 The problem

Analysts need to write SQL against Iceberg/Delta tables. Need ms-to-second query latency for the 90% case.

### 16.2 Setup

```python
spark.sql("""
SELECT date_trunc('day', event_time) as day,
       country,
       count(*) as events,
       sum(revenue) as revenue
FROM events
WHERE event_time >= current_date - interval 30 day
GROUP BY 1, 2
""")
```

Spark SQL via thrift server / JDBC for BI tools. Catalyst optimizes; AQE adapts.

### 16.3 vs Trino / Presto

```
Trino / Presto:
  Designed for ad-hoc SQL; faster start time; better concurrency.
  Lower latency for many small queries.
  
Spark SQL:
  Same engine as Spark; reuses code; better at huge complex queries.
  Higher latency per query (driver overhead).
  
For BI / ad-hoc: Trino often wins.
For ETL / heavy queries: Spark.
```

### 16.4 The thrift server pattern

```bash
$SPARK_HOME/sbin/start-thriftserver.sh

# JDBC: jdbc:hive2://thrift-server:10000
# Tableau / Looker / Superset / DBeaver connect via JDBC.
```

Thrift server runs a long-lived Spark application; sub-second queries possible (no per-query startup).

### 16.5 What I'd actually do

For ad-hoc SQL on the data lake: Trino (or Athena) for analyst use; Spark SQL for ETL and complex queries. Both query the same Iceberg tables.

---

## 17. Scenario 16 — ML Model Training (MLlib + Beyond)

### 17.1 The problem

Train a logistic regression on 1B rows × 1000 features.

### 17.2 MLlib

```python
from pyspark.ml.classification import LogisticRegression
from pyspark.ml.feature import VectorAssembler

assembler = VectorAssembler(inputCols=feature_cols, outputCol="features")
data = assembler.transform(raw_df)

lr = LogisticRegression(featuresCol="features", labelCol="label", maxIter=100)
model = lr.fit(data)
```

MLlib distributed training: gradient descent across executors. Decent for simple models; slower than specialized tools for complex.

### 17.3 vs alternatives

```
MLlib: integrated with Spark; OK for simple distributed models.
TensorFlow on Spark / Horovod: deep learning at scale.
SparkXgBoost: XGBoost integration; popular.
Spark for feature prep + framework for training: most common.
```

In practice: Spark for feature engineering + I/O; ML training on dedicated infra (SageMaker, GCP AI Platform, Ray).

### 17.4 Hyperparameter tuning

```python
from pyspark.ml.tuning import CrossValidator, ParamGridBuilder
grid = ParamGridBuilder().addGrid(lr.regParam, [0.01, 0.1, 1.0]).build()
cv = CrossValidator(estimator=lr, estimatorParamMaps=grid, numFolds=5)
best = cv.fit(data)
```

Distributed CV: each fold on a Spark task. For complex hyperparameter search: use Optuna / Ray Tune externally.

### 17.5 Trade-offs

| Approach | Trade |
|---|---|
| **MLlib distributed** | Integrated; simple models |
| **XGBoost on Spark** | Best for tabular |
| **Horovod / TF / PyTorch on Spark** | DL on Spark cluster |
| **Spark prep → SageMaker** | Spark for I/O, SM for training |
| **Ray on Spark** | Hybrid; rich Python ecosystem |

### 17.6 What I'd actually do

For ML at MAANG:
- Spark for ETL + feature engineering.
- XGBoost on Spark or distributed training framework (Ray/Horovod) for training.
- Avoid using MLlib for production-grade complex models.

---

## 18. Scenario 17 — Pandas API on Spark

### 18.1 The problem

Data scientists know pandas. Production data is too big. Don't want to teach Spark API.

### 18.2 Pandas API on Spark (PySpark.pandas)

```python
import pyspark.pandas as ps

df = ps.read_parquet("s3://data/events/")
result = df.groupby("country")["revenue"].sum()
result.head()
```

Translates pandas operations to Spark DataFrame. ~80% of pandas API supported.

### 18.3 Performance

For most operations: similar to native Spark.

For pandas-specific patterns: may be slow or unsupported. Iterating row-by-row, using `apply` heavily, etc., translates poorly.

### 18.4 When it shines

Data scientists with established pandas workflows; small-to-mid data that doesn't quite fit memory.

### 18.5 When to switch to native PySpark

When perf matters; when using complex Spark features (Catalyst hints, custom partitioning, etc.).

### 18.6 What I'd actually do

For prototyping and exploration: pandas-on-Spark. For production: native PySpark.

---

## 19. Scenario 18 — Cross-Source Federation

### 19.1 The problem

Query joins data from Iceberg (warehouse), Postgres (transactional), DynamoDB (KV), Cassandra (time-series).

### 19.2 Spark connectors

```python
postgres = spark.read.format("jdbc") \
    .option("url", "...").option("dbtable", "users").load()

dynamodb = spark.read.format("com.amazon.spark-dynamodb") \
    .option("tableName", "events").load()

cassandra = spark.read.format("org.apache.spark.sql.cassandra") \
    .options(table="metrics", keyspace="ks").load()

iceberg = spark.read.format("iceberg").load("warehouse.events")

joined = iceberg.join(postgres, "user_id") \
                .join(dynamodb, "session_id") \
                .join(cassandra, "device_id")
```

### 19.3 Considerations

```
JDBC source (Postgres): pulls all data over JDBC. Slow for large tables.
  Mitigations: predicate pushdown (where), partitioning hints (numPartitions, partitionColumn).

DynamoDB: reads via Scan or partition key parallel. Expensive RCU.
  Mitigations: filter by PK; use S3 export of DDB.

Cassandra: predicate pushdown on partition key. Slow for full scans.

Iceberg: native; fast.
```

### 19.4 Materialize vs federate

For repeated joins: materialize source data into Iceberg (CDC or daily ETL); join inside Iceberg only.

For one-off ad-hoc: federate.

### 19.5 Trade-offs

| Approach | Trade |
|---|---|
| **Federate at query** | Real-time data; slow joins |
| **Materialize via CDC** | Fast queries; pipeline ops |
| **Trino + connectors** | Better for ad-hoc federation |

### 19.6 What I'd actually do

For production: materialize sources into Iceberg via CDC; join in warehouse. Federation only for ad-hoc.

---

## 20. Scenario 19 — Resource Sizing

### 20.1 The mental model

```
Cluster:
  - N nodes, each with cores C and memory M.
  - Driver: 1 (often dedicated node).
  - Executors: distributed across nodes.

Per executor:
  - cores (4-5 sweet spot)
  - memory (4-32 GB)
  - off-heap (Tungsten)
  - overhead (~10-20% of memory)

Per task: 1 core; 1 partition processed.

Parallelism: total tasks = total cores across executors.
```

### 20.2 The rule of thumb

```
For an r5.4xlarge node (16 cores, 128 GB):
  - Reserve ~2 cores, ~10 GB for OS + YARN.
  - Available: 14 cores, 118 GB.
  - 4 executors × 4 cores × 24 GB = 16 cores, 96 GB (some headroom).
  
Or 1 executor × 14 cores × 100 GB = single fat executor.
  Pro: minimizes shuffles between executors on same node.
  Con: GC pressure on 100 GB heap.

Sweet spot: 4-5 cores per executor; multiple executors per node.
```

### 20.3 Dynamic allocation

```python
spark.conf.set("spark.dynamicAllocation.enabled", "true")
spark.conf.set("spark.dynamicAllocation.minExecutors", "10")
spark.conf.set("spark.dynamicAllocation.maxExecutors", "100")
spark.conf.set("spark.dynamicAllocation.initialExecutors", "20")
```

Spark adds/removes executors based on pending tasks. Combined with K8s/YARN auto-scaling for true elasticity.

### 20.4 Memory tuning

```
spark.executor.memory = N GB (heap)
spark.executor.memoryOverhead = 0.1 × N GB (JVM overhead)
spark.executor.memoryOffHeap.size = M GB (Tungsten off-heap)

For shuffle-heavy: increase shuffle spill.
For caching: ensure storage memory fraction.

Common bug: setting too small → OOM; too large → fewer executors → less parallelism.
```

### 20.5 Spot instances

```
Spot can save 60-80%. Risk: terminations.

Mitigations:
  - Multiple instance types (diversify).
  - Spot fleet with on-demand fallback.
  - Driver on on-demand (cheap; loss = full job restart).
  - Task-level retries (Spark retries failed tasks).
  - Checkpointing for long-running.

EMR / Databricks have Spot support built in.
```

### 20.6 What I'd actually do

For batch ETL:
- 4-core executors with 24 GB memory.
- Dynamic allocation with min/max bounds.
- Spot for executors; on-demand for driver.
- Auto-scaling on cluster level (EMR managed scaling).

---

## 21. Performance Tuning Deep Dive

### 21.1 Profile first

```
Spark UI:
  - Stages tab: identify slow stage.
  - Tasks: skew (max vs median time).
  - Shuffle: read/write size.
  - Storage: cached RDDs/DFs memory usage.
  - SQL tab: physical plan.

Tools:
  - Spark History Server.
  - Databricks profiles.
  - Sparklens (open source).
```

### 21.2 The common bottlenecks

```
1. Default 200 shuffle partitions on big data → OOM or slow.
   Fix: tune to total_shuffle_bytes / 200 MB.

2. Skew → straggler tasks.
   Fix: AQE skew join, salting, hot/cold split.

3. Small files (1000s of 1MB Parquet) → slow read.
   Fix: compact via .repartition() before write.

4. Too many files → driver OOM listing.
   Fix: same compaction.

5. Cartesian / cross join unintended.
   Fix: review join keys; SQL EXPLAIN.

6. Wide transformation when narrow possible.
   Fix: reduce shuffles via mapPartitions.

7. Deep DataFrame lineage.
   Fix: checkpoint or cache mid-pipeline.

8. Driver-side collection.
   Fix: avoid .collect() on big data; toPandas() collects all.
```

### 21.3 EXPLAIN reading

```python
df.explain(mode="extended")

# Look for:
# - PartitionFilters: predicate pushdown working?
# - DataFilters: same
# - PushedFilters: same for Parquet
# - Exchange: shuffle steps
# - BroadcastHashJoin / SortMergeJoin / ShuffledHashJoin: join type
# - HashAggregate: aggregation
```

Reading EXPLAIN is a staff-level skill. Most performance work starts here.

### 21.4 Caching

```python
df.cache()  # MEMORY_AND_DISK
df.persist(StorageLevel.MEMORY_ONLY_SER)  # serialized; less memory, more CPU
df.unpersist()  # free
```

Cache when:
- DataFrame used in multiple actions.
- Source is slow to read (S3 list, JDBC).
- Computation is expensive.

Don't cache:
- One-shot DataFrames.
- DataFrames bigger than memory (cache thrashing).

### 21.5 Predicate pushdown

```python
# Pushed: filter applied at storage layer (only matching rows read)
df = spark.read.parquet("s3://...").filter("date = '2026-04-28'")

# Not pushed: filter after a transformation
df = spark.read.parquet("s3://...").withColumn("d", F.col("date")).filter("d = '...'")
```

Always check EXPLAIN for `PushedFilters`. Format must support pushdown (Parquet, ORC ✓; JSON, CSV partial).

### 21.6 Broadcast hint

```python
df.join(F.broadcast(small_df), "key")
# or SQL:
spark.sql("SELECT /*+ BROADCAST(small) */ * FROM big JOIN small ON big.k = small.k")
```

When AQE doesn't auto-broadcast: manual hint.

### 21.7 Repartition vs coalesce

```
.repartition(N): full shuffle to N partitions.
.coalesce(N): reduce to N partitions, no shuffle (just merges).

Use coalesce when reducing (e.g., before write).
Use repartition when increasing or balancing.
```

---

## 22. Anti-Patterns — Staff-Level Red Flags

### 22.1 .collect() / .toPandas() on big data

Pulls everything to driver. OOM for anything > ~1 GB. Use `.show(N)`, `.take(N)`, or save to S3.

### 22.2 .count() in production hot paths

`count()` is a full action: triggers full execution. Avoid in conditional logic.

### 22.3 UDFs when built-in functions exist

UDFs are slow (de/serialization between JVM and Python). Use `pyspark.sql.functions` whenever possible. For Python logic: pandas UDFs (vectorized).

### 22.4 Inefficient string operations

`F.concat`, `F.substr` are fine. `regexp_extract` is slow at scale; consider parsing upfront.

### 22.5 Default 200 shuffle partitions on 10 TB job

Tune to data volume. Otherwise OOM or slow.

### 22.6 Reading unpartitioned data + filtering

Without partition pruning, full scan. Always filter on partition column when possible.

### 22.7 Wide transformations in loops

```python
# BAD
for x in some_list:
    df = df.withColumn(f"col_{x}", ...)
# Many incremental transformations; long DAG; slow planning.

# GOOD
df = df.select("*", *[F.lit(...).alias(f"col_{x}") for x in some_list])
```

### 22.8 Broadcasting too-large DataFrames

If broadcast > driver memory: OOM. Tune `autoBroadcastJoinThreshold` carefully.

### 22.9 Caching everything

Memory pressure; eviction; slows everything. Cache only what's reused.

### 22.10 Forgetting checkpoint

Long lineage → planning slow. Checkpoint mid-pipeline materializes the data.

### 22.11 Using RDDs unnecessarily

DataFrames are faster (Catalyst optimizes). RDDs for advanced/legacy only.

### 22.12 Mixing actions and transformations sloppily

`df.write.format(...).save()` then `df.cache().count()` — second action recomputes if not cached.

### 22.13 Single big executor

One executor with 100 GB RAM and 32 cores: GC pressure, single point of failure within node. Multiple smaller executors better.

### 22.14 Tiny Parquet files

Cluster output: 100 partitions × 100 small writers = 10K files. Subsequent reads are slow. Coalesce before write.

### 22.15 No idempotency in streaming sinks

Sink doesn't dedup → on retry, duplicates. Use Iceberg / Delta MERGE or dedup downstream.

### 22.16 OOM "fixed" by raising memory

If you keep raising executor memory: investigate skew, join strategy, or query plan first.

### 22.17 No checkpoint location for streaming

Streaming without checkpoint = no recovery. Always set `checkpointLocation`.

### 22.18 Running production pipelines on a notebook

Notebooks for prototyping. Production: airflow / dbt / dagster orchestration.

### 22.19 Not handling Spot interruptions

Long jobs without checkpoint → Spot kill → restart from scratch.

### 22.20 Mixing batch and streaming logic in one job

Cleaner: separate jobs, separate code paths, separate clusters.

---

## 23. Decision Framework

### 23.1 Step 0 — Is Spark the right tool?

```
Yes:
  - Data > 100 GB (single-node not enough).
  - Distributed compute available (cluster or managed).
  - Familiarity with DataFrame / SQL APIs.
  - Need batch + streaming in one engine.

Maybe:
  - 10-100 GB: pandas / polars often suffice.
  - Streaming-first: Flink may be better.
  - Pure SQL on warehouse: BigQuery / Snowflake / Trino.

No:
  - Sub-second latency.
  - Transactional DB workload.
  - Tiny data (< 10 GB).
```

### 23.2 Step 1 — Pick deployment

EMR / Databricks / EKS / Glue / managed alternative.

### 23.3 Step 2 — Choose table format

Iceberg / Delta / Hudi for any table you'll evolve. Plain Parquet for short-lived intermediate.

### 23.4 Step 3 — Schema and partitioning

Partition by date (or natural time column). Bucket if joins repeated. Pre-aggregate where possible.

### 23.5 Step 4 — Resource sizing

4-5 core executors. Dynamic allocation. Spot for batch.

### 23.6 Step 5 — Performance tuning

AQE on. Tune shuffle partitions. Verify broadcast joins.

### 23.7 Step 6 — Operational lifecycle

Orchestration (Airflow). DQ checks (Deequ). Monitoring (Spark UI history). Cost dashboards.

---

## 24. Mental Models a Staff Engineer Carries

1. **Lazy DAG, not imperative.** Spark plans; you don't execute line by line.

2. **Shuffles are the cost.** Most performance work is reducing or optimizing them.

3. **AQE is the modern speedup.** Always enable.

4. **Default shuffle partitions are wrong.** Tune to data volume.

5. **Broadcast small dimensions.** Broadcast threshold default is 10 MB; raise.

6. **Skew is inevitable.** AQE first; salting if extreme.

7. **Predicate pushdown matters.** Filter at source; verify with EXPLAIN.

8. **Parquet/Iceberg are the table formats.** Plain Parquet only for staging.

9. **Streaming is micro-batch.** Latency floor 100s of ms; Flink for sub-second.

10. **Stateful streaming needs RocksDB.** Default state store is slow.

11. **Watermarks GC state.** Without watermark, state grows unbounded.

12. **Idempotent sinks for exactly-once.** Iceberg/Delta MERGE; or epoch IDs.

13. **Bucket for repeated joins.** Pre-pay shuffle.

14. **EXPLAIN reading is a staff-level skill.** Read PushedFilters, Exchange, JoinType.

15. **.collect() is a foot-gun.** Almost never the right action.

16. **UDFs are slow.** Use built-in / pandas UDFs.

17. **Caching is targeted, not blanket.** Cache only re-used DataFrames.

18. **Dynamic allocation + Spot for cost.** Tune both bounds.

19. **Compact small files.** 10K small files = 100× slow.

20. **Backfills via REPLACE WHERE.** Iceberg/Delta atomic.

21. **DQ checks at every stage.** Deequ or equivalent.

22. **Federation is for ad-hoc.** Materialize for production.

23. **Notebooks for prototyping; orchestration for production.** Don't blur.

24. **Driver is sacred.** Minimize driver work; never collect big data.

25. **Checkpoint long lineages.** Planning gets slow on deep DAGs.

26. **Memory tuning by profile.** Don't guess; read Spark UI.

27. **Spark vs Flink vs warehouse: per-shape.** Defend the choice.

28. **Boring is a feature.** A Spark pipeline that runs nightly for 5 years and quietly works is the goal.

---

## 25. Closing Notes

Spark is the workhorse. It's not the fastest at any one thing — Flink does streaming better, ClickHouse does aggregates better, Trino does SQL ad-hoc better, BigQuery does serverless better. But Spark is the *most general* compute engine: batch + streaming, SQL + Python + Scala + R, MLlib + GraphX, every cloud, every storage backend.

Staff-level mastery means:
- Recognizing when Spark is the right tool *and* defending the choice.
- Reading the DAG, EXPLAIN, and Spark UI fluently.
- Sizing executors correctly the first time.
- Anticipating skew and shuffles before they bite.
- Knowing when to reach for Flink or Trino instead.

The best Spark pipelines are *boring*: nightly batches that complete in 4 hours every night, streaming jobs that run for months without restart, ML pipelines feeding production with no surprises. Boring is the result of careful design: schema, partitioning, executor sizing, AQE tuning, DQ checks, monitoring, alerts, retries, idempotency.

Done well, Spark is a 10-year investment that scales with the business. Done poorly, it's a $100K/month bill on stuck-in-shuffle jobs that nobody understands.

The art is in the design.

> Companion docs:
> - `awsS3ScenariosAtScale.md` — storage layer Spark reads/writes.
> - `dynamoDBScenariosAtScale.md` — when KV is right; Spark complements.
> - `druidScenariosAtScale.md` — Spark batch produces aggregates Druid serves.
> - `eventPlatformsAtScale.md` — Kafka/MSK ingestion before Spark.
> - `logProcessingAndAggregation.md` — pipeline overview.