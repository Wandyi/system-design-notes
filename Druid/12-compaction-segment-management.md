# 12 · Compaction and Segment Management

Streaming ingest creates many small segments. Without compaction, you accumulate fragmentation that destroys query latency. The Coordinator's **auto-compaction** is the answer.

## 12.1 Why compaction matters

A Kafka supervisor produces a segment every `taskDuration` (e.g., per hour). At `taskCount=4, replicas=2`, that's 8 segments per hour per partition group. Over a day: ~200 segments. Over a week: ~1500.

A query that spans a week then fans out to 1500 segments. The Broker → Historicals → merge has per-segment overhead. Latency degrades.

Compaction merges these small segments into 300-700 MB ones, dramatically cutting fan-out.

## 12.2 Auto-compaction config

```json
{
  "dataSource": "events",
  "tuningConfig": {
    "maxRowsPerSegment": 5000000,
    "maxNumConcurrentSubTasks": 4
  },
  "granularitySpec": {
    "segmentGranularity": "DAY",
    "queryGranularity": "MINUTE"
  },
  "skipOffsetFromLatest": "PT1H",
  "taskPriority": 25
}
```

Reading:
- Target: 5M rows per segment.
- `skipOffsetFromLatest = PT1H` — don't compact data newer than 1 hour (streaming might still be ingesting into it).
- `maxNumConcurrentSubTasks = 4` — up to 4 compaction tasks at once.
- `taskPriority = 25` — lower than ingestion (typical 75), so ingestion preempts.

Submitted to the Coordinator via `POST /druid/coordinator/v1/config/compaction`.

## 12.3 The compaction lifecycle

1. Coordinator's compaction submitter periodically scans datasources for fragmentation.
2. Picks intervals where small-segment count exceeds threshold.
3. Issues a compaction task to the Overlord.
4. The task reads the existing segments, writes new segments with the configured `segmentGranularity` / `queryGranularity` / `clusteredBy` / etc.
5. New segments overshadow old (higher version).
6. Old segments are marked unused. Later, a kill task deletes them from deep storage.

Compaction is **online** — queries see consistent results before, during, and after (the segment swap is atomic per-interval).

## 12.4 Re-rollup during compaction

Compaction tasks can re-apply rollup. If the streaming ingest used a fine granularity (e.g., MINUTE) and compaction is configured to a coarser granularity (e.g., HOUR), the rollup ratio improves.

This is a clean way to **age-down precision**: keep minute-grain for last 7 days, hour-grain for last 30 days, day-grain for older — all via compaction config (with multiple compaction specs by interval).

## 12.5 Compaction and re-ingest interplay

If you re-ingest (via MSQ REPLACE) into the same interval that's actively being compacted, the latest version wins; older versions are overshadowed.

Best practice: pause auto-compaction during heavy backfills, or use `skipOffsetFromLatest` aggressively to keep them clear of each other.

## 12.6 Kill tasks — physically delete

Compacted / unused segments are marked **unused** in the metadata DB but the segment files remain in deep storage. Kill tasks physically delete them.

Auto-kill (in newer versions):
```json
"killDataSourceWhitelist": "*",
"killPeriod": "P1D",
"killDurationToRetain": "P30D"
```

Translates to: every day, run a kill task that deletes unused segments older than 30 days from deep storage.

Without auto-kill, deep storage grows monotonically — a common Druid ops mistake.

## 12.7 Marking unused / re-attaching

```bash
# mark all segments for a datasource unused (segment is not deleted, just not loaded)
POST /druid/coordinator/v1/datasources/events/markUnused

# re-enable
POST /druid/coordinator/v1/datasources/events/markUsed
```

Useful for staging: ingest into a "staging" datasource, validate, mark used. Or to quickly hide a bad datasource without deleting.

## 12.8 Segment troubleshooting

```sql
-- via sys schema
SELECT datasource, count(*) AS segments, sum(size) AS bytes
FROM sys.segments
WHERE is_published = 1
GROUP BY 1
ORDER BY 3 DESC;

-- find small segments
SELECT datasource, interval, count(*) AS segments_in_interval
FROM sys.segments
WHERE is_published = 1 AND size < 50000000
GROUP BY 1, 2
HAVING count(*) > 10
ORDER BY 3 DESC;
```

Indicators of trouble:
- Many segments under 100 MB → compaction is behind or not configured.
- Replica count below target → Coordinator can't place segments (disk full? tier missing?).
- Long-running compaction tasks → tune `maxNumConcurrentSubTasks`.

## 12.9 The "OPTIMIZE FINAL" anti-pattern doesn't apply (much)

ClickHouse engineers often reach for `OPTIMIZE FINAL` to "fix" things. The Druid equivalent — a one-off massive compaction across the entire datasource — is similarly bad if you trigger it during ingest hours. Set it up to run continuously in background instead, or schedule one-off compactions in off hours.

## 12.10 Schema-changing compaction

Compaction can change the schema (within limits):
- Drop columns no future queries need.
- Change `queryGranularity` (with re-rollup).
- Change `clusteredBy` (resort segments).
- Add `metricsSpec` (only if computable from existing columns).

Not allowed via compaction:
- Add a dimension that wasn't in the source (because the original raw values are gone after rollup).
- Change a dimension's type incompatibly.

For those, re-ingest from the source.

## 12.11 The TTL story

Druid's "TTL" is **drop rules** in load rules:

```json
{ "type": "dropByPeriod", "period": "P90D" }
```

Segments older than 90 days are dropped from Historicals (still in deep storage). Add a kill task to physically delete.

Compare to ClickHouse's `TTL ts + INTERVAL 90 DAY DELETE` — same logical effect.

## 12.12 The cold tier as soft retention

Alternative to dropping: move to `cold` tier with fewer replicas. Data is still queryable but on cheaper hardware, smaller footprint.

```json
[
  { "type": "loadByPeriod", "period": "P30D", "tieredReplicants": { "hot": 2 } },
  { "type": "loadByPeriod", "period": "P365D", "tieredReplicants": { "cold": 1 } },
  { "type": "dropForever" }
]
```

## 12.13 Backup and restore

- **Backup the metadata DB.** This is the single most important Druid backup. Without it, the segments in deep storage are unattributed.
- **Backup deep storage.** S3 cross-region replication is standard. Don't rely on a single bucket.
- **Test the restore.** Spin up a new cluster pointing at restored metadata + deep storage; segments come up.

DR pattern:
- Primary cluster runs ingest + serves queries.
- DR cluster: metadata DB replicated; deep storage cross-region replicated; spin up Historicals on demand.

## 12.14 Anti-patterns

- **No auto-compaction** — segment count grows; queries slow.
- **No auto-kill** — deep storage cost grows unbounded.
- **Compaction during peak ingest** — tasks contend for slots; either may suffer.
- **Compacting too aggressively** (very large `maxRowsPerSegment`) — huge segments, slow downloads.
- **Marking unused but never deleting** → fills metadata DB segment table.
- **Skipping `skipOffsetFromLatest`** → compaction touches still-ingesting intervals → tasks conflict.

## 12.15 Must-internalize

- Auto-compaction prevents small-segment fan-out from killing query latency.
- Compaction can re-rollup (age-down precision over time).
- Mark-unused + kill-task = physical deletion from deep storage. Auto-kill in newer versions.
- Load + drop rules govern retention; the cold tier is soft retention.
- Back up the metadata DB; cross-region replicate deep storage.
- Compaction is online; the segment swap is atomic per interval.

---

## Sources

- [Compaction — official](https://druid.apache.org/docs/latest/data-management/compaction/)
- [Automatic compaction](https://druid.apache.org/docs/latest/data-management/automatic-compaction/)
- [Deleting data](https://druid.apache.org/docs/latest/data-management/delete/)
- [Update](https://druid.apache.org/docs/latest/data-management/update/)
