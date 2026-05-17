# 10 · Joins — All Six Algorithms and When to Use Each

Joins are the most misunderstood part of ClickHouse for newcomers. ClickHouse has full SQL JOIN support — but the right algorithm depends on shape and size, and "denormalize" or "use a dictionary" beats most joins for analytics workloads.

## 10.1 Mental model: when to JOIN at all

ClickHouse is happiest when the analytical fact lives in **one wide table** and any small dimensions are in **dictionaries**. The first thing to ask in an interview: *do you actually need this JOIN?*

| Alternative | Use when |
|-------------|----------|
| **Denormalize** (write the dimension fields into the fact table) | Dimension is stable / slowly changes |
| **Dictionary** (`dictGet` or direct join) | Small/medium reference table accessed by ID |
| **GLOBAL IN** (semi-join via subquery) | Filter fact by a list from another table |
| **Materialized view** (pre-join at insert) | Same join shape over and over |
| **JOIN** | Genuine ad-hoc analytical join between two big tables |

If the join is unavoidable, the question becomes: *which algorithm?*

## 10.2 Hash join — the default

Most general join algorithm. Two phases:

1. **Build**: stream the *right* side into memory, build a hash table keyed on the join key.
2. **Probe**: stream the *left* side, look up each row in the hash table.

```sql
SELECT a.*, b.country
FROM events AS a
JOIN users  AS b ON a.user_id = b.id;
```

The right table goes into the hash table. **Put the smaller table on the right.** ClickHouse won't pick automatically; 
the wrong choice means OOM.

**Memory**: roughly `right_rows × right_keys_size`. Bound by `max_bytes_in_join`.

**Throughput**: very high if the build side fits.

## 10.3 Parallel hash join

The hash table is built in parallel buckets (default 16):

```sql
SET join_algorithm = 'parallel_hash';
```

Faster build for large right sides. Uses more memory (each bucket has its own table).

When ClickHouse should pick: large right side, plenty of CPU and memory.

## 10.4 Grace hash join — spill-to-disk

If the right side doesn't fit in memory, hash join OOMs. Grace hash partitions both sides into N buckets on disk, joins bucket-by-bucket:

```sql
SET join_algorithm = 'grace_hash';
```

Slower than in-memory hash, but **doesn't OOM**. Use as a fallback for very large right tables.

## 10.5 Full sorting merge join

Sorts both sides on the join key, then streams them in a merge step:

```sql
SET join_algorithm = 'full_sorting_merge';
```

- **Doesn't OOM** — sort can spill to disk (`max_bytes_before_external_sort`).
- **Skips the sort** if either side is already sorted by the join key (e.g., the table's ORDER BY includes it). Massive win.
- Slower than hash for unsorted inputs because sorting both sides is expensive.

When to pick: at least one side is already sorted on the join key. ClickHouse can detect and skip the sort.

## 10.6 Partial merge join

Sort right side, then iterate left side in chunks; for each chunk, sort and merge with right:

```sql
SET join_algorithm = 'partial_merge';
```

- **Lowest memory** of the merge family.
- **Slowest** of the join algorithms. Only when you really can't afford memory.

## 10.7 Direct join

For special right-side tables (`Dictionary`, `Join`, `EmbeddedRocksDB`) that already support key-value lookup, you can skip the build phase entirely.

```sql
SET join_algorithm = 'direct';

SELECT a.user_id, dictGet('users_dict', 'country', a.user_id) AS country
FROM events AS a;
```

Or, more idiomatically, just `dictGet`. The "direct join" path uses the lookup structure directly during probe.

- **Fastest** for the case it supports.
- **Constraints**: only LEFT ANY join, only specific right-side engines.

## 10.8 The `join_algorithm = 'auto'` mode

ClickHouse can adaptively pick at runtime:

```sql
SET join_algorithm = 'auto';
```

Starts with hash; if `max_bytes_in_join` would be exceeded, falls back to grace_hash or partial_merge. Reasonable default for general workloads.

## 10.9 Join semantics — the gotchas

### Default strictness is ALL (vs SQL's first-match-only ambiguity)

```sql
-- in ClickHouse this returns ALL matching rows from b for each a row
SELECT * FROM a JOIN b ON a.id = b.id;
```

`SELECT ... JOIN ANY` returns at most one match per left row (the equivalent of "any of the matches").

`ASOF JOIN` joins on the closest match in a sorted sequence — useful for time-series enrichment.

### LEFT / RIGHT / INNER / FULL / CROSS

All supported. RIGHT and FULL are slower (they have to track unmatched right-side rows).

### ANY / ALL / ASOF / SEMI / ANTI

- `ANY` — at most one match.
- `ALL` — all matches (Cartesian if multiple).
- `ASOF` — closest-match on a sortable column.
- `SEMI` — left rows that have at least one right match; no right columns returned.
- `ANTI` — left rows that have **no** right match.

### Multi-column join keys

`ON a.x = b.x AND a.y = b.y` works. The hash key is a tuple.

### Non-equi joins

`ON a.x > b.x` is supported but slow — it becomes a Cartesian-with-filter. Use range-based equi-join hacks where possible.

## 10.10 GLOBAL JOIN vs. JOIN in distributed

A `JOIN` in a Distributed query runs *on each shard locally*. If the right side is a regular table, each shard joins against its local copy.

If the right side is a Distributed table or subquery, you usually want **GLOBAL JOIN**:

```sql
SELECT ...
FROM events_dist AS a
GLOBAL JOIN (
    SELECT user_id, country FROM premium_users
) AS b ON a.user_id = b.user_id;
```

The subquery runs once on the initiator, the result is sent to every shard, then the per-shard local hash join proceeds.

**Anti-pattern**: a plain JOIN on a Distributed right side without GLOBAL — each shard re-runs the right query in full. N× the work.

## 10.11 Dictionary as the join replacement

The fastest analytical lookup is `dictGet` against an in-memory dictionary.

```sql
CREATE DICTIONARY users_dict (...) ...; -- see [03]

SELECT
    event,
    dictGet('users_dict', 'country', user_id) AS country,
    count()
FROM events
WHERE ts > now() - INTERVAL 1 DAY
GROUP BY event, country;
```

- O(1) lookup per row, no hash-build cost per query.
- Loaded once per server; reused.
- Best for small/medium reference data (a few million keys max for HASHED; more for CACHE/SSD_CACHE).

**Rule**: if a join's right side is a slowly-changing dimension, make it a dictionary.

## 10.12 Worked example — pick the right algorithm

| Right table size | Sorted on join key? | Memory budget | Best algorithm |
|------------------|---------------------|---------------|----------------|
| Tiny (< 100K rows), known set | n/a | n/a | **Dictionary** (`dictGet`) or `IN` |
| Small (< 100M rows) | no | comfortable | **hash** (default) |
| Medium (100M–few B) | no | tight | **parallel_hash** if CPU; **grace_hash** if mem-bound |
| Very large | yes (both sides sorted by key) | tight | **full_sorting_merge** (skips sort) |
| Very large | no | very tight | **partial_merge** |
| Small lookup with KV semantics | n/a | n/a | **direct** with Dictionary / RocksDB |

## 10.13 ASOF JOIN — the time-series special

Join each left row to the *last* right row with a smaller-or-equal value on a sort column.

```sql
SELECT
    e.ts, e.user_id, e.event,
    s.session_id
FROM events e
ASOF LEFT JOIN sessions s
ON  e.user_id = s.user_id
AND e.ts >= s.start_ts
ORDER BY e.ts;
```

For each event, finds the most recent session that started at or before the event. Common for sessionization, attribution, ML feature enrichment.

## 10.14 Anti-patterns

- **Joining two big tables without a plan.** Always know the build/probe sides; know the memory budget.
- **Right side larger than left.** Swap them.
- **JOIN inside a subquery on a Distributed table without GLOBAL.** Catastrophic.
- **Using JOIN for what should be a dictionary.** 10× slower.
- **`JOIN ALL` against a duplicated dimension** → row blow-up.
- **CROSS JOIN by accident** (missing ON / mismatched aliases).
- **SELECT \* with JOIN** → reads all columns from both sides, costing column-store advantages.

## 10.15 Diagnosing a slow join

```sql
EXPLAIN PLAN actions=1
SELECT ...;

-- pipeline view
EXPLAIN PIPELINE
SELECT ...;

-- after run
SELECT
    query_duration_ms, memory_usage,
    ProfileEvents['JoinBuildTableRowCount']  AS build_rows,
    ProfileEvents['JoinProbeTableRowCount']  AS probe_rows
FROM system.query_log
WHERE query_id = '...'
ORDER BY event_time DESC LIMIT 1;
```

Look for: build/probe side sizes (is the smaller one on the right?), join algorithm chosen, fallbacks to disk.

## 10.16 Must-internalize

- Default = hash join; put small side on right.
- parallel_hash for CPU; grace_hash for memory; merge variants when sorted; direct for dictionary-like right sides.
- Use `join_algorithm = 'auto'` for ergonomic adaptiveness.
- GLOBAL JOIN / GLOBAL IN for distributed correctness.
- Replace small joins with dictionaries.
- ASOF JOIN for time-aligned enrichment.
- Denormalization / pre-joining via MV beats most repeated joins.

---

## Sources

- [JOINs guide — official](https://clickhouse.com/docs/guides/joining-tables)
- [Joins under the hood — Hash + Parallel + Grace (part 2)](https://clickhouse.com/blog/clickhouse-fully-supports-joins-hash-joins-part2)
- [Joins under the hood — Merge (part 3)](https://clickhouse.com/blog/clickhouse-fully-supports-joins-full-sort-partial-merge-part3)
- [Joins under the hood — Direct (part 4)](https://clickhouse.com/blog/clickhouse-fully-supports-joins-direct-join-part4)
- [Choosing the right join algorithm (part 5)](https://clickhouse.com/blog/clickhouse-fully-supports-joins-how-to-choose-the-right-algorithm-part5)
- [GlassFlow — ClickHouse joins explained](https://www.glassflow.dev/blog/clickhouse-joins)
