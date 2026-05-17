# 2 · Live Streaming — The JioHotstar 72.5M Concurrent Story

This file is the headline interview topic: how do you stream one live event to tens of millions of concurrent viewers? 
JioHotstar's 72.5M peak during the India vs New Zealand T20 World Cup 2026 final is the most concrete case study in the world right now. Walk through it end-to-end and you've covered most of what a staff interviewer wants to hear.

## 2.1 The shape of the problem

A cricket final is *not* a hard scaling problem in absolute throughput — it's a hard *concurrency* problem with a brutal *shape*:

- **Predictable peak time** — toss is at a known minute; the next 3.5 hours of load are known to within ±15%.
- **Synchronous behavior** — viewers join within a 5-minute window around toss; they leave within a 5-minute window after the final wicket.
- **Single content surface** — one stream (well, a few: English, Hindi, regional, low-data) gets effectively *all* the traffic. 
  - Cache hit ratios for the live stream itself approach 100%; cache helps less than you'd hope because the content is constantly being created.
- **High emotional cost of failure** — buffering during the last over is a national incident.

The architecture is sized for this shape, not for steady-state.

## 2.2 The end-to-end pipeline (the picture in your head)

```
   Stadium / Broadcast truck (Star Sports OB van)
       |
       | dual independent links: dedicated fibre (primary) + satellite (backup)
       v
   Contribution / Ingest (multi-region edge ingress, e.g. AWS MediaConnect / SRT)
       |
       v
   Live transcoder farm (AWS MediaLive or in-house) — produces ABR ladder + AV1/HEVC/H.264
       |
       v
   Packager (HLS + DASH manifests + CMAF chunks)   ----  DRM key broker (Widevine/FairPlay/PlayReady)
       |                                                    |
       v                                                    v
   Origin (S3 + custom origin shield)              encrypted segments
       |
       v
   Multi-CDN fanout (Akamai + CloudFront + Jio + others) with in-house load optimizer
       |
       v
   ISP / Carrier edge (Jio embedded caches inside Jio's network)
       |
       v
   Player (Android / iOS / Web / Smart TV / JioTV)
       |
       v
   QoE telemetry stream  --->  Kafka  --->  ClickHouse / real-time dashboards
```

## 2.3 Contribution — getting the feed in

The Star Sports OB van produces a clean program feed and pushes it out over two independent paths:

- **Primary**: dedicated fiber to JioHotstar's ingest PoP. Often via a managed connectivity provider (LumenCBS Sports / TPN / etc.) with diverse routes.
- **Backup**: satellite uplink. Higher latency, lower bitrate, but completely independent failure domain.

Both paths land in the ingest tier, which performs:

- **Hot/cold failover** — automatic switching if the primary feed drops a frame for more than N seconds (the threshold is small — broadcast-quality failover is sub-second-aware).
- **Quality verification** — checksums, signal-quality metrics, audio-loudness checks (LUFS).
- **Multi-language mixing** — separate audio tracks for English / Hindi / Tamil / Telugu / Bengali / Marathi etc.; PiP feeds; Hindi commentary on a separate track.

Contribution protocols you should be able to name:

- **SRT (Secure Reliable Transport)** — UDP-based, error-corrected, encrypted. 1–2 second latency. Becoming the standard for IP contribution.
- **Zixi** — proprietary equivalent, very common in broadcast.
- **SMPTE 2110** — IP-based replacement for SDI in IP studios; uncompressed video over multicast.
- **RTMP** — legacy. Still around but not used for new high-end contribution.
- **RIST** — Reliable Internet Stream Transport, open-source alternative to Zixi/SRT.

## 2.4 Live transcoding — the most expensive component

The contribution feed comes in at one bitrate (e.g., 50–100 Mbps high-quality). It must be encoded into the **ABR ladder** — typically 5–8 renditions:

| Profile | Resolution | Bitrate | Codec | Audience |
|---------|------------|---------|-------|----------|
| 4K | 3840×2160 | 15–25 Mbps | HEVC / AV1 | Premium tier, smart TV, fiber |
| 1080p | 1920×1080 | 5–7 Mbps | H.264 / HEVC | Most home users |
| 720p | 1280×720 | 2.5–3.5 Mbps | H.264 | Mobile on Wi-Fi |
| 480p | 854×480 | 1.0–1.5 Mbps | H.264 | Mobile on 4G |
| 360p | 640×360 | 0.5–0.8 Mbps | H.264 | Constrained 4G |
| 240p | 426×240 | 0.2–0.3 Mbps | H.264 | 2G/3G / "low-data" mode |
| Audio-only | — | 32–96 kbps | AAC | Data-saver mode |

JioHotstar publishes a "low data mode" specifically for cricket — audio-only commentary with text scorecard. That's not a feature, it's a *capacity strategy*: it lets ten times more viewers join under degraded conditions.

Engineering choices on the transcoder farm:

- **GPU vs. ASIC vs. CPU** — encoding is expensive. GPU (NVIDIA NVENC) is fast but lower quality per bit; ASICs (AWS Elemental, Google Argos) are great per bit at scale; 
- CPU (libx264, x265, SVT-AV1) gives best quality but is slowest.
- **Per-segment parallelism** — encoding is parallelized across many segments simultaneously to keep up with real time.
- **Encoder redundancy** — N+1 active-active encoders per rendition; if one fails the next produces the segment.
- **GOP alignment across renditions** — IDR frames must align across all bitrate variants for clean ABR switching. Force GOP length (e.g., 2-second GOP) and align segment boundaries to GOP boundaries.

## 2.5 Packaging — manifests and chunks

After encoding, content goes through a **packager** that:

- Generates a **manifest** — HLS `.m3u8` for Apple devices, DASH `.mpd` for everything else. The manifest lists available renditions and segment URLs.
- Cuts the encoded stream into **segments** (usually 2–6 seconds) or, for LL-HLS/LL-DASH, into **chunks** (200–500 ms parts within each segment).
- Optionally **encrypts** segments under per-rendition keys, with key IDs that reference the DRM key broker.

The output (manifest + segment files) is what gets cached at the edge. Manifests change every few seconds during live — they're hot, small, must update fast. Segments are written once and cached for the rest of the event.

A subtle but critical detail: **manifests should not be cached aggressively at the edge** (only seconds), but **segments should be cached for the entire event** (hours). The CDN cache-control headers need to differentiate.

## 2.6 The multi-CDN architecture

JioHotstar deliberately fans out across multiple CDNs. Why:

- **Risk diversification** — any single CDN can have a bad afternoon. The biggest streaming outages in history involved single-CDN failures.
- **Cost negotiation leverage** — multiple vendors compete; you commit volume strategically.
- **Capacity** — even the biggest CDNs prefer not to take 100% of a single event's traffic.
- **Regional coverage** — different CDNs are stronger in different geos.

Typical vendor mix for JioHotstar (publicly observed via DNS / network probes):
- **Akamai** — historically deep coverage, premium tier.
- **Amazon CloudFront** — colocated with their AWS infra.
- **Jio's own CDN** — caches placed inside Jio's wireline / wireless network.
- Occasionally **Cloudflare, Verizon EdgeCast, Google Cloud CDN**.

The **in-house multi-CDN load balancer** is the secret sauce:

- Continuously measures per-CDN per-region performance (latency, throughput, error rate) using both client beacons and synthetic probes.
- Issues **steering decisions** to players via a small token-DNS-based or HTTP-based redirection mechanism on each manifest fetch.
- Can shift 10% of viewers from CDN A to CDN B in seconds if A starts struggling.
- Implements **graceful degradation** — if all CDNs are saturated, falls back to lower bitrates first, then to audio-only.

## 2.7 Carrier integration (the Jio advantage)

This is what really makes JioHotstar different from a generic AWS streamer:

- Jio (the carrier) has ~450M+ subscribers and a vast fiber backbone covering most of India.
- JioHotstar can place caches **inside** Jio's network — in metro PoPs, at aggregation routers, even on cell tower controllers in some configurations.
- Cricket-final traffic that stays inside Jio's network never touches the public Internet. That dramatically reduces both cost and latency.
- A viewer on Jio fiber may be 5 ms from their cached segment vs. 50 ms from a public CDN edge.

If Netflix's Open Connect put caches *into ISPs*, JioHotstar's parent company *is* the ISP for a huge fraction of its users.

## 2.8 Origin and origin shield

Behind every CDN there's an **origin** that holds the canonical copy. Two-tier setup:

- **Edge CDN** — closest to user. ~1000 PoPs total across all vendors. Most reads served here.
- **Origin shield** — mid-tier caches that sit between edges and the actual origin. Their job is to *deduplicate origin requests*. 
- Without an origin shield, 100 edge PoPs might all request the same hot segment from the origin, leading to a stampede.
- **Origin** — usually object storage (S3 + custom front-end) + manifest service. Sized for ~1% of peak edge traffic.

For live, **only the manifest and the most-recent segment** are hot at the origin. Older segments are cold (in DVR scenarios) or evicted.

## 2.9 Pre-warming and capacity planning

Auto-scaling alone cannot ramp from 1M to 70M in 5 minutes. Pre-warming is mandatory.

JioHotstar's playbook (publicly described in AWS talks and engineering blogs):

- **Predict the peak** using historical sports data — toss → peak in 25 minutes, sustained for the match, secondary peak in last 10 overs.
- **Pre-spin compute** — bring up encoder, packager, API, auth, ad-decisioning services to 1.2× expected peak well before toss.
- **Pre-warm CDN cache** — push the first segments to every PoP a few seconds before going live; force-warm by issuing internal requests for static assets, manifest, ad creatives, key responses.
- **Pre-warm autoscaler** — increase max-pod thresholds and warm-pool sizes ahead of time; their custom autoscaler reportedly brings up pods in ~30 seconds vs. the default 1–2 min.
- **DNS pre-warming** — push CDN steering tokens with the expected per-CDN split.

A staff-level interview answer should call out: *autoscaling is for the second-order load; pre-warming is for the first-order load*.

## 2.10 Load shedding under pressure

Despite pre-warming, things still break. The interview will probe: what do you give up first?

Hierarchy of degradation:

1. **Reduce ABR ladder ceiling** — cap at 720p instead of 1080p. Cuts egress ~2×.
2. **Insert ads less aggressively** — ad-decisioning is a known hot service; degrading to a default ad reduces load.
3. **Shed non-essential APIs** — recommendations, comments, login of new sessions go to maintenance mode; existing sessions keep working.
4. **Disable DVR / rewind** — only live edge available.
5. **Switch to audio-only or "lite" experience** for selected segments of users.
6. **Refuse new logins** — protect the experience of those already watching.
7. **Last resort: reduce stream resolution globally for everyone**.

The *ordering* matters more than any single mitigation. Have the runbook before the event.

## 2.11 Real-time observability

You can't fix what you can't see. JioHotstar (and similar) instrument with:

- **Player-side QoE beacons** — every client reports start time (TTFV), buffering events, ABR switches, throughput estimate, errors. 
- Sampled to 1–5% during normal load, 0.1% at peak to avoid the beacons themselves becoming the problem.
- **Server-side metrics** — per-CDN per-region origin requests, edge hit ratio, manifest age, encoder bitrate, packager queue depth.
- **Synthetic probes** — emulated player sessions from many vantage points to detect issues even when real-user telemetry is sampled away.
- **Real-time dashboards** — sub-second-refreshing displays in the NOC during the match.

The metric that gets watched most: **rebuffering rate** (% of users with buffer events / minute). When it ticks up, you have minutes to fix it before social media catches on.

## 2.12 The chat / interaction layer

A cricket match isn't just video — viewers interact via:

- **Live chat / "watch parties"** — millions of messages/minute.
- **Polls / predictions** — "who will win the next over?" with prize incentives.
- **Live commentary text feed**.
- **Score widgets** that update push-style.

These need a separate scale-out story:

- **WebSocket / SSE fanout** — millions of long-lived connections. Layered: edge proxies → regional fanout brokers → topic-sharded message bus (Kafka with key-by-room).
- **Throttling and moderation** — per-user rate limits; profanity filter; spam detection. Async pipeline; messages may be delayed by 1–2s for moderation.
- **Backpressure** — slow consumers must not block fast ones; per-connection bounded buffers.

See [10-realtime-analytics-and-chat.md](10-realtime-analytics-and-chat.md) for more.

## 2.13 The numbers on a 72.5M-viewer event

Rough order-of-magnitude back-of-envelope:

- 72.5M concurrent.
- Average bitrate (weighted by mobile-heavy India audience): say 1.5 Mbps.
- Egress: 72.5M × 1.5 Mbps = **~109 Tbps**.
- Per 6-second segment: 72.5M × (1.5 Mbps × 6 s) = **~82 GB delivered every 6 s segment**. That's 13 GB/s out of every PoP times PoP count — manageable spread across many PoPs.
- Player beacons at 1% sampling, 1 beacon/30s = ~24K beacons/sec; full sampling at 1/30s = 2.4M beacons/sec. ClickHouse-class ingest.
- Manifest requests: every player refetches every ~3–6s. 72.5M / 6s = **~12M manifest requests/sec** from the field, mostly served at edge.
- Authentication / session API: only at session start, so ~72.5M over a few minutes = a few hundred K/sec; survivable.

The two scariest numbers: 12M manifest reqs/sec and 109 Tbps egress.

## 2.14 Failure modes — the rehearsed list

| Failure | First symptom | Mitigation |
|---------|---------------|------------|
| Primary contribution feed drops | Frame freeze at encoder | Auto-cut to satellite backup (sub-second) |
| One encoder dies | Segment N missing in one rendition | Standby encoder picks up; CDN serves last cached segment with TTL extended; player ABR shifts to neighbor rendition |
| Packager queue grows | Manifest age increases | Scale packager pods; if can't, drop one rendition (e.g., kill 4K to save CPU) |
| One CDN starts erroring | Per-CDN error rate spikes in QoE beacons | Multi-CDN steering shifts traffic to others within seconds |
| ISP transit congestion | Throughput drops for users on one ASN | Steer those users to a CDN with better peering to that ASN |
| Origin saturates | Cache miss rate climbs at edges | Origin shield absorbs; if still saturated, freeze new manifests, serve last good |
| Ad-decisioning service slow | Ad break starts late | Fall back to default house ad |
| Auth/login overloaded | New users see login spinner | Existing sessions unaffected; queue new logins; surface "try again in 30s" |
| DDoS at edge | Per-PoP CPU spike on TLS | WAF + L4 rate limit at the LB tier |

## 2.15 The end-of-match cliff

Almost as hard as the ramp-up: when the match ends, 72M users disconnect within a few minutes.

- **Don't immediately tear down capacity** — the post-match highlights surge can pull 20–30M concurrent for a while.
- **Graceful scale-down** with cool-down periods.
- **Telemetry batches that were buffered locally on slow connections** will trickle in for 30–60 min. Don't be surprised when "viewers" appear to peak again at minute 200.

## 2.16 Worked design: the prompt you'll get

**Prompt**: "Design a live streaming service for a sports event expecting 50M concurrent viewers."

A staff answer hits all of these in 45 minutes:

1. **Clarify**: which event, when, geography concentration, latency SLO (broadcast parity = 5s? lower? higher OK?), DVR requirements, multi-language, devices, monetization (ads/sub).
2. **Pipeline**: contribution (dual-path SRT) → ingest → encoder farm with redundancy → packager → DRM key broker → origin → origin shield → multi-CDN → carrier embed where possible → players.
3. **ABR ladder design** with codec choices motivated by audience (e.g., HEVC on iOS, AV1 increasingly on Android, H.264 universally).
4. **CDN strategy**: multi-CDN, steering, expected hit ratios, peering.
5. **Pre-warming**: explicit pre-warm of compute, cache, DNS.
6. **Load shedding hierarchy**: which features degrade first.
7. **Observability**: QoE beacons + sampling, key SLIs (TTFV p95 < 3s, rebuffer ratio < 1%, video quality index).
8. **Chat / interaction architecture** (briefly).
9. **Failure-mode tabletop**: 3–4 from the table above.
10. **Cost**: egress dominates, single-event egress cost estimate, multi-CDN as cost-leverage.

## 2.17 Must-internalize

- Live concurrency is a *shape* problem, not just a size problem.
- Pre-warming > autoscaling for predictable peaks.
- Dual independent contribution paths (fiber + sat) is broadcast-grade table stakes.
- ABR ladder must include a "low-data" rung. It's a *capacity strategy*, not just an affordability one.
- Multi-CDN with in-house steering is mandatory at this scale.
- Carrier integration (Jio caches inside Jio's network) is the cost-and-latency advantage.
- Load shedding hierarchy is rehearsed *before* the event.
- QoE telemetry must be sampled — at peak, full beacons would crush the analytics tier.
- End-of-match cliff is a hazard equal to the ramp-up.

---

## Sources

- [Storyboard18 — JioHotstar 72.5M concurrent](https://www.storyboard18.com/amp/brand-marketing/jiohotstar-hits-500-million-maus-72-5-million-concurrency-drives-%E2%82%B936248-crore-fy26-gross-revenue-96368.htm)
- [WION — T20 World Cup 2026 final concurrent record](https://www.wionews.com/sports/72-5-million-digital-concurrent-users-during-t20-world-cup-2026-final-india-vs-new-zealand-ahmedabad-break-global-streaming-records-icc-viewership-1774276432956)
- [How JioHotstar Streams IPL 2026 to 65M Concurrent on AWS](https://www.abhs.in/blog/jiohotstar-ipl-2026-streaming-infrastructure-65-million-concurrent-viewers-aws)
- [System Design of Live Cricket Streaming for 6.5 Crore Concurrent](https://blog.codekerdos.in/how-6-5-crore-concurrent-viewers-watched-the-t20-world-cup-final-system-design-of-live-cricket-streaming/)
- [How JioHotstar Engineered 82.1 Crore Concurrent — DevOps deep dive](https://www.pritamroy.com/blog/posts/how-jiohotstar-engineered-821-crore-concurrent-streams-a-devops-deep-dive-into-t.html)
- [Scaling Hotstar.com for 25M Concurrent Viewers (AWS re:Invent 2019 PDF)](https://d1.awsstatic.com/events/reinvent/2019/Scaling_Hotstar.com_for_25_million_concurrent_viewers_CMY302.pdf)
- [How Hotstar Scaled With 10.3M Concurrent — ScaleYourApp](https://scaleyourapp.com/how-hotstar-scaled-with-10-3-million-concurrent-users-an-architectural-insight/)
- [Hotstar's Tech Infrastructure for Mass Live-Streaming — Markhub24](https://www.markhub24.com/post/hotstar-s-tech-infrastructure-for-mass-live-streaming)
- [Live Stream 25M Concurrent — Sukhad Anand (Medium)](https://sukhadanand.medium.com/live-stream-25-million-concurrent-users-3273351e0102)