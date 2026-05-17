# 16 · System Tables and Observability

ClickHouse exposes its internal state through `system.*` tables. Mastering them is what separates a power user from a staff engineer.

## 16.1 The "show me what's happening" tables

### system.processes — running queries right now

```sql
SELECT query_id, user, elapsed, memory_usage, query
FROM system.processes
ORDER BY elapsed DESC;
```

Kill a runaway: `KILL QUERY WHERE query_id = '...'`.

### system.query_log — every completed query (sampled or full)

```sql
SELECT
    type, query_id, user, query_duration_ms,
    read_rows, read_bytes, memory_usage, exception
FROM system.query_log
WHERE event_date = today() AND type != 'QueryStart'
ORDER BY event_time DESC LIMIT 50;
```

Stores both `QueryStart` and `QueryFinish` (and `ExceptionWhileProcessing`). Use `QueryFinish` for completed queries' final stats.

Settings: enable with `log_queries = 1`; sample with `log_queries_probability` if volume is too high.

### system.query_thread_log — per-thread breakdown

For deep-dive: which thread spent time on what.

### system.text_log — application log lines

ClickHouse's own logs as a table. Useful for SQL-style log analysis.

## 16.2 Storage and parts

### system.parts — active and inactive parts

```sql
SELECT
    database, table, partition, name,
    rows, formatReadableSize(bytes_on_disk) AS size,
    active, level, marks
FROM system.parts
WHERE database = 'default' AND table = 'events'
ORDER BY modification_time DESC LIMIT 20;
```

`active = 1` means currently part of the table; `active = 0` means superseded by a merge (will be cleaned up by `old_parts_lifetime` (default 8 min)).

Aggregate to see fragmentation:

```sql
SELECT
    database, table,
    count() AS total_parts,
    sum(active) AS active_parts,
    sum(if(active, rows, 0)) AS active_rows
FROM system.parts
GROUP BY database, table
ORDER BY total_parts DESC;
```

If `active_parts` > 100 per table you should investigate (too many small parts).

### system.parts_columns — per-column storage stats

```sql
SELECT column, formatReadableSize(sum(data_compressed_bytes))   AS compressed,
              formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed,
              sum(data_uncompressed_bytes)/sum(data_compressed_bytes) AS ratio
FROM system.parts_columns
WHERE active AND database='default' AND table='events'
GROUP BY column ORDER BY ratio DESC;
```

Finds columns with bad compression.

### system.detached_parts — quarantined parts

Parts that failed checksum or were detached. Investigate before re-attaching or dropping.

## 16.3 Merges and mutations

### system.merges — in-flight merges

```sql
SELECT
    database, table, elapsed, progress, num_parts,
    source_part_names, result_part_name, total_size_bytes_compressed
FROM system.merges;
```

Long-running merges indicate big parts; a stuck merge means a problem.

### system.mutations — async DDL mutations

```sql
SELECT
    database, table, mutation_id, create_time,
    parts_to_do, is_done, latest_fail_reason
FROM system.mutations
WHERE NOT is_done
ORDER BY create_time;
```

Stuck mutations (`is_done=0` for a long time) block follow-on mutations.

## 16.4 Replication

### system.replicas — per-table replica state

```sql
SELECT
    database, table, replica_name,
    is_leader, absolute_delay, queue_size, log_pointer,
    is_session_expired, last_queue_update
FROM system.replicas;
```

- `absolute_delay` (seconds) is the alert metric — replica behind leader.
- `queue_size` growing = stuck or slow replication; check `replication_queue`.
- `is_session_expired = 1` = lost Keeper session; restart needed.

### system.replication_queue — pending work per replica

```sql
SELECT type, num_tries, last_exception, last_attempt_time
FROM system.replication_queue
WHERE last_exception != '';
```

Look at failures; common causes: missing part, schema skew.

### system.replicated_fetches — in-flight cross-replica fetches

```sql
SELECT * FROM system.replicated_fetches;
```

Useful when a new replica is catching up or when a replica is fetching missed parts.

## 16.5 Keeper

### system.zookeeper — read Keeper / ZK directly

```sql
SELECT name, mtime FROM system.zookeeper WHERE path = '/clickhouse/tables';
```

Useful for debugging cluster metadata.

### system.zookeeper_connection — current Keeper connection

```sql
SELECT * FROM system.zookeeper_connection;
```

## 16.6 Distributed-related

### system.clusters — current cluster topology

```sql
SELECT cluster, shard_num, replica_num, host_name, errors_count
FROM system.clusters;
```

`errors_count > 0` means recent communication errors.

### system.distribution_queue — async insert queue at Distributed table

Distributed inserts buffer locally; this table shows the queue.

## 16.7 Metrics and counters

### system.metrics — current point-in-time gauges

```sql
SELECT metric, value FROM system.metrics WHERE metric LIKE '%Merge%';
```

Examples: `BackgroundMergesAndMutationsPoolTask`, `MemoryTracking`, `Read`, `Write`.

### system.events — monotonically increasing counters

```sql
SELECT event, value FROM system.events;
```

Rate-of-events (compute diffs over time) is what observability stacks track.

### system.asynchronous_metrics — periodically computed metrics

```sql
SELECT metric, value FROM system.asynchronous_metrics WHERE metric LIKE '%Disk%';
```

Disk usage, network, memory, etc.

These three (`metrics`, `events`, `asynchronous_metrics`) are what Prometheus's clickhouse-exporter scrapes.

## 16.8 Schema and metadata

```sql
SELECT * FROM system.tables WHERE database = 'default';
SELECT * FROM system.columns WHERE database = 'default' AND table = 'events';
SELECT * FROM system.functions WHERE name LIKE 'window%';
SELECT * FROM system.data_skipping_indices WHERE table='events';
SELECT * FROM system.projections WHERE table='events';
```

## 16.9 RBAC and policies

```sql
SELECT * FROM system.users;
SELECT * FROM system.roles;
SELECT * FROM system.role_grants;
SELECT * FROM system.row_policies;
SELECT * FROM system.quotas;
SELECT * FROM system.grants;
```

Critical for multi-tenant audits.

## 16.10 Settings

```sql
SELECT name, value, changed, description
FROM system.settings
WHERE changed
ORDER BY name;
```

`SELECT * FROM system.merge_tree_settings WHERE changed;` for table-level.

## 16.11 The "is the server happy?" dashboard query set

```sql
-- 1. Background pool saturation
SELECT metric, value FROM system.metrics
WHERE metric IN (
    'BackgroundMergesAndMutationsPoolTask',
    'BackgroundFetchesPoolTask',
    'BackgroundCommonPoolTask'
);

-- 2. Memory tracking
SELECT formatReadableSize(value) AS used FROM system.metrics WHERE metric = 'MemoryTracking';

-- 3. Replication health
SELECT database, table, max(absolute_delay) AS max_delay, max(queue_size) AS max_queue
FROM system.replicas
GROUP BY database, table
HAVING max_delay > 60 OR max_queue > 100;

-- 4. Slow queries
SELECT user, query_duration_ms, memory_usage, query
FROM system.query_log
WHERE type = 'QueryFinish' AND event_date = today() AND query_duration_ms > 10000
ORDER BY query_duration_ms DESC LIMIT 20;

-- 5. Stuck mutations
SELECT database, table, mutation_id, latest_fail_reason
FROM system.mutations WHERE NOT is_done AND latest_fail_reason != '';

-- 6. Too-many-parts risk
SELECT database, table, count() AS parts, sum(active) AS active
FROM system.parts
GROUP BY database, table
HAVING active > 100
ORDER BY active DESC;
```

Embed these in Grafana or pop a `watch -n 10 'clickhouse-client -q "..."'` for live triage.

## 16.12 Common alerts (what an SRE wires up)

| Alert | Threshold | Why |
|-------|-----------|-----|
| `absolute_delay` per replica | > 60s | Replica lagging |
| Replicated queue size | > 1000 | Stuck replication |
| Active parts per (table, partition) | > 300 | "Too many parts" looming |
| Mutation `not is_done` age | > 1 hour | Stuck mutation |
| Keeper unavailable | any | DDL + replication broken |
| `MemoryTracking` vs configured cap | > 90% | OOM risk |
| Background merge pool full | > 80% | Insert throttling soon |
| Disk free | < 15% | TTL or part-rewrite hazard |
| Slow query rate | > N/s | Capacity / regression |

## 16.13 Trace_id propagation

ClickHouse supports OpenTelemetry: set `traceparent` header on queries (HTTP) or via TCP client extension. Server logs and `query_log` carry `trace_id` / `span_id` so you can correlate with upstream services.

## 16.14 Must-internalize

- `system.query_log` is the single most valuable forensic table.
- `system.parts` / `system.merges` / `system.mutations` for storage health.
- `system.replicas` / `system.replication_queue` for replication health.
- `system.metrics` / `events` / `asynchronous_metrics` for SRE metrics.
- Wire 10 alerts before going to prod; don't be surprised.

---

## Sources

- [System tables — official](https://clickhouse.com/docs/operations/system-tables)
- [Monitoring — official](https://clickhouse.com/docs/operations/monitoring)
- [PostHog handbook — operations](https://posthog.com/handbook/engineering/clickhouse/operations)
