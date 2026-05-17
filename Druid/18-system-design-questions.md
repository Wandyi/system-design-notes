# 18 · System Design Questions — Druid Edition

Twelve realistic design-round prompts. Same rubric as the ClickHouse pack: clarify → schema → tradeoffs → failure modes → alternatives.

## Rubric

1. Clarify scope, latency, retention, ingest rate, concurrency, tenancy.
2. Pick **rollup config** and **sketch metrics**.
3. Draft datasource definition + `PARTITIONED BY` + `CLUSTERED BY`.
4. Choose ingest path (Kafka supervisor / MSQ).
5. Replication + tiering.
6. Compaction + retention.
7. Observability + alerts.
8. Failure modes + recovery.

---

## Q1 · Product analytics for a SaaS (PostHog/Mixpanel-class)

**Clarifications**:
- 1B events/day across 10K tenants.
- Latency target: < 1s for dashboards.
- Retention: 1 year hot, 5 years cold.

**Schema**: see [13.2](13-schema-design-patterns.md#132-pattern-1--event-tracking-product-analytics-the-canonical).

**Ingest**: Kafka supervisor, `taskCount=20, replicas=2, taskDuration=PT15M`.

**Tiers**:
- `hot`: 7 days, 2 replicas, NVMe Historicals.
- `warm`: 90 days, 1 replica, SSD.
- `cold`: 5 years, 1 replica, large HDD.
- Beyond: dropForever.

**Compaction**: target 700MB segments daily, re-rollup MINUTE → HOUR after 30 days.

**Multi-tenancy**: tenant_id leading dimension; row-level filter at the auth layer enforces `WHERE tenant_id = ?`. For top-10 tenants: dedicated datasources.

**Failure modes**:
- Supervisor lag → scale taskCount; pause heavy queries.
- Hot tier full → demote oldest segments to warm; add Historicals.
- Bad data in last hour → MSQ REPLACE the affected window.

---

## Q2 · Real-time ad analytics

**Clarifications**:
- 100B impressions/day; 1B clicks; 100M conversions.
- Per-campaign, per-creative reporting; cohort analysis (users who saw ad and bought).
- Latency: < 30s end-to-end ingest → query.

**Schema**: three datasources (impressions, clicks, conversions) all with theta sketch of `user_hash`.

**Cohort query**: theta-sketch intersection (see [14.7](14-query-patterns-and-corner-cases.md#147-pattern-set-algebra--a-but-not-b)).

**Tiers**: hot/warm/cold by age; 13-month retention for billing.

**Failure mode**: late conversions (T+24h) → ingest into the right time interval; segment-level overshadowing handles it.

---

## Q3 · User-facing embedded analytics at high QPS

**Clarifications**:
- 5K tenants; 10K dashboard requests/s peak.
- p99 < 200ms per tile.
- Sub-minute data freshness.

**Schema**: pre-rolled-up datasources for each common dashboard query shape.

**Acceleration**:
- Heavy use of HLL sketches for distinct counts.
- Pre-computed daily/hourly rollups in chained datasources.
- Druid Broker caching enabled.
- Imply Pivot or equivalent BI layer with its own caching.

**Concurrency**: lots of Brokers behind a load balancer; Historicals tiered.

**Cost levers**: aggressive rollup, compress everything via sketches.

---

## Q4 · Observability backend (logs + metrics + traces)

Three datasources: `logs`, `metrics`, `traces`.

- **Logs**: `rollup: false`, day-partitioned, bitmap on service+severity+trace_id, token bloom for body.
- **Metrics**: rollup with quantile sketches, hour-partitioned.
- **Traces**: by trace_id with bitmap lookup; flame-graph data per span.

Cross-datasource queries via Broker join (small joins) or via separate dashboards.

Cold tier for old data; auto-kill at 90 days.

---

## Q5 · Replace ELK for log search

Approach:
- Phase 1: dual-write logs to ES + Druid.
- Phase 2: Compare query results / cost over 30 days. Druid wins on agg, ELK wins on search relevance.
- Phase 3: Move analytics queries to Druid; keep ES for ad-hoc text search.

**Expected savings**: 5-10× storage; 3-5× compute. Trade-off: lose Elastic's ranked search relevance.

---

## Q6 · MSQ-based ETL pipeline

Scenario: every night, aggregate yesterday's raw events into a daily summary datasource.

```sql
REPLACE INTO daily_summary
  OVERWRITE WHERE __time = TIMESTAMP '2026-05-17'
SELECT TIME_FLOOR(__time, 'P1D') AS __time,
       tenant_id, event_name,
       COUNT(*) AS events,
       APPROX_COUNT_DISTINCT_DS_HLL(user_id) AS users,
       SUM(revenue) AS revenue
FROM raw_events
WHERE __time = TIMESTAMP '2026-05-17'
GROUP BY 1, 2, 3
PARTITIONED BY DAY CLUSTERED BY tenant_id;
```

Schedule via Airflow / cron. Multiple-day re-runs in parallel (one MSQ task per day).

---

## Q7 · Disaster recovery design

**Targets**: RPO < 5 min; RTO < 30 min.

**Architecture**:
- Primary cluster in region A, full Druid stack.
- DR: deep storage cross-region replicated (S3 CRR); metadata DB physically replicated (Postgres streaming replication); ZK is local to each cluster.
- On failover: spin up DR cluster, point at replicated metadata DB + deep storage. Historicals download segments. App switches.

**Practice quarterly.**

---

## Q8 · Migration from Druid to ClickHouse (or vice versa)

A surprisingly common interview scenario.

**Druid → ClickHouse**:
1. Dual-ingest: same Kafka topic to both.
2. Replay Druid native batch / MSQ to backfill historical data into CH.
3. Translate Druid SQL queries to CH SQL (mostly mechanical).
4. Per-dashboard cutover.

**ClickHouse → Druid**:
1. Build Druid schema (pick rollup, sketches).
2. Bulk-export CH data to S3 as Parquet; ingest into Druid via MSQ.
3. Dual-ingest going forward.
4. Per-dashboard cutover.

The non-trivial part: equivalences for engine-specific features (argMax ↔ LATEST_BY; uniqCombined ↔ APPROX_COUNT_DISTINCT_DS_HLL; AggregatingMergeTree MV ↔ rollup datasource; etc.).

---

## Q9 · Multi-stage funnel with cohort intersection

Question: "Show me users who completed a 3-step funnel within 1 week, broken down by signup cohort."

Approach: theta sketches per (event_name, week). Intersect at query time.

For order-strict: re-process via MSQ to build a per-user funnel summary; query that.

---

## Q10 · Streaming exactly-once with re-processing capability

Setup: Kafka supervisor with replicas=2. Additionally, archive raw Kafka events to S3.

To re-process: stop supervisor, run MSQ REPLACE from the S3 archive for the affected interval.

This is the canonical "streaming + replay" architecture; Druid handles both natively.

---

## Q11 · Hot/cold storage with re-rollup on age

Streaming ingestion at MINUTE rollup → hot tier for 7 days.

Auto-compaction at 7d → re-rollup to HOUR; move segments to warm tier.

Auto-compaction at 90d → re-rollup to DAY; move to cold tier.

At 5 years → dropForever.

Result: storage cost decreases dramatically with age; precision decreases gracefully.

---

## Q12 · Realtime alerting on streaming events

Druid isn't a streaming alerting platform (Flink/KSQL is). But you can:

- Ingest events via supervisor.
- Run Druid SQL on a tight schedule (every 10s) via a cron / Imply Pivot scheduled query.
- If result exceeds threshold, fire an alert.

For sub-second alerts: use a real streaming engine; for minute-grain alerts on aggregates, Druid is fine.

---

## Cross-cutting interviewer asks

- "What changes for 10× scale?" — name the next bottleneck (Broker fan-out, Historical disk, Coordinator cycle time, metadata DB).
- "How do you cost this?" — Historicals (compute), deep storage, MiddleManagers (ingest CPU), metadata DB.
- "What's your observability?" — JMX metrics + sys schema + alerts.
- "How do you handle a big tenant?" — dedicated datasource or dedicated tier.
- "How do you fix bad data?" — MSQ REPLACE on the interval.
- "What if Coordinator/Overlord dies?" — standby is leader-elected; failover automatic.
- "What if ZK is down?" — no leader election, no segment announcement; queries on existing data still work; new ingest stalls.
- "What if metadata DB is down?" — no new ingest, no new task submission; existing segments still queryable.
