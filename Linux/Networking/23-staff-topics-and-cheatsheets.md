# 23 · Staff Engineer Topics and Cheatsheets

The cross-cutting "what does staff look like" file + the night-before flashcards. Read this last.

## 23.1 Scope and leadership signals

Staff engineering in a kernel-networking role is a strange beast. Unlike app-layer where staff often means a 10-person team's lead, here scope tends to be:

- **Owner of a critical subsystem** in the data path (e.g., L4 LB, kernel TLS, conntrack scaling, CNI plugin internals).
- **Maintainer of an OSS component** with external visibility (Cilium component, BPF tool, kernel netdev patch series).
- **Reviewer of architecture for adjacent teams** when their changes touch the network.

A staff engineer should be able to:

1. **Read code others wrote** (kernel, Cilium, Envoy) and explain it.
2. **Write a design doc** that survives review from kernel/network-cynical reviewers.
3. **Run an incident** — be the technical lead during a complex outage.
4. **Mentor seniors** — including the awkward "this is the wrong abstraction" conversation.
5. **Bridge** — translate between network engineering (BGP, MPLS) and kernel hacker (skb, NAPI).

## 23.2 Migration / deprecation campaigns

A staff-level project is rarely "build feature X" — it's "decommission the old way and migrate." Examples:

- **iptables → nftables**. Multi-quarter, every distro, every internal tool.
- **kube-proxy iptables → Cilium / IPVS**. Per cluster; risk-managed.
- **IPVS L4 LB → XDP-based**. Cost savings + new failure modes.
- **Voldemort → Espresso** (LinkedIn). Years.
- **Open Connect FreeBSD pre-kTLS → kTLS** (Netflix). Performance wins.

Staff-level execution:
- Build a parallel path; don't yank the old.
- Per-cell / per-cluster rollout with rollback.
- Telemetry to compare; declare "old is dead" when metrics agree.
- Document for posterity; the next staff engineer will inherit.

## 23.3 Build vs OSS vs buy

The staff call: "should we build, adopt OSS, or buy?"

Framework:
- **Buy** when commodity (TLS termination = AWS NLB / ALB).
- **OSS** when active and trusted (Envoy, Cilium, NGINX, Linux kernel).
- **Build** when (a) OSS doesn't fit, (b) competitive moat, (c) you have the bandwidth.

Anti-patterns:
- "We rolled our own LB" — usually a smell unless you're Google/Cloudflare scale.
- "We use this random GitHub project" — auditing required.
- "Buy and customize heavily" — vendor lock + lost agility.

The framework matters more than the answer.

## 23.4 Working with open source

If the role is at LF or kernel-adjacent:

- Know the **maintainer model**: who can merge, what's the cadence (Linux: rc1-7 then release; tagged every 2 months).
- **Mailing list etiquette**: terse, reply-to-all, no top-posting (kernel mailing lists strict).
- **Patch series**: 1-2-3 patches, each independently bisectable. Cover letter explains why.
- **Test infrastructure**: kernel CI is multi-arch, all configs. Don't break alpha.

For interview: "tell me about a patch you tried to upstream" is a likely behavioral.

## 23.5 The on-call story

Staff is on-call. Have a story:
- The page came in.
- The symptom (be specific: "p99 latency to backend X spiked from 5ms to 500ms").
- Initial hypotheses tested in order.
- The actual cause (often subtle: conntrack overflow, single-CPU bottleneck, BGP flap).
- The fix (immediate, then long-term).
- Process change.

The grader hears: "this person can be trusted with prod."

## 23.6 ADRs (Architecture Decision Records)

For staff: write ADRs. Format:

```
# ADR-007: Switch from kube-proxy iptables to Cilium

Status: Accepted

## Context
- 5000 services × 10 endpoints = 50K iptables rules per node.
- Per-rule scan = O(N) latency.
- Reload during deploy = 30s of degraded service routing.

## Decision
- Adopt Cilium 1.16 with kube-proxy replacement.
- Per-cluster rollout, starting with non-prod.

## Consequences
- Service routing latency drops ~5×.
- Operational learning curve (eBPF, Hubble UI).
- Migration cost: ~1Q of effort per environment.

## Alternatives considered
- IPVS mode (partial improvement, retains iptables for NAT).
- Stay on iptables (untenable at 10K services projected next year).
```

Cite these in interview answers; signals "I document, I make decisions visible."

## 23.7 Mental model: what staff doesn't do

- **Doesn't write all the code.** They review, mentor, design.
- **Doesn't make every decision.** They set the framework; teams decide details.
- **Doesn't ignore careers of others.** Mentorship is a deliverable.
- **Doesn't bluff currency.** Kernel moves fast; admit unknowns; reason from primitives.

## 23.8 60-second elevator pitches (memorize all)

These are the templates for any prompt.

### Linux network stack

> "Packet RX: NIC → IRQ → softirq → NAPI poll → XDP hook → skb → GRO → IP rcv → L4 demux → socket. TX: socket → tcp_sendmsg → ip_queue_xmit → netfilter → qdisc → driver. Escape hatches: XDP (pre-skb), tc-bpf (post-skb), kTLS, MSG_ZEROCOPY, io_uring, AF_XDP, DPDK, RDMA."

### TCP

> "11-state machine. SYN/SYN-ACK/ACK handshake; SYN queue + accept queue per listen. Data via sliding window + cumulative ACK + SACK; loss detection via RACK (8985, modern default). Congestion: Cubic (default), BBR (high-RTT bulk), DCTCP (ECN datacenter). Flow control: receive window auto-tuned via tcp_rmem. Nagle + delayed-ACK = 40ms RPC stalls; TCP_NODELAY fixes."

### Failover

> "Layered. Link: LACP / active-backup bonding. IP: VRRP (active/passive), anycast (active/active). Transport: MPTCP for connection migration. Routing: BGP + BFD for sub-second. LB: Maglev for consistent flow assignment. App: idempotent retries, circuit breakers."

### Load balancer

> "Anycast → ECMP → L4 (Maglev/Katran, DSR) → L7 (Envoy/NGINX, TLS termination, routing). Each layer has its own failure mode and detection. Backends use consistent hashing; outlier detection ejects bad ones."

### eBPF / XDP

> "Verified bytecode in kernel. XDP: pre-skb (~30Mpps drop). tc-bpf: post-skb, more features. Maps share state with userspace. Famous uses: Katran (FB L4 LB), Cilium (CNI), Cloudflare DDoS. Verifier proves termination + memory safety; CO-RE via BTF for cross-kernel."

### Container networking

> "netns isolates everything network: interfaces, routes, conntrack, iptables. veth pairs bridge namespaces. CNI plugins choose overlay (flannel-vxlan), routed (Calico-BGP), or eBPF (Cilium). Service VIPs implemented as iptables DNAT (kube-proxy iptables), IPVS, or eBPF maps (Cilium)."

### Observability

> "Layered tools per layer. App: pprof, perf. Socket: ss -tin. Packet: tcpdump (or SPAN at >10G). Counters: nstat -az. NIC: ethtool -S. Kernel: bpftrace tracepoint:tcp:*. Drops: dropwatch / bpftrace skb:kfree_skb. Production: Prometheus + Grafana + Hubble/Pixie for eBPF auto-instrumentation."

## 23.9 Cheatsheet — high-leverage sysctls

```
# TCP buffers (size to BDP)
net.ipv4.tcp_rmem = 4096 131072 6291456   → bump max if cross-region
net.ipv4.tcp_wmem = 4096 16384  4194304
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216

# Listen queues
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 4096

# TCP behavior
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.tcp_mtu_probing = 1
net.ipv4.tcp_keepalive_time = 600        # was 7200(!)
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_congestion_control = bbr (or cubic)
net.core.default_qdisc = fq (or fq_codel)

# Conntrack
net.netfilter.nf_conntrack_max = 1048576
net.netfilter.nf_conntrack_tcp_timeout_established = 86400  # not 5 days

# Forwarding
net.ipv4.ip_forward = 1

# NAPI
net.core.netdev_budget = 600
net.core.netdev_max_backlog = 8192

# Ports
net.ipv4.ip_local_port_range = "1024 65535"

# ARP / neighbor (k8s nodes)
net.ipv4.neigh.default.gc_thresh1 = 32768
net.ipv4.neigh.default.gc_thresh2 = 65536
net.ipv4.neigh.default.gc_thresh3 = 131072
```

## 23.10 Cheatsheet — ports

| Service | Port | Protocol |
|---------|------|----------|
| FTP | 20/21 | TCP |
| SSH | 22 | TCP |
| Telnet | 23 | TCP |
| SMTP | 25/587 | TCP |
| DNS | 53 | UDP/TCP |
| DHCP server | 67 | UDP |
| DHCP client | 68 | UDP |
| TFTP | 69 | UDP |
| HTTP | 80 | TCP |
| NTP | 123 | UDP |
| NetBIOS | 137-139 | both |
| LDAP | 389 | TCP |
| HTTPS | 443 | TCP / QUIC |
| SMB | 445 | TCP |
| Syslog | 514 | UDP |
| LDAPS | 636 | TCP |
| FTPS | 990 | TCP |
| IMAP | 143/993 | TCP |
| POP3 | 110/995 | TCP |
| MySQL | 3306 | TCP |
| RDP | 3389 | TCP |
| PostgreSQL | 5432 | TCP |
| Redis | 6379 | TCP |
| Kafka | 9092 | TCP |
| Elasticsearch | 9200 | TCP |
| Prometheus | 9090 | TCP |
| Grafana | 3000 | TCP |
| HTTP-alt | 8080 | TCP |
| HTTPS-alt | 8443 | TCP |
| BGP | 179 | TCP |
| Open vSwitch | 6633 | TCP (OpenFlow) |
| WireGuard | (varies, often 51820) | UDP |
| QUIC | varies | UDP |
| RoCE v2 | 4791 | UDP |
| Kubernetes API | 6443 | TCP |
| Kubelet | 10250 | TCP |

## 23.11 Cheatsheet — RFCs

| RFC | Topic |
|-----|-------|
| 791 | IPv4 |
| 792 | ICMP |
| 793 | TCP (replaced by 9293) |
| 768 | UDP |
| 1122 | Host requirements |
| 1191 | PMTUD |
| 1323 / 7323 | TCP extensions (timestamp, window scale) |
| 2018 | TCP SACK |
| 2581 / 5681 | TCP congestion control |
| 2675 | IPv6 jumbograms |
| 2883 | DSACK |
| 4271 | BGP-4 |
| 4724 | BGP graceful restart |
| 4861 | IPv6 Neighbor Discovery |
| 5246 | TLS 1.2 |
| 5681 | TCP congestion control |
| 5798 | VRRP |
| 5880 | BFD |
| 6052 | NAT64 |
| 6298 | TCP RTO |
| 6675 | Recovery from loss |
| 6928 | initcwnd=10 |
| 7413 | TCP Fast Open |
| 7858 | DNS over TLS |
| 8200 | IPv6 |
| 8446 | TLS 1.3 |
| 8484 | DNS over HTTPS |
| 8684 | MPTCP v1 |
| 8985 | RACK + TLP |
| 9000 | QUIC |
| 9001 | QUIC + TLS |
| 9002 | QUIC loss recovery |
| 9114 | HTTP/3 |
| 9293 | TCP (replaces 793) |

## 23.12 Cheatsheet — latencies (memorize)

| Operation | Latency |
|-----------|---------|
| L1 cache hit | ~1 ns |
| L2 cache hit | ~4 ns |
| L3 cache hit | ~10 ns |
| Main memory | ~100 ns |
| QPI/UPI cross-socket | ~100 ns |
| PCIe round trip | ~500 ns |
| NIC RX DMA + IRQ | ~5 µs |
| Loopback (`lo`) RTT | ~5 µs |
| Within-rack RTT | ~10-50 µs |
| Within-datacenter | ~100-500 µs |
| Cross-continent (US-EU) | 80-100 ms |
| Cross-continent (US-Asia) | 150-200 ms |
| Disk seek | ~5 ms |
| SSD seek | ~100 µs |
| NVMe read 4KB | ~10 µs |
| Cross-AZ in AWS | ~1 ms |
| TCP handshake | 1 RTT (~1 ms LAN, ~100 ms WAN) |
| TLS 1.3 handshake | 1 RTT (0-RTT with resumption) |
| TLS 1.2 handshake | 2 RTT |
| QUIC handshake | 1 RTT (0-RTT with PSK) |

## 23.13 Cheatsheet — throughput estimates

| Path | Per CPU |
|------|---------|
| Kernel TCP | ~10 Gbps (single flow, tuned) |
| Kernel TCP + GSO/GRO | ~30-80 Gbps |
| BIG_TCP | ~30% more on 100Gbps IPv6 |
| kTLS sendfile | ~95% of TCP (AES-GCM) |
| XDP drop | ~30 Mpps |
| XDP forward | ~20-30 Mpps |
| AF_XDP | ~20-30 Mpps |
| DPDK | ~50 Mpps |
| RDMA | ~200 Gbps small msgs |
| iptables | ~3-5 Mpps (degrades with rules) |
| nftables | ~5-10 Mpps |
| IPVS | ~10 Mpps |
| NGINX HTTP (no TLS) | ~500K rps |
| Envoy HTTP | ~50-200K rps |
| Katran L4 LB (XDP) | ~25-30 Mpps |

## 23.14 Cheatsheet — kernel hot paths

| Function | File | Role |
|----------|------|------|
| `tcp_sendmsg` | net/ipv4/tcp.c | TX entry from app |
| `tcp_write_xmit` | net/ipv4/tcp_output.c | Decide when to send |
| `tcp_v4_rcv` | net/ipv4/tcp_ipv4.c | RX entry |
| `tcp_rcv_established` | net/ipv4/tcp_input.c | Fast path |
| `tcp_ack` | net/ipv4/tcp_input.c | ACK processing |
| `tcp_retransmit_skb` | net/ipv4/tcp_output.c | Retransmit |
| `napi_poll` | net/core/dev.c | NAPI dispatch |
| `dev_queue_xmit` | net/core/dev.c | TX from stack |
| `__netif_receive_skb` | net/core/dev.c | RX entry |
| `ip_rcv` | net/ipv4/ip_input.c | IP RX |
| `ip_queue_xmit` | net/ipv4/ip_output.c | IP TX |
| `nf_hook` | include/linux/netfilter.h | Netfilter dispatch |

## 23.15 Cheatsheet — tools by symptom

| Symptom | First command |
|---------|---------------|
| Connection slow | `ss -tin`, `bpftrace tracepoint:tcp:tcp_retransmit_skb` |
| Many retransmits | `nstat -az TcpRetransSegs`, `mtr` |
| NIC drops | `ethtool -S eth0` |
| One CPU pegged | `mpstat -P ALL 1`, `/proc/interrupts` |
| Conntrack full | `conntrack -S`, `/proc/sys/.../nf_conntrack_count` |
| Backlog overflow | `nstat -az TcpExtListenOverflows`, `ss -lnt` |
| TIME_WAIT pile | `ss -tan state time-wait \| wc -l` |
| CLOSE_WAIT pile | `ss -tn state close-wait` |
| Drops in stack | `dropwatch` / `bpftrace tracepoint:skb:kfree_skb` |
| TLS handshake slow | `openssl s_client -connect h:443 -timed`, perf |
| Routing wrong | `ip route get DEST` |
| Firewall wrong | `iptables -nvL`, `nft list ruleset`, `conntrack -L` |
| In container | `nsenter -n -t $pid` then any of the above |
| NUMA mismatch | `cat /sys/class/net/eth0/device/numa_node`; `mpstat -P ALL 1` |
| BGP issue | `vtysh` (FRR), `birdc`, `show ip bgp` |

## 23.16 Cheatsheet — the failure-mode reflex

When asked "what could go wrong?"

| Layer | Failure | Detection | Recovery |
|-------|---------|-----------|----------|
| Link | Cable cut | MII / BFD | LACP failover, BGP reroute |
| NIC | Hardware | dmesg, ethtool | Replace; multi-NIC bonding |
| IP | IP conflict | dmesg | DHCP rebind |
| Route | Black hole | `mtr`, `ip route get` | BGP withdrawal, route fix |
| ARP | Stale entry | `ip neigh show` | `ip neigh flush` |
| Conntrack | Full | `conntrack -S` | Raise max, shorten timeout |
| TCP | RTO | nstat retransmits | None (it's TCP) |
| TLS | Cert expired | error logs, cert monitor | Rotate; ACME automation |
| App | OOM | OOM killer; metrics | Cap memory; HPA scale |
| Pod | OOMKilled | k8s events | Right-size; investigate leak |
| Cluster | API server down | kubectl times out | etcd backup; restart API |
| Region | All down | external monitoring | Failover to other region |

## 23.17 Behavioral interview prep — STAR framework

Have ~6 stories ready:

1. **Conflict with maintainer / vendor** — kernel-community-flavored.
2. **A big migration** — old → new infrastructure.
3. **An incident you led** — page came in at 2am.
4. **A design you championed** — convince a skeptical org.
5. **A failure** — what you learned.
6. **A mentee you developed** — what changed.

Each: Situation, Task, Action, Result. Practice aloud. Time them at 90s.

## 23.18 Final reading list

If you have weeks left:

- `Documentation/networking/` in linux.git — official kernel docs.
- *TCP/IP Illustrated, Vol 1 & 2* (Stevens / Fall).
- *Understanding Linux Network Internals* (Christian Benvenuti).
- Brendan Gregg's *BPF Performance Tools*.
- Marek Majkowski's Cloudflare blog (every networking post).
- LWN.net networking articles.
- The Maglev (2016 NSDI), Espresso (2017 SIGCOMM), Andromeda (2018 NSDI) papers.

If you have a week:

- This pack + memorize the cheatsheets + run a lab.

## Must-internalize

- The 60-second pitches in this file — recite cold.
- The sysctl, RFC, port, latency, throughput tables — flash-card them.
- Have 6 STAR stories ready.
- Be specific. Vague is the enemy of staff signal.
- Trust the process; ask clarifying questions before fabricating answers.
- The grader is grading *how you think*, not *what you've memorized*. The cheatsheets make the substrate; your reasoning is the show.

---

You've got this. Read this pack twice; build a netns lab; trace a packet end-to-end on paper. Show up rested.