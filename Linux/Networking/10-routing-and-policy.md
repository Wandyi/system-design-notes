# 10 · Routing and Policy

The kernel's "where does this packet go" decision. iproute2's `ip route`, `ip rule`, FIB, BGP integration. Underlies multi-tenant networking, anycast, source-based routing, and every datacenter overlay.

## 10.1 The forwarding decision

For each packet (received-for-forward or generated-locally), the kernel must answer:
1. Which next-hop IP do I send this to?
2. Which interface do I send it out of?

This is **routing**: a lookup keyed by destination address.

```
packet dst=8.8.8.8 ──► route lookup ──► next-hop 192.0.2.1 via eth0
```

In Linux, the lookup uses the **FIB** (Forwarding Information Base): a trie organized by destination prefix.

## 10.2 ip route — the user surface

```bash
ip route                       # show main table
ip route show table 200        # show table 200
ip -4 route get 8.8.8.8        # ask: how would you route this?
ip route add 192.168.1.0/24 via 10.0.0.1 dev eth0
ip route add default via 10.0.0.1 metric 100
```

Routes have:
- Destination prefix (with /mask).
- Next-hop (`via`) or directly attached (`dev`).
- Output interface (`dev`).
- Metric (priority; lower wins).
- Type (`unicast`, `local`, `broadcast`, `multicast`, `blackhole`, `unreachable`, `prohibit`, `nat`).
- Scope (`global`, `link`, `host`).
- Protocol (`kernel`, `static`, `dhcp`, `bgp`, etc. — metadata).

## 10.3 Multiple routing tables — policy routing

Linux supports up to 252 routing tables. Built-in:
- `local` (table 255) — IPs assigned to the host.
- `main` (table 254) — the normal table you edit with `ip route`.
- `default` (table 253).

Custom tables 1–252 for policy routing.

### ip rule — match → choose table

The **RPDB** (Routing Policy DataBase) is a list of rules. Each rule matches the packet on (source, mark, iif, fwmark, tos, uidrange) and points to a table.

```bash
ip rule add from 10.1.0.0/16 table 100  # pkts from 10.1/16 → table 100
ip rule add fwmark 0x1 table 200        # marked by iptables → table 200
ip rule add iif eth1 table 300          # received on eth1 → table 300
```

Lookup order: scan rules by priority (lower wins), use the first matching table's lookup. If no match in a table, fall through to next.

Uses:
- **Source-based routing**: route differently per-source (multi-WAN: customer A's traffic via ISP-1, customer B via ISP-2).
- **Mark-based routing**: iptables MARK → routes via specific path.
- **VRF (Virtual Routing and Forwarding)**: complete routing-table isolation (next section).

## 10.4 VRF — Virtual Routing and Forwarding

A **VRF** is a netns-lite: separate routing table, but shared interfaces (with VRF assignment).

```bash
ip link add vrf-blue type vrf table 100
ip link set dev eth1 master vrf-blue   # eth1 now lives in VRF blue
ip route add 0.0.0.0/0 via 10.0.0.1 dev eth1 table 100
```

Sockets can be bound to a VRF via `SO_BINDTODEVICE` on the VRF interface.

Use case: multi-tenant routers where each tenant has its own routing table but same physical box. Linux + FRR can run a real BGP router with hundreds of VRFs.

Differences from netns:
- VRF shares ARP/conntrack/iptables with the host.
- VRF is L3 only; netns is L2+L3.

## 10.5 FIB structure

Pre-3.6: hash-based, per-CPU caches. Hard to scale to many routes.

Modern (3.6+): **LPM (Longest Prefix Match) trie** in `net/ipv4/fib_trie.c`.

- O(W) lookup where W = prefix width (32 for IPv4, 128 for IPv6).
- For real internet routing tables (~1M IPv4 routes), trie is ~30MB and lookup ~200ns per packet.

`ip route show` traverses; `ip route get` runs the lookup.

## 10.6 Multipath — ECMP

Equal-Cost Multi-Path: multiple next-hops for the same prefix; kernel picks one.

```bash
ip route add default \
  nexthop via 10.0.0.1 dev eth0 weight 1 \
  nexthop via 10.0.0.2 dev eth1 weight 1
```

Selection: per-packet (random) historically; per-flow (5-tuple hash) since 4.4 via `fib_multipath_hash_policy`.

Why per-flow matters:
- Per-packet ECMP reorders TCP segments → spurious retransmits.
- Per-flow → no reorder; less even distribution if few flows.

Hash policy:
- 0: L3 hash (src/dst IP).
- 1: L4 hash (5-tuple).
- 2: L3 hash + inner header (encap-aware).
- 3: GTP hash.

ECMP is the basis of leaf-spine datacenter routing.

## 10.7 Nexthop objects — modern routing

Pre-5.3: each route had its own copy of the nexthop. Programming N routes through the same gateway = N nexthop entries.

Post-5.3: `ip nexthop` is a first-class object. Routes reference nexthops by ID. Big perf win for BGP daemons with millions of routes.

```bash
ip nexthop add id 1 via 10.0.0.1 dev eth0
ip nexthop add id 2 via 10.0.0.2 dev eth1
ip nexthop add id 100 group 1,2     # ECMP group
ip route add 192.168.0.0/16 nhid 100
```

## 10.8 BGP integration (FRR, BIRD)

User-space daemons (FRR, BIRD, Quagga (legacy)) speak BGP/OSPF/etc. and push routes into the kernel FIB via netlink (`RTM_NEWROUTE`).

```
   BGP peer ──► BGPd (FRR) ──► zebra (FRR's RIB manager) ──► netlink ──► kernel FIB
```

Modern kernels: **FIB offload** to NIC hardware (Mellanox Spectrum). The kernel pushes the routes to the NIC's TCAM; hardware does the lookup.

`ip route show ... offload` indicates offloaded routes.

This is how a Linux box becomes a serious router.

## 10.9 Anycast

Anycast = same IP, multiple hosts, BGP routes the request to the nearest one.

Linux side:
- Each host announces the same prefix to BGP.
- BGP attribute (MED, AS_PATH length, LocalPref) determines which is picked.
- The host has the anycast IP assigned, usually on a loopback dummy interface.

```bash
ip link add dummy-any type dummy
ip addr add 1.1.1.1/32 dev dummy-any
ip link set dummy-any up
```

Withdraw the route to gracefully fail off the host.

Used by: Cloudflare DNS (1.1.1.1), Google DNS (8.8.8.8), root DNS servers, every CDN edge.

Failure modes:
- **TCP flows that get rebalanced** — different host doesn't share state; RST. Mitigation: shorter routing convergence; **flow consistency via L4 LB tier**; or graceful drain (withdraw route slowly).
- **BFD** for sub-second detection of dead peer.

## 10.10 Source-based routing

`ip rule add from 10.1.0.0/16 table 100` routes traffic from that subnet via a specific path. Common in:
- Multi-WAN hosts (one IP per ISP).
- Forwarding-only routers with policy.
- Container networking with multiple overlays.

Combined with `ip route ... src 10.1.0.5` you can force outbound source IP.

## 10.11 Common interview probes

- **"How is anycast different from unicast LB?"** Anycast routes at the BGP layer; unicast LB routes at TCP/HTTP. Anycast handles geographic distribution natively; can rebalance mid-flow (bad). Unicast LB has consistent flow assignment but adds a hop.
- **"What if a route fails (link down)?"** Without BFD: TCP retransmit storm until BGP detects (default ~180s hold time). With BFD: ~150ms detection. ECMP routes drop dead path immediately if `ip link` state changes.
- **"How does the kernel pick the source IP when sending?"** From `ip route get` — kernel picks based on dst, route, and source-IP-of-the-interface (`src` attribute). User can override with `bind()`.
- **"Show me a multi-tenant routing config."** Per-tenant VRF (or netns), per-VRF table, BGP per-VRF, source-based rule to map host → VRF.
- **"How does Cilium ROUTE traffic?"** eBPF maps key=dst IP, value=ifindex+nexthop. Bypasses FIB. Cilium specifically replaces the routing table for cluster traffic.

## 10.12 Corner cases

- **`ip route get` differs from actual forwarding.** `ip route get` uses default rule lookup; FIB cache (for some kernels) may differ. Test with actual traffic.
- **Default route on multiple interfaces.** Both have metric 100 → kernel picks based on something arbitrary. Set explicit metric.
- **Route cache flushed unexpectedly.** Some kernel changes invalidate `rt_cache`; pre-3.6 this was visible. Mostly mitigated.
- **VRF and conntrack.** Same conntrack table per netns, but VRF doesn't shard. Multi-tenant VRFs share conntrack — may need `-j CT --zone` for isolation.
- **GRE / VXLAN routes mix.** Encapsulation interface has its own route to find the real next-hop; cascading lookups.
- **IPv6 unique-local vs link-local.** Link-local needs scope; many routing daemons require explicit `zone` config.
- **Stale neighbor entries.** `ip neigh show` may have FAILED entries; clear with `ip neigh flush` or wait for GC.

## 10.13 Routing daemons cheat sheet

| Daemon | Protocols | Used by |
|--------|----------|---------|
| **FRR** | BGP, OSPF, IS-IS, RIP, EIGRP, PIM, LDP | Cumulus, Calico, kube-router, many DC |
| **BIRD** | BGP, OSPF, RIP, RAdv | Anycast/CDN ops, NTP pool |
| **Quagga** | BGP, OSPF, RIP | Legacy; FRR forked from this |
| **gobgp** | BGP | Cloud-native, programmable |
| **GoBGP / GoFRR** | Niche control planes | Cloud routing services |

## 10.14 Multipath TCP — bonus

Mentioned in `03`; relevant to routing because MPTCP creates multiple subflows often via different routes. Routing rules can steer subflows; `ip mptcp` configures endpoints.

## 10.15 Alternative approaches

| Need | Path 1 | Path 2 | Path 3 |
|------|--------|--------|--------|
| Multi-tenant isolation | netns | VRF | iptables-mark + routing tables |
| Multi-WAN failover | active/passive routes | mwan3 (OpenWrt) | BGP/BFD per WAN |
| HA service IP | keepalived + VRRP | anycast + BGP | floating IP via cloud APIs |
| ECMP load balance | per-flow hash | per-packet (rare) | weighted ECMP |
| Container traffic | bridge routing | per-pod route via veth | eBPF maps (Cilium) |
| BGP routes | FRR | BIRD | gobgp |

## Must-internalize

- Routing answers "next-hop IP + output interface" via FIB (LPM trie).
- 252 + 3 built-in tables; `ip rule` chooses by source/mark/iif.
- VRF = L3 isolation (multi-tenant routing); netns = L2+L3 isolation.
- ECMP: multiple nexthops, **per-flow** hash (not per-packet) to avoid TCP reordering.
- `ip nexthop` (5.3+) makes BGP routes much cheaper to install.
- Anycast = same IP, multiple hosts, BGP routes to nearest. TCP flows are NOT migration-safe.
- FRR / BIRD push routes via netlink; modern NICs can offload FIB to TCAM.
- ip rule + iptables MARK = mark-based routing (very common pattern).