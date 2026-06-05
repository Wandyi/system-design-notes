# 21 · System Design Questions

12 prompts that have been asked in staff-level Linux networking interviews, with full answer sketches. Each is a 45–60 minute design round. The signal isn't the "right answer" — it's the *structured walk* through scope, components, scale, failure modes, and trade-offs.

## 21.1 Design a global L4 load balancer

**Prompt**: scale to 10M qps, 100Gbps, sub-ms added latency.

**Walk**:
1. **Scope**: TCP/UDP, no L7. Health-checked backends. Survive POP failure.
2. **Topology**:
   ```
   Client ──► anycast IP via BGP
                  │
                  ▼
              POP nearest client (10 globally)
                  │
                  ▼
              ToR switch ECMP
                  │
                  ▼
              L4 LB cluster (8 boxes × 100Gbps NIC)
                  │ (XDP + Maglev consistent hash)
                  ▼
              Backend pool (50 hosts per POP)
   ```
3. **L4 LB tech**: XDP-based Katran/Unimog. Consistent hash on 5-tuple → Maglev table (65537 entries). DSR via IPIP or GUE encapsulation. Backend has VIP on loopback; replies directly to client (LB doesn't see reply).
4. **Scale**: each LB at 25Mpps. 8 boxes = 200Mpps capacity per POP.
5. **Health checks**: passive (Envoy-style outlier detection) + active TCP probe per backend every 5s; remove from Maglev table on failure.
6. **Failure modes**:
   - LB host dies → ECMP rebalances; some flow disruption (Maglev limits to ~M/N flows).
   - Backend dies → removed from Maglev table; affected flows die. Acceptable.
   - POP fiber cut → BGP withdraws; sub-second with BFD.
   - Maglev table churn during big rolling update → stagger removals.
7. **Observability**: per-backend pps/drops in BPF map → scraped to Prometheus. tcpdump for individual debugging.

**Staff signal**: name Maglev, DSR, IPIP/GUE; specify the BFD timing; pre-empt the "what if a flow rebalances" question.

## 21.2 Design a CDN edge (Cloudflare-style)

**Prompt**: 1M qps from anywhere; cache HTTP assets; TLS terminated.

**Walk**:
1. **Scope**: HTTPS to anycast IP; cache hot assets; revalidate against origin.
2. **Topology**:
   ```
   Client ──► anycast (BGP)
                  │
                  ▼
              POP nearest (200+ globally)
                  │
                  ▼
              ECMP
                  │
                  ▼
              L4 LB (XDP) → L7 LB (NGINX)
                  │
                  ▼
              Cache lookup (RAM + SSD, BPF maps for hot lookups)
                  │
                  ├── hit → respond
                  ▼
              Origin (customer's server)
   ```
3. **TLS**: kTLS + NIC offload; `sendfile()` from cache file directly to TLS socket.
4. **Caching**: tier 1 RAM (BPF-keyed hash for hot N), tier 2 NVMe, miss → origin. Per-asset TTL.
5. **Routing rules**: by host/path in NGINX config; reloaded via API.
6. **Failure modes**:
   - POP fails → anycast withdraws.
   - Origin slow → stale-while-revalidate; serve cached for some grace period.
   - Cache stampede on cold object → coalescing (single fetch even if many requests).
7. **Scale numbers**: per box 50Gbps (NIC kTLS); 100 boxes per POP = 5Tbps per POP.

**Staff signal**: kTLS, stale-while-revalidate, anycast + BFD.

## 21.3 Design a K8s CNI

**Prompt**: 5000 nodes, 100K pods, network policies, multi-tenant.

**Walk**:
1. **Scope**: pod-to-pod across nodes; service VIPs; network policies; multi-tenant (namespace = tenant).
2. **Data plane**: Cilium-style. Each pod gets a veth; eBPF programs at tc-ingress/egress; service map and conntrack as eBPF maps.
3. **Control plane**: cilium-agent on each node; talks to Kubernetes API; computes BPF maps for services and policies; pushes via netlink.
4. **Inter-node**: routed (BGP per-node CIDR) or overlay (VXLAN with NIC offload).
5. **Service VIPs**: BPF map keyed by VIP+port; lookup gives endpoints; consistent hash to pick.
6. **Network policies**: BPF program at egress checks (source identity, dest identity, port); deny if disallowed.
7. **Failure modes**:
   - Agent crash → kernel BPF programs continue forwarding; just no policy changes until restart.
   - Node disconnects from control plane → continues to forward.
   - Pod IP exhaustion → IPAM with proper CIDR sizing.
8. **Observability**: Hubble (Cilium's flow observability) — every flow event into a ring buffer; pulled to UI.

**Staff signal**: Cilium > kube-proxy iptables for >1K services; identity-based policy (not IP-based) for cloud-native.

## 21.4 Design a service mesh

**Prompt**: mTLS between services, retry/circuit-break, observability, multi-cluster.

**Walk**:
1. **Topology choice**: sidecar (Envoy per pod) vs ambient (eBPF + node-level proxy).
2. **mTLS**: SPIRE for identity (SPIFFE IDs); short-lived certs (~1h); rotated automatically. Cilium can do mTLS in kernel.
3. **Routing**: xDS API; Istio/Linkerd control plane pushes config.
4. **Cross-cluster**: federation via shared root CA; multi-cluster service discovery.
5. **Observability**: Envoy emits trace span per hop; trace ID propagated via header.
6. **Failure modes**:
   - Control plane outage → data plane keeps last-known config (Envoy's "static" mode).
   - mTLS cert expires → rotate; should be invisible.
   - One service slow → outlier ejection + retry budget.

**Staff signal**: mention SPIRE/SPIFFE; xDS; the sidecar overhead (~50MB / pod) and the ambient alternative.

## 21.5 Design DNS at scale (anycast)

**Prompt**: 10M qps, 100ms p99, DDoS-resistant.

**Walk**:
1. **Topology**: anycast IP at 50 POPs. Each POP has N DNS servers.
2. **Server stack**: `SO_REUSEPORT` workers; `recvmmsg`/`sendmmsg`; UDP GRO/GSO (4.18+). XDP DDoS filter at NIC.
3. **Cache**: per-server resolver cache; consistent zone files (rsync or DNS-AXFR).
4. **DDoS**: XDP_DROP for source-IP rate-limit-exceed; XDP for amplification-attack signatures (DNSSEC big-response probes).
5. **DoT/DoH**: TLS terminated at L7 LB inside POP for privacy variants.
6. **Failure**: POP down → BGP withdraw; client retries → resolves to next POP.
7. **Scale**: each server 200K qps; 50 servers/POP × 50 POPs = 500M qps capacity.

**Staff signal**: name `SO_REUSEPORT`, recvmmsg, UDP GRO, XDP DDoS, anycast.

## 21.6 Design a real-time chat / messaging

**Prompt**: 100M concurrent users; presence; messaging; latency <200ms.

**Walk**:
1. **Connection layer**: WebSocket (or QUIC) from clients to edge. Anycast distributes geographically.
2. **Edge servers**: long-lived connection holders. Use epoll or io_uring; ~1M conn per box.
3. **Presence**: server maintains "is user X online" via heartbeat. Distributed pub/sub (Redis cluster or in-house).
4. **Message routing**: A→B sends to A's edge; A's edge looks up B's edge (via routing service); forwards.
5. **Storage**: messages persisted (e.g., to LinkedIn's Espresso, Facebook's MyRocks, Google's Spanner).
6. **Scale**: each edge box holds 1M conns; need 100 boxes for 100M users (with headroom for failover).
7. **Failure**: edge dies → clients reconnect (5s typical); message in-flight retransmits.

**Staff signal**: epoll/io_uring scaling; per-CPU REUSEPORT; how presence scales (sharded).

## 21.7 Design an L7 API gateway

**Prompt**: 100K rps from public; JWT auth; rate limit; canary deploy.

**Walk**:
1. **Edge**: anycast → L4 LB → Envoy.
2. **Envoy**:
   - WASM or external authz for JWT.
   - Rate-limit filter (per-API-key bucket).
   - Routing by host/path.
   - Canary via header (`x-canary: true`).
3. **Backends**: gRPC / HTTP services.
4. **Observability**: every Envoy access log → Kafka; sampling for traces.
5. **Failure**: outlier detection at Envoy ejects bad backend; rate-limit memstore replicated.
6. **Scale**: per Envoy 50K rps; 10 boxes = 500K rps.

**Staff signal**: WASM filters, xDS push, retry budgets, outlier detection.

## 21.8 Design a multi-region database replication

**Prompt**: write replication from region A to region B with bounded staleness.

**Walk**:
1. **Transport**: TCP between regions; tune `tcp_rmem.max` to BDP (10Gbps × 100ms = 125MB).
2. **Congestion control**: BBR for high-RTT.
3. **Application**: log-shipping (binlog, MVCC log) over a connection.
4. **Multi-stream**: 16 parallel TCP streams to fill BDP without single-CPU bottleneck.
5. **Failure**: connection drop → replication catches up from last-acked LSN; replication lag SLO.
6. **Optional**: encrypted via kTLS (TLS is mandatory for cross-region).

**Staff signal**: BDP math, BBR, multi-stream, lag SLO.

## 21.9 Design an in-cluster service for telemetry (metrics + logs + traces)

**Prompt**: 1M containers; logs, metrics, traces; near-real-time queryable.

**Walk**:
1. **Collection**: eBPF auto-instrumentation (Pixie, Hubble) for traces; fluent-bit/vector for logs; Prometheus/node_exporter for metrics.
2. **Transport**: gRPC over TLS to ingest tier.
3. **Ingest**: Kafka / Pulsar / NATS for buffering. Kafka topics partitioned by service.
4. **Storage**:
   - Metrics: Prometheus + Thanos (or Mimir, VictoriaMetrics).
   - Logs: Loki or Elasticsearch.
   - Traces: Jaeger backed by Cassandra/ClickHouse.
5. **Query**: Grafana frontends each. Trace lookup by trace ID.
6. **Failure**: collection buffers locally on disk if Kafka unreachable; replay later.

**Staff signal**: eBPF for telemetry; Kafka as the buffer; per-signal-type backend.

## 21.10 Design a transparent proxy / sidecar

**Prompt**: intercept all egress from a pod, mTLS, rate limit, no app code changes.

**Walk**:
1. **Iptables redirect**: pod outbound → iptables REDIRECT to sidecar (Envoy) on local port.
2. **Sidecar**: Envoy with `iptables` capture or TPROXY; mTLS to remote; back to local on receive.
3. **Or Cilium ambient**: kernel-level redirect via eBPF (no iptables, no sidecar).
4. **Failure**: sidecar dies → pod has no egress (sometimes by design).
5. **Performance**: 1-2ms added latency for sidecar; 10-20% CPU. Ambient is 10× lower overhead.

**Staff signal**: TPROXY, the cost of sidecar, Cilium ambient as the modern answer.

## 21.11 Design an MQTT / IoT broker for 10M devices

**Prompt**: 10M devices each connecting; pub/sub; QoS levels; sub-second delivery.

**Walk**:
1. **Connection layer**: MQTT over TLS. Many short-write devices = many TCP keepalive. `tcp_keepalive` low to detect dead devices.
2. **Edge**: SO_REUSEPORT + 32 workers + io_uring (or epoll); 1M conns per box.
3. **Pub/sub fanout**: in-memory broker (Mosquitto cluster, EMQ X). For huge subscriber counts: Kafka-style fan-out behind broker.
4. **Failure**: device disconnects → broker retains last-will message; subscribers see device-offline event.

**Staff signal**: connection density numbers; `tcp_keepalive` tuning; broker vs Kafka.

## 21.12 Design a packet capture pipeline for SOC

**Prompt**: full packet capture at 100Gbps; queryable in 5 minutes.

**Walk**:
1. **Capture**: AF_XDP-based capture (zero-copy from NIC). Write to local NVMe ring buffer (last N minutes).
2. **Index**: streaming index (5-tuple, time) → ClickHouse / Vertica.
3. **Storage**: full pcap on tiered storage (NVMe → SSD → S3).
4. **Query**: BPF-based filter on stream OR full pcap scan for forensic.
5. **Failure**: capture box dies → another mirror takes over (SPAN port to multiple boxes).
6. **Scale**: 100Gbps = ~12.5GB/s; 1 minute = 750GB; need fat NVMe + tiered eviction.

**Staff signal**: AF_XDP, SPAN/mirror, ClickHouse for indexed lookup.

## 21.13 Common pitfalls in the design round (avoid)

- **No failure modes mentioned.** Always at least 3.
- **No scale numbers.** Always pps/qps/Gbps/RTT.
- **Single LB tier.** Real systems have anycast → L4 → L7 layers.
- **"Use Redis" without saying why.** Justify with consistency model.
- **No observability.** What signal tells you the system is healthy.
- **No on-call story.** Pages? Runbooks?
- **No cost.** A 10-region anycast is expensive; at staff you should know roughly.

## 21.14 The 60-second pitch on a generic staff-level design

> "Start with scope and scale numbers. Then layer top-down: client routing (DNS, anycast), L4 LB (Maglev/Katran, DSR), L7 LB (Envoy/NGINX, TLS, routing rules), backends, storage. For each layer name the tech and the scale per box. Walk through three failure modes — link/POP/backend — and explain detection (BFD, health probe, outlier) and recovery (BGP withdraw, ECMP rebalance, Maglev shuffle). Note observability per layer (BGP routes, LB metrics, backend latency p99). Mention cost and operational drift. The grader is watching for *whether your defaults are right* and *whether you ask about constraints before fabricating answers*."

## Must-internalize

- The "design Cloudflare's edge" answer is the universal template: anycast + L4 + L7 + cache.
- Always name 3 failure modes and how each is detected + recovered.
- Always name scale numbers per box (e.g., 25Mpps for Katran, 50Gbps for kTLS NGINX, 200K qps for DNS).
- A staff design always has observability per layer.
- Trade-offs > correctness — graders reward *named choices*, not perfect answers.