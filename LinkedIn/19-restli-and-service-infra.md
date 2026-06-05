# 19 · Rest.li, D2, and Service Infrastructure

How LinkedIn's services talk to each other. Crucial backgrounder for any architectural discussion — if you say "the service calls the other service over HTTP" without specificity, you're a senior, not a staff. Use the LinkedIn-internal vocabulary.

## 19.1 Rest.li — the primary RPC framework

> A REST framework on the JVM with strongly-typed schemas, code generation for clients in multiple languages, async-first request-handling, and an "Action / Finder / Get / Create / Update / Delete" resource model richer than plain REST.

Open-sourced. Mature. Used by ~all internal Java/Scala services at LinkedIn for RPC.

### Why Rest.li (over plain REST or gRPC)

- **Schema enforcement**: PDL (or older PDSC) schemas → guaranteed contract. Less prone to silent breakage.
- **Code generation**: client stubs and server scaffolding generated from schema.
- **Rich resource model**: GET (single), GET_ALL, FINDER (parameterized search), ACTION (RPC-like), BATCH_GET, etc. — supports realistic API shapes.
- **Pagination**: built into the contract.
- **Async support**: ParSeq + CompletionStage idioms.

### Resource model

A Rest.li resource is a class:

```java
@RestLiCollection(name = "members", namespace = "com.linkedin.member")
public class MemberResource extends CollectionResourceTemplate<Long, Member> {
  @Override
  public Member get(Long key) { /* ... */ }
  @Finder("byCompany")
  public List<Member> byCompany(@QueryParam("companyId") long companyId) { /* ... */ }
  @Action(name = "endorse")
  public void endorse(@ActionParam("skill") String skill, @ActionParam("endorsee") long endorsee) { /* ... */ }
}
```

### PDL — the schema language

Successor to PDSC. Looks like:

```
namespace com.linkedin.member

record Member {
  id: long
  firstName: string
  lastName: string
  industry: optional Industry
  connectionCount: int = 0
}
```

- Strong types; primitives + records + arrays + maps + enums + typerefs.
- Versioning: backward-compatibility rules enforced at compile time.
- A central schema repo tracks all schemas; CI prevents incompatible changes.

### Client generation

For each service, a client jar is published. Other services depend on the client jar and write code like:

```java
GetRequest<Member> req = new MembersGetRequestBuilder()
    .id(123L)
    .build();
Response<Member> resp = client.sendRequest(req).get();
```

No string-typed URL handling; no JSON-by-string-key parsing. Compile-time safety.

## 19.2 D2 — service discovery + load balancing

> A client-side discovery + load-balancing framework. Services register their endpoints with ZooKeeper; clients subscribe to changes; D2 routes requests to healthy endpoints with adaptive load-balancing.

### Architecture

```
   Service instances → register endpoints in ZooKeeper
                            │
                            ▼
                      ┌───────────┐
                      │ ZooKeeper │
                      └─────┬─────┘
                            │ watches
                            ▼
                      ┌───────────┐
                      │ Client    │ ◀── application makes calls
                      │ (D2 lib)  │
                      └─────┬─────┘
                            │ chosen endpoint
                            ▼
                       Target service
```

Each service has a service-name (e.g., `members-service`). Clients call `client.get("d2://members-service/...")`; D2 resolves to a healthy endpoint.

### Degrader-based load balancing

Each client maintains per-endpoint health stats:
- Avg latency
- Error rate
- Request count

Endpoints with worsening stats are progressively "degraded" — assigned less traffic. Recovered endpoints ramp back up. This was an early outlier-detection algorithm, pre-dating modern service meshes (Envoy outlier detection, Linkerd) but conceptually the same.

### Sticky sessions

Optional: D2 can pin a client's requests to a particular endpoint for a session, supporting use cases like cache-locality.

### Cross-DC

D2 supports cross-DC routing with preferences (prefer-local) and failover.

## 19.3 The service-mesh discussion

Modern service-mesh frameworks (Linkerd, Istio/Envoy, Consul Connect) overlap with D2 in concept:
- Service discovery.
- Load balancing.
- Retries, timeouts, circuit breaking.
- mTLS.
- Observability.

D2 predates them; LinkedIn historically built its own. As LinkedIn migrates to Kubernetes and adopts cloud-native patterns, **Envoy-based service meshes** appear in some newer environments, while D2 still dominates the long-tail of Rest.li services.

A staff candidate might be asked:

> **"Would you migrate from D2 to Envoy/Istio today?"**

Real answer: it depends. D2 has years of operational maturity. Envoy gives broader ecosystem and HTTP/2/gRPC affinity. The migration cost is substantial — months of dual-running, retraining, tooling integration. Likely a multi-year journey, done incrementally for new services first.

## 19.4 GraphQL gateway

LinkedIn adopted GraphQL for some BFF surfaces:

- A central **GraphQL gateway** federates many Rest.li resources.
- Mobile/web clients query the gateway with a tailored query shape.
- Gateway parses, resolves each field via the underlying Rest.li (or other) service, hydrates, and returns.

Benefits:
- Fewer round-trips for mobile.
- Tailored payloads (no over-fetching).
- Schema-first across frontends.

Drawbacks:
- N+1 query risk; mitigated by DataLoader-style batching.
- Authorization complexity (per-field, not per-endpoint).
- New layer to operate.

## 19.5 Resilience patterns

Staff-level discussion points:

### Retries

- Built into Rest.li client (configurable).
- Idempotency-aware: only retry idempotent operations by default.
- Exponential backoff with jitter.
- Retry budget — cap total retries per second to avoid retry storms.

### Timeouts

- Every call has a timeout.
- Deadline propagation: caller's deadline passed to callee; chained timeouts.

### Circuit breakers

- Per-endpoint circuit breaker via degrader.
- Open circuit on persistent failures; periodic probe to test recovery.

### Hedging

- For latency-sensitive calls, issue a duplicate request after delay D if first hasn't returned.
- Cancel whichever response arrives second.
- Reduces tail latency at the cost of some duplicate work.

### Bulkheads

- Separate thread pools / connection pools per downstream service.
- Failures in one don't exhaust capacity for others.

### Backpressure

- Reactive Streams idioms via ParSeq.
- Queue-based limits per service.

## 19.6 Observability

- **Tracing**: OpenTelemetry; trace IDs propagated through Rest.li headers.
- **Metrics**: per-call metrics exposed via internal InGraphs.
- **Logs**: structured, with correlation IDs.

## 19.7 Multi-DC and request routing

- Each DC has its own D2 / ZooKeeper.
- Default: prefer-local routing.
- Fallback: if local endpoints unhealthy, route cross-DC.
- "Sticky DC" for sessions where user-affinity matters.

## 19.8 Common interview questions

> **"How does Rest.li compare to gRPC?"**
gRPC is more cross-language and uses Protobuf. Rest.li is JVM-centric with PDL. gRPC has HTTP/2 streaming built-in; Rest.li has more limited streaming. For a polyglot org, gRPC fits better; for a JVM org with deep tooling investment, Rest.li offers richer resource model and tooling.

> **"What's the bottleneck in scaling D2 to 10K services?"**
ZooKeeper write throughput on registration churn. Watches scaling. Client memory holding endpoint lists. Mitigations: shard ZooKeeper, watch coalescing, smarter watches.

> **"How would you migrate a Rest.li endpoint to GraphQL?"**
GraphQL gateway adds a resolver pointing at the Rest.li endpoint; clients consume via GraphQL; old Rest.li clients continue working. Eventually the gateway becomes the canonical entry; clients migrate.

> **"How do you ensure idempotency in a Rest.li action?"**
Idempotency key in request header; server stores recently-seen keys (TTL). Repeated request with same key returns cached response.

> **"How do you handle a service whose API changed in an incompatible way?"**
Major version bump in the schema → new client package (`v1` → `v2`). Old clients continue calling v1 endpoint; new clients call v2. Eventually deprecate v1 after migration window.

> **"How do you propagate user context across service hops?"**
Caller-ID header + member-ID header set by the BFF; D2 forwards on each hop. mTLS for service-to-service auth.

> **"How would you implement request-cancellation in Rest.li?"**
Async client returns a CompletableFuture; cancelling it sends a HTTP request cancellation; server-side checks cancellation flag and aborts processing. Limited support — best for I/O-bound calls.

> **"What's the trade-off of client-side discovery (D2) vs. server-side (load balancer)?"**
Client-side: lower latency (no extra hop), smarter routing decisions, more client complexity. Server-side: simpler clients, easier to enforce policy centrally, but adds a hop and a SPOF. LinkedIn went client-side; modern service meshes blur the line (sidecar = local proxy = client-side latency with server-side policy).