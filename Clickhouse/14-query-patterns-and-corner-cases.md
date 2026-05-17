# 14 · Query Patterns, Corner Cases, and Alternative Solutions

The user prompt asked specifically for **query patterns + corner cases + alternative better solutions**. This file is that. For each pattern: the naive approach, the corner cases that break it, and 2–3 alternative implementations with tradeoffs.

A staff-level signal in an interview is being able to name *three* ways to solve the same thing and say which is best for which situation.

## 14.1 Pattern: "Latest record per entity"

Get the latest row per user_id from a table that may have many versions.

### Approach A: SELECT FINAL (ReplacingMergeTree)

```sql
SELECT * FROM users FINAL WHERE user_id = 42;
```

**Corner cases**:
- FINAL forces merge-time logic at read time. Historically single-threaded; parallel since 23.3.
- On a 1B-row table, FINAL ranges over all parts.
- If the table isn't ReplacingMergeTree, FINAL is a no-op (or for Aggregating, an aggregation).
- `FINAL` may interact badly with PREWHERE optimization.

### Approach B: argMax

```sql
SELECT
    user_id,
    argMax(name,    updated_at) AS name,
    argMax(email,   updated_at) AS email,
    argMax(plan,    updated_at) AS plan,
    max(updated_at)             AS updated_at
FROM users
WHERE user_id = 42
GROUP BY user_id;
```

**Corner cases**:
- Need one `argMax` per non-key column — verbose for wide tables.
- Tie-breaking on equal `updated_at` is undefined; if you have a secondary version, use `argMax((name, ts2), (updated_at, ts2))` or tuples.

### Approach C: LIMIT 1 BY

```sql
SELECT *
FROM users
WHERE user_id = 42
ORDER BY user_id, updated_at DESC
LIMIT 1 BY user_id;
```

**Corner cases**:
- Implicit Sort on the read path. For a single user this is trivial; for "all users" this is heavy.
- `LIMIT 1 BY` returns the *first* row after sort; you must ORDER BY to get the right "first".

### Approach D: Subquery against an MV (best for ad-hoc-style)

Maintain a per-user-current-state MV; the read is trivial:

```sql
SELECT * FROM users_current WHERE user_id = 42;
```

### Selection rubric

| Situation | Approach |
|-----------|----------|
| Small result set, infrequent | FINAL is fine |
| Large result set | argMax or LIMIT 1 BY |
| Hot dashboard | maintain an MV with argMaxState; query its argMaxMerge |
| Mostly inserts + occasional latest reads | ReplacingMergeTree + argMax pattern |

## 14.2 Pattern: "Top N items per group"

E.g., top 5 most-popular products per category, top 10 longest queries per user.

### Approach A: groupArray + slice

```sql
SELECT
    category,
    arraySlice(groupArray(product), 1, 5) AS top5
FROM (
    SELECT category, product
    FROM sales
    GROUP BY category, product
    ORDER BY count() DESC
)
GROUP BY category;
```

Memory-heavy; subquery materializes everything before grouping.

### Approach B: LIMIT N BY

```sql
SELECT category, product, count() AS c
FROM sales
GROUP BY category, product
ORDER BY category, c DESC
LIMIT 5 BY category;
```

Cleaner; ClickHouse's idiomatic answer. Still does a full sort.

### Approach C: arrayMap + topK approximate

```sql
SELECT
    category,
    topK(5)(product) AS top5
FROM sales
GROUP BY category;
```

Approximate; uses Filtered Space Saving algorithm. Fast and memory-bounded — best for very large groups.

### Approach D: window function row_number()

```sql
SELECT category, product, c FROM (
    SELECT category, product, count() AS c,
           row_number() OVER (PARTITION BY category ORDER BY count() DESC) AS rn
    FROM sales
    GROUP BY category, product
)
WHERE rn <= 5;
```

Works; window functions are supported but historically more expensive than `LIMIT BY`.

### Rubric

- Small / exact top-N → `LIMIT N BY`.
- Very large / approximate → `topK(N)`.
- Per-group with row_number for arbitrary downstream filtering → window function.

## 14.3 Pattern: Count distinct (uniques)

### Approach A: exact

```sql
SELECT uniqExact(user_id) FROM events;
```

Memory grows linearly with cardinality. OOM-friendly.

### Approach B: HyperLogLog

```sql
SELECT uniq(user_id)           FROM events; -- ~1% error
SELECT uniqHLL12(user_id)      FROM events; -- smaller HLL
SELECT uniqCombined(user_id)   FROM events; -- HLL + sets, best for medium cardinality
```

### Approach C: uniqExact for small groups, uniqCombined for big

A common pattern via `multiIf`:

```sql
-- pseudocode: choose per-tenant
```

### Corner cases
- `count(DISTINCT x)` is rewritten to `uniqExact(x)`. Adding NULL semantics changes behavior.
- HLL is a one-way state; you can union HLLs (`uniqMerge` over `uniqState`) but not subtract.
- Combining HLLs across many groups: `uniqState` in an AggregatingMergeTree + `uniqMerge` at read.

### Alternative
- Pre-aggregate to an AggregatingMergeTree of `uniqState(user_id)` per (day, dimension). Then ad-hoc queries union states cheaply.

## 14.4 Pattern: Percentiles / quantiles

### Approach A: exact

```sql
SELECT quantileExact(0.99)(latency_ms) FROM requests WHERE ts > now() - INTERVAL 1 HOUR;
```

Memory-heavy; sorts.

### Approach B: tdigest / GK / approx

```sql
SELECT quantileTDigest(0.99)(latency_ms) FROM requests WHERE ...;
SELECT quantileGK(1000, 0.99)(latency_ms) FROM requests WHERE ...;
```

`quantileTDigest` is the canonical good-default. `quantileGK` has an explicit accuracy parameter.

### Approach C: pre-aggregated state

In an AggregatingMergeTree:

```sql
CREATE TABLE latency_min (
    minute     DateTime,
    service    LowCardinality(String),
    p50_state  AggregateFunction(quantileTDigest, UInt32),
    p99_state  AggregateFunction(quantileTDigest, UInt32)
) ...;
```

Read with `quantileTDigestMerge(0.5)` etc.

### Corner cases

- Quantile aggregation states are *not* mergeable across different quantile functions (you can't combine a tdigest and a GK).
- TDigest accuracy degrades slightly per merge — usually fine, but cumulative.
- "Median" = `quantile(0.5)` = `median()` — same thing.

## 14.5 Pattern: Time-bucketing and gap filling

### Bucketing

```sql
SELECT toStartOfMinute(ts) AS minute, count() AS c
FROM events
WHERE ts >= now() - INTERVAL 1 HOUR
GROUP BY minute
ORDER BY minute;
```

Other bucketers: `toStartOfFiveMinutes`, `toStartOfFifteenMinutes`, `toStartOfHour`, `toStartOfDay`, `toStartOfWeek`, `toStartOfInterval(ts, INTERVAL 30 MINUTE)`.

### Gap filling (zero-fill missing minutes)

Naive `GROUP BY` only returns minutes that have data.

#### Approach A: WITH FILL

```sql
SELECT toStartOfMinute(ts) AS minute, count() AS c
FROM events
WHERE ts >= now() - INTERVAL 1 HOUR
GROUP BY minute
ORDER BY minute WITH FILL FROM toStartOfMinute(now() - INTERVAL 1 HOUR) TO toStartOfMinute(now()) STEP INTERVAL 1 MINUTE;
```

`WITH FILL` is the SQL extension; clean and the right answer in CH.

#### Approach B: ARRAY JOIN against a generated range

```sql
WITH arrayMap(i -> toStartOfMinute(now()) - INTERVAL i MINUTE, range(60)) AS minutes
SELECT m AS minute, ifNull(c, 0) AS cnt
FROM (
    SELECT toStartOfMinute(ts) AS minute, count() AS c FROM events WHERE ... GROUP BY minute
) right
ARRAY JOIN minutes AS m
WHERE m = minute OR ...;
```

More verbose; useful when WITH FILL doesn't compose with your outer joins.

## 14.6 Pattern: Funnel

E.g., "view → cart → checkout" with a time window.

### Approach A: windowFunnel

```sql
SELECT level, count() AS users
FROM (
    SELECT user_id,
           windowFunnel(3600)(ts,
               event = 'view',
               event = 'cart',
               event = 'checkout'
           ) AS level
    FROM events
    WHERE event IN ('view','cart','checkout') AND ts >= today() - 7
    GROUP BY user_id
)
GROUP BY level;
```

`windowFunnel(N)(ts, cond1, cond2, ...)` returns the longest prefix of conditions matched in order within N seconds. Use `GROUP BY` to count by level.

Variant: `windowFunnel(3600, 'strict_order')` enforces no other events between steps.

### Approach B: sequenceMatch / sequenceCount

```sql
SELECT sequenceMatch('(?1)(?2)(?3)')(ts, event = 'view', event = 'cart', event = 'checkout')
FROM events GROUP BY user_id;
```

For complex regex-like sequence patterns. More flexible than windowFunnel.

### Approach C: manual join

For arbitrary funnels or when the steps have complex conditions:

```sql
SELECT count(*) FROM (
    SELECT user_id, min(ts) AS view_t FROM events WHERE event='view' GROUP BY user_id
) v
JOIN (
    SELECT user_id, min(ts) AS cart_t FROM events WHERE event='cart' GROUP BY user_id
) c ON v.user_id = c.user_id AND c.cart_t > v.view_t AND c.cart_t < v.view_t + 3600
JOIN ...;
```

Verbose; sometimes the only way for non-trivial sequence logic.

### Rubric

- Standard ordered funnel → windowFunnel.
- Sequence with patterns → sequenceMatch.
- Custom step conditions / multi-table → manual joins.

## 14.7 Pattern: Retention / cohort

Users who came back N days later.

```sql
SELECT
    toDate(first_seen) AS cohort_day,
    floor(dateDiff('day', first_seen, returned_ts) / 7) AS week_n,
    count() AS users
FROM (
    SELECT
        user_id,
        min(ts) AS first_seen
    FROM events
    GROUP BY user_id
) AS first
ASOF LEFT JOIN events ON events.user_id = first.user_id AND events.ts > first.first_seen
GROUP BY cohort_day, week_n
ORDER BY cohort_day, week_n;
```

Or use the **retention** aggregator:

```sql
SELECT
    retention(
        ts BETWEEN today() - 28 AND today() - 21,    -- cohort window
        ts BETWEEN today() - 21 AND today() - 14,    -- week 1
        ts BETWEEN today() - 14 AND today() - 7,     -- week 2
        ts BETWEEN today() - 7  AND today()          -- week 3
    ) AS r
FROM events
GROUP BY user_id;
```

Returns array of booleans per user. Then aggregate to a retention curve.

## 14.8 Pattern: Sessionization

Group consecutive events by user into sessions defined by `> 30 min` gap.

```sql
SELECT
    user_id,
    arrayCumSum(arrayMap(i -> if(i = 0, 1, (ts_arr[i+1] - ts_arr[i]) > 1800), range(length(ts_arr)))) AS session_idx,
    ts_arr
FROM (
    SELECT user_id, groupArray(ts) AS ts_arr
    FROM (SELECT user_id, ts FROM events ORDER BY user_id, ts)
    GROUP BY user_id
);
```

Or with the `neighbor`/`runningDifference` family of functions per row.

Or simpler: pre-compute session_id at ingest time (in your application layer); store it as a column; no SQL gymnastics.

## 14.9 Pattern: Pivot / transpose

ClickHouse doesn't have PIVOT syntax. Two ways:

### Approach A: sumIf

```sql
SELECT
    day,
    sumIf(amount, status = 'paid')     AS paid,
    sumIf(amount, status = 'refunded') AS refunded,
    sumIf(amount, status = 'failed')   AS failed
FROM payments
GROUP BY day;
```

Clean and fast. Works when columns are known.

### Approach B: groupArray + arrayElement

For dynamic pivots, build arrays and decompose client-side.

### Approach C: -Resample combinator

Not really pivoting but: `sumResample(0, 24, 1)(amount, hour)` returns an array of 24 sums (one per hour). Useful for fixed-bucket pivots.

## 14.10 Pattern: Rolling / sliding window aggregates

### Approach A: window functions

```sql
SELECT
    ts,
    user_id,
    avg(latency) OVER (PARTITION BY user_id ORDER BY ts ROWS BETWEEN 100 PRECEDING AND CURRENT ROW) AS rolling_avg
FROM events;
```

Supported. Per-spec SQL.

### Approach B: neighbor / runningDifference

Older idioms. `runningDifference(x)` returns `x - prev(x)` within an ordered block. `neighbor(x, n)` returns x's value n rows ahead.

### Approach C: groupArrayMovingAvg

```sql
SELECT
    ts,
    groupArrayMovingAvg(100)(latency) OVER (PARTITION BY user_id ORDER BY ts) AS rolling
FROM events;
```

Special moving-window aggregator.

## 14.11 Pattern: Heavy hitter / top-K from a stream

```sql
SELECT topK(10)(url) FROM requests WHERE ts > now() - INTERVAL 1 HOUR;
```

`topK(N)` is approximate but memory-bounded; the right tool 99% of the time over `arraySort + ARRAY JOIN`.

Approximate state version for MVs:
```sql
topKState(10)(url) AggregateFunction(topK(10), String)
```

## 14.12 Pattern: Bitmap aggregates (set algebra at scale)

```sql
SELECT
    groupBitmapState(user_id) AS users_who_did_x
FROM events WHERE event = 'X';

-- intersect:
SELECT groupBitmapAnd(states) FROM ...;

-- count distinct:
SELECT groupBitmapMerge(states) AS roar, bitmapCardinality(roar);
```

`groupBitmap` uses **Roaring Bitmaps** internally — compact, mergeable, and supports AND/OR/NOT. Use for "users who did X AND Y AND NOT Z" kinds of cohort logic.

## 14.13 Pattern: Joining a small dimension

Always use **dictionary** (`dictGet`) or denormalize.

```sql
SELECT
    event_id,
    dictGet('countries_dict', 'name', country_code) AS country_name
FROM events;
```

vs the bad way:
```sql
SELECT e.*, c.name
FROM events e JOIN countries c ON e.country_code = c.code;
```

Even with a small `countries` table, `dictGet` avoids the hash-build cost per query.

## 14.14 Pattern: Joining two big tables

If you must, the order is:
1. Are they sharded by the same key? Local hash join is fast.
2. Is one small enough to fit in memory? Hash join with small on right.
3. Are both sorted by the join key? Full sorting merge with skip-sort.
4. Otherwise → grace_hash; will spill.

Or: pre-join at ingest via a materialized view that writes the joined shape.

## 14.15 Pattern: Date-dimension lookup (calendar table)

Don't store a calendar table. ClickHouse has rich date functions:

```sql
SELECT
    toStartOfMonth(ts)  AS month,
    toDayOfWeek(ts)     AS dow,
    toQuarter(ts)       AS q,
    toYear(ts)          AS year
FROM events;
```

For arbitrary range filling, use `numbers(N)` + `addDays(start, n)`.

## 14.16 Pattern: Inserting with deduplication

### Approach A: ReplicatedMergeTree block dedup

Insert the same block twice → Keeper dedups by block hash.

### Approach B: ReplacingMergeTree with version

Insert duplicates; merger picks the highest version.

### Approach C: pre-dedup at the application

For exactly-once-per-event-id with strict timing, use a sequence number and `ReplacingMergeTree` with `version`. Pair with periodic OPTIMIZE if you really need clean data.

### Approach D: Kafka engine consumer offsets

Kafka engine commits offsets when the row is inserted; replaying from a position is at-most-once given offsets are committed, or at-least-once if you re-consume. Dedup at the destination as needed.

## 14.17 Pattern: Sampling for cost

```sql
SELECT count() * 10
FROM events SAMPLE 0.1
WHERE ts > today();
```

Requires a `SAMPLE BY` clause on the table:
```sql
ORDER BY (date, intHash32(user_id))
SAMPLE BY intHash32(user_id)
```

Use for cheap-estimate queries (top-N exploratory analyses). Real-time-billing analytics must not sample.

## 14.18 Pattern: Compare current period vs previous (week-over-week)

### Approach A: union/group with case

```sql
SELECT
    sumIf(c, period='this')     AS this_week,
    sumIf(c, period='last')     AS last_week,
    (this_week - last_week) / last_week AS pct_change
FROM (
    SELECT count() AS c, 'this' AS period FROM events WHERE ts >= today() - 7 AND ts < today()
    UNION ALL
    SELECT count() AS c, 'last' AS period FROM events WHERE ts >= today() - 14 AND ts < today() - 7
);
```

### Approach B: subqueries

```sql
SELECT
    (SELECT count() FROM events WHERE ts >= today() - 7  AND ts < today()) AS this_week,
    (SELECT count() FROM events WHERE ts >= today() - 14 AND ts < today() - 7) AS last_week;
```

Same plan; cleaner code.

## 14.19 Pattern: Live tail (last N events)

Naive: `SELECT * FROM events ORDER BY ts DESC LIMIT 100;` — works but does a sort over the table.

Better: if ORDER BY includes `ts` as a leading column DESC isn't possible (CH ORDER BY is ASC). But you can `SELECT * FROM events WHERE ts > now() - INTERVAL 1 MINUTE ORDER BY ts DESC LIMIT 100;` — small range, sort over few rows.

Best: use the `LIVE VIEW` (experimental) for true push.

## 14.20 Must-internalize

- Every pattern has 2–3 implementations. Pick the one that fits the cardinality and read frequency.
- FINAL is the slow but correct path; argMax / LIMIT 1 BY are the practical paths.
- Approximate aggregates (uniq, quantileTDigest, topK) are usually the right choice in real-time analytics.
- windowFunnel / sequenceMatch / retention are first-class for product analytics.
- groupBitmap (Roaring) for set algebra at scale.
- WITH FILL for gap-filled time series.
- `LIMIT N BY` for top-N-per-group.
- `dictGet` over JOIN for small dimensions.

---

## Sources

- [argMax / argMin](https://clickhouse.com/docs/sql-reference/aggregate-functions/reference/argmax)
- [LIMIT BY](https://clickhouse.com/docs/sql-reference/statements/select/limit-by)
- [windowFunnel](https://clickhouse.com/docs/sql-reference/aggregate-functions/parametric-functions#windowFunnel)
- [sequenceMatch](https://clickhouse.com/docs/sql-reference/aggregate-functions/parametric-functions#sequenceMatch)
- [WITH FILL](https://clickhouse.com/docs/sql-reference/statements/select/order-by#with-fill-modifier)
- [Roaring bitmap functions](https://clickhouse.com/docs/sql-reference/functions/bitmap-functions)
- [Quantile / Approximate aggregates](https://clickhouse.com/docs/sql-reference/aggregate-functions/reference/quantiletdigest)
- [Window functions](https://clickhouse.com/docs/sql-reference/window-functions)
