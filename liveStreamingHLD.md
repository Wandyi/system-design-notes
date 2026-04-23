# Live Streaming Platform — Comprehensive High-Level Design
## (Disney+ Hotstar Scale: 50M+ Concurrent Viewers)

## Table of Contents

1. [Requirements & Scale](#1-requirements--scale)
2. [Back-of-Envelope Estimation](#2-back-of-envelope-estimation)
3. [High-Level Architecture](#3-high-level-architecture)
4. [Core API Design](#4-core-api-design)
5. [Data Models](#5-data-models)
6. [Video Ingestion Pipeline](#6-video-ingestion-pipeline)
7. [Real-Time Transcoding & Packaging](#7-real-time-transcoding--packaging)
8. [Origin Server & Segment Storage](#8-origin-server--segment-storage)
9. [CDN Architecture & Edge Distribution](#9-cdn-architecture--edge-distribution)
10. [Adaptive Bitrate Streaming (Client-Side)](#10-adaptive-bitrate-streaming-client-side)
11. [Low-Latency Live Streaming](#11-low-latency-live-streaming)
12. [DRM & Content Protection](#12-drm--content-protection)
13. [Live Chat & Reactions at Scale](#13-live-chat--reactions-at-scale)
14. [Server-Side Ad Insertion (SSAI)](#14-server-side-ad-insertion-ssai)
15. [DVR / Rewind & Catch-Up](#15-dvr--rewind--catch-up)
16. [Authentication & Concurrency Control](#16-authentication--concurrency-control)
17. [Database & Storage Architecture](#17-database--storage-architecture)
18. [Caching Architecture](#18-caching-architecture)
19. [Scalability Deep Dive](#19-scalability-deep-dive)
20. [Reliability & Fault Tolerance Deep Dive](#20-reliability--fault-tolerance-deep-dive)
21. [Observability & Operational Excellence](#21-observability--operational-excellence)
22. [Corner Cases & Hard Problems](#22-corner-cases--hard-problems)

---

## 1. Requirements & Scale

### Functional Requirements

- **Live Streaming**: Broadcast live events (sports, concerts, premieres) to millions simultaneously
- **Adaptive Bitrate**: Automatic quality adjustment based on viewer's network conditions
- **Multi-Language Audio**: Simultaneous commentary tracks (Hindi, English, Tamil, Telugu, etc.)
- **Low Latency**: Stream should be < 15-30 seconds behind the actual live event
- **DVR / Rewind**: Viewers can pause, rewind, and resume live content
- **Live Scoreboard & Overlays**: Real-time scores, stats displayed as overlays on the stream
- **Live Chat & Reactions**: Real-time emoji reactions and chat during live events
- **DRM Protection**: Prevent piracy and unauthorized redistribution
- **Ad Insertion**: Server-side ad insertion during natural breaks (innings, halftime)
- **Multi-Device**: Mobile (Android/iOS), Smart TV, Web, Fire Stick, Chromecast
- **Concurrent Device Limits**: Max N simultaneous streams per subscription

### Non-Functional Requirements

- **Concurrent Viewers**: 50 million+ for marquee events (IPL Final, World Cup)
- **Availability**: 99.99% during live events (downtime = millions see a black screen)
- **Latency**: Glass-to-glass < 30 seconds (camera → viewer screen)
- **Start-up Time**: Stream should begin playing within 2 seconds of pressing Play
- **Buffering Ratio**: < 0.5% (fewer than 1 in 200 viewers should experience a buffer at any moment)
- **Scale-Up Time**: Go from 5M to 50M viewers in < 10 minutes (match start / key moment)

### What Makes Live Streaming Fundamentally Different

```
Video on Demand (VOD — Netflix):          Live Streaming (Hotstar — IPL):
  - Content pre-encoded, stored             - Content generated in real-time
  - CDN pre-warmed (popular titles cached)  - CDN cold at stream start (nothing to cache yet)
  - Viewers start at different times        - ALL viewers at same point in time
  - Diverse content (millions of titles)    - Single stream: EVERYONE watches the same thing
  - Gradual traffic growth                  - Instant spike: 0 → 50M at match start
  - Buffer 30+ seconds ahead               - Can only buffer a few seconds of future
  - Playback failure = retry from start     - Playback failure = miss the moment (irreversible)
  - Segment already exists on CDN           - Segment doesn't exist until encoder produces it
```

---

## 2. Back-of-Envelope Estimation

```
Peak Event: IPL Final / Cricket World Cup Final
  Concurrent viewers:      50,000,000  (Hotstar's recorded peak: 59.7M in 2024)
  Average stream bitrate:  3 Mbps (weighted across quality tiers)

Bandwidth:
  50M viewers × 3 Mbps = 150 Tbps total egress
  
  That's 150 TERABITS per second.
  For context: total internet bandwidth of some small countries.

  Breakdown by quality tier:
    4K (8 Mbps):      2% of viewers =  1M × 8 Mbps  =   8 Tbps
    1080p (5 Mbps):   15% of viewers = 7.5M × 5 Mbps =  37.5 Tbps
    720p (2.5 Mbps):  35% of viewers = 17.5M × 2.5   =  43.75 Tbps
    480p (1.2 Mbps):  30% of viewers = 15M × 1.2     =  18 Tbps
    360p (0.6 Mbps):  15% of viewers = 7.5M × 0.6    =   4.5 Tbps
    Audio-only (128k): 3% of viewers = 1.5M × 0.128  =   0.19 Tbps
    Total:                                            ≈  112 Tbps
  (Lower than 150 Tbps because most viewers are on mobile with 720p/480p)

Segment Requests:
  HLS segment duration: 6 seconds (standard) or 2 seconds (low-latency)
  50M viewers requesting a new segment every 6 seconds:
    50M / 6 = 8.3 million segment requests/sec
  
  But CDN caching makes this manageable:
    Each edge server serves ~50K-200K viewers
    Edge needs to fetch each segment from origin only ONCE
    Then serves it from local cache to all local viewers
    Origin sees: ~5,000 edge servers × 1 request/segment × 1 segment/6s
                = ~800 segment requests/sec to origin (tiny)

  The magic of live streaming: EVERYONE watches the SAME content.
  CDN cache hit rate approaches 99.99% because all viewers want the same segment.

Encoding:
  6 quality tiers × 4 audio tracks × 2 packaging formats (HLS + DASH)
    = 48 parallel encode/package streams
  Each at real-time speed: 1 second of video produced per 1 second of real time
  GPU requirement: ~24 GPUs (2 segments per GPU with modern hardware)

Storage (live window + DVR):
  6 tiers × 4 audio × average segment size 1.5 MB × 1 segment/6s:
    = 4 MB/s new data across all variants
  For 4-hour cricket match: 4 MB/s × 14,400s = 57.6 GB total encoded content
  Trivial storage — the challenge is BANDWIDTH, not storage.
```

**Key Insight**: Live streaming at 50M concurrent is fundamentally a **CDN bandwidth problem**, not a compute or storage problem. 
The origin produces ~800 segments/sec; everything else is about distributing those segments to 50M viewers through a massively parallel CDN.

---

## 3. High-Level Architecture

```
   Stadium Cameras                    Commentary Booths
   (4K SDI feeds)                     (Hindi, English, Tamil...)
        │                                    │
        ▼                                    ▼
   ┌─────────────┐                    ┌──────────────┐
   │  Broadcast  │                    │ Audio Mixing │
   │  Truck      │                    │(per language)│
   │  (SDI → IP) │                    └──────┬───────┘
   └──────┬──────┘                            │
          │  RTMP / SRT (redundant links)     │
          ▼                                    ▼
   ┌──────────────────────────────────────────────────┐
   │              INGEST LAYER                         │
   │  Primary Ingest (Region A)                        │
   │  Backup Ingest (Region B)    ← hot standby        │
   │  Protocol: SRT (low-latency, loss-resilient)      │
   └───────────────────┬──────────────────────────────┘
                       │
   ┌───────────────────▼──────────────────────────────┐
   │              TRANSCODE LAYER                      │
   │                                                   │
   │  ┌──────────┐ ┌──────────┐ ┌──────────┐         │
   │  │ 4K H.265 │ │1080p H264│ │720p H.264│  ...    │
   │  │ 8 Mbps   │ │ 5 Mbps   │ │ 2.5 Mbps │         │
   │  └────┬─────┘ └────┬─────┘ └────┬─────┘         │
   │       └─────────────┼───────────┘                 │
   │                     ▼                             │
   │  ┌──────────────────────────────────┐             │
   │  │ Packager (HLS + DASH + CMAF)     │             │
   │  │ Segment Duration: 6s (std)       │             │
   │  │                   2s (LL-HLS)    │             │
   │  │ + DRM encryption (Widevine,      │             │
   │  │   FairPlay, PlayReady)           │             │
   │  └──────────────┬──────────────────┘             │
   └─────────────────┼────────────────────────────────┘
                     │
   ┌─────────────────▼────────────────────────────────┐
   │              ORIGIN LAYER                         │
   │                                                   │
   │  Manifest Server (serves .m3u8 / .mpd)            │
   │  Segment Store (serves .ts / .m4s segments)       │
   │  Multi-region origin with origin shield           │
   └─────────────────┬────────────────────────────────┘
                     │
   ┌─────────────────▼────────────────────────────────┐
   │              CDN LAYER (Multi-CDN)               │
   │                                                  │
   │  ┌────────────┐  ┌────────────┐  ┌────────────┐  │
   │  │ CDN A      │  │ CDN B      │  │ CDN C      │  │
   │  │ (Akamai)   │  │ (CloudFront│  │(Own PoPs / │ │
   │  │            │  │  / Fastly) │  │ ISP embed) │ │
   │  └─────┬──────┘  └─────┬──────┘  └──────┬─────┘ │
   │        └────────────────┼───────────────┘       │
   └─────────────────────────┼────────────────────────┘
                             │
                  ┌──────────▼──────────┐
                  │    50M+ Viewers     │
                  │  (Mobile, TV, Web)  │
                  └────────────────────┘

   ┌──────────────────────────────────────────────────┐
   │           SUPPORTING SERVICES                     │
   │                                                   │
   │  Auth Service ── Chat/Reactions ── Scoreboard    │
   │  Ad Insertion ── Analytics ── DRM License Server  │
   │  Notifications ── User Service ── Payment/Subs   │
   └──────────────────────────────────────────────────┘
```

### Service Decomposition

| Service | Responsibility | Scaling Model |
|---|---|---|
| **Ingest Service** | Receive RTMP/SRT from broadcast, redundancy switching | 2-4 instances (active-passive per feed) |
| **Transcode Service** | Real-time encode to ABR ladder (6+ bitrates) | GPU fleet, horizontally scaled per quality tier |
| **Packager** | Segment video into HLS/DASH, encrypt with DRM | CPU-bound, scaled per packaging format |
| **Origin Server** | Serve manifests (.m3u8/.mpd) and segments to CDN | Shielded origin, ~1K req/sec per stream |
| **CDN Layer** | Edge distribution to 50M viewers | Multi-CDN, thousands of PoPs |
| **Manifest Manipulation** | Per-viewer manifest for ads, bitrate, DRM | Stateless, auto-scaled |
| **DRM License Server** | Issue decryption keys to authorized players | ~50M license requests at stream start |
| **Chat/Reactions** | Live emoji reactions and chat messages | Aggregated, sampled, not per-message |
| **Scoreboard Service** | Real-time score/stats overlay data | Low-throughput, high-availability |
| **SSAI (Ad Insertion)** | Splice ads into stream per-viewer | Per-viewer manifest rewriting |
| **Analytics** | Viewer count, buffering, bitrate, errors | Stream processing (Flink/Kafka Streams) |

---

## 4. Core API Design

### Stream Playback

```
GET    /v1/streams/{stream_id}/manifest
  Headers: Authorization: Bearer {token}
  Query:   ?format=hls|dash  &device=mobile|tv|web
  → 302 Redirect to CDN URL:
    https://cdn.example.com/live/{stream_id}/master.m3u8?token=signed_token

  The master manifest contains all quality tiers:
    #EXT-X-STREAM-INF:BANDWIDTH=8000000,RESOLUTION=3840x2160
    https://cdn/live/{stream_id}/4k/playlist.m3u8
    #EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
    https://cdn/live/{stream_id}/1080p/playlist.m3u8
    #EXT-X-STREAM-INF:BANDWIDTH=2500000,RESOLUTION=1280x720
    https://cdn/live/{stream_id}/720p/playlist.m3u8
    ...

  Each quality-level playlist is a ROLLING WINDOW of segments:
    #EXTINF:6.000,
    https://cdn/live/{stream_id}/720p/seg_1234.ts
    #EXTINF:6.000,
    https://cdn/live/{stream_id}/720p/seg_1235.ts
    #EXTINF:6.000,
    https://cdn/live/{stream_id}/720p/seg_1236.ts
    (Player fetches latest segment, buffers 2-3 ahead)
```

### Stream Control (Admin)

```
POST   /v1/admin/streams
  Body: { event_id, title, scheduled_start, ingest_url, languages: [...],
          drm_policy: "widevine+fairplay", ad_break_config: {...} }
  → 201 { stream_id, ingest_endpoint, stream_key }

POST   /v1/admin/streams/{stream_id}/start
  → 200 { status: "LIVE", started_at }

POST   /v1/admin/streams/{stream_id}/ad-break
  Body: { duration_seconds: 30, ad_pod_id: "pod_abc" }
  → 200 { break_id, inserted_at }

POST   /v1/admin/streams/{stream_id}/stop
  → 200 { status: "ENDED", vod_asset_id: "vod_123" }
  (Stream converted to VOD for catch-up viewing)
```

### Chat & Reactions

```
WebSocket: wss://chat.example.com/v1/streams/{stream_id}/chat
  Client → Server: { type: "reaction", emoji: "🏏" }
  Client → Server: { type: "message", text: "What a six!" }
  
  Server → Client: { type: "reaction_summary", window_seconds: 5,
                      counts: { "🏏": 142000, "🔥": 89000, "❤️": 54000 } }
  Server → Client: { type: "messages", messages: [
                      { user: "fan42", text: "What a six!", ts: "..." },
                      ... (sampled: 20 messages per 5-second window)
                    ] }

  (At 50M viewers, you CANNOT broadcast every message.
   Aggregate reactions. Sample chat messages. See Section 13.)
```

### Viewer Analytics (Internal)

```
POST   /v1/analytics/heartbeat    (sent by player every 30 seconds)
  Body: { stream_id, user_id, device, bitrate_bps, buffer_health_ms,
          dropped_frames, cdn_edge, isp, lat, lng, session_id }
  → 204
```

---

## 5. Data Models

### PostgreSQL — Event & Stream Metadata

```sql
CREATE TABLE live_events (
    event_id         BIGINT PRIMARY KEY,
    title            VARCHAR(500),
    sport            VARCHAR(50),
    scheduled_start  TIMESTAMPTZ,
    actual_start     TIMESTAMPTZ,
    actual_end       TIMESTAMPTZ,
    status           VARCHAR(20),      -- SCHEDULED, LIVE, ENDED, CANCELLED
    venue            VARCHAR(200),
    thumbnail_url    TEXT,
    created_at       TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE live_streams (
    stream_id        BIGINT PRIMARY KEY,
    event_id         BIGINT REFERENCES live_events(event_id),
    ingest_url       TEXT,
    stream_key       VARCHAR(128),
    status           VARCHAR(20),      -- IDLE, INGESTING, LIVE, ENDED
    drm_policy       VARCHAR(50),
    segment_duration INT DEFAULT 6,    -- seconds
    dvr_window_hours INT DEFAULT 4,
    languages        TEXT[],           -- {'hindi','english','tamil','telugu'}
    created_at       TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE stream_qualities (
    quality_id       BIGINT PRIMARY KEY,
    stream_id        BIGINT REFERENCES live_streams(stream_id),
    label            VARCHAR(20),      -- '4k', '1080p', '720p', '480p', '360p', 'audio'
    resolution_w     INT,
    resolution_h     INT,
    bitrate_bps      INT,
    codec            VARCHAR(20),      -- 'h264', 'h265', 'aac'
    manifest_path    TEXT               -- /live/{stream_id}/720p/playlist.m3u8
);
```

### Redis — Real-Time State

```
Viewer Count (HyperLogLog — approximate unique count):
  Key:    viewers:{stream_id}
  Type:   HyperLogLog
  Cmd:    PFADD viewers:stream_123 {user_id}
          PFCOUNT viewers:stream_123 → ~50,234,000
  Memory: 12 KB per stream (HLL is fixed-size regardless of cardinality)
  Error:  ±0.81% (50M ± 405K — precise enough for display)

Reaction Counters (per 5-second window):
  Key:    reactions:{stream_id}:{window_ts}
  Type:   Hash
  Fields: { "🏏": 142000, "🔥": 89000, "❤️": 54000 }
  TTL:    30s

Active Sessions (for concurrent device limit):
  Key:    sessions:{user_id}
  Type:   Sorted Set
  Score:  last_heartbeat_timestamp
  Member: session_id
  TTL:    300s
  → ZCARD to check active device count
  → ZRANGEBYSCORE to evict stale sessions (no heartbeat > 60s)

CDN Routing Decision:
  Key:    cdn_routing:{region}
  Type:   Hash
  Fields: { "primary": "akamai", "secondary": "cloudfront", "weight_primary": 70 }
  TTL:    60s (updated by CDN health checker)
```

---

## 6. Video Ingestion Pipeline

### From Stadium Camera to Cloud

```
Stadium                    Backhaul                    Cloud Ingest
┌──────────┐          ┌──────────────┐          ┌──────────────────┐
│ Camera 1 │──SDI────→│              │          │                  │
│ Camera 2 │──SDI────→│  Broadcast   │──SRT────→│  Ingest Server A │
│ Camera 3 │──SDI────→│  Truck /     │  (primary│  (Region 1)      │
│ ...      │          │  OB Van      │   link)  │                  │
│ Camera N │──SDI────→│              │          └──────────────────┘
└──────────┘          │  (vision     │
                      │   mixing,    │──SRT────→┌──────────────────┐
                      │   graphics,  │  (backup │  Ingest Server B │
                      │   replays)   │   link)  │  (Region 2)      │
                      └──────────────┘          └──────────────────┘

Key decisions:

1. SRT over RTMP for backhaul:
   RTMP: TCP-based → head-of-line blocking, no FEC, high retransmit latency
   SRT:  UDP-based → forward error correction, adaptive bitrate recovery,
         handles 5-10% packet loss gracefully, ~200ms lower latency than RTMP
   At 50M concurrent, every millisecond of glass-to-glass latency matters.

2. Dual-path ingest:
   Two independent network paths from stadium to cloud.
   Different ISPs, different physical routes.
   If primary path fails: automatic switchover in < 1 second.
   Viewers see a brief glitch (1-2 frames) at most.

3. Ingest format:
   From truck: 1080p60 or 4K60, 50-80 Mbps, H.264 (mezzanine quality)
   This is the HIGHEST quality. Everything downstream is a lossy transcode.
```

### Ingest Server Internals

```
┌──────────────────────────────────────────────────────┐
│                  Ingest Server                       │
│                                                      │
│  1. SRT Listener                                     │
│     - Accept incoming SRT stream                     │
│     - Validate stream key (authentication)           │
│     - Measure incoming bitrate, FEC stats, RTT       │
│                                                      │
│  2. Frame Validator                                  │
│     - Check for black frames (source failure)        │
│     - Check for frozen frames (encoder crash)        │
│     - Check audio levels (silence detection)         │
│     - If bad frames > 2 seconds: trigger failover    │
│       to backup ingest                               │
│                                                      │
│  3. Timestamping                                     │
│     - Assign wall-clock PTS (presentation timestamp) │
│     - Ensures all downstream encoders produce        │
│       segments aligned to the same timeline          │
│                                                      │
│  4. Fan-Out to Transcoders                           │
│     - Forward raw frames to N transcoder instances   │
│     - Via internal network (not internet)            │
│     - Using multicast or pub/sub for efficiency      │
│                                                      │
└──────────────────────────────────────────────────────┘
```

---

## 7. Real-Time Transcoding & Packaging

### ABR Encoding Ladder

```
Input: 1080p60 mezzanine @ 50 Mbps from ingest

┌────────────┬────────────┬──────────┬────────────┬─────────────────┐
│ Profile    │ Resolution │ Bitrate  │ Codec      │ Target Device   │
├────────────┼────────────┼──────────┼────────────┼─────────────────┤
│ 4K UHD     │ 3840×2160  │ 15 Mbps  │ H.265/HEVC │ Smart TV, FStick│
│ 4K         │ 3840×2160  │ 8 Mbps   │ H.265/HEVC │ Smart TV        │
│ 1080p      │ 1920×1080  │ 5 Mbps   │ H.264      │ Web, tablet     │
│ 720p       │ 1280×720   │ 2.5 Mbps │ H.264      │ Mobile (4G/WiFi)│
│ 480p       │ 854×480    │ 1.2 Mbps │ H.264      │ Mobile (3G/slow)│
│ 360p       │ 640×360    │ 600 Kbps │ H.264      │ Mobile (2G)     │
│ Audio only │ N/A        │ 128 Kbps │ AAC        │ Low bandwidth   │
└────────────┴────────────┴──────────┴────────────┴─────────────────┘

× 4 audio tracks (Hindi, English, Tamil, Telugu)
× 2 packaging formats (HLS for Apple, DASH for Android/Web)
= 7 video profiles × 4 audio × 2 formats
= 56 simultaneous encoding/packaging pipelines
(In practice: encode video and audio separately, mux at packaging stage)

Hardware:
  Each video profile: 1 GPU (NVIDIA T4 or newer) for real-time H.264/H.265
  7 video profiles × 2 (primary + backup) = 14 GPUs for video encoding
  Audio encoding: CPU-only, negligible resources
  Packaging: CPU-only, ~4 cores per format
```

### Segment Generation

```
Encoder produces a continuous stream of video.
Packager splits it into fixed-duration segments.

Standard HLS:
  Segment duration: 6 seconds
  Each segment is a self-contained video file (.ts or .m4s)
  Playlist updated every 6 seconds with the new segment:
  
    #EXTM3U
    #EXT-X-TARGETDURATION:6
    #EXT-X-MEDIA-SEQUENCE:1234
    #EXTINF:6.000,
    seg_1234.ts         ← 6 seconds ago
    #EXTINF:6.000,
    seg_1235.ts         ← just produced
    
  Player polls playlist every 3 seconds (half segment duration)
  Sees new segment → requests it → plays it
  
  End-to-end latency contribution:
    Segment accumulation:   6s (encoder must buffer a full segment)
    Segment transfer:       ~1s (to origin + CDN edge)
    Player playlist poll:   ~3s (average wait for next poll)
    Player buffer:          ~6s (buffers at least 1 segment ahead)
    Total:                  ~16-20 seconds behind live

Low-Latency HLS (LL-HLS):
  Segment duration: 6 seconds BUT divided into 2-second "partial segments"
  Playlist includes partial segments as they're produced:
  
    #EXT-X-PART:DURATION=2.0,URI="seg_1235_part0.m4s"
    #EXT-X-PART:DURATION=2.0,URI="seg_1235_part1.m4s"
    #EXT-X-PART:DURATION=2.0,URI="seg_1235_part2.m4s"  ← just produced
    
  Player requests partial segments as soon as they appear.
  Can also use HTTP/2 push or blocking playlist requests.
  
  End-to-end latency:
    Partial segment:  2s
    Transfer:         ~0.5s
    Player buffer:    ~2s
    Total:            ~5-8 seconds behind live
```

### Encoding Redundancy

```
Every encoder runs as an active-active pair:

  ┌──────────────┐     ┌──────────────┐
  │ Encoder A    │     │ Encoder B    │
  │ (Primary)    │     │ (Backup)     │
  │              │     │              │
  │ Produces:    │     │ Produces:    │
  │ seg_1235.ts  │     │ seg_1235.ts  │
  │ (identical)  │     │ (identical)  │
  └──────┬───────┘     └──────┬───────┘
         │                    │
         └──────────┬─────────┘
                    │
         ┌──────────▼──────────┐
         │  Segment Selector   │
         │                     │
         │  Takes from A.      │
         │  If A's segment is  │
         │  late (>200ms) or   │
         │  corrupt: switch    │
         │  to B's segment.    │
         │                     │
         │  Seamless to viewer │
         │  (same PTS, same    │
         │  content, aligned   │
         │  segment boundaries)│
         └─────────────────────┘

Both encoders receive the same input frames with the same timestamps.
Both produce bit-identical segments (same GOP structure, same keyframe alignment).
If Encoder A crashes mid-segment, Encoder B's segment is ready.
Switchover time: 0 ms (no gap, no glitch).
```

---

## 8. Origin Server & Segment Storage

### Architecture

```
           Packager produces segments
                    │
                    ▼
         ┌──────────────────┐
         │   Segment Store  │
         │                  │
         │  Hot: RAM disk   │  Last 30 seconds (5 segments)
         │  Warm: NVMe SSD  │  Last 4 hours (DVR window)
         │  Cold: S3        │  Full event (for VOD conversion)
         │                  │
         └────────┬─────────┘
                  │
         ┌────────▼─────────┐
         │  Origin Server   │
         │                  │
         │  Serves:         │
         │  - Manifests     │  (.m3u8 / .mpd) — DYNAMIC, rewritten per request
         │  - Segments      │  (.ts / .m4s) — STATIC once produced
         │                  │
         │  Behind:         │
         │  Origin Shield   │  (CDN-provided caching layer)
         │                  │
         └──────────────────┘
```

### Why Origin Shield is Critical

```
Without origin shield:
  5,000 CDN edge servers worldwide.
  New segment produced every 6 seconds.
  Each edge server requests the new segment from origin.
  = 5,000 requests to origin every 6 seconds = ~833 req/sec per segment.
  × 7 quality tiers = ~5,800 req/sec.
  Manageable but fragile under spikes.

  REAL problem: segment produced at t=0.
  At t=0.001s: 5,000 edge servers all request it simultaneously.
  = thundering herd at origin for every single segment.

With origin shield (2-3 shield nodes in strategic regions):
  
  Edge → Origin Shield → Origin
  
  5,000 edges → 3 shields → 1 origin
  
  Shield caches the segment after first request.
  4,997 of the 5,000 edge requests are served from shield cache.
  Origin sees: 3 requests per segment (one from each shield).
  = 3 × 7 tiers / 6 seconds = 3.5 req/sec to origin.
  
  Origin load reduced 1,000x.

  ┌────────────┐  ┌────────────┐  ┌────────────┐
  │Edge (India)│  │Edge (US)   │  │Edge (EU)   │
  │ 2000 PoPs  │  │ 1000 PoPs  │  │ 1000 PoPs  │
  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘
        │               │               │
        ▼               ▼               ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ Shield   │    │ Shield   │    │ Shield   │
  │ Mumbai   │    │ Virginia │    │ Frankfurt│
  └─────┬────┘    └─────┬────┘    └─────┬────┘
        │               │               │
        └───────────────┼───────────────┘
                        │
                  ┌─────▼─────┐
                  │  Origin   │  (sees ~3 req/sec, not 5,000)
                  └───────────┘
```

---

## 9. CDN Architecture & Edge Distribution

### The Bandwidth Problem

```
150 Tbps cannot be served by a single CDN provider.

No single CDN has 150 Tbps of capacity in India alone.
Akamai global capacity: ~300 Tbps (total, all customers, all regions)
Hotstar would need 50% of Akamai's GLOBAL capacity for ONE cricket match.

Solution: Multi-CDN + ISP-embedded caching.
```

### Multi-CDN Strategy

```
┌────────────────────────────────────────────────────────────────┐
│                   CDN Traffic Distribution                      │
│                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │  CDN A       │  │  CDN B       │  │  CDN C               │ │
│  │  (Akamai)    │  │  (CloudFront)│  │  (Own / ISP-embedded)│ │
│  │              │  │              │  │                      │ │
│  │  40% traffic │  │  30% traffic │  │  30% traffic         │ │
│  │  ~60 Tbps    │  │  ~45 Tbps    │  │  ~45 Tbps            │ │
│  │              │  │              │  │                      │ │
│  │  4000 PoPs   │  │  2000 PoPs   │  │  500 ISP PoPs        │ │
│  │  (global)    │  │  (global)    │  │  (India-specific)    │ │
│  └──────────────┘  └──────────────┘  └──────────────────────┘ │
│                                                                │
│  CDN selection per viewer (at manifest request time):          │
│  1. Viewer location (geo-IP → nearest CDN with capacity)      │
│  2. CDN health (real-time: error rate, latency per PoP)       │
│  3. ISP affinity (Jio users → ISP-embedded cache)             │
│  4. Cost optimization (cheaper CDN preferred if equal quality) │
│  5. Load balancing (redistribute if one CDN is saturated)     │
└────────────────────────────────────────────────────────────────┘
```

### ISP-Embedded Caches (India-Specific, Critical)

```
India's internet topology:
  - 500M+ smartphone users, mostly on 4G
  - Jio alone: ~450M subscribers
  - Jio's last-mile bandwidth to users: massive
  - But Jio's peering with CDN backbone: limited
  
  Problem: 20M Jio users want the same cricket stream.
  All traffic must flow: CDN backbone → Jio peering point → Jio network → users.
  Peering point becomes the bottleneck.

  Solution: Place cache servers INSIDE Jio's network.

  ┌──────────────────────────────────────────────────────────┐
  │                    Jio Network                            │
  │                                                          │
  │  ┌─────────────┐                                        │
  │  │ Hotstar     │  Physically located in Jio's datacenter│
  │  │ Cache Server │  Connected to Jio's internal backbone  │
  │  │ (Mumbai)     │  Serves 5M viewers WITHOUT crossing    │
  │  └──────┬──────┘  the peering point                     │
  │         │                                                │
  │  ┌──────▼────────────────────────────────────┐          │
  │  │         Jio Internal Backbone              │          │
  │  └──────────┬─────────┬──────────┬───────────┘          │
  │             │         │          │                       │
  │         ┌───▼──┐  ┌───▼──┐  ┌───▼──┐                   │
  │         │ Cell │  │ Cell │  │ Cell │  ...               │
  │         │Tower │  │Tower │  │Tower │                    │
  │         └──┬───┘  └──┬───┘  └──┬───┘                   │
  │            │         │         │                         │
  │         ┌──▼──┐   ┌──▼──┐  ┌──▼──┐                     │
  │         │Users│   │Users│  │Users│                      │
  │         └─────┘   └─────┘  └─────┘                     │
  │                                                          │
  └──────────────────────────────────────────────────────────┘

  Replicated for each major ISP: Jio, Airtel, Vi, BSNL.
  Each ISP cache server: 1-2 standard servers with 10 Gbps NIC.
  Serves the most popular bitrate (720p, 480p) from local cache.
  Cache hit rate: ~99.9% (everyone watches the same live stream).
  
  Net effect: 80% of India's live streaming traffic never leaves
  the viewer's own ISP network. The internet backbone barely notices.
```

### CDN Failover (Real-Time)

```
CDN health monitoring runs every 5 seconds:

  For each CDN edge PoP:
    1. Synthetic requests: fetch latest segment, measure latency + success
    2. Real User Metrics (RUM): aggregate error rate from player telemetry
    3. Throughput test: is the PoP delivering at expected bitrate?

  If CDN A Mumbai PoP error rate > 5% for 15 seconds:
    1. CDN router marks CDN A Mumbai as unhealthy
    2. New manifest requests for Mumbai viewers → CDN B / CDN C
    3. Existing viewers: player detects segment fetch failures
       → player-side CDN fallback (manifest contains alternate CDN URLs)
    4. Failover time: < 10 seconds (next segment fetch goes to backup CDN)

  Multi-CDN manifest (player has fallback built in):
    #EXT-X-STREAM-INF:BANDWIDTH=2500000
    https://cdn-a.example.com/live/720p/playlist.m3u8
    #EXT-X-STREAM-INF:BANDWIDTH=2500000
    https://cdn-b.example.com/live/720p/playlist.m3u8
    
    Player tries cdn-a first. If 2 consecutive failures: switch to cdn-b.
```

---

## 10. Adaptive Bitrate Streaming (Client-Side)

### ABR Algorithm

```
The player continuously estimates available bandwidth and switches
between quality levels to maximize quality while avoiding buffering.

  ┌────────────────────────────────────────────────────────────────┐
  │                 ABR Decision Loop (every segment)              │
  │                                                                │
  │  Inputs:                                                       │
  │   - Measured throughput (bytes received / download time)        │
  │   - Buffer level (seconds of video buffered ahead)             │
  │   - Historical throughput (rolling average, last 5 segments)   │
  │   - Segment duration (6 seconds)                               │
  │                                                                │
  │  Algorithm (simplified BOLA / buffer-based):                   │
  │                                                                │
  │  if buffer_level < 4 seconds:                                  │
  │    EMERGENCY: switch to lowest bitrate immediately             │
  │    (prevent buffering at all costs)                            │
  │                                                                │
  │  else if buffer_level < 8 seconds:                             │
  │    CONSERVATIVE: use bitrate ≤ 70% of measured throughput      │
  │    (build buffer back up)                                      │
  │                                                                │
  │  else if buffer_level < 15 seconds:                            │
  │    NORMAL: use bitrate ≤ 90% of measured throughput            │
  │    (steady state)                                              │
  │                                                                │
  │  else (buffer > 15 seconds):                                   │
  │    AGGRESSIVE: try next-higher bitrate                         │
  │    (buffer is healthy, try to improve quality)                 │
  │                                                                │
  │  Constraint: never switch more than 1 level at a time          │
  │  (avoid oscillation between 4K and 360p)                       │
  │                                                                │
  └────────────────────────────────────────────────────────────────┘
```

### India-Specific Challenges

```
India has extreme network diversity in the same audience:

  ┌──────────────────┬─────────────┬──────────────┬────────────┐
  │ Segment          │ Bandwidth   │ Typical      │ % of 50M   │
  │                  │             │ Quality      │ viewers    │
  ├──────────────────┼─────────────┼──────────────┼────────────┤
  │ Urban WiFi       │ 50-100 Mbps │ 4K / 1080p   │ 10%        │
  │ Urban 4G (Jio)   │ 10-30 Mbps  │ 1080p / 720p │ 35%        │
  │ Semi-urban 4G    │ 5-15 Mbps   │ 720p / 480p  │ 30%        │
  │ Rural 4G         │ 2-5 Mbps    │ 480p / 360p  │ 15%        │
  │ Rural 3G / 2G    │ 0.5-2 Mbps  │ 360p / audio │ 8%         │
  │ Congested tower  │ 0.1-1 Mbps  │ audio-only   │ 2%         │
  └──────────────────┴─────────────┴──────────────┴────────────┘

During an IPL Final:
  - Cell towers near stadiums are completely saturated
  - Residential towers in cricket-mad cities see 3-5x normal load
  - Effective bandwidth drops for everyone

  Player must react in < 10 seconds when bandwidth drops:
  720p → 480p (within 1 segment) → 360p if still degrading
  
  Buffering ratio target: < 0.5%
  At 50M viewers: 0.5% = 250,000 viewers buffering at any moment.
  That's still too many. Target: < 0.1% = 50,000.
```

---

## 11. Low-Latency Live Streaming

### Glass-to-Glass Latency Breakdown

```
Event happens on field → viewer sees it on screen

Standard HLS (6-second segments):
  ┌───────────────────────────────────────────────────────────┐
  │ Camera capture + broadcast truck processing:       0.5s   │
  │ SRT backhaul (stadium → cloud):                    0.5s   │
  │ Transcoding + packaging (accumulate 6s segment):   6.0s   │
  │ Origin → CDN edge propagation:                     0.5s   │
  │ Player playlist poll wait (avg half segment):      3.0s   │
  │ Player buffer (at least 1 segment):                6.0s   │
  │ Player decode + render:                            0.1s   │
  ├───────────────────────────────────────────────────────────┤
  │ TOTAL:                                            ~16-17s  │
  └───────────────────────────────────────────────────────────┘

Low-Latency HLS (LL-HLS, 2-second partial segments):
  ┌───────────────────────────────────────────────────────────┐
  │ Camera capture + broadcast truck:                  0.5s   │
  │ SRT backhaul:                                      0.3s   │
  │ Transcoding + packaging (2s partial segment):      2.0s   │
  │ Origin → CDN edge:                                 0.3s   │
  │ Player blocking playlist request (near-instant):   0.2s   │
  │ Player buffer (1 partial segment):                 2.0s   │
  │ Player decode + render:                            0.1s   │
  ├───────────────────────────────────────────────────────────┤
  │ TOTAL:                                             ~5-6s   │
  └───────────────────────────────────────────────────────────┘

CMAF + Chunked Transfer Encoding (ultra-low-latency):
  ┌───────────────────────────────────────────────────────────┐
  │ Camera + truck + backhaul:                         0.8s   │
  │ Encode chunk (0.5s of video):                      0.5s   │
  │ Chunked HTTP transfer (streaming, not store-and-   0.3s   │
  │   forward):                                               │
  │ Player decode (no full segment needed):            0.2s   │
  │ Tiny buffer:                                       0.5s   │
  ├───────────────────────────────────────────────────────────┤
  │ TOTAL:                                             ~2-3s   │
  └───────────────────────────────────────────────────────────┘
  
  Trade-off: Ultra-low-latency = more CDN load (smaller segments = more requests)
             and less resilient to network jitter (tiny buffer).
  
  Hotstar's choice for cricket: ~8-15 seconds.
  Good enough for sports. 2-3 seconds is for betting/trading use cases.
```

---

## 12. DRM & Content Protection

### Multi-DRM Architecture

```
                                   ┌──────────────────┐
                                   │  DRM License      │
                                   │  Server            │
                                   │                    │
                                   │  Issues keys for:  │
                                   │  - Widevine (Android, Chrome, Smart TV)
                                   │  - FairPlay (iOS, Safari, Apple TV)
                                   │  - PlayReady (Edge, Xbox, Windows)
                                   └─────────┬────────┘
                                             │
  Packager encrypts segments:                │ Player requests license:
  ┌────────────┐                             │
  │ seg_1235   │                   ┌─────────▼────────┐
  │ .ts/.m4s   │ ── AES-128-CTR → │   Encrypted       │
  │ (clear)    │    encryption     │   seg_1235.m4s   │
  └────────────┘                   └──────────────────┘
  
  Key rotation: new key every 5 minutes (limits exposure if key is leaked)
  
  Scale challenge: 50M viewers all need a license at stream start.
    50M license requests in ~60 seconds = 833K license requests/sec
    
    Solution:
    1. License pre-fetch: player requests license 5 seconds before stream starts
       (during loading screen / countdown). Spreads the load.
    2. License caching: license valid for 4 hours. Player caches locally.
       No re-request on quality switch or seek.
    3. DRM license server: horizontally scaled, stateless (license is a
       signed blob computed from key ID + device certificate).
       50 instances × 20K req/sec each = 1M req/sec capacity.
```

---

## 13. Live Chat & Reactions at Scale

### The Problem

```
50M viewers sending reactions during a cricket match.
A six is hit → 10M users tap the reaction button within 2 seconds.
That's 5 million reactions per second.

Broadcasting every reaction to every viewer:
  5M reactions/sec × 50M viewers = 250 trillion message deliveries/sec
  Physically impossible.

Solution: Aggregate and sample. Don't broadcast individual messages.
```

### Architecture

```
  50M viewers sending reactions
        │
        ▼
  ┌──────────────────────────────────┐
  │  Reaction Ingestion              │
  │  (Kafka topic, partitioned       │
  │   by stream_id)                  │
  │                                  │
  │  Accept at: ~5M events/sec peak  │
  │  Kafka handles this easily       │
  │  (100 partitions, ~50K/partition)│
  └───────────────┬──────────────────┘
                  │
  ┌───────────────▼──────────────────┐
  │  Aggregation Service             │
  │  (Flink / Kafka Streams)         │
  │                                  │
  │  Time window: 5 seconds          │
  │  Output per window:              │
  │  { "🏏": 2,140,000,              │
  │    "🔥": 1,890,000,              │
  │    "❤️":   420,000 }              │
  │                                  │
  │  + Sample 20 chat messages       │
  │    (randomly selected, filtered  │
  │     for profanity)               │
  │                                  │
  └───────────────┬──────────────────┘
                  │
  ┌───────────────▼──────────────────┐
  │  Push to Viewers                 │
  │                                  │
  │  NOT individual WebSocket to 50M │
  │  Instead:                        │
  │                                  │
  │  Option A: Include in manifest   │
  │  (reaction data as timed         │
  │   metadata in HLS segments)      │
  │  → Zero extra connections.       │
  │  → Latency: same as video.      │
  │                                  │
  │  Option B: Lightweight SSE       │
  │  5-second poll to CDN-cached     │
  │  endpoint:                       │
  │  GET /reactions/{stream_id}      │
  │  → CDN caches for 5 seconds     │
  │  → Origin produces 1 response   │
  │    every 5 seconds               │
  │  → CDN serves it to 50M viewers  │
  │  → Origin load: ~1 req/5sec     │
  │                                  │
  └──────────────────────────────────┘

Result: 5M reactions/sec IN, but only 1 aggregated update/5sec OUT.
  Each viewer sees: "2.1M 🏏 reactions in the last 5 seconds!"
  + 20 randomly sampled chat messages scrolling on screen.
  
  Feels ALIVE and participatory, costs almost nothing to serve.
```

---

## 14. Server-Side Ad Insertion (SSAI)

### Why Server-Side, Not Client-Side

```
Client-side ad insertion (CSAI):
  Player fetches ad from ad server → plays ad → resumes stream.
  Problem: Ad blockers strip the ad request. 40%+ of desktop viewers block ads.
  Problem: Ad-to-content transition causes buffering (different servers, cold connection).

Server-side ad insertion (SSAI):
  Server splices ad INTO the video stream BEFORE delivering to viewer.
  From the player's perspective: it's all one continuous video stream.
  Ad blocker can't distinguish ad segments from content segments.
  No transition buffering (same CDN, same connection, same bitrate).
```

### SSAI Architecture

```
  Normal segment flow:
    Packager → seg_1235.ts (content) → CDN → Viewer

  During ad break:
    1. Admin triggers ad break: POST /admin/streams/{id}/ad-break
    2. Manifest Manipulation Service rewrites each viewer's manifest:
    
       BEFORE (content segments):
         seg_1235.ts  ← content
         seg_1236.ts  ← content
       
       AFTER (ad-break triggered):
         seg_1235.ts  ← content (last content segment)
         ad_seg_0.ts  ← 30-second ad, pre-encoded to match stream bitrate
         ad_seg_1.ts  ← (same ad, segment 2)
         ad_seg_2.ts  ← (same ad, segment 3)
         ...
         ad_seg_4.ts  ← (last ad segment)
         seg_1240.ts  ← content resumes

  Per-viewer personalization:
    User A (free tier): sees 30-second ad (generic ad from ad exchange)
    User B (premium):   sees no ad (ad segments replaced with content)
    User C (targeted):  sees specific ad (based on demographics)
    
    Each viewer gets a UNIQUE manifest URL (per-session token).
    Manifest Manipulation Service rewrites manifest per-viewer.
    But ad SEGMENTS are shared across viewers watching same ad
    → CDN caches ad segments just like content segments.
    
    Scale: 50M personalized manifests, but each manifest is <2 KB.
    Manifest server: 50M / 6 seconds = 8.3M manifest requests/sec.
    CDN caches manifests per-token group (same ad → same cache key).
    Actual origin: ~100K unique manifests/sec (grouping by ad assignment).
```

---

## 15. DVR / Rewind & Catch-Up

```
DVR window: 4 hours (viewer can rewind up to 4 hours during live stream)

Storage:
  7 quality tiers × 4 audio tracks × 1 segment/6sec × 4 hours:
  = 7 × 4 × 600 segments × 1.5 MB avg = ~25 GB per stream for 4-hour DVR window
  Stored on NVMe SSD attached to origin servers.

When viewer rewinds:
  1. Player requests older segments from playlist (seg_0100 instead of seg_1200)
  2. CDN may not have old segments cached (evicted by LRU)
  3. Request goes to origin → origin serves from SSD
  4. CDN caches it (for other viewers who rewind to same point)

  During a key moment (wicket, goal):
  Many viewers rewind to replay it simultaneously.
  This IS a thundering herd on old segments.
  
  Mitigation: Origin shield caches old segments too.
  First rewind request to shield → fetch from origin.
  Subsequent: served from shield cache.

Post-event:
  When stream ends, full DVR archive is uploaded to S3.
  Transcoded into VOD-optimized format (longer segments, more efficient encoding).
  Available as catch-up within 30 minutes of event end.
```

---

## 16. Authentication & Concurrency Control

### Session Management at Scale

```
50M concurrent viewers × session validation on every manifest request (every 6 seconds):
  = 8.3M session validations/sec

Cannot hit a database for each:

Solution: Signed token with embedded permissions.

  Token (JWT-like, but custom for performance):
    {
      user_id: 12345,
      subscription_tier: "premium",
      max_concurrent: 2,
      issued_at: 1713620000,
      expires_at: 1713634400,
      stream_permissions: ["live_sports"],
      signature: HMAC-SHA256(payload, rotating_secret)
    }
  
  Validation: Verify HMAC signature (CPU-only, no network call).
  ~1 microsecond per validation. 8.3M/sec = 8.3 CPU-seconds/sec. Trivial.
  
  Token issued at login, valid for 4 hours (duration of a match).
  Embedded in every manifest URL as query parameter.
  CDN edge validates token before serving segments.
  No per-request call to auth service.
```

### Concurrent Device Limit

```
Policy: Premium subscriber can watch on max 2 devices simultaneously.

Enforcement:

  1. Player sends heartbeat every 30 seconds:
       POST /v1/analytics/heartbeat { user_id, session_id, stream_id }
  
  2. Heartbeat service: ZADD sessions:{user_id} {timestamp} {session_id}
     Then: ZCARD sessions:{user_id}
     
     If count > 2:
       Oldest session (lowest timestamp) is evicted:
       ZPOPMIN sessions:{user_id}
       Push notification to evicted device: "Playback stopped — active on another device."
  
  3. Player on evicted device: next segment request includes token.
     Token is still valid (not revoked), but session_id is no longer in Redis.
     Heartbeat returns: { "status": "evicted", "reason": "concurrent_limit" }
     Player shows: "You've been signed out because you started watching on another device."

  Scale: 50M heartbeats every 30 seconds = 1.67M writes/sec to Redis.
  Each ZADD + ZCARD: ~2 microseconds. Total: 3.3 CPU-seconds/sec.
  One Redis cluster handles this effortlessly.
```

---

## 17. Database & Storage Architecture

```
┌────────────────────┬────────────────────┬────────────────────────────────┐
│ Data               │ Storage            │ Why                            │
├────────────────────┼────────────────────┼────────────────────────────────┤
│ Event / Stream     │ PostgreSQL         │ Low volume, relational,        │
│ metadata           │                    │ queried by admins              │
├────────────────────┼────────────────────┼────────────────────────────────┤
│ User accounts      │ PostgreSQL         │ Standard user CRUD             │
│ Subscriptions      │ (sharded by        │                                │
│                    │  user_id)          │                                │
├────────────────────┼────────────────────┼────────────────────────────────┤
│ Live segments      │ RAM → NVMe → S3    │ Hot: RAM for latest segments   │
│ (video files)      │ (tiered)           │ Warm: SSD for DVR window       │
│                    │                    │ Cold: S3 for VOD archive       │
├────────────────────┼────────────────────┼────────────────────────────────┤
│ DRM keys           │ HSM-backed KMS     │ Hardware security module for   │
│                    │                    │ content key storage            │
├────────────────────┼────────────────────┼────────────────────────────────┤
│ Viewer analytics   │ Kafka → Flink →    │ 50M heartbeats/30sec =        │
│ (heartbeats)       │ ClickHouse         │ 1.67M events/sec.             │
│                    │                    │ Time-series OLAP.             │
├────────────────────┼────────────────────┼────────────────────────────────┤
│ Chat / Reactions   │ Kafka → Flink →    │ Write-heavy, aggregated.       │
│                    │ Redis (real-time)  │ Don't need per-message         │
│                    │                    │ persistence.                   │
├────────────────────┼────────────────────┼────────────────────────────────┤
│ Session state      │ Redis              │ Concurrent device tracking,    │
│                    │                    │ CDN routing, real-time counts  │
├────────────────────┼────────────────────┼────────────────────────────────┤
│ Viewer count       │ Redis (HyperLogLog)│ 12 KB per stream, ±0.81%     │
│                    │                    │ accuracy. 50M counted in 12KB.│
└────────────────────┴────────────────────┴────────────────────────────────┘
```

---

## 18. Caching Architecture

```
Live streaming has the BEST cache characteristics of any workload:
  - ALL 50M viewers want the EXACT SAME content
  - Content is immutable once produced (segment 1235 never changes)
  - Content is small (1-3 MB per segment)
  - Content has natural TTL (live window moves forward)

Cache hit rate: 99.99%+ at CDN edge
  Each edge serves ~100K viewers. All want the same segment.
  Edge fetches once from shield → serves 100K times from cache.
  
  Cache hit rate breakdown:
    CDN Edge:     99.99%  (segment cached, served locally)
    Origin Shield: 99.9%  (segment cached at shield level)
    Origin:        0.01%  (only first request per segment per shield)

  This means origin handles:
    ~3 shields × 7 quality tiers × 1 request/6 seconds = ~3.5 req/sec
    The entire 50M viewer workload generates 3.5 requests per second to origin.
    You could run origin on a Raspberry Pi. (Don't actually do this.)

What ISN'T cacheable (and must scale differently):
  - Manifest requests (per-viewer, personalized for ads) → Manifest CDN
  - DRM license requests (per-device, one-time) → License server fleet
  - Heartbeats (per-viewer) → Kafka → analytics
  - Chat messages (per-viewer) → aggregated, CDN-cached summary
```

---

## 19. Scalability Deep Dive

### The Ramp-Up Problem

```
IPL Final: Toss at 7:00 PM, match starts 7:30 PM.

  6:00 PM:   5M viewers (pre-match show)
  7:00 PM:   15M viewers (toss happened, interest spikes)
  7:30 PM:   35M viewers (match starts)
  7:45 PM:   50M viewers (everyone tuned in)
  8:30 PM:   45M viewers (first innings settling)
  9:15 PM:   55M viewers (second innings chase → peak tension)
  10:00 PM:  30M viewers (match ending / ended)
  10:30 PM:  5M viewers (post-match)

  The system must scale from 5M → 55M viewers in ~3 hours.
  But the CDN handles this naturally:
    - More viewers at an edge PoP = more requests to that PoP
    - But content is the SAME = cache hit rate stays 99.99%
    - Edge bandwidth scales linearly with viewers (more TCP connections)
    - No new origin load (still ~3.5 req/sec regardless of viewer count)

  What DOES need to scale:
    1. Manifest Manipulation (SSAI): scales with viewer count (each needs unique manifest)
    2. DRM License Server: spike at stream start (all viewers request license)
    3. Heartbeat Ingestion: scales linearly with viewers
    4. Chat Ingestion: spikes at key moments (6, wicket, goal)
    5. Session Management: scales linearly

  What DOESN'T need to scale:
    1. Origin (content serving): constant regardless of viewer count
    2. Encoding/Transcoding: constant (one input stream, always encoding)
    3. Database: rarely touched during live stream
    4. CDN edge: auto-scales (provider's responsibility)
```

### Pre-Warming Strategy

```
Before a mega-event:

T-24h:  Provision dedicated encoding/transcoding GPU fleet
T-12h:  Pre-warm CDN: push test segments to all edge PoPs
         (forces CDN to establish connections to origin shields)
T-6h:   Scale up manifest manipulation service (10x normal)
T-2h:   Scale up DRM license server (100x normal)
T-1h:   Scale up heartbeat ingestion (Kafka partitions, Flink parallelism)
T-30m:  Start test stream (internal only) — validates entire pipeline end-to-end
T-15m:  Open pre-match stream to viewers — gradual ramp-up
T-0:    Main event starts. All systems pre-warmed and at capacity.

Key: NEVER go from cold to 50M. Always ramp gradually.
```

---

## 20. Reliability & Fault Tolerance Deep Dive

### Failure Scenarios

```
┌──────────────────────────────┬───────────────────────────────────────────────────┐
│ Failure                      │ Impact & Mitigation                               │
├──────────────────────────────┼───────────────────────────────────────────────────┤
│ Camera / broadcast truck     │ Backup camera feed auto-switches (broadcast       │
│ failure                      │ truck has redundant inputs). Viewers see brief     │
│                              │ angle change. Ingest layer unaffected.            │
├──────────────────────────────┼───────────────────────────────────────────────────┤
│ Primary ingest link fails    │ SRT session on backup link takes over in < 1s.    │
│ (network between stadium     │ Viewers see 1-2 frame glitch at most.             │
│  and cloud)                  │ Segment boundary alignment ensures seamless join. │
├──────────────────────────────┼───────────────────────────────────────────────────┤
│ Primary encoder crashes      │ Backup encoder (active-active) immediately takes  │
│                              │ over. Segment Selector picks backup's output.     │
│                              │ Viewers see nothing (segments are identical).     │
├──────────────────────────────┼───────────────────────────────────────────────────┤
│ Origin server crashes        │ Origin shield has cached the last ~30 segments.   │
│                              │ Shield serves from cache for ~30 seconds.         │
│                              │ Standby origin promoted (< 5 seconds).            │
│                              │ If standby also fails: shield has 3 minutes of    │
│                              │ DVR content cached. Time to spin up new origin.   │
├──────────────────────────────┼───────────────────────────────────────────────────┤
│ One CDN provider goes down   │ Multi-CDN: traffic shifted to remaining CDNs.     │
│ (e.g., Akamai incident)     │ Player-side fallback: try alternate CDN URL.       │
│                              │ CDN router (DNS/manifest level) removes bad CDN.  │
│                              │ Remaining CDNs absorb extra load (~50% more each).│
│                              │ Pre-provisioned headroom: each CDN can handle 60%.│
├──────────────────────────────┼───────────────────────────────────────────────────┤
│ ISP peering congestion       │ ISP-embedded caches bypass the peering point.     │
│ (Jio ↔ CDN backbone)        │ If ISP cache itself is overwhelmed:               │
│                              │  - Player ABR drops to lower bitrate (less BW)    │
│                              │  - 720p → 480p for affected users                │
│                              │  - Buffering ratio increases but stream continues │
├──────────────────────────────┼───────────────────────────────────────────────────┤
│ DRM license server overload  │ License pre-fetch (during loading screen) spreads │
│ (50M simultaneous requests)  │ load. License cached on device for 4 hours.       │
│                              │ If server down: viewers with cached license OK.   │
│                              │ New viewers: "Loading..." until recovered.         │
│                              │ Emergency: serve license from CDN-cached blob     │
│                              │ (less secure but keeps stream playing).           │
├──────────────────────────────┼───────────────────────────────────────────────────┤
│ Regional power outage        │ Viewers in affected area go offline (unavoidable).│
│ (e.g., grid failure in       │ CDN edge in that region powers down.              │
│  major Indian city)          │ No impact on other regions.                       │
│                              │ When power returns: viewers rejoin, player         │
│                              │ resumes from live edge. DVR available for catch-up.│
├──────────────────────────────┼───────────────────────────────────────────────────┤
│ Manifest Manipulation        │ Fall back to generic manifest (no ad insertion).   │
│ Service failure              │ Viewers see content without personalized ads.     │
│                              │ Revenue impact but no viewer experience impact.   │
│                              │ Ad revenue loss: ~$0.50/viewer/hour × N viewers.  │
├──────────────────────────────┼───────────────────────────────────────────────────┤
│ Chat/Reaction service down   │ Stream continues perfectly (chat is separate).    │
│                              │ Reaction overlay disappears from UI.              │
│                              │ Non-critical. Fix at leisure.                     │
└──────────────────────────────┴───────────────────────────────────────────────────┘
```

### Graceful Degradation Tiers

```
As system load increases or components fail, shed non-critical features:

Tier 0: Full Experience (normal operation)
  ✓ All quality tiers (4K to 360p)
  ✓ Live chat + reactions
  ✓ Personalized ads (SSAI)
  ✓ DVR / rewind
  ✓ Multi-language audio
  ✓ Scoreboard overlay

Tier 1: High Load (50M+ viewers, some CDN pressure)
  ✓ All quality tiers
  ✓ Reactions (aggregated only, no individual chat)
  ✓ Generic ads (not personalized — reduces manifest computation)
  ✓ DVR limited to 1 hour (instead of 4)
  ✓ Multi-language audio
  ✓ Scoreboard overlay

Tier 2: Extreme Load (CDN approaching capacity)
  ✓ Quality tiers: 1080p and below (drop 4K — saves 8 Tbps)
  ✗ Chat/Reactions disabled
  ✓ No ad insertion (pure content stream)
  ✓ DVR disabled (saves origin storage I/O)
  ✓ Primary language audio only (drop 3 of 4 audio tracks)
  ✓ Scoreboard overlay

Tier 3: Emergency (major infrastructure failure)
  ✓ Quality tiers: 720p and below only
  ✗ Everything non-essential disabled
  ✓ Single audio track
  ✓ No DVR
  ✓ Static scoreboard (updated manually, not real-time)
  Goal: Keep the stream playing for as many viewers as possible.

Each tier sheds ~30-40% of infrastructure load.
Tier 0 → Tier 3 reduces total bandwidth from 150 Tbps to ~40 Tbps.
```

---

## 21. Observability & Operational Excellence

### War Room Dashboard (During Live Event)

```
┌─────────────────────────────────────────────────────────────────────┐
│          IPL FINAL — LIVE   |   Match: 2nd Innings, Over 15.3      │
│          Started: 7:30 PM   |   Elapsed: 2h 15m                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  VIEWERS                         STREAM HEALTH                      │
│  ├─ Concurrent:    51,234,000    ├─ Buffering ratio:  0.08% ✓      │
│  ├─ Peak today:    54,102,000    ├─ Avg start time:   1.8s ✓       │
│  ├─ Avg bitrate:   2.8 Mbps     ├─ Error rate:       0.02% ✓      │
│  └─ Audio-only:    1,200,000    ├─ Avg latency:      12s ✓        │
│                                  └─ Dropped frames:   0.1% ✓       │
│  QUALITY DISTRIBUTION            CDN HEALTH                         │
│  ├─ 4K:      1.1M  ( 2.1%)     ├─ Akamai:     ✓  err: 0.01%     │
│  ├─ 1080p:   7.8M  (15.2%)     ├─ CloudFront: ✓  err: 0.02%     │
│  ├─ 720p:   18.2M  (35.5%)     ├─ ISP-Jio:    ✓  err: 0.03%     │
│  ├─ 480p:   15.1M  (29.5%)     ├─ ISP-Airtel: ✓  err: 0.01%     │
│  ├─ 360p:    7.8M  (15.2%)     └─ Total BW:    108 Tbps          │
│  └─ Audio:   1.2M  ( 2.5%)                                        │
│                                  ENCODING                           │
│  GEOGRAPHIC                      ├─ Pipeline:    ✓  healthy       │
│  ├─ India:   48.2M  (94%)       ├─ Segment lag:  0.2s ✓           │
│  ├─ US/UK:    1.8M  ( 3%)       ├─ Primary enc:  ✓  active       │
│  └─ Other:    1.2M  ( 3%)       └─ Backup enc:   ✓  standby      │
│                                                                     │
│  ALERTS                                                             │
│  └─ ✓ All systems nominal                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Key SLIs (Service Level Indicators)

```
┌─────────────────────────────┬──────────┬──────────────────────────────┐
│ Metric                      │ Target   │ Measurement                  │
├─────────────────────────────┼──────────┼──────────────────────────────┤
│ Video Start Time            │ < 2s p95 │ Player telemetry             │
│ Buffering Ratio             │ < 0.5%   │ Player telemetry (% of time) │
│ Stream Availability         │ 99.99%   │ Synthetic monitors + RUM     │
│ Glass-to-Glass Latency      │ < 15s    │ Reference clock in video     │
│ Segment Fetch Error Rate    │ < 0.1%   │ CDN logs + player telemetry  │
│ Manifest Latency            │ < 100ms  │ CDN edge measurement         │
│ DRM License Latency         │ < 500ms  │ License server metrics       │
│ Concurrent Viewer Accuracy  │ ±1%      │ HyperLogLog vs sampled count │
└─────────────────────────────┴──────────┴──────────────────────────────┘
```

---

## 22. Corner Cases & Hard Problems

### 1. Thundering Herd at Match Start (0 → 30M in 5 Minutes)

```
Problem: Match starts at 7:30 PM. At 7:29 PM, 30M users press Play simultaneously.
  Each viewer needs:
    1. Manifest (personalized)      → 30M manifest requests
    2. DRM license                  → 30M license requests
    3. First 3-4 video segments     → 30M × 4 = 120M segment requests
  Total: ~180M requests in ~60 seconds = 3M requests/sec

CDN can handle segment requests (cached).
But manifests and DRM licenses are NOT cached (per-viewer).

Solution:
  1. Pre-stream loading: Start returning manifests 5 minutes before match starts.
     Show pre-match content / countdown. This spreads the 30M over 5 minutes
     instead of 1 minute → 6x reduction in peak.
     
  2. DRM license pre-fetch: Issue license during pre-match phase.
     By 7:30 PM, 80% of viewers already have their license.
     Only 6M new viewers need licenses in the first minute.
     
  3. Manifest caching by group: Group viewers by (subscription_tier, region, device).
     ~50 unique groups. Cache manifest per group for 3 seconds.
     30M viewers → 50 unique manifests × 1 req/3sec = ~17 req/sec to origin.
     
  4. CDN pre-warming: Push first few segments to all edge PoPs before match starts.
     When 30M viewers request their first segment: instant cache hit.
```

### 2. Wicket Falls / Goal Scored — Viral Spike

```
Problem: Star batsman gets out. Social media explodes.
  5M additional viewers tune in within 60 seconds to see the replay.
  
  These are COLD viewers: no cached manifest, no DRM license, no buffered segments.
  5M cold starts in 60 seconds = 83K new viewer setups/sec.

  Each cold start: manifest + DRM license + 3 segments = 5 requests.
  Total: 83K × 5 = 415K additional req/sec suddenly.

Mitigation:
  - CDN absorbs segment requests (already cached for existing 50M viewers).
  - DRM license server auto-scales (stateless, can add pods in 30s).
  - Manifest service auto-scales.
  - Player "fast start": fetch the LOWEST quality first segment (360p, ~100KB).
     Start playing in < 1 second.
     ABR steps up to 720p/1080p over next 10 seconds as bandwidth is measured.
     User sees stream immediately (low quality) rather than buffering at high quality.
```

### 3. Cell Tower Congestion (India-Specific)

```
Problem: 50M viewers, many on mobile. A cell tower serves ~1000 users.
  If 200 of those 1000 users are streaming at 2.5 Mbps:
  200 × 2.5 Mbps = 500 Mbps downstream demand.
  Tower backhaul capacity: ~1 Gbps shared across all services (not just Hotstar).
  Other users on the tower see degraded internet.

Impact on Hotstar:
  - Effective bandwidth per viewer drops from 2.5 Mbps to 0.5-1 Mbps.
  - ABR kicks in: quality drops to 360p-480p for all viewers on that tower.
  - Buffering increases.

Platform-level mitigation:
  1. ISP-embedded caches: Serve from within the ISP network.
     Traffic between ISP cache and cell tower uses internal backbone
     (not internet peering), which has more capacity.
     
  2. Audio-only fallback: For the worst-connected 2-3%, offer audio-only stream.
     128 Kbps audio vs 2.5 Mbps video = 20x bandwidth reduction per viewer.
     Player shows: "Low connectivity. Switch to audio-only?"
     
  3. Data saver mode: Pre-selected by user.
     Forces 360p max regardless of bandwidth measurement.
     Reduces per-viewer bandwidth by 75% (2.5 Mbps → 600 Kbps).
```

### 4. Encoder Produces a Corrupt Segment

```
Problem: GPU glitch causes one segment to contain garbage pixels.
  Segment pushed to origin, cached on CDN, delivered to 50M viewers.
  50M viewers simultaneously see 6 seconds of garbled video.

Detection:
  1. Post-encode quality check: VMAF/SSIM score computed on each segment.
     If score < threshold: segment marked as corrupt.
  2. Player telemetry: spike in decoder errors across all viewers simultaneously.
  3. Automated alert: > 1% of viewers reporting decode errors in 10-second window.

Mitigation:
  1. Immediate: Serve backup encoder's segment instead.
     Origin replaces corrupt segment with backup's version.
     CDN cache invalidation for that specific segment URL (CDN purge API).
     
  2. Viewers who already downloaded corrupt segment:
     Most players handle decode errors by skipping corrupted frames.
     Viewer sees a brief glitch (1-6 seconds) then resumes normally.
     
  3. Prevention: Backup encoder runs in parallel.
     Segment Selector runs VMAF on both outputs.
     Only publishes the one with higher quality score.
     If both corrupt: hold segment and extend previous segment's duration
     (player handles variable segment length gracefully).
```

### 5. Time Zone Difference — Staggered Global Peak

```
Problem: FIFA World Cup Final — global audience.
  India peak (evening local time) ≠ Europe peak ≠ Americas peak.
  
  Kickoff 8 PM IST = 2:30 PM GMT = 9:30 AM EST
  
  India viewers (40M) + Europe viewers (10M) + Americas viewers (5M) = 55M
  
  But peak times OVERLAP (match is 2 hours, all regions are "awake"):
  Unlike a TV show that airs at different times per region.

CDN implication:
  - India CDN PoPs: 70% of traffic
  - Europe PoPs: 20% of traffic
  - Americas PoPs: 10% of traffic
  
  Multi-CDN traffic weights are shifted PER REGION:
    India: 50% ISP-embedded + 30% Akamai + 20% CloudFront
    Europe: 60% Akamai + 40% CloudFront
    Americas: 50% CloudFront + 50% Akamai
  
  Each region's CDN mix optimized for that region's ISP topology.
```

### 6. Mid-Stream DRM Key Rotation

```
Problem: DRM keys rotate every 5 minutes (security policy).
  50M viewers must seamlessly transition to new key without interruption.

  If key server is down during rotation: viewers can't decrypt new segments.
  50M viewers see black screen in 5 minutes.

Mitigation:
  1. Overlap period: New key active for 30 seconds before old key expires.
     Segments encrypted with new key. Players request new license.
     Spread: not all players request at the same instant
     (rotation time is per-viewer based on when they joined).
     
  2. Key pre-delivery: Player fetches next key 60 seconds before rotation.
     Stored locally. When rotation happens: zero network call needed.
     
  3. Fallback: If key server is down during rotation:
     Extend current key validity by 30 minutes (server-side configuration).
     All segments continue with current key. Players don't notice.
     Security trade-off: key exposed 30 min longer. Acceptable during outage.
```

### 7. "Last Ball" Scenario — Maximum Tension, Maximum Viewers

```
Problem: Cricket World Cup Final, last ball of the match.
  Entire country is watching. Social media is on fire.
  
  Viewer count: 59M concurrent (Hotstar's actual recorded peak).
  Reaction rate: 10M reactions in 5 seconds (everyone taps the screen).
  
  AND: 2M additional viewers joining in the last 30 seconds
  (people who weren't watching, saw social media, opened the app).
  
  These 2M new viewers ALL need:
  - Manifest → license → first segment → play
  In < 3 seconds (they'll miss the moment if it takes longer).

  This is the MOST extreme load combination:
  - Peak viewers (59M) → peak CDN bandwidth (~165 Tbps)
  - Peak reactions (10M/5sec) → peak ingestion
  - Peak cold starts (2M in 30 sec) → peak manifest + license + segment
  - Peak chat (millions of messages) → peak aggregation

Solution: This scenario IS the design target.
  Every component is provisioned for this exact moment.
  
  Pre-warming handled the base 57M viewers.
  The 2M spike: CDN handles segment delivery (cached).
  Manifest + DRM: pre-provisioned at 100x normal capacity.
  Player fast-start: lowest quality first segment in < 1 second.
  
  The viewer joins, sees 360p for 2 seconds, ABR steps up to 720p,
  and they see the last ball delivered in near-real-time.
  
  This is why live streaming at this scale requires
  6+ months of capacity planning per major event.
```

---

## Summary: Key Design Decisions

| Decision | Choice | Why |
|---|---|---|
| Ingest protocol | SRT (not RTMP) | UDP-based, FEC for packet loss, lower latency |
| Encoding | Active-active dual encoders with segment selector | Zero-gap failover on encoder crash |
| Segment format | HLS + DASH, 6s segments (standard), 2s partials (LL-HLS) | Universal device support, ~8-15s live latency |
| Origin scaling | Origin shield (3 nodes) between origin and CDN | Reduces origin load from 5K req/sec to ~3 req/sec |
| CDN strategy | Multi-CDN (Akamai + CloudFront + ISP-embedded) | No single CDN has 150 Tbps; ISP caches bypass peering bottleneck |
| ISP-embedded caches | Cache servers inside Jio/Airtel network | 80% of traffic never leaves viewer's ISP; eliminates peering as bottleneck |
| ABR algorithm | Buffer-based (BOLA-like) with emergency low-quality fallback | India's extreme network diversity requires aggressive downshifting |
| DRM | Multi-DRM (Widevine + FairPlay + PlayReady) with pre-fetch | License at stream start spread over pre-match period |
| Ad insertion | SSAI (server-side) | Ad-blocker proof, no client-side buffering at ad boundary |
| Chat/Reactions | Aggregate + sample (not per-message broadcast) | 5M reactions/sec impossible to broadcast; aggregated summary is sufficient |
| Viewer count | Redis HyperLogLog | 50M unique viewers tracked in 12 KB with ±0.81% accuracy |
| Session/auth | Signed token (HMAC), validated at CDN edge | 8.3M validations/sec with zero network calls |
| Concurrent device | Redis sorted set with heartbeat TTL | 1.67M writes/sec; automatic stale session eviction |
| Graceful degradation | 4-tier shedding (4K → chat → DVR → audio tracks) | Each tier sheds ~30-40% load; preserves core streaming |
| Segment caching | 99.99% CDN cache hit rate (all viewers watch same content) | Live streaming's unique advantage: identical content for everyone |