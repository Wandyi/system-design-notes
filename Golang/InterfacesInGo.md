⏺ Staff-Level Design Paradigms for Managing Interfaces in Go

---
Foundational Principle: Interfaces Are Structural, Discovered, and Owned by the Consumer

Go interfaces are not Java/C# interfaces. There are three properties that change every interface decision:

1. **Structural / implicit satisfaction.** A type satisfies an interface by having the right methods. There is no `implements` keyword. The compiler verifies the satisfaction *at the use site*, not at the type definition.

2. **Discovered, not declared.** The right interface emerges from how a function *uses* a value, not from a designer's up-front taxonomy. You write the function, look at what it actually does, then define the interface from that.

3. **Owned by the consumer.** The package that *uses* a value should define the interface; the package that *provides* the value returns a concrete type. This is the inversion of Java's "interface in the provider package" convention.

```go
// BAD: interface defined where the concrete type lives — caller is forced to
// import the provider package just to depend on the interface
package storage

type Storage interface {
    Get(key string) ([]byte, error)
    Put(key string, val []byte) error
}

type S3Storage struct { /* ... */ }

func (s *S3Storage) Get(key string) ([]byte, error) { /* ... */ }
func (s *S3Storage) Put(key string, val []byte) error { /* ... */ }

// In consumer:
import "myorg/storage"
type Service struct { st storage.Storage }
// Service is forced to know about the storage package; any change to Storage
// is a breaking change in storage's public API.

// GOOD: interface defined in the consumer; provider returns concrete type
package storage

type S3Storage struct { /* ... */ }

func (s *S3Storage) Get(key string) ([]byte, error) { /* ... */ }
func (s *S3Storage) Put(key string, val []byte) error { /* ... */ }

// In consumer:
package service

// We use only Get and Put; that's our local contract
type blobStore interface {
    Get(key string) ([]byte, error)
    Put(key string, val []byte) error
}

type Service struct { st blobStore }
// service does not depend on storage at compile time;
// can substitute a memory implementation in tests without storage's permission.
```

Staff insight: this single inversion — **define interfaces where they're used, not where they're implemented** — eliminates ~80% of "I need to update three packages to add a method" pain. 
It also makes mocking trivial without exposing test-only types in production packages.

---
1. Accept Interfaces, Return Concrete Types

The Go community's terse rule. The rationale is purely about *who has the flexibility*.

```go
// BAD: returning interface forces caller into a specific contract
// Adding a method later is a breaking change for everyone who implements it
package cache

type Cache interface {
    Get(string) ([]byte, bool)
    Set(string, []byte)
}

func NewRedisCache(addr string) Cache { /* returns *redisCache wrapped */ }

// Caller can't see Redis-specific methods like Pipeline(), Subscribe()
c := cache.NewRedisCache("localhost:6379")
c.Pipeline() // doesn't compile — Cache doesn't have it

// GOOD: return concrete type; caller assigns to interface if they want
func NewRedisCache(addr string) *RedisCache { /* ... */ }

// Caller chooses
var c CacheInterface = cache.NewRedisCache(addr)  // when they want abstraction
rc := cache.NewRedisCache(addr)                    // when they want specifics
rc.Pipeline()                                      // works
```

```go
// BAD: accepting concrete types over-constrains the caller
func Process(items []Item, w *bytes.Buffer) error {
    // Why bytes.Buffer? What if I want to write to a file? An HTTP response?
}

// GOOD: accept the smallest interface that does the job
func Process(items []Item, w io.Writer) error {
    // Any io.Writer works: file, buffer, http.ResponseWriter, gzip writer
}
```

### The exceptions to "return concrete"

1. **The concrete type is unexported.** The interface IS the public surface.
   ```go
   func NewWriter(w io.Writer, fmt Format) io.Writer {
       switch fmt {
       case JSON: return &jsonWriter{w: w}  // jsonWriter unexported
       case CSV:  return &csvWriter{w: w}   // csvWriter unexported
       }
       return w
   }
   ```

2. **The function returns one of multiple types.** Returning an interface is the only way.
   ```go
   func NewCache(distributed bool) Cache {
       if distributed { return newRedisCache() }
       return newLocalCache()
   }
   ```

3. **The interface itself is the contract** — `error`, `io.Reader`, `io.Writer`. Returning these is fine because they ARE the API.

Staff insight: the maxim is heuristic, not law. The deep version is: **return the smallest set of operations the caller will ever need to do with this value.** For 90% of constructors, that's the concrete type. For specifically-polymorphic factories, that's the interface.

---
2. Interface Size — Smaller Is Better

The standard library is the canonical example. The most-used interfaces have **one method**:

```go
type error interface { Error() string }
type io.Reader interface { Read(p []byte) (int, error) }
type io.Writer interface { Write(p []byte) (int, error) }
type io.Closer interface { Close() error }
type fmt.Stringer interface { String() string }
type sort.Interface interface {  // three, but still tight
    Len() int
    Less(i, j int) bool
    Swap(i, j int)
}
```

These compose into bigger interfaces by embedding, **not** by listing:

```go
type ReadWriter interface {
    Reader
    Writer
}

type ReadCloser interface {
    Reader
    Closer
}

type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}
```

### Why smaller wins

```go
// BAD: one fat interface — every implementor must implement everything,
//      every test mock must stub everything, methods that aren't used are noise
type Storage interface {
    Get(string) ([]byte, error)
    Set(string, []byte) error
    Delete(string) error
    List(prefix string) ([]string, error)
    Snapshot() ([]byte, error)
    Restore([]byte) error
    Close() error
    Stats() Stats
    Pipeline() Pipeline
}

// Function that only reads:
func render(s Storage, key string) ([]byte, error) {
    return s.Get(key) // uses ONE method; depends on NINE
}

// GOOD: small interfaces, composed only where needed
type blobGetter interface {
    Get(string) ([]byte, error)
}

func render(g blobGetter, key string) ([]byte, error) {
    return g.Get(key) // depends on what it uses
}

// In tests:
type fakeGetter map[string][]byte
func (f fakeGetter) Get(k string) ([]byte, error) { return f[k], nil }
// 3 lines. Done.
```

### "Many small" vs "one big" — the import-graph effect

When you split a fat interface into single-method interfaces, each consumer depends on only the methods it uses. This means:

- **Adding a method** to one small interface only affects implementors of that interface — not everyone touching the bigger one.
- **Tests** only need stubs for the methods their target actually calls.
- **Refactoring** (moving a method to a different concrete type) doesn't cascade.

Staff insight: the Go proverb is "the bigger the interface, the weaker the abstraction." Practically: count methods in the interfaces in your codebase. If any consumer-side interface has more than 3 methods, you almost certainly want to split it into two or three.

---
3. Define Interfaces at the Consumer

The "where to put the interface declaration" decision is the highest-leverage architectural choice in Go.

```go
// BAD: provider-side interface — consumer imports provider just for the type
package storage

type Storage interface { /* ... */ }
type S3Storage struct { /* implements Storage */ }
type GCSStorage struct { /* implements Storage */ }

package svc
import "myorg/storage"
type Service struct { s storage.Storage }  // forced import
```

```go
// GOOD: consumer-side interface — providers don't even know it exists
package storage // does NOT define Storage interface

type S3Storage struct { /* concrete */ }
type GCSStorage struct { /* concrete */ }

// each consumer defines the slice of operations it cares about
package svc

type blobStore interface {
    Get(key string) ([]byte, error)
    Put(key string, val []byte) error
}

type Service struct { s blobStore }

// In wiring/main:
func main() {
    s3 := storage.NewS3Storage(cfg)
    svc := svc.New(s3) // s3 implicitly satisfies blobStore
}
```

The S3Storage type **never declares** that it satisfies `blobStore`. Go's implicit satisfaction connects them at the use site. The package boundary becomes thin: `svc` knows nothing about `storage`.

### When to define interfaces in the provider

There's exactly one common case: **the provider's contract is the abstraction.**

```go
// io.Reader, io.Writer, io.Closer — these ARE the abstractions
// They live in the io package because the io package defines the contract

// sql.Driver — the contract for "anything that drives a SQL connection"
// lives in database/sql because the contract is the package's purpose

// http.Handler — the contract for "anything servable as HTTP"
// lives in net/http because that's the contract net/http defines
```

These are *protocol* packages — the interface is the product. For your business code, the provider rarely has a contract that's intrinsic to it; the interfaces belong with the consumers.

Staff insight: if you find yourself writing "package X defines the IFoo interface and the FooImpl struct," ask why. In Go, X usually defines `Foo` (concrete) and the *callers* define `Fooer` if they need it. Java patterns transferred to Go this way create unnecessary coupling.

---
4. Compile-Time Interface Assertions

When you intend a concrete type to satisfy an interface, **prove it at compile time**:

```go
// At the top of the file or near the type
var _ io.Reader = (*MyReader)(nil)
var _ io.Closer = (*MyReader)(nil)

type MyReader struct { /* ... */ }

func (r *MyReader) Read(p []byte) (int, error) { /* ... */ }
func (r *MyReader) Close() error               { /* ... */ }
```

The `_ = ...` line compiles iff `*MyReader` satisfies the named interface. If somebody changes `Read`'s signature or removes `Close`, the build breaks at this assertion — *not* at some distant caller.

### Why this matters

```go
// Without the assertion, the breakage shows up here:
func consumer(r io.Reader) { /* ... */ }

func main() {
    var mr *MyReader
    consumer(mr) // "cannot use mr (type *MyReader) as type io.Reader" — only here
}

// With the assertion at type definition, the breakage shows up there:
var _ io.Reader = (*MyReader)(nil)
// error points right at the file with the type definition
```

This is **the single highest-leverage idiom** for keeping interfaces and implementations in sync. Put it on every interface a type is meant to satisfy.

### Where to put the assertion

```go
// 1. Top of the file (most common)
var _ io.Reader = (*MyReader)(nil)
type MyReader struct { /* ... */ }

// 2. Right after the type, before methods (also fine)
type MyReader struct { /* ... */ }
var _ io.Reader = (*MyReader)(nil)
func (r *MyReader) Read(p []byte) (int, error) { /* ... */ }

// 3. In a build-tagged file with all assertions (clean, but harder to discover)
// interfaces_assertion.go
package mypkg
var (
    _ io.Reader = (*MyReader)(nil)
    _ io.Closer = (*MyReader)(nil)
    _ http.Handler = (*Handler)(nil)
)
```

Staff insight: every place a concrete type is intended to satisfy an interface, add a compile-time assertion. It's free, it documents intent, it catches refactors at the source, and the diff in code review shows immediately when an interface relationship changes.

---
5. Nil Interface vs Nil Pointer Wrapped in Interface — The Classic Trap

The single most-asked Go interview question. Worth internalizing precisely.

```go
type MyError struct{ msg string }
func (e *MyError) Error() string { return e.msg }

// Function that conditionally returns an error
func doSomething() error {
    var err *MyError       // err is a *MyError, value nil
    if shouldFail() {
        err = &MyError{"boom"}
    }
    return err             // err is *MyError(nil); WRAPPED in error interface, NOT nil
}

// Caller:
if err := doSomething(); err != nil {
    fmt.Println(err) // PRINTS even when err == nil!
}
```

### Why it happens

An interface value in Go is a **(type, value) pair**:
- `nil` interface = `(nil, nil)`
- A `*MyError` wrapped in `error` = `(*MyError, nil)` if the pointer is nil

The interface comparison `err != nil` compares the *pair*. `(*MyError, nil)` != `(nil, nil)`.

### The fix — return `nil` explicitly, not a typed nil

```go
func doSomething() error {
    var err *MyError
    if shouldFail() {
        err = &MyError{"boom"}
    }
    if err == nil {
        return nil          // explicit nil interface
    }
    return err              // explicit *MyError
}
```

### Better — don't keep a typed nil variable around

```go
func doSomething() error {
    if shouldFail() {
        return &MyError{"boom"}
    }
    return nil              // returns untyped nil; trivial to get right
}
```

### Where this bites in practice

```go
// BAD: function returns a custom error type — caller's nil check fails
func validate() *ValidationError { /* may return nil */ }

func process() error {
    return validate() // returns *ValidationError; nil pointer becomes non-nil interface
}

// process returns (*ValidationError, nil) which is NOT a nil interface
err := process()
if err != nil {  // ← always true
    log.Print(err)
}
```

The fix: have `process` check and convert.

```go
func process() error {
    if verr := validate(); verr != nil {
        return verr
    }
    return nil
}
```

Staff insight: **never return a concrete-pointer-type from a function declared to return an interface, unless you guarantee it's non-nil at every return path.** The combination of "function returns error" and "concrete type used as the return variable" is a footgun mine. Always convert at the boundary.

---
6. Method Sets — Value vs Pointer Receivers

The interface method set rule: which receiver style determines which type satisfies the interface.

```go
type Greeter interface { Greet() string }

type Hi struct{}
func (h Hi) Greet() string { return "hello" }      // value receiver
// Hi    satisfies Greeter ✓
// *Hi   satisfies Greeter ✓

type Yo struct{}
func (y *Yo) Greet() string { return "yo" }        // pointer receiver
// *Yo  satisfies Greeter ✓
// Yo   does NOT satisfy Greeter ✗
```

Why: a pointer-receiver method needs a pointer; you can't take the address of an interface's underlying value implicitly.

### The error this produces

```go
type Logger interface { Log(string) }

type FileLogger struct { /* ... */ }
func (f *FileLogger) Log(s string) { /* ... */ }   // pointer receiver

var l Logger = FileLogger{}        // ERROR: FileLogger does not implement Logger
                                   // (Log method has pointer receiver)
var l Logger = &FileLogger{}       // works
```

### The convention

If *any* of your methods are pointer-receiver, make *all* of them pointer-receiver. Mixing is technically allowed but confusing.

```go
// BAD: mixed receivers — readers must check each method's signature
type Cart struct { items []Item }
func (c Cart)  Items() []Item { return c.items }   // value
func (c *Cart) Add(i Item)    { c.items = append(c.items, i) } // pointer

// GOOD: all-pointer if any method mutates
type Cart struct { items []Item }
func (c *Cart) Items() []Item { return c.items }
func (c *Cart) Add(i Item)    { c.items = append(c.items, i) }

// GOOD: all-value if no method mutates and the type is small/immutable
type Point struct { X, Y float64 }
func (p Point) Add(q Point) Point { return Point{p.X + q.X, p.Y + q.Y} }
func (p Point) Len() float64     { return math.Sqrt(p.X*p.X + p.Y*p.Y) }
```

### The map-value gotcha

You cannot take the address of a map value. So pointer-receiver methods don't auto-promote on map values:

```go
m := map[string]Cart{"a": {}}
// m["a"].Add(item) → "cannot take the address of m[a]"
// You must assign and reassign, or use map[string]*Cart
```

Staff insight: pointer receivers ↔ "this value is identity-having and may mutate"; value receivers ↔ "this value is data, copying is fine, no identity to preserve." Make the call once per type and apply it uniformly.

---
7. The "any" Type and When to Avoid It

`any` is the 1.18 alias for `interface{}`. Same type, friendlier name.

```go
type Cache struct {
    data map[string]any  // can hold anything
}

func (c *Cache) Get(k string) (any, bool) { v, ok := c.data[k]; return v, ok }
func (c *Cache) Set(k string, v any)      { c.data[k] = v }
```

`any` is the **escape hatch** from Go's type system. It says "I cannot — or do not want to — know this value's type at compile time."

### When `any` is correct

```go
// 1. Reflection-based code (encoding, ORM, RPC)
func Marshal(v any) ([]byte, error) // encoding/json

// 2. Truly polymorphic containers (rare in Go)
// most cases are better served by generics now

// 3. Logging / printing
slog.Info("event", "data", x) // logger takes any

// 4. Protocol boundary (you decode then check)
var payload any
if err := json.Unmarshal(body, &payload); err != nil { /* ... */ }
m, ok := payload.(map[string]any) // type-assert at boundary
```

### When `any` is wrong (use generics or concrete type)

```go
// BAD: any to model "anything that has a Key()"
type Item struct { Data any }
func Process(items []any) { for _, i := range items { /* ... */ } }

// GOOD: generic
func Process[T Keyed](items []T) { for _, i := range items { _ = i.Key() } }

// BAD: any to avoid declaring an interface
func Sum(nums []any) any {
    var total float64
    for _, n := range nums {
        switch v := n.(type) {
        case int: total += float64(v)
        case float64: total += v
        }
    }
    return total
}

// GOOD: generic with constraint
type Number interface { ~int | ~int64 | ~float64 }
func Sum[T Number](nums []T) T {
    var total T
    for _, n := range nums { total += n }
    return total
}
```

### Type-assert at boundaries, never deep in the code

```go
// BAD: any leaks deep into the codebase
func ServeJSON(w http.ResponseWriter, payload any) { /* ... */ }
func parse(payload any) any { /* ... */ }
func validate(data any) error { /* ... */ }

// GOOD: assert at the boundary, work with concrete types after
func ServeJSON(w http.ResponseWriter, payload any) {
    user, ok := payload.(*User)
    if !ok {
        http.Error(w, "bad payload", 500)
        return
    }
    // From here, user is *User — type-safe
    process(user)
}
```

Staff insight: `any` is to interfaces what `void *` is to C. Used at boundaries it's fine; spread through the body of your code, it tells future readers "the type system gave up." Since 1.18, **generics are the right tool** for almost every "this would work for many types" case.

---
8. Type Assertions and Type Switches

### Type assertion

```go
var i any = "hello"

// One-result form: PANICS on mismatch
s := i.(string)

// Two-result form: returns zero value + false on mismatch — preferred
s, ok := i.(string)
if !ok { /* handle */ }
```

### Type switch

```go
switch v := i.(type) {
case string:
    fmt.Println("string", v)
case int:
    fmt.Println("int", v)
case fmt.Stringer:
    fmt.Println("stringer", v.String())
default:
    fmt.Println("unknown", v)
}
```

Cases match in order; the first match wins. Concrete types win over interfaces; specific interfaces win over more general ones.

### Type assertion on interface types

You can type-assert to **interfaces** as well as concrete types — this asks "does this value also implement Foo?"

```go
// Common pattern: optional progress reporting
func Copy(dst io.Writer, src io.Reader) (int64, error) {
    if rt, ok := dst.(io.ReaderFrom); ok {
        return rt.ReadFrom(src)  // optimized path
    }
    // fallback: byte-by-byte copy
}
```

This is how `io.Copy` uses `io.ReaderFrom` / `io.WriterTo` for fast paths.

### Anti-pattern: type switching on your own types

```go
// BAD: a Service that type-switches on its inputs — you've reinvented inheritance
type Item interface { /* ... */ }
type Book struct{ Title string }
type Movie struct{ Title string }

func Process(i Item) {
    switch v := i.(type) {
    case *Book:  fmt.Println("book:", v.Title)
    case *Movie: fmt.Println("movie:", v.Title)
    }
}

// GOOD: polymorphism — let the type decide
type Item interface { Process() }
type Book struct{ Title string }
type Movie struct{ Title string }
func (b *Book) Process()  { fmt.Println("book:", b.Title) }
func (m *Movie) Process() { fmt.Println("movie:", m.Title) }
```

Type-switching is appropriate when:
- You're at a deserialization / RPC boundary.
- You're working with the standard library's `any` (logging, reflection).
- You're optimizing for a specific subtype (the `ReaderFrom` pattern).

Type-switching is inappropriate when:
- You own all the types.
- The "switch" will need to be updated every time someone adds a type.

Staff insight: if you find yourself adding a new case every time someone adds a new type, you're using a type switch where you should use polymorphism. Move the behavior onto the type.

---
9. Interface Embedding and Composition

Interfaces compose via embedding. The composed interface requires all embedded methods.

```go
type Reader interface { Read(p []byte) (int, error) }
type Writer interface { Write(p []byte) (int, error) }
type Closer interface { Close() error }

type ReadWriter interface { Reader; Writer }
type ReadCloser interface { Reader; Closer }
type WriteCloser interface { Writer; Closer }
type ReadWriteCloser interface { Reader; Writer; Closer }
```

### Compose, don't list

```go
// BAD: redeclare methods — adding a method to Reader requires updating every alias
type ReadWriter interface {
    Read(p []byte) (int, error)
    Write(p []byte) (int, error)
}

// GOOD: embed — methods stay in one place
type ReadWriter interface {
    Reader
    Writer
}
```

### Embedded interface in struct — implementing partial interface

A struct can embed an interface to "inherit" its method set. The methods come from the embedded interface value at runtime.

```go
type Logger interface { Log(string) }

type LoggingDB struct {
    Logger          // embedded interface; LoggingDB has a Log(string) method
    underlying *DB
}

func (ldb *LoggingDB) Query(q string) ([]Row, error) {
    ldb.Log("query: " + q) // promoted from embedded Logger
    return ldb.underlying.Query(q)
}

// Usage:
ldb := &LoggingDB{Logger: stdLogger, underlying: db}
```

If you embed `Logger` but don't set the field, calling `Log` panics with nil pointer. This is a common bug; either set it always or guard:

```go
func (ldb *LoggingDB) Query(q string) ([]Row, error) {
    if ldb.Logger != nil {
        ldb.Logger.Log("query: " + q)
    }
    return ldb.underlying.Query(q)
}
```

### Partial implementation via embedded interface

A useful pattern: implement an interface partially by embedding another implementation, overriding only what differs.

```go
type Storage interface { Get(string) ([]byte, error); Put(string, []byte) error; Delete(string) error }

// I want a "read-only" view of a Storage
type ReadOnlyStorage struct {
    Storage // embed — inherits all methods
}
func (r ReadOnlyStorage) Put(string, []byte) error { return errors.New("read-only") }
func (r ReadOnlyStorage) Delete(string) error      { return errors.New("read-only") }

// Get is inherited from the embedded Storage; Put/Delete are overridden.
```

Staff insight: interface embedding lets you build narrow contracts from existing ones (`io.ReadWriteCloser`) and partial implementations from full ones (`ReadOnlyStorage`). Use it; reject the urge to redeclare methods.

---
10. Sealed Interfaces — Controlling Implementers

Sometimes you want to *prevent* outside packages from implementing your interface. Go doesn't have a `sealed` keyword, but the unexported-method trick works:

```go
package events

type Event interface {
    Timestamp() time.Time
    isEvent() // unexported — only types in THIS package can satisfy
}

type LoginEvent struct { /* ... */ }
func (e *LoginEvent) Timestamp() time.Time { /* ... */ }
func (e *LoginEvent) isEvent()             {}

type LogoutEvent struct { /* ... */ }
func (e *LogoutEvent) Timestamp() time.Time { /* ... */ }
func (e *LogoutEvent) isEvent()             {}

// External package CANNOT implement Event — the isEvent method is unexported.
```

### Why use this

- **Exhaustive type switch.** A caller can `switch e.(type)` knowing the set is closed.
- **Versioning.** Adding a new variant doesn't break callers (who must handle all cases via `default`).
- **API stability.** Outsiders can't accidentally add behaviors.

### Caveat — testing requires you to expose a fake

If `Event` is sealed, your tests can't define a fake `Event`. Choices:
1. Expose a test-only fake from the package itself (`TestEvent`).
2. Don't seal — accept that external implementors exist.
3. Seal but provide a `NewTestEvent(opts...) Event` constructor.

Staff insight: sealing is rarely needed. Use it only when you have a small, closed set of variants — a sum type in disguise. Most Go interfaces benefit from being open. If you find yourself sealing many interfaces, generics may give you what you actually wanted.

---
11. Interfaces for Mocking — The Right and Wrong Shape

The classic Go interface use case is **testing**: replace a real dependency with a stub.

```go
// GOOD: small interface defined by the consumer
package svc

type userFetcher interface {
    Fetch(ctx context.Context, id string) (*User, error)
}

type Service struct { uf userFetcher }

func (s *Service) Greet(ctx context.Context, id string) (string, error) {
    u, err := s.uf.Fetch(ctx, id)
    if err != nil {
        return "", err
    }
    return "hi " + u.Name, nil
}

// In tests:
type stubFetcher struct{ user *User; err error }
func (s stubFetcher) Fetch(context.Context, string) (*User, error) { return s.user, s.err }

func TestGreet(t *testing.T) {
    s := &Service{uf: stubFetcher{user: &User{Name: "Alice"}}}
    got, err := s.Greet(context.Background(), "1")
    require.NoError(t, err)
    require.Equal(t, "hi Alice", got)
}
```

### Anti-pattern: exporting interfaces FROM the production package solely to enable mocks

```go
// BAD: provider exports interface PURELY for tests
package storage

type Storage interface { Get(string) ([]byte, error); Put(string, []byte) error }
type S3Storage struct{ /* ... */ }

// This interface exists because consumers want to mock — it has zero domain value here.
// It pollutes the API surface and creates coupling.

// GOOD: consumer defines its own narrow interface
package svc
type blobStore interface { Get(string) ([]byte, error) }
```

### Don't generate mocks from fat interfaces

```go
// BAD: mockgen Storage → 200-line mock file with stubs for every method
// even though your test uses 2 methods
//go:generate mockgen -package mocks Storage MockStorage

// GOOD: define a narrow interface, hand-write the stub
type onlyGet interface { Get(string) ([]byte, error) }
type fakeGet struct{ data map[string][]byte }
func (f fakeGet) Get(k string) ([]byte, error) { return f.data[k], nil }
```

### Mockgen vs gomock vs testify mocks vs hand-written

- **Hand-written stub** — best for small interfaces. Fast, readable, no generation.
- **testify/mock** — programmable matchers, when you need "called with X exactly once."
- **gomock** — heavier; useful for very complex interaction tests.
- **mockgen-generated** — bridges interface → mock; rarely worth the build complexity for small interfaces.

The Go community has been moving **away from mock-heavy testing** toward "real or fake-but-realistic" dependencies. A SQLite in-memory store often beats a mock DB.

Staff insight: mocks are a smell when they're complicated. A 50-line stub that exercises real behavior is usually more reliable than a 200-line mock with expectation matchers. Use frameworks when they earn their keep; default to a handwritten fake.

---
12. Generics vs Interfaces — When to Use Which

Go 1.18 added generics. The relationship to interfaces is subtle.

```go
// Pre-generics: interface for any-type behavior
func Sum(nums []float64) float64 { /* only works for float64 */ }

// With interface: works for any "Addable" but boxes — runtime cost + allocation
type Addable interface { Add(Addable) Addable }
func SumI(nums []Addable) Addable { /* ... */ }

// With generics: works for many types, compile-time, no boxing
func SumG[T Number](nums []T) T {
    var total T
    for _, n := range nums { total += n }
    return total
}
```

### The decision rule

```
Use generics when:
├── The function/type works for many types
├── The types share STRUCTURAL behavior (operators, common methods)
├── You don't need runtime polymorphism (mixed types in same slice)
└── Example: container types (List[T]), arithmetic, sorting, mapping

Use interfaces when:
├── Multiple concrete types share a behavior contract
├── Callers want to swap implementations at runtime
├── Different types live in the same collection
├── You want to mock for tests
└── Example: io.Reader, error, http.Handler, mock targets
```

### A mix is sometimes correct

```go
// Generic constraint that requires a method
type Closer interface { Close() error }

func CloseAll[T Closer](items []T) error {
    var errs []error
    for _, item := range items {
        if err := item.Close(); err != nil {
            errs = append(errs, err)
        }
    }
    return errors.Join(errs...)
}
```

### Generics don't replace interfaces — they extend them

Generics let you write code parametric in a type. Interfaces let you abstract over behavior. They solve different problems; often you'll combine them.

```go
// Generic over the item type; interface for behavior
type Repository[T any] interface {
    Get(ctx context.Context, id string) (*T, error)
    List(ctx context.Context) ([]*T, error)
}

// Concrete:
type UserRepo struct{ db *sql.DB }
func (u *UserRepo) Get(ctx context.Context, id string) (*User, error) { /* ... */ }
func (u *UserRepo) List(ctx context.Context) ([]*User, error)         { /* ... */ }

// Use anywhere we want:
var _ Repository[User] = (*UserRepo)(nil)
```

Staff insight: in 2026, the question "should this be an interface or a generic?" has a heuristic: if the abstraction is **same operation on different types**, use generics. If it's **different implementations of the same operation**, use interfaces. Many type-parameterized libraries use both — generic over the data, interface for the behavior.

---
13. The fmt.Stringer Interface and String() Magic

A single-method interface that controls how your type prints:

```go
type Stringer interface { String() string }

type User struct { ID string; Email string }
func (u User) String() string { return fmt.Sprintf("User(%s)", u.ID) }

fmt.Println(user) // User(u-123) — Stringer.String() invoked
```

### Gotchas

```go
// 1. Implementing String() with a Sprintf("%v", *u) inside is a panic/infinite loop
func (u *User) String() string { return fmt.Sprintf("%v", *u) }
// %v on a value calls String() → infinite recursion → stack overflow

// FIX: use %+v on a struct literal or build the string by hand
func (u *User) String() string { return fmt.Sprintf("User{ID:%s}", u.ID) }

// 2. Nil pointer with Stringer method panics on print
var u *User // nil
fmt.Println(u) // calls u.String() which dereferences nil → panic

// FIX: nil-check inside String()
func (u *User) String() string {
    if u == nil { return "<nil User>" }
    return fmt.Sprintf("User(%s)", u.ID)
}

// 3. Don't put sensitive data in String() — it'll show up in logs
func (u *User) String() string {
    return fmt.Sprintf("User(%s pw=%s)", u.ID, u.Password) // password in logs!
}
// Some teams forbid implementing Stringer for any type with secret fields.
```

### Same for fmt.GoStringer (`%#v`) and error.Error()

The pattern repeats:

```go
// GoStringer — for %#v
type GoStringer interface { GoString() string }
// error — for %s, %v
type error interface { Error() string }
```

Same anti-recursion rules apply.

Staff insight: implementing Stringer adds value; doing it carelessly creates panic or recursion. Always guard against nil; never use `%v` on the same type's value inside its own `String()`.

---
14. Sort.Interface and the "Interface as Algorithm" Pattern

```go
package sort

type Interface interface {
    Len() int
    Less(i, j int) bool
    Swap(i, j int)
}

func Sort(data Interface) { /* ... */ }
```

The trick: an algorithm is **decoupled from the data** by requiring the data to expose a tiny operational surface. Anything `sort` can do, you can do with your own type.

```go
type ByName []User
func (b ByName) Len() int           { return len(b) }
func (b ByName) Less(i, j int) bool { return b[i].Name < b[j].Name }
func (b ByName) Swap(i, j int)      { b[i], b[j] = b[j], b[i] }

sort.Sort(ByName(users))
```

### slices.SortFunc replaces this (1.21+)

```go
slices.SortFunc(users, func(a, b User) int {
    return cmp.Compare(a.Name, b.Name)
})
```

The 1.21+ `slices` package made the explicit interface form mostly unnecessary, but it's a foundational pattern to recognize. Many older libraries (`heap.Interface`, `flag.Value`) use the same shape.

Staff insight: the "interface as a small surface that an algorithm requires" pattern is one of Go's most elegant ideas. Recognize it when you see it; reach for `slices.*` and `maps.*` (1.21+) instead of writing your own when you can.

---
15. Functional Interfaces — When a func Type Is Cleaner

A single-method interface is equivalent to a function type with the same signature. Sometimes the function is cleaner:

```go
// Interface form — verbose for one method
type Greeter interface { Greet(name string) string }
type uppercaseGreeter struct{}
func (uppercaseGreeter) Greet(name string) string { return strings.ToUpper("hi " + name) }

// Function form — terse
type GreeterFunc func(name string) string
greet := GreeterFunc(func(name string) string { return strings.ToUpper("hi " + name) })
```

### The HandlerFunc pattern

`http.HandlerFunc` is the canonical example: a func type that adapts to the `http.Handler` interface:

```go
type Handler interface { ServeHTTP(ResponseWriter, *Request) }

type HandlerFunc func(ResponseWriter, *Request)
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) { f(w, r) }

// Usage — anonymous function becomes a Handler
http.Handle("/foo", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "hello")
}))
```

### Best of both — provide both

```go
package retry

type Operation interface { Do(ctx context.Context) error }
type OperationFunc func(ctx context.Context) error
func (f OperationFunc) Do(ctx context.Context) error { return f(ctx) }

func Run(ctx context.Context, op Operation, n int) error { /* ... */ }

// Callers can pass either a struct-with-method or a function
retry.Run(ctx, OperationFunc(httpCall), 3)
retry.Run(ctx, myCustomOp, 3)
```

### When function is better

- The interface has one method.
- The caller mostly wants to wrap a one-shot computation.
- There's no associated state to inject.

### When interface is better

- The interface has more than one method.
- The implementation has injected state (logger, config, dependency).
- The method is mutually-recursive with other methods on the same value.

Staff insight: a single-method interface and a function type are isomorphic. If 90% of your callers will pass anonymous functions, lead with the function type and add the interface wrapper for the 10% who want structured implementations. Mirror the `http.Handler`/`HandlerFunc` pair.

---
16. Interface Dispatch Cost — When It Matters

Calling an interface method involves an **itab** (interface table) lookup. The runtime stores a per-(interface, concrete-type) table mapping methods to function pointers.

```go
var w io.Writer = &bytes.Buffer{}
w.Write(b) // 1. read itab from interface header
           // 2. read Write method pointer from itab
           // 3. call indirectly through pointer
```

Cost:
- ~2-3 ns per call (modern CPU).
- Cannot inline (the function isn't known at compile time).
- Disables a number of compiler optimizations on the value.

### When it matters

```go
// BAD for hot loops: interface dispatch every iteration
func sumI(xs []Integer) int64 {
    var t int64
    for _, x := range xs { t += x.Int64() } // interface call ×N
    return t
}

// GOOD: pre-extract or generic
func sumG[T constraints.Integer](xs []T) T {
    var t T
    for _, x := range xs { t += x }       // direct add, no dispatch
    return t
}
```

Benchmarks of large loops can show **5-10x speedup** from interface → concrete or generic.

### When it doesn't matter

```go
// Almost everywhere else. An HTTP handler that does:
fmt.Fprintln(w, "hello")
// has 100µs+ of network I/O cost; the few ns of dispatch are invisible.
```

### Boxing — the other interface cost

```go
// When you assign a value to an interface, the runtime may need to copy it to the heap
var v interface{} = SmallStruct{X: 1, Y: 2}
// v's header points to a heap-allocated copy of SmallStruct

// Avoid in hot paths:
type Item struct { X, Y int }
var items []any  // every Item assigned in here gets heap-boxed
```

Staff insight: ~99% of code doesn't need to worry about interface dispatch cost. The 1% that does — hot loops, parsers, decoders — should benchmark and may need to drop the interface for direct or generic dispatch. Premature optimization here is its own anti-pattern.

---
17. Interfaces in the Critical Path — io.Reader / io.Writer Composition

The io package is the canonical example of how interface composition scales.

```go
// Anything that can read bytes
type Reader interface { Read(p []byte) (int, error) }

// Anything that can write bytes
type Writer interface { Write(p []byte) (int, error) }

// Bigger types compose
type ReadWriter interface { Reader; Writer }
type ReadCloser interface { Reader; Closer }

// Adapters — transform one Reader into another
type LimitReader func(r Reader, n int64) Reader            // returns a Reader
func TeeReader(r Reader, w Writer) Reader                  // reads from r, copies to w, returns Reader
func MultiReader(readers ...Reader) Reader                 // concatenates Readers
func Pipe() (*PipeReader, *PipeWriter)                     // in-memory pipe
```

This is why `io.Copy(dst, src)` works between **any** Reader and **any** Writer: a file, a buffer, an HTTP body, a gzipped stream, an encrypted stream, a TCP socket — all of them satisfy these tiny interfaces.

### Apply this lesson to your domain

```go
// BAD: a "DataSource" interface with 15 methods
type DataSource interface {
    Open() error
    Close() error
    GetByID(string) (Item, error)
    List() []Item
    Watch(chan<- Item)
    /* 10 more */
}

// GOOD: small interfaces that compose
type Opener interface { Open() error }
type Itemer interface { GetByID(string) (Item, error) }
type Lister interface { List() []Item }
type Watcher interface { Watch(chan<- Item) }
// implementations can satisfy any combination
```

Staff insight: when designing a new domain, model the operations as the smallest possible interfaces and let consumers pick the combinations they want. That's how you get the `io.Copy` magic for your own code.

---
18. The "Single-Implementation Interface" Smell

If an interface has exactly one implementation in your codebase **forever**, it might not be earning its keep.

```go
// SMELL: interface with one impl and no test mock
package user

type Service interface { Get(id string) (*User, error) }
type service struct { /* ... */ }  // ONLY implementation
func (s *service) Get(id string) (*User, error) { /* ... */ }
```

Reasons this can be a smell:
- Extra indirection in code reading.
- Verbose mocks.
- Compiler can't inline.

Reasons it can be OK:
- You're planning a second implementation imminently.
- You want to swap for tests.
- The interface is at a public boundary.

The check: ask "what's the second implementation?" If you can't name one with conviction, the interface may be premature.

### The "I want to mock it" case is real but often misapplied

Mocking is a fine reason — but it should be **the consumer-side narrow interface**, not a fat provider-side one.

```go
// GOOD: narrow consumer-side interface; one production impl, easy to fake in tests
package handler

type userGetter interface { Get(id string) (*User, error) } // only what handler uses

// Production wires *user.UserRepo as the userGetter; tests pass a struct stub.
```

Staff insight: the rule "every dependency goes through an interface" is over-prescribed. The real rule is: interface where you need swappability (tests, multiple impls, public contract); concrete where you don't. Mockability is a real reason for the former — but keep the interface narrow.

---
19. Interface Evolution — Adding Methods Is Breaking

The fact you must internalize: **adding a method to an interface is a breaking change** for every implementor.

```go
// 1.0
type Storage interface {
    Get(key string) ([]byte, error)
    Put(key string, val []byte) error
}

// 1.1 — added Delete
type Storage interface {
    Get(key string) ([]byte, error)
    Put(key string, val []byte) error
    Delete(key string) error  // ← every external implementor now broken
}
```

### Strategies for backward compatibility

```go
// 1. Define a new, larger interface — keep the old one
type Storage interface {
    Get(string) ([]byte, error)
    Put(string, []byte) error
}

type DeletableStorage interface {
    Storage
    Delete(string) error
}

// Code that supports both:
func cleanup(s Storage, key string) {
    if d, ok := s.(DeletableStorage); ok {
        d.Delete(key)
    }
}

// 2. Provide a default-implementing helper struct that callers can embed
type StorageBase struct{}
func (StorageBase) Delete(key string) error { return errors.New("not supported") }

// Implementors embed it and get a default Delete; they can override:
type MyStorage struct { StorageBase /* ... */ }
```

### Adding methods to interfaces YOU OWN

If you control both the interface and all implementors (closed system), adding is fine — you update them together. If anyone outside your module implements the interface, adding is breaking.

### Marking interfaces "do not implement outside this package"

Use the unexported-method "sealed" trick (§10). External code physically cannot implement it, so adding methods is non-breaking.

Staff insight: design interfaces for **today's actual needs**, not "we might want this later." If you genuinely think the interface will grow, start sealed; you can unseal later by removing the sentinel method, but you can't seal after the fact.

---
20. Interface Documentation — The Contract Lives in Godoc

An interface's *signature* is what the compiler enforces. Its *contract* — preconditions, postconditions, what panics, what's blocking — lives in godoc:

```go
// Read reads up to len(p) bytes into p.  It returns the number of bytes
// read (0 <= n <= len(p)) and any error encountered.
//
// Even if Read returns n < len(p), it may use all of p as scratch space
// during the call.
//
// If some data is available but not len(p) bytes, Read conventionally
// returns what is available instead of waiting for more.
//
// When Read encounters an error or end-of-file condition after
// successfully reading n > 0 bytes, it returns the number of bytes read.
// It may return the (non-nil) error from the same call or return the
// error (and n == 0) from a subsequent call.  An instance of this general
// case is that a Reader returning a non-zero number of bytes at the end
// of the input stream may return either err == EOF or err == nil.  The
// next Read should return 0, EOF.
//
// Callers should always process the n > 0 bytes returned before
// considering the error err.  Doing so correctly handles I/O errors that
// happen after reading some bytes and also both of the allowed EOF
// behaviors.
//
// Implementations of Read are discouraged from returning a zero byte
// count with a nil error, except when len(p) == 0.  Callers should treat
// a return of 0 and nil as indicating that nothing happened; in
// particular it does not indicate EOF.
//
// Implementations must not retain p.
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

`io.Reader` has ~25 lines of documentation for a 1-method interface. That's not overkill — it's the **only** way to know:

- What happens at EOF.
- That `n > 0 && err != nil` is valid.
- That implementations must not retain `p`.
- That `n == 0 && err == nil` is reserved for `len(p) == 0`.

### Document every interface you define

```go
// CacheStore is an opaque key-value store.
//
// Implementations must:
//   - Be safe for concurrent use by multiple goroutines.
//   - Return ErrNotFound (not a zero-value pair) for missing keys.
//   - Treat a TTL of 0 as "no expiry"; negative TTLs as an error.
//
// Get is non-blocking and must return within a few milliseconds in
// the common case. Implementations that go to the network must use
// the context for cancellation.
type CacheStore interface {
    Get(ctx context.Context, key string) (Value, error)
    Set(ctx context.Context, key string, value Value, ttl time.Duration) error
}
```

Staff insight: an interface without documented preconditions and postconditions is incomplete. The signature constrains the names and types; the godoc constrains the behavior. Implementations that pass the signature check but violate the godoc cause subtle bugs.

---
21. Dependency Injection — Interfaces As Wiring Seams

Go doesn't have a DI framework in the standard library (though Uber's fx, Google's wire, and others exist). The idiom is **constructor injection**: dependencies are passed in via constructor, typed as interfaces.

```go
// GOOD: constructor takes interfaces; main wires real impls
type Service struct {
    users    userRepo
    payments paymentRepo
    log      *slog.Logger
}

type userRepo interface { Get(ctx context.Context, id string) (*User, error) }
type paymentRepo interface { GetByUser(ctx context.Context, uid string) ([]*Payment, error) }

func NewService(u userRepo, p paymentRepo, log *slog.Logger) *Service {
    return &Service{users: u, payments: p, log: log}
}

// main wires:
func main() {
    db := db.Open(cfg.DBDSN)
    svc := NewService(
        &postgres.UserRepo{DB: db},
        &postgres.PaymentRepo{DB: db},
        slog.Default(),
    )
}
```

### Avoid these DI anti-patterns

```go
// 1. Global variables instead of injected interfaces
var DB *sql.DB
func GetUser(id string) (*User, error) {
    return DB.Query(...) // implicit dependency; untestable without rewriting global
}

// 2. Service locator pattern
type Container struct { /* huge bag of dependencies */ }
func (s *Service) Do() {
    db := s.container.Get("db") // dynamic; type unsafe; rebinds at runtime
}

// 3. Factory functions that create dependencies inside the constructor
func NewService() *Service {
    return &Service{db: postgres.New()} // can't substitute for tests
}
```

Staff insight: in Go, "dependency injection" usually means "pass interfaces as constructor args." Don't reach for a framework; the language plus constructors gets you 95% of the value. Where you do need wiring (large apps with hundreds of components), `wire` (compile-time) is more idiomatic than `fx` (runtime reflection).

---
22. Sum-Type Patterns — Closed Sets via Sealed Interfaces

Go doesn't have algebraic data types. The closest analog is a **sealed interface** with a small set of implementations.

```go
package event

// Event represents one of: Login, Logout, Click.
// External packages cannot implement this interface (sealed).
type Event interface {
    Timestamp() time.Time
    isEvent()
}

type LoginEvent struct {
    Time time.Time
    UserID string
}
func (e *LoginEvent) Timestamp() time.Time { return e.Time }
func (*LoginEvent) isEvent() {}

type LogoutEvent struct {
    Time time.Time
    UserID string
}
func (e *LogoutEvent) Timestamp() time.Time { return e.Time }
func (*LogoutEvent) isEvent() {}

type ClickEvent struct {
    Time time.Time
    URL string
}
func (e *ClickEvent) Timestamp() time.Time { return e.Time }
func (*ClickEvent) isEvent() {}
```

```go
// Caller handles ALL variants via type switch
func handle(e event.Event) {
    switch e := e.(type) {
    case *event.LoginEvent:  handleLogin(e)
    case *event.LogoutEvent: handleLogout(e)
    case *event.ClickEvent:  handleClick(e)
    default:
        // shouldn't happen — but defensive logging
        slog.Error("unknown event type", "type", fmt.Sprintf("%T", e))
    }
}
```

### Trade-offs

```
Pros:
├── Closed set — adding a variant is a package change visible in code review
├── Each variant has its own typed fields
├── No "Kind" field + tag-based switch (the C-union pattern)
└── Type-safe at compile time

Cons:
├── Adding a variant requires every type switch to be updated
├── No compiler enforcement of exhaustive matching (unlike Rust/Haskell)
└── A `default` case is mandatory hygiene
```

### When this beats other patterns

- **JSON / protobuf decoded into a polymorphic shape** (`oneOf` in proto3).
- **Domain events** with distinct payloads per type.
- **AST nodes** (parser pipeline; common in compilers).
- **Result types** (success/various-failure variants).

Staff insight: sum types via sealed interfaces are Go's nearest equivalent to Rust's `enum`. They're verbose but explicit. The compiler doesn't enforce exhaustive switches; you compensate with `default` + telemetry. For a small fixed set, this is cleaner than carrying a `Kind` field and a union of optional payloads.

---
23. Interface Method Sets — The Addressability Edge Cases

```go
type Closer interface { Close() error }

type DB struct { /* ... */ }
func (db *DB) Close() error { /* ... */ }

// *DB satisfies Closer.
// DB (value) does NOT satisfy Closer.

var c Closer
c = &DB{} // OK
c = DB{}  // COMPILE ERROR

// Auto-addressing only happens with addressable values:
db := DB{}
c = &db    // OK — &db produces *DB

// Map value is NOT addressable:
m := map[int]DB{1: {}}
c = &m[1]  // COMPILE ERROR — cannot take address of map element
```

### Implication for collection types

```go
// Slice of values — works because slice index IS addressable
dbs := []DB{{}, {}, {}}
for i := range dbs {
    var c Closer = &dbs[i] // OK
    _ = c
}

// Map of values — broken
m := map[string]DB{"a": {}}
// var c Closer = &m["a"]  // ERROR

// Solution: store pointers
m := map[string]*DB{"a": {}}
var c Closer = m["a"] // OK
```

### When this bites

You're refactoring; you change `m["a"].Close()` into a pattern that needs an interface. The map-of-values now doesn't compile. Either change the map to map-of-pointers (mutability semantics shift) or restructure.

### Don't confuse with "interface satisfaction by value vs pointer"

```go
type Greeter interface { Greet() }

type V struct{}
func (v V)  Greet() {}   // value receiver
type P struct{}
func (p *P) Greet() {}   // pointer receiver

var g Greeter
g = V{}      // OK — value receiver works on value
g = &V{}     // OK — value receiver auto-promotes via pointer

g = &P{}     // OK
g = P{}      // ERROR — pointer receiver doesn't satisfy from value
```

Staff insight: receiver style + addressability is the topic where Go's gentlest learning curve hides its sharpest edges. Make all methods on a type use the same receiver style; store collections of pointers if you need to take addresses.

---
24. Interface{}-Indexed Maps — Use Concrete Types

A common temptation: `map[any]any` as a "flexible" structure. Almost always wrong.

```go
// BAD: type-unsafe, no compiler help
attrs := map[any]any{}
attrs["user_id"] = "u-123"
attrs[42] = "what does this key mean?"

// 50 lines later:
uid, _ := attrs["user_id"].(string) // dance every time

// GOOD: define the schema with concrete types
type Attrs struct {
    UserID    string
    Timestamp time.Time
    Country   string
}
```

When `map[string]any` IS reasonable (parsed JSON, dynamic dispatch tables), assert at the boundary, work with concrete types after.

```go
var payload map[string]any
json.Unmarshal(body, &payload)

// At the boundary:
uid, ok := payload["user_id"].(string)
if !ok { return fmt.Errorf("user_id missing or not string") }
country, ok := payload["country"].(string)
if !ok { country = "unknown" }

// Now work with strongly-typed vars
process(uid, country)
```

Staff insight: every `map[string]any` is a deferred decision about the data's shape. Defer it as briefly as possible — assert at the boundary, then move to a typed struct.

---
25. slog.LogValuer — Custom Log Output via Interface

`slog` (1.21+) honors a hierarchy of formatting interfaces.

```go
type LogValuer interface { LogValue() slog.Value }

type User struct { ID string; Email string; Password string }
func (u User) LogValue() slog.Value {
    return slog.GroupValue(
        slog.String("id", u.ID),
        slog.String("email", u.Email),
        // Password intentionally omitted
    )
}

// In code:
slog.Info("login", "user", u)
// Output: ... user.id=u-123 user.email=alice@x.com  (no password)
```

Same pattern works for errors, custom types, time formats. The interface gives you per-type control over how things appear in logs.

Staff insight: implement `LogValue()` on every type that crosses log boundaries with structured data. It's the modern replacement for ad-hoc `slog.String("user_id", u.ID)` calls everywhere.

---
26. Interface Comparison — When `==` Works and When It Panics

Interface values are comparable with `==` and `!=`. The comparison checks **both type and value**:

```go
var a, b error
a = errors.New("x")
b = errors.New("x")
fmt.Println(a == b) // false — different *errorString pointers

a = io.EOF
b = io.EOF
fmt.Println(a == b) // true — same sentinel
```

### The panic case

If the **underlying type is not comparable**, `==` panics at runtime:

```go
var a, b any = []int{1, 2}, []int{1, 2}
fmt.Println(a == b) // PANIC: runtime error: comparing uncomparable type []int
```

Comparable: bool, numeric, string, pointer, channel, interface (whose underlying is comparable), struct (all fields comparable), array (element comparable).
Not comparable: slice, map, function.

### When you need to compare structurally

Use `reflect.DeepEqual`:

```go
reflect.DeepEqual([]int{1, 2}, []int{1, 2}) // true
```

…but `reflect.DeepEqual` is the wrong tool 95% of the time. If you need structural compare, use a typed `Equal()` method on your concrete type, or `cmp.Equal` from `github.com/google/go-cmp` for tests.

Staff insight: interface `==` is fine for sentinels and for known-comparable types; for anything user-provided, type-assert to a concrete type and compare in its native form. If you can't predict what's coming in, `cmp.Equal` with a typed option is safer than `reflect.DeepEqual`.

---
27. Anti-Patterns Reference

```go
// ────────────────────────────────────────────────────────────────────
// 1. Provider-side interfaces just to enable mocking
package storage
type Storage interface { /* exported solely for tests */ }
// → Consumer-side narrow interface

// ────────────────────────────────────────────────────────────────────
// 2. Fat interfaces (>3-4 methods on a non-protocol interface)
type UserManager interface { Create; Read; Update; Delete; List; Search; Audit; ... }
// → Split into single-purpose interfaces; compose where needed

// ────────────────────────────────────────────────────────────────────
// 3. Returning interfaces from constructors when only one impl exists
func NewS3Storage() Storage { return &s3Storage{} }
// → Return *S3Storage; let caller decide on interface

// ────────────────────────────────────────────────────────────────────
// 4. Typed nil returned from "interface error" function
func foo() error { var e *MyError; return e } // returns non-nil interface!
// → Return nil explicitly, or convert at the boundary

// ────────────────────────────────────────────────────────────────────
// 5. Mixed value/pointer receivers on same type
func (c Cart) Items() []Item { ... }
func (c *Cart) Add(i Item)   { ... }
// → All-value or all-pointer

// ────────────────────────────────────────────────────────────────────
// 6. `any` deep in business logic
func process(data any) any { ... }
// → Use generics or define a typed interface

// ────────────────────────────────────────────────────────────────────
// 7. Type-switching on your own types
switch v := i.(type) { case *Book: ...; case *Movie: ... }
// → Polymorphism via a Process() method on the interface

// ────────────────────────────────────────────────────────────────────
// 8. Sealed interfaces without exhaustive default cases
switch e := e.(type) { case *LoginEvent: ...; case *LogoutEvent: ... }
// → Always include a default { /* log unexpected */ }

// ────────────────────────────────────────────────────────────────────
// 9. Implementing String() with %v on the same value/type
func (u *User) String() string { return fmt.Sprintf("%v", *u) }
// → Hand-format or use %+v on a struct literal

// ────────────────────────────────────────────────────────────────────
// 10. Stringer/Error that doesn't nil-check
func (u *User) String() string { return u.ID } // panic if u == nil
// → if u == nil { return "<nil>" }

// ────────────────────────────────────────────────────────────────────
// 11. Single-implementation interface "just in case"
type Service interface { /* 12 methods */ }
type service struct { /* only impl */ }
// → Use concrete *Service until you have 2+ implementations or testing needs

// ────────────────────────────────────────────────────────────────────
// 12. Wide interface to enable mockgen
//go:generate mockgen -package mocks Storage
// → Define narrow consumer-side interface; hand-write the fake

// ────────────────────────────────────────────────────────────────────
// 13. Method names matching too many interfaces accidentally
// A type with `Read(p []byte) (int, error)` accidentally satisfies io.Reader
// → Use distinct method names for your domain (e.g., ReadConfig instead of Read)

// ────────────────────────────────────────────────────────────────────
// 14. Forgetting Unwrap() on a typed error so errors.Is/As don't traverse
type MyErr struct { Cause error }
func (e *MyErr) Error() string { return e.Cause.Error() }
// (no Unwrap) → errors.Is(err, ErrCause) fails
// → Implement Unwrap() error

// ────────────────────────────────────────────────────────────────────
// 15. Returning multiple types via interface{} — switch downstream
func parse(data []byte) (any, error) { ... } // caller must type-assert
// → Return a sealed interface; caller switches over a known closed set

// ────────────────────────────────────────────────────────────────────
// 16. Forgetting compile-time interface assertions
type MyHandler struct{}
func (h *MyHandler) ServeHTTP(...) { ... }
// → var _ http.Handler = (*MyHandler)(nil) at top of file

// ────────────────────────────────────────────────────────────────────
// 17. Names ending in -er for "manager" types
type UserManager interface { ManageUser() }
// → -er suffix is for behavior interfaces; if you can't name it that way,
//   it's probably not a coherent interface

// ────────────────────────────────────────────────────────────────────
// 18. Interface with map values where you need addressable values
m := map[string]V{}; var c Closer = &m["a"]  // ERROR
// → map[string]*V

// ────────────────────────────────────────────────────────────────────
// 19. Storing interface values where you need identity
m := map[Closer]bool{} // CLOSER is interface; comparable but identity-fragile
// → Use the concrete pointer type if you need stable identity

// ────────────────────────────────────────────────────────────────────
// 20. Interface dispatch in hot loops
for ... { x.Read(...) } // interface call per iteration
// → Extract concrete type or use generic; benchmark first

// ────────────────────────────────────────────────────────────────────
// 21. Embedding interface in struct without initialization
type X struct { Logger } // Logger is nil; panic on call
// → Always initialize the embedded interface field

// ────────────────────────────────────────────────────────────────────
// 22. Comparing interfaces with == when values may be uncomparable
var a, b interface{} = []int{1}, []int{1}
fmt.Println(a == b) // PANIC: comparing uncomparable type []int
// → Comparing interfaces is only safe when underlying types are comparable

// ────────────────────────────────────────────────────────────────────
// 23. Interface bloat names: Doer, Handler, Manager, Service, Processor
type Doer interface { Do() }
// → Be specific: type Closer, type Validator, type RateLimiter

// ────────────────────────────────────────────────────────────────────
// 24. Returning interface for fluent API
func (b *Builder) WithFoo() Builder { ... } // chained calls lose concrete type
// → Return *Builder (concrete pointer) for chaining

// ────────────────────────────────────────────────────────────────────
// 25. "Service locator" anti-pattern: passing a god-bag of dependencies
type Container struct { /* 30 fields */ }
func New(c *Container) *Svc { ... }
// → Constructor injection with typed interfaces, one per dependency

// ────────────────────────────────────────────────────────────────────
// 26. Returning interface{} from a generic-able function (pre-1.18 leftover)
func First(xs []any) any { return xs[0] }
// → Use generics: func First[T any](xs []T) T

// ────────────────────────────────────────────────────────────────────
// 27. Defining interface in test file used in production code
package svc_test
type fakeRepo interface { ... } // test-only — production svc imports svc_test? NO
// → Define production interface in package, fake struct in test file
```

---
28. Summary — Decision Reference

```
What to do                            When
─────────────────────────────────────────────────────────────────────────
Define interface at consumer          Default. Producer returns concrete; consumer
                                      describes only what it uses.

Define interface at producer          Only when the interface IS the abstraction
                                      (io.Reader, http.Handler, sql.Driver).

Compile-time _ assertion              Every type intended to satisfy an interface.
                                      Catches breakage at definition, not at use.

Accept interfaces                     Function parameters — small, narrow.
Return concrete types                 Function returns — caller assigns to interface
                                      if they want.

Use generics                          Same operation across many types.
Use interfaces                        Different implementations of the same operation.

Single method interface               Wherever possible. The standard library is the
                                      proof.

Embed interfaces                      To compose (io.ReadWriter) or to inherit a
                                      default impl (LoggingDB embeds Logger).

Sealed interface                      For sum types or closed sets where outside
                                      implementation is harmful. Add unexported method.

Type switch                           At boundaries (JSON, RPC, log). On sealed
                                      interfaces with `default` for safety. NOT for
                                      your own polymorphic types.

Type assertion                        At deserialization boundaries; for optional
                                      capabilities (io.ReaderFrom check); rarely else.

Pointer receiver methods              When the type has identity or mutates.
Value receiver methods                Small immutable structs (Point, Time).
Never mix value + pointer receivers on the same type.

Compile-time interface check          var _ Iface = (*MyType)(nil) on every type-iface
                                      pair.

slog.LogValuer                        Types that cross log boundaries; controls how
                                      they appear in structured logs.

Stringer                              Types that have a meaningful one-line repr.
                                      Never recurse via %v on the same type.

HandlerFunc-style adapter             Single-method interface + companion func type
                                      adapter. Mirrors http.Handler/http.HandlerFunc.

mockgen / gomock                      Only for fat interfaces or strict expectations.
                                      Hand-written stubs are cleaner for small ones.

interface{} / any                     Boundaries only. Type-assert and move on.

Generics                              Same-operation-many-types. Containers, math,
                                      sorting, batch helpers.
```

```
Anti-pattern grep checklist for code review:
─────────────────────────────────────────────────────────────────────────
type X interface {  (10+ methods)                → split it
return interface{   (from a constructor)         → return concrete
return MyError      (where err is *MyError)      → check nil first
func (x X) ...      and func (x *X) ...          → unify receivers
map[any]any                                       → typed struct
switch v := i.(type) { case *MyType: ... }        → polymorphism (when you own the types)
type Doer interface                               → name by behavior
exported interface PURELY for mocks               → consumer-side narrow
fmt.Sprintf("%v", *r) inside r.String()           → recursion bomb
embedded interface field never initialized        → panic on call
```

```
The five questions to ask of every interface in code review:
─────────────────────────────────────────────────────────────────────────
1. Is it defined where it's USED (consumer side), not where it's IMPLEMENTED?
2. Is it as small as possible? (Ideally 1-3 methods)
3. Does the concrete type have a compile-time assertion?
4. Are receivers all value OR all pointer (not mixed)?
5. Is "nil interface vs nil pointer" impossible by construction?
```

The overarching staff-level principle: **interfaces in Go are about what a caller needs, not what an implementor provides.** Java/C# interfaces describe a class's capabilities; Go interfaces describe a function's requirements. Get this inversion right and most of the "where should this interface live" debates resolve themselves.

The standard library spent 15 years modeling this discipline. `io.Reader` is 1 method because every reader in Go's universe only needs to be one method. `error` is 1 method because every fallible operation only needs to communicate one piece of information. `http.Handler` is 1 method because that's the entire surface area of "serve this request." Your interfaces should aspire to the same minimalism.

Every interface you make is a contract you must support — adding methods later breaks callers, removing methods breaks implementors. Make them small, name them by behavior (`-er`), put them next to the code that consumes them, and assert satisfaction at compile time. The mechanism is just one keyword; the discipline is the staff-level skill.
