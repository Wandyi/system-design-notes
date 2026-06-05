# 3 · LinkedIn — Macro Architecture

This is the 30,000-foot view: how the pieces in `02-engineering-culture-and-stack.md` actually fit together when a member loads `linkedin.com`, scrolls their feed, accepts a connection, and gets a push notification — all in one session.

If you can sketch a version of this diagram on a whiteboard, you can survive almost any system-design question by anchoring your answer to it.

## 3.1 The journey of a request

```
                     ┌──────────────────────────────────────────────────────────┐
                     │           Member (web / mobile / API client)             │
                     └────────────────────────┬─────────────────────────────────┘
                                              │ HTTPS
                                              ▼
                     ┌──────────────────────────────────────────────────────────┐
                     │   Edge / Anycast PoPs (Azure Front Door + own edges)     │
                     │   • TLS termination, DDoS, WAF                          │
                     │   • GeoDNS / latency-based routing to nearest DC        │
                     └────────────────────────┬─────────────────────────────────┘
                                              │
                                              ▼
                     ┌──────────────────────────────────────────────────────────┐
                     │           Web tier / API gateway / BFF                   │
                     │   • Rest.li or GraphQL gateway                          │
                     │   • Auth (member session, OAuth, SAML for enterprise)   │
                     │   • Rate limiting, throttling, abuse signals             │
                     │   • LIX (A/B test) evaluation                            │
                     └────────────────────────┬─────────────────────────────────┘
                                              │  D2 service discovery (ZooKeeper)
                                              ▼
        ┌─────────────┬──────────────┬──────────────┬──────────────┬──────────────┐
        │             │              │              │              │              │
        ▼             ▼              ▼              ▼              ▼              ▼
   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
   │ Profile │   │  Feed   │   │ Search  │   │  Jobs   │   │  Msg    │   │  ...    │
   │ Service │   │  (FF /  │   │ (Galene │   │  Service│   │  Real-  │   │  500+   │
   │         │   │ Concrs) │   │ Bobcat) │   │         │   │  time   │   │  more   │
   └────┬────┘   └────┬────┘   └────┬────┘   └────┬────┘   └────┬────┘   └─────────┘
        │             │              │              │              │
        ▼             ▼              ▼              ▼              ▼
   ┌─────────┬─────────┬─────────┬─────────┬─────────┬─────────┐
   │ Espresso│Voldemort│ Venice  │  Pinot  │  Ambry  │  Kafka  │
   │ (online │ (older  │(derived │ (OLAP)  │ (blobs) │ (event  │
   │ docs)   │  KV)    │  data)  │         │         │  bus)   │
   └─────────┴─────────┴─────────┴─────────┴─────────┴─────────┘
                                       ▲
                                       │
                      ┌────────────────┴───────────────┐
                      │  Stream + Batch (compute)      │
                      │  • Samza / Flink → Kafka       │
                      │  • Spark / Hive (Hadoop)       │
                      │  • Azkaban orchestration       │
                      └────────────────────────────────┘
                                       ▲
                                       │
                      ┌────────────────┴───────────────┐
                      │  ML / GenAI                    │
                      │  • Feature store (Feathr)      │
                      │  • Pro-ML training/serving     │
                      │  • GAI platform (RAG, LLMs)    │
                      └────────────────────────────────┘
```

## 3.2 Traffic flow — concrete example: "open the feed"

Member opens the app at 8:00am, sees their feed. Here's the data plane:

1. **Mobile app** sends an HTTPS request to `www.linkedin.com/feed?count=10`.
2. **Edge** (Azure Front Door / LinkedIn's edge PoPs) terminates TLS, runs WAF, routes to the nearest active DC.
3. **API gateway / BFF** validates the member session cookie. Calls **LIX** to fetch this member's treatment for the feed-experiment. Throttles abuse.
4. **Feed Service (Concourse + FollowFeed + Sieve)** is invoked.
   - Concourse owns the feed surface; orchestrates fanout / fetch.
   - Calls **Identity / Network Service** to know who the member follows.
   - For each followed entity, retrieves recent activity from **FollowFeed** (pull model) or from the member's pre-computed timeline (push fanout).
   - Calls **Ranking Service**, which:
     - Fetches **features** from Venice / feature store (member features, content features, contextual features).
     - Runs the ranking model (could be a GBDT / DNN, served in JVM or via Triton).
     - Returns ordered candidate set.
   - Calls **Content Service** to hydrate each item (text, images via Ambry CDN, like/comment counts from Pinot or a counters service).
5. **Tracking events** are emitted at every step into **Kafka** — impressions, ranking features, feed-render latency, etc.
6. **Response** flows back to the BFF, gets transformed into the JSON shape the client wants, and returns.
7. **Asynchronous follow-ups**: notifications service may see "this member opened the app" and decide *not* to send a push 30s later because they already saw the unread content; analytics pipelines aggregate the impression events.

End-to-end p99 latency target for a feed page-load on mobile: ~500ms server-side.

## 3.3 Write-side flow — example: "post a comment"

1. Member taps "Post" on a comment.
2. Client sends `POST /comments` to BFF → Feed Service.
3. Feed Service writes the comment to **Espresso** (durable primary).
4. Espresso replicates synchronously within DC (master + 1 sync replica), async cross-DC.
5. A **Brooklin** (CDC) connector reads Espresso's binlog and emits a `CommentCreated` event to **Kafka**.
6. Many downstream consumers process the event:
   - **FollowFeed** indexes the comment for fanout to followers of the commenter.
   - **Notifications Service (ATC)** decides who should be notified (poster + followers of poster + others mentioned).
   - **Search indexer** indexes the comment text into **Galene**.
   - **Trust / Anti-abuse pipeline** runs spam classifiers; may take down or shadow-ban.
   - **Analytics pipeline** aggregates into Pinot / Hive.
   - **ML feature pipeline** updates the commenter's engagement features in Venice.
7. Notifications service decides on delivery channel (push / email / in-app) and rate-limits.
8. Push goes out via APNs / FCM (or LinkedIn's own push platform).

## 3.4 Cross-cutting infrastructure

### Identity, session, auth

- Session cookies (JSESSIONID-like) for web, opaque tokens for mobile.
- OAuth 2.0 for third-party API access.
- SAML / SCIM for enterprise SSO.
- Internally, services authenticate to each other via mutual TLS + signed JWTs (think SPIFFE-ish).

### Rate limiting & abuse

- L7 rate limiting at edge + at API gateway.
- **Per-member, per-IP, per-app-key** quotas.
- Abuse signals fed through a real-time ML pipeline (anti-spam, anti-scraping, fake-account detection).
- Trust engineering owns this; very mature.

### Multi-DC & failover

- LinkedIn historically operated 4 primary DCs (Ashburn, Hillsboro, LCA1, etc.) plus Azure regions.
- **Active-active** for most stateless services.
- **Active-passive** (or active-active with conflict-free strategies) for stateful systems.
- DR drills are routine.
- Each DC is a "stamp" — full stack capable of serving the product alone.

### Caching

- Multi-tier:
  - **Edge / CDN** for static assets and some authenticated content via signed URLs.
  - **Couchbase / Memcached** in front of expensive Espresso reads.
  - **In-process caches** (Caffeine for Java) for hottest paths.
- Cache-invalidation patterns: write-through, or async invalidation via Kafka events. Lots of read-your-writes consistency engineering.

### Tracking events ("EventBus")

- Every consumer-facing surface emits structured tracking events to Kafka.
- Schema-governed (Avro + a schema registry).
- Hundreds of billions of events/day.
- Backbone of analytics, ML feature engineering, abuse detection, and post-incident analysis.

### Search & discovery

- **Galene** — the search platform. Lucene-based.
- Many indexes (people, jobs, content, groups, etc.) each tuned for that surface.
- Real-time indexing via Samza pipeline.
- Vector search increasingly important for AI features.

### Realtime platform

- LinkedIn has its own "**Real-Time Frontend (RTFE)** / **Real-Time Platform**" that powers messaging, presence, typing indicators, notifications.
- WebSocket-based, with fallback to long-poll for restrictive networks.
- Backed by Kafka for fanout and an in-memory presence service.

## 3.5 Capacity rough numbers (order-of-magnitude)

These are public or widely cited:

- **Members**: 1B+ globally.
- **MAU**: ~310M+.
- **DAU**: not always disclosed; rough order of 100M.
- **Feed views / day**: tens of billions.
- **Kafka throughput**: 7+ trillion messages/day across LinkedIn (one of the largest deployments anywhere).
- **Tracking events**: hundreds of billions/day.
- **Voldemort / Venice**: petabytes of derived data.
- **Espresso**: tens of TB primary online state (compressed).
- **Profile updates**: tens of millions/day.
- **Connection requests**: tens of millions/day.
- **Messages sent**: hundreds of millions/day.

For the staff loop, you don't need exact numbers — you need to be able to *estimate*. (See `24-quick-reference-cheatsheets.md` for back-of-the-envelope formulas.)

## 3.6 Three architectural transitions worth knowing

### Monolith → SOA (2008–2014)

- Original monolith was **Leo** (Java + Tomcat + a single Oracle DB).
- Couldn't scale; deployments were quarterly; on-call was painful.
- **Project Inversion** halted feature work for 2+ months and forced the SOA transition.
- Today: 1000+ services, microservice mesh via Rest.li + D2.

### Polyglot data → consolidation on Espresso/Voldemort/Pinot/Venice

- Long tail of bespoke MySQL / Oracle / Cassandra clusters consolidated onto a handful of internal platforms.
- This is the *staff-engineer migration playbook* you'll be quizzed on.

### On-prem → hybrid Azure

- LinkedIn historically ran its own data centers (legendary capacity / cost efficiency).
- Microsoft acquisition mandated Azure migration; multi-year journey.
- Several waves: lift-and-shift, refactor, cloud-native rewrite.
- Be ready to discuss tradeoffs of forklifting Espresso/Voldemort to Azure vs. using managed Cosmos DB or Azure SQL.

## 3.7 What this means for your design answers

A few patterns interviewers love when designing on LinkedIn-shaped systems:

1. **"Tracking event from minute one"** — every design should emit observability + ML events to Kafka.
2. **"Index for the query"** — never serve a query directly from the OLTP store if the access pattern is heavy. Build a derived dataset in Venice or a Pinot table.
3. **"Schema-first"** — strong types over JSON-anything. Rest.li + Avro / PDL idiom.
4. **"Member-first"** — what's the worst experience for a member if this fails? Optimize for that.
5. **"Multi-region from day one"** — write designs that explicitly state which data is per-region vs. globally consistent.
6. **"Cost lens"** — at staff level, account for $/qps. Pinot is cheaper than Druid here. Venice is cheaper than Voldemort RW for derived data. Etc.

You don't have to use every LinkedIn-internal name — but using *some* of them signals you've done the homework.