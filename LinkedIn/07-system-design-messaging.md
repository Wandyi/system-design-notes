# 7 · System Design — Messaging (InMail + Chat)

LinkedIn Messaging is the chat surface inside `linkedin.com` and the mobile app. It's also the substrate for **InMail** (paid messages to non-connections, the highest-monetized message type — Recruiter and Premium). It's a real-time chat at the scale of a billion users.

Mental model: it has to *feel* as fast as WhatsApp while integrating with Recruiter's enterprise workflows (templates, tracking, throttles), Sales Navigator (CRM sync), the Economic Graph (you can only message a 2nd-degree under certain rules), and the notification system.

## 7.1 Requirements

### Functional

- 1:1 and small group chats (typical group size < 50; some larger groups for Sales/recruiting use).
- Send text + images + attachments + voice notes + video.
- Delivery: pending → sent → delivered → read.
- Presence: online / away / offline.
- Typing indicators.
- Conversation list ("inbox") sorted by recent activity.
- Search messages.
- Threading not required (LinkedIn is flat-thread per conversation).
- Sponsored InMail: business sends bulk-targeted messages with conversion tracking.
- Recruiter InMail: paid messages to non-connections with delivery quota.

### Non-functional

- **Scale**: hundreds of millions of messages/day; tens of millions of concurrent online sessions.
- **Latency**: p95 send→receive < 500ms; typing indicator < 200ms.
- **Availability**: 99.99% — chat outages cascade to InMail revenue.
- **Durability**: no message loss after server acks.
- **Consistency**: read-your-writes for the sender; eventual for receivers.
- **Ordering**: per-conversation total order (FIFO from each side; merged by server timestamp / Lamport).

## 7.2 Capacity back-of-envelope

- **Members**: 1B; **MAU** of messaging: ~200M.
- **Concurrent online connections**: a few million at peak.
- **Messages/day**: ~500M.
- **Avg msg size**: ~200 bytes text; attachments handled separately via Ambry.
- **Daily storage growth**: ~100 GB hot, 1 TB cold (including embeddings/index).
- **WebSocket connections per machine**: ~50–100K with optimized stacks; need ~50 machines at peak.

## 7.3 Architecture

```
                                         Tracking
   Client (web / mobile / API)              │
       │       ▲                             ▼
       │ WS    │ WS push                ┌─────────┐
       ▼       │                        │  Kafka  │
   ┌───────────────┐                    └─────────┘
   │ Real-Time     │  ◀───── presence    ▲
   │ Gateway (RTG) │                     │
   │ (WebSocket)   │                     │
   └─────┬─────────┘                     │
         │                               │
         ▼                               │
   ┌──────────────┐   write   ┌──────────────────┐
   │ Messaging    │──────────▶│  Espresso        │
   │ Service      │           │  (conversation,  │
   │              │   read    │   message store) │
   │              │◀──────────┴──────────────────┘
   └──────┬───────┘
          │
          ├─► Presence Service (Redis-like + Kafka)
          ├─► Notifications (ATC) — for offline recipients
          ├─► Search indexer — message search index
          ├─► Trust ML — spam / abuse classifier
          └─► Analytics
```

Key services:

- **RTG (Real-Time Gateway)** — handles WebSocket connections. Stateless-ish: each WS holds member-session state; the gateway routes messages via a session-affinity layer.
- **Messaging Service** — owns conversation + message records. Writes to Espresso. Routes incoming to RTG for fanout.
- **Presence Service** — tracks online status, last-seen, typing indicators. Sub-second updates over Kafka or direct fanout.
- **Notifications (ATC)** — when a recipient is offline, route to push / email instead.

## 7.4 Data model

```
conversations {
  conversation_id: BIGINT (PK)
  participants: SET<member_id>      // small set for 1:1 and small groups
  type: ENUM (DIRECT, GROUP, INMAIL_THREAD)
  created_at, updated_at: TIMESTAMP
  last_message_id: BIGINT
  is_inmail: BOOL
  sponsor_id: BIGINT?               // for Sponsored InMail
}

messages {
  message_id: BIGINT (PK, time-ordered, sortable)
  conversation_id: BIGINT
  sender_id: BIGINT
  body: STRING (inline if small, Ambry pointer if media)
  type: ENUM (TEXT, IMAGE, VIDEO, ATTACHMENT, SYSTEM)
  sent_at: TIMESTAMP
  status: ENUM (PENDING, SENT, DELIVERED, READ)
  // delivery_state per-recipient stored in a sub-table or separate index
}

participants {
  conversation_id: BIGINT
  member_id: BIGINT
  joined_at: TIMESTAMP
  last_read_message_id: BIGINT      // for unread counts / read receipts
  notification_preferences: JSON
}
```

Storage tier:
- **Espresso** for hot tier (last ~90 days).
- **Cold archive** for older messages (HDFS / object storage / cheaper Espresso tables).
- **Ambry** for media.

Sharding: by `conversation_id` (hash). All messages for a conversation co-located, enabling efficient range scans for "load conversation history". Per-participant unread counters stored alongside.

## 7.5 Send path (member A sends a message to member B)

1. Client → WebSocket (or HTTP fallback) → **RTG**.
2. RTG forwards to **Messaging Service**.
3. Messaging Service:
   - Generates `message_id` (Snowflake-like, monotonic per conversation).
   - Validates: A is a participant of the conversation; conversation not blocked; abuse classifier didn't flag earlier messages from A.
   - Writes to Espresso: `messages` row + updates `conversations.last_message_id` and `participants.last_read_message_id` for A.
   - Emits `MessageSent` event onto Kafka.
4. Synchronously returns `message_id` + `sent_at` to client A (read-your-writes).
5. **Async fanout** from Kafka consumer:
   - For each participant in the conversation (excluding sender), check if they have an active RTG session.
     - If yes: push the message over their WebSocket.
     - If no: enqueue a notification (ATC) for push/email.
   - Update presence: A's last-active timestamp.
   - Index message into search.
   - Run abuse classifier on the message body; if positive, may shadow-flag.

## 7.6 Delivery & read-receipt semantics

Four states:
- **Pending** — client has the message; server hasn't ack'd yet.
- **Sent** — server ack'd; persisted.
- **Delivered** — recipient's device received it (RTG push went out, ack'd, or recipient came online and fetched it).
- **Read** — recipient opened the conversation past this message.

Implementation:
- The `messages` row tracks status from the sender's perspective.
- `participants.last_read_message_id` is the high-water mark — a single integer per (conversation, participant). Any message with `id <= last_read_message_id` is read; everything after is unread.
- Read-receipt push: when participant B reads, an event flows back to A's clients ("B read up to msg X").

Privacy: read receipts are user-configurable; the UI respects the setting.

## 7.7 Presence and typing indicators

### Presence

- Three states: ONLINE / AWAY / OFFLINE.
- Stored in a **Presence Service** — a Redis-like in-memory store with TTLs.
- Update flow: WS connection heartbeat → presence service updates last-seen.
- Read flow: when conversation list is loaded, query presence service for each participant.

### Typing indicator

- Pure ephemeral state — never written to storage.
- Client → RTG → push to other conversation participants' RTG → forward as a "typing" event.
- TTL of ~5 seconds; auto-clears.
- Rate-limited (no spam typing events).

## 7.8 WebSocket layer (RTG)

- Long-lived WS connection per active session.
- Sticky to a particular RTG instance via consistent hashing on `member_id`.
- Authentication on connect (member session token).
- Heartbeat every 30s; reconnect on drop.
- For restrictive networks (corporate proxies), fall back to long-poll or SSE.

### Routing — finding the right RTG for a delivery

Two patterns:
- **Centralized routing**: a member→RTG-instance map in a fast store (Redis / ZooKeeper-like). Sender's instance looks up recipient's instance and forwards. Latency: 1 extra hop.
- **Hash-based**: every RTG instance owns members in `hash(member_id) % N`. Senders compute target deterministically.

LinkedIn's pattern is closer to hash-based with periodic membership reshuffles managed by a control plane.

### Scaling

- ~50–100K WS connections per machine (with tuned kernel, epoll, low GC pressure JVM).
- Connection storms after a global event (e.g., LinkedIn outage recovery) are a known failure mode — exponential-backoff on the client side is non-negotiable.

## 7.9 InMail specifics

### Recruiter InMail

- Recruiter has a quota of paid InMails per seat per month (e.g., 150).
- Sending an InMail decrements quota; if recipient responds within 90 days, the quota credit is *refunded*.
- Bulk send supported with throttles.
- Tracking: opens, clicks, replies, accepts, declines.

Implementation:
- `inmail_quota` table tracks per-Recruiter-seat usage.
- A specialized **InMail Throttle Service** rate-limits sends.
- Refund credits emitted from a **response watcher** that consumes message events.
- Tracking events emitted to Kafka, aggregated nightly into Recruiter analytics (Pinot).

### Sponsored InMail

- Business buys a campaign; targets a member segment.
- A **campaign manager** generates messages; pacing ensures budget is spent across the day.
- Compliance: members can mute sponsor categories; honor "do not contact".
- Tracked as ads (impressions, opens, conversions).

## 7.10 Search

- Messages indexed in a per-member private search index (Galene-based).
- Updated near-real-time via Kafka.
- Member can only search their own messages — strict access control.
- Federated with the main search bar (a global search shows messages alongside profiles, posts, etc.).

## 7.11 Multi-region

- **Conversations** sharded by `conversation_id`; each shard primarily homed in one region; replicated async to others.
- **Member presence** is regional — if member's session is in DC-east, their presence record lives there.
- Cross-DC messaging: sender's region writes the message; CDC + Kafka mirror propagate to recipient's region; recipient's RTG fans out.
- Failover: if a region goes down, conversations in that region failover to a secondary; some messages near the failover boundary may need replay from Kafka WAL.

## 7.12 Failure modes

- **WS server crash** — clients reconnect to a different instance; presence state shows a brief flicker; in-flight messages are durable in Espresso so nothing is lost.
- **Espresso slow** — Messaging Service degrades to write-ahead-log mode: persist the message to Kafka WAL, return ack, async drain to Espresso.
- **Presence service outage** — fall back to "unknown" presence; don't block message send.
- **Spam wave** — Trust ML throttles + rate-limits per-sender; some senders get shadow-banned.
- **Hot conversation** (e.g., a viral 50-person group) — pin to a high-capacity shard; rate-limit per-conversation messages/sec.

## 7.13 Operational concerns

- **Privacy**: messages are private; logging redacts content; encryption at rest.
- **Compliance**: legal hold, GDPR delete-my-data, e-discovery for enterprise customers.
- **Abuse**: spam classifier real-time + retroactive sweep.
- **Cost**: WS infrastructure is expensive; presence-only signals can ride a cheaper path (UDP / lightweight protocol).
- **A/B testing**: changes (e.g., new attachment types) ramp via LIX with read/write split.

## 7.14 Things to say in the interview

- Acknowledge **read-your-writes for sender** + **eventual for recipients**.
- Explain **at-least-once delivery** + **idempotent client display** (message_id deduplication).
- Detail the **conversation sharding** key choice.
- Cover **WS scale numbers** (50–100K per machine) and how to scale horizontally.
- Discuss **presence** as a separate concern — different consistency, different storage.
- Bring up **InMail quotas** if the question is LinkedIn-specific.
- Discuss **multi-region** with failover plan.

## 7.15 Common follow-ups

> **"How do you guarantee no message loss?"**
Client retries with idempotency key (client-generated `request_id`). Server: write to Kafka WAL before acking — Espresso write is async. Recovery on Messaging-Service restart consumes the WAL.

> **"What if a recipient is offline for 2 weeks?"**
Conversation list shows unread count on next login. ATC sends throttled email digests after periods of inactivity.

> **"How would you support encrypted messages (E2EE)?"**
Big design shift. Server can no longer index or moderate. Discuss key management (member-device pairs), key rotation, forward secrecy, and the loss of server-side search. LinkedIn does **not** offer E2EE — partly because compliance + abuse moderation are baked in. Be ready to defend why a Recruiter platform can't reasonably do E2EE.

> **"What's the throughput of typing indicators?"**
Bounded — one event per typing-start, one per typing-stop, per conversation, per second max. Cheap.

> **"How would you serve messages to 1000 concurrent group members typing?"**
Coalesce typing events at the conversation level — emit a single "X people are typing" rather than fan out per typer. Aggregate at RTG layer.

> **"How do you guarantee per-conversation FIFO ordering when two participants send simultaneously?"**
Server-assigned monotonic message_id per conversation (via the partition leader). Even if clients see different orders momentarily, eventual consistency on the conversation's canonical ordering converges.