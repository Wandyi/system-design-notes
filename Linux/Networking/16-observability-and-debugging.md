# 16 · Observability and Debugging

The staff signal *most* graders look for. "How would you debug this" is asked in every interview; the depth of tool fluency separates senior from staff.

## 16.1 The toolbox by layer

```
Layer       Tool(s)                                  Reads
─────────────────────────────────────────────────────────────────────────
App         strace, ltrace, perf, gdb, pid logs      syscalls, user state
Socket      ss, netstat, lsof, /proc/net/*           socket states
TCP         ss -tin, tcpdump, /proc/net/snmp         TCP counters, retransmits
IP/route    ip, traceroute, mtr, /proc/net/route     routing, neighbors
Netfilter   iptables -nvL, nft list, conntrack -L    rule counters, flows
qdisc/tc    tc -s, tc -s class                       queue drops, backlog
Driver/NIC  ethtool, /proc/interrupts, /sys/class    HW stats, IRQs, queues
Kernel      perf, ftrace, bpftrace, trace-cmd        function-level events
Hardware    NIC vendor tools (mlxreg, mlnx_perf)     ASIC counters
Cross-cut   dropwatch, drop_monitor                  where packets are dropped
```

A staff engineer can name the *right tool first*, run it, interpret it, and act. Below is each one with the staff-level facts.

## 16.2 ss — the socket workhorse

`ss` replaces `netstat`. Reads via netlink (fast even with millions of sockets).

```
ss -tnH                    # TCP, numeric, no header
ss -tan state established  # ESTABLISHED only
ss -tin                    # TCP info: rtt, cwnd, retransmits, snd_wnd, rcv_wnd
ss -tlpn                   # listening sockets with pid
ss -tan state time-wait    # TIME_WAITs (often shows scale issues)
ss -tan state close-wait   # CLOSE_WAITs (app fd leak)
ss -K dport = 5432          # kill all sockets to port 5432 (yes, ss can do this)
ss -e -o                   # ext info: timer, internal queue, retrans count
```

The most-used staff command is `ss -tin`. Output decoded:

```
ESTAB  0 0 10.0.0.5:443 10.0.0.6:39812  cubic wscale:7,7 rto:204 rtt:0.5/0.25 ato:40 mss:1448 cwnd:10 ssthresh:7 bytes_acked:..  retrans:0/1 ...
```

- `cubic`: congestion alg.
- `wscale:7,7`: send and receive window scale.
- `rto:204` ms.
- `rtt:0.5/0.25` ms (smoothed/variance).
- `cwnd:10` (segments).
- `retrans:0/1` (current/total).

For staff, this is bread and butter.

## 16.3 tcpdump — the packet pcap

```
tcpdump -i any -nn -tttt -s 0 -w out.pcap port 443
tcpdump -i eth0 -nn 'tcp and (tcp-syn|tcp-fin) != 0'  # only SYN/FIN
tcpdump -nn 'ip[2:2] > 1500'                          # IP datagrams > 1500
tcpdump -i any -nnvvX 'host 1.2.3.4'                  # hex+ascii payload
```

Read in Wireshark; tcpdump has built-in parsers for common protos.

`-B 4096` bumps capture buffer for high-rate links. `-s 0` captures full packet (default truncates).

In a high-rate environment, **tcpdump drops packets**. Always check the "X packets dropped by kernel" footer. For high-rate captures, use AF_PACKET ring (`-W` rotation, `--immediate-mode` low-latency) or just `dumpcap`/`tshark`.

For 10Gbps+: don't tcpdump live. Mirror via switch SPAN to a separate box.

## 16.4 ip — the iproute2 polyglot

```
ip addr / ip a       # interfaces + addresses
ip link / ip l       # link state
ip route / ip r      # routing table
ip rule              # routing policy database
ip neigh / ip n      # ARP / ND cache
ip xfrm              # IPsec policy
ip netns             # network namespaces
ip mptcp             # MPTCP endpoints
ip vrf               # VRFs
ip -s link           # link stats (rx/tx packets, errors, drops)
ip -s -s link        # detailed errors (collisions, dropped, etc.)
```

The newer iproute2 covers nearly everything; `ifconfig` and `route` are deprecated.

## 16.5 conntrack — the NAT state inspector

```
conntrack -L                              # list all flows
conntrack -L -p tcp                       # tcp only
conntrack -L --orig-src 10.0.0.1          # source-filtered
conntrack -E                              # subscribe to events
conntrack -D --orig-dst 10.0.0.2           # delete entries
conntrack -S                              # stats: insert, search, drop, etc.
cat /proc/sys/net/netfilter/nf_conntrack_count  # current entries
cat /proc/sys/net/netfilter/nf_conntrack_max    # max
```

`conntrack -S` is the staff move: shows `insert_failed`, `drop`, `early_drop`, `search_restart`. If `insert_failed` > 0, table is full or contended.

## 16.6 ethtool — the NIC inspector

```
ethtool eth0                              # link speed, duplex
ethtool -i eth0                           # driver, fw version
ethtool -k eth0                           # offload features
ethtool -K eth0 gro off                   # toggle offloads
ethtool -S eth0                           # hardware stats (rx_dropped, rx_no_buffer_count, etc.)
ethtool -c eth0                           # interrupt coalescing
ethtool -C eth0 rx-usecs 30               # set coalescing
ethtool -l eth0                           # queue count
ethtool -L eth0 combined 16               # set 16 RX/TX combined queues
ethtool -g eth0                           # ring sizes
ethtool -G eth0 rx 4096 tx 4096           # set ring sizes
ethtool -x eth0                           # RSS hash table
ethtool -X eth0 equal 8                   # spread RSS over 8 queues
ethtool -n eth0                           # n-tuple flow rules
ethtool -t eth0                           # self-test
ethtool -p eth0 5                         # blink LED for 5s (identify cable!)
ethtool -m eth0                           # SFP/QSFP module info
ethtool -d eth0                           # register dump
ethtool --statistics eth0 | grep -i drop  # drop counters
ethtool --show-fec eth0                   # FEC settings
```

`ethtool -S eth0 | grep drop`: when the NIC reports drops, you have a hardware-level problem (PCIe bandwidth, ring buffer too small, RSS misconfiguration).

## 16.7 /proc/net — the kernel exposes itself

```
/proc/net/dev                  # per-interface stats
/proc/net/snmp                 # SNMP-style protocol counters (TcpInSegs, TcpRetransSegs, ...)
/proc/net/netstat              # extended counters (TcpExtSyncookiesRecv, TcpExtTW, ...)
/proc/net/sockstat             # global socket usage
/proc/net/tcp / tcp6           # TCP socket list (slow on big systems; use ss instead)
/proc/net/route                # routing table (older format than ip route)
/proc/net/arp                  # ARP cache (older than ip neigh)
/proc/net/nf_conntrack         # conntrack (slow scrape; use conntrack -L)
/proc/interrupts               # IRQ counts per CPU
/proc/softirqs                 # softirq counts per CPU
```

`nstat -az` parses /proc/net/snmp + netstat into one big counter dump. Diff over time = staff debug.

```
nstat -az | grep -i retrans
nstat -az TcpExt*
```

## 16.8 sysctl — the kernel's policy dials

`sysctl -a | grep -i tcp` shows everything. The notable ones live in `17-perf-tuning-and-sysctls.md`.

## 16.9 perf — the canonical profiler

```
perf top                                          # live profile
perf record -F 99 -a -g sleep 30; perf report     # 30s system-wide flamegraph
perf trace                                        # syscall trace (like strace but cheaper)
perf stat -e cycles,instructions,cache-misses ... # counters
perf sched record / report                        # scheduler analysis
perf trace -e tcp:tcp_retransmit_skb              # specific tracepoint
perf list                                         # all events
```

Used with `FlameGraph.pl` to make Brendan Gregg's flamegraphs.

When CPU is hot and you don't know why: `perf top -e cycles -p $PID`.

## 16.10 ftrace — the kernel function tracer

```
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo tcp_v4_rcv > /sys/kernel/debug/tracing/set_graph_function
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe
```

Or use `trace-cmd record -p function_graph -g tcp_v4_rcv` (much friendlier).

Use case: "where is this packet going inside the kernel" — function graph from `tcp_v4_rcv` shows the call chain.

## 16.11 bpftrace — modern, one-liner observability

The most powerful tool for ad-hoc kernel inspection. Examples:

```bash
# Retransmits per process
bpftrace -e 'tracepoint:tcp:tcp_retransmit_skb { @[comm] = count(); }'

# TCP connect latency
bpftrace -e '
  kprobe:tcp_v4_connect { @start[tid] = nsecs; }
  kretprobe:tcp_v4_connect /@start[tid]/ {
    @us = hist((nsecs - @start[tid])/1000);
    delete(@start[tid]);
  }'

# Per-flow TX byte counter
bpftrace -e 'kprobe:tcp_sendmsg { @[args->sk] = sum(args->size); }'

# Trace state changes for TCP
bpftrace -e 'tracepoint:sock:inet_sock_set_state { printf("%-16s %d %d->%d\n", comm, pid, args->oldstate, args->newstate); }'

# Drops
bpftrace -e 'tracepoint:skb:kfree_skb { @[kstack] = count(); }'
```

The bcc toolkit (Brendan Gregg) ships dozens of ready-made tools: `tcptracer`, `tcpconnect`, `tcpaccept`, `tcpdrop`, `tcpretrans`, `tcplife`, `tcpstates`. Memorize the names.

## 16.12 dropwatch / drop_monitor

The kernel emits a `kfree_skb` event with the location (function pointer) for every dropped packet. `dropwatch` (or eBPF tracing of `tracepoint:skb:kfree_skb`) shows *where* drops happen.

```
dropwatch -l kas
> start
```

You'll see entries like:

```
2 drops at tcp_v4_rcv+15a (0xffffffff8123)
5 drops at icmp_unreach+1b (...)
```

Translate the addresses (using `/proc/kallsyms`); `dropwatch -l kas` does this. This pinpoints which kernel function drops.

For modern kernels: `perf record -e skb:kfree_skb -a` or bpftrace.

## 16.13 mtr — the smart traceroute

`mtr 8.8.8.8` runs continuous traceroute + ping; shows packet loss per hop. Catches intermittent issues.

`mtr -T -P 443 host` does TCP-based probes (some firewalls drop ICMP/UDP traceroute).

## 16.14 nc / socat — the swiss army knife

```
nc -lvp 8080                  # listen, verbose
nc 1.2.3.4 80                 # raw TCP client
echo "GET /" | nc 1.2.3.4 80  # simple HTTP probe
socat - TCP:1.2.3.4:80        # equivalent
socat OPENSSL:host:443 -      # TLS-aware
socat UDP:1.2.3.4:53 -        # UDP
```

For TLS probing: `openssl s_client -connect host:443 -showcerts`.

## 16.15 pprof / gperftools / async-profiler

For application-level profiling. Go's `net/http/pprof` is the default; Java's async-profiler. Out of scope for kernel-networking but mention them when the bottleneck is userspace.

## 16.16 The staff debug recipe — "TCP is slow"

Standard incident response:

1. **Is it everywhere or one box?** Compare nodes. If one: that node is the suspect; isolate it.
2. **Is it all flows or some?** `ss -tin | sort -k...` for retransmits/cwnd.
3. **Are NIC drops up?** `ethtool -S eth0 | grep drop`. If yes: ring size, RSS distribution, PCIe.
4. **Are kernel drops up?** `nstat -az TcpExtListenDrops*`, `nstat -az TcpExtTCP*`. `dropwatch` to localize.
5. **Are softirqs hot?** `mpstat -P ALL 1`. If one CPU at 100% `%si`: RSS hash misconfig; enable RPS.
6. **Is conntrack full?** `conntrack -S`; `/proc/sys/net/netfilter/nf_conntrack_count`.
7. **Is qdisc dropping?** `tc -s qdisc show dev eth0`.
8. **Is TCP retransmitting?** `nstat -az TcpRetransSegs` vs `TcpOutSegs`. Retransmit ratio > 0.5% is suspicious.
9. **Is the path lossy?** `mtr -T -P 443 dst`.
10. **Is the app reading slowly?** `ss -tin` shows `Recv-Q` > 0 = backlog at receiver.

Each step has a clear next probe; the rhythm is what staff want to see.

## 16.17 The staff debug recipe — "Connection times out"

1. **Resolve DNS.** `dig host` (compare to `host`).
2. **Reach host.** `mtr host`; check ICMP allowed.
3. **TCP reachable.** `nc -zv host port`.
4. **TLS works?** `openssl s_client -connect host:port`.
5. **HTTP works?** `curl -v -m 5 https://host/`.
6. **Firewall in path?** `iptables -nvL`, `nft list ruleset`, conntrack counts.
7. **Backend overloaded?** `ss -lnt` for listen queue.

## 16.18 Common interview probes

- **"Walk through how you'd debug high tail latency."** Above recipe; emphasis on the *order*, not just naming the tools.
- **"What's `ss -tin` showing me?"** Per-socket TCP state including congestion alg, RTT, cwnd, retransmits.
- **"How do you find where packets get dropped?"** `dropwatch` / `bpftrace tracepoint:skb:kfree_skb` to localize the kernel function; `ethtool -S` for NIC-level; `/proc/net/snmp` for protocol-level.
- **"What does `nstat -az TcpExtTCPRcvCoalesce` mean?"** Number of times the receiver coalesced consecutive segments. Useful for understanding GRO effectiveness.
- **"How do you test failover without breaking prod?"** netem in a staging cell; tc to inject loss/delay; bpftrace to confirm path.

## 16.19 Logging vs metrics vs tracing

- **Logs** — text events. Slow, high cost. Best for rare incidents.
- **Metrics** — numeric time-series. Cheap. Best for trends.
- **Traces** — per-request spans across services. Mid-cost. Best for slow-request investigation.
- **Profiles** — sampled stack traces. Best for "why is X hot."

Staff designs should specify which of the four for each signal.

## 16.20 Production observability stacks

- **Prometheus + Grafana** for metrics.
- **OpenTelemetry** for traces, increasingly metrics.
- **Jaeger / Tempo** for trace storage.
- **Loki / ELK** for logs.
- **Pixie / Hubble (Cilium) / Pyroscope** — eBPF-native auto-instrumentation.
- **Datadog / New Relic / Splunk** — commercial unified.

For Linux networking specifically:
- **node_exporter** for sysctls, /proc stats.
- **cAdvisor** for container.
- **netflow / sflow / IPFIX** for flow records at switches.

## Must-internalize

- `ss -tin` is the per-socket TCP state inspector — memorize the fields.
- `tcpdump` + Wireshark is mandatory; tcpdump drops at high rates (use SPAN port for >10G).
- `ethtool -S` for NIC-level drops; `nstat -az` for protocol-level.
- `dropwatch` / bpftrace `kfree_skb` for "where is the drop."
- `conntrack -S` shows insert_failed counters — diagnostic for table overflow.
- bpftrace is the modern way to inspect anything in the kernel.
- Staff debug recipe: layered (link → IP → TCP → app) with a clear order and tools per layer.