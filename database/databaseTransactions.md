# Staff-Level Questions: Database Transactions, Production Edge Cases, and Financial State Transitions

A deep-dive Q&A on managing DB transactions in production. Each question is framed at a staff+ level: not "what is ACID" but "how does your system behave when two services race to refund the same payment, and which invariant actually breaks first." PostgreSQL-flavored, but the reasoning translates to MySQL/InnoDB, Spanner, CockroachDB, and most OLTP engines.

---

## Table of Contents

1. Isolation Levels & Concurrency Anomalies
2. Locking: Pessimistic, Optimistic, and Advisory
3. Connection & Transaction Lifetime in Production
4. Deadlocks and Their Causes
5. Long-Running Transactions & Their Hidden Costs
6. Retries, Idempotency, and At-Least-Once Execution
7. Distributed Transactions, Outbox, and Saga
8. Financial Systems: Double-Spend, State Machines, Reconciliation
9. Valid State Transitions: Enforcement Patterns
10. Failure Modes: Crashes, Partitions, Clock Skew
11. Observability for Transactional Systems
12. Interview-Style Scenario Walkthroughs

---

## 1. Isolation Levels & Concurrency Anomalies

### Q1.1 — Walk me through every anomaly allowed by each isolation level, with a concrete SQL example of each.

There are five canonical anomalies. The SQL-92 standard only names three; write skew and lost update are the ones that get staff engineers in trouble.

| Anomaly            | Read Uncommitted | Read Committed | Repeatable Read | Serializable |
|--------------------|:----------------:|:--------------:|:---------------:|:------------:|
| Dirty read         | Possible         | Prevented      | Prevented       | Prevented    |
| Non-repeatable read| Possible         | Possible       | Prevented       | Prevented    |
| Phantom read       | Possible         | Possible       | Possible*       | Prevented    |
| Lost update        | Possible         | Possible       | Prevented**     | Prevented    |
| Write skew         | Possible         | Possible       | Possible        | Prevented    |

\* Postgres's RR (snapshot isolation) prevents phantoms for reads but not for range-based serialization constraints.
\** Prevented because RR will raise `could not serialize access due to concurrent update` on the second writer.

**Dirty read** — reading uncommitted data another transaction will roll back.
```sql
-- T1                              -- T2
BEGIN;
UPDATE accounts SET bal = 0
 WHERE id = 42;
                                    BEGIN;
                                    SELECT bal FROM accounts WHERE id = 42;  -- sees 0 (dirty)
ROLLBACK;
                                    -- T2 acted on phantom zero balance
```
Postgres never allows this — the lowest effective level is Read Committed.

**Non-repeatable read** — same row read twice returns different values.
```sql
-- T1                              -- T2
BEGIN;
SELECT bal FROM accounts WHERE id=42;  -- 100
                                    BEGIN; UPDATE accounts SET bal=200 WHERE id=42; COMMIT;
SELECT bal FROM accounts WHERE id=42;  -- 200 under RC; still 100 under RR
```

**Phantom read** — same predicate returns new rows.
```sql
-- T1
BEGIN;
SELECT count(*) FROM orders WHERE user_id=7 AND status='pending';  -- 3
-- (concurrently T2 inserts a pending order for user 7)
SELECT count(*) FROM orders WHERE user_id=7 AND status='pending';  -- 4 under RC
```

**Lost update** — classic read-modify-write over the same row.
```sql
-- Both T1 and T2 read bal=100, add 10, write 110. Net +10 instead of +20.
SELECT bal FROM accounts WHERE id=42;   -- 100
UPDATE accounts SET bal = 100 + 10 WHERE id=42;   -- last writer wins
```
Fix: `UPDATE … SET bal = bal + 10` (atomic), or `SELECT … FOR UPDATE`, or optimistic with a version column.

**Write skew** — the most subtle, and the one financial bugs almost always come from. Two transactions each read a consistent snapshot, each make a decision based on that snapshot, and their individual writes don't conflict — but the combined effect violates an invariant that held in each snapshot.
```sql
-- Invariant: at least one doctor must be on-call.
-- T1 and T2 each run, for different doctors:
SELECT count(*) FROM doctors WHERE on_call=true;  -- both see 2
-- Both decide "OK, I can go off-call" and:
UPDATE doctors SET on_call=false WHERE id=<mine>;
-- After both commit: 0 on call. Snapshot isolation allowed it.
```
Only SERIALIZABLE (and SSI in Postgres) blocks write skew. The practical alternatives: `SELECT … FOR UPDATE` to force materialized conflict, or a guarded `UPDATE` with a `WHERE` clause that checks the invariant, or a CHECK-enforcing trigger.

### Q1.2 — Why does Postgres's "REPEATABLE READ" not mean what the standard says, and why does that matter?

Postgres's RR is *snapshot isolation*. The SQL standard's RR is weaker (forbids only dirty + non-repeatable read and *may* allow phantoms); 
SI forbids phantoms in the read sense but still allows write skew. 
The gap matters because engineers assume "REPEATABLE READ = strong enough for money" — it isn't. MySQL InnoDB's RR actually goes further (uses gap locks for plain `SELECT` and locks for `SELECT … FOR UPDATE`) so it sometimes avoids write skew the way SI cannot. 
Know your engine, never assume portability.

### Q1.3 — What isolation level do you default to in a new service, and why?

Default: Read Committed. Pay the cost of stronger isolation only where correctness *requires* it.

- Read Committed: throughput, simple retry semantics, matches application intuition ("each statement sees committed data").
- Repeatable Read (SI): for multi-statement reports where you need a consistent point-in-time view across many reads.
- Serializable (SSI): for specific transactions with invariants that cross rows (on-call scheduling, inventory reservation, ledger balance checks). 
  Wrap retries in app code because SSI will raise `serialization_failure` (SQLSTATE 40001).

A common pattern: 95% of service code runs at RC, a handful of critical transactions explicitly `SET TRANSACTION ISOLATION LEVEL SERIALIZABLE` with retry.

---

## 2. Locking: Pessimistic, Optimistic, and Advisory

### Q2.1 — `SELECT … FOR UPDATE` vs `FOR NO KEY UPDATE` vs `FOR SHARE` vs `FOR KEY SHARE`. When do you use which, and what's the trap?

These differ in what concurrent operations they block.

| Clause           | Blocks concurrent | Blocks concurrent | Blocks concurrent | Common use |
|------------------|-------------------|-------------------|-------------------|-----------|
|                  | FOR UPDATE        | FOR NO KEY UPDATE | FOR SHARE         |           |
| FOR UPDATE       | ✅                | ✅                | ✅                | Strongest. Use when you will modify the row and/or its key columns. |
| FOR NO KEY UPDATE| ✅                | ✅                | ❌                | Weakest write lock. Use for non-key column updates; avoids blocking FK-check SHARE locks. |
| FOR SHARE        | ✅                | ❌                | ❌                | Read-intent-to-write-soon. Blocks actual writers but multiple readers coexist. |
| FOR KEY SHARE    | ✅ (if key change)| ❌                | ❌                | Weakest lock. What Postgres takes implicitly on FK-referenced rows. |

**Trap 1 — Foreign key contention.** Before PG 9.3, any `UPDATE` on a parent row took a `FOR UPDATE`-strength lock and would block every child 
`INSERT` doing an FK check. 9.3 introduced `FOR NO KEY UPDATE` / `FOR KEY SHARE` precisely to fix that. If you `SELECT … FOR UPDATE` on a parent row just to read it, you're serializing every child insert.

**Trap 2 — `FOR UPDATE` with `ORDER BY` + `LIMIT`.** You must order deterministically on an indexed column, or the planner may pick a different plan under contention and lock a different row than you expect. Always pair `FOR UPDATE` with `SKIP LOCKED` or `NOWAIT` in a queue-like pattern.

**Trap 3 — `FOR UPDATE` doesn't prevent the row being deleted.** It prevents *writes by others while we hold the lock*; but if we release without an update, someone can delete next.

### Q2.2 — When do you pick optimistic over pessimistic locking?

Pick **optimistic** (version column, compare-and-swap) when:
- Contention is low (most transactions succeed first try)
- Transaction spans a user think-time or network hop (editing a form, HTTP round-trip)
- You do *not* want to hold DB locks across application latency

Pick **pessimistic** (`SELECT … FOR UPDATE`) when:
- Contention is high and retry storms would dominate
- You need to serialize a short critical section (already inside a transaction)
- Correctness depends on blocking the second writer rather than losing its work

Optimistic pattern:
```sql
UPDATE orders
   SET status = 'shipped', version = version + 1
 WHERE id = $1 AND version = $expected_version;
-- Check row_count. 0 = concurrent modification, retry the whole read-modify-write.
```

Pessimistic pattern:
```sql
BEGIN;
SELECT status FROM orders WHERE id=$1 FOR UPDATE;
-- decide based on status
UPDATE orders SET status='shipped' WHERE id=$1;
COMMIT;
```

Staff-level nuance: in high-contention pessimistic workloads, your bottleneck is lock-wait time, not CPU. Monitor `pg_locks` and `pg_stat_activity.wait_event_type = 'Lock'`. 
In high-contention optimistic workloads, your bottleneck is retry amplification — 30% failure rate means 30% wasted work; at some point pessimistic wins even under latency.

### Q2.3 — When do you reach for advisory locks?

`pg_advisory_xact_lock(key)` and friends. Use when:
- You need a mutex scoped to a *logical* resource that isn't a row (e.g., "only one replica may run daily reconciliation")
- You want row-granularity mutual exclusion on a row that doesn't exist yet (insert-if-absent patterns)
- The lock must survive a `SELECT` that returned no rows (you can't `FOR UPDATE` a row that isn't there)

Trap: advisory locks are keyed by a 64-bit int. Engineers hash strings into the int and get collisions. Use two 32-bit ints (`pg_advisory_xact_lock(namespace_int, key_int)`) and reserve a namespace per use-case.

---

## 3. Connection & Transaction Lifetime in Production

### Q3.1 — What is the cost of an open transaction that just sits there?

In Postgres specifically, every open transaction has a snapshot. MVCC garbage (dead row versions produced after that snapshot was taken) cannot be cleaned by autovacuum 
until the oldest snapshot advances past them. Consequences:

- **Table and index bloat grow unboundedly.** A 30-minute idle transaction on a hot table can balloon it by gigabytes.
- **`pg_stat_activity.xact_start`** and `backend_xmin` is how you find the culprit. Alert on any transaction older than your SLO (e.g., 60s for OLTP).
- **Replica lag blows up.** Replicas apply vacuum decisions from primary; a long transaction on either side delays cleanup.
- **`max_connections` pressure.** A connection stuck in `idle in transaction` holds a slot.

Mitigation — always set these in production:
```sql
-- session/role level
SET idle_in_transaction_session_timeout = '30s';
SET statement_timeout = '5s';              -- or appropriate per workload
SET lock_timeout = '1s';                   -- fail fast on lock wait
```
Your connection pool (pgbouncer, pgx) should also enforce a max query time. Staff-level: these settings are not nice-to-have; missing any one of them is how a stuck backend takes down your database at 3am.

### Q3.2 — Transaction-mode pooling (pgbouncer) breaks what?

Under `pool_mode=transaction`, pgbouncer returns the server connection to the pool at COMMIT/ROLLBACK. That means anything *session-scoped* doesn't work:
- Prepared statements (server-side)
- `SET` commands (session GUCs) — `SET LOCAL` is fine because it's transaction-scoped
- Advisory locks taken as `pg_advisory_lock` (session) — use `pg_advisory_xact_lock` instead
- `LISTEN/NOTIFY` — subscriptions are per-session
- Temporary tables

The common production incident: a feature flag that does `SET search_path = foo, public` at session start works in dev (session pool) and silently misroutes queries in prod (transaction pool). Always use `SET LOCAL` inside a transaction, or server-side config.

### Q3.3 — Your API request takes 30s because of a slow downstream call. What transaction shape do you use?

**Do not** hold a DB transaction across the downstream call. Shape:
```
1. BEGIN; read inputs, validate, write intent row (status='pending'); COMMIT.
2. Call downstream (no DB transaction open).
3. BEGIN; update intent row based on result (status='completed' or 'failed'); COMMIT.
```
Why: open transactions over network I/O are the #1 source of `idle in transaction` pile-ups. Also: every external call should have a timeout tighter than your transaction timeout, not the reverse.

---

## 4. Deadlocks and Their Causes

### Q4.1 — What causes a deadlock, how does Postgres handle it, and how do you prevent it?

Deadlock: two transactions each hold a lock the other needs.
```
T1: UPDATE accounts WHERE id=1;  -- holds row 1
T2: UPDATE accounts WHERE id=2;  -- holds row 2
T1: UPDATE accounts WHERE id=2;  -- waits on T2
T2: UPDATE accounts WHERE id=1;  -- waits on T1  → DEADLOCK
```
Postgres runs a deadlock detector every `deadlock_timeout` (default 1s). It picks a victim (usually the one that can be rolled back with least cost) and aborts it with SQLSTATE 40P01.

Prevention patterns:
- **Consistent lock ordering.** Always lock rows in a global order (e.g., `ORDER BY id`). Critical for multi-row updates.
- **Single-statement updates with `WHERE`.** `UPDATE … WHERE id IN (…)` takes locks in index order, which is deterministic.
- **`SELECT … FOR UPDATE ORDER BY id`** before doing modifications.
- **Short transactions.** Most deadlocks happen because a transaction held too many locks for too long.

Example: transferring funds between two accounts *must* lock in `min(from,to), max(from,to)` order, not in `(from,to)` order.
```go
// Wrong: deadlock risk
tx.Exec("SELECT ... FROM accounts WHERE id=$1 FOR UPDATE", from)
tx.Exec("SELECT ... FROM accounts WHERE id=$1 FOR UPDATE", to)

// Right: deterministic order
a, b := from, to
if a > b { a, b = b, a }
tx.Exec("SELECT ... FROM accounts WHERE id IN ($1,$2) ORDER BY id FOR UPDATE", a, b)
```

### Q4.2 — You have a deadlock between an `INSERT` and an `UPDATE` with no obvious lock ordering. Where to look?

Usually it's an *implicit* lock. Candidates:
- **Foreign keys.** Insert into child table takes FOR KEY SHARE on parent; concurrent update on parent with key-touching change takes conflicting lock. Fix by using `FOR NO KEY UPDATE` or avoiding updates to referenced keys.
- **Unique indexes / ON CONFLICT.** Multiple writers hitting the same potential conflict key can serialize in surprising ways.
- **Serializable SSI.** Not strictly a deadlock — surfaces as `serialization_failure`, but from the app POV it looks similar. The fix is retry.
- **Triggers.** Are they touching another row in another table? That's your hidden second lock.

Diagnostic: enable `log_lock_waits = on`, `deadlock_timeout = 1s`. Postgres will log the blocking/blocked statements. For live diagnosis: query `pg_locks` joined with `pg_stat_activity`.

---

## 5. Long-Running Transactions & Their Hidden Costs

### Q5.1 — You have a nightly job that reads 100M rows in a single transaction. What breaks?

- **Vacuum starvation.** Snapshot held for the job's duration prevents autovacuum from cleaning tuples modified during that window. Several hours = massive bloat.
- **Replication lag.** Standby must keep those tuples visible for the snapshot if `hot_standby_feedback = on`.
- **WAL retention.** If the job also writes, WAL grows. If replication slots are open, WAL can fill the disk.
- **CPU for visibility checks.** Every read has to evaluate MVCC visibility against a now-stale snapshot.

Fix — chunk the work:
```sql
-- Instead of one transaction reading everything:
SELECT id FROM events WHERE processed_at IS NULL;

-- Chunk by keyset, each chunk its own transaction:
SELECT id FROM events WHERE processed_at IS NULL AND id > $last ORDER BY id LIMIT 10000;
-- Process. Commit. Next chunk.
```
Each batch is a fresh transaction with a fresh snapshot. Vacuum can advance between batches.

### Q5.2 — A long transaction is also in the middle of holding a row lock. What are the upstream effects?

Every transaction trying to write that row waits on it. This propagates:
- Waiters fill the connection pool (stuck in `Lock` wait state).
- New requests that need those connections time out.
- If the app retries aggressively, you get a thundering herd re-hitting the same lock.

Detection:
```sql
SELECT pid, age(clock_timestamp(), xact_start) AS xact_age,
       state, wait_event_type, wait_event, query
  FROM pg_stat_activity
 WHERE xact_start IS NOT NULL
 ORDER BY xact_start ASC;
```

Cure: kill it. Cancel with `pg_cancel_backend(pid)` (soft, lets it clean up); terminate with `pg_terminate_backend(pid)` (hard). Always set `statement_timeout` and `lock_timeout` so you never need to manually kill.

---

## 6. Retries, Idempotency, and At-Least-Once Execution

### Q6.1 — Why does every payment write need an idempotency key, and how do you implement it?

Failure modes that force retries:
- Client got a timeout before the response arrived; request actually committed.
- Gateway saw 502 from a healthy but slow backend.
- Worker crashed between "commit transaction" and "ack the queue message."

Without idempotency, all three cause double-charges. The contract: for a given `(tenant, idempotency_key)`, the *result* of the operation is the same — whether this is the first, second, or Nth attempt.

Implementation (Postgres):
```sql
CREATE TABLE idempotency_keys (
    key           TEXT PRIMARY KEY,
    request_hash  TEXT NOT NULL,       -- hash of the request body
    status        TEXT NOT NULL,       -- 'in_progress' | 'committed' | 'failed'
    response      JSONB,                -- the response we returned on first success
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at    TIMESTAMPTZ NOT NULL  -- e.g., now() + '24h'
);

-- On receipt of a request:
BEGIN;
INSERT INTO idempotency_keys (key, request_hash, status, expires_at)
VALUES ($1, $2, 'in_progress', now() + '24h')
ON CONFLICT (key) DO NOTHING
RETURNING *;

-- If INSERT returned a row, we're the first — do the work.
-- If it didn't, SELECT the existing row and:
--   - If status='committed' and request_hash matches, return the stored response.
--   - If status='committed' and request_hash differs, 409 Conflict (key reuse with different body).
--   - If status='in_progress' and the original tx is still running, 409 (or wait, app-specific).
--   - If status='in_progress' and tx is dead (check creation age), treat as failed and let caller retry.
COMMIT;
```

Staff-level nuances:
- **Hash the request body** so you catch "same key, different body" mistakes.
- **TTL the keys.** Don't keep them forever — but long enough to cover the longest retry window (24h is common; longer for batch jobs).
- **Store the response.** Otherwise a retry that "wins the race" against a still-running original will disagree.
- **Scope the key.** Include `tenant_id` in the PK so one tenant can't collide another's keys.

### Q6.2 — A retry hits the DB but the DB can't tell if it committed. What do you do?

This is the "fencing" problem. Scenarios:
- `COMMIT` was sent; network dropped before the ack returned.
- Client considers it failed; the DB considers it committed.
- Client retries — now you have two.

The only defenses:
1. **Idempotency keys** (above) — the second attempt no-ops.
2. **Conditional writes** — `UPDATE … WHERE status='pending' AND version=$v`. If the first succeeded, the second matches 0 rows.
3. **Outbox + external effect deduplication** — effects downstream (payment gateway call, email send) must themselves be idempotent.

Never rely on "I'll check if the row exists before inserting." That's TOCTOU. Always express the check as a single atomic statement.

### Q6.3 — Exponential backoff with jitter — how much, and why jitter?

Retries without jitter synchronize. If 1000 clients all get a 503 at t=0 and all retry at t=1s, you get 1000 simultaneous retries at t=1. Jitter de-correlates them.

Pattern (full jitter):
```
delay = random(0, min(cap, base * 2^attempt))
```
Typical values: base=50ms, cap=10s, attempts=5 for user-facing APIs; base=1s, cap=60s, attempts=10 for background workers.

For 40001 serialization failures and 40P01 deadlocks, retry is *expected*. Wrap the transaction in a loop; retry up to N times; surface only after persistent failure. 
For `23505` unique-violation, retry only if you're running an upsert-like pattern — otherwise propagate.

---

## 7. Distributed Transactions, Outbox, and Saga

### Q7.1 — Across services X and Y, how do you commit atomically?

You don't. "Distributed 2PC over microservices" is technically possible (XA, Spanner) but operationally a nightmare for most orgs: coordinator failure blocks resources, long locks across network, tight runtime coupling.

Patterns in order of preference:

**1. Outbox pattern (dual-write replacement).**
Write the business row and an outbox row in the *same local transaction*. A publisher polls outbox and publishes to Kafka, then marks the row sent. Atomic local commit + eventual delivery — never lose an event, may duplicate (consumer must dedupe).
```sql
BEGIN;
INSERT INTO orders(...);
INSERT INTO outbox(aggregate_id, event_type, payload, created_at)
VALUES (...);
COMMIT;
-- Publisher (separate process) drains outbox.
```

**2. Saga (compensating transactions).**
A multi-step workflow where each step is its own local transaction, and each step has a compensating action for rollback.
```
Reserve inventory   ↔  Release inventory
Charge card         ↔  Refund card
Ship order          ↔  Cancel shipment
```
Either orchestrated (a coordinator drives each step) or choreographed (events trigger next steps). Staff-level truth: choreography becomes un-debuggable past ~3 steps; most production sagas should be orchestrated.

**3. Event sourcing.**
The state is the log. Every change is an immutable event. Projections build current state. Transactions are naturally append-only.

### Q7.2 — Your outbox is backlogged by 10 minutes. What do you look at?

In order:
1. Publisher throughput — is it keeping up with insert rate? Check last-sent lag.
2. Kafka broker health — ISR, producer errors.
3. Outbox table bloat — if never cleaned up, scans slow down.
4. Poll query efficiency — must be `WHERE sent_at IS NULL ORDER BY id LIMIT 1000` with partial index `WHERE sent_at IS NULL`.
5. Downstream consumer lag — does it matter? If the outbox publisher is fine and Kafka is fine, the "10 minutes" may be consumer lag, not outbox lag.

Partial index is load-bearing:
```sql
CREATE INDEX outbox_unsent ON outbox(id) WHERE sent_at IS NULL;
```
Without it, the poll scans an ever-growing table.

### Q7.3 — A saga step's compensation fails. What now?

This is where saga production engineering lives. Options:
- **Retry the compensation.** The compensation must be idempotent. Most are.
- **Escalate to a human queue.** Financial sagas especially — a failed refund after a successful charge is a real-money open-liability event. Never silently "give up."
- **Hold the forward path.** Mark the saga `compensation_pending`. No new sagas may affect the same aggregate until this one resolves.

Design principle: compensations are not "undos." A charge has happened; you can't un-happen it. A refund is a *new forward event* that leaves the ledger in a compensated state. Keep both records.

---

## 8. Financial Systems: Double-Spend, State Machines, Reconciliation

### Q8.1 — Design a bank transfer between two accounts. Walk through every edge case.

**Schema:**
```sql
CREATE TABLE accounts (
    id        BIGSERIAL PRIMARY KEY,//use -> GENERATED ALWAYS AS IDENTITY --blocks manual inserts to the column unless explicitly overridden, which prevents the sequence from falling out of sync with table data
    balance   NUMERIC(20,4) NOT NULL CHECK (balance >= 0),
    version   BIGINT NOT NULL DEFAULT 0
);

CREATE TABLE transfers (
    id              UUID PRIMARY KEY,
    idempotency_key TEXT UNIQUE NOT NULL,
    from_account    BIGINT NOT NULL REFERENCES accounts(id),
    to_account      BIGINT NOT NULL REFERENCES accounts(id),
    amount          NUMERIC(20,4) NOT NULL CHECK (amount > 0),
    status          TEXT NOT NULL CHECK (status IN ('pending','committed','reversed','failed')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    reason          TEXT
);

CREATE TABLE ledger_entries (
    id           BIGSERIAL PRIMARY KEY,
    transfer_id  UUID NOT NULL REFERENCES transfers(id),
    account_id   BIGINT NOT NULL REFERENCES accounts(id),
    amount       NUMERIC(20,4) NOT NULL,   -- signed: -X on source, +X on dest
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(transfer_id, account_id)
);
```

**Happy path — single-DB transfer:**
```sql
BEGIN;

-- Step 1: idempotency check (outside tx in pre-step, but we INSERT the transfer row here)
INSERT INTO transfers (id, idempotency_key, from_account, to_account, amount, status)
VALUES ($1, $2, $3, $4, $5, 'pending')
ON CONFLICT (idempotency_key) DO NOTHING
RETURNING id;
-- if no row returned, look up existing and short-circuit.

-- Step 2: lock rows in canonical order (smaller id first) to avoid deadlock
SELECT id, balance FROM accounts
 WHERE id IN ($from_account, $to_account)
 ORDER BY id
 FOR UPDATE;

-- Step 3: validate balance
-- (fetch the source row's balance from the prior SELECT result; don't re-read)

-- Step 4: apply movement (double-entry)
INSERT INTO ledger_entries (transfer_id, account_id, amount)
VALUES ($tid, $from_account, -$amount), ($tid, $to_account, +$amount);

UPDATE accounts SET balance = balance - $amount, version = version + 1
 WHERE id = $from_account AND balance >= $amount;
-- If row_count = 0: insufficient funds. ROLLBACK; mark transfer 'failed'.

UPDATE accounts SET balance = balance + $amount, version = version + 1
 WHERE id = $to_account;

-- Step 5: finalize transfer
UPDATE transfers SET status='committed', completed_at=now() WHERE id=$tid;

COMMIT;
```

**Edge cases to name explicitly:**

| Edge case | Defense |
|-----------|---------|
| Duplicate request (client retry) | Idempotency key on `transfers` |
| Concurrent transfer from the same source (double-spend) | `FOR UPDATE` on source row; `CHECK (balance >= 0)`; the `UPDATE … WHERE balance >= $amount` conditional. |
| Deadlock when A→B and B→A transfer concurrent | Canonical lock order (`ORDER BY id`) |
| Transfer to same account | Reject at validation; never do `from=to` as a no-op — it masks bugs. |
| Negative or zero amount | `CHECK (amount > 0)` |
| Overflow on large sums | `NUMERIC(20,4)`, never `FLOAT` for money. |
| Account frozen/closed mid-transfer | Join to `accounts` with a status check under `FOR UPDATE`; reject if not 'active'. |
| App crash after step 4 before step 5 | The transaction didn't commit → all reverted. Safe. |
| App crash after COMMIT before returning to client | Idempotency key lets the client retry and read the committed state. |
| Precision drift on currency conversion | Compute in minor units (cents as BIGINT) *or* NUMERIC with explicit rounding rules; never store multiplied `float` values. |

### Q8.2 — Why do real ledgers use double-entry, and what invariant do you assert?

Every movement writes two entries summing to zero: debit on one account, credit on another. The invariant:
```sql
SELECT SUM(amount) FROM ledger_entries;   -- always 0 (across internal accounts)
```
This is the single best reconciliation check. Any single-entry write is a bug.

Production practice:
- **Immutable ledger.** `ledger_entries` is append-only. Corrections are new entries, not updates.
- **Account balance = SUM of ledger entries,** not a separately maintained column. If you *do* maintain a denormalized balance column for fast reads, assert it matches the SUM periodically.
- **Partitioning by time** (monthly) — ledgers grow forever.
- **Foreign keys are your friend.** Every ledger entry references the transfer that produced it.

### Q8.3 — A bug double-credited an account last week. How do you fix it?

Never `UPDATE` or `DELETE` ledger entries. The fix is a correcting entry:
```sql
INSERT INTO ledger_entries (transfer_id, account_id, amount, memo)
VALUES (<new-correction-transfer>, $acct, -$overcredit, 'Correction for transfer X: over-credit due to bug #1234');
```
Audit trail preserved. If regulators ask, you can show the full history.

Separately, you run a reconciliation job that verifies `SUM(ledger_entries) = 0` daily and flags discrepancies. Staff-level: an unreconciled ledger difference is a P0 incident; don't let these accumulate silently.

### Q8.4 — Cross-ledger (cross-DB) transfers — when do you need two-phase commit vs saga?

If both ledgers are under your control and in one DB: local transaction. Done.
If they're separate DBs (e.g., one per tenant, or one per bank):
- **2PC (XA)** if you genuinely need all-or-nothing atomicity and can tolerate the coordinator complexity. Rare.
- **Saga with reservation pattern** in practice:
  1. Reserve funds on source (debit to holding account, status='pending').
  2. Call destination API with idempotency key.
  3. On success: commit on source (release holding, settle). On failure: reverse on source.
- The reservation gives you a *time-boxed* promise: if the saga doesn't complete in N minutes, a reaper reverses it automatically.

---

## 9. Valid State Transitions: Enforcement Patterns

### Q9.1 — Model an order lifecycle and enforce that only legal transitions occur.

States: `pending → paid → shipped → delivered` with branches: `pending → canceled`, `paid → refunded`, `shipped → returned → refunded`.

```
┌─────────┐     ┌──────┐     ┌─────────┐     ┌───────────┐
│ pending │────▶│ paid │────▶│ shipped │────▶│ delivered │
└────┬────┘     └──┬───┘     └────┬────┘     └───────────┘
     │             │              │
     ▼             ▼              ▼
┌──────────┐  ┌──────────┐   ┌──────────┐
│ canceled │  │ refunded │   │ returned │──▶ refunded
└──────────┘  └──────────┘   └──────────┘
```

Three enforcement layers — defense in depth.

**Layer 1: SQL predicate in the UPDATE.**
```sql
UPDATE orders
   SET status = 'paid', paid_at = now()
 WHERE id = $1
   AND status = 'pending';   -- guard
-- Check row_count. 0 = illegal transition or concurrent change.
```
The guard in the `WHERE` clause makes the transition atomic: you cannot observe a partial intermediate state.

**Layer 2: CHECK or trigger enforcing the transition matrix.**
```sql
CREATE OR REPLACE FUNCTION orders_validate_transition() RETURNS TRIGGER AS $$
BEGIN
  IF OLD.status = NEW.status THEN RETURN NEW; END IF;  -- no-op update
  IF NOT (
       (OLD.status = 'pending'  AND NEW.status IN ('paid', 'canceled'))
    OR (OLD.status = 'paid'     AND NEW.status IN ('shipped', 'refunded'))
    OR (OLD.status = 'shipped'  AND NEW.status IN ('delivered', 'returned'))
    OR (OLD.status = 'returned' AND NEW.status =  'refunded')
  ) THEN
    RAISE EXCEPTION 'illegal transition: % -> %', OLD.status, NEW.status;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER orders_transition_guard
BEFORE UPDATE OF status ON orders
FOR EACH ROW EXECUTE FUNCTION orders_validate_transition();
```

**Layer 3: Application-side state machine.**
```go
var transitions = map[Status][]Status{
    Pending: {Paid, Canceled},
    Paid:    {Shipped, Refunded},
    Shipped: {Delivered, Returned},
    Returned:{Refunded},
}

func (o *Order) Transition(to Status) error {
    allowed, ok := transitions[o.Status]
    if !ok { return fmt.Errorf("terminal state %s", o.Status) }
    for _, s := range allowed {
        if s == to { o.Status = to; return nil }
    }
    return fmt.Errorf("illegal transition %s -> %s", o.Status, to)
}
```

Why three layers? Because any one can be bypassed:
- SQL guard alone: a DB admin running ad-hoc SQL bypasses application logic.
- Trigger alone: works for direct SQL but you still need the SQL `WHERE` guard to avoid lost-update races.
- App state machine alone: two app instances can each check-then-update; only the DB-level guard makes it atomic.

### Q9.2 — A payment state machine: `initiated → authorized → captured → settled`, with `refunded` and `disputed`. What's subtle?

- **Partial captures.** `authorized($100)` can be followed by `captured($70)`. The remaining $30 auto-expires. Your state machine must model "captured_amount" separately from "status".
- **Partial refunds.** `captured($100)` → `refunded($30)` → `refunded($50)`. Still partially refunded; only reaches "fully refunded" when cumulative refund equals captured.
- **Disputes can arrive against any state.** A chargeback 90 days after `settled` reverses downstream state. Treat `disputed` as an orthogonal flag, not a state replacement.
- **Idempotency on each transition.** Don't re-capture the same authorization twice.

Modeling tip: separate `status` (workflow state) from amount accumulators. Enforce amount invariants in CHECK constraints:
```sql
CHECK (captured_amount <= authorized_amount),
CHECK (refunded_amount <= captured_amount)
```

### Q9.3 — Where do you store the transition history?

Always. `status_changes` audit table:
```sql
CREATE TABLE order_status_changes (
    id          BIGSERIAL PRIMARY KEY,
    order_id    BIGINT NOT NULL REFERENCES orders(id),
    from_status TEXT NOT NULL,
    to_status   TEXT NOT NULL,
    actor       TEXT NOT NULL,   -- user id / system id
    reason      TEXT,
    metadata    JSONB,
    at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```
Written from the same trigger or application code path. Regulators, support, and your future self will thank you. The current `status` column is a projection; the log is the truth.

---

## 10. Failure Modes: Crashes, Partitions, Clock Skew

### Q10.1 — The application crashes mid-transaction. What is the state of the DB?

If `COMMIT` has not been acknowledged, the transaction has *not* committed. Postgres will, on the next startup or on the connection dying, roll it back as part of WAL recovery. Safe. No torn writes visible.

*But*: any side-effects outside the DB (external API calls, emails sent, Kafka publishes) that happened before the crash are not rolled back. 
This is why the outbox pattern matters — every external effect must be driven from a DB row the transaction also wrote, so it only fires after commit.

### Q10.2 — A network partition splits the primary from the application. What happens?

- App-side: all in-flight transactions time out. Connections drop.
- Primary-side: WAL retention may grow if the app was the main load, but DB keeps humming for any other clients.
- For replicas and failover: if an automated failover (e.g., Patroni) promotes a replica while the primary is still running, you get a split brain. 
- Fence aggressively — the old primary must self-demote or be killed before the new one accepts writes.

Critical for staff engineers: every DB write must tolerate at-least-once. 
Never trust "the app got a 200 back so it definitely committed" — build your downstream logic on idempotent consumption of DB state, not on app-side confirmations.

### Q10.3 — Clock skew between two nodes — when does it hurt you?

- **TTL expiry.** `expires_at < now()` evaluated on a replica with a skewed clock can early-expire. Fix: compute TTL on the primary at insert time and store an absolute timestamp; don't use replica clock for evaluation.
- **Distributed locks.** A lease issued for "30 seconds" on node A may be interpreted differently by node B. Use fencing tokens (monotonically increasing sequence numbers), not clock-based leases, for correctness.
- **Idempotency key expiry.** Don't use client-supplied timestamps for TTL. Always server-side `now()`.

For distributed transaction systems (Spanner, CockroachDB), TrueTime / HLC (hybrid logical clocks) solves ordering. For a single-DB app, the primary's clock is authoritative; design to that.

---

## 11. Observability for Transactional Systems

What to instrument, in priority order:

**Per transaction (spans):**
- Duration
- Isolation level
- Retry count
- Outcome (commit / serialization failure / deadlock / statement timeout / app error)

**Per connection pool:**
- Active / idle / idle-in-transaction count
- Wait time for a connection
- Max connection age
- Check-out duration histogram

**Per database:**
- Long-running transactions (`pg_stat_activity`, `xact_start`)
- Lock waits (`pg_locks` joined with activity)
- Bloat (`pg_stat_user_tables`, dead tuples)
- WAL generation rate, replication lag
- `pg_stat_statements` top-N by total_time and by stddev_time (variance is where bugs hide)

**Per business transaction (financial):**
- Attempt → success rate
- Time from `pending` → `committed`
- Reconciliation residual (`SUM(ledger_entries)`)
- Orphan count (`transfers` with status='pending' older than N minutes)

Alerts that actually fire in prod:
- `idle in transaction` > 60s: someone forgot a COMMIT.
- Deadlocks per minute > baseline × 5: a new code path introduced lock-order inversion.
- Serialization failures spiking: contention is rising; re-examine pessimistic vs optimistic choice.
- Reconciliation residual != 0: stop writes, page on-call.

---

## 12. Interview-Style Scenario Walkthroughs

### S1. Inventory reservation at Black Friday scale

**Problem:** Single SKU, 100 units, 50k concurrent add-to-cart requests in the first second. No overselling. No "sold out" shown to a user whose reservation just expired.

**Design:**
1. Don't decrement stock on add-to-cart — *reserve* with a TTL.
   ```
   reservations(id, sku, user_id, qty, expires_at, status)
   ```
2. Stock is `total - SUM(qty WHERE status IN ('reserved','sold') AND expires_at > now())`.
3. Insert reservation atomically with a capacity check:
   ```sql
   INSERT INTO reservations (sku, user_id, qty, expires_at, status)
   SELECT $sku, $uid, $qty, now() + '10 min', 'reserved'
    WHERE (
      SELECT coalesce(sum(qty), 0) FROM reservations
       WHERE sku=$sku AND status IN ('reserved','sold')
         AND (status='sold' OR expires_at > now())
    ) + $qty <= (SELECT total FROM inventory WHERE sku=$sku);
   ```
   This is a *predicate insert*: if the capacity check fails, 0 rows insert. No lock held across user think time.
4. Concurrent inserts under SERIALIZABLE catch the write-skew anomaly of two reservations each seeing "there's room." Under RC, use a row-level lock on the inventory row:
   ```sql
   SELECT total FROM inventory WHERE sku=$sku FOR UPDATE;
   ```
5. On checkout: transition `reserved → sold` with a `WHERE status='reserved' AND expires_at > now()` guard.
6. A reaper sweeps expired reservations back to available. Make sure the reaper's own update is idempotent and time-bounded.

**Edge cases:**
- User's reservation expires *during* checkout. Two options: extend reservation at "proceed to checkout" click (short extension — 2 min), or surface the conflict cleanly if capacity is now gone.
- Reservation idempotency: cart token → reservation_id mapping prevents a refresh from reserving twice.
- Hot SKU write contention: either shard the SKU's row into N "buckets" and sum, or switch to a Redis-based reservation with Redis as source-of-truth and a DB projection.

### S2. Idempotent "Apply Coupon" with race against another pricing update

Two writers: one applies a coupon, another updates shipping address (which can alter price via tax). Both read order total at time T, both compute new price, both write.

**Bad:** `UPDATE orders SET total=X WHERE id=Y`. Last writer wins. Coupon silently lost.

**Good, optimistic:**
```sql
UPDATE orders
   SET total = $new_total, version = version + 1, modified_by='coupon'
 WHERE id = $id AND version = $seen_version;
```
If row_count=0, re-read and re-compute. The coupon application must be deterministic on current state.

**Better:** model the *intent*, not the computed price. Store `coupons_applied` as a separate table; `total` is `SUM(line_items) - SUM(coupons) + tax(current_address)`. Each writer appends to the dimension it owns. No races.

Principle: reduce stateful updates to *append-only facts + derivations* wherever you can. Races disappear.

### S3. A queue built on Postgres — don't let two workers pick the same job

```sql
CREATE TABLE jobs (
  id          BIGSERIAL PRIMARY KEY,
  status      TEXT NOT NULL DEFAULT 'queued',
  run_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  locked_by   TEXT,
  locked_until TIMESTAMPTZ
);
CREATE INDEX jobs_ready ON jobs(run_at) WHERE status='queued';
```

Claim query:
```sql
UPDATE jobs
   SET status='running', locked_by=$worker, locked_until=now() + '60s'
 WHERE id = (
    SELECT id FROM jobs
     WHERE status='queued' AND run_at <= now()
     ORDER BY run_at
     FOR UPDATE SKIP LOCKED
     LIMIT 1
 )
 RETURNING *;
```
`SKIP LOCKED` means worker B doesn't block on worker A's in-flight claim — it just moves to the next row. Without it, N workers serialize on the same head-of-queue row.

**Edge cases:**
- Worker crashes holding a job: `locked_until` is a backstop. A sweeper resets expired locks.
- Job fails: explicit `status='failed'` + retry row is cleaner than auto-retry in place (preserves history).
- Fairness: if you need FIFO, you have it (ORDER BY run_at). If you need priority, add a column and include it in ORDER BY.
- Back-pressure: cap `locked_until` below your `statement_timeout`, so a stuck worker doesn't hold the row past the useful horizon.

---

## Closing: Staff-Level Mental Models

Across all of this, three mental models keep appearing:

1. **Every write is eventually observed by a retry.** Design every mutation to be idempotent, or to fail-closed on a duplicate attempt.

2. **An anomaly is the absence of a guard, not the presence of a bug.** Write skew isn't a DB quirk; it's the DB honestly telling you that snapshot isolation was too weak for your invariant. Either strengthen isolation, add a lock, or express the invariant in a `WHERE` predicate.

3. **The ledger is always right; the summary is always stale.** If your system has money in it, the append-only log of events is the source of truth. Balances, totals, statuses — these are projections. When they disagree with the log, the log wins, and the projection is the bug.

A staff engineer is the person who notices, in a design review, that `SELECT balance; if balance > amount: UPDATE balance = balance - amount` is the bug — and can explain precisely which isolation level, lock, or predicate fixes it without slowing down the happy path.
