# 11 · Networking Fundamentals — The Basics You Can't Fumble

Infoblox is a networking-infrastructure company. The interview *will* test that you know networking at a level deeper than a typical app-engineer. This file is the cheat sheet: enough depth on each topic that you can answer at least one follow-up question.

## 11.1 The OSI and TCP/IP layer models

You'll be expected to know both, even though TCP/IP is the operational reality.

| OSI | TCP/IP | Examples | Devices |
|-----|--------|----------|---------|
| 7 Application | Application | HTTP, DNS, DHCP, SMTP | App servers |
| 6 Presentation | (Application) | TLS, JSON | |
| 5 Session | (Application) | RPC | |
| 4 Transport | Transport | TCP, UDP, QUIC | LB, FW (L4) |
| 3 Network | Internet | IP, ICMP, OSPF, BGP | Router |
| 2 Data Link | Network access | Ethernet, 802.1Q VLAN, ARP | Switch |
| 1 Physical | Network access | Copper, fiber | NIC, hub |

DNS is an L7 application that uses both UDP (default) and TCP (zone transfers, large responses). DHCP is L7 over UDP, with L2 broadcast in the discovery phase.

## 11.2 IP addressing

- **IPv4**: 32 bits, dotted decimal. ~4.3B addresses, exhausted globally.
- **IPv6**: 128 bits, hex with `::` for zero compression. `2001:db8::1`.
- **CIDR**: `10.0.0.0/24` = 256 addresses, mask `255.255.255.0`.
- **Private ranges (RFC 1918)**: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`.
- **Link-local**: `169.254.0.0/16` (IPv4 APIPA), `fe80::/10` (IPv6).
- **Multicast**: `224.0.0.0/4` (IPv4), `ff00::/8` (IPv6).
- **Loopback**: `127.0.0.1/8`, `::1`.
- **Carrier-grade NAT**: `100.64.0.0/10`.

Know the math: `/24` = 256 addresses (254 usable, .0 = network, .255 = broadcast). `/30` = 4 (2 usable) — common for point-to-point links. `/31` = 2 (both usable for P2P per RFC 3021). `/32` = a single host.

## 11.3 TCP vs. UDP — when to pick which

**TCP**: connection-oriented, reliable, ordered, congestion-controlled. Handshake (SYN, SYN-ACK, ACK). Used by HTTP, DNS-over-TCP, BGP, SMTP.

**UDP**: connectionless, unreliable, no ordering, no congestion control by default. Used by DNS, DHCP, NTP, VoIP, QUIC's underlay.

**QUIC**: built on UDP, but provides reliable streams, congestion control, encryption, multiplexing. Used by HTTP/3 and DoQ.

When you'd pick UDP for an app: low-latency, message-oriented, can tolerate or repair loss yourself, want to multiplex many things without head-of-line blocking.

When you'd pick TCP: you want OS to handle ordering/retransmission and you can pay the handshake cost.

## 11.4 ARP and Neighbor Discovery

- **ARP** (IPv4): "who has this IP, tell me your MAC". L2 broadcast on the local segment.
- **Neighbor Discovery (NDP)** (IPv6): equivalent function, plus router discovery, prefix learning, duplicate address detection.

Gratuitous ARP — a host announces its own MAC for its own IP. Used in HA failovers to refresh switches' MAC tables.

ARP cache poisoning is a real L2 attack vector. Defended via dynamic ARP inspection (DAI) on switches, plus 802.1x port auth.

## 11.5 Routing — OSPF, BGP, anycast

### OSPF (Open Shortest Path First)

Interior gateway protocol. Link-state. Each router floods LSAs (Link-State Advertisements); all routers compute the same shortest-path tree using Dijkstra. Areas reduce LSA scope. Used inside an AS. Convergence: seconds.

### BGP (Border Gateway Protocol)

The protocol of the Internet. Path-vector. Routers (BGP speakers) advertise reachability to neighbors. Policy-driven (not necessarily shortest path).

For an infrastructure engineer: BGP is how you advertise an anycast prefix from many sites. Each PoP runs a BGP session with the upstream and announces the service prefix. The Internet routes to the nearest.

- **eBGP** — between ASes. TTL=1 typically, multi-hop possible.
- **iBGP** — within an AS. Often via route reflectors at scale.
- **Convergence**: minutes globally; seconds locally.

### Anycast

Multiple physical sites advertise the **same IP prefix**. The network routes packets to whichever is "best" per BGP policy.

Properties:
- **Implicit failover** — if a PoP withdraws, traffic routes elsewhere.
- **Implicit geo-proximity** — usually the nearest by BGP wins.
- **Stateless services are easy** — DNS, NTP. Stateful services struggle (TCP sessions can migrate mid-flow if BGP re-converges).

DNS roots and modern recursive resolvers (1.1.1.1, 8.8.8.8) are anycast.

## 11.6 NAT and stateful firewalls

NAT (Network Address Translation) rewrites source/destination IPs and ports as packets cross a boundary. Two main flavors:

- **Source NAT (SNAT) / PAT** — outgoing connections from many internal IPs share one external IP, demultiplexed by port. The default home-router behavior.
- **Destination NAT (DNAT) / port forwarding** — external traffic to `external_ip:port` is rewritten to an internal host.

Stateful firewalls track connection state (5-tuple: src IP, src port, dst IP, dst port, proto) and allow return traffic. They have a finite connection table — relevant for DDoS sizing.

Implications for DDI:
- DHCP must work for clients behind NAT (it does — NAT is upstream, DHCP is local).
- DNS responses larger than MTU may fragment; some NATs drop fragments. EDNS + TCP fallback mitigate.

## 11.7 MTU and fragmentation

The **Maximum Transmission Unit** is the largest packet a link can carry. Standard Ethernet: 1500 bytes. Jumbo: 9000. Internet path MTU is often constrained by tunnels (PPPoE: 1492, IPSec: 1438ish).

If a packet is too big and the **DF (Don't Fragment)** bit is set, the router drops it and returns ICMP "Fragmentation Needed". The sender lowers its MTU. Known as **Path MTU Discovery (PMTUD)**.

**Black-holing**: some networks drop ICMP, breaking PMTUD. Symptom: small packets work, large ones don't. Workaround: TCP MSS clamping at the edge (rewrite SYN packets to advertise a smaller MSS).

For DNS: a response > MTU may fragment over UDP, and stateless middleboxes may drop fragments. This is why TC=1 → TCP fallback exists and why EDNS buffer size negotiation matters.

## 11.8 VLAN, VXLAN, VRF

- **VLAN (802.1Q)** — L2 segmentation. 12-bit VLAN ID = 4096 max VLANs. Tag added to Ethernet frame.
- **VXLAN (RFC 7348)** — L2 over L3 tunneling. 24-bit VNI = 16M segments. Encapsulates Ethernet in UDP. Used in modern data centers (Cisco ACI, Arista EOS, K8s with Calico/Cilium VXLAN mode).
- **VRF (Virtual Routing and Forwarding)** — multiple routing tables on one router. Each VRF is an isolated routing domain. The same `10.1.1.1` can exist independently in two VRFs.

For IPAM: each VRF is a separate address-space "context". Two customers' `10.0.0.0/8` in separate VRFs are not the same block.

## 11.9 IPSec, GRE, WireGuard — site-to-site tunnels

- **IPSec** — layer-3 encryption (AH for integrity, ESP for confidentiality+integrity). Used for VPNs. Complex to configure.
- **GRE** — encapsulation only, no encryption. Useful when you need L2 over L3 but want simpler than VXLAN.
- **WireGuard** — modern alternative. Simpler, faster, uses Curve25519 + ChaCha20. Becoming the default for new VPN deployments.
- **OpenVPN** — what Infoblox's Grid uses. TLS-based, runs over UDP/TCP. Slower than WireGuard but well-tested.

## 11.10 TLS — the parts a staff engineer must know

- **TLS 1.2** vs. **TLS 1.3**: 1.3 has fewer round trips (1-RTT default, 0-RTT for resumption), simpler cipher suites, mandatory forward secrecy.
- **mTLS** — both client and server present certs. Used heavily in service-to-service traffic.
- **SNI** — Server Name Indication, plaintext in the ClientHello. Lets a server choose a cert based on hostname. **Encrypted Client Hello (ECH)** is the WIP replacement.
- **ALPN** — Application-Layer Protocol Negotiation. How the client says "I want HTTP/2" or "I want DoT".
- **Certificate transparency** — public log of issued certs. Lets domain owners detect rogue issuances.

For DoT and DoH: TLS termination is hot-path. Session resumption is critical for performance.

## 11.11 Load balancers — L4 vs. L7

- **L4 LB** — sees TCP/UDP only. Faster, simpler, can do DR (Direct Server Return) where reply bypasses LB. Examples: AWS NLB, HAProxy in TCP mode, IPVS.
- **L7 LB** — sees HTTP. Can route by header/path/cookie, terminate TLS, do retries, do circuit breaking. Examples: AWS ALB, Envoy, Nginx, HAProxy in HTTP mode.

DNS load balancers typically operate at L4 (UDP/TCP/853) or are anycast PoPs.

## 11.12 ICMP — the protocol that's not just `ping`

- **Echo request/reply** — ping.
- **Destination Unreachable** — host/port unreachable, fragmentation needed (critical for PMTUD).
- **Time Exceeded** — TTL hit zero. Powers `traceroute`.
- **Redirect** — "use a better gateway for that". Often disabled for security.

If you blanket-block ICMP, you break PMTUD. Recommendation: allow Type 3 (Destination Unreachable) and Type 11 (Time Exceeded) inbound.

## 11.13 Multicast

Used for:
- IGMP/MLD for group management.
- PIM for multicast routing.
- VRRP (router redundancy) — advertises on `224.0.0.18`.
- mDNS / Bonjour — `224.0.0.251`.
- Some application protocols.

For DDI: usually not a concern at the protocol level, but switching infrastructure choices affect multicast scalability (IGMP snooping, etc.).

## 11.14 BGP for service operators (anycast specifics)

If you're going to claim experience with anycast, expect probes:

- **Local-pref** — outbound policy. Higher wins. Used to prefer specific upstreams.
- **MED (Multi-Exit Discriminator)** — hint to neighbor about preferred entry point. Lower wins. Often ignored.
- **AS-path prepending** — make your route look "longer" to discourage use. Crude but effective.
- **Communities** — tags on routes for policy. RFC 4360.
- **RPKI / Route origin authorization** — cryptographic verification that an AS is allowed to originate a prefix. Mitigates BGP hijacks.

For anycast: each PoP runs BGP, announces the anycast prefix; the global Internet path-vectors traffic to nearest. If a PoP goes unhealthy, your health-check pulls the BGP announcement → traffic shifts.

## 11.15 Service discovery in modern environments

- **DNS-based** — SRV records, K8s pod DNS. Slow updates (cache TTL bound), works everywhere.
- **Service mesh** — Envoy / Istio / Linkerd. Sidecar proxies discover endpoints from a control plane (Pilot, Consul, etc.) and do health checks.
- **API-gateway based** — Kong, Ambassador, AWS API Gateway. Centralized routing.

The CoreDNS Kubernetes plugin is the canonical service-discovery DNS implementation.

## 11.16 Tools you should be able to use in an interview

- `dig` / `dig +trace` / `dig +dnssec` — DNS troubleshooting.
- `nslookup` — older equivalent.
- `tcpdump` / `wireshark` — packet inspection.
- `traceroute` / `mtr` — path analysis.
- `ip` / `iproute2` — network config on Linux.
- `iptables` / `nftables` — Linux firewall.
- `ss` — socket statistics (replaces `netstat`).
- `curl --resolve` — test DNS-bypassing HTTP.

Be ready to say "I'd run `dig +trace example.com` to see exactly which delegation step is failing".

## 11.17 Must-internalize

- Layer model + know which protocols live where.
- CIDR math fluency.
- TCP/UDP/QUIC tradeoffs and when each is right.
- BGP & anycast — at least conceptually solid.
- NAT, MTU/PMTUD, ICMP gotchas.
- VLAN/VXLAN/VRF distinctions, especially VRF as the IPAM context dimension.
- TLS 1.3 + mTLS + ALPN basics.
- ICMP types you must not block.
- One DNS troubleshooting flow with `dig +trace`.