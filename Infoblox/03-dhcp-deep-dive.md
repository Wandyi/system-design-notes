# 3 · DHCP — Deep Dive

DHCP is the second leg of DDI. It's simpler than DNS, but the *state management* is harder: the server must track every active lease and reconcile changes across redundant servers. 
Expect at least one question on DHCP operations, lease databases, or HA modes.

## 3.1 DORA — the four-way handshake

```
Client                              Server
  |                                   |
  |---- DHCPDISCOVER (broadcast) ---->|   "anybody got an IP?"
  |                                   |
  |<--- DHCPOFFER -------------------|   "here's 10.0.0.5, take it"
  |                                   |
  |---- DHCPREQUEST (broadcast) ---->|   "I'll take 10.0.0.5 from server X"
  |                                   |
  |<--- DHCPACK ---------------------|   "confirmed, lease for 24h"
```

Why does the client `REQUEST` after `OFFER`? Because multiple DHCP servers may answer the `DISCOVER`. The `REQUEST` is broadcast so all servers see which one was chosen. The losers release their tentative reservations.

Other messages:
- **DHCPNAK** — "no, you can't have that lease" (e.g., the client moved subnets).
- **DHCPDECLINE** — client says "the offered IP is already in use" (it ARP'd and got a response).
- **DHCPRELEASE** — graceful release at shutdown (often skipped — clients just disappear).
- **DHCPINFORM** — client has a statically configured IP but wants other options (e.g., DNS servers).

## 3.2 The DHCP packet (the bits that matter)

A DHCPv4 packet is a `BOOTREQUEST` / `BOOTREPLY` (DHCP is built on BOOTP). The fields:

```
op | htype | hlen | hops
xid (transaction ID)
secs | flags
ciaddr (client IP — set if client has one)
yiaddr (your IP — server's offer)
siaddr (next-server, for PXE)
giaddr (gateway IP — set by relay agent)
chaddr (client hardware address — MAC)
sname, file (BOOTP fields)
options (variable; magic cookie 0x63825363, then TLVs)
```

The interesting work is in **options** — a variable-length TLV list. Options to know:

| Code | Option | Purpose |
|------|--------|---------|
| 1 | Subnet mask | |
| 3 | Router | Default gateway |
| 6 | DNS servers | List of resolver IPs |
| 12 | Hostname | |
| 15 | Domain name | |
| 50 | Requested IP | What the client wants |
| 51 | Lease time | In seconds |
| 53 | Message type | DISCOVER/OFFER/REQUEST/ACK/NAK/etc. |
| 54 | Server ID | Identifies the answering server |
| 55 | Parameter Request List | What options the client wants in OFFER/ACK |
| 60 | Vendor Class Identifier | e.g., "MSFT 5.0", used for fingerprinting |
| 61 | Client ID | |
| 66/67 | TFTP server / Bootfile name | PXE boot |
| 82 | **Relay Agent Information** | Sub-options: circuit ID, remote ID — written by relay |
| 121 | Classless static route | |

Option 82 is critical in ISP / large-enterprise deployments — the relay agent (an edge switch or router) injects subscriber identity, 
and the DHCP server uses this to apply policy (which pool, which DNS, what speed).

## 3.3 Relay agents

Most enterprise networks have multiple subnets. DHCP DISCOVER is L2 broadcast — it won't cross a router. So routers run a **DHCP relay** (a.k.a. IP helper) that:

1. Receives the broadcast on the local interface.
2. Fills `giaddr` with its own IP (so the server knows which subnet to allocate from).
3. Adds option 82 with circuit/remote IDs.
4. Unicasts the packet to the DHCP server.
5. Forwards the server's reply back to the client.

A staff-level question: "Our DHCP server got moved to a new VLAN — what configuration changes? What's the failure mode if the relay is misconfigured?"

## 3.4 The lease database

This is where DHCP gets interesting from a systems-design perspective. The lease DB must:

- Survive server restarts (durable).
- Handle thousands of writes/second in big networks.
- Be reconciled across redundant servers.

ISC dhcpd used a flat-file `dhcpd.leases` written sequentially (append + periodic compaction — basically a write-ahead log). **Kea** moved to a real database (MySQL/PostgreSQL backend, optional memfile).

Tradeoffs:

- **Memfile** — fast, but lease state is per-server and only one server can be authoritative.
- **MySQL/Postgres** — multiple Kea servers can share state; you pay relational-DB latency on every lease op. The DB becomes a bottleneck at very high lease churn.
- **Cassandra/NoSQL backend** — Kea has experimental support. Scales horizontally; harder to do transactional lease allocation safely.

**Concurrency**: allocating an IP must be atomic — two clients must never get the same address. With a SQL backend: row locking, or a single allocator goroutine. With a distributed backend: usually a leader per pool.

## 3.5 Kea HA modes (vs. ISC DHCPv4 failover)

ISC DHCPv4 had a complex **failover protocol** (RFC draft, never standardized): primary and secondary servers split a pool into MCLT-bounded ranges and gossiped lease state over TCP 647. Operationally painful; many edge cases.

Kea replaces this with **High Availability hooks** (a library you load). Modes:

- **hot-standby** — primary handles everything; secondary is a hot spare with replicated lease state. If the primary fails, secondary takes over. After the primary recovers, it syncs back.
- **load-balancing** — both servers process requests, partitioned by `Hash(client_ID) mod 2`. Both replicate lease state. Higher throughput, more complex.
- **passive-backup** — secondary just receives lease updates, never serves. Used for read-only failover scenarios.

State sync between Kea HA peers happens via REST + heartbeat. If the link breaks, each side enters **partner-down** state after a timeout and can serve the full pool — this risks split-brain if they both think the other is dead.

**Why this matters**: Infoblox's NIOS does its own HA. BloxOne uses Kea. Any "design DHCP HA" question is about exactly these tradeoffs.

## 3.6 DHCPv6 — different enough to call out

DHCPv6 (RFC 8415) is not a clone of DHCPv4. Differences:

- **No broadcast** — IPv6 has no broadcast; uses link-local multicast `ff02::1:2` for "all DHCP relay agents and servers".
- **DUID** — DHCPv6 Unique Identifier replaces the MAC-based identification.
- **IA_NA / IA_TA / IA_PD** — Identity Associations for non-temporary, temporary, and *prefix delegation*. PD is unique to v6: an ISP delegates a /56 or /60 prefix to your CPE, which then auto-numbers your home network.
- **Stateless DHCPv6** — clients use SLAAC for address autoconfiguration but query DHCPv6 for DNS/NTP/domain (option-only, no IP).
- **Rapid commit** — optional 2-message exchange (SOLICIT + REPLY) when there's no contention.

DHCPv6 sits *next to* SLAAC and Router Advertisements (RAs). Real IPv6 deployments are a mix: SLAAC for address, DHCPv6 for DNS, with RA bits (`M` and `O` flags) telling the client which.

## 3.7 DHCP fingerprinting

Option 55 (Parameter Request List) and option 60 (Vendor Class) form a fingerprint that often identifies the device type — Windows, macOS, iOS, Android, network printer, ISP-supplied router. Useful for asset inventory and security policy (block "unknown" devices, alert on unexpected types). Infoblox uses this heavily for asset visibility.

## 3.8 DHCP-DNS integration ("DDNS")

When a client gets a DHCP lease, it (or the DHCP server on its behalf) updates DNS — typically `hostname.corp.example.com → leased_ip`. RFC 2136 defines **DNS UPDATE** for dynamic record changes. RFC 4701 / 4703 define DHCID and the FQDN option (81) to mediate which side updates and prevent races between two clients claiming the same name.

Two flavors:
- **Server-initiated** — DHCP server updates DNS on lease grant/release.
- **Client-initiated** — client sends DNS UPDATE itself (with credentials).

Operational gotcha: orphaned DNS records when a lease expires and the cleanup fails. Infoblox's Grid handles this end-to-end because it controls both the DHCP and DNS sides — that's the value proposition.

## 3.9 DHCP at carrier scale (telco / ISP)

Operator-scale DHCP looks different from enterprise:

- Millions of CPE (cable modems, routers, set-top boxes) renewing leases.
- Heavy use of **option 82** subscriber identification.
- **RADIUS** integration for AAA: DHCP server queries RADIUS to decide what IP/pool to assign, often based on the subscriber's profile.
- **PPPoE** for some access types instead of DHCP.
- TTL/lease time tuning: long leases reduce control-plane load; short leases improve mobility.

Kea is widely deployed by ISPs (it's open-source and the perf is good). Infoblox's NIOS-X for service providers competes here.

## 3.10 Common failure modes

| Symptom | Likely cause |
|---------|-------------|
| Client gets 169.254.x.x | APIPA fallback — no DHCP response at all (server unreachable, relay misconfigured) |
| Client gets an IP from the *wrong* subnet | Multiple DHCP servers answering; or relay's giaddr is wrong |
| Duplicate IP detected (DHCPDECLINE) | Static config conflict, or two servers without coordinated pools |
| Lease database corruption | Server crash mid-write; recover from journal |
| Slow DHCP grant under load | DB contention on lease allocation; tune pool sharding |
| DHCPv6 client gets address but no DNS | SLAAC + missing stateless DHCPv6 / wrong RA flags |

## 3.11 Worked design problem — DHCP for 10M endpoints

**Prompt**: design a DHCP service for a tier-1 ISP with 10M cable modems renewing every 24h.

**Sketch**:

- 10M clients × 1 renewal/day = ~115 renewals/sec average; peaks ~3–5×.
- 4 DHCP servers behind anycast `10.0.0.1` advertised via OSPF, each colocated with a regional aggregation router.
- **Subscriber identity** via option 82 (circuit ID per CMTS port).
- **Lease backend**: Kea with a sharded MySQL cluster — shard key is the option-82 circuit ID (so lease state lives near the server that handles that region).
- **HA**: Kea HA in hot-standby per region; failover by anycast withdrawal.
- **DDNS**: per-region authoritative DNS; the DHCP server updates `<modem-id>.<region>.cable.example.net` on grant.
- **Observability**: every DORA logged to Kafka; ClickHouse for analytics; alerting on grant-rate, error-rate, lease-pool exhaustion per pool.
- **Scale**: 10M leases × ~256 bytes/lease ≈ 2.5 GB lease state; trivially in memory or DB.
- **Edge case**: post-outage, all 10M renew within minutes. Use random renewal jitter in CPE firmware, or rate-limit at the relay.

This is the kind of system-design answer they want — concrete numbers, named failure modes, identified bottlenecks.

## 3.12 Must-internalize

- DORA flow + why REQUEST is broadcast.
- Key options: 53, 54, 55, 60, 61, 82.
- Relays use `giaddr` + option 82.
- Kea HA modes: hot-standby, load-balancing, passive-backup.
- Lease DB backends: memfile, MySQL/Postgres; concurrency requires atomic allocation.
- DHCPv6 isn't DHCPv4 — DUID, IA_NA/IA_PD, SLAAC interplay.
- DDNS via RFC 2136; Infoblox does it end-to-end because they own both planes.
- Carrier-scale: option 82 + RADIUS + sharded backends.

---

## Sources

- [RFC 2131 — DHCP](https://www.rfc-editor.org/rfc/rfc2131.html)
- [RFC 2132 — DHCP Options](https://www.rfc-editor.org/rfc/rfc2132.html)
- [RFC 3046 — DHCP Relay Agent Information Option](https://www.rfc-editor.org/rfc/rfc3046.html)
- [RFC 8415 — DHCPv6](https://www.rfc-editor.org/rfc/rfc8415.html)
- [Kea documentation — ISC](https://www.isc.org/kea/)
- [Kea HA hook library](https://kea.readthedocs.io/en/latest/arm/hooks-ha.html)
- [Kea 2.0 performance summary](https://kb.isc.org/docs/kea-20-performance-tests)
- [Kea High Availability vs. ISC DHCP Failover](https://kb.isc.org/docs/aa-01617)