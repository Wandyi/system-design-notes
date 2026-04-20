⏺ Staff-Level Design Patterns for Returning Values in Go

---
Foundational Principle: Zero Value as a Contract

The most overlooked design decision happens before writing a single function — whether your type's zero value is useful. This shapes every return in your entire API.

// BAD: zero value is a trap — callers can't tell if it was initialized
type RateLimiter struct {
initialized bool
rate        float64
burst        int
}

func NewRateLimiter(rate float64) *RateLimiter {
return &RateLimiter{initialized: true, rate: rate, burst: 10}
}

// If someone does: var rl RateLimiter — it silently does nothing or panics                                                                                                                                         

// GOOD: zero value is meaningful and safe                                                                                                                                                                          
type RateLimiter struct {
rate  float64       // 0 = unlimited — valid, documented behavior                                                                                                                                               
burst int           // 0 = no burst — valid                                                                                                                                                                     
mu    sync.Mutex    // zero value is unlocked — correct
tokens float64      // 0 = start empty — correct                                                                                                                                                                
}

// var rl RateLimiter works and is safe                                                                                                                                                                             
// NewRateLimiter just sets non-zero defaults

Staff insight: When you design your type's zero value to be correct, you eliminate an entire class of "must call Init() first" bugs. sync.Mutex, bytes.Buffer, http.Client all have useful zero values. Design yours
the same way — it makes returning values from constructors optional rather than mandatory.

---
1. Value vs Pointer Returns — Memory Semantics

The decision tree

Return value when:
├── struct is small (≤ 3-4 machine words, ~32 bytes)
├── caller should not mutate the original
├── nil is NOT a meaningful state
└── immutability is part of the API contract

Return pointer when:
├── struct is large (avoid copy cost)
├── nil IS meaningful ("no result" differs from "zero result")
├── caller must mutate the returned value
└── identity matters (two callers sharing the same object)

// GOOD: small struct — return value, copy is cheap
// Compiler can stack-allocate, no heap pressure                                                                                                                                                                    
type Point struct{ X, Y float64 }

func Midpoint(a, b Point) Point {
return Point{(a.X + b.X) / 2, (a.Y + b.Y) / 2}                                                                                                                                                                  
}

// GOOD: large struct — return pointer, avoid 2KB copy on every call                                                                                                                                                
type QueryResult struct {
Rows     []Row            // slice header + backing array                                                                                                                                                       
Metadata QueryMetadata
Stats    ExecutionStats                                                                                                                                                                                         
// ... 20 more fields
}

func (db *DB) Query(ctx context.Context, q string) (*QueryResult, error) {
// ...      
}

The nil-as-signal pattern

// nil pointer has semantic meaning: "nothing was found"                                                                                                                                                            
// This is different from a zero-value Config{}                                                                                                                                                                     

func (r *Repo) FindByEmail(ctx context.Context, email string) (*User, error) {
// Returns nil, nil when not found — nil is not an error here                                                                                                                                                   
// Caller must check: if user == nil { ... }                                                                                                                                                                    
var u User
err := r.db.QueryRowContext(ctx, query, email).Scan(&u.ID, &u.Name)
if errors.Is(err, sql.ErrNoRows) {
return nil, nil  // explicit: "not found" is not an error                                                                                                                                                   
}
if err != nil {
return nil, fmt.Errorf("find by email: %w", err)
}
return &u, nil
}

// vs — when absence IS an error (different domain semantics)                                                                                                                                                       
func (r *Repo) GetByID(ctx context.Context, id string) (*User, error) {
// GetByID implies the caller expects the record to exist                                                                                                                                                       
// Returns ErrNotFound — absence is treated as an error                                                                                                                                                         
var u User
err := r.db.QueryRowContext(ctx, query, id).Scan(&u.ID, &u.Name)
if errors.Is(err, sql.ErrNoRows) {
return nil, fmt.Errorf("user %s: %w", id, ErrNotFound)
}
// ...                                                                                                                                                                                                          
}

Staff insight: Whether nil is an error vs a valid "not found" signal is a domain decision, not a technical one. Be consistent within a package — don't have Find* methods that return ErrNotFound alongside Get*    
methods that return nil for the same "not found" condition.

---
2. The (value, error) Pattern — Deep Semantics

Never return a meaningful value alongside a non-nil error

// BAD: what does the caller do with a non-nil User when err != nil?
// Do they use it? Ignore it? This is undefined behavior in your API.                                                                                                                                               
func (r *Repo) Get(ctx context.Context, id string) (*User, error) {
u := &User{ID: id}          // partially populated                                                                                                                                                              
if err := r.scan(u); err != nil {
return u, err           // caller now has to decide what to trust
}
return u, nil
}

// GOOD: zero value on error — caller's contract is clear                                                                                                                                                           
func (r *Repo) Get(ctx context.Context, id string) (*User, error) {
var u User
if err := r.scan(&u); err != nil {
return nil, fmt.Errorf("get user %s: %w", id, err)
}
return &u, nil
}

The io.Reader exception — document when you break the rule

// io.Reader is the canonical exception: n > 0 AND err != nil is valid                                                                                                                                              
// This is explicitly documented and well-understood                                                                                                                                                                
type Reader interface {
Read(p []byte) (n int, err error)
// Contract: callers MUST process n bytes BEFORE examining err                                                                                                                                                  
// err = io.EOF can coexist with n > 0 on the final read                                                                                                                                                        
}

// If your domain has a similar "partial success" pattern, document it explicitly                                                                                                                                   
// Returns (processed int, err error)
// processed may be > 0 even when err != nil — partial results are valid                                                                                                                                            
func (p *Processor) Process(items []Item) (int, error) {
var processed int
for _, item := range items {
if err := p.handle(item); err != nil {
// Return how many succeeded before the failure                                                                                                                                                         
// Caller can resume from processed+1                                                                                                                                                                   
return processed, fmt.Errorf("item %s: %w", item.ID, err)
}
processed++
}
return processed, nil
}

Sentinel errors vs typed errors — return type determines API flexibility

// Sentinel: simple, caller uses errors.Is()                                                                                                                                                                        
// GOOD for: well-known, stable error conditions                                                                                                                                                                    
var ErrNotFound = errors.New("not found")
var ErrConflict = errors.New("conflict")

func (r *Repo) Create(ctx context.Context, u User) error {
_, err := r.db.ExecContext(ctx, query, u.ID, u.Name)
if isUniqueViolation(err) {
return fmt.Errorf("user %s: %w", u.ID, ErrConflict)
}
return err
}

// Typed error: caller uses errors.As() to extract data
// GOOD for: errors that carry actionable structured information                                                                                                                                                    
type RateLimitError struct {
RetryAfter time.Duration
Limit      int
Remaining  int
}

func (e *RateLimitError) Error() string {
return fmt.Sprintf("rate limited: retry after %v", e.RetryAfter)
}

func (c *Client) Call(ctx context.Context, req Request) (*Response, error) {
resp, err := c.http.Do(req)
if resp.StatusCode == 429 {
retryAfter, _ := time.ParseDuration(resp.Header.Get("Retry-After") + "s")
return nil, &RateLimitError{
RetryAfter: retryAfter,
Limit:      parseIntHeader(resp.Header, "X-RateLimit-Limit"),
Remaining:  0,
}
}                                                                                                                                                                                                               
// ...      
}

// Caller extracts retry timing from the error                                                                                                                                                                      
var rlErr *RateLimitError
if errors.As(err, &rlErr) {
time.Sleep(rlErr.RetryAfter)
return c.Call(ctx, req) // retry                                                                                                                                                                                
}

---
3. (value, bool) — The Comma-OK Idiom

When to use bool instead of error

// Use (T, bool) when:
//   absence is NORMAL and EXPECTED — not an exceptional condition                                                                                                                                                  
//   there's no additional context needed about why it's absent                                                                                                                                                     

// Cache lookup — miss is normal                                                                                                                                                                                    
func (c *Cache) Get(key string) (Value, bool) {
c.mu.RLock()
v, ok := c.data[key]
c.mu.RUnlock()
return v, ok
}

// Use (T, error) when:                                                                                                                                                                                             
//   absence is unexpected or you need to communicate WHY
//   the operation can fail in multiple distinct ways

// Database fetch — miss might be an error depending on caller's intent                                                                                                                                             
func (r *Repo) FindUser(ctx context.Context, id string) (*User, error) {
// ...                                                                                                                                                                                                          
}

The config/option lookup pattern

// Clean layered config: returns zero value + false when key absent
// Forces callers to think about defaults explicitly                                                                                                                                                                
type Config struct {
data map[string]string
}

func (c *Config) Get(key string) (string, bool) {
v, ok := c.data[key]
return v, ok
}

func (c *Config) GetOr(key, defaultVal string) string {
if v, ok := c.Get(key); ok {
return v
}
return defaultVal
}

// Usage is explicit about what happens when key is absent                                                                                                                                                          
timeout, ok := cfg.Get("timeout")
if !ok {
timeout = "30s"  // clearly intentional default
}

// vs the less explicit:                                                                                                                                                                                            
timeout := cfg.GetOr("timeout", "30s")

---
4. Named Return Values — Precision Use

When named returns add value

// USE CASE 1: defer modifies the error — canonical Go pattern
func (r *Repo) CreateUser(ctx context.Context, u User) (err error) {
defer func() {
if err != nil {
// Wraps ANY error that exits this function — including panics recovered elsewhere
err = fmt.Errorf("repo.CreateUser %s: %w", u.ID, err)
}                                                                                                                                                                                                           
}()

tx, err := r.db.BeginTx(ctx, nil)
if err != nil {
return  // naked return OK here — defer handles wrapping
}
defer tx.Rollback()

if err = r.insertUser(ctx, tx, u); err != nil {
return
}
if err = r.insertAuditLog(ctx, tx, u); err != nil {
return
}
return tx.Commit()
}

// USE CASE 2: documentation — named returns as parameter docs                                                                                                                                                      
// Communicates what each return value represents
func divide(a, b float64) (result float64, remainder float64, err error) {
if b == 0 {
err = errors.New("division by zero")
return  
}
result = math.Trunc(a / b)
remainder = a - result*b
return
}

When named returns are harmful

// BAD: naked returns in long functions — reader has to scroll back to find declarations
func processOrder(order Order) (total float64, tax float64, discount float64, err error) {
// ... 80 lines of logic ...                                                                                                                                                                                    
total = computeTotal()
tax = computeTax(total)
discount = applyPromotion(order)
return  // What is being returned? Reader must search for declarations
}

// GOOD: explicit returns make the code self-documenting                                                                                                                                                            
func processOrder(order Order) (float64, float64, float64, error) {
total := computeTotal()
tax := computeTax(total)
discount := applyPromotion(order)
return total, tax, discount, nil  // crystal clear                                                                                                                                                              
}

// BETTER: when you have more than 2-3 related returns, use a struct                                                                                                                                                
type OrderSummary struct {
Total    float64
Tax      float64
Discount float64
}

func processOrder(order Order) (OrderSummary, error) {
return OrderSummary{
Total:    computeTotal(),
Tax:      computeTax(computeTotal()),
Discount: applyPromotion(order),
}, nil
}

---
5. Return Structs for Multiple Related Values

The N > 2 rule

// BAD: 4 return values — impossible to use without named variables
func parseConfig(path string) (string, int, bool, error) {
// ...                                                                                                                                                                                                          
}

// Caller:                                                                                                                                                                                                          
host, port, tls, err := parseConfig("config.yaml")
// "host", "port", "tls" are just locally named — the function doesn't enforce their meaning                                                                                                                        

// GOOD: struct communicates meaning, enables partial use, extensible                                                                                                                                               
type ServerConfig struct {
Host    string
Port    int
TLS     bool
Timeout time.Duration  // can add fields without breaking callers                                                                                                                                               
}

func parseConfig(path string) (ServerConfig, error) {
// ...
}

// Caller can use only what they need
cfg, err := parseConfig("config.yaml")
fmt.Println(cfg.Host)  // intent is clear

Result structs enable future extensibility

// Today: only need data + error
// Later: need data + metadata + error + partial results                                                                                                                                                            

// BAD: if you start with multiple returns, adding fields is a breaking change                                                                                                                                      
func (db *DB) Query(ctx context.Context, q string) ([]Row, error) { ... }
// Changing to ([]Row, QueryMeta, error) breaks every caller                                                                                                                                                        

// GOOD: struct return is backward compatible — add fields without breaking callers                                                                                                                                 
type QueryResult struct {
Rows     []Row
// Added later — zero value means "no metadata" — callers that don't care, unaffected
Meta     QueryMeta
Stats    ExecutionStats
}

func (db *DB) Query(ctx context.Context, q string) (*QueryResult, error) { ... }

---
6. Return Concrete Types, Not Interfaces

The maxim: accept interfaces, return concrete types.

// BAD: returning interface boxes you into a contract
// What if you need to add a method later? Breaking change.                                                                                                                                                         
// What if callers need type-specific behavior? They have to assert.                                                                                                                                                
type Storage interface {
Get(key string) ([]byte, error)
Set(key string, val []byte) error
}

func NewStorage(cfg Config) Storage { // returns interface                                                                                                                                                          
return &s3Storage{...}
}

// Caller can't access S3-specific methods without type assertion:                                                                                                                                                  
s := NewStorage(cfg)
s3, ok := s.(*s3Storage)  // breaks abstraction, defeats the purpose                                                                                                                                                

// GOOD: return concrete type                                                                                                                                                                                       
// Callers can always assign to an interface themselves                                                                                                                                                             
func NewS3Storage(cfg Config) *S3Storage {
return &S3Storage{...}
}

// Caller assigns to interface when needed:                                                                                                                                                                         
var store Storage = NewS3Storage(cfg)  // caller's choice, not forced

When returning interfaces IS correct

// 1. error is always the interface — this is idiomatic Go
func (r *Repo) Get(id string) (*User, error) { ... }

// 2. The concrete type is private — you WANT to hide implementation                                                                                                                                                
func NewWriter(w io.Writer, format Format) io.Writer {
switch format {
case JSON:
return &jsonWriter{w: w}  // jsonWriter is unexported                                                                                                                                                       
case CSV:
return &csvWriter{w: w}   // csvWriter is unexported
}                                                                                                                                                                                                               
// Interface return is correct here — the type IS the abstraction
}

// 3. Explicit polymorphism at construction time                                                                                                                                                                    
// Multiple implementations, caller shouldn't know which they got
func NewCache(cfg CacheConfig) Cache {
if cfg.Distributed {
return newRedisCache(cfg)
}
return newLocalCache(cfg)
}

---
7. Functional Options — Returning Configured Objects

// The problem: constructors with many optional parameters
// NewServer(addr, timeout, maxConns, tls, logger, ...) — combinatorial explosion

type Server struct {
addr       string
timeout    time.Duration
maxConns   int
tls        *tls.Config
logger     *slog.Logger
}

type Option func(*Server) error  // error-returning option — catches bad config at construction                                                                                                                     

func WithTimeout(d time.Duration) Option {
return func(s *Server) error {
if d <= 0 {
return fmt.Errorf("timeout must be positive, got %v", d)
}
s.timeout = d
return nil
}           
}

func WithTLS(certFile, keyFile string) Option {
return func(s *Server) error {
cert, err := tls.LoadX509KeyPair(certFile, keyFile)
if err != nil {
return fmt.Errorf("load TLS cert: %w", err)
}
s.tls = &tls.Config{Certificates: []tls.Certificate{cert}}
return nil
}           
}

func NewServer(addr string, opts ...Option) (*Server, error) {
s := &Server{
addr:    addr,
timeout: 30 * time.Second,  // sane defaults
maxConns: 1000,
}
for _, opt := range opts {
if err := opt(s); err != nil {
return nil, fmt.Errorf("server option: %w", err)
}                                                                                                                                                                                                           
}
return s, nil
}

// Clean call site — only specify what differs from defaults                                                                                                                                                        
srv, err := NewServer(":8080",
WithTimeout(10 * time.Second),
WithTLS("/etc/certs/cert.pem", "/etc/certs/key.pem"),
)

---
8. Builder Pattern with Sticky Errors

For chaining operations where any step can fail:

// Sticky error: first error is remembered, subsequent operations are no-ops
// Result: clean chained API without if-err-return at every step                                                                                                                                                    
type QueryBuilder struct {
table   string
columns []string
wheres  []string
limit   int
err     error  // sticky — once set, stays set                                                                                                                                                                  
}

func NewQuery(table string) *QueryBuilder {
return &QueryBuilder{table: table}
}

func (b *QueryBuilder) Select(cols ...string) *QueryBuilder {
if b.err != nil {
return b  // already failed — no-op
}
if len(cols) == 0 {
b.err = errors.New("Select: at least one column required")
return b
}
b.columns = append(b.columns, cols...)
return b
}

func (b *QueryBuilder) Where(condition string) *QueryBuilder {
if b.err != nil {
return b
}
if condition == "" {
b.err = errors.New("Where: empty condition")
return b
}
b.wheres = append(b.wheres, condition)
return b
}

func (b *QueryBuilder) Limit(n int) *QueryBuilder {
if b.err != nil {
return b
}
if n <= 0 {
b.err = fmt.Errorf("Limit: must be positive, got %d", n)
return b
}
b.limit = n
return b
}

// Terminal method — returns the final result and surfaces any accumulated error                                                                                                                                    
func (b *QueryBuilder) Build() (string, error) {
if b.err != nil {
return "", b.err
}
// construct and return query string
}

// Usage: error handling deferred to the end                                                                                                                                                                        
query, err := NewQuery("users").
Select("id", "name", "email").
Where("active = true").
Where("created_at > '2026-01-01'").
Limit(100).
Build()  // ← only place you handle errors

---
9. Iterator Returns — Go 1.23 iter.Seq

For streaming large result sets without materializing into a slice:

import "iter"

// iter.Seq[V]: yield func(V) bool — single value                                                                                                                                                                   
// iter.Seq2[K, V]: yield func(K, V) bool — key/value pair

// Return iter.Seq2 for fallible sequences (value + error)                                                                                                                                                          
func (r *Repo) AllUsers(ctx context.Context) iter.Seq2[*User, error] {
return func(yield func(*User, error) bool) {
rows, err := r.db.QueryContext(ctx, "SELECT id, name, email FROM users")
if err != nil {
yield(nil, fmt.Errorf("query: %w", err))
return
}
defer rows.Close()

for rows.Next() {
var u User
if err := rows.Scan(&u.ID, &u.Name, &u.Email); err != nil {
yield(nil, fmt.Errorf("scan: %w", err))
return
}
if !yield(&u, nil) {
return  // caller broke out of loop — stop producing                                                                                                                                                
}                                                                                                                                                                                                       
}
if err := rows.Err(); err != nil {
yield(nil, fmt.Errorf("rows: %w", err))
}
}
}

// Caller: familiar range syntax, lazy evaluation, early exit works                                                                                                                                                 
for user, err := range repo.AllUsers(ctx) {
if err != nil {
return fmt.Errorf("iterate users: %w", err)
}
if user.Name == "admin" {
break  // early exit — stops the DB cursor, no goroutine leak                                                                                                                                               
}
process(user)
}

Why this beats returning []User or <-chan *User:
[]User:       materializes entire result set in memory — bad for large datasets
<-chan *User: goroutine leak if caller doesn't drain, awkward error propagation                                                                                                                                     
iter.Seq2:    lazy, zero allocation, early exit safe, error propagation natural

---
10. The Must Pattern — Initialization vs Runtime

// Must: panics on error — ONLY for initialization-time operations
// where failure means the program cannot possibly function                                                                                                                                                         

// Template parsing at startup — if this fails, the server is broken                                                                                                                                                
var userTemplate = template.Must(template.ParseFiles("templates/user.html"))

// Regex compilation — if the regex is invalid, the code is wrong, not the input
var emailRegex = regexp.MustCompile(`^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$`)

// Your own Must wrapper                                                                                                                                                                                            
func MustParseConfig(path string) *Config {
cfg, err := ParseConfig(path)
if err != nil {
panic(fmt.Sprintf("fatal: cannot parse config %s: %v", path, err))
}
return cfg
}

// NEVER use Must in:
//   - Request handlers
//   - Functions called at runtime with user input                                                                                                                                                                  
//   - Library code (let the caller decide how to handle errors)
//   - Anything that runs more than once                                                                                                                                                                            

// Package-level var: OK — happens once at startup                                                                                                                                                                  
var defaultConfig = MustParseConfig("/etc/app/config.yaml")

// In a handler: NEVER
func (h *Handler) GetUser(w http.ResponseWriter, r *http.Request) {
cfg := MustParseConfig(r.Header.Get("Config-Path")) // panics on bad input — catastrophic                                                                                                                       
}

---
11. All-or-Nothing State Mutation

// BAD: error return leaves object in partially modified state
// Object invariants violated — subsequent calls on this object are undefined                                                                                                                                       
func (s *ShoppingCart) AddItems(items []Item) error {
for _, item := range items {
s.items = append(s.items, item)     // state modified
s.total += item.Price               // state modified                                                                                                                                                       
if err := s.validateStock(item); err != nil {
return err  // s.items and s.total are now inconsistent
}                                                                                                                                                                                                           
}
return nil
}

// GOOD: validate first, mutate only on full success                                                                                                                                                                
func (s *ShoppingCart) AddItems(items []Item) error {
// Phase 1: validate everything — no state mutation                                                                                                                                                             
for _, item := range items {
if err := s.validateStock(item); err != nil {
return fmt.Errorf("item %s: %w", item.ID, err)
}                                                                                                                                                                                                           
}

// Phase 2: apply changes — guaranteed to succeed
for _, item := range items {
s.items = append(s.items, item)
s.total += item.Price                                                                                                                                                                                       
}
return nil
}

// Alternative: copy-on-write — return new state, never mutate receiver                                                                                                                                             
func (s ShoppingCart) WithItems(items []Item) (ShoppingCart, error) {
for _, item := range items {
if err := s.validateStock(item); err != nil {
return ShoppingCart{}, fmt.Errorf("item %s: %w", item.ID, err)
}
s.items = append(s.items[:len(s.items):len(s.items)], item)  // copy before append
s.total += item.Price
}
return s, nil
}
// Original cart is never touched — caller atomically swaps on success

---
12. Generic Result Type — When to Use

// Useful for: APIs that return homogeneous results across multiple operations
// Not idiomatic everywhere — use judiciously                                                                                                                                                                       
type Result[T any] struct {
Value T
Err   error
}

func OK[T any](v T) Result[T]        { return Result[T]{Value: v} }
func Fail[T any](err error) Result[T] { return Result[T]{Err: err} }

func (r Result[T]) Unwrap() (T, error) { return r.Value, r.Err }

func (r Result[T]) Map(f func(T) T) Result[T] {
if r.Err != nil {
return r
}
return OK(f(r.Value))
}

// Best fit: batch operations returning mixed success/failure                                                                                                                                                       
func fetchAll(ctx context.Context, ids []string) []Result[*User] {
results := make([]Result[*User], len(ids))
var wg sync.WaitGroup
for i, id := range ids {
i, id := i, id
wg.Add(1)
go func() {
defer wg.Done()
user, err := fetchUser(ctx, id)
if err != nil {
results[i] = Fail[*User](err)
} else {
results[i] = OK(user)
}
}()
}
wg.Wait()
return results
}

// Caller handles success and failure per-item
for i, result := range fetchAll(ctx, ids) {
user, err := result.Unwrap()
if err != nil {
log.Printf("id %s failed: %v", ids[i], err)
continue
}
process(user)
}

---
Summary: Decision Reference

What to return          When
──────────────────────────────────────────────────────────────────
(T, error)              Any fallible operation — the default choice
(T, bool)               Lookup where absence is normal (cache, map, optional config)
(*T, error)             Large struct, nil is meaningful, or identity matters
Struct result           > 2 related return values, or API needs future extensibility
Named returns           Only when defer modifies return value — avoid naked returns otherwise
iter.Seq2[T, error]     Streaming/lazy sequences (Go 1.23+)
Functional options      Constructors with optional configuration
Builder + sticky error  Chainable APIs with validation
Must(T)                 Package init / test setup only — never in request path
Concrete type           Default — not interface (except error, io.Reader/Writer, private types)

The overarching staff-level principle: return types are API contracts, not implementation details. Every return type decision constrains your callers and your future self. Design them with the same care you'd    
give a public REST API — once callers depend on them, changing them is expensive.   