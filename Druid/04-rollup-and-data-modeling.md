# 4 · Rollup and Data Modeling

Rollup is Druid's most distinctive feature and the single biggest "wait, you can do that?" moment for newcomers from other OLAPs. ClickHouse achieves the same effect via materialized views to `AggregatingMergeTree`; Druid does it inline at ingest, as a *first-class concept of the storage layer*.

A staff interview will ask: *when should I roll up, how aggressively, and what corner cases come up?*

## 4.1 What rollup is

At ingest time, Druid can group rows that share the same dimension values into a single row, with metrics aggregated across them.

Raw input:

```
ts                    user_id  country  event_type  revenue
2026-05-17T12:01:00   1234     US       purchase    9.99
2026-05-17T12:01:23   5678     US       purchase    19.99
2026-05-17T12:01:45   9012     US       purchase    4.99
2026-05-17T12:02:10   2345     CA       view        0
```

Ingest spec:
```json
{
  "dimensionsSpec": { "dimensions": ["country", "event_type"] },
  "metricsSpec":   [
    { "type": "count",        "name": "events" },
    { "type": "doubleSum",    "name": "revenue", "fieldName": "revenue" }
  ],
  "granularitySpec": {
    "queryGranularity":  "MINUTE",
    "segmentGranularity": "HOUR",
    "rollup": true
  }
}
```

Stored after rollup:

```
__time                country  event_type  events  revenue
2026-05-17T12:01:00   US       purchase    3       34.97
2026-05-17T12:02:00   CA       view        1       0
```

`user_id` was dropped (not in `dimensions`), `__time` was truncated to minute (`queryGranularity`), three rows collapsed into one.

## 4.2 The mental model: dimensions vs metrics

This is *the* Druid modeling decision. Each column is one or the other:

- **Dimensions** = the things you GROUP BY and filter on. Stored with dictionary + bitmap. Identity is preserved.
- **Metrics** = the things you aggregate (sum, count, distinct, percentile). Stored as pre-aggregated values per (dimension-tuple, time-bucket).

Rule of thumb:
- "How many?" → metric (count, sum).
- "Distinct?" → HLL/Theta sketch metric.
- "What percentage?" → quantile sketch metric.
- "Group by it?" → dimension.
- "Filter on it?" → dimension.

Druid's strict typing forces you to think about this up-front. Bad rollup design produces either:
- **Insufficient rollup** — every row has unique dimension values, rollup ratio = 1× (no benefit, only overhead).
- **Over-rollup** — you dropped a dimension users want to drill into, requiring re-ingest to add it back.

## 4.3 Roll-up ratio — the headline number

```
rollup_ratio = raw_row_count / stored_row_count
```

- 1× = no rollup happening (probably a misconfiguration).
- 2-3× = modest; consider dropping a high-cardinality dimension.
- 10× = healthy for event-tracking workloads.
- 100× = great for low-cardinality dimensions + coarse `queryGranularity`.

Imply's `druid.indexer.runner.rollupRatio` metric tracks this; check it after ingest.

## 4.4 Query granularity = rollup grain

`queryGranularity` controls the time precision rows are bucketed to *during rollup*:

- `NONE` — no truncation; every row keeps its exact timestamp.
- `MINUTE`, `FIVE_MINUTE`, `HOUR`, `DAY` — truncate `__time` to this boundary.

Once segments are stored at this granularity, queries can't ask for *finer* time precision. You can ask `SELECT ... GROUP BY HOUR` from a segment with `queryGranularity = MINUTE` (just aggregates upward), but not `SELECT ... GROUP BY SECOND`.

Picking `queryGranularity`:
- **MINUTE** — typical for sub-second user-facing analytics on user activity.
- **HOUR** — for daily / weekly trend dashboards where minute resolution is overkill.
- **DAY** — for monthly / quarterly reports.
- **NONE** — only when you genuinely need per-event detail (e.g., trace analysis); rollup ratio drops to 1.

Pick the **coarsest granularity that satisfies the queries you'll actually run**. Trade fidelity for storage and speed.

## 4.5 Segment granularity vs query granularity

A common interview confusion:

- `segmentGranularity` = how big a time interval each segment covers (`HOUR`, `DAY`, `MONTH`). A *physical* property.
- `queryGranularity` = how aggressively rows are rolled up by time within a segment. A *logical* property.

Both can be set independently. Common pair: `segmentGranularity = DAY, queryGranularity = MINUTE`. One segment per day, with rows rolled up to per-minute granularity.

## 4.6 Without rollup (rollup: false)

You can disable rollup. The segment then stores every raw row.

When this is right:
- You need per-event details (e.g., individual user-event analysis).
- The dimensions are so high-cardinality that rollup doesn't help.
- You're using Druid as a log store (every line distinct).

The segment is still time-partitioned and bitmap-indexed; you just don't get the row-collapse.

## 4.7 Metric types — the catalog

### Basic numeric

- `count` — count of rows (before rollup).
- `longSum`, `doubleSum`, `floatSum` — sum of a column.
- `longMax`, `doubleMax`, `floatMax`, `longMin`, ...
- `longFirst`, `longLast` — first/last value within the rollup group (by `__time`).
- `doubleMean`, `doubleVar`, `stddev` — statistical aggregates.

### Approximate (DataSketches)

- `HLLSketchBuild` — distinct-count sketch (HyperLogLog).
- `thetaSketch` — distinct-count + set operations (union/intersection/diff).
- `quantilesDoublesSketch` — quantile sketch.
- `tupleSketch` — multi-column sketch.

### Approximate (Druid-built-in)

- `cardinality` — older HLL-style; mostly superseded by DataSketches.
- `hyperUnique` — older HLL.

### Other

- `filtered` — wraps another aggregator with a filter.
- `javascript` — arbitrary JS aggregator (deprecated; avoid).

## 4.8 The HLL / Theta sketch story

The single most important rollup optimization for high-cardinality distinct counts.

Without sketches:

```
events: country=US, country=US, country=CA, country=US, country=DE
       → after rollup with userId dim: 5 rows (no rollup gain)

To compute "distinct users by country":
       → must keep userId as a dim → no rollup possible.
```

With HLL sketch:

```
events:
  Before rollup:
    country, userId
  After rollup (with HLL):
    country, distinctUsers_HLLSketch     (userId dropped, replaced by an HLL of userId)

For US: HLL contains {1234, 5678, 9012} → sketch
For CA: HLL contains {2345} → sketch

Query "distinct users by country" reads HLL sketches, returns ~exact distinct count.
```

The HLL sketch is a fixed-size binary blob (typically 8-32 KB) per (country, time-bucket). Merging two HLL sketches gives the HLL of the union. Two-orders-of-magnitude storage savings vs storing every userId.

Theta sketches extend HLL with set operations — useful for "users who did A AND did B" type cohort analysis.

## 4.9 Multi-value dimensions

A column where each row can have multiple values (array of strings):

```json
"dimensions": [
  {"name": "tags", "type": "string", "multiValueHandling": "ARRAY"}
]
```

Each value gets its own bitmap bit per row. Queries can:
- Filter: `WHERE tags = 'sport'` (match any row with that tag).
- Group: `GROUP BY MV_TO_ARRAY(tags)` (explodes per value).
- Set: `WHERE MV_CONTAINS_ALL(tags, ARRAY['sport','live'])`.

Comparable to ClickHouse's `Array(String)` — same idea but Druid indexes each element with a bitmap.

## 4.10 Schema design — explicit vs auto-discovered (Druid 26+)

Two flavors:

### Explicit (traditional)

You declare every dimension and metric in the ingestion spec. Predictable, type-safe, but rigid.

### Schema auto-discovery

```json
"dimensionsSpec": {
  "useSchemaDiscovery": true,
  "includeAllDimensions": true
}
```

Druid infers dimensions from the incoming data, with types learned from the first occurrence. New fields appear in future segments automatically.

When to pick which:
- **Explicit** if your schema is stable and you want predictable storage.
- **Auto** if your schema is evolving (e.g., new event types add new fields). Pair with **JSON columns** for the highest-flexibility option.

## 4.11 Dimension ordering and segment sort

Within a segment, rows are sorted (by default) by `__time` first, then by the dimensions in the order they're declared. Sort matters because:
- **Better compression** — adjacent rows are similar.
- **Faster scan** for predicates on the leading sort columns.

For event-tracking, declare dimensions in **ascending cardinality** order (low-cardinality first). This mirrors the ClickHouse ORDER BY guidance.

`forceSegmentSortByTime: true` (default) puts `__time` first; `forceSegmentSortByTime: false` lets dimensions lead (newer Druid versions allow this).

## 4.12 Worked example — event tracking schema with rollup

```sql
-- via MSQ
REPLACE INTO events OVERWRITE WHERE __time >= TIMESTAMP '2026-05-17'
                                 AND __time <  TIMESTAMP '2026-05-18'
SELECT
  FLOOR(__time TO MINUTE)              AS __time,
  country,
  event_type,
  device,
  COUNT(*)                              AS events,
  SUM(revenue)                          AS revenue,
  APPROX_COUNT_DISTINCT_DS_HLL(user_id) AS distinct_users,
  DS_QUANTILES_SKETCH(latency_ms)       AS latency_quantiles
FROM (
  SELECT * FROM TABLE(EXTERN(...))    -- raw events from Kafka offset range
)
GROUP BY 1, 2, 3, 4
PARTITIONED BY DAY
CLUSTERED BY country, event_type;
```

Reading:
- Time bucketed to MINUTE.
- 3 dimensions kept (country, event_type, device).
- user_id dropped (rolled into HLL sketch).
- latency_ms dropped (rolled into quantile sketch).
- One segment per day, clustered by (country, event_type) for better compression and filter speed.

Query:
```sql
SELECT country,
       SUM(events)                       AS events,
       SUM(revenue)                      AS revenue,
       APPROX_COUNT_DISTINCT_DS_HLL(distinct_users) AS users,
       APPROX_QUANTILE_DS(latency_quantiles, 0.99)  AS p99
FROM events
WHERE __time >= TIMESTAMP '2026-05-17' AND __time < TIMESTAMP '2026-05-18'
GROUP BY country
ORDER BY revenue DESC;
```

Each pre-aggregated metric is merged at read time. Sub-second on a year of data, easy.

## 4.13 Common rollup mistakes

1. **Including a high-cardinality dimension (UUID, session_id) in dimensions** — rollup ratio drops to 1×; storage balloons.
2. **Setting `queryGranularity = NONE` when MINUTE would do** — wastes 60× the row count for no business value.
3. **Storing the same data both rolled-up and raw** — pick one; rolled-up for dashboards, raw if absolutely needed.
4. **Picking dimensions that change frequently** — every change creates a new dimension tuple, defeating rollup.
5. **Forgetting to add HLL sketch metrics for distinct count** — without them, you can't ask "distinct users" on rolled-up data.

## 4.14 Reverting / changing rollup decisions

You can't ALTER rollup. To change rollup config:
1. Re-ingest the affected time range with the new spec (REPLACE).
2. Or run a compaction task with the new config.

This is why staff-level Druid design always starts with a careful think on `queryGranularity` and dimension list.

## 4.15 Compare with ClickHouse

| | ClickHouse | Druid |
|--|------------|-------|
| Pre-aggregation mechanism | `AggregatingMergeTree` + MV | Native rollup at ingest |
| Granularity control | MV-defined GROUP BY (`toStartOfMinute(ts)`) | `queryGranularity` |
| Distinct count | `uniqCombinedState` in AggregatingMergeTree | HLL/Theta sketch metric |
| Quantile | `quantileTDigestState` | quantile sketch metric |
| Dimension/metric split | Implicit (columns + aggregators in MV) | **Explicit** in schema |
| Re-aggregate at finer granularity | Possible if MV grain is fine enough | Possible if `queryGranularity` is fine enough |

The pattern is the same; Druid just makes it inline and explicit.

## 4.16 Must-internalize

- Rollup = collapsing rows that share dimension values into one row with aggregated metrics.
- Pre-decide: which fields are dimensions, which are metrics, which are dropped, which become sketches.
- `queryGranularity` controls time bucketing during rollup; `segmentGranularity` controls physical segment size.
- HLL/Theta sketches let you keep distinct-count semantics without keeping the high-cardinality dim.
- Multi-value dimensions for tags/labels.
- Schema discovery + JSON columns let schema evolve organically.
- Can't change rollup config retroactively; re-ingest the affected time range.

---

## Sources

- [Rollup — schema-design docs](https://druid.apache.org/docs/latest/ingestion/rollup/)
- [Schema design tips](https://druid.apache.org/docs/latest/ingestion/schema-design/)
- [DataSketches HLL](https://druid.apache.org/docs/latest/development/extensions-core/datasketches-hll.html)
- [DataSketches Theta](https://druid.apache.org/docs/latest/development/extensions-core/datasketches-theta.html)
- [DataSketches Quantiles](https://druid.apache.org/docs/latest/development/extensions-core/datasketches-quantiles.html)
- [How to create roll-ups (Rill blog)](https://www.rilldata.com/blog/how-to-create-roll-ups-in-apache-druid)
