# Faceted Search at Scale — Inverted Indexes, Bitmap Intersection, Distributed Aggregation

How systems like Datadog, Splunk, and Elasticsearch deliver sub-second filtering and counts across **20+ dimensions** over **trillions of log events**, while serving **100 K+ concurrent users**. Anchored on observability/log analytics, but the patterns generalize to any high-cardinality multi-dimensional filtering problem: e-commerce product search, security event analytics, fraud investigation tools, ad analytics dashboards.

The user-facing illusion is "type a filter, see counts update in 200 ms." The mechanics underneath: a layer cake of inverted indexes, bitmap algebra, columnar reads, approximation algorithms, scatter-gather, and aggressive caching, each amortizing a different cost dimension.

---

## 1. The Problem, Concretely

A Datadog-class log query looks like:
```
service:auth-service env:prod status:>=500 region:us-east-1
   AND error.type:TimeoutException
   AND host:i-0abc...
```
…and the UI must show, *for every dimension simultaneously*:
- Top values + counts under the current filter (`42 385 ERROR`, `8 234 auth-service`, `3 102 us-east-1`).
- Updates as the user toggles facets, in < 500 ms p95.

### Scale axes
| Axis | Order of magnitude |
|---|---|
| Events/day | 10¹² (12 M / sec sustained, 50 M / sec peak) |
| Distinct dimensions | 20–200 |
| Cardinality per dimension | 10² (status codes) to 10⁹ (container IDs) |
| Storage | Hot tier 100s of TB, warm/cold PB, lifetime EB |
| Concurrent queries | 10⁵ |
| Per-query budget | < 500 ms p95 |

### Why naive approaches die
- **Full scan**: 1 PB / 500 ms = **2 PB/s** of I/O per query. Impossible.
- **Per-row B-tree per dimension**: 20 indexes × 10¹² rows ≈ 10¹³ index entries written per dimension per day. Write amplification eats the disks before reads ever start.
- **Pre-aggregated cubes** (OLAP-style): 20 dimensions × ~10⁵ values = 10²⁶ possible cube cells. Sparse, but the cubes themselves are PB-scale and stale.
- **In-memory stores**: 10¹² events × 1 KB = 1 EB. RAM is not arriving.

The actual answer is a stack of structures, each of which solves a sub-problem cheaply, and a query planner that composes them.

---

## 2. The Inverted Index — The Foundation

The unifying primitive: for every (`field`, `term`) pair, store the **sorted list of document IDs** that contain it.

```
service:auth-service   →  [1, 7, 9, 11, 18, 22, 27, 31, ...]
service:payments       →  [2, 3, 8, 12, 14, ...]
status:500             →  [1, 4, 9, 11, 17, 22, ...]
```

Filter `service:auth-service AND status:500` becomes:
```
intersect([1, 7, 9, 11, 18, 22, 27, 31, ...], [1, 4, 9, 11, 17, 22, ...])
        = [1, 9, 11, 22, ...]
```

This is the move. Filtering by a dimension stops being "scan rows checking field=value" and becomes "load a precomputed list and intersect with other lists."

### 2.1 Lucene-style segment layout

Each shard is a collection of **immutable segments** (Lucene's structure, copied by Elasticsearch, OpenSearch, Solr, and conceptually by everyone else):

```
segment/
├── .tim, .tip   term dictionary + index    (FST: from term → postings file offset)
├── .doc         postings (doc IDs)
├── .pos         positions (for phrase queries)
├── .pay         payloads / offsets
├── .dvd, .dvm   doc values (column-store of per-doc field values, for aggregations / sort)
├── .fdt, .fdx   stored fields (the original document)
├── .nvd, .nvm   norms (for scoring)
└── .liv         live-docs bitset (deleted markers)
```

Two structures matter for faceted search: **postings** (for filtering) and **doc values** (for aggregations).

### 2.2 The term dictionary — FST

Mapping `term → postings offset` for billions of unique terms requires a structure that's:
- Memory-efficient (mmap'd, can't blow heap).
- Fast to look up exactly.
- Supports range / prefix scans (`status:>=500`).

**Finite State Transducer (FST)** — a minimal acyclic automaton that maps strings to outputs, sharing prefixes *and* suffixes. ~10× more compact than a sorted-string table for natural-language-like terms. Lucene uses FSTs for the in-memory term-index file.

For dimensions where terms are dense integers (status codes, ports), a different layout — sorted block of fixed-width values — is cheaper. Most engines use a hybrid.

### 2.3 Postings list compression

A postings list is a sorted ascending sequence of `doc_id`s. Two ideas stack:

**(a) Delta encoding** — store gaps, not absolute IDs:
```
[1, 9, 11, 22, 41]  →  [1, 8, 2, 11, 19]    // gaps
```
Gaps are smaller than IDs → encode in fewer bits.

**(b) FOR / PFOR-Delta** — pack each block of N gaps in *just enough bits* to hold the max gap in the block.
- Block size: 128 (Lucene default).
- For each block: compute `bits_per_value = ceil(log2(max_gap))`; pack the block as a bit-stream.
- Outliers (PFOR variant) stored separately so one giant gap doesn't blow up the block's bit-width.

Compression ratios in practice: postings lists shrink to **< 10 %** of naive 64-bit encoding.

### 2.4 Skip lists for fast intersection

Intersecting a long postings list (`status:200`, billions of entries) with a short one (`error.type:OOM`, thousands) doesn't need to scan every entry of the long list. A **skip list** (Lucene calls them "skip data") stores `(doc_id, file_offset)` waypoints every K entries (K = 8 or 16). To advance to a target ID `t`, jump on the skip list to the largest waypoint ≤ t, then linear-scan within the block.

Intersection of a list of length N with one of length M (M ≪ N) becomes O(M log(N/K)) instead of O(N). The "leapfrog" intersection pattern.

### 2.5 Multi-term lookups and wildcards

- **Range** (`status:>=500`): walk the FST from `500` to `599`; OR the postings lists.
- **Prefix** (`error.type:Timeout*`): walk all FST states reachable from the prefix; OR.
- **Wildcard** middle (`*Exception`): bad — requires walking every term. Engines build a **n-gram index** in addition for fast contains/wildcard, at storage cost.

---

## 3. Bitmap Indexes — Where the Real Speed Comes From

For low-/medium-cardinality dimensions (env, status, region, level), a more compact and faster representation exists: **a bitmap per term**.

```
env:prod       →  bitmap with bit i set ⟺ doc i has env=prod
status:500     →  bitmap with bit i set ⟺ doc i has status=500
```

Filter intersection is **bitwise AND**:
```
result = bitmap[env=prod] AND bitmap[status=500] AND bitmap[region=us-east-1]
```

Bitwise AND on packed words is the fastest intersection physics allows: 256 bits/cycle on modern CPUs with AVX-2, more with AVX-512. A 1 B-doc bitmap in 125 MB is ANDed in a few hundred ms with a *single* core; SIMDed and parallelized, milliseconds.

The catch: a 10⁹-doc bitmap is 125 MB even when only a hundred bits are set. We need compression that preserves random-access AND.

### 3.1 Compressed bitmap formats

**WAH (Word-Aligned Hybrid)**: run-length encoding at word boundaries. ~2× faster than uncompressed for sparse bitmaps; AND/OR work directly on the compressed form.

**EWAH (Enhanced WAH)**: WAH plus tightly-packed marker words. Long-time default; superseded by Roaring.

**Roaring bitmaps** — the modern winner, used by Lucene (after Lucene 5), Druid, ClickHouse-via-extension, InfluxDB, Pinot, and most "fast OLAP" systems:

```
Roaring layout:
  Doc IDs split into 16-bit high / 16-bit low parts.
  For each high part with at least one set bit, choose container by density:
    - Array container:  if < 4096 set bits, store sorted uint16 list of low parts.
    - Bitmap container: if ≥ 4096 set bits, store dense 8 KB bitmap (= 65536 bits).
    - Run container:    if many consecutive set bits, store as runs (start, length).
```

Properties:
- AND / OR pick the fastest algorithm per container pair (array∧array = sort-merge intersect; bitmap∧bitmap = SIMDed AND; array∧bitmap = check each array entry against bitmap).
- Compression ratio: **5–100× over WAH** for typical real-world distributions.
- Random access (test bit i): O(1).
- Cardinality (popcount): incremental; maintained on insert.

### 3.2 Why bitmaps and postings coexist

For high-cardinality fields (`container_id`, `request_id`, `user_id`), one bitmap per value would be billions of bitmaps — impractical. Stick with postings lists.

For low-/medium-cardinality fields and aggregations, bitmaps win. Lucene's modern strategy: **postings + bitmap when the postings would be denser than bitmap.** This is invisible to the query layer; the index decides per-term.

### 3.3 Bitmap-based filter caching

Once a query computes `bitmap[env=prod AND region=us-east-1]`, that intermediate is *itself* a bitmap. Cache it keyed by the canonical filter string. Subsequent queries with the same filter get the bitmap free; their incremental work is just ANDing with the new dimension. Elasticsearch's "filter cache" and Druid's "filter cache" are exactly this.

---

## 4. Doc Values — Column Store for Aggregations

Filtering produces a set of doc IDs. Computing facet counts (`how many of these matched have service=auth-service?`) requires reading the field value for each matched doc.

Reading values **row-wise** from stored fields is wrong: it pulls every other field's bytes too.

Engines maintain a **column store** alongside the inverted index — Lucene calls them **doc values**. Per field:
```
doc_id 0  →  service: 17  (ordinal into a per-segment dictionary)
doc_id 1  →  service: 17
doc_id 2  →  service: 42
...
```
Stored as packed integer columns (often LZ4 + delta + bit-packing), compressed by ~10×, sequentially scannable.

For faceting on dimension D over result-set R (a bitmap):
```
counters = HashMap<ordinal, count>
for doc_id in R:                    // iterate set bits of bitmap
    ord = docvalues[D][doc_id]      // O(1) random access
    counters[ord] += 1
return top_K(counters)
```

### 4.1 Global ordinals — across-segment merging

Each segment has its own value→ordinal mapping. To aggregate across all segments in a shard, the engine builds a **global ordinal map**: per-segment ordinal → shard-wide ordinal. Built lazily on first aggregation; cached. This is why a "first faceted query after a refresh" can be slow — global ordinals warm up.

For high-cardinality columns (`user_id`), eager global ordinals can cost more than the query saves. Tunable per field.

### 4.2 Top-K within a shard

Aggregating `top 10 services` in a shard with 10 K distinct services: maintain a heap, not a full sort. O(N log K).

For the famous-faces problem (`top 10 IPs across 10⁹ events with 10⁸ distinct IPs`), exact heaps blow up — see §6 (approximation).

---

## 5. The Faceting Algorithm — End to End

Putting filter + aggregate together, for one shard:

```
def query(shard, filter_clauses, facet_dimensions, top_k):
    # 1. Build the result bitmap by intersecting per-clause bitmaps
    bm = ones_bitmap()
    for (field, value) in filter_clauses:
        bm &= bitmap_index.get(field, value)        // or postings → bitmap on demand

    # 2. Optionally cache `bm` keyed by filter
    cache.put(filter_canonical, bm)

    # 3. For each facet dimension, scan doc values restricted to bm
    facets = {}
    for d in facet_dimensions:
        counters = {}
        for doc_id in bm:                            // iterate set bits
            ord = docvalues[d][doc_id]
            counters[ord] = counters.get(ord, 0) + 1
        facets[d] = top_k(counters, top_k)

    return {
        total: bm.cardinality(),
        facets: facets,
    }
```

Three constants to tune obsessively:
- **Bitmap representation** (Roaring vs uncompressed) — depends on density.
- **Iteration order** (bitmap vs postings) — bitmap iteration is faster for dense results; postings iteration is faster for sparse intersection.
- **Doc-values block size** — smaller blocks → less data read for sparse results, more random I/O; tune for the workload.

---

## 6. Approximation — When Exact Isn't Worth the Cost

At 10¹² events, exact answers stop earning their cost. Three approximate-algorithm pillars:

### 6.1 Cardinality estimation: HyperLogLog (HLL)

"How many distinct user_ids matched?"

HLL: hash each value to a 64-bit int; use the first `p` bits to pick a register (2^p registers); track the max number of leading zeros in the rest within each register. Estimate cardinality from the harmonic mean of register values.

Properties:
- **Standard error ≈ 1.04 / √m** where m = 2^p. p=14 → m=16384 registers → ~0.8% error, **16 KB per HLL**.
- Mergeable: `HLL(A ∪ B) = register-wise max(HLL(A), HLL(B))` — perfect for distributed scatter-gather.
- Insertable: O(1) per element.
- Used everywhere: Druid `hyperUnique`, Elasticsearch `cardinality` agg, ClickHouse `uniqHLL12`, BigQuery `APPROX_COUNT_DISTINCT`.

For a query like `cardinality(user_id) where service:auth-service and region:us-east-1`, each shard maintains a per-segment HLL (or builds one on the fly from doc values), the coordinator merges them register-wise. **One number per shard, regardless of cardinality.**

### 6.2 Top-K with Count-Min Sketch + heap

Exact top-K of `error.type` over 10⁹ events with 10⁶ distinct types: too much memory.

**Count-Min Sketch (CMS)**: small 2D array of counters. Hash each item with `d` independent hash functions; increment the corresponding counter in each row. Estimate count = min over rows. Combined with a heap of "likely top-K candidates" → approximate top-K with bounded memory.

Better in practice: **Misra-Gries** (Space-Saving) — a streaming summary that maintains exactly K counters; new items replace the smallest. Mergeable across shards. Used by Druid `topN` and Elastic `terms` agg with `shard_size`.

### 6.3 Percentiles: T-Digest

"p99 latency over the matched set" — naive: sort everything (O(N log N)) and pick the p99 index. At 10⁹ matched docs, infeasible.

**T-Digest**: hierarchical centroid summary. Inserts in O(log N) amortized; quantile estimation in O(1); space ~5 KB per digest with strong accuracy at the tails (where percentiles matter). Mergeable.

Used by Elastic `percentiles` agg, Druid `quantilesDoublesSketch`, Datadog histograms internally.

### 6.4 The trade-off

Approximate answers come with:
- **Bounded error** (often documented in the API).
- **Constant memory per dimension per shard** (regardless of distinct values).
- **Mergeability** (essential for distributed aggregation).
- **No "exact" option past a threshold** — Elasticsearch `terms` agg with `size: 100, shard_size: 1000` returns approximate top-100 because each shard returns its top 1000 and the coordinator merges; rare values that happen to be globally top-100 but not top-1000 in any single shard can be missed.

This is the price of horizontal scale. Document it; expose error bounds in the UI ("≈" or shard-confidence intervals).

---

## 7. Distributed Aggregation — Scatter-Gather

A shard fits ≤ 50 GB. PB-scale data = 10⁵ shards. Queries fan out and reduce.

```
Coordinator
   │
   ├──▶ Shard 1   (filter, agg, top-K, HLL, T-Digest)  ──▶ partial result
   ├──▶ Shard 2                                         ──▶ partial result
   ├──▶ Shard 3                                         ──▶ partial result
   │     ...
   └──▶ Shard N
   ◀── merge partials ──
   │     - sum counters
   │     - merge HLLs (register-wise max)
   │     - merge T-Digests
   │     - re-rank top-K from union
   │
   final result
```

### 7.1 Coordinator merge specifics

- **Sum aggregations**: trivial.
- **Top-K terms**: each shard returns its top `shard_size` (typically ~10× the requested size for accuracy); coordinator merges all (term, count) pairs; sums collisions; re-extracts global top-K.
- **HLL union**: register-wise max — cheap, lossless w.r.t. each individual HLL.
- **T-Digest merge**: append + compress to budget — bounded loss.

### 7.2 Pre-filtering shards

For time-bucketed indexes (every observability system), each shard's metadata advertises its time range. Queries with time predicates **skip irrelevant shards entirely**. A query for "last 15 minutes" out of 30 days of indexes touches < 0.1 % of shards. This is the single largest production win.

Other pre-filter dimensions: `service`, `env` — if shards are partitioned by service, only relevant shards see the query.

### 7.3 Shard placement and routing

- **Time-partitioned indexes**: one index per day or hour. Old indexes are read-only.
- **Service-routed shards** (custom routing key on ingest): logs for a service co-locate; service-filtered queries hit fewer shards.
- **Replicas for query parallelism**: round-robin reads across replicas to scale concurrent query load.

### 7.4 Partial results / failure handling

A query against 10⁵ shards: at any moment some shards are slow / dead / restarting. Choices:
- **Wait for all** — tail-latency-dominated; a single slow shard makes every query slow.
- **Return partial** with `_shards.failed > 0` exposed to caller — fast, sometimes wrong.
- **Hedged requests** — issue to two replicas, take the first. Doubles load, kills tail.
- **Adaptive timeout** + fallback to "approximate" mode.

Datadog's stated approach: aggressively bound per-shard time, surface partiality in UI ("based on 92% of data"), allow user to retry slower-but-complete.

---

## 8. Real Production Architectures

### 8.1 Datadog Husky

Datadog publicly described migrating off Elasticsearch to **Husky**, their custom ingestion + storage + query engine, motivated by elasticity and cost.

Key properties (from public talks):
- **Disaggregated compute and storage**: data on object storage (S3 / GCS), workers stateless.
- **Columnar Parquet-like format** for cold storage.
- **Worker pool sized to query load**, not data volume — fundamental cost lever vs Elasticsearch's "you store it, you pay for the JVM."
- **Real-time tier** (in-memory + local SSD) for last few minutes; **historical tier** (object store) for older.
- **Query planner** decides, per query, which workers fetch which Parquet files; Bloom filters and zone maps prune at file level.
- **Approximate by default**: HLL, sketches throughout.

### 8.2 Splunk SmartStore

- **Buckets**: ~10 GB time-bucketed data + index files.
- **tsidx files**: Splunk's term-sorted index — the inverted index.
- **Per-bucket Bloom filters** for term existence — coordinator skips buckets that definitely don't have a term.
- **SmartStore**: hot/warm cache on indexer local SSD, cold on S3 — search reads SSD when hot, fetches from S3 on miss.
- **Indexer cluster** for ingest + per-bucket search; **search head cluster** for orchestration + result merging.

### 8.3 Elasticsearch / OpenSearch

- **Lucene-based** segments (§2.1).
- **Indexes** = collection of shards = collection of segments.
- **Refresh interval** controls visibility-of-new-docs (default 1 s, tunable up for write-heavy workloads).
- **Doc values** mmap'd → OS page cache is the aggregation cache.
- **Filter cache** (Roaring bitmaps) on per-segment basis.
- **Eager global ordinals** for known-hot facet fields.
- **ILM** (Index Lifecycle Management): hot → warm → cold → frozen → delete, by age.
- **Frozen tier (searchable snapshots)**: data on S3, queried directly with cache; ~10× cheaper than hot tier, slower.

### 8.4 Apache Druid

Purpose-built for OLAP-on-events:
- **Segments**: time-bucketed columnar files, immutable.
- **Roaring bitmap indexes** per dimension.
- **Dictionary-encoded columns** for low-cardinality.
- **Approximate everywhere**: HLL, T-Digest, Theta sketches.
- **Three query types**: timeseries (time-bucketed agg), topN (single-dim top-K), groupBy (multi-dim, expensive).
- **Tier separation**: real-time nodes (Kafka ingestion + recent data) + historical nodes (older immutable data).

### 8.5 ClickHouse

Columnar, MergeTree storage:
- **Primary index** is sparse (one entry per ~8 K rows = "granule") → tiny in memory, points to byte ranges to scan.
- **Skip indexes** per column: `set` (small distinct sets), `bloom_filter`, `minmax`, `tokenbf` (n-gram bloom for text). Pruning at granule level.
- **Aggregating MergeTree** maintains pre-aggregations during merges → near-instant counts on common dimensions.
- **Materialized views** project new shapes on ingest.
- The contrarian take: with columnar + good pruning, you may not need explicit inverted indexes for many faceting workloads. ClickHouse routinely beats inverted-index systems on log-analytic patterns when the filter is a few high-selectivity predicates.

### 8.6 The convergence

All of the above implement essentially the same algorithmic core:
- Time-bucketed immutable segments / buckets.
- Per-segment inverted-or-bitmap indexes for filtering.
- Per-segment columnar values for aggregation.
- Mergeable approximate sketches.
- Scatter-gather coordinator.
- Tiered storage with object-store cold tier.

The differences are operational (managed vs self-host), economic (compute-storage coupling), and ecosystemic (ingestion, alerting, UI).

---

## 9. Storage Tiering and Index Lifecycle

```
[ Real-time / Hot ]   in-memory + local NVMe
    ↓ rolls over after 1 hour or N docs
[ Warm ]              local SSD, segments merged
    ↓ rolls over after 1–3 days
[ Cold ]              object storage (S3) with caching
    ↓ rolls over after 30+ days
[ Frozen / Searchable Snapshot ]   object storage, no replicas, cache on demand
    ↓
[ Delete ]           after retention
```

Cost lever: hot tier is ~10× the cost per GB of cold; cold tier is ~5× the cost of frozen. 90 % of queries hit < 5 % of the data (the recent tail). Tier accordingly.

Searchable snapshots: the index files live entirely on S3; query nodes lazily fetch what they need into a local cache. First query slow (cache miss); subsequent fast. Acceptable for "investigate an incident from last quarter" workloads.

---

## 10. Caching Strategy

A multi-level cache hierarchy explains how p50 < 100 ms is sustainable:

| Cache | What | Invalidation |
|---|---|---|
| **OS page cache** | Index files via mmap | Implicit (LRU) |
| **Filter cache** | Bitmaps for filter sub-expressions | Per-segment; segment immutable → cache permanent for that segment |
| **Field-data / global-ordinals cache** | Per-field ordinal maps for aggregations | On segment open / close |
| **Query cache** | Final result per (query, time-window, filter) | TTL or on data refresh |
| **Distributed result cache** | Coordinator-level; same query from different users | TTL |

The dominant cost in *real* production is **filter cache hit rate**. Dashboards repeat the same predicates; once cached, subsequent queries are near-free.

Anti-pattern: time predicates that *change every second* (`now-15m`) would defeat caching. Engines round time predicates to bucket boundaries (e.g., last completed minute) for cache reuse.

---

## 11. High-Cardinality Fields — The Hard Case

`request_id`, `trace_id`, `container_id`, `user_id`, free-text `message`. These break naive faceting:
- Per-value bitmaps explode.
- Top-K aggregation is meaningless ("the top 10 of 10⁹").
- Doc-values column is enormous.

### Mitigations

- **Don't facet on it.** Surface as a search field (filter by exact value), not as a "show me the top values" facet.
- **Hash-bucket it.** Index `hash(container_id) % 1024` as a faceted dimension, the raw value as a search-only field. Acceptable for dashboarding.
- **HLL only.** Expose cardinality (`distinct count`) — cheap and useful — without exposing top values.
- **Sample.** Datadog and New Relic both sample high-cardinality dimensions during ingest to control index growth.
- **Deferred materialization.** For `message` text, store full-text inverted index but no doc-values; aggregations over `message` are inherently slow / unsupported.

The hard architectural choice: **decide which dimensions are faceted, which are search-only, before ingest.** Retrofitting a dimension as faceted later requires reindexing.

---

## 12. The 100 K Concurrent User Problem

Splunk's "faceted search handles 100 K+ concurrent users" claim — how?

- **Search head clustering**: the orchestrator tier scales horizontally; user sessions sticky-routed to a search head.
- **Per-tenant query queues with quotas**: noisy tenant cannot starve neighbors. Token-bucket per tenant.
- **Result reuse / dashboard scheduled searches**: dashboards refresh on a server-side schedule once and fan results to all viewers, instead of N clients running N queries.
- **Aggressive caching**: 100 K users running the *same* 5 saved searches share cached results.
- **Read-replica scale-out**: shards have multiple replicas; queries spread across replicas; ingest goes to primary.

The user-facing throughput is mostly cache hits + replicas. The actual aggregation compute is a fraction of nominal QPS.

---

## 13. Trade-Offs and Anti-Patterns

| Decision | Trade-off |
|---|---|
| **Roaring bitmap filter** vs **postings list** | Bitmap: O(1) intersection; high memory for dense results. Postings: skip-list-fast, low memory, more random I/O. Engines pick per-term. |
| **Eager global ordinals** | First-query latency vs steady-state latency. Eager helps known-hot fields; hurts cold ones. |
| **Approximate aggregations by default** | Bounded error vs exactness. Document error in UI; offer exact-mode for critical paths. |
| **Higher refresh interval** | Lower indexing cost vs read-after-write delay. 5 s is the secret default for log volume; 1 s is over-spent. |
| **Bigger shards (50 GB)** vs smaller (5 GB) | Bigger: better compression, fewer per-query coordination costs; longer recovery. |
| **Per-index per-day** vs **per-week** | Per-day: more shards, finer pruning. Per-week: less metadata. Match retention granularity. |
| **Frozen tier** | Order-of-magnitude cheaper, queries are slower and less concurrent. Acceptable for compliance-only data. |
| **Force-merge old segments** | Better steady-state query, costly one-time CPU + I/O. Schedule overnight. |

### Anti-patterns to flag

- **Faceting on high-cardinality fields** with no sampling/hash-bucketing — silently destroys index size and query latency.
- **Time predicates with second-resolution `now`** — defeats result caching; round to nearest minute.
- **Wildcard facets** (`error.type:*`) — every term scanned; not a facet, a fishing expedition.
- **Refresh interval = 1 s** when you write 10 K docs/sec — segment thrash. 5–30 s for write-heavy.
- **No ILM** — hot tier accumulates 90 days, costs explode, queries get slower.
- **Aggregation on free-text `message`** — there's no doc-values for arbitrary text; you'll either re-tokenize at query time or be told it's not supported.
- **Returning exact `terms` agg by raising shard_size to billion** — fights the architecture; embrace approximate or denormalize.
- **Ignoring partial results** — your dashboard says "100 % of errors" because the slowest shard fell out of the result. Surface partiality.

---

## 14. Putting It Together — A Query Walkthrough

The user types:
```
service:auth-service env:prod status:>=500 region:us-east-1
   AND error.type:TimeoutException
   GROUP BY service, host, region
   FOR THE LAST 15 MINUTES
```

1. **Coordinator** receives query. Parses, validates, plans. Determines time window → 15 minutes.
2. **Shard pruning**: of 10⁵ shards, only those whose time-range overlaps last 15 minutes survive. ~50 shards.
3. **Coordinator scatters** the query to those 50 shards' replicas (least-loaded).
4. **Per shard**:
   a. Look up bitmaps: `service:auth-service`, `env:prod`, `status:[500..599]` (range → OR of 100 term bitmaps), `region:us-east-1`, `error.type:TimeoutException`.
   b. AND them. Result: a Roaring bitmap of matching doc IDs in this shard. Possibly cached from recent identical filter.
   c. For each facet dimension (`service`, `host`, `region`): iterate the bitmap, for each set bit read the doc-value ordinal, increment a counter. Maintain top-1000 heap per dimension (`shard_size = 1000`).
   d. Compute total count (bitmap cardinality).
   e. Return partial: `{total, facets: {service: top-1000, host: top-1000, region: top-1000}}`.
5. **Coordinator merges**: sums totals, merges top-K maps (sum colliding keys, re-extract global top-10).
6. **Result cached** keyed by `(filter, time-window, facet-dims)` with TTL = remaining-bucket-life.
7. **Response** to user: 100–300 ms typical for a hot filter on a hot tier.

The user toggles `region:eu-west-1`. Coordinator goes back to step 2 — same shard list (same time window), one less filter clause, hits filter cache for the unchanged predicates, recomputes only the differential. Often < 100 ms.

---

## 15. What Makes This Staff-Level

1. **Naming the right primitives**: inverted index + Roaring bitmaps + doc values, not "Lucene magic."
2. **Knowing where each cost is paid**: CPU on bitmap AND, memory on global ordinals, I/O on doc-value scans, network on scatter-gather, RAM on filter cache.
3. **Approximation as a first-class design choice**, not a fallback — HLL, T-Digest, Misra-Gries, with explicit error bounds and merge semantics.
4. **Coordinator merge correctness**: knowing why `terms` agg is approximate even with `shard_size: 1000`, when to raise it, when to denormalize.
5. **Tiered storage and lifecycle** — recognizing 90 % of queries hit 5 % of data; build the cost model accordingly.
6. **Filter-cache hit rate** as the true production metric; rounding time predicates to enable reuse.
7. **High-cardinality discipline** — deciding *at ingest* what is faceted vs search-only vs cardinality-only.
8. **Engine choice as a fit problem** (Lucene vs Druid vs ClickHouse vs custom Husky) — driven by workload shape, not feature checklists.
9. **Anti-patterns named** — wildcards, second-resolution `now`, no ILM, refresh-interval misuse — the things that take production from "snappy" to "drowning."
10. **Partial-result honesty**: when the architecture is approximate, the UI must say so.

The deeper insight: faceted search at this scale isn't one algorithm — it's a *layered amortization*. Each cost dimension (CPU, RAM, I/O, network, latency) has its own structure designed to push that cost off the critical path. Composed well, the user gets sub-second filtering across petabytes; composed poorly, any one layer becomes the bottleneck.