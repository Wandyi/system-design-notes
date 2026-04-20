Globally Distributed Cache — Full System Design

  ---
1. Requirements & Constraints

Functional
- Read and write key-value data from any region
- TTL-based expiration
- Support for partial invalidation and bulk operations

Non-functional targets
Read latency:    p50 < 1ms (local), p99 < 10ms (cross-region)
Write latency:   p99 < 50ms
Throughput:      10M+ ops/sec globally
Availability:    99.99% (< 53 min/year downtime)
Consistency:     Tunable (eventual by default, strong on demand)
Data size:       Up to a few KB per value (not a blob store)

  ---
2. High-Level Architecture

┌─────────────────────────────────────────────────────────────┐
│                        CLIENTS                              │
│         (services, APIs, backends in each region)           │
└────────────┬───────────────────────┬────────────────────────┘
             │                       │
┌────────▼────────┐    ┌─────────▼───────┐
│   US-EAST       │    │    EU-WEST      │   ... more regions
│  ┌───────────┐  │    │  ┌───────────┐  │
│  │  L1 Cache │  │    │  │  L1 Cache │  │   ← in-process, per service instance
│  └─────┬─────┘  │    │  └─────┬─────┘  │
│        │        │    │        │        │
│  ┌─────▼─────┐  │    │  ┌─────▼─────┐  │
│  │  L2 Cache │  │    │  │  L2 Cache │  │   ← regional Redis/Dragonfly cluster
│  │  Cluster  │  │    │  │  Cluster  │  │
│  └─────┬─────┘  │    │  └─────┬─────┘  │
└────────┼────────┘    └────────┼────────┘
         │                      │
┌────────▼──────────────────────▼────────┐
│         GLOBAL CONTROL PLANE           │
│  ┌─────────────────────────────────┐   │
│  │  Replication  │  Invalidation   │   │
│  │  Coordinator  │  Bus (Kafka)    │   │
│  └───────────────┴─────────────────┘   │
│  ┌─────────────────────────────────┐   │
│  │  Metadata Store (topology,      │   │
│  │  routing, config) — etcd/Consul │   │
│  └─────────────────────────────────┘   │
└────────────────────────────────────────┘
                 │
        ┌────────▼────────┐
        │   ORIGIN / DB   │   ← source of truth, cache-aside fallback
        └─────────────────┘

  ---
**3. Three-Tier Cache Hierarchy**

**L1 — In-Process Cache (per service instance)**

- Embedded in the application (e.g., Caffeine in JVM, lru-cache in Node, ristretto in Go)
- Nanosecond access, zero network hops
- Tiny capacity: 100MB–1GB per instance
- Short TTLs: 1–30 seconds (stale tolerance is low)
- Populated on L2 hit — not on origin reads

Read path:
L1 hit  → return (nanoseconds)
L1 miss → check L2
L2 hit  → populate L1, return (< 1ms)
L2 miss → read origin, populate L2 + L1, return

**L2 — Regional Cluster (per region, multi-AZ)**

- Redis Cluster or Dragonfly (higher throughput, lower memory overhead)
- Partitioned via consistent hashing across nodes
- Replication factor 3 (1 primary + 2 replicas per shard) across AZs
- Capacity: TBs of data per region
- TTLs: minutes to hours

┌─────────────────────────────────────────┐
│           US-EAST Regional Cluster      │
│                                         │
│  AZ-1          AZ-2          AZ-3       │
│ ┌────────┐   ┌────────┐   ┌────────┐    │
│ │Shard 0 │   │Shard 1 │   │Shard 2 │    │
│ │Primary │   │Primary │   │Primary │    │
│ └────┬───┘   └────┬───┘   └────┬───┘    │
│      │ replica    │ replica    │ replica│
│  AZ-2,3        AZ-1,3       AZ-1,2      │
└─────────────────────────────────────────┘

**L3 — Global Coordination**

- Not a cache tier — handles cross-region replication metadata, invalidation routing, and topology
- Uses Kafka for async replication events between regions
- etcd or Consul for cluster membership, routing tables, and configuration

  ---
4. Partitioning Strategy

Consistent hashing with virtual nodes:

Physical nodes: N = 100 per region
Virtual nodes:  V = 150 per physical node
Ring positions: 15,000 slots

Key → hash(key) % ring → find owner node

Why virtual nodes?
- Even distribution even with heterogeneous node sizes
- When a node fails, its slots spread across all remaining nodes (not just neighbors)
- Adding capacity = add nodes with virtual slots, minimal reshuffling

Replication within region:
For key K owned by node A:
Replica 1 → next distinct physical node clockwise (different AZ)
Replica 2 → next distinct physical node after that (different AZ)

  ---
5. Replication Model — Cross-Region

Strategy: Active-Active with Async Replication

Write to Region A:
1. Write to L2 cluster in Region A (sync, acknowledged)
2. Publish WriteEvent to Kafka replication topic (async)
3. Region B, C consumers read from Kafka, apply writes

Consistency window:
Intra-region:  strong (sync replication across AZs)
Cross-region:  eventual (async via Kafka, ~50-200ms lag)

Conflict resolution — Last Write Wins with HLC timestamps:

Every write carries a Hybrid Logical Clock (HLC) timestamp:
HLC = max(physical_time, last_known_HLC) + logical_counter

On conflict: higher HLC wins

HLCs are better than wall clocks because they respect causality — a write that happened-after another always has a higher HLC, even across nodes with clock skew.

Tunable Consistency (per-key or per-request):

CACHE_GET key CONSISTENCY=eventual   → read local region (default, fastest)
CACHE_GET key CONSISTENCY=bounded    → reject if replica lag > Xms
CACHE_GET key CONSISTENCY=strong     → coordinate with quorum of regions
(2 of 3 regions must agree)

Strong consistency uses a lightweight quorum read — rarely needed (pricing, inventory).

  ---
6. Write Path Detail

Client write: SET user:123:profile {data} TTL=300s

1. Client SDK hashes key → identifies home region shard
2. Write sent to shard primary in local region
3. Primary writes to its AZ replicas (sync, quorum ACK = 2/3)
4. Primary ACKs client ← client sees success here
5. Primary publishes to Kafka topic: cache-replication-{region}
6. Remote region consumers apply write to their L2 cluster
7. L1 caches in remote regions are NOT proactively updated
   → they expire naturally (short TTL) or are invalidated (see §8)

  ---
7. Read Path Detail

Client read: GET user:123:profile

1. Check L1 (in-process) → HIT: return immediately
2. Check L2 (regional cluster) → HIT: populate L1, return
3. L2 MISS:
   a. Acquire distributed lock on key (prevents stampede — see §10)
   b. Re-check L2 (another request may have populated it while waiting)
   c. Read from origin DB
   d. Populate L2 with TTL
   e. Populate L1
   f. Release lock
   g. Return value

Read repair: On L2 hit, if the value's HLC timestamp is older than the replication lag threshold, asynchronously re-fetch from origin 
             and refresh L2 in the background.

  ---
8. Invalidation

Two mechanisms — use both:

TTL-Based (passive)

Every key has a TTL. Stale data auto-expires. No coordination needed.

Event-Driven Invalidation (active)

Origin DB change (e.g., user profile updated)
→ publish to Kafka topic: cache-invalidations
→ all regional cache clusters subscribe
→ each region deletes or marks key as stale
→ L1 caches invalidated via pub/sub channel (Redis PUBLISH)

Invalidation fan-out pattern:
Invalidation message: {key: "user:123:profile", version: 4521, regions: ["*"]}

Regional invalidator:
1. DELETE key from L2
2. Publish "invalidate:user:123:profile" on Redis pub/sub
3. All L1 instances subscribed → evict from local map

Tag-based invalidation:
Group related keys under a tag:
SET user:123:profile   TAG=user:123
SET user:123:settings  TAG=user:123
SET user:123:feed      TAG=user:123

INVALIDATE TAG user:123 → atomically deletes all 3 keys
Implemented via a tag→keys index in Redis (using Redis Sets).

  ---
9. Hot Key Problem

A single key (e.g., a celebrity's profile, a viral product) can overwhelm a single shard.

Solution: Local Replication of Hot Keys

Hot key detector (sliding window counter per key):
if request_rate(key) > threshold (e.g., 10k req/s):
mark key as HOT
replicate to N local shadow shards (e.g., N=20)

Read path for HOT key:
hash(key + random(0, N)) → route to one of N shadow shards
→ load distributed across 20 nodes instead of 1

Write path for HOT key:
write to all N shadow shards (fan-out on write)
→ acceptable because writes are rare vs reads for hot keys

L1 as natural hot key mitigation:
For truly viral keys, L1 absorbs most traffic. If a key is read 1000x/sec per instance and L1 TTL is 5s, L2 only sees ~0.2 req/s per instance for that key.

  ---
10. Cache Stampede (Thundering Herd) Prevention

When a popular key expires, thousands of requests simultaneously miss and all hit the origin.

Solution 1: Probabilistic Early Expiration (PER)
# Don't wait for TTL to hit zero — start refreshing early with probability
def should_refresh(key):
remaining_ttl = get_ttl(key)
beta = 1.0  # tunable
delta = time_to_recompute(key)  # estimated recompute time
return (current_time() - delta * beta * log(random())) > (expiry_time - remaining_ttl)
Keys start getting refreshed before they expire, probabilistically. No stampede.

Solution 2: Request Coalescing (mutex on miss)
Thread 1: L2 MISS → acquire lock(key) → fetch origin → populate → release
Thread 2: L2 MISS → lock exists → WAIT (or return stale) → lock released → L2 HIT
Thread 3: same as Thread 2

Only 1 origin request instead of N.

Solution 3: Stale-While-Revalidate
Key has two TTLs:
soft_ttl: when to start background refresh (key still served as-is)
hard_ttl: when to stop serving stale data entirely

On soft_ttl expiry:
serve stale value immediately
trigger async background refresh
client never waits

  ---
11. Availability Design

Region Failure

If Region A goes down:
DNS/load balancer failover → Route 53 health checks + latency routing
Clients rerouted to Region B or C
L2 miss rate spikes temporarily → origin absorbs load
Replication lag: Region B may serve slightly stale data (bounded by last replication)

Node Failure within Region

Shard primary fails:
Replica promotion: automatic (Redis Sentinel / Cluster)
Promotion time: < 30 seconds (typically < 10s)
During promotion: reads served by replicas (may be slightly stale)
writes queued in client SDK (with timeout)

Network Partition (split-brain)

Region A and B can't communicate:
Both continue serving reads from local L2 (AP — availability over consistency)
Writes accepted locally, queued for replication when partition heals
On heal: conflict resolution via HLC timestamps (Last Write Wins)
Ops team alerted via monitoring; manual review if write conflict rate is high

Availability math:
Single node MTBF:           ~1 year
Regional cluster (100 nodes, RF=3):
P(full shard loss) ≈ negligible with RF=3 across AZs
Multi-region (3 regions):
P(all 3 down simultaneously) ≈ negligible for independent failure modes
Target: 99.99% = 52 min downtime/year — achievable with this design

  ---
12. Handling TTL & Expiration at Scale

Lazy expiration (Redis default): key checked on access, deleted if expired.

Active expiration: background thread samples keys and deletes expired ones.

Problem at scale: if 10M keys expire at the same second (e.g., all set with TTL=3600 at startup), you get an expiration storm → origin flood.

Solution: TTL Jitter
// Instead of:
SET key value EX 3600

// Do:
jitter = random(-300, 300)  // ±5 minutes
SET key value EX (3600 + jitter)
Spreads expiration over a 10-minute window. Origin load stays flat.

  ---
13. Security

Encryption in transit:    TLS 1.3 between all components
Encryption at rest:       AES-256 on disk (for persistent cache data)
Authentication:           mTLS between services and cache cluster
Authorization:            Key-prefix ACLs (service A can only read/write "userservice:*")
Network isolation:        VPC peering, no public endpoints
Audit logging:            All writes logged with caller identity and timestamp

  ---
14. Observability

Key metrics to track:

Cache hit rate:       per tier (L1, L2), per key prefix
Eviction rate:        if high → cache too small or TTLs too short
Replication lag:      cross-region HLC delta (alert if > 500ms)
Hot key rate:         keys exceeding threshold (trigger shadow shard creation)
Origin fallback rate: spikes indicate L2 problems
p99 read latency:     per region, per tier
Connection pool util: % of pool used (alert at 80%)

Alerting thresholds (example):
L2 hit rate < 80%        → page on-call (cache undersized or TTLs too short)
Replication lag > 1s     → warn (cross-region consistency degraded)
Replication lag > 10s    → page (replication pipeline broken)
Origin fallback > 5%     → page (L2 cluster degraded)
Node down > 30s          → page (replica promotion stalled)

  ---
15. Technology Choices

┌──────────────────────┬──────────────────────────────────┬─────────────────────────────────────────────────────┐
│      Component       │           Recommended            │                         Why                         │
├──────────────────────┼──────────────────────────────────┼─────────────────────────────────────────────────────┤
│ L1 cache             │ Caffeine (JVM), Ristretto (Go)   │ High hit rate with adaptive eviction                │
├──────────────────────┼──────────────────────────────────┼─────────────────────────────────────────────────────┤
│ L2 cache             │ Dragonfly or Redis 7 Cluster     │ Dragonfly: 25x throughput vs Redis on same hardware │
├──────────────────────┼──────────────────────────────────┼─────────────────────────────────────────────────────┤
│ Replication bus      │ Kafka                            │ Durable, replayable, handles regional lag           │
├──────────────────────┼──────────────────────────────────┼─────────────────────────────────────────────────────┤
│ Coordination         │ etcd                             │ Strong consistency for topology metadata            │
├──────────────────────┼──────────────────────────────────┼─────────────────────────────────────────────────────┤
│ Invalidation pub/sub │ Redis PUBLISH or Kafka           │ Low-latency broadcast                               │
├──────────────────────┼──────────────────────────────────┼─────────────────────────────────────────────────────┤
│ Load balancing       │ AWS Global Accelerator / Anycast │ Route to nearest healthy region                     │
├──────────────────────┼──────────────────────────────────┼─────────────────────────────────────────────────────┤
│ Monitoring           │ Prometheus + Grafana             │ Standard, rich Redis exporters exist                │
└──────────────────────┴──────────────────────────────────┴─────────────────────────────────────────────────────┘

  ---
16. Back-of-Envelope Capacity

Target: 10M reads/sec globally, 3 regions

Per region: ~3.3M reads/sec
L1 absorbs ~80% → L2 sees ~660K reads/sec per region
L2 node throughput (Dragonfly): ~1M ops/sec per node
Nodes needed per region: 1-2 nodes for throughput (size for data, not throughput)

Data sizing: 1TB per region, 64GB RAM per node → 16 nodes per region
With RF=3 → 48 physical nodes per region

Cross-region replication: 660K writes/sec × 500 bytes avg = ~330 MB/s Kafka throughput
Kafka: 3-broker cluster per region handles this easily

  ---
Summary — Design Decisions & Tradeoffs

┌─────────────────────┬──────────────────────────┬───────────────────────────────────────────────────────┐
│      Decision       │          Choice          │                   Tradeoff Accepted                   │
├─────────────────────┼──────────────────────────┼───────────────────────────────────────────────────────┤
│ Consistency model   │ Eventual (default)       │ Stale reads possible across regions                   │
├─────────────────────┼──────────────────────────┼───────────────────────────────────────────────────────┤
│ Replication         │ Active-active async      │ Write conflicts possible (resolved via HLC)           │
├─────────────────────┼──────────────────────────┼───────────────────────────────────────────────────────┤
│ Partition tolerance │ Serve stale on partition │ Prefer availability over consistency (AP)             │
├─────────────────────┼──────────────────────────┼───────────────────────────────────────────────────────┤
│ Hot key mitigation  │ Shadow shards            │ Write fan-out cost for hot keys                       │
├─────────────────────┼──────────────────────────┼───────────────────────────────────────────────────────┤
│ Stampede prevention │ PER + coalescing         │ Slight complexity in client SDK                       │
├─────────────────────┼──────────────────────────┼───────────────────────────────────────────────────────┤
│ TTL strategy        │ Jittered TTL             │ Values expire slightly earlier or later than intended │
├─────────────────────┼──────────────────────────┼───────────────────────────────────────────────────────┤
│ L1 tier             │ Short TTL, small         │ Higher L2 traffic than longer L1 TTL                  │
└─────────────────────┴──────────────────────────┴───────────────────────────────────────────────────────┘

The core principle: push reads as close to the client as possible, accept bounded staleness, and design every failure mode to degrade gracefully rather than fail hard.

✻ Baked for 1m 39s
