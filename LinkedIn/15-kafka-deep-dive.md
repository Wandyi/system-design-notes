# 15 · Kafka — Deep Dive

Kafka was conceived inside LinkedIn in 2010 by Jay Kreps, Neha Narkhede, and Jun Rao, open-sourced in 2011, donated to the Apache Foundation in 2012. LinkedIn is still its largest single user — trillions of messages/day. Expect detailed Kafka questions in any infra-adjacent loop.

This file goes deep enough that you can defend any Kafka follow-up at the staff level.

## 15.1 Why Kafka existed in the first place

The problem at LinkedIn ~2010:
- Many producers: every service emits events (page views, ad impressions, clicks, profile updates, etc.).
- Many consumers: data warehouse, search indexer, recommendation pipelines, monitoring, fraud detection.
- O(N×M) point-to-point integrations was unmaintainable.
- Existing message queues (ActiveMQ, RabbitMQ) couldn't handle the throughput LinkedIn projected.

The insight: build a **distributed, persistent, replicated log** that producers append to and consumers read from at their own pace. The log is the contract.

## 15.2 Core data model

- **Topic** — a named, persistent log. Conceptually append-only.
- **Partition** — a topic is divided into N partitions. Each partition is a totally-ordered sequence. Messages in different partitions have no order between them.
- **Offset** — the position of a message within a partition. Monotonic integer per partition.
- **Producer** — writes messages to topics (optionally specifies a partition or a partition key).
- **Consumer** — reads messages from one or more partitions; tracks its own offset (committed to Kafka itself in a special `__consumer_offsets` topic).
- **Consumer group** — a set of consumers that cooperatively consume a topic; each partition is assigned to exactly one consumer in the group at a time.
- **Broker** — a Kafka server process. A cluster has many.
- **Replica** — each partition has one leader and N follower replicas on different brokers. Leader handles all writes; followers replicate.

## 15.3 Storage layout

Per broker, per partition, on local disk:
- A **log directory** containing **log segment files** (`00000000000000123456.log` and corresponding `.index` and `.timeindex`).
- Each segment file holds messages with offsets in a contiguous range.
- The **active segment** is being written; older segments are immutable.
- Messages are stored in a wire-compatible binary format (header + key + value + timestamp).
- The `.index` file is a sparse offset → file-position index for fast seek.

Compaction:
- **Retention by time / size**: old segments deleted.
- **Log compaction**: keep the latest value per key indefinitely. Used for topics with table semantics (e.g., `__consumer_offsets`).

## 15.4 Replication and durability

- A partition has a **leader** and N-1 **followers**.
- All writes go to the leader; followers fetch and replicate.
- The leader maintains an **ISR (in-sync-replicas)** set — replicas that are caught up.
- Producer can wait for `acks=0` (fire-and-forget), `acks=1` (leader only), `acks=all` (all in ISR).
- `acks=all` + `min.insync.replicas >= 2` is the durability sweet spot.

If leader fails:
- One of the ISR followers is elected new leader.
- If ISR is empty (rare, all replicas behind), config decides:
  - **Unclean leader election** allowed → may lose data, but availability preserved.
  - Otherwise → no leader, partition unavailable until recovery.

## 15.5 ZooKeeper vs. KRaft

Historically Kafka used ZooKeeper for cluster metadata (broker membership, controller election, partition assignment). **KRaft** mode (Kafka 2.8+ as preview, 3.x as production) replaces ZooKeeper with a Raft-based metadata quorum *inside* Kafka.

Benefits of KRaft:
- One fewer system to operate.
- Faster controller failover.
- Better scale (millions of partitions).
- Cleaner ops model.

LinkedIn was an early KRaft adopter. Staff candidates should know:
- KRaft uses a Raft quorum of controllers (typically 3 or 5).
- Metadata log is itself a Kafka log (special topic).
- Migration from ZK to KRaft is supported in modern Kafka versions.

## 15.6 Producer client

- **Batching**: messages are batched per-partition, flushed every `linger.ms` or when `batch.size` is reached.
- **Compression**: per-batch (gzip, snappy, lz4, zstd).
- **Partitioner**: default = murmur hash of key; sticky partitioner; round-robin if no key.
- **Idempotent producer**: producer-side sequence numbers + producer ID → exactly-once *per-partition* writes.
- **Transactional producer**: write to multiple partitions atomically — enables exactly-once across multiple topics or across consume-process-produce.

## 15.7 Consumer client

- **Pull-based** (consumers fetch).
- **Consumer group** — partitions distributed across members.
- **Rebalance**: when membership changes, partitions reassigned. Rebalance can be:
  - **Eager**: all consumers stop, reassign, restart. Causes a stop-the-world pause.
  - **Cooperative (Incremental)**: only affected partitions move. Modern default.
- **Offset commit**: consumer commits its position to Kafka periodically. On crash, restart from last committed offset.
- **Auto-commit vs. manual** — auto is convenient but loses fine-grained control of "did I process this?".

## 15.8 Exactly-once semantics

A famously hard topic.

Three flavors:
1. **At-most-once** — fire-and-forget; messages may be lost.
2. **At-least-once** — retries on failure; duplicates possible.
3. **Exactly-once** — Kafka achieves this with idempotent producer + transactions + isolation level on consumer.

How exactly-once-end-to-end works (for a consume-process-produce pipeline):
1. Consumer reads from input topic.
2. Application processes, writes results to output topic, and commits new consumer offset, all in a transaction.
3. Either both commit or neither.

Consumer isolation level `read_committed` ensures consumers only see committed transactions.

Important: **exactly-once for non-Kafka sinks** (e.g., a database) requires the sink to support idempotency or 2PC-like protocols. Kafka alone can't guarantee it.

## 15.9 Multi-DC: MirrorMaker 2 and Brooklin

- **MirrorMaker 2** replicates topics between Kafka clusters (active-passive or active-active).
- Uses Kafka Connect framework.
- Handles consumer-offset translation.
- Active-active: same topic on both clusters, prefixed to avoid cycles.

LinkedIn-built **Brooklin** is a more general data-streaming bridge: Kafka↔Kafka, Kafka↔Espresso CDC, etc. See `18-venice-databus-brooklin.md`.

## 15.10 Performance characteristics

Kafka's celebrated throughput:
- Linear writes to disk (sequential, ~hundreds of MB/s per disk).
- Zero-copy reads via `sendfile()` (kernel-to-network, bypass user space).
- OS page cache warm for recent reads → memory-speed.
- Batching amortizes per-message overhead.

LinkedIn's largest clusters:
- 100s of brokers per cluster.
- 100K+ partitions.
- 7M+ messages/sec sustained per cluster.
- 99th-percentile produce latency single-digit ms.

## 15.11 Operational concerns

- **Cruise Control** (open-sourced from LinkedIn) — auto-rebalances partition assignments for balanced disk / network / CPU.
- **Burrow** (open-sourced) — consumer lag monitoring. Avoids relying on consumers to self-report.
- **Disk space management** — retention policies tuned per-topic; cold tiering via Tiered Storage (KIP-405) — segments older than threshold offloaded to object storage.
- **Broker rolling restarts** — graceful: leader migration first, then restart, then re-replicate.
- **Schema registry** — separate service (Confluent or LinkedIn-internal) enforces Avro / Protobuf compatibility.

## 15.12 Common interview questions on Kafka

> **"What happens if a producer can't reach the leader?"**
Producer retries with exponential backoff (`retries` config). Metadata refresh forces re-discovery of new leader after failover. If `acks=all`, the producer waits for ISR ack before considering write committed.

> **"How do you guarantee message ordering?"**
Within a partition: guaranteed by Kafka. Across partitions: not guaranteed. To get global ordering, use a single partition (limits throughput). For per-key ordering, partition by key.

> **"How do you scale a consumer group?"**
Add more consumers up to the partition count. Beyond that, no parallelism; need to increase partitions. Re-partitioning is non-trivial (no online reshuffle); typically requires a topic migration.

> **"What's the tradeoff between batch size and latency?"**
Bigger batches: higher throughput, more memory pressure, higher latency. Lower batches: lower latency, lower throughput, higher per-message overhead.

> **"How do you handle a poison-pill message that crashes the consumer?"**
Dead-letter queue pattern: try-catch around processing; on persistent failure, send to a DLQ topic; alert. Don't block the partition forever.

> **"How does Kafka achieve high throughput with replication?"**
Linear-disk writes + zero-copy reads + batching + compression + page-cache friendly. Replication is pull-based; followers fetch in batches.

> **"How would you migrate from RabbitMQ to Kafka?"**
Dual-write phase from RMQ to Kafka. Migrate consumers one at a time. Watch ordering / delivery semantics differences. Eventually deprecate RMQ writes.

> **"How do you size a Kafka cluster?"**
Throughput-per-broker × retention-days × replication-factor → disk. Network ~throughput × replication-factor. CPU ~throughput × compression overhead. Always overprovision (~2x).

> **"How do you delete a single message (e.g., GDPR right-to-erasure)?"**
You can't delete arbitrary messages from a log. Two patterns: (1) log compaction with a tombstone for that key; (2) re-encrypt with a per-user key and "forget" the key. LinkedIn's approach: encrypt PII at the producer with a member-scoped key; on deletion request, delete the key.

> **"How do you handle a hot partition?"**
Re-key with finer granularity (e.g., add an entropy suffix). Increase partition count and rehash. Decompose the topic into multiple topics.

## 15.13 Things LinkedIn engineers love to discuss

- **Tiered storage** — moving cold segments to S3/Azure Blob; Kafka becomes practically infinite-retention.
- **The KRaft migration** — operational benefits, scale benefits, the long tail of tooling that knew ZK.
- **The streaming SQL story** — kSQL / Flink SQL on Kafka.
- **Kafka Streams vs. Samza** — both are stream-processing libraries over Kafka; Streams is from Confluent, Samza from LinkedIn. Tradeoffs: Streams is library-style; Samza is more service-oriented.
- **Exactly-once on multi-region** — hard. The boundary of a transaction is the cluster.
- **Cost** — Kafka at LinkedIn scale is a top-ten infra cost line item.