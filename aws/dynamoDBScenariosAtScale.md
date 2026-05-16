# AWS DynamoDB — Realistic Scenarios at Staff-Engineer Depth

> A practical, opinionated reference covering the features of DynamoDB and how to combine them to solve real problems at scale. Each scenario presents the problem, the naive approach, the DynamoDB-native architecture, the trade-offs, alternatives, and what I would actually do. Written for engineers past "DynamoDB is a key-value store" who now have to choose primary keys for petabyte tables, fix hot-partition outages at 3 AM, and explain why the Aurora team's RDS-shaped instinct is wrong here.

> Companion to `awsS3ScenariosAtScale.md`, `snsSqsEventBridgeAtScale.md`, `databaseTransactionScenarios.md`, `statefulSystemsAtMAANGScale.md`, `postgresReadOptimizationMAANGScale.md`. Together they map the AWS-native data and event surface.

---

## 0. The Staff-Level Frame

DynamoDB rewards *correct schema design upfront* and punishes ad-hoc query evolution. Unlike Postgres, you can't add an index after launch and have all your queries work. At staff level the questions are:

1. **What are the access patterns, all of them, including the ones marketing will demand in 6 months?** (Modeling DynamoDB starts from queries, not data.)
2. **What is the partition-key's hotness profile?** (One hot key = one shard = capped throughput.)
3. **What's the consistency budget?** (Strong reads cost 2× capacity; GSIs are eventually consistent always.)
4. **What's the cost shape at scale?** ($/RCU, $/WCU, $/GB, $/Stream-shard. Hot partitions inflate WCU bills 10×.)
5. **What does failure look like?** (Provisioned hits ceiling → throttle → cascade. On-demand has cold-start ramp limits. Global Tables conflict resolution is LWW.)
6. **What's the operational lifecycle?** (Schema evolution. Adding a GSI on a 10TB table takes hours and costs $$$. Backups, restores, migrations.)

The mistake everyone makes: treating DynamoDB like a relational DB with funny syntax. The right model is closer to
**a managed, partitioned, log-structured KV store that happens to support some range queries**. Embrace that and the design choices follow.

---

## 1. Mental Model — What DynamoDB Is and Isn't

### 1.1 What it is

- A **partitioned, replicated, serverless KV store** with optional sort keys for ranges within a partition.
- **Single-digit ms p99** at any scale (with proper design).
- **Auto-sharded** across thousands of partitions; each ~10 GB / 3K read units / 1K write units.
- **Multi-AZ replicated** within a region; 11×9 durability; 99.99% / 99.999% (multi-region) availability.
- **Conditional writes, transactions** (up to 100 items per transaction), **TTL**, **Streams (CDC)**, **GSI/LSI**.
- **Two pricing modes**: provisioned (set capacity) or on-demand (pay-per-request).

### 1.2 What it isn't

- **Not a relational DB.** No joins. No ad-hoc queries. No SQL aggregations.
- **Not Cassandra.** Cassandra is closest in shape; but DynamoDB has stronger consistency, transactions, no compaction tuning.
- **Not for analytical workloads.** Scans are ruinous at scale. Use a warehouse or stream to S3.
- **Not eventually consistent by default for primary table reads** (since 2014 strongly-consistent reads exist at 2× cost). GSIs *are* eventually consistent.
- **Not free-form schema.** You design around access patterns. Adding a new query pattern after launch may require GSI build (hours, $$$) or full table redesign.

### 1.3 The pricing model — internalize this

```
PROVISIONED:
  Per RCU/hr  = $0.00013 (us-east-1)
    1 RCU = 4 KB / sec strongly-consistent read OR 8 KB / sec eventually-consistent
  Per WCU/hr = $0.00065
    1 WCU = 1 KB / sec write
  Storage = $0.25/GB/month

ON-DEMAND:
  Read = $0.125 per million RRU
  Write = $0.625 per million WRU
  (No capacity to manage; pay per request)
  Storage same.

Streams:        $0.02 per 100K read requests on shard.
DAX:            EC2-like instance pricing.
Global Tables:  per-region replicated WCUs.
Backups:       $0.10/GB-month for continuous; $0.20/GB on-demand.
```

A staff-level mistake: not modeling cost before launch. A single hot-key table at on-demand can swing 10× in monthly bill.

### 1.4 Hard limits that drive design

| Limit | Value | Implication |
|---|---|---|
| **Partition size** | 10 GB | One PK group can't exceed (rare; usually item-collection limit binds first) |
| **Item size** | 400 KB | Don't store blobs; offload to S3 |
| **PK length** | 2048 bytes | Plenty |
| **SK length** | 1024 bytes | Plenty |
| **Per-partition throughput** | 3K RCU / 1K WCU | The "hot partition" ceiling |
| **GSIs per table** | 20 | Compose access patterns; don't over-index |
| **LSIs per table** | 5 | Defined at table creation only |
| **Transaction items** | 100 (per transaction) | 4 MB total |
| **Query/scan response** | 1 MB | Pagination required |
| **Streams retention** | 24 hours | Beyond that → archive elsewhere |
| **TTL precision** | within 48 hours of expiry | Best-effort, not exact |

---

## 2. Scenario 1 — User Profile / Session Store (Key-Value at Scale)

### 2.1 The problem

Authoritative user profile store. Hundreds of millions of users. Read 100K+ RPS at peak (every login, every API call). Single-digit ms p99 latency. Multi-region.

### 2.2 The design

```
Table: users
  PK: user_id (string, e.g., UUID v4)
  Attributes: email, name, plan, created_at, ...

Reads:
  GetItem(PK=user_id)             → ~5 ms p99, 0.5 RCU per call (1 KB item)
Writes:
  PutItem / UpdateItem            → ~10 ms p99, 1 WCU per call

Capacity (provisioned):
  100K reads/sec × 0.5 RCU = 50K RCU
  10K writes/sec × 1 WCU   = 10K WCU
```

This is the canonical KV use case. Single-partition by user_id means **uniform access** across thousands of underlying partitions. No hot keys (unless one specific user is celebrity-hot — see §11).

### 2.3 Why DynamoDB beats RDS here

```
RDS Postgres at 100K RPS for user lookups:
  - Need r6g.16xlarge or similar; 5K-10K connections via PgBouncer.
  - Replicas for read scaling; replication lag.
  - Manage failover, cluster ops.
  - $5K-15K/mo all-in.

DynamoDB at 100K RPS:
  - Provisioned: 50K RCU × $0.00013/hr × 720 hr = $4,680/mo.
  - On-demand: 100K × 86400 × 30 / 1M × $0.125 = $32K/mo (much higher; convert to provisioned).
  - Zero cluster management.
```

For pure key-by-id lookups at scale: DynamoDB is operationally and (often) financially cheaper.

### 2.4 Strongly vs eventually consistent reads

```
GetItem default: eventually consistent (cheaper, 1 RCU per 8 KB).
GetItem(ConsistentRead=True): strongly consistent (2× cost; reads from leader).

For most user lookups: eventual is fine (post-write, ~10ms staleness).
For payment/identity-critical reads: ConsistentRead=True.
```

A common over-spend: defaulting to ConsistentRead=True everywhere. Half the read budget down the drain for no UX benefit.

### 2.5 Adding lookup-by-email

The new requirement: also lookup user by email. Email is not the PK.

**Option 1: GSI on email**
```
GSI:  email-index
  PK: email
  Attributes (projected): user_id, name, plan
```

Cost: GSI doubles write capacity (every write replicates to GSI). Storage doubles. Read GSI for email lookups; eventually consistent only.

**Option 2: Second item with reversed key**
```
Same table, two items per user:
  Item 1: PK="USER#"+user_id, type="user", attrs...
  Item 2: PK="EMAIL#"+email, type="email-lookup", user_id=...
```

Two writes per user creation (use TransactWriteItems for atomicity). Cheaper than GSI for low-write workloads; more flexible.

**Option 3: External index**
```
Hot by email? Cache email → user_id in Redis.
Search by partial email? Use OpenSearch.
```

For just email lookup at scale: **GSI**. For complex search (substring, fuzzy): OpenSearch fed by Streams.

### 2.6 Multi-region: Global Tables

```
Global Tables v2:
  Active-active replication across regions.
  Eventually consistent (~1 sec lag typical).
  LWW conflict resolution (last writer by timestamp wins).
  Per-region WCU + replicated WCU billing.
```

For a global user base reading from their nearest region: enable Global Tables. Watch for conflict-prone fields (counters); shard or use atomic increments to avoid LWW silent loss.

### 2.7 Trade-offs and alternatives

| Approach | Trade |
|---|---|
| **DynamoDB single table** | Predictable ms latency; KV-only |
| **RDS Postgres** | Rich queries; ops cost |
| **Aurora** | Better than RDS; still tier-bound |
| **Cassandra / ScyllaDB** | DynamoDB-shape, self-managed |
| **Cloud Spanner / CockroachDB** | SQL + global; expensive |
| **Redis** | µs latency; not durable as primary store |

### 2.8 What I'd actually do

For user profiles at MAANG scale: DynamoDB primary, single-table by user_id, GSIs for the 1-3 most common alt-key queries (email, phone), 
Streams to Lambda for cache invalidation and OpenSearch indexing for search, Global Tables for multi-region. PITR enabled. DAX for the read-heavy hot tier.

---

## 3. Scenario 2 — Shopping Cart / Session Store

### 3.1 The problem

E-commerce shopping cart. Users add/remove items; abandoned carts must persist for days. Carts are accessed every page load. 50K reads/sec, 5K writes/sec at peak. Latency budget: 10 ms.

### 3.2 The design

```
Table: carts
  PK: user_id (or session_id for anon)
  Attributes:
    items: [
      { sku, qty, price_at_add, added_at }
    ],
    updated_at,
    total
  TTL: expires_at (30 days)
```

Single item per cart; updates via UpdateItem with list_append, list_remove. Whole cart fits in one item if < 400 KB.

### 3.3 Atomic add-to-cart

```python
table.update_item(
  Key={'user_id': uid},
  UpdateExpression='SET #items = list_append(if_not_exists(#items, :empty), :new), updated_at = :now',
  ExpressionAttributeNames={'#items': 'items'},
  ExpressionAttributeValues={
    ':new': [{'sku': 'X', 'qty': 1, 'price_at_add': 9.99}],
    ':empty': [],
    ':now': int(time.time()),
  }
)
```

One round-trip; atomic; concurrency-safe (DynamoDB serializes per-key).

### 3.4 The list grows — when to denormalize

If a cart has 50+ items, the item gets large; every UpdateItem rewrites the full list. Cost grows linearly in cart size.

**Alternative: per-item rows**
```
PK: "CART#" + user_id
SK: "ITEM#" + sku
Attributes: qty, price_at_add, added_at
```

- Each AddToCart = one PutItem (~1 KB).
- ReadCart = Query(PK="CART#" + user_id) → returns all items.
- RemoveFromCart = DeleteItem.

This pattern (entity by PK, items by SK) is the **item-collection** pattern. Use when entity has multiple sub-items that change independently.

### 3.5 TTL for abandoned carts

```yaml
TTL attribute: expires_at (epoch seconds)
```

DynamoDB checks every ~48 hours; expired items auto-deleted (no read/write cost). Effectively **free garbage collection**.

```python
# Set TTL on cart update
expires_at = int(time.time()) + 30 * 86400
```

A common bug: not refreshing TTL on update → cart expires while user is active. Always update TTL on every cart write.

### 3.6 Stream consumer for cart events

```
DynamoDB Streams: NEW_AND_OLD_IMAGES.
Lambda triggers on every modify.
Use cases:
  - "Cart abandoned" detection (no update for 1 hour).
  - Real-time analytics.
  - Recommendation system updates.
```

### 3.7 Trade-offs and alternatives

| Approach | Trade |
|---|---|
| **DynamoDB single-item cart** | Simple; cart size limit |
| **DynamoDB item-collection** | Scales; query cost |
| **Redis** | µs latency; durability concerns |
| **Postgres** | Rich queries; ops cost |
| **ElastiCache (Redis)** | Hot path; pair with DDB for durability |

### 3.8 What I'd actually do

For carts: DynamoDB item-collection pattern (PK=CART#user, SK=ITEM#sku). TTL for abandonment. Streams for analytics. ElastiCache in front of DDB for the hottest reads (current logged-in users). PITR for accidental data loss.

---

## 4. Scenario 3 — Single-Table Design for Complex Domain

### 4.1 The problem

Domain has: users, orders, order items, payments, addresses. Multi-faceted access patterns:
- Get user with their orders (last 10).
- Get order with items and payment status.
- Get all payments for a user.
- Get all addresses for a user.
- Get an order by ID.

In RDBMS: 5 tables, joins. In DynamoDB: single-table design (Rick Houlihan's pattern).

### 4.2 The single-table layout

```
Table: app
  PK (partition key)
  SK (sort key)

Items:
  PK="USER#u1",        SK="META",            type=user, name=...
  PK="USER#u1",        SK="ADDR#a1",         type=address, street=...
  PK="USER#u1",        SK="ADDR#a2",         type=address, ...
  PK="USER#u1",        SK="ORDER#o1",        type=order_summary, total=, status=...
  PK="USER#u1",        SK="ORDER#o2",        type=order_summary, ...
  PK="ORDER#o1",       SK="META",            type=order, total=, ...
  PK="ORDER#o1",       SK="ITEM#i1",         type=order_item, sku=, qty=...
  PK="ORDER#o1",       SK="ITEM#i2",         type=order_item, ...
  PK="ORDER#o1",       SK="PAYMENT#p1",      type=payment, amount=, status=...
```

Each row's PK groups related items; SK organizes within. One-table holds everything.

### 4.3 Query patterns

```python
# Get user with addresses & order summaries:
Query(PK="USER#u1")  # returns USER#META, ADDR#*, ORDER#* — all in one query.

# Get user metadata only:
GetItem(PK="USER#u1", SK="META")

# Get order with items and payment:
Query(PK="ORDER#o1")  # returns ORDER#META, ITEM#*, PAYMENT#*

# Get all payments for a user:
# Need a GSI:
GSI1: PK1=user_id, SK1="PAYMENT#" + payment_id
Query(GSI1, PK1=u1)
```

### 4.4 GSIs for inverted access

A common pattern: define generic GSIs (`GSI1PK`, `GSI1SK`) and have items populate them based on type.

```
For ORDER items:
  GSI1PK = "USER#" + user_id
  GSI1SK = "ORDER#" + order_id
→ Query GSI1 by user → all orders.

For PAYMENT items:
  GSI1PK = "USER#" + user_id
  GSI1SK = "PAYMENT#" + payment_id
→ Query GSI1 by user → all payments.

(Each item type defines what its GSI keys are.)
```

This is the **inverted index pattern**. The same GSI serves multiple access patterns by overloading.

### 4.5 Why single-table

Pros:
- One round-trip for related data: `Query(PK="ORDER#x")` returns metadata, items, payment.
- Atomic operations across related entities.
- Better cost (fewer GetItem calls).
- Aligned with DynamoDB's partition-locality model.

Cons:
- Hard to read; the table looks chaotic.
- Schema migrations are app-level (no FK / no constraints).
- Item-overloading means careful type discipline (every item has a `type` attribute).
- Onboarding new engineers takes longer.

### 4.6 When NOT to single-table

- Domain entities are independent and rarely accessed together.
- Different entities have wildly different access patterns / capacities.
- Team unfamiliar with the pattern; documentation overhead high.

For independent entities (a _user table, an audit log, a product catalog_) — multi-table is fine. Single-table shines for **closely-related, frequently-co-accessed entities**.

### 4.7 The schema design discipline

Single-table demands:
1. **Enumerate every access pattern** before designing keys.
2. **Reserve space** for GSI keys (`GSI1PK`, `GSI1SK`, `GSI2PK`, …).
3. **Type tag every item** (`type` attribute).
4. **Document the schema** rigorously — table definition is 90% of the project knowledge.

If someone asks "can we add a query for X?", the answer is either "yes, here's how to use existing keys/indexes" or "we need a new GSI / table re-design". Either is honest; both require thought.

### 4.8 Trade-offs and alternatives

| Approach | Trade |
|---|---|
| **Single-table DynamoDB** | Cost-efficient; opaque |
| **Multi-table DynamoDB** | Cleaner; more queries |
| **RDBMS** | Flexible queries; costs at scale |
| **NoSQL document (Mongo, DocDB)** | Easier; performance ceiling |
| **NewSQL (Spanner, CockroachDB)** | Best of both; expensive |

### 4.9 What I'd actually do

For a product with well-known, related domain entities and high read scale: **single-table design**. Spend days upfront on access-pattern enumeration. 
Build Terraform module that defines the schema, GSIs, and a documented item taxonomy. The discipline pays back over years.

For a CRUD app with shifting requirements: **multi-table or RDBMS**. Single-table's rigidity hurts more than it helps.

---

## 5. Scenario 4 — Time-Series / IoT Telemetry

### 5.1 The problem

IoT devices emitting telemetry. 1M devices, 10 events/sec each = 10M events/sec total. Need recent data per device queryable; long-term analytics.

### 5.2 The naive (broken) design

```
PK = device_id
SK = timestamp (epoch ms)
Attributes: payload
```

Issues:
- One device's writes all hit one partition (PK = device_id). With 10 events/sec/device, manageable.
- Read-by-recent works: Query with PK=device_id, SK descending, Limit=100.
- **But**: a device with 10K events/sec saturates its partition (1K WCU/partition).
- **And**: a "popular" device draws all reads to one partition.

### 5.3 The time-bucketed pattern

```
PK = device_id + "#" + bucket(timestamp, 1 hour)
SK = timestamp
```

Each device has hourly partitions. Queries for last hour: one PK. Queries for last day: 24 PKs (use BatchGetItem or parallel Queries).

This **bounds partition size** and **caps per-partition write rate** even as devices accumulate years of data.

### 5.4 The TTL pattern

```
TTL attribute: expires_at = timestamp + 30 days
```

Old records auto-delete. No explicit cleanup. No write cost. No reaper.

For longer retention: stream to S3 via Kinesis Data Streams for DynamoDB or Streams + Lambda → S3 (Parquet, partitioned by date).

### 5.5 Cold + hot tiering

```
Hot path (last 30 days):     DynamoDB.
Cold path (everything):       S3 in Parquet, partitioned by year/month/day/device.
Analytics on cold:            Athena, EMR.
```

DynamoDB stores recent for low-latency device-specific queries. S3 stores forever for analytics. Streams + Firehose moves data from one to the other.

### 5.6 Per-device aggregates

For "average temperature for device X in last 24 hours":
- Naive: scan 24 hours of per-device data, aggregate. Expensive.
- Better: precomputed rollups in another table:
  ```
  PK = device_id
  SK = bucket(ts, 1 hour)
  Attributes: avg_temp, count, max, min
  ```
  Updated by stream consumer or batch job. Reads are O(1) per bucket.

### 5.7 At MAANG scale

```
1M devices × 10 events/sec = 10M WPS.
At 1KB each → 10M WCU.
Cost: 10M × $0.00065/hr × 720 = $4.7M/month provisioned.
```

A single DDB table serving 10M WPS is feasible but expensive. Consider:
- **Timestream** (managed time-series): cheaper for time-series specifically.
- **Kinesis → S3 Parquet → Athena**: cheapest for analytics-heavy workloads.
- **InfluxDB / TimescaleDB**: self-managed; richer query language.

DynamoDB shines for **device-keyed point queries with low latency**. For pure analytics: use S3.

### 5.8 Trade-offs and alternatives

| Approach | Trade |
|---|---|
| **DynamoDB time-bucketed** | Hot point queries; expensive at PB |
| **AWS Timestream** | Time-series specific; mid-cost |
| **InfluxDB / Timescale** | Self-managed; rich features |
| **Kinesis + S3 Parquet + Athena** | Cheapest analytics; high latency |
| **OpenSearch** | Search + dashboard; expensive |

### 5.9 What I'd actually do

For IoT telemetry at MAANG scale:
- **Hot 30 days in DynamoDB** with time-bucketed PK for low-latency device queries.
- **All data in S3** via Kinesis Firehose, Parquet, partitioned for Athena.
- **TTL on DDB table** to age out hot data.
- **Pre-aggregated rollups** in a second DDB table for dashboards.

---

## 6. Scenario 5 — Leaderboard / Sorted Ranking

### 6.1 The problem

Game leaderboard: top 100 players globally. Real-time updates. Millions of players. "What's my rank?" query for any player.

### 6.2 The naive design

```
PK = player_id
Attributes: score, name, ...
```

Doesn't support "top 100 by score" — would need a Scan.

### 6.3 The bucketed leaderboard

```
PK = "GAME#" + game_id    # one game, all scores share PK
SK = score (zero-padded for lexicographic sort)

Query:
  Query(PK="GAME#" + game_id, ScanIndexForward=False, Limit=100)
  → Top 100 by score.
```

But: one PK = one partition = 1K WCU ceiling. Game with 100K writes/sec ≫ 1K → throttle.

### 6.4 Sharded leaderboard

```
PK = "GAME#" + game_id + "#" + (player_id % N)
SK = score (zero-padded)

Writes spread across N partitions.
Reads: parallel Query across N partitions, merge top 100 client-side.
```

For top-K: read all N shards' top-100, merge, take top 100.

For exact rank ("am I rank 12,348?"): more complex. Often solved with a separate sorted-set service (Redis ZSET) for live leaderboard plus DynamoDB for durable state.

### 6.5 The Redis alternative

Redis sorted sets (ZSET) are designed for exactly this:

```
ZADD leaderboard 12345 player_id
ZRANGE leaderboard 0 99 WITHSCORES REV   # top 100
ZRANK leaderboard player_id              # rank of player
```

µs latency, exact rank, atomic. The cost: Redis is in-memory, less durable, requires separate operation.

Common production pattern: **Redis for live leaderboard, DynamoDB for durable score history and recovery**.

### 6.6 Trade-offs and alternatives

| Approach | Trade |
|---|---|
| **DDB single PK** | Simple; throughput-capped |
| **DDB sharded** | Scales; merge cost |
| **Redis ZSET** | Best for live leaderboard; volatile |
| **OpenSearch sorted** | Dashboard-friendly; not real-time |

### 6.7 What I'd actually do

For real-time live leaderboards: **Redis ZSET** (ElastiCache) backed by **DynamoDB** for durability. Score writes go to both (or DDB → Streams → Lambda → Redis sync). 
Top-K and exact rank from Redis. Recovery from DDB.

---

## 7. Scenario 6 — Multi-Tenant SaaS

### 7.1 The problem

SaaS with 10K customers. Each has its own users, projects, documents. Need:
- Per-tenant data isolation.
- Per-tenant performance isolation (one tenant's bursts don't slow others).
- Per-tenant billing (track usage).
- Cross-tenant queries are rare and admin-only.

### 7.2 Data isolation via composite PK

```
PK = "TENANT#" + tenant_id + "#" + entity_type
SK = entity_id

E.g.:
  PK="TENANT#abc#USER",     SK="u1",   ...
  PK="TENANT#abc#PROJECT",  SK="p1",   ...
  PK="TENANT#xyz#USER",     SK="u2",   ...
```

All of tenant A's data is keyed under `TENANT#abc#`. Easy IAM scoping by prefix.

### 7.3 IAM access scoping

```json
// Customer A's role
{
  "Effect": "Allow",
  "Action": ["dynamodb:GetItem", "dynamodb:PutItem", "dynamodb:Query"],
  "Resource": "arn:aws:dynamodb:*:*:table/app",
  "Condition": {
    "ForAllValues:StringLike": {
      "dynamodb:LeadingKeys": ["TENANT#abc#*"]
    }
  }
}
```

DynamoDB IAM conditions can scope access by partition key prefix. Tenant A literally cannot read Tenant B's data. Critical for multi-tenant SaaS.

### 7.4 Per-tenant performance isolation

Single shared table → noisy neighbor. Tenant X's burst eats capacity. Mitigations:

```
1. Provisioned mode + per-tenant rate limit (in app or via API Gateway throttle).
2. On-demand mode (pay per request; AWS handles scaling).
3. Dedicated table per "premium" tenant.
4. Per-tenant read/write tracking; throttle in app.
```

For most SaaS: on-demand handles bursts; in-app rate limiting protects against runaway tenants. Dedicated tables only for enterprise tier.

### 7.5 Per-tenant cost allocation

```
DynamoDB PartiQL or Streams + analytics → calculate per-tenant RCU/WCU.
Or: tag-based cost allocation (one tag per tenant) — but only for table-level, not item-level.
```

Item-level cost tracking requires app instrumentation (count requests per tenant). The bill from AWS is per table/index; you derive per-tenant from your own logs.

### 7.6 Mega-tenant problem

A 100× tenant (Disney+) on shared table → may exceed per-partition limits. Solutions:
- Migrate to dedicated table (operationally invasive).
- Use sub-shard prefix in their PK.
- Use on-demand mode for them only (separate table).

This is the "noisy neighbor" pattern from §16 of the stateful doc applied to DynamoDB.

### 7.7 Trade-offs and alternatives

| Approach | Trade |
|---|---|
| **Shared table, prefix-by-tenant** | Cheapest; isolation via IAM |
| **Table per tenant** | Strongest isolation; ops cost (1000s of tables) |
| **Hybrid (shared + dedicated for premium)** | Best balance |
| **RDS schema-per-tenant** | If queries are SQL-rich; cluster ops |

### 7.8 What I'd actually do

For multi-tenant SaaS at MAANG scale:
- **Shared single-table DynamoDB** with `TENANT#<id>#` prefix.
- **IAM conditions** scoping per-tenant access.
- **On-demand mode** for capacity (or provisioned with auto-scaling).
- **Per-tenant dedicated tables** for top 1% (mega-tenants).
- **In-app rate limiting** to protect against noisy neighbors.
- **Cost allocation** via instrumentation.

---

## 8. Scenario 7 — Hot Partition Mitigation

### 8.1 The problem

A naturally hot key. A celebrity user. A globally-shared counter. A "system" account every transaction touches. The hot partition is the #1 cause of DynamoDB outages.

### 8.2 Detecting hot partitions

```
CloudWatch metrics:
  - ConsumedReadCapacityUnits per partition.
  - ConsumedWriteCapacityUnits per partition.
  - ThrottledRequests count.
  - SystemErrors.

CloudWatch Contributor Insights:
  - Top-K partition keys consuming most RCU/WCU.
  - Critical for production.

DynamoDB adaptive capacity:
  - Auto-rebalances to give hot partitions more RCU/WCU.
  - Sub-minute reaction.
  - Helps with moderate hot-spots; not unlimited.
```

Even with adaptive capacity, the per-partition ceiling (3K RCU, 1K WCU) eventually binds.

### 8.3 Write-side sharding

```
Original:
  PK = "global_counter"
  WCU saturates at 1K WPS.

Sharded:
  PK = "global_counter#" + (random 0..N-1)
  Random shard per write.
  Read: scatter-gather across N shards; sum.
```

For N=64 shards: 64K WPS capable, summed read at the cost of N reads.

### 8.4 Read-side sharding (for celebrity user)

```
Celebrity user's profile, read 1M times/sec.

Option A: Cache (DAX or Redis) → 99% hit ratio → DDB sees 10K reads/sec. Done.
Option B: Replicate item to N shards on write → reads pick random replica.
  PK = "USER#" + user_id + "#" + (random 0..N-1)
  Writes: PutItem to all N (transaction or async).
  Reads: random shard.
```

Caching is almost always the right answer. Read replication trades write amplification for read scalability — only justified when cache miss is intolerable.

### 8.5 The composite PK trick

```
For a "post" with viral hot reads and per-user-comment writes:

Single PK="POST#<id>" — comments serialized; can throttle.
Composite PK="POST#<id>#" + (user_id % N) for comment writes:
  Each comment lands on a sharded partition.
  Reading all comments = N parallel queries.
```

### 8.6 Adaptive capacity and burst limits

```
Adaptive capacity:
  Auto-rebalances within seconds.
  Up to 1000% short burst.
  Won't help if sustained load > limits.

Burst capacity:
  Each partition has 5 minutes of burst budget.
  After exhaustion: hard throttle.
```

Knowing burst is a buffer, not a sustained capacity, is critical for capacity planning.

### 8.7 On-demand vs provisioned for hot partitions

```
Provisioned: hot partition exceeds → throttle.
On-demand: hot partition gets adaptive capacity faster; auto-scales.
  → On-demand handles unpredictable hot-spots better.
  → But on-demand $$ at high volume.
```

For workloads with unpredictable hot keys: on-demand. For predictable: provisioned + sharding strategy.

### 8.8 Trade-offs

| Mitigation | Trade |
|---|---|
| **Cache (DAX/Redis)** | Best for read-hot; not for write-hot |
| **Write sharding** | Scales writes; read scatter-gather |
| **Read replication** | Scales reads; write amplification |
| **On-demand** | Auto-scale; cost |
| **Adaptive capacity** | Free; limited |
| **Per-key dedicated table** | Niche; ops cost |

### 8.9 What I'd actually do

For hot partition concerns:
- Day 1: design partition key for high cardinality. Avoid "ALL", "GLOBAL", or any single-value PK.
- For inevitable hot reads: DAX or ElastiCache in front.
- For hot writes: sharded counters + scatter-gather reads.
- CloudWatch Contributor Insights enabled.
- Alarm on per-partition throttling.
- Quarterly review: which keys are getting hot? Plan re-sharding.

---

## 9. Scenario 8 — Idempotency Key Store

### 9.1 The problem

Payments / order systems need idempotency keys (covered in transactions doc). Need a fast, durable key-value store for "have I seen this request before?"

### 9.2 The design

```
Table: idempotency_keys
  PK: key (UUID or hash of request)
  Attributes:
    status: 'IN_PROGRESS' | 'DONE' | 'FAILED'
    response: <serialized result>
    created_at
  TTL: expires_at (24-48 hours)
```

Fast point lookups, conditional writes for atomicity.

### 9.3 Atomic create-or-fail

```python
try:
  table.put_item(
    Item={'key': k, 'status': 'IN_PROGRESS', 'created_at': now, 'expires_at': now + 86400 * 2},
    ConditionExpression='attribute_not_exists(#k)',
    ExpressionAttributeNames={'#k': 'key'},
  )
  # we own this idempotency key; do the work.
  do_work()
  table.update_item(Key={'key': k}, UpdateExpression='SET #s = :done, response = :r',
                    ExpressionAttributeNames={'#s': 'status'},
                    ExpressionAttributeValues={':done': 'DONE', ':r': resp})
except ConditionalCheckFailedException:
  # someone else has it; fetch their result or fail
  existing = table.get_item(Key={'key': k})['Item']
  if existing['status'] == 'DONE': return existing['response']
  raise InProgressError("retry later")
```

### 9.4 At scale considerations

```
At 100K idempotent operations/sec:
  100K WCU + 100K RCU.
  Storage: 100K × 86400 × 2 days = 17B keys × ~1 KB = 17 TB.
  
With TTL: storage stays bounded.
```

Use on-demand or provisioned with auto-scaling. TTL is the GC. Don't try to delete manually.

### 9.5 In transactions

```python
table.update_item(
  Key={'key': k},
  UpdateExpression='SET #s = :done',
  ConditionExpression='attribute_exists(#k) AND #s = :inprogress',
  ...
)
# Wrap with the actual side effect in TransactWriteItems for true atomicity.

dynamodb.transact_write_items(TransactItems=[
  {'Update': {... idempotency key set to DONE ...}},
  {'Put': {... order record ...}},
])
```

This gives true atomicity between idempotency state and side-effect. Pattern from transactions doc applies directly.

### 9.6 Trade-offs

| Approach | Trade |
|---|---|
| **DynamoDB** | Fast, durable, scalable, TTL-managed |
| **Redis** | Faster, less durable |
| **Postgres unique constraint** | Simpler; doesn't scale to 100K WPS easily |

### 9.7 What I'd actually do

For idempotency at scale: DynamoDB on-demand with TTL. Conditional writes for atomicity. TransactWriteItems if multi-row atomic with side effect.

---

## 10. Scenario 9 — Distributed Locks

### 10.1 The problem

Only one of N workers should run a job. Coordinated lock with TTL (so dead worker doesn't hold lock forever).

### 10.2 The conditional-write pattern

```python
def acquire_lock(name, holder, ttl_seconds=60):
  now = int(time.time())
  expires = now + ttl_seconds
  try:
    table.put_item(
      Item={
        'lock_name': name,
        'holder': holder,
        'acquired_at': now,
        'expires_at': expires,
      },
      ConditionExpression='attribute_not_exists(lock_name) OR expires_at < :now',
      ExpressionAttributeValues={':now': now},
    )
    return True
  except ConditionalCheckFailedException:
    return False

def release_lock(name, holder):
  try:
    table.delete_item(
      Key={'lock_name': name},
      ConditionExpression='holder = :h',
      ExpressionAttributeValues={':h': holder},
    )
  except ConditionalCheckFailedException:
    pass  # we don't hold it (or it expired)
```

Conditional write: lock is acquired if no existing lock OR existing lock has expired. Atomic.

### 10.3 The fencing token pattern

```python
def acquire_lock_with_fencing(name, holder, ttl_seconds=60):
  now = int(time.time())
  try:
    response = table.update_item(
      Key={'lock_name': name},
      UpdateExpression='SET holder=:h, acquired_at=:a, expires_at=:e ADD fence_token :one',
      ConditionExpression='attribute_not_exists(holder) OR expires_at < :now',
      ExpressionAttributeValues={':h': holder, ':a': now, ':e': now+ttl_seconds, ':one': 1, ':now': now},
      ReturnValues='ALL_NEW',
    )
    return response['Attributes']['fence_token']  # monotonic
  except ConditionalCheckFailedException:
    return None
```

Each acquisition increments a monotonic token. Downstream operations include the token; reject stale ones (the GC-pause split-brain mitigation from the stateful doc).

### 10.4 Lock at scale

For many fine-grained locks (millions): DynamoDB scales fine. For very-high-frequency lock contention on a single key: Redis SETNX with Redlock is faster, though more complex correctness story.

### 10.5 Trade-offs

| Approach | Trade |
|---|---|
| **DynamoDB conditional write** | Native; durable; cheap |
| **Redis SETNX EX** | Faster; less durable |
| **etcd / ZooKeeper** | Strong primitives; ops |
| **Postgres advisory lock** | Tied to DB instance |

### 10.6 What I'd actually do

For distributed locks in AWS: DynamoDB with TTL + fencing token. Simple, durable, no extra infrastructure.

---

## 11. Scenario 10 — Rate Limiting / Token Bucket

### 11.1 The problem

Per-user rate limit: 100 requests/minute. Distributed across N API servers. Counter must be coordinated.

### 11.2 The atomic counter pattern

```python
def check_rate_limit(user_id, limit=100, window_sec=60):
  now = int(time.time())
  bucket = now // window_sec  # current minute
  pk = f"{user_id}#{bucket}"
  try:
    response = table.update_item(
      Key={'pk': pk},
      UpdateExpression='ADD count :one SET expires_at = if_not_exists(expires_at, :exp)',
      ConditionExpression='attribute_not_exists(#c) OR #c < :limit',
      ExpressionAttributeNames={'#c': 'count'},
      ExpressionAttributeValues={':one': 1, ':limit': limit, ':exp': now + window_sec * 2},
      ReturnValues='UPDATED_NEW',
    )
    return True
  except ConditionalCheckFailedException:
    return False
```

Single atomic operation: increment + check + return. Per-user, per-minute bucket.

### 11.3 Sliding window

```python
# Approximation: weighted current + previous window
prev_bucket = bucket - 1
fraction = (now % window_sec) / window_sec
total = current_count + prev_count * (1 - fraction)
```

Two GetItem calls + math; less precise than ZSET-based sliding log but lower cost.

### 11.4 At scale

```
1M users × 1 rate-limit-check per request × 100 reqs/sec aggregate:
  Hot key per user is unlikely (each user is a separate PK).
  Scales naturally.

Concern:
  - WCU per check: 1.
  - 100K checks/sec → 100K WCU.
  - Use on-demand or auto-scaling.
```

Each user's bucket is its own partition; no hot-key issues unless one user is making millions of req/sec — at which point you've already detected and blocked.

### 11.5 Redis alternative

Redis token bucket / leaky bucket: faster, in-memory. For ms-level latency on rate checks: Redis. For durability + simplicity: DDB.

### 11.6 What I'd actually do

For per-tenant rate limiting in serverless / Lambda environments: **DynamoDB**. For ultra-high-frequency in-process: Redis.

---

## 12. Scenario 11 — Audit Trail / Append-Only Log

### 12.1 The problem

Audit every change to user data. Immutable; queryable per user; queryable per resource; long retention.

### 12.2 The design

```
Table: audit
  PK: "USER#" + user_id
  SK: timestamp_micros (sortable; epoch microseconds)
  Attributes:
    action, resource_type, resource_id, ip, before, after, ...
```

Per-user partition holds all events. Query by user with timestamp range.

### 12.3 Cross-cut by resource

```
GSI:
  GSI1PK = "RESOURCE#" + resource_type + "#" + resource_id
  GSI1SK = timestamp_micros
```

Now: query all events for a specific resource.

### 12.4 Immutability

```python
table.put_item(
  Item={...},
  ConditionExpression='attribute_not_exists(#sk)',
  ExpressionAttributeNames={'#sk': 'sk'},
)
```

Conditional put fails if SK already exists. With unique sortable timestamps, ensures append-only.

### 12.5 Long retention

```
Hot 90 days:        DynamoDB.
Older:              S3 via Streams + Firehose, Parquet partitioned.
```

DynamoDB for queryable recent; S3 for long-term storage and analytics.

### 12.6 At scale

```
1M events/min × 60 × 24 × 90 = 130B records over 90 days.
At 1KB each: 130 TB.
$0.25/GB/mo × 130,000 GB × 3 months = $97K total.
```

Significant. Tiering to S3 makes sense earlier than 90 days for high-volume audits.

### 12.7 Trade-offs

| Approach | Trade |
|---|---|
| **DynamoDB only** | Fast queries; expensive at scale |
| **DDB hot + S3 cold** | Best of both; complexity |
| **S3 only (Iceberg)** | Cheapest; not real-time |
| **Postgres** | Easier; doesn't scale to billions cheap |

### 12.8 What I'd actually do

DynamoDB for hot audit (30-90 days) with GSI for resource-cross-cut. Streams → Firehose → S3 Parquet for archive. Athena for cold queries.

---

## 13. Scenario 12 — Workflow / Job State

### 13.1 The problem

Track state of long-running jobs (e.g., video transcoding, large data exports). State machine: PENDING → RUNNING → COMPLETE / FAILED. Reads on job_id; admin queries on user_id; status.

### 13.2 The design

```
Table: jobs
  PK: job_id
  SK: 'META' (single item per job)
  Attributes: state, owner, started_at, completed_at, params, result
  
GSI1: by user
  PK: user_id
  SK: created_at
GSI2: by state (sparse, only for active jobs)
  PK: state
  SK: started_at
```

Sparse GSI: only items with `state = RUNNING` populate GSI2 (use empty/null on completion). Lets admin "list all running jobs" cheaply.

### 13.3 State transitions

```python
def transition(job_id, from_state, to_state):
  table.update_item(
    Key={'job_id': job_id, 'sk': 'META'},
    UpdateExpression='SET #s = :to',
    ConditionExpression='#s = :from',
    ExpressionAttributeNames={'#s': 'state'},
    ExpressionAttributeValues={':from': from_state, ':to': to_state},
  )
```

Conditional: only succeed if current state matches expected. Prevents double-transition or stale workers.

### 13.4 Optimistic concurrency

Add a `version` attribute incremented on every update; conditional `version = :prev`. Reject stale writes.

### 13.5 Sparse GSI for active jobs

```
Item attributes for completed job:
  state = "COMPLETE"
  active_marker (not present)
  
Item attributes for running job:
  state = "RUNNING"
  active_marker = "1"

GSI on active_marker:
  Only items with active_marker present appear in GSI.
  Listing active jobs = scanning small index, not whole table.
```

Sparse GSIs are critical for "find all items where condition X" patterns.

### 13.6 Trade-offs

| Approach | Trade |
|---|---|
| **DynamoDB single-table** | Fast point lookups; sparse GSI |
| **Step Functions only** | No DDB needed for workflow |
| **DDB + Step Functions** | DDB stores state visible to UI; Step Functions executes |
| **Aurora** | SQL-rich queries; ops cost |

### 13.7 What I'd actually do

For job state at MAANG scale: DynamoDB single-table with versioned state, sparse GSI on active_marker. Step Functions for actual execution; updates DDB at each step. UI reads from DDB.

---

## 14. Scenario 13 — GSI Patterns (Inverted, Sparse, Composite)

### 14.1 Inverted index

For "look up A by B's value":

```
Main:
  PK = item_id
  Attributes: ..., owner_id, ...

GSI:
  PK = owner_id
  SK = item_id
```

Now: Query GSI by owner_id → all items owned. The GSI "inverts" the relationship.

### 14.2 Sparse GSI

```
Items whose attribute X is set → appear in GSI.
Items without X → don't appear.

Use cases:
  - Active vs completed
  - Tagged vs untagged
  - Premium vs free
  
Cost: storage in GSI only for items that match.
```

### 14.3 Composite SK

```
PK: user_id
SK: type + "#" + id  e.g., "ORDER#2026-01-23#abc"

Queries:
  Begins-with "ORDER#"     → all orders.
  Begins-with "ORDER#2026" → 2026 orders.
  Range "ORDER#2026-01"     → January 2026 orders.
```

Composite SKs encode hierarchy; begins-with queries enable prefix lookup.

### 14.4 GSI overload

```
GSI keys reused across item types:
  GSI1PK | GSI1SK | meaning
  ----------------------------
  USER#a | ORDER#1 | user's order
  USER#a | ADDR#x  | user's address
  ITEM#x | ORDER#1 | order containing item
```

Same GSI serves multiple access patterns by tagging. The "single GSI for many queries" pattern saves on GSI count (limit: 20).

### 14.5 GSI projection

```
ALL:                   all attributes (most expensive storage).
KEYS_ONLY:             only the keys (cheapest; need second GetItem for full data).
INCLUDE list:          specific attributes only (sweet spot).
```

For GSIs supporting list views: INCLUDE the displayed attributes. For "find then look up": KEYS_ONLY.

### 14.6 GSI eventual consistency

GSIs are *always* eventually consistent. ConsistentRead=True is not allowed on GSI. Lag is typically <1s; can spike under heavy write load.

If you need strong consistency on a derived view: maintain a second item in same partition (manually written via TransactWriteItems).

### 14.7 GSI cost

```
Each GSI:
  - Doubles write capacity for items that propagate.
  - Adds storage of projected attributes.
  - Adds query cost on the GSI.
```

20 GSIs × 1× write = 21× write capacity for that table. Be deliberate.

### 14.8 What I'd actually do

For complex domains:
- Enumerate access patterns first.
- Use composite SK for hierarchical queries.
- Sparse GSI for "find items where X".
- Inverted index for relationship traversal.
- INCLUDE projection with the minimum attributes needed for the query result.
- Document each GSI's purpose; remove unused ones (they cost write capacity).

---

## 15. Scenario 14 — Many-to-Many Relationships

### 15.1 The problem

Users follow other users (Twitter-style). Need:
- Get followers of user X.
- Get followees of user Y.
- Check "does A follow B?"

### 15.2 The adjacency list pattern

```
Table: relationships
  PK: "USER#" + user_id
  SK: "FOLLOWS#" + other_user_id  (when user follows other)
  Or: "FOLLOWED_BY#" + other_user_id (when other follows user)

Or single direction with GSI inverse:
Main:
  PK: "USER#" + follower
  SK: "FOLLOWS#" + followee
GSI1:
  PK1: "USER#" + followee
  SK1: "FOLLOWS#" + follower

Queries:
  Followees of X:    Query main table PK="USER#X", begins_with "FOLLOWS#"
  Followers of X:    Query GSI1 PK1="USER#X", begins_with "FOLLOWS#"
  Does A follow B?:  GetItem(PK="USER#A", SK="FOLLOWS#B")
```

### 15.3 Counts

For "how many followers does X have?":
- Naive: count Query results — expensive at millions.
- Better: maintain `followers_count` on the user item, atomic increment on follow/unfollow.

```python
# On follow:
TransactWriteItems:
  - Put: PK="USER#A", SK="FOLLOWS#B"          # follow record
  - Update: PK="USER#A", SK="META", followees_count += 1
  - Update: PK="USER#B", SK="META", followers_count += 1
```

Atomic across all three; counters stay consistent.

### 15.4 Hot followee (celebrity)

A celebrity user with 100M followers. Query for their followers = 100M items in one partition? No — the GSI partition for them holds 100M items.

Mitigations:
- **Don't query "all followers"** at runtime; this is an analytics/offline job.
- **Cache "follower count"** on the user item.
- **Pagination** for "my followers list" UI.
- **Stream-based fanout** for "celebrity posts to all followers" — see fan-out pattern.

### 15.5 Trade-offs

| Approach | Trade |
|---|---|
| **DDB adjacency list** | Native; fast |
| **Graph DB (Neptune)** | Better for traversals; ops cost |
| **RDBMS join table** | Familiar; doesn't scale |
| **Cassandra** | Same shape; self-managed |

### 15.6 What I'd actually do

For social-graph at scale: DDB adjacency list with inverse GSI. Cached counts. Neptune for true graph traversals (multi-hop). For scale beyond ~1B edges: serious investment in graph DB or custom solution.

---

## 16. Scenario 15 — Multi-Region Active-Active (Global Tables)

### 16.1 The problem

Globally-distributed product. Users in US, EU, APAC. Need low-latency reads and writes for nearest region. Survive region failure.

### 16.2 Global Tables v2

```
Enable Global Tables:
  Region us-east-1 → replicates to eu-west-1, ap-southeast-1.
  Each region: read/write locally.
  Replication: async, ~1-second lag typical.
  Conflict: LWW (last writer by timestamp wins).
```

### 16.3 The conflict model

```
Region A: UPDATE user.name = "Alice" at t1.
Region B: UPDATE user.name = "Bob"   at t2.
Replication crosses; both regions converge.
If t2 > t1: name = "Bob" everywhere. "Alice" silently lost.
```

LWW means writes can be silently overwritten. Mitigations:
- For counters: use atomic ADD operations (commute under LWW).
- For lists/sets: use SS / NS (string set / number set) which merge; or maintain in separate items.
- For business-critical updates: use a **leader region per item** (route writes to home region only).

### 16.4 Atomic operations and Global Tables

```
Atomic counter (ADD):
  Region A: ADD count 1 (count was 5; now 6)
  Region B: ADD count 1 (count was 5; now 6)
  Replication: each side applies the other's delta.
  Final: count = 7 in both. Correct!

vs SET:
  Region A: SET count = 6
  Region B: SET count = 7  (read 5, increment, set)
  Replication: LWW. One winner; off by 1.
```

Use **atomic increments / decrements (ADD)** for counters in Global Tables. Don't read-modify-write.

### 16.5 Cost

```
Global Tables:
  Each replicated region pays full WCU + RCU.
  3-region: 3× write capacity cost.
  Plus inter-region replication transfer ($0.02/GB).
```

Significant cost amplifier. Replicate only what truly needs global presence.

### 16.6 Region failure

```
us-east-1 fails:
  Reads in us-east-1: client must failover (your responsibility).
  Reads in eu-west-1: continue (no impact).
  Writes from us-east-1 clients: route to eu-west-1.
  When us-east-1 recovers: replicate accumulated writes back.
```

Application must handle the failover. AWS doesn't auto-route clients; you do (Route 53 health checks, app logic).

### 16.7 Trade-offs

| Approach | Trade |
|---|---|
| **Global Tables** | AWS-native; LWW conflict |
| **Region-pinned + replication for backup** | Simpler; cross-region read latency |
| **Spanner / CockroachDB** | Strong consistency; expensive |
| **Cassandra cross-DC** | Self-managed; tunable consistency |

### 16.8 What I'd actually do

For multi-region active-active at MAANG scale:
- **Global Tables for tier-1 data** (user identity, session, cart).
- **Atomic operations** wherever possible to avoid LWW issues.
- **Region affinity** for users (write to home region; replicate for read).
- **Region-pinned writes** for financial/critical state (Spanner-class or single-region).
- DR drills.

---

## 17. Scenario 16 — TTL / Time-Bounded Data

### 17.1 The problem

Sessions, OTPs, temporary tokens, ephemeral cache, abandoned carts. Auto-expire without explicit cleanup.

### 17.2 The TTL feature

```
Table: tokens
  PK: token
  Attributes: user_id, ...
  TTL attribute: expires_at (epoch seconds)
```

DynamoDB scans periodically (~48 hours guarantee) and removes expired items. **No write/read cost** for the deletion.

### 17.3 What TTL is good for

- Sessions / cookies.
- OTPs (5-minute expiry).
- Magic-link tokens.
- Cached responses.
- Abandoned shopping carts.
- Rate-limit windows.

### 17.4 What TTL is NOT good for

- Anything requiring exact deletion timing (TTL is best-effort within 48 hours).
- Compliance retention (must delete by date X) — use explicit deletion.
- Items that need to "fail closed" on expiration — app must check expires_at, not rely on deletion.

### 17.5 The TTL gotcha

```
User logs in; sets session expires_at = now + 1 hour.
After 1 hour: session expires; should be inaccessible.

Reality: TTL takes up to 48 hours to actually delete. Item still exists.
Bug: app reads item, doesn't check expires_at, treats as valid.
Fix: ALWAYS check expires_at >= now in app code, regardless of TTL.
```

TTL is a janitor, not a gatekeeper. App must enforce expiry semantically.

### 17.6 TTL streams

```
DynamoDB Streams emit a "TTL deletion" event when TTL deletes an item.
  eventName: REMOVE
  userIdentity: { type: "Service", principalId: "dynamodb.amazonaws.com" }
```

Useful for: archive on expiry, downstream cleanup, metrics.

### 17.7 Trade-offs

| Approach | Trade |
|---|---|
| **DynamoDB TTL** | Free deletion; up to 48h delay |
| **Cron job DELETE** | Exact timing; capacity cost |
| **App-level check + lazy delete** | Predictable; no GC needed |

### 17.8 What I'd actually do

For ephemeral data: TTL with app-level expiry check. Stream consumer for archive on TTL deletion if needed.

---

## 18. Scenario 17 — Streams + Downstream Propagation

### 18.1 The problem

Every change to DDB must reach: search index (OpenSearch), data warehouse, audit log, cache invalidation.

### 18.2 Streams basics

```
Enable on table; choose view type:
  KEYS_ONLY:           PK + SK only.
  NEW_IMAGE:           item after change.
  OLD_IMAGE:           item before change.
  NEW_AND_OLD_IMAGES:  both.

24-hour retention. 2 readers max per shard for typical processing.
```

### 18.3 Lambda trigger

```
Lambda triggered by stream events.
Batched up to 100 records / 6 MB.
Auto-scales (one Lambda per shard concurrent).
DLQ on failure (configure!).
```

```python
def handler(event, context):
  for record in event['Records']:
    if record['eventName'] in ('INSERT', 'MODIFY'):
      new = record['dynamodb']['NewImage']
      # write to OpenSearch, etc.
    elif record['eventName'] == 'REMOVE':
      old = record['dynamodb']['OldImage']
      # delete from OpenSearch
```

### 18.4 EventBridge Pipes (modern alternative)

```
Stream → Pipe → (optional filter) → (optional enrich) → target

E.g.:
  DDB Stream → Pipe → filter (only orders) → enrich (add user data) → EventBridge bus
```

Replaces boilerplate Lambda glue code. Cheaper for source-to-sink without complex logic.

### 18.5 Kinesis Data Streams for DynamoDB

```
Alternative to native Streams: KDS for DynamoDB (longer retention, better tooling).
24-hour-to-365-day retention.
Multi-consumer fan-out.
Cost: $0.10/M events (vs Streams' $0.02/100K reads = $0.20/M reads).
```

For long-retention or multi-consumer: KDS for DDB. Otherwise native Streams.

### 18.6 The dual-write problem (revisited)

If app writes to DDB *and* publishes to Kafka separately: dual-write inconsistency. Streams (or Pipes) eliminate this — single write to DDB is the source; CDC propagates.

### 18.7 Trade-offs

| Approach | Trade |
|---|---|
| **Native Streams + Lambda** | Standard; 24h retention |
| **EventBridge Pipes** | Less code; same Streams under hood |
| **KDS for DynamoDB** | Long retention; multi-consumer; cost |
| **App-level dual write** | Don't (inconsistency risk) |

### 18.8 What I'd actually do

DynamoDB Streams + Pipes → EventBridge bus → multiple consumers (OpenSearch indexer, S3 archiver, cache invalidator). Pipes for source-to-target glue; Lambda only when transformation is non-trivial.

---

## 19. Scenario 18 — Counters and Aggregations

### 19.1 The problem

Track running totals: post likes, page views, sales. Needs to be fast and accurate.

### 19.2 Naive counter

```
UpdateItem with ADD:
  ADD likes 1 on PK="POST#x"
```

Atomic per-key. But: hot key. Single-partition write throughput cap = 1K WPS. A viral post gets throttled.

### 19.3 Sharded counter

```
PK = "POST#" + post_id + "#" + (random 0..N-1)
N = 64 typical.

Increment: pick random shard; ADD 1.
Read: Query all N shards; sum.
```

64-shard counter: 64K WPS.

```python
def increment(post_id, n=64):
  shard = random.randrange(n)
  table.update_item(
    Key={'pk': f"POST#{post_id}#{shard}", 'sk': 'COUNTER'},
    UpdateExpression='ADD likes :one',
    ExpressionAttributeValues={':one': 1},
  )

def read(post_id, n=64):
  total = 0
  for shard in range(n):
    item = table.get_item(Key={'pk': f"POST#{post_id}#{shard}", 'sk': 'COUNTER'}).get('Item')
    if item: total += item.get('likes', 0)
  return total
```

Read costs 64 RCU; writes are 1 WCU each, parallelizable.

### 19.4 Caching the read

```
Cache read result in Redis; refresh every 5 seconds.
Most reads hit cache; only periodic Refresh hits DDB.
```

For "view count" UX: cache for 1-5 seconds. Imprecise but cheap.

### 19.5 Stream-aggregated counters

```
Each like → write to a "likes_log" table (append-only, partitioned by user/post).
Stream consumer aggregates → updates "post_stats" table.

Or: Kinesis stream with stateful processor (Kinesis Data Analytics, Flink).
```

For very-high volume: stream processing is more scalable than direct increment.

### 19.6 Approximate counters

For ultra-high volume display ("1.2M views"):
- HyperLogLog for unique counts.
- Probabilistic increment (count with 1/100 probability; multiply on read).

Used for view counts where precision doesn't matter.

### 19.7 Trade-offs

| Approach | Trade |
|---|---|
| **Direct ADD** | Simple; hot-key cap |
| **Sharded counter** | Scales writes; read scatter-gather |
| **Cached read** | Lower read cost; staleness |
| **Stream-aggregated** | Most scalable; pipeline complexity |
| **Probabilistic** | Cheapest; approximate |

### 19.8 What I'd actually do

For social counters (likes, views) at viral scale:
- Sharded counters in DDB.
- Cached aggregates in Redis (1-5 sec TTL).
- Stream consumer for analytics pipeline.

---

## 20. Scenario 19 — Caching with DAX

### 20.1 The problem

Read-heavy DDB workload. p99 latency 5-10 ms. Need sub-ms.

### 20.2 DAX

```
DynamoDB Accelerator: write-through, read-through cache for DDB.
In-memory; cluster of nodes; transparent to app (uses DDB SDK).
Sub-ms latency for cached reads.
```

```python
import amazondax
dax = amazondax.AmazonDaxClient(endpoint_url=DAX_ENDPOINT)
table = dax.Table('my-table')
# Same API as DDB; transparently caches.
```

### 20.3 What DAX caches

```
Item cache:    cached GetItem responses by key.
Query cache:   cached Query/Scan responses by key+condition.
TTL: configurable (default 5 min item, 5 min query).
```

### 20.4 When DAX wins

- Read >> writes.
- Same items read frequently (high hit ratio).
- Latency-sensitive (single-digit ms unacceptable).

### 20.5 When DAX doesn't help

- Writes dominate.
- Reads access disjoint items (poor hit ratio).
- Strong consistency required (DAX uses eventual consistency).

### 20.6 DAX vs ElastiCache (Redis)

```
DAX:
  + DDB SDK transparent.
  + No app code change.
  - DDB-specific.
  - Item TTL only; no fine-grained invalidation.

Redis (ElastiCache):
  + Universal cache.
  + Rich data structures (sorted sets, hashes).
  + Fine-grained TTL.
  - App must explicitly cache.
```

For DDB-specific transparent caching: DAX. For richer caching needs: Redis.

### 20.7 Trade-offs

| Approach | Trade |
|---|---|
| **DAX** | Transparent; DDB-only |
| **ElastiCache Redis** | Universal; explicit cache logic |
| **Application-level cache** | In-process; per-pod |

### 20.8 What I'd actually do

For read-heavy DDB at scale: DAX for the table-cache layer, ElastiCache Redis for application-domain cache (sessions, computed views).

---

## 21. Scenario 20 — Hierarchical Data / Adjacency List

### 21.1 The problem

Org chart, file system, taxonomy tree. Need:
- Get all descendants of a node.
- Get path from root to node.

### 21.2 The adjacency list pattern

```
Table:
  PK: parent_id
  SK: child_id

Query(PK=parent_id) → direct children.
For full tree: recurse.
```

Recursion is N round-trips; slow for deep trees.

### 21.3 The materialized path pattern

```
Each node stores its full ancestor path:
  Item { id: "x", path: "/root/dept/team/x", parent: "team" }

Queries:
  Children of dept: Query GSI on parent="dept".
  All descendants of dept: Query (or Scan with filter) where path starts with "/root/dept/".
```

Path lookup is one query. Update on subtree-move requires updating path on all descendants.

### 21.4 The closure table pattern

```
Separate table: ancestor → descendant for every (ancestor, descendant) pair (transitive closure).
Item { ancestor: "dept", descendant: "x", depth: 2 }

Insert N: requires N entries (one per ancestor).
Delete N: requires removing all (any-ancestor, N) and (N, any-descendant).
```

Highest read efficiency; highest write cost. For trees with frequent reads, infrequent updates: closure table.

### 21.5 Trade-offs

| Approach | Trade |
|---|---|
| **Adjacency list** | Simple; recursion for descendants |
| **Materialized path** | One-query descendants; subtree-move heavy |
| **Closure table** | Best reads; write multiplier |
| **Graph DB (Neptune)** | True graph traversal; ops cost |

### 21.6 What I'd actually do

For hierarchies up to ~1M nodes: materialized path in DDB. For deep, complex graph: Neptune.

---

## 22. Scenario 21 — Geospatial Queries

### 22.1 The problem

"Find all users / restaurants within 5 miles of a point."

### 22.2 The geohash pattern

```
Geohash: encode (lat, lng) into a string.
  Prefix length determines precision.
  e.g., "9q8yy" = ~5km × 5km cell.

Items keyed by geohash:
  PK: geohash_prefix (e.g., 5 chars = ~5km cell)
  SK: geohash_full + entity_id

For "within X km of point P":
  Compute geohash of P with appropriate precision.
  Query that cell + 8 neighboring cells (Moore neighborhood).
  Filter results by exact distance.
```

### 22.3 Geohash precision trade

```
Precision 4: ~20 km cell  → fewer cells but coarse filter
Precision 6: ~1 km cell    → more cells, finer filter

Pick precision matching your typical query radius.
```

### 22.4 The DynamoDB Geo library

AWS Labs provides `dynamodb-geo` (Java/JS): handles geohash + bounding box queries on top of DDB.

### 22.5 Alternatives

| Approach | Trade |
|---|---|
| **DDB + geohash** | Native; manual |
| **PostGIS (Postgres)** | Best geospatial features; ops cost |
| **OpenSearch geo queries** | Easy; eventual; expensive |
| **Redis GEO** | µs latency; in-memory |

### 22.6 What I'd actually do

For sub-second nearby queries at moderate scale: DDB with geohash. For complex geo (polygons, routes): PostGIS. For "find drivers near me" at Uber scale: dedicated H3-indexed sharded service.

---

## 23. Scenario 22 — Schema Evolution / Migration

### 23.1 The problem

DynamoDB table evolves: new attributes, new GSIs, renamed fields, new entity types.

### 23.2 What's easy

- **Adding optional attribute**: free. New items have it; old items don't.
- **Adding GSI**: backfill takes time and capacity. Online but throughput-cost.
- **Adding new item type to single-table**: code change; no DDB action.

### 23.3 What's hard

- **Renaming attribute**: dual-write phase + backfill + cutover.
- **Changing attribute type**: equivalent to rename.
- **Removing a GSI**: drops the index; loses access pattern.
- **Re-shard partition key**: migration to new table.

### 23.4 The expand-contract pattern (revisited)

```
PHASE 1: Add new attribute alongside old.
PHASE 2: Code reads both; writes both.
PHASE 3: Backfill (Lambda iterates table; copies old → new).
PHASE 4: Code reads new only; still writes both.
PHASE 5: Code writes new only.
PHASE 6: Drop old attribute (reclaim storage via Update).
```

Each phase is a deploy. Some run for weeks.

### 23.5 Backfill strategy

```python
# Scan + update; throttle to control capacity
for item in table.scan(): 
  table.update_item(...)  # add new attr from old.
```

Issues:
- Scan is expensive (every item is read).
- Concurrent writes during backfill → race conditions.
- 100M-item table: backfill takes hours.

Better:
- **Streams + Lambda** to copy on each write going forward.
- **Parallel scan** (Segment + TotalSegments) to backfill in parallel.
- Use **on-demand mode during backfill** to avoid throttling.

### 23.6 GSI add/remove

```
Add GSI on existing table:
  AWS auto-backfills.
  Takes hours-to-days for large tables.
  Costs WCU on the new GSI for backfill.
  Doesn't impact main table reads/writes.
```

### 23.7 Table-to-table migration

For PK changes:

```
1. Create new table.
2. Code writes to BOTH old and new.
3. Backfill old → new.
4. Verify (item count, sample comparison).
5. Code reads from new (still writes both).
6. Code writes to new only.
7. Drop old table.
```

This is a multi-week project for production tables.

### 23.8 Trade-offs

| Approach | Trade |
|---|---|
| **Add attribute** | Free, fast |
| **Add GSI** | Online; backfill cost |
| **Rename attribute** | Multi-phase migration |
| **Change PK** | Full table migration |

### 23.9 What I'd actually do

Get the schema right at design time. For inevitable evolution:
- Add-only is cheap.
- Rename / restructure: expand-contract over weeks.
- Plan schema reviews before launch; decisions there cost less than migrations later.

---

## 24. Performance and Scaling Deep Dive

### 24.1 Capacity modes

```
PROVISIONED:
  Set RCU/WCU.
  Auto-scaling possible (target utilization).
  Cheaper at predictable load.
  Risk: under-provisioned → throttle.
  
ON-DEMAND:
  Pay per request.
  No capacity planning.
  Cold-start: initial throughput limited; ramps up.
  Bursts handled.
  ~7× more expensive than provisioned at sustained load.
  Switch between modes: 1× / 24 hours.
```

Default for new tables: on-demand. Switch to provisioned once load is predictable.

### 24.2 Auto-scaling

```
Target tracking: e.g., 70% utilization target.
Scales up/down based on actual usage.
Lag: minutes to scale (not seconds).
Min/max bounds.
```

For predictable diurnal patterns: auto-scaling works. For spiky bursts: on-demand or bigger min.

### 24.3 Adaptive capacity

```
DynamoDB internally rebalances WCU/RCU among partitions.
Sub-minute reaction.
Up to 5× burst above provisioned average for short bursts.
Doesn't help if total table provisioned is too low.
```

### 24.4 Hot partition diagnosis

```
CloudWatch Contributor Insights for DDB:
  Top-K partition keys by RCU/WCU.
  Crucial for finding hot keys.
  Always enable in production.
```

### 24.5 Pagination

```
Query/Scan responses limited to 1 MB.
Continue with LastEvaluatedKey → ExclusiveStartKey.
Don't assume one call returns all.
```

A common bug: not handling pagination, missing items.

### 24.6 Parallel scan

```
TotalSegments: split table into N segments.
Each worker scans one segment in parallel.
For backfills, exports.
Don't use parallel scan in production hot paths.
```

### 24.7 BatchGetItem / BatchWriteItem

```
Up to 100 items per Get / 25 items per Write.
Failed items returned for retry.
Saves API calls but not capacity.
```

For high-throughput systems, batches reduce Lambda cold-start invocation count.

---

## 25. Cost Engineering

### 25.1 The cost levers

```
1. Choose capacity mode wisely (provisioned for sustained, on-demand for bursty).
2. Reduce GSI count (each GSI adds 1× write).
3. Use sparse GSIs.
4. Project minimum attributes (KEYS_ONLY or INCLUDE).
5. Don't store blobs (offload to S3).
6. Compress payloads if large.
7. Use TTL for ephemeral data.
8. Stream to S3 for cold data.
9. Cache reads (DAX or Redis).
10. Avoid scans; use Query.
```

### 25.2 The on-demand vs provisioned crossover

```
At ~14% sustained utilization, provisioned breaks even with on-demand.
Above: provisioned cheaper.
Below: on-demand cheaper (no idle cost).
```

### 25.3 GSI cost amplification

```
1 GSI = 1× WCU on every write that propagates.
20 GSIs = 21× write cost.

Always: review GSI usage; remove unused.
```

### 25.4 Streams cost

```
$0.02 per 100K reads on stream shard.
At 1M writes/sec → 1M stream reads/sec → $20/sec → $50K/month.
Significant.

For very high volume: KDS for DDB or stream sampling.
```

### 25.5 Backups

```
PITR (continuous): $0.20/GB-month.
On-demand backups: $0.10/GB stored.
Restore: $0.15/GB.

For a 10TB table:
  PITR: $2K/mo.
  On-demand backups: depends on count.
```

### 25.6 What I'd actually do

For cost optimization on DDB:
- CloudWatch dashboard per table.
- Quarterly review of capacity vs actual usage.
- DAX for hot reads.
- TTL for ephemeral.
- S3 archival via Streams for cold.

---

## 26. Anti-Patterns — Staff-Level Red Flags

### 26.1 Scanning in hot paths

`Scan` reads every item in the table; cost grows with table. Never in production hot paths.

### 26.2 Using DynamoDB as a general DB

Joins, ad-hoc queries, aggregates → use a relational DB. DDB is for designed access patterns.

### 26.3 Designing without enumerating access patterns

Naming PK after data (`user_id`) without considering queries → can't add lookup-by-X without GSI churn.

### 26.4 ALL-projection on every GSI

Storage doubles per GSI. Use INCLUDE with minimum attrs.

### 26.5 ConsistentRead by default

2× cost. Only use when correctness requires it.

### 26.6 No TTL on time-bounded data

Manual cleanup is expensive. TTL is free.

### 26.7 Storing blobs in DDB

400 KB item limit; expensive. Use S3 with reference in DDB.

### 26.8 Not checking expires_at

Relying on TTL for security gating. App must validate.

### 26.9 Hot PK on viral content

Unsharded counters / single-PK celebrity items. Must shard.

### 26.10 Single PK="all-records"

Capacity capped at 1 partition. Always design for high-cardinality PK.

### 26.11 Ignoring throttling errors

Throttle = too much load. Investigate hot keys, not just retry.

### 26.12 Scans for "list all"

Even Query with bad PK is suspect. Design GSIs for "list" queries.

### 26.13 Cross-region read after write

GSIs and Global Tables are eventually consistent. Don't expect read-your-write across regions.

### 26.14 Forgetting Stream DLQ

Lambda processing stream errors → silent drops without DLQ.

### 26.15 Mixing capacity modes randomly

Switching back and forth costs money. Pick one based on workload shape.

### 26.16 Naming GSIs poorly

`GSI1`, `GSI2`, … is opaque; future engineers hate you. Name semantically: `email-lookup-index`, `active-jobs-index`.

### 26.17 Not testing at scale

Local DynamoDB or DDB Local doesn't replicate partition behavior. Load-test against AWS.

### 26.18 Multi-region without atomic ops

LWW silently loses data on concurrent updates. Use atomic ADD.

---

## 27. Decision Framework

### 27.1 Step 0 — Is DynamoDB the right primitive?

```
Yes if:
  - Predictable access patterns.
  - Need ms latency at scale.
  - KV / range queries on a fixed key.
  - Multi-region active-active.
  - Rich event-driven (Streams, Pipes).

No if:
  - Ad-hoc queries / SQL.
  - Joins required.
  - Aggregations / analytics.
  - Need rich text search.
```

### 27.2 Step 1 — Enumerate access patterns

Before designing keys, list:
- Get X by Y (point lookup).
- List X where Y range (range query).
- Find all X with attribute Z (filter).
- Aggregate count of X by Y.

Each pattern → design key or GSI.

### 27.3 Step 2 — Design keys

PK chosen for:
- High cardinality (avoid hot partitions).
- Common access pattern (most reads).

SK chosen for:
- Range queries.
- Hierarchical encoding.
- Sort order.

### 27.4 Step 3 — Design GSIs

For each access pattern not served by PK/SK, design GSI:
- PK and SK for that query.
- Project minimum attributes.
- Sparse if filter applies.

### 27.5 Step 4 — Plan capacity

- Provisioned vs on-demand.
- Auto-scaling bounds.
- DAX/cache for hot reads.

### 27.6 Step 5 — Plan operational lifecycle

- TTL for ephemeral.
- Streams for downstream.
- PITR for backup.
- Multi-region if needed.
- Schema evolution plan.

### 27.7 Step 6 — Plan failure modes

- Hot partition mitigation.
- Region failure (Global Tables).
- Application-level retry/backoff on throttle.
- DLQ on Stream consumers.

---

## 28. Mental Models a Staff Engineer Carries

1. **DynamoDB rewards upfront design.** Access patterns first, schema second.

2. **PK cardinality determines scalability.** Low cardinality = hot partition.

3. **GSI is not free.** Doubles write capacity. Sparse + INCLUDE minimum.

4. **Default eventual consistency for primary reads.** Strong reads are 2× cost.

5. **GSIs are always eventually consistent.** No exception.

6. **Hot keys are inevitable.** Plan for them: cache, shard, isolate.

7. **TTL is best-effort, not a security gate.** App must validate.

8. **Single-table design is for closely-related entities.** Multi-table for independent.

9. **Composite SK encodes hierarchy.** Begins-with for prefix queries.

10. **Sparse GSI for "find items where X".** Don't filter all rows.

11. **Streams + Pipes for CDC.** No app-level dual-write.

12. **DAX for transparent caching.** Redis for richer needs.

13. **Global Tables = LWW.** Use atomic ADD for counters.

14. **On-demand for unpredictable.** Provisioned for sustained.

15. **Adaptive capacity is a buffer, not a ceiling.** Plan steady-state below limits.

16. **CloudWatch Contributor Insights for hot keys.** Always enable.

17. **400 KB item limit.** Offload to S3.

18. **Scans are death at scale.** Use Query or GSI.

19. **TransactWriteItems for atomic across rows.** Up to 100 items.

20. **Conditional writes for OCC.** Atomic test-and-set.

21. **Backfilling adds capacity.** Use on-demand during backfill.

22. **Schema evolution = expand-contract.** Multi-phase, multi-deploy.

23. **PITR is cheap insurance.** Always enable.

24. **VPC Gateway Endpoint is free.** Avoid NAT Gateway egress.

25. **Multi-region replication doubles cost.** Replicate only what needs it.

26. **Boring is a feature.** A 10M-RPS DDB serving sub-ms latency that quietly works.

27. **DDB ≠ Postgres.** Stop trying to make it.

28. **The bill scales with bad design.** Hot key, missing GSI, scans, and forgotten Streams compound.

---

## 29. Closing Notes

DynamoDB is the most-misused AWS service: easy to start with, brutal to scale poorly, transformative when modeled right. Staff-level mastery is recognizing that the schema is 90% of the design and the remaining 10% is operational discipline.

The patterns repeat:
- Pick PK for cardinality.
- Pick SK for range.
- GSI for inverse access.
- Streams for downstream.
- TTL for ephemeral.
- Cache for hot reads.
- Shard for hot writes.
- TransactWriteItems for atomic.

Done well, DynamoDB serves billions of requests at single-digit ms p99 with predictable cost. Done poorly, it's a frustrating, expensive, throttling key-value store that everyone wants to migrate off.

The art is doing the design work upfront, when it's cheap.

> Companion docs:
> - `awsS3ScenariosAtScale.md` — when DDB items are too big.
> - `snsSqsEventBridgeAtScale.md` — what to do with Streams.
> - `databaseTransactionScenarios.md` — atomicity patterns including DDB.
> - `statefulSystemsAtMAANGScale.md` — broader stateful design.
> - `postgresReadOptimizationMAANGScale.md` — when DDB isn't right; Postgres at scale.