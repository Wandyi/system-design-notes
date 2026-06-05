# 20 · Distributed Systems Fundamentals (Staff Depth)

Concepts every Staff candidate must know cold. The interview won't test you on textbook definitions, but it *will* test whether you can apply these concepts when the design gets adversarial. If the interviewer asks "what happens during a network partition?" and your answer is hand-wavy, you've signalled Senior, not Staff.

This is dense by design. Skim sections you know; read deeply on the ones you don't.

## 20.1 CAP and PACELC

### CAP (Brewer)

In the presence of a **P**artition, you must choose between **C**onsistency and **A**vailability.

- **CP** systems sacrifice availability during partition: refuse to serve.
- **AP** systems sacrifice consistency: continue serving, allow divergence, reconcile later.

CAP is a fact, not a marketing slogan. Most production systems are *partially* CP and *partially* AP depending on configuration.

### PACELC (Abadi)

The richer formulation:

- IF **P**artition, choose between **A**vailability and **C**onsistency.
- ELSE (no partition), choose between **L**atency and **C**onsistency.

Example mappings:
- **Espresso**: PC/EC — strongly consistent when partitioned (refuses divergent writes), consistent in steady state.
- **Voldemort RW**: PA/EL — available during partition, accepts divergence; in steady state, optimizes for latency over consistency.
- **DynamoDB single-region**: PA/EL by default; configurable.
- **Kafka**: PC/EC for committed writes (`acks=all`); CP-leaning.

Use this vocabulary in interviews.

## 20.2 Consistency models

A spectrum from strict to permissive:

### Linearizability

The strictest. Every operation appears to happen instantly at some point between its start and end. All clients see a single consistent timeline. Equivalent to "thinking like a single machine."

- **Cost**: at least one round-trip for every write; typically requires consensus.
- **Achievable via**: Raft / Paxos consensus; single-leader replication with synchronous quorum.

### Sequential consistency

All clients agree on an order, but it need not be real-time. Weaker than linearizability.

### Causal consistency

If A causes B (A happens before B), all clients see A before B. Non-causal operations can be seen in any order.

- **Cost**: lower than linearizability; requires tracking causal dependencies (vector clocks).
- **Useful for**: collaborative editing, social timelines.

### Read-your-writes

After you write, you read your own write. Other clients may still see the old value.

### Monotonic reads

Once you've seen value V at time T, you don't see an older value later.

### Eventual consistency

Eventually, if no new writes, all replicas converge. No bounds on how long.

### Strong vs. weak in practice

In an interview, **always state explicitly**:
- "We need read-your-writes for the publisher; eventual for followers."
- "We need linearizability on this counter; everything else can be eventual."
- "We can tolerate stale reads up to N seconds here."

Vague "strong consistency" or "eventual consistency" without scope = junior signal.

## 20.3 Consensus: Raft and Paxos

The mechanism for replicated, linearizable state.

### Raft (the one to know cold)

Five servers, one is the leader. Leader appends to its log, replicates to followers; once a majority ack, the entry is committed.

Key ideas:
- **Leader election** via randomized timeouts.
- **Term numbers** monotonically increasing; higher term wins.
- **Log replication** — leader pushes entries; followers ack.
- **Safety**: a committed entry stays committed even after leader change; new leaders must include all committed entries from previous terms.

### Multi-Paxos

Conceptually similar to Raft but with subtler invariants. Harder to teach (the original Paxos paper is famously dense). Most modern systems use Raft for understandability.

### Where consensus is used at LinkedIn

- **Kafka KRaft** — controller metadata.
- **ZooKeeper** — ZAB protocol (Raft-ish).
- **Helix** — uses ZooKeeper underneath.
- **Espresso leader election** — via Helix / ZooKeeper.
- **Etcd / Consul** — in newer K8s-native components.

### When NOT to use consensus

Consensus is expensive (at least one network round trip per operation). For high-throughput data, leader-follower replication with async followers is often enough. Use consensus for *metadata* (membership, configuration), not for *every data write*.

## 20.4 Replication strategies

### Leader-follower (primary-replica)

- One leader handles writes; followers replicate.
- Pro: simple, strong consistency on leader.
- Con: leader is a SPOF; failover takes time; cross-DC requires async or you accept the latency.

### Multi-leader (active-active)

- Multiple regions accept writes.
- Pro: lower write latency for distant regions.
- Con: conflict resolution required.

### Leaderless (Dynamo-style)

- Any node can accept writes; quorum reads/writes.
- Pro: no single point of failure.
- Con: complex consistency semantics; vector clocks / version vectors.

### Synchronous vs. asynchronous

- **Synchronous**: leader waits for follower ack before responding. Stronger durability, higher latency.
- **Asynchronous**: leader responds immediately; followers catch up. Lower latency, risk of data loss on leader failure.
- **Quorum-based**: ack from majority. Middle ground.

### LinkedIn examples

- **Espresso**: leader-follower per partition; synchronous in-DC, async cross-DC.
- **Kafka**: leader-follower per partition; quorum ack (`min.insync.replicas`).
- **Voldemort RW**: leaderless; quorum.
- **Venice**: all replicas consume Kafka independently — "log-as-the-source-of-truth" pattern.

## 20.5 Sharding / Partitioning

### Hash-based

`partition = hash(key) % N`. Uniform distribution.

- **Con**: changing N is expensive (most keys move).
- **Solution**: consistent hashing — keys map to a ring; only `1/N` keys move per node change.

### Range-based

`partition = bucket containing key`. Useful for range scans.

- **Con**: hotspots (popular range overloaded).

### Composite / hierarchical

E.g., shard by `(tenant_id, item_id)` so each tenant is co-located, but split tenants across shards.

### LinkedIn examples

- **Espresso**: hash by primary key (typically). 1024 partitions per DB.
- **Kafka**: hash by message key (per topic). Partition count fixed at topic creation.
- **Pinot**: by hash on a chosen column; segments are time-bucketed.
- **Venice**: hash by key.

### Resharding

Notoriously hard online.

Patterns:
- **Pre-sharded**: choose a large enough partition count up front (e.g., 1024 even if you have 16 nodes).
- **Virtual nodes**: each physical node holds many virtual nodes; can move virtual nodes for rebalancing.
- **Dual-write + migration**: build the new sharding online; cut over.

## 20.6 Idempotency

Critical for distributed systems where retries are common.

Patterns:
- **Idempotency key**: client generates a UUID; server dedupes.
- **Versioning**: client sends version; server rejects if version mismatch (optimistic concurrency control).
- **Natural idempotency**: operations like "set X = 5" are idempotent.

Anti-pattern: relying on "client should not retry" — they will, somehow.

## 20.7 Backpressure and flow control

Without backpressure, a fast producer crashes a slow consumer.

Mechanisms:
- **Token buckets** / **leaky buckets**.
- **Sliding window** (TCP-style).
- **Queue with bounded size** + reject-or-drop policy.
- **Reactive Streams** (request-N model).

LinkedIn usage:
- Kafka producers respect broker quotas.
- ParSeq in Java has flow-control primitives.
- Service-to-service rate limits at the gateway and in client libraries.

## 20.8 Failure modes you must reason about

Default to assuming all of these *will* happen in production:

- **Network partition**: brief or sustained.
- **Slow / partial network**: some hops up, others down.
- **Slow downstream**: not down, just slow — worse than down because it ties up resources.
- **Clock skew**: NTP can drift; never trust two machines' clocks to agree to millisecond.
- **Garbage collection pauses**: a JVM can pause for seconds.
- **Disk full**.
- **Memory pressure**: OOM, swap.
- **Single-bit corruption**: rare but real at scale; checksums everywhere.
- **Bug in a deploy**: caught by canary if you have one.
- **Bad configuration**: caught by sanity checks if you have them.
- **Cascading failure**: one service slows down → callers retry → upstream queues fill → everything dies.

Staff-level answers explicitly enumerate which failures are designed-for, which are accepted-as-impact, and which are out-of-scope.

## 20.9 Quorum math

For N replicas, R reads, W writes:

- **Strong consistency**: `R + W > N`. Read and write quorums must overlap.
- **Common choice**: N=3, R=2, W=2.
- **Read-optimized**: R=1, W=N. Always-fresh reads, slow writes.
- **Write-optimized**: R=N, W=1. Slow reads, fast writes.

Staff candidates pull these numbers out of memory.

## 20.10 Time and ordering

### Three notions of time

- **Wall clock**: real-world time. Skewed across machines.
- **Logical clock (Lamport)**: monotonic per-process; "happens-before" can be inferred.
- **Vector clock**: per-process counter array; captures concurrency exactly.
- **Hybrid Logical Clock (HLC)**: wall clock + logical counter; useful for cross-region.

### Ordering guarantees

- **FIFO**: per producer/consumer pair.
- **Causal**: A causes B implies A before B for all observers.
- **Total**: every observer sees the same order.

Kafka guarantees per-partition total order; consumers across partitions see no inter-partition order.

## 20.11 The "thundering herd" pattern

A cache expires; many requests miss simultaneously; all hammer the database.

Mitigations:
- **Request coalescing**: in-flight request shared across simultaneous callers.
- **Probabilistic refresh**: refresh before expiry with some probability based on how close to expiry.
- **Staggered TTLs**: add jitter so keys don't expire at the same instant.
- **Lock-then-fetch**: only one request fetches; others wait or serve stale.

## 20.12 Cache patterns

- **Cache-aside (lazy load)**: app reads cache; miss → read DB → put in cache.
- **Read-through**: cache library reads DB on miss transparently.
- **Write-through**: writes go to cache and DB.
- **Write-back / write-behind**: writes go to cache; flushed to DB async.
- **Refresh-ahead**: cache refreshes before expiry.

Cache invalidation:
- TTL-based: simplest; tolerate staleness.
- Event-based: invalidate on data-change events (Kafka). Stronger consistency.
- Versioned keys: write produces a new key; old keys eventually GC'd.

Pitfall: **cache stampede** (see thundering herd).

## 20.13 Two-phase commit (2PC), Saga, and Outbox

Distributed transactions are hard.

### 2PC

Coordinator asks all participants "can you commit?", all respond yes/no, coordinator decides commit/abort.

- Blocking on coordinator failure.
- Used internally in some systems; rarely in microservices today.

### Saga

A long-running transaction split into local transactions, each with a compensating action. If step 5 fails, run compensations for steps 1–4.

- Choreography (events trigger steps) or orchestration (central orchestrator).
- More resilient than 2PC; eventually consistent.
- Used heavily for cross-service business processes at LinkedIn (job application → ATS sync → analytics → notifications, etc.).

### Outbox pattern

When a service writes to DB and emits an event:
- Single transaction writes to a `outbox` table in the same DB.
- A separate poller reads the outbox table and publishes to Kafka.
- Guarantees the event is published iff the DB write committed.
- LinkedIn pattern: outbox-via-CDC (Brooklin reads the binlog, no explicit outbox table needed).

## 20.14 SLOs, SLIs, SLAs

- **SLI** (indicator): a measured number (e.g., p99 latency).
- **SLO** (objective): a target on that indicator (e.g., p99 latency < 200ms 99% of the time).
- **SLA** (agreement): customer-facing commitment; usually weaker than SLO.

Error budget = `1 - SLO`. Spend it on risky deploys; protect it by deferring deploys.

At LinkedIn, every Tier-0 service has explicit SLOs reviewed in operational reviews.

## 20.15 Distributed tracing

- **Trace ID** propagated through every hop.
- **Spans** record per-hop latency / metadata.
- Used to diagnose latency spikes, find slow downstream.
- OpenTelemetry is the modern standard.

A staff candidate brings up tracing when discussing latency budgets — it's the tool to actually find where the budget went.

## 20.16 Common interview probes

> **"Walk me through the consistency of your design."**
Identify each data piece. Say where it's primary-stored. Say what consistency is needed for each access pattern. Say which writes are sync vs. async. Say what happens during partition.

> **"What happens during a network partition between regions?"**
Local reads/writes continue. Cross-region replication lags. Some operations may need to fail (e.g., global counters); state them. Recovery: log-replay-based catch-up.

> **"What's the worst-case data loss scenario?"**
Single-DC: limited to in-flight writes that hadn't propagated to a sync replica. Multi-DC: limited to lag-window of cross-region replication. Quantify if possible.

> **"What's your retry policy?"**
Idempotent operations: exponential backoff with jitter, capped retries. Non-idempotent: don't auto-retry; require client-supplied idempotency key.

> **"How do you handle a service that returns slow but not error?"**
Timeout aggressively. Trip circuit breaker on persistent slowness. Hedge to a replica. Alert.