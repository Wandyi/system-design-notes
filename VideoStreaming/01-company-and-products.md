# 1 · Company & Product Landscape

A staff engineer is expected to know not just architecture but the *business* of streaming — because the architecture is downstream of the business model. Live sports streamers (JioHotstar, DAZN, Peacock for NFL) face very different problems from VOD studios (Netflix, Disney+) or UGC platforms (YouTube, TikTok) or live UGC (Twitch, Kick). The interviewer is checking that you can talk about why an architecture exists, not just what it is.

## 1.1 JioHotstar — the live-sports-at-massive-scale player

JioHotstar is the result of the Jio/Reliance + Disney's Star India joint-venture consolidation. It's the largest video-streaming service in India and now globally significant.

Headline numbers (FY26):
- **500M+ monthly active users**.
- **72.5M peak concurrent viewers** during the ICC Men's T20 World Cup 2026 final (India vs New Zealand) — the new global live-streaming record.
- The previous internal record (65M) was set during the India vs England semi-final of the same tournament.
- **₹36,248 crore (≈US$4.3B) gross revenue** for FY26.
- Mix of free, ad-supported, and subscription tiers.

What makes JioHotstar architecturally interesting:

- **Concurrency, not aggregate**: cricket peaks are extreme — millions of viewers logging in within minutes of toss, all watching the same content, then dropping off at the same time. 
- Pre-warming and predictable load curves dominate the engineering problem.
- **Carrier integration**: Jio operates the underlying telecom — 450M+ wireless subscribers, the largest 4G/5G base in India. 
- JioHotstar can place caches inside Jio's network (PoPs colocated with carrier aggregation), not just at the public Internet edge. That changes the cost equation.
- **Multi-CDN load balancing**: they don't rely on a single CDN. Their in-house load optimizer routes viewers to the least-congested CDN per region, per minute.
- **Single-event scale**: a cricket final is 3-4 hours of one stream. A YouTube day is billions of streams of millions of videos. The bottlenecks are different.

## 1.2 YouTube — the VOD-and-everything UGC platform

YouTube is the canonical streaming system. Two design influences:

- **UGC scale**: 500+ hours of video uploaded *per minute*. Transcoding, storage, indexing, moderation at that ingest rate is itself a planet-scale problem.
- **Long-tail consumption**: the head is hot (Mr. Beast videos do 100M views), but the tail is enormous and individually cold. Cache strategy must serve both.

Their stack:

- **Google's global infrastructure** — ~3,000 edge locations, B4/B2 backbone, Spanner / Bigtable / Colossus underneath.
- **Custom encoder silicon** — the **Argos VCU** (Video Coding Unit), an ASIC designed for transcoding at fleet scale. Around 10× more efficient than commodity x86 for VP9/AV1 encoding.
- **Three-tier cache hierarchy**: edge cache → mid-tier (regional) → origin (Colossus). Most queries terminate at edge or mid-tier.
- **Two-tower recommendation model**: candidate generation (millions → thousands) then ranking (thousands → tens). Optimized for expected watch time, not CTR.
- **DASH for delivery on most platforms; HLS reserved for Apple devices.**
- **Live ingest via RTMP**, served via LL-HLS for low-latency live.

## 1.3 Netflix — the VOD-only operational masterclass

Netflix doesn't really do live (with rare exceptions like the Tyson-Paul fight). Their reputation is for operational excellence on VOD:

- **Open Connect Appliances (OCA)**: physical cache servers Netflix gives to ISPs to host inside their networks. Practically eliminates the cost of crossing ISP boundaries for popular content.
- **Per-title encoding**: instead of a fixed bitrate ladder, every title gets its own optimal ladder based on its complexity (a still-image talking-head video doesn't need the same bitrates as an action movie). 20–40% bandwidth savings reported.
- **Chaos engineering**: Chaos Monkey, Simian Army — practices originally Netflix's, now industry standard.
- **A/B everything**: artwork, codec choice, ABR algorithm, preview behavior — all gated by experiments.

## 1.4 Twitch — live-first, low-latency, interactive

Twitch is the canonical *interactive* live streaming platform: streams must be low-latency because chat reacts in seconds, and viewers compare delays with each other.

- **Sub-3-second latency target** for most live streams; sub-second optional.
- **Ingest in RTMP** (legacy) and **WebRTC** (low-latency mode).
- **LL-HLS** delivery with chunk-aligned segments.
- **Chat at millions of messages/sec** for big streams — a separate engineering problem.
- **Owned by Amazon** since 2014; backed by AWS.

## 1.5 Disney+ / Hulu / Max / Peacock / Prime Video

Big VOD-and-live SVOD players. Tech-wise they're variations on:
- Studio-grade DRM with very strict windowing rules.
- Multi-CDN with sophisticated chooser logic.
- Strong per-title encoding pipelines.
- Live sports rights pushing them into the JioHotstar-style "sudden concurrent peak" problem.

Peacock's NFL exclusive games and Prime Video's Thursday Night Football have made them invest heavily in live-at-scale.

## 1.6 Cloudflare Stream / Mux / Bitmovin / AWS Elemental — the picks-and-shovels tier

Not consumer-facing, but worth knowing because their products *are* what most other streamers run on top of.

- **AWS Elemental** — MediaLive (live encode), MediaPackage (packager), MediaConvert (VOD transcode), MediaConnect (transport). The default toolkit for someone building a streamer on AWS.
- **Mux** — a Stripe-for-video. API-first VOD and live, with Mux Data for QoE analytics.
- **Bitmovin** — high-quality encoder + player SDK; popular for premium studios.
- **Cloudflare Stream** — bundled storage + transcoding + delivery + DRM with WebRTC for ultra-low-latency.

## 1.7 Business-model implications for architecture

| Model | Architectural emphasis |
|-------|------------------------|
| **Ad-supported (YouTube, Hulu, Hotstar free tier)** | Server-side ad insertion (SSAI), strong audience segmentation, brand-safety controls |
| **Subscription (Netflix, Disney+, JioHotstar Premium)** | Account management, entitlement, DRM, multi-device sessions |
| **Live UGC (Twitch, Kick)** | Low-latency ingest + chat + monetization (subs, bits, ads), creator tools |
| **Live sports (JioHotstar, Peacock, DAZN)** | Predictable peaks, rights enforcement, latency parity with broadcast (or better), multi-language audio |
| **VOD studio (Netflix, Max, Apple TV+)** | Per-title encoding, strong DRM, complex windowing, parental controls, captions |

## 1.8 What "scale" actually looks like (numbers to anchor on)

- A 1080p stream at H.264 is roughly **5 Mbps**.
- 72.5M concurrent at 5 Mbps = **~362 Tbps** of egress, *if everyone is on 1080p*. In practice many are on lower bitrates (mobile/India context), so real egress was likely **100–200 Tbps**.
- Cricket final: ~3.5 hours. 100 Tbps × 3.5h = ~157 PB delivered in that window.
- YouTube: ~1 PB/day average through their CDN, **10+ PB/day** at major-event peaks.
- Netflix: about **15% of global downstream Internet bandwidth** at evening peak in major markets, per Sandvine reports historically.
- A single sports event can produce **3–5× the peak traffic** of an entire normal day on a streaming service.

## 1.9 Why interviewers love this domain

- It's a beautiful intersection: networking + storage + distributed systems + ML + UI + business model.
- The numbers are crisp: bitrates, latencies, segment durations.
- Failure modes are visible and viral — buffering during a sports moment is a public-relations event.
- The same primitive (a video segment) flows through every layer of the stack, so a candidate's depth shows.

## 1.10 Must-internalize

- Two reference systems: **YouTube (VOD UGC, planetary scale)** and **JioHotstar (live sports, extreme concurrency)**.
- 72.5M concurrent isn't bigger than YouTube's daily aggregate — it's a different problem (concurrency vs. throughput).
- Carrier integration is a strategic moat (Jio for Hotstar, Open Connect for Netflix).
- Per-title encoding, multi-CDN, sub-100ms TTFV are table-stakes vocabulary.
- The business model shapes the architecture: ads → SSAI; subscription → DRM + entitlement; live → predictable peaks; UGC → ingest at hundreds-of-hours-per-minute.

---

## Sources

- [Storyboard18 — JioHotstar 500M MAUs, 72.5M concurrency, FY26 revenue](https://www.storyboard18.com/amp/brand-marketing/jiohotstar-hits-500-million-maus-72-5-million-concurrency-drives-%E2%82%B936248-crore-fy26-gross-revenue-96368.htm)
- [WION — 72.5M concurrent during T20 World Cup 2026 final](https://www.wionews.com/sports/72-5-million-digital-concurrent-users-during-t20-world-cup-2026-final-india-vs-new-zealand-ahmedabad-break-global-streaming-records-icc-viewership-1774276432956)
- [ICC — global streaming record set on JioHotstar](https://www.icc-cricket.com/media-releases/icc-men-s-t20-world-cup-2026-sets-new-global-streaming-record-on-jiohotstar-during-india-england-semi-final)
- [vdocipher — YouTube tech stack](https://www.vdocipher.com/blog/youtube-tech-stack-architecture/)
- [Netflix Open Connect](https://openconnect.netflix.com/)
- [AWS re:Invent 2019 CMY302 — Scaling Hotstar.com (PDF)](https://d1.awsstatic.com/events/reinvent/2019/Scaling_Hotstar.com_for_25_million_concurrent_viewers_CMY302.pdf)