# Promotion Timezone Ambiguity — Deep Dive

> "Sale ends Friday." Which timezone? The customer is in Tokyo, the server is in US-East, the promotion was created by a merchandiser in London. The sale ends at 3 different times depending on perspective.

The quick dismissal is "just store everything in UTC." That's the starting point, not the answer. The real questions are harder: **whose Friday does the sale end on?** What does the customer see on the countdown timer? What time do your tax/compliance reports bucket it under? How do you handle DST the weekend the sale spans? What happens to a cart that was valid at 23:59:59 and reached `POST /checkout` at 00:00:01?

Every one of these is a staff-level decision with legal, UX, and operational implications.

---

## 1. Why "Just Use UTC" Is Not The Answer

UTC is a storage format. It's not a product decision. Before you can store anything, you need to answer:

1. **Is this a global sale at a single instant, or a wall-clock sale?**
    - Global instant: "Black Friday starts at 2026-11-27T05:00:00Z globally." Every region sees the launch at the same moment. The UK is at 5am, Tokyo is at 2pm, LA is at 9pm the night before.
    - Wall-clock: "Sale runs 9am-5pm local, every day this week." Different absolute start/end in every region. Tokyo starts 15 hours before New York.
2. **Whose wall clock?** The merchandiser's? The customer's? A single "store timezone" configured per tenant?
3. **Who owns the display format?** "Ends Friday 11:59pm" — is the server rendering "Friday 11:59pm" as a string, or is the client rendering an absolute instant in the user's locale?

Storing UTC doesn't answer any of these. It just means the format of the column is unambiguous once you've answered them.

The actual architectural move: **every promotion has two timestamps and one timezone, and the contract between the three is what you're designing.**

---

## 2. The Three Parties, Three Clocks

```
┌─────────────────────┐
│ Merchandiser         │  wall clock: Europe/London
│ (creates promo)      │  says: "ends Friday 11:59 PM"
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Backend / DB         │  stores: 2026-05-08T22:59:59Z  (UTC)
│                      │  tz:     Europe/London         (intent)
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Customer             │  wall clock: Asia/Tokyo
│ (views the sale)     │  sees:  "ends Saturday 7:59 AM JST"
└─────────────────────┘
```

The bug pattern most companies ship on their first promotion system:
- Merchandiser types "23:59" into a form.
- Backend stores `'2026-05-08 23:59:00'` as a *naive* timestamp.
- Server is in US-East (UTC-4). Application code runs `NOW() < expires_at`, both interpreted in server-local time.
- Customer in Tokyo who tried to check out at "11:50 PM my time Friday" finds the sale expired hours ago.
- Merchandiser in London, who set it for 23:59 local, finds the sale expired 5 hours early by their clock too.
- Nobody is happy, and the bug only surfaces after customer complaints.

The fix isn't subtle, but it's multi-layered: naive timestamps are banned, the intent timezone is persisted explicitly, display is *always* localized to the viewer.

---

## 3. Canonical Data Model

```sql
CREATE TABLE promotions (
    id                   BIGINT PRIMARY KEY,
    code                 TEXT,                   -- "SPRING25" (nullable for automatic promos)
    name                 TEXT NOT NULL,
    kind                 TEXT NOT NULL,          -- 'global_instant' | 'wall_clock'

    -- Absolute bounds. ALWAYS TIMESTAMPTZ (stored UTC, display tz-agnostic).
    starts_at            TIMESTAMPTZ NOT NULL,
    ends_at              TIMESTAMPTZ NOT NULL,

    -- The intent timezone. IANA name, e.g. 'Europe/London', 'America/Los_Angeles'.
    -- For global_instant: typically the merchandiser's or "company" tz — used only for display.
    -- For wall_clock: the authoritative local tz that defines start/end.
    intent_timezone      TEXT NOT NULL,

    -- Per-region override table for wall-clock sales that differ by region.
    -- (If NULL, intent_timezone is authoritative globally.)
    -- See promotion_regions below.

    -- Grace behavior on the boundary.
    grace_seconds        INT NOT NULL DEFAULT 0, -- tolerance window past ends_at

    -- What happens to carts/quotes straddling the boundary.
    straddle_rule        TEXT NOT NULL,          -- 'must_checkout_before_end'
                                                 -- | 'must_start_before_end'
                                                 -- | 'quote_locked_at_time_X'

    created_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by           BIGINT NOT NULL,
    created_by_timezone  TEXT NOT NULL,          -- audit: "what tz was the admin in when they created this?"

    CHECK (starts_at < ends_at)
);

-- For wall-clock promos that run "9am-5pm local" in each region:
CREATE TABLE promotion_regions (
    promotion_id         BIGINT NOT NULL REFERENCES promotions(id),
    region_code          TEXT NOT NULL,          -- 'US-CA', 'JP', 'EU-UK'
    timezone             TEXT NOT NULL,          -- 'America/Los_Angeles' etc.
    local_starts_at      TIMESTAMP NOT NULL,     -- NO timezone — this is a wall clock
    local_ends_at        TIMESTAMP NOT NULL,
    PRIMARY KEY (promotion_id, region_code)
);
```

Two columns do the heavy lifting:

- **`TIMESTAMPTZ`** for absolute bounds. In Postgres, this stores UTC internally and converts on display based on session `TIMEZONE` setting. Comparisons are UTC-safe regardless of where the query runs.
- **`intent_timezone`** as an IANA name (`Europe/London`, not `BST`, not `+01:00`). IANA names survive DST changes and political boundary shifts. UTC offsets don't.

Why the `intent_timezone` is load-bearing even for `TIMESTAMPTZ`:
- **Display:** "Ends Friday 11:59 PM merchant's time" requires rendering in the merchant tz, not the viewer tz. Without storing it, you can't do that without a global config.
- **Edit semantics:** when the merchandiser edits the promo to "end Friday 11:55 PM" (5 min earlier), the backend must interpret the new time string in their tz. Without storing it, you'd convert against server tz and shift the bound.
- **DST correctness:** converting a wall-clock string to UTC requires knowing the zone at the moment of the conversion (see §6).

Why `TIMESTAMP WITHOUT TIME ZONE` is wrong for absolute bounds:
```sql
-- DON'T
ends_at TIMESTAMP NOT NULL   -- naive timestamp. Ambiguous.
```
When you compare `NOW() < ends_at`, Postgres interprets both against `TIMEZONE` session setting. Change the session tz (e.g., a connection pool that doesn't set it, or a replica in a different region), and the comparison shifts. I have seen exactly this cause a company to run a promotion for 5 extra hours on a Saturday morning. Naive timestamps are a trapdoor.

---

## 4. The Two Promotion Flavors

### 4.1 Global-instant promos

"Black Friday launches worldwide at 2026-11-27T05:00:00Z." Single absolute instant.

```
Creation:
  Merchandiser (London) picks "Fri Nov 27, 5am GMT."
  Backend parses against 'Europe/London' (their session tz) → 2026-11-27T05:00:00Z.
  Stores: ends_at=2026-11-28T05:00:00Z, intent_timezone='UTC' (display-only).

Evaluation:
  Server: NOW() < ends_at (both TIMESTAMPTZ, compared as UTC)
  Everywhere on earth: the same truth value.

Customer display:
  UK user:  "Ends Saturday, 28 Nov at 05:00 GMT"
  JP user:  "Ends Saturday, 28 Nov at 14:00 JST"
  US user:  "Ends Friday, 27 Nov at 12:00 AM EST"
```

For global-instant, `intent_timezone` is decorative — used for the merchandiser's admin UI and for reports binned to "store" time. The truth is the UTC instant.

### 4.2 Wall-clock promos

"Sale ends Friday at 11:59 PM store time." Here the intent timezone is authoritative.

```
Creation:
  Merchandiser (London) types "Fri May 8, 11:59 PM" and picks tz='Europe/London'.
  Backend parses the naive string + tz + rules_at_that_moment → 2026-05-08T22:59:00Z.
  Stores: ends_at=2026-05-08T22:59:00Z, intent_timezone='Europe/London'.

Evaluation:
  Same: NOW() < ends_at.

Customer display:
  Any viewer: "Ends Friday, May 8, at 11:59 PM London time"
              AND/OR: "That's {converted to viewer's tz}"
```

For wall-clock, the intent timezone is what the sale "really means." If the merchandiser later decides to shift it to "11:45 PM" without moving the date, you must re-interpret the new time in `Europe/London`, not in whoever edited it.

### 4.3 Per-region wall-clock (the hardest flavor)

"Sale runs 9am-5pm *local* every day this week."

This is N different absolute timestamps, one per region. Model as per-region rows. Evaluation is "find the row for this customer's region, compare against the row's `local_*` fields converted-to-UTC-using-the-region-tz at evaluation time."

This is where storing naive `TIMESTAMP` for the local wall clock is *correct* — the value is "9am" in the abstract, not a specific UTC instant. Converting it to UTC depends on the date you apply it to (DST!), so compute per-day bounds at evaluation time.

---

## 5. Display: Always Local, Always Labeled

The customer should never see "Ends 2026-05-08T22:59:00Z". They should see:

```
Sale ends in 3h 42m.
Friday, May 9, 07:59 AM JST                   ← their local time
(Merchant time: Friday, May 8, 11:59 PM BST)  ← optional secondary, for trust
```

Rendering rules:
- **Backend sends `ends_at` as ISO-8601 with offset or Z suffix**: `"2026-05-08T22:59:00Z"`. Never a pre-formatted string.
- **Client detects user tz** (`Intl.DateTimeFormat().resolvedOptions().timeZone`) and formats locally.
- **Always include the timezone label** on the formatted string. `11:59 PM` alone is a bug waiting to happen; `11:59 PM JST` is unambiguous.
- **For countdown timers, show relative time** (`3h 42m remaining`). Absolute time is secondary. Users don't do mental timezone math; they trust the countdown.
- **Email notifications include both zones** (user tz + store tz) to avoid "why did your email say 2 PM but the sale ended at noon?"

Common server-rendered bug: the backend formats the string using the *server's* tz (because nobody set the session tz on the connection pool) and ships "Ends May 8 at 6:59 PM" to all users regardless of location. Fix: the server must never format timestamps into user-facing strings. That's a client (or edge) concern.

---

## 6. DST, Politics, and Other Timezone Landmines

### 6.1 DST spring-forward: the missing hour

March 9, 2026, 02:00 Pacific → 03:00 Pacific. The hour `02:00–03:00` does not exist. A promotion set to end at "2:30 AM local Pacific" on that day is undefined.

Three valid policies:
- **Reject at creation.** UI refuses to schedule a time in the gap. Cleanest but least flexible.
- **Interpret as "before the gap"** → 01:30 UTC-8 equivalent. Conservative; sale ends an hour earlier than the user might expect.
- **Interpret as "after the gap"** → 02:30 UTC-7 equivalent. Liberal; sale ends an hour later.

Pick one, document it, write the test. Most platforms pick "reject at creation" because the pay-off is clarity.

### 6.2 DST fall-back: the duplicate hour

November 2, 2026, 02:00 Pacific → 01:00 Pacific. The hour `01:00–02:00` occurs twice — once in PDT (UTC-7), once in PST (UTC-8). An end time of "01:30 AM local Pacific" maps to two possible UTC instants.

Policy: default to the **later** occurrence (more generous to the customer). Document this. Make sure your date library respects it: Python's `pytz.localize(..., is_dst=False)` gives the later; Joda-Time's `DateTimeZone.forID(...).toDateTimeLocal(...)` has its own rules. Test with a specific date that falls in this gap.

### 6.3 Political timezone changes

IANA timezone names can change:
- Russia dropped DST in 2011 and re-added in 2014.
- Turkey shifted to permanent UTC+3 in 2016.
- Samoa crossed the International Date Line in December 2011, skipping December 30.
- Egypt added DST, removed it, re-added it, through the 2010s.

If your promotion stored a UTC offset (`+0300`), it's wrong the day the zone changes. If it stored the IANA name (`Europe/Istanbul`), the tz database update handles it. **Always store the IANA name.**

Consequence: your servers and databases must ship tz database updates as part of routine patching. An outdated `tzdata` package on one node means its timezone conversions disagree with the rest of the fleet. Audit this in CI: a test that asserts "Europe/Istanbul resolves to +03:00 for date X" catches drift.

### 6.4 Leap seconds

Technically a timezone landmine. Practically: for promotional timing, don't worry about them. Nobody cares if a promo ends at 23:59:59 or 23:59:60.

---

## 7. Boundary Behavior: The Last-Minute Cart

The sale ends at `T`. A customer:
- Put items in the cart at `T - 10m`.
- Arrived at the checkout page at `T - 30s`.
- Clicked "Place Order" at `T + 5s`.

Do they get the promo price or not?

Three policies, each defensible:

**Policy A: "Must checkout before end."**
Order must be submitted at the server before `T`. Fair by the letter. Harsh for the customer who was one click away.

**Policy B: "Must start checkout before end."**
Apply the promo if the *quote was minted* before `T`, even if the order commits after. Generous. Can extend the sale by the length of your quote TTL (typically 15 minutes). Requires careful quote-expiry handling.

**Policy C: "Grace window."**
Sale technically ends at `T`, but orders placed within `T + grace_seconds` still qualify. Typical grace: 60-300 seconds.

The model above stores `straddle_rule` and `grace_seconds`; the checkout evaluator reads them.

Evaluation code shape:
```go
func isPromoValid(promo Promotion, now, quoteMinted time.Time) bool {
    switch promo.StraddleRule {
    case MustCheckoutBeforeEnd:
        return now.Before(promo.EndsAt.Add(time.Duration(promo.GraceSeconds) * time.Second))
    case MustStartBeforeEnd:
        return quoteMinted.Before(promo.EndsAt)
    case QuoteLockedAtTime:
        // Quote stored the applicable promos; honor whatever was locked in.
        return true  // validated at quote-mint time
    }
    return false
}
```

Staff-level wisdom: **ship with Policy B + 15-minute quote TTL.** It's customer-friendly, it matches the "add to cart during the sale, finish checkout over dinner" pattern, and it makes the edge easier to test. If fraud becomes an issue (users minting quotes late to abuse the sale), tighten to Policy A with grace.

Write the expiry behavior into the `price_quotes` table (see `stalePricingCache.md §4`). A quote that includes a promo has `promo_valid_until = min(quote_expiry, promo_ends_at + grace)`.

---

## 8. Countdown Timers: The Lying Clock Problem

Display shows "Ends in 3h 42m 08s." The client device clock is off by 10 minutes (user manually changed their phone clock). They see "Ends in 3h 52m" when the server thinks it's "Ends in 3h 42m."

Do not trust client clocks. Implement:

1. **Server sends `ends_at` (absolute) + `server_now` (absolute) in the same response.**
2. **Client computes `offset = server_now - client_now`.** This is the delta between the two clocks.
3. **Countdown = (ends_at - (client_now + offset)).**
4. Re-sync offset periodically (e.g., on page load, on visibility change from hidden → visible, on websocket tick).

This makes a skewed client clock irrelevant. The timer reflects the server's view.

For long countdowns (days), periodically re-sync to account for clock drift. For short countdowns (flash sale), one sync at page load is usually enough.

### The "0-second tick" problem

Client shows `00:00:00`. User clicks "Buy." Server says `ends_at` passed 50ms ago. Who wins?

Server wins. UX-wise: either (a) the button becomes disabled at tick zero, or (b) the failed attempt returns a clear "Sale just ended. Removing discount — $100 instead of $80. Proceed?" prompt. Never silently charge full price.

---

## 9. Legal and Compliance

### 9.1 Advertised sale duration

In several jurisdictions (EU, UK, India), you must advertise a sale's duration clearly and honor it for the full stated period. "Ends Friday" is vague; regulators can interpret "Friday" as "all of Friday in the customer's zone," which may mean the sale must run until midnight in *every* zone — i.e., global UTC+14 midnight (Kiribati) is the latest. In practice: show the merchant tz prominently, and either set a global-instant that includes the latest zone, or honor "until midnight local" as a per-region wall-clock promo.

### 9.2 Tax and reporting periods

Revenue from the sale has to be booked into the correct period. If your financial quarter ends March 31 and a sale closes at 23:59 local Pacific on March 31, that transaction occurred on April 1 UTC. Which quarter does it belong to?

Practical rule: book revenue in the **merchant's** tz (the one on the company's tax filings). Store the `booking_tz` on the order so reports can bucket consistently. This is orthogonal to promo display tz; don't conflate them.

### 9.3 Disclosure laws

Some regions require showing an explicit sale end time with timezone to claim a percent-off discount. Ship the tz label visibly (see §5). Don't hide it in a tooltip.

---

## 10. The "Historical Reconstruction" Problem

The next bullet in your corner-cases doc mentions this. A customer disputes a charge from three months ago. What *was* the promo state at time of purchase?

You must be able to answer: at `order.placed_at`, which promos applied to this cart, and at what price?

Two patterns:
- **Immutable promo snapshots on the order.** `orders.applied_promos` is a JSONB of the promo(s) applied at that moment, including `promo_id`, `ends_at`, `discount_amount`, `version`. Never compute promo applicability retroactively from the `promotions` table — by then it may have been edited or deleted.
- **Versioned promo history.** `promotions_history` captures every edit with `valid_from/valid_to`. Reconstruction reads the row whose `[valid_from, valid_to)` contains the order timestamp.

The snapshot approach is simpler and almost always sufficient. History tables are for merchant audit, not customer dispute resolution.

For the reconstruction query: *always use the order's `placed_at` TIMESTAMPTZ*, never "today's" date. The entire point is to recompute the state as-of that instant.

---

## 11. Testing: The Only Way To Trust The Code

Timezone code cannot be tested with one happy-path assertion. The test matrix that catches bugs:

```go
var cases = []struct {
    name         string
    tz           string
    nowLocal     string
    endsAtLocal  string
    shouldApply  bool
}{
    {"merchant-tz before end",    "Europe/London", "2026-05-08T11:30:00", "2026-05-08T23:59:00", true},
    {"merchant-tz after end",     "Europe/London", "2026-05-09T00:01:00", "2026-05-08T23:59:00", false},
    {"dst spring forward gap",    "America/Los_Angeles", "2026-03-09T02:30:00", "2026-03-09T02:30:00", /* handle per policy */ false},
    {"dst fall back duplicate",   "America/Los_Angeles", "2026-11-02T01:30:00", "2026-11-02T01:30:00", /* later occurrence */ true},
    {"leap day",                  "UTC", "2028-02-29T23:59:00", "2028-02-29T23:59:59", true},
    {"customer different tz",     "Asia/Tokyo", "2026-05-08T15:00:00", "2026-05-08T23:59:00+Europe/London", true},
    {"iana change (edge)",        "Europe/Moscow", "2011-10-30T02:30:00", ..., /* policy */ false},
    {"within grace",              "UTC", "2026-05-09T00:00:30", "2026-05-09T00:00:00+grace=60", true},
    {"past grace",                "UTC", "2026-05-09T00:01:01", "2026-05-09T00:00:00+grace=60", false},
    {"quote minted before end",   ..., /* policy B */ true},
    {"quote minted after end",    ..., /* policy B */ false},
}
```

Plus property-based tests:
- For any `(tz, start, end, customer_tz)`: a customer at `customer_tz` viewing at `(start + duration/2)` in their clock sees the promo as active.
- For any valid `(tz, end)`: the promo is inactive at `end + 1 second` server-time.

And explicit clock manipulation in CI:
- Pin the test environment to `TZ=UTC` on the OS.
- Freeze `now()` using a testing library (`github.com/jonboulle/clockwork` for Go).
- Run a subset of tests under `TZ=Asia/Tokyo` in a separate CI job to catch any accidental reliance on server tz.

---

## 12. Observability

Alerts that have caught real bugs in my experience:

- **Promo expired by server but countdown still running on CDN-cached page.** Detect by comparing ES/Redis-cached promo state vs DB. Alert if mismatched >60s.
- **Orders placed with a promo ID whose `ends_at < order.placed_at`.** Should be zero (quote expiry prevents it). If non-zero, quote logic is broken.
- **Customer-support ticket tag "promo-didnt-apply".** A dashboard metric. Any spike = investigate tz, not just "the promo code."
- **Cross-region sale-end skew.** For wall-clock promos with per-region rows: alert if any region's `local_ends_at` converted-to-UTC disagrees with what the DB is evaluating. Catches tzdata drift.
- **Audit `created_by_timezone` vs parsed `ends_at`.** Mismatch (e.g., admin in Tokyo created a promo whose `ends_at` looks London-tz-ish) is a UI confusion, not a crash — but worth catching.

---

## 13. The Staff-Level Summary

When this comes up in a design review, the answer is:

1. **Timestamps in the DB are `TIMESTAMPTZ` (UTC-stored).** Never `TIMESTAMP` for absolute instants.
2. **Every promotion persists its `intent_timezone`** as an IANA name. This is what "ends Friday" is anchored to.
3. **Display is always localized to the viewer,** always labeled with the tz name, always rendered client-side from an ISO-8601 instant. Backend never formats human-readable strings.
4. **Per-region wall-clock promos are modeled as N rows,** each with a naive local time + tz. Converted to UTC per evaluation.
5. **DST policies are explicit** (gap hours rejected; duplicate hours default to later). Tested.
6. **Boundary behavior is explicit** (`straddle_rule` + `grace_seconds`). Default to Policy B + 15-min quote TTL.
7. **Countdown timers use a server-anchored clock offset,** not the client's local clock.
8. **Historical reconstruction is via snapshot on the order,** not via recomputation from the live promo row.
9. **`tzdata` is part of the deployment surface.** Audit in CI; update as part of OS patching.
10. **Revenue booking tz is orthogonal to promo display tz.** Book in merchant tz; display in viewer tz.

The corner-case line "sale ends at 3 different times depending on perspective" is only a bug when you think there's supposed to be *one* true time. There isn't. There are three legitimately different renderings of the same underlying UTC instant — and the failure is not that they're different, but that the system doesn't treat that as a design requirement.
