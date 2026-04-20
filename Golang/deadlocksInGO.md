Avoiding Deadlocks with Goroutines in Go
                                                                                                                                                                                                                 
---                                                                                                                                                                                                            
What Causes a Deadlock

A deadlock requires all four conditions simultaneously:

1. Mutual Exclusion    — a resource is held by one goroutine at a time
2. Hold and Wait       — a goroutine holds a resource while waiting for another
3. No Preemption       — resources cannot be forcibly taken away
4. Circular Wait       — G1 waits for G2, G2 waits for G1 (cycle)

Break ANY one condition → no deadlock

Go's runtime detects complete deadlocks (all goroutines blocked) and panics:                                                                                                                                   
fatal error: all goroutines are asleep - deadlock!

Partial deadlocks (some goroutines stuck, others running) are harder — they manifest as hangs, goroutine leaks, and degraded throughput. The runtime won't catch these.
                                                                                                                                                                                                                 
---                                                                                                                                                                                                            
Category 1: Mutex Deadlocks

Lock ordering violation — the most common mutex deadlock

var mu1, mu2 sync.Mutex

// DEADLOCK:    
// Goroutine A: acquires mu1, waits for mu2                                                                                                                                                                    
// Goroutine B: acquires mu2, waits for mu1
// Circular wait — neither can proceed

go func() {                                                                                                                                                                                                    
mu1.Lock()  
time.Sleep(1 * time.Millisecond) // makes race more likely to manifest                                                                                                                                     
mu2.Lock()   // waits for B to release mu2                                                                                                                                                                 
mu2.Unlock()                                                                                                                                                                                               
mu1.Unlock()                                                                                                                                                                                               
}()

go func() {     
mu2.Lock()
time.Sleep(1 * time.Millisecond)
mu1.Lock()   // waits for A to release mu1
mu1.Unlock()                                                                                                                                                                                               
mu2.Unlock()
}()

// FIX: always acquire locks in the same global order
// Define the order once and enforce it everywhere

// Convention: mu1 always before mu2 — document this                                                                                                                                                           
go func() {     
mu1.Lock(); mu2.Lock()                                                                                                                                                                                     
mu2.Unlock(); mu1.Unlock()                                                                                                                                                                                 
}()

go func() {     
mu1.Lock(); mu2.Lock()  // same order — no circular wait
mu2.Unlock(); mu1.Unlock()                                                                                                                                                                                 
}()

Resource hierarchy — lock ordering for dynamic resources

// Classic bank transfer: two accounts, need both locks                                                                                                                                                        
// If transfer(A→B) and transfer(B→A) run concurrently — deadlock

type Account struct {                                                                                                                                                                                          
id      int64                                                                                                                                                                                              
mu      sync.Mutex
balance decimal.Decimal
}

// BAD: lock order depends on caller's argument order                                                                                                                                                          
func transfer(from, to *Account, amount decimal.Decimal) {
from.mu.Lock()   // transfer(A,B): locks A first                                                                                                                                                           
to.mu.Lock()     // transfer(B,A): locks B first — DEADLOCK with above                                                                                                                                     
from.balance = from.balance.Sub(amount)
to.balance = to.balance.Add(amount)                                                                                                                                                                        
to.mu.Unlock()
from.mu.Unlock()                                                                                                                                                                                           
}

// GOOD: always lock in order of account ID (stable global ordering)                                                                                                                                           
func transfer(from, to *Account, amount decimal.Decimal) {
// Determine which to lock first using a stable, global criterion                                                                                                                                          
first, second := from, to                                                                                                                                                                                  
if from.id > to.id {
first, second = to, from                                                                                                                                                                               
}

      first.mu.Lock()
      defer first.mu.Unlock()
      second.mu.Lock()
      defer second.mu.Unlock()                                                                                                                                                                                   
  
      // Now safe regardless of argument order                                                                                                                                                                   
      from.balance = from.balance.Sub(amount)
      to.balance = to.balance.Add(amount)                                                                                                                                                                        
}

Recursive locking — Go's mutex is NOT reentrant

// DEADLOCK: same goroutine tries to lock an already-held mutex
type Service struct {                                                                                                                                                                                          
mu    sync.Mutex
cache map[string]string                                                                                                                                                                                    
}

func (s *Service) Get(key string) string {
s.mu.Lock()
defer s.mu.Unlock()
return s.getInternal(key) // calls a method that also locks s.mu
}

func (s *Service) getInternal(key string) string {                                                                                                                                                             
s.mu.Lock()   // DEADLOCK: same goroutine, mu already held
defer s.mu.Unlock()                                                                                                                                                                                        
return s.cache[key]                                                                                                                                                                                        
}

// FIX: split into public (locks) and private (assumes lock held) methods
func (s *Service) Get(key string) string {                                                                                                                                                                     
s.mu.Lock()
defer s.mu.Unlock()                                                                                                                                                                                        
return s.locked_get(key)  // naming convention signals lock requirement
}

func (s *Service) locked_get(key string) string {                                                                                                                                                              
// Caller is responsible for holding s.mu
// No lock acquisition here                                                                                                                                                                                
return s.cache[key]
}

// This is a common pattern: public methods acquire lock,                                                                                                                                                      
// internal *Locked methods assume the lock is already held

Holding a lock while calling external code

// DEADLOCK: callback tries to acquire the same lock you're holding
type EventBus struct {                                                                                                                                                                                         
mu        sync.Mutex
listeners []func(string)                                                                                                                                                                                   
}

func (b *EventBus) Emit(event string) {
b.mu.Lock()
defer b.mu.Unlock()

      for _, fn := range b.listeners {
          fn(event)  // listener may call b.Subscribe() or b.Emit() — DEADLOCK                                                                                                                                   
      }                                                                                                                                                                                                          
}

func (b *EventBus) Subscribe(fn func(string)) {                                                                                                                                                                
b.mu.Lock()  // blocked if Emit is calling us via callback
defer b.mu.Unlock()                                                                                                                                                                                        
b.listeners = append(b.listeners, fn)                                                                                                                                                                      
}

// FIX: copy the slice, release the lock, then call callbacks
func (b *EventBus) Emit(event string) {                                                                                                                                                                        
b.mu.Lock()
// Take a snapshot — safe because slices are copied by value (header only)                                                                                                                                 
// but we copy the underlying slice to avoid races on append                                                                                                                                               
listeners := make([]func(string), len(b.listeners))                                                                                                                                                        
copy(listeners, b.listeners)                                                                                                                                                                               
b.mu.Unlock()  // release BEFORE calling external code

      for _, fn := range listeners {
          fn(event)  // lock not held — callbacks can safely call back into EventBus
      }                                                                                                                                                                                                          
}
                                                                                                                                                                                                                 
---                                                                                                                                                                                                            
Category 2: Channel Deadlocks

Unbuffered channel with no concurrent receiver

// DEADLOCK: send to unbuffered channel blocks until a receiver is ready
// If send and receive are in the same goroutine — deadlock                                                                                                                                                    
ch := make(chan int)                                                                                                                                                                                           
ch <- 42    // blocks forever — this goroutine can't also receive                                                                                                                                              
fmt.Println(<-ch)

// DEADLOCK variant: goroutine sends, but the receiver never starts                                                                                                                                            
ch := make(chan int)                                                                                                                                                                                           
go func() {                                                                                                                                                                                                    
ch <- 42  // starts goroutine, but main returns before receiving
}()                                                                                                                                                                                                            
// main exits — goroutine leaks

// FIX A: receiver in separate goroutine before sender
ch := make(chan int)                                                                                                                                                                                           
go func() { fmt.Println(<-ch) }()  // receiver ready
ch <- 42                            // sender — can now proceed

// FIX B: buffered channel when capacity is known                                                                                                                                                              
ch := make(chan int, 1)                                                                                                                                                                                        
ch <- 42  // doesn't block — buffer absorbs the send
fmt.Println(<-ch)

Goroutine blocked on channel, caller abandons it

// PARTIAL DEADLOCK: ctx cancelled, function returns,
// but goroutine is stuck forever trying to send on the abandoned channel

func fetchData(ctx context.Context, id string) (Result, error) {                                                                                                                                               
resultCh := make(chan Result)  // unbuffered

      go func() { 
          r := expensiveQuery(id)     // takes 10 seconds
          resultCh <- r               // nobody reading if ctx was cancelled — goroutine leaks                                                                                                                   
      }()                                                                                                                                                                                                        
                                                                                                                                                                                                                 
      select {                                                                                                                                                                                                   
      case r := <-resultCh:
          return r, nil                                                                                                                                                                                          
      case <-ctx.Done():
          return Result{}, ctx.Err() // returns, but goroutine above is now orphaned forever                                                                                                                     
      }                                                                                                                                                                                                          
}

// FIX: always use buffered channel (size 1) for goroutine results
// The goroutine can always send its result and exit cleanly                                                                                                                                                   
func fetchData(ctx context.Context, id string) (Result, error) {                                                                                                                                               
resultCh := make(chan Result, 1)  // buffered: goroutine never blocks on send

      go func() { 
          r := expensiveQuery(id)                                                                                                                                                                                
          select {
          case resultCh <- r:  // send if caller is still waiting
          case <-ctx.Done():   // bail out if caller is gone                                                                                                                                                     
          }                                                                                                                                                                                                      
      }()                                                                                                                                                                                                        
                                                                                                                                                                                                                 
      select {    
      case r := <-resultCh:
          return r, nil
      case <-ctx.Done():
          return Result{}, ctx.Err()
      }
}

Circular channel dependency

// DEADLOCK: A sends to chA (waiting for B to receive)
//           B sends to chB (waiting for A to receive)                                                                                                                                                         
//           Neither can proceed — circular wait

chA := make(chan int)
chB := make(chan int)

go func() { // goroutine A
chA <- 1    // A sends — blocks until B reads chA
<-chB       // A waits — but B is blocked on chB send                                                                                                                                                      
}()

go func() { // goroutine B                                                                                                                                                                                     
chB <- 2    // B sends — blocks until A reads chB
<-chA       // B waits — but A is blocked on chA send
}()                                                                                                                                                                                                            
// Both blocked — deadlock

// FIX: break the circular dependency
// One goroutine receives first, then sends

go func() { // goroutine A: send first
chA <- 1                                                                                                                                                                                                   
<-chB                                                                                                                                                                                                      
}()

go func() { // goroutine B: receive first, then send
v := <-chA   // receive from A — unblocks A
chB <- v + 1 // send to A — unblocks A's second statement                                                                                                                                                  
}()

Range over channel that is never closed

// DEADLOCK: range blocks forever waiting for channel to close
ch := make(chan int)                                                                                                                                                                                           
go func() {
ch <- 1                                                                                                                                                                                                    
ch <- 2     
ch <- 3
// forgot to close(ch)
}()

for v := range ch {  // blocks after receiving 3 — waiting for close that never comes                                                                                                                          
fmt.Println(v)
}

// FIX: always close the channel when done sending
ch := make(chan int)                                                                                                                                                                                           
go func() {
defer close(ch)  // ensures channel is closed even on panic or early return                                                                                                                                
ch <- 1                                                                                                                                                                                                    
ch <- 2
ch <- 3                                                                                                                                                                                                    
}()

for v := range ch {  // terminates cleanly when ch is closed                                                                                                                                                   
fmt.Println(v)
}

Closing a channel that might already be closed

// PANIC (leads to deadlock-like state): closing a closed channel panics                                                                                                                                       
ch := make(chan int)                                                                                                                                                                                           
close(ch)
close(ch)  // panic: close of closed channel

// Scenario: multiple goroutines racing to close
var wg sync.WaitGroup                                                                                                                                                                                          
ch := make(chan struct{})                                                                                                                                                                                      
for i := 0; i < 5; i++ {
wg.Add(1)                                                                                                                                                                                                  
go func() {
defer wg.Done()
close(ch)  // multiple goroutines closing same channel — panic                                                                                                                                         
}()
}

// FIX: sync.Once guarantees close happens exactly once                                                                                                                                                        
var once sync.Once
closeOnce := func() { once.Do(func() { close(ch) }) }

for i := 0; i < 5; i++ {
wg.Add(1)                                                                                                                                                                                                  
go func() {
defer wg.Done()
closeOnce()  // safe — only first call closes, rest are no-ops
}()                                                                                                                                                                                                        
}
                                                                                                                                                                                                                 
---             
Category 3: Mixed Mutex + Channel Deadlocks

Holding a mutex while blocking on a channel

// DEADLOCK:    
// G1: holds mu, tries to send to ch (waits for receiver)                                                                                                                                                      
// G2: tries to acquire mu (waits for G1 to release)
// G1 can't release mu until it sends; G2 can't receive until it gets mu

var mu sync.Mutex                                                                                                                                                                                              
ch := make(chan int)

go func() { // G1                                                                                                                                                                                              
mu.Lock()
ch <- 42    // blocks — G2 needs to receive, but G2 is blocked waiting for mu                                                                                                                              
mu.Unlock()                                                                                                                                                                                                
}()

go func() { // G2
mu.Lock()   // blocks — G1 holds mu
v := <-ch   // G2 would receive, but never gets here                                                                                                                                                       
mu.Unlock()                                                                                                                                                                                                
fmt.Println(v)                                                                                                                                                                                             
}()                                                                                                                                                                                                            
// G1 holds mu waiting to send; G2 waiting for mu to receive — DEADLOCK

// FIX: never hold a mutex while blocking on a channel
// Release mutex before channel operations

go func() { // G1                                                                                                                                                                                              
mu.Lock()   
data := prepareData()
mu.Unlock()  // release BEFORE blocking on channel
ch <- data   // send without holding mutex                                                                                                                                                                 
}()

go func() { // G2
v := <-ch    // receive without holding mutex
mu.Lock()
process(v)                                                                                                                                                                                                 
mu.Unlock()
}()
                  
---
Category 4: WaitGroup Deadlocks

Done() called more times than Add()

// PANIC: negative WaitGroup counter → undefined state
var wg sync.WaitGroup                                                                                                                                                                                          
wg.Add(2)       
wg.Done()                                                                                                                                                                                                      
wg.Done()       
wg.Done()  // panic: sync: negative WaitGroup counter

// FIX: keep Add() and Done() symmetric
// Use defer wg.Done() immediately when goroutine starts                                                                                                                                                       
var wg sync.WaitGroup

process := func(item Item) {                                                                                                                                                                                   
defer wg.Done()  // declared right after Add, impossible to forget
// ... work                                                                                                                                                                                                
}

for _, item := range items {
wg.Add(1)
go process(item)
}                                                                                                                                                                                                              
wg.Wait()

Wait() called before all Add() calls

// PARTIAL DEADLOCK: Wait returns before all goroutines finish

var wg sync.WaitGroup

// BAD: Add inside the goroutine — racy                                                                                                                                                                        
for i := 0; i < 5; i++ {
go func() {                                                                                                                                                                                                
wg.Add(1)       // may run AFTER wg.Wait() returns                                                                                                                                                     
defer wg.Done()
doWork()                                                                                                                                                                                               
}()         
}                                                                                                                                                                                                              
wg.Wait()  // may return immediately before any goroutine calls Add

// GOOD: Add before starting goroutine — happens-before is established                                                                                                                                         
for i := 0; i < 5; i++ {                                                                                                                                                                                       
wg.Add(1)           // guaranteed to happen before Wait() starts counting                                                                                                                                  
go func() {                                                                                                                                                                                                
defer wg.Done()
doWork()                                                                                                                                                                                               
}()         
}
wg.Wait()

  ---
Category 5: Context and Select Deadlocks

Select with all cases permanently blocked

// DEADLOCK: no default, all channels nil or empty, no ctx timeout
ch1 := make(chan int)                                                                                                                                                                                          
ch2 := make(chan int)

// Nothing ever sends to ch1 or ch2                                                                                                                                                                            
select {
case v := <-ch1:                                                                                                                                                                                               
fmt.Println(v)
case v := <-ch2:
fmt.Println(v)
}
// blocks forever

// FIX: always include ctx.Done() in production select statements
select {                                                                                                                                                                                                       
case v := <-ch1:
fmt.Println(v)                                                                                                                                                                                             
case v := <-ch2:
fmt.Println(v)                                                                                                                                                                                             
case <-ctx.Done():
return ctx.Err()
}

// Or add a timeout as a safety net                                                                                                                                                                            
select {
case v := <-ch1:                                                                                                                                                                                               
process(v)  
case <-time.After(5 * time.Second):
return fmt.Errorf("operation timed out")                                                                                                                                                                   
}

Sending to a nil channel blocks forever

// A nil channel blocks on both send and receive — forever
// Commonly caused by uninitialized channels

var ch chan int  // nil                                                                                                                                                                                        
ch <- 42        // blocks forever — not a panic, just hangs

// Useful pattern: disable a case in select by setting channel to nil                                                                                                                                          
func merge(ctx context.Context, ch1, ch2 <-chan int) <-chan int {                                                                                                                                              
out := make(chan int, 2)                                                                                                                                                                                   
go func() {                                                                                                                                                                                                
defer close(out)                                                                                                                                                                                       
for ch1 != nil || ch2 != nil {                                                                                                                                                                         
select {
case v, ok := <-ch1:
if !ok {
ch1 = nil  // disables this case — nil channel never selected
continue                                                                                                                                                                                   
}
out <- v                                                                                                                                                                                       
case v, ok := <-ch2:
if !ok {
ch2 = nil  // disables this case
continue
}
out <- v
case <-ctx.Done():                                                                                                                                                                                 
return
}                                                                                                                                                                                                  
}       
}()
return out
}

  ---
Category 6: Goroutine Leak as Slow Deadlock

Goroutine leaks don't cause immediate deadlocks, but accumulate until the system degrades completely.

// LEAK: goroutine blocked on receive, nobody ever sends
func startWorker(id int) {                                                                                                                                                                                     
jobs := make(chan Job)  // created but never sent to
go func() {                                                                                                                                                                                                
for job := range jobs {  // blocks forever
process(job)                                                                                                                                                                                       
}       
}()
// jobs never closed, channel never sent to — goroutine leaks                                                                                                                                              
}

// LEAK: goroutine blocked on send to a full buffered channel
func producer(ctx context.Context) {                                                                                                                                                                           
ch := make(chan Event, 10)
go func() {                                                                                                                                                                                                
for {
ch <- generateEvent()  // blocks when buffer full and consumer stops                                                                                                                               
// if consumer exits, this goroutine is stuck forever                                                                                                                                              
}                                                                                                                                                                                                      
}()                                                                                                                                                                                                        
}

// FIX: every goroutine must have a guaranteed exit path

func startWorker(ctx context.Context, id int) {
jobs := make(chan Job, 10)                                                                                                                                                                                 
go func() {
for {
select {
case job, ok := <-jobs:                                                                                                                                                                            
if !ok {
return  // channel closed — clean exit                                                                                                                                                     
}
process(job)
case <-ctx.Done():
return  // context cancelled — clean exit                                                                                                                                                      
}
}                                                                                                                                                                                                      
}()         
return jobs  // caller controls the channel lifecycle
}

// Detect goroutine leaks in tests                                                                                                                                                                             
import "go.uber.org/goleak"

func TestWorker(t *testing.T) {
defer goleak.VerifyNone(t)  // fails if any goroutines leaked during test

      ctx, cancel := context.WithCancel(context.Background())
      jobs := startWorker(ctx, 1)                                                                                                                                                                                
      jobs <- Job{ID: "test"}                                                                                                                                                                                    
      cancel()  // signals goroutine to exit
      // goleak verifies clean exit                                                                                                                                                                              
}
  
---                                                                                                                                                                                                            
Detecting Deadlocks in Production

// 1. pprof goroutine dump — shows all blocked goroutines with stack traces
import _ "net/http/pprof"

go func() {                                                                                                                                                                                                    
http.ListenAndServe(":6060", nil)                                                                                                                                                                          
}()

// curl http://localhost:6060/debug/pprof/goroutine?debug=2                                                                                                                                                    
// Look for goroutines stuck in:
//   chan receive    → waiting for channel send                                                                                                                                                                
//   chan send       → waiting for channel receiver                                                                                                                                                            
//   semacquire      → waiting for mutex                                                                                                                                                                       
//   select          → all cases blocked

// 2. Monitor goroutine count — growth indicates leaks heading toward deadlock                                                                                                                                 
go func() {
ticker := time.NewTicker(30 * time.Second)                                                                                                                                                                 
for range ticker.C {                                                                                                                                                                                       
slog.Info("runtime stats",
"goroutines", runtime.NumGoroutine(),                                                                                                                                                              
)                                                                                                                                                                                                      
// Alert if goroutines grow unboundedly
}                                                                                                                                                                                                          
}()

// 3. Timeout as last resort safety net — prevents permanent hangs                                                                                                                                             
func withDeadlockTimeout[T any](
ctx context.Context,                                                                                                                                                                                       
timeout time.Duration,
fn func(context.Context) (T, error),                                                                                                                                                                       
) (T, error) {                                                                                                                                                                                                 
ctx, cancel := context.WithTimeout(ctx, timeout)
defer cancel()                                                                                                                                                                                             
return fn(ctx)
}

  ---
Summary: Rules That Prevent Most Deadlocks

Mutex Rules
────────────────────────────────────────────────────────────────────
1. Define a global lock order for all mutexes — never acquire in reverse
2. Never lock the same mutex twice in the same goroutine (not reentrant)
3. Release locks before calling external code or callbacks
4. Keep critical sections short — never block (I/O, channel ops) while holding a lock
5. Never copy a struct that contains a mutex

Channel Rules                                                                                                                                                                                                  
────────────────────────────────────────────────────────────────────
6. Sender closes the channel — never the receiver
7. Use sync.Once to close a channel that has multiple potential closers
8. Use buffered channel (size 1) for goroutine result — prevents goroutine leak
9. Always include ctx.Done() in select when blocking on channels
10. Range over channel only when the sender will definitely close it

Goroutine Rules                                                                                                                                                                                                
────────────────────────────────────────────────────────────────────
11. Every goroutine must have a documented exit path
12. Never hold a mutex while blocking on a channel send/receive
13. wg.Add(n) must happen before go func() — not inside the goroutine
14. wg.Done() and wg.Add() must be symmetric — defer wg.Done() immediately
15. Use goleak in tests to catch leaks before they reach production

General                                                                                                                                                                                                        
────────────────────────────────────────────────────────────────────
16. Always propagate context — let callers cancel blocking operations
17. Use timeouts on every external call — no unbounded waits in production
18. Run go test -race -count=100 in CI — catches races that lead to deadlocks
19. Monitor goroutine count — unbounded growth signals an impending deadlock  