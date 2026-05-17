# Apache Druid — Staff Software Engineer Interview Deep Dive

A complete preparation pack for a Staff Software Engineer interview at **Imply** (the Druid commercial company) or any Druid-heavy team. Builds on the [Clickhouse pack](../Clickhouse/README.md) — when an answer maps cleanly to a ClickHouse concept I tell you, and where the two engines diverge I make the contrast explicit.

Apache Druid is a real-time OLAP database, originally built at Metamarkets in 2011, donated to Apache, and now the engine behind a long list of user-facing analytics platforms — Netflix, Lyft, Confluent, Salesforce, Walmart, Cisco, Twitter (X), TripAdvisor, Airbnb, Pinterest. **Imply** is the founder-led commercial company (Pivot UI + Polaris managed service). At staff level the bar is the same shape as the ClickHouse bar: **deep on segment internals + correct engine/schema choice for any analytical workload + judgment on cost, multi-tenancy, and operations**.

The fundamental difference vs. ClickHouse: **Druid is built for ingest at streaming velocity with sub-second user-facing queries on time-partitioned, immutable segments**. ClickHouse is built for the same general goal but with one process type and a sparse-primary-index-driven scan model. Druid pays more operational complexity (6 process types + ZK + deep storage + metadata DB) for stronger out-of-the-box real-time ingest, segment-level tiering, and user-facing concurrency. The right call is workload-dependent.

## How to use this pack

Each file is self-contained. Read `02-architecture-fundamentals.md` first if Druid's process model is new to you, then jump to your weak spot. The "must-internalize" block at the end of each file is the 5-minute review.

## Table of contents

| # | File | Topic | Why it matters |
|---|------|-------|----------------|
| 1 | [01-company-and-product.md](01-company-and-product.md) | Druid origins, Imply, Polaris, customers, where the team is | Frames the "why Druid" answer |
| 2 | [02-architecture-fundamentals.md](02-architecture-fundamentals.md) | The 6 process types, deep storage, metadata DB, ZooKeeper, segment lifecycle | Most interview signal lives here |
| 3 | [03-segments-and-columnar-storage.md](03-segments-and-columnar-storage.md) | Segment format, columnar storage, dictionary encoding, Roaring bitmaps, front coding | Druid's "MergeTree" equivalent |
| 4 | [04-rollup-and-data-modeling.md](04-rollup-and-data-modeling.md) | Rollup, dimensions vs metrics, query granularity, schema-discovery vs explicit | The single most distinctive Druid feature |
| 5 | [05-ingestion-streaming.md](05-ingestion-streaming.md) | Kafka/Kinesis supervisors, exactly-once, lateness handling, task lifecycle | Druid's marquee real-time story |
| 6 | [06-ingestion-batch-and-msq.md](06-ingestion-batch-and-msq.md) | Native batch, Hadoop (legacy), the MSQ engine, INSERT/REPLACE | Modern batch + transformation |
| 7 | [07-indexes-and-datasketches.md](07-indexes-and-datasketches.md) | Bitmap indexes, numeric indexes, JSON columns, DataSketches (HLL/Theta/Quantile/Tuple) | Druid's acceleration toolkit |
| 8 | [08-query-engine-native.md](08-query-engine-native.md) | The 5 native query types (Timeseries, TopN, GroupBy v2, Scan, Search) | Native = fast path |
| 9 | [09-druid-sql.md](09-druid-sql.md) | Calcite-based SQL planner, what's supported, what's not, query → native translation | Most queries arrive as SQL today |
| 10 | [10-joins-lookups-subqueries.md](10-joins-lookups-subqueries.md) | Broadcast hash joins, inline subqueries, lookups, IN-filter optimization | Druid joins are different from CH joins |
| 11 | [11-coordinator-overlord-rules.md](11-coordinator-overlord-rules.md) | Load/drop rules, replication, tier balancing, task assignment | The control plane |
| 12 | [12-compaction-segment-management.md](12-compaction-segment-management.md) | Auto compaction, kill task, marking unused, segment lifecycle in deep storage | Operational core |
| 13 | [13-schema-design-patterns.md](13-schema-design-patterns.md) | Worked schemas for event tracking, time-series, logs, ad-tech, multi-tenant, user-facing | The most concrete content |
| 14 | [14-query-patterns-and-corner-cases.md](14-query-patterns-and-corner-cases.md) | Top-N, distinct, quantiles, funnels, retention, cohorts — each with 2-3 alternatives | The "corner cases + alternatives" content |
| 15 | [15-anti-patterns.md](15-anti-patterns.md) | Tiny segments, mutability, high-card GroupBy, overrollup, no compaction | Avoidable disasters |
| 16 | [16-system-tables-and-observability.md](16-system-tables-and-observability.md) | sys.* schema, JMX metrics, Coordinator console, observability stacks | The on-call view |
| 17 | [17-comparison-with-alternatives.md](17-comparison-with-alternatives.md) | **Detailed Druid-vs-ClickHouse-vs-Pinot, vs Snowflake/BQ/ES** | Cross-references the ClickHouse pack |
| 18 | [18-system-design-questions.md](18-system-design-questions.md) | 12 prompts with full answers | Design-round mainline |
| 19 | [19-sql-and-coding-problems.md](19-sql-and-coding-problems.md) | Druid SQL puzzles + Go-flavored ingest/client code | The coding round |
| 20 | [20-staff-engineer-topics.md](20-staff-engineer-topics.md) | Cost, multi-tenancy, migration, SLOs, ADRs, leadership | Cross-cutting signal |
| 21 | [21-quick-reference-cheatsheets.md](21-quick-reference-cheatsheets.md) | Process matrix, query-type matrix, sketch matrix, rule examples | Night-before review |

## Interview process (typical at Imply)

1. **Recruiter screen** — role fit, comp. Imply hires globally remote.
2. **HM technical screen** — resume deep-dive, why-Druid, one design probe.
3. **Coding round (DSA)** — usual leetcode-medium territory.
4. **Coding round (data/system)** — write a SQL, build an ingest pipeline, design a small subsystem.
5. **System design round** — a Druid-adjacent design problem (real-time analytics backend, user-facing dashboard at high concurrency).
6. **Deep technical round** — Druid internals: segments, ingestion, MSQ, query engine. Goes 3 layers deep.
7. **Cross-functional / behavioral / bar-raiser**.
8. **Sometimes a take-home** — a SQL/schema rewrite or perf analysis.

## High-frequency topic clusters

| Cluster | Probability | Where to study |
|---------|-------------|----------------|
| Druid process model (6 types + ZK + deep storage + metadata) | **Very high** | [02](02-architecture-fundamentals.md) |
| Segment format (columnar, bitmap, dict, front-coded) | **Very high** | [03](03-segments-and-columnar-storage.md) |
| Rollup design (when to roll up, how much) | **Very high** | [04](04-rollup-and-data-modeling.md) |
| Streaming ingestion (Kafka supervisors, lateness) | **Very high** | [05](05-ingestion-streaming.md) |
| MSQ for batch + INSERT/REPLACE | High | [06](06-ingestion-batch-and-msq.md) |
| DataSketches (HLL, Theta, Quantile) | **Very high** | [07](07-indexes-and-datasketches.md) |
| Query type choice (timeseries vs topN vs groupBy) | High | [08](08-query-engine-native.md) |
| Joins / lookups / broadcast model | High | [10](10-joins-lookups-subqueries.md) |
| Coordinator rules / tiering | Medium-high | [11](11-coordinator-overlord-rules.md) |
| Auto-compaction | Medium-high | [12](12-compaction-segment-management.md) |
| Schema design + dimension/metric split | **Very high** | [13](13-schema-design-patterns.md) |
| Anti-patterns + corner cases | **Very high** | [14](14-query-patterns-and-corner-cases.md), [15](15-anti-patterns.md) |
| Druid vs ClickHouse / Pinot | High | [17](17-comparison-with-alternatives.md) |

## The 60-second elevator pitch on Druid (memorize)

> "Apache Druid is a real-time OLAP database designed for sub-second analytical queries over big append-mostly event streams. Data lives in **immutable, time-partitioned segments** (a columnar format with **dictionary-encoded string columns**, **Roaring-bitmap inverted indexes** for every dimension, and front-coded dictionaries for sorted prefixes), stored canonically in **deep storage** (S3/HDFS/GCS) and cached on **Historical nodes** for fast querying. The architecture splits cleanly into a **Master** tier (Coordinator + Overlord) for control, a **Data** tier (MiddleManager/Indexer running ingest tasks; Historical serving segments), and a **Query** tier (Broker fans out to data nodes; optional Router routes by data source). Metadata is in MySQL/Postgres; cluster state coordination is in ZooKeeper. The marquee feature is **streaming ingest with exactly-once semantics** via Kafka/Kinesis supervisor tasks plus **roll-up at ingest**, where Druid can pre-aggregate rows that share the same dimension key into a single row, dramatically shrinking storage. Modern ingestion uses **MSQ** (Multi-Stage Query) — a SQL-based engine that does INSERT/REPLACE with full data-shuffle. Queries are **Calcite-based SQL** that translates to **native** queries (Timeseries, TopN, GroupBy v2, Scan, Search). The big design choices: **immutability** (no updates, only re-ingest), **time-first partitioning** (everything is bucketed by time interval), **broadcast joins only** (no shuffle join), **DataSketches for approximate everything** (HLL distinct count, Theta sketches for set algebra, Quantile sketches for percentiles)."

## The 60-second contrast — Druid vs ClickHouse

> "ClickHouse and Druid are the two biggest real-time OLAP engines and they solve overlapping problems with very different operational shapes. ClickHouse is **one process type, MergeTree-driven, sparse-primary-index based**, with full SQL JOINs (6 algorithms), schema-on-write typed columns, and the cloud model (SharedMergeTree on S3). Druid is **six process types, segment-based, Roaring-bitmap index every dimension, broadcast-only joins, time-partitioning is mandatory, and the rollup feature pre-aggregates at ingest**. ClickHouse wins on SQL ergonomics, raw scan speed for ad-hoc analytics, and operational simplicity. Druid wins on **streaming ingest exactly-once**, **segment-level tiering** for hot/cold, **user-facing concurrency** for very high QPS dashboards, and **DataSketches** as a first-class citizen. ClickHouse has UPDATE/DELETE (heavy mutations, lightweight delete); Druid has no UPDATE/DELETE — you re-ingest the affected time range. For an event-tracking analytics product with millions of dashboard users, Druid is often the right pick; for an observability backend with heavy ad-hoc queries and full SQL, ClickHouse is often the right pick."

## The 6 processes to never confuse

- **Coordinator** — manages segment placement / load / drop / replication on Historicals.
- **Overlord** — manages ingestion tasks (assigns to MiddleManagers / Indexers).
- **Broker** — query router; fans out to Historicals and indexing tasks, merges results.
- **Historical** — serves immutable segments from local cache (downloaded from deep storage).
- **MiddleManager** / **Indexer** — runs ingestion tasks (Peons under MiddleManager; in-process under Indexer).
- **Router** — optional API gateway in front of Brokers/Coordinator/Overlord.

(Plus the **external dependencies**: deep storage, metadata DB, ZooKeeper.)

## Sources used to build this pack

- [Apache Druid docs (druid.apache.org/docs)](https://druid.apache.org/docs/latest/)
- [Druid architecture (latest)](https://druid.apache.org/docs/latest/design/architecture/)
- [Druid processes and servers](https://druid.apache.org/docs/latest/design/processes.html)
- [Segments format](https://druid.apache.org/docs/latest/design/segments/)
- [Schema design tips](https://druid.apache.org/docs/latest/ingestion/schema-design/)
- [DataSketches extension](https://druid.apache.org/docs/latest/development/extensions-core/datasketches-extension.html)
- [MSQ (SQL-based ingestion)](https://druid.apache.org/docs/latest/multi-stage-query/)
- [GroupBy / TopN / Timeseries / Scan / Search queries](https://druid.apache.org/docs/latest/querying/)
- [Auto-compaction](https://druid.apache.org/docs/latest/data-management/automatic-compaction/)
- [Coordinator rules](https://druid.apache.org/docs/latest/operations/rule-configuration/)
- [Imply blog — front-coded dictionaries](https://imply.io/blog/introducing-incremental-encoding-for-apache-druid-dictionary-encoded-columns/)
- [Hellmar Becker's Druid blog](https://blog.hellmar-becker.de/)
- [StarTree — three real-time OLAP databases comparison](https://startree.ai/resources/a-tale-of-three-real-time-olap-databases/)
- [RisingWave / Tinybird comparison posts](https://risingwave.com/blog/big-data-olap-systems-apache-pinot-vs-clickhouse-vs-druid/)
- The companion [ClickHouse pack](../Clickhouse/) for direct contrasts.
