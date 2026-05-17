# 8 · Native Query Types

Druid has **5 native query types**, each optimized for a different shape of analytical query. SQL queries (the modern way to talk to Druid) get translated by Calcite into one of these. Knowing which is which — and which the SQL planner picks — is the difference between sub-second and 30-second.

## 8.1 The five types

| Native query | SQL pattern | When | Speed |
|--------------|-------------|------|-------|
| **Timeseries** | `SELECT __time, agg() ... GROUP BY __time` (only) | No dimensions, just time-bucketed aggregates | Fastest |
| **TopN** | `SELECT dim, agg() ... GROUP BY dim ORDER BY agg DESC LIMIT N` (single dim) | Top-N per single dimension | Very fast (approximate) |
| **GroupBy v2** | `SELECT dim1, dim2, ..., agg() ... GROUP BY dim1, dim2, ...` (multi-dim) | Multi-dim group by | Slower; uses hash table |
| **Scan** | `SELECT ... FROM events WHERE ... ORDER BY __time LIMIT N` | Raw row fetch | Fast for small N |
| **Search** | `SELECT ... WHERE dim LIKE '%foo%'` | Find values matching a pattern | For UI search bars |

## 8.2 Timeseries query

The fastest path. Streaming aggregate over time intervals, no GROUP BY dimensions.

```sql
SELECT TIME_FLOOR(__time, 'PT1M') AS minute, COUNT(*) AS cnt
FROM events
WHERE __time >= TIMESTAMP '2026-05-17' AND __time < TIMESTAMP '2026-05-18'
GROUP BY 1
ORDER BY 1;
```

Native:
```json
{
  "queryType": "timeseries",
  "dataSource": "events",
  "intervals": ["2026-05-17/2026-05-18"],
  "granularity": "minute",
  "aggregations": [ { "type": "count", "name": "cnt" } ]
}
```

Implementation:
- Each segment's Historical computes per-minute aggregates by streaming through rows.
- No hash table needed.
- Broker simply unions partial timeseries from each segment.

This is the canonical Druid query. Sub-100ms on big data.

## 8.3 TopN query

For "top N per single dimension" with optional approximation.

```sql
SELECT country, SUM(revenue) AS rev
FROM events
WHERE __time >= TIMESTAMP '2026-05-17' AND __time < TIMESTAMP '2026-05-18'
GROUP BY country
ORDER BY rev DESC
LIMIT 10;
```

Native:
```json
{
  "queryType": "topN",
  "dataSource": "events",
  "intervals": ["2026-05-17/2026-05-18"],
  "granularity": "all",
  "dimension": "country",
  "metric": "rev",
  "threshold": 10,
  "aggregations": [ { "type": "doubleSum", "name": "rev", "fieldName": "revenue" } ]
}
```

Implementation:
- Each Historical computes its own top-N per segment.
- Each returns its top N rows.
- Broker merges → final top N.

**Approximation**: each Historical returns `max(threshold, 1000)` rows by default. If your "true top N" requires a rare value, it might miss; the algorithm errs on the side of returning popular values.

For exact top-N, fall back to GroupBy v2 with ORDER BY — slower but exact.

## 8.4 GroupBy v2 query

Multi-dimension group-by. The workhorse for analytical queries.

```sql
SELECT country, event_type, SUM(revenue) AS rev
FROM events
WHERE __time >= TIMESTAMP '2026-05-17'
GROUP BY country, event_type
ORDER BY rev DESC
LIMIT 100;
```

Native:
```json
{
  "queryType": "groupBy",
  "dataSource": "events",
  "intervals": ["2026-05-17/3000"],
  "granularity": "all",
  "dimensions": ["country", "event_type"],
  "aggregations": [ { "type": "doubleSum", "name": "rev", "fieldName": "revenue" } ],
  "limitSpec": { "type": "default", "columns": [ {"dimension": "rev", "direction": "descending"} ], "limit": 100 }
}
```

Implementation:
- Each Historical builds a hash table per segment with (dim-tuple → aggregates).
- Returns the hash table to Broker.
- Broker merges hash tables (more open-addressing) and final-sorts.

The `groupBy` engine uses an **open-addressing hash table** with linear probing. Tuned heavily for cache efficiency.

Memory: bounded by `druid.processing.buffer.sizeBytes` per worker + `druid.query.groupBy.maxOnDiskStorage` for spill.

## 8.5 Scan query

Raw row fetch — no aggregation. Useful for "show me 100 recent events".

```sql
SELECT __time, country, event_type, revenue
FROM events
WHERE __time >= TIMESTAMP '2026-05-17' AND country = 'US'
ORDER BY __time DESC
LIMIT 100;
```

Native:
```json
{
  "queryType": "scan",
  "dataSource": "events",
  "intervals": ["2026-05-17/3000"],
  "filter": { "type": "selector", "dimension": "country", "value": "US" },
  "columns": ["__time", "country", "event_type", "revenue"],
  "order": "descending",
  "limit": 100
}
```

Scan returns rows. The order is by `__time`; arbitrary ordering requires GroupBy or post-processing.

## 8.6 Search query

For UI typeahead — find dimension values matching a substring.

```sql
SELECT DISTINCT country FROM events WHERE country LIKE '%uni%';
```

Native:
```json
{
  "queryType": "search",
  "dataSource": "events",
  "intervals": ["...", "..."],
  "searchDimensions": ["country"],
  "query": { "type": "insensitive_contains", "value": "uni" },
  "limit": 1000
}
```

Searches dictionaries efficiently. Doesn't fetch rows.

## 8.7 SQL → native translation rules

Calcite picks the native query based on the SQL shape. Rules:

| SQL pattern | Native picked |
|-------------|---------------|
| `GROUP BY __time-bucket` only | Timeseries |
| `GROUP BY single-dim ORDER BY agg LIMIT N` | TopN |
| `GROUP BY multi-dim` | GroupBy |
| `SELECT ... ORDER BY __time LIMIT N` | Scan |
| `WHERE dim LIKE '%text%'` standalone | Search (some queries) |

`EXPLAIN PLAN FOR` shows the chosen native query. Always check this when debugging perf.

## 8.8 Common SQL patterns and the underlying query

### Time bucketed counts

```sql
SELECT FLOOR(__time TO MINUTE) AS minute, count(*) AS cnt
FROM events
WHERE __time >= TIMESTAMP '2026-05-17'
GROUP BY 1;
```
→ **Timeseries** ✅

### Top-10 by metric

```sql
SELECT country, SUM(revenue) AS rev
FROM events
GROUP BY country ORDER BY rev DESC LIMIT 10;
```
→ **TopN** (approximate)

For **exact** top-10: add `ORDER BY rev DESC, country ASC LIMIT 10` (multi-key sort hint), or use GroupBy explicitly.

### Multi-dim group by

```sql
SELECT country, event_type, count(*) AS cnt
FROM events
GROUP BY 1, 2;
```
→ **GroupBy**

### Latest 100 events for a user

```sql
SELECT * FROM events WHERE user_id = 12345
ORDER BY __time DESC LIMIT 100;
```
→ **Scan**

### Find country values starting with "U"

```sql
SELECT DISTINCT country FROM events WHERE country LIKE 'U%';
```
→ **Search**

## 8.9 Performance hierarchy

Approximate per-query latency on healthy cluster, 1B-row datasource:

```
Timeseries (no groupBy)     1-10 ms per segment
TopN (single dim)           5-50 ms per segment
Scan (small LIMIT)          5-50 ms per segment
GroupBy multi-dim           50-500 ms per segment
GroupBy with sketches       50-500 ms per segment
GroupBy with high-card dim  500ms+ (potentially slow)
```

Always prefer the cheapest native query that satisfies your need.

## 8.10 GroupBy v2 tuning

If you can't avoid GroupBy:
- Lower `druid.query.groupBy.maxResultsPerSegment` if memory is tight.
- Increase `druid.processing.numThreads` for more parallelism per Historical.
- Set `druid.query.groupBy.maxOnDiskStorage` to enable spill (vs. OOM).
- Avoid GroupBy on high-cardinality dimensions — pre-aggregate via rollup or sketches.

## 8.11 Subqueries and intermediate results

Druid SQL supports subqueries. The Broker executes the inner query, materializes results as an **inline datasource**, then runs the outer query against it.

```sql
SELECT user_id, count(*) AS sessions
FROM (
  SELECT user_id FROM events WHERE event = 'login'
  GROUP BY user_id, FLOOR(__time TO HOUR)
)
GROUP BY user_id;
```

Caveats:
- The intermediate result must fit in Broker memory (bounded by `druid.server.http.maxSubqueryRows`).
- For large intermediates, MSQ is the right tool — it shuffles intermediate data across workers via disk.

## 8.12 Window functions

Newer Druid versions support window functions (`ROW_NUMBER()`, `RANK()`, `LAG`, `LEAD`, etc.). They run on the Broker after the underlying query — so they're limited by Broker memory. For window functions over very large data, MSQ has full support.

```sql
SELECT
  country, __time,
  SUM(revenue) AS rev,
  SUM(SUM(revenue)) OVER (PARTITION BY country ORDER BY __time
                          ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS rev_7day
FROM events
GROUP BY country, __time;
```

## 8.13 Approximate vs exact

| | Approximate | Exact |
|--|-------------|-------|
| Distinct count | `APPROX_COUNT_DISTINCT_DS_HLL` | `COUNT(DISTINCT)` (subquery + GroupBy) |
| Quantile | `APPROX_QUANTILE_DS` | `PERCENTILE_CONT` (newer) or sort + offset |
| Top-N | TopN native query | GroupBy + ORDER BY + LIMIT |
| Cohort | Theta sketch set algebra | Subquery + JOIN |

Default toward approximate for user-facing analytics; pay the exact cost only when correctness demands.

## 8.14 Must-internalize

- **Five native query types**: Timeseries, TopN, GroupBy, Scan, Search. Each optimized differently.
- SQL → native translation; always check `EXPLAIN PLAN FOR` to see what Druid picked.
- Timeseries is fastest; TopN is approximate-by-default; GroupBy is the workhorse but slower.
- Scan for "fetch N rows"; Search for typeahead.
- Window functions and subqueries materialize on the Broker — memory-bound. Use MSQ for very large intermediates.
- Approximate is the default for distinct/quantile — accept ~1% error for 10× speed.

---

## Sources

- [Native queries — official](https://druid.apache.org/docs/latest/querying/)
- [Timeseries queries](https://druid.apache.org/docs/latest/querying/timeseriesquery/)
- [TopN queries](https://druid.apache.org/docs/latest/querying/topnquery/)
- [GroupBy queries (v2)](https://druid.apache.org/docs/latest/querying/groupbyquery/)
- [Scan queries](https://druid.apache.org/docs/latest/querying/scan-query/)
- [Search queries](https://druid.apache.org/docs/latest/querying/searchquery/)
- [Query execution](https://druid.apache.org/docs/latest/querying/query-execution/)
