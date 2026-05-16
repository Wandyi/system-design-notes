# 11 · System Design Questions — Streaming Edition

Twelve realistic system-design prompts. Same rubric as the Infoblox version: clarify → sketch → tradeoffs → failure modes. At staff level, naming the bottleneck and the failure-mode response is more important than the diagram.

## Rubric to apply to every question

1. Clarify scope, scale, latency SLO, geography, devices.
2. Sketch dataflow first, then storage.
3. State consistency model explicitly.
4. Name your failure modes.
5. Name your bottleneck with a number.
6. Multi-tenancy / cost / observability — mentioned voluntarily.

---

## Q1 · Design YouTube

**Clarifications**:
- VOD-focused, with live as a second class.
- Upload rate? 500+ hours/min.
- Catalog? Billions.
- Audience? 2.5B users globally.
- Latency? TTFV < 100ms p50.

**Architecture**:
- **Upload**: resumable HTTP upload, direct-to-storage with pre-signed URLs.
- **Pre-processing**: virus scan, ContentID fingerprint match, ASR for captions.
- **Transcoding**: per-segment parallel, per-title ABR ladders, custom silicon (Argos VCU equivalent) for AV1/VP9.
- **Packaging**: CMAF for both HLS + DASH.
- **Storage**: object store (Colossus / S3 / GCS) with global replication.
- **CDN**: three-tier (edge → mid-tier → origin) with own-CDN; 98–99% edge hit ratio.
- **Player**: pre-fetch on hover, lowest-bitrate-first startup, hybrid ABR.
- **Recommendations**: two-tower model (candidate generation + ranking + re-ranking).
- **Search**: inverted index over title/description; semantic search via embeddings.
- **Analytics**: Kafka → ClickHouse → dashboards.

**Tradeoffs**:
- DASH everywhere vs. HLS for Apple — both, via CMAF.
- Custom CDN vs. third-party — at YouTube scale, own. Otherwise start with CloudFront/Akamai.
- Per-title vs. fixed ladders — per-title for top X% of catalog; fixed for tail.
- ABR algorithm — hybrid (BBA + throughput + lookahead).

**Bottlenecks**:
- **Transcoding compute** — fleet-scale; addressed with custom silicon.
- **Storage** — multi-PB; addressed by replication + tiered storage.
- **CDN egress** — owned CDN keeps cost down; carrier peering.

**Failure modes**:
- Hot video breaks origin → origin shield + request collapsing.
- Recommendation service down → fall back to popularity-ranked homepage.
- CDN region fails → DNS steering shifts traffic.

---

## Q2 · Design JioHotstar Live Cricket Stream

(Same shape as the worked design in [02-live-streaming-deep-dive.md](02-live-streaming-deep-dive.md#216-worked-design-the-prompt-youll-get). Don't repeat; refer.)

Key staff-level points to emphasize:
- **Pre-warming** > autoscaling for predictable peaks.
- **Multi-CDN with in-house chooser**.
- **Carrier embedding** (Jio caches in Jio).
- **Load-shedding hierarchy** (4K → 1080p → 720p → low-data → audio-only).
- **Dual contribution paths** (fiber + satellite).

---

## Q3 · Design Live Chat for 20M Concurrent Connections

**Clarifications**:
- One mega-room (the cricket match) + many sub-rooms.
- Message rate: ~500K msgs/sec peak from the mega-room.
- Fanout multiplier: each message broadcast to 20M = 10TB/sec naive.
- Latency: < 2 s end-to-end.

**Architecture**:
- **Edge WS proxy** (Nginx / custom Go server with tuned epoll). Each node handles 100K conns.
- **Regional fanout broker** with room sharding.
- **Subsharding the mega-room** into 100 "lanes"; each user assigned to one lane; the lane shows a sampled subset of messages.
- **Kafka** between brokers for cross-region delivery.
- **Moderation**: async pipeline; flagged messages retracted via "delete" event.
- **Rate limiting**: 1 msg / 3s per user.
- **Compression**: permessage-deflate to cut bytes by ~70%.

**Tradeoffs**:
- WebSocket vs. SSE — WebSocket for bidirectional; SSE if it's mostly server-to-client + REST POSTs.
- Lossless delivery vs. drop — chat is droppable; trade for capacity.
- Real-time moderation vs. async — async + retraction is the standard compromise.

**Bottlenecks**:
- **Fanout bandwidth** — addressed by subsharding + compression + sampling.
- **Connection memory** — addressed by horizontal scaling of edge nodes.

**Failure modes**:
- Broker dies → connections re-establish to neighbor; messages buffered for N seconds.
- Moderation lags → low-confidence messages held briefly, high-confidence pass through.
- One subshard hot → rebalance lanes.

---

## Q4 · Design a Multi-CDN Chooser

(Covered in [06-cdn-and-edge.md](06-cdn-and-edge.md#615-worked-design-multi-cdn-chooser-for-a-50m-viewer-event). Recall:)
- Player beacons → aggregator → chooser → manifest service.
- Per-region per-CDN SLI computed every 10s.
- Hysteresis to avoid flapping.
- Fallback to static config if chooser is down.

---

## Q5 · Design a Video Encoding Pipeline for UGC at 500h/min

**Clarifications**:
- 500h/min × 60 = 30,000 minutes of video per minute uploaded.
- ABR ladder: 7 rungs × 3 codec families = 21 variants per video.
- Wall-clock target: median upload to playable in 10 min; long-tail to 1 hour.

**Architecture**:
- **Ingest**: tus / resumable upload, multipart S3.
- **Source split** into 6-second chunks aligned to GOP boundaries.
- **Job queue** (Kafka / SQS) with priority — short videos prioritized for low latency.
- **Encoder fleet** = mix of ASIC (most), GPU (peak burst), CPU (highest-quality long-tail).
- **Per-segment parallel encode** — N chunks × M renditions = K jobs, dispatched broadly.
- **Stitcher** glues per-rendition chunks back into per-rendition files.
- **Packager** produces CMAF/HLS/DASH; optional DRM-encrypt.
- **Catalog update** marks playable when first rung complete; remaining rungs come online later.

**Tradeoffs**:
- Equal-quality across rungs vs. progressive (publish lower rungs first) — progressive lets users watch sooner.
- Encode upon-upload vs. upon-first-view — Netflix encodes everything; YouTube also encodes all uploads but in tiers (popular content gets premium codec ladder).
- Per-title encoding cost vs. egress saving — per-title for everything above N daily views.

**Bottlenecks**:
- **Encoder compute** — addressed by ASIC + fleet scaling.
- **Storage write throughput** — addressed by S3-class object store.

---

## Q6 · Design VOD Recommendations (Two-Tower)

(Covered in [09-recommendations-and-ml.md](09-recommendations-and-ml.md). Repeat the bullets: candidate gen via ANN search → ranking via DNN predicting watch time → re-ranking for diversity/freshness/business rules. Cold-start via content features + popularity. Train on logged interactions with bias correction.)

---

## Q7 · Design DVR / Catch-Up for Live

**Clarifications**:
- Users want to "rewind" the live stream to see a replay.
- Window: 30 min to 4 hours.
- Storage: rolling buffer per stream.
- Player UX: seek bar, jump-to-live button.

**Architecture**:
- **Encoder writes segments to object store** as they're produced.
- **Manifest service** maintains two manifest types:
  - **Live**: sliding window, latest N segments visible.
  - **DVR**: all segments since stream start.
- **Player** can request either; UI lets user toggle.
- **CDN cacheability**:
  - Segments are cached normally (long TTL once written).
  - Manifests differ — live manifest short TTL, DVR manifest may be longer per range.
- **DRM**: same content keys for live and DVR; per-session licenses with policy for "rewind allowed".
- **Storage**: warm storage for the live window; can migrate to cold after the event.

**Tradeoffs**:
- DVR window size — longer = more storage + manifest size grows.
- Per-user DVR position tracking — small per-user state needed.
- Time-shifted ads — ad break can be re-decisioned on rewind ("show different ad on second watch") with implications for SSAI cache.

**Failure modes**:
- Segment write failed mid-stream → gap in DVR; player skips with audible click; logged for forensics.
- Storage tier failure → live keeps working; DVR seek fails gracefully.

---

## Q8 · Design Server-Side Ad Insertion (SSAI)

**Clarifications**:
- Live stream + dynamic ad breaks.
- Personalized ad selection per user.
- Ad-blocker resistance is the value proposition.
- Latency: ad break decision in <1s.

**Architecture**:
- **Stream has SCTE-35 markers** (broadcast ad cue points).
- **Ad decisioning service** (in-house or third-party VAST/VPAID-class) selects creatives per user per break.
- **Manifest stitcher** generates per-user manifest with ad segments inserted at break boundaries.
- **Ad content** lives in object store, encoded into the same ABR ladder.
- **Shared segment cache** — even though manifests differ per user, the underlying segment URLs are shared (only the *sequence* in the manifest differs).
- **Beaconing** — ad-impression beacons fired by the player (or stitcher) for billing accuracy.

**Tradeoffs**:
- Per-user manifest vs. shared — per-user is necessary for personalization; mitigate cache cost by sharing segments.
- Pre-decisioned vs. on-demand — pre-decision the break in cohorts (every "user group" gets the same ad sequence); on-demand only for high-value personalization.
- Ad creative caching — ads must be pre-fetched at the CDN before the break or you get cold-cache stalls during ad breaks (deadly UX).

**Failure modes**:
- Ad-decision service down → fall back to default house ad (always cached).
- Bad creative (too short / corrupted) → skip; alert; bill the advertiser nothing.

---

## Q9 · Design QoE Telemetry

(Covered in [10-realtime-analytics-and-chat.md](10-realtime-analytics-and-chat.md#108-worked-design-telemetry-for-a-50m-viewer-event). Recall: beacons → Kafka → Flink → ClickHouse + dashboards; sample under load.)

---

## Q10 · Design Content Moderation at Upload Time

**Clarifications**:
- UGC platform, 500h/min uploads.
- Categories: copyright (ContentID), nudity, violence, hate, spam.
- Latency: most content available within minutes of upload.
- Some categories require human review.

**Architecture**:
- **Fingerprinting** (audio + video perceptual hash) → match against reference database (copyright); near-real-time.
- **Classifier ML models** (vision + audio + text-on-screen via OCR + ASR-of-audio) → score each category.
- **Threshold-based routing**:
  - High confidence violation → auto-block, notify uploader.
  - Medium confidence → publish with restricted distribution, queue for human review.
  - Low confidence → publish normally; spot-check via sampling.
- **Human review tier** — workforce of moderators with priority queues.
- **Appeals process** — uploaders contest decisions; re-review.
- **Feedback loop** — moderator decisions become training data for next-gen classifier.

**Tradeoffs**:
- Precision vs. recall — false negatives let bad content through; false positives anger creators. Both are bad; tune per category.
- Pre-publish hold vs. post-publish removal — UGC platforms typically publish optimistically; remove on detection.
- Cost of human review — limit to ambiguous cases; classifier handles most.

**Failure modes**:
- Classifier model regression → catch via canary deployment + monitoring of moderation-action rates.
- Reference-DB outage → fingerprint matches stall; uploads still publish, ContentID retrofits when DB returns.

---

## Q11 · Design a Bookmark / Continue-Watching Service

**Clarifications**:
- Per-user, per-video position tracking.
- Cross-device sync.
- Read: very high (homepage row).
- Write: every 5–10 s per active session.
- Consistency: read-your-writes per device; eventual cross-device.

**Architecture**:
- **Player** POSTs `{user, video, position, updated_at}` every N seconds.
- **Write API** validates, enqueues to Kafka.
- **Writer** consumes Kafka, updates KV store keyed by `(user_id, video_id)`. Last-write-wins by `updated_at`.
- **Cache** — per-user recent bookmarks cached at edge; TTL ~30s.
- **Read API** combines cache + DB for cold reads.
- **Cleanup** — bookmarks > 1 year old eligible for pruning.

**Tradeoffs**:
- DynamoDB / Cassandra / Spanner / Postgres for the KV store — choose by region distribution + cost.
- Sync via push (server tells device) vs. pull (device polls) — pull on app foreground is usually enough.
- Last-write-wins vs. CRDT — last-write-wins is sufficient; conflicts are extremely rare.

**Bottlenecks**:
- **Write QPS** dominates — addressed by Kafka buffer + horizontally-scaled writers.

---

## Q12 · Design Notifications for a Live Sports Event

**Clarifications**:
- 70M subscribers want pushes for match starts / wickets / final-over.
- APNs / FCM per-app throughput limits (Apple ~9K push/sec; FCM scales but rate-limits).
- Latency: end-to-end < 30 s for routine alerts; < 5 s for "match starting now".

**Architecture**:
- **Topic-based subscription** — users subscribe to "India matches", "T20WC final", etc.
- **Notification dispatcher** holds the recipient list per topic.
- **Throttled fan-out** — stagger pushes over a 1–3 minute window to stay within provider limits.
- **Multi-provider** — multiple APNs creds, multiple FCM projects.
- **In-app fallback** — when the app opens, fetch any missed alerts from a "notifications" API endpoint.
- **Retry** for transient failures (exponential backoff).

**Tradeoffs**:
- Push everyone at once (overwhelming providers) vs. stagger (some users get it 30s later) — stagger always.
- Topic-based vs. per-user query — topic-based scales better; per-user only for highly targeted alerts.
- Server-side rate limit vs. client-side dedup — both; client should dedup by notification ID to avoid duplicates if a retry succeeds after the first attempt.

**Failure modes**:
- FCM throttles → fall back to in-app + email digest.
- Some users don't get push → in-app surface still works.

---

## Cross-cutting interviewer asks

- "What changes for 10× scale?" — name the next bottleneck.
- "What if a region fails?" — failover and degraded-mode behavior.
- "How do you cost this?" — egress dominates; engineering choices that move the bill.
- "What experiments would you run?" — A/B, what's the metric, sample size.
