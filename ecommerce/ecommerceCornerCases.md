E-Commerce: Distributed Systems Constraints & Corner Cases

1. Inventory Management

The Core Problem: Overselling

The single hardest problem in e-commerce. Two users buy the last item simultaneously.

T1: User A reads inventory(SKU-123) = 1
T2: User B reads inventory(SKU-123) = 1
T3: User A decrements → inventory = 0, order confirmed
T4: User B decrements → inventory = -1, order confirmed ← OVERSOLD

Corner cases:

- Flash sale thundering herd. 100,000 users hit "buy" on the same SKU within 1 second. Even with UPDATE inventory SET qty = qty - 1 WHERE qty > 0, the single row becomes a
  serialization bottleneck. Throughput collapses to ~1,000 TPS due to row-level lock contention.
    - Mitigation: Pre-split inventory into shards. 1,000 units → 10 rows of 100 each. Requests are distributed across shards. When a shard hits 0, it's skipped.
- Cart reservation expiry race. User adds the last item to cart. System reserves it for 15 minutes. At minute 14:59, user starts checkout. Reservation expires at 15:00 while
  payment is in flight. Another user grabs it. First user's payment succeeds but inventory is gone.
    - Mitigation: Extend reservation when checkout begins. Or treat payment capture as the reservation — don't reserve at cart-add time.
- Inventory sync across warehouses. Total inventory = 50 (30 in warehouse A, 20 in warehouse B). A regional outage disconnects warehouse B. The system thinks inventory = 30. It
  sells 30 units. Warehouse B comes back and also sold 15 during the partition. Total sold = 45 against 50 units — close, but the allocation is wrong. Some orders are assigned to the
  wrong warehouse.
- Return-to-inventory timing. Customer returns an item. Warehouse receives it, but the inventory system update is delayed (async event). During the delay, the item shows as
  out-of-stock. Lost sales for an item that's physically on the shelf.
- [low]Pre-order / backorder inventory. You sell 500 units of a pre-order item with an expected restock date. Supplier delivers only 300. Now you have 200 confirmed orders you can't
  fulfill. Which 200 do you cancel? FIFO? Loyalty tier? This is a business logic problem with distributed data dependencies.
- Bundle inventory. A bundle contains items A, B, C. Item A has 2 left, B has 100, C has 50. The bundle is limited to 2 — but nothing prevents someone from buying item A
  individually while a bundle order is in flight. A is now at 0 and the bundle can't be fulfilled.
- Inventory count drift. Systems accumulate rounding errors, missed events, and edge cases. After 6 months, the database says SKU-456 has 47 units but the warehouse physically has
43. Without periodic reconciliation (cycle counts), you oversell 4 units.

  ---
2. Payments & Checkout

The Core Problem: Exactly-Once Money Movement

Charging a customer twice is worse than not charging them at all. But distributed systems give you at-least-once or at-most-once, not exactly-once.

Corner cases:

- Double charge on timeout. Payment gateway takes 30 seconds (network issue). Client times out at 10 seconds and retries. Gateway processed the first request. Now two charges
  exist.
    - Mitigation: Idempotency key per checkout attempt. **Gateway deduplicates by key.** But: the idempotency key must be generated client-side before the first attempt, not server-side
      — otherwise the retry generates a new key.
- Payment succeeds, order creation fails. Stripe returns charge.succeeded. Your order service crashes before persisting the order. Customer is charged, no order exists.
    - Mitigation: Two approaches:
      i. Authorization + capture. Authorize first (no money moves). Create order. Then capture. If order creation fails, don't capture. Auth expires automatically.
      ii. Webhook reconciliation. Listen for charge.succeeded webhooks independently. If an orphaned charge exists without an order, either create the order or refund automatically.
- Partial payment failure in multi-tender. Customer pays $50 with gift card + $50 with credit card. Gift card deducted successfully. Credit card declined. Now you must reverse the
  gift card deduction. If the gift card service is down, the customer has lost $50 with no order.
    - Mitigation: Saga pattern with compensation. But compensation can also fail — you need a reconciliation job that catches stuck partial payments.
- Currency conversion race. Customer sees price as €85.00 (converted from $100 at rate 0.85). By the time they complete checkout 10 minutes later, the rate is 0.87. Do you charge
  €85 (honor the displayed price, absorb the loss) or €87 (charge the current rate, surprise the customer)?
- 3DS / SCA redirect failure. Customer is redirected to bank's 3D Secure page. Bank's page is slow. Customer closes the tab. Payment is in "pending_authentication" state. Cart is
  locked. Customer opens a new tab, tries again — but the old pending payment is blocking a new attempt for the same cart.
- Fraud check latency. Fraud detection takes 5 seconds (ML model + third-party API). Customer is waiting. At scale, this serializes the checkout flow. If the fraud service is down,
  do you: (a) block all orders, (b) let all orders through (risk fraud), or (c) let orders through up to a dollar threshold? Each choice has different failure characteristics.
- Gift card balance in distributed cache. Gift card balance is cached for performance. Customer checks balance ($50), cached. A concurrent phone order drains $30. Cache still shows
  $50. Customer tries to use $50 online. Real balance is $20. Charge fails at the payment layer but the customer saw $50.
- Refund after chargeback. Customer requests a refund. While refund is processing (24-48 hours), they also file a chargeback with their bank. Both succeed. Customer gets double
  money back.

  ---
3. Shopping Cart

The Core Problem: Cart is Session State That Must Survive Failures

Corner cases:

- Cart merge on login. Anonymous user adds items A, B to cart. Logs in. Their account already has items B, C in a saved cart. What's the merged cart? A, B, C? What if the anonymous
  cart had B with quantity 3 and the saved cart had B with quantity 1? Is the merged quantity 3 or 4 or 1?
- Price change while item is in cart. Customer adds item at $50. Leaves for 3 hours. Item goes on sale for $30. Customer checks out. Which price? If $50, the customer is angry. If
  $30, what about the 10,000 other carts that also have this item at $50 — do you retroactively adjust?
- Item goes out of stock while in cart. Customer adds item, browses for 30 minutes, goes to checkout. Item sold out 10 minutes ago. Do you show an error at checkout (bad UX) or
  silently remove it (surprising, potentially missed)? If it's a bundle, does the whole bundle become invalid?
- Cart storage at scale. 10M active carts, most abandoned. If stored in a database, that's significant write load for data that's 90% disposable. If stored in Redis, a node failure
  loses active carts mid-checkout. If stored client-side (cookie/localStorage), you can't enforce server-side invariants (price, stock).
- Multi-device cart sync. Customer adds items on mobile. Opens laptop. Expects to see the same cart. But the mobile app uses a local-first approach with eventual sync. If they add
  items on both devices before sync, you have a conflict resolution problem — CRDTs for shopping carts are non-trivial.
- Cart size limit and abuse. No limit → a bot adds 100,000 SKUs to a cart, reserving inventory and denying it to real customers. With limits → legitimate B2B customers can't place
  large orders.
- Coupon interaction with cart modification. Customer applies coupon "20% off if cart > $100." Cart is $120 → coupon applies → effective $96. Customer removes a $25 item → cart is
  $95 before discount → coupon condition no longer met → coupon removed → cart is $95. Customer adds a $10 item → cart is $105 → coupon re-applies → $84. This recalculation on every
  cart mutation is computationally expensive at scale and creates a confusing UX.

  ---
4. Product Catalog & Search

The Core Problem: Read-Heavy, Consistency-Tolerant, But Freshness-Sensitive

Corner cases:

- Search index lag. Product is created in the primary database. Elasticsearch index lags by 30 seconds. Customer searches for the product → not found. They refresh → found.
  Inconsistent experience. During flash sales, new items appearing with a 30-second delay means lost revenue.
- Stale cache serving wrong price. Product catalog is behind a CDN with 5-minute TTL. Price changes from $100 to $80. For 5 minutes, some users see $100, others see $80 (cache hit
  vs. miss). Users comparing notes on social media see different prices — trust erosion.
- Category tree inconsistency. Category "Electronics > Phones > Android" is restructured to "Electronics > Mobile > Android." During the migration, some products point to the old
  category path, some to the new. Search facets show both. Breadcrumbs are broken.
- Variant explosion. A T-shirt with 5 sizes × 8 colors × 3 fits = 120 variants. A configurable laptop with 4 CPUs × 3 RAM × 3 storage × 2 displays = 72 variants. Each variant has
  its own inventory, price, and images. Catalog queries that join product → variant → inventory become expensive. At 1M products × 100 variants, that's 100M rows.
- Search relevance vs. inventory. Customer searches "blue running shoes." The most relevant result is out of stock. Do you show it (relevant but frustrating), hide it (less
  frustrating but the search results seem worse), or show it grayed out (more UI complexity, requires real-time inventory check per search result — expensive)?
- Internationalization of catalog. Same product, different name/description in 20 languages, different prices in 30 currencies, different availability in 15 countries, different
  legal restrictions (can't sell alcohol in Saudi Arabia, certain electronics in Russia). Every catalog query now has a (locale, currency, country) dimension that multiplies storage,
  cache cardinality, and query complexity.
- Product delisting race. A product is delisted (legal issue, recall). It must vanish from search, category pages, direct URLs, carts, and wishlists — simultaneously. But each of
  these is a different system (Elasticsearch, CDN, cart service, wishlist service). The direct URL might be cached in Google's search results for weeks.

  ---
5. Order Management

The Core Problem: An Order Is a Distributed Saga Across Many Services

Order Lifecycle (happy path):
Cart → Checkout → Payment Auth → Inventory Reserve → Order Created
→ Payment Capture → Warehouse Notified → Picked → Packed → Shipped
→ Delivered → (Return Window) → Closed

Each arrow is a potential failure point.

Corner cases:

- Order state machine divergence. The order service says "shipped." The warehouse service says "picking." The payment service says "captured." Which is the source of truth? If each
  service maintains its own view of order state, they can diverge permanently after a missed event.
- Split shipment complexity. An order has 3 items. Item A ships from warehouse 1, items B+C from warehouse 2. The customer sees 2 shipments. If item A is lost in transit, do you
  refund just item A or wait for B+C? If the customer returns B, the refund calculation must account for split shipping costs, proportional discounts, and tax across different
  jurisdictions.
- Order edit after placement. Customer places an order and immediately wants to change the shipping address. But the order event has already been published. The warehouse has
  already started processing. The payment was authorized for the original tax calculation (different state = different tax). Editing an in-flight order means compensating across
  payment, tax, inventory, and fulfillment — each in a different service.
- Duplicate order on retry. "Place order" button clicked twice. Two orders created with the same items. If you deduplicate by cart contents, what about legitimate repeat orders
  (customer intentionally ordering the same thing again)?
    - Mitigation: **Checkout token** — a one-time-use token generated when the checkout page loads. Second click reuses the token, server rejects it.
- Order archival and retrieval. Active orders are in a hot database. After 90 days, they're archived to cold storage. Customer calls support about an order from 2 years ago.
  Support tool queries cold storage — 5-second latency. Or worse: the archive format has changed twice since then, and the deserialization fails.

  ---
6. Pricing, Promotions & Tax

The Core Problem: Price Calculation Is Contextual, Non-Deterministic, and Must Be Reproducible

Corner cases:

- Promotion stacking abuse. Coupon A: 20% off. Coupon B: $10 off orders > $50. Loyalty discount: 5%. Free shipping for orders > $75. A carefully constructed cart can yield negative
  prices or near-zero totals. Each promotion was designed independently — their interaction was never tested.
- Price consistency across the funnel. Product listing page shows $45. Product detail page shows $42 (different cache). Cart shows $45. Checkout shows $43. Each page queries a
  different service or cache tier with different staleness. The customer screenshots the discrepancy and tweets it.
- Tax calculation at scale. US has ~12,000 tax jurisdictions. Tax rate depends on: product category, buyer location, seller nexus, product origin, buyer tax exemption status, and
  sometimes the specific product (food is exempt in some states, but candy isn't — and the definition of "candy" varies by state). Tax calculation is an external API call (Avalara,
  TaxJar). At 10,000 checkouts/minute, that's 10,000 API calls/minute to a third party. If it's down, checkout is blocked.
- Promotion timezone ambiguity. "Sale ends Friday." Which timezone? The customer is in Tokyo, the server is in US-East, the promotion was created by a merchandiser in London. The
  sale ends at 3 different times depending on perspective.
- Historical price for returns/disputes. Customer disputes a charge from 3 months ago. You need to reproduce the exact price calculation: base price at time of purchase + promotion
  that was active + tax rate at that time + shipping cost. If you only stored the final total, you can't justify the breakdown. If any of those data sources have changed (promotion
  deleted, tax rate updated), reconstruction is impossible.
- Rounding errors across currencies. $10.00 / 3 items = $3.333... per item. Do you charge $3.33 × 3 = $9.99 (you lose $0.01) or $3.34 × 3 = $10.02 (you overcharge $0.02)? At
  millions of transactions, these pennies add up. Different jurisdictions have different rounding rules. Some currencies have no cents (JPY).
- Dynamic pricing race. Pricing engine adjusts prices based on demand. High traffic → price goes up. But the price change propagates to the catalog with a delay. 1,000 users see
  the old price. 500 add to cart. You either honor the old price (lose margin) or show an error at checkout (lose trust).

  ---
7. User Sessions & Authentication

Corner cases:

- Session fixation across login. Anonymous session S1 has items in cart. User logs in → session upgraded to authenticated. If the session ID doesn't rotate, an attacker who planted
  S1 (via URL, XSS) now has an authenticated session.
- Token refresh race. JWT expires. Two concurrent requests both detect expiry. Both call the refresh endpoint. Refresh endpoint invalidates the old refresh token. First request
  gets a new token pair. Second request fails (old refresh token already invalidated). User sees an intermittent 401.
- Geo-distributed session store. User is in Singapore, session stored in US-East. Every request has 200ms latency just for session lookup. You replicate sessions to SG-region → now
  session writes must propagate across regions. User logs out in one tab (US-East session deleted). Another tab in the same browser hits SG-region, session still exists (replication
  lag). User isn't logged out.
- Account takeover + active order. An account is compromised. The attacker changes the shipping address and email. A legitimate order is now being shipped to the attacker's
  address. By the time the real user notices, the package is in transit. Reversing this requires coordination between identity, order, shipping, and fraud — across system boundaries.
- Guest checkout to account conversion. User checks out as guest (email: alice@example.com). Later creates an account with the same email. How do you link the guest order to the
  new account? What if there are 5 guest orders from the same email but 2 are actually a different person sharing the email (family)?

  ---
8. Shipping & Fulfillment

Corner cases:

- Shipping rate calculation dependency. Rate depends on: origin warehouse (not known until inventory allocation), package dimensions (not known until picked/packed), carrier
  availability, delivery speed, and destination. At checkout time, some of these are estimates. Actual shipping cost can differ from quoted cost. Who absorbs the difference?
- Address validation failure. Customer enters "123 Main St Apt 4B." USPS says it doesn't exist. UPS says it does. FedEx says it exists but the ZIP is wrong. Different carriers have
  different address databases. You validate against one carrier's API, then ship with another.
- Multi-warehouse routing optimization. Order has 3 items. Item A is in warehouses {1, 2}. Item B is in {2, 3}. Item C is in {1, 3}. Optimal is to ship A+C from warehouse 1, B from
  warehouse 3 (2 shipments). But if warehouse 1 is at capacity, you ship A from 2, B from 3, C from 1 (3 shipments — higher cost). This is a constraint satisfaction problem that
  must be solved per-order at checkout speed.
- Carrier API outage. FedEx API is down. You can't generate shipping labels. Orders are placed but can't ship. Backlog accumulates. FedEx API comes back. Now 10,000 label
  generation requests hit simultaneously. FedEx rate-limits you. The backlog drains slowly while SLA timers tick.
- Delivery promise vs. reality. You promise "delivery by Thursday" based on carrier SLA estimates. Carrier misses the SLA. Customer contacts support. You have no real-time
  visibility into the carrier's internal pipeline — tracking data updates hours behind reality.

  ---
9. Notifications & Communication

Corner cases:

- Notification ordering. Order placed → payment confirmed → shipped. If the "shipped" email is delivered before the "order confirmed" email (different services, different queue
  latencies), the customer sees a shipment notification for an order they haven't been told about.
- Email/SMS delivery deduplication. A retry sends "Your order is confirmed" twice. Customer thinks they were charged twice. Or worse: a marketing email blast retries and sends 3
  copies to 2M users — 6M emails, reputation damage, potential CAN-SPAM violations.
- Notification preference propagation. Customer unsubscribes from marketing emails. The unsubscribe event is processed by the marketing service but hasn't propagated to the
  transactional email service's suppression list. A promotional insert in a transactional email still goes out. Legal risk under GDPR/CAN-SPAM.
- Webhook delivery guarantees. You expose webhooks for order events to merchants/partners. Their endpoint is down. You retry with exponential backoff. After 3 days of retries, you
  give up. The merchant never learns about the order. Contractual SLA is violated. But unbounded retry means unbounded queue growth.

  ---
10. Search, Recommendations & Personalization

Corner cases:

- Cold start problem. New user, no purchase history, no browsing history. Recommendations are generic. Conversion rate for new users is 3x lower. But collecting enough signal to
  personalize takes 5-10 sessions. During that period, you're essentially guessing.
- Feedback loop / filter bubble. User buys running shoes. Recommendations show more running shoes. User only sees running shoes. They never discover the hiking boots they would
  have loved. Revenue from that user plateaus because you're optimizing for local maxima.
- Recommendation freshness. ML model is retrained daily. A product goes viral on TikTok at 2 PM. The model won't learn about it until the next training run at midnight. For 10
  hours, the hottest product isn't recommended.
- A/B test interaction effects. You run A/B test on search ranking and another on checkout flow simultaneously. A user in the "better search" group finds products faster and buys
  more. The checkout test attributes this lift to the new checkout flow. Both tests report positive results, but only one actually caused the lift. At scale, dozens of concurrent
  tests create a combinatorial interaction space that's nearly impossible to attribute correctly.

  ---
11. Platform Scalability Constraints

Corner cases:

- Database connection exhaustion during peak. Normal: 50 app servers × 20 connections = 1,000 connections. Black Friday: autoscaler adds 200 app servers → 250 × 20 = 5,000
  connections. PostgreSQL max_connections = 2,000. New instances can't connect. Orders fail.
    - Mitigation: PgBouncer / ProxySQL for connection pooling at the proxy layer, not the app layer.
- Hot partition in the orders table. Orders are partitioned by created_date. On Black Friday, all orders go to today's partition. That single partition handles 100x normal write
  load. The other 364 partitions are idle.
- CDN cache invalidation propagation. You update a product image. CDN has it cached at 200 edge locations worldwide. Invalidation takes 5-30 seconds to propagate. During that
  window, some users see the old image, some the new. For time-sensitive changes (legal takedown, incorrect pricing), this window is a liability.
- Rate limiting per tenant in a marketplace. Seller A uses the API responsibly (10 req/s). Seller B scripts 10,000 req/s to scrape competitor pricing. 
  Global rate limiting punishes A along with B. Per-seller rate limiting requires tracking state for millions of sellers — itself a scalability problem.
- Event bus backpressure. Order events → inventory service, shipping service, notification service, analytics service, tax service. Each consumer processes at different speeds. The
  slowest consumer (analytics, doing heavy aggregation) falls behind. If using Kafka, the consumer lag grows. If using a push-based system (SNS+SQS), the SQS queue grows
  unboundedly. Either way, the slow consumer becomes a ticking time bomb — if it crashes and needs to replay, the replay takes hours.
- Autoscaling lag during flash sales. Traffic goes from 1,000 RPS to 50,000 RPS in 60 seconds (sale goes live). Autoscaler detects the spike at T+30s, requests new instances at
  T+35s, instances boot at T+90s, health checks pass at T+120s, traffic routes at T+125s. For 2 minutes, existing instances absorb 50x load. They either shed load (503s) or die
  (OOM). The autoscaler scales based on "desired" capacity but the actual capacity lags behind reality.
- Data locality and cross-region latency. US customer buys from EU merchant on a global marketplace. Inventory is in EU database. Payment is processed via US gateway. Tax
  calculation hits EU tax service. Each cross-region hop adds 80-150ms. Checkout takes 3+ seconds just from network latency.

  ---
12. Data Consistency Across Microservices

Corner cases:

- Eventual consistency user experience. Customer places order (order service). Redirected to "my orders" page (read from order query service). Order isn't there yet (event not
  propagated). Customer panics, places order again. Now two orders.
    - Mitigation: After successful order creation, redirect to a page that reads from the primary (not the replica/read model). Or return the order ID in the response and poll for it
      specifically.
- Saga rollback partial visibility. A saga creates an order, reserves inventory, and charges payment. Payment fails → saga compensates by releasing inventory and cancelling the
  order. But between "order created" and "order cancelled," the customer sees the order in "my orders." They call support about an order that's about to be cancelled.
- Cross-service join. "Show me all orders with their shipping status and payment status." This requires joining data from order service, shipping service, and payment service. You
  can't do a SQL join across microservices. Options:
  a. API composition (N+1 queries, high latency)
  b. Denormalized read model (CQRS — but now you maintain a projection)
  c. Shared database (defeats the purpose of microservices)
- Orphaned data after service failure. Order service creates an order. Publishes "OrderCreated" event. Inventory service processes it and reserves stock. Shipping service processes
  it and creates a shipment. Order service crashes and loses the order (database corruption). Inventory and shipping now have records for a non-existent order.
- Event ordering across aggregates. "CustomerAddressChanged" and "OrderShipped" are published from different services. If "OrderShipped" is processed before
  "CustomerAddressChanged," the package goes to the old address. But these events have no causal ordering because they come from different aggregates.

  ---
13. Security & Abuse

Corner cases:

- Coupon brute forcing. Coupons are 8-character alphanumeric codes. An attacker scripts POST /apply-coupon with random codes at 1,000 req/s. At 36^8 = ~2.8 trillion possibilities
  it's infeasible — unless your codes are sequential (PROMO001, PROMO002...) or short (6 chars = 2.1B).
- Price manipulation via API. Customer intercepts the checkout API call, changes the price from $100 to $1. If the server trusts the client-sent price instead of looking it up from
  the catalog, the order is processed at $1. **Server must always recompute the price from canonical sources.**
- Inventory denial-of-service. Bot adds all inventory of a hot item to carts, never checks out. Legitimate customers see "out of stock." Cart reservation timeout is 15 minutes. Bot
  re-adds every 14 minutes. Inventory is effectively locked out from real customers indefinitely.
- Enumeration attacks. /api/orders/12345 returns order details. An attacker iterates 12345, 12346, 12347... and scrapes all orders. Even with authentication, if the authorization
  check is "is the user logged in?" instead of "does this order belong to this user?", any authenticated user can read any order (IDOR).
- PII in logs and events. An order event includes customer name, address, email, phone, and payment method. This event flows through Kafka, is consumed by 8 services, stored in
  Elasticsearch for debugging, and replicated to a data lake. PII is now in 10+ systems. A GDPR "right to deletion" request means purging PII from all of them — including Kafka
  (immutable log, can't delete individual messages).

  ---
14. Observability & Incident Response

Corner cases:

- Metric lag masking revenue loss. Orders/minute metric is delayed by 2 minutes (aggregation pipeline lag). Revenue drops to zero. You detect it 2 minutes late. At $50K/minute
  during Black Friday, that's $100K lost before anyone even looks at a dashboard.
- Distributed tracing gaps. A request flows through 12 services. Service 7 (legacy, no tracing instrumentation) breaks the trace. You see two disconnected traces: services 1-6 and
  services 8-12. You can't correlate them. The root cause is in service 7 — the one you can't see.
- Alert storm during cascading failure. Database latency spikes → every service that depends on it fires alerts → PagerDuty sends 200 alerts in 3 minutes → on-call is paralyzed.
  The root cause (a single slow query) is buried under derivative alerts.
- Cost observability lag. You auto-scale to 500 instances during peak. Peak passes. Scale-down is conservative (cooldown period). You run 500 instances for 2 hours at $0.50/hr =
  $500 extra. The cost alert fires the next day when the AWS bill is tabulated. By then, the money is spent.

  ---
15. Compliance & Legal

Corner cases:

- GDPR right to deletion vs. financial record retention. Customer requests deletion of all their data. But tax law requires you to retain transaction records for 7 years. You must
  delete PII while retaining the financial record — pseudonymization of existing records across every system that has a copy.
- PCI DSS scope creep. A developer logs the full HTTP request body for debugging. The request body contains a credit card number. The log system is now in PCI scope. The entire
  logging infrastructure must meet PCI DSS requirements — or the log must be sanitized before storage. If this isn't caught immediately, the audit finding is a major compliance
  violation.
- Cross-border data residency. EU customer data must stay in EU (GDPR). Indian customer data must stay in India (PDPA). But your recommendation engine is in US-East and needs
  purchase history from all regions to train the model. Federated learning or anonymized data exports add latency and complexity.
- Accessibility (ADA/WCAG) at scale. A visually impaired customer uses a screen reader. Your dynamically-rendered product page (React SPA) has no ARIA labels. The screen reader
  reads "button button button image image link link." The customer can't use your site. At scale, this is a lawsuit vector (ADA Title III — thousands of cases per year against
  e-commerce sites).

  ---
Cross-Cutting Summary: Where Things Break Simultaneously

┌─────────────────────────┬──────────────────────────────────────────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│        Scenario         │            Services Affected             │                                              Why It's Hard                                              │
├─────────────────────────┼──────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Flash sale launch       │ Inventory, cart, payments, CDN,          │ 100x traffic in seconds, every service at breaking point simultaneously                                 │
│                         │ autoscaling, search                      │                                                                                                         │
├─────────────────────────┼──────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Payment provider outage │ Checkout, orders, fulfillment,           │ No fallback — you can't charge customers through a backup provider without PCI re-certification         │
│                         │ notifications                            │                                                                                                         │
├─────────────────────────┼──────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Database failover       │ Everything                               │ Even a 30-second failover causes connection pool exhaustion, transaction rollbacks, and stale reads     │
│                         │                                          │ across every service                                                                                    │
├─────────────────────────┼──────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Region-wide cloud       │ Everything                               │ DR failover requires pre-tested runbooks. Most companies discover their DR doesn't work during the      │
│ outage                  │                                          │ actual disaster                                                                                         │
├─────────────────────────┼──────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Data corruption in      │ All consumers of the corrupted events    │ Poisoned data propagates to 10+ services before detection. Rollback requires compensating events to     │
│ event bus               │                                          │ each consumer                                                                                           │
├─────────────────────────┼──────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ DDoS attack             │ CDN, API gateway, auth, all downstream   │ Legitimate and attack traffic are interleaved. Blocking too aggressively drops real orders. Blocking    │
│                         │ services                                 │ too loosely lets the attack through                                                                     │
└─────────────────────────┴──────────────────────────────────────────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────┘

The recurring theme: in e-commerce, every edge case has a direct revenue impact. A 500ms latency increase reduces conversion by ~1%. A 1-minute checkout outage during Black Friday
costs six figures. The constraints aren't theoretical — they're measured in money lost per second.