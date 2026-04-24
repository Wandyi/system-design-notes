# B-Tree Right-Edge Contention — Deep Dive

> Sequential `order_id` primary key: every INSERT hits the rightmost leaf of the B-tree. At high concurrency, writers serialize on the leaf's latch. Classic "right-edge index" contention. Postgres 12+ has deduplication that helps somewhat; version 13's btree improvements help more; none change the fundamental topology of a B-tree with monotonic keys.

This expands §2.2 of `hotPartition.md`. The failure mode is subtle because every piece looks correct in isolation: B-trees are fast, `BIGSERIAL` is idiomatic, indexes are mandatory on PKs. The pathology emerges only at the intersection of **monotonic key generation**, **high write concurrency**, and **physical page-level locking inside the index**.

---

## Table of Contents

1. B-Tree Anatomy in Postgres (The Physical Truth)
2. What Happens During a Single INSERT
3. Concurrency: Page Latches vs Row Locks vs Transaction Locks
4. Why Monotonic Keys Funnel to One Leaf
5. The Serialization Math
6. WAL, FPI, and the Amplification Beyond CPU
7. What Postgres 12 Dedup and 13 B-Tree Work Actually Do
8. Why No Config Change Can Fix the Topology
9. How to Diagnose It in Production
10. The Remediations, Ranked
11. UUIDv7 vs Snowflake vs Bit-Reversal: Trade-offs
12. Worked Example: 10K Writers on a Sequential PK
13. Edge Cases and Non-Obvious Interactions
14. Staff-Level Summary

---

## 1. B-Tree Anatomy in Postgres (The Physical Truth)

Postgres's default index is a **Lehman-Yao B+ tree** (a variant of B-tree with sibling pointers at the leaf level and right-links at all levels). The structure:

```
                      [ ROOT ]              <- one page (8 KB)
                     /   |    \
                 [I1]   [I2]   [I3]         <- internal pages
                /  \    /  \    /  \
             [L1]-[L2]-[L3]-[L4]-[L5]-[L6]  <- leaf pages, linked list
              |    |    |    |    |    |
              v    v    v    v    v    v
           heap tuples (the actual row data, in table pages)
```

Key physical properties:

- **Every node is one 8 KB page** (Postgres's `BLCKSZ`).
- **Leaf pages hold `(indexed_key, tid)` pairs** where `tid` points to the heap row. For `BIGSERIAL` keys, each entry is ~20 bytes; a leaf holds ~300–400 entries.
- **Leaf pages are doubly linked** (left/right pointers) so range scans walk sideways without climbing back to the root.
- **Internal pages hold `(separator_key, child_pointer)` pairs**. Separator keys let a search descend deterministically.
- The tree is **kept balanced**: splits cascade upward when a page is full.

For an orders table with 100M rows:
- Leaf level: ~300K pages × 8 KB = **~2.4 GB of index**.
- Internal level: ~1000 pages.
- Root: 1 page.
- Tree height: 3–4 levels.

A point lookup traverses 3–4 pages; a range scan hits a leaf then walks the sibling chain.

### The "rightmost leaf" is a specific page

For a B-tree ordered by key, the **rightmost leaf** is the page that holds the largest-valued keys currently in the index. If your PK is `BIGSERIAL`, this is the page whose keys are in the range `[current_max_id - ~300, current_max_id]`. Every new insert with key `current_max_id + 1` lands here.

Postgres has a specific **fast-path optimization** for this: a cached hint called the `BTREE_FASTPATH`. If the new key is greater than the current max, the insert skips the root-to-leaf descent and jumps directly to the cached rightmost leaf pointer. This is designed for exactly the monotonic-key case — and ironically makes the contention **sharper** because every writer converges on the same page via the same shortcut.

---

## 2. What Happens During a Single INSERT

An `INSERT INTO orders (...) VALUES (...)` does roughly this sequence for the PK index:

1. **Tuple construction**: compute the index key from the row.
2. **Descent (or fast-path to rightmost)**: find the target leaf page. For monotonic keys, this hits the cached rightmost-leaf hint.
3. **Buffer pin**: acquire a **pin** on the leaf page buffer (increments refcount so it can't be evicted while in use).
4. **LWLock in exclusive mode** on the buffer's **content lock** (the "latch"). This is the contention point.
5. **Find insertion position** within the page (binary search within 8 KB).
6. **Check free space**: is there room for the new entry?
   - **If yes**: write the entry in place, write a WAL record (`XLOG_BTREE_INSERT_LEAF`), mark page dirty, release content lock, release pin.
   - **If no**: **page split** — allocate a new page, redistribute half the entries, update the parent's separator, write a compound WAL record (`XLOG_BTREE_SPLIT_R`). Split cost is ~5–10× a regular insert, and holds the content lock longer.
7. The heap insert (table row) happens separately with its own locks.
8. On `COMMIT`: WAL flush to disk, durability ack.

Crucially, step 4 (exclusive LWLock on the leaf's content) **only one backend at a time** can hold. Everyone else queues.

---

## 3. Concurrency: Page Latches vs Row Locks vs Transaction Locks

There are three completely different lock types involved, and conflating them is the #1 diagnostic mistake.

| Lock type | Duration | Granularity | Granted how |
|---|---|---|---|
| **LWLock (content lock / "latch")** | Microseconds | Per-buffer (one 8 KB page) | Queued, exclusive/shared |
| **Buffer pin** | Nanoseconds to microseconds | Per-buffer | Reference count |
| **Tuple lock (SELECT FOR UPDATE)** | Transaction duration | Per heap row | In lock table, visible in `pg_locks` |
| **Transaction lock / xid lock** | Transaction duration | Per transaction | Visible in `pg_locks` |

The right-edge contention is at the **LWLock / latch** level. It is:
- **Not visible in `pg_locks`** (LWLocks are internal; pg_locks shows heavyweight locks only).
- **Visible in `pg_stat_activity.wait_event`** as `BufferContent` on an index page.
- **Visible in `pg_wait_sampling`** or Postgres 13+'s `pg_stat_activity.wait_event` sampling.

If you grep `pg_locks` for contention and see nothing, this is why. The contention is at a lower layer than `pg_locks` reports. You need `wait_event`-level instrumentation:

```sql
SELECT wait_event_type, wait_event, count(*)
FROM pg_stat_activity
WHERE state = 'active'
GROUP BY 1, 2
ORDER BY 3 DESC;
```

Expected output under right-edge contention:

```
 wait_event_type |  wait_event   | count
-----------------+---------------+-------
 LWLock          | BufferContent |   847
 Client          | ClientRead    |   120
 ...
```

`BufferContent` dominating means writers are waiting on a specific page's content latch. Combined with a monotonic PK, it's almost certainly the rightmost leaf.

---

## 4. Why Monotonic Keys Funnel to One Leaf

Consider 100 concurrent writers, each inserting an order. Key assignment:

- Writer 1: gets `order_id = 1_000_001`.
- Writer 2: `order_id = 1_000_002`.
- ...
- Writer 100: `order_id = 1_000_100`.

All 100 keys are in a range of 100. The **current rightmost leaf page** holds keys roughly in the range `[999_700, 1_000_000]`. After these 100 inserts, it might span into `[999_700, 1_000_100]` — still one page (300 entries fit in 8 KB).

Every one of those 100 inserts targets the same 8 KB of memory. The LWLock on that page is exclusive. They serialize.

Contrast with UUIDv4 (random keys) on a large index:
- 100 writers, 100 random keys.
- Those keys are distributed across ~100 different leaf pages (because the tree has ~300K leaves for a 100M-row table and randomness scatters across them).
- 100 inserts hit 100 different LWLocks. All run in parallel.

**The physics is identical; the access pattern is completely different.**

### Quantifying "the rightmost leaf"

In an append-only workload with a monotonic key, **99%+ of inserts touch the same leaf page** until it fills. Then:
- The page splits.
- For a monotonic key, the split is a **right-side split**: the old page keeps the lower half; a new page holds the upper half; the new page becomes the rightmost.
- The rightmost-leaf cache updates to point to the new page.
- Now 99% of inserts hit the new leaf.

This is Postgres's "split to the right" optimization (since ~9.5): it recognizes monotonic inserts and splits so that the old page stays 90% full (not 50%), maximizing space efficiency. Good for space. **Still a single hot page at any moment.**

---

## 5. The Serialization Math

The LWLock on a leaf page is held for microseconds, but at high concurrency the wait queue grows.

If each insert's critical section on the leaf is 20 µs (descend + find position + write entry + release), then:
- **Throughput ceiling: 1 / 20 µs = 50,000 inserts/sec** on a single leaf.
- This is **per-leaf**, so with a monotonic key it's **the total write ceiling** for that index.

Real-world measurements on modern hardware:
- Sequential `BIGSERIAL` PK on orders: ~40K–80K inserts/sec before latch contention dominates.
- Random UUID PK on the same setup: 200K+ inserts/sec (spread across many leaves).
- UUIDv7: 100K–150K inserts/sec (time-ordered enough for cache locality, random enough to distribute across multiple leaves).

And that's just the PK. If you have 3 secondary indexes, each with its own right-edge or random topology, the contention is compounded: if any one of them serializes, your write throughput is bounded by the slowest index.

### The queue amplifies

Queue theory: if arrival rate λ approaches service rate µ, wait time blows up as 1/(µ − λ). At 80% utilization of the leaf's LWLock, P99 wait is already several milliseconds (versus 20 µs service time). At 95%, multi-tens of milliseconds. The shape is a hockey stick.

This is why "everything was fine until suddenly it wasn't." At 40K inserts/sec you're at 80% of leaf capacity and P99 is 2 ms. At 48K you're at 96% and P99 is 50 ms and climbing. There's no smooth degradation — you fall off a cliff as you approach the service rate.

---

## 6. WAL, FPI, and the Amplification Beyond CPU

The leaf latch isn't the only amplifier. Monotonic inserts also hit:

### Full-Page Images (FPI)

After a checkpoint, the first modification to any page writes a **Full-Page Image** to WAL (8 KB + overhead), not just the delta. This guards against torn-page writes.

With a hot leaf page and frequent checkpoints, this leaf generates **one FPI per checkpoint interval** (not per modification — only the first modification after a checkpoint). At default 5-minute checkpoints with a hot page, that's one 8 KB FPI per 5 min for this page — negligible.

**But**: the heap pages where the inserted rows land are *also* monotonic (new rows go to the newest heap page until it fills). Same FPI dynamic, but now with lots of heap pages rolling over rapidly. WAL generation rate for monotonic workloads is often 2–5× the logical data size because of FPI volume.

### Write set on the leaf

Each insert to the leaf:
- Modifies the leaf page → page becomes dirty.
- A single dirty page can absorb many modifications before being flushed (good — but needs flushing by bgwriter or checkpointer).
- But the **WAL record per modification is non-coalescible**. 50K inserts/sec = 50K WAL records/sec on this one page.

### Visibility map and FSM updates

Less intense but still: free-space map (FSM) updates for the heap and visibility-map bit clears. These are lighter than the leaf latch but add bytes to WAL and I/O pressure.

### Backend pinning

Even when a backend only holds a **shared pin** (reading the page), it prevents page eviction by the buffer manager. During right-edge contention, dozens of backends have the leaf pinned; the buffer manager can't swap it out. Not a problem here (we want it in memory), but in extreme cases (many different hot pages) this forces the buffer pool to hold pages that compete with other data.

---

## 7. What Postgres 12 Dedup and 13 B-Tree Work Actually Do

There's a fair amount of folklore here. Let's be precise.

### Postgres 12: B-tree deduplication for secondary indexes

The feature (`CREATE INDEX ... WITH (deduplicate_items = on)`, default on since 13): when a leaf page has multiple entries with the **same key** pointing to different heap tuples, store the key once and a list of TIDs. This is called a **posting list**.

**Use case**: a non-unique index on a low-cardinality column (e.g., `status`) where `status = 'pending'` repeats thousands of times. Dedup shrinks the index 2–5×.

**For a unique monotonic PK**: deduplication does **nothing**. Every key is distinct. No posting list is formed. The leaf page layout is identical to pre-12.

### Postgres 13: B-tree "bottom-up" deletion and general improvements

- **Bottom-up index deletion** (13+): when a leaf is about to split, check if heap tuples pointed to by the leaf entries are dead (due to UPDATEs). If so, remove those entries and skip the split. Helps UPDATE-heavy workloads.
  - For append-only INSERT workloads, not much: there are no dead tuples on the leaf yet.
- **Smaller index entries**: various savings (up to 20% on some indexes). Packs more entries per leaf page. More entries per leaf = more inserts before a split = **slightly more contention per page** because a hot page now takes longer to fill and split.
- **Better index statistics / planning**: unrelated to contention.

### Postgres 14+: parallel inserts (limited)

Parallel workers can build indexes in parallel (CREATE INDEX), but concurrent INSERT statements from different backends still serialize on the leaf latch. These are orthogonal features.

### What the version history does not change

- The **physical topology**: a B-tree has a rightmost leaf. Monotonic keys always go there. That's a mathematical property of the data structure plus the key distribution.
- The **LWLock granularity**: one content lock per page. Not subdividable without a different index type.
- The **rightmost-leaf fast path**: helpful for single-thread throughput, neutral for concurrent throughput, sharpens the hotspot.

So when someone says "Postgres 13's B-tree improvements help," they're right — throughput under mixed-workload scenarios is 10–30% better due to general optimization. But the **shape of the contention curve** (cliff at ~50K inserts/sec on a single leaf) is unchanged.

---

## 8. Why No Config Change Can Fix the Topology

Plausible-sounding "fixes" that don't work:

| Attempt | Why it doesn't fix the core problem |
|---|---|
| `shared_buffers = huge` | The leaf is already in memory. Bigger buffer pool doesn't make the single page less contended. |
| `max_connections = more` | More connections means more backends queuing on the same LWLock. Can make it worse. |
| `synchronous_commit = off` | Reduces WAL flush latency. Doesn't touch the LWLock on the leaf. May give +20% throughput until the latch is the bottleneck again. |
| `wal_writer_delay` tuning | Same — WAL pathway, not index pathway. |
| Bigger CPU / more cores | The workload is not CPU-bound on this path; it's serialized. Amdahl's law: if 90% of the insert is parallel but 10% is on the leaf latch, the theoretical max speedup is 10×, no matter how many cores. |
| Faster disk (NVMe, RAID) | Same reason — not I/O bound here. |
| `fillfactor = 70` on the index | Makes leaves less full → more splits → more contention on parent pages during splits. Worse for append-only. |
| Deduplication | No-op on unique keys. |
| Larger `block_size` at compile time | More entries per leaf → longer hot-page lifespan → same or worse contention. |

The only solutions are topological: change the key so inserts distribute, or change the index structure so there's no single hot point.

---

## 9. How to Diagnose It in Production

### Signal 1: wait events

```sql
SELECT wait_event_type, wait_event, count(*)
FROM pg_stat_activity
WHERE state = 'active' AND wait_event IS NOT NULL
GROUP BY 1, 2
ORDER BY 3 DESC;
```

Right-edge contention: `LWLock` / `BufferContent` dominant among active sessions.

### Signal 2: which page is hot

Postgres doesn't expose the specific buffer ID easily, but you can narrow to the index:

```sql
-- requires pg_buffercache extension
SELECT c.relname, count(*) AS buffer_count
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
WHERE b.isdirty
GROUP BY c.relname
ORDER BY buffer_count DESC;
```

If one index has many dirty buffers concentrated at high block numbers, it's the rightmost-leaf area.

### Signal 3: per-index I/O stats

```sql
SELECT indexrelname, idx_tup_read, idx_tup_fetch, idx_blks_read, idx_blks_hit
FROM pg_stat_user_indexes
WHERE schemaname = 'public' AND relname = 'orders';
```

For a monotonic-PK index under stress, `idx_blks_hit` is enormous (always in cache) and `idx_blks_read` is near zero. All the action is CPU+LWLock, not I/O.

### Signal 4: latency histogram

If you have `pg_stat_statements` with `pg_stat_statements.track = all`:

```sql
SELECT query, calls, total_exec_time, mean_exec_time, stddev_exec_time
FROM pg_stat_statements
WHERE query ILIKE 'INSERT INTO orders%'
ORDER BY total_exec_time DESC;
```

A high `stddev_exec_time` relative to `mean_exec_time` (ratio > 3) signals queue-dependent latency — the signature of LWLock contention.

### Signal 5: ebpf / perf sampling

For the confident operator: `perf top -p <postgres backend pid>` during the incident shows `LWLockAcquire` and `BufferContent` symbols at the top of the profile. Definitive.

---

## 10. The Remediations, Ranked

### R1 — UUIDv7 primary keys

Replace `BIGSERIAL` with **UUIDv7** (RFC 9562, time-ordered UUIDs). Format:

```
unix_ts_ms (48 bits) | ver (4) | rand_a (12) | var (2) | rand_b (62)
```

- Monotonic at the **millisecond** level (so inserts are in a narrow trailing region of the tree — cache locality for buffer).
- Within the same millisecond, the 74 bits of randomness scatter across many leaves.
- At 10K inserts/sec, that's ~10 inserts/ms — those 10 land on ~10 different leaves.

Result: leaf latch contention dissolves. Writes scale linearly with cores until some other bottleneck appears.

**Trade-offs:**
- 16 bytes per key instead of 8 → index is 2× bigger. Acceptable.
- UUIDs in logs / URLs are less human-friendly. Usually a minor UX issue.
- Sort order is time-ish but not strictly monotonic — fine for `ORDER BY id DESC LIMIT N` style queries on recent data.

### R2 — Snowflake-style IDs (sharded monotonic)

```
41 bits timestamp | 10 bits machine_id | 12 bits sequence
```

Generated in-app. Monotonic globally, but **different machines use different `machine_id` prefixes**, so their key ranges don't collide at the leaf level (they're in different parts of the tree).

Works well when:
- You have a fleet of N app servers; inserts distribute across N key ranges.
- You want time-ordered IDs for database scans.

**Trade-offs:**
- Coordination: each machine needs a unique `machine_id` (via ZK, etcd, or deployment config).
- Still 10 keys/ms per machine hit the same leaf (per-machine right-edge), but spread across N machines gives N× headroom.

### R3 — Bit-reversed sequential keys

Take `BIGSERIAL`, reverse the bit order. Key 1 becomes `1 << 63`, key 2 becomes `1 << 62`, etc. Neighboring sequential values end up far apart in key space.

```sql
-- Conceptually (pseudocode)
id_stored = bit_reverse(bigserial_next())
```

Distributes writes perfectly. Completely destroys range scan on the ID column.

**Use when:**
- You never scan by `id` as a range (only by equality / join).
- You need the smallest possible key size (still 8 bytes).

**Avoid when:**
- `ORDER BY id DESC LIMIT 100` is a hot query — becomes a full scan.

### R4 — Hash sub-partitioning (from `hotPartition.md §3.1`)

Doesn't change the key type, but spreads writes across **N physical tables**, each with its own B-tree and its own rightmost leaf. N× reduction in per-leaf contention.

Stacks with R1/R2/R3: UUIDv7 + hash sub-partitioning gives both within-partition distribution and across-partition distribution.

### R5 — Write buffering via Kafka + bulk COPY

Covered in `hotPartition.md §3.3`. Bulk `COPY` uses a different code path that inserts many rows under fewer LWLock acquisitions. A single `COPY` of 1000 rows is not 1000× the latch ops — it batches leaf acquisitions.

### R6 — Drop non-essential indexes during peak

Every secondary index has its own right-edge (or random) topology. Dropping an analytics index during Black Friday and rebuilding after can cut write amplification by 30–50%. Requires ops planning.

### R7 — Different data structure

Move to a database with a log-structured merge tree (LSM): RocksDB-backed systems (CockroachDB, TiDB with TiKV, YugabyteDB) absorb monotonic writes without leaf contention because writes go to a memtable, not an in-place index.

Structural: if you consistently need > 100K monotonic writes/sec, Postgres's B-tree may be the wrong engine.

---

## 11. UUIDv7 vs Snowflake vs Bit-Reversal: Trade-offs

| Property | `BIGSERIAL` | UUIDv7 | Snowflake | Bit-reversed |
|---|---|---|---|---|
| Key size | 8 B | 16 B | 8 B | 8 B |
| Time-ordered | ✔ | ✔ (ms) | ✔ | ✘ |
| Client-generable (offline) | ✘ | ✔ | ✔ | ✘ (trivially) |
| Collision-safe across services | ✘ | ✔ | ✔ (with coord.) | ✘ |
| Range scan on PK | ✔ | ~✔ (per-ms) | ✔ | ✘ |
| Insert contention | High | Low | Low | Lowest |
| Human readable in URLs | ✔ | ~ | ~ | ✘ |
| Migration effort from existing | — | Medium | Medium | High |

**Default recommendation for new tables: UUIDv7.** It's the cleanest trade-off for high-concurrency writes with minimal behavior surprise. Good tooling support now (Postgres 18 has `uuidv7()` built in; older versions need an extension or app-side generation).

**Use Snowflake when:**
- You're in a polyglot environment where Kafka partitions / Spark jobs need time-sortable 8-byte IDs.
- You want to embed machine/shard origin in the ID.

**Use bit-reversed sequential when:**
- You're pathologically space-constrained AND never range-scan by PK AND can't afford UUID width.

**Rarely use `BIGSERIAL` for tables with > 10K inserts/sec.** It's fine for low-volume tables (users, products), not for orders/events/metrics.

---

## 12. Worked Example: 10K Writers on a Sequential PK

Setup: Postgres 15, 32-core machine, orders table with `id BIGSERIAL PRIMARY KEY`. Load generator: 10K concurrent writers, each INSERTing an order every ~100 ms → aggregate ~100K inserts/sec target.

Expected measurements:

- **Attempted rate**: 100K inserts/sec.
- **Actual sustained rate**: 45–60K inserts/sec. Queueing throttles.
- **P50 insert latency**: 2 ms.
- **P99 insert latency**: 80 ms.
- **`pg_stat_activity.wait_event`**: `BufferContent` dominates, typically 60–80% of active backends.
- **`pg_stat_statements.stddev_exec_time / mean_exec_time`** for the INSERT: > 5 (extremely variable).
- **CPU**: 40–60% utilized (not pegged — serialization, not compute).
- **I/O**: moderate. WAL bandwidth ~200 MB/s. Not maxed.
- **Primary bottleneck**: single leaf page LWLock.

Now swap `id BIGSERIAL` for `id UUID DEFAULT uuidv7()`:

- **Attempted rate**: 100K inserts/sec.
- **Actual sustained rate**: ~95K inserts/sec. Scales nearly linearly.
- **P50 insert latency**: 1.5 ms.
- **P99 insert latency**: 8 ms.
- **Wait events**: diffused across many buffer pages; `BufferContent` no longer dominates.
- **CPU**: 80–90% utilized.
- **I/O**: WAL ~280 MB/s (16-byte keys instead of 8-byte → ~1.4× WAL).
- **Primary bottleneck**: CPU / memory bandwidth — a wholesome bottleneck you can throw hardware at.

This is a synthetic example but reflects typical production experience. The gap between "sequential PK, 45K/s" and "UUIDv7, 95K/s" on the same hardware is the concrete cost of right-edge contention.

---

## 13. Edge Cases and Non-Obvious Interactions

### 13.1 Secondary indexes also have right-edges

A `created_at TIMESTAMPTZ` index with `DEFAULT now()` is also monotonic. It has its own rightmost leaf. Switching PK to UUIDv7 fixes the PK index, but `CREATE INDEX ON orders(created_at)` still serializes.

Mitigation: either drop the index if it's not hot-path, or accept the contention on that index (often the PK is the dominant hot one).

### 13.2 Primary key WITH INCLUDE columns

`PRIMARY KEY (id) INCLUDE (status, total)` puts extra columns in the leaf (covering index). Makes leaves bigger → fewer entries per page → splits more often → even more right-edge pressure if key is monotonic.

Generally fine with UUIDv7 (distribution absorbs the cost); harmful with `BIGSERIAL`.

### 13.3 INSERT ... ON CONFLICT DO NOTHING

Adds a lookup step that acquires a **shared** (not exclusive) LWLock on the leaf first to check for conflict. Doesn't serialize writers among themselves but does add read pressure to the same hot page. Shared locks don't mutually exclude, but they can delay exclusive lock acquisition slightly.

### 13.4 HOT updates near the hot leaf

HOT (Heap-Only Tuple) update: row updated in place when new version fits on same heap page. Skips index update. Great for updates to non-indexed columns.

Unrelated to leaf latch contention — index isn't touched. But heap-page latch contention can still be an issue for rows being updated rapidly (same row, many updates) — see `hotPartition.md §2.3` on heap tuple locking.

### 13.5 Logical replication decoders

The logical replication slot has to decode each WAL record. Monotonic inserts produce a flood of small WAL records. The decoder can fall behind; replication slot grows; WAL can't be recycled. An unrelated system (the replica) now causes the primary to accumulate WAL.

Monitor slot lag: `SELECT slot_name, pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn) AS lag_bytes FROM pg_replication_slots;`.

### 13.6 TOAST

Order rows with big text columns (memos, raw JSON) get TOASTed: the row header in the heap page is small, the big value lives in a TOAST relation. TOAST insertions have their own B-tree with its own right-edge. Usually not a concern unless TOAST traffic is very high; but it's there.

### 13.7 Partition routing latency

With hash sub-partitioning on UUIDv7, each INSERT computes `hash(id) % 32` to find the sub-partition. This is near-free, but it adds a small plan-time cost for non-prepared statements. Use prepared statements (`PREPARE`/`EXECUTE` or server-side prepared) to amortize.

### 13.8 VACUUM FREEZE on append-only tables

Append-only partition = no dead tuples = autovacuum barely runs... but `vacuum freeze` must run before `autovacuum_freeze_max_age` (default 200M transactions) to prevent wraparound. If you ignore it, one day the system starts aggressive anti-wraparound VACUUM on a huge append-only partition at the worst possible moment.

Pre-schedule `VACUUM FREEZE` on closed (previous-day/week) partitions during quiet hours. Never leave them to anti-wraparound emergency vacuum.

### 13.9 Connection pool position interacts

Under right-edge contention, backends hold active connections while waiting on the latch. Connection pool (pgbouncer) has a fixed upper bound. As latency spikes, more connections are busy-waiting; pool fills; new requests queue at pgbouncer. Front-end latency (client-visible) compounds: backend wait + pgbouncer wait + HTTP queue.

Reducing pool size forces fast-fail at the boundary, which is often better than silent queuing. Counter-intuitive but important.

### 13.10 Parallel COPY

`COPY ... FROM STDIN` in parallel streams (e.g., multiple pgbouncer connections doing COPY simultaneously) can paradoxically serialize on the rightmost leaf the same way single INSERTs do, if keys are monotonic across the parallel streams. To actually get parallel COPY throughput benefit, you need either UUIDv7 keys or separate partitions per stream.

---

## 14. Staff-Level Summary

1. **Right-edge contention is a data-structure property, not a Postgres bug.** Any B-tree with monotonic keys has a single hottest leaf at any moment; any high-concurrency writer set funnels into it; any version of Postgres serializes writers on that leaf's LWLock.

2. **The symptom is specific and measurable.** `LWLock` / `BufferContent` wait event dominates; `pg_stat_statements.stddev / mean` for the INSERT blows up; CPU is not pegged (40–60%); IO is not pegged (plenty of WAL headroom). If you see this pattern, it's the right-edge.

3. **Postgres 12 deduplication and 13+ improvements reduce index size and mixed-workload overhead** but **do not touch the topology**. No config flag, no amount of RAM, no faster disk fixes it. The fix is structural.

4. **The cheapest structural fix is UUIDv7.** You give up 8 bytes per key and a tiny bit of human-friendliness; you gain near-linear write scaling. For most new systems at moderate scale, this is the only decision that matters on this topic.

5. **Sub-partitioning stacks with UUIDv7.** Hash sub-partitioning gives you N× distribution across physical tables; UUIDv7 within each sub-partition gives additional distribution across leaves. Combined, a 10K-writer workload scales cleanly on a single Postgres instance to 100K+ inserts/sec.

6. **At 200K+ writes/sec, rethink the engine.** CockroachDB, YugabyteDB, and Spanner use LSM-based storage (no in-place B-tree hot leaf), and they range-split hot ranges automatically. If your ceiling is consistently right-edge-bound and you've done all the topological fixes, it may be time for a different engine rather than bigger Postgres.

7. **Prepare, don't react.** Right-edge contention appears as a cliff, not a gradient. You go from "everything fine" to "P99 is 500 ms" over 10% more load. Pre-peak load tests must include the failure mode, not just average throughput.

8. **The diagnosis is wait events, not `pg_locks`.** The lock in question is an LWLock (a latch), not a heavyweight lock. If on-call pulls up `pg_locks` and sees nothing, they'll miss it. Standardize on `wait_event` sampling in your DB observability stack.

9. **This is the canonical example of "database advice aging badly".** Every Postgres tutorial uses `BIGSERIAL` for the primary key. It's correct for tutorials. It's wrong for production tables that will see > 10K inserts/sec, and that threshold gets easier to cross every year. Default to UUIDv7 for new high-write tables; reserve `BIGSERIAL` for tables that provably don't need the write scaling.

10. **Right-edge contention teaches a general lesson**: data-structure topology matters more than hardware past a certain scale. You can throw silicon at CPU-bound workloads; you cannot throw silicon at a single 8 KB page that everyone wants to write simultaneously.