# 9 · Coding Problems — Go-Flavored, Tuned to Infoblox

Coding rounds at Infoblox tend to be 45–60 minutes, screen-share, live coding. Reports mention Go-language debugging rounds, classical DSA (linked lists, trees, fast/slow pointers, two-sum), and at least one problem with a networking flavor (parse a packet, build a trie, build a rate limiter). At staff level you should also expect a problem where the *follow-up* matters more than the initial code — "now make it concurrent", "now make it persist", "now handle millions of these per second".

The repo's convention is **Go**. All samples below are Go. Tests, where helpful, are sketched, not full.

## P1 · LRU Cache (the canonical opener)

**Prompt**: implement an LRU cache with `Get(key)` and `Put(key, value)` in O(1) average time. Specify capacity at construction.

**Approach**: hash map → doubly-linked-list node. Move-to-front on access.

```go
package lrucache

type node struct {
    key, val   int
    prev, next *node
}

type LRU struct {
    cap        int
    m          map[int]*node
    head, tail *node // sentinels
}

func New(capacity int) *LRU {
    h, t := &node{}, &node{}
    h.next, t.prev = t, h
    return &LRU{cap: capacity, m: make(map[int]*node, capacity), head: h, tail: t}
}

func (c *LRU) Get(key int) (int, bool) {
    n, ok := c.m[key]
    if !ok {
        return 0, false
    }
    c.detach(n)
    c.attachAfter(c.head, n)
    return n.val, true
}

func (c *LRU) Put(key, val int) {
    if n, ok := c.m[key]; ok {
        n.val = val
        c.detach(n)
        c.attachAfter(c.head, n)
        return
    }
    if len(c.m) == c.cap {
        lru := c.tail.prev
        c.detach(lru)
        delete(c.m, lru.key)
    }
    n := &node{key: key, val: val}
    c.attachAfter(c.head, n)
    c.m[key] = n
}

func (c *LRU) detach(n *node) {
    n.prev.next = n.next
    n.next.prev = n.prev
}

func (c *LRU) attachAfter(prev, n *node) {
    n.prev = prev
    n.next = prev.next
    prev.next.prev = n
    prev.next = n
}
```

**Follow-ups they'll ask**:
- **Concurrency** — wrap with a `sync.Mutex`. Discuss reader/writer lock vs. plain mutex. (Plain mutex usually wins for LRU because Get also mutates.)
- **TTL** — add an `expiresAt` field; lazy-expire on Get, optionally a janitor goroutine for proactive eviction.
- **Sharded** — N shards keyed by `hash(key) % N` to reduce lock contention. *Now* RWMutex helps less, sharding helps more.
- **Memory accounting** — bound by total bytes instead of count; track per-entry size.

This is also exactly how a DNS resolver cache is built.

---

## P2 · DNS-Name Trie (suffix tree for domain names)

**Prompt**: implement a structure that holds millions of blocked domain names and answers "is `x.y.z` blocked, by what rule?" in O(label-count) time. Wildcards (`*.evil.example`) supported.

**Approach**: trie keyed by **reversed labels**. So `mail.evil.example` is inserted as `example` → `evil` → `mail`. The trie node optionally carries a "rule" (block/allow). A node with the wildcard flag matches any descendant.

```go
package dnstrie

import "strings"

type node struct {
    children map[string]*node
    rule     string // "" means no rule
    wildcard bool
}

type Trie struct {
    root *node
}

func New() *Trie { return &Trie{root: &node{}} }

func (t *Trie) Insert(pattern, rule string) {
    labels := strings.Split(pattern, ".")
    cur := t.root
    wild := false
    // strip leading "*"
    if len(labels) > 0 && labels[0] == "*" {
        wild = true
        labels = labels[1:]
    }
    // walk root-to-leaf in TLD-first order
    for i := len(labels) - 1; i >= 0; i-- {
        l := labels[i]
        if cur.children == nil {
            cur.children = make(map[string]*node)
        }
        nxt, ok := cur.children[l]
        if !ok {
            nxt = &node{}
            cur.children[l] = nxt
        }
        cur = nxt
    }
    cur.rule = rule
    cur.wildcard = wild
}

// Lookup returns the most-specific matching rule, or "" if none.
func (t *Trie) Lookup(name string) string {
    labels := strings.Split(name, ".")
    cur := t.root
    best := ""
    for i := len(labels) - 1; i >= 0; i-- {
        // a wildcard at the current level covers any deeper name
        if cur.wildcard && cur.rule != "" {
            best = cur.rule
        }
        nxt, ok := cur.children[labels[i]]
        if !ok {
            return best
        }
        cur = nxt
        if cur.rule != "" && !cur.wildcard {
            best = cur.rule
        }
    }
    if cur.wildcard && cur.rule != "" {
        best = cur.rule
    }
    return best
}
```

**Follow-ups**:
- **Memory** for 10M domains: optimize by interning label strings; or replace `map[string]*node` with a sorted slice when small.
- **Thread safety** — typically write-once at boot, read-many. Use `atomic.Pointer[Trie]` to swap on rebuild.
- **Updates** — instead of mutating, build a new trie offline, atomic-swap.
- **Bloom-filter front** for fast "definitely not blocked" rejection.

This is exactly the data structure behind every DNS firewall / RPZ implementation.

---

## P3 · Token-Bucket Rate Limiter (concurrent)

**Prompt**: implement a token-bucket rate limiter usable from many goroutines. `Allow(key)` returns true if a token is available.

```go
package ratelimit

import (
    "sync"
    "time"
)

type bucket struct {
    tokens     float64
    lastRefill time.Time
}

type Limiter struct {
    rate     float64 // tokens per second
    capacity float64
    mu       sync.Mutex
    buckets  map[string]*bucket
}

func New(rate, capacity float64) *Limiter {
    return &Limiter{rate: rate, capacity: capacity, buckets: make(map[string]*bucket)}
}

func (l *Limiter) Allow(key string) bool {
    now := time.Now()
    l.mu.Lock()
    defer l.mu.Unlock()
    b, ok := l.buckets[key]
    if !ok {
        b = &bucket{tokens: l.capacity, lastRefill: now}
        l.buckets[key] = b
    }
    elapsed := now.Sub(b.lastRefill).Seconds()
    b.tokens += elapsed * l.rate
    if b.tokens > l.capacity {
        b.tokens = l.capacity
    }
    b.lastRefill = now
    if b.tokens >= 1 {
        b.tokens--
        return true
    }
    return false
}
```

**Follow-ups**:
- Memory leak: a huge keyspace fills `l.buckets`. Add a janitor that evicts buckets that are full *and* haven't been touched in N seconds.
- Lock contention at high QPS: shard by `fnv32(key) % N`.
- Distributed (cross-instance): move state to Redis with a Lua script that atomically refills and decrements. Acknowledge that this trades latency for global accuracy.
- Use `golang.org/x/time/rate.Limiter` for the per-key case — but staff is expected to know what's under the hood.

---

## P4 · Concurrent DNS-Query Worker Pool (the "real" coding round)

**Prompt**: process a stream of DNS query strings concurrently — for each query, perform a synthetic DNS lookup and store the result. Bound concurrency to N workers. Support context cancellation. Return all results in submission order.

```go
package main

import (
    "context"
    "fmt"
    "sync"
)

type Query struct {
    ID   int
    Name string
}

type Result struct {
    ID  int
    IP  string
    Err error
}

func lookup(ctx context.Context, name string) (string, error) {
    // pretend to be a resolver
    select {
    case <-ctx.Done():
        return "", ctx.Err()
    default:
    }
    return "10.0.0.1", nil
}

func ProcessQueries(ctx context.Context, queries []Query, workers int) []Result {
    results := make([]Result, len(queries))
    in := make(chan Query)
    var wg sync.WaitGroup
    wg.Add(workers)
    for i := 0; i < workers; i++ {
        go func() {
            defer wg.Done()
            for q := range in {
                ip, err := lookup(ctx, q.Name)
                results[q.ID] = Result{ID: q.ID, IP: ip, Err: err}
            }
        }()
    }
    for i, q := range queries {
        q.ID = i
        select {
        case in <- q:
        case <-ctx.Done():
            close(in)
            wg.Wait()
            return results
        }
    }
    close(in)
    wg.Wait()
    return results
}

func main() {
    qs := []Query{{Name: "a.com"}, {Name: "b.com"}, {Name: "c.com"}}
    out := ProcessQueries(context.Background(), qs, 2)
    for _, r := range out {
        fmt.Printf("%+v\n", r)
    }
}
```

**Follow-ups**:
- Bound latency tail: add per-query timeout via `context.WithTimeout`.
- Backpressure: switch `in` to a buffered channel of size N to absorb bursts; or use a semaphore (`make(chan struct{}, N)`) and `errgroup.Group`.
- Result ordering when results stream out: switch to an output channel + a buffered reorderer keyed by ID.
- Failure handling: retry vs. fail-fast — drive from policy parameter, not hard-coded.

---

## P5 · Find Conflicting Subnets

**Prompt**: given a list of CIDR strings, return any pair that overlaps.

**Approach**: parse each to `(start, end)` integers; sort by start; sweep to find overlaps. O(n log n).

```go
package conflict

import (
    "encoding/binary"
    "net"
    "sort"
)

type r struct {
    cidr     string
    start, end uint32
}

func cidrRange(c string) (r, error) {
    _, ipnet, err := net.ParseCIDR(c)
    if err != nil {
        return r{}, err
    }
    ip := ipnet.IP.To4()
    if ip == nil {
        return r{}, nil
    }
    start := binary.BigEndian.Uint32(ip)
    ones, bits := ipnet.Mask.Size()
    size := uint32(1) << uint32(bits-ones)
    return r{cidr: c, start: start, end: start + size - 1}, nil
}

type Conflict struct{ A, B string }

func FindConflicts(cidrs []string) ([]Conflict, error) {
    rs := make([]r, 0, len(cidrs))
    for _, c := range cidrs {
        rg, err := cidrRange(c)
        if err != nil {
            return nil, err
        }
        rs = append(rs, rg)
    }
    sort.Slice(rs, func(i, j int) bool { return rs[i].start < rs[j].start })
    var out []Conflict
    for i := 1; i < len(rs); i++ {
        if rs[i].start <= rs[i-1].end {
            out = append(out, Conflict{A: rs[i-1].cidr, B: rs[i].cidr})
        }
    }
    return out, nil
}
```

**Follow-ups**:
- IPv6 — use `*big.Int` or `netip.Prefix` (Go 1.18+).
- Find *all* overlapping pairs, not just adjacent — interval tree (O((n+k) log n) where k = output size).
- Incremental: maintain a sorted set under insertions; check overlap on add — O(log n) per insert.

---

## P6 · Build a Sliding-Window Rate Counter

**Prompt**: count number of events per key in the last 60 seconds, sliding.

**Approach**: per key, a ring of 60 one-second buckets, plus a running sum. Compact and fast.

```go
package window

import (
    "sync"
    "time"
)

type counter struct {
    mu      sync.Mutex
    buckets [60]int
    sum     int
    lastSec int64
}

type Counter struct {
    mu sync.RWMutex
    cs map[string]*counter
}

func New() *Counter { return &Counter{cs: make(map[string]*counter)} }

func (c *Counter) get(key string) *counter {
    c.mu.RLock()
    cc, ok := c.cs[key]
    c.mu.RUnlock()
    if ok {
        return cc
    }
    c.mu.Lock()
    defer c.mu.Unlock()
    cc, ok = c.cs[key]
    if !ok {
        cc = &counter{lastSec: time.Now().Unix()}
        c.cs[key] = cc
    }
    return cc
}

func (c *Counter) Add(key string, n int) {
    cc := c.get(key)
    cc.mu.Lock()
    defer cc.mu.Unlock()
    cc.advance(time.Now().Unix())
    cc.buckets[cc.lastSec%60] += n
    cc.sum += n
}

func (c *Counter) Count(key string) int {
    cc := c.get(key)
    cc.mu.Lock()
    defer cc.mu.Unlock()
    cc.advance(time.Now().Unix())
    return cc.sum
}

func (cc *counter) advance(now int64) {
    diff := now - cc.lastSec
    if diff <= 0 {
        return
    }
    if diff >= 60 {
        for i := range cc.buckets {
            cc.buckets[i] = 0
        }
        cc.sum = 0
        cc.lastSec = now
        return
    }
    for i := int64(1); i <= diff; i++ {
        idx := (cc.lastSec + i) % 60
        cc.sum -= cc.buckets[idx]
        cc.buckets[idx] = 0
    }
    cc.lastSec = now
}
```

**Follow-ups**:
- Replace fixed 60 with parameter.
- Memory growth from many keys: TTL eviction; estimate footprint per key.
- Persistence across restarts: not necessary for rate limiting; sliding window can rebuild.

---

## P7 · Detect a Cycle in a Singly-Linked List (the warmup)

Sometimes asked verbatim. Floyd's tortoise-and-hare.

```go
type Node struct {
    Val  int
    Next *Node
}

func HasCycle(head *Node) bool {
    slow, fast := head, head
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
        if slow == fast {
            return true
        }
    }
    return false
}
```

Follow-up: find the *start* of the cycle. After meeting, reset one pointer to head, advance both at the same speed; they meet at the start. (RFC: see "tortoise and hare cycle detection".)

---

## P8 · Top K Most Frequent Domains

**Prompt**: given a stream of domain queries, return the top K most frequent.

**Two regimes**:

1. **Exact, offline**: count with a map, then partial sort / min-heap of size K. O(n) space, O(n log K) time.
2. **Approximate, streaming, bounded memory**: **Count-Min Sketch** + min-heap of size K. Sub-linear memory.

```go
package topk

import "container/heap"

type entry struct {
    name  string
    count int
    idx   int
}

type minHeap []*entry

func (h minHeap) Len() int            { return len(h) }
func (h minHeap) Less(i, j int) bool  { return h[i].count < h[j].count }
func (h minHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i]; h[i].idx = i; h[j].idx = j }
func (h *minHeap) Push(x interface{}) { e := x.(*entry); e.idx = len(*h); *h = append(*h, e) }
func (h *minHeap) Pop() interface{}   { old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x }

func TopK(stream <-chan string, k int) []string {
    counts := make(map[string]*entry)
    h := &minHeap{}
    for s := range stream {
        e, ok := counts[s]
        if ok {
            e.count++
            heap.Fix(h, e.idx)
            continue
        }
        e = &entry{name: s, count: 1}
        counts[s] = e
        if h.Len() < k {
            heap.Push(h, e)
        } else if e.count > (*h)[0].count {
            removed := heap.Pop(h).(*entry)
            delete(counts, removed.name)
            heap.Push(h, e)
        }
    }
    out := make([]string, 0, h.Len())
    for h.Len() > 0 {
        out = append(out, heap.Pop(h).(*entry).name)
    }
    // reverse to descending
    for i, j := 0, len(out)-1; i < j; i, j = i+1, j-1 {
        out[i], out[j] = out[j], out[i]
    }
    return out
}
```

**Follow-up**: explain Count-Min Sketch tradeoff — `O(width × depth)` memory, never under-counts, may over-count by a bounded ε with probability 1-δ.

---

## P9 · Parse and Validate a DHCP Option-82 Packet

**Prompt**: given a `[]byte` containing DHCP options starting at a known offset, find option 82 (sub-options 1=circuit-id, 2=remote-id) and return them.

```go
package dhcp

import "errors"

func ParseOption82(opts []byte) (circuit, remote []byte, err error) {
    i := 0
    for i < len(opts) {
        if opts[i] == 0xff { // end
            break
        }
        if opts[i] == 0 { // pad
            i++
            continue
        }
        if i+1 >= len(opts) {
            return nil, nil, errors.New("truncated option")
        }
        code := opts[i]
        length := int(opts[i+1])
        if i+2+length > len(opts) {
            return nil, nil, errors.New("truncated option value")
        }
        if code == 82 {
            sub := opts[i+2 : i+2+length]
            j := 0
            for j < len(sub) {
                if j+1 >= len(sub) {
                    return nil, nil, errors.New("truncated sub-option")
                }
                sc := sub[j]
                sl := int(sub[j+1])
                if j+2+sl > len(sub) {
                    return nil, nil, errors.New("truncated sub-option value")
                }
                switch sc {
                case 1:
                    circuit = sub[j+2 : j+2+sl]
                case 2:
                    remote = sub[j+2 : j+2+sl]
                }
                j += 2 + sl
            }
            return circuit, remote, nil
        }
        i += 2 + length
    }
    return nil, nil, errors.New("option 82 not present")
}
```

**Follow-up**: hostile input handling (extremely long lengths, infinite loops on zero-length), fuzz testing, returning subslices vs. copies (security: don't hand a caller a reference into a buffer they shouldn't see).

---

## P10 · Build a Naive DNS Resolver Cache with TTL

Combines LRU + TTL + bounded memory.

```go
package resolvercache

import (
    "sync"
    "time"
)

type cacheKey struct {
    name  string
    qtype uint16
}

type cacheVal struct {
    data    []byte
    expires time.Time
}

type Cache struct {
    mu sync.RWMutex
    m  map[cacheKey]cacheVal
}

func New() *Cache { return &Cache{m: make(map[cacheKey]cacheVal)} }

func (c *Cache) Get(name string, qtype uint16) ([]byte, bool) {
    c.mu.RLock()
    v, ok := c.m[cacheKey{name, qtype}]
    c.mu.RUnlock()
    if !ok || time.Now().After(v.expires) {
        return nil, false
    }
    return v.data, true
}

func (c *Cache) Put(name string, qtype uint16, data []byte, ttl time.Duration) {
    c.mu.Lock()
    c.m[cacheKey{name, qtype}] = cacheVal{data: data, expires: time.Now().Add(ttl)}
    c.mu.Unlock()
}

// Janitor evicts expired entries; call periodically.
func (c *Cache) GC(now time.Time) {
    c.mu.Lock()
    for k, v := range c.m {
        if now.After(v.expires) {
            delete(c.m, k)
        }
    }
    c.mu.Unlock()
}
```

**Follow-ups**:
- Bound by memory; add LRU.
- Negative caching: cache NXDOMAIN with its own TTL bounds (RFC 2308 — min of SOA MINIMUM and SOA TTL).
- Prefetch: refresh hot entries before TTL expiry.
- Sharded for concurrency.
- Cache stampede on miss → singleflight to deduplicate concurrent lookups for the same name.

---

## Tips for the live round

- **Talk through your model** before writing code. Two minutes of design saves twenty minutes of fixing.
- **Name your invariants** out loud. "I'm going to keep the head sentinel always pointing to the most-recently-used."
- **Use `golang.org/x/sync/singleflight`** and `errgroup` when relevant — it shows you've written real concurrent Go.
- **Don't over-engineer**. Solve the stated problem first; the follow-ups are where you flex.
- **Handle the cancellation context** in any I/O-bound problem. Forgetting it is a tell.
- **Acknowledge what you'd test**. "I'd add table tests covering: cache hit, cache miss, TTL expiry, concurrent Get/Put, eviction under pressure."

---

## Sources

- [Glassdoor — Infoblox Sr Software Engineer Interview Questions](https://www.glassdoor.ca/Interview/Infoblox-Sr-Software-Engineer-Interview-Questions-EI_IE35108.0,8_KO9,29.htm)
- [Interview Query — Infoblox Software Engineer Guide](https://www.interviewquery.com/interview-guides/infoblox-software-engineer)
- [GeeksForGeeks — Infoblox Interview Experience](https://www.geeksforgeeks.org/interview-experiences/infoblox-interview-experience-on-campus/)