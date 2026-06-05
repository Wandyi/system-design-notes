# 14 · System Design — Questions Bank

A curated list of LinkedIn system-design questions reported across Glassdoor, Levels.fyi, Blind, LeetCode discuss, and informal sources. Ordered roughly by reported frequency. Each entry has a short "what they're testing" + "starting points" so you can rehearse mentally even when you can't sit down to write a full design.

If you have only 10 hours to prep system design — do questions 1–10 in depth.

## Tier 1 — High-frequency consumer designs

### 1. Design the LinkedIn News Feed

The canonical question. Covered in `05-system-design-feed.md`.

**What they test**: fanout strategies, ranking pipeline, freshness, scale estimation, multi-region.

**Starting points**: hybrid push/pull, FollowFeed, two-tier ranking, ML feature serving from Venice.

### 2. Design the LinkedIn messaging system

Covered in `07-system-design-messaging.md`.

**What they test**: WebSocket scale, delivery semantics, presence, multi-device, ordering.

**Starting points**: WS gateway with consistent hashing, Espresso storage, Kafka WAL, idempotent client.

### 3. Design "People You May Know" (PYMK) / a friend recommender

Covered in `09-system-design-connections-graph.md`.

**What they test**: graph algorithms, batch + nearline pipelines, ranking, cold-start.

**Starting points**: signal aggregation, Spark candidate generation, Venice serving, online reranking.

### 4. Design LinkedIn Search

Covered in `06-system-design-search.md`.

**What they test**: inverted indexes, sharding, ranking, typeahead vs. full search, federated search.

**Starting points**: Galene/Lucene-style index, scatter-gather, two-stage ranking, near-real-time updates.

### 5. Design the notifications system

Covered in `08-system-design-notifications.md`.

**What they test**: rate limiting, multi-channel routing, idempotency, batching, ML send decisions.

**Starting points**: ATC architecture, decision engine, channel adapters, dedup keys, frequency capping.

### 6. Design "Who Viewed My Profile"

A specialized analytics question.

**What they test**: real-time counting, OLAP, privacy, member-facing dashboards.

**Starting points**: profile-view events to Kafka → Pinot aggregation by `(viewed_member, day)` → fast read API. Privacy: viewer mode (anonymous vs. semi-anonymous vs. full). Premium-gated detailed list.

Discuss:
- Sub-second aggregation at scale: Pinot's strength.
- TTL / retention: typically last 90 days surfaced.
- Anonymity: hide viewer identity unless they're public; aggregated metrics still computed.

### 7. Design "Jobs You May Be Interested In" (JYMBII)

Covered in `10-system-design-jobs-and-recs.md`.

**What they test**: recommender architecture, two-stage retrieval/rank, feature engineering.

**Starting points**: offline candidate gen + online rerank, ML features in Venice, fairness considerations.

### 8. Design the LinkedIn Ads platform

Covered in `11-system-design-ads-platform.md`.

**What they test**: real-time auction, budget pacing, attribution, multi-region monetization.

**Starting points**: per-impression auction <50ms, eCPM ranking, frequency caps, attribution pipeline.

### 9. Design LinkedIn Live

Covered in `13-system-design-realtime-and-live.md`.

**What they test**: video pipelines, CDN, low-latency tradeoffs, chat fanout.

**Starting points**: RTMP ingest → transcode → HLS/LL-HLS → CDN; chat over real-time platform.

### 10. Design a rate limiter (library / service)

Common OO design question.

**What they test**: token bucket / leaky bucket / fixed window / sliding window; distributed coordination; usability.

**Starting points**:
- Interface: `allow(key, cost) → bool` with optional `wait(key) → duration`.
- Storage: in-memory for local; Redis-shaped for distributed.
- Algorithms: token bucket (smooth + bursty), sliding window (precise but heavier).
- Sharding: hash by key; consistent hashing for resilience.
- Failure: fail-open (allow) vs. fail-closed (deny) — discuss the tradeoff.
- Observability: counters per key.

## Tier 2 — Frequently asked

### 11. Design a URL shortener (with click analytics)

A classic warm-up; not LinkedIn-specific but appears in phone screens.

**What they test**: ID generation, KV storage, analytics, custom domains, link expiration.

### 12. Design the LinkedIn typeahead service

Stand-alone version of the typeahead piece of search.

**What they test**: trie / FST data structures, prefix latency, personalization, cache hierarchies.

**Starting points**: per-region trie shards, edge cache, member-specific re-rank step.

### 13. Design "follow / unfollow" at LinkedIn

A subset of the graph problem with directionality.

**What they test**: edge storage, fanout for feeds, multi-region.

**Starting points**: store directed follow edges; consumer pipelines (FollowFeed subscription updates, notifications for first-time follows from someone the followee may know).

### 14. Design the LinkedIn Learning video platform

VOD specialization.

**What they test**: video on-demand pipeline, progress tracking, recommendations.

**Starting points**: Ambry storage, transcoding ladders, HLS/DASH, progress tracking in Espresso, recommendation overlay.

### 15. Design a feature flag service (LIX)

**What they test**: low-latency evaluation, deterministic bucketing, schema evolution, observability.

**Starting points**:
- Bucketing: `hash(member_id, experiment_id) % 100`.
- Distribution: flag definitions pushed to all services from a central registry (with caching).
- Evaluation: pure-function, no remote calls in the hot path.
- Audit: every evaluation can be logged (sampled) for analytics.

### 16. Design a distributed counter (for likes / impressions / etc.)

**What they test**: write throughput, approximate vs. exact, conflict resolution.

**Starting points**: sharded counters with periodic reconciliation; per-shard ack with eventual consistency; CRDT for cross-region.

### 17. Design a sponsored InMail delivery system

**What they test**: campaign management, throttling, delivery pacing, conversion tracking.

**Starting points**: similar to ATC but specialized; targeted segment query → throttled send → conversion-tracking pixel.

### 18. Design the InMail spam filter

**What they test**: real-time ML scoring, training data pipeline, feedback loops.

**Starting points**: feature extraction at send-time, model serving < 50ms, member-flag feedback re-trains, cohort-level fairness.

### 19. Design the LinkedIn endorsement system

"Endorse Vaibhav for Kafka" — many-to-many edges, with deduplication, ranking.

**What they test**: graph-shaped data, anti-abuse (collusion rings), ranking signals.

**Starting points**: endorsement edges (member→member→skill), sharded by skill, periodic batch to compute "top endorsements per member per skill".

### 20. Design "Connections in Common" between two profiles

**What they test**: graph queries, online vs. offline, caching.

**Starting points**: intersect adjacency lists; for hot pairs, cache; for cold pairs, online intersect via LIquid.

## Tier 3 — Behind-the-scenes / infrastructure designs

### 21. Design Kafka

Yes, they ask this. LinkedIn invented it.

**What they test**: distributed log fundamentals, partitioning, replication, exactly-once.

See `15-kafka-deep-dive.md`.

### 22. Design Espresso (a distributed document store)

**What they test**: schema-on-read, partitioning, replication, secondary indexes, online schema evolution.

See `16-voldemort-espresso-ambry.md`.

### 23. Design Pinot (a real-time OLAP store)

**What they test**: columnar storage, segments, real-time vs. offline tables, scatter-gather queries.

See `17-pinot-and-samza.md`.

### 24. Design Venice (a derived-data serving store)

**What they test**: read-optimized KV, bulk load, online updates, partitioning.

See `18-venice-databus-brooklin.md`.

### 25. Design D2 (service discovery)

**What they test**: client-side discovery, dynamic membership, load-aware routing.

**Starting points**: ZooKeeper-backed registry, client subscribes to service paths, degrader load-balancing.

### 26. Design the LinkedIn URL routing layer

Edge / API gateway architecture.

**What they test**: TLS, request routing, abuse mitigation, A/B traffic shifting.

### 27. Design the Brooklin / change-data-capture pipeline

**What they test**: CDC, tailing, schema evolution, replay.

See `18-venice-databus-brooklin.md`.

### 28. Design a multi-tenant ML feature store

**What they test**: feature definition DSL, online vs. offline, train/serve skew.

**Starting points**: Feathr-like DSL; Venice for online, Iceberg for offline; lineage and access control.

### 29. Design the LinkedIn observability stack

**What they test**: metrics ingestion, time-series store, alerting, tracing.

**Starting points**: InGraphs-like agent → time-series DB → Grafana; Iris for alerting; OTel for tracing.

### 30. Design the LinkedIn deployment system

**What they test**: canary, blue-green, ramping, rollback, multi-region.

**Starting points**: ramp via LIX, canary-then-fleet, automated rollback on guardrail breach.

## Tier 4 — OO / API design

These appear in the "Coding 2 / API design" round at L5.

### 31. Design the **API** for the notifications service.

What endpoints does a service need to publish a notification candidate? What are the error semantics? How do you express "send only if not already sent in the last 24 hours"?

### 32. Design the **class hierarchy** for a Job Posting.

Show: paid vs. organic posting, multiple application types (Easy Apply vs. External Apply), expiration, schedule. Discuss inheritance vs. composition.

### 33. Design the **class hierarchy** for a Message and its attachments.

### 34. Design the **API** for LIX (the experimentation framework).

What does `getTreatment(experiment, member, context)` return? How do you support multiple variants? How do you express defaults?

### 35. Design the **API** for a search service.

Query DSL, pagination cursors, error semantics, partial-result hints.

### 36. Design a **plugin model** for the notifications service so new channels (e.g., WhatsApp) can be added without modifying core code.

### 37. Design the **state machine** for a job application.

States: applied → viewed → in-review → interviewed → rejected/hired/withdrawn. Discuss invalid transitions, audit logging, and how Recruiters interact with state changes.

### 38. Design the **API** for the connection-graph service.

`isConnected(a, b)`, `getDegree(a, b)`, `getMutual(a, b)`, `getNthDegreeCount(a, n)`. Define semantics, errors, pagination, batching.

### 39. Design the **interface** for a feature store.

`getFeature(key, name)`, `getFeatures(key, names[])`, batching, point-in-time correctness for training.

### 40. Design the **API** for InMail delivery (Recruiter side).

Quota management, draft saving, scheduled sends, delivery acks, conversion tracking.

---

## How to use this list

- Pick 5–8 questions you're least confident about. Spend 30 min each whiteboarding.
- For the 5 highest-frequency (1–5), do them so often you can sketch them from cold start in 5 minutes.
- For OO questions, prep 2–3 templates: "plugin system", "state-machine domain object", "rate-limiter library".
- For each design, have a back-pocket "what would I do differently if scope X were different" — that's the tell of staff-level thinking.

A staff candidate doesn't just answer the question — they shape it. If the interviewer says "design the feed", you say "great, I want to clarify: are we designing for the average member or the celebrity creator? are we starting greenfield or migrating from $existing? what's the success metric we're optimizing?" That framing — every time — is the calibration anchor.