# 21 · Quick Reference Cheatsheets

The night-before review.

## 21.1 The six processes

| Process | Role |
|---------|------|
| **Coordinator** | Segment placement, replication, balance, compaction, kill |
| **Overlord** | Task assignment + supervisor management |
| **Broker** | Query gateway, fan-out, merge |
| **Historical** | Serves segments from local cache |
| **MiddleManager + Peons** / **Indexer** | Run ingestion tasks |
| **Router** | Optional API gateway |

Plus external: **deep storage** (S3/HDFS/GCS), **metadata DB** (MySQL/Postgres), **ZooKeeper**.

## 21.2 Segment cheat-sheet

- Identifier: `datasource_interval_version_shardSpec`
- Sweet spot size: **300-700 MB**
- Rows per segment: **5-15M**
- Format: columnar, dictionary-encoded strings, **Roaring bitmap** per dimension value, optional **front-coded** dictionaries

## 21.3 Native query types

| Type | When | Speed |
|------|------|-------|
| Timeseries | Time-only group-by | Fastest |
| TopN | Top-N single dim (approx) | Very fast |
| GroupBy v2 | Multi-dim group-by | Slower |
| Scan | Raw row fetch with small LIMIT | Fast |
| Search | LIKE / dimension value search | UI-fast |

## 21.4 Metric / sketch matrix

| Need | Metric type |
|------|-------------|
| Count | `count` |
| Sum | `longSum` / `doubleSum` / `floatSum` |
| Min/Max | `longMin` / `longMax` |
| First/Last | `longFirst` / `longLast` |
| Distinct (approx) | **HLLSketch** |
| Distinct + set algebra | **thetaSketch** |
| Quantile / percentile | **quantilesDoublesSketch** |
| Multi-column distinct | **tupleSketch (arrayOfDoubles)** |
| Conditional metric | `filtered` (wraps another aggregator) |

## 21.5 Rollup config

```json
{
  "granularitySpec": {
    "segmentGranularity": "DAY",
    "queryGranularity": "MINUTE",
    "rollup": true
  }
}
```

Common pairs:
- Streaming hot: `HOUR` + `MINUTE`
- Standard analytics: `DAY` + `MINUTE`
- Long-range reports: `MONTH` + `HOUR`
- Forensic logs: `DAY` + `NONE` (`rollup: false`)

## 21.6 Sharding (`PARTITION BY` / `CLUSTERED BY` in MSQ; `partitionsSpec` in native)

- `PARTITIONED BY HOUR | DAY | MONTH | YEAR | ALL` — segment granularity (time).
- `CLUSTERED BY a, b, c` — secondary sort within segment; low-cardinality first.

## 21.7 Load rules pattern

```json
[
  { "type": "loadByPeriod", "period": "P7D",   "tieredReplicants": { "hot": 2 } },
  { "type": "loadByPeriod", "period": "P30D",  "tieredReplicants": { "_default": 1 } },
  { "type": "loadByPeriod", "period": "P365D", "tieredReplicants": { "cold": 1 } },
  { "type": "dropForever" }
]
```

## 21.8 Auto-compaction config

```json
{
  "dataSource": "events",
  "skipOffsetFromLatest": "PT1H",
  "tuningConfig": { "maxRowsPerSegment": 5000000 },
  "granularitySpec": { "segmentGranularity": "DAY", "queryGranularity": "MINUTE" }
}
```

## 21.9 Kafka supervisor essentials

```json
{
  "type": "kafka",
  "spec": {
    "dataSchema": { ... },
    "ioConfig": {
      "topic": "events",
      "consumerProperties": { "bootstrap.servers": "kafka:9092" },
      "inputFormat": { "type": "json" },
      "taskCount": 4,
      "replicas": 2,
      "taskDuration": "PT1H"
    },
    "tuningConfig": {
      "type": "kafka",
      "maxRowsInMemory": 1000000,
      "maxRowsPerSegment": 5000000,
      "intermediatePersistPeriod": "PT10M"
    }
  }
}
```

## 21.10 MSQ INSERT / REPLACE skeleton

```sql
INSERT INTO events
SELECT ... FROM TABLE(EXTERN('{...}','{...}','[...]'))
PARTITIONED BY DAY
CLUSTERED BY tenant_id, event_name;

REPLACE INTO events
  OVERWRITE WHERE __time >= TIMESTAMP '...' AND __time < TIMESTAMP '...'
SELECT ... FROM ...
PARTITIONED BY DAY;
```

## 21.11 Druid SQL idioms

```sql
-- time filter (always)
WHERE __time >= CURRENT_TIMESTAMP - INTERVAL '1' DAY

-- time bucketing
TIME_FLOOR(__time, 'PT1M')
FLOOR(__time TO MINUTE)

-- approximate distinct
APPROX_COUNT_DISTINCT_DS_HLL(distinct_users_hll)

-- approximate quantile
APPROX_QUANTILE_DS(quantile_sketch, 0.99)

-- theta sketch ops
THETA_SKETCH_ESTIMATE(THETA_SKETCH_INTERSECT(a.sk, b.sk))
THETA_SKETCH_ESTIMATE(THETA_SKETCH_NOT(a.sk, b.sk))

-- lookup
LOOKUP(country_code, 'country_lookup')

-- filtered aggregate
SUM(revenue) FILTER (WHERE event = 'purchase')

-- JSON
JSON_VALUE(props, '$.browser')
```

## 21.12 sys schema cheatsheet

```sql
SELECT count(*) FROM sys.segments WHERE is_published = 1;
SELECT * FROM sys.tasks WHERE status = 'FAILED' AND created_time > CURRENT_TIMESTAMP - INTERVAL '1' DAY;
SELECT * FROM sys.supervisors WHERE state != 'RUNNING';
SELECT server, tier, curr_size/1e9 FROM sys.servers WHERE server_type = 'historical';
```

## 21.13 Alert thresholds

- Unavailable segments > 0 for 5m: **page**
- Under-replicated segments > N for 30m: warn
- Ingest lag > 1m for 5m: **page**
- Failed query rate > 1% / 5m: **page**
- Broker JVM heap > 90%: **page**
- Coordinator cycle > 2m: warn
- Historical disk free < 15%: **page**

## 21.14 The 30-second pre-interview reminder

- **6 processes**: Coordinator (segments), Overlord (tasks), Broker (queries), Historical (serves), MiddleManager/Indexer (ingests), Router (gateway). External: deep storage, metadata DB, ZK.
- **Segments**: immutable, time-partitioned, columnar, dictionary + Roaring bitmap per dim, 300-700 MB sweet spot.
- **Time partitioning is mandatory**; rollup at ingest is the first-class story.
- **DataSketches** (HLL/theta/quantile) replace high-cardinality dims for distinct/cohort/percentile metrics.
- **Streaming supervisors** with exactly-once via metadata-DB-stored Kafka offsets.
- **MSQ** for batch + REPLACE for data fixes; no UPDATE/DELETE.
- **5 native query types**: Timeseries (fastest), TopN (approx top-N), GroupBy (multi-dim), Scan (raw rows), Search (typeahead).
- **Joins are broadcast hash only**; use **lookups** for small dims; denormalize for repeated patterns.
- **Coordinator rules** = retention + tiering. **Auto-compaction** prevents small-segment fan-out.
- **Always time-filter**. **EXPLAIN PLAN** to verify native query type.
- **Approximate is the default** for distinct / quantile.
- **No UPDATE/DELETE**; use REPLACE for any data fix.
- **System tables**: `sys.segments`, `sys.tasks`, `sys.supervisors`, `sys.servers`, `sys.server_segments`.
- **Wire JMX metrics** to Prometheus before going to prod.
- **Druid vs ClickHouse**: 6 processes vs 1; rollup-at-ingest vs MV; broadcast joins only vs 6 algorithms; exactly-once streaming vs at-least-once with dedup; no DML vs full DML + lightweight ops.

Breathe.
