# 9 · Druid SQL

Druid SQL is what most queries arrive as. Calcite-based planner; produces native queries underneath. This file covers what's supported, what isn't, and the canonical SQL patterns.

## 9.1 The SQL stack

```
User SQL
   │
   ▼  parsed + analyzed by Calcite
Logical plan
   │
   ▼  rules pick native query type
Native query
   │
   ▼
Native execution path (Broker → Historicals → merge → return)
```

## 9.2 What's supported

The standard analytical surface:
- `SELECT ... FROM ... WHERE ... GROUP BY ... HAVING ... ORDER BY ... LIMIT`.
- All standard aggregates (`COUNT`, `SUM`, `MIN`, `MAX`, `AVG`).
- Approximate aggregates (`APPROX_COUNT_DISTINCT_DS_HLL`, `APPROX_QUANTILE_DS`, etc.).
- Window functions (`ROW_NUMBER`, `RANK`, `LAG`, `LEAD`, sliding windows).
- CTEs (`WITH`).
- Subqueries (in FROM, in WHERE, scalar).
- Joins (broadcast hash; left/inner/cross; no shuffle).
- `UNION ALL`, `EXCEPT`, `INTERSECT` (limited).
- `CASE WHEN`.
- `IN` lists, `BETWEEN`, `LIKE`, `NOT LIKE`.
- `DISTINCT` on subqueries.
- Time functions: `TIME_FLOOR`, `TIME_PARSE`, `TIME_FORMAT`, `TIME_SHIFT`, `EXTRACT`.
- String functions: `LIKE`, `REGEXP_LIKE`, `SUBSTRING`, `LOWER`, `UPPER`, `CONCAT`, etc.
- Math, conditional, lookup functions.
- Array functions for multi-value dimensions.
- JSON functions: `JSON_VALUE`, `JSON_QUERY`, `JSON_OBJECT`.
- MSQ-specific: `INSERT`, `REPLACE`, `EXTERN`, `PARTITIONED BY`, `CLUSTERED BY`.

## 9.3 What's not supported (or differs)

- **UPDATE / DELETE** — no row-level mutation. Use REPLACE for interval-level reprocessing.
- **Transactions** — no multi-statement transactions. Each query is atomic on its own.
- **FULL OUTER JOIN** — limited support; usually requires subquery.
- **Recursive CTEs** — not supported.
- **Stored procedures, user-defined functions** — limited (some UDAFs via extensions).
- **Complex correlated subqueries** — may force materialization and run slowly.

## 9.4 Time functions — every Druid query touches them

Druid stores `__time` as a long (millis since epoch). Common functions:

```sql
TIME_FLOOR(__time, 'PT1M')           -- truncate to minute boundary
TIME_FLOOR(__time, 'P1D')            -- truncate to day
TIME_FLOOR(__time, 'P1D', 'UTC')     -- truncate to day at a specific TZ
TIME_PARSE('2026-05-17', 'yyyy-MM-dd')
TIME_FORMAT(__time, 'yyyy-MM-dd')
TIME_SHIFT(__time, 'PT1H', 1)        -- shift forward by 1 hour
EXTRACT(HOUR FROM __time)
```

ISO 8601 duration format throughout (`PT1H` = 1 hour, `P1D` = 1 day, `P1M` = 1 month).

## 9.5 Filtering for time always

A query without a time filter scans **every** segment of the datasource — bad. Always include `WHERE __time BETWEEN ... AND ...` or `WHERE __time >= ...`.

Druid uses the time filter for **segment pruning** before anything else.

```sql
-- bad: full scan of every segment ever ingested
SELECT count(*) FROM events;

-- good: only relevant segments
SELECT count(*) FROM events WHERE __time >= CURRENT_TIMESTAMP - INTERVAL '7' DAY;
```

A standard guardrail: configure `druid.sql.planner.requireTimeCondition = true` to refuse queries without a time filter.

## 9.6 EXPLAIN PLAN

```sql
EXPLAIN PLAN FOR
SELECT country, sum(revenue) FROM events
WHERE __time >= TIMESTAMP '2026-05-17'
GROUP BY country
ORDER BY 2 DESC LIMIT 10;
```

Returns the chosen native query + the segment intervals it'll touch. Use to verify:
- The right native query type was picked.
- Time pruning narrowed segments.
- The right indexes / dimensions are used.

## 9.7 Joins in SQL

```sql
SELECT e.country, l.continent, SUM(e.revenue) AS rev
FROM events e
LEFT JOIN countries_lookup l ON e.country = l.country_code
WHERE e.__time >= TIMESTAMP '2026-05-17'
GROUP BY 1, 2;
```

How this runs:
- `countries_lookup` is a **lookup** (preloaded in Broker memory) → free join, the lookup is consulted inline per row.
- If it weren't a lookup, the Broker would materialize the small side as an inline datasource and broadcast-hash-join.

See [10](10-joins-lookups-subqueries.md) for depth.

## 9.8 IN-filter optimization

A simple `IN` against another query gets specially handled:

```sql
SELECT country, count(*) FROM events
WHERE user_id IN (
  SELECT user_id FROM premium_users
);
```

The subquery materializes into an in-memory set; the outer scan uses bitmap filtering for `user_id IN set` if the bitmap exists, or scans otherwise.

## 9.9 Filtered aggregates

```sql
SELECT
  country,
  SUM(revenue) FILTER (WHERE event = 'purchase') AS purchase_rev,
  COUNT(*) FILTER (WHERE event = 'view') AS views
FROM events
GROUP BY country;
```

Translates to native `filtered` aggregators. Single scan, multiple aggregates with different filters. Very efficient.

## 9.10 GROUPING SETS / ROLLUP / CUBE

```sql
SELECT country, event_type, SUM(revenue)
FROM events
GROUP BY ROLLUP(country, event_type);

-- equivalent to:
GROUP BY GROUPING SETS (
  (country, event_type),
  (country),
  ()
);
```

Computes all sub-aggregates in one query. Faster than running each separately.

## 9.11 Array and multi-value functions

```sql
SELECT MV_TO_ARRAY(tags) AS tag, count(*)
FROM events
GROUP BY 1;
```

`MV_TO_ARRAY` (or `MV_OFFSET`) explodes multi-value dimensions. `ARRAY_CONTAINS`, `ARRAY_OVERLAP` for set-membership.

## 9.12 JSON functions

```sql
SELECT JSON_VALUE(props, '$.browser')                      AS browser,
       JSON_VALUE(props, '$.os' RETURNING VARCHAR)         AS os,
       JSON_QUERY(props, '$.utm')                          AS utm_obj,
       JSON_OBJECT('country':country, 'event':event_type)  AS payload
FROM events;
```

Path syntax is JSONPath. The Calcite planner can push these down to the JSON column's typed subcolumns for fast access.

## 9.13 Cross-datasource queries

A query can hit multiple datasources:

```sql
SELECT e.country, sum(e.revenue), sum(p.payouts)
FROM events e
LEFT JOIN payouts p ON e.country = p.country AND e.__time = p.__time
WHERE e.__time >= TIMESTAMP '2026-05-17'
GROUP BY 1;
```

Broker handles both queries, joins on the merged result. Limited by Broker memory for join broadcast.

## 9.14 Query context settings

A `SET` (or `?queryContext`) can tune execution per-query:

```
sqlQueryId               -- give your query an ID for tracing
priority                 -- prioritize against other queries
timeout                  -- ms before timeout
useApproximateCountDistinct  -- toggle automatic approximation for COUNT(DISTINCT)
useApproximateTopN       -- toggle TopN's approximation
maxNumericInFilters      -- limit IN-list size
maxScatterGatherBytes    -- limit per-segment bytes returned to Broker
```

`useApproximateCountDistinct = true` means `COUNT(DISTINCT x)` automatically becomes `APPROX_COUNT_DISTINCT_DS_HLL`. Many production setups enable this globally.

## 9.15 Result formats

Druid SQL endpoints can return JSON, CSV, TSV, Arrow, line-delimited JSON. Choose what fits your client.

## 9.16 Authentication & authorization in SQL

`GRANT` syntax is supported (with the appropriate authorizer extension):

```sql
GRANT READ ON DATASOURCE events TO ROLE analyst;
GRANT WRITE ON DATASOURCE events TO ROLE ingester;
```

Combined with row-level filtering via lookups for multi-tenancy.

## 9.17 Differences from ClickHouse SQL

| | ClickHouse | Druid |
|--|------------|-------|
| ANSI compliance | High | Higher (Calcite-based) |
| FROM table | Optional in simple queries | Required |
| Implicit JOIN ALL semantics | Yes (`JOIN`/`ALL` returns all matches) | Standard ANSI |
| Time functions | `toStartOfMinute(ts)`, `toDate(ts)`, etc. | `TIME_FLOOR(__time, 'PT1M')`, `FLOOR(__time TO MINUTE)` |
| Approximate distinct | `uniq`, `uniqCombined`, `uniqHLL12` | `APPROX_COUNT_DISTINCT_DS_HLL` (via sketch) |
| LIMIT BY | Yes (`LIMIT N BY dim`) | Window function + filter |
| Window functions | Supported | Supported |
| Materialized views in SQL | `CREATE MATERIALIZED VIEW` | Not a SQL construct; ingestion specs / MSQ instead |
| UPDATE/DELETE | Mutations + lightweight | None — REPLACE per interval |
| Transactions | None | None |
| CTEs | Supported | Supported |

## 9.18 Must-internalize

- Calcite-based planner; query → native one of (timeseries, topN, groupBy, scan, search).
- Always include a time filter — segment pruning depends on it.
- `EXPLAIN PLAN FOR` to verify the plan.
- Filtered aggregates + GROUPING SETS for efficient multi-metric queries.
- JSON / array / MV functions for semi-structured queries.
- No UPDATE/DELETE; use REPLACE INTO for fixes.
- Approximate-by-default is encouraged for user-facing dashboards.

---

## Sources

- [Druid SQL — official](https://druid.apache.org/docs/latest/querying/sql/)
- [SQL functions](https://druid.apache.org/docs/latest/querying/sql-functions/)
- [SQL JSON functions](https://druid.apache.org/docs/latest/querying/sql-json-functions/)
- [Query context](https://druid.apache.org/docs/latest/querying/query-context/)
- [Calcite](https://calcite.apache.org/)
