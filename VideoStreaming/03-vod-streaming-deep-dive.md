# 3 · VOD Streaming — The YouTube/Netflix Playbook

VOD (video-on-demand) is the other half of the streaming world. The bottlenecks are different from live: ingest happens once-per-video over a long horizon, transcoding is batch and can be fully parallelized, and the access pattern is a long-tailed catalog rather than a single hot event. The interview hat to wear: storage economics, encoding economics, recommendation-driven access patterns, and how Netflix-class operational excellence is achieved.

## 3.1 The shape of the problem

- **Ingest is asynchronous**: hours-to-days from upload to playable. Engineers optimize for cost and quality, not real-time latency.
- **Catalog is huge**: YouTube has billions of videos; Netflix has tens of thousands of premium titles. Different storage strategies.
- **Long tail**: 80% of views come from 20% of content (Pareto-ish). The hot 20% must be globally cached; the cold 80% can be on cheaper storage at higher latency.
- **Access pattern is recommendation-driven**: people watch what the homepage shows them. Recommendations and CDN cache placement should be co-designed.
- **Playback latency matters in a different way**: not stream-vs-broadcast latency, but **time-to-first-video** (TTFV) when the user hits play. YouTube aims for sub-100ms p50.

## 3.2 The end-to-end pipeline

```
   Upload (mobile/web/RTMP for live-then-VOD)
       |
       v
   Raw upload landing (S3 / Google Cloud Storage / Colossus)
       |
       v
   Pre-processing: virus scan, MIME check, content moderation pre-filter
       |
       v
   Transcode farm (per-segment parallel encoding into ABR ladder + codecs)
       |
       v
   Per-title analysis (complexity → ladder tuning) + thumbnail generation + caption alignment
       |
       v
   Packager: HLS + DASH + CMAF + DRM-encrypt
       |
       v
   Origin (S3 / Colossus / GFS-class object store)
       |
       v
   Push-or-pull CDN distribution (hot content pre-pushed; cold pulled on demand)
       |
       v
   Player TTFV path: catalog API → entitlement → manifest → first segment → first frame
       |
       v
   Telemetry → recommendations training data → next-up recommendation
```

## 3.3 Upload and ingest

The upload tier is its own subsystem with engineering details that show up in interviews:

- **Resumable uploads** — the canonical protocol is Google's *resumable upload* over HTTP, or **tus.io**. Big files (gigabytes) must survive flaky networks.
- **Direct-to-storage uploads** with pre-signed URLs so the upload server is bypassed entirely for the data path.
- **Multipart uploads** to S3 / GCS — parallel chunk upload, integrity-checked.
- **Server-side virus scan** and ad-policy / copyright pre-check (perceptual-hash matching against ContentID / similar).
- **Quality validation** — frame-rate, codec, audio sanity checks; reject malformed at ingest, not at the encoder.

At YouTube scale: 500+ hours of video per minute uploaded. The ingest tier is a planet-scale system on its own.

## 3.4 Transcoding — the parallelization story

The defining property: VOD transcoding is **embarrassingly parallel**. A 2-hour movie can be split into 1200 six-second chunks; each chunk encoded independently on a different machine; then stitched back. Wall-clock time = single-chunk encode time + scheduling overhead, not total-runtime time.

The standard pipeline:

1. **Demux** the source: separate video, audio tracks, subtitle tracks.
2. **Split** the video into chunks. Chunks must align to scene boundaries or at least to GOP/IDR boundaries; the encoder needs IDRs at boundaries so chunks are independently decodable.
3. **Dispatch** each chunk × each rendition to a worker. A 6-rendition ladder × 1200 chunks = 7200 jobs for a single movie.
4. **Encode** at the worker. The actual hard work: codec choice, two-pass encoding for quality, per-title rate-control parameters.
5. **Merge** chunks back into per-rendition files; verify continuity.
6. **Package** into HLS/DASH/CMAF.
7. **Encrypt** under DRM keys; publish to origin; mark catalog entry as playable.

YouTube's optimization is **specialized hardware** — the **Argos Video Coding Unit (VCU)**, a Google-designed ASIC, brings per-watt encoding efficiency far above commodity GPU/CPU. At fleet scale this saves serious money and power.

Netflix's optimization is **per-title encoding**. Instead of a fixed bitrate ladder, each title gets a custom ladder based on its visual complexity. A still-camera talk-show gets lower bitrates than an explosion-heavy action movie at the same perceived quality. They publicly reported 20–40% bandwidth savings.

A more recent evolution: **shot-based encoding** — split by scene, encode each scene with its own parameters, glue back together. Even better quality-per-bit. Netflix discussed this for AV1 rollout.

## 3.5 Storage tiers

| Tier | What lives here | Latency to first byte | Cost |
|------|------------------|----------------------|------|
| **CDN edge cache** | Hot popular content | 5–50 ms | Highest per byte (but small footprint) |
| **CDN mid-tier / origin shield** | Warm content + cold misses passing through | 30–100 ms | Medium |
| **Origin object store (S3 / Colossus)** | Canonical encoded files | 50–200 ms | Standard |
| **Archive / cold storage (Glacier / Coldline)** | Source masters; cold catalog | Seconds to minutes | Cheap |

A staff-level question: "where does a 5-year-old YouTube video with 12 views/year live?" Answer: probably not at the edge, possibly evicted even from mid-tier; the first request after a long gap may go to origin and incur a few hundred ms TTFV penalty.

## 3.6 The TTFV path — start a video in 100 ms

Time-to-first-video is the dominant UX metric for VOD. YouTube's reported p50 is sub-100ms. To hit that:

1. **DNS resolution** — TTL-tuned, pre-warmed via OS DNS cache. Goal: 0 ms in steady state.
2. **TLS handshake** — TLS 1.3 with session resumption (1-RTT or 0-RTT); QUIC is even better (no 3-way handshake).
3. **Catalog / metadata API** — pre-fetched on page load before user even clicks. Cached at CDN with short TTL.
4. **Entitlement / auth** — token verified locally where possible; only revoke-check goes to a server.
5. **Manifest fetch** — served from CDN edge; size ~10 KB, gzipped.
6. **First segment** — the first chunk pre-decoded into player buffer; often pre-fetched at lowest bitrate to get *something* playing fast.
7. **Decoder warmup** — hardware decoder initialization; tens of milliseconds on older devices.
8. **First frame paint** — finally visible.

Key optimizations:
- **Speculative pre-warm**: when the user hovers on a thumbnail, start fetching its manifest and first segment.
- **Lowest-bitrate-first**: start at the bottom rung; ABR upshifts after a few seconds. Trades first-frame visual quality for instant start.
- **Single-RTT critical path**: manifest + first segment + entitlement in one batch if possible.

## 3.7 ABR ladder design for VOD

The interplay between encoding economics and player experience:

- **Number of rungs** — too few = poor adaptation; too many = encoding cost and CDN cache footprint. 5–7 is a typical sweet spot.
- **Rung spacing** — perceptual-quality spacing (VMAF-based, ~6 points apart) beats linear bitrate spacing.
- **Codec-specific ladders** — H.264 ladder + HEVC ladder + AV1 ladder, each with its own rungs because their quality-per-bit differs.
- **Per-title customization** — see Netflix above.

A ladder example for a feature film:

| Rendition | Resolution | H.264 bitrate | HEVC bitrate | AV1 bitrate |
|-----------|------------|---------------|--------------|-------------|
| 4K | 3840×2160 | n/a | 12 Mbps | 8 Mbps |
| 1440p | 2560×1440 | n/a | 8 Mbps | 5 Mbps |
| 1080p | 1920×1080 | 6 Mbps | 3.5 Mbps | 2.5 Mbps |
| 720p | 1280×720 | 3 Mbps | 1.8 Mbps | 1.2 Mbps |
| 480p | 854×480 | 1.2 Mbps | 0.8 Mbps | 0.5 Mbps |
| 360p | 640×360 | 0.6 Mbps | 0.4 Mbps | 0.25 Mbps |
| 240p | 426×240 | 0.3 Mbps | 0.2 Mbps | 0.15 Mbps |

(Numbers are illustrative; real ladders vary by content.)

## 3.8 Long-tail caching strategy

A naive caching strategy stores all videos at all edges. Doesn't work — the catalog is bigger than any single edge cache.

Better: **popularity-tiered caching**.

- **Tier 0 — global hot**: top ~1000 videos worldwide, pre-pushed to every edge.
- **Tier 1 — regional hot**: next ~10K-100K per region, cached at regional mid-tier.
- **Tier 2 — long tail**: pulled from origin on first request in a region; cached at mid-tier for a few days based on access pattern.
- **Tier 3 — cold**: only at origin / archive.

The dispatcher uses video metadata (popularity, recency, recommendation likelihood) to decide what tier a video belongs to. ML-driven cache placement is a thing — predict which videos in a region will be hot in the next hour, push them.

Netflix's Open Connect Appliances (OCAs) push fresh popular content into ISPs nightly, when bandwidth is cheap. The next day's hot content is already at the edge.

## 3.9 Recommendations as a caching signal

Because watchers go where the recommendation engine sends them, recommendations actively shape the cache. The engineering pattern:

- After recommendations are computed for a region, the cache placement system pre-warms the top-N candidate videos in that region's edge.
- If the recommender suddenly surfaces a new viral video (e.g., a creator's video gets trending) the cache control plane is notified to promote it in real-time.

This co-design — recommendations as a cache prediction signal — is what makes YouTube/Netflix-class CDN economics work.

## 3.10 DRM and key management for VOD

For premium VOD (Netflix, Disney+, etc.):

- Content is encrypted under per-title content keys.
- Keys are wrapped under DRM-system keys (Widevine / FairPlay / PlayReady).
- Player requests keys at playback time from a license server, presenting an entitlement token.
- License server enforces concurrency limits, output-protection (HDCP), offline-download rules, geographic restrictions.

UGC platforms (YouTube) generally don't DRM most user content — only when a creator opts in or for paid content. Free YouTube videos are unencrypted.

See [08-drm-and-security.md](08-drm-and-security.md) for depth.

## 3.11 Catalog and metadata services

Behind playback there's a sprawling metadata estate:

- **Asset catalog** — title, description, cast, language tracks, captions, image assets.
- **Entitlement / rights** — who can watch what where? Studio contracts define geo / device / time windows.
- **Playback session** — concurrency limits ("only 2 streams per account"), bookmark / continue watching.
- **Search index** — Elasticsearch / Lucene / Solr at scale.

These are normal CRUD-shaped systems at large scale: read-heavy, eventually consistent across regions, with strict tenant/account isolation.

## 3.12 Continue-watching and bookmarks

A small feature with surprising depth:

- Every player periodically POSTs "position = X seconds" to a bookmarks API.
- Reads are very common (homepage rail shows "Continue watching").
- Writes need to be cheap (every 5–10s during playback × every user). Tens of thousands of writes per second.
- Eventual consistency is fine — losing a few seconds of bookmark is acceptable. Cross-device sync within 10–30 seconds is expected.

Architecture: writes go to a queue (Kafka), consumed by a writer that updates a KV store keyed by `(user_id, video_id)`. Reads from a per-user cache; warm on login.

## 3.13 Captions, subtitles, and accessibility

Often glossed over but interview-worthy:

- **Caption formats**: WebVTT for web/HTML5, TTML for broadcast and Netflix/Disney+, SRT as a legacy.
- **Caption delivery**: in-band via fMP4 / CMAF tracks, or out-of-band sidecar files referenced from the manifest.
- **Multi-language**: separate audio tracks and separate caption files per language.
- **ASR for UGC**: YouTube auto-generates captions for uploaded videos using ASR. Quality has improved enormously since 2020.
- **Audio descriptions** for blind/low-vision users; a separate audio track with narrated descriptions.

## 3.14 Thumbnails

Thumbnails are themselves a small ML system:

- Sampled frames extracted at upload.
- Best thumbnail chosen via a model trained on "which thumbnail predicts click-through".
- A/B tested over time.
- Served from a CDN like any other asset.

## 3.15 The control plane vs. the data plane

For a staff-level discussion: cleanly separate.

- **Data plane**: serves video segments. Massive scale. Stateless edge servers. Optimized for cache hit ratio.
- **Control plane**: catalog APIs, entitlement, billing, recommendations, search. Lower scale (per-user actions, not per-segment). Stateful and complex.

The data plane should never be in the critical path of a control-plane outage. If the recommender is down, the user can still play the video they're watching. The watching path must be resilient to API failures.

## 3.16 Worked design — VOD service for a small streamer

**Prompt**: "Design a VOD service for 10M users, 100K titles, mixed device support."

A staff answer:

1. **Clarify**: ad/sub/free, peak concurrent (maybe 1M), DRM requirements, devices, regions.
2. **Ingest**: tus-resumable upload to S3, async transcode trigger.
3. **Transcode**: per-chunk Lambda / containers; per-title ladder generation.
4. **Packaging**: CMAF + DASH + HLS; Widevine + FairPlay (+ PlayReady if Windows).
5. **Origin**: S3 with origin-shield (CloudFront's origin shield works; otherwise a self-hosted tier).
6. **CDN**: single CDN to start (CloudFront / Akamai), multi-CDN at scale.
7. **Players**: native SDKs per platform (Exoplayer Android, AVPlayer iOS, Shaka/HLS.js web).
8. **Catalog**: Postgres + Elasticsearch for search.
9. **Entitlement**: signed JWT tokens; license server with concurrency check.
10. **Recommendations**: collaborative filtering to start, two-tower model when data justifies.
11. **Telemetry**: Mux-style QoE beacons; Kafka → ClickHouse.
12. **Costs**: egress dominates; multi-CDN later, CMAF up-front to halve packaging costs.

## 3.17 Must-internalize

- VOD = batch transcoding + smart caching + long-tail economics; not the same problem as live.
- Per-segment parallelization is what makes VOD encoding scale.
- Per-title (or shot-based) encoding is the bandwidth-savings unlock.
- TTFV is the dominant UX metric; pre-fetch on hover, start at lowest rung for instant play.
- Popularity-tiered caching matches the long tail.
- Recommendations and CDN cache should be co-designed.
- Control plane and data plane separation: data plane survives control-plane outage.

---

## Sources

- [YouTube tech stack — vdocipher](https://www.vdocipher.com/blog/youtube-tech-stack-architecture/)
- [How YouTube Works: Video Streaming Architecture Deep Dive — DEV](https://dev.to/matt_frank_usa/how-youtube-works-video-streaming-architecture-deep-dive-44bc)
- [How YouTube Streams Billions Every Day — Servify](https://medium.com/@servifyspheresolutions/how-youtube-streams-billions-of-videos-every-day-inside-googles-cdn-9cf8f5b84bfb)
- [Designing YouTube — 0xkishan](https://www.0xkishan.com/blogs/designing-youtube)
- [Netflix per-title encoding (Netflix Tech Blog historical)](https://netflixtechblog.com/per-title-encode-optimization-7e99442b62a2)
- [Netflix Open Connect Appliances](https://openconnect.netflix.com/)
- [tus.io resumable uploads](https://tus.io/)
