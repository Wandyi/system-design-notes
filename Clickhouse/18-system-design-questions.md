# 18 · System Design Questions — ClickHouse Edition

Twelve realistic design-round prompts. Same rubric as the other packs: clarify → schema sketch → tradeoffs → failure modes → alternatives.

## Rubric

1. **Clarify scope, scale, latency SLO, query patterns, ingest pattern, tenancy, retention.**
2. **Pick engine** and explain why over alternatives.
3. **Draft ORDER BY / PARTITION / codecs.**
4. **MVs / projections for repeated queries.**
5. **Replication + sharding choice.**
6. **Failure modes: ingest spike, mutation backlog, replica down, Keeper outage, hot tenant.**
7. **Cost: storage, compute, egress.**
8. **Migration / rollout if relevant.**

---

## Q1 · Observability backend (logs + metrics + traces)

**Clarifications**:
- 50K hosts × 100 services emitting logs and metrics; 10B log lines/day; 1B metric points/day; 100M spans/day.
- Multi-tenant (10 tenants); strict isolation.
- Query latency SLO: p95 < 1s for dashboards.
- Retention: 7 days hot, 30 days warm, 90 days cold.

**Schema**:
- Three tables (`logs`, `metrics`, `traces`) with OTel-like schemas (see [13](13-schema-design-patterns.md)).
- `ORDER BY (tenant_id, service, severity, ts)` for logs; `(tenant_id, metric, ts)` for metrics; `(tenant_id, trace_id, ts)` for traces.
- Bloom filter on `trace_id` across all three for cross-correlation queries.
- Token bloom filter on log `message`.

**Ingest**:
- OTel collectors → Kafka → ClickHouse Kafka engine → MV → tables.
- Batch every 5s or 100K rows, whichever first.

**Acceleration**:
- MV per-minute rollups for metrics (avg, min, max, p99 via tdigest, count).
- MV per-minute log-volume by `(service, severity)`.
- MV per-minute span counts and durations by `service` for trace dashboards.

**Storage tiers**:
- Hot disk (NVMe / Cloud file-cache) for 7 days.
- S3 cold for 7–90 days.
- TTL: `TO DISK 'cold'` at 7 days, `DELETE` at 90.

**Cluster**:
- 3 shards × 2 replicas + 3 Keepers (or Cloud equivalent).
- Per-tenant row policies; per-tenant quotas.

**Failure modes**:
- Ingest spike → batch via async_insert; backpressure to collectors.
- Hot tenant → dedicated shard or Cloud service for that tenant.
- Bloom-filter false positive on trace lookup → minor extra scan; acceptable.

**Cost knobs**:
- ZSTD(3) over LZ4 for log message → 1.5-2× storage savings.
- TTL columns aggressively.
- MV aggregates make dashboards read 1 MB instead of 100 GB.

**Alternative**:
- ELK at this scale: 5-10× more storage, harder ops.
- Honeycomb / Splunk: turn-key but multi-million-dollar.

---

## Q2 · Real-time ad analytics (Cloudflare / Hotstar style)

**Clarifications**:
- 100B impressions/day; 1B clicks; 100M conversions.
- Reports: per advertiser per campaign per hour, per geo, per device.
- Latency: end-to-end < 30s from impression to dashboard.
- Retention: 13 months for billing.

**Schema**:
- `impressions`, `clicks`, `conversions` tables sharded by `cityHash64(user_hash)` so user-attributed joins are co-located.
- `ORDER BY (advertiser_id, campaign_id, ts)`.
- ASOF JOIN clicks → impressions on `(user_hash, click_ts >= imp_ts AND click_ts <= imp_ts + INTERVAL 30 MIN)` for attribution.

**MVs**:
- `campaign_min_agg` per (advertiser, campaign, minute, geo, device).
- `campaign_hour_agg` cascaded from minute.
- `attribution_agg` from joined impressions+clicks.

**Ingest**:
- Edge collectors → Kafka → CH Kafka engine + MV.
- 30s end-to-end is achievable if Kafka lag and merge lag stay under control.

**Failure modes**:
- 4-day outage on attribution due to bad clock → reconcile via reingest.
- Tenant whale (one advertiser dominates) → dedicated shard.
- Late-arriving impressions → ReplacingMergeTree pattern for attribution rows.

---

## Q3 · User-facing embedded analytics in a SaaS app

**Clarifications**:
- 5K tenants; each sees their own analytics dashboard.
- 10K dashboard requests / second peak.
- p95 < 200ms per dashboard tile.
- Filters: date range, dimension, metric.

**Schema**:
- `events` table `ORDER BY (tenant_id, event_type, day, user_id, ts)`.
- Cascade MVs at minute / hour / day grain.
- Each tile reads from the appropriate MV.

**Multi-tenancy**:
- Row policies tied to `currentSetting('tenant_id')`.
- Per-tenant quotas.
- For the biggest 1-2% of tenants: dedicated CH service or shard.

**CDN-style**:
- Query result cache for popular dashboards.
- Pre-materialize trending queries via refreshable MVs.

**Alternative**:
- Pinot would also work (better at extreme QPS); CH chosen for SQL ergonomics + Cloud auto-scale.

---

## Q4 · Time-series metrics platform (Prometheus replacement)

**Clarifications**:
- 100M active series.
- 10M samples/second.
- PromQL-style queries: rates, sums, by-label aggregations.
- Retention: 13 months.

**Schema**:
- `metrics` table `ORDER BY (metric_name, ts)` partitioned monthly.
- DoubleDelta on `ts`, Gorilla on `value`.
- Labels as `Map(LowCardinality, LowCardinality)`.
- MV rollups at 1m, 5m, 1h, 1d.

**Query layer**:
- A query translator from PromQL → SQL with aggregates and `WITH FILL` for gap filling.
- Use `runningDifference` for `rate()`.

**Failure modes**:
- High-cardinality label explosion (URL paths, IDs) → enforce label cardinality limits at ingest.
- Spike of new series → MergeTree handles, but watch part count.

**Alternative**:
- VictoriaMetrics — purpose-built; competitive with CH for pure metrics. CH chosen for unified observability stack with logs/traces.

---

## Q5 · Replace ElasticSearch for log search

**Clarifications**:
- Currently 50 nodes ES cluster, 200 TB on disk.
- Top queries: substring search in `message` over last 24h scoped by `service` and `severity`.
- Migration: dual-write during transition.

**Schema**: see [13.4](13-schema-design-patterns.md#134-pattern-3--logs--observability-elk-replacement) — `(service, severity, ts)` ORDER BY + token bloom filter on `message`.

**Migration**:
- Phase 1: dual-write to ES + CH. Backfill historical from ES via export.
- Phase 2: switch read traffic to CH per service. Validate.
- Phase 3: decommission ES nodes by tier.

**Expected savings**:
- Storage: 200 TB ES → 30-40 TB CH (5-6× compression improvement).
- Hardware: 50 nodes ES → ~10 nodes CH.

**Failure modes**:
- Query that needs ES-style ranking → keep ES for those queries; CH for filter/agg.

---

## Q6 · Event sourcing read model (CQRS read side)

**Clarifications**:
- 500 events/sec/tenant; 100 tenants; 100M events/day.
- Build read views: per-account current state, per-account history, aggregated metrics.

**Schema**:
- `events_raw` MergeTree append-only, ordered by `(tenant, aggregate_id, version)`.
- `accounts_state` ReplicatedReplacingMergeTree built via MV from events, ordered by `(tenant, account_id)`.
- `account_metrics` AggregatingMergeTree via MV for per-account counters.

**Pattern**:
- Write each event once to `events_raw`.
- MVs propagate to read tables.
- Reads from the right shape for the right query.

**Failure modes**:
- A mis-applied event → reset the read model and re-derive from raw events.
- Schema evolution → re-derive (replay) the MVs.

---

## Q7 · GDPR-compliant user data store

**Clarifications**:
- User events with PII.
- Right-to-erasure: delete all data for user X within 30 days.
- Retention: 13 months.

**Schema**:
- Standard event table.
- PII fields in a separate "user dimension" table tied by user_id; the events table only references user_id.
- ReplacingMergeTree on the dimension with `is_deleted` + clean-deleted-rows.

**Erasure flow**:
- App receives request → mark `is_deleted = 1` in dimension with new updated_at → MV → propagates → ReplacingMergeTree drops on next merge.
- For the events table: lightweight DELETE on the rows; physical purge happens at merge time.

**Audit**:
- Audit log stored separately with longer retention.

---

## Q8 · Multi-region failover

**Clarifications**:
- Primary region X (US-East), DR region Y (US-West).
- RPO < 5 minutes; RTO < 30 minutes.

**Architecture**:
- Active in X (full CH cluster).
- DR in Y receives async replication via:
  - Cross-region ReplicatedMergeTree replicas (works but ingest pays RTT).
  - OR backup/restore to Y nightly + Kafka MirrorMaker for replay.
  - OR (Cloud) S3 cross-region replication of the data bucket + Keeper restore.

**Failover**:
- App switches read/write to Y; DNS or service-discovery flip.
- Validate replication lag was within RPO before cutover.

---

## Q9 · ETL load from Postgres (with CDC)

**Clarifications**:
- Source: Postgres OLTP with 100 tables.
- 1M writes/day; need near-real-time mirror in CH for analytics.

**Architecture**:
- Option A: `MaterializedPostgreSQL` engine — logical replication subscriber.
- Option B: Debezium → Kafka → CH Kafka engine → MV → ReplicatedReplacingMergeTree per table.

**Tradeoffs**:
- A is simpler ops; B is more flexible (you can route through transformations, send to other sinks too).
- Both make the CH tables eventually consistent with Postgres; latency a few seconds.

**Schema mapping**:
- Postgres types → CH types (BIGSERIAL → UInt64; TIMESTAMPTZ → DateTime64 UTC; JSONB → JSON or Map).
- Add CH-specific `updated_at` + `is_deleted` for ReplacingMergeTree.

---

## Q10 · Ad-hoc analyst workload on top of ingest pipeline

**Clarifications**:
- Analysts run ad-hoc SQL on a billion-row table; need < 10s.
- Should not impact ingest performance.

**Pattern**:
- Add an extra replica (Cloud: a "read" service) for analyst traffic.
- Strict per-user quotas (`MAX QUERIES = 30/min`, `MAX RESULT_BYTES = 10Gi`).
- Sampled tables (`SAMPLE BY intHash32(user_id)`) for exploratory queries.
- Query result cache for repeated dashboards.

---

## Q11 · Vector search / RAG analytics layer

**Clarifications**:
- 10M document embeddings (1024-dim float).
- ANN search by cosine similarity, plus metadata filters (date, source, tenant).
- 100s of QPS.

**Schema**:
- `documents` table with embedding as `Array(Float32)` or the new vector type.
- `ORDER BY (tenant, source, date)` for metadata pruning.
- Add a vector index (`USearch` / `Annoy` integration; in flight in CH).

**Query**:
```sql
SELECT id, cosineDistance(embedding, query_embedding) AS d
FROM documents
WHERE tenant = ?
ORDER BY d ASC LIMIT 10;
```

Filters first (granule pruning), then ANN over the surviving subset. Hybrid search = combine keyword filter + vector.

**Alternative**:
- A dedicated vector DB (Pinecone, Weaviate, Milvus) for very large vector catalogs with tight latency. CH wins when you want one DB for the embedding *and* its metadata + analytical aggregations.

---

## Q12 · Migrate from Druid

**Clarifications**:
- 500 TB Druid cluster across many segments.
- Need to retire Druid for cost / ops reasons.

**Plan**:
1. **Audit query mix** on Druid (`broker` logs).
2. **Design CH schema** per data source: ORDER BY matching common filters, MVs matching common aggregations.
3. **Dual-ingest**: send the same Kafka topics to both Druid and CH.
4. **Validation**: run the top 100 queries on both; compare results within tolerance.
5. **Per-data-source cutover**: switch BI to CH source-by-source.
6. **Backfill history** from Druid via `INSERT INTO ch_table SELECT ... FROM druid` (via JDBC) or by re-ingesting from Kafka history if available.
7. **Decommission Druid** after grace period.

**Wins**:
- Operational simplicity (1 server type vs 6).
- Lower cost.
- Full SQL.

**Risks**:
- Queries that depended on Druid features (e.g., dataSketches) need CH translations (`quantileTDigest`, `uniqCombined`).
- Real-time ingest with exactly-once may need Kafka offsets + dedup in CH.

---

## Cross-cutting interviewer follow-ups

- "How does this scale to 10×?" — name the next bottleneck (Keeper writes, S3 GET QPS, merger pool, query memory).
- "What if a region fails?" — DR section above.
- "How do you cost this?" — egress, storage, compute; per-tenant attribution.
- "What's your observability?" — system tables + Prometheus exporter + Grafana.
- "How do you migrate?" — dual-write + shadow-validate + per-tenant cutover.
