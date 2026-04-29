# Scaling Stateful Systems at MAANG Scale — Staff-Level Insights

> A deep, opinionated reference for designing, evolving, and operating *stateful* services that serve **1B+ users/day** or **>10M RPS** sustained, with petabyte working sets, multi-region presence, and tight (p99 ≤ 10–50 ms) SLOs. This is the operational-and-architectural knowledge that separates senior engineers (who scale things 10x) from staff engineers (who can take a system 1000x and survive the migrations, regressions, and 3 AM pages along the way).

---

## 0. How to Read This Document

This is a *taxonomy of decisions and their consequences*, not a tutorial. Each section answers three questions a staff engineer is expected to answer cold in a design review:

1. **What changes qualitatively** at this scale (i.e., where does naive engineering quietly fail)?
2. **What are the design dimensions**, and what is each dimension's blast radius when it goes wrong?
3. **What are the second- and third-order effects** that you, as a staff engineer, are paid to anticipate?

The numbers anchor reasoning. If you can't ballpark "how much disk does 10 PB cost", "what does 99.99% mean in failure budget", or "what is the p99 of a fan-out of 30 with per-node p99 of 20 ms", your design will be dismissed by senior reviewers within minutes.

---

## 1. What 1B Users/Day or 10M RPS Actually Means

### 1.1 Translating user counts to RPS

```
1B users/day  =  ~11.6K users/second on average
              =  ~50K–100K users/second at peak (5–10× peak/avg ratio)

Per-user activity (typical social/feed product):
  - 100 reads/day             →   1.16M reads/sec average,    ~10M RPS peak
  - 5 writes/day              →   ~58K writes/sec average,    ~500K WPS peak
  - 30 background sync ops/day → ~350K ops/sec average,      ~3M ops/sec peak
```

So "1B users/day" is essentially "10M RPS read peak, 500K WPS peak, with 3–5× spikes during product events (live launches, breaking news, sports, holidays)".

For comparison:
- **YouTube**: ~1B+ hours watched daily, peak ~50M concurrent viewers, ~100K live edge events/sec
- **Instagram**: ~2B daily actives, ~5–10M feed RPS at peak, hundreds of millions of stories writes/day
- **Facebook News Feed**: ~100M reads/sec at peak, ~1M writes/sec
- **WhatsApp**: ~100B messages/day = ~1.2M msgs/sec average, ~5M/sec peak
- **Netflix**: ~250M subscribers, ~1B+ hrs/week, dominated by **stateful** session and recommendation state, not message throughput

### 1.2 Storage scale that follows from the QPS

```
1M writes/sec × 86,400 sec/day × 1KB/write     = ~83 TB/day raw new data
× 365 days                                       = ~30 PB/yr raw
× 3 replicas                                     = ~90 PB/yr replicated
× 1.5 (indexes, secondary state)                 = ~135 PB/yr provisioned
```

You cannot manage 100+ PB/yr growth with single-cluster designs. **Tiered storage** (hot SSD ↔ warm HDD ↔ cold object store) and **TTL/archive policies** become first-class architectural concerns.

### 1.3 Latency math

A **fan-out read** of 30 backend calls, each with p99 = 20 ms, has overall p99 ≈ **80–120 ms** (you don't add p99s, you compose the *probability of any one being slow* via tail-amplification — see §13). At MAANG scale, fan-out is the dominant source of latency, not single-shard work.

### 1.4 Failure budget

99.99% availability = **52 min/year** of total downtime allowed.

If your stateful tier has **10 dependencies** that are each 99.99%, your effective max is 99.9% (~8.7 hr/yr), assuming serial dependence. This is the **"ALL"-availability arithmetic**: stateful systems must be more reliable than the weakest dependent link, *or* you must architect around failures (hedging, fallbacks, partial degradation).

---

## 2. Why Stateful ≠ Stateless × N

It is a defining staff-level insight that **scaling stateful systems is not "scaling stateless services and hooking up a database."** The differences:

| Concern | Stateless service | Stateful system |
|---|---|---|
| **Replacement** | Kill pod, start a new one. Done. | Cannot just "kill" a stateful node — it owns data. Hand off, drain, rebalance first. |
| **Routing** | Any instance handles any request. | Request must reach the *owning* shard/leader for correctness. |
| **Capacity unit** | CPU/RAM. Add more instances. | Disk, IOPS, working-set memory, **data gravity**. Cannot easily add capacity to a hot shard. |
| **Deploy** | Rolling restart, seconds. | Drains, rebalances, replication catches up, hours. |
| **Failure mode** | Crash & restart, no data risk. | Crash & restart can risk durability, consistency, split-brain. |
| **Recovery** | Stateless restart from artifact. | WAL replay, snapshot restore, re-replication, sometimes hours/days. |
| **Capacity planning** | Linear with QPS. | Non-linear: skew, tails, GC, compaction, snapshots cause bursts. |
| **Tests** | Stateless = idempotent. Easy to test. | Stateful = tests interact with prior state; harder to repro. |
| **Migrations** | Replace binary, done. | Schema migrations, dual-writes, backfills, cutover. Months. |

**The state-locality principle**: at MAANG scale, every architectural choice is forced by *where the state lives* and *who owns it at any moment*. Stateless layers exist primarily to *route* and *transform*; the stateful core is the bottleneck and the only thing whose failure is unrecoverable in seconds.

### 2.1 The Three Pillars of Stateful Design

```
                    ┌───────────────────────┐
                    │  STATEFUL SYSTEM       │
                    │  (1B users / 10M RPS)  │
                    └───────────┬───────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
        ▼                       ▼                       ▼
  ┌──────────┐          ┌──────────────┐         ┌──────────────┐
  │ Sharding │          │ Replication  │         │  Storage     │
  │ (where)  │          │ (how many,   │         │  Engine      │
  │          │          │  how strong) │         │  (how stored)│
  └────┬─────┘          └──────┬───────┘         └──────┬───────┘
       │                       │                        │
       └───── orthogonal but tightly coupled ───────────┘
                    cannot reason about one
                    without the others
```

A staff engineer is fluent in *all three* and the trade-offs between them.

---

## 3. Sharding — The Hard Part

### 3.1 Sharding strategies and their failure modes

| Strategy | How keys map | Scales reads | Scales writes | Hotspot resistance | Resharding cost |
|---|---|---|---|---|---|
| **Range partitioning** | `[startKey, endKey)` per shard | Medium | Medium | Poor (sequential keys hot) | Easy split/merge |
| **Hash partitioning** | `hash(key) mod N` | High | High | Excellent | Painful — every key may move |
| **Consistent hashing** | `hash(key) → ring; node owns arc` | High | High | Good | Add/remove shifts ~1/N keys |
| **Consistent hash + virtual nodes (vnodes)** | Each node owns 100s of vnodes | High | High | Excellent (load smoothed) | Smooth, gradual rebalance |
| **Directory-based / lookup table** | `key → shard` via centralized map | High | High | Excellent | Free movement, but lookup is critical path |
| **Geo-sharding** | Shard per region (`user.region`) | Per-region | Per-region | Poor for global hot users | High (cross-region) |
| **Tenant sharding** | Shard per customer/account | High (independent) | High | Good (one tenant ≠ all) | Complex (tenant moves) |
| **Composite (geo + hash)** | First by region, then by hash | High | High | Excellent | Two-level rebalance |

**Staff insight**: there is no universally-best strategy. The right choice is dictated by **the access pattern of the dominant query** and **the operational lifecycle** (resharding frequency, tenant onboarding, geo expansion).

### 3.2 Hash sharding's hidden cost

`hash(key) mod N` is the textbook answer. But:

- Adding a node moves `(N-1)/N` of the keys. At N=100, you move 99% of data on every resize. **Unusable in production.**
- That's why **consistent hashing + virtual nodes** (Cassandra, DynamoDB, Riak, ZippyDB) are the production answer.

### 3.3 Consistent hashing + vnodes — the production sharding standard

```
       Ring [0, 2^64)
        ┌──────────┐
        │  vnodeA1 │ ← Node A
        │  vnodeB1 │ ← Node B
        │  vnodeA2 │ ← Node A (multiple positions on the ring)
        │  vnodeC1 │ ← Node C
        │   ...    │ ← typically 128–256 vnodes per node
        └──────────┘
                       Each key → hashed to a ring position →
                       owned by the next vnode clockwise.
```

- Each physical node hosts **128–512 vnodes**, each at a random ring position.
- Adding a node: it picks 128 random positions; takes ownership of one slice from each existing node. Total data moved = **1/N** (perfectly balanced).
- Removing/dead node: its 128 slices distribute back to existing nodes.

**Failure modes still happen** at this level:
- **Skew**: bad random seed → vnode positions cluster → some nodes own 2× their share. Mitigation: rendezvous hashing or careful position seeding.
- **Hot vnode**: a single hot key (celebrity user) sits in one vnode → that vnode is overloaded. Vnodes don't help if the *key itself* is hot.
- **Replication interaction**: if RF=3, the 3 replicas are the *next 3 vnodes* on the ring, and these may all live on the same physical node if vnode placement isn't rack-aware. **Always use rack/zone awareness in placement.**

### 3.4 Hot keys — the killer of hash sharding

Even with perfect uniform hashing, **the user keys themselves are not uniformly accessed**. Power-law / Zipfian distributions dominate:

- Top 0.1% of keys may receive **30–50%** of reads (celebrity profiles, viral posts).
- Top 1% receive **70%+** of reads.

A single hot key in a single shard can:
- Saturate that shard's CPU
- Eat the cache eviction budget for the whole shard
- Cause queue buildup that makes the *cold* keys on that shard slow too

**Mitigation toolkit (each has tradeoffs):**

1. **Replicated hot keys (read fan-out)**
   - Detect hot key → replicate it to N shards → reads go to a random replica.
   - Trades **write amplification** for **read scalability**. Strong consistency suffers.

2. **In-memory caching tier (Redis / Memcached / Mcrouter)**
   - Hot key gets pinned in cache; backend sees cache misses only.
   - Adds invalidation complexity; cache itself may have hot shard problem (see §7.4).

3. **Request coalescing / single-flight**
   - If 10K concurrent requests for the same key arrive, do *one* DB read; broadcast result to all waiters.
   - Critical at edge layers (CDN, varnish, proxy).

4. **Probabilistic sampling at write time** for analytics/counter use cases.
   - "View count" updated at sampled rate, not every view.

5. **Sharded counters / write-side fan-out**
   - For a hot *write* key (e.g., "global like counter"): split into N sub-counters, sum on read.
   - Used heavily in counter / metric systems (e.g., FB's Memcached counters, Cassandra "counter" types).

6. **Tenant-aware shedding**
   - At application layer: detect that one customer is generating 90% of QPS on a shard → start shedding/queueing only that customer's traffic. Protects neighbors.

7. **Move the hot key to a dedicated shard**
   - Directory-based sharding lets you do this surgically. Hash sharding doesn't.

**Staff insight**: at sufficient scale, **hot key handling is not "one mitigation" but a layered defense**: cache → coalesce → replicate → shard-isolate → shed.

### 3.5 The directory shard — when to leave consistent hashing behind

When tenants or hot keys vary 1000:1 in size or load, consistent hashing collapses (you cannot move a single hot vnode to a different physical node easily). At that point, consider **directory-based sharding** (used by Slack, Figma, Notion, Vitess, FB Messenger):

```
            ┌─────────────────────┐
            │  Directory service  │  (small, replicated, mostly cached)
            │  key/tenant → shard │
            └──────────┬──────────┘
                       │  (lookup, often per-connection cached)
                       ▼
       ┌────────────┬───────────┬────────────┐
       │  Shard 1   │  Shard 2  │  Shard 3   │
       │ (small     │  (medium  │  (huge     │
       │ tenants)   │  tenants) │  tenant X) │
       └────────────┴───────────┴────────────┘
```

- Pro: full freedom to move/split/merge tenants. You can place "Disney+" alone on dedicated shards while small tenants share a shard.
- Con: directory itself becomes a critical-path lookup (must be highly available, very cached, very fast). Stale directory → wrong shard → drop or proxy → latency hit.

**Staff insight**: directory sharding is a more powerful, but operationally heavier model. Use it for **multi-tenant SaaS** where tenant skew is huge; avoid it for **uniform user-keyed** workloads where consistent hashing + vnodes wins.

### 3.6 Online resharding — the operation that ends careers

You will need to grow capacity. Resharding while live serving 10M RPS is:

- A 6–12 month project for a major cluster
- The single highest-risk operation in your stateful tier
- The reason your initial sharding choice matters so much

The mechanics:

```
PHASE 1 — DUAL WRITES
  Old shards (S1..Sn): receive all writes
  New shards (T1..Tm): receive forwarded writes (fire-and-forget or sync)
  Reads: served from old shards only
  Risk: write amplification; new shards lag. Need to detect divergence.

PHASE 2 — BACKFILL
  Stream historical data from S → T using:
    - Snapshot + change-data-capture (CDC) replay, OR
    - Logical replication / WAL streaming
  Verify: per-key checksum / Merkle tree compare across S and T.
  Watch for: backfill rate too high → CPU/IO saturation on source.

PHASE 3 — DUAL READS / SHADOW READS
  Reads: query both S and T (compare); return S, log mismatches.
  Goal: prove T is correct before any user traffic touches it.

PHASE 4 — CUTOVER (per-shard, per-key, gradual)
  Routing layer: 1% of keys → T, rest → S. Ramp 1% → 10% → 50% → 100%.
  Each ramp step: 24–72 hour soak with anomaly monitors.

PHASE 5 — DECOMMISSION
  Old shards: read-only mode. Then archive. Then delete.
  Never delete on the same day as cutover. Wait at least 7–30 days.
```

**Stuff that bites at staff level:**
- **Schema differences** between S and T: even a "compatible" schema migration can mask a bug. Always compare *semantic* fields, not raw bytes.
- **TTL drift**: keys that expire in S during backfill never make it to T. If your TTL is 24 hours and backfill takes a week, the active set at T is wrong.
- **Tombstones / deletes** in CDC: deletes in S during backfill must propagate to T. Forgetting this is a classic resurrection bug.
- **Invariants across shards**: foreign keys, uniqueness constraints, cross-key transactions. Resharding may violate them mid-migration.
- **Routing layer failure during ramp**: if your router crashes at 50% cutover, a portion of traffic goes to a shard that doesn't have the key. Need a **double-check fallback** path.
- **The "we'll just shrink the dataset first" trap**: deleting old data to make resharding easier... while the system is taking writes. The deletion itself is a heavy stateful operation that competes with the resharding work.

A real production reshard is run by a small **war room** (4–6 engineers) on a runbook with **abort criteria** (e.g., "if mismatch rate >0.01%, halt"). The router is the single point of orchestration.

---

## 4. Replication and Consistency

### 4.1 The CAP/PACELC reframe

At MAANG scale, the crude CAP theorem ("pick 2 of CAP") is too coarse. **PACELC** (if Partition: A or C; Else: L (latency) or C) is the operational reality:

- During a partition, you choose A or C (Cassandra: A; Spanner: C).
- During *normal* operation, you also choose **latency** (replicate async, lower latency, eventual consistency) **or consistency** (sync, higher latency, linearizable).

99.99% of the time there's no partition. So your **steady-state choice is L vs C**, not A vs C. The L/C trade is the one you'll be defending in design reviews.

### 4.2 Replication topologies

```
1. LEADER-FOLLOWER (single-master)
   Writes → leader → replicate (sync or async) to followers.
   Reads → leader (linearizable) or follower (eventual).
   Used by: Postgres, MySQL replicas, Kafka partitions, Redis replicas.

   ┌─────────┐
   │ Leader  │ ──► Follower 1
   │   (rw)  │ ──► Follower 2
   └─────────┘ ──► Follower 3

   Pros: simple consistency story, well-understood.
   Cons: leader is bottleneck for writes; failover is a stop-the-world event.

2. MULTI-LEADER (multi-master)
   Writes can hit any of N leaders. Conflicts resolved via LWW, CRDTs, or app logic.
   Used by: Cassandra (per-key per-DC leader), Couchbase, multi-region Postgres-BDR.

   ┌─────────┐ ◄══► ┌─────────┐
   │ Leader1 │      │ Leader2 │
   │   (rw)  │      │  (rw)   │
   └────┬────┘      └────┬────┘
        ▼                ▼
   followers         followers

   Pros: write scale, geo-locality, partition tolerance.
   Cons: conflict resolution is hard; "lost update" is real.

3. LEADERLESS (Dynamo-style)
   Client sends to any N replicas; reads from R; quorum: W + R > N.
   Used by: Cassandra (in some modes), DynamoDB (under the hood), Riak, ScyllaDB.

   ┌────────┐    ┌────────┐    ┌────────┐
   │Replica1│    │Replica2│    │Replica3│
   └────┬───┘    └────┬───┘    └────┬───┘
        │             │             │
        └─────► coord/client ◄──────┘
        write to W of N    read from R of N

   Pros: no leader to fail over; smoothest tail.
   Cons: tunable but tricky consistency (read repair, hinted handoff).

4. CHAIN REPLICATION
   Writes start at head; flow through chain; reads from tail.
   Used by: ZippyDB-like, FaRM, some object stores.

   Client → Head → Mid → Tail
                              ↓
                          Reads (always linearizable from tail)

   Pros: linearizable reads at tail, no read-amp on writes.
   Cons: single chain throughput; failure of any link halts chain.

5. CRDT-BASED EVENTUAL
   Each replica accepts writes; states merge using CRDT semantics.
   Used by: collaborative editing (Figma, Google Docs), shopping carts (Dynamo classic).

   Pros: no coordination, partition-tolerant, mergeable.
   Cons: only works for data types that have CRDT semantics; no transactions.
```

### 4.3 Quorums — the math you must internalize

For a leaderless system with `N` replicas, write quorum `W`, read quorum `R`:

- **`W + R > N`** ⇒ overlapping read/write set ⇒ at least one replica in the read quorum has the latest write ⇒ **strong consistency**.
- `W + R ≤ N` ⇒ may read stale.
- `W = N, R = 1` ⇒ fast reads, slow writes, can't tolerate any replica down for writes.
- `W = 1, R = N` ⇒ fast writes, slow reads, similarly fragile.
- **`N=3, W=2, R=2`** is the standard "Dynamo" config: tolerate 1 failure, strong consistency, balanced.
- **`N=5, W=3, R=3`** tolerates 2 failures; common for higher-criticality workloads.
- **`N=3, W=3, R=1`** is "bad idea but tempting" — any replica down halts writes.

**Sloppy quorum** (Dynamo-style): if some of the N "preferred" replicas are down, write to any N nodes (including non-preferred), then **hint-handoff** later. Trades availability for consistency boundary clarity.

### 4.4 Consistency model spectrum

```
STRONG / LINEARIZABLE
  ▲   "Looks like a single copy."
  │   Spanner, Aurora-multi-AZ, Postgres sync replica.
  │   Cost: cross-region 2PC or Paxos round-trip = 50–300ms.
  │
SEQUENTIAL CONSISTENCY
  │   All ops in one global order, but not necessarily real-time.
  │
CAUSAL CONSISTENCY
  │   "If A causally precedes B, all observers see A before B."
  │   Used in social feeds, message order. Vector clocks / hybrid logical clocks.
  │
READ-YOUR-WRITES
  │   You see your own writes immediately; others may lag.
  │   Common via session-pinning to primary, or LSN-tracking.
  │
MONOTONIC READ
  │   Each client never sees time go backwards on reads.
  │
EVENTUAL CONSISTENCY
  ▼   "If writes stop, reads converge."
      Cheapest, fastest, semantically loosest.
      DynamoDB default, Cassandra LOCAL_ONE.
```

**Staff insight**: most product features need only **causal** + **read-your-writes** consistency, not linearizability. Engineers who reach for Spanner-grade strong consistency by default are over-paying by 10–100× in latency and infra cost. The skill is matching consistency to the **business invariant being protected**, not to the engineer's anxiety level.

### 4.5 Replication lag — the silent killer

Async replication has a lag, normally sub-second. But:

- **Lag spikes** during write bursts: replica falls behind by tens of seconds, replays slower than primary writes (because replay is single-threaded in many systems).
- **Long-running queries on replica** can pause replay (Postgres: `hot_standby_feedback` interaction with primary `vacuum`).
- **Replay during backup snapshots**: I/O contention — both fight for disk.
- **Network hiccup** on async link: lag compounds; recovery may take hours of catchup.

Mitigations:
- **Parallel replay** (Postgres 14+, MySQL multi-thread replication, Cassandra parallel hint replay)
- **Chunked apply with checkpoints**
- **Saturation alarms** at lag > 10s
- **Read fencing**: refuse reads from replicas that lag > N seconds (and serve from primary or fail open with tagged stale data)

The **read-your-own-writes** problem in async setups: a user posts a comment, then refreshes — their own comment is missing because the read hit a lagging replica. Mitigations: pin to primary for `replica_lag_seconds` after a write, or use `LSN` (log sequence number) tracking — the client sends the LSN it wrote, and the read waits for the replica to reach that LSN.

### 4.6 Geo-replication and the speed of light

```
NYC ↔ LON: ~70 ms one-way, ~140 ms round-trip
NYC ↔ TOK: ~150 ms one-way, ~300 ms round-trip

A 2PC across NYC + LON + TOK = 300+ ms per write. Period.
You cannot beat physics with engineering.
```

Strategies:

- **Async geo-replication** (default for most products): writes commit locally; replicate to other regions in background. Cross-region read sees stale or missing.
- **Quorum-spanning** (Spanner approach): every write requires a Paxos round across geo-distributed replicas. Strong consistency, **but** every write pays geo-RTT. Spanner mitigates with Paxos groups per shard, TrueTime, and selective placement.
- **Tunable per-row**: Cassandra/DynamoDB Global Tables: write to local region, asynchronously replicate; conflict resolution = LWW.
- **Region-pinned tenants**: each user's data lives in *one* region; cross-region access is rare and explicit.
- **Active-active with conflict resolution**: writes in any region accepted; conflicts merged (LWW, CRDT, app-specific).

**Staff insight**: most products do **async geo-replication with region-pinning by user**. Going active-active is a *huge* engineering lift and only justified by specific business needs (e.g., mobile users moving across continents, or compliance-required data residency that still needs availability everywhere).

### 4.7 Conflict resolution — the unglamorous reality of multi-leader

When two regions both accept writes for the same key:

| Strategy | Mechanic | Where it works | Where it fails |
|---|---|---|---|
| **Last-Write-Wins (LWW)** | Compare timestamps; later wins. | Simple, scalable. | Clock skew → silent data loss. Default DynamoDB / Cassandra. |
| **Hybrid Logical Clocks (HLC)** | Wall-clock + logical counter. | Causality preserved. | Slightly more storage; still fragile near clock-jump. |
| **Vector clocks** | Per-replica counters. | Detect concurrent writes → reject or merge. | Storage grows with replica count; merge is app-specific. |
| **CRDTs** | Mathematically guaranteed merge. | Counters, sets, registers. | Doesn't fit complex business logic (think: bank balance with overdraft rules). |
| **App-level reconciliation** | "Manual" resolution. | Anything. | Engineers must own it; bugs expensive. |
| **Conditional writes (compare-and-swap)** | Reject if version drifted. | Strong consistency on hot keys. | Forces serialization; latency hit. |

**Staff insight**: pick conflict resolution **per data class**, not per system. Counters → CRDT; user profile fields → LWW with HLC; financial state → CAS/transactional. Mixing strategies in one schema is normal and necessary.

---

## 5. Storage Engine — Choices and Their Cost

### 5.1 The B-tree vs LSM trade

| Aspect | B-tree (Postgres, MySQL InnoDB) | LSM-tree (RocksDB, Cassandra, BigTable) |
|---|---|---|
| **Read latency** | Predictable, single-shot lookup | Read amp (multiple SST levels), needs bloom filters |
| **Write latency** | Random I/O on update | Sequential append; very fast |
| **Write amplification** | Low | High (compaction rewrites) |
| **Read amplification** | Low | High (multi-SST scans) |
| **Space amplification** | Low (in-place updates) | Medium-to-high (deferred deletes) |
| **Compaction stalls** | None | Real — can cause minute-long write stalls if not tuned |
| **Best for** | Read-heavy, mixed, transactional | Write-heavy, append-mostly, time-series |
| **Memory pressure** | Index in shared buffer pool | Bloom filters + block cache + memtable |

**At MAANG scale:**
- **Write-amplification bills you at the disk-IO level**: a Cassandra cluster doing 1M WPS may do **10M actual disk writes/sec** after compaction. This compounds your IO cost projection. Always model write amp.
- **Compaction storms** are the #1 LSM operational pain. A compaction running on a hot shard causes:
  - Read latency p99 spikes (CPU contention, I/O queuing)
  - Disk space spikes (old + new SSTs coexist) → can OOM disk
  - Cache pollution → reads miss for minutes after
- **B-tree bloat & vacuum**: at the analogous Postgres level, MVCC dead tuples accumulate, autovacuum lags, table bloat hits 2–10× live data, query planner gets confused. Aggressive autovacuum tuning is a full-time concern.

### 5.2 Memtable / WAL / SSTable architecture (LSM in detail)

```
Write path:
  Client write
     │
     ▼
  WAL (sequential append, fsync) ← durability barrier
     │
     ▼
  Memtable (in-RAM sorted map)
     │
     ▼ when memtable full
  Flush → L0 SSTable (immutable, on disk)
     │
     ▼ compaction
  L1 SSTables (size-tiered or leveled)
     │
     ▼
  L2, L3, ... (each ~10× larger than previous)

Read path:
  - Memtable (1 lookup)
  - L0 SSTables (~4-10 files; check each via bloom filter)
  - L1 SSTables (~10 files; bloom filter narrows)
  - L2, L3, ...
```

Key tuning:
- **Memtable size** vs **flush rate**: too small → many flushes → many tiny SSTs → compaction storms. Too big → long fsync stalls and recovery time after crash.
- **Compaction strategy**:
  - **Size-tiered** (Cassandra default): merges similar-size SSTs. Lower write amp but high read amp + space amp.
  - **Leveled** (RocksDB default): each level is ~10× larger. Higher write amp but stable read latency.
  - **Time-windowed** (TWCS, for time-series): merges only within time windows. Best for TTL'd data.
- **Bloom filter false-positive rate**: 1% standard. At the cost of memory you can drive it to 0.1%, doubling the bloom RAM but cutting unnecessary disk reads by 10×.

### 5.3 Page cache, OS layer, and the pitfalls of "just buy more RAM"

- The OS **page cache** holds recently-read disk blocks. Most stateful systems rely on it for ~80%+ of read hits.
- **`sync` vs `fsync` vs `O_DIRECT`**:
  - `sync` (default writeback) → fast but data on disk lags. Can lose seconds on power loss.
  - `fsync` after every write → safe, but ~10× slower. Use selectively (WAL).
  - `O_DIRECT` → bypass page cache. Used by databases that manage their own buffer pool (Postgres can but doesn't by default; MySQL InnoDB does; RocksDB optionally).
- **Madvise tuning**: `MADV_RANDOM` vs `MADV_SEQUENTIAL` vs `MADV_WILLNEED`. Wrong mode = page-cache eviction storms.
- **Huge pages**: 2 MB or 1 GB pages reduce TLB pressure on multi-TB-RAM machines. Mandatory at scale; default kernel setting is wrong.

### 5.4 Tiered storage — hot, warm, cold

At MAANG, you cannot afford to keep all data on NVMe SSD:

```
HOT (≤7 days, 5%):  NVMe SSD, in-cluster  →  sub-ms read
WARM (7–90 days, 30%):  HDD, cluster-attached  →  10–50ms read
COLD (>90 days, 65%):  S3/GCS/Glacier         →  100ms–seconds read

Total: 100% of data; <10% of it on the expensive tier.
```

Mechanisms:
- **Time-series data**: TimescaleDB hypertables, ClickHouse partitioning, Cassandra TWCS.
- **Object stores**: tier old SSTables to S3 (RocksDB has plugins; ScyllaDB Enterprise; AWS Aurora's tiered storage).
- **App-level**: keep recent data in OLTP DB; archive old to data lake (Iceberg/Delta).

**Staff insight**: tiering is an *availability* concern as much as cost. If a product surfaces "old data" via UI, the tier latency must be acceptable. If old data is "rare cold reads", you can tolerate seconds. Define this contract early.

---

## 6. Caching — State Amplifier or State Liability

### 6.1 The cache hierarchy at MAANG scale

```
Client / SDK ─► CDN edge cache (10ms, 100s of locations)
            │
            ▼
        L7 / API Gateway cache (response cache, ETag)
            │
            ▼
        Service-local in-process cache (LRU/LFU, μs)
            │
            ▼
        Distributed cache cluster (Redis / Memcached / Mcrouter, ~1ms)
            │
            ▼
        Application read replica (~5ms)
            │
            ▼
        Source of truth (primary DB)
```

Cache hit ratios at each layer (typical MAANG-mature service):
- CDN: 90–98% for static / hashable URLs
- Gateway/L7: 30–60%
- In-process: 50–80%
- Distributed cache: 95–99%
- DB hit (after all caches): <0.5% of original RPS

This is the only way to land a 10M RPS frontend on a few-thousand-shard backend.

### 6.2 The four cache invalidation patterns

```
1. CACHE-ASIDE (most common)
   read:  app → cache. miss → DB → fill cache → return
   write: app → DB → invalidate (or update) cache
   Pro: simple. Con: race window where DB updated but cache stale.

2. WRITE-THROUGH
   write: app → cache → cache writes to DB synchronously
   Pro: cache always fresh. Con: writes pay both latencies.

3. WRITE-BACK / WRITE-BEHIND
   write: app → cache. cache flushes to DB asynchronously.
   Pro: fastest writes. Con: durability risk if cache dies.

4. READ-THROUGH (cache as "frontend" to DB)
   read: app → cache. cache fetches from DB on miss.
   Symmetric to write-through; cache is "source of API" for the app.
```

### 6.3 Cache stampede / dog-pile

**Scenario**: a hot key expires from cache. 10K concurrent requests all miss → all 10K hit DB → DB falls over.

Mitigations:
- **Single-flight / coalescing**: only one request per key allowed to refetch; others wait.
- **Probabilistic early expiration**: refresh slightly before TTL with randomized jitter, so not all keys expire simultaneously.
- **TTL jitter**: never set fixed TTL across keys. Random ±10% prevents synchronized expiration.
- **Stale-while-revalidate**: serve stale cache while async-refreshing in background.
- **Lock-on-miss**: only one fetcher per key; others get last-known value.

### 6.4 Cache hot-shard problem

Even with consistent hashing on cache cluster, **a single hot key still concentrates on one cache node**. At 1M QPS for one viral post → one Redis shard at 100% CPU.

Mitigations:
- **Replicated hot keys** (top-K detection layer; replicate to N shards; round-robin reads).
- **Edge / in-process cache** for top-K hot keys (handled close to client, not at distributed cache).
- **Mcrouter**-style **L1 (per-host) + L2 (cluster) cache**: every app host has its own small cache; shared distributed cache is L2.
- **Consistent-hashed gutter pool** (Memcached/Mcrouter): on cache server failure, traffic spills to a small "gutter" pool, not the DB.

### 6.5 Cache consistency at scale — the unfixable problem

You cannot have **strict consistency** between cache and DB without writing through both atomically (which costs ~2× write latency). So at scale you accept:

- **Eventual cache consistency**: cache may be stale up to TTL.
- **Invalidation lag**: write commits, cache invalidation propagates, ~10–500ms window of staleness.
- **Failed invalidations**: invalidation message lost → cache stale until TTL. Use **versioned keys** (`key:v3`) so a write bumps the version and old cache entries become unreachable; old entries expire naturally.

A surprising staff-level rule: **"the cache is a hint, not a source of truth."** Any code that *requires* fresh state must go to the DB (and accept the latency). Caches are optimizations on top of correctness, not part of the correctness contract.

### 6.6 The "negative cache" / miss cache

Caching that a key **does not exist** is as important as caching its value:

- Without it, every request for a deleted/non-existent key bypasses cache → DB.
- Used in heavy-attack scenarios (someone scraping with random IDs to pierce your cache).

Trade: must be invalidated when the key is *created*; harder to manage than positive cache.

---

## 7. Hot Keys, Skew, and Load Shedding

### 7.1 Detection — you cannot fix what you cannot see

At MAANG scale, hot keys come from:
- Viral content (post goes viral in 10 minutes)
- Misbehaving client (a buggy mobile build retries one ID 100 times)
- Adversarial behavior (scraping, attack)
- Schema/feature change (new "trending" widget queries top-10 every refresh)

Detection layers:
- **Heavy-hitter sketches** (Count-Min, Misra-Gries) at L7 or sidecar — log keys exceeding a threshold.
- **Per-shard QPS metrics** with key sampling (every 100th request → "what key?").
- **Top-K detection at proxy/cache** (Mcrouter has this).

### 7.2 Mitigation — multi-layered, automatic, tested

A real production "hot key playbook":

```
1. Detection alert fires: key X exceeds 10K QPS on one shard
2. Auto-mitigation (≤1 min):
   - Add key X to in-process cache on every app host (TTL 5s)
   - Replicate to gutter pool
3. Human in loop (5–15 min):
   - If still hot: replicate to N additional shards via routing override
   - If still hot AND user-triggered: throttle that user's requests
4. Post-mortem (next day):
   - Was this a viral event (expected) or a bug (regress)?
   - Should we adjust TTL/replication factor for this content class?
```

The key insight: **automation is required**. A 10M RPS cluster cannot wait for a human at 3 AM to react to a hot key. Your top-K detector must be wired to your cache + routing layer.

### 7.3 Adaptive load shedding

When load exceeds capacity, **drop low-value traffic before high-value**:

```
Priority bands:
  P0 — user-critical reads/writes (login, checkout)
  P1 — interactive (timeline, search)
  P2 — bulk / batch (data sync, reindex, ETL)
  P3 — background analytics, internal tools

Shedding policy under overload:
  >80% capacity:  drop P3 (return 429)
  >90% capacity:  drop P2
  >95% capacity:  drop P1
  >98% capacity:  drop P0 — alarm; this is shouldn't-happen territory
```

Implementations:
- **Per-tenant rate limiting** (token bucket per customer).
- **Adaptive concurrency limiting** (Netflix's `concurrency-limits`, AIMD-based).
- **Queue-based admission control**: stateful service has a finite request queue; when queue depth > threshold, reject new arrivals. Better than letting them queue and time out.
- **Circuit breakers** (per dependency): when DB is overloaded, stop sending more.

### 7.4 The "thundering herd" on dependency recovery

When a downstream service comes back online after an outage, every upstream client retries simultaneously → instant overload → second crash.

Mitigations:
- **Exponential backoff with jitter** (full jitter or decorrelated jitter, not constant backoff).
- **Capacity-aware retry** (slow ramp-up).
- **Circuit breaker half-open state**: only allow a trickle of probe requests before reopening fully.
- **Coordinated retry budgets** (Netflix Hystrix / Envoy) — limit *total* retries cluster-wide.

---

## 8. Write Path Optimization — The Hardest Path

Writes are intrinsically harder to scale than reads because:

- Every write must be durable (fsync/replication).
- Every write usually invalidates caches.
- Every write must propagate to all replicas (read-replica + cross-region).
- Locks/conflicts may serialize writes on hot keys.

### 8.1 Write batching, coalescing, and amortization

Single-write throughput is bound by:
- WAL fsync latency (~1–5 ms per fsync on SSD)
- Network round-trip to replicas
- Index update overhead

**Batching** (group commit) amortizes fsync: 100 writes share 1 fsync → 100× throughput at the cost of slight latency hit.

```
Naive:        write1.fsync, write2.fsync, ... = N × fsync_latency
Group commit: [write1, write2, ..., writeN].fsync = 1 × fsync_latency
              + N × marshalling cost (very fast)
```

Used by Postgres (`commit_delay`), Kafka (`linger.ms`), most LSMs (memtable batching).

### 8.2 Write fan-out — the "social graph problem"

A celebrity user posts → the post must appear in feeds of 100M followers. Two strategies:

**Fan-out on write (push)**:
- On post: write the post ID into the timeline of every follower (100M writes).
- Read: just read your timeline (fast).
- Cost: huge write amplification per post; storage = O(followers × posts).

**Fan-out on read (pull)**:
- On post: 1 write only.
- On read: fetch posts from each followee, merge, sort.
- Cost: huge read amplification; ~hundreds of follower fetches per timeline view.

**Hybrid (production reality)**:
- Fan-out on write for normal users (~1K followers).
- Fan-out on read for celebrities (>1M followers).
- Some celebrities cause cluster-wide load events; need top-K handling.
- Mixed timeline merger handles both.

This is what Twitter (Manhattan + cache fan-out) and Instagram (user-graph cache) actually do. The point: **the choice is workload-driven, not architecture-driven**.

### 8.3 Asynchronous writes and durability classes

Not every write needs the same durability:

| Class | Mechanism | Latency | Loss tolerance |
|---|---|---|---|
| **Sync, replicated, fsync** | RAFT/Paxos commit, fsync on N replicas | 5–50 ms | Zero |
| **Sync, replicated, no fsync** | Replicate but rely on OS cache | 1–10 ms | Seconds (power loss across all replicas) |
| **Async, batched** | Queue + worker | 50–500 ms | Minutes |
| **Best effort** | Fire-and-forget | < 1 ms | Hours/forever |

**Staff insight**: choose durability per write *type*, not per system. "Like" events: best-effort or async. Account creation: sync replicated fsync. Ad clicks: at-least-once via Kafka, dedup downstream. **Mismatching the durability class to the use case is one of the most common over-spend or under-protect failures in stateful design.**

### 8.4 Backpressure as a first-class concept

Every stateful system has a maximum sustainable write rate. Beyond it:

- WAL fills up faster than disk can write
- Memtables can't flush fast enough → flush stalls
- Replicas fall behind → lag alarm → lag panic
- Compaction can't keep up → SST count grows → reads slow → cascading

**You cannot "infinite-buffer" your way out**. Eventually disk fills and the system stalls hard. The right answer is **explicit backpressure to the producer**:

- Kafka: `producer.linger.ms` + `max.in.flight` + ack=all + bounded buffer.
- DB: connection pool with bounded queue → 503 to client when full.
- Stream processor: pull-based (Flink, Kafka Streams) so consumer drives the rate.

A stateful tier without explicit backpressure is a stateful tier waiting for outage.

---

## 9. Read Fan-out and Tail Latency

### 9.1 The fan-out tail-amplification effect

If a request fans out to N backend calls, your overall p99 is *much worse* than a single backend's p99.

```
Single backend p99 = 20ms means 1% of calls take >20ms.
Fan-out of 10:  P(none slow) = 0.99^10 = 0.904, so P(any slow) ≈ 9.6%
                     → fan-out p99 dominated by the 9.6% mark of 20+ms.
                     → fan-out p99 is roughly the single backend's p99.96.

Fan-out of 100: P(any slow) ≈ 63%. Your p99 is now where a single backend's
                     p99.99 is — likely 100–200ms.
```

This is why Google's "tail at scale" paper (Dean & Barroso, 2013) is canonical: at fan-out of 100s, **p99 is dictated by the *long tail* of single-backend latency, not its median**.

### 9.2 Hedged requests / tied requests

Send the same read to two replicas; take the first response.

```
Hedged:
  t=0: send to replica A.
  t=10ms (only if no response): also send to replica B.
  Take whichever arrives first; cancel the other.

Tied:
  t=0: send to A and B simultaneously, with mutual cancellation tokens.
```

- **Wins**: cuts p99 dramatically (you only suffer when both replicas are slow).
- **Cost**: 2× backend load on the hedge path. Tunable via the hedge delay.
- **Caveat**: only applies to **idempotent** reads. Hedged writes are dangerous (must be idempotent + deduped).

This is implemented in Bigtable, Spanner, gRPC `grpc.WithDefaultServiceConfig` hedging policies, and most CDNs.

### 9.3 Speculative execution

Beyond hedged: kick off a backup query *before* you know the primary is slow, based on heuristics (which replica was slow last time, etc.). Used in MapReduce, BigQuery.

### 9.4 Reducing fan-out via denormalization

A 200-fan-out query is often a sign of overly normalized data. Denormalizing — duplicating data so a single read can fulfill a query — trades write complexity for read latency. At scale, **read latency is the user-visible metric**; engineers under-denormalize relative to what scale demands.

Examples:
- Materialized timelines (vs read-time merge across followees)
- Pre-computed friend counts (vs `SELECT COUNT(*) FROM friends`)
- Wide rows in Cassandra: store user's last 100 posts as one row to avoid 100 reads.

### 9.5 Coordinator vs scatter-gather

```
SCATTER-GATHER (typical search/analytics):
  Coordinator → scatter to all N shards in parallel
              → wait for all (or quorum) responses
              → merge / aggregate
              → return to client

  Bottleneck: slowest of N shards.
  Mitigation: hedged requests, partial results (return 95% complete after timeout).

DIRECTED ROUTE (typical OLTP):
  Coordinator → key-aware routing → exactly 1 shard.
  Latency: single backend's latency.
```

**Staff insight**: at scale, scatter-gather queries should be **partial-result tolerant**. Returning "best-effort top 100 from 95% of shards" within SLA is better than blocking on one slow shard.

---

## 10. Failure Modes Specific to Stateful Systems

### 10.1 Split brain

Two nodes both believe they are the leader for a partition:

- **Cause**: leader's lease/lock expired (e.g., during a long GC pause); a new leader was elected; but the old leader's process resumed and didn't notice.
- **Effect**: both accept writes → divergent state → data loss or corruption when merged.
- **Mitigation**:
  - **Fencing tokens**: every write carries a monotonically increasing token; downstream rejects writes with stale token.
  - **Lease refresh with kill switch**: if I can't refresh my lease, I crash myself rather than continue.
  - **Quorum-anchored leadership** (Raft, Paxos): leader cannot accept writes unless it can reach a majority of voters.

### 10.2 Gray failure / brownout

A node is up, responds to health checks, but is slow:

- **Cause**: bad NIC, disk on its way out, GC pressure, kernel issue, partial network partition.
- **Effect**: routing layer keeps sending it traffic; users see tail latency spikes.
- **Mitigation**:
  - **Outlier detection** in load balancer (Envoy, Mcrouter): eject hosts whose error/latency exceeds peers' by N std devs.
  - **End-to-end probes**: synthetic transactions from outside the cluster catch graybus.
  - **Adaptive concurrency**: if a host is slow, its concurrency window shrinks → less traffic.

### 10.3 Cascading failure

Component A overloaded → tries to retry on B → B overloaded → A's retries multiply → meltdown.

- **Mitigation**:
  - **Retry budgets** (Hystrix, Envoy): cluster-wide bound on total retries (e.g., 10% of normal load).
  - **Token-based admission control**.
  - **Bulkheads**: separate pools per tenant so one tenant's overload doesn't drown others.

### 10.4 Correlated failures

You designed for "any one zone failing." Reality:

- A bad config push hits all 3 zones simultaneously.
- A kernel CVE patch crashes 30% of fleet at the same time.
- A bug in your Kafka producer causes 100% of replicas to OOM at once.
- An AWS regional control-plane issue affects all 3 AZs.

Mitigations:
- **Staged rollouts**: 1% canary, 24-hour soak, then 5%, 25%, 50%, 100%.
- **Chaos engineering** (Netflix Chaos Monkey, Gremlin, AWS FIS): regularly fail random instances; verify auto-recovery.
- **Cell-based architecture**: independent "cells" with no shared dependencies. A cell failure caps blast radius. Used by AWS, Slack, and most large SaaS.

### 10.5 Data corruption — the silent killer

Bit-flips, software bugs, hardware errors, replication bugs, **and human actions** (a misissued `UPDATE` without `WHERE`).

- Replication does **not** protect against software bugs or human errors. The bug replicates faithfully.
- **Backup + checksums + delayed replicas** are the only protection:
  - Checksums on every read (Cassandra, RocksDB, ZFS, Btrfs).
  - **Delayed replicas**: a replica that lags by 4–24 hours; gives time to spot a bad change before it propagates.
  - **Point-in-time backups** with retention bands (1 day, 7 day, 30 day, 1 year).
  - **Restore drills**: monthly verification that you can actually restore from backup. Backups you've never restored are not backups; they are theoretical files.

---

## 11. Capacity Planning & Performance Math

### 11.1 Little's Law

```
L = λ × W

L = average number of requests in the system
λ = arrival rate (RPS)
W = average response time

If λ = 1M RPS, W = 10ms = 0.01s, L = 10,000 in-flight requests.
You need at least 10K threads / sockets / connection slots.
```

A staff engineer applies this everywhere:
- **Connection pool sizing**: `pool_size ≥ λ × W`. At 50K QPS and 10ms query, pool of 500.
- **Queue sizing**: queue too small → reject good traffic; queue too big → latency builds up. Sweet spot is `λ × W_target`.
- **Replica count**: replicas per shard = `(read RPS × p99) / (per-replica capacity × utilization)`.

### 11.2 USL (Universal Scalability Law)

Real systems do not scale linearly. The USL says throughput is bounded by:
- **Contention** (α): serialized work that adds with N.
- **Coherence** (β): cross-talk that grows quadratically with N.

```
X(N) = N / (1 + α(N-1) + βN(N-1))

When β > 0, throughput peaks at some optimal N, then declines.
```

A practical implication: adding nodes **decreases** throughput beyond the optimum. This is why you often see better total throughput with fewer, bigger nodes than many tiny ones — fewer cross-node coherence costs.

### 11.3 Working set vs total dataset

```
WORKING SET = data accessed in last W time window (e.g., 1 hour, 1 day)
TOTAL DATASET = everything

If working set fits in RAM: cache hit > 95%, latencies low.
If working set is 2× RAM: page-cache thrashing, p99 explodes.
```

Always size memory by **working set**, not total dataset. The working set is workload-specific (Pareto: ~20% of data gets ~80% of accesses). Measure it with histograms.

### 11.4 Queuing delay and utilization

A single-server queue's average wait time:

```
W = ρ / (μ × (1 - ρ))   where ρ = utilization, μ = service rate

ρ = 50%: wait ≈ 1× service time
ρ = 80%: wait ≈ 4× service time
ρ = 90%: wait ≈ 9× service time
ρ = 95%: wait ≈ 19× service time
```

**Never run stateful systems above ~70% sustained utilization.** The latency tail explodes far before throughput does.

### 11.5 Headroom for bursts

Provision for **2–3× steady-state peak**. The math:
- p99 of arrival rate over 1-min windows is ~2× p50.
- Plus 20–30% buffer for "unexpected" (a feature launch, a bug retry storm, an upstream change).
- Plus capacity to lose 1 zone (in 3-zone setup → 1.5× capacity per zone).

So a "1M sustained RPS" stateful tier provisions for ~2–3M peak.

### 11.6 The unit-economics number every staff engineer carries

```
Cost per 1M requests, broken down:
  - Compute (per-shard CPU)
  - Storage (working set in RAM + cold on disk + archive)
  - Network (cross-AZ ingress/egress is the surprise budget item)
  - Replication (3× the obvious data cost)
```

At MAANG scale, **AZ-egress** alone can be 20–30% of total stateful tier cost. Avoid cross-AZ chatter where possible (place replicas across AZs but route reads in-AZ).

---

## 12. Multi-Region — The 100x Cost Multiplier

### 12.1 Why multi-region

- **Latency**: serve users from closer region (RTT savings on every request).
- **DR / availability**: survive a region failure.
- **Compliance**: data residency (EU GDPR, India PDP, Russia, China).
- **Capacity**: a single region runs out of headroom.

### 12.2 Topology choices

**Active-passive (failover)**:
- One region serves; another is warm-standby, getting async replication.
- On failure: DNS failover or traffic-manager flip; promote standby.
- **Pro**: simple, deterministic.
- **Con**: standby capacity is paid-for-but-idle 99% of the time. Failover practiced rarely → fails when needed.

**Active-active, region-pinned (preferred for most user data)**:
- User is *primary* in one region; replicated to others.
- Reads in-region (most are). Cross-region rare.
- **Pro**: all regions earning their capacity; locality wins.
- **Con**: cross-region operations (a user moves) are heavy.

**Active-active, multi-master, conflict-resolved**:
- Any region accepts writes for any key; conflicts merged.
- **Pro**: maximum availability.
- **Con**: conflict semantics; LWW data loss; "pick one" reconciliation.

**Spanner-like (synchronously consistent across regions)**:
- Writes commit only after Paxos round across regions.
- **Pro**: linearizable global view.
- **Con**: 100ms+ write latency; complex to operate (TrueTime, careful Paxos group placement).

### 12.3 Region failover — the rehearsal that breaks first

A region fails. You promote the standby. Things that go wrong:

- DNS TTL too long → clients still hitting dead region for hours.
- Connection pools/load balancers cached old endpoints.
- Promoted region wasn't actually fully replicated (lag was 30 min); some writes lost.
- The promoted region didn't have capacity for both regions' traffic → it crashes too.
- Schema versioning between regions diverged (a migration was applied in one but not the other).
- Failover changes the write region; clients writing to the old leader get "no longer leader" errors that are not gracefully handled.

**The only way to make region failover work** is to do it in production, on schedule, every quarter. The cost of "we do it once a year" is that it never works when it actually matters. AWS, Google, and Netflix all run regular forced-failover drills.

### 12.4 Data residency and split-brain compliance

If user data must stay in EU and another copy must stay in US:
- Split your dataset by region with no replication in between.
- Cross-region reads: explicit, possibly cached locally with masking.
- Cross-region writes: blocked or proxied with audit logs.
- A user moving regions: a deliberate, supported migration operation, not free movement.

This is operationally heavy but legally mandatory in many jurisdictions.

---

## 13. Online Schema Migrations & Evolution

### 13.1 Why "ALTER TABLE" is illegal at scale

A naive `ALTER TABLE` on a 10 TB table:
- Takes minutes to hours.
- Holds an exclusive lock → blocks reads/writes.
- May not finish before timeout.
- Replicas re-replay the migration → cascade of slowness.

At MAANG scale, **all schema changes are online, multi-step, and reversible**.

### 13.2 The expand–contract pattern

```
PHASE 1 — EXPAND
  Add new column / new table / new index, **nullable** or **default**.
  Existing code unaware of it.
  Cost: O(small) per row at ALTER time, or batch backfill.

PHASE 2 — DUAL WRITE
  Code writes to both old and new fields simultaneously.

PHASE 3 — BACKFILL
  Async job copies old → new for existing rows.
  Verify: per-row checksum equality. Tolerate skew, repair, repeat.

PHASE 4 — DUAL READ
  Code reads from new field; falls back to old; logs mismatches.

PHASE 5 — CONTRACT
  Stop dual-writing; remove old field; clean up.

Each phase = a separate deploy. Some phases live for weeks.
```

This pattern applies to schema changes, system migrations, sharding changes, even sometimes API changes. **Never collapse phases**; the safety lies in the gradualism.

### 13.3 Tools and patterns

- **gh-ost / pt-online-schema-change** (MySQL): create shadow table, copy rows in chunks, apply concurrent changes via triggers, atomic rename swap.
- **pg_repack / pg_squeeze** (Postgres): rebuild table without long lock.
- **Schema registries** (Confluent Schema Registry for Kafka, Protobuf evolution rules): make schema evolution a build-time concern, not a runtime panic.
- **Database-as-a-service migrations** (Aurora, Spanner): often add features to mitigate schema-change pain.
- **Application-side polymorphic reads**: code handles "old shape OR new shape", removing the deploy/migration coupling.

---

## 14. Multi-Tenancy and Isolation

A single stateful tier serving 100K customers (B2B SaaS) or 1B users (B2C) needs **isolation** so:

- One tenant's runaway query doesn't slow the others
- One tenant's data growth doesn't fill the cluster
- One tenant's outage doesn't affect others
- Different tenants can have different SLAs

### 14.1 Isolation models

**Shared everything (pooled)**:
- All tenants in the same DB / cluster / shards.
- Row-level filtering by `tenant_id`.
- **Pro**: cheapest per tenant.
- **Con**: noisy neighbor; data leakage risk; per-tenant SLA impossible.

**Shared cluster, separate schema/database**:
- One DB cluster; per-tenant schema.
- **Pro**: row-level isolation easier; per-tenant ops (backup) clearer.
- **Con**: schema sprawl; many DBs in one cluster has its own management cost.

**Cell / shard per tenant tier**:
- "Free" tenants share a shard; "premium" get dedicated; enterprise tier gets dedicated cluster.
- **Pro**: graduate isolation by SLA.
- **Con**: tenant placement is a routing concern; movement is operational.

**Cluster per tenant**:
- Each customer = own cluster.
- **Pro**: total isolation; perfect SLAs.
- **Con**: cost-per-tenant >> shared model. Used only for large enterprise.

### 14.2 Quotas and noisy neighbor protection

- **Per-tenant rate limits** at API layer.
- **Per-tenant query concurrency limits** (max simultaneous queries).
- **Per-tenant resource accounting** (CPU seconds, IO bytes).
- **Slow-query timeouts** so one query can't run forever.
- **Workload manager** (e.g., Postgres `RLS` + resource queues, MySQL Resource Groups).

### 14.3 The "Disney+ tenant" problem

In any large multi-tenant system, ~1% of tenants are 100–10000× larger than the median:

- Single tenant might exceed a single shard's capacity.
- Their migrations / backfills take days.
- Their SLAs are higher.
- Their queries are bigger.

Treat **mega-tenants** as their own architectural constraint. Place them on dedicated shards/clusters early, before they break shared infra.

---

## 15. Disaster Recovery and Backups

### 15.1 RPO and RTO — the lingua franca

- **RPO (Recovery Point Objective)**: how much data are we OK losing? "5 minutes" → backups every 5 minutes.
- **RTO (Recovery Time Objective)**: how fast must we be back online? "1 hour" → DR plan must complete in 1 hour.

**Stateful systems often advertise RPO=0, RTO=1min** for primary regions, but achieve it via replication, not backups. Backups protect against **logical corruption** (bug, human, ransomware), not hardware loss.

### 15.2 Backup strategies

| Strategy | Granularity | Recovery time | Storage cost |
|---|---|---|---|
| **Full snapshot daily** | 1 day | Hours | High |
| **Incremental + WAL archive** | Seconds (point-in-time) | Hours | Medium |
| **Continuous archiving (PITR)** | Seconds | Hours | Medium |
| **Streaming backup to object store** | Real-time | Hours | Low |
| **Logical backups (`pg_dump`)** | Per-table | Days for large data | Medium |

At scale, **incremental + WAL/log archive** is the default, with full snapshots weekly.

### 15.3 Restore drills — non-negotiable

Things that go wrong in restores you didn't drill:

- Backup is encrypted with a key in a system that depends on the DB you're trying to restore. (Yes, this happens.)
- Restore takes 18 hours, not "a few hours." Your RTO is dead on arrival.
- Restore succeeds but the index isn't rebuilt → reads are 100× slower.
- Restored data references foreign IDs that no longer exist in dependent systems.
- Restored cluster's password is in the secrets store you also lost.

**Quarterly restore drills**, blind, on a clean environment, reading real backup data. Anything else is theater.

### 15.4 Defense against logical corruption

- **Delayed replicas**: 4–24 hour lag. Spot a bad change, kill the apply, restore from this replica.
- **Versioned backups** with retention bands.
- **Audit logs / WAL retention** for forensics.
- **Schema-aware diff tools** to compare backups.
- **Soft delete** by default in business logic; hard delete via TTL/garbage collection.

---

## 16. Observability — You Cannot Operate What You Cannot See

### 16.1 The three pillars (and the per-shard one)

1. **Metrics**: Prometheus-style time-series. Per-shard, per-tenant, per-host.
2. **Logs**: structured, queryable, retained for 7–30 days (longer for audit).
3. **Traces**: distributed traces across the call graph.
4. **(Stateful-specific) Per-shard health view**: CPU, RAM, disk, write rate, compaction backlog, replication lag, GC, queue depths, connection pool fill.

### 16.2 The metrics you actually need

For each shard:
- **Read/write QPS**: total, per-tenant, per-key-class (top-K).
- **Latency**: p50/p95/p99/p999 for read & write, separately.
- **Working set hit ratio**: page cache, internal cache, distributed cache.
- **WAL fsync latency**: tail of write durability.
- **Replication lag**: per-replica.
- **Compaction backlog**: count of pending SSTables, total bytes pending.
- **GC pause distribution** (if relevant).
- **Connection pool**: active, idle, waiting, rejected.
- **Disk**: free %, read/write IOPS, queue depth.
- **CPU**: total, system, iowait.

### 16.3 Cardinality — the trap

Per-tenant, per-key, per-region metrics combine into thousands or millions of time series. Cardinality blows up Prometheus. Solutions:
- Sample / aggregate at tail (heavy-hitter sketches).
- Streaming-aggregated metrics (drop tags below threshold).
- Push to a tier optimized for high cardinality (M3, VictoriaMetrics, Cortex).

### 16.4 Per-shard SLO dashboards

A staff-level mistake is to look at **cluster-level p99**. At 10K shards, the cluster p99 hides 100s of broken shards (because it's a percentile of the 10M-RPS aggregate). You need:

- **Per-shard p99**, alarming when any shard exceeds threshold.
- **Top-N slowest shards per minute**.
- **Per-tenant SLO compliance**.
- **Heatmaps**: shard ID × time × p99. Bad shards stand out as horizontal hot rows.

### 16.5 Forensics and tracing

When something is wrong:
- Trace IDs end-to-end, propagated through every layer (CDN → API → service → DB).
- Slow-query logs with parameter values (PII-redacted).
- WAL position trackers per replica.
- Distributed traces with sampling at the high-latency tail (head-based sampling misses these).

---

## 17. Common MAANG Patterns — Concrete Designs

### 17.1 Facebook TAO — graph store fronted by cache

```
              ┌─────────────────────────────┐
              │  TAO API (read & write)     │
              └──────────────┬──────────────┘
                             │
              ┌──────────────┴──────────────┐
              │   Cache tier (graph cache)  │  ← 99% read hit
              │   per-region, sharded       │
              └──────────────┬──────────────┘
                             │ (on miss / write)
              ┌──────────────┴──────────────┐
              │   MySQL shards (per-region) │  ← source of truth
              │   sharded by object id      │
              └──────────────┬──────────────┘
                             │
                       Async replication
                             │
              ┌──────────────┴──────────────┐
              │   Read-only replica regions  │
              └──────────────────────────────┘
```

- Reads served from cache (hit ratio >99%).
- Writes propagate cache → MySQL → cross-region replication.
- Eventual consistency across regions; per-user "read your write" guaranteed by routing to user's home region.

### 17.2 DynamoDB — hosted Dynamo lineage

```
Partition by key → consistent hash → replica set (typically 3 replicas).
Replicas: Paxos-coordinated for strong-consistent option.
Storage layer: log-structured, with separate compaction.
Adaptive capacity: hot partitions can borrow capacity from cold ones.
Global tables: async cross-region replication, LWW conflict resolution.
```

Operational design choices visible to architects:
- **Partition key design** is the most important schema decision (low cardinality = hot partition).
- **Item size** capped (1MB) → forces normalization or item-grouping.
- **GSI** (global secondary indexes) propagate eventually; new GSI is itself a multi-hour reshard.

### 17.3 Spanner — globally consistent SQL

```
Tablet: a range partition of a table.
Each tablet: replicated via Paxos across geo-distributed replicas.
TrueTime: GPS+atomic-clock-backed bounded uncertainty interval for timestamps.
   Reads at a TrueTime past the uncertainty interval are linearizable.
```

The trade: any write that needs strong consistency pays a Paxos round-trip across regions. Spanner mitigates by:
- Fine-grained Paxos groups (per tablet)
- Reads that don't need strong consistency: served from a single replica without coordination
- Read-only transactions at chosen timestamps: coordinator-free, very fast

### 17.4 Cassandra — masterless wide-column

```
Partition key → consistent hash ring → coordinator routes to RF replicas.
Per-DC replication: writes propagate to remote DCs async.
Consistency level: tunable per query (ONE, QUORUM, LOCAL_QUORUM, ALL).
Storage: LSM (memtable → SSTables → compaction).
Repair: anti-entropy via Merkle tree comparisons.
```

Pain points:
- Tombstones: deletes are markers, not actual removals. Heavy tombstone load → query slowness; eventually GC'd.
- Read repair: lazy consistency mechanism; can cause coordinator load on first reads after partition.
- Compaction storms: well-known operational pain.

### 17.5 Kafka — log-as-database

```
Topic = ordered log, partitioned.
Each partition: leader + N followers (in-sync replicas, ISR).
Producer: writes to leader, fsync, replicate to ISR.
Consumer: pulls from leader (or followers in newer versions, "fetch from follower").
Offset: per-(consumer-group, partition).
```

Used at MAANG for:
- Event sourcing (write to Kafka first, derive everything downstream).
- Change data capture (DB → Kafka → consumers).
- Stream processing (Kafka Streams, Flink).
- Microservice de-coupling.

Edge cases:
- **In-sync replica shrinkage**: under load, slow followers drop out; ISR shrinks; producer's `acks=all` becomes weaker.
- **Rebalance storms**: a consumer group adds/removes a member → all partitions reassigned → minutes of pause.
- **Compacted topics** (key-based deduplication): compaction is a background job that competes with ingest.

### 17.6 BigTable / HBase — wide-row LSM at scale

- Row-key sorted; range queries cheap.
- Tablet = row range; auto-split when too large.
- LSM storage in DFS (GFS / HDFS / Colossus).
- One server per tablet; failover = reassign the range.

Excellent at: time-series, social graphs, telemetry, analytics. **Not** transactional; no joins.

---

## 18. Patterns at the Boundaries — Where Stateful Meets Stateless

### 18.1 Event sourcing + CQRS

```
WRITES → events → durable log (Kafka, EventStore, Postgres outbox)
                              │
                              ├─► state projection 1 (read-optimized)
                              ├─► state projection 2 (analytics)
                              └─► state projection 3 (search index)
```

- Source of truth = the log.
- Multiple read models, each optimized for a query.
- Recovery = replay log into projection.
- **Pain**: log grows forever; need snapshots / compaction. Schema evolution of events is permanent.

### 18.2 Outbox pattern (transactional event publishing)

To atomically commit a DB write *and* publish an event:

```
TX BEGIN
  INSERT into orders ...
  INSERT into outbox_events ...
TX COMMIT

Background worker: read outbox → publish to Kafka → mark sent.
```

Avoids the dual-write problem (DB committed, event publish failed → divergent state). Used widely for reliable event publication from stateful tiers.

### 18.3 Saga pattern

Long-running cross-service workflows without distributed transactions:

```
Step 1: charge card → ok
Step 2: reserve inventory → ok
Step 3: create shipping → FAIL
Compensating actions: refund card, release inventory.
```

Implemented via **state machines** (AWS Step Functions, Temporal, Cadence) backed by stateful storage. The state machine itself is a stateful system — its durability and failover requirements are MAANG-grade in mature systems.

### 18.4 Idempotency keys

For at-least-once delivery semantics:

```
Client sends a unique key with each request.
Server: dedup table (key → (response, timestamp)).
Duplicate request → return cached response.

Storage cost: ~24-48hr retention of dedup keys.
```

A first-class invariant in payment, ordering, messaging systems. Without it, retry-on-failure causes double-charges.

---

## 19. Anti-Patterns — Staff-Level Red Flags

These come up in design reviews. A staff engineer learns to spot them in seconds.

### 19.1 "We'll just shard later"

Building a system on a single DB and assuming sharding can be added later. **It is the single largest engineering project you will not budget for.** Shard early or design for sharding upfront, even if you start with N=1.

### 19.2 "We'll use distributed transactions"

2PC across services or shards: high latency, blocking on coordinator failure, all-or-nothing visibility issues. Almost never the right answer at scale. Sagas + idempotency are the right tool.

### 19.3 "We'll pin everything to the leader"

Your strong-consistency answer becomes a single bottleneck. Every read goes to the leader; the leader's CPU/IO is the cluster's ceiling. This is fine until ~10K QPS; beyond, you must split reads to replicas.

### 19.4 "We'll cache for consistency too"

Treating cache as the source of truth. The cache lies sometimes; the DB is truth. If the business invariant cannot survive a cache miss returning stale data, the read must hit the DB.

### 19.5 "We'll use Spanner / global strong consistency for everything"

Most data does not need it. Default to causal + read-your-writes. Reach for global consistency only for the small slice of data that needs it (financial state, account ownership, inventory in oversold-prone systems).

### 19.6 "We'll add more cache for the hot key"

Adding cache layers without addressing fan-out / hot-key pinning just moves the problem. The hot key is *one node's* problem at every cache layer too.

### 19.7 "We'll just retry on failure"

Without backoff, jitter, retry budgets, idempotency, this is a recipe for cascading failure on partial outages. Retries amplify load 2–10× during the worst time to amplify load.

### 19.8 "We'll do the migration in one step"

Schema migrations, data migrations, system migrations: never do them in one step. Always expand–contract. Always with a rollback path. Always with shadow reads/writes for validation.

### 19.9 "We don't need backup; we have replication"

Replication faithfully replicates corruption. A misissued `DELETE` on the leader replicates to all replicas in seconds. Backups + delayed replicas are independent protections.

### 19.10 "We'll figure out monitoring after launch"

You will not. The data needed to debug a stateful production incident must be already-in-place at the moment of incident — adding it after is too late, and shows you don't operate the system you're building.

### 19.11 "Strong consistency is automatic in our database"

It depends on configuration: your replication mode, quorum settings, fsync policy, isolation level. Most defaults are "fast and slightly wrong" rather than "slow and correct." Always verify.

### 19.12 "Eventually consistent, so we don't need to think about ordering"

Causal ordering matters for human-perceived correctness. A user sees "you replied" before "you said" → confusing. Hybrid logical clocks or per-conversation orderings are not optional in messaging/feed systems.

### 19.13 "We'll just split that table"

Splitting a table at scale is itself a multi-month project. The decision to "just split" hides resharding, dual-writes, cutover, validation, decommission. Estimate 2–4 quarters per major shard split.

### 19.14 "GC is fine, our heap is small"

GC pauses on a 64GB heap can be 5–30 seconds. Your replication lease was 10s. Now you have split-brain. GC is a system-design constraint, not a JVM tuning concern.

### 19.15 "We don't need region failover until we go multi-region"

By the time you go multi-region, the dependency graph is so dense that retro-fitting failover is impossible. Plan the failover topology before you place the first byte in the second region.

---

## 20. The Decision Framework — How a Staff Engineer Reasons in Real Time

When asked to design or evolve a stateful system, walk this framework:

### 20.1 Step 0: Establish the contract

- **What is the read/write QPS, today and projected 2 years out?**
- **What is the dataset size, working set, growth rate?**
- **What is the latency SLO (p50, p99, p999)?**
- **What is the availability SLO and the failure budget?**
- **What invariant must hold — financial accuracy? Causal ordering? Read-your-writes?**
- **What jurisdictions / data residency rules apply?**
- **What is the operational team's capacity (people-hours/year for this system)?**

These constraints rule out 80% of the design space immediately. Engineers who skip this step build the wrong thing.

### 20.2 Step 1: Pick the consistency / availability point

```
If financial / inventory invariant: strong / linearizable.
If user-facing UX (feed, profile): causal + read-your-writes.
If telemetry / analytics: eventual.
If real-time messaging: causal + monotonic.
```

This decision pre-shapes your replication & quorum choices.

### 20.3 Step 2: Pick the sharding strategy

```
If keys are uniform and pure key-value: consistent hash + vnodes.
If skew is huge or tenants vary: directory-based.
If access pattern is range/time: range partitioning.
If geo-bound users: shard by region first, then by key.
```

### 20.4 Step 3: Pick the storage engine

```
Read-heavy, transactional, complex queries: B-tree (Postgres / MySQL).
Write-heavy, append-mostly, time-series: LSM (Cassandra, RocksDB, Kafka).
Mixed, low-latency, OLTP+OLAP boundary: Aurora / Spanner / TiDB / CockroachDB.
Search/analytics: dedicated (Elasticsearch, ClickHouse, Pinot, Druid).
```

### 20.5 Step 4: Design the cache hierarchy

```
CDN ↔ gateway ↔ in-process ↔ distributed cache ↔ DB
Decide which layer owns invalidation.
Define stale-tolerance per data class.
Plan for hot-key replication and stampede prevention.
```

### 20.6 Step 5: Plan the failure modes

```
For each: leader loss, zone loss, region loss, replica lag, hot key, hot tenant,
correlated bug, data corruption, GC pause, network partition, gray failure.
Define detection, mitigation, recovery.
Then run game-day drills against each.
```

### 20.7 Step 6: Plan the operational lifecycle

```
- Resharding plan (when, how, who).
- Schema evolution plan (expand-contract, tooling).
- Backup/restore drills (cadence, ownership).
- Observability scaffolding (per-shard, per-tenant, SLOs).
- Capacity model (forecast, expansion playbook).
- Multi-region promotion plan (drills).
- Cost model (unit economics, allocation by tenant/feature).
```

A design without this is incomplete.

### 20.8 Step 7: Estimate the people-cost

Stateful systems are people-expensive:
- Steady-state: 2–4 engineers/cluster of significant scale, plus on-call rotation.
- Sharding/migration project: 4–10 engineers × 6–12 months.
- Multi-region launch: 6–10 engineers × 6 months.
- DR program: 1–2 engineers ongoing.

If your headcount can't support this, your architecture is wrong (too custom). Use managed services, simplify, or descope.

---

## 21. Mental Models a Staff Engineer Carries

A condensed reference of mental models that produce correct staff-level reasoning quickly:

1. **State has gravity.** Compute moves cheap; data does not. Architectures form around data placement.

2. **Consistency is a budget.** Spending it is a cost (latency, complexity). Budget per data class.

3. **Tail dominates at scale.** p99 of one call is irrelevant; p99 of a 100-fan-out is the user experience.

4. **Steady state is a lie.** Systems live in transient states (rebalances, deploys, GC, compaction, partial outages). Design for the transient.

5. **Failure modes compose.** Two independent 99.9% systems serially = 99.8%. Composition reduces; redundancy multiplies. Know which structure you have.

6. **Backpressure is the only sustainable shedding.** Anything else is unbounded queueing or hidden retries.

7. **You will operate this for 5–10 years.** Day-1 cleverness becomes year-3 burden. Optimize for clarity and observability first.

8. **Migrations are the hard part, not the steady-state.** Most engineering effort over a system's life is in evolving it, not running it.

9. **The boring choice is usually right.** Postgres + sharding + cache + Kafka covers ~95% of MAANG-scale workloads. Reach for exotic when there's a clear reason.

10. **Latency budgets compose.** Each layer takes some of the 200ms total. Design top-down, allocate, then verify each layer fits.

11. **Caches are hints; durable stores are truth.** Code that requires correctness must pierce the cache (and accept the cost).

12. **Hot keys are inevitable. Plan for them.** Detection + automation + multi-layer mitigation is a must-have, not nice-to-have.

13. **Pre-mature optimization is the second-worst disease. Premature pessimization is the worst.** Baseline quickly; measure; then tune.

14. **Operational cost is engineering cost.** A "simpler architecture" that requires 24/7 hot-fixes is not simpler.

15. **Drills > documentation.** A runbook untouched in 12 months is fiction. Quarterly DR drills are the only way to know your DR works.

16. **Cells contain blast radius.** A cell-based architecture lets you survive failures of components you don't control. Single-cluster designs above ~1M RPS are a liability.

17. **Always have a rollback path.** Every deploy, every migration, every config change. The ability to revert in <5 minutes is more valuable than the change itself.

18. **Capacity planning is a function of skew, not mean.** Plan for the worst shard, not the average.

19. **The most expensive byte is the one in the wrong region.** Cross-region traffic, especially repeated cross-region writes, dominates cost at scale.

20. **The team's psychological safety is part of the SLA.** A team afraid to deploy or to fail-fast cannot operate a 1B-user system. Healthy on-call and blameless culture are not soft skills; they are infrastructure.

---

## 22. Interview-Style Framing (Bonus, How to Argue This in a Loop)

When a system-design interview asks "design Twitter timeline / Instagram feed / WhatsApp / X at 1B users":

1. **Establish scale numbers** in 90 seconds: DAU, RPS read/write, dataset size, latency SLO. *Don't accept "1B users" as the only spec — derive concrete numbers.*

2. **Identify the dominant access pattern** — is this read-heavy, write-heavy, append-mostly, query-heavy? *Architecture follows the dominant pattern.*

3. **Make the consistency call explicitly**: causal + read-your-writes for most products, strong only for inventory/payments/identity.

4. **Pick sharding strategy** and explicitly call out hot-key strategy. *Hot keys are the most missed dimension by mid-level candidates.*

5. **Add the cache hierarchy** with specific TTLs and invalidation discipline.

6. **Address the fan-out** for write or read, hybrid for celebrities.

7. **Discuss replication topology** and quorums; pick concrete numbers.

8. **Discuss failure modes**: leader loss, hot key, hot shard, region failure, schema migration. Demonstrate you've thought past "happy path."

9. **Discuss the lifecycle**: rollout plan, observability, on-call, capacity plan, DR.

10. **Acknowledge the operational reality**: how many engineers, how often deploys, how often resharding. *Senior engineers describe a system; staff engineers describe a system **plus the team that runs it.***

That last point is the difference between an excellent senior interview and a passing staff one.

---

## 23. Closing Note — The Staff Mindset

The technical content above matters, but the staff-level shift is in **what you optimize for**:

- A senior engineer optimizes for the system delivering its stated function correctly.
- A staff engineer optimizes for the **organization's ability to evolve the system over a decade**, including:
  - When the requirements change (and they will)
  - When the team changes (and they will)
  - When the load grows 10× (and it will)
  - When the dependencies shift (cloud provider, language, framework)
  - When someone makes a mistake at 3 AM and we need to recover gracefully

That is the long-game. Architectures that look "elegant" in a design doc but require constant heroic operation are bad architectures. Architectures that look "boring" but absorb growth, failures, and change without drama are good architectures.

At MAANG scale, **boring is a feature**. The mature engineer's job is to make a 10M-RPS, 1B-user, multi-region, stateful system feel boring to operate. Everything in this document is in service of that goal.