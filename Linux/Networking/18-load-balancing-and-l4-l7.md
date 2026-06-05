# 18 · Load Balancing — L4 and L7

The connecting tissue between Linux networking and "system design." Every staff interview has at least one LB question; the depth here is what distinguishes senior from staff.

## 18.1 The four layers of LB

```
   Client
      │
      ▼
   Anycast / DNS GSLB         ← geographic / TTL-based
      │
      ▼
   L4 LB (Maglev/Katran/IPVS) ← TCP/UDP, hash-based, stateless or near-stateless
      │
      ▼
   L7 LB (Envoy/NGINX/HAProxy) ← HTTP, routes by URL/header, stateful
      │
      ▼
   App / Service Mesh sidecar ← optional finer routing
```

Each layer has a different latency, scope, and failure model. The staff interview asks: *why all four?* answer: blast radius, scale, and feature surface.

## 18.2 DNS-based LB / GSLB

**Mechanism**: multiple A/AAAA records; client picks one. TTL controls convergence.

**Smart variants**:
- **Geo-DNS** — return record closest to client (resolver-based).
- **Latency-based** — Route 53 latency routing.
- **Weighted** — return records in proportion.
- **Health-checked** — withdraw unhealthy targets.

**Pros**: cheap, scalable to internet-wide.
**Cons**: resolver caching makes convergence slow (TTL + client cache); doesn't react to per-flow performance.

**Limits**: never the primary HA mechanism. TTL of 60s means a minute of bad routing for some clients.

## 18.3 Anycast — same IP, many sites

Already covered in `10` and `13`. Anycast routes the client to the nearest POP via BGP. Used by:
- Cloudflare 1.1.1.1 DNS.
- Google 8.8.8.8.
- All major CDNs.
- L4 LBs at the edge.

**Pros**: instant geo-routing without DNS gymnastics; sub-second BGP convergence with BFD.
**Cons**: flow reassignment under route change (TCP RSTs); requires BGP-speaking infrastructure.

## 18.4 L4 LB — the throughput tier

Operates on TCP/UDP. Doesn't terminate connections (mostly). Routes by 5-tuple hash to a backend.

### Topology: DSR (Direct Server Return)

```
  Client ──► LB ── encapsulate ──► Backend
     ▲                                │
     │                                │ Direct reply with src IP=LB-VIP
     └─────────── reply ◄─────────────┘
```

LB sees only the request (forward path). Reply goes directly from backend to client. The LB is on the *request hot path only* — massively scalable.

Implementation:
- **IPIP encapsulation**: LB wraps with outer IP header → backend.
- **GUE (Generic UDP Encapsulation)**: UDP-wrapped; firewall-friendly.
- Backend's stack decapsulates; src IP stays original client IP.
- Backend's "loopback" or anycast IP holds the VIP; SYN-ACK sent with that as source.

Used by: Google Maglev, Facebook Katran, AWS NLB.

### Topology: Reverse Proxy

```
  Client ──► LB ── new TCP conn ──► Backend
     ▲                                │
     │                                │
     └────── reply over orig conn ◄───┘
```

LB terminates client connection, opens new connection to backend. Sees all traffic both ways.

**Pros**: full visibility, easy to do L7 logic, can rewrite headers.
**Cons**: two connections, twice the state, half the throughput per LB CPU.

Used by: NGINX, HAProxy, Envoy by default.

### Choosing DSR vs proxy

- **DSR for L4 only, high pps**: when you don't need to inspect, just forward.
- **Proxy for L7**: when you need TLS termination, header rewriting, routing rules.

Often both: anycast → DSR L4 → proxy L7. (Cloudflare uses this stack.)

## 18.5 Consistent hashing — Maglev

The connection-stickiness problem: how do you route a flow to the same backend even as the backend set changes?

**Naive hash mod N**: when N changes, all flows reshuffle. TCP connections die.

**Maglev** (Google paper, 2016): a lookup table of size M (typically 65537 prime). Pre-compute permutation per backend; fill the table such that each backend owns ~M/N entries with minimal disruption on backend churn.

```
Hash(5-tuple) → table index → backend
```

When a backend joins/leaves, only ~M/N entries change → only that fraction of flows reshuffle. Existing flows mostly stick.

Cilium's L4 LB implements this in eBPF. Katran does it in XDP.

### Alternatives: rendezvous hashing, jump consistent hash

Different math, similar guarantees. Maglev's table-based approach is the most cache-friendly in software.

## 18.6 IPVS — kernel L4 LB

`ipvsadm` configures. Modes:
- **NAT** — LB DNATs to backend; SNATs reply.
- **DR (Direct Routing)** — LB sends to backend at L2; backend has loopback VIP; replies directly. Same idea as DSR but at L2.
- **TUN (IP-in-IP tunneling)** — like DSR but tunneled.

Algorithms: RR, WRR, LC (least-conn), WLC, SH (source hash), DH, LBLC, LBLCR, SED, NQ.

Performance: ~10M pps on a tuned box. Scales linearly with CPUs.

Where to use: Kubernetes kube-proxy IPVS mode; some standalone DC LBs.

## 18.7 Katran — XDP-based L4 LB

Facebook's open-source L4 LB. Runs as an XDP program at the host. Maglev hashing. Encapsulates with IP-in-IP to backend.

Architecture:
```
   NIC ──► XDP program (Katran) ──► XDP_TX (encapsulated) ──► NIC
                │
                ▼
            stats / config maps
                ▲
                │
           userspace agent (configures via maps)
```

Replaces dedicated LB hardware boxes. ~10× cheaper per pps than IPVS, ~30× cheaper than a Cisco LB.

Open-source; many ports / forks. Github: facebook/katran.

## 18.8 L7 LB — feature-rich tier

Operates on HTTP/gRPC/etc. Inspects, routes, transforms.

| LB | Style | Strengths |
|----|-------|-----------|
| **NGINX** | event-driven, multi-worker | Static config + Lua; battle-tested |
| **HAProxy** | event-driven | Performance, observability, TCP+HTTP |
| **Envoy** | C++, hot-reloadable config | Modern, service-mesh-ready, xDS API |
| **Traefik** | Go | k8s-native, dynamic discovery |
| **Caddy** | Go | Auto-TLS via Let's Encrypt |
| **Linkerd-proxy** | Rust | Service mesh focused |

Features:
- **TLS termination** (kTLS makes this cheap).
- **Routing rules** (host, path, header).
- **Rate limiting** (token bucket per key).
- **Circuit breaking** (open after error rate).
- **Retry budgets** (% of requests retryable).
- **Health checking** (active probes).
- **Observability** (logs, traces, metrics).

## 18.9 Envoy + xDS (the service mesh standard)

Envoy is configured by a control plane (Istio, Consul, Linkerd's own) via the **xDS** API:
- **LDS**: listener discovery.
- **CDS**: cluster discovery (backends + LB rules).
- **EDS**: endpoint discovery.
- **RDS**: route discovery.
- **SDS**: secret discovery (certs).

Control plane pushes config; Envoy hot-reloads without dropping connections.

This decoupling is *the* reason service meshes work: data plane (Envoy) is stable; control plane (Istio/Linkerd/etc.) evolves separately.

## 18.10 Sidecar vs ambient mesh

Two service mesh deployment models:

- **Sidecar**: Envoy runs alongside each app pod. Container-level isolation. High memory cost.
- **Ambient** (Istio 2023+, Cilium): one shared proxy per node (for L4 + mTLS), optional L7 proxy per service. Much lower overhead.

eBPF-based Cilium Service Mesh: kernel-level redirects to a per-node Envoy; no sidecar. The "no-sidecar" pattern.

## 18.11 Health checks — the operational glue

| Mode | What | Cost |
|------|------|------|
| **Passive** | Track failures from actual traffic | Cheap, slow detection |
| **Active TCP** | Open TCP conn periodically | Catches L4 fails |
| **Active HTTP** | Issue HTTP GET on a path (`/healthz`) | Catches L7 fails |
| **Active app-protocol** | gRPC health check, MySQL ping | App-specific |

Tradeoffs:
- Too aggressive → false positive, flapping.
- Too lenient → traffic to dead backends.
- Sticky session aware → drain before remove.

**Outlier detection**: Envoy's. Eject a backend after N consecutive failures, regardless of explicit health-check.

## 18.12 Failure modes (the staff stories)

- **Maglev backend ejection storm**: massive simultaneous backend churn → many flows reshuffle. Mitigation: stagger removal, slow drain.
- **L4 LB without L7 health check**: L4 sees backend's TCP socket alive but L7 returns 500s. Mitigation: probe at the application level.
- **Asymmetric routing in DSR**: client→LB→backend; backend→client directly. If the client's NAT changes mid-flow, reply may not find the client. Mitigation: stable source IPs, or proxy mode.
- **Hot-key**: one client/header generates 80% of traffic; load balanced to one backend → that backend smokes. Mitigation: rate limit per source; hash on (key, backend-set) and re-shard.
- **Slow Loris / connection holding**: clients hold idle connections to exhaust LB's connection table. Mitigation: idle timeout, max conn per source.
- **Backend overload + retry storm**: backends slow → clients retry → more load → cascading. Mitigation: retry budgets (Envoy "outlier" + max-retries).

## 18.13 The staff design — global L4 LB

A real-world stack (this is the "design Cloudflare's edge" answer):

```
   Client (anywhere)
      │
      ▼
   Anycast IP via BGP                      ← geo-routes to nearest POP
      │
      ▼
   Top-of-rack switch ECMP (per-flow hash) ← spreads across LB tier
      │
      ▼
   L4 LB cluster (XDP Maglev)              ← consistent hashes flow → backend
      │ (DSR encapsulation, IPIP)
      ▼
   Backend / L7 proxy (Envoy)              ← TLS termination, HTTP routing
      │
      ▼
   App / origin
```

Failure handling:
- BGP withdraws → traffic shifts to other POP.
- L4 LB removed → ECMP rebalances; Maglev tries to preserve flows.
- Backend down → outlier detection ejects; Maglev moves new flows; existing TCP gets RST.

This stack handles millions of qps with O(N) cost where N = backend count.

## 18.14 Specific company stories

| Company | L4 LB | L7 LB | Notes |
|---------|-------|-------|-------|
| Google | Maglev (in-house) | GFE (Google Front End) | Maglev paper 2016 |
| Facebook/Meta | Katran (XDP) | Proxygen | Katran open-source |
| Cloudflare | Unimog (XDP) | NGINX | Unimog blog 2018 |
| Netflix | (cloud-routed) | Zuul | Java-based at Netflix |
| AWS | NLB (custom) | ALB (custom) | NLB does cross-zone |
| Microsoft | Ananta (custom) | Azure Front Door | Ananta paper 2013 |
| LinkedIn | (in front of D2/REST) | NGINX/Envoy | Internal mesh |

Memorize 2-3 to cite in interview answers.

## 18.15 Common interview probes

- **"Design a global load balancer."** Anycast → ECMP → L4 (Maglev) → L7 (Envoy). Spec'd above.
- **"Why DSR over proxy?"** Reply traffic dominates request traffic for big content. DSR offloads reply from LB.
- **"Why consistent hashing matters?"** Backend churn (adds, removes, fails) doesn't shuffle every flow. Maglev limits to ~M/N flows shuffled.
- **"What about UDP / QUIC?"** UDP LB on 5-tuple hash; QUIC LB on connection ID. Cloudflare uses connection-ID-aware LB for QUIC.
- **"How does session affinity work?"** L7 cookie or hash on source IP. Trade-off: clients behind NAT all share same affinity.
- **"How would you do gradual rollouts?"** L7 LB with header/weight-based routing → 1% canary, then 10%, then 100%. Envoy / Istio do this natively.

## 18.16 Corner cases

- **Maglev table size choice.** Smaller table = more aggressive rebalancing on churn; larger = less collision. 65537 is the common pick.
- **ECMP + L4 LB collision.** If both ECMP and L4 LB hash on 5-tuple, they may concentrate flows. Mitigate by including layer 4 ports in both hashes (or use different keys).
- **Long-lived connections (WebSocket) + LB drain.** Closing the listener doesn't close existing; need backend to drain. Set `terminationGracePeriodSeconds` in k8s.
- **Asymmetric NAT in DSR.** Client behind NAT may have a different src port for return packets. Usually fine for TCP (4-tuple is set), but UDP-with-NAT can break.
- **TLS termination + client cert.** mTLS at L7 LB means LB sees client cert; backend doesn't. Pass via header (X-Client-Cert) or use sidecar that re-establishes mTLS.
- **Slow start / TCP cwnd on long-lived backends.** When LB opens many short connections to backend, each starts in slow start. Mitigation: connection pool with keepalive.

## 18.17 Performance numbers

| LB | pps per CPU | Notes |
|----|-------------|-------|
| iptables-based kube-proxy | ~500K (degrades with rules) | Linear scan |
| IPVS | ~10M | Kernel-level |
| HAProxy | ~1M (L7), ~5M (L4) | Connections matter |
| NGINX | ~500K rps L7 | TLS heavy |
| Envoy | ~50K-200K rps L7 | C++; sidecar adds 1ms |
| Katran (XDP) | ~30M pps | Line-rate for short pkts |
| DPDK + VPP | ~50M+ pps | Userspace |
| Maglev (Google) | "many millions of qps per machine" | Proprietary |

## 18.18 Alternative approaches

| Need | Path 1 | Path 2 | Path 3 |
|------|--------|--------|--------|
| Geo LB | DNS GSLB | Anycast + BGP | client-side multi-endpoint |
| L4 LB | IPVS | Katran (XDP) | DPDK VPP |
| L7 LB | NGINX | Envoy + xDS | HAProxy |
| Service mesh | Istio + Envoy sidecar | Linkerd | Cilium (ambient eBPF) |
| Health check | Active HTTP | Active gRPC | Passive (Envoy outlier) |
| Session affinity | Cookie | Source IP hash | Maglev consistent |
| Cross-region | Latency-based DNS | Anycast | App-aware routing |

## 18.19 The 60-second pitch (memorize)

> "A globally available service stacks four LB layers. **DNS / GSLB** for geo-affinity at name-resolution time. **Anycast** for instant geographic routing without DNS TTL lag. **L4 LB** (Maglev or Katran XDP-based) using consistent hashing so backend churn doesn't reshuffle every flow; DSR (direct server return) so reply traffic doesn't traverse the LB. **L7 LB** (Envoy or NGINX) for TLS termination, HTTP-aware routing, rate limiting, circuit breaking; in a service mesh, this is the sidecar or ambient proxy. Each layer has a distinct cost-per-event: anycast is BGP-fast (~150ms with BFD); L4 LB is microseconds with XDP; L7 LB is single-digit ms. Choose layers based on whether you need geo, throughput, or feature surface — typically all three."

## Must-internalize

- 4-tier LB stack: DNS/Anycast → ECMP → L4 (Maglev/Katran) → L7 (Envoy).
- DSR (Direct Server Return) offloads reply path from LB; needed for high-egress workloads.
- Maglev consistent hashing minimizes flow shuffle on backend churn.
- IPVS: kernel-level L4; Katran: XDP-level L4; DPDK/VPP: userspace.
- Envoy + xDS = data plane / control plane separation.
- Service mesh: sidecar (per-pod) vs ambient (per-node, eBPF).
- Health checks: active HTTP for app-level liveness; passive for cheap monitoring.
- Common failures: hot-key, retry storm, slow loris, asymmetric NAT in DSR.