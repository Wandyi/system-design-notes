# 09 · Netfilter, iptables, nftables, and conntrack

The packet-inspection / mangling layer. Every Linux firewall, every Docker NAT rule, every kube-proxy iptables service is here.

## 9.1 Netfilter — the framework

**Netfilter** is the in-kernel hooking infrastructure (since 2.4, 1999). It defines five hook points (per protocol family: ipv4, ipv6, bridge, arp).

```
                              ┌─────────────────┐
              ┌─────────────► │  routing        │
              │               │  decision       │
              │               └───────┬────┬────┘
   incoming   │                       │    │
   ──► PREROUTING        ┌────────────┘    └──► FORWARD ──► POSTROUTING ──►
       │                 │                       │
       │                 ▼                       │
       │             INPUT                       │
       │                 │                       │
       │             local                       │
       │             process                     │
       │                 │                       │
       │                 ▼                       │
       │             OUTPUT ────► routing ───────┘
       │                          decision
       └─ from process: OUTPUT ─►
```

Each hook can register multiple handlers, executed in priority order. Handlers return verdict: `ACCEPT`, `DROP`, `STOLEN`, `QUEUE`, `REPEAT`, `STOP`.

This is the substrate; `iptables`, `nftables`, `conntrack`, `ipset`, `ip6tables`, `ebtables` all hook here.

## 9.2 The five hook points

| Hook | Triggered |
|------|-----------|
| `NF_INET_PRE_ROUTING` | Just after entering, before routing |
| `NF_INET_LOCAL_IN` | After routing, destined for local process |
| `NF_INET_FORWARD` | After routing, will be forwarded |
| `NF_INET_LOCAL_OUT` | Right after local process sends |
| `NF_INET_POST_ROUTING` | Just before leaving the host |

Memorize: this is on every staff interview.

## 9.3 iptables — chain + table organization

iptables organizes hooks into **tables** × **chains**.

Tables (in priority order at each hook):
- `raw` — pre-conntrack; `notrack` lives here.
- `mangle` — packet mark/TOS modification.
- `nat` — NAT only, per-conntrack-entry (first packet of flow).
- `filter` — accept/drop (the firewall proper).
- `security` — SELinux markings.

Chains map to hooks:

| Chain | Hook |
|-------|------|
| PREROUTING | `NF_INET_PRE_ROUTING` |
| INPUT | `NF_INET_LOCAL_IN` |
| FORWARD | `NF_INET_FORWARD` |
| OUTPUT | `NF_INET_LOCAL_OUT` |
| POSTROUTING | `NF_INET_POST_ROUTING` |

Not every table has every chain. e.g., `nat` doesn't have INPUT (historically).

Common patterns:

- **MASQUERADE** (POSTROUTING in nat) — outbound NAT to interface's IP, dynamic. Docker uses this.
- **DNAT** (PREROUTING in nat) — redirect incoming to internal IP. Service VIPs use this.
- **SNAT** (POSTROUTING in nat) — outbound NAT to fixed IP.
- **REDIRECT** (PREROUTING in nat) — redirect to local port. Transparent proxies.

## 9.4 iptables' scaling problem

iptables compiles every rule into a **linear list**. Each packet matches by walking O(n) rules per hook.

Kubernetes with 1000 services × 10 endpoints = 10K-20K iptables rules in the nat table. Per-packet cost: linear scan. Reload time: 5–60s (the *whole* table must be flushed-and-rebuilt for any change).

For a busy node: kube-proxy iptables reload during a deployment becomes a bottleneck (services briefly unreachable during reload of 50K-rule sets).

This is why we have IPVS mode and eBPF (Cilium) — they have O(1) or O(log n) lookup.

## 9.5 nftables — the modern replacement

`nft` (since 3.13, 2014; deprecating iptables in distros since 2019 — RHEL 8, Debian 11 default).

Key differences:

| Property | iptables | nftables |
|----------|----------|----------|
| Engine | Per-table linear scan | Bytecode VM (single classification pass) |
| Syntax | Many tools (iptables, ip6tables, ebtables) | One tool (`nft`) |
| Atomicity | Per-rule (or `iptables-restore` whole-table) | Native atomic transactions |
| Maps/sets | `ipset` external | First-class sets and maps |
| IPv4/IPv6 | Separate | Unified (`inet` family) |
| Performance | O(N) per chain | Much better, set lookup is O(log N) or O(1) |
| Counters | Always-on | Optional (faster when off) |

Example: deny a set of IPs in nftables (O(1)):
```
table inet fw {
  set blackhole {
    type ipv4_addr
    flags interval
    elements = { 10.0.0.0/24, 192.168.1.0/24 }
  }
  chain input {
    type filter hook input priority 0
    ip saddr @blackhole drop
  }
}
```

iptables equivalent: 100 separate rules, all scanned per packet.

## 9.6 conntrack — connection tracking

The kernel's stateful firewall + NAT brain. Tracks the state of every flow.

State machine for TCP:
```
NEW ──► SYN_SENT ──► SYN_RECV ──► ESTABLISHED ──► FIN_WAIT/CLOSE_WAIT ──► TIME_WAIT ──► (expire)
```

Plus generic states for UDP (just `NEW`/`ASSURED`), ICMP, GRE, SCTP.

Conntrack entries:
- 5-tuple (protocol, src ip, src port, dst ip, dst port).
- Inverse 5-tuple (for return direction).
- State, timeout.
- NAT info (original vs translated).
- Mark, zone (for multi-tenant separation).

Used by:
- `iptables -m conntrack --ctstate ESTABLISHED,RELATED` (stateful firewall).
- NAT (MASQUERADE/DNAT/SNAT all rely on conntrack).
- Anything that uses `ct status` matching.

## 9.7 conntrack at scale

- Table size: `net.netfilter.nf_conntrack_max` (default ~256K for small machines, higher for big).
- Hash buckets: `net.netfilter.nf_conntrack_buckets` (default `max/4`).
- Timeout sysctls:
  - `nf_conntrack_tcp_timeout_established` — default 5 days (!)
  - `nf_conntrack_tcp_timeout_close_wait` — 60s
  - `nf_conntrack_tcp_timeout_time_wait` — 120s
  - `nf_conntrack_udp_timeout` — 30s
  - `nf_conntrack_udp_timeout_stream` — 120s

A misconfigured kube node with 256K conntrack_max and high TCP churn fills the table in minutes. Symptoms: `nf_conntrack: table full, dropping packet`, conn reset, NAT misbehavior.

Mitigations:
1. **Raise** `nf_conntrack_max` (RAM permitting; each entry ~300 bytes).
2. **Shorten** timeouts (especially `tcp_timeout_established` to ~1 day).
3. **NOTRACK** for high-volume known-safe paths: `iptables -t raw -A PREROUTING -p tcp --dport 80 -j NOTRACK` (skips conntrack for this traffic).
4. **Use eBPF datapath** (Cilium) that maintains its own flow table.
5. **Hard split** conntrack zones (`-j CT --zone 1`) for multi-tenant.

## 9.8 NAT — types and quirks

**DNAT** (PREROUTING):
- Rewrites destination. Used for service VIPs, port forwarding.
- Conntrack stores original tuple → translated tuple.

**SNAT** (POSTROUTING):
- Rewrites source. Used to make multi-host private network look like one IP.

**MASQUERADE**:
- SNAT to outgoing interface's IP, dynamic. Used by Docker/k8s for pod-to-internet.
- Drops state on link down (vs SNAT which keeps state).

**FULL CONE NAT (a.k.a. 1:1 NAT)**:
- Linux doesn't natively do "preserve outbound port" — it's symmetric NAT.
- For real full-cone (gaming, VoIP), need conntrack helpers (`nf_conntrack_h323`, etc.) or app-layer SIP ALG.

**STUN / TURN / ICE**:
- Application-layer protocols to discover NAT type and traverse. WebRTC's bread and butter.

**Port exhaustion**:
- A NAT box has ~28K SNAT ports per (src-ip, dst-ip, dst-port). At high outbound rates this is the limit.
- Mitigations: more SNAT IPs, hairpin to multiple pool IPs, longer connection reuse.

## 9.9 Service VIPs and kube-proxy

Kubernetes Service VIP (10.96.0.x typical) is implemented in PREROUTING DNAT:

```
iptables -t nat -A PREROUTING -d 10.96.0.10 -p tcp --dport 80 -j KUBE-SERVICES
iptables -t nat -A KUBE-SERVICES -d 10.96.0.10 -p tcp --dport 80 -j KUBE-SVC-XXX
iptables -t nat -A KUBE-SVC-XXX -m statistic --mode random --probability 0.50 -j KUBE-SEP-A
iptables -t nat -A KUBE-SVC-XXX -j KUBE-SEP-B
iptables -t nat -A KUBE-SEP-A -j DNAT --to-destination 10.244.1.5:80
iptables -t nat -A KUBE-SEP-B -j DNAT --to-destination 10.244.1.6:80
```

Notice: probability-based — coin flip per packet. Not consistent hashing. Not weighted.

IPVS replaces this with a real LB (RR, WRR, LC, WLC, SH, DH, LBLC, LBLCR, SED, NQ). Cilium uses eBPF maps with consistent hashing (Maglev).

## 9.10 ipset — kernel hash set

Used by iptables to match against thousands of IPs in O(1). `nft` has built-in sets, but `ipset` is still common where iptables rules exist.

```
ipset create blockips hash:ip
ipset add blockips 1.2.3.4
iptables -A INPUT -m set --match-set blockips src -j DROP
```

100K IPs in a hash:ip set: ~10MB RAM, O(1) matches.

## 9.11 TPROXY — transparent proxy

Allows a proxy to intercept traffic without DNAT (so source IP is preserved end-to-end).

```
iptables -t mangle -A PREROUTING -p tcp --dport 80 \
  -j TPROXY --tproxy-mark 0x1/0x1 --on-port 8080
ip rule add fwmark 0x1 lookup 100
ip route add local 0.0.0.0/0 dev lo table 100
```

Plus `IP_TRANSPARENT` on the proxy socket. Used by Squid, sslh, Envoy in some modes.

## 9.12 Common interview probes

- **"Trace how a packet reaches your nginx container."** PREROUTING (DNAT to pod IP via service rule) → FORWARD (host forwards) → POSTROUTING (no SNAT for cluster traffic) → veth → container → INPUT.
- **"What is conntrack's role in NAT?"** It stores the (orig 5-tuple, translated 5-tuple) so the *first* packet sets up state, subsequent packets are matched and translated automatically.
- **"Why is iptables slow at scale?"** O(n) per-chain rule scan; flushes are non-atomic per rule on legacy `iptables`; modern `iptables-restore --atomic` exists but the linear scan remains.
- **"How would you migrate from iptables to nftables?"** Tools: `iptables-translate`, `iptables-nft`. Run both for a while; gradually move table by table.
- **"Conntrack table full — what do you do?"** Read `dmesg`; check `nf_conntrack_count`; raise `nf_conntrack_max`; shorten timeouts; consider NOTRACK or eBPF bypass.

## 9.13 Corner cases

- **Conntrack races with DNAT under burst.** Two packets of a new flow race to create state; one ends up with original tuple, one translated. Mitigation: avoid `iptables-restore` during burst (or use IPVS).
- **`/proc/sys/net/netfilter/nf_conntrack_max` change doesn't apply.** It's per-netns since 4.10+; you must set inside the relevant netns.
- **DNAT loop.** A packet DNATed back to source. Solution: `-d <src>` skip rule.
- **MASQUERADE on slow-changing IP.** Conntrack state has the old src IP; new connections OK, in-flight broken.
- **NAT64 / NPTv6.** IPv4↔IPv6 NAT. `iptables` doesn't do; `nft` and `Jool` do.
- **iptables-legacy vs iptables-nft on RHEL/Ubuntu.** Default may be `iptables-nft` (wraps nftables); rule syntax same but kernel backend differs.

## 9.14 Performance numbers

- iptables: ~3-5M pps per CPU at small rulesets; degrades to ~500Kpps at 10K rules.
- nftables: ~5-10M pps; degrades far less with rule count.
- IPVS: ~10M pps (kernel-level connection table).
- eBPF (Cilium): ~25-50M pps (compiled bytecode, BPF maps).
- XDP_DROP: ~30Mpps per CPU (pre-skb).

These rough numbers help size capacity.

## 9.15 Alternative approaches

| Need | Path 1 | Path 2 | Path 3 |
|------|--------|--------|--------|
| Stateful firewall | iptables filter | nftables | eBPF (Cilium NetworkPolicy) |
| Outbound NAT | iptables MASQUERADE | nftables masquerade | Cilium eBPF SNAT |
| Service VIP | iptables DNAT (kube-proxy) | IPVS | Cilium eBPF map |
| Block large IP set | iptables + ipset | nftables set | XDP_DROP + map |
| Track state | conntrack | conntrack zones | eBPF flow map |
| Pass through high-vol path | NOTRACK | eBPF bypass | XDP pass |
| Transparent proxy | TPROXY | tc redirect | eBPF + ULP |

## Must-internalize

- 5 hooks: PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING.
- Tables order: raw → mangle → nat → filter → security.
- iptables = O(n) linear chain scan; reload non-atomic.
- nftables = bytecode VM, sets/maps native, atomic transactions.
- conntrack = stateful flow store; powers all NAT and stateful rules; sized by `nf_conntrack_max`.
- TCP-ESTABLISHED conntrack default timeout = 5 days. Tune to 1 day or less for high-churn.
- Kubernetes service VIPs are PREROUTING DNAT with probability matching (random, not weighted).
- IPVS / eBPF are the scale fixes for iptables.