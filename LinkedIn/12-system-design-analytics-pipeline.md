# 12 · System Design — Analytics Pipeline

The analytics pipeline is the *nervous system* of LinkedIn. Every member impression, every click, every API call, every server log, every model decision flows through here. It powers:

- **Member-facing analytics** (Creator dashboards, "Who viewed my profile?", Page Admin dashboards).
- **Business reporting** (Recruiter analytics, LMS reporting, finance).
- **Engineering observability** (latency dashboards, error rates).
- **ML feature engineering** (offline training data, online features).
- **A/B testing** (LIX metrics).
- **Anti-abuse and trust** (event-level abuse detection).

Total scale: hundreds of billions of events/day. Roughly: ingest pipeline + stream processing + OLAP store + batch warehouse + serving layer.

## 12.1 Requirements

### Functional

- Capture structured events from every service.
- Normalize / enrich with member, content, and context metadata.
- Provide:
  - **Real-time aggregates** (last minute, last hour, last day).
  - **Historical aggregates** (months, years).
  - **Ad-hoc queries** for engineers and analysts.
  - **Member-facing analytic dashboards** (creator views, Page admin).
- Provide a feature store for ML.
- Replay capability — re-run a pipeline if a bug is found.

### Non-functional

- **Scale**: hundreds of billions of events/day ingested. Hundreds of TB/day raw; ~10× that with enrichment.
- **Latency**: real-time queries < 1s. Member-dashboard updates < 5min. Batch reports daily.
- **Availability**: 99.9% for ingestion (lossy is bad, but a few-min delay is OK).
- **Durability**: zero data loss after Kafka acceptance.
- **Schema evolution**: producers and consumers evolve independently.

## 12.2 Architecture

```
   Services emit tracking events (Avro)
                │
                ▼
        ┌───────────────┐
        │ EventBus (Kafka) │  ← Kafka clusters, hundreds; trillions of msgs/day
        └────────┬──────┘
                 │
       ┌─────────┼──────────┬──────────────┐
       ▼         ▼          ▼              ▼
   ┌────────┐ ┌────────┐ ┌────────┐  ┌────────────┐
   │ Samza/ │ │ Pinot  │ │ Spark / │  │  Trust ML  │
   │ Flink  │ │ ingest │ │ Hadoop  │  │  pipelines │
   │ stream │ │        │ │ batch   │  └────────────┘
   │ jobs   │ │        │ │ ETL     │
   └────────┘ └────┬───┘ └────┬────┘
                   │           │
                   ▼           ▼
              ┌────────┐  ┌─────────────────┐
              │ Pinot  │  │ Iceberg /       │
              │ tables │  │ OpenHouse (data │
              │        │  │ lake on Azure)  │
              └────────┘  └─────────────────┘
                                   │
                                   ▼
                          ┌─────────────────┐
                          │ Trino / Spark   │
                          │ ad-hoc query    │
                          └─────────────────┘
```

## 12.3 EventBus — the Kafka layer

The bedrock. Several layers:

- **Local Kafka** — per data center / region, raw events.
- **Aggregate Kafka** — mirrored / merged across regions for global pipelines.
- **Tracking Kafka** — separate cluster for low-priority, high-volume tracking events with retention tuned (~7–14 days).
- **Critical Kafka** — separate cluster for transactional, must-not-lose events with higher replication / SLA.

Key practices:
- **Avro schemas** with a schema registry. Producers must register schemas; backward/forward compatibility enforced.
- **Topic naming convention**: `<domain>.<action>.<version>` or similar.
- **MirrorMaker 2 / Brooklin** for cross-cluster replication.

LinkedIn invented Kafka — and pushed it harder than anyone else. See `15-kafka-deep-dive.md`.

## 12.4 Stream processing

### Samza

LinkedIn's home-grown stream processor (Apache Samza). Used for:
- Real-time aggregation (e.g., "comment counts per post" updated as comments stream in).
- Joining streams (e.g., enrich a click event with the member's current profile attributes).
- Building near-real-time feature stores (write to Venice).
- Anomaly detection.

Mental model: stateful processors with **local RocksDB**; state changes also written to Kafka changelog topics for replayable durability.

### Flink

Growing adoption. Stronger event-time / windowing semantics than Samza. Used for newer pipelines that need exactly-once SQL semantics or complex CEP (complex event processing).

### Beam

Occasionally used as an abstraction over Flink/Samza.

## 12.5 Pinot — real-time OLAP

This is the workhorse for *interactive* analytics. See `17-pinot-and-samza.md` for deep details.

Pattern:
- A **Pinot table** is fed from Kafka (real-time) and/or Hadoop (offline).
- Columnar, with inverted indexes on dimensions and aggregations precomputed via star-tree indexes.
- Queries: SQL with aggregations + filters; sub-second over billions of rows.

Use cases at LinkedIn:
- **Who viewed my profile** ("WVMP") — counts of impressions/clicks per member.
- **Creator analytics** — engagement metrics per post.
- **Recruiter analytics** — funnel metrics.
- **LMS reporting** — campaign performance.
- **Anomaly detection** — real-time monitoring of business metrics.

## 12.6 Batch / data lake

- **Hadoop (HDFS)** historically; migrating to **Azure Data Lake / object storage**.
- **Iceberg** as the modern table format (or LinkedIn's variant via **OpenHouse**).
- **Spark** for transformations; **Trino** for ad-hoc SQL.
- **Hive** still around for legacy.

Batch is the source-of-truth for analytics. The pattern:
- **Bronze**: raw events.
- **Silver**: cleaned, schema-enforced, deduped.
- **Gold**: aggregated by entity (per member per day, per campaign per day, etc.).

Lambda-architecture-style: stream + batch produce metrics; batch corrects stream's approximations. (LinkedIn has been moving toward Kappa-architecture for some workloads — single Kafka → Flink pipeline that doubles as the source-of-truth — but Lambda is still common.)

## 12.7 Schema registry and EventBus contracts

A staff candidate will discuss:
- Producers register Avro schemas with the schema registry.
- Consumer schemas evolve; deserialization handles missing/added fields.
- **Schema validation in CI**: a backward-incompatible change is caught at PR time.
- **Deprecation flow**: removing a field requires multi-quarter consumer migration.

This is the *contract* layer that makes the whole org function — without it, every producer change breaks downstream consumers silently.

## 12.8 Feature store

ML pipelines need:
- **Online features**: read-side at inference time (low-latency, freshness within minutes).
- **Offline features**: training-side (point-in-time correct, weeks/months of history).

LinkedIn uses:
- **Feathr** (open-sourced) — DSL for defining features once, materialized both ways.
- **Venice** as the online store.
- **HDFS / Iceberg** as the offline store.

Critical issue: **train/serve skew**. The same feature definition must produce identical values online and offline. Feathr's promise.

## 12.9 Member-facing analytics surfaces

### Who Viewed My Profile (WVMP)

Classic example. Each profile view is an event → aggregated per (viewed_member, day, viewer_attributes) → Pinot.

- Privacy: viewer-anonymity is honored unless viewer in public mode.
- Free members see capped counts; Premium sees full list.
- Real-time-ish: a view should appear within minutes.

### Creator Analytics

- For each post the creator made: impressions, engagement, demographic breakdown.
- Pinot powers the dashboard.
- Sub-second queries on aggregated data.

### Page Admin Dashboards

- Same pattern: events flow → Pinot → dashboard.

## 12.10 LIX (experimentation)

A specialized pipeline:
- Member is assigned to a treatment cohort (deterministic via member_id hash).
- Events emitted with the treatment label.
- Analyzer joins event streams with treatment assignments → computes lift on metrics.
- Statistical significance + guardrail checks.
- Results visible to engineers within hours / days.

Staff-level discussion: the statistics (peeking, false-discovery, sequential testing, novelty effects).

## 12.11 Trust & anti-abuse pipelines

- Real-time scoring of every event against ML models (spam, account takeover, fraud rings).
- Block / shadow-ban / step-up auth at sub-second decision.
- Retrospective batch sweeps catch slow-moving abuse the real-time model missed.

## 12.12 Multi-region

- Local Kafka per region.
- Cross-region replication for global aggregates.
- Pinot tables can be regional or global depending on use case.
- Member-facing dashboards: served from member's home region; eventually consistent across regions.

## 12.13 Failure modes

- **Kafka outage** — buffer at producers (in-memory, with disk overflow); replay on recovery. Some loss possible if producer dies before flush.
- **Stream-processor lag** — downstream metrics behind. Pinot real-time-table-only mode falls back to offline-table mode for older intervals.
- **Schema mismatch** — DLQ topic for unparseable events; alert; fix and replay.
- **Pinot segment corruption** — rebuild from offline; serve from offline-tier while real-time recovers.
- **Disk full on data lake** — auto-scale; cold-storage tiering.

## 12.14 Common follow-ups

> **"How would you support 'real-time' member-facing dashboards with < 30s latency?"**
Producer → Kafka → Pinot real-time table. Pinot real-time has minute-level freshness; with tighter ingest tuning, ~30s. Pre-aggregate hot dimensions.

> **"What about historical comparison (last year same week)?"**
Combine real-time table with offline historical table in a Pinot SQL union. Or precompute YoY deltas in batch.

> **"How do you guarantee no double-counting on a click?"**
Idempotency key on the event (request_id). Stream processor dedup window (e.g., 24 hour state). Batch recompute corrects.

> **"How do you serve a metric that requires joining two streams?"**
Samza/Flink stateful join with a keyed window. Or denormalize at the producer side. Or use Pinot's "lookup join" against a Venice store.

> **"How do you debug a metric regression that nobody noticed for a week?"**
Replay from Kafka (extended retention). Re-run the offline batch transform. Reconcile.

> **"How do you protect against a noisy producer flooding Kafka?"**
Per-topic quotas. Producer SLO monitoring. Auto-throttle in EventBus client library.

> **"How would you migrate a major schema (add a new dimension to all tracking events)?"**
Co-existence — old + new schemas in parallel. Backfill old events with imputed values where possible. Coordinate consumer-by-consumer rollout. Don't try to do it atomically.