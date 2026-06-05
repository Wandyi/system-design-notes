# Linux Networking — Staff Software Engineer Interview Deep Dive

End-to-end prep for a **Staff Software Engineer / Linux Networking** loop (Linux Foundation, kernel-adjacent role, cloud networking infra, CDN/edge company, hyperscaler SDN — any role where the bar is *"this person owns the kernel network stack of a production system."*).

The content is organized as **22 numbered files + this README**. Each file is self-contained: read out of order, hit the weak spots, but read `02` first if you've never traced a packet through the Linux stack.

The depth target is **staff-level**: you should be able to name the function in `net/ipv4/tcp_input.c`, the kernel version that introduced it, the sysctl that tunes it, and the production scenario that motivated the change. Where I cite RFCs, kernel function names, or commit SHAs, they are real; verify against `git log` on `linux.git` if anything looks off.

## How to use this pack

- **6+ weeks out**: read `01` → `04` to internalize the stack mental model + TCP/UDP internals. Trace one connection on `tcpdump` + `bpftrace` per day until you can sketch the data path blind.
- **3–4 weeks out**: deep dive on operational layers (`05`–`15`). Build a netns lab and do every example yourself: `ip netns add`, `tc qdisc`, `xdp-loader`, `nft`, etc.
- **1–2 weeks out**: case studies (`19`), corner-case + alternatives (`20`), system-design prompts (`21`), coding (`22`).
- **Last 48 hours**: `23-staff-topics-and-cheatsheets.md`, the 60-second pitches in this README, and your STAR stories.

## File map

| # | File | What it covers |
|---|------|----------------|
| 01 | [01-role-and-interview-context.md](01-role-and-interview-context.md) | LF / kernel-adjacent role, interview loop, level expectations, signals graders look for |
| 02 | [02-network-stack-architecture.md](02-network-stack-architecture.md) | Packet's journey RX→TX, `sk_buff`, `net_device`, NAPI, layers, kernel-space vs userspace |
| 03 | [03-tcp-deep-dive.md](03-tcp-deep-dive.md) | TCP state machine, congestion control (Reno/Cubic/BBR), RTO, RACK, TFO, SACK, MPTCP intro |
| 04 | [04-udp-quic-and-dns.md](04-udp-quic-and-dns.md) | UDP path, GSO/GRO for UDP, QUIC userspace + kernel offload, DNS perf, recvmmsg |
| 05 | [05-socket-api-and-options.md](05-socket-api-and-options.md) | `socket(2)`, `bind/listen/accept`, options (`SO_REUSEPORT`, `TCP_FASTOPEN`, `SO_TIMESTAMPING`), AF_PACKET, AF_UNIX |
| 06 | [06-io-multiplexing.md](06-io-multiplexing.md) | `select`/`poll`/`epoll` (level vs edge), io_uring, comparison rubric, thundering herd |
| 07 | [07-zero-copy-and-offloads.md](07-zero-copy-and-offloads.md) | `sendfile`, `splice`, `MSG_ZEROCOPY`, TSO/GSO/LRO/GRO, RSS/RPS/RFS, XPS, NIC offloads |
| 08 | [08-network-namespaces-and-veth.md](08-network-namespaces-and-veth.md) | netns, veth, bridge, macvlan/ipvlan, container networking, CNI taxonomy |
| 09 | [09-netfilter-iptables-nftables.md](09-netfilter-iptables-nftables.md) | netfilter hook points, iptables vs nftables, conntrack, NAT, kube-proxy modes |
| 10 | [10-routing-and-policy.md](10-routing-and-policy.md) | FIB, multi-table routing, `ip rule`, ECMP, source-based routing, BGP integration, anycast |
| 11 | [11-ebpf-xdp-and-tc.md](11-ebpf-xdp-and-tc.md) | eBPF verifier, XDP (native/offload/skb), tc-bpf, eBPF maps, Cilium/Katran case studies |
| 12 | [12-qdisc-and-traffic-control.md](12-qdisc-and-traffic-control.md) | tc, classful/classless qdiscs, fq, fq_codel, CAKE, BBR pacing, HTB, ingress shaping |
| 13 | [13-bonding-mptcp-and-failover.md](13-bonding-mptcp-and-failover.md) | bonding modes, teamd, MPTCP, ECMP, BFD, anycast failover, ARP/ND quirks |
| 14 | [14-tls-ktls-and-quic.md](14-tls-ktls-and-quic.md) | TLS handshake, kTLS (TX + RX offload), session resumption, 0-RTT, NIC TLS offload |
| 15 | [15-rdma-dpdk-af_xdp.md](15-rdma-dpdk-af_xdp.md) | Kernel bypass spectrum, RDMA (RoCE/iWARP/IB), DPDK, AF_XDP, when each wins |
| 16 | [16-observability-and-debugging.md](16-observability-and-debugging.md) | tcpdump, ss, ip, conntrack, ethtool, perf, ftrace, bpftrace, drop_monitor, mlxsw |
| 17 | [17-perf-tuning-and-sysctls.md](17-perf-tuning-and-sysctls.md) | Every sysctl that matters: TCP buffers, `net.core.*`, `tcp_*`, IRQ pinning, NUMA |
| 18 | [18-load-balancing-and-l4-l7.md](18-load-balancing-and-l4-l7.md) | DSR, Maglev consistent hashing, LVS, Katran, Envoy, NGINX, anycast LB topologies |
| 19 | [19-case-studies-and-realistic-scenarios.md](19-case-studies-and-realistic-scenarios.md) | Cloudflare, Netflix, Facebook, Google, AWS — concrete production scenarios |
| 20 | [20-corner-cases-and-alternatives.md](20-corner-cases-and-alternatives.md) | The "name 3 ways to solve this" file: head-of-line blocking, slow start, half-closed, TIME_WAIT, conntrack overflow, SYN flood |
| 21 | [21-system-design-questions.md](21-system-design-questions.md) | 12 design prompts with full answers (L4 LB, CDN edge, anycast DNS, service mesh, etc.) |
| 22 | [22-coding-and-syscall-problems.md](22-coding-and-syscall-problems.md) | C/Go problems: TCP echo, epoll server, raw socket, XDP filter, parser, queue |
| 23 | [23-staff-topics-and-cheatsheets.md](23-staff-topics-and-cheatsheets.md) | Tech leadership scope + last-48-hours flashcards: sysctls, ports, RFCs, latencies |

## Why Linux networking matters for this role

Staff at a kernel-adjacent role (Linux Foundation, hyperscaler SDN, CDN, cloud-VPC team) is the person who:

- Can debug a 30Gbps packet-drop incident at 2am with `ss`, `ethtool -S`, `bpftrace`, and the dropwatch tool.
- Can sketch a load-balancing tier (DSR + Maglev + anycast) on a whiteboard with the failure modes named.
- Can decide between iptables, nftables, eBPF, and XDP for a kube-proxy-class problem and justify each.
- Can read `net/ipv4/tcp_output.c` and explain when `tcp_write_xmit()` actually pushes a packet.
- Can sit between a network engineer (BGP, MPLS, QoS) and a kernel hacker (skb, NAPI, GRO) and translate.

This pack focuses on the **kernel + edge** view. Application-layer (HTTP/gRPC) and pure SRE topics are touched but not central.

## What "Staff" means in a kernel-networking role

| Track | Title | Scope |
|-------|-------|-------|
| L5 | Staff SWE | Owns a sub-system (e.g., conntrack scaling, TLS termination), influences a project (e.g., kube-proxy → Cilium migration), mentors senior engineers |
| L6 | Sr. Staff | Owns the architecture of a multi-team area (e.g., the entire L4/L7 load balancing tier) |
| L7 | Principal | Cross-org technical strategy, sets multi-year direction, represents the company at LPC / Netconf |

For Linux Foundation specifically — most kernel maintainers are employed by member companies (Red Hat, Google, Meta, Intel, IBM, ARM), not LF directly. LF the org hires SWEs to run **CI infrastructure (LFX), project-specific eng (CNCF projects, LF Networking projects like FD.io VPP, ONAP, Tungsten Fabric), and developer experience**. Don't bluff — ask the recruiter exactly what the team builds. The networking depth you'll be tested on is the same regardless.

## The 60-second elevator pitch — the Linux network stack

> "A packet arrives at a NIC. The driver moves it via DMA into a ring buffer; the NIC fires an IRQ. NAPI kicks in: the IRQ handler schedules a soft-IRQ (`NET_RX_SOFTIRQ`) which runs `napi->poll()` in `softirqd` context, draining the ring up to `netdev_budget` packets. Each packet is wrapped in an `sk_buff` and pushed up: GRO coalesces small packets into one larger one, RPS may steer it to another CPU's backlog, then it hits `__netif_receive_skb()` and walks the protocol handlers (`ip_rcv` → `ip_local_deliver` → `tcp_v4_rcv` for TCP). TCP demultiplexes by 4-tuple, drops it into the socket's `sk_receive_queue`, wakes the userspace process. On TX, userspace `write()` lands in `tcp_sendmsg()`, which copies (or zero-copies) the payload, builds a TCP segment, hands it to IP (`ip_queue_xmit`), which routes (FIB lookup), netfilter-filters (OUTPUT/POSTROUTING hooks), then `dev_queue_xmit()` hands it to a qdisc (default `fq_codel`), which dequeues to the driver via `ndo_start_xmit`. The driver DMA-maps the skb and rings the doorbell. The middle is full of escape hatches: XDP runs *before* skb allocation (line-rate drop/redirect), eBPF at tc/socket/cgroup hooks, kTLS bypasses userspace TLS, `MSG_ZEROCOPY` skips the copy, io_uring skips the syscall, AF_XDP gives userspace zero-copy. Half of staff work is knowing which escape hatch matches the problem."

## The 60-second pitch — TCP/IP

> "TCP is a state machine (RFC 793) over a sliding-window reliable transport. The send side maintains `snd_una/snd_nxt/snd_wnd`; receive side `rcv_nxt/rcv_wnd`. Reliability uses cumulative ACKs + SACKs (RFC 2018) for selective retransmission. Congestion is signaled by loss (Reno, Cubic) or RTT increase (BBR, Vegas). Loss detection is RTO (fallback) or RACK (time-based, RFC 8985, the modern default). Flow control is the receive window; congestion control is `cwnd`. The kernel auto-tunes both buffer sizes via `tcp_rmem`/`tcp_wmem`. Modern Linux ships **Cubic** by default and **BBR v3** as the heavy-traffic choice. The escape hatches are TFO (zero-RTT data, RFC 7413), MPTCP (multiple subflows, RFC 8684), and the upcoming move-to-QUIC for everything that can't pay for the TCP overhead."

## The 60-second pitch — the modern fast-path

> "Three layers of bypass exist. **kTLS** moves crypto to the kernel (or NIC) so `sendfile()` can stream encrypted bytes — Netflix uses this to serve 800Gbps from a single box. **XDP** runs an eBPF program *before* `skb` allocation, lets you drop/forward at line-rate; Cloudflare uses it for DDoS, Facebook uses Katran (XDP-based L4 LB) to replace IPVS. **AF_XDP** gives a userspace ring direct access to the NIC RX queue with kernel-managed buffers — the half-way house between kernel and DPDK. Beyond that: **DPDK** owns the NIC, polls in userspace, no kernel involvement — VPP and many telco data planes. **RDMA** (Infiniband / RoCE / iWARP) does memory-to-memory with the NIC, no CPU. The selection rubric is by latency floor and ops cost: tc/iptables (µs, easy ops) → eBPF/XDP (sub-µs, medium ops) → AF_XDP (ns, more ops) → DPDK/RDMA (ns, full stack control)."

## High-frequency topic clusters (what they actually probe)

| Cluster | Where to study | Probability |
|---------|----------------|-------------|
| Packet path RX/TX + sk_buff lifecycle | `02` | **Very high** |
| TCP internals (state machine, congestion, retransmit) | `03` | **Very high** |
| epoll vs io_uring tradeoffs | `06` | **Very high** |
| netfilter / nftables / conntrack | `09` | High |
| eBPF + XDP design patterns | `11` | **Very high** (any modern role) |
| Container networking (netns, veth, CNI) | `08` | **Very high** (cloud/k8s roles) |
| Performance tuning, sysctls, NIC offloads | `07`, `17` | High |
| Observability + debugging tools | `16` | **Very high** (staff signal) |
| L4/L7 LB design (DSR, Maglev, anycast) | `18`, `21` | **Very high** |
| Kernel bypass (DPDK / AF_XDP / RDMA) | `15` | Medium-high (edge/HFT/HPC roles) |
| Corner-case + alternative-solution recall | `20` | **Very high** (every interview) |

## "What signals do interviewers grade on?"

Past staff interviewers from kernel-adjacent companies cite four discriminators:

1. **Vocabulary precision.** "ACK" vs "cumulative ACK" vs "SACK". "iptables" vs "netfilter". "MTU" vs "MSS" vs "PMTU". Confusing these tells the room you've read about networking, not done it.
2. **Layer awareness.** When asked "why is this slow," staff candidates say *which layer* before diagnosing. Driver/IRQ vs softirq backlog vs qdisc vs TCP vs application is a 5-line decision tree, not a guess.
3. **Failure-mode storytelling.** Junior: "the system uses anycast for HA." Staff: "anycast fails the routing announcement when the BGP session breaks; we have BFD on the underlay for sub-second detection, plus a healthcheck that withdraws the route if the data-plane fails — but conntrack flow state is *not* shared, so an in-flight TCP connection rebalanced to another POP gets RST."
4. **Tool fluency.** "How would you debug this?" gets a *specific tool list* with the order — `ss -tin`, `ethtool -S`, `tcpdump -i any -nn`, `bpftrace -e 'tracepoint:tcp:tcp_retransmit_skb { ... }'`, `dropwatch`. Junior says "I'd add logging."

## A note on currency

The Linux kernel networking stack moves fast. Key recent (2023–2026) developments to be aware of:

- **MPTCP** is mainline since 5.6, in production at Apple, Samsung; expect questions on use cases vs. application-layer (HTTP/3) multipath.
- **io_uring** continues to expand — `IORING_OP_SEND_ZC` is the modern zero-copy send; `IORING_OP_PROVIDE_BUFFERS` for buffered I/O; some networks use io_uring exclusively for hot-path now.
- **BBRv3** has shipped; question on Cubic vs BBR-vN is common.
- **AF_XDP** is now stable and used in production (Cloudflare, Cilium); the formerly experimental zero-copy mode is the default on most modern NICs (Mellanox CX-6, Intel E810).
- **Cilium / eBPF** has displaced kube-proxy in most modern Kubernetes installs. Cilium 1.16+ has full mesh support without sidecars.
- **kTLS RX** (not just TX) is widely available; 5.0+ for TX, 5.2+ for RX, hardware offload on Mellanox ConnectX-6 Dx and newer.
- **DCTCP / DCQCN** for DC fabrics; **RoCEv2** with PFC + ECN; expect a question on RDMA congestion control if the role is HPC/AI-cluster.
- **Wireguard** is in mainline (5.6); the simplification vs IPsec is a frequent comparison.

Don't bluff currency. If asked about something post-cutoff, "I've read mainline through about $DATE; happy to reason from primitives" is a far better answer than fabricating a kernel version.

## Sources used to build this pack

- Linux kernel source — `net/`, `Documentation/networking/`
- *Understanding Linux Network Internals* (Christian Benvenuti)
- *TCP/IP Illustrated, Vol 1 & 2* (Stevens; for protocol, RFCs)
- LPC (Linux Plumbers Conference) talks: NAPI, XDP, MPTCP, kTLS sessions
- Netdev 0x## conference proceedings (the canonical kernel-networking conference)
- Cloudflare blog (Marek Majkowski's posts on XDP, eBPF, conntrack; "How to receive a million packets")
- Netflix blog (Drew Gallatin's 800Gbps post, kTLS posts)
- Facebook Engineering (Katran, BPF stories)
- Google Cloud (Espresso, Andromeda, Maglev paper)
- Cilium docs, Calico docs, kube-proxy modes documentation
- Brendan Gregg — *BPF Performance Tools*; LWN.net (`lwn.net/Kernel/Index/`)
- man pages: `tcp(7)`, `ip(7)`, `socket(7)`, `epoll(7)`, `io_uring(7)`, `netlink(7)`, `bpf(2)`