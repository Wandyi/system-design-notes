# 04 · UDP, QUIC, and DNS

UDP looks "easy" — and the kernel UDP code is short — but the production story is rich: GRO/GSO for UDP arrived in 4.18, QUIC moved everything to userspace, DNS is the most stressed UDP workload on the internet.

## 4.1 UDP — the protocol

RFC 768. 8-byte header: src port, dst port, length, checksum. No state, no retransmission, no flow control. The kernel does the bare minimum: route, demux by 4-tuple to socket, drop into `sk_receive_queue`.

```
                        ┌────────────────┐
recvmsg() ◄──────────── │ sk_receive_queue│ ◄── udp_queue_rcv_skb
                        └────────────────┘
```

When a UDP packet arrives:
1. `udp_rcv()` validates checksum.
2. `udp4_lib_lookup_skb()` finds the matching socket (port + optionally IP).
3. `udp_queue_rcv_skb()` charges the packet to the socket's receive buffer (`sk_rcvbuf`).
4. If buffer is full → drop. Increment `UDPInErrors` counter.

Receive buffer is critical: UDP has no backpressure. If the app is slow, the kernel drops. Defaults are too small for high-rate UDP (10–100Mpps DNS); tune `net.core.rmem_max` and `SO_RCVBUF`.

## 4.2 UDP send path

`sendmsg()` → `udp_sendmsg()` → build UDP header → `ip_send_skb()` → IP → routing → qdisc → driver.

If the message is bigger than MTU, kernel fragments (or drops with EMSGSIZE if `DF` bit set). Default `IP_PMTUDISC_WANT`: probe PMTU; fragmentation only if peer signals smaller.

UDP fragments are expensive: reassembly buffer per-flow on receiver; one lost fragment loses the whole datagram. Pro tip: keep UDP datagrams ≤ 1472 bytes (1500 MTU − 20 IP − 8 UDP).

## 4.3 UDP GRO/GSO — the 4.18 kernel revolution

Pre-4.18: each UDP packet caused a separate IP/UDP/skb walk. At 10Mpps this saturated a CPU.

Post-4.18: UDP GSO/GRO. Linux can:
- TX side (GSO): app sends a 64KB UDP "super packet"; kernel/NIC segments into MTU-sized datagrams. `setsockopt(UDP_SEGMENT, msslen)`.
- RX side (GRO): NIC or kernel coalesces consecutive same-flow UDP packets into one skb. `setsockopt(UDP_GRO)`.

This is **how QUIC got fast on Linux**. Without UDP GSO/GRO, QUIC syscall overhead made HTTP/3 slower than HTTP/2.

Cloudflare quic-go benchmark (2020): UDP GSO/GRO moved their per-CPU QUIC throughput from 0.8Gbps to 4.5Gbps.

## 4.4 recvmmsg / sendmmsg — batched UDP

`recvmmsg()` / `sendmmsg()` (man 2) let you do many recv/send in one syscall. Critical for high-rate UDP workloads (DNS, NTP, statsd).

```c
struct mmsghdr msgs[32];
int n = recvmmsg(fd, msgs, 32, 0, NULL);  // up to 32 datagrams in one call
```

Reduces syscall overhead from N to 1. With UDP GRO, even fewer syscalls per packet.

## 4.5 UDP corner cases

| Symptom | Cause | Fix |
|---------|-------|-----|
| `UDPInErrors` growing | Receive buffer full | Tune `rmem_max`, `SO_RCVBUF`, faster app |
| Per-CPU softirq pegged | Single 4-tuple hash to one CPU | RPS/RFS, multi-port load balancing |
| Reassembly OOM | Many partial fragments | `ipfrag_high_thresh`, drop fragments at edge |
| DNS resolver hangs | Source-port reuse vs caching | Random ephemeral, `SO_REUSEPORT` |
| Packet reordering breaks app | UDP is best-effort | App-layer sequencing |

## 4.6 QUIC — what changed and why

RFC 9000 (2021). UDP-based transport, encrypted, with: connection IDs, multiple streams (no HOL inside one connection), 0-RTT resumption, integrated TLS 1.3.

Key design decisions:
- **Userspace transport.** Kernel doesn't implement QUIC (no `socket(AF_INET, SOCK_QUIC, ...)`). It would be hard to update; userspace iteration is faster.
- **Encrypted headers.** Even connection setup is encrypted; middleboxes can't see flags.
- **Connection IDs.** Survive IP changes (mobile network roam); the 4-tuple isn't the identity.
- **Streams.** Multiple independent byte streams per connection. Loss on one doesn't block others.
- **Loss recovery.** RACK-style, with stream-level retransmission.

Production usage: Google (90%+ traffic), Facebook (mvfst), Cloudflare (quiche), Apple, Microsoft. Mobile and high-loss paths win biggest.

Kernel involvement (despite userspace transport):
- UDP GSO/GRO — the only reason QUIC is competitive in CPU.
- `SO_TXTIME` — pacing in kernel (avoids CPU spikes when scheduler sleeps userspace).
- kTLS doesn't apply (QUIC's TLS is per-packet, not stream).

Pending: **in-kernel QUIC** experimental patches exist (lwn.net/Articles/958571/). May land for some perf-critical use cases (e.g., file servers analogous to kTLS sendfile).

## 4.7 QUIC vs HTTP/2 vs HTTP/3 — the decision

| Property | HTTP/2 over TCP+TLS | HTTP/3 over QUIC |
|----------|---------------------|------------------|
| HOL within connection | Yes (TCP) | No (QUIC streams) |
| HOL within stream | n/a | Yes (in-stream ordering preserved) |
| Connection setup (cold) | 2 RTT (TCP+TLS) | 1 RTT |
| Connection setup (warm) | 1 RTT (TFO+TLS1.3) | 0 RTT |
| IP migration | Reconnect | Native |
| Server push | Yes (deprecated) | Yes |
| Middlebox compat | High | Lower (UDP often rate-limited) |
| Kernel TCP socket fast paths | Yes (kTLS, sendfile) | No (yet) |

When to deploy HTTP/3 — high latency / packet loss paths, mobile, IP-mobility critical.
When to skip — strict UDP filtering enterprise networks; well-tuned datacenter where TCP is already RTT-bound.

## 4.8 DNS — the most-tortured UDP workload

DNS on UDP/53 is the single largest packet-per-second UDP workload on the public internet. Cloudflare's 1.1.1.1 receives ~10M qps at peak.

Why DNS is hard:
- Tiny packets (typically 50–500 bytes). Per-packet CPU dominates.
- Mostly **request-response, low fanout per query, high cardinality**. Caching helps but only so much.
- Amplification attacks: 1-byte query → 4KB response with DNSSEC. Reflector for DDoS.
- DNS over TCP for >512-byte responses; DoT (DNS over TLS, RFC 7858); DoH (DNS over HTTPS, RFC 8484); DoQ (DNS over QUIC, RFC 9250).

### DNS server perf tricks

| Tactic | Effect |
|--------|--------|
| `SO_REUSEPORT` workers | Linear scale to N CPUs |
| `recvmmsg`/`sendmmsg` | 5–10× syscall reduction |
| UDP GRO/GSO | 2–4× CPU reduction |
| Pin process to CPU + RSS to same | Cache locality |
| XDP DDoS filter | Drop amplification at NIC, never reach userspace |
| Anycast | Distribute load and shorten RTT |
| Cache compression + sharing memory | Resolver scaling |

Cloudflare blog (Marek Majkowski) covers this in detail; expect a probe.

## 4.9 DNSSEC, DoT, DoH, DoQ

| Protocol | Transport | Use case | Cost |
|----------|-----------|----------|------|
| DNS classic | UDP/53 | Default | Cheap but spoofable |
| DNSSEC | UDP/53 | Authenticated answers | Bigger responses → fragmentation risk |
| DoT | TCP/853, TLS | Privacy, integrity | TCP overhead |
| DoH | HTTPS/443 | Privacy, censorship-resistance | HTTPS framing cost |
| DoQ | QUIC/853 | Privacy + 0-RTT | Newer; deploy maturity |

Apple iCloud Private Relay uses DoH + Oblivious DNS for privacy.

## 4.10 UDP — non-DNS heavy hitters

- **NTP** (time sync, UDP/123). Latency-critical: 1ms matters.
- **statsd / OpenTelemetry over UDP**. Lossy by design.
- **VoIP (RTP)**: UDP carries audio; jitter buffers absorb late arrivals.
- **WireGuard**: VPN over UDP, in-kernel.
- **VXLAN / GENEVE**: overlays over UDP; tunnel offload critical.
- **gRPC bidirectional streaming**: usually HTTP/2, but HTTP/3 grpc is rising.
- **Game traffic**: low-RTT UDP with custom reliability (Steam, EA, Riot).

## 4.11 Reliability over UDP — the design pattern

If you build on UDP, you re-invent these:
- Sequence numbers (detect loss + reorder).
- ACK / NACK scheme (cumulative, selective, or hybrid).
- Retransmit timer + RTO estimator.
- Window / pacing (don't melt the network).
- Fragmentation / reassembly above MTU.
- Encryption (DTLS, QUIC's own, or app-layer).

Three escape hatches:
1. **Just use QUIC.** RFC 9000-compliant libraries (quiche, mvfst, msquic, ngtcp2, lsquic). You get all of the above.
2. **Use SCTP.** Multi-stream, ordered or unordered, in mainline kernel. Underused; middlebox compat poor.
3. **Use TCP and live with HOL.** If a single-stream connection is your model, TCP is mature and free.

## 4.12 Corner cases (extra)

- **UDP fragmentation cliff.** Sending a 1473-byte UDP datagram causes IP fragmentation; one lost fragment → whole packet lost. Apps usually cap at 1200 to survive PPPoE/IPv6 headers.
- **UDP source-port rotation.** Linux defaults to a random ephemeral src port. DNS resolvers historically had predictable ports (Kaminsky cache poisoning); now ports are randomized + 0x20 query case mixing.
- **UDP socket sharing.** Multiple processes on `SO_REUSEPORT` UDP socket — kernel hashes by 4-tuple. But replies from a worker may use a different src port → response not associated. Use `IP_FREEBIND` / one-socket-per-flow.
- **`connect()` on UDP.** Sets the default destination; subsequent `send()` works without `sendto()`. Also enables ICMP error delivery to the socket.
- **UDP checksum offload.** NIC computes checksum; if disabled, kernel does it (CPU cost). `ethtool -K eth0 tx-checksum-ipv4 on`.
- **GSO+UDP and recvmsg with `MSG_PEEK`.** Subtle interaction; peek may see partial coalesced data.

## 4.13 Alternative approaches

| Need | UDP path | Alternative 1 | Alternative 2 |
|------|----------|---------------|---------------|
| Reliable in-order | App + UDP | TCP | QUIC |
| Low-latency lossy | UDP + jitter buffer | RTP | QUIC-based real-time |
| Multipath | App-layer | MPTCP (TCP) | QUIC migration |
| Encrypted | DTLS | TLS+TCP | QUIC (integrated) |
| Multi-stream | n/a | SCTP | QUIC streams |
| Datagram with auth | None native | DTLS | QUIC datagram extension |

## 4.14 The future — `AF_QUIC`?

The kernel community has explored in-kernel QUIC (LWN, 2023). Motivation: kTLS-style offload for QUIC, especially for file serving (à la Netflix's 800Gbps kTLS sendfile). Not merged as of 2026; expect this to be a future-of-Linux interview probe.

## Must-internalize

- UDP receive drops happen at `sk_rcvbuf`; bump it for high-rate workloads.
- UDP GSO/GRO arrived in 4.18 and is why QUIC is viable on Linux.
- `recvmmsg`/`sendmmsg` reduce syscall count linearly; mandatory for >1Mpps UDP.
- QUIC moves reliability + encryption to userspace; HTTP/3 = HTTP over QUIC.
- DNS at scale = `SO_REUSEPORT` + recvmmsg + UDP GRO + XDP DDoS filter + anycast.
- UDP datagrams > MTU fragment; one fragment lost = whole packet lost.
- `connect()` on UDP enables both default dst and ICMP error delivery.