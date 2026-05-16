# 4 · Streaming Protocols — HLS, DASH, CMAF, LL-HLS, LL-DASH, WebRTC, SRT, RTMP, MoQ

The protocol layer is one of the densest interview targets in this domain. A staff engineer should be able to describe each protocol's wire format, latency budget, when to pick it, and how it fails. This file covers the lot.

## 4.1 The two-axis mental model

There are two orthogonal questions:

1. **What protocol carries video from camera → CDN ingest?** (the *contribution* leg)
2. **What protocol carries video from CDN → player?** (the *delivery* leg)

Different protocols dominate each leg.

| Leg | Common protocols |
|-----|------------------|
| Contribution | SRT, RTMP, Zixi, SMPTE 2110, RIST, WebRTC (WHIP) |
| Delivery | HLS, MPEG-DASH (often CMAF-packaged), WebRTC (WHEP), HESP, future: MoQ |

## 4.2 HTTP-based adaptive streaming — HLS and DASH at a glance

Both follow the same template: **a manifest file** points at **segments of media** (a few seconds each), all over plain HTTP/HTTPS. The player downloads segments, parses them, and plays. Caching is just regular HTTP caching, which is why CDNs handle it trivially.

| | HLS | MPEG-DASH |
|---|-----|-----------|
| Manifest | `.m3u8` (text playlist) | `.mpd` (XML) |
| Container | Originally MPEG-TS; modern: fMP4 / CMAF | fMP4 / CMAF |
| Vendor | Apple (RFC 8216) | ISO/IEC 23009-1 |
| Codec independence | Tight | Loose |
| Wide device support | All Apple devices natively; everywhere via player | Everywhere *except* iOS native (iOS supports DASH only via custom players, and not in Safari) |
| DRM | FairPlay native; Widevine/PlayReady via CMAF | Widevine/PlayReady native; FairPlay via CMAF |
| Default latency | 15–30 s | 10–20 s |

The 2026 norm: **encode once with CMAF, package twice (HLS .m3u8 + DASH .mpd) referencing the same fMP4 chunks**. One set of bytes on disk; two manifests over the wire.

## 4.3 HLS manifest in 1 minute

Top-level manifest (a "master playlist") lists the available renditions:

```
#EXTM3U
#EXT-X-VERSION:7

#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080,CODECS="avc1.640028,mp4a.40.2"
1080p/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=2500000,RESOLUTION=1280x720,CODECS="avc1.4d401f,mp4a.40.2"
720p/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=1200000,RESOLUTION=854x480,CODECS="avc1.4d401e,mp4a.40.2"
480p/index.m3u8
```

Per-rendition manifest (a "media playlist") lists segments:

```
#EXTM3U
#EXT-X-VERSION:7
#EXT-X-TARGETDURATION:6
#EXT-X-MEDIA-SEQUENCE:1234
#EXTINF:6.0,
segment1234.m4s
#EXTINF:6.0,
segment1235.m4s
#EXTINF:6.0,
segment1236.m4s
```

For **live**, the media playlist is repeatedly re-fetched (every segment duration / 3 is the recommendation). For **VOD**, the playlist is static; `#EXT-X-ENDLIST` marks the end.

## 4.4 DASH manifest in 1 minute

DASH's `.mpd` is XML. The same content:

```xml
<MPD type="dynamic" minimumUpdatePeriod="PT2S" availabilityStartTime="2026-05-15T18:00:00Z" ...>
  <Period>
    <AdaptationSet contentType="video" mimeType="video/mp4">
      <Representation id="1080p" bandwidth="5000000" width="1920" height="1080" codecs="avc1.640028">
        <SegmentTemplate timescale="1000" duration="6000"
                         initialization="$RepresentationID$/init.mp4"
                         media="$RepresentationID$/seg-$Number$.m4s"
                         startNumber="1234" />
      </Representation>
      <Representation id="720p" bandwidth="2500000" ...>
        ...
      </Representation>
    </AdaptationSet>
    <AdaptationSet contentType="audio" ...>
      ...
    </AdaptationSet>
  </Period>
</MPD>
```

Two ways to address segments:
- **SegmentTemplate** (above) — clean URLs computed from number/time. CDN-friendly.
- **SegmentList** / **SegmentTimeline** — explicit list of segments with durations. Used when segment durations vary (e.g., aligned to ad breaks).

DASH also supports **byte-range requests** into a single file (a "single-file" representation) which can reduce CDN object count and slightly improve caching.

## 4.5 CMAF — one container to rule both

Common Media Application Format (ISO/IEC 23000-19) is a fMP4-based container that both HLS (since iOS 10) and DASH support. Why it matters at scale:

- **One encoding** → two manifests. Halves storage and origin egress.
- **Chunked encoding** support: a CMAF segment can be made of small **chunks** (parts) that can be transferred and played before the segment is fully closed. This is the basis of LL-HLS and LL-DASH.
- **Common encryption** (CENC) means one set of encrypted bytes works for Widevine + PlayReady + FairPlay (with appropriate key derivation).

## 4.6 Low-latency HLS (LL-HLS)

Standard HLS has 15–30 s latency because the player can only request segments that are *already finished*. LL-HLS shrinks this by:

- **Partial segments** (parts) — segments are broken into ~200ms parts; each part is requestable before the whole segment is done.
- **Blocking playlist reload** — `_HLS_msn`/`_HLS_part` query params let the player long-poll for the next part.
- **Preload hints** — `EXT-X-PRELOAD-HINT` tells the player "the next part will be at this URL".
- **Rendition reports** — `EXT-X-RENDITION-REPORT` lets the player track other renditions without separately fetching their manifests, smoothing ABR switching.

Target latency: 2–5 s glass-to-glass. Works over standard HTTPS / CDN infrastructure (with HTTP/2 push optionally) — that's the appeal vs. WebRTC.

## 4.7 Low-latency DASH (LL-DASH / CMAF-LL)

Equivalent for DASH. Uses **chunked transfer encoding** on HTTP/1.1: the CDN starts streaming the segment as the encoder produces chunks of it. The player parses chunks as they arrive.

- **Availability time offset** — the manifest declares "this segment becomes available N seconds before its end time" to coordinate the chunked transfer.
- Player uses **CMAF chunks** (typically 200–500 ms) for incremental decoding.
- Target latency: 2–4 s glass-to-glass.

LL-DASH and LL-HLS are now within ~1–2 s of each other and both deliver good user experiences. Pick based on the device mix.

## 4.8 WebRTC — sub-second when you need real-time

WebRTC was designed for video calls; it carries video over **UDP** with SRTP + DTLS, with explicit congestion control and forward error correction. Latency: 200–500 ms typical.

Pros:
- Genuinely real-time. Suitable for auctions, sports betting, watch-party-with-voice, interactive game shows.
- Built into every modern browser.
- NAT traversal via STUN/TURN.

Cons:
- Doesn't scale via standard HTTP CDN — you need a **WebRTC SFU/MCU fanout tier**.
- Per-viewer compute is higher than HTTP delivery.
- No native ad insertion ecosystem.
- DRM story is weaker (no Widevine/PlayReady/FairPlay flow defined).

Mainstream services use HLS/DASH for the long-tail and **switch to WebRTC for the interactive moment** (the live auction, the betting window) if at all.

### WHIP and WHEP

**WHIP** (WebRTC-HTTP Ingestion Protocol, RFC 9725) standardizes the *publish* side — a simple HTTP POST exchanges SDP offer/answer to start an ingest. Replaces ad-hoc signaling for ingest.

**WHEP** (WebRTC-HTTP Egress Protocol) is the equivalent for the *subscribe* side. Together they make WebRTC a more straightforward protocol to integrate.

## 4.9 SRT — the contribution king

Secure Reliable Transport. UDP-based, with selective retransmission for packet loss, AES encryption, and ARQ-style error correction. Latency: ~1–2 s typical.

- Originally developed by Haivision; now open-source under the SRT Alliance.
- Designed specifically for unreliable IP transport — the kind of network you have between a sports venue and an ingest PoP.
- Most modern broadcast equipment supports SRT.
- Variant: **RIST** (Reliable Internet Stream Transport) is the open standard for the same job, with broader vendor neutrality.

If you're asked "how does the video get from the stadium to your ingest", "SRT or Zixi over diverse IP paths, with broadcast-grade satellite backup" is the modern answer.

## 4.10 RTMP — the legacy ingest

Real-Time Messaging Protocol. TCP-based, originally Adobe Flash's protocol. Latency: 3–5 s.

- Still the most common ingest protocol because every encoder supports it (OBS, hardware encoders, mobile SDKs).
- Plain TCP — no encryption (RTMPS adds TLS), no NAT-traversal magic.
- YouTube, Twitch, Facebook all still accept RTMP from creators.
- Being slowly displaced by **WebRTC (WHIP)** for contribution from browsers, and **SRT** for higher-end production.

## 4.11 HESP and other less-common protocols

- **HESP** (High Efficiency Streaming Protocol) — by THEO Technologies, HTTP-based, sub-2-s latency, with a clever initialization stream that lets a new viewer join at any moment without waiting for an IDR.
- **Apple Low-Latency HLS** — what we called LL-HLS above.
- **CMAF-LL** — what we called LL-DASH above.

## 4.12 MoQ (Media over QUIC) — the future

**MoQ Transport** is an IETF in-progress standard (working group active in 2024–2026). The idea:

- Use **QUIC** as transport (UDP-based, multiplexed streams, encrypted, congestion-controlled, head-of-line-blocking-free between streams).
- Define a publish/subscribe model for media objects on top of QUIC.
- Designed to span the *contribution AND delivery* legs with one protocol.
- Target: sub-second latency at HTTP-CDN-class scale.

Several big players (Meta, Cisco, Akamai, Cloudflare) are pushing this. If it stabilizes, expect adoption to start with major streamers in 2027–2028. Worth name-checking in an interview as the direction-of-travel.

## 4.13 Latency budget table — pick the right tool

| Use case | Latency target | Protocol pick |
|----------|----------------|---------------|
| Broadcast-parity sports | 5–10 s | Standard HLS / DASH |
| Engaged sports (you want to beat the TV) | 2–5 s | LL-HLS or LL-DASH (CMAF) |
| Interactive (auction, betting, live commentary that takes Q&A) | <1 s | WebRTC, or HESP, or MoQ when ready |
| Video conferencing | <300 ms | WebRTC native |
| Telehealth / remote surgery | <100 ms one-way | Custom; usually not over public Internet |

## 4.14 The HTTP details that matter at CDN scale

A subtle but interview-worthy area:

- **Manifest cache TTL** must be short (a few seconds for live) but never zero — bursts of clients all refreshing manifest would crush origin if not cached briefly.
- **Segment cache TTL** can be very long. Segment URLs are content-addressed; once published, never change.
- **Cache-key normalization** — strip query strings (or whitelist specific ones), unify Accept-Encoding handling, vary by codec hint if needed.
- **Range requests** — DASH single-file profiles use byte-range; CDN must honor and forward correctly.
- **HTTP/2 or HTTP/3** for delivery — HTTP/2 multiplexing reduces TCP connection overhead; HTTP/3 (QUIC) reduces handshake time and improves loss recovery on mobile.

## 4.15 Common pitfalls

- **GOP misalignment across renditions** → unsightly artifacts on ABR switch.
- **Segment-duration mismatch with chunk size** → playback hiccups; choose 2-6 s segments with chunks ≤ segment/5 for LL-HLS.
- **Manifest cached too long** → players play stale data, late-joiners join at the wrong position.
- **Manifest cached too short** → origin pounded on every player's playlist refresh.
- **Missing CORS headers** for browser players → CSP / fetch failures, very common cause of "video doesn't work in browser but works on mobile".
- **Inconsistent encryption keys** across renditions or segments → silent decoding failures, often only after some users have already started watching.

## 4.16 Worked design — "design the protocol stack for a new sports streamer"

**Prompt**: "You're starting a new sports streaming app. Pick the protocol stack."

- **Contribution**: SRT (primary) over dedicated fiber + satellite SRT backup. WHIP/WebRTC for creator-style live shoulder content from phones.
- **Encoding**: CMAF fMP4 chunked, 2-s segments with 200 ms parts.
- **Delivery**: LL-HLS *and* LL-DASH from the same CMAF source, picked by the player at runtime.
- **DRM**: CMAF + CENC, Widevine + FairPlay + PlayReady keys via a single license-broker endpoint.
- **Transport**: HTTP/2 default, HTTP/3 (QUIC) on supported clients.
- **Interactive layer (chat, betting window)**: WebRTC over UDP/TURN.
- **Target glass-to-glass latency for normal viewing**: 3 s. For interactive features overlay: 500 ms.

## 4.17 Must-internalize

- HLS + DASH are HTTP-based, CDN-friendly, the default delivery protocols.
- CMAF lets one encoded byte serve both HLS and DASH.
- LL-HLS and LL-DASH bring HTTP-based delivery to ~3 s.
- WebRTC is for sub-second but needs SFU/MCU fanout, not standard CDN.
- SRT is the modern contribution protocol; RTMP is legacy but everywhere.
- MoQ is the future; know it's coming.
- Manifest TTL short, segment TTL long; CORS, range requests, cache-key normalization are CDN gotchas.

---

## Sources

- [HLS specification (RFC 8216)](https://datatracker.ietf.org/doc/html/rfc8216)
- [Apple Low-Latency HLS](https://developer.apple.com/documentation/http_live_streaming/protocol_extension_for_low-latency_hls_preliminary_specification)
- [MPEG-DASH overview (ISO/IEC 23009-1)](https://www.iso.org/standard/79329.html)
- [DASH Industry Forum](https://dashif.org/)
- [LL-HLS vs LL-DASH vs WebRTC — Cloudinary](https://cloudinary.com/guides/live-streaming-video/low-latency-hls-ll-hls-cmaf-and-webrtc-which-is-best)
- [Performance of LL-DASH/CMAF and LL-HLS (at-scale)](https://atscaleconference.com/videos/performance-of-low-latency-dash-cmaf-and-low-latency-hls-systems/)
- [WebRTC at scale (Cloudflare blog)](https://blog.cloudflare.com/webrtc-whip-whep-cloudflare-stream/)
- [SRT Alliance](https://www.srtalliance.org/)
- [Media over QUIC working group](https://datatracker.ietf.org/wg/moq/about/)
