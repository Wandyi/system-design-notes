# 24 · Quick Reference Cheat Sheets

Last-48-hours flashcards. Print these or read on phone. Numbers, names, formulas. The kind of thing you want loaded into RAM walking into the loop.

## 24.1 Capacity and back-of-envelope numbers

### Time

| Unit | Approx duration |
|---|---|
| 1 second | 1s — a network round-trip cross-DC, a couple of disk reads |
| 1 millisecond (ms) | Memory access × 10⁵; disk seek ÷ 10 |
| 1 microsecond (μs) | Cache miss; 1000 ns |
| 1 nanosecond (ns) | L1 cache access |

### Latency numbers every engineer should know

| Operation | Latency |
|---|---|
| L1 cache reference | ~0.5 ns |
| Branch mispredict | ~5 ns |
| L2 cache reference | ~7 ns |
| Mutex lock/unlock | ~25 ns |
| Main memory reference | ~100 ns |
| Compress 1K bytes (Zippy) | ~3 μs |
| Send 2K bytes over 1 Gbps | ~20 μs |
| SSD random read | ~150 μs |
| Round trip same DC | ~500 μs (0.5 ms) |
| Disk seek | ~10 ms |
| Read 1 MB sequentially from disk | ~30 ms |
| Send packet CA→Netherlands→CA | ~150 ms |

(Memorize these — Jeff Dean's latency cheat sheet.)

### Throughput / storage

| Quantity | Approx |
|---|---|
| 1 KB | small text doc |
| 1 MB | photo, small file |
| 1 GB | small video, RAM-sized index |
| 1 TB | medium dataset |
| 1 PB | LinkedIn-scale data lake |
| 1 EB | global-scale storage |

Common conversions to memorize:
- 1 KB = 10³ B; 1 MB = 10⁶ B; 1 GB = 10⁹ B; 1 TB = 10¹² B.
- (Or in binary: KiB = 2¹⁰, MiB = 2²⁰, etc.)
- Seconds in a day: 86,400 ≈ 10⁵.
- Seconds in a month: ~2.5 × 10⁶.
- Seconds in a year: ~3 × 10⁷.

## 24.2 LinkedIn scale numbers (order-of-magnitude)

| Metric | Approx |
|---|---|
| Members | 1B+ |
| MAU | ~310M |
| DAU | ~100M (estimated) |
| Feed views/day | tens of billions |
| Messages sent/day | hundreds of millions |
| Connection invites/day | tens of millions |
| Kafka messages/day | 7T+ |
| Tracking events/day | 100s of B |
| Job postings active | tens of millions |
| Applications/day | ~5M |
| Companies represented | 67M+ |

## 24.3 Storage formulas

- **Total storage** = `events/day × bytes/event × retention_days × replication_factor`.
- **Throughput** = `events/day ÷ 86400` (for steady state); ×2-3 for peak.
- **Disk for Pinot/columnar** ≈ raw × 0.1–0.3 (compression).
- **Disk for OLTP row-store** ≈ raw × 1.5–3 (overhead, indexes).
- **RAM for in-memory** = total × replication; expect 100–500 GB per server today.

## 24.4 QPS estimates by tier

Order-of-magnitude steady-state QPS for LinkedIn-scale systems (peak ~3x these):

| System | QPS |
|---|---|
| Feed loads (page) | ~50K |
| Feed cards rendered | ~1.5M |
| Profile views | ~50K |
| Search queries (across all surfaces) | ~30K |
| Messages sent | ~3K (writes); ~50K reads (inbox loads) |
| Notifications delivered | ~10K |
| Ad auctions | ~1M (impressions/sec across LinkedIn) |
| Kafka messages produced | ~80M+ |

## 24.5 Acronyms / names cheat sheet

| Name | What it is |
|---|---|
| Espresso | Document store; primary OLTP |
| Voldemort | KV store; older |
| Venice | Derived-data serving |
| Ambry | Blob store |
| Pinot | Real-time OLAP |
| Samza | Stream processing |
| Brooklin | CDC + general streaming bridge |
| Databus | Older CDC (Oracle); replaced by Brooklin |
| Galene | Search platform (Lucene-based) |
| Bobcat | Structured / Boolean query layer |
| Helix | Cluster manager |
| Burrow | Kafka consumer-lag monitor |
| Cruise Control | Kafka cluster auto-balancer |
| FollowFeed | Pull-feed back-end |
| Concourse | Feed orchestrator |
| ATC | Air Traffic Controller (notifications) |
| RTG / RTFE | Real-time gateway / frontend (WS) |
| LIquid | In-memory graph engine |
| LIX | A/B test framework |
| Rest.li | RPC framework |
| D2 | Service discovery + LB |
| Iris | Alerting / escalation |
| InGraphs | Metrics platform |
| Pro-ML | ML platform |
| Feathr | Feature store |
| PYMK | People You May Know |
| JYMBII | Jobs You May Be Interested In |
| WVMP | Who Viewed My Profile |
| LMS | LinkedIn Marketing Solutions (ads) |
| GAI | LinkedIn's GenAI platform name |
| OpenHouse | LinkedIn-built unified table service |
| TonY | TensorFlow on YARN |

## 24.6 Distributed-systems quick recall

### Quorum

For N replicas, strong consistency requires `R + W > N`. Typical: N=3, R=2, W=2.

### Consensus

Raft has 5 servers, leader election with terms, log replication, majority quorum. Read on paper if rusty.

### CAP

In partition, choose C or A. Most systems are mixed by configuration.

### Consistency models (strict → loose)

Linearizable → Sequential → Causal → Read-your-writes → Monotonic reads → Eventual.

### Common failure causes

Network partition, slow downstream, GC pause, clock skew, deploy bug, bad config, cascading failure, disk full.

### Cache invalidation patterns

TTL; event-based (Kafka); versioned keys; lazy refresh-ahead.

### Retry policy

Idempotent: exponential backoff with jitter, capped retries, retry budget. Non-idempotent: don't retry; client supplies idempotency key.

## 24.7 The system-design interview structure

5-minute requirements → 5-minute estimation → 10-minute high-level → 25-minute deep dives → 10-minute failure / ops → 5-minute buffer.

**Phrases to use**:
- "Let's clarify functional requirements first."
- "What's the scale we should target?"
- "I'd want to make sure I cover [X, Y, Z] — does that look right?"
- "Let me back-of-envelope this..."
- "The trade-off here is..."
- "An alternative would be... and I'd choose this because..."
- "How do we know it's working? Here are the metrics I'd track..."
- "On failure: ..."
- "From an operational perspective..."

## 24.8 Behavioral round template

For every question, structure as STAR:
- **S**: 30s context.
- **T**: 30s your role.
- **A**: 2-3 min actions.
- **R**: 30s outcome with numbers.

End with "what I'd do differently / what I learned."

Stories you must have ready:
1. Most impactful project.
2. Failed project / project you'd redo.
3. Disagreed with senior person.
4. Mentored someone.
5. Cross-team initiative without authority.
6. Production incident you led.
7. Tech debt vs. feature pressure.
8. Said no to a stakeholder.
9. Changed your mind based on data.
10. Long-term vision you championed.

## 24.9 The "design X" template (memorize)

Open: "Let me clarify scope. Is it for [member surface / business surface / both]? What's the scale we're targeting? Any specific bottleneck or constraint I should weight?"

Estimation: "Let me size this. Members × per-member-rate = QPS. Storage = QPS × size × retention × replication."

High-level: "Here's the overall flow [draw]. We have these services: [name them]. We use these stores for these reasons: [name them]. The data flow is [trace from request to response]."

Deep dive (pick the most interesting component):
- API contract.
- Data model.
- Sharding strategy.
- Read path.
- Write path.
- Failure modes.

Multi-region: "I'll explicitly state which data is per-region, which is globally consistent, and how cross-region replication works."

Failure modes: "Here are the 3 most likely failures. Here's how each degrades the system. Here's how we mitigate."

Operations: "Monitoring, alerting, capacity, on-call, A/B testing all wired in."

Trade-offs: "If we had to optimize for X over Y, here's what changes."

## 24.10 Tracing your design back to business value

Always close a design with: "The business impact of this design is — it serves [X members] with [Y outcome] at [Z cost]. The biggest risks are A and B; the mitigation strategy is C and D."

Staff calibration anchor.

## 24.11 30-second LinkedIn pitches (memorize)

### Kafka
"Distributed log; partitioned, replicated; producers append, consumers pull with their own offset; LinkedIn-born; powers most data movement at LinkedIn. Exactly-once via idempotent producer + transactions. KRaft replaces ZooKeeper. Tiered storage for cheap retention."

### Pinot
"Real-time OLAP. Columnar segments with inverted, range, sorted, and star-tree indexes. Real-time table consumes Kafka; offline table from Hadoop. Sub-second analytics on billions of rows. LinkedIn-born, Apache project."

### Espresso
"Distributed document store. Schema-on-Avro. Each partition is a MySQL. Helix-managed. Brooklin-CDC for cross-DC and downstream. Used for member profile, messages, jobs, anything OLTP."

### Venice
"Derived-data serving. Batch + stream + hybrid pushes. Replicas independently consume Kafka. RocksDB underneath. Replaces Voldemort RO/RW. Used for ML feature store + JYMBII + feed."

### FollowFeed / Concourse
"Hybrid push-pull feed. Pull-default via per-author timelines (FollowFeed); push for hot follower/cold author pairs. Concourse orchestrates: candidate gen → ranking (two-stage) → hydration. Tracking events at every step."

### ATC (notifications)
"Multi-channel router. Source events on Kafka → decision engine (member prefs, rate limits, ML send-probability) → channel adapter (push/email/in-app/SMS). Dedup keys for idempotency. Frequency capping. ML for long-term engagement, not short-term clicks."

### LIquid
"In-memory graph engine. Sharded by member_id. Bi-directional edge replication. Helix-managed. Real-time updates via Kafka. Powers degree-counts, PYMK at query time, visibility checks."

### LIX
"A/B testing framework. Deterministic bucketing via hash(member_id, experiment_id). Treatment evaluated client-side or server-side. Analyzer joins event stream with treatment cohort for lift. Statistical rigour."

### Rest.li / D2
"Rest.li: schema-first RPC, PDL types, code-gen clients, richer than plain REST (Finders, Actions). D2: client-side service discovery via ZooKeeper; degrader load balancing."

### Samza
"Stream processor on Kafka. Input/output/state-changelog all in Kafka. Stateful tasks with local RocksDB; restored from changelog on restart. LinkedIn-born; Apache project. Heavily used; Flink growing for newer pipelines."

### Brooklin
"General-purpose streaming data pipe. Connector model — source (Espresso CDC, Kafka, MySQL) → destination (Kafka, Espresso, etc.). Helix-managed. Replaced Databus and a lot of MirrorMaker."

### Ambry
"Immutable blob store. Append-only logs per partition; in-memory hash index per partition; no global metadata. Multi-DC replication. LinkedIn-built; some workloads now moving to Azure Blob."

### Galene
"Lucene-based search platform. Sharded inverted indexes per index (people, jobs, content, etc.). Real-time + offline indexing. Two-stage ranking. Vector search added for AI surfaces."

### Helix
"Generic cluster manager. State machine model. Used by Espresso, Pinot, Venice, Brooklin. Backed by ZooKeeper. Handles partition assignment, leader election, rebalancing."

## 24.12 Last 90 seconds before each round

- Breathe.
- Remind yourself: state requirements, estimate scale, name the alternatives.
- "I don't know everything about LinkedIn-internal systems. I'll use first principles where I'm not sure."
- Smile (audible in voice on phone interviews; visible on video).

Now go ship.