# 3 · Segments and Columnar Storage

Segments are to Druid what parts are to ClickHouse. Understanding the **segment format** is the precondition for understanding Druid query speed, indexing choices, and ingestion behavior.

## 3.1 What a segment is

A **segment** is an **immutable, self-contained, columnar file** that contains a time interval's worth of rows for a single datasource at a single version.

Properties:
- **Time interval** — e.g., `2026-05-17T00:00:00Z/2026-05-18T00:00:00Z`.
- **Version** — a string (usually a timestamp) that uniquely identifies this iteration.
- **Shard spec** — within a time interval, segments can be further partitioned (hashed, ranged, single, linear).
- **Size on disk** — typically 300-700 MB (sweet spot).
- **Self-describing** — contains its own column metadata, dictionaries, bitmaps.

The identifier: `datasource_interval_version_shardSpec`, e.g., `events_2026-05-17/2026-05-18_2026-05-17T00:01:00.123Z_0`.

## 3.2 Segment file layout (the binary format)

A segment is a directory (in deep storage and on Historical local disk) containing:

```
<segment-dir>/
  version.bin                  -- segment format version
  meta.smoosh                  -- catalog of where each column starts in data.smoosh
  00000.smoosh                 -- column data, packed in a single file (or 00001, 00002 if > 2GB)
```

The "smoosh" file format is a custom packed container — many logical files in one physical file, sized to fit in memory-mapped reads.

Inside the smoosh: one entry per **column**, plus metadata sections like:
- `__time` column (always present — the primary time column).
- Each dimension column.
- Each metric column.
- Bitmap indexes for dimension columns.
- Dictionary for each string dimension.
- Special columns (spatial, JSON nested columns).

The Historical memory-maps the smoosh file and serves queries from the OS page cache.

## 3.3 Column types

### `__time` column

Always present, always first. Stored as long (millis since epoch). Druid uses it for time-filter pruning.

### Dimension columns

Three flavors:

1. **String dimensions** — the classic. Dictionary-encoded + bitmap-indexed.
2. **Long/Double/Float numeric dimensions** — stored as compressed numerics; no bitmap index by default (optional numeric index in newer versions).
3. **Multi-value string dimensions** — each row can have an array of values for the dimension; each value gets its own bitmap bit.

### Metric columns

Numeric aggregates: `Long`, `Double`, `Float`. Plus complex types:
- **HLL sketch** (HyperLogLog for distinct count).
- **Theta sketch** (set algebra over distinct values).
- **Quantile sketch** (estimating percentiles).
- **Tuple sketch** (multi-dimensional).
- **Variance / stddev / first / last**.

Metrics are pre-computed at ingest time (rollup) — they're not raw values that get aggregated at query time (well, they are, but the aggregation function is already locked in).

### JSON / nested columns (Druid 26+)

A `COMPLEX<json>` column stores arbitrary JSON. Druid auto-discovers paths within it and creates **per-path subcolumns** with their own dictionaries and indexes. Queries can filter / project on `JSON_VALUE(col, '$.path.to.field')`.

Comparable to ClickHouse's new JSON type — same idea.

## 3.4 String dimensions in detail (the heart of Druid)

For a column like `country` with values `['US', 'CA', 'DE', 'US', 'FR', ...]`:

### Step 1: dictionary

```
dictionary[0] = 'CA'
dictionary[1] = 'DE'
dictionary[2] = 'FR'
dictionary[3] = 'US'
```

Sorted ascending. Each row's value is replaced by its dictionary index:

```
encoded values per row: [3, 0, 1, 3, 2, ...]
```

### Step 2: front-coded dictionary (Druid 26+)

For string dictionaries with long common prefixes (like URLs), front coding stores the common prefix once:

```
Index 0: "https://example.com/a"  (full string)
Index 1: 21, "b"                   (share 21 bytes with [0], then "b")
Index 2: 21, "c"                   (share 21 bytes with [0], then "c")
...
```

Typical savings: **30-50%** smaller dictionaries for URL/path-like data. Faster reads too (less I/O).

### Step 3: bitmap index per value

For each distinct value, a bitmap (Roaring-compressed) marks which rows contain it:

```
bitmap['US'] = 1100100... (rows 0, 1, 4, ...)
bitmap['CA'] = 0010000...
```

This is the key data structure for fast `WHERE` evaluation: `WHERE country = 'US'` is "scan one bitmap and find set bits"; `WHERE country IN ('US','CA')` is bitmap OR of two bitmaps. Roaring is fast at both AND, OR, andnot, cardinality.

### Step 4: the data array

The encoded integer values per row, stored compressed (LZ4 by default).

## 3.5 Roaring bitmap compression — why it matters

A **Roaring bitmap** chops the 32-bit row-id space into 16-bit "containers" of 65536 rows each. Each container is one of:
- **Bitset** — when the container is dense (many bits set), stored as a 65536-bit (8 KB) bitmap.
- **Array** — when the container is sparse (few bits set), stored as a sorted list of 16-bit integers.
- **Run** — when bits come in long runs, stored as a list of (start, length) pairs.

The container type per chunk is chosen automatically based on what compresses best. Result: very compact for both dense and sparse bitmaps, plus fast intersect/union/andnot via specialized algorithms per container type.

This is what makes `WHERE dim1 = X AND dim2 = Y` fast: bitmap AND of two bitmaps, no row scan.

## 3.6 Numeric columns

Compressed via LZ4 or LZF. No bitmap index by default — Druid scans the column and applies the predicate.

**Numeric range indexes** (newer feature) provide per-value-range bitmaps for filtering numeric columns: `WHERE price BETWEEN 10 AND 50` can use bitmap operations instead of scanning all values.

**Auto-typed value indexes** for JSON nested columns work similarly.

## 3.7 The cost of indexing every dimension

Druid indexes **every** string dimension by default — no opt-in needed (unlike ClickHouse where skip indexes are explicit).

The cost:
- Storage: bitmap data adds ~15-30% to segment size for typical schemas.
- Ingest: bitmap construction is real CPU cost.
- Update: never (segments are immutable).

The benefit:
- Every filter on a string dimension has bitmap fast-path.
- This is what gives Druid its consistent sub-second query latency.

If you have very-high-cardinality dimensions (millions of distinct values), the bitmap index becomes huge and the benefit per query degrades. Workarounds:
- Don't index that dimension (skip with `createBitmapIndex: false`).
- Numeric-encode it if it's actually numeric.
- Drop it from the schema if not needed for filtering.

## 3.8 Segment sizing — the goldilocks zone

- **Too small** (< 100 MB) — Broker has to fan out to too many segments per query. Coordinator metadata DB row-count grows.
- **Too big** (> 1 GB) — slow segment download to Historicals, memory pressure on scan, hard-to-balance load.
- **Sweet spot**: **300-700 MB** compressed (Druid's recommended range).

Target rows per segment: **5-15 million**.

If real-time ingest produces small segments, **auto-compaction** consolidates them. See [12](12-compaction-segment-management.md).

## 3.9 Segment versioning (overshadowing)

When a new ingestion creates a segment that covers the same time interval as an existing one, the new segment's *version* is higher (lexicographically). The Coordinator marks the older segment as **overshadowed** and replaces it.

Use cases:
- **Re-ingest** to fix data: ingest the same time interval with a new version → new segment supersedes.
- **Compaction**: the compaction task produces a new version that replaces the old segments.
- **MSQ REPLACE**: replaces all segments in the target interval with new versions.

Overshadowed segments are marked **unused** in the metadata DB. The Coordinator's kill task eventually deletes them from deep storage.

## 3.10 Schema evolution

Druid is more permissive than you'd think:

- **Add a dimension**: future segments have it; old segments are queried with NULL or default for that column. No rewrite needed.
- **Add a metric**: same — future segments have it; old segments return 0 or NULL.
- **Remove a dimension/metric**: future segments don't have it; old segments still do.
- **Change the type of a column**: requires re-ingest of affected segments.
- **Change rollup granularity** (e.g., minute → hour): requires re-ingest.

Most of the time you just keep ingesting and let the schema evolve organically. Older segments will lack newer columns but queries work fine.

## 3.11 Compact / Wide / Sparse segments

Segment configurations to know about:
- **forceSegmentSortByTime** — modern segments may sort by `__time` first; legacy can use other orderings.
- **sparse columns** — columns that are mostly null are stored efficiently.
- **JSON columns** with sparse auto-discovered paths.

## 3.12 Worked example — what a segment looks like for an events table

Datasource: `events`. One segment for 2026-05-17.

Schema:
```
__time        — Long, millis
user_id       — String (high cardinality — bitmap is large)
event_type    — String (low cardinality)
country       — String (low cardinality)
device        — String (low cardinality)
session_id    — String (very high cardinality, no bitmap)
revenue       — Double metric (sum aggregation)
clicks        — Long metric (count aggregation)
distinct_users — HLLSketch metric (HLL aggregation)
```

Segment contents:
- `__time`: 10M longs, compressed → ~30 MB.
- `event_type` dict (50 values) + bitmap (50 bitmaps × 10M bits, very sparse) → ~5 MB.
- `country` dict (200) + bitmaps → ~20 MB.
- `device` dict (10) + bitmaps → ~3 MB.
- `user_id` dict (1M) + bitmaps (1M bitmaps × sparse) → ~80 MB (the biggest, by far).
- `session_id`: no bitmap (createBitmapIndex: false) → ~50 MB (just dict + encoded ints).
- `revenue` metric: 10M doubles compressed → ~40 MB.
- `clicks` metric: 10M longs compressed → ~25 MB.
- `distinct_users` HLL sketch: stored as binary per row → ~20 MB.

Total: ~270 MB. Right in the sweet spot.

Query `SELECT country, sum(revenue) FROM events WHERE event_type = 'purchase' GROUP BY country`:
1. Bitmap[event_type='purchase'] gives row mask.
2. Scan `country` and `revenue` columns, filtered by mask.
3. Group by country, sum revenue. Done in tens of milliseconds.

## 3.13 Tools for inspecting segments

- **dump-segment CLI** — print human-readable segment metadata.
- **DruidSegmentReader** (Java API) — read a segment outside Druid.
- **Coordinator web console** — see segments per datasource, their sizes, replication state.

## 3.14 Must-internalize

- A segment is **immutable**, **columnar**, **time-bounded**, with a **dictionary + Roaring bitmap per string dimension**.
- Sweet-spot segment size: **300-700 MB**, **5-15M rows**.
- String dimension storage = dictionary + encoded ints + per-value Roaring bitmap. Often **front-coded** for prefix-similar strings.
- Numeric columns are LZ4-compressed; bitmaps optional via numeric range index.
- Metric columns hold pre-aggregated values, optionally complex types (DataSketches).
- JSON `COMPLEX<json>` columns auto-discover paths into subcolumns.
- Overshadowing: a higher-version segment replaces a lower-version one in the same interval.
- Schema evolution: add/remove dim/metric is fine. Type changes require re-ingest.

---

## Sources

- [Segments — official](https://druid.apache.org/docs/latest/design/segments/)
- [Front-coded dictionaries blog (Imply)](https://imply.io/blog/introducing-incremental-encoding-for-apache-druid-dictionary-encoded-columns/)
- [Indexes in Druid — Hellmar Becker](https://blog.hellmar-becker.de/2023/06/28/indexes-in-apache-druid/)
- [Roaring bitmaps](https://roaringbitmap.org/)
- [Columnar storage in Druid — Medium](https://medium.com/@darshan.piyush302/how-columnar-storage-makes-apache-druid-blazing-fast-a-deep-dive-f52fc9e48981)
- [dump-segment tool](https://druid.apache.org/docs/latest/operations/dump-segment/)
