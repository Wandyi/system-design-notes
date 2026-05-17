# 7 · Indexes and DataSketches

Druid's "acceleration layer" is composed of: every-dimension-bitmap-indexes (free, by default), numeric range indexes (opt-in), JSON column indexes, and the DataSketches family (the secret sauce for approximate aggregations).

## 7.1 The bitmap index (revisited)

Every string dimension gets one. For each distinct value, a Roaring bitmap marks which rows contain it.

Use cases:
- `WHERE dim = X` → one bitmap read.
- `WHERE dim IN (X, Y, Z)` → bitmap OR.
- `WHERE dim != X` → bitmap NOT (slow on dense bitmaps; OK on sparse).
- `WHERE dim1 = X AND dim2 = Y` → bitmap AND of two bitmaps.

Performance is so good that filtering is rarely the bottleneck — aggregation usually is.

### Skip the bitmap for a dimension

For very-high-cardinality dimensions (UUIDs, session IDs) where the bitmap is huge but never used for filtering:

```json
{
  "name": "session_id",
  "type": "string",
  "createBitmapIndex": false
}
```

You save segment storage at the cost of `WHERE session_id = X` becoming a column scan. Worth it for dimensions never filtered on.

## 7.2 Numeric range indexes

Druid 26+ has numeric range indexes for long/double/float columns:

```json
{ "type": "long", "name": "price", "createBitmapIndex": true }
```

Stores per-range bitmaps. Filters like `WHERE price BETWEEN 10 AND 50` use bitmap operations.

Compare to ClickHouse's `minmax` and `set` skip indexes — same goal, different mechanism.

## 7.3 JSON column auto-indexing

For a `COMPLEX<json>` column, Druid auto-discovers paths during ingest and creates **per-path subcolumns** with their own dictionaries and bitmap indexes.

```json
{ "name": "props", "type": "json" }
```

Query:
```sql
SELECT count(*) FROM events WHERE JSON_VALUE(props, '$.browser') = 'Chrome';
```

The optimizer pushes down to the `props.browser` subcolumn's bitmap. Fast.

Comparable to ClickHouse's new `JSON(...)` type — same idea: typed paths with index acceleration.

## 7.4 Spatial / geo indexes

For columns with shape-like data (point-in-polygon queries):

```json
{
  "spatialDimensions": [
    {
      "dimName": "coords",
      "dims": ["lat", "lon"]
    }
  ]
}
```

R-tree-style indexing. Useful for "all events within this radius of this point" queries.

## 7.5 DataSketches — the secret sauce

The single most-loved feature for analysts. Approximate aggregations stored as compact binary sketches, mergeable across rows and segments.

The four to memorize:

### HLLSketch — distinct count

```json
{ "type": "HLLSketchBuild", "name": "distinct_users", "fieldName": "user_id", "lgK": 12 }
```

Query:
```sql
SELECT APPROX_COUNT_DISTINCT_DS_HLL(distinct_users) FROM events;
```

- ~1.6% error at default `lgK = 12`.
- Sketch size: ~16 KB.
- Merge: union of HLL sketches gives HLL of the union of inputs.

### thetaSketch — distinct count + set algebra

```json
{ "type": "thetaSketch", "name": "users", "fieldName": "user_id", "size": 16384 }
```

Query (set algebra at read time):
```sql
SELECT THETA_SKETCH_ESTIMATE(THETA_SKETCH_INTERSECT(
  filtered(users, event='login'),
  filtered(users, event='purchase')
)) AS logged_in_and_purchased FROM events;
```

Use case: cohort analysis. "Users who did A and B" = intersect two theta sketches. "Users who did A but not B" = set difference.

### quantilesDoublesSketch — percentiles

```json
{ "type": "quantilesDoublesSketch", "name": "latency_q", "fieldName": "latency_ms", "k": 128 }
```

Query:
```sql
SELECT APPROX_QUANTILE_DS(latency_q, 0.99) FROM events;
```

- Returns p99 (or any quantile).
- Mergeable across segments.
- Error bound: ~1% at default `k = 128`.

### tupleSketch — multi-column

For tracking distinct count alongside a numeric aggregate:

```json
{ "type": "arrayOfDoublesSketch", "name": "users_revenue", "fieldName": "user_id", "metricColumns": ["revenue"] }
```

Stores: distinct users + sum of their revenue, in one sketch. Useful for "average revenue per user" without join.

## 7.6 Why sketches are revolutionary for Druid

Without sketches:
- "Distinct users by country" requires keeping `user_id` as a dimension → no rollup → 10× storage.

With HLL sketch:
- `user_id` is dropped from dimensions; replaced by an HLL metric.
- Rollup works fully (rows with same `(country, event_type, minute)` collapse into one).
- Distinct count is ~1% off; storage is 10× smaller.

For most user-facing analytics, ~1% error on a "78,492 distinct users" number is unnoticeable; the speed and storage wins are dramatic.

## 7.7 Sketch sizing

The sizing parameter (`lgK`, `size`, `k`) controls accuracy vs. storage:
- Higher = more accurate, bigger sketch.
- Lower = less accurate, smaller sketch.

Defaults are sensible for ~1% error. Increase only if accuracy needs are explicit.

## 7.8 Combining sketches at query time

Sketches are **mergeable**. The Broker can union sketches from many segments to produce a global estimate:

```sql
SELECT country,
       APPROX_COUNT_DISTINCT_DS_HLL(distinct_users) AS users
FROM events
WHERE __time BETWEEN '...' AND '...'
GROUP BY country;
```

Each Historical returns per-segment HLL sketches per country; the Broker merges them; the user gets distinct count per country over the whole interval.

## 7.9 Theta sketch set operations (the cohort superpower)

```sql
-- Users who saw a campaign AND made a purchase in the same week
WITH ad_viewers AS (
  SELECT campaign_id, DS_THETA(user_id) AS sk
  FROM events WHERE event = 'ad_view' AND ts >= '2026-05-10' AND ts < '2026-05-17'
  GROUP BY campaign_id
),
buyers AS (
  SELECT DS_THETA(user_id) AS sk
  FROM events WHERE event = 'purchase' AND ts >= '2026-05-10' AND ts < '2026-05-17'
)
SELECT
  a.campaign_id,
  THETA_SKETCH_ESTIMATE(THETA_SKETCH_INTERSECT(a.sk, b.sk)) AS conversion_count
FROM ad_viewers a CROSS JOIN buyers b;
```

This kind of cohort/funnel analysis is brutal without sketches; trivial with them.

## 7.10 Compaction interaction with sketches

When Druid compacts segments, it can merge sketches via their natural merge operation. Two segments each with `distinct_users` HLL sketches for `country='US'` will produce one merged sketch.

This means rollup happens *again* at compaction time — the rollup ratio improves over the segment's lifetime.

## 7.11 Anti-patterns

- **Storing `user_id` as a dimension + ALSO an HLL sketch.** Duplicate storage; one of them is redundant.
- **Using exact distinct (`COUNT(DISTINCT x)` without sketch) on rolled-up data.** Druid will either error or fall back to GroupBy which can be very slow.
- **Sketch with too-low `lgK`** when you need precision. Error compounds across many segments.
- **Mixing theta sketch and HLL sketch on the same dimension.** They don't union with each other; pick one type.

## 7.12 Filtered aggregators

Useful for conditional metrics:

```json
{ "type": "filtered", "filter": {"type":"selector","dimension":"event","value":"purchase"},
  "aggregator": { "type": "doubleSum", "name": "purchase_revenue", "fieldName": "revenue" } }
```

In SQL:
```sql
SELECT SUM(revenue) FILTER (WHERE event = 'purchase') AS purchase_revenue FROM events;
```

Lets you pre-define per-event aggregates without splitting datasources.

## 7.13 Must-internalize

- Bitmap indexes are automatic on string dimensions. Roaring-compressed.
- Skip the bitmap for ultra-high-cardinality, never-filtered columns (`createBitmapIndex: false`).
- Numeric range indexes (newer) + JSON column auto-indexing fill gaps.
- DataSketches family: **HLL** (distinct), **theta** (distinct + set algebra), **quantilesDoublesSketch** (percentiles), **tuple** (multi-column).
- Sketches are mergeable across rows and segments — that's why rollup works for distinct-count metrics.
- Don't double-store: `user_id` as both dimension and HLL is wasted bytes.
- Filtered aggregators for per-condition metrics inline.

---

## Sources

- [DataSketches extension overview](https://druid.apache.org/docs/latest/development/extensions-core/datasketches-extension.html)
- [HLL sketch](https://druid.apache.org/docs/latest/development/extensions-core/datasketches-hll.html)
- [Theta sketch](https://druid.apache.org/docs/latest/development/extensions-core/datasketches-theta.html)
- [Quantiles sketch](https://druid.apache.org/docs/latest/development/extensions-core/datasketches-quantiles.html)
- [Indexes — Hellmar Becker blog](https://blog.hellmar-becker.de/2023/06/28/indexes-in-apache-druid/)
- [DataSketches website](https://datasketches.apache.org/)
- [Counting unique visitors across overlapping segments](https://blog.hellmar-becker.de/2022/06/05/druid-data-cookbook-counting-unique-visitors-for-overlapping-segments/)
