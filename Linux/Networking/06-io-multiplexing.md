# 06 · I/O Multiplexing — select, poll, epoll, io_uring

The set of mechanisms for handling many concurrent connections from one thread. Every modern server (NGINX, HAProxy, Envoy, Go runtime, Node.js, Java NIO) uses one of these. Staff candidates should explain the *exact* differences and the failure mode of each.

## 6.1 The problem

You have N sockets. You want to know when *any* of them has I/O ready, without spawning N threads.

Naive `recv()` blocks one socket at a time. Polling each socket non-blocking burns CPU. The solution is *one syscall that tells you which fds are ready*.

## 6.2 select() — RFC-era multiplexing

```c
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

- Three fd-sets (bitmasks, 1024 bits each by default → max FD_SETSIZE=1024).
- Kernel walks every bit, checks each fd, copies result back.
- O(N) per call regardless of how many fds have events.

Pros: portable, simple.
Cons: FD_SETSIZE limit; full bitmap copy each call; O(N) scan; no kernel-side state.

Use today: only for cross-platform code.

## 6.3 poll() — slightly modernized

```c
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

- Array of `(fd, events, revents)`; no FD_SETSIZE limit.
- Kernel walks every entry, sets revents.
- Still O(N) per call.

Same fundamental problem as select.

## 6.4 epoll — the Linux winner

```c
int epoll_create1(int flags);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

Three ops:
- `EPOLL_CTL_ADD` — register fd with set of events.
- `EPOLL_CTL_MOD` — change events.
- `EPOLL_CTL_DEL` — unregister.

Kernel keeps a **red-black tree** of registered fds and a **ready list** of fds with pending events.

When a fd becomes ready (via `sk_data_ready()` callback for sockets), the kernel adds it to the ready list. `epoll_wait()` returns the list.

Cost: O(K) where K = number of *ready* fds, not number of *registered* fds.

This is the qualitative leap. C10K (1999 essay) is solved by epoll.

### Level-triggered vs Edge-triggered

`EPOLLET` flag changes the semantics:
- **Level-triggered (default).** epoll_wait returns as long as the fd is *ready*. If you read half the data, you'll get notified again next epoll_wait.
- **Edge-triggered.** epoll_wait returns only when the state *changes* from not-ready to ready. You must drain (read until EAGAIN) before waiting again.

Edge-triggered is faster (fewer wakeups) but harder to get right (must drain fully).

### EPOLLONESHOT
Auto-disable after one fire. Re-arm with `EPOLL_CTL_MOD`. Useful for thread pool that picks up the fd and processes.

### EPOLLEXCLUSIVE (4.5+)
Wake only one waiter when multiple epoll instances share a fd. Solves *thundering herd* on shared listen sockets.

Classic NGINX worker pattern:
```
N workers, each has its own epoll_fd.
Shared listen socket added to each with EPOLLEXCLUSIVE.
On SYN arrival, only one worker wakes.
```

### Compare/select epoll modes

| Mode | When wake | Drain required | Use when |
|------|-----------|----------------|----------|
| LT (default) | While ready | No | Beginners; safe |
| ET | On readiness edge | Yes | High-rate servers (NGINX) |
| LT + ONESHOT | Once until rearm | No | Thread pool dispatch |
| ET + ONESHOT | Once on edge | Yes | Specialized |
| LT + EXCLUSIVE | One waiter only | No | Shared listen socket |

### Common bugs

- **Forgetting to drain in ET.** Half-read leaves socket "still ready" but you never get another event. Hung connection.
- **Adding the same fd twice.** Was an error pre-2.6.something; now check return.
- **EPOLLRDHUP confusion.** Peer half-closed; some apps treat as EPOLLHUP and close immediately, losing buffered data.
- **EPOLLPRI for OOB data.** Rarely used; can confuse code that checks `events & EPOLLIN`.

## 6.5 epoll internals

```
epoll_create:    allocates an epoll_fd → underlying eventpoll struct
                 with RB tree (registered fds) + ready list + wait queue

epoll_ctl ADD:   inserts into RB tree, installs callback on fd's wait queue

fd ready:        callback fires, adds the fd entry to ready list,
                 wakes any epoll_wait waiters

epoll_wait:      sleeps on wait queue (if no ready) or returns ready list
```

Performance characteristics:
- O(log N) for `epoll_ctl` (RB tree).
- O(1) for the wake callback.
- O(K) for `epoll_wait` returning K events.

For 100K connections with 1K active at a time: epoll handles it; select/poll would spend 99% of CPU walking the list.

## 6.6 io_uring — the modern async API

Linux 5.1 (2019). Designed by Jens Axboe (block I/O maintainer). Originally for storage I/O; now covers most of the syscall surface.

Two ring buffers shared with kernel:
- **Submission Queue (SQ)**: app writes operations (read, write, accept, send, etc.).
- **Completion Queue (CQ)**: kernel writes results.

```
   userspace                                    kernel
   ┌────────────┐                          ┌────────────┐
   │  SQ ring   │ ── push ops ───────────► │ submit     │
   │  (mmap)    │                          │  thread/poll│
   └────────────┘                          └────────────┘
                                                 │ does the I/O
   ┌────────────┐                                │
   │  CQ ring   │ ◄── push results ──────────────┘
   │  (mmap)    │
   └────────────┘
```

App enqueues SQEs (submission queue entries), then optionally calls `io_uring_enter()` to inform kernel. Kernel processes entries and writes CQEs (completion queue entries).

### Two modes
- **Interrupt-driven**: `io_uring_enter()` wakes the kernel.
- **`IORING_SETUP_SQPOLL`**: dedicated kernel thread polls SQ → near-zero syscalls in steady state.

### Operations supported (subset)
- `IORING_OP_READ` / `WRITE`
- `IORING_OP_RECV` / `SEND`
- `IORING_OP_ACCEPT` / `CONNECT`
- `IORING_OP_RECVMSG` / `SENDMSG`
- `IORING_OP_SEND_ZC` (zero-copy send, 6.0+)
- `IORING_OP_OPENAT` / `CLOSE`
- `IORING_OP_SPLICE` / `TEE`
- `IORING_OP_NOP` (benchmark)
- Linked / chained operations (next runs on prior success)

### Why io_uring matters

- **Async by default.** No `O_NONBLOCK` dance.
- **Batched syscalls.** Submit 100 ops with one syscall.
- **Zero-copy buffer rings** (`IORING_OP_PROVIDE_BUFFERS`): kernel picks a buffer from a pool, copies/uses it, returns ID in CQE.
- **No per-op fd setup.** Registered buffers and fds avoid setup per op.

Benchmarks: a simple TCP echo server with io_uring is ~30% faster than epoll for short-lived connections. For TCP, advantage shrinks vs epoll due to existing socket fast path.

### Where io_uring genuinely wins
- High-rate I/O with many short ops (HTTP/3 / QUIC).
- Storage + network mixed (DB or proxy paginating from disk).
- Per-connection async logic without epoll bookkeeping.

### io_uring concerns
- **Security surface.** Many ops; some CVEs in early years. Some sandboxes (CrowdStrike) disable.
- **Verifier complexity.** Each op has its own kernel-side logic.
- **Adoption.** Mostly cutting-edge servers (tigerbeetle, Netflix VPP) and some hyperscalers; mainstream NGINX has experimental support.

## 6.7 select/poll/epoll/io_uring — decision rubric

| Property | select | poll | epoll | io_uring |
|----------|--------|------|-------|----------|
| Max fds | 1024 (FD_SETSIZE) | ulimit | ulimit | ulimit |
| Time per call | O(N registered) | O(N registered) | O(K ready) | O(K completed) |
| Syscalls per op | 1 | 1 | 1 (wait) + 1 (ctl) | ~0 with SQPOLL |
| Async accept/connect | No | No | No (manual EAGAIN) | Yes |
| Async file I/O | No | No | No (regular files aren't pollable) | Yes (registered buffers) |
| Cross-OS | Yes | Yes | Linux only | Linux only |
| Maturity | 40+ years | 25+ years | 20+ years | 5+ years |

When to pick what:
- Cross-platform → poll (or libev/libuv abstracting select/epoll/kqueue).
- Linux, high concurrency, file+network → io_uring.
- Linux, high concurrency, network only → epoll.

## 6.8 The thundering herd problem

A wakeup wakes more threads than have work. Common scenarios:

### 1. Shared listen socket via fork()
Pre-`SO_REUSEPORT`, N workers do `accept()` on same fd. SYN arrives → kernel wakes one worker. *But it used to wake all*; only one would get the connection, rest go back to sleep. Wasted wakeups.

Fix: kernel since 2.6 wakes one; `EPOLL_EXCLUSIVE` for shared epoll case.

### 2. Multiple epoll_fds sharing a fd
Each epoll has the fd; all of them wake on readiness. Use `EPOLLEXCLUSIVE`.

### 3. SO_REUSEPORT N workers
Different sockets; kernel hashes SYN → one socket. No herd.

This is why modern servers (NGINX, HAProxy) use SO_REUSEPORT + one epoll per worker.

## 6.9 Userspace event loops on top

| Library | Backend | Used by |
|---------|---------|---------|
| libev | select/poll/epoll/kqueue | dovecot, node (early) |
| libuv | select/poll/epoll/kqueue/io_uring | Node.js |
| libevent | similar | tor, memcached |
| Boost.Asio | similar + Windows IOCP | C++ |
| tokio | epoll/io_uring | Rust |
| Go runtime | netpoll (epoll/kqueue) | every Go server |
| Java NIO/Netty | epoll/kqueue | almost all Java servers |

Each abstracts the platform, but the staff candidate should know which backend their stack uses.

## 6.10 Go's netpoll — case study

Go's runtime built its own scheduler on top of epoll. Each goroutine doing I/O parks on a netpoll; runtime registers fd with epoll (LT mode). When ready, goroutine resumes on a worker P.

Key points:
- One epoll fd per netpoll instance (per-P or per-thread varies by Go version).
- Goroutine is the unit of scheduling, not a thread; M:N scheduler.
- `netpoll` runs as part of scheduler `findRunnable` — so latency is bounded by scheduler tick.
- Go does NOT use ET — too easy to get wrong; LT is the safe default.

Cost: a goroutine costs ~2KB stack + bookkeeping. Tens of millions per host possible.

Comparison: Node.js has 1 thread + libuv; Go has many threads + per-goroutine state. Trade-off: Node is simpler but can't use multiple cores without IPC; Go uses them naturally.

## 6.11 Common interview questions

- **"Why doesn't poll scale?"** → O(N) per call; kernel walks every registered fd. epoll is event-driven instead.
- **"What's the difference between LT and ET?"** → LT fires while ready; ET fires only on transition.
- **"Why does NGINX use ET?"** → Fewer wakeups per request; designed around drain-loop pattern.
- **"Why does Go use LT?"** → Simpler runtime; goroutine resume is cheap.
- **"How would you implement an event loop?"** → epoll instance, fd → callback map, accept loop, drain on read, write when EPOLLOUT set.
- **"What's io_uring's advantage over epoll for TCP?"** → Avoids per-op syscall; allows pre-registered buffer rings; async accept/connect. For TCP-only it's incremental; bigger win in mixed I/O.

## 6.12 Corner cases

- **`epoll_wait` returns 0** — timeout; treat as no events, not error.
- **`EPOLLHUP` without `EPOLLIN`** — peer closed, no data left. Always check this.
- **`EPOLLERR`** — async error; read `SO_ERROR` to get errno.
- **`SIGPIPE` on write to closed peer** — install handler or use `MSG_NOSIGNAL` per send.
- **Stale event after close** — closing an fd that's in epoll: kernel removes; but if you close in another thread while wait is in progress, races possible. `EPOLL_CTL_DEL` before close.
- **io_uring SQPOLL CPU pinning** — the polling thread can saturate a CPU; pin it (`SQ_THREAD_CPU`) to keep it off application cores.
- **io_uring file ops on networked filesystems** — async wraps to a workqueue; not always asynchronous in the literal sense.

## 6.13 Alternative implementations (the "3 ways" reflex)

| Need | Path 1 | Path 2 | Path 3 |
|------|--------|--------|--------|
| Multi-conn echo | thread/conn (blocking) | epoll | io_uring |
| Reduce syscall | epoll batch | io_uring SQPOLL | userspace polling (DPDK) |
| Async accept | accept + EAGAIN loop | io_uring OP_ACCEPT | XDP redirect + custom accept |
| Async filesystem | thread pool | io_uring OP_READ | AIO (deprecated) |
| Cross-platform | libuv | tokio | Go runtime |
| Low-latency one-conn | blocking read | non-block + busy poll | DPDK polling |

## Must-internalize

- select: O(N) per call, 1024-fd cap, copies bitmap. Cross-platform fallback.
- poll: O(N) per call, no FD limit. Same problem.
- epoll: O(K ready), kernel-tracked. Linux's answer to C10K.
- io_uring: shared-memory ring, async by default, batched syscalls. The future for mixed I/O.
- LT (level-triggered): safe default. ET: faster, must drain.
- `EPOLLEXCLUSIVE` and `SO_REUSEPORT`: prevent thundering herd.
- NGINX uses ET + SO_REUSEPORT + per-worker epoll. Go uses LT + netpoll.