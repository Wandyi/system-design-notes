# 4 · IPAM — IP Address Management

IPAM is the "I" in DDI. It's the *management plane* that sits over DNS and DHCP: a system of record for every IP block, subnet, address, and the relationships between them. Unlike DNS/DHCP, there's no IETF RFC for IPAM — every vendor invents their own data model. Infoblox's IPAM data model is one of the strongest pieces of the platform.

## 4.1 What IPAM owns

- **IP blocks / supernets** — e.g., the corporate `10.0.0.0/8` aggregate.
- **Networks / subnets** — `10.1.0.0/16`, `10.1.1.0/24`, etc.
- **Address pools** — DHCP pools inside a subnet, e.g., `10.1.1.50–10.1.1.200`.
- **Reservations** — static IPs reserved for specific hosts (printers, infrastructure).
- **Fixed addresses** — MAC-bound DHCP reservations.
- **Hosts** — logical objects that tie together a name, one or more IPs, and DHCP/DNS configuration.
- **VLANs / VRFs** — overlay context (a given subnet might exist in multiple VRFs).
- **Cloud objects** — VPCs, subnets, ENIs imported from AWS/GCP/Azure.

The Infoblox model overlays all of these into a hierarchy that lets a single object (a "host") represent the full DNS-name-plus-DHCP-lease-plus-IP-allocation relationship.

## 4.2 The IPAM data model — building it from scratch

A solid mental model: **a tree (or forest) of CIDR blocks**, with leaf attributes for allocations.

```
Block:        10.0.0.0/8        (corporate aggregate)
  ├─ Network: 10.1.0.0/16       (data center)
  │    ├─ Subnet: 10.1.1.0/24
  │    │    ├─ Static: 10.1.1.1 (gateway)
  │    │    ├─ Pool:   10.1.1.50 - 10.1.1.200 (DHCP)
  │    │    └─ Host:   10.1.1.10 (server1, FQDN, MAC)
  │    └─ Subnet: 10.1.2.0/24
  └─ Network: 10.2.0.0/16       (branch offices)
```

Key invariants the system enforces:

1. **No overlap among children of the same parent** at the same level/context.
2. **Children are strictly contained** in the parent CIDR.
3. **Reserved/fixed/dynamic** at the leaf are mutually exclusive for a given IP.
4. **DHCP pools** must lie within a subnet but not overlap statics/fixed.

## 4.3 Data structures for fast IPAM queries

The non-obvious systems problem in IPAM is fast "find next available subnet / IP" queries over big spaces. A naive linear scan over `2^32` IPv4 addresses doesn't scale, and IPv6 (`2^128`) makes naive impossible.

**Standard approach**: a **radix tree** (a.k.a. Patricia trie / binary trie) keyed by the CIDR.

```
                              (root)
                             /      \
                           0          1
                         /   \      /   \
                      ...  10.0   192.0  ...
```

Each node is a CIDR; each internal node may have an "allocation summary" rolling up the children: total addresses, used, free, fragmentation. With this, queries like:

- "Find a free /24 inside `10.1.0.0/16`" → walk down, find an internal node whose subtree has enough free space, allocate at the smallest matching node.
- "Find the next available IP in `10.1.1.0/24`" → scan a bitmap on the leaf node; very fast for /24, harder for /8.

For IPv6, you can't materialize bitmaps. You store **allocations as ranges/intervals** in an interval tree or sorted set; "next free range" becomes a tree query.

**A real IPAM** also needs:

- **VRF / context dimension** — the radix tree is per-VRF, so the same `10.0.0.0/8` can exist in multiple contexts.
- **Concurrency control** — two engineers can't both claim the same subnet. Use optimistic concurrency with version numbers, or a per-parent-block lock.
- **Audit log** — every allocation/deallocation must be recorded.

## 4.4 Conflict detection

The classic IPAM job: detect when two devices claim the same IP.

- **ARP-based** — sweep the subnet with ARP requests; multiple replies for one IP = conflict.
- **DHCP log analysis** — two leases for the same address (lease race).
- **Static vs. dynamic** — a static IP overlaps the DHCP pool; client gets a lease, then conflict on ARP.
- **Rogue DHCP server** — clients in your subnet get leases from a server you don't run. Detect by listening for DHCPOFFERs that didn't originate from your server (Infoblox does this).

In a multi-VLAN environment, you need an agent (NIOS-X, BloxOne host) on each VLAN to do the L2 probing.

## 4.5 Integration with DNS and DHCP

What unifies DDI: an IPAM "host" object can be the source-of-truth for:

- A **forward DNS record** (`host.corp.example.com → IP`).
- A **reverse DNS record** (`PTR` in `in-addr.arpa`).
- A **DHCP fixed address** binding the IP to a MAC.
- An **IPAM allocation** taking the IP out of the free pool.

In NIOS, creating a single "host" object writes all four. The point: **transactional consistency**. You don't want a DNS record for an IP that isn't allocated, or a DHCP reservation that conflicts with the static IPAM record.

This is the implementation challenge — you're effectively running a small distributed transaction across DNS, DHCP, and IPAM subsystems. Infoblox uses a single underlying database (in NIOS, a custom store; in BloxOne, microservices that share a transactional control plane).

## 4.6 IPAM at cloud scale: import & sync

Modern enterprises have IPAM data in many places:

- On-prem networks (the classical IPAM).
- AWS VPCs (managed via tags / VPC API).
- GCP VPCs.
- Azure VNets.
- Kubernetes CNI (each pod gets an IP from a per-node CIDR slice).

Infoblox's Universal DDI ingests from all of these, normalizes to its own data model, and shows them in one tree. Sync strategies:

- **Discovery agents** — pull data from each cloud's API on a poll loop. Tag/label-driven.
- **Push events** — receive cloud-event streams (e.g., AWS EventBridge for VPC creation).
- **Reconciliation loop** — periodic full-state diff to recover from missed events.

The hard part is *bi-directional sync* — letting a user allocate from Infoblox and have it appear in AWS, or vice versa. In practice, most deployments use Infoblox as the system of record and either provision into AWS via Terraform/CFN or just observe.

## 4.7 API surface

Infoblox NIOS exposes the **WAPI** — a REST API. Endpoints model the data hierarchy (networks, ranges, fixedaddresses, hosts, records). Operations are typically POST/PUT/DELETE with strict input validation.

BloxOne is gRPC-internally + REST/GraphQL externally.

Common interview question: "design the IPAM API for allocating an IP from a subnet." Sketch answer:

```http
POST /v1/networks/{network-cidr}/next-available-ip
Content-Type: application/json
{
  "count": 1,
  "exclude": ["10.1.1.1", "10.1.1.255"],
  "tags": {"app": "redis-prod"}
}

200 OK
{
  "allocations": [
    { "ip": "10.1.1.42", "lease_id": "uuid", "expires_at": "..." }
  ]
}
```

Important details:
- **Idempotency-Key** header to avoid double-allocating on retry.
- **Optimistic concurrency** — if-match etag on the network resource.
- **Conditional reservations** — allow temporary "soft" allocation that expires if not confirmed within N seconds (so a workflow can roll back).

## 4.8 Allocation strategies

When asked for an IP in a subnet, which one do you return?

- **Lowest free** — `10.1.1.2` first. Simple, deterministic. Bad: tight clustering, no jitter for security.
- **Sequential after last allocated** — round-robin. Easy with a per-subnet counter.
- **Random** — pick a random free address. Spreads allocations, defeats simple scanners.
- **Bitmap-based** — find first free bit. Fast.

For DHCP pools, the policy is usually "least-recently-used" so a recycled IP isn't reissued immediately (helps DNS caches and ARP tables settle).

## 4.9 IPv6-specific complications

- Address space is so vast that *exhaustion* isn't a concern, but *organization* is. Most enterprises subnet at /64 per VLAN regardless of host count.
- **EUI-64** addresses include the MAC; privacy extensions (RFC 4941) randomize the host part periodically — making "find this device by IP" unreliable.
- **SLAAC** lets clients self-assign without contacting DHCPv6. IPAM has to *observe* (via ARP/ND snooping) rather than allocate.
- **Prefix delegation** (DHCPv6-PD) — an ISP gives the customer a /48 or /56; the customer router subnets it. IPAM must track delegated prefixes as first-class allocations.

## 4.10 Worked design problem — IPAM for 100M addresses

**Prompt**: design an IPAM service for a large telco managing 100M IPv4 addresses and a /32 IPv6 allocation, across 5,000 PoPs.

**Sketch**:

- **Data model**: Patricia trie per VRF, persisted to a relational DB. Each block/subnet is a row; children reference parent via `parent_id`. Add a `range(start_ip, end_ip)` index for range overlap queries (Postgres `GiST` on `inet` works).
- **Hot path**: `next_available_ip(subnet)` — Lua script in Redis on a cached bitmap; periodic sync to canonical DB.
- **Audit log** in Kafka, replayed into a search index (Elasticsearch / ClickHouse).
- **API**: gRPC for service-to-service, REST for human consumers.
- **Multi-region**: regional DBs partitioned by region (each PoP belongs to one region). Cross-region reads go through a federation layer.
- **Consistency model**: strong consistency *within* a subnet (locking on allocation), eventual across regions.
- **Throughput estimate**: 100M IPs × ~5% churn/day = 5M ops/day = ~60 ops/sec average, ~600/sec peak. Modest at the global level; the per-subnet hot-spotting is what you scale for.
- **Edge case**: a misbehaving caller hammering one subnet — token-bucket per `(caller, subnet)`.
- **Observability**: time-series of pool utilization per subnet; alerts at >80%.

## 4.11 Must-internalize

- IPAM = source of truth for all IP allocations + the glue between DNS/DHCP and the IP plane.
- Data structure: Patricia/radix trie over CIDR space; bitmaps at leaf level for IPv4.
- Invariants: no overlap, strict containment, mutual exclusion of allocation types.
- Conflict detection via ARP, DHCP logs, rogue-server detection.
- Cloud integration: discovery + reconciliation, not bidirectional sync by default.
- Atomic "next available IP" with idempotency and optimistic concurrency.
- IPv6 changes the game: SLAAC, privacy extensions, prefix delegation.

---

## Sources

- [Infoblox NIOS docs — Networks and address ranges](https://docs.infoblox.com/space/nios90/)
- [Infoblox WAPI documentation](https://www.infoblox.com/wp-content/uploads/infoblox-deployment-infoblox-rest-api.pdf)
- [How IPAM helps prevent IP conflicts](https://www.manageengine.com/products/oputils/blog/how-ipam-helps-prevent-ip-conflicts-rogue-devices-and-subnet-chaos.html)
- [What is IPAM (IP Address Management)](https://www.manageengine.com/products/oputils/what-is-ipam.html)
- [RFC 4291 — IPv6 addressing architecture](https://www.rfc-editor.org/rfc/rfc4291.html)
- [RFC 4941 — IPv6 privacy extensions](https://www.rfc-editor.org/rfc/rfc4941.html)