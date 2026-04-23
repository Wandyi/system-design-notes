# Stale Cache Serving Wrong Price — Deep Dive

> Product catalog is behind a CDN with 5-minute TTL. Price changes from $100 to $80. For 5 minutes, some users see $100, others see $80 (cache hit vs. miss). Users comparing notes on social media see different prices — trust erosion.

The surface-level framing ("cache invalidation is hard") hides the actual architectural problem: **at scale, no single caching layer exists — and every layer has its own staleness semantics.** The question is not "how do I invalidate the cache" but "where is it safe to cache price at all, and what is the binding contract between the displayed price and the charged price?"

---

## 1. Why This Problem Is Worse Than It Looks

A stale price on one page is a UX bug. A stale price *that users act on* is, depending on jurisdiction, a legal exposure.

**Trust erosion at user scale.** User A sees $80, tells friend. Friend sees $100, feels cheated. Screenshots spread on Twitter/Reddit. The cost is not the $20 delta; it's brand damage that outlives the cache TTL by years.

**Legal/regulatory exposure.**
- **Advertising law.** In many jurisdictions (US FTC, EU Consumer Rights Directive, India CCPA), advertising a price and then charging a higher one at checkout is deceptive. "Our CDN was stale" is not a defense.
- **MAP (Minimum Advertised Price) contracts.** Manufacturers set a floor price. If a bug pushes the displayed price below MAP even briefly, you can be delisted from wholesale agreements. CDN staleness after a price correction upward is exactly this risk inverted.
- **Price parity clauses.** If you promise "same price on app and web," a cache divergence between channels can be a contract breach.
- **Discrimination concerns.** If cache topology correlates with geography or user segment (e.g., users on mobile carriers with one edge POP see $100, wired users see $80), you've accidentally implemented differential pricing. GDPR + consumer-protection regulators have views on this.

**Revenue leakage in both directions.**
- Price increased $80 → $100: stale cache keeps selling at $80 for 5 minutes. At 1000 orders/sec on that SKU, that is $1.2M of margin given up.
- Price decreased $100 → $80: stale cache continues showing $100, customers abandon because a competitor shows $80. You lose conversions you had already earned.

**Checkout-vs-display mismatch = cart abandonment.** The single biggest UX sin: user sees $80 on product page, adds to cart, lands on checkout at $100 (because checkout reads from source of truth). Conversion rate on that cohort craters — A/B tests put this at 20-40% abandonment spike.

---

## 2. The Layers Where Price Actually Lives

Before you can solve staleness, you must enumerate every place price is cached. In a typical e-commerce stack there are at least six:

```
┌─────────────────────────────────────────────────────────────────┐
│  L0: Browser memory / localStorage        TTL: session          │
│      (cart widget, react-query cache, service-worker cache)     │
├─────────────────────────────────────────────────────────────────┤
│  L1: CDN edge cache (Fastly / CloudFront) TTL: minutes – hours  │
│      (~200 POPs worldwide; each has its own cache)              │
├─────────────────────────────────────────────────────────────────┤
│  L2: Origin / regional reverse proxy      TTL: seconds – min    │
│      (Varnish in your own DC; nginx cache)                      │
├─────────────────────────────────────────────────────────────────┤
│  L3: Application-level cache              TTL: seconds – min    │
│      (Redis / Memcached in the product service)                 │
├─────────────────────────────────────────────────────────────────┤
│  L4: Search index projection              Lag: seconds – min    │
│      (Elasticsearch / Algolia; may embed price)                 │
├─────────────────────────────────────────────────────────────────┤
│  L5: Replica DB                           Lag: ms – sec         │
│      (read replicas lag behind primary by replication delay)    │
├─────────────────────────────────────────────────────────────────┤
│  L6: Primary DB                           source of truth       │
└─────────────────────────────────────────────────────────────────┘
```

Every horizontal line is a staleness boundary. A price change at L6 does not appear at L0 until it has propagated through — or invalidated — every layer above. "My CDN TTL is 5 minutes" is only one of six sources of drift.

A common mental-model bug: engineers think of cache as "a layer." It's not. It's a tree of independent stores with independent eviction policies, each of which can be ahead or behind any other. Fixing L1 without fixing L4 is a partial fix.

---

## 3. The Core Architectural Move: Separate Static from Volatile

The single most effective change: **stop caching price at the catalog layer.** Cache everything else about the product (name, description, images, specifications, category) aggressively at the CDN. Do not put price on the cacheable payload.

```
GET /api/products/sku-1234           ← cache heavily (24h)
{
  "sku": "sku-1234",
  "title": "Running Shoes",
  "images": [...],
  "description": "...",
  "category": "footwear"
  // NO PRICE HERE
}

GET /api/pricing/sku-1234            ← cache briefly or not at all
{
  "sku": "sku-1234",
  "currency": "USD",
  "price": 80.00,
  "was": 100.00,
  "valid_until": "2026-04-22T18:00:00Z",
  "version": 42
}
```

Two calls from the page. Product data is cheap to cache — it barely changes. Price is a separate, short-TTL (5s-30s), cheap-to-fetch call that can be routed to a different layer entirely.

Benefits:
- Product catalog remains heavily cacheable → CDN offload → low origin load → low cost.
- Price changes propagate in seconds, not TTL-duration.
- Invalidating pricing does not invalidate product content (which is expensive to re-render and re-fill).

Costs:
- One extra round trip. Mitigate with HTTP/2 multiplexing or a single `/catalog-view` endpoint that composes server-side (aggregator pattern).
- Pricing service must handle product-detail traffic volumes. Scale it for that — it's still cheaper than re-pushing full catalog bytes.

**Variant:** serve a "safe" price on the cached payload (a floor or "starting at $X") and require a separate uncached call for the binding price. Catalog pages show "from $99"; product page shows exact price via live fetch. This lets you keep the "from $99" in CDN edge while keeping the binding price fresh.

---

## 4. Binding-Price Contract: Display vs. Checkout

The second architectural move: **treat displayed price as advertising, and checkout price as contract.** Make the distinction explicit in code, in UX, and in legal terms.

The binding rule:
> Price shown on catalog/product pages is **indicative**. The price at checkout, computed at cart-validation time against the current source of truth, is **binding**.

Every mature e-commerce platform does a version of this. The UX is carefully designed so users are never surprised:

```
Product page:   "$80.00"       (may be cached; indicative)
Add to cart:    $80.00         (snapshot at add time, stored on cart line)
Cart page:      "$80.00"  +
                "⚠ Price may update at checkout"     (honest disclosure if stale)
Checkout page:  $80.00         (re-fetched from source, FROZEN on this page)
Order confirm:  $80.00         (written to immutable order)
```

Two server-side guarantees underneath:

1. **Revalidate at cart-load and at checkout-entry.** The cart line stores `price_at_add`; when loading the cart, compare against current price. If different, show "Price updated" UX and ask user to accept the new price before allowing checkout. This removes the entire "shown one price, charged another" class of complaint.
2. **Price-quote token on checkout.** When user enters checkout, the server mints a `price_quote` row with a 15-minute expiry, capturing the exact prices, taxes, and discounts. The order commits against the quote. If the quote expires mid-checkout, the user is re-prompted — not silently re-priced.

```sql
CREATE TABLE price_quotes (
    id              UUID PRIMARY KEY,
    cart_id         BIGINT NOT NULL,
    line_items      JSONB NOT NULL,     -- {sku, qty, unit_price, taxes, discounts}
    currency        TEXT NOT NULL,
    total           NUMERIC(20,4) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ NOT NULL,
    consumed_by_order UUID
);
```

At order-placement, the handler verifies `expires_at > now()` and atomically marks `consumed_by_order`. If expired, the order fails with `PRICE_QUOTE_EXPIRED` and the user re-confirms at the new price. This is the same pattern as OAuth authorization codes: mint → use-once → expire.

The point: you can tolerate a 5-minute CDN stale window for the catalog *because* catalog price is not the binding one. The binding price is bound briefly, in a separate server-authoritative flow.

---

## 5. Cache Invalidation Techniques (Ranked)

Given that *some* caching is still desirable, here are the strategies in order of operational maturity.

### 5.1 TTL-only — baseline, acceptable for static data, not for price

Just set `Cache-Control: max-age=300`. Accept staleness up to TTL.

Use it for: product descriptions, images, categories.
Don't use it for: price, stock, shipping cost, tax.

### 5.2 Active purge via CDN API — the right default for price

On price change in the admin tool, the pricing service fires a purge to the CDN for every affected URL.

```go
func OnPriceChange(sku string) error {
    // 1. Write to source of truth (DB).
    if err := pricingDB.UpdatePrice(sku, newPrice); err != nil {
        return err
    }
    // 2. Invalidate app-level cache synchronously.
    redis.Del(fmt.Sprintf("price:%s", sku))

    // 3. Invalidate CDN (async but audited).
    purgeEvents.Publish(PurgeEvent{
        URLs: []string{
            fmt.Sprintf("https://cdn.example.com/api/pricing/%s", sku),
            fmt.Sprintf("https://cdn.example.com/p/%s", sku),   // product page
        },
        SurrogateKey: fmt.Sprintf("sku-%s", sku),
    })
    return nil
}
```

**Watch-outs with CDN purge:**
- Purge is eventually consistent across POPs. Fastly claims sub-150ms globally; CloudFront is typically seconds; older CDNs are minutes. Measure your own propagation time end-to-end.
- Purge-by-URL requires knowing every URL that embeds the price. Purge-by-surrogate-key (Fastly, Cloudflare) is much stronger: tag every response with `surrogate-key: sku-1234` and purge the whole tag, hitting every URL that served that SKU.
- Purge floods on bulk catalog updates can rate-limit you. Batch them or use a key that groups SKUs (e.g., `surrogate-key: category-electronics` for a category-wide sale).

**Surrogate key pattern:**
```
GET /p/sku-1234 HTTP/1.1
← 200 OK
  Cache-Control: max-age=300
  Surrogate-Key: sku-1234 cat-electronics brand-nike

On price change:
  POST /purge { "key": "sku-1234" }   → every cached URL tagged with this key is invalidated across all POPs
```

### 5.3 Versioned URLs / cache-busting — for assets, not for price API

`/images/shoe-v42.jpg` is great for images. For price API endpoints, putting the version in the URL means every page render either (a) looks up the current version first (another round trip) or (b) renders with a stale version it embedded earlier. Net: you've just moved the staleness problem to "whatever computes the version."

### 5.4 Stale-while-revalidate — the best compromise for price API

`Cache-Control: max-age=5, stale-while-revalidate=30`. Cache fresh for 5s, serve stale for up to 30s more while fetching new data in the background.

- 99% of traffic gets a sub-ms cached response.
- Fresh data appears within 5s of a change, not 5 minutes.
- Origin load is bounded by the revalidation rate, not the request rate.

This is the sweet spot for pricing APIs: fresh-enough, cheap, and no thundering herd on the origin during price changes.

### 5.5 Push invalidation (pub-sub cache) — required at extreme scale

For price-sensitive pages at massive scale (marketplaces, travel), the CDN/app cache subscribes to a change stream:

```
pricing-service --writes--> Kafka(price_changes)
                                    │
              ┌─────────────────────┼────────────────┐
              ▼                     ▼                ▼
         app cache #1          app cache #2    edge-compute (Cloudflare Workers)
         (invalidate local)    (invalidate)    (evict edge cache)
```

Caches become eventually consistent with sub-second lag, not TTL-duration. The invariant shifts from "cache is correct" to "cache converges quickly and display price is non-binding."

### 5.6 Edge-side composition — for pages that must always show live price

Some pages (pricing ladders, flash-sale countdowns) must be live. Two options:

- **Edge-side includes (ESI / Cloudflare Workers / Fastly Compute@Edge).** Cache the page HTML shell with a placeholder; at edge-request time, fetch the live price from origin and inject. Separates "template cache" (hours) from "price" (no-cache).
- **Client-side hydration.** Serve a cached HTML shell; a small JS fetch replaces price before the user reads it. Risk: the user *did* read the pre-hydrated price if the JS is slow. Mitigation: render `---` placeholder until price loads, trading flash-of-empty-price for never-showing-stale-price.

---

## 6. The Full Mitigation Stack — What Production Actually Looks Like

For a serious e-commerce platform, combine these layers:

```
┌────────────────────────────────────────────────────────────┐
│  Catalog payload (cached 1h at CDN)                        │
│    Title, description, images, spec — NO price field       │
│    Surrogate-keyed by SKU for purge                        │
├────────────────────────────────────────────────────────────┤
│ Pricing payload (cached 5-30s w/ SWR at CDN, or not at all)│
│    Price, currency, sale flag, valid_until                 │
│    Active purge on change via surrogate key                │
│    Push-invalidated via Kafka at app cache level           │
├────────────────────────────────────────────────────────────┤
│  Cart-line snapshot: price_at_add (frozen on each line)    │
│    Revalidated on cart-load; UX prompt on change           │
├────────────────────────────────────────────────────────────┤
│  Price quote at checkout: server-minted, 15-min TTL        │
│    Atomically consumed by order; rejected if stale         │
├────────────────────────────────────────────────────────────┤
│  Order record: price_at_order (immutable)                  │
│    The legal price. Never re-computed.                     │
└────────────────────────────────────────────────────────────┘
```

Each layer has a different staleness tolerance and a different consequence-of-staleness. The catalog layer can be stale for minutes (nobody's buying yet). 
The pricing layer must be fresh within seconds. The cart and checkout layers cannot be "stale" at all — 
they are snapshots, not caches, and are explicitly revalidated against live data at transition points.

---

## 7. Edge Cases in the Edge Case

### 7.1 Price drops during checkout

User enters checkout at $100. Price drops to $80 halfway through. What do you show them?

**Option A: they get the better price.** Re-fetch on every checkout step; show "Great news — price dropped!" This is ethically clean and converts better, but requires every step of checkout to revalidate.

**Option B: they keep the original price.** The price quote was issued at $100; they pay $100. Simpler, and legally defensible (quote contract). But customer-hostile if they notice.

**Production choice:** most platforms pick A for drops (customer-friendly) and B for rises (quote-protective). Asymmetry is fine — the binding document is "the price we showed you at the quote moment or better."

### 7.2 Price rises during checkout

Same as above but inverse. Customer entered quote at $80, price rose to $100.
- Honor the quote until expiry (15 min).
- After expiry, re-quote at new price; prompt user to accept.
- **Never** silently charge the higher price.

### 7.3 Wishlist and saved-for-later

These are explicitly about "I want to know when price drops." You need a separate invalidation path: when a price drops on a SKU, fan-out to users who have it wishlisted, optionally with a notification. 
This is fundamentally a change-data-capture problem from the pricing table.

### 7.4 Comparison-shopping aggregators

Aggregators (Google Shopping, Kayak-style) scrape your feed on a schedule. If their crawl lands during a stale window, they advertise your stale price. A user clicks from the aggregator, lands on your site, sees the new (different) price.

Mitigations:
- **Feed format includes effective date range.** The aggregator knows when the quote is valid and can suppress display after expiry.
- **Deep-link behavior.** On arrival from an aggregator with a stale price in the URL, serve the live price but acknowledge the discrepancy: "Price has updated since you clicked. Now $80."

### 7.5 Locale and currency

The cache key must include locale + currency. `/pricing/sku-1234` is not a cache key; `/pricing/sku-1234?country=IN&currency=INR` is. Fail to include all dimensions and user B sees user A's currency — subtler than a stale price and often worse.

### 7.6 Flash-sale start at exact time

Sale begins at 9:00 AM. Stale caches continue serving old price past 9:00. First-mover advantage is real; a 30-second delay on a flash sale is lost revenue and negative press.

Technique: **pre-warm the new price into all caches ahead of time with `valid_at`**. The pricing payload includes `{ price: 80, valid_at: "2026-04-22T09:00:00Z" }`. The rendering layer (app or edge compute) picks "current valid price" based on server time at render. Cache can be written in advance; the switch-over is time-based, not cache-invalidation-based. This avoids the thundering purge at 9:00 AM.

### 7.7 Different users seeing different prices legitimately

Loyalty tiers, B2B contracts, segment-based promotions — these are real reasons the same SKU shows different prices to different users. 
Now your cache key must include the user-pricing-segment. You cannot cache at the edge the same way for everyone.

Solutions:
- **Vary by cookie / header.** `Vary: X-Pricing-Tier`. CDN will store separate variants. Works but cardinality grows.
- **Don't cache the pricing API for segmented users — fetch live.** Reserve edge caching for anonymous / public-tier traffic.
- **Segment-at-edge.** Edge compute reads the user's segment cookie and selects the right pre-cached variant.

---

## 8. Detection, Observability, and Alerts

You cannot fix what you cannot see. Instrument for *divergence*, not just for freshness.

**Core metric: display-vs-checkout price delta.**
```
% of orders where order.line.unit_price != cart_add.price_at_add
% of orders where order.line.unit_price != display_price_at_last_view (sampled)
```
Healthy baselines are <0.1%. A spike above 1% is a caching or pricing-service incident.

**Cache freshness probe.**
- Synthetic monitors write a known price, then poll each CDN POP and each app cache until the new price appears. p99 propagation time is a SLO.
- Alert if any POP lags by >60s after a purge.

**Surrogate-key purge audit.**
- Log every purge with its surrogate key, timestamp, and CDN-reported ack time.
- Daily reconcile: every price change in DB must have a corresponding purge event within 5 seconds.

**Pricing-API p99 freshness.**
- Measure `now() - response.X-Origin-Timestamp` on cached responses. This is *true* staleness per response, not theoretical TTL.
- Alert if p99 > expected TTL × 2.

**Price-quote expiry rate.**
- % of checkouts that see expired quotes. A spike = users are taking longer, OR checkout is failing and the retry fell outside the quote window, OR your quote TTL is too short.

**Customer-reported price mismatches.**
- A support ticket tag `price_mismatch` with a dashboard. This is slow (days) but catches everything instrumentation missed.

---

## 9. What to Say in the Design Review

When pushed on this in a review, the staff-level answer is the shape of the full solution, not a single cache parameter:

1. **Price is not cached with the catalog.** Separate endpoint, separate TTL, separate invalidation.
2. **Displayed price is indicative; checkout price is binding.** Enforced via a server-minted price quote with an expiry.
3. **Invalidation is push, not pull.** On price change, we purge by surrogate key across all cache layers (CDN, app cache, edge) as part of the same workflow that writes the DB.
4. **Stale-while-revalidate gives us fresh-within-seconds without thundering-herd.**
5. **Every order records its immutable price.** The DB is the final source of truth; caches can fail open without breaking correctness.
6. **We monitor for divergence, not just freshness.** Display-vs-charge price delta is a SLO.

The "5 minute TTL" in the original problem is a symptom. The cure is not "shorten the TTL" — the cure is "don't bind the customer contract to a TTL'd value in the first place."

---

## 10. Quick-Reference: Decision Matrix

| Dimension                       | Cache aggressively? | Where? | TTL |
|---------------------------------|---------------------|--------|-----|
| Product title, description      | Yes                 | CDN    | Hours |
| Product images                  | Yes                 | CDN    | Immutable (versioned URL) |
| Product category, breadcrumb    | Yes                 | CDN    | Hours |
| Price (public, anonymous)       | Limited             | CDN w/ SWR, app cache w/ push-invalidate | 5-30s |
| Price (tiered / logged-in)      | No CDN; app cache only | Redis | 5s |
| Stock count (exact)             | No                  | — | Live |
| Stock availability (in/out)     | Yes                 | App cache | 30s |
| Shipping cost estimate          | No (too dynamic)    | — | Live |
| Tax                             | No                  | — | Live |
| Cart line price (snapshot)      | N/A (immutable snapshot) | DB | — |
| Checkout price quote            | N/A (server-minted token) | DB | 15 min |
| Order price                     | N/A (immutable record)| DB | Forever |

The column "Cache aggressively?" is really asking: "is the staleness cost less than the cost of a live fetch?" For product content: yes, staleness is cheap. For price shown at the binding moment: no, staleness is a legal event.