# 09 · Network Policies and Service Mesh

The security + policy layer on top of basic CNI networking. Staff candidates should know NetworkPolicy semantics, which CNI implements policies how, and where service meshes (Istio, Linkerd, Cilium) fit.

## 9.1 NetworkPolicy — the native resource

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: db-deny}
spec:
  podSelector:
    matchLabels: {app: postgres}
  policyTypes: [Ingress, Egress]
  ingress:
  - from:
    - podSelector:
        matchLabels: {role: backend}
    ports:
    - port: 5432
      protocol: TCP
  egress:
  - to:
    - namespaceSelector:
        matchLabels: {kubernetes.io/metadata.name: kube-system}
      podSelector:
        matchLabels: {k8s-app: kube-dns}
    ports:
    - port: 53
      protocol: UDP
```

### Semantics (the staff-level points)

- **Default deny once selected**: if any NetworkPolicy selects a pod, the pod is **deny-by-default** for that direction. Add allow rules to permit traffic. Pods not selected by any policy are unrestricted.
- **Ingress vs Egress**: separate. Common pattern: a "deny all egress" policy + specific allow rules.
- **Pod selector** is namespace-scoped; cross-namespace requires `namespaceSelector`.
- **Targets**: `podSelector`, `namespaceSelector`, `ipBlock` (CIDR), or combinations.
- **No fully-qualified domain support**: policies are L3/L4 only. No `allow access to api.github.com`.

### Common patterns

```yaml
# 1. Deny all egress, allow only DNS + cluster
metadata: {name: deny-egress}
spec:
  podSelector: {}    # selects ALL pods in namespace
  policyTypes: [Egress]
  egress:
  - to:
    - namespaceSelector:
        matchLabels: {kubernetes.io/metadata.name: kube-system}
      podSelector:
        matchLabels: {k8s-app: kube-dns}
    ports: [{port: 53, protocol: UDP}]
  - to:
    - podSelector: {}   # any pod in this namespace
---
# 2. Deny ingress except from labeled
metadata: {name: ingress-restrict}
spec:
  podSelector: {matchLabels: {app: prod}}
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector: {matchLabels: {tier: gateway}}
```

### Caveats

- **No FQDN support** — pure-L4. Use Cilium FQDN policies or service mesh L7 for "allow github.com."
- **Conntrack still applies** — return traffic for established connections is allowed regardless of policy.
- **Policies are pod-targeted, not service-targeted** — a Service's traffic is filtered at the pod the packet arrives at.
- **Not all CNIs implement NetworkPolicy**: flannel-vxlan doesn't (needs Calico-bridge layer). Calico, Cilium, Antrea, OVN do.
- **Namespace-scoped**: cross-namespace requires explicit `namespaceSelector`.

## 9.2 NetworkPolicy variants — CNI-specific

### Cilium NetworkPolicy (CRD)

Extends k8s NetworkPolicy with:
- L7 rules (HTTP method/path, Kafka topic, gRPC service).
- FQDN matching.
- ToServices (allow to a Service rather than a label).
- ToEntities (host, cluster, world, all).

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
spec:
  endpointSelector:
    matchLabels: {app: api}
  egress:
  - toFQDNs:
    - matchPattern: "*.github.com"
    toPorts:
    - ports: [{port: "443", protocol: TCP}]
  - toServices:
    - k8sService:
        serviceName: db-primary
        namespace: backend
```

### Calico GlobalNetworkPolicy

Cluster-wide policies; can apply to host networking (not just pods).

### NetworkPolicy v2 (KEP-2091, "AdminNetworkPolicy")

In progress: cluster-admin-scoped policies that override namespace policies. Tiered enforcement (admin always wins).

## 9.3 Service mesh — the L7 overlay

Service meshes add:
- **mTLS** between services (encrypted + authenticated).
- **L7 routing** (HTTP/gRPC, retries, timeouts, circuit breaking).
- **Observability** (auto-instrumented metrics + traces).
- **Traffic policy** (canary, fault injection, rate limit).

### The three players (2026)

| Mesh | Data plane | Control plane |
|------|-----------|---------------|
| **Istio** | Envoy sidecar (or ztunnel + Envoy for ambient) | Istiod |
| **Linkerd** | linkerd-proxy (Rust, micro-proxy) | linkerd control plane |
| **Cilium service mesh** | eBPF + optional Envoy per-node | Cilium agent |

### Sidecar vs ambient

```
Sidecar (Istio classic, Linkerd):
   Pod = [app container] + [envoy/linkerd-proxy container]
   - One proxy per pod
   - ~50MB memory, ~1ms latency
   - 10K pods × 50MB = 500GB just for proxies
```

```
Ambient (Istio ambient, Cilium service mesh):
   Pod = [app container] (no sidecar)
   Per-node L4 proxy (ztunnel) handles mTLS
   Per-service L7 proxy (waypoint, optional) handles HTTP policies
   - Massively less overhead for L4-only use cases
```

## 9.4 Istio architecture

```
                  ┌──────────────────┐
                  │   Istiod         │     control plane:
                  │   (pilot, citadel, galley combined since 1.5) │
                  └────────┬─────────┘
                           │ xDS push (LDS/CDS/EDS/RDS/SDS)
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
       ┌─────────┐    ┌─────────┐    ┌─────────┐
       │ Envoy   │    │ Envoy   │    │ Envoy   │
       │ sidecar │    │ sidecar │    │ sidecar │
       └─────────┘    └─────────┘    └─────────┘
```

Istio injects Envoy sidecars via MutatingAdmissionWebhook. Outbound traffic from app → captured by iptables → routed to local Envoy → mTLS to remote Envoy → forwarded to app.

### Key Istio CRDs

- **VirtualService** — routing rules (host, path, header → destination).
- **DestinationRule** — per-destination policies (LB algorithm, mTLS mode, connection pool).
- **Gateway** — edge ingress configuration.
- **ServiceEntry** — register external services in the mesh.
- **AuthorizationPolicy** — RBAC for service-to-service.
- **PeerAuthentication** — mTLS requirements (STRICT, PERMISSIVE, DISABLE).
- **EnvoyFilter** — escape hatch for arbitrary Envoy config.

### mTLS

Each pod gets a workload identity (SPIFFE-format: `spiffe://cluster.local/ns/foo/sa/bar`) and a short-lived cert (~24h, auto-rotated). PeerAuthentication CRD sets the mode:

- **STRICT**: only mTLS accepted.
- **PERMISSIVE**: accept both mTLS and plaintext.
- **DISABLE**: plaintext only.

The PERMISSIVE mode is critical during migration — let mesh + non-mesh pods coexist.

## 9.5 Linkerd

Lighter than Istio. Uses linkerd2-proxy (Rust). Focus on simplicity:
- No CRDs beyond ServiceProfile and TrafficSplit.
- Auto-mTLS by default.
- Auto-instrumented metrics.
- No "fault injection" / "circuit breaker" config — opinionated about what to expose.

Production tradeoff: less powerful than Istio but easier to operate.

## 9.6 Cilium service mesh

Cilium replaces both NetworkPolicy and (optionally) the service mesh:
- **L4 mTLS** in eBPF (no sidecar, no envoy).
- **L7 policies** via Envoy per-node (waypoint pattern) when needed.
- **Hubble** — observability built-in (every flow, every L7 request).
- **Cluster mesh** — cross-cluster service discovery.

The "ambient mesh" model (no sidecar) is Cilium's native pattern. Istio adopted similar for 1.18+ ambient mode.

## 9.7 mTLS without service mesh

You can do mTLS without a mesh:
- **cert-manager** issues certs to apps.
- Apps load certs themselves and authenticate.
- Pros: no sidecar overhead.
- Cons: apps must implement TLS; rotation must be plumbed; no observability glue.

Sometimes the right call for small clusters.

## 9.8 SPIFFE / SPIRE

A standard for workload identity. SPIRE is the reference impl:
- Each workload gets a SPIFFE ID (`spiffe://trust-domain/path`).
- Short-lived X.509 certs (SVID).
- Identity is bound to k8s selectors (namespace + ServiceAccount + label).

Istio, Cilium service mesh, and modern HashiCorp Consul Connect all use SPIFFE-style identities under the hood.

## 9.9 Traffic policy patterns

### Canary

```yaml
# Istio: VirtualService with weighted routing
spec:
  hosts: [my-svc]
  http:
  - route:
    - destination: {host: my-svc, subset: v1}
      weight: 90
    - destination: {host: my-svc, subset: v2}
      weight: 10
```

### Header-based routing

```yaml
http:
- match:
  - headers: {x-canary: {exact: "true"}}
  route:
  - destination: {host: my-svc, subset: v2}
- route:
  - destination: {host: my-svc, subset: v1}
```

### Circuit breaker

```yaml
# DestinationRule
spec:
  trafficPolicy:
    connectionPool:
      tcp: {maxConnections: 100}
      http: {http1MaxPendingRequests: 1024, http2MaxRequests: 1024}
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

### Retry budgets

```yaml
http:
- retries:
    attempts: 3
    perTryTimeout: 2s
    retryOn: 5xx,reset,connect-failure
```

## 9.10 Gateway API + Mesh

Gateway API (1.27+ stable) is becoming the unified routing layer:
- Ingress controllers implement it.
- Service meshes implement it (Istio supports HTTPRoute as VirtualService alternative).
- Provides standard cross-vendor config.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
spec:
  parentRefs:
  - kind: Service
    name: my-svc   # mesh-mode: route belongs to a Service
  rules:
  - matches: [{path: {type: PathPrefix, value: "/v2"}}]
    backendRefs:
    - {name: my-svc-v2, port: 80, weight: 100}
```

## 9.11 Observability via mesh

Auto-instrumented mesh metrics:
- Request rate, error rate, p50/p95/p99 latency per service-pair.
- Distributed tracing (with proper header propagation in apps).
- Connection-level health.

Without mesh, apps must instrument themselves. The mesh gives you "free" observability — modulo the cost of the sidecar.

## 9.12 Common interview probes

- **"How does NetworkPolicy work?"** Selects pods; deny-by-default on selected direction; allow rules permit traffic. Implemented by CNI (Calico, Cilium, etc.).
- **"Why no FQDN in NetworkPolicy?"** Native NP is L3/L4. CNI-specific (Cilium) extends to FQDN.
- **"Sidecar vs ambient mesh?"** Sidecar = per-pod proxy, high overhead. Ambient = per-node L4 proxy + optional L7 — much lower overhead for L4-only use cases.
- **"How does mTLS in a mesh work?"** SPIFFE workload identity → short-lived cert → mutual TLS between sidecars. PeerAuthentication PERMISSIVE allows migration.
- **"Can I do mTLS without a mesh?"** Yes — cert-manager + app-level TLS. Trade-off: no automatic, no observability.
- **"How would you implement canary with k8s primitives only?"** Two Deployments + Service with 90/10 label selector — but Services don't weight; you'd need two Services + DNS round-robin or app-level routing. Service mesh is the cleaner answer.

## 9.13 Corner cases

- **NetworkPolicy with conntrack** — return traffic is allowed by conntrack regardless of egress policy. Egress restrictions only affect new outbound.
- **NetworkPolicy + hostNetwork pod** — hostNetwork bypasses CNI; NP doesn't apply.
- **NetworkPolicy doesn't apply to DNS** by default — easy to forget; many "broken pod" stories.
- **Mesh sidecar startup race** — main container starts before Envoy ready, first requests fail. Use `holdApplicationUntilProxyStarts` (Istio) or KEP-753 (sidecar containers).
- **Mesh + StatefulSet ordering** — pod identity must be stable; mesh identity ties to SA + namespace.
- **DNS hijacking by mesh** — Istio's iptables rules intercept all egress; non-mesh-aware apps may misbehave on unusual ports.

## 9.14 Alternative approaches

| Need | Path 1 | Path 2 | Path 3 |
|------|--------|--------|--------|
| Block all egress except DNS | NetworkPolicy default-deny + allow DNS | Cilium NP with FQDN + DNS-only | OPA Gatekeeper at admission |
| mTLS between services | Service mesh (Istio/Linkerd) | Cilium service mesh (eBPF) | App-level + cert-manager |
| Canary deployment | Service + two Deployments + DNS RR | Istio VirtualService weighted | Argo Rollouts (canary controller) |
| Circuit breaker | Service mesh outlier detection | App-level (Resilience4j, Polly) | Envoy as gateway |
| Trace propagation | Mesh auto-injects headers | App + OpenTelemetry SDK | tcpdump + manual correlation |
| Cluster-wide policy | NetworkPolicy per namespace | AdminNetworkPolicy (v2) | OPA / Kyverno admission |

## 9.15 The future — eBPF over sidecars

The direction:
- L4 mTLS in eBPF (Cilium has it; Istio working on it).
- L7 only where needed (waypoint pattern).
- No per-pod proxy except for L7 features.
- Observability via eBPF, not sidecar logs.

Expect interview questions about "why is sidecar mesh dying?" Answer: per-pod overhead (memory, latency), startup races, ops complexity. eBPF + waypoint mesh gives 80% value at 20% cost.

## Must-internalize

- NetworkPolicy: deny-by-default once selected; L3/L4; per-pod, per-namespace.
- Always allow DNS egress in restrictive policies.
- Cilium FQDN policies; Calico GlobalNetworkPolicy; AdminNetworkPolicy (v2).
- Service mesh = mTLS + L7 routing + observability.
- Istio (full-featured, sidecar or ambient); Linkerd (simple, sidecar); Cilium service mesh (eBPF + waypoint).
- SPIFFE/SPIRE for workload identity.
- Gateway API is the future for ingress + mesh routing.
- Sidecar overhead is real (50MB per pod); ambient mesh is the response.
