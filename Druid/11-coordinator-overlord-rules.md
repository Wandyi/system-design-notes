# 11 · Coordinator, Overlord, and Rules

The control plane. Coordinator manages **segments** (placement, replication, compaction, kill). Overlord manages **tasks** (ingestion, MSQ, compaction). Both are leader-elected (single active, others standby).

## 11.1 The Coordinator's responsibilities

Runs on a fixed cadence (default `druid.coordinator.period = PT1M`):

1. **Load segments** — for each unassigned segment, pick Historical(s) per rules and replication count.
2. **Drop segments** — for segments matching drop rules or marked unused, remove from Historicals.
3. **Balance** — periodically move segments from overloaded Historicals to underloaded ones.
4. **Compact** — issue auto-compaction tasks for fragmented intervals.
5. **Kill** — issue kill tasks to delete unused segments from deep storage.
6. **Monitor** — track per-segment replication state.

## 11.2 Load rules — the retention policy DSL

Rules are ordered. Each segment matches the **first** rule that applies.

### Common rule types

```json
[
  // Hot tier: last 7 days, 2 replicas
  {
    "type": "loadByPeriod",
    "period": "P7D",
    "includeFuture": true,
    "tieredReplicants": { "hot": 2 }
  },
  // Default tier: last 30 days, 1 replica
  {
    "type": "loadByPeriod",
    "period": "P30D",
    "tieredReplicants": { "_default": 1 }
  },
  // Cold tier: last year, 1 replica
  {
    "type": "loadByPeriod",
    "period": "P1Y",
    "tieredReplicants": { "cold": 1 }
  },
  // After 1 year: drop entirely (no Historicals serve)
  {
    "type": "dropForever"
  }
]
```

### Rule types reference

| Rule | Effect |
|------|--------|
| `loadForever` | Keep on the named tiers forever |
| `loadByPeriod` | Keep segments within the past N period |
| `loadByInterval` | Keep segments within an explicit interval |
| `dropForever` | Drop all matching segments |
| `dropByPeriod` | Drop segments older than N |
| `dropByInterval` | Drop within explicit interval |
| `dropBeforeByPeriod` | Drop segments before a period |
| `broadcastForever` | Load segment on every Historical (for tiny "broadcast" datasources used in joins) |

Rules can be **global** or **per-datasource**.

## 11.3 Tiers and replication

Tiers are logical groupings of Historicals. Common setup:

- `hot` tier: small fleet of NVMe-backed Historicals for last 7 days.
- `_default` tier: SSD-backed for last month.
- `cold` tier: HDD or smaller machines for older data.

Each tier is configured in `historical.properties` via `druid.server.tier = hot` / `_default` / `cold`.

The Coordinator places segments to tiers per the load rules. Within a tier, replication count is independent.

Cost play: hot tier has expensive hardware but small footprint (last week). Cold tier has cheap hardware and big footprint (last year). Total cost is much less than uniform hot-tier sizing.

Compare to ClickHouse: CH does tiering via storage policies (TTL TO DISK) within a node's storage. Druid does it across separate node fleets. Druid's model is more granular but more ops to run.

## 11.4 Segment balancing

The Coordinator periodically moves segments between Historicals to even load:
- **Cost-based balancer** (default) — minimizes a cost function (segment size, age, host load).
- **Random balancer** — uniform random.

Balancing throttle: `druid.coordinator.balancer.strategy` + `maxSegmentsToMove` per cycle. Tuning balance against ingestion noise is a real op.

## 11.5 The Overlord's responsibilities

Manages **tasks**:
1. Receives task submissions (ingestion specs, MSQ queries, compaction tasks, kill tasks).
2. Assigns tasks to MiddleManagers / Indexers with available slots.
3. Tracks task lifecycle (PENDING / WAITING / RUNNING / SUCCESS / FAILED).
4. Manages **supervisors** for streaming.

### Task slots

Each MiddleManager declares `druid.worker.capacity` (slots). The Overlord assigns up to that many tasks per MiddleManager. Indexers run multiple tasks per JVM up to a configured limit.

Slot exhaustion → tasks wait in PENDING. Scale by adding MiddleManagers / Indexers, or by increasing per-node slot counts (which trades concurrency for isolation).

### Task types

- `index_kafka` — Kafka streaming task.
- `index_kinesis` — Kinesis streaming task.
- `index_parallel` — native batch.
- `query_controller`, `query_worker` — MSQ.
- `compact` — auto-compaction.
- `kill` — segment deletion from deep storage.

## 11.6 The Coordinator web console

The built-in UI (served via Router or directly):
- **Data sources** — list, segment counts, sizes, segment-state breakdown.
- **Segments** — drill-down per segment: interval, version, size, location.
- **Tasks** — Overlord-side task list with logs.
- **Supervisors** — streaming supervisors and their state.
- **Services** — process inventory.
- **Lookups** — load/refresh/delete lookups.

A staff engineer should be comfortable navigating this for incident triage.

## 11.7 RBAC and authorization (extension)

Druid Basic Security extension or Imply's enterprise extensions:

```sql
GRANT READ ON DATASOURCE events TO ROLE analyst;
GRANT WRITE ON DATASOURCE events TO ROLE ingester;
GRANT ALL ON ALL DATASOURCES TO ROLE admin;
```

Row-level filtering is via lookups + filter rules. (Druid is less out-of-the-box for row-level security than ClickHouse's row policies.)

## 11.8 Common operational tasks via API

```bash
# Submit a supervisor
curl -X POST -H 'Content-Type: application/json' -d @supervisor.json \
     http://overlord:8081/druid/indexer/v1/supervisor

# Suspend a supervisor
curl -X POST http://overlord:8081/druid/indexer/v1/supervisor/<id>/suspend

# Reset a supervisor (clear bad state)
curl -X POST http://overlord:8081/druid/indexer/v1/supervisor/<id>/reset

# Submit a kill task
curl -X POST -H 'Content-Type: application/json' \
     -d '{"type":"kill","dataSource":"events","interval":"2024-01-01/2024-02-01"}' \
     http://overlord:8081/druid/indexer/v1/task

# Update load rules for a datasource
curl -X POST -H 'Content-Type: application/json' -d @rules.json \
     http://coordinator:8081/druid/coordinator/v1/rules/events

# Compact a datasource (one-off)
curl -X POST -H 'Content-Type: application/json' \
     -d @compact-spec.json \
     http://overlord:8081/druid/indexer/v1/task
```

## 11.9 Common operational scenarios

### "Add capacity to the hot tier"

Spin up 3 more Historicals with `druid.server.tier = hot`. Coordinator detects, starts balancing segments.

### "Demote old data from hot to cold"

Update the load rules to shorten the `hot` period (e.g., `P7D` → `P3D`). Coordinator drops segments from hot Historicals on the next cycle; cold tier picks them up.

### "Drop a whole datasource"

Mark all segments unused (`POST .../datasources/events/markUnused`); the kill task eventually purges from deep storage.

Or simply DROP DATASOURCE in SQL (newer versions).

### "Replica count change"

Edit load rules, increase `tieredReplicants`. Coordinator adds replicas on the next cycle.

### "Investigation: why isn't a segment loading?"

- Check Coordinator console → segments → look for "missing replicas".
- Check segment in metadata DB (`SELECT * FROM druid_segments WHERE datasource = ...`).
- Check Historical disk free space.
- Check Coordinator logs for assignment errors.

## 11.10 Anti-patterns

- **Too many small segments** → Coordinator load cycles drag.
- **No auto-compaction** → fragmentation grows; query fan-out cost balloons.
- **No kill tasks** → deep storage fills up with unused segments.
- **Load-forever everything** → Historicals fill up.
- **Tier with one Historical** → no HA; failure = data offline.
- **Coordinator + Overlord on undersized hardware** → control loop falls behind, tasks pile up.

## 11.11 Compare with ClickHouse

| | ClickHouse | Druid |
|--|------------|-------|
| Control process | n/a (each server runs its share) | Coordinator + Overlord (single leader each) |
| Replication coordination | Keeper (Raft) per-table | ZooKeeper + metadata DB |
| Tiering | Storage policies (TTL TO DISK) | Tiered Historicals + load rules |
| Auto compaction | Background merger | Auto-compaction tasks (coordinator-orchestrated) |
| Backup / DR | clickhouse-backup, S3 cross-region | Deep storage replication + metadata-DB backup |
| Re-ingest cost | High (mutations) | None (REPLACE) |

## 11.12 Must-internalize

- Coordinator = segments (load rules, replication, balancing, compaction, kill).
- Overlord = tasks (ingest, MSQ, compaction).
- Load rules are first-match, per-datasource or global, with tier + replica count.
- Tiers = separate fleets of Historicals with different hardware profiles.
- Standard tiers: `hot` / `_default` / `cold`; rules push segments through them as they age.
- Auto-compaction prevents the small-segment problem.
- Web console + REST APIs are the operator's tools.

---

## Sources

- [Coordinator service](https://druid.apache.org/docs/latest/design/coordinator/)
- [Overlord service](https://druid.apache.org/docs/latest/design/overlord/)
- [Load rules](https://druid.apache.org/docs/latest/operations/rule-configuration/)
- [Configuration reference](https://druid.apache.org/docs/latest/configuration/)
- [API reference](https://druid.apache.org/docs/latest/api-reference/coordinator-api/)
