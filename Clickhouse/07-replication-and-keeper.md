# 7 · Replication and ClickHouse Keeper

Replication = how ClickHouse keeps multiple replicas of a table consistent and how it survives node failures. Keeper = the metadata coordination service that makes it work. Both are mandatory understanding for staff-level.

## 7.1 What gets replicated

- Schema (DDL, via "Replicated database" engine or `ON CLUSTER` queries).
- Data parts (via Keeper-tracked logs + HTTP fetch between replicas).
- Mutations and merges (the merge plan is decided by one replica and assigned to others).
- Async insert metadata.

What's **not** replicated by default:
- User accounts, settings, dictionaries — replicated only if you use `Replicated`-prefixed entities or external storage (Keeper or RBAC backends).
- Per-host system tables.

## 7.2 ReplicatedMergeTree

The replicated MergeTree variants (`ReplicatedMergeTree`, `ReplicatedReplacingMergeTree`, …) use **Keeper** to:
- Coordinate which replica owns the assignment of each merge.
- Track the **replication log** — operations to apply (insert, merge, mutation).
- Hold per-part metadata (which parts exist, their checksums).

### DDL

```sql
CREATE TABLE events ON CLUSTER my_cluster (
    ts        DateTime,
    user_id   UInt64,
    event     LowCardinality(String)
) ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/events',     -- Keeper path
    '{replica}'                              -- replica name
)
ORDER BY (ts, user_id);
```

The macros `{shard}` and `{replica}` are resolved from the server config — every replica must have a unique `{replica}` value, and replicas of the same shard share `{shard}`.

### Insert dedup

Each block inserted into a `ReplicatedMergeTree` is **hashed**, and the hash is stored in Keeper. Subsequent inserts of the same block (hash collision) are rejected as duplicates. This gives at-most-once dedup within a configurable window (`replicated_deduplication_window`, default 100 blocks).

Use case: retrying an insert after a network failure won't double-write. Use case: feeding Kafka with at-least-once semantics — duplicates dedup naturally.

Catch: dedup is by **block content + insert quorum settings**, not by user-row-id. If you re-insert the same row in a *different* batch with other rows, dedup doesn't help.

### Quorum inserts

```sql
INSERT INTO events SETTINGS insert_quorum = 2, insert_quorum_timeout = 5000 VALUES ...;
```

The insert isn't acknowledged until at least N replicas have it. Slower, but guarantees that a subsequent read against any quorum-aware replica will see the data.

Pair with `select_sequential_consistency = 1` on the SELECT side to enforce that the replica you're reading has caught up.

### Merge / mutation coordination

When a replica decides to merge parts, it writes the assignment to Keeper. Other replicas fetch the new merged part once it's done (or perform the merge themselves if `replicated_max_parallel_fetches_for_table` says they should).

A mutation (`ALTER UPDATE`/`DELETE`) is written to Keeper with a monotonic version. Each replica applies it locally; progress is tracked per replica.

## 7.3 ClickHouse Keeper — the Raft replacement for ZooKeeper

ZooKeeper has been the coordination service for ClickHouse since the start. Keeper is the in-house C++ rewrite (announced 2021, GA 2022+) that:

- Implements the same **ZooKeeper client wire protocol** — drop-in replacement.
- Uses **Raft** (via the `NuRaft` library) instead of Zab.
- 40-60× less RAM, similar or faster throughput.
- Can be embedded *inside* a ClickHouse server (for small deployments) or run standalone (recommended for production).

### Typical topology

For production: **3 Keeper nodes** (Raft quorum of 2). Survives 1 node failure.

Larger: **5 Keeper nodes** for tolerance of 2 failures.

### Why not embedded?

Embedding Keeper inside the ClickHouse process couples coordination availability to ClickHouse server health. For production, run standalone. For dev or small clusters (< 5 nodes), embedded is convenient.

### Keeper performance

The Raft path is a serializable write across the quorum. So writes are bounded by Raft's RTT to majority. Reads are linearizable (default) or sequential (cheaper).

- Writes: a few thousand ops/sec is plenty for a typical ClickHouse cluster.
- Latency: single-digit ms for committed write within a DC; cross-region adds RTT.

### What goes into Keeper

- `/clickhouse/tables/{shard}/{table}/log` — replication log entries.
- `/clickhouse/tables/{shard}/{table}/replicas/{replica}` — per-replica state.
- `/clickhouse/tables/{shard}/{table}/blocks` — block-dedup hashes.
- `/clickhouse/task_queue/ddl` — distributed DDL queue.
- (Cloud) SharedMergeTree-specific metadata about parts on object storage.

## 7.4 The Replicated database engine

Newer alternative to `ON CLUSTER` for distributed DDL.

```sql
CREATE DATABASE myapp ENGINE = Replicated('/clickhouse/databases/myapp', 'shard1', 'replica1');
```

DDL inside this database is automatically replicated to other replicas of the same database. Cleaner than scattering `ON CLUSTER` everywhere.

## 7.5 Distributed DDL

`ON CLUSTER my_cluster` runs the DDL on every node in `my_cluster`. The DDL is enqueued in Keeper; each node consumes the queue.

Failure mode: one node is down → DDL stuck waiting for that node. Use `distributed_ddl_task_timeout` to bound the wait. Re-run after the laggard recovers.

## 7.6 Consistency model — what you get

- **Async replication** by default. Writes return as soon as the local replica persists; other replicas catch up over seconds.
- **Eventual** consistency for reads, unless you use `insert_quorum` + `select_sequential_consistency`.
- **Block-level dedup** prevents accidental double-writes on retry.
- **Strict ordering inside one replica** — reads-your-writes if you read from the same replica.

If you need strong consistency:
1. Use quorum inserts.
2. Use `select_sequential_consistency = 1` on read.
3. Accept latency increase.

## 7.7 Recovery and re-sync

When a replica is down for a while and comes back:
- It reads its position in the Keeper log.
- For each missed entry, it fetches the part from another replica.
- If the gap is too large (`max_replicated_logs_to_keep` exceeded), the replica is "lost" and must be re-initialized from scratch (rebuild from another replica). This is a deliberate operational fence — don't let logs grow unbounded.

`SYSTEM RESTORE REPLICA <table>` can fast-track a lost replica's recovery.

## 7.8 Replica role and selection

By default, the client randomly picks a replica per query. Settings:
- `load_balancing` — `random`, `nearest_hostname`, `in_order`, `first_or_random`, `round_robin`.
- `prefer_localhost_replica` — true by default; useful if you proxy through nodes.
- `use_hedged_requests` — issue to a backup replica if the first is slow.

For predictable performance, pick `nearest_hostname` (same DC) or `in_order` with backup.

## 7.9 Common operational patterns

### Add a replica

1. Provision the new node, same config (matching `{replica}` macro).
2. Create the table with the same `ReplicatedMergeTree(...)` definition.
3. The new replica reads the log, fetches all existing parts.
4. Watch `system.replication_queue` and `system.replicated_fetches`.

### Remove a replica

1. Stop the node.
2. `SYSTEM DROP REPLICA 'replica_name' FROM TABLE events;`
3. Cleanup Keeper entries.

### Detect lag

```sql
SELECT replica_name, queue_size, absolute_delay
FROM system.replicas
WHERE table = 'events';
```

`absolute_delay` is the seconds behind the leader. Alert if it grows.

### Failed merges

```sql
SELECT * FROM system.replication_queue WHERE last_exception != '';
```

Common causes: corrupt part, missing column, schema-version skew.

## 7.10 Cloud's twist — SharedMergeTree

In ClickHouse Cloud, replicas don't replicate data via HTTP; they share data on object storage. Keeper still coordinates metadata. See [09](09-cloud-and-sharedmergetree.md).

This eliminates:
- The catch-up cost (replica is "just metadata + object storage" — no re-fetch).
- The replica scale limit (was ~10–20 replicas per table; now hundreds).
- The cost-doubling per replica.

## 7.11 Anti-patterns

- **No replication in production.** A single replica = data loss waiting to happen.
- **Putting Keeper on the same disks as ClickHouse data.** Disk-saturated ClickHouse will starve Keeper and cause cascading outages.
- **Single-node Keeper.** No HA; a Keeper outage breaks all replication and DDL.
- **5+ Keeper nodes.** Diminishing returns past 5; latency grows.
- **Cross-region Keeper without considering RTT.** Every replicated write pays the cross-DC RTT.
- **High `replicated_deduplication_window`** that doesn't actually help.
- **Letting `system.replication_queue` grow unboundedly** without alerting.

## 7.12 Must-internalize

- ReplicatedMergeTree + Keeper is the standard production unit.
- Keeper = Raft-based ZK replacement; run 3 or 5 standalone nodes.
- Async replication by default; quorum + sequential-consistency for strong.
- Block-level insert dedup gives "retry-safe" semantics.
- Use Replicated database engine OR ON CLUSTER for distributed DDL.
- Watch `system.replicas.absolute_delay` and `system.replication_queue`.
- Cloud's SharedMergeTree replaces the replica-data-copy model; metadata still in Keeper.

---

## Sources

- [Data replication — official](https://clickhouse.com/docs/engines/table-engines/mergetree-family/replication)
- [ClickHouse Keeper — official](https://clickhouse.com/docs/guides/sre/keeper/clickhouse-keeper)
- [Keeper — Raft alternative blog](https://clickhouse.com/blog/clickhouse-keeper-a-zookeeper-alternative-written-in-cpp)
- [Replicated database engine](https://clickhouse.com/docs/engines/database-engines/replicated)
- [InfoQ — Keeper Raft alternative](https://www.infoq.com/news/2023/12/clickhouse-keeper-raft/)
