# 2 · Architecture Fundamentals

This is the most important file in the pack. Druid's six-process model is the single biggest mental load when learning Druid and the most common interview probe.

Compare-and-contrast with ClickHouse: where CH has **one** `clickhouse-server` process and pushes everything onto MergeTree, Druid has **six** different process types each doing one job — coordinating, ingesting, serving, routing — and a strong split between **control plane** (Coordinator/Overlord), **data plane** (Historical/MiddleManager-Indexer), and **query plane** (Broker/Router).

## 2.1 The mental model

```
                       External clients (BI tools, app servers)
                                       │
                                       ▼
                            ┌──────────────────┐
                            │     Router       │ ← optional API gateway
                            └────┬─────┬──┬────┘
                                 │     │  │
                          ┌──────┘     │  └─────────┐
                          ▼            ▼            ▼
                   ┌──────────┐  ┌──────────┐  ┌──────────┐
                   │  Broker  │  │Coordinator│ │ Overlord │
                   └────┬─────┘  └────┬──────┘ └────┬─────┘
                        │             │             │
            ┌───────────┴────────┐    │             │
            ▼                    ▼    ▼             ▼
      ┌──────────┐         ┌────────────────┐ ┌────────────────────┐
      │Historical│  ...    │ MiddleManager  │ │ MiddleManager      │
      │ (data)   │         │   + Peons      │ │   + Peons          │
      │          │         │ (ingest tasks) │ │  (ingest tasks)    │
      └────┬─────┘         └────┬───────────┘ └─────┬──────────────┘
           │                    │                   │
           │  (fetch segments)  │  (publish segments)
           │                    ▼                   │
           │            ┌──────────────────┐        │
           └───────────►│   Deep Storage   │◄───────┘
                        │  (S3 / HDFS /    │
                        │   GCS / Azure)   │
                        └──────────────────┘
                                ▲
                                │
                        ┌───────┴────────┐
                        │  Metadata DB   │  ← schema, segment table, supervisor state
                        │ (MySQL/Postgres)│
                        └────────────────┘
                                ▲
                                │
                        ┌───────┴────────┐
                        │   ZooKeeper    │ ← cluster state, leader election, segment announcements
                        └────────────────┘
```

Three big ideas:

1. **Role separation** — each process type does one job, scales independently.
2. **Time-partitioned, immutable segments** are the unit of data. Once published, they don't change.
3. **Deep storage is the source of truth.** Historicals are just a cache of segments. Compute and storage are *physically* separable.

## 2.2 The six processes — what each one does

### Coordinator (Master tier)

- Decides **which segments go on which Historical nodes** and how many replicas.
- Runs **load rules** and **drop rules** (retention).
- Issues **compaction tasks** (auto-compaction is its job).
- Issues **kill tasks** to physically delete segments from deep storage.
- Reads from **metadata DB** (the segment table) and **ZooKeeper** (current cluster state).
- Single active leader (others are standby).

### Overlord (Master tier)

- Manages **ingestion tasks** — knows what's pending, assigns to MiddleManagers/Indexers.
- Tracks task status (RUNNING / SUCCESS / FAILED / WAITING).
- Manages **supervisors** for streaming ingest (Kafka/Kinesis).
- Single active leader.

**Often the Coordinator and Overlord run combined** (`druid.coordinator.asOverlord.enabled = true`). For very large clusters they're split.

### Broker (Query tier)

- Receives the user's query.
- Identifies **which segments are needed** by matching query time range against segment metadata.
- Fans out **subqueries to Historicals + indexing tasks** that own the relevant segments.
- **Merges results** and returns to the user.
- Holds **lookup cache** (preloaded lookups, used directly without join).
- Holds **broker cache** (per-segment query results) when enabled.

### Historical (Data tier)

- **Downloads segments from deep storage** to local disk and serves queries against them.
- **No mutation** — they only download (and unload, when the Coordinator says to).
- Run **scan / aggregate** operators on segments, in parallel.
- Tiered: you can have a `_default` tier and additional tiers like `hot`, `cold` with different specs.

### MiddleManager + Peons / Indexer (Data tier — ingest)

- **MiddleManager** is a parent process. For each ingestion task, it spawns a **Peon** (a separate JVM) that runs the task.
- **Indexer** is the alternative: tasks run as threads inside one Indexer JVM (less isolation, less overhead).
- Tasks build segments **incrementally in memory**, **persist as intermediate segments to local disk**, then **publish final segments to deep storage** (and notify the Coordinator via the metadata DB).

You pick **MiddleManager+Peons** for isolation (one task crash doesn't take others down), **Indexer** for resource efficiency.

### Router (Query tier, optional)

- API gateway in front of everything. Routes requests to Broker / Coordinator / Overlord by URL.
- Useful for: serving the Web Console (built-in UI), TLS termination, simple authn proxying.
- Optional — you can hit Brokers directly.

## 2.3 The three external dependencies

### Deep storage

- **Where the canonical, immutable segments live.** S3 (most common), GCS, Azure Blob, HDFS, local FS (dev only).
- Segments are pushed here by ingest tasks; Historicals download from here; Coordinator's kill tasks delete from here.
- **This is your durability** — lose all Historicals, all segments are still on S3. Spin up new Historicals; they re-download what they own.

### Metadata DB

- **Where Druid keeps its state** — datasource list, segment table (every published segment with its interval + version + size), supervisor state, task history, rules, lookups.
- MySQL or Postgres. **Highly available** (failover is your responsibility).
- **The single point of state.** Backup it. Treat it like a database.

### ZooKeeper

- **Where live cluster state lives** — which Historicals are up, which segments each is serving, which task slot is available on which MiddleManager, leader election for Coordinator/Overlord.
- Druid is moving toward **embedded coordination via Druid Coordinator Cluster** in newer versions, but ZooKeeper is still the production default at the time of writing.
- Typically 3-node or 5-node ZK ensemble.

## 2.4 The segment lifecycle (the single most useful diagram in Druid)

```
1. Ingestion task running on MiddleManager / Indexer
        │
        │  (builds in-memory segment / IncrementalIndex)
        ▼
2. Incremental segments persisted to local disk (per task)
        │
        │  (task finishes / time chunk closes)
        ▼
3. Final segment written to deep storage
        │
        │  (task publishes via metadata DB)
        ▼
4. Coordinator notices new segment in metadata DB
        │
        │  (assigns to Historical(s) per load rules)
        ▼
5. Historical downloads segment from deep storage to local disk
        │
        │  (announces via ZooKeeper)
        ▼
6. Broker learns segment is available and routes queries to that Historical
```

Each step is asynchronous; the system survives any single failure.

## 2.5 Querying — what happens on a SELECT

```
SELECT user_country, count(*) FROM events WHERE ts BETWEEN '2026-01-01' AND '2026-01-02';

Step 1: Broker receives query
Step 2: Broker queries metadata cache → finds segments matching time interval
        (for events datasource, interval 2026-01-01..02)
Step 3: Broker discovers (via ZK) which Historicals serve each needed segment
        (and which MiddleManager Peons serve the currently-ingesting segments)
Step 4: Broker fans out subqueries to each node, in parallel
Step 5: Each Historical / Peon scans the segments it owns, returns partial results
Step 6: Broker merges and returns final result
```

The Broker is the bottleneck for cross-segment merge; Historicals are CPU/IO-bound on scan.

## 2.6 Time partitioning is mandatory

Every segment has a **time interval**. Every datasource is partitioned by time first. You can't *not* time-partition. (Compare: in ClickHouse partitioning is optional, even if PARTITION BY a time column is almost always the right choice.)

Time partition granularity is set per ingestion: `segmentGranularity = HOUR | DAY | MONTH | YEAR`.

Within a time interval, segments can be further partitioned via **shard specs**:
- **linear** — append-only, one shard per task.
- **hashed** — hash partition on a list of dimensions (default for batch).
- **range** — range partition on a single dimension (good for high-cardinality dimensions you query on equality).
- **single** — one segment per interval.

## 2.7 Query granularity vs segment granularity

A subtle distinction Druid newcomers miss:

- **segmentGranularity**: the time interval each segment covers. (E.g., one segment per day.)
- **queryGranularity**: the time precision rollup uses. (E.g., all events within the same minute roll up.)

These are independent. A common config: `segmentGranularity = DAY, queryGranularity = MINUTE` — one segment per day, with all events within a minute pre-aggregated into one row.

## 2.8 Tiering — hot/cold/archive

Coordinator load rules let you assign segments to **tiers** of Historicals:
- `hot` (NVMe-backed) for last 7 days, 2 replicas.
- `default` (SSD) for last 30 days, 1 replica.
- `cold` (HDD/large) for last 365 days, 1 replica.
- After 365 days: no Historical (segment lives only in deep storage and won't be queried unless re-pulled).

Compare to ClickHouse: CH does tiering via storage policies (TTL TO DISK 'cold') *within* a single replica's storage; Druid does it across *separate node tiers* with separate hardware.

## 2.9 Replication

Replication is per-tier in the load rules: "load 2 replicas to `hot`, 1 to `default`". The Coordinator ensures the desired replica count.

This is **at the segment level**, not at the cluster level. Different datasources or different time ranges can have different replication.

## 2.10 What goes where (the cheat-sheet for first-time operators)

| State | Lives in |
|-------|----------|
| Canonical segment data | Deep storage |
| Per-Historical segment cache | Local disk on Historical |
| Segment metadata (interval, version, size, datasource) | Metadata DB (segment table) |
| Schema | Metadata DB |
| Datasource list | Metadata DB |
| Supervisor configs | Metadata DB |
| Task history | Metadata DB |
| Current Historical assignments | ZooKeeper |
| Leader election | ZooKeeper |
| Available task slots | ZooKeeper |
| Streaming consumer offsets | Druid's own (via supervisor + metadata DB) for exactly-once |
| Lookups | Metadata DB + cached in Brokers |

## 2.11 Compare-and-contrast with ClickHouse

| Aspect | ClickHouse | Druid |
|--------|------------|-------|
| # of process types | 1 | 6 |
| Storage unit | MergeTree part | Segment |
| Sort/index | Sparse primary index on sort key | Bitmap index per dimension + dictionary encoding |
| Time partition | Optional (recommended) | **Mandatory** |
| Mutations | ALTER UPDATE/DELETE + lightweight delete | None — re-ingest |
| Replication coordination | Keeper (Raft) | ZooKeeper + metadata DB |
| Deep storage | Optional (Cloud uses S3) | **Mandatory** — always S3/HDFS/GCS/Azure |
| Joins | 6 algorithms incl. shuffle | Broadcast hash only |
| SQL | Full (CH-flavored) | SQL via Calcite, translates to native query types |
| Cluster scaling | Add shards (manual rebalance); SharedMergeTree fixes for cloud | Add Historicals (auto-balance via Coordinator) |
| Streaming exactly-once | Via Kafka engine + dedup | Native via supervisor + metadata DB |
| Rollup at ingest | Via AggregatingMergeTree + MV | First-class native feature |

For an analytics product picking between them, the key question: **do you need streaming exactly-once + user-facing high-concurrency dashboards (Druid wins) or full SQL + simpler ops + ad-hoc queries (ClickHouse wins)?**

## 2.12 Sizing numbers to anchor on

- Segment size sweet spot: **300-700 MB** compressed.
- Rows per segment: **5-15 million** (depends on roll-up + columns).
- Default `index_granularity` (Druid-speak: the bitmap index spans the whole segment; granule concept doesn't apply the same way as in CH).
- A Historical node typically holds **dozens of TB** across thousands of segments on local SSD/NVMe.
- A Broker can fan out to **hundreds of Historicals** per query.
- Per-segment scan throughput: **hundreds of millions of rows/sec/core** on cached data.

## 2.13 Common ops gotchas (foreshadowing [15-anti-patterns.md](15-anti-patterns.md))

- **Tiny segments** (< 100 MB) — too many, Broker fan-out cost dominates.
- **Huge segments** (> 1 GB) — memory pressure on Historicals during scan, slow segment download.
- **Metadata DB is the SPOF you forget about** — back it up, monitor it.
- **ZK lag** under high write rate → segment announcements delayed → Brokers don't see new data.
- **Deep storage cost** grows monotonically; without kill tasks, S3 fills with unused segment versions.

## 2.14 Must-internalize

- **6 processes**: Coordinator (segments), Overlord (tasks), Broker (queries), Historical (serves), MiddleManager/Indexer (ingests), Router (gateway).
- **3 dependencies**: deep storage (canonical), metadata DB (state), ZooKeeper (coordination).
- **Immutable segments**, time-partitioned, replicated across tiers per Coordinator rules.
- **Deep storage is the source of truth**; Historicals are a cache.
- **Segment lifecycle**: task builds → persists to deep storage → metadata published → Coordinator assigns → Historical downloads → Broker queries.
- Time partitioning is **mandatory**; `segmentGranularity` vs `queryGranularity` are independent.
- No UPDATE/DELETE; data fixes via re-ingest.
- Joins are broadcast-only (no shuffle).

---

## Sources

- [Architecture (latest)](https://druid.apache.org/docs/latest/design/architecture/)
- [Processes and servers](https://druid.apache.org/docs/latest/design/processes.html)
- [Segments](https://druid.apache.org/docs/latest/design/segments/)
- [Coordinator service](https://druid.apache.org/docs/latest/design/coordinator/)
- [Overlord service](https://druid.apache.org/docs/latest/design/overlord/)
- [Broker service](https://druid.apache.org/docs/latest/design/broker/)
- [Historical service](https://druid.apache.org/docs/latest/design/historical/)
- [Seattle Data Guy — Druid architecture overview](https://www.theseattledataguy.com/apache-druids-architecture-how-druid-processes-data-in-real-time-at-scale/)
