# Real-Time State Broadcast at Scale — Presence Fan-Out

How to broadcast state changes to thousands (or millions) of subscribers simultaneously without melting under memory pressure, thread exhaustion, and lock contention. 
Anchored on the canonical case — **a Discord-style guild presence update** — but the patterns generalize to Slack workspaces, MMO zones, stock tickers, sports score feeds, IoT fleet telemetry, and collaborative document cursors.

The problem looks simple: "user X went online, tell everyone in the guild." It is not. At guild sizes of 1 M+ members, with 100 K state-change events per second across the platform, a naive fan-out collapses in three predictable ways. The job is to design a system where each of those collapse modes is structurally impossible, not just unlikely.

---

## 1. The Problem, Concretely

### 1.1 The shape

- Tens of thousands of **gateway** WebSocket connections per host (with sane TLS termination).
- Hundreds of millions of concurrent users platform-wide; tens of millions online at peak.
- A **guild / channel / room** is a subscription scope; users belong to many.
- **State changes** = presence (online/offline/away/dnd), typing indicators, voice-channel activity, message-receipts, custom-status updates. Different rate profiles, same fan-out shape.
- Largest guilds have **millions of members**. Most are *not actively viewing* the guild at any given moment — a critical property we will exploit.
- The fast path: a state change must reach a viewing subscriber **within ~150 ms p99**, or it doesn't feel real-time.

### 1.2 The naive design and exactly how it dies

```python
def on_presence_change(user_id, new_state):
    for guild in user.guilds:
        for member in guild.members:                # could be 1 000 000
            socket = socket_for(member)
            socket.send(serialize(new_state))       # blocking I/O? new thread?
```

This collapses three ways under load:

**(a) Memory pressure.** A 1 KB message broadcast to 1 M members allocates 1 GB if you serialize per-recipient. 
Multiply by 100 K presence changes/sec and you're at 100 TB/sec of allocation churn — your young-gen GC dies first, the kernel's slab allocator next.

**(b) Thread / fiber exhaustion.** "Spawn one task per recipient" sounds clean. At 1 M tasks per change × 100 K changes/sec, you've created 10¹¹ tasks/sec. 
Your scheduler is now a fan, and not the cooling kind.

**(c) Lock contention.** The guild's member list is read by every fan-out and written every time someone joins/leaves. 
A `RWMutex` looks fine until 1 M readers all wake on a presence event and trample each other. The lock itself becomes the bottleneck before the network does.

Two more failure modes appear at scale:

**(d) Slow-consumer blocking.** One subscriber on a 3G connection has its TCP send buffer full. 
The naive design either blocks the fan-out on it (head-of-line blocking) or unbounded-buffers messages until OOM.

**(e) Thundering reconnects.** A single gateway host restarts; 50 000 clients reconnect in 5 seconds, each demanding initial state for hundreds of guilds. The control plane melts.

Every pattern below targets one or more of these explicitly. The thesis: **fan-out cost must be amortized somewhere — across gateways, across messages, across time, across viewers — and never paid per-(message × recipient) on a single host.**

---

## 2. Architecture — Two-Tier with Explicit Fan-Out Boundary

```
   Clients (WebSocket / WebTransport)
              │
              ▼
   ┌──────────────────────────┐
   │   Gateway Tier           │   stateful, holds N connections per host
   │   (TLS, WS, auth,        │   one process per CPU core, async I/O
   │    per-conn bounded q)   │
   └──────────┬───────────────┘
              │ subscribe / publish
              ▼
   ┌──────────────────────────┐
   │   Fan-out / Pub/Sub      │   scope = guild | channel | room | user
   │   (custom on top of      │   in-memory routing, sharded by scope-id
   │    Kafka/NATS/Redis or   │
   │    Erlang-style mesh)    │
   └──────────┬───────────────┘
              │ produce
              ▼
   ┌──────────────────────────┐
   │   Application Services   │   presence svc, chat svc, voice svc
   │   (state owners,         │   produce events; never address recipients directly
   │    business logic)       │
   └──────────┬───────────────┘
              │
              ▼
   State stores (Cassandra, Redis, Postgres) — only for cold/persistent state
```

The most important architectural property: **the application service emits one logical event** ("user 123's presence is now `idle`") and is done. It does not enumerate recipients. The fan-out tier owns the fact that *some scope* is interested. The gateway tier owns the fact that *some connection* is in that scope.

The recipient enumeration is split across three places, each of which sees a small fraction of the work — the central insight that makes the architecture survive.

---

## 3. The Fan-Out Algorithm — Push to Gateways, Then Locally to Sockets

The core algorithm — anchored in how Discord, Slack, and Twitch all converged on the same pattern:

### 3.1 Per-scope subscription on the gateway

Each gateway host maintains, in-process, a map:
```
scope_id  →  set<connection_id>
```
For a gateway holding 50 000 connections, where each connection subscribes to ~50 scopes (guilds), this is ~2.5 M entries — fine for a single process.

When a client opens a connection and subscribes to scope `g`, the gateway:
1. Adds `conn_id` to its local `scope_id → connections` map.
2. **Tells the fan-out tier** "this gateway is interested in scope g" — but only if it wasn't already subscribed by some other connection on the same host.

That second step is the optimization that makes the whole thing work: **the fan-out tier sees one subscriber per (scope, gateway), not one per (scope, user)**. A guild with 1 M members spread across 200 gateway hosts has *200* subscribers in the fan-out tier — not 1 M.

### 3.2 Publish path

When the presence service emits `PresenceChanged{user: 123, scope: g, state: idle}`:

```
fan_out_tier.publish(scope=g, payload):
    for gateway in subscribers_of(g):       # 200, not 1M
        gateway.send(payload)               # one network hop per gateway

gateway.on_receive(payload, scope):
    serialize_once(payload)                  # one allocation
    for conn in local_scope_map[scope]:      # 5000 of the 50000 total conns
        conn.outbound_queue.try_push(buffer) # bounded; drop on overflow
```

Two cost reductions stacked:
- **Cross-host fanout is O(gateways)** — not O(members).
- **Within-host fanout is local memcpy** — not network sends; one serialized buffer is shared across all connections on the same host.

### 3.3 Why this beats Redis Pub/Sub naively

Naive Redis Pub/Sub: `PUBLISH guild-g {payload}` → Redis dispatches to every subscriber. If every subscriber is a single connection, you're back to O(members). The fix is the same in any system: **gateways subscribe**, not connections. Whether the substrate is Redis, NATS, Kafka, or a custom gossip mesh is operational, not architectural.

---

## 4. Per-Connection Bounded Queues — How We Defeat Slow Consumers

Each WebSocket connection has its own outbound queue:

```
struct Connection {
    socket:        async_socket,
    outbound:      bounded_ring_buffer<Frame>,   // e.g. 256 slots
    drop_policy:   DropPolicy,                   // DropOldest | CoalesceState | Disconnect
    last_sent_state_per_user: HashMap<user_id, state_seq>,  // for coalescing
}
```

The fan-out hot path **never blocks**:
```
try_push(frame):
    if queue.full():
        apply_drop_policy()         // never block, never grow queue
    else:
        queue.push(frame)
```

A dedicated writer task per connection drains the queue to the socket asynchronously. If the socket's write blocks, only that one writer task is parked — the fan-out hot path moved on long ago.

### Drop policies (chosen per message type)

- **Presence updates** → coalesce by `(scope, user_id)`: a queued presence event for user 123 is replaced by the newer one. Subscribers care about the *current* state, not the history of transitions. **This single optimization eliminates 80%+ of memory pressure during presence storms.**
- **Typing indicators** → drop oldest unconditionally. Latency-sensitive, easily expired.
- **Chat messages** → must not drop. If queue full, the connection is *dead* — start the disconnect protocol; client will reconnect and resync from a sequence cursor (§ 8).
- **Voice signaling** → urgent, never drop, but bounded queue still applies — if overflow, terminate the call gracefully.

The crucial invariant: **a slow client cannot affect any other client.** Backpressure is per-connection. The fan-out side never waits.

---

## 5. Defeating Memory Pressure — Serialize Once, Share Buffer

### 5.1 Single allocation per fan-out

Naively, "send to 5 000 connections" allocates 5 000 buffers. With reference-counted byte buffers (Netty's `ByteBuf`, Tokio's `Bytes`, Erlang's binaries), it's:

```
buf = Arc<Bytes>(serialize(payload))  // one allocation
for conn in conns:
    conn.try_push(buf.clone())        // refcount bump, no copy
```

Connections share the immutable buffer; the kernel writes the same bytes from each socket. Allocation: **O(1)** per fan-out, regardless of subscriber count.

For the cross-host hop, the gateway receives one network message and constructs the buffer once. Within the host, fan-out is pointer arithmetic.

### 5.2 Object pooling for hot types

Pool the wrapper structs (`Frame`, `Notification`, queue nodes). Sustained allocation rate of these in the steady state is high; making them GC-pressure-free pays off on JVM/CLR/Go runtimes. On Rust/C++ the allocator already handles this well; pooling is usually unnecessary.

### 5.3 Off-heap and zero-copy

For the very hottest paths, use `sendfile`-style or io_uring's zero-copy where the socket layer can write the buffer directly without copying through user space. Diminishing returns past the basic "serialize once" — measure first.

---

## 6. Defeating Lock Contention — Lock-Free / Sharded Subscription Registries

The `scope → connections` map is read on every fan-out (very hot) and written on every subscribe/unsubscribe (much cooler). Naive `RwLock<HashMap>` works until 1 M readers, then the lock cacheline becomes the bottleneck.

### Three options, ordered by complexity

**(a) Sharded map** — split the map into N shards by `hash(scope_id) % N`, each with its own lock. Concurrent fan-outs to different scopes are independent. N = 64 to 256. Defeats most contention; trivial to implement. The "good enough" answer.

**(b) Copy-on-write** — replace `Arc<HashMap>` atomically on writes; readers see a stable snapshot. Reads are lock-free pointer-load + iteration. Writes allocate a new map (not necessarily the whole thing — persistent data structures help). Best for very-high-read-low-write workloads — exactly fan-out.

**(c) Per-scope sharded actor / Erlang process** — one lightweight process per active scope. Subscribe / unsubscribe / publish all go through the process's mailbox; no locks, no shared mutability. Erlang/OTP's GenServer or Akka's Actors do this natively. The pattern Discord uses extensively.

### Within a connection, no locks at all

The bounded queue is single-producer-single-consumer (fan-out task pushes, writer task pops) — use a wait-free SPSC ring buffer (Aeron, the LMAX Disruptor pattern). No CAS, no syscalls, just sequence-counter arithmetic. ~10 ns per enqueue.

---

## 7. Subscription Pruning — Don't Broadcast to Lurkers

The hidden lever in large-guild fan-out: **most members are not viewing the guild right now.** Sending presence updates to them is pure waste.

### The pattern

- Default subscription state for a member is "not viewing." They're *in* the guild but receive nothing on the fast path.
- When the user navigates to the guild in the UI, the client sends `SubscribeToScope(g)`; gateway adds the connection to local map. Initial-state burst is bounded (§ 8).
- When the user navigates away (or the tab backgrounds, or the screen sleeps), client sends `UnsubscribeFromScope(g)` and stops receiving.
- Critical state (e.g. you got @-mentioned) still routes to the user via their **personal subscription** (`scope = user:123`) which is always on while connected.

### Active fraction

In a guild of 1 M members:
- ~5 % might be online (= 50 K).
- Of those, ~1–5 % are *currently viewing this specific guild* (= 500 – 2 500).
- That's the actual fan-out cost: **2 500**, not 1 000 000. **Three orders of magnitude reduction.**

### Bounded-cost guarantee

We can promise: per-presence-change fanout cost is bounded by the number of *active viewers* of the affected scope, not by total membership. This is the unlock that makes million-member guilds tractable.

---

## 8. Coalescing and Rate Limiting — Time-Domain Amortization

### 8.1 Server-side coalescing

A user toggles presence five times in a second (laptop opens / closes lid / reconnects). Naive: 5 broadcasts × 2 500 viewers = 12 500 messages.

Better: coalesce in a 100 ms tumbling window per `user_id`. The presence service holds a small per-user buffer:

```
on_presence_event(user_id, new_state):
    buffer[user_id] = new_state          // overwrite
    schedule_flush(user_id, in=100ms)    // idempotent

on_flush(user_id):
    state = buffer.remove(user_id)
    publish PresenceChanged{user_id, state}
```

Throughput drops by an order of magnitude during flap storms; user-perceived latency stays under 100 ms.

### 8.2 Per-recipient rate limiting

For typing indicators ("user X is typing…" emitted every 5 s by the client), the gateway *also* coalesces on the egress side: don't send a typing event to a connection if the same `(scope, user)` typing event was sent in the last 4 s. Saves bandwidth and CPU on mobile clients.

### 8.3 Batching across messages

The connection writer task reads up to N frames from the queue and writes them in one syscall (vectored I/O / `writev`). Cuts syscalls by 10–20× under load. The kernel and TCP do the rest.

---

## 9. The Gateway Tier — Connection Density and Topology

### 9.1 Connection density

A modern Linux box with TLS termination on userspace (e.g. `rustls`, `BoringSSL`) and async I/O can hold **500 K – 1 M** WebSocket connections at < 50 % CPU, given:
- File descriptor limit raised (`ulimit -n` to 2 M).
- TCP keepalives tuned (idle 60 s, probes 3, interval 10 s).
- Per-conn memory ~8 KB (app state + small kernel buffers).
- TLS session resumption enabled.

Discord publicly documented holding 2.6 M concurrent sockets per host on commodity hardware.

### 9.2 Connection placement — consistent hashing of users

Map each user (or session) to a gateway via consistent hashing on `user_id`. Properties:
- Adding/removing a gateway only displaces ~`1/N` of connections.
- Stable session-server affinity → state cached on the right host.
- A user's reconnects come back to the same gateway (sticky), so per-user cached state (subscription set, last-sent cursors) survives reconnect cycles.

Trade-off: skew. A celebrity user with millions of followers — but in this design, *their* connection isn't the hot key; the **scope** is. The gateways subscribed to the celebrity's user-scope can be many; if the load on a *single* gateway is the issue, see §10.

### 9.3 Stickiness vs failover

Sticky connections via consistent hashing risk hotspots if the hash is unlucky. Mitigate with **bounded-load consistent hashing** — connections fall to the next gateway if the natural target is over-capacity. Or, simpler: **L4 load-balance to gateways with rendezvous hashing on user_id**, and let the gateway lazily fetch any user state it doesn't have.

---

## 10. The Celebrity Scope — Hierarchical Fan-Out Trees

For the rare scope with millions of *active* viewers (e.g. a global announcement channel during an event, or a stock ticker for SPY during open), the basic two-tier still has a fan-out fan-in problem: hundreds of gateways subscribed to one scope means the fan-out tier sends hundreds of messages per event.

**Hierarchical fan-out tree**:

```
Producer →  Root broadcaster  (1)
              ├─ Tier-1 broadcaster (16)
              │   ├─ Tier-2 broadcaster (256)
              │   │   ├─ Gateway (4096)
              │   │   └─ Gateway
              │   └─ ...
              └─ ...
```

Each level fans out to a fixed branching factor (e.g. 16). Total levels = `log_16(N)`. For 4 096 gateways, 3 hops. Each broadcaster does at most 16 sends per event — no host is the single point of fan-out.

For ultra-low-latency cases (financial market data), use **PIM-SSM IP multicast** within the data center — one network packet reaches every receiver. Rare in cloud (multicast is usually disabled), but available in some on-prem and bare-metal.

---

## 11. Initial-State Sync and the Reconnect Storm

The other production problem nobody mentions in tutorials: **clients connecting / reconnecting**.

### 11.1 The problem

A user opens the app: needs the current state of all visible scopes (online members, recent messages, presence per friend, …). At ~50 scopes × ~1 KB each, that's 50 KB *per client*. A gateway holding 500 K connections that all reconnect simultaneously (because the host before it was rebooted) needs to push 25 GB of initial state in a few seconds. That's 200 Gbps if it has to.

### 11.2 Mitigations stacked

**(a) Sequence cursors.** Every event stream has a monotonic seq. Clients persist the last seq they saw per scope. On reconnect: `Resume{seq: N}` — server replies with events since N + small heartbeat, not full state. Recovers to live tail in a single round trip if recent enough.

**(b) Ring buffer of recent events per scope.** Server keeps last ~10 minutes of events in memory per scope. If client's cursor falls within the buffer, resume; otherwise, `IdentifyFresh` and full snapshot. Most reconnects are < 10 minutes (transient network blip) → no snapshot needed.

**(c) Lazy state hydration.** Don't send the snapshot for every guild on connect — send it on `SubscribeToScope`. The client only loads the visible UI; everything else is dormant until it's looked at.

**(d) Reconnect jitter.** When a gateway dies, clients reconnect with random jitter (`backoff = base * 2^attempt + random(0, base * 2^attempt)`) to avoid the herd. Server-issued reconnect signals can specify a target window for staggered return.

**(e) Connection draining for planned restarts.** Gateway restarts are graceful: tell each client `Reconnect{after: random(5s, 60s), to: <other-gateway>}` and wait for them to leave. Zero thundering herd.

---

## 12. Wire Protocol — Binary, Framed, and Skinnier Than JSON

For the hot path, a binary protocol matters:

- **Protobuf or FlatBuffers** for payloads — 5–10× smaller than equivalent JSON, 10× faster decode. FlatBuffers wins on decode-laziness (you only parse fields you read).
- **MessagePack / CBOR** as the JSON-shaped middle ground — easier migration from existing JSON systems.
- **Frame-level compression** (per-message Zstd or per-stream permessage-deflate) — 50 %+ savings on text-heavy payloads. Pay CPU for bandwidth and battery; on mobile, often a win.
- **WebSockets** for browser compatibility; **WebTransport** (over QUIC) where supported — multiple streams over one connection (no head-of-line blocking between scopes), 0-RTT resumption, no TCP slow-start on reconnect.

**Frame structure** with explicit sequence numbers and scope IDs lets clients deduplicate, reorder defensively, and detect gaps without parsing payloads:

```
[ frame_seq:uint64 | scope_id:uint64 | type:uint8 | flags:uint8 | payload_len:uint32 | payload:bytes ]
```

---

## 13. Observability — What You Actually Watch

| Metric | Why |
|---|---|
| **Per-connection queue depth (p99, p99.9)** | Slow consumers; if rising, drop policy is engaging |
| **Per-scope active-viewer count** | Detects accidental "broadcast to lurkers" regressions |
| **Fan-out tier subscribers per scope** | Should be ≈ number of distinct gateways with viewers — not number of viewers |
| **Coalesce-ratio per message type** | Validates §8; should be > 0.5 in steady state for presence |
| **Drop-rate per drop-policy class** | Coalesce-drops should be ≫ disconnect-drops |
| **Gateway egress Gbps & syscall rate** | Detects writev batching regressions |
| **Reconnect rate, init-state bytes/sec** | Catch reconnect storms early |
| **End-to-end latency (event-emitted → frame-acked)** | The single user-perceived SLO |
| **CPU-bound vs IO-bound profile per gateway** | TLS CPU vs scheduling overhead — different fixes |

---

## 14. Failure Modes and Their Specific Cures

| Failure | Cure |
|---|---|
| **Gateway OOM from per-conn queue growth** | Bounded queues, drop policy, OOM-watcher disconnects worst offenders before crash |
| **Pub/Sub partition lag (Kafka)** | Per-scope partition; if scope is hot, sub-shard by `(scope, member-bucket)` |
| **Slow-consumer head-of-line** | Per-connection writer task; never share a writer across connections |
| **Lock contention spike during burst** | Sharded subscription map; copy-on-write for read-heavy paths |
| **Hot scope** ("everyone joining the same channel") | Hierarchical fan-out tree; horizontal scale of fan-out broadcasters |
| **TLS handshake CPU storm on restart** | TLS session tickets + 0-RTT resumption; reconnect jitter |
| **Cross-region split** | Each region runs an independent fan-out plane; cross-region only for messages explicitly addressed across regions; presence is region-local for active viewers |
| **Broadcast-to-deaf-client** (NAT timeout, half-open TCP) | TCP keepalives + app-layer ping every 30 s; if no pong in 90 s, force-close |
| **Subscription leak** (scope not unsubscribed on disconnect) | On socket close, gateway emits a single bulk-unsubscribe for all that conn's scopes; tested under chaos |
| **Cascading restart** (one gateway dies, herd hits next, etc.) | Pre-warmed standby capacity; rate-limit `Identify` per-second per-gateway; clients honor `Reconnect{after: jitter}` |
| **Slow producer of state events** (presence service lag) | Producer backpressure; consumers see staleness rather than reorder; explicit "presence may be stale" UI when lag breaches threshold |

---

## 15. Trade-Offs and Alternatives

| Decision | Alternative | Why we picked this |
|---|---|---|
| **Gateway-as-subscriber, not user-as-subscriber** | Each connection subscribes individually | O(gateways) vs O(users); only viable design at million-member scale |
| **Bounded queues with drop policy** | Unbounded buffers + flow control | Unbounded → OOM under partition; drop-with-coalesce keeps latency bounded and state correct |
| **Coalesce presence updates** | Send every transition | 10× throughput reduction; consumers want current state, not history |
| **Subscription pruning (lazy hydrate)** | Push everything to all members | Active-viewer count is 1000× smaller than membership; the only path to million-member-guild correctness |
| **Hierarchical fan-out for celebrity scopes** | Wide fan-out from one broadcaster | Single broadcaster CPU/NIC tops out at ~100 K subs; tree gives unbounded scale |
| **Sticky connection via consistent hash** | Random LB | Cached per-user gateway state survives reconnect; trades hot-spot risk for warmth |
| **WebSocket (optionally WebTransport)** | Long-poll, SSE | Bidirectional; lower per-message overhead; SSE fine for read-only fan-out |
| **Protobuf/FlatBuffers** | JSON | 5–10× smaller, 10× faster decode; mobile battery wins |
| **In-memory ring of recent events for resume** | Persistent log replay | Recent buffer covers > 95 % of reconnects; persistent log for the rest |
| **Erlang/Elixir or Rust async** | JVM or Go | Erlang's per-process isolation suits actor model perfectly; Rust async is the no-GC alternative; Go works if you accept GC pressure tuning. JVM works but pool aggressively |
| **Gateway-side client identity** | Auth at every fan-out | Auth once at handshake, attach principal to connection, every subsequent op is O(1) |
| **Per-scope sharded actor process** | Single shared map + locks | Scales linearly with active scopes; isolated failure domain per actor |

---

## 16. Putting It All Together — The Hot-Path Walkthrough

A user changes presence to `idle`. Trace the path:

1. **Client → gateway**: `PresenceUpdate{state: idle}` over WebSocket. Gateway authenticates via attached principal (no fresh auth call), validates scope, hands off to presence service via internal gRPC.
2. **Presence service**: applies coalesce window (100 ms). On flush, emits one event `PresenceChanged{user: 123, scope_set: {g1, g2, g3, …}}` to the pub/sub layer.
3. **Pub/sub layer** (one publish per scope the user belongs to, but partitioned across N brokers — so the work parallelizes): each scope's partition delivers to all gateways subscribed to that scope. **Gateways**, not users.
4. **Each receiving gateway**: serializes the payload **once** into a refcounted buffer; iterates its local `scope → conns` map (sharded to avoid contention); for each conn, `try_push(buffer.clone())` into the per-conn bounded queue. Coalesce in queue if a presence event for the same user is already there.
5. **Per-conn writer task**: drains the queue with `writev`; TLS-encrypts; sends frames.
6. **Client**: parses frame; updates local state; rerenders the member list with the new dot color.

Every step is O(local subscribers) or O(1). Memory is amortized. No locks held across awaits. Slow consumers can't block. Memory is bounded. **The system structurally cannot collapse the way the naive design does** — that's the whole point.

---

## 17. What Makes This Staff-Level

1. **Identifying the three structural failure modes upfront** (memory, threads, locks) and showing each pattern targets one or more, not just throwing techniques at the wall.
2. **Pushing the recipient enumeration to the lowest tier possible** — gateway-as-subscriber instead of user-as-subscriber is the architectural unlock; everything else is consequence.
3. **Bounded queues with drop policies as a design principle, not an afterthought** — slow consumers must not affect anyone else, ever.
4. **Subscription pruning** — exploiting that "membership" ≠ "active viewer" gives 1000× cost reduction; identifying this is what separates scalable from doomed.
5. **Coalescing in time** — recognizing presence/typing/cursor are *reducible* state; we owe consumers the *current* value, not the transition history.
6. **Concrete concurrency primitives**: sharded maps for read-heavy state, COW for very-read-heavy, actor model for isolated mutability, SPSC ring buffers per connection.
7. **Reconnect storm mitigations** baked in (cursor resume, ring buffer of recent events, jittered client reconnect, drained restarts).
8. **Hierarchical fan-out for celebrity scopes** — knowing the basic two-tier breaks at extreme cardinality and what to reach for.
9. **Per-scope SLO budgets and observability metrics** (queue depth, coalesce ratio, drop policy engagement) — operational maturity, not just architectural prettiness.
10. **Wire-format and protocol choices** justified on real grounds (Protobuf decode latency, WebTransport stream isolation, permessage-deflate CPU cost on mobile).