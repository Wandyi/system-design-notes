# 02 · Linux Network Stack Architecture

This is the foundation file. Everything else assumes you can trace a packet end-to-end. Read this twice.

## 2.1 The mental model — packet path RX

```
NIC ── DMA ──► RX ring ──IRQ──► CPU/IRQ handler
                                    │ schedules NET_RX_SOFTIRQ
                                    ▼
                              napi_poll() loop
                              (in softirqd context, budget=netdev_budget=300)
                                    │
                                    ├── XDP hook  ── DROP / TX / REDIRECT
                                    │  (no sk_buff yet, raw page+offset)
                                    ▼
                              build sk_buff
                                    │
                              GRO coalescing
                              (merges segments to one big skb)
                                    │
                              RPS / RFS steering
                              (re-queue on a different CPU's backlog)
                                    │
                              __netif_receive_skb_core()
                                    │
                              tc ingress hook (eBPF, qdisc-ingress)
                                    │
                              netfilter PREROUTING hook
                                    │
                              ip_rcv() → ip_rcv_finish()
                                    │
                              FIB lookup: local or forward?
                                    │
                              netfilter INPUT hook (local) or FORWARD (route)
                                    │
                              ip_local_deliver() → tcp_v4_rcv() / udp_rcv()
                                    │
                              TCP demux by 4-tuple → sock lookup
                                    │
                              data → sk_receive_queue
                                    │
                              sk_data_ready() wakes userspace
                                    │
                              app reads via recvmsg()
```

## 2.2 The mental model — packet path TX

```
app: send()/write()/sendmsg() ──► tcp_sendmsg()
                                       │
                                  copy from userspace
                                  (skb_zerocopy or copy_from_iter)
                                       │
                                  TCP build segment: tcp_write_xmit()
                                       │
                                  IP build header: ip_queue_xmit()
                                       │
                                  netfilter OUTPUT hook
                                       │
                                  ip_output() → ip_finish_output()
                                       │
                                  netfilter POSTROUTING hook
                                       │
                                  fragment if needed (rare; PMTUD usually avoids)
                                       │
                                  dev_queue_xmit()
                                       │
                                  qdisc enqueue (default: fq_codel)
                                       │
                                  qdisc dequeue → __qdisc_run()
                                       │
                                  tc egress hook (eBPF)
                                       │
                                  dev_hard_start_xmit() → driver
                                       │
                                  ndo_start_xmit()
                                       │
                                  TSO if enabled; DMA-map skb
                                       │
                                  NIC fetches DMA descriptors, sends
                                       │
                                  TX completion IRQ → free skb
```

## 2.3 The two key data structures

### `sk_buff` ("skb")

The kernel's universal packet container. Defined in `include/linux/skbuff.h`. One skb = one packet (or one segmented frame post-GSO).

```
+---------------+  ◄── struct sk_buff (metadata, ~232 bytes)
|  head/data ptrs    |
|  protocol/family   |
|  device            |
|  napi_id, mark, ...|
+---------------+
       │
       ▼ points to data area:
+--------+--------+-----------+--------+
| hdroom | data   | tailroom  | sk_buff_shared_info |
+--------+--------+-----------+--------+
         ▲        ▲
         |        |
       data    tail        ◄── pointers that push as protocols add headers
```

Key fields:
- `head`, `data`, `tail`, `end` — pointers into the linear data region.
- `len` — current data length; `truesize` — actual memory accounting.
- `sk` — owning socket (NULL on RX before sock-lookup).
- `dev` — net_device that received or will transmit.
- `protocol` — Ethernet type (htons(ETH_P_IP), etc.).
- `mark`, `priority`, `tc_index` — used by tc/iptables for classification.
- `cb[48]` — control block, per-protocol scratch.
- `shinfo` (at tail) — fragments list for SG/GSO/GRO.

Why it matters: **every per-packet cost in the kernel is an skb cost.** Allocation, copy, free. XDP exists because it operates *before* skb allocation.

### `net_device` ("netdev")

The kernel's "this NIC / virtual interface" abstraction. Defined in `include/linux/netdevice.h`.

Key fields:
- `name` — `eth0`, `wlan0`, etc.
- `dev_addr` — hardware address (MAC).
- `mtu` — max transmission unit.
- `flags` — `IFF_UP`, `IFF_PROMISC`, `IFF_BROADCAST`, `IFF_LOOPBACK`.
- `netdev_ops` — function pointer table (`ndo_open`, `ndo_start_xmit`, `ndo_set_mac_address`, ...).
- `ethtool_ops` — for the ethtool surface.
- `_rx`, `_tx` arrays — RX/TX queues.

Userspace surface: `ip link`, `ifconfig`, `ethtool`, the `rtnetlink` socket family.

## 2.4 NAPI — the bridge between IRQ and softirq

Pre-NAPI: every received packet triggered an IRQ → driver pulled it → handed to stack. At 10Gbps this melted CPUs (interrupt storm).

NAPI ("New API," circa 2003) introduced **interrupt mitigation + polling**:

1. First packet arrives → IRQ fires.
2. IRQ handler disables further IRQs on this queue, schedules `NET_RX_SOFTIRQ`.
3. Soft IRQ runs `napi->poll()` which drains the RX ring up to budget (`netdev_budget`, default 300; per-poll quota `netdev_budget_usecs`).
4. If the budget is exhausted but more packets exist → re-schedule softirq.
5. If the ring is empty → re-enable IRQs.

This means: **per IRQ you process many packets**. CPU cost is amortized; bursts batch together.

Tuning knobs:
- `net.core.netdev_budget` — packets per softirq poll. Up to 600 for high-throughput, down for low-latency.
- `net.core.netdev_budget_usecs` — time budget; whichever is reached first stops.
- `net.core.dev_weight` — per-poll weight (legacy SO_RCVBUF flushing).
- Per-driver coalescing: `ethtool -C eth0 rx-usecs N rx-frames M`.

Failure mode: **softirq starvation** under sustained burst. `top` shows `%si` (softirq) ≈ 100% on one CPU, queue stall. Workaround: enable RPS to spread to many CPUs, or use `XDP` to drop unwanted packets before NAPI handoff.

## 2.5 GRO / GSO — segmentation offload

**GRO (Generic Receive Offload)** coalesces consecutive packets into one larger skb at NAPI time. A TCP stream's 64 1500-byte segments become one 96000-byte skb (with `shinfo->frags`), saving 63 protocol-stack walks.

```
RX ring: [pkt1 pkt2 pkt3 ... pkt64] each 1500B
              │
              ▼ GRO
       one skb of 96KB, frags-linked
```

Conditions for coalescing: same 5-tuple, in-order seq, no exotic flags, MTU bounded by `gro_max_size` (default 65536; can go to 192KB with BIG_TCP).

**GSO (Generic Segmentation Offload)** is the TX mirror: TCP hands a big skb (up to 64KB / `gso_max_size`) to the driver; driver+NIC segments to MTU-sized frames.

**TSO/UFO/LRO** are hardware-assisted versions of GSO/GRO. `ethtool -k eth0` lists what's enabled.

Why staff cares:
- GRO is **the** reason TCP works at 100Gbps with sane CPU. Disable it and CPU usage triples.
- GRO can hide latency — a 64-segment coalesce holds the last segment for up to `gro_flush_timeout` (default 0; relevant for some low-jitter workloads).
- GRO interacts with XDP: XDP runs *before* GRO. If your XDP redirects, you lose coalescing — usually fine since the redirect target may or may not GRO depending on its hook.
- BIG_TCP (5.19+) raised the GRO+GSO max from 64KB to 4GB for IPv6 — Cloudflare reports 30% throughput improvement on long-RTT links.

## 2.6 The protocol handler dispatch

After `__netif_receive_skb()`, the kernel walks `ptype_base` (a hash by `skb->protocol`):

```
ETH_P_IP   → ip_rcv()
ETH_P_IPV6 → ipv6_rcv()
ETH_P_ARP  → arp_rcv()
ETH_P_8021Q → vlan_skb_recv()
```

`ip_rcv()` validates the IP header, calls netfilter `NF_INET_PRE_ROUTING`, then routes:

- If destined locally → `ip_local_deliver()` → `NF_INET_LOCAL_IN` hook → upper layer (TCP/UDP/ICMP/raw).
- If forwarded → `ip_forward()` → `NF_INET_FORWARD` → `ip_output()`.

Upper-layer demux:
- TCP: `tcp_v4_rcv()` does the 4-tuple lookup in `tcp_hashinfo` (the `LISTEN` hash + `ESTABLISHED` hash). If found and ESTABLISHED, into `tcp_rcv_established()`.
- UDP: `udp_rcv()` → `udp4_lib_lookup_skb()` → `udp_queue_rcv_skb()` → enqueue on socket.

## 2.7 Layers and where they live

```
Layer                Where in kernel              User surface
─────────────────────────────────────────────────────────────
L1 (PHY)             driver / firmware            ethtool -t
L2 (Ethernet/MAC)    net/ethernet/, ARP           ip link, arp
L2.5 (VLAN, MPLS)    net/8021q/, net/mpls/        ip link add ... type vlan
L3 (IP, ICMP)        net/ipv4/, net/ipv6/         ip addr, ip route
L4 (TCP, UDP, SCTP)  net/ipv4/tcp_*, net/sctp/    ss, sockets
L5–L7 (TLS, app)     userspace; some kTLS         openssl, app
```

`ip(7)` and `socket(7)` are the manual pages every staff has read.

## 2.8 Kernel-space vs userspace boundary

Three trips across the boundary per packet on the slow path:
1. Userspace `recvmsg()` → kernel; data copied out.
2. (For TX) Userspace `send()` → kernel; data copied in.
3. NIC DMA bypasses userspace.

Each copy is ~3GB/s/CPU memory bandwidth at most. At 100Gbps that's 12.5GB/s of data — multiple copies starve a CPU.

Mitigations (each is a separate file):
- `sendfile()` — kernel pipes a file fd directly to a socket. No userspace.
- `splice()` — generalized pipe between fds.
- `MSG_ZEROCOPY` — kernel "pins" userspace pages, DMAs them directly.
- `io_uring` — async submit/complete, no syscall per op.
- `AF_XDP` — userspace gets a ring directly attached to NIC queue.
- DPDK — userspace owns the NIC, kernel doesn't see traffic.

## 2.9 Per-CPU and locking discipline

Network code is heavily per-CPU to avoid cross-CPU cache traffic:
- RX ring → softirq on the CPU that took the IRQ.
- TCP `ESTABLISHED` socket → no global lock; per-socket lock (`bh_lock_sock`).
- `LISTEN` socket → wider lock; sharded via `SO_REUSEPORT`.
- Routing cache → per-CPU `rt_genid`.
- Netfilter rules → per-CPU table snapshots in nftables; iptables uses RCU.

Common scaling issue: a single `listen` socket on many CPUs → accept queue contention → use `SO_REUSEPORT` with one socket per worker process and let the kernel hash incoming SYNs to a worker.

## 2.10 The flow-control flow

Where does TCP get its window from?

- Receive side: socket `sk_rcvbuf` allocates buffer; auto-tuned via `tcp_rmem`. Window advertised = remaining buffer / `tcp_adv_win_scale`.
- Send side: `sk_sndbuf` bounds outstanding bytes; auto-tuned via `tcp_wmem`. Congestion window (`cwnd`) is *separate* — sender will not exceed `min(cwnd, snd_wnd)`.
- Backpressure: `sk->sk_write_space()` wakes app when `sk_sndbuf` drains.

Sysctls (memorize):
```
net.ipv4.tcp_rmem = 4096 131072 6291456
net.ipv4.tcp_wmem = 4096 16384  4194304
   (min default max — autotune ranges)
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
```

For a 100ms × 10Gbps path: BDP = 125MB. If `tcp_rmem.max` < BDP, TCP can't fill the pipe regardless of congestion control. This is the **most common reason "fast" links seem slow**.

## 2.11 Corner cases worth knowing

- **Out-of-order arrival.** Multipath / load-balancing can deliver out-of-order; TCP NACK / SACK; UDP just delivers in arrival order; QUIC reorders at the application layer.
- **CPU steering bug.** RPS hashes by 4-tuple; if the hash bias hits one CPU, you see hot-CPU softirq starvation. Workaround: `set RPS_FLOW_CNT` and use RFS (which steers based on the CPU the app runs on).
- **Conntrack and the slow path.** If conntrack is loaded, every packet goes through it on PREROUTING — even on FORWARD. Pure-router workloads disable conntrack for performance.
- **VLAN double-tagging (QinQ).** Kernel handles two tags via `ip link add type vlan ... protocol 802.1ad`. ethtool can show offload.
- **MTU mismatch.** Path MTU discovery (PMTUD) breaks if ICMP "Frag Needed" is filtered. Workaround: PMTUD black-hole detection (`tcp_mtu_probing`).
- **GRO over tunneled traffic.** GRO works for VXLAN, GENEVE, GRE when the NIC has tunnel offload; otherwise tunnels reassemble in software (slower).

## 2.12 Alternative implementations (the "name 3 ways" reflex)

| Need | Path 1 (default) | Path 2 (faster) | Path 3 (specialist) |
|------|-----------------|----------------|---------------------|
| Receive packets | sockets + recvmsg | recvmmsg + busy poll | AF_XDP |
| Send a file | read() + send() | sendfile() | splice() / kTLS sendfile |
| Filter packets | iptables | nftables | XDP eBPF |
| Hash on 4-tuple for accept | one listen socket | SO_REUSEPORT | XDP_REDIRECT to per-CPU socket |
| Track connection state | conntrack | eBPF map (custom) | hardware flow tables |

## Must-internalize

- RX: NIC → IRQ → softirq → NAPI poll → (XDP) → skb → GRO → IP → L4 → socket.
- TX: socket → TCP → IP → netfilter → qdisc → driver → NIC.
- The skb is the unit of cost; XDP exists to skip it.
- NAPI = polled batched IRQ-mitigated draining; budget is the throughput-vs-latency knob.
- GRO + TSO/GSO are why 10Gbps works on a single CPU; without them the stack is 3–5× slower.
- For 100Gbps + 100ms RTT, set `tcp_rmem.max` to ≥125MB or you cap.