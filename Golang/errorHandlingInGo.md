⏺ Staff-Level Design Paradigms for Error Handling in Go

---
Foundational Principle: Errors Are Values, Not Exceptions

Most language communities treat errors as exceptional control flow. Go treats them as **first-class values** that are part of the function's API surface. This means every error decision is a design decision — *what shape* it takes, *what data* it carries, *where it is wrapped*, and *who is allowed to compare it* are all permanent contracts with your callers.

The error interface itself is one method:

```go
type error interface {
    Error() string
}
```

That `Error() string` is for *humans reading logs*. It is **never** the comparison surface — never compare error strings, never `strings.Contains(err.Error(), "not found")`. The comparison surface is `errors.Is`, `errors.As`, and identity (`err == ErrSentinel`).

```go
// BAD: string-comparing errors is a fragile contract with no compiler help
// If the upstream library rephrases the message, your code breaks silently
if err != nil && strings.Contains(err.Error(), "no rows") {
    // ...
}

// GOOD: errors.Is matches by identity through the wrap chain
if errors.Is(err, sql.ErrNoRows) {
    // ...
}
```

Staff insight: the moment you compare an error's *string*, you have created an undocumented coupling between two packages that the type system cannot enforce. If the only stable thing about an error is "it exists," you cannot build any retry, fallback, or differentiated logging on top of it. **The shape of your error is a public API**; treat it with the seriousness you'd give a JSON schema.

---
1. Sentinel Errors — When, Where, and How

A **sentinel** is a singleton `var ErrX = errors.New(...)` exported by a package. Callers compare with `errors.Is`. This is the right tool when:

- The error is a stable, well-known **condition** the caller may want to branch on.
- There is no additional structured data the caller needs.
- The condition is a *contract* of the API, not an implementation detail.

```go
// GOOD: package-level sentinels, exported, named with Err* prefix
package userrepo

import "errors"

var (
    ErrNotFound      = errors.New("user not found")
    ErrAlreadyExists = errors.New("user already exists")
    ErrInvalidEmail  = errors.New("invalid email format")
)

func (r *Repo) Get(ctx context.Context, id string) (*User, error) {
    var u User
    err := r.db.QueryRowContext(ctx, query, id).Scan(&u.ID, &u.Name)
    if errors.Is(err, sql.ErrNoRows) {
        // Wrap with context, but the sentinel survives errors.Is checks
        return nil, fmt.Errorf("user %s: %w", id, ErrNotFound)
    }
    if err != nil {
        return nil, fmt.Errorf("scan user %s: %w", id, err)
    }
    return &u, nil
}

// Caller — clean branching
u, err := repo.Get(ctx, id)
switch {
case errors.Is(err, userrepo.ErrNotFound):
    return http.StatusNotFound
case err != nil:
    return http.StatusInternalServerError
}
```

### Sentinels — common mistakes

```go
// BAD: comparing with == fails through wrapping
if err == userrepo.ErrNotFound { // only works if err is literally that value
    // ...
}

// GOOD: errors.Is walks the wrap chain
if errors.Is(err, userrepo.ErrNotFound) {
    // ...
}

// BAD: defining the sentinel inside a function — re-allocated each call,
// not comparable by identity across calls
func (r *Repo) Get(id string) (*User, error) {
    notFound := errors.New("not found") // NEW value each call
    if missing {
        return nil, notFound
    }
}

// BAD: not exporting the sentinel callers need to match on
var errNotFound = errors.New("not found") // unexported — caller cannot use errors.Is
func Get(id string) error { return errNotFound }
```

Staff insight: sentinel errors are part of your **public API surface**. Adding a new sentinel is non-breaking; removing or renaming one is a breaking change. Document each sentinel in the package godoc with the conditions that produce it — callers will write `errors.Is` against your published list.

---
2. Typed Errors — When the Caller Needs Structured Data

Sentinels carry no payload. If the caller needs to know *the retry-after time*, *the field that was invalid*, *the conflicting record*, you need a typed error.

```go
// GOOD: typed error carrying actionable data
type ValidationError struct {
    Field   string
    Value   any
    Reason  string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed: field %q value %v: %s", e.Field, e.Value, e.Reason)
}

func (s *Service) CreateUser(ctx context.Context, in CreateUserInput) (*User, error) {
    if !emailRegex.MatchString(in.Email) {
        return nil, &ValidationError{
            Field:  "email",
            Value:  in.Email,
            Reason: "must match RFC 5322",
        }
    }
    // ...
}

// Caller — extract structured data with errors.As
var verr *ValidationError
if errors.As(err, &verr) {
    return Response{
        Code:    http.StatusBadRequest,
        Field:   verr.Field,
        Message: verr.Reason,
    }
}
```

### The two-pointer rule for errors.As

```go
// BAD: errors.As takes a NON-NIL POINTER TO POINTER (or to interface)
// This panics — you passed a value, not a pointer
var verr ValidationError
if errors.As(err, verr) { // PANIC: errors.As: second argument must be a non-nil pointer
    // ...
}

// BAD: pointer to value type — type assertion targets the wrong shape
var verr ValidationError
if errors.As(err, &verr) { // works only if your error type is ValidationError, not *ValidationError
    // ...
}

// GOOD: pointer to pointer — matches a *ValidationError
var verr *ValidationError
if errors.As(err, &verr) {
    fmt.Println(verr.Field)
}

// Rule of thumb: if Error() is defined on *T, declare var x *T and pass &x.
//                if Error() is defined on T (value receiver), declare var x T and pass &x.
```

### Sentinel vs typed — the picking heuristic

```
Use sentinel when:
├── Stable, known condition with no payload
├── Caller's only question is "did this specific thing happen?"
└── e.g., sql.ErrNoRows, io.EOF, context.Canceled

Use typed error when:
├── Caller needs structured fields off the error
├── Multiple variants of the same category exist
├── You want the error to carry observability (request ID, trace ID, etc.)
└── e.g., *url.Error, *json.SyntaxError, *net.OpError
```

```go
// You can combine both — typed error with a sentinel field, or
// typed error that wraps a sentinel:
type RateLimitError struct {
    RetryAfter time.Duration
    Cause      error // can wrap a sentinel: ErrRateLimitExceeded
}

func (e *RateLimitError) Error() string { return fmt.Sprintf("rate limit: retry %v", e.RetryAfter) }
func (e *RateLimitError) Unwrap() error { return e.Cause }

// Caller can ask BOTH questions:
if errors.Is(err, ErrRateLimitExceeded) {
    var rle *RateLimitError
    if errors.As(err, &rle) {
        time.Sleep(rle.RetryAfter)
    }
}
```

Staff insight: defining `Unwrap() error` on your typed error is what lets `errors.Is` traverse into it. Forget this and `errors.Is(err, mySentinel)` returns false even though the sentinel is "in there." The contract is: implement `Error()` for humans, `Unwrap()` (or `Unwrap() []error` since 1.20) for `errors.Is`/`errors.As`.

---
3. Error Wrapping — fmt.Errorf with %w

Every time an error crosses a function or layer boundary, it should be wrapped with **context** that says *what your code was trying to do*. The wrap chain becomes the stack trace.

```go
// BAD: returning err unchanged loses all context
// You'll get "sql: no rows" with no idea which call produced it
func (r *Repo) GetUserByEmail(email string) (*User, error) {
    var u User
    err := r.db.QueryRow(query, email).Scan(&u.ID, &u.Name)
    if err != nil {
        return nil, err // who am I? what was I doing?
    }
    return &u, nil
}

// BAD: %v breaks the wrap chain — errors.Is no longer works
func (r *Repo) GetUserByEmail(email string) (*User, error) {
    var u User
    if err := r.db.QueryRow(query, email).Scan(&u.ID, &u.Name); err != nil {
        return nil, fmt.Errorf("get user %s: %v", email, err) // %v not %w — wrap is dropped
    }
    return &u, nil
}

// GOOD: %w preserves the chain
func (r *Repo) GetUserByEmail(email string) (*User, error) {
    var u User
    if err := r.db.QueryRow(query, email).Scan(&u.ID, &u.Name); err != nil {
        return nil, fmt.Errorf("get user by email %s: %w", email, err)
    }
    return &u, nil
}
```

### The wrap-message style guide

Make wrap messages **lowercase**, **without trailing punctuation**, and **describing what the code was doing** — not what went wrong (the inner error already says that):

```go
// BAD: trailing punctuation, sentence case, redundant "failed to"
return fmt.Errorf("Failed to load user!: %w", err)

// BAD: redundant — "failed to load user: failed to scan: no rows"
return fmt.Errorf("failed to load user: %w", err)

// GOOD: concise verb phrase describing the operation
return fmt.Errorf("load user %s: %w", id, err)
```

The convention reads cleanly when concatenated:

```
load user u-123: query users: scan: no rows in result set
```

Each layer adds one phrase; reading the chain tells you exactly what was attempted, in order.

### When NOT to wrap

```go
// 1. Crossing a domain/API boundary where the inner error is leaking impl details
// The upstream database error is internal; you don't want it in HTTP responses
func (s *Service) GetUser(id string) (*User, error) {
    u, err := s.repo.GetByID(id)
    if errors.Is(err, sql.ErrNoRows) {
        // Convert internal error to public domain error — don't wrap the sql error
        return nil, ErrUserNotFound
    }
    if err != nil {
        // Wrap with context but the layer below is internal
        return nil, fmt.Errorf("service.GetUser: %w", err)
    }
    return u, nil
}

// 2. Wrapping a sentinel that the caller already knows about
// (debatable — some prefer to wrap for stack-trace value)
return ErrNotFound // bare return — caller does errors.Is(err, ErrNotFound)
```

### Multiple %w — Go 1.20+

You can wrap multiple errors:

```go
// 1.20+: multiple %w verbs allowed
return fmt.Errorf("transaction: commit %w, rollback %w", commitErr, rollbackErr)

// Or use errors.Join
return errors.Join(commitErr, rollbackErr)

// errors.Is/As walks the tree — both errors are matchable
if errors.Is(err, sql.ErrConnDone) {
    // matches if EITHER joined error is ErrConnDone
}
```

`errors.Join` is the cleaner API for the "I have a list of errors" case; `%w` × N for "two things happened during cleanup."

Staff insight: the wrap chain is your async stack trace. Don't break it with `%v`, don't lose it by returning bare `err` without context, don't bloat it with redundant "failed to" prefixes. A well-wrapped error reads like a path-traced sentence describing the operation that failed.

---
4. errors.Is vs errors.As vs == — The Comparison Surface

Three different questions, three different APIs:

```go
// errors.Is — "is this a kind of X?" (identity-based, walks chain)
if errors.Is(err, sql.ErrNoRows) {
    // err is, or wraps, sql.ErrNoRows
}

// errors.As — "can I extract an X from this?" (type-based, walks chain)
var perr *fs.PathError
if errors.As(err, &perr) {
    // perr is now usable: perr.Path, perr.Op, perr.Err
}

// == — "is this literally X?" (identity-only, NO walk)
if err == io.EOF {
    // works ONLY if err is unwrapped io.EOF
    // dangerous if anyone in the call chain wraps it
}
```

### Why `err == ErrFoo` is dangerous

```go
// Caller A does:
io.ReadAll(r) // returns io.EOF on clean EOF — bare, unwrapped

// Caller B does (in a custom reader):
return fmt.Errorf("buffered read: %w", io.EOF) // wraps

// If your code checks err == io.EOF, it works for A but silently breaks for B.
// Use errors.Is(err, io.EOF) — works for both.
```

### Custom Is/As methods

You can implement `Is(error) bool` and `As(any) bool` on your error type to override matching logic. Rarely needed, but powerful:

```go
type DBError struct {
    Code int // SQLSTATE
}

func (e *DBError) Error() string { return fmt.Sprintf("db error %d", e.Code) }

// All "23xxx" SQLSTATE codes are integrity violations — group them
var ErrIntegrityViolation = errors.New("integrity violation")

func (e *DBError) Is(target error) bool {
    if target == ErrIntegrityViolation {
        return e.Code >= 23000 && e.Code < 24000
    }
    return false
}

// Caller can ask the category question without enumerating codes:
if errors.Is(err, ErrIntegrityViolation) {
    return http.StatusConflict
}
```

Staff insight: think of `errors.Is`/`errors.As` as a tiny query language over the wrap chain. The chain is built up the call stack; the queries are run at decision points. Custom `Is` methods let you create *categories* (rate-limit-class, retryable-class, fatal-class) that callers can match cleanly.

---
5. errors.Join and Multi-Error Patterns

Sometimes one operation produces multiple errors:

- Bulk validation: collect all field errors, return them together.
- Cleanup paths: defer'd closes that all may fail.
- Concurrent fan-out: multiple goroutines, each may fail.

### errors.Join (Go 1.20+)

```go
// GOOD: 1.20 errors.Join
func validate(in CreateUserInput) error {
    var errs []error
    if in.Email == "" {
        errs = append(errs, &ValidationError{Field: "email", Reason: "required"})
    }
    if in.Age < 0 {
        errs = append(errs, &ValidationError{Field: "age", Reason: "must be >= 0"})
    }
    return errors.Join(errs...) // returns nil if errs is empty
}

// Caller can still match each with errors.Is/As
err := validate(in)
var verr *ValidationError
if errors.As(err, &verr) {
    // first matching ValidationError; loop with iteration if you need all
}
```

### Iterating all joined errors (1.20+)

```go
type multiErr interface {
    Unwrap() []error
}

func eachError(err error, fn func(error)) {
    fn(err)
    if me, ok := err.(multiErr); ok {
        for _, e := range me.Unwrap() {
            eachError(e, fn)
        }
    } else if u := errors.Unwrap(err); u != nil {
        eachError(u, fn)
    }
}
```

### The cleanup-defer pattern

```go
// BAD: only first defer error is reported; second is silently lost
func write(path string) (err error) {
    f, err := os.Create(path)
    if err != nil {
        return fmt.Errorf("create: %w", err)
    }
    defer f.Close() // close error swallowed
    defer f.Sync()  // sync error swallowed
    _, err = f.Write(data)
    return err
}

// GOOD: capture defer errors via named returns + join
func write(path string) (err error) {
    f, err := os.Create(path)
    if err != nil {
        return fmt.Errorf("create: %w", err)
    }
    defer func() {
        err = errors.Join(err, f.Sync(), f.Close())
    }()
    if _, werr := f.Write(data); werr != nil {
        return fmt.Errorf("write: %w", werr)
    }
    return nil
}
```

### Avoid `go.uber.org/multierr` and `hashicorp/go-multierror` in new code

Both pre-date `errors.Join`. New code should use `errors.Join` unless you have a strict reason to keep external deps.

Staff insight: errors aren't always strictly one-at-a-time. Bulk validation is the canonical "collect all" use case — `errors.Join` is the right tool. Don't return only the first error if the caller could fix more in one round trip.

---
6. Panic and Recover — Boundaries, Not Control Flow

Panic is not Go's try/catch. It is **for unrecoverable invariant violations** — programmer errors, not operational errors. The rule:

```
Use panic when:
├── A precondition the *programmer* must guarantee is violated
├── The program has reached a state from which it cannot continue safely
└── Examples: nil dereference, integer divide by zero, out-of-bounds

Use error when:
├── The condition is recoverable
├── The condition is caused by input/environment, not by a coding mistake
└── Examples: file missing, network down, validation failure
```

### Don't panic in library code

```go
// BAD: a library that panics on bad input forces every caller to defer/recover
package mylib

func Parse(s string) Result {
    if s == "" {
        panic("Parse: empty input") // caller has no way to handle this gracefully
    }
    // ...
}

// GOOD: return an error; let the caller decide
func Parse(s string) (Result, error) {
    if s == "" {
        return Result{}, errors.New("empty input")
    }
    // ...
}
```

### Recover only at boundaries

There are exactly four legitimate places for `recover`:

```
1. The top of each goroutine you spawn (otherwise: process crash)
2. The HTTP middleware that wraps your handlers
3. RPC server interceptors (gRPC, Twirp, etc.)
4. Plugin / user-code dispatch (where a third-party panic can't take you down)
```

### The "every goroutine you spawn" rule

```go
// BAD: an unrecovered panic in a goroutine crashes the entire process
go func() {
    process(item) // if this panics, BOOM
}()

// GOOD: wrap goroutines with a recovery
func safeGo(fn func()) {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                slog.Error("goroutine panic",
                    "panic", r,
                    "stack", string(debug.Stack()),
                )
            }
        }()
        fn()
    }()
}

safeGo(func() { process(item) })
```

This is the **single most-skipped staff-level Go pattern**: every `go` statement should have a recovery on the topmost function call, or every panic kills the process.

### HTTP middleware pattern

```go
func recoverMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                slog.Error("handler panic",
                    "url", r.URL.String(),
                    "panic", rec,
                    "stack", string(debug.Stack()),
                )
                // Send a 500 only if headers haven't been written
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

### Panic with a value, not just a string

```go
// BAD: panic("user not authenticated") — caller can't distinguish from other panics
panic("user not authenticated")

// GOOD: panic with a typed value, recover can switch on it
type authPanic struct{ Reason string }

panic(authPanic{Reason: "session expired"})

// Recovery:
defer func() {
    if r := recover(); r != nil {
        if ap, ok := r.(authPanic); ok {
            // handle auth specially
            _ = ap
        } else {
            // generic
        }
    }
}()
```

### Panic-driven control flow is an anti-pattern

```go
// BAD: using panic to skip layers of error handling
func parse(s string) Tree {
    defer func() { /* recover and convert to error */ }()
    parseExpr() // panics on parse error
    parseStmt() // panics on parse error
    // ...
}

// Go's stdlib does this in exactly one place: encoding/json's older internals,
// and the maintainers have publicly regretted it.

// GOOD: explicit error propagation — verbose but predictable
func parse(s string) (Tree, error) {
    e, err := parseExpr()
    if err != nil { return Tree{}, err }
    s, err := parseStmt()
    if err != nil { return Tree{}, err }
    // ...
}
```

Staff insight: panic should be **rare**, **typed**, and **recovered at a known boundary**. Every panic that escapes a boundary kills the process. Every panic that is recovered must be logged with `debug.Stack()` — otherwise you've turned the panic into a silent bug.

---
7. Context Errors — Cancellation Is Not a Failure

Context propagates **cancellation and deadlines**, not domain errors. When a context is cancelled, `ctx.Err()` returns `context.Canceled` or `context.DeadlineExceeded`. These are signals, not bugs.

```go
// BAD: treating context cancellation as a 500-level error
func (s *Service) Lookup(ctx context.Context, id string) (*User, error) {
    u, err := s.repo.Get(ctx, id)
    if err != nil {
        slog.Error("lookup failed", "id", id, "err", err) // logs noisy cancellations
        return nil, err
    }
    return u, nil
}

// GOOD: distinguish cancellation from real failures
func (s *Service) Lookup(ctx context.Context, id string) (*User, error) {
    u, err := s.repo.Get(ctx, id)
    if err != nil {
        if errors.Is(err, context.Canceled) || errors.Is(err, context.DeadlineExceeded) {
            // Client gave up; this isn't an error in our service
            slog.Debug("lookup cancelled", "id", id, "err", err)
            return nil, err
        }
        slog.Error("lookup failed", "id", id, "err", err)
        return nil, err
    }
    return u, nil
}
```

### Always check ctx.Err() after long operations, not just db calls

```go
// BAD: tight loop ignores cancellation
func process(ctx context.Context, items []Item) error {
    for _, item := range items {
        if err := handle(item); err != nil { // handle doesn't take ctx
            return err
        }
    }
    return nil
}

// GOOD: cooperative cancellation check
func process(ctx context.Context, items []Item) error {
    for _, item := range items {
        if err := ctx.Err(); err != nil {
            return err // returns context.Canceled or DeadlineExceeded
        }
        if err := handle(item); err != nil {
            return fmt.Errorf("handle %s: %w", item.ID, err)
        }
    }
    return nil
}
```

### Don't bury ctx.Err under a generic error

```go
// BAD: wrap mask makes errors.Is(err, context.Canceled) brittle if you
// re-create a new error from the message
if ctx.Err() != nil {
    return errors.New("operation timeout") // breaks the chain
}

// GOOD: preserve the context error in the chain
if err := ctx.Err(); err != nil {
    return fmt.Errorf("process: %w", err)
}
```

Staff insight: **context.Canceled is not an error in your service** — it's the client saying "I don't care anymore." Log it at Debug, not Error. Alarms triggered on `Canceled` rates dominate noise; alarms triggered on `DeadlineExceeded` for *internal* deadlines do indicate problems (your code couldn't meet its SLO). Distinguish.

---
8. errgroup — Concurrent Operations with First-Error Semantics

`golang.org/x/sync/errgroup` is the idiomatic concurrent-error pattern. It cancels its context when any goroutine returns an error and aggregates the *first* error to bubble up.

```go
// GOOD: errgroup for concurrent fan-out with shared cancellation
func (s *Service) AggregateOrder(ctx context.Context, orderID string) (*Order, error) {
    g, ctx := errgroup.WithContext(ctx)

    var order *Order
    var items []Item
    var payment *Payment

    g.Go(func() error {
        var err error
        order, err = s.orderRepo.Get(ctx, orderID)
        return err
    })
    g.Go(func() error {
        var err error
        items, err = s.itemsRepo.List(ctx, orderID)
        return err
    })
    g.Go(func() error {
        var err error
        payment, err = s.paymentRepo.Get(ctx, orderID)
        return err
    })

    if err := g.Wait(); err != nil {
        // The other goroutines were cancelled — ctx is done
        return nil, fmt.Errorf("aggregate order %s: %w", orderID, err)
    }
    order.Items = items
    order.Payment = payment
    return order, nil
}
```

### errgroup.SetLimit — bounded concurrency

```go
// 1.20+: limit concurrency without a manual semaphore
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(10) // max 10 in flight

for _, id := range ids {
    id := id // capture loop var — pre-1.22 idiom
    g.Go(func() error {
        return s.process(ctx, id)
    })
}
if err := g.Wait(); err != nil {
    return err
}
```

### Capturing ALL errors, not just the first

errgroup gives you the first error and cancels the rest. If you need every error, collect manually:

```go
type result struct {
    id  string
    err error
}
results := make(chan result, len(ids))
var wg sync.WaitGroup
for _, id := range ids {
    id := id
    wg.Add(1)
    go func() {
        defer wg.Done()
        results <- result{id: id, err: s.process(ctx, id)}
    }()
}
wg.Wait()
close(results)

var errs []error
for r := range results {
    if r.err != nil {
        errs = append(errs, fmt.Errorf("id %s: %w", r.id, r.err))
    }
}
return errors.Join(errs...)
```

### Gotcha: closure captures and goroutines

```go
// BAD pre-Go 1.22: loop variable captured by reference
for _, id := range ids {
    g.Go(func() error {
        return s.process(ctx, id) // all goroutines see the LAST value of id
    })
}

// GOOD pre-1.22: shadow the loop variable
for _, id := range ids {
    id := id
    g.Go(func() error { return s.process(ctx, id) })
}

// 1.22+: loop variables are per-iteration; original pattern is correct
for _, id := range ids {
    g.Go(func() error { return s.process(ctx, id) })
}
```

Staff insight: errgroup is one of the highest-leverage patterns in Go. It encapsulates *fan-out + first-error + shared cancellation* — three concerns that hand-rolled implementations get wrong constantly. The day you find yourself writing `sync.WaitGroup + chan error + sync.Once`, you should be reaching for errgroup instead.

---
9. Don't Log AND Return — The Double-Logging Anti-Pattern

This is the single most common mistake at every level in Go codebases.

```go
// BAD: every layer logs the error, then returns it
// Result: one error produces 5 log lines, none of which are useful

func (r *Repo) Get(id string) (*User, error) {
    u, err := r.db.Query(...)
    if err != nil {
        slog.Error("repo query failed", "err", err) // log 1
        return nil, err
    }
    return u, nil
}

func (s *Service) Get(id string) (*User, error) {
    u, err := s.repo.Get(id)
    if err != nil {
        slog.Error("service get failed", "err", err) // log 2
        return nil, err
    }
    return u, nil
}

func (h *Handler) Get(w http.ResponseWriter, r *http.Request) {
    u, err := h.service.Get(r.PathValue("id"))
    if err != nil {
        slog.Error("handler get failed", "err", err) // log 3
        http.Error(w, "error", 500)
        return
    }
    // ...
}
```

```go
// GOOD: wrap with context as the error climbs; LOG ONCE at the top
//       OR LOG ONCE at a known boundary, then return without re-logging

func (r *Repo) Get(id string) (*User, error) {
    u, err := r.db.Query(...)
    if err != nil {
        return nil, fmt.Errorf("repo.Get %s: %w", id, err) // wrap, don't log
    }
    return u, nil
}

func (s *Service) Get(id string) (*User, error) {
    u, err := s.repo.Get(id)
    if err != nil {
        return nil, fmt.Errorf("service.Get %s: %w", id, err)
    }
    return u, nil
}

func (h *Handler) Get(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    u, err := h.service.Get(id)
    if err != nil {
        // ONE log at the boundary — contains the entire wrap chain
        slog.Error("get user", "id", id, "err", err)
        http.Error(w, "error", 500)
        return
    }
    // ...
}
```

The wrap chain provides the trace: `handler.Get u-123: service.Get u-123: repo.Get u-123: sql: no rows`.

### The rule of thumb

```
Log when:
├── You are at a boundary (handler, RPC, top of goroutine)
├── You decide NOT to propagate (you swallow the error)
└── You're adding diagnostic info that can't be wrapped (request body dump, slow-query plan)

Wrap when:
└── You are NOT at a boundary — return the error with context, don't log it
```

Staff insight: if your codebase has more log lines than handled errors, somebody is logging at every layer. Audit by counting `slog.Error` calls vs `return.*Errorf`. The latter should outnumber the former by 5–10×.

---
10. HTTP / gRPC / RPC Error Translation

Internal errors are not your network protocol. At the boundary, **map** internal errors to the protocol's error model.

```go
// GOOD: explicit mapping layer
func toHTTPStatus(err error) int {
    switch {
    case errors.Is(err, userrepo.ErrNotFound):
        return http.StatusNotFound
    case errors.Is(err, userrepo.ErrAlreadyExists):
        return http.StatusConflict
    case errors.Is(err, context.DeadlineExceeded):
        return http.StatusGatewayTimeout
    case errors.Is(err, context.Canceled):
        return 499 // client closed request (nginx convention)
    }

    var verr *ValidationError
    if errors.As(err, &verr) {
        return http.StatusBadRequest
    }

    var rle *RateLimitError
    if errors.As(err, &rle) {
        return http.StatusTooManyRequests
    }

    // Unknown error — internal server error
    return http.StatusInternalServerError
}

// In your handler:
func (h *Handler) Get(w http.ResponseWriter, r *http.Request) {
    u, err := h.service.Get(r.PathValue("id"))
    if err != nil {
        status := toHTTPStatus(err)
        if status >= 500 {
            slog.Error("get user", "err", err) // log only internal errors
        }
        w.WriteHeader(status)
        json.NewEncoder(w).Encode(toErrorBody(err)) // safe public message
        return
    }
    json.NewEncoder(w).Encode(u)
}
```

### Never leak internal error strings to clients

```go
// BAD: lets a SQL constraint name leak to a public API
http.Error(w, err.Error(), http.StatusInternalServerError)

// GOOD: public message is curated; internal details only in logs
func toErrorBody(err error) ErrorResponse {
    if errors.Is(err, userrepo.ErrNotFound) {
        return ErrorResponse{Code: "user_not_found", Message: "user not found"}
    }
    var verr *ValidationError
    if errors.As(err, &verr) {
        return ErrorResponse{
            Code:    "validation_failed",
            Message: fmt.Sprintf("field %s: %s", verr.Field, verr.Reason),
        }
    }
    // Generic — never include err.Error() in the response
    return ErrorResponse{Code: "internal_error", Message: "something went wrong"}
}
```

### gRPC — status.Status

```go
// GOOD: map to canonical gRPC codes
func toGRPCStatus(err error) error {
    if err == nil {
        return nil
    }
    switch {
    case errors.Is(err, userrepo.ErrNotFound):
        return status.Error(codes.NotFound, "user not found")
    case errors.Is(err, userrepo.ErrAlreadyExists):
        return status.Error(codes.AlreadyExists, "user already exists")
    case errors.Is(err, context.DeadlineExceeded):
        return status.Error(codes.DeadlineExceeded, "deadline exceeded")
    case errors.Is(err, context.Canceled):
        return status.Error(codes.Canceled, "canceled")
    }
    return status.Error(codes.Internal, "internal error")
}

// Or use status.WithDetails to attach structured info that survives transport
func toGRPCError(verr *ValidationError) error {
    st := status.New(codes.InvalidArgument, "validation failed")
    st, _ = st.WithDetails(&errdetails.BadRequest{
        FieldViolations: []*errdetails.BadRequest_FieldViolation{
            {Field: verr.Field, Description: verr.Reason},
        },
    })
    return st.Err()
}
```

Staff insight: a service has two error vocabularies — the **internal**, which is rich and engineering-facing, and the **external**, which is curated and stable as part of the API. The mapping function between them is a critical piece of code; treat it like a serializer.

---
11. Retryable vs Permanent — The Categorization Pattern

Higher layers often want to ask "should I retry this?" rather than "what specifically went wrong?" Encode the category once and let callers ask the category.

```go
// GOOD: category interface, implemented by error types that opt in
type Temporary interface {
    Temporary() bool
}

type ConnectError struct {
    Cause error
    Addr  string
}

func (e *ConnectError) Error() string { return fmt.Sprintf("connect %s: %v", e.Addr, e.Cause) }
func (e *ConnectError) Unwrap() error { return e.Cause }
func (e *ConnectError) Temporary() bool { return true } // network errors are retryable

// Caller asks the category question
func isRetryable(err error) bool {
    var t Temporary
    if errors.As(err, &t) {
        return t.Temporary()
    }
    return false
}

// Retry loop
for attempt := 0; attempt < maxAttempts; attempt++ {
    err := doWork(ctx)
    if err == nil {
        return nil
    }
    if !isRetryable(err) {
        return err
    }
    backoff(attempt)
}
```

### Common categories worth modeling

```
Temporary()     — Retry might succeed (network blip, lock contention)
Timeout()       — Operation exceeded deadline (was historically net.Error)
Retryable()     — Caller-defined retry semantics
Fatal()         — Must NOT be retried (auth failure, malformed input)
Loggable()      — Should this be logged at Error vs Warn vs Debug?
PublicMessage() — Safe to expose to end-user
HTTPStatus()    — Maps cleanly to an HTTP code
StatusCode()    — Maps cleanly to a gRPC code
```

```go
// Example: a UserFacing error carries a safe public message
type UserFacing interface {
    error
    PublicMessage() string
}

type publicErr struct {
    cause error
    msg   string
}

func (e *publicErr) Error() string         { return e.cause.Error() }
func (e *publicErr) Unwrap() error         { return e.cause }
func (e *publicErr) PublicMessage() string { return e.msg }

func PublicWrap(err error, msg string) error { return &publicErr{cause: err, msg: msg} }

// At the HTTP boundary
func publicMessage(err error) string {
    var pf UserFacing
    if errors.As(err, &pf) {
        return pf.PublicMessage()
    }
    return "internal server error"
}
```

Staff insight: **categories outlast specific error values.** The set of sentinels grows; the set of categories (retryable, fatal, user-facing) is small and stable. Coding around categories scales; coding around `errors.Is(err, ErrFoo) || errors.Is(err, ErrBar) || ...` does not.

---
12. Ignoring Errors — When and How

Sometimes you must ignore an error. There are exactly three legitimate reasons:

```go
// 1. The error is structurally impossible
// strconv.Itoa never returns an error; bytes.Buffer.Write never returns an error
var buf bytes.Buffer
buf.WriteString("hello") // error return is always nil

// 2. The error is informational and you're already in a teardown path
defer func() {
    _ = file.Close() // best-effort cleanup; nothing useful to do with the error
}()

// 3. The next operation will surface the same problem more cleanly
_ = json.Unmarshal(badJSON, &v) // followed by a validation step that will fail anyway
```

### Always use _ explicitly

```go
// BAD: silent ignore looks like an oversight
file.Close()

// GOOD: explicit _ communicates "I considered this and chose to ignore"
_ = file.Close()
```

`errcheck` (golangci-lint) will flag the first; that's the point. The `_ =` ack tells the linter and the next reader.

### When defer Close can lose data

```go
// BAD: write file, defer close — but Close() may flush; flush may fail
func write(path string) error {
    f, err := os.Create(path)
    if err != nil {
        return err
    }
    defer f.Close()                  // close error silently dropped
    if _, err := f.Write(data); err != nil {
        return err
    }
    return nil
}

// GOOD: explicit close with error capture
func write(path string) (err error) {
    f, err := os.Create(path)
    if err != nil {
        return err
    }
    defer func() {
        if cerr := f.Close(); err == nil && cerr != nil {
            err = fmt.Errorf("close %s: %w", path, cerr)
        }
    }()
    if _, werr := f.Write(data); werr != nil {
        return fmt.Errorf("write %s: %w", path, werr)
    }
    return nil
}
```

This is **especially** important for `os.File` on networked / sync'd filesystems where the write may succeed but flush may fail on close.

Staff insight: ignoring an error is a fine choice in 5% of cases. The discipline is to mark it explicitly so future readers (and lints) can see it was deliberate, not accidental. Every `_ =` should be paired with a comment if it's not obviously fall-into category 1, 2, or 3 above.

---
13. Error Stack Traces — pkg/errors Is Dead; What Replaces It

`pkg/errors` was the de facto stack-trace library before Go 1.13's wrap support. It is now archived. New code should not use it.

### Stack traces via wrapping (the path Go chose)

```
"handler.Get u-123: service.Get u-123: repo.Get u-123: sql: no rows"
```

This is *function-level* context, not line-level. For most production debugging it is sufficient. For finer detail, attach a frame via `runtime/debug` only at unusual catch points:

```go
type tracedError struct {
    err   error
    stack []byte
}

func (e *tracedError) Error() string { return e.err.Error() }
func (e *tracedError) Unwrap() error { return e.err }
func (e *tracedError) Stack() []byte { return e.stack }

func WithStack(err error) error {
    if err == nil { return nil }
    return &tracedError{err: err, stack: debug.Stack()}
}

// Use sparingly — only where the call chain isn't otherwise traceable
err := doSomething()
if err != nil {
    return WithStack(err) // attaches stack at this point
}
```

### Use distributed tracing for cross-service stacks

For multi-service systems, the right primitive is **OpenTelemetry trace spans**. A traced operation that fails records the error on the span; the trace UI shows the full propagation. Spans give you what stack traces don't — cross-process visibility.

```go
import "go.opentelemetry.io/otel/codes"

func (s *Service) Get(ctx context.Context, id string) (*User, error) {
    ctx, span := tracer.Start(ctx, "Service.Get")
    defer span.End()

    u, err := s.repo.Get(ctx, id)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "repo.Get failed")
        return nil, fmt.Errorf("Service.Get %s: %w", id, err)
    }
    return u, nil
}
```

Staff insight: at process boundaries, the trace span is your stack frame. Inside the process, the wrap chain is your stack frame. Together they cover the end-to-end story without library help.

---
14. Defining Errors — Style, Naming, Placement

### Naming

```
Sentinel errors: ErrX (capital E)
   - exported if part of public API
   - unexported (errX) if internal

Typed errors: XError or XErr suffix
   - DNSError, ValidationError, RateLimitError
   - exported by default (callers usually want to inspect)

Error constructors: NewXError, or just package-level constructors
   - net.ErrClosed (sentinel)
   - &net.OpError{...} (typed, often constructed in-place)
```

### Placement

```go
// GOOD: at top of the package's file (or in a dedicated errors.go)
package userrepo

import "errors"

var (
    ErrNotFound      = errors.New("user not found")
    ErrAlreadyExists = errors.New("user already exists")
    ErrInvalidEmail  = errors.New("invalid email")
)

// Typed errors in same file or errors.go
type ValidationError struct { /* ... */ }

// Error CONSTRUCTORS (when there's logic involved) in same file
func newRetryableError(cause error, retryAfter time.Duration) error { /* ... */ }
```

### Don't define errors only at the point of use

```go
// BAD: error defined inline; cannot be matched by callers
func (r *Repo) Save(u User) error {
    if u.ID == "" {
        return errors.New("user has no id") // caller cannot errors.Is this
    }
}

// GOOD: error defined as a package var; matchable
var ErrMissingID = errors.New("missing id")

func (r *Repo) Save(u User) error {
    if u.ID == "" {
        return fmt.Errorf("Save: %w", ErrMissingID)
    }
}
```

### Wrap at the package boundary

```go
// GOOD: every method in the package returns errors wrapped with a package prefix
// Callers reading "userrepo:" know which layer surfaced the error
return fmt.Errorf("userrepo.Get %s: %w", id, err)
```

Some shops use a stronger convention — every error returned by a package is wrapped by a `package.Prefix()` helper. That works; pick a style and apply it everywhere.

Staff insight: error names, placement, and wrap conventions are *cosmetic until they aren't*. When you're grep-ing across 50 services trying to find where `"missing id"` originated, naming and placement become survival skills. Decide the conventions early; enforce them with lint rules.

---
15. Sentinel Pitfalls — Mutability and Equality

```go
// errors.New returns a *errorString — a unique pointer per call.
// Two sentinels with the same message are NOT equal.
var ErrA = errors.New("not found")
var ErrB = errors.New("not found")

fmt.Println(ErrA == ErrB) // false — distinct pointers
errors.Is(ErrA, ErrB)     // false

// This is by design: identity, not text, defines a sentinel.
```

### Don't redefine sentinels per call

```go
// BAD: each call creates a new value — errors.Is fails for callers
func (r *Repo) Get(id string) error {
    return errors.New("not found")
}

// Caller:
err := repo.Get(id)
err2 := repo.Get(id)
errors.Is(err, err2) // false! different values
```

The error must be a stable, exported (or unexported but module-internal) singleton.

### Don't mutate sentinels

```go
// errors.New returns a value with a string field — but the value itself
// has no mutation surface. If you make a custom typed sentinel, do not
// expose mutation:

type DBError struct {
    Code int
    Msg  string
}

func (e *DBError) Error() string { return e.Msg }

// BAD: a "sentinel" that's a mutable pointer is a footgun
var ErrConnLost = &DBError{Code: 1, Msg: "connection lost"}

// Some other code does: ErrConnLost.Code = 42
// Now every place that does errors.Is(err, ErrConnLost) is silently wrong.

// If you need a typed sentinel, freeze it:
var ErrConnLost error = &DBError{Code: 1, Msg: "connection lost"} // declared as interface, not *DBError
```

Staff insight: a sentinel is identity, not content. Treat it as immutable; expose it through the smallest interface that callers need. Generally that interface is `error`, full stop.

---
16. The %w Chain Depth Gotcha

```go
// errors.Is walks the chain — wrap depth is not free, but it's cheap.
// HOWEVER, walking is O(depth); if you wrap N times per layer, this matters.

// BAD: redundantly wrapping with the same prefix
func (a *A) Do() error {
    err := a.next.Do()
    if err != nil {
        return fmt.Errorf("a.Do: %w", err) // OK
    }
    return nil
}

func (b *B) Do() error {
    err := b.next.Do()
    if err != nil {
        return fmt.Errorf("b.Do: a.Do: %w", err) // BAD — re-states what's already in the chain
    }
    return nil
}
```

### The "skip a layer" anti-pattern

```go
// BAD: skipping a wrap because "it's already wrapped by the layer below"
// works until somebody refactors the layer below and the chain has a hole

// GOOD: every layer adds exactly one wrap with its own context
```

### Print the chain when debugging

```go
// %+v on a wrapped error doesn't show the chain — only %w + %v does
fmt.Printf("%v\n", err)  // top message only with traversal
fmt.Printf("%+v\n", err) // depends on errors that implement Format

// Manual chain walk
for e := err; e != nil; e = errors.Unwrap(e) {
    fmt.Printf("  %T: %v\n", e, e)
}
```

Staff insight: each `%w` is one frame of context. Don't pack two frames into one (`"a.Do: b.Do: %w"`); don't skip frames; don't crash through 5 layers without wrapping. The chain should mirror the call stack 1:1.

---
17. Error Equality in Tests

```go
// BAD: comparing on error message — brittle and noisy
if err.Error() != "user not found" {
    t.Errorf("got %v want %q", err, "user not found")
}

// GOOD: errors.Is / errors.As
if !errors.Is(err, userrepo.ErrNotFound) {
    t.Errorf("got %v want %v", err, userrepo.ErrNotFound)
}

// GOOD: extract typed error
var verr *ValidationError
if !errors.As(err, &verr) {
    t.Fatalf("expected ValidationError, got %T", err)
}
if verr.Field != "email" {
    t.Errorf("got field %q, want email", verr.Field)
}
```

### testify ErrorIs / ErrorAs

```go
import "github.com/stretchr/testify/require"

require.ErrorIs(t, err, userrepo.ErrNotFound)
require.ErrorContains(t, err, "user not found") // last resort — message match

var verr *ValidationError
require.ErrorAs(t, err, &verr)
require.Equal(t, "email", verr.Field)
```

### Table tests for errors

```go
tests := []struct {
    name    string
    input   string
    wantErr error
}{
    {"empty", "", ErrMissingID},
    {"invalid email", "@@@", ErrInvalidEmail},
    {"ok", "x@y.com", nil},
}
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        err := validate(tt.input)
        if tt.wantErr == nil {
            require.NoError(t, err)
        } else {
            require.ErrorIs(t, err, tt.wantErr)
        }
    })
}
```

Staff insight: test the *category* of error, not the message. Tests that depend on the exact message break whenever the wrap chain changes — which is rarely the thing you wanted to test in the first place.

---
18. Performance — Errors in the Hot Path

Errors are values; allocating them costs memory. In a hot loop, this matters.

```go
// BAD: builds a new error every call — heap allocates, calls fmt.Sprintf
for _, item := range items {
    if !validate(item) {
        return fmt.Errorf("invalid item %s", item.ID) // alloc per call
    }
}

// GOOD: package-level sentinel — no allocation on the failure path
var ErrInvalidItem = errors.New("invalid item")

for _, item := range items {
    if !validate(item) {
        return ErrInvalidItem // no alloc
    }
}

// If you need context, wrap ONLY at the boundary, not in the loop
for _, item := range items {
    if !validate(item) {
        return fmt.Errorf("validate items, item %s: %w", item.ID, ErrInvalidItem)
    }
}
// One allocation at the top of the error path, not N allocations.
```

### errors.New vs fmt.Errorf cost

```
errors.New("static string")           — single allocation at init; no per-call cost
fmt.Errorf("format %s", x)            — allocation + Sprintf cost per call
fmt.Errorf("static prefix: %w", err)  — allocation per call (Errorf returns a *wrapError)
```

If you're returning the same error message hundreds of times per second, **make it a sentinel**.

### Benchmark and look

Use `go test -bench -benchmem` to confirm. Error allocs show up in `allocs/op`. If you see a regression after adding an error path, this is usually why.

```go
// 4 allocs/op vs 0 — the difference between sentinel and dynamic wrap
func BenchmarkSentinel(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = doWithSentinel()
    }
}
func BenchmarkDynamic(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = doWithFmtErrorf()
    }
}
```

Staff insight: 99% of code shouldn't think about error allocation cost. The 1% that does is in tight loops, in protocol parsers, in DB drivers. In that 1%, prefer sentinels and avoid `fmt.Errorf` inside the hot path. Wrap at the function boundary, not at every internal step.

---
19. Closing Resources — The errgroup-of-Closers Pattern

Multiple resources, all must be closed, want a single error result:

```go
// GOOD: explicit close with error aggregation
type resources struct {
    closers []io.Closer
}

func (r *resources) Add(c io.Closer) { r.closers = append(r.closers, c) }

func (r *resources) Close() error {
    var errs []error
    // Reverse order — most-recently-opened closed first
    for i := len(r.closers) - 1; i >= 0; i-- {
        if err := r.closers[i].Close(); err != nil {
            errs = append(errs, err)
        }
    }
    return errors.Join(errs...)
}

// Usage
func process() (err error) {
    res := &resources{}
    defer func() { err = errors.Join(err, res.Close()) }()

    db, e := sql.Open(...)
    if e != nil { return e }
    res.Add(db)

    cache, e := openCache(...)
    if e != nil { return e }
    res.Add(cache)

    // ...
    return nil
}
```

### File close error: do not ignore on the WRITE path

```go
// On read: ignoring Close is fine (you've already read what you needed)
defer f.Close()

// On write: ignoring Close can lose data — flush may fail at close
defer func() {
    if cerr := f.Close(); err == nil {
        err = cerr // promote close error if no other error occurred
    }
}()
```

Staff insight: the cleanup path is the most error-prone path because it's the path everyone forgets. Every long-lived resource your function opens should have a known close-with-error pattern. errgroup-style aggregation is the clean version of "wait, did all 5 of my deferred closes succeed?"

---
20. The "Errors Should Be Handled Once" Principle

Each error in your codebase should be handled in **exactly one** of these ways:

```
1. Returned up (with wrap)              — the default
2. Logged and swallowed                  — only at known boundaries
3. Converted to a protocol response      — at the HTTP/RPC layer
4. Triggered a retry                     — by a retry wrapper
5. Triggered a panic                     — only for invariant violations
```

Any error that is handled TWICE (logged AND returned, retried AND returned, swallowed AND logged-twice) creates noise.

### Concrete checklist

```
For every "if err != nil" block in your codebase, exactly one of:
[ ] return fmt.Errorf("ctx: %w", err)        ← propagate with wrap
[ ] slog.Error(...); return nil               ← log-and-swallow at boundary
[ ] return convertToHTTP(err)                 ← protocol mapping at handler
[ ] continue                                   ← intentional skip (loop)
[ ] _ = something                              ← ignore (rare)
```

If you have two of these in the same block, you have a smell.

Staff insight: error handling is the place where junior code and staff code most visibly differ. Junior code: 5 layers each log + return. Staff code: 5 layers each wrap, one logs at the top, one maps at the boundary. The discipline is to **not duplicate handling**.

---
21. Anti-Patterns Reference

```go
// ────────────────────────────────────────────────────────────────────
// 1. String-comparing errors
if strings.Contains(err.Error(), "not found") { ... }
// → Use errors.Is(err, ErrNotFound)

// ────────────────────────────────────────────────────────────────────
// 2. Returning value AND error when err != nil
return user, err  // when user is partially populated
// → Return nil and err; or document the partial-success contract clearly

// ────────────────────────────────────────────────────────────────────
// 3. Naked `return err` with no context
if err != nil { return err }
// → return fmt.Errorf("doing X: %w", err)
// Exception: at the very top boundary where wrap adds no value

// ────────────────────────────────────────────────────────────────────
// 4. fmt.Errorf("%s", err) or fmt.Errorf("%v", err)
return fmt.Errorf("query: %v", err)
// → Use %w; %v silently breaks errors.Is/errors.As

// ────────────────────────────────────────────────────────────────────
// 5. panic in library code on bad input
func Parse(s string) Result { if s == "" { panic("empty") } ... }
// → Return an error; let the caller decide

// ────────────────────────────────────────────────────────────────────
// 6. Ignored errors without `_ =`
file.Close()
// → _ = file.Close() (explicit and lint-friendly)

// ────────────────────────────────────────────────────────────────────
// 7. Logging errors at every layer
slog.Error("...", err); return err
// → Log only at boundaries (handler, top of goroutine, cron job runner)

// ────────────────────────────────────────────────────────────────────
// 8. Wrapping with %w in a tight loop
for ... { return fmt.Errorf("item %s: %w", id, ErrX) }
// → Wrap at the function boundary, not inside the loop

// ────────────────────────────────────────────────────────────────────
// 9. Sentinels defined in a function body
func X() error { notFound := errors.New("not found"); return notFound }
// → var ErrNotFound = errors.New("not found") at package level

// ────────────────────────────────────────────────────────────────────
// 10. Treating context.Canceled as a 500-level error
slog.Error("call failed", err)  // when err is context.Canceled
// → Distinguish: log Canceled at Debug, real errors at Error

// ────────────────────────────────────────────────────────────────────
// 11. Goroutines without recover at the top
go work(item) // panic crashes process
// → Wrap with a recover; log debug.Stack() on recover

// ────────────────────────────────────────────────────────────────────
// 12. Bare error values leaking to public APIs
http.Error(w, err.Error(), 500)
// → Map internal errors to safe public messages

// ────────────────────────────────────────────────────────────────────
// 13. errors.As with value instead of pointer-to-pointer
var verr ValidationError; errors.As(err, &verr) // wrong if ValidationError methods are on *ValidationError
// → var verr *ValidationError; errors.As(err, &verr)

// ────────────────────────────────────────────────────────────────────
// 14. Custom error type without Unwrap()
type Err struct{ Cause error }
func (e *Err) Error() string { return e.Cause.Error() }
// → Implement Unwrap() error so errors.Is/As traverse

// ────────────────────────────────────────────────────────────────────
// 15. defer Close on write paths without capturing the error
defer f.Close()
// → defer with named return assignment to capture close error on writes

// ────────────────────────────────────────────────────────────────────
// 16. Using pkg/errors in new code
errors.Wrap(err, "msg") // pkg/errors API
// → fmt.Errorf("msg: %w", err) — stdlib equivalent since 1.13

// ────────────────────────────────────────────────────────────────────
// 17. Two %w in same fmt.Errorf pre-1.20
fmt.Errorf("a: %w b: %w", e1, e2) // pre-1.20 only first %w wraps
// → Pre-1.20: errors.Join or custom multi-error
// → 1.20+: multiple %w is supported

// ────────────────────────────────────────────────────────────────────
// 18. Comparing errors with ==
if err == sql.ErrNoRows { ... }
// → errors.Is(err, sql.ErrNoRows)
// Works in narrow cases; breaks the moment someone wraps the error.

// ────────────────────────────────────────────────────────────────────
// 19. Returning errors from deferred functions silently
defer func() { f.Close() }() // close error lost
// → defer func() { err = errors.Join(err, f.Close()) }() with named return

// ────────────────────────────────────────────────────────────────────
// 20. Panic-driven control flow
panic(parseError{...}) // and recover at top of parser
// → Explicit return error chain; verbose but predictable
```

---
22. Slog and Structured Error Logging

`log/slog` (stdlib since 1.21) is the modern logger. Errors play with it cleanly:

```go
// GOOD: structured error logging
slog.Error("process item",
    "item_id", item.ID,
    "user_id", user.ID,
    "err", err, // slog formats with err.Error() and attaches as a structured attr
)
```

### Attaching error attributes via slog.Attr

```go
// If you want fields from the error in the log
var verr *ValidationError
if errors.As(err, &verr) {
    slog.Error("validation failed",
        "field", verr.Field,
        "reason", verr.Reason,
        "err", err, // full chain still attached
    )
}
```

### The LogValuer interface

Your error types can implement `slog.LogValuer` to control how they're logged:

```go
type RateLimitError struct {
    RetryAfter time.Duration
    Limit      int
}

func (e *RateLimitError) Error() string { return "rate limited" }
func (e *RateLimitError) LogValue() slog.Value {
    return slog.GroupValue(
        slog.Duration("retry_after", e.RetryAfter),
        slog.Int("limit", e.Limit),
    )
}

// slog auto-uses LogValue when this error is logged
slog.Error("call failed", "err", rateLimitErr)
// Output: ... err.retry_after=2s err.limit=100
```

Staff insight: structured logging means your error types' fields can be queried as log attributes. Stop logging `err.Error()` as one big string; lift the structured data out so dashboards can group by it.

---
23. Don't Leak Internal Errors Across API Boundaries

```go
// BAD: internal errors flow out to client
func (h *Handler) Get(w http.ResponseWriter, r *http.Request) {
    u, err := h.repo.Get(r.PathValue("id"))
    if err != nil {
        http.Error(w, err.Error(), 500)
        // leaks: "pq: connection to db:5432 refused, certificate expired"
    }
}

// GOOD: domain error → public message at the boundary
func (h *Handler) Get(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    u, err := h.repo.Get(id)
    if err != nil {
        if errors.Is(err, repo.ErrNotFound) {
            writeJSON(w, http.StatusNotFound, errorBody{Code: "not_found"})
            return
        }
        // Log everything for engineers
        slog.Error("repo.Get", "id", id, "err", err)
        // Tell the client the minimum
        writeJSON(w, http.StatusInternalServerError, errorBody{Code: "internal_error"})
        return
    }
    writeJSON(w, http.StatusOK, u)
}
```

Why it matters:
- **Security**: error strings reveal infrastructure (db hostname, file paths).
- **API stability**: callers begin to depend on error messages; you can never change them.
- **Localization**: client may need translated messages — `err.Error()` is English-only.

Staff insight: every API has two error vocabularies: the **engineer-facing wrap chain** (rich, English, logged) and the **client-facing error code** (stable, opaque, mappable). The boundary handler is the translator; don't skip it.

---
24. Errors and Observability — Putting It Together

A staff-grade error flow:

```
1. Error is produced at a leaf (DB driver, HTTP call, file system).
2. Each layer up wraps with one phrase of context (no logging).
3. At the boundary:
   a. Log ONCE with structured attributes (slog).
   b. Record the error on the OpenTelemetry span.
   c. Increment a metric (counter labeled by error category).
   d. Convert to public protocol error.
4. Higher boundary (frontend, monitoring) aggregates by category.
```

```go
// The composed handler middleware:
func errorMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := r.Context()
        span := trace.SpanFromContext(ctx)

        rw := &recordingWriter{ResponseWriter: w}
        defer func() {
            if rec := recover(); rec != nil {
                err := fmt.Errorf("panic: %v", rec)
                span.RecordError(err)
                span.SetStatus(codes.Error, "handler panic")
                slog.ErrorContext(ctx, "handler panic",
                    "url", r.URL.String(),
                    "panic", rec,
                    "stack", string(debug.Stack()),
                )
                metrics.HandlerPanics.Inc()
                http.Error(w, "Internal Server Error", 500)
                return
            }
            if rw.status >= 500 {
                metrics.HandlerErrors.WithLabelValues(strconv.Itoa(rw.status)).Inc()
            }
        }()

        next.ServeHTTP(rw, r)
    })
}
```

Each error is now:
- in **logs** (slog, structured)
- in **traces** (RecordError on span)
- in **metrics** (counter per category)
- in **client response** (mapped to safe public code)

That is the staff-level shape. Hand-wave any of these and you'll be debugging blind.

---
25. Summary — Decision Reference

```
What to use                       When
─────────────────────────────────────────────────────────────────────────
errors.New("...") as package var  Sentinel for a stable, payload-free condition
&MyError{...} typed               Caller needs structured fields off the error
fmt.Errorf("ctx: %w", err)        Adding context as the error climbs the stack
errors.Join(errs...)              Multiple errors from one logical operation
errors.Is(err, ErrX)              "Is this a kind of X?" (identity through chain)
errors.As(err, &target)           "Can I extract an X from this?" (type through chain)
err == ErrX                       Almost never — use errors.Is
%w                                Default wrap verb — preserves chain
%v or %s with error               NEVER — breaks errors.Is/As
panic                             Invariant violation; programmer error; init-time
recover                           Top of every goroutine; HTTP middleware; RPC interceptor
context.Canceled                  Client gave up — log Debug, not Error
context.DeadlineExceeded          Internal SLO miss — log Warn or Error depending on layer
errgroup                          Concurrent fan-out with first-error + cancellation
Custom Is(error) bool             Category match (e.g., is-retryable, is-temporary)
Custom Unwrap() error             Mandatory on typed errors that wrap a cause
Custom LogValue() slog.Value      When error has structured fields you want in logs
_ = file.Close()                  Best-effort cleanup on read paths
defer with named return           Capturing close errors on write paths
pkg/errors                        Don't — use stdlib since 1.13
```

```
Anti-patterns to grep for in code review:
─────────────────────────────────────────────────────────────────────────
strings.Contains(err.Error(), ...)      → use errors.Is
fmt.Errorf("...: %v", err)              → use %w
slog.Error(...); return err             → wrap and return; log only at boundary
panic in library code                   → return error
http.Error(w, err.Error(), 500)         → map to safe public message
defer f.Close() on write path           → capture close error
errors.New inside a function body       → make it a package-level var
go work() without recover               → wrap goroutines
"failed to do X: %w"                    → "do X: %w" (drop "failed to")
err == sentinel                         → errors.Is(err, sentinel)
errors.As(err, &someValue)              → confirm value vs pointer
```

```
The five questions to ask of every error in a code review:
─────────────────────────────────────────────────────────────────────────
1. Does it preserve the wrap chain? (uses %w, not %v)
2. Is it matched by category, not by string?
3. Is it handled exactly once (returned OR logged OR mapped — pick one)?
4. Does it cross a boundary without leaking internals?
5. Does it carry the structured data the caller needs (typed if so)?
```

The overarching staff-level principle: **error handling is API design**. Every error your code returns is part of your contract with callers. The wrap chain is your stack frame. The sentinels are your enum. The typed errors are your structured data. The categories (`Temporary`, `UserFacing`) are your taxonomy. Get them right and downstream code is clean and self-documenting; get them wrong and every layer accumulates `if err != nil` warts trying to pry meaning out of a string.

The Go language gave you the cheapest possible primitive — an interface with one method. The whole craft is in everything you build on top.