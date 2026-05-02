# Google Calendar — Staff-Level High-Level Design

A deep dive into how to design a globally-distributed calendaring platform: events, recurrences, attendees, free/busy, reminders, sharing, and federation across foreign calendar systems. The focus is on the **algorithms** (RRULE expansion, time-zone arithmetic, interval trees, find-a-time, hierarchical timing wheels, exactly-once reminder dispatch), the **corner cases** (DST, recurrence overrides, multi-device sync, RSVP races), and the **scalability / reliability / availability** posture you would expect at staff level.

Calendar looks deceptively simple. It is, in fact, one of the hardest distributed-systems problems Google operates: an unbounded event stream, time arithmetic riddled with cultural exceptions, exactly-once notification semantics, and federation with thousands of foreign systems that disagree about what a "Tuesday" is.

---

## 1. Requirements

### 1.1 Functional
1. **Events**: create, update, delete, RSVP to events with title, description, location, attendees, attachments, conferencing link, color, reminders, visibility.
2. **Recurrence**: RFC 5545 RRULE — daily / weekly / monthly / yearly patterns with arbitrary BY-rules, EXDATE exclusions, RDATE additions, RECURRENCE-ID overrides ("edit this and following").
3. **Multiple calendars** per user (personal, work, holidays, birthdays, sports), each with independent ACLs.
4. **Sharing**: per-calendar ACLs (free/busy / read / write / admin), per-event visibility (private / public / default).
5. **Free/busy**: query "is X busy between t1 and t2?" — the most-called read API.
6. **Find-a-time** ("Suggested Times"): given N attendees and a duration, find all open slots within a window.
7. **Reminders**: notify owner + attendees N minutes before; fan out via email, push notification, popup.
8. **Reservable resources**: meeting rooms, video equipment — must not double-book.
9. **Federation**: invitations to/from external systems (Outlook/Exchange, iCloud, CalDAV, public ICS feeds).
10. **Multi-device sync**: web, mobile, native macOS Calendar.app, Outlook plugin — every change propagates within seconds.
11. **Search** across event titles, descriptions, attendees.
12. **Working hours, focus time, out-of-office** — calendar-level metadata that influences scheduling.

### 1.2 Non-Functional
- **Scale**: ~3 B users, ~50 B active events, ~500 B materialized event instances after recurrence expansion (we won't materialize them — see §4.1).
- **Availability**: 99.99 % read, 99.95 % write. Reminders: 99.999 % deliverability (missed reminder = lost-trust-forever event).
- **Latency**: event read p99 < 80 ms; free/busy for ≤ 30 attendees p99 < 200 ms; reminder dispatch within ±2 s of target time at p99.
- **Consistency**: strong for resource booking (no double-booking) and for ACL changes; eventual elsewhere; read-your-writes for the owner.
- **Durability**: 11 nines (zero tolerance for losing an event).
- **Time correctness**: events fire at the *intended local wall-clock time* even if the IANA tz database changes after creation.

### 1.3 Capacity (back-of-envelope)
- 3 B users × ~200 events/year × 5 yr active history → ~3 T events stored.
- 100 M DAU × 30 reads/day → ~35 K event reads/sec sustained, ~250 K/sec peak.
- Free/busy lookups dominate: each "find a time" with 10 attendees fans into 10 free/busy queries; ~500 K/sec at peak.
- ~5 M reminders/sec at the top of the hour (everyone schedules 9:00, 10:00, 11:00…). This is the **single hardest rate** in the system.
- ~10 M notifications/sec fanout at top of the hour (reminder × email + push + popup × attendees).

---

## 2. High-Level Architecture

```
                    ┌─────────────────────────────────────────────┐
   Clients ────────▶│         Edge / API Gateway (Anycast)         │
 (web, iOS, Android,└────┬────────────────────────────────────────┘
  Calendar.app, Outlook) │   AuthN/Z, rate limit, schema validation
                         ▼
        ┌────────────────┬─────────────────┬─────────────────┬────────────────┐
        ▼                ▼                 ▼                 ▼                ▼
 ┌──────────────┐  ┌─────────────┐  ┌────────────┐   ┌──────────────┐  ┌─────────────┐
 │   Event      │  │ Free/Busy   │  │ Scheduling │   │ Reservation  │  │  ACL /      │
 │   Service    │  │   Service   │  │ Assistant  │   │ (Rooms/Resc.)│   │  Sharing    │
 │              │  │ (interval   │  │ "Find time"│   │  Strong-     │  │  (Zanzibar) │
 │              │  │  tree cache)│  │            │   │   consistent │  │             │
 └──────┬───────┘  └─────┬───────┘  └─────┬──────┘   └─────┬────────┘  └─────┬───────┘
        │                │                │                │                 │
        ▼                ▼                ▼                ▼                 ▼
 ┌────────────────────────────────────────────────────────────────────────────────┐
 │                    Calendar Metadata DB (Spanner, sharded by calendar_id)      │
 │   tables: calendars, events, recurrences, overrides, attendees, acl, sync_log  │
 └──────────────────────────────────┬─────────────────────────────────────────────┘
                                    │ change stream
        ┌───────────────────────────┼─────────────────────────────────────┐
        ▼                           ▼                                     ▼
 ┌─────────────┐            ┌──────────────────┐                  ┌─────────────────┐
 │  Sync /     │            │ Reminder         │                  │ Search Indexer  │
 │  Notify     │            │ Scheduler        │                  │ (Elasticsearch) │
 │  Service    │            │ (Hierarchical    │                  │                 │
 │  (gRPC      │            │  Timing Wheel +  │                  └─────────────────┘
 │   stream /  │            │  Persistent Q)   │
 │   CalDAV)   │            └──────┬───────────┘
 └─────────────┘                   │
                                   ▼
                          ┌──────────────────┐
                          │ Notification     │
                          │ Fanout (email,   │
                          │ FCM/APNs, popup) │
                          └──────────────────┘

                       ┌──────────────────┐
                       │ Federation Gateway│  ⇄  Exchange / iCloud / public ICS
                       └──────────────────┘
```

### Component responsibilities

| Service | Owns | Storage |
|---|---|---|
| **Event Service** | CRUD on events, RRULE parsing, override application, RSVP | Spanner |
| **Free/Busy Service** | High-QPS busy-interval lookup with caching | Read-through cache → Spanner |
| **Scheduling Assistant** | "Find a time" across N attendees | Stateless, pulls from Free/Busy |
| **Reservation** | Atomic booking of meeting rooms / resources | Spanner with serializable isolation |
| **ACL/Sharing** | Per-calendar permissions; event-level overrides | Spanner (Zanzibar tuples) |
| **Sync/Notify** | gRPC streams + CalDAV server; sync-token deltas | Stateful, sharded by user |
| **Reminder Scheduler** | Schedule and deliver pre-event reminders | Hierarchical timing wheel + persistent delayed queue |
| **Notification Fanout** | Render and dispatch via email/push/popup | Stateless workers |
| **Search Indexer** | Full-text index of events | Elasticsearch + B-tree on time |
| **Federation Gateway** | Translate to/from Exchange, CalDAV, ICS | Stateless adapters |

---

## 3. Data Model

### 3.1 Schema (Spanner-style, sharded by `calendar_id`)

```sql
calendars(
  calendar_id PK,           -- a calendar = an ACL boundary (personal / work / shared / room)
  owner_user_id,
  type ENUM(personal, group, resource, holiday, birthday, ics_subscription),
  default_timezone,         -- e.g. "America/New_York"
  visibility,
  color,
  ...
)

events(
  calendar_id, event_id PK,
  series_id NULLABLE,       -- non-null if part of recurrence; the canonical "master" event id
  is_master BOOL,           -- the master holds the RRULE; instances reference it
  recurrence_id NULLABLE,   -- if this row is a single-instance override of a master, the original start time
  status ENUM(confirmed, tentative, cancelled),
  title, description, location,
  start_dt, end_dt,         -- TIMESTAMP WITH ZONE OR DATE for all-day
  start_tz, end_tz,         -- IANA tz string; preserved separately from UTC
  rrule TEXT,               -- the RFC 5545 rule, e.g. "FREQ=WEEKLY;BYDAY=MO,WE;UNTIL=20300101"
  rdate JSONB,              -- additional one-off occurrences
  exdate JSONB,             -- excluded occurrences (DATE-TIME values)
  organizer_user_id,
  conferencing_uri,
  visibility,
  etag,                     -- monotonic version, used for optimistic concurrency
  created_at, updated_at,
  is_deleted BOOL,          -- soft delete
  ...
)
INDEX events_by_calendar_time (calendar_id, start_dt)
INDEX events_master_lookup    (series_id, recurrence_id)

attendees(
  calendar_id, event_id, attendee_user_id PK,
  response ENUM(needsAction, accepted, declined, tentative),
  optional BOOL,
  is_organizer BOOL,
  ...
)

acl(
  calendar_id, principal_id, role ENUM(freeBusy, reader, writer, owner),
  inherited BOOL
)

reminders(
  calendar_id, event_id, recipient_user_id, method ENUM(email,push,popup),
  minutes_before INT
)

sync_log(
  calendar_id, seq BIGINT PK,   -- monotonic per-calendar
  op ENUM(create, update, delete),
  event_id, recurrence_id NULLABLE, snapshot JSONB,
  committed_at TIMESTAMP
)

resources(
  resource_id PK, name, capacity, building, floor, equipment[], booking_calendar_id
)

reservations(
  resource_id, start_dt, end_dt PK,   -- range type with EXCLUDE constraint forbids overlap
  event_id, calendar_id, status
)
```

Key choices:
- **Master + override pattern** for recurrence: one row per series + one row per *exception* (override or cancellation). 1 B recurring events × no expansion = trillions of rows avoided.
- **Store start_tz separately from start_dt**: lets us preserve user intent across IANA tz changes (see §4.2).
- **ETag for optimistic concurrency**: enables CalDAV `If-Match` and lock-free multi-writer correctness.
- **Per-calendar `sync_log`**: monotonic sequence drives delta-sync. Same pattern as Google Drive.
- **`reservations` with range type + EXCLUDE constraint**: Postgres-style overlap prevention enforced at the DB level — no double-booked rooms even with concurrent writers.

### 3.2 Sharding
- **Calendar metadata**: shard by `calendar_id`. A user's calendars cluster on one shard for cheap multi-calendar reads.
- **Resources** (meeting rooms): shard by `building_id` to keep room-availability queries shard-local.
- **Sync log**: co-located with the events shard (same Paxos group) → atomic write of event + log row.

---

## 4. Core Algorithms

This is the bulk of the design. Calendar's correctness lives or dies in these algorithms.

### 4.1 RRULE Expansion (RFC 5545)

The killer feature. A 2-line RRULE can expand to millions of occurrences ("every weekday forever"). We must:
- **Never materialize** the expansion in the DB (would blow up storage).
- Expand **on demand** for the time window the client is viewing (a few weeks at a time).
- Apply EXDATE / RDATE / RECURRENCE-ID **overrides** correctly.

#### 4.1.1 RRULE grammar (minimum we must support)

```
RRULE = {
  FREQ:       SECONDLY | MINUTELY | HOURLY | DAILY | WEEKLY | MONTHLY | YEARLY
  INTERVAL:   N (default 1)             -- "every N units"
  COUNT:      N        OR  UNTIL: timestamp   -- termination
  WKST:       MO|TU|... (default MO)    -- defines week boundary for WEEKLY
  BYDAY:      list of (±N)?(MO|TU|WE|TH|FR|SA|SU)   -- e.g. "1MO" = first Monday, "-1FR" = last Friday
  BYMONTH:    list of 1..12
  BYMONTHDAY: list of ±1..31
  BYYEARDAY:  list of ±1..366
  BYWEEKNO:   list of ±1..53 (ISO 8601 weeks)
  BYHOUR:     list of 0..23
  BYMINUTE:   list of 0..59
  BYSECOND:   list of 0..60
  BYSETPOS:   list of ±1..366  -- selects from the candidate set (e.g. "the last weekday of the month")
}
```

Example: `FREQ=MONTHLY;BYDAY=MO,TU,WE,TH,FR;BYSETPOS=-1` = "the last weekday of every month".

#### 4.1.2 Expansion algorithm

The RFC 5545 expansion is a layered filter pipeline. Implementing it naively (loop forever, test every day) explodes for `FREQ=YEARLY;COUNT=10000`. The correct approach is **constructive**:

```
def expand(rule, dtstart, window_start, window_end):
    # 1. Generate frequency-aligned anchor points
    anchors = generate_freq_periods(rule.FREQ, rule.INTERVAL, dtstart, window_end)
    #    e.g. for FREQ=WEEKLY → list of week-start dates

    occurrences = []
    for anchor in anchors:
        # 2. Within each period, generate all candidate datetimes
        candidates = anchor_period(anchor)   # all days in this week / month / year
        candidates = filter_by_BYMONTH(candidates, rule.BYMONTH)
        candidates = filter_by_BYWEEKNO(candidates, rule.BYWEEKNO)      # YEARLY only
        candidates = filter_by_BYYEARDAY(candidates, rule.BYYEARDAY)    # YEARLY only
        candidates = filter_by_BYMONTHDAY(candidates, rule.BYMONTHDAY)
        candidates = filter_or_expand_BYDAY(candidates, rule.BYDAY)
        # BYDAY for WEEKLY: filter; for MONTHLY/YEARLY with N: expand ("3rd Monday")
        candidates = expand_by_BYHOUR(candidates, rule.BYHOUR or [dtstart.hour])
        candidates = expand_by_BYMINUTE(...)
        candidates = expand_by_BYSECOND(...)

        # 3. BYSETPOS picks indices from this candidate set
        if rule.BYSETPOS:
            candidates = pick(candidates, rule.BYSETPOS)

        # 4. Filter to window and append
        for c in candidates:
            if c >= dtstart and c <= rule.UNTIL and c in [window_start, window_end]:
                occurrences.append(c)
            if rule.COUNT and len(occurrences) >= rule.COUNT: return
    return occurrences
```

Notes:
- The **filter vs expand** rules are precise (RFC 5545 §3.3.10): for `FREQ=WEEKLY`, BYDAY filters; for `FREQ=MONTHLY/YEARLY`, BYDAY can expand.
- COUNT must be enforced from `dtstart`, not from `window_start` — otherwise pagination breaks the count.
- We always cap expansion at a server-side limit (e.g. 10 000 occurrences per query) to defeat pathological RRULEs.

#### 4.1.3 Apply overrides

```
def expand_with_overrides(master_event, window):
    occurrences = expand(master_event.rrule, master_event.dtstart, window.start, window.end)
    occurrences += master_event.rdate                                # explicit additions
    occurrences -= master_event.exdate                                # explicit removals
    overrides = fetch_overrides_for_series(master_event.series_id, window)
    by_recurrence_id = {o.recurrence_id: o for o in overrides}
    for occ_dt in list(occurrences):
        if occ_dt in by_recurrence_id:
            replace occ_dt with by_recurrence_id[occ_dt]   # may shift to different time, mark cancelled, etc.
    return sorted(occurrences)
```

#### 4.1.4 Caching

Naively re-expanding on every read is expensive. Cache:
- **Per-series, per-month** materialized view of expanded instances → keyed by `(series_id, month, override_etag)`.
- Invalidate when the master event mutates or any override is added/changed.
- For ultra-hot series (e.g. company-wide standups with 10 000 attendees) keep in distributed memory cache (Memcached/Redis) with 5-min TTL.
- Don't expand more than ~5 years into the future; expand on demand if a client paginates further.

#### 4.1.5 "Edit this and following"

When a user picks "this and all future", we:
1. **Truncate** the original master with `UNTIL = recurrence_id - 1`.
2. **Create** a new master starting at `recurrence_id` carrying the new properties + a fresh `series_id`.
3. Carry forward existing exceptions whose recurrence_id ≥ the cut point (or detach and migrate them).

This is correct, atomic, and never loses past attendance/RSVP data.

### 4.2 Time Zone Handling — the hardest pure-correctness problem in calendar

Calendar's #1 bug class. Three rules drilled in:

#### Rule 1 — store wall-clock time + IANA tz, **not UTC**, for events tied to local life

If you store "3 PM UTC" for a recurring 9 AM ET standup, and Congress changes the DST schedule, your meeting now happens at 8 AM ET. **Wrong**. The user said "9 AM Eastern" — that's the source of truth.

Correct storage:
```
start_dt = 2026-04-30T09:00:00     -- naive local datetime
start_tz = "America/New_York"       -- IANA zone identifier
```

When we need to fire a reminder or display in another zone, we resolve at *that moment* using the current tz database:
```
absolute_utc = ZonedDateTime(start_dt, start_tz).to_utc()  // resolved fresh
```

#### Rule 2 — DST transitions create non-existent and ambiguous local times

- **Spring forward**: 2 AM jumps to 3 AM. The local time `2:30 AM` doesn't exist on that day.
- **Fall back**: 2 AM jumps back to 1 AM. The local time `1:30 AM` exists *twice*.

Conventions (RFC 5545 / java.time / ICU):
- **Non-existent local time**: shift forward by the gap (1 hour usually). Event scheduled at 2:30 AM → fires at 3:30 AM that day.
- **Ambiguous local time**: pick the **earlier** occurrence (the pre-fall-back one) by default; allow override.
- These rules apply when *expanding* recurrences — most occurrences are unambiguous; only the DST-transition day is special.

#### Rule 3 — IANA tz database changes; future events may shift

Countries change their DST policies (Egypt 2014, Russia 2014, Chile 2016, Lebanon 2023…). When the tzdata package updates, **every future event resolves differently from then on**. We embrace this — the user wants "9 AM Cairo" regardless of what the politicians decide.

Implementation:
- Calendar service ships with the latest IANA tzdata (and is rebuilt + redeployed within hours of a release).
- On every reminder fire, the absolute UTC time is recomputed from `(start_dt, start_tz)`. Don't pre-materialize UTC into a queue more than ~60 minutes ahead — see §4.5.
- For very-near-term reminders that slipped past a tzdata update mid-flight, we accept they fire at the old-rules time. The < 1-hour blast radius is acceptable.

#### All-day events
Stored as `DATE`, not `DATETIME`. They're the *date* in *every viewer's* zone — your birthday is your birthday everywhere. No tz arithmetic required.

#### Floating events
Some imports (rare) have no tz. They float — they happen at whatever local zone the device is in. Store as wall-clock with `start_tz = "FLOATING"`; resolve to viewer's current zone at display time.

### 4.3 Free/Busy via Interval Trees + Sweep Line

The "is X free between t1 and t2?" query and the broader "give me X's busy intervals in [a, b]" are the highest-QPS read paths in the system.

#### 4.3.1 Storage layout
For each calendar we maintain a **per-month bucketed list** of busy intervals (after expanding recurrences for that month and applying overrides). Cache key: `freebusy:{calendar_id}:{yyyy-mm}`. Invalidated on event mutation in that month.

#### 4.3.2 Single-calendar query
Given query `[q_lo, q_hi]`, fetch the months it covers, return all intervals overlapping. Each month's list is sorted by start time → O(log n + k) via binary search.

#### 4.3.3 Multi-calendar union (the find-a-time inner loop)
Given N calendars, compute the **union of busy intervals** to find common free time. Two algorithms:

**(a) Merge intervals (offline)**: concatenate all busy lists, sort by start, sweep:
```
def merge(intervals):
    intervals.sort(key=lambda i: i.start)
    out = []
    for i in intervals:
        if out and i.start <= out[-1].end:
            out[-1].end = max(out[-1].end, i.end)
        else:
            out.append(i)
    return out
```
O((Σnᵢ) log(Σnᵢ)). Fine for N ≤ ~50 calendars.

**(b) k-way merge (online)**: each calendar's list is already sorted; merge with a min-heap keyed by next interval start. O(Σnᵢ · log N). Streamable — emit free slots as we go.

For "find a time" in §4.4, (b) is preferred because we can early-exit once we find enough slots.

#### 4.3.4 Interval tree for cross-calendar overlap queries
For administrative queries ("which rooms are free at 3 PM Tuesday?") across thousands of resources, build a per-building interval tree (centered interval tree) indexed by time. O(log n + k) for an overlap query against the entire building.

For higher write throughput, use a **segment tree** keyed by 5-minute time buckets — O(1) update on bucket flip, O(window/5min) range query. We pay storage for query speed; bookings are rare relative to lookups.

### 4.4 Find-a-Time / Scheduling Assistant

Problem: given attendees A₁..Aₙ, duration D, working-hours window, time zones, and preferences (mornings vs afternoons, avoid lunch), return ranked candidate slots.

#### 4.4.1 Naïve sweep
1. Compute union of busy intervals across all attendees (§4.3.3).
2. Compute complement within working-hours window → free intervals.
3. Filter free intervals of length ≥ D.
4. Score and return top K.

```
def find_time(attendees, duration, window):
    busy = merge_intervals(union(a.busy_in(window) for a in attendees))
    free = complement(busy, working_hours(window))
    return [interval for interval in free if interval.length >= duration]
```

O(N log N) where N = total busy intervals. Fast.

#### 4.4.2 Soft constraints — scoring

Real "Suggested Times" is not just "find a slot"; it's "find the *best* slot". Score function:

```
score(slot) = w1 * preference_match(slot)            // mornings? afternoons?
            - w2 * count(optional_attendees_busy)    // optional attendees soft-block
            - w3 * timezone_inconvenience(slot)      // 3 AM for someone is bad
            + w4 * proximity_to_existing_events      // back-to-back is sometimes preferred
            - w5 * focus_time_violation(slot)
```

Top-K via a bounded heap. The scoring weights are tuned offline against acceptance data.

#### 4.4.3 With optional attendees — set-cover-ish

When attendees are split into required vs optional, we want to maximize coverage of optionals while keeping all requireds free. Reduce to:
- Find slots where all required attendees are free (hard).
- Among those, rank by number of optionals also free.

This is a linear scan, **not** an NP-hard set cover, because slots are independent. O(F · O) where F = free slots, O = optionals.

#### 4.4.4 Complexity guard
For meetings with > ~100 attendees ("all hands"), we don't run the full scheduler. We surface a free/busy *grid* and let the organizer pick. NP-hardness lurks if you constrain enough; we sidestep it by capping N.

### 4.5 Reminder Scheduling — Hierarchical Timing Wheel + Persistent Queue

The hardest dataplane problem. Requirements:
- **Exactly-once-ish** delivery (at-least-once + idempotent fanout = effectively once).
- **Punctual**: fire within ±2 s at p99.
- **Burst-tolerant**: 5 M reminders at 10:00:00 sharp. (Everyone sets reminders for the top of the hour.)
- **Survives crashes**: if a scheduler dies, no reminder is lost.
- **Reschedulable**: events move; reminders must follow.

#### 4.5.1 Two-tier scheduler

Tier 1 — **Persistent delayed queue** (e.g. Kafka with delay topic, or a sharded SQL queue):
- Stores `(event_id, recurrence_id, recipient_id, fire_at_utc)` rows.
- Polled in chunks of "next 60 minutes worth" by Tier 2.
- On event mutation → tombstone old row, insert new (no in-place update).

Tier 2 — **In-memory hierarchical timing wheel** in each scheduler shard:
- Levels: seconds (60 spokes), minutes (60), hours (24).
- Resolution: 1 sec at the bottom level. To schedule fire-at-T: insert at level whose granularity covers `T - now`. As wheels rotate, items cascade down to finer wheels until they hit the second wheel and fire.
- O(1) insert, O(1) fire — no log N like a heap. This is the standard pattern from the Linux kernel and Netty.

```
class TimingWheel:
    levels = [Wheel(60, 1s), Wheel(60, 60s), Wheel(24, 3600s)]
    def schedule(item, fire_at):
        delta = fire_at - now
        for w in levels:
            if delta < w.span:
                w.insert(item, delta // w.tick); return
        # else: stash in "far future" overflow list, re-evaluate later
    def tick():
        items = bottom_wheel.pop_current_spoke()
        for it in items: emit(it)
        if bottom_wheel.wrapped():
            cascade_from_above()
```

#### 4.5.2 The 60-minute hydration window

Tier 1 holds reminders far in the future (years for recurring meetings). Every minute, each scheduler shard pulls the next 60 minutes' worth of due reminders for its assigned key range and loads them into its in-memory wheel.

Why 60 min? Long enough to absorb DB blips, short enough to keep memory bounded and to absorb tzdata-update churn within an acceptable blast radius (§4.2 Rule 3).

#### 4.5.3 Sharding
Reminder rows sharded by `hash(recipient_user_id)`. Each scheduler shard owns a key range. Failover via Paxos lease — if a scheduler dies, its peer takes over after lease timeout. State is recoverable from the durable Tier-1 queue → in-memory wheel can be lost.

#### 4.5.4 Exactly-once semantics
True exactly-once doesn't exist over a network. We get "effectively once":
1. **Idempotency key** = `(event_id, recurrence_id, recipient_id, channel, fire_minute)`. Stored in a "fired" KV (Bigtable / DynamoDB) with 24 h TTL.
2. Fanout worker: `if not exists(key): mark(key); send()`. The mark is atomic via CAS.
3. Send-side dedup is also enforced by downstream (FCM/APNs dedupe by collapse_key; SMTP idempotency via Message-ID).
4. **At-least-once + idempotent**: the worker can crash mid-send, retry will see the key set; if mark succeeded but send failed, we miss the reminder for that channel. Mitigation: **mark *after* send succeeds**, accept rare duplicates instead. Trade-off chosen consciously.

#### 4.5.5 Late-fire handling
If a scheduler shard was down during a reminder window, on recovery it scans Tier 1 for "fire_at < now AND not_marked" rows. We dispatch them with a "late" flag — clients can decide whether to suppress (a 2-hour-late "your meeting starts in 10 min" is worse than nothing). Default: suppress reminders > 5 min late, surface a "missed reminder" diagnostic instead.

#### 4.5.6 Why not a heap?
A min-heap would work but is O(log n) per op and contention-heavy under bursts. The timing wheel is O(1), better cache behavior, and the level structure naturally batches the "5 M reminders at 10:00:00" thundering herd into the same spoke → we can batch-emit and let downstream flow control kick in.

### 4.6 Sync — Sync Tokens + Delta Pagination (CalDAV-compatible)

#### 4.6.1 Per-calendar sync log
Every mutation appends a row to `sync_log` keyed by monotonic `seq`. Clients hold a `sync_token` (= last `seq` they processed).

#### 4.6.2 Delta protocol
```
client → POST /sync { calendar_id, sync_token = T }
server  →
  changes: [ {op, event_id, snapshot}, ... ]   // all rows with seq > T
  next_sync_token: T'
  more: false
```
- O(changes_since_T) — independent of total event count.
- Pageable: server caps response at e.g. 500 entries; client retries with new token.
- **Token invalidation**: if the client's token is older than retention (say 30 days), server returns `410 Gone` → client falls back to **full resync** (whole calendar contents, returns a fresh token). Common pattern: long-disconnected clients get a full snapshot.

#### 4.6.3 Why per-calendar, not per-user?
A user's calendars may be on different shards (shared with them by other users). Per-calendar tokens scale linearly with active calendars (~5 per user) and keep each token cheap.

#### 4.6.4 CalDAV interop
Native CalDAV (RFC 4791) uses `REPORT sync-collection` with WebDAV `sync-token`. Our protocol maps 1:1 — the gateway translates HTTP REPORT verbs to internal gRPC. ETag per event enables `If-Match` optimistic concurrency for native macOS Calendar.app users.

### 4.7 Optimistic Concurrency — ETags

Two writers (web tab + mobile) edit the same event. Last-write-wins silently loses a field.

Better: every event row carries a monotonic `etag`. Writer sends `If-Match: <etag>`:
- Server transaction:
  ```
  BEGIN
    SELECT etag FROM events WHERE event_id = ? FOR UPDATE
    IF etag != if_match: ABORT 412 Precondition Failed
    UPDATE events SET ..., etag = etag + 1
    INSERT INTO sync_log
  COMMIT
  ```
- 412 → client refetches, presents 3-way merge UI ("the event was changed by Bob; merge?"), or for trivial fields just retries.

For event RSVP (which only mutates one row in `attendees`), no etag conflict is needed — the principal_id is the natural key, and last-write-wins is correct (your RSVP overrides your own previous RSVP).

### 4.8 Resource Booking — Strong Consistency

Meeting rooms cannot be double-booked. This is the only place in calendar where we need full serializable isolation.

#### 4.8.1 Postgres-style range exclusion
```sql
CREATE TABLE reservations (
    resource_id BIGINT,
    span TSTZRANGE,    -- [start, end)
    event_id BIGINT,
    EXCLUDE USING GIST (resource_id WITH =, span WITH &&)
);
```
The `EXCLUDE` constraint uses a GiST index to forbid any insert that overlaps an existing row for the same resource. Concurrent attempts to book the same room → one wins, the other gets a constraint-violation error → translates to user-visible "Room just got booked, try another". No double-bookings even at 10⁶ concurrent attempts.

#### 4.8.2 Spanner equivalent
Spanner doesn't have GiST. We achieve the same with:
- A **booking key bucket** strategy: bucket time into 5-min slots; a reservation writes a row per slot it spans, with primary key `(resource_id, slot_start)` and a unique constraint. Concurrent writers conflict on the slot rows → Paxos rejects all but one. O(duration / 5 min) writes per booking; acceptable for typical 30 / 60 / 120 min meetings.
- **Or**: serializable transaction with read-then-write on `reservations` filtered by `(resource_id, span && new_span)`. Spanner serializability via lock acquisition makes this safe but blocking.

#### 4.8.3 Tentative bookings
When the user is *picking* a room in the UI, we hold a **5-minute soft lock** (advisory lease keyed by `(resource_id, time, user_id)`). Soft lease ≠ hard reservation; expires automatically. Prevents the "five people staring at the same Find-a-Room dialog all click Book" thundering herd.

### 4.9 Search

Index pipeline:
- On event commit → enqueue indexing job.
- Worker extracts (title, description, attendee names, location) and pushes to Elasticsearch with fields `(calendar_id, start_dt_unix, body)`.
- Range queries on `start_dt_unix` use a B-tree.
- Recurring events: index the *master* once with metadata that it recurs; expand at query time for the visible window. Don't bloat the index with materialized instances.

Permission filter: Zanzibar tuples → expand to a calendar_id list at query time → filter top-K. Same trick as Drive (§Drive 7).

---

## 5. Multi-Device Sync — full corner-case treatment

| Case | What goes wrong | Our handling |
|---|---|---|
| **Two devices edit the same event offline** | Both push when online; last-writer-wins drops a field | ETag check → second write gets 412 → client merges (per-field 3-way merge, or surface conflict UI) |
| **Organizer changes time; attendee already declined** | Should attendee's response carry over? | RFC 5545 says a SEQUENCE bump invalidates RSVPs → all attendees' responses reset to needsAction. We surface a notification "this event was rescheduled — please respond again." |
| **Attendee declines a single instance of a recurring meeting** | Must not affect master | Create an override row with `recurrence_id = that_instance_dt` and that attendee's response = declined. Master untouched. |
| **Clock skew between devices** | Stale write looks newer | Use server-assigned monotonic `etag` and `seq`, never client wall clock. |
| **DST transition on a recurring event** | "9 AM every Monday" — what happens on the Monday following spring-forward? | Wall-clock + tz storage (§4.2) keeps it 9 AM local. UTC moves by 1 hour automatically. |
| **IANA tzdata update** | Future-dated meetings shift in UTC | Recompute UTC at fire time from wall-clock + tz; do not cache UTC > 60 min ahead (§4.5.2). |
| **Cross-time-zone meeting** | "11 AM London" rendered as "6 AM New York" | Render in viewer's zone always; backend stores organizer's zone as canonical. |
| **All-day birthday on Feb 29** | Leap-year only | Calendar convention: roll forward to Mar 1 in non-leap years (or to Feb 28; user-configurable). |
| **Event in a country that abolishes DST mid-year** | Same wall-clock event suddenly happens at a different UTC | Acceptable per Rule 1 — wall-clock intent preserved. |
| **Recurring event with COUNT, attendee deletes one instance** | Does it consume one of the COUNT? | Per RFC 5545: deletion via EXDATE does **not** decrement COUNT (the slot was generated). Implementation must enforce. |
| **Master event deleted; overrides remain** | Orphaned override rows | On master delete, cascade-delete overrides. (Don't leave dangling rows pointing at a tombstoned series_id.) |
| **"This and following" applied to first instance** | Equivalent to editing the whole series | Detect: cut point == dtstart → just update master in place, no series split. |
| **Bursty client (mobile reconnect)** | 1 000 events in queue → 1 000 individual writes | Client batches via a multi-write API (`POST /events:batch`); server validates, applies in one Spanner transaction. |
| **Sync-token expired** | Client offline for 60 days | 410 Gone → client falls back to full resync of the calendar. |
| **Federation-imported event with malformed RRULE** | One bad ICS file breaks rendering | RRULE parser is strict; on parse error, store original text + a flag, render as one-off non-recurring event, surface diagnostic to user. Don't crash. |
| **Time-travel attack via past event creation** | Audit gap | All writes carry a server timestamp; user-supplied `created_at` ignored except for ICS imports. Audit log immutable. |
| **Reservation race for room** | Two users book at same instant | EXCLUDE constraint / serializable transaction → only one succeeds (§4.8). |
| **User in flight (no connectivity) RSVPs** | Reminder fires for declined event | Reminder dispatch checks current RSVP at fire time, not at scheduling time. The reminder row carries event_id + recipient; lookup is one query. |
| **Email→Calendar import (Gmail event extraction)** | Hallucinated dates | Extracted events go to a separate `Suggested` calendar with low-trust flag; user must accept to promote to primary calendar. |
| **Forwarding a calendar invite** | New attendee not on original list | Organizer must approve (configurable). Otherwise forwarded recipient gets a "tentative" entry that doesn't propagate to the real attendees list. |
| **Organizer leaves the company / account deleted** | Orphaned recurring meeting forever | Background job promotes the longest-tenured attendee to organizer if `organizer_account_status = deleted`; surfaces a notification. |

### 5.1 Push to devices
- Same pattern as Drive: long-lived gRPC streams sharded by user_id; on commit, publish to per-user channel; clients receive and run delta-sync.
- Apple devices: APNs background push with `content-available: 1` triggers app refresh.
- CalDAV clients: WebDAV-Push extension where supported; fallback to polling every 5 min.

---

## 6. Sharing, ACL, and Federation

### 6.1 ACL
- **Per-calendar role** (Zanzibar relation tuples): freeBusy / reader / writer / owner.
- **Per-event override**: `visibility = private` hides title from non-organizer readers (only "Busy" shown).
- **Strong consistency for revocation**: same fast-path tombstone pattern as Drive — propagation of grants is eventual, propagation of revokes is strong.

### 6.2 Cross-organization sharing
When a user at company A invites a user at company B:
- Invitation flows through the **Federation Gateway** as an iCalendar (`.ics`) attachment + structured headers.
- B's calendar system (Exchange / iCloud / another Google domain) imports it.
- RSVP responses come back as iCalendar `METHOD:REPLY` messages → gateway parses → updates `attendees` row on the master event.
- Free/busy across orgs uses **Free/Busy URL** (RFC 4791 §7.10) — limited capability, returns busy intervals only, no titles.

### 6.3 Trust boundaries
- Inbound iCalendar from random senders: anti-phishing — strip executable conferencing links from untrusted domains; sandbox descriptions for HTML-injection.
- Public ICS subscriptions: read-only; refreshed every 12 h; rate-limited per source domain.

---

## 7. Notifications

Three channels, three different reliability profiles:
| Channel | Latency target | Delivery semantics | Provider |
|---|---|---|---|
| **In-app popup** | < 100 ms after fire | Best-effort; client must be online | Internal long-poll/stream |
| **Push notification** | < 5 s | At-least-once; dedupe by `collapse_key` | FCM / APNs |
| **Email** | < 30 s | At-least-once; dedupe by `Message-ID` | Internal SMTP fleet |

Fanout worker reads `(reminder_fired_event)` from a Kafka topic, materializes per-recipient-per-channel, delivers, and writes to an audit log. Dead-letter queue for unrecoverable failures (e.g. email bounced) → user notification.

Quiet hours and Do-Not-Disturb override channel selection but never delete the reminder — popup is suppressed, in-app badge still increments.

---

## 8. Scalability

### 8.1 Sharding
- **Events** by `calendar_id`: most ops single-shard (a calendar is the unit of storage and ACL).
- **Reminders** by `recipient_user_id`: single-shard for top-K-soonest queries against a user.
- **Resources** by `building_id`: keeps room queries shard-local.
- Hot-shard mitigation: a viral shared calendar (1 M followers) is *read-only* for them → push state to a CDN of "recent events" snapshots, refreshed every minute. Followers pull from CDN, not from the origin shard.

### 8.2 Caching
| Tier | What | TTL | Invalidation |
|---|---|---|---|
| Edge CDN | Public calendar feeds | 5 min | LRU + explicit purge on owner write |
| Free/busy cache | Per-(calendar_id, month) busy intervals | 1 hr | Cache miss on event write within month |
| Event cache | Hot events (your today/tomorrow) | 5 min | Cache busted on `etag` change |
| RRULE expansion cache | (series_id, month) | 24 hr | Bust on master mutation |
| Negative cache | "no such event" | 30 sec | TTL only |

### 8.3 Storage tiering
- Active events (within ±2 yr) on SSD with 3× replication.
- Historical (> 2 yr in past) on HDD with EC; reads on demand, accept higher latency.
- Future events (> 5 yr ahead) re-expanded lazily; we rarely materialize those far out.

### 8.4 Geographic distribution
- Events sharded by `calendar_id`; leader pinned to user's home region.
- Free/busy reads served from regional read replicas (eventual consistency tolerable for "Bob's busy intervals were ~2 s ago").
- Cross-region invitations route through Federation Gateway with the recipient's home region.

### 8.5 Backpressure
- Per-user write QPS cap (token bucket).
- Bulk imports (large ICS file) routed to a separate slow-path queue → don't compete with live traffic.
- Find-a-time with > 100 attendees rejected with "use a poll instead" — hard-coded N cap.

### 8.6 The 10 AM thundering herd
At the top of every hour the reminder system processes ~5 M fires/sec. Mitigations:
- **Time-jittered hydration**: scheduler shards stagger their tier-1 → tier-2 reload by 100 ms each → smooths the DB read burst.
- **Batched fanout**: for each event, generate one rendered notification and fan out per recipient, instead of N independent renders.
- **Provider rate limits**: APNs and FCM have per-app QPS caps; we buffer in front of them with token-bucket flow control. SLA aware: better to fire 1 s late than to be throttled and fire 5 min late.
- **Pre-warming**: at the 55th minute of every hour, pre-populate caches with the next hour's hot calendars so the fire path is hot.

---

## 9. Reliability

### 9.1 Durability stack
- 3× replicated Spanner across regions, Paxos quorum writes.
- Every event mutation produces a `sync_log` row in the same transaction → no event can be "applied" without being syncable.
- Daily snapshot to cold storage (PITR), retained 90 days.
- Audit log of all mutations (immutable, separate cluster) for compliance + restore.

### 9.2 Failure handling
| Failure | Handling |
|---|---|
| Reminder scheduler shard dies | Paxos lease → peer takes over within ~10 s; loses in-mem wheel; rehydrates from Tier-1 queue; late-fire window absorbs the gap |
| Notification provider outage (FCM) | Per-channel circuit breaker; degrade to email; surface "missed push" badge on next app open |
| Spanner region failover | Leader election → ~10 s of write unavailability for affected shards; reads served from peer regions |
| Free/busy cache poisoned by stale event | Etag mismatch on read → cache invalidated and refilled; never trust cache for security decisions |
| Resource booking constraint violation under network partition | Two-phase: tentative reservation (advisory) → confirmation (transactional). On partition, neither is permanent. Worst case: user retries. |
| Federated peer (Exchange) returning malformed iCalendar | Parser tolerant; on failure, mark invitation `needs_attention`; surface to user with raw-text view |
| Tzdata update mid-flight | < 1 hr blast radius (§4.5.2); affected reminders are imperceptibly off. Daily integration test against current tzdata catches policy changes. |

### 9.3 Reminder reliability — the special case
Lost reminder = lost user trust. Defense in depth:
1. **Synchronous write to Tier-1** before user-visible event creation success.
2. **Tier-2 wheel** is purely a performance optimization; loss is recoverable from Tier-1.
3. **Watchdog**: separate process scans Tier-1 every 5 min for `fire_at < now - 5 min AND not_marked` — late-fires anything missed.
4. **Cross-region duplication**: reminder row replicated to standby region; standby has its own scheduler that runs in passive mode (marked rows propagate); on primary outage, standby promotes → at most 5 min RPO.
5. **Internal SLA**: missed-reminder rate < 10⁻⁶. Tracked obsessively.

---

## 10. Availability

### 10.1 Read SLO 99.99 %
- Stateless service tier behind L7 balancer, ≥ 5 zones.
- Free/busy and event reads served from regional caches; degrade gracefully to direct DB on cache outage.
- CDN absorbs subscribed-feed and public-calendar reads.

### 10.2 Write SLO 99.95 %
- Spanner write requires Paxos quorum → fewer 9s than read. Per-shard impact only.
- Client buffers writes locally; queues replay on reconnect. Web UI shows "offline — changes saved locally" rather than failing.

### 10.3 Reminder SLO 99.999 %
- Higher than read! Achieved by the multi-tier defense in §9.3. Reminders are the product's most user-visible reliability surface; losing one ruins trust permanently.

### 10.4 Graceful degradation
- Search down → fallback to title-substring SQL on metadata DB.
- Find-a-time down → present free/busy grid only; user picks manually.
- Push channel down → email-only delivery; in-app badge counts.
- Federation gateway down → external invitations queue at gateway; show "pending" status; retry.
- Resource booking unavailable → block room booking, allow event creation without room.

---

## 11. Observability

- **Reminder accuracy histogram**: actual_fire_time − scheduled_fire_time. p50, p99, p99.9. Page on regression > 2 s.
- **RRULE expansion cost**: per-call CPU; alert on series with > 10 000 occurrences in a year (likely abusive or malformed).
- **Free/busy cache hit ratio**: target > 95 %.
- **Sync lag**: server commit → device received. p99 < 5 s.
- **Federation success rate** per peer (Exchange/iCloud/etc.) — independent SLO each.
- **Distributed trace** on every reminder fire — sample 100 % of late or missed fires.
- **DST integration test**: synthetic events crossing every upcoming DST boundary; alert if any fires at the wrong wall-clock time.

---

## 12. Trade-offs Worth Defending

| Decision | Alternative | Why we picked this |
|---|---|---|
| **Master + override storage** for recurrences | Materialize all instances | Saves 10 000× storage; expansion is cheap and cacheable |
| **Wall-clock + tz** storage | UTC | Preserves user intent across DST / tzdata changes |
| **Hierarchical timing wheel** for reminders | Min-heap | O(1) ops, batches the 10 AM thundering herd |
| **Tier-1 persistent + Tier-2 in-memory** | Pure persistent | Pure persistent can't hit 1-sec accuracy at 5 M QPS |
| **Per-calendar sync log** | Per-user sync log | Per-cal scales linearly with sharing; per-user has fan-in problems |
| **At-least-once + idempotent** reminders | Two-phase commit / true exactly-once | Cheaper, simpler; downstream dedup makes it effectively-once |
| **Strong consistency for room booking** | Eventual + reconciliation | Double-booking is a UX failure no amount of reconciliation fixes |
| **Eventual for free/busy** | Strong | 2-second lag tolerable; strong would 10× latency |
| **412 + client merge** for concurrent edits | Last-write-wins | LWW silently loses fields; product can't tolerate |
| **CalDAV / WebDAV-Push compatibility** | Proprietary protocol only | Native Apple/Outlook integration is a must-have |
| **Strict RRULE parser** | Tolerant parser | Tolerant parsers normalize bugs into permanent semantics; strict catches them at creation |
| **Cap N attendees on find-a-time** | Optimize the algorithm | The user experience of a 200-person scheduling assistant is bad regardless; cap forces a better UX (poll) |

---

## 13. What Makes This Staff-Level

1. **Algorithms named, with edge cases**: RRULE expansion (filter-vs-expand semantics, BYSETPOS, UNTIL, COUNT), interval-tree free/busy, hierarchical timing wheels, at-least-once + idempotency for reminder semantics, GiST EXCLUDE for resource booking.
2. **Time correctness handled head-on**: wall-clock vs UTC, DST gaps and overlaps, tzdata mutability, floating events, all-day vs timed.
3. **Multi-device sync corner cases enumerated** — DST, recurrences, declines on overrides, sync token expiry, federation parse failures.
4. **Quantified SLOs**: 99.99 read, 99.95 write, **99.999 reminder delivery** (higher than DB itself — defended via multi-tier).
5. **Consistency boundaries explicit**: strong on bookings + ACL revocation, eventual everywhere else; ETag for optimistic concurrency on event mutation.
6. **Failure modes mapped**: scheduler crash, region failover, tzdata mid-flight, federated peer corruption.
7. **Operational reality**: tzdata update cadence, 10 AM thundering herd, watchdog late-fires, audit log, integration tests for DST boundaries.

---

## 14. Open Questions / Extensions

- **AI scheduling agents** booking rooms autonomously: needs rate limits + intent caps to prevent runaway booking storms.
- **Calendar-wide undo** (restore any deletion within N days): needs versioning analogous to Drive's; expensive on storage but high user value.
- **Travel-time blocks** auto-inserted from Maps: tight integration risks, time-sensitive correctness.
- **Cross-account merge** (work + personal): privacy boundaries; per-event visibility filters; tricky.
- **Smart conflict suggestions** ("you're double-booked; reschedule X to Y?"): a small ML model on top of free/busy + meeting-importance heuristics.
- **Climate-aware** time pickers (avoid scheduling across hemisphere DST cliffs)? Nice but niche.
- **Quantum-ready signature** of ICS files for cross-org integrity — out of scope today, but a credible 10-yr horizon item.