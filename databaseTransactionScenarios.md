# Database Transactions — Realistic Scenarios at Staff-Engineer Depth

> A practical reference for when to reach for a transaction, what isolation level to choose, what failure modes the choice introduces, and what scalable alternatives exist when the textbook `BEGIN…COMMIT` doesn't survive contact with 1B-user reality. Every scenario below is one I (or someone in my org) has shipped, broken, fixed, and broken again.

> Companion to `statefulSystemsAtMAANGScale.md` and `statelessSystemsAtMAANGScale.md`. Where those describe the *system* tiers, this describes the *correctness primitive* that connects them — the database transaction — and the dozens of ways its semantics are abused, misunderstood, or worked around at scale.

---

## 0. The Staff-Level Frame

A transaction is a **correctness contract** with a runtime cost. The naive answer to "should I use a transaction?" is "if you need atomicity, yes." The staff answer is:

1. **What invariant am I trying to preserve?** (Atomicity ≠ correctness; many invariants don't require atomicity.)
2. **What's the cost of that invariant at my scale?** (Locks, MVCC bloat, deadlocks, WAL pressure, replica lag, throughput cap.)
3. **What's the cheapest mechanism that preserves it?** (Sometimes a transaction; sometimes idempotency keys, CAS, sagas, event sourcing, CRDTs, reservation tables, LWW with versioning.)
4. **What goes wrong when this scales 100×?** (Hot row contention, deadlock storms, connection pool exhaustion, long-running transactions choking VACUUM.)
5. **Who maintains it?** (A 4-line `BEGIN…COMMIT` is forever; a 4-service saga is forever-with-on-call rotations. Both have a cost.)

Engineers reach for transactions because they're easy to reason about. Staff engineers reach for them when they're the right tool, knowing the bill that comes with each use.

---

## 1. Quick Refresher (so we share vocabulary)

### 1.1 ACID

```
A — Atomicity:    all-or-nothing on commit/abort
C — Consistency:  invariants defined by schema + app preserved
I — Isolation:    concurrent txns appear serial (to varying degrees)
D — Durability:   committed data survives crash
```

### 1.2 Isolation levels (SQL standard, what most engines actually do)

| Level | Dirty read | Non-repeatable read | Phantom | Lost update | Write skew | Cost |
|---|---|---|---|---|---|---|
| **Read Uncommitted** | Yes | Yes | Yes | Yes | Yes | Cheapest |
| **Read Committed** (default in Postgres, Oracle) | No | Yes | Yes | Yes (in race) | Yes | Cheap |
| **Repeatable Read / Snapshot** (default in MySQL InnoDB) | No | No | No (in PG MVCC) | Detected via SI* | Yes | Medium |
| **Serializable Snapshot Isolation (SSI)** (PG) | No | No | No | No | No | Higher |
| **Strict Serializable** (Spanner, Aurora multi-AZ via Paxos) | No | No | No | No | No | Highest |

*PG uses **Snapshot Isolation** for "repeatable read"; "serializable" is SSI which detects dangerous read-write cycles.

### 1.3 Anomalies in plain English

```
DIRTY READ:           Read someone else's uncommitted change.
NON-REPEATABLE READ:  Same query in same txn returns different rows because
                      another txn changed them.
PHANTOM READ:         New rows appear in a re-issued range query (someone
                      INSERTed into your range).
LOST UPDATE:          Two txns read X, both write X + 1. Final value is X + 1,
                      not X + 2.
WRITE SKEW:           Two txns each read consistent snapshot, each modify
                      different rows in a way that violates an invariant
                      (e.g., "at least one doctor on call" — both go off-call
                      because each thought the other was still on).
PHANTOM WRITE:        Insert that violates a predicate-based invariant
                      another txn relied on.
```

These anomalies are what you're paying isolation cost to prevent. Pick the cheapest level that prevents the ones you actually have.

### 1.4 What "scalable" means for transactions

```
- Single-row throughput, no contention:    1M+ TPS (basically free)
- Single-row hot contention:                10–1K TPS (lock-bound)
- Multi-row, same shard:                    10K–500K TPS
- Cross-shard 2PC:                          100–10K TPS (RTT-bound)
- Cross-region strong consistency:          10–1K TPS (geo-RTT-bound)
```

A 4-row transaction's throughput is bounded by the slowest single row's contention, plus the WAL fsync latency. Plan accordingly.

---

## 2. The Scenario Taxonomy

```
┌────────────────────────────────────────────────────────────┐
│ AXIS 1: Number of rows touched                              │
│   Single row  —  Few rows  —  Many rows  —  Cross-table     │
│                                                              │
│ AXIS 2: Number of partitions/shards                         │
│   Single shard  —  Few shards  —  Many shards  —  Multi-DB  │
│                                                              │
│ AXIS 3: Number of services involved                         │
│   1 service  —  2-3 services  —  N services (saga)          │
│                                                              │
│ AXIS 4: Latency budget                                      │
│   Real-time (ms)  —  Interactive (100ms)  —  Async (sec+)   │
│                                                              │
│ AXIS 5: Conflict probability                                │
│   No conflicts  —  Occasional  —  Hot row  —  Always        │
│                                                              │
│ AXIS 6: Durability requirement                              │
│   Best-effort  —  Standard  —  Financial-grade              │
└────────────────────────────────────────────────────────────┘
```

Scenarios below are organized by which combination of these axes dominate the design.

---

## 3. Scenario 1 — The Canonical Money Transfer

**The example everyone uses, but rarely correctly.**

### 3.1 The problem

```
Transfer $100 from account A to account B.

Naive:
  UPDATE accounts SET balance = balance - 100 WHERE id = A;
  UPDATE accounts SET balance = balance + 100 WHERE id = B;

Issues:
  - If process crashes between statements: A debited, B not credited. $100 vanishes.
  - If two transfers run concurrently against A:
       T1: read A=500, compute 400, write 400
       T2: read A=500, compute 400, write 400
       → A = 400 instead of 300.
  - Negative balance allowed unless we check.
```

### 3.2 The transactional fix

```sql
BEGIN;
SELECT balance FROM accounts WHERE id = A FOR UPDATE;  -- lock row
UPDATE accounts SET balance = balance - 100 WHERE id = A;
UPDATE accounts SET balance = balance + 100 WHERE id = B;
COMMIT;
```

`FOR UPDATE` takes a row-level exclusive lock; subsequent T2 against A blocks until T1 commits. The two-row update is atomic; durability survives crash.

### 3.3 The first scaling problem — deadlocks

```
T1: lock A, then try lock B.
T2: lock B, then try lock A.
→ Deadlock detector kills one. Retry storm at high QPS.
```

**Fix**: always lock in a deterministic order — by `min(A, B)` then `max(A, B)`. Deadlocks become rare.

### 3.4 The second scaling problem — hot account

If A is a "fee account" or "merchant payout", every transfer touches it. The row lock serializes everything.

```
1M TPS aspired.
Hot row throughput on PG: ~5K TPS for trivial UPDATE
                          ~1K TPS with row lock contention
                          → bottleneck.
```

**Mitigations:**

1. **Sharded counters** for the hot row.
   ```sql
   CREATE TABLE account_shards (account_id BIGINT, shard_id INT, balance NUMERIC, PRIMARY KEY(account_id, shard_id));
   ```
   Each transfer picks a random shard for A: `UPDATE … WHERE account_id = A AND shard_id = (rand 0..N)`. Reading the total balance reads all shards: `SUM(balance) WHERE account_id = A`.
   - Pro: write contention divided by N (typical N = 16–256).
   - Con: read is N-row scan. Unsuitable if reads are hotter than writes.

2. **Queue-per-account ordering**: write transfers to a per-account queue; a single worker per account processes them serially.
   - Pro: no row-lock contention; throughput becomes per-account-worker bound.
   - Con: per-account latency ~ queue depth × work time; reordering risks.

3. **Batch coalescing**: aggregate 1000 transfers in 100ms windows; apply as one row update.
   - Pro: WAL amplification per row drops by 1000×.
   - Con: increased latency; partial-failure complexity.

### 3.5 The third scaling problem — cross-shard

Account A is on shard 1, account B is on shard 5. The `BEGIN…COMMIT` is one shard's transaction, not both.

**Options:**

| Option | Mechanism | Trade |
|---|---|---|
| **2PC** | Coordinator orchestrates `PREPARE` then `COMMIT` across shards | Blocking on coordinator failure; latency = max(shard) RTT × 2 |
| **Saga + compensation** | Debit A first; if credit B fails, refund A | Eventually consistent; user-visible "in-flight" state |
| **Reservation + finalization** | Step 1: reserve $100 on A (`balance -= 100, reserved += 100`). Step 2: credit B. Step 3: clear reservation on A | Reservation can be auto-expired after T |
| **Single-ledger, per-account journals** | All transfers write to a global ledger; balances are projections | Eventually consistent ledger; great for audit |
| **TrueTime / Spanner-class** | Distributed strict serializable txn | Heavy infra; latency cost |

**At a real bank/exchange (Stripe, Square, Brex, AWS Pay), this is what runs:**

```
Ledger table (append-only, partitioned by date):
  (txn_id, account_id, delta, balance_after, timestamp, idempotency_key)

Each transfer = 2 rows (debit and credit) atomically inserted.
Atomicity guaranteed within a single shard if accounts colocated, or via 2PC if not.

Balance = SUM(delta) for account_id (cached).
Audit and reconciliation: replay the ledger.
```

This is **double-entry bookkeeping** in disguise. Treat the ledger as authoritative; balances are derived state.

### 3.6 What I'd actually do at staff level

1. Co-locate A and B on the same shard via **directory-based sharding by user-pair** when possible (not always practical).
2. Use **append-only ledger** for the source of truth.
3. Cache derived balances; reconcile periodically.
4. **Reservations + compensations** for cross-shard.
5. **Idempotency keys** (see §7) to make retries safe.
6. **Outbox + CDC** for downstream notifications, to avoid dual-write inconsistency.

The takeaway: at scale, "money transfer" is rarely one transaction. It's a sequence of well-bounded operations whose composition has the right invariants, with a single transaction at each atomic step.

---

## 4. Scenario 2 — E-commerce Checkout

**The textbook "multi-table transaction."**

### 4.1 The problem

When a user clicks "Place Order":

```
1. Validate inventory (1 row per item × N items)
2. Decrement inventory
3. Insert order header
4. Insert order lines
5. Charge payment method
6. Insert payment record
7. Send confirmation email / kick off fulfillment
```

Naive transactional approach:

```sql
BEGIN;
SELECT stock FROM inventory WHERE sku IN (...) FOR UPDATE;
UPDATE inventory SET stock = stock - N WHERE sku = ...;
INSERT INTO orders (...) VALUES (...);
INSERT INTO order_lines (...) VALUES (...);
-- charge payment...
INSERT INTO payments (...) VALUES (...);
COMMIT;
```

### 4.2 Why this falls apart at scale

1. **Payment is an external HTTP call.** It cannot be inside a DB transaction. If you put it inside, the txn holds locks for hundreds of ms while waiting on Stripe → throughput collapses, contention soars.
2. **Inventory contention on Black Friday**: "iPhone" SKU is hot. 10K concurrent checkouts → row-lock waiter pile-up → timeouts → retry storm.
3. **Order tables are append-only and partitioned by date**: locks across `inventory`, `orders`, `payments` may span partitions, hitting many B-trees.
4. **Confirmation email / kick-off** is a side-effect that can't be rolled back. If the txn commits but the email send fails, the system is inconsistent.

### 4.3 The staff-level decomposition

```
PHASE 1 — Reservation (atomic, fast)
  Reserve inventory: UPDATE inventory
                       SET reserved = reserved + N, available = available - N
                       WHERE sku = ? AND available >= N
                       RETURNING reserved;
  If 0 rows: out-of-stock; reject.
  Insert order with state='PENDING'.
  Single shard, single txn, ms-scale.

PHASE 2 — External payment (no DB lock held)
  Call payment provider with idempotency key.
  Wait for provider response (up to 30s).

PHASE 3 — Finalization (atomic)
  If payment success:
    BEGIN;
    Move inventory: SET reserved -= N (already deducted from available)
    Update order state: state='CONFIRMED'.
    Insert payment record.
    Insert outbox event (for fulfillment, email).
    COMMIT;
  If payment failed:
    BEGIN;
    Release reservation: SET reserved -= N, available += N
    Update order state: state='FAILED'.
    COMMIT;

PHASE 4 — Async side effects
  Outbox worker picks up events → publishes to Kafka.
  Fulfillment service consumes; emails go out.
  Idempotent at consumer side.

PHASE 5 — Garbage collection
  Background job: reservations older than T are auto-released.
  Protects against payment-stuck or crashed checkouts.
```

This is the **two-phase booking** pattern (also seen in airline, hotel, ticket systems). Transactions are small, single-shard, fast. External calls live between transactions, with explicit state.

### 4.4 The hot-SKU mitigation

For "iPhone Black Friday" hot SKUs:

- **Per-shard inventory partitioning**: split SKU's inventory across N rows; each checkout picks a random shard.
- **Optimistic concurrency**: read available, write `WHERE available >= N AND version = X`; retry on version mismatch. Avoids row lock; throughput scales to ~50K TPS.
- **Token-bucket admission control**: only allow K concurrent checkouts per SKU; queue the rest with "high traffic, please wait."
- **Inventory cache + reconciliation**: edge cache says "in stock"; central authority allocates; over-allocation refunded. Used by Amazon for ultra-hot items.

### 4.5 Trade-offs and alternatives

| Approach | Atomicity | Scale | Latency | Complexity |
|---|---|---|---|---|
| **Single big transaction** | Strong | Low (lock-bound) | High | Low |
| **Two-phase reservation** | Per-step | High | Low | Medium |
| **Saga with compensation** | Eventual | High | Variable | High |
| **Event-sourced order** | Append-only | Very high | Variable | High |
| **Reservation cache + central allocator** | Best-effort | Highest | Lowest | Highest |

**Staff insight**: Amazon, Shopify, Flipkart all use variants of two-phase reservation. The "single transaction" textbook answer scales to maybe 1K orders/sec on a single hot SKU. Real systems do 100K+/sec via decomposition.

---

## 5. Scenario 3 — Inventory Reservation with TTL

**Tickets, hotel rooms, flights, parking, GPU minutes — anywhere "soft holds" are real.**

### 5.1 The problem

User starts a checkout flow. Inventory should be held for them for 10 minutes while they enter payment. If they abandon, release the inventory.

### 5.2 Naive approach (and why it breaks)

```sql
INSERT INTO reservations (user_id, sku, expires_at, ...)
  VALUES (?, ?, NOW() + INTERVAL '10 minutes', ...);
UPDATE inventory SET available -= 1 WHERE sku = ? AND available >= 1;
```

Issues:
1. **Reservation cleanup**: who deletes expired reservations? A cron job? Per-read pruning? Hot tail on the cleanup query.
2. **Race on cleanup vs confirm**: cleanup deletes a reservation just as user confirms → double-decrement.
3. **Counting**: `available` is materialized; a stale value can over-allocate.

### 5.3 The "available = total - SUM(active reservations) - SUM(confirmed orders)" pattern

Don't materialize `available`. Compute it:

```sql
SELECT total
       - (SELECT COALESCE(SUM(qty), 0) FROM reservations
            WHERE sku = ? AND expires_at > NOW())
       - (SELECT COALESCE(SUM(qty), 0) FROM orders
            WHERE sku = ? AND status = 'CONFIRMED')
  FROM inventory_total WHERE sku = ?;
```

- Pro: no double-counting; no cleanup race.
- Con: read amplification; expensive at high QPS.

### 5.4 Hybrid: cached `available` + reconciliation

```
Cached available (atomic decrement):
  UPDATE inventory SET available -= 1
    WHERE sku = ? AND available >= 1
    RETURNING available;

  If 0 rows: rejected.

INSERT reservation row with TTL.

Background job (every minute):
  For each SKU, recount from authoritative source (orders + reservations).
  Correct any drift.

On reservation expiry:
  Background job deletes expired reservations AND increments cached available.

On reservation confirm:
  In one transaction:
    DELETE reservation;
    INSERT order (CONFIRMED state);
    -- inventory was already decremented at reservation time.
```

The transaction is small (single row in `reservations`, single row in `orders`).

### 5.5 The reaper anti-pattern

```sql
DELETE FROM reservations WHERE expires_at < NOW();  -- bad at scale
```

- Locks N rows; long transaction; replication lag spike.
- WAL traffic spike (full DELETE on each row).
- VACUUM pressure on Postgres.

**Better**: partition `reservations` by date/hour; drop partitions older than X. Constant-time, no scan.

### 5.6 Alternatives at scale

- **Redis with expiry keys**: `SETEX user:reservation:sku:abc 600 1`. Reservation is the key existing. No cleanup. Loss-on-failure ok for soft holds.
- **TTL'd ledger**: append reservation; ignore on read past TTL. No mutation needed; periodic compaction.
- **Token-based**: hand user a signed token "you have hold on SKU until T"; verify on confirm; no DB row.

For low-stakes (cart) → Redis. For higher-stakes (concert tickets) → DB with partitioning. For zero-tolerance (commercial flight) → DB + outbox + reconciliation.

---

## 6. Scenario 4 — High-Throughput Counter Increments

**View counts, like counts, follower counts.**

### 6.1 Why a transaction is the wrong starting place

```sql
UPDATE post_stats SET like_count = like_count + 1 WHERE post_id = ?;
```

Looks fine. Until:
- Hot post: 100K likes/sec.
- Single row, single shard: 1K–5K UPDATE/sec ceiling.
- Lock contention: each UPDATE waits for the previous; latency rises.
- WAL traffic: each UPDATE is a row mutation, full WAL record. Hot row dominates WAL.

### 6.2 Sharded counter

```sql
CREATE TABLE post_likes_sharded (
  post_id BIGINT,
  shard INT,
  count BIGINT,
  PRIMARY KEY (post_id, shard)
);

-- Increment: pick random shard 0..63
UPDATE post_likes_sharded SET count = count + 1
  WHERE post_id = ? AND shard = ?;

-- Read total
SELECT SUM(count) FROM post_likes_sharded WHERE post_id = ?;
```

Throughput multiplied by shard count. Read cost is N rows; cache the SUM aggressively.

### 6.3 Even cheaper: stream-based counter

```
User clicks like → emit event to Kafka (key = post_id).
Kafka topic partitioned by post_id.
Stream processor (Flink, Kafka Streams):
  Per-key state: count.
  Periodically commit aggregates to DB or serve from in-memory.
```

- Per-key throughput limited only by Kafka partition leader (10K+ msg/sec/partition).
- Reads served from cache or stream-store (e.g., Pinot, Druid).
- DB never sees hot row.

### 6.4 Probabilistic counters

For very large counts (1B views), exact precision doesn't matter. Use:

- **HyperLogLog** for unique-counts (e.g., unique viewers): ~0.5–1% error, KB of memory.
- **Sampled counters**: increment with probability 1/100; multiply on read.

Used heavily in Twitter view counts, Pinterest analytics, Reddit upvote rendering.

### 6.5 The "Like + insert action" multi-row case

If "like" is also an audit event:

```sql
BEGIN;
INSERT INTO actions (user_id, post_id, action='like', created_at);
UPDATE post_likes_sharded SET count += 1 WHERE post_id = ? AND shard = ?;
COMMIT;
```

Now you have a true txn. At hot post, even sharded UPDATE contends with INSERT to actions table.

**Decomposition**: write the like event only. Async aggregation reads the action stream and updates the counter.

```
Sync path: INSERT INTO actions (1 row, no contention).
Async path: stream processor reads actions; updates counter.
Read path: serve from cache; periodic refresh from counter table.
```

This is the **CQRS** pattern in disguise: writes are simple inserts; reads are projections.

### 6.6 Transactional invariants for counters

Sometimes the counter has business meaning:
- "User has 1000 followers, 1000 followee count" must match (graph invariant).
- "Total inventory" across all warehouses must equal sum of per-warehouse stocks.

If the invariant is **strong** (audit, regulatory), keep the transaction (and pay the contention cost). If it's **soft** (display), eventual consistency via stream is fine and infinitely faster.

---

## 7. Scenario 5 — Idempotent Operations

**Every retryable, side-effect-having operation needs this.**

### 7.1 The problem

Mobile app sends "place order" request. Network glitch; client doesn't see response. Client retries. Now there are two orders.

### 7.2 The pattern

```sql
CREATE TABLE idempotency_keys (
  key TEXT PRIMARY KEY,
  result JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  status TEXT  -- 'IN_PROGRESS', 'COMPLETED', 'FAILED'
);

-- On request:
BEGIN;
INSERT INTO idempotency_keys (key, status) VALUES (?, 'IN_PROGRESS');
-- if conflict (ON CONFLICT DO NOTHING returns 0): another in-flight or done
-- → fetch existing; return its result if COMPLETED, else 409/wait.

-- if inserted: do the work.
INSERT INTO orders (...);
UPDATE idempotency_keys SET result = ?, status = 'COMPLETED' WHERE key = ?;
COMMIT;
```

The `idempotency_keys` row is in the same transaction as the side-effect. Either both commit or both roll back.

### 7.3 What goes wrong if you skip this

- **Double-charge**: payment retried; charged twice.
- **Double order**: same order twice in DB.
- **Inventory drained 2×**.
- **Lost client expectation**: if retry returns different response than original, client logic breaks.

### 7.4 At scale considerations

- **Storage**: keys retained 24–48 hours typically. At 100M orders/day → 200M rows × 2KB ≈ 400 GB hot. Partition by date; drop old.
- **Hot key**: if every retry hits same key, lock contention on that row. Mitigation: short transaction; explicit "is in progress" handling; no waiting on row lock.
- **Cross-shard**: idempotency key store should be on the same shard as the side-effect. If it's not, you have a 2PC-like coupling.

### 7.5 The async/event-driven version

For Kafka-based event processing:

```
Consumer:
  read message;
  key = message.idempotency_key;
  begin;
    INSERT INTO processed_keys (key) ON CONFLICT DO NOTHING;
    -- if 0 rows: already processed → ack and skip.
    -- else: do work.
    INSERT INTO domain_table (...);
  commit;
  ack message;
```

Critical: the `processed_keys` insert and the work must be in the **same transaction**. If they're not (e.g., you write to DB, then ack Kafka), a crash between them means re-processing.

### 7.6 Alternatives

- **Natural key uniqueness**: `(user_id, request_id)` UNIQUE constraint. INSERT fails on duplicate; idempotent for free.
- **Versioned writes (CAS)**: `UPDATE … WHERE version = ?`. Retry on conflict.
- **Application dedup**: "did I already process this?" via cache. Cheaper but eventual.

---

## 8. Scenario 6 — Outbox Pattern (Atomic DB + Event)

**The "dual write" problem solved.**

### 8.1 The problem

```
On user signup:
  INSERT INTO users (...);
  Kafka.publish('user-signup-event', user_id);
```

What if the INSERT commits and the Kafka publish fails? Or vice versa? Dual write → divergent state.

### 8.2 The outbox pattern

Write the event to a DB table inside the same transaction:

```sql
BEGIN;
INSERT INTO users (...);
INSERT INTO outbox_events (id, type, payload, created_at) VALUES (?, 'user-signup', ?, NOW());
COMMIT;
```

A separate worker tail-reads the outbox and publishes to Kafka:

```
loop:
  SELECT * FROM outbox_events WHERE published = false ORDER BY id LIMIT 100;
  for each event:
    Kafka.publish(event);
    UPDATE outbox_events SET published = true WHERE id = ?;
```

Or, more elegant: **CDC (Change Data Capture)**. Debezium/Kafka Connect read the DB's WAL and stream INSERTs to Kafka without poll.

### 8.3 The trade-offs

- **Latency**: event publication is delayed by polling interval (typically <1s) or CDC stream lag (~10s of ms).
- **At-least-once delivery**: the event may be published more than once if the worker crashes between publish and update. Consumers must be idempotent.
- **Outbox table growth**: needs partitioning + retention.
- **Cross-shard outbox**: if writes are sharded, outbox is per-shard; aggregator merges streams.

### 8.4 Why this beats alternatives

| Alternative | Issue |
|---|---|
| **2PC across DB and Kafka** | Kafka doesn't support XA cleanly; complex; stalls on coordinator failure. |
| **Best-effort publish** | Lost events on crash. |
| **Event-as-source-of-truth (event sourcing)** | Powerful but huge architectural shift. |
| **Outbox** | Simple; only needs DB transaction; works with any messaging system. |

This is what nearly every modern microservice does (Confluent recommends it; AWS recommends DynamoDB Streams or RDS + DMS variant; Postgres+Debezium is the canonical OSS combo).

### 8.5 The reverse — Inbox pattern

When consuming events from Kafka and writing to DB, the symmetric problem:

```sql
BEGIN;
INSERT INTO inbox_events (event_id) ON CONFLICT DO NOTHING;
-- if 0 rows: already processed.
INSERT INTO domain_table (...);
COMMIT;
ack Kafka;
```

The inbox table is essentially a deduplication idempotency store, scoped to event_id.

---

## 9. Scenario 7 — Cross-Shard Transactions

**When the data you need atomicity over isn't co-located.**

### 9.1 The dilemma

You designed for sharding. You did consistent hashing on `user_id`. Now you have a feature that updates two users' rows in one operation: a follow, a friend, a chat.

Options:

| Option | Trade |
|---|---|
| **Re-shard so they're co-located** | Painful; user-pair sharding doubles storage |
| **2PC** | Stalls on coordinator failure; throughput limited |
| **Saga** | Eventually consistent; user-visible "in-flight" |
| **Single-leader for cross-shard ops** | Simpler but bottleneck |
| **Centralize the affected data** | Move out of sharded table into central one |

### 9.2 2PC mechanics

```
Coordinator → all shards: PREPARE (write WAL, lock rows)
All shards reply: PREPARED.
Coordinator → all shards: COMMIT.
All shards: write commit, release locks.
Coordinator: durably record commit.

If any shard says NO at PREPARE: ABORT to all.
If coordinator fails after PREPARE: shards block on locks until coordinator recovers.
```

The key flaw: if coordinator dies after some shards COMMITted, others are still PREPARED with locks held. Recovery requires durable coordinator log.

**Throughput**: 2PC adds 2× round-trips and forced log flushes. At a real distributed Postgres, expect 5–10× the latency of a single-shard txn.

### 9.3 Saga pattern (compensating txns)

```
Step 1: shard 1 commit (debit)
Step 2: shard 2 commit (credit)
Step 3: shard 3 commit (audit log)

If step 3 fails:
  Compensating step 3 (none, was last).
  Compensating step 2: refund credit.
  Compensating step 1: refund debit.
```

The system is **eventually consistent**; intermediate states are visible to readers. Reads must handle "in-flight" status.

Tools: Temporal, AWS Step Functions, Cadence, Camunda.

### 9.4 The "transactional state machine" pattern

```
Order has state field: NEW → RESERVED → PAID → SHIPPED → DELIVERED.
Each transition is its own transaction (single shard, single row).
Worker drives transitions; failure → mark FAILED + compensations.
```

This combines saga with explicit state. Easier to reason about, recover, audit.

### 9.5 Spanner / CockroachDB / TiDB — globally distributed transactions

- True distributed strict-serializable transactions.
- Cost: ~10–100ms per cross-shard txn; latency dominated by Paxos rounds + (in Spanner) TrueTime wait.
- Scale: high (Google scale), but per-row throughput on hot keys is bounded by Paxos throughput (~10K TPS per group).

Worth using when:
- Business needs strong consistency across shards.
- Volume is not extreme on any single hot row.
- Engineering team can absorb operational complexity.

Not worth using when:
- Eventual consistency is acceptable.
- Throughput targets are 10×+ what Paxos delivers per shard.

### 9.6 What I'd actually do

90% of cross-shard "transactions" can be reframed as:
- **Saga + compensation** for write paths.
- **State machine** for ordering.
- **Outbox + CDC** for downstream propagation.
- **Idempotency keys** for retry safety.

The remaining 10% (financial settlement across regions, regulatory atomic close-of-books) → Spanner-class or careful 2PC with engineered coordinator.

---

## 10. Scenario 8 — Cross-Service Distributed Transactions

**Microservices and the death of "BEGIN…COMMIT".**

### 10.1 The problem

Order Service, Payment Service, Inventory Service, Shipping Service. Each has its own DB. "Place order" needs all to succeed or none.

There is no `BEGIN` that spans them.

### 10.2 The saga pattern at service-boundary level

```
Place Order saga:
  Step 1: Order Service     — create order, status='PENDING' (txn local)
  Step 2: Inventory Service — reserve inventory                 → if fail, comp 1
  Step 3: Payment Service   — charge                            → if fail, comp 2, comp 1
  Step 4: Shipping Service  — schedule shipment                 → if fail, comp 3, comp 2, comp 1
  Step 5: Order Service     — mark CONFIRMED                    → terminal

Compensations:
  Comp 1: Order Service — mark FAILED
  Comp 2: Inventory Service — release reservation
  Comp 3: Payment Service — refund (idempotent)
  Comp 4: Shipping Service — cancel shipment
```

Each step is a local transaction. The saga is the *orchestration*, persisted in a workflow engine (Temporal, Step Functions) so it survives crashes.

### 10.3 Two saga styles

**Choreography (pub/sub)**:
```
Order Service publishes 'OrderPlaced'
  ↓
Inventory Service subscribes; on success publishes 'InventoryReserved'
  ↓
Payment Service subscribes 'InventoryReserved'; on success publishes 'PaymentCharged'
  ↓
... etc.

On failure, services publish failure events; subscribers compensate.
```
- Pro: loosely coupled.
- Con: workflow logic spread across services; hard to debug; failures are harder to track.

**Orchestration (central)**:
```
Orchestrator (Temporal workflow) explicitly calls services in sequence.
Persists state; on crash, resumes.
```
- Pro: single place to read/debug workflow.
- Con: orchestrator is a critical-path service.

Most teams I've worked with end up at orchestration after starting with choreography. Debugging multi-service flows via event logs is painful at scale.

### 10.4 Implementation pitfalls

- **Idempotency at every step**: a saga retry must not double-charge.
- **Compensation idempotency**: compensations must also be idempotent (a compensation may run multiple times after partial failure).
- **Compensations are not always possible**: an email already sent can't be unsent. Design so that irreversible side effects are last.
- **Long-running transactions**: a saga may run for hours. The "in-flight" state is visible to other parts of the system.
- **Read-during-saga**: another service reads the order while it's in PENDING. Define what's allowed.

### 10.5 The "transactional outbox + saga" combination

```
Service A:
  BEGIN;
  Local DB write;
  INSERT outbox event;
  COMMIT;

Outbox publisher → Kafka.
Service B consumes; runs its own local txn + outbox + Kafka publish.
Repeat.

End-to-end: at-least-once delivery; idempotency at each consumer; eventually consistent.
```

This is the de-facto standard for B2B / e-commerce / payments at MAANG scale.

### 10.6 When NOT to use sagas

- For very fast (< 100ms) operations with low conflict, single transaction in a co-located DB is fine.
- For purely read-side composition, GraphQL / API gateway aggregation is cheaper.
- For workflows < 5 steps and infrequent (< 100 RPS), simple try/catch with manual compensation is enough.

Sagas are cost: workflow engine, idempotency, observability, retry budgets, on-call. Adopt when the volume justifies it.

---

## 11. Scenario 9 — Schema Migrations Inside Transactions

**Where DDL meets DML.**

### 11.1 What works in a transaction

PostgreSQL supports **transactional DDL** for most operations:

```sql
BEGIN;
ALTER TABLE users ADD COLUMN email_verified BOOLEAN DEFAULT FALSE;
UPDATE users SET email_verified = TRUE WHERE confirmation_done = TRUE;
COMMIT;
```

If anything fails, the schema change rolls back.

MySQL InnoDB does **not** support transactional DDL — `ALTER TABLE` commits implicitly. A migration failure leaves you in a half-applied state.

### 11.2 What breaks at scale

```sql
ALTER TABLE huge_table ADD COLUMN new_col TEXT;
```

On a 10TB table:
- PG: takes a brief metadata-only lock if no `DEFAULT` (instant). With `DEFAULT`, PG 11+ uses a fast path; older needs full rewrite.
- MySQL InnoDB online DDL: rebuilds table; can take hours; replicas lag.
- Either: long lock blocks all writes/reads on the table.

### 11.3 The expand-contract pattern (covered in stateful doc, summarized here)

```
PHASE 1: ADD column nullable (fast, metadata-only).
PHASE 2: Backfill in chunks: UPDATE WHERE id BETWEEN low AND high; sleep; repeat.
PHASE 3: Code reads from new column; falls back to derive-on-the-fly.
PHASE 4: Code writes to new column.
PHASE 5: Drop old column / rename / etc.
```

Each phase is its own deploy. None is one giant transaction. **Big ALTER TABLE in one transaction is the wrong tool at scale.**

### 11.4 DDL transactions for safety-critical changes

For correctness-critical changes (e.g., enforcing a UNIQUE constraint after deduplication), wrap in a single transaction:

```sql
BEGIN;
DELETE FROM users a USING users b WHERE a.id < b.id AND a.email = b.email;
ALTER TABLE users ADD CONSTRAINT users_email_uniq UNIQUE (email);
COMMIT;
```

If the dedup races with concurrent INSERTs, the constraint fails at COMMIT and rolls everything back. Better than a half-applied state.

### 11.5 Tools

- **gh-ost** / **pt-online-schema-change** (MySQL): shadow table + trigger-based replication of writes during copy.
- **pg_repack** / **pg_squeeze** (PG): rebuild bloated tables without long locks.
- **PG 11+ fast-default** for column adds.
- **Liquibase** / **Flyway**: migration version control; doesn't help with safe execution itself.

---

## 12. Scenario 10 — Audit / Compliance Atomicity

**Regulators don't accept "we'll fix it eventually."**

### 12.1 The problem

A bank's transfer must:

1. Update balances.
2. Insert audit row with full context (who, when, why, IP, session ID).
3. Insert immutable journal entry.

If any of these fails, the others must roll back. Regulators require: "every financial mutation has a traceable audit trail."

### 12.2 The transactional approach

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = ?;
UPDATE accounts SET balance = balance + 100 WHERE id = ?;
INSERT INTO audit_log (user_id, action, ip, session_id, ...);
INSERT INTO ledger_journal (txn_id, debit_account, credit_account, amount, hash);
COMMIT;
```

`hash` is a chained hash linking this entry to the previous (`hash = SHA256(prev_hash || row_data)`) — tamper-evident.

### 12.3 At scale

- The audit + ledger inserts are 2 extra rows per write — modest overhead.
- The chained hash forces serialization (each row depends on the previous's hash). Use a per-account journal to parallelize.
- Audit log grows fast; partition by date; archive to S3 + immutable storage; retain per regulatory period (often 7 years).

### 12.4 Hash chains and Merkle trees

For heavy compliance:

```
Each ledger entry: hash = SHA256(prev_hash || canonical_row_data).
Periodically: build Merkle tree over a window; publish root to immutable store.
```

This makes any tampering detectable. Used by exchanges, healthcare, voting.

### 12.5 Alternatives — at the cost of correctness

- **Async audit**: write to async queue; background job persists. Loses audits on crash window.
- **Sampled audit**: only some operations audited. **Never acceptable for financial / healthcare**.

For audit, **transactional inclusion is non-negotiable**. The cost is an extra row insert per operation. Pay it.

### 12.6 Append-only event-sourced audit

The ultimate version: the audit *is* the source of truth.

```
Domain table is a projection of an event log.
Every business event is appended to event_log.
event_log is the audit; balance is derived.
```

This is event sourcing. Heaviest engineering, most defensible; used in regulatory-heavy systems (settlement, exchange, healthcare).

---

## 13. Scenario 11 — Workflow State Machines

**Long-running business processes with crash recovery.**

### 13.1 The problem

A loan application moves through states:

```
SUBMITTED → DOC_VERIFICATION → CREDIT_CHECK → MANUAL_REVIEW → APPROVED / DENIED
```

Each transition is triggered by external events (user uploads doc, bureau replies). The process spans days. The system can crash and resume.

### 13.2 The transactional state model

```sql
CREATE TABLE applications (
  id BIGINT PRIMARY KEY,
  state TEXT NOT NULL,
  state_data JSONB,
  version INT NOT NULL DEFAULT 0,
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Transition:
BEGIN;
SELECT version, state, state_data FROM applications WHERE id = ? FOR UPDATE;
-- check current state matches expected; if not, abort (concurrent transition)
UPDATE applications SET state = ?, state_data = ?, version = version + 1
  WHERE id = ? AND version = ?;
INSERT INTO state_transitions (application_id, from_state, to_state, ts, ...);
INSERT INTO outbox_events (...);  -- notify next service
COMMIT;
```

Each transition: ~3 row writes, single shard, fast. The transaction guarantees you don't have a "half-transitioned" state.

### 13.3 Why this beats imperative orchestration

A naive Python script:
```
verify_doc(app_id)
check_credit(app_id)
approve(app_id)
```

Issues:
- Crash mid-flow: app stuck.
- Rerun: verify_doc runs again.
- Concurrency: two operators advancing same app.

The state machine forces:
- Every step persisted.
- Every step idempotent (you can re-emit the transition; the version check rejects double-advance).
- Crash → restart resumes from current state.

### 13.4 Workflow engines

For deeper workflows:
- **Temporal / Cadence**: durable workflows in code; the engine handles persistence.
- **AWS Step Functions**: state machine as JSON; AWS-managed.
- **Airflow / Argo / Dagster**: data-pipeline focused; less suited for low-latency.
- **Camunda / Zeebe**: BPMN-style; enterprise-friendly.

These engines internally use transactions to durably advance workflow state.

### 13.5 Trade-offs

| Approach | Persistence | Latency | Complexity | Best for |
|---|---|---|---|---|
| **DB state machine** | Strong | Low | Low | Few states, simple, low volume |
| **Workflow engine** | Strong | Medium | Medium | Complex flows, high volume |
| **Event-sourced** | Strong (log) | Medium | High | Audit-heavy, replay-needed |
| **Imperative** | None | Lowest | Lowest | Don't use for anything that crosses minutes |

---

## 14. Scenario 12 — Optimistic Concurrency Control

**The alternative to row locks.**

### 14.1 The pattern

Add a `version` column. Read it; write with `WHERE version = ?`; on mismatch, retry.

```sql
-- Read
SELECT id, balance, version FROM accounts WHERE id = ?;
-- (perform business logic in app)
UPDATE accounts SET balance = ?, version = version + 1
  WHERE id = ? AND version = ?;
-- if 0 rows: someone else updated; retry from read.
```

### 14.2 When to use OCC vs locks

| Scenario | Choose |
|---|---|
| **Low conflict** (most updates don't race) | OCC; cheaper, no lock |
| **High conflict** (hot row) | Pessimistic lock; OCC will retry-storm |
| **Long-running thinking time** (user reviews then submits) | OCC; locking would hold for minutes |
| **Transactional invariant across rows** | Pessimistic; OCC doesn't compose |

### 14.3 OCC at scale

The retry-storm risk: under contention, OCC's success rate drops; everyone retries; CPU consumed by retries; you achieve less throughput than pessimistic locking would have.

Mitigations:
- **Exponential backoff with jitter** on retry.
- **Bounded retries** (e.g., 3 attempts; then 409 to client).
- **Hybrid**: switch to lock-based when retry rate exceeds threshold.

### 14.4 ETags in HTTP APIs

OCC's REST cousin:
```
GET /resource → 200 OK, ETag: "v3"
PUT /resource (If-Match: "v3") → 200 if version matches, 412 Precondition Failed if not.
```

Same pattern: client tells server which version it's editing; server rejects on mismatch.

### 14.5 Per-field OCC for collaborative editing

For Google Docs / Figma-style:
- Per-character or per-cell version.
- CRDTs to merge concurrent edits.
- OCC at the document level + CRDT at the field level.

### 14.6 Trade-off summary

```
Locks (pessimistic):
+ Simple to reason about
+ Correct under contention
- Holds resources during user think-time
- Lock escalation, deadlocks at scale

OCC:
+ No held resources
+ High throughput at low conflict
- Retry-storm at high conflict
- Application logic for retry / merge
```

Real systems use both, picking per data class.

---

## 15. Scenario 13 — Distributed Lock / Lease Acquisition

**"Only one of N workers should do this thing."**

### 15.1 The problem

A scheduled job. 10 worker pods. Only one should run the job at a time.

### 15.2 DB-based lock

```sql
BEGIN;
INSERT INTO locks (name, holder, acquired_at, expires_at)
  VALUES ('cron:job-X', ?, NOW(), NOW() + INTERVAL '60 seconds')
  ON CONFLICT (name) DO UPDATE SET holder = ?, acquired_at = NOW(), expires_at = NOW() + INTERVAL '60 seconds'
  WHERE locks.expires_at < NOW();

-- if 0 rows updated: someone else holds it.
-- if 1 row: we got it. release at end:
DELETE FROM locks WHERE name = ? AND holder = ?;

COMMIT;
```

Or with `SELECT … FOR UPDATE` semantics:

```sql
SELECT * FROM locks WHERE name = ? FOR UPDATE NOWAIT;
-- block on conflict / fail fast
```

### 15.3 Why leases (TTLs) are critical

Without TTL: holder dies, lock held forever, system hung.

With TTL: lock auto-expires; another holder can take it. But **lease expiry must not race with active work**:

```
Holder acquires for 60s.
Holder GC pauses for 65s.
Lock expires; another holder takes it.
First holder wakes, thinks it still holds the lock, does work.
→ two holders running concurrently.
```

**Mitigation: fencing tokens.** Lock returns a monotonically increasing token. Every protected operation includes the token. Downstream rejects writes with stale tokens.

### 15.4 Alternatives

- **Redis SET NX EX** (with Redlock controversy).
- **etcd / ZooKeeper / Consul** for ephemeral locks (lease bound to session; auto-released on session loss).
- **Postgres advisory locks** (`pg_advisory_lock(N)`) — fast, in-memory, automatically released on disconnect.

### 15.5 At scale

A single lock row on the DB is a hot row. For tens of thousands of locks, this is fine. For millions of fine-grained locks, use a lock-management service (etcd) or shard the lock table.

### 15.6 The lock-free alternative

Often the "lock" is really "claim a unit of work." Better:

```sql
UPDATE jobs SET status = 'CLAIMED', worker_id = ?, claimed_at = NOW()
  WHERE id = (
    SELECT id FROM jobs WHERE status = 'PENDING'
    ORDER BY created_at LIMIT 1
    FOR UPDATE SKIP LOCKED
  )
  RETURNING *;
```

`FOR UPDATE SKIP LOCKED` (PG 9.5+, MySQL 8+): skip rows another txn locked. Workers concurrently claim different jobs without blocking. Eliminates the global lock entirely.

This is the single best PG transaction trick most engineers don't know. Used in pgmq, Skytools, every modern PG-backed queue.

---

## 16. Scenario 14 — Bulk Operations / Batch ETL

**When transactions cost more than they're worth.**

### 16.1 The problem

Nightly: import 100M rows from S3 into a transactional DB.

Naive:
```sql
BEGIN;
COPY users FROM '/data/users.csv';
COMMIT;
```

For 100M rows:
- WAL: ~50 GB.
- Replicas lag: minutes.
- VACUUM pressure: huge.
- Lock duration: long.
- Crash mid-import: full rollback (slow).

### 16.2 Batched approach

```python
for chunk in chunks_of_10000(file):
    BEGIN;
    COPY chunk;
    COMMIT;
```

Each chunk is a separate transaction. Smaller WAL bursts; replicas keep up; crash loses only the last chunk.

But: not atomic. If you need "all or nothing", you need a single transaction (and the cost).

### 16.3 The shadow table swap

```
1. CREATE TABLE users_new (LIKE users INCLUDING ALL);
2. Bulk load into users_new (no transaction; many small commits).
3. Verify count, sums, checksums.
4. BEGIN;
   ALTER TABLE users RENAME TO users_old;
   ALTER TABLE users_new RENAME TO users;
   COMMIT;
5. DROP TABLE users_old (later, after verification window).
```

The atomic swap is two metadata renames — millisecond txn. Bulk load is unconstrained.

### 16.4 INSERT … ON CONFLICT for upserts

```sql
INSERT INTO users (id, email, ...) VALUES ...
  ON CONFLICT (id) DO UPDATE SET email = EXCLUDED.email, updated_at = NOW();
```

Single statement, atomic per-row. At 50K rows/batch, ~1s per batch on modern hardware.

### 16.5 Tools at scale

- **Postgres COPY** with parallel streams.
- **CDC** for incremental sync (Debezium → Kafka → consumer).
- **pg_bulkload** for offline loads bypassing WAL.
- **Snowflake / BigQuery COPY INTO** for warehouse loads.

The general pattern: **don't put bulk data in a single transaction**. Decompose; prove invariants outside the txn; atomic-swap final.

---

## 17. Scenario 15 — Read-Your-Write Consistency

**The user posts; the user refreshes; the post is missing.**

### 17.1 The problem

```
User T1: write to primary.
User T1: read from replica (load-balanced).
Replica hasn't received the write yet.
→ user sees old state.
```

Within a transaction, this isn't an issue (READ COMMITTED+ sees own writes). But across transactions / requests, with read replicas, it's a real UX bug.

### 17.2 Solutions

1. **Pin to primary for N seconds after write**:
   ```
   user.last_write_at = NOW()
   if NOW() - user.last_write_at < replica_lag_max:
     read from primary
   else:
     read from replica
   ```
   N typically 5–30 seconds.

2. **LSN tracking**:
   ```
   On write: server returns LSN (PG: pg_current_wal_lsn()).
   Client stores LSN.
   On read: server checks replica's LSN; if behind, route to primary or wait.
   ```
   PG: `SELECT pg_last_wal_replay_lsn()`. Spanner has read timestamps.

3. **Write-then-cache pattern**:
   On write, also update cache so the immediate read hits cache, not replica.

4. **Sticky-session routing**: same user always hits same replica; replica is configured to receive that user's writes synchronously.

### 17.3 Why this is a transaction-adjacent topic

Within a transaction (auto-commit off), READ COMMITTED guarantees you see your own writes. The problem is across transactions, where the replica may not have caught up.

The fix is **session-level consistency**, which extends transaction semantics to a session.

### 17.4 At scale

Pinning to primary costs read scalability. The right balance:
- Heavy-write users (back-office, admin): pin always.
- Normal users: pin for 30s post-write.
- Read-only users: pure replica reads.

A staff engineer designs this routing layer carefully. The default in many ORMs is pure-replica, which silently breaks UX.

---

## 18. Scenario 16 — Cross-Region Transactions

**The hardest and most expensive transaction.**

### 18.1 The cost

```
Cross-region RTT:
  US-East ↔ US-West: ~70ms one-way, ~140ms RTT
  US ↔ EU:           ~100ms one-way, ~200ms RTT
  US ↔ APAC:         ~150ms one-way, ~300ms RTT

A 2PC across 3 regions: 2 RTTs minimum × 200ms = 400ms per transaction.
Cannot beat physics.
```

### 18.2 Choices

| Approach | Latency | Consistency |
|---|---|---|
| **Sync 2PC across regions** | 200–400ms | Strict serializable |
| **Spanner-style (Paxos + TrueTime)** | 50–200ms | Strict serializable, lower than 2PC |
| **Async replication, region-pinned** | <10ms local | Eventual cross-region |
| **CRDT-based, conflict-free** | <10ms local | Strong eventual |
| **Saga across regions** | Varies | Eventual; explicit compensation |

### 18.3 The region-pinned design

Most products do this:

```
User belongs to home region (US-EAST or EU-WEST).
Their data is primary in that region; replicated async to others.
Writes go to home region. If user travels, they still write to home region (slower) or are migrated.

Cross-region operations (rare):
  - Two users in different regions interact.
  - Use saga with compensation.
  - Or designated "global" tables (e.g., user identity) replicated synchronously.
```

### 18.4 The "global" data tier

Some data must be globally consistent: user ID → home-region mapping; account existence; abuse / fraud blocklist. This goes in a Spanner-class globally consistent tier, kept small and rarely written.

### 18.5 What I'd avoid

- 2PC across geo-regions for high-throughput hot paths. Latency kills user experience.
- Reaching for "global strong consistency" for everything. Most data is region-local.

The staff-level call: identify the **few data classes that need global strong consistency** (identity, money, regulatory state). Everything else: region-pinned + async replication + sagas.

---

## 19. Scenario 17 — Multi-Tenant Resource Quota and Transfer

**"Tenant A has 1000 GPU minutes; Tenant B has 500. Transfer 100 from A to B."**

### 19.1 The pattern

```sql
BEGIN;
SELECT balance FROM tenant_quotas WHERE tenant_id = A FOR UPDATE;
-- check balance >= 100, else abort
UPDATE tenant_quotas SET balance = balance - 100 WHERE tenant_id = A;
UPDATE tenant_quotas SET balance = balance + 100 WHERE tenant_id = B;
INSERT INTO quota_transfers (from_tenant, to_tenant, amount, ts);
COMMIT;
```

### 19.2 Throughput

If one row per tenant per quota type, throughput per tenant is bounded (~5K TPS). For most B2B SaaS, that's fine: tenants don't transfer at sub-ms intervals.

For high-frequency quota usage (per-API-call):
- Don't transact per request.
- Use a token-bucket cache (Redis) with periodic reconciliation against the DB.
- Batch consumption: deduct chunks of 1000 from DB; serve from local bucket; refill async.

### 19.3 Cross-tenant invariants

"Tenant A's balance + tenant B's balance must equal a constant (no creation of value)."

The transaction enforces this, atomically.

But: ad-hoc quota grants can break invariant. Mitigation: dedicated "system" tenant from which grants are made; transfers are always between tenants.

### 19.4 At scale: the per-tenant ledger

```
tenant_ledger (append-only, partitioned by tenant_id):
  (tenant_id, txn_id, delta, balance_after, type, ts)

Transfer: 2 rows (one negative for A, one positive for B), same shard/transaction if possible.
Balance: SUM(delta) for tenant, cached.
Audit: replay the ledger.
```

This is the bookkeeping pattern again — ledgers are the right structure for transactional, auditable, eventually-summable state.

---

## 20. Isolation Levels — When to Use Each

A staff-level reference for picking the right level.

### 20.1 Read Committed (default in PG, Oracle)

```
Reads see only committed data (no dirty reads).
Each statement sees a fresh snapshot.
Two reads in the same transaction may differ.
```

**Use for**: 80% of workloads. Most CRUD, most reads, most writes that don't span statements.

**Caveat**: lost updates possible. `UPDATE x SET v = v+1` with `WHERE` matching is fine (atomic). Read-then-write at app level is not (use OCC or `FOR UPDATE`).

### 20.2 Repeatable Read / Snapshot Isolation

```
Same snapshot for entire transaction.
No phantoms (in PG).
Detects lost updates (two txns updating same row → second fails).
```

**Use for**: read-heavy reports needing consistent view; multi-statement business logic that needs stable reads.

**Caveat**: snapshot is taken at first read; long-running txn → snapshot grows old → MVCC bloat. **Avoid keeping txn open for long.**

### 20.3 Serializable Snapshot Isolation (SSI in PG)

```
Serializable result; achieved by detecting and aborting transactions involved in
"dangerous structures" (read-write cycles).
```

**Use for**: invariants across multiple rows that simple locking would miss (the doctor-on-call write-skew classic).

**Caveat**: serialization failures (SQLSTATE 40001) require app retry. ~5–20% performance overhead vs RR.

### 20.4 Strict Serializable (Spanner, FaunaDB, Aurora multi-AZ via Paxos)

```
Globally consistent; all transactions in one true order.
```

**Use for**: financial settlement, identity, regulatory atomicity.

**Caveat**: latency cost (Paxos rounds, TrueTime wait). Throughput per group bounded by Paxos.

### 20.5 The pragmatic default

```
Default: READ COMMITTED.
Specific txns needing stronger: explicitly upgrade to REPEATABLE READ or SERIALIZABLE.
Money / regulatory / cross-account invariants: SERIALIZABLE with retry handling.
```

Rarely the right call: SERIALIZABLE everywhere "to be safe". You're paying for invariants you don't have.

---

## 21. Anti-Patterns — Staff-Level Red Flags

### 21.1 Long-running transactions

```sql
BEGIN;
... lots of work, app processing, external HTTP, user thinking time ...
COMMIT;
```

Holds locks; bloats MVCC; replicas lag. **Maximum txn duration: seconds, not minutes.** Any "hold txn while waiting for X" is wrong.

### 21.2 Transactions across HTTP calls

```python
with db.transaction():
    user = db.find(user_id)
    response = http.get('https://api.x.com/...')  # NEVER
    user.update(response.data)
```

DB connection is held; downstream slowness becomes DB lock. Refactor to: read → external call → re-validate → update inside short txn.

### 21.3 Catching and ignoring transaction errors

```python
try:
    with db.transaction():
        do_thing()
except SerializationFailure:
    pass  # silently lost work
```

Serialization failures must be retried. Constraint violations indicate real bugs. Don't swallow.

### 21.4 Putting the entire request in a transaction

Every web framework that does "auto-transaction-per-request" wastes resources. Most reads don't need txns; some writes need explicit ones. Be deliberate.

### 21.5 Using transactions for caching / rate limiting

```sql
BEGIN;
SELECT count FROM rate_limits WHERE user = ? FOR UPDATE;
... check, increment ...
COMMIT;
```

Lock contention on hot users, full DB cost for what should be µs in Redis. Use a real rate-limit store.

### 21.6 Implicit cross-shard transaction expectations

```python
with db.transaction():
    user_in_shard_1.do()
    other_user_in_shard_5.do()
```

Most ORMs will silently do separate txns per shard, with no atomicity. Make the expectation explicit; design for 2PC or saga.

### 21.7 Idempotency-key-less retry

```python
retry(lambda: charge_card(amount))
```

Without idempotency key, retry can double-charge. Always pair retry with idempotency.

### 21.8 "Transactions are slow, let's not use them"

Sometimes correct (counters, eventual data). Often wrong (anything with multi-row invariants). The cost of a wrong transaction-skip is data corruption, found in production months later.

### 21.9 Mixing isolation levels in one txn

Possible in some engines, never sane. Pick one isolation level per txn.

### 21.10 Using SELECT FOR UPDATE without ORDER BY

```sql
SELECT * FROM orders WHERE status = 'PENDING' FOR UPDATE;
-- no ORDER BY
```

Different transactions may iterate in different orders → deadlocks. Always ORDER BY a stable key for FOR UPDATE.

### 21.11 Forgetting that triggers and constraints run in the txn

A trigger that calls an external service via HTTP. A `CHECK` constraint that does heavy SELECT. Hidden cost; surprises in profiling.

### 21.12 Not understanding what your ORM generates

Hibernate / Active Record / SQLAlchemy can generate N+1 queries, N+1 transactions, or surprise lock acquisitions. Profile actual SQL.

### 21.13 BEGIN; ... ; COMMIT in a loop

```python
for item in big_list:
    with db.transaction():
        update(item)
```

vs

```python
with db.transaction():
    for item in big_list:
        update(item)
```

First: many small txns; replication lag; commit cost. Second: one big txn; locks held; bloat. Pick deliberately based on item count and isolation needs.

### 21.14 Not retrying SerializationFailure

In PG SSI or any DB with snapshot isolation, txns can be aborted by the DB to preserve invariants. App code must catch and retry. Without retry, intermittent 500s.

### 21.15 Two-phase commit "as a default"

2PC is heavy and fragile. Reserve for the small set of operations that truly need cross-shard atomicity. Default to saga + compensation.

---

## 22. The Decision Framework

When deciding "transaction or not, and how":

### 22.1 Step 0 — What invariant are you protecting?

State it precisely. "Sum of debits = sum of credits", "Inventory >= 0", "User has exactly one default address", "Two doctors not both off-call".

If you can't state it, you don't need a transaction yet.

### 22.2 Step 1 — Single row?

If yes, atomic UPDATE/INSERT is sufficient. No transaction needed for atomicity (single statement is atomic). Use if conflict possible:
- OCC (`WHERE version = ?`) for low conflict.
- `FOR UPDATE` for medium conflict.
- Sharded pattern for hot row (split into N rows).

### 22.3 Step 2 — Multiple rows, same shard?

A transaction is the right tool. Pick isolation:
- Read Committed: 80% of cases.
- Repeatable Read / Serializable: invariants across rows.

Watch for:
- Lock ordering (deadlock prevention).
- Hot row contention.
- Long txn duration.

### 22.4 Step 3 — Multiple shards / databases?

Avoid 2PC unless absolutely necessary. Prefer:
- **Co-locate** the data if possible.
- **Saga** with compensation.
- **State machine** with per-step transactions.
- **Outbox + CDC** for downstream propagation.

### 22.5 Step 4 — External services?

Never hold a DB transaction across an external call. Decompose:
- Reservation (txn) → external call (no txn) → finalization (txn).
- Idempotency keys at every step.
- Compensations for rollback.

### 22.6 Step 5 — High throughput?

For >10K TPS on a hot row:
- Sharded counters.
- Streaming aggregation.
- Cached + reconciled.
- Redis or specialized counter store.

### 22.7 Step 6 — Cross-region?

Avoid cross-region txns where possible. Region-pin the data; do sagas across regions for the rare cross-region operation.

### 22.8 Step 7 — What does failure look like?

For each component:
- What if the txn aborts?
- What if the process crashes mid-txn?
- What if a retry happens?
- What if a compensation fails?
- What if a downstream service is down for 1 hour?

A transaction design is incomplete without a failure-mode map.

---

## 23. Mental Models a Staff Engineer Carries

1. **A transaction is a correctness contract with a runtime cost.** Use it where the contract is needed; pay the cost.

2. **Atomicity ≠ correctness.** Many invariants don't require atomicity (counters, cached aggregates, eventual data).

3. **Isolation level is per-txn, not per-database.** Default to READ COMMITTED; upgrade specific txns.

4. **Long-running transactions are bugs.** Maximum duration: seconds, not minutes.

5. **External calls live between transactions, never inside.** Decompose with reservations + finalizations.

6. **Idempotency keys make retries safe.** Pair every retry with one.

7. **Outbox eliminates the dual-write problem.** Default for "DB + event publication."

8. **2PC is a last resort.** Default to saga + compensation for cross-shard / cross-service.

9. **Sagas need workflow engines at scale.** Imperative orchestration is fragile.

10. **Optimistic concurrency wins at low conflict; pessimistic at high.** Know your contention.

11. **Hot rows are sharded counters waiting to happen.** Don't lock-fight; partition.

12. **Append-only ledgers > mutable balances** for audit-heavy / financial workloads.

13. **State machines beat imperative orchestration** for workflows that span time/services.

14. **`FOR UPDATE SKIP LOCKED` is a superpower** for queue-like workloads.

15. **DDL transactions exist (in PG); use them for atomic schema migrations.**

16. **Cross-region atomicity is bounded by the speed of light.** Region-pin where possible.

17. **Read-your-writes is a UX feature, not just a transaction property.** Design routing for it.

18. **Bulk ops belong outside transactions.** Decompose; prove invariants; atomic-swap final.

19. **Workflow tools (Temporal, Step Functions) abstract distributed transactions** correctly when used right.

20. **The transaction is the *easy* part. The retry, idempotency, compensation, observability around it is where the system lives.**

---

## 24. Closing Notes

There's a moment in every staff engineer's career where they realize: most of the systems they admire — Stripe's ledger, Amazon's checkout, Google's Spanner-backed Ads, Netflix's billing — are not built on textbook transactions. They are built on **decomposed state machines with single-row transactions inside each step, glued by idempotent retries, sagas, outboxes, and compensations**, with the transaction reserved as a precision tool for the small set of things that genuinely need atomicity.

That mindset shift — from "wrap it in a transaction" to "decompose into the smallest atomic operations" — is what scaling a transactional system to 1B users actually looks like. The transaction never goes away; it just stops being the load-bearing wall and becomes the rebar inside concrete.

Treat transactions as a **skill of restraint**: knowing when not to use them is just as important as knowing how to write them. The transaction's natural habitat is the smallest atomic unit, repeated millions of times. Let it live there. Build the rest of the system around it.

---

> Companion docs:
> - `statefulSystemsAtMAANGScale.md` — designing the stateful tier this lives in.
> - `statelessSystemsAtMAANGScale.md` — designing the stateless tier that talks to it.
> - `postgresReadOptimizationMAANGScale.md` — read-side optimization for Postgres-class engines.

Together: the four-document map of how to scale OLTP-style platforms at MAANG-grade load.