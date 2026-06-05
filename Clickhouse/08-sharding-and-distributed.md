# 8 · Sharding and the Distributed Engine

ClickHouse scales horizontally by **sharding** — partitioning a table's rows across nodes. The `Distributed` engine is a virtual front-end that hides shard topology from the client.

## 8.1 The two-table pattern

For every sharded table you create *two* tables:

```sql
-- local table on every shard
CREATE TABLE events_local ON CLUSTER my_cluster (
    ts        DateTime,
    user_id   UInt64,
    event     LowCardinality(String)
) ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/events',
    '{replica}'
)
ORDER BY (ts, user_id);

-- distributed table on every node
CREATE TABLE events ON CLUSTER my_cluster
AS events_local
ENGINE = Distributed(
    my_cluster,                  -- cluster name from config
    default,                     -- database
    events_local,                -- per-shard table
    cityHash64(user_id)          -- sharding key (optional)
);
```

Clients write to and read from `events` (the Distributed). It handles routing transparently.

## 8.2 Cluster config (the source of truth)

```xml
<remote_servers>
    <my_cluster>
        <shard>
            <internal_replication>true</internal_replication>
            <replica><host>node-1a</host><port>9000</port></replica>
            <replica><host>node-1b</host><port>9000</port></replica>
        </shard>
        <shard>
            <internal_replication>true</internal_replication>
            <replica><host>node-2a</host><port>9000</port></replica>
            <replica><host>node-2b</host><port>9000</port></replica>
        </shard>
    </my_cluster>
</remote_servers>
```

`internal_replication=true` means the Distributed engine will write to **one** replica per shard and let `ReplicatedMergeTree` handle the rest. With `false`, 
the Distributed writes to *every* replica directly — bypasses ReplicatedMergeTree's coordination. Default modern setup is `true`.

## 8.3 Sharding key — pick wisely

The `Distributed` engine takes an optional sharding-key expression. On `INSERT`, it routes rows to shards by `hash(key) % shardCount` (with weights).

### Common choices

| Sharding key | When |
|--------------|------|
| `rand()` | No data locality needed; uniform spread. Bad if you'll later filter by entity. |
| `cityHash64(user_id)` | Co-locate user's rows. Good for per-user queries. |
| `cityHash64(tenant_id)` | Multi-tenant — entire tenant lives on one shard. Big wins if queries are tenant-scoped. |
| `intHash64(account_id)` | Same idea, different hash. |
| Date-based | Don't. You'll get hot-shard-of-the-day. |

### Hot-shard rule

If your sharding key clusters a hot entity onto one shard, you get a hot shard. For a multi-tenant SaaS, this is fine if tenants are roughly uniform; very bad if you have a couple of whales.

**Mitigation**: split big tenants into virtual sub-tenants, or use composite keys (`(tenant_id, user_id)`).

### Insert path

- **Client-side sharding** — client computes the shard and _writes directly to one shard's local table_. Best performance; no Distributed-engine fan-out cost. Used in production by mature teams.
- **Server-side sharding via Distributed** — convenient; the Distributed table accepts the row, hashes the sharding key, and routes. Cost: an extra hop per row.

For high-throughput streaming ingest (Kafka, OTel), client-side is materially faster.

### Bypass / no sharding key

If `Distributed(..., events_local)` (no key), inserts go to a single shard chosen round-robin per query. Useful for ingest where you'll re-shard later.

## 8.4 Read path

```sql
SELECT count() FROM events WHERE ts > now() - INTERVAL 1 HOUR;
```

The receiving node:
1. Sends the query (or a rewritten subquery) to **one replica per shard**.
2. Receives partial results.
3. Combines them (final aggregation).

You can see the shard fan-out in `system.query_log` and `EXPLAIN PIPELINE`.

### GLOBAL IN / GLOBAL JOIN — solving subquery distribution

A subquery in a `Distributed`-side query is **evaluated on each shard locally**, by default. That's wrong if the subquery should be the same across shards.

```sql
-- WRONG: each shard runs its own user-set
SELECT count() FROM events WHERE user_id IN (
    SELECT user_id FROM premium_users
);

-- RIGHT: subquery runs once on initiator, results broadcast to shards
SELECT count() FROM events WHERE user_id GLOBAL IN (
    SELECT user_id FROM premium_users
);
```

Same with `GLOBAL JOIN`. Use it whenever the right side should be coherent across shards.

### Distributed JOIN strategies

- **`distributed_product_mode = 'allow'`** — naive, joins are local to each shard. Right if you sharded both tables by the join key.
- **`distributed_product_mode = 'global'`** — auto-promote to GLOBAL JOIN.
- **`distributed_product_mode = 'local'`** — assume tables are co-sharded.
- **`distributed_product_mode = 'deny'`** (default) — refuse if it'd produce wrong results; force the developer to think.

Best: shard both joined tables by the same key (co-located joins); failing that, broadcast the small side via GLOBAL.

## 8.5 Parallel replicas — single query across replicas

A modern feature: a single query can be parallelized across replicas of one shard, not just across shards.

```sql
SET allow_experimental_parallel_reading_from_replicas = 1;
SET max_parallel_replicas = 4;
```

The initiator divides the part-list among replicas; each scans its share; partial results combine on initiator. Useful for analytics queries on a small cluster.

In Cloud / SharedMergeTree, this is more natural because every replica has access to the same data on S3.

## 8.6 Adding shards

Painful with classic ClickHouse. Steps:
1. Add nodes to the cluster config.
2. Re-create the table on the new shard.
3. The new shard receives all *new* writes (if using `rand()`) or its hash bucket (if using a hash key).
4. **Old data does not redistribute automatically.**
5. To rebalance: re-ingest, use `INSERT SELECT` from old shards into the Distributed table.

This is why you over-provision the shard count at the start. Some teams pre-shard into 100s of "virtual shards" mapped onto fewer physical nodes; growing means re-mapping virtual-to-physical, not changing the shard key.

In Cloud (SharedMergeTree on S3), the painful re-shard problem largely goes away because all replicas share data; "adding capacity" is adding more compute nodes, not more shards.

## 8.7 Cluster topology patterns

| Pattern | Description | When |
|---------|-------------|------|
| 1 shard × N replicas | Pure HA; no horizontal partitioning | Small workloads, < tens of TB |
| N shards × M replicas | Standard pattern: N partitions, each replicated M ways | Most large on-prem |
| Cross-region cluster | Replicas in two regions; async replication across | DR requirement; high latency |
| Two clusters with one-way replication | Active in region A; passive copy in B | Cleaner DR than cross-region cluster |
| Cloud "fanout replicas" via SharedMergeTree | 1 shard × 100s replicas | Sub-second user-facing read at very high QPS |

## 8.8 Co-location and dictionary distribution

Dictionaries live per node. If you use `dictGet` in a sharded query, each node uses its own copy. Keep them loaded everywhere via Replicated dictionary engine or by sourcing from a replicated table.

## 8.9 Operational diagnostics

```sql
-- shard status
SELECT * FROM system.clusters WHERE cluster = 'my_cluster';

-- per-shard part counts
SELECT host_name, count() FROM clusterAllReplicas('my_cluster', system.parts)
WHERE table = 'events' AND active
GROUP BY host_name
ORDER BY host_name;

-- query fan-out
SELECT query_id, hostname() AS node, query_duration_ms
FROM clusterAllReplicas('my_cluster', system.query_log)
WHERE query_id = '...';
```

## 8.10 Anti-patterns

- **Sharding by date** → hot shard of the day.
- **Joining two distributed tables without co-location** → cross-shard fan-out blow-up.
- **No `internal_replication`** → ReplicatedMergeTree and Distributed both write; duplicate work.
- **Tiny shards** → 50 shards of 2 GB each; better as 5 shards of 20 GB.
- **No client-side sharding for high ingest** → Distributed-side hot spot.
- **Re-sharding ad-hoc** → multi-week project for a non-trivial cluster. Plan capacity early or move to Cloud.

## 8.11 Must-internalize

- Sharding pattern: `Distributed` engine over per-shard `ReplicatedMergeTree`.
- Sharding key: hash of an entity ID for locality; avoid date/time.
- `internal_replication=true` is the modern setup.
- Use `GLOBAL IN`/`GLOBAL JOIN` for subqueries that must be coherent across shards.
- Co-shard joined tables on the join key for local joins.
- Parallel replicas distribute one query across replicas of a shard.
- Adding shards is painful; over-provision or use Cloud.

---

## Sources

- [Distributed engine — official](https://clickhouse.com/docs/engines/table-engines/special/distributed)
- [Sharding key — official docs](https://clickhouse.com/docs/engines/table-engines/special/distributed#distributed-clusters)
- [Parallel replicas](https://clickhouse.com/docs/operations/settings/settings#max_parallel_replicas)
- [GLOBAL IN / JOIN](https://clickhouse.com/docs/sql-reference/operators/in#distributed-subqueries)
- [Cluster operations](https://clickhouse.com/docs/operations/system-tables/clusters)
