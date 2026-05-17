# 20 · Staff Engineer Topics

Cross-cutting topics for staff signal. Mirrors the ClickHouse pack [20](../Clickhouse/20-staff-engineer-topics.md), tuned to Druid.

## 20.1 Cost levers

For Imply Polaris (managed) or self-hosted:

1. **Rollup ratio** — biggest single lever. A 10× rollup = 10× less data on disk, 10× faster queries.
2. **Tiered storage** — small hot fleet (NVMe) + large cold fleet (cheap disk) much cheaper than uniform.
3. **Auto-compaction** — keeps segment count low, query cost low.
4. **Auto-kill** — bounds deep storage cost.
5. **Replication count by tier** — 2× on hot (HA), 1× on cold (rely on deep storage for durability).
6. **DataSketches** — replace high-cardinality dims with sketches; massive storage win.
7. **MSQ for batch** — cheap when scheduled off-hours.
8. **Lookup vs broadcast join** — lookup is free, broadcast costs per query.

## 20.2 SLI / SLO

For Druid-backed analytics:

| SLI | SLO | Alert |
|-----|-----|-------|
| Query p95 latency (per workload) | < 1s for dashboards, < 5s for ad-hoc | 2× burn over 1h |
| Ingest end-to-end (event → queryable) | < 30s | > 60s for 5m |
| Supervisor lag (Kafka offsets vs latest) | < 60s | > 120s for 5m |
| Unavailable segments | 0 | any > 0 for 5m |
| Under-replicated segments | 0 sustained | > 0 for 30m |
| Auto-compaction backlog | < 100 small segments | > 500 |
| Failed task rate | < 1% | > 5% in 1h |
| Coordinator cycle | < 1m | > 2m |
| Metadata DB latency | < 100ms p99 | > 500ms |
| Historical disk free | > 25% | < 15% |

## 20.3 Multi-tenancy

Two main patterns ([13.6](13-schema-design-patterns.md#136-pattern-5--multi-tenant-saas-analytics)):

### Logical (shared datasource)
- `tenant_id` first dimension.
- Row-level filter at the auth/proxy layer.
- Per-tenant query rate limits.

### Physical (per-tenant datasource)
- Clean isolation.
- Per-tenant rules/compaction/tiering.
- Limited to ~1000s of datasources before Coordinator strain.

### Hybrid (best for whales + long tail)
- Top tenants: dedicated datasources (and maybe dedicated tier).
- Long tail: shared datasource with logical isolation.

## 20.4 Migration patterns

### Migrate into Druid
- Plan rollup, sketches, granularity upfront.
- MSQ INSERT for backfill from S3 Parquet.
- Streaming supervisor for forward data.
- Dual-write phase if migrating from another OLAP (ClickHouse / Snowflake / etc.).

### Migrate out of Druid
- Identify queries' equivalents in target system.
- Dual-write and shadow-validate.
- Per-dashboard cutover.

## 20.5 Schema change governance

- **Adding a dimension** is cheap (future segments only).
- **Changing rollup** requires re-ingest.
- **Removing a dimension** is cheap (just stop emitting in ingest spec).
- Have a schema-review process for high-volume datasources.

## 20.6 ADR template (Druid-flavored)

```
# ADR 0042: Use DataSketches HLL for distinct counts in events DS

## Status
Accepted, 2026-04-01

## Context
We need distinct user counts in the dashboard with sub-second latency.
Storing raw user_id as dimension kills rollup (10× more storage; rollup ratio falls
from 12× to 1×).

## Decision
Drop user_id from dimensions. Add HLL sketch metric (lgK=12, ~1.6% error).
Update all distinct-user queries to use APPROX_COUNT_DISTINCT_DS_HLL.

## Consequences
+ Storage down 10× for this datasource.
+ Query latency from ~5s → ~200ms.
+ Mergeable across segments.
- ~1.6% error on distinct count; accepted (dashboard rounding to thousands anyway).
- Cohort intersection requires theta sketch — add as separate metric.

## Alternatives Considered
- Keep raw user_id as dimension: rollup ratio = 1, 10× cost.
- Use lower-precision HLL (lgK=10): less storage, higher error — not worth the savings.
```

## 20.7 Influence patterns

Specific situations:
- **Convince product** to accept ~1% error on distinct count for 10× cost savings.
- **Convince an upstream team** to align Kafka topics with Druid datasource boundaries.
- **Convince infra** to add Historicals for a tier or invest in MSQ scaling.
- **Convince a tenant team** that their schema decisions cost the whole platform money.

Pattern: one-pager, alternatives + cost/benefit, get input, decide.

## 20.8 Behavioral round prompts

Druid-flavored:
- "Tell me about a time you re-designed a datasource for performance."
- "How did you handle a hot tenant?"
- "How did you migrate a workload from one OLAP to another?"
- "What's a Druid feature you wish existed?"
- "How do you decide between Druid and ClickHouse for a new use case?"
- "How do you balance ingest reliability vs query latency?"

## 20.9 Questions for the interviewer

- "What's the team's biggest open architectural question right now? (Storage/compute split? UPSERT? Vector?)"
- "Where do you see Druid losing today to ClickHouse / Pinot, and what's the response?"
- "How is the team split between Apache Druid contributions and Polaris-only features?"
- "What's the on-call burden like? How fragmented is the cluster surface area?"
- "How is design-doc / ADR culture at the team?"
- "What's the biggest single piece of tech debt?"
- "Tell me about a Druid customer migration you're proud of."

## 20.10 Must-internalize

- Cost levers: rollup, tiers, compaction, kill, sketches.
- SLI/SLO numbers above; alerts wired before prod.
- Multi-tenant: logical / physical / hybrid.
- Migration: dual-write + shadow-validate + cutover.
- ADR culture with quantified tradeoffs.
- Influence by written briefs and clear tradeoffs.
