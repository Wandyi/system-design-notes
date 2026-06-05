# 19 · Case Studies and Realistic Scenarios

The "what really happens in production" file. Each case study comes from the public engineering blog of a major company; each lesson generalizes. Memorize at least one story per category.

## 19.1 Cloudflare — "How to Receive a Million Packets per Second" (Marek Majkowski, 2015)

**Problem**: a single Linux box receiving 1Mpps single-flow UDP. Default config: ~370Kpps.

**Findings**:
- Single 4-tuple hash → all packets to one CPU → softirq saturates.
- `SO_REUSEPORT` (3.9+) plus N userspace workers each bound to same port → kernel hashes incoming SYNs/datagrams across workers.
- IRQ pinning + RPS for the multi-CPU case.
- Final: 1Mpps achieved with 8 workers + correct IRQ affinity.

**Lesson**: per-CPU bottleneck is the universal pattern. Solution: shard at every layer — RSS, RPS, SO_REUSEPORT, conntrack per-CPU.

**Interview applications**: any "how do you scale to N CPUs" question. Cite the workers + REUSEPORT + IRQ pinning recipe.

## 19.2 Cloudflare — "BPF, the Forgotten Bytecode" + Unimog L4 LB

Cloudflare's L4 LB ("Unimog") is XDP-based, similar to Katran. Maglev consistent hashing in BPF maps. Encapsulates with GUE (Generic UDP Encapsulation, not IPIP).

**Why GUE**: UDP-encap is friendlier to middleboxes than IPIP; some ECMP hashers don't handle IPIP well.

**Scale numbers**: ~25Mpps per host at line rate.

**Lesson**: XDP is the kernel's response to dedicated LB ASICs; affordable scale-out.

## 19.3 Cloudflare DDoS — XDP_DROP at line rate

A 71M req/s attack landed in 2022. Cloudflare's defense: XDP programs running on every NIC drop traffic matching known-bad signatures before sk_buff allocation.

**Architecture**:
- Threat intelligence pushes signatures to BPF maps.
- XDP programs match (rate limits per source, malformed packets, specific signatures) → `XDP_DROP`.
- CPU cost: nominal even at Mpps drop rate.

**Lesson**: drop early. The closer to the wire, the cheaper. iptables at 10Mpps is unworkable; XDP shrugs.

## 19.4 Netflix — 800Gbps from a single 4U box (Drew Gallatin, 2021 EuroBSDcon)

The famous talk. Netflix's "open connect appliances" (OCAs) serve ~95% of their traffic. They needed ~200Gbps per box, then ~400Gbps, then ~800Gbps.

**Stack**:
- FreeBSD (not Linux! but architecturally the same lessons).
- NGINX serving video files.
- TLS — required by clients.
- **kTLS sendfile**: kernel encrypts on the path from page cache to NIC.
- **NIC TLS offload**: Mellanox CX-6 Dx does AEAD in HW.
- Many 100GbE NICs (8 × 100GbE = 800 nominal; ~95% utilization).

**Bottlenecks they hit**:
- Userspace TLS limited to ~75Gbps (CPU-bound on AES copy).
- kTLS without HW: ~400Gbps (AES-NI is the limit).
- kTLS + HW offload: 800Gbps achieved.

**Lesson**: at line rate, the data must never touch the CPU. kTLS + sendfile + NIC offload is the only way TLS scales to 1Tbps.

## 19.5 Facebook Katran — XDP-based L4 LB

**Problem**: IPVS-based L4 LB hit limits at ~100k+ servers. Required dedicated boxes with low utilization.

**Solution**: Katran (2018, open-sourced). XDP program at every server's NIC.

**Architecture**:
```
   Client ──► Anycast IP ──► border router ECMP ──► (any) server's NIC
                                                          │
                                                          ▼
                                                  Katran XDP program
                                                  consistent-hash to real backend
                                                  IPIP encapsulate
                                                  XDP_TX (bounce back to NIC)
                                                          │
                                                          ▼
                                                 Real backend (decapsulates)
                                                 Direct reply to client
```

**Scale numbers**: handles all Facebook's traffic. Each box ~10× cheaper per pps than dedicated LB.

**Lesson**: dedicated LB hardware is being replaced by commodity-server-with-XDP. Maglev hashing is the consistent-flow trick.

## 19.6 Google Maglev — the paper (2016 NSDI)

The reference paper for software L4 LB. Predates XDP — runs in user space with kernel-bypass (presumably DPDK-like internally).

**Key ideas**:
- Consistent hashing via the *Maglev table* (size 65537 typical) — flow → backend.
- Backend churn shuffles only ~M/N entries → minimal flow disruption.
- N-tuple hash key (5-tuple usually).
- Stateless within an LB instance (re-derive on every packet) — enables scale-out.

**Performance**: "tens of Gbps" per server (in 2016; today probably 100s).

**Lesson**: consistent hashing > stateful flow tracking for scale.

## 19.7 Google Espresso — BBR's birthplace (2017 SIGCOMM)

Espresso is Google's peering edge stack. Routes Google traffic onto the internet from various peering POPs.

- Custom hashing for path selection.
- **BBR congestion control** — born here for use over WAN to peers.

**Lesson**: at the edge, classic Cubic underperforms vs BBR for long-RTT bulk traffic. Google's experimental rollout informed BBR's mainline addition.

## 19.8 Facebook eBPF observability

Facebook has the largest deployed BPF estate. Used for:
- Per-server flow accounting.
- Slow-syscall detection (tracepoints).
- Container introspection.
- DDoS signature matching.

**Lesson**: eBPF replaced ad-hoc kernel modules for observability; one tool covers many needs.

## 19.9 Cilium / k8s — replacing kube-proxy

**Problem**: kube-proxy iptables mode hits limits at ~5K services × 10 endpoints = 50K rules. Reload-during-deploy = 30s of degraded service routing. CPU per packet O(N).

**Solution**: Cilium. eBPF programs at tc-ingress/egress, socket, cgroup. Service map is a BPF hash; ~O(1) lookup. Maglev hashing for endpoint selection.

**Scale numbers**: tested to ~500K services, ~1M endpoints, with no perf regression.

**Lesson**: BPF maps + tail-calls replace iptables at scale.

## 19.10 LinkedIn — D2 / Rest.li (older story)

LinkedIn's pre-mesh service-to-service: D2 client-side LB over ZooKeeper-discovered backends. Rest.li the RPC layer.

**Mechanism**:
- Service registers in ZK on startup.
- Client subscribes to service path; gets current backend list.
- Client-side LB (round-robin, weighted, sticky).
- Kafka/RPC over the resulting connections.

**Lesson**: client-side LB has lower latency (no extra hop) but higher complexity (every client needs the registry). Service mesh sidecars are the modern compromise.

## 19.11 Cilium Service Mesh — ambient/sidecarless (2022+)

Where service mesh was going: every pod gets a sidecar Envoy. ~50MB+ memory each. ~1ms latency added.

Cilium's pitch: do L4 mesh entirely in eBPF (no sidecar). Add an Envoy-per-node only for L7 features (when needed).

**Scale numbers**: 90% lower memory, 50% lower latency for L4-only traffic.

**Lesson**: ambient mesh is the cost-effective answer when L7 features aren't needed for every flow.

## 19.12 AWS — anatomy of NLB and ALB

**NLB (Network Load Balancer)**: L4, ALB-on-anycast, flow-aware. Backed by Hyperplane (custom dataplane).

**ALB (Application Load Balancer)**: L7, HTTP/2/gRPC, WAF integration.

**Key**: NLB preserves client source IP (DSR-like). ALB doesn't (proxy).

**Scale**: tens of millions of qps per LB. Cross-zone optional (NLB) or default (ALB).

## 19.13 The microbenchmark — Cloudflare's go-test for QUIC

Cloudflare reported (2020) that QUIC was slower than HTTP/2 because of UDP per-packet syscall overhead. UDP GSO/GRO (4.18 kernel) closed the gap.

**Numbers**: UDP-GSO bumped per-CPU QUIC throughput from 800Mbps to 4.5Gbps.

**Lesson**: a "modern transport" is only fast on a modern kernel. Old kernel + QUIC = sad CPU. Always confirm kernel version.

## 19.14 Real war story — conntrack overflow in production

**Incident**: a Kubernetes cluster running 50 nodes, each with default `nf_conntrack_max=256K`. New service deployed: opens 100K connections to upstream database per minute (badly-sized connection pool).

**Symptoms**: random connection resets cluster-wide; latency spikes; logs show `nf_conntrack: table full, dropping packet`.

**Investigation**:
- `conntrack -S` shows `insert_failed` rising on every node.
- `/proc/sys/net/netfilter/nf_conntrack_count` near max.
- `tcp_timeout_established` at default 5 days → flows can't recycle fast enough.

**Fixes (applied in order)**:
1. Raise `nf_conntrack_max` to 2M per node. Quick patch.
2. Shorten `tcp_timeout_established` to 1d, `time_wait` to 30s.
3. Long-term: switch to Cilium (eBPF flow tracking, bypasses conntrack for cluster traffic).

**Lesson**: conntrack is invisible-until-it's-not. Tune defaults proactively.

## 19.15 Real war story — single-flow elephant

**Incident**: a 100Gbps NIC running at 30Gbps. CPU graphs show one core at 100% softirq.

**Investigation**:
- `mpstat -P ALL 1` confirms one CPU pegged.
- `cat /proc/interrupts | grep eth0` shows one queue's IRQ count rising; others idle.
- `ss -tin` shows one connection doing 25Gbps.
- RSS hash routes that 4-tuple to one queue.

**Fixes**:
1. Enable RPS on the affected queue to spread to other CPUs.
2. App-level: split the connection into multiple parallel TCP streams (each with its own 4-tuple) — gets RSS to spread.
3. Investigate why one TCP connection. (Often: bad design; multi-stream is the right answer.)

**Lesson**: RSS doesn't help single-flow. Either multi-stream the app or accept the per-CPU limit.

## 19.16 Real war story — TIME_WAIT explosion

**Incident**: a batch job opens millions of connections to a database; gets `EADDRNOTAVAIL` errors after a while.

**Investigation**:
- `ss -tan state time-wait | wc -l` shows 300K.
- `cat /proc/sys/net/ipv4/ip_local_port_range` shows 28K available ports.
- Job has only one source IP.

**Fixes**:
1. Connection pooling — the proper fix.
2. `tcp_tw_reuse=1` — kernel allows reuse if older than 1s.
3. Multiple source IPs (bind to different).

**Lesson**: TIME_WAIT is by design; pooling avoids it; `tw_reuse` is the safety net.

## 19.17 Real war story — PFC storm in DC

**Incident**: a RoCE-using DC fabric goes from 1Tbps aggregate to ~100Gbps. Latency for non-RoCE traffic explodes.

**Investigation**:
- Switch counters show massive Priority Pause Frames.
- One workload sends bursty 100Gbps RDMA from 100 senders simultaneously → microburst at uplink → PFC pause back upstream → cascades.
- Pause propagation deadlock briefly.

**Fixes**:
1. Tune DCQCN (Mellanox congestion algorithm) parameters.
2. Move some traffic off the lossless priority class.
3. Increase switch buffer per port.
4. Look at ECN-only options instead of PFC.

**Lesson**: lossless ethernet is hard. RoCE deployments live and die by PFC tuning.

## 19.18 Real war story — Bufferbloat on home internet

**Incident**: an interactive ssh session over a home cable modem stalls when family member starts watching Netflix.

**Investigation**:
- Cable modem has a huge buffer (~1MB).
- Netflix download fills it.
- ssh packets sit behind 1MB queue → seconds of delay.

**Fix**: install OpenWrt on home router; enable CAKE qdisc with bandwidth set to slightly less than cable modem's downlink. CAKE drops early; modem buffer never fills.

**Lesson**: bufferbloat is real; CAKE/fq_codel are the cure. Same principle applies in DC (microbursts → tail latency).

## 19.19 Real war story — SYN flood

**Incident**: a public-facing server suddenly stops accepting new connections; existing ones fine.

**Investigation**:
- `ss -lnt` shows accept queue full.
- `nstat -az TcpExtListenOverflows` rising.
- `tcpdump 'tcp[tcpflags] & tcp-syn != 0'` shows ~1Mpps SYNs from many spoofed sources.
- `cat /proc/sys/net/ipv4/tcp_syncookies` is 1 (good).
- But `tcp_max_syn_backlog` is small.

**Fixes**:
1. Bump `tcp_max_syn_backlog` to 65536.
2. Already have syncookies on — these handle full backlog.
3. XDP drop at NIC for clearly-malicious sources.
4. Long-term: anycast + DDoS mitigation upstream.

**Lesson**: SYN flood is mitigated by syncookies (since the 90s); the bottleneck is the syn-backlog size; XDP is the modern line-rate drop.

## 19.20 Real war story — DNS rate explosion at AWS Route 53

**Incident**: a Lambda function had a bug causing it to DNS-query every API call instead of caching. Rate hit 100K qps per region. Costs and latency exploded.

**Investigation**:
- CloudWatch shows DNS query rate × 1000.
- `dig +trace` shows zone has TTL 60s; that's fine, but Lambda re-resolves every call.
- Local resolver cache disabled (or no glibc-cache-equivalent).

**Fix**:
1. Application-level cache.
2. Use `getaddrinfo` with appropriate TTL behavior.
3. Run a local NSS-cache (nscd, systemd-resolved) on each box.

**Lesson**: DNS is implicit infrastructure; assume nothing caches.

## 19.21 The "design Cloudflare's edge" interview

A full system-design example. Walk through:

```
Client (DNS resolves) ──► anycast IP (BGP)
                              │
                              ▼
                       Cloudflare POP nearest client
                              │
                              ▼
                       Top-of-rack switch ECMP
                              │
                              ▼
                       L4 LB (Unimog XDP)         ← consistent-hashes by 5-tuple
                              │
                              ▼
                       NGINX with kTLS + sendfile  ← terminates TLS
                              │
                              ▼
                       Cache (BPF maps + RAM + SSD)
                              │
                              ▼ (miss)
                       Origin (customer's server)
```

Failure modes:
- POP loses fiber: anycast withdraws; nearest POP takes over (sub-second with BFD).
- LB fails: ECMP rebalances; Maglev minimizes flow disruption.
- NGINX overloaded: outlier detection ejects; new flows go elsewhere.
- Origin slow: cache absorbs; stale-while-revalidate.

This is the kind of full-stack answer staff want. Memorize it.

## 19.22 The takeaway patterns

| Pattern | Where |
|---------|-------|
| Anycast + L4 (Maglev) + L7 (Envoy) | Cloudflare, Facebook, Google |
| XDP for drop / LB at line rate | Katran, Unimog, DDoS defense |
| kTLS + sendfile for line-rate TLS | Netflix OCA |
| eBPF for observability | Facebook BPF, Cilium Hubble, Pixie |
| Consistent hashing (Maglev) for LB | Everywhere |
| SO_REUSEPORT + N workers | Anything high-pps |
| Conntrack tuning + eBPF bypass | High-rate proxies, k8s |
| Multi-stream TCP for elephant flows | Big bulk transfer |

## Must-internalize

- One war story per topic: 1Mpps Cloudflare; 800Gbps Netflix; Katran; conntrack overflow; bufferbloat; SYN flood.
- The pattern is always: shard across CPUs; drop early; tune buffers to BDP; consistent-hash flows; offload TLS to kernel/NIC.
- Quote a real production number (10Mpps, 100Gbps, 50K services) when discussing scale.
- The "design Cloudflare's edge" answer is the canonical staff-level system design — practice it cold.