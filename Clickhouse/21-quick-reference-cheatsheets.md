# 21 · Quick Reference Cheatsheets

The night-before review. One section per topic family.

## 21.1 Engine matrix

| Engine | Purpose | Notes |
|--------|---------|-------|
| **MergeTree** | Default OLAP | Sorted parts, sparse PK |
| **ReplacingMergeTree(version)** | Latest-state dedup | Pair with argMax/LIMIT 1 BY for reads |
| **SummingMergeTree([cols])** | Auto-sum on merge | Cheaper than AggregatingMergeTree |
| **AggregatingMergeTree** | States via *State / *Merge | MV target |
| **CollapsingMergeTree(sign)** | +1/-1 cancel | Out-of-order fragile |
| **VersionedCollapsingMergeTree(sign, ver)** | Same with version | Robust |
| **GraphiteMergeTree** | Graphite-style rollup | Niche |
| **Replicated*** | Adds replication | Keeper-coordinated |
| **SharedMergeTree** | Cloud leaderless on S3 | Stateless compute |
| **Distributed** | Routing/virtual | No storage |
| **Kafka** | Consume topic | Pair with MV → MergeTree |
| **Dictionary** | KV lookup | `dictGet`, direct join |
| **MaterializedView** | Insert-time trigger | Pair with target table |
| **RefreshableMaterializedView** | Scheduled refresh | APPEND or REPLACE |
| **Buffer** | RAM staging | Legacy; prefer async_insert |
| **MaterializedPostgreSQL / MySQL** | CDC ingest | Logical replication |
| **S3 / HDFS / Iceberg / Delta / URL / File** | Foreign storage | Table functions |

## 21.2 Data type cheatsheet

| Type | Bytes | Use |
|------|-------|-----|
| Int/UInt 8/16/32/64/128/256 | 1/2/4/8/16/32 | Pick smallest with headroom |
| Float32 / Float64 | 4 / 8 | Metrics; not money |
| Decimal32/64/128/256(scale) | 4/8/16/32 | Money |
| String / FixedString(N) | var / N | Text |
| LowCardinality(T) | dict-encoded | Strings with < millions distinct |
| Enum8 / Enum16 | 1 / 2 | Fixed-set strings |
| Date / Date32 | 2 / 4 | Day precision |
| DateTime | 4 | unix-second, to 2106 |
| DateTime64(p) | 8 | Sub-second |
| UUID | 16 | Random, doesn't compress |
| IPv4 / IPv6 | 4 / 16 | Network functions |
| Array(T) | var | Multi-value |
| Tuple(...) | var | Fixed shape |
| Map(K,V) | var | Open-key |
| Nested(...) | var | Parallel arrays |
| JSON / Dynamic / Variant | var | Semi-structured |
| Nullable(T) | T + mask | Use sentinels when possible |

## 21.3 Codec cheatsheet

| Codec | Best for |
|-------|----------|
| LZ4 | Default; fast |
| ZSTD(1-3) | Better ratio, still fast |
| ZSTD(5-9) | Best ratio, slower write |
| ZSTD_QAT | Hardware-accelerated |
| Delta(N) | Sorted integers / timestamps |
| DoubleDelta | Evenly spaced timestamps |
| Gorilla | Slowly-varying floats |
| T64 | Integers with variable magnitude |
| FPC | Floats (alt) |
| NONE | Pre-compressed data |

Recipes:
- DateTime → `CODEC(Delta, ZSTD(1))`
- Float metric → `CODEC(Gorilla, ZSTD(1))`
- Variable-magnitude UInt → `CODEC(T64, ZSTD(1))`
- Free text → `CODEC(ZSTD(3))`
- UUID → `CODEC(LZ4)` (incompressible)

## 21.4 Skip-index types

| Type | Use |
|------|-----|
| `minmax` | Range filters on non-PK numeric/date |
| `set(N)` | `IN`/`=` on small distinct sets |
| `bloom_filter(p)` | Equality on high-cardinality |
| `tokenbf_v1(size, hashes, seed)` | `LIKE '%word%'` substring |
| `ngrambf_v1(n, size, hashes, seed)` | Substring with n-grams |
| `text` (26.2+) | Full-text-style |

All use `GRANULARITY N` (groups of N×8192 rows).

## 21.5 Joins quick pick

| Right side | Pick |
|------------|------|
| Small fixed-schema lookup | Dictionary + `dictGet` or direct join |
| Small in-memory | `hash` |
| Large + RAM | `parallel_hash` |
| Large + no RAM | `grace_hash` |
| Both sorted by key | `full_sorting_merge` |
| Lowest memory | `partial_merge` |
| Time-series enrichment | `ASOF JOIN` |
| Distributed | `GLOBAL JOIN` / `GLOBAL IN` |

## 21.6 Quantile / uniq pick

| Need | Function |
|------|----------|
| Exact uniques | `uniqExact(x)` |
| Approx uniques | `uniq(x)`, `uniqCombined(x)`, `uniqHLL12(x)` |
| Exact quantile | `quantileExact(0.99)(x)` |
| Approx quantile | `quantileTDigest(0.99)(x)` |
| Top-K | `topK(N)(x)` |
| Heavy hitters states | `topKState`, `topKMerge` |
| Bitmap sets | `groupBitmapState(x)`, `groupBitmapAnd/Or/Andnot` |

## 21.7 The 30-second pre-interview

- **MergeTree = columnar + sorted-immutable parts + sparse PK + background merge.**
- **ORDER BY** is the primary index; low-cardinality high-selectivity first.
- **8192** rows per granule; **65,536** rows per execution block; **150/300** parts thresholds.
- **PREWHERE** filters column-wise; auto-promoted in many cases.
- **Avoid FINAL** in hot paths; use `argMax`/`LIMIT 1 BY`.
- **Materialized views** are insert-time triggers; pair with **AggregatingMergeTree**.
- **Always GROUP BY + *Merge** when reading AggregatingMergeTree.
- **Replication via Keeper**; **SharedMergeTree** in Cloud removes per-replica storage.
- **Dictionaries** replace most small-table joins.
- **Use `GLOBAL IN`/`GLOBAL JOIN`** for distributed subqueries.
- **Batch inserts** or **async_insert**; never trickle.
- **LowCardinality** anywhere with < a few million distinct strings.
- **Delta + ZSTD(1)** on timestamps; **Gorilla + ZSTD(1)** on slow-varying floats.
- **TTL DELETE / MOVE** at part-granularity, partition by time accordingly.
- **Approximate aggregates** (`uniq`, `quantileTDigest`, `topK`) are usually right.
- **Don't OPTIMIZE FINAL** in prod.
- **System tables**: `query_log`, `parts`, `merges`, `mutations`, `replicas`, `replication_queue`.

## 21.8 Settings to remember

| Setting | Use |
|---------|-----|
| `max_threads` | Per-query parallelism |
| `max_memory_usage` | Per-query cap |
| `max_bytes_before_external_group_by/sort` | Enable spill-to-disk |
| `join_algorithm` | Adaptive: `'auto'` |
| `mutations_sync` | 0 async, 1 local-wait, 2 all-replicas-wait |
| `async_insert` | Server-side batching |
| `use_query_cache` | Cache result for repeated dashboard queries |
| `select_sequential_consistency` | Pair with `insert_quorum` for strong consistency |
| `distributed_product_mode` | Force `'global'` or `'deny'` to catch wrong joins |
| `optimize_read_in_order` | Skip sort when ORDER BY matches PK |

## 21.9 System table top 10

1. `system.processes` — running queries
2. `system.query_log` — completed queries
3. `system.parts` — part-level storage
4. `system.merges` — in-flight merges
5. `system.mutations` — DDL background tasks
6. `system.replicas` — replica state
7. `system.replication_queue` — replication backlog
8. `system.clusters` — topology
9. `system.metrics` / `events` / `asynchronous_metrics` — metrics
10. `system.parts_columns` — per-column storage

## 21.10 The 5 cloud-specific terms

- **SharedMergeTree** — Cloud engine.
- **Stateless compute** — pods with no durable state.
- **File cache** — local SSD layer.
- **Service** — tenant's logical cluster.
- **Auto-pause / auto-scale** — Cloud cost knobs.

## 21.11 Things you should NOT say

- "OPTIMIZE FINAL fixes it."
- "Let's add an index on every column."
- "JOIN the two big distributed tables."
- "Store timestamps as String for readability."
- "Use Nullable everywhere."
- "Run mutations during peak hours."
- "Single Keeper is fine, it's just metadata."
- "Use FINAL in a hot dashboard."

## 21.12 Things you SHOULD say

- "Pick the ORDER BY for the dominant filter pattern."
- "Pre-aggregate via MV + AggregatingMergeTree."
- "Replace small joins with dictionaries."
- "Approximate aggregates for 1% error and 10× cheaper."
- "Codecs matter — Delta on timestamps, Gorilla on slow floats."
- "Partition for lifecycle, not pruning."
- "Use Keeper standalone, 3 nodes."
- "GLOBAL IN/JOIN for distributed correctness."
- "Batch your inserts."
- "Wire alerts for parts, merges, mutations, replicas — before prod."
- "On Cloud, SharedMergeTree separates compute from storage; replicas are leaderless on S3."
- "Always look at `system.query_log` and `EXPLAIN PLAN indexes=1` first."

Breathe.
