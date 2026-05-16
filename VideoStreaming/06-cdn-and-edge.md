# 6 · CDN and Edge — The Bandwidth Half of the Cost Equation

Egress dominates streaming economics. A staff engineer needs to be able to reason about CDN architecture (own vs. rent vs. mix), origin shielding, cache hit math, peering, and the operational realities of running on multiple CDNs at once.

## 6.1 What a CDN actually is

A CDN is just **a hierarchy of HTTP caches geographically distributed and routed to**. The "magic" is in how the routing happens and how the caches coordinate.

Components:

1. **Edge PoPs** — physical or virtual data centers in many cities. Tens to thousands depending on the provider.
2. **Edge servers** — typically reverse proxies (Nginx, Varnish, custom) with SSD caches.
3. **Mid-tier / regional caches** — fewer locations, larger cache footprint. Serve as a "shield" for the origin.
4. **Origin** — the canonical storage. The customer's S3 / object store, or the CDN's own origin layer.
5. **Routing layer** — DNS-based steering (GeoDNS, EDNS Client Subnet), or Anycast-based (BGP).
6. **Control plane** — purges, prewarming, configuration push.

Major players: Akamai, Cloudflare, Amazon CloudFront, Google Cloud CDN, Fastly, Verizon/Edgio, BunnyCDN. Streamers often have **their own**: Netflix Open Connect, YouTube's Google CDN, Meta's CDN.

## 6.2 Why hierarchy matters

Without a mid-tier, a 1000-PoP CDN with a cold cache would hammer the origin 1000-fold on the first request for a hot asset. With a mid-tier of, say, 30 regional shields, the origin sees only ~30 requests. The mid-tier becomes the *de facto* origin for the edges.

**Origin shield** (a CloudFront term) is the practice of designating one region's cache as the mid-tier for all other edges of that region. The shield serves a deduplication function.

Three-tier YouTube architecture (publicly described):
- **Edge** — closest to user, fast, small cache.
- **Mid-tier** — regional, larger cache.
- **Origin** — Colossus / Borg-backed storage; rarely touched for hot content.

Reported edge hit ratio: **98–99%**. Mid-tier absorbs most of the remaining 1–2%.

## 6.3 Routing — DNS-based vs. Anycast

### DNS-based (GeoDNS + EDNS Client Subnet)

Player asks DNS for `cdn.example.com`. The DNS resolver — knowing the user's network location (via Anycast of the resolver, or EDNS Client Subnet) — returns the IP of a nearby edge.

Pros: simple, tunable per geography / per ASN.
Cons: DNS caching means changes take time to propagate; coarse granularity.

### Anycast

The same IP is announced via BGP from many PoPs. The network routes the user to the nearest one based on BGP best-path. Cloudflare and Google are anycast-heavy.

Pros: instant failover; no DNS TTL coupling; clean.
Cons: routing decisions are at the mercy of the global BGP table; less direct control per-flow.

Most large CDNs mix both: anycast for the edge-facing TCP, with DNS-based regional pre-steering.

## 6.4 The cache hit math

The core equation:
```
origin_load = total_request_rate × (1 - hit_ratio)
```

With 12M manifest requests/sec at 99% hit ratio: 120,000 reqs/sec to origin. Survivable.
At 95% hit ratio: 600,000 reqs/sec to origin. Painful.
At 90% hit ratio: 1.2M reqs/sec to origin. Origin dies.

Why a small percentage matters so much: every 1% drop in hit ratio doubles or triples the origin load. **Defending hit ratio is a primary CDN operational concern.**

Levers that affect hit ratio:
- **Cache key design** — strip irrelevant query strings, normalize headers.
- **Object size** — large objects cache better (less metadata overhead).
- **Manifest TTL** — short enough to be fresh, long enough to absorb refresh storms.
- **Vary headers** — `Vary: Accept-Encoding` is fine; `Vary: User-Agent` fragments the cache catastrophically.
- **Personalization** — anything per-user destroys cache. SSAI ad insertion is the classic example of this tension.

## 6.5 Server-side ad insertion (SSAI)

Live streams want personalized ad breaks. Two flavors:

- **Client-side (CSAI)** — player switches to ad URL during break, then back. Cleaner for content, but **ad-blockers** kill it.
- **Server-side (SSAI)** — ads are stitched into the manifest server-side; the player just plays through. Ad-blocker proof. But **personalized manifest → cache fragmentation**.

The SSAI compromise: stitch only the ad break personalization at the manifest level; keep the segments themselves cacheable. A common pattern is **manifest manipulation + shared segment cache** — each user gets a personalized manifest but shared cached segments for everyone watching the same content + ad-creative combo.

## 6.6 Multi-CDN architecture

For an event-scale streamer (JioHotstar, DAZN, Peacock, Disney+), one CDN isn't enough.

Reasons:
- **Capacity** — even the biggest CDNs prefer not to handle 100% of an extreme event.
- **Resilience** — single CDN failures take down a huge fraction of the Internet sometimes.
- **Cost negotiation** — competition keeps prices honest.
- **Geographic coverage** — different CDNs have different strengths.

Multi-CDN patterns:

- **DNS-level chooser** — CNAME chained to one of several CDNs based on logic. Coarse, slow to change.
- **Manifest-level chooser** — each manifest fetch returns segment URLs from the chosen CDN. Fine-grained, can shift per-user mid-stream.
- **Token-based** — player presents a token; backend rewrites manifest URLs to the chosen CDN.

The **CDN chooser** logic:
- Measure per-CDN per-region performance continuously (player beacons + synthetic probes).
- Bias by cost commitments (must hit X% on CDN A by month-end).
- Penalty for error spikes.
- Tunable for capacity reserves ("don't put more than Y on CDN B").

JioHotstar's in-house CDN load optimizer reportedly does this with sub-minute reaction time.

## 6.7 Origin shield in detail

A modern origin shield does more than cache:

- **Request collapsing** — N simultaneous requests for the same uncached object become 1 origin fetch (a.k.a. "singleflight").
- **Conditional GETs** to keep cached objects fresh.
- **Stale-while-revalidate** — serve stale content briefly while fetching a fresh copy in the background.
- **Stale-if-error** — keep serving stale on origin errors.
- **Tiered TTL** — segments get long TTL; manifests get short TTL with stale-while-revalidate.

Without these, a sudden spike (e.g., a viral video) can cause an origin stampede that takes the system down.

## 6.8 Peering and transit

CDN economics are largely about *not paying transit*:

- **Peering** — direct interconnect between two networks. Often **settlement-free** (no payment between the parties) for roughly-balanced traffic.
- **Transit** — paying an ISP to carry your traffic to the rest of the Internet.
- **Private network interconnect (PNI)** — a direct fiber between a CDN and a major ISP, for high-volume relationships.

A streamer running on Cloudflare effectively gets Cloudflare's peering for free (it's in the price). A streamer running its own CDN must negotiate peering directly — that's a significant role for senior network engineers.

**Carrier embedding** (Netflix Open Connect, Hotstar in Jio): you put a cache *inside* the ISP's network. From the ISP's perspective, you're using their network's first-hop bandwidth (free for them, you give them appliances), and from the streamer's perspective, you save the cost of crossing the boundary into the ISP.

## 6.9 P2P CDN (peer-assisted delivery)

Some streamers (Hotstar has trialed this; Peer5 / Streamroot / others as third-parties) use peer-to-peer mesh for the long-tail offload. Players that have already downloaded a segment serve it to nearby players (WebRTC-based).

Tradeoffs:
- Saves money on the CDN bill (peers do free distribution).
- Adds complexity (signaling, NAT traversal, fallback to CDN).
- Quality must not degrade — must fall back fast if a peer is slow.
- Privacy concerns — peers can see what others are watching (mitigated by encrypted segments).
- Mostly useful for live where many peers are watching the same segments simultaneously.

## 6.10 Edge compute (compute at the PoP)

Modern CDNs let you run code at the edge:
- **Cloudflare Workers, AWS Lambda@Edge, Fastly Compute@Edge, Akamai EdgeWorkers.**
- Use cases for streaming: token validation, A/B test routing, manifest manipulation, ad-stitching, geo-blocking enforcement, per-user CDN steering, dynamic image/thumbnail resizing.

Per-request latency at the edge is sub-10 ms when done right. Don't move heavy logic here; do move the small enforcement and personalization logic.

## 6.11 Cost — the bill that decides architecture

CDN pricing models (rough; volumes negotiate down by 80–95%):

- **CloudFront**: starts at $0.085/GB egress, falls to ~$0.02/GB at PB-scale committed volume.
- **Cloudflare**: bundled-price model; effectively ~$0.005-$0.01/GB at enterprise volume.
- **Akamai**: traditional enterprise pricing; usually $0.01–0.02/GB at scale.
- **Self-hosted CDN** (cabinet space + transit): can be $0.001-$0.005/GB at very large scale but requires significant CapEx.

At 100 Tbps × 3.5 hours = 157 PB on a single event:
- At $0.01/GB: $1.57M for that event.
- At $0.005/GB: $785K.
- At $0.001/GB (carrier-embedded): $157K.

That's the math behind multi-CDN, carrier embedding, and own-CDN investment. Single events justify infrastructure that pays back in months.

## 6.12 Cache warming and prefetch

For predictable events:
- **Cold cache → first viewer pays the cost**. Bad UX, bad origin load.
- **Pre-warm** by issuing synthetic requests from a server-side warmer that touches every PoP for the hot objects (first segment, manifest, ad creatives, error fallbacks).
- Most CDNs offer a **prefetch API**; some require third-party tools.

For live, you can pre-push the first 60 seconds of segments before going live so when a user joins, the cache already has them.

For VOD release nights (new Netflix season drop), pre-push titles to all OCAs the night before.

## 6.13 CDN observability

Per-PoP, per-region, per-CDN metrics that the NOC watches:
- **Cache hit ratio** — primary KPI.
- **Origin egress** — proxy for hit ratio issues.
- **Per-PoP request rate** — flags load distribution issues.
- **TLS handshake time** — degraded cert / cipher renegotiation.
- **TCP retransmits / RTT** — peering issues.
- **5xx rate** — origin sickness or cache config errors.
- **Cost per GB by route** — for ongoing optimization.

## 6.14 Common failure modes

- **Cache poisoning** — a malformed request returns a 500 that gets cached. All subsequent users see 500. Mitigation: never cache 5xx; very short TTL on 4xx.
- **TLS cert expiry** — certificate at the edge expires, all clients get errors. Mitigation: automate renewal; alert N days before.
- **BGP withdrawal** — a single PoP withdraws routes; traffic shifts elsewhere; nearby PoPs overload. Mitigation: load-balanced PoP capacity; alerts on rapid traffic shifts.
- **DNS issue at the chooser** — chooser DNS goes down, players fall back to default CDN, that CDN overloads. Mitigation: chooser must be redundant across DNS providers.
- **Origin shield fails** — edges fan out directly to origin; origin gets crushed. Mitigation: shield redundancy; rate limit per-origin from edge layer.

## 6.15 Worked design — multi-CDN chooser for a 50M-viewer event

**Prompt**: design the CDN chooser logic.

**Architecture**:
- **Player SDK** sends a beacon every 30s with current CDN, throughput, error count.
- **Server-side beacon aggregator** (Kafka → ClickHouse or similar) computes per-CDN per-region SLI every 10s.
- **Chooser service** (in-house) computes the per-region desired CDN distribution (e.g., 60% Akamai, 30% Cloudfront, 10% Jio) using:
  - SLI scores
  - Cost-commitment targets
  - Capacity reserves
- **Manifest service** consults the chooser per manifest request, returns the appropriate CDN's URLs.
- **Per-user stickiness** — once a user is on CDN A, keep them there unless their SLI degrades. Avoid flapping.
- **Failure mode**: chooser dies → fallback to a static per-region default in the player config.

**Numbers**: ~10M beacons/sec at peak; chooser decisions cached for 10s, served from memory. Very tractable.

## 6.16 Must-internalize

- 99% hit ratio is mandatory; 95% kills origin.
- Three-tier (edge / mid-tier / origin) with shield is the standard pattern.
- Anycast vs. DNS-based steering — both, often layered.
- Multi-CDN with in-house chooser is the streaming-scale norm.
- Carrier embedding (Open Connect / Jio caches in Jio) is the big cost lever.
- Pre-warming before the event > autoscaling during.
- SSAI vs. cache: design to keep segments shared even when manifests are per-user.
- The cost math justifies own/embedded CDN at very large scale.

---

## Sources

- [Cloudflare network](https://www.cloudflare.com/network/)
- [Netflix Open Connect](https://openconnect.netflix.com/)
- [CloudFront origin shield](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/origin-shield.html)
- [YouTube CDN deep dive — Servify](https://medium.com/@servifyspheresolutions/how-youtube-streams-billions-of-videos-every-day-inside-googles-cdn-9cf8f5b84bfb)
- [Streaming CDN protocol comparison — BlazingCDN](https://blog.blazingcdn.com/en-us/streaming-cdn-protocol-comparison-hls-vs-mpeg-dash)
- [P2P CDN — Streamroot historical writeups](https://streamroot.io/)
