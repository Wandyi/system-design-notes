# Variant Explosion — Deep Dive

> A T-shirt with 5 sizes × 8 colors × 3 fits = 120 variants. A configurable laptop with 4 CPUs × 3 RAM × 3 storage × 2 displays = 72 variants. Each variant has its own inventory, price, and images. Catalog queries that join product → variant → inventory become expensive. At 1M products × 100 variants, that's 100M rows.

Most engineers first meet this problem when they model a T-shirt as a row. It works until someone asks "show me all red medium T-shirts in stock" and the query plan lights up. The real lesson is not "denormalize more" or "use Mongo" — it's that **a product is not a row; it is a Cartesian product of option values, constrained by a subset of valid combinations, each of which has independent inventory and price lifecycles.** Every modeling decision has to respect that structure.

---

## 1. Why The Obvious Models Fail

### Model A: One row per product, columns for variants

```sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    title TEXT,
    sizes TEXT[],       -- {'S','M','L','XL','XXL'}
    colors TEXT[],      -- {'red','blue',...}
    fits TEXT[]
);
```
Fine for browsing. Useless the moment you ask:
- "Is the red M in stock?" — there's no row for `(red, M)` to attach inventory to.
- "How much does the blue L cost?" — no place to store per-variant price.
- "How many blue XLs did we sell this quarter?" — no SKU to aggregate on.

This is what happens when a product engineer models a product and an operations engineer wasn't in the room.

### Model B: One row per variant (flat SKU table)

```sql
CREATE TABLE skus (
    id BIGINT PRIMARY KEY,
    product_id BIGINT,
    size TEXT, color TEXT, fit TEXT,
    price NUMERIC, stock INT
);
```
Works for a T-shirt (3 attributes). Breaks the moment a laptop arrives with (CPU, RAM, storage, display, keyboard_layout, warranty). 
Do you `ALTER TABLE ADD COLUMN` for every new option type across the whole catalog? Now a cable has a `cpu` column. A perfume has a `ram` column. 
This is the "wide table with many nulls" anti-pattern, and it does not scale past a single category.

### Model C: EAV (entity-attribute-value)

```sql
CREATE TABLE variant_attrs (
    variant_id BIGINT,
    attr_key TEXT,
    attr_value TEXT,
    PRIMARY KEY (variant_id, attr_key)
);
```
Infinitely flexible — and infinitely slow. Every facet query becomes N self-joins (one per attribute). This is why classic Magento was notorious: 
its EAV model meant that loading a product page triggered dozens of joins, and its solution was to flatten EAV into per-store projection tables via a nightly cron. 
If you ever have to write a cron that flattens your OLTP, your OLTP is wrong.

### The mental model that wins

A product in a real catalog is:
```
Product              = { option_type_1, option_type_2, ... option_type_N }
option_type_i        = { option_value_i_1, option_value_i_2, ... }
Variant              = ordered tuple (v_1_j, v_2_k, ... v_N_m)
Valid variants       = (some subset of) the Cartesian product
```
The schema must reflect this. Flatten anywhere, and you lose either flexibility (Model B) or performance (Model C).

---

## 2. The Canonical Schema

The schema that survives at scale (Shopify, BigCommerce, Magento 2 post-refactor, commercetools, and every serious platform I have seen in the wild):

```sql
-- The product. One row per "thing being sold as a concept."
CREATE TABLE products (
    id              BIGINT PRIMARY KEY,
    title           TEXT NOT NULL,
    description     TEXT,
    brand_id        BIGINT,
    category_id     BIGINT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- The axes of variation this product has (color, size, CPU, storage ...).
-- Ordered: display-left to display-right on the PDP (product detail page).
CREATE TABLE product_option_types (
    product_id      BIGINT NOT NULL REFERENCES products(id),
    position        SMALLINT NOT NULL,   -- 1..N
    name            TEXT NOT NULL,       -- "Color", "Size"
    PRIMARY KEY (product_id, position)
);

-- The allowed values for each axis.
CREATE TABLE product_option_values (
    id              BIGINT PRIMARY KEY,
    product_id      BIGINT NOT NULL,
    option_position SMALLINT NOT NULL,
    value           TEXT NOT NULL,       -- "Red", "M"
    metadata        JSONB,               -- {"hex":"#e11","swatch_url":"..."}
    FOREIGN KEY (product_id, option_position) REFERENCES product_option_types(product_id, position)
);

-- The actual SKUs — one row per VALID combination.
-- Not every Cartesian combination exists (laptop may not ship 32GB+i3).
CREATE TABLE variants (
    id              BIGINT PRIMARY KEY,
    product_id      BIGINT NOT NULL REFERENCES products(id),
    sku             TEXT UNIQUE NOT NULL,   -- external human-readable, e.g. "TEE-RED-M"
    barcode         TEXT,
    price           NUMERIC(20,4) NOT NULL,
    compare_at      NUMERIC(20,4),          -- "was"-price for struck-through display
    weight_grams    INT,
    status          TEXT NOT NULL,          -- active|discontinued|coming_soon
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Which option values make up this variant (M-to-M between variant and option_value).
-- Exactly one row per (variant, option_position) — enforced by the PK.
CREATE TABLE variant_option_values (
    variant_id      BIGINT NOT NULL REFERENCES variants(id),
    option_position SMALLINT NOT NULL,
    option_value_id BIGINT NOT NULL REFERENCES product_option_values(id),
    PRIMARY KEY (variant_id, option_position)
);

-- Inventory is per variant, per warehouse/location.
CREATE TABLE inventory (
    variant_id      BIGINT NOT NULL,
    location_id     BIGINT NOT NULL,
    on_hand         INT NOT NULL,            -- physical
    committed       INT NOT NULL DEFAULT 0,  -- reserved by open orders
    incoming        INT NOT NULL DEFAULT 0,  -- inbound POs
    PRIMARY KEY (variant_id, location_id)
);

-- Images, typically per-variant with a fallback to product-level.
CREATE TABLE variant_images (
    variant_id      BIGINT,
    image_id        BIGINT,
    position        SMALLINT,
    PRIMARY KEY (variant_id, image_id)
);
```

Why this is durable:
- **`products`** holds only what is shared (title, description, brand). These change on a product-lifecycle cadence (weeks, months).
- **`variants`** holds per-SKU data that has its own lifecycle (price changes hourly, stock changes per second).
- **`variant_option_values`** encodes the tuple that *identifies* the variant within the product. You never join three copies of `variant` to itself.
- Adding a new option type (e.g., "engraving") is a data change, not a schema change.

### The "valid subset" constraint

A laptop may not ship with every Cartesian combination. 32GB-only for i7+ configs. 4K display only on i9. Two ways to encode:

- **Enumerate valid variants.** Only `INSERT` into `variants` the combinations you actually sell. The Cartesian space is implicit; only the subset exists as rows. This is what most platforms do.
- **Model a constraint graph.** `variant_rules` table with predicates like "IF cpu='i3' THEN ram IN {'8','16'}". Useful for configure-to-order products (laptops with thousands of combinations that don't all warrant a pre-created SKU). Combined with on-the-fly SKU generation at order time.

Decision rule: if the number of valid combinations is < ~500 and each has a distinct stock position, pre-create SKUs. If it's in the thousands and most are "built to order," use a constraint engine + synthetic SKUs at order time.

---

## 3. The Cardinality Math

The user's framing: 1M products × 100 variants = 100M rows. Let's stress-test each join they imply.

### Storage

```
variants:                 100M × ~200 bytes  = ~20 GB     (with indexes: ~60 GB)
variant_option_values:    100M × 3 options × 32 bytes = ~10 GB
inventory:                100M × 5 locations × 32 bytes = ~16 GB (per-location FC setup)
variant_images:           100M × 5 images × 16 bytes = ~8 GB
```

This is *fine* for a modern OLTP DB. The trouble isn't storage. It's query cost on the hot paths.

### The hot queries

```sql
-- Q1: Product detail page. Show all variants of product 42.
SELECT v.id, v.sku, v.price,
       json_agg(json_build_object(
           'position', vov.option_position,
           'value', pov.value
       ) ORDER BY vov.option_position) AS options,
       coalesce(sum(i.on_hand - i.committed), 0) AS available
FROM variants v
JOIN variant_option_values vov ON vov.variant_id = v.id
JOIN product_option_values pov ON pov.id = vov.option_value_id
LEFT JOIN inventory i ON i.variant_id = v.id
WHERE v.product_id = 42 AND v.status = 'active'
GROUP BY v.id;
```

Index plan: `CREATE INDEX variants_product_id ON variants(product_id) WHERE status='active';`. Reads ~100 variant rows + ~300 option rows + ~500 inventory rows. Runs in single-digit ms. OK.

```sql
-- Q2: Category listing. 40 products per page, each with min/max price and "in stock" flag.
SELECT p.id, p.title,
       MIN(v.price) AS min_price, MAX(v.price) AS max_price,
       bool_or((i.on_hand - i.committed) > 0) AS in_stock
FROM products p
JOIN variants v ON v.product_id = p.id AND v.status='active'
LEFT JOIN inventory i ON i.variant_id = v.id
WHERE p.category_id = 123
GROUP BY p.id
LIMIT 40 OFFSET 0;
```

Hot path for every browse. At 40 products × 100 variants × 5 locations = 20K rows aggregated per page render. Hundreds of category pages per second. This is where you *must* precompute.

### The precompute that saves the category listing

Projection table, refreshed on variant/inventory change:

```sql
CREATE TABLE product_rollup (
    product_id      BIGINT PRIMARY KEY,
    category_id     BIGINT,
    min_price       NUMERIC(20,4),
    max_price       NUMERIC(20,4),
    total_available INT,
    any_in_stock    BOOLEAN,
    variant_count   INT,
    updated_at      TIMESTAMPTZ
);

CREATE INDEX product_rollup_category ON product_rollup(category_id, min_price);
CREATE INDEX product_rollup_stock ON product_rollup(category_id) WHERE any_in_stock = true;
```

Now the category listing is a single indexed read from `product_rollup`, no joins.

Maintenance options:
- **Materialized view + periodic refresh.** Simple, acceptable lag (minutes).
- **Trigger on `variants`/`inventory`.** Low lag but ties write throughput to rollup speed.
- **Change-data-capture (Debezium → stream processor → rollup table).** Highest throughput, most operational complexity.

Which one: for stores with < 100K products and infrequent writes, MV + pg_cron every 5 min is fine. For hot-selling platforms with 1M products and live stock swings, CDC-driven rollup is the only option.

### The trap: stock-sensitive listing order

If your category page sorts "in stock first, then by popularity," the rollup's `any_in_stock` flag drives sort order. A single stock change on one variant can cause a product to jump or fall in rank. If the rollup is lagged by 5 minutes, a user browsing during a big sale sees out-of-stock products near the top. That UX break is worse than the perf win.

Mitigation: compute `any_in_stock` at read-time from a fast-path inventory cache (Redis hash `stock:product_id` → int), not from the MV. Keeps the MV cheap and the sort fresh.

---

## 4. Inventory Per Variant — The Real Cost Driver

Inventory is where variant explosion becomes a production problem. Consider:

- 1M products × 100 variants × 5 warehouses = 500M inventory rows.
- Peak write load during warehouse pick-pack: 10K+ inventory updates per second across a marketplace.
- Read load: every add-to-cart, every stock check on a listing.

### Schema choices

**Option 1: Dense row per variant per location.**
```sql
inventory(variant_id, location_id, on_hand, committed, incoming, updated_at)
```
500M rows, ~30GB. Works, but hot variants become hot rows — the 80/20 problem. A flash-sale SKU at 1K orders/sec all update the same `(variant, warehouse)` row and serialize on the row lock.

**Option 2: Sharded inventory (per-variant split into buckets).**
```sql
inventory_shard(variant_id, shard_id, location_id, on_hand)
--           shard_id = hash(order_id) % N, e.g., 10 shards
```
Each shard holds 1/N of the stock. Writers update any random shard. Readers sum across shards. Kills the hot-row contention at the cost of read complexity.

This is how high-throughput inventory systems (AWS Inventory, flash sales at Shopee/Lazada) avoid row-lock collapse. Worth doing only for your top-N hottest SKUs — most SKUs are cold and don't need it.

**Option 3: Event-sourced inventory.**
```sql
inventory_events(event_id, variant_id, location_id, delta, reason, at_ts)
```
Current stock = SUM(delta). Never updates a row; always inserts. Kills lock contention entirely. Cost: reads must aggregate or project. Pair with a rollup table (Option 1) refreshed from the event stream, giving you the best of both.

Most large-scale commerce systems end up with a hybrid: event log as source of truth, projection table for fast reads, optionally sharded for the top-K hottest SKUs.

### The read-path trap

For the add-to-cart flow, you cannot read stock from a replica. Replica lag (even 500ms) means you accept orders against stock that another transaction already committed to someone else. Always read stock from the primary, or from a Redis projection that is primary-driven.

```go
// WRONG — replicas lag
ok, _ := replicaDB.QueryRow("SELECT (on_hand - committed) > 0 FROM inventory WHERE variant_id=$1", sku).Scan(&ok)
if ok { reserveStock(sku) }

// RIGHT — primary or Redis counter updated by primary
stock, _ := redis.DecrBy(ctx, fmt.Sprintf("stock:%d", sku), qty).Result()
if stock < 0 {
    redis.IncrBy(ctx, fmt.Sprintf("stock:%d", sku), qty)  // rollback
    return ErrOutOfStock
}
// persist reservation in DB after
```

Redis DECR is atomic and single-digit-microsecond. That's your reservation lock. The DB is the audit log.

---

## 5. Pricing Per Variant — What to Inherit and What Not To

Variants differ in price in predictable ways:
- Size M costs less than XXL (fabric).
- Larger storage on a laptop costs more.
- Certain colors cost more due to dyeing process.

Schema options:

**Option A: Full price on every variant.** Each variant row has its own `price`. Simple. Works. All 100 variants store a price even if 80 of them are the same.

**Option B: Product base price + variant delta.** `variants.price_delta` is added to `products.base_price`. Saves storage; bad for operations. Every price report, promotion engine, and tax system now has to do the math in its own query. One wrong sign somewhere = bug. Don't.

**Option C: Full price on variant + "pricing rule."** Used when a merchandiser wants "every size-XXL across all shirts costs +$2." Store the computed price on the variant; the rule is *how the system writes it*, not *how it's read*. Reads are always full-precision material price.

Decision: **Option A**, always, for reads. The storage cost of 100M prices is trivial. The operational cost of computed prices is large. Whatever your admin UI lets merchandisers do (percentage increases, absolute bumps), it should translate into concrete per-variant price writes via a batch update. Your read path should never care where the price "came from."

See `stalePricingCache.md` for how price changes propagate through caches.

---

## 6. Images: Product-Level, Variant-Level, and Axis-Level

Naive: each variant has an image set. For a T-shirt with 120 variants × 5 images each = 600 images per product. Multiply by 1M products = 600M image records, most of which are duplicates (size M and size L of the same color look identical in photos).

Reality: images vary by a **subset** of the option axes. For apparel: images are per-color, shared across sizes and fits. For laptops: images are per-chassis (same body regardless of CPU/RAM/storage).

Schema with axis-scoped images:

```sql
CREATE TABLE product_images (
    id              BIGSERIAL PRIMARY KEY,
    product_id      BIGINT NOT NULL,
    url             TEXT NOT NULL,
    position        SMALLINT,
    -- Axis scoping: image is valid for variants matching ALL listed constraints.
    scope           JSONB NOT NULL DEFAULT '[]'
                    -- e.g. [{"option_position":2, "option_value_id":707}]  → all Red variants
                    -- []  → product-level default (shown if no variant-specific match)
);
```

At read-time, for a given variant, pick:
1. Images whose scope is a subset of the variant's option values.
2. Fall back to `scope = []` if none match.

This drops image cardinality from "per variant" to "per color" or "per chassis" — one or two orders of magnitude reduction.

---

## 7. UX Implications of 120 Variants

You cannot render 120 radio buttons. Standard pattern:

- Render one selector per option type (size, color, fit).
- When a user picks "Size = M", all variants that are not `size=M` are filtered out.
- For remaining axes, each candidate value has a state:
  - **Available.** A variant with this combo exists and has stock.
  - **Out of stock.** Variant exists; no stock. Style: struck-through but clickable (leads to "notify me").
  - **Unavailable.** No such variant (e.g., M+Slim-Fit-Red doesn't ship). Style: disabled.

To render this efficiently, the server must tell the client *the full option × stock matrix* for the product. A compact form:

```json
{
  "option_types": [
    { "position": 1, "name": "Color", "values": [{"id":701,"value":"Red"},{"id":702,"value":"Blue"}]},
    { "position": 2, "name": "Size",  "values": [{"id":801,"value":"S"}, {"id":802,"value":"M"}]}
  ],
  "variants": [
    {"sku":"TEE-RED-S","option_value_ids":[701,801],"price":20,"available":false},
    {"sku":"TEE-RED-M","option_value_ids":[701,802],"price":20,"available":true},
    {"sku":"TEE-BLU-S","option_value_ids":[702,801],"price":20,"available":true},
    {"sku":"TEE-BLU-M","option_value_ids":[702,802],"price":20,"available":false}
  ]
}
```

Client code does the matrix intersection. The server contract is flat and cache-friendly.

---

## 8. Search: Denormalize Variants into the Index

Search is the second-hottest read path. Joining `products → variants → option_values → inventory` on every query doesn't scale. Denormalize into the search index (Elasticsearch / OpenSearch / Algolia).

Two indexing strategies:

**Strategy A: Document per product, variant data embedded.**
```json
{
  "id": 42,
  "title": "Cotton T-shirt",
  "colors": ["red","blue"],
  "sizes": ["S","M","L","XL"],
  "min_price": 18, "max_price": 24,
  "any_in_stock": true,
  "variants": [
    {"sku":"TEE-RED-S","price":18,"in_stock":false},
    ...
  ]
}
```
Pro: one doc per product, natural for product listings.
Con: updating a single variant's stock rewrites the whole product doc (Elasticsearch treats docs as atomic). At 1M products × 10 updates/sec you're churning a lot.

**Strategy B: Document per variant, with `product_id` grouping.**
Search returns variants; collapse by `product_id` using `field collapsing` (Elasticsearch) or `distinct` (Algolia).
Pro: each stock change rewrites one doc.
Con: more documents (100× more), and collapsing has its own performance quirks.

**Production choice:** B for marketplaces where variant-level facets matter (search for "red medium dress" must match the *specific variant*). A for simpler catalogs.

Either way: ES index is *denormalized*. Updates flow from CDC. You accept seconds of lag — search is read-freshness-tolerant by convention, unlike add-to-cart.

---

## 9. Edge Cases That Surface At Scale

### 9.1 Partial launch / staggered activation

A new variant is created but should not appear until launch date. Use `variants.status='coming_soon'` and a `status_effective_at` timestamp. Filter in reads. Do not rely on the variant being absent — operations still need the row for ERP syncing before launch.

### 9.2 Delisting a single variant

Red M is recalled for a fabric defect. You must:
- `status='discontinued'` on the variant (not delete — preserves order history linkage).
- Remove from category listings (rollup recompute — `any_in_stock` may now be false).
- Remove from search (CDC propagates).
- Handle in-flight carts and wishlists with that variant — notify users gracefully.
- Refund any outstanding pre-orders.

The atomicity here is a saga; see `databaseTransactions.md §7` for the pattern.

### 9.3 Reusing a variant definition across products

Brand ships "the black version" of five different T-shirt styles. Tempting to share a variant row. Don't. One variant = one SKU = one set of inventory. Even if the `(color=black)` option-value reference is shared, the variant is a separate row because inventory is separate. Deduplicate option values (reuse `product_option_values.id`), not variants.

### 9.4 Bundles: a product whose "variants" are other products

A gift set contains 3 T-shirts, customer picks sizes for each. This is not the same as variants; it's a composite product. Model separately as `bundle_components`. Inventory is derived: bundle is in stock iff *all* its components are in stock. Reservation of a bundle reserves each component. Do not try to force this into the variant schema — many platforms have tried, and every one of them ends up with a `bundles` table anyway.

### 9.5 Configure-to-order with infinite combinations

A laptop with 8 dimensions, not all priced a priori. Two approaches:
- Pre-create only the "canonical" combinations (top 20 SKUs by sales forecast). Others are synthesized at order time.
- Price engine returns a price for any combination from component prices + build cost. Generate the variant row on demand, persist it when the order is placed.

This is a different problem class. The schema above handles it if you allow `variants` to be created at order time. What you lose: pre-warmed caches, ES index coverage for the long tail. What you gain: ability to sell a combinatorial catalog without 10M dormant rows.

### 9.6 Variant-level promotions

"20% off all Blue variants." The promo engine must be able to target at the option-value level, not just product-level or variant-id level. Store promos as predicates over `option_value_id`, evaluate at pricing time. Avoid copying the promo onto every matched variant row — that denormalization invalidates every time a variant is added.

### 9.7 Option ordering matters for display but not for identity

`(Red, M)` and `(M, Red)` are the same variant. The variant's identity is its multiset of option_value_ids. But the display is order-sensitive — shoppers expect Color first, Size second. That's what `option_types.position` is for. Never let position change affect variant identity (never key a variant by a tuple whose order can change).

### 9.8 Backfills and schema evolution

Adding a new option type to an existing product (e.g., adding a "length" axis to pants that only had size + color) means all existing variants become ambiguous — which length do they represent? Practical answer: migrate by assigning a "default" option value to every existing variant (`length = 'regular'`), then allow new variants with other lengths. Don't try to "infer" from title strings; be explicit in the migration.

---

## 10. Putting It Together: Query Plan for a Real PDP Render

For a single product page render on a 100-variant product, the backend issues in parallel:

1. **Product metadata** (title, description, category, brand) — single PK lookup on `products`; cached aggressively (hours).
2. **Variant matrix** (id, option values, price, basic dimensions) — indexed range scan on `variants` by `product_id`; cacheable with variant-set version key; invalidated when any variant changes.
3. **Stock map** (variant_id → available) — Redis hash lookup; sub-millisecond; always live.
4. **Images with scope** — indexed scan on `product_images`; cached.
5. **Reviews, ratings, related products** — independent services, parallel fetches.

Total round-trip: ~5-15ms at the edge. The DB sees one indexed query; everything else is either cache or a different store. This is what makes 100M-variant catalogs feel instant.

---

## 11. Observability: What Goes Wrong With Variants

- **Orphan variants:** `variants` rows with no matching `variant_option_values` (migration bug). Alert if any exist. A variant without options is un-renderable.
- **Option value drift:** a `product_option_values.id` referenced by `variant_option_values` got deleted (broken FK in a schema with deferred constraints). Variant becomes ghost.
- **Duplicate variants:** two variant rows with the same `(product_id, option_value_tuple)`. Caused by race conditions in variant creation. Enforce a unique index on the *sorted* option_value_ids:
  ```sql
  CREATE UNIQUE INDEX variants_uniq_combo ON variants(
      product_id,
      (SELECT array_agg(option_value_id ORDER BY option_position)
         FROM variant_option_values WHERE variant_id = variants.id)
  );
  ```
  Cannot do this directly (correlated subquery in index), so enforce via trigger + daily reconcile job.
- **Rollup staleness:** `product_rollup.any_in_stock` disagrees with live inventory. Alert when divergence > 1% across a sample.
- **Skew in variant cardinality:** most products have 5 variants; a handful have 10,000. Query planner chooses indexes based on average selectivity and can misplan for the outliers. Either cap max variants per product, or route heavy-variant products to a specialized query path.

---

## 12. When to Not Use This Schema

Some products really don't vary. Digital goods (ebooks, course licenses), service bookings, insurance policies. Forcing them into product + variant is overhead for no gain.

Rule of thumb: if > 90% of your catalog is single-variant, model products as directly sellable (`products.price`, `products.stock`) and reserve the variants subsystem for the minority that needs it. Cleaner APIs, simpler reads.

If your catalog is 50/50, use the full schema uniformly — single-variant products are just products with one variant. The consistency is worth the minor per-row overhead.

---

## 13. Closing: The Staff-Level Lens

When this comes up in a design review, the right answer isn't a schema. It's the framing:

1. **A product is a Cartesian product of option values, constrained to a valid subset, where each element has independent operational lifecycles.** That shape dictates the schema; anything else is an approximation that will leak.
2. **Inventory, price, and images have different cardinalities and different change rates.** Model each to its own axis — don't collapse them into a "variant" blob.
3. **Projection tables are not optional at scale.** The normalized schema is for correctness; the rollup tables are for read latency. Keep both; sync via CDC.
4. **Hot-row contention is the real cost driver, not row count.** 100M rows is fine; 100M rows where 20 are touched 10K times per second is not. Shard or event-source the hot ones.
5. **Search and browse are denormalized by design.** Stop trying to make them run against the OLTP schema. They're read stores with their own update cadence.

The "100M rows" in the corner-case framing is a red herring. 100M rows in a well-modeled schema with proper indexes and rollups is trivial. 10M rows in the wrong schema will melt the database on a Tuesday at 2pm. The staff-level engineer's job is to tell them apart before either one ships.