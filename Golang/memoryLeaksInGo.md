Memory Leak Scenarios with Context in Go

  ---
1. Not Calling the Cancel Function — The Most Common Leak

Every WithCancel, WithTimeout, and WithDeadline allocates:
- An entry in the parent's child list (so parent cancellation propagates)
- A timer goroutine (for timeout/deadline variants)

Neither is freed until cancel() is called.

// LEAK: cancel function discarded with blank identifier
func (h *Handler) GetUser(w http.ResponseWriter, r *http.Request) {
ctx, _ := context.WithTimeout(r.Context(), 5*time.Second)
// The timer goroutine runs for 5 seconds even if this function returns in 1ms
// r.Context() holds a reference to ctx until the timer fires
user, err := h.svc.Get(ctx, id)
// ...
}

// LEAK: cancel declared but never called on all paths
func process(ctx context.Context) error {
ctx, cancel := context.WithCancel(ctx)

      if err := validate(); err != nil {
          return err  // cancel() never called — parent holds child reference forever
      }
      defer cancel()  // only reached if validate() passes
      // ...
}

// FIX: always defer cancel immediately after creation
func (h *Handler) GetUser(w http.ResponseWriter, r *http.Request) {
ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
defer cancel()  // always fires, regardless of return path
// ...
}

What leaks internally:

parent context children list:
[child1, child2, child3 ← never removed, holds reference to ctx]

runtime timer heap:
[timer → fires cancel in 5s, but ctx still referenced until then]

In a high-traffic service with thousands of requests/second, each missing cancel() means thousands of dangling child entries accumulating in the parent context's child list.

  ---
2. Goroutine Leak — Not Listening to ctx.Done()

A goroutine that ignores context cancellation keeps running after the work is no longer needed. The goroutine itself, its stack, and everything it references are all leaked.

// LEAK: goroutine runs forever regardless of context
func (s *Service) StartPoller(ctx context.Context) {
go func() {
for {
data, err := s.fetchLatest()  // blocking call, no context
if err != nil {
log.Println(err)
continue
}
s.process(data)
time.Sleep(5 * time.Second)  // Sleep ignores ctx cancellation
}
// This goroutine NEVER exits
}()
}

// Every call to StartPoller adds another immortal goroutine

// LEAK: goroutine blocked on channel send with no way out
func fanOut(ctx context.Context, input <-chan Job) {
for job := range input {
result := process(job)
go func() {
resultChan <- result  // blocks if resultChan is full, forever
// ctx cancellation has no effect — goroutine is stuck on channel op
}()
}
}

// FIX: every goroutine must have an exit path tied to context
func (s *Service) StartPoller(ctx context.Context) {
go func() {
ticker := time.NewTicker(5 * time.Second)
defer ticker.Stop()

          for {
              select {
              case <-ctx.Done():
                  return  // clean exit
              case <-ticker.C:
                  data, err := s.fetchLatest(ctx)  // context-aware
                  if err != nil {
                      if errors.Is(err, context.Canceled) {
                          return
                      }
                      log.Println(err)
                      continue
                  }
                  s.process(data)
              }
          }
      }()
}

// FIX: channel send with ctx escape hatch
go func() {
select {
case <-ctx.Done():
return
case resultChan <- result:
}
}()

  ---
3. Goroutine Leak via errgroup Without Draining Channels

// LEAK: errgroup workers exit but channel producer keeps sending
// nobody reads from jobs anymore → producer goroutine blocked forever
func process(ctx context.Context, items []Item) error {
jobs := make(chan Item)  // unbuffered

      g, ctx := errgroup.WithContext(ctx)

      // producer
      g.Go(func() error {
          defer close(jobs)
          for _, item := range items {
              select {
              case <-ctx.Done():
                  return ctx.Err()
              case jobs <- item:
              }
          }
          return nil
      })

      // consumer — returns early on first error, stops reading jobs
      g.Go(func() error {
          for job := range jobs {  // if this returns early, producer blocks on jobs <-
              if err := handle(ctx, job); err != nil {
                  return err  // ctx cancelled, but producer may be mid-send
              }
          }
          return nil
      })

      return g.Wait()
}

// FIX: producer always selects on ctx.Done() so it exits when ctx is cancelled
// The errgroup.WithContext ensures ctx is cancelled when consumer returns error
// Producer's select on ctx.Done() unblocks it → both goroutines exit cleanly
g.Go(func() error {
defer close(jobs)
for _, item := range items {
select {
case <-ctx.Done():
return ctx.Err()  // unblocks even if consumer stopped reading
case jobs <- item:
}
}
return nil
})

  ---
4. Context Values Holding Large Objects

Context values are held by reference. If a context lives longer than expected (passed to a long-running goroutine, stored somewhere), everything the value references is kept alive.

// LEAK: large result set attached to context lives for the entire request
// including all downstream goroutines spawned from this context
func (h *Handler) Search(w http.ResponseWriter, r *http.Request) {
results, _ := h.db.Query(r.Context(), query)  // results = 50MB of data

      // Attaching 50MB to context — every goroutine derived from this ctx
      // holds a reference to the entire result set
      ctx := context.WithValue(r.Context(), ctxKeyResults, results)

      go func() {
          h.audit(ctx, "search performed")  // audit only needs trace ID
          // but holds reference to 50MB results via ctx for its entire lifetime
      }()
}

// FIX: pass only what each goroutine needs, not the whole context
func (h *Handler) Search(w http.ResponseWriter, r *http.Request) {
results, _ := h.db.Query(r.Context(), query)
traceID, _ := TraceIDFromContext(r.Context())

      go func() {
          // Pass only the trace ID, not the context carrying 50MB
          h.audit(context.Background(), traceID, "search performed")
      }()

      // render results, then let them go out of scope
      render(w, results)
}

  ---
5. Unbounded WithValue Chain Growth

Each WithValue call wraps the previous context in a new layer. In a loop or recursive path, this creates a deeply nested chain that is never garbage collected while the outermost context lives.

// LEAK: context grows with every iteration
func (p *Pipeline) Run(ctx context.Context, stages []Stage) error {
for i, stage := range stages {
// Each iteration wraps the previous context
ctx = context.WithValue(ctx, fmt.Sprintf("stage_%d", i), stage.Meta)
// After 10000 stages: 10000-deep context chain
// All previous stage metadata kept alive
if err := stage.Execute(ctx); err != nil {
return err
}
}
return nil
}

// FIX: collect values into one struct, set once
type PipelineMeta struct {
Stages []StageMeta
}

func (p *Pipeline) Run(ctx context.Context, stages []Stage) error {
meta := PipelineMeta{}
for i, stage := range stages {
meta.Stages = append(meta.Stages, stage.Meta)
if err := stage.Execute(ctx); err != nil {
return err
}
}
// Set once at the end if downstream needs it, or don't use ctx at all
return nil
}

  ---
6. HTTP Client Requests Without Context

// LEAK: in-flight HTTP request cannot be cancelled
// If caller's context is cancelled, this goroutine and the TCP connection
// are held open until the remote server responds (could be minutes)
func (c *Client) Fetch(ctx context.Context, url string) ([]byte, error) {
req, _ := http.NewRequest(http.MethodGet, url, nil)
// ctx is NOT attached — connection held open even if our context is cancelled

      resp, err := c.http.Do(req)
      // ...
}

// In a service handling 1000 req/s with 500ms timeouts:
// Each cancelled request leaves a dangling goroutine + TCP connection
// → connection pool exhaustion → all requests start failing

// FIX: always use NewRequestWithContext
func (c *Client) Fetch(ctx context.Context, url string) ([]byte, error) {
req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
if err != nil {
return nil, fmt.Errorf("build request: %w", err)
}
resp, err := c.http.Do(req)
// ctx cancellation aborts the request, releases connection back to pool
// ...
}

  ---
7. Context Derived from Background in a Goroutine — Severs Lifecycle

Not a memory leak in the classic sense, but creates goroutines that outlive their intended scope — a logical lifecycle leak.

// LEAK: goroutine's context is disconnected from caller's lifecycle

```
        func (s *Service) HandleRequest(ctx context.Context, req Request) error {
        go func() {
        // context.Background() never cancels — this goroutine runs forever
        // even if the original request was cancelled, timed out, or server is shutting down
        if err := s.audit(context.Background(), req); err != nil {
        log.Println(err)
        }
        }()
        return s.process(ctx, req)
        }
```

// FIX: use a separate long-lived context for background work,
// not Background() inline, and not the request context
```
        type Service struct {
        bgCtx context.Context  // set at service construction, cancelled on shutdown
        }
        
        func (s *Service) HandleRequest(ctx context.Context, req Request) error {
        go func() {
        // Tied to service lifecycle, not request lifecycle
        // Cancelled when the service shuts down
        auditCtx, cancel := context.WithTimeout(s.bgCtx, 10*time.Second)
        defer cancel()
        if err := s.audit(auditCtx, req); err != nil {
        log.Println(err)
        }
        }()
        return s.process(ctx, req)
        }

```

  ---
8. Ticker / Timer Goroutine Leaks Inside Context-Aware Functions

// LEAK: ticker never stopped if function returns early
func watchdog(ctx context.Context, threshold time.Duration) {
go func() {
ticker := time.NewTicker(threshold)
// defer ticker.Stop() missing — ticker goroutine leaks on early return
for {
select {
case <-ctx.Done():
return
case t := <-ticker.C:
checkHealth(t)
}
}
}()
}

// LEAK: **time.After** in a loop creates a new timer every iteration
// old timers not GC'd until they fire
func poll(ctx context.Context) {
for {
select {
case <-ctx.Done():
return
case <-time.After(5 * time.Second):  // new timer object every iteration
doWork(ctx)
}
}
}

// FIX: always Stop() tickers and reuse them
func watchdog(ctx context.Context, threshold time.Duration) {
go func() {
ticker := time.NewTicker(threshold)
defer ticker.Stop()  // crucial — releases the ticker goroutine and channel
for {
select {
case <-ctx.Done():
return
case t := <-ticker.C:
checkHealth(t)
}
}
}()
}

// FIX: use ticker outside the loop
func poll(ctx context.Context) {
ticker := time.NewTicker(5 * time.Second)
defer ticker.Stop()
for {
select {
case <-ctx.Done():
return
case <-ticker.C:
doWork(ctx)
}
}
}

  ---
9. Database Transaction Held by Uncancelled Context

// LEAK: transaction open indefinitely if ctx is never cancelled
// and commit/rollback is never called
func (r *Repo) Process(ctx context.Context) error {
tx, err := r.db.BeginTx(ctx, nil)
if err != nil {
return err
}
// No defer tx.Rollback()
// If the function panics or returns early without commit/rollback:
// - DB connection held open
// - Row locks held
// - ctx keeps the transaction alive
if err := doWork(ctx, tx); err != nil {
return err  // tx never rolled back — connection leaked to pool
}
return tx.Commit()
}

// FIX: always defer rollback — it's a no-op after successful commit
func (r *Repo) Process(ctx context.Context) error {
tx, err := r.db.BeginTx(ctx, nil)
if err != nil {
return err
}
defer tx.Rollback()  // safe to call after Commit — returns ErrTxDone, ignored

      if err := doWork(ctx, tx); err != nil {
          return err  // Rollback() fires via defer
      }
      return tx.Commit()
}

  ---
Detecting Leaks in Practice

// 1. Use goleak in tests to detect goroutine leaks
import "go.uber.org/goleak"

func TestStartPoller(t *testing.T) {
defer goleak.VerifyNone(t)  // fails if any goroutines leak after test

      ctx, cancel := context.WithCancel(context.Background())
      svc := NewService()
      svc.StartPoller(ctx)

      cancel()
      time.Sleep(100 * time.Millisecond)
      // goleak checks no goroutines from this test are still running
}

// 2. Expose goroutine count metric
http.HandleFunc("/debug/goroutines", func(w http.ResponseWriter, r *http.Request) {
fmt.Fprintf(w, "goroutines: %d\n", runtime.NumGoroutine())
})

// 3. pprof goroutine profile — shows stack traces of all live goroutines
// GET /debug/pprof/goroutine?debug=2
import _ "net/http/pprof"

// 4. Periodic goroutine count logging
go func() {
ticker := time.NewTicker(30 * time.Second)
defer ticker.Stop()
for range ticker.C {
slog.Info("runtime stats", "goroutines", runtime.NumGoroutine())
}
}()

  ---
Summary Table

┌───────────────────────────────────┬───────────────────────────────────────────┬────────────────────────────┐
│             Scenario              │                What Leaks                 │         Detection          │
├───────────────────────────────────┼───────────────────────────────────────────┼────────────────────────────┤
│ Missing cancel() call             │ Timer goroutine + parent child list entry │ go vet, code review        │
├───────────────────────────────────┼───────────────────────────────────────────┼────────────────────────────┤
│ Goroutine ignores ctx.Done()      │ Goroutine stack + all its references      │ goleak, pprof              │
├───────────────────────────────────┼───────────────────────────────────────────┼────────────────────────────┤
│ Large value in long-lived ctx     │ Referenced objects held by GC             │ Heap profile               │
├───────────────────────────────────┼───────────────────────────────────────────┼────────────────────────────┤
│ Unbounded WithValue chain         │ Nested context wrappers                   │ Code review                │
├───────────────────────────────────┼───────────────────────────────────────────┼────────────────────────────┤
│ HTTP request without ctx          │ TCP connection + request goroutine        │ Connection pool metrics    │
├───────────────────────────────────┼───────────────────────────────────────────┼────────────────────────────┤
│ context.Background() in goroutine │ Goroutine outlives intended scope         │ goleak                     │
├───────────────────────────────────┼───────────────────────────────────────────┼────────────────────────────┤
│ time.After in loop                │ Timer objects until they fire             │ Heap/goroutine profile     │
├───────────────────────────────────┼───────────────────────────────────────────┼────────────────────────────┤
│ Unclosed DB transaction           │ DB connection + row locks                 │ DB connection pool metrics │
└───────────────────────────────────┴───────────────────────────────────────────┴────────────────────────────┘

The single rule that prevents most of these: every goroutine you spawn must have a documented exit condition, and that exit must be tied to a context.

✻ Worked for 1m 11s

❯ How do you integrate context with I/O layers — e.g. database or HTTP clients?

⏺ Integrating Context with I/O Layers in Go

  ---
The Core Pattern

Every I/O operation has two parts: initiating it and waiting for it. Context hooks into the waiting phase — it doesn't magically interrupt syscalls, it signals the library to abandon the operation and return
an error.

ctx cancelled
│
▼
library detects ctx.Done() closed
│
├── in-flight: cancels the underlying syscall / connection
└── pending:   never starts the operation, returns ctx.Err() immediately

  ---
1. database/sql

Every method has a context variant — always use it

// Without context — cannot be cancelled or timed out
rows, err := db.Query("SELECT ...")
row := db.QueryRow("SELECT ...")
_, err = db.Exec("INSERT ...")

// With context — always prefer these
rows, err := db.QueryContext(ctx, "SELECT ...")
row := db.QueryRowContext(ctx, "SELECT ...")
_, err = db.ExecContext(ctx, "INSERT ...")

What context cancellation actually does at the DB level

func (r *Repo) GetUser(ctx context.Context, id int64) (*User, error) {
// If ctx is cancelled BEFORE this line:
//   → db.QueryRowContext returns immediately with ctx.Err()
//   → no network round-trip at all
//
// If ctx is cancelled WHILE waiting for DB response:
//   → driver sends a cancellation to the DB server (PostgreSQL: pg_cancel_backend)
//   → DB aborts the query, releases locks
//   → QueryRowContext returns with ctx.Err()
//
// If ctx is cancelled AFTER response received but before Scan:
//   → Scan still succeeds (data already in memory)
row := db.QueryRowContext(ctx, `SELECT id, name, email FROM users WHERE id = $1`, id)

      var u User
      if err := row.Scan(&u.ID, &u.Name, &u.Email); err != nil {
          if errors.Is(err, sql.ErrNoRows) {
              return nil, ErrNotFound
          }
          return nil, fmt.Errorf("scan user %d: %w", err)
      }
      return &u, nil
}

Acquiring a connection from the pool respects context

// db.QueryContext does NOT immediately send the query.
// Step 1: acquire a connection from the pool — this blocks if pool is exhausted.
// Step 2: send query over the connection.
// Context cancellation works at BOTH steps.

func (r *Repo) SlowReport(ctx context.Context) (*Report, error) {
// If all 25 connections in the pool are busy and ctx times out
// while waiting for one → returns immediately without touching the DB
rows, err := r.db.QueryContext(ctx, `SELECT ... FROM large_table ...`)
if err != nil {
if errors.Is(err, context.DeadlineExceeded) {
return nil, fmt.Errorf("timed out waiting for db connection: %w", err)
}
return nil, fmt.Errorf("query report: %w", err)
}
defer rows.Close()
// ...
}

Transactions — context ties the transaction lifecycle

func (r *Repo) Transfer(ctx context.Context, fromID, toID int64, amount decimal.Decimal) error {
// BeginTx: acquires connection + starts transaction
// context cancellation mid-transaction causes automatic rollback
tx, err := r.db.BeginTx(ctx, &sql.TxOptions{
Isolation: sql.LevelSerializable,
})
if err != nil {
return fmt.Errorf("begin tx: %w", err)
}
defer tx.Rollback() // no-op after Commit

      // Each statement forward ctx — if cancelled between statements,
      // the next ExecContext returns immediately, defer Rollback fires
      _, err = tx.ExecContext(ctx,
          `UPDATE accounts SET balance = balance - $1 WHERE id = $2`, amount, fromID)
      if err != nil {
          return fmt.Errorf("debit account %d: %w", fromID, err)
      }

      _, err = tx.ExecContext(ctx,
          `UPDATE accounts SET balance = balance + $1 WHERE id = $2`, amount, toID)
      if err != nil {
          return fmt.Errorf("credit account %d: %w", toID, err)
      }

      _, err = tx.ExecContext(ctx,
          `INSERT INTO transfers (from_id, to_id, amount) VALUES ($1, $2, $3)`,
          fromID, toID, amount)
      if err != nil {
          return fmt.Errorf("record transfer: %w", err)
      }

      return tx.Commit()
}

Iterating rows — check context between rows for long scans

func (r *Repo) StreamUsers(ctx context.Context, fn func(*User) error) error {
rows, err := r.db.QueryContext(ctx, `SELECT id, name, email FROM users ORDER BY id`)
if err != nil {
return fmt.Errorf("query users: %w", err)
}
defer rows.Close()

      for rows.Next() {
          // rows.Next() itself checks ctx — returns false if cancelled
          // but only between network reads, not mid-row
          // For extra safety on huge result sets, check explicitly:
          if err := ctx.Err(); err != nil {
              return fmt.Errorf("stream interrupted: %w", err)
          }

          var u User
          if err := rows.Scan(&u.ID, &u.Name, &u.Email); err != nil {
              return fmt.Errorf("scan: %w", err)
          }
          if err := fn(&u); err != nil {
              return err
          }
      }
      return rows.Err() // captures any error that ended iteration early
}

Prepared statements with context

type Repo struct {
db      *sql.DB
getUserStmt *sql.Stmt
}

func NewRepo(ctx context.Context, db *sql.DB) (*Repo, error) {
// PrepareContext: context for the preparation step, not execution
stmt, err := db.PrepareContext(ctx, `SELECT id, name FROM users WHERE id = $1`)
if err != nil {
return nil, fmt.Errorf("prepare getUser: %w", err)
}
return &Repo{db: db, getUserStmt: stmt}, nil
}

func (r *Repo) GetUser(ctx context.Context, id int64) (*User, error) {
// Execution uses its own context — independent of preparation context
row := r.getUserStmt.QueryRowContext(ctx, id)
var u User
if err := row.Scan(&u.ID, &u.Name); err != nil {
return nil, fmt.Errorf("scan: %w", err)
}
return &u, nil
}

  ---
2. HTTP Client

The right way — NewRequestWithContext

func (c *APIClient) GetProduct(ctx context.Context, id string) (*Product, error) {
req, err := http.NewRequestWithContext(ctx, http.MethodGet,
c.baseURL+"/products/"+id, nil)
if err != nil {
return nil, fmt.Errorf("build request: %w", err)
}
req.Header.Set("Authorization", "Bearer "+c.token)

      resp, err := c.http.Do(req)
      if err != nil {
          // Distinguish context cancellation from network error
          if errors.Is(err, context.DeadlineExceeded) || errors.Is(err, context.Canceled) {
              return nil, fmt.Errorf("request cancelled: %w", err)
          }
          return nil, fmt.Errorf("do request: %w", err)
      }
      defer resp.Body.Close()

      if resp.StatusCode != http.StatusOK {
          return nil, fmt.Errorf("unexpected status %d", resp.StatusCode)
      }

      var p Product
      if err := json.NewDecoder(resp.Body).Decode(&p); err != nil {
          return nil, fmt.Errorf("decode: %w", err)
      }
      return &p, nil
}

Client-level timeout vs context timeout — understand the difference

// Client-level timeout: applies to every request, regardless of context
// This is a hard ceiling — the request WILL be cancelled after this duration
httpClient := &http.Client{
Timeout: 30 * time.Second, // absolute max for any single request
}

// Context timeout: per-call budget, can be shorter than client timeout
// Effective timeout = min(client.Timeout, context deadline)
func (c *Client) FetchWithBudget(ctx context.Context, url string) (*Response, error) {
// Give this specific call 5s — even though client allows 30s
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()

      req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
      return c.http.Do(req)
}

Scenario: client.Timeout=30s, ctx deadline in 3s
→ effective timeout is 3s (context fires first)

Scenario: client.Timeout=30s, ctx has no deadline
→ effective timeout is 30s (client timeout fires)

Scenario: client.Timeout=2s, ctx deadline in 10s
→ effective timeout is 2s (client timeout fires first)

Reading the response body respects context

func (c *Client) Download(ctx context.Context, url string) ([]byte, error) {
req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
resp, err := c.http.Do(req)
if err != nil {
return nil, err
}
defer resp.Body.Close()

      // Context cancellation DURING body read:
      // io.ReadAll reads in chunks — if ctx is cancelled mid-read,
      // the next chunk read returns ctx.Err() wrapped in a net error
      body, err := io.ReadAll(resp.Body)
      if err != nil {
          return nil, fmt.Errorf("read body: %w", err)
      }
      return body, nil
}

// For large responses, use context-aware reading explicitly:
func (c *Client) StreamDownload(ctx context.Context, url string, dst io.Writer) error {
req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
resp, err := c.http.Do(req)
if err != nil {
return err
}
defer resp.Body.Close()

      buf := make([]byte, 32*1024)
      for {
          if err := ctx.Err(); err != nil {
              return fmt.Errorf("download cancelled: %w", err)
          }
          n, err := resp.Body.Read(buf)
          if n > 0 {
              if _, werr := dst.Write(buf[:n]); werr != nil {
                  return fmt.Errorf("write: %w", werr)
              }
          }
          if err == io.EOF {
              return nil
          }
          if err != nil {
              return fmt.Errorf("read: %w", err)
          }
      }
}

Retry logic with context — respect remaining budget

func (c *Client) FetchWithRetry(ctx context.Context, url string) (*http.Response, error) {
var lastErr error
backoff := 100 * time.Millisecond

      for attempt := 0; attempt < 3; attempt++ {
          // Check BEFORE the attempt — don't start if already cancelled
          if err := ctx.Err(); err != nil {
              return nil, fmt.Errorf("cancelled before attempt %d: %w", attempt+1, err)
          }

          req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
          resp, err := c.http.Do(req)
          if err == nil && resp.StatusCode < 500 {
              return resp, nil
          }

          lastErr = err
          if attempt < 2 {
              // Wait with context awareness — don't sleep through cancellation
              select {
              case <-ctx.Done():
                  return nil, fmt.Errorf("cancelled during backoff: %w", ctx.Err())
              case <-time.After(backoff):
                  backoff *= 2
              }
          }
      }
      return nil, fmt.Errorf("all retries failed: %w", lastErr)
}

  ---
3. Redis (go-redis)

import "github.com/redis/go-redis/v9"

type Cache struct {
rdb *redis.Client
}

func (c *Cache) Get(ctx context.Context, key string) (string, error) {
// All go-redis operations accept context as first argument
val, err := c.rdb.Get(ctx, key).Result()
if err != nil {
if errors.Is(err, redis.Nil) {
return "", ErrCacheMiss
}
return "", fmt.Errorf("redis get %s: %w", key, err)
}
return val, nil
}

func (c *Cache) Set(ctx context.Context, key string, val any, ttl time.Duration) error {
if err := c.rdb.Set(ctx, key, val, ttl).Err(); err != nil {
return fmt.Errorf("redis set %s: %w", key, err)
}
return nil
}

// Pipeline with context
func (c *Cache) GetMulti(ctx context.Context, keys []string) (map[string]string, error) {
pipe := c.rdb.Pipeline()
cmds := make([]*redis.StringCmd, len(keys))

      for i, key := range keys {
          cmds[i] = pipe.Get(ctx, key)
      }

      // Context applies to the pipeline execution
      if _, err := pipe.Exec(ctx); err != nil && !errors.Is(err, redis.Nil) {
          return nil, fmt.Errorf("pipeline exec: %w", err)
      }

      result := make(map[string]string, len(keys))
      for i, cmd := range cmds {
          if val, err := cmd.Result(); err == nil {
              result[keys[i]] = val
          }
      }
      return result, nil
}

  ---
4. gRPC — Context Carries Deadlines Automatically

gRPC propagates context deadlines and cancellation over the wire — the remote server receives and honours them.

// Client side
func (c *UserClient) GetUser(ctx context.Context, id string) (*User, error) {
// If ctx has a deadline, gRPC sends it as a grpc-timeout header
// Remote server's context is cancelled when our deadline fires
resp, err := c.grpc.GetUser(ctx, &pb.GetUserRequest{Id: id})
if err != nil {
// gRPC wraps context errors in status errors
st, ok := status.FromError(err)
if ok && st.Code() == codes.DeadlineExceeded {
return nil, fmt.Errorf("grpc deadline exceeded: %w", context.DeadlineExceeded)
}
return nil, fmt.Errorf("grpc get user: %w", err)
}
return protoToUser(resp), nil
}

// Server side — context is already wired to client's deadline
func (s *UserServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
// This ctx is cancelled when the client's deadline fires or client disconnects
user, err := s.repo.Get(ctx, req.Id)  // forward ctx to all I/O
if err != nil {
if errors.Is(err, context.DeadlineExceeded) {
return nil, status.Error(codes.DeadlineExceeded, "deadline exceeded")
}
return nil, status.Errorf(codes.Internal, "get user: %v", err)
}
return userToProto(user), nil
}

Propagating metadata (trace IDs) via context

// Client: inject trace ID into outgoing context
func withTraceID(ctx context.Context) context.Context {
traceID, _ := TraceIDFromContext(ctx)
md := metadata.Pairs("x-trace-id", traceID)
return metadata.NewOutgoingContext(ctx, md)
}

resp, err := c.grpc.GetUser(withTraceID(ctx), req)

// Server: extract from incoming context
func (s *Server) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
md, ok := metadata.FromIncomingContext(ctx)
if ok {
if traceIDs := md.Get("x-trace-id"); len(traceIDs) > 0 {
ctx = WithTraceID(ctx, traceIDs[0])
}
}
// ctx now carries trace ID through all downstream calls
return s.repo.Get(ctx, req.Id)
}

  ---
5. Wrapping Libraries That Don't Support Context

Sometimes you work with libraries that have no context support. Two patterns:

Pattern A: Run in goroutine, race against context

func queryLegacyDB(ctx context.Context, query string) ([]Row, error) {
type result struct {
rows []Row
err  error
}
ch := make(chan result, 1) // buffered — goroutine never blocks

      go func() {
          rows, err := legacyDB.Query(query) // no context support
          ch <- result{rows, err}
      }()

      select {
      case <-ctx.Done():
          // Context cancelled — we return, but the goroutine still runs to completion
          // The buffered channel ensures it doesn't leak (it will send and exit)
          return nil, fmt.Errorf("legacy query cancelled: %w", ctx.Err())
      case r := <-ch:
          return r.rows, r.err
      }
}

Important: the goroutine still runs to completion even after context is cancelled — you're abandoning the result, not cancelling the operation. This is unavoidable without library support. Budget for this in
your connection pool sizing.

Pattern B: Adapter struct that checks context before delegating

// Wrap the legacy client to check context at each call boundary
type ContextualLegacyClient struct {
inner *legacy.Client
}

func (c *ContextualLegacyClient) Get(ctx context.Context, key string) (string, error) {
// Fast path: don't even try if already cancelled
if err := ctx.Err(); err != nil {
return "", fmt.Errorf("get %s: %w", key, err)
}

      // The actual call is not cancellable, but at least we don't start it needlessly
      val, err := c.inner.Get(key)
      if err != nil {
          return "", fmt.Errorf("legacy get %s: %w", key, err)
      }

      // Check again after — did context expire while we were waiting?
      if err := ctx.Err(); err != nil {
          return "", fmt.Errorf("get %s completed but context done: %w", key, err)
      }
      return val, nil
}

  ---
6. Propagating Context Through Middleware Chains

HTTP server middleware

// Each middleware enriches the context and passes it down
func TraceMiddleware(next http.Handler) http.Handler {
return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
traceID := r.Header.Get("X-Trace-Id")
if traceID == "" {
traceID = uuid.New().String()
}
ctx := WithTraceID(r.Context(), traceID)
w.Header().Set("X-Trace-Id", traceID)
next.ServeHTTP(w, r.WithContext(ctx)) // r.WithContext creates shallow copy with new ctx
})
}

func TimeoutMiddleware(timeout time.Duration) func(http.Handler) http.Handler {
return func(next http.Handler) http.Handler {
return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
ctx, cancel := context.WithTimeout(r.Context(), timeout)
defer cancel()
next.ServeHTTP(w, r.WithContext(ctx))
})
}
}

func AuthMiddleware(next http.Handler) http.Handler {
return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
user, err := parseToken(r.Header.Get("Authorization"))
if err != nil {
http.Error(w, "unauthorized", http.StatusUnauthorized)
return
}
ctx := WithUser(r.Context(), user) // attach auth user to context
next.ServeHTTP(w, r.WithContext(ctx))
})
}

// Wire them up — each wraps the next
mux.Handle("/api/", TraceMiddleware(
TimeoutMiddleware(10*time.Second)(
AuthMiddleware(apiHandler),
),
))

// Handler receives a context that has:
// - trace ID (from TraceMiddleware)
// - 10s deadline (from TimeoutMiddleware)
// - authenticated user (from AuthMiddleware)
// - connection lifecycle (from r.Context() baseline)
func apiHandler(w http.ResponseWriter, r *http.Request) {
ctx := r.Context()
user, _ := UserFromContext(ctx)     // set by AuthMiddleware
traceID, _ := TraceIDFromContext(ctx) // set by TraceMiddleware
// ctx deadline: 10s from TimeoutMiddleware
}

  ---
7. Testing I/O with Context

// Test that your I/O respects cancellation
func TestGetUser_CancelledContext(t *testing.T) {
// Create an already-cancelled context
ctx, cancel := context.WithCancel(context.Background())
cancel() // cancel immediately

      repo := NewRepo(testDB)
      _, err := repo.GetUser(ctx, 123)

      // Should return quickly with context error, not attempt DB query
      if !errors.Is(err, context.Canceled) {
          t.Errorf("expected context.Canceled, got %v", err)
      }
}

func TestGetUser_Timeout(t *testing.T) {
ctx, cancel := context.WithTimeout(context.Background(), 1*time.Nanosecond)
defer cancel()
time.Sleep(1 * time.Millisecond) // ensure deadline has passed

      repo := NewRepo(testDB)
      _, err := repo.GetUser(ctx, 123)

      if !errors.Is(err, context.DeadlineExceeded) {
          t.Errorf("expected DeadlineExceeded, got %v", err)
      }
}

// Test that goroutines don't leak when context is cancelled
func TestStartPoller_NoGoroutineLeak(t *testing.T) {
defer goleak.VerifyNone(t)

      ctx, cancel := context.WithCancel(context.Background())
      svc := NewService(mockClient)
      svc.StartPoller(ctx)

      cancel()
      time.Sleep(50 * time.Millisecond) // give goroutine time to exit
      // goleak verifies no goroutines remain
}

  ---
Summary — Rules Per Layer

database/sql     use *Context variants for every call
BeginTx always gets ctx
defer tx.Rollback() always
check ctx.Err() between rows on large scans

http.Client      always NewRequestWithContext
client.Timeout = hard ceiling, ctx = per-call budget
check ctx before retry attempts
use buffered channel in retry backoff select

redis/go-redis   ctx is first arg on every method
pipeline.Exec(ctx) covers all pipelined commands

gRPC             ctx deadlines propagate over the wire automatically
server receives and must forward ctx to its own I/O

no-ctx library   goroutine + buffered channel + select (accept goroutine outlives ctx)
or check ctx.Err() before and after the call

middleware       r.WithContext(ctx) to thread enriched ctx through HTTP chain
each layer adds its concern (trace, timeout, auth)
handlers always read from r.Context(), never create new roots

✻ Cooked for 1m 26s

❯ How Go runtime interrupts syscalls on context cancel?

⏺ How Go Runtime Interrupts Syscalls on Context Cancel

This goes deep into the runtime. The answer is different depending on what kind of I/O is being interrupted.

  ---
The Fundamental Model First

Go uses an M:N threading model — N goroutines multiplexed onto M OS threads. The key insight for context cancellation:

For network I/O:   goroutine is parked, OS thread is FREE
cancel → unpark goroutine → returns error
OS thread was never blocked

For blocking I/O:  goroutine AND OS thread are both blocked
cancel CANNOT interrupt the syscall in progress
goroutine resumes after syscall, then sees ctx.Err()

This distinction drives everything below.

  ---
Path 1: Network I/O — The Netpoller

How network reads work without context

conn.Read(buf)
│
├── Go sets socket to O_NONBLOCK mode at creation
│
├── runtime calls read() syscall
│   ├── data available → returns immediately ✓
│   └── would block → returns EAGAIN
│             │
│             └── goroutine parks itself
│                 fd registered with epoll/kqueue
│                 OS thread (M) freed to run other goroutines
│
└── epoll/kqueue fires when data arrives
→ netpoller wakes the goroutine
→ goroutine resumes, retries read()

The OS thread is never blocked for network I/O. The goroutine is logically blocked but the thread is free.

How context cancellation hooks into this

Neither context nor net.Conn know about each other directly. The bridge is SetDeadline:

ctx cancelled
│
│  (goroutine watching ctx.Done())
▼
conn.SetDeadline(time.Now())   ← sets deadline to the past
│
▼
internal/poll.FD.SetDeadline()
│
▼
runtime_pollSetDeadline(pd *pollDesc, d int64, mode int)
│
├── pd.rd = deadline (read deadline, nanoseconds)
├── pd.wd = deadline (write deadline)
│
├── deadline already passed?
│       YES → call netpollunblock() immediately
│              → goroutine blocked on Read is made runnable right now
│
└── deadline in future?
→ arm a runtime timer to call netpollunblock() at deadline

pollDesc is the per-file-descriptor structure inside the runtime:

// runtime/netpoll.go (simplified)
type pollDesc struct {
link *pollDesc   // linked list

      fd      uintptr  // file descriptor
      closing bool

      rg  atomic.Uintptr  // goroutine waiting for read, or pdReady/pdWait
      wg  atomic.Uintptr  // goroutine waiting for write

      rd  int64  // read deadline (nanoseconds, 0 = no deadline)
      wd  int64  // write deadline

      rt  timer   // read deadline timer
      wt  timer   // write deadline timer
}

When netpollunblock fires:

// runtime/netpoll.go (simplified)
func netpollunblock(pd *pollDesc, mode int32, ioready bool) *g {
gpp := &pd.rg  // or pd.wg for writes

      // Atomically swap the waiting goroutine pointer out
      old := gpp.Swap(pdReady)

      if old == pdWait {
          // There was a goroutine waiting — return it to be scheduled
          return gp  // scheduler puts this goroutine on a run queue
      }
      return nil
}

The goroutine's Read call then returns:

// internal/poll/fd_unix.go (simplified)
func (fd *FD) Read(p []byte) (int, error) {
for {
n, err := syscall.Read(fd.Sysfd, p)
if err != nil {
if err == syscall.EAGAIN && fd.pd.pollable() {
if err = fd.pd.waitRead(fd.isFile); err == nil {
continue  // retry read
}
}
return n, err  // ← returns here with "i/o timeout" error
}
return n, nil
}
}

Full call chain: cancel() → goroutine unblocked

ctx.cancel()
│
│ closes done channel
│ child contexts cancelled
│
└── goroutine in http.Transport watching ctx.Done() fires
│
▼
persistConn.cancelRequest()
│
▼
conn.SetDeadline(time.Now())   [net/net.go]
│
▼
fd.SetDeadline(t)              [net/fd_posix.go]
│
▼
poll.FD.SetDeadline(t)         [internal/poll/fd_poll_runtime.go]
│
▼
runtime_pollSetDeadline(pd, t) [runtime/netpoll.go]
│
├── deadline expired → netpollunblock(pd)
│           │
│           └── goroutine blocked in Read made RUNNABLE
│                       │
│                       ▼
│               Read() returns "use of closed network connection"
│
└── stores timer → fires later → netpollunblock(pd)

  ---
Path 2: Blocking Syscalls — File I/O, DNS, Pipes

Regular files (os.File) on Linux cannot use epoll. A read() on /var/data/file.bin blocks the OS thread until data arrives.

How Go handles blocking syscalls

// Before blocking syscall: runtime/proc.go
func entersyscall() {
// Detach this goroutine's M from the P (processor)
// P is now free to run other goroutines on a different M
// This M is "stuck" in the syscall
releasep()
}

// After syscall returns: runtime/proc.go
func exitsyscall() {
// Try to reacquire a P to continue running
// If none available, goroutine goes to run queue
}

During the blocking syscall, the OS thread is genuinely blocked. The sysmon (system monitor) goroutine detects this and hands the P to a new M so other goroutines keep running.

Context cancellation with blocking file I/O

Short answer: it cannot interrupt the in-progress syscall.

func readFile(ctx context.Context, path string) ([]byte, error) {
f, _ := os.Open(path)
defer f.Close()

      buf := make([]byte, 1024*1024)

      // ctx is cancelled RIGHT NOW
      // This read() syscall is already in progress on the OS thread
      // The OS thread is blocked in kernel space
      // There is no mechanism to interrupt it
      n, err := f.Read(buf)  // ← this WILL complete regardless of ctx

      // ONLY after the syscall returns does context get checked
      if err := ctx.Err(); err != nil {
          return nil, err
      }
      return buf[:n], nil
}

os.File.SetDeadline does NOT exist for regular files — only for network connections and pipes on some platforms.

The correct pattern for cancellable file reading:

func readFileCancellable(ctx context.Context, path string) ([]byte, error) {
type result struct {
data []byte
err  error
}
ch := make(chan result, 1) // buffered: goroutine never leaks

      go func() {
          data, err := os.ReadFile(path) // may block, cannot be interrupted
          ch <- result{data, err}        // always completes and sends
      }()

      select {
      case <-ctx.Done():
          // We abandon the result, but the goroutine runs to completion
          // The buffered channel ensures it doesn't leak
          return nil, fmt.Errorf("read file: %w", ctx.Err())
      case r := <-ch:
          return r.data, r.err
      }
}

Note: the goroutine outlives the context cancellation. You're abandoning the result, not cancelling the OS operation.

  ---
Path 3: database/sql — Two Different Mechanisms

Drivers with native context support

// driver.QueryerContext interface
type QueryerContext interface {
QueryContext(ctx context.Context, query string, args []NamedValue) (Rows, error)
}

For PostgreSQL (lib/pq, pgx): sends a cancellation request — a separate TCP connection to the same DB server with the process ID and secret key of the running query.

cancel() fires
│
▼
driver detects ctx.Done()
│
▼
opens NEW TCP connection to postgres :5432
sends CancelRequest packet {processID, secretKey}
│
▼
postgres backend receives it
kills the running query
original connection returns "ERROR: canceling statement due to user request"

This is a real DB-level cancellation — the query is actually aborted in postgres, releasing locks and CPU.

Drivers without context support — ctxutil.go fallback

// database/sql/ctxutil.go (simplified)
func ctxDriverQuery(ctx context.Context, queryerCtx driver.QueryerContext,
queryer driver.Queryer, query string, nvdargs []driver.NamedValue) (driver.Rows, error) {

      if queryerCtx != nil {
          // Driver supports context — delegate directly
          return queryerCtx.QueryContext(ctx, query, nvdargs)
      }

      // Driver does NOT support context — race it
      // Start query in separate goroutine
      rowsCh := make(chan rowsiErr, 1)
      go func() {
          rows, err := queryer.Query(query, dargs)
          rowsCh <- rowsiErr{rows, err}
      }()

      select {
      case <-ctx.Done():
          // Caller gets error immediately
          // BUT the goroutine above keeps running until query completes
          // When it finishes, it sends to buffered channel and exits cleanly
          return nil, ctx.Err()
      case rwse := <-rowsCh:
          return rwse.rows, rwse.err
      }
}

Key difference: native context support = real query cancellation at the DB. Fallback = abandons the result on the Go side, DB query runs to completion.

  ---
Path 4: Goroutine Preemption via SIGURG

This is separate from context cancellation but often confused with it. Go 1.14 introduced asynchronous preemption.

The scheduler's sysmon goroutine sends SIGURG to OS threads running goroutines that have been running for > 10ms:

sysmon detects goroutine running > 10ms on a thread
│
▼
sends SIGURG to that OS thread
│
▼
signal handler: runtime/signal_unix.go → doSigPreempt()
│
├── goroutine at a safe preemption point?
│       YES → save state, put goroutine back on run queue, run scheduler
│       NO  → set asyncPreempt flag, check at next safe point
│
└── "safe point" = not in a syscall, not holding locks, not in runtime code

This has nothing to do with context. SIGURG is scheduling fairness — preventing one goroutine from monopolizing a thread. Context cancellation is application-level signaling.

However, SIGURG is why goroutines in tight loops (no function calls, no allocations) are now preemptible — before Go 1.14, they weren't.

  ---
Summary: What Actually Happens Per I/O Type

Network I/O (TCP, UDP, Unix sockets)
─────────────────────────────────────
Mechanism:   Non-blocking fd + epoll/kqueue/iocp via netpoller
On cancel:   SetDeadline(now) → runtime_pollSetDeadline → netpollunblock
Goroutine made runnable, Read/Write returns error
Interrupted: YES — mid-operation, goroutine unblocked immediately

Regular file I/O
─────────────────
Mechanism:   Blocking syscall, OS thread parked
On cancel:   Cannot interrupt. Goroutine checks ctx.Err() after syscall returns
Interrupted: NO — must wait for syscall to complete, then error propagates

Database (native ctx driver, e.g. pgx)
────────────────────────────────────────
Mechanism:   Driver-specific. PostgreSQL: CancelRequest on separate connection
On cancel:   Real DB-side query cancellation, locks released
Interrupted: YES — at DB level

Database (no ctx driver fallback)
───────────────────────────────────
Mechanism:   Goroutine race in ctxutil.go
On cancel:   Go-side abandonment only, DB query runs to completion
Interrupted: NO at DB level, YES from caller's perspective

HTTP client
────────────
Mechanism:   Transport goroutine watches ctx.Done(), calls conn.SetDeadline
On cancel:   Same as network I/O path
Interrupted: YES

Goroutine preemption (unrelated to ctx)
────────────────────────────────────────
Mechanism:   SIGURG from sysmon after 10ms
Purpose:     Scheduling fairness, not cancellation

  ---
Why This Matters Practically

// This looks like it cancels a file read on timeout — it DOES NOT
ctx, cancel := context.WithTimeout(ctx, 100*time.Millisecond)
defer cancel()
data, err := os.ReadFile("/slow/nfs/mount/bigfile") // ignores ctx entirely

// This correctly cancels a network read on timeout
ctx, cancel := context.WithTimeout(ctx, 100*time.Millisecond)
defer cancel()
resp, err := http.Get(req.WithContext(ctx)) // properly interrupted

// This correctly cancels a DB query at the DB level (pgx)
ctx, cancel := context.WithTimeout(ctx, 100*time.Millisecond)
defer cancel()
rows, err := db.QueryContext(ctx, "SELECT pg_sleep(10)") // postgres kills the query

The runtime can only interrupt what it controls. Network I/O goes through the runtime's netpoller — fully controllable. Blocking OS syscalls bypass the scheduler entirely — not interruptible.

