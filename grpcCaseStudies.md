# Staff-Level gRPC Case Studies

Realistic scenarios for using gRPC in production systems. Each case study focuses on the three pillars: **scalability**, **availability**, and **fault tolerance**, and calls out the non-obvious design traps.

---

## 1. Inter-service RPC at High Fan-Out (Netflix/Meta pattern)

### Scenario
A `VideoMetadataService` is called by Recommendation, Playback, Search, and Home feed services. Traffic: 100K+ RPS aggregate, P99 SLO < 20 ms. Each client opens calls to ~2000 backend replicas across 3 AZs.

### Design
- **Client-side load balancing** via xDS (Envoy control plane) or gRPC-LB. No L7 proxy hop per call — saves 1–2 ms P99 and halves infra cost.
- **Subsetting (deterministic)**: Each client connects to ~25 backends, not all 2000. Keeps per-process FD count bounded, keeps HTTP/2 connections warm (TCP slow-start + TLS handshake is 3–5 RTTs — you don't want to pay that per request).
- **Long-lived HTTP/2 connections** with multiplexed streams. One TCP connection carries 100+ concurrent RPCs via HTTP/2 streams.
- **Deadline propagation**: Caller's `grpc-timeout` flows via context through every hop. If the edge gave you 200 ms and 150 ms is gone, the DB call gets 50 ms — not its hardcoded 2 s. This is the single most important fault-tolerance feature in a gRPC mesh.
- **Hedged requests** (gRFC A6): fire a duplicate after P95 latency. Trades ~5% extra load to kill tail latency from a single slow replica (GC pause, noisy neighbor).

### Scalability traps
- **MAX_CONCURRENT_STREAMS default is 100** per HTTP/2 connection. At 1000 RPS per client-server pair with 20 ms latency, you need ~20 concurrent streams — fine. At 10K RPS × 50 ms = 500 concurrent streams — you'll queue. Bump to 1000 or open multiple subchannels.
- **Head-of-line blocking at the HTTP/2 layer**: one fat response can stall streams on the same connection. Separate latency-sensitive and batch endpoints onto different services (or at minimum different connections).

### Fault tolerance
- **GOAWAY-aware graceful shutdown**: on deploy, server sends GOAWAY with last-stream-id, then waits for in-flight streams to drain (deadline-bound), then exits. Client picks a new backend for new RPCs without failing in-flight ones.
- **Reconnect with exponential backoff + jitter** (gRFC A6): default is 1s → 120s max. Without jitter you get thundering herd on mass restart.
- **Retry policy** (service config): only retry on `UNAVAILABLE`, `RESOURCE_EXHAUSTED`. Never retry `INVALID_ARGUMENT` (client bug) or `FAILED_PRECONDITION` (business logic). Retry budget: max 10% extra load — beyond that, retries amplify outages.

---

## 2. Geo-Distributed Real-Time Dispatch (Uber pattern)

### Scenario
Driver location updates (50M/sec globally), rider-match must see driver freshness ≤ 3 sec. Drivers hold persistent connections; matching runs per-city.

### Design
- **Client streaming** for driver→dispatcher location pings. One stream per driver session for the entire shift. Avoids TCP slow-start on every ping; syscall + CPU cost is ~10× lower than unary RPC per message.
- **Bidirectional streaming** for dispatch→driver job offer + driver ACK on the same stream. Natural half-duplex interaction, ordered delivery for free.
- **Keepalive tuning**: `GRPC_KEEPALIVE_TIME_MS=30s`, `GRPC_KEEPALIVE_TIMEOUT_MS=10s`. Mobile networks are flaky — without keepalives, dead connections linger (TCP FIN is lost on NAT rebinding) and you send job offers to drivers who left 10 min ago.
- **Regional sharding**: driver sessions sticky to home region. Cross-region only on airport pickups — read-your-writes not required, 3 sec staleness is fine.

### Scalability
- A single gRPC server in Go/C++ handles ~1M concurrent streams per host with tuned kernel (somaxconn, tcp_mem, nofile). Bottleneck is usually memory per stream (~8–16 KB).
- **Don't put driver streams behind a standard L7 proxy** — most terminate HTTP/2 and forward as HTTP/1.1, which kills streaming. Use L4 (TCP) passthrough, or Envoy with HTTP/2 end-to-end.

### Fault tolerance
- **Backpressure on the server-stream**: if dispatcher is slow, `stream.Send()` blocks at the flow-control window (default 64 KB). Don't buffer unbounded in app code — push OOM back to the sender. Use `SendMsg` with deadline and drop oldest-location-wins semantics.
- **At-most-once vs at-least-once**: location pings can be lost (newer ping obsoletes older). Job offers cannot — make them idempotent via `job_id` and retry on stream break.

### Trap: **Load-balanced long-lived streams pin traffic**
Once a stream is established, it stays on one backend for its lifetime. If you scale out, new backends get no traffic until drivers reconnect. Mitigation: server-side **MAX_CONNECTION_AGE** (e.g., 30 min) forces periodic reconnection, which rebalances.

---

## 3. Financial Trading / Order Management (Robinhood, Coinbase pattern)

### Scenario
Order placement API: strict ordering per account, exactly-once semantics, P99 < 5 ms, 99.99% availability. Can't lose an order, can't duplicate one.

### Design
- **Unary RPC** (not streaming) for orders — each order is a discrete transaction, not a stream. Streams complicate recovery after failure.
- **Idempotency keys** in request metadata (`x-idempotency-key`). Server dedupes via Redis/durable store. Client retries safely on `UNAVAILABLE`.
- **Deadline = 2 × expected latency** (10 ms), no retries beyond that. Better to fail fast and let the client decide than to leave the user wondering if the order landed.
- **mTLS with SPIFFE identities** for service authN; per-method authZ via interceptor.

### Availability — the critical bit
- **Active-active across two regions** with a global load balancer. Problem: ordering guarantees per account. Solution: account-affinity routing (hash account_id to region) with failover only on full-region loss.
- **Circuit breaker on the client** (gRFC A40 or app-level): after N consecutive failures, fail fast for M seconds. Prevents a hung dependency from exhausting caller threadpools/goroutines.

### Fault tolerance
- **`OK` vs ambiguous failures**: if the client gets `DEADLINE_EXCEEDED` or `UNAVAILABLE`, did the server process the order? Unknown. Server must make the operation **idempotent** (via the key) so retry is safe. Never retry `OK`-but-slow without idempotency — you'll double-place orders.
- **Write to WAL before ACK**: server persists to Kafka/Raft log before returning `OK`. A process crash between "DB write" and "RPC response" is indistinguishable from network loss; the idempotency key + WAL replay covers both.

### Trap: **Graceful shutdown race**
On deploy, if the server returns `OK` but dies before the TCP ACK reaches the client, the client sees `UNAVAILABLE` and retries. Without an idempotency key, you'd double-order. Always pair graceful shutdown with idempotency — the two are a package deal.

---

## 4. Fan-Out Aggregation Gateway (Search, Feed, GraphQL BFF)

### Scenario
A BFF/gateway fans out a single user request to 30+ downstream gRPC services (user, posts, ads, friends, …) and aggregates. P99 < 100 ms. One slow service shouldn't tank the whole response.

### Design
- **Parallel fan-out with bounded concurrency** — `errgroup` in Go or `CompletableFuture.allOf` in Java.
- **Per-downstream deadline < overall deadline**: if user SLO is 100 ms, downstream deadlines are 80 ms, leaving 20 ms for merging + overhead.
- **Partial results / graceful degradation**: ads service times out? Return the feed without ads, don't fail the whole request. Requires classifying dependencies as **critical** vs **optional** at the gateway layer (usually a config/annotation).

### Scalability
- **Connection pool per downstream**. Don't share a single HTTP/2 connection across 30 services — head-of-line blocking will ruin P99.
- **Response size matters**: gRPC default MAX_RECV_MESSAGE_SIZE is 4 MB. A posts service returning images inline will silently truncate. Set it explicitly or (better) return URLs, not blobs.

### Fault tolerance
- **Per-dependency circuit breaker**: one broken service shouldn't exhaust gateway goroutines. Shed load on that dep while others keep working.
- **Adaptive concurrency** (Netflix concurrency-limits, Vegas algorithm): gateway learns downstream capacity from latency gradient and throttles calls. Better than a static rate limit, which is either too tight or too loose.

### Trap: **Deadline pyramid collapse**
If gateway's 100 ms deadline propagates, and each downstream passes *its* remaining deadline to *its* downstreams, one slow hop starves everything below it. Fix: have each service reserve a minimum budget (e.g., 10 ms) for its own work; if less remains, fail fast with `DEADLINE_EXCEEDED` rather than calling a DB with 2 ms left.

---

## 5. Server-Streaming for Large Result Sets (BigQuery / Data APIs)

### Scenario
Analytical API returns 1M+ rows. Unary RPC would OOM the client and the server (4 MB default limit, and even 100 MB is too much to hold in memory).

### Design
- **Server streaming**: server pages results, client consumes as it reads. Memory bounded by page size × HTTP/2 flow control window.
- **Resumable streams**: include a `resume_token` in each message. On disconnect, client reconnects with the last token; server seeks to that point. Critical for mobile / long queries.
- **Compression** (gRPC supports gzip, zstd per-message): turn on for streams — repeated row structures compress 5–10×.

### Scalability
- Flow control is the subtle part. `SendMsg` blocks when the client hasn't consumed — this is good (backpressure). Don't wrap in a goroutine that buffers; you'll hide the backpressure signal and OOM.

### Fault tolerance
- **Never assume the stream completes**. Network hiccup at row 999,999 must be recoverable — hence resume tokens. Build this in from day one; retrofitting is painful because the server needs to be stateless-with-seekable-source.

### Trap: **Proxy timeouts kill long streams**
Most L7 proxies idle-timeout streams at 60 s or so. If your stream takes 10 minutes, either (a) send keepalive pings every 30 s to mark activity, or (b) configure the proxy stream timeout explicitly. Users will blame "gRPC is broken" when it's actually the proxy.

---

## 6. Service Mesh + gRPC (Istio / Linkerd at scale)

### Scenario
500-service platform migrating from REST+JSON to gRPC. Need observability, retries, mTLS, traffic shifting without touching app code.

### Design
- **Sidecar (Envoy) per pod**: terminates and originates HTTP/2 locally. App talks plaintext gRPC to localhost; sidecar handles mTLS, retries, tracing.
- **xDS for service discovery**: control plane pushes endpoint updates to sidecars. Client-side LB without baking discovery into app.
- **Traffic shifting**: canary 5% → 50% → 100% via weighted xDS routes. Per-header routing for shadow traffic (duplicate to new version, discard response).

### Scalability
- Sidecar adds **0.5–1 ms P99** latency per hop and ~100 MB RAM per pod. For a 500-service call graph that's real money — budget for it.
- **Ambient mesh** (Istio Ambient, Linkerd 2.14+) eliminates per-pod sidecar for L4, reducing overhead dramatically. Worth it at scale.

### Fault tolerance
- **Outlier detection** ejects unhealthy backends from LB pool without needing health endpoints — consecutive-5xx count over a window.
- **Retries at sidecar** centralize policy: one config change re-tunes all services. But — **retries don't compose**. Sidecar retries + app retries = retry-storm amplification. Pick one layer. Usually sidecar for infra-level, app for business-level.

### Trap: **gRPC status vs HTTP status confusion**
Sidecars historically made retry/routing decisions based on HTTP 200 (gRPC always returns HTTP 200 + a `grpc-status` trailer). Without gRPC-aware config, the proxy retries nothing because everything is "successful". Make sure your mesh is configured for gRPC, not generic HTTP.

---

## 7. Protobuf Schema Evolution at Scale

Not a scenario per se, but the #1 source of production incidents in multi-team gRPC deployments.

### Rules
- **Never reuse or renumber field tags**. Tags are the wire format; reusing them silently corrupts data.
- **Never change field types** (except a few compatible pairs: int32↔int64↔bool, etc.). Change = new field.
- **Required is forbidden in proto3**. Even in proto2, don't use it — you can't remove it later without breaking old clients.
- **Enums**: always include `UNKNOWN = 0;` as the zero value. Old clients receiving a new enum value decode as `UNKNOWN` rather than throwing.
- **Rollout order**: deploy server with new field → wait for full rollout → deploy client that reads it. Reverse for removal.

### Tooling
- **Buf breaking-change detector** in CI. Blocks PRs that break wire/source compat.
- **Proto registry** (BSR, Prism, or homegrown): single source of truth, versioned. Don't vendor .protos into 50 repos — you'll have 50 copies drift.

### Trap: **Field presence in proto3**
Before `optional` in proto3 (2020), you couldn't distinguish "unset" from "default value" for scalars. An API that accepted `set_volume(level=0)` couldn't tell "mute" from "not provided". Use `optional` keyword for proto3 scalars where presence matters, or wrap in `google.protobuf.Int32Value`.

---

## Quick Decision Reference

| Problem | Reach for |
|---|---|
| Low-latency RPC in a mesh | Unary + client-side LB + deadline propagation |
| Persistent device connections | Server or bidi streaming + keepalives + MAX_CONNECTION_AGE |
| Exactly-once writes | Unary + idempotency key + WAL |
| Fan-out aggregation | Parallel unary + per-dep deadline + circuit breaker |
| Large result set | Server streaming + resume token + compression |
| Multi-region active-active | Affinity routing, NOT global consistency via gRPC |
| Platform-wide observability | Sidecar mesh, but budget for latency + memory |

---

## The Three Things Staff Engineers Get Right

1. **Deadlines propagate or you have no fault isolation.** A system without end-to-end deadline propagation will experience retry storms under partial failure. This is non-negotiable.
2. **Idempotency is a property of the operation, not the transport.** gRPC gives you the hooks (metadata, status codes, retry policies), but the server has to actually make operations idempotent. Retry without idempotency = corruption.
3. **Streaming is a contract, not just an optimization.** Once you pick streaming, you own flow control, resumability, and LB rebalancing. Don't stream because it's "cooler" — stream because the data model demands it.