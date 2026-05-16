# 13 · Quick Reference Cheatsheets

The night-before review. Everything important, compressed.

## 13.1 DNS record types — one-line each

| Type | Code | Purpose |
|------|------|---------|
| A | 1 | IPv4 address |
| NS | 2 | Authoritative nameserver |
| CNAME | 5 | Canonical name (alias) |
| SOA | 6 | Start of Authority |
| PTR | 12 | Pointer (reverse DNS) |
| MX | 15 | Mail exchanger |
| TXT | 16 | Text (SPF/DKIM/verification) |
| AAAA | 28 | IPv6 address |
| SRV | 33 | Service location (host+port+priority+weight) |
| DS | 43 | Delegation Signer (DNSSEC, in parent) |
| RRSIG | 46 | Signature over an RRset (DNSSEC) |
| NSEC | 47 | Next-secure (denial of existence) |
| DNSKEY | 48 | Public key (DNSSEC) |
| NSEC3 | 50 | Hashed next-secure |
| TLSA | 52 | TLS cert pinning via DNSSEC (DANE) |
| SVCB | 64 | Service binding |
| HTTPS | 65 | HTTPS service binding |
| CAA | 257 | Certificate Authority Authorization |

## 13.2 DNS header flags

| Flag | Purpose |
|------|---------|
| QR | Query (0) or Response (1) |
| OPCODE | Standard (0), Inverse (1), Status (2), Notify (4), Update (5) |
| AA | Authoritative Answer |
| TC | Truncated (retry over TCP) |
| RD | Recursion Desired |
| RA | Recursion Available |
| AD | Authentic Data (DNSSEC) |
| CD | Checking Disabled (DNSSEC) |
| RCODE | NOERROR(0), FORMERR(1), SERVFAIL(2), NXDOMAIN(3), NOTIMP(4), REFUSED(5) |

## 13.3 DHCP options — the ones to remember

| Option | Purpose |
|--------|---------|
| 1 | Subnet mask |
| 3 | Router (default gateway) |
| 6 | DNS servers |
| 12 | Hostname |
| 15 | Domain name |
| 50 | Requested IP |
| 51 | Lease time |
| 53 | DHCP message type (DISCOVER/OFFER/REQUEST/ACK/NAK) |
| 54 | Server identifier |
| 55 | Parameter request list |
| 60 | Vendor class identifier |
| 61 | Client identifier |
| 66 | TFTP server name |
| 67 | Bootfile name |
| 81 | Client FQDN |
| 82 | Relay Agent Information (subopts: 1 circuit ID, 2 remote ID) |
| 121 | Classless static route |

## 13.4 DHCP message types

| Type | Direction | Purpose |
|------|-----------|---------|
| DISCOVER (1) | Client → broadcast | Find a server |
| OFFER (2) | Server → client | Here's an IP |
| REQUEST (3) | Client → broadcast | I'll take it |
| DECLINE (4) | Client → server | Address conflicts |
| ACK (5) | Server → client | Confirmed |
| NAK (6) | Server → client | No / wrong subnet |
| RELEASE (7) | Client → server | Done with lease |
| INFORM (8) | Client → server | Configured statically, need other options |

## 13.5 Port numbers

| Port | Protocol | Service |
|------|----------|---------|
| 53/UDP+TCP | DNS | |
| 67/UDP | DHCPv4 | Server |
| 68/UDP | DHCPv4 | Client |
| 547/UDP | DHCPv6 | Server |
| 546/UDP | DHCPv6 | Client |
| 853/TCP | DoT | DNS over TLS |
| 443/TCP | DoH (and everything HTTPS) | DNS over HTTPS |
| 853/UDP | DoQ | DNS over QUIC |
| 1194/UDP | OpenVPN (Infoblox Grid) | |
| 2114/UDP | Infoblox Grid key exchange | |
| 179/TCP | BGP | |
| 514/UDP | Syslog | |
| 161/UDP | SNMP | |
| 162/UDP | SNMP trap | |
| 88/TCP+UDP | Kerberos | |
| 389/TCP | LDAP | |
| 636/TCP | LDAPS | |
| 123/UDP | NTP | |

## 13.6 RFCs to namedrop accurately

| RFC | Topic |
|-----|-------|
| 1034, 1035 | DNS basics |
| 1995 | IXFR |
| 1996 | DNS NOTIFY |
| 2131 | DHCPv4 |
| 2132 | DHCP options |
| 2136 | DNS UPDATE |
| 2308 | Negative caching |
| 3046 | DHCP Relay Agent Information option |
| 4033/4/5 | DNSSEC |
| 4291 | IPv6 addressing |
| 4941 | IPv6 privacy extensions |
| 5936 | AXFR |
| 6891 | EDNS(0) |
| 6781 | DNSSEC operational practices |
| 7348 | VXLAN |
| 7858 | DNS over TLS |
| 8484 | DNS over HTTPS |
| 8415 | DHCPv6 |
| 8945 | TSIG |
| 9250 | DNS over QUIC |
| 9460 | SVCB/HTTPS records |

## 13.7 CIDR cheat sheet

| Prefix | Hosts | Mask |
|--------|-------|------|
| /8 | 16,777,214 | 255.0.0.0 |
| /16 | 65,534 | 255.255.0.0 |
| /20 | 4,094 | 255.255.240.0 |
| /22 | 1,022 | 255.255.252.0 |
| /23 | 510 | 255.255.254.0 |
| /24 | 254 | 255.255.255.0 |
| /25 | 126 | 255.255.255.128 |
| /26 | 62 | 255.255.255.192 |
| /27 | 30 | 255.255.255.224 |
| /28 | 14 | 255.255.255.240 |
| /29 | 6 | 255.255.255.248 |
| /30 | 2 | 255.255.255.252 |
| /31 | 2 (P2P) | 255.255.255.254 |
| /32 | 1 | 255.255.255.255 |

## 13.8 DNS attacks → mitigation matrix

| Attack | Quick mitigation |
|--------|------------------|
| Cache poisoning | DNSSEC; 0x20 case randomization; source-port randomization |
| Amplification DDoS | RRL; no open recursive resolvers; BCP38 |
| Tunneling / exfiltration | Behavioral analysis on query streams; ML detection |
| DGA C2 | ML classifier on lexical features |
| NXDOMAIN flood / random subdomain | NXDOMAIN cutting; rate limit on cache miss |
| DNS rebinding | Filter private-IP answers in recursive resolver |
| Typosquatting / phishing | Threat intel feeds; brand monitoring |

## 13.9 Infoblox products — one-line each

| Product | One-liner |
|---------|-----------|
| NIOS | On-prem DDI appliance, organized as a Grid |
| Grid Master / GMC / Member / HA Pair | NIOS roles |
| BloxSYNC | NIOS replication protocol |
| BloxOne / Universal DDI | Cloud-native DDI SaaS |
| NIOS-X | On-prem footprint of BloxOne (containers) |
| Threat Defense | DNS-layer security product |
| TIDE | Threat Intelligence Data Exchange |
| Zero-Day DNS | Newly-registered-domain detector |
| Universal Asset Insights | Network asset discovery |
| SOC Insights | AI correlation for SOC analysts |
| WAPI | NIOS REST API |

## 13.10 Tradeoff one-liners (drop these in design rounds)

- "Strong consistency within a subnet, eventual across regions."
- "Anycast handles failover by BGP withdrawal; no orchestrator needed."
- "Reads cache; writes go to the canonical store; on cache miss, singleflight to deduplicate."
- "Backpressure by bounded queue + shed at the API boundary, not by blocking upstream."
- "DNSSEC validation cost is real but cacheable per RRset."
- "Migrate via dual-write with reconciliation; cut over per-tenant with rollback."
- "Tenant_id in every record; row-level isolation in the DB; per-tenant rate limits at the gateway."
- "Telemetry to Kafka, fan out to ClickHouse hot + S3 cold + threat-intel pipeline."
- "Observe with three SLIs per service: latency, error rate, freshness."
- "Local cache for speed, global cache for accuracy, shared store for source-of-truth."

## 13.11 Concurrency in Go — patterns to drop

```go
// Bounded concurrency with errgroup
g, ctx := errgroup.WithContext(ctx)
sem := make(chan struct{}, 16)
for _, item := range items {
    item := item
    sem <- struct{}{}
    g.Go(func() error {
        defer func() { <-sem }()
        return process(ctx, item)
    })
}
return g.Wait()

// Singleflight — deduplicate concurrent identical work
var sf singleflight.Group
v, err, _ := sf.Do(key, func() (interface{}, error) {
    return fetch(key)
})

// Atomic pointer swap for read-mostly state
var t atomic.Pointer[Trie]
// readers:
trie := t.Load()
// writer:
newTrie := buildTrie()
t.Store(newTrie)
```

## 13.12 Numbers to anchor on

- A single CoreDNS instance handles **20K–100K QPS** depending on plugin chain and hardware.
- A Kea DHCPv4 instance can sustain **5K–20K leases/sec** for DORA.
- ClickHouse ingests **~1M rows/sec/node** for typical schemas; queries scan **GB/sec/node**.
- Anycast PoP count for global presence: **30–100+** (Cloudflare has ~300, smaller operators ~30).
- Round-trip latency budget for a hot-path security check on DNS: **< 5 ms p99**.
- Acceptable cache hit ratio for a recursive resolver: **80–95%**.

## 13.13 The 30-second pre-interview reminder

- DNS = walked the hierarchy; cache + DNSSEC.
- DHCP = DORA; lease DB; option 82 for relay.
- IPAM = Patricia trie + invariants + integration glue.
- Grid = NIOS cluster; manual GM failover; data plane survives control-plane loss.
- BloxOne = K8s + CoreDNS + Kea + Kafka + ClickHouse.
- Threat Defense = DNS-layer security with global threat intel.
- Migration story = dual-write + WAPI shim + per-tenant cutover.
- For every design: clarify, sketch, tradeoffs, failure modes, observability.
- Have 3 questions ready for the interviewer.
- Breathe.