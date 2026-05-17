# 13 · Schema Design Patterns — Worked Examples

The schema is the architecture in Druid even more than in ClickHouse — because rollup decisions are baked at ingest and (mostly) can't be reversed. This file walks through eight workload patterns with concrete ingestion specs (or MSQ DDL), the reasoning, and the corner cases.

For each:
1. Workload shape.
2. Datasource definition.
3. Why this schema (rollup decisions, dim/metric split, granularities).
4. Common queries.
5. Variants for scale.
6. Anti-patterns.

## 13.1 General principles

1. **Pick `queryGranularity` for the coarsest precision your dashboards need**. MINUTE is the typical sweet spot.
2. **Drop high-cardinality columns or replace with sketches.** UUID → HLL sketch. URL path → sketch the path's classification, not the path itself.
3. **Time partition (`segmentGranularity`)** matches your query span. DAY for last-month dashboards; HOUR for last-week with finer time queries.
4. **CLUSTERED BY** = your secondary sort order; pick filtered-on dimensions, low-cardinality first.
5. **Lookups** for small dimensions; **denormalize** for repeated joins.
6. **Auto-compaction always** in production.

---

## 13.2 Pattern 1 — Event tracking (product analytics, the canonical)

**Workload**: Each user action emits an event. Dashboards: events/day, distinct users, funnels, retention.

```sql
INSERT INTO product_events
SELECT
  TIME_FLOOR(TIME_PARSE(ts), 'PT1M') AS __time,
  project_id,
  event_name,
  country,
  browser,
  os,
  device_type,
  -- HLL sketch instead of raw user_id
  APPROX_COUNT_DISTINCT_DS_HLL(user_id) AS distinct_users,
  COUNT(*) AS events,
  SUM(revenue) AS revenue
FROM TABLE(EXTERN(...))   -- batch backfill source
GROUP BY 1, 2, 3, 4, 5, 6, 7
PARTITIONED BY DAY
CLUSTERED BY project_id, event_name, country;
```

Streaming equivalent uses a Kafka supervisor with same rollup config.

### Why this schema

- `project_id` first in CLUSTERED BY → multi-tenant; most queries are scoped.
- `event_name` second → analyses are usually for a specific event.
- `country` third → frequent filter.
- `user_id` is replaced by HLL sketch.
- `queryGranularity = MINUTE` → finest grain dashboards need.
- `segmentGranularity = DAY` → ~one segment per day per cluster partition.

### Queries

```sql
-- events per minute, all projects
SELECT __time, SUM(events) AS cnt
FROM product_events
WHERE __time >= CURRENT_TIMESTAMP - INTERVAL '1' HOUR
GROUP BY __time;
-- → Timeseries

-- top countries by revenue, last day, project 42
SELECT country, SUM(revenue) AS rev
FROM product_events
WHERE project_id = 42 AND __time >= CURRENT_TIMESTAMP - INTERVAL '1' DAY
GROUP BY country
ORDER BY rev DESC LIMIT 10;
-- → TopN

-- distinct users for an event last 7 days
SELECT APPROX_COUNT_DISTINCT_DS_HLL(distinct_users)
FROM product_events
WHERE project_id = 42 AND event_name = 'checkout'
  AND __time >= CURRENT_TIMESTAMP - INTERVAL '7' DAY;
```

### Anti-patterns

- Including `user_id` as a dimension *and* as an HLL sketch → wasted bytes.
- `queryGranularity = NONE` when MINUTE would do → 60× more rows.
- One datasource per project → mass datasource sprawl; coordinator overhead.

### Variants

- For ultra-high-cardinality projects: shard by project (separate datasources for the largest); for everyone else use shared.
- For mid-cardinality `session_id` queries: keep `session_id` as dimension with `createBitmapIndex: false`.

---

## 13.3 Pattern 2 — Time-series metrics (Prometheus-like)

**Workload**: Numeric metrics with labels; rate/avg/p95/p99 over time intervals.

```sql
INSERT INTO metrics
SELECT
  TIME_PARSE(ts) AS __time,
  metric_name,
  service,
  host,
  AVG(value) AS avg_value,
  MAX(value) AS max_value,
  MIN(value) AS min_value,
  DS_QUANTILES_SKETCH(value, 128) AS quantile_sketch
FROM TABLE(EXTERN(...))
GROUP BY 1, 2, 3, 4
PARTITIONED BY HOUR
CLUSTERED BY metric_name, service;
```

### Why

- `metric_name` first → most queries pick a single metric.
- HOUR segments because metric data is densely sampled.
- Quantile sketch lets us compute p99 on rolled-up rows.
- Avg/max/min pre-computed for fast time-series queries.

### Queries

```sql
-- p99 latency over time, last hour, for one service
SELECT __time,
       APPROX_QUANTILE_DS(quantile_sketch, 0.99) AS p99
FROM metrics
WHERE metric_name = 'request_latency'
  AND service = 'api'
  AND __time >= CURRENT_TIMESTAMP - INTERVAL '1' HOUR
GROUP BY __time;
```

### Variants

- Cascade datasources: `metrics` (raw 1m) → `metrics_5m` (rolled-up) → `metrics_1h` for long-range queries.
- Apply different `dropByPeriod` rules per granularity.

### Anti-patterns

- Storing high-cardinality `host` for million-host fleets → use a hash or drop it.
- Per-host quantile sketch when only service-level matters.

---

## 13.4 Pattern 3 — Logs / observability

**Workload**: Log lines; queries are "count by service/severity over time", "search for a substring", "trace lookup by ID".

```sql
INSERT INTO logs
SELECT
  TIME_PARSE(ts) AS __time,
  service,
  severity,
  host,
  trace_id,
  body,
  attrs  -- JSON column
FROM TABLE(EXTERN(...))
PARTITIONED BY DAY
CLUSTERED BY service, severity;
```

`rollup: false` — every line distinct.

### Why

- Logs aren't rolled up — preserve each line.
- `service` + `severity` for fast filtering.
- `trace_id` indexed automatically (bitmap), so trace lookup is O(1).
- `body` is a String dimension → searchable via LIKE (slow on full body, OK for keyword search).
- `attrs` JSON column for arbitrary structured fields.

### Queries

```sql
-- count by service/severity per minute
SELECT __time, service, severity, count(*)
FROM logs
WHERE __time >= CURRENT_TIMESTAMP - INTERVAL '1' HOUR
GROUP BY 1, 2, 3;
-- → GroupBy

-- trace lookup
SELECT * FROM logs WHERE trace_id = 'abc123' AND __time >= TIMESTAMP '...';
-- → Scan

-- search
SELECT * FROM logs WHERE body LIKE '%timeout%' AND service = 'api'
  AND __time >= CURRENT_TIMESTAMP - INTERVAL '1' HOUR LIMIT 100;
-- → Scan, body is column-scanned but service+time prune fast
```

### Variants

- For full-text search at scale: pair Druid with OpenSearch for text-heavy queries (Druid for aggregates).
- Or use the experimental `text` index in newer Druid versions for inverted full-text.

---

## 13.5 Pattern 4 — Ad analytics (impressions/clicks/conversions)

**Workload**: Massive append; multi-advertiser; report aggregates per advertiser/campaign per minute.

```sql
INSERT INTO impressions
SELECT
  TIME_FLOOR(TIME_PARSE(ts), 'PT1M') AS __time,
  advertiser_id,
  campaign_id,
  creative_id,
  geo,
  device,
  COUNT(*) AS imps,
  SUM(cost_cents) AS cost,
  SUM(revenue_cents) AS revenue,
  APPROX_COUNT_DISTINCT_DS_HLL(user_hash) AS users
FROM TABLE(EXTERN(...))
GROUP BY 1, 2, 3, 4, 5, 6
PARTITIONED BY DAY
CLUSTERED BY advertiser_id, campaign_id;
```

### Queries

```sql
-- per-campaign report
SELECT campaign_id,
       SUM(imps), SUM(cost)/100.0 AS cost_usd,
       SUM(revenue)/100.0 AS rev_usd,
       APPROX_COUNT_DISTINCT_DS_HLL(users) AS unique_users
FROM impressions
WHERE advertiser_id = 42 AND __time >= CURRENT_TIMESTAMP - INTERVAL '1' DAY
GROUP BY campaign_id
ORDER BY rev_usd DESC LIMIT 100;
```

### Variants

- Clicks and conversions in separate datasources, joined via theta sketch on `user_hash`.
- Per-advertiser denormalization (advertiser_name via lookup).

---

## 13.6 Pattern 5 — Multi-tenant SaaS analytics

**Workload**: Many tenants; each sees their own dashboards; strict isolation.

### Option A: One datasource, `tenant_id` first dimension

```sql
INSERT INTO tenant_events
SELECT
  TIME_FLOOR(TIME_PARSE(ts), 'PT1M') AS __time,
  tenant_id,
  event_name,
  user_id_hll: APPROX_COUNT_DISTINCT_DS_HLL(user_id) AS distinct_users,
  COUNT(*) AS events
FROM TABLE(EXTERN(...))
GROUP BY 1, 2, 3
PARTITIONED BY DAY
CLUSTERED BY tenant_id, event_name;
```

Authorization layer enforces `tenant_id = ?` on every query.

### Option B: One datasource per tenant

- Cleaner isolation at the storage level.
- Per-tenant retention (different drop rules).
- Per-tenant compaction settings.
- Coordinator overhead grows with datasource count — manageable up to ~thousands.

### Selection rubric

- < 1000 tenants → Option B works.
- > 1000 or fast-growing tenant count → Option A.
- "Whale" tenants → dedicated datasource each (Option B subset), shared for the rest.

### Anti-patterns

- Putting tenant_id last in CLUSTERED BY → no co-location.
- Trusting client-side `WHERE tenant_id = ?` without server-side enforcement.

---

## 13.7 Pattern 6 — Cohort / retention

**Workload**: "Which users did A this week and B next week?"

```sql
INSERT INTO user_actions
SELECT
  TIME_FLOOR(__time, 'P1D') AS __time,
  event_name,
  DS_THETA(user_id, 16384) AS users_theta
FROM raw_events
GROUP BY 1, 2
PARTITIONED BY MONTH
CLUSTERED BY event_name;
```

### Query (cohort intersection via theta sketches)

```sql
WITH viewers AS (
  SELECT DS_THETA_UNION(users_theta) AS sk
  FROM user_actions
  WHERE event_name = 'view'
    AND __time >= TIMESTAMP '2026-05-10' AND __time < TIMESTAMP '2026-05-17'
),
buyers AS (
  SELECT DS_THETA_UNION(users_theta) AS sk
  FROM user_actions
  WHERE event_name = 'purchase'
    AND __time >= TIMESTAMP '2026-05-17' AND __time < TIMESTAMP '2026-05-24'
)
SELECT THETA_SKETCH_ESTIMATE(THETA_SKETCH_INTERSECT(v.sk, b.sk))
FROM viewers v, buyers b;
```

### Why theta sketches

Exact "users who did A and B" requires per-user rows. With sketches: rolled-up data + set algebra at query time. Sub-second on years of data.

---

## 13.8 Pattern 7 — Live + historical hybrid

**Workload**: Last hour from streaming, last 13 months historical (cold tier).

```
Kafka supervisor → events (DAY-partitioned, 7-day hot tier)
                                ↓ as data ages
                          cold tier (1 replica, 12 months)
                                ↓ at 13 months
                          dropForever
```

Load rules:
```json
[
  { "type": "loadByPeriod", "period": "P7D",
    "tieredReplicants": { "hot": 2 } },
  { "type": "loadByPeriod", "period": "P13M",
    "tieredReplicants": { "cold": 1 } },
  { "type": "dropForever" }
]
```

Auto-compaction continuously compacts streaming-produced segments to 700 MB. Auto-kill removes deep-storage segments after they've been unused for 30 days.

---

## 13.9 Pattern 8 — Wide event with JSON catch-all

**Workload**: Heterogeneous event schemas; product engineers add new fields without notice.

```sql
INSERT INTO events_wide
SELECT
  TIME_FLOOR(TIME_PARSE(ts), 'PT1M') AS __time,
  tenant_id,
  event_name,
  country,
  -- explicit common dimensions
  device, browser, os,
  -- JSON catch-all for everything else
  props,  -- JSON column
  COUNT(*) AS events
FROM TABLE(EXTERN(...))
GROUP BY 1, 2, 3, 4, 5, 6, 7, 8
PARTITIONED BY DAY
CLUSTERED BY tenant_id, event_name;
```

Queries can filter on JSON paths via `JSON_VALUE(props, '$.path')`. Druid auto-discovers paths into subcolumns with indexes.

Tradeoff: rollup ratio is hurt when `props` varies row-to-row. Acceptable for the flexibility.

---

## 13.10 Cross-cutting recommendations

- **Test rollup ratio** post-ingest: `SELECT total_input_rows / total_output_rows`.
- **Test compression**: per-column bytes from segment metadata.
- **Test query pruning**: `EXPLAIN PLAN` to verify segment count for the time range.
- **Layer datasources**: `raw_events` → `daily_summary` for long-range queries.

## 13.11 Must-internalize

- Decide rollup at ingest; you can't un-decide cheaply.
- HLL/theta sketches replace high-cardinality dims for distinct-count.
- CLUSTERED BY = secondary sort, low-cardinality high-selectivity first.
- Time partition matches query span; auto-compaction prevents fragmentation.
- Multi-tenant: tenant_id leading + auth + per-tenant rules.
- JSON columns for unknown / evolving schemas.
- For cohort analysis: theta sketches + set algebra.

---

## Sources

- [Schema design tips](https://druid.apache.org/docs/latest/ingestion/schema-design/)
- [MSQ examples](https://druid.apache.org/docs/latest/multi-stage-query/examples/)
- [DataSketches cookbook posts (Hellmar Becker)](https://blog.hellmar-becker.de/categories/druid/)
- [Theta sketches for cohorts](https://blog.hellmar-becker.de/2022/06/05/druid-data-cookbook-counting-unique-visitors-for-overlapping-segments/)
