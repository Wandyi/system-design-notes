# Event Sourcing and CQRS in Production — A Staff-Level Treatment

Event sourcing is one of those patterns that gets demonstrated with toy bank accounts and then misapplied to entire enterprises. This doc separates the cargo cult from the actual production engineering: when event sourcing earns its complexity, what you actually build, how it fails, and what trade-offs no tutorial discusses.

The three motivating scenarios:

1. **Payment system** — you need to know *exactly* how an account reached its current balance. A regulator can ask "explain this $4.27" three years from now and the answer must be reproducible to the byte.
2. **Calendar undo / time-travel** — "what did my schedule look like at 3 PM yesterday before Bob moved the meeting?" CRUD systems destroy this information at every UPDATE.
3. **Ads click aggregator** — every click is billable. Losing one is losing revenue. Counting one twice is invoicing fraud. The pipeline must be reprocessable without disturbing the ledger.

The thing all three share: **state is derived, history is the source of truth**. The moment you accept that, event sourcing stops feeling clever and starts feeling inevitable.

---

## 1. Why CRUD Falls Short — Mechanically

A normalized CRUD row stores *current truth*. Each UPDATE is a destructive write. Three structural problems for the systems above:

- **No history**: `UPDATE balance = 500` overwrites the prior value. You can keep an audit log table, but it's now disconnected from the source of truth — they can drift, and they will.
- **No replay**: when a billing bug is fixed, you can't recompute everything from scratch because the inputs are gone — only the outputs were stored.
- **Lossy semantics**: "balance went from 700 to 500" is ambiguous — was it a $200 withdrawal, a $500 deposit and $700 withdrawal, or a refund-and-rebuy? The *intent* of the change is not captured.
- **No native concurrency story**: two simultaneous UPDATEs need locks or version columns; the version column is essentially the start of an event sequence and most teams retrofit it badly.

The audit-log workaround is half-event-sourcing with worse properties: write twice (table + log), reconcile by hope, and pretend the log is canonical until it isn't.

---

## 2. Event Sourcing — The Core Idea

> The state of the system is a left-fold over an immutable, append-only sequence of events.

Concretely:
- The unit of write is an **event** — a fact that *happened*, named in past tense (`MoneyWithdrawn`, `MeetingRescheduled`, `ClickRecorded`).
- Events are **immutable**. You never UPDATE or DELETE. Compensation is a new event (`MoneyRefunded`, not "undelete the withdrawal").
- The store is **append-only**, ordered per **aggregate** (the consistency boundary — usually one account, one calendar event, one campaign).
- **Current state is a projection**: `state = fold(events, initial)`. Run that on demand or maintain it incrementally.

```
events for account-123:
  e1: AccountOpened{amount: 0}
  e2: MoneyDeposited{amount: 1000}
  e3: MoneyWithdrawn{amount: 300}
  e4: MoneyDeposited{amount: 50}

balance(now) = 0 + 1000 - 300 + 50 = 750
balance(after e2) = 1000        ← time travel for free
balance(if we replay without e3) = 1050  ← replay-after-bug-fix for free
```

That tiny example is also the entire payment system in miniature. The complexity that scales it up is operational, not conceptual.

---

## 3. CQRS — The Necessary Companion

Event sourcing optimizes writes (one append). It is **terrible at queries**: "show me everyone with balance > $10K" is a fold across every account's full history.

**CQRS** (Command-Query Responsibility Segregation) splits the model:
- **Write side**: commands → validated → produce events → appended to store.
- **Read side**: events → projected → denormalized read models tailored to query patterns.

Read models can be SQL tables, Elasticsearch indices, materialized views, key-value caches, OLAP cubes — whatever the query needs. Many read models can subscribe to the same event stream. They are **disposable**: drop and rebuild from event history any time the query shape changes.

The two sides are eventually consistent. **Embracing this is the price of admission.** If a use case requires read-your-write, you read from the write-side aggregate (which is still queryable by ID), not from the eventually-consistent projection.

```
Command ──▶ Aggregate ──▶ Event Store ──┬──▶ Projector A ──▶ Read DB (SQL)
                                        ├──▶ Projector B ──▶ Search Index
                                        ├──▶ Projector C ──▶ Cache
                                        └──▶ Projector D ──▶ Analytics warehouse
```

CQRS without event sourcing exists (and is sometimes the right call). Event sourcing without CQRS is rare — at any meaningful read rate the projections are mandatory.

---

## 4. Production Scenario 1: Payment System

This is the canonical fit. Let's get specific.

### 4.1 Aggregates and events

- **Aggregate**: `Account` (one customer, one currency). Consistency boundary: balance changes are linearizable per-account.
- **Events**: `AccountOpened`, `MoneyDeposited`, `MoneyWithdrawn`, `TransferInitiated`, `TransferCompleted`, `TransferFailed`, `AccountFrozen`, `AccountClosed`, `InterestAccrued`, `FeeCharged`, `RefundIssued`, `ChargebackInitiated`, `ChargebackResolved`.
- Each event carries: `account_id`, `version` (per-aggregate sequence number), `event_id` (UUID, idempotency), `occurred_at`, `recorded_at`, `causation_id` (the command that produced it), `correlation_id` (the trace that contains it), and a typed payload.

### 4.2 Command handling — the write path

```
handle(command) =
    1. Load aggregate: events = store.read(aggregate_id)
       state = fold(events, AccountState.empty)
    2. Validate command against state:
         WithdrawMoney requires balance >= amount AND status == active
    3. Decide events: [MoneyWithdrawn{amount, new_balance}]
    4. Append with optimistic concurrency:
         store.append(aggregate_id, expected_version=N, new_events)
         → on version conflict: retry from step 1 (or fail)
```

`expected_version` is the linchpin. Two concurrent withdrawals each load version 7, each compute `new_events`, both try to append at version 8 — only one wins; the other retries with the latest state. **No locks, no two-phase commits — Paxos-equivalent correctness via optimistic concurrency on a single key.**

### 4.3 Why this is non-negotiable for money

- **Audit**: regulators don't ask "what is the balance" — they ask "trace this $4.27." Event sourcing makes that trivial; with audit-log CRUD it's "we hope the log matches the table."
- **Dispute resolution**: a customer claims a charge is wrong. Replay the account from open to dispute date. Show every event. The bug, if any, is visible.
- **Reconciliation**: matches against bank statements compare events ↔ external ledger lines, not row states. Fixes are *new compensating events*, never UPDATEs.
- **Reprocessing for bug fixes**: a bug computed interest at 0.5 % too high for 6 months. With CRUD, you reverse-engineer corrections. With ES, you replay, swap in the corrected interest projection, diff against current, emit `InterestAdjusted` events for the deltas. Mathematical, auditable.
- **Double-entry bookkeeping** falls out naturally: every transfer is two events (`MoneyDebited` on source, `MoneyCredited` on destination) tied by a shared `causation_id`. Reconciliation checks the global invariant `sum(credits) == sum(debits)`.

### 4.4 The cross-aggregate problem — Sagas

`TransferMoney(from=A, to=B, amount=100)` touches two aggregates. Strict ACID across aggregates is what we deliberately gave up by sharding per-account. The solution is a **saga** (a.k.a. process manager):

```
1. Command on A → MoneyHeld{amount: 100}                   [version A_n+1]
2. Saga listens, sends Command on B → MoneyCredited{...}    [version B_m+1]
3a. On success: command on A → MoneyTransferred{}            [version A_n+2]
3b. On failure: compensating command on A → MoneyReleased{}  [version A_n+2]
```

Properties:
- Each step is a single-aggregate transaction → simple, fast.
- The saga itself is durable (also event-sourced — `TransferSagaStarted`, `LegCompleted`, `TransferSagaCompleted`).
- Failure modes: leg 2 fails after leg 1 → compensating event on A. Leg 2 succeeds but ack is lost → idempotent retry (B sees same event_id and discards).
- Eventually consistent. The hold makes the temporary inconsistency invisible to the customer (the held amount doesn't appear as available balance).

### 4.5 Read side — projections

| Read model | Source | Built for |
|---|---|---|
| `account_balances` (SQL) | All `Money*` events | Real-time balance API |
| `transaction_history` (SQL) | All events with display metadata | Statements UI |
| `daily_balance_snapshot` (OLAP) | All events bucketed by day | Analytics, reporting |
| `aml_signals` (Kafka topic) | `MoneyDeposited`, `MoneyWithdrawn` filtered by amount | Anti-money-laundering pipeline |
| `customer_search` (Elasticsearch) | `Account*` lifecycle events | Support search |

Each projection has its own checkpoint (last `(stream, version)` processed). When a projection lags or breaks, it can be **rebuilt from scratch** — the only cost is replay time, not data loss.

---

## 5. Production Scenario 2: Calendar — Undo and Time Travel

Calendar's product feature "show what my schedule looked like yesterday at 3 PM" is impossible in CRUD without manual versioning of every event. With event sourcing it's a one-liner: `replay until t = yesterday 3 PM`.

### 5.1 Events

`MeetingCreated`, `MeetingTimeChanged`, `MeetingTitleChanged`, `AttendeeInvited`, `AttendeeResponded`, `MeetingCancelled`, `RecurrenceOverridden`, `RecurrenceTruncated`. Each carries the *delta*, not the full new state — preserves intent.

### 5.2 Time-travel projection

```
def schedule_at(user_id, t):
    relevant_streams = streams_user_participates_in(user_id, as_of=t)
    state = {}
    for stream in relevant_streams:
        events = event_store.read(stream, until=t)
        state[stream] = fold(events, MeetingState.empty)
    return [m for m in state.values() if m.intersects(today_window) and m.status != cancelled]
```

For "yesterday's view at 3 PM": `t = yesterday 3 PM`. Done.

### 5.3 Undo

Undo is **not** "delete the last event" — events are immutable. Undo is a new compensating event:

```
last_event = MeetingTimeChanged{from: 9am, to: 11am, version: 7}
undo →     MeetingTimeChanged{from: 11am, to: 9am, version: 8, undo_of: 7}
```

This:
- Preserves audit trail (the user *did* change it, then changed it back).
- Plays nicely with concurrent writes (Bob also moved the meeting between Alice's change and Alice's undo → conflict surfaces, undo can be partial, UI can prompt).
- Generalizes to multi-step undo stacks.

### 5.4 The recurrence-override case

Recurring meetings (see `googleCalendarHLD.md`) have master + override semantics. Both fit naturally:
- `RecurrenceMasterCreated` (the RRULE).
- `OccurrenceOverridden{recurrence_id, new_time}` for "this instance only".
- `RecurrenceTruncatedAt{recurrence_id}` + new master for "this and following".

The projection materializes the visible window from these events. Undo of "this and following" untruncates the original master and tombstones the new one — *as new events*, never as edits.

### 5.5 Why CRUD breaks here

You'd need: `meetings` table + `meeting_versions` history table + triggers + careful "as-of" SELECTs + manual undo logic that re-INSERTs old rows. People build this. It's brittle, slow, and the second the trigger breaks (it always does), history is silently corrupt. ES makes the versioned representation the *only* representation.

---

## 6. Production Scenario 3: Ads Click Aggregator

This one is interesting because it's where ES and **stream processing** become indistinguishable.

### 6.1 The shape

- 10⁶ click events/sec at peak.
- Each click is a billable fact: `ClickRecorded{click_id, ad_id, advertiser_id, user_id, timestamp, ip, user_agent, ...}`.
- Multiple downstream consumers: realtime dashboards (1-min latency), billing (hourly), fraud detection (seconds), analytics warehouse (daily).
- Reprocessing is regular: fraud rules change, attribution windows change, deduplication logic improves.

### 6.2 Event store = Kafka (with discipline)

The event log *is* Kafka topic `clicks.v1`, partitioned by `ad_id`. Retention: forever (tiered storage to S3 after 7 days). Compacted? **No** — every click is a unique fact; we want them all.

Each consumer is a projection:

| Consumer | Input | Output | Replayable? |
|---|---|---|---|
| Realtime aggregator (Flink) | `clicks.v1` | Redis sorted sets, per-ad counters | Yes, by reset offset |
| Billing | `clicks.v1` | Postgres `billing_events` by hour | Yes, by reset offset + idempotent insert |
| Fraud | `clicks.v1` + `users.v1` | `fraud_signals.v1` | Yes |
| Warehouse | `clicks.v1` | BigQuery / Snowflake | Yes |

Each consumer maintains its own **offset** (= "version" in ES terms). Reprocessing: rewind offset, replay. The event store doesn't care.

### 6.3 The "every click matters" property

Two failure modes that ES + Kafka handle together:

**(a) Lost click**: handled by the producer side — the click ingest writes to Kafka with `acks=all`, replication factor 3, and the client retries on transient failure. Producer-side idempotence (`enable.idempotence=true` + producer ID) ensures retries don't double-write.

**(b) Double-counted click**: handled at the consumer with **idempotency keys**. Each downstream insert is keyed on `click_id`:
```
INSERT INTO billing_events (click_id, advertiser_id, amount, ...)
ON CONFLICT (click_id) DO NOTHING
```
At-least-once delivery + idempotent consumer = effectively-exactly-once.

### 6.4 Reprocessing for billing logic changes

Q3 you find that one IP range was generating bot clicks; you've already invoiced 3 months. With ES:

1. Spin up a new projection `billing_v2` consuming from `clicks.v1` from the offset corresponding to the start of the disputed period.
2. Apply the new fraud rule.
3. Diff `billing_v2` vs `billing_v1` → produce credit memos via new events `BillingAdjusted{amount, reason}`.
4. Switch the read API to `billing_v2` once it caught up.
5. Retire `billing_v1`.

Without ES this is forensic accounting. With ES it's a deployment.

### 6.5 The window-watermark problem (out-of-order events)

Real ads pipelines see click events arrive late (mobile retries, network partitions). The naive "tumbling 1-hour window" misses them. Solutions:

- **Watermarks** (Flink-style): the projection tracks "we've probably seen all events with `timestamp ≤ t`" via a heuristic (max-seen-timestamp − allowed-lateness). Closes windows past the watermark. Late events go to a side-output stream and produce **late-correction events** in the projection.
- **Reprocessable windows**: every window's output is a *new event* in a downstream stream, never an UPDATE. If a late event causes recomputation, emit a delta event. Same idea as compensating events in payments.

---

## 7. The Mechanics — What You Actually Build

### 7.1 Event store options

| Store | Strengths | Trade-offs |
|---|---|---|
| **EventStoreDB / Axon** | Purpose-built; per-stream optimistic concurrency; subscriptions; projections built-in | Smaller community; another datastore to operate |
| **Postgres / SQL** | Operationally familiar; transactions; ACID | DIY everything; partitioning needed past TBs; subscription bolts on (CDC via Debezium) |
| **Kafka** | Massive throughput; native streaming; great for log-style ES | Per-partition ordering only; no per-aggregate optimistic concurrency without external coordination; retention/compaction quirks |
| **DynamoDB** | Managed, scalable; transactional writes per partition | DIY tooling; query patterns rigid |
| **Cassandra/Scylla** | Time-series shape fits; high write throughput | Read-after-write consistency complications; LWT for OCC is expensive |

The **shape** matters more than the brand:
- Schema: `(stream_id, version, event_id, type, payload, metadata, recorded_at)`
- Primary key: `(stream_id, version)`
- Indexes: by `stream_id`, by `event_type` (for projections), by `recorded_at` (for replay-from-time)
- Constraint: unique `(stream_id, version)` enforces optimistic concurrency at the DB layer.

### 7.2 Optimistic concurrency control

```
APPEND with WHERE current_max_version(stream_id) = expected_version
   IF rows_affected = 0: throw ConcurrencyException
```

In Postgres:
```sql
INSERT INTO events (stream_id, version, type, payload)
SELECT $1, $2, $3, $4
WHERE COALESCE((SELECT MAX(version) FROM events WHERE stream_id = $1), -1) = $5
RETURNING version;
-- $2 = expected_version + 1, $5 = expected_version
```

In Kafka: there isn't one natively. Patterns to emulate:
- **External lock service** (Zookeeper, etcd) per aggregate — slow, defeats purpose.
- **Single-partition single-consumer per aggregate group**: the consumer is the only writer for its keys → no contention possible. Best Kafka pattern.
- **State-store-backed OCC** (Kafka Streams / Flink with per-key state): the processor reads state-store version, validates command, writes event + new state — the partition is the lock.

### 7.3 Snapshots — when fold-from-zero gets too slow

Aggregate with 100 K events → fold takes a second. Take a snapshot every N events:

```
snapshot{stream_id, version: N, state}
```

Load: `state_at_N + events_after_N` instead of `all_events`.

Strategies:
- **Every N events** (simple, predictable): N = 1000 typical.
- **Time-based** (every day): bounded recovery cost regardless of write rate.
- **Hybrid**: whichever fires first.
- **Lazy**: take a snapshot when an aggregate hot-loads with > N events to fold.

Snapshots are an **optimization**, not source of truth. If a snapshot is corrupted or stale (because a projection logic bug was discovered), discard it and re-fold. Never serve a snapshot you can't reproduce from events.

### 7.4 Event schema evolution — the part that breaks production

Events live forever. Code changes weekly. Every event you ship today must be deserializable by every future version of the system. Strategies, in increasing rigor:

**Tolerant reader**: deserializers ignore unknown fields, default missing fields. Works for additive changes only.

**Versioned events**: `MoneyWithdrawn.v1`, `MoneyWithdrawn.v2`. New version is a new type.

**Upcasters**: at read time, transform old-version payloads into the current shape before passing to the aggregate. Upcasters are pure functions, chained by version.

```
read v1 from disk → upcast v1→v2 → upcast v2→v3 → hand to aggregate (which only knows v3)
```

**Never** rewrite history to migrate data — it breaks audit, breaks replay determinism, and corrodes trust in the store. Upcasters keep history immutable while letting code evolve.

**Event design discipline** that prevents most pain:
- Past tense, named for *what happened*, not *what changed*. `MeetingTimeChanged` not `MeetingUpdated`.
- Carry the *intent* and the *minimum context* — not whole entity snapshots, not just raw deltas.
- No code references inside payloads (no enum ordinals — use strings; no class names — they get refactored).
- Treat the event schema like a public API: review every change, version every break.

### 7.5 Projections — operational realities

- **Stateful**: each projection holds its own checkpoint (last processed event id) and its own materialization. Atomic update of `(state, checkpoint)` per event batch (in one DB transaction or via idempotent writes) — otherwise you double-apply on crash recovery.
- **Rebuildable**: drop the table, reset checkpoint to 0, replay. The whole appeal of CQRS is that read models are disposable.
- **Multiple versions side by side** (blue/green): build `accounts_v2` projection alongside `accounts_v1`; cut over once it caught up; delete v1.
- **Partitioned**: high-volume projections shard by aggregate id to parallelize. Each partition has its own checkpoint.
- **Replay throttling**: rebuild from beginning of time can saturate downstream stores. Throttle event-per-second on rebuild paths; bump priority on live tail.

### 7.6 The outbox pattern (the integration glue)

When a command needs to atomically update an aggregate *and* publish an event externally (e.g., to Kafka for downstream subscribers), you have two writes — to the event store and to the broker — and they can desync.

**Outbox**: in the same DB transaction, write the event row to a `outbox` table. A separate poller reads `outbox`, publishes to Kafka, marks as published. At-least-once external publication, atomic with the aggregate write.

If your event store *is* Kafka, the outbox is unnecessary at that boundary — but you might still need it from Kafka to a third system (analytics, partner webhook).

### 7.7 Read-your-write and consistency

Embracing eventual consistency on the read side doesn't mean every UI is jarring:

- **For the user's own writes**: after the write, return the new state from the aggregate (write side) directly. UI updates optimistically; the projection catches up within ms.
- **For other users' writes**: tolerate the lag. Most products do.
- **Where it must be strong** (security/auth): read from the write side, not from a projection.

Common pattern: write returns the new aggregate version + state in the response. Subsequent reads from projections include `If-Min-Version` semantics; if the projection hasn't caught up, the read waits up to N ms or falls back to write side.

---

## 8. When NOT to Use Event Sourcing

Cargo cult is real. ES costs are real. The pattern is *wrong* when:

- **Domain is genuinely stateless / CRUD-shaped**: a contact list, a settings page. UPDATE the row. Move on.
- **No replay value**: nobody will ever ask "what did this look like a year ago?" Don't pay for time travel you won't use.
- **No audit requirement**: not every domain has regulators.
- **Read-mostly with no derived projections**: if your only access pattern is `SELECT * FROM x WHERE id = ?`, ES is overkill.
- **Team isn't ready** for the operational surface: schema evolution, idempotent consumers, eventual consistency in UX, debugging via projection diffs. ES is a *culture* shift; without buy-in it becomes a half-implemented hybrid that combines the worst of both worlds.

A useful rule: **if your domain experts naturally describe state changes in past tense facts** (deposits, cancellations, shipments, clicks), ES fits. If they describe entities and properties (a customer record, a settings blob), it doesn't.

CQRS *without* ES is often the right middle ground: keep CRUD writes, project to read models for query needs.

---

## 9. Trade-Offs and Anti-Patterns

| Decision | Trade-off |
|---|---|
| **Append-only storage** | History is preserved; storage grows monotonically. Archive cold streams to object storage; bring back via lazy hydration. |
| **Eventual consistency on reads** | Projections lag (ms-seconds). For every UX surface, decide explicitly: tolerate, optimistic UI, or read from write-side. |
| **Schema evolution complexity** | Every event lives forever; every shape change is a versioning event. Mitigate with strict review and upcasters. |
| **Operational surface** | Event store, projection workers, broker, snapshots, outboxes — more moving parts than CRUD. Pay only when the domain demands it. |
| **GDPR / right-to-erasure** | Immutability vs deletion. Solutions: per-subject crypto-shredding (encrypt PII with per-user key; throw away key on erasure → events become unreadable for that subject). Or scrub PII payloads to tombstones, leaving the structural event. Both are non-trivial; design for it from day one if you're in scope. |
| **Aggregate boundaries are hard** | Too small = sagas everywhere; too big = lock contention and large folds. Re-aggregating is painful — invest in DDD context modeling up front. |

### Anti-patterns to flag in design review
- **Event-as-CRUD**: events named `EntityUpdated` carrying the whole new entity. Loses intent, loses replay value. Force past-tense, intent-named events.
- **Putting commands in the event store**: events are *what happened*; commands are *what was asked*. Don't log requests as events; log their outcomes.
- **Cross-aggregate transactions inside the write side**: defeats the consistency boundary. Use a saga.
- **Stale snapshots as truth**: serving a snapshot whose computation logic has since changed. Always version snapshots; invalidate on logic change.
- **Ignoring event schema like it's "internal"**: it's a contract — across versions of your own service, across teams consuming the projections. Treat as public API.
- **Projecting to a single monolithic read model**: misses the whole point. Build narrow projections per query pattern.
- **No idempotency on consumers**: at-least-once delivery hits, double-applies happen, ledgers diverge. Every consumer must be keyed on `event_id`.
- **No replay rehearsal**: you discover replay is broken when you need it. Run a replay drill quarterly.

---

## 10. Operational Practices

- **Replay drills**: rebuild a major projection from zero every quarter. Catches schema-evolution bugs before they bite in incident.
- **Event volume forecasting**: project storage and replay time at 10× current rate. Plan archive tiers.
- **Per-stream metrics**: write-version monotonicity, projection lag, snapshot hit rate, upcaster CPU.
- **Audit query as a product**: "explain this state at this time" should be a one-click tool for support, not a custom SQL forensics adventure.
- **Schema registry**: every event type and version registered, validated at write, validated at read. Avro/Protobuf/JSON-Schema. Reject unknown event types from misconfigured producers — better to fail loudly than silently corrupt the log.
- **Out-of-band correctness checks**: cron jobs that re-fold a sample of aggregates and assert projection state matches. Catches projection drift early.
- **Disaster recovery**: event store is the gold copy. Backups + cross-region replication of the store, not of projections (those are derived). Recover by restoring store + replaying.

---

## 11. The Three Scenarios — Side by Side

| Aspect | Payment | Calendar | Click Aggregator |
|---|---|---|---|
| Aggregate | Account | Meeting / series | (none — pure stream) |
| Cardinality | 10⁸ aggregates, ~10⁴ events each | 10⁹ aggregates, ~10² events each | 10⁶ events/sec, no aggregate |
| Why ES wins | Audit, replay, dispute resolution | Time-travel, undo, recurrence semantics | Reprocessing, idempotency, billing correctness |
| Store | Postgres or EventStoreDB; transactional OCC | Postgres or specialty store | Kafka with tiered S3 retention |
| Cross-aggregate | Transfers via saga | Cross-calendar invitations via saga | N/A |
| Projection examples | balance, statement, AML signals | calendar view, free/busy, search | hourly billing, fraud features, warehouse |
| Hardest problem | Schema evolution on a 10-year audit horizon | Recurrence override semantics with time-travel | Watermarks + late-event corrections |

The patterns rhyme: events as immutable facts, projections as disposable views, replay as the universal remedy. The scale axis is different, but the architecture is the same.

---

## 12. What Makes This Staff-Level

1. **Naming the consistency boundary explicitly** (the aggregate) and accepting eventual consistency *between* aggregates as the price of single-aggregate strong consistency.
2. **Optimistic concurrency control** as the linchpin instead of locks/2PC. Knowing how to emulate it on Kafka (single-writer-per-key) when the store doesn't natively support it.
3. **Schema evolution discipline**: tolerant readers, upcasters, schema registry — never rewriting history.
4. **Sagas for cross-aggregate workflows** with compensating events, not distributed transactions.
5. **Idempotent consumers** by `event_id`; at-least-once + idempotency = effectively-exactly-once.
6. **Watermarks and late-event handling** for streaming projections — knowing why naive tumbling windows are wrong.
7. **GDPR pragmatism**: crypto-shredding, not "well, we're append-only so we can't comply."
8. **Knowing when *not* to use ES**: most domains aren't event-shaped, and forcing them into ES is a multi-year regret.
9. **Operational reflexes**: replay drills, schema registry, projection-vs-store correctness audits.
10. **Treating events like a public API**: review, version, never break unannounced.

The pattern's power is not "we can replay history." It's that **history, intent, and current state become a single coherent thing**, and the system inherits properties (audit, time-travel, reprocessing, replayability) that CRUD systems can only fake with ever-growing complexity.