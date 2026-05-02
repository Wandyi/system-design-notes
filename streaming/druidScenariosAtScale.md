# Apache Druid — Realistic Scenarios at Staff-Engineer Depth

> A practical, opinionated reference for Apache Druid at MAANG scale — when it's the right tool, when it's a costly mistake, and how to combine its features (rollup, sketches, tiering, compaction, multi-stage queries) for the workloads it was actually designed for: real-time slice-and-dice OLAP over time-series event data, with sub-second latency on TB–PB datasets. Written for staff engineers past "Druid is a time-series DB" who now need to size clusters for 10M events/sec, defend against the ClickHouse/Pinot question in design reviews, and operate a system whose ZooKeeper hiccups can take down query traffic.

> Companion to `awsS3ScenariosAtScale.md`, `snsSqsEventBridgeAtScale.md`, `dynamoDBScenariosAtScale.md`, `eventPlatformsAtScale.md`, `logProcessingAndAggregation.md`. Druid sits at the intersection of streaming ingestion (Kafka), columnar storage (S3 + segments), and OLAP querying — the few-but-distinct slot it owns in the modern data stack.

---

## 0. The Staff-Level Frame

Druid is **not** a general-purpose database. It is a purpose-built engine for one shape of problem: *"slice-and-dice over time, with low latency, on append-only event data, where most queries filter by a time range and group by some dimensions."* Every architectural decision in Druid encodes this assumption.

At staff level the questions are:

1. **Is the workload actually OLAP-shaped?** (Time-bucketed events, group-by dimensions, aggregate metrics. Not row-by-row lookup. Not transactional.)
2. **What's the cardinality profile?** (Druid handles billions of rows fine; high-cardinality dimensions and group-by exact-distinct kill it.)
3. **What's the latency target?** (P95 sub-second on dashboards, ad-hoc < 10s; batch reindex hours.)
4. **What's the data freshness target?** (Real-time = sub-second, but at cost. Near-real-time = 1-min lag is the cheapest knob.)
5. **What's the operational cost?** (Druid is a 6-process distributed system with ZooKeeper, metadata DB, deep storage. Operationally heavy. People-hours matter.)
6. **What about ClickHouse, Pinot, BigQuery, Snowflake?** (Each wins in specific shapes; staff engineers must defend the choice.)

The mistake everyone makes: reaching for Druid because "it's fast at analytics" without checking whether the workload's *shape* matches Druid's strengths. Druid is exquisite at its niche and miserable outside it.

---

## 1. Mental Model — What Druid Is and Isn't

### 1.1 What Druid is

A **time-partitioned, columnar, immutable-segment, distributed OLAP engine** with:
- **Real-time + batch ingestion** (Kafka native; HDFS/S3 batch; native push).
- **Sub-second p95 query latency** on TB datasets, p99 single-digit-second on PB.
- **Bitmap-indexed dimensions** for fast filter+group-by.
- **Pre-aggregation (rollup)** at ingest time — collapses ~10× rows for dashboard datasets.
- **Approximation libraries** (HyperLogLog, Theta sketches, Quantile sketches) for high-cardinality counts.
- **Tiered storage** (hot/cold tiers; segments in S3-compatible deep storage).
- **SQL via Calcite** (Druid SQL).
- **Multi-Stage Query Engine (MSQ)** since 2022 for heavy reindex / large GROUP BY / SQL-based ingestion.

Used at: Airbnb, Netflix, Pinterest, Walmart, Lyft, Confluent, Yahoo (where it originated). The "internal-dashboard-and-anomaly-detection" tier at most data-heavy companies.

### 1.2 What Druid is NOT

- **Not a transactional DB.** No updates, no deletes (mostly). Everything is append-only with eventual reindex.
- **Not a data warehouse.** Doesn't handle complex joins (limited, slow), doesn't support arbitrary SQL well.
- **Not Elasticsearch.** Druid does aggregates, ES does search. Don't full-text in Druid.
- **Not Kafka.** Druid consumes from Kafka; doesn't replace it.
- **Not a low-cardinality KV store.** Single-row lookups by ID are slow; that's not its job.
- **Not BigQuery/Snowflake.** Those are massively-parallel SQL engines on their own scale-out storage. Druid is faster for dashboards; slower for arbitrary SQL/joins.

### 1.3 The architecture that follows from the model

```
┌─────────────────────────────────────────────────────────────┐
│ INGESTION TIER                                              │
│  ┌──────────┐    ┌────────────┐    ┌───────────────────┐    │
│  │ Overlord │───►│ MiddleMgr  │───►│ Peon (per-task)   │    │
│  │ (master) │    │ (workers)  │    │ task = ingestion  │    │
│  └──────────┘    └────────────┘    └────────┬──────────┘    │
│                                             │ writes        │
└─────────────────────────────────────────────┼───────────────┘
                                              ▼
                                    ┌──────────────────────┐
                                    │ Deep Storage (S3/HDFS) │
                                    │ Immutable segments     │
                                    └──────────┬───────────┘
                                               │ load
┌──────────────────────────────────────────────┼───────────────┐
│ QUERY TIER                                   ▼               │
│  ┌──────────┐    ┌────────────┐    ┌───────────────────┐    │
│  │  Router  │───►│   Broker   │───►│    Historical     │    │
│  │ (optional)│    │ (planner)  │    │ (segment server)  │    │
│  └──────────┘    └────────────┘    └───────────────────┘    │
│       ▲                                                       │
│       │ HTTP/SQL                                              │
│       Client (Superset, Grafana, app)                         │
└───────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ COORDINATION                                                 │
│  Coordinator (segment placement, balancing)                  │
│  Metadata DB (Postgres/MySQL — segment manifest)             │
│  ZooKeeper (cluster state, leader election, task assignment) │
└─────────────────────────────────────────────────────────────┘
```

Every process has one job. Failures of any single tier degrade differently:
- **Historical**: serving fewer queries, can rebalance.
- **Broker**: query path degraded; replicas help.
- **Coordinator**: segment loading paused; queries continue.
- **Overlord**: ingestion paused; queries continue.
- **ZooKeeper**: catastrophic; queries and ingestion both fail.
- **Metadata DB**: catastrophic.
- **Deep storage**: data still queryable from Historical caches; new segments can't load.

A staff engineer designs around these dependencies and tests each failure mode.

### 1.4 The data model

```
Datasource: like a table.
  Time column: mandatory (Druid is partitioned by time).
  Dimensions: strings, multi-value strings, numbers used for filter/group.
  Metrics: pre-aggregated numerical columns (count, sum, min, max, sketches).

Segment: time-partitioned, columnar, immutable file.
  Default 5M rows; sized to ~500MB-1GB on disk.
  Time chunk: e.g., 1 hour or 1 day.
  Number of segments per chunk: shards/replicas.
```

**Rollup**: at ingest, rows with the same (time-bucket, all-dimensions) are collapsed; metrics summed/added.

```
Raw events:
  t=12:00:01 user=A page=home → 1 view
  t=12:00:05 user=A page=home → 1 view
  t=12:00:09 user=A page=home → 1 view

After rollup at minute granularity:
  t=12:00:00 user=A page=home → 3 views
```

3:1 reduction here; in real workloads 10–100× compression is typical for dashboard data.

### 1.5 Quick reference

| Aspect | Value / shape |
|---|---|
| **Per-broker query throughput** | 100s–1Ks QPS |
| **p50 latency** | 10–100 ms |
| **p99 latency** | 500 ms – few sec |
| **Ingestion rate** | 1M+ events/sec/cluster (with replicas, more) |
| **Data scale** | TBs to PBs (commonly 10s of TB hot, PBs cold) |
| **Segment size** | 500 MB – 1 GB target |
| **Time chunk** | usually 1h, 1d, or 1mo |
| **Hot tier storage** | local SSD on Historicals (NVMe ideally) |
| **Deep storage** | S3 / HDFS / GCS |
| **Query path** | client → router → broker → historicals → broker (merge) → client |
| **Cost shape** | Historical EC2 dominates; deep storage cheap; ingestion EC2 second |

---

## 2. Scenario 1 — Real-Time Clickstream Analytics

### 2.1 The problem

E-commerce site. Events: `page_view`, `product_view`, `add_to_cart`, `checkout_start`, `purchase`. 10M events/sec at peak. Dashboards must show:
- Real-time conversion funnel (events of last 5 min).
- Top pages last hour.
- Hourly/daily revenue.
- A/B test cohort results.
- p95 page-load time per geo.

Latency: dashboards refresh < 5 sec; data freshness < 30 sec.

### 2.2 Why Druid fits

- Event data with a time column ✓
- Group-by dimensions (page, geo, A/B group) ✓
- Aggregate metrics (count, sum revenue, p95 latency) ✓
- Sub-second dashboards ✓
- 10M events/sec ingestion ✓

This is Druid's home turf.

### 2.3 The pipeline

```
Browser/App → Kafka topic (clickstream) → Druid Kafka indexing service
                                                  │
                                                  ▼
                                          Real-time segments
                                                  │
                                                  ▼ (handoff after time chunk)
                                          Deep storage (S3) + Historical
                                                  
Queries: Broker fans out to Historicals + real-time tasks → merges → returns.
```

`Kafka indexing service` is Druid's native streaming ingestion: exactly-once-ish processing using Kafka offsets stored in Druid metadata.

### 2.4 The schema design

```yaml
dataSource: clickstream
timestampColumn: event_time
dimensions:
  - user_id              # high cardinality; consider hashing
  - session_id
  - page
  - product_id
  - geo_country
  - geo_city
  - device_type
  - ab_test_group
metrics:
  - name: count
    type: count
  - name: revenue_sum
    type: doubleSum
    fieldName: revenue
  - name: page_load_p95
    type: quantilesDoublesSketch
    fieldName: page_load_ms
  - name: unique_users
    type: HLLSketch
    fieldName: user_id
queryGranularity: minute
segmentGranularity: hour
```

Rollup at minute granularity: every (minute, page, geo, ab_group, ...) is one row.

**HLL sketch for unique users**: exact distinct on `user_id` would be ruinous; HLL gives ~1% error in bytes per group.

**Quantile sketch for p95 latency**: aggregating individual latencies → sketch lets later queries compute any quantile.

### 2.5 Funnel analysis pattern

```sql
-- Conversion funnel in last hour
SELECT
  ab_test_group,
  SUM(CASE WHEN page = '/landing' THEN count ELSE 0 END) as landings,
  SUM(CASE WHEN page = '/cart' THEN count ELSE 0 END) as carts,
  SUM(CASE WHEN page = '/checkout' THEN count ELSE 0 END) as checkouts,
  SUM(CASE WHEN page = '/success' THEN count ELSE 0 END) as conversions
FROM clickstream
WHERE __time >= CURRENT_TIMESTAMP - INTERVAL '1' HOUR
GROUP BY ab_test_group;
```

This runs in 100s of ms on a properly-sized cluster. Same query in Postgres on raw events: minutes.

### 2.6 Trade-offs and alternatives

| Approach | Trade |
|---|---|
| **Druid** | Sub-second dashboards; fixed access patterns; ops cost |
| **ClickHouse** | Similar role; richer SQL; less mature streaming |
| **Pinot** | Similar role; LinkedIn lineage; user-facing analytics niche |
| **BigQuery** | Dollar-per-query; cheap at low volume; expensive at high QPS |
| **Snowflake** | Same as BigQuery; warehouse rather than dashboard |
| **Postgres + materialized views** | Limited scale; familiar |
| **Athena** | S3-native; high latency; cheap; no real-time |

### 2.7 What I'd actually do

For real-time clickstream at 10M events/sec:
- **Kafka** as the buffer (not Kinesis at that scale).
- **Druid** for real-time dashboards and funnel queries.
- **S3 (Parquet, Iceberg)** for raw event archive (long-term, ad-hoc analytics via Athena).
- **Druid auto-compaction** to roll up older segments more aggressively.
- **Hot/cold tiering**: last 7 days on NVMe Historicals; older on cheaper EBS / cold tier.

Druid serves the dashboards; S3 serves the deep-history queries.

---

## 3. Scenario 2 — Ad Analytics / Impression Tracking

### 3.1 The problem

Ad platform. Tracks impressions, clicks, conversions per (advertiser, campaign, creative, geo, hour). Volume: 1M impressions/sec peak. Advertisers see real-time dashboards with:
- Spend, impressions, CTR, CPM, CPC, CVR.
- Top creatives by ad-group.
- Geographic heatmaps.
- p99 latency of ad serving.

### 3.2 Why Druid

Ad analytics is *the* canonical Druid use case; it's why Yahoo built Druid (originally for ad targeting analytics). Characteristics that map:
- Append-only events.
- Time-bucketed.
- Heavy filter by advertiser/campaign.
- Group-by dimension combinations.
- Counts and sums.

### 3.3 Schema

```yaml
dataSource: ad_events
dimensions:
  - advertiser_id
  - campaign_id
  - creative_id
  - ad_group_id
  - geo_country
  - device_type
  - placement
  - event_type      # impression / click / conversion
metrics:
  - count (count)
  - spend_sum (doubleSum on bid_price for impressions)
  - revenue_sum (doubleSum on revenue for conversions)
  - latency_p99 (quantile sketch on serving_latency_ms)
queryGranularity: minute
segmentGranularity: hour
```

### 3.4 The advertiser-isolation problem

Each advertiser sees only their own data. Approaches:

**Filter at query time**:
```sql
WHERE advertiser_id = ?
```

Cheap; relies on app-layer auth to enforce. Druid doesn't have native row-level security strong enough for cross-tenant isolation.

**Per-advertiser datasource (table)**:
- Strongest isolation; per-tenant retention/compaction.
- Operationally heavy at 100K advertisers (datasource explosion).
- Used for top-tier large advertisers; shared datasource for the long tail.

**Druid Router with tenant-based broker selection**:
- Different broker pools per tenant tier.
- Mega-advertiser gets dedicated broker tier; smaller advertisers share.

### 3.5 Sketches at scale

For "unique users reached":
- Exact distinct on `user_id` over a billion impressions: ridiculously slow + heavy.
- **Theta sketch**: probabilistic distinct + supports set operations (union, intersection).
  - "Users who saw creative A AND creative B" = sketch intersect.
  - "Users in campaign A who clicked campaign B" = intersect.
  - ~1-2% error; KB per group.

This is unique to sketch-aware engines. In ClickHouse, similar via `uniqHLL` etc.

### 3.6 Backfill and reprocessing

Ad attribution sometimes corrects retroactively (purchase attributed to a click 7 days ago). Druid's append-only model means:

```
Original segment (immutable):
  event_time=12:00, advertiser=X, count=1

New attribution event arrives at 14:00 referring to 12:00:
  event_time=12:00, advertiser=X, attribution_value=$50

Both rows exist in their respective time chunks.
Queries SUM(attribution) over the 12:00 hour see the late attribution.
```

For corrections that *replace* (rare): re-ingest the affected time chunk via batch ingestion, or use **MSQ (Multi-Stage Query Engine)** for SQL-based reindex.

### 3.7 Trade-offs

| Approach | Trade |
|---|---|
| **Druid + theta sketches** | Real-time + unique counts |
| **ClickHouse** | More flexible SQL, less rigid time partitioning |
| **BigQuery** | Cheap at low query volume; expensive at advertiser-dashboard QPS |
| **Custom (e.g., Manhattan + summation)** | LinkedIn-style; engineering cost |

### 3.8 What I'd actually do

For a major ad platform:
- **Druid for real-time advertiser dashboards** (last 30 days).
- **Theta sketches** for unique reach + set operations.
- **Per-tier broker pools**: top 100 advertisers each on dedicated tiers, rest shared.
- **S3 + Iceberg archive** for long-term + ML training feature data.
- **MSQ for backfill / reattribution** windows.

---

## 4. Scenario 3 — Application Performance Monitoring (APM)

### 4.1 The problem

Internal observability platform. Services emit metrics + traces:
- 10K services × 100 metrics each × 1 min granularity = 1M data points/sec aggregate.
- Plus ad-hoc traces for slow requests.

Engineers want:
- Sub-second dashboard refresh.
- Slice by service, host, version, region, custom tags.
- Time-shifted queries: "compare this hour to same hour last week."
- Alerts on metric anomalies.

### 4.2 Why Druid often fits APM

- Time-series ✓
- Multi-dimensional ✓
- Real-time ✓
- High-cardinality (10K hosts, 100K services * versions) — handled with care.

### 4.3 The cardinality challenge

A naive APM schema with `host_id` as a dimension:
- 10K hosts × group-by → 10K rows/group → fine.
- But 100K hosts × week-long retention × multiple metrics → segments grow large.
- Hot dimensions inflate index size.

Mitigations:
- **Time-bucket size**: smaller segments (every minute, not hour) for high-cardinality, but more segments → more overhead.
- **Hash dimensions** if exact value not needed (`hash(host_id) % 1024`).
- **Sketches** for unique-host counts.
- **Separate datasources** for high-cardinality vs low-cardinality data.

### 4.4 Hot tier sizing

```
1M data points/sec × 60s × 60m × 24h × 7d = 600B rows over 7 days.
With 50× rollup (minute granularity): 12B rows.
At 50 bytes/row compressed: 600 GB hot data.

Historical fleet: ~10 nodes × 64 GB SSD = serves 600 GB easily.
```

### 4.5 vs Prometheus / Mimir / VictoriaMetrics

```
Prometheus:
  - Single-server (scrape model).
  - Pull-based; not real-time.
  - Limited historical retention.
  - Rich PromQL.

Cortex / Mimir / VictoriaMetrics:
  - Horizontally-scalable Prometheus-compat.
  - Multi-tenant.
  - Long retention.
  - PromQL.

Druid:
  - General OLAP; not just metrics.
  - Real-time event ingestion.
  - SQL-based; less rich for metrics-specific operations.
  - Better for high-cardinality where metrics-DBs struggle.

For pure infra metrics: Mimir/VictoriaMetrics often better.
For metrics + custom events + business analytics in one engine: Druid.
```

### 4.6 What I'd actually do

For APM at MAANG:
- **Pure infra metrics (CPU, mem, latency)**: Prometheus/Mimir/VictoriaMetrics.
- **Custom application events** (transactions, business metrics): Druid.
- **Distributed traces**: dedicated trace store (Jaeger, Tempo); Druid for trace aggregates.

Combining: traces emit metric-events to Druid for aggregate analysis; raw traces in Tempo.

---

## 5. Scenario 4 — User Behavior Analytics & Funnels

### 5.1 The problem

Product analytics: what do users do, in what order, where do they drop off? Cohorts, funnels, retention.

### 5.2 The schema

```yaml
dataSource: user_events
dimensions:
  - user_id            # high cardinality
  - event_name         # signup, login, view, purchase, etc.
  - cohort             # signup_week or similar
  - country
  - device
  - app_version
metrics:
  - count (count)
  - revenue (doubleSum)
  - unique_users_hll (HLLSketch on user_id)
queryGranularity: minute
segmentGranularity: day
```

### 5.3 Funnel queries

Simple step funnel: "How many users did A, then B, then C in this period?"

In Druid, funnels with strict ordering between events (e.g., A *before* B *before* C per user) require:
- **Theta sketch intersection**: users in (A∩B∩C) — but loses ordering.
- **Druid Theta sketch with timestamps** (advanced): retain time per sketch; order-preserving intersect.
- **Apache Druid funnel extension** (community/contrib).
- **Offline funnel computation** in Spark; load results to Druid as a daily metric.

Druid's funnel support has matured but is not as native as in product-analytics platforms (Amplitude, Mixpanel, Heap). For complex funnels, often the right answer is:
- Druid for aggregate behavioral metrics.
- Daily Spark/Flink job for per-user funnel/retention; load results back to Druid.

### 5.4 Retention cohorts

"Of users who signed up in week N, how many returned in week N+1, N+2, ..."

Approach 1: HLL sketches per (cohort, week_of_activity).
- Cohort = signup week.
- For each (cohort_week, activity_week): HLL of unique users.
- Retention = |HLL(cohort, week_N+k) ∩ HLL(cohort_signup_week)|.

Approach 2: Pre-compute in Spark; load weekly cohort tables to Druid for fast queries.

### 5.5 Trade-offs

| Approach | Trade |
|---|---|
| **Druid + theta sketches** | Fast aggregates; limited ordering |
| **Amplitude / Mixpanel / Heap** | Best UX for product analytics; expensive |
| **Spark precompute + Druid serve** | Powerful + fast; pipeline ops |
| **ClickHouse** | Native SQL aggregates with windows; less rigid |

### 5.6 What I'd actually do

For product analytics at MAANG scale:
- Druid for aggregate metrics, distinct counts (sketches).
- Daily Spark job for per-user funnels/retention; result tables loaded to Druid.
- Consumer-facing product analytics tools (Mixpanel) for self-serve PMs if budget allows.

---

## 6. Scenario 5 — Real-Time KPI Dashboards

### 6.1 The problem

Internal Grafana/Superset dashboard refreshing every 5 seconds. KPIs:
- Orders/sec (current, 5-min trailing).
- Revenue/min (current vs previous day same time).
- Top-selling products (last hour).
- Geo distribution.

### 6.2 Druid's sweet spot

Real-time dashboard = many small, fast queries. Druid is built for this. Each panel's query is:

```sql
SELECT TIME_FLOOR(__time, 'PT1M') as minute, SUM(count) as orders
FROM orders
WHERE __time >= CURRENT_TIMESTAMP - INTERVAL '5' MINUTE
GROUP BY 1 ORDER BY 1;
```

Sub-100 ms response on a properly-sized cluster.

### 6.3 Caching layer

Druid has built-in result cache and segment-level cache:
- **Result cache**: query result by query hash. Effective for static dashboards.
- **Segment cache**: cached on Historicals locally.
- **Broker cache**: per-broker result cache.

With proper cache config: same dashboard query repeated → microsecond response.

### 6.4 Concurrency

Many users hit the same dashboard simultaneously. Broker handles 100s–1000s QPS with caches.

For massive concurrency (10K+ QPS dashboards):
- Multiple broker replicas behind a load balancer.
- Query result cache hit ratio > 90% (key for cost).
- Pre-aggregate further with materialized rollups.

### 6.5 Variables/templates

Grafana / Superset use template variables (`$service`, `$env`). Druid's SQL handles parameterized queries fine; ensure dashboards use bound queries (Broker cache hit relies on identical query).

### 6.6 Alternatives

| Approach | Trade |
|---|---|
| **Druid** | Real-time, low latency, ops cost |
| **ClickHouse** | Same role; sometimes simpler ops |
| **Pinot** | Similar; LinkedIn-flavor |
| **Materialized views in Postgres** | Doesn't scale to TB |
| **Cube.js + warehouse** | Caching layer; latency higher |

### 6.7 What I'd actually do

For internal dashboards at MAANG scale:
- Druid as the engine.
- Superset / Metabase / Grafana as the UI.
- Broker cache + result cache enabled.
- Pre-rollup at minute granularity.
- Auto-scale broker tier on QPS.

---

## 7. Scenario 6 — Multi-Tenant SaaS Analytics

### 7.1 The problem

SaaS analytics product: 1000s of customers each with their own data. Each customer:
- Sees their own dashboards.
- Cannot see others' data.
- Different volumes (some 10K events/day, some 100M).
- Different SLAs.

### 7.2 Multi-tenancy patterns

**Pattern A: Single shared datasource, filter at query time**
```sql
SELECT ... FROM events WHERE tenant_id = ? AND ...;
```

- Cheapest.
- Relies on app-layer auth.
- Long-tail tenants share resources well.
- Mega-tenants drag down others.

**Pattern B: Datasource per tenant**
- Strongest isolation.
- Per-tenant retention, compaction, capacity.
- Operationally heavy at 1000s of tenants.

**Pattern C: Tiered (shared + dedicated)**
- Top 1% of customers: dedicated datasource.
- Long tail: shared datasource.
- Best of both.

### 7.3 Per-tenant query throttling

Druid's broker has a thread pool. One tenant's heavy GROUP BY can starve others.

```yaml
# broker.runtime.properties
druid.broker.http.numConnections: 200
druid.processing.numThreads: 8
```

Tune for concurrency over individual query speed.

For per-tenant fairness: Druid Router with rules — route premium tenant queries to a dedicated broker pool.

### 7.4 Per-tenant retention

```yaml
# Coordinator rule: drop segments older than 90 days for free-tier
loadByInterval (last 30 days, free-tier) → tier = "shared"
loadByInterval (older than 30 days, free-tier) → drop
loadByInterval (last 365 days, premium-tier) → tier = "dedicated"
```

Per-datasource rules express retention. For shared datasource: harder; need TTL-style application logic, or store in a separate datasource per tier.

### 7.5 Trade-offs

| Approach | Trade |
|---|---|
| **Single datasource + filter** | Cheapest; weak isolation |
| **Per-tenant datasource** | Strong isolation; ops |
| **Tiered (shared + dedicated)** | Best balance |
| **Self-hosted per customer (Druid as app)** | Strongest; not realistic at scale |

### 7.6 What I'd actually do

For B2B analytics SaaS:
- **Tiered datasources**: shared for free/standard, dedicated for enterprise.
- **Router + dedicated brokers** for premium tier.
- **Per-tenant retention rules**.
- **App-level rate limiting** before queries hit Druid.

---

## 8. Scenario 7 — Network Telemetry / Flow Analytics

### 8.1 The problem

Network operator (telco, CDN, large enterprise). NetFlow / sFlow / VPC flow logs: src_ip, dst_ip, port, bytes, packets, ASN, region. Volumes: 10M flows/sec.

### 8.2 The cardinality nightmare

```
src_ip: 4 billion possible.
dst_ip: 4 billion.
(src_ip, dst_ip): up to 10^18 distinct pairs.

If treated as dimensions: segments explode; rollup gives ~zero benefit.
```

### 8.3 The mitigations

```
1. Aggregate at the ingestion layer:
   - Per-(src_subnet, dst_subnet) instead of full IPs.
   - Per-ASN / per-region, drop individual IPs.

2. Use sketches for cardinality:
   - HLL for unique src_ips.
   - Quantile sketches for byte distribution.

3. Top-K dimensions only:
   - Track only top 1000 source IPs explicitly; "other" bucket for rest.

4. Time + dimension pruning:
   - 5-minute rollup (not minute) for noisy dims.

5. Separate datasources:
   - High-card flow data with sampling.
   - Low-card aggregates without.
```

### 8.4 Sampling

For volumes that can't be reduced any other way: probabilistic sampling at ingestion. Sample rate 1/100. Multiply on read. Loses precision for low-volume flows; OK for top-line metrics.

### 8.5 Trade-offs

| Approach | Trade |
|---|---|
| **Druid with sketches + rollup** | Best for aggregate analytics |
| **ClickHouse** | Better for detailed forensics; requires more storage |
| **Elastic** | Search-and-zoom UX; expensive at flow volume |
| **DPDK + custom store** | Telco-grade; massive investment |
| **S3 + Athena** | Cheapest; not real-time |

### 8.6 What I'd actually do

For network telemetry at hyperscale:
- **Edge aggregation** (FPGA / smart switch) reduces volume 10–100×.
- **Druid with sketches** for real-time dashboards.
- **ClickHouse / S3** for forensic-detail queries.
- Pre-compute "top-N per minute" aggregates daily.

---

## 9. Scenario 8 — Anomaly / Fraud Detection

### 9.1 The problem

Detect unusual patterns: sudden spike in transaction volume from a country, drop in success rate, unusual user behavior. Real-time + ad-hoc forensics.

### 9.2 Druid's role

Druid provides the *queryable substrate*. Detection logic is external:
- Streaming anomaly detector reads Druid (or upstream stream) periodically.
- Triggers alerts.
- Forensic dashboards in Druid show context around alert.

```
Kafka → Druid (real-time analytics)
       └─► Stream processor (Flink) → anomaly model → alerts → Slack/PagerDuty
                                                     ↓
                                                 Forensic dashboard query against Druid
```

### 9.3 Time-shifted comparisons

"Today's hourly volume vs same hour last week":

```sql
SELECT
  TIME_FLOOR(__time, 'PT1H') as hour,
  COUNT(*) as today_count,
  (SELECT COUNT(*) FROM events WHERE __time = (TIME_FLOOR(__time, 'PT1H') - INTERVAL '7' DAY))
    as last_week_count
FROM events
WHERE __time >= CURRENT_TIMESTAMP - INTERVAL '24' HOUR
GROUP BY 1;
```

(Druid SQL supports this; subqueries allowed.)

### 9.4 Trade-offs

| Approach | Trade |
|---|---|
| **Druid + Flink** | Real-time + queryable forensics |
| **Splunk** | Real-time alerting + search; expensive |
| **Elastic + Watcher** | Search-shaped; eventual |
| **Custom ML pipeline** | Bespoke; engineering cost |

### 9.5 What I'd actually do

For real-time fraud / anomaly:
- Flink for streaming detection (windowed thresholds, ML inference).
- Druid for forensic dashboards engineers use post-alert.

---

## 10. Scenario 9 — A/B Test Analytics

### 10.1 The problem

Running 100s of concurrent A/B tests. Each test: which variant did users see, what they did. Need:
- Per-test conversion rate per variant.
- Per-test statistical significance (continuously updated).
- Cross-test interaction analysis.

### 10.2 The schema

```yaml
dataSource: experiments
dimensions:
  - experiment_id
  - variant
  - user_id
  - event_name
  - user_attrs (cohort, country, ...)
metrics:
  - count
  - conversion (count where event = "conversion")
  - revenue (doubleSum)
  - unique_users (HLL)
```

### 10.3 Per-test results

```sql
SELECT
  variant,
  COUNT(*) as exposures,
  SUM(CASE WHEN event = 'conversion' THEN count ELSE 0 END) as conversions,
  SUM(revenue) as revenue,
  APPROX_COUNT_DISTINCT_DS_HLL(user_id) as unique_users
FROM experiments
WHERE experiment_id = 'feature_x_v2'
  AND __time >= '2026-04-01'
GROUP BY variant;
```

Sub-second on Druid. Statistical significance computed downstream (Python with scipy or built-in if available).

### 10.4 Cross-test interactions

"Users in test A AND test B variant 2 vs control":
- Theta sketch intersection.
- Compute set sizes.
- Lift analysis.

### 10.5 Trade-offs

| Approach | Trade |
|---|---|
| **Druid** | Fast aggregates; sketch-friendly |
| **Postgres** | Limited scale |
| **Custom platform (Optimizely-style)** | Best UX; expensive |

### 10.6 What I'd actually do

For A/B test analytics at scale:
- Druid as the analytics engine.
- Daily Spark jobs for advanced analyses (heterogeneous treatment effects, propensity scores).
- Internal A/B test platform UI on top.

---

## 11. Scenario 10 — High-Cardinality with Sketches

### 11.1 The problem

"How many unique users did event X for any combination of dimensions?" Cardinality of user_id: 100M. Naive `COUNT(DISTINCT user_id) GROUP BY ...`: blows up.

### 11.2 The sketch family

| Sketch | Use case | Error | Storage |
|---|---|---|---|
| **HLL (HyperLogLog)** | Cardinality (unique count) | ~1-2% | ~KB per group |
| **Theta** | Cardinality + set ops (∪, ∩, diff) | ~1-2% | ~KB per group |
| **Quantiles** | p50, p95, p99 | configurable | ~KB |
| **Bloom** | Membership test | configurable false positive rate | bits |
| **Tuple** | Theta + per-element value | ~1-2% | per-element |

### 11.3 The pattern

```yaml
metrics:
  - name: unique_users
    type: thetaSketch
    fieldName: user_id
    size: 16384      # accuracy parameter

# Query:
SELECT THETA_SKETCH_ESTIMATE(unique_users) as users
FROM events
WHERE __time >= ... AND ...
```

### 11.4 Set operations

```sql
-- Users who saw page A AND page B
SELECT THETA_SKETCH_ESTIMATE(
  THETA_SKETCH_INTERSECT(
    FILTER(unique_users, page = 'A'),
    FILTER(unique_users, page = 'B')
  )
) as a_and_b
FROM events;
```

This is *the* killer feature. Exact distinct doesn't compose like this; sketches do.

### 11.5 Trade-offs

| Approach | Trade |
|---|---|
| **Druid sketches** | Fast, composable, approximate |
| **Postgres exact distinct** | Exact; slow |
| **ClickHouse uniqExact / uniqHLL** | Both options native |
| **Spark precompute** | Daily; loaded results into Druid |

### 11.6 What I'd actually do

For any cardinality-sensitive workload: sketches by default. Reach for exact only when ~1% error matters (financial, regulatory). Document sketch error in dashboard text.

---

## 12. Scenario 11 — Geographic / Spatial Analytics

### 12.1 The problem

Events with lat/lng or country/region. Need:
- "Active users heatmap by country."
- "Events from EU vs US."
- "Within 50km of city X."

### 12.2 Druid's geo

```
Druid is NOT a geo-spatial DB.
Lat/lng can be stored as dimensions but Druid doesn't compute haversine, geohash bbox, etc.

Workarounds:
  - Pre-compute geohash at ingest; use as dimension.
  - Pre-compute country/region/city at ingest (via lookup against MaxMind).
  - For "within X miles of point": filter by geohash prefix; refine downstream.
```

### 12.3 Lookups for enrichment

Druid lookups: small (KB-MB) reference data loaded into broker/historicals.

```yaml
lookup: country_lookup
namespace: country
type: cachedNamespace
extractionNamespace:
  type: jdbc
  ...
```

Use:
```sql
SELECT LOOKUP(geo_country, 'country_name'), COUNT(*)
FROM events GROUP BY 1;
```

For dimension enrichment without re-ingesting raw events.

### 12.4 Alternatives

| Approach | Trade |
|---|---|
| **Druid + geohash** | Good for grid-aggregate |
| **PostGIS** | Best for true geo; doesn't scale |
| **Elasticsearch geo** | Search + geo; expensive |
| **BigQuery geo** | Built-in; expensive at hot QPS |

### 12.5 What I'd actually do

For analytics with geo:
- Pre-bucket at ingest (geohash, country, city, region).
- Use Druid for aggregate dashboards.
- Use PostGIS if true geo computation matters.

---

## 13. Scenario 12 — Mobile App Event Analytics

### 13.1 The problem

Mobile SDK emits events: `app_open`, `screen_view`, `purchase`, custom events. 50M MAU. 1B events/day. Funnels, retention, custom events.

### 13.2 The schema

```yaml
dimensions:
  - app_id
  - device_id
  - user_id
  - event_name
  - screen
  - app_version
  - os_version
  - country
metrics:
  - count
  - revenue
  - unique_users (HLL)
queryGranularity: minute
segmentGranularity: hour
```

### 13.3 Late-arriving events

Mobile events often arrive late (offline, batched). Druid handles this:

```yaml
# Allow late arrivals up to 7 days
ingestion:
  lateMessageRejectionPeriod: P7D
```

Late events update the historical time chunks. Auto-compaction may merge them later.

### 13.4 Custom events

If the schema is "name + arbitrary properties," that doesn't fit Druid's typed-dimension model well. Common workarounds:
- **Top-K events**: dimension = event_name; limited known set.
- **Properties as JSON column**: limited query.
- **Per-event datasource for high-cardinality custom**.
- **Schema-on-read**: Iceberg + Athena for arbitrary custom events; Druid for top-K standard events.

### 13.5 Trade-offs

| Approach | Trade |
|---|---|
| **Druid with rolled-up standard events** | Fast dashboards; rigid schema |
| **Amplitude / Mixpanel** | Best UX for product analytics; expensive |
| **Iceberg + Athena** | Cheapest; not real-time |
| **Hybrid (Druid + Iceberg)** | Best of both; pipeline ops |

### 13.6 What I'd actually do

For mobile analytics at scale:
- Druid for standard event metrics + dashboards.
- Iceberg + Athena for arbitrary deep-dive queries.
- Spark daily job for retention cohorts loaded back to Druid.

---

## 14. Scenario 13 — Hybrid Batch + Streaming (Lambda/Kappa)

### 14.1 The problem

Real-time events stream in via Kafka. Daily batch ETL produces enriched / corrected / backfilled data.

### 14.2 Druid's approach

```
Streaming path:
  Kafka → Kafka indexing service → real-time segments → handed off after time chunk.

Batch path:
  S3 / HDFS / SQL → batch ingestion → batch segments REPLACE the streaming segments.
```

Druid supports overwriting time chunks: batch segments replace streaming for the same interval. Backfill / correction without app-level merge logic.

### 14.3 The reindex pattern

```yaml
# Reindex existing data with corrections / new schema:
type: index_parallel
inputSource: druid (read from existing)
transformSpec: ...    # apply enrichments
granularitySpec: ...
```

Multi-Stage Query (MSQ) since 2022 lets you do this via SQL:

```sql
INSERT INTO events_v2
SELECT __time, dimensions, ... FROM events
WHERE __time BETWEEN ... AND ...
PARTITIONED BY DAY
CLUSTERED BY user_id;
```

### 14.4 Late-arriving correction

Yesterday's data was wrong (forgot a dimension). Reingest yesterday's chunk:
- Stop streaming for that interval (or rely on lateMessageRejectionPeriod).
- Run batch ingestion from S3 with corrected data.
- Druid swaps segments atomically.

### 14.5 Trade-offs

| Approach | Trade |
|---|---|
| **Druid hybrid** | Native; segment-level swap |
| **Lambda architecture (Druid + Hive)** | Two engines; more ops |
| **Kappa (single stream)** | Simpler; harder to correct |

### 14.6 What I'd actually do

Use Druid's native hybrid: stream from Kafka, batch reingest from S3 daily for corrections / enrichments. MSQ for SQL-driven reindex.

---

## 15. Scenario 14 — Tiered Storage / Cold Data

### 15.1 The problem

7 days hot (queried often), 90 days warm (occasional queries), 7 years cold (audit/compliance, almost never queried).

### 15.2 Druid tiers

```yaml
# Coordinator config
tiers:
  hot:
    nodes: [historical-hot-1, historical-hot-2, ...]   # NVMe instances
    replicas: 2
  cold:
    nodes: [historical-cold-1, ...]                    # cheaper EBS / spinning disk
    replicas: 1

# Rules
loadByPeriod:
  P7D: hot, replicas=2
  P90D: cold, replicas=1
beyond: drop (segments removed from Historicals; remain in deep storage)
```

Recent data: replicated on fast disks. Older: single-replica on cheaper disks. Beyond retention: only in deep storage; queries fail unless restored.

### 15.3 The deep-storage-only tier

For data not loaded onto any Historical:
- Stored only in S3.
- Queries against this time range require the segment to be loaded first (slow).
- "Cold but recoverable" mode.

For 7-year retention compliance: usually drop from Historicals at 1 year; rely on deep storage + reindex if needed.

### 15.4 Cost shape

```
Hot tier: i3.4xlarge with NVMe — ~$1/hr per node.
Cold tier: r5.2xlarge with EBS — ~$0.50/hr per node.
Deep storage: S3 ~$0.023/GB-month.

For 100 TB hot + 1 PB warm + 10 PB cold:
  Hot: 100 TB / 4 TB/node × $1/hr × 720 = $18K/mo (NVMe-bound).
  Warm: 1 PB / 8 TB/node × $0.50/hr × 720 = $45K/mo.
  Cold: 10 PB × $0.023 = $230K/mo (S3).
  Total: ~$293K/mo (rough).
```

Storage tiering is the biggest cost lever. Done right: 5–10× saving vs all-hot.

### 15.5 Trade-offs

| Approach | Trade |
|---|---|
| **Druid native tiering** | Operationally clean |
| **Druid + S3 archive (Iceberg)** | Cheaper for cold; queries don't go to Druid |
| **Sequential Druid clusters** | Niche; complex |

### 15.6 What I'd actually do

Hot 7 days on Druid NVMe. Warm 30-90 on Druid EBS. Cold archive to Iceberg + Athena for ad-hoc historical queries. Beyond 1 year: only Iceberg.

---

## 16. Scenario 15 — Compaction Strategy

### 16.1 The problem

Streaming ingestion creates many small segments per time chunk (one per task). Over time, a 1-hour chunk may have 100 segments. Query performance degrades.

### 16.2 Auto-compaction

```yaml
# Coordinator auto-compaction config per datasource
dataSource: events
compaction:
  inputSegmentSizeBytes: 800000000      # target 800 MB per segment
  taskPriority: 25
  taskCount: 5
  granularity: 
    segmentGranularity: hour
  tuningConfig:
    maxRowsPerSegment: 5000000
```

Druid's coordinator submits compaction tasks: read segments in a chunk → write fewer, larger segments.

### 16.3 What compaction does

- Combines small segments into larger.
- Re-rolls up at higher granularity (e.g., minute → hour) — saves storage + speeds queries.
- Re-sorts on better dimensions.
- Replaces fragmented segment manifest with clean one.

### 16.4 Trade-offs

```
Aggressive compaction (re-roll to higher granularity):
  + Storage saving (10×).
  + Query speed-up.
  - Lose fine-grained query (can't query at minute level).

Conservative compaction (just merge small segments):
  + Preserve queryable granularity.
  - Less storage saving.
```

Common pattern: progressive compaction.
- Hour-old segments: merge small to big at minute granularity.
- Day-old: re-roll to 5-minute granularity.
- Week-old: re-roll to hourly.
- Month-old: re-roll to daily.

### 16.5 What I'd actually do

Auto-compaction with multi-stage rules:
- Recent (1 day): merge to 1 GB segments, keep minute granularity.
- 1-7 days: re-roll to 5-minute.
- 7-30 days: re-roll to hour.
- 30+ days: re-roll to day, archive to deep-storage-only.

---

## 17. Scenario 16 — Lookups for Dimension Enrichment

### 17.1 The problem

Events have `country_code` (US, UK). UI wants country name (United States, United Kingdom). Don't want to repeat strings in every event.

### 17.2 Druid lookups

```yaml
type: cachedNamespace
namespace: country
extractionNamespace:
  type: uri  # or jdbc, kafka, etc.
  uri: s3://config/countries.json
  ...
```

JSON file: `{"US": "United States", "UK": "United Kingdom", ...}`. Loaded into broker memory.

Query:
```sql
SELECT LOOKUP(country_code, 'country'), COUNT(*) FROM events GROUP BY 1;
```

### 17.3 Use cases

- Country code → name.
- User ID → user attributes (small dataset).
- Product ID → product details.
- Test ID → test name.

### 17.4 Limits

```
Druid lookups: KB to ~10s of MB.
Loaded entirely in memory on every broker/historical.
Updated periodically from source.

For larger reference data: pre-join at ingest (the "wide row" pattern) — denormalize.
```

For very large lookups (millions of rows): better to denormalize at ingest. Lookups are for ~thousands-of-rows reference data.

### 17.5 Update lag

Lookups refresh on a configurable interval. Updates to source (e.g., new country added) lag the refresh. For real-time-fresh lookups: Kafka-based lookup namespace.

### 17.6 What I'd actually do

For small reference data (countries, currencies, plan tiers): JSON-based lookups refreshed daily.
For larger reference data: denormalize at ingest.

---

## 18. Scenario 17 — Backfill / Reindexing

### 18.1 The problem

Schema changed; need to backfill last 90 days with the new schema. Or: bug in ingestion missed events; need to re-ingest from S3 archive.

### 18.2 The MSQ approach

Multi-Stage Query Engine (MSQ) since 2022:

```sql
INSERT INTO events_v2
SELECT
  __time,
  user_id,
  page,
  -- new column derived from existing
  CASE WHEN page LIKE '%product%' THEN 'product_page' ELSE 'other' END as page_category,
  ...
FROM events_v1
WHERE __time BETWEEN '2026-01-01' AND '2026-04-01'
PARTITIONED BY DAY
CLUSTERED BY user_id;
```

This runs as a distributed job on Druid's MSQ engine. Can read from existing datasources, S3, Kafka.

### 18.3 Replace vs append

```sql
-- REPLACE (overwrite) for the time window
REPLACE INTO events
OVERWRITE WHERE __time >= '2026-01-01' AND __time < '2026-04-01'
SELECT ... FROM source;
```

### 18.4 Backfill from S3

Common case: events archived in S3 Parquet; reload into Druid.

```sql
INSERT INTO events
SELECT * FROM TABLE(
  EXTERN(
    '{"type":"s3","uris":["s3://archive/2026/04/*"]}',
    '{"type":"parquet"}'
  )
)
PARTITIONED BY DAY;
```

### 18.5 Considerations

- MSQ jobs use cluster resources; co-exists with online queries (sometimes contended).
- Long-running (hours for big backfills).
- Memory pressure on MSQ workers.

### 18.6 What I'd actually do

Use MSQ for any reindex / backfill larger than a few hours of data. Run during low-traffic windows. Monitor cluster resources.

---

## 19. Scenario 18 — SQL Access for BI Tools

### 19.1 The problem

Connect Tableau, Looker, Superset to Druid for self-serve analysts.

### 19.2 Druid SQL

```
Druid SQL: based on Apache Calcite.
Endpoint: HTTP / Avatica JDBC / native.
Most SELECT, GROUP BY, ORDER BY, LIMIT supported.
Joins: limited (lookup-style; broadcast small tables).
Subqueries: supported with caveats.
Window functions: supported in newer versions.
```

### 19.3 BI tool integration

Most modern BI tools support Druid via JDBC (Avatica). Some have native Druid drivers. For Tableau / Looker: install JDBC driver; configure connection.

### 19.4 What works well

- Aggregations, group by, filter by time + dimension.
- Top-N queries.
- Sketch-based unique counts.
- Time-floor functions.

### 19.5 What works poorly

- Multi-table joins (limited).
- CROSS JOIN.
- Heavy analytical functions (window functions are slow).
- COUNT(DISTINCT) without a sketch.

### 19.6 The "BI on Druid" anti-pattern

Analysts try to write Postgres-style SQL with joins. It runs slowly or fails.

Mitigations:
- Train analysts on Druid SQL idioms.
- Provide pre-aggregated marts (rollup datasources).
- Materialize joins at ingest (denormalize).
- For complex SQL: BigQuery/Snowflake on Iceberg/S3, not Druid.

### 19.7 What I'd actually do

For BI:
- Druid for dashboard queries (pre-defined, fast).
- Snowflake / BigQuery for ad-hoc analyst SQL.
- Iceberg/S3 as the source-of-truth shared between them.

---

## 20. Performance and Scaling Deep Dive

### 20.1 Sizing the Historical tier

```
Working dataset = N TB.
Per-node SSD/NVMe: 4 TB.
Replicas: 2.
Required nodes: N TB × 2 / 4 TB = N/2 nodes.

Each Historical: 64–256 GB RAM (segment caches).
CPU: 16–32 cores for query processing.
```

For 100 TB hot data: 50 i3.4xlarge nodes (4 TB NVMe each) × 2 replicas = $$.

### 20.2 Sizing the Broker

```
Brokers handle query planning + result merging.
1 broker can handle ~100s QPS for typical dashboard queries.
For 10K QPS dashboards: 10–20 brokers behind LB.

Brokers are stateless (caching only); easy to scale horizontally.
```

### 20.3 Sizing ingestion

```
Per-Peon (task): ~10K-100K events/sec depending on dimensions.
Tasks per topic partition: 1-2 (Kafka indexing).
Total: tasks = topic partitions × replication factor.

For 1M events/sec: ~10-100 partitions, similar count of tasks.
MiddleManagers: 5-20 nodes hosting peons.
```

### 20.4 ZooKeeper sizing

ZooKeeper is the backbone for cluster state. 3- or 5-node ensemble. Don't co-locate with anything heavy. Watch for excessive watches (a Druid issue at scale).

### 20.5 Metadata DB

Postgres / MySQL holds segment manifest. For PB-scale Druid: this DB has 100Ks of segment rows; needs careful tuning. Indices on `interval`, `version`, `used`. Don't run on a single AZ.

### 20.6 Query latency tuning

```
Slow queries: usually because:
  - Large time range (10s of TB scanned).
  - High cardinality group-by.
  - Missing indexes (Druid auto-indexes, but rare misses).
  - Slow Historical disk.
  - Network saturation between broker and Historical.

Tools:
  - Druid query metrics (per-stage breakdown).
  - "/druid/v2/sql/explain" for query plan.
  - Cluster state on Coordinator console.
```

### 20.7 Concurrent queries

```
Brokers have a thread pool: druid.processing.numThreads.
Single big query can saturate.
For multi-tenant: tier brokers, throttle per-tenant.
```

---

## 21. Cost Engineering

### 21.1 The cost shape

```
Historical tier (EC2 + storage): 60-70% of total
Ingestion (MiddleManager EC2): 10-20%
Broker / Router (EC2): 5-10%
Coordinator / Overlord (small): negligible
Deep storage (S3): negligible
Metadata DB (Postgres): negligible
ZooKeeper (3-5 nodes): minor
Network: cross-AZ between Historicals and Broker
```

The Historical tier dominates. NVMe instances (i3, i4i) are the workhorse and the cost driver.

### 21.2 Optimization levers

```
1. Tiered storage: cold to cheaper instances, deep storage drop.
2. Aggressive rollup (sacrifice granularity for storage).
3. Sketches instead of exact distinct.
4. Compaction (smaller, fewer segments).
5. Broker query result cache (high hit rate = fewer Historical hits).
6. Right-sizing replicas (1 for cold, 2 for hot).
7. Spot instances for batch ingestion / MSQ workers.
8. VPC endpoints to avoid NAT Gateway egress.
```

### 21.3 The "all NVMe" trap

Tempting to put everything on NVMe (i3.16xlarge for fastness). But:
- 90% of data queried < 1× per day.
- NVMe is overkill for cold data.

Tier early; tier aggressively.

### 21.4 Cost vs ClickHouse / Pinot

```
Druid: most operationally complex; richest ecosystem of integrations.
ClickHouse: simpler ops; comparable or better single-node performance.
Pinot: similar to Druid; better real-time + user-facing analytics.
```

For pure cost-per-event-ingested-and-queried: ClickHouse often wins on raw efficiency. For mature ecosystem + real-time + fault tolerance: Druid still has its niche.

### 21.5 What I'd actually do

For Druid cost optimization:
- Quarterly review of tier rules.
- Auto-compaction with progressive rollup.
- Drop datasources / partitions never queried.
- Sketches everywhere reasonable.
- Spot instances for MSQ.

---

## 22. Operational Lifecycle

### 22.1 Day-1 setup

- 3-AZ deployment (Coordinator, Overlord, Broker, Historical, MM in each AZ).
- ZooKeeper 3-node ensemble (different AZ).
- Metadata DB on RDS Multi-AZ.
- Deep storage in S3, encrypted.
- Monitoring: every Druid metric to CloudWatch / Prometheus.

### 22.2 Upgrades

```
Druid is 6+ processes; rolling upgrades require care:
  - Coordinator/Overlord: leader-elected; can rolling-upgrade.
  - Broker: stateless, rolling.
  - Historical: drain segments before stop; segments rebalanced.
  - MM: drain tasks; restart.
  - ZK / Metadata DB: separate upgrade procedures.

Test upgrades in staging; production upgrades planned monthly.
```

### 22.3 Common operational issues

- **ZK overload**: too many watches / high churn. Tune.
- **Coordinator slowness**: Metadata DB slow → segment loads delayed.
- **Historical out-of-disk**: forgot to enforce drop rules.
- **MM tasks fail**: ingestion pauses; backlog grows in Kafka.
- **Broker OOM**: heavy GROUP BY blew memory.
- **Cross-AZ data transfer**: huge cost surprise; shore up affinity.

### 22.4 Backup / restore

```
Deep storage = the backup. All segments durably in S3.
Metadata DB needs separate backup (RDS snapshots).
ZooKeeper state can be reconstructed from metadata.

Restore drill: nuke a Historical, watch Coordinator rebalance.
                Nuke entire cluster, restore from deep storage + metadata.
                Test once per quarter at minimum.
```

### 22.5 Multi-region

Druid clusters are typically per-region. For DR:
- Replicate Kafka cross-region.
- Druid in each region ingests independently.
- Cross-region query: app-level federation.

There's no native "Druid global tables" equivalent.

---

## 23. Anti-Patterns — Staff-Level Red Flags

### 23.1 Using Druid for transactional data

Druid is append-only and time-partitioned. CRUD use cases don't fit.

### 23.2 Heavy joins between large datasources

Druid joins are limited. For multi-table SQL: BigQuery / Snowflake / Trino on Iceberg.

### 23.3 Treating Druid as a data warehouse

DW = arbitrary SQL, joins, ad-hoc. Druid = pre-defined OLAP queries. Match the tool.

### 23.4 No rollup on dashboard data

Ingesting raw events 1:1 → segments huge, queries slow. Rollup at minute or 5-minute typically gives 10× compression.

### 23.5 Exact distinct on high cardinality

`COUNT(DISTINCT user_id)` over a billion users: minutes, OOM. Use HLL/Theta sketch.

### 23.6 ZooKeeper colocated with heavy services

ZooKeeper under load → cluster fragility. Dedicate hardware.

### 23.7 No segment compaction

Many tiny segments → query plan explodes. Auto-compaction is mandatory.

### 23.8 Treating Druid like Elasticsearch

ES = search; Druid = aggregate. Don't full-text in Druid.

### 23.9 Schema-on-read without typed dimensions

Putting JSON blobs in Druid loses the index benefit. Define dimensions; pre-extract fields at ingest.

### 23.10 Overly complex GROUP BY in BI tools

Analysts writing 10-table JOINs: timeouts. Train analysts; pre-aggregate marts.

### 23.11 Single-AZ deployment

ZooKeeper or Coordinator in one AZ → AZ failure = cluster down. Always multi-AZ.

### 23.12 Not monitoring task success / lag

Kafka indexing tasks fall behind silently → ingestion lag. Always alarm.

### 23.13 Over-replication of cold data

`replicas=2` everywhere doubles cost. Cold tier: 1 replica is fine (deep storage is the backup).

### 23.14 Long retention without cold tier

7-year hot retention = bankruptcy. Tier or drop.

### 23.15 Mixing transactional + analytical workloads

OLTP source DB queried by BI tool → slowdown for app. Pipeline to Druid; query Druid.

### 23.16 Custom UDFs in Druid SQL

Druid SQL extensibility is limited. Expensive to add. Pre-compute at ingest.

### 23.17 Ignoring dimension cardinality

A new high-cardinality dim (e.g., URL fragment) → rollup ineffective; segments balloon. Vet new dimensions before adding.

### 23.18 Loading datasources nobody queries

"We loaded it for someday." Nobody queries; pays cluster resources. Audit and drop.

### 23.19 Using Druid for sub-100ms point lookups

Druid optimized for aggregates. For "get exact row by ID": use a KV store.

### 23.20 No restore drills

Same anti-pattern as everywhere else. Backups untested don't exist.

---

## 24. Decision Framework

### 24.1 Step 0 — Is Druid the right tool?

```
Yes if:
  - Append-only event data.
  - Time-bucketed.
  - Slice-and-dice over dimensions.
  - Real-time ingestion + sub-second dashboard queries.
  - Aggregate metrics (count, sum, sketches).

No if:
  - Need transactions.
  - Heavy joins / arbitrary SQL.
  - Sub-100ms point lookups.
  - Volume is small (use Postgres or hosted).
  - Team has no ops capacity for a complex distributed system.
```

### 24.2 Step 1 — Map workload to features

- Real-time? Kafka indexing service.
- Cardinality? HLL / Theta sketches.
- Multi-tenant? Tier brokers + per-tenant retention.
- Cold data? Tiered storage + deep-storage-only.
- BI access? Druid SQL via Avatica.

### 24.3 Step 2 — Schema design

- Time column granularity (minute typical).
- Dimensions (lower cardinality first; pre-bucket high-cardinality).
- Metrics (raw counts, sums, sketches).
- Rollup granularity.

### 24.4 Step 3 — Cluster sizing

- Working set / 4 TB × 2 = Historical NVMe nodes.
- Ingest rate / 50K → MM peon count.
- QPS / 100 → broker count.

### 24.5 Step 4 — Operational design

- Multi-AZ.
- Tiered storage rules.
- Auto-compaction.
- Monitoring + alarms.
- Backup + restore drill.

### 24.6 Step 5 — Cost model

- Historical-tier cost dominates.
- Tier early, tier aggressively.
- Sketches save 100×.

### 24.7 Step 6 — Validate against alternatives

- ClickHouse for simpler ops.
- Pinot for user-facing analytics.
- BigQuery for bursty / sporadic.
- Iceberg + Athena for cold + ad-hoc.

---

## 25. Mental Models a Staff Engineer Carries

1. **Druid is for slice-and-dice OLAP, not general analytics.** Match shape.

2. **Time is mandatory; everything is partitioned by it.** Bucket carefully.

3. **Rollup is the cheapest 10× lever.** Don't ingest raw events 1:1.

4. **Sketches over exact for cardinality.** ~1% error, 100× faster.

5. **Bitmap-indexed dimensions are the speedup.** Pick dimensions for filter+group.

6. **Segments are immutable.** Updates = re-ingest the time chunk.

7. **Cardinality kills.** Vet new dimensions; pre-bucket high-card.

8. **Tiered storage is operational hygiene, not optional.** Hot/warm/cold/drop.

9. **Auto-compaction is mandatory.** Without it, segments fragment.

10. **ZooKeeper is the heart.** Don't overload it; multi-AZ; monitor.

11. **Druid SQL is OLAP SQL, not Postgres SQL.** No big joins, no fancy windows.

12. **Multi-tenant via tier or filter.** Datasource-per-tenant doesn't scale to thousands.

13. **Multi-stage Query Engine (MSQ) for reindex / heavy SQL.** Not for hot path.

14. **Lookups for small reference data.** Denormalize for big.

15. **Brokers are stateless; scale horizontally.** Historicals are stateful; careful.

16. **NVMe matters.** Hot tier on i3/i4i; cold on cheaper.

17. **Kafka native ingestion is exactly-once-ish.** Druid stores Kafka offsets.

18. **Late-arriving events are normal.** Configure window; auto-compact.

19. **Real-time tasks hand off to deep storage.** Then queries hit Historicals.

20. **Cross-AZ network can dominate cost.** Affinity-aware deployment.

21. **Ops complexity is real.** Druid is 6+ processes + ZK + metadata DB.

22. **For pure metrics: Mimir/VictoriaMetrics often win.** Druid for events + metrics + business.

23. **For arbitrary SQL: BigQuery/Snowflake.** Druid is dashboards, not warehouses.

24. **For user-facing analytics: Pinot/ClickHouse comparable.** Defend choice deliberately.

25. **For batch-only: Iceberg + Athena cheaper.** Druid earns its keep on real-time.

26. **Quarterly cost review.** Tier rules drift; cardinality grows.

27. **Restore drills.** Reload from deep storage after metadata loss.

28. **Boring is a feature.** A 10M-event/sec Druid serving sub-second dashboards quietly is the goal.

---

## 26. Closing Notes

Druid earns a place in the modern stack when the workload is *clearly OLAP-shaped* — append-only events, time-partitioned, slice-and-dice over dimensions, real-time + batch ingestion, dashboards and alerts. In that niche, nothing matches it for sub-second p95 on TBs.

Outside that niche, Druid is wrong:
- Use Postgres / RDS for transactional.
- Use BigQuery / Snowflake for arbitrary SQL.
- Use ClickHouse for simpler ops at similar performance.
- Use Pinot for LinkedIn-style user-facing analytics.
- Use Athena/Iceberg for cheap cold queries.

The staff-level skill is recognizing the shape and defending the choice in design reviews. "We need Druid because dashboards are slow" is rarely the right answer; "we need Druid because at 10M events/sec, we need sub-second slice-and-dice with rollup and sketches, and ClickHouse's exactly-once Kafka story is weaker for our reliability targets" is.

The ecosystem around Druid (Kafka, Iceberg, Spark, Superset) is part of the reason to choose it. Done well, Druid is one of the most boring, reliable analytics platforms at hyperscale. Done poorly, it's an unstable cluster eating engineer-hours.

The art is in the schema design, the rollup discipline, the tier rules, the compaction strategy, and the operational drills. Master those and Druid is a tool you'll keep reaching for.

> Companion docs:
> - `awsS3ScenariosAtScale.md` — deep storage backing Druid.
> - `snsSqsEventBridgeAtScale.md` — event ingestion before Druid.
> - `dynamoDBScenariosAtScale.md` — when KV is the right tool.
> - `eventPlatformsAtScale.md` — Kafka/MSK ingestion.
> - `logProcessingAndAggregation.md` — the analytics pipeline overview.