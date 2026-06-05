# 5 · System Design — LinkedIn News Feed

The Feed is the canonical LinkedIn system-design question. If you're underprepared for one design, fix this one first. Almost every Staff candidate gets asked this or a close variant ("design the feed for a social network at LinkedIn scale, but for content creators specifically", "design the feed but for jobs", etc.).

The internal name for the back-end most people associate with the feed is **FollowFeed** (the pull system) and **Concourse** (the orchestrator). The original push-only system was deprecated as the graph grew.

## 5.1 Requirements (what you should ask)

### Functional

1. A member opens the app — show the most engaging activity from their network (connections + companies/influencers/groups followed).
2. Activity types: text posts, image posts, video, articles, polls, shares (re-shares), comments, reactions, job changes, work-anniversaries, company updates.
3. Members can scroll back ~2 weeks of content.
4. New posts appear within seconds for the publisher and within minutes for followers (eventual consistency).
5. Members can like / comment / share / save — these are write paths that affect ranking and fanout.

### Non-functional

- **Scale**: 1B members, 300M+ MAU, ~100M DAU. Feed-view QPS at peak: tens of thousands of feed loads/sec, hundreds of thousands of cards rendered/sec.
- **Latency**: p99 feed-load server-side ~500ms. p99 post-publish-to-followers-feed-visible: < 60s.
- **Availability**: 99.95%+. A degraded feed (older content) is preferable to a 5xx.
- **Consistency**: read-your-writes for the publisher; eventual for followers.
- **Storage**: hot tier ~2 weeks; older content cold-stored.
- **Multi-region**: active-active across multiple DCs; member's feed served from nearest.

### Things you should NOT design (cut scope clearly)

- The post-creation editor / image upload UI.
- The push-notification path (that's `08-system-design-notifications.md`).
- The video transcoding pipeline (that's `13-system-design-realtime-and-live.md`).

## 5.2 Capacity back-of-envelope

- **Members**: 1B; **followers per member**: median ~50, p90 ~500, top tier (influencers/companies): millions.
- **Posts per day**: tens of millions. Plus comments and reactions: hundreds of millions of write-events.
- **Feed views per day**: tens of billions. With ~25 cards rendered per view → ~hundreds of billions of card impressions/day.
- **Average card size in storage**: ~2 KB metadata + media via Ambry; in-memory representation ~500 bytes.
- **Daily new content + reactions**: ~1 TB compressed (cold), ~100 GB hot.

These numbers are order-of-magnitude — the *method* matters more than the digits.

## 5.3 The fanout dilemma

The defining question. Two extremes:

### Push (fanout-on-write)

When member A posts, the system writes a copy of the post-pointer to each follower's timeline cache. When a follower opens the feed, you just read their pre-computed timeline.

- **Pros**: read is cheap (single timeline lookup).
- **Cons**: write amplification is brutal for popular accounts (millions of timelines to write). Also wastes work — most followers won't see the post in time.

### Pull (fanout-on-read)

Store posts indexed by author. When member B reads the feed, the system queries each of B's followed authors for recent posts and merges.

- **Pros**: cheap writes. Naturally handles celebrity accounts.
- **Cons**: reads are expensive — query 500 authors, merge, rank. p99 latency suffers.

### Hybrid — what LinkedIn does

- **For most members**: pull-based via **FollowFeed**. Each author has a per-author timeline; member-feed loads pull from each followed author and merge.
- **For high-fanout accounts**: push to "active follower" timelines (members likely to log in soon). Reduces unnecessary writes for dormant followers.
- **For very active members**: maintain a pre-computed denormalized timeline (push) to amortize read cost.

Decision is **per-pair** (author popularity × follower activity). Two heuristics together: push if author is small *or* follower is active.

## 5.4 High-level architecture

```
   Publish post                         Read feed
        │                                   │
        ▼                                   ▼
   ┌────────────┐                      ┌──────────────┐
   │ Posting Svc │                     │ Concourse    │ (feed orchestrator)
   │ (writes     │                     │              │
   │ to Espresso)│                     └──────┬───────┘
   └─────┬──────┘                             │
         │                                    │ fan-out reads
         ▼                                    ▼
   ┌────────────┐  CDC  ┌──────────┐    ┌────────────┐
   │  Espresso  │──────▶│  Kafka   │    │ FollowFeed │  ← per-author timelines
   │ (post DB)  │       │ post-evt │    │  service    │
   └────────────┘       └────┬─────┘    └─────┬──────┘
                              │                │
                              ▼                ▼
                       ┌─────────────┐  ┌──────────────┐
                       │ Indexers /  │  │ Ranking Svc  │ → features from Venice
                       │ pipelines   │  │              │   model serving
                       └─────────────┘  └──────────────┘
                              │
                              ▼
                       ┌─────────────┐
                       │ Search      │
                       │ Notif (ATC) │
                       │ Trust ML    │
                       │ Analytics   │
                       └─────────────┘
```

## 5.5 Write path — "member A posts a text update"

1. Client → BFF → **Posting Service**.
2. Posting Service writes to **Espresso** `posts` table:
   - `(post_id, author_id, type, body_ref, created_at, visibility, ...)`.
   - `post_id` is a Snowflake-style time-ordered ID for natural ordering.
   - Body for short posts: inline. For long posts / media: pointer to Ambry.
3. Espresso commits → primary acknowledges → secondary replicas async / sync depending on table policy.
4. **Brooklin** CDC connector tails Espresso → emits `PostCreated` event onto Kafka topic.
5. Downstream consumers fan out:
   - **FollowFeed indexer** writes the post-ref into the author's per-author timeline.
   - **Push-fanout worker** (for hot followers + low-fanout authors) writes into recipient timelines.
   - **Search indexer** indexes the post text into Galene.
   - **Trust ML pipeline** runs spam classifier.
   - **Notifications service** queues up potential push deliveries.
   - **Analytics** aggregates the write event.

End-to-end SLA: post visible to author within <1s (read-your-writes via cache); to followers via FollowFeed within minutes.

## 5.6 Read path — "member B opens their feed"

1. Client → BFF → **Concourse**.
2. Concourse calls **First Pass / Candidate Generation**:
   - Identify member B's followed entities (people, companies, hashtags, groups, etc.) — from the network service.
   - For each followed entity, fetch recent N posts from **FollowFeed**.
   - For *push-stored* recipients, also read the pre-computed timeline shard.
   - Add other candidate streams: jobs you may like, suggested posts, sponsored content.
   - Deduplicate (same post from multiple paths) and gather ~1000 candidates.
3. Concourse calls **Ranking Service**:
   - Pull member features (interests, engagement history) from **Venice** feature store.
   - Pull content features (post topic embeddings, freshness, engagement velocity) from another Venice store.
   - Pull contextual features (time of day, device, dwell-time prior).
   - Run **two-tier ranking**:
     - First: a fast linear or lightweight model to score all 1000 → top 200.
     - Second: a heavier GBDT/DNN model to score top 200 → ordered final ~50.
4. Concourse calls **Hydration** to fetch full content for top 25 cards:
   - Body, media (Ambry URLs), likes/comments counts, author info, contextual data ("X also liked this").
5. **Promo/Ad injection** — sponsored cards interleaved per business rules.
6. **Tracking events** emitted into Kafka for every impression + ranking score.
7. Response returned; client renders.

Total server-side budget: ~500ms p99. Ranking is usually the biggest cost (~150–250ms).

## 5.7 Storage details

### Posts table (Espresso)

```
posts {
  post_id: BIGINT (PK, time-ordered)
  author_id: BIGINT
  type: ENUM
  body_ref: STRING (inline or Ambry pointer)
  visibility: ENUM (PUBLIC, CONNECTIONS, GROUP)
  language: STRING
  created_at: TIMESTAMP
  edited_at: TIMESTAMP
  deleted: BOOL
}
indexed by: author_id (secondary), created_at
```

### Per-author timeline (FollowFeed)

- Author → list of post_ids, descending time.
- Stored in a partitioned KV store (Venice / Voldemort / similar), one row per author.
- TTL ~2 weeks or 1000 posts, whichever larger.

### Per-member precomputed timeline (push tier)

- Member → list of (post_id, author_id, push_ts) for posts pushed by hot-fanout flows.
- Trimmed aggressively (top N).
- Backed by Venice or an in-memory store; cold members evicted.

### Counters

- Likes/comments/reshares counts per post: maintained by a **counters service**.
- Approximate (eventually consistent) for performance — exact via background reconciliation.
- Backed by a sharded KV with optimistic increments.

### Engagement events (Pinot)

- All impressions + clicks land in Pinot.
- Used for retrospective analytics, A/B test deltas, and feature engineering.

## 5.8 Ranking (the heart of the system)

A whole interview can be spent on this. Key concepts:

### Features

- **Member features**: industry, seniority, recent topics engaged, network density.
- **Content features**: post topic, embeddings, freshness, engagement velocity.
- **Author features**: author authority, prior engagement with this member, "first-degree closeness".
- **Context features**: device, time of day, network speed.
- **Cross features**: member–author interaction history, member–topic affinity.

### Models

- Two-stage retrieval-then-ranking is canonical.
- **Stage 1 (Retrieval)**: candidate generation from multiple sources (followed, recommended, viral). Cheap.
- **Stage 2 (Ranking)**: GBDT / DNN producing a probability of {click, dwell, react, share}.
- **Multi-objective**: a weighted sum of probabilities for engagement actions, balanced with diversity, freshness, content type mix.

### Calibration

- Models output scores; calibration maps to probabilities so we can compare across signals.
- Important for ads: under-calibrated bid-shading damages revenue.

### Online learning

- Feature store updates near-real-time via Samza/Flink → Venice.
- Models retrain offline daily on Spark over Hive/Iceberg.
- Online learning for some features (e.g., creator velocity) keeps freshness.

### Cold-start

- New members → use demographic + employer + skills-based heuristics until interaction history accrues.
- New posts → start with author-prior + topic-popularity priors; update as engagement accumulates.

## 5.9 Multi-region

- **Posts**: written to local DC's Espresso → replicated async to all DCs.
- **FollowFeed timelines**: built per-DC by consuming the global post stream via Kafka mirroring (MirrorMaker or Brooklin).
- **Ranking models**: replicated to each DC; serving is local.
- **Reads**: served from member's primary region (sticky).

Conflict resolution: posts are immutable once created (edits are tombstones + new versions); counters are CRDTs with periodic global reconciliation; deletes propagate as tombstones.

## 5.10 Failure modes you must call out

1. **Hot author** — a celebrity posts; their fanout floods the queue. **Mitigation**: rate-limit fanout per author, prioritize hot followers, fall back to pull for cold followers.
2. **Ranking-model outage** — fall back to a heuristic ranker (chronological + minimal popularity score). Better than 5xx.
3. **Venice unavailable** — serve with default features (calibration suffers but feed still loads).
4. **Cross-DC replication lag** — surface read-your-writes via cache on the publisher's region; other regions catch up.
5. **Cache stampede** — request coalescing on hydration calls.
6. **Spam wave** — Trust ML must shadow-ban posts before they hit ranking; rate-limit per-author write QPS.

## 5.11 Operational concerns

- **A/B testing every change**: ranking changes ramp through LIX. Guardrail metrics: complaint rate, p99 latency, downstream revenue.
- **On-call**: feed is a Tier-0 service. Drill incidents quarterly.
- **Capacity planning**: model the QPS curve; pre-warm caches before high-event days (e.g., Microsoft earnings, major election news).
- **Logging**: every ranking decision logged with feature snapshot (for offline retraining and debugging).
- **Cost**: ranking is the biggest cost. Quantizing models, distilling, and pushing to cheaper hardware are constant projects.

## 5.12 Common follow-ups and answers

> **"How do you handle a 1000x viral post?"**
Author-fanout queue gets back-pressure; switch to pull-only for that author; alert; lift rate limits for the impacted shard.

> **"What if Espresso has a 30s outage in one region?"**
BFF degrades to read-only feeds from cache; writes get queued in a write-ahead Kafka topic and re-applied; member sees a banner "posting is paused".

> **"How would you migrate from push-only to hybrid?"**
Dark-launch the pull path; run both; diff outputs in shadow mode for weeks; ramp pull traffic by member percent; watch ranking metrics; eventually deprecate push for cold followers; preserve push for active followers as the perf optimization.

> **"How do you prevent a model regression from hurting engagement?"**
Multi-arm bandit between champion and challenger; auto-rollback if guardrails breach; per-segment monitoring.

> **"What about graph privacy — should this post be visible to that viewer?"**
Visibility evaluated at read time: privacy service + connection-degree check + group-membership check. Cache visibility decisions per (post, viewer) lazily.

> **"What if a 'second-degree only' post needs to know if viewer is 2nd degree from author at read time?"**
Maintain a "second-degree-fingerprint" cache for hot author/viewer pairs; for cold pairs, online query against the graph store.

## 5.13 The deepest dive: building the timeline at read time

Imagine you must build FollowFeed from scratch. Here's the spec the interviewer wants you to recite:

- **Sharding**: by author_id. Each shard holds per-author append-only timelines.
- **Storage**: RocksDB on Helix-managed nodes; bullet-style indexing.
- **API**:
  - `appendPost(author_id, post_id, post_meta)`
  - `getRecentPosts(author_id, since_ts, limit)`
- **Replication**: 3-way within DC; async cross-DC.
- **Consumer (read path)**: BFF fans out N parallel `getRecentPosts` calls (N = followed entity count, often hundreds). Aggregator merges and trims.
- **Fanout optimization**: for members following 5000+ entities (which exists — power users), use a two-stage pull: scatter to a tier of intermediate aggregators each handling a sub-list of authors; gather results.
- **Tail latency**: hedged requests, request coalescing, cancellation tokens.

A staff candidate sketches this cleanly and acknowledges: yes, FollowFeed is a fan-out-at-read service that depends on the right shard map and aggressive caching of the hottest authors' timelines.