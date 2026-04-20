⏺ Avoiding Race Conditions in Go
                                                                                                                                                                                                               
---                                                                                                                                                                                                            
First: Detect Before You Fix

# Run with the race detector — catches races at runtime
go test -race ./...                                                                                                                                                                                            
go run -race main.go
go build -race -o myapp && ./myapp

# The detector instruments memory accesses and reports:
# WARNING: DATA RACE
# Write at 0x... by goroutine N:
#   main.increment() /app/main.go:12
# Previous read at 0x... by goroutine M:
#   main.read() /app/main.go:20

The race detector has ~10x memory overhead and ~2-20x CPU overhead — use in tests and staging, not production.
                  
---                                                                                                                                                                                                            
The Go Memory Model — Foundational Rule

Before any pattern, understand what Go guarantees. Two goroutines accessing the same variable without synchronization is undefined behavior — even if one is only reading.

// INVISIBLE BUG: no synchronization — goroutine may never see x = 42
var x int                                                                                                                                                                                                      
go func() {     
fmt.Println(x) // may print 0, 42, or anything — undefined                                                                                                                                                 
}()                                                                                                                                                                                                            
x = 42

// The Go memory model only guarantees visibility through:                                                                                                                                                     
//   - Channel send/receive
//   - sync.Mutex Lock/Unlock                                                                                                                                                                                  
//   - sync.Once.Do
//   - sync/atomic operations                                                                                                                                                                                  
//   - Goroutine start (but NOT goroutine completion without Wait/channel)
                                                                                                                                                                                                                 
---             
1. sync.Mutex — The Standard Workhorse

// BAD: concurrent read-modify-write on a plain int
var counter int                                                                                                                                                                                                
go func() { counter++ }()  // load, add, store — not atomic
go func() { counter++ }()  // races with first goroutine

// GOOD: mutex protects the critical section
type SafeCounter struct {                                                                                                                                                                                      
mu    sync.Mutex
value int                                                                                                                                                                                                  
}

func (c *SafeCounter) Increment() {
c.mu.Lock()
defer c.mu.Unlock() // always defer — prevents leak if panic occurs
c.value++                                                                                                                                                                                                  
}

func (c *SafeCounter) Value() int {                                                                                                                                                                            
c.mu.Lock()
defer c.mu.Unlock()                                                                                                                                                                                        
return c.value
}

Keep critical sections minimal

// BAD: holding lock during I/O blocks all other goroutines unnecessarily
func (s *Store) Save(key string, val any) error {                                                                                                                                                              
s.mu.Lock()                                                                                                                                                                                                
defer s.mu.Unlock()

      data, err := json.Marshal(val)  // CPU-bound — OK
      if err != nil {                                                                                                                                                                                            
          return err
      }
      return os.WriteFile(key, data, 0644)  // I/O — BAD inside lock                                                                                                                                             
}

// GOOD: lock only protects the shared state, not the I/O                                                                                                                                                      
func (s *Store) Save(key string, val any) error {
data, err := json.Marshal(val)  // outside lock                                                                                                                                                            
if err != nil {                                                                                                                                                                                            
return err
}

      s.mu.Lock()
      s.pending[key] = data  // only the map write is protected
      s.mu.Unlock()                                                                                                                                                                                              
  
      return s.flush(key, data) // I/O outside lock                                                                                                                                                              
}

sync.RWMutex — Read-heavy workloads

// Multiple goroutines can hold RLock simultaneously                                                                                                                                                           
// Only one goroutine can hold Lock (write lock)

type Cache struct {
mu    sync.RWMutex                                                                                                                                                                                         
items map[string][]byte                                                                                                                                                                                    
}

func (c *Cache) Get(key string) ([]byte, bool) {
c.mu.RLock()            // multiple readers can proceed concurrently
defer c.mu.RUnlock()                                                                                                                                                                                       
v, ok := c.items[key]
return v, ok                                                                                                                                                                                               
}

func (c *Cache) Set(key string, val []byte) {
c.mu.Lock()             // exclusive — all readers blocked during write
defer c.mu.Unlock()                                                                                                                                                                                        
c.items[key] = val
}

// When NOT to use RWMutex:                                                                                                                                                                                    
// If writes are frequent, RWMutex has more overhead than Mutex
// Profile first — RWMutex only helps when reads >> writes (> 5:1)
                                                                                                                                                                                                                 
---
2. sync/atomic — Lock-Free for Simple Types

import "sync/atomic"

// Atomic operations: single machine instruction, no lock overhead                                                                                                                                             
var counter int64

// Increment atomically
atomic.AddInt64(&counter, 1)

// Read atomically — even reads need to be atomic if writes are concurrent                                                                                                                                     
n := atomic.LoadInt64(&counter)

// Compare-and-swap — optimistic locking pattern
func (b *Bucket) TryIncrement(max int64) bool {                                                                                                                                                                
for {                                                                                                                                                                                                      
current := atomic.LoadInt64(&b.count)
if current >= max {                                                                                                                                                                                    
return false
}
// Only swap if value hasn't changed since we read it                                                                                                                                                  
if atomic.CompareAndSwapInt64(&b.count, current, current+1) {
return true                                                                                                                                                                                        
}       
// Someone else changed it — retry                                                                                                                                                                     
}           
}

atomic.Value — for swapping any value atomically

// Ideal for: config hot-reload, routing tables, feature flags                                                                                                                                                 
// Readers get a consistent snapshot; writers swap the whole pointer

type Server struct {                                                                                                                                                                                           
config atomic.Value  // stores *Config — never mutate the stored value                                                                                                                                     
}

func (s *Server) ReloadConfig(newCfg *Config) {                                                                                                                                                                
s.config.Store(newCfg)  // atomic pointer swap — all future readers see new config
}

func (s *Server) Handle(w http.ResponseWriter, r *http.Request) {                                                                                                                                              
cfg := s.config.Load().(*Config)  // atomic read — consistent snapshot
// use cfg — safe even if ReloadConfig is called concurrently                                                                                                                                              
// NEVER mutate cfg here — it's shared                                                                                                                                                                     
}

atomic.Pointer (Go 1.19+) — typed atomic pointer

// Type-safe alternative to atomic.Value when storing pointers                                                                                                                                                 
var current atomic.Pointer[RuleSet]

// Writer                                                                                                                                                                                                      
current.Store(&RuleSet{rules: newRules})

// Reader — gets a consistent snapshot
rs := current.Load()
rs.Evaluate(request)
  
---                                                                                                                                                                                                            
3. Channels — Share by Communicating

The CSP approach: instead of sharing state and locking it, give ownership of state to one goroutine and communicate via channels.

// Single-owner pattern: one goroutine owns all state
// others communicate via channels — no mutexes needed                                                                                                                                                         
type RateLimiter struct {                                                                                                                                                                                      
tokens  int
refill  <-chan time.Time                                                                                                                                                                                   
allow   chan chan bool  // request channel: send a reply channel                                                                                                                                           
stop    chan struct{}                                                                                                                                                                                      
}

func NewRateLimiter(rate int, per time.Duration) *RateLimiter {                                                                                                                                                
rl := &RateLimiter{
tokens: rate,
refill: time.NewTicker(per).C,                                                                                                                                                                         
allow:  make(chan chan bool, 10),
stop:   make(chan struct{}),                                                                                                                                                                           
}           
go rl.run(rate) // single goroutine owns `tokens`
return rl                                                                                                                                                                                                  
}

func (rl *RateLimiter) run(maxTokens int) {                                                                                                                                                                    
for {
select {                                                                                                                                                                                               
case reply := <-rl.allow:
if rl.tokens > 0 {
rl.tokens--                                                                                                                                                                                    
reply <- true
} else {                                                                                                                                                                                           
reply <- false
}
case <-rl.refill:
rl.tokens = maxTokens // safe: only this goroutine touches tokens
case <-rl.stop:                                                                                                                                                                                        
return
}                                                                                                                                                                                                      
}           
}

func (rl *RateLimiter) Allow() bool {                                                                                                                                                                          
reply := make(chan bool, 1)
rl.allow <- reply                                                                                                                                                                                          
return <-reply
}

Channel ownership rules

// Rule: only the SENDER closes a channel
// If multiple senders exist, use sync.Once or a coordinator

// BAD: multiple goroutines closing the same channel panics                                                                                                                                                    
results := make(chan int)                                                                                                                                                                                      
go func() { results <- 1; close(results) }()  // may panic                                                                                                                                                     
go func() { results <- 2; close(results) }()  // may panic

// GOOD: single closer via WaitGroup + coordinator goroutine
results := make(chan int)                                                                                                                                                                                      
var wg sync.WaitGroup

for i := 0; i < 5; i++ {                                                                                                                                                                                       
wg.Add(1)   
go func(i int) {
defer wg.Done()
results <- i  // only send, never close                                                                                                                                                                
}(i)
}

// Coordinator: close only after all senders finish                                                                                                                                                            
go func() {
wg.Wait()                                                                                                                                                                                                  
close(results)  // single closer
}()

for v := range results {                                                                                                                                                                                       
fmt.Println(v)
}

  ---
4. Common Pitfalls and Fixes

Pitfall 1: Loop variable capture in goroutines

// BAD (Go < 1.22): all goroutines capture the same variable `i`
for i := 0; i < 5; i++ {                                                                                                                                                                                       
go func() {
fmt.Println(i) // i is shared — likely prints 5,5,5,5,5                                                                                                                                                
}()                                                                                                                                                                                                        
}

// FIX A: shadow with a new variable (works in all Go versions)                                                                                                                                                
for i := 0; i < 5; i++ {
i := i  // new `i` per iteration                                                                                                                                                                           
go func() {
fmt.Println(i)                                                                                                                                                                                         
}()         
}

// FIX B: pass as argument                                                                                                                                                                                     
for i := 0; i < 5; i++ {
go func(n int) {                                                                                                                                                                                           
fmt.Println(n)
}(i)
}

// Note: Go 1.22+ loop variables are per-iteration by default                                                                                                                                                  
// FIX B is still the clearest and most portable

Pitfall 2: Map concurrent access

// BAD: maps are not safe for concurrent use                                                                                                                                                                   
m := make(map[string]int)
go func() { m["a"]++ }()  // write
go func() { _ = m["a"] }()  // read — DATA RACE, can corrupt or panic

// FIX A: sync.Mutex                                                                                                                                                                                           
type SafeMap struct {                                                                                                                                                                                          
mu sync.RWMutex                                                                                                                                                                                            
m  map[string]int
}
func (s *SafeMap) Inc(key string) {                                                                                                                                                                            
s.mu.Lock()
defer s.mu.Unlock()                                                                                                                                                                                        
s.m[key]++  
}

// FIX B: sync.Map (for mostly-read, stable key-set patterns)                                                                                                                                                  
var sm sync.Map
sm.Store("a", 1)                                                                                                                                                                                               
sm.LoadOrStore("a", 0)  // atomic: load if exists, store if not
sm.Range(func(k, v any) bool {                                                                                                                                                                                 
fmt.Println(k, v)                                                                                                                                                                                          
return true                                                                                                                                                                                                
})

Pitfall 3: Defer inside a loop

// BAD: defer runs when the FUNCTION returns, not when the loop iteration ends
// The lock is held for ALL iterations, not released per iteration                                                                                                                                             
func processAll(items []Item) {                                                                                                                                                                                
for _, item := range items {                                                                                                                                                                               
mu.Lock()                                                                                                                                                                                              
defer mu.Unlock()  // releases at end of processAll(), not end of iteration                                                                                                                            
process(item)                                                                                                                                                                                          
}
}

// FIX A: explicit unlock
for _, item := range items {
mu.Lock()
process(item)
mu.Unlock()  // released per iteration
}

// FIX B: extract to method (defer now scoped correctly)                                                                                                                                                       
for _, item := range items {
processOne(item)                                                                                                                                                                                           
}               
func processOne(item Item) {
mu.Lock()                                                                                                                                                                                                  
defer mu.Unlock()  // scoped to processOne
process(item)                                                                                                                                                                                              
}

Pitfall 4: Copying a mutex

// BAD: copying a struct that contains a mutex copies the lock state                                                                                                                                           
// go vet detects this, but it's easy to miss                                                                                                                                                                  
type Cache struct {
mu    sync.Mutex                                                                                                                                                                                           
items map[string]int                                                                                                                                                                                       
}

func NewCache(src Cache) *Cache {
return &src  // copies Cache — copies the mutex — undefined behavior
}

// FIX: use pointer receivers and pointer passing                                                                                                                                                              
func copyCache(src *Cache) *Cache {
src.mu.Lock()                                                                                                                                                                                              
defer src.mu.Unlock()
newCache := &Cache{items: make(map[string]int)}                                                                                                                                                            
for k, v := range src.items {
newCache.items[k] = v                                                                                                                                                                                  
}
return newCache                                                                                                                                                                                            
}

Pitfall 5: Check-then-act (TOCTOU)

// BAD: state can change between the check and the action
func (c *Cache) GetOrLoad(key string) *Value {                                                                                                                                                                 
c.mu.RLock()                                                                                                                                                                                               
if v, ok := c.items[key]; ok {                                                                                                                                                                             
c.mu.RUnlock()                                                                                                                                                                                         
return v  // OK path
}                                                                                                                                                                                                          
c.mu.RUnlock()

      // RACE WINDOW: between RUnlock and Lock below,                                                                                                                                                            
      // another goroutine may load and store the same key
      v := loadFromDB(key)                                                                                                                                                                                       
                  
      c.mu.Lock()                                                                                                                                                                                                
      c.items[key] = v  // may overwrite another goroutine's result
      c.mu.Unlock()                                                                                                                                                                                              
      return v
}

// FIX: hold lock across the entire check-and-act
// But don't hold lock during the DB call — use singleflight instead
var group singleflight.Group

func (c *Cache) GetOrLoad(key string) (*Value, error) {                                                                                                                                                        
c.mu.RLock()
if v, ok := c.items[key]; ok {                                                                                                                                                                             
c.mu.RUnlock()
return v, nil
}
c.mu.RUnlock()

      // singleflight: only one goroutine loads for this key                                                                                                                                                     
      // all concurrent misses wait and share the result
      v, err, _ := group.Do(key, func() (any, error) {                                                                                                                                                           
          val, err := loadFromDB(key)
          if err != nil {                                                                                                                                                                                        
              return nil, err                                                                                                                                                                                    
          }
          c.mu.Lock()                                                                                                                                                                                            
          c.items[key] = val
          c.mu.Unlock()
          return val, nil
      })
      return v.(*Value), err
}

Pitfall 6: WaitGroup.Add inside the goroutine

// BAD: Add might be called after Wait() already returned
var wg sync.WaitGroup                                                                                                                                                                                          
for i := 0; i < 5; i++ {
go func() {                                                                                                                                                                                                
wg.Add(1)     // too late — wg.Wait() may have already returned                                                                                                                                        
defer wg.Done()
work()                                                                                                                                                                                                 
}()         
}                                                                                                                                                                                                              
wg.Wait()  // may return before goroutines even started

// GOOD: Add before the goroutine starts                                                                                                                                                                       
for i := 0; i < 5; i++ {
wg.Add(1)         // happens-before the goroutine runs                                                                                                                                                     
go func() {
defer wg.Done()
work()                                                                                                                                                                                                 
}()
}                                                                                                                                                                                                              
wg.Wait()

Pitfall 7: Goroutine outliving its data

// BAD: goroutine references stack variable that may no longer exist                                                                                                                                           
func handler(w http.ResponseWriter, r *http.Request) {                                                                                                                                                         
result := make([]byte, 0)  // local variable

      go func() { 
          // This goroutine may outlive the handler function                                                                                                                                                     
          // After handler returns, result is gone (or reused)
          result = append(result, fetchData()...)  // DATA RACE                                                                                                                                                  
      }()                                                                                                                                                                                                        
                                                                                                                                                                                                                 
      // handler returns — result stack frame may be freed                                                                                                                                                       
}

// FIX: ensure goroutine lifetime is bounded by the function
func handler(w http.ResponseWriter, r *http.Request) {
var wg sync.WaitGroup                                                                                                                                                                                      
result := make([]byte, 0)

      wg.Add(1)   
      go func() {
          defer wg.Done()
          // goroutine lifetime bounded by wg.Wait()                                                                                                                                                             
          data := fetchData()
          mu.Lock()                                                                                                                                                                                              
          result = append(result, data...)
          mu.Unlock()                                                                                                                                                                                            
      }()
                                                                                                                                                                                                                 
      wg.Wait()  // handler blocks until goroutine completes
      w.Write(result)
}
  
---                                                                                                                                                                                                            
5. sync.Once — Safe One-Time Initialization

// BAD: double-checked locking is broken in Go without sync.Once
var (                                                                                                                                                                                                          
instance *DB
mu       sync.Mutex                                                                                                                                                                                        
)

func GetDB() *DB {
if instance == nil {   // first check — not synchronized
mu.Lock()                                                                                                                                                                                              
if instance == nil {
instance = initDB()  // write                                                                                                                                                                      
}                                                                                                                                                                                                      
mu.Unlock()
}                                                                                                                                                                                                          
return instance  // read of instance outside lock — DATA RACE
}

// GOOD: sync.Once guarantees exactly-once execution safely                                                                                                                                                    
var (           
db   *DB                                                                                                                                                                                                   
once sync.Once
)

func GetDB() *DB {
once.Do(func() {
db = initDB()  // runs exactly once, all callers block until done                                                                                                                                      
})
return db  // safe: once.Do establishes happens-before with all callers                                                                                                                                    
}
  
---                                                                                                                                                                                                            
6. Copy-on-Write — High Read Concurrency Without Lock Contention

// For data that is read constantly but written rarely
// Readers get a snapshot — no lock needed during read

type RouteTable struct {                                                                                                                                                                                       
current atomic.Pointer[map[string]string]                                                                                                                                                                  
}

func NewRouteTable() *RouteTable {                                                                                                                                                                             
rt := &RouteTable{}
empty := make(map[string]string)                                                                                                                                                                           
rt.current.Store(&empty)
return rt                                                                                                                                                                                                  
}

func (rt *RouteTable) Lookup(host string) string {                                                                                                                                                             
routes := *rt.current.Load()  // atomic pointer load — zero lock overhead
return routes[host]            // read from immutable snapshot — no race                                                                                                                                   
}

func (rt *RouteTable) Update(newRoutes map[string]string) {                                                                                                                                                    
// Build new map — don't mutate the existing one (readers may be using it)
copy := make(map[string]string, len(newRoutes))                                                                                                                                                            
for k, v := range newRoutes {                                                                                                                                                                              
copy[k] = v                                                                                                                                                                                            
}                                                                                                                                                                                                          
rt.current.Store(&copy)  // atomic swap — readers instantly see new routes
// Old map is garbage collected once existing readers finish with it                                                                                                                                       
}
                                                                                                                                                                                                                 
---                                                                                                                                                                                                            
7. Testing for Race Conditions

// Use -race in all CI test runs
// Also: write tests that deliberately exercise concurrent access

func TestCounter_ConcurrentIncrement(t *testing.T) {                                                                                                                                                           
c := NewSafeCounter()                                                                                                                                                                                      
var wg sync.WaitGroup                                                                                                                                                                                      
const goroutines = 1000

      for i := 0; i < goroutines; i++ {
          wg.Add(1)                                                                                                                                                                                              
          go func() {
              defer wg.Done()
              c.Increment()
          }()
      }

      wg.Wait()                                                                                                                                                                                                  
  
      if got := c.Value(); got != goroutines {                                                                                                                                                                   
          t.Errorf("expected %d, got %d", goroutines, got)
      }
}
// Run: go test -race -count=100 ./...
// -count=100: run each test 100 times — races are non-deterministic
                                                                                                                                                                                                                 
---                                                                                                                                                                                                            
Decision Guide

Situation                                  Solution
────────────────────────────────────────────────────────────────────────
Simple counter, flag, pointer swap         sync/atomic (AddInt64, CompareAndSwap,                                                                                                                              
atomic.Value, atomic.Pointer)

Struct with multiple related fields        sync.Mutex (or RWMutex if reads >> writes)                                                                                                                          
that must be updated together            Embed mutex in struct

Read-heavy, rarely written config          atomic.Value or copy-on-write pattern

Concurrent map with stable key set        sync.Map

Concurrent map with changing key set      sync.Mutex + map[K]V

One-time initialization                   sync.Once

Ownership naturally belongs to one        Channel-based single owner                                                                                                                                           
goroutine (event loop, worker)

Cache miss causing stampede               singleflight.Group

Fan-out/fan-in coordination               sync.WaitGroup

Gradual feature flag / hot config reload  atomic.Pointer[Config] + copy-on-write

Loop variable capture in goroutine        Pass as argument: go func(i int){}(i)

Closing channel from multiple goroutines  sync.Once for close, or coordinator goroutine

The guiding rules:
1. If data is accessed by multiple goroutines and at least one writes → synchronize
2. Reads need synchronization too if any concurrent write exists
3. Keep critical sections as small as possible — no I/O, no heavy computation
4. Never copy a sync.Mutex (use pointers)
5. WaitGroup.Add must happen before the goroutine starts
6. defer mu.Unlock() in loops releases at function return, not loop end
7. Channels own their senders — only sender (or coordinator) closes
8. Always run tests with -race in CI        