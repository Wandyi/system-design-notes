# 17 · Performance Tuning and Sysctls

The settings every staff engineer has tuned at least once. Knowing the right knob (and what it does) separates "I've tuned a kernel" from "I read a blog."

## 17.1 The high-leverage sysctls

```
# TCP buffer auto-tuning ranges (min, default, max)
net.ipv4.tcp_rmem  = 4096 131072 6291456     # raise max for high-BDP
net.ipv4.tcp_wmem  = 4096 16384  4194304

# Global maximums (cap for SO_RCVBUF/SO_SNDBUF)
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.rmem_default = 262144
net.core.wmem_default = 262144

# Listen / accept queues
net.core.somaxconn = 4096                    # was 128 historically!
net.ipv4.tcp_max_syn_backlog = 4096

# Connection tracking
net.netfilter.nf_conntrack_max = 1048576     # bump for high-rate
net.netfilter.nf_conntrack_buckets = 262144  # default = max/4
net.netfilter.nf_conntrack_tcp_timeout_established = 86400  # 1d not 5d
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 60
net.netfilter.nf_conntrack_udp_timeout = 30
net.netfilter.nf_conntrack_udp_timeout_stream = 180

# TCP behavior
net.ipv4.tcp_syncookies = 1                  # on always
net.ipv4.tcp_tw_reuse = 1                    # safe in modern kernels
net.ipv4.tcp_fin_timeout = 30                # was 60
net.ipv4.tcp_keepalive_time = 600            # was 7200(!)
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_slow_start_after_idle = 0       # don't restart slow start
net.ipv4.tcp_no_metrics_save = 1             # don't cache per-route TCP info
net.ipv4.tcp_mtu_probing = 1                 # PMTUD blackhole detection
net.ipv4.tcp_fastopen = 3                    # client + server TFO

# Congestion control
net.ipv4.tcp_congestion_control = bbr        # or cubic (default)
net.core.default_qdisc = fq                  # required for BBR pacing

# Ephemeral ports
net.ipv4.ip_local_port_range = 1024 65535    # widen for high-fan-out

# IP forwarding (routers, K8s nodes)
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
net.ipv4.conf.all.rp_filter = 0              # 0 if multi-homed; 1 by default

# Per-CPU softirq budget
net.core.netdev_budget = 600
net.core.netdev_budget_usecs = 8000
net.core.netdev_max_backlog = 8192

# UDP
net.ipv4.udp_rmem_min = 8192
net.ipv4.udp_wmem_min = 8192

# IPv6
net.ipv6.bindv6only = 0

# ARP / neighbor table
net.ipv4.neigh.default.gc_thresh1 = 1024
net.ipv4.neigh.default.gc_thresh2 = 4096
net.ipv4.neigh.default.gc_thresh3 = 8192
```

Memorize the top half (TCP buffers, somaxconn, conntrack, keepalive). Bottom half = "I'd look up but here's the area."

## 17.2 The TCP buffer story (the most common production miss)

```
BDP (Bandwidth Delay Product) = bandwidth × RTT
```

For a 10Gbps × 100ms path: BDP = 125MB.

If `tcp_rmem.max` < BDP → TCP cannot fill the pipe regardless of congestion control.

Default `tcp_rmem.max` is 6MB. Adequate for 6MB × 8 = 48Mbps × 1s = 50Mbps × 1s. Fine for LAN; falls over on transcontinental.

Staff move: **size buffers to your worst-case BDP**. For a global CDN: max RTT ~250ms; 10Gbps × 250ms = 312MB. Set `tcp_rmem.max=256MB+`.

Two ways to scale:
- Bump system-wide (sysctl) → kernel autotunes per-socket.
- Set `SO_RCVBUF/SO_SNDBUF` per socket → **disables autotune**, fixed at value.

99% of the time you want option 1.

## 17.3 The listen backlog story

`somaxconn` caps the `backlog` argument of `listen()`. Default was 128 forever (way too small); 4096+ in modern distros.

Symptom of being too small:
- Under burst (deploy, traffic spike), accept queue overflows.
- Kernel drops the final ACK → client retransmits → SLA-impacting delays.
- Counter: `nstat -az TcpExtListenOverflows`.

Set in `/etc/sysctl.conf` AND the application's `listen(fd, backlog)` argument; both must be high.

## 17.4 The TIME_WAIT story

After close, the TCP connection sits in TIME_WAIT for 60s (`tcp_fin_timeout`, naming is unfortunate). This:
- Holds the 4-tuple.
- Burns an ephemeral port (for outgoing connections).
- Eats a small amount of memory per entry.

For a busy client (proxy, batch job, scraper): can pile up to millions of TIME_WAITs, exhausting ports.

Mitigations:
- `tcp_tw_reuse = 1` — safe; reuse for outgoing only when older than 1s.
- `tcp_tw_recycle` — **removed in 4.10**; broke NAT. Don't use.
- Keep connections alive (HTTP keepalive, pool); reduce churn.
- Multiple source IPs.

`ss -tan state time-wait | wc -l` is the count.

## 17.5 The conntrack table story

Default `nf_conntrack_max` ~256K. A high-rate proxy with 5000 new flows/sec at 5d default timeout = 2.16B potential entries → fills in minutes if timeouts not shortened.

Symptom: `dmesg | grep nf_conntrack` shows `table full, dropping packet`.

Mitigations:
- Raise `nf_conntrack_max`. Each entry ~300 bytes; 4M entries = 1.2GB RAM.
- Shorten `tcp_timeout_established` (default 432000s = 5d → set 86400s = 1d).
- Use NOTRACK or eBPF datapath (Cilium).

## 17.6 NIC queues, IRQ, NUMA

```
ethtool -L eth0 combined 16                  # 16 RX+TX queues (one per CPU)
ethtool -G eth0 rx 4096 tx 4096              # ring sizes (defaults often 256-512)
ethtool -C eth0 rx-usecs 30 rx-frames 32 adaptive-rx on   # coalescing
```

IRQ affinity: pin each queue's IRQ to a specific CPU.

```
# Find IRQ numbers
grep eth0 /proc/interrupts

# Pin IRQ 130 to CPU 0
echo 1 > /proc/irq/130/smp_affinity

# Or use Mellanox's set_irq_affinity.sh
mlnx_tune
```

NUMA: pin RX queues to CPUs on the NIC's NUMA node.

```
cat /sys/class/net/eth0/device/numa_node
```

If node 0: pin IRQs to CPUs 0-15 (assuming 16 cores per socket). Pin app threads similarly.

## 17.7 RPS / RFS — when single-CPU RX

If RSS hashes pathologically to one CPU (single elephant flow, or one client doing 80% of traffic):

```
# Enable RPS on RX queue 0, spread to CPUs 0-15
echo ffff > /sys/class/net/eth0/queues/rx-0/rps_cpus

# RFS uses sock-flow table for locality
echo 32768 > /proc/sys/net/core/rps_sock_flow_entries
echo 32768 > /sys/class/net/eth0/queues/rx-0/rps_flow_cnt
```

RPS adds latency per packet. Use only when RSS imbalance is real.

## 17.8 Kernel/application tuning checklist

For a high-throughput server:

1. **Buffers**: `tcp_rmem.max`, `tcp_wmem.max`, `rmem_max`, `wmem_max` sized to worst-case BDP.
2. **somaxconn**: 4096+; matched by app's `listen(fd, backlog)`.
3. **Conntrack**: `nf_conntrack_max` sized; `tcp_timeout_established` shortened.
4. **TIME_WAIT**: `tcp_tw_reuse=1`; widen ephemeral range.
5. **Keepalive**: `tcp_keepalive_time` ~600s (was 7200 forever).
6. **Slow start**: `tcp_slow_start_after_idle=0`.
7. **MTU probing**: `tcp_mtu_probing=1`.
8. **TFO**: `tcp_fastopen=3` if you control client+server.
9. **Congestion control**: BBR for high-RTT, Cubic default.
10. **Default qdisc**: `fq` if BBR; `fq_codel` otherwise.
11. **NIC queues**: max combined; ring sizes 4096.
12. **IRQ affinity**: pin to NUMA-local CPUs.
13. **NAPI budget**: 600.
14. **Offloads**: GSO/TSO/GRO on; LRO off; checksum offload on.
15. **For TLS**: kTLS on, NIC offload if available.
16. **For containers**: `ip_forward=1`; per-netns conntrack max.
17. **For routers**: `rp_filter=0` if multi-homed; large `gc_thresh*`.
18. **For 10Gbps+**: jumbo frames or BIG_TCP (IPv6); MTU > 1500.

## 17.9 The most common production miss: tcp_rmem.max

A high-RTT (cross-region) workload running on default `tcp_rmem.max=6MB`:
- Can fill ~48Mbps per connection.
- Sees "the link is 10Gbps but my transfer is 50Mbps."
- Engineer suspects everything except the buffer.

Set it. Memorize the BDP × RTT formula.

## 17.10 The second-most-common: somaxconn

Default 128 (now 4096 on most distros). Old container image with `nginx` listening at backlog=128 + traffic burst = drops.

Tune both kernel AND app.

## 17.11 Special: routers and forwarding

```
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1

# Disable RP filter for multi-homed (strict mode rejects asymmetric returns)
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0

# Don't accept source routing
net.ipv4.conf.all.accept_source_route = 0

# Don't accept redirects (security)
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0

# ARP table sizing for big neighbor counts (k8s nodes)
net.ipv4.neigh.default.gc_thresh1 = 32768
net.ipv4.neigh.default.gc_thresh2 = 65536
net.ipv4.neigh.default.gc_thresh3 = 131072

# Disable conntrack on pure router workloads
echo "blacklist nf_conntrack" >> /etc/modprobe.d/conntrack.conf
```

## 17.12 Special: BBR

```
sysctl net.ipv4.tcp_congestion_control=bbr
sysctl net.core.default_qdisc=fq
```

Per-socket: `setsockopt(IPPROTO_TCP, TCP_CONGESTION, "bbr")`.
Per-route: `ip route change ... congctl bbr`.

BBR needs `fq` (or `fq_codel` in modern kernels) to honor the pacing rate.

## 17.13 Cgroup-level limits

```
# Per-cgroup network priority
echo "eth0 5" > /sys/fs/cgroup/.../net_prio.ifpriomap

# Per-cgroup classid (for tc)
echo 0x10001 > /sys/fs/cgroup/.../net_cls.classid
tc filter add dev eth0 ... cgroup ...
```

In modern systems: eBPF cgroup-attach is more common than net_cls.

## 17.14 BIG_TCP (IPv6 only)

```
ip link set dev eth0 gso_max_size 196608 gro_max_size 196608
```

Raises GSO/GRO max past 64KB. ~30% gain at 100Gbps IPv6. Mainline 5.19+.

## 17.15 Common interview probes

- **"You're seeing 50Mbps on a 10Gbps cross-continent link."** → `tcp_rmem.max` < BDP. Compute BDP, set the sysctl, confirm with `ss -tin` showing larger send/receive windows.
- **"Backlog overflow on deploy."** → `somaxconn` and app's `listen` backlog both too small.
- **"Conntrack table full, what now?"** → Raise `nf_conntrack_max`, shorten timeouts, consider eBPF bypass for hot paths.
- **"What's slow_start_after_idle?"** → After ~RTO of idle, TCP returns to slow start (per RFC 5681). Bad for HTTP/1.1 keepalive with bursty traffic. Set to 0 for web workloads.
- **"Difference between SO_RCVBUF and tcp_rmem?"** → SO_RCVBUF is per-socket fixed; tcp_rmem is autotune bounds. Setting SO_RCVBUF disables autotune.
- **"How would you tune a 100Gbps server?"** → Walk through buffer sizes, queues, IRQ pinning, GRO/GSO/BIG_TCP, kTLS if TLS, BBR/Cubic, ring sizes.

## 17.16 Corner cases

- **Sysctl applies to root netns only by default.** Per-netns ones (most TCP, conntrack) apply to the current netns. Some are still global; check with `sysctl -e`.
- **`SO_RCVBUF` capped by `rmem_max`.** App sets 100MB but `rmem_max=16MB` → silently capped at 16MB.
- **`SO_RCVBUFFORCE`/`SO_SNDBUFFORCE`** — bypass the cap (CAP_NET_ADMIN required). Rare.
- **`tcp_mem` (different from tcp_rmem!).** Global per-protocol memory tracking. Defaults usually OK; only matters at huge connection counts.
- **`tcp_app_win`/`tcp_adv_win_scale`.** Fraction of rcvbuf reserved for app processing vs window advertised. Default usually fine.
- **`tcp_max_orphans`** — orphaned sockets (no fd, in FIN_WAIT). Cap; reduces resource use.
- **netdev_max_backlog**. Per-CPU queue for packets received but waiting for protocol stack (post-NAPI, pre-protocol). Bursty workloads on slow CPU may need bigger.

## 17.17 Performance benchmarking

`iperf3` is the staple for TCP throughput tests. `nuttcp` is another. `netperf` for many test types (TCP_RR for latency, TCP_STREAM for throughput, etc.).

Tools for hotspot perf:
- `mpstat -P ALL 1` — per-CPU utilization.
- `sar -n DEV 1` — per-interface throughput.
- `dstat -tn 1` — combined.
- `bmon`, `iftop`, `nethogs` — live per-process traffic.

## 17.18 Alternative approaches

| Need | Default | Better 1 | Better 2 |
|------|---------|----------|----------|
| Saturate 100Gbps | autotune | tuned tcp_rmem | BIG_TCP + kTLS sendfile |
| Reduce TIME_WAIT pressure | wait | `tcp_tw_reuse=1` | connection pool |
| Handle SYN flood | default tcp_syncookies | larger syn_backlog | XDP_DROP for bad SYNs |
| Reduce conntrack | raise max | shorten timeouts | eBPF bypass |
| Reduce IRQ load | default | coalescing | busy poll |
| Pacing | none | `fq` qdisc | EDT via SO_TXTIME |
| Cross-region throughput | Cubic | BBR | QUIC |

## Must-internalize

- BDP = bandwidth × RTT; if `tcp_rmem.max` < BDP, TCP caps. Most common production miss.
- `somaxconn` 4096+; match in app listen backlog.
- `tcp_tw_reuse=1` is safe and reduces TIME_WAIT pressure.
- `tcp_slow_start_after_idle=0` for web workloads.
- BBR + `fq` qdisc for high-RTT bulk transfer.
- Conntrack: `nf_conntrack_max` × `tcp_timeout_established` sized to your churn.
- IRQ affinity → NUMA-local CPUs; ring size 4096; combined queues = CPU count.
- Top miss debug: `ss -tin`, `nstat -az`, `ethtool -S` show the missing tuning.