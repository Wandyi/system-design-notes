# 17 · Pinot and Samza — LinkedIn's Streaming + OLAP Stack

Two of LinkedIn's most-cited engineering exports. Pinot does interactive OLAP at sub-second latency over billions of rows; Samza is the stream-processing engine that LinkedIn invented and uses for the bulk of its real-time pipelines.

A staff candidate will be asked about either or both — sometimes directly ("design Pinot"), sometimes as a constraint in a system-design ("how would you serve real-time member-facing analytics?").

---

## Part A — Pinot

## 17.1 What is Pinot?

> A distributed, columnar, real-time OLAP datastore designed for low-latency analytical queries over very large datasets.

Apache project since 2018 (LinkedIn-incubated, contributed). Now a top-level Apache project with significant external contributions (Uber, Stripe, others).

## 17.2 Why Pinot exists

LinkedIn needed to serve **interactive, sub-second** analytical queries over multi-billion-row datasets, with both **batch-ingested historical** data and **streaming real-time** data. Existing options didn't fit:
- **Druid** — close, but at LinkedIn's scale operational cost was high; some query patterns weren't fast enough.
- **Hive / Trino** — too slow for member-facing dashboards.
- **Vertica / commercial** — cost prohibitive.

So they built Pinot.

## 17.3 Architecture

```
   Real-time Kafka                Offline batch (Spark / Hadoop)
        │                                       │
        ▼                                       ▼
   ┌───────────────┐                ┌───────────────────────┐
   │ Realtime      │                │ Offline               │
   │ Server        │                │ Server                │
   │ (consumes     │                │ (loads pre-built      │
   │  Kafka, builds │                │  segments from       │
   │  segments)    │                │  HDFS/object store)  │
   └─────┬─────────┘                └─────┬─────────────────┘
         │                                │
         └─────────┬────────────────┬─────┘
                   │                │
                   ▼                ▼
              ┌───────────────────────┐
              │   Broker             │ ← routes query, fans to servers
              └─────────┬─────────────┘
                        │
                        ▼
              ┌───────────────────────┐
              │  Controller (Helix)  │ ← cluster mgmt, table config
              └───────────────────────┘
```

Key components:

- **Server** — holds **segments** (chunks of data); answers per-segment queries.
- **Broker** — receives queries, plans, scatters to servers holding relevant segments, gathers, returns.
- **Controller** — cluster metadata, table configs; uses Apache Helix.
- **Minion** — offline segment-building, compaction.
- **ZooKeeper** — coordination (or KRaft-equivalent for Helix).

## 17.4 Data model

- **Table** — analogous to SQL table. Has a schema (dimension cols, metric cols, time col).
- **Real-time table**: ingests from Kafka.
- **Offline table**: ingests from HDFS / object storage via batch jobs.
- **Hybrid table**: real-time + offline together; queries union them; offline takes precedence for time ranges covered by both.

### Schema

```yaml
dimensions:
  - member_id (LONG)
  - country (STRING)
  - device (STRING)
metrics:
  - impressions (LONG)
  - clicks (LONG)
time:
  - event_time (TIMESTAMP, granularity: MINUTE)
```

## 17.5 Segments

The fundamental unit of storage and parallelism.

- **Real-time segments**: built in memory on a Realtime Server as Kafka messages stream in; flushed to disk when row count / time threshold hit; eventually finalized & uploaded.
- **Offline segments**: built by a batch job (Spark) and uploaded to Pinot.
- Segments are immutable once finalized.
- Each segment has metadata (min/max time, dimension cardinalities, etc.) so the broker can prune.

## 17.6 Columnar storage and indexes

Per-segment storage is columnar. Per column, several encodings:
- **Dictionary encoded** + bitmap inverted index — typical for low-cardinality dimensions.
- **Raw / forward index** — for high-cardinality columns.
- **Sorted index** — for time columns.
- **Range index** — for numeric range queries.
- **Star-tree index** — pre-aggregated cube-like structure; the **killer Pinot feature** for low-latency aggregation queries with high-cardinality dimensions.
- **JSON / Text index** — for text search inside a column.
- **Geo index** — for spatial queries.

### Star-tree index

The idea: for a query like `SELECT SUM(clicks) WHERE country='US' AND device='mobile' GROUP BY day`, you don't want to scan billions of rows.

A **star tree** is a tree where each level is a dimension; leaves are aggregated metrics. The tree branches on each dimension value, with a "star" branch summing all values for that dimension. A query traverses the tree from root, taking the relevant branch at each dimension level → reads pre-aggregated leaves → returns in microseconds.

Trade-off: storage overhead, build time, maintenance. Worth it for hot queries.

## 17.7 Real-time ingestion

- Each real-time table partition is consumed by one server.
- Server materializes rows in memory in an "in-memory segment".
- When `realtime.threshold.segment.size` or `realtime.threshold.segment.rows` or `realtime.threshold.segment.time` is hit, the segment is sealed.
- Sealed segments are committed to deep storage (HDFS / Azure Blob).
- New in-memory segment starts.

### Trade-offs

- Lower thresholds → more segments, more metadata, fresher data, more overhead.
- Higher thresholds → bigger segments, faster queries, but staler data and more memory pressure.

## 17.8 Query path

1. Client sends SQL to Broker (Pinot supports SQL plus a legacy PQL).
2. Broker parses, plans.
3. Broker identifies relevant segments via metadata (time-range pruning, partition pruning).
4. Broker scatters per-segment queries to Servers.
5. Each Server executes on each segment (using indexes, star-trees) and returns partial aggregates.
6. Broker merges partial aggregates → final result.

Latency target: median tens of ms; p99 < 1s for the vast majority of queries.

## 17.9 Use cases at LinkedIn

- **Who viewed my profile** — per-member aggregations.
- **Creator analytics dashboards**.
- **Recruiter analytics** — funnel metrics.
- **LMS reporting** — campaign performance.
- **Anomaly detection** — real-time monitoring.
- **Trust / abuse signals** — abuse patterns from event streams.

## 17.10 Failure modes

- **Server crash** — segments re-replicated via Helix from peers.
- **Slow server** — broker times out per-segment query; partial result with a warning.
- **Bad segment** — quarantined; query continues with the rest.
- **Kafka lag in real-time** — segments stale; alert.
- **Deep-storage outage** — sealed segments can't be uploaded; in-memory segments grow; recovery on storage restoration.

## 17.11 Operational concerns

- **Segment management** at scale: hundreds of thousands of segments per table.
- **Capacity** — memory dominates; segment in-memory representation must fit.
- **Query workload management** — tenant isolation, query throttling.
- **Tier separation** — hot data on SSD, cold on slower disk or remote (tiered storage in newer versions).
- **Schema changes** — backward-compatible additions are fine; type changes or removals require careful versioning.

---

## Part B — Samza

## 17.12 What is Samza?

> A distributed stream-processing framework that uses Kafka for input, output, and state changelog, and Apache Helix (or YARN historically) for cluster management.

Open-sourced 2013, Apache project 2014. Born to serve LinkedIn's stream-processing needs once Kafka was established.

## 17.13 Mental model

- A Samza job consumes one or more Kafka input topics.
- Each input partition is owned by exactly one task (no cross-task ordering issues).
- A task processes one message at a time (or in a windowed batch); maintains local state in RocksDB.
- Local state changes are also written to a **changelog Kafka topic** so the state is recoverable if the task crashes.
- Output is written to Kafka output topics.

## 17.14 Architecture

```
   Input Kafka topic                  Output Kafka topic
   (partitions p0..pN)                (partitions q0..qM)
        │                                       ▲
        ▼                                       │
   ┌──────────────────────────────────────────────┐
   │ Samza Container (1 per machine usually)     │
   │   ┌──────────┐  ┌──────────┐  ┌──────────┐  │
   │   │ Task p0  │  │ Task p1  │  │ Task p2  │  │
   │   │ + RocksDB│  │ + RocksDB│  │ + RocksDB│  │
   │   └──────────┘  └──────────┘  └──────────┘  │
   └──────────────────────────────────────────────┘
                       │
                       ▼ changelog
                  Kafka changelog topic
```

## 17.15 State and local stores

A defining Samza feature: **stateful processing with low-latency local access**.

- Each task has its own RocksDB.
- Writes to local state also written to changelog Kafka topic.
- On task restart, state is restored by replaying the changelog.
- Compaction ensures the changelog topic stays bounded.

Use cases:
- **Joining streams**: join click events with a session table maintained from session-events.
- **Windowed aggregates**: minute-by-minute counts.
- **Enrichment**: lookup member metadata at processing time (member-meta-table updated from another Kafka topic).

## 17.16 Time semantics

- **Processing-time** (default): when the event arrives.
- **Event-time**: from the event's timestamp.
- **Windowing**: tumbling, hopping, session windows.
- **Watermarks**: for handling out-of-order data (later versions; Flink is stronger here).

## 17.17 Samza vs. Kafka Streams vs. Flink

All three are competitors with overlapping use cases. Quick rubric:

| Property | Samza | Kafka Streams | Flink |
|---|---|---|---|
| Origin | LinkedIn | Confluent | TU Berlin → Apache |
| Deployment | Service / Helix or YARN | Library (embed in JVM app) | Service (job cluster) |
| Storage backend | RocksDB | RocksDB | Pluggable |
| Exactly-once | Yes (Kafka transactions) | Yes | Yes |
| Event-time + watermarks | Limited | Limited | Strong |
| Windowing | Tumbling, hopping | Tumbling, hopping, session | Rich |
| State scale | Large (TB) | Large | Large |
| SQL | Limited | KSQL | Flink SQL (mature) |
| LinkedIn adoption | Heavy historical | Some | Growing |

## 17.18 Use cases at LinkedIn

- **Tracking event enrichment** — join raw events with member-metadata to produce enriched events.
- **Real-time aggregation** — counts, sums, percentile updates.
- **Feature pipelines** — compute ML features from streams; write to Venice.
- **Anti-abuse** — real-time abuse score per event.
- **Notifications routing** — Samza job watches event streams, decides who should be notified.

## 17.19 Failure modes (Samza)

- **Task crash** — re-assigned by Helix; state restored from changelog; processing resumes.
- **Changelog Kafka cluster outage** — task can't checkpoint state changes; stalls.
- **Slow input** — backpressure naturally via Kafka consumer lag.
- **Bug in processing logic** — replay from earlier offset to reprocess after fix.

## 17.20 Common interview questions

> **"When should you use Pinot vs. Druid?"**
Pinot is generally more memory-efficient and faster for star-tree-friendly queries. Druid is more mature in some operational tooling. Pinot is LinkedIn's choice and the displacing system.

> **"When should you use Pinot vs. ClickHouse?"**
ClickHouse is excellent for SQL analytics at scale, often with simpler ops than Pinot. Pinot is stronger for very-low-latency dashboard-style queries (sub-second on billions of rows) and for hybrid real-time/offline tables. Both have their place; the choice often comes down to ecosystem and ops familiarity.

> **"How does Pinot achieve sub-second latency?"**
Columnar storage + multiple index types (inverted, range, sorted, star-tree) + segment-level parallelism + careful query planning + in-memory caching of segment metadata.

> **"How do you handle a schema change in Pinot?"**
Backward-compatible additions (new column with default) — fine. Type changes — require reingest. Document the deprecation; coordinate consumers; rebuild offline segments.

> **"How would you join two streams in Samza?"**
Stateful join: each task holds RocksDB store keyed by join key. On each event, lookup in store; emit joined event if match; else buffer. Window for time-bounded joins.

> **"What's the throughput of a single Samza task?"**
Depends on the work. Light enrichment: tens of thousands msgs/sec. Heavy stateful: thousands. Bounded mostly by RocksDB and JVM GC.

> **"How would you migrate a Samza job to Flink?"**
Common pattern: dual-pipeline. Run Samza and Flink in parallel; diff outputs. Once Flink stable, cut over and decommission Samza. Be careful about state migration: Flink state isn't binary-compatible with Samza RocksDB.

> **"How do you ensure exactly-once from Kafka → Samza → Pinot?"**
Kafka transactions for produce-consume-process. Pinot ingestion uses offset tracking; segments built deterministically; replays produce the same segments. Idempotency at all layers.

> **"How do you handle a late event in Samza?"**
Configurable max-lateness; events beyond it dropped or sent to a side topic; updates to already-emitted windows handled via "retraction" patterns or via Flink's stronger model.