# 11 · System Design — Marketing Solutions (Ads Platform)

LinkedIn Marketing Solutions (LMS) is the ads platform. It is the second-largest revenue segment after Talent Solutions. Ad formats include Sponsored Content (in-feed), Message Ads (InMail), Dynamic Ads (personalized), Conversation Ads (interactive InMail), Sponsored Video, and Audience Network (extending reach to third-party sites).

The ads system is built around a real-time auction at every impression. A staff candidate is expected to know the auction, the ranker, the budget pacer, the attribution pipeline, and the privacy guardrails.

## 11.1 Requirements

### Functional

- Advertisers create campaigns with targeting (audience), creative, budget, bid, schedule, optimization goal.
- Audience targeting: location, industry, job title, seniority, skill, company size, members in specific companies, retargeting (visited a URL), matched audiences (upload a list).
- Ad formats: Sponsored Content, Message Ads, Dynamic Ads, Video, Carousel, Conversation Ads.
- Real-time auction: for every member-impression opportunity, select an ad.
- Conversion tracking: pixels, server-to-server APIs, LinkedIn Insight Tag.
- Reporting: real-time and historical, multi-dimensional (campaign × audience segment × placement × time).
- Brand safety: ads must not appear next to inappropriate content.
- Frequency capping: don't show the same ad to the same member too often.

### Non-functional

- **Scale**: hundreds of thousands of active campaigns, hundreds of millions of ad impressions/day, billions of auction decisions/day (most auctions don't result in an impression).
- **Auction latency budget**: 50–100ms (inside the feed-load p99 of ~500ms).
- **Reporting freshness**: near-real-time for advertisers (< 5min lag preferred); final reconciled within 24h.
- **Availability**: 99.99%+. An ad outage is direct revenue loss.
- **Click attribution**: 30-day window typical; multi-touch attribution increasingly important.

## 11.2 Architecture

```
       Member (impression opportunity in Feed)
              │
              ▼
       ┌──────────────┐
       │ Feed Service │
       └──────┬───────┘
              │  "fill ad slot for member M, context C"
              ▼
       ┌──────────────────────────┐         ┌────────────────┐
       │  Ad Serving (auction)    │ ◀────── │ Ad Index       │
       │  - retrieve eligible ads │         │ (campaigns,    │
       │  - score each (pCTR,bid) │         │  creatives,    │
       │  - rank, apply budget    │         │  targeting)    │
       │  - apply policies        │         └────────────────┘
       │  - pick winner           │
       └─────┬────────────────────┘
             │
             ▼
       ┌──────────────────┐
       │ Tracking (impr,  │ ───► Kafka ───► Pinot (real-time reports)
       │  click, convert) │              │
       └──────────────────┘              └─► Hadoop/Spark batch
                                              │
                                              ▼
                                        ┌──────────────────┐
                                        │ Attribution &    │
                                        │ Reconciliation   │
                                        └──────────────────┘
                                              │
                                              ▼
                                        ┌──────────────────┐
                                        │ Billing /        │
                                        │ Invoicing        │
                                        └──────────────────┘
```

## 11.3 The real-time auction

Per-impression, in < 50ms:

1. **Targeting retrieval**: from the impression context (member M, page P, content C), retrieve all eligible campaigns:
   - Audience match (M is in advertiser's targeted segments).
   - Budget remaining.
   - Schedule active.
   - Format compatible with placement.
   This filter must be fast. Approach: pre-indexed inverted lookup (Galene-like) of campaigns by audience predicates. Retrieve hundreds of candidates.

2. **Predicted CTR (pCTR) scoring**: for each eligible ad, predict the probability M will click. Inputs: member features × ad features × context features. Model: a lightweight DNN or GBDT serving sub-ms per inference; batch the candidate set.

3. **Bid retrieval & second-price ranking**:
   - Each ad has a bid (CPM, CPC, CPL — depending on the optimization goal).
   - Compute **eCPM = bid × pCTR × quality_score**.
   - Apply LinkedIn's quality multipliers (creative quality, relevance, brand safety).
   - Rank by eCPM.
   - Apply **second-price auction** (winner pays just above the runner-up's eCPM).

4. **Frequency cap check**: has this ad been shown to M too many times today?

5. **Brand safety check**: is the surrounding content acceptable to the brand? (Brand-safety classifier on the host content.)

6. **Pacing check**: should the budget pacer slow this campaign?

7. **Winner selection** → return ad creative + tracking IDs.

8. **Emit auction events** to Kafka regardless of impression: every impression, every win, every loss (sampled). These power model training and reporting.

## 11.4 Budget pacing

A campaign with $10K/day budget should spend it across the day, not in the first hour. The pacer:

- Estimates daily impression supply for the targeted audience.
- Computes "pacing rate" — fraction of eligible auctions in which to participate now.
- Adjusts every few minutes based on actual spend.
- Avoids end-of-day starvation (gradual ramp-down).

Implementation:
- Per-campaign **spend tracker** — fast counter, eventually consistent.
- Per-campaign **pacing controller** — closed-loop PID-like.
- Spend tracker updates from real-time auction events; reconciled with billing nightly.

## 11.5 Frequency capping

Member-ad-pair tracking: how many times this member has seen this ad / campaign / advertiser today / this week.

- Stored in a fast KV per (member_id, advertiser_id, day) — TTL-based.
- Read on every auction (must be fast: < 5ms).
- Updated on impression — write-behind allowed if read-consistent.

## 11.6 Attribution

When a member clicks an ad and later converts (visits a page, fills a form, buys), attribute the conversion to the ad. Hard problems:

- **Multi-touch**: member may have seen ads from multiple campaigns over weeks.
- **Cross-device**: clicked on mobile, converted on web.
- **Privacy**: third-party tracking is increasingly restricted; iOS App Tracking Transparency, EU restrictions.
- **Fraud**: invalid clicks, click farms, bot traffic.

Attribution pipeline (Spark batch + Samza near-real-time):
- **Joiner**: join impression / click events with conversion events keyed by `(member_id, advertiser_pixel)`.
- **Attribution rules**: last-click within 30 days (default); first-click; multi-touch with credit splitting.
- **Validation**: fraud filters (bot detection, click flooding).
- **Output**: attributed conversions per campaign per day. Feeds billing + reporting.

For real-time advertiser dashboards: a near-real-time approximation; final numbers reconciled overnight.

## 11.7 Insight Tag and conversion APIs

- **Insight Tag**: JS pixel on advertiser's website tracking visits, downloads, form submissions.
- **Conversion API**: server-to-server post-back, more reliable in a privacy-restricted browser world.
- **Matched Audiences**: advertiser uploads a CSV of emails; matched against LinkedIn members; segment used for targeting.

## 11.8 Reporting (Pinot is the workhorse)

- Advertiser dashboards show metrics by campaign × audience × placement × time × format.
- Pinot's strength: real-time ingestion + multi-dimensional aggregation + sub-second query.
- Data model: a wide event table per impression/click/conversion with all dimensions denormalized.
- Pinot tables: real-time (Kafka-fed) + offline (Hive-backed for historical, more compressed).

A staff candidate must be able to discuss Pinot's segment model — see `17-pinot-and-samza.md`.

## 11.9 Brand safety

- Every ad surface has a **brand safety classifier** that scores the surrounding content (post / page / article).
- If score below threshold → ad is suppressed for that impression.
- Advertisers can opt into stricter tiers (no political, no controversial).
- ML pipeline trained on labeled content; updated weekly.

## 11.10 Multi-region

- Auction must be local (latency budget too tight for cross-region).
- Campaign + creative + targeting metadata replicated globally via Brooklin.
- Spend counters: per-region, reconciled globally every few minutes.
- Risk: minor over-spend across regions on the boundary of budget exhaustion. Reconciled via nightly truing.

## 11.11 Failure modes

- **Ad index unavailable** — degrade to "house ads" (no monetization, but no blank slot).
- **Auction service slow** — feed proceeds without ads; revenue loss but no member impact.
- **Conversion API down** — buffer events in Kafka; replay when restored.
- **Spend counter under/over-counts** — reconcile nightly; refund advertisers on over-charge.
- **Fraud spike** — auto-quarantine suspicious traffic; alert; manual review.

## 11.12 Operational concerns

- **Revenue impact** of every change is measured. A latency regression in auction can shrink monetization (lower bids picked because timeout).
- **Advertiser trust** depends on accurate reporting. Discrepancies > a few % trigger advertiser escalations.
- **Regulatory compliance** — GDPR, ePrivacy, CCPA, EU Digital Services Act, ad-disclosure laws.
- **Fairness in targeting** — explicit guardrails prevent discriminatory targeting on jobs and housing.
- **Member experience** — too many ads erodes trust. Frequency caps, quality scores, member feedback ("hide this ad") all feed back into the system.

## 11.13 Common follow-ups

> **"How do you avoid showing the same campaign twice in one feed scroll?"**
Session-level frequency cap; per-session impression tracker held briefly in cache; auction sees the session ID and excludes recently-served campaigns.

> **"How do you handle a 100x viral content event affecting brand safety?"**
Streaming brand-safety classifier on Kafka content stream; if a content node gets classified low, suppress ads on related surfaces immediately. Alert.

> **"How do you train pCTR models without privacy violation?"**
Aggregated features only; differential privacy where possible; member-data access tightly controlled; on-prem model training.

> **"How do you support real-time budget exhaustion?"**
Reach 95% of budget → tighten pacer aggressively. Hit 100% → auto-pause campaign; alert advertiser. Brief over-spend allowance (1–2%) absorbed by LMS as a feature.

> **"How do you support cross-device attribution without cookies?"**
LinkedIn has a logged-in identity graph (member_id is the same across devices once logged in). For non-LinkedIn-logged-in traffic (Audience Network), fingerprinting / hashed-email matching where opted-in.