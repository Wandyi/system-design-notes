# Optimizing Postgres Clusters for Reads at MAANG Scale — Production Scenarios and Edge Cases

> A deep dive into the architectural patterns, configuration knobs, replication topologies, caching layers, and failure-mode handling you need to run Postgres at MAANG-level read throughput (millions of QPS, petabyte datasets, multi-region, tight p99 SLOs).

---

## 1. Scale Context

"MAANG-scale reads" here means, concretely:

- **Throughput**: 100K–10M QPS of reads across a single logical dataset.
- **Dataset size**: 10 TB–several PB per logical cluster (sharded if larger than a single primary can hold hot set).
- **Latency SLO**: p50 ≤ 2 ms, p99 ≤ 20 ms for point lookups.
- **Availability**: 99.99%+ across regional brownouts.
- **Workload mix**: typically 95%+ reads, 5% writes. Heavy fan-out on hot keys (user profiles, feed metadata, order status, ad-targeting lookups).
- **Footprint**: tens–hundreds of replicas per shard, per region, fed by logical or streaming replication.

Vanilla Postgres is not natively horizontally read-scalable beyond streaming replicas on one primary. At MAANG scale, the platform is usually a combination of:
- **Streaming/logical replication** fan-out
- **Managed variants** (Aurora Postgres, AlloyDB, Google Spanner-postgres-compat, YugabyteDB, Citus)
- **Connection poolers** (PgBouncer, Odyssey, Pgpool-II)
- **Front-line caches** (Redis, Memcached, tiered CDN caches for API responses)
- **Sharding layer** (application-level, Citus, Vitess-like for PG)

The design problem is *almost never* "can a single Postgres primary serve this read?"; it's *"how do we fan out reads, keep replicas fresh, 
and survive the failure modes that emerge at scale."*

---

## 2. Read-Scaling Architecture Options

### 2.1 Single primary + streaming replicas

```
                  ┌───────── Primary (rw) ─────────┐
                  │                                │
         WAL shipping (async/sync)      Connection: writes only
                  │
      ┌───────────┼────────────┬──────────────┐
      ▼           ▼            ▼              ▼
  Replica1    Replica2     Replica3  …   ReplicaN
   (ro)        (ro)         (ro)           (ro)
      └────────────────────┬───────────────────┘
                           ▼
              HAProxy / PgBouncer / Router
                           │
                      Application (read queries)
```

- Easiest to operate, well-understood.
- Replicas get WAL via streaming; lag typically sub-second under normal load.
- Scales reads until **WAL replay becomes the bottleneck** on replicas (single-threaded recovery process) — usually ~tens of thousands of writes/sec.

### 2.2 Cascading / tiered replication

```
Primary ──► Regional root replica (us-east-1) ──► N local replicas
         ──► Regional root replica (us-west-2) ──► N local replicas
         ──► Regional root replica (eu-west-1) ──► N local replicas
```

- Reduces WAL egress from primary (one stream per region instead of per replica).
- Local reads stay in-region → latency and cost wins.
- Tradeoff: one extra hop of lag per tier.

### 2.3 Logical replication sharding

- Split tables by tenant / hash / range; each shard has its own primary + replica set.
- Application layer or middleware (Citus, PgCat) routes reads to the right shard.
- Horizontal scale for both reads AND writes, at the cost of cross-shard joins being painful.

### 2.4 Aurora / storage-compute separation

- **Aurora Postgres**: decouples compute from storage; a single storage volume shared by up to 15 readers. 
 No WAL replay on replicas — they read from the shared storage page cache directly. 
 This changes the game for read scaling — replicas scale independently of primary write throughput.
- **AlloyDB**: similar, with columnar accelerator for analytical reads.
- **Spanner (PG dialect) / YugabyteDB / CockroachDB**: distributed consensus-based — true horizontal scale, but different consistency/latency profile.

### 2.5 Read-through cache tier (most common at scale)

```
 App ─► Cache (Redis/Memcached cluster) ─► Postgres read replicas (on miss)
```

- 95%+ of production read volume at MAANG is served from caches, not from Postgres directly.
- Postgres becomes the *source of truth* and *repopulation layer* — its read QPS target drops by 1–2 orders of magnitude.

---

## 3. Schema & Index Strategies for Reads

| Strategy | When to use | Cost |
|---|---|---|
| **B-tree on equality/range keys** | Default | Write overhead per index |
| **Covering indexes (INCLUDE)** | Index-only scans for hot paths | Bloat when payload changes |
| **Partial indexes (WHERE ...)** | Hot predicate selects tiny subset | Must match query predicate exactly |
| **BRIN** | Time-series / append-only huge tables | Lossy; only good with natural ordering |
| **GIN / GiST** | JSONB, fulltext, arrays | Larger, slower updates |
| **Bloom** | Multi-column equality, no natural key order | Approximate |
| **Hash** | Strict equality only (rare in practice) | No range |
| **Expression indexes** | Searching on computed values | Must recompute on update |
| **Partitioning (RANGE/LIST/HASH)** | Tables > ~50 GB, especially time-series | Planning cost, attach/detach complexity |
| **Materialized views** | Pre-aggregated rollups, periodic refresh | Staleness, refresh storms |
| **Declarative partitioning + constraint exclusion** | Time-series where queries filter by time | Plan-time overhead |

### 3.1 HOT (Heap-Only Tuple) updates

- When no indexed column changes, Postgres can do in-place update without updating indexes.
- Hugely important for read-heavy workloads: fewer index bloat, fewer buffer-cache invalidations.
- Tuned via `fillfactor` (leave free space in each page) — typical value 80–90 for hot tables.

### 3.2 Visibility map & index-only scans

- Requires `VACUUM` to keep visibility map up to date.
- An index-only scan avoids heap fetches entirely — 10–100× speedup on wide rows.
- At MAANG scale, aggressive autovacuum settings on hot read tables are mandatory.

---

## 4. Connection & Query Management

### 4.1 Connection pooler (non-negotiable at scale)

- **PgBouncer**: transaction pooling mode; 10K client connections → 200 backend connections.
- **Odyssey** (Yandex): multi-threaded PgBouncer alternative.
- **Pgpool-II**: adds load balancing + replica health but heavier.
- **Aurora proxy / RDS proxy**: managed equivalents.

Without a pooler, 10K app workers each opening a persistent connection kills the primary — every connection = one backend process = ~10MB RAM + PG planner state.

### 4.2 Prepared statements

- Reduces parse/plan cost 2–10× on repetitive queries.
- But with transaction-pooled PgBouncer, prepared statements are tricky (stateful per session). Use session pooling or PG 14+ protocol-level prepared statement handling.

### 4.3 Read/write split

- Application or middleware (ProxySQL-for-PG, PgCat, Aurora endpoint) routes:
  - Reads → replica endpoint
  - Writes → primary endpoint
  - Read-your-own-write: pin to primary for ~replica-lag seconds after a write, or use session-consistent routing with LSN-tracking.

### 4.4 Statement timeouts

- Set `statement_timeout` per role (e.g., analytics role = 30s, serving role = 200ms). Prevents a single slow query from choking the replica.

---

## 5. Replication Mechanics You Must Know

### 5.1 Streaming vs logical

| | Streaming | Logical |
|---|---|---|
| What flows | Raw WAL bytes | Row-level change events |
| Filter per table | No | Yes (publish/subscribe) |
| Version compatibility | Must match major | Cross-version ok |
| Conflict handling | N/A (read-only replica) | Required |
| Throughput | High | Lower (decoding overhead) |

At MAANG scale, **streaming for HA replicas**, **logical for downstream systems** (CDC → Kafka → data warehouse).

### 5.2 Synchronous vs asynchronous

- `synchronous_commit=on`, `synchronous_standby_names='N replicas'` → writes wait for confirmation.
- At scale this is usually *quorum* (e.g., 2 of 4 replicas ack) for durability without killing write latency.
- Too many synchronous replicas → write latency = worst replica's latency.

### 5.3 `hot_standby_feedback`

- Replica tells primary about its oldest snapshot → primary won't vacuum rows still needed by replica.
- Prevents query cancellation on replica ("canceling statement due to conflict with recovery").
- But: runaway replica query → bloat on primary.
- At scale, enable for short-lived replicas; disable for long-running analytical replicas.

### 5.4 Replication slot

- Persistent bookmark so primary retains WAL for a replica even if it disconnects.
- **Danger**: if replica goes offline forever, primary WAL grows unboundedly → disk full → outage.
- Always monitor `pg_replication_slots.active + pg_wal_lsn_diff` and set `max_slot_wal_keep_size`.

---

## 6. Production Scenarios at MAANG Scale

### 6.1 User profile read fan-out (Facebook/Meta style)

**Workload**: 10M QPS point reads on `user_profile` by `user_id`.

**Optimization stack**:
1. **Cache tier (Redis)** absorbs ~98% of reads — cache key `profile:{user_id}`, TTL 60s or invalidated on write via CDC.
2. **Shard by user_id** (hash-mod) across 64+ Postgres primaries.
3. Each shard: **1 primary + 5–10 read replicas** behind HAProxy.
4. **Index-only scan**: `CREATE INDEX ON user_profile (user_id) INCLUDE (email, display_name, …)` covers 90% of profile payload.
5. **Covering index with fillfactor 85** so HOT updates work.
6. **Connection pooler** at each shard endpoint.
7. **Consistency**: cache is last-writer-wins, eventual consistency tolerable for profile.

### 6.2 E-commerce catalog reads (Amazon style)

**Workload**: product lookups, search, recommendation lookups; 5M QPS; bursty during events (Prime Day).

**Optimization stack**:
1. **CDN + edge cache** for catalog pages; Postgres serves misses only.
2. **Materialized views** for denormalized product bundles; refreshed via logical replication.
3. **GIN index on JSONB attributes** for faceted filtering.
4. **Partial indexes** on `WHERE active=true` and `WHERE in_stock=true`.
5. **Replica pools tiered**: "hot" replicas with higher shared_buffers for active catalog; "cold" for long-tail queries.
6. **Pre-warmed replicas** brought online before Prime Day via `pg_prewarm` loading top-product pages into buffer cache.

### 6.3 Order history / transactional reads

**Workload**: "show me my last 50 orders" — high-cardinality, low fan-out per user.

**Optimization stack**:
1. **Composite index** `(user_id, created_at DESC) INCLUDE (order_id, status, total)`.
2. **Keyset pagination** not OFFSET — avoids O(N) scan at page N.
3. **Partitioning by range on `created_at`** (monthly partitions), old partitions on cold storage/slower disks.
4. **Read replicas in same region as user** for latency; geographic routing.
5. **Read-your-write**: pin to primary for 5s after a new order, fall back to replica afterwards.

### 6.4 Analytics / reporting on OLTP (ad hoc)

**Workload**: BI queries, 10s–minutes long, full scans, aggregations.

**Optimization stack**:
1. **Dedicated analytics replica** with `hot_standby_feedback=off` and larger `max_standby_streaming_delay` so queries don't get canceled.
2. **Or: logical replication to a columnar store** (AlloyDB columnar, Redshift, BigQuery, Snowflake) — keep analytical reads off OLTP entirely.
3. **`work_mem` tuned per role** — analytics role gets more; serving role gets less.
4. **Per-query `statement_timeout`** caps runaway queries.

### 6.5 Feed / timeline assembly

**Workload**: generate a personalized timeline from N followers' recent posts.

**Optimization stack**:
1. Don't assemble at read time — **pre-compute the timeline on write** (fan-out-on-write) or **hybrid** (fan-out for normal users, fan-out-on-read for celebrities with millions of followers).
2. Postgres stores the canonical posts; the assembled timeline lives in Redis sorted sets.
3. Postgres replica only serves backfill / repair.

### 6.6 Ad-targeting lookups (Google/Meta ads)

**Workload**: sub-10ms lookup of audience membership for millions of ad requests/sec.

**Optimization stack**:
1. **Postgres is the source of truth**; hot data lives in specialized in-memory stores (Bigtable, memcached, Aerospike).
2. **Logical replication → Kafka → materialization jobs** populate the in-memory tier.
3. Postgres reads are for admin UI, repair, audit — low QPS.

### 6.7 Geographic read routing

**Workload**: global users; reads must originate in the same region as the user.

**Optimization stack**:
1. **Region-local replicas** per region; DNS or client-side routing picks local endpoint.
2. **Single global primary** (or per-geography shard primaries).
3. Writes are cross-region (slow); reads are local (fast).
4. For true multi-master writes, use YugabyteDB / Spanner / CockroachDB — native Postgres doesn't do multi-master.

### 6.8 Spike / flash-crowd events

**Workload**: 20× read spike in 2 minutes (launch, viral moment, Black Friday).

**Optimization stack**:
1. **Pre-scale replicas** ahead of scheduled events (known events).
2. **Auto-scale replicas** via autoscaler on CPU + lag; 5–10 min cold-start cost (initial base backup).
3. **Warm pool** of pre-built replicas, kept replicated but not in the LB; flip into LB on demand.
4. **Cache stampede protection**: request coalescing (one miss recomputes; others wait on it).
5. **Graceful degradation**: return stale cache when Postgres saturates.

---

## 7. Edge Scenarios and How to Handle Them

These are the failure modes you will hit at MAANG scale — design for them up front.

### 7.1 Replication lag explosion under bulk writes

**Scenario**: A bulk backfill job writes 10M rows. WAL replay on replicas single-threaded — lag climbs from 100 ms to 10 minutes. Reads from replicas now stale.

**Handling**:
- **Throttle bulk writes**: batch size limits, `COMMIT` every N rows, use `pg_backend_backoff_ms` or application-level pacing.
  - **Parallel WAL apply** (PG 16+ limited support, Aurora has this natively; AlloyDB too).
        - Parallel WAL apply is a PostgreSQL feature (since v14/15+) that accelerates logical replication by using multiple background workers to apply large, 
           in-progress transactions simultaneously on a subscriber. It significantly reduces replication lag by allowing parallel application of changes. 
           By setting streaming = parallel and configuring max_parallel_apply_workers, the subscriber reduces the bottleneck of applying large transactions in a single thread, enhancing performance.
    
          **Key Aspects of Parallel Apply:**
          Logical Replication Speed: It is designed to handle large transactions by not waiting for the final COMMIT message from the publisher to start applying changes.
          Worker Configuration: It is controlled via max_parallel_apply_workers_per_subscription and utilizes max_logical_replication_workers.
          Data Consistency: Although multiple workers apply changes, the order of WAL records for a given transaction or data block is maintained to ensure consistency.
          Limitations: It applies to logical replication, not inherently to crash recovery on physical standby servers, which usually runs single-threaded.
          Implementation:
          To activate this, you must configure the subscription on the subscriber side to handle streaming in parallel, allowing for multiple workers to pick up and apply the incoming logical changes to the database
- **Route reads to primary** for affected tables during bulk window via feature flag.
- **Monitor `pg_last_wal_receive_lsn() - pg_last_wal_replay_lsn()`**; alert at > 1 GB behind.
- **Hot standby feedback off during bulk** (otherwise primary bloats too).

### 7.2 "Canceling statement due to conflict with recovery"

**Scenario**: Long read query on replica while primary VACUUMs old tuples replica still needs. Replica kills the query.

**Handling**:
- `hot_standby_feedback=on` → primary keeps tuples alive. But: primary bloat risk.
- `max_standby_streaming_delay=30s` → replica pauses recovery for up to 30s to let queries finish. But: replica lag.
- **Dedicated analytics replica** where we tolerate lag (high `max_standby_streaming_delay`), serving replicas cancel aggressively (low).
- **Retry the query** on application side — most analytics are idempotent.

### 7.3 Connection storm on failover

**Scenario**: Primary fails. Replica promoted. All app connections reconnect simultaneously → 10K TCP handshakes + auth in < 1s → new primary overloaded before serving a query.

**Handling**:
- **PgBouncer in front** handles the storm; fewer backend connections.
- **Application connection retry with jittered backoff** (not tight loop).
- **DNS TTL tuning** so clients resolve the new endpoint quickly (but not so fast they thrash).
- **Patroni / Stolon / RDS multi-AZ** manages VIP/DNS cutover atomically.
- **Pre-warm** promoted replica if it was running at lower buffer cache utilization.

### 7.4 Hot single row / hot key

**Scenario**: One row (celebrity profile, popular product) receives 1M reads/sec; single replica is CPU-saturated on a buffer lock.

**Handling**:
- **Cache tier handles it** — this row should be in Redis, single-digit-ms lookup.
- **`pg_prewarm`** keeps it in buffer cache.-->
  -     pg_prewarm is a PostgreSQL extension that loads table or index data into memory (OS cache or PostgreSQL shared buffers) before 
    -   queries run, reducing cold-start latency and expensive disk I/O. It is particularly effective for optimizing performance immediately after server restarts or reboots, mitigating the slowness of initial queries.
      - SELECT pg_prewarm('my_table'); SELECT pg_prewarm('my_table_idx');
- **Read-only copies across all replicas** — LB distributes; no single replica hot.
- **Row-level sharding** (app-level): duplicate the hot row to N "shadow" rows, read from a random one; invalidate all on write.
- If the row's columns are read-mostly, **materialize it into app memory** (sidecar cache updated via CDC).

### 7.5 Replica lag → stale reads → user complaint

**Scenario**: User places order, refreshes, doesn't see it (order write went to primary, read went to lagging replica).

**Handling**:
- **Read-your-writes consistency**: after a write, pin that session to primary for `replica_lag + safety_margin` seconds.
  - **LSN tracking**: on write, record LSN in session; on read, require replica.lsn ≥ session.lsn or fall back to primary.
            ```
            LSN (Log Sequence Number) tracking is a technique used in PostgreSQL to achieve read-your-writes consistency while utilizing 
            asynchronous read replicas. It ensures that a user does not experience "data vanishing" after an update due to replication lag. 
            
            Core Mechanism Components
            Write Log Sequence Number (LSN): A 64-bit integer acting as a unique, monotonically increasing identifier for a write operation in the Write-Ahead Log (WAL).
            Replay LSN: The current point up to which a replica has applied changes from the primary.
            Session Management: Storing the LSN of a user’s last write (usually in a shared cache like Redis with a Time-To-Live). 
           
            The LSN Tracking Algorithm
            On Write: The application writes data to the primary, captures the LSN of that commit, and records it in the user’s session state.
            On Read: Before serving a read from a replica, the application compares the required LSN from the session with the replica's current replay LSN.
            Validation:
            If replica.lsn 
             session.lsn: The replica is caught up and safe to read from.
            If replica.lsn 
             session.lsn: The replica is lagging. The query is routed to the primary (or waits for the replica to catch up)
            ```
- **Cache the just-written value client-side** and return it directly until confirmed on replica.
- **Session cookies** containing LSN hint used by routing layer.

### 7.6 Vacuum / autovacuum lagging

**Scenario**: Write-heavy table; autovacuum can't keep up; dead tuples accumulate; index bloat; query plans degrade; replicas bloat too.

**Handling**:
- **Tune autovacuum aggressively**: lower `autovacuum_vacuum_scale_factor` (e.g., 0.02), increase `autovacuum_max_workers`, `autovacuum_vacuum_cost_limit`.
- **Per-table autovacuum settings** on hot tables.
- **pg_repack** for online rebuild of bloated tables without long locks.
- **Partition the table** — old partitions don't need frequent vacuum; drop aged partitions instead of DELETE.
- **Avoid long-running transactions** (they hold back `xmin` and block vacuum globally). Monitor `pg_stat_activity.xact_start`.

### 7.7 Long-running transaction blocks vacuum globally

**Scenario**: An analyst runs `SELECT ...` for 4 hours on a replica with `hot_standby_feedback=on`. 
              Primary can't vacuum anything newer than the oldest xmin on that replica → fleet-wide bloat.

**Handling**:
- **Monitor oldest xmin** per replica; alert at > 15 min.
- **`idle_in_transaction_session_timeout`** terminates idle transactions.
- **`statement_timeout`** per role.
  - **Separate analytical replica** with `hot_standby_feedback=off`; accept that long queries may be canceled but primary stays healthy.
    - Setting hot_standby_feedback = off (default) allows a PostgreSQL primary server to VACUUM dead rows without waiting for standby queries, 
      preventing primary table bloat. However, this risks "replication conflicts," where long-running queries on the standby are canceled if they try to read rows removed by the primary.
      
    - Key Implications of hot_standby_feedback = off:
      Reduced Bloat on Primary: The primary does not hold onto rows needed by the standby, keeping the database cleaner.
      Query Cancellations on Standby: Active queries on the standby may fail with "conflict with vacuum" errors if they take too long.
      Best for: Systems where primary performance is critical and standby queries are short-lived.
      Use on for: Long-running reporting queries on the standby to stop them from failing, at the risk of higher bloat on the primary.

      Alternative Remedies:
      If you keep it off but encounter errors, consider raising max_standby_streaming_delay to allow queries more time to finish before being killed.
- **Kill the offending session**: `pg_terminate_backend(pid)` if emergency.

### 7.8 Replication slot disk full

**Scenario**: Logical replication consumer (CDC) dies; slot retains WAL; primary disk fills.

**Handling**:
- **`max_slot_wal_keep_size`** (PG13+): primary auto-invalidates slot if it would exceed this limit, preventing outage. Consumer must re-snapshot.
- **Monitor slot lag** via `pg_replication_slots`.
- **Dead-man switch**: auto-drop slot if consumer offline > N hours (tradeoff: CDC has to re-init).
- **Separate WAL volume** so slot bloat doesn't fill the data disk.

### 7.9 Plan regression on replica

**Scenario**: Replica has slightly different statistics from primary (`ANALYZE` runs independently). A query picks a different plan on replica and is 100× slower.

**Handling**:
- **`pg_stat_statements`** on each replica to find regressed queries.
- **Plan freezing**: extensions like `pg_plan_filter` / `pg_hint_plan` / **generic plans** via `plan_cache_mode=force_generic_plan`.
- **Synchronize statistics** — primary's statistics aren't replicated via streaming by default; replicas build their own during `ANALYZE`. Consider `pg_dump --statistics-only` or `pg_statistic` copy hack.
- **Auto-explain + slow query log** to catch regressions early.

### 7.10 Subtransaction overflow → cache thrash

**Scenario**: A transaction uses thousands of SAVEPOINTs (e.g., due to ORMs like Hibernate). Subtransaction cache overflows 64 entries → full scan of pg_subtrans on every snapshot acquisition → replicas grind.

**Handling**:
- **Limit savepoints in ORMs** — configure Hibernate/ActiveRecord to not wrap each statement in a savepoint.
- **Monitor `subtransaction_overflow`** metric (PG 14+).
- **Batch updates** into a single transaction without sub-savepoints.

### 7.11 Snapshot too old / MVCC bloat on long queries

**Scenario**: A replica query runs long; primary has moved on; replica's snapshot is no longer valid; query fails with "snapshot too old."

**Handling**:
- **`old_snapshot_threshold`** per cluster — balance between allowing long queries and controlling bloat.
- **Offload analytical reads** to a separate pipeline (CDC → warehouse).
- **Retry with fresh snapshot** on transient failure; paginate large scans.

### 7.12 Checkpoint spike → replica stutter

**Scenario**: Primary checkpoint flushes 4 GB of dirty pages. WAL spike propagates. Replicas stall replay; read latency p99 spikes 10×.

**Handling**:
- **Spread checkpoint**: `checkpoint_completion_target=0.9` and longer `checkpoint_timeout` (15+ min).
- **`max_wal_size`** large enough so WAL-triggered checkpoint is rare.
- **Dedicated WAL disk** with high IOPS; replicas have plenty of disk bandwidth to apply.

### 7.13 Replica cold start

**Scenario**: A brand-new replica is promoted into LB. Buffer cache empty. First reads hit disk. p99 is seconds, not milliseconds.

**Handling**:
- **Pre-warm** using `pg_prewarm` with a snapshot of primary's top hot pages (captured via extension `pg_buffercache`).
- **Gradual traffic ramp** via weighted LB — 1% → 10% → 100% over minutes.
- **Storage-local caching** (Aurora uses shared buffer pool at storage layer — less cold-start pain).

### 7.14 Foreign key / trigger cascade under load

**Scenario**: A single user deletion cascades through 20 FKs; triggers run; replicas receive 20× write amplification; replay lags.

**Handling**:
- **Soft-delete** instead of cascade delete.
- **Archive + batch-purge** old rows in off-hours.
- **Remove expensive triggers** from hot write path; move logic to async consumers.

### 7.15 pg_wal runaway on primary

**Scenario**: Async replica disconnected; WAL not archived; `pg_wal` fills.

**Handling**:
- **`max_slot_wal_keep_size`** (see 7.8).
- **Archive to S3** (`archive_command`) — primary can purge WAL after archive success.
- **Monitor `pg_wal` disk use** at 50% / 70% / 85%.
- **Pre-provisioned headroom** (2× expected peak WAL generation for 4 hours).

### 7.16 Failover while replica has uncommitted split-brain

**Scenario**: Network partition. Old primary keeps taking writes. New primary promoted. Both have writes that diverge.

**Handling**:
- **Fencing**: Patroni/Stolon uses DCS (etcd/Consul) with leader lease. Losing leader = stops accepting writes.
- **`synchronous_commit=remote_apply`** + `synchronous_standby_names` so writes require sync replica before acking — no data loss on failover.
- **Automated STONITH** (Shoot The Other Node In The Head) — kill the old primary instance from the orchestrator.

### 7.17 Stale DNS / client connection to old primary

**Scenario**: Failover happened; client DNS cache still points to old primary (now read-only). Writes fail mysteriously.

**Handling**:
- **Service discovery via health-checking proxy** (HAProxy w/ Consul-template, Patroni REST API for leader discovery, RDS endpoint).
- **Short DNS TTL** (5–30s) for DB endpoints.
- **Client driver awareness** — pgjdbc supports `targetServerType=primary` and retries on the next host in the list.
- **Feature flag** to dual-write / fall back during a cutover window.

### 7.18 Per-query memory explosion

**Scenario**: `work_mem=256MB` × 500 concurrent queries × 3 sort nodes per query = 384 GB — OOM.

**Handling**:
- **Tune `work_mem` low globally**, **raise per-role or per-session** for analytical work.
- **Connection pooler caps concurrent backends** at N, making memory accounting tractable.
- **`hash_mem_multiplier`** (PG13+) gives hash joins more mem without affecting sort.
- **cgroups / container memory limit** with headroom.

### 7.19 Idle-in-transaction sessions

**Scenario**: App bug — connection goes idle inside a transaction. Holds snapshot, blocks vacuum, holds row locks.

**Handling**:
- **`idle_in_transaction_session_timeout=60s`** kills them.
- **PgBouncer transaction pooling** naturally returns connections fast.
- **Application retry logic** when receiving `57014 transaction terminated`.

### 7.20 TOAST / wide-row reads

**Scenario**: A row contains a large JSONB blob (several MB). Every read fetches the TOAST pages → IO amplification.

**Handling**:
- **Move large blob to S3**, store only URL in Postgres.
- **Compress TOAST** (`default_toast_compression=lz4` in PG 14+).
- **Don't select large columns** unless needed — write narrow SELECTs.
- **Separate table** for blob columns, joined on demand.

### 7.21 Observability flood

**Scenario**: `pg_stat_statements` on 100 replicas × 10K distinct queries × metrics scrape every 10s = huge cardinality.

**Handling**:
- **Normalize and pre-aggregate** at replica before shipping metrics.
- **Use `pg_stat_statements.max` (default 5000)** to cap cardinality.
- **Tag by shard/cluster, not per-replica**, for most metrics.
- **Sampled slow-query log** — don't log every query, log p95+ only.

### 7.22 Backup / base backup interfering with reads

**Scenario**: `pg_basebackup` from primary saturates IO; read latency spikes.

**Handling**:
- **Base backup from replica** (lower-priority replica), not primary.
- **Incremental / continuous backup** (WAL-G, pgBackRest) — avoids full-copy bursts.
- **QoS on IO** via cgroups, so backup doesn't starve reads.
- **Dedicated backup replica**, not a serving replica.

### 7.23 Extension incompatibility on failover

**Scenario**: Replica was missing an extension version; failover happens; queries fail.

**Handling**:
- **Standardize image/container** across all nodes — same extensions installed, same versions.
- **CI check** that replica and primary extension lists match.
- **Replay WAL before cutover** — stop writes briefly, let replica catch up, promote.

### 7.24 Security / audit reads on hot path

**Scenario**: Compliance team enables row-level security policies + pgaudit + SSL on every connection. Query latency doubles.

**Handling**:
- **Defer audit logging** to background worker, not synchronous write.
- **Cache RLS-evaluated predicates** where safe.
- **Mutual TLS session reuse** — SSL handshakes are expensive; pool connections keep sessions alive.
- **Per-role audit policy** — don't audit high-volume read roles if they're read-only.

---

## 8. Configuration Cheat Sheet (MAANG-Scale Starting Points)

These are *starting points* for a tuned, high-end primary or replica (e.g., 96-core, 768 GB RAM, NVMe). Tune per workload.

| Parameter | Value | Why |
|---|---|---|
| `shared_buffers` | 192 GB (25% RAM) | Postgres buffer cache |
| `effective_cache_size` | 576 GB (75% RAM) | Planner hint for OS cache |
| `work_mem` | 32–64 MB | Per sort/hash operation |
| `maintenance_work_mem` | 2 GB | VACUUM/CREATE INDEX |
| `max_connections` | 1000 (with PgBouncer) | Pooler does the multiplexing |
| `checkpoint_timeout` | 15min | Spread checkpoint |
| `checkpoint_completion_target` | 0.9 | Smooth IO |
| `max_wal_size` | 64 GB | Delay forced checkpoints |
| `wal_compression` | on | Save bandwidth |
| `wal_level` | logical | Enables both streaming and logical |
| `synchronous_commit` | on (local) / remote_apply (cross-AZ) | Durability vs latency |
| `hot_standby_feedback` | on (serving) / off (analytical) | See §7.2/7.7 |
| `max_standby_streaming_delay` | 30s (serving) / 30min (analytical) | Read vs lag tradeoff |
| `autovacuum_vacuum_scale_factor` | 0.02 | Aggressive vacuum for hot tables |
| `autovacuum_max_workers` | 8 | Parallelism |
| `default_toast_compression` | lz4 | Cheaper than pglz |
| `random_page_cost` | 1.1 | NVMe — near-equal to seq |
| `effective_io_concurrency` | 200 | NVMe parallel IO |
| `idle_in_transaction_session_timeout` | 60s | See §7.19 |
| `statement_timeout` | role-dependent | See §4.4 |
| `track_io_timing` | on | Observability |
| `log_min_duration_statement` | 100ms | Catch slow queries |

---

## 9. Decision Table — Read Optimization by Symptom

| Symptom | First lever | Second lever | Heavy-hammer |
|---|---|---|---|
| p99 read latency up | Check plan regression, buffer cache hit | Add replica, re-balance LB | Pre-warm, scale vertically |
| Replica lag rising | Throttle writes, parallel apply | Tier replicas (cascading) | Move to Aurora / storage-separated engine |
| CPU saturated on primary | Read/write split, pooler | Add replicas | Shard |
| Hot key | Cache tier | Shadow rows across replicas | Dedicated in-mem tier |
| Vacuum backlog | Tune autovacuum, partition | `pg_repack` | Archive old data |
| Bloat on wide row | Narrow SELECTs, TOAST compression | Split blob to separate table | Offload blob to S3 |
| Stale reads | Read-your-write pin | LSN-aware routing | Synchronous replicas |
| Failover storm | PgBouncer, jittered retry | Warm standby | Multi-primary (Yugabyte/Spanner) |
| Plan regression | `pg_hint_plan`, plan_cache_mode | ANALYZE tuning | Materialized view |
| Analytics hurting OLTP | Dedicated analytical replica | CDC to warehouse | Columnar engine (AlloyDB) |

---

## 10. Key Takeaways

1. **At MAANG scale, Postgres is rarely the primary read surface** — it's the *source of truth* behind a cache tier and a CDC pipeline feeding materializations. Design for that topology.
2. **Replication lag, vacuum health, and connection management are the three dominant operational concerns.** Tune for them before micro-optimizing queries.
3. **Sharding is inevitable beyond a few TB of hot data / tens of thousands of write TPS.** Choose: app-level, Citus, or a distributed-SQL engine. Each has tradeoffs.
4. **Failure modes compound.** A lag spike + autovacuum falling behind + long analytical query = cascading bloat + stale reads + query cancellations. Detect early, isolate by workload class (serving replicas vs. analytical replicas).
5. **Aurora/AlloyDB storage-compute separation is a qualitative change** — read scaling decouples from write throughput. If that fits your workload and lock-in tolerance, it eliminates whole classes of edge cases (WAL replay bottleneck, cold-start base backup).
6. **Observability-first operation**: `pg_stat_statements`, `pg_stat_activity`, `pg_stat_replication`, `pg_buffercache`, `pg_stat_user_tables` are your oracles. Wire them into Prometheus/Atlas from day one.
7. **Consistency is a product decision, not a database decision.** Read-your-writes, eventual, session, bounded-staleness — choose deliberately per endpoint, and build the routing / caching / LSN-tracking to enforce it.

---

## 11. Further Reading

- Postgres Wiki: "Tuning Your PostgreSQL Server", "Hot Standby"
- Aurora Postgres architecture whitepapers (AWS)
- "The Internals of PostgreSQL" — Hironobu Suzuki
- Citus documentation (distributed Postgres)
- YugabyteDB, CockroachDB architecture docs (distributed SQL comparisons)
- pgBouncer / Odyssey / Patroni operator docs
- MAANG engineering blogs: Facebook's UDB/MyRocks (MySQL, but analogous read-scale patterns), Uber's Schemaless, Netflix's NDBench, Instagram's Django-on-Postgres scaling story