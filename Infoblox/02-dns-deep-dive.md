# 2 · DNS — Deep Dive

DNS is *the* topic at Infoblox. A staff engineer is expected to discuss it at the packet level, at the system-architecture level, and at the operational-failure-mode level. 
The interviewer will likely probe at least two layers deep on whatever you bring up.

## 2.1 The DNS hierarchy in one diagram

```
                       . (root)
                      /     \
                   .com    .org    .io    .uk
                    |        |       |     |
              example.com  example.org    bbc.co.uk
                  /   \
            www.example.com   api.example.com
```

- **Root zone** — 13 named root server letters (A through M), but each is anycast to hundreds of physical sites globally. Operated by 12 different organizations.
- **TLD nameservers** — .com is operated by Verisign, .org by PIR, ccTLDs by national registries.
- **Authoritative nameservers** — answer queries for a specific zone (the zone owner's servers).
- **Recursive resolvers** — what your laptop talks to. Walks the hierarchy on behalf of the client. Caches results.
- **Stub resolvers** — the OS-level library on the client that knows how to ask a recursive resolver.

## 2.2 Recursive vs. iterative resolution

Two query types, often confused.

**Recursive query**: "Find me the answer, don't come back until you have it." The recursive resolver does all the work.

**Iterative query**: "Tell me the best you know; I'll go ask the next server." Each step returns either an answer or a referral.

The flow when your laptop queries `www.example.com`:

1. Laptop's stub resolver sends a **recursive** query to the recursive resolver (configured via DHCP, e.g., `1.1.1.1`, `8.8.8.8`, or the corporate DNS).
2. Recursive resolver checks its cache. Cache miss.
3. Recursive resolver sends an **iterative** query to a root server: "who handles .com?"
4. Root returns a referral: "ask `a.gtld-servers.net` for .com."
5. Recursive sends iterative query: "who handles example.com?"
6. .com returns referral: "ask `ns1.example.com`."
7. Recursive sends iterative query: "what's the A record for `www.example.com`?"
8. Authoritative answers: `93.184.216.34`.
9. Recursive resolver caches the answer for TTL seconds and returns it to the laptop.

**Why this matters in an interview**: a question like "design a recursive resolver" or "what happens when a recursive resolver's cache is poisoned" needs this model in your head. Infoblox sells both authoritative DNS (NIOS zones) and recursive resolvers (Threat Defense), so you must be able to discuss both sides.

## 2.3 DNS record types — the ones to know cold

| Record | Purpose | Notes |
|--------|---------|-------|
| **A** | Name → IPv4 | Most common. |
| **AAAA** | Name → IPv6 | "Quad-A". |
| **CNAME** | Name → another name (alias) | Cannot coexist with other records at the same name. No CNAME at zone apex. |
| **NS** | Name → nameserver for the zone | Delegation. |
| **MX** | Mail server for a domain | Has a priority. |
| **TXT** | Free-form text | Used for SPF, DKIM, domain ownership verification. |
| **SOA** | Start of Authority — zone metadata | Serial number, refresh, retry, expire. |
| **PTR** | IP → name (reverse DNS) | Lives in `in-addr.arpa` / `ip6.arpa`. |
| **SRV** | Service location (host + port) | Used by AD, SIP, LDAP, K8s service-discovery. |
| **CAA** | Which CAs may issue certs for this domain | Enforced by CAs at issuance. |
| **DS / DNSKEY / RRSIG / NSEC / NSEC3** | DNSSEC | Chain-of-trust records. |
| **SVCB / HTTPS** | Service binding (modern alternative to A+SRV) | RFC 9460. Encodes ALPN, port, IP hints. |
| **TLSA** | Cert pinning via DNSSEC (DANE) | Rarely used outside email (MTA-STS uses HTTPS instead). |
| **CDS / CDNSKEY** | Child-published DS records for parent | Automates DNSSEC key rollover. |

**Common trick question**: "Why can't you have a CNAME at the zone apex (`example.com.`)?" Because the apex must have SOA and NS records, and a CNAME may not coexist with any other record at the same owner name (RFC 1034). Modern providers use `ALIAS`/`ANAME` (non-standard) or `HTTPS`/`SVCB` to work around this.

## 2.4 Anatomy of a DNS message

```
+---------------------+
|       Header        |  12 bytes: ID, flags, counts
+---------------------+
|      Question       |  what's being asked
+---------------------+
|       Answer        |  answer RRs
+---------------------+
|     Authority       |  authority NS RRs
+---------------------+
|     Additional      |  glue records, EDNS OPT, etc.
+---------------------+
```

Header flags worth knowing:
- **QR**: query (0) or response (1)
- **OPCODE**: standard query, inverse, status, notify, update
- **AA**: authoritative answer
- **TC**: truncated (response didn't fit in UDP, retry on TCP)
- **RD**: recursion desired (client asks resolver to recurse)
- **RA**: recursion available (server can recurse)
- **RCODE**: NOERROR(0), FORMERR(1), SERVFAIL(2), NXDOMAIN(3), NOTIMP(4), REFUSED(5)

UDP DNS messages were originally capped at 512 bytes. **EDNS(0)** (RFC 6891) lets clients advertise a larger UDP buffer size (typically 4096) via an OPT pseudo-record in the Additional section. Without EDNS, large responses get TC=1 and clients fall back to TCP.

## 2.5 Caching, TTLs, and negative caching

Each RR carries a TTL — how long resolvers may cache it. Engineering tradeoffs:

- **Low TTL (e.g., 60s)** → fast failover, more query load on authoritative servers.
- **High TTL (e.g., 1 day)** → less load, but stale answers when you need to fail over.
- **Negative caching** (RFC 2308) — `NXDOMAIN` is cached too, bounded by the *minimum* of the SOA `MINIMUM` field and the SOA's own TTL.

Common operational mistake: trying to fail over a service with a 24h TTL on the A record. The cache won't roll over in time; users see the old IP for up to 24h.

## 2.6 Zone transfers: AXFR and IXFR

How secondary nameservers replicate from a primary.

- **AXFR** (RFC 5936) — full zone transfer. TCP only. The whole zone is shipped.
- **IXFR** (RFC 1995) — incremental zone transfer. Only changes since the secondary's serial number.
- **NOTIFY** (RFC 1996) — primary sends an unsolicited NOTIFY to secondaries when the zone changes; secondaries pull via IXFR/AXFR.

**Authentication**: use **TSIG** (RFC 8945) — a shared HMAC key signs the transfer. Or **SIG(0)** with asymmetric keys. Cleartext zone transfers without TSIG are a security incident waiting to happen.

NIOS uses a proprietary mechanism (BloxSYNC) between Grid members, not AXFR — but it still supports AXFR/IXFR with non-Grid secondaries.

## 2.7 DNSSEC — the deep cut

DNSSEC adds cryptographic origin authentication. It does **not** encrypt — anyone watching the wire still sees your queries and answers. It just lets a validating resolver prove the answer wasn't forged.

### Key concepts

- **ZSK** (Zone Signing Key) — short-lived, signs the zone records. Produces `RRSIG` records.
- **KSK** (Key Signing Key) — long-lived, signs only the DNSKEY RRset. Its hash (`DS` record) is published in the *parent* zone.
- **Chain of trust**: root has a trust-anchor KSK. The root signs `.com` DS. `.com` signs `example.com` DS. `example.com` signs its own records. A validating resolver verifies every link.

### Records introduced

- `DNSKEY` — public keys.
- `RRSIG` — signature over an RRset, with key tag, algorithm, validity window.
- `DS` — hash of a child's KSK, in the parent zone.
- `NSEC` / `NSEC3` — authenticated denial-of-existence ("no record exists between these two names"). NSEC3 adds salted hashing to prevent zone enumeration.

### Operational pain points (high signal in interviews)

- **Key rollover** (RFC 6781, RFC 7583) — KSK rollover requires updating the DS in the parent. Get it wrong and the whole zone is bogus.
- **Validation failures** are catastrophic. The whole zone goes dark to validating resolvers.
- **Algorithm rollover** — moving from RSA-SHA256 (algorithm 8) to ECDSA-P256-SHA256 (13) needs care; some old resolvers don't support newer algorithms.
- **NSEC3 vs NSEC**: NSEC3 prevents zone walking; NSEC is simpler but lets attackers enumerate.

Read [Slack's DNSSEC rollout postmortem](https://slack.engineering/what-happened-during-slacks-dnssec-rollout/) for a real-world horror story.

## 2.8 Encrypted DNS: DoT, DoH, DoQ

| | Port | Transport | Pros | Cons |
|--|------|-----------|------|------|
| **DoT** (RFC 7858) | 853 TCP | TLS | Easy to identify and operate on-network; OS-wide config | Distinguishable on the wire (port 853) |
| **DoH** (RFC 8484) | 443 TCP | HTTPS | Hidden in normal HTTPS traffic; per-app override possible | Hidden from corporate visibility — controversial |
| **DoQ** (RFC 9250) | 853 UDP | QUIC | 0-RTT, multiplexed, no head-of-line blocking | Newer, less universal support |

**Why Infoblox cares**: in a corporate environment, DoH bypasses the recursive resolver the security team built. 
Apps (browsers especially) can ship pinned DoH endpoints that ignore the OS DNS config. Threat Defense customers expect Infoblox to detect or block this — 
through firewall policy, deep packet inspection, or by being the configured DoH endpoint themselves.

Common interview question: "How would you design a corporate DNS gateway that supports DoH but enforces policy?" 
Answer: terminate DoH at the gateway, decode the DNS message, apply policy (RPZ, blocklist, response policy), forward upstream as plain DNS or DoH.

## 2.9 DNS attacks and mitigations

### Cache poisoning (Kaminsky-style)

Attacker forges a response that arrives before the legitimate one, with a matching transaction ID and source port. Mitigations:

- **Random source ports** + 16-bit transaction ID — together 32 bits of entropy.
- **0x20 encoding** — randomize case in the query name (`wWw.eXamPle.cOm`); reject responses with mismatched case.
- **DNSSEC** — cryptographic validation, the proper fix.

### DNS amplification (DDoS reflection)

Send a small query (60 bytes) with a spoofed source IP to an open recursive resolver; get a large response (1 KB+) sent to the victim. Amplification factor 30–50×. Mitigations:

- Don't run **open resolvers** — restrict recursion to trusted clients.
- **Response Rate Limiting (RRL)** on authoritative servers.
- BCP 38 — ISPs filter spoofed source addresses (sadly under-deployed).

### DNS tunneling / exfiltration

Encode arbitrary data into DNS queries (e.g., `<base32-encoded-payload>.attacker.com`). The attacker's authoritative server receives the data. Detection:

- Anomalous query rate to a single domain.
- Long subdomain labels.
- Entropy of query names (random-looking strings).
- Uncommon record types (TXT, NULL).
- ML models on query streams — this is exactly what Infoblox Threat Defense does.

### Domain Generation Algorithms (DGA)

Malware generates pseudo-random domain names daily so command-and-control can't be blocked by static blacklists. 
The attacker only needs to register one. Detection: ML classifier on lexical features of domain strings.

### NXDOMAIN floods / pseudo-random subdomain attacks

Attacker queries millions of random subdomains of a victim's domain. Each query must traverse to the authoritative server (cache misses), exhausting the authoritative or recursive resolver. 
Mitigation: rate limiting on cache miss, NXDOMAIN cutting.

### DNS rebinding

Attacker controls a domain with a very short TTL. The browser fetches the page, then the attacker's DNS server returns an internal IP (e.g., `192.168.0.1`) on the next request, letting the attacker's JS attack internal services from same-origin. 
Mitigation: filter private-IP responses in recursive resolvers ("DNS rebinding protection" — most resolvers do this by default).

## 2.10 Operationally interesting numbers

- A busy recursive resolver handles **100K–1M queries per second**.
- Cloudflare ingests **~1 billion queries per second** across their anycast network at peak.
- Average DNS query/response is ~60–200 bytes.
- Authoritative DNS responses can be amplified to a few KB with EDNS.
- Real-world cache hit ratios at a recursive resolver: **80–95%** depending on workload diversity.

## 2.11 Authoritative vs. recursive — different system designs

A staff-level question is often disguised as "design a DNS server". Always clarify: authoritative or recursive?

| | Authoritative | Recursive |
|---|---------------|-----------|
| Source of truth | Yes — owns the zone data | No — caches answers from elsewhere |
| Cache | No (the data *is* the cache) | Yes, central design concern |
| Anycast | Critical (Anycast lets you serve from many sites with one IP) | Often anycast inside a network, but cache locality matters |
| State | Zone data must be replicated to all instances consistently | Per-node cache; eventual consistency fine |
| Failure mode | Stale zone data | Stale cache, or fall back to authoritative |
| Hot-path work | Look up the right record set, optionally sign with DNSSEC | Walk the delegation chain, validate DNSSEC |

## 2.12 Building a high-performance DNS data plane (Infoblox-flavored)

Some tactics they actually use, or that you should bring up if asked:

- **Userspace networking** — DPDK or XDP/eBPF to bypass the kernel network stack. CoreDNS isn't there by default, but high-end forwarders/load balancers are.
- **Lock-free / sharded LRU cache** keyed by `(qname, qtype, qclass)`.
- **Negative caching with proper TTL bounds** (RFC 2308).
- **Pre-fetching** — refresh popular records before TTL expiry to avoid the latency spike.
- **Connection reuse** — DoT/DoH benefit hugely from TLS session resumption and HTTP/2 multiplexing.
- **Anycast** the recursive resolver to drop latency.

## 2.13 Must-internalize

- The query path: stub → recursive (recursive walk: root → TLD → authoritative) → cached.
- Major record types and which ones can/can't coexist (CNAME edge cases).
- DNSSEC chain of trust: parent DS → child DNSKEY → RRSIG over RRset.
- EDNS for >512-byte UDP responses; TC=1 triggers TCP fallback.
- AXFR/IXFR with TSIG for zone transfers.
- DoT/DoH/DoQ tradeoffs.
- Five attack classes and one mitigation each: cache poisoning (DNSSEC + 0x20), amplification (RRL + close open resolvers), tunneling (ML on streams), DGA (classifier), rebinding (filter private IPs).
- Authoritative vs. recursive are *different* systems; clarify in any design question.

---

## Sources

- [RFC 1034 / 1035 — DNS basics](https://www.rfc-editor.org/rfc/rfc1034.html)
- [RFC 6891 — EDNS(0)](https://www.rfc-editor.org/rfc/rfc6891.html)
- [RFC 4033/4034/4035 — DNSSEC](https://www.rfc-editor.org/rfc/rfc4033.html)
- [RFC 7858 — DNS over TLS](https://www.rfc-editor.org/rfc/rfc7858)
- [RFC 8484 — DNS over HTTPS](https://www.rfc-editor.org/rfc/rfc8484)
- [RFC 9250 — DNS over QUIC](https://www.rfc-editor.org/rfc/rfc9250)
- [Slack engineering — DNSSEC rollout](https://slack.engineering/what-happened-during-slacks-dnssec-rollout/)
- [Cloudflare — how we analyze 1M DNS queries/sec](https://blog.cloudflare.com/how-cloudflare-analyzes-1m-dns-queries-per-second/)
- [ICANN — DNSSEC explainer](https://www.icann.org/resources/pages/dnssec-what-is-it-why-important-2019-03-05-en)