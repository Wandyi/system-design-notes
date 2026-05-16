# 5 · Encoding and Codecs

Encoding is where most of the egress cost is decided. A 10–15% bitrate reduction at the same perceived quality saves an eight-figure CDN bill at JioHotstar / YouTube scale. A staff engineer should be able to discuss codec families, the rate-control mechanics, and the operational tradeoffs of running an encoder farm.

## 5.1 The codec lineup

| Codec | Standard year | Quality vs. H.264 (same bitrate) | Encode CPU vs. H.264 | Decode in HW | License cost |
|-------|---------------|----------------------------------|----------------------|--------------|--------------|
| H.264 / AVC | 2003 | baseline | 1× | universal | royalty pool (MPEG LA) |
| H.265 / HEVC | 2013 | ~30% better | 5–10× | most modern devices | complex (multiple pools) |
| VP9 | 2013 | ~30% better | 5–10× | most modern devices | royalty-free |
| AV1 | 2018 | ~30% better than HEVC | 10–50× (improving) | growing fast (smart TVs, recent mobile) | royalty-free (AOMedia) |
| VVC / H.266 | 2020 | ~50% better than HEVC | 10× HEVC | very limited | royalty pools forming |
| LCEVC | 2020 | enhancement over any codec | small overhead | software | royalty pool |

The 2026 deployment reality:
- **H.264** is the universal fallback, always present in the ladder.
- **HEVC** dominates iOS and most smart TVs.
- **AV1** is shipping in Chrome, Firefox, Edge, Safari (recent), Android (most), shipping in TVs from ~2022 forward, in YouTube's encoder fleet at scale.
- **VP9** is mostly legacy now (YouTube used it before AV1 was ready).
- **VVC** and **LCEVC** are emerging but not yet at mainstream-streaming scale.

## 5.2 Why HEVC/AV1 are 30% better

Coding gain comes from better tools:
- **Larger block sizes** (CTU up to 64×64 in HEVC, 128×128 in AV1) vs. fixed 16×16 macroblocks.
- **More intra-prediction modes** (35 in HEVC vs. 9 in H.264).
- **Better motion compensation** — quarter-pel motion, larger reference frame buffers, advanced motion vector prediction.
- **Improved entropy coding** (CABAC variants, arithmetic coding in AV1).
- **In-loop filters** that reduce blocking artifacts.
- **Tile-based parallelism** (HEVC, AV1) — encode/decode regions in parallel for performance.

The price: encoder complexity goes way up. AV1 encoders are still maturing; SVT-AV1 (open source) has gotten dramatically faster since 2020.

## 5.3 The encoder farm — what staff engineers care about

For a streaming service, the question isn't "what's the best encoder" — it's "what's the right encoder for each rendition × content type × platform target × cost budget".

Operational considerations:

- **Throughput per dollar** — frames-per-second-per-watt or per-rupee on each hardware option.
  - **GPU (NVENC, QuickSync)** — fast, low quality per bit.
  - **CPU (libx264, x265, SVT-AV1)** — best quality per bit, slowest.
  - **ASIC (AWS Elemental, Google Argos VCU, Meta Scribe)** — best perf-per-watt at scale.
- **Per-segment parallelism** — for VOD, encode chunks in parallel; for live, run multiple encoder instances per rendition for redundancy.
- **Two-pass vs. one-pass** — two-pass gives better quality (encoder analyzes the whole video first, then encodes with informed rate control). VOD-only luxury; live is one-pass.
- **Rate control modes**:
  - **CBR (constant bitrate)** — buffer-predictable, easy CDN math.
  - **VBR (variable bitrate)** — better quality for the same average bitrate.
  - **CRF (constant rate factor)** — quality-targeted, variable bitrate. Best for VOD where size isn't fixed.
  - **Capped CRF** — VOD compromise. Target quality, but cap max bitrate.
- **Hardware availability and queueing** — ASICs are scarce; CPU pools are flexible; mix to absorb peaks.

## 5.4 Per-title encoding (the Netflix insight)

A fixed ABR ladder over-provisions simple content and under-provisions complex content. Per-title encoding measures the content:

1. **Encode the source at many points** along the (resolution, bitrate) plane.
2. **Measure quality** at each point — VMAF score is the de facto metric.
3. **Plot the convex hull** of (bitrate, VMAF) — the Pareto frontier.
4. **Pick rungs** along the hull at perceptually meaningful intervals (e.g., 6 VMAF apart).

The result: a still-camera talk show might have a ladder like (240p/180k, 360p/350k, 480p/600k, 720p/1.0M, 1080p/2.0M) — much lower than the action-movie ladder.

Bandwidth savings: 20–40% at the same perceived quality, reported by Netflix. Encoder cost goes up (you do the multi-point pre-encoding), but egress savings dwarf it.

## 5.5 Shot-based / dynamic optimizer encoding

The next refinement: split the video into **shots** (changes in scene), encode each shot with its own parameters. The "action sequence" shots get higher bitrates; the "talking head" shots get lower. Quality-per-bit goes up another 10–20%.

Netflix's "Dynamic Optimizer" generalizes this. Used widely for premium AV1 deployments.

## 5.6 Live encoding constraints

Live can't do the per-title analysis pass (the content hasn't happened yet). Workarounds:

- **Look-ahead** — buffer N seconds before encoding, do a cheaper analysis on the look-ahead.
- **Content-aware bitrate** — adjust on the fly based on detected scene complexity. AWS Elemental supports this.
- **Per-stream tuning** — calibrate the ladder for the *type* of content (sports vs. talk show vs. concert) before the event starts.

GOP structure for live:
- **Closed GOPs** with IDR every 2 seconds. Aligns to segment boundary.
- **No B-frames at the IDR boundary** to avoid timing issues at ABR switching.
- **Reference frames** kept short for low-latency variants.

## 5.7 Audio encoding

Less interview-glamorous but real:

- **AAC-LC** — universal, 96–256 kbps typical.
- **HE-AAC** / **AAC-HE v2** — better at low bitrates; used for 32–64 kbps audio-only.
- **AC-3 / E-AC-3 (Dolby Digital, Dolby Digital Plus)** — surround sound for premium VOD; Netflix/Disney+ uses E-AC-3.
- **Opus** — best free codec, used heavily in WebRTC; HLS supports it since iOS 13.
- **xHE-AAC** — Apple's modern variant; used in Apple Music; growing in HLS.

Multi-language live: each language is a separate audio track inside the same manifest. Players let users switch on the fly.

## 5.8 Subtitles and captions

- **WebVTT** — modern web standard.
- **TTML / IMSC1** — used by broadcast and premium studios; supports rich styling, kerning.
- **SRT** — legacy plain text.
- **CEA-608 / CEA-708** — embedded in MPEG-TS streams; the original broadcast captions.

Captions can be **in-band** (within the video container) or **out-of-band** (sidecar files referenced from the manifest).

## 5.9 Forensic / session-based watermarking

For premium content, two flavors of watermarking:

- **Pre-baked**: a static visible/invisible watermark applied at encode. Cheap, but the same for everyone — you can't trace a leak to a specific user.
- **A/B variant watermarking**: encode two versions of each segment (A and B) with imperceptible per-segment differences. At delivery, each user gets a unique sequence of A/B variants. If the stream leaks, you can recover the user's bit-pattern from the leak.

A/B variants double encoding cost but enable per-user traceability. Major sports rights deals require it.

## 5.10 Encoding cost math (back-of-envelope)

A rough rule for an AV1 encoder on commodity hardware:
- ~5–20 fps real-time encode for 1080p.
- A movie is ~24 fps.
- A single CPU core can encode a 1.5-hour movie in 1.5–6 hours at high quality.
- For 6 renditions × 100K titles, you need millions of CPU-hours over time. Per-segment parallelism reduces wall-clock but not total CPU.

This is why YouTube built Argos VCU — at fleet scale, custom ASIC pays back in months.

## 5.11 Encoding for HDR and 4K

- **HDR formats**: HDR10 (static metadata), HDR10+ (dynamic), Dolby Vision (12-bit, dynamic, license required), HLG (broadcast).
- **Bitrate**: 4K HDR can run 15–25 Mbps depending on codec; Dolby Vision adds a small overhead for the enhancement layer.
- **Workflow**: HDR encode chain must preserve color metadata end-to-end; player + display must support the format. Easy to get wrong.

JioHotstar offered 4K HDR cricket on the T20 final for select smart TVs and high-end mobiles.

## 5.12 Worked design — choose the codec mix

**Prompt**: "You're spending $50M/year on egress. Where do you invest the codec budget?"

A staff answer:

1. **Audit** current ladder: what % of egress is in each codec / rung.
2. **AV1 for top-3 rungs (4K/1080p/720p)** on supported devices — 30% saving on the highest-egress rungs.
3. **HEVC for iOS** (default; cheap to maintain — already encoded).
4. **H.264 fallback** unchanged.
5. **Per-title encoding** rollout for the top-10K titles by view count — 20–40% saving on the bulk of egress.
6. **Look at audio** — switch from AAC 128k to xHE-AAC 96k where supported.
7. **Watermarking** for studio-mandated titles — accept the cost.
8. Estimate: a realistic 15–25% net egress reduction in year one. At $50M, that's $7–12M saved against, say, $3M extra encoding cost. **Net savings ~$5–10M**.

## 5.13 Must-internalize

- H.264 universal fallback; HEVC for iOS; AV1 for everywhere it's supported.
- VP9 fading; VVC and LCEVC emerging.
- Per-title encoding (Netflix-style) is the biggest single bandwidth-savings lever after picking a modern codec.
- Shot-based encoding pushes the savings further.
- ASIC > GPU > CPU for fleet-scale throughput; CPU > ASIC > GPU for quality per bit.
- Live = one-pass + closed GOPs aligned to segments; VOD = two-pass + per-title ladders.
- Watermarking (A/B variants) for premium content.
- Audio matters too: AAC default, xHE-AAC growing, Opus where WebRTC.

---

## Sources

- [Netflix per-title encode optimization](https://netflixtechblog.com/per-title-encode-optimization-7e99442b62a2)
- [Netflix dynamic optimizer](https://netflixtechblog.com/optimized-shot-based-encodes-now-streaming-4b9464204830)
- [AV1 — Alliance for Open Media](https://aomedia.org/)
- [SVT-AV1 (open-source AV1 encoder)](https://gitlab.com/AOMediaCodec/SVT-AV1)
- [Apple HLS supported codecs](https://developer.apple.com/documentation/http_live_streaming)
- [VMAF — Netflix open-source perceptual metric](https://github.com/Netflix/vmaf)
- [Google Argos Video Coding Unit (research paper)](https://research.google/pubs/warehouse-scale-video-acceleration-co-design-and-deployment-in-the-wild/)
