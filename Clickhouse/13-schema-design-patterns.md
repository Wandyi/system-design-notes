# 13 · Schema Design Patterns — Concrete Worked Examples

The schema is the architecture. This file walks through eight common workloads, the right schema for each, the rationale, and the corner cases. Use these as templates and tune.

For each scenario:
1. The workload shape.
2. A DDL you could ship.
3. The reasoning (why this engine, ORDER BY, codecs, indexes).
4. The MV(s) for common queries.
5. Anti-patterns to avoid.
6. Variants for scale.

## 13.1 General principles (the shared rubric)

1. **One wide flat table** beats a normalized schema in CH. Denormalize.
2. **ORDER BY** = your most common filter + locality. Put low-cardinality high-selectivity columns first.
3. **PARTITION BY** = your TTL/lifecycle boundary (often `toYYYYMM(ts)`). Avoid > 1000 active partitions.
4. **LowCardinality** for any string with < millions of distinct values.
5. **DateTime** with `Delta + ZSTD(1)` codec.
6. **Materialized views** to AggregatingMergeTree for repeated aggregations.
7. **Dictionaries** for any lookup of slowly-changing dimension data.
8. **TTL** for retention; **DROP PARTITION** for big bulk deletes.

---

## 13.2 Pattern 1 — Event tracking / product analytics (PostHog-like)

**Workload**: Each user action emits an event. Aggregate by event type, day, user; build funnels and retention queries.

```sql
CREATE TABLE events (
    project_id     UInt32,
    user_id        UInt64,
    session_id     UUID,
    ts             DateTime64(3) CODEC(Delta, ZSTD(1)),
    event_name     LowCardinality(String),
    properties     Map(LowCardinality(String), String),     -- open-ended
    revenue        Decimal64(4) CODEC(T64, ZSTD(1)),
    country        LowCardinality(String),
    browser        LowCardinality(String),
    os             LowCardinality(String),
    device_type    Enum8('desktop'=1, 'mobile'=2, 'tablet'=3, 'other'=4),
    page_url       String CODEC(ZSTD(3)),
    referrer_host  LowCardinality(String)
)
ENGINE = ReplicatedMergeTree('/ch/tables/{shard}/events','{replica}')
PARTITION BY toYYYYMM(ts)
ORDER BY (project_id, event_name, toDate(ts), user_id, ts)
TTL toDateTime(ts) + INTERVAL 18 MONTH;
```

Why:
- `project_id` first → multi-tenant locality (most queries are scoped).
- `event_name` → analytics queries usually filter by event ("page_view", "checkout").
- `toDate(ts)` → daily ranges are the hot filter.
- `user_id` then `ts` → within a day, user-scoped lookups are fast and rows are timestamp-ordered (good for Delta on `ts`).
- `Map(LowCardinality(String), String)` for free-form properties; keys are deduplicated.
- `ts` codec for time-series compression.

### MV: per-day per-event counts and revenue

```sql
CREATE TABLE daily_event_agg (
    project_id   UInt32,
    day          Date,
    event_name   LowCardinality(String),
    cnt          AggregateFunction(count),
    users        AggregateFunction(uniq, UInt64),
    revenue      AggregateFunction(sum, Decimal64(4))
)
ENGINE = ReplicatedAggregatingMergeTree('/ch/tables/{shard}/daily_event_agg','{replica}')
ORDER BY (project_id, day, event_name);

CREATE MATERIALIZED VIEW daily_event_agg_mv TO daily_event_agg AS
SELECT
    project_id,
    toDate(ts) AS day,
    event_name,
    countState()              AS cnt,
    uniqState(user_id)        AS users,
    sumState(revenue)         AS revenue
FROM events
GROUP BY project_id, day, event_name;
```

### Funnel query (PostHog-style)

```sql
SELECT
    level,
    count() AS users
FROM (
    SELECT
        user_id,
        windowFunnel(3600)(ts,
            event_name = 'view_product',
            event_name = 'add_to_cart',
            event_name = 'checkout'
        ) AS level
    FROM events
    WHERE project_id = 42
      AND ts >= today() - 7
      AND event_name IN ('view_product','add_to_cart','checkout')
    GROUP BY user_id
)
GROUP BY level;
```

### Anti-patterns

- `ORDER BY (ts, ...)` first — ruins per-tenant pruning.
- `String` for `event_name` — should be `LowCardinality`.
- `JSON` for `properties` when you know all key types — use typed columns.
- Per-user partitions.

### Scale variants

- Single tenant gets > 10% of writes → shard by `cityHash64(user_id)` to spread.
- Properties grow free-form → consider new `JSON` type for `properties`.

---

## 13.3 Pattern 2 — Time-series metrics (Prometheus-like)

**Workload**: One numeric metric per timestamp per (metric_name, label set). Read aggregations over time.

```sql
CREATE TABLE metrics (
    metric_name  LowCardinality(String),
    ts           DateTime CODEC(DoubleDelta, ZSTD(1)),
    value        Float64 CODEC(Gorilla, ZSTD(1)),
    labels       Map(LowCardinality(String), LowCardinality(String))
)
ENGINE = ReplicatedMergeTree('/ch/tables/{shard}/metrics','{replica}')
PARTITION BY toYYYYMM(ts)
ORDER BY (metric_name, ts);
```

Why:
- `metric_name` first — most queries pick a metric.
- `ts` second with DoubleDelta — regular sampling (Prometheus is every 15-60s).
- Gorilla codec for the float column — slowly-varying-ish values.
- `Map(LC, LC)` for labels.

### MV: per-minute rollups

```sql
CREATE TABLE metrics_1m (
    metric_name LowCardinality(String),
    minute      DateTime,
    labels_hash UInt64,
    labels      Map(LowCardinality(String), LowCardinality(String)),
    avg_value   AggregateFunction(avg, Float64),
    max_value   AggregateFunction(max, Float64),
    min_value   AggregateFunction(min, Float64),
    last_value  AggregateFunction(argMax, Float64, DateTime),
    cnt         AggregateFunction(count)
)
ENGINE = ReplicatedAggregatingMergeTree('/ch/tables/{shard}/metrics_1m','{replica}')
ORDER BY (metric_name, minute, labels_hash);

CREATE MATERIALIZED VIEW metrics_1m_mv TO metrics_1m AS
SELECT
    metric_name,
    toStartOfMinute(ts) AS minute,
    cityHash64(labels)  AS labels_hash,
    any(labels)         AS labels,
    avgState(value)     AS avg_value,
    maxState(value)     AS max_value,
    minState(value)     AS min_value,
    argMaxState(value, ts) AS last_value,
    countState()        AS cnt
FROM metrics
GROUP BY metric_name, minute, labels_hash;
```

### Variants

- Use `Nested` instead of `Map` if labels have a fixed schema per metric (rare).
- Cascade `metrics_1m → metrics_5m → metrics_1h → metrics_1d` for multi-granularity.

### Corner cases

- Labels with very high cardinality (e.g., URL paths) → blow up cardinality and storage. Force users to drop or bucket high-cardinality labels.
- Counter values that reset (Prometheus rate calculation) → store raw, compute `rate()` at read time via `runningDifference` or windowed functions.

---

## 13.4 Pattern 3 — Logs / observability (ELK replacement)

**Workload**: Log lines with timestamp, level, service, message, and structured fields. Search by text + filter.

```sql
CREATE TABLE logs (
    ts          DateTime64(3) CODEC(Delta, ZSTD(1)),
    service     LowCardinality(String),
    env         LowCardinality(String),
    host        LowCardinality(String),
    severity    Enum8('TRACE'=1, 'DEBUG'=2, 'INFO'=3, 'WARN'=4, 'ERROR'=5, 'FATAL'=6),
    trace_id    String CODEC(ZSTD(1)),
    span_id     String CODEC(ZSTD(1)),
    message     String CODEC(ZSTD(3)),
    attrs       Map(LowCardinality(String), String),
    INDEX  idx_trace      trace_id TYPE bloom_filter(0.01) GRANULARITY 4,
    INDEX  idx_msg_token  message  TYPE tokenbf_v1(32768, 4, 0) GRANULARITY 4
)
ENGINE = ReplicatedMergeTree('/ch/tables/{shard}/logs','{replica}')
PARTITION BY toYYYYMMDD(ts)
ORDER BY (service, severity, ts)
TTL toDateTime(ts) + INTERVAL 14 DAY;
```

Why:
- `service`, `severity` first — almost every query filters by both.
- `ts` last in ORDER BY — within a service+severity, sort by time.
- Daily partitions for 14-day retention; ~14 active partitions.
- Bloom filter on `trace_id` for fast trace lookup.
- Token bloom filter on `message` for `LIKE '%foo%'` searches.

### Variant: OpenTelemetry-style

ClickHouse has growing OTel support; recent versions ship semantic-convention table templates (`otel_logs`, `otel_traces`, `otel_metrics`).

### Anti-patterns

- `ORDER BY ts` first → all-service scans become slow.
- Trying to index every `attrs` key → use a `Map`, query with `attrs['key']`.
- `String` for severity — use Enum8 / LowCardinality.

---

## 13.5 Pattern 4 — User sessions / "latest state" per user

**Workload**: A user record that changes over time. We want the current state quickly.

```sql
CREATE TABLE users (
    user_id     UInt64,
    name        String,
    email       String,
    country     LowCardinality(String),
    plan        Enum8('free'=1,'pro'=2,'enterprise'=3),
    is_deleted  UInt8 DEFAULT 0,
    updated_at  DateTime
)
ENGINE = ReplicatedReplacingMergeTree(
    '/ch/tables/{shard}/users','{replica}',
    updated_at
)
ORDER BY user_id;
```

### Read patterns

```sql
-- naive (works but slow on big tables)
SELECT * FROM users FINAL WHERE user_id = 42;

-- preferred for analytical queries
SELECT
    user_id,
    argMax(name,    updated_at) AS name,
    argMax(email,   updated_at) AS email,
    argMax(plan,    updated_at) AS plan,
    argMax(country, updated_at) AS country,
    max(updated_at)             AS updated_at
FROM users
WHERE user_id = 42
GROUP BY user_id;

-- top latest per user (no FINAL)
SELECT * FROM users ORDER BY user_id, updated_at DESC LIMIT 1 BY user_id;
```

### MV: maintain a clean current-state table

```sql
CREATE TABLE users_current (
    user_id     UInt64,
    state       AggregateFunction(argMax, Tuple(String,String,LowCardinality(String), Enum8('free'=1,'pro'=2,'enterprise'=3)), DateTime),
    updated_at  AggregateFunction(max, DateTime)
)
ENGINE = ReplicatedAggregatingMergeTree(...)
ORDER BY user_id;

CREATE MATERIALIZED VIEW users_current_mv TO users_current AS
SELECT
    user_id,
    argMaxState((name, email, country, plan), updated_at) AS state,
    maxState(updated_at) AS updated_at
FROM users
GROUP BY user_id;
```

### Cleaning deleted rows

```sql
ALTER TABLE users MODIFY SETTING clean_deleted_rows = 'Always';
```

With this setting, ReplacingMergeTree drops rows where `is_deleted = 1` on merge.

---

## 13.6 Pattern 5 — Ad-tech impressions and clicks

**Workload**: Massive append; multi-tenant; report aggregates per advertiser/campaign per minute.

```sql
CREATE TABLE impressions (
    ts             DateTime CODEC(Delta, ZSTD(1)),
    advertiser_id  UInt32,
    campaign_id    UInt32,
    creative_id    UInt32,
    geo            LowCardinality(String),
    device         Enum8('desktop'=1,'mobile'=2,'tv'=3,'tablet'=4),
    user_hash      UInt64,
    cost_cents     UInt32 CODEC(T64, ZSTD(1)),
    revenue_cents  UInt32 CODEC(T64, ZSTD(1))
)
ENGINE = ReplicatedMergeTree(...)
PARTITION BY toYYYYMM(ts)
ORDER BY (advertiser_id, campaign_id, ts);
```

### MV: per-minute campaign rollup

```sql
CREATE TABLE campaign_minute_agg (
    minute        DateTime,
    advertiser_id UInt32,
    campaign_id   UInt32,
    impressions   AggregateFunction(count),
    cost          AggregateFunction(sum, UInt32),
    revenue       AggregateFunction(sum, UInt32),
    users         AggregateFunction(uniq, UInt64)
)
ENGINE = ReplicatedAggregatingMergeTree(...)
ORDER BY (advertiser_id, campaign_id, minute);

CREATE MATERIALIZED VIEW campaign_minute_mv TO campaign_minute_agg AS
SELECT
    toStartOfMinute(ts) AS minute,
    advertiser_id, campaign_id,
    countState()              AS impressions,
    sumState(cost_cents)      AS cost,
    sumState(revenue_cents)   AS revenue,
    uniqState(user_hash)      AS users
FROM impressions
GROUP BY minute, advertiser_id, campaign_id;
```

### Reporting

```sql
SELECT
    advertiser_id, campaign_id,
    sumMerge(impressions) AS imps,
    sumMerge(cost)/100    AS cost,
    sumMerge(revenue)/100 AS rev,
    uniqMerge(users)      AS users
FROM campaign_minute_agg
WHERE minute >= now() - INTERVAL 1 DAY
GROUP BY advertiser_id, campaign_id
ORDER BY rev DESC LIMIT 100;
```

### Variant: clicks/conversions in separate tables, joined via `cityHash64(user_hash)`

If you really need click-through attribution, store impressions, clicks, conversions in three tables sharded by `user_hash`, do a co-located GLOBAL JOIN on the hash.

---

## 13.7 Pattern 6 — Multi-tenant SaaS analytics

**Workload**: Many tenants, each with their own filterable analytics. Strict isolation required.

```sql
CREATE TABLE tenant_events (
    tenant_id   UInt32,
    user_id     UInt64,
    ts          DateTime64(3) CODEC(Delta, ZSTD(1)),
    event       LowCardinality(String),
    props       Map(LowCardinality(String), String)
)
ENGINE = ReplicatedMergeTree(...)
PARTITION BY toYYYYMM(ts)
ORDER BY (tenant_id, event, ts, user_id);

-- row-level security
CREATE ROW POLICY tenant_isolation ON tenant_events
FOR SELECT
USING tenant_id = toUInt32(currentSetting('tenant_id'))
TO ALL;

-- quotas
CREATE QUOTA tenant_quota
FOR INTERVAL 1 MINUTE MAX QUERIES = 60, MAX RESULT_BYTES = '1Gi'
TO tenant_users;
```

Then your application sets the tenant_id via session settings:
```sql
SET tenant_id = '12345';
SELECT count() FROM tenant_events WHERE event = 'login';
-- row policy injects: AND tenant_id = 12345
```

### Sharding for big tenants

Shard the cluster by `cityHash64(tenant_id)` so each tenant lives on one shard. Big tenants get their own shard via re-bucketing.

### Variants

- **One database per tenant** for very high-isolation requirements (compliance) — cleaner blast radius, more operational overhead.
- **Dedicated Cloud service per top tenant** — common for the "whale" pattern.

---

## 13.8 Pattern 7 — CDC / replicated mutable database mirror

**Workload**: Mirror a PostgreSQL/MySQL table into ClickHouse with continuous updates.

Option A: **MaterializedPostgreSQL** / **MaterializedMySQL** engine.

```sql
CREATE DATABASE app_mirror
ENGINE = MaterializedPostgreSQL('host:5432', 'app', 'user', 'pw')
SETTINGS materialized_postgresql_tables_list = 'orders,users';
```

ClickHouse becomes a logical replication subscriber; each Postgres table appears in ClickHouse, kept up to date.

Option B: **Debezium / CDC into Kafka → Kafka engine → MV → ReplacingMergeTree**.

```sql
CREATE TABLE orders_cdc (
    op           LowCardinality(String),  -- 'c'/'u'/'d'
    payload      String,
    ts_ms        UInt64
) ENGINE = Kafka('broker:9092','orders_cdc','ch-cdc','JSONEachRow');

CREATE TABLE orders (
    order_id     UInt64,
    user_id      UInt64,
    status       LowCardinality(String),
    total_cents  UInt64,
    updated_at   DateTime,
    is_deleted   UInt8 DEFAULT 0
) ENGINE = ReplicatedReplacingMergeTree(updated_at)
ORDER BY order_id
SETTINGS clean_deleted_rows = 'Always';

CREATE MATERIALIZED VIEW orders_cdc_mv TO orders AS
SELECT
    JSONExtractUInt(payload, 'order_id')                     AS order_id,
    JSONExtractUInt(payload, 'user_id')                      AS user_id,
    JSONExtractString(payload, 'status')                     AS status,
    JSONExtractUInt(payload, 'total_cents')                  AS total_cents,
    toDateTime(ts_ms / 1000)                                 AS updated_at,
    op = 'd'                                                 AS is_deleted
FROM orders_cdc;
```

Read with the `argMax` or `LIMIT 1 BY` patterns to get current state, or use FINAL for ad-hoc.

---

## 13.9 Pattern 8 — Wide-table observability / single-pane-of-glass

**Workload**: dozens to hundreds of columns; analytical queries pick small subsets. Common in observability, ML feature stores.

```sql
CREATE TABLE wide_events (
    ts        DateTime CODEC(Delta, ZSTD(1)),
    tenant    UInt32,
    -- 100 typed columns
    col_001 String CODEC(ZSTD(3)),
    col_002 LowCardinality(String),
    ...
    col_100 Float64 CODEC(Gorilla, ZSTD(1)),
    -- and a JSON catch-all for new fields
    extras   JSON
) ENGINE = ReplicatedMergeTree(...)
PARTITION BY toYYYYMM(ts)
ORDER BY (tenant, ts);
```

ClickHouse handles wide tables well because column-store reads only what you need. Per-column codecs let you size each column independently.

The new `JSON` type lets you keep schema-evolving fields as a catch-all without re-deploying. Hint frequently-accessed keys:

```sql
extras JSON(latency_ms UInt32, region LowCardinality(String))
```

---

## 13.10 Cross-cutting recommendations

- **Test compression**: `system.parts_columns` to see actual ratios per column.
- **Test pruning**: `EXPLAIN PLAN indexes=1` to see how many granules were skipped.
- **Test cardinality**: `SELECT count(distinct col) FROM t` on a sample.
- **Test the read query you actually care about**, not a synthetic.
- **Layer MVs / projections**: don't try to make one schema serve every query — multiple schemas for one logical fact is fine.

## 13.11 Must-internalize

- Denormalize. One wide flat table beats joining many narrow ones.
- ORDER BY: low-cardinality high-selectivity columns first, time last (usually).
- Partition by time for TTL/lifecycle; don't over-partition.
- LowCardinality, typed time columns, codecs — non-negotiable.
- Aggregating MVs are the canonical real-time-analytics acceleration.
- ReplacingMergeTree for latest-state; argMax / LIMIT 1 BY for reads.
- Row policies + quotas for multi-tenant.
- CDC via MaterializedPostgreSQL or Debezium-Kafka-ReplacingMT pattern.

---

## Sources

- [Schema design — official docs](https://clickhouse.com/docs/data-modeling/schema-design)
- [Time-series schema (oneuptime)](https://oneuptime.com/blog/post/2026-03-31-clickhouse-design-time-series-schema/view)
- [Denormalization in ClickHouse](https://github.com/ClickHouse/clickhouse-docs/blob/main/docs/data-modeling/denormalization.md)
- [Wide table schema design](https://oneuptime.com/blog/post/2026-03-31-clickhouse-design-wide-table-schema/view)
- [Schema design review checklist](https://oneuptime.com/blog/post/2026-03-31-clickhouse-schema-design-review-checklist/view)
- [windowFunnel](https://clickhouse.com/docs/sql-reference/aggregate-functions/parametric-functions#windowFunnel)
