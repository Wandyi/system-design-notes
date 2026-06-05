# 18 · Venice, Databus, and Brooklin — Derived Data and CDC at LinkedIn

Three systems that together solve one big problem:

> How do you take data computed *somewhere* (offline batch, stream processing, an OLTP store) and serve it *online* at low latency, kept fresh, replicated across regions, and observable?

- **Venice** — the modern derived-data serving platform. Open-sourced 2022.
- **Databus** — older change-data-capture (CDC) system for Oracle, replaced by Brooklin.
- **Brooklin** — current CDC + general-streaming-data-pipe service. Open-sourced.

## Part A — Venice

## 18.1 What is Venice?

> Venice is a distributed, derived-data serving platform: write-heavy ingest from Kafka/Spark, read-heavy serving at low latency, with bulk + incremental update modes.

LinkedIn open-sourced it in 2022. It's the successor to Voldemort RO (for batch-pushed data) and Voldemort RW (for stream-pushed data), unified under one system.

## 18.2 Why Venice

Derived data is a huge category at LinkedIn:
- ML features for ranking (member features, content features).
- Pre-aggregated rollups for fast reads.
- Denormalized views of the Economic Graph.
- Recommendation candidate lists.

Two main update modes:
- **Batch**: a Spark job recomputes the whole dataset (or partition) and bulk-loads.
- **Stream**: incremental updates from a Kafka stream (real-time deltas).
- **Hybrid**: bulk + stream — start with a daily bulk snapshot, then apply streaming deltas.

Voldemort RO handled the bulk case but not stream. Voldemort RW handled stream but had ops issues at scale. Venice unifies.

## 18.3 Architecture

```
   Source: Spark / Hadoop bulk      Stream: Samza / Flink / direct Kafka
        │                                       │
        ▼                                       ▼
   ┌────────────────┐                  ┌────────────────┐
   │ Venice Push     │                 │  Kafka          │ (write topic)
   │ Job (controller │                 └────────┬────────┘
   │  orchestrated)  │                          │
   └────────┬────────┘                          │
            │ bulk → Kafka                       │
            ▼                                    ▼
   ┌──────────────────────────────────────────────────┐
   │           Venice Server cluster                  │
   │   - consumes Kafka                              │
   │   - writes to per-store RocksDB                 │
   │   - serves reads (HTTP / RPC)                   │
   └────────────────────┬─────────────────────────────┘
                        │
                        ▼
                   ┌──────────────┐
                   │  Client      │
                   │  (router /   │
                   │   thin)      │
                   └──────────────┘
```

Components:

- **Venice Controller** — manages stores, versions, partitions; uses Helix.
- **Venice Server** — holds partition replicas, consumes Kafka, serves reads.
- **Venice Router** — clients send requests to a router that fans out to the right server.
- **Push Job** — for batch loads, orchestrates the build of a new "version" and switches reads to it.

## 18.4 Store, version, partition

- **Store** — analogous to a table. Has a schema, partitioning strategy, replication factor.
- **Version** — each batch push creates a new version. Active version is what serves reads. Older versions retained for rollback / staging.
- **Partition** — slice of a store. Each partition is a Kafka topic-partition and a RocksDB store on each replica.

When a batch push completes:
- All replicas finish ingesting.
- Controller flips the active version atomically (for that store).
- Old version remains for rollback window, then GC.

## 18.5 Hybrid mode

- A store can have both batch and stream pushes.
- Batch establishes a baseline; stream applies deltas.
- Newer stream offset must be tracked per replica.
- Total ordering of (key, version) is maintained per partition.

## 18.6 Read path

1. Client sends GET key or batch-GET keys to a Router.
2. Router computes target partition (hash of key); knows replicas hosting it.
3. Router picks one (load-balanced); forwards.
4. Server reads from local RocksDB; returns value.
5. Optional: client-side caching for hottest keys.

Latency target: p99 < 10ms for single-key, < 50ms for batch.

## 18.7 Replication and durability

- Each partition has N replicas (typical 3).
- All replicas consume the same Kafka topic-partition independently.
- No leader-follower for reads — all replicas eligible.
- Eventually consistent on writes (Kafka offsets converge).
- For multi-DC: per-region Venice cluster + Kafka mirror.

## 18.8 Use cases at LinkedIn

- **Feature store online layer** (ML inference features).
- **Feed FollowFeed timelines** (in some configurations).
- **Recommendation candidate lists** (JYMBII candidates).
- **Search reranking signals** (precomputed scores).
- **Pre-aggregated counters / rollups**.

## 18.9 Venice vs. Voldemort RO/RW

| Property | Voldemort RO | Voldemort RW | Venice |
|---|---|---|---|
| Bulk load | Yes (Hadoop) | No | Yes |
| Stream update | No | Yes | Yes |
| Hybrid | No | No | Yes |
| Cross-DC | Manual | Built-in | Built-in (Kafka mirror) |
| Atomic version swap | Yes | N/A | Yes |
| Storage engine | mmap chunk files | BDB / RocksDB | RocksDB |
| Active development | No (frozen) | Maintenance | Yes |

## 18.10 Common interview questions on Venice

> **"How would you push a 1TB derived dataset to Venice nightly?"**
Spark job partitions by Venice partition key, writes per-partition data to Kafka (via Venice's push semantics). Servers consume; when all partitions caught up, Controller flips version.

> **"How do you handle a bad batch push that produced corrupt data?"**
Rollback: Controller flips active version back to previous. Investigate the source job; fix; re-push.

> **"What's the trade-off of having all replicas consume Kafka independently vs. leader-follower?"**
Independent: simpler, no leader election needed, replicas naturally diverge slightly. Leader-follower: more consistency but more coordination cost. Venice picked independent for simplicity at scale.

> **"How does Venice support large values (1MB+)?"**
Chunked storage: large values split into chunks; manifest record points to chunks. Read path reassembles.

> **"How would you do a 'find all keys for member X' query?"**
Not Venice's strength. Either partition by member_id (all keys for X go to one partition; do a prefix scan) or maintain a separate index. Venice is point-lookup-optimized.

---

## Part B — Databus

## 18.11 What is Databus?

Open-sourced 2013. LinkedIn's original change-data-capture system, originally for Oracle.

Mental model: tail the source database's transaction log → emit changes as a stream → consumers subscribe.

Architecture:
- **Relay**: tails the source binlog; serves the change stream to consumers.
- **Bootstrap server**: serves the historical state to new consumers (so a new consumer can build up state from scratch).
- **Consumer**: clients that subscribe.

Mostly displaced by Brooklin today.

## 18.12 Why CDC matters

Imagine Espresso is the primary store for member profiles. Many downstream systems need to react to profile changes:
- Search indexer.
- FollowFeed (so a profile picture change updates downstream caches).
- Notifications (so connections see "X changed jobs").
- Analytics.

Three approaches:
1. Each downstream polls Espresso periodically — bad, doesn't scale.
2. Application code dual-writes to a queue and to Espresso — risk of inconsistency.
3. CDC — Espresso writes are tailed; changes flow into a Kafka stream that any consumer can subscribe to. **This is the canonical answer.**

CDC is the foundation of LinkedIn's "events are first-class" architecture.

---

## Part C — Brooklin

## 18.13 What is Brooklin?

Open-sourced 2018. A general-purpose distributed streaming-data-pipe service.

Architecture:
- A pluggable connector model: source connector reads from somewhere (Espresso CDC, Kafka, MySQL binlog), destination connector writes somewhere (Kafka, Espresso, BigQuery, etc.).
- A central control plane manages connector lifecycle.
- Built on Apache Helix for cluster management.

Brooklin replaced Databus and Kafka MirrorMaker for many use cases at LinkedIn.

### Key use cases

1. **Espresso CDC** — tail Espresso's MySQL binlog; emit change events to Kafka.
2. **Kafka mirroring** — replicate topics across DCs / clouds.
3. **Database bridge** — sync between heterogeneous stores.
4. **Sourcing for derived stores** — feed Venice or search indexers from primary stores.

### Why Brooklin won

- Same pattern unified across Kafka↔Kafka, DB↔Kafka, etc.
- Helix-managed → ops uniformity.
- Connectors composable.
- Scales horizontally.

## 18.14 Common interview questions

> **"Why not just have services dual-write to Espresso and Kafka?"**
Dual-write has a race: write to Espresso succeeds, write to Kafka fails. State diverges. CDC ensures Kafka faithfully reflects Espresso.

> **"What's the latency of CDC?"**
Espresso → Brooklin → Kafka is typically sub-second, often <100ms. Cross-DC adds latency.

> **"What if Brooklin lags?"**
Downstream consumers see stale data. Buffer at Brooklin scales with throughput. Eventually catches up. Monitor lag; alert if persistent.

> **"How does CDC handle schema changes?"**
Source schema versioned. Connector emits events with schema_id. Schema registry resolves. Consumers tolerate or upgrade.

> **"How would you bootstrap a new consumer that wants to consume from the beginning?"**
Two approaches: (1) replay from Kafka if retention covers the period; (2) snapshot the source state (via Bootstrap-like service or a direct dump) + apply incremental changes from a known point. LinkedIn uses both.

> **"How do you ensure no event is lost during a Brooklin failover?"**
Each connector tracks its source position (binlog position, Kafka offset). Position persisted in ZooKeeper / Kafka. On failover, new connector resumes from the persisted position. At-least-once delivery; downstream dedupes.

> **"What's the trade-off between using CDC vs. application-emitted events?"**
CDC: faithful to the store, no app changes, captures all writes. App-emitted: richer event shape (can include domain context), but susceptible to dual-write inconsistency. Best practice: CDC for state, app-emitted events for domain actions; both feed Kafka.