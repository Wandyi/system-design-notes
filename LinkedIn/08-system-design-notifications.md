# 8 · System Design — Notifications (Air Traffic Controller)

LinkedIn's notification system is internally called **Air Traffic Controller (ATC)**. The metaphor is exact: hundreds of millions of "flights" (potential notifications) per day, multiple "airports" (channels: push, email, in-app, SMS), and the system's job is to *route, throttle, batch, and deliver* each one without crashing the runway (= the member's inbox tolerance).

Notifications are the lever LinkedIn pulls to drive engagement. Done badly, they erode trust and burn email-deliverability reputation. Done well, they re-engage dormant members. The job of ATC is to maximize long-term engagement, not short-term clicks.

## 8.1 Requirements

### Functional

- Source events: connection requests, accepted connections, likes, comments, mentions, profile-view alerts, who-viewed-your-profile, job-match suggestions, learning reminders, hiring updates, Sales Navigator alerts, marketing campaigns.
- Channels: in-app, push (APNs / FCM), email, SMS (sparingly), browser-web-push.
- Per-member preferences: which categories on which channels (granular).
- Rate limiting: don't send more than N notifications/day per member per channel (typically 1–5 emails/day, 5–20 pushes/day).
- Batching: when multiple events happen close in time, combine ("5 new likes on your post") rather than sending each individually.
- Localization: time-zone-aware (don't push at 3am local).
- Compliance: respect global "pause all" (e.g., during a global outage), legal opt-outs (GDPR, CASL), unsubscribe-link semantics.
- A/B testing of notification copy, timing, and channel choice.

### Non-functional

- **Scale**: tens of billions of *candidate* notifications/day; only a fraction get sent. Roughly 1–2 billion actual deliveries/day.
- **Latency**: real-time tier — within 5 seconds for high-value (e.g., "your job application got a response"). Digest tier — minutes to hours.
- **Availability**: 99.9%. A delayed notification is acceptable; a delivered-twice is more painful.
- **Idempotency**: a notification must not be delivered twice for the same event.

## 8.2 Capacity

- Source events ingested: ~100B/day (Kafka).
- Notifications generated (post de-dup, pre-rate-limit): ~10B/day.
- Notifications actually delivered: ~1–2B/day.
- Members: 1B.

So per member, average ~1–2 delivered notifications/day. Power members hit caps; cold members get few.

## 8.3 Architecture

```
                Source events (Kafka topics — likes, comments, jobs, etc.)
                                    │
                                    ▼
                          ┌────────────────────┐
                          │   ATC Aggregator   │   (Samza/Flink)
                          │ - dedupe           │
                          │ - join with member │
                          │   prefs            │
                          │ - join with ML     │
                          │   score (sendrate) │
                          └─────────┬──────────┘
                                    │
                                    ▼
                          ┌────────────────────┐
                          │   Decision Engine  │
                          │ - send/skip/delay  │
                          │ - which channel    │
                          │ - dedupe / batch   │
                          │ - rate-limit       │
                          └─────────┬──────────┘
                                    │
                ┌──────────┬────────┼────────┬───────────┐
                ▼          ▼        ▼        ▼           ▼
           ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
           │ Push   │ │ Email  │ │ In-App │ │ SMS    │ │ Web    │
           │ (APNs/ │ │ (SES/  │ │ (RTG → │ │        │ │ Push   │
           │  FCM)  │ │  own)  │ │ client)│ │        │ │        │
           └────────┘ └────────┘ └────────┘ └────────┘ └────────┘
                                    │
                                    ▼
                          ┌────────────────────┐
                          │ Tracking +         │
                          │ Engagement events  │
                          │ (Kafka → Pinot)    │
                          └────────────────────┘
```

## 8.4 Decision engine — the heart

For each candidate notification (member M, event E, suggested channel C), the decision engine answers:

1. **Should we send at all?** — based on:
   - Member's preferences (channel + category opt-in).
   - Rate limits (daily / hourly per channel).
   - Recent history (did we already send something similar?).
   - ML "send probability" score — how likely is this to drive long-term engagement?
   - Member's predicted active time — defer until member's typical use window.
2. **Which channel?** — push if member is mobile-active; email if dormant; in-app if currently online.
3. **Bundle / batch with other pending notifications?** — if 5 likes happened in 10 minutes, send one "5 new likes" notification, not 5 individual ones.
4. **When?** — immediate vs. scheduled batch (digest).

This is a multi-objective ML problem. Features include:
- Member: recent activity, channel responsiveness, time-zone.
- Event: novelty, expected engagement, criticality (a job offer beats a like).
- Channel: open rates, click-through, unsubscribe rate.
- Cross: this-member-on-this-channel response history.

The model output is a *send-or-skip* decision + a *channel + time* recommendation. Trained offline daily on engagement outcomes (open, click, dwell, unsubscribe).

## 8.5 Idempotency and dedup

Every candidate notification has a `dedup_key` — typically `hash(member_id, event_id, channel)`.

- The decision engine writes to a "decisions" KV store keyed by `dedup_key` with TTL.
- If a duplicate event arrives (Kafka at-least-once), the second time we see the key, skip.
- Cross-channel dedup: also key by `(member_id, event_topic, event_thread)` so we don't send a push *and* an email for the same comment.

Storage: a high-throughput KV (Venice / Redis-ish).

## 8.6 Channel-specific details

### Push (APNs / FCM)

- Per-device tokens stored in `member_devices` table.
- Stale-token cleanup: APNs/FCM return "invalid token" → mark device tombstoned.
- Per-device rate limit (don't push more than N/hour).
- Critical / silent push: used sparingly (background data refresh).

### Email

- LinkedIn runs its own email infra (huge volume; SES / own MTAs at scale).
- Bounce / complaint tracking: respect ISP feedback.
- Reputation management: dedicated IP pools per traffic type (transactional, marketing).
- DKIM, SPF, DMARC — strict policies.
- Unsubscribe link required (CAN-SPAM, CASL).
- Digest emails: aggregated multi-event emails (e.g., "Weekly: what's happening in your network").

### In-app

- Pushed over RTG WebSocket if the member is online.
- Persisted in the member's notifications table; bell-icon count maintained.
- Auto-marked-read on view.

### Web push

- Service-worker subscription.
- Browser-side throttling.

### SMS

- Rare; transactional only (2FA codes, critical security events). Compliance burden is high.

## 8.7 Rate limiting

A staff candidate must talk about the *types* of rate limits:

1. **Per-member, per-channel, per-day** — hard caps.
2. **Per-member, per-channel, per-hour** — burst protection.
3. **Per-member, per-category** — e.g., max 1 "Who viewed your profile" per week.
4. **Global** — system-wide circuit-breaker if email reputation tanks.
5. **Pacing** — campaigns spread sends over windows, not all at once.

Implementation: counters in a fast KV store (Redis-shaped). Token-bucket per (member, channel) refreshed at fixed intervals.

## 8.8 Multi-region

- Source events flow into a regional Kafka.
- ATC instance per region; reads regional Kafka.
- Decision engine reads from a globally-replicated preferences store + a regional decision-store (for rate-limit counters).
- Cross-region dedup is harder — handled by writing a global `delivered_keys` index that any region can check (with relaxed consistency; double-deliveries during failover are rare but possible).

## 8.9 Failure modes

- **Decision-engine slow** — buffer source events on Kafka (with extended retention); catch up after recovery.
- **Email provider failure** — fail over to backup MTA pool.
- **Push provider down** — degrade to in-app + email; alert when restored.
- **Rate-limit storage outage** — fall back to *conservative* defaults (assume rate limit hit). Better to under-send than to spam.
- **Member preferences service slow** — cache aggressively at the ATC level (TTL minutes).
- **Spam wave** — global circuit-breaker can pause categories.

## 8.10 Operational concerns

- **Email reputation** is a long-term asset. One bad campaign can damage IP reputation for weeks. Strict review process for new campaigns.
- **Unsubscribe** must be one-click; honor lifecycle.
- **A/B tests on notifications** — held to long-term retention metrics, not just click-through. (Pushing more aggressively might short-term lift clicks but cause unsubscribes.)
- **Auditability** — every decision logged; member-facing "why did I get this?" UI.

## 8.11 Common follow-ups

> **"How do you avoid notification fatigue?"**
ML model trained on long-term outcomes (30/90-day retention) rather than short-term click. Penalize over-sending in the reward function. Member-aware caps. Per-category preference granularity.

> **"How would you support a 'snooze for 2 hours' button?"**
A `snoozed_until` timestamp per (member, category). Decision engine checks; if before now, skip and re-evaluate after.

> **"How do you handle a campaign sending 100M emails?"**
Pacing: spread over 6 hours by member region. Coordinate with email-reputation throttles. Pre-warm the deliverability pipeline.

> **"What if a notification's underlying event is invalidated (e.g., the post got deleted)?"**
ATC emits a delivery suppression for not-yet-sent notifications. Already-sent notifications carry a "tombstone-able" reference; in-app cleanup; emails can't be unsent.

> **"How would you support a 'do not disturb' window per member?"**
Member preference stores DND window. Decision engine respects: defer or use lower-intrusion channel.