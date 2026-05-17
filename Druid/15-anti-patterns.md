# 15 · Anti-Patterns

Avoidable disasters in Druid. Tuned to the engine; some directly contrast ClickHouse anti-patterns.

## 15.1 No auto-compaction

Streaming ingest produces small segments. Without auto-compaction they multiply. Brokers fan out to thousands of segments per query → latency degrades from sub-second to multi-second.

**Fix**: configure auto-compaction from day one. Target 300-700 MB segments.

## 15.2 No auto-kill

Compacted / unused segments stay in deep storage forever. S3 bill grows monotonically; metadata DB segment table fills up.

**Fix**: auto-kill old unused segments (after a safety period like 30 days).

## 15.3 Treating Druid as mutable

"We'll just UPDATE the event when the user's name changes." Druid has no UPDATE. Designs that assume mutability fail.

**Fix**: model entity-current-state with a separate datasource maintained via MSQ REPLACE (or use lookups for stable dims).

## 15.4 Including high-cardinality dimensions

Putting `session_id` or `request_id` in `dimensions` → rollup ratio 1× → segments bloat.

**Fix**: replace with HLL/theta sketch metrics; or keep raw with `createBitmapIndex: false` (saves bitmap storage).

## 15.5 No time filter in queries

```sql
SELECT count(*) FROM events;  -- scans every segment ever ingested
```

Druid fans out to every Historical owning a segment in `events`. Slow, expensive, sometimes crashes.

**Fix**: always include `WHERE __time >= ... AND __time < ...`. Set `druid.sql.planner.requireTimeCondition = true` to enforce.

## 15.6 Wrong queryGranularity choice

- `queryGranularity = NONE` when MINUTE would do → 60× more rows.
- `queryGranularity = HOUR` when queries need MINUTE granularity → forced over-aggregation; can't drill down.

**Fix**: pick coarsest granularity your dashboards need.

## 15.7 Wrong segmentGranularity

- Hourly segments for years of retention → 8760 segments per year per shard; Coordinator overhead.
- Yearly segments for streaming → segment open all year, never compactable.

**Fix**: typical: HOUR for hot streaming → compact to DAY for older. Or DAY everywhere.

## 15.8 Joining big tables on the query path

Broadcast joins on a million-row right side blow up memory. Druid isn't built for shuffle joins.

**Fix**: denormalize via MSQ; use lookups; use theta sketches for cohort-style.

## 15.9 Excessive datasources

One datasource per tenant when you have 10K tenants → Coordinator load loops drag; metadata DB segment table is huge.

**Fix**: shared datasource with `tenant_id` leading dimension; per-tenant access via authorization layer.

## 15.10 Many small lookups

Loading 200 lookups, each refreshing every minute → Broker memory pressure, network thrash.

**Fix**: consolidate; refresh rarely; cache.

## 15.11 Over-rollup

Querying for fine-grain detail on heavily-rolled-up data — can't drill into individual events.

**Fix**: maintain a raw datasource (with `rollup: false`) for forensic analysis alongside the rolled-up one.

## 15.12 Mixing rolled-up and raw in one datasource

If you ingest some events with rollup and some without, queries become unpredictable.

**Fix**: separate datasources. Or use consistent rollup config across all ingestion supervisors.

## 15.13 Forgetting deep storage cleanup

Even with auto-kill, mis-configured tasks can leave orphan files in S3.

**Fix**: periodic S3 inventory + reconcile against `sys.segments` to find orphans.

## 15.14 Single ZooKeeper / metadata DB / deep storage

All three are SPOFs.

**Fix**:
- ZK: 3 or 5 nodes.
- Metadata DB: replicated; failover tested.
- Deep storage: S3 with cross-region replication.

## 15.15 Coordinator + Overlord on the same small VM

For a small cluster this is OK. For a large one (1000s of segments / many tasks), they need their own beefy machines.

**Fix**: scale up Master tier.

## 15.16 Broker memory tuning ignored

Subqueries materialize on Broker. Big queries OOM the Broker.

**Fix**:
- `druid.server.http.maxSubqueryRows` cap.
- `druid.processing.buffer.sizeBytes` per processing thread.
- Heap sized for the largest expected subquery.

## 15.17 Over-replication on cold tier

Cold tier with 3 replicas of segments nobody queries → wasted storage cost.

**Fix**: 1 replica on cold tier; rely on deep storage for true durability.

## 15.18 Not monitoring metadata DB latency

Slow metadata DB queries → tasks fail, supervisors lag, Coordinator load cycles drag.

**Fix**: monitor query latency on the metadata DB. Index `druid_segments` properly (most managed Druid distros do this).

## 15.19 Schema discovery + huge dimension explosion

Auto-discovery + unbounded user-uploaded field names = dimension count explodes; segment size balloons.

**Fix**: cap discovered dimensions; use JSON columns for truly unknown fields; periodically audit dimension list.

## 15.20 Late events triggering many small segments

Late events that arrive hours after their timestamp create new segments in already-published intervals. Without compaction, you end up with dozens of tiny segments per old interval.

**Fix**: auto-compaction with appropriate window; or `lateMessageRejectionPeriod` if late data isn't required.

## 15.21 Querying without LIMIT in user-facing endpoints

A user clicks a bad filter → query returns millions of rows → Broker OOMs.

**Fix**: enforce `LIMIT` at API gateway; set `druid.server.http.maxQueryResults` cap.

## 15.22 Compaction during ingest peak

Compaction tasks contend for slots with ingestion tasks.

**Fix**: lower compaction `taskPriority`; cap `maxNumConcurrentSubTasks`; schedule heavy compaction off-hours.

## 15.23 No EXPLAIN PLAN check

Shipping a query without verifying it picks the right native query type leads to surprises.

**Fix**: `EXPLAIN PLAN FOR` in code review; flag GroupBy where TopN/Timeseries was expected.

## 15.24 Storing money as `Double` metric

Floating-point imprecision for billing. (Druid metrics are typically Long/Double/Float — no Decimal.)

**Fix**: store as Long in smallest unit (cents). Convert at query time.

## 15.25 Using SQL `COUNT(DISTINCT)` on big data without `useApproximateCountDistinct`

Exact distinct count over a billion rows → OOM or extremely slow.

**Fix**: set `useApproximateCountDistinct = true` globally; or use pre-aggregated HLL sketches.

## 15.26 Single Historical per tier

Failure = data offline for that tier.

**Fix**: 2+ Historicals per tier; appropriate replication count.

## 15.27 ZK paths growing unbounded

If something stops cleaning up ZK paths (rare bug or deliberate hold), ZK degrades.

**Fix**: monitor ZK node count; investigate growth.

## 15.28 Coordinator console as the only ops surface

The console is fine for a few people. Real ops needs metrics → Prometheus / Datadog → alerts.

**Fix**: wire JMX/Druid emitter metrics into your observability stack.

## 15.29 Not testing restore from backup

Backups that you've never restored aren't backups; they're optimism.

**Fix**: quarterly drill: restore metadata DB to a non-prod cluster, point at deep storage, validate.

## 15.30 Single-region for everything

Region outage = total Druid outage.

**Fix**: cross-region deep storage replication + secondary cluster (cold standby).

## 15.31 Must-internalize

- Auto-compaction + auto-kill are mandatory.
- Always time-filter; enforce via `requireTimeCondition`.
- Don't include high-cardinality dims; use sketches.
- Pre-decide rollup; you can't undo it cheaply.
- Joins via lookup or denormalize; never shuffle.
- Multi-tenant: shared DS + leading tenant_id + auth.
- Monitor metadata DB, ZK, Broker memory, segment count.
- Use approximate aggregates by default.
- Replication + cross-region for DR.
