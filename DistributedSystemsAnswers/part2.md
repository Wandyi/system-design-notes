Dependency cycle: Service A calls B, B calls C, C calls A for enrichment. Under load, all three services deadlock their thread pools waiting for each other.

Distributed Deadlock via Service Cycle

Root cause

Each service has a bounded concurrency resource — a thread pool, an HTTP client connection pool, an event-loop queue, a database connection pool. Under load, the cycle A→B→C→A means:

1. A's worker holds a slot waiting for B's response.
2. B's worker holds a slot waiting for C's response.
3. C's worker holds a slot waiting for A's response.
4. A has no free workers to serve C, because they're all in step 1.
5. The cycle closes faster than slots free. Total deadlock.

The thread pool isn't the bug — it's doing exactly what a bounded resource should do: apply backpressure. The bug is that synchronous call cycles between services are a latent deadlock. Without load, each call returns before the cycle closes; with load, it doesn't. The system has a hidden critical traffic level where it transitions from healthy to deadlocked, with no graceful intermediate state.

The structural analogy is lock-ordering violations in concurrent programming. The classic fix there is to impose a global acquisition order. The same fix applies between services: forbid cycles in the dependency graph. The thread pool exhaustion is the surface; the cycle is the defect.

Reframe the question

The framing — "three services deadlock under load" — invites tactical fixes (bigger pools, shorter timeouts, more retries). Those mitigate; none of them remove the deadlock potential. The staff-level move: treat the cycle itself as the defect. Mitigations buy time for the architecture fix; they're not the fix.

The secondary insight: how did the cycle get introduced? Usually A→B existed forever, B→C was added in some release, then somebody added "just one call back to A for enrichment" in a later PR with no architecture review. Cycles enter incrementally and aren't visible without explicit dependency tracking. So the meta-question is: why did nothing in your process stop this from being introduced in the first place?

Design space

1. Break the cycle (the only fix that removes the deadlock; everything else mitigates)

Several techniques, pick the one that fits the actual data flow:

- Push the data, don't pull it. If C calls A for enrichment, A can publish enrichment data to a stream; C maintains a local projection (see the FK-across-services answer). C's request path never touches A. This is usually the right answer when the call is for relatively static reference data.
- Extract the shared piece. A→B→C→A often decomposes: there's some logic in A that C actually needs, but it's not really A's domain — extract it into a new service D. Now A→D, B→C, C→D. The cycle is gone because D has no callers among {A, B, C}.
- Merge. If A and C have a tight bidirectional dependency on the critical path, the service boundary between them may be fictional. Two services that can't operate without each other are one service split by accident — fix the boundary.
- Pre-fetch upstream. If A→B and C needs A's data, can A pass that data down to B (and onward to C) in the original request? The enrichment becomes data on the call path rather than a callback.
- Reverse the direction. Sometimes C→A is actually expressible as A→C in another form ("compute enrichment before dispatch" rather than "fetch enrichment during processing").

The architectural fix is the only one that prevents recurrence. Everything below is defense in depth.

2. Bulkhead per downstream (mandatory; mitigates the deadlock and many other failure modes)

Within each service, partition the concurrency budget by downstream:

- A has 200 total worker threads (or 200-concurrency async budget).
- Cap calls-to-B at 50.
- Cap calls-to-C at 30.
- The remaining 120 are available for serving A's own requests.

If B saturates, A's "calls-to-B" pool fills, those requests fail-fast or queue with bounded depth, but A still has capacity to serve C's callbacks. The cycle no longer takes down the whole system; instead the symptom degrades to "A→B is failing, A is otherwise healthy."

This is the Hystrix/Resilience4j bulkhead pattern, or service-mesh upstream-cluster connection-pool limits in Envoy/Istio/Linkerd. A single shared HTTP client pool for all downstreams is the anti-pattern: one slow downstream eats all the threads, regardless of cycle.

3. Layered timeouts with the right direction

Timeouts must be configured so the downstream gives up before the upstream does. Otherwise the upstream's timeout fires while the downstream is still consuming a thread, multiplying the resource cost:

- A→B timeout: 800ms
- B→C timeout: 500ms (shorter than A→B)
- C→A timeout: 300ms (shorter than B→C)

This guarantees threads return to the pool even on slow downstream; the cycle still exists logically but can't hold threads indefinitely. Without this layering, retries amplify the saturation rather than relieving it.

Pair with circuit breakers: after N consecutive failures or sustained P99 breaches, open the circuit and fail-fast for a cool-off. This both protects the upstream from piling on threads and gives the downstream a window to recover.

4. Async I/O raises the saturation threshold

The Java-style "one OS thread per in-flight request" model is the worst case for cycle deadlock — threads are expensive and bounded. Move to:

- Async I/O / reactive frameworks: Netty + Project Reactor / WebFlux, Vert.x. A waiting request holds state, not a thread.
- Go goroutines, Rust async, Node.js — runtimes where in-flight requests cost kilobytes, not megabytes.
- Virtual threads (Java 21+ Loom) — preserves the synchronous programming model but removes the OS-thread cost.

This doesn't remove the cycle, but the saturation threshold moves from "thousands of concurrent requests" to "millions." The cycle becomes a memory/file-descriptor problem rather than a thread problem at the same load — and "millions of in-flight requests" is the kind of saturation that's visible long before total deadlock. Still not a fix; still need to break the cycle.

5. Async messaging instead of sync calls

For interactions that don't need a synchronous response on the user's request path, use events: C emits "needs enrichment for X," A handles it asynchronously, response flows back through a separate channel or stays asynchronous. No thread is blocked waiting. No cycle in the synchronous path.

Trade-off: user-facing "give me this answer now" requests can't be async without UX changes. But a surprising fraction of cross-service calls can be async without anyone noticing — notifications, denormalization, materialization, side-effects.

6. Detect cycles before they ship

- Static dependency declarations: services declare their downstreams in a manifest; a CI check builds the graph and fails the build on any cycle. Backstage and similar internal platforms support this.
- Trace-based detection: distributed tracing (Tempo, Jaeger, etc.) emits actual call patterns. Periodically build the observed call graph and alert on cycles, including indirect ones longer than 2 hops.
- Architecture review for new cross-service calls: any new downstream addition should require checking the cycle invariant. "Does this introduce a cycle with anything in the transitive closure?"

The cheap version: a CI job that builds the graph from declared dependencies and fails on any cycle. Once it's in place, this entire class of bug stops being introduced.

Anti-patterns

- "Increase the thread pool size." Buys time, doesn't remove the deadlock. Moves the saturation threshold higher without changing the shape of the failure.
- "Longer timeouts so requests have time on slow days." Longer timeout = longer thread occupancy = lower effective concurrency capacity. Tighter timeouts with explicit retry budgets are almost always better.
- Aggressive retries on the upstream side. Each retry consumes another thread; retries during saturation accelerate it. Use bounded retries with jittered backoff and a circuit breaker.
- Single shared HTTP/connection pool for all downstreams. One bad downstream blows everything up.
- Treating a cycle deadlock as a "race condition" or "flake." It's a deterministic outcome of the architecture above a critical load. It will recur, almost always at the worst possible moment.
- Trusting that the cycle "doesn't matter at our traffic." Cycles get worse exactly when you most need the system to work — peak load, retry storms after a partial outage, traffic redistribution after a failover.
- Hiding the cycle behind async wrappers (CompletableFuture, Promise) without changing the underlying connection-pool semantics. Looks async, still blocks the connection pool, still deadlocks. The runtime matters, not the surface API.
- Adding the cycle via a "small enrichment" PR with no architecture review. This is how every cycle in production was introduced.

Operational rigor

- Per-downstream pool saturation metrics, not just per-service. "Calls-to-B pool at 95%" is the leading indicator; "A is unhealthy" is the trailing one.
- Distributed tracing with cycle detection, alerting when the call graph contains a cycle (planned or unplanned).
- Service dependency graph, refreshed from observation, not from documentation. The docs are always wrong by the time the cycle exists.
- Chaos test the slow-downstream scenario: inject latency or partial failure into one service and verify that (a) the bulkheads contain it, (b) the cycle doesn't deadlock the whole system, (c) circuit breakers actually open. The first time you find out your bulkheads aren't real should not be during an incident.
- Critical-path inventory: which services are on the user's synchronous request path? Anything in that set is a candidate for cycle bugs; anything outside it should arguably be async messaging.

What I'd actually do

1. Break the cycle. Treat the architecture as the defect. Most commonly: C maintains a local projection of A's enrichment data via events; C never calls A on the request path. Other cycles get extract-shared-service or merge treatment.
2. Bulkhead per downstream in every service, with strict caps. One bad downstream cannot exhaust the whole concurrency budget.
3. Layered timeouts where downstream timeouts are tighter than upstream, plus circuit breakers per downstream.
4. Move to an async runtime (virtual threads, async/await, reactive) so the saturation threshold isn't measured in thousands. Doesn't fix the architecture; gives you more room.
5. CI cycle detection on declared or traced dependencies. Make cycles literally impossible to ship without explicit override.
6. Chaos test the failure modes: slow downstream, partial outage, retry storms. If the system survives, you actually have what you think you have.

Bottom line

A service dependency cycle is a latent deadlock with a critical-load threshold; under retries, failovers, or traffic spikes, you eventually cross it and the entire cluster goes from healthy to unrecoverable in seconds. The lasting fix is to break the cycle architecturally — push data via events instead of pulling synchronously, extract shared pieces into acyclic dependencies, or merge boundaries that were fictional. The defense-in-depth — per-downstream bulkheads, layered timeouts with circuit breakers, async runtimes, CI-level cycle detection — keeps the system survivable while the architecture catches up and protects against many adjacent failures (retry storms, gray-failure cascades, slow downstream dragging the cluster). But none of those remove the cycle itself; only the architecture does. The fact that a cycle was introduced at all is a process question: every cross-service-call PR should require a cycle check, and once CI enforces it, the class of bug stops recurring.

✻ Worked for 2m 6s

❯ Rate limiting by the wrong dimension: You rate-limit by IP address. A corporate NAT puts 10,000 users behind one IP. Legitimate users are throttled. Meanwhile, an attacker with a
botnet has thousands of unique IPs and is untouched.

Rate-Limiting by the Wrong Dimension

Root cause

The framing collapses three different problems into one limit:

1. Resource fairness (one tenant shouldn't starve others).
2. Abuse prevention (no actor should be able to credential-stuff, scrape, or scalp).
3. DoS protection (infrastructure shouldn't fall over under volumetric load).

These are different problems with different right answers. IP address is a network identity — it's a tolerable proxy for problem 3 but a poor proxy for 1 and 2, because in 2026 the IP→user relationship is many-to-many in both directions:

- One IP, many users: corporate NAT, mobile carrier CGNAT (the entire mobile internet, in many countries), university networks, public Wi-Fi, VPN exits, Tor.
- One user, many IPs: residential proxy services ($50/month for 10M+ residential IPs), botnets, mobile IPv6 (a single user gets a /64; cheap rotation), AWS/GCP egress, dual-stack IPv4/v6, mobile devices on roaming.

So "per-IP limit" punishes the population that least deserves it (CGNAT users, enterprise) and fails to constrain the population it's meant to (botnets, distributed scrapers, credential stuffers). The two failure modes the question describes aren't a coincidence — they're symmetric consequences of choosing a dimension that's neither a tight upper bound on users nor a tight lower bound on attackers.

Reframe the question

Rate limiting is identity × action. Before you can bound "how much can this thing do?" you have to define "what is this thing?" If you can't establish identity for a request, you can't rate-limit it meaningfully — you can only do volume-based DoS protection, which is a different and weaker tool.

The staff-level move is to stop looking for "the right dimension" and instead layer multiple dimensions, each chosen for its threat model:

┌─────────────────────┬───────────────────────────────┬────────────────────────────────┬──────────────────────────────────────────────────────┐
│       Threat        │   What the attacker varies    │      What stays constant       │                   Right dimension                    │
├─────────────────────┼───────────────────────────────┼────────────────────────────────┼──────────────────────────────────────────────────────┤
│ Credential stuffing │ usernames tried               │ the attacker                   │ per-username AND per-IP AND global failed-login rate │
├─────────────────────┼───────────────────────────────┼────────────────────────────────┼──────────────────────────────────────────────────────┤
│ Account takeover    │ password attempts on one user │ the target account             │ per-user-being-attempted                             │
├─────────────────────┼───────────────────────────────┼────────────────────────────────┼──────────────────────────────────────────────────────┤
│ Scraping            │ URLs crawled                  │ session/fingerprint            │ per-session + behavioral                             │
├─────────────────────┼───────────────────────────────┼────────────────────────────────┼──────────────────────────────────────────────────────┤
│ API abuse (paid)    │ API calls                     │ API key                        │ per-API-key                                          │
├─────────────────────┼───────────────────────────────┼────────────────────────────────┼──────────────────────────────────────────────────────┤
│ Volumetric DoS      │ request volume                │ network origin                 │ per-IP/ASN/edge                                      │
├─────────────────────┼───────────────────────────────┼────────────────────────────────┼──────────────────────────────────────────────────────┤
│ Signup fraud        │ new accounts created          │ payment / device / fingerprint │ per-payment-method, per-fingerprint, per-IP          │
└─────────────────────┴───────────────────────────────┴────────────────────────────────┴──────────────────────────────────────────────────────┘

No single dimension covers all these. The mistake in the question is trying to make IP do all of them.

Design space

1. Multi-layer rate limiting (the architecture)

Each layer is in the right place for its threat model:

- Edge / WAF / CDN (Cloudflare, Akamai, Fastly, ALB): IP- and ASN-based limits at very high thresholds (catch volumetric only), plus TLS/HTTP fingerprinting (JA3/JA4), bot scoring, geographic anomalies, known-bad-IP feeds. This is where you stop floods, not where you stop credential stuffing.
- Application layer: per-authenticated-identity limits — user, account, API key, tenant. This is where fairness and abuse prevention live, because authenticated requests have real identity.
- Endpoint-specific layer: high-risk endpoints (login, signup, password reset, expensive search, payment) get composite limits — multiple buckets evaluated in parallel, the most restrictive wins.
- Global rails: a global limit on total failed-login rate, total signups/min, etc. Catches distributed attacks that defeat per-key limits by spreading across keys.

2. Composite buckets on sensitive endpoints (the trick that catches the question's botnet)

For login as the canonical example, evaluate all of these in parallel:

allow if all hold:
bucket(per username being attempted)  → catches single-account brute force
bucket(per source IP)                  → catches one-attacker-many-users (CGNAT-tolerant if sized for it)
bucket(per device fingerprint)         → catches IP rotation
bucket(global failed-logins / second)  → catches distributed credential stuffing

The distributed-botnet attack from the question defeats per-IP (each IP is below threshold) but trips the global failed-login bucket and the per-username bucket (because they're trying common usernames). The corporate NAT case passes per-username (each user is attacking one account) and trips the per-IP bucket only if it's set too low for that ASN — which is fixable by sizing the per-IP bucket for legitimate enterprise volumes, since per-username protects you from the actual brute force.

Each bucket is sized for its threat, not as a one-size-fits-all knob. The per-IP bucket can be generous (10k/min for an enterprise NAT) because the per-username bucket is the one stopping the brute force.

3. Identity-equivalent signals when IP is unreliable

For anonymous traffic where you have no account identity, get an identity-equivalent signal:

- TLS / HTTP fingerprinting (JA3, JA4, HTTP/2 fingerprint): a probabilistic identity that survives IP rotation. Most bots have characteristic fingerprints that mismatch real browsers.
- Behavioral / browser challenges (Cloudflare Turnstile, hCaptcha, Akamai Bot Manager, PerimeterX): invisible-or-low-friction challenges that produce a "real client" signal. The challenge itself is a per-actor rate limit because solving N challenges costs effort.
- Mobile attestation (Apple App Attest, Google Play Integrity): cryptographic proof from the OS that this is a real device running your real app. Largely defeats mobile botnets when used correctly.
- Privacy-preserving tokens (PrivacyPass, Private Access Tokens, supported by Apple, Cloudflare, others): cryptographic "one-time tokens" that prove a real client without identifying the user. Used to rate-limit anonymous flows without IP.

These give you a per-actor rate-limit dimension that survives the failure modes of IP.

4. Risk-based, not hard-rules-based

A staff-level rate-limit system isn't a static table; it's a risk score with action gating:

- Risk score combines signals: IP reputation, ASN, fingerprint match, time-since-signup, payment-method history, behavior consistency, geo anomalies.
- Response varies by score: low risk → pass; medium → challenge with CAPTCHA or step-up auth; high → throttle; very high → block + log.
- Velocity vs. baseline: a corporate NAT IP doing 1,000 logins/min may be normal for that ASN historically — compare to peer baselines, not absolute thresholds.

This converts "yes/no per IP" into "graduated friction per risk." Legitimate enterprise users see a transparent or low-friction experience; bots see escalating friction. The cost asymmetry is the actual defense.

5. Graceful response, not hard 429

When a limit fires on a plausibly legitimate user (corporate NAT under burst), 429 is a customer-relations problem. Better options:

- Soft throttle: introduce latency rather than failure. Bots care about throughput; humans rarely notice 200ms of added latency.
- Challenge: CAPTCHA / Turnstile. If they're human, they pass; if they're a bot, they don't.
- Step-up auth: require MFA or re-login. Legitimate users authenticate; bots don't.
- Honest backpressure: 429 + Retry-After for machine-to-machine clients that respect HTTP semantics.

Hard 429 is for the last resort, not the first response.

6. Allowlist as escape hatch, not as defense

Maintained allowlists for known enterprise IPs exist because the system has to support corporate users; they're not a defense, they're an operational escape valve. Run them carefully: ownership, expiration dates, audit trail, regular review. An allowlist with 5,000 entries and no owner becomes its own attack surface.

Anti-patterns

- One global threshold for everyone. Either tight (kills enterprise NAT) or loose (useless against attackers). No setting is correct because IP isn't the right identity.
- Tightening the IP limit when attacked. Punishes legitimate users; the distributed attacker is below threshold either way. You're scoping the wrong variable.
- Trusting X-Forwarded-For from arbitrary upstream. Trivially spoofed. Strip it at your edge and append only the verified client IP. Same for X-Real-IP.
- Whitelisting individual corporate IPs as the primary mitigation. Doesn't scale, becomes ops debt, and a determined attacker can compromise or impersonate one whitelisted IP.
- No rate limit on unauthenticated endpoints. Login, signup, password reset, search, public APIs — exactly where credential-stuffing and scraping live.
- Failing closed on the rate-limit store outage. During the exact incident where rate limiting matters most, your rate-limit store goes down → block all traffic → user-facing outage from your own protection. Fail open with a circuit breaker is usually right, but instrument it so you know you're flying without rate limits.
- Conflating DoS protection with abuse prevention. They live at different layers; trying to do both with one limit fails at both.
- Rate limit decisions logged but not surfaced as a metric. When 30% of legitimate logins start getting throttled, you should find out in seconds — not from a customer ticket.
- Limits set by guess. They should be derived from the observed P99 of legitimate traffic for that endpoint and that key dimension, with headroom. Re-tune quarterly.

Operational rigor

- Rate-limit hit-rate metrics, broken down by dimension, endpoint, user tier, ASN. A jump in hit rate is either an attack or a legitimate-traffic anomaly — either one warrants attention.
- False-positive feedback loop: a way for users (and CS) to flag "I was blocked legitimately." Tag the dimensions and review weekly. If 80% of false positives are CGNAT, the per-IP threshold for that ASN is wrong.
- Continuous calibration. Traffic distributions change; limits set 18 months ago are wrong now. Re-derive from observed P99 quarterly.
- Distributed implementation: rate-limit counters in a fast shared store (Redis Cluster, DynamoDB) with consistent hashing by key. Local in-process counters with periodic global sync for very hot endpoints. Use monotonic time, not wall clock — token buckets break under clock skew.
- Adversarial testing: a red-team exercise from a distributed proxy network or simulated botnet. If your rate limits don't catch you, fix them before someone else exercises them.
- Retry-After and RateLimit-* headers for API consumers, per the HTTP RateLimit headers draft. Lets well-behaved clients back off correctly.

What I'd actually do

1. Stop rate-limiting by IP for authenticated requests. Rate-limit by user/account/API key. IP becomes a secondary signal for anomaly detection, not the primary limit.
2. For unauthenticated abuse-prone endpoints, use composite per-dimension buckets: per-target-identity (e.g., per-username on login), per-IP (sized for enterprise NAT), per-fingerprint, plus global rails on failed events. The composite catches all four corners of the threat matrix.
3. Add identity-equivalent signals (Turnstile / mobile attestation / browser fingerprinting) so anonymous flows have a per-actor dimension that survives IP rotation. The shift from "block IP" to "challenge actor" is the architectural change that matters.
4. Risk-based response: graduated friction (latency → challenge → step-up auth → block) based on a composite score, not a single boolean.
5. Manage an enterprise allowlist as an escape hatch with explicit ownership, not as the defense.
6. Instrument and tune: hit-rate metrics, false-positive feedback, quarterly re-calibration. Static limits become wrong over time.

Bottom line

Rate-limiting by IP fails both legitimate users and attackers because IP is the wrong identity for the threat. The fix isn't a better single dimension — it's recognizing that rate limiting is identity × action, and identity comes from different sources for different threats: account ID for authenticated abuse, target identity (the username being brute-forced) for credential stuffing, fingerprint/attestation for anonymous bots, IP/ASN only for volumetric DoS. The architectural move is multiple composite buckets evaluated in parallel, each sized for its specific threat, combined with identity-equivalent signals (fingerprint, attestation, browser challenges) where IP can't carry the load. The enterprise-NAT problem and the distributed-botnet problem are symmetric, and the same architecture solves both: per-username catches the brute force regardless of IP source, and the per-IP bucket can be sized generously for enterprise because it's no longer the primary defense — it's the volumetric backstop.

✻ Cooked for 2m 27s

❯ Redlock safety violation: A Redlock across 5 Redis nodes acquires the lock on 3 nodes (majority). One node's clock jumps forward, expiring the lock early. A second client acquires the
lock on that node + 2 others = 3 nodes. Two clients hold the "lock" simultaneously.

Redlock Safety Violation

Root cause

Redlock's safety model rests on three assumptions that the question shows breaking:

1. Bounded clock drift across the Redis nodes (so TTLs expire at roughly the same wall-clock instant).
2. Bounded process pauses on the client (so the client doesn't sleep through its own TTL).
3. Bounded network delay (so the lock isn't already expired by the time messages arrive).

When any of these assumptions fails, Redlock's "majority of 5" loses meaning — because two different majorities can exist at the same wall-clock moment if some nodes think the lock expired and others don't. The clock-jump scenario in the question is one of several known attacks; others include:

- Client GC / OS pause: client acquires lock, JVM stops-the-world for 30s, TTL expires globally, second client acquires, original client wakes and continues with a stale "lock."
- Network delay between lock and resource: by the time the client's write reaches the protected resource, the lock has already expired and been re-acquired.
- Asymmetric partition: client believes it has the lock; the Redis nodes are partitioned from the resource; other nodes form a majority and grant a new lock.
- Restart without durable persistence: a node restarts, forgets the lock (AOF not yet fsynced), the lock can be re-acquired on (restarted node + others) = new majority.

The deeper structural problem: Redlock is not consensus. Five independent Redis instances voting on a SET command is not the same as five replicas running a state machine via Raft/Paxos. The independent instances don't agree on time, on membership, or on order of operations across the cluster. They each independently make local decisions based on their local clock, and Redlock layers a "majority" rule on top of those independent decisions. That doesn't compose into mutual exclusion.

This was the Martin Kleppmann critique in 2016, and the antirez response: Redlock is fine for efficiency but not for correctness. The distinction is decisive and almost never made explicitly in production code.

Reframe the question

The right first question isn't "how do we fix Redlock?" — it's what is the lock actually protecting?

- Efficiency lock: "we'd rather not do this job twice." If two workers occasionally both process the same item, you waste compute but nothing is corrupted, no money moves wrong, no data is destroyed. Redlock is acceptable.
- Correctness lock: "only one writer at a time may touch this resource." If two writers do it simultaneously, the resource is corrupted, the balance is wrong, the file has a partial write, the inventory goes negative. Redlock is not acceptable, under any tuning, because the failure modes above are realistic.

Most production "we use Redlock to coordinate" code is silently in the second bucket while the team thinks it's in the first. The clock-jump example in the question is correctness territory: two clients holding the lock means whatever they're protecting can race.

Design space

1. Use fencing tokens — the lock is advisory, the token is mandatory (the Kleppmann fix)

Don't trust the lock. Trust a monotonically increasing token enforced by the protected resource:

- Lock service issues a strictly monotonic token each time the lock is acquired (token 17, then 18, then 19, never going backward).
- Client passes the token with every write to the protected resource.
- The resource records the highest token it has ever seen and rejects writes with a token less than that.

Now the clock-jump scenario plays out differently:
- Client A holds the lock and got token 42.
- Clock jumps; client B "acquires" the lock and gets token 43.
- Client A writes to the resource with token 42 → rejected, because the resource has now seen 43.

The resource is the source of truth. The lock is just optimistic mutual exclusion; the fencing token is the actual safety mechanism. This requires:
- The lock service must issue monotonic tokens (Zookeeper sequential znodes, etcd revision numbers, Spanner timestamps, RDBMS sequences).
- The resource must check the token. Databases can do this with UPDATE … WHERE seen_token <= ?. Object stores often can't; queues often can't; arbitrary microservices often can't without explicit support.

The resource being able to enforce the token is what makes this safe. If the resource can't (legacy system, external API, file system), fencing doesn't help and you need a different design.

2. Use a consensus-backed lock service

If you genuinely need a distributed lock for correctness, use a system that solves consensus properly:

- etcd: leases with Raft consensus. Lease expiration is decided by the cluster, not by individual node clocks. Pair with a fencing token derived from the lease revision.
- Zookeeper: ephemeral sequential znodes. The sequence number is your fencing token; the ephemeral semantics give you "owner died" detection via session, not via local clock.
- Consul: also Raft-based.
- Spanner / CockroachDB / FoundationDB: use a real ACID transaction; the database serializes you.

These are slower than Redis. That's the price of safety. The fact that Redis is faster than etcd is not a justification for Redlock if the use case needs mutual exclusion — it's pricing apples against oranges.

Note the safety property comes from both consensus and fencing tokens — a consensus lock without a fencing-token check at the resource is still vulnerable to "client pauses, lease expires, another client acquires, original client wakes up and writes." Pause-after-lease-expiry is just as dangerous as clock-jump-after-TTL.

3. Avoid the lock entirely (usually the best answer)

Distributed locks are a last resort. Most "I need a distributed lock" requirements have a better answer:

- Idempotency keys: two workers process the same job, both write with the same key, the second is a no-op. No lock needed. Default for most queue-driven workflows.
- Compare-and-swap / optimistic concurrency: UPDATE row SET ... WHERE version = N — the database serializes for you. Retry on conflict.
- Single-writer-per-partition: route all work for key K to one consumer (Kafka consumer-per-partition is exactly this). There's only ever one writer for K by construction; no lock to acquire.
- Leader election with epoch numbers: elect a single leader per scope (Raft, Zookeeper, etcd). Each election bumps an epoch. The epoch is your fencing token; only writes with the current epoch are accepted.
- CRDTs / commutative operations: design so concurrent writes converge correctly. Order-independent operations don't need locks.

For most "I'm reaching for Redlock" problems, one of these is cheaper and safer.

4. If Redlock is the chosen tool, scope it correctly

For pure efficiency cases — avoiding redundant work that's harmless if done twice — Redlock works. To make this explicit and not regress over time:

- Document at the call site: "this lock is for efficiency only; double execution is safe."
- Make the protected operation idempotent anyway. Belt and suspenders.
- Don't ever take the lock as the sole protection for anything correctness-sensitive — there should be a second mechanism (idempotency key, CAS) that survives if the lock fails.

5. The clock-jump problem beyond Redlock

The broader rule: never use wall-clock time as a safety boundary across hosts. Anything that does is wrong, because:

- NTP can step the clock (jumping forward or backward by seconds-to-minutes).
- VM live migration / snapshot resume causes the guest clock to jump.
- Leap seconds have historically broken systems badly.
- Container freeze/thaw, hypervisor pauses, OS suspend can introduce arbitrary jumps.

Use instead:
- Monotonic clocks (CLOCK_MONOTONIC) for durations on one host only.
- Logical clocks / Lamport / vector clocks for ordering across hosts.
- Hybrid Logical Clocks (HLC) when you need wall-clock-ish ordering plus logical safety. CockroachDB uses these.
- TrueTime (Spanner): bound clock uncertainty with atomic clocks + GPS and wait out the uncertainty interval. Hardware-grade solution; most orgs don't have it.
- Leases checked by consensus: lease expiration is decided by a quorum vote, not by one node's clock.

Anti-patterns

- Using Redlock to protect a row, file, balance, or any resource that can be corrupted by concurrent writes. Correctness use case, wrong tool, will eventually break under the realistic failure modes above.
- Assuming clocks are "synchronized enough." Unbounded skew is the default; bounded skew requires hardware (TrueTime) or consensus (lease-by-quorum).
- Trusting the lock without fencing. Even with a perfect lock service, GC pauses and network delays mean a "lock holder" can be a stale process about to write. The resource must reject stale tokens.
- Increasing TTL to "fix" safety. Longer TTL = longer split-brain window after a real failure, lower availability after legitimate crashes, no safety improvement.
- Using Redis AOF persistence to make Redlock "safe." AOF lag, fsync semantics, partial sync — all of these still leave a window where a restart loses the lock state. AOF helps durability, not the safety property the question violates.
- Conflating "five Redis nodes" with "consensus." They are independent stores with a majority vote on top, not a replicated state machine. The difference is exactly what creates the safety violation.
- No idempotency at the protected operation. If the lock fails, you have no second line of defense. Always make the operation idempotent if you can; the lock is then optimization, not safety.

Operational concerns

- Monitor clock skew across the Redis cluster if you must use Redlock. NTP offset metrics, alert on jumps > 100ms. This detects the failure mode; it doesn't prevent it.
- Monitor process pauses on lock-holding clients: GC pause times, scheduler latency. A 30s GC pause invalidates any Redlock with a 30s TTL.
- Test the failure modes you depend on not happening: chaos-test by stepping the clock on one Redis node, by pausing the lock-holding client, by partitioning. If your system breaks, you didn't have what you thought.
- Audit existing Redlock uses: classify each as efficiency or correctness. The correctness ones are bugs waiting to fire.

What I'd actually do

1. For the specific incident: replace Redlock with etcd or Zookeeper for the lock + a monotonic fencing token enforced at the protected resource. Lock service issues monotonic token; resource rejects writes with stale tokens. The clock-jump scenario now ends in "Client A's write is rejected" instead of "both clients write."
2. Audit other Redlock uses in the system. Each one needs to be classified: efficiency (keep, document, add idempotency) or correctness (migrate to consensus + fencing token).
3. Default future "distributed coordination" requirements to non-lock primitives — idempotency, CAS, per-key single writer, leader election with epoch. Locks are the last resort.
4. Treat wall-clock time as never-safety. Code review should reject any new use of wall-clock time for safety boundaries across hosts. Use monotonic clocks for durations, logical clocks for ordering, consensus-managed leases for liveness.

Bottom line

The Redlock safety violation in the question isn't a bug in a specific deployment — it's a property of the algorithm. Redlock layers a majority vote over five independent stores whose only synchronization is the wall clock, and any deviation from synchronized wall clocks (clock jump, GC pause, network delay, asymmetric partition) breaks the mutual exclusion guarantee. For efficiency use cases — avoiding redundant work — Redlock is fine, and the standard practice is to combine it with idempotency at the operation level. For correctness use cases — anywhere two simultaneous writers would corrupt the resource — Redlock is the wrong tool, period, and the fix is consensus-backed lock service + monotonic fencing token enforced at the protected resource. The token is what carries the safety property, not the lock; the lock is just performance optimization. And the underlying lesson generalizes: never use wall-clock time as a safety boundary across hosts.

✻ Baked for 2m 14s

❯ Lock expiry during processing: Process acquires a lock with a 30-second TTL. Processing takes 35 seconds (due to GC, slow I/O, etc.). Lock expires at 30s. Another process acquires it.
Both processes now operate concurrently on the shared resource.

Lock Expiry During Processing

Root cause

This is the stop-pause vs fail-stop problem, and it's the deeper structural issue underneath the Redlock question — present even with a perfectly-synchronized clock and a consensus-backed lock service.

TTL/lease-based locks conflate two distinct conditions:

- "The client crashed" — process is gone, the resource will never be written, it's safe to grant the lock to someone else.
- "The client is paused or slow" — process is alive, work is still in progress, the resource will be written, but not for another N seconds.

From the lock service's perspective these look identical: silence. TTL expiry decides one way; the client decides the other; they disagree. The result is the two-writer scenario in the question.

Realistic causes of pause/overrun, in 2026 production environments:

- JVM stop-the-world GC — easily 5–30s on heap-pressured services, longer with bad tuning.
- CPU throttling under cgroup limits — Kubernetes pods can be throttled to effectively zero CPU for seconds.
- Hypervisor freeze during live VM migration, snapshot, or noisy-neighbor scheduling — tens of seconds is documented.
- Swap/memory pressure — a process touching swapped pages can block for seconds per page fault.
- Slow syscalls / blocking I/O — NFS hangs, slow DNS, network microbursts, EBS throttle, full disk pause.
- Debugger / strace / profiler attachment — yes, this has caused production incidents.
- Container freeze for live migration or quota enforcement.

The pause budget is unbounded in the general case. There is no safe value for "TTL > worst-case pause" because the worst case is "as long as the kernel decides." This is the foundational reason TTL alone cannot provide mutual exclusion.

Classical distributed-systems literature often assumes fail-stop: processes work correctly or crash, and the difference is observable. Real systems are fail-recoverable-after-arbitrary-pause, which is a strictly harder model. Algorithms designed for fail-stop are unsafe under stop-pause.

Reframe the question

The framing — "lock expires because processing took longer" — invites two wrong fixes:

1. "Make the TTL longer." Pushes the bug to a longer pause and reduces availability after a real crash (the resource is locked for 5 minutes instead of 30 seconds while everyone waits for a TTL on a process that's actually dead).
2. "Make processing faster." Helpful as engineering, but doesn't change the underlying property. The next 99th-percentile pause finds you.

The actual reframe: the lock cannot make this safe by itself. The lock service is making a decision under uncertainty — "is this client still working?" — and it will sometimes be wrong, in both directions. What makes the system safe is not the lock decision but what the resource accepts. The lock is liveness; the resource enforces safety. Same conclusion as the Redlock question, by a different path.

Design space

1. Fencing tokens enforced at the resource (the safety mechanism)

Identical to the previous answer:

- Lock service issues a strictly monotonic token at each acquisition.
- Client sends the token on every write.
- Resource records the highest token seen and rejects writes with lower tokens.

Trace the question's scenario with fencing:
- Process P acquires the lease, gets token 42, starts work.
- P pauses (GC, throttle, whatever) for 35s. Lease expires at 30s.
- Process Q acquires the lease, gets token 43, writes to the resource. Resource records seen = 43.
- P wakes up at t=35s, writes with token 42. Resource rejects (42 < 43).
- P discovers it has lost the lease, handles it as failure, retries from a known state.

Both processes thought they held the lock simultaneously, but only one wins where it matters. The lock decision being wrong becomes a non-event rather than a correctness violation.

This is the load-bearing pattern. Everything else in this answer is auxiliary.

2. Lease + heartbeat — and the bugs to avoid

Heartbeat-based leases (etcd lease, Zookeeper session, Consul session) are the right primitive for the lock side. But the heartbeat thread architecture matters:

- Heartbeat from a separate thread: the renewal continues even when the work thread is paused. This is the wrong pattern: the lease stays alive, another client can't acquire, but the work isn't progressing either — and worse, when the work thread wakes up, it has stale assumptions about the world. Combined with no fencing, this gives you "lock not held by anyone else, but stale data." Avoid.
- Heartbeat from inside the work loop: a GC pause stops the work and the heartbeat. From the lock service's view, the client has correctly stopped; the lease expires; another client acquires. The original client wakes up, observes its lease is gone, must abort. This is the correct architecture, but requires the work to be structured as a loop with explicit heartbeat checkpoints.
- Check the lease right before writing: TOCTOU race. The lease can expire between check and write. Doesn't help on its own.

The pattern that works:

loop:
check_and_renew_lease()       # fails → abort, do not write
do_one_chunk_of_work()
write_with_fencing_token()    # resource enforces token

Heartbeat-from-the-work-loop catches "I'm progressing"; fencing token catches "I lost the lease while writing"; chunked work bounds the worst-case lease-expiry-then-write window. All three together are robust.

3. Bound the work in the critical section

If you can chunk the work so each chunk is short (≤1/3 of the TTL), the pause-vs-TTL race becomes far less likely:

- Acquire lease at token N → do small chunk → write with token N → renew or release.
- Crash/pause at any point loses at most one chunk's worth of work.
- The protected resource sees a stream of small atomic writes, each fenced.

This is the "small transactions" pattern from database design applied to distributed locking. Long critical sections are dangerous; short ones are forgiving.

4. Make the protected operation idempotent

Even with fencing tokens and proper leases, idempotency is the third line of defense. If the work itself can be safely repeated (same input → same outcome, no double side-effects), then "two clients did the work" becomes "wasted compute" instead of "corrupted state." Most queue-driven workflows already need this for other reasons (retries, redelivery).

Combine: lease for liveness + fencing token for safety + idempotency for sanity. All three together is the durable pattern.

5. When the resource can't enforce tokens

Fencing requires the resource to participate. It works cleanly for:
- Databases (CAS or conditional update on a token column).
- Object stores with conditional writes (S3 ETag/If-Match, GCS generation match).
- Your own services (you can add the check).
- Queues that accept idempotency keys.

It doesn't work for:
- Most third-party APIs (payment processors, SaaS endpoints).
- Side-effect operations (sending email, SMS, push notifications, control of physical hardware).
- Legacy systems you can't modify.

For these, TTL-based locks are not safe, full stop. The mitigations are different:
- Single-writer-by-routing: route all work for key K to one consumer always (Kafka per-partition pattern). No concurrent access by construction.
- Idempotency at the receiver: deduplicate at the external system if it supports it (Stripe idempotency keys, mail-merge UUIDs).
- Accept that the operation might fire twice, and design the surrounding system to tolerate it.
- Wrap external operations in an "intent + commit" pattern: write the intent transactionally; a separate single-writer process executes the intent against the external system; the intent is the safety boundary.

You cannot wave hands and call this "safe with a 30-second TTL." If the resource can't reject stale writers, the lock is at best best-effort.

Anti-patterns

- TTL-only lock with no fencing token. The scenario in the question, full stop. Treat any such code as a latent correctness bug.
- "We checked the JVM and our GC pauses are under 1 second." Today, at this heap size, on this hardware, under this load. Tomorrow's heap pressure, container throttling, hypervisor freeze, or syscall hang doesn't care about today's measurement.
- Heartbeat thread independent of the work thread. Heartbeat keeps the lease alive while the work is stalled — strictly worse than no heartbeat, because now the lock service thinks the client is healthy.
- Increasing TTL to "safely" cover the worst case. There is no safe worst case. Longer TTL trades availability after legitimate crashes for marginal protection against pauses.
- Treating "TTL > P99 processing time" as sufficient. P99 is "99 out of 100 executions." The 1% are exactly when this fires, and they correlate with load — i.e., production traffic peaks.
- Checking the lock state on the client right before writing. TOCTOU. The lease can expire between check and write. The check must happen at the resource, with the token.
- No idempotency at the operation. When (not if) the fencing or lease fails, idempotency is the third line of defense. Removing it makes a recoverable bug into a corrupted-state incident.

Operational rigor

- Monitor pause-causing metrics: JVM GC P99 and max pause times, scheduler latency (cpu_throttled_seconds_total in cgroups), syscall tail latency, swap-in events. These are the leading indicators for "lease expiry during processing" incidents.
- Bound the lease-expiry budget per workload: track (processing_duration_p999, lease_ttl) per critical job. If P999 is creeping toward TTL, the architecture is degrading; alert before the failure fires.
- Fence-rejection metrics: count writes rejected at the resource for stale tokens. Non-zero is a signal that lease/work overruns are happening; the system handled it but you should investigate.
- Chaos test the overrun: inject a forced pause longer than the TTL into the work loop in staging. Verify the resource rejects the post-pause write and the client handles the rejection correctly. If you've never tested the pause path, you don't know if your fencing actually works.

What I'd actually do

1. Fencing tokens at the resource: the lock service issues monotonic tokens; the resource accepts only the highest. This is the actual safety property. If the resource can't enforce it, you can't safely use TTL-based locks for it.
2. Lease + heartbeat from the work loop (not a separate thread), backed by a consensus-based lock service (etcd, Zookeeper). The pause stops both the work and the heartbeat; the lease expires correctly; the client aborts on wake-up.
3. Chunk the work: short critical sections, each atomic and individually fenced. Long-running work is structured as a sequence of fenced steps, not one long lock-holder.
4. Idempotency on the protected operation as the third line of defense. Two writes producing the same outcome means the bug is a wasted-compute incident, not a corrupted-state incident.
5. Monitor and chaos-test the overrun path: inject pauses, verify rejection at the resource, track real-world processing-time distributions against lease TTLs.
6. For resources that can't enforce fencing (third-party APIs, side effects), don't pretend a TTL lock makes this safe. Use single-writer-by-routing, external idempotency keys, or intent-commit patterns.

Bottom line

A TTL-based lock cannot provide mutual exclusion under realistic process pauses, because the lock service can't distinguish "client crashed" from "client paused." The fundamental fix is the same as for Redlock: the lock is liveness, the fencing token is safety, the resource is the source of truth. Pair a consensus-backed lease (with the heartbeat inside the work loop, so a pause kills both) with a monotonic fencing token enforced at the resource and idempotent operation semantics. That combination is robust to GC pauses, CPU throttling, hypervisor freezes, network partitions, and all the other ways the stop-pause model violates the fail-stop assumption that TTL-based locks were designed for. For resources that can't enforce fencing — third-party APIs, irreversible side effects, hardware — TTL locks are best-effort at best, and the architecture has to provide safety some other way (single-writer routing, external idempotency, intent-commit).

✻ Crunched for 2m 9s

※ recap: You're working through staff-level answers to a series of distributed systems failure-mode questions; we just finished the lock-expiry-during-processing one. Next: send the next question whenever you're ready. (disable recaps in /config)

❯ Token replay across regions: A JWT is issued in region US-EAST. Region EU-WEST has a 5-minute token revocation propagation delay. A compromised token is revoked in US-EAST but still
accepted in EU-WEST for 5 minutes.

─────────────────────────────────────────────────────────────────────────────────────────────────── 1 new message ────────────────────────────────────────────────────────────────────────────────────────────────────

Token Replay Across Regions

Root cause

JWTs are stateless by design: the token carries its own claims, signed by the issuer, and any verifier with the public key can validate it locally without contacting the issuer. That's the whole reason JWTs scale — no auth-server round-trip on every request.

The price of statelessness is that validity is what the token says it is. The exp claim is set at issuance and can't be retroactively shortened. To make a token stop being valid before its exp, you need state somewhere — a denylist, a session record, a revocation list — and that state must propagate to every verifier before any further request from the compromised token can succeed.

The 5-minute propagation delay isn't a bug, it's the deliberate (often implicit) trade-off that was made when the architecture was set up:

- Cross-region authoritative checks on every request = high latency, low availability under partition.
- Local validation against locally-replicated state = fast, available, but stale during the replication lag.

The team picked door #2 and the lag is the cost. The question is whether the cost matches the threat model.

This generalizes: compromise window = max(access token lifetime, revocation propagation lag, anomaly-detection lag). Whichever of those is largest is what an attacker gets after the compromise. Most production incidents in this space are about teams who never made this number explicit, and only discovered it during the incident.

Reframe the question

"5-minute propagation delay" is not really the bug — it's a symptom of conflating long-lived authentication state with short-lived per-request authorization. The standard architectural separation:

- Long-lived state (the session, the user being authenticated): stateful, centrally revocable, immediate effect when revoked.
- Short-lived stateless token (the access token used per request): validated locally, intentionally bounded in lifetime so that revocation of the underlying session takes effect within the bound.

If you make this separation explicit, the "5-minute revocation propagation" problem becomes "5-minute access-token lifetime" — which is a design choice, not a replication property. The system regains determinism: the worst-case compromise window is now the TTL of the access token, set by you, knowable without measuring replication lag.

So the staff-level reframe is: don't try to make revocation propagate faster; shorten the lifetime of what's being revoked.

Design space

1. Short-lived access tokens + stateful refresh tokens (the standard baseline; usually the right default)

The OAuth2 pattern, applied honestly:

- Access token: JWT, 5–15 minutes, validated statelessly at every API call.
- Refresh token: opaque, long-lived, stored in a database. Used only to mint new access tokens at the auth server.

Revocation flow:
- Admin revokes the user's session. Refresh token is deleted from the DB. Immediately effective everywhere because the DB is the only place refresh tokens live.
- Existing access tokens continue to work until their exp (worst case = access token lifetime).
- When the access token expires, the client tries to refresh, refresh fails, user is logged out.

Compromise window = access token TTL. For sensitive systems, 5 min is typical. For low-stakes, 15–60 min. The "5-minute propagation lag" in the question is what you'd get for free with a 5-minute access token TTL — without any cross-region replication of denylists at all.

This is the highest-leverage change. The question is essentially asking how to fix a bad case of long-lived JWTs + complex denylist propagation when a clean short-TTL design eliminates the problem.

2. Stateful (opaque) tokens for high-value flows

For flows where even a 5-minute window is too long — money movement, admin actions, healthcare data — drop JWTs entirely. Use opaque tokens that hit an authoritative store on every request:

- Token is a random string with no embedded claims.
- Every request reaches a session store (Redis, DynamoDB, the auth service) to look up "is this still valid?"
- Revocation is a single write; the next request anywhere sees it.

Trade-off: a round trip per request, and the session store becomes a critical-path dependency. Mitigate with:
- Regional session stores with strong consistency (DynamoDB Global Tables with strong consistency; Spanner; CockroachDB).
- Read-through caches with push invalidation on revoke, not TTL — TTL-based caches reintroduce the propagation lag.

The point is to push revocation latency from "replication SLA" down to "single write + invalidation propagation," which can be sub-second.

3. Event-driven denylist with bounded staleness

If you must keep the JWT model but want sub-second revocation, build a proper revocation pipeline:

- Each token has a jti claim (unique token ID).
- Revocation writes to a "revoked tokens" topic/stream (Kafka, NATS, Pub/Sub, SNS+SQS).
- Each region's verifiers subscribe and maintain an in-memory bloom filter of revoked jtis. False positives trigger an authoritative check.
- The bloom filter only needs to hold tokens that haven't yet expired (sliding window), so memory stays bounded.

Tuning levers:
- Replace periodic poll with push to get propagation under 1 second.
- Track end-to-end propagation latency (revoke initiated → present in EU-WEST bloom filter) as an SLO; alert when it exceeds threshold.
- A high-watermark sequence number lets verifiers detect lag ("my last revocation event is from 30 seconds ago; this token might be revoked and I don't know yet") and fail closed for high-value endpoints.

This is a more complex implementation than #1 and pays off only when access tokens need to be longer than 15 minutes and revocation needs to be near-immediate. Usually #1 is the better trade.

4. Risk-tiered validation

Not all API surface is equally sensitive. Tier the endpoints:

- Low stakes (read public data, list-my-stuff): cached/stateless validation. Accept the propagation lag.
- Medium stakes (read user-private data): validate against a region-local store with bounded staleness.
- High stakes (writes, money, admin, MFA changes): require authoritative validation, even at higher latency. Possibly cross-region if necessary; possibly step-up auth (re-MFA).

The compromise window for GET /me can be the full 5 minutes if you want; for POST /transfers it should be zero. Same token, different validation requirements at different endpoints.

5. Token binding — proof of possession

A different axis: even if the token is stolen, can we make replay difficult?

- DPoP (Demonstrating Proof of Possession): client signs each request with a key bound to the token. The token alone isn't usable; you need the key. RFC 9449, supported by major auth platforms.
- mTLS / client certificates: client must present a cert in addition to the token. Stolen token + no cert = useless.
- Token-to-device binding: ties the token to a device-specific key (TPM, secure enclave). Mobile platforms make this easy.

These don't fix revocation propagation; they reduce the cases where revocation has to fire by making stolen tokens harder to replay. Often the right defense in depth for high-value tokens.

6. Behavioral / anomaly-driven revocation

Detect compromise from usage:
- Token used from a new geography seconds after legitimate use in the original (token replay across continents — your scenario's literal example).
- Token used at unusual hours, unusual rate, unusual operation mix.
- Token used after the underlying user changed password.

When detected: revoke proactively, force re-authentication. This doesn't fix the propagation problem but shrinks the detection-to-revocation time and reduces reliance on the user reporting the compromise. Pairs with #3 (fast event-driven revocation) so detection actually translates to fast mitigation.

7. Refresh-token rotation

Tangential but related: refresh tokens themselves should rotate on every use. Each refresh request returns a new refresh token; the old one is invalidated immediately. If an attacker steals a refresh token and the legitimate user refreshes once, the stolen token is dead. This is now standard in well-designed OAuth implementations (Auth0, Okta, modern OAuth2.1 draft).

Anti-patterns

- Long-lived JWTs (hours/days) with no denylist. Compromise window = full exp. There is no way to revoke. This is the original sin that creates the question's scenario.
- Denylist that grows unbounded. Without sliding-window pruning (entries older than max token TTL can be discarded), the list becomes too big to consult quickly or hold in memory.
- No jti claim. You can't revoke individual tokens; only entire keys or users. Coarse-grained, expensive.
- TTL-based denylist caches. Reintroduces propagation lag by definition. Use push-based invalidation on revocation.
- Storing revocation only in one region's database. When that region is partitioned, you either accept everything (insecure) or reject everything (outage). Replicate, or use a globally-consistent store for revocation.
- Treating "exp" as a security property you can shorten. You can't. The signed exp is what it is; only the new tokens you issue can have shorter exp. Existing tokens remain valid until their stated exp unless denylisted.
- Refresh tokens that never rotate. A stolen refresh token compromises the account for its entire lifetime — often months.
- Same compromise window for GET /public and POST /transfer. Wastes security budget on the wrong place. Tier the validation requirement to the action.
- Telling users "you're logged out" but actually leaving the access token valid. Some apps drop the refresh token client-side but the access token continues to work from another client until its exp. Bug if you thought revocation was instant.

Operational concerns

- Revocation-latency SLO: explicit target for "revocation event in US-EAST → effective in EU-WEST." Page when exceeded. If you don't have this number, you can't reason about your actual compromise window.
- End-to-end revocation propagation metric: revoke at T₀, observe rejection from the furthest region at T₁. Track P99.
- Audit logs on every token use: when a compromise happens, you need to know what the attacker did. Per-jti access log, queryable by token ID.
- Mass-revocation testing: a password reset can revoke thousands of refresh tokens at once. Verify the revocation pipeline handles bursts.
- Key rotation as nuclear option: if the signing key is compromised or mass revocation is needed, rotate the signing key. All tokens signed with the old key invalidate at once. Coarse-grained but immediate. Have this runbook ready.
- Game-day: induce a "compromise" and measure your actual revocation latency end-to-end. The first time you measure this should not be during an incident.

What I'd actually do

1. Shorten access tokens to 5–15 minutes. Refresh tokens stay long-lived but stateful, stored in a DB, revoked instantly there. The "revocation propagation" problem mostly evaporates because it's bounded by the access-token TTL by design.
2. Refresh-token rotation on every use. Stolen refresh tokens get exactly one use before invalidation.
3. Tier validation by sensitivity: high-value endpoints require authoritative validation against the session store (accept the latency cost where it matters); low-value endpoints validate statelessly.
4. For sub-15-minute revocation needs, build event-driven denylist propagation with a jti in every token, bloom filter at verifiers, sliding-window pruning, and a measured propagation SLO.
5. DPoP or mTLS for high-value tokens so a stolen access token isn't independently replayable.
6. Behavioral anomaly detection that proactively triggers revocation on suspicious usage (impossible-travel, rate spikes, new device + sensitive action). Pairs with #4 so detection translates to fast revocation.
7. Treat key rotation as the nuclear option with a tested runbook, and instrument revocation latency end-to-end with an SLO and a game-day.

Bottom line

The 5-minute propagation lag isn't really the bug — it's the deliberate cost of choosing locally-validated, replicated revocation state. The staff-level fix is architectural separation between long-lived authentication state (stateful, instantly revocable) and short-lived authorization tokens (stateless, lifetime-bounded). Once access tokens are 5–15 minutes, the worst-case compromise window equals the access-token TTL, set by you, and you don't need to fight cross-region replication for revocation propagation at all. For systems that need sub-minute revocation under longer-lived tokens, build proper event-driven denylist propagation with jti and bloom filters, tier validation by sensitivity, and add proof-of-possession (DPoP, mTLS) so stolen tokens are harder to replay in the first place. Make the compromise-window number explicit, measured against an SLO, matched to the threat model — and design the auth architecture to deliver it rather than discovering it during an incident.


```
Certificate rotation race: During mTLS cert rotation, node A gets the new cert but node B still has the old CA bundle. B rejects A's connections. This lasts until B gets the update —
during which inter-service communication is partially broken.
```


mTLS Certificate Rotation Race

Root cause

Mutual TLS has a fundamental coordination property: trust is bilateral, but updates are unilateral. Each end of every connection has four things that must agree with its peer at handshake time:

- Its own certificate (presented to the peer).
- Its own private key (proof of possession).
- Its trust bundle / CA store (used to verify the peer's cert).
- Optionally: pinning, SAN allow-lists, or chain-validation policy.

Each of these is updated independently, by independent mechanisms, on each node in the fleet. There is no atomic "rotate" across nodes. So every rotation has a window where some nodes have new state and others have old state, and during that window, every pair of nodes is in one of four states — old↔old, new↔new, old↔new, new↔old — and any cross-version pair can fail if the trust chains don't overlap.

The specific failure in the question — node B doesn't trust node A's new cert — is the most common manifestation: the credential rolled out before the trust did. That's a logical error in the rotation ordering, not a fundamental impossibility. There's a correct order that avoids it; the question describes what happens when it's not followed.

A related family of failures all share this shape:

- New cert presented before peers trust its CA (the question).
- Old cert removed from trust before all peers have updated their incoming certs.
- Pinned cert/key removed while clients still pin the old value.
- Long-lived TLS connections continuing on the old cert past its expiry.
- CA bundle propagation lag during cert renewal, causing intermittent rejection.

All of them are coordination bugs in the rotation procedure, not protocol problems with mTLS.

Reframe the question

The framing — "this lasts until B gets the update" — invites the question "how do we make B update faster?" The staff-level reframe: make B already trust the new cert before A presents it. That's not "faster propagation"; that's a different sequencing of operations:

▎ Trust propagates before credentials. Credentials propagate before trust is removed.

This is the same invariant as the DNS TTL question (lower the TTL before changing the record) and the FK-across-services question (parent must exist in the destination before the child can reference it): a downstream dependency must be established before the upstream commits to using it. Once you, the "race" stops being a race.

The deeper move: make rotation continuous and frequent rather than periodic and rare, so thtantly and every node assumes rotation is happening in the background. A rotation event isan opportunity for a bug; rotation as a steady-state property doesn't have rotation events.

Design space

1. Trust overlap, with the right phase ordering (the procedural fix)

For any CA or trust change, the rotation must be done in four phases — never skipping or compressing them:

Phase 1 — Distribute new trust. Add CA_new to every node's trust bundle while keeping CA_old. Now every node trusts both. No new certs issued yet; existing CA_old certs still work. Verify by inspecting every
node's effective trust bundle; gate the next phase on full propagation.

Phase 2 — Issue new certs. Roll out CA_new-signed certs to nodes. Any node receiving a new ecause all peers trust CA_new. Old certs continue to work because all peers also still trust CA_old.

Phase 3 — Wait for full cert turnover. Either let old certs expire naturally or force-rotate. End of phase 3: no CA_old cert in active use anywhere.

Phase 4 — Remove old trust. Strip CA_old from every node's trust bundle. Any straggler with an old cert is now rejected; that's the desired terminal state.

The invariants:
- Phase 1 fully completes before Phase 2 starts. This is the property the question's incide
- Phase 3 fully completes before Phase 4 starts. Symmetric mistake on the back end of rotation.

In practice, Phase 1 → Phase 4 takes days to weeks. Compressing it because "we need to rotate fast for a compromise" is a leading cause of outages. If you need fast rotation, use the next pattern.

2. Short-lived certs + continuous rotation (the architectural fix)

The modern pattern (SPIFFE/SPIRE, Istio Citadel/Istiod, cert-manager with short-TTL issuers, AWS Private CA, HashiCorp Vault PKI):

- Workload certs valid for hours, not years (typically 1h–24h).
- Workloads automatically renew at ~1/2 lifetime, with retry and jitter.
- The CA bundle is longer-lived but also rotated with the four-phase procedure above.
- Identity attestation (SPIFFE ID, Kubernetes ServiceAccount + projected token, instance mewithout manual config.

Properties this gives you:
- Rotation is happening constantly; there is no "rotation event" to coordinate.
- Compromise window is bounded by the cert TTL (same property as short-lived JWT access tok
- "Cert expired" stops being an incident category because expiry is a continuous, automated, monitored process.
- A misconfigured node fails immediately on its next rotation cycle, not weeks later when asurface fast and small.

Costs: requires a real identity infrastructure (SPIFFE/SPIRE, mesh, cert-manager) and worklgracefully. The investment pays back fast in any fleet large enough that manual rotation isa real-world problem.

3. Service mesh sidecars (remove TLS from application code)

Push mTLS termination into a sidecar (Envoy, Linkerd-proxy):
- App speaks plaintext over loopback to the sidecar.
- Sidecar handles cert presentation, CA bundle, mTLS handshake, rotation.
- The mesh control plane (Istio, Linkerd, Consul) atomically pushes cert/CA updates to all ond propagation, ordered, monitored.

The mesh has already solved the trust-overlap coordination problem for you; the applicationts. This is the highest-leverage option for any fleet large enough to have rotationincidents.

4. Hot reload without connection drops

Even with perfect rotation procedure, a node updating its own cert shouldn't terminate in-flight connections. Properly implemented:

- New TLS contexts are loaded alongside old ones; new connections use the new cert.
- Existing connections continue with the old cert until they close.
- Background drain closes old connections gradually (max_connection_age, server-initiated GOAWAY on HTTP/2).
- The old TLS context is released when the last connection using it closes.

Nginx, HAProxy, Envoy, gRPC servers, modern application servers all support this. JVM SSLCofiguration reload. Without this, every rotation is a brief connection-drop event.

5. Push-based trust distribution, not pull

Trust bundles are not the place to use periodic polling:
- Pull (polling): nodes check /etc/ssl/certs or a secret store every N minutes. Worst-case propagation = N. The question's "5-minute lag" is the JWT-revocation analog of this.
- Push (control plane): mesh xDS, gossip, message bus. Propagation in sub-second; verifiablent.

For trust bundles specifically, push is much safer because the consequence of stale trust ia degradation, an outage.

6. Pin the CA, not the cert

Cert pinning has its place (defending against issuer compromise), but pinning individual ceery rotation a coordinated deploy of every client. Pin the CA, or pin multiple certs(current + backup) to allow rotation without code change. HPKP-style cert pinning in browser contexts has been deprecated for exactly this reason.

7. Stagger rotation, monitor as you go

Even with continuous rotation, two operational guardrails:
- Don't rotate every cert in the fleet simultaneously. Stagger so a bad rotation affects a ewal jitter, deployment slices, canary nodes.
- Connection-success-rate is the indicator. TLS handshake failure rate per source→destination pair, broken down by error code (cert expired, cert untrusted, unknown CA, hostname mismatch). A spike during rotation
  is a signal to halt and investigate.

Anti-patterns

- Rotating a CA without trust overlap. The exact scenario in the question; the procedural m Always Phase-1 first.
- Removing old CA from trust while old certs may still exist. Symmetric mistake at the back end of rotation; same outage shape from the other direction.
- Pinning specific certs in application code. Couples every rotation to a code deploy. Pin ning.
- Manual annual rotation. "We have to remember to do this every year." It will be forgotten until the expiry-induced outage. Make it continuous and automated.
- No hot reload. Cert rotation requires process restart, which drops in-flight connections. severe in large fleets where rotation is ongoing.
- Configuration drift between nodes' CA bundles. Some nodes silently trust things others don't. Detect with bundle-hash audits across the fleet.
- Uniform expiry across the fleet. All certs issued the same day, expiring the same day, rorenewal pipeline hiccups, everything fails together. Jitter the issuance times.
- No expiry alerting. Cert expiry incidents are 100% preventable with multi-horizon alerts (30d, 7d, 24h, 1h). If you're getting paged because a cert expired, your process is the bug.
- Treating mTLS as a deploy concern rather than an identity concern. Ad-hoc rotation via Andentity service — the latter scales, the former eventually breaks.
- No rollback path. If new cert/CA rollout breaks something, you should be able to revert. Keep old CA trusted until rotation is verified clean.

Operational rigor

- Multi-horizon expiry alerts: 30d, 7d, 24h, 1h. Page-worthy. Cert expiry should never be the cause of an incident.
- Per-node trust-bundle audit: hash every node's effective CA bundle; alert on drift from e't propagate to this node" before the credential change.
- Handshake-failure rate broken down by direction and error code, alerting at >baseline. The fastest indicator of a misordered rotation in flight.
- Rotation rehearsal in staging: actually do the four-phase rotation procedure in a stagingduction routing. Find the bugs there, not in prod.
- Documented runbook for the four phases with explicit gating criteria — never compress phases for speed.
- Rotation halt mechanism: a kill switch to pause cert issuance / propagation in flight whe
- Connection-age limits so long-lived connections don't outlive their underlying certs. max_connection_age (HAProxy / Envoy / mesh sidecars), HTTP/2 GOAWAY-driven cycling.

What I'd actually do

1. Adopt SPIFFE/SPIRE-style identity with short-lived certs (1–24h) automatically rotated by the platform. Continuous rotation eliminates the discrete-event class of bug.
2. Service mesh sidecar (Istio, Linkerd, Consul Connect) for any nontrivial microservice flation concerns; control plane handles ordered trust + cert updates atomically.
3. Trust-overlap discipline for CA rotation, with the four-phase procedure documented, gated by automated verification of each phase, and rehearsed in staging.
4. Hot reload + connection-age limits everywhere so rotation is invisible to in-flight traf
5. Push-based trust bundle distribution (xDS, control plane, GitOps with reload triggers) — not periodic filesystem polling.
6. Trust-bundle drift audit + handshake-failure SLO as the operational backstop. Treat any s a paging incident — it's almost always a rotation race in progress.
7. Multi-horizon expiry alerts so a missed renewal is a warning days in advance, never a 3am outage.

Bottom line

The mTLS rotation race is a coordination problem, not a protocol problem. It happens whenever new credentials are deployed before the trust those credentials depend on has propagated — and the symmetric mistakeat the back end of rotation, removing old trust before all old credentials are retired. Theedural one (four-phase rotation with trust overlap: new trust before new credentials, oldcredentials retired before old trust is removed); and the architectural one (short-lived certs with continuous automated rotation via SPIFFE/SPIRE or a service mesh, hot reload, push-based control plane). With those in place, rotation stops being an event you coordinate and becomes a steady-state proen, every rotation is a coordination dance whose every step has to be done in the rightorder, and the question's incident is what happens when one step is skipped.