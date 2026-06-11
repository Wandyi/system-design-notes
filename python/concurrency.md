staff level software engineer questions on effectively handling concurrency in python. give different realistic sceanrios and case studies, include the utility of libraries like asyncio

Same structure as the Go answer — scenario, question, strong answer, the trap. Python's concurrency story has more model-selection up front (GIL, threads, processes, asyncio) and more subtlety once you're inside asyncio, so I'll lean into both.

---
1. Choosing the model

Tripstack's flight aggregator does 50 req/sec on Flask + a thread pool. Each request fans out to 10 GDS providers, each provider call is ~200ms of network. Goal is 500 req/sec.

Question: How do you scale, and why?

Strong answer covers:
- Math first. 500 req/sec × 10 calls × 0.2s = 1,000 concurrent outbound calls average. Threads work for I/O (the GIL releases on syscalls), but each thread is ~8 MB of stack plus kernel scheduling. 1,000 threads is feasible but wasteful; 5,000 is painful.
- Switch to asyncio + aiohttp. The event loop multiplexes thousands of in-flight sockets on one OS thread. Memory per "concurrent unit" drops from MBs to KBs.
- If response post-processing is CPU-heavy (XML parsing, fare construction, decimal arithmetic), that work blocks the event loop. Push it to ProcessPoolExecutor, or split it into a separate worker tier altogether.
- Connection reuse matters more than the framework choice — one shared ClientSession per process, with TCPConnector(limit=…, limit_per_host=…) tuned to your providers.
- Ordering of wins: async I/O (≈10×), shared session + keep-alive (≈2×), process pool only if CPU profile demands it.

Trap: "Just use multiprocessing." Multiprocessing copies memory (fork) or pickles + spawns (spawn) — far too heavy for I/O-bound concurrency. Right tool for CPU parallelism, wrong tool here.

---
2. The hidden blocking call

Async service running ~200 req/sec normal. Suddenly P99 jumps to 30 seconds. CPU is at 20%. Workers aren't restarting.

Question: Walk me through diagnosis.

Strong answer covers:
- Low CPU + high tail latency in async services almost always = event loop stalled on a blocking call. The event loop runs on one OS thread; whatever sits between awaits holds it.
- Turn on asyncio debug mode: PYTHONASYNCIODEBUG=1 (or loop.set_debug(True)). Logs "slow callback" warnings for tasks running >100ms. Adjust threshold via loop.slow_callback_duration.
- Run py-spy dump --pid <pid> against the live process to see exactly where the loop is parked.
- The usual suspects:
    - requests.get(...) inside a coroutine instead of aiohttp / httpx.AsyncClient.
    - Synchronous DB driver (psycopg2, mysqlclient) inside async code — every query blocks.
    - JSON parsing on multi-MB payloads; default json.loads is C but still blocking.
    - Logging with a sync file handler under heavy I/O contention.
    - re.match(...) with catastrophic backtracking.
    - time.sleep instead of await asyncio.sleep.
- Fix per case: await asyncio.to_thread(fn, ...) to push to a worker thread, or replace with a true async alternative (asyncpg, aiomysql, aiologger).

Trap: "Add more workers." Each worker has its own loop with the same blocking call. Symptom moves, root cause stays.

---
3. await is a yield point — race conditions in single-threaded asyncio

async def book_seat(seat_id, user):
seat = await db.fetch_seat(seat_id)
if seat.held_by is None:
await db.set_held(seat_id, user)
return True
return False

Question: Is this safe?

Strong answer covers:
- No. Single-threaded ≠ single-task. At every await, the event loop can run other tasks. Between fetch_seat and set_held, another booking can race in.
- Fix paths:
  a. Do it in one DB statement. UPDATE seats SET held_by=$1 WHERE id=$2 AND held_by IS NULL and check rowcount. Atomic, durable, no async lock needed.
  b. asyncio.Lock per resource. OK for in-process correctness; useless across multiple processes. Use a dict of locks keyed by seat_id, with eviction.
  c. Optimistic concurrency. Version column + retry on conflict.
- Async lock gotcha: a task holding a lock can be cancelled. async with lock: releases on cancellation; ad-hoc acquire/release with intermediate awaits does not.

Trap: "Python has the GIL, so I don't have races." The GIL is about bytecode atomicity for threads. Within asyncio, every await is an explicit yield point — that's where the races live.

---
4. gather vs TaskGroup — cancellation semantics

You launch 10 outbound flight searches. One raises. You want all others cancelled, no leaks.

# Pre-3.11 — three flavors, three different semantics
results = await asyncio.gather(*[search(p) for p in providers])
# If one raises: that exception is raised at the gather call.
# Siblings KEEP RUNNING in the background. They are not cancelled.

results = await asyncio.gather(*[...], return_exceptions=True)
# Returns a list of results-or-exceptions. You must inspect each.

# 3.11+ — structured concurrency
async with asyncio.TaskGroup() as tg:
tasks = [tg.create_task(search(p)) for p in providers]
# If any task raises, ALL siblings are cancelled, the block waits for them,
# and you get an ExceptionGroup at the `async with` exit.

Question: Walk me through.

Strong answer covers:
- gather doesn't implement structured concurrency. Siblings leak on failure unless you cancel them manually.
- TaskGroup is the modern default in 3.11+. Cancellation, waiting, and exception propagation are baked in. Errors come back as ExceptionGroup; pattern-match with except*.
- For "first-success wins" (return on first 200 OK, cancel the rest): asyncio.wait(tasks, return_when=FIRST_COMPLETED) then cancel pending and await them. Or as_completed + break + explicit cleanup.
- For "best-effort, take what's done by deadline T": asyncio.wait(tasks, timeout=T). Pending ones are returned to you so you can cancel + collect partial results.

Trap: Writing asyncio.gather(*coros) and assuming exception = all stopped. It isn't. Pre-3.11 codebases should pair gather with manual cancellation, or upgrade.

---
5. Connection pool exhaustion

aiohttp service runs 500 concurrent requests, each making 5 outbound provider calls. Default aiohttp.TCPConnector(limit=100). Symptom: random 30-second hangs.

Question: What's happening?

Strong answer covers:
- 500 × 5 = 2,500 concurrent outbound attempts contending for a 100-slot connector. Beyond the limit, calls queue on the connector's semaphore. Without acquire_timeout, they wait until the per-request timeout kills them.
- Fixes:
    - Raise the connector limit, but file descriptors and provider rate limits become the new ceiling.
    - Bound concurrent fan-out with asyncio.Semaphore(N) — orthogonal to the connector limit. Pick N from upstream capacity, not from connector size.
    - limit_per_host=10 to be a good citizen; one greedy host can't starve others.
- Share one ClientSession per process. Per-request session is the single biggest anti-pattern in aiohttp code — no keep-alive, no DNS cache, full TCP/TLS handshake every call.
- For multi-tenant cases (each request maps to a different upstream auth), use one session with per-call headers; don't make a session per tenant.

# Lifespan-managed shared session
@asynccontextmanager
async def lifespan(app):
app.state.session = aiohttp.ClientSession(
connector=aiohttp.TCPConnector(limit=500, limit_per_host=20),
timeout=aiohttp.ClientTimeout(total=5),
)
yield
await app.state.session.close()

Trap: async with aiohttp.ClientSession() as s: inside every handler. Looks clean, kills throughput, and breaks at scale.

---
6. CPU-bound work in an async handler

POST /itinerary-pdf does 800ms of pure-Python PDF generation. The endpoint is async. Throughput collapses to ~1 req/sec.

Question: Why, and what are the options?

Strong answer covers:
- The 800ms holds the event loop. All other requests wait.
- await asyncio.to_thread(make_pdf, data) moves the work off the loop thread — the loop stays responsive — but the GIL serializes pure-Python work across threads. If make_pdf is 100% Python, throughput is bounded at one PDF at a time.
- loop.run_in_executor(ProcessPoolExecutor(), make_pdf, data) actually parallelizes; cost is pickling args + return + IPC.
- C extensions that release the GIL (Pillow, lxml, NumPy, most cryptography) parallelize on threads. If your PDF lib is C-backed, to_thread is fine.
- Production answer: split it. PDF generation is asynchronous from the user's perspective anyway — push the job to a queue (arq, Celery, RQ), return 202 + job ID, deliver via webhook or polling. The async front door stays responsive; the worker pool autoscales independently.

Trap: Using to_thread for pure-Python CPU work and expecting parallel execution. GIL still serializes. Profile to confirm whether your "CPU work" actually releases the GIL.

---
7. Backpressure in an async pipeline

Streaming pipeline: Kafka consumer → parse → enrich (slow external API) → ClickHouse write. Consumer can do 50k/sec; enrich does 5k/sec. Memory grows steadily.

Question: Design.

Strong answer covers:
- Bounded asyncio.Queue(maxsize=N) between each stage. await queue.put(x) blocks when full → backpressure propagates upstream → ultimately, the consumer stops fetching from Kafka and the broker becomes the buffer (durable, by design).
- For the enrich stage: M worker tasks pulling from the input queue concurrently, bounded by a Semaphore around the outbound API call (separate concept — M controls how many things are simultaneously being enriched; the semaphore controls upstream API rate).
- Drop-newest vs drop-oldest is a policy decision. Use put_nowait with QueueFull handling for drop; for oldest-drop you need a custom structure (asyncio.Queue doesn't support it directly — use collections.deque(maxlen=N) + an asyncio.Event for signaling).
- Observability: gauge queue depth per stage; counter on drops; histogram on per-item latency end-to-end. Without these you can't see where the system is stuck.

Trap: asyncio.Queue() with no maxsize. Looks fine in dev. In prod, the queue is your leak.

---
8. Cancellation handling

A handler is mid-DB transaction. The client disconnects; the framework cancels the task. CancelledError lands on the next await. The transaction is left half-open.

Question: What do you do?

Strong answer covers:
- CancelledError must propagate. Catch it for cleanup, then re-raise:

try:
await db.execute(...)
except asyncio.CancelledError:
await transaction.rollback()  # itself await — may also be cancelled!
raise

- For "this cleanup must finish even if I'm being cancelled," wrap the cleanup in asyncio.shield(). Shield isolates the inner coroutine from the cancellation, but the outer await still raises CancelledError to the caller.
- For 3.11+: asyncio.Timeout context manager and task.uncancel() give finer control over nested cancellation. Useful when a deadline is overlaid on user-initiated cancel.
- CancelledError is a BaseException (since 3.8), not Exception. except Exception: does not swallow it — good. But except BaseException: does, and so does bare except:. Avoid both in async code.
- For "fire-and-forget cleanup" on shutdown: asyncio.shield + storing the task in a set that lifespan iterates and awaits.

Trap: Treating CancelledError as a normal exception, logging it and continuing. The cancellation never propagates, the task stays "running" from asyncio's perspective, and shutdown hangs.

---
9. The sync/async boundary

Legacy code uses psycopg2 (sync). New service is FastAPI (async). You can't migrate the DB layer this quarter.

Question: How do you bridge?

Strong answer covers:
- Async calling sync: await asyncio.to_thread(sync_fn, ...). Sync code runs in a thread; loop stays responsive. Default executor is a ThreadPoolExecutor with min(32, cpu_count + 4) workers — under load, sized your pool here matters more than people realize.
- For Django, asgiref.sync.sync_to_async does the same but propagates Django's thread-local state (current request, transactions). Use it in Django async views.
- The actual fix is asyncpg. It's typically 5–10× the throughput of psycopg2 even before counting the freed threads. If migration is a quarter away, accept the throttling and oversize the thread pool deliberately.
- Sync calling async: asyncio.run(coro()) creates a fresh loop. Once per program lifetime in production code. Throws if a loop is already running in the same thread.
- nest_asyncio is a hack to allow re-entering a running loop (used in Jupyter). It's a smell in production — usually means a design where async and sync are interleaved incorrectly.
- Architectural answer: pick a direction. Either go all-async at the top (FastAPI/aiohttp) and use to_thread for sync libs, or go all-sync and call APIs from a worker thread. Don't interleave.

Trap: asyncio.run(coro()) inside a Flask request handler. Creates and tears down an event loop per request — performance is dreadful, and connection pools (which assume a stable loop) break in surprising ways.

---
10. Multiprocessing pitfalls

Team uses multiprocessing.Pool for a CPU-bound task. Works on Linux. On macOS, intermittent deadlocks and "leaked semaphore" warnings at exit.

Question: What's going on?

Strong answer covers:
- Default start method differs: macOS is spawn since 3.8 (fork is unsafe after Apple frameworks initialize their internal threads), Linux is still fork. Behavior of code that "works" depends on the method.
- With fork: children inherit memory. A lock held by a thread in the parent becomes a permanently-held lock in the child (the thread doesn't come along). This is the classic post-fork deadlock.
- With spawn: children are fresh interpreters. Module-level state must be reconstructed. Everything passed to the pool must be picklable — closures, lambdas, and locally-defined functions will fail.
- Make start method explicit: multiprocessing.set_start_method('spawn', force=True) at entry. Now your dev box matches prod, and you've forced yourself to write pickle-safe code.
- "Leaked semaphore" on exit usually = pool wasn't close()d + join()d. Use the with block.
- DB connections: never share. Open them inside the worker, not at module load. With fork, multiple processes share a single socket and corrupt each other. With spawn, that's not a risk, but resource leaks are.
- Ergonomics: prefer concurrent.futures.ProcessPoolExecutor over multiprocessing.Pool — same machinery, cleaner API, futures-based.

Trap: Closing over a complex object (a class with a thread, a DB connection, a logger handler) and passing it through the pool. Pickling sometimes succeeds, unpickling fails in the child, you get an opaque error mid-job.

---
11. Graceful shutdown of an async service

uvicorn + FastAPI + asyncpg pool + a Redis pub/sub subscriber running as a background task. Kubernetes sends SIGTERM with a 30s grace.

Question: Sequence.

Strong answer covers:
- uvicorn handles SIGTERM by stopping accept, draining in-flight, then exiting. Tune --timeout-graceful-shutdown. Use --lifespan on so FastAPI's lifespan hook actually runs.
- FastAPI lifespan is where background tasks and pools live:

@asynccontextmanager
async def lifespan(app):
app.state.pool = await asyncpg.create_pool(DSN)
app.state.bg = asyncio.create_task(subscriber(app.state.pool))
try:
yield
finally:
app.state.bg.cancel()
try:
await app.state.bg
except asyncio.CancelledError:
pass
await app.state.pool.close()

- Order: flip readiness → 503 → uvicorn drains handlers → lifespan teardown cancels background tasks → pool closes last (handlers may have been mid-query during drain).
- The pub/sub subscriber gotcha: if it's blocked inside await pubsub.get_message() with no timeout, cancellation may not land until the underlying read returns. Wrap with asyncio.wait_for(get_message(), timeout=1) so cancellation has a bounded window.
- Kubernetes hygiene: preStop: sleep 5 to bridge the gap between SIGTERM and Endpoints removal. terminationGracePeriodSeconds ≥ your worst-case drain time + cleanup.

Trap: Background tasks created via asyncio.create_task(...) without holding the reference. The task can be garbage-collected mid-execution. 3.11 added a warning; 3.12 made the issue more prominent. Always store the task on app.state and cancel it explicitly on shutdown.

---
12. Free-threaded Python (3.13+)

Question: What changes, and when do you switch?

Strong answer covers:
- 3.13 ships python3.13t, an experimental build with the GIL removed. CPU-bound threading actually parallelizes. Default availability targeted for 3.15+; production-ready timing depends on the ecosystem.
- What it doesn't change: asyncio. The event loop never depended on the GIL for correctness — only on bytecode-level atomicity, which is preserved. Async code runs the same.
- What it does change:
    - threading becomes a real option for CPU work — no more "use multiprocessing for CPU."
    - Race conditions previously hidden by GIL bytecode atomicity become visible. counter += 1 was effectively atomic because the GIL serialized bytecodes; now it isn't. dict[k] = v is still individually atomic by design choices in the free-threaded implementation, but if k in d: d[k] += 1 is not.
    - C extensions assumed GIL ownership for thread safety. NumPy, lxml, Pillow have varying degrees of free-threaded support; check before you switch.
- Pure single-threaded performance is slightly worse on free-threaded builds (overhead from per-object locks). For most services, the trade-off only pays off if you have real multi-core CPU contention in Python code.
- Pragmatic stance: don't move production to free-threaded yet (as of mid-2026). Add locks where the GIL was implicitly protecting you, test under the free-threaded build in CI, and migrate when your critical dependencies are stable.

Trap: Assuming "no GIL = drop-in speedup." Existing thread-unsafe code with latent races finally hits them at runtime, and C extensions that haven't opted in serialize on a different lock anyway.

---
13. Testing async code

A test fails 1 in 200 runs. It fans out to 5 mocked services and asserts on the merged result.

Question: How do you make it deterministic?

Strong answer covers:
- gather doesn't guarantee return order relative to scheduling, but the result list is in input order. Flake usually comes from elsewhere: ordering of side effects (mock call recording), shared state between tests (singleton sessions), or real-time asyncio.sleep racing with assertions.
- Make assertions order-independent where order isn't a contract — assert on sets, sorted lists, or specific keys, not raw list equality.
- Mock time. freezegun works with sync time. For asyncio time, monkeypatch asyncio.sleep to a no-op or use trio/anyio testing facilities (anyio has decent virtual time; asyncio ergonomics are improving but lag).
- Isolate event loops. @pytest.fixture(scope="function") on the loop fixture (pytest-asyncio). Hidden cross-test state in module-level coroutines is a top cause of flake.
- For genuine concurrency tests, run the test under load: pytest-repeat -n 1000 nightly. Failing one in 200 means failing roughly every other thousand-run.
- For mocking aiohttp/httpx: aioresponses and respx respectively. They give deterministic, ordered mock responses without real I/O.

Trap: Sharing a single ClientSession or Engine across tests as a module-level fixture. Looks like a perf win, becomes a debugging nightmare when one test leaves state for the next.

---
These map roughly onto the staff-interviewer rubric for Python concurrency: model selection (1), event loop hygiene (2), the asyncio-specific concurrency model (3, 4, 8), I/O resource discipline (5, 7), CPU work in async contexts (6), the sync/async boundary (9), the multiprocessing landmines (10), lifecycle (11), where the language is heading (12), and how to test the whole pile (13).

If you want, I can go deeper on any one — for instance, a full structured-concurrency design for the flight aggregator (TaskGroup + per-provider circuit breaker + tail-latency budget), or a worked migration from psycopg2 + Flask to asyncpg + FastAPI laying out the staged rollout and observability gates. I can also generate adversarial follow-ups for any scenario in the style staff interviewers use when probing depth.