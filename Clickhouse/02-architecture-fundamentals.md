# 2 · Architecture Fundamentals

This is the single most important file. Almost every other answer in a ClickHouse interview compresses to "...because MergeTree works like this." Read it twice.

## 2.1 The mental model

```
              ┌────────────────────────────────────────────────┐
INSERT  ───►  │   Memory accumulator (BlocksToInsert / WAL?)   │
              └─────────────────────┬──────────────────────────┘
                                    │  flush as a "part" on disk
                                    ▼
              ┌───────────────────────────────────────────────────┐
              │   Data parts directory                            │
              │   /var/lib/clickhouse/data/<db>/<table>/<part>/   │
              │     ├ <col>.bin       (compressed column data)    │
              │     ├ <col>.mrk2      (marks: offsets per granule)│
              │     ├ primary.idx     (sparse PK in RAM)          │
              │     ├ partition.dat   (which partition this part) │
              │     ├ minmax_<col>.idx (per-column min/max)       │
              │     └ checksums.txt                               │
              └───────────────────────────────────────────────────┘
                                    ▲
                                    │  background merger picks 2-8 parts,
                                    │  merges them into a bigger part
                                    │  (LSM-style)

SELECT  ◄─── reads marks ◄─── reads only the column files it needs
        └── PREWHERE prunes granules using primary index + skip indexes
```

Three big ideas:

1. **Columnar layout** — each column lives in its own file (or its own stream within a compact part).
2. **Sorted, immutable parts** — every insert produces a new part; parts are merged in the background.
3. **Sparse primary index** — one entry per 8192-row granule, fits in RAM, prunes scans.

## 2.2 Columnar vs row layout

Same logical table, two physical layouts:

```
Row store (Postgres-class):
[ id=1  ts=...  user=alice  bytes=42 ]
[ id=2  ts=...  user=bob    bytes=10 ]
...

Column store (ClickHouse):
ids:    [1, 2, 3, ...]
ts:     [..., ..., ...]
users:  ["alice", "bob", ...]
bytes:  [42, 10, ...]
```

Why columnar wins for analytics:
- **Reads only what you need.** `SELECT sum(bytes) FROM t WHERE user='alice'` reads two columns, not the whole row.
- **Compression**: adjacent column values are similar → 10-100× compression. Delta-of-delta on timestamps; LZ4/ZSTD on text; T64 on integers.
- **Vectorized execution**: each column arrives as a buffer; we apply SIMD-friendly tight loops.

Why columnar loses for OLTP:
- Single-row reads need to seek N files (one per column).
- Single-row writes mean append into N files; the merger may be busy.
- No multi-row transactions.

ClickHouse picks the analytical side hard.

## 2.3 The MergeTree storage layout

A `MergeTree` table on disk:

```
/var/lib/clickhouse/data/default/events/
   202605_1_1_0/        # part: partition=202605, minBlock=1, maxBlock=1, level=0
     ts.bin             # column file (compressed blocks)
     ts.cmrk2           # marks (compact, modern)
     user_id.bin
     user_id.cmrk2
     primary.idx        # sparse index, one entry per granule
     count.txt          # row count
     columns.txt        # column metadata
     checksums.txt
   202605_2_2_0/        # next insert → next part
   202605_1_2_1/        # merged: covers blocks 1..2, level=1
```

Naming convention `<partition>_<minBlock>_<maxBlock>_<level>` — the merger increments `level`, expanding the `[minBlock, maxBlock]` range.

Two part formats:
- **Wide** (default for large parts) — one file per column.
- **Compact** (small parts, configurable threshold) — all column data interleaved in one file. Saves filesystem inodes.

## 2.4 Granules and marks

Inside each part, the column data is divided into **granules** — fixed-size groups of rows (default `index_granularity = 8192`).

For each granule, ClickHouse keeps:
- **One mark** in the `.mrk2` file: byte offset into the column's `.bin` file + the row offset within decompressed data.
- **One primary index entry** (in the part's `primary.idx`): the value of the sorting key at the *first* row of that granule.

That's why the primary index is "sparse": you get the PK every 8192 rows, not every row. A 10-billion-row table has ~1.2M index entries; small enough to fit fully in RAM.

To read rows `[100,000 .. 102,000]`:
1. Binary-search the primary index → find the granule (e.g., granule #12).
2. Look up mark[12] → byte offset into the column file.
3. Decompress that block, find the requested rows.

This is why your **ORDER BY = your primary index = your filter superpower**.

## 2.5 The primary key vs. sorting key — same thing by default

```sql
CREATE TABLE events
(
    ts        DateTime,
    user_id   UInt64,
    event     LowCardinality(String),
    bytes     UInt32
)
ENGINE = MergeTree
ORDER BY (ts, user_id);
```

Here `ORDER BY (ts, user_id)` does three things at once:

1. Defines the physical sort order within each part.
2. Defines the primary key (= the sparse-index columns).
3. Defines the granule boundaries.

You can decouple them:

```sql
ORDER BY (ts, user_id, event)   -- physical sort + uniqueness for dedup engines
PRIMARY KEY (ts, user_id)       -- just the PK (subset of ORDER BY, must be a prefix)
```

Common reason to split: dedup engines (Replacing/Aggregating) need a longer `ORDER BY` than you want in the index.

## 2.6 Partitioning

Optional. `PARTITION BY` carves the table into independent physical groups:

```sql
PARTITION BY toYYYYMM(ts)
```

Each partition's parts merge only with other parts in the same partition. Use cases:
- **Data lifecycle** — `ALTER TABLE DROP PARTITION '202401'` instantly deletes a month.
- **TTL** boundaries — TTL DELETE / MOVE happens at part granularity, which is partition-bounded.
- **Backup granularity** — `ALTER TABLE FREEZE PARTITION` snapshots one partition.

**Anti-pattern**: too many partitions. Each is independent metadata. Daily partitions for 5 years = 1800 partitions; doable. Hourly partitions for 5 years = 43,800; you'll feel it. 
Rule of thumb: keep **< 1000 active partitions** per table.

`PARTITION BY` is **not** for query pruning by itself — that's what the primary index is for. Partitioning is for lifecycle and management.

## 2.7 Background merges

The "Merge" in MergeTree. A background thread pool:
1. Picks 2–N parts that are eligible to merge (same partition, similar sizes).
2. Merges them via a streaming sorted-merge (the parts are already sorted).
3. Writes a new part with the union of rows.
4. Marks the old parts inactive (their files are deleted later by a cleanup task).

The merger has policies (`max_bytes_to_merge_at_*`, `parts_to_throw_insert`, `parts_to_delay_insert`) that throttle:
- If a table has > `parts_to_delay_insert` (default ~150) inactive parts, the engine throttles inserts.
- If > `parts_to_throw_insert` (default 300) parts pile up, inserts are rejected outright: "Too many parts."

**This is the #1 operational mistake**: many small inserts → merger can't keep up → "Too many parts" → outage. The cure: batch inserts. Buffer 100K-1M rows client-side and insert in batches every few seconds.

## 2.8 Async insert — the small-insert escape hatch

For workloads that can't batch client-side (e.g., per-request event from many app servers), ClickHouse has **async inserts**:

```sql
SET async_insert = 1, wait_for_async_insert = 0;
INSERT INTO events VALUES (...);
```

The server buffers many small inserts and creates one part. Latency: `async_insert_busy_timeout_ms` (default 200ms) or `async_insert_max_data_size` (default 100MB).

Alternative: a **Buffer engine** table in front of MergeTree (older approach; superseded by async inserts for most cases).

## 2.9 Vectorized execution

A query in ClickHouse isn't tuple-at-a-time. It's **block-at-a-time** (default 65,536 rows per block). Each operator (filter, project, aggregate) processes a whole block in tight loops the compiler auto-vectorizes (SIMD). 
This is why a "scan everything and sum" query at 1 billion rows/sec/core is realistic.

Implications:
- Avoid per-row functions written naively in `arrayJoin` blowing the block model.
- Wide blocks favor cache locality; very narrow tables get less benefit.
- Aggregates use per-block hash tables → merged across threads.

## 2.10 Compression

Two layers:

1. **Per-block codec** on each column (LZ4 default, ZSTD common, Delta/DoubleDelta/Gorilla/T64 for numeric/time-series).
2. **Generic compression** as a wrapper (LZ4 / ZSTD).

Typical ratios on real data:
- Timestamps with `Delta(8), ZSTD(1)`: 30-50× compression.
- LowCardinality(String) for enum-like text: 10-20× over raw.
- Highly entropic UUIDs: 1.1–1.3× (already near-random).

See [05-codecs-and-compression.md](05-codecs-and-compression.md) for the menu.

## 2.11 The query path (what happens on SELECT)

1. **Parse** SQL → AST.
2. **Analyze**: resolve names, types, expression rewrites.
3. **Optimize**: query-tree level optimizations (predicate pushdown, PREWHERE, projection selection, constant folding).
4. **Plan**: physical operators (TableScan, Filter, Aggregate, Join, Sort).
5. **Pipeline / processor execution**: chain of `IProcessor` running in parallel. Each scans parts in parallel; per-thread partial states; final merge.
6. **Return** rows to the client.

Use `EXPLAIN SYNTAX`, `EXPLAIN`, `EXPLAIN PIPELINE`, `EXPLAIN ESTIMATE`, `EXPLAIN AST` to see each layer. Use `system.query_log` post-hoc.

## 2.12 PREWHERE — the column-store filter trick

`WHERE` reads all needed columns first, then filters. `PREWHERE` reads only the filter columns first, then reads other columns only for surviving rows.

ClickHouse automatically promotes some `WHERE` predicates to `PREWHERE`. For wide tables with cheap filters, this is a huge win.

```sql
-- explicit PREWHERE
SELECT user_id, large_text_blob
FROM events
PREWHERE event_type = 'login'
WHERE ts > now() - INTERVAL 1 DAY;
```

- ClickHouse reads `event_type`, filters → small subset of granules.
- Then reads `user_id` and `large_text_blob` only for those granules.

The automatic promotion: simple predicates on cheap-to-read columns. Override when ClickHouse picks badly (e.g., it chose a high-cardinality column when a `LowCardinality` would prune more).

## 2.13 Data skipping indexes — beyond the primary index

If your filter isn't on the sort key prefix, the primary index can't help. Skip indexes (also called secondary indexes) fill the gap:

- `minmax` — min/max per N granules. For numeric/date filters far from the sort order.
- `set(N)` — set of up to N distinct values per N granules.
- `bloom_filter` — probabilistic membership.
- `tokenbf_v1`, `ngrambf_v1` — bloom over string tokens / n-grams (text search-ish; being deprecated for the new `text` index in 26.2+).

Skip indexes work at the **granule** level — they can prove a granule has no matching rows and skip it. They never produce false negatives that would lose data; bloom-filter-class ones produce false positives that don't lose data, just read more.

See [06-indexes-projections-and-mvs.md](06-indexes-projections-and-mvs.md).

## 2.14 Projections — another physical layout

A **projection** is an extra physical copy of the data in a different sort order / with aggregation. The query optimizer picks the projection automatically if it accelerates the query.

```sql
ALTER TABLE events ADD PROJECTION p_by_user (
    SELECT * ORDER BY (user_id, ts)
);
ALTER TABLE events MATERIALIZE PROJECTION p_by_user;
```

Now `WHERE user_id = X` can use the projection's sort order instead of scanning by `(ts, user_id)`. Cost: storage doubles for that projection.

## 2.15 Materialized views — pre-aggregation on insert

ClickHouse MVs are **insert-time triggers**, not refresh-time recomputations (the old PostgreSQL meaning). When you insert into the source table, the MV's SELECT runs against the *new block* and the result is written to the target table.

Combined with `AggregatingMergeTree`, this gives you incrementally maintained roll-ups for almost free.

Newer addition: **Refreshable Materialized Views** which *are* refresh-time, scheduled. Useful when the aggregation can't be incrementalized.

See [06-indexes-projections-and-mvs.md](06-indexes-projections-and-mvs.md).

## 2.16 Replication — at a glance

`ReplicatedMergeTree` adds replication. Replicas talk to ClickHouse Keeper (Raft) for metadata: which parts exist, which merges have happened, which mutations are in flight. They fetch parts from each other over HTTP.

`SharedMergeTree` (Cloud) shares the parts on object storage; replicas just read Keeper metadata to learn what's available.

See [07-replication-and-keeper.md](07-replication-and-keeper.md).

## 2.17 Sharding — at a glance

ClickHouse shards via the **Distributed** engine — a virtual table that routes queries to per-node local tables. INSERTs are either client-side hashed or use a sharding key. The shard count is fixed; rebalancing requires re-ingest.

See [08-sharding-and-distributed.md](08-sharding-and-distributed.md).

## 2.18 Three numbers to anchor on

- **8192**: default `index_granularity`. One PK entry per 8192 rows.
- **65536**: default `max_block_size`. Vectorized execution block size.
- **150 / 300**: `parts_to_delay_insert` / `parts_to_throw_insert`. The "too many parts" thresholds.

Memorize these. They come up in coding rounds.

## 2.19 Must-internalize

- **MergeTree = columnar + sorted-immutable parts + sparse PK + background merge.**
- Sparse PK = one entry per 8192-row granule = primary.idx fits in RAM.
- ORDER BY is your sort order *and* your primary index. Pick it for your dominant query filter.
- Partition for **lifecycle**, not pruning. Keep partition count modest.
- Don't trickle small inserts — batch or use async insert.
- PREWHERE = column-store filter trick; automatic in many cases.
- Skip indexes & projections fill gaps the primary index can't.
- MVs are insert-time triggers; pair with AggregatingMergeTree for free roll-ups.
- Vectorized execution + columnar = the whole performance story.
- Replication via Keeper; sharding via Distributed; Cloud via SharedMergeTree.

---

## Sources

- [Sparse primary indexes — official guide](https://clickhouse.com/docs/guides/best-practices/sparse-primary-indexes)
- [MergeTree engine docs](https://clickhouse.com/docs/engines/table-engines/mergetree-family/mergetree)
- [Data parts and granules (oneuptime)](https://oneuptime.com/blog/post/2026-03-31-clickhouse-what-are-data-parts-and-granules/view)
- [I spent 8 hours learning the ClickHouse MergeTree — VuTrinh](https://vutr.substack.com/p/i-spent-8-hours-learning-the-clickhouse)
- [BigData Boutique — MergeTree engine](https://bigdataboutique.com/blog/clickhouse-mergetree-engine)
- [PostHog handbook — data storage / MergeTree](https://posthog.com/handbook/engineering/clickhouse/data-storage)
- [PREWHERE docs](https://clickhouse.com/docs/optimize/prewhere)
