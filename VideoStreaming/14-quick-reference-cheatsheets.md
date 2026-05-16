# 14 · Quick Reference Cheatsheets

The night-before review. One page for each topic family.

## 14.1 Codec quick reference

| Codec | Standard year | ~Quality vs. H.264 | Encode CPU | Decode HW | Royalty |
|-------|---------------|---------------------|------------|-----------|---------|
| H.264 / AVC | 2003 | baseline | 1× | universal | royalty pool |
| H.265 / HEVC | 2013 | ~30% better | 5–10× | most modern | complex |
| VP9 | 2013 | ~30% better | 5–10× | most modern | royalty-free |
| AV1 | 2018 | ~30% better than HEVC | 10–50× | growing | royalty-free |
| VVC / H.266 | 2020 | ~50% better than HEVC | 10× HEVC | limited | royalty pool |

## 14.2 Streaming protocol latency table

| Protocol | Glass-to-glass latency | Use case |
|----------|------------------------|----------|
| RTMP | 3–5 s | Legacy ingest |
| SRT | 1–2 s | Modern contribution |
| Standard HLS | 15–30 s | Broadcast-parity VOD/live |
| Standard DASH | 10–20 s | Broadcast-parity VOD/live |
| LL-HLS | 2–5 s | Engaged sports / live |
| LL-DASH / CMAF-LL | 2–4 s | Engaged sports / live |
| HESP | <2 s | Premium low-latency |
| WebRTC | 200–500 ms | Real-time interactive |
| MoQ (future) | <1 s | Future of HTTP-class real-time |

## 14.3 Bitrate ladder rough rule

| Resolution | H.264 | HEVC | AV1 |
|------------|-------|------|-----|
| 4K (2160p) | n/a | 12–15 Mbps | 8–10 Mbps |
| 1440p | n/a | 8 Mbps | 5 Mbps |
| 1080p | 5–6 Mbps | 3–4 Mbps | 2–3 Mbps |
| 720p | 2.5–3 Mbps | 1.5–2 Mbps | 1–1.5 Mbps |
| 480p | 1–1.5 Mbps | 0.7–1 Mbps | 0.5–0.7 Mbps |
| 360p | 0.5–0.8 Mbps | 0.3–0.5 Mbps | 0.2–0.4 Mbps |
| 240p | 0.2–0.3 Mbps | 0.15 Mbps | 0.1 Mbps |
| Audio only (AAC) | 64–128 kbps | — | — |

## 14.4 DRM matrix

| DRM | Devices | Encryption mode |
|-----|---------|-----------------|
| Widevine | Android, Chrome, Chromecast, smart TVs | cenc, cbcs |
| FairPlay | iOS, macOS, tvOS, Safari | cbcs only |
| PlayReady | Windows, Xbox, smart TVs | cenc, cbcs |

**Common encryption**: package once with CMAF + cbcs; serve all three DRMs.

## 14.5 Ports / protocols quick list

| Service | Port |
|---------|------|
| HTTP / HLS / DASH / DoH | 80 / 443 TCP |
| RTMP | 1935 TCP |
| RTMPS | 443 TCP |
| SRT | UDP, configurable (default ~9999) |
| WebRTC media | UDP, dynamic via STUN/TURN |
| WebSocket | 80 / 443 TCP (over HTTP upgrade) |
| QUIC / HTTP/3 | 443 UDP |
| RIST | UDP, configurable |

## 14.6 RFCs to namedrop

| RFC / spec | Topic |
|-----------|-------|
| RFC 8216 | HLS |
| RFC 9725 | WHIP (WebRTC HTTP ingestion) |
| RFC 7230–7232 | HTTP/1.1 |
| RFC 9114 | HTTP/3 |
| RFC 9000 | QUIC |
| ISO/IEC 23009-1 | MPEG-DASH |
| ISO/IEC 23000-19 | CMAF |
| ISO/IEC 23001-7 | Common Encryption (CENC) |
| CTA-5004 | Common Media Client Data (CMCD) |
| SCTE-35 | Ad insertion / splice info |
| RFC 5246 / 8446 | TLS 1.2 / 1.3 |

## 14.7 QoE metrics

| Metric | Target |
|--------|--------|
| TTFV (time to first video) | < 100 ms p50, < 3 s p95 |
| Rebuffer ratio | < 1% |
| Fatal error rate | < 0.05% |
| Average bitrate / VMAF | maximize given the above |
| Live latency (LL-HLS) | 2–5 s |

## 14.8 Cache hit math

| Hit ratio | Origin load multiplier |
|-----------|------------------------|
| 99% | 1× |
| 95% | 5× |
| 90% | 10× |
| 80% | 20× |
| 50% | 50× |

Every 1% drop ~halves the headroom before origin saturates.

## 14.9 JioHotstar record at a glance

- **72.5M peak concurrent viewers** during India vs New Zealand T20 World Cup 2026 final.
- 65M previous record during India vs England semi-final of the same tournament.
- 500M+ MAUs; ₹36,248 cr gross revenue FY26.
- ~800 microservices on AWS EKS; 8K CPU cores; 16 TB RAM; 32 Gbps peak per cluster (figures from public summaries).
- Custom autoscaler bringing pods up in ~30 s.
- Multi-CDN with in-house chooser; ~90% CDN cache offload.
- Dual contribution paths: fiber primary, satellite backup.
- ABR ladder including a "low-data" audio-only mode.
- Carrier integration with Jio fiber + 4G/5G.

## 14.10 YouTube at a glance

- 2.5B users.
- 500+ hours uploaded per minute.
- ~3,000 edge locations.
- 98–99% edge cache hit ratio.
- Sub-100ms TTFV p50.
- Custom encoder ASIC (Argos VCU).
- DASH on most platforms, HLS on Apple, CMAF underneath.
- Two-tower recommendation: candidate gen + ranking, optimized for expected watch time.

## 14.11 Big-event numbers to anchor

| Event metric | Value |
|--------------|-------|
| Avg bitrate (mobile-heavy audience) | 1.5 Mbps |
| Avg bitrate (Western 1080p audience) | 3.5 Mbps |
| Egress at 50M × 1.5 Mbps | ~75 Tbps |
| Egress at 72.5M × 1.5 Mbps | ~109 Tbps |
| Egress at 100M × 5 Mbps | ~500 Tbps (theoretical) |
| Cricket match duration | ~3.5 hours |
| 100 Tbps × 3.5 h | ~157 PB |

## 14.12 The 30-second pre-interview reminder

- **Live ≠ VOD** — different problems, different architectures.
- **Egress dominates cost** — codec choice, per-title encoding, multi-CDN, carrier embedding.
- **Pre-warm > autoscale** for predictable peaks.
- **CMAF + cbcs + DRM trio** = one byte serves all platforms.
- **HLS for Apple, DASH everywhere else**, both manifests over the same CMAF bytes.
- **ABR phases**: startup low → steady balanced → rebuffer recovery low.
- **Two-tower recommender** at scale: candidate gen → ranking → re-rank.
- **QoE telemetry** is the truth — player beacons + ClickHouse + dashboards.
- **Chat at scale**: WebSocket fanout + room sharding + lossy is fine.
- **Failure modes rehearsed** — load-shed hierarchy named before the event.
- **Numbers**: 72.5M (JH record), 2.5B (YT users), 99% (CDN hit), 100 Tbps (event egress), 3 s (LL-HLS), <100 ms (YT TTFV).

Breathe.
