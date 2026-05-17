# 6 · Batch Ingestion and MSQ

Batch ingestion in Druid has three eras:
1. **Hadoop ingestion** (legacy, declining).
2. **Native batch** (`index_parallel` task) — current default.
3. **MSQ (Multi-Stage Query) engine** — modern, SQL-based, the future.

Most new Druid work targets MSQ.

## 6.1 Native batch — `index_parallel`

A JSON spec submitted as a task to the Overlord. Reads from an external source (S3, HDFS, local file, JDBC, Kafka offset range), partitions across worker tasks, writes segments.

```json
{
  "type": "index_parallel",
  "spec": {
    "dataSchema": { ... },                     -- same dimensionsSpec/metricsSpec as streaming
    "ioConfig": {
      "type": "index_parallel",
      "inputSource": {
        "type": "s3",
        "uris": ["s3://bucket/events/2026-05-17/*.json.gz"]
      },
      "inputFormat": { "type": "json" }
    },
    "tuningConfig": {
      "type": "index_parallel",
      "maxNumConcurrentSubTasks": 50,
      "partitionsSpec": { "type": "hashed", "numShards": 8 }
    }
  }
}
```

Two-stage: a controller task discovers input + creates subtasks; subtasks read and write segments in parallel.

Modes:
- **Append** — adds new segments without overshadowing existing ones.
- **Replace** (via `appendToExisting: false` + interval) — new segments overshadow existing in the same interval.

Native batch is the workhorse for one-off backfills and scheduled batch loads.

## 6.2 MSQ — the modern SQL-based ingestion

**Multi-Stage Query** (MSQ) is a query engine that runs SQL `INSERT` and `REPLACE` statements as batch tasks. It can also run `SELECT` queries (experimental, for very large analytical reads).

The model: SQL with `INSERT INTO ... SELECT ...` (or `REPLACE ... SELECT`) that ingests from an external source via `EXTERN`.

```sql
INSERT INTO events
SELECT
  TIME_PARSE(ts) AS __time,
  country,
  event_type,
  user_id,
  CAST(revenue AS DOUBLE) AS revenue
FROM TABLE(
  EXTERN(
    '{"type":"s3","uris":["s3://bucket/events/2026-05-17/*.json.gz"]}',
    '{"type":"json"}',
    '[{"name":"ts","type":"string"},{"name":"country","type":"string"},
      {"name":"event_type","type":"string"},{"name":"user_id","type":"long"},
      {"name":"revenue","type":"double"}]'
  )
)
PARTITIONED BY HOUR
CLUSTERED BY country, event_type;
```

What's happening:
- **Stage 1**: read S3 files in parallel, parse JSON, project columns.
- **Stage 2**: shuffle by partition (HOUR buckets here).
- **Stage 3**: per-partition rollup + segment build.
- **Stage 4**: publish segments to deep storage + register in metadata DB.

Each stage is a set of worker tasks that read from previous stage's outputs and write to next. Looks a lot like Spark.

## 6.3 INSERT vs REPLACE

### INSERT

```sql
INSERT INTO events SELECT ...
PARTITIONED BY DAY
CLUSTERED BY country;
```

- Appends segments to existing intervals.
- If an existing segment covers the same interval, both coexist (Druid handles overshadowing if versions differ).

### REPLACE

```sql
REPLACE INTO events
  OVERWRITE WHERE __time >= TIMESTAMP '2026-05-17' AND __time < TIMESTAMP '2026-05-18'
SELECT ...
PARTITIONED BY DAY
CLUSTERED BY country;
```

- Replaces all data in the specified time interval with the new SELECT result.
- Atomic from a query's perspective: queries see either the old or the new, never partial.

`REPLACE` is how you fix bad data, reprocess with a new schema, or backfill historical data.

## 6.4 MSQ vs native batch

| | Native batch | MSQ |
|--|-------------|-----|
| Interface | JSON spec | SQL |
| Joins / transformations | Limited | Full SQL (CTEs, subqueries, joins, aggregates) |
| Stage shuffle | No | Yes |
| Use case | Pure ingestion | Ingestion + transformation + ETL |
| Engine | Indexer's incremental index | A multi-stage executor that resembles Spark |

For new work: prefer MSQ. It's the strategic direction and offers SQL ergonomics on top of ingestion.

## 6.5 MSQ stages and workers

A typical MSQ INSERT runs as:
1. **Controller task** (`query_controller`) — runs on the Overlord-assigned MiddleManager/Indexer slot. Manages the query.
2. **Worker tasks** (`query_worker`) — one or more per stage. Each worker handles a partition of the work.

Stages communicate via **shuffle files** written to disk (locally on the worker or in a configurable shuffle store, including S3 for cross-region resilience).

Resource sizing knobs:
- `taskAssignment` — how aggressively to spawn worker tasks.
- `maxNumTasks` — cap on total tasks.
- `intermediateMaxNumRows`, `maxRowsInMemory` — per-stage memory.
- `selectDestination` — for SELECT queries, where to write results.

## 6.6 EXTERN and reading external data

```sql
SELECT count(*)
FROM TABLE(
  EXTERN(
    '{"type":"s3","uris":["s3://my/path/*.parquet"]}',
    '{"type":"parquet"}',
    '[{"name":"col1","type":"string"},...]'
  )
);
```

`EXTERN` takes three JSON strings: input source, input format, signature (schema hint). Supports S3, GCS, HDFS, Azure, HTTP, local file, JDBC.

Format support: JSON, CSV, TSV, Parquet, ORC, Avro, Protobuf.

This is essentially Druid's table function for ad-hoc reads — query S3 data without first ingesting.

## 6.7 Iceberg / Delta read

Newer Druid versions can read Apache Iceberg / Delta tables directly via the iceberg/delta input source:

```sql
SELECT ... FROM TABLE(EXTERN('{"type":"iceberg",...}', ...))
```

Useful for hybrid lakehouse + Druid architectures.

## 6.8 MSQ for batch transformations

MSQ shines for ETL-style work:

```sql
REPLACE INTO daily_summary
  OVERWRITE WHERE __time = TIMESTAMP '2026-05-17'
SELECT
  TIME_FLOOR(__time, 'P1D')         AS __time,
  country,
  COUNT(*)                          AS events,
  SUM(revenue)                      AS revenue,
  APPROX_COUNT_DISTINCT_DS_HLL(user_id) AS users
FROM events
WHERE __time >= TIMESTAMP '2026-05-17'
  AND __time <  TIMESTAMP '2026-05-18'
GROUP BY 1, 2
PARTITIONED BY DAY;
```

This reads the raw events table, rolls up to daily granularity per country, and writes to a `daily_summary` datasource. Comparable to ClickHouse's `INSERT INTO target SELECT FROM source` pattern.

## 6.9 CLUSTERED BY (secondary sort)

`CLUSTERED BY country, event_type` declares the dimension sort order within each segment. Equivalent to ClickHouse's `ORDER BY (country, event_type)`. Drives:
- Better compression (similar values adjacent).
- Faster filter on leading columns.

Pick the columns you most often filter on, low-cardinality first.

## 6.10 PARTITIONED BY (time granularity)

`PARTITIONED BY HOUR | DAY | MONTH | YEAR | ALL` = `segmentGranularity`. Required.

Pick the interval that:
- Hits the 300-700 MB sweet spot per segment.
- Matches your query time-range patterns.

## 6.11 Backfill workflow (common pattern)

Typical scenario: a new feature requires reprocessing 30 days of past data.

```sql
-- One REPLACE per day, run in parallel as separate tasks:
REPLACE INTO events
  OVERWRITE WHERE __time >= TIMESTAMP '2026-04-17'
                AND __time <  TIMESTAMP '2026-04-18'
SELECT ...
FROM TABLE(EXTERN(...s3 path for that day...))
PARTITIONED BY DAY
CLUSTERED BY country;
```

The Overlord queues them; the Coordinator handles the segment swap atomically per day. Backfill 30 days = 30 tasks; can run sequentially or concurrently.

For very large backfills (months), schedule outside business hours to avoid contending with streaming ingest tasks.

## 6.12 Combining streaming + batch on the same datasource

Common Druid pattern: stream recent data via Kafka supervisor, batch-reingest historical data via MSQ for cleanup or schema change.

The Coordinator handles the segment swap atomically. The Kafka supervisor continues running unaffected; its segments coexist with the batch-produced segments.

## 6.13 MSQ caveats

- **Memory-heavy for large groupings**. The aggregation hash table lives in memory per worker. Tune `maxRowsInMemory`.
- **Shuffle disk usage** — large intermediate writes; configure `intermediateMaxNumRows` to control.
- **Cross-stage skew** — uneven partition sizes (a hot tenant) creates straggler tasks. Use `CLUSTERED BY` wisely.
- **No incremental updates** — INSERT/REPLACE only. Modifications need re-ingest of the affected interval.

## 6.14 Worked example — daily ETL job

Scenario: every night, aggregate yesterday's raw events into a smaller summary table.

```sql
REPLACE INTO daily_summary
  OVERWRITE WHERE __time = TIMESTAMP '2026-05-17'
WITH base AS (
  SELECT
    TIME_FLOOR(__time, 'P1D') AS day,
    country,
    event_type,
    user_id,
    revenue
  FROM events
  WHERE __time >= TIMESTAMP '2026-05-17'
    AND __time <  TIMESTAMP '2026-05-18'
)
SELECT
  day                                       AS __time,
  country,
  event_type,
  COUNT(*)                                  AS events,
  SUM(revenue)                              AS revenue,
  APPROX_COUNT_DISTINCT_DS_HLL(user_id)     AS distinct_users
FROM base
GROUP BY day, country, event_type
PARTITIONED BY DAY
CLUSTERED BY country, event_type;
```

Submit as an MSQ task; runs in 1-2 minutes for a billion-row input. Dashboards on `daily_summary` are ~10× faster than on `events`.

## 6.15 Must-internalize

- Native batch = JSON-spec task; MSQ = SQL-based, the modern path.
- MSQ does INSERT (append) and REPLACE OVERWRITE (atomic interval swap).
- `EXTERN` reads external sources directly; supports JSON/Parquet/CSV/Avro/ORC/Protobuf/Iceberg/Delta.
- `PARTITIONED BY HOUR/DAY/MONTH` is required.
- `CLUSTERED BY` is the secondary sort order within segments.
- Backfills are per-time-range REPLACEs; combine streaming + batch on the same datasource freely.
- MSQ can also serve as an ETL engine (rollup, transformation, etc.).

---

## Sources

- [SQL-based ingestion — official](https://druid.apache.org/docs/latest/multi-stage-query/)
- [Concepts](https://druid.apache.org/docs/latest/multi-stage-query/concepts/)
- [Reference](https://druid.apache.org/docs/latest/multi-stage-query/reference/)
- [Examples](https://druid.apache.org/docs/latest/multi-stage-query/examples/)
- [Native batch (index_parallel)](https://druid.apache.org/docs/latest/ingestion/native-batch/)
