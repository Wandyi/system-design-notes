# 12 · Coding Problems — Go-Flavored, Streaming-Themed

Coding rounds at streaming companies are usually classic DSA plus one systems-flavored problem (rate limiter, sharded cache, sliding-window QoE counter, manifest parsing). Same pattern as Infoblox: solve the stated problem first, then handle the staff-level follow-ups.

All examples in Go.

## P1 · Sliding-Window QoE Counter (rebuffer ratio in last minute)

**Prompt**: report the % of users in the last 60s who experienced a rebuffer event. Updates per second.

Approach: per-user state with timestamps + global rolling counters.

```go
package qoe

import (
    "sync"
    "time"
)

type bucket struct {
    sec       int64
    total     int
    rebuffer  int
}

type RebufferRatio struct {
    mu    sync.Mutex
    ring  [60]bucket
    head  int64 // current second
}

func New() *RebufferRatio { return &RebufferRatio{head: time.Now().Unix()} }

func (r *RebufferRatio) advance(now int64) {
    diff := now - r.head
    if diff <= 0 {
        return
    }
    if diff >= 60 {
        for i := range r.ring {
            r.ring[i] = bucket{}
        }
    } else {
        for i := int64(1); i <= diff; i++ {
            r.ring[(r.head+i)%60] = bucket{sec: r.head + i}
        }
    }
    r.head = now
}

func (r *RebufferRatio) RecordSession(rebuffered bool) {
    now := time.Now().Unix()
    r.mu.Lock()
    defer r.mu.Unlock()
    r.advance(now)
    idx := now % 60
    r.ring[idx].total++
    if rebuffered {
        r.ring[idx].rebuffer++
    }
}

func (r *RebufferRatio) Ratio() float64 {
    now := time.Now().Unix()
    r.mu.Lock()
    defer r.mu.Unlock()
    r.advance(now)
    total, reb := 0, 0
    for i := range r.ring {
        total += r.ring[i].total
        reb += r.ring[i].rebuffer
    }
    if total == 0 {
        return 0
    }
    return float64(reb) / float64(total)
}
```

**Follow-ups**:
- Per-region buckets — promote to `map[region]*RebufferRatio` keyed by region.
- High-cardinality (millions of regions) — shard by `hash(region) % N`.
- Lock contention — RWMutex isn't enough since `Ratio` advances; consider per-region locks.
- Persistence on crash — not needed; metrics are recoverable from the next minute.

---

## P2 · Parse an HLS Master Playlist

**Prompt**: parse an HLS `.m3u8` master playlist and return the list of `(bandwidth, resolution, uri)` tuples.

```go
package hls

import (
    "bufio"
    "errors"
    "fmt"
    "io"
    "strconv"
    "strings"
)

type Variant struct {
    Bandwidth  int
    Resolution string
    Codecs     string
    URI        string
}

func ParseMaster(r io.Reader) ([]Variant, error) {
    sc := bufio.NewScanner(r)
    sc.Buffer(make([]byte, 1024*1024), 1024*1024)
    var out []Variant
    var pending *Variant
    if sc.Scan() {
        if strings.TrimSpace(sc.Text()) != "#EXTM3U" {
            return nil, errors.New("missing #EXTM3U")
        }
    }
    for sc.Scan() {
        line := strings.TrimSpace(sc.Text())
        if line == "" {
            continue
        }
        if strings.HasPrefix(line, "#EXT-X-STREAM-INF:") {
            attrs := line[len("#EXT-X-STREAM-INF:"):]
            v, err := parseAttrs(attrs)
            if err != nil {
                return nil, err
            }
            pending = v
            continue
        }
        if strings.HasPrefix(line, "#") {
            continue
        }
        if pending != nil {
            pending.URI = line
            out = append(out, *pending)
            pending = nil
        }
    }
    if err := sc.Err(); err != nil {
        return nil, err
    }
    return out, nil
}

func parseAttrs(s string) (*Variant, error) {
    v := &Variant{}
    for _, kv := range splitAttrs(s) {
        eq := strings.IndexByte(kv, '=')
        if eq < 0 {
            continue
        }
        key, val := kv[:eq], strings.Trim(kv[eq+1:], `"`)
        switch key {
        case "BANDWIDTH":
            b, err := strconv.Atoi(val)
            if err != nil {
                return nil, fmt.Errorf("bandwidth: %w", err)
            }
            v.Bandwidth = b
        case "RESOLUTION":
            v.Resolution = val
        case "CODECS":
            v.Codecs = val
        }
    }
    return v, nil
}

// splitAttrs handles comma separation but respects quoted values
func splitAttrs(s string) []string {
    var out []string
    var sb strings.Builder
    inQuote := false
    for i := 0; i < len(s); i++ {
        c := s[i]
        if c == '"' {
            inQuote = !inQuote
            sb.WriteByte(c)
            continue
        }
        if c == ',' && !inQuote {
            out = append(out, sb.String())
            sb.Reset()
            continue
        }
        sb.WriteByte(c)
    }
    if sb.Len() > 0 {
        out = append(out, sb.String())
    }
    return out
}
```

**Follow-ups**:
- Handle `EXT-X-MEDIA`, `EXT-X-I-FRAME-STREAM-INF`, `EXT-X-SESSION-KEY` for full coverage.
- Resolve relative URIs against the playlist's base URL.
- Stream-parse a multi-MB playlist without buffering all of it.

---

## P3 · ABR-Picker (buffer-aware throughput-bounded)

**Prompt**: given a list of available bitrates, a current buffer level, and a throughput estimate, return the rendition to pick.

```go
package abr

import "sort"

type Rendition struct {
    Bitrate int // bps
}

type ABR struct {
    BufferLow  float64 // sec, below which we drop hard
    BufferHigh float64 // sec, above which we go top
    Safety     float64 // throughput multiplier, e.g., 0.85
}

func New() *ABR { return &ABR{BufferLow: 4, BufferHigh: 30, Safety: 0.85} }

// Pick returns the chosen rendition index.
func (a *ABR) Pick(rends []Rendition, bufferSec, throughputBps float64) int {
    if len(rends) == 0 {
        return -1
    }
    sort.SliceStable(rends, func(i, j int) bool { return rends[i].Bitrate < rends[j].Bitrate })
    // Buffer-based extremes
    if bufferSec <= a.BufferLow {
        return 0
    }
    if bufferSec >= a.BufferHigh {
        return len(rends) - 1
    }
    // Throughput-bound pick
    cap := throughputBps * a.Safety
    pick := 0
    for i, r := range rends {
        if float64(r.Bitrate) <= cap {
            pick = i
        }
    }
    // Buffer-mediated tilt: scale pick by buffer fill
    fraction := (bufferSec - a.BufferLow) / (a.BufferHigh - a.BufferLow)
    bufferPick := int(fraction * float64(len(rends)-1))
    if bufferPick > pick {
        pick = bufferPick
    }
    return pick
}
```

**Follow-ups**:
- Hysteresis — require sustained excess before switching up.
- Lookahead — accept a future-segment-size estimate; predict buffer after download.
- BOLA-style — implement Lyapunov-based selection.
- Live-edge constraint — penalize selections that would push the player off live.

---

## P4 · Shard a Hot Cache for Concurrent Access

**Prompt**: implement a sharded `Get`/`Put` cache to reduce lock contention.

```go
package shardedcache

import (
    "hash/fnv"
    "sync"
)

type shard[V any] struct {
    mu sync.RWMutex
    m  map[string]V
}

type Cache[V any] struct {
    shards []*shard[V]
    mask   uint32
}

func New[V any](shardCount int) *Cache[V] {
    if shardCount&(shardCount-1) != 0 {
        panic("shardCount must be power of two")
    }
    s := make([]*shard[V], shardCount)
    for i := range s {
        s[i] = &shard[V]{m: make(map[string]V)}
    }
    return &Cache[V]{shards: s, mask: uint32(shardCount - 1)}
}

func (c *Cache[V]) shard(key string) *shard[V] {
    h := fnv.New32a()
    h.Write([]byte(key))
    return c.shards[h.Sum32()&c.mask]
}

func (c *Cache[V]) Get(key string) (V, bool) {
    sh := c.shard(key)
    sh.mu.RLock()
    defer sh.mu.RUnlock()
    v, ok := sh.m[key]
    return v, ok
}

func (c *Cache[V]) Put(key string, val V) {
    sh := c.shard(key)
    sh.mu.Lock()
    defer sh.mu.Unlock()
    sh.m[key] = val
}
```

**Follow-ups**:
- TTL — augment `shard` with expirations and a janitor.
- LRU per shard — bounded memory.
- Atomic snapshot for reading — copy-on-write or epoch-based.
- Lock-free reads via `sync.Map` — discuss when it wins (read-mostly, large keyspace) and when it loses.

---

## P5 · Token-Bucket Rate Limiter per Client IP

**Prompt**: implement a per-IP rate limiter for `Allow(ip)` returning true if a token is available. (Same as Infoblox version; refer to that file. The streaming-specific use is at the beacon endpoint or manifest endpoint.)

---

## P6 · Distributed-Style Sliding Window via Redis (sketch)

**Prompt**: rate-limit by a sliding window stored in Redis (so all instances share state).

Sketch:
- Key: `rl:{user}:{minute}`.
- Operation: `INCR` and `EXPIRE` atomically via Lua.
- Aggregate over current + previous bucket weighted by overlap fraction.

```lua
local current_key = KEYS[1]
local previous_key = KEYS[2]
local limit = tonumber(ARGV[1])
local now_ms = tonumber(ARGV[2])
local window_ms = tonumber(ARGV[3])

local cur = tonumber(redis.call('GET', current_key) or '0')
local prev = tonumber(redis.call('GET', previous_key) or '0')
local elapsed = now_ms % window_ms
local prev_weight = (window_ms - elapsed) / window_ms
local count = cur + math.floor(prev * prev_weight)
if count >= limit then
  return 0
end
redis.call('INCR', current_key)
redis.call('PEXPIRE', current_key, window_ms * 2)
return 1
```

**Follow-ups**:
- Use `golang.org/x/time/rate` for in-process limits; Redis only for cross-process correctness.
- Eventual-only consistency under Redis partition — accept some over-allowance under split-brain.

---

## P7 · Bounded WebSocket Fanout (sketch)

**Prompt**: implement a hub that maintains N WebSocket connections and broadcasts a message to all of them with bounded per-connection buffers (drop oldest on overflow).

```go
package wshub

import "sync"

type Client struct {
    Send chan []byte
}

type Hub struct {
    mu      sync.RWMutex
    clients map[*Client]struct{}
}

func NewHub() *Hub { return &Hub{clients: make(map[*Client]struct{})} }

func (h *Hub) Register(c *Client) {
    h.mu.Lock()
    defer h.mu.Unlock()
    h.clients[c] = struct{}{}
}

func (h *Hub) Unregister(c *Client) {
    h.mu.Lock()
    defer h.mu.Unlock()
    delete(h.clients, c)
    close(c.Send)
}

func (h *Hub) Broadcast(msg []byte) {
    h.mu.RLock()
    defer h.mu.RUnlock()
    for c := range h.clients {
        select {
        case c.Send <- msg:
        default:
            // drop or evict oldest from c.Send
        }
    }
}
```

**Follow-ups**:
- Per-room sharding so a 20M-conn Hub is broken into 1M-conn shards.
- Compression of broadcast (permessage-deflate).
- Backpressure: instead of drop, send a control message to slow consumer telling it to skip ahead.

---

## P8 · Top-K Most-Watched Videos in a Stream

**Prompt**: stream of `(video_id, seconds_watched)` events; return the top K videos by total seconds. (Same as Infoblox topK; reuse heap + map approach. The streaming twist: weights instead of counts.)

---

## P9 · Implement an LRU Cache for a Player Manifest Cache

(Same as Infoblox LRU. The streaming-specific follow-up: TTL aligned to manifest's own TTL header.)

---

## P10 · Find the Longest Substring That Forms a Valid HLS Discontinuity Boundary

(A more contrived problem; mostly DSA — sliding window with a constraint check. Mention in case the interviewer asks something string-heavy with a streaming framing.)

---

## Patterns to internalize

- **Sharded locking** for hot caches.
- **Sliding-window counters** with ring buffers.
- **Bounded channels + drop policy** for backpressure.
- **Lua scripts in Redis** for atomic cross-process state.
- **Atomic pointer swap** for read-mostly state.
- **errgroup + semaphore** for bounded concurrency.
- **Context cancellation** in every I/O path.
- **Table-driven tests** in your write-up.

---

## Sources

- [Go concurrency patterns — golang.org](https://go.dev/blog/pipelines)
- [HLS spec RFC 8216](https://datatracker.ietf.org/doc/html/rfc8216)
- [golang.org/x/time/rate](https://pkg.go.dev/golang.org/x/time/rate)
