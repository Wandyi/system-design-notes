# 14 · Query Patterns, Corner Cases, and Alternatives

For each common analytical pattern: the naive approach, the corner cases that break it, and 2–3 alternative implementations with tradeoffs. Mirrors the ClickHouse pack [14](../Clickhouse/14-query-patterns-and-corner-cases.md), tuned to Druid.

## 14.1 Pattern: "Distinct count over big data"

### Approach A: `COUNT(DISTINCT x)`

```sql
SELECT COUNT(DISTINCT user_id) FROM events WHERE __time >= ...;
```

- Druid translates to a GroupBy under the hood — keeps every distinct user_id in memory.
- Memory-bounded; on big data this OOMs or hits `maxResultsPerSegment`.

### Approach B: HLL sketch with `useApproximateCountDistinct = true`

```sql
SET useApproximateCountDistinct = true;
SELECT COUNT(DISTINCT user_id) FROM events WHERE __time >= ...;
```

- Druid silently uses HLL.
- ~1% error.
- Constant memory.

### Approach C: Pre-aggregate as HLL sketch metric

```sql
SELECT APPROX_COUNT_DISTINCT_DS_HLL(distinct_users_hll) FROM rolled_up_events WHERE __time >= ...;
```

- No raw `user_id` stored at all.
- Sub-millisecond per segment.
- The right answer for any high-cardinality distinct count in production.

### Selection rubric

- Need exact count + small data → Approach A.
- Need fast count, OK with ~1% error → Approach B (post-hoc) or **Approach C** (pre-aggregated, fastest).

## 14.2 Pattern: "Top N per group"

E.g., top 10 events per country.

### Approach A: TopN (single dim)

```sql
SELECT country, SUM(revenue) AS rev
FROM events GROUP BY country ORDER BY rev DESC LIMIT 10;
```

- Native TopN.
- Approximate by default (per-segment thresholding); may miss "tail" entries.

### Approach B: GroupBy + ORDER BY + LIMIT (exact)

```sql
SELECT country, SUM(revenue) AS rev
FROM events GROUP BY country
ORDER BY rev DESC, country ASC LIMIT 10;  -- multi-key sort hint → GroupBy
```

- Forces GroupBy path; exact.
- Slower; full hash table per segment.

### Approach C: Window function `ROW_NUMBER`

```sql
SELECT country, event_name, rev FROM (
  SELECT country, event_name, SUM(revenue) AS rev,
         ROW_NUMBER() OVER (PARTITION BY country ORDER BY SUM(revenue) DESC) AS rn
  FROM events GROUP BY country, event_name
)
WHERE rn <= 10;
```

- Top-N **per group**; the only one that does this in pure SQL.
- Materializes on Broker; memory-bounded.

### Selection rubric

- Top-N single dim, approximate OK → TopN.
- Top-N single dim, exact required → GroupBy.
- Top-N per group → window function (or MSQ for large data).

## 14.3 Pattern: "Latest value per entity"

E.g., "latest known status per user".

### Approach A: `argMax` via `LAST_VALUE` aggregation

Druid has `LATEST_BY(value, timeCol)` and `LATEST(value)`:

```sql
SELECT user_id, LATEST(status) AS current_status
FROM events
GROUP BY user_id;
```

### Approach B: subquery + `ROW_NUMBER`

```sql
SELECT user_id, status FROM (
  SELECT user_id, status,
         ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY __time DESC) AS rn
  FROM events
)
WHERE rn = 1;
```

- Window function; materializes on Broker.

### Approach C: Maintain a separate `users_current` datasource

Use MSQ to compute and REPLACE periodically:

```sql
REPLACE INTO users_current OVERWRITE ALL
SELECT user_id, LATEST(status) AS status, MAX(__time) AS __time
FROM events
GROUP BY user_id
PARTITIONED BY ALL;
```

Query: `SELECT * FROM users_current WHERE user_id = 42;` — direct lookup.

### Selection rubric

- One-off ad-hoc → `LATEST()` or window.
- Frequent reads → maintain separate datasource.

Note: Druid's `LATEST` is less ergonomic than CH's `argMax` because it requires a `timeCol` (auto-`__time` for default).

## 14.4 Pattern: "Funnel"

E.g., view → cart → buy within 1 hour.

### Approach A: theta-sketch intersection

Maintain per-event-name theta sketches of `user_id`:

```sql
WITH s AS (
  SELECT event_name, DS_THETA_UNION(user_theta) AS sk
  FROM rolled_up
  WHERE __time >= TIMESTAMP '2026-05-10' AND __time < TIMESTAMP '2026-05-17'
  GROUP BY 1
)
SELECT THETA_SKETCH_ESTIMATE(THETA_SKETCH_INTERSECT(v.sk, c.sk, b.sk)) AS converted
FROM (SELECT sk FROM s WHERE event_name = 'view') v,
     (SELECT sk FROM s WHERE event_name = 'cart') c,
     (SELECT sk FROM s WHERE event_name = 'buy') b;
```

- Approximate; very fast on rolled-up data.
- Doesn't enforce *order* of events — anyone who did all three at some point.

### Approach B: subquery per step + JOIN by user_id

```sql
SELECT count(distinct user_id) FROM (
  SELECT user_id FROM events WHERE event_name='view' AND __time >= ts1 AND __time < ts2
  INTERSECT
  SELECT user_id FROM events WHERE event_name='cart' AND __time >= ts1 AND __time < ts2
  INTERSECT
  SELECT user_id FROM events WHERE event_name='buy' AND __time >= ts1 AND __time < ts2
);
```

- Exact, in-order requires further filtering.
- Slow; raw distinct counts.

### Approach C: timestamp-ordered subquery + window

Build a per-user array of events, then check sequence:

```sql
SELECT count(*) FROM (
  SELECT user_id, ARRAY_AGG(event_name ORDER BY __time) AS seq
  FROM events
  WHERE event_name IN ('view','cart','buy') AND __time >= ts1 AND __time < ts2
  GROUP BY user_id
)
WHERE ARRAY_CONTAINS(seq, 'view')
  AND ARRAY_CONTAINS(seq, 'cart')
  AND ARRAY_CONTAINS(seq, 'buy')
  AND ARRAY_OFFSET(seq, 'view') < ARRAY_OFFSET(seq, 'cart')
  AND ARRAY_OFFSET(seq, 'cart') < ARRAY_OFFSET(seq, 'buy');
```

- Closer to ClickHouse's `windowFunnel`.
- Memory-heavy.

### Selection rubric

- Order-independent funnel → theta sketch intersection.
- Order-strict funnel → ARRAY_AGG approach.
- For complex sequence patterns → MSQ or pre-compute per-user funnel state.

(Druid doesn't have a `windowFunnel` equivalent built-in — this is a place where ClickHouse has a sharper tool.)

## 14.5 Pattern: "Time bucket with gap fill"

### Approach A: GROUP BY truncated time

```sql
SELECT TIME_FLOOR(__time, 'PT1M') AS minute, count(*) FROM events GROUP BY 1;
```

Missing minutes don't appear.

### Approach B: External `series_generate`-style gap fill

Druid doesn't have `WITH FILL` (ClickHouse) or `generate_series` natively. Workaround:
- Generate the time series in your application layer (BI tool) and outer-join.
- Or use the experimental `FILL_VALUE` or post-processing in Pivot / Superset.

This is one of Druid's ergonomic gaps vs CH.

## 14.6 Pattern: "Pivot / transpose"

### Approach A: filtered aggregates

```sql
SELECT country,
  SUM(events) FILTER (WHERE event_name = 'view') AS views,
  SUM(events) FILTER (WHERE event_name = 'click') AS clicks,
  SUM(events) FILTER (WHERE event_name = 'buy') AS buys
FROM events
WHERE __time >= ...
GROUP BY country;
```

Translates to native filtered aggregators — single scan, multiple metrics.

### Approach B: GROUPING SETS

```sql
SELECT country, event_name, SUM(events)
FROM events
GROUP BY GROUPING SETS ((country, event_name), (country), ());
```

Computes multiple grain rollups in one query.

## 14.7 Pattern: "Set algebra — A but not B"

Theta sketches shine:

```sql
WITH a AS (
  SELECT DS_THETA(user_id) AS sk FROM events
  WHERE event_name='view' AND __time >= ...
),
b AS (
  SELECT DS_THETA(user_id) AS sk FROM events
  WHERE event_name='buy' AND __time >= ...
)
SELECT THETA_SKETCH_ESTIMATE(THETA_SKETCH_NOT(a.sk, b.sk)) AS viewed_but_did_not_buy
FROM a, b;
```

- `THETA_SKETCH_NOT(a, b)` = `a` minus `b` (estimated cardinality of A ∖ B).
- Sub-second on rolled-up data.

## 14.8 Pattern: "Sessionization"

Druid doesn't have first-class session detection. Approaches:

### Approach A: Pre-compute session_id at ingest

Best. Compute session_id upstream (in your application or via Flink), include as a dimension. Group by session_id for session-level queries.

### Approach B: MSQ post-processing

```sql
REPLACE INTO sessions OVERWRITE ALL
SELECT
  user_id,
  ARRAY_AGG(__time) AS event_times,
  MIN(__time) AS session_start,
  MAX(__time) AS session_end
FROM events
GROUP BY user_id
HAVING MAX(__time) - MIN(__time) <= 30 * 60 * 1000   -- 30 min
PARTITIONED BY ALL;
```

Crude; doesn't split sessions properly without windowing. Real sessionization needs MSQ with proper window logic or upstream computation.

## 14.9 Pattern: "Joining a small dimension"

### Approach A: Lookup

```sql
SELECT country_code, LOOKUP(country_code, 'country_lookup') AS name, count(*)
FROM events
GROUP BY 1, 2;
```

### Approach B: Broadcast hash join

```sql
SELECT e.country_code, l.name, count(*)
FROM events e LEFT JOIN countries l ON e.country_code = l.code
GROUP BY 1, 2;
```

### Approach C: Denormalize at ingest

Pre-compute the country_name at ingest time via lookup.

### Selection rubric

- < 10K rows → Lookup.
- 10K-1M rows → broadcast join (or lookup if heap allows).
- Always used in this shape → denormalize.

## 14.10 Pattern: "Joining two big tables"

Druid's answer: **don't**.

### Approach A: Pre-join via MSQ

```sql
INSERT INTO events_with_user_dim
SELECT e.*, u.signup_date, u.country
FROM events e LEFT JOIN users u ON e.user_id = u.user_id
PARTITIONED BY DAY;
```

### Approach B: Theta sketch set algebra

If you only need cohort counts, not row joins, theta sketches replace the join entirely.

### Approach C: Last resort — broadcast with size limits

If the right side fits in Broker memory, it works. Otherwise the query fails.

## 14.11 Pattern: "Approximate quantile (p99) over time"

### Approach A: Pre-aggregate as quantile sketch

```sql
INSERT INTO latency_min
SELECT TIME_FLOOR(__time, 'PT1M') AS __time,
       service,
       DS_QUANTILES_SKETCH(latency_ms, 128) AS sketch
FROM raw_events
GROUP BY 1, 2;
```

Query:
```sql
SELECT __time, service,
       APPROX_QUANTILE_DS(sketch, 0.99) AS p99
FROM latency_min
WHERE __time >= ...
GROUP BY __time, service;
```

### Approach B: ad-hoc on raw

```sql
SELECT __time_bucket, APPROX_QUANTILE_DS(latency_ms, 0.99) FROM raw_events ...;
```

Works but slower (sketch built per query).

## 14.12 Pattern: "Compare week-over-week"

### Approach A: subqueries with subtraction

```sql
SELECT
  (SELECT SUM(events) FROM ev WHERE __time >= TIMESTAMP_OF_LAST_WEEK_START)  AS this_wk,
  (SELECT SUM(events) FROM ev WHERE __time >= TIMESTAMP_OF_PREV_WEEK_START
                                AND __time < TIMESTAMP_OF_LAST_WEEK_START)   AS last_wk;
```

### Approach B: CASE-when buckets

```sql
SELECT
  SUM(events) FILTER (WHERE __time >= '2026-05-10' AND __time < '2026-05-17') AS this_wk,
  SUM(events) FILTER (WHERE __time >= '2026-05-03' AND __time < '2026-05-10') AS last_wk
FROM events
WHERE __time >= '2026-05-03' AND __time < '2026-05-17';
```

Single scan, two metrics. Faster.

## 14.13 Pattern: "Live ingest visibility"

Druid queries see in-flight ingest tasks' data. So:

```sql
SELECT COUNT(*) FROM events WHERE __time >= CURRENT_TIMESTAMP - INTERVAL '1' MINUTE;
```

Returns data ingested seconds ago. The Broker fans out to Historicals + MiddleManager Peons.

In ClickHouse: similar — data is queryable right after insert. Druid emphasizes this as a feature (streaming with sub-second freshness).

## 14.14 Pattern: "Re-process bad data"

Druid: **MSQ REPLACE**.

```sql
REPLACE INTO events
  OVERWRITE WHERE __time >= TIMESTAMP '2026-05-17'
                AND __time <  TIMESTAMP '2026-05-18'
SELECT ... FROM TABLE(EXTERN(... corrected data ...))
PARTITIONED BY DAY;
```

Old segments overshadowed atomically. Queries see new data immediately.

In ClickHouse: ALTER UPDATE (heavy mutation) or `DROP PARTITION` + re-insert.

## 14.15 Pattern: "Detect anomalies in metrics"

### Approach A: window function over recent N points

```sql
SELECT __time, service, value,
       value - AVG(value) OVER (PARTITION BY service ORDER BY __time
                                ROWS BETWEEN 60 PRECEDING AND CURRENT ROW) AS deviation
FROM metrics
WHERE __time >= ... AND service = 'api';
```

### Approach B: pre-compute moving stats via MSQ

Roll up rolling-window aggregates into a separate datasource.

## 14.16 Must-internalize

- Most patterns have 2-3 implementations; pick by data size and frequency.
- HLL/Theta sketches are the right answer for distinct/cohort/set algebra.
- TopN is approximate by default; force GroupBy for exact.
- LATEST_BY / window functions for latest-value queries.
- No first-class WITH FILL or windowFunnel — work around with subqueries / MSQ.
- Lookups + denormalize beat real JOINs.
- REPLACE for any data fix.
- Window functions live on Broker — memory-bound; MSQ for big shuffles.

---

## Sources

- [SQL functions](https://druid.apache.org/docs/latest/querying/sql-functions/)
- [Approximations](https://druid.apache.org/docs/latest/querying/sql-aggregations/)
- [Quantiles cookbook](https://blog.hellmar-becker.de/2022/03/20/druid-data-cookbook-quantiles-in-druid-with-datasketches/)
- [Cohorts cookbook](https://blog.hellmar-becker.de/2022/06/05/druid-data-cookbook-counting-unique-visitors-for-overlapping-segments/)
