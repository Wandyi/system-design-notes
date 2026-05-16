# 10 · Real-Time Analytics and Live Chat

For a live streaming event, the "is it working?" question is answered by the QoE telemetry pipeline; the "what's the audience doing?" question is answered by the live chat / interaction layer. Both have to scale with the audience — millions of concurrent connections, hundreds of thousands of events per second, sub-second freshness. This file covers both.

## 10.1 Real-time QoE (Quality of Experience) telemetry

### What gets reported

The player periodically sends beacons describing what's happening on the device:

- `session_id`, `user_id` (anonymized), `device`, `OS`, `network_type` (Wi-Fi / 4G / 5G).
- `event` (play, pause, seek, rebuffer_start, rebuffer_end, fatal_error, ABR_switch, ad_start, ad_complete).
- `current_bitrate`, `current_resolution`, `current_buffer_seconds`.
- `cdn_used` (which CDN edge served the last segment).
- `manifest_url`, `segment_url` (for failure diagnosis).
- `throughput_kbps` (estimated).
- `wall_time` and `playback_position`.

Beacon cadence:
- Lifecycle events (play/error/ABR switch) — immediate.
- Periodic heartbeat — every 30–60s.
- Sampling — full 100% at normal load; sample down to 1–10% at extreme peak so the analytics tier doesn't get crushed by its own users.

### The ingest pipeline

```
Player beacon (HTTPS POST)
   |
   v
Beacon endpoint (anycast HTTP, sized for 10× normal traffic)
   |
   v
Kafka / Pulsar topic (partitioned by session_id)
   |
   +---->  Stream processor (Flink / Spark Structured Streaming)
   |          |
   |          +---->  Real-time aggregates (per-minute QoE by region/CDN/device/bitrate)
   |          |
   |          +---->  Anomaly detection (rebuffer spike triggers alert)
   |
   +---->  ClickHouse (raw events, retained 30–90 days)
   |
   +---->  S3 / Parquet (cold archive)
   |
   +---->  Real-time dashboards (Grafana / Datadog)
```

### The metrics that matter

Standard SLIs:

- **Rebuffer ratio** — % of users with a rebuffer in last minute. Goal: < 1%.
- **TTFV p95** — startup latency. Goal: < 3s.
- **Average bitrate** — proxy for visual quality.
- **Fatal error rate** — % of sessions that hit a fatal player error.
- **CDN error rate** — per CDN, per region.
- **Manifest age** — how stale is the manifest at the player.

These get sliced by region, ISP, device, app version, CDN, bitrate rung.

### Cardinality challenges

A naive Prometheus-style metric system blows up with high cardinality (millions of users × devices × ISPs × renditions). Solutions:

- **Pre-aggregate** at the stream processor before storing.
- **Bucket** dimensions (group rare values into "other").
- **Sample** raw events; keep aggregates lossless.
- **Use a time-series DB designed for high cardinality** — ClickHouse, Druid, M3DB.

ClickHouse is the dominant choice for this exact workload — Cloudflare runs DNS-and-HTTP analytics on it at trillions of rows scale.

### Alerting

Alert sources:
- **Burn-rate alerts** on rebuffer-ratio SLO (fast burn vs. slow burn).
- **Anomaly detection** — sudden spike in any error metric, region, or CDN.
- **Per-CDN health** — if CDN A's error rate exceeds X% in region Y, shift traffic to CDN B.
- **End-to-end synthetic probes** — emulated players from many vantage points; alert if any can't play.

During a major event, the NOC has a live dashboard refreshed sub-second, with humans watching for trends the alerts haven't caught yet.

## 10.2 Live chat and interaction

Cricket, Twitch streams, IPL polls — live audio/video draws live conversation. The chat system is a separate scale-out problem.

### Connection topology

Two main protocols:
- **WebSocket** — bidirectional persistent connection. Standard for chat.
- **Server-Sent Events (SSE)** — server → client one-way; clients POST messages over a separate HTTP request. Slightly simpler ops, lower per-connection cost.

A 72.5M-concurrent event implies up to 72.5M live connections — though typically only 10–30% of viewers open chat at any moment, so 10–20M concurrent connections is a more realistic chat scale.

### Architecture

```
Player (chat client)
   |
   v
Edge WebSocket proxy (millions of conns per node — needs tuning of ulimits, epoll)
   |
   v
Regional fanout broker (sharded by chat room ID)
   |
   v
Message bus (Kafka, with one topic per "room family" or per shard)
   |
   v
Persistence (per-room time-series store; users only see recent N messages)
   |
   +---->  Moderation pipeline (async — flagged messages may be retracted seconds later)
```

### Sharding

By **room ID** — all connections in one room land on the same shard so the broker can fan out efficiently.

A single huge room (a cricket-final chat) is itself a hot spot. Subsharding within a room: split into 100 "lanes" of users; each lane sees a sample of messages (with priority for messages from verified accounts, the streamer, etc.). Avoids one room becoming a single-broker bottleneck.

### Throughput numbers

Reported by Hotstar engineering:
- Per second peak messages: hundreds of thousands → ~1M during peak moments.
- Per user message rate-limited (1 msg / 3 seconds typical) — most viewers don't message at all.
- Fanout: 1 message → broadcast to all listeners of the room. With 10M listeners on a single room, that's 10M deliveries per popular message. Pulse-amplification is real.

### Backpressure and lossiness

Chat is not durable; **drop is acceptable**. Design choices:

- Per-connection bounded send buffer. If the client is slow, drop oldest messages.
- Per-room message rate cap — past some throughput, downsample (e.g., serve every 10th message) to keep the user experience legible.
- Priority lanes — verified users, paying subscribers, moderators always get through.

### Moderation

- **Async pipeline** — message goes out, then is scanned for profanity / spam / hate / banned phrases / repeat-spammers.
- Flagged messages can be **retracted** within seconds (clients receive a "delete" event).
- ML classifiers + heuristics + word-list filters; per-language tuning.
- Human moderators for the highest-engagement rooms.

### Polls and other interactions

- **Polls** — server announces the poll, collects votes via separate API, broadcasts results periodically.
- **Reactions** (Twitch's "heart" floods) — extremely high write rate, broadcast as aggregate counts not individual events.
- **Predictions** — IPL-style "who wins next over?" with prize incentives; needs idempotency, anti-cheat (multiple accounts).

## 10.3 Presence

"Show me how many viewers are watching" — this is a presence problem.

For coarse counts (the number on the screen): a sample-based approach is fine; sum across edge nodes via a periodic aggregation. Updated every few seconds.

For per-user "online" indicators (Twitch follower lists): more expensive, requires per-user state across servers, typically a Redis-style fast KV.

## 10.4 Notifications

Push notifications: "Match starting!", "Wickets fall!", "Your team won!".

For 70M+ users, the push provider becomes the bottleneck (APNs / FCM both have per-app throughput limits). Strategies:
- **Multi-provider** push fanout (multiple APNs/FCM creds).
- **Stagger by user** — send 70M pushes over a 5-minute window instead of instantly.
- **Topic-based subscriptions** — users subscribe to "India matches"; pushing to the topic is one operation, recipients are derived.
- **In-app notifications** as fallback so even if push is delayed, the app shows the alert on next open.

## 10.5 Real-time data infrastructure

The same Kafka + ClickHouse spine often serves:
- QoE telemetry → real-time dashboards.
- Chat metrics → moderation triage.
- Player events → recommendation training data (delayed batch).
- Ad telemetry → billing (with strict accuracy SLA).
- Security events → anomaly detection (DDoS, bot patterns).

A staff-level point: don't bolt on a new pipeline for each consumer. One ingest spine, many sinks, with clear contract on event schema.

## 10.6 Cost considerations

WebSocket connections are cheap to hold (~10 KB each, mostly memory) but expensive to send to (network egress). With 20M conns × 50 msgs/min × 200 bytes = ~3.3 GB/s = ~30 Gbps just for chat broadcast.

Strategies:
- Compress (gzip/Brotli per-frame; permessage-deflate).
- Aggregate (reaction floods → reaction-count message per second).
- Sample (every Nth user gets every message; others get a curated subset).

## 10.7 Failure modes

| Failure | Symptom | Mitigation |
|---------|---------|------------|
| Beacon endpoint saturated | Players unable to send beacons; observability gap | Aggressive sampling under load; degrade gracefully |
| Kafka cluster lagged | Real-time metrics delayed | Auto-scale consumers; alert on lag |
| ClickHouse node failure | Dashboards stale or partial | Replica failover; alerts on query latency |
| Chat broker overload | Messages delayed / dropped | Add capacity; downsample; user-visible "high-volume mode" |
| Moderation pipeline backed up | Bad messages reach users | Tighten pre-send filtering; defer secondary checks |
| Push provider throttled | Notifications delayed | Multi-provider; in-app fallback |

## 10.8 Worked design — telemetry for a 50M-viewer event

**Prompt**: "Build the QoE telemetry pipeline."

- **Beacon endpoint**: anycast, 100+ regional PoPs, behind L4 LB. Sized for ~5M req/s peak.
- **Sampling**: dynamic — 100% under 10M concurrent, falling to 1% at 50M.
- **Ingest**: Kafka, partitioned by `session_id`. Topic-per-event-class for selective consumption.
- **Stream processing**: Flink jobs computing rolling metrics by `(region, cdn, bitrate)` over 10s/1m/5m windows.
- **Storage**: ClickHouse hot store; S3+Parquet cold store; aggregates retained 1 year.
- **Dashboards**: Grafana with sub-second refresh, pre-rendered for the NOC; on-call alerting via PagerDuty.
- **Alerts**: burn-rate alerts (fast: 2% of monthly budget in 1h; slow: 5% in 6h); CDN-specific alerts; geographic anomaly alerts.
- **Failure mode**: if the pipeline can't keep up, drop raw events but keep aggregates; alert.

## 10.9 Must-internalize

- QoE telemetry = player beacons → Kafka → stream processor → ClickHouse → dashboards/alerts.
- Sample under load; the analytics tier mustn't fall over from its own users.
- Chat at 10M+ connections = WebSocket fanout via sharded regional brokers, drop is acceptable.
- Moderation is async with retraction; ML + heuristics + word lists.
- Push notifications are throttled by provider; stagger + topic-based subscription.
- One ingest spine, many sinks. Don't bolt new pipelines per consumer.

---

## Sources

- [ClickHouse for observability (Cloudflare blog)](https://blog.cloudflare.com/http-analytics-for-6m-requests-per-second-using-clickhouse/)
- [Scaling Hotstar — chat & polls (AWS re:Invent 2019 PDF)](https://d1.awsstatic.com/events/reinvent/2019/Scaling_Hotstar.com_for_25_million_concurrent_viewers_CMY302.pdf)
- [Apache Flink documentation](https://flink.apache.org/)
- [Designing data-intensive applications — Kleppmann (book)](https://dataintensive.net/)
