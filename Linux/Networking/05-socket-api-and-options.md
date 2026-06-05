# 05 · Socket API and Options

The socket API is BSD 4.2 (1983) and has barely changed. The depth lives in setsockopt — every nontrivial network application sets at least a dozen of these, and staff candidates should know which ones matter and why.

## 5.1 The lifecycle

```
  socket()  ─►  bind()  ─►  listen()  ─►  accept()  ─►  read/write  ─►  close()
                                                ▲
                                                │
                                                └─ active side: connect() instead
```

`socket(domain, type, protocol)`:
- `domain`: AF_INET, AF_INET6, AF_UNIX, AF_PACKET, AF_NETLINK, AF_XDP, AF_VSOCK, AF_ALG, AF_KCM.
- `type`: SOCK_STREAM, SOCK_DGRAM, SOCK_RAW, SOCK_SEQPACKET. Also flag bits: `SOCK_NONBLOCK`, `SOCK_CLOEXEC`.
- `protocol`: 0 for default; `IPPROTO_TCP`, `IPPROTO_UDP`, etc.

Returns an fd. Closing it (or process exit) releases the resource.

## 5.2 The socket fd is a kernel object

Inside the kernel: `struct file` → `struct socket` → `struct sock` (`struct tcp_sock` for TCP).

- `socket` is the file-system view; supports `read`, `write`, `poll`, etc.
- `sock` is the protocol-agnostic socket state.
- `tcp_sock`, `udp_sock`, `unix_sock` are protocol-specific extensions.

This layering is why `epoll` works on sockets like any other fd — VFS is the abstraction.

## 5.3 bind() and listen()

`bind(fd, addr, addrlen)` associates a fd with a local address. For TCP:
- Permits accepting connections to that address.
- Reserves an ephemeral port if `port=0` (kernel picks).

Common errors:
- `EADDRINUSE` — port taken. Mitigations: `SO_REUSEADDR`, `SO_REUSEPORT`, wait for TIME-WAIT.
- `EACCES` — privileged port (<1024) without CAP_NET_BIND_SERVICE.

`listen(fd, backlog)` puts the socket in `LISTEN` state.
- `backlog` is the **accept-queue depth**, capped by `somaxconn` (sysctl `net.core.somaxconn`, default 4096 in modern kernels, was 128 historically).
- Backlog being too small causes connection reset under burst.

## 5.4 accept() and accept4()

`accept(fd, addr, addrlen)` returns a new fd for the accepted connection. The listener fd stays in LISTEN.

`accept4()` adds flags: `SOCK_NONBLOCK`, `SOCK_CLOEXEC`. Use `accept4()` — avoids a separate `fcntl()` call.

For high-rate accept loops:
- Set `SO_REUSEPORT` and spawn one listener per worker → kernel sharding.
- Set `EPOLLEXCLUSIVE` on epoll instance (kernel 4.5+) → one wakeup per event, not all waiters.

## 5.5 connect() and the active path

`connect(fd, addr, addrlen)`:
- Sends SYN, waits for SYN-ACK + ACK exchange.
- Blocks unless `O_NONBLOCK` set; returns `EINPROGRESS` then completes async (use `epoll` for write-ready).
- For UDP, just sets default destination (no packets sent).

Local source port is picked by `inet_csk_get_port()`. Range: `ip_local_port_range` (default `32768-60999`).

### Ephemeral port exhaustion

With ~28K source ports, a single client can have at most ~28K simultaneous outbound connections to one (dst-ip, dst-port). Common in proxies (NGINX, HAProxy) talking to a single upstream.

Mitigations:
- Multiple proxy IPs (round-robin source) — `SO_BINDTODEVICE` or `IP_BIND_ADDRESS_NO_PORT`.
- TIME-WAIT reuse (`tcp_tw_reuse=1` is safe; `tcp_tw_recycle` was removed).
- Keep-alive / connection pooling (the right answer 95% of the time).

## 5.6 The setsockopt menu — what staff knows

### Universal options (`SOL_SOCKET`)

| Option | Effect |
|--------|--------|
| `SO_REUSEADDR` | Bind to address even if in TIME-WAIT. |
| `SO_REUSEPORT` | Multiple sockets share the same (IP, port); kernel load-balances incoming. |
| `SO_KEEPALIVE` | Enable TCP keepalive (probe idle connections). |
| `SO_SNDBUF` / `SO_RCVBUF` | Set send/receive buffer size. **Disables autotune.** |
| `SO_SNDLOWAT` / `SO_RCVLOWAT` | Minimum bytes for `select`/`poll` to fire. |
| `SO_LINGER` | On close, block until data drained or timeout. |
| `SO_BROADCAST` | Allow sending to broadcast addresses. |
| `SO_BINDTODEVICE` | Bind socket to a specific interface (CAP_NET_RAW). |
| `SO_PRIORITY` | Set skb priority for qdisc classification. |
| `SO_MARK` | Set skb mark for netfilter/policy routing. |
| `SO_TIMESTAMPING` | Get hardware/software TX/RX timestamps in cmsg. |
| `SO_BUSY_POLL` | Poll the NIC for this socket; trade CPU for latency. |
| `SO_ZEROCOPY` | Enable MSG_ZEROCOPY for send. |
| `SO_INCOMING_CPU` | Bind socket to the CPU receiving its packets. |

### TCP options (`IPPROTO_TCP`)

| Option | Effect |
|--------|--------|
| `TCP_NODELAY` | Disable Nagle. |
| `TCP_CORK` | Hold data until uncork or MSS. |
| `TCP_QUICKACK` | Disable delayed ACK briefly. |
| `TCP_KEEPIDLE` | Seconds before first keepalive probe (default `tcp_keepalive_time`=7200). |
| `TCP_KEEPINTVL` | Seconds between probes. |
| `TCP_KEEPCNT` | Probes before giving up. |
| `TCP_USER_TIMEOUT` | Max time data may stay unACKed before connection aborts. |
| `TCP_FASTOPEN` | Server-side: enable TFO. |
| `TCP_FASTOPEN_CONNECT` | Client: pre-arm TFO data. |
| `TCP_CONGESTION` | Per-socket congestion control choice. |
| `TCP_NOTSENT_LOWAT` | Don't fill kernel buffer past this; useful for HTTP/2 prioritization. |
| `TCP_INFO` | Read socket stats (RTT, retransmits, cwnd). |
| `TCP_DEFER_ACCEPT` | accept() doesn't return until data arrives. |
| `TCP_MAXSEG` | Override MSS (rare). |
| `TCP_REPAIR` | For container live migration: dump+restore TCP state. |
| `TCP_TX_DELAY` | Pacing delay (server-set). |
| `TCP_ULP` | Attach a "upper layer protocol" — kTLS sets this. |
| `TCP_INQ` | recvmsg returns bytes-available in cmsg. |
| `TCP_ZEROCOPY_RECEIVE` | Read with mmap-style zero-copy (5.4+). |

### IP options (`IPPROTO_IP`)

| Option | Effect |
|--------|--------|
| `IP_TOS` | Set ToS / DSCP. |
| `IP_TTL` | TTL for outgoing. |
| `IP_MTU_DISCOVER` | PMTUD mode: DO / DONT / WANT / PROBE. |
| `IP_PKTINFO` | Receive ancillary data about each packet (dst IP, ifindex). |
| `IP_FREEBIND` | bind() to non-local address (for transparent proxy, anycast). |
| `IP_TRANSPARENT` | Receive packets destined to any address (TPROXY). |
| `IP_RECVERR` | Get extended error info via MSG_ERRQUEUE. |
| `IP_BIND_ADDRESS_NO_PORT` | Defer port selection until connect — better ephemeral utilization. |
| `IP_LOCAL_PORT_RANGE` | Per-socket port range (5.16+). |

## 5.7 `SO_REUSEPORT` deep dive

Pre-`SO_REUSEPORT`, scaling accept() to N CPUs required either:
- One thread doing `accept()` on a shared listen socket → lock contention on `listen_lock`.
- N processes inheriting the listen fd via fork → races and unfair distribution.

`SO_REUSEPORT` (3.9+) lets N sockets bind the same (IP, port). Kernel hashes incoming SYN by 4-tuple to one of the sockets.

Key properties:
- Each socket has its own accept queue.
- Hashing is deterministic for a 4-tuple → connection migration preserved.
- Adding/removing a socket re-hashes → existing connections unaffected (mapped via established hash), new SYNs may land elsewhere.
- `SO_REUSEPORT_CBPF` / `SO_ATTACH_REUSEPORT_EBPF` (4.5+): provide your own hash function. Cilium / Katran use this to steer to a specific worker.

Pitfall: if one worker dies, its accept queue is *orphaned* until the kernel reassigns. SYNs hash to a dead socket → drops until something rebalances. Mitigation: liveness check + remove the dead socket.

## 5.8 Non-blocking I/O

Blocking I/O blocks the thread. Non-blocking I/O returns `EAGAIN`/`EWOULDBLOCK` if no progress.

Set with `O_NONBLOCK` via `fcntl()` or `SOCK_NONBLOCK` flag at socket creation.

Idioms:
- `accept4(...SOCK_NONBLOCK...)` instead of `accept` + `fcntl`.
- `recv` returns `-1` with `EAGAIN` → no data, poll the fd later.
- `send` returns partial count or `EAGAIN` → buffer fills, wait for write-ready.

Non-blocking is the building block for epoll, io_uring, async runtimes.

## 5.9 Edge cases in send/recv

- **Partial reads/writes.** Especially with non-blocking. Always loop and accumulate.
- **`MSG_PEEK`.** Read without consuming. Useful for upper-layer protocol sniffing.
- **`MSG_WAITALL`.** Block until full requested bytes received (only for SOCK_STREAM and only if no signal).
- **`MSG_DONTWAIT`.** Single-call non-blocking, even if socket is blocking.
- **`MSG_MORE`.** Like CORK for one send.
- **Multiple messages: `sendmmsg`/`recvmmsg`.** Cuts syscalls drastically.
- **`MSG_ZEROCOPY` on send.** Avoids the copy_from_user; completion via MSG_ERRQUEUE.

## 5.10 AF_UNIX — Unix domain sockets

Local IPC over filesystem path or abstract namespace (`\0name`).

```
sockets:    SOCK_STREAM, SOCK_DGRAM, SOCK_SEQPACKET
faster than TCP (no IP/TCP stack)
   ─ no fragmentation, no congestion control
   ─ supports passing file descriptors (SCM_RIGHTS), credentials (SCM_CREDENTIALS)
   ─ used by Docker (docker.sock), systemd, X11, dbus
```

Performance numbers (rough): on a single machine, AF_UNIX stream is ~2× faster than TCP loopback. AF_UNIX datagram is faster still but caps at small messages.

SCM_RIGHTS is critical for sandboxing: a parent process can hand a file fd to a less-privileged child without that child having permission to open it itself.

## 5.11 AF_PACKET — raw L2 sockets

Bypasses the IP layer. Used by tcpdump (libpcap), Wireshark, custom packet processors.

Modes:
- `SOCK_RAW` — includes Ethernet header.
- `SOCK_DGRAM` — header stripped.

Special:
- `PACKET_MMAP` — zero-copy ring buffer (TX_RING / RX_RING).
- `PACKET_FANOUT` — multiple sockets share via hash / cpu / queue mapping (scaling).
- `PACKET_AUXDATA` — receive VLAN tag and checksum status.

A naive tcpdump on a 10Gbps link drops packets unless `PACKET_MMAP` is used; `tcpdump -B` sets buffer.

## 5.12 AF_NETLINK — kernel ↔ userspace

The "kernel control plane" socket family. iproute2 (`ip`, `tc`, `ss`), netlink-aware daemons (NetworkManager, systemd-networkd, FRRouting, BIRD).

Families:
- `NETLINK_ROUTE` — routes, addresses, interfaces, neighbors.
- `NETLINK_NETFILTER` — firewall.
- `NETLINK_AUDIT` — audit events.
- `NETLINK_GENERIC` — extensible; used by many subsystems.
- `NETLINK_XFRM` — IPsec policy.

Netlink messages are TLV; the kernel speaks one canonical format. Way more discoverable than the older `ioctl()` API.

## 5.13 AF_PACKET vs AF_XDP

Both let userspace see raw packets.
- AF_PACKET (with PACKET_MMAP) — copies into a mmaped buffer.
- AF_XDP — true zero-copy on supported NICs; ring directly mapped from NIC. ~5–10× faster.

For a packet sniffer at 100Gbps: AF_XDP. For a casual tool: AF_PACKET.

## 5.14 Common interview probe: "how does `nc | nc` move data?"

```
read(stdin) ─► socket1.send() ─► kernel TCP buffer ─► network ─► remote
                                                       │
                                                       ▼
                                                socket2.recv() ─► stdout
```

The kernel does the copying. Three copies per byte without zero-copy: user→kernel (send), kernel→kernel (skbs), kernel→user (recv).

`socat` and tools can use `splice()` for zero-copy: kernel-to-kernel pipe, no userspace touch.

## 5.15 Corner cases

| Symptom | Cause | Mitigation |
|---------|-------|------------|
| `EADDRINUSE` on quick restart | TIME-WAIT holding port | `SO_REUSEADDR`; or wait `2×MSL` |
| `accept` very slow | Backlog small or app slow | Bump `somaxconn`; profile |
| `connect` returns `EADDRNOTAVAIL` | Ephemeral port exhausted | More source IPs; reuse |
| `send` returns less than asked | Send buffer fill | Loop; check `EAGAIN` |
| `recv` returns 0 | Peer closed | App must close fd; else CLOSE-WAIT pile |
| Wakeup thunder | Many threads on shared listen | `EPOLLEXCLUSIVE`, `SO_REUSEPORT` |
| Stuck CLOSE on app exit | `SO_LINGER` with timeout=0 sends RST | Use linger=1 for graceful |

## 5.16 Alternative implementations

| Need | Default | Alternative 1 | Alternative 2 |
|------|---------|---------------|---------------|
| Single-thread server | blocking accept + thread per conn | epoll + non-blocking | io_uring |
| Many CPU server | shared listen + SO_REUSEPORT | accept loop per worker | XDP_REDIRECT to per-CPU socket |
| Local IPC | TCP loopback | AF_UNIX | shared memory + futex |
| Raw packet capture | AF_PACKET | PACKET_MMAP | AF_XDP |
| Pass file fd | open + dup over fork | AF_UNIX SCM_RIGHTS | pidfd + getfd |
| Kernel control | ioctl | netlink | sysfs |
| Per-process firewall | iptables | nftables | eBPF cgroup hook |

## Must-internalize

- Socket fd is a `struct file` over `struct socket` over `struct sock`; that's why epoll works.
- `SO_REUSEPORT` shards accept() across CPUs; with `SO_ATTACH_REUSEPORT_EBPF` you control the hash.
- `TCP_NODELAY` for RPC; `TCP_QUICKACK` and `TCP_CORK` for special cases.
- `SO_RCVBUF`/`SO_SNDBUF` disable autotune — set only if you understand the BDP.
- `accept4(...SOCK_NONBLOCK|SOCK_CLOEXEC...)` saves syscalls.
- Ephemeral port exhaustion at ~28K outbound to single (dst-ip, dst-port).
- AF_UNIX with SCM_RIGHTS is the kernel's fd-passing mechanism.
- AF_NETLINK is how iproute2 talks to the kernel.