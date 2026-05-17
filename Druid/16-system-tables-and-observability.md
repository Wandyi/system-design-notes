# 16 · System Tables and Observability

Druid exposes internal state via the **`sys` schema** (queryable like any datasource) and via **JMX metrics** emitted to your monitoring system. A staff engineer should be fluent in both.

## 16.1 The `sys` schema

Accessible via Druid SQL. Tables include:

### sys.segments — all segments known to the cluster

```sql
SELECT datasource, count(*) AS segments, sum(size) / 1024 / 1024 AS mb
FROM sys.segments
WHERE is_published = 1 AND is_available = 1
GROUP BY 1 ORDER BY 3 DESC;
```

Columns include: `datasource`, `interval`, `version`, `size`, `num_rows`, `is_published`, `is_available`, `is_realtime`, `is_overshadowed`, `shard_spec`, `replication_factor`.

Use this for:
- "How many segments per datasource?"
- "What's my smallest/largest segment?" (signals compaction need)
- "Which segments are missing replicas?"

### sys.tasks — running and recent tasks

```sql
SELECT task_id, type, datasource, status, duration, error_msg
FROM sys.tasks
WHERE created_time > CURRENT_TIMESTAMP - INTERVAL '1' DAY
ORDER BY created_time DESC;
```

Use for:
- "What tasks failed?"
- "How many compaction tasks are running?"
- "Which datasources are still ingesting?"

### sys.supervisors — streaming supervisors

```sql
SELECT supervisor_id, datasource, state, healthy
FROM sys.supervisors;
```

State should be `RUNNING`. If it's `SUSPENDED` or `UNHEALTHY`, alert.

### sys.servers — cluster inventory

```sql
SELECT server, server_type, tier, curr_size, max_size,
       (curr_size * 100.0) / max_size AS pct_used
FROM sys.servers
WHERE server_type = 'historical';
```

Use for: detecting overloaded Historicals before they spill.

### sys.server_segments — which server serves which segment

```sql
SELECT server, count(*) AS segments_owned, sum(size)/1024/1024/1024 AS gb
FROM sys.server_segments s
JOIN sys.segments seg ON s.segment_id = seg.segment_id
GROUP BY 1 ORDER BY 3 DESC;
```

Use for: load distribution check.

## 16.2 Druid Metrics emitter

Druid emits metrics via the `druid-metrics` extension. Common emitters:
- **noop** (default; don't use in prod).
- **statsd** (push to StatsD aggregator).
- **graphite**.
- **prometheus** (via the contrib prometheus emitter or pulled via JMX exporter).
- **OpenTelemetry**.
- **kafka** (emit metrics to Kafka for downstream processing).

## 16.3 The metrics you actually monitor

### Ingestion health

| Metric | Use |
|--------|-----|
| `ingest/events/processed` | Throughput per supervisor/task |
| `ingest/events/unparseable` | Bad-data rate |
| `ingest/events/duplicate` | Dedup activity |
| `ingest/events/thrownAway` | Lateness-rejected events |
| `ingest/persists/cpu` | Local persist CPU time |
| `ingest/handoff/count` | Successful handoffs to Historicals |
| `ingest/handoff/failed` | Failed handoffs — alert |

### Coordinator metrics

| Metric | Use |
|--------|-----|
| `segment/loadQueue/count` | Segments still loading |
| `segment/dropQueue/count` | Segments being dropped |
| `segment/count` per tier | Total segments per tier |
| `segment/size` per tier | Total bytes per tier |
| `segment/unavailable/count` | Segments not loaded anywhere — alert |
| `segment/underReplicated/count` | Replicas behind target — alert |
| `coordinator/time` | Coordinator cycle duration |

### Query metrics

| Metric | Use |
|--------|-----|
| `query/time` | Per-query duration (histogram) |
| `query/cache/total/hits` | Broker cache hits |
| `query/segment/time` | Per-segment scan time |
| `query/bytes` | Bytes returned per query |
| `query/wait/time` | Time queue-bound before processing |
| `query/fail/count` | Failed queries — alert |

### Historical metrics

| Metric | Use |
|--------|-----|
| `segment/scan/pending` | Scan queue depth |
| `segment/cache/used` | Page cache usage |
| `jvm/mem/used` | JVM heap |

### Sketch-specific

| Metric | Use |
|--------|-----|
| Custom metrics for sketch error/accuracy if tracked | |

## 16.4 The "is the cluster happy" dashboard query

A Prometheus-style top-level dashboard typically has:
- **Ingest QPS** per datasource (rate(ingest/events/processed)).
- **Ingest lag** for streaming (Kafka consumer lag).
- **Segment count and size** per datasource per tier.
- **Unavailable segments** — should be 0.
- **Under-replicated segments** — should be 0.
- **Query QPS, p50/p99 query time** per datasource.
- **Failed query count**.
- **Broker JVM heap** usage.
- **Historical disk usage** per tier.
- **Compaction backlog**.
- **Task slot usage**.
- **ZK request rate / latency**.
- **Metadata DB query latency**.

## 16.5 Common alert thresholds

| Alert | Threshold |
|-------|-----------|
| Unavailable segments > 0 for 5m | Page |
| Under-replicated segments > N for 30m | Warn |
| Ingest lag > 1m for 5m | Page (for streaming) |
| Failed queries > 1% in 5m | Page |
| Query p99 > SLO for 5m | Warn |
| Historical heap > 90% | Warn |
| Broker heap > 90% | Page |
| Coordinator cycle > 2m | Warn |
| Auto-compaction queue growing | Warn |
| Metadata DB connection failures | Page |
| ZK leader election event | Warn (something just happened) |

## 16.6 Pivot / Web Console for the on-call

Druid ships a built-in Web Console (via Router or Coordinator):
- **Data sources** — list, metadata, recent tasks.
- **Segments** — drill down.
- **Tasks** — Overlord-side; logs accessible.
- **Supervisors** — current state.
- **Services** — process list and status.
- **Query** — run SQL with EXPLAIN.
- **Lookups** — manage.

For real BI: Pivot (Imply commercial) or Superset / Tableau / Looker integrations.

## 16.7 Logging

Druid logs to standard log4j. Routes:
- **Druid server logs** — engine messages, query failures, segment loading.
- **Task logs** — per-task; stored in deep storage (with `druid.indexer.logs.type = s3`).

Logs at scale: forward to ELK / Splunk / your log stack.

## 16.8 Tracing

Newer Druid supports OpenTelemetry tracing — span the lifecycle of an ingestion task or a query through Broker → Historicals.

## 16.9 Diagnostic queries (cheatsheet)

```sql
-- Segments per datasource by tier
SELECT s.datasource, srv.tier, count(*) AS segs, sum(s.size)/1e9 AS gb
FROM sys.segments s JOIN sys.server_segments ss ON s.segment_id = ss.segment_id
JOIN sys.servers srv ON ss.server = srv.server
WHERE s.is_published = 1 AND srv.server_type = 'historical'
GROUP BY 1, 2 ORDER BY 1, 2;

-- Small-segment count (compaction candidates)
SELECT datasource, count(*) AS small_segments
FROM sys.segments
WHERE is_published = 1 AND size < 100 * 1024 * 1024
GROUP BY 1 HAVING count(*) > 10
ORDER BY 2 DESC;

-- Recent task failures
SELECT task_id, type, datasource, error_msg
FROM sys.tasks
WHERE status = 'FAILED' AND created_time > CURRENT_TIMESTAMP - INTERVAL '1' DAY
ORDER BY created_time DESC LIMIT 50;

-- Historical disk usage
SELECT server, tier, curr_size/1e9 AS gb_used, max_size/1e9 AS gb_max,
       (curr_size * 100.0)/max_size AS pct
FROM sys.servers
WHERE server_type = 'historical' ORDER BY pct DESC;

-- Long-running supervisors
SELECT supervisor_id, datasource, state, sustained_running_seconds
FROM sys.supervisors WHERE state != 'RUNNING';
```

## 16.10 Compare with ClickHouse

| | ClickHouse | Druid |
|--|------------|-------|
| System tables | `system.*` (very rich) | `sys.*` (similar concept; less rich) |
| Metric emit | Prometheus exporter scrapes system.metrics/events | JMX or contrib emitters (Prometheus/StatsD/Kafka) |
| Per-query log | `system.query_log` | task-log emit + Broker `query/time` metric |
| Console | clickhouse-client + Grafana | built-in Web Console + Pivot |

Both engines provide a queryable layer for observability. Druid leans more on JMX metrics + the console; CH leans more on `system.*` tables for everything.

## 16.11 Must-internalize

- `sys.segments`, `sys.tasks`, `sys.supervisors`, `sys.servers`, `sys.server_segments` are your forensic tables.
- Wire metrics to Prometheus / Datadog / StatsD before going to prod.
- Alert on: unavailable segments, ingest lag, failed queries, Broker heap, Historical disk.
- The Web Console handles routine triage; observability stack handles trend / alerting.
- Cross-reference: `EXPLAIN PLAN` + `sys.tasks` + per-query metric trace to find slow queries.

---

## Sources

- [SQL system tables (sys schema)](https://druid.apache.org/docs/latest/querying/sql-metadata-tables/)
- [Metrics — emitting](https://druid.apache.org/docs/latest/configuration/extensions/)
- [Web console](https://druid.apache.org/docs/latest/operations/web-console/)
- [Metrics reference](https://druid.apache.org/docs/latest/operations/metrics/)
- [Druid OpenTelemetry emitter](https://druid.apache.org/docs/latest/configuration/extensions/#opentelemetry-emitter)
