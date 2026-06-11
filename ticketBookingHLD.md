# High-Traffic Ticket Booking System — Comprehensive High-Level Design
## (World Cup, Super Bowl, Concert Mega-Events)

## Table of Contents

1. [Requirements & Scale](#1-requirements--scale)
2. [Back-of-Envelope Estimation](#2-back-of-envelope-estimation)
3. [High-Level Architecture](#3-high-level-architecture)
4. [Core API Design](#4-core-api-design)
5. [Data Models](#5-data-models)
6. [Virtual Waiting Room & Traffic Shaping](#6-virtual-waiting-room--traffic-shaping)
7. [Seat Inventory & Reservation Locking](#7-seat-inventory--reservation-locking)
8. [Booking Flow — End to End](#8-booking-flow--end-to-end)
9. [Payment Integration](#9-payment-integration)
10. [Order Lifecycle & State Machine](#10-order-lifecycle--state-machine)
11. [Ticket Delivery & Validation](#11-ticket-delivery--validation)
12. [Anti-Bot & Fraud Prevention](#12-anti-bot--fraud-prevention)
13. [Notifications & Communication](#13-notifications--communication)
14. [Search & Event Discovery](#14-search--event-discovery)
15. [Database & Storage Architecture](#15-database--storage-architecture)
16. [Caching Architecture](#16-caching-architecture)
17. [Scalability Deep Dive](#17-scalability-deep-dive)
18. [Reliability & Fault Tolerance Deep Dive](#18-reliability--fault-tolerance-deep-dive)
19. [Observability & Operational Excellence](#19-observability--operational-excellence)
20. [Corner Cases & Hard Problems](#20-corner-cases--hard-problems)

---

## 1. Requirements & Scale

### Functional Requirements

- **Browse Events**: Discover upcoming events, view venue seat maps, check prices by section/tier
- **Virtual Queue**: Fair queuing system when demand exceeds capacity (millions of users, thousands of seats)
- **Seat Selection**: Interactive seat map OR best-available auto-assign
- **Temporary Hold**: Lock selected seats for a limited time window while user completes payment
- **Payment**: Charge user, hold/capture pattern for payment integrity
- **Order Confirmation**: Issue ticket (QR code / barcode / mobile ticket)
- **Cancellation & Refund**: Cancel order, release seats, process refund
- **Notifications**: Email/SMS/push for queue position, booking confirmation, event reminders
- **Scheduled Sales**: Tickets go on sale at a precise time (e.g., 10:00 AM sharp)
- **Tiered Access**: Presale (fan club, credit card holders), general sale
- **Transfer & Resale**: Transfer tickets to another user; optional official resale marketplace

### Non-Functional Requirements
[eventDrivenPlatform](eventDrivenPlatform)
- **Availability**: 99.99% during the on-sale window (zero tolerance for downtime during a World Cup sale)
- **Consistency**: **Strong consistency for seat inventory** — overselling even one seat is unacceptable
- **Latency**: Seat availability check < 100ms, booking confirmation < 3 seconds
- **Fairness**: Users who arrive first should get served first (no queue-jumping)
- **Scalability**: Handle 10M+ concurrent users for a single event sale
- **Fraud Resistance**: Bots should not be able to hoard seats

### What Makes This Different from Normal E-Commerce

```
Normal E-Commerce (Amazon):                Ticket Booking (Super Bowl):
  - Inventory: 10,000 units of same SKU     - Inventory: 80,000 UNIQUE seats
  - Fungible: any unit satisfies order       - Non-fungible: seat A1-R5 ≠ seat B3-R12
  - Gradual demand over hours/days           - ALL demand in first 60-120 seconds
  - Oversell → backorder (annoying)          - Oversell → two people at same seat (lawsuit)
  - Low contention per item                  - EXTREME contention: 10M users, 80K seats
  - Price is static                          - Price varies by section, tier, demand
  - No time-limited holds needed             - Must hold seat while user enters payment info
  - Cart can sit for days                    - Hold expires in 5-10 minutes
```

---

## 2. Back-of-Envelope Estimation

```
Event: FIFA World Cup Final
  Venue capacity:          80,000 seats
  Interested users:        50 million (globally)
  Users hitting "Buy" at sale open: ~10 million in first 2 minutes
  
Traffic spike:
  Initial page load:       10M requests in ~30 seconds = 333K requests/sec
  Peak QPS:                ~500K requests/sec (first 10 seconds)
  
  For context: normal Ticketmaster traffic ≈ 5K QPS
  On-sale spike = 100x normal = classic thundering herd

Booking funnel:
  10,000,000  hit the queue
    2,000,000  admitted through queue (staggered over 10 minutes)
      500,000  select seats (rest see "sold out" and leave)
      200,000  attempt to book (race condition on 80K seats)
       80,000  successfully book (= venue capacity)
      120,000  see "seat no longer available" and retry or leave

Transaction volume:
  80,000 successful bookings in ~10 minutes = 133 bookings/sec average
  Peak: ~500 bookings/sec (first wave from queue)
  
  But behind each successful booking:
  - ~3 seat-hold attempts (user picks a seat, it's taken, tries another)
  - ~1 payment authorization
  Total backend transactions: ~500 bookings/sec × 4 ops = 2,000 write ops/sec peak

Seat inventory checks:
  2M admitted users checking seat availability map:
  = 2M × 3 refreshes × 20 sections = 120M seat-availability reads in 10 min
  = 200K reads/sec (must be cached — cannot hit DB)

Payment:
  80K charges in 10 minutes = 133 charges/sec to payment gateway
  Each with auth-hold + capture pattern = 266 payment API calls/sec
  Payment gateway must handle this burst (Stripe/Braintree support 1K+/sec)
```

**Key Insight**: This is not a throughput problem (133 bookings/sec is trivial). 
It's a **concurrency** problem (10M users contending for 80K unique, non-fungible seats) and a **fairness** problem (first-come-first-served under adversarial conditions with bots).

---

## 3. High-Level Architecture

```
        50 million interested users
                    │
           ┌────────▼────────┐
           │       CDN       │  Static: event pages, seat map images, JS/CSS
           │ (CloudFront)    │  Absorbs 90%+ of initial page-load traffic
           └────────┬────────┘
                    │
           ┌────────▼────────┐
           │  API Gateway    │  Rate limiting, auth, geo-routing
           │  + WAF          │  Bot detection (first layer)
           └────────┬────────┘
                    │
        ┌───────────┼───────────────┐
        │           │               │
  ┌─────▼──────┐ ┌──▼────────┐ ┌────▼─────────┐
  │  Virtual   │ │  Event /  │ │  Booking     │
  │  Queue     │ │  Catalog  │ │  Service     │
  │  Service   │ │  Service  │ │              │
  └─────┬──────┘ └──┬────────┘ └────┬─────────┘
        │           │               │
        │      ┌────▼──────┐  ┌─────▼─────────┐
        │      │ Seat Map  │  │  Reservation  │
        │      │ Service   │  │  Service      │
        │      └────┬──────┘  │  (seat locks) │
        │           │         └─────┬─────────┘
        │           │               │
        │    ┌──────▼───────────────▼──────────────┐
        │    │          Inventory Service          │
        │    │  (source of truth for seat status)  │
        │    └──────────────┬──────────────────────┘
        │                   │
  ┌─────▼───────────────────▼──────────────────────┐
  │                 Event Bus (Kafka)              │
  └──┬─────────┬──────────┬──────────┬─────────────┘
     │         │          │          │
┌────▼───┐ ┌──▼─────┐ ┌───▼────┐ ┌───▼──────────┐
│Payment │ │Notif.  │ │Ticket  │ │ Analytics /  │
│Service │ │Service │ │Delivery│ │ Fraud Engine │
└────────┘ └────────┘ └────────┘ └──────────────┘
                         │
              ┌──────────▼───────────┐
              │   Database Layer     │
              │                      │
              │  PostgreSQL          │
              │  (events, orders,    │
              │   inventory, users,  │
              │   payments)          │
              │                      │
              │  Redis               │
              │  (queue state,       │
              │   seat locks,        │
              │   availability cache,│
              │   rate limiters)     │
              └──────────────────────┘
```

### Service Decomposition

| Service | Responsibility | Scaling Characteristic |
|---|---|---|
| **Virtual Queue** | Admits users at controlled rate, assigns queue positions | Stateless workers + Redis sorted set |
| **Event / Catalog** | Event metadata, venue info, pricing tiers | Read-heavy, fully cacheable |
| **Seat Map Service** | Real-time seat availability per section | Read-heavy, cached with short TTL |
| **Inventory Service** | Source of truth for seat status (available/held/sold) | Write-heavy during sale, strong consistency |
| **Reservation Service** | Temporary seat holds with TTL-based expiry | Write-heavy, Redis + DB |
| **Booking Service** | Orchestrates: hold → payment → confirm → ticket | Saga orchestrator |
| **Payment Service** | Auth-hold, capture, refund via payment gateways | External API calls, idempotent |
| **Ticket Delivery** | Generate QR/barcode, deliver via email/app | Async, post-booking |
| **Notification Service** | Queue position updates, confirmations, reminders | High-volume push/email/SMS |
| **Fraud / Anti-Bot** | Bot detection, velocity checks, device fingerprinting | Real-time scoring on every request |

---

## 4. Core API Design

### Event Discovery

```
GET    /v1/events?category=sports&date_from=2026-12-01&cursor=X&limit=20
  → 200 { events: [...], next_cursor }

GET    /v1/events/{event_id}
  → 200 { event_id, name, venue, date, description, sale_start_at,
          presale_start_at, status, pricing_tiers: [...],
          seat_map_url, total_capacity, available_count }

GET    /v1/events/{event_id}/seat-map
  → 200 { sections: [
      { section_id: "A", name: "Lower Bowl", tier: "premium",
        price: 1500.00, total: 5000, available: 3200,
        rows: [ { row: "1", seats: [ { seat_id: "A1-R1-S1", status: "available" },
                                      { seat_id: "A1-R1-S2", status: "held" }, ... ] } ] }
    ] }
    (Full seat-level detail only for selected section; section-level counts for overview)
```

### Queue & Booking Flow

```
POST   /v1/events/{event_id}/queue/join
  Headers: X-Device-Fingerprint, X-Captcha-Token
  → 200 { queue_token: "qt_abc123", position: 148392, estimated_wait_minutes: 12,
          status: "WAITING" }

GET    /v1/events/{event_id}/queue/status
  Headers: Authorization: Bearer {queue_token}
  → 200 { position: 2041, status: "WAITING" | "ADMITTED" | "EXPIRED",
          estimated_wait_minutes: 2 }
  (Client polls every 5-10 seconds. When status = "ADMITTED", proceed to booking.)

POST   /v1/events/{event_id}/hold
  Headers: Authorization: Bearer {queue_token}  ← proves user was admitted through queue
  Body: { seat_ids: ["A1-R5-S10", "A1-R5-S11"], hold_duration_seconds: 420 }
  → 201 { hold_id: "hold_xyz", seats: [...], expires_at: "...",
          hold_duration_seconds: 420 }
  → 409 { error: "seats_unavailable", unavailable: ["A1-R5-S11"],
          suggestion: { nearby_available: ["A1-R5-S12", "A1-R5-S13"] } }

POST   /v1/bookings
  Body: { hold_id: "hold_xyz", payment_method_id: "pm_abc",
          buyer: { name, email, phone } }
  → 201 { booking_id: "bk_001", status: "CONFIRMING", seats: [...] }
  → 410 { error: "hold_expired" }   ← hold timed out before payment completed

GET    /v1/bookings/{booking_id}
  → 200 { booking_id, status: "CONFIRMED" | "CONFIRMING" | "FAILED",
          seats: [...], total_amount: 3000.00, tickets: [...],
          event: {...}, receipt_url }

POST   /v1/bookings/{booking_id}/cancel
  → 200 { refund_amount: 2850.00, cancellation_fee: 150.00,
          refund_status: "PROCESSING" }
```

### Pagination & Real-Time Updates

```
Seat availability:
  Full seat map is NOT fetched on every refresh. Instead:

  GET /v1/events/{event_id}/availability?sections=A,B,C
  → 200 { sections: [
      { section_id: "A", available: 3200, total: 5000, updated_at: "..." },
      { section_id: "B", available: 0, total: 3000, updated_at: "..." },
      ...
    ]}
  TTL: 2 seconds (cached). Section-level counts, not per-seat.

  Only when user clicks a specific section:
  GET /v1/events/{event_id}/sections/{section_id}/seats
  → per-seat availability (heavier query, shorter cache TTL of 1 second)
```

---

## 5. Data Models

### PostgreSQL — Event & Venue

```sql
CREATE TABLE events (
    event_id         BIGINT PRIMARY KEY,
    name             VARCHAR(500),
    description      TEXT,
    venue_id         BIGINT REFERENCES venues(venue_id),
    event_date       TIMESTAMPTZ,
    sale_start_at    TIMESTAMPTZ,      -- when general sale opens
    presale_start_at TIMESTAMPTZ,      -- fan club / credit card presale
    status           VARCHAR(20),      -- DRAFT, ON_SALE, SOLD_OUT, COMPLETED, CANCELLED
    total_capacity   INT,
    available_count  INT,              -- denormalized (updated atomically with seat changes)
    created_at       TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE venues (
    venue_id     BIGINT PRIMARY KEY,
    name         VARCHAR(300),
    city         VARCHAR(100),
    country      VARCHAR(100),
    capacity     INT,
    seat_map_url TEXT                  -- static SVG/JSON of venue layout
);

CREATE TABLE sections (
    section_id   BIGINT PRIMARY KEY,
    venue_id     BIGINT REFERENCES venues(venue_id),
    name         VARCHAR(100),        -- "Lower Bowl A", "Upper Deck 300"
    tier         VARCHAR(50),         -- "platinum", "premium", "standard", "economy"
    total_seats  INT
);
```

### PostgreSQL — Seat Inventory (The Critical Table)

```sql
CREATE TABLE seats (
    seat_id       BIGINT PRIMARY KEY,
    event_id      BIGINT NOT NULL,
    section_id    BIGINT NOT NULL,
    row_name      VARCHAR(10),
    seat_number   VARCHAR(10),
    tier          VARCHAR(50),
    price         NUMERIC(10,2),
    status        VARCHAR(20) NOT NULL DEFAULT 'AVAILABLE',
       -- AVAILABLE, HELD, SOLD, BLOCKED (obstructed view, ADA, etc.)
    held_by       BIGINT,             -- user_id if status = HELD
    held_until    TIMESTAMPTZ,        -- expiry of hold
    sold_to       BIGINT,             -- user_id if status = SOLD
    booking_id    BIGINT,
    version       INT DEFAULT 0,      -- optimistic locking version
    CONSTRAINT uq_seat_event UNIQUE (event_id, section_id, row_name, seat_number)
);

-- Critical index for availability queries
CREATE INDEX idx_seats_event_section_status
    ON seats (event_id, section_id, status);

-- Index for hold expiry cleanup
CREATE INDEX idx_seats_held_until
    ON seats (held_until)
    WHERE status = 'HELD';

-- Partition by event_id for isolation
-- Each high-traffic event gets its own partition (or even its own database)
CREATE TABLE seats_event_12345 PARTITION OF seats FOR VALUES IN (12345);
```

### PostgreSQL — Bookings & Orders

```sql
CREATE TABLE bookings (
    booking_id     BIGINT PRIMARY KEY,
    event_id       BIGINT NOT NULL,
    user_id        BIGINT NOT NULL,
    status         VARCHAR(30) NOT NULL,
       -- HOLD_ACQUIRED, PAYMENT_PENDING, PAYMENT_AUTHORIZED, CONFIRMED,
       -- CANCELLED, REFUNDED, FAILED
    seat_count     INT,
    subtotal       NUMERIC(12,2),
    service_fee    NUMERIC(10,2),
    total_amount   NUMERIC(12,2),
    currency       VARCHAR(3) DEFAULT 'USD',
    payment_intent VARCHAR(100),       -- Stripe/Braintree payment intent ID
    hold_id        VARCHAR(64),
    hold_expires_at TIMESTAMPTZ,
    idempotency_key VARCHAR(64) UNIQUE, -- prevents duplicate bookings
    created_at     TIMESTAMPTZ DEFAULT NOW(),
    confirmed_at   TIMESTAMPTZ,
    cancelled_at   TIMESTAMPTZ
);

CREATE TABLE booking_seats (
    booking_id     BIGINT REFERENCES bookings(booking_id),
    seat_id        BIGINT REFERENCES seats(seat_id),
    PRIMARY KEY (booking_id, seat_id)
);

CREATE TABLE tickets (
    ticket_id      BIGINT PRIMARY KEY,
    booking_id     BIGINT REFERENCES bookings(booking_id),
    seat_id        BIGINT,
    event_id       BIGINT,
    holder_name    VARCHAR(200),
    holder_email   VARCHAR(255),
    qr_code        TEXT,               -- encrypted payload for gate scanning
    barcode        VARCHAR(64),
    status         VARCHAR(20),        -- ACTIVE, USED, CANCELLED, TRANSFERRED
    issued_at      TIMESTAMPTZ,
    used_at        TIMESTAMPTZ
);
```

### Redis — Real-Time State

```
Queue State:
  Key:    queue:{event_id}
  Type:   Sorted Set
  Score:  join_timestamp (epoch ms)
  Member: user_id or queue_token
  → ZRANK to get position, ZRANGEBYSCORE to admit next batch

Queue Admission Cursor:
  Key:    queue:{event_id}:admitted_up_to
  Type:   String (score value)
  → All members with score ≤ this value have been admitted

Seat Holds (Distributed Lock):
  Key:    hold:{event_id}:{seat_id}
  Type:   String (user_id)
  TTL:    420 seconds (7 minutes)
  → SET NX (only succeeds if key doesn't exist = seat not held)
  → On expiry: seat automatically released (no cleanup job needed)

Section Availability Cache:
  Key:    avail:{event_id}:{section_id}
  Type:   String (integer count)
  TTL:    2 seconds
  → DECR on successful hold, INCR on hold release/expiry
  → Periodically reconciled with DB (source of truth)

Rate Limiter (per-user):
  Key:    rl:{user_id}:{event_id}
  Type:   Sorted Set (sliding window)
  → Max 5 hold attempts per minute per user per event

Anti-Bot Score:
  Key:    bot_score:{session_id}
  Type:   String (JSON)
  Value:  { score: 0.87, features: [...], blocked: false }
  TTL:    3600s
```

---

## 6. Virtual Waiting Room & Traffic Shaping

### The Core Problem

```
10,000,000 users hit "Buy Tickets" at 10:00:00.000 AM.
Your booking system can process ~500 bookings/sec.
Venue has 80,000 seats.

Without a queue:
  10M requests → API gateway → booking service → database
  Database receives 10M concurrent connections
  Connection pool exhaustion in 100ms
  Cascading failure across all services
  0 tickets sold. Everyone sees errors. PR disaster.

With a queue:
  10M users → virtual waiting room (absorbs the stampede)
  Queue admits 2,000 users/min to the booking page
  Booking service sees controlled, manageable load
  80,000 tickets sold cleanly in ~15 minutes
  Users see fair position indicator and wait time estimate
```

### Architecture

```
  10M users arrive
       │
       ▼
  ┌──────────────────────────────────────────────────────┐
  │              Virtual Waiting Room                    │
  │                                                      │
  │  Static HTML page served from CDN                    │
  │  (no backend hit for page load — pure CDN)           │
  │                                                      │
  │  Client-side JS:                                     │
  │   1. POST /queue/join → get queue_token + position   │
  │   2. Poll GET /queue/status every 5-10 seconds       │
  │   3. When status = "ADMITTED" → redirect to booking  │
  │                                                      │
  │  Visual: animated progress bar, position number,     │
  │          estimated wait time                         │
  └──────────────────┬───────────────────────────────────┘
                     │
                     ▼
  ┌──────────────────────────────────────────────────────┐
  │              Queue Backend (Redis)                   │
  │                                                      │
  │  ZADD queue:{event_id} {timestamp_ms} {queue_token}  │
  │  → 10M members in sorted set                         │
  │  → Memory: 10M × ~80 bytes = 800 MB (fits easily)    │
  │                                                      │
  │  Admission controller (runs every 1 second):         │
  │   1. Read current admission rate from config         │
  │      (e.g., 200 users/second)                        │
  │   2. ZRANGEBYSCORE queue:{event_id} -inf {cursor}    │
  │      LIMIT 0 200                                     │
  │   3. For each admitted user: set their status to     │
  │      ADMITTED in Redis                               │
  │   4. Advance cursor                                  │
  │                                                      │
  │  Admission rate auto-tuned:                          │
  │   - Monitor booking service latency                  │
  │   - If p95 > 500ms: reduce admission rate            │
  │   - If p95 < 200ms: increase admission rate          │
  │   - Target: keep booking service at 70% capacity     │
  └──────────────────────────────────────────────────────┘
                     │
                     │ (only ~200/sec make it through)
                     ▼
  ┌──────────────────────────────────────────────────────┐
  │              Booking Service                         │
  │  (sees manageable, controlled traffic)               │
  │                                                      │
  │  Every request must present a valid queue_token      │
  │  that has status = ADMITTED.                         │
  │  Requests without valid token → 403 (queue bypass    │
  │  attempt)                                            │
  └──────────────────────────────────────────────────────┘
```

### Queue Fairness & Anti-Gaming

```
Problem: Bots join the queue thousands of times with different accounts.
  Each bot instance gets a queue position.
  10,000 bot entries at positions 1-10,000.
  Real users are pushed to positions 10,001+.

Defense layers:

Layer 1: CAPTCHA at queue join
  - Invisible reCAPTCHA v3 (score-based, no friction for humans)
  - If score < 0.5 → require interactive CAPTCHA
  - Blocks ~80% of basic bots

Layer 2: Device fingerprint dedup
  - Each device gets a fingerprint hash (canvas, WebGL, fonts, screen, etc.)
  - One queue position per device fingerprint per event
  - Second join attempt from same fingerprint → return existing position
  - Blocks multiple accounts from same machine

Layer 3: IP rate limiting
  - Max 3 queue joins per IP per event (residential IPs)
  - Max 10 per IP for known VPN/datacenter IPs
  - Blocks datacenter bot farms

Layer 4: Behavioral analysis (during wait)
  - While in queue, client-side JS collects signals:
    Mouse movements, scroll behavior, tab focus/blur patterns
  - Pure bot: zero mouse movement, tab never focused
  - Suspicious scores → moved to back of queue or required re-CAPTCHA

Layer 5: Queue position is tied to session, not account
  - Logging into a different account doesn't give a new position
  - Position is bound to (device_fingerprint + IP + session) tuple
```

### Queue Status Polling — Scaling to 10M Polls

```
10M users polling every 5 seconds = 2M status checks/sec

If each status check hits Redis:
  ZSCORE queue:{event_id} {queue_token}  → O(1) = ~1µs
  2M/sec × 1µs = 2 seconds of CPU/sec on Redis → trivially handled

But the HTTP layer must handle 2M req/sec:
  Solution: Edge caching + long-poll optimization

  1. Queue status response includes Cache-Control: max-age=3
     CDN caches the response for 3 seconds per user
     Actual backend QPS: 2M / 3 = 667K/sec (still high)

  2. Better: batch position updates
     Instead of per-user cache, the queue service publishes:
       "As of now, admitted up to position 148,000"
     Client knows its position (received at join time).
     Client-side: if my_position > 148,000 → still waiting (no backend call needed)
     Client only calls backend when it's CLOSE to the admission cursor.

     This reduces backend QPS from 2M/sec to ~10K/sec (only users near the front).
```

---

## 7. Seat Inventory & Reservation Locking

This is the single hardest technical problem in the entire system. Two users must **never** be able to book the same seat.

### Approach 1: Pessimistic Locking (SELECT FOR UPDATE)

```sql
BEGIN;
  -- Lock the seat row exclusively. Other transactions trying to lock
  -- the same row will BLOCK until this transaction commits or rolls back.
  SELECT * FROM seats
  WHERE seat_id = 12345 AND event_id = 999 AND status = 'AVAILABLE'
  FOR UPDATE;

  -- If row returned: seat is available and we hold the lock
  UPDATE seats
  SET status = 'HELD', held_by = 42, held_until = NOW() + INTERVAL '7 minutes',
      version = version + 1
  WHERE seat_id = 12345;
COMMIT;

Advantages:
  ✓ Simple. Impossible to oversell. Database enforces exclusivity.
  ✓ No application-level locking needed.

Problems at scale:
  ✗ 200 users trying to hold the same seat → 199 transactions BLOCKED
    waiting for the lock. Each holds a database connection while waiting.
  ✗ Connection pool exhaustion: 200 blocked connections × 1000 concurrent
    seat races = 200,000 connections needed. PostgreSQL caps at ~5000.
  ✗ Lock wait timeout: blocked transactions time out (5s default) →
    error responses → user frustration → retries → more load.
  ✗ Deadlocks: User A holds seat 1, wants seat 2. User B holds seat 2,
    wants seat 1. PostgreSQL detects and kills one, but it's still disruptive.
```

### Approach 2: Optimistic Locking (Version Column)

```sql
-- No locks held. Read the current version.
SELECT seat_id, status, version FROM seats
WHERE seat_id = 12345 AND event_id = 999 AND status = 'AVAILABLE';
-- Returns: version = 3

-- Try to update. The WHERE clause includes the version.
-- If another transaction changed the row, version won't match → 0 rows affected.
UPDATE seats
SET status = 'HELD', held_by = 42,
    held_until = NOW() + INTERVAL '7 minutes',
    version = 4
WHERE seat_id = 12345 AND status = 'AVAILABLE' AND version = 3;

-- Check affected rows:
-- affected_rows = 1 → SUCCESS: we got the seat
-- affected_rows = 0 → CONFLICT: someone else got it first, retry with different seat

Advantages:
  ✓ No blocking. Failed transactions return immediately.
  ✓ No connection pool exhaustion.
  ✓ Natural "fast path": winner completes instantly, losers find out instantly.

Problems:
  ✗ High contention on popular seats: 200 users trying same seat →
    1 wins, 199 get affected_rows=0 → 199 must retry with different seat.
    If they all pick the same "next best" seat → contention cascade.
  ✗ Thundering herd on retry.
```

### Approach 3: Redis Pre-Lock + DB Confirm (What Production Systems Use)

```
Two-phase approach:
  Phase 1: Fast lock in Redis (microseconds, handles the stampede)
  Phase 2: Durable confirm in PostgreSQL (milliseconds, after lock acquired)

This is like a bouncer at a club:
  Redis = bouncer (fast, decides who gets in)
  PostgreSQL = the actual club (durable, authoritative)
```

```
User selects seat A1-R5-S10:

Phase 1: Redis atomic lock (fast path)
  -- SET NX: "set only if key does NOT exist" — atomic, single-threaded in Redis
  SET hold:{event_id}:{seat_id} {user_id} NX EX 420
  
  If return = OK:
    → You got the lock. Seat is yours for 420 seconds (7 minutes).
    → Proceed to Phase 2.
  
  If return = nil:
    → Someone else holds it. Seat unavailable.
    → Return 409 immediately. User picks different seat.
    → Total time: ~1ms. No database hit. No blocking.

Phase 2: Durable record in PostgreSQL (slow path, only for winners)
  UPDATE seats
  SET status = 'HELD', held_by = {user_id},
      held_until = NOW() + INTERVAL '7 minutes', version = version + 1
  WHERE seat_id = {seat_id} AND status = 'AVAILABLE';

  If affected_rows = 0:
    → DB and Redis are out of sync. Redis lock was stale.
    → DEL hold:{event_id}:{seat_id}   (release Redis lock)
    → Return 409 to user.
  
  If affected_rows = 1:
    → Both Redis and DB agree. Hold is durable.
    → Return 201 with hold_id.

Phase 3: Automatic expiry
  If user doesn't complete payment within 7 minutes:
    - Redis key auto-expires (TTL = 420s)
    - Background job checks: seats WHERE status = 'HELD' AND held_until < NOW()
      → UPDATE status = 'AVAILABLE', held_by = NULL
    - Seat is now available for others.
```

### Multi-Seat Atomic Hold (Booking 4 Seats Together)

```
Problem: User wants seats A1-R5-S10, S11, S12, S13 (4 adjacent seats).
  Must hold ALL FOUR or NONE (no partial holds — family shouldn't be split).

Solution: Redis Lua script for atomic multi-key operation.

  -- Lua script: atomic "hold all or hold none"
  EVAL "
    local held = {}
    for i, seat_key in ipairs(KEYS) do
      local result = redis.call('SET', seat_key, ARGV[1], 'NX', 'EX', ARGV[2])
      if not result then
        -- One seat failed. Roll back all previously held seats.
        for _, held_key in ipairs(held) do
          redis.call('DEL', held_key)
        end
        return 0  -- FAILED
      end
      table.insert(held, seat_key)
    end
    return 1  -- SUCCESS: all seats held
  " 4 hold:999:s10 hold:999:s11 hold:999:s12 hold:999:s13  user_42  420

  Redis executes Lua scripts atomically (single-threaded).
  Either ALL four SET NX succeed → return 1 (all held)
  Or ANY fails → rollback all → return 0 (none held)
  
  No partial state possible. User is never split across seats.
```

### Hold Expiry & Seat Release

```
Three mechanisms for releasing held seats:

1. User completes booking → seat transitions to SOLD (happy path)

2. User explicitly cancels → 
   DEL hold:{event_id}:{seat_id}
   UPDATE seats SET status = 'AVAILABLE' WHERE seat_id = ... AND status = 'HELD'

3. Hold expires (user abandoned checkout):
   Redis: key auto-expires after TTL (420s). No action needed.
   PostgreSQL: background job runs every 30 seconds:
   
   UPDATE seats
   SET status = 'AVAILABLE', held_by = NULL, held_until = NULL
   WHERE status = 'HELD' AND held_until < NOW()
   RETURNING seat_id, event_id;

   For each released seat:
   → Decrement sold/held counter, increment available counter
   → Publish event: { type: "seats_released", event_id, seat_ids }
   → Seat map cache invalidated for those sections
   → Waiting users may see new availability
```

---

## 8. Booking Flow — End to End

```
User Journey (Timeline):

t=0s     User joins queue → position 148,392 → "~12 min wait"
t=10m    Admitted through queue → redirected to seat selection page
t=10m05s User sees seat map (cached, 2s TTL) → selects 2 seats in Section A
t=10m06s POST /hold → Redis SET NX (1ms) → DB UPDATE (5ms) → 201 Hold acquired
         "You have 7 minutes to complete purchase."
t=10m30s User enters payment info
t=10m45s POST /bookings → Booking Service orchestrates:

         ┌─────────────────────────────────────────────────────┐
         │              Booking Saga                           │
         │                                                     │
         │  Step 1: Validate hold (still active? not expired?) │
         │          Redis GET hold:{event_id}:{seat_id}        │
         │          ✓ Hold valid (5:15 remaining)              │
         │                                                     │
         │  Step 2: Create booking record (status = PENDING)   │
         │          INSERT INTO bookings (...)                 │
         │                                                     │
         │  Step 3: Authorize payment (hold funds on card)     │
         │          POST /payments/authorize                   │
         │          → Stripe: PaymentIntent.create(            │
         │              amount: 3000, capture_method: manual)  │
         │          ✓ Authorized: pi_abc123                    │
         │                                                     │
         │  Step 4: Confirm seats (HELD → SOLD, atomic)        │
         │          UPDATE seats SET status = 'SOLD',          │
         │            booking_id = bk_001                      │
         │          WHERE seat_id IN (...) AND status = 'HELD' │
         │            AND held_by = user_42                    │
         │          ✓ 2 rows affected (both seats confirmed)   │
         │                                                     │
         │  Step 5: Capture payment (actually charge the card) │
         │          POST /payments/capture                     │
         │          → Stripe: PaymentIntent.capture(pi_abc123) │
         │          ✓ Captured                                 │
         │                                                     │
         │  Step 6: Update booking status → CONFIRMED          │
         │          Release Redis holds (no longer needed)     │
         │          DEL hold:{event_id}:{seat_id}              │
         │                                                     │
         │  Step 7: Issue tickets (async via Kafka)            │
         │          Generate QR codes, send confirmation email │
         │                                                     │
         └─────────────────────────────────────────────────────┘

t=10m48s  User sees: "Booking Confirmed! 2 tickets for World Cup Final."
          Email arrives within 30 seconds with tickets attached.

Total booking time: ~3 seconds (Steps 1-6)
```

### Saga Compensations (When Things Fail)

```
If Step 3 fails (payment declined):
  → Release hold: DEL Redis keys + UPDATE seats SET status = 'AVAILABLE'
  → Update booking: status = 'PAYMENT_FAILED'
  → Notify user: "Payment declined. Your seats have been released."

If Step 4 fails (seats no longer held — hold expired between Step 3 and Step 4):
  → Void payment authorization: Stripe PaymentIntent.cancel(pi_abc123)
  → Update booking: status = 'HOLD_EXPIRED'
  → Notify user: "Your hold expired. Please try again."
  → No charge to user's card.

If Step 5 fails (payment capture fails after seats confirmed):
  → This is the DANGEROUS case. Seats are SOLD in DB but payment not captured.
  → Do NOT release seats immediately.
  → Queue background retry: attempt capture every 30 seconds for 5 minutes.
  → If all retries fail: manual review queue (ops team decides).
  → The user sees "Booking confirmed, payment processing."
  → In practice: capture failure after successful auth is extremely rare (<0.01%).
```

---

## 9. Payment Integration

### Auth-Hold / Capture Pattern

```
Why not just charge immediately?

Problem: If we charge first and then seat confirmation fails (seat was sold between
hold and confirm due to a race), we must refund. Refunds take 5-10 business days.
User sees: "Charged $3000, booking failed, refund in 5-10 days." → Rage.

Auth-Hold / Capture:
  1. Auth-hold: "Reserve $3000 on the card. Don't actually charge."
     → Card shows "pending" charge. Funds reserved but not taken.
     → If booking fails: cancel auth. Pending charge disappears. No refund needed.
     
  2. Capture: "OK, actually take the $3000."
     → Only called AFTER seats are confirmed in DB.
     → At this point, booking is guaranteed to succeed.

Timeline:
  Authorization: ~500ms (Stripe API call)
  Capture: ~300ms (Stripe API call)
  Auth-hold valid for: 7 days (card networks allow up to 7 days before auto-void)
  In our system: capture happens within 3 seconds of auth. No risk of expiry.
```

### Idempotent Payments

```
Every payment request includes an idempotency key:

  POST /v1/payments/authorize
  Headers:
    X-Idempotency-Key: booking_bk001_auth_v1

  If network failure → retry with same key → Stripe returns same result.
  Impossible to double-charge.

Idempotency key format: {booking_id}_{operation}_{version}
  - booking_bk001_auth_v1
  - booking_bk001_capture_v1
  - booking_bk001_refund_v1
```

---

## 10. Order Lifecycle & State Machine

```
┌──────────┐  hold acquired  ┌──────────────────┐  payment auth  ┌───────────────────┐
│  (start) │────────────────→│  HOLD_ACQUIRED   │──────────────→ │ PAYMENT_AUTHORIZED│
└──────────┘                 └────────┬─────────┘                └─────────┬─────────┘
                                      │                                    │
                               hold expires /                       seats confirmed
                               user cancels                        + payment captured
                                      │                                    │
                                      ▼                                    ▼
                             ┌─────────────────┐                  ┌────────────────┐
                             │    CANCELLED    │                  │   CONFIRMED    │
                             └─────────────────┘                  └───────┬────────┘
                                                                          │
                                                                   user requests
                                                                   cancellation
                                                                          │
                                                                          ▼
                                                                  ┌────────────────┐
                                                                  │   REFUNDING    │
                                                                  └───────┬────────┘
                                                                          │
                                                                   refund processed
                                                                          │
                                                                          ▼
                                                                  ┌────────────────┐
                                                                  │    REFUNDED    │
                                                                  └────────────────┘

  Error states:
    PAYMENT_AUTHORIZED → PAYMENT_FAILED   (capture failed, seats released)
    HOLD_ACQUIRED      → HOLD_EXPIRED     (7-minute timer, seats released)
    Any state          → SYSTEM_ERROR     (unrecoverable, manual review)
```

---

## 11. Ticket Delivery & Validation

### Ticket Generation (Post-Booking, Async)

```
Booking confirmed → Kafka event → Ticket Service:

1. Generate unique ticket payload:
   {
     ticket_id: "T-123456",
     event_id: 999,
     seat: "Section A, Row 5, Seat 10",
     holder: "John Smith",
     booking_id: "BK-001",
     issued_at: "2026-12-01T10:15:00Z",
     signature: HMAC-SHA256(payload, secret_key)
   }

2. Encode as QR code (contains signed payload)

3. Encode as barcode (contains ticket_id for fallback lookup)

4. Store in tickets table

5. Deliver:
   - Email: PDF with QR code + event details
   - Push notification: deep-link to in-app ticket
   - App: ticket stored locally for offline access
```

### Gate Validation (Event Day)

```
Scanner reads QR code at venue gate:

1. Decode QR → extract payload + signature
2. Verify HMAC signature (offline-capable, no network needed)
3. Check ticket_id against local DB snapshot (pre-loaded to gate devices)
4. If valid + not already used:
   → Mark as USED (sync to central DB when connected)
   → Green light, person enters
5. If already used:
   → Red light, alert security (duplicate / fraud)
6. If invalid signature:
   → Red light (forged ticket)

Offline capability: Gate devices have a pre-synced snapshot of all 80,000 tickets.
  Even if venue internet goes down, gates continue operating.
  Used-ticket marks sync back when connectivity restores.
```

---

## 12. Anti-Bot & Fraud Prevention

### Defense in Depth

```
Layer 1: WAF + Rate Limiting (Infrastructure)
  │  IP rate limits, geographic filtering, known bot IP blocklists
  │  Blocks: ~60% of bot traffic
  ▼
Layer 2: CAPTCHA (Queue Join)
  │  Invisible reCAPTCHA v3 + interactive challenge for low scores
  │  Blocks: ~25% of remaining bots
  ▼
Layer 3: Device Fingerprinting (Queue + Booking)
  │  Canvas fingerprint, WebGL, installed fonts, screen resolution
  │  One position per unique device
  │  Blocks: multi-account-from-same-device attacks
  ▼
Layer 4: Behavioral Analysis (During Queue Wait)
  │  Mouse movement entropy, keystroke dynamics, page interaction patterns
  │  Pure bots: zero entropy → flagged and deprioritized
  ▼
Layer 5: Velocity Checks (Booking)
  │  Max N hold attempts per user per minute
  │  Max N bookings per user per event (e.g., max 4 tickets)
  │  Max N bookings per payment method
  ▼
Layer 6: Post-Booking Fraud Review (Async)
     Orders from suspicious sessions held for 30-second review.
     ML model flags: datacenter IP, multiple bookings same payment,
     known reseller patterns.
     Flagged orders: manually reviewed or auto-cancelled with refund.
```

### Ticket Limit Enforcement

```
Policy: Max 4 tickets per person per event.

Enforcement across dimensions:
  1. Per user_id: SELECT COUNT(*) FROM bookings WHERE user_id=? AND event_id=? AND status='CONFIRMED'
  2. Per email: Same check by email (catches multiple accounts, same person)
  3. Per payment method: Same check by payment_method_id
  4. Per device fingerprint: Same check by device_fp
  5. Per IP address (loose): Flag if > 10 bookings from same IP (shared office is OK)

  If ANY dimension exceeds limit → block booking, release hold.
```

---

## 13. Notifications & Communication

```
┌──────────────────────────┬────────────────────────────────────────┐
│ Event                    │ Channels                               │
├──────────────────────────┼────────────────────────────────────────┤
│ Queue position update    │ In-app only (via polling response)     │
│ Admitted to booking page │ Push notification + SMS                │
│ Hold acquired            │ In-app only                            │
│ Hold expiring (2 min)    │ In-app + push                          │
│ Booking confirmed        │ Email + push + SMS                     │
│ Payment failed           │ Email + push                           │
│ Tickets available        │ Email (PDF) + push (deep link)         │
│ Event reminder (24h)     │ Push + email                           │
│ Event cancelled/changed  │ Email + SMS + push (all channels)      │
│ Refund processed         │ Email                                  │
└──────────────────────────┴────────────────────────────────────────┘
```

---

## 14. Search & Event Discovery

```
  ┌─────────────────┐
  │  PostgreSQL     │  (source of truth)
  │  events table   │
  └────────┬────────┘
           │ CDC (Debezium)
           ▼
  ┌─────────────────┐
  │     Kafka       │
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │ Elasticsearch   │
  │                 │
  │ Indexed fields: │
  │  - event name   │
  │  - artist/team  │
  │  - venue name   │
  │  - city/country │
  │  - date range   │
  │  - category     │
  │  - price range  │
  │  - availability │
  │    (updated     │
  │     every 60s)  │
  └─────────────────┘

Search API:
  GET /v1/search?q=world+cup+final&city=new+york&date=2026-12&min_price=100&max_price=2000
  → Full-text search + filters + geo-distance + date range
  → Results ranked by relevance + date proximity + popularity
```

---

## 15. Database & Storage Architecture

### Per-Event Database Isolation

```
This is the most critical architectural decision.

Normal approach: All events share the same database.
  Problem: World Cup Final sale at 10 AM generates 500K writes/sec.
           This saturates the shared DB.
           Users trying to book a local theater show can't either.
           One event's traffic kills the entire platform.

Production approach: Per-event database isolation for mega-events.

                    ┌─────────────────────────────┐
                    │     Routing Layer           │
                    │  event_id → database        │
                    └──────────┬──────────────────┘
                               │
              ┌────────────────┼───────────────┐
              │                │               │
    ┌─────────▼──────┐ ┌──────▼────────┐ ┌─────▼───────────┐
    │  Mega-Event DB │ │  Normal Pool  │ │  Mega-Event DB  │
    │  (World Cup    │ │  (all other   │ │  (Super Bowl    │
    │   Final)       │ │   events)     │ │   LVIII)        │
    │                │ │               │ │                 │
    │  - Dedicated   │ │  - Shared     │ │  - Dedicated    │
    │    PostgreSQL  │ │    across     │ │    PostgreSQL   │
    │  - 3 replicas  │ │    1000s of   │ │  - 3 replicas   │
    │  - Provisioned │ │    events     │ │  - Provisioned  │
    │    for 10K QPS │ │  - Standard   │ │    for 10K QPS  │
    │  - Spun up 24h │ │   provisioning│ │  - Spun up 24h  │
    │    before sale │ │               │ │    before sale  │
    │  - Torn down   │ │               │ │  - Torn down    │
    │    after sale  │ │               │ │    after sale   │
    └────────────────┘ └───────────────┘ └─────────────────┘

Benefits:
  ✓ World Cup sale cannot affect other events
  ✓ Database provisioned exactly for expected load
  ✓ Can be in the region closest to majority of buyers
  ✓ Connection pools tuned per event
  ✓ After sale: tear down, save cost

The seats table is partitioned by event_id anyway,
so moving one partition to a dedicated server is straightforward:
  pg_dump the seats partition → restore on dedicated server → update routing
```

### PostgreSQL Topology (Per Mega-Event DB)

```
  ┌──────────────────────────────────────────┐
  │          Mega-Event DB (World Cup)       │
  │                                          │
  │   Primary (us-east-1a)                   │
  │     ├── Sync Standby (us-east-1b)   HA   │
  │     ├── Read Replica 1 (us-east-1c) ←──  │ seat map reads
  │     ├── Read Replica 2 (us-east-1c) ←──  │ seat map reads
  │     └── Async Replica (eu-west-1a)  DR   │
  │                                          │
  │   Writes: ~2K/sec (holds + confirmations)│
  │   Reads: ~200K/sec (seat availability)   │
  │     → 99% served from Redis cache        │
  │     → 1% reach read replicas             │
  │     → 0% reach primary for reads         │
  └──────────────────────────────────────────┘
```

---

## 16. Caching Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Caching Layers                                   │
│                                                                         │
│  Layer 0: CDN (90%+ of initial traffic)                                 │
│    - Event listing pages (5-minute TTL)                                 │
│    - Venue seat map images / SVG (24-hour TTL)                          │
│    - Static JS/CSS assets (fingerprinted, indefinite)                   │
│    - Queue waiting room page (static HTML, cached indefinitely)         │
│                                                                         │
│    KEY INSIGHT: The waiting room page is the FIRST thing millions see.  │
│    It must be served from CDN edge, NOT from your servers.              │
│    Your servers don't even see the load until users are admitted.       │
│                                                                         │
│  Layer 1: Redis — Real-Time State (~2 TB for mega-events)               │
│    ┌──────────────────────────────┬────────┬───────────────────┐        │
│    │ Key Pattern                  │ Size   │ TTL               │        │
│    ├──────────────────────────────┼────────┼───────────────────┤        │
│    │ queue:{event_id}             │ 800 MB │ event duration    │        │
│    │ (sorted set, 10M members)    │        │                   │        │
│    ├──────────────────────────────┼────────┼───────────────────┤        │
│    │ hold:{event_id}:{seat_id}    │ 50 MB  │ 420s              │        │
│    │ (80K possible keys)          │        │                   │        │
│    ├──────────────────────────────┼────────┼───────────────────┤        │
│    │ avail:{event_id}:{section}   │ 1 MB   │ 2s                │        │
│    │ (section availability count) │        │                   │        │
│    ├──────────────────────────────┼────────┼───────────────────┤        │
│    │ seatmap:{event_id}:{section} │ 500 MB │ 1s                │        │
│    │ (per-seat status for section)│        │                   │        │
│    ├──────────────────────────────┼────────┼───────────────────┤        │
│    │ rl:{user_id}:{event_id}      │ 200 MB │ 60s               │        │
│    │ (rate limiter per user)      │        │                   │        │
│    ├──────────────────────────────┼────────┼───────────────────┤        │
│    │ bot_score:{session_id}       │ 100 MB │ 3600s             │        │
│    └──────────────────────────────┴────────┴───────────────────┘        │
│                                                                         │
│  Layer 2: Application In-Memory Cache                                   │
│    - Rate card / pricing tiers (changes rarely, cached per pod)         │
│    - Venue metadata (static)                                            │
│    - CAPTCHA configuration                                              │
│                                                                         │
│  Layer 3: Database (only for writes + cache misses)                     │
│    - Seat holds / confirmations (writes)                                │
│    - Booking records (writes)                                           │
│    - Historical queries (past bookings for user)                        │
└─────────────────────────────────────────────────────────────────────────┘
```

### Seat Map Cache Invalidation

```
Problem: User sees seat as "available" on their screen (cached view).
  They click it. By the time the request arrives, it's already held.
  
This is EXPECTED and acceptable — the response is simply 409 Conflict.
The UI handles this gracefully: "This seat was just taken. Here are nearby alternatives."

Cache strategy: Aggressive short-TTL caching + optimistic UI.

  Section-level counts: 2-second TTL (cheap to serve, slightly stale)
  Per-seat status:      1-second TTL (heavier, more fresh)
  
  Write-through on hold/confirm:
    When a seat is held → DECR avail:{event_id}:{section}
    When a hold expires → INCR avail:{event_id}:{section}
  
  Periodic reconciliation (every 10 seconds):
    SELECT section_id, COUNT(*) FILTER (WHERE status = 'AVAILABLE') FROM seats
    WHERE event_id = 999 GROUP BY section_id;
    
    Overwrite Redis counters with DB truth.
    This fixes any drift from lost events, crashes, or race conditions.
```

---

## 17. Scalability Deep Dive

### Traffic Phases & Scaling Strategy

```
Phase 1: Pre-Sale (days before)
  Traffic: ~5K QPS (browsing, event details)
  Strategy: Normal capacity. CDN handles most of it.

Phase 2: Queue Open (sale start time)
  Traffic: 500K QPS → 10M users in 30 seconds
  Strategy:
    - CDN serves the waiting room page (zero backend load for page views)
    - Queue join: 10M writes to Redis sorted set (~333K/sec for 30s)
      → Single Redis cluster handles this easily
    - Queue status polling: mostly client-side logic (see Section 6)
    - Backend: minimal load (only queue management)

Phase 3: Admission & Booking (1-15 minutes)
  Traffic: ~200 admitted users/sec × ~4 operations each = ~800 backend ops/sec
  Strategy:
    - Booking service: 20 pods (each handles ~40 bookings/sec)
    - PostgreSQL: dedicated mega-event DB with read replicas
    - Redis: seat locks at ~500 SET NX/sec (trivial)
    - Payment gateway: ~133 charges/sec (within Stripe's limits)

Phase 4: Sold Out (after ~10-15 minutes)
  Traffic: drops 99% immediately
  Strategy: Scale down aggressively. Tear down mega-event DB within hours.

        500K QPS
          │╲
          │ ╲
          │  ╲
          │   ╲____
          │        ╲_______________
          │                        ╲___________
    5K QPS│                                    ╲______ → baseline
          └──────────────────────────────────────────→ time
          sale     1 min    5 min    15 min    1 hour
          opens
```

### Handling Multiple Simultaneous Mega-Events

```
Problem: Three mega-events on sale at the same time.
  World Cup Final (10M users), Super Bowl (8M users), Taylor Swift Tour (5M users).

Solution: Per-event isolation (already designed in Section 15).
  Each event has its own:
    - Redis namespace (queue, holds, cache)
    - PostgreSQL database (or partition)
    - Booking service deployment (separate pods, separate resource quotas)
    - Queue with independent admission rate

  Events share:
    - CDN (infinite scale)
    - API Gateway (stateless, auto-scales)
    - Payment gateway (single account, but high-volume plan)
    - Notification infrastructure (Kafka + email/SMS providers)

  Kubernetes resource quotas:
    namespace: event-worldcup
      cpu: 200 cores reserved
      memory: 400 GB reserved
    namespace: event-superbowl
      cpu: 160 cores reserved
      memory: 320 GB reserved
    namespace: event-taylorswift
      cpu: 100 cores reserved
      memory: 200 GB reserved

  One event's traffic cannot steal resources from another.
```

---

## 18. Reliability & Fault Tolerance Deep Dive

### Failure Scenarios

```
┌──────────────────────────────┬───────────────────────────────────────────────────┐
│ Failure                      │ Impact & Mitigation                               │
├──────────────────────────────┼───────────────────────────────────────────────────┤
│ Redis (queue) goes down      │ CRITICAL: Users can't join queue.                 │
│                              │ Mitigation:                                       │
│                              │  - Redis Sentinel auto-failover (< 10s)           │
│                              │  - Standby Redis pre-loaded with queue snapshot   │
│                              │  - During gap: CDN serves "please wait" page      │
│                              │  - Queue state recoverable from Kafka events      │
├──────────────────────────────┼───────────────────────────────────────────────────┤
│ Redis (seat locks) goes down │ CRITICAL: Can't hold seats.                       │
│                              │ Mitigation:                                       │
│                              │  - Fall back to PostgreSQL FOR UPDATE locks       │
│                              │  - Slower (50ms vs 1ms) but correct               │
│                              │  - Auto-switchback when Redis recovers            │
│                              │  - All existing holds still valid in DB           │
│                              │    (Redis was an optimization, DB is truth)       │
├──────────────────────────────┼───────────────────────────────────────────────────┤
│ PostgreSQL primary failover  │ 10-30 second write outage.                        │
│                              │ During failover:                                  │
│                              │  - Seat holds via Redis still work (user can      │
│                              │    select seats)                                  │
│                              │  - Booking confirmation writes fail → retry       │
│                              │  - Users see "confirming your booking..." for     │
│                              │    10-30 seconds longer                           │
│                              │  - No data loss (sync replication to standby)     │
│                              │  - After promotion: all queued writes flush       │
├──────────────────────────────┼───────────────────────────────────────────────────┤
│ Payment gateway down         │ Users can hold seats but can't pay.               │
│ (Stripe outage)              │ Mitigation:                                       │
│                              │  - Multi-gateway: failover to Braintree/Adyen     │
│                              │  - If all gateways down:                          │
│                              │    · Extend hold timers from 7 min to 15 min      │
│                              │    · Show: "Payment systems temporarily slow.     │
│                              │      Your seats are reserved."                    │
│                              │    · Queue background retry every 30 seconds      │
│                              │  - Seats remain held (not released) during outage │
│                              │  - When gateway recovers: complete all pending    │
│                              │    payments in order                              │
├──────────────────────────────┼───────────────────────────────────────────────────┤
│ CDN failure (edge PoP)       │ Waiting room page not served from that PoP.       │
│                              │ CDN auto-routes to next-closest PoP.              │
│                              │ Users in that region: extra 50-100ms latency.     │
│                              │ No functional impact.                             │
├──────────────────────────────┼───────────────────────────────────────────────────┤
│ Booking service pod crash    │ In-flight bookings on that pod fail.              │
│ (mid-saga)                   │ Saga recovery:                                    │
│                              │  - Hold still exists in Redis (TTL-based, safe)   │
│                              │  - If payment was authorized but not captured:    │
│                              │    background recovery job picks it up            │
│                              │  - If nothing was committed: hold expires → seat  │
│                              │    released → user can retry                      │
│                              │  - Idempotency keys prevent double-booking        │
├──────────────────────────────┼───────────────────────────────────────────────────┤
│ Kafka broker failure         │ Event processing delayed (notifications, tickets).│
│                              │ Booking flow NOT affected (synchronous path       │
│                              │ doesn't depend on Kafka).                         │
│                              │ Ticket delivery delayed by seconds to minutes.    │
│                              │ No data loss (replication factor 3).              │
├──────────────────────────────┼───────────────────────────────────────────────────┤
│ Clock skew between servers   │ Hold expiry drift: hold timer could be ±2s off.   │
│                              │ Redis TTL is absolute (server clock-independent). │
│                              │ DB held_until checked against DB's NOW() (same    │
│                              │ clock). No cross-server clock comparison needed.  │
│                              │ Redis and DB may disagree by ~2s on expiry →      │
│                              │ Redis releases lock, DB still shows held.         │
│                              │ Reconciliation job fixes within 30 seconds.       │
└──────────────────────────────┴───────────────────────────────────────────────────┘
```

### Data Consistency Guarantees

```
┌───────────────────┬──────────────┬────────────────────────────────────────────┐
│ Data              │ Consistency  │ Why                                        │
├───────────────────┼──────────────┼────────────────────────────────────────────┤
│ Seat status       │ STRONG       │ Overselling is unacceptable.               │
│ (available/held/  │ (serialized  │ Two users must never book the same seat.   │
│  sold)            │  via Redis   │ Redis SET NX provides atomic exclusion.    │
│                   │  + DB version│ DB version column is the safety net.       │
│                   │  check)      │                                            │
├───────────────────┼──────────────┼────────────────────────────────────────────┤
│ Booking / Payment │ STRONG       │ Financial transaction. Must not charge     │
│                   │              │ without confirmed seats. Must not confirm  │
│                   │              │ seats without successful payment.          │
├───────────────────┼──────────────┼────────────────────────────────────────────┤
│ Queue position    │ STRONG       │ Fairness. Users must be served in order.   │
│                   │ (within      │ Redis sorted set guarantees ordering.      │
│                   │  Redis)      │ No "queue jumping" possible.               │
├───────────────────┼──────────────┼────────────────────────────────────────────┤
│ Seat availability │ EVENTUAL     │ Section-level counts can be 1-2 seconds    │
│ (display counts)  │ (2s stale)   │ stale. Users see "~3200 available" not     │
│                   │              │ exact count. Acceptable.                   │
├───────────────────┼──────────────┼────────────────────────────────────────────┤
│ Notifications     │ EVENTUAL     │ Email arriving 30s late is fine.           │
│                   │ (seconds)    │                                            │
├───────────────────┼──────────────┼────────────────────────────────────────────┤
│ Ticket delivery   │ EVENTUAL     │ Ticket email within 5 minutes of booking   │
│                   │ (minutes)    │ is acceptable. Event is weeks away.        │
└───────────────────┴──────────────┴────────────────────────────────────────────┘
```

### Exactly-Once Booking Guarantee

```
Problem: User clicks "Confirm Booking" → network timeout → user clicks again.
  Without protection: two bookings created, charged twice, same seats.

Solution: Client-generated idempotency key.

  Client generates a UUID when the booking page loads:
    idempotency_key = "ik_" + uuid4()

  Every subsequent POST /bookings includes this key.

  Server:
    1. Check: SELECT 1 FROM bookings WHERE idempotency_key = 'ik_abc123'
    2. If exists: return the existing booking (no new side effects)
    3. If not: proceed with booking, store the key

  Even if the user:
    - Clicks twice rapidly → same key → second request returns existing booking
    - Refreshes the page → same key from JS storage → same behavior
    - Opens new tab → new key → new booking attempt (but old hold may block seats)
    
  Combined with seat holds (only one user can hold a seat):
    → Impossible to double-book. Impossible to double-charge.
```

---

## 19. Observability & Operational Excellence

### War Room Dashboard (During Mega-Event Sale)

```
┌─────────────────────────────────────────────────────────────────────┐
│                 WORLD CUP FINAL — SALE LIVE                         │
│                 Started: 10:00:00 AM | Elapsed: 4m 32s              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  QUEUE                          INVENTORY                           │
│  ├─ In queue:        8,234,102  ├─ Total seats:    80,000           │
│  ├─ Admitted so far:   42,000   ├─ Sold:           31,247   (39%)   │
│  ├─ Admission rate:    200/sec  ├─ Held:           12,483   (16%)   │
│  ├─ Est. sellout:      ~8 min   ├─ Available:      36,270   (45%)   │
│  └─ Bots blocked:      14,291   └─ Blocked:            0            │
│                                                                     │
│  BOOKINGS                       PAYMENTS                            │
│  ├─ Attempts/sec:       142     ├─ Auth rate:       142/sec        │
│  ├─ Success rate:       94.2%   ├─ Capture rate:    133/sec        │
│  ├─ Avg booking time:   2.8s    ├─ Decline rate:    3.1%           │
│  ├─ Hold expire rate:   4.1%    ├─ Gateway latency: p50: 450ms     │
│  └─ Conflict rate:      12.3%   └─                  p99: 1.2s      │
│                                                                    │
│  INFRASTRUCTURE                  ALERTS                            │
│  ├─ CDN QPS:        412K/sec    ├─ ✓ All systems nominal           │
│  ├─ API GW QPS:      12K/sec    ├─ ⚠ Hold expire rate above 4%     │
│  ├─ Redis ops/sec:   85K/sec    │   (extending hold timer to 8min) │
│  ├─ DB writes/sec:    1.8K     └─                                  │
│  ├─ DB read (replica): 2.1K                                        │
│  ├─ Kafka lag:          124 msgs                                   │
│  └─ Pod count:          booking: 20, queue: 8, inventory: 6        │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 20. Corner Cases & Hard Problems

### 1. The 10:00:00.000 Thundering Herd

```
Problem: Sale opens at exactly 10:00 AM. Millions of users have their finger
  on the button. Network time synchronization means they all fire within
  the same 100ms window.

  Without mitigation: 500K requests hit API gateway in 100ms.
  Even the queue join endpoint would be overwhelmed.

Solution: Jittered soft-open.

  1. CDN-level: The waiting room JS adds a random delay of 0-2000ms before
     calling POST /queue/join.
     
     // Client-side
     const jitter = Math.random() * 2000;
     setTimeout(() => joinQueue(), jitter);
     
     Result: 500K requests spread over 2 seconds = 250K/sec (manageable).

  2. Server-level: Queue join endpoint has its own admission rate.
     Even if 250K/sec arrive, backend processes 50K/sec and returns
     queue position. Remaining requests wait in API gateway queue
     (connection-level backpressure, not application failure).

  3. Pre-queue: Users can join the waiting room 30 minutes before sale.
     Position assigned immediately (random order for those who joined before sale start).
     At 10:00 AM: queue starts admitting. No thundering herd at all.
     This is what Ticketmaster uses.
```

### 2. User Holds Seats But Never Pays (Hold Hoarding)

```
Problem: Scalper bot joins queue legitimately, gets admitted.
  Holds 4 seats. Starts 7-minute timer.
  At 6:59, releases seats and immediately holds 4 different seats.
  Repeats. Prevents real users from buying.

Detection:
  - Track hold-then-release patterns per user:
    rl_hold_release:{user_id}:{event_id} → count of releases without purchase
  - If releases > 3 in 15 minutes → flag as hoarding

Mitigation:
  - After 2nd hold release without purchase: cooldown period (5 minutes before
    next hold attempt)
  - After 3rd: banned from holding for this event. Must select "Best Available"
    (auto-assigned, no browsing)
  - Max 2 hold attempts per user per event (after first expires without booking)
```

### 3. Seat Map Shows Available, But Hold Fails (Stale Cache Race)

```
Problem: User sees Section A seat R5-S10 as green (available) on the seat map.
  Cache was 1.5 seconds old. Another user held it 0.5 seconds ago.
  User clicks → POST /hold → 409 Conflict.

This is EXPECTED at high concurrency. Not a bug.

UX solution: Graceful degradation.

  1. 409 response includes suggested alternatives:
     { "error": "seat_unavailable",
       "alternatives": [
         { "seat_id": "A1-R5-S12", "price": 1500.00 },
         { "seat_id": "A1-R5-S13", "price": 1500.00 },
         { "seat_id": "A1-R6-S10", "price": 1500.00 }
       ]
     }
     
  2. Client shows: "This seat was just taken! Here are nearby seats."
     One click → try alternative → likely succeeds.

  3. For extremely hot sections: disable per-seat selection.
     Show: "Best available in Section A" → server picks seats.
     Eliminates stale-cache race entirely.
     This is what most mega-event sales actually do.
```

### 4. Payment Gateway Timeout Mid-Booking

```
Problem: Auth-hold sent to Stripe. No response for 10 seconds. Timeout.
  Did the charge go through? We don't know.

  If we retry: might double-charge (Stripe may have processed the first).
  If we don't retry: user might not get tickets (auth may have succeeded).

Solution: Idempotency key + eventual reconciliation.

  1. Retry with same idempotency key → Stripe returns original result (no duplicate)
  
  2. If retry also times out:
     - Mark booking as PAYMENT_UNCERTAIN
     - Do NOT release the seat hold (seats stay reserved)
     - Background job queries Stripe: GET /v1/payment_intents/{pi_abc123}
     - Three outcomes:
       a. Auth succeeded → proceed with capture + confirm
       b. Auth failed → release seats, notify user
       c. Not found → create new auth with new idempotency key
     
  3. User sees: "Confirming your payment... This may take a moment."
     Not ideal, but correct. No double charge. No lost booking.
```

### 5. Entire Section Sells Out Mid-Selection

```
Problem: User is browsing seats in Section A. While they browse,
  all remaining 200 seats in Section A are sold.
  User selects a seat → 409 → selects another → 409 → frustration.

Solution: Real-time availability push.

  1. While user is on the seat selection page, maintain an SSE connection:
     GET /v1/events/{event_id}/availability/stream
     
  2. Server pushes section availability changes:
     data: {"section":"A","available":0,"status":"SOLD_OUT"}
     
  3. Client immediately greys out Section A and highlights sections
     with remaining availability.

  4. If ALL sections sell out:
     Push: data: {"event_status":"SOLD_OUT"}
     Client shows: "This event is now sold out."
     Option: "Join waitlist" (in case of cancellations).
```

### 6. Coordinated Bot Attack (Sophisticated Scalping)

```
Problem: Professional scalping operation:
  - 500 residential proxy IPs (not datacenter IPs)
  - 500 different accounts with real phone numbers
  - Human-like mouse movements (ML-generated)
  - Pass CAPTCHA using human solver farms (2captcha.com)
  - Each account buys 4 tickets (within limit) = 2,000 tickets hoarded

  This passes all automated defenses.

Layered response:

  1. Post-sale analysis:
     - ML model clusters bookings by behavioral similarity
     - 500 bookings all completed within 30 seconds, all from similar
       browser fingerprints, all with sequential IP ranges
     - Flagged for review

  2. Identity verification:
     - Random 10% of bookings require photo ID upload within 24 hours
     - ID must match name on booking
     - Scalpers can't provide 500 unique matching IDs

  3. Named tickets:
     - Ticket holder name printed on ticket (non-transferable)
     - ID check at venue gate
     - Scalpers can't resell named tickets

  4. Official resale marketplace:
     - Allow ticket transfers at face value through official platform
     - Undercuts scalper economics (why pay scalper 3x when official resale
       has tickets at 1x?)

  5. Dynamic pricing:
     - If demand is 10x supply: price tickets higher to match market value
     - Eliminates scalper profit margin (they can't resell above face value
       if face value already reflects demand)
```

### 7. Hold Timer Drift Between Redis and PostgreSQL

```
Problem: Redis TTL says hold expires in 420 seconds.
  DB held_until says hold expires at NOW() + 7 minutes.
  Redis clock and DB clock may differ by 1-2 seconds.

  Redis expires the lock at t=420s.
  DB still shows held_until = t+422s (clock skew).
  
  In the 2-second gap:
  - Redis: seat is unlockable (key expired)
  - DB: seat still shows HELD (held_until hasn't passed)
  
  New user tries to hold:
  - Redis SET NX succeeds (key was expired)
  - DB UPDATE fails (status = 'HELD', not 'AVAILABLE')
  - Inconsistency.

Solution: Redis is the LOCK, DB is the RECORD.

  - Redis TTL is the authoritative expiry for the lock.
  - DB held_until is informational (for background cleanup).
  - On new hold attempt:
    1. Redis SET NX → success (we have the lock)
    2. DB UPDATE WHERE status IN ('AVAILABLE', 'HELD') AND
       (held_until IS NULL OR held_until < NOW())
       → This UPDATE accepts both AVAILABLE and expired-HELD rows.
  - Background job periodically:
    UPDATE seats SET status = 'AVAILABLE', held_by = NULL
    WHERE status = 'HELD' AND held_until < NOW() - INTERVAL '30 seconds'
    (30-second buffer absorbs clock skew)
```

### 8. Sale Starts But Event Gets Cancelled Mid-Sale

```
Problem: World Cup Final sale is live. 40,000 tickets already sold.
  Breaking news: event cancelled due to security threat.
  Must stop selling immediately and refund all buyers.

Response:

  1. Emergency stop (< 10 seconds):
     - Set event status = 'CANCELLED' in DB + Redis
     - Queue service: stop admitting, push "event cancelled" to all in queue
     - Booking service: reject all new POST /hold and POST /bookings
     - Seat map: show "EVENT CANCELLED" banner
     
  2. In-flight bookings (tricky):
     - Bookings in HOLD_ACQUIRED: release holds, do nothing (no charge)
     - Bookings in PAYMENT_AUTHORIZED but not captured:
       void all payment authorizations (no charge to user)
     - Bookings in CONFIRMED: initiate refund for all
     
  3. Mass refund (40,000 refunds):
     - Queue all refunds via Kafka: { booking_id, amount, payment_intent }
     - Process at 100/sec (payment gateway rate limit) → ~7 minutes
     - Send notification per user: "Event cancelled. Full refund processing."
     
  4. Post-cancellation:
     - All tickets invalidated (status = CANCELLED)
     - QR codes deactivated (gate scanners won't accept)
     - Event page shows cancellation notice
```

### 9. Double-Booking Across Regions (Multi-Region Active-Active)

```
Problem: If the booking system runs active-active in two regions:
  - US-East user and EU-West user both hold the same seat
  - Each region's Redis says the hold succeeded (local SET NX)
  - When writes replicate cross-region: conflict

This is why ticket booking should NOT be active-active for writes.

Architecture rule: For a given event, there is exactly ONE write region.

  World Cup Final (venue in US):
    Write region: US-East (closest to venue, lowest latency for majority of buyers)
    EU-West: read replicas only (seat map display, event details)
    EU-West booking requests: routed to US-East for write operations
    
  Extra latency for EU users: +80-120ms (cross-Atlantic round trip)
  But: correctness is guaranteed. No split-brain. No double booking.
  
  This 100ms penalty is invisible to humans (total booking is ~3s anyway).
```

---

## Summary: Key Design Decisions

| Decision | Choice | Why |
|---|---|---|
| Traffic control | Virtual waiting room + controlled admission | 10M users can't all hit booking; queue absorbs stampede |
| Queue fairness | Redis sorted set by join timestamp | O(1) position lookup, strict FIFO ordering |
| Seat locking | Redis SET NX (fast lock) + DB version check (durable truth) | Microsecond locking for stampede, millisecond DB for correctness |
| Multi-seat hold | Redis Lua script (atomic all-or-nothing) | Family of 4 must sit together or not at all |
| Payment pattern | Auth-hold → capture (not direct charge) | Failed booking = void auth (instant), not refund (5-10 days) |
| Booking flow | Saga with compensation steps | Each step reversible; idempotency prevents duplicates |
| Database isolation | Dedicated DB per mega-event | World Cup sale can't kill theater booking traffic |
| Seat availability | Redis cache (2s TTL) + DB reconciliation every 10s | 200K reads/sec from cache, not DB |
| Anti-bot | 6-layer defense: WAF → CAPTCHA → fingerprint → behavior → velocity → post-sale ML | No single defense is sufficient; layering catches >99% |
| Consistency | Strong for seats/payments, eventual for display counts | Can't oversell a seat; display count 2s stale is fine |
| Multi-region | Single write region per event | Correctness over latency; 100ms cross-region penalty acceptable |
| Hold expiry | Redis TTL (auto-expire, no cleanup job) + DB background sweep | Self-healing; no cron job single point of failure |
| Thundering herd | CDN waiting room + client-side jitter + pre-queue | Zero backend load for page views; spread join requests over seconds |





┌────────────────────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│      Section       │                                                                  Key Content                                                                  │
├────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Requirements &     │ What makes live streaming fundamentally different from VOD: content doesn't exist until produced, CDN is cold at stream start, everyone       │   
│ Scale              │ watches the SAME content, playback failure = missed moment                                                                                    │
├────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤   
│ Back-of-Envelope   │ 50M concurrent × 3 Mbps avg = ~112 Tbps egress. But CDN cache hit rate is 99.99% because all viewers want identical content — origin sees     │
│                    │ only ~3.5 req/sec. The problem is bandwidth, not compute.                                                                                     │   
├────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Architecture         │ Full pipeline: Stadium cameras → SRT ingest (dual-path) → GPU transcode (active-active) → HLS/DASH packager → Origin + Shield → Multi-CDN   │   
│                      │ (Akamai + CloudFront + ISP-embedded) → 50M viewers                                                                                          │   
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤   
│ API Design           │ Manifest serving (CDN redirect with signed token), admin stream control, chat/reactions WebSocket with aggregated responses, player         │   
│                      │ heartbeat telemetry                                                                                                                         │   
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤   
│ Data Models          │ PostgreSQL (events, streams, qualities), Redis (HyperLogLog viewer count in 12KB, reaction counters, session sorted sets, CDN routing       │   
│                      │ weights)                                                                                                                                    │   
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Video Ingestion     │ SRT over RTMP (UDP, FEC, loss-resilient), dual-path backhaul from stadium, frame validator (black frame / frozen frame / silence detection), │   
│                     │  wall-clock PTS alignment                                                                                                                    │   
├─────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Transcoding &       │ 7-rung ABR ladder (4K→audio-only) × 4 languages × 2 formats = 56 pipelines. Active-active encoder pairs with segment selector. 6s standard / │   
│ Packaging           │  2s LL-HLS partial segments.                                                                                                                 │   
├─────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Origin Server       │ RAM → NVMe → S3 tiered storage. Origin shield reduces origin load 1000x (5000 edge servers → 3 shields → 1 origin). Detailed shield topology │   
│                     │  diagram.                                                                                                                                    │   
├─────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ CDN Architecture    │ Multi-CDN (no single CDN has 150 Tbps). ISP-embedded caches inside Jio/Airtel networks — 80% of traffic never leaves the viewer's ISP.       │   
│                     │ Per-viewer CDN selection. Real-time CDN failover with player-side fallback URLs.                                                             │   
├─────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ ABR (Client-Side)   │ Buffer-based algorithm with 4 modes (emergency/conservative/normal/aggressive). India-specific network diversity table (urban WiFi to rural  │   
│                     │ 2G). Target buffering ratio < 0.1%.                                                                                                          │   
├─────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Low-Latency         │ Glass-to-glass breakdown: standard HLS ~16-17s, LL-HLS ~5-6s, CMAF chunked ~2-3s. Hotstar's sweet spot: 8-15s for cricket.                   │   
├─────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤   
│ DRM                 │ Multi-DRM (Widevine + FairPlay + PlayReady), key rotation every 5 min, license pre-fetch during loading screen, 50M license requests spread  │
│                     │ over pre-match period                                                                                                                        │   
├─────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Live Chat &         │ 5M reactions/sec impossible to broadcast individually. Kafka → Flink aggregation → 1 summary per 5 seconds cached on CDN → origin produces   │   
│ Reactions           │ ~1 response every 5 seconds. Feels alive, costs nothing.                                                                                     │   
├─────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ SSAI (Ad Insertion) │ Server-side splice: ad-blocker proof, no transition buffering. Per-viewer manifest rewriting. 50M personalized manifests grouped by (tier,   │   
│                     │ region, device) into ~50 cache-friendly groups.                                                                                              │   
├─────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ DVR / Rewind        │ 4-hour DVR window on NVMe SSD. Rewind thundering herd (key moment replay) handled by origin shield. Post-event VOD conversion to S3.         │   
├─────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤   
│ Auth & Concurrency │ HMAC-signed token validated at CDN edge (8.3M validations/sec, no network call). Concurrent device limit via Redis sorted set + heartbeat     │
│                    │ (1.67M writes/sec).                                                                                                                           │   
├────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Database & Storage │ Storage per data type: PostgreSQL (metadata), RAM→NVMe→S3 (segments), HSM-backed KMS (DRM keys), Kafka→Flink→ClickHouse (analytics), Redis    │   
│                    │ (real-time state)                                                                                                                             │   
├────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Caching            │ Live streaming's unique advantage: 99.99%+ cache hit rate because all 50M viewers want identical content. What ISN'T cacheable: manifests,    │   
├───────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤   
│ Scalability       │ Ramp-up from 5M→55M over 3 hours. What scales with viewers (manifests, DRM, heartbeats) vs what doesn't (origin, encoding, DB). Pre-warming    │
│                   │ strategy: T-24h to T-0 checklist.                                                                                                              │   
├───────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤   
│                   │ 10 failure scenarios with mitigations: camera failure, ingest link, encoder crash, origin crash, CDN provider down, ISP peering congestion,    │
│ Fault Tolerance   │ DRM server overload, regional power outage, manifest service failure, chat service failure. 4-tier graceful degradation (full → drop 4K → drop │   
│                   │  chat/DVR → audio-only) reducing load from 150 Tbps to ~40 Tbps.                                                                               │   
├───────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Observability     │ War room dashboard with real-time metrics: 51M concurrent, 0.08% buffering, 108 Tbps CDN bandwidth, quality distribution, per-CDN health. Key  │   
│                   │ SLIs: start time <2s, buffering <0.5%, latency <15s.                                                                                           │
├───────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤   
│                   │ 7 hard problems: match-start thundering herd (pre-stream loading + CDN pre-warming), viral wicket spike (5M cold starts in 60s, fast-start at  │
│ Corner Cases      │ 360p), cell tower congestion (ISP caches + audio-only fallback), corrupt encoder segment (VMAF quality gate + CDN purge), global time zone     │   
│                   │ overlap, mid-stream DRM key rotation (pre-delivery + extend-on-failure), "last ball" maximum tension scenario (59M concurrent + 2M cold starts │   
│                   │  + 10M reactions in 5s — the actual design target)                                                                                             │
└───────────────────┴────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘   