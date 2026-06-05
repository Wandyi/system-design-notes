# 13 · Bonding, MPTCP, and Failover

How Linux makes a connection survive a link death, a NIC failure, an ISP outage. Three orthogonal layers: link-level bonding, transport-level MPTCP, routing-level BGP/anycast.

## 13.1 Link-layer bonding (`bonding` driver)

Two+ NICs combined into one logical interface (`bond0`). Modes:

| Mode | Number | What |
|------|--------|------|
| `balance-rr` | 0 | Round-robin packets across slaves. Reorder risk for TCP. |
| `active-backup` | 1 | One slave active; failover on link down. Most common. |
| `balance-xor` | 2 | Hash-based per flow (5-tuple or MAC). No reorder. |
| `broadcast` | 3 | All slaves transmit. Redundant fan-out. |
| `802.3ad` (LACP) | 4 | IEEE link aggregation; switch must support. Standard for DC. |
| `balance-tlb` | 5 | Adaptive transmit load balance. |
| `balance-alb` | 6 | tlb + receive (via ARP tricks). |

LACP (mode 4) is the standard for ToR↔server in datacenters. Both sides advertise LACPDU; load balances by configurable hash.

Failure detection options:
- **MII** (Media Independent Interface link monitor): polls each slave's `IFF_UP` state. Fast for physical removal.
- **ARP monitor**: sends ARPs to a target; uses replies as health probe. Catches upstream-network failures, not just direct link.

Bonding gotcha: hashing must be 4-tuple aware for fair load distribution; use `xmit_hash_policy=layer3+4`. Older default `layer2` (MAC-only) puts all traffic on one slave if dest MAC is the same router.

## 13.2 Team driver — modern alternative

`teamd` is a userspace alternative to kernel bonding. Same modes (active-backup, LACP, etc.) but configured via JSON, with hot-plug support and modularity.

Mostly a Red Hat thing; bonding remains dominant. Either is fine for staff interview — know the modes.

## 13.3 VRRP / keepalived — IP-level HA

VRRP (RFC 5798): a "virtual" IP shared by multiple hosts; only the master holds it; failover via VRRP advertisements.

`keepalived` is the userspace daemon: monitors services, advertises VRRP, scripts the floating IP.

Use case: two LB nodes sharing a frontend IP. Master dies → backup takes over within ~3 advertisements (default 1s each).

Limits:
- Active/passive only (one master). Not for scale-out.
- Hairpin: master receives all; doesn't scale beyond one host's capacity.
- Mitigation: VRRP + GARP for IP move; one VIP per service grouping.

## 13.4 Anycast (BGP-level) — the modern scale-out HA

Anycast = same IP, many hosts; BGP routes to nearest. Each host announces the prefix; withdrawal = traffic moves.

Properties:
- Scale-out: each POP sees a slice of traffic.
- Latency-aware: BGP picks topologically nearest.
- HA via withdrawal: if process dies, withdraw the route; BGP convergence.

Convergence speed:
- BGP default hold-time = 180s (too slow for ops).
- With **BFD** (Bidirectional Forwarding Detection): sub-second.
- With graceful restart (RFC 4724): no traffic disruption during BGP daemon restart.

Failure modes (and the staff answer):
- **TCP flows lose state** when re-routed to a different POP. Mitigation: prefer L4 LB tier behind anycast that keeps flow consistency (Maglev, Katran). Or use shorter-lived connections, or accept some RSTs at convergence.
- **BGP flap**: route oscillates. Use route-flap damping or hold timers.
- **Split-brain**: two POPs think they're the master. With anycast it's NOT split-brain — both serve traffic; that's the point.

## 13.5 ECMP (link-level scale-out)

Per-flow hash → multipath. Already covered in `10-routing-and-policy.md`. The interaction with bonding/anycast:

- Anycast → BGP → ECMP → multiple equal-cost paths → leaf-spine fabric.
- TCP flow consistency relies on the ECMP hash being deterministic per 5-tuple. Some hardware allows per-packet ECMP (causes reorder); usually disabled.

## 13.6 MPTCP — Multipath TCP

RFC 8684. Mainline 5.6+. One TCP connection across multiple "subflows" on different paths.

```
Client                                  Server
─ subflow 1 (over WiFi)  ◄────────────────────►
─ subflow 2 (over LTE)   ◄────────────────────►
─ subflow 3 (added later) ─────────────────►
```

Joining a subflow: `MP_JOIN` option in SYN; cryptographic handshake.

Schedulers (which subflow gets the next segment):
- `default` — lowest RTT.
- `backup` — only use marked subflows when primary is unusable.
- `redundant` — send on all (used by some real-time apps).
- `bpf` — programmable in eBPF.

Use cases:
- iPhone Siri (WiFi + LTE).
- Multi-uplink server (DC redundancy).
- Mobile LB (carriers).

Limits:
- Middlebox compatibility: many NATs/DPIs strip MP_CAPABLE → falls back to plain TCP.
- Receive buffer: must be sized for *aggregate* BDP across subflows.

Sysctl: `net.mptcp.enabled=1`. `ip mptcp endpoint add 10.1.0.2 dev eth1 backup`.

## 13.7 ARP / ND / GARP — failover plumbing

When IP moves between hosts:
- New owner sends **Gratuitous ARP** ("hey, I'm 10.0.0.5, MAC is XX:YY").
- Upstream switches update CAM table.
- Clients with stale ARP entry hit the wrong MAC → RST or no-answer.

Sysctls:
- `arp_announce` — what src IP to advertise.
- `arp_ignore` — when to reply to ARP.
- `arp_filter` — for multi-NIC servers, prefer the NIC the ARP came in on.

In IPv6, the equivalent is Neighbor Discovery (RFC 4861). The "Override" flag in NA messages triggers update.

## 13.8 BFD — Bidirectional Forwarding Detection

RFC 5880. Hello protocol with very short timers; detects link/peer failure in 100s of ms.

```
Peer A  ◄── BFD packets every 50ms ──►  Peer B
```

If 3 consecutive misses → declare down.

Integrated with BGP, OSPF, static routes. FRR, BIRD, Cisco/Juniper all support.

Use case: data-center fabric with sub-second failover for ECMP/BGP.

## 13.9 Failure-mode storytelling — the staff move

When asked "design an HA system," walk through the layers:

1. **Physical layer**: bonding (LACP), dual switch (MLAG).
2. **Link layer**: ARP convergence, gratuitous ARP on failover.
3. **L3 layer**: BGP/OSPF + BFD for sub-second detection; ECMP for load distribution.
4. **L4 layer**: anycast IP + L4 LB (Maglev/Katran) for consistent flow assignment; conntrack synchronization between LBs.
5. **L7 layer**: health checks, circuit breakers, retry budgets.
6. **App layer**: idempotent operations, retry-safe APIs, graceful drain.

Each layer is independent and serves a different failure class.

## 13.10 Conntrack synchronization (conntrackd)

For active/active LB pairs sharing flows: `conntrackd` daemon replicates conntrack entries across LBs so failover preserves state.

Modes:
- `NOTRACK` — don't track (no replication needed).
- `FT-FW` (fault-tolerant firewall) — replicate state via multicast.

In modern eBPF dataplanes (Cilium), flow tables are eBPF maps; sync via Cilium's own replication or a higher-layer LB.

## 13.11 Cloud-specific failover

- **AWS ENI move**: detach ENI from one EC2, attach to another. ~1-2s.
- **GCP routes API**: update route to point to different VM. ~5s.
- **Floating IPs**: cloud-native VIPs via API. Slow but reliable.

For sub-second: BGP + BFD (or VRRP) with cloud's BGP capability (e.g., AWS Cloud WAN, GCP Cloud Router). Or Layer 2 cluster (limited cloud support).

## 13.12 Geographic failover

DNS-based: client resolves a name; multiple A records; client picks one.
- TTL governs convergence; under 60s common for HA.
- Browsers/clients cache; convergence not instant.

GSLB (Global Server Load Balancing): smarter DNS, picks closest. F5 BIG-IP, NS1, AWS Route 53 Geolocation, dnsdist + Lua scripts.

Anycast is faster than DNS for failover (BGP convergence < DNS TTL usually).

## 13.13 Common interview probes

- **"You have 2 LB nodes. How do you make them HA?"** Walk through: VRRP active/passive, anycast active/active, BGP+BFD for sub-second, conntrack sync if stateful.
- **"Why doesn't anycast work for stateful TCP?"** Because route changes can land a packet on a different host that doesn't have the flow state. Mitigation: L4 LB tier that consistent-hashes flows to a fixed backend regardless of which POP it arrived at.
- **"How does MPTCP help mobile?"** Connection survives WiFi↔LTE handoff. No app changes; kernel manages subflows.
- **"How fast can you detect a NIC failure?"** With MII: link-state change in 10s of ms (driver-dependent). With ARP monitor: probe interval. With BFD: 100s of ms.
- **"What's the difference between LACP and active-backup?"** LACP uses both NICs simultaneously (with switch participation); active-backup is failover only.

## 13.14 Corner cases

- **LACP partner timeout** — short (1s) vs long (30s). Mismatched timers = flapping.
- **MLAG asymmetric hash** — two switches must hash identically; ASIC differences can cause one-side hash mismatch.
- **GARP not propagated** — STP convergence in old switches eats GARP. Use ARP refresh from new owner.
- **VRRP w/o preempt** — backup keeps mastership after recovery (bad if you want primary-preferred).
- **BFD over GRE** — works in some kernels with hint; pure-GRE without IP-in-IP can break BFD.
- **MPTCP server sends RST on subflow** — middleboxes that strip options can panic the connection. Fallback to plain TCP needed.
- **Cross-POP anycast routing flap** — flow lands on POP-A then POP-B; client sees out-of-order. Persistent if route oscillates.

## 13.15 Performance numbers

| Mechanism | Detection time | Recovery time |
|-----------|----------------|---------------|
| MII bonding | 10–100 ms (driver) | ~MII interval |
| ARP monitor | configured (e.g., 1s) | next interval |
| VRRP default | 3 × advert (3s) | within window |
| BGP default hold | 180s | 90–180s |
| BGP + BFD | ~150–500 ms | sub-second |
| GARP convergence | ~100 ms | depends on switch |

## 13.16 Alternative approaches

| Need | Path 1 | Path 2 | Path 3 |
|------|--------|--------|--------|
| Dual-NIC redundancy | LACP bonding | active-backup bonding | dual default routes with metric |
| Floating IP between hosts | VRRP (keepalived) | anycast + BGP | cloud floating IP API |
| Mobile multipath | MPTCP | QUIC connection migration | app-layer polling for IP change |
| DC fabric failover | BGP + BFD | OSPF + BFD | static + ip rule |
| Cross-region | anycast + BGP | GSLB DNS | dual hostnames |
| Cross-region with state | Maglev L4 LB | session affinity cookies | client-driven retry |

## 13.17 The 60-second pitch (memorize)

> "Failover is layered. At link: LACP for bandwidth + standby; MII monitor for sub-100ms detection. At IP: VRRP for active/passive shared IP, anycast for active/active shared IP. At transport: MPTCP for connection-spanning subflows. At routing: BGP + BFD for sub-second convergence. At LB: Maglev consistent hashing so flows stick to backends through churn. At app: idempotent retries, health checks, circuit breakers. The trade-offs are clean: VRRP is simple but can't scale out; anycast scales but can rebalance mid-flow; MPTCP needs middlebox tolerance; BGP needs underlay control. Pick the layer that matches your failure model."

## Must-internalize

- Bonding modes: active-backup (1), LACP (4), balance-xor (2) are the common ones; LACP is the DC default.
- VRRP: shared IP, active/passive; keepalived implements; ~1-3s failover.
- Anycast: same IP, many hosts, BGP routes; sub-second with BFD; doesn't preserve TCP state through reroute.
- MPTCP: in-kernel since 5.6; multipath single connection; iPhone Siri user.
- BGP + BFD: the modern DC failover combo.
- Conntrackd / Cilium for stateful-LB failover.
- ARP / GARP gotchas: gratuitous ARP after IP move; verify upstream switch updates.