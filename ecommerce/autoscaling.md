# Autoscaling an E-Commerce System — Deep Dive

> Match capacity to demand, while protecting the things that can't scale.

Autoscaling is usually framed as "add pods when CPU is high." That framing is what breaks systems at Black Friday. 
The correct frame: **autoscaling is a control loop that must scale the elastic layers without overwhelming the inelastic ones (database, payment gateways, third-party APIs, warehouse ERP).** 
Ninety percent of autoscaling incidents I have seen are not "we didn't scale fast enough" — they are "we scaled too fast and melted something downstream."

This doc covers the ecommerce stack top-to-bottom (CDN → front-end → app services → workers → caches → message brokers → search), per-component signals and mechanisms, and the cross-cutting concerns (cold start, connection pools, rate limits, predictive scaling, graceful shutdown). 
The database layer is covered only at the integration boundary; see `databaseTransactions.md`, `hotPartition.md`, and `stalePricingCache.md` for in-depth treatments.

---

## 1. The Framing: Elastic vs Inelastic Components

Every ecommerce system has two classes of resource.

**Elastic (scale horizontally in seconds to minutes):**
- Stateless app pods (product, cart, checkout, search gateway, BFF)
- Workers consuming from queues (outbox publishers, email workers, webhook dispatchers)
- Cache reads (Redis clusters with replica scale-out)
- CDN edge capacity (managed; scales transparently)
- Object storage (S3 / GCS)

**Inelastic (fixed capacity, or scale in hours/days):**
- Primary relational database (OLTP writes)
- External APIs you don't control (Stripe, PayPal, tax engines, shipping APIs, marketing providers)
- Warehouse / ERP systems (often mainframe or SaaS with strict rate limits)
- Internal rate-limited services (fraud, risk scoring)
- Humans in the loop (customer support, manual review queues)

The control loop's job: scale elastic layers so they *fully absorb* load spikes, while the pressure on inelastic layers stays below their hard limits. "Scale the pods to keep the DB safe" — not "scale the pods because CPU is high."

If you scale pods to the point where DB connections saturate, you've turned a user-facing slowdown into a database-wide outage. 
**The correct action in that situation is to *stop scaling up* or even to scale *down* and shed load at the edge. Naive HPA does the opposite.**

---

## 2. The Scaling Signals — Leading vs Lagging

CPU and memory are **lagging** indicators. By the time CPU hits 80%, requests have already queued and latency has already degraded. Scaling on CPU gets you a **reactive loop** that's permanently behind the curve.

The staff-level signal ordering:

```
LEADING                                                            LAGGING
─────────────────────────────────────────────────────────────────────────▶
Queue depth   →   In-flight requests   →   p99 latency   →   CPU / memory
(seconds      (~50ms ahead)                (hundreds of     (minutes behind)
 ahead)                                     ms behind)
```

**Queue depth (best for workers)**: the number of unprocessed messages in Kafka/SQS/Redis Streams. Scales based on "how much work is waiting," not "how hard are we struggling right now." This is the KEDA pattern — pod count scales linearly with backlog.

**In-flight requests (best for sync services)**: active requests per pod. When in-flight > concurrency target, scale up. Captures both "high RPS" and "slow downstream" without needing separate signals.

**p99 latency**: user-visible symptom. Scale up when p99 exceeds SLO; scale down when p99 is well under. Best paired with in-flight as a tripwire.

**RPS (requests per second)**: OK for stable, predictable workloads. Bad for workloads with variable per-request cost (e.g., search requests range from 5ms to 500ms depending on query).

**CPU**: still useful as a *correlated* signal, not the primary one. If CPU is high and the service is IO-bound, you've confirmed something is wrong but don't know what.

Use composite HPA (Kubernetes v2 supports multiple metrics): scale up on the max of (in-flight > X OR queue depth > Y OR CPU > 70%); scale down only when *all* are low. 
Asymmetry matters — be quick to scale up, slow to scale down.

---

## 3. Scaling Mechanisms

### 3.1 Horizontal Pod Autoscaler (HPA)

Kubernetes-native. Polls metrics every 15s; adjusts replica count. Standard for stateless services.

Key config:
- `behavior.scaleUp`: how aggressive (`stabilizationWindowSeconds: 0`, add +100% of current replicas per 30s for fast spikes).
- `behavior.scaleDown`: how conservative (`stabilizationWindowSeconds: 300`, remove 10% per minute to avoid flapping).
- `minReplicas`: never go below this; protects against failure modes that kill all pods simultaneously.
- `maxReplicas`: upper bound; protects downstream (DB connections, rate-limited APIs).

### 3.2 Vertical Pod Autoscaler (VPA)

Adjusts CPU/memory *requests* based on observed usage. Useful for right-sizing during normal load, not for spike response (VPA restarts pods to apply new sizing). 
**Don't use VPA for checkout-path services at peak — pod restarts drop in-flight requests.**

Common pattern: VPA in `recommendationOnly` mode during development; switch to `auto` for non-critical backends; leave off for critical paths.

### 3.3 KEDA — event-driven autoscaling

Scales pods based on external event sources: Kafka lag, RabbitMQ queue depth, SQS messages, Redis Streams, Prometheus metrics, even arbitrary HTTP endpoints. 
The single most useful autoscaler for worker pools.

Example: outbox publisher
```yaml
# KEDA ScaledObject: scale outbox publisher based on Kafka consumer lag
triggers:
- type: kafka
  metadata:
    bootstrapServers: kafka:9092
    consumerGroup: outbox-publisher
    topic: orders.events
    lagThreshold: "100"           # +1 pod per 100 unprocessed messages
```

KEDA can scale pods from 0 (for batch jobs) to thousands. For event-driven workers, it replaces HPA entirely.

### 3.4 Cluster Autoscaler

Adds/removes *nodes* from the cluster when pods can't schedule (or nodes are idle). Works together with HPA/KEDA: HPA wants 500 replicas, cluster has room for 200; 
cluster autoscaler spins up more nodes. Node provisioning is slow (30s-3min for a new EC2 instance), which is why you over-provision nodes during peaks.

Pair with **Karpenter** (AWS) or **GKE's node auto-provisioning** for faster node scaling and better instance-type selection.

### 3.5 Predictive / scheduled scaling

The single biggest win for ecommerce: **pre-scale before known peaks.**

Examples:
- Black Friday week → scale up to 3x steady-state capacity starting Thursday midnight.
- Daily traffic curve → scale up at 8am local, down at 2am local.
- Product launch / flash sale → scale to launch-spec 30 min before announced start.

Mechanisms: scheduled HPA (Kubernetes CronHPA), AWS predictive scaling (ML-based), or just `kubectl scale` in a Jenkins/GitHub-Action-driven playbook.

Why predictive beats reactive: reactive autoscaling has a latency tail. Pod startup + readiness probe + JVM warmup + cache warmup can take 2-5 minutes. In that window, traffic is being served by under-capacity fleet. Predictive avoids the latency entirely — capacity is already there when the spike hits.

### 3.6 Serverless (Lambda, Cloud Run, Fargate)

Scales to zero; scales up in ~100ms. Right for sporadic workloads (webhooks, image transformations, occasional report generators). 
Wrong for steady high-throughput (cost dominates; cold starts bite). Rule of thumb: <100 RPS or bursty workload → serverless; >1000 RPS steady → containers on Kubernetes.

---

## 4. Layer-by-Layer Scaling Strategies

### 4.1 CDN / Edge Layer

**Scaling profile:** transparent. Managed service (Cloudflare, Fastly, CloudFront, Akamai) handles it.

**What you control:**
- **Cache hit ratio.** A 98% cache hit rate means origin sees 2% of traffic. At 98→95 (seemingly small drop), origin load tripled. Monitor and alert on cache hit ratio.
- **Purge rate.** Bulk purges on price changes can rate-limit you. **Batch or use surrogate keys.**
- **Origin shield** (a regional middle cache between CDN edges and origin). Absorbs long-tail misses. Always on for high-cardinality catalogs.
- **Rate limiting at the edge.** Block bots, limit per-IP request rates. A DDoS hitting the edge is cheaper to drop there than at your origin.

**Staff-level move:** invest in edge-side composition (Cloudflare Workers, Fastly Compute@Edge) for pages that mix static + dynamic. The product page shell is cached; price and stock are fetched live from origin. Origin load scales with *dynamic* content volume, not total traffic. This changes origin capacity requirements by 10-100x.

### 4.2 Load Balancer / API Gateway

**Scaling profile:** managed (AWS ALB/NLB, GCP Cloud LB). AWS ALB is elastic but takes ~15 min to fully scale. Pre-warm with AWS support for known peaks, or use NLB (scales instantly but TCP-only).

**What to tune:**
- **Connection concurrency limits per target.** ALB defaults to unlimited; you want it bounded so one pod doesn't get slammed with 10x others during deregistration.
- **Health check timing.** Aggressive health checks (2s interval, 2 failures = unhealthy) detect dead pods fast but can false-positive under load. Tune to your startup time.
- **Connection draining / deregistration delay.** When scaling down, the pod must finish in-flight requests before terminating. Set deregistration delay > p99 request duration (typically 30s for checkout, 5s for browse).
- **Idle connection timeout.** Close idle connections so pods can be removed. ALB default is 60s; usually fine.

**API Gateway** (Kong, Envoy, AWS API Gateway) typically scales on its own managed infrastructure. The gateway's own capacity is rarely the bottleneck; its rate-limiting configuration is what protects downstream.

### 4.3 BFF / Frontend Servers

Server-rendered or SSR React/Next.js. Memory-heavy (each request builds a DOM), latency-sensitive.

**Scaling profile:**
- Stateless → HPA-friendly.
- Metric: in-flight requests per pod + p99 latency.
- Cold start: Node.js ~2-3s, Next.js with SSR ~5-10s, Java/Spring ~30-60s, Go ~100ms.
- Startup probe delay must cover cold start.

**Traps:**
- SSR fetches from backend APIs. A slow backend makes BFF pods look slow and triggers HPA to add more — but the new pods are equally blocked on the same backend. Scaling BFF doesn't fix downstream latency. Monitor by request dependency, not just pod-level.
- Memory leaks show up here first (long-lived Node processes). Rolling restart every N hours as hygiene.

**Strategy:**
- Autoscale aggressively (30s scale-up window, +50% replicas per step).
- Pre-scale for known peaks — BFF cold start is minutes, too slow to react.
- Use static generation (SSG / ISR) wherever you can. A statically-generated product page doesn't need a BFF pod at render time; only the dynamic-price fetch does.

### 4.4 Catalog / Product Service

**Scaling profile:** read-heavy, cache-heavy. 95%+ of reads should hit Redis or CDN.

**Autoscaling metrics:**
- **RPS** and **cache miss rate** jointly. A spike in misses means either cache is cold (warm it) or the working set grew (add cache capacity, not service capacity).
- **Downstream DB connection utilization.** Scale up the service only if cache miss + DB response time are both acceptable. If DB is at 80% CPU, stop scaling the service — you'll take the DB down.

**Pattern:**
- Tier 1: **L1 cache in-process (Caffeine/ristretto)**, TTL 10-60s, for ultra-hot SKUs.
- Tier 2: Redis cluster.
- Tier 3: Read replica (fallback).
- Tier 4: Primary DB (last resort).

Autoscale the service to handle the L2 miss rate × L2 hit latency budget. Don't size it for "what if Redis is down."

### 4.5 Search Service

**Scaling profile:** CPU- and IO-intensive (Elasticsearch/OpenSearch/Solr). Each query hits multiple shards; relevance scoring is expensive.

**Autoscaling pattern:**
- Query-coordinator layer (scales horizontally, stateless).
- Data nodes (stateful; scale with care — adding a node triggers shard rebalancing that costs IO).
- Add **pre-aggregated caches for common facets at the coordinator layer.**

**Staff-level specifics:**
- **Slow queries burn the cluster.** Enable slow-query log; **cancel queries past 1s**. One runaway wildcard query can eat entire cluster capacity.
- **Isolate workloads.** Browse search vs. admin search vs. batch indexing share the cluster and starve each other. Separate indices or separate clusters.
- **Read-through cache** for common queries. Redis-cache the top N search results with short TTL; 80% of queries are 20% of the query space.

Don't scale Elasticsearch by HPA on CPU. Scale by query latency SLO (p99 < 100ms for product search) and cluster queue depth (`thread_pool.search.queue`).

### 4.6 Cart Service

**Scaling profile:** write-heavy (every add/remove), but writes are cheap per operation.

**State:** session-like; often in Redis or DynamoDB rather than RDB. See `cartUpdates.md`.

**Autoscaling:**
- Stateless service pods → HPA on in-flight requests.
- Redis cluster behind → add shards as data grows, but this is a manual-ish operation.
- Per-user locality: sticky sessions help keep a user's cart hits hitting the same pod for local caching. Consistent hashing at the LB routes `user_id → pod`.

Cart is usually *not* the bottleneck at peak. It's fast, stateless, and scales horizontally cleanly. Budget less here than you think.

### 4.7 Checkout Service

**Scaling profile:** the high-stakes, low-RPS layer. Checkout RPS is typically 1-5% of browse RPS, but each request is slow (100ms-2s) and hits many dependencies (inventory, pricing, payment pre-auth, fraud, tax, shipping).

**Autoscaling:**
- Scale on **in-flight requests per pod**, not RPS. One request holding 4 downstream calls open counts as "one in-flight" but ties up a thread/goroutine for its full duration.
- Generous `maxReplicas`, but bounded by **downstream capacity** — especially payment gateway rate limits (Stripe **typically allows 100 req/sec per account**; Braintree varies by agreement). Scaling checkout to 1000 pods each doing 1 req/sec is fine. Scaling to 1000 pods each doing 10 req/sec will get you throttled.
- Pre-scale aggressively before peak. Checkout path pods should start the Black Friday week at 5-10x normal.

**Bulkheads:**
- **Thread-pool isolation** per downstream. A slow tax API should not exhaust all threads and block payment calls. Use **bulkhead patterns (Hystrix-style / resilience4j) with per-dependency thread pools.**
- **Circuit breakers** on each downstream. If payment gateway is slow, fail fast; don't let requests pile up.

**Back-pressure:**
- **If checkout is at 80% capacity and rising, queue new checkouts behind a virtual waiting room. Better UX than timing out.**

### 4.8 Payment Service

**Scaling profile:** nearly 1:1 with checkout RPS; each checkout calls payment 1-3 times (auth → capture → maybe refund).

**Autoscaling:**
- Similar to checkout: in-flight requests as primary metric.
- **Hard-capped by gateway limits.** Do not scale above gateway rate limits; it's a DDoS on your own provider.
- Route between multiple gateways when one is throttled. Fallback logic built into the service, not just at the edge.

### 4.9 Order Service

**Scaling profile:** write once, read many. Order creation is transactional (one commit per order). Order reads are hot (customers refresh the order page).

**Write path autoscaling:**
- Bounded by DB write capacity. No benefit to scaling past that.
- Use a Kafka buffer in front of order inserts if the write rate can spike past DB capacity (see `hotPartition.md §3.3`).
- Workers consuming from Kafka scale on lag, not CPU.

**Read path autoscaling:**
- Stateless → HPA.
- Heavy caching (order detail page cached per-user for 5-10 minutes, invalidated on status change via pub-sub).

### 4.10 Inventory Service

**Scaling profile:** read-heavy (every product page checks stock), write-hot (every order decrements stock on hot SKUs).

**Read scaling:**
- Redis DECR / HSCAN; 100K+ ops/sec per Redis primary. Scale by adding shards or replicas.
- Service pods are thin wrappers; stateless → HPA.

**Write scaling:**
- Hot SKU contention is the bottleneck. Solution is *architectural* (sharded inventory, event-sourced counters) — see `variantExplosion.md §4`.
- Autoscaling the inventory *service* doesn't help if one Redis row is saturated.

### 4.11 Notification Service (Email/SMS/Push)

**Scaling profile:** asynchronous, batch-friendly, high throughput, external-rate-limited.

**Pattern:**
- Events arrive on Kafka. KEDA scales consumers on lag.
- Consumers bulk-send to provider (SendGrid, Twilio, AWS SNS) respecting provider rate limits.
- Cap concurrency at provider limit; overflow stays in Kafka backlog (acceptable — email can be delayed minutes).

**Staff-level move:** different queues per priority (transactional order-confirmation vs. marketing blasts). Transactional queue scales aggressively and preempts marketing. Never let a marketing blast delay password resets.

### 4.12 Recommendation / ML Inference

**Scaling profile:** CPU/GPU-heavy, latency-sensitive (<50ms for inline recs).

**Autoscaling:**
- **GPU pods** scale in minutes, not seconds (container pull is large). Pre-scale for known peaks.
- **Graceful degradation**: if recs service is slow or down, the page still renders without recs. Circuit-break, don't block.
- **Model serving** (TensorFlow Serving, TorchServe, Triton): scales via standard HPA + custom metrics (inference queue depth).

**Offline path:** batch rec regeneration jobs scale with data volume. Run on spot / preemptible nodes; cheap.

### 4.13 User / Auth Service

**Scaling profile:** login spikes with traffic; reads heavier than writes.

**Autoscaling:**
- Token validation is pure CPU, stateless → HPA straightforward.
- JWT validation can be offloaded to the edge (Cloudflare Workers, Envoy authz filter) so the auth service isn't in the critical path for every request.
- Session store (Redis) scales by sharding on user_id.

### 4.14 Pricing / Promotion Service

See `stalePricingCache.md` and `promotionTimezone.md`. Autoscale cacheable reads with HPA; behind a CDN for public pricing queries. Write path (price changes, promo creation) is low-volume and doesn't need autoscaling.

---

## 5. Worker Fleets and Stream Processors

Workers handle asynchronous processing: outbox publishing, order state transitions, warehouse notification, fraud re-check, email, analytics events. They're the single biggest win for autoscaling because they're:
- Stateless
- Metric-clear (queue depth)
- Batch-friendly (one worker can process many messages)
- Tolerant of restarts

**Pattern for every worker:**
```
KEDA ScaledObject (trigger: queue lag) → 0..N pods
Each pod: consume batch → process → commit offset → repeat
```

**Scaling knobs:**
- **Pod count** scales with lag.
- **Batch size** scales with per-message cost. Large batches → fewer DB round-trips but higher latency per message.
- **Parallelism within the pod** (goroutines / threads) should be tuned to downstream capacity.

**Shutdown:**
- Graceful shutdown is critical. On SIGTERM, stop consuming new messages; finish in-flight; commit offsets; exit. Otherwise KEDA/HPA scale-in drops messages.
- Use Kafka `pause()` / commit-then-exit. Set `terminationGracePeriodSeconds` > max message processing time.

**Flink / Spark streaming:**
- Long-running streaming jobs don't autoscale well (stateful, checkpoint boundaries). Scale by savepoint-and-restart with higher parallelism, or use Flink's reactive mode (v1.13+). Plan scaling as a deployment, not a reactive metric.

---

## 6. Cache Layer: Redis, Memcached, L2 Caches

**Scaling profile:**
- Redis/Memcached: throughput scales per-shard; memory scales per-shard. Add shards for both.
- Resharding is disruptive (data moves). Design for it from day one (consistent hashing with virtual nodes; managed services handle it for you).

**Autoscaling:**
- Typically *not* autoscaled on the fly. Capacity is planned in advance.
- Managed Redis (ElastiCache, MemoryDB, Upstash): can add replicas for read scale-out within minutes. Primary scaling is slower.
- Monitor `maxmemory-policy` evictions. A spike means working set exceeded capacity → add shards.

**The real cache-scaling lever:** hit ratio. Doubling cache size may raise hit rate from 95% to 97% (2% absolute, but the miss rate halved, halving origin load). Measure miss rate, not cache size.

---

## 7. Message Brokers

Kafka/Pulsar/RabbitMQ underpin every decoupled flow. They're the buffer that prevents hot-partition-in-orders from collapsing under a spike.

**Scaling:**
- **Topic partitions = maximum consumer parallelism.** If Kafka topic has 10 partitions, max 10 consumer pods per consumer group productively consume it. More pods just idle.
- **Pre-partition for peak.** Changing partition count is operationally painful (rebalances, key routing). Create topics with 2-5x current partition count so you can scale consumers later without reconfiguring the topic.
- **Broker-level scaling** is manual (add brokers, rebalance partitions). Plan capacity; don't expect reactive scaling.

**Consumer autoscaling with KEDA:**
```
lag = sum(current_offset - committed_offset) across partitions
pods = ceil(lag / target_lag_per_pod)
```
Practical values: target_lag_per_pod = 1000-10000 messages. Tune so scaling stabilizes before overshooting.

**Back-pressure:** when producers outpace consumers for too long, Kafka disk fills. Monitor `log_dir_used_bytes`; alert at 70%. Add brokers or scale consumers before disk full.

---

## 8. The Database (Brief)

The elastic layers above scale freely; the DB does not. Autoscaling the app without respecting DB limits is how you cause outages.

**Protections:**
- **Bounded `maxReplicas` on app HPA** such that `maxReplicas × connections_per_replica ≤ DB max_connections × 0.7`. Leave 30% headroom for admin, migrations, replication.
- **pgbouncer (or RDS Proxy)** between app and DB. Multiplexes connections; a 10K-pod fleet can share 500 DB connections via transaction-mode pooling. Without this, `max_connections` is a hard wall.
- **Read replicas** for read scaling. Autoscaling-friendly; you can add replicas online. Caveat: replication lag + read-your-writes semantics.
- **Write path:** DB write capacity is largely fixed. If you can't shard (see `hotPartition.md`), you can't horizontally scale writes. Queue excess load in Kafka and drain at DB-friendly rate.

**The rule:** the app HPA's `maxReplicas` is set by the DB, not by CPU ceilings. Increasing `maxReplicas` without a DB-capacity review is a lurking outage.

Further detail in `databaseTransactions.md §3` and `hotPartition.md`.

---

## 9. Cross-Cutting Concerns

### 9.1 Cold start

Scale-up latency is dominated by cold start: pull image → start container → JVM warm-up → JIT → fill caches → ready. Times:

| Stack                       | Cold start     |
|-----------------------------|----------------|
| Go / Rust static binary     | 100-500ms      |
| Node.js                     | 1-3s           |
| Next.js SSR                 | 3-10s          |
| Python (Django)             | 3-8s           |
| Java/Spring Boot            | 20-60s         |
| Java + JIT warm-up          | +30-120s for steady p99 |
| ML model inference pod      | 30s-3min (model load) |

Mitigations:
- **Pre-pull images** (DaemonSet, node pre-warming).
- **Smaller images** (distroless, slim bases).
- **Readiness != warmed up.** Add a synthetic warm-up step: hit 50 representative requests before declaring ready.
- **JVM AOT compilation** (GraalVM native-image) for services where fast cold start matters.

If cold start is 60s and you need to scale up 5× in 30s, HPA cannot save you. **Pre-scale for peaks.**

### 9.2 Startup probes vs. readiness vs. liveness

- **Startup probe:** fires once, allows long initial warm-up (up to minutes) without liveness killing the pod. Use it for anything with a >30s startup.
- **Readiness probe:** determines whether LB sends traffic. Runs continuously. Fail readiness if a downstream is broken and you can't serve requests; do *not* fail liveness (LB stops sending, but the pod keeps running; once downstream recovers, readiness passes).
- **Liveness probe:** only fails when the process is genuinely broken (deadlocked, unresponsive). Causes restart. Too aggressive → restart loop; too lax → dead pods keep traffic.

**Common bug:** liveness probe calls the DB. DB has a hiccup. Every pod gets its liveness fail and restarts. Cascading outage. **Liveness probes must only check the pod's own health, not dependencies.**

### 9.3 Graceful shutdown

On SIGTERM:
1. Remove from LB (readiness → false).
2. Drain connections (close keepalive; finish in-flight).
3. Flush queues / commit offsets.
4. Exit.

Set `terminationGracePeriodSeconds` ≥ max request duration + buffer. For checkout, 60s. For browse, 10s. If a pod exits with in-flight work, some user sees a 5xx.

### 9.4 Connection pool sizing

Each pod has a connection pool to DB, Redis, Kafka, external APIs. Scale-out multiplies connection count:
- 100 pods × 20 DB connections = 2000 total (DB max_connections must absorb).
- At autoscaled peak (1000 pods), 20000 connections → DB meltdown if not pooled via pgbouncer.

**Rule:** app-side pool size = DB_max_connections × safety_factor / max_pod_count. Safety factor 0.7. Know these numbers before enabling autoscaling.

### 9.5 Third-party rate limits

Most external services have hard rate limits:
- Stripe: ~100 req/sec (varies by account tier)
- SendGrid: 100 req/sec per subuser
- Twilio: 1 req/sec per messaging service by default
- Google Maps: 50 req/sec baseline

Scaling your service past these limits gets you 429s, which your clients interpret as errors. Implement token-bucket rate limiters at the egress layer (not per-pod — per-service, coordinated via Redis or a dedicated rate-limit service). Your autoscaler should respect these ceilings.

### 9.6 Stateful services

Redis, Elasticsearch, Kafka brokers, databases — don't autoscale these reactively. Capacity plan; scale manually in maintenance windows; use managed services that handle scale operations (rebalancing, replication) gracefully. Reactive autoscaling of a stateful service causes the thing autoscaling is meant to prevent: correlated failure during load.

---

## 10. The Pre-Peak Playbook

For Black Friday, product launches, or marketing pushes:

**T-30 days:**
- Forecast peak RPS per service based on historical spikes + marketing forecast.
- Capacity review: current HPA maxReplicas, DB capacity, cache capacity, third-party rate limits. Find the weakest link.
- Load test at 2x forecast peak. Fix what breaks.

**T-14 days:**
- Code freeze for checkout path. No non-essential changes.
- Game day: fail an AZ, fail a cache, fail a downstream. Confirm graceful degradation./ Chaos engineering.
- Pre-warm cache by hitting top N product pages.

**T-7 days:**
- Bump HPA `minReplicas` to 2-3x normal for critical services.
- Pre-provision nodes (Karpenter reservations; AWS ODCR).
- Verify runbooks for incident scenarios.

**T-1 day:**
- Scale critical services to 80% of peak capacity. No reactive scaling during event.
- Disable non-essential batch jobs that compete for DB.
- Verify on-call rotation.

**During event:**
- Watch leading indicators (queue depth, in-flight).
- Have a shed-load switch: if DB is at 95%, enable **waiting room** / disable non-essential features.
- Don't autoscale based on CPU alone; trust the composite signals.

**Post-event:**
- Scale down slowly (over hours) to avoid flapping.
- Post-mortem; update capacity models.

---

## 11. Failure Modes: Autoscaling That Makes Things Worse

**1. OOM spiral.** Pod OOMs → Kubernetes restarts → other pods absorb the load → they OOM → cascade. Fix: right-size memory; set requests = limits for memory; use pod disruption budgets.

**2. Cold-start pile-up.** Traffic spike → HPA scales up → new pods cold-start for 60s → during which existing pods are overloaded → they time out → HPA sees more lag and scales more → eventually too many pods for the DB. Fix: pre-scale; use faster-starting stacks; implement startup throttling.

**3. Flapping.** HPA adds pods, latency drops, HPA removes pods, latency rises, HPA adds pods. Fix: `stabilizationWindowSeconds` on scale-down (300s+); hysteresis between scale-up and scale-down thresholds.

**4. Downstream overload.** Service scales to 1000 pods; DB saturates; every pod's latency goes up; HPA wants to scale more. Fix: `maxReplicas` bounded by DB capacity; circuit breakers.

**5. Retry storms.** Downstream slow → clients retry → effective load doubles or triples → autoscaler reacts to the amplified load → scale-up puts more pressure on downstream. Fix: exponential backoff with jitter; token-bucket retry budgets.

**6. Scaling the wrong layer.** CPU is high on the BFF; HPA scales BFF; but the root cause is a slow backend. Adding BFF pods doesn't help. Fix: observability at the layer granularity; diagnose before scaling.

**7. Sudden scale-down during draining.** Cluster autoscaler removes a node; pods on that node get 30s to drain; they hold 10-minute checkout sessions. Sessions break. Fix: `terminationGracePeriodSeconds` aligned with max session length; pod disruption budgets; sticky sessions that tolerate node changes.

---

## 12. Observability for Autoscaling

What to dashboard:

**Per-service:**
- Current replicas, desired replicas, max replicas. Gap between current and desired is scaling latency.
- Events/sec processed (for workers).
- Queue depth over time (leading indicator).
- p99 latency (SLO adherence).
- Scale events (count, direction, duration).

**Per-cluster:**
- Node count vs capacity utilization.
- Pending pods (pods waiting for node capacity).
- Resource waste (requested vs used).

**Per-dependency:**
- DB connection utilization.
- Cache hit ratio.
- Third-party rate-limit consumption.

**Alert on:**
- Scale events that *don't complete* (stuck at desiredReplicas < currentReplicas for >5min: node capacity, quota, or crash loop).
- `maxReplicas` reached for >1 minute (the fleet is capped; something is under-provisioned).
- Scale-down stops (hysteresis too aggressive; flapping).
- Queue depth climbing despite pods at max.

---

## 13. Cost and Capacity Tradeoffs

Autoscaling is also cost optimization. The knobs:

- **Right-sizing requests.** Over-requested pods waste capacity; under-requested get OOMKilled. VPA recommendations guide this.
- **Spot / preemptible nodes** for stateless workloads (workers, batch jobs, stateless APIs with multiple AZs). 60-90% cost savings, with the cost of interruption (30s-2min). Good for workers; bad for checkout.
- **Scheduled scale-down.** Off-peak hours (2am-6am local) don't need daytime capacity. Scheduled HPA → save 30-50% on cost.
- **Bin-packing.** Multi-tenant node pools; Karpenter auto-consolidation.

The math: for a steady 100 RPS service, 10 pods of 10 RPS each cost less in nodes than 20 pods of 5 RPS each (provisioning overhead). Bigger pods, fewer of them, bin-pack better — until you hit the "blast radius of a single pod failure" concern. Rule of thumb: never run fewer than 3 replicas of a critical service (2 can fail during a rolling update; one left is a single point of failure).

---

## 14. The Staff-Level Summary

1. **Autoscaling is a protection system for inelastic resources, not just a capacity-matching tool.** Set ceilings that protect the DB, rate-limited APIs, and stateful services. Don't scale elastic layers past what they can safely feed.
2. **Scale on leading indicators (queue depth, in-flight requests), not lagging ones (CPU).** The lag between "load arrives" and "pods ready" must be less than the lag between "load arrives" and "metric fires."
3. **Pre-scale for known peaks.** Reactive scaling is minutes late; predictive scaling is zero late. Every major ecommerce player pre-scales for Black Friday; the ones who don't have bad press.
4. **Graceful shutdown, generous probes, right-sized connection pools — the boring things.** These prevent the failure modes that look like autoscaling bugs but are actually lifecycle bugs.
5. **Different layers need different scalers.** HPA for stateless. KEDA for event-driven workers. Predictive/scheduled for peaks. Manual for stateful. Don't use one tool for all problems.
6. **The database is the ceiling.** Every autoscaling decision in the app layer must respect it. When DB is saturated, scale *down* the elastic layer (shed load at the edge) — not up.
7. **Observe at the layer granularity.** "The service is slow" → which layer? Scaling the wrong one makes it worse.

The corner-case "Hot partition" is the DB layer failing to scale. Everything above the DB *can* scale — as long as you design the control loop to keep the whole system inside its operating envelope, not just keep each layer inside its own.