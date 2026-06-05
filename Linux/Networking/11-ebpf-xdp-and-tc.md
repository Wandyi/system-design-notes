# 11 · eBPF, XDP, and tc-bpf

eBPF is the single biggest change in Linux kernel networking since NAPI. Every modern data-plane technology — Cilium, Katran, Cloudflare's L4 LB, Facebook's DDoS protection, much of Netflix's observability — is eBPF. Staff candidates *must* be conversant.

## 11.1 What eBPF is, in one paragraph

**eBPF** is a kernel-mode VM with a verifier. You write a small C program (`*.bpf.c`), compile to BPF bytecode with clang, load via `bpf(2)` syscall. The verifier statically proves the program (a) terminates, (b) doesn't read uninitialized memory, (c) doesn't access out-of-bounds pointers. If accepted, it's JIT-compiled to native code and attached to one of many hook points: socket filter, tc, XDP, kprobe, tracepoint, cgroup, sched, etc. eBPF programs can read/write to **maps** (kernel-side data structures: hash, array, lru_hash, ringbuf) shared with userspace. This lets you build custom dataplane/observability without forking the kernel.

## 11.2 The hooks (networking-relevant)

| Hook | Where | Use case |
|------|-------|----------|
| **socket filter** (`SO_ATTACH_BPF`) | per-socket on recv | DROP/PASS, simple per-socket filter |
| **socket lookup** (`BPF_PROG_TYPE_SK_LOOKUP`) | listen-time | Pick which listen socket gets the SYN (custom REUSEPORT) |
| **sockmap / sockhash** | redirect data | Sidecar bypass — Cilium redirects to local socket |
| **cgroup/skb** | per-cgroup ingress/egress | Cgroup firewall |
| **cgroup/sock_ops** | TCP state events | Per-conn telemetry, redirect data |
| **tc-bpf** (clsact ingress/egress) | qdisc level | Mid-stack filter/redirect; replaces iptables |
| **XDP** | pre-skb, post-IRQ | Line-rate drop/redirect/tx |
| **kprobe / tracepoint** | any kernel function | Observability (bpftrace, bcc) |

## 11.3 XDP (eXpress Data Path) — the headline

XDP runs an eBPF program *before* skb allocation, on a raw page+offset. Verdicts:

- `XDP_DROP` — discard immediately (~30Mpps per CPU; line-rate on 10G).
- `XDP_PASS` — continue up the stack.
- `XDP_TX` — bounce out the same interface (used by L4 LBs for DSR).
- `XDP_REDIRECT` — send to another iface or to an AF_XDP socket.
- `XDP_ABORTED` — error (counted).

Three modes:
- **Native** — driver hooks XDP directly into RX. Default for supported drivers (Mellanox, Intel, Broadcom).
- **Offload** — NIC firmware runs the BPF (Netronome agilio). Rare.
- **Generic** (`xdpgeneric`) — fallback after skb allocation. Slower but works on any driver. Useful for testing.

XDP is the kernel's answer to "I want DPDK perf without leaving the kernel."

## 11.4 XDP + AF_XDP

XDP can redirect packets into an `AF_XDP` socket — a userspace ring directly attached to the NIC RX queue.

```
NIC RX ──► XDP program ──► XDP_REDIRECT to AF_XDP queue ──► userspace mmap'd ring
```

This is **userspace-zero-copy networking with kernel-managed memory**. The userspace program reads packets via the ring without `recvmsg()`.

Used in:
- Suricata IDS at line rate.
- Cilium for fast forward.
- Custom HFT / telco stacks.

## 11.5 tc-bpf (clsact)

`tc qdisc add dev eth0 clsact` adds a special qdisc that supports BPF filters at ingress and egress:

```bash
tc filter add dev eth0 ingress bpf da obj prog.o sec ingress
tc filter add dev eth0 egress  bpf da obj prog.o sec egress
```

tc-bpf operates on full skbs (post-skb allocation). Slower than XDP but:
- Can modify skbs (header rewrite, push/pop encapsulation).
- Has full netfilter / conntrack metadata available.
- Works on both ingress and egress (XDP is only ingress).

Cilium uses tc-bpf for most pod-to-pod traffic.

## 11.6 eBPF maps

Shared kernel-userspace data structures.

| Type | Use |
|------|-----|
| `BPF_MAP_TYPE_HASH` | Generic key→value |
| `BPF_MAP_TYPE_LRU_HASH` | Auto-evict LRU; bounded |
| `BPF_MAP_TYPE_ARRAY` | Indexed by uint |
| `BPF_MAP_TYPE_PERCPU_ARRAY` | One copy per CPU; lock-free updates |
| `BPF_MAP_TYPE_LPM_TRIE` | IP prefix → value |
| `BPF_MAP_TYPE_RINGBUF` | Userspace consumer ring; modern, faster than perf buf |
| `BPF_MAP_TYPE_PERF_EVENT_ARRAY` | Per-CPU perf event ringbuf |
| `BPF_MAP_TYPE_SOCKMAP` / `SOCKHASH` | Socket fd → key; for sockmap redirect |
| `BPF_MAP_TYPE_DEVMAP` | ifindex map for XDP_REDIRECT |
| `BPF_MAP_TYPE_CPUMAP` | CPU number → can XDP_REDIRECT for re-balance |
| `BPF_MAP_TYPE_XSKMAP` | AF_XDP socket map |
| `BPF_MAP_TYPE_PROG_ARRAY` | Tail-call to other BPF program |

PERCPU maps are critical for per-packet counters — no atomic on hot path.

## 11.7 The verifier

The reason eBPF is safe enough to run in the kernel.

Constraints:
- Program size: up to 1M instructions (pre-5.2 was 4K).
- Loop depth: unbounded loops not allowed; **bounded loops** (5.3+) up to verifier limit.
- All branches must terminate; can't ascend out of a function.
- All memory accesses must be provably in-bounds.
- Pointer arithmetic must be checked.
- Helper function calls limited (no recursion).

Common pain: programs that *should* work but verifier rejects. The error message names the failed branch and the path it took. Debugging eBPF is "reading the verifier log."

## 11.8 BTF — type info for BPF

**BTF** (BPF Type Format) is debugging-info-equivalent for BPF programs and kernel types. Enables:
- CO-RE (Compile Once, Run Everywhere): one BPF program runs on many kernel versions via runtime relocations.
- `bpftool gen skeleton` — generate boilerplate.
- bpftrace can introspect kernel types.

Modern (5.x+) eBPF assumes BTF.

## 11.9 Famous eBPF/XDP production stories

### Facebook Katran (open source 2018)
L4 LB. Replaced IPVS with XDP. Handles ~millions of qps per box. Uses XDP_TX for **DSR** (Direct Server Return — LB sees only SYN/ACK; server replies to client directly).

### Cloudflare DDoS
XDP_DROP for known-bad patterns at line rate. Mitigates Mpps attacks without burning CPU.

### Cloudflare L4Drop
XDP filters before sk_buff allocation. Defended against 71M req/sec attacks.

### Cilium (the CNI)
Replaced kube-proxy. eBPF programs at tc-ingress/egress, socket, XDP. Maglev consistent hashing in eBPF. Hubble for observability.

### Netflix's BCC tools
Brendan Gregg's tooling. Tools like `tcpconnect`, `tcpaccept`, `tcpretrans`, `tcpdrop` — single-line bpftrace recipes for production debugging.

### Pixie
Auto-instrumented k8s observability. eBPF traces every HTTP/gRPC.

### Andromeda (Google Cloud)
Custom dataplane using eBPF + custom kernel. The hyperscaler "we built our own" example.

### Sentry / OpenTelemetry
APM via uprobes (kernel attaches BPF at userspace function entry).

## 11.10 Common interview probes

- **"What's the difference between XDP and tc-bpf?"** XDP is pre-skb (faster, more restricted); tc-bpf is post-skb (more features, slower). XDP only ingress; tc-bpf both.
- **"Why is eBPF safer than a kernel module?"** Verifier proves termination + memory safety statically; no escape; can't oops the kernel.
- **"How do you debug an eBPF program?"** `bpftool prog show`, `bpftrace -lv` for tracepoints, `bpf_printk()` → `/sys/kernel/debug/tracing/trace_pipe`, perf events for completion.
- **"Maglev hashing in eBPF?"** Cilium implements Maglev: build a lookup table with prime-spaced offsets per backend; lookup by hash; consistent under backend churn.
- **"Could XDP replace the kernel stack?"** For pure-pass-through use cases (LB, filter, encap), yes. For TCP termination, no — TCP state machine isn't there. AF_XDP + userspace TCP (mtcp, F-Stack) is the workaround.

## 11.11 XDP for L4 load balancer — sketch

A typical XDP L4 LB program:

```c
SEC("xdp")
int lb(struct xdp_md *ctx) {
    // 1. parse ethernet, IP, TCP/UDP
    // 2. extract 5-tuple
    // 3. consistent-hash to a backend
    // 4. encapsulate (IPIP or GUE) toward backend
    // 5. XDP_TX (bounce out same iface)
}
```

The backend sees encapsulated traffic, terminates, and replies *directly* to the client (DSR). LB is on the request path only — much more scalable.

This is the Katran/Maglev pattern.

## 11.12 Cilium architecture sketch

```
                            cilium-agent (userspace)
                                  │
                                  ▼
                  loads/manages BPF programs and maps
                                  │
              ┌───────────────────┼────────────────────┐
              ▼                   ▼                    ▼
       cgroup/sock_ops       tc-bpf (veth)            XDP
       (socket-level)        (per-pod ingress         (host eth0 LB,
                              + egress)                DDoS, NAT)
              │                   │                    │
              ▼                   ▼                    ▼
             eBPF maps: services, endpoints, policy, conntrack
```

The data plane is entirely eBPF — iptables is bypassed for service handling.

## 11.13 bpftrace — the one-liners

```bash
# Trace tcp_retransmit_skb calls
bpftrace -e 'tracepoint:tcp:tcp_retransmit_skb { @[comm] = count(); }'

# Count syscall errors per process
bpftrace -e 'tracepoint:syscalls:sys_exit_* /args->ret < 0/ { @[comm] = count(); }'

# Show TCP state changes
bpftrace -e 'tracepoint:sock:inet_sock_set_state { printf("%s %d -> %d\n", comm, args->oldstate, args->newstate); }'

# Latency histogram for tcp_v4_connect
bpftrace -e 'kprobe:tcp_v4_connect { @start[tid] = nsecs; } kretprobe:tcp_v4_connect /@start[tid]/ { @us = hist((nsecs - @start[tid])/1000); delete(@start[tid]); }'
```

Memorize a few; they're staff-level signals on tool fluency.

## 11.14 Corner cases

- **Verifier rejection on what *should* work.** Often pointer arithmetic — add explicit bounds checks.
- **Map churn.** `BPF_MAP_TYPE_HASH` updates aren't free; high-rate updates contend on internal locks. Use PERCPU or LRU variants.
- **`__sync_fetch_and_add` on PERCPU maps** — wrong; PERCPU is implicitly atomic per-CPU, but aggregating requires a userspace loop.
- **XDP on bonded interface.** Modern kernels (5.x+) support; older required workaround.
- **AF_XDP frame size**. Must be page-aligned and pre-registered. Misuse → EINVAL.
- **eBPF on tc-bpf with conntrack mark.** Conntrack runs at netfilter hooks; tc-bpf runs separately. Crossing them takes care (set mark in tc-bpf, conntrack zone propagates).
- **XDP doesn't see VLAN-tagged traffic** by default — need to handle 802.1q in your program.
- **Stack-trace fidelity.** `bpf_get_stackid` requires CONFIG_BPF_KPROBE_OVERRIDE for some setups.

## 11.15 Performance — where eBPF shines

| Task | Iptables | nftables | eBPF/tc | XDP |
|------|---------|---------|---------|-----|
| Drop 1B blacklisted IPs | unworkable | OK with set | OK with map | line-rate |
| L4 LB at 10Mpps | needs IPVS | similar | possible | yes (Katran) |
| Per-conn telemetry | not really | limited | yes (sock_ops) | n/a |
| DDoS filter | rule scan kills CPU | better | very good | best |

## 11.16 Alternative approaches

| Need | Path 1 | Path 2 | Path 3 |
|------|--------|--------|--------|
| Custom packet drop | iptables -j DROP | XDP_DROP | DPDK |
| L4 LB | IPVS | Katran (XDP) | DPDK (VPP) |
| Service mesh sidecar | Envoy with iptables redirect | Cilium sockops bypass | DPDK userspace mesh |
| Trace fn latency | ftrace | bpftrace | uprobe + custom |
| Pod-pod conntrack | kernel conntrack | Cilium eBPF flow table | OVS flows |
| Encap (VXLAN) | kernel VXLAN device | tc-bpf push/pop | XDP encap |

## 11.17 Build-vs-OSS — the staff call

Three tiers of eBPF adoption:

1. **Off-the-shelf** — Cilium, Falco, Pixie, Hubble. Mature, large community.
2. **Library** — libbpf, libxdp, cilium/ebpf (Go), redbpf (Rust). Build on top.
3. **From-scratch** — clang + bpf(2) + libbpf for hot-path. Only when off-the-shelf doesn't fit.

Staff candidates should reach for tier 1 first, tier 2 with justification, tier 3 with strong reason.

## Must-internalize

- eBPF = verified, JITed bytecode runs in kernel; programs attached to hooks, share state via maps.
- XDP = pre-skb, fastest (10–30 Mpps); tc-bpf = post-skb, more features.
- Verifier statically proves safety; "verifier said no" is the most common failure mode.
- AF_XDP = userspace zero-copy via XDP redirect; the modern kernel-bypass story.
- Cilium replaces kube-proxy; Katran replaces IPVS L4 LB; bpftrace replaces ad-hoc tracing.
- BTF + CO-RE = run one program across kernel versions.
- For staff interview: name 3 production stories (Cilium, Katran, Cloudflare DDoS).