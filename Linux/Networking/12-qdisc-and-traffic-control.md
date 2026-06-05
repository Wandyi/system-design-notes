# 12 · Queueing Disciplines (qdisc) and Traffic Control (tc)

The "fairness + shaping + scheduling" layer of the Linux TX path. Every packet leaving Linux passes through a qdisc. The default has changed over time: `pfifo_fast` → `fq_codel` (3.6+ many distros) → in some setups `fq` (BBR's preferred).

## 12.1 Where the qdisc lives

```
                 ┌─────────────────────┐
                 │ dev_queue_xmit()    │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │ qdisc_enqueue()     │  ◄── classify; bucket; mark
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │ qdisc dequeue       │  ◄── may delay (rate limit, schedule)
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │ ndo_start_xmit()    │  ── driver, NIC
                 └─────────────────────┘
```

Egress qdisc is per-netdev (per output interface). On a multi-queue NIC, there's actually one qdisc per TX queue (MQ — multiqueue qdisc — wraps them).

## 12.2 The qdisc taxonomy

**Classless qdiscs** (no internal classification):
- `pfifo` / `bfifo` — packet FIFO / byte FIFO. Simple.
- `pfifo_fast` — 3-band priority FIFO (matches TOS bits). Old default.
- `sfq` / `esfq` — Stochastic Fair Queue. Hash-based fairness.
- `fq_codel` — Fair Queue + Controlled Delay. Modern default.
- `codel` — just CoDel (no fair queueing).
- `fq` — Fair Queue with pacing support (used with BBR).
- `cake` — Common Applications Kept Enhanced; consumer-class shaping + AQM.
- `red` / `gred` — Random Early Detection / Generic RED.
- `tbf` — Token Bucket Filter. Rate limit, no fairness.
- `netem` — Network emulator. Add delay, loss, reorder for testing.

**Classful qdiscs** (have a hierarchy of classes):
- `htb` — Hierarchical Token Bucket. Per-class rate / ceiling / burst.
- `hfsc` — Hierarchical Fair Service Curves. More mathematical, less common.
- `prio` — N priority bands.
- `drr` — Deficit Round Robin.

## 12.3 fq_codel — the modern default

**fq_codel** is the result of years of bufferbloat research.

Mental model:
- **Stochastic Fair Queueing**: hash flows into 1024 buckets; each bucket gets round-robin service. Prevents one flow starving others (the bufferbloat fix).
- **CoDel** (Controlled Delay) per-bucket: track each packet's queueing time; if persistent excess delay → drop early (signal to TCP to back off).

Why it matters:
- Pre-fq_codel: a single large flow could fill the device queue with seconds of data (bufferbloat). Web latency exploded.
- Post-fq_codel: each flow gets its fair share of the queue; small flows have low latency even when bulk flows compete.

CoDel parameters (sane defaults):
- `target = 5ms` — desired queue delay.
- `interval = 100ms` — measurement window.

Drop happens early enough that TCP halves cwnd, freeing space.

## 12.4 fq — the BBR companion

**fq** = Fair Queue with **pacing**. Used by BBR (and modern Cubic optionally).

Pacing: instead of sending a cwnd burst then waiting, space packets evenly across the RTT. This prevents queue-bloat at intermediate routers.

```
Burst send:  ████████ ········· ████████ ··········
Paced send:  █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █
```

`fq` lets each socket request a pacing rate; the kernel schedules accordingly.

When to use fq:
- BBR congestion control (mandatory in older kernels; optional in newer).
- High-throughput servers where bursty TX hurts other flows in shared switch buffers.

```bash
tc qdisc replace dev eth0 root fq
```

## 12.5 CAKE — the consumer answer

**CAKE** is fq_codel + AQM + per-host fairness + DiffServ awareness. The "set and forget" qdisc for home routers + bufferbloat fix.

Knobs: bandwidth limit, encapsulation overhead, RTT.

In datacenter networking: not used. In CDN edges and ISP CPE: increasingly common.

## 12.6 HTB — hierarchical rate shaping

When you need: "Customer A gets 10Mbps, Customer B gets 5Mbps, both can burst to 20Mbps total."

```bash
tc qdisc add dev eth0 root handle 1: htb default 30
tc class add dev eth0 parent 1: classid 1:1 htb rate 100mbit
tc class add dev eth0 parent 1:1 classid 1:10 htb rate 10mbit ceil 20mbit  # customer A
tc class add dev eth0 parent 1:1 classid 1:20 htb rate  5mbit ceil 20mbit  # customer B
tc filter add dev eth0 parent 1: protocol ip prio 1 u32 match ip src 1.2.3.4 flowid 1:10
```

HTB is the workhorse for traffic shaping in ISP, CDN egress, and VM hosting.

Modern alternative: BPF-based shaping. EDT (Earliest Departure Time) using `SO_TXTIME` skips HTB complexity for pacing.

## 12.7 EDT and SO_TXTIME

A modern pacing model: each skb carries a "send no earlier than" timestamp. The qdisc just sorts by departure time.

`SO_TXTIME` on the socket; app sets `SCM_TXTIME` per send.

EDT lets userspace (or BPF) make the scheduling decision. Used by:
- QUIC pacers (avoid sending faster than estimated bottleneck).
- Custom rate limiters (avoid HTB's per-class overhead).
- Microbench precision (deterministic spacing).

`fq` is EDT-aware.

## 12.8 Token Bucket (tbf) — simple rate limit

```bash
tc qdisc add dev eth0 root tbf rate 1mbit burst 32k latency 400ms
```

Single-class, no fairness. Useful for the simplest "cap egress" need. For per-flow fairness, you need HTB or fq.

## 12.9 ingress qdisc / clsact

By default qdisc is egress. For ingress, attach a special qdisc.

```bash
tc qdisc add dev eth0 ingress
tc filter add dev eth0 parent ffff: u32 match ip src 1.2.3.4 police rate 10mbit
```

Or, for eBPF use:

```bash
tc qdisc add dev eth0 clsact
tc filter add dev eth0 ingress bpf da obj prog.o sec ingress
tc filter add dev eth0 egress  bpf da obj prog.o sec egress
```

clsact is a special qdisc that supports both ingress and egress for tc-bpf programs.

## 12.10 netem — failure injection

Add latency, loss, reorder, duplicate to simulate WAN conditions.

```bash
tc qdisc add dev eth0 root netem delay 100ms 10ms loss 1%
```

- `delay 100ms 10ms` — mean 100, ±10 jitter.
- `loss 1%` — drop 1% randomly.
- `loss random 1% 25%` — correlated loss.
- `corrupt 0.1%` — corrupt 1 in 1000.
- `duplicate 0.5%` — duplicate 1 in 200.
- `reorder 25% 50%` — reorder 25% of packets after delay X with correlation 50%.

Use for chaos engineering, integration tests.

## 12.11 Multi-queue NIC, MQ qdisc

Modern NICs have N TX queues. Linux exposes each as a separate qdisc.

```
                          eth0
                          ┌─────────────────┐
                          │ MQ qdisc        │  ◄── wrapper, not a real qdisc
                          ├─────────────────┤
                          │ tx0: fq_codel   │  ── used by CPU 0
                          │ tx1: fq_codel   │  ── used by CPU 1
                          │ ...             │
                          │ txN: fq_codel   │  ── used by CPU N
                          └─────────────────┘
```

XPS (Transmit Packet Steering) maps CPU → TX queue. So each CPU has its own qdisc; minimal cross-CPU contention.

`tc qdisc show dev eth0` lists each one.

## 12.12 Bufferbloat — the original sin

Pre-2010, network buffers were sized for throughput (huge!). A WAN link buffer of 1MB at 1Mbps = 8s of queueing.

Result: a single big flow filled the buffer; web latency went to seconds.

Fix: AQM (Active Queue Management) — fq_codel, CoDel, PIE. Drop early so TCP backs off.

Modern Linux has fq_codel default since 3.6 (most distros). The internet is much better for it.

## 12.13 Common interview probes

- **"What's the default qdisc on a modern Linux box?"** `fq_codel` for most distros; `mq` wrapper on multi-queue NIC; each leaf is `fq_codel`.
- **"How do you rate-limit egress?"** TBF for simple cap; HTB for hierarchical; EDT/`fq` for pacing; eBPF tc for custom.
- **"Why does BBR want `fq`?"** BBR computes a pacing rate; the qdisc must respect it. `fq` does; `fq_codel` mostly does (recent kernels). HTB doesn't necessarily.
- **"How do you debug a slow connection?"** `tc -s qdisc show dev eth0` shows drops/backlog; `ethtool -S` shows NIC drops; `ss -tin` shows TCP-level state.
- **"How does fq_codel achieve fairness?"** SFQ hashes flows into buckets; DRR (or similar) gives equal service among non-empty buckets; CoDel drops early per bucket.

## 12.14 Corner cases

- **Default qdisc not applied** — distro chose `pfifo_fast` (some embedded). Set via `sysctl net.core.default_qdisc=fq_codel`.
- **HTB borrow loops** — children borrow from parent up to ceiling; misconfig can let one customer steal all parent rate.
- **TBF burst too small** — bursty TCP can't fit a single send window; underutilizes bandwidth. Burst = rate × RTT typical.
- **netem with low delay** — netem itself has overhead; very low delays (sub-ms) noisy.
- **Conntrack + netem** — netem queues without timer adjustment; conntrack expects timestamps.
- **`tc filter` priority numbering** — lower wins; if you forget, rules apply in insertion order, surprising.
- **MQ inheritance** — changing `default_qdisc` doesn't retroactively change existing devices; bring interface down/up.
- **`ifb` (Intermediate Functional Block)** — fake interface to apply egress-style shaping to ingress (since ingress qdisc is limited).

## 12.15 Performance numbers

Default `fq_codel` on a tuned host:
- Adds <1µs latency at light load.
- Hash bucket count = 1024 (default); enough for 10K-flow workloads.

HTB at 10Gbps with 10 classes:
- 5-10% CPU overhead for classification + scheduling.

netem at 10Gbps:
- Significant CPU; not for production load.

## 12.16 Alternative approaches

| Need | Path 1 | Path 2 | Path 3 |
|------|--------|--------|--------|
| Reduce bufferbloat | fq_codel | CAKE | per-flow EDT |
| Per-flow fairness | fq_codel | SFQ | fq |
| Rate limit single flow | TBF | HTB single class | eBPF tc tail-drop |
| Hierarchical shaping | HTB | HFSC | eBPF tc with maps |
| Pacing for BBR | fq | fq_codel (since 5.x) | EDT-aware app |
| Failure injection | netem | iptables -j DROP --probability | eBPF tc drop with map |
| Ingress shaping | ingress qdisc police | ifb + egress shaping | XDP_DROP |

## 12.17 The qdisc choice in production

| Scenario | Qdisc |
|----------|-------|
| Web server (general) | fq_codel (default) |
| BBR-using server | fq |
| Multi-tenant rate shaping | HTB |
| Home router / CPE | CAKE |
| Custom rate limit | eBPF tc + maps |
| Latency-critical (HFT) | pfifo or noqueue |
| Test environments | netem |

## Must-internalize

- Every TX packet goes through a qdisc; default = `fq_codel` since ~3.6.
- `fq_codel` solves bufferbloat by SFQ-per-flow + CoDel early-drop.
- `fq` is BBR's preferred qdisc for pacing.
- `htb` for hierarchical rate shaping (per-customer caps with borrowing).
- `tbf` for simple single-rate cap.
- `netem` for testing — delay, loss, reorder.
- Multi-queue NIC: per-queue qdisc; XPS maps CPU → queue.
- `clsact` is the special qdisc for tc-bpf programs (ingress + egress).
- `tc -s qdisc show` is your friend (shows drops, backlog).