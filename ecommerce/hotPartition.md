# Hot Partition in the Orders Table — Deep Dive

> Orders are partitioned by `created_date`. On Black Friday, all orders go to today's partition. That single partition handles 100x normal write load. The other 364 partitions are idle.

The framing of "hot partition" sounds like a database-administration problem. It is not. It is a **partition-scheme design problem, downstream of a read-vs-write 
workload conflict that was baked in the day someone chose `PARTITION BY RANGE (created_date)` without asking how writes would be distributed.** 
Fixing it requires either accepting the read regression to spread writes, or adding a second dimension so both paths work.

The naive fixes — "just add more hardware," "increase connection pool," "scale up the primary" — don't work. The bottleneck isn't the machine; it's the serialization 
point: one partition, one WAL stream, one set of indexes, one autovacuum process. You cannot write-scale a single row-structure on a single node past its per-node IOPS 
and lock-wait ceiling, no matter how much RAM you throw at it.

---

## 1. Why Time-Partitioning Creates The Problem

Partitioning is two different optimizations wearing the same name.

**For reads:** partitioning lets the planner prune. A query `WHERE created_date BETWEEN '2026-03-01' AND '2026-03-31'` touches one month's partitions, not 12 years of history. 
Fewer rows scanned, fewer index pages, less memory. Analytics queries want this.

**For writes:** partitioning lets you spread work across different physical resources — different disks, different nodes, different locks. 
Uniformly-distributed writes over N partitions mean each partition sees 1/N of the load.

These two goals conflict in the axis you pick.

```
PARTITION BY RANGE (created_date)
  ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┐
  │ Jan │ Feb │ Mar │ Apr │ May │ Jun │ ... │
  └─────┴─────┴─────┴─────┴─────┴─────┴─────┘
   ↑                                   ↑
   idle                               100% of writes

  Read "this month's orders"  →  1 partition   ← planner-friendly
  Write an order              →  1 partition   ← hotspot
```

The time axis is naturally skewed: every new order lands on "today." 364/365 of the partitions are cold. One is molten.

```
PARTITION BY HASH (order_id)   -- or (customer_id)
  ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
  │  0  │  1  │  2  │  3  │  4  │  5  │  6  │  7  │
  └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
   ↑     ↑     ↑     ↑     ↑     ↑     ↑     ↑
   even load distribution

  Read "this month's orders"  →  ALL partitions   ← no pruning
  Write an order              →  ANY partition    ← well-distributed
```

Hash distribution fixes writes, breaks the analytics query pattern. Pick either, and you break the other. That's the underlying problem — any real solution addresses *both* axes simultaneously.

---

## 2. What "100x Writes" Actually Breaks

On the hot partition, at 100x write rate, the failure modes surface in this order:

### 2.1 WAL pressure

Every `INSERT` writes to WAL. The hot partition's INSERTs produce WAL that must be flushed to disk before the commit can acknowledge. 
At 10K writes/sec on a partition with 5 indexes, that's ~60K WAL records per second (row insert + index updates + toast pointers). 
On a single SSD with ~30K IOPS, you hit wal_sync throughput limits before CPU.

Mitigations: `synchronous_commit = off` (lose durability guarantee), `wal_compression = on` (CPU for IO), `commit_delay` / group commit, put WAL on faster disk. None eliminate the bottleneck; they push it back by 2-5x.

### 2.2 Index B-tree contention 
//TODO understand in detail 

Sequential order_id primary key: every INSERT hits the rightmost leaf of the B-tree. At high concurrency, writers serialize on the leaf's latch. 
Classic "right-edge index" contention. Postgres 12+ has deduplication that helps somewhat; version 13's btree improvements help more; none change the fundamental topology of a B-tree with monotonic keys.

Fixes:
- **Snowflake-shaped IDs** (epoch-ms + shard + sequence). Still monotonic globally but distributed across multiple index roots if you shard.
- **UUIDv7** (time-ordered UUID). Monotonic enough for cache locality, random enough that multiple inserters don't fight the same leaf page (with 2^62 bits of random, 10K concurrent writers rarely collide on a leaf).
- **Reverse-byte hash key.** Convert sequential key to random via bit reversal. Distributes leaf writes. Kills range-scan performance on the key, so only do this when range-scanning the PK is not a hot path (usually true for orders — you scan by `created_date`, not by `order_id`).

### 2.3 Heap tuple locking

Order insert triggers update to inventory (decrement stock). That inventory row is hot — every order for the same SKU touches it. Lock contention on the inventory row becomes the bottleneck, not the orders partition itself.

This is often misdiagnosed as a partitioning problem when it's actually a hot-row problem. Fix at the inventory layer (see `variantExplosion.md §4` on sharded inventory / event-sourced counters) — partitioning won't help.

### 2.4 Autovacuum can't keep up

At 100x write rate, dead tuples accumulate (from updates to status fields, or HOT-update failures). Autovacuum on the hot partition runs continuously; it falls behind; 
table bloat grows; index bloat grows; query planner stats go stale. A week of Black Friday volume can bloat the partition to 3-5x its logical size.

Per-partition autovacuum tuning is essential:
```sql
ALTER TABLE orders_2026_11_27 SET (
    autovacuum_vacuum_scale_factor = 0.01,   -- vacuum much more aggressively
    autovacuum_analyze_scale_factor = 0.02,
    autovacuum_vacuum_cost_delay = 0,        -- don't throttle
    autovacuum_vacuum_cost_limit = 10000
);
```

### 2.5 Replication lag

Every WAL record must be shipped to replicas. If the primary is writing faster than the replica can apply, lag grows. 
A replica 30 minutes behind during checkout means any read-after-write flow that hit the replica reads pre-order state.

### 2.6 Connection pool exhaustion

At 10× concurrency (users refreshing order-pending pages during outages), the pool fills up. Each connection waits on a lock or an IO. 
New requests queue; queue length grows; customer-facing latency spikes; some users retry; cascade.

---

## 3. The Solutions, In Order Of Increasing Invasiveness

### 3.1 Sub-partitioning — the minimum useful change

Keep the time-based partitioning (analytics still works) but **sub-partition each day by hash**:

```sql
-- Monthly range parent
CREATE TABLE orders (
    id              BIGSERIAL,
    customer_id     BIGINT,
    created_date    DATE NOT NULL,
    status          TEXT,
    total           NUMERIC(20,4),
    -- ...
    PRIMARY KEY (id, created_date)  -- partition key must be in PK
) PARTITION BY RANGE (created_date);

-- Each month is itself partitioned by hash(id)
CREATE TABLE orders_2026_11 PARTITION OF orders
    FOR VALUES FROM ('2026-11-01') TO ('2026-12-01')
    PARTITION BY HASH (id);

CREATE TABLE orders_2026_11_h0 PARTITION OF orders_2026_11
    FOR VALUES WITH (MODULUS 32, REMAINDER 0);
-- ... through h31
CREATE TABLE orders_2026_11_h31 PARTITION OF orders_2026_11
    FOR VALUES WITH (MODULUS 32, REMAINDER 31);
```

Effect:
- Writes for a given day hash into 32 physical tables. Each sub-partition gets 1/32 of the day's volume.
- Index right-edge contention drops from "one rightmost leaf across all inserts" to "rightmost leaf across 1/32 of inserts." That's often enough to move the bottleneck somewhere else.
- Analytics `WHERE created_date BETWEEN ...` still prunes at the month level. The planner reads all 32 sub-partitions for the matched months, but that's a 32× scan per month, not a full-history scan.
- Each sub-partition has its own WAL write contention, its own autovacuum cadence, and (if the storage is on different devices / tablespaces) its own IO path.

Tradeoffs:
- 32 more files per month. Postgres handles hundreds of partitions fine; thousands start to slow planning time (`partition_pruning` is O(log N) but has constant factors). Monitor `pg_stat_all_tables` size.
- Unique constraints must include the partition key (id, created_date). Usually fine; sometimes awkward for FK relationships.
- Queries that filter only by `id` (no date) touch all sub-partitions within matched months. Provide an index that the planner can use, or denormalize `created_date` into the query.

This is the "cheap" answer that buys you 10-30x write headroom with modest code changes. It's usually enough for Black Friday.

### 3.2 Horizontal sharding — the answer past a single node

A single Postgres instance tops out at some write rate (hardware-dependent, typically 20-100K commits/sec for OLTP with durability on). Past that, you need multiple nodes.

**Citus / Postgres sharding extensions:** distributes tables across nodes by shard key. **Co-locate tables that are joined** (orders + order_lines, both sharded by `customer_id` or `order_id`, land on the same node). 
Cross-shard queries are supported but expensive.

**Application-level sharding:** your service computes `shard = hash(customer_id) % N` and routes to shard N. Most mature; most operational work. Each shard is an independent Postgres; failover, backup, schema migrations must be orchestrated across all shards.

**Distributed-native (CockroachDB, Spanner, Yugabyte):** partitions are ranges of the key space, automatically split when hot. 
The "hot partition" problem is detected and split in real-time: when range R sustains >N writes/sec, CockroachDB splits it into R1 and R2 and moves one to a different node. 
This is the one architectural answer that *automatically* handles the Black Friday scenario without pre-planning.

Choose based on throughput requirement and organizational maturity:
- < 30K writes/sec: single beefy Postgres + sub-partitioning. Don't shard.
- 30K-200K writes/sec: Citus or app-level sharding.
- > 200K writes/sec: distributed-native DB with automatic range splitting-> **COCKROACHDB_RANGE_SPLIT_THRESHOLD**.

### 3.3 Write buffering — decouple application throughput from DB throughput

Introduce a queue between the app and the DB. Orders come in → Kafka → consumer inserts in batches of 1000 with COPY → DB sees 1/1000 of the connection count.

```
App  →  Kafka  →  Consumer (batch 1000)  →  COPY INTO orders
         │
         └── durability guarantee while DB catches up
```

Effect:
- App-side latency unchanged (Kafka ack in ~5ms).
- DB absorbs peaks asynchronously. A 100× spike becomes a 100× backlog that drains over N minutes.
- Bulk COPY is 5-10× faster than single-row INSERT (see `systemDesign/database/postgres.md`).

Tradeoffs:
- Customer sees "order placed" before it's in the DB. If the consumer is behind, a refresh may show "no orders yet." Handle with a "processing" state in the UI and a client-side local cache.
- Order ID generation moves to the app (can't use `SERIAL` since insert is async). **Use UUIDv7 or Snowflake IDs.**
- Ordering guarantees need to be modeled carefully — Kafka partition key = customer_id ensures per-customer order arrival order.
- Failure modes: Kafka full, consumer crashed, DB down. Each needs explicit handling (but typically easier than "DB on fire").

This is the pattern that turns "write spike of 100×" into "write *average* of 3× over 30 minutes." The DB hardware requirement collapses.

### 3.4 Hot/cold separation

Most queries touch recent orders. Old orders are queried rarely (support tickets, audits).

Architecture:
```
orders_hot    → fast OLTP (Postgres primary/replicas, last 90 days)
orders_cold   → columnar archive (S3 + Parquet + Athena, or Clickhouse)
```

Nightly/weekly: move partitions older than N days from hot to cold. `DETACH PARTITION` → export to Parquet → archive. The hot DB stays small (no 5-year history bloating the planner).

Benefits for Black Friday specifically:
- The partition under write pressure is not competing for buffer cache with years of old data.
- Backup/restore is fast because the hot DB is small.
- Schema migrations are fast because they only run over hot partitions.

Implement early. Migrating to hot/cold after you're at 10TB is painful.

---

## 4. The Composite Schema That Usually Wins

For an orders table with both analytics and OLTP workloads, the production-grade structure:

```sql
CREATE TABLE orders (
    id              UUID NOT NULL,           -- UUIDv7: time-ordered, high entropy
    tenant_id       BIGINT NOT NULL,         -- if multi-tenant; great shard key
    customer_id     BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    created_date    DATE GENERATED ALWAYS AS (created_at::date) STORED,
    status          TEXT NOT NULL,
    total           NUMERIC(20,4) NOT NULL,
    currency        TEXT NOT NULL,
    -- ...
    PRIMARY KEY (id, created_date)           -- partition columns in PK
) PARTITION BY RANGE (created_date);

-- Monthly time partitions
CREATE TABLE orders_y2026m11 PARTITION OF orders
    FOR VALUES FROM ('2026-11-01') TO ('2026-12-01')
    PARTITION BY HASH (id);

-- Hash sub-partitions within each month: 32 shards
CREATE TABLE orders_y2026m11_s00 PARTITION OF orders_y2026m11
    FOR VALUES WITH (MODULUS 32, REMAINDER 0);
-- ... s01 through s31

-- Indexes defined on the parent; Postgres cascades to children
CREATE INDEX orders_customer_idx ON orders (customer_id, created_at DESC);
CREATE INDEX orders_status_idx ON orders (status) WHERE status IN ('pending','processing');
```

Properties:
- Writes spread across 32 sub-partitions per month (bounded write amplification).
- Analytics `WHERE created_date BETWEEN '2026-11-01' AND '2026-11-30'` prunes to 32 partitions in a single month — still manageable, and parallelizable via `max_parallel_workers_per_gather`.
- Customer order history (`WHERE customer_id = $1 ORDER BY created_at DESC`) uses the covering index; the planner prunes by scanning the index on all partitions but applying LIMIT early.
- `DETACH PARTITION orders_y2025m11` in O(1) for cold migration.
- Each sub-partition is a small file Postgres can vacuum, reindex, and back up independently.

What you give up:
- A **global unique constraint on just `id`** requires the constraint to include `created_date` or the use of a separate lookup table to enforce global uniqueness 
  (UUIDv7 keeps collision probability at the heat death of the universe, so practically you don't need one).
- Global foreign keys referencing `orders(id)` must include `created_date` too. In practice this is fine — children (order_lines, shipments) are created at the same time as the order.

---

## 5. Pre-Black-Friday Playbook

Operationally, you can dodge 80% of the pain by preparing before peak:

**Two weeks before:**
- Pre-create all partitions for the next 3-6 months (pg_partman or a manual script). Don't discover you forgot to create Dec partitions on Nov 29.
- Run `VACUUM FULL` (or `pg_repack` online) on the current partition if bloat is > 20%. Enter peak with a compact table.
- Benchmark current throughput against forecast peak. If forecast is 3× current and current is at 40% CPU, you have headroom. If current is at 80%, you don't.
- Run game-day: replay prod traffic at 2× rate against a staging clone. Measure what fails first.

**One week before:**
- Review index necessity. A 100× write rate is a 100× index write rate. **Drop indexes that aren't hot-path read-critical during Black Friday; re-create afterward.**
- Scale up primary DB instance if needed (CPU, RAM, WAL disk IOPS).
- Increase connection pool carefully — more connections means more lock waiters if writes are the bottleneck. Usually *decrease* pool size to force fast-failing on contention.
- Pre-warm caches. Hit representative product/order pages to populate the buffer cache.

**Day of:**
- Temporarily disable non-essential triggers (analytics events, denormalization updates) and fire them from a queue instead.
- Increase autovacuum aggressiveness on today's partition (as shown in §2.4).
- Watch WAL generation rate, replication lag, lock waits. Dashboards, not logs.
- Have a pre-written "shed load" switch — if the DB is at 95% CPU and climbing, gracefully degrade (show "checkout delayed, order will confirm in a moment" instead of 500s).

**Day after:**
- Vacuum the day's partition aggressively; it's where your bloat is.
- Re-enable anything you disabled //triggers, indexes, etc.
- Run a full-history query to check for any missed indexes.
- Post-mortem what broke. Something always breaks.

---

## 6. Edge Cases At The Partition Boundary

### 6.1 Midnight crossover

The day's partition flips at 00:00. A transaction that started at 23:59:58 and commits at 00:00:02 — which partition does its row go in?

Postgres: the partition is determined by the *value* of `created_date` in the row, not the commit time. If the app sets `created_at = NOW()` *inside the transaction*, 
NOW() is transaction-start time — 23:59:58 — so the row lands in the "today" partition even though it commits tomorrow. Consistent.

The trap: if `created_at` is set by the *client* (timestamp sent from app server), clock skew between app and DB can put an order into the wrong partition. 
Best practice: use DB `DEFAULT now()` for `created_at`, not app-supplied values. Ensures partitioning truth and DB truth agree.

### 6.2 Late-arriving data

Order was created at 23:58, but its row is inserted via async outbox at 00:05 the next day. If the code uses `now()` at insert time, it goes into tomorrow's partition — business-day reporting disagrees with DB partitioning.

Fix: the outbox event carries `created_at`; insertion uses that value, not `now()`. The order ends up in the correct day's partition.

### 6.3 Back-dated orders (returns, manual entries)

A customer-service tool creates a record dated 3 months ago (say, for a correction). It lands in an old partition which may have different constraints, different indexes after prior schema changes, or may be detached / in cold storage.

Policies:
- Reject the write (you can't modify historical partitions). Clean but rigid.
- Allow in a special "adjustments" partition, separate from the time-range partitioning. Reconciliation job links them to original orders. This is how most financial systems handle it.

### 6.4 Unique constraint across partitions

You want `(customer_id, idempotency_key)` unique. Postgres enforces unique constraints *within* each partition, not across — unless you include the partition key in the constraint.

```sql
UNIQUE (id, created_date)                              -- enforceable; created_date in PK
UNIQUE (customer_id, idempotency_key)                  -- cannot enforce across partitions
UNIQUE (customer_id, idempotency_key, created_date)    -- enforceable; but doesn't prevent cross-day dupes
```

Real-world fix: a separate `idempotency_keys` table (not partitioned, or partitioned by the key's hash) that acts as the uniqueness enforcer. Insert into it first; on conflict, return the existing result. See `systemDesign/database/databaseTransactions.md §6.1` for the pattern.

### 6.5 Foreign keys to partitioned tables

An `order_lines.order_id → orders.id` FK can reference the partitioned parent only if the partition key is in both sides of the FK. In practice, this means `order_lines` needs `created_date` too (denormalized). That's usually fine because lines are created at the same time as the order; both go into matching partitions.

Alternative: don't use database FKs for orders/lines; enforce in application code. Many high-scale systems do this. Tradeoff: lose DB-level integrity checking; gain schema-migration flexibility and avoid cross-partition FK overhead.

### 6.6 Index right-edge hotspot survives sub-partitioning if keys are sequential

Hash sub-partitioning spreads inserts across 32 tables — but within each table, if the PK is still `BIGSERIAL`, each table's rightmost leaf is still hot. You win a 32× factor but the fundamental right-edge problem persists.

Fix by using a non-sequential but still time-ordered key (UUIDv7). Each sub-partition's rightmost leaf is hit by ~1/32 of writes, and *within* the sub-partition, the insert position varies across many leaves rather than always the same rightmost one.

---

## 7. Observability Per Partition

You cannot manage what you cannot see at the partition level. Key dashboards:

**Write rate per partition**
```sql
SELECT relname, n_tup_ins, n_tup_upd, n_live_tup, n_dead_tup
FROM pg_stat_user_tables
WHERE relname LIKE 'orders_%'
ORDER BY n_tup_ins DESC LIMIT 20;
```
Spot imbalance. If sub-partitioning by hash is working, the top-32 rows should have similar counts. If not, your hash function is skewed (bug) or one tenant is pathologically large (hot tenant).

**Lock waits per partition**
Alert if any partition has > N seconds of cumulative lock waits per minute.

**Bloat per partition**
Run a bloat query (`pgstattuple`, or an approximation query) across all partitions. Alert if bloat > 40% on any hot partition.

**Autovacuum log**
`log_autovacuum_min_duration = 0` to log every autovacuum run. Dashboard: time between vacuums per partition, duration of each vacuum. If time-between is growing (autovacuum falling behind), alert.

**Replication lag per partition — tricky**
Physical replication doesn't work per-partition. But you can **add logical replication per partition group** (e.g., a publication for orders_2026_11_*) and track lag independently.

**Planner partition pruning**
`EXPLAIN (analyze, verbose)` on hot queries to confirm partition pruning is active. Regressions happen when someone writes `WHERE created_date::text LIKE '2026-11-%'` — the cast defeats pruning. 
Alert on planner metrics showing "all partitions scanned" for OLTP queries.

---

## 8. Migration Paths Out Of The Hot-Partition Trap

If you're already running monthly-only partitioning and feeling the pain:

### 8.1 Live migrate to hash sub-partitioning

1. Create new partitioned table with the composite scheme (`orders_v2`).
2. Dual-write: application writes to both tables during migration window.
3. Backfill historical partitions from old to new (one partition at a time, during low traffic).
4. Cut reads over once backfill is complete.
5. Drop the old table.

Duration: weeks for multi-TB tables. Use Citus's online shard splitting, or an external tool like `pg_dump` + replay, or logical replication to a new schema.

### 8.2 Staged approach: add hash sub-partitioning to new partitions only

Easier in the short term: change the partition-creation template so new partitions (going forward) are hash-sub-partitioned, while historical partitions remain unchanged. Peak season traffic hits the new structure. Old data's query patterns don't care (and are cold).

Concretely: modify `pg_partman`'s template table, or whatever script creates new partitions. New partitions show up with sub-partitions; old don't. Eventually age-out and drop the old ones.

### 8.3 Pre-split the hot day's partition

A day before Black Friday, create the Black Friday partition as hash-sub-partitioned even if the surrounding days aren't. Lives only for its hot day; rolls into cold with the rest; quirky schema is transient.

```sql
CREATE TABLE orders_y2026m11d27 PARTITION OF orders
    FOR VALUES FROM ('2026-11-27') TO ('2026-11-28')
    PARTITION BY HASH (id);
-- 64 sub-partitions for this day only
```

This is a pragmatic compromise when you can't afford a full schema migration before peak.

---

## 9. The Staff-Level Summary

When this shows up in a design review:

1. **Partitioning has two workloads, and they conflict.** Analytics wants time-based; OLTP wants hash-based. A single partition column serves one at the expense of the other.
2. **Composite partitioning (range time → hash id) resolves the conflict.** The cost is 32× partition count and minor query-planning overhead. The benefit is write-spread + read-pruning simultaneously.
3. **100× write spikes are an operational event, not an architectural crisis — if you've prepared.** Pre-partitioning, sub-partitioning, buffering via Kafka, and pre-warmed caches make the difference between "slow checkout" and "site down."
4. **Distributed databases (CockroachDB / Spanner / Yugabyte) solve this automatically.** If you're under 200K writes/sec you can handle it in Postgres with sub-partitioning; past that, consider the distributed-native option.
5. **Hot/cold separation is orthogonal and important.** The hot DB should not carry 5 years of history into peak load. Move old partitions to cheap columnar storage; the planner, the buffer cache, and the backup pipeline will thank you.
6. **Observability is per-partition.** Aggregate "table is slow" tells you nothing. "Partition `orders_y2026m11d27` has 40% bloat and autovacuum is 3 hours behind" is actionable.

The "365 partitions, 1 hot" pattern is the most common symptom of a table that was designed for analytics queries by an engineer who never had to run it under Black Friday load. The fix isn't clever config tuning; it's acknowledging that the write workload and read workload must each get their own dimension in the partition key.