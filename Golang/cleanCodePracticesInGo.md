# Clean Code Practices in Go — Staff-Level Reference

> A pragmatic, opinionated reference for writing Go code that survives review, scales beyond the initial author, and runs reliably in MAANG-grade production. Not a beginner's tour: this is what I expect to see (and what I expect not to see) from a senior engineer trying to operate at staff level. Each item below is an answer to a question I've heard in real PRs: "why this way, and not that way?"

> Companion to the other Golang docs in this folder (`memoryLeaksInGo.md`, `handlingRaceConditionsInGo.md`, `deadlocksInGO.md`, `returningInGo.md`).

---

## 0. The Go Philosophy — and Why It Matters

Go's design philosophy is an unusually strong constraint on style. Internalize these and most "should I do X" questions answer themselves:

1. **Clarity > cleverness.** Code is read 10× more than written. The reader's time is the thing being optimized.
2. **Composition > inheritance.** No class hierarchies. Embed types; satisfy interfaces structurally.
3. **Errors are values.** Not exceptions. Handled at every step or explicitly propagated.
4. **Concurrency is a feature, not a library.** Designed into the language; demands discipline.
5. **One way to do it.** Strong cultural pressure for idiomatic patterns. Resist creative deviation.
6. **Tooling is part of the language.** `gofmt`, `go vet`, `go test`, `go build` shape what "good code" means.
7. **Small interfaces.** "The bigger the interface, the weaker the abstraction." (Pike)
8. **Don't communicate by sharing memory; share memory by communicating.** Channel-first concurrency mindset, even when channels aren't the right fit.
9. **A little copying is better than a little dependency.** DRY isn't a virtue; coupling is a vice.
10. **Documentation is code.** Doc comments live next to declarations and are part of the API surface.

**Staff-level shift**: stop trying to make Go feel like Java/Python/Rust/Haskell. Go has its own taste. The fastest path from senior to staff in Go is to *deeply* internalize idioms — then know when to break them.

---

## 1. Naming

Naming is 80% of style in Go. Get this right and most other things follow.

### 1.1 General rules

```go
// GOOD: short, contextual, lowercase camelCase for local
i, n, ok := 0, len(items), true
buf := bytes.Buffer{}
ctx := context.Background()
err := doWork(ctx)

// BAD: redundantly long
itemIndex, totalNumberOfItems, isOk := 0, len(items), true
bufferOfBytes := bytes.Buffer{}
context_object := context.Background()
errorVariable := doWork(context_object)
```

- **Local variables**: short. `i`, `n`, `err`, `ok`, `ctx`, `buf`, `req`, `res`, `cur`. The shorter the scope, the shorter the name.
- **Package-level**: longer, descriptive. `MaxRetryAttempts`, `defaultTimeout`.
- **Acronyms uppercase entirely**: `parseURL` not `parseUrl`; `userID` not `userId`; `HTTPClient` not `HttpClient`.
- **No Hungarian, no underscores in names** (except `_test.go`, `_linux.go` build tags).

### 1.2 Receiver names

```go
// GOOD: short, consistent across all methods of the type
func (s *Server) Start() error { ... }
func (s *Server) Stop() error { ... }
func (s *Server) handle(req *Request) { ... }

// BAD: descriptive, inconsistent
func (server *Server) Start() error { ... }
func (this *Server) Stop() error { ... }
func (myServer *Server) handle(req *Request) { ... }
```

The receiver is one or two letters, derived from the type name, and **identical across every method on that type**. Reviewers spot inconsistency immediately.

### 1.3 Interfaces

```go
// GOOD: -er suffix for single-method interfaces
type Reader interface { Read(p []byte) (n int, err error) }
type Logger interface { Log(msg string, fields ...Field) }
type UserStore interface { Save(ctx context.Context, u User) error; Find(ctx context.Context, id ID) (User, error) }

// BAD: prefix with "I"; meaningless suffix
type IReader interface { ... }
type LoggerInterface interface { ... }
type AbstractUserStore interface { ... }
```

Single-method: `-er`. Multi-method: name by the role (`UserStore`, not `UserStoreInterface`).

### 1.4 Package names

```go
// GOOD
package user
package httputil
package retry
package metrics

// BAD
package userPackage
package user_utils       // underscores; non-idiomatic
package UserService      // capitalized; non-idiomatic
package util             // generic; smells of a junk drawer
```

- **Short, lowercase, no underscores**.
- Avoid `util`, `common`, `helpers`, `misc` — package names that mean nothing. Indicates lack of design.
- The package name is **part of the API**: `time.Duration` reads better than `timeutil.TimeDuration`.

### 1.5 Errors

```go
// GOOD: sentinel errors prefixed with Err
var ErrNotFound = errors.New("user not found")

// GOOD: error types named -Error
type ValidationError struct { Field string; Reason string }

// GOOD: error variables in their natural place
package db
var ErrNoRows = errors.New("no rows in result set")

// BAD
var NotFoundError = errors.New("...")    // missing "Err" prefix
type validationErr struct { ... }         // unexported when should be exported
```

### 1.6 Constants

```go
// GOOD: PascalCase for exported, camelCase for unexported.
const (
    MaxConnections = 100
    defaultTimeout = 30 * time.Second
)

// GOOD: enum-like via typed constants
type Status int
const (
    StatusPending Status = iota
    StatusActive
    StatusDone
)

// BAD: SCREAMING_CASE (not Go style)
const MAX_CONNECTIONS = 100
const DEFAULT_TIMEOUT = 30
```

---

## 2. Packages and Imports

A Go package is a **unit of design**. Treat package boundaries as architectural decisions.

### 2.1 What goes in a package

A package should answer one cohesive question:
- `package user` — everything about users.
- `package retry` — retry primitives.
- `package storage/postgres` — postgres-backed storage.

Avoid:
- **Catch-all packages**: `util`, `helpers`, `common`. Sign of design rot.
- **Layer-named packages**: `models`, `controllers`, `services` (Java/Rails import). Go organizes by **feature**, not **layer**.
- **Single-file packages with grandiose names**: `package authentication_handler`.

### 2.2 Package size

- 1–10 files typical.
- 50+ files: package is doing too much. Split.
- 1 file: probably belongs in a parent package.

### 2.3 Import organization

```go
import (
    // standard library first
    "context"
    "errors"
    "fmt"
    "net/http"

    // third-party second
    "github.com/google/uuid"
    "go.opentelemetry.io/otel"

    // first-party (your org) last
    "github.com/myorg/myrepo/internal/auth"
    "github.com/myorg/myrepo/internal/store"
)
```

Three groups, blank line between. `goimports` does this automatically.

### 2.4 The `internal/` directory

```
mymodule/
├── cmd/
│   └── server/main.go     // public entry point
├── internal/
│   ├── auth/              // not importable from outside the module
│   ├── store/
│   └── server/
└── pkg/                   // public, importable by other modules (rare)
    └── client/
```

`internal/` is a Go convention — packages here cannot be imported from outside the module tree. Use it aggressively to prevent accidental coupling.

### 2.5 Import cycles

Go forbids them. If you hit one, the design is wrong:
- **Extract the shared interface or type** to a third package.
- **Push the dependency through a callback** (function or interface).
- **Question whether two packages should be one.**

Workarounds (interface in package A, impl in package B importing A) are typical.

### 2.6 The `init()` trap

```go
// AVOID: package-level init that has side effects
func init() {
    db = mustOpenDB()           // bad: cannot test in isolation
    metrics.Register("foo", ...) // bad: implicit registration
}
```

`init()` runs before `main()`, in undefined order across packages, with no error path. Reserve for very narrow cases (registering a database driver, registering a JSON codec). For everything else, make initialization explicit:

```go
// GOOD
func New(cfg Config) (*Server, error) {
    s := &Server{...}
    if err := s.connect(); err != nil { return nil, err }
    return s, nil
}
```

---

## 3. Functions and Methods

### 3.1 Length and complexity

- **Most functions: 5–30 lines.** Beyond ~50, extract.
- **Cyclomatic complexity: 10 or below**. `gocyclo` enforces.
- **One level of abstraction per function.** Don't mix high-level orchestration with bit-twiddling.

### 3.2 Parameters

- **0–3 parameters**: ideal.
- **4–6**: borderline; consider grouping into a struct.
- **7+**: red flag; group into a `Request` / `Options` struct.

```go
// BAD
func CreateUser(ctx context.Context, name, email, phone string, age int, country string, premium bool) error

// GOOD
type CreateUserRequest struct {
    Name, Email, Phone, Country string
    Age                         int
    Premium                     bool
}
func CreateUser(ctx context.Context, req CreateUserRequest) error
```

### 3.3 Context first, error last

```go
// GOOD: idiomatic signature
func DoThing(ctx context.Context, req Request) (Response, error)

// BAD
func DoThing(req Request, ctx context.Context) (error, Response)
```

`context.Context` is **always** the first parameter when present. `error` is **always** the last return value when present. These are unwritten Go laws.

### 3.4 Return values

- **Up to 3 values**: idiomatic. `(value, ok)`, `(value, err)`, `(value, count, err)`.
- **More than 3**: wrap in a struct.
- **Named returns**: useful for documentation; avoid for control flow.

```go
// GOOD: named returns clarify what's returned, but body still uses explicit returns
func split(s string) (head, tail string, err error) {
    i := strings.IndexByte(s, '.')
    if i < 0 {
        return "", "", fmt.Errorf("no separator in %q", s)
    }
    return s[:i], s[i+1:], nil
}

// AVOID: bare return relying on named returns is harder to follow
func split(s string) (head, tail string, err error) {
    head = s[:5]
    tail = s[6:]
    return  // bare return; reviewer must trace assignments
}
```

### 3.5 Methods vs functions

```go
// Method: when the operation is about the receiver's state
func (u *User) Validate() error

// Function: when the operation is over data, possibly multiple types
func ValidateAll(users []User) error
```

If a method doesn't use the receiver's fields/methods, it should probably be a function.

### 3.6 Pointer vs value receivers

```go
// VALUE receiver: small, immutable, copy is cheap
type Point struct { X, Y float64 }
func (p Point) Distance(q Point) float64 { ... }

// POINTER receiver: large, mutable, or implementing an interface that requires it
type Server struct { conn net.Conn; buf []byte; ... }
func (s *Server) Start() error { ... }
```

**Rules of thumb:**

1. If any method is pointer, **all should be pointer** (consistency).
2. If the type is large (> ~64 bytes), pointer receiver to avoid copy.
3. If the method mutates, pointer receiver (else mutation is on a copy, no effect).
4. For small immutable types (`time.Time`, `Point`, `Color`), value receiver.
5. If unsure, default to pointer.

### 3.7 Avoid `interface{}` (now `any`) at API boundaries

```go
// BAD: caller doesn't know what to pass
func Process(data any) error

// GOOD: typed parameters
func ProcessUser(u User) error
func ProcessOrder(o Order) error

// GOOD (with generics, Go 1.18+): typed and reusable
func Process[T Validatable](v T) error
```

`any` at the boundary forces runtime type assertions; lose compile-time guarantees.

### 3.8 Variadic — sparingly

```go
// GOOD: fits the API
func fmt.Printf(format string, args ...any)

// AVOID: variadic that hides intent
func CreateUser(opts ...Option) (*User, error)  // sometimes ok with functional options
func DoThings(ids ...int) error                  // why not []int?
```

Variadic is a callsite convenience; it is not a substitute for clear types.

---

## 4. Error Handling

The most-discussed and most-debated topic in Go. Staff-level practice:

### 4.1 Errors are values; treat them seriously

```go
// CANONICAL
res, err := doThing()
if err != nil {
    return fmt.Errorf("doing thing: %w", err)
}
```

Three things every error site should do:
1. **Check** the error.
2. **Add context** with `fmt.Errorf("...: %w", err)` so the chain is informative.
3. **Wrap, don't shadow**: use `%w` (not `%v`) so `errors.Is` and `errors.As` work upstream.

### 4.2 Wrapping vs creating

```go
// Wrap: preserve the underlying error so callers can errors.Is / errors.As
return fmt.Errorf("loading user %d: %w", id, err)

// Replace: only when the underlying error should NOT be exposed (security, abstraction)
return errors.New("user not found")  // hide DB error from API caller

// Sentinel: at the package boundary, define stable errors
var ErrNotFound = errors.New("user: not found")
```

### 4.3 errors.Is vs errors.As

```go
// errors.Is: checking against a sentinel
if errors.Is(err, sql.ErrNoRows) { ... }

// errors.As: extracting a typed error
var perr *fs.PathError
if errors.As(err, &perr) { fmt.Println(perr.Path) }

// AVOID: type assertion on wrapped errors
if pe, ok := err.(*fs.PathError); ok { ... }  // misses wrapped cases
```

`errors.Is` and `errors.As` walk the wrap chain. Direct type assertions don't.

### 4.4 Don't `panic` for recoverable errors

```go
// BAD: mostly recoverable issue
func MustParseInt(s string) int {
    n, err := strconv.Atoi(s)
    if err != nil { panic(err) }
    return n
}

// GOOD: panic only for truly unrecoverable / programmer-error situations
func mustNotBeNil(v any) {
    if v == nil { panic("internal: must not be nil") }
}
```

`panic` is for **programmer errors** (nil deref, off-by-one, contract violation), not for runtime conditions. Most of `panic` in production code is wrong.

### 4.5 Panic recovery — at well-defined boundaries

```go
// HTTP middleware: recover so one panicking handler doesn't kill the process
func recoverMiddleware(h http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                log.Printf("panic: %v\n%s", rec, debug.Stack())
                http.Error(w, "internal server error", http.StatusInternalServerError)
            }
        }()
        h.ServeHTTP(w, r)
    })
}
```

`recover` only at goroutine / request boundaries. Not in arbitrary functions; that's hiding bugs.

### 4.6 Error types vs sentinels

```go
// Sentinel: when only the identity matters
var ErrAlreadyExists = errors.New("already exists")

// Type: when context must be carried
type ValidationError struct {
    Field string
    Reason string
}
func (v *ValidationError) Error() string {
    return fmt.Sprintf("validation: field=%s reason=%s", v.Field, v.Reason)
}
```

Use sentinels for the simple case; types when the error carries data the caller needs.

### 4.7 Error returns are not optional

```go
// BAD: silently swallowing
res, _ := doThing()  // why?

// GOOD: explicit handling
res, err := doThing()
if err != nil {
    return nil, fmt.Errorf("...: %w", err)
}

// LEGITIMATE _: only when the err is genuinely uninteresting (already-checked invariant)
buf, _ := io.ReadAll(strings.NewReader(s))  // strings.Reader never errors
```

Discarding errors with `_` is a code smell that needs justification.

### 4.8 Don't log AND return

```go
// BAD: logged twice as error percolates up
log.Printf("error: %v", err)
return err

// GOOD: log only at the boundary where the error stops propagating
return fmt.Errorf("loading user: %w", err)
```

Caller decides what to do (log, retry, fail). Logging in the middle pollutes logs.

### 4.9 The error budget for boilerplate

A common complaint: Go has lots of `if err != nil`. Yes. Resist hiding it with macros. The verbosity is the point — every error is a decision. Tools to make it tolerable:
- IDE snippets for `if err != nil { return err }`.
- `errcheck`, `errorlint` linters.
- Functional helpers (`errgroup`, `multierr`) for specific cases.

But **do not** invent a Result type or panic-and-recover-as-control-flow. You'll make your code unreadable to every other Go engineer.

---

## 5. Interfaces

Where Go gets its real power. Done right, interfaces enable testability, modularity, and decoupling. Done wrong, they are noise.

### 5.1 Small interfaces

```go
// Standard library is the model:
type Reader interface { Read(p []byte) (n int, err error) }
type Writer interface { Write(p []byte) (n int, err error) }
type Closer interface { Close() error }

// Composed:
type ReadCloser interface { Reader; Closer }
```

The smaller the interface, the more types satisfy it, the more flexible the API. The Go proverb: **"the bigger the interface, the weaker the abstraction."**

### 5.2 Define interfaces where they're consumed, not where they're implemented

```go
// BAD: defining the interface in the implementation package forces consumers to import you
package userstore
type UserStore interface { Save(...); Find(...) }
type postgresStore struct { ... }

// GOOD: consumer defines what it needs
package userhandler
type userStore interface { Save(...); Find(...) }
func New(s userStore) *Handler { ... }
```

This way, the consumer's needs drive the interface — not the implementation's accidental method set. It also makes mocking trivial; you don't need a mock library.

### 5.3 Accept interfaces, return structs

```go
// BAD: returning interface forces caller to use only that shape
func NewService() ServiceInterface

// GOOD: returning concrete type lets caller use any of its methods
func NewService() *Service
func DoWork(s readerLike) error  // accept interface
```

Caller passes whatever satisfies the interface; returns the concrete type so the receiver gets full functionality.

### 5.4 Don't pre-emptively interface-ify

```go
// BAD: every concrete type wrapped in a single-impl interface "for testability"
type UserService interface { CreateUser(...) error }
type userService struct { ... }
// ... only one implementation ever exists

// GOOD: interface emerges when there are 2+ implementations or a real testing need
type UserService struct { ... }
```

Interfaces are abstraction; abstraction has a cost (indirection, mock complexity, harder to navigate). Add when needed, not in advance.

### 5.5 Empty interface (any) is sometimes right

```go
// GOOD: APIs that genuinely need to accept anything
func log.Print(args ...any)
func json.Marshal(v any) ([]byte, error)

// BAD: using any to dodge type design
type Cache map[string]any
```

If your domain genuinely accepts heterogeneous values, `any` is right. If you're using it to skip type design, refactor.

### 5.6 Interface satisfaction is structural and silent

```go
type Greeter interface { Greet() string }
type Person struct { Name string }
func (p Person) Greet() string { return "hi " + p.Name }

// Person automatically satisfies Greeter — no declaration.
var g Greeter = Person{Name: "Sam"}
```

This is powerful but means: **you can break an interface contract by changing a method signature far from the interface declaration**. Mitigation: an explicit `var _ Greeter = (*Person)(nil)` near the type, asserting it satisfies the interface at compile time.

```go
type Person struct { ... }
var _ Greeter = (*Person)(nil)  // compile-time check
func (p *Person) Greet() string { ... }
```

A standard idiom in production code.

### 5.7 Interface pollution

```go
// BAD: package full of "Service", "Repository", "Manager" interfaces
type UserService interface { ... }
type OrderService interface { ... }
type PaymentService interface { ... }
type NotificationService interface { ... }

// Each has exactly one implementation. Net effect: indirection without benefit.
```

The Go community calls this "interface pollution." The Java reflex of "interface for every class" is non-idiomatic and harmful to readability.

---

## 6. Concurrency — Goroutines

Go's flagship feature; also the source of most production bugs.

### 6.1 The cardinal rules

1. **Every goroutine has an owner** — the function that started it. The owner knows when it ends.
2. **Goroutines must be cancellable** — via `context.Context` or by closing a channel.
3. **Goroutines must terminate** — leaks are silent and accumulate.
4. **Goroutines must not panic uncaught** — panic in a goroutine kills the whole process.

### 6.2 Don't start goroutines you don't own

```go
// BAD: fire-and-forget; no way to wait for it or cancel it
func handle(req *Request) {
    go process(req)   // forgotten
    return
}

// GOOD: ownership is clear via wait group / errgroup / context
func handle(ctx context.Context, req *Request) error {
    g, gctx := errgroup.WithContext(ctx)
    g.Go(func() error { return process(gctx, req) })
    return g.Wait()
}
```

If you start a goroutine and don't `Wait` on it, you've created an unsupervised process inside your process. At scale, these are leaks waiting to be found.

### 6.3 Goroutine leaks — the canonical patterns

**Sender block on full channel**:
```go
// LEAK: receiver gone; sender blocks forever on channel send
func leak() {
    ch := make(chan int)        // unbuffered
    go func() { ch <- 1 }()     // sender
    return                      // receiver never started; goroutine stuck
}
```

**Receiver block on never-sent channel**:
```go
// LEAK: sender forgot to close or send
func leak() <-chan int {
    ch := make(chan int)
    go func() {
        // forgot to send or close
    }()
    return ch  // receiver forever stuck reading from ch
}
```

**Forgotten select case**:
```go
// LEAK: ctx cancellation not handled
func leak(ctx context.Context, ch <-chan int) {
    go func() {
        for {
            select {
            case v := <-ch:
                process(v)
            // missing: case <-ctx.Done(): return
            }
        }
    }()
}
```

Always include `<-ctx.Done()` in long-running selects.

### 6.4 errgroup — the right primitive for parallel work

```go
import "golang.org/x/sync/errgroup"

func fetchAll(ctx context.Context, urls []string) ([]Response, error) {
    g, ctx := errgroup.WithContext(ctx)
    results := make([]Response, len(urls))

    for i, url := range urls {
        i, url := i, url  // capture (pre-Go-1.22 loop variable)
        g.Go(func() error {
            res, err := fetch(ctx, url)
            if err != nil { return err }
            results[i] = res
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}
```

`errgroup`:
- Bounds goroutines to a known scope.
- First error cancels the others via the derived context.
- `Wait()` is the synchronization point.

For larger fan-out, use `g.SetLimit(N)` to cap concurrency.

### 6.5 sync.WaitGroup — when errgroup overkill

```go
var wg sync.WaitGroup
for _, item := range items {
    wg.Add(1)
    go func(item Item) {
        defer wg.Done()
        process(item)
    }(item)
}
wg.Wait()
```

Use when:
- No errors to propagate.
- No early cancellation needed.
- Just "wait for all to finish."

Beware:
- `wg.Add` must be called *before* the goroutine starts, not inside it.
- `wg.Done` *must* be deferred to fire on panic.

### 6.6 Channels — use with care

```go
// Buffered vs unbuffered
ch := make(chan int)      // unbuffered: synchronous handoff
ch := make(chan int, 100) // buffered: async until full

// Direction in function signatures
func produce(out chan<- int)        // send-only
func consume(in <-chan int)         // receive-only
func transform(in <-chan int, out chan<- int)
```

**Channel-direction in signatures** is documentation and a compile-time check. Always use it.

**Closing**: only the **sender** closes a channel. Closing in the receiver, or by multiple senders, is a panic. For multi-sender, use `sync.Once` or a coordination signal.

### 6.7 The "for range" over channel

```go
// GOOD: terminates when sender closes
for v := range ch {
    process(v)
}

// Equivalent verbose form
for {
    v, ok := <-ch
    if !ok { break }
    process(v)
}
```

### 6.8 Select with default — non-blocking

```go
// Non-blocking send
select {
case ch <- v:
default:
    // would block; drop or alt path
}

// Non-blocking receive
select {
case v := <-ch:
    use(v)
default:
    // nothing to receive; alt path
}
```

Useful for shedding under load, but **don't use as polling loop** — it CPU-spins.

### 6.9 Worker pools

```go
func workerPool(ctx context.Context, n int, jobs <-chan Job) error {
    g, ctx := errgroup.WithContext(ctx)
    for i := 0; i < n; i++ {
        g.Go(func() error {
            for {
                select {
                case <-ctx.Done():
                    return ctx.Err()
                case job, ok := <-jobs:
                    if !ok { return nil }
                    if err := handle(ctx, job); err != nil { return err }
                }
            }
        })
    }
    return g.Wait()
}
```

A worker pool is a bounded number of goroutines pulling from a job channel. Bounds memory and downstream load.

### 6.10 Panic in a goroutine

A panic in a goroutine that's not recovered crashes **the whole process**. Always recover at goroutine boundaries unless you intentionally want to crash:

```go
go func() {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("worker panic: %v\n%s", r, debug.Stack())
        }
    }()
    work()
}()
```

In production code, every spawned goroutine should be wrapped in such a recovery handler — preferably via a helper:

```go
func safeGo(name string, fn func()) {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                log.Printf("[%s] panic: %v\n%s", name, r, debug.Stack())
            }
        }()
        fn()
    }()
}
```

---

## 7. Concurrency — Sync Primitives

Channels are not always the answer. Go's `sync` package is for state, not communication.

### 7.1 sync.Mutex / sync.RWMutex

```go
type SafeCounter struct {
    mu sync.Mutex
    n  int
}

func (c *SafeCounter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.n++
}
```

- **Always defer Unlock** so a panic doesn't leave the mutex held.
- **RWMutex** when reads massively outnumber writes (and reads are non-trivial). For a single-int read, don't bother — Mutex is faster.
- **Don't pass mutex by value** — copies are not the same lock. Vet catches this.

### 7.2 sync.RWMutex caveat

```go
// CONTRADICTION: an RWMutex with mostly fast reads is often slower
// than a plain Mutex due to its more complex internal coordination.
```

Benchmark before using `RWMutex`. Go's `sync.RWMutex` has a writer-starvation bias and lock overhead higher than `Mutex`. Reach for it only when reads are slow enough to amortize the overhead.

### 7.3 sync.Once

```go
var once sync.Once
var conn *Conn

func getConn() *Conn {
    once.Do(func() {
        conn = mustConnect()
    })
    return conn
}
```

Lazy initialization, exactly once. Beats `init()` when initialization should be deferred or has dependencies.

### 7.4 sync.Pool — for short-lived allocations

```go
var bufPool = sync.Pool{
    New: func() any { return new(bytes.Buffer) },
}

func process(data []byte) {
    buf := bufPool.Get().(*bytes.Buffer)
    defer bufPool.Put(buf)
    buf.Reset()
    // use buf...
}
```

**When to use**: high-frequency allocations of equivalent objects (request buffers, parser state). GC-pressure relief.
**When NOT to use**: long-lived state, one-off allocations, very large objects (Pool may unboundedly grow).
**Caveat**: items can be reclaimed at any time (between two `Get`s of the same goroutine), so don't assume Pool retains state.

### 7.5 sync.Map — rarely the right answer

```go
// Specialized concurrent map. Beware:
// - Less type safety than map+mutex.
// - Slower in most cases than map+RWMutex unless your workload matches its sweet spot:
//   "many goroutines reading a stable set of keys, with infrequent writes."
```

In practice, `map[K]V` + `sync.RWMutex` (or `sync.Mutex`) outperforms `sync.Map` for almost everything. Use `sync.Map` only after benchmarking.

### 7.6 Atomic operations

```go
import "sync/atomic"

var counter int64
atomic.AddInt64(&counter, 1)
n := atomic.LoadInt64(&counter)
```

For lock-free counters and flags. Faster than mutex for trivial state, but easy to misuse. Go 1.19+ introduced typed `atomic.Int64`, `atomic.Pointer[T]`, etc., which are safer:

```go
var counter atomic.Int64
counter.Add(1)
n := counter.Load()
```

Prefer the typed forms.

### 7.7 Don't roll your own concurrency primitive

Custom semaphores, custom queues, custom locks are almost always wrong. Use:
- `errgroup` for parallel work with error propagation.
- `semaphore.Weighted` for bounded concurrency.
- `sync.Cond` only if you really know what you're doing (rarely).
- Channels for handoff and coordination.

If you find yourself reaching for `sync.Cond`, step back. There's almost always a channel-based design that's clearer.

---

## 8. Context

`context.Context` is Go's solution to deadlines, cancellation, and request-scoped values. Misuse is rampant.

### 8.1 The rules

1. **Pass `ctx` as the first argument** to functions that do I/O, are cancellable, or call into other functions that do.
2. **Never store `ctx` in a struct.** It's per-call, not per-instance.
3. **Don't pass `nil` `ctx`** — use `context.TODO()` or `context.Background()` if you really have nothing.
4. **Don't use `ctx.Value` as a substitute for parameters.** Reserved for cross-cutting metadata (request ID, trace, auth subject).
5. **Always check `ctx.Done()`** in long-running loops.
6. **Cancellation is best-effort**: code must check; cancellation doesn't kill goroutines automatically.

### 8.2 Idiomatic patterns

```go
// Deadline / timeout
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()

result, err := slowOp(ctx)
```

Always `defer cancel()` — even if the operation finishes early. Prevents resource leaks of the timer.

### 8.3 Cancellation propagation

```go
func parent(ctx context.Context) error {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()

    go child(ctx)  // child sees cancellation when parent's ctx is cancelled or returns
    ...
}
```

Cancellation flows down the tree. Children inherit parent's deadline (whichever fires first wins).

### 8.4 ctx.Value — sparingly

```go
// GOOD: cross-cutting metadata
type ctxKey int
const (
    keyRequestID ctxKey = iota
    keyTrace
)

ctx = context.WithValue(ctx, keyRequestID, "req-123")
id := ctx.Value(keyRequestID).(string)

// BAD: passing business parameters via Value
ctx = context.WithValue(ctx, "user", user)  // string keys collide; not type-safe
processOrder(ctx)  // user is hidden; reviewers don't see it
```

The string-keyed `context.WithValue` is a Java reflex. In Go, use a private `ctxKey` type and helpers:

```go
type ctxKey int
const userKey ctxKey = 1

func withUser(ctx context.Context, u User) context.Context {
    return context.WithValue(ctx, userKey, u)
}
func userFrom(ctx context.Context) (User, bool) {
    u, ok := ctx.Value(userKey).(User)
    return u, ok
}
```

### 8.5 Don't pass `context.TODO()` permanently

`context.TODO()` is "I don't know what context to use here, fix later." It's a marker for incomplete code. Replace with the actual context as you propagate.

---

## 9. Memory — Pointers, Values, and Allocation

### 9.1 Pointer vs value — the practical guide

```go
// VALUE: small, immutable, predictable
type Point struct { X, Y float64 }

// POINTER: large, mutable, identity matters
type Server struct { conn net.Conn; ... }

// VALUE: explicit copy, small, no identity
func dist(a, b Point) float64 { ... }

// POINTER: avoid copy, allow mutation
func (s *Server) Start() error { ... }
```

**Rule**: if `unsafe.Sizeof(T) > 64 bytes`, default to pointer. If T contains a `Mutex`, you must use pointer (mutexes can't be copied).

### 9.2 Don't copy mutexes

```go
type Counter struct {
    mu sync.Mutex
    n  int
}

func bad(c Counter) { ... }   // c.mu is now a separate mutex; intent broken
func good(c *Counter) { ... }
```

`go vet` catches mutex copies. Trust the linter.

### 9.3 Allocations and escape analysis

```go
// Stack allocation: cheap, GC-free, automatic
func foo() int {
    n := 42        // stack
    return n
}

// Heap allocation: GC pressure
func bar() *int {
    n := 42        // n escapes to heap because pointer is returned
    return &n
}
```

Inspect with `go build -gcflags="-m"`. At hot paths, fewer allocations = lower GC pause.

Common escape causes:
- Returning `&local`.
- Storing local in a struct field that outlives the function.
- Capturing local in a closure that escapes.
- Type-asserting concrete types to interfaces (boxes the value).

### 9.4 Pre-allocate slices when size is known

```go
// BAD: grows multiple times, multiple allocations
result := []int{}
for i := 0; i < 1000; i++ {
    result = append(result, i)
}

// GOOD: one allocation
result := make([]int, 0, 1000)
for i := 0; i < 1000; i++ {
    result = append(result, i)
}

// BETTER: if you know size, allocate exactly
result := make([]int, 1000)
for i := range result {
    result[i] = i
}
```

Same for `map`: `make(map[K]V, expectedSize)`.

### 9.5 String concatenation

```go
// BAD: O(n²) in loop
s := ""
for _, p := range parts {
    s += p
}

// GOOD: O(n)
var b strings.Builder
b.Grow(estimatedSize)
for _, p := range parts {
    b.WriteString(p)
}
s := b.String()

// ALSO GOOD: known small set
s := strings.Join(parts, "")
```

### 9.6 Byte slices vs strings

```go
// Convert with care; both directions copy
b := []byte(s)
s := string(b)

// Avoid round-tripping
return string([]byte(s))  // pointless
```

`string` ↔ `[]byte` conversions copy. Use `unsafe` tricks (`unsafe.String`, `unsafe.Slice`) only when profiling shows it; the gain is small and the risk of aliasing bugs is real.

### 9.7 Slice subtleties

```go
// Slice retains underlying array; can prevent GC of large array
big := make([]byte, 1<<30)
small := big[:10]  // small references the 1GB array; big won't be freed

// Fix: copy
small := append([]byte(nil), big[:10]...)
```

A common memory leak. If you're slicing a tiny part out of a huge allocation, copy.

### 9.8 Map iteration order is randomized

```go
// Each iteration order may differ, even on same data
for k, v := range m {
    ...
}
```

Don't depend on order. If you need stability, sort the keys.

---

## 10. Slices, Maps, Strings — Gotchas

### 10.1 The append aliasing trap

```go
a := []int{1, 2, 3, 4}
b := a[:2]
c := append(b, 5)   // b's cap is 4 (shared with a); writes index 2
fmt.Println(a)      // [1, 2, 5, 4] — surprise mutation
```

`append` may or may not allocate, depending on cap. Defensive copy is safest when the original might be modified later.

### 10.2 Don't return shared slices from APIs

```go
// BAD: caller can mutate internal state
func (s *Service) Items() []Item { return s.items }

// GOOD: defensive copy or read-only contract
func (s *Service) Items() []Item {
    out := make([]Item, len(s.items))
    copy(out, s.items)
    return out
}
```

Or: explicitly document "do not modify."

### 10.3 nil slice vs empty slice

```go
var a []int           // nil
b := []int{}          // non-nil, length 0
c := make([]int, 0)   // non-nil, length 0

// All three: len() == 0, range works the same.
// JSON marshaling differs: nil → "null", empty → "[]"
```

For most purposes, treat them the same. Watch for JSON serialization differences in API responses.

### 10.4 Map of struct values — can't address fields

```go
type Point struct { X, Y int }
m := map[string]Point{"a": {1, 2}}
m["a"].X = 5  // COMPILE ERROR: can't take address of map element

// Fix: replace whole value
p := m["a"]
p.X = 5
m["a"] = p

// Or: use pointer values
m := map[string]*Point{"a": {1, 2}}
m["a"].X = 5  // works
```

### 10.5 String iteration is rune-based with `range`

```go
s := "héllo"
for i, r := range s {
    // i is byte index, r is rune
}
// vs byte iteration
for i := 0; i < len(s); i++ {
    b := s[i]  // byte
}
```

`range` on string yields runes (Unicode code points). For multi-byte UTF-8 chars, `i` jumps. Index-based access yields bytes.

### 10.6 Avoid `delete` in `range`-over-map without care

```go
// SAFE: delete during range is allowed by spec
for k := range m {
    if shouldDelete(k) {
        delete(m, k)
    }
}
```

Allowed but be aware: future iterations may or may not see new entries added during iteration (undefined). Only delete is well-defined.

---

## 11. Struct Design

### 11.1 Field alignment for memory efficiency

```go
// BAD: 24 bytes due to padding
type X struct {
    a bool   // 1 byte + 7 padding
    b int64  // 8 bytes
    c bool   // 1 byte + 7 padding
}

// GOOD: 16 bytes
type X struct {
    b int64  // 8 bytes
    a bool   // 1 byte
    c bool   // 1 byte (+6 padding)
}
```

Order fields **largest to smallest**. Tools: `fieldalignment` linter. At MAANG scale with millions of struct instances, this matters.

### 11.2 Zero values that mean something

```go
// BAD: zero value is a mid-state
type Server struct {
    started bool
    port    int
}

// GOOD: zero value is sensible / unusable
type Server struct {
    cfg    Config       // zero-value Config means "unconfigured"; methods reject it
}
```

Aim to make `var s Server` either fully-functional or unambiguously invalid. The `bytes.Buffer` zero value is a usable buffer; `sync.Mutex` zero value is unlocked.

### 11.3 Constructors

```go
// New returns a *Server. Convention: New, NewClient, NewServer.
func New(cfg Config) (*Server, error) {
    if cfg.Addr == "" {
        return nil, errors.New("server: addr required")
    }
    return &Server{cfg: cfg}, nil
}
```

When zero value isn't safe, provide a constructor. Don't expose unsafe public types.

### 11.4 Functional options

```go
type ServerOption func(*Server)

func WithTimeout(d time.Duration) ServerOption {
    return func(s *Server) { s.timeout = d }
}

func New(opts ...ServerOption) *Server {
    s := &Server{timeout: defaultTimeout}
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// Usage
s := New(WithTimeout(5*time.Second), WithMaxConns(100))
```

Use **only** when:
- 5+ optional configuration params.
- Backward compatibility important (adding new option doesn't break existing callers).

For 1–3 params, just take a `Config` struct. Functional options are over-used.

### 11.5 Embedding for composition

```go
type Logger struct { ... }
func (l *Logger) Log(msg string) { ... }

type Server struct {
    *Logger  // embedded
    addr string
}

// Server now has Log() method via promotion
s := &Server{Logger: &Logger{}, addr: ":8080"}
s.Log("starting")  // works
```

Powerful but easy to overuse:
- **Pro**: composition without inheritance ceremony.
- **Con**: if both embedded and outer have a method with same name, shadow rules can confuse. Also, embedded fields appear in JSON/yaml at top level — surprising sometimes.

### 11.6 Tagged fields (struct tags)

```go
type User struct {
    ID    int    `json:"id" db:"user_id"`
    Name  string `json:"name" db:"name" validate:"required"`
    Email string `json:"email,omitempty"`
}
```

- Tags are strings; typos are silent. Use a linter (`govet -structtag`).
- Don't overload tags with too many concerns. If you have JSON, DB, validator, OpenAPI, gRPC tags all on one struct, consider separating into per-layer types.

---

## 12. Generics (Go 1.18+)

Generics arrived late and are still finding their place.

### 12.1 When to use generics

```go
// GOOD: reusable container/algorithm
func Map[T, U any](s []T, f func(T) U) []U {
    out := make([]U, len(s))
    for i, v := range s {
        out[i] = f(v)
    }
    return out
}

// GOOD: type-safe collection
type Set[T comparable] map[T]struct{}

// GOOD: removing duplicate code across types
func Min[T constraints.Ordered](a, b T) T { if a < b { return a } else { return b } }
```

### 12.2 When NOT to use generics

```go
// BAD: generic for one type — just write it concretely
func ProcessUser[T User](u T) { ... }

// BAD: any-as-generic — defeats purpose
func Foo[T any](v T) { fmt.Println(v) }  // same as fmt.Println(any)

// BAD: forced into a constraint that's just "anything with Method()"
type Doer interface { Do() }
func ForAll[T Doer](xs []T) { ... }  // could just be: func ForAll(xs []Doer)
```

### 12.3 The generic-vs-interface trade

- **Interface**: dynamic dispatch (runtime), boxing (heap allocation per call), simpler.
- **Generic**: monomorphized (mostly, with shape-based stenciling), no boxing in many cases, more code.

For hot paths where allocation matters, generics can help. For most code, interfaces are fine.

### 12.4 Generic type constraints

```go
import "golang.org/x/exp/constraints"

type Number interface {
    constraints.Integer | constraints.Float
}

func Sum[T Number](xs []T) T {
    var sum T
    for _, x := range xs { sum += x }
    return sum
}
```

The `constraints` package (now `golang.org/x/exp/constraints` or hand-written) gives you `Ordered`, `Integer`, `Float`, etc.

### 12.5 Don't generalize until you have 2+ uses

The first concrete implementation is rarely the right shape for a generic. Wait for a pattern.

---

## 13. Testing

Go's `testing` package is minimal by design. Good Go tests are simple, fast, and table-driven.

### 13.1 Table-driven tests

```go
func TestParse(t *testing.T) {
    cases := []struct {
        name    string
        input   string
        want    Result
        wantErr bool
    }{
        {"empty", "", Result{}, true},
        {"basic", "abc", Result{V: "abc"}, false},
        {"with spaces", "  abc  ", Result{V: "abc"}, false},
    }

    for _, c := range cases {
        t.Run(c.name, func(t *testing.T) {
            got, err := Parse(c.input)
            if (err != nil) != c.wantErr {
                t.Fatalf("err = %v, wantErr = %v", err, c.wantErr)
            }
            if got != c.want {
                t.Errorf("got %v, want %v", got, c.want)
            }
        })
    }
}
```

This is the canonical pattern. Each case is named (subtest), independently runnable (`go test -run TestParse/basic`).

### 13.2 No assertion library (mostly)

Go's stdlib `testing` deliberately has no `assertEqual`. Use `if/t.Errorf`. Reasons:

- Failures show context naturally.
- No magic stack manipulation.
- `t.Errorf` reports continuing failures; `t.Fatalf` stops.

Some teams use `testify` for assertions. It works, but in stdlib-purist Go shops you'll see plain `if`. Either is defensible.

### 13.3 Parallel tests

```go
func TestX(t *testing.T) {
    t.Parallel()
    // ...
}

func TestY(t *testing.T) {
    t.Parallel()
    cases := []struct { ... }{...}
    for _, c := range cases {
        c := c  // capture (Go < 1.22)
        t.Run(c.name, func(t *testing.T) {
            t.Parallel()
            // ...
        })
    }
}
```

`t.Parallel()` lets the test run concurrently with other parallel tests. Use unless tests share mutable state.

### 13.4 Test fixtures via `testdata/`

```
mypackage/
├── main.go
├── main_test.go
└── testdata/
    ├── input1.json
    └── expected1.json
```

`testdata/` is a special dir Go ignores during `go build`. Put fixture files there; read with `os.ReadFile("testdata/input1.json")`. Cleaner than embedded literal blobs.

### 13.5 Mocking — minimal, interface-based

```go
type userRepo interface {
    Save(ctx context.Context, u User) error
    Find(ctx context.Context, id ID) (User, error)
}

type stubRepo struct {
    saveErr error
    user    User
}

func (s *stubRepo) Save(ctx context.Context, u User) error { return s.saveErr }
func (s *stubRepo) Find(ctx context.Context, id ID) (User, error) { return s.user, nil }
```

Hand-written stubs are usually clearer than mocks generated by `gomock` / `mockery`. Use code generation when you have many methods or complex expectations; otherwise stubs win.

### 13.6 Avoid `init` in tests

`TestMain(m *testing.M)` is the right hook for setup/teardown:

```go
func TestMain(m *testing.M) {
    setup()
    code := m.Run()
    teardown()
    os.Exit(code)
}
```

### 13.7 Integration vs unit tests

```go
//go:build integration

package mypackage_test
// run with: go test -tags=integration
```

Use build tags to separate slow integration tests from fast unit tests. CI runs all; local dev runs unit by default.

### 13.8 Fuzzing (Go 1.18+)

```go
func FuzzParse(f *testing.F) {
    f.Add("hello")
    f.Fuzz(func(t *testing.T, s string) {
        out, err := Parse(s)
        if err == nil && out.Value != s {
            t.Errorf("round-trip: in=%q out=%q", s, out.Value)
        }
    })
}
```

Run with `go test -fuzz=FuzzParse`. Use for parsers, codecs, security-sensitive code. Found bugs in stdlib repeatedly.

### 13.9 Use of `t.Helper()`

```go
func mustOpen(t *testing.T, path string) *os.File {
    t.Helper()  // failures point to caller, not this function
    f, err := os.Open(path)
    if err != nil {
        t.Fatalf("open %q: %v", path, err)
    }
    return f
}
```

Mark test helpers; failure line numbers point to the caller, not the helper.

---

## 14. Benchmarking and Profiling

### 14.1 Benchmarks

```go
func BenchmarkParse(b *testing.B) {
    input := generateInput()
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        _, _ = Parse(input)
    }
}
```

- `b.ResetTimer()` after setup.
- `b.ReportAllocs()` to surface allocation count.
- `go test -bench=. -benchmem -count=10` for stable comparisons.
- Use `benchstat` to compare two runs statistically.

### 14.2 Profiling

```bash
# CPU profile
go test -bench=. -cpuprofile=cpu.prof
go tool pprof cpu.prof

# Memory profile
go test -bench=. -memprofile=mem.prof
go tool pprof mem.prof

# Live profile via net/http/pprof
import _ "net/http/pprof"
go func() { http.ListenAndServe(":6060", nil) }()
# then: go tool pprof http://localhost:6060/debug/pprof/profile
```

Always-on `net/http/pprof` in production (behind admin-network ACL) is invaluable for live debugging.

### 14.3 Continuous profiling

In MAANG-grade Go services: continuous profiling via Pyroscope, Parca, or vendor tools (Datadog, Lightstep). Sample CPU/heap/goroutines from production fleet; query later. Catches leaks and regressions before they page.

---

## 15. Documentation

In Go, **docs are part of the API**. They're rendered by `pkg.go.dev` and `go doc`.

### 15.1 Doc comments

```go
// Server handles incoming HTTP requests for the user service.
// It must be created via [New]; the zero value is not usable.
type Server struct { ... }

// Start begins serving and blocks until ctx is canceled or an error occurs.
// It returns nil on graceful shutdown via ctx, or the underlying error otherwise.
func (s *Server) Start(ctx context.Context) error { ... }
```

Rules:
- Public identifier? Doc comment required.
- Comment starts with the identifier name.
- First sentence summarizes; rest gives detail.
- `[Name]` syntax (Go 1.19+) creates docs hyperlinks.
- Examples (functions named `ExampleFoo`) are runnable docs.

### 15.2 Examples

```go
func ExampleParse() {
    out, _ := Parse("abc")
    fmt.Println(out.Value)
    // Output: abc
}
```

`go test` runs examples; the `// Output:` comment is verified. They appear on `pkg.go.dev` and are the best documentation for usage.

### 15.3 Avoid useless comments

```go
// BAD: comment restates the code
// userID is the user ID
userID := getUserID()

// GOOD: comment adds context
// userID is fetched eagerly because all subsequent paths need it.
// Falls back to the request's anonymous-session ID for unauthenticated callers.
userID := getUserID(ctx, req)
```

Comments explain **why**, not **what**.

### 15.4 README and design docs

A package should have:
- A doc.go with package overview.
- A README at the module root explaining purpose, install, examples.
- For non-trivial features, a design doc (separate, often in a `/docs` folder or wiki).

---

## 16. Project Layout

There's no "official" Go project layout, but conventions exist.

### 16.1 Typical layout

```
myservice/
├── cmd/
│   ├── server/main.go         # binary entry point
│   └── migrate/main.go        # secondary binary
├── internal/                  # package(s) not importable outside this module
│   ├── auth/
│   ├── user/
│   ├── store/
│   └── server/
├── pkg/                       # rarely needed; for libraries others import
├── api/
│   └── proto/                 # Protobuf / gRPC definitions
├── deploy/                    # Helm charts, k8s manifests, Dockerfile
├── scripts/                   # build/release scripts
├── go.mod
├── go.sum
├── README.md
└── Makefile
```

### 16.2 Things to avoid

- `pkg/` for everything (the famous "fake `pkg/`"). Use it only if you genuinely export libraries.
- Mirroring Java/Python: `models/`, `controllers/`, `services/`. Organize by feature.
- A single deeply-nested package; flatten when possible.

### 16.3 Multi-module repositories

For very large monorepos, multi-module (`go.mod` per service) is sometimes used. Beware: adds complexity (replace directives, version coordination). Single-module is simpler if your repo can support it.

---

## 17. Dependency Management

### 17.1 go.mod and semver

```
module github.com/myorg/myservice

go 1.22

require (
    github.com/google/uuid v1.5.0
    go.opentelemetry.io/otel v1.21.0
)
```

- `require` lines pin versions.
- `go.sum` stores hashes for verification.
- `go mod tidy` reconciles.

### 17.2 Choose dependencies carefully

Each dependency:
- Adds attack surface.
- Adds binary size.
- Couples your project to its lifecycle.
- Is a maintenance burden (updates, audits).

Go culture is famously dep-light. Stdlib + 5–10 well-chosen libs covers most apps. **Question every new dep.** A "great library" with one maintainer at midnight commits is a future incident.

### 17.3 Vendoring

```
go mod vendor
```

Creates `vendor/`. Builds use it. Useful in air-gapped environments or for reproducibility under untrusted-proxy CI.

### 17.4 Major version upgrades

Go enforces import path versioning for v2+: `import "github.com/x/y/v2"`. Annoying but explicit. Read changelogs; v2 means breaking changes.

### 17.5 Replace directives

```
replace github.com/x/y => ../local-fork
```

For local development or temporary forks. **Don't ship to production with replace directives** — confusing for consumers and signals work-in-progress.

---

## 18. Logging and Observability

### 18.1 Use slog (Go 1.21+) or zerolog/zap

```go
import "log/slog"

logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
logger.Info("user created", "user_id", uid, "email", email)
```

- `slog`: stdlib structured logger.
- `zap` / `zerolog`: faster, more features. Industry-standard before slog.
- Avoid the bare `log` package for production services.

### 18.2 Structured logging conventions

```go
slog.Info("request handled",
    "method", r.Method,
    "path", r.URL.Path,
    "status", w.Status(),
    "duration_ms", dur.Milliseconds(),
    "trace_id", traceID,
    "user_id", userID,
)
```

- Key/value pairs, not `fmt.Sprintf` strings.
- Consistent key names across services (`trace_id`, `user_id`, `request_id`).
- Don't log secrets, tokens, PII.
- Sample at hot paths; full at error.

### 18.3 Log levels

| Level | When |
|---|---|
| `Debug` | Detailed flow; off in production |
| `Info` | Normal events worth recording |
| `Warn` | Concerning but recoverable |
| `Error` | Failed operation; needs attention |
| `Fatal` (slog: `LevelError + os.Exit`) | Process should die |

Avoid `Fatal` in libraries. Caller decides whether to die.

### 18.4 Tracing (OpenTelemetry)

```go
import "go.opentelemetry.io/otel"

ctx, span := tracer.Start(ctx, "ProcessOrder")
defer span.End()

span.SetAttributes(attribute.String("order.id", orderID))
if err != nil {
    span.RecordError(err)
    span.SetStatus(codes.Error, err.Error())
}
```

Propagate `ctx` everywhere; tracing rides on it. Enable across services for end-to-end visibility.

### 18.5 Metrics

```go
import "github.com/prometheus/client_golang/prometheus"

var requestsTotal = prometheus.NewCounterVec(
    prometheus.CounterOpts{Name: "requests_total"},
    []string{"path", "status"},
)
requestsTotal.WithLabelValues("/users", "200").Inc()
```

Cardinality discipline: don't put `user_id` as a label (millions of values). Path, status, method, region — bounded sets.

---

## 19. Configuration

### 19.1 Config struct + environment

```go
type Config struct {
    Addr       string        `env:"ADDR" default:":8080"`
    Timeout    time.Duration `env:"TIMEOUT" default:"30s"`
    DBURL      string        `env:"DB_URL,required"`
    LogLevel   string        `env:"LOG_LEVEL" default:"info"`
}

func Load() (Config, error) { ... }
```

- Single struct; one source of truth for what's configurable.
- Defaults in code or in env spec.
- `required` on critical config; fail fast on missing.
- 12-factor: config from env, not files committed to repo.

### 19.2 Don't read env from arbitrary places

```go
// BAD
addr := os.Getenv("ADDR")  // scattered through code

// GOOD
addr := cfg.Addr  // loaded once at startup
```

Reading env at point of use makes testing painful and makes config invisible.

### 19.3 Validate config at startup

```go
func (c *Config) Validate() error {
    if c.Addr == "" { return errors.New("addr required") }
    if c.Timeout <= 0 { return errors.New("timeout must be positive") }
    return nil
}
```

Fail fast at startup, not on the first request that hits the bad config.

### 19.4 Don't log secrets in config dumps

```go
// BAD: logs full config including DB credentials
log.Printf("config: %+v", cfg)

// GOOD: redact
func (c Config) String() string {
    return fmt.Sprintf("Config{Addr:%s, DB:<redacted>}", c.Addr)
}
```

Implement `Stringer` to redact secrets, or split secrets into a separate secret-tagged struct.

---

## 20. APIs and Protocols

### 20.1 HTTP handlers

```go
// GOOD: handler is small; logic is delegated.
func (s *Server) handleCreateUser(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid request", http.StatusBadRequest)
        return
    }
    if err := req.Validate(); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    user, err := s.users.Create(ctx, req)
    if err != nil {
        s.handleError(w, err)
        return
    }
    writeJSON(w, http.StatusCreated, user)
}
```

- Handler does request parsing + delegation, not business logic.
- Use `r.Context()` for cancellation propagation.
- Centralize error responses (`s.handleError`).

### 20.2 gRPC vs HTTP/JSON

| Use HTTP/JSON when | Use gRPC when |
|---|---|
| Public API, browser/mobile clients | Internal service-to-service |
| Caller variety | Polyglot fleet, generated clients |
| Simplicity matters | Performance matters |

Don't expose gRPC to browsers (use grpc-web or REST gateway). Don't internal-service via JSON when proto is available.

### 20.3 Idempotency

For PUT/DELETE: idempotent by HTTP semantics. For POST: support `Idempotency-Key` header. Pattern from §7 of the transactions doc applies.

### 20.4 Versioning

```
GET /v1/users/123
GET /v2/users/123
```

URL-versioning is simple and visible. Header-versioning (`Accept: application/vnd.myorg.v2+json`) is flexible but harder to debug.

---

## 21. Performance Idioms

### 21.1 Profile first, optimize after

Until you've profiled, every "optimization" is a guess. The slow part is rarely where you think. `pprof` first, hand-tune second.

### 21.2 Avoid premature interfaces

Concrete types are faster than interfaces (no dynamic dispatch). Add interface only when polymorphism is needed.

### 21.3 Reduce allocations in hot paths

- Pre-allocate slices and maps with `make(..., N)`.
- Use `sync.Pool` for short-lived objects of consistent shape.
- Pass `[]byte` rather than `string` if you'll modify in-place.
- Use `strings.Builder`, not `+`.
- Prefer fixed-size arrays over slices when size is known.

### 21.4 Avoid reflection in hot paths

`reflect`, `encoding/json`, `text/template` use reflection. They are slow per call. For high-RPS encoding, hand-write or use `easyjson`/`ffjson` codegen.

### 21.5 Use fast paths

```go
// GOOD: shortcut common case
if len(s) == 0 { return "" }
if len(s) == 1 { return s }
// general case below
```

Common-case optimization beats general-case sophistication for hot paths.

### 21.6 `goroutine` is not always free

Spawning a goroutine costs ~2KB stack initially (grows). Spawning millions for short tasks is wasteful. Use a worker pool.

### 21.7 Lock-free vs lock-based

Atomic and channel-based concurrency can outperform mutex-based for small contention. But: lock-free code is harder to reason about. Use mutex by default; reach for lock-free only with profiling proof.

### 21.8 NUMA and CPU affinity

For latency-critical Go services on bare metal:
- `GOMAXPROCS` matched to allocated CPUs.
- Avoid container CPU limits that suspend the process.
- Pin to NUMA node if memory access matters.
- Tune GC (`GOGC`, `GOMEMLIMIT`).

---

## 22. Tools and Linting

### 22.1 The non-negotiable set

| Tool | Purpose |
|---|---|
| `gofmt` | Formatting (always run) |
| `goimports` | Formatting + import organization |
| `go vet` | Static analysis |
| `staticcheck` | Stronger static analysis |
| `golangci-lint` | Aggregator running 50+ linters |
| `errcheck` | Catches unchecked errors |
| `gosec` | Security issues |
| `govulncheck` | Known-vulnerable dependencies |

Add to CI; fail builds on violations.

### 22.2 golangci-lint config

```yaml
# .golangci.yml
linters:
  enable:
    - errcheck
    - gosec
    - govet
    - staticcheck
    - unused
    - ineffassign
    - prealloc
    - errorlint
    - bodyclose
    - gocyclo
```

Tune thresholds (`gocyclo: max: 15`); enable enough to be useful, not so many that the team ignores them.

### 22.3 Pre-commit hook

```bash
#!/bin/sh
gofmt -l -w . || exit 1
go vet ./... || exit 1
golangci-lint run || exit 1
go test -short ./... || exit 1
```

Catches issues before review. Lighter version in pre-commit; full version in CI.

### 22.4 Code generation

```go
//go:generate stringer -type=Status
type Status int
```

Run `go generate ./...` to regenerate. Keeps generated code visible in repo. Common uses: `stringer`, `protoc-gen-go`, `mockery`.

---

## 23. Code Review Checklist (Staff-Level)

What I look for in a Go PR review:

### 23.1 Correctness

- [ ] All errors checked and either handled or wrapped with context (`%w`).
- [ ] Resource cleanup: `defer Close()`, `defer cancel()`.
- [ ] Goroutines have clear ownership and termination.
- [ ] No data races (run with `-race`).
- [ ] No silent type assertions (`x.(T)` without `, ok`).
- [ ] Context propagated through I/O paths.
- [ ] No `panic` in library code.

### 23.2 Idiom

- [ ] `gofmt` clean.
- [ ] Naming: short for local, descriptive for exported.
- [ ] Methods consistent in receiver type (all pointer or all value).
- [ ] Interfaces small; defined where consumed.
- [ ] No `interface{}` / `any` at API boundaries unless justified.
- [ ] No `init()` with side effects.

### 23.3 Tests

- [ ] Table-driven where applicable.
- [ ] Subtests with `t.Run`.
- [ ] No panics in tests masking real failures.
- [ ] `t.Parallel()` where safe.
- [ ] Tests run quickly (< 1s typical).
- [ ] Edge cases covered (empty input, errors, boundaries).

### 23.4 Performance

- [ ] No accidental N² loops.
- [ ] Pre-allocated slices/maps where size is known.
- [ ] No `string + string` in loops.
- [ ] Hot paths checked with profile if performance matters.
- [ ] Channel use sane (no panics on closed/full).

### 23.5 Observability

- [ ] Critical paths have metrics.
- [ ] Errors logged at the boundary, not at every layer.
- [ ] Logs don't contain secrets/PII.
- [ ] Traces propagated where applicable.

### 23.6 Documentation

- [ ] Exported identifiers have doc comments.
- [ ] Comments explain *why*, not *what*.
- [ ] Examples for non-obvious APIs.
- [ ] README updated if public API changed.

### 23.7 Architecture

- [ ] Package boundaries make sense.
- [ ] No new circular import attempts.
- [ ] No new "util" or "common" package.
- [ ] Dependencies justified; no random new third-party libs.
- [ ] Backward compatibility considered for public APIs.

---

## 24. Anti-Patterns — Staff-Level Red Flags

### 24.1 Over-engineering

```go
// BAD
type IUserService interface { ... }
type UserService struct { ... }  // single impl
type UserServiceFactory interface { ... }
type UserServiceFactoryImpl struct { ... }
type UserServiceProvider struct { ... }
```

Java reflex. Go doesn't need factories of factories.

### 24.2 "Util" package

```
package util  // catch-all functions
```

Sign of design rot. Each function belongs in a package with a real name.

### 24.3 Empty interface for "flexibility"

```go
type Cache map[string]any  // no type safety
```

Designed flexibility on day 1 = no design at all. Use concrete or generic types.

### 24.4 Hiding errors

```go
result, _ := doThing()
```

Always justify. If "this never errors", say so in a comment and `_` is acceptable. Otherwise, handle.

### 24.5 Shadowing variables

```go
err := doA()
if err != nil { ... }
err := doB()  // shadow; previous err lost
```

Vet catches. Use `=`, not `:=`, when reusing a variable.

### 24.6 String-typed APIs

```go
// BAD
func Connect(driver string) (*DB, error) {
    switch driver {
        case "postgres": ...
        case "mysql": ...
    }
}

// GOOD
type Driver int
const (DriverPostgres Driver = iota; DriverMySQL)
func Connect(d Driver) (*DB, error)
```

Stringly-typed APIs are unsafe.

### 24.7 Nil pointer maps

```go
var m map[string]int
m["a"] = 1  // PANIC: assignment to nil map
```

Always `make(map[K]V)` before writing.

### 24.8 Goroutines without exit

```go
go func() {
    for {
        time.Sleep(time.Second)
        doWork()
    }
}()
```

No way to stop it. Always include cancellation.

### 24.9 sync.WaitGroup misuse

```go
// BAD: Add inside goroutine; race
go func() {
    wg.Add(1)
    defer wg.Done()
    work()
}()

// GOOD: Add before goroutine starts
wg.Add(1)
go func() {
    defer wg.Done()
    work()
}()
```

### 24.10 Defer in a loop

```go
// BAD: defers stack up; resources held until function returns
for _, file := range files {
    f, _ := os.Open(file)
    defer f.Close()  // 1000 deferred closes
    // ...
}

// GOOD: explicit close per iteration
for _, file := range files {
    func() {
        f, _ := os.Open(file)
        defer f.Close()
        // ...
    }()
}
```

### 24.11 Pre-Go-1.22 loop variable capture

```go
// BAD: pre-1.22 — i shared across closures
for i := 0; i < 10; i++ {
    go func() { fmt.Println(i) }()  // all print 10
}

// GOOD: capture
for i := 0; i < 10; i++ {
    i := i
    go func() { fmt.Println(i) }()
}
```

Go 1.22 fixed this; loop variables are per-iteration. But many codebases pre-date 1.22; recognize the bug.

### 24.12 Returning the receiver mutated

```go
// BAD: Builder pattern that mutates and returns
func (s *Server) WithTimeout(d time.Duration) *Server {
    s.timeout = d
    return s
}

// Caller doesn't expect mutation; aliasing bug.
```

For builder, return new struct or use functional options.

### 24.13 Big functions with deep nesting

```go
func handle(req Request) error {
    if cond1 {
        if cond2 {
            if cond3 {
                ...nested 6 levels deep...
            }
        }
    }
}
```

Use early returns to flatten:
```go
if !cond1 { return errA }
if !cond2 { return errB }
if !cond3 { return errC }
// happy path
```

### 24.14 Hidden globals

```go
var defaultClient = &http.Client{...}  // package-level mutable state

func Do(...) { defaultClient.Get(...) }
```

Globals are state; state is harder to test, harder to reason about, harder to swap. Prefer dependency injection.

### 24.15 time.Now() everywhere

```go
func handle() {
    now := time.Now()
    // ...
}
```

Untestable: tests can't control time. Inject a clock:
```go
type Clock interface { Now() time.Time }

func handle(clock Clock) {
    now := clock.Now()
}
```

For most cases this is overkill; for time-sensitive logic (TTLs, schedulers), it's worth it.

### 24.16 "Clever" use of channels for non-concurrency

```go
// BAD: using a channel as a "queue" but never concurrent
ch := make(chan Item, 100)
for _, item := range items { ch <- item }
close(ch)
for item := range ch { process(item) }
// equivalent to: for _, item := range items { process(item) }
```

Channels are coordination; if there's no concurrency, use a slice.

### 24.17 Over-using `any`

```go
func ProcessAll(items []any) {
    for _, item := range items {
        switch v := item.(type) {
        case User: ...
        case Order: ...
        }
    }
}
```

Type-switching on `any` is a sign you've lost type structure. Refactor to a real polymorphism or generics.

### 24.18 fmt.Sprintf for hot paths

`fmt.Sprintf` allocates and uses reflection. For hot paths, use `strconv.AppendInt`, `strconv.AppendFloat`, etc., into a `bytes.Buffer`. Saves significant time at scale.

### 24.19 Logging at hot paths without sampling

```go
// BAD: every request logged at INFO; logs overflow
slog.Info("request", "path", r.URL.Path)
```

Sample, or log at DEBUG. At 10K RPS, naive INFO logging is a self-DoS.

### 24.20 Initialization races

```go
// BAD
func GetService() *Service {
    if svc == nil { svc = newService() }
    return svc
}
```

Race between the nil-check and assignment. Use `sync.Once` or initialize at package init (with care).

---

## 25. Mental Models a Staff Engineer Carries

A condensed reference of mental models that produce correct staff-level reasoning quickly:

1. **Errors are values.** Every error is a decision point. Boilerplate isn't a burden; it's a feature.

2. **Goroutines have owners.** No fire-and-forget. Every goroutine has a function that knows when it ends.

3. **Cancellation is cooperative.** `context.Cancel()` signals; code must check.

4. **Panics are for programmer errors.** Recoverable conditions return `error`.

5. **Interfaces are tiny.** "The bigger the interface, the weaker the abstraction."

6. **Define interfaces where consumed, not where implemented.**

7. **Accept interfaces, return structs.**

8. **Composition > inheritance** because Go has no inheritance. Embedding is the closest analogue.

9. **Channels coordinate; mutexes protect.** Pick the right tool. Channels aren't always better.

10. **The zero value should be usable** when it can be. `bytes.Buffer{}`, `sync.Mutex{}` set the standard.

11. **Pointer vs value: large/mutable → pointer; small/immutable → value.** Be consistent across methods.

12. **Profile before optimizing.** Most "performance fixes" are guesses without data.

13. **A little copying is better than a little dependency.** Don't create coupling for DRY.

14. **gofmt resolves all formatting debates.** Don't argue style; run the tool.

15. **Test what users see, not internals.** Black-box tests survive refactors.

16. **Doc comments are part of the API.** Render on `pkg.go.dev`. Treat them seriously.

17. **Don't pre-emptively make things generic or interface-bound.** Wait for the second use.

18. **Concrete types beat interfaces in hot paths.** Indirection is cost.

19. **Allocations matter at scale.** Pre-allocate; pool; avoid string/byte round-trips.

20. **Tools shape culture.** `gofmt`, `go vet`, `staticcheck`, `golangci-lint` are not optional; they are the language.

21. **Idiom is documentation.** Code that follows community conventions reads itself; clever code requires interpretation.

22. **The compiler is your friend.** Lean on type errors, escape analysis, race detector. Run `-race` in CI.

23. **Boring code wins.** A 5-year-old service whose maintenance is "boring" is the highest engineering accomplishment.

24. **Hide complexity behind small interfaces; never the other way.** The package boundary is your unit of design.

25. **Go is not Python, Java, Rust, or Haskell.** Stop trying to make it those. Internalize Go-think.

---

## 26. The Go Engineer's Daily Practice

The habits that make a staff-level Go engineer:

### 26.1 Daily

- Run `gofmt`, `go vet`, `go test ./...` before every commit.
- Read the diff before opening a PR; would a stranger understand it?
- Run `-race` whenever touching concurrent code.

### 26.2 Per PR

- Keep PRs small (<400 lines is a healthy default).
- Self-review before requesting review.
- Update docs alongside code.
- Add tests for new behavior; regression tests for fixed bugs.

### 26.3 Per design

- Sketch the package boundary first.
- Define interfaces from the consumer side.
- Identify the zero-value contract.
- Plan goroutine ownership.
- Plan error handling semantics.

### 26.4 Per quarter

- Profile production: hot paths, allocations, goroutine counts.
- Audit dependencies: prune, upgrade, scan for vulns.
- Run `govulncheck`.
- Review and update linter rules.
- Walk the codebase for anti-patterns; fix before they spread.

---

## 27. Closing Notes

Clean Go code is not "clean code in Go" — it is its own genre.

The defining staff-level shift is recognizing that **idiom is not arbitrary**. Each Go convention exists because someone tried the alternative at scale and it hurt:

- Errors-as-values vs exceptions: because exceptions hide control flow at scale.
- Small interfaces: because big interfaces couple too much.
- No inheritance: because inheritance trees ossify designs.
- Capital-letter export: because explicit is better than `private`/`public` keyword fatigue.
- `gofmt`: because style debates are wasted human time.
- Concurrency-first: because concurrency-after is unsafe by construction.

Internalize these and code quality is largely automatic. Fight them and you build code that no one else in the Go community can comfortably maintain.

The art is knowing **when to break the rules** — and being able to articulate why. That ability separates senior from staff. It is earned by writing many lines of idiomatic code first.

> Companion docs in this folder:
> - `memoryLeaksInGo.md` — finding and fixing memory leaks
> - `handlingRaceConditionsInGo.md` — race detection and prevention
> - `deadlocksInGO.md` — concurrency deadlock patterns
> - `returningInGo.md` — return-value idioms

Together they form the working manual for Go engineering at MAANG-grade reliability.