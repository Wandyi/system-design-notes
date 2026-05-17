# 6 · Indexes, Projections, and Materialized Views — The Acceleration Layer

After picking a sane schema, this is where 80% of further query speedups come from. Three tools, used together:

1. **Primary index** — sparse, mandatory, defined by `ORDER BY`.
2. **Skip indexes** — secondary indexes that prove "no matches in this granule".
3. **Projections** and **Materialized Views** — extra physical copies of data in different shapes.

## 6.1 The primary (sparse) index — quick review

Already covered in [02](02-architecture-fundamentals.md). Summary:

- One entry per `index_granularity` (8192) rows.
- Entries are the leading columns of the `ORDER BY`.
- Used to find which granules to read for a `WHERE` predicate on the leading columns.

**Cardinality rule**: order leading columns from **low to high cardinality**, *given your query patterns*. The first column should be the one your queries filter on most often.

Common pattern for events:
```sql
ORDER BY (tenant_id, event_type, ts)
```

The `tenant_id` (low cardinality, ~100s) prunes to one tenant fast. Then `event_type` (~10s of values). Then `ts` (high cardinality but always filtered by range).

If you almost always filter "last hour for tenant X across all event types":
```sql
ORDER BY (tenant_id, ts, event_type)
```

The "right" order depends on the query mix. Profile it.

## 6.2 Skip indexes (secondary indexes)

Defined on existing columns; ClickHouse builds an extra structure that lets the scanner skip granules.

```sql
ALTER TABLE events
  ADD INDEX idx_event_type event_type TYPE set(100) GRANULARITY 4,
  ADD INDEX idx_user_bf    user_id    TYPE bloom_filter(0.01) GRANULARITY 8,
  ADD INDEX idx_status_mm  status     TYPE minmax GRANULARITY 4,
  ADD INDEX idx_msg_tok    msg        TYPE tokenbf_v1(32768, 4, 0) GRANULARITY 4;
```

`GRANULARITY N` means the index is computed over groups of N granules (so the index marks are 1 per N×8192 rows). Smaller = more selective, more storage; larger = less selective, less storage.

### `minmax`

Stores min/max per index granule. Best for numeric / date columns that have some clustering but aren't the primary key.

**Use when**: filtering on a range of a column that's correlated with the sort order but isn't itself in the PK. Eg: `WHERE price BETWEEN 10 AND 50` on a table sorted by `(ts, ...)`.

### `set(N)`

Stores up to N distinct values per index granule. Best for low-to-medium cardinality columns.

**Use when**: filtering on `column IN (...)` or `column = X` for a column not in the PK. Eg: filtering by HTTP method or browser.

### `bloom_filter(p)`

Bloom filter with target false-positive rate p. General-purpose membership testing.

**Use when**: filtering by equality on a high-cardinality column (UUIDs, hashes, user_ids that aren't in PK). Bloom filters are probabilistic; false positives mean "read the granule and find no rows" (correctness preserved, perf hit).

### `tokenbf_v1(size, hash_count, seed)`

Bloom filter over **tokens** in a string (split on non-alphanumeric). Good for searching log lines for known words.

```sql
ADD INDEX idx_log_tok log_message TYPE tokenbf_v1(32768, 4, 0) GRANULARITY 4
-- then
SELECT * FROM logs WHERE log_message LIKE '%timeout%' AND ...
```

Helps `LIKE '%word%'` because the planner extracts the token "word" and consults the bloom filter.

### `ngrambf_v1(n, size, hash_count, seed)`

Bloom filter over **n-grams** of the string. For full-text-ish queries where token splitting isn't enough (Asian languages, identifiers).

### `text` (26.2+, replaces tokenbf_v1)

A new full-text-style index that supersedes the bloom-filter token variants. Better quality, more features.

### Skip-index gotchas

- They're probabilistic / approximate; they can **only prune negatives** when they're sure.
- Negation predicates (`NOT LIKE`, `<>`, `NOT IN`) usually can't use bloom-filter-class indexes.
- Adding an index on an existing table requires `MATERIALIZE INDEX` to back-fill old parts.
- Each index has CPU and storage cost on insert; only add when you measured the win.
- `GRANULARITY` matters. Test sizes; defaults are not always optimal.

## 6.3 Projections

A **projection** is an additional physical copy of data in a different shape. The query optimizer transparently picks projections when they accelerate a query.

### Sort-order projection

```sql
ALTER TABLE events ADD PROJECTION p_by_user (
    SELECT *
    ORDER BY (user_id, ts)
);
ALTER TABLE events MATERIALIZE PROJECTION p_by_user;
```

Now queries filtered by `user_id` use the projection's sort order. The base table stays `ORDER BY (ts, ...)`.

Cost: storage doubles for that projection. Inserts write to both.

### Aggregating projection

```sql
ALTER TABLE events ADD PROJECTION p_per_minute (
    SELECT
        toStartOfMinute(ts) AS minute,
        event_type,
        count() AS cnt
    GROUP BY minute, event_type
);
```

A subset of an AggregatingMergeTree pattern, but integrated into the same table. Used when a query's GROUP BY matches the projection.

### Projection vs. materialized view

| Aspect | Projection | Materialized View |
|--------|------------|---------------------|
| Storage | Inside the parent table | Separate table |
| Query rewrite | Automatic by optimizer | Explicit (you SELECT from the MV target) |
| Refresh model | Always in sync with base table | Insert-time trigger (or scheduled if Refreshable) |
| ALTER cost | High (rebuilds the projection) | Low (drop and recreate the MV) |
| Use case | Different sort order / pre-agg of *same* table | Different shape / cross-table / external sources |

A staff-level answer: projections are easier (auto-rewrite), but materialized views are more flexible and easier to evolve. Prefer MVs unless the auto-rewrite buys you something explicit.

## 6.4 Materialized views — the canonical acceleration pattern

ClickHouse MVs are **insert-time triggers**. When you INSERT into the source table, the MV's SELECT is evaluated against the *new block* and the result is INSERTed into the target table.

The pattern is:

```sql
CREATE TABLE raw_events (
    ts        DateTime,
    user_id   UInt64,
    event     LowCardinality(String),
    revenue   Decimal64(2)
) ENGINE = MergeTree
ORDER BY (ts, user_id);

-- The destination, an AggregatingMergeTree
CREATE TABLE daily_rev (
    day        Date,
    event      LowCardinality(String),
    rev_state  AggregateFunction(sum, Decimal64(2)),
    users      AggregateFunction(uniq, UInt64)
) ENGINE = AggregatingMergeTree
ORDER BY (day, event);

-- The materialized view connects them
CREATE MATERIALIZED VIEW daily_rev_mv TO daily_rev AS
SELECT
    toDate(ts)                         AS day,
    event,
    sumState(revenue)                  AS rev_state,
    uniqState(user_id)                 AS users
FROM raw_events
GROUP BY day, event;
```

Reading:

```sql
SELECT day, event,
       sumMerge(rev_state)   AS revenue,
       uniqMerge(users)      AS users
FROM daily_rev
GROUP BY day, event
ORDER BY day, event;
```

### How it actually works on insert

1. INSERT happens into `raw_events`. The new block has, say, 100k rows.
2. ClickHouse runs the MV's SELECT against this block — produces (say) 5k aggregated rows.
3. Those 5k rows are inserted into `daily_rev`.
4. AggregatingMergeTree later merges parts; rows with the same `(day, event)` have their `*State` aggregates combined.

So the MV doesn't aggregate the whole base table — it aggregates each incoming block. The AggregatingMergeTree merge does the cross-block combination.

### Reading from a partially-merged AggregatingMergeTree

Until merges fully consolidate, you'll see multiple rows for the same `(day, event)`. **You must always GROUP BY + Merge** in your read query. Doing `SELECT * FROM daily_rev` without GROUP BY gives you the un-merged state and is a common bug.

### Cascading MVs (multi-granularity)

Want hourly, daily, and monthly aggregates? Don't run three MVs against the raw table — chain them.

```sql
-- hourly from raw
CREATE MATERIALIZED VIEW hourly_mv TO hourly_rev AS SELECT ...;

-- daily from hourly (cheaper)
CREATE MATERIALIZED VIEW daily_mv TO daily_rev AS
SELECT toDate(hour) AS day, event,
       sumMergeState(rev_state) AS rev_state,
       uniqMergeState(users)    AS users
FROM hourly_rev
GROUP BY day, event;
```

Note `sumMergeState` (combines states from a state column, producing a state). The same pattern stacks indefinitely.

### MVs and JOINs

MVs see the *inserted block only*. So joining to a dimension table works only if the dimension is loaded — usually via a **dictionary** (`dictGet`), not a JOIN.

```sql
CREATE MATERIALIZED VIEW user_country_rev_mv TO user_country_rev AS
SELECT
    toDate(ts) AS day,
    dictGet('users_dict', 'country', user_id) AS country,
    sumState(revenue) AS rev_state
FROM raw_events
GROUP BY day, country;
```

### MV gotchas

- An INSERT that *partially* fails the MV target leaves the source updated. Use `materialized_views_ignore_errors = 1` to relax, or use `chained` MVs.
- A schema change to the target table breaks the MV until the MV is dropped/recreated.
- Backfilling existing data: use `INSERT INTO mv_target SELECT ... FROM source` with the MV's SELECT manually; don't expect the MV to retro-aggregate.
- A heavy MV slows every INSERT. Multiple MVs amplify this.
- An MV with a `WHERE` filter only writes rows matching the filter.

### Refreshable Materialized Views

For aggregations that can't be incrementalized (medians, rank, top-N-per-group requiring whole-table sort, joins of two huge dynamic tables):

```sql
CREATE MATERIALIZED VIEW top_users
REFRESH EVERY 1 HOUR
TO top_users_target AS
SELECT user_id, count() AS events
FROM events
WHERE ts > now() - INTERVAL 24 HOUR
GROUP BY user_id
ORDER BY events DESC
LIMIT 1000;
```

`APPEND` mode adds rows to the target each refresh; `REPLACE` (default) replaces the whole target. APPEND is useful for snapshot-history-of-aggregates patterns.

## 6.5 Picking the right tool for "I want a fast filter on X"

| Situation | Best tool |
|-----------|-----------|
| X is a leading column of ORDER BY | (free) primary index |
| X is moderately correlated with ORDER BY and is numeric/date | `minmax` skip index |
| X has < ~10K distinct values | `set(N)` skip index |
| X is high-cardinality equality (UUIDs, IDs) | `bloom_filter` skip index |
| X is a substring of a free-text column | `tokenbf_v1` or `text` (26.2+) |
| Your queries always group/agg by Y | MV → AggregatingMergeTree on Y |
| Your queries often re-sort by Z | projection ORDER BY (Z, ...) |
| You need top-N each window | refreshable MV with LIMIT BY |

## 6.6 Layered acceleration in one example

A request-log table at ~1B rows/day:

```sql
CREATE TABLE requests (
    ts          DateTime CODEC(Delta, ZSTD(1)),
    tenant_id   UInt32,
    service     LowCardinality(String),
    status      UInt16,
    user_id     UInt64,
    latency_ms  UInt32,
    url         String
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/requests','{replica}')
PARTITION BY toYYYYMM(ts)
ORDER BY (tenant_id, service, ts);

-- skip index for user_id lookups
ALTER TABLE requests ADD INDEX idx_user user_id TYPE bloom_filter(0.01) GRANULARITY 4;

-- skip index for url substring
ALTER TABLE requests ADD INDEX idx_url url TYPE tokenbf_v1(32768, 4, 0) GRANULARITY 4;

-- projection re-sort for user-centric queries
ALTER TABLE requests ADD PROJECTION p_by_user (
    SELECT * ORDER BY (user_id, ts)
);

-- MV for per-minute / per-tenant / per-service aggregates
CREATE TABLE per_min_agg (
    minute      DateTime,
    tenant_id   UInt32,
    service     LowCardinality(String),
    cnt         AggregateFunction(count),
    p50_state   AggregateFunction(quantileTDigest, UInt32),
    p99_state   AggregateFunction(quantileTDigest, UInt32),
    err_cnt     AggregateFunction(sumIf, UInt8, UInt8)
) ENGINE = ReplicatedAggregatingMergeTree('/clickhouse/tables/{shard}/per_min_agg','{replica}')
ORDER BY (tenant_id, service, minute);

CREATE MATERIALIZED VIEW per_min_mv TO per_min_agg AS
SELECT
    toStartOfMinute(ts) AS minute,
    tenant_id,
    service,
    countState()                        AS cnt,
    quantileTDigestState(latency_ms)    AS p50_state,
    quantileTDigestState(latency_ms)    AS p99_state,
    sumIfState(1, status >= 500)        AS err_cnt
FROM requests
GROUP BY minute, tenant_id, service;
```

Now:
- "Last 24h request count per tenant per service per minute" → MV (sub-second).
- "Find requests for user 42 in last 7d" → bloom filter + projection (~ms).
- "Find requests with URL containing 'cart'" → token bloom filter narrows then scans.
- "All requests for tenant 5, service X, ts range" → primary index path (fastest).

## 6.7 Must-internalize

- Primary index = sparse, defined by ORDER BY, the most important index.
- Skip indexes are extra; they only help when the primary index can't.
- Bloom-style indexes are probabilistic, can only prune.
- Projections = automatic-rewrite alternate physical layouts.
- Materialized views = insert-time triggers, paired with AggregatingMergeTree for incremental rollups.
- Always GROUP BY + *Merge when reading from AggregatingMergeTree.
- Chain MVs for multi-granularity, don't fan out from raw.
- Refreshable MVs for things you can't incrementalize.
- Layer all of these — the best schemas use 2-3 simultaneously.

---

## Sources

- [Sparse primary indexes](https://clickhouse.com/docs/guides/best-practices/sparse-primary-indexes)
- [Data skipping indices](https://clickhouse.com/docs/best-practices/use-data-skipping-indices-where-appropriate)
- [Projections](https://clickhouse.com/docs/sql-reference/statements/alter/projection)
- [AggregatingMergeTree + MV](https://clickhouse.com/docs/engines/table-engines/mergetree-family/aggregatingmergetree)
- [Materialized views guide — BigData Boutique](https://bigdataboutique.com/blog/clickhouse-materialized-views-guide)
- [Refreshable materialized views (PR/blog)](https://clickhouse.com/blog/clickhouse-release-24-09)
- [Altinity — skipping indices black magic](https://altinity.com/blog/clickhouse-black-magic-skipping-indices)
