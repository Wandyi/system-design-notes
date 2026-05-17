# 20 · Staff Engineer Topics — Cost, Multi-Tenancy, Migration, Leadership

Cross-cutting topics that distinguish staff signal from senior signal. ClickHouse-flavored.

## 20.1 The cost equation

For ClickHouse Cloud:
- **Compute**: per-second per-vCPU. Largest cost line for read/write heavy.
- **Storage**: per GB-month on S3.
- **Egress**: GB out of region.
- **Backups**: incremental snapshots.

Levers from biggest to smallest impact:
1. **Materialized views**: turn 100 GB/query into 1 MB/query.
2. **Codec / type tuning**: 2-10× storage cut, also speeds reads.
3. **TTL aggressively**: stop paying for data nobody queries.
4. **Right-size compute**: pause idle services; auto-scale up only for peak.
5. **Avoid `SELECT *` and FINAL**: reduce S3 GET volume.
6. **Multi-CDN-class workload steering**: route analyst queries to a dedicated read service.

Have a P&L story for any architecture decision: "this MV costs $X/month in extra storage but saves $Y/month in dashboard compute. ROI Y/X."

## 20.2 SLI / SLO for an analytical platform

Per service:

| SLI | SLO | Alert |
|-----|-----|-------|
| Query p95 latency (per workload class) | < 1s dashboards, < 30s ad-hoc | 2× burn over 1h |
| Ingest end-to-end latency (event → queryable) | < 30s | > 60s for 5m |
| Insert success rate | > 99.9% | 99% in 5m |
| Replica `absolute_delay` | < 60s | > 120s for 5m |
| Mutation backlog | 0 stuck | any stuck > 1h |
| Active parts per (table, partition) | < 200 | > 300 |
| Memory tracking | < 80% of limit | > 90% |
| Disk free | > 25% | < 15% |

Each SLI mapped to a system-table query so the on-call has a one-line dashboard.

## 20.3 Multi-tenancy at scale

Three architectural levers:

### Logical isolation
- `tenant_id` as leading column.
- Row policies enforce.
- Quotas limit blast radius.

### Service isolation
- Big tenants get dedicated Cloud services (or shards on bare metal).
- Small tenants share a multi-tenant cluster.

### Network isolation
- VPC peering / PrivateLink for sensitive tenants.

A whale-customer that's 30% of a shared cluster is a signal to peel them off.

## 20.4 Migration playbook (CH ↔ X)

The general dual-write pattern works for any migration:

1. **Capture current workload**: query log, ingest topology, retention.
2. **Build the new system in parallel.**
3. **Dual-write** (Kafka → both sinks).
4. **Shadow-validate** queries on both for a window; compare results.
5. **Per-workload cutover**: BI dashboards / API consumers move source-by-source.
6. **Backfill historical** if needed (via SELECT INTO from old or replay from Kafka history).
7. **Decommission** after a grace period.

ClickHouse-specific: the source schema usually informs the CH schema (e.g., Druid dimensions → CH columns; ES fields → CH columns).

## 20.5 Capacity planning

For predictable peaks (sports streamers, Black Friday, NFL game day):
- Pre-warm CH Cloud service (force scale up before traffic).
- Pre-create partitions for `today()` to avoid the first-partition merge.
- Pre-load dictionaries to avoid first-query population.
- Pre-warm query cache for hot dashboards.
- Pre-create MV refresh windows.

## 20.6 Observability of *the ClickHouse cluster itself*

Already covered in [16](16-system-tables-and-observability.md). Staff-level points:
- Bring system tables into Prometheus via `clickhouse-exporter`.
- Wire alerts before going to prod.
- Have a runbook for the top 10 alerts.
- Practice incident drills: kill a Keeper node, kill a replica, deliberately fill the disk to 95%.

## 20.7 Schema evolution governance

In a multi-team org, lots of teams want to add columns. Without governance:
- Wide tables sprawl.
- Mis-ordered ORDER BYs ship.
- Bad type choices ship.
- Heavy mutations get scheduled during peak hours.

Patterns:
- **Schema review** step in CI for any DDL targeting a hot table.
- **Cookiecutter templates** for common tables (events, metrics, logs).
- **Owner per table** with named responsibility.
- **ALTER policies** — heavy mutations require off-hours scheduling.

## 20.8 Backup and DR

OSS: `clickhouse-backup` (third-party but de-facto), `BACKUP / RESTORE` (built-in for newer versions), partition-level FREEZE for point-in-time-ish snapshots.

Cloud: handled by Cloud team; periodic snapshots; configurable retention.

DR pattern (OSS):
- Daily full backup to S3.
- Hourly incremental.
- Kafka MirrorMaker to DR region for forward replay.
- Practice restore quarterly.

## 20.9 Performance review patterns

When a query is slow, the runbook:

1. `EXPLAIN PLAN indexes=1` — confirm index usage.
2. Look at `system.query_log` for ProfileEvents (read rows, bytes, marks).
3. Check `system.merges` for ongoing contention.
4. Check `system.replicas.absolute_delay`.
5. Check `system.parts` count for the table.
6. If the workload is regression-prone, add a perf-test in CI.

A staff engineer often writes the *test* before the *fix*, so the regression doesn't return.

## 20.10 ADR template (ClickHouse-flavored)

```
# ADR 0027: Adopt SharedMergeTree for new tenant pools

## Status
Accepted, 2026-04-01

## Context
We're onboarding 200 new tenants in Q3. Each will have variable usage with
infrequent peaks. Current on-prem cluster is sized for steady-state; peaks
cause partial outages.

## Decision
Provision a new ClickHouse Cloud account; new tenants land there.
Use SharedMergeTree for their tables; rely on auto-scale and pause-when-idle.

## Consequences
+ No infrastructure ops for new tenants.
+ Pay-per-use aligns with their early-stage spend.
+ Fast scale-up for peaks.
- Sub-second hot-cache reads are slightly slower than NVMe on-prem (10-50ms vs 1-5ms).
- Egress to other regions costs money; design queries to stay in-region.
- Stuck-mutation behavior differs slightly between OSS and Cloud; need new runbook.

## Alternatives Considered
- Buy 10× more bare-metal capacity: $X/month forever, idle most of the time.
- Add ZooKeeper-coordinated overflow cluster on-prem: high ops cost, weeks of work.
```

## 20.11 Cross-functional influence

Specific moments where the staff engineer's voice matters:
- **Convince the data team** to denormalize where they want a star schema.
- **Convince a BI team** to limit per-tenant quotas.
- **Convince product** to accept eventual consistency between OLTP and CH (vs. expensive real-time sync).
- **Convince infra** to invest in a Cloud service vs. expanding bare-metal.
- **Convince compliance** that lightweight DELETE + TTL satisfies erasure.

The pattern: write a one-pager with alternatives and tradeoffs; quantify; invite challenge; revise; commit.

## 20.12 The behavioral round

The same STAR templates from the streaming and Infoblox packs apply. ClickHouse-flavored framings:

- "Tell me about a time you migrated a non-trivial workload to or from ClickHouse."
- "How do you decide between adding a column vs. a separate table?"
- "How did you handle a hot tenant that was hurting other tenants?"
- "How do you mentor someone new to ClickHouse?"
- "What's a ClickHouse decision you regret?"
- "How would you organize a team to own a CH platform serving many internal users?"

## 20.13 Questions for the interviewer

- "What's the most operationally painful workload on your CH today?"
- "Are you on Cloud, OSS, or a mix? What drove the choice?"
- "How is the engine team organized — by subsystem (executor / storage / replication) or by surface (Cloud / OSS / integrations)?"
- "What's the biggest open architectural question right now? (E.g., MoQ-style for analytics, vector search, transactions.)"
- "Where do you see CH losing today to a competitor, and what's the plan?"
- "How is design-doc / ADR culture at the team?"
- "What's the on-call burden like?"

## 20.14 Must-internalize

- Cost levers ranked: MVs > codecs/types > TTL > right-size compute > query hygiene.
- SLI/SLO discipline; alerts wired before prod.
- Multi-tenant: logical + service-level + network-level isolation.
- Migration: dual-write + shadow-validate + per-workload cutover.
- Capacity planning for predictable peaks: pre-warm.
- Observability via system tables → Prometheus → Grafana.
- Schema evolution governance + per-table ownership.
- DR practiced, not just designed.
- Influence via written ADRs with quantified tradeoffs.
