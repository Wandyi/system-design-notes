```
staff level software engineer questions on effectively handling concurrency in golang. give different realistic sceanrios and case studies
```


I'll write these as a staff interviewer would pose them — scenario, the question, what a strong answer covers, and the common trap. They escalate in subtlety rather than topic.

---
1. The phantom goroutine leak

Production memory grows over hours. pprof shows 50k goroutines, mostly in chan receive. The service is a flight search aggregator fanning out to 8 GDS providers per request.

func search(ctx context.Context, q Query) ([]Result, error) {
results := make(chan Result)
for _, p := range providers {
go func(p Provider) {
r, _ := p.Search(ctx, q)
results <- r
}(p)
}
var out []Result
for i := 0; i < len(providers); i++ {
select {
case r := <-results:
out = append(out, r)
case <-ctx.Done():
return out, ctx.Err()
}
}
return out, nil
}

Question: Where's the leak, and how do you fix it?

A strong answer covers:
- When ctx.Done() fires first, the parent returns. Providers still in flight finish their HTTP call and block forever on results <- r because nobody is receiving.
- Fix: buffer results to len(providers). Senders never block on full buffer; once the parent returns, GC collects the channel and senders unblock.
- The provider's Search must actually honor ctx. Passing context isn't enough — net/http honors it, but a third-party SDK may ignore it and finish a 30-second call regardless. Wrap with a goroutine + select if you don't trust the SDK.
- Production hardening: a goroutines_in_flight gauge per provider, alert on growth.

Trap: "Use errgroup." Correct in spirit, but errgroup doesn't manage the result channel — the goroutine leak is on that channel, not on the error path.

---
2. First-N-of-M with a tail-latency budget

Same search service. Product says: return the first 5 results within 300ms, all within 2s, never block on a single slow provider beyond 2s.

Question: Design the concurrency model.

Strong answer covers:
- Two deadlines: a soft "early return" timer and a hard 2s ceiling.
- Main loop has three exits: (a) collected ≥5 AND 300ms elapsed → return what we have; (b) all M done → return; (c) 2s deadline → return what we have.
- Critical detail: when you return early at 300ms, in-flight goroutines should keep running and populate a warm cache for the next user. Don't cancel them when the request context cancels. Detach with context.WithoutCancel(ctx) (Go 1.21+) carrying its own 2s deadline.

bg := context.WithoutCancel(ctx)
bg, cancel := context.WithTimeout(bg, 2*time.Second)
// goroutines use bg, not ctx

Trap: Passing the request context all the way down means cancellation on early return kills the very background fetches that would have made the next user's search fast.

---
3. Cache stampede on a hot key

Pricing cache. One popular fare's TTL expires while 2,000 concurrent requests hold it. All miss, all call origin, DB CPU pegs.

Question: Solutions, and their trade-offs.

Strong answer covers:
- golang.org/x/sync/singleflight: collapse all in-flight loads on the same key into one. All callers share the result. This is the right primitive.
- Subtlety: if any caller cancels its context, you don't want to poison the shared call for everyone else. Use DoChan and select on the per-caller ctx; the inner loader function should use a detached context.
- Probabilistic early refresh (XFetch): start refreshing before TTL hits, weighted by remaining TTL and recompute cost. Smooths the cliff.
- Stale-while-revalidate: serve stale for up to 2× TTL while one background goroutine refreshes. Trades freshness for availability.

Trap: "Put a sync.Mutex around the lookup." You've now serialized all 2,000 requests on every miss — including misses for different keys if the lock is global. Per-key locking is what people usually mean, and singleflight already implements that correctly.

---
4. Atomic counter vs. consistent snapshot

A metrics package exposes Inc(), Dec(), and Snapshot() Stats where Stats has 6 counters. Inc/Dec is called millions of times/sec; Snapshot every 10s by the scraper. The current sync.Mutex shows heavy contention.

Question: You switch to atomics. What breaks, and how do you fix it?

Strong answer covers:
- Field-by-field atomic reads in Snapshot() no longer see a consistent moment. Two fields can be from "before" an event, four from "after." Whether that matters depends on the invariant the consumer expects.
- If consistency matters: sync.RWMutex where writers don't lock at all (atomic Inc/Dec) and only Snapshot acquires the write lock briefly. The read frequency is so low that the rare blocking is fine. Or do epoch-based double buffering: writers write to shard[epoch]; reader swaps epoch atomically, waits a grace period, reads the old shard.
- Per-P sharded counters (sharded by runtime.GOMAXPROCS) eliminate contention; snapshot sums all shards in O(P).
- False sharing: 6 atomic.Int64 fields in one struct can fall on the same cache line. Pad with _ [56]byte between hot ones if profiling shows it. Often unnecessary, but worth knowing.

Trap: "Atomics are always faster than mutexes." On low contention, a single mutex around a struct can be faster than six atomics because there's one cache-line invalidation instead of six.

---
5. Backpressure in webhook delivery

You run a webhook service. Producer emits 50k events/sec. Consumer (HTTP POST to customer URLs) does 5k/sec peak. Buffer holds 1M. One customer goes down for 4 hours.

Question: What happens, and design the system.

Strong answer covers:
- The buffer fills in 20 seconds. Three real options:
  a. Block the producer — OK only if upstream can apply pressure further up.
  b. Drop policy — drop newest, drop oldest, or drop random. Drop-oldest is often right for state-update webhooks ("only the latest order status matters").
  c. Spill durable — move overflow into Kafka/SQS/disk-backed queue. Bounded by disk, replays when the customer recovers.
- Bulkhead by customer: a per-customer queue with its own worker pool, so one dead destination can't starve healthy ones. Classic "noisy neighbor" pattern.
- Circuit breaker per destination URL: after K consecutive failures, fast-fail for T seconds. Saves the work of holding a goroutine for a full HTTP timeout.
- Retry with jitter and dead-letter after K attempts.
- Observability: queue depth and oldest-unacked-age per customer; drop counter; retry histogram. Without these you're flying blind.

Trap: Global queue + global worker pool. One slow destination consumes all worker capacity and every healthy customer's webhooks stall behind it.

---
6. Pipeline cancellation with errgroup

ETL: read CSV → enrich via API → batch insert. Each stage is goroutines connected by channels. Stage 3 fails on row 50,000 due to a constraint violation.

Question: How do stages 1 and 2 stop, and what's the cleanup order?

Strong answer covers:
- errgroup.WithContext. First returned error cancels the group's ctx. Every stage selects on ctx.Done() when sending or receiving.
- Producer pattern (golden rule): always close your output channel via defer close(out) so consumers' range terminates regardless of success or failure path.
- Send always under select:

g.Go(func() error {
defer close(out)
for _, item := range items {
select {
case out <- item:
case <-ctx.Done():
return ctx.Err()
}
}
return nil
})

- g.Wait() returns the first error; deferred resource cleanup happens after that.

Trap: Closing a channel from the receiver side, or from a non-sole sender. Will panic on the next send. Multiple senders need a sync.Once around close or a separate done-channel.

---
7. Distributed lock for a seat hold

Booking flow: user clicks "hold this seat." Lock must work across 8 instances. You implement Redis SETNX with a 30s TTL. A user takes 35 seconds because their card was being authorized.

Question: What's the bug, and how do you fix it?

Strong answer covers:
- The lock expired mid-work. Another instance acquired it for a different user. Both instances now believe they hold the seat.
- Lease extension (a renewer goroutine pushing the TTL out every 10s) reduces the window but doesn't eliminate it — if the renewer is paused (GC, hypervisor pause), the lease still lapses.
- Fencing tokens (Kleppmann): lock acquisition returns a monotonically increasing token; the downstream system (the inventory DB) rejects any write whose token is below the highest seen. This is the only thing that's safe against arbitrary process pauses.
- Single Redis isn't safe under failover with replication lag. Redlock is controversial — for correctness-critical paths, use etcd/Consul, or accept the latency hit.
- The pragmatic answer for a seat hold: don't use a distributed lock at all. The database is your source of truth:

UPDATE seats
SET held_by = $user, held_until = NOW() + INTERVAL '15 min'
WHERE id = $seat AND (held_by IS NULL OR held_until < NOW())

Check rows-affected. Atomic, durable, no lock service needed.

Trap: "Just make the TTL longer." Pushes the failure window out without closing it. Anything whose safety depends on wall-clock time is unsafe under GC / VM / network pauses.

---
8. Hot-reload config without tearing reads

A 40-field config struct is read on every request. Updated once a minute by a watcher. Today guarded by sync.RWMutex. RLock shows up in flame graphs.

Question: Your options?

Strong answer covers:
- atomic.Pointer[Config] (Go 1.19+). Writer builds a new Config and Stores the pointer atomically. Readers Load once at the start of their work and use the snapshot. No locks, no contention. The old config is GC'd when the last reader drops its reference.
- This works because Config is immutable after publication. Make this an explicit type discipline: unexported fields, accessor methods only.
- Older API for the same pattern: sync/atomic.Value. Prefer atomic.Pointer[T] in new code for type safety.
- sync.RWMutex is not "free for readers" — RLock touches a contended cache line. At very high read concurrency, the cache traffic itself dominates.

Trap: "I'll read each field with its own atomic." Now you have tearing: a handler reads MaxRetries (new value) and RetryDelay (old value) and applies an inconsistent config. Pointer swap publishes the whole struct atomically — that's the point.

---
9. select starvation

Hot event loop. One channel streams events, another is shutdown. Under load, shutdown isn't processed promptly.

for {
select {
case ev := <-events:
handle(ev)
case <-shutdown:
return
}
}

Question: What's happening?

Strong answer covers:
- select is pseudo-random across ready cases. But you spend most wall-time inside handle(ev), not at the select. By the time you re-enter, events is ready again. Even when shutdown does become ready, half the random coin flips go the other way.
- Fix is to prioritize:

for {
select {
case <-ctx.Done():
return
default:
}
select {
case ev := <-events:
handle(ev)
case <-ctx.Done():
return
}
}

- For weighted fairness across many data channels, build it explicitly — round-robin, or a priority pattern with a non-blocking high-priority check followed by the general select.

Trap: "Add a timeout to the events case." That punishes the happy path without prioritizing the signal you actually care about.

---
10. Per-route counter under hot contention

Flight search service updates a per-route popularity counter on every request. Today: map[string]int guarded by sync.Mutex. Profile shows 30% in lock acquisition at peak.

Question: Walk me through optimizations, ordered by impact.

Strong answer covers:
- sync.Map is not the right tool — it's optimized for read-heavy with mostly-stable keys, not write-heavy counters.
- Shard the map. [256]struct{ sync.Mutex; m map[string]int }. Hash the key mod 256, lock that shard. Contention drops near-linearly until cache effects dominate. Biggest single win for least complexity.
- Per-goroutine accumulation, periodic flush. Each request thread updates a local struct; a janitor goroutine merges into the global every 100ms. Global is touched rarely.
- If exact counts aren't required, count-min sketch or HyperLogLog gives sublinear memory and lock-free updates.
- For metrics specifically, push it out of the hot path: prometheus.CounterVec already does sharded atomic counters internally. Don't roll your own for an observability use case.

Trap: Jumping to a lock-free hashmap. They are notoriously hard to write correctly in Go (no hazard pointers, no epoch GC built in). Sharding gets you 90% of the wins for 10% of the risk.

---
11. Graceful shutdown with deadlines

HTTP server, two background workers, DB pool. SIGTERM arrives. Kubernetes will SIGKILL the pod in 30s.

Question: Sequence the shutdown.

Strong answer covers:
- Order is forced by dependency direction:
  a. Flip /readyz to 503 so the load balancer stops sending new traffic.
  b. http.Server.Shutdown(ctx) with a ~20s ctx. Drains in-flight requests.
  c. Cancel the top-level context that background workers select on. They finish their current unit and stop picking up new.
  d. errgroup.Wait() for workers, bounded by another timeout.
  e. Close the DB pool last — handlers and workers needed it through step 4.
- Use signal.NotifyContext to turn SIGTERM into ctx cancellation.
- Kubernetes detail: preStop: sleep 5 is the canonical hack to bridge the gap between kubelet sending SIGTERM and Endpoints/Service propagation actually stopping new traffic.
- Subtle: handlers that ignore r.Context() will run to completion. Shutdown waits for handler return; it doesn't kill goroutines. If a handler does a 60s DB query that ignores ctx, your 20s drain budget is gone.

Trap: Deferring DB pool close before Shutdown returns. You'll close the pool while handlers are mid-query.

---
12. The memory-model bug that "works on my machine"

var inst *Cache
func Get() *Cache {
if inst == nil {
inst = newCache()
}
return inst
}

Question: Code review.

Strong answer covers:
- Data race: concurrent reads and writes of inst with no synchronization. The race detector will flag it.
- Even without torn writes, the Go memory model gives no happens-before edge between newCache()'s field writes and another goroutine's read of inst. A reader can observe inst != nil but see fields at zero values. Real on ARM, masked on x86 (stronger memory model).
- Fix with sync.Once:

var (
inst *Cache
once sync.Once
)
func Get() *Cache {
once.Do(func() { inst = newCache() })
return inst
}

sync.Once.Do provides happens-before from the initializing call to every subsequent caller.
- Alternatives: package-level init() (eager); atomic.Pointer with CAS (lock-free hot path with rare init).

Trap: "Use double-checked locking with a plain if inst != nil outside the lock." Same memory-model bug. In Go, use sync.Once; don't reinvent.

---
These map roughly onto the topic clusters a staff interviewer probes — goroutine lifecycle (1), latency-aware fan-out (2), caching coordination (3), atomics vs locks (4), bounded resources (5), pipelines (6), distributed coordination (7), publication patterns (8), scheduler nuance (9), high-contention data structures (10), lifecycle/shutdown (11), and the Go memory model (12).
