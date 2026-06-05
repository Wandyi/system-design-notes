# 20 · Corner Cases and Alternative Solutions

The "name three ways to solve this" file. The user prompt explicitly asked for this, and so does every staff interviewer. Per problem: the naive solution, what breaks, then 2–3 better alternatives with their tradeoffs.

A staff-level signal is the *reflex* of producing three options and picking one with reasoning.

## 20.1 Pattern: TCP TIME_WAIT pile-up

**Naive**: just close connections; suffer.

### Approach A: connection pool / keepalive
Reuse connections; avoid the close-then-reopen pattern.
- Pros: solves the problem at the source; reduces handshake cost too.
- Cons: requires app cooperation; pool sizing nontrivial.

### Approach B: `tcp_tw_reuse = 1`
Kernel reuses TIME_WAIT for outgoing if older than 1s.
- Pros: free; safe on modern kernels.
- Cons: only for outgoing; not all distros enable by default.

### Approach C: multiple source IPs
Bind socket via `SO_BINDTODEVICE` or `IP_BIND_ADDRESS_NO_PORT` across multiple source IPs.
- Pros: linearly scales port space.
- Cons: requires IP allocation; routing config.

### Approach D: tcp_tw_recycle (don't)
Was removed in 4.10 because it broke NAT (RFC 1323 timestamp clash with shared egress).
- **Anti-pattern**. Never use.

### Selection rubric

| Situation | Choice |
|-----------|--------|
| Client → fixed upstream | Pool |
| Egress proxy (HAProxy) | Pool + tw_reuse |
| Constraint: can't change client | tw_reuse + multi-IP |
| Burstable batch | accept TIME_WAIT load |

## 20.2 Pattern: CLOSE_WAIT pile-up

**Naive**: app didn't close fd → CLOSE_WAIT never advances.

### Approach A: fix the app (canonical)
Audit `recv()=0` paths; `close()` the fd.

### Approach B: TCP_USER_TIMEOUT
`setsockopt(IPPROTO_TCP, TCP_USER_TIMEOUT, ...)` — kernel aborts if no progress.
- Pros: defensive; protects against app bugs.
- Cons: doesn't tell you about the bug, just hides it.

### Approach C: per-process fd limit
Set ulimit; OOM kills the leaky process when too many fds open.
- Anti-pattern as a fix, but useful as a circuit-breaker.

### Approach D: keepalive
With `tcp_keepalive_*` configured, kernel sends probes; eventually aborts.
- Pros: catches not-just-CLOSE_WAIT but all stale connections.
- Cons: indirect; default `tcp_keepalive_time` is 2hrs.

## 20.3 Pattern: ephemeral port exhaustion

A client doing many outgoing connections to the same (dst, port) maxes at ~28K (default range).

### Approach A: connection pool
The right answer 95% of the time.

### Approach B: widen `ip_local_port_range`
```
sysctl net.ipv4.ip_local_port_range="1024 65535"
```
Gives ~64K. Still limited.

### Approach C: multiple source IPs
With `IP_BIND_ADDRESS_NO_PORT`, kernel picks port on connect (deferred). Add multiple source IPs and round-robin.
- N source IPs → N × 64K capacity.

### Approach D: `tcp_tw_reuse=1`
Reuses TIME_WAIT ports for outgoing.

### Approach E: per-cgroup port ranges (5.16+)
`IP_LOCAL_PORT_RANGE` per-cgroup. Useful for multi-tenant.

## 20.4 Pattern: conntrack overflow

`nf_conntrack: table full, dropping packet`.

### Approach A: raise `nf_conntrack_max`
Each entry ~300 bytes; 4M entries = 1.2GB. Feasible on most servers.

### Approach B: shorten timeouts
`tcp_timeout_established` defaults 5 days (!). Set to 1 day.

### Approach C: NOTRACK for known-safe paths
```
iptables -t raw -A PREROUTING -p tcp --dport 80 -j NOTRACK
```
Bypasses conntrack for matching traffic.

### Approach D: switch to eBPF dataplane (Cilium)
Cilium maintains its own flow map in eBPF; doesn't use conntrack.

### Approach E: separate conntrack zones
`-j CT --zone N` puts traffic in a different conntrack instance. Allows multi-tenant isolation.

## 20.5 Pattern: SYN flood

Attacker sends millions of SYNs with spoofed sources.

### Approach A: SYN cookies (`tcp_syncookies=1`)
Kernel doesn't allocate SYN-queue entry; encodes state in seq #. Default on modern Linux.
- Pros: built-in.
- Cons: loses SACK + timestamps options (mostly recovered in 5.x via cookie extensions).

### Approach B: increase `tcp_max_syn_backlog`
Don't trigger syn_cookies until really under attack.

### Approach C: XDP drop
Pattern-match SYNs from clearly-bad sources; `XDP_DROP` at line rate.

### Approach D: upstream DDoS scrubbing
Cloudflare / AWS Shield / Akamai mitigate before traffic reaches you.

## 20.6 Pattern: single-flow elephant (one CPU bottleneck)

10Gbps NIC, one TCP flow doing 5Gbps, one CPU at 100%.

### Approach A: RPS to spread across CPUs
`echo ffff > /sys/class/net/eth0/queues/rx-0/rps_cpus` spreads to many.
- Pros: simple kernel sysfs change.
- Cons: adds latency; sometimes hurts cache locality.

### Approach B: GRO max-size + BIG_TCP
Bigger super-packets → fewer per-packet costs.

### Approach C: multi-stream application (the real fix)
Split into N parallel TCP connections. Each gets its own 4-tuple; RSS spreads.
- Pros: linear scale.
- Cons: app must support; some protocols don't.

### Approach D: AF_XDP / DPDK
Userspace stack with multi-core; bypasses kernel.

## 20.7 Pattern: head-of-line blocking in TCP

One lost packet stalls all data behind it in the stream.

### Approach A: just accept it (single TCP)
For long-lived bulk transfer (single file), this is intrinsic to TCP and fine.

### Approach B: HTTP/2 — multiplex streams, but per-connection HOL still
Many parallel streams in one TCP — but loss still blocks all streams. Net no.

### Approach C: HTTP/3 over QUIC — per-stream loss recovery
Stream-level HOL stays; cross-stream HOL gone. The right fix.

### Approach D: multiple TCP connections
Per-stream-per-conn. Pre-HTTP/2 pattern. ~10x worse for small assets.

## 20.8 Pattern: half-closed connections

App calls `shutdown(SHUT_WR)`; reads continue from peer.

### Approach A: handle it (correct)
Detect `EPOLLRDHUP` separately from `EPOLLHUP`; continue to read until 0.

### Approach B: full close on either side
App closes both directions on either FIN. Loses data still in flight.
- Anti-pattern unless app semantics demand.

### Approach C: shutdown timeout
`SO_LINGER` with timeout — if other side stalls, RST.

## 20.9 Pattern: NAT'd client behind us

NAT (CGNAT, corporate, mobile carrier) makes many clients look like one IP.

### Approach A: source-IP rate limit (broken)
Rate limit per src IP → punishes everyone behind the NAT.

### Approach B: use additional discriminators (HTTP cookies, headers, fingerprints)
Best-effort identification of unique clients.

### Approach C: source-port-aware
Each NAT'd client typically has unique src port. Add to hash for per-client behavior.

### Approach D: client-side identity (login)
The right answer for logged-in flows.

## 20.10 Pattern: load balancer behind anycast doesn't preserve flow on route change

Route changes mid-flow → packet to different POP → RST.

### Approach A: accept the trade
Anycast routes are mostly stable; rare RSTs.

### Approach B: stateful flow sync across POPs
Replicate conntrack between LBs; receiver of orphan packet looks up and forwards. High complexity.

### Approach C: GUE/IPIP wrapping with "this flow belongs to POP X" header
Stateless: orphan packet re-encapsulates and forwards. Used by some hyperscalers.

### Approach D: prefer L4-LB inside each POP with consistent hashing
Within a POP, Maglev ensures flow stickiness. Cross-POP migration is rare.

## 20.11 Pattern: slow start on every new connection

TCP starts at `initcwnd=10`; takes RTTs to ramp up.

### Approach A: keep-alive / connection pool
Don't open new connections.

### Approach B: TFO (TCP Fast Open)
0-RTT data for repeat clients.

### Approach C: increase initcwnd
Per-route: `ip route change ... initcwnd 30 initrwnd 30`.
- Pros: faster ramp.
- Cons: more burst at switch buffers; can hurt other flows.

### Approach D: QUIC 0-RTT
For resumed sessions, send data with first packet.

## 20.12 Pattern: receiver too slow → backpressure

App reads slower than sender → buffer fills → TCP zero-window → sender stalls.

### Approach A: drop messages (UDP) or backpressure (TCP)
For UDP, accept loss. For TCP, kernel handles via zero window.

### Approach B: bigger buffer
Increase `tcp_rmem.max` if the burst can fit.

### Approach C: faster app (the real fix)
Often there's a smell — single-threaded read; expensive parse; sync syscall blocking.

### Approach D: app-level flow control
Send rate matches consume rate via explicit credit (gRPC streaming, MQTT QoS).

## 20.13 Pattern: PMTU blackhole

Path drops ICMP "Frag Needed"; TCP keeps sending too-big packets that get dropped silently.

### Approach A: `tcp_mtu_probing=1`
Kernel detects no-progress + binary-searches MSS down.

### Approach B: clamp MSS at boundary
`iptables -t mangle -A FORWARD -p tcp --syn -j TCPMSS --clamp-mss-to-pmtu`.

### Approach C: avoid PMTUD entirely
Set DF=0 on TCP (rarely done; defeats point).

## 20.14 Pattern: bufferbloat

Big queue at intermediate router → fills with bulk traffic → latency for interactive surges.

### Approach A: `fq_codel` everywhere
Default in modern Linux; drops bulk early to keep small flows latency-low.

### Approach B: CAKE on edge router
Layer-aware shaping for home/SMB CPE.

### Approach C: BBR sender
BBR doesn't fill queues the way Cubic does (because it doesn't wait for loss).

### Approach D: smaller buffers
Right-size network buffers. Requires fabric change.

## 20.15 Pattern: TCP keepalive default is 2 hours

A dead peer with no traffic → stuck connection for 2hrs.

### Approach A: tune
```
sysctl net.ipv4.tcp_keepalive_time=600
sysctl net.ipv4.tcp_keepalive_intvl=30
sysctl net.ipv4.tcp_keepalive_probes=5
```

### Approach B: per-socket
`setsockopt(TCP_KEEPIDLE, ...)` per connection.

### Approach C: app-level heartbeat
Often more responsive (10-30s) and protocol-aware.

### Approach D: `TCP_USER_TIMEOUT`
Aborts a connection that hasn't ACKed for N seconds (regardless of keepalive).

## 20.16 Pattern: cross-region throughput pitiful

10Gbps link, 100ms RTT, throughput 50Mbps.

### Approach A: tune `tcp_rmem.max`
BDP = 125MB; default 6MB caps. Set max to 256MB+.

### Approach B: BBR
`tcp_congestion_control=bbr`; better at high-BDP.

### Approach C: multi-stream
N parallel connections per logical flow → aggregate the BDP.

### Approach D: QUIC
Per-stream loss recovery, faster ramp via 0-RTT.

## 20.17 Pattern: TLS handshake CPU

Many new connections → ECDHE handshakes pile up → CPU saturates.

### Approach A: session resumption
TLS 1.3 PSK / 1.2 session tickets. ~0.5ms vs full ~3ms.

### Approach B: TLS offload at LB
Terminate at L7 LB; backend gets plain HTTP. LB has dedicated TLS hardware.

### Approach C: kTLS + NIC offload
AEAD in NIC; ECDHE in userspace; overall ~5x cheaper.

### Approach D: connection pool (the cheaper answer)
Avoid handshake by reusing.

## 20.18 Pattern: thundering herd on shared listen socket

N workers do `accept` on shared fd → SYN arrives → all wake → one wins.

### Approach A: `SO_REUSEPORT` (one socket per worker)
Kernel hashes incoming to one socket. No herd.

### Approach B: `EPOLLEXCLUSIVE`
One epoll waiter at a time wakes.

### Approach C: lock per worker
Pre-`SO_REUSEPORT` pattern; works but complex.

## 20.19 Pattern: high pps drops at NIC ring

`ethtool -S` shows `rx_no_buffer_count` rising.

### Approach A: bigger RX ring
`ethtool -G eth0 rx 4096`.

### Approach B: more queues + RSS
`ethtool -L eth0 combined N`.

### Approach C: bigger NAPI budget
`sysctl net.core.netdev_budget=600`.

### Approach D: XDP drop early
Don't even create skb for unwanted traffic.

## 20.20 Pattern: encryption breaking sendfile

TLS in userspace → sendfile can't be used → 2x CPU.

### Approach A: kTLS
Push keys to kernel; sendfile works.

### Approach B: NIC TLS offload
Even cheaper; AEAD in HW.

### Approach C: userspace + io_uring + IORING_OP_SEND_ZC
Different zero-copy path; works without kTLS.

### Approach D: don't encrypt (when safe)
Internal traffic that's already in a TLS tunnel doesn't need re-encrypt.

## 20.21 Pattern: NIC drops on receive due to single-CPU softirq

Big flow → one CPU at 100% %si → drops.

### Approach A: RSS spread
Force multi-queue, set hash to include L4 ports.

### Approach B: RPS
Software re-steer.

### Approach C: XDP at NIC drop unwanted traffic
Don't even pay softirq cost for trash.

### Approach D: multi-stream the source
Many flows → RSS naturally spreads.

## 20.22 Pattern: containers losing pod IP after restart

CNI plugin leaks IP allocations.

### Approach A: pick a CNI with strong IPAM
Calico, Cilium have robust IPAM.

### Approach B: external IPAM
AWS VPC CNI delegates to AWS; no leak possible.

### Approach C: garbage collection script
Periodic sweep of allocations vs live pods. Hack but works.

## 20.23 Pattern: kube-proxy iptables reload during deploy

10K iptables rules; non-atomic flush+rebuild → seconds of mis-routing.

### Approach A: IPVS mode
`kube-proxy --proxy-mode=ipvs` — kernel-level LB, fast reload.

### Approach B: Cilium (replaces kube-proxy)
eBPF maps; per-rule update is O(1); no flush.

### Approach C: split kube-proxy across nodes
Don't make every change a full reload.

## 20.24 Pattern: BGP flap during incident

Route flapping causes oscillation; clients see RSTs.

### Approach A: BGP route flap damping (RFC 2439)
Don't accept updates from oscillating peers.

### Approach B: longer hold timer
Reduces flapping rate but slows convergence too.

### Approach C: BFD for fast detection
Detect once, declared down, withdraw.

### Approach D: graceful restart (RFC 4724)
BGP daemon can restart without flushing routes.

## 20.25 Pattern: DDoS at multiple Mpps

### Approach A: XDP drop at edge
Kill bad traffic before sk_buff allocation. Mpps capacity per CPU.

### Approach B: upstream scrubbing
Cloudflare, AWS Shield, Akamai. Cleansed traffic comes to you.

### Approach C: rate-limit per (source IP, signature)
eBPF maps + XDP drop on threshold.

### Approach D: anycast burn-off
Distribute to many POPs; each absorbs a fraction.

## 20.26 Pattern: latency tail caused by TCP slow start after idle

Long-lived keepalive connection idle 1s → kernel resets cwnd → next request slow.

### Approach A: `tcp_slow_start_after_idle=0`
Don't reset.

### Approach B: app keepalive at sub-RTT interval
Keep packets flowing.

### Approach C: pacing (BBR + fq)
BBR doesn't reset the same way.

## 20.27 Pattern: kernel UDP RX drops at high pps

Single thread reading `recvmsg` → can't keep up → drops.

### Approach A: `recvmmsg` batched reads
Drop one syscall per batch.

### Approach B: `SO_REUSEPORT` + N workers
Each gets its share.

### Approach C: UDP GRO
Coalesce on RX before delivering.

### Approach D: AF_XDP
Zero-copy ring; saturate cores.

## 20.28 Pattern: TCP_NODELAY vs Nagle vs delayed ACK

40ms stalls between small writes.

### Approach A: `TCP_NODELAY`
Disable Nagle. The right answer for RPC.

### Approach B: `TCP_CORK`
Hold until enough or uncork. Bulk-friendly.

### Approach C: app-level batching
Batch writes into one `send()`.

### Approach D: `TCP_QUICKACK`
Reduce delayed-ACK on receiver. Less effective alone.

## 20.29 Pattern: container DNS slow (ndots:5)

Pod resolves `external.example.com` → searches kube-system.svc.cluster.local first → NXDOMAINs → finally external.

### Approach A: lower ndots
Set `ndots:1` in `/etc/resolv.conf` (`spec.dnsConfig.options` in pod spec).

### Approach B: FQDN with trailing dot
`example.com.` skips search; direct lookup.

### Approach C: NodeLocalDNS
Per-node DNS cache. Reduces DNS hop and reuses connections.

## 20.30 Pattern: TLS termination at edge loses client cert

Mutual TLS at LB; backend doesn't see client cert.

### Approach A: pass-through cert in header
LB extracts cert; forwards via `X-Client-Cert` header.

### Approach B: re-establish mTLS LB→backend
Two mTLS hops. Doubles handshake cost.

### Approach C: TCP pass-through LB
L4 LB; backend terminates mTLS itself. Loses L7 LB features.

## 20.31 Selection rubrics (consolidated)

| Scenario | First choice | If problem persists |
|----------|--------------|---------------------|
| Many connections to one upstream | Connection pool | `tcp_tw_reuse=1` + widen port range |
| Conntrack overflow | Raise max + shorten timeouts | eBPF bypass (Cilium) |
| Single-flow bottleneck | RPS | Multi-stream the app |
| SYN flood | syncookies | XDP filter |
| Bufferbloat | fq_codel | CAKE or BBR |
| Slow cross-region | Tune buffers to BDP | BBR + multi-stream |
| TLS CPU | Session resumption | kTLS + NIC offload |
| DNS slow in containers | Lower ndots | NodeLocalDNS |
| kube-proxy reload slow | IPVS | Cilium |

## Must-internalize

- For every problem in production, you should be able to name **3** solutions.
- The cheapest fix is often application-level (pool, batch, cache).
- Kernel tuning fixes the symptom; app fixes the cause.
- eBPF/XDP is the modern "we can't fix this in the kernel, give us our own dataplane."
- Memorize the selection rubrics — they're staff-level table-stakes.