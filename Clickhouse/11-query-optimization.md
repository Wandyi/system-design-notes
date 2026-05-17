# 11 · Query Optimization — PREWHERE, FINAL, Settings, Profile

A staff engineer can take a slow query and make it fast. This file is the toolkit for that, organized as: what to check, what to change, how to measure.

## 11.1 The diagnostic loop

1. Reproduce the slow query with a fixed `query_id`.
2. Use `EXPLAIN PLAN`, `EXPLAIN PIPELINE`, `EXPLAIN ESTIMATE` to understand the plan.
3. After running, query `system.query_log` and `system.query_thread_log` for per-thread stats.
4. Check `ProfileEvents` for granular counters (bytes read, marks scanned, hash collisions, etc.).
5. Apply one change at a time; re-measure.

```sql
SELECT
    query_id, query_duration_ms, read_rows, read_bytes,
    memory_usage, result_rows, exception,
    ProfileEvents
FROM system.query_log
WHERE query_id = 'xyz'
  AND type = 'QueryFinish'
ORDER BY event_time DESC LIMIT 1;
```

## 11.2 EXPLAIN flavors

- `EXPLAIN SYNTAX` — show the rewritten SQL.
- `EXPLAIN AST` — parsed abstract syntax tree.
- `EXPLAIN` (default = PLAN) — logical plan.
- `EXPLAIN PIPELINE` — physical processors and parallelism.
- `EXPLAIN ESTIMATE` — estimated rows / bytes to read.
- `EXPLAIN PLAN actions = 1, indexes = 1` — show what indexes were used.

A good first move on any slow query: `EXPLAIN PLAN indexes=1` to see whether the primary index, skip indexes, or projections are being used.

## 11.3 PREWHERE — column-store filter trick

Recap from [02](02-architecture-fundamentals.md): PREWHERE reads filter columns first, materializes only surviving rows from other columns.

```sql
SELECT user_id, big_text_blob
FROM events
PREWHERE event_type = 'login'
WHERE  ts > now() - INTERVAL 1 DAY;
```

ClickHouse auto-promotes simple predicates. Manually annotate when the optimizer picks badly. Verify with EXPLAIN PLAN.

**When automatic PREWHERE goes wrong**:
- The promoted predicate is on a column more expensive to read than the filtered columns.
- The promoted predicate's selectivity is low (filters very few rows) so the savings don't materialize.

Override with explicit `PREWHERE`.

## 11.4 FINAL — only when you must

`SELECT ... FROM t FINAL` forces ClickHouse to apply merge-time logic (dedup for Replacing, collapse for Collapsing, etc.) at query time. Expensive — historically single-threaded; since 23.3 parallel, controlled by `max_final_threads`.

**Alternatives that are usually faster**:

```sql
-- 1. ReplacingMergeTree with version column:
SELECT *
FROM users
WHERE (user_id, updated_at) IN (
    SELECT user_id, max(updated_at) FROM users GROUP BY user_id
);

-- 2. argMax pattern:
SELECT
    user_id,
    argMax(name,    updated_at) AS name,
    argMax(email,   updated_at) AS email,
    max(updated_at)             AS updated_at
FROM users
GROUP BY user_id;

-- 3. LIMIT 1 BY:
SELECT *
FROM users
ORDER BY user_id, updated_at DESC
LIMIT 1 BY user_id;
```

These avoid FINAL on the read path and let normal granule pruning work.

When FINAL is right: small result set, infrequent query, dev work. Never inside a hot dashboard.

## 11.5 Key settings to know

| Setting | Default | What it does |
|---------|---------|--------------|
| `max_threads` | 0 (= #cores) | Threads per query |
| `max_block_size` | 65,536 | Rows per execution block |
| `max_memory_usage` | varies | Per-query memory cap (OOMs gracefully if exceeded) |
| `max_bytes_before_external_group_by` | 0 | If > 0, GROUP BY spills to disk past this |
| `max_bytes_before_external_sort` | 0 | Same for ORDER BY |
| `max_bytes_in_join` | 0 | Same for JOIN hash table |
| `join_algorithm` | direct,parallel_hash | Adaptive algorithm pick |
| `optimize_read_in_order` | 1 | Use sort order for ORDER BY queries (skip sort) |
| `optimize_aggregation_in_order` | 0 | Use sort order for GROUP BY (memory-efficient) |
| `optimize_distinct_in_order` | 1 | Use sort order for DISTINCT |
| `use_uncompressed_cache` | 0 | Cache decompressed data (useful for hot queries) |
| `merge_tree_min_rows_for_concurrent_read` | 163840 | Threshold to read parts concurrently |
| `prefer_localhost_replica` | 1 | Prefer local replica for Distributed queries |
| `distributed_group_by_no_merge` | 0 | Skip final merge if shards return distinct keys |
| `optimize_move_to_prewhere` | 1 | Auto PREWHERE promotion |
| `enable_optimize_predicate_expression` | 1 | Push predicates into subqueries |
| `parallel_view_processing` | 1 | Parallelize MV inserts |
| `async_insert` | 0 (1 in Cloud) | Server-side small-insert batching |

For OOMs: lower `max_threads` (less per-thread mem) or set spill limits (`max_bytes_before_external_*`) — slow but reliable.

For CPU starvation under concurrency: lower per-query `max_threads` so queries share CPU more fairly.

## 11.6 Memory model

A query's memory is bounded by `max_memory_usage`. Each thread takes a slice; aggregations, joins, sorts can be the biggest consumers.

If you OOM:
1. Reduce `max_threads`.
2. Enable spilling (`max_bytes_before_external_group_by`, `max_bytes_before_external_sort`).
3. For joins: switch to `grace_hash` or `partial_merge`.
4. Use approximate aggregates (`uniqCombined` vs `uniqExact`, `quantileTDigest` vs `quantileExact`).
5. Push the work into a materialized view so the read query is light.

## 11.7 Avoiding SELECT *

ClickHouse is column-store. `SELECT *` reads every column → defeats the columnar advantage. Always project only what you need.

```sql
-- BAD on a 200-column table
SELECT * FROM events WHERE id = 42;

-- GOOD
SELECT id, name, ts FROM events WHERE id = 42;
```

## 11.8 Approximate aggregates

Real-time analytics often tolerates ~1% error for 10× speed and memory cuts.

| Exact | Approximate | Notes |
|-------|-------------|-------|
| `uniqExact(x)` | `uniq(x)` (HLL) | ~1% error; fixed memory |
| `uniqExact(x)` | `uniqCombined(x)` | Better at scale; HLL + sets |
| `uniqExact(x)` | `uniqHLL12(x)` | Smaller HLL variant |
| `quantileExact(x)` | `quantileTDigest(x)` | T-digest; great accuracy |
| `quantileExact(x)` | `quantileGK(x)` | Greenwald-Khanna |
| `topK(x, N)` | `topK(N, threshold)(x)` | Approximate heavy-hitters |

For pre-aggregated states (in AggregatingMergeTree), the *State / *Merge variants compose nicely.

## 11.9 Cardinality pitfalls

A `GROUP BY` with very high cardinality (UUIDs, etc.) blows up memory.

Workarounds:
- Bucket the high-cardinality column: `GROUP BY intDiv(user_id, 1000)`.
- Use a pre-aggregating MV.
- Use a sampling clause (`SAMPLE 0.1`) on tables that have a sample-by key.
- Use approximate functions (`uniq` instead of `groupArray`).

## 11.10 The `arrayJoin` blow-up

```sql
SELECT count() FROM events ARRAY JOIN tags;
```

`arrayJoin` (or `ARRAY JOIN`) explodes arrays — 1 row × 10 tags = 10 rows. Memory and CPU grow accordingly.

Use it deliberately; alternatives include `arrayMap`, `arraySum`, `arrayFilter` to do per-array work without exploding.

## 11.11 SAMPLE

Tables can be declared with a `SAMPLE BY` key. Queries can then run on a fraction:

```sql
CREATE TABLE web_events (
    ...
) ENGINE = MergeTree
ORDER BY (date, intHash32(user_id))
SAMPLE BY intHash32(user_id);

SELECT count() * 10 FROM web_events SAMPLE 0.1 WHERE date = today();
```

Fast estimates over a uniform 10% slice. Caveat: aggregate accuracy depends on uniformity; outliers can skew small samples.

## 11.12 Reading the profile (ProfileEvents)

Useful counters from `system.query_log.ProfileEvents`:

- `SelectedParts` / `SelectedRanges` / `SelectedMarks` — how aggressive the index pruning was.
- `ReadCompressedBytes` / `ReadUncompressedBytes` — I/O volume.
- `FilteredRowsByPrewhere` — PREWHERE effectiveness.
- `MergeTreePartsLockHoldMicroseconds` — contention on metadata.
- `OSCPUVirtualTimeMicroseconds` / `OSCPUWaitMicroseconds` — CPU vs. wait.
- `NetworkSendBytes` / `NetworkReceiveBytes` — Distributed cost.
- `QueryCacheHits` — if the query cache was enabled.

Compare a fast vs. slow run; the diffs reveal the bottleneck.

## 11.13 Common rewrites

### Filter early

```sql
-- bad: filter after join
SELECT ... FROM a JOIN b ON ... WHERE a.ts > now() - INTERVAL 1 DAY;

-- good: filter inside subquery
SELECT ... FROM (
    SELECT * FROM a WHERE ts > now() - INTERVAL 1 DAY
) a JOIN b ON ...;
```

The optimizer often does this for you, but verify with EXPLAIN.

### Move filter to PREWHERE

Already covered above.

### Replace JOIN with dictionary

Already covered in [10](10-joins.md).

### Replace SELECT FINAL with argMax / LIMIT 1 BY

Already covered in §11.4.

### Replace `count(distinct x)` with `uniq(x)`

```sql
SELECT uniq(user_id) FROM events;  -- HLL, ~1% error
```

### Push GROUP BY into a materialized view

If the same aggregation is computed many times, materialize it.

### Skip the final merge for Distributed if shards return disjoint groups

```sql
SET distributed_group_by_no_merge = 1;
SELECT shardNum(), key, sum(x) FROM dist GROUP BY key;
```

When sharding key aligns with grouping key.

## 11.14 Query cache (newer)

ClickHouse 23.x added a query result cache:

```sql
SET use_query_cache = 1;
SET query_cache_ttl = 60;

SELECT ... -- cached for 60s
```

Useful for dashboards refreshing the same query repeatedly. Disabled by default (mutations would create staleness for non-MV-aware caches; it's tunable).

## 11.15 Settings profiles and quotas

For multi-tenant or BI workloads:

```sql
CREATE SETTINGS PROFILE dashboard_user
SETTINGS max_memory_usage = '2Gi', max_execution_time = 30, readonly = 1;

CREATE QUOTA dashboard_quota
FOR INTERVAL 1 MINUTE MAX QUERIES = 100, MAX RESULT_BYTES = '10Gi';
```

Limit blast radius from a runaway BI query.

## 11.16 Anti-patterns

- `SELECT *` on a wide table.
- `OPTIMIZE FINAL` to "fix" perf — almost never the right tool.
- `FINAL` in hot paths.
- Forgetting to use PREWHERE on cheap, selective predicates.
- Running heavy queries with `max_threads = 0` (defaults) on a server serving many users — causes thrashing.
- Mutations that touch many parts during dashboard load.

## 11.17 Must-internalize

- EXPLAIN PLAN / PIPELINE / ESTIMATE — your first tool.
- PREWHERE auto-promotes; verify and override when needed.
- Avoid FINAL; use argMax / LIMIT 1 BY / subquery patterns.
- Cardinality and selectivity drive everything; check before optimizing.
- Approximate aggregates (uniq, quantileTDigest) for 1% error → 10× speed.
- Spill to disk via `max_bytes_before_external_*` for memory survival.
- Materialized views and projections are usually a better answer than query tuning.

---

## Sources

- [Query optimization definitive guide (2026)](https://clickhouse.com/resources/engineering/clickhouse-query-optimisation-definitive-guide)
- [PREWHERE optimization](https://clickhouse.com/optimize/prewhere)
- [Performance tuning — oneuptime](https://oneuptime.com/blog/post/2026-02-20-clickhouse-performance-tuning/view)
- [System tables: query_log, parts, asynchronous_metrics](https://clickhouse.com/docs/operations/system-tables)
- [Approximate aggregations](https://clickhouse.com/docs/sql-reference/aggregate-functions/reference)
