# 22 · Coding and Syscall Problems

The networking-flavored coding round. Expect one of these to land in a 60-minute slot. Solve in C or Go (per your memory: this repo defaults to Go). Each problem comes with the "what's being tested" so you don't over-engineer.

## 22.1 TCP echo server (the warmup)

**Prompt**: write a TCP server that echoes back whatever it receives. Make it scale to 10K concurrent connections.

**Tests**: basic socket lifecycle, non-blocking I/O, edge cases.

**Skeleton (Go)**:

```go
package main

import (
    "io"
    "log"
    "net"
)

func main() {
    ln, err := net.Listen("tcp", ":8080")
    if err != nil { log.Fatal(err) }
    for {
        c, err := ln.Accept()
        if err != nil { log.Println(err); continue }
        go handle(c)
    }
}

func handle(c net.Conn) {
    defer c.Close()
    if _, err := io.Copy(c, c); err != nil {
        log.Println(err)
    }
}
```

**Scale-up discussion**:
- Go's runtime uses epoll under the hood — `net.Conn` is non-blocking; `io.Copy` parks the goroutine on read-ready.
- Goroutine per conn at 2KB stack each = ~20MB for 10K. Fine.
- 1M conns? Need to consider:
  - `ulimit -n` (file descriptors).
  - `tcp_max_tw_buckets`, `tcp_mem`.
  - `epoll` registration cost (M:N scheduler hides it).
- Read/write buffer sizing: `bufio.NewReader(c)` to amortize syscalls.

**C variant** uses `accept4(SOCK_NONBLOCK)` and `epoll` directly. Discuss why Go's runtime ≈ that pattern.

## 22.2 epoll-based concurrent server (C)

**Prompt**: write a single-threaded TCP server with epoll handling up to 10K conns.

**Tests**: epoll lifecycle, edge-triggered vs level-triggered, drain-loops, EPOLLERR/EPOLLHUP.

**Sketch (C, level-triggered for safety)**:

```c
#include <sys/epoll.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>

int set_nonblock(int fd) {
    return fcntl(fd, F_SETFL, fcntl(fd, F_GETFL) | O_NONBLOCK);
}

int main(void) {
    int s = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK | SOCK_CLOEXEC, 0);
    int one = 1;
    setsockopt(s, SOL_SOCKET, SO_REUSEADDR, &one, sizeof one);
    struct sockaddr_in sa = { .sin_family = AF_INET, .sin_port = htons(8080) };
    bind(s, (void*)&sa, sizeof sa);
    listen(s, 4096);

    int ep = epoll_create1(EPOLL_CLOEXEC);
    struct epoll_event ev = { .events = EPOLLIN, .data.fd = s };
    epoll_ctl(ep, EPOLL_CTL_ADD, s, &ev);

    struct epoll_event events[1024];
    char buf[16384];
    for (;;) {
        int n = epoll_wait(ep, events, 1024, -1);
        for (int i = 0; i < n; i++) {
            int fd = events[i].data.fd;
            if (fd == s) {
                while (1) {
                    int c = accept4(s, NULL, NULL, SOCK_NONBLOCK | SOCK_CLOEXEC);
                    if (c < 0) { if (errno == EAGAIN) break; perror("accept"); break; }
                    ev.events = EPOLLIN | EPOLLRDHUP;
                    ev.data.fd = c;
                    epoll_ctl(ep, EPOLL_CTL_ADD, c, &ev);
                }
            } else {
                ssize_t r = read(fd, buf, sizeof buf);
                if (r <= 0) { close(fd); continue; }
                ssize_t off = 0;
                while (off < r) {
                    ssize_t w = write(fd, buf+off, r-off);
                    if (w < 0) { if (errno == EAGAIN) break; close(fd); break; }
                    off += w;
                }
            }
        }
    }
}
```

**Discussion points**:
- Level-triggered with EPOLLIN: fires while readable. Don't have to drain.
- Why `accept4` in a loop: kernel may have multiple SYNs ready; one epoll wake.
- Edge-triggered would require draining (`while (read returns positive)`); if you miss, the fd stalls.
- `EPOLLEXCLUSIVE` would avoid thundering herd if multiple processes shared.

## 22.3 Implement an LRU cache for connection state

**Prompt**: design and implement an in-memory LRU keyed by 5-tuple → last-seen-time.

**Tests**: data structure choice, lock-free reads, memory accounting.

**Sketch (Go)**:

```go
type Key struct{ Src, Dst uint32; SPort, DPort uint16; Proto uint8 }
type Cache struct {
    mu    sync.Mutex
    list  *list.List
    items map[Key]*list.Element
    cap   int
}
type entry struct{ k Key; t time.Time }

func (c *Cache) Get(k Key) (time.Time, bool) {
    c.mu.Lock()
    defer c.mu.Unlock()
    if e, ok := c.items[k]; ok {
        c.list.MoveToFront(e)
        return e.Value.(*entry).t, true
    }
    return time.Time{}, false
}
func (c *Cache) Set(k Key, t time.Time) {
    c.mu.Lock()
    defer c.mu.Unlock()
    if e, ok := c.items[k]; ok {
        e.Value.(*entry).t = t
        c.list.MoveToFront(e)
        return
    }
    if c.list.Len() >= c.cap {
        e := c.list.Back()
        c.list.Remove(e)
        delete(c.items, e.Value.(*entry).k)
    }
    e := c.list.PushFront(&entry{k, t})
    c.items[k] = e
}
```

**Discussion**:
- One global mutex; scales linearly. For 10M qps: shard by hash of Key (similar to Go's map sharding).
- LRU vs LFU vs ARC: LRU is easiest; LFU is better for skewed workload.
- Memory: each entry is Key (12 bytes) + time (24) + map/list overhead. 1M entries ~ 64MB.
- For per-CPU lock-free: rely on Go's atomic + per-CPU maps OR per-shard sync.RWMutex.

## 22.4 Parse a TCP header from a byte slice

**Prompt**: given `data []byte`, parse a TCP header and return src port, dst port, seq, ack, flags.

**Tests**: endianness, struct layout, bounds checking.

```go
type TCPHdr struct {
    Src, Dst    uint16
    Seq, Ack    uint32
    DataOff     uint8
    Flags       uint8
    Win         uint16
    Csum, Urg   uint16
}

func ParseTCP(data []byte) (*TCPHdr, error) {
    if len(data) < 20 {
        return nil, errors.New("short")
    }
    h := &TCPHdr{
        Src:     binary.BigEndian.Uint16(data[0:2]),
        Dst:     binary.BigEndian.Uint16(data[2:4]),
        Seq:     binary.BigEndian.Uint32(data[4:8]),
        Ack:     binary.BigEndian.Uint32(data[8:12]),
        DataOff: data[12] >> 4,    // upper 4 bits = header words
        Flags:   data[13],
        Win:     binary.BigEndian.Uint16(data[14:16]),
        Csum:    binary.BigEndian.Uint16(data[16:18]),
        Urg:     binary.BigEndian.Uint16(data[18:20]),
    }
    if int(h.DataOff)*4 > len(data) {
        return nil, errors.New("header overflow")
    }
    return h, nil
}
```

**Edge cases**:
- DataOff in 32-bit words; multiply by 4 to get header bytes.
- Options follow; parse separately.
- Bounds check every slice access.
- Network byte order (big-endian) — use `encoding/binary`.

## 22.5 SO_REUSEPORT worker pool (Go)

**Prompt**: spawn N goroutines, each with its own `Listen` using SO_REUSEPORT. Test load distribution.

**Tests**: knowledge of SO_REUSEPORT, syscall integration in Go.

Go's `net` package doesn't expose SO_REUSEPORT directly; use `golang.org/x/sys/unix`:

```go
func reuseListen(port int) (net.Listener, error) {
    lc := net.ListenConfig{
        Control: func(_, _ string, c syscall.RawConn) error {
            var err error
            c.Control(func(fd uintptr) {
                err = unix.SetsockoptInt(int(fd), unix.SOL_SOCKET, unix.SO_REUSEPORT, 1)
            })
            return err
        },
    }
    return lc.Listen(context.Background(), "tcp", fmt.Sprintf(":%d", port))
}

func main() {
    n := runtime.NumCPU()
    var wg sync.WaitGroup
    for i := 0; i < n; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            ln, err := reuseListen(8080)
            if err != nil { log.Fatal(err) }
            // each worker's own accept loop
            for {
                c, err := ln.Accept()
                if err != nil { return }
                go handle(c, id)
            }
        }(i)
    }
    wg.Wait()
}
```

**Discussion**:
- Each goroutine has its own listen socket bound to same port.
- Kernel hashes incoming SYN to one of the sockets → one worker handles.
- Stress test with `wrk -t 16 -c 1000 ...`; verify each worker sees balanced load.
- For non-default hash, attach SO_ATTACH_REUSEPORT_EBPF — out of scope for the interview but worth mentioning.

## 22.6 Implement a producer-consumer queue with epoll

**Prompt**: one producer writes to a Unix domain socket; consumer reads via epoll; bounded backlog with backpressure.

**Tests**: pipes vs sockets, epoll, blocking strategy.

Could be solved with pipe or socketpair. Sketch:

```go
fds, _ := unix.Socketpair(unix.AF_UNIX, unix.SOCK_STREAM, 0)
producerFd, consumerFd := fds[0], fds[1]
// Write n in producer; on EAGAIN, wait via epoll EPOLLOUT.
// Consumer reads via epoll EPOLLIN; backpressures by not reading.
```

**Discussion**:
- Backpressure naturally via socket send buffer (`SO_SNDBUF`).
- Avoid `MSG_DONTWAIT` blocking; use non-blocking + epoll.
- Mention `eventfd` for signaling between threads if just need wake-up not data.

## 22.7 Write a UDP echo server

**Prompt**: implement; use `recvmmsg` for batching.

Go's standard library doesn't expose `recvmmsg`; need `golang.org/x/net/internal/socket` or `cgo`.

In C:

```c
#define MAX_MSGS 32
struct mmsghdr msgs[MAX_MSGS];
struct iovec iovs[MAX_MSGS];
char bufs[MAX_MSGS][2048];
struct sockaddr_in addrs[MAX_MSGS];

for (int i = 0; i < MAX_MSGS; i++) {
    iovs[i].iov_base = bufs[i];
    iovs[i].iov_len = sizeof bufs[i];
    msgs[i].msg_hdr.msg_iov = &iovs[i];
    msgs[i].msg_hdr.msg_iovlen = 1;
    msgs[i].msg_hdr.msg_name = &addrs[i];
    msgs[i].msg_hdr.msg_namelen = sizeof addrs[i];
}

int n = recvmmsg(sock, msgs, MAX_MSGS, 0, NULL);
sendmmsg(sock, msgs, n, 0);
```

**Discussion**:
- One syscall per batch; massive throughput gain at high pps.
- Combine with `SO_REUSEPORT` for multi-worker.
- UDP GRO (5.0+) further batches at kernel level.

## 22.8 Implement a simple XDP filter (pseudocode)

**Prompt**: drop all TCP packets to port 22 from outside our subnet.

In `bpf/`:

```c
SEC("xdp")
int filter(struct xdp_md *ctx) {
    void *data = (void *)(long)ctx->data;
    void *end  = (void *)(long)ctx->data_end;
    struct ethhdr *eth = data;
    if ((void*)(eth+1) > end) return XDP_PASS;
    if (eth->h_proto != bpf_htons(ETH_P_IP)) return XDP_PASS;

    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip+1) > end) return XDP_PASS;
    if (ip->protocol != IPPROTO_TCP) return XDP_PASS;

    struct tcphdr *tcp = (void *)ip + ip->ihl * 4;
    if ((void *)(tcp+1) > end) return XDP_PASS;

    if (tcp->dest != bpf_htons(22)) return XDP_PASS;

    __u32 src = ip->saddr;
    // Subnet 10.0.0.0/8 allowed
    if ((bpf_ntohl(src) & 0xff000000) == 0x0a000000) return XDP_PASS;
    return XDP_DROP;
}
char _license[] SEC("license") = "GPL";
```

**Discussion**:
- Verifier needs all pointer accesses bounded by `end`.
- Don't dereference past data_end; verifier rejects.
- Compile: `clang -O2 -target bpf -c filter.c -o filter.o`.
- Attach: `ip link set dev eth0 xdpdrv obj filter.o`.

## 22.9 Implement consistent hashing (Maglev)

**Prompt**: given N backends, build a lookup table of size M (prime) such that ID → backend with minimal disruption on backend add/remove.

**Sketch (Go)**:

```go
const M = 65537 // prime

type Maglev struct {
    table  []int       // M entries; idx → backend index
    perm   [][]int     // N x M permutations
}

func NewMaglev(backends []string) *Maglev {
    n := len(backends)
    perm := make([][]int, n)
    for i, b := range backends {
        offset := hash(b + "off") % M
        skip := hash(b + "skip")%(M-1) + 1
        p := make([]int, M)
        for j := 0; j < M; j++ {
            p[j] = (int(offset) + j*int(skip)) % M
        }
        perm[i] = p
    }
    table := make([]int, M)
    for i := range table { table[i] = -1 }
    next := make([]int, n)
    filled := 0
    for filled < M {
        for i := 0; i < n; i++ {
            for {
                c := perm[i][next[i]]
                next[i]++
                if table[c] == -1 {
                    table[c] = i
                    filled++
                    break
                }
            }
            if filled == M { break }
        }
    }
    return &Maglev{table: table, perm: perm}
}

func (m *Maglev) Lookup(flowHash uint32) int {
    return m.table[flowHash%M]
}
```

**Discussion**:
- Prime M minimizes collisions in modular arithmetic.
- Each backend has its own offset/skip → unique permutation.
- The fill loop guarantees a balanced table; one entry per backend round-robin.
- Disruption analysis: removing a backend, only its ~M/N entries need to be repopulated.

## 22.10 Implement a token bucket rate limiter

**Prompt**: thread-safe; allow N requests per second per key.

```go
type Bucket struct {
    mu      sync.Mutex
    tokens  float64
    rate    float64
    cap     float64
    last    time.Time
}

func (b *Bucket) Allow() bool {
    b.mu.Lock()
    defer b.mu.Unlock()
    now := time.Now()
    elapsed := now.Sub(b.last).Seconds()
    b.tokens = math.Min(b.cap, b.tokens + elapsed*b.rate)
    b.last = now
    if b.tokens >= 1 {
        b.tokens--
        return true
    }
    return false
}
```

**Discussion**:
- Per-key map of buckets; shard for high concurrency.
- For distributed: use Redis with Lua script for atomic decrement.
- Variants: leaky bucket (smooth output), sliding window log (precise), GCRA.

## 22.11 Detect a packet loop in routing

**Prompt**: given a routing graph (node → next-hop), detect if a packet starting at A ever loops.

DFS with cycle detection. Standard graph algo; not really networking but framed as one.

```go
func DetectLoop(routes map[string]string, start string) bool {
    seen := map[string]bool{}
    for cur := start; ; {
        if seen[cur] { return true }
        seen[cur] = true
        next, ok := routes[cur]
        if !ok { return false }
        cur = next
    }
}
```

**Discussion**:
- TTL is the protocol-level safety; this is the offline analysis.
- For ECMP (multi-next-hop) it's BFS.

## 22.12 Build a simple eBPF program counter via libbpf-go

**Prompt**: count syscalls per process.

Mostly libbpf/cilium-ebpf glue. Sketch the architecture:

```go
import "github.com/cilium/ebpf"
import "github.com/cilium/ebpf/link"
// load BPF object, attach to tracepoint sys_enter_*, expose map to userspace.
```

Discuss verifier constraints; PERCPU_ARRAY for lock-free per-CPU counters.

## 22.13 Coding patterns to internalize

| Pattern | Where it appears |
|---------|------------------|
| Non-blocking accept loop | Every C server |
| epoll level-triggered drain | C, Rust |
| Goroutine-per-conn | Go (the standard idiom) |
| SO_REUSEPORT sharding | Multi-worker servers |
| recvmmsg/sendmmsg | High-rate UDP |
| Token bucket | Rate limiter |
| Consistent hashing | LB |
| 5-tuple hash → backend | LB |
| BPF map + tail call | eBPF data plane |
| Lock-free per-CPU counter | Metrics |
| Read-then-write on socket | TCP echo / proxy |

## 22.14 Common interview gotchas

- **EAGAIN handling**: forgetting to handle it on either `read()` or `write()` will silently stall.
- **Partial writes**: `write(fd, buf, n)` may return < n; always loop.
- **`accept` race**: between epoll wake and accept, another worker may grab.
- **`close` from wrong thread**: close after another thread is in `read()` → undefined.
- **Endianness**: forgetting to swap to network byte order.
- **Bounds checking**: kernel-space code; missing bounds = verifier reject.
- **Resource cleanup**: every alloc → matching free; `epoll_ctl` registration → matching DEL before close.
- **Use of `defer` in Go**: paying ~10ns per defer; not in hot path.

## Must-internalize

- Go idiom: goroutine per conn; runtime hides epoll.
- C idiom: accept4(NONBLOCK) + epoll + drain loop.
- High-rate UDP: recvmmsg + SO_REUSEPORT.
- Rate limit: token bucket (sliding window for precision).
- Consistent hash: Maglev table (size 65537).
- eBPF: bounds-check every dereference for the verifier.