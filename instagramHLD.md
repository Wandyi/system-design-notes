# Instagram — Comprehensive High-Level Design

## Table of Contents

1. [Requirements & Scale](#1-requirements--scale)
2. [Back-of-Envelope Estimation](#2-back-of-envelope-estimation)
3. [High-Level Architecture](#3-high-level-architecture)
4. [Core API Design](#4-core-api-design)
5. [Data Models](#5-data-models)
6. [Media Upload & Processing Pipeline](#6-media-upload--processing-pipeline)
7. [Feed Generation System](#7-feed-generation-system)
8. [Social Graph Service](#8-social-graph-service)
9. [Stories & Reels Subsystem](#9-stories--reels-subsystem)
10. [Search & Explore](#10-search--explore)
11. [Notifications Pipeline](#11-notifications-pipeline)
12. [Direct Messaging](#12-direct-messaging)
13. [Database & Storage Architecture](#13-database--storage-architecture)
14. [Caching Architecture](#14-caching-architecture)
15. [Scalability Deep Dive](#15-scalability-deep-dive)
16. [Reliability & Fault Tolerance Deep Dive](#16-reliability--fault-tolerance-deep-dive)
17. [Observability & Operational Excellence](#17-observability--operational-excellence)
18. [Corner Cases & Hard Problems](#18-corner-cases--hard-problems)

---

## 1. Requirements & Scale

### Functional Requirements

- **Post**: Upload photos/videos with caption, tags, location
- **Feed**: Personalized, ranked home feed from followed accounts
- **Stories**: Ephemeral 24-hour photo/video content
- **Reels**: Short-form video (up to 90 seconds)
- **Social Graph**: Follow/unfollow, followers list, following list
- **Explore**: Discovery page with trending/recommended content
- **Search**: Users, hashtags, locations
- **Notifications**: Likes, comments, follows, mentions, DM alerts
- **Direct Messaging**: 1:1 and group messaging with media support
- **Likes & Comments**: On posts, stories, reels
- **Live**: Real-time video broadcasting (mentioned but not deep-dived)

### Non-Functional Requirements

- **Availability**: 99.99% (< 52 minutes downtime/year)
- **Latency**: Feed load < 200ms p50, < 500ms p99
- **Consistency**: Eventual consistency for feed/likes (seconds); strong consistency for DMs and follow graph mutations
- **Durability**: Zero data loss for uploaded media (11 nines)
- **Scale**: 2B+ monthly active users, 500M+ daily active users

### Out of Scope (for this doc)

- Ads platform / ad ranking
- Creator monetization / shopping
- Live video infrastructure (WebRTC/RTMP)
- Content moderation ML models (referenced but not designed)
- GDPR data deletion pipeline internals

---

## 2. Back-of-Envelope Estimation

```
Users:
  MAU:                 2 billion
  DAU:                 500 million
  Peak concurrent:     ~50 million

Content Creation:
  New posts/day:       ~100 million
  New stories/day:     ~500 million
  New reels/day:       ~50 million
  Average photo size:  ~2 MB (original), ~200 KB (compressed/resized)
  Average video size:  ~15 MB (reel), ~5 MB (story video)
  Average post photos: 3 (carousel)

Storage (daily new):
  Photos:   100M posts × 3 photos × 2 MB = 600 TB/day (originals)
  Stories:  500M × 1 MB avg = 500 TB/day
  Reels:    50M × 15 MB = 750 TB/day
  Total:    ~1.85 PB/day of new media
  Per year: ~675 PB (before replication)

Read Traffic:
  Feed loads/day:      DAU × 10 sessions × 20 posts = 100 billion post reads/day
  Feed QPS:            100B / 86400 ≈ 1.15 million reads/sec (average)
  Peak QPS:            ~3-5 million reads/sec (2-3x average)

Write Traffic:
  Post writes/sec:     100M / 86400 ≈ 1,150/sec
  Like writes/sec:     ~5 billion likes/day ÷ 86400 ≈ 58,000/sec
  Comment writes/sec:  ~500M / 86400 ≈ 5,800/sec
  Follow mutations:    ~50M / 86400 ≈ 580/sec

Bandwidth:
  Egress (feed serving): 5M peak QPS × 200 KB avg = 1 TB/sec peak egress
  Ingress (uploads):     ~21 GB/sec average
```

**Key Insight**: The read-to-write ratio is ~1000:1 for feed reads vs post writes. The system is overwhelmingly read-heavy, which drives every architectural decision.

---

## 3. High-Level Architecture

```
                                         ┌──────────────────┐
                                         │   DNS (Route 53) │
                                         └────────┬─────────┘
                                                  │
                                         ┌────────▼─────────┐
                                         │       CDN        │
                                         │ (CloudFront /    │
                                         │  Akamai / Meta   │
                                         │  Edge Network)   │
                                         └────────┬─────────┘
                                                  │
                                   ┌──────────────┴─────────────┐
                                   │                            │
                          ┌────────▼────────┐          ┌────────▼────────┐
                          │   API Gateway   │          │  WebSocket LB   │
                          │ (REST / GraphQL)│          │  (DMs, Live)    │
                          └────────┬────────┘          └────────┬────────┘
                                   │                            │
                    ┌──────────────┼──────────────┐             │
                    │              │              │             │
            ┌───────▼──────┐ ┌────▼─────┐ ┌─────▼──────┐ ┌────▼──────┐
            │ Post Service │ │  Feed    │ │  User /    │ │  Messaging│
            │              │ │  Service │ │  Social    │ │  Service  │
            └───────┬──────┘ └────┬─────┘ │  Graph     │ └────┬──────┘
                    │             │       └─────┬──────┘      │
         ┌──────────┤             │             │             │
         │          │             │             │             │
   ┌─────▼──────┐   │      ┌──────▼─────┐ ┌────▼──────┐      │
   │   Media    │   │      │  Feed      │ │  Graph    │      │
   │ Processing │   │      │  Cache     │ │  Cache    │      │
   │  Pipeline  │   │      │  (Redis)   │ │ (Redis)   │      │
   └─────┬──────┘   │      └──────┬─────┘ └────┬──────┘      │
         │          │             │            │             │
   ┌─────▼──────┐   │      ┌──────▼─────────────▼──────┐      │
   │  Object    │   │      │       PostgreSQL /         │      │
   │  Storage   │   │      │       Cassandra /          │◄─────┘
   │  (S3)      │   │      │       DynamoDB Clusters    │
   └────────────┘   │      └───────────────────────────┘
                    │
         ┌──────────▼──────────────────────────┐
         │           Event Bus                  │
         │         (Kafka)                      │
         └──┬────────┬────────┬────────┬───────┘
            │        │        │        │
      ┌─────▼──┐ ┌───▼───┐ ┌──▼──┐ ┌──▼────────┐
      │Notif.  │ │Search │ │Feed │ │Analytics  │
      │Service │ │Indexer│ │Fan- │ │Pipeline   │
      │        │ │(ES)   │ │Out  │ │(Spark)    │
      └────────┘ └───────┘ └─────┘ └───────────┘
```

### Service Decomposition

| Service | Responsibility | Primary Data Store |
|---|---|---|
| **Post Service** | CRUD for posts, comments, likes | PostgreSQL (sharded by postID) |
| **Feed Service** | Feed assembly, ranking, pagination | Redis (precomputed feeds) + Cassandra (fallback) |
| **User / Social Graph** | Profiles, follow/unfollow, follower counts | PostgreSQL (profiles) + TAO/graph cache |
| **Media Processing** | Upload, transcode, resize, thumbnail generation | S3 + message queue |
| **Notification Service** | Push, in-app, email notifications | Cassandra (notification log) + APNs/FCM |
| **Search Service** | User/hashtag/location search | Elasticsearch |
| **Messaging Service** | DMs, group chats, read receipts | Cassandra (messages) + Redis (presence) |
| **Stories Service** | Ephemeral content with 24h TTL | Redis (hot) + Cassandra (warm) |
| **Explore/Recommendation** | Content discovery, trending | ML ranking service + feature store |

---

## 4. Core API Design

### Post APIs

```
POST   /v1/posts
  Body: { media_ids: [uuid], caption: string, location?: {lat, lng, name},
          tags?: [user_id], hashtags?: [string] }
  → 201 { post_id, created_at, media_urls: [...] }

GET    /v1/posts/{post_id}
  → 200 { post_id, user_id, media_urls, caption, like_count, comment_count,
          created_at, location, tags }

DELETE /v1/posts/{post_id}
  → 204 No Content

POST   /v1/posts/{post_id}/like
  → 200 { liked: true, like_count }     (idempotent)

DELETE /v1/posts/{post_id}/like
  → 200 { liked: false, like_count }

POST   /v1/posts/{post_id}/comments
  Body: { text: string, reply_to?: comment_id }
  → 201 (Created) { comment_id, text, user_id, created_at }

GET    /v1/posts/{post_id}/comments?cursor=X&limit=20
  → 200 { comments: [...], next_cursor }
```

### Feed APIs

```
GET    /v1/feed?cursor=X&limit=20
  → 200 { posts: [...enriched post objects...], next_cursor, ranking_trace_id }
  Headers: X-Feed-Session-ID (for ranking consistency within a session)

GET    /v1/explore?cursor=X&limit=30&category=?
  → 200 { items: [...], next_cursor }
```

### Social Graph APIs

```
POST   /v1/users/{user_id}/follow
  → 200 { following: true }

DELETE /v1/users/{user_id}/follow
  → 200 { following: false }

GET    /v1/users/{user_id}/followers?cursor=X&limit=50
  → 200 { users: [...], next_cursor, total_count }

GET    /v1/users/{user_id}/following?cursor=X&limit=50
  → 200 { users: [...], next_cursor, total_count }

GET    /v1/users/{user_id}/profile
  → 200 { user_id, username, full_name, bio, avatar_url,
          post_count, follower_count, following_count, is_private }
```

### Media Upload APIs

```
POST   /v1/media/upload/init
  Body: { content_type: "image/jpeg", file_size: 2048000, context: "post" }
  → 200 { media_id, upload_url, upload_headers, expires_at }
  (Returns a pre-signed S3 URL — client uploads directly to S3)

POST   /v1/media/upload/{media_id}/complete
  → 200 { media_id, status: "processing", thumbnail_url }

GET    /v1/media/{media_id}/status
  → 200 { media_id, status: "ready"|"processing"|"failed", variants: {...} }
```

### Stories APIs

```
POST   /v1/stories
  Body: { media_id, stickers?: [...], mentions?: [...] }
  → 201 { story_id, expires_at }

GET    /v1/stories/feed
  → 200 { story_trays: [ { user_id, username, avatar_url, stories: [...], seen: bool } ] }

POST   /v1/stories/{story_id}/seen
  → 204
```

### Search API

```
GET    /v1/search?q=keyword&type=user|hashtag|location&cursor=X&limit=20
  → 200 { results: [...], next_cursor }
```

### Pagination Strategy

All list endpoints use **cursor-based pagination** (not offset-based):

```
Why not OFFSET:
  OFFSET 10000, LIMIT 20 → DB scans and discards 10,000 rows
  At scale: OFFSET grows linearly slower, inconsistent with real-time inserts

Cursor-based:
  cursor = opaque encoded (timestamp, id) pair
  WHERE (created_at, id) < (cursor_ts, cursor_id) ORDER BY created_at DESC LIMIT 20
  Constant performance regardless of page depth
  Stable under concurrent inserts (no skips or duplicates)
```

---

## 5. Data Models

### PostgreSQL — User Service (sharded by user_id)

```sql
-- Users table (one per user, ~2B rows)
CREATE TABLE users (
    user_id       BIGINT PRIMARY KEY,       -- snowflake ID
    username      VARCHAR(30) UNIQUE,
    full_name     VARCHAR(150),
    bio           TEXT,
    avatar_url    TEXT,
    email         VARCHAR(255),
    phone         VARCHAR(20),
    is_private    BOOLEAN DEFAULT false,
    is_verified   BOOLEAN DEFAULT false,
    post_count    INT DEFAULT 0,
    follower_count INT DEFAULT 0,            -- denormalized counter
    following_count INT DEFAULT 0,           -- denormalized counter
    created_at    TIMESTAMPTZ DEFAULT NOW(),
    updated_at    TIMESTAMPTZ DEFAULT NOW()
);
```

### PostgreSQL — Post Service (sharded by post_id)

```sql
-- Posts table (~50B+ rows cumulative)
CREATE TABLE posts (
    post_id       BIGINT PRIMARY KEY,       -- snowflake ID
    user_id       BIGINT NOT NULL,
    caption       TEXT,
    location_id   BIGINT,
    like_count    INT DEFAULT 0,             -- denormalized, updated async
    comment_count INT DEFAULT 0,             -- denormalized, updated async
    media_type    SMALLINT,                  -- 1=photo, 2=video, 3=carousel
    is_archived   BOOLEAN DEFAULT false,
    created_at    TIMESTAMPTZ DEFAULT NOW()
);
-- Secondary index for user profile grid
CREATE INDEX idx_posts_user_time ON posts (user_id, created_at DESC);

-- Post media (photos in a carousel)
CREATE TABLE post_media (
    media_id      BIGINT PRIMARY KEY,
    post_id       BIGINT NOT NULL,
    media_url     TEXT NOT NULL,
    thumbnail_url TEXT,
    media_type    SMALLINT,
    width         INT,
    height        INT,
    ordering      SMALLINT DEFAULT 0
);
CREATE INDEX idx_postmedia_post ON post_media (post_id);
```

### Cassandra — Likes (write-heavy, ~58K writes/sec)

```sql
-- Likes by post (who liked this post?)
CREATE TABLE likes_by_post (
    post_id     BIGINT,
    user_id     BIGINT,
    created_at  TIMESTAMP,
    PRIMARY KEY (post_id, created_at, user_id)
) WITH CLUSTERING ORDER BY (created_at DESC, user_id ASC);

-- Likes by user (what did this user like?)
CREATE TABLE likes_by_user (
    user_id     BIGINT,
    post_id     BIGINT,
    created_at  TIMESTAMP,
    PRIMARY KEY (user_id, created_at, post_id)
) WITH CLUSTERING ORDER BY (created_at DESC, post_id ASC);

-- Both tables are written to atomically via Kafka consumer
-- This dual-write pattern is standard in Cassandra (no JOINs, model for your queries)
```

### Cassandra — Comments

```sql
CREATE TABLE comments_by_post (
    post_id       BIGINT,
    comment_id    BIGINT,            -- snowflake (embeds timestamp)
    user_id       BIGINT,
    text          TEXT,
    reply_to      BIGINT,            -- NULL for top-level comments
    like_count    INT,
    created_at    TIMESTAMP,
    PRIMARY KEY (post_id, created_at, comment_id)
) WITH CLUSTERING ORDER BY (created_at ASC, comment_id ASC);
```

### Social Graph — Adjacency Lists

```sql
-- "Who does user X follow?" (sharded by follower)
CREATE TABLE following (
    follower_id   BIGINT,
    followee_id   BIGINT,
    created_at    TIMESTAMP,
    PRIMARY KEY (follower_id, followee_id)
);

-- "Who follows user X?" (sharded by followee)
CREATE TABLE followers (
    followee_id   BIGINT,
    follower_id   BIGINT,
    created_at    TIMESTAMP,
    PRIMARY KEY (followee_id, follower_id)
);
```

### Feed Storage — Redis

```
Key:     feed:{user_id}
Type:    Sorted Set
Score:   ranking_score (combines recency, engagement, relevance)
Member:  post_id

ZREVRANGE feed:{user_id} 0 19  → top 20 posts for first page
Max size: 500-1000 entries per user (older content falls off, fetched from DB)
```

---

## 6. Media Upload & Processing Pipeline

### The Problem

Users upload raw photos (10MB DSLR JPEGs, HEIC from iPhones) and videos (4K, variable codecs). The platform must:

1. Accept the upload without blocking the user
2. Produce multiple resized variants (thumbnail, feed-size, full-res)
3. Transcode videos to standard formats/bitrates
4. Distribute globally via CDN
5. Do all of this for 100M+ new posts/day

### Upload Flow

```
  Client                     API Gateway              S3                  Media Pipeline
    │                            │                     │                       │
    │  POST /media/upload/init   │                     │                       │
    │ ─────────────────────────→ │                     │                       │
    │                            │ Generate pre-signed │                       │
    │                            │ S3 URL              │                       │
    │  { upload_url, media_id }  │                     │                       │
    │ ←───────────────────────── │                     │                       │
    │                            │                     │                       │
    │  PUT <upload_url>          │                     │                       │
    │ ────────────────────────────────────────────────→│                       │
    │  (direct to S3, bypasses   │                     │                       │
    │   API servers entirely)    │                     │                       │
    │                            │                     │                       │
    │  200 OK                    │                     │                       │
    │ ←────────────────────────────────────────────────│                       │
    │                            │                     │                       │
    │  POST /media/{id}/complete │                     │  S3 Event Notification│
    │ ─────────────────────────→ │                     │─────────────────────→ │
    │                            │ Publish to Kafka    │                       │
    │                            │ ──────────────────────────────────────────→ │
    │  { status: "processing" }  │                     │                       │
    │ ←───────────────────────── │                     │                       │
    │                            │                     │       ┌───────────┐   │
    │  (user can now POST /posts │                     │       │ Resize    │   │
    │   referencing media_id)    │                     │       │ 150x150   │   │
    │                            │                     │       │ 640x640   │   │
    │                            │                     │       │ 1080x1080 │   │
    │                            │                     │       │ original  │   │
    │                            │                     │       └─────┬─────┘   │
    │                            │                     │             │ Write   │
    │                            │                     │  ←──────────┘ variants│
    │                            │                     │             to S3     │
    │                            │                     │                       │
    │                            │     Update DB:      │                       │
    │                            │  ←────────────────────────────────────────  │
    │                            │   status=ready,     │                       │
    │                            │   variant URLs      │                       │
```

**Why pre-signed URLs?**

```
Without pre-signed URLs:
  Client → API Server → S3
  API server is a bottleneck: proxies every byte of a 10MB photo
  100M posts × 3 photos × 2MB = 600TB/day flowing through API servers
  Need thousands of API servers just for upload proxying

With pre-signed URLs:
  Client → S3 directly
  API server only generates a signed URL (microseconds, no data transfer)
  S3 handles the upload, is infinitely scalable
  API servers free for actual business logic
```

### Image Processing Pipeline

```
                     Raw Upload (S3 "incoming" bucket)
                              │
                              ▼
                    ┌──────────────────┐
                    │  Content Safety  │   ← ML model: NSFW detection,
                    │  Check           │     policy violation scan
                    └────────┬─────────┘
                             │ (pass)
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │ Thumbnail │  │  Feed    │  │   Full   │
        │ 150×150   │  │ 640×640  │  │ 1080×1080│
        │ quality=70│  │ quality= │  │ quality= │
        │           │  │ 80       │  │ 85       │
        └─────┬─────┘  └─────┬────┘  └─────┬────┘
              │              │              │
              ▼              ▼              ▼
        ┌──────────────────────────────────────┐
        │      S3 "processed" bucket           │
        │  /media/{media_id}/150.jpg           │
        │  /media/{media_id}/640.jpg           │
        │  /media/{media_id}/1080.jpg          │
        │  /media/{media_id}/original.jpg      │
        └──────────────────┬───────────────────┘
                           │
                           ▼
                    ┌──────────────┐
                    │     CDN      │
                    │  (edge cache │
                    │   + origin   │
                    │   shield)    │
                    └──────────────┘
```

### Video Transcoding (Reels)

```
Input: 4K H.265 MOV, 45 seconds, 80MB

Output variants:
  ┌──────────────┬────────────┬────────────┬──────────────┐
  │ Resolution   │ Bitrate    │ Codec      │ Use case     │
  ├──────────────┼────────────┼────────────┼──────────────┤
  │ 1080p        │ 5 Mbps     │ H.264      │ Wi-Fi / HD   │
  │ 720p         │ 2.5 Mbps   │ H.264      │ LTE          │
  │ 480p         │ 1 Mbps     │ H.264      │ 3G / slow    │
  │ 240p         │ 400 Kbps   │ H.264      │ Extreme slow │
  │ Audio only   │ 128 Kbps   │ AAC        │ Minimize data│
  └──────────────┴────────────┴────────────┴──────────────┘

Each variant is segmented into HLS chunks (2-4 second segments)
→ Adaptive bitrate streaming: client switches quality mid-playback
   based on network conditions.

Processing time: ~30 seconds for a 45s reel across a GPU farm
Processing at scale: 50M reels/day × 30s each = 1.5B CPU-seconds/day
  → Need ~17,000 vCPU-equivalents dedicated to transcoding
```

### Storage Tiering (Cost Optimization)

```
Hot (< 7 days):     S3 Standard         $0.023/GB/mo    ← new posts, active stories
Warm (7-90 days):   S3 Infrequent       $0.0125/GB/mo   ← older posts still accessed
Cold (> 90 days):   S3 Glacier Instant   $0.004/GB/mo    ← rarely accessed posts
Archive (> 1 year): S3 Glacier Deep      $0.00099/GB/mo  ← abandoned accounts

At 675 PB/year, tiering saves tens of millions per year:
  All S3 Standard:  675 PB × $0.023 × 12 ≈ $186M/year
  With tiering:     ~$40-60M/year (depends on access patterns)
```

---

## 7. Feed Generation System

This is the single hardest problem in the entire system. A user opens the app, and within 200ms, they must see a personalized, ranked stream of posts from the hundreds (or thousands) of accounts they follow.

### The Core Problem

```
User follows 500 accounts.
Each account posts ~1 per day.
Opening the feed means: "fetch the latest posts from 500 accounts,
rank them, and return the top 20."

Naive approach:
  SELECT * FROM posts WHERE user_id IN (select followee_id from following
    WHERE follower_id = ?) ORDER BY created_at DESC LIMIT 20;

  → Scans 500 user timelines, merges, sorts
  → At 5M QPS, this query runs 5 million times per second
  → Database dies instantly
```

### Fan-Out-on-Write (Push Model)

When a user creates a post, **push** it into every follower's precomputed feed.

```
Alice posts a photo
  │
  ▼
Post Service writes to posts table
  │
  ▼
Publishes event to Kafka: { type: "new_post", post_id: 123, user_id: alice }
  │
  ▼
Fan-Out Workers consume the event:
  1. Fetch Alice's follower list: [Bob, Carol, Dave, ... 10,000 followers]
  2. For each follower:
       ZADD feed:{follower_id} <score> <post_id>
       ZREMRANGEBYRANK feed:{follower_id} 1000 -1   (trim to max 1000)
  │
  ▼
Bob opens app → ZREVRANGE feed:bob 0 19 → instant response from Redis
```

**Advantages:**

- Feed read is O(1) — just a Redis ZREVRANGE
- Feeds are precomputed, latency is sub-millisecond
- Read-heavy workload becomes trivially fast

**Problems:**

```
Celebrity problem:
  Selena Gomez has 400M followers.
  She posts a photo.
  Fan-out worker must write to 400 million Redis keys.
  
  At 100K Redis writes/sec per worker: 400M / 100K = 4,000 seconds = 67 minutes
  Even with 1,000 parallel workers: 400M / (1000 × 100K) = 4 seconds
  
  But during that 4 seconds:
  - 4 seconds × 100K writes/sec × 1000 workers = 400M writes
  - Redis memory spike: 400M × ~100 bytes = 40 GB sudden memory pressure
  - Network bandwidth: 400M × 100 bytes = 40 GB in 4 seconds = 10 GB/sec
  
Wasted work:
  Many followers haven't opened the app in months.
  Writing to their feeds is pure waste.
  ~40% of DAU opens app daily → 60% of fan-out writes are never read.

Storage:
  500M DAU × 1000 post IDs per feed × 8 bytes per entry = 4 TB of Redis
  At $6/GB/month for Redis: 4 TB × $6 = $24,000/month just for feed storage
```

### Fan-Out-on-Read (Pull Model)

When a user opens the app, **pull** posts from every account they follow and merge in real-time.

```
Bob opens the app
  │
  ▼
Feed Service:
  1. Get Bob's following list: [Alice, Carol, Dave, ... 500 accounts]
  2. For each: fetch latest 5 posts from cache/DB
  3. Merge-sort all results
  4. Apply ranking model
  5. Return top 20
```

**Advantages:**

- No fan-out on write — posting is O(1)
- No wasted writes for inactive users
- No celebrity problem

**Problems:**

```
Latency:
  Bob follows 500 accounts.
  Fetching latest posts from each: even with batching, ~50-100ms
  Ranking: ~20ms
  Total: ~120ms best case
  
  But at p99 (one of the 500 accounts has a cache miss):
  → 500ms+ latency, violates SLA

Thundering herd:
  Celebrity posts → millions of users pull simultaneously
  → All hit the celebrity's post timeline concurrently
  → Same problem as cache stampede
```

### Hybrid Approach (What Instagram Actually Uses)

```
┌─────────────────────────────────────────────────────────┐
│                    Hybrid Fan-Out                       │
│                                                         │
│  "Regular" users (< 10K followers):  Fan-out-on-WRITE   │
│  "Celebrity" users (> 10K followers): Fan-out-on-READ   │
│                                                         │
│  The 0.1% of users who are celebrities produce ~30%     │
│  of all feed content but would cause 99% of fan-out     │
│  write volume. By excluding them from push, we          │
│  eliminate the explosion while keeping most feeds fast. │
└─────────────────────────────────────────────────────────┘
```

```
Bob opens the app:

Feed Service:
  1. ZREVRANGE feed:bob 0 49           ← pre-pushed posts from regular users
  2. Get list of celebrities Bob follows: [selenagomez, therock, ...]
  3. For each celebrity: fetch latest 3 posts from celebrity timeline cache
  4. Merge pre-pushed + pulled celebrity posts
  5. Apply ranking model
  6. Return top 20

Timeline:
  Step 1: ~1ms (Redis sorted set)
  Step 2: ~1ms (cached set)
  Step 3: ~5-10ms (parallel fetches, all cached)
  Step 4+5: ~10-20ms (ranking model inference)
  Total: ~15-30ms
```

### Feed Ranking

```
Raw candidate pool (500 posts from fan-out + celebrity pull)
  │
  ▼
┌──────────────────────────────────────┐
│        Feature Extraction            │
│                                      │
│  Post features:                      │
│  - Age (seconds since posted)        │
│  - Like count, comment count         │
│  - Media type (photo/video/carousel) │
│  - Has location, has hashtags        │
│  - Content embedding (ML)            │
│                                      │
│  User-Post affinity features:        │
│  - How often does user interact      │
│  │  with this author?                │
│  - Does user like this content type? │
│  - Time since user last saw author?  │
│                                      │
│  User features:                      │
│  - Session number today              │
│  - Time of day                       │
│  - Device type, connection speed     │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│         Ranking Model                │
│   (lightweight neural network)       │
│                                      │
│  Predicts P(engagement):             │
│   score = w1×P(like) + w2×P(comment) │
│         + w3×P(share) + w4×P(save)   │
│         - w5×P(hide)                 │
│                                      │
│  Must run in < 10ms for 500 posts    │
│  (batched inference on GPU)          │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│       Post-Ranking Rules             │
│                                      │
│  - Diversity: no more than 2         │
│    consecutive posts from same user  │
│  - Freshness boost: < 1hr gets 1.5x  │
│  - Type mixing: interleave photo/    │
│    video/carousel                    │
│  - Seen suppression: demote posts    │
│    user already scrolled past        │
└──────────────┬───────────────────────┘
               │
               ▼
        Top 20 posts → client
```

---

## 8. Social Graph Service

### Scale

```
2B users × average 200 following each = 400 billion edges
But distribution is extreme:
  - Median user follows ~150 accounts
  - Top 1% follows 2000+ accounts
  - Celebrities have 100M-400M followers
  - Average follower list: 150 entries (~1.2KB)
  - Max follower list: 400M entries (~3.2GB for one user)
```

### Storage: Dual Adjacency Lists

```
Why two tables (following + followers) instead of one edges table?

Single edges table:
  "Who does Alice follow?"  → WHERE follower_id = alice  ← efficient (partition key)
  "Who follows Alice?"      → WHERE followee_id = alice  ← requires full table scan
                                                           (followee_id is not partition key)

Two tables (denormalized):
  following(follower_id, followee_id) → partitioned by follower_id
  followers(followee_id, follower_id) → partitioned by followee_id

  Both queries hit only the partition key → O(1) lookup
  Cost: double the writes (write to both on follow/unfollow)
  Benefit: reads are 1000x more frequent than writes, so this is a huge net win
```

### Follow/Unfollow — The Consistency Problem

```
Alice follows Bob:

  1. Write to following table:  INSERT (alice, bob)
  2. Write to followers table:  INSERT (bob, alice)
  3. Increment Alice's following_count
  4. Increment Bob's follower_count
  5. Trigger fan-out: add Bob's recent posts to Alice's feed
  6. Send notification to Bob

Steps 1-4 must be atomic (or at least eventually consistent with no permanent drift).
Steps 5-6 are async and can tolerate delay.
```

```
Race condition: Alice follows Bob, then immediately unfollows Bob.

  Follow:   INSERT (alice, bob) into following + followers
  Unfollow: DELETE (alice, bob) from following + followers

  If these two operations interleave across shards:
  - following table says: Alice does NOT follow Bob (delete landed)
  - followers table says: Bob IS followed by Alice (insert landed, delete didn't)
  
  Result: Bob's follower count is wrong, Alice sees Bob's posts in feed despite unfollowing.

Solution: Use **Kafka with per-user partitioning.**
  All follow/unfollow events for Alice go to the same Kafka partition.
  A single consumer processes them in order → no interleaving.
  Both tables updated atomically from the same consumer.
```

### Graph Cache (TAO-style)

Meta/Facebook built TAO (The Associations and Objects cache), and Instagram uses a similar pattern:

```
                    ┌─────────────────────────────┐
                    │         Graph Cache          │
                    │       (TAO-like layer)       │
                    │                              │
                    │  Objects: user profiles      │
                    │  Associations: follow edges  │
                    │                              │
                    │  API:                        │
                    │   assoc_get(alice, FOLLOWS)  │
                    │   → [bob, carol, dave, ...]  │
                    │                              │
                    │ assoc_count(bob, FOLLOWED_BY)│
                    │   → 150,000                  │
                    │                              │
                    │ assoc_range(bob, FOLLOWED_BY,│
                    │     cursor, 50)              │
                    │   → [user_1, user_2, ...     │
                    │                              │
                    │ Implementation:              │
                    │  Leader cache (one per shard)│
                    │  Follower caches (many, LRU) │
                    │  Write-through to leader     │
                    │  Read from nearest follower  │
                    └─────────────────────────────┘

Benefits:
  - Read QPS for graph: ~10M/sec (all cache hits)
  - Write QPS for graph mutations: ~1K/sec (write-through to DB)
  - Cache hit rate: ~99.9% (graph data is extremely hot)
  - Handles the celebrity follower list problem: cached once, read millions of times
```

### "Is Following" Check — The Hot Path

```
Every time Bob looks at Alice's profile, the client needs to know:
"Does Bob follow Alice?"  → to show the Follow/Following button

This is called on every profile view, every post render (mutual follows badge), etc.
Volume: billions of times per day.

Naive: SELECT 1 FROM following WHERE follower_id = bob AND followee_id = alice
  → Hits DB every time → dies at scale

Solution: **Bloom filter per user.**

  When Bob's session starts, load a Bloom filter of all user_ids Bob follows.
  ~200 follows × 10 bits per element = ~250 bytes per user.
  Fits in a cookie or app memory.

  isFollowing(bob, alice):
    1. Check Bloom filter (client-side or edge cache): if NO → definitely not following
    2. If MAYBE → verify against graph cache (rare, <1% false positive rate)

  Result: 99% of "is following" checks never hit any backend service.
```

---

## 9. Stories & Reels Subsystem

### Stories — Ephemeral Content

```
Properties:
  - Visible for exactly 24 hours after posting
  - Viewers list tracked per story
  - Displayed as a "tray" at the top of feed (ordered by recency + affinity)
  - "Seen" state per viewer must be tracked

Scale:
  500M stories/day, 24-hour TTL
  At any moment: ~500M active stories in the system
  Story views: ~5B/day
  "Seen" writes: ~5B/day
```

### Storage Architecture

```
Stories are HOT for 24 hours, then completely worthless.
This is the perfect Redis TTL use case.

Redis (active stories):
  Key:    story:{story_id}
  Type:   Hash
  Fields: user_id, media_url, created_at, view_count
  TTL:    86400 (24 hours)
  → Automatically evicted after 24h. Zero cleanup needed.

Story Trays (which stories to show a user):
  Key:    story_tray:{user_id}
  Type:   Sorted Set
  Score:  ranking_score (affinity + recency)
  Member: storyPoster_user_id
  TTL:    3600 (rebuild every hour or on-demand)

Story Viewers:
  Key:    story_viewers:{story_id}
  Type:   Sorted Set
  Score:  view_timestamp
  Member: viewer_user_id
  TTL:    86400

Memory estimate:
  500M active stories × 500 bytes each = 250 GB in Redis
  Viewer sets: 500M stories × avg 50 viewers × 16 bytes = 400 GB
  Total: ~650 GB across Redis cluster

  After 24h: memory freed automatically. No vacuum, no archival jobs.
```

### Stories Tray Generation

```
Bob opens the app → needs to see story trays at the top

Naive: Check every account Bob follows for active stories
  Bob follows 500 → 500 Redis lookups → 10-20ms

Optimized: Pre-built tray via fan-out

When Alice posts a story:
  1. Publish event: { type: "new_story", user_id: alice, story_id: 456 }
  2. Fan-out worker: for each of Alice's followers:
       ZADD story_tray:{follower_id} <score> alice
       (score = recency × affinity_weight)
  3. When Bob opens app:
       ZREVRANGE story_tray:bob 0 19
       For each user in tray: fetch their active stories

This is the same hybrid fan-out as the feed — celebrities are pulled, not pushed.
```

### Reels

```
Reels are different from stories:
  - Not ephemeral (permanent unless deleted)
  - Discovery-focused (Explore/Reels tab shows content from non-followed accounts)
  - Heavy video processing (HLS transcoding, multiple bitrates)
  - Recommendation-driven (not follow-graph-driven)

The Reels feed is more like TikTok's "For You" page:
  - Candidate generation from a content pool (billions of reels)
  - ML ranking model scores each reel for the user
  - Diversity rules applied
  - No fan-out involved — purely pull-based with heavy caching

Storage: Same as posts (PostgreSQL for metadata, S3 for media)
  but reels have additional tables for engagement signals used by the
  recommendation model (watch duration, replays, shares, etc.)
```

---

## 10. Search & Explore

### Search Architecture

```
                    ┌────────────────┐
                    │  Search Query  │  GET /v1/search?q=travel&type=hashtag
                    └───────┬────────┘
                            │
                    ┌───────▼────────┐
                    │ Search Router  │  Routes to appropriate index
                    └───────┬────────┘
                            │
           ┌────────────────┼────────────────┐
           ▼                ▼                ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │ User Index   │ │ Hashtag Index│ │Location Index│
    │ (ES cluster) │ │ (ES cluster) │ │ (ES cluster) │
    │              │ │              │ │              │
    │ Fields:      │ │ Fields:      │ │ Fields:      │
    │ - username   │ │ - tag name   │ │ - place name │
    │ - full_name  │ │ - post_count │ │ - lat/lng    │
    │ - bio        │ │ - trending   │ │ - post_count │
    │ - follower_  │ │   score      │ │ - city       │
    │   count      │ │              │ │ - country    │
    │ - is_verified│ │              │ │              │
    └──────────────┘ └──────────────┘ └──────────────┘

Indexing pipeline:
  PostgreSQL → Debezium CDC → Kafka → ES Index Workers → Elasticsearch

  Index lag: < 5 seconds (new user/post searchable within 5s of creation)
```

### Explore Page

```
The Explore page is Instagram's main content discovery surface.
It shows posts/reels from accounts the user does NOT follow.

Candidate Generation (offline, every few hours):
  1. Collaborative filtering: "Users similar to you liked these posts"
  2. Content-based: "Posts with topics/hashtags you've engaged with"
  3. Trending: High engagement velocity in last N hours
  4. Geographic: Popular near user's location

Real-time Ranking (per request, < 50ms):
  1. Fetch ~1000 candidates from pre-computed pools
  2. Score with lightweight ranking model
  3. Apply diversity rules (no duplicate authors, mix media types)
  4. Return top 30

The Explore page is entirely ML-driven — no fan-out, no follow graph.
Scalability is bounded by the ML inference cost, not graph traversal.
```

---

## 11. Notifications Pipeline

### Scale

```
Events that generate notifications:
  - Likes:    5B/day
  - Comments: 500M/day
  - Follows:  50M/day
  - Mentions: 100M/day
  - DMs:      500M/day
  - Story replies: 200M/day

  Total: ~6.35 billion notification events/day
  Peak: ~120K events/sec

Push notifications (APNs/FCM):
  Not all events become push notifs (user settings, dedup, batching)
  After filtering: ~2B push notifications/day
```

### Architecture

```
  Event Source (Kafka)
       │
       ▼
  ┌──────────────────────┐
  │ Notification Router  │  Consumes all events from Kafka
  │                      │
  │ Responsibilities:    │
  │ 1. Deduplication     │  ← Did we already notify for this?
  │ 2. User preferences  │  ← Does user want push for likes?
  │ 3. Rate limiting     │  ← Max 5 push notifs per minute
  │ 4. Batching          │  ← "alice and 4 others liked your post"
  │ 5. Priority          │  ← DM > comment > like
  └──────────┬───────────┘
             │
    ┌────────┼────────┐
    ▼        ▼        ▼
  ┌──────┐ ┌──────┐ ┌────────┐
  │ Push │ │In-App│ │ Email  │
  │APNs/ │ │Notif │ │ (batch │
  │ FCM  │ │Store │ │  daily)│
  └──────┘ └──┬───┘ └────────┘
              │
              ▼
         Cassandra
    (notification log per user,
     TTL = 90 days,
     partitioned by user_id)
```

### Notification Batching (The "and 47 others" Problem)

```
Problem:
  Celebrity posts a photo. 1 million likes in 5 minutes.
  Sending 1 million push notifications saying "X liked your photo"
  is useless and annoying.

Solution: Time-window batching

  1. First like arrives → start a 30-second window
  2. During window, accumulate likes in a Redis counter:
       INCR notif_batch:post:123:like    (returns count)
       EXPIRE notif_batch:post:123:like 30
  3. First like triggers a delayed notification (30s delay)
  4. After 30 seconds, notification fires:
     - Count = 1: "alice liked your photo"
     - Count = 5: "alice, bob, and 3 others liked your photo"
     - Count = 10000: "10,000 people liked your photo"
  5. Next batch starts fresh

This reduces 1 million push notifications to ~1-5 batched notifications.
```

---

## 12. Direct Messaging

### Requirements

```
- 1:1 and group messaging (up to 250 members)
- Text, photos, videos, voice messages, posts (sharing)
- Read receipts, typing indicators
- End-to-end encryption (optional, being rolled out)
- Message ordering guaranteed within a conversation
- Offline message delivery (when recipient comes online)
```

### Architecture

```
     Alice (sender)                                    Bob (recipient)
         │                                                │
    ┌────▼─────┐                                    ┌─────▼────┐
    │WebSocket │                                    │WebSocket │
    │Connection│                                    │Connection│
    └────┬─────┘                                    └─────▲────┘
         │                                                │
    ┌────▼──────────────────────────────────────────┐     │
    │              Message Router                    │     │
    │                                                │     │
    │  1. Validate & authenticate                    │     │
    │  2. Write to Cassandra (durable)               │     │
    │  3. Check if Bob is online:                    │     │
    │     YES → route to Bob's WS connection ────────┼─────┘
    │     NO  → queue for offline delivery           │
    │  4. Update conversation metadata               │
    │  5. Send push notification (if offline)        │
    └────────────────────────────────────────────────┘

Connection Registry (Redis):
  Key:    ws:user:{user_id}
  Value:  { server_id: "ws-server-42", connected_at: timestamp }
  TTL:    Refreshed on every heartbeat

  "Where is Bob connected?"
  → HGET ws:user:bob → ws-server-42
  → Route message to ws-server-42 via internal message bus
```

### Message Storage

```sql
-- Cassandra: Messages by conversation
-- Partitioned by conversation_id, clustered by message_id (time-sorted)
CREATE TABLE messages_by_conversation (
    conversation_id  BIGINT,
    message_id       BIGINT,      -- snowflake: embeds timestamp
    sender_id        BIGINT,
    message_type     TINYINT,     -- 1=text, 2=image, 3=video, 4=shared_post
    content          TEXT,
    media_url        TEXT,
    created_at       TIMESTAMP,
    PRIMARY KEY (conversation_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);

-- Conversation inbox per user (ordered by last activity)
CREATE TABLE inbox_by_user (
    user_id           BIGINT,
    conversation_id   BIGINT,
    last_message_at   TIMESTAMP,
    last_message_preview TEXT,
    unread_count      INT,
    PRIMARY KEY (user_id, last_message_at, conversation_id)
) WITH CLUSTERING ORDER BY (last_message_at DESC, conversation_id DESC);
```

### Message Ordering Guarantee

```
Problem: Two messages sent in quick succession can arrive out of order.

  Alice sends: "I'm breaking up with you" then "...my phone bill"
  If reordered: "...my phone bill" then "I'm breaking up with you"

Solution: Server-assigned monotonic message IDs per conversation.

  1. Each conversation has a per-conversation sequence counter in Redis:
       INCR conv_seq:{conversation_id}

  2. Message is assigned this sequence number at the server, not the client.
     Client timestamp is stored but NOT used for ordering.

  3. Cassandra clustering key is the server-assigned message_id (snowflake with
     the sequence embedded), guaranteeing total order within a conversation.

  4. Client displays messages sorted by message_id, not by local clock.
```

---

## 13. Database & Storage Architecture

### Sharding Strategy

```
┌──────────────────────────────────────────────────────┐
│                 Sharding Overview                    │
├──────────────────┬───────────────┬───────────────────┤
│ Data             │ Shard Key     │ Why               │
├──────────────────┼───────────────┼───────────────────┤
│ Users/Profiles   │ user_id       │ All user data     │
│                  │               │ co-located        │
├──────────────────┼───────────────┼───────────────────┤
│ Posts            │ post_id       │ Distributes evenly│
│                  │               │ (snowflake IDs)   │
├──────────────────┼───────────────┼───────────────────┤
│ Posts by User    │ user_id       │ Profile grid needs│
│ (secondary index)│               │ all posts by user │
├──────────────────┼───────────────┼───────────────────┤
│ Following        │ follower_id   │ "Who do I follow?"│
│                  │               │ is the hot query  │
├──────────────────┼───────────────┼───────────────────┤
│ Followers        │ followee_id   │ "Who follows me?" │
│                  │               │ is the hot query  │
├──────────────────┼───────────────┼───────────────────┤
│ Likes            │ post_id       │ "Who liked this    │
│                  │               │ post?" co-located  │
├──────────────────┼───────────────┼───────────────────┤
│ Messages         │conversation_id│ All messages in a  │
│                  │               │ thread co-located  │
├──────────────────┼───────────────┼───────────────────┤
│ Notifications    │ user_id       │ User's notif inbox │
│                  │               │ is a single read   │
└──────────────────┴───────────────┴───────────────────┘
```

### ID Generation — Snowflake IDs

```
Instagram cannot use auto-increment IDs at scale:
  - Multiple shards → ID collisions
  - Sequential IDs leak information (post count, user count)
  - Auto-increment is a single point of contention

Snowflake ID (64-bit):
  ┌───────────────────────────────────────────────────────┐
  │ 1 bit │    41 bits      │  10 bits   │   12 bits      │
  │unused │ timestamp (ms)  │ machine ID │ sequence number│
  └───────────────────────────────────────────────────────┘

  41 bits of timestamp: ~69 years from epoch
  10 bits of machine ID: 1024 unique generators
  12 bits of sequence: 4096 IDs per millisecond per machine

  Total capacity: 1024 machines × 4096 IDs/ms = 4 million IDs/ms

  Instagram's variant: (timestamp_ms - custom_epoch) << 23 | shard_id << 10 | sequence
  This embeds the shard ID in the ID itself, making routing trivial:
    shard = (post_id >> 10) & 0x1FFF
```

### Database Topology (Per Service)

```
                    ┌──────────────────────────────────┐
                    │          Shard 1                 │
                    │                                  │
                    │   Primary (Region A)             │
                    │      │                           │
                    │      ├── Sync Replica (Region A) │  ← for HA failover
                    │      ├── Async Replica (Region B)│  ← cross-region DR
                    │      └── Read Replica (Region A) │  ← read traffic
                    │                                  │
                    │   Shard range: user_id % 256 = 0 │
                    └──────────────────────────────────┘
                    ┌──────────────────────────────────┐
                    │          Shard 2                 │
                    │   ...same topology...            │
                    │   Shard range: user_id % 256 = 1 │
                    └──────────────────────────────────┘
                    ...
                    ┌──────────────────────────────────┐
                    │          Shard 256               │
                    │   ...                            │
                    │  Shard range: user_id % 256 = 255│
                    └──────────────────────────────────┘

Total: 256 shards × 4 replicas each = 1,024 PostgreSQL instances per service
(Instagram reportedly runs thousands of PostgreSQL shards)
```

---

## 14. Caching Architecture

### Multi-Layer Cache

```
Layer 0: Client Cache (App Memory + Disk)
  │  Feed data, user profiles, media cached on device
  │  TTL: varies (profile = 5min, media = indefinite)
  │  Hit rate: ~60-70% of all data requests never leave the device
  │
  ▼
Layer 1: CDN Edge Cache (Akamai / Meta PoPs)
  │  Static media (images, video segments, JS/CSS)
  │  Hit rate: ~95% for media
  │  100+ edge locations worldwide
  │
  ▼
Layer 2: Application Cache (Memcached / Redis)
  │
  │  ┌─── Redis Clusters ────────────────────────────────────────────┐
  │  │                                                               │
  │  │  Feed Cache          Graph Cache         Session Cache        │
  │  │  - feed:{user_id}    - following:{uid}   - session:{token}    │
  │  │  - Sorted Sets       - Sets              - Hash               │
  │  │  - 4 TB              - 2 TB              - 500 GB             │
  │  │                                                               │
  │  │  Story Cache          Counter Cache       Rate Limit          │
  │  │  - story:{id}        - likes:{post_id}   - rl:{user_id}       │
  │  │  - Hash w/ TTL       - Strings            - Sliding window    │
  │  │  - 650 GB            - 1 TB               - 200 GB            │
  │  │                                                               │
  │  └───────────────────────────────────────────────────────────────┘
  │  Total: ~8-10 TB of Redis/Memcached
  │  Hit rate: ~99% for hot paths
  │
  ▼
Layer 3: Database (PostgreSQL, Cassandra)
  Only ~1% of read requests reach the database in steady state.
```

### Cache Invalidation Patterns

```
Pattern 1: Write-Through (strong consistency)
  Used for: user profiles, follow graph
  
  Update user bio:
    1. Write to DB
    2. Write to cache (same transaction boundary)
    3. If cache write fails → invalidate cache key (next read repopulates)

Pattern 2: Write-Behind / Eventual (performance over consistency)
  Used for: like counts, comment counts
  
  User likes a post:
    1. Increment Redis counter: INCR likes:{post_id}    (instant)
    2. Publish to Kafka: { post_id, like_count }
    3. Kafka consumer batch-updates DB every 5 seconds
    
  The counter in Redis is always "ahead" of the DB by up to 5 seconds.
  This is acceptable: like counts don't need to be exact in real-time.

Pattern 3: Cache-Aside (lazy population)
  Used for: post data, media metadata
  
  Read post:
    1. Check cache: GET post:{post_id}
    2. Cache miss → read from DB → write to cache with TTL
    3. Cache hit → return directly

Pattern 4: TTL-Based Expiry (self-healing)
  Used for: everything as a safety net
  
  Every cache key has a TTL:
    - Feed: 5 minutes
    - Profile: 10 minutes
    - Counters: 1 hour
    - Media URLs: 24 hours
  
  Even if invalidation fails, stale data expires automatically.
  TTL is the "eventually" in "eventually consistent."
```

### Thundering Herd Protection

```
Celebrity posts → millions of users open the app → cache key expires
→ all simultaneously query the DB for the same data

             1,000,000 concurrent requests
                       │
                       ▼
                 Cache MISS
                       │
          ┌────────────┼────────────────┐
          │            │                │
          ▼            ▼                ▼
     Thread 1     Thread 2  ...   Thread 1,000,000
     SELECT *     SELECT *        SELECT *
     FROM posts   FROM posts      FROM posts
     WHERE id=X   WHERE id=X      WHERE id=X
          │            │                │
          └────────────┼────────────────┘
                       │
                  Database dies

Solution: Single-flight / request coalescing

     1,000,000 concurrent requests
                       │
                       ▼
                 Cache MISS
                       │
            ┌──────────▼──────────┐
            │   Singleflight      │
            │                     │
            │ Only ONE request    │
            │ proceeds to DB.     │
            │ Remaining 999,999   │
            │ wait on a channel.  │
            └──────────┬──────────┘
                       │
                  1 DB query
                       │
                  Write to cache
                       │
            ┌──────────▼──────────┐
            │ Wake all waiters    │
            │ Return cached result│
            └─────────────────────┘
```

---

## 15. Scalability Deep Dive

### Horizontal Scaling by Layer

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Scaling Each Layer                               │
├─────────────────┬──────────────┬────────────────────────────────────┤
│ Layer           │ Strategy     │ Details                            │
├─────────────────┼──────────────┼────────────────────────────────────┤
│ CDN             │ Edge PoPs    │ 200+ global edge locations.        │
│                 │              │ Absorbs 95% of media traffic.      │
│                 │              │ Auto-scales. Infinite for reads.   │
├─────────────────┼──────────────┼────────────────────────────────────┤
│ API Gateway     │ Auto-scale   │ Stateless pods behind L7 LB.       │
│                 │              │ Scale by CPU/request count.        │
│                 │              │ 10K+ pods at peak.                 │
├─────────────────┼──────────────┼────────────────────────────────────┤
│ Application     │ Microservice │ Each service scales independently. │
│ Services        │ + auto-scale │ Feed service: CPU-bound (ranking)  │
│                 │              │ Post service: I/O-bound (DB)       │
│                 │              │ Media: GPU-bound (transcoding)     │
├─────────────────┼──────────────┼────────────────────────────────────┤
│ Cache (Redis)   │ Cluster +    │ Redis Cluster: add shards for      │
│                 │ sharding     │ capacity. Read replicas for QPS.   │
│                 │              │ Separate clusters per use case.    │
├─────────────────┼──────────────┼────────────────────────────────────┤
│ Database        │ Sharding +   │ PostgreSQL: hash-based sharding.   │
│                 │ read replicas│ Read replicas per shard.           │
│                 │              │ Cassandra: add nodes for linear    │
│                 │              │ scale (no resharding needed).      │
├─────────────────┼──────────────┼────────────────────────────────────┤
│ Event Bus       │ Partitions   │ Kafka: add partitions per topic.   │
│ (Kafka)         │              │ Each partition is an independent   │
│                 │              │ ordered log. 100K+ partitions.     │
├─────────────────┼──────────────┼────────────────────────────────────┤
│ Object Storage  │ Managed      │ S3: infinite scale, 5500 PUT/sec   │
│ (S3)            │              │ per prefix. Use random prefixes.   │
└─────────────────┴──────────────┴────────────────────────────────────┘
```

### Database Shard Rebalancing

```
Problem: Over time, some shards get hotter (more active users landed on them).
  Shard 42 has 2x the write throughput of shard 100.

Approach 1: Virtual shards (consistent hashing)
  Instead of 256 physical shards, use 16,384 virtual shards.
  Map virtual → physical: vshard % 256 = physical shard.
  To rebalance: move virtual shard ranges to different physical hosts.
  Data migration is smaller (1/16384th of data per virtual shard).

Approach 2: Shard splitting
  Shard 42 is too hot → split into shard 42a and 42b.
  Use range-based routing: user_id hash < midpoint → 42a, else → 42b.
  Requires online data migration (double-write during transition).

Instagram's approach: Started with hash-based sharding on PostgreSQL.
  Each logical shard is a PostgreSQL schema.
  Multiple schemas per physical server.
  To rebalance: move schemas between servers (pg_dump + streaming replication).
```

### Rate Limiting

```
Multi-tier rate limiting:

Tier 1: API Gateway (global)
  - 100 requests/minute per user (general)
  - 5 posts/hour per user
  - 200 likes/hour per user
  - Implementation: Sliding window counter in Redis
  
    EVAL "
      local key = KEYS[1]
      local window = tonumber(ARGV[1])
      local limit = tonumber(ARGV[2])
      local now = tonumber(ARGV[3])
      
      redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
      local count = redis.call('ZCARD', key)
      
      if count < limit then
        redis.call('ZADD', key, now, now .. math.random())
        redis.call('EXPIRE', key, window)
        return 0  -- allowed
      end
      return 1    -- rate limited
    " 1 rl:{user_id}:likes 3600 200 <now_ms>

Tier 2: Service-level (adaptive)
  - Circuit breaker per downstream service
  - If DB is slow (p99 > 500ms): shed 50% of non-critical reads
  - If Kafka consumer lag > 1M: pause non-critical producers

Tier 3: Client-side (cooperative)
  - Exponential backoff on 429 responses
  - Client SDK enforces minimum interval between requests
```

### Hot Partition Mitigation

```
Problem: Celebrity user_id lands on one shard.
  All 400M follower lookups hit the same shard.

Solutions:

1. Read replicas per shard (straightforward):
   Hot shard gets 5 read replicas instead of the default 2.
   Reads distributed across replicas.
   Limitation: only helps reads, not writes.

2. Scatter-gather for follower lists:
   Instead of one partition key, split the follower list:
   
   followers:selenagomez:0  (first 100K followers)
   followers:selenagomez:1  (next 100K followers)
   ...
   followers:selenagomez:3999 (last 100K followers)
   
   "Get all followers" = scatter across 4000 partitions + gather.
   Each partition lives on a different shard → load distributed.

3. Caching layer absorbs the hotspot:
   Celebrity data cached at every level.
   Cache TTL: 60 seconds for followers, 5 seconds for post data.
   99.99% of requests never reach the shard.
   The shard only handles cache misses → manageable load.
```

---

## 16. Reliability & Fault Tolerance Deep Dive

### Cell-Based Architecture

```
Instagram (Meta) uses a cell-based architecture to contain blast radius.

┌─────────────────────────────────────────────────────────────┐
│                    Global Layer                             │
│  DNS, CDN, Global Load Balancer, Configuration Service      │
└────────────┬────────────────────────────┬───────────────────┘
             │                            │
    ┌────────▼────────┐          ┌────────▼────────┐
    │     Cell A      │          │     Cell B      │
    │   (US-East)     │          │   (US-West)     │
    │                 │          │                 │
    │ ┌─────────────┐ │          │ ┌─────────────┐ │
    │ │All services │ │          │ │All services │ │
    │ │(post, feed, │ │          │ │(post, feed, │ │
    │ │ graph, DM,  │ │          │ │ graph, DM,  │ │
    │ │ notif, ...) │ │          │ │ notif, ...) │ │
    │ └─────────────┘ │          │ └─────────────┘ │
    │ ┌─────────────┐ │          │ ┌─────────────┐ │
    │ │ DB shards   │ │          │ │ DB shards   │ │
    │ │ Redis       │ │          │ │ Redis       │ │
    │ │ Kafka       │ │          │ │ Kafka       │ │
    │ │ S3          │ │          │ │ S3          │ │
    │ └─────────────┘ │          │ └─────────────┘ │
    │                 │          │                 │
    │ Users: 0-49%    │          │ Users: 50-100%  │
    │ (by user_id     │          │ (by user_id     │
    │  hash range)    │          │  hash range)    │
    └─────────────────┘          └─────────────────┘

Key property: Each cell is a fully independent stack.
  - User 12345 is assigned to Cell A. ALL their requests go to Cell A.
  - Cell A failure → only 50% of users affected.
  - Cell B continues serving the other 50% with zero impact.
  - No cross-cell dependencies during normal operation.

Cross-cell only for:
  - Follow graph mutations (Alice in Cell A follows Bob in Cell B)
  - DMs between users in different cells
  - These use async replication via Kafka cross-cell bridge.
```

### Failure Scenarios and Mitigations

```
┌──────────────────────────┬──────────────────────────────────────────────┐
│ Failure                  │ Mitigation                                   │
├──────────────────────────┼──────────────────────────────────────────────┤
│ Single API pod crash     │ Load balancer health check removes it in 5s. │
│                          │ Other pods absorb traffic. No user impact.   │
├──────────────────────────┼──────────────────────────────────────────────┤
│ Database primary failure │ Sync replica promoted in 10-30 seconds.      │
│ (one shard)              │ Writes to that shard fail during failover.   │
│                          │ Reads served from read replicas (unaffected).│
│                          │ Retry with exponential backoff.              │
├──────────────────────────┼──────────────────────────────────────────────┤
│ Redis cluster node down  │ Redis Cluster auto-failover (< 5s).          │
│                          │ During gap: cache miss → DB serves reads.    │
│                          │ Feed service: return slightly stale feed     │
│                          │ from local cache rather than error.          │
├──────────────────────────┼──────────────────────────────────────────────┤
│ Kafka broker failure     │ Partition leadership moves to follower ISR.  │
│                          │ Producers retry with idempotent writes.      │
│                          │ Consumer rebalance picks up from committed   │
│                          │ offset. No message loss.                     │
├──────────────────────────┼──────────────────────────────────────────────┤
│ CDN edge failure         │ DNS automatically routes to next-closest PoP.│
│                          │ Origin shield absorbs cache-miss spike.      │
├──────────────────────────┼──────────────────────────────────────────────┤
│ S3 region outage         │ Cross-region replication to backup region.   │
│ (extremely rare)         │ DNS switch to backup region's S3 endpoint.   │
│                          │ New uploads queued, served from CDN cache.   │
├──────────────────────────┼──────────────────────────────────────────────┤
│ Entire Cell failure      │ Global LB drains traffic from dead cell.     │
│ (region-level)           │ Surviving cell takes 100% of traffic.        │
│                          │ Pre-provisioned with 2x headroom.            │
│                          │ Users in dead cell see errors for 30-60s     │
│                          │ during DNS/routing switchover.               │
├──────────────────────────┼──────────────────────────────────────────────┤
│ Slow dependency          │ Circuit breaker opens after 5 consecutive    │
│ (DB slow, not down)      │ timeouts. Returns degraded response:         │
│                          │ - Feed: show cached feed (may be stale)      │
│                          │ - Like: queue locally, sync later            │
│                          │ - Profile: show cached version               │
│                          │ Half-open probe every 10s to detect recovery.│
└──────────────────────────┴──────────────────────────────────────────────┘
```

### Circuit Breaker Implementation

```
States:
  CLOSED   → all requests pass through (normal operation)
  OPEN     → all requests immediately fail-fast (dependency is down)
  HALF-OPEN → one probe request allowed (testing if dependency recovered)

                    success
            ┌───────────────────┐
            │                   │
            ▼                   │
         CLOSED ──────────────────→ OPEN
            │    5 failures in      │
            │    10 seconds         │ timeout (30s)
            │                       ▼
            └──────────────── HALF-OPEN
                 probe succeeds     │
                                    │ probe fails
                                    └──→ OPEN

Degraded responses when circuit is OPEN:

  Feed Service (circuit to ranking model OPEN):
    → Return posts in reverse chronological order (no ranking)
    → User sees slightly less relevant feed, but still sees content

  Like Service (circuit to Cassandra OPEN):
    → Accept like, write to local buffer
    → Return success to user (optimistic)
    → Flush buffer when circuit closes
    → Risk: buffer lost if pod crashes → like lost (acceptable: user can re-like)

  Search Service (circuit to Elasticsearch OPEN):
    → Return cached trending searches
    → Show "search temporarily unavailable" for specific queries
```

### Data Durability Guarantees

```
Media (photos/videos):
  - S3: 99.999999999% (11 nines) durability
  - Cross-region replication: survives entire region loss
  - Once uploaded, media is NEVER lost

Post metadata:
  - PostgreSQL with synchronous replication to at least 1 replica
  - WAL shipped to S3 every 5 minutes for point-in-time recovery
  - RPO (recovery point objective): < 5 minutes

Messages (DMs):
  - Cassandra with replication factor 3
  - Write acknowledged after 2 replicas confirm (QUORUM)
  - Anti-entropy repair runs weekly to fix any replica drift

Feed cache (Redis):
  - Not durable by design — it's a cache
  - Loss = feeds rebuilt from DB (slower but no data loss)
  - Redis persistence (RDB snapshots) only for warm restart, not durability
```

### Idempotency Everywhere

```
Problem: Network failure between client and server.
  Client sends "like post 123" → timeout → retries → double like?

Solution: Idempotency keys at every mutation endpoint.

  Client generates a unique request ID per action:

  POST /v1/posts/123/like
  Headers:
    X-Request-ID: 550e8400-e29b-41d4-a716-446655440000
    X-Idempotency-Key: like:user42:post123

  Server:
    1. Check Redis: EXISTS idempotency:{key}
    2. If exists → return cached response (no side effects)
    3. If not → execute, then SET idempotency:{key} {response} EX 3600

  Cost: One extra Redis read per mutation (~0.1ms)
  Benefit: No duplicate likes, follows, comments, orders, etc.
```

---

## 17. Observability & Operational Excellence

### Metrics (The Four Golden Signals per Service)

```
┌────────────┬──────────────────────────────────────────────────┐
│ Signal     │ What to measure                                  │
├────────────┼──────────────────────────────────────────────────┤
│ Latency    │ p50, p95, p99 of every API endpoint              │
│            │ Feed load latency (end-to-end)                   │
│            │ DB query latency per shard                       │
│            │ Cache hit/miss latency                           │
├────────────┼──────────────────────────────────────────────────┤
│ Traffic    │ Requests/sec per service                         │
│            │ Feed loads/sec (the #1 business metric)          │
│            │ Uploads/sec, likes/sec                           │
├────────────┼──────────────────────────────────────────────────┤
│ Errors     │ 5xx rate per endpoint                            │
│            │ Kafka consumer errors                            │
│            │ DB connection pool exhaustion events             │
│            │ Cache eviction rate                              │
├────────────┼──────────────────────────────────────────────────┤
│ Saturation │ CPU, memory, disk per host                       │
│            │ DB connection pool usage (%)                     │
│            │ Kafka consumer lag (messages behind)             │
│            │ Redis memory usage vs max                        │
└────────────┴──────────────────────────────────────────────────┘
```

### Distributed Tracing

```
Single feed load trace:

  [API Gateway]  12ms
    └─ [Feed Service]  45ms
         ├─ [Redis: feed cache]  1ms  (HIT)
         ├─ [Redis: celebrity timelines]  3ms  (3 parallel fetches)
         ├─ [Ranking Service]  18ms
         │    └─ [Feature Store]  5ms
         ├─ [User Service: batch profile fetch]  8ms
         │    └─ [Redis: user cache]  1ms  (HIT for 18/20, MISS for 2)
         │         └─ [PostgreSQL]  4ms  (2 cache misses)
         └─ [Media Service: URL signing]  3ms

  Total: 45ms  (well within 200ms SLA)

  Trace ID propagated via X-Trace-ID header through all services.
  Stored in Jaeger/Tempo with 1% sampling (100% for errors).
```

---

## 18. Corner Cases & Hard Problems

### 1. Celebrity Account Posts During Peak

```
Problem: Selena Gomez posts at 8 PM EST (peak US traffic).
  - Fan-out skipped (she's a celebrity, pulled on read)
  - But 50M users open app in next 5 minutes and all pull her latest post
  - Her post timeline cache key receives 50M reads in 5 minutes = 167K reads/sec

Solution:
  - Post data cached at CDN edge (cache post metadata as JSON, 30s TTL)
  - Redis read replica set specifically for celebrity timelines
  - Post data pushed to edge caches proactively on publish (cache warming)
  - Result: 99.9% of reads served from cache, < 200 reach origin per second
```

### 2. Viral Post Causing Like Counter Overflow

```
Problem: Post gets 10M likes in 1 hour.
  That's 2,778 like writes/sec to a single post_id partition.

If using single Redis counter:
  INCR likes:post:123  → no problem, Redis handles 100K+/sec on one key

If using Cassandra counter:
  Cassandra counters have known issues with high-contention hot keys.
  Solution: Sharded counters.
  
  Instead of one counter, use 100 sub-counters:
    likes:post:123:shard:0  ... likes:post:123:shard:99
  
  Each INCR goes to a random shard: shard = rand() % 100
  Total count = SUM of all 100 shards
  
  2,778 writes/sec ÷ 100 shards = 28 writes/sec per shard (trivial)
  
  Read count: read all 100 shards + sum (cached for 5 seconds)
```

### 3. Follow/Unfollow Storm (Bot or Mass Action)

```
Problem: User follows and unfollows 1000 accounts rapidly.
  Each follow triggers fan-out (add posts to feed).
  Each unfollow triggers fan-out removal.
  Rapid toggle = massive wasted work.

Solution:
  - Debounce follow/unfollow at the event bus level
  - If follow(alice, bob) and unfollow(alice, bob) arrive within 30 seconds,
    they cancel each other out → no fan-out at all
  - Rate limit: max 60 follow/unfollow actions per hour per user
  - Accounts exceeding this are flagged for bot review
```

### 4. Stale Feed After Unfollowing

```
Problem: Alice unfollows Bob. Bob's posts are still in Alice's precomputed feed.
  Alice opens the app and sees Bob's post → "I just unfollowed them!"

Solution:
  - On unfollow: synchronously remove Bob's posts from Alice's feed cache
    ZRANGEBYSCORE feed:alice -inf +inf → filter by user_id = bob → ZREM
  - This is O(n) on feed size but feeds are capped at 1000 → fast enough
  - Also: client-side filter list. Unfollow event sent via WebSocket to app.
    App locally filters out posts from unfollowed users until feed refreshes.
```

### 5. Network Partition Between Cells

```
Problem: Cell A and Cell B can't communicate.
  Alice (Cell A) sends a DM to Bob (Cell B).

During partition:
  - DM written to Cell A's Kafka
  - Cross-cell Kafka bridge is down → DM stuck in Cell A
  - Bob (Cell B) doesn't receive the message

Resolution:
  - Alice sees "message sent" (her cell has it)
  - Bob doesn't see the message yet
  - When partition heals: cross-cell bridge replays queued messages
  - Bob receives the DM with original timestamp (appears in correct position)
  - No message loss, just delayed delivery
  
  Acceptable tradeoff: DM latency of minutes during a network partition
  is better than message loss or blocking the sender.
```

### 6. Cascading Failure from Slow Database

```
Problem: One PostgreSQL shard becomes slow (disk issue, long-running query).
  Requests to that shard start timing out at 5 seconds.
  Thread pool fills up → backpressure to API gateway → other users affected.

Cascade:
  Shard 42 slow → thread pool full → request queue grows → memory exhaustion
  → Pod OOM killed → load shifts to remaining pods → they also overwhelm
  → Entire Feed Service goes down → 500M users see errors

Prevention (defense in depth):
  1. Per-shard circuit breaker:
     If shard 42 latency > 200ms for 10 consecutive requests → circuit opens.
     Requests to shard 42 fail fast (5ms) instead of waiting 5 seconds.
     Only users whose data is on shard 42 are affected.

  2. Bulkhead isolation:
     Separate thread pools per shard (or per shard group).
     Shard 42 exhausting its pool does NOT affect shard 43's pool.

  3. Request deadlines:
     Every request carries a deadline: "this request must complete by T+200ms."
     If a downstream call would exceed the deadline, skip it.
     Return partial/degraded response rather than waiting.

  4. Load shedding:
     If overall error rate > 5%: start rejecting lowest-priority requests
     (explore, search) to protect core path (feed, DMs).
```

### 7. Data Consistency During Shard Migration

```
Problem: Moving shard 42 from old host to new host.
  During migration, both hosts have partial data.
  Reads could go to either → inconsistent results.

Solution: Double-write + cutover

Phase 1 (Dual Write):
  - New writes go to BOTH old and new host
  - Reads still go to old host
  - Background job copies existing data from old → new

Phase 2 (Shadow Read):
  - Writes still dual
  - Reads go to old, but also shadow-read from new (compare results, log diffs)
  - Fix any drift

Phase 3 (Cutover):
  - Update routing: reads and writes go to new host
  - Old host continues receiving dual writes for 1 hour (safety net)

Phase 4 (Cleanup):
  - Stop dual writes
  - Decommission old host's copy of shard 42

Total migration time per shard: ~2-4 hours with zero downtime.
```

---

## Summary: Key Design Decisions

| Decision | Choice | Why |
|---|---|---|
| Feed generation | Hybrid fan-out (push for regular, pull for celebrities) | Balances write amplification vs read latency |
| Post storage | PostgreSQL sharded by post_id | Strong consistency for mutations, mature tooling |
| Likes/Comments | Cassandra | Write throughput (58K/sec), no MVCC bloat, linear scale |
| Feed cache | Redis Sorted Sets | Sub-ms reads, natural ranking via scores, TTL eviction |
| Media storage | S3 + CDN + pre-signed upload | Decouples upload from API servers, infinite scale |
| ID generation | Snowflake (shard-embedded) | No coordination, shard routing from ID alone |
| Social graph | Dual adjacency lists + TAO cache | O(1) lookups for both directions, 99.9% cache hit |
| Messaging | Cassandra + WebSocket + connection registry | Partition-tolerant, ordered within conversation |
| Search | Elasticsearch via CDC | Decoupled indexing, sub-5s freshness |
| Notifications | Kafka → router → batching → APNs/FCM | Decouple, deduplicate, batch celebrity notifications |
| Fault isolation | Cell-based architecture | Blast radius capped at 50% of users per cell |
| Consistency model | Eventual for reads, strong for mutations | Acceptable: stale feed for 5s, but never lost DMs |