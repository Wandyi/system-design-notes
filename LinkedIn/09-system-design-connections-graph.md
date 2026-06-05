# 9 · System Design — Connections Graph

The Economic Graph is LinkedIn's signature. The connections sub-graph alone is enormous: 1B nodes, tens of billions of edges. Many of the product's most valuable features depend on graph queries:

- **People You May Know (PYMK)** — friend-of-friend recommendations.
- **2nd-degree / 3rd-degree** counts for Recruiter search.
- **"How are you connected to this person?"** — find the connection path.
- **Network breadth** — "how many people in your network work at $company?"
- **Visibility checks** — is the viewer 2nd-degree from the post author?

The internal graph store has had multiple generations. The current one (as of recent public talks) is called **LIquid** — a horizontally-scaled, in-memory graph engine.

## 9.1 Requirements

### Functional

- Send / accept / decline / withdraw connection requests.
- Bi-directional acceptance: A invites B; once B accepts, both have the edge.
- Visibility: profile fields & content visibility computed by connection degree.
- Graph queries: PYMK (1st-of-1sts), 2nd/3rd degree counts, shortest-path, "people at $company in your network".
- Capacity: per-member connection cap (currently 30,000 — LinkedIn calls this "1st-degree cap"). Above that, follow-only.

### Non-functional

- **Scale**: ~1B nodes, ~30B+ first-degree edges, ~hundreds of billions of 2nd-degree implied edges.
- **Latency**: 2nd-degree count: < 100ms. PYMK candidate generation: < 200ms.
- **Freshness**: new connection visible within seconds for all queries.
- **Availability**: 99.95%+. PYMK can degrade to a cached fallback; degree-counts cannot.

## 9.2 Capacity & numbers

- Avg connections per member: ~500 (mode ~100, p99 ~10K).
- Some members at 30K cap (legacy power users).
- 2nd-degree footprint per member: ~250K (median) — the famous "friends of friends" explosion.
- 3rd-degree: ~5M+ — usually not exposed as a count.
- Connection write rate: tens of millions of new edges/day at peak.

## 9.3 Storage architectures (evolution)

Worth discussing in the interview — staff candidates demonstrate that they know the *journey* of LinkedIn's graph.

### Generation 1 — relational on Oracle

Initial approach. Doesn't scale; abandoned early.

### Generation 2 — sharded MySQL via Espresso

Edges stored as rows: `(from_id, to_id, created_at, type)`. Sharded by `from_id`. 1st-degree fetch is fast; 2nd-degree requires fanning out to many shards.

### Generation 3 — graph servers + offline batch

Offline Hadoop precomputes:
- Per-member 2nd-degree counts.
- Per-member PYMK candidate list.
Online graph servers store edges in memory (adjacency lists), serve "is X 2nd-degree from Y" queries via in-memory lookup.

### Generation 4 — LIquid (current)

A purpose-built in-memory graph engine:
- **Horizontally sharded** by node ID. Each shard holds its members' adjacency lists.
- **Edge replication**: edges are bi-directional, so each edge is stored on the shards of both endpoints.
- **Query engine**: BFS traversal, parallelized across shards.
- **Real-time updates**: new edges propagate via Kafka.
- **Snapshot + WAL**: durability via periodic snapshot + write-ahead log.
- **In-memory only**: each shard's data fits in RAM (~hundreds of GB per machine).

LIquid is the kind of system staff candidates love discussing — bring it up unprompted if relevant.

## 9.4 The Data Model

```
node {
  member_id: BIGINT (PK)
  // sparse metadata for graph queries (employer, location)
}

edge {
  from_id: BIGINT
  to_id: BIGINT
  type: ENUM (CONNECTION, FOLLOW, BLOCK, etc.)
  created_at: TIMESTAMP
}
```

Connections are *symmetric* `(A, B) ↔ (B, A)` and stored both ways for fast lookup. Follow edges are directed.

## 9.5 The query patterns

### Q1: "Are A and B connected?"

- Hash to shard owning A; check if B in A's neighbor set.
- O(1) lookup if neighbor set is a hash; O(log N) if sorted.
- Latency: < 5ms.

### Q2: "What's A's degree of separation from B?"

- 1st-degree check (above).
- 2nd-degree: are A and B's neighbor sets intersecting? Send A's set to B's shard or B's set to A's shard; intersect.
- 3rd-degree: do BFS from A side and from B side; meet in the middle.
- Latency: 100–500ms for typical sizes.

### Q3: "How many 2nd-degree connections does A have?"

- A's neighbors: known.
- For each neighbor n, sum up |n.neighbors| - overlap.
- Precompute and cache; refresh on write.
- Used for "Network breadth" widgets.

### Q4: "PYMK candidates for A"

- For each 1st-degree neighbor n, collect their neighbors who aren't already 1st-degree of A.
- Rank by `score = f(num_mutual, employer_match, school_match, prior_interaction)`.
- Trim to top ~200.
- Mostly precomputed offline (PYMK pipeline in Hadoop / Spark) with real-time deltas via Samza.

### Q5: "Is this post visible to viewer V given visibility setting CONNECTIONS-ONLY of author A?"

- Equivalent to "Are A and V 1st-degree?"
- Cached aggressively (it doesn't change often).

## 9.6 Write path

Member A sends invitation to B → B accepts:
1. Invitation Service writes a pending invitation row in Espresso.
2. Notifications sends B a "connection request" notification.
3. B accepts → Invitation Service deletes the pending row; writes a Connection edge.
4. Connection edge → Espresso → Brooklin CDC → Kafka event.
5. Kafka consumers:
   - **LIquid** updates A's and B's adjacency lists in memory.
   - **Search indexers** update profile reachability fields.
   - **PYMK pipeline** schedules a recompute for both.
   - **Feed FollowFeed** subscribes A to B's posts (and vice versa).
   - **Analytics** logs the event.

Latency: edge visible across services in < 5 seconds typical.

## 9.7 PYMK (People You May Know) — the canonical recommendation system

PYMK is LinkedIn's most iconic ML system. It drove the original growth engine. A senior/staff interview will explore it.

### Signals

- **Mutual connections** (strongest signal).
- **Shared employer / school**.
- **Imported address book** matches.
- **Co-attendance** of events / groups.
- **Profile-view overlaps** ("people who viewed A also viewed B").
- **Common groups, hashtags, content interactions**.

### Generation pipeline

Two-stage:

1. **Candidate generation (offline batch)**:
   - For each member A, generate ~10K candidates via signal aggregation.
   - Spark job; runs daily.
   - Output stored in Venice (key: member_id, value: candidate list with feature snapshots).
2. **Online ranking**:
   - At surface time, load A's candidate list.
   - Rerank with up-to-date features (recent connections, recent profile views).
   - Filter out already-connected, blocked, dismissed.
   - Surface top 20.

### Real-time delta

Samza pipeline:
- On every new connection, queue a partial recompute for both endpoints' PYMK candidate lists.
- Updates the candidate set incrementally.

### Avoiding stale/bad recommendations

- "I don't know this person" dismiss feedback feeds into negative training signal.
- Stable companion-recommendation patterns (don't show the same person every day).
- Privacy: respect blocks, mutes, do-not-recommend-me flags.

## 9.8 Multi-region

- LIquid runs in each major region.
- Edge updates flow via Kafka mirror.
- Eventual consistency: new edges may appear in another region with seconds-level delay.
- A connection request acceptance is read-your-writes in the same region.
- Cross-region invitation flows: rare path but supported; latency may be sub-second for the accept.

## 9.9 Failure modes

- **LIquid shard OOM** — load shedding; fall back to cached 2nd-degree counts (slightly stale).
- **Edge propagation lag** — visibility checks may temporarily return "not connected" for a brand-new connection; UI handles gracefully ("connection accepted; might take a moment to update").
- **Hot member (e.g., a celebrity with maxed-out connections)** — their adjacency list is huge; partition further.
- **Adversarial scraping** — Trust ML detects bulk profile viewing or bulk invite spam; throttles.

## 9.10 Operational concerns

- **Edge growth** is steady; capacity plans are predictable.
- **Snapshot / restore** of LIquid shards must be fast — required for failover and DR.
- **Versioning** of the graph: occasionally needed for "what did the graph look like at time T?" — supported via snapshot history.
- **Compliance**: GDPR "delete my account" must propagate to graph — tombstone the node; tombstone implies edges are effectively removed; defer permanent deletion to nightly compaction.

## 9.11 Common follow-ups

> **"How would you implement 'find shortest path A → B (up to length 5)'?"**
Bidirectional BFS in LIquid: BFS from A and B in parallel, expand frontier on whichever has fewer expansions, stop when they meet. Cap depth. Streaming results to UI as paths are found.

> **"How would you support 100M PYMK lookups/min?"**
PYMK pre-computed in Venice → simple KV lookup at p95 < 5ms. Online reranking adds ~30ms.

> **"How do you handle a member who hits the 30K connection cap?"**
Soft-block new invites; convert excess to follows; UX nudges to remove inactive connections. Caps prevent power users from skewing PYMK signals for everyone.

> **"How would you offer 'mutual connections' for a Recruiter searching candidates?"**
The Recruiter query already specifies "candidates in my 2nd-degree" as a filter. The graph engine intersects each candidate with the Recruiter's 1st-degree set; ranks by mutual count. Heavy precompute for Recruiter searches.

> **"How do you migrate from a sharded MySQL graph to LIquid without downtime?"**
Dual-write to both; build LIquid offline from MySQL snapshot + WAL; dark-launch reads on LIquid; compare results to MySQL; ramp; eventually deprecate MySQL. Months-long migration.