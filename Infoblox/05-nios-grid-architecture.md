# 5 · NIOS Grid — Architecture Deep Dive

The Grid is Infoblox's on-prem clustering architecture and the source of much of their competitive moat. It predates Kubernetes by a decade and solves a similar problem — operating dozens to thousands of appliances as a coherent system. Expect at least one question about how the Grid handles failure, replication, or scale.

## 5.1 What the Grid is, in one line

A Grid is a logical cluster of NIOS appliances (physical, virtual, or container) that share configuration, distribute services, and replicate state from a central **Grid Master** to **Grid Members** over an authenticated VPN overlay.

## 5.2 Roles

- **Grid Master (GM)** — single authoritative source for configuration and the database. Hosts the management UI and REST/WAPI. Pushes config and data to members.
- **Grid Master Candidate (GMC)** — a warm standby. Holds a full DB replica. Can be **promoted** to GM if the primary fails. Promotion is **manual** by default — Infoblox does not do automatic GM failover, because GM split-brain is worse than downtime.
- **Grid Member** — runs the data-plane services (DNS, DHCP, etc.) in a location. Pulls config and zone/lease data from GM.
- **HA pair** — two members in the same site, sharing a virtual IP via VRRP-like protocol. Used so individual sites have local HA without depending on the GM.
- **Reporting Server** — optional appliance that aggregates logs/metrics across the Grid.

## 5.3 The Grid overlay network

Every Grid member opens an authenticated **OpenVPN-based tunnel** to the Grid Master.

- **UDP 1194** — VPN data channel.
- **UDP 2114** — key exchange / control.

All Grid-internal traffic — replication, configuration push, software upgrades, reporting — flows over this overlay. Practical implications:

- The Grid Master must be reachable from every member over UDP 1194/2114.
- Firewalls and NAT in front of the GM need either a static public IP or careful NAT-traversal config.
- TLS-style mutual auth using per-Grid certificate authority.

## 5.4 BloxSYNC — the replication protocol

BloxSYNC is Infoblox's proprietary replication mechanism. It's the closest thing the Grid has to a "raft" or "paxos" — though it's not a consensus protocol; it's a **master-replica streaming** scheme.

What you can infer publicly:
- The Grid Master is the writer; members are readers/followers for their slice.
- Changes are journaled and streamed to members.
- Members re-apply changes in order to their local copy.
- Different data has different replication scope: zone data only goes to authoritative members for that view; DHCP lease state syncs between HA partners.

**Failure model**: if a member is offline, it queues catch-up; once back, the GM streams the missed delta. If a member is offline too long, it may need a full re-sync (similar to how AXFR backstops IXFR).

**Why not auto-failover the Grid Master?** Split-brain is the killer scenario — two GMs writing divergent configuration to different members. Infoblox's design choice is operator-driven promotion with verification. Newer NIOS versions add "**enhanced GM-Candidate**" where the candidate is closer to hot-standby, but promotion is still operator-blessed.

## 5.5 How services run on the Grid

DNS, DHCP, and IPAM are services hosted by members. The Grid Master is *not* in the data path of a DNS query — your laptop's recursive lookup hits a *member* (or an HA pair) at the local site, not the GM.

This is critical: **control plane (GM) failure does not stop DNS/DHCP service**. Members can serve cached zone data and existing lease assignments. They can't accept *configuration* changes while the GM is down, but the dataplane keeps running.

A common interview clarifying question if you're given the Grid: "what stops working when the GM goes down?" — Answer: UI/API/config changes; lease creation in some HA modes; reporting. What keeps working: DNS resolution, existing DHCP renewals, IPAM read-only queries.

## 5.6 Capacity, sizing, scale numbers

Public Infoblox sizing data, from product pages and admin guides:

- A Grid can have **thousands** of members in a single cluster.
- Single appliance models scale to **hundreds of thousands of QPS** for DNS.
- A Grid Master typically peaks at **single-digit thousands of API requests/sec** — it's a control plane, not data plane.
- Replication delay between GM and members: typically <1s on a healthy link.

The interesting bottleneck is the GM. A staff-level question is "you have an 800-member Grid and the GM is CPU-saturated — what do you do?" Approaches:
1. Push down work — make members own more local state (less round-tripping).
2. Shard config writes — multiple GMs per region (Infoblox has built variants of this).
3. Move to BloxOne — the cloud-native rewrite was largely motivated by GM scale limits.

## 5.7 Upgrade strategy

NIOS Grid upgrades are a sensitive operation — you're patching the appliances that resolve every name in the enterprise.

Strategy:
1. **Upgrade the Grid Master Candidate first**. Verify health.
2. Upgrade the Grid Master.
3. Roll the members in waves (e.g., 10% at a time) with health checks between waves.
4. Always have a documented rollback plan.

The Grid supports **distributed-upgrade pre-staging** — the image is downloaded to all members, but each is rebooted in a controlled order.

## 5.8 HA pairs in detail

Two members can form an HA pair at a single site:

- Share a **virtual IP** (the service IP that clients use).
- Use a VRRP-style protocol to elect active vs. passive.
- Replicate state between themselves continuously.
- Failover happens automatically (vs. GM promotion which is manual).

**Split-brain risk**: if the heartbeat link between the two HA members fails but both can reach clients, both could think they're active. Standard mitigations: stonith-style fencing (turn off the loser), or arbitration via a third party (the GM).

## 5.9 Multi-site / geo-distribution

A large enterprise might have:

- **GM and GMC** in the primary data center.
- **Members** in each remote site.
- **HA pairs** in critical sites.

DNS query routing: clients in a region resolve against the local member (configured via DHCP option 6). If that member fails, clients fall back to the HA partner or to a distant member. **Anycast** is sometimes used to make this transparent — multiple members advertise the same service IP and BGP/OSPF routes the client to the nearest live one.

## 5.10 Operations: monitoring and forensics

Critical metrics the operator watches:
- **GM↔member replication lag.**
- DNS QPS per member, NXDOMAIN rate, SERVFAIL rate.
- DHCP grant rate, lease pool utilization, declines.
- Grid VPN tunnel up/down.
- Disk / DB-journal headroom on GM.

Common incident type: "the GM was offline for 4 hours due to maintenance, and now members are out of sync." Investigation: check replication catch-up state, possibly trigger re-sync, watch for capacity blowup on the GM.

## 5.11 Failure-mode catalog (interview gold)

| Failure | Behavior | Recovery |
|---------|----------|----------|
| Single member down | HA partner takes over via VIP; remote site loses local resolver, clients hit fallback | Reboot/replace |
| Both HA members down | Site outage for DNS/DHCP; clients must use upstream/remote resolvers | On-call swap |
| GM down | UI/API offline; data plane unaffected; new config blocked | GMC promotion (manual) |
| GM↔member link partition | Member runs on cached state; lease creation may be blocked depending on policy | Restore network, deltas catch up |
| GM disk full | DB writes fail; alert fires; UI unresponsive | Free space, restart services |
| Corrupt database on GM | Restore from GMC or backup | Promote GMC, rebuild old GM |
| Software-bug regression after upgrade | Members serving wrong answers | Rollback per documented plan |
| Compromise of Grid CA | Whole Grid trust broken | Rekey Grid CA, redistribute certs (rare, brutal) |

## 5.12 NIOS-X — the modern variant

NIOS-X is the next-gen on-prem footprint. Major differences from classical NIOS:

- Ships as **containers**, not bare appliances.
- Designed to be **managed from BloxOne** (the cloud control plane), not from a local GM.
- Smaller blast radius — the cloud control plane handles configuration, the local NIOS-X just runs the data plane.

NIOS-X effectively swaps "Grid Master" for "BloxOne SaaS". An interview question that compares the two architectures is fair game.

## 5.13 Must-internalize

- Grid roles: GM, GMC, member, HA pair. GM promotion is **manual**.
- Grid overlay = OpenVPN-based, UDP 1194/2114.
- **BloxSYNC** replicates GM → members; data plane keeps serving if GM is down.
- HA pair = local high availability via VRRP-style VIP; auto-failover.
- Failure-mode catalog above — be able to recite the top 5 from memory.
- NIOS-X = containerized data plane managed from BloxOne, not a local GM.

---

## Sources

- [Infoblox Grid product page](https://www.infoblox.com/products/infoblox-grid/)
- [NIOS 9.0 docs — About Grids](https://docs.infoblox.com/space/nios90/280407969)
- [NIOS 9.0 docs — Creating a Grid Master](https://infoblox-docs.refined.site/space/nios90/319488802)
- [Infoblox Community — Grid advantages](https://community.infoblox.com/t5/nios-dns-dhcp-ipam/what-is-the-advantage-of-grid-in-nios/td-p/24913)
- [Infoblox NIOS Grid Architecture Overview](https://www.scribd.com/document/924472546/1-1-1-Infoblox-DDI-Grid-Terminology)