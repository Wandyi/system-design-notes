Handling Caches with Large Structures

  ---
Why Large Structures Are Problematic

Small value (< 1KB):   GET/SET is cheap — negligible serialization, network, memory cost
Large value (> 10KB):  Every operation compounds across multiple dimensions

Problem                   Effect
──────────────────────────────────────────────────────────────────────
Full invalidation         Update one field → entire object invalidated → expensive rebuild
Network amplification     Reading 2 fields from a 50-field object → 50x unnecessary data
Serialization overhead    Marshal/unmarshal 100KB JSON on every access → CPU pressure
Cache stampede            Large object expires → N goroutines all rebuild simultaneously
Memory contention         One 1MB object evicts 1000 1KB objects from LRU
Partial update cost       Read-modify-write cycle for single field → 3 round trips

The root cause: treating a large structure as a single atomic cache unit when its parts have different access patterns, update frequencies, and consumers.

  ---
Strategy 1: Decompose by Access Pattern

Split one large structure into focused sub-structures. Each part gets its own TTL, invalidation rule, and eviction priority.

// BAD: monolithic cache entry
// Any field change → full invalidation → expensive DB rebuild
type UserProfile struct {
// Identity (changes: never)
ID        string
CreatedAt time.Time

      // Display (changes: rarely)
      Name      string
      Avatar    string
      Bio       string

      // Preferences (changes: occasionally)
      Theme     string
      Language  string
      Timezone  string

      // Live stats (changes: every action)
      PostCount      int64
      FollowerCount  int64
      LastActiveAt   time.Time

      // Heavy relations (rarely needed)
      RecentOrders   []Order   // could be 50 items
      Addresses      []Address
      PaymentMethods []PaymentMethod
}

// GOOD: decomposed by axis of change

Key                         Content                  TTL     Invalidated by
──────────────────────────────────────────────────────────────────────────
user:{id}:identity          id, created_at           ∞       never
user:{id}:display           name, avatar, bio        24h     profile update
user:{id}:preferences       theme, language, tz      1h      settings update
user:{id}:stats             post_count, followers    30s     any activity
user:{id}:orders:page:{n}   paginated orders         5m      new order created
user:{id}:addresses         []Address                1h      address change

// Each part is fetched independently — only load what the caller needs
type UserCache struct {
rdb *redis.Client
}

func (c *UserCache) GetDisplay(ctx context.Context, id string) (*UserDisplay, error) {
// Header component: only needs name + avatar
return get[UserDisplay](ctx, c.rdb, fmt.Sprintf("user:%s:display", id))
}

func (c *UserCache) GetStats(ctx context.Context, id string) (*UserStats, error) {
// Activity feed: only needs counts
return get[UserStats](ctx, c.rdb, fmt.Sprintf("user:%s:stats", id))
}

// Profile page: assembles from multiple focused cache reads
// Each read is small — no 50KB monolith
func (c *UserCache) GetProfilePage(ctx context.Context, id string) (*ProfilePage, error) {
displayCh  := asyncGet[UserDisplay](ctx, c.rdb, fmt.Sprintf("user:%s:display", id))
statsCh    := asyncGet[UserStats](ctx, c.rdb, fmt.Sprintf("user:%s:stats", id))
prefsCh    := asyncGet[UserPrefs](ctx, c.rdb, fmt.Sprintf("user:%s:preferences", id))

      display := <-displayCh
      stats   := <-statsCh
      prefs   := <-prefsCh

      // Assemble from independent cached parts
      return &ProfilePage{
          Display: display.value,
          Stats:   stats.value,
          Prefs:   prefs.value,
      }, firstErr(display.err, stats.err, prefs.err)
}

  ---
Strategy 2: Redis Hash — Field-Level Operations

When decomposition isn't enough and you still have a wide struct, use Redis HASH. You can read or update individual fields without touching the rest.

// BAD: full object in a string key
// Updating one field = GET full object + deserialize + modify + serialize + SET full object
user, _ := redis.Get(ctx, "user:123").Result()
profile := deserialize(user)
profile.LastActiveAt = time.Now()
redis.Set(ctx, "user:123", serialize(profile), ttl)
// 2 round trips, full serialization both ways, entire object transferred twice

// GOOD: Redis HASH — field-level granularity
// Store each field as a hash field
redis.HSet(ctx, "user:123",
"name",           "Alice",
"email",          "alice@example.com",
"follower_count", 1042,
"last_active_at", time.Now().Unix(),
"bio",            "Engineer at...",
)

// Read only what you need — 2 fields, not 50
name, email := redis.HMGet(ctx, "user:123", "name", "email").Result()

// Update one field — no read-modify-write cycle
redis.HSet(ctx, "user:123", "last_active_at", time.Now().Unix())
// Single round trip, tiny payload, other fields untouched
redis.HIncrBy(ctx, "user:123", "follower_count", 1)
// Atomic increment — no race condition

// Clean wrapper using struct tags
type UserHash struct {
Name          string `redis:"name"`
Email         string `redis:"email"`
FollowerCount int64  `redis:"follower_count"`
LastActiveAt  int64  `redis:"last_active_at"`
Bio           string `redis:"bio"`
}

func (c *UserCache) SetUser(ctx context.Context, id string, u UserHash) error {
key := "user:" + id
return c.rdb.HSet(ctx, key, u).Err()  // go-redis marshals struct to hash fields
}

func (c *UserCache) GetUser(ctx context.Context, id string) (*UserHash, error) {
key := "user:" + id
var u UserHash
if err := c.rdb.HGetAll(ctx, key).Scan(&u); err != nil {
return nil, err
}
return &u, nil
}

func (c *UserCache) UpdateFields(ctx context.Context, id string, fields map[string]any) error {
// Update arbitrary subset of fields atomically
args := make([]any, 0, len(fields)*2)
for k, v := range fields {
args = append(args, k, v)
}
return c.rdb.HSet(ctx, "user:"+id, args...).Err()
}

  ---
Strategy 3: Projections — Cache Different Views

Different consumers need different slices of the same data. Cache a projection per consumer rather than one large object that everyone over-fetches.

// One source of truth (DB), multiple cached projections optimized per use case
//
// Use case               Cache key                   Fields
// ─────────────────────────────────────────────────────────────────────
// Comment author chip    user:{id}:chip              name, avatar_url (200B)
// Search result card     user:{id}:card              name, avatar, bio, stats (1KB)
// Full profile page      user:{id}:profile           everything except orders (5KB)
// Admin panel            user:{id}:admin             all fields (20KB)

type ProjectionCache struct {
rdb    *redis.Client
loader UserLoader
}

// Each projection is built and cached independently
func (c *ProjectionCache) GetChip(ctx context.Context, id string) (*UserChip, error) {
key := fmt.Sprintf("user:%s:chip", id)
if val, err := getJSON[UserChip](ctx, c.rdb, key); err == nil {
return val, nil
}

      // Miss: build projection from source
      user, err := c.loader.Get(ctx, id)
      if err != nil {
          return nil, err
      }

      chip := &UserChip{Name: user.Name, AvatarURL: user.AvatarURL}
      setJSON(ctx, c.rdb, key, chip, 1*time.Hour)
      return chip, nil
}

// Batch load chips efficiently — single pipeline round trip
func (c *ProjectionCache) GetChips(ctx context.Context, ids []string) (map[string]*UserChip, error) {
keys := make([]string, len(ids))
for i, id := range ids {
keys[i] = fmt.Sprintf("user:%s:chip", id)
}

      // Pipeline: all GETs in one round trip
      pipe := c.rdb.Pipeline()
      cmds := make([]*redis.StringCmd, len(keys))
      for i, key := range keys {
          cmds[i] = pipe.Get(ctx, key)
      }
      pipe.Exec(ctx)

      result := make(map[string]*UserChip)
      var misses []string

      for i, cmd := range cmds {
          var chip UserChip
          if err := json.Unmarshal([]byte(cmd.Val()), &chip); err == nil {
              result[ids[i]] = &chip
          } else {
              misses = append(misses, ids[i])
          }
      }

      // Load misses from DB in bulk
      if len(misses) > 0 {
          users, err := c.loader.GetBatch(ctx, misses)
          if err != nil {
              return nil, err
          }
          pipe := c.rdb.Pipeline()
          for _, user := range users {
              chip := &UserChip{Name: user.Name, AvatarURL: user.AvatarURL}
              result[user.ID] = chip
              data, _ := json.Marshal(chip)
              pipe.Set(ctx, fmt.Sprintf("user:%s:chip", user.ID), data, 1*time.Hour)
          }
          pipe.Exec(ctx)
      }

      return result, nil
}

  ---
Strategy 4: Singleflight — Prevent Stampede on Large Object Rebuild

Large object rebuild is expensive. When it expires, N concurrent requests all try to rebuild simultaneously. Singleflight collapses them into one.

import "golang.org/x/sync/singleflight"

type ProductCatalogCache struct {
rdb   *redis.Client
db    *sql.DB
group singleflight.Group
}

// Catalog is 200KB — expensive to build (5 DB queries, aggregation)
func (c *ProductCatalogCache) GetCatalog(ctx context.Context, categoryID string) (*Catalog, error) {
key := "catalog:" + categoryID

      // Check cache first (outside singleflight — if hit, no coordination needed)
      if cached, err := getJSON[Catalog](ctx, c.rdb, key); err == nil {
          return cached, nil
      }

      // Cache miss: use singleflight to ensure only ONE goroutine rebuilds
      // All other concurrent misses wait for this one and get the same result
      v, err, shared := c.group.Do(key, func() (any, error) {
          // Double-check: another goroutine may have populated cache while we waited
          if cached, err := getJSON[Catalog](ctx, c.rdb, key); err == nil {
              return cached, nil
          }

          catalog, err := c.buildCatalog(ctx, categoryID) // expensive: 5 queries
          if err != nil {
              return nil, err
          }

          data, _ := json.Marshal(catalog)

          // Probabilistic Early Expiration: start refreshing before TTL hits zero
          // Prevents the "thundering herd at TTL=0" problem for very hot keys
          ttl := addJitter(5*time.Minute, 30*time.Second)
          c.rdb.Set(ctx, key, data, ttl)

          return catalog, nil
      })

      if err != nil {
          return nil, err
      }
      if shared {
          slog.Debug("singleflight coalesced catalog rebuild", "category", categoryID)
      }
      return v.(*Catalog), nil
}

func addJitter(base, max time.Duration) time.Duration {
return base + time.Duration(rand.Int63n(int64(max)))
}

  ---
Strategy 5: Compression — Reduce Memory and Network Cost

Large structures are often highly compressible (JSON, repeated field names, similar values).

const compressionThreshold = 1024 // compress values larger than 1KB

type CompressedCache struct {
rdb *redis.Client
}

func (c *CompressedCache) Set(ctx context.Context, key string, value any, ttl time.Duration) error {
data, err := json.Marshal(value)
if err != nil {
return fmt.Errorf("marshal: %w", err)
}

      var payload []byte
      if len(data) >= compressionThreshold {
          // zstd: ~3x better ratio than gzip, ~5x faster decompression
          compressed, err := zstd.Compress(nil, data)
          if err != nil {
              payload = data // fall back to uncompressed
          } else {
              // Prefix with marker byte so Get() knows to decompress
              payload = append([]byte{0x01}, compressed...)
              slog.Debug("cache compression",
                  "key", key,
                  "original", len(data),
                  "compressed", len(compressed),
                  "ratio", fmt.Sprintf("%.1fx", float64(len(data))/float64(len(compressed))),
              )
          }
      } else {
          payload = append([]byte{0x00}, data...)
      }

      return c.rdb.Set(ctx, key, payload, ttl).Err()
}

func (c *CompressedCache) Get(ctx context.Context, key string, dst any) error {
payload, err := c.rdb.Get(ctx, key).Bytes()
if err != nil {
return err
}

      var data []byte
      switch payload[0] {
      case 0x01: // compressed
          data, err = zstd.Decompress(nil, payload[1:])
          if err != nil {
              return fmt.Errorf("decompress: %w", err)
          }
      case 0x00: // uncompressed
          data = payload[1:]
      default:
          return fmt.Errorf("unknown payload format: %x", payload[0])
      }

      return json.Unmarshal(data, dst)
}

Compression ratios in practice:
Large JSON user profile (50 fields, 8KB):    → zstd → ~1.2KB  (6.7x)
Product catalog (200 items, 150KB JSON):     → zstd → ~18KB   (8.3x)
Event log batch (1000 events, 500KB JSON):   → zstd → ~35KB   (14x)
Already-compressed binary (PNG, JPEG):       → zstd → ~same   (1x, skip)

  ---
Strategy 6: Tiered Caching — Hot Data in Process, Full Data in Redis

// L1: in-process (Ristretto) — nanosecond, size-bounded
// L2: Redis — millisecond, large capacity
// L3: Database — source of truth

import "github.com/dgraph-io/ristretto"

type TieredCache struct {
l1  *ristretto.Cache  // in-process, hot keys only
l2  *redis.Client     // Redis, full working set
db  *sql.DB
}

func NewTieredCache() (*TieredCache, error) {
l1, err := ristretto.NewCache(&ristretto.Config{
MaxCost:     256 << 20,  // 256MB for in-process
NumCounters: 1e7,        // tracks frequency for LFU eviction
BufferItems: 64,
Cost: func(value any) int64 {
// Cost = size in bytes — large objects "cost" more, get evicted faster
data, _ := json.Marshal(value)
return int64(len(data))
},
})
if err != nil {
return nil, err
}
return &TieredCache{l1: l1}, nil
}

func (c *TieredCache) Get(ctx context.Context, key string, dst any) error {
// L1: in-process hit — zero network, nanoseconds
if val, ok := c.l1.Get(key); ok {
reflect.ValueOf(dst).Elem().Set(reflect.ValueOf(val).Elem())
return nil
}

      // L2: Redis hit — single network hop, <1ms
      data, err := c.l2.Get(ctx, key).Bytes()
      if err == nil {
          if err := json.Unmarshal(data, dst); err == nil {
              // Promote to L1 for next access
              // L1 TTL is shorter (seconds) — L2 TTL is source of truth (minutes)
              c.l1.SetWithTTL(key, dst, 0, 10*time.Second)
              return nil
          }
      }

      // L3: DB — load, populate L2 and L1
      if err := c.loadFromDB(ctx, key, dst); err != nil {
          return err
      }

      data, _ = json.Marshal(dst)
      c.l2.Set(ctx, key, data, 5*time.Minute)
      c.l1.SetWithTTL(key, dst, 0, 10*time.Second)
      return nil
}

// L1 invalidation must be broadcast across all instances
// Use Redis pub/sub so all pods flush their local L1 copy
func (c *TieredCache) Invalidate(ctx context.Context, key string) error {
c.l1.Del(key)                         // local instance L1
c.l2.Del(ctx, key)                    // Redis L2
c.l2.Publish(ctx, "cache:invalidate", key)  // signal all other pods
return nil
}

// Each pod subscribes and flushes its L1 on broadcast
func (c *TieredCache) ListenForInvalidations(ctx context.Context) {
sub := c.l2.Subscribe(ctx, "cache:invalidate")
defer sub.Close()

      for msg := range sub.Channel() {
          c.l1.Del(msg.Payload)
      }
}

  ---
Strategy 7: Lazy / Sectioned Loading for Very Large Structures

Don't cache the whole structure at once. Cache sections as they're accessed.

// Large report: 50 sections, most users only look at 3-5
// Caching all 50 sections upfront wastes memory and is slow to build

type ReportCache struct {
rdb    *redis.Client
loader ReportLoader
group  singleflight.Group
}

// Each section cached independently — only build what's accessed
func (c *ReportCache) GetSection(ctx context.Context, reportID, section string) (*ReportSection, error) {
key := fmt.Sprintf("report:%s:section:%s", reportID, section)

      if val, err := getJSON[ReportSection](ctx, c.rdb, key); err == nil {
          return val, nil
      }

      // Singleflight per section key — not per whole report
      v, err, _ := c.group.Do(key, func() (any, error) {
          sec, err := c.loader.LoadSection(ctx, reportID, section)
          if err != nil {
              return nil, err
          }
          setJSON(ctx, c.rdb, key, sec, 15*time.Minute)
          return sec, nil
      })
      if err != nil {
          return nil, err
      }
      return v.(*ReportSection), nil
}

// Batch prefetch: if a user requests section 1, prefetch 2 and 3 in background
// Reduces latency for likely next clicks
func (c *ReportCache) PrefetchSections(ctx context.Context, reportID string, sections []string) {
for _, section := range sections {
section := section
go func() {
prefetchCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
c.GetSection(prefetchCtx, reportID, section) // result discarded — just warming cache
}()
}
}

  ---
Strategy 8: Size-Aware Eviction and Limits

Prevent large objects from crowding out many small objects.

type SizeAwareCache struct {
rdb         *redis.Client
maxValueSize int  // reject values larger than this
}

// Reject oversized values rather than storing them
func (c *SizeAwareCache) Set(ctx context.Context, key string, value any, ttl time.Duration) error {
data, err := json.Marshal(value)
if err != nil {
return err
}

      if len(data) > c.maxValueSize {
          // Log and skip — don't cache, let it go to origin every time
          slog.Warn("cache value exceeds size limit — skipping",
              "key", key,
              "size", len(data),
              "limit", c.maxValueSize,
          )
          return nil  // not an error — caller falls back to origin
      }

      return c.rdb.Set(ctx, key, data, ttl).Err()
}

// Redis-level memory policy — set in redis.conf:
// maxmemory 4gb
// maxmemory-policy allkeys-lfu   ← LFU better than LRU for large-object workloads
//                                  Large rarely-accessed objects evicted before
//                                  small frequently-accessed objects

LRU vs LFU for large structures:
LRU (Least Recently Used):
Evicts the item not accessed longest
Problem: one large object accessed once stays in cache longer
than 1000 small objects accessed slightly longer ago

LFU (Least Frequently Used):
Evicts the item accessed least often over time
Better for large structures: a rarely-used 1MB object gets evicted
before frequently-used 1KB objects regardless of recency

  ---
Strategy 9: Versioned Cache Keys — Safe Invalidation for Complex Structures

// When a large structure has complex dependencies,
// version the cache key instead of tracking individual invalidations

type VersionedCache struct {
rdb *redis.Client
}

// Version key: incremented whenever the underlying data changes
func (c *VersionedCache) GetVersion(ctx context.Context, entityID string) (int64, error) {
key := "version:" + entityID
ver, err := c.rdb.Get(ctx, key).Int64()
if errors.Is(err, redis.Nil) {
return 0, nil
}
return ver, err
}

func (c *VersionedCache) BumpVersion(ctx context.Context, entityID string) (int64, error) {
return c.rdb.Incr(ctx, "version:"+entityID).Result()
}

// Cache key includes the version — stale keys naturally expire, no explicit delete needed
func (c *VersionedCache) GetCatalog(ctx context.Context, categoryID string) (*Catalog, error) {
ver, err := c.GetVersion(ctx, "catalog:"+categoryID)
if err != nil {
return nil, err
}

      // Key includes version — old version keys still exist but aren't accessed
      // They expire naturally via TTL — no thundering herd from mass invalidation
      key := fmt.Sprintf("catalog:%s:v%d", categoryID, ver)

      if val, err := getJSON[Catalog](ctx, c.rdb, key); err == nil {
          return val, nil
      }

      catalog, err := c.buildCatalog(ctx, categoryID)
      if err != nil {
          return nil, err
      }
      setJSON(ctx, c.rdb, key, catalog, 1*time.Hour)
      return catalog, nil
}

// On any change: bump version — old cached object becomes orphaned and expires
func (c *VersionedCache) OnProductUpdated(ctx context.Context, categoryID string) {
c.BumpVersion(ctx, "catalog:"+categoryID)
// No DELETE needed — next request uses new version key, builds fresh
// Old version key expires via TTL quietly
}

  ---
Summary: Pattern Selection

Situation                              Strategy
─────────────────────────────────────────────────────────────────────────
Object has fields with different       Decompose by axis of change
update frequencies                   Different key, TTL, and invalidation per part

Need to update individual fields       Redis HASH (HSET/HGET/HMGET/HINCRBY)
without full read-modify-write       One round trip per field, no race condition

Different consumers need different     Projections — one cache key per consumer view
slices of the same object            Each projection optimised for its use case

Expensive rebuild, high concurrency    Singleflight collapses concurrent misses into one
on cache miss                        Probabilistic early expiration prevents TTL cliff

Object > 1KB                           Compress with zstd before storing
Typical ratio: 4-10x for JSON

Hot keys accessed nanosecond-speed     Tiered: in-process L1 (seconds TTL, size-bounded)
+ Redis L2 (minutes TTL, full working set)

Very large structure (50+ sections)    Lazy section loading
most users access < 20%              Cache sections on access + background prefetch

Large objects crowding small objects   Size-aware eviction + maxmemory-policy allkeys-lfu
Reject oversized values above threshold

Complex dependency graph for           Versioned keys — bump version on any change
invalidation                         Stale keys orphaned and expire naturally

The core principle: a cache entry should represent the smallest independently invalidatable unit of data. When a cache entry is too large, it means you've bundled together things that change for different
reasons — and paying for full invalidation every time any part changes.
