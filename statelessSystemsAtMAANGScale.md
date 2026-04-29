# Scaling Stateless Systems at MAANG Scale — Staff-Level Insights

> A deep, opinionated reference for designing, evolving, and operating *stateless* services at **1B+ users/day** or **>10M RPS** sustained, with sub-50ms p99 SLOs, multi-region presence, and hyper-fast deploy cadences. Stateless tiers are the *muscle* of the platform — API gateways, edge proxies, business-logic services, fan-out coordinators, transformation/auth/validation layers. Their textbook reputation as "easy to scale by adding instances" is wrong at this scale: at 10M RPS the bottlenecks shift to network, runtime, GC, scheduling, and human-process boundaries. This doc captures the staff-level mental models that survive scale.

> Companion to `statefulSystemsAtMAANGScale.md`. Read both. The two tiers are inverted twins: stateless owes its scale-out to never owning state; stateful owes its scale-in to always owning it. The boundary between them is the most important interface a staff engineer designs.

---

## 0. How to Read This

Each section answers three things:

1. **What changes qualitatively** at this scale (where naive engineering quietly fails).
2. **The dimensions of the choice** and the failure mode each one introduces.
3. **The second- and third-order effects** that distinguish a senior architect from a staff engineer.

Stateless ≠ trivial. The lazy version of "stateless service" was solved 15 years ago: throw containers behind a load balancer. The hard version — running 100,000 stateless pods at 10M RPS, with 100ms p99 SLOs, mTLS, multi-region failover, ~minute deploy cadence, no thundering-herd retries, and predictable cost — is one of the highest-skill domains in MAANG infrastructure.

---

## 1. What "Stateless at MAANG Scale" Actually Means

### 1.1 Translating users to load

```
1B users/day   → ~50K–100K RPS to "the platform" at peak
              ↓
              fan-out (~10–30× per user request through internal services)
              ↓
       10M RPS internal at the stateless service mesh
              ↓
       per-service fleets of 1K–10K pods
              ↓
       per-pod load: 1K–10K RPS each (depends on language, work per request)
```

**Real numbers (rough, MAANG-typical):**
- Google search frontends: ~100K RPS edge, ~3M RPS internal fan-out across the search, ranking, ads, and auxiliary tiers
- Facebook News Feed: ~1M reads/sec at user-facing edge, 100M+ internal RPC/sec across Tao, ranking, feature stores, and ML inference tiers
- Netflix: ~2M RPS internal at peak, ~100K instances, ~2,500 microservices
- AWS API Gateway: handles 10s of millions of RPS aggregate across customers; the gateway tier itself is stateless
- Cloudflare edge: ~57M HTTP requests/sec across their edge in 2024 — a planetary stateless tier

### 1.2 The qualitative shift

```
At 1K RPS: any framework works.
At 100K RPS: language and framework choice starts to matter.
At 1M RPS:  GC, kernel networking, and TCP stack dominate.
At 10M RPS: you are an infrastructure team, not a feature team.
At 100M RPS internal: the infrastructure has its own infrastructure.
```

The qualitative changes that hit hardest:

| Below ~100K RPS | Above ~1M RPS |
|---|---|
| Add instances → linear scaling | Adding instances increases coherence cost; fanout-amplification hurts |
| Kubernetes default scheduling fine | Need topology-aware scheduling, node affinity, NUMA awareness |
| Default GC settings fine | GC pause = latency tail; runtime tuning is full-time concern |
| HTTP/1.1 fine | HTTP/2 mandatory; HTTP/3 increasingly so; connection multiplexing dominates |
| Default DNS fine | Caching, refresh, TTL drift become operational issues |
| Restarts cost nothing | Cold starts cost p99; warmup planning matters |
| Logging via stdout | Structured logs, sampled traces, cardinality budgets |
| Single LB | LB hierarchy: edge (anycast), regional, zonal, cluster, sidecar |
| Manual rollouts | Multi-stage progressive delivery with auto-rollback |
| Rate limit at app | Multi-tier rate limiting (edge, gateway, service, dependency) |

### 1.3 Latency budget

A 200ms user-facing SLO is *not* 200ms for one service:

```
DNS resolution               5–20 ms (mostly cached)
TCP + TLS handshake          20–80 ms (or ~0 with connection reuse)
Edge routing (CDN/anycast)   2–10 ms
Load balancer                1–5 ms
API gateway (auth + WAF)     2–10 ms
Service mesh sidecar (in)    1–3 ms
Application work             10–80 ms
Service mesh sidecar (out)   1–3 ms × N (fan-out calls)
Fan-out tail (10–30 calls)   p99 dominant
Sidecar (back)               1–3 ms × N
DB / cache calls             1–20 ms each
Response serialization       1–5 ms
TLS encrypt                  1–3 ms
Network back to user         20–100 ms
                            ─────────────────
Total p99 budget             ~200 ms (must add up!)
```

A staff engineer holds this kind of math live in design reviews. **The slowest single layer "spends" 30–40% of the budget**. Allocate from the top.

---

## 2. The Illusion of Stateless — Hidden State Everywhere

The core staff insight: **most "stateless" services are full of hidden state**. Recognizing it is the first step to scaling correctly.

### 2.1 Things that look stateless but aren't

| Apparent statelessness | Hidden state |
|---|---|
| HTTP request handler | Connection pools to downstream, DNS cache, TLS session cache, JIT compiled code, hot CPU caches |
| Function-as-a-Service | Container warmth, V8 isolate state, DNS cache in runtime, lambda execution context |
| Load balancer | Per-host health state, outlier-detection state, sticky-session tables, rate-limit counters |
| Microservice with no DB | In-memory request coalescing, Bloom filters, feature flag cache, config snapshots |
| API gateway | Auth token cache, JWT key cache, route table cache, rate-limit windows |
| CDN edge | The content cache itself; SSL session resumption; TCP fast-open cookies |
| Stream processor (no DB) | Watermarks, in-flight buffers, JIT, hash-shuffle state |

This hidden state means:
- **Cold starts are real**: a fresh pod takes seconds to minutes to reach the same p99 as a warm one. Premature traffic to cold pods hurts.
- **Restarts have cost**: kill a pod, you lose its connection pool, JIT, and caches; downstream sees a connection storm.
- **"Add more pods"** is not free: each new pod must warm up. Autoscaling that's too reactive is worse than too slow.

### 2.2 The three forms of hidden state

1. **Soft state (process memory)**: caches, pools, JIT code, GC structures. Lost on restart, rebuilt cheaply.
2. **Coupling state (between layers)**: connection state, sticky sessions, in-flight requests. Lost on restart, sometimes correctness-affecting.
3. **Configuration state**: feature flags, route tables, secrets. Distributed config. Refresh discipline matters.

Failure to recognize these is why "stateless tier" scaling reviews fail. A staff engineer asks at the design level: *"What hidden state do you have, and what is its lifecycle?"*

---

## 3. The Horizontal Scaling Math

### 3.1 USL (Universal Scalability Law)

Even purely stateless tiers do not scale linearly:

```
X(N) = N / (1 + α(N-1) + βN(N-1))

α = contention (serial work that doesn't parallelize: locks, mutexes, shared queues)
β = coherence (cross-instance coordination cost: cache invalidation, gossip, fan-in joins)
```

Where stateless tiers hit the wall:

- **α-bound**: a global mutex (Python GIL, single-threaded event loop fanning to one thread, single-thread queue) → throughput plateaus at 1/α.
- **β-bound**: cross-pod coordination via pub-sub, distributed locks, or shared cache invalidation → throughput peaks then declines as N grows.

**Staff insight**: when you see "we doubled instances and got 30% more throughput," your bottleneck is α or β, not capacity. Look for shared resources (a Redis queue all pods poll, a config server, a single-shard rate limiter, a logging collector).

### 3.2 Amdahl's Law on the request path

```
Speedup = 1 / ((1-P) + P/N)

P = parallelizable fraction
N = number of cores / instances
```

At MAANG, the request-path is *almost* fully parallel across instances, but there's always a non-parallel fraction:
- Cross-pod cache invalidations
- Centralized rate limiting
- Shared metrics aggregation
- Logging fan-in

This non-parallel sliver caps your speedup. A pure stateless tier with 1% serial work caps at 100× — and one common reason is logging.

### 3.3 The "two pizzas" of capacity

A simple sanity check before throwing instances at a problem:

```
Required pods  = (peak RPS × p99 latency × concurrency factor) / per-pod throughput
                / target utilization (e.g., 0.6 for headroom)

Example:
  Peak: 10M RPS
  per-pod: 5K RPS
  redundancy / regional / AZ overhead: ×1.7
  utilization ceiling: 0.6
  → 10M / 5K / 0.6 × 1.7 ≈ 5,700 pods

  At 4 vCPU each:        22,800 vCPU
  At 8 GB RAM each:       45 TB RAM
  At ~$50/pod/month       $285K/month
```

Plus cross-AZ network, deploy slop, autoscaling buffer, and you can quickly project total cost. Get the back-of-envelope before adopting any architecture.

### 3.4 Tail at scale (the canonical Dean & Barroso reading)

If a request fans out to N services, the request's p99 ≈ each service's `p_(99 - log10(N))` percentile:

```
Single service p99 = 10 ms means 1% are >10 ms.
Fan-out 10:  ~9.6% of requests have at least one slow leg → p99 → ~p99.0 is dominated by the slow leg
Fan-out 100: ~63% → your p99 is now ~p99.99 of a single service
Fan-out 1000:~99.99% → your p99 is now ~p99.9999 — essentially worst-case
```

The implication: **at large fan-out, the slow tail dominates regardless of average performance**. Improving p50 of dependencies barely moves the needle. Improving p99 (and p99.9) is the lever. Hedging, partial-result tolerance, and timeout discipline matter more than median.

---

## 4. Load Balancing — Where the Distribution Decisions Happen

### 4.1 The strategies, ranked by sophistication

```
1. RANDOM
   For each request, pick a backend uniformly at random.
   Pros: stateless on LB. Tail amplifies due to variance.
   Used for: quick prototypes, when backends are very similar.

2. ROUND-ROBIN
   Cycle through backends.
   Pros: deterministic, fair on average.
   Cons: doesn't account for backend variance (slow pod gets the same load).

3. LEAST-CONNECTIONS / LEAST-REQUESTS
   Send next request to backend with fewest in-flight.
   Pros: adapts to backend speed.
   Cons: requires LB to track state per backend; can herd new pods.

4. POWER OF TWO CHOICES (P2C)
   Pick 2 random backends; send to whichever has fewer in-flight.
   Pros: near-optimal balance with O(1) state per request.
   Cons: requires sharing in-flight count (gossip, EDS).
   Used by: Envoy, Linkerd, Finagle, gRPC.

5. WEIGHTED ROUND-ROBIN / LEAST-CONNECTIONS
   Weight backends by capacity.
   Pros: handles heterogeneous fleet (mixed instance types).
   Cons: weights drift; need active rebalance.

6. CONSISTENT HASHING
   Hash request key → consistent ring → backend.
   Pros: same key → same backend (cache locality, sticky cache).
   Cons: hot key → hot backend; rebalance on add/remove.
   Used for: cache-fronting tiers, partition-aware routing.

7. MAGLEV / RING HASH
   Google's improvement on consistent hashing for LB.
   Pros: minimal disruption on backend churn (1/N keys move).
   Used by: Google front-ends, Envoy's "ring hash".

8. EWMA (Exponentially-Weighted Moving Average) / latency-aware
   Track per-backend latency; send to fastest.
   Pros: avoids slow pods automatically (gray failure).
   Cons: requires latency tracking; can over-react to noise.
   Used by: Finagle (Twitter), Envoy least_request with bias.

9. PEAK-EWMA
   Latency tracker that prioritizes recency of slowness.
   Pros: catches gray failures faster.

10. ZONE-AWARE / TOPOLOGY-AWARE
    Prefer backends in the same zone/rack as the LB.
    Pros: cuts cross-AZ traffic and latency drastically.
    Used in: Envoy + xDS topology, GKE, EKS topology-aware hints.
```

### 4.2 The hidden costs of each

- **Round-robin** at scale: a slow pod gets just as many requests; queue grows; eventually trips outlier detection. But until then, the slow pod has been hurting tail.
- **Least-connections**: works great for homogeneous workloads; **fails** when request work is bimodal (some 1ms, some 200ms). The fastest-completing pod gets disproportionately hammered.
- **P2C**: provably near-optimal *if* in-flight is a good proxy for load. **Falls apart** when requests differ wildly in CPU cost.
- **Consistent hashing**: needed for cache locality but introduces hot-shard risk if keys are skewed.
- **EWMA**: very good for gray failure, but requires careful smoothing (over-reacts to a single 1-second pause; under-reacts to slowly degrading host).

### 4.3 Outlier detection — the L7 immunity layer

Production-grade load balancing always includes **outlier detection** that ejects misbehaving backends:

```
Per-backend tracking:
  - 5xx rate (consecutive errors, %)
  - Latency p50/p95/p99 vs cluster
  - Connection failures
  - Slow stream resets

Eject when:
  - Consecutive 5xx > N (e.g., 5 in a row)
  - p99 > 3× cluster median
  - Connection refused

Ejection params:
  - base_ejection_time: e.g., 30s
  - max_ejection_percent: e.g., 50% (don't eject so many that load builds elsewhere)
  - success_rate_minimum_hosts: only enforce when fleet is large enough
```

Implemented in Envoy's `outlier_detection`, gRPC's `outlier_detection`, Kubernetes ingress proxies. **Without outlier detection, gray failures will dominate your tail**.

### 4.4 Layered LB — the actual production topology

A real MAANG topology is rarely "one LB":

```
                          Client (mobile / web)
                                │
                                ▼
          ┌─────────────────────────────────────────┐
          │ Anycast IP / GeoDNS (BGP, multi-region) │  ← global edge
          └────────────────┬────────────────────────┘
                           ▼
          ┌─────────────────────────────────────────┐
          │ Edge proxy (Envoy/Nginx, TLS term, WAF)  │  ← regional ingress
          └────────────────┬────────────────────────┘
                           ▼
          ┌─────────────────────────────────────────┐
          │ Regional gateway (auth, rate limit, OPA) │
          └────────────────┬────────────────────────┘
                           ▼
          ┌─────────────────────────────────────────┐
          │ Service mesh sidecar (Envoy)             │  ← pod-local
          └────────────────┬────────────────────────┘
                           ▼
                       Application pod
                           │
                           ▼ (outbound)
          ┌─────────────────────────────────────────┐
          │ Sidecar → other service via xDS/EDS LB   │
          └────────────────┬────────────────────────┘
                           ▼
                       Other service pod
```

Each hop is a load balancer with its own algorithm:

- Anycast/GeoDNS: distance-based.
- Edge: P2C with topology awareness.
- Gateway: weighted, based on customer tier.
- Sidecar: P2C + zone-aware + latency-aware.

A staff-level mistake: assuming "the load balancer" picks backends. There are 3–5 in series, each with own decisions, each with own failure modes.

### 4.5 The C10K → C10M problem

A single LB instance handling load:

| Generation | Concurrent connections | Problem |
|---|---|---|
| C10K (early 2000s) | 10K | Threaded I/O hits process/thread limits |
| C100K | 100K | epoll/kqueue solves it; tune `nofile` |
| C1M | 1M | Per-connection memory, kernel sockets, TIME_WAIT pile-up |
| C10M | 10M | Kernel networking limits; kernel-bypass (DPDK), io_uring, or sharded LBs |

Tunables that matter at MAANG scale:

```
sysctl -w net.core.somaxconn=65535
sysctl -w net.ipv4.tcp_max_syn_backlog=262144
sysctl -w net.core.netdev_max_backlog=262144
sysctl -w net.ipv4.tcp_tw_reuse=1
sysctl -w net.ipv4.tcp_fin_timeout=15
sysctl -w net.ipv4.ip_local_port_range="10000 65000"
sysctl -w fs.file-max=12000000
ulimit -n 1048576
```

And NIC-level:
- Multi-queue NICs (RSS/RPS) so traffic spreads across CPU cores.
- `SO_REUSEPORT` for multiple processes accepting on same port.
- IRQ affinity pinned to specific cores; application threads pinned to others.

If your LB is hitting 80% CPU, the answer is rarely "more pods" — it's tuning, multi-process accept, NIC queue mapping, or kernel bypass.

---

## 5. Service Discovery — The Backbone

### 5.1 Discovery models

```
1. CLIENT-SIDE DISCOVERY
   Client queries registry, picks a backend, opens connection.
   Pro: no extra hop; simple.
   Con: every client implements it; library upgrade hell.
   Used by: Eureka + Ribbon (old Netflix), gRPC's xds_resolver.

2. SERVER-SIDE DISCOVERY
   Client → LB (which queries registry) → backend.
   Pro: single implementation in LB; clients are dumb.
   Con: extra hop, LB is critical path.
   Used by: AWS ALB, GCP LBs.

3. SIDECAR DISCOVERY
   Client → sidecar (per-pod) → backend, sidecar handles registry.
   Pro: language-agnostic; client is dumb; in-process locality.
   Con: sidecar costs CPU/RAM per pod.
   Used by: Envoy + xDS, Istio, Consul Connect, Linkerd.

4. DNS-BASED
   `service.cluster.local` → DNS round-robin or SRV records.
   Pro: zero infra.
   Con: DNS TTL caching, no health awareness, no rich metadata.
   Used by: Kubernetes services (default), EKS.
```

### 5.2 The xDS family — Envoy's gold standard

```
LDS  Listener Discovery Service       — what ports/listeners are open
RDS  Route Discovery Service          — routing rules per listener
CDS  Cluster Discovery Service        — upstream service definitions
EDS  Endpoint Discovery Service       — actual IP:ports for backends
SDS  Secret Discovery Service         — TLS certs, mTLS identities
ADS  Aggregate Discovery Service      — single stream with all of the above
```

Pushed from a **control plane** (Istiod, Pilot, custom xDS server) to data plane sidecars. Push-based (control plane streams updates), so propagation is fast (milliseconds) and consistent.

### 5.3 Failure modes of service discovery at scale

- **Registry overload**: 100K pods × heartbeat every 30s = ~3,300 RPS to registry. Tractable; but at 1M pods, you'll need a registry tier of its own.
- **Stale registry**: a pod died but registry still lists it. Clients try to connect → connection refused. **Mitigation**: short health-check timeouts + outlier detection in LB to mask the staleness.
- **Registry partition**: registry unreachable. Fallback to last-known cache. **Mitigation**: cache TTL longer than registry outage tolerance, with degraded-mode flag.
- **Push storm**: a config change flips state of 50K endpoints; xDS pushes update to 100K sidecars simultaneously. Network/CPU spike. **Mitigation**: incremental xDS (delta updates), debouncing, jittered push.
- **DNS TTL conflicts**: TTL=60s but pods restart every 30s → half traffic to dead IPs. **Mitigation**: very short TTL or LB-level tracking, not DNS.

### 5.4 The "config surface" trap

At scale, the config distributed to sidecars (routes, clusters, endpoints, certs) becomes huge:

```
1,000 services × ~50 routes each × full config to all sidecars =
  → multi-GB push to each sidecar
  → sidecar takes seconds to digest
  → CPU spike, memory spike, brief downtime per sidecar
```

Mitigations:
- **Scoped/sandbox xDS**: only send a sidecar the config relevant to its service.
- **Incremental delta xDS**: only send changes.
- **Lazy-load**: discover endpoints on first request, not preemptively.

This is what Istio's "ambient mesh" and similar second-generation meshes are addressing.

---

## 6. Connection Management — The Hidden Cost Tier

### 6.1 The connection pool trade

Every stateless service maintains pools to its dependencies. At MAANG scale, these pools are the most operationally subtle aspect of the tier:

```
Pool too small:
  - Threads block waiting for connection
  - Tail latency builds up
  - Throughput plateaus below capacity

Pool too big:
  - Memory pressure on client side
  - Connection storm on server (open + handshake costs)
  - Server's connection limit hit; new connections refused
  - TCP TIME_WAIT pileup on client when pool churns

Sweet spot:
  pool_size ≈ peak in-flight requests to that backend
              = peak RPS × p99 latency to that backend
```

### 6.2 Per-pod fan-out math

```
Pod A has clients to 30 downstream services.
Each client: pool of 50 connections.
Total open connections from one pod: 1,500.
Across 5,000 pods: 7.5M open connections fanning out.

At each downstream service:
  Inbound connections = 5,000 pods × 50 connections per pod = 250,000 connections.
  Even if connections are mostly idle, they consume:
    - File descriptors (1 per conn)
    - Kernel sockets (a few KB each)
    - Memory at server (TCP buffers, ~16KB per conn) → 4 GB RAM just for buffers.
```

This is why **connection multiplexing** (HTTP/2 streams, gRPC streams) is essential: instead of 50 connections per client, 1 connection with 100 multiplexed streams.

### 6.3 Connection lifecycle pathology

```
1. Cold connection storm
   Pod restart. 5K open conns gone. They re-open simultaneously to downstreams.
   Downstream is hit with 5K SYNs at once. SYN queue overflows. Some connections fail.
   Mitigation: graceful drain on shutdown (existing requests finish, new ones to other pods);
              warm-up period for new pods (low traffic ramp).

2. TIME_WAIT pile-up
   Many short-lived connections (HTTP/1.1 with no keep-alive) → TIME_WAIT entries linger.
   Local port exhaustion (only 65K ports per IP).
   Mitigation: keep-alive, HTTP/2, tcp_tw_reuse=1.

3. Pool starvation under burst
   Burst of requests > pool size → tasks queue → tasks time out → cascading failure.
   Mitigation: bounded queue with shedding, not unbounded; "fail fast" preferred over queue.

4. Connection imbalance
   When a new pod joins, existing pods don't immediately re-pool; new pod gets 0 traffic.
   Mitigation: connection rebalancing (Envoy "outlier_detection" + "min_request_count").

5. Slow connections
   One backend pod has packet loss → its connections are slow but not failing.
   Pool's least-loaded heuristic keeps sending to it (because in-flight is small thanks to slow drain).
   Mitigation: outlier detection on latency, not just errors.
```

### 6.4 Keep-alive and HTTP/2 vs HTTP/1.1

```
HTTP/1.1 + keep-alive:
  - 1 TCP connection per concurrent request to same backend
  - Pipelining rarely usable (head-of-line blocking)
  - Connection per pool entry

HTTP/2:
  - 1 TCP connection multiplexes 100+ concurrent streams
  - Per-stream priorities, flow control
  - Dramatically lower connection count
  - But: TCP head-of-line blocking still exists (one packet loss → all streams pause)

HTTP/3 (QUIC):
  - UDP-based; per-stream packet flow
  - True stream independence (one loss → one stream pauses)
  - Better tolerance of mobile network reordering
  - 0-RTT handshake (with care for replay attacks)
```

At MAANG scale, HTTP/2 is mandatory internally; HTTP/3 is increasingly preferred for client-facing edge (Cloudflare, Google, FB all run QUIC widely now).

### 6.5 mTLS — the security cost no one budgets for

mTLS handshake: ~1–5 ms per connection (depends on cipher, key size, RTT). For long-lived connections, amortized to ~0. For short-lived, it's a per-request cost.

At fleet scale:
```
100K mTLS-enabled pods, each opening 100 outbound mTLS conns/sec at startup =
  10M handshakes/sec briefly during a deploy.
  CPU on the certificate authority and on every pod spikes.
```

Mitigations:
- **Long-lived keep-alive** (so handshakes amortize).
- **Session resumption** (TLS 1.3 0-RTT) — note 0-RTT replay caveats for non-idempotent ops.
- **SDS push** for certificate rotation without restart.
- **Certificate caching at sidecar** so app doesn't re-handshake within a pod.

---

## 7. Cold Starts and Warmup — The Tail Killer

### 7.1 What slows a fresh pod

```
1. Application bootstrap
   - JVM/CLR class loading (seconds for large apps)
   - Lazy-loaded dependencies
   - Initial config fetch
   - Initial service-discovery population
   - Initial cache warming

2. Connection establishment
   - 30+ outbound connections to dependencies
   - mTLS handshakes
   - Pool warm-up

3. JIT compilation
   - First few thousand requests are interpreted, not JIT'd
   - p99 of those is 5–50× slower than steady state
   - Java HotSpot tiered compilation: starts with C1, escalates to C2 (heavily optimized)

4. CPU caches and TLB
   - Cold CPU caches → first requests pay memory-fetch latency
   - TLB cold → translations cause page-table walks

5. Memory allocator state
   - Heap not yet expanded; hot allocations cause page faults

6. OS-level page cache
   - Container image pages not yet hot in cache
```

A fresh pod's p99 may be 10× steady-state for the first 60–300 seconds. Sending it production traffic immediately *is* a service degradation.

### 7.2 Warm-up strategies

**Synthetic traffic priming**:
- Before announcing readiness, the pod runs a script that hits its own endpoints with realistic queries.
- Goal: trigger JIT compilation, fill caches, establish connections.

**Slow-start LB**:
- New pod gets fractional traffic for first N seconds, ramping up.
- Envoy `slow_start_config`, NGINX `slow_start`. Critical for high-throughput tiers.

**Pre-pulled images**:
- Container image pre-fetched on the node before scheduling, so cold-start doesn't include image download.

**Pre-flight health checks**:
- The readiness probe waits for warm-up to complete, not just "process is up."

**Snapshotted JIT**:
- Java's CRaC (Coordinated Restore at Checkpoint), AppCDS, GraalVM native-image to skip JIT entirely.

### 7.3 Thundering herd on cold start

A scale-up event creates 100 new pods at once. They all:
- Connect to dependencies → connection storm.
- Fetch config → config server spike.
- Pull cert → CA spike.
- Warm caches → fan-out reads to backend.

Mitigations:
- **Jittered start**: new pods start over a window, not simultaneously.
- **Connection rate limit at startup**: a pod opens connections gradually.
- **Config caching with proximity**: each node has a local config cache; pods read from there.
- **Image side-loading**: image already on the node.

---

## 8. Runtime, GC, and Memory at Scale

### 8.1 GC pause = latency tail

```
Java G1GC default config on a 32GB heap:
  - Young GC: 5–20 ms (frequent)
  - Mixed GC: 50–200 ms (less frequent)
  - Full GC: 500ms–10s (rare but catastrophic)

Go runtime (goroutine + concurrent GC):
  - Stop-the-world phases: 0.1–1 ms typical, can spike under pressure
  - GC runs concurrent with allocation; CPU cost ~25% peak

Node.js (V8):
  - Major GC: 30–200 ms
  - Mark-sweep with concurrent compaction

Rust / C++:
  - No GC; latency tail driven by allocator + page faults + thread scheduler
```

At p99 = 10ms SLO, even a 20ms GC pause is a tail event. At p999 = 50ms SLO, a single full GC is your daily breach.

### 8.2 Tuning for tail

**JVM**:
- ZGC or Shenandoah for sub-10ms pauses on >32GB heaps.
- G1GC with `-XX:MaxGCPauseMillis=20` and aggressive young-gen sizing.
- Off-heap caches (Caffeine, Chronicle) to avoid GC pressure on hot data.
- `-XX:+UseLargePages` for huge-page-backed heap.

**Go**:
- `GOGC` tuning (default 100; lower = more frequent but smaller GCs).
- `GOMEMLIMIT` to cap memory and force GC when needed.
- Avoid pointer-heavy data structures (slice of struct vs slice of *struct).
- Pool of byte buffers (`sync.Pool`) for ephemeral allocations.

**Node**:
- `--max-old-space-size`, `--optimize-for-size`.
- Avoid retained large objects; weak references where possible.
- HTTP/2 and worker threads to spread CPU.

**Rust**:
- jemalloc or mimalloc instead of system allocator.
- Watch for allocator fragmentation under churn.
- Profile with `heaptrack`, `perf`, `cargo flamegraph`.

### 8.3 Memory budget per pod

```
Pod request work memory:    ~per-request × concurrency
Pod cache memory:            varies (10–80% of pod RAM)
Pod GC overhead:             1.2–2× live working set
Pod runtime / framework:     baseline (50MB Go to 1GB+ Java)

Total pod RAM = baseline + (per-req × concurrency) + caches + GC slack
              ~ 1–8 GB typical
```

Provision 30–50% headroom over peak; OOM-kills have a cost (request loss, retry storm).

### 8.4 NUMA and CPU affinity

On large servers (32+ cores, multi-socket):
- Memory access across NUMA nodes is 2–3× slower.
- Default scheduling spreads threads → cross-NUMA traffic.
- `numactl --cpunodebind` to pin processes; `mempolicy` to prefer local memory.
- For ultra-low-latency: pin one process per NUMA node, dedicated NIC queue.

A staff engineer caring about p99.9 in latency-sensitive tiers (LB, edge proxy) cares about NUMA. For mid-tier services it's usually a non-issue.

---

## 9. Network Stack Tuning at the Highest Throughput

### 9.1 Where the kernel becomes the bottleneck

At >1M PPS (packets/sec) per host:
- Kernel TCP stack per-packet cost (~1 μs) becomes the limit.
- Context switches between kernel and user space dominate.
- A single CPU core handling NIC interrupts saturates.

Solutions:
- **Multi-queue NIC + RSS**: NIC distributes flows across many CPU cores' queues.
- **Kernel-bypass (DPDK, RDMA, AF_XDP)**: user-space drives the NIC, no kernel TCP stack.
- **eBPF / XDP**: process packets in eBPF programs at NIC hook (used by Cloudflare, Cilium).
- **io_uring** (Linux 5.1+): efficient async I/O without per-syscall cost.

These are used in the highest-throughput tiers (anycast LBs, edge proxies, CDN, DDoS scrubbers). They are operationally heavy (debugging is harder; observability tools are NIC-stack-dependent).

### 9.2 Connection limits and TIME_WAIT

```
Local port exhaustion:
  - Each outbound connection consumes a (src_ip, src_port, dst_ip, dst_port) tuple.
  - At ~28K ports per local IP (in default range) → connection limit.
  - Mitigation: tcp_tw_reuse, multiple source IPs (SO_BINDTODEVICE), connection pooling.

TIME_WAIT pile-up:
  - 60s default on Linux. Connections stuck waiting for late packets.
  - At high churn, pile up to millions. Memory pressure.
  - Mitigation: longer-lived connections (HTTP/2), tcp_tw_reuse=1, tcp_fin_timeout=15.

SYN queue overflow:
  - Brief incoming-connection burst; kernel backlog full; SYNs dropped.
  - Tunable: net.ipv4.tcp_max_syn_backlog, net.core.somaxconn.

Buffer sizes:
  - Default TCP send/recv buffer (~16KB) too small for high-bandwidth paths (BDP > 1 MB).
  - Tunable: net.ipv4.tcp_rmem, tcp_wmem; or per-socket SO_RCVBUF / SO_SNDBUF.
```

### 9.3 Latency-sensitive socket options

```
TCP_NODELAY: disable Nagle's algorithm (default delay 200ms for small packets).
             — Always enable for interactive RPC paths.
TCP_QUICKACK: send ACKs immediately (vs delayed ack default).
TCP_USER_TIMEOUT: kill connection if no ACK for N ms (faster failure detection).
SO_REUSEPORT: multiple processes accept on same port (kernel does fanout).
SO_BUSY_POLL: poll the NIC instead of waiting for interrupt (lower latency, higher CPU).
```

---

## 10. Autoscaling — The Subtle Art

### 10.1 The scaling signals

```
- CPU utilization (HPA default)
- Memory
- Custom metrics (RPS per pod, queue depth)
- External metrics (SQS queue length, Kafka consumer lag)
- Predictive (forecast based on time of day / week)
```

CPU is the lazy default and is often wrong:
- Some workloads bottleneck on memory or I/O before CPU.
- Some are CPU-bound by inefficient code; scaling doesn't help correctness.
- CPU averaging hides per-pod variance.

A staff engineer prefers **load-based** signals: RPS-per-pod or queue-depth, with CPU as a fallback.

### 10.2 The autoscaling lag problem

```
Load spike at t=0
  ↓ 
Metrics aggregate (15–60s window)
  ↓
HPA evaluates (every 15s)
  ↓
Scale decision: add pods
  ↓
Scheduler picks nodes (5–30s)
  ↓
Image pull (10–60s if not cached)
  ↓
Container start (5–20s)
  ↓
App bootstrap (10–60s)
  ↓
Warm-up (30–300s)
  ↓
Pod actually serving at full p99
  ────────────────────────────
  Total: 2–10 minutes from spike to relief
```

In a 10M RPS tier, 5 minutes of unrelieved spike = blowing through capacity = SLO breach.

Mitigations:
- **Pre-warmed pool** of standby pods (idle but warm).
- **Predictive scaling** (scale up before the daily peak).
- **Faster scale-up than scale-down** (asymmetric thresholds).
- **Buffer capacity** built into steady-state (run at 50–60% utilization, so a 2× spike is absorbed).
- **Graceful degradation** (drop low-priority work when overloaded, see §15).

### 10.3 The thrashing trap

If your scale-up threshold is at 70% CPU and scale-down at 50%:
- Scale up at 70%; new pods bring average down to 60%.
- 10 minutes later, traffic dips → 45% → scale down.
- Now back at 65% → near scale-up threshold.
- A small wobble → flapping.

Mitigations:
- **Stabilization windows**: don't scale down for N minutes after scale up.
- **Hysteresis**: large gap between up and down thresholds.
- **Predictive smoothing**: scale based on trend, not snapshot.

### 10.4 Cluster autoscaler vs HPA

Two independent autoscalers:

- **HPA (Horizontal Pod Autoscaler)**: scales pods within a cluster.
- **Cluster Autoscaler / Karpenter**: scales nodes when pods are unschedulable.

They can fight or compose poorly:
- HPA wants more pods → CA must add nodes → 1–3 minute lag.
- CA scales down a node with running pods → eviction → pod restart.

Mitigations:
- **Pod Disruption Budgets**: cap how many pods a deployment can lose at once.
- **Scheduling policies**: prefer to fill existing nodes (bin-pack) over spreading.
- **Node pools** by workload class (memory-intensive vs CPU-intensive).
- **Preemption policies**: ensure critical pods are not evicted.

### 10.5 Cost vs latency in autoscaling

- Tighter scaling → lower cost, higher latency tail during spikes.
- Looser scaling → higher cost, smoother tail.
- The right balance depends on **what the user sees**: a 5x cost difference between "scale at p50" and "scale at peak forecast" is real.

Spot instances / preemptibles are 60–90% cheaper but require fault-tolerance:
- Workloads that can tolerate sudden node loss.
- Mixed pools with on-demand fallback.
- Graceful drain on preemption signal (~30s warning on AWS Spot, 30s on GCP preemptible).

---

## 11. Retries, Backoff, Circuit Breakers — The Resilience Stack

### 11.1 Retry math — why naive retry kills

A downstream is overloaded. Each upstream retries 3× on failure. Now downstream is hit with 3× the load it couldn't handle the first time. **Cascading failure within seconds.**

The first staff-level rule: **retries amplify load during the worst possible time.**

### 11.2 Backoff strategies

```
Constant: retry every X ms          → bad. Synchronizes retries.
Linear:   X, 2X, 3X, ...            → better, but still synchronized.
Exponential: X, 2X, 4X, 8X...        → worse synchronization than linear, oddly.
Exponential + jitter:                → good. Spreads retries.
  Full jitter:    rand(0, exp_backoff)
  Equal jitter:   exp/2 + rand(0, exp/2)
  Decorrelated:   rand(base, prev_backoff × 3)
```

AWS recommends **decorrelated jitter** (paper by Marc Brooker). It avoids the "thundering" of full jitter while smoothing retries.

### 11.3 Retry budgets

Cluster-wide cap on retry traffic:

```
Allow up to 10% of total RPS to be retries.
If retries exceed budget → drop them (just return failure to caller).
```

Implemented in:
- Envoy (`retry_budget`).
- gRPC (configurable).
- Hystrix / Resilience4j.

The point: even if individual callers retry "responsibly", the *aggregate* retry traffic from 10K pods can DDoS your downstream. A budget is the only true bound.

### 11.4 Circuit breaker — failure-detection at fan-out

```
States:
  CLOSED:   normal operation, requests flow through.
  OPEN:     too many failures → fail fast without calling backend.
  HALF-OPEN: probe with a few requests; if success, close; else stay open.

Triggers:
  - Error rate > 50% over 30s window
  - Concurrent errors > N
  - p99 latency > threshold

Effect:
  - Stops cascading failures.
  - Releases pressure on overloaded backends.
  - Gives backends time to recover.
```

Two flavors:
- **Per-host circuit breaker** (outlier detection): isolate one bad pod.
- **Per-cluster circuit breaker**: stop calling a service entirely.

### 11.5 Bulkheads — isolation between tenants and dependencies

Inspired by ship hulls: separate compartments so flooding in one doesn't sink the ship.

```
Dependency-bulkhead:
  Service A calls B and C.
  Use *separate* connection pools / thread pools for B and C.
  If B is slow, A's calls to B exhaust *only* the B-pool, not C-pool.

Tenant-bulkhead:
  Per-tenant queue / pool.
  Tenant X's runaway traffic exhausts only X's resources.
```

A staff-level system without bulkheads will, given enough time, suffer "head-of-line blocking from the slowest dependency" — and you'll spend a quarter eliminating it after the incident.

### 11.6 Hedged requests (revisited)

```
For idempotent reads:
  send to A.
  if no response in 95th-percentile latency → also send to B.
  take whichever returns first.
  cost: ~5% extra load, p99 cut by ~50%.
```

Hedge delay is critical:
- Too short: doubles load with little tail benefit.
- Too long: hedging fires only after timeout, no benefit.
- Right value: ~p95 of the call (so only the slow 5% trigger hedge).

For writes: only safe if writes are idempotent and downstream deduplicates.

### 11.7 Speculative execution

A more aggressive form: kick off both calls upfront, take first.

- 100% extra load.
- Only justifies for highest-priority requests with strict tail SLOs.
- Used in BigQuery, Spanner, some payment paths.

---

## 12. Rate Limiting and Fairness

### 12.1 Where to rate-limit

```
Edge (CDN / gateway):
  - DDoS protection
  - Geo-rate-limits
  - Per-IP for unauthenticated

Gateway (per-customer, per-tier):
  - Quota enforcement
  - Differentiated service (free tier vs enterprise)

Service-to-service:
  - Per-tenant
  - Per-priority

Dependency:
  - Token bucket per outbound destination
  - Concurrent-request limit (semaphore)
```

A request hitting a free-tier user might have 5 rate limiters in series; a paid request gets 2.

### 12.2 Algorithms

```
1. FIXED WINDOW
   "100 reqs per minute, reset at :00 of each minute."
   Pro: simple. Con: bursts at boundaries (1.99× the limit at the :59→:00 cusp).

2. SLIDING WINDOW LOG
   Store every request timestamp; count those in the last 60s.
   Pro: precise. Con: memory grows with QPS.

3. SLIDING WINDOW COUNTER
   Two counters: previous and current minute.
   Estimate by weighted average of overlap.
   Pro: low memory, near-precise.

4. TOKEN BUCKET
   Bucket fills at rate R tokens/sec, capped at burst size B.
   Each request takes one token; reject if empty.
   Pro: handles bursts. Standard at MAANG.

5. LEAKY BUCKET
   Queue with constant drain rate.
   Pro: smooths bursts. Con: queueing latency.

6. CONCURRENT-LIMIT (semaphore)
   "At most N in-flight requests."
   Pro: protects against resource exhaustion. Often combined with rate limit.
```

### 12.3 Distributed rate limiting

A single service instance's local rate limit can't enforce per-tenant quota across thousands of pods. Need a coordinated counter:

```
1. CENTRAL COUNTER (Redis-based)
   Every request: INCR redis:tenant:X:current_minute.
   Pros: precise. Cons: latency per request, central failure point.

2. PER-POD QUOTA WITH CENTRAL RESHARING
   Each pod has 1/N share of quota.
   Periodically rebalance based on usage.
   Pros: fast. Cons: imprecise during rebalance.

3. APPROXIMATE TOKEN POOL (gossip)
   Pods gossip their consumption; aggregate every few seconds.
   Pros: scales. Cons: brief over-quota moments.

4. EDGE-ENFORCED, ASYNC RECONCILIATION
   Edge enforces best-effort; reconcile + bill from log later.
   Pros: zero latency cost. Cons: can over-serve briefly.
```

Stripe / Cloudflare use variations of #3 and #4. AWS API Gateway uses ~#2 with finer reconciliation.

### 12.4 Fairness — preventing one tenant from monopolizing

Even within rate limits, one heavy tenant can saturate shared CPU/queue:

```
Weighted fair queueing:
  Each tenant has a weight; CPU/queue allocated proportionally.

Deficit round-robin:
  Each tenant accumulates "deficit" tokens; serve in order of need.

Stochastic fair queueing:
  Hash tenant to a queue; round-robin queues.
  Approximates fairness with O(1) per request.

Least-recently-used scheduling:
  Whichever tenant hasn't been served longest gets next slot.
```

Used by AWS Lambda's per-account concurrency, GCS for per-bucket QPS, Twitter's fairness scheduler.

---

## 13. Backpressure Across Stateless Tiers

Because stateless tiers usually don't store load, **the tier in front feels overload via timeouts and 503s**. Without backpressure, this becomes:

- Caller sees timeout → retries → makes it worse.
- Caller's connection pool fills up → caller's caller times out → propagates back.

### 13.1 Cooperative backpressure

The downstream service signals overload explicitly:

```
HTTP 429 + Retry-After header
HTTP 503 + Retry-After
gRPC RESOURCE_EXHAUSTED
Custom: "X-Quota-Remaining: 0" header
TCP-level: ECN (Explicit Congestion Notification)
```

The upstream listens and:
- Backs off
- Sheds its own work proportionally
- Doesn't retry
- Maybe even sheds *its* upstream

### 13.2 Adaptive concurrency limiting

Netflix's `concurrency-limits` library / AWS App Mesh:

```
Each caller maintains a concurrency limit per dependency.
Adjust based on observed latency:
  - Latency stable → grow limit slightly (additive).
  - Latency rising → cut limit aggressively (multiplicative).
  - AIMD: like TCP congestion control.
```

This makes upstream "feel" the downstream's pain through latency and adapt without explicit signals.

### 13.3 The end-to-end deadline

A request enters with a deadline ("must respond in 200ms"). Each hop subtracts elapsed time and propagates remaining deadline:

```
Edge:        deadline=200ms, elapsed=10ms, propagate 190ms
Gateway:     deadline=190ms, elapsed=15ms, propagate 175ms
Service A:   deadline=175ms, elapsed=20ms, fan-out to B with 155ms
Service B:   deadline=155ms, work, return at 30ms
A composes,  return at 80ms total.
```

If at any hop the deadline is already exceeded → return immediately (don't even start work). This is the most efficient form of backpressure: no wasted work for already-doomed requests.

Implemented in: gRPC's `Context` deadline, OpenTelemetry's W3C trace context, Envoy's timeout propagation. **The number-one tool for staying under SLO at fan-out.**

---

## 14. Fan-out Composition Patterns

Stateless tiers compose dependencies. Three patterns dominate:

### 14.1 Sequential

```
A → B → C → D
total p99 ≈ p99_A + p99_B + p99_C + p99_D (unless deadlines or hedging used)
```

Compounding tail. Avoid serialization where parallelizable.

### 14.2 Parallel scatter-gather

```
A → [B, C, D, E] in parallel
  → wait for all
  → compose
total p99 ≈ max(p99_B, p99_C, p99_D, p99_E)
```

But: with N parallel calls each at p99 = X, the fan-out's p99 ≈ X^(1/N) percentile of any single call — basically the slowest of N. Tail amplifies as N grows.

Mitigations:
- **Partial-result tolerance**: return after first M of N or after deadline, with degradation flag.
- **Hedging** on the fan-out arms.
- **Lower per-arm timeout** to cap tail.

### 14.3 Speculative parallel + cancel

```
A → [B, B'] in parallel (B' is hedged)
  → take first response
  → cancel the other
```

A "tied request" — same logical call, two physical sends. Cuts tail at modest extra cost.

### 14.4 Streaming aggregation

Instead of waiting for all responses, *stream* responses as they arrive:

```
Client requests "show top 10 results"
A scatter-gathers from 30 shards, returning to client as fast as each shard responds.
Client renders progressively.
p99-equivalent latency: time-to-first-result, not time-to-completion.
```

Used in Google search, Netflix recommendations, Twitter timeline composition.

---

## 15. Graceful Degradation

When the system is overloaded or partially failing, **return less, faster** rather than nothing slowly.

### 15.1 Degradation hierarchy

```
Full service       — all features, fresh data, all fan-outs
       ↓
Reduced features   — drop non-essential calls (skip "trending sidebar")
       ↓
Stale data        — serve from cache even if expired
       ↓
Default fallbacks — skeleton UI, generic content
       ↓
Static page       — pre-rendered, no personalization
       ↓
Maintenance page  — apologize and explain
```

Each level returns successfully but with reduced fidelity. Users get *something* instead of an error.

### 15.2 Implementing degradation

- **Feature flags**: kill switches for non-essential features.
- **Timeout-driven defaults**: if dependency call exceeds 50ms, return cached/default.
- **Circuit-breaker fallback**: open circuit → fallback function returns default.
- **Priority shedding**: under load, drop P3, P2, P1 in order (see §11 of stateful doc).

A real production stateless service has **dozens of fallback paths**, each tested in chaos drills.

### 15.3 The shadow-of-doubt principle

When degraded, the user should know:
- A small banner: "Some features are temporarily unavailable."
- An "outdated" indicator on stale data.

This is honesty + UX. Hiding degradation makes users think the product is broken when it's actually intentional brownout.

---

## 16. Deployment Patterns at Scale

A 10K-pod stateless tier deploys 5–50 times per day. Bad deploys are the single most common cause of production incidents (more than infra failures).

### 16.1 Progressive delivery

```
Stage 0: pre-production (canary in staging)
Stage 1: 1 pod (1 in 5,000) for 30 min
Stage 2: 1% of fleet for 1 hour
Stage 3: 5% for 1 hour
Stage 4: 25% for 1 hour
Stage 5: 50% for 30 min
Stage 6: 100%

Total: ~5 hours minimum for a sensitive change.
```

Each stage:
- **Anomaly detection**: compare new vs old version's metrics (latency, error rate, RPS distribution).
- **Auto-rollback**: if anomaly score > threshold, revert immediately.
- **Soak time**: long enough to surface delayed effects (memory leaks, cache eviction, etc.).

Tools: Argo Rollouts, Flagger, AWS CodeDeploy, custom (FB's Mira, Google's Borg-based rollers).

### 16.2 Blue/Green

```
Blue (current production)  ─┐
                            ├─► Router
Green (new version)        ─┘

Cut over from blue → green atomically.
Roll back: cut back to blue.
```

- **Pro**: instant rollback.
- **Con**: 2× capacity during deploy; not great for stateful at scale; safe for stateless if connection draining is right.

### 16.3 Shadow / dark traffic

```
Live request → service A (production response)
            → service A' (new version), discard response

Compare responses, latency, error rate.
No user-visible impact.
```

Used heavily in deep-learning serving (test new model), payment flows (test new code without risking $$).

### 16.4 Connection draining

When taking a pod out of service:

```
1. Stop accepting new requests (LB removes from pool, ~10s propagation).
2. Wait for in-flight requests to complete (drain timeout, e.g., 60s).
3. Send SIGTERM; let app close gracefully.
4. After grace period, SIGKILL.
```

Sloppy draining = dropped requests = user errors.

### 16.5 Configuration deploys vs code deploys

Config deploys (feature flags, route changes) are 10–100× more frequent than code deploys. They have their own failure modes:

- Bad config rolls out instantly to all pods → no canary.
- Mitigation: progressive config rollout (FB's Conduit, GCP's Runtime Config, Cloudflare's gradual config).

A staff-level lesson: **config is code, treat it as such**. Reviews, tests, rollbacks, audit logs.

---

## 17. Serverless / FaaS at MAANG Scale

### 17.1 What FaaS is great for

- **Bursty workloads**: 0 → 10K RPS in seconds, then back to 0.
- **Independent functions**: image transforms, parsers, hooks, ETL stages.
- **Event-driven**: respond to S3 events, Kafka, queues, schedule.
- **Cost optimization on sub-1-RPS workloads**: pay per invocation, not idle.

### 17.2 What FaaS is not great for

- **Sustained high RPS**: per-invocation overhead is expensive at 10M RPS (Lambda is ~$0.20 per million invocations + per-ms compute). At sustained scale, a fleet of containers is cheaper.
- **Latency-critical paths**: cold starts can be 100ms–10s.
- **Long-running workflows**: 15-min Lambda limit; need Step Functions or move to containers.
- **Stateful or session-bound**: can be done but awkward.

### 17.3 Cold start mitigation

```
Provisioned concurrency:
  Pre-warm N instances; pay for them whether used or not.
  Ideal for predictable peaks.

Snapshotting (Lambda SnapStart, GraalVM):
  Snapshot a warmed-up runtime; instant restore.

Smaller artifacts:
  Smaller deployment package = faster cold start.
  Native binary > JVM > Python > Node.

Connection pooling outside the function:
  RDS Proxy, DAX, ElastiCache so each invocation doesn't open a new DB connection.
```

### 17.4 The "concurrency = scale" model

Lambda concurrency:
- Per-account concurrency limit (default 1000 per region).
- Per-function reserved concurrency.
- Burst limit (300–3000 invocations/min initial, scale-up rate).

At MAANG scale, Lambda is a *workload-class*, not a default. Pick it for what it's great at; stay with containers for the high-RPS hot path.

---

## 18. Edge / CDN — the Outermost Stateless Tier

### 18.1 What's at the edge

```
Geo-DNS / Anycast (BGP)        — directs to closest PoP
TLS termination                 — client-facing TLS
HTTP routing / WAF              — basic policy
Static asset caching            — images, JS, CSS, video segments
API response caching            — short-TTL caching of GET responses
Rate limiting                   — DDoS, per-IP
Edge compute (Workers, Lambda@Edge) — request mutation, A/B routing, auth
```

### 18.2 Cache hit ratio is the metric

```
CDN hit ratio of 98%:
  - 2% of requests reach origin.
  - Origin's effective load is 1/50th of edge.
  - Origin sized for 200K RPS instead of 10M RPS.
```

How to maximize:
- **Cache key normalization**: strip non-significant query params (utm_source, etc.).
- **Vary headers** carefully (don't shard cache by accept-language unless content varies).
- **Long TTLs with stale-while-revalidate**: serve stale, refresh background.
- **Cache versioning**: `/v3/css/main.css` so stale entries become unreachable.
- **Edge personalization**: do per-user logic at edge so cache key is still common.

### 18.3 Cache invalidation at edge

The hardest part of edge caching:

- **TTL-based** (default): wait for TTL.
- **Purge by URL**: explicit invalidate; can take 30s–5min to propagate.
- **Purge by tag**: invalidate all URLs with given tag (Cloudflare cache tags, Fastly surrogate keys).
- **Versioned key**: bump version on origin, old keys rot naturally.

A staff-level mistake: relying on instant edge purge for correctness. Edge purge is best-effort.

### 18.4 Edge compute

Beyond static caching, edge functions (Cloudflare Workers, Lambda@Edge, Vercel Edge Functions, AWS CloudFront Functions):

- Run JavaScript/Rust/Wasm at the edge in <1ms.
- Use cases: A/B testing routing, auth, request mutation, response transformation.
- Scale: handle 10s of millions of req/sec across edge fleet.
- Limits: very small CPU/memory budgets per request; no long-lived state.

This is **where stateless meets ultra-distributed**: 100s of locations worldwide, each running the same code, no coordination.

---

## 19. Multi-Region Traffic Management

### 19.1 Region selection strategies

```
Geo-DNS:
  DNS resolver returns the closest region's IP.
  Pro: simple. Con: 1-min TTL minimum, IP-based routing, doesn't react to outages fast.

Anycast:
  Same IP advertised from many regions; BGP routes to closest.
  Pro: instant failover (BGP withdraws bad region). Con: complex to operate, IP exhaustion risk.

Smart client (latency-based):
  Client measures latency to multiple endpoints; picks fastest.
  Pro: optimal per-client. Con: requires client logic; cold-start client doesn't know.

Application-level routing:
  Edge dispatches to region based on user attributes (region affinity), not just latency.
  Pro: aligns with data residency / shard placement. Con: more logic at edge.
```

### 19.2 Region failover

When a region fails, traffic must shift to another region:

```
Detection: ~30s (synthetic probes, error rates).
DNS update: 1–5 min (TTL-bound).
Or: Anycast withdrawal: ~30s (BGP propagation).
Or: Edge route change: ~10s.

Capacity in the surviving region: must absorb full traffic.
  → Provisioned at 1.5–2× steady state per region.
```

Without that 1.5×, "failover" means "now both regions are overloaded."

### 19.3 Active-active vs active-passive

```
Active-active:
  All regions serve traffic.
  Capacity: each region at ~50–60%; one fails, others absorb.
  Continuous validation; failover is "more traffic to other region", not "switch on cold standby".

Active-passive:
  One region serves; another is warm-standby.
  Cost: 50–100% standby capacity unused.
  Risk: standby may not work when needed (untested).
```

For stateless tiers, **active-active is almost always preferred** — capacity is utilized, failover is continuous-state.

### 19.4 The "region-pinned user" pattern

Even with active-active stateless tiers, the *stateful* layer often pins users to a region (data residency, latency). The stateless tier still serves any user from any region, but routes them to their pinned stateful region.

This means: the stateless tier may need to call across regions for stateful operations. Plan for that latency in your budget.

---

## 20. Failure Modes Specific to Stateless Tiers

### 20.1 Retry storms

Already covered (§11). Worth repeating because it's the #1 way stateless tiers cause cascading outages.

### 20.2 Connection storms

Mass restart → all pods reconnect → downstream blasted with SYNs and TLS handshakes. Mitigations: jittered restart, slow start, deploy windows.

### 20.3 The "deploy that takes down the cluster"

Most outages at MAANG come from deploys, not infrastructure. Examples:
- Memory leak that didn't show up in canary because canary wasn't long enough.
- Config that's syntactically correct but semantically wrong (wrong endpoint, wrong region).
- Code change that's faster on canary's cold cache but slower on production's warm cache (inverse of common case).
- Change that breaks under specific traffic mix the canary doesn't see.

Mitigations:
- **Long soak times** at each canary stage.
- **Anomaly detection across many metrics**, not just RPS and errors.
- **A/B traffic mirroring** so canary sees representative traffic.
- **Auto-rollback** is the safety net.

### 20.4 The "fix that makes it worse"

A staff-level pattern: an outage is detected, an engineer pushes a "fix," the fix introduces a different bug, now there are two outages. Mitigations:
- Outages are not deploy-time. Fixes go through canary.
- Code-as-config disabled during incidents (no flagging changes during firefight).
- Roll back instead of forward-fix.

### 20.5 DNS partial failure

DNS is the highest-blast-radius dependency. A misconfiguration affects everything. Mitigations:
- Multiple resolvers with redundancy.
- Local stub resolver caching.
- Negative cache (NXDOMAIN cached short).
- Health-check-based DNS (Route53, etc.).

### 20.6 Time skew

A pod's clock drifts; signed JWTs reject as expired or not-yet-valid; auth fails for that pod. Or, conversely, the pod thinks tokens are still valid that have expired. Mitigations:
- NTP ubiquitous, monitored for skew.
- Allow ±60s skew on token validation.
- Trust the issuer's "exp" with grace.

### 20.7 Memory leak

Slow leak in a stateless service: no immediate symptom; days later, OOM-kills cascade across fleet. Mitigations:
- Per-pod memory limits + automatic restart on threshold.
- Heap-dump on OOM; analyze post-incident.
- Release-on-restart pattern: restart pods every N hours to absorb slow leaks.
- "Death and rebirth" deployment pattern (Erlang).

### 20.8 Slow leak in connection / file descriptor

Forgotten `defer close()` → leaks connections → eventually pod fails to make new connections. Mitigations:
- Monitor FD count per pod.
- Connection pool maximum + eviction.
- Restart on threshold.

### 20.9 Cache poisoning

Bad value cached at edge → all users see broken response until TTL expires. Mitigations:
- Validate response before caching (don't cache 5xx).
- Short TTLs with frequent revalidation.
- Ability to instantly purge by URL or tag.

---

## 21. Capacity Planning Math (Beyond Stateful)

### 21.1 The real capacity equation

```
Required pods = (peak RPS / per-pod max RPS) × headroom × redundancy
              = (peak RPS / per-pod max RPS) × (1 / target_utilization) × (1 + region_overhead)

Example:
  peak: 10M RPS
  per-pod max: 5K RPS at 80% CPU
  target util: 60% steady
  region overhead (3 region active-active, lose 1): 1.5×

  pods = 10M / 5K / 0.6 × 1.5 ≈ 5,000 pods total across all regions
```

### 21.2 Latency budget allocation

Top-down:

```
SLO: p99 = 200 ms total

Edge routing            10 ms
TLS                     20 ms (or near-0 with reuse)
Gateway                 10 ms
Service mesh            5 ms
Application work        80 ms
  Fan-out depth 3, each at p99 25 ms → 50ms with hedging
  Local processing: 30 ms
DB calls (cached, mostly): 20 ms
Serialization           5 ms
Egress                  20 ms
Network return          20–60 ms
                       ─────────────
                        ~190 ms — fits

Each layer: budget × 80% = the level you actually engineer to,
            leaving slack for variance.
```

Designs that don't allocate budget top-down end up with a single layer eating 60% of the budget and other layers starved.

### 21.3 The 99.9%, 99.95%, 99.99% trade

```
99.0%   = 3.65 days/yr down — cheap
99.9%   = 8.7 hr/yr — basic redundancy
99.95%  = 4.3 hr/yr — multi-AZ
99.99%  = 52 min/yr — multi-region
99.999% = 5 min/yr — heroic
```

Each "9" costs roughly 10× the previous. Most user-facing products at MAANG target 99.95–99.99%; backend infra targets 99.99%+; payments / identity hit 99.999%.

### 21.4 Erlang-C and queueing

For systems where each request takes a known service time and arrivals are Poisson:

```
Required capacity to keep waiting probability < 1%:
  M/M/1: ρ < 0.99 just to be stable; for low queueing, ρ < 0.7
  M/M/c (c parallel servers): even at ρ = 0.85, wait probability << 1%

Rule of thumb:
  Stateless tier with c=many (many pods): can run hotter (~70%).
  Single-server queue (one pod for one tenant): must run cooler (~50%).
```

---

## 22. Cost Engineering at Scale

### 22.1 The cost stack

```
Compute (CPU/RAM):       30–50% of total
Network egress:           10–25% (cross-region, cross-AZ, internet egress)
Data transfer (intra):    5–15%
Storage (state, logs):   10–20%
Managed services:        10–20% (load balancers, queues, secrets)
```

A staff-level focus area: **network is often the largest unappreciated cost**, especially:
- Cross-AZ for HA setups.
- Cross-region replication.
- Egress to internet (CDN, mobile clients).
- NAT gateway costs (per-GB traversed).

### 22.2 Per-request cost

```
Total monthly infra cost / total monthly requests = $/M requests

At MAANG, hot paths are $0.01–$0.10 per million requests.
Cold paths can be $1+ per million.

A 1¢ improvement at 10M RPS × 86,400 sec × 30 days = ~$25M/yr saved.
```

This is why micro-optimizations in stateless tiers (lower memory per request, fewer cross-AZ hops, better CPU instructions) can fund their own engineering teams.

### 22.3 Strategies

- **Bin-packing**: dense pod placement on nodes; fewer nodes; better utilization.
- **Spot / preemptible**: 60–90% discount for fault-tolerant workloads.
- **Reserved capacity / Savings Plans**: 30–60% off for predictable steady-state.
- **Right-sizing pods**: smaller pods scaled wider often beat over-sized pods.
- **AZ-aware routing**: keep traffic in-AZ; cross-AZ ~$0.01/GB; at 10 PB/month that's $100K.
- **Compression**: gzip for HTTP; Snappy for internal RPC; trade CPU for network.
- **Protobuf over JSON**: 5–10× smaller, 2× faster to parse.
- **Cache aggressively**: cache hits cost $0; misses cost everything.
- **Move work to edge**: edge compute is cheaper than origin compute; less origin traffic.

### 22.4 The "10x cheaper" rule

If a re-architecture saves <10×, the migration cost usually exceeds the saving. Aim for an order of magnitude before moving.

---

## 23. Observability for Stateless Tiers

### 23.1 The four pillars

```
1. Metrics (Prometheus, M3, Atlas)
   - RED method: Rate, Errors, Duration per service
   - USE method: Utilization, Saturation, Errors per resource
   - Per-pod, per-tenant, per-route, per-dependency

2. Logs (structured, JSON)
   - Sampled at high QPS; full at error
   - Trace ID linkage
   - PII redaction at source
   - Retention: 7 days hot, 30 days warm, 1 year cold

3. Traces (OpenTelemetry, Jaeger)
   - Head-based sampling (e.g., 1%)
   - Tail-based sampling for slow/errored requests
   - Cross-service propagation via W3C trace context

4. Profiling (continuous)
   - Always-on CPU profiler (Pyroscope, parca, async-profiler)
   - Heap profiler periodically
   - Off-CPU profiler for blocked threads
```

### 23.2 The cardinality budget

Per-tenant × per-route × per-region × per-pod metrics: cardinality explodes.

```
1000 tenants × 50 routes × 3 regions × 5K pods = 750M time series.
Prometheus chokes at 10M; even M3 / Cortex at 100M without strategy.
```

Strategies:
- Drop low-value labels.
- Heavy-hitter sampling (top-1000 most active tenants only).
- Aggregated tiers (per-region, per-route → per-tenant aggregations downstream).
- Per-pod metrics → discard pod label except for debugging windows.

### 23.3 SLOs and error budgets

```
SLO: 99.9% of requests succeed within 200ms over 30-day window.

Error budget = 0.1% of requests = 1 in 1000 may fail.

If you've consumed 70% of budget by mid-month → freeze risky changes.
If you've consumed 100% → no changes until next window.
```

Used by Google SRE; broadly adopted. Aligns engineering risk-taking with reliability targets.

### 23.4 Distributed tracing pitfalls

- **Sampling bias**: 1% head-sampling misses rare slow requests. Use **tail-based** for slow/errored.
- **Trace propagation gaps**: missing a hop means traces look incomplete; debugging blame games.
- **Trace context size**: too many baggage items inflate request size; limit to ~1 KB.
- **Trace storage**: at 10M RPS × 1% sampling × 1KB/trace = 100 GB/sec. Need scalable backend.

---

## 24. Common MAANG Patterns — Concrete Implementations

### 24.1 Netflix Zuul + Hystrix → Envoy mesh

- Zuul as edge gateway.
- Hystrix for circuit breaking, bulkheads, fallback.
- Eureka for service discovery.
- Ribbon for client-side LB.

Modern Netflix: Envoy sidecars + xDS control plane (descended from the same lineage).

Lessons:
- Polyglot stack made it impractical to keep client libraries in sync.
- Sidecar approach moved resilience out of language libraries.
- Internal mesh now serves billions of internal RPCs/day.

### 24.2 AWS Global Accelerator + ALB + ECS / Lambda

- Anycast Global Accelerator entry point.
- ALB regional load balancer with health checks.
- ECS Fargate or Lambda for compute.
- API Gateway for managed API frontend.

Used by countless SaaS at AWS-scale. Anycast + edge optimization gives 30%+ TCP setup time reduction globally.

### 24.3 Google's frontend stack (GFE → Maglev → Borg)

- GFE: Google Front End. TLS termination, basic routing.
- Maglev: Google's L4 LB; consistent hashing, BGP-distributed, kernel-bypass-style.
- Borg: container scheduler.
- gRPC: internal RPC.
- Stubby (predecessor) → gRPC.

Maglev's claim to fame: a single Maglev box handles 10+ Gbps of LB traffic without dedicated hardware. The model: use commodity hardware + clever software (consistent hashing + BGP ECMP) to scale horizontally.

### 24.4 Cloudflare Workers / Edge

- 300+ PoPs worldwide.
- Workers run V8 isolates (not containers — much faster to start, ~5ms).
- Per-request CPU budget of ~50ms.
- 50M+ requests/sec across edge.

Lessons: at edge, **isolate-based runtimes** beat container-based for cold start. Tradeoff: must use a sandboxed JS/Wasm runtime; can't run arbitrary binaries.

### 24.5 Facebook / Meta's edge (Proxygen + Cartographer + Service Router)

- Proxygen: their HTTP server framework (open-sourced).
- Service Router: internal service-discovery + LB.
- TAO for graph state (covered in stateful doc).
- Cartographer for traffic-engineering.

The pattern: every tier has a **control plane** that pushes config; data plane is dumb.

---

## 25. Anti-Patterns — Staff-Level Red Flags

These should ring alarm bells in design reviews.

### 25.1 "It's stateless, so we'll just add instances"

Hidden state, GC, networking, DB connection limits, downstream load — all break this. Specifically:
- Are you load-testing with cold cache? Warmed connections? GC mid-burst?
- Does adding instances actually scale linearly to 10×, or does β-coherence cap you?
- Will you saturate your downstream first?

### 25.2 "We'll retry on failure" (no budget, no jitter)

Cascading failure waiting to happen. Always: retry budget + jittered backoff + circuit breaker.

### 25.3 "We'll just put it behind a load balancer"

Load balancers have their own failure modes and capacity limits. They are a tier with its own design (algorithm choice, outlier detection, draining, layered LB).

### 25.4 "Health check is a /healthz that returns 200"

True health check verifies actual dependencies (DB connection, downstream, disk). Liveness vs readiness vs startup probes are distinct concepts (Kubernetes makes this explicit; engineers often conflate).

### 25.5 "We'll deploy without canary because the change is small"

Most outages come from "small" changes. Treat config changes the same: progressive rollout, monitoring, auto-rollback.

### 25.6 "We'll add observability later"

By the time you need it, you can't add it cleanly. Tracing, metrics, structured logging are first-class day-1 work.

### 25.7 "We'll fix the leak by restarting periodically"

Sometimes pragmatic, but masks the bug. Eventually the leak rate exceeds the restart rate. Find and fix.

### 25.8 "We don't need rate limiting; we have a service mesh"

Service mesh gives the *primitives*; you still need to define *policies*. Default no-rate-limit means one bad client kills you.

### 25.9 "The deploy succeeded, so it must work"

Successful deploy ≠ working code. The next 30 minutes / 24 hours are when you find out. Soak time + anomaly detection.

### 25.10 "We have 99.9% SLO, so failures are fine"

99.9% is a budget, not a license. Spending it means less room for the next incident or change. Treat it like a financial budget.

### 25.11 "We'll handle that edge case in code"

Boundary conditions (clock skew, partial network, GC pause, OOM, hot key, retry storm) are not edge cases — they are guaranteed events at MAANG scale. Plan for them in design.

### 25.12 "It works on the canary"

Canaries don't see all traffic patterns. Specifically, they see new traffic (lighter pre-warmed cache than steady state) and may not see specific tenant queries. A canary is a smoke test, not validation.

### 25.13 "Just turn off the feature flag if it's bad"

Feature-flag systems themselves have outages. Design for the flag system being down (default to safe behavior).

### 25.14 "Per-host state is fine; we have sticky sessions"

Sticky sessions are anti-stateless. They couple availability to specific pods, prevent rolling deploys, and break when the pod dies. Avoid.

### 25.15 "We'll worry about cost when we're at scale"

By then, the cost is baked into the architecture. Design for unit economics from day 1: per-request cost, per-tenant cost, per-feature cost.

---

## 26. Decision Framework — How a Staff Engineer Reasons

When asked to design or scale a stateless tier:

### 26.1 Establish the contract

- **What's the peak RPS, and the burst peak above it?**
- **What's the latency SLO? Including which percentiles?**
- **What's the availability SLO?**
- **What does fan-out look like (depth and breadth)?**
- **What's the geographic distribution of users?**
- **What are the dependency latencies and SLOs?**
- **What's the deploy cadence?**
- **What does failure look like — silent corruption, data loss, dropped requests, slow but correct?**

### 26.2 Pick the topology

```
Inbound topology:
  Anycast → regional edge → gateway → service mesh → app

Outbound topology:
  App → sidecar → mesh → backend service
                        → cache
                        → queue / async
```

Each layer has a purpose; remove any that doesn't.

### 26.3 Pick the LB algorithms

Per layer: P2C? Consistent hash? Latency-aware? Topology-aware?

Pin algorithm to characteristics of the workload at that layer.

### 26.4 Pick the runtime / language

```
Latency-critical, low-overhead: Go, Rust, C++.
High-throughput batch / fan-out: Java (with ZGC/Shenandoah).
Glue / scripts: Python, Node.
Edge isolates: V8, Wasm.
```

The choice cascades into tooling, hiring, libraries — be deliberate.

### 26.5 Plan the resilience stack

- Timeouts at every hop.
- Retries with jitter and budget.
- Circuit breakers per dependency.
- Bulkheads per dependency / tenant.
- Fallbacks for graceful degradation.
- Hedging where idempotent.

### 26.6 Plan deploys, autoscaling, and the operational lifecycle

- Progressive delivery stages.
- Canary metrics and auto-rollback.
- HPA signals and limits.
- Pre-warmed buffer.
- Drain and rollback procedures.
- DR drills.

### 26.7 Plan observability

- Per-pod, per-tenant, per-route metrics.
- Sampling strategy for traces and logs.
- SLO dashboard with error budget tracking.
- Alerting hierarchy (page, ticket, log).

### 26.8 Estimate cost and capacity

- Per-request cost.
- Per-tenant cost.
- Burst capacity buffer.
- AZ/region failover capacity.
- Cost forecast for 1y, 3y growth.

---

## 27. Mental Models a Staff Engineer Carries (Stateless-Specific)

A condensed reference of mental models that produce correct staff-level reasoning quickly:

1. **Stateless is a lie.** Every stateless service has hidden state — pools, caches, JIT, runtime. Manage it.

2. **The tail dominates.** At fan-out, you're not optimizing average; you're optimizing the slow tail.

3. **Layers compose latency.** Allocate top-down; verify bottom-up.

4. **Retries amplify load.** Without budgets and jitter, retries are a self-DoS.

5. **Cold starts are real.** Slow-start pods, jittered restarts, pre-warmed pools.

6. **GC is a system constraint, not a tuning knob.** Pick runtimes that match SLO.

7. **Outlier detection > average health checks.** Gray failures evade simple checks.

8. **Backpressure is the only sustainable shedding.** Without it, you queue or retry until collapse.

9. **End-to-end deadlines** save more than any per-call optimization.

10. **Connection pools are state.** Size them based on load math, not vibes.

11. **Multi-region is active-active.** Active-passive untested = doesn't work.

12. **Deploys cause most outages.** Progressive delivery + auto-rollback are infra requirements.

13. **The slowest layer eats the latency budget.** Usually it's cross-region or fan-out.

14. **Cost = compute + network + memory.** Network is always more than you think.

15. **Hot-path optimization funds itself at MAANG scale.** A 10% CPU win in a fleet of 5K pods pays a senior engineer's salary in compute alone.

16. **The mesh is a tier.** Sidecar overhead is real (1–3 ms per hop). Plan for it.

17. **DNS is a critical-path dependency.** Cache, redundant resolvers, monitor.

18. **Bin-packing is operations.** Scheduler decisions are infrastructure decisions.

19. **Capacity = peak × burst × redundancy / utilization.** Memorize this.

20. **The slowest SLO is the system SLO.** Composed services chain availability.

21. **Edge is the most leverage.** Every request shaved from origin saves 10×.

22. **Boring is a feature.** Boring stateless tier = boring 3 AM. That's the goal.

23. **Hidden coupling kills.** Sticky sessions, in-process state, shared mutable globals. Make coupling explicit or eliminate it.

24. **The control plane is its own product.** Config distribution, service discovery, routing — give them their own SLOs.

25. **No request is more important than the system.** Drop the bottom 1% to save the other 99%.

---

## 28. The Stateless / Stateful Boundary — The Most Important Interface

The hardest design problem in MAANG infra is not stateless or stateful in isolation — it's **the contract between them**.

```
Stateless tier:                      Stateful tier:
- Many small pods                    - Few large pods (per shard)
- Add/remove freely                  - Add/remove via reshard
- Fast deploy                        - Slow deploy
- Per-request idempotent             - Per-request stateful
- Bursty                             - Steady (with hot keys)
- Optimizes for tail                 - Optimizes for durability
                ────────────────────────────
                The interface between them is the
                hot zone:
                - connection pooling
                - prepared statement caching
                - shard routing
                - read-your-writes guarantees
                - circuit breaking
                - idempotency keys
                - request coalescing
                - timeouts and deadlines
```

Decisions in this zone make or break the platform:

1. **Connection pool size at stateless calling stateful**: too big saturates DB; too small queues at app.
2. **Idempotency keys**: how does stateless retry without double-charging stateful?
3. **Read-your-writes**: how does the stateless tier know to route reads to a primary after a write?
4. **Cache invalidation**: who owns it, when, with what consistency?
5. **Circuit breaking on the DB**: when DB is slow, stateless tier must shed before adding more pressure.
6. **Routing**: how does stateless know which stateful shard owns a key (directory lookup, hash, etc.)?

A staff engineer thinks of this boundary as a **first-class system**, not just "the DB call." It deserves explicit SLO, capacity model, and resilience design.

---

## 29. Closing Note — The Staff Mindset for Stateless

The technical content matters, but the staff-level shift in mindset is:

- A senior engineer optimizes for the service handling its load correctly.
- A staff engineer optimizes for the **tier as a fleet of services** that:
  - Survives cascading failures.
  - Deploys safely 50× per day without incident.
  - Scales cost-linearly with traffic.
  - Stays within latency budget under deep fan-out.
  - Recovers without human intervention from typical failures.
  - Provides observability for unknown future failure modes.
  - Continues to function during dependency degradation.
  - Is operable by a team that can sleep at night.

That's a much higher bar. The textbook "stateless = horizontal scaling" answer doesn't get you there. The architecture, tooling, runbooks, observability, and operational discipline together get you there.

At MAANG scale, **boring is a feature**. A stateless tier that handles 10M RPS and never pages anyone is the highest engineering accomplishment in this domain. Everything in this document is in service of that goal.

> Twin doc: `statefulSystemsAtMAANGScale.md`. The two together describe the full design space of MAANG-grade serving infrastructure.