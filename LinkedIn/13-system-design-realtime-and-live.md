# 13 · System Design — Real-time Platform and LinkedIn Live

This file covers two related but distinct surfaces:

1. **The Real-time Platform** — the WebSocket-based infrastructure that powers messaging, typing indicators, presence, push notifications, real-time feed updates, live event chat. (Internal name varies; commonly called **"Real-Time Frontend"** or **RTFE**.)
2. **LinkedIn Live** — the video broadcast product (one-to-many video streaming, often used for company town halls, professional events, executive interviews).

A staff candidate is often asked about either as a standalone system design.

---

## Part A — Real-time Platform

## 13.1 What the platform serves

- **Messaging deliveries** (push messages to recipients without polling).
- **Typing indicators**, **read receipts**, **presence updates**.
- **Real-time notifications** (in-app bell-icon updates).
- **Feed updates** (new posts appear without refresh).
- **Live event signals** (LinkedIn Live chat, reactions, viewer count).
- **Coordination signals** for collaborative surfaces.

## 13.2 Requirements

### Functional

- A client (mobile / web) maintains a long-lived bi-directional channel with the server.
- Server can push events to any subscribed client by member_id (or session_id).
- Clients can subscribe to specific topics (a conversation, a Live event, a feed).
- Delivery semantics: at-least-once with client-side dedup; out-of-order allowed but client orders by server-assigned timestamp.

### Non-functional

- **Scale**: tens of millions of concurrent connections.
- **Latency**: server-to-client < 200ms for typing-indicator-class events.
- **Availability**: 99.95%.
- **Reconnects**: graceful — clients restore state quickly after drop.

## 13.3 Architecture

```
   Client (WebSocket)
        │
        ▼
   ┌────────────────────┐
   │ Real-Time Gateway  │   ← stateless-ish; holds WS connections
   │ (RTG instances)    │
   └─────────┬──────────┘
             │
   subscription registry (member_id → RTG instance)
             │
             ▼
   ┌─────────────────────────┐
   │ Pub/Sub backbone (Kafka │
   │ or per-topic mesh)      │
   └─────────────────────────┘
             ▲
             │ publishers (Messaging, Feed, Notifications, ...)
```

### RTG instances

- Java/Netty-based servers handling tens of thousands of WS connections each.
- TLS terminated at the edge or at the RTG.
- Each WS session is bound to a `(member_id, device_id)`.
- Heartbeat every 30s; idle drop after a few minutes of no traffic.
- Fallback: long-poll or SSE for restrictive networks.

### Routing

When a publisher wants to push to `member_id = X`:
- Lookup `X → RTG instance` in a fast registry.
- Forward the event over an internal RPC to the right RTG.
- RTG pushes to all of X's active sessions (a member can have multiple — web + mobile).

Registry options:
- **Centralized** — fast KV (Redis-like / ZooKeeper).
- **Hash-based** — every RTG owns `hash(member_id) % N` members; deterministic.

LinkedIn uses a hybrid: hash-based for the steady-state, with control-plane for failover/membership changes.

### Sessions and reconnects

- On reconnect: client sends last-event-id; server resumes from that point if possible.
- For missed events during disconnect: backfill via a per-session event queue (durable, short TTL).

## 13.4 Pub/Sub backbone

Common shapes:

- **Kafka topics per subscription type**: messaging-events, presence-events, feed-events. RTG instances are consumers; they get all events for their member partitions and filter.
- **Custom mesh**: targeted point-to-point delivery, lower latency but more complex.

LinkedIn uses Kafka heavily for the eventually-consistent paths and dedicated RPC for low-latency targeted pushes.

## 13.5 Scale numbers

- 50–100K WS connections per RTG machine with tuned kernel + Netty event loop + minimal GC.
- For 50M concurrent connections: ~500–1000 RTG instances.
- Memory dominated by per-connection buffers; CPU dominated by heartbeats and (rarely) message pushes.

## 13.6 Failure modes

- **RTG instance crash** — clients reconnect; presence shows brief drop; pending pushes restored from the backbone.
- **Backbone outage** — fall back to polling (clients periodically GET /events?since=X).
- **Registry inconsistency** — short-window mis-routing; events drained from a fallback path (the Messaging service knows where the member is, separately).
- **Connection storm** after recovery — exponential backoff on the client.

## 13.7 Common follow-ups (real-time)

> **"What's the difference between WebSocket and SSE for this?"**
WS is bi-directional, SSE is server-to-client only. For features like typing indicators, WS is natural; for pure server-push, SSE works and is simpler. LinkedIn uses WS primarily; SSE as a fallback.

> **"How would you support presence at 1B members?"**
Presence is *light* per member: a heartbeat timestamp. Even at 100M concurrent, an in-memory store with sharding is feasible. Subscribers to presence (e.g., "are my chat partners online?") get push from the presence service.

> **"How do you handle a flapping connection?"**
Hysteresis: don't mark offline immediately; wait N seconds of no heartbeat. Suppress presence-change events that revert within a window.

---

## Part B — LinkedIn Live

LinkedIn Live is video broadcasting: a host streams from a laptop / phone / production rig, viewers watch on web/mobile, chat happens in real-time, replays are available.

## 13.8 Requirements (Live)

### Functional

- Hosts can go live (1:many) via the LinkedIn mobile/web app or via RTMP push from external software (OBS).
- Viewers watch live with low latency (< 5s glass-to-glass typical).
- Live chat / reactions during broadcast.
- Replays available after broadcast ends, watchable on-demand.
- Captions (auto-generated via ASR; editable).
- Notifications when a followed creator goes live.
- Monetization hooks (paid events, sponsored content during stream).

### Non-functional

- **Scale**: tens of thousands of concurrent broadcasts; some with hundreds of thousands of concurrent viewers.
- **Latency**: glass-to-glass 2–8 seconds depending on protocol (lower with WebRTC, higher with HLS).
- **Availability**: a dropped stream is a brand crisis for a sponsoring company.

## 13.9 Architecture

```
   Host (creator app or OBS)
        │  RTMP / SRT / WebRTC
        ▼
   ┌────────────────┐
   │ Ingest         │ (RTMP server cluster)
   │ - transcoding  │ (FFmpeg-based, per-quality renditions)
   │ - packaging    │ (HLS / DASH segments)
   └─────┬──────────┘
         │ segments
         ▼
   ┌────────────────┐
   │ Origin Storage │ ← Ambry or specialized object store
   └─────┬──────────┘
         │
         ▼
   ┌────────────────┐
   │ CDN (multi-CDN │
   │ Akamai/Fastly/ │
   │ Azure CDN)     │
   └─────┬──────────┘
         │ HLS/DASH or LL-HLS
         ▼
   ┌────────────────┐
   │ Viewer player  │
   └────────────────┘

   Parallel: live chat / reactions ride on the real-time platform.
```

## 13.10 Ingest

- **Protocols**: RTMP (most common for OBS), SRT (more resilient), WebRTC (lowest latency).
- **Ingest cluster**: regional, anycast'd. Hosts connect to nearest PoP.
- **Authentication**: stream key bound to host + scheduled session.
- **Reliability**: redundant ingest path; switch to backup if primary fails.

## 13.11 Transcoding

- Receive a high-bitrate source.
- Transcode to multiple ladders (e.g., 4K, 1080p60, 720p, 480p, 240p).
- Encode for HLS (TS or fMP4 segments) and / or DASH.
- Hardware-accelerated (Intel QSV, AMD VCN, NVIDIA NVENC) for cost.

## 13.12 Packaging and delivery

- **HLS** with ~6s segments → 12–20s end-to-end latency. Standard.
- **Low-Latency HLS (LL-HLS)** with partial segments → 3–6s latency.
- **DASH** with low-latency profile.
- **WebRTC** for sub-second (used for stage-call features, not typically for 1:many at scale).

CDN distribution: multi-CDN with failover. Edge servers cache segments aggressively (each segment is immutable once published).

## 13.13 Chat / reactions during live

- Chat events flow to the real-time platform (RTFE).
- Fanout to viewers via WebSocket.
- For very high-concurrent broadcasts (100K+ viewers), batch reaction emojis (don't send every emoji; aggregate as "+5K hearts in last 1s").
- Chat moderation: ML classifier in line for spam / abuse; rate-limit per viewer.

## 13.14 Notifications

- When creator goes live, ATC sends "live now" push to followers (subject to caps).
- Spike traffic: anticipate the click-through and pre-warm CDN at the segment URL.

## 13.15 Replays

- Once broadcast ends, segments are stitched into a VOD (video-on-demand) asset.
- Stored in Ambry / object storage; transcoded to additional ladders if needed.
- Indexed for search; available in member's profile content.

## 13.16 Failure modes

- **Ingest drop** — hot-swap to backup ingest path; some sub-second loss may occur.
- **CDN outage in one region** — DNS / Anycast shifts viewers to another CDN.
- **Transcoder failure** — re-route to a different transcoder pool.
- **Chat backbone overload** — degrade reactions to aggregated; queue chat with backpressure.
- **Catastrophic failure mid-broadcast** — restart broadcast; switch to a new stream key; notify viewers; preserve VOD up to the failure.

## 13.17 Operational concerns

- **Cost per viewer-hour** is a key metric. CDN egress is the dominant cost.
- **Brand safety** — Live content can go off-rails fast; ML monitors for graphic/abusive content and can interrupt.
- **DMCA / copyright** — content fingerprinting on ingest (rolling hashes of audio and video).
- **Captioning** — increasingly required (ADA / accessibility).
- **Multi-region** — viewers served from nearest CDN; origin can be regional or global depending on broadcast scale.

## 13.18 Common follow-ups (Live)

> **"How would you support 1M concurrent viewers on a Live event?"**
CDN-heavy: every segment ~immutable, edge-cacheable; minimal origin egress. Pre-warm: when 5min before scheduled event, push first-segment placeholder to CDN edges. Multi-CDN failover. Chat backbone shards by event_id; aggregate reactions.

> **"How do you get latency below 3s?"**
LL-HLS or LL-DASH; CMAF chunked segments; CDN supports low-latency. WebRTC is even faster but doesn't scale as cheaply for 1:many.

> **"How do you handle a creator's mid-stream disconnect?"**
Buffer 30s of audio on the ingest side. If client reconnects within 30s, splice seamlessly. Otherwise, end the broadcast and provide replay.

> **"How do you monetize?"**
Sponsorship overlays (server-stitched ads, mid-roll for some), paid-event ticketing (Eventbrite-style), Premium-required broadcasts.

> **"How would you support real-time captions for a Live broadcast?"**
ASR model on the audio stream (WebSocket from ingest); caption-event over the real-time platform to viewers; rendering at the player.

> **"What's the right database for storing chat messages from a Live event?"**
Per-event log (Kafka or similar) + Pinot/Espresso for queryable archive. Hot reads from in-memory; cold reads from durable storage.