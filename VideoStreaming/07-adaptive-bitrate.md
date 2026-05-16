# 7 · Adaptive Bitrate (ABR) — Player-Side Depth

ABR is the client-side algorithm that picks which rendition to play next. It's where the player's intelligence lives. The 30%-better encoder helps only if the player picks the right rung. A staff-level interview will probe how ABR algorithms work, when each fails, and how QoE is measured.

## 7.1 The setup

Player has a manifest listing N renditions (R_1 < R_2 < … < R_N by bitrate). For each segment (every few seconds), the player must pick a rendition. Goals:

- **Maximize visual quality** — pick the highest bitrate.
- **Avoid rebuffering** — pick low enough that the buffer doesn't drain.
- **Smooth experience** — don't switch up/down rapidly (oscillation is jarring).
- **Fast startup** — pick low at the start to fill buffer quickly.

These goals trade off. ABR algorithms make different choices.

## 7.2 The buffer model

The player maintains a buffer of decoded video frames. Buffer level = seconds of video queued.

- If buffer is **high** (e.g., 30s), the player can afford to fetch a high bitrate even if download is slow.
- If buffer is **low** (e.g., 3s), the player must fetch a lower bitrate to avoid running dry.

Most modern ABR algorithms use buffer level as a primary input. They live in a (buffer_level, predicted_throughput) state space.

## 7.3 The three algorithm families

### 1. Throughput-based ABR (the old way)

The classic algorithm:
- Estimate the throughput of recent segment downloads (moving average / EWMA).
- Pick the highest rendition whose bitrate ≤ estimated throughput × safety factor (e.g., 0.85).

Pros: simple, easy to reason about.
Cons: throughput estimation is noisy; mobile networks oscillate; oscillation in throughput → rendition oscillation. Doesn't directly know if the buffer will drain.

Players that started here: early HLS native, early DASH.js.

### 2. Buffer-based ABR (BBA, BOLA)

Stanford's BBA (Buffer-Based Adaptation, Huang et al., SIGCOMM 2014) takes the bold position: ignore throughput, only look at buffer level.

- If buffer > B_high: switch to highest rendition.
- If buffer < B_low: switch to lowest rendition.
- Linear mapping in between.

Surprisingly robust because it adapts to the *outcome* (buffer level) rather than the *input* (throughput estimate). Less oscillation.

**BOLA** (Buffer Occupancy-based Lyapunov Algorithm, Spiteri et al., INFOCOM 2016) formalizes this with Lyapunov optimization. Used in dash.js as the default ABR.

### 3. Hybrid / model-predictive (MPC, Pensieve)

**MPC** (Model Predictive Control, Yin et al., SIGCOMM 2015): combine throughput estimate + buffer level + lookahead. At each segment, solve a small optimization: maximize a quality-and-smoothness function over the next K segments given predicted throughput.

**Pensieve** (Mao et al., SIGCOMM 2017): replace the optimization with a neural network trained by reinforcement learning. Outperforms hand-tuned heuristics in evaluation.

In practice, production players use a combination of all three plus heuristics for edge cases. Netflix's, YouTube's, JioHotstar's ABR algorithms are proprietary; they likely take ideas from MPC/Pensieve but include device-specific tuning, ad-break handling, startup phase, etc.

## 7.4 Phases of an ABR algorithm

A real ABR has phases:

- **Startup** — buffer is empty. Must pick low to fill fast and start playback fast (TTFV).
- **Steady state** — buffer is reasonable; pick the optimal balance.
- **Rebuffer recovery** — buffer drained; pick lowest until refill.
- **Seek / track switch** — buffer flush; restart startup phase.
- **Ad transition** — explicit handling needed for ad bitrate vs. content bitrate; sometimes a separate ABR runs during ad.

## 7.5 QoE metrics (what ABR is actually optimizing)

The player measures and reports:

- **Time-to-first-video (TTFV)** — click to first frame. Sub-100ms is YouTube-tier.
- **Rebuffer ratio** — total rebuffer time / total play time. Goal: < 1%.
- **Average bitrate / video quality index** — weighted by perceptual quality (VMAF, or vendor-specific quality score).
- **Bitrate switches per minute** — oscillation indicator.
- **Startup bitrate** — quality of first segment shown.
- **Player errors / fatal errors** — outright playback failure.

Multi-metric QoE scores combine these (e.g., Netflix's reported QoE score, ITU-T P.1203 standard).

## 7.6 Bandwidth estimation methods

Players need a throughput estimate. Common techniques:

- **Per-segment EWMA** — exponential weighted moving average of recent segment download throughputs.
- **Harmonic mean** — robust to outliers; used by dash.js's "throughput rule".
- **Sliding window mean** — last K segments.
- **Pacing measurement** — measure time-between-packets within a segment download.

Pitfalls:
- **CDN connection reuse**: if the connection is already warm, the throughput appears higher than for a cold connection.
- **Slow-start**: TCP slow-start can underestimate true throughput for small segments.
- **CDN priorities**: a CDN that prioritizes small responses (manifests) can fool the throughput estimate.

## 7.7 ABR pitfalls and corner cases

- **Oscillation between two rungs** — common when throughput hovers near a boundary. Mitigation: hysteresis (require sustained excess before switching up).
- **Cliff at boundary bitrate** — a rendition that's "barely sustainable" leads to drain → recovery cycles. Mitigation: stricter safety factor, smarter buffer thresholds.
- **Slow startup** — picking too-low for startup leaves first-frame quality terrible. Mitigation: warm cache for first segment, parallel-pre-fetch of first few segments.
- **Variable bitrate** — within a VBR rendition, instantaneous bitrate spikes can exceed the throughput. ABR using *peak* bitrate is too conservative; using *average* leaves rebuffer risk.
- **HTTP/2 priority / multiplexing** — affects download patterns; ABR must adapt.
- **ABR on second screen / multi-app** — shared connection competing with other traffic.

## 7.8 Server-side hints

Modern systems augment client ABR with server-side intelligence:

- **Common Media Client Data (CMCD)** — standardized headers/query params the player sends with each request (e.g., buffer level, requested bitrate, session ID). Lets the CDN see what's going on at the player.
- **Common Media Server Data (CMSD)** — server-to-client hints about delivery (estimated throughput, congestion, fairness).
- **Server-side ABR** — for live, the server can decide which segment to serve based on player hints. Lets bandwidth-fair allocation happen at the server rather than the player.

CMCD/CMSD are standardized by the CTA. Adoption is growing.

## 7.9 ABR for live vs. VOD

For live, ABR has additional constraints:
- **Live edge** — player wants to stay close to the live point; falling too far behind looks like a stall.
- **Catch-up mode** — when buffer drains, the player can briefly speed up playback (1.05–1.1×) to catch up rather than rebuffer.
- **Latency target** — LL-HLS players have a target latency; ABR has to balance bitrate vs. latency vs. buffer.

For VOD, ABR can be more aggressive about buffering ahead (no live-edge constraint).

## 7.10 The startup tradeoff

Two extreme strategies:

1. **Start at lowest, ramp up** — first-frame fast (good TTFV), but initial visual quality is poor.
2. **Start at predicted-throughput rendition** — better first-frame quality, but TTFV slower (one segment download).

Most players take strategy 1 with very short first segments (~1s) so the ramp is fast.

YouTube's approach: pre-fetch first segment at lowest bitrate; play immediately; let ABR upshift over the next few seconds.

## 7.11 ABR for the long tail and the head

A subtle point: the optimal ABR depends on content type.

- **News**: low motion, talking head — even low bitrates look fine. ABR can stay low.
- **Sports / action**: high motion, complex — needs higher bitrates to avoid blocking artifacts.
- **Music videos**: medium motion, audio is critical — bitrate can be lower if audio quality is preserved.

Per-content-type ABR tuning is a thing. Plus, ABR can be coupled to **content-aware encoding** (the encoder told the player "this is a high-complexity scene, prefer a higher rendition during scene N+1 if you can").

## 7.12 The player implementation reality

Reference players to know:
- **shaka-player** — Google's open-source DASH/HLS player.
- **hls.js** — open-source HLS player for browsers.
- **dash.js** — DASH Industry Forum reference player.
- **ExoPlayer / Media3 ExoPlayer** — Android.
- **AVPlayer** — iOS / macOS native.
- **Bitmovin Player, JW Player, THEOPlayer** — commercial SDKs with strong analytics and DRM integration.

Each has knobs for ABR tuning. Real production players are heavily customized: per-device, per-network, per-content tuning.

## 7.13 Worked design — pick the ABR algorithm

**Prompt**: "What ABR algorithm would you ship for a sports-live app on Android, iOS, web?"

Answer:
1. **Default**: a hybrid — buffer-driven (BOLA-style) plus throughput as a sanity check. Same algorithm on all platforms for predictability.
2. **Startup phase**: lowest bitrate for first 2 seconds, then aggressive upshift.
3. **Steady state**: hysteresis-protected up/down switching.
4. **Rebuffer**: drop two rungs and stay at lowest until buffer recovers.
5. **Live latency target**: 3 s; if behind, briefly play at 1.05× to catch up.
6. **CMCD enabled** for server-side visibility.
7. **A/B test** any algorithm change on 5% of users before rollout.
8. **Metric to watch**: rebuffer ratio + average VMAF, scored together.

## 7.14 Must-internalize

- Throughput-based ABR is legacy; buffer-driven (BOLA) and hybrid (MPC, Pensieve) dominate.
- ABR has phases — startup, steady, rebuffer recovery — each with different goals.
- QoE = TTFV + rebuffer ratio + average quality + switch rate, weighted.
- Hysteresis prevents oscillation.
- Live ABR has a latency target; can speed playback to catch up.
- CMCD/CMSD bring server-side awareness; growing adoption.
- A/B test every algorithm change.

---

## Sources

- [BOLA paper — Spiteri et al., INFOCOM 2016](https://arxiv.org/abs/1601.06748)
- [MPC paper — Yin et al., SIGCOMM 2015](https://users.ece.cmu.edu/~vsekar/papers/sigcomm15_pancake.pdf)
- [Pensieve paper — Mao et al., SIGCOMM 2017](https://web.mit.edu/pensieve/)
- [Buffer-Based Adaptation — Huang et al., SIGCOMM 2014](https://yuba.stanford.edu/~nickm/papers/sigcomm2014-video.pdf)
- [Common Media Client Data (CTA-5004)](https://cdn.cta.tech/cta/media/media/resources/standards/pdfs/cta-5004-final.pdf)
- [shaka-player](https://github.com/shaka-project/shaka-player)
- [dash.js BOLA rule](https://github.com/Dash-Industry-Forum/dash.js)
