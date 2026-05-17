# 17 · Comparison with Alternatives — Why ClickHouse over X

The "why ClickHouse, not Snowflake / Druid / Pinot / BigQuery / Elasticsearch" question is unavoidable. A staff engineer should be able to give an *honest* answer that includes where ClickHouse loses.

## 17.1 The big map

| System | Type | Storage | Ingest | Query latency | Concurrency | DML | Operational complexity | Cost |
|--------|------|---------|--------|---------------|-------------|-----|------------------------|------|
| **ClickHouse** | OLAP | Local / S3 (Cloud) | Append-batch | Sub-second over big agg | Hundreds (Cloud unlimited) | Limited (mutations + lightweight DELETE/UPDATE) | Medium (lower with Cloud) | Low |
| **Druid** | Real-time OLAP | HDFS / S3 | Real-time + batch | Sub-second | High | Re-ingest only | High (5-6 node types + ZK) | Medium |
| **Pinot** | Real-time OLAP | Local + segments | Real-time + batch | Very low latency | Very high (user-facing) | Append + UPSERT | High (multi-node-type, controller, broker) | Medium |
| **Snowflake** | Cloud DW | Proprietary | Batch + Snowpipe streaming | Seconds | Limited (concurrency credits) | Full DML | Low (managed) | High |
| **BigQuery** | Cloud DW | Capacitor (columnar) | Streaming / batch | Seconds (BI Engine: sub-sec) | Slot-based | Full DML | Low (managed, serverless) | Pay-per-query or slot reservations |
| **Redshift** | MPP DW | Local / RA3 with S3 | COPY from S3 | Seconds | Limited | Full DML | Medium | Medium |
| **Elasticsearch** | Search / log | Local | Real-time | Variable; great for search | Medium | Eventually consistent | High at scale | High at scale |
| **InfluxDB** | TSDB | Local | Real-time | Fast for TS | Medium | Append + delete | Low-Medium | Low-Medium |
| **TimescaleDB** | TS on Postgres | Local (postgres) | Real-time | Decent | Postgres limits | Full SQL/Postgres | Low | Low |
| **DuckDB** | Embedded OLAP | Local | Single-machine | Sub-second small data | Single-process | Full DML | Trivial | Free |
| **StarRocks / Doris** | MPP OLAP | Local + S3 | Real-time + batch | Sub-second | High | Full DML + UPSERT | Medium | Low |

## 17.2 ClickHouse vs Snowflake / BigQuery / Redshift (cloud warehouses)

Where ClickHouse wins:
- **Latency**: warehouses are tuned for seconds-to-minutes; CH is tuned for ms-to-second.
- **Concurrency**: 100s of QPS sustained, vs warehouses' compute-credit-bounded model.
- **Cost**: at high QPS or constant utilization, CH is 5-10× cheaper.
- **User-facing dashboards**: warehouses are not designed to serve embedded BI to millions of users; CH is.
- **Real-time ingest**: warehouses charge a premium (Snowpipe Streaming) for what CH does naturally.

Where warehouses win:
- **Ad-hoc SQL ergonomics**: full DML, transactions, snapshots, time-travel.
- **Data engineering features**: zero-copy cloning, sharing across accounts, secure data exchange.
- **Strong ACID + multi-statement transactions.**
- **Managed service maturity**: less to operate.
- **Joins of large warehouse tables**: warehouses' query planners handle these better.
- **Compliance frameworks**: warehouses have more out-of-the-box.

Common pattern: warehouses for the "data warehouse" workloads (ELT, ad-hoc analyst queries, BI on big slow data), CH for the "real-time analytics application" workloads (embedded BI, customer-facing dashboards, observability).

## 17.3 ClickHouse vs Druid

Both are real-time OLAP. Differences:

- **Architecture**: Druid has 6 server types (Coordinator, Overlord, Broker, Historical, MiddleManager, Router) + ZooKeeper + a metadata DB. CH has 1 server type + Keeper. Druid is more operationally complex.
- **SQL**: Druid SQL is a subset; CH SQL is full(ish) ANSI-ish.
- **Ingest**: Druid has stronger out-of-the-box real-time ingest from Kafka with exactly-once. CH does the same with Kafka engine + MV but with at-least-once + dedup.
- **Updates**: Druid has no UPDATE/DELETE — you re-ingest a segment. CH has mutations + lightweight delete.
- **Scan speed**: CH typically wins; Druid optimizes more for indexed lookups.
- **User-facing concurrency**: Druid is robust here; CH catches up.

Pick CH when SQL ergonomics and ad-hoc flexibility matter more than batch-segment-managing UX. Pick Druid for high-concurrency dashboards on heavily pre-aggregated data where its data flow fits.

## 17.4 ClickHouse vs Pinot

Both are real-time OLAP. Pinot is the "user-facing low-latency" champ.

- **Architecture**: Pinot has Controller, Broker, Server, Minion + ZK. More moving parts.
- **Ingest**: Pinot's Kafka real-time path is mature; CH is mature too via Kafka engine.
- **Concurrency**: Pinot edges in *very-high-QPS user-facing*; CH catches up.
- **SQL**: Pinot SQL is narrower; CH is broader.
- **Indexes**: Pinot has rich pluggable indexes (inverted, range, geospatial, JSON, text); CH has fewer but well-tuned ones (sparse PK, skip indexes, projections).
- **UPSERT**: Pinot supports native UPSERT; CH does it via ReplacingMergeTree.

Pick Pinot when you need *latency parity with serving systems* at very high QPS and have the ops bandwidth. Pick CH for breadth and operational simplicity.

## 17.5 ClickHouse vs Elasticsearch

Elastic is search-first; CH is analytics-first. People migrate from ELK to CH for logs/metrics:

- **Storage cost**: CH 5-10× cheaper for the same data (better compression).
- **Aggregation speed**: CH faster for `count`/`sum`/`uniq` over millions of docs.
- **Full-text search**: Elastic still wins on document-oriented ranked search; CH has bloom-style token indexes + the new `text` index but doesn't compete on relevance ranking.
- **SQL**: real SQL in CH; Elastic SQL is a wrapper around its DSL.
- **Ops at scale**: Elastic clusters get painful at petabyte scale; CH stays simpler.

Pick ES when "search relevance" is the use case. Pick CH when "analytics over log/event data" is. Many shops run both.

## 17.6 ClickHouse vs InfluxDB / TimescaleDB

For time series specifically:

- **InfluxDB v2/v3**: purpose-built TSDB with Flux. v3 (IOx) is column-store like CH. CH typically beats it on raw scale and on flexible joins.
- **TimescaleDB**: Postgres extension with hypertables. Postgres SQL ergonomics; relatively limited columnar (compression added recently).
- **CH**: not a TSDB but excels at time-series workloads via Delta/Gorilla codecs + MergeTree.

Pick TimescaleDB if you want a Postgres-compatible TSDB; pick CH for scale and flexibility; pick Influx for purpose-built TSDB ergonomics on smaller datasets.

## 17.7 ClickHouse vs DuckDB

Both columnar, vectorized, ZSTD-class compression. Different sweet spots:

- **DuckDB**: in-process, single-machine, MIT license, fantastic CSV/Parquet reading, perfect for ad-hoc / notebook / ELT.
- **ClickHouse**: server, distributed, scales to petabytes.
- **chDB** (CH embedded variant): brings CH's engine to the DuckDB-style use case.

Use DuckDB for: analyst notebooks, small data, embedded analytics in an app.
Use ClickHouse for: server-side analytics, real-time ingest, multi-user.

## 17.8 ClickHouse vs StarRocks / Doris

Younger MPP-style columnar DBs out of the Chinese ecosystem. They've made strides:

- **MPP joins**: StarRocks has classic MPP shuffle joins; can do better than CH on big-table joins.
- **UPSERT**: native primary-key UPSERT models.
- **Adoption outside China**: still catching up.

Pick CH when ecosystem maturity and OSS adoption matter; consider StarRocks/Doris if MPP joins or native UPSERT are central.

## 17.9 The "when not ClickHouse" honest answers

- **OLTP**: high-QPS small-row writes/reads with transactions. Use Postgres / MySQL / Cockroach.
- **Search relevance ranking**: Use Elasticsearch / OpenSearch / Vespa.
- **Document-oriented per-document queries**: Use MongoDB / Couchbase.
- **Single-machine analytical scripting**: DuckDB.
- **Federated query across many sources**: Trino / Presto.
- **Strict ACID multi-statement transactions**: Snowflake / BigQuery / a real RDBMS.

## 17.10 Interview pitch

> "ClickHouse is the right call when you need sub-second analytical queries over very-large append-mostly data, at price-points that warehouses can't match, with operational complexity in between a warehouse (managed) and a Druid/Pinot (multi-server-type). The wins are columnar compression + sparse PK + vectorized exec + a rich aggregation language. The honest losses are: no multi-statement transactions, weak point-update model (mutations are heavy; lightweight is improving), search relevance ranking is not its game, and operational simplicity at extreme scale is still less than a managed warehouse."

## 17.11 Must-internalize

- CH wins: latency, cost, concurrency, real-time ingest.
- CH loses: full DML, transactions, search relevance, single-machine notebook ergonomics.
- Warehouses for ELT/ad-hoc; CH for user-facing real-time analytics.
- Druid/Pinot are CH's nearest peers; trade SQL ergonomics for ingest-pipeline maturity.
- Elastic for search; CH for analytics; some shops run both.
- DuckDB for embedded/notebook; chDB bridges the gap.

---

## Sources

- [Roman Leventov — OSS OLAP comparison (still cited)](https://leventov.medium.com/comparison-of-the-open-source-olap-systems-for-big-data-clickhouse-druid-and-pinot-8e042a5ed1c7)
- [How to choose a database for real-time analytics (CH blog)](https://clickhouse.com/resources/engineering/how-to-choose-a-database-for-real-time-analytics-in-2026)
- [Big Data OLAP systems (RisingWave)](https://risingwave.com/blog/big-data-olap-systems-apache-pinot-vs-clickhouse-vs-druid/)
- [A tale of three real-time OLAP DBs (StarTree)](https://startree.ai/resources/a-tale-of-three-real-time-olap-databases/)
- [The Great OLAP Divide (TechLife)](https://techlife.blog/posts/olap-database-comparison/)
