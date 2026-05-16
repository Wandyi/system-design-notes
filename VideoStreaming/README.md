# Video Streaming — Staff Software Engineer Interview Deep Dive

Built around the two reference systems you should be able to discuss cold:

- **YouTube** — 2.5B+ users, ~1 PB/day average through Google's CDN, video-on-demand at planetary scale, the canonical example for the **VOD playbook**.
- **JioHotstar** — set the new global live-streaming record of **72.5 million peak concurrent viewers** during the India vs New Zealand ICC Men's T20 World Cup 2026 final. The canonical example for the **live-streaming playbook**.

A staff engineer at any streaming company (YouTube, JioHotstar, Netflix, Twitch, Disney+, Amazon Prime Video, Cloudflare Stream, Mux) needs to reason about video at packet level (codecs, segments, manifests), at distribution level (CDN, peering, edge), at application level (player, ABR, DRM), and at operational level (pre-warming, multi-CDN orchestration, real-time analytics under tens-of-millions concurrency).

## Table of contents

| # | File | Topic | Why it matters |
|---|------|-------|----------------|
| 1 | [01-company-and-products.md](01-company-and-products.md) | YouTube, JioHotstar, Netflix, Twitch overview & business models | Frames "why this company" answers |
| 2 | [02-live-streaming-deep-dive.md](02-live-streaming-deep-dive.md) | JioHotstar 72.5M concurrent — the full architecture story | The headline interview topic |
| 3 | [03-vod-streaming-deep-dive.md](03-vod-streaming-deep-dive.md) | YouTube/Netflix VOD pipeline: ingest → transcode → store → serve | The other half of streaming |
| 4 | [04-protocols-hls-dash-cmaf.md](04-protocols-hls-dash-cmaf.md) | HLS, MPEG-DASH, CMAF, LL-HLS, LL-DASH, WebRTC, SRT, RTMP, MoQ | Protocol-level depth |
| 5 | [05-encoding-and-codecs.md](05-encoding-and-codecs.md) | H.264/AVC, H.265/HEVC, VP9, AV1, per-title encoding | Real bandwidth savings, real CPU/GPU cost |
| 6 | [06-cdn-and-edge.md](06-cdn-and-edge.md) | Multi-CDN, origin shield, peering, hierarchical caching | The bandwidth half of the cost equation |
| 7 | [07-adaptive-bitrate.md](07-adaptive-bitrate.md) | ABR algorithms — throughput, buffer, MPC, BOLA, Pensieve | Player-side depth |
| 8 | [08-drm-and-security.md](08-drm-and-security.md) | Widevine, FairPlay, PlayReady, CENC, forensic watermarking | Content owners demand it |
| 9 | [09-recommendations-and-ml.md](09-recommendations-and-ml.md) | YouTube two-tower model, candidate generation + ranking | Engagement engine |
| 10 | [10-realtime-analytics-and-chat.md](10-realtime-analytics-and-chat.md) | QoE telemetry, live chat at millions of concurrents | The "how do you know it works" |
| 11 | [11-system-design-questions.md](11-system-design-questions.md) | 12 staff-level prompts with full solutions | The bulk of an interview loop |
| 12 | [12-coding-problems.md](12-coding-problems.md) | Go-flavored problems: rate limiter, sharded cache, sliding window, manifest parser | Coding round prep |
| 13 | [13-staff-engineer-topics.md](13-staff-engineer-topics.md) | Scale tradeoffs, multi-region, cost, observability, migration | Staff-level signal |
| 14 | [14-quick-reference-cheatsheets.md](14-quick-reference-cheatsheets.md) | Codecs, ports, bitrates, latency tiers, RFC numbers | The night-before review |

## The 60-second elevator pitch — JioHotstar's 72.5M record

> "JioHotstar peaked at 72.5 million concurrent viewers during the India vs New Zealand T20 World Cup 2026 final. The backbone is ~800 microservices on AWS EKS with a custom autoscaler that brings up pods in roughly 30 seconds, a multi-CDN strategy with an in-house load optimizer that routes viewers to the least-congested CDN in real time (achieving roughly 90% CDN-cache offload), and an encoding pipeline that produces 4K/1080p/720p/480p/360p renditions simultaneously for ABR. The contribution feed comes in over dual independent paths — fiber primary plus satellite backup — into an ingest tier that hands off to live transcoders (AWS Elemental MediaLive in the canonical pattern), then packaged into HLS/DASH and pushed to edge. The structural advantage is Jio's own telecom network — 450M+ subscribers and Reliance fiber on which JioHotstar can place caches inside the carrier, not just outside it. Engineering choices that mattered: pre-warm capacity for known peaks rather than rely on autoscale, cap-and-shed at the edge under saturation, hierarchical caching with mid-tier origins, P2P CDN trials, and obsessive QoE monitoring."

## The 60-second pitch — YouTube

> "YouTube serves 2.5B+ users from Google's edge network (~3,000 PoPs), uses RTMP for live ingest and LL-HLS / DASH for delivery, encodes with custom silicon (their Argos VCU) for AV1 at scale, sustains 98–99% cache hit ratios via three-tier caching (edge → mid-tier → origin), and powers recommendations with a two-tower neural network (candidate generation then ranking, trained to optimize expected watch time, not CTR). Video-start time is sub-100 ms p50. The pipeline is microservices-based, encoding parallelized at the segment level, with rolling per-title ABR ladders that match content complexity rather than using one-size-fits-all rungs."

## Interview process (typical for streaming companies)

1. **Recruiter screen** — role fit, comp.
2. **Hiring manager technical screen** — resume deep-dive, one shallow design question.
3. **Coding rounds** (2 × 60 min) — DSA and at least one systems-flavored problem (rate limiter, sharded cache, sliding window — see [12-coding-problems.md](12-coding-problems.md)).
4. **System design round** (60 min) — usually a streaming-adjacent design (live, VOD, chat, analytics). See [11-system-design-questions.md](11-system-design-questions.md) for 12 worked examples.
5. **Staff-level deep dive** — protocols, codecs, ABR, CDN math, cost. The interviewer probes 2–3 layers deep on whatever you bring up.
6. **Cross-functional / behavioral / bar-raiser**.

## High-frequency topic clusters

| Cluster | Probability | Where to study |
|---------|-------------|----------------|
| Live streaming architecture at tens of millions concurrent | **Very high** for JioHotstar/Disney+/Twitch | [02-live-streaming-deep-dive.md](02-live-streaming-deep-dive.md) |
| HLS/DASH/CMAF protocol details | **Very high** everywhere | [04-protocols-hls-dash-cmaf.md](04-protocols-hls-dash-cmaf.md) |
| ABR algorithms and player tradeoffs | High | [07-adaptive-bitrate.md](07-adaptive-bitrate.md) |
| CDN architecture, multi-CDN, cost | **Very high** | [06-cdn-and-edge.md](06-cdn-and-edge.md) |
| Codecs and encoding economics | Medium-high | [05-encoding-and-codecs.md](05-encoding-and-codecs.md) |
| DRM and content protection | Medium (high if VOD studio content) | [08-drm-and-security.md](08-drm-and-security.md) |
| Recommendation ML | Medium-high at YouTube/Netflix | [09-recommendations-and-ml.md](09-recommendations-and-ml.md) |
| Real-time chat / presence | Medium-high (Twitch/Hotstar) | [10-realtime-analytics-and-chat.md](10-realtime-analytics-and-chat.md) |
| Pre-warming and capacity planning for predictable peaks | **Very high** for sports streamers | [02-live-streaming-deep-dive.md](02-live-streaming-deep-dive.md) |

## The four products to keep straight

- **YouTube** — VOD-first, live as a second-class but huge offering. Google's stack: their own CDN, their own codec silicon, two-tower recommendations.
- **JioHotstar** — live-first for sports (cricket above all), VOD on the side. AWS-heavy with Jio carrier integration. Built for short, predictable, massive peaks.
- **Netflix** — VOD-only, no live (yet, mostly). Famous for Open Connect (their own embedded CDN appliances inside ISPs), per-title encoding, A/B everything.
- **Twitch** — live-first for gaming/IRL, low-latency emphasis. Owned by Amazon, on AWS, with their own ingest/transcode pipeline tuned for sub-3-second latency.

Don't conflate "live cricket scale (millions concurrent for an hour)" with "VOD scale (billions of views over a day)" — they're solved by different architectures.

## Sources used to build this pack

- [JioHotstar hits 500M MAUs, 72.5M concurrency — Storyboard18](https://www.storyboard18.com/amp/brand-marketing/jiohotstar-hits-500-million-maus-72-5-million-concurrency-drives-%E2%82%B936248-crore-fy26-gross-revenue-96368.htm)
- [72.5M concurrent during T20 World Cup final — WION/ICC](https://www.wionews.com/sports/72-5-million-digital-concurrent-users-during-t20-world-cup-2026-final-india-vs-new-zealand-ahmedabad-break-global-streaming-records-icc-viewership-1774276432956)
- [How JioHotstar Streams IPL 2026 to 65M Concurrent on AWS — Abhishek Gautam](https://www.abhs.in/blog/jiohotstar-ipl-2026-streaming-infrastructure-65-million-concurrent-viewers-aws)
- [How JioHotstar Engineered 82.1 Crore Concurrent Streams — Pritam Roy](https://www.pritamroy.com/blog/posts/how-jiohotstar-engineered-821-crore-concurrent-streams-a-devops-deep-dive-into-t.html)
- [Decoding the scalability of JioHotstar — TechAhead](https://www.techaheadcorp.com/blog/decoding-the-incredible-scalability-of-disneyhotstar-app-system-structure-concurrency-more/)
- [CMY302 — Scaling hotstar.com for 25M concurrent (AWS re:Invent 2019)](https://d1.awsstatic.com/events/reinvent/2019/Scaling_Hotstar.com_for_25_million_concurrent_viewers_CMY302.pdf)
- [How Hotstar Scaled With 10.3M Concurrent — ScaleYourApp](https://scaleyourapp.com/how-hotstar-scaled-with-10-3-million-concurrent-users-an-architectural-insight/)
- [YouTube Tech Stack: 2.5B users at scale — vdocipher](https://www.vdocipher.com/blog/youtube-tech-stack-architecture/)
- [Deep Neural Networks for YouTube Recommendations (Covington et al., RecSys 2016)](https://dl.acm.org/doi/10.1145/2959100.2959190)
- [LL-HLS, CMAF, WebRTC: which is best — Cloudinary](https://cloudinary.com/guides/live-streaming-video/low-latency-hls-ll-hls-cmaf-and-webrtc-which-is-best)
- [Performance of Low-Latency DASH/CMAF and LL-HLS (at-scale conference)](https://atscaleconference.com/videos/performance-of-low-latency-dash-cmaf-and-low-latency-hls-systems/)
- [WebRTC live streaming at scale — Cloudflare blog](https://blog.cloudflare.com/webrtc-whip-whep-cloudflare-stream/)