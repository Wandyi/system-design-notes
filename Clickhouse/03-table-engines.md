# 3 · Table Engines — The Full Family

A ClickHouse "engine" defines the storage layout, the indexing, the replication behavior, and any merge-time logic. 
Picking the right one is the most common interview probe — every system-design question is partly "what engine and why?".

## 3.1 The MergeTree family

### MergeTree (the default)

Append-only, sorted, merged. Nothing happens at merge time except concatenation.

```sql
CREATE TABLE events (
    ts        DateTime,
    user_id   UInt64,
    bytes     UInt32
) ENGINE = MergeTree
ORDER BY (ts, user_id)
PARTITION BY toYYYYMM(ts);
```

**When to use**: most analytics workloads. Default choice.

### ReplacingMergeTree — deduplicate on merge

Keeps the row with the *highest* version (or the last inserted if no version) for each unique `ORDER BY` key, 
deleting older versions *when those rows happen to merge*.

```sql
CREATE TABLE users (
    user_id    UInt64,
    name       String,
    email      String,
    updated_at DateTime
) ENGINE = ReplacingMergeTree(updated_at)
ORDER BY user_id;
```

**Caveats**:
- Dedup happens **only when parts merge**. Until a merge happens, you'll see duplicates in `SELECT`.
- Use `FINAL` to force at-query-time dedup (slow on big data) or `argMax(...)` patterns (see [14-query-patterns-and-corner-cases.md](14-query-patterns-and-corner-cases.md)).
- Dedup is **per part** until the parts merge — so cross-part duplicates persist until they happen to land in the same merge.
- Pairs well with a `is_deleted UInt8` column + the new `clean_deleted_rows` setting to physically purge on merge.

**When to use**: CDC-style "latest state" tables, slowly-changing dimensions, mutable entities.

### CollapsingMergeTree — net positive/negative rows

Each row has a `Sign Int8` (+1 or -1). At merge time, rows with the same `ORDER BY` key cancel: one +1 and one -1 disappear.

```sql
CREATE TABLE user_state (
    user_id    UInt64,
    status     String,
    Sign       Int8
) ENGINE = CollapsingMergeTree(Sign)
ORDER BY user_id;
```

Use when you have a stream of state changes and want to subtract old state and add new state.

```sql
-- updating a user's status
INSERT INTO user_state VALUES (1, 'active', -1);  -- cancel old
INSERT INTO user_state VALUES (1, 'inactive', 1); -- add new
```

**Caveat**: order of inserts matters; if -1 lands without a matching +1, you get a negative-state row. Operationally fragile. 
Use `VersionedCollapsingMergeTree` if you can attach versions.

### VersionedCollapsingMergeTree

Same idea as Collapsing but with an explicit version column to resolve out-of-order arrivals correctly.

```sql
ENGINE = VersionedCollapsingMergeTree(Sign, Version)
```

### SummingMergeTree — sum-up duplicates on merge

For each `ORDER BY` key, sum the configured numeric columns; one row remains.

```sql
CREATE TABLE pageviews_by_hour (
    hour      DateTime,
    page      String,
    views     UInt64,
    bytes     UInt64
) ENGINE = SummingMergeTree
ORDER BY (hour, page);
```

After merge, multiple inserts with the same `(hour, page)` become one row with summed `views` and `bytes`.

**When to use**: pre-aggregated counts on a known granularity. Cheaper than an `AggregatingMergeTree` if you only need `SUM`.

### AggregatingMergeTree — the workhorse for materialized views

Stores **intermediate aggregation states** (binary blobs produced by `*State` functions). 
At merge time, states for the same key are **combined** via the aggregate's combine function.

```sql
CREATE TABLE rev_by_day (
    day            Date,
    country        LowCardinality(String),
    revenue_state  AggregateFunction(sum, Decimal64(2)),
    users_state    AggregateFunction(uniq, UInt64)
) ENGINE = AggregatingMergeTree
ORDER BY (day, country);
```

At write time you call `*State` (e.g., `sumState`, `uniqState`); at read time you call `*Merge`.

```sql
SELECT day, country,
       sumMerge(revenue_state)  AS revenue,
       uniqMerge(users_state)   AS users
FROM rev_by_day
GROUP BY day, country;
```

**When to use**: pre-aggregating from raw events via a materialized view. The canonical real-time-analytics acceleration pattern. 
See [06](06-indexes-projections-and-mvs.md).

### GraphiteMergeTree

Specialized for Graphite time-series. Rolls up old data into coarser intervals automatically. Niche but useful for that exact use case.

## 3.2 Replicated variants

Every MergeTree variant has a `Replicated*` twin (e.g., `ReplicatedMergeTree`, `ReplicatedAggregatingMergeTree`). 
They use ClickHouse Keeper for coordination — see [07](07-replication-and-keeper.md).

Required for any production cluster.

## 3.3 SharedMergeTree (Cloud-only)

Cloud's leaderless variant for shared S3 storage. Same surface as MergeTree but the data layer is S3 and metadata is Keeper. 
Replicas share the data; scaling is free of replication. See [09](09-cloud-and-sharedmergetree.md).

In ClickHouse Cloud, when you write `MergeTree` it's silently mapped to `SharedMergeTree`.

## 3.4 The Log family

For tiny tables or scratch use. Not for production analytics.

- **TinyLog** — one file per column. No marks. No PK. Can't be queried in parallel. Useful for testing.
- **Log** — one file per column + a common `__marks.mrk`. Slightly better than TinyLog.
- **StripeLog** — single data file. Most space-efficient of the trio.

All Log-family tables lack: replication, mutations, parallel reads, primary indexes. Use them rarely.

## 3.5 Memory engines

- **Memory** — entire table in RAM. Lost on restart. Useful for staging / temporary results.
- **Set** — RAM-resident set, written-once, used in `IN`. Useful for fast membership testing.
- **Join** — RAM-resident pre-built hash table, used by direct joins.
- **EmbeddedRocksDB** — KV in RocksDB, queryable as a ClickHouse table. Also a direct-join target.

## 3.6 Buffer engine

A RAM staging table that flushes to a backing table on size/age thresholds. 
Used to absorb small-insert workloads before async-insert existed. Today mostly replaced by `async_insert = 1`.

```sql
CREATE TABLE events_buffer AS events
ENGINE = Buffer(default, events, 16, 5, 60, 10000, 100000, 1000000, 10000000);
```

## 3.7 Integration engines (foreign data)

ClickHouse can query external systems directly via "table engines that aren't tables here":

- **Kafka** — consumes a Kafka topic; combined with a materialized view to a MergeTree, you get streaming ingest with exactly-once consumer offsets.
- **RabbitMQ**, **NATS**, **Redis**, **MongoDB**, **MySQL**, **PostgreSQL**, **MSSQL** — single-table foreign tables.
- **S3 / GCS / Azure Blob** — table functions/engines for object storage; reads Parquet, CSV, JSON, ORC, Avro, ProtoBuf.
- **HDFS / WebHDFS**.
- **URL** — fetch from an HTTP endpoint, parse with one of many formats.
- **File** — read a local file.
- **JDBC / ODBC** — generic external sources.
- **MaterializedPostgreSQL** / **MaterializedMySQL** — CDC ingest via the source DB's replication protocol; mirrors a logical database into ClickHouse.
- **Iceberg / Delta / Hudi** — read lakehouse tables. Increasingly write-too.

### Kafka pattern (canonical)

```sql
CREATE TABLE kafka_events (
    ts UInt64, user_id UInt64, event String
) ENGINE = Kafka(
    'broker:9092',
    'events',
    'ch-consumer-group',
    'JSONEachRow'
);

CREATE TABLE events (...) ENGINE = ReplicatedMergeTree(...);

CREATE MATERIALIZED VIEW kafka_to_events TO events AS
SELECT ts, user_id, event FROM kafka_events;
```

The materialized view runs the Kafka consumer; rows from `kafka_events` are inserted into `events`.

## 3.8 Distributed engine

A virtual / proxy engine. Holds no data. Routes queries to per-shard *local* tables, aggregates results.

```sql
CREATE TABLE events_local (...) ENGINE = ReplicatedMergeTree(...);

CREATE TABLE events_dist ON CLUSTER my_cluster
AS events_local
ENGINE = Distributed(my_cluster, default, events_local, rand());
```

See [08](08-sharding-and-distributed.md).

## 3.9 Dictionary engine

A `Dictionary` is a key-value lookup loaded from a source (table, MySQL, PostgreSQL, file, HTTP). Use cases:
- Lookup by ID in a query: `dictGet('users', 'name', user_id)`.
- Direct join target: ClickHouse can do `LEFT JOIN dict` very fast via the direct-join algorithm.

```sql
CREATE DICTIONARY users_dict (
    user_id  UInt64,
    name     String,
    country  String
)
PRIMARY KEY user_id
SOURCE(CLICKHOUSE(host 'localhost' table 'users'))
LAYOUT(HASHED())
LIFETIME(MIN 300 MAX 600);
```

**Layouts** matter:
- `FLAT` — array indexed by integer key. Fastest if keys are dense.
- `HASHED` — open-addressed hash. General-purpose.
- `COMPLEX_KEY_HASHED` — multi-column key.
- `CACHE` — bounded LRU; fetches misses from source.
- `SSD_CACHE` — like CACHE but spills to SSD.
- `RANGE_HASHED` — range-keyed (e.g., date ranges).
- `IP_TRIE` — IP prefix lookup.
- `POLYGON` — geo lookup by polygon.

A dictionary is the right answer for **small-to-medium reference data** that you'd otherwise repeatedly join.

## 3.10 MaterializedView engine

The MV itself is a table (with an engine — `TO target` form is preferred). The MV-as-a-pseudo-engine is mostly an implementation detail; 
you almost always use the `TO target_table` form so the MV writes to a normal MergeTree.

## 3.11 RefreshableMaterializedView

New. The MV refreshes on a schedule (every N seconds/minutes/hours), running its full SELECT and replacing (or appending to, with `APPEND` mode) the target.

Use for aggregations that can't be incrementalized (`MEDIAN`, `RANK`, anything window-based, complex joins).

```sql
CREATE MATERIALIZED VIEW daily_top_users
REFRESH EVERY 1 HOUR
TO target_table AS
SELECT day, user_id, count() AS events
FROM events
GROUP BY day, user_id
ORDER BY events DESC
LIMIT 100 BY day;
```

## 3.12 View (non-materialized)

A standard SQL VIEW. No storage; just a stored query. Useful for ergonomic abstraction. Performance is the underlying SELECT's performance.

## 3.13 Engine selection rubric

A staff-level answer to "which engine?":

| Scenario | Engine |
|----------|--------|
| Append-only events / logs / metrics | **MergeTree** (or **ReplicatedMergeTree** in production) |
| Tens of thousands of writes per second per node | MergeTree with batched or async inserts |
| "Latest state" per entity (CDC, slowly changing dim) | **ReplacingMergeTree** + `argMax`/FINAL at query time |
| Pre-aggregated counts / sums | **SummingMergeTree** or **AggregatingMergeTree** |
| Pre-aggregated complex stats (uniq, quantile, etc.) | **AggregatingMergeTree** |
| Pre-aggregated rollups maintained from raw events | **AggregatingMergeTree** + MATERIALIZED VIEW |
| Net inserts/deletes of state | **CollapsingMergeTree** (or **VersionedCollapsingMergeTree**) |
| Sub-day-old data only | MergeTree with TTL DELETE; or Buffer; or Memory for testing |
| Reading from Kafka | **Kafka** engine + MATERIALIZED VIEW → MergeTree |
| Reading from another DB once-off | Foreign table (MySQL/Postgres/etc.) |
| Small lookup table joined repeatedly | **Dictionary** + `dictGet` or direct join |
| Multi-shard table | **Distributed** in front of ReplicatedMergeTree per shard |
| Cloud production table | **SharedMergeTree** (the Cloud default) |

## 3.14 Must-internalize

- MergeTree is default; everything else is a specialization.
- ReplacingMergeTree / SummingMergeTree / AggregatingMergeTree / CollapsingMergeTree handle four different mutation flavors at merge time.
- Replicated* is mandatory for production. SharedMergeTree is the Cloud-only leaderless variant.
- Kafka engine + MV is the canonical streaming ingest.
- Dictionary + direct join replaces almost every small JOIN you'd otherwise write.
- Distributed is a router with no storage; the actual data lives in per-shard local tables.

---

## Sources

- [MergeTree family — official](https://clickhouse.com/docs/engines/table-engines/mergetree-family/)
- [ReplacingMergeTree](https://clickhouse.com/docs/engines/table-engines/mergetree-family/replacingmergetree)
- [AggregatingMergeTree](https://clickhouse.com/docs/engines/table-engines/mergetree-family/aggregatingmergetree)
- [SummingMergeTree](https://clickhouse.com/docs/engines/table-engines/mergetree-family/summingmergetree)
- [CollapsingMergeTree](https://clickhouse.com/docs/engines/table-engines/mergetree-family/collapsingmergetree)
- [Kafka table engine](https://clickhouse.com/docs/engines/table-engines/integrations/kafka)
- [Dictionaries](https://clickhouse.com/docs/sql-reference/dictionaries)
- [Distributed engine](https://clickhouse.com/docs/engines/table-engines/special/distributed)
- [SharedMergeTree (Cloud)](https://clickhouse.com/docs/cloud/reference/shared-merge-tree)
- [AggregatingMergeTree explained — glassflow](https://www.glassflow.dev/blog/aggregatingmergetree-clickhouse)
