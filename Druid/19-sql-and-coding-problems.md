# 19 · SQL and Coding Problems

Druid coding rounds combine **Druid SQL puzzles** (write/optimize the query, predict the native query type) with **client-side coding** (a Java/Go ingest client, a supervisor manager, a query gateway).

## SQL puzzles

### S1 · Top 10 events per minute per country

```sql
SELECT TIME_FLOOR(__time, 'PT1M') AS minute, country, event_name, count(*) AS cnt
FROM events
WHERE __time >= CURRENT_TIMESTAMP - INTERVAL '1' HOUR
GROUP BY 1, 2, 3
ORDER BY 1, 2, cnt DESC
LIMIT 10 BY (1, 2);   -- Druid doesn't support LIMIT BY; use window:
```

Fix via window:

```sql
SELECT minute, country, event_name, cnt FROM (
  SELECT minute, country, event_name, cnt,
         ROW_NUMBER() OVER (PARTITION BY minute, country ORDER BY cnt DESC) AS rn
  FROM (
    SELECT TIME_FLOOR(__time, 'PT1M') AS minute, country, event_name, count(*) AS cnt
    FROM events
    WHERE __time >= CURRENT_TIMESTAMP - INTERVAL '1' HOUR
    GROUP BY 1, 2, 3
  )
)
WHERE rn <= 10;
```

**Follow-up**: at 10× scale, the outer window function materializes on Broker. Solution: pre-aggregate into a datasource that's already grouped by (minute, country, event_name); query is then just a GroupBy with bounded result size.

### S2 · DAU and retention via theta sketches

```sql
WITH daily AS (
  SELECT TIME_FLOOR(__time, 'P1D') AS day,
         DS_THETA(user_id, 16384) AS sk
  FROM events
  WHERE __time >= CURRENT_TIMESTAMP - INTERVAL '30' DAY
  GROUP BY 1
)
SELECT a.day,
       THETA_SKETCH_ESTIMATE(a.sk) AS dau,
       THETA_SKETCH_ESTIMATE(THETA_SKETCH_INTERSECT(a.sk, b.sk)) AS retained_d1
FROM daily a LEFT JOIN daily b ON b.day = a.day + INTERVAL '1' DAY;
```

For pre-aggregated data, store theta sketch as a metric.

### S3 · Funnel — view → cart → buy (order-independent)

(See [14.4 Approach A](14-query-patterns-and-corner-cases.md#144-pattern-funnel).)

### S4 · Time bucketing with gap fill

Druid SQL doesn't have `WITH FILL`. Approach: union with a generated series, or post-process in BI tool.

```sql
SELECT ts, COALESCE(cnt, 0) AS cnt
FROM (
  SELECT TIME_FLOOR(__time, 'PT1M') AS minute, count(*) AS cnt
  FROM events
  WHERE __time BETWEEN ts1 AND ts2
  GROUP BY 1
) data
RIGHT JOIN (
  -- generated minutes from ts1..ts2; pass in as inline subquery from app
  SELECT minute FROM TABLE(EXTERN(...))
) gen ON data.minute = gen.minute;
```

### S5 · Detect over-large segments

```sql
SELECT datasource, interval, count(*) AS segs, sum(size)/1e9 AS gb
FROM sys.segments
WHERE is_published = 1 AND size > 1.5 * 1024 * 1024 * 1024
GROUP BY 1, 2 ORDER BY 4 DESC LIMIT 50;
```

### S6 · Optimize a slow query

Given:
```sql
SELECT user_id, COUNT(*) FROM events
WHERE __time >= CURRENT_TIMESTAMP - INTERVAL '1' DAY
GROUP BY user_id;
```

Problems:
- `user_id` likely high-cardinality → GroupBy hash table is huge.
- Returns potentially millions of rows.

Rewrites:
- Filter to a subset of users via `WHERE user_id IN (...)`.
- Or rephrase: do we really want per-user counts, or top-N users?
- Or pre-aggregate: maintain a `user_daily` datasource with `(day, user_id, count)`.

### S7 · Cohort intersection — A but not B

(See [14.7](14-query-patterns-and-corner-cases.md#147-pattern-set-algebra--a-but-not-b).)

### S8 · Detect a sudden ingest drop

```sql
SELECT TIME_FLOOR(__time, 'PT1M') AS m, count(*) AS cnt
FROM events
WHERE __time >= CURRENT_TIMESTAMP - INTERVAL '1' HOUR
GROUP BY 1 ORDER BY 1;
```

Visualize; alert if any minute is < N% of trailing average.

### S9 · Find under-replicated segments

```sql
SELECT datasource, count(*) AS under_repl
FROM sys.segments
WHERE is_published = 1 AND replication_factor < 2
GROUP BY 1 ORDER BY 2 DESC;
```

### S10 · Inspect a single task's lifecycle

```sql
SELECT task_id, type, status, duration, error_msg
FROM sys.tasks
WHERE task_id = 'index_kafka_events_abc123';
```

## Coding problems

### C1 · Java ingestion client (the standard SDK shape)

```java
public class DruidIngester {
    private final OkHttpClient http;
    private final String overlordUrl;

    public String submitSupervisor(String supervisorJson) throws IOException {
        Request req = new Request.Builder()
            .url(overlordUrl + "/druid/indexer/v1/supervisor")
            .post(RequestBody.create(supervisorJson, MediaType.parse("application/json")))
            .build();
        try (Response resp = http.newCall(req).execute()) {
            return resp.body().string();
        }
    }

    public TaskStatus getStatus(String taskId) throws IOException {
        Request req = new Request.Builder()
            .url(overlordUrl + "/druid/indexer/v1/task/" + taskId + "/status")
            .get().build();
        try (Response resp = http.newCall(req).execute()) {
            return mapper.readValue(resp.body().bytes(), TaskStatus.class);
        }
    }
}
```

**Follow-ups**:
- Retry with backoff.
- Auth (basic / OIDC).
- Long-poll for task completion.
- Wrap as a Spring/Micronaut/Quarkus service.

### C2 · Go ingestion client via the Druid HTTP API (POSTing events to a Tranquility/HTTP endpoint)

Druid no longer has Tranquility; modern ingest is Kafka or batch. But you may still need a client to push to a Kafka producer:

```go
package druidpub

import (
    "encoding/json"
    "github.com/segmentio/kafka-go"
)

type Publisher struct {
    w *kafka.Writer
}

func New(brokers []string, topic string) *Publisher {
    return &Publisher{w: &kafka.Writer{
        Addr:     kafka.TCP(brokers...),
        Topic:    topic,
        Balancer: &kafka.Hash{},
        BatchSize: 1000,
        BatchTimeout: 100 * time.Millisecond,
    }}
}

func (p *Publisher) Publish(ctx context.Context, key string, event any) error {
    body, err := json.Marshal(event)
    if err != nil { return err }
    return p.w.WriteMessages(ctx, kafka.Message{
        Key: []byte(key),
        Value: body,
    })
}
```

**Follow-ups**:
- Batching for throughput.
- Idempotency via Kafka producer.
- Schema validation before publish.

### C3 · Druid SQL query client (Java)

```java
public List<Map<String, Object>> query(String sql) throws IOException {
    String body = mapper.writeValueAsString(Map.of(
        "query", sql,
        "resultFormat", "object",
        "context", Map.of("sqlQueryId", UUID.randomUUID().toString(), "timeout", 30_000)
    ));
    Request req = new Request.Builder()
        .url(brokerUrl + "/druid/v2/sql")
        .post(RequestBody.create(body, MediaType.parse("application/json")))
        .build();
    try (Response resp = http.newCall(req).execute()) {
        return mapper.readValue(resp.body().bytes(), new TypeReference<>(){});
    }
}
```

**Follow-ups**:
- Streaming response (`resultFormat=arrayLines` for line-delimited JSON).
- Connection pooling.
- Async with CompletableFuture.

### C4 · Supervisor health checker (Go)

```go
type SupervisorHealth struct {
    SupervisorID string
    Healthy      bool
    State        string
    LagSeconds   int64
}

func CheckAll(ctx context.Context, overlordUrl string) ([]SupervisorHealth, error) {
    resp, err := http.Get(overlordUrl + "/druid/indexer/v1/supervisor?full")
    if err != nil { return nil, err }
    defer resp.Body.Close()
    var data []struct {
        ID    string `json:"id"`
        Spec  struct {
            DataSchema struct{ DataSource string }
        }
        State string `json:"state"`
    }
    if err := json.NewDecoder(resp.Body).Decode(&data); err != nil {
        return nil, err
    }
    var out []SupervisorHealth
    for _, s := range data {
        out = append(out, SupervisorHealth{
            SupervisorID: s.ID,
            Healthy: s.State == "RUNNING",
            State: s.State,
        })
    }
    return out, nil
}
```

**Follow-ups**:
- Per-supervisor lag from Kafka offsets vs latest topic offset.
- Auto-restart on UNHEALTHY.
- Page on lag > threshold.

### C5 · Implement a query result cache (sketch)

```go
type CacheKey struct {
    SQL          string
    TimeBucketHash uint64
}

type cacheVal struct {
    data    []byte
    expires time.Time
}

type Cache struct {
    mu sync.RWMutex
    m  map[CacheKey]cacheVal
}

func (c *Cache) Query(ctx context.Context, sql string, ttl time.Duration, fetch func() ([]byte, error)) ([]byte, error) {
    key := CacheKey{SQL: sql, TimeBucketHash: timeBucket(ttl)}
    c.mu.RLock()
    v, ok := c.m[key]
    c.mu.RUnlock()
    if ok && time.Now().Before(v.expires) {
        return v.data, nil
    }
    data, err := fetch()
    if err != nil { return nil, err }
    c.mu.Lock()
    c.m[key] = cacheVal{data: data, expires: time.Now().Add(ttl)}
    c.mu.Unlock()
    return data, nil
}
```

**Follow-ups**:
- Single-flight to deduplicate concurrent identical queries.
- TTL aligned to data freshness (5s for tail, 60s for hour-aggregates).
- Eviction on memory pressure.

## Patterns to internalize

- SQL: know the idiomatic Druid form (TIME_FLOOR, APPROX_COUNT_DISTINCT_DS_HLL, FILTER, ROW_NUMBER, theta sketch set algebra).
- Always include time filter.
- EXPLAIN PLAN to verify native query type.
- Client code: idempotent ingest, batched publish, cached queries, healthy supervisor checks.

---

## Sources

- [Druid SQL functions](https://druid.apache.org/docs/latest/querying/sql-functions/)
- [Druid SQL JSON functions](https://druid.apache.org/docs/latest/querying/sql-json-functions/)
- [HTTP API reference](https://druid.apache.org/docs/latest/api-reference/sql-api/)
- [Overlord API](https://druid.apache.org/docs/latest/api-reference/tasks-api/)
- [Coordinator API](https://druid.apache.org/docs/latest/api-reference/coordinator-api/)
