# 07 · Zero-Copy and Offloads

The reason a modern Linux box can serve 100Gbps is **not** because the CPU got faster. It's because copies were eliminated and the NIC took over more of the work.

## 7.1 Where the copies happen

A naive `read(file_fd, buf, n)` + `send(socket_fd, buf, n)` does this:

```
disk ─► page cache ─► userspace buf ─► kernel skb ─► NIC DMA buffer
       (copy 1)        (copy 2)       (copy 3?)
```

Each copy is a CPU memcpy that consumes ~3GB/s of memory bandwidth and pollutes caches. At 100Gbps that's 12.5GB/s — three copies = 37.5GB/s of CPU memory traffic, which exceeds most CPU's bandwidth.

Zero-copy mechanisms eliminate one or more.

## 7.2 sendfile() — file to socket, no userspace

```c
ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
```

Kernel pipes file data from page cache directly to a socket. Zero copies between page cache and skb (when DMA-capable NIC).

```
disk ─► page cache ─────────────► skb (just pointer/ref) ─► NIC DMA
```

Used everywhere: NGINX, Apache, FTP servers, all the big file servers.

Limitations:
- `in_fd` must support mmap (regular file usually).
- `out_fd` was socket-only in older kernels; now more flexible.
- TLS breaks sendfile — encryption requires CPU touching the data. **kTLS fixes this** (next section).
- The data may bounce through skb-frag without a copy; called "page reference."

## 7.3 splice() — generalized pipe

```c
ssize_t splice(int fd_in, off_t *off_in, int fd_out, off_t *off_out, size_t len, unsigned int flags);
```

Moves data between two fds via an in-kernel pipe buffer, no userspace. Either side can be a regular file, socket, or pipe.

```
fd_in ─► pipe (in kernel) ─► fd_out
```

Use cases:
- Proxying: pipe(2) splice from socket A to socket B, no copy.
- Logging: socket → pipe → file.

`tee()` is the multi-dest variant (writes data to two fds).

socat with `-O,O_DIRECT` uses splice for zero-copy proxying.

## 7.4 MSG_ZEROCOPY — userspace buffer, no copy

`send(fd, buf, len, MSG_ZEROCOPY)` (4.14+).

Kernel pins `buf`'s pages, DMA-maps them, NIC reads directly. When done, app gets a completion via `MSG_ERRQUEUE` (recv with MSG_ERRQUEUE returns SCM_TIMESTAMPING with metadata).

App must not modify `buf` until completion.

```
userspace buf ─► (pinned) ─► NIC DMA
                    │
                    └─► completion notification → app frees buffer
```

Win: skips the copy from userspace to kernel send buffer.

Cost: pinning/unpinning is not free; per-call overhead. Win above ~10KB messages; loss for small.

Use cases: large-message servers (storage proxies, ZeroMQ apps, in-house dataplanes).

## 7.5 IORING_OP_SEND_ZC — io_uring zero-copy send

io_uring's version (6.0+). Equivalent semantics but integrated into the io_uring async model. Completion comes via CQE.

For pure-network apps moving to io_uring, this is the recommended path forward.

## 7.6 TCP_ZEROCOPY_RECEIVE — receive side

`getsockopt` with TCP_ZEROCOPY_RECEIVE (5.4+). Kernel returns the receive data as mmap'd pages, no copy.

```c
struct tcp_zerocopy_receive zc = {
    .address = (uintptr_t)mmap(...)/buffer,
    .length = expected,
};
getsockopt(fd, IPPROTO_TCP, TCP_ZEROCOPY_RECEIVE, &zc, &sizeof_zc);
```

Constraints:
- Only works on pages aligned to page size (4KB).
- Slower for small messages.
- Receive data must be aligned, contiguous.

Used by HFT shops, in-house DB replication.

## 7.7 TSO / GSO / UFO / GRO / LRO — segmentation offloads

| Acronym | What | Direction |
|---------|------|-----------|
| **TSO** | TCP Segmentation Offload | TX, hardware |
| **GSO** | Generic Segmentation Offload | TX, software fallback |
| **UFO** | UDP Fragmentation Offload | TX, deprecated (now USO in NICs) |
| **GRO** | Generic Receive Offload | RX, software |
| **LRO** | Large Receive Offload | RX, hardware |

The mental model:

**TX side**: app/TCP creates a "super-packet" (up to 64KB or 256KB with BIG_TCP). Kernel hands to NIC. NIC (TSO) or kernel (GSO) segments into MTU-sized packets.

**RX side**: NIC delivers MTU-sized packets. Kernel (GRO) or NIC (LRO) coalesces same-flow packets into one big skb.

Stack walk happens once per super-packet, not per MTU-packet. 10–100× CPU savings.

### LRO vs GRO

LRO is hardware-only; can be lossy (no exact 5-tuple match guarantee in some implementations). GRO is software; preserves strict invariants. Modern best practice: **GRO on, LRO off** for kernel correctness; LRO sometimes used in network appliances.

`ethtool -k eth0` lists; `ethtool -K eth0 gro on lro off` sets.

### GRO timeout

`net.core.gro_normal_batch`: how many packets to bundle. `gro_flush_timeout` (per netdev): max time to hold an incomplete coalesce. For low-latency apps, set timeout = 0 (flush immediately) or disable GRO.

## 7.8 RSS, RPS, RFS — RX flow steering

Single NIC + multi-CPU = need to spread RX across CPUs.

### RSS (Receive Side Scaling) — hardware

NIC hashes incoming packet's 4-tuple (or N-tuple via Toeplitz hash) → picks an RX queue → fires IRQ on the queue's affined CPU.

`ethtool -L eth0 combined N` sets queue count. `ethtool -X eth0 equal N` sets hash table.

Failure modes:
- Single flow (one elephant connection) → one CPU → bottleneck. RSS can't help.
- Non-IP traffic → not hashed → all on queue 0.
- Misconfigured affinity → IRQ on wrong CPU → cache misses.

### RPS (Receive Packet Steering) — software fallback

After RSS, kernel can re-hash and queue on another CPU's backlog. Configured via `/sys/class/net/eth0/queues/rx-X/rps_cpus` bitmask.

Pro: works on any NIC, can split a single RSS queue.
Con: extra per-packet hop; cost added.

### RFS (Receive Flow Steering) — locality-aware

Like RPS, but uses a flow → CPU map based on where the app's last `recvmsg()` came from. Goal: deliver packets to the CPU running the app.

`net.core.rps_sock_flow_entries` (global table size), `rps_flow_cnt` per-queue.

Why it matters: avoids cache miss bouncing the skb between CPUs.

### XPS (Transmit Packet Steering)

Maps CPU → TX queue. So a thread on CPU 5 always uses TX queue 5. Reduces lock contention on TX queue.

`/sys/class/net/eth0/queues/tx-X/xps_cpus`.

## 7.9 Interrupt coalescing

NICs can defer IRQs: wait for N packets or M microseconds, then fire one IRQ.

`ethtool -C eth0 rx-usecs 10 rx-frames 32`

Lower values → lower latency, higher CPU. Higher values → higher throughput, more latency.

NICs auto-tune (adaptive coalescing) — `adaptive-rx on`.

## 7.10 NIC offloads — checklist

`ethtool -k eth0`:

| Feature | What it does | When to enable |
|---------|--------------|----------------|
| `rx-checksumming` | NIC validates checksum | Always |
| `tx-checksumming` | NIC computes checksum | Always |
| `scatter-gather` | TX can use multiple buffers | Always |
| `tcp-segmentation-offload` (TSO) | NIC segments large TCP | Always for TCP |
| `generic-segmentation-offload` (GSO) | Software TSO | Always (fallback) |
| `generic-receive-offload` (GRO) | Software LRO | Always |
| `large-receive-offload` (LRO) | Hardware coalesces RX | Off (kernel correctness) |
| `tx-tcp-mangleid-segmentation` | TSO with TCP id mangling | NIC-specific |
| `tx-udp_tnl-segmentation` | TSO for tunneled (VXLAN) | If you use VXLAN |
| `rx-vlan-offload` | VLAN tagging in NIC | If you use VLANs |
| `rx-fcs` | Pass FCS up | Off normally |
| `hw-tc-offload` | tc rules pushed to NIC | If NIC supports (Mellanox CX-5+) |
| `rx-gro-hw` | NIC does GRO (avoid double-coalesce) | NIC-dependent |
| `tls-hw-record-creation` | NIC TLS (kTLS offload) | If NIC supports |
| `tls-hw-tx-offload` | TLS TX HW | If NIC supports |
| `tls-hw-rx-offload` | TLS RX HW | If NIC supports |

## 7.11 BIG_TCP (5.19+)

Default GSO/GRO max = 64KB (16-bit total length in IPv4 + 16-bit IPv6 jumbo extension limit).

BIG_TCP: for IPv6, the jumbogram option (RFC 2675) allows total length > 64KB. Kernel patch lets GSO/GRO up to 4GB.

Result: stack walks ~256× less per 16MB chunk. Cloudflare measured 30% throughput gain on Mellanox CX-6.

`ip link set dev eth0 gro_max_size 196608 gso_max_size 196608` (typical setting).

Caveats: only IPv6, NIC must support, peers must support (or fragments on egress).

## 7.12 NUMA awareness

In a NUMA box, NIC is attached to one socket. RX on a CPU in the wrong socket pays a QPI/UPI hop (~100ns per cacheline).

Best practice:
- Pin IRQs to CPUs on the NIC's NUMA node (`/proc/irq/<N>/smp_affinity`).
- Pin app threads to the same NUMA node (`taskset`, cgroups).
- Use `SO_INCOMING_CPU` (returns CPU last processing this socket; for thread placement).

`cat /sys/class/net/eth0/device/numa_node` shows which node.

## 7.13 Performance baselines

Single-flow TCP, 100Gbps NIC, modern Xeon:
- Default config (no tuning): ~30Gbps
- Tuned (GRO, irq pin, buffer sizes): ~80Gbps
- With BIG_TCP: ~95Gbps
- With kTLS sendfile: ~95Gbps at TLS

Multi-flow:
- Saturate 100Gbps requires ~16 CPUs with default config
- ~8 CPUs with tuned, GRO, RFS

Pps (packet rate):
- Default: ~3Mpps per CPU on RX, ~2Mpps on TX
- XDP_DROP: ~30Mpps per CPU
- DPDK: ~50Mpps per CPU (full NIC bypass)

These numbers are useful in interviews; learn them.

## 7.14 Common interview probes

- **"How do you go from 10Gbps to 100Gbps?"** → GSO/GRO/TSO/LRO, RSS+multi-queue, sendfile, large rmem/wmem, IRQ pin, NUMA, jumbo frames or BIG_TCP, kTLS sendfile if TLS.
- **"What if GRO doesn't merge?"** → Check 5-tuple match, gso_max_size, IPv6 routing header, NIC-specific limits. `ethtool -S` shows GRO counters.
- **"Why is single-flow 100Gbps hard?"** → One CPU does TCP processing; even with GRO/TSO, CPU saturates at ~80Gbps. Need either application-level striping (multi-flow) or higher per-CPU speed (kTLS sendfile, AF_XDP).
- **"How does sendfile interact with TLS?"** → Pre-kTLS: can't. kTLS makes TLS records in kernel; sendfile() now works for TLS. Netflix uses this to do 800Gbps from a single 4U box.

## 7.15 Corner cases

- **GRO drops a packet for being out-of-order.** Hash mismatch or IP options break coalescing. The kernel falls back to single-packet path; not a correctness issue but a perf loss.
- **TSO + IPv6 + tunneled.** Some old NICs don't support all combinations. `ethtool -k` will show, but `dmesg | grep -i tso` may reveal disabled paths.
- **MSG_ZEROCOPY completion lag.** If app forgets to read MSG_ERRQUEUE, completions queue and consume memory. Always drain.
- **`rmem_max` vs SO_RCVBUF.** Userspace can't exceed `rmem_max`. Bump both.
- **IRQ affinity reset on driver reload.** Some drivers reset on `ip link down/up`. Use systemd unit to re-apply.

## 7.16 Alternative approaches (the "3 ways" reflex)

| Need | Default | Alternative 1 | Alternative 2 |
|------|---------|---------------|---------------|
| Serve file over TCP | read+send | sendfile | kTLS+sendfile (TLS) |
| Send large message | send (copy) | MSG_ZEROCOPY | IORING_OP_SEND_ZC |
| Receive large message | recv (copy) | TCP_ZEROCOPY_RECEIVE | AF_XDP |
| Spread RX across CPUs | RSS only | RSS+RPS+RFS | XDP_REDIRECT |
| Reduce IRQ load | NAPI default | Coalescing | Busy polling |
| Maximize PPS | tuned stack | XDP_DROP | DPDK / AF_XDP |
| Saturate 100Gbps | tuned + GRO | + BIG_TCP | + kTLS sendfile |

## Must-internalize

- Three copies (page cache → user → skb → NIC) is too many for 100Gbps.
- `sendfile` eliminates user→skb; `splice` generalizes.
- `MSG_ZEROCOPY` and `IORING_OP_SEND_ZC` skip the kernel-side copy too.
- TSO/GSO on TX, GRO/LRO on RX → one stack walk per ~64KB.
- RSS spreads RX across NIC queues; RPS extends to more CPUs; RFS adds locality.
- `ethtool -k`, `ethtool -K`, `ethtool -S` — memorize these.
- BIG_TCP raises GRO/GSO max past 64KB for IPv6; ~30% gain on 100Gbps.
- NUMA: pin IRQ + app to the NIC's NUMA node.