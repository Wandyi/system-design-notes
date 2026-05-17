# 15 · Anti-Patterns — Avoidable Disasters

Things that will get you nodded out of an interview, or worse, paged at 3 AM.

## 15.1 Trickle inserts (the "too many parts" classic)

```sql
INSERT INTO events VALUES (1, ...);  -- one row at a time, per request
```

Every insert creates a new part. The merger can't keep up. Past `parts_to_delay_insert` (default 150), inserts slow down; past `parts_to_throw_insert` (default 300), they're rejected.

**Fixes**:
- Batch on the client (1K–100K rows/insert).
- `async_insert = 1` server-side batching (default in Cloud).
- Buffer engine in front (legacy).

**Telltale**: `system.parts` shows hundreds of small (< 1 MB) parts per table; merge thread pool is busy; insert latency climbs.

## 15.2 OPTIMIZE TABLE FINAL as a panic button

```sql
OPTIMIZE TABLE events FINAL;
```

Forces merge of all parts into one. On a 1 TB table this runs for hours, monopolizes CPU/IO, blocks new merges. New inserts may get rejected. Replicas may lag.

**Fixes**:
- For dedup behavior: use `argMax` or `LIMIT 1 BY` in queries, not OPTIMIZE.
- For "fix many parts": pause inserts to let the natural merger catch up.
- For one-time cleanup after re-ingest: OPTIMIZE partition-by-partition with `PARTITION` qualifier.

## 15.3 ORDER BY ts first

```sql
ORDER BY (ts, tenant_id, ...)
```

Means tenant-scoped queries can't prune granules by tenant — they have to scan ranges by time first, then filter tenant within them.

**Fix**: lead with the high-selectivity dimension (`tenant_id` then `ts`).

Exception: if your *only* query pattern is range scans over recent time and tenant filter is rare, time-first is right.

## 15.4 High-cardinality ORDER BY

```sql
ORDER BY (user_id, ts)  -- user_id is a UInt64 random ID
```

The primary index has one entry per granule; pruning by `user_id = X` reads all the granules from a single user. With UInt64 random user IDs, that's still all parts.

**Fix**: lead with low-cardinality, high-selectivity columns. Or add a `bloom_filter` skip index on user_id. Or use a projection ORDER BY (user_id, ts).

## 15.5 SELECT * on wide table

Defeats columnar storage. Reads every column file.

**Fix**: always project explicitly. If you really need all columns, ask whether you actually need a row store instead.

## 15.6 Nullable everywhere

A migration from PostgreSQL often makes every column Nullable. Each Nullable column has an extra "is_null" mask. Some optimizations (Delta, LowCardinality combinations) become slower.

**Fix**: use sentinels (0, '', `1970-01-01`) instead of NULL where semantic meaning is preserved.

## 15.7 Map / JSON when typed columns would do

`Map(String, String)` and `JSON` cost parse / lookup overhead per row. If you know the schema, typed columns are 5–10× faster and compress better.

**Fix**: declare typed columns. Use `Map`/`JSON` only for truly open-ended fields.

## 15.8 Joining two big Distributed tables without co-location

```sql
SELECT *
FROM events_dist e
JOIN users_dist u ON e.user_id = u.id;  -- no GLOBAL, not co-sharded
```

Each shard runs the join locally with its slice of the right table. Wrong results. Or, with GLOBAL, the whole `users_dist` is broadcast to every shard → memory blow-up.

**Fix**: co-shard both tables by `user_id`. Then plain JOIN is local and correct. Or replace `users_dist` with a dictionary.

## 15.9 Wide JSON ingest into typed columns later

Inserting `JSON('{"k1":...,"k2":...,...}')` into a `JSON` column and *then* migrating to typed columns. The migration is a full part rewrite (an `ALTER MODIFY`).

**Fix**: design the schema upfront. Use `JSON(k1 UInt32, k2 String)` to hint known keys at table creation.

## 15.10 Mutations on a hot table

Running an `ALTER UPDATE` that touches every part of a 10 TB table during peak ingest.

**Fix**:
- Schedule heavy mutations during low-traffic windows.
- Prefer lightweight DELETE for occasional deletes.
- Use TTL + DROP PARTITION for bulk.
- Engine-level alternatives (ReplacingMergeTree etc.) for ongoing dedup.

## 15.11 Too many partitions

`PARTITION BY toDate(ts)` for a 5-year-retention table = ~1800 partitions per shard. Each partition is independent metadata. CPU at startup, merge planning, and DDL all suffer.

**Fix**: `PARTITION BY toYYYYMM(ts)` is enough for most. Hourly partitions are an anti-pattern unless retention is short.

## 15.12 No replication in production

Single replica = data loss waiting to happen. Even on Cloud (where S3 holds the bytes), single-compute means downtime during pod restarts.

**Fix**: 2+ replicas. Keeper 3+ nodes.

## 15.13 Single-node Keeper

Keeper-as-SPOF defeats all replication's reliability. Production deploys 3 Keepers (or 5).

## 15.14 Forgetting `internal_replication = true`

If `false`, the Distributed engine writes to every replica directly *and* ReplicatedMergeTree also replicates — doubles the work, causes write amplification.

**Fix**: always `internal_replication = true` for Replicated tables.

## 15.15 `count(DISTINCT user_id)` on big data

Exact uniques over high cardinality blow memory.

**Fix**: use `uniq` / `uniqCombined`.

## 15.16 Joining without realizing the small side is on the right

```sql
SELECT * FROM users JOIN events ON users.id = events.user_id;
```

`events` is the right side; it gets built into a hash table. OOM.

**Fix**: put the smaller table on the right.

## 15.17 Indexing every column "just in case"

Each skip index has CPU cost per insert + storage cost. Adding 10 skip indexes on a table can slow ingest 5×.

**Fix**: profile real queries; add indexes that *measurably* help.

## 15.18 Storing timestamps as String

```sql
ts String  -- "2026-05-17T12:34:56Z"
```

- 20+ bytes vs. 4 (DateTime) or 8 (DateTime64).
- Breaks Delta compression.
- Breaks all date functions.

**Fix**: `DateTime` or `DateTime64`.

## 15.19 IP addresses as String

`String` for "1.2.3.4" = 7+ bytes per row vs. 4 with IPv4. Plus loses network-aware functions.

**Fix**: `IPv4` / `IPv6` types.

## 15.20 Doing window/sort over too many rows

```sql
SELECT row_number() OVER (ORDER BY ts) FROM events;
```

Sort over a 10B-row table. Memory and CPU blow-up.

**Fix**: partition the window (`PARTITION BY user_id`), or limit the rows first.

## 15.21 Heavy MV that doesn't filter

```sql
CREATE MATERIALIZED VIEW big_mv TO ... AS SELECT ... FROM events;  -- no WHERE
```

Every insert runs the MV's SELECT against the whole new block. If your MV doesn't reduce row count or do heavy work, you're paying for nothing.

**Fix**: MV should aggregate or filter. Otherwise you've just duplicated the table.

## 15.22 Many MVs on one source table

10 MVs on `events` = every insert runs 10 SELECTs. Insert latency multiplies.

**Fix**: cascade MVs (raw → 1m → 5m → 1h). Or consolidate where possible.

## 15.23 Mutations that don't filter

```sql
ALTER TABLE events UPDATE event = lower(event) WHERE 1;
```

Rewrites every part. Don't.

**Fix**: scope to a partition or to a small set of rows.

## 15.24 Cross-region replicas without thinking about latency

Replicas in two regions = every replicated insert pays the cross-region RTT. Insert QPS drops dramatically.

**Fix**: keep replicas in one region; use async cross-region replication or backups for DR.

## 15.25 Ignoring system.parts and system.merges

```sql
SELECT count(), sum(active) FROM system.parts WHERE table = 'events';
```

If `count()` is many times `sum(active)`, you have a backlog. If `parts_to_delay_insert` triggers, ingest will slow.

**Fix**: monitor; alert on part counts > 100 per (table, partition).

## 15.26 Letting `system.mutations` accumulate stuck mutations

Stuck mutations block follow-on mutations on the same table.

**Fix**: `SELECT * FROM system.mutations WHERE not is_done AND latest_fail_reason != '';` — kill or fix.

## 15.27 Treating ClickHouse as OLTP

Per-row reads/writes, multi-statement transactions, single-row updates at high QPS. CH will do these but it'll be slow.

**Fix**: use Postgres / RocksDB / DynamoDB for OLTP. Use CH for analytical queries.

## 15.28 Embedding Keeper in production

Keeper inside the ClickHouse process couples coordination availability to ClickHouse process health. Server restart = brief Keeper outage = broken replication and DDL.

**Fix**: standalone Keeper cluster (3 nodes).

## 15.29 Not using `ON CLUSTER` for DDL

Running `CREATE TABLE` on one node forgets the others. The Distributed table won't find the table on other shards.

**Fix**: `ON CLUSTER` or use a `Replicated` database.

## 15.30 `INSERT INTO` to a Distributed table at very high QPS

Per-row routing through the Distributed engine adds latency.

**Fix**: client-side sharding (compute the shard, write directly to the local table).

## 15.31 Must-internalize

- Batch inserts. Always.
- Don't OPTIMIZE FINAL in production.
- ORDER BY = your filter, not just time.
- Avoid Nullable, String-for-timestamp, String-for-IP, Map-when-typed-would-do.
- One wide flat table > many narrow ones with joins.
- Co-shard joined tables; otherwise GLOBAL JOIN; otherwise dictionary; otherwise denormalize at ingest.
- Replication mandatory; standalone Keeper mandatory; `internal_replication=true`.
- Monitor system.parts, system.merges, system.mutations, system.replicas continuously.
- CH is analytical, not OLTP. Use it for what it's good at.
