# 19 · SQL and Coding Problems

ClickHouse coding rounds usually mix **SQL puzzles** (write the query, optimize it) with **client-side coding** (a Go/Python ingest pipeline, a sharding router, a connection pool). The follow-ups are where staff signal lives.

## SQL puzzles

### S1 · Active users per day (DAU) and retention

Given an `events(ts, user_id)` table:

```sql
-- DAU
SELECT toDate(ts) AS day, uniq(user_id) AS dau
FROM events
WHERE ts >= today() - 30
GROUP BY day
ORDER BY day;

-- D7 retention (returned on day 7 after first seen)
SELECT count() AS retained
FROM (
    SELECT user_id, min(toDate(ts)) AS first_day
    FROM events
    WHERE ts >= today() - 30
    GROUP BY user_id
) f
WHERE EXISTS (
    SELECT 1 FROM events e
    WHERE e.user_id = f.user_id
      AND toDate(e.ts) = f.first_day + INTERVAL 7 DAY
);
```

**Follow-up**: optimize for the case where the table has 10B rows and we run this hourly.
- Build a MV `daily_user(day, user_id)` = `groupBitmapState(user_id) by day`.
- DAU = `bitmapCardinality(groupBitmapMergeState(...))`.
- Retention = `bitmapAnd(day_n, day_0)` cardinality.

### S2 · Funnel — view → cart → buy with 1 hour window

```sql
SELECT level, count() AS users
FROM (
    SELECT user_id, windowFunnel(3600)(
        ts,
        event = 'view',
        event = 'cart',
        event = 'buy'
    ) AS level
    FROM events
    WHERE event IN ('view','cart','buy') AND ts >= today() - 7
    GROUP BY user_id
)
GROUP BY level;
```

**Follow-up**: order independence (any order). Use `windowFunnel(3600, 'strict_order')` (strict) or omit (default). For unordered, use `bitmapAnd` of per-event user sets.

### S3 · Top-10 referrers per day, with rank

```sql
SELECT day, referrer, c, row_number() OVER (PARTITION BY day ORDER BY c DESC) AS rk
FROM (
    SELECT toDate(ts) AS day, referrer, count() AS c
    FROM events GROUP BY day, referrer
)
WHERE rk <= 10
ORDER BY day, rk;
```

**Follow-up**: same with `LIMIT 10 BY day` (more idiomatic + faster for small N):

```sql
SELECT toDate(ts) AS day, referrer, count() AS c
FROM events GROUP BY day, referrer
ORDER BY day, c DESC LIMIT 10 BY day;
```

### S4 · Sessionize events with > 30 min gap

```sql
SELECT
    user_id,
    arrayCumSum(
        arrayMap(i ->
            if(i = 0, 1, (ts_arr[i + 1] - ts_arr[i]) > 1800),
            arrayEnumerate(ts_arr)
        )
    ) AS session_idx,
    ts_arr
FROM (
    SELECT user_id, groupArray(ts) AS ts_arr
    FROM (SELECT user_id, ts FROM events ORDER BY user_id, ts)
    GROUP BY user_id
);
```

**Follow-up**: produce one row per session.

```sql
SELECT user_id, session_idx, min(ts) AS started, max(ts) AS ended, count() AS events
FROM (... above query, exploded with ARRAY JOIN ...)
GROUP BY user_id, session_idx;
```

Or sessionize at ingest time and avoid the SQL gymnastics.

### S5 · Pivot status counts by day

```sql
SELECT toDate(ts) AS day,
    countIf(status = 'ok')     AS ok,
    countIf(status = 'warn')   AS warn,
    countIf(status = 'error')  AS error
FROM events
GROUP BY day
ORDER BY day;
```

### S6 · Gap-fill missing minutes with zeros

```sql
SELECT minute, ifNull(c, 0) AS cnt
FROM (
    SELECT toStartOfMinute(ts) AS minute, count() AS c
    FROM events
    WHERE ts BETWEEN now() - INTERVAL 1 HOUR AND now()
    GROUP BY minute
)
ORDER BY minute
WITH FILL FROM toStartOfMinute(now() - INTERVAL 1 HOUR) TO toStartOfMinute(now()) STEP INTERVAL 1 MINUTE;
```

### S7 · Find users who did A but not B

```sql
SELECT a.user_id
FROM (
    SELECT user_id FROM events WHERE event = 'A' AND ts >= today() - 30 GROUP BY user_id
) a
LEFT ANTI JOIN (
    SELECT user_id FROM events WHERE event = 'B' AND ts >= today() - 30 GROUP BY user_id
) b ON a.user_id = b.user_id;
```

Alternative with bitmaps:
```sql
SELECT bitmapAndnot(
    groupBitmapStateIf(user_id, event = 'A'),
    groupBitmapStateIf(user_id, event = 'B')
) AS only_a
FROM events WHERE ts >= today() - 30;
```

### S8 · Optimize this slow query

Given:
```sql
SELECT user_id, count() FROM events FINAL WHERE ts > now() - INTERVAL 1 DAY GROUP BY user_id;
```

Issues: FINAL is slow.

Rewrite:
```sql
SELECT user_id, count()
FROM events
WHERE ts > now() - INTERVAL 1 DAY
GROUP BY user_id;
```

If dedup matters:
```sql
SELECT user_id, count()
FROM (
    SELECT user_id, ts FROM events WHERE ts > now() - INTERVAL 1 DAY
    LIMIT 1 BY user_id, ts
) GROUP BY user_id;
```

Or maintain an AggregatingMergeTree of `(day, user_id) -> count`.

### S9 · Choose the right ORDER BY for these queries

Queries:
1. `WHERE tenant_id = ? AND ts BETWEEN ?`
2. `WHERE tenant_id = ? AND user_id = ?`
3. `WHERE event_type = ? AND tenant_id = ?`

Discussion: `ORDER BY (tenant_id, event_type, ts)` works for 1 & 3 well; for 2 add bloom filter / projection on `user_id`. If 2 is hot, projection sorted by `(tenant_id, user_id)`.

### S10 · Write a query to identify too-many-parts risk

```sql
SELECT database, table, partition,
       count() AS parts, sum(rows) AS rows, formatReadableSize(sum(bytes_on_disk)) AS bytes
FROM system.parts
WHERE active AND database = 'default'
GROUP BY database, table, partition
HAVING parts > 100
ORDER BY parts DESC;
```

## Coding problems (Go-flavored, since repo defaults to Go)

### C1 · Batched ingest client

Write a Go function that accepts events and batches them into ClickHouse INSERTs.

```go
package chingest

import (
    "context"
    "database/sql"
    "sync"
    "time"

    _ "github.com/ClickHouse/clickhouse-go/v2"
)

type Event struct {
    TS     time.Time
    UserID uint64
    Event  string
    Bytes  uint32
}

type Ingester struct {
    db            *sql.DB
    table         string
    flushEvery    time.Duration
    flushAt       int
    mu            sync.Mutex
    buf           []Event
    stop          chan struct{}
}

func New(db *sql.DB, table string, every time.Duration, at int) *Ingester {
    ing := &Ingester{
        db: db, table: table, flushEvery: every, flushAt: at,
        buf: make([]Event, 0, at),
        stop: make(chan struct{}),
    }
    go ing.run()
    return ing
}

func (i *Ingester) Add(ev Event) {
    i.mu.Lock()
    i.buf = append(i.buf, ev)
    overflow := len(i.buf) >= i.flushAt
    i.mu.Unlock()
    if overflow {
        i.Flush(context.Background())
    }
}

func (i *Ingester) Flush(ctx context.Context) error {
    i.mu.Lock()
    batch := i.buf
    i.buf = make([]Event, 0, i.flushAt)
    i.mu.Unlock()
    if len(batch) == 0 {
        return nil
    }
    tx, err := i.db.Begin()
    if err != nil { return err }
    stmt, err := tx.Prepare("INSERT INTO " + i.table + " (ts, user_id, event, bytes) VALUES (?, ?, ?, ?)")
    if err != nil { return err }
    for _, e := range batch {
        if _, err := stmt.Exec(e.TS, e.UserID, e.Event, e.Bytes); err != nil {
            return err
        }
    }
    return tx.Commit()
}

func (i *Ingester) run() {
    t := time.NewTicker(i.flushEvery)
    defer t.Stop()
    for {
        select {
        case <-t.C:
            _ = i.Flush(context.Background())
        case <-i.stop:
            _ = i.Flush(context.Background())
            return
        }
    }
}

func (i *Ingester) Close() { close(i.stop) }
```

**Follow-ups**:
- Retry with exponential backoff on failure.
- Idempotency: ClickHouse ReplicatedMergeTree dedups identical blocks; the client can also include a `_partition_key` hash.
- Backpressure: bound the buffer and have `Add` block (or drop) when full.
- Async insert: switch to `SETTINGS async_insert = 1, wait_for_async_insert = 0` and let the server batch.
- For high QPS: use `clickhouse-go` v2's native `Batch()` API, which is much faster than `database/sql`.

### C2 · Sharding router

Compute the shard for a `user_id` given N shards.

```go
package shard

import "github.com/cespare/xxhash/v2"

func Pick(userID uint64, shards int) int {
    h := xxhash.Sum64(uint64Bytes(userID))
    return int(h % uint64(shards))
}

func uint64Bytes(u uint64) []byte {
    b := make([]byte, 8)
    for i := 0; i < 8; i++ { b[i] = byte(u >> (i * 8)) }
    return b
}
```

**Follow-ups**:
- Use the same hash as ClickHouse's `cityHash64` if you want client-side hashing to match the server's Distributed routing.
- Consistent hashing for re-sharding: implement Jump Consistent Hash to minimize re-bucketing when shard count changes.

### C3 · Connection pool with hedged requests

```go
package chpool

import (
    "context"
    "errors"
    "sync"
)

type Replica struct {
    name string
    do   func(ctx context.Context, q string) ([]byte, error)
}

type Pool struct {
    replicas []Replica
    mu       sync.Mutex
    nextIdx  int
}

func (p *Pool) Query(ctx context.Context, q string, hedgeAfter time.Duration) ([]byte, error) {
    p.mu.Lock()
    a := p.replicas[p.nextIdx%len(p.replicas)]
    b := p.replicas[(p.nextIdx+1)%len(p.replicas)]
    p.nextIdx++
    p.mu.Unlock()

    type result struct { data []byte; err error }
    ch := make(chan result, 2)
    go func() { d, e := a.do(ctx, q); ch <- result{d, e} }()
    select {
    case <-time.After(hedgeAfter):
        go func() { d, e := b.do(ctx, q); ch <- result{d, e} }()
    case r := <-ch:
        return r.data, r.err
    }
    r := <-ch
    return r.data, r.err
}
```

**Follow-ups**:
- Cancel the loser via context.
- Cap concurrent hedges to avoid amplification under load.
- Health checks to demote slow replicas.

### C4 · Backpressure-aware Kafka-to-ClickHouse consumer

Pseudocode:

```go
for batch := range kafkaConsumer.Stream {
    for !ch.HealthyForInsert() {           // checks "Too many parts" via system table
        time.Sleep(100 * time.Millisecond)
    }
    if err := ch.InsertBatch(batch); err != nil {
        if isTooManyParts(err) {
            backoff *= 2
            continue
        }
        // retry or DLQ
    }
    kafkaConsumer.Commit(batch.Offsets)
}
```

**Follow-ups**:
- Idempotency by reusing partition+offset as block-dedup hint.
- Schema drift handling — auto-detect missing column, ALTER (carefully).

### C5 · LRU cache for hot dashboard queries

(Already covered in earlier packs; reuse the LRU + TTL pattern for caching query results client-side.)

## Patterns to internalize

- For SQL: know the idiomatic CH form (`LIMIT BY`, `argMax`, `windowFunnel`, `WITH FILL`, bitmap states).
- For coding: batch + async + idempotency + backpressure + hedged reads.
- Always discuss the failure modes and the metric you'd monitor.

---

## Sources

- [clickhouse-go driver](https://github.com/ClickHouse/clickhouse-go)
- [Aggregation function reference](https://clickhouse.com/docs/sql-reference/aggregate-functions/reference)
- [Bitmap functions](https://clickhouse.com/docs/sql-reference/functions/bitmap-functions)
- [Window functions](https://clickhouse.com/docs/sql-reference/window-functions)
