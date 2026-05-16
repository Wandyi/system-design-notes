# 13 · Staff-Engineer Topics — Streaming Edition

The cross-cutting topics that separate a senior from a staff engineer. Most of these apply across any large distributed system; the streaming examples make them concrete.

## 13.1 The bottleneck is always egress (until it isn't)

For streaming, egress dominates the cost structure of any v1 architecture. A staff-level conversation always names egress first and works backwards:

- "We're spending $X/month on CDN egress."
- "Where can we move bits cheaper?" — better codec, per-title encoding, multi-CDN to negotiate, carrier embedding.
- "What's the secondary cost?" — origin compute, storage, encoding, license servers.

But for a high-feature-velocity team, the bottleneck might be **engineering velocity, not egress**. A staff engineer reads the org's actual constraints, not the textbook ones.

## 13.2 SLI / SLO / SLA — the streaming-flavored set

Per service, name three SLIs, their SLO, alert thresholds:

**Player / playback**
- TTFV p95 — SLO < 3s. Alert if > 5s for 5 min.
- Rebuffer ratio — SLO < 1%. Alert if > 2% for 5 min.
- Fatal-error rate — SLO < 0.05%. Alert if > 0.1%.

**CDN**
- Edge hit ratio per region — SLO > 95%. Alert if < 90%.
- Origin egress GB/s — alert on >2x baseline (sign of cache problem).

**Origin / packager**
- Manifest age seconds — SLO < 5s. Alert if > 15s.
- Packager queue depth — alert on growing trend.

**Recommendation service**
- p99 latency — SLO < 200ms. Alert if > 500ms.
- Recommendation diversity (unique items / total items in served lists) — drift alert.

**Live ingest**
- Frame-drop rate per encoder — SLO 0; alert immediately.
- Audio drift / sync — SLO ≤ 40 ms; alert > 80 ms.

## 13.3 Capacity planning for predictable peaks

The JioHotstar / sports playbook is the canonical example, but applies anywhere:

1. **Forecast** from history. T20 final = 1.2× IPL final ≈ X.
2. **Pre-warm** to peak + safety margin (~20%). Don't rely on autoscale for the first-order load.
3. **Test** at capacity ahead of the event. Synthetic load tests at planned peak.
4. **Pre-position** content at the edge.
5. **Rehearse failover** — actually cut traffic to the secondary contribution path during a low-stakes match.
6. **Define the load-shed ladder** in advance.
7. **Watch a real dashboard** during the event with humans, not just alerts.

## 13.4 Multi-region architecture

For a global VOD (YouTube, Netflix) the data plane needs presence near every user, but the control plane can be more centralized.

- **Data plane**: CDN edges in many regions; cache replication; origin shielding.
- **Control plane**: catalog, entitlement, recommendations — regional copies, cross-region replicated, with a designated primary per record.
- **Active-active vs. active-passive**: most modern stacks are active-active with conflict resolution (last-write-wins, CRDTs, or business-rule arbitration).
- **Data residency**: India / EU / Russia / China have data-residency laws that force regional data isolation.

For JioHotstar (mostly India-focused), regional architecture is simpler — many PoPs within India, one or two global PoPs for diaspora viewers.

## 13.5 Migration playbook (live to better-live)

A live example: upgrading from HLS to LL-HLS without breaking existing viewers.

1. **Encode both** for a window — segments + parts. Storage cost goes up briefly.
2. **Publish both manifests**.
3. **Player feature flag** — small % of users on LL-HLS first.
4. **Compare QoE metrics** for old vs. new cohorts.
5. **Roll forward** if metrics improve; **roll back** by flipping the flag if not.
6. **Decommission** old encoding once 99%+ of clients are on LL-HLS-capable players.

The dual-write-and-shadow-validate pattern (also used for Infoblox NIOS→BloxOne migration) applies here.

## 13.6 Observability — the streaming spin

What gets monitored is shaped by what fails. Streaming-specific:

- **QoE telemetry** — beacons from the player are the only true measure of user experience. Server-side metrics are necessary but insufficient.
- **Per-CDN per-region SLI** — multi-CDN demands per-vendor visibility.
- **Encoder pipeline health** — frame-loss, encoder fall-behind alarms.
- **License server SLA** — DRM outage = no new sessions starting.
- **Search index freshness** — new uploads visible within minutes.
- **Recommendation drift** — model serving the same predictions for too long is a sign of a stuck feature pipeline.

## 13.7 Cost-aware design

For streaming, every architectural decision should be costed:

- **Per-title encoding cost** vs. egress savings. Net positive at most scales.
- **Custom silicon (Argos VCU)** — high CapEx, ROI in months at fleet scale.
- **Carrier-embedded caches** — they pay for themselves quickly in high-volume markets.
- **Multi-CDN** — engineering cost up, vendor cost down via negotiation.
- **AV1 rollout** — encoder CPU up, egress down. Net positive everywhere but the very small operator.

A staff engineer can sketch the rough P&L impact of any architecture decision.

## 13.8 Multi-tenancy (for picks-and-shovels SaaS like Mux, Bitmovin)

For streaming-platform-as-a-service:

- **Tenant isolation** — segment isolation at the storage + CDN layers; per-tenant API keys; logical separation of analytics.
- **Quotas / rate limits** per tenant.
- **Noisy-neighbor** isolation — biggest customers get dedicated shards.
- **Cost attribution** per tenant — even for fixed-price plans, you need to know which customers cost what.

## 13.9 Security at the platform level

- **Token leak detection** — short-lived tokens, anomaly detection on token usage.
- **Bot detection** at the manifest endpoint — credentialed bots downloading entire catalogs.
- **DDoS** — WAF, anycast, rate limiting at edge.
- **Insider risk** — content access logs; audit; least-privilege for internal users.
- **Vendor security review** — every CDN, every license-server vendor, signed PSAs.

## 13.10 Privacy

- **Watch history** is sensitive data. Encrypt at rest, restrict access, allow user deletion.
- **Recommendations training** — anonymize where possible; aggregate where possible.
- **Cross-account leak** — a strict P0 if a user sees another user's continue-watching list.
- **Children's content** — separate profile, no targeted ads, limited recommendation surface.

## 13.11 Resilience patterns

- **Circuit breakers** between services — a slow recommendation service shouldn't slow the playback path.
- **Bulkheads** — separate thread pools / connection pools for each downstream.
- **Timeout budgets** — explicit timeouts per call; never let a slow downstream block a request indefinitely.
- **Graceful degradation** — homepage works without recommender; playback works without ads.
- **Idempotency** — retries everywhere mean idempotency keys everywhere.
- **Failover** — every singleton has a backup; every failover is rehearsed.

## 13.12 ADRs and design docs in the streaming context

A standard streaming-platform ADR is more product-aware than a generic one:

```
# ADR 0017: Adopt CMAF + cbcs for new content

## Status
Accepted, 2026-04-15

## Context
We currently encode HLS in MPEG-TS and DASH in fMP4 separately. Storage cost
is ~1.8× of canonical encoded bytes. License server has separate code paths
for FairPlay (cbcs) and Widevine/PlayReady (cenc).

## Decision
Switch all new content to CMAF fMP4 with cbcs encryption. Maintain MPEG-TS
HLS for legacy iOS 9 devices until 99% of active users are on iOS 10+.

## Consequences
+ Storage drops ~40% for encrypted titles.
+ License server simplifies to one cipher mode.
- ~1% of users (iOS 9) need legacy ladder maintained for the migration window.
- Need to re-encode existing premium catalog (~50K titles).

## Alternatives Considered
- cenc-only: rules out FairPlay; rejected.
- Maintain split forever: ongoing 1.8× storage cost; rejected.
```

## 13.13 The influence-without-authority play in streaming

Specific situations to be ready for:
- **Convince the product team** to accept TTFV regressions for streaming-cost wins.
- **Convince the CDN partner** to invest in a new region for your fast-growing market.
- **Convince infra** to deprecate a legacy codec / format.
- **Convince finance** to invest in custom silicon before the egress savings show on the P&L.

The pattern: write the brief, name the alternatives, quantify the impact, name the people who must sign off. Get into the room before the decision.

## 13.14 The leadership signals interviewers look for

Same as the Infoblox file, with streaming-specific flavor:

1. Clarify scope/scale early.
2. Articulate tradeoffs as costs *and* benefits, with numbers.
3. Mention observability before it's asked.
4. Mention cost before it's asked.
5. Mention multi-tenancy if relevant.
6. Mention privacy if user data is in play.
7. Show pragmatism — "good enough for v1, instrument heavily, plan the v2".
8. Curiosity about the actual business — "is this for live? is the audience global?".

## 13.15 Must-internalize

- Egress dominates cost. Always.
- SLI/SLO numbers above for each component.
- Pre-warming > autoscaling for predictable peaks.
- Multi-region: data plane many regions, control plane fewer.
- Migration: dual-encode + shadow-validate + flag-controlled rollout.
- Cost-aware architecture: every decision sketched in P&L impact.
- Resilience patterns: circuit breakers, bulkheads, timeouts, graceful degradation.
- Influence pattern: brief, alternatives, quantified impact, signers.
