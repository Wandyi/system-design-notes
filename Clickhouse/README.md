# ClickHouse — Staff Software Engineer Interview Deep Dive

A complete preparation pack for a Staff Software Engineer interview at **ClickHouse, Inc.** Covers the engine internals, the cloud product (SharedMergeTree, stateless compute), schema-design playbooks for every common analytical scenario, query patterns with their corner cases and alternative solutions, and 12 design-round prompts with full answers.

ClickHouse is a columnar OLAP DBMS originally built at Yandex (2009, OSS 2016), spun out as ClickHouse, Inc. in 2021, and now powering the **ClickHouse Cloud** managed service. The engine has become the de-facto default for real-time analytics, observability backends (Cloudflare, Uber, eBay, GitLab, Sentry, PostHog all run on it), and any workload that does big-aggregate-over-billions-of-rows in sub-second.

At staff level the bar is *internalized depth* on MergeTree mechanics + the ability to pick the right schema/engine/index for any analytical workload + judgment about cost, correctness and operational tradeoffs.

## How to use this pack

Each file is self-contained. Hit `02-architecture-fundamentals.md` first if you've never been deep in ClickHouse; otherwise skip to your weak spots. The "must-internalize" section at the end of every file is the highest-leverage 5 minutes per topic.

## Table of contents

| # | File | Topic | Why it matters |
|---|------|-------|----------------|
| 1 | [01-company-and-product.md](01-company-and-product.md) | ClickHouse Inc, OSS, Cloud, customers, why now | Frames every "why ClickHouse" answer |
| 2 | [02-architecture-fundamentals.md](02-architecture-fundamentals.md) | Columnar layout, vectorized execution, MergeTree, parts, granules, marks, sparse PK | The single most important file. Most interview signal lives here |
| 3 | [03-table-engines.md](03-table-engines.md) | Full MergeTree family + special engines + integration engines | Engine choice is a constant interview probe |
| 4 | [04-data-types.md](04-data-types.md) | All scalar/compound types, LowCardinality, Nested, JSON/Dynamic/Variant, Geo | Bad type choice = 10× storage, 10× scan cost |
| 5 | [05-codecs-and-compression.md](05-codecs-and-compression.md) | LZ4, ZSTD, Delta, DoubleDelta, Gorilla, T64, FPC | Compression is half the speed story |
| 6 | [06-indexes-projections-and-mvs.md](06-indexes-projections-and-mvs.md) | Sparse PK, skip indexes (bloom/tokenbf/ngrambf/minmax/set), projections, materialized views | The acceleration layer |
| 7 | [07-replication-and-keeper.md](07-replication-and-keeper.md) | Keeper (Raft), ReplicatedMergeTree, quorum, distributed DDL | Replication = correctness + availability |
| 8 | [08-sharding-and-distributed.md](08-sharding-and-distributed.md) | Shard keys, Distributed engine, GLOBAL IN/JOIN, parallel replicas | How ClickHouse scales horizontally |
| 9 | [09-cloud-and-sharedmergetree.md](09-cloud-and-sharedmergetree.md) | ClickHouse Cloud, SharedMergeTree, stateless compute, S3 storage | What ClickHouse builds new code against |
| 10 | [10-joins.md](10-joins.md) | All 6 algorithms (hash, parallel hash, grace hash, partial merge, full sorting merge, direct), join selection rubric | Joins are the most-misunderstood piece |
| 11 | [11-query-optimization.md](11-query-optimization.md) | PREWHERE, FINAL, max_threads, memory limits, EXPLAIN, profile, settings | The hot-path interview |
| 12 | [12-mutations-deletes-updates.md](12-mutations-deletes-updates.md) | Heavy mutations, lightweight DELETE/UPDATE, TTL DELETE/MOVE, ON CLUSTER | Mutability in an immutable engine |
| 13 | [13-schema-design-patterns.md](13-schema-design-patterns.md) | Time-series, event tracking, logs, user sessions, wide tables, denormalization, multi-tenant | Concrete schema scenarios with worked DDL |
| 14 | [14-query-patterns-and-corner-cases.md](14-query-patterns-and-corner-cases.md) | Top-N, percentiles, retention, funnels, gap filling, pivots, latest-state — each with 2-3 alternative solutions | The "corner cases and alternatives" the prompt asked for |
| 15 | [15-anti-patterns.md](15-anti-patterns.md) | Small inserts, OPTIMIZE FINAL, high-cardinality PK, joining whales, naive SELECT *, mutations on hot tables | Avoidable disasters |
| 16 | [16-system-tables-and-observability.md](16-system-tables-and-observability.md) | system.query_log, parts, merges, replication_queue, mutations, asynchronous_metrics | What the on-call watches |
| 17 | [17-comparison-with-alternatives.md](17-comparison-with-alternatives.md) | vs Druid, Pinot, Snowflake, BigQuery, Redshift, Elasticsearch, Influx/Timescale, DuckDB | "Why ClickHouse over X" |
| 18 | [18-system-design-questions.md](18-system-design-questions.md) | 12 prompts with full answers (observability backend, ad analytics, log search, e-commerce funnel, etc.) | Design-round mainline |
| 19 | [19-sql-and-coding-problems.md](19-sql-and-coding-problems.md) | SQL puzzles + Go-flavored client-side coding (ingest pipeline, async client, sharding router) | The coding round |
| 20 | [20-staff-engineer-topics.md](20-staff-engineer-topics.md) | Cost, multi-tenancy, migration, SLOs, leadership, ADRs | The cross-cutting signal |
| 21 | [21-quick-reference-cheatsheets.md](21-quick-reference-cheatsheets.md) | Engine matrix, codec table, settings, type sizes, RFCs, system tables | Night-before review |

## Interview process (typical for ClickHouse / Altinity / OSS-DB roles)

1. **Recruiter screen** — role fit, comp. ClickHouse Inc. mostly hires globally remote.
2. **Hiring-manager technical screen** — resume deep dive, why-ClickHouse, one shallow technical probe.
3. **Coding round** (1–2 × 60 min) — usually one DSA (medium) + one **SQL/schema** problem in ClickHouse syntax. Bring a real CH installation up for the coding screen; explain `EXPLAIN`.
4. **System / database design round** — design an analytics backend, an observability pipeline, a real-time metrics store, an e-commerce funnel. The interviewer wants you to *pick the right engine, sketch the schema, name the bottleneck, and explain the failure modes*.
5. **Deep technical round** — MergeTree internals: parts, merges, primary index, projections, materialized views, joins. They probe 3 layers deep.
6. **Cross-functional / behavioral / bar-raiser** — same as Infoblox & streaming pack: STAR stories, conflict, ambiguity.
7. **(Sometimes) take-home**: a small write-up about a query you'd optimize, or a schema you'd redesign.

## High-frequency topic clusters

| Cluster | Probability | Where to study |
|---------|-------------|----------------|
| MergeTree mechanics (parts, granules, primary index, merges) | **Very high** | [02-architecture-fundamentals.md](02-architecture-fundamentals.md) |
| Picking the right engine for a scenario | **Very high** | [03-table-engines.md](03-table-engines.md), [13-schema-design-patterns.md](13-schema-design-patterns.md) |
| Schema design + ORDER BY + partitioning | **Very high** | [13-schema-design-patterns.md](13-schema-design-patterns.md) |
| Materialized views + AggregatingMergeTree | **Very high** | [06-indexes-projections-and-mvs.md](06-indexes-projections-and-mvs.md) |
| Joins + when to denormalize vs dictionary | High | [10-joins.md](10-joins.md) |
| Replication / Keeper / cluster topology | High | [07-replication-and-keeper.md](07-replication-and-keeper.md) |
| SharedMergeTree + ClickHouse Cloud | **Very high** for ClickHouse Inc | [09-cloud-and-sharedmergetree.md](09-cloud-and-sharedmergetree.md) |
| Query optimization (PREWHERE, FINAL, settings) | High | [11-query-optimization.md](11-query-optimization.md) |
| Mutations & lightweight ops | Medium-high | [12-mutations-deletes-updates.md](12-mutations-deletes-updates.md) |
| Anti-patterns + corner cases | **Very high** | [14-query-patterns-and-corner-cases.md](14-query-patterns-and-corner-cases.md), [15-anti-patterns.md](15-anti-patterns.md) |
| Comparisons (Druid/Pinot/Snowflake/BQ) | Medium | [17-comparison-with-alternatives.md](17-comparison-with-alternatives.md) |
| Multi-tenant SaaS isolation | Medium-high for Cloud roles | [20-staff-engineer-topics.md](20-staff-engineer-topics.md) |

## The 60-second elevator pitch on ClickHouse (memorize)

> "ClickHouse is a columnar OLAP DBMS where the dominant table engine — `MergeTree` — stores data as **immutable, sorted parts**, with a **sparse primary index** (one mark per 8192-row 'granule') that fits in RAM and a **background-merge** thread that combines parts into bigger ones. Queries are **vectorized** (SIMD-friendly batch processing), read only the columns they need (columnar layout), and skip granules using both the primary index and **data-skipping indexes**. Replication is async, coordinated by **ClickHouse Keeper** (a Raft-based ZooKeeper replacement). The cloud product separates compute and storage: **SharedMergeTree** replaces ReplicatedMergeTree, storing data on S3/GCS/Azure object store, with metadata in Keeper — stateless compute nodes can scale to hundreds of replicas without re-replicating data. The defining design choices: **append-mostly** (mutations are expensive, lightweight delete is a row-level tombstone), **column-store** (compresses 10-100× on real data), **batch-favoring** (one big INSERT >> a thousand small ones; the merger can't catch up otherwise), and **pre-aggregate when possible** (materialized views feeding AggregatingMergeTree are the canonical acceleration). Engineering wisdom: get ORDER BY right (sort key = primary key by default), denormalize liberally, use dictionaries for small lookup tables, prefer one wide flat table over a star schema, and never panic-OPTIMIZE FINAL on a hot table."

## The 60-second pitch — ClickHouse Cloud / SharedMergeTree

> "ClickHouse Cloud is a stateless-compute / object-storage architecture. Compute nodes have no local state — they read from and write to S3 (or GCS/Azure), and coordinate via ClickHouse Keeper for metadata. The cloud engine is **SharedMergeTree** which replaces ReplicatedMergeTree: instead of every replica owning a copy of the data and gossiping via Keeper, all replicas share the data on S3 and Keeper just owns metadata. Three big wins: (1) replicas spin up in seconds because there's no data to copy, (2) you can have 100s of replicas without paying replicated storage, (3) the metadata path is leaderless — any replica can ingest and others see it via Keeper updates. This came with **lightweight updates and deletes** that don't require part rewrites, **stateless restarts**, and **fast scale-up** for query bursts. The trade-off vs. on-prem: every read may touch S3 (mitigated by a local page cache), and you pay attention-tax on S3 cost / API rate limits."

## The four engines to never confuse

- **MergeTree** — the default, append-only, sorted, merged in background.
- **ReplicatedMergeTree** — MergeTree + Keeper-coordinated async replication. Required for HA.
- **SharedMergeTree** — Cloud-only variant: data on S3, metadata in Keeper, stateless compute. Replaces ReplicatedMergeTree in Cloud.
- **Distributed** — a virtual / proxy engine. Routes queries to underlying shards. Holds no data itself.

Mixing these up in an interview is a tell. Practice saying each name in context.

## Sources used to build this pack

- [ClickHouse docs (clickhouse.com/docs)](https://clickhouse.com/docs)
- [ClickHouse Cloud — SharedMergeTree](https://clickhouse.com/docs/cloud/reference/shared-merge-tree)
- [ClickHouse Cloud stateless compute blog](https://clickhouse.com/blog/clickhouse-cloud-stateless-compute)
- [ClickHouse Keeper — Raft-based ZK replacement](https://clickhouse.com/blog/clickhouse-keeper-a-zookeeper-alternative-written-in-cpp)
- [ClickHouse Joins Under the Hood — 5-part series](https://clickhouse.com/blog/clickhouse-fully-supports-joins-hash-joins-part2)
- [Sparse primary indexes — practical introduction](https://clickhouse.com/docs/guides/best-practices/sparse-primary-indexes)
- [Schema design — official docs](https://clickhouse.com/docs/data-modeling/schema-design)
- [Definitive guide to query optimization (2026)](https://clickhouse.com/resources/engineering/clickhouse-query-optimisation-definitive-guide)
- [JSON / Dynamic / Variant — new powerful JSON data type](https://clickhouse.com/blog/a-new-powerful-json-data-type-for-clickhouse)
- [Materialized views guide (BigData Boutique)](https://bigdataboutique.com/blog/clickhouse-materialized-views-guide)
- [Mutations and lightweight delete](https://clickhouse.com/docs/guides/developer/mutations)
- [Glassdoor — ClickHouse interview reports](https://www.glassdoor.com/Interview/ClickHouse-Interview-Questions-E6022924.htm)
- Altinity blog, BigData Boutique, OneUptime CH series, PostHog handbook
