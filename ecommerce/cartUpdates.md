
Cart storage at scale. 10M active carts, most abandoned. If stored in a database, that's significant write load for data that's 90% disposable. If stored in Redis, a    
node failure                                                                                                                                                             
loses active carts mid-checkout. If stored client-side (cookie/localStorage), you can't enforce server-side invariants (price, stock).

⏺ Cart Storage at Scale — Deep Dive

The core tension: carts are write-heavy, read-heavy, and mostly worthless. 90% of them will be abandoned, but the 10% that convert are the entire revenue of the
business. Every storage choice optimizes for one axis at the cost of another.

The Numbers That Drive the Problem

Typical e-commerce at scale:

Active carts:            10M
Cart updates/sec:        50,000  (add item, change qty, remove, apply coupon)
Carts that convert:      ~10%   (1M)
Carts abandoned < 1hr:   ~60%   (6M)  ← pure waste if stored durably
Carts abandoned < 24hr:  ~85%   (8.5M)
Average cart size:        ~2KB   (5 items × ~400 bytes each)

Storage footprint:       10M × 2KB = 20GB active state
Write throughput:        50K writes/sec × 2KB = 100MB/sec sustained

For that 100MB/sec of writes, only ~10MB/sec actually matters.
The other 90MB/sec is writing data that will never be read again.

Option 1: Relational Database — What Actually Breaks

PostgreSQL as cart storage:

                          ┌────────────────────────────┐
                          │       carts table           │
                          │  50K writes/sec             │
                          │  10M rows                   │
                          └─────────────┬──────────────┘
                                        │
                      ┌─────────────────┼──────────────────┐
                      │                 │                   │
                      ▼                 ▼                   ▼
               WAL amplification   Row-level locks    Vacuum pressure

-- The schema looks simple enough
CREATE TABLE carts (
id          UUID PRIMARY KEY,
user_id     UUID,
session_id  VARCHAR(64),
items       JSONB,           -- [{product_id, qty, price_at_add, ...}]
coupon_code VARCHAR(32),
updated_at  TIMESTAMPTZ,
expires_at  TIMESTAMPTZ
);

CREATE INDEX idx_carts_user ON carts(user_id) WHERE user_id IS NOT NULL;
CREATE INDEX idx_carts_expires ON carts(expires_at);

Problem 1: WAL write amplification

Every cart update:
1. Write to WAL (write-ahead log)           → disk I/O
2. Update heap tuple (PostgreSQL is MVCC,    → creates a NEW row version
so every UPDATE is really INSERT + mark old dead)
3. Update JSONB index if you have one        → more WAL
4. Replicate WAL to standby                  → network I/O

A single "change quantity from 2 to 3" on a 2KB cart row:
→ ~6KB of actual disk I/O (WAL + heap + index)

    At 50K writes/sec: 50,000 × 6KB = 300MB/sec of disk I/O
    For a cart that has a 90% chance of being abandoned.

Problem 2: MVCC bloat and vacuum death spiral

PostgreSQL MVCC: UPDATE doesn't overwrite, it creates a new version.
Old versions are "dead tuples" cleaned by VACUUM.

At 50K updates/sec:
→ 50K dead tuples/sec created
→ autovacuum must clean 50K dead tuples/sec just to keep up

If vacuum falls behind (it will during traffic spikes):
→ Table bloats (10M rows, but 50M dead tuples on disk)
→ Index bloat follows (indexes point to dead tuples)
→ Sequential scans slow down
→ Disk usage grows unbounded
→ "VACUUM is running but the table keeps growing" — classic Postgres failure mode

This is why Uber famously moved off PostgreSQL — MVCC write amplification
on high-update workloads.

Problem 3: Connection exhaustion

50K cart writes/sec ÷ ~5ms per write = need ~250 concurrent connections

But PostgreSQL connection handling:
- Each connection = OS process (~10MB RAM)
- 250 connections = 2.5GB just for connection overhead
- max_connections above 200-300 degrades performance
- Contention on shared buffer pool, lock manager

Even with PgBouncer in transaction mode:
- Still 250 active transactions competing for buffer pool
- Lock contention on hot rows (user updating same cart rapidly)

When a DB is appropriate for carts: low-to-moderate scale (< 5K writes/sec), or when cart data must be transactionally consistent with other tables (inventory
reservation at add-to-cart time).

Option 2: Redis — What Actually Breaks

Redis as cart storage:

    Strengths:                    Weaknesses:
    ✓ 100K+ ops/sec per node     ✗ RAM-bound (expensive at scale)
    ✓ Sub-millisecond latency    ✗ Persistence has tradeoffs
    ✓ Hash type fits carts       ✗ Node failure = data loss window
    ✓ Built-in TTL/expiry        ✗ No complex queries

// Redis cart storage using Hash type
type RedisCartStore struct {
client *redis.Client
ttl    time.Duration // e.g., 7 days
}

func (s *RedisCartStore) AddItem(ctx context.Context, cartID string, item CartItem) error {
data, _ := json.Marshal(item)
pipe := s.client.Pipeline()

      // HSET cart:{id} {product_id} {json}
      pipe.HSet(ctx, cartKey(cartID), item.ProductID, data)
      // Reset TTL on every interaction
      pipe.Expire(ctx, cartKey(cartID), s.ttl)
      _, err := pipe.Exec(ctx)
      return err
}

func (s *RedisCartStore) GetCart(ctx context.Context, cartID string) ([]CartItem, error) {
result, err := s.client.HGetAll(ctx, cartKey(cartID)).Result()
if err != nil {
return nil, err
}

      items := make([]CartItem, 0, len(result))
      for _, v := range result {
          var item CartItem
          json.Unmarshal([]byte(v), &item)
          items = append(items, item)
      }
      return items, nil
}

func cartKey(cartID string) string { return "cart:" + cartID }

Problem 1: The persistence dilemma

Redis persistence options:

RDB snapshots (point-in-time dumps):
- Snapshot every 60 seconds
- Node crashes at second 59 → lose 59 seconds of cart updates
- During peak: 59 sec × 50K writes/sec = up to 2.95M cart operations lost
- Users mid-checkout see empty carts

AOF (append-only file):
- appendfsync always  → fsync every write, defeats purpose of Redis (drops to ~1K ops/sec)
- appendfsync everysec → lose up to 1 second of data (50K operations)
- appendfsync no      → OS decides when to flush, could lose minutes

    Even with AOF everysec, a crash during Black Friday:
      50,000 writes/sec × 1 sec = 50,000 users lose their carts
      If 10% were about to checkout → 5,000 lost conversions
      At $50 average → $250,000 revenue at risk

Problem 2: Redis Cluster failover gap

Timeline of a node failure in Redis Cluster:

00:00.000  - Primary node for slots 5461-10922 crashes
00:00.000  - Writes to carts on those slots start failing
00:01.000  - Sentinel/Cluster detects node is unreachable (min 1s)
00:01.000  - Failover vote begins among remaining primaries
00:02.000  - Replica promoted to primary
00:02.000  - Clients discover new topology
00:02.500  - Writes resume

              ┌─────────────────────────────────────┐
              │  2.5 second window of total outage   │
              │  for 1/3 of all carts                │
              │                                      │
              │  + any data not yet replicated from   │
              │    old primary to replica is GONE     │
              └─────────────────────────────────────┘

With async replication (default):
- Primary ACKs the write to your app
- Then replicates to replica asynchronously
- If primary dies before replication: data is lost permanently
- This is NOT eventual consistency — it's data loss

Problem 3: Memory cost

10M carts × 2KB average = 20GB of data
Redis overhead per key: ~100-200 bytes (dict entry, SDS headers, etc.)
10M carts × 5 hash fields each × 150 bytes overhead = ~7.5GB overhead

Total memory: ~28GB

Redis Cluster with replication (3 primary + 3 replica):
Each primary holds 1/3 of data: ~9.3GB
Replica mirrors primary: another 9.3GB
Total cluster memory: ~56GB

At AWS ElastiCache pricing (~$0.068/GB/hr for r6g):
56GB × $0.068 = $3.81/hr = $2,780/month

    Just for cart storage.
    And 90% of that is abandoned carts.

Option 3: Client-Side Storage — What Actually Breaks

localStorage cart:

    Browser stores: [{productId: "P1", qty: 2, price: 29.99}, ...]

    Strengths:                    Weaknesses:
    ✓ Zero server cost            ✗ No server-side validation
    ✓ Infinite "scale"            ✗ Lost on browser clear / device switch
    ✓ Works offline               ✗ No cross-device sync
    ✓ No abandoned cart cleanup   ✗ Price/stock manipulation

The real danger: client-side price and stock manipulation

Attack scenario:

1. User opens product page, item costs $500
2. JavaScript adds to cart: {productId: "P1", qty: 1, price: 500.00}
3. User opens browser DevTools → Application → localStorage
4. Edits price: {productId: "P1", qty: 1, price: 0.01}
5. Proceeds to checkout
6. Checkout API receives cart from client with price $0.01

If your checkout API trusts client-sent prices:
→ You just sold a $500 item for $0.01

If your checkout API looks up current prices:
→ "Why is it $500? It said $29.99 when I added it!"
→ Price changed between add-to-cart and checkout
→ No server-side record of what price the user was shown

Stock manipulation scenario:

1. Item has 2 units in stock
2. User A adds 2 to client-side cart
3. User B adds 2 to client-side cart
4. Server has no idea 4 units are "reserved"
5. Both go to checkout → one of them gets an error at the last step
   (worst possible UX — they filled out address, payment, everything)

The Real Solution: Tiered Hybrid Architecture

No single storage layer works. Production systems use a tiered approach where the storage durability scales with the cart's proximity to conversion.

                      Cart Lifecycle Stages

      ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
      │   BROWSING   │───→│   ENGAGED    │───→│  CHECKOUT    │
      │  (cold cart) │    │ (warm cart)  │    │ (hot cart)   │
      └──────────────┘    └──────────────┘    └──────────────┘

      Add first item       Add 2+ items /       Click "Checkout"
                           return within 1hr

      Storage: Cookie +    Storage: Redis        Storage: Redis + DB
      localStorage         (with TTL)            (write-through)

      Durability: None     Durability: Best      Durability: Full ACID
      Cost: $0             effort                Cost: Highest
                           Cost: Moderate

      Loss impact: Low     Loss impact: Medium   Loss impact: Revenue
      (user just browsing) (user annoyed)        (lost sale)

type TieredCartStore struct {
redis    *redis.Client
db       *sql.DB
metrics  *prometheus.CounterVec
}

type CartStage string

const (
StageBrowsing CartStage = "browsing" // just added first item
StageEngaged  CartStage = "engaged"  // multiple items or returned to cart
StageCheckout CartStage = "checkout" // clicked checkout button
)

// AddItem stores based on current cart stage.
func (s *TieredCartStore) AddItem(ctx context.Context, cartID string, item CartItem) error {
stage := s.determineStage(ctx, cartID)

      switch stage {
      case StageBrowsing:
          // Redis only, short TTL. If lost, user barely noticed.
          return s.redisAddItem(ctx, cartID, item, 2*time.Hour)

      case StageEngaged:
          // Redis with longer TTL. User has shown intent.
          return s.redisAddItem(ctx, cartID, item, 7*24*time.Hour)

      case StageCheckout:
          // Redis + synchronous DB write. Cannot lose this.
          if err := s.redisAddItem(ctx, cartID, item, 24*time.Hour); err != nil {
              // Redis failure during checkout — fall through to DB only
              log.Warn("redis write failed during checkout, falling to DB", "err", err)
          }
          return s.dbAddItem(ctx, cartID, item)

      default:
          return s.redisAddItem(ctx, cartID, item, 2*time.Hour)
      }
}

// PromoteToCheckout is called when the user clicks "Checkout".
// Atomically moves the cart from Redis-only to Redis+DB.
func (s *TieredCartStore) PromoteToCheckout(ctx context.Context, cartID string) error {
// Read full cart from Redis
items, err := s.redisGetCart(ctx, cartID)
if err != nil {
return fmt.Errorf("failed to read cart for promotion: %w", err)
}

      // Validate server-side invariants BEFORE persisting
      validated, err := s.validateCart(ctx, items)
      if err != nil {
          return fmt.Errorf("cart validation failed: %w", err)
      }

      // Write to database (ACID, survives Redis failure)
      tx, err := s.db.BeginTx(ctx, nil)
      if err != nil {
          return err
      }
      defer tx.Rollback()

      _, err = tx.ExecContext(ctx,
          `INSERT INTO checkout_carts (id, items, promoted_at, expires_at)
           VALUES ($1, $2, NOW(), NOW() + INTERVAL '30 minutes')
           ON CONFLICT (id) DO UPDATE SET items = $2, promoted_at = NOW()`,
          cartID, validated)
      if err != nil {
          return err
      }

      return tx.Commit()
}

func (s *TieredCartStore) determineStage(ctx context.Context, cartID string) CartStage {
// Check if cart is already in checkout (DB flag)
var inCheckout bool
s.db.QueryRowContext(ctx, `SELECT EXISTS(SELECT 1 FROM checkout_carts WHERE id = $1 AND expires_at > NOW())`, cartID).Scan(&inCheckout)
if inCheckout {
    return StageCheckout
}

      /*Check engagement signals in Redis*/
      itemCount, _ := s.redis.HLen(ctx, cartKey(cartID)).Result()
      if itemCount >= 2 {
          return StageEngaged
      }

      return StageBrowsing
}

Server-Side Price Locking (Solving Client Manipulation)

Never trust client-sent prices. But you also need to show the user a consistent price throughout their session.

// When an item is added to cart, lock the price server-side.
func (s *TieredCartStore) redisAddItem(ctx context.Context, cartID string, item CartItem, ttl time.Duration) error {
// ALWAYS look up the current price server-side.
// The client sends product_id and quantity only.
currentPrice, err := s.catalog.GetPrice(ctx, item.ProductID)
if err != nil {
return fmt.Errorf("price lookup failed: %w", err)
}

      // Store the server-verified price
      cartItem := StoredCartItem{
          ProductID:  item.ProductID,
          Quantity:   item.Quantity,
          PriceAtAdd: currentPrice,        // server-verified
          AddedAt:    time.Now(),
      }

      data, _ := json.Marshal(cartItem)
      pipe := s.redis.Pipeline()
      pipe.HSet(ctx, cartKey(cartID), item.ProductID, data)
      pipe.Expire(ctx, cartKey(cartID), ttl)
      _, err = pipe.Exec(ctx)
      return err
}

// At checkout time, re-validate everything.
func (s *TieredCartStore) validateCart(ctx context.Context, items []StoredCartItem) ([]byte, error) {
for i, item := range items {
// Re-fetch current price
currentPrice, err := s.catalog.GetPrice(ctx, item.ProductID)
if err != nil {
return nil, fmt.Errorf("product %s no longer available", item.ProductID)
}

          // Check for price changes since add-to-cart
          if currentPrice != item.PriceAtAdd {
              // Policy decision: what do you do?
              //
              // Option A: Use new price silently (user might be surprised)
              // Option B: Block checkout, show updated price (friction)
              // Option C: Honor old price if within threshold (e.g., < 10% change)
              //
              // Most e-commerce sites use Option C:
              priceDrift := math.Abs(currentPrice-item.PriceAtAdd) / item.PriceAtAdd
              if priceDrift > 0.10 {
                  return nil, fmt.Errorf(
                      "price for %s changed from %.2f to %.2f (%.0f%% change)",
                      item.ProductID, item.PriceAtAdd, currentPrice, priceDrift*100,
                  )
              }
              items[i].PriceAtAdd = currentPrice // accept small drift
          }

          // Check stock
          stock, err := s.inventory.GetAvailable(ctx, item.ProductID)
          if err != nil || stock < item.Quantity {
              return nil, fmt.Errorf("insufficient stock for %s (need %d, have %d)",
                  item.ProductID, item.Quantity, stock)
          }
      }

      return json.Marshal(items)
}

Abandoned Cart Cleanup (The 90% Problem)

// Redis handles this automatically with TTL.
// Browsing carts: expire after 2 hours (the user left).
// Engaged carts: expire after 7 days.
// Checkout carts in DB need explicit cleanup:

-- Periodic job (run every 10 minutes)
-- Only deletes carts that started checkout but never completed
DELETE FROM checkout_carts
WHERE expires_at < NOW()
AND id NOT IN (SELECT cart_id FROM orders WHERE status != 'cancelled');

-- For marketing: extract abandoned carts before deleting
INSERT INTO abandoned_cart_events (cart_id, user_id, items, abandoned_at)
SELECT id, user_id, items, NOW()
FROM checkout_carts
WHERE expires_at < NOW()
AND user_id IS NOT NULL;      -- only logged-in users (can email them)

Redis Failure During Checkout (The Critical Path)

// The scariest scenario: Redis dies while a user is in checkout.
// Their cart is in Redis, not yet promoted to DB.

func (s *TieredCartStore) GetCartForCheckout(ctx context.Context, cartID string) ([]StoredCartItem, error) {
    // Try Redis first (fast path)
    items, err := s.redisGetCart(ctx, cartID)
    if err == nil && len(items) > 0 {
        return items, nil
    }

      // Redis failed or empty — check DB (cart may have been promoted earlier)
      items, err = s.dbGetCart(ctx, cartID)
      if err == nil && len(items) > 0 {
          return items, nil
      }

      // Both empty — cart is truly lost.
      // This is the failure case. Mitigation strategies:
      //
      // 1. Redis Cluster with WAIT command (sync replication at checkout)
      //    → Guarantees at least one replica has the data before ACKing
      //
      // 2. Proactive promotion: promote to DB as soon as cart reaches 3+ items
      //    → Narrows the window where cart exists only in Redis
      //
      // 3. Client-side backup: JS stores cart items in localStorage as a fallback
      //    → On "cart not found", POST client cart to server for re-validation

      return nil, ErrCartNotFound
}

// Using WAIT for synchronous replication during checkout:
func (s *TieredCartStore) redisAddItemDurable(ctx context.Context, cartID string, item CartItem) error {
data, _ := json.Marshal(item)
pipe := s.redis.Pipeline()
pipe.HSet(ctx, cartKey(cartID), item.ProductID, data)
pipe.Expire(ctx, cartKey(cartID), 24*time.Hour)
// WAIT: block until at least 1 replica has acknowledged the write
// Timeout 500ms — if replica is slow, fall through to DB write
pipe.Do(ctx, "WAIT", 1, 500)
_, err := pipe.Exec(ctx)
return err
}

Summary: Which Approach for What Scale

┌────────────────┬──────────────┬───────────────┬────────────────────────┐
│ Scale          │ Cart writes  │ Approach      │ Tradeoff               │
│                │ /sec         │               │                        │
├────────────────┼──────────────┼───────────────┼────────────────────────┤
│ Small          │ < 1K         │ PostgreSQL    │ Simple, ACID, but will │
│ (startup)      │              │ only          │ hit ceiling eventually │
├────────────────┼──────────────┼───────────────┼────────────────────────┤
│ Medium         │ 1K - 10K     │ Redis primary │ Fast, but accept ~1s   │
│ (growth stage) │              │ + DB backup   │ data loss window       │
├────────────────┼──────────────┼───────────────┼────────────────────────┤
│ Large          │ 10K - 100K   │ Tiered:       │ Complexity is high,    │
│ (at scale)     │              │ cookie →      │ but each tier is       │
│                │              │ Redis → DB    │ optimized for its role │
├────────────────┼──────────────┼───────────────┼────────────────────────┤
│ Massive        │ 100K+        │ Tiered +      │ DynamoDB handles       │
│ (Amazon-scale) │              │ DynamoDB /    │ write throughput,      │
│                │              │ Cassandra     │ no vacuum, no bloat    │
│                │              │ replaces DB   │                        │
└────────────────┴──────────────┴───────────────┴────────────────────────┘

The key insight: match storage durability to conversion probability. A browsing cart that's 5 minutes old has a ~2% conversion chance — storing it in PostgreSQL is
wasting 98% of your write budget. A cart in checkout has a ~60% conversion chance — storing it only in Redis is gambling with revenue.
