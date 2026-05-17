# 1 · ClickHouse — Company, Product, and Context

ClickHouse is one of the rare companies where the database design *is* the moat. Knowing the business arc helps you frame "why ClickHouse" answers, target the right team, and read the cultural cues in interviews.

## 1.1 History in one paragraph

ClickHouse was built inside **Yandex** (the Russian search company) starting around **2009** by Alexey Milovidov's team to power **Yandex.Metrica**, a Google-Analytics-like product. It needed to ingest billions of events per day and serve sub-second aggregates over them. Existing OLAP options (Vertica, BigQuery, Greenplum) were either too slow, too expensive, or unavailable to them. The team wrote a column-store from scratch in C++. It was open-sourced under Apache 2.0 in **2016**. By 2020 it was being used by **Cloudflare, Uber, eBay, Spotify, CERN, Bloomberg** for analytics and observability. In **2021** Aaron Katz (ex-Elastic), Yury Izrailevsky, and Alexey Milovidov founded **ClickHouse, Inc.** (HQ Bay Area, registered in Delaware), raised ~$300M Series B at a $2B valuation, and launched **ClickHouse Cloud** in 2022 on AWS, then GCP and Azure.

## 1.2 Product portfolio

### Core open-source engine

- **License**: Apache 2.0.
- **Language**: C++23 (heavy template metaprogramming, SIMD via wide intrinsics).
- **Release cadence**: monthly stable releases; Altinity ships LTS-style "stable" builds.
- **Distribution**: official Docker images, .deb/.rpm, Helm charts, an `clickhouse-local` single-binary mode, a `chDB` embedded variant.

### ClickHouse Cloud

- **Architecture**: stateless compute + object storage (S3/GCS/Azure Blob) + ClickHouse Keeper for metadata.
- **Engine**: **SharedMergeTree** (cloud-only) replaces ReplicatedMergeTree.
- **Tiers**: Basic, Scale, Enterprise (latter adds VPC peering, SSO, role-based access, audit log, BYOK).
- **Compute auto-scaling**: scale up/down in seconds; idle services pause to zero.
- **Billing**: per-second compute + per-GB stored + per-GB egress.

### chDB / clickhouse-local

- An embedded ClickHouse — like DuckDB's role, but ClickHouse's engine.
- `clickhouse-local` reads files (CSV, Parquet, JSON, ORC, Avro, Protobuf, MsgPack) directly with full SQL.
- Use case: ad-hoc analytics, ELT, scripting, "ClickHouse in a notebook".

### ecosystem and integrations

- **clickhouse-jdbc** + **clickhouse-go** + **clickhouse-py** (official drivers).
- **Kafka engine** (table that consumes from a Kafka topic).
- **S3 / HDFS / GCS / Azure** table functions for ad-hoc querying of object storage.
- **MaterializedPostgreSQL / MaterializedMySQL** engines for change-data-capture ingest.
- **Iceberg / Delta / Hudi** table function for reading lakehouse tables.
- **Vector**, **Fluent Bit**, **OpenTelemetry**, **Vector.dev**, **Logstash** sinks.
- **Grafana**, **Superset**, **Metabase**, **Tableau**, **Power BI** as BI front-ends.

## 1.3 Who runs ClickHouse in production

Public references:

- **Cloudflare** — DNS analytics, HTTP request analytics, security analytics. Trillions of rows. Multi-region clusters.
- **Uber** — Logs and observability backend (replaced ELK). Petabyte scale.
- **eBay** — internal metrics platform.
- **GitLab** — analytics + logs.
- **PostHog** — entire product-analytics backend.
- **Sentry** — events & search backend.
- **Anthropic, OpenAI** — LLM telemetry (per public talks).
- **Stripe, Lyft, Spotify** — analytics use cases.
- **Cisco**, **Microsoft**, **DigitalOcean** — observability internal stacks.
- **Discord** — message storage migration (later moved off; an interesting case study you should be ready to discuss critically).
- **Tinybird, ClickPipes, Plausible, RudderStack** — products *built on top of* ClickHouse for embedded analytics.

## 1.4 The competitive landscape

| Competitor | Where ClickHouse wins | Where they win |
|------------|------------------------|------------------|
| **Snowflake / BigQuery / Redshift** | Latency (sub-second), cost at high QPS, edge/embed deployment | Better SQL ergonomics, ad-hoc data engineer experience, broader DML |
| **Apache Druid** | Simpler ops (one server type vs 6), better SQL, faster scans | Out-of-the-box real-time ingest from Kafka with exactly-once + better ad-hoc workload management |
| **Apache Pinot** | Simpler ops, ad-hoc SQL queries, lower cold-query latency | Native upserts, user-facing latency under heavy concurrency |
| **Elasticsearch** | 10–100× cheaper at the same scale, true SQL, columnar compression | Full-text search ergonomics, document-oriented model |
| **TimescaleDB** | Scale, real columnar storage, perf | Fully SQL-PostgreSQL-compatible, multi-row transactional UPSERT |
| **InfluxDB** | Order of magnitude faster + cheaper at scale | Purpose-built TSDB ergonomics |
| **DuckDB** | Distributed, server, write-heavy, very large data | Single-machine embedded ergonomics, MIT-license simplicity |
| **StarRocks / Doris** | Maturity, ecosystem, cloud product | Better MPP/joins, mature internal SaaS adoption in China |

The pitch ClickHouse uses: "general-purpose columnar OLAP at real-time analytics latency, with full SQL, that scales from a laptop (chDB) to petabytes (Cloud)".

## 1.5 The 2026 engineering direction

Based on public roadmaps and recent releases:

- **JSON / Dynamic / Variant** types (production in 25.x) — handle semi-structured data natively, with subcolumn projections that get all MergeTree benefits.
- **Refreshable Materialized Views** (production, with APPEND mode) — scheduled refresh for views that can't be incremental.
- **Lightweight updates** (cloud-first) — UPDATE without rewriting the whole part.
- **Vector search / hybrid search** — vector index types, ANN integration; pursuing the LLM RAG market.
- **OpenTelemetry-first** — first-class log/trace/metric ingest with semantic conventions.
- **Iceberg / Delta / Hudi** read + write — playing well with lakehouse architectures.
- **Parallel replicas** — distribute single-query work across multiple replicas of one shard.
- **Stronger transactional semantics** — experimental MVCC / transactions across multiple statements.

## 1.6 Reading the team you're interviewing for

ClickHouse Inc. has several engineering tracks:

- **Core engine** (C++) — query execution, storage formats, codecs. Hardest bar.
- **Cloud / infrastructure** — Kubernetes operators, control plane, S3 storage layer, multi-region operations.
- **Integrations** — connectors (Kafka, OTel, Kinesis), drivers, ecosystem.
- **DevEx / dbt / BI** — schemas, query helpers, integration with Tinybird-like tooling.
- **Data / SRE** — running the internal data lakes, customer-data platform, observability of the cloud.

A staff role will live in one of these. Tune your stories accordingly — a core-engine staff hires for C++ + compiler + lock-free data structures depth; a cloud staff hires for Kubernetes + S3 + multi-tenant SaaS depth.

## 1.7 The 60-second history & why-now

> "ClickHouse was built at Yandex to power Yandex.Metrica — a Google-Analytics-style product that needed sub-second aggregates over billions of events. Open-sourced in 2016, it became the default for real-time analytics and observability — Cloudflare, Uber, Sentry, PostHog, GitLab all run on it. ClickHouse Inc. spun out in 2021 and launched ClickHouse Cloud the next year, which separates compute from storage via the SharedMergeTree engine on top of S3-class object storage. The 'why now' tailwind: the rise of observability (logs/metrics/traces as data), the LLM era's need to ingest and search huge bodies of events, and customers' fatigue with Snowflake-class warehouse latency for user-facing dashboards. ClickHouse fits the niche of sub-second queries over very large append-only data, at one-tenth the cost of warehouse-class systems."

## 1.8 Must-internalize

- Open-sourced 2016, ClickHouse Inc. 2021, ClickHouse Cloud 2022.
- Reference customers: Cloudflare, Uber, Sentry, PostHog, GitLab, Stripe — they signal what the platform is good at (analytics, observability, user-facing dashboards).
- Product surface: OSS engine, ClickHouse Cloud, chDB embedded, ecosystem connectors.
- Engine families to know: MergeTree (default), ReplicatedMergeTree (HA), SharedMergeTree (Cloud).
- 2026 direction: native JSON/Dynamic, refreshable MVs, lightweight updates, vector search, OpenTelemetry, parallel replicas.
- Where it wins vs. Snowflake: latency + cost + user-facing concurrency. Where it loses: full DML and warehouse-class ad-hoc data engineering ergonomics.

---

## Sources

- [ClickHouse — company page](https://clickhouse.com/company)
- [Aaron Katz / Tanya Bragin interview — Shriftman](https://shriftman.substack.com/p/clickhouse-is-the-engine-behind-ai)
- [From Open Source to SaaS: ClickHouse journey (InfoQ)](https://www.infoq.com/presentations/open-source-saas/)
- [Yandex.Metrica origin story (ClickHouse blog)](https://clickhouse.com/blog/the-billion-row-club-with-bil-clemons)
- [ClickHouse Cloud architecture](https://clickhouse.com/cloud)
- [chDB embedded](https://github.com/chdb-io/chdb)
- [Adopters list (official)](https://clickhouse.com/docs/about-us/adopters)
