# Multithreading in Python — A Staff Engineer's Guide

> Audience: engineers who already know what a thread is, what a mutex is, and have shipped concurrent code in at least one language. The goal here is not "what is `threading.Thread`" — it is the mental model, trade-offs, production patterns, and failure modes you need when you are the person on the call deciding whether to thread, multiprocess, asyncio, or rewrite in a faster language.

---

## Table of Contents

1. [Mental Model: Concurrency vs Parallelism in CPython](#1-mental-model)
2. [The GIL — What It Actually Does](#2-the-gil)
3. [When Threading Helps and When It Hurts](#3-when-threading-helps)
4. [Threading Primitives — Beyond `Thread` and `Lock`](#4-threading-primitives)
5. [Real-World Scenario 1: High-Throughput Web Scraper / Crawler](#scenario-1-web-scraper)
6. [Real-World Scenario 2: Database Connection Pool](#scenario-2-db-pool)
7. [Real-World Scenario 3: Producer–Consumer Ingestion Pipeline](#scenario-3-pipeline)
8. [Real-World Scenario 4: Background Job Worker (Celery-style)](#scenario-4-worker)
9. [Real-World Scenario 5: Concurrent API Aggregator (Fan-out / Fan-in)](#scenario-5-fanout)
10. [Real-World Scenario 6: Cache Stampede Prevention (Single-flight)](#scenario-6-stampede)
11. [Real-World Scenario 7: Token-Bucket Rate Limiter](#scenario-7-rate-limit)
12. [Real-World Scenario 8: Graceful Shutdown of a Multithreaded Daemon](#scenario-8-shutdown)
13. [Anti-Patterns and Production Pitfalls](#13-anti-patterns)
14. [Threading vs Multiprocessing vs Asyncio — Decision Framework](#14-decision-framework)
15. [Modern Python: Sub-interpreters (3.12) and Free-Threaded Mode (3.13+)](#15-modern-python)
16. [Testing, Observability, and Debugging](#16-testing-observability)
17. [Cheat Sheet](#17-cheat-sheet)

---

<a id="1-mental-model"></a>
## 1. Mental Model: Concurrency vs Parallelism in CPython

Two ideas that get conflated and *must* be separated when reasoning at scale:

- **Concurrency**: structuring a program so multiple logical tasks make progress in overlapping time windows. A single core, time-sliced, is concurrent.
- **Parallelism**: physically executing multiple tasks at the same instant on multiple cores.

CPython threads give you **concurrency**, but the GIL means you only get **parallelism for I/O and certain C-extension code paths** — not for pure-Python CPU work.

This is the single most important sentence in this document. Internalize it before reading further:

> In CPython, threads are a tool for **overlapping waiting**, not for **multiplying compute**.

If your bottleneck is `requests.get`, `psycopg2.execute`, `boto3.put_object`, `time.sleep`, or any blocking syscall — threads will help. If your bottleneck is a tight Python loop computing checksums or transforming dicts — threads won't, and may make things worse due to GIL contention overhead.

### Threads vs Processes vs Coroutines (one-paragraph summary)

| Mechanism | Memory | Switch cost | Parallel CPU? | Parallel I/O? | Per-unit overhead |
|---|---|---|---|---|---|
| OS thread (`threading.Thread`) | Shared | Medium (kernel) | No (GIL) | Yes | ~8 MB stack default, ~50–200 µs creation |
| OS process (`multiprocessing.Process`) | Isolated (IPC needed) | High (kernel + page tables) | Yes | Yes | MBs RSS, ~5–50 ms creation |
| Coroutine (`asyncio` task) | Shared | Low (userspace) | No (single thread) | Yes (cooperative) | ~3 KB, ~µs |

A staff engineer should pick the model based on the workload's *blocking profile* and *inter-task communication needs*, not on what the team is most familiar with.

---

<a id="2-the-gil"></a>
## 2. The GIL — What It Actually Does

The Global Interpreter Lock is a single mutex inside the CPython runtime. To execute a Python bytecode instruction, a thread must hold the GIL. That's it. That's the rule.

### When the GIL is released

This is what most engineers don't internalize:

1. **At regular intervals** during pure-Python execution (default: every 5 ms, controlled by `sys.setswitchinterval()`). The current thread releases, an OS scheduler decision happens, some thread reacquires.
2. **Around blocking I/O syscalls** in the CPython runtime — `read`, `write`, `recv`, `send`, `select`, `poll`, `epoll_wait`, etc. The C function in question explicitly drops the GIL via the `Py_BEGIN_ALLOW_THREADS` macro, runs the syscall, then reacquires.
3. **Inside well-behaved C extensions** that release the GIL during heavy compute (NumPy linalg, Pillow image ops, lxml parsing, hashlib for large buffers, zlib, bz2). This is why numerical Python *does* see thread parallelism.
4. **`time.sleep()`** releases the GIL — common interview gotcha.

### What this means in practice

```python
# Threads help: each requests call drops the GIL during the socket recv()
def fetch(url): return requests.get(url).text

# Threads do NOT help: pure Python compute, threads serialize on the GIL
def hash_pure_python(s):
    h = 0
    for c in s:                 # GIL held for the entire loop body
        h = (h * 31 + ord(c)) & 0xFFFFFFFF
    return h

# Threads help again: hashlib.sha256 releases the GIL for buffers >~2KB
def hash_c_extension(b): return hashlib.sha256(b).hexdigest()
```

### GIL contention: a real cost, not just an absence of speedup

When N threads all want to run pure-Python code, they ping-pong the GIL every switch interval. Each handoff is a syscall + cache miss + scheduler decision. On NUMA hardware or busy machines, this can make a multithreaded CPU-bound Python program **slower** than the same program single-threaded. There is published measurement of this — David Beazley's "Inside the Python GIL" talks demonstrate ~2× slowdowns on dual-core for CPU-bound workloads.

### The 3.12 / 3.13 escape hatches (covered in §15)

- **PEP 684** — per-interpreter GIL (3.12)
- **PEP 703** — optional free-threaded build (3.13+, experimental; default in 3.14+)

These change the calculus. But for almost all production Python deployed today, the rule above holds.

---

<a id="3-when-threading-helps"></a>
## 3. When Threading Helps and When It Hurts

A decision matrix:

| Workload character | Threading? | Better alternative |
|---|---|---|
| Many slow HTTP/DB calls, modest concurrency (≤500) | **Yes** | asyncio if you control the whole stack |
| Many slow HTTP calls, very high concurrency (≥10k) | No (8 MB × 10k = 80 GB) | asyncio |
| CPU-bound pure Python | No | multiprocessing, Cython, native rewrite |
| CPU-bound with NumPy/SciPy/Polars | **Yes** (releases GIL) | — |
| Mixed: small CPU + large I/O | **Yes** | — |
| Need shared mutable state across cores | **Yes** | — (multiprocessing requires shared memory tricks) |
| Need true CPU parallelism + shared state | No | 3.13 free-threaded build (carefully), or `multiprocessing.shared_memory` |
| Embedded/integrated with blocking C library that hates being called from multiple threads | No | one process per worker |

The "modest concurrency" caveat is important: **OS threads are not free**. The default stack is 8 MB on Linux. You can lower it with `threading.stack_size(...)` but kernels still have per-task overhead. Past a few thousand threads, you are leaving performance on the floor versus asyncio.

---

<a id="4-threading-primitives"></a>
## 4. Threading Primitives — Beyond `Thread` and `Lock`

A staff-level practitioner should reach for the right primitive without thinking. A quick tour, with the *non-obvious* notes:

### `threading.Lock` vs `RLock`

- `Lock` — non-reentrant. The same thread re-acquiring it deadlocks itself. Use this by default; the non-reentrancy forces you to confront accidental recursion through your locking.
- `RLock` — reentrant. Same thread can acquire N times, must release N times. Use only when you genuinely have nested critical sections in *the same call stack* that share the lock (e.g., a public API method calls a private helper that also acquires).

> Convention I'd push on a team: prefer `Lock`, only switch to `RLock` after you have a concrete, documented reason. RLock often hides design problems where two layers should not be sharing a single lock.

### `threading.Condition`

For "wait until predicate becomes true" patterns. Always pair with a *while loop*, never an `if`:

```python
with cv:
    while not predicate():     # spurious wake-up safety
        cv.wait()
    # predicate is now true and we hold the lock
```

`notify()` wakes one waiter, `notify_all()` wakes all. Prefer `notify_all()` unless you have measured contention; `notify()` with multiple distinct predicates is a famous source of subtle "lost wakeup" bugs (the "broadcast vs signal" issue, well-documented since Mesa-style monitors in the 1980s).

### `threading.Event`

A one-shot or sticky boolean. Once `.set()`, all current and future `.wait()` calls return immediately until `.clear()`. Use for **shutdown signals** and **initialization-complete signals**. Do not use for queueing — use `queue.Queue`.

### `threading.Semaphore` and `BoundedSemaphore`

Counting permits. Use for **limiting concurrency to an external resource** (max 10 concurrent DB connections, max 4 concurrent HTTP calls to a third-party API).

`BoundedSemaphore` raises `ValueError` on `release()` past initial value — this catches the bug where you accidentally `release()` more than you `acquire()`, which silently increases your concurrency limit forever. **Always use `BoundedSemaphore` over `Semaphore` unless you have an explicit reason.**

### `queue.Queue`, `LifoQueue`, `PriorityQueue`

Thread-safe, blocking, supports `maxsize` for backpressure. The MVP for almost all producer–consumer designs. Properties to remember:

- `put(item, timeout=...)` blocks if queue is full — this is the natural backpressure mechanism. Never set `maxsize=0` (unbounded) in production.
- `task_done()` + `join()` is a clean way to wait for all submitted work to finish.
- `Queue` uses an `RLock` internally and a `Condition` for signalling — don't reimplement it.

### `concurrent.futures.ThreadPoolExecutor`

Modern API. Pool of reusable threads. Returns `Future` objects. Use this in 95% of cases; only reach for raw `threading.Thread` when you need something the pool cannot model (e.g., long-running daemons with custom lifecycles).

```python
with ThreadPoolExecutor(max_workers=32) as ex:
    futures = [ex.submit(fetch, url) for url in urls]
    for fut in as_completed(futures):
        try:
            result = fut.result(timeout=30)
        except Exception:
            log.exception("worker failed")
```

Notes:
- `executor.map(...)` swallows exceptions until you iterate the result. Prefer explicit `submit` + `as_completed` for production code where you want to fail fast or partial-fail loudly.
- The `with` block calls `shutdown(wait=True)`. Be aware: this **waits for queued tasks to finish**. For fast shutdown, use `shutdown(wait=False, cancel_futures=True)` (3.9+).

### `threading.local()`

Per-thread storage. Useful for thread-bound DB connections, request contexts. Pitfall: with `ThreadPoolExecutor`, threads are **reused**, so `threading.local` state from a previous task leaks into the next. Always `reset()` or use a context manager.

### `threading.Barrier`

N threads call `wait()`, none proceed until all N arrive. Use for parallel pipeline stages with synchronization points (e.g., parallel matrix decomposition phases). Rare in I/O-bound code; common in numerical code.

---

<a id="scenario-1-web-scraper"></a>
## 5. Real-World Scenario 1: High-Throughput Web Scraper / Crawler

**Problem:** crawl 50,000 URLs, store HTML in S3, respect per-domain rate limits, retry on 5xx with backoff, surface failures.

**Why threading fits:** the work is overwhelmingly waiting on remote I/O. Each thread spends ~95% of its life blocked on a socket. With 64 threads, 60 of them are blocked at any instant — the GIL is rarely the bottleneck.

**Why not asyncio?** if the team's existing codebase, ORM, S3 client, and metrics library are all sync, switching to asyncio is a rewrite. Threading leverages the existing sync ecosystem.

```python
import logging
import threading
import time
from collections import defaultdict
from concurrent.futures import ThreadPoolExecutor, as_completed
from queue import Queue
from urllib.parse import urlparse

import boto3
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

log = logging.getLogger(__name__)

# Per-domain semaphore: cap concurrency to politeness budget.
# This is a classic and easy-to-get-wrong pattern — note the lock.
class PerDomainLimiter:
    def __init__(self, per_domain_max: int = 4):
        self._per_domain_max = per_domain_max
        self._sems: dict[str, threading.BoundedSemaphore] = {}
        self._lock = threading.Lock()

    def acquire(self, domain: str):
        with self._lock:
            sem = self._sems.get(domain)
            if sem is None:
                sem = threading.BoundedSemaphore(self._per_domain_max)
                self._sems[domain] = sem
        sem.acquire()                  # blocking call OUTSIDE the dict lock — important!
        return sem


def make_session() -> requests.Session:
    """A pooled, retry-aware session. Pool sized to expected per-thread reuse."""
    s = requests.Session()
    retry = Retry(
        total=3, backoff_factor=0.5,
        status_forcelist=(500, 502, 503, 504),
        allowed_methods=frozenset(["GET", "HEAD"]),
    )
    adapter = HTTPAdapter(pool_connections=64, pool_maxsize=64, max_retries=retry)
    s.mount("http://", adapter)
    s.mount("https://", adapter)
    return s


# ONE session per thread: requests.Session is thread-safe for concurrent get/post,
# but the connection pool is more efficiently used per-thread for keep-alive reasons.
_thread_local = threading.local()

def get_session() -> requests.Session:
    sess = getattr(_thread_local, "sess", None)
    if sess is None:
        sess = make_session()
        _thread_local.sess = sess
    return sess


def fetch_one(url: str, limiter: PerDomainLimiter, s3, bucket: str) -> tuple[str, str]:
    domain = urlparse(url).netloc
    sem = limiter.acquire(domain)
    try:
        resp = get_session().get(url, timeout=(5, 30))   # (connect, read)
        resp.raise_for_status()
        key = f"crawl/{domain}/{abs(hash(url))}.html"
        s3.put_object(Bucket=bucket, Key=key, Body=resp.content)
        return url, "ok"
    finally:
        sem.release()


def crawl(urls: list[str], bucket: str, workers: int = 64) -> dict[str, str]:
    limiter = PerDomainLimiter(per_domain_max=4)
    s3 = boto3.client("s3")            # boto3 client is thread-safe per docs
    results: dict[str, str] = {}

    with ThreadPoolExecutor(max_workers=workers, thread_name_prefix="crawl") as ex:
        future_to_url = {ex.submit(fetch_one, u, limiter, s3, bucket): u for u in urls}
        for fut in as_completed(future_to_url):
            url = future_to_url[fut]
            try:
                _, status = fut.result()
                results[url] = status
            except Exception as e:
                log.warning("fetch failed url=%s err=%s", url, e)
                results[url] = f"err:{type(e).__name__}"
    return results
```

### Staff-level callouts on this code

1. **Lazy semaphore creation under a lock**, but `acquire()` happens outside the lock — otherwise you serialize ALL crawls behind the bookkeeping lock.
2. **`thread_name_prefix`** — your stack traces, py-spy dumps, and metrics will all be more readable. Free win.
3. **Two timeouts** on `requests`, not one — connect timeout protects against unresponsive networks, read timeout against slow servers.
4. **`s3.put_object` is thread-safe** because boto3 clients are stateless wrappers over thread-local sessions. Do not share the *resource* objects (`boto3.resource(...)`) across threads — they cache state.
5. **`make_session()` per thread, not per request** — TCP keep-alive and TLS session resumption. With 64 threads and 4 conns each, the host needs ~256 sockets — bump nofile if needed.
6. **No global retry queue** — `urllib3.Retry` handles 5xx; for true dead-letter logic you'd push failures to a separate queue and reprocess.

### Anti-pattern someone will write

```python
sem = threading.BoundedSemaphore(4)
def fetch_one(url):
    with sem:                          # global cap — 4 across ENTIRE crawl
        ...
```

This caps total concurrency at 4 instead of per-domain. You'll get politeness from the user's POV, but throughput collapses. The fix is per-key semaphores, as above.

---

<a id="scenario-2-db-pool"></a>
## 6. Real-World Scenario 2: Database Connection Pool

**Problem:** N application threads, M database connections, M ≪ N. Threads check connections out, use them briefly, return them. Must handle connection death, must not leak on exception.

This is the canonical use of `BoundedSemaphore` + `Queue`:

```python
import threading
import contextlib
import time
from queue import Queue, Empty

import psycopg2

class ConnectionPool:
    def __init__(self, dsn: str, size: int = 10, max_age_seconds: int = 1800):
        self._dsn = dsn
        self._size = size
        self._max_age = max_age_seconds
        self._pool: Queue = Queue(maxsize=size)
        self._sem = threading.BoundedSemaphore(size)
        self._lock = threading.Lock()
        self._created = 0
        # Lazy creation: don't build conns we may never need at boot time.

    def _new_conn(self):
        conn = psycopg2.connect(self._dsn)
        conn._created_at = time.monotonic()
        return conn

    def _is_stale(self, conn) -> bool:
        return time.monotonic() - conn._created_at > self._max_age

    @contextlib.contextmanager
    def acquire(self, timeout: float = 5.0):
        if not self._sem.acquire(timeout=timeout):
            raise TimeoutError("no connection available")
        conn = None
        try:
            try:
                conn = self._pool.get_nowait()
                if self._is_stale(conn) or conn.closed:
                    conn.close()
                    conn = self._new_conn()
            except Empty:
                with self._lock:
                    if self._created < self._size:
                        self._created += 1
                        conn = self._new_conn()
                    else:
                        # Should not happen — sem guarantees space, but defensive.
                        conn = self._pool.get(timeout=timeout)
            yield conn
        except psycopg2.Error:
            # Bad connection — drop it, do not return to pool.
            if conn is not None:
                try: conn.close()
                except Exception: pass
                conn = None
            raise
        finally:
            if conn is not None and not conn.closed:
                try:
                    if conn.status != psycopg2.extensions.STATUS_READY:
                        conn.rollback()                    # don't return mid-tx
                except Exception:
                    try: conn.close()
                    except Exception: pass
                    conn = None
            if conn is not None:
                self._pool.put(conn)
            self._sem.release()
```

### Staff-level callouts

1. **`BoundedSemaphore` enforces the cap** — the queue alone can't, because a buggy caller could `get_nowait()` past the limit if you ever miscount.
2. **Health check on checkout** (`closed`, `_is_stale`) — connections die behind your back: NAT timeouts, DB restarts, cluster failovers. A pool that hands out dead connections is a footgun.
3. **`rollback()` on return** — if a thread crashed mid-transaction without committing, the connection is stuck in `IDLE IN TRANSACTION`, which on PostgreSQL holds locks and bloats. Rolling back on return is cheap insurance.
4. **`acquire(timeout=...)`** — never `acquire()` indefinitely in production. You will wedge under saturation.
5. **Lazy creation** — opening 100 DB connections at boot is wasteful and contributes to thundering-herd reconnects after a DB failover.

### Why not just use `psycopg2.pool.ThreadedConnectionPool`?

In real life, **use the library**. The above is the reference implementation you should be able to write on a whiteboard. Production code uses `psycopg2.pool.ThreadedConnectionPool`, `SQLAlchemy.QueuePool`, or `pgbouncer`. But knowing the internals lets you debug them when they misbehave (and they do — pgbouncer in transaction-pooling mode has surprises with prepared statements, `SET LOCAL`, etc.).

---

<a id="scenario-3-pipeline"></a>
## 7. Real-World Scenario 3: Producer–Consumer Ingestion Pipeline

**Problem:** ingest events from Kafka → enrich via HTTP API → write to ClickHouse in batches. Stages have very different throughput characteristics: Kafka is fast, HTTP enrichment is the bottleneck, ClickHouse loves big batches.

Multi-stage pipelines map naturally to **bounded queues between stages**. Each stage is its own thread pool, sized independently. Bounded queues provide backpressure — slow stages slow down upstream stages instead of OOMing.

```python
import threading
import logging
from queue import Queue
from concurrent.futures import ThreadPoolExecutor

log = logging.getLogger(__name__)

SHUTDOWN = object()    # sentinel marking end-of-stream

class Pipeline:
    def __init__(self, kafka_consumer, http_client, ch_writer):
        self.kafka = kafka_consumer
        self.http = http_client
        self.ch = ch_writer
        # Queue sizes tuned to per-stage throughput.
        # If enrichment is the slowest stage, raw queue should NOT be huge —
        # otherwise you bloat memory and increase end-to-end latency.
        self.raw_q: Queue = Queue(maxsize=10_000)
        self.enriched_q: Queue = Queue(maxsize=5_000)
        self.stop = threading.Event()

    def stage_consume(self):
        try:
            for msg in self.kafka:                      # blocking iterator
                if self.stop.is_set(): break
                self.raw_q.put(msg)                     # blocks on backpressure
        finally:
            self.raw_q.put(SHUTDOWN)

    def stage_enrich(self):
        while not self.stop.is_set():
            msg = self.raw_q.get()
            if msg is SHUTDOWN:
                self.raw_q.put(SHUTDOWN)                # propagate to siblings
                self.enriched_q.put(SHUTDOWN)
                return
            try:
                enriched = self.http.enrich(msg)
                self.enriched_q.put(enriched)
            except Exception:
                log.exception("enrich failed; dropping msg=%s", msg.id)
            finally:
                self.raw_q.task_done()

    def stage_write(self, batch_size: int = 1_000, max_latency_s: float = 2.0):
        import time
        batch = []
        deadline = time.monotonic() + max_latency_s
        seen_shutdown = False
        while not seen_shutdown:
            timeout = max(0.0, deadline - time.monotonic())
            try:
                item = self.enriched_q.get(timeout=timeout)
                if item is SHUTDOWN:
                    seen_shutdown = True
                else:
                    batch.append(item)
            except Exception:        # Empty -> queue.Empty, treat as flush trigger
                pass
            now = time.monotonic()
            if len(batch) >= batch_size or now >= deadline:
                if batch:
                    self.ch.bulk_insert(batch)
                    batch = []
                deadline = now + max_latency_s
        if batch:
            self.ch.bulk_insert(batch)

    def run(self):
        with ThreadPoolExecutor(max_workers=1, thread_name_prefix="consume") as p_consume, \
             ThreadPoolExecutor(max_workers=16, thread_name_prefix="enrich") as p_enrich, \
             ThreadPoolExecutor(max_workers=2, thread_name_prefix="write") as p_write:

            p_consume.submit(self.stage_consume)
            for _ in range(16):
                p_enrich.submit(self.stage_enrich)
            for _ in range(2):
                p_write.submit(self.stage_write)
```

### Staff-level callouts

1. **Bounded queues, sized differently per stage**. If your stages all use the same queue size you're flying blind on backpressure semantics. Tune by measuring per-stage throughput under load.
2. **Sentinel-based shutdown propagates through queues**, not via `Event` alone. A worker blocked on `get()` will never check the event. The sentinel pattern guarantees every worker eventually wakes up. The `self.raw_q.put(SHUTDOWN)` re-publish is critical — it's how stages with N consumers all see the sentinel.
3. **Time-or-size based batching** — `max_latency_s` ensures low-traffic moments still flush, `batch_size` ensures high-traffic moments don't OOM. Both bounds matter.
4. **Stage-level error handling**: enrichment failures log and drop, but don't kill the worker. Whether to drop, dead-letter, or halt is a *product decision* — make it explicit, not implicit.
5. **`task_done()` only matters if you want to `join()`**; if shutdown is sentinel-driven you can omit.

### When to switch from threads to a real broker

If your queue is non-trivial (durable, multi-process, multi-host), don't reinvent — put Kafka, NATS, RabbitMQ, or Redis Streams between stages. In-process threaded pipelines are right when:

- All stages run in one process (simpler ops, lower latency)
- You can afford to lose in-flight items on crash, or you ack from Kafka *only after* writing to ClickHouse
- Throughput fits one machine

If any of those break — externalize.

---

<a id="scenario-4-worker"></a>
## 8. Real-World Scenario 4: Background Job Worker (Celery-style)

**Problem:** poll Redis (or SQS) for jobs, dispatch by job-type, ensure visibility timeouts are extended for long-running jobs (heartbeat), handle graceful shutdown without losing jobs.

Two threads-per-job patterns are interesting here:

```python
import threading
import time
import logging
from concurrent.futures import ThreadPoolExecutor

log = logging.getLogger(__name__)

class Worker:
    def __init__(self, sqs_client, queue_url, handlers, concurrency=8,
                 visibility_timeout=60, heartbeat_interval=20):
        self.sqs = sqs_client
        self.queue_url = queue_url
        self.handlers = handlers           # {job_type: callable}
        self.executor = ThreadPoolExecutor(max_workers=concurrency,
                                           thread_name_prefix="job")
        self.visibility = visibility_timeout
        self.heartbeat = heartbeat_interval
        self.stop = threading.Event()
        self.in_flight = set()
        self.in_flight_lock = threading.Lock()

    def _heartbeat_loop(self, receipt_handle, done_event):
        """Extend visibility timeout while job is running."""
        while not done_event.wait(self.heartbeat):
            try:
                self.sqs.change_message_visibility(
                    QueueUrl=self.queue_url,
                    ReceiptHandle=receipt_handle,
                    VisibilityTimeout=self.visibility,
                )
            except Exception:
                log.exception("heartbeat failed; job may be redelivered")
                return

    def _process(self, msg):
        receipt = msg["ReceiptHandle"]
        done = threading.Event()
        hb_thread = threading.Thread(
            target=self._heartbeat_loop, args=(receipt, done),
            name=f"hb-{msg['MessageId'][:8]}", daemon=True,
        )
        hb_thread.start()
        try:
            body = msg["Body"]
            job_type = msg["MessageAttributes"]["type"]["StringValue"]
            handler = self.handlers[job_type]
            handler(body)
            self.sqs.delete_message(QueueUrl=self.queue_url, ReceiptHandle=receipt)
        except Exception:
            log.exception("job failed; will be redelivered")
            # Do NOT delete message — let SQS redeliver after visibility timeout.
        finally:
            done.set()
            hb_thread.join(timeout=2)
            with self.in_flight_lock:
                self.in_flight.discard(msg["MessageId"])

    def run(self):
        try:
            while not self.stop.is_set():
                resp = self.sqs.receive_message(
                    QueueUrl=self.queue_url,
                    MaxNumberOfMessages=10,
                    WaitTimeSeconds=20,
                    VisibilityTimeout=self.visibility,
                    MessageAttributeNames=["type"],
                )
                for msg in resp.get("Messages", []):
                    if self.stop.is_set(): break
                    with self.in_flight_lock:
                        self.in_flight.add(msg["MessageId"])
                    self.executor.submit(self._process, msg)
        finally:
            log.info("draining %d in-flight jobs", len(self.in_flight))
            self.executor.shutdown(wait=True)
```

### Staff-level callouts

1. **The heartbeat thread per job** is necessary because SQS has *visibility timeouts*. If your job takes 5 minutes but the timeout is 60 seconds, SQS will redeliver to another worker — you'll process the same job twice, and your idempotency story had better be airtight. The heartbeat extends visibility while the job runs.
2. **`done_event.wait(interval)` instead of `time.sleep(interval)`** — wakes immediately on completion, no needless extra heartbeat after the job is done.
3. **Daemon threads for heartbeats** — if the worker process is killed hard, daemons die with it. We don't want orphan heartbeat threads keeping a job from being redelivered.
4. **"Don't delete on failure"** — relying on visibility timeout for redelivery is the SQS idiom. Other queue systems differ (RabbitMQ uses NACK, Redis Streams uses XACK + claim). Match the semantics to the broker.
5. **Graceful drain on shutdown** — `executor.shutdown(wait=True)` waits for in-flight jobs. Pair with a SIGTERM handler that sets `self.stop`. Most orchestrators (Kubernetes, ECS) give you 30s before SIGKILL — make sure your `terminationGracePeriodSeconds` is ≥ longest-job-time.

### Discussion: at-least-once vs at-most-once

This worker is **at-least-once**: redelivery on failure means a job can run twice. Your handlers must be idempotent (e.g., use `INSERT ... ON CONFLICT DO NOTHING` keyed by `MessageId`). At-most-once is achievable but requires deleting the message *before* processing — and you lose work on crash. **At-least-once + idempotency is almost always the right choice.**

---

<a id="scenario-5-fanout"></a>
## 9. Real-World Scenario 5: Concurrent API Aggregator (Fan-out / Fan-in)

**Problem:** for each user request, call 5 downstream microservices in parallel, aggregate results, return to client. Total latency = max(downstream) + aggregation overhead.

This is a classic fan-out/fan-in. Threading is a clean fit because each downstream call blocks. The tempting wrong move is to spawn a `Thread` per call, per request — which means per-user-request thread creation cost.

```python
from concurrent.futures import ThreadPoolExecutor, FIRST_EXCEPTION, wait
from dataclasses import dataclass
from typing import Callable, Any

# Process-wide pool, sized for the per-request fan-out × concurrent requests.
# If each request fans out to 5 calls and you serve 200 RPS at 100ms each,
# instantaneous concurrency is ~100 active calls, so a pool of 128 is roughly right.
# If you under-size, requests queue at the executor; if you over-size, you waste threads.
_AGGREGATOR_POOL = ThreadPoolExecutor(max_workers=128, thread_name_prefix="agg")


@dataclass
class CallSpec:
    name: str
    fn: Callable[..., Any]
    timeout: float
    required: bool = True


def aggregate(specs: list[CallSpec], request_deadline_s: float) -> dict[str, Any]:
    futures = {_AGGREGATOR_POOL.submit(s.fn): s for s in specs}
    results: dict[str, Any] = {}
    errors: dict[str, Exception] = {}

    done, not_done = wait(futures, timeout=request_deadline_s)

    # Cancel any still-pending: cancellation only works on tasks that haven't started.
    # In-flight calls keep running until they block on socket; they can't be killed
    # cleanly. This is a known limitation of Python threads.
    for fut in not_done:
        fut.cancel()

    for fut, spec in futures.items():
        if fut in not_done:
            errors[spec.name] = TimeoutError(f"{spec.name} exceeded request deadline")
            continue
        exc = fut.exception()
        if exc is not None:
            errors[spec.name] = exc
        else:
            results[spec.name] = fut.result()

    # Required-vs-optional logic: if a non-required service errored, we serve degraded.
    fatal = [name for name, e in errors.items()
             if next(s for s in specs if s.name == name).required]
    if fatal:
        raise RuntimeError(f"required calls failed: {fatal}; errors: {errors}")
    return results
```

### Staff-level callouts

1. **Process-wide pool**, not per-request pool. Per-request pools mean per-request thread *creation*, which is precisely what a pool exists to avoid.
2. **Pool sizing math**: `pool_size ≈ peak_RPS × avg_call_latency × per_request_fanout × headroom_factor`. Under-sized pool → tasks queue at the executor and you fail SLO without saturating downstream. Over-sized → wasted memory + GIL contention. **Measure**, don't guess.
3. **`wait(..., timeout=...)`** with `FIRST_EXCEPTION` (or just `ALL_COMPLETED`) instead of per-future timeouts — single deadline for the whole request. Per-future timeouts double-count the budget.
4. **Cancellation caveat**: `Future.cancel()` only succeeds if the task hasn't started running. Once a thread is in `socket.recv`, you can't kill it from outside Python. The thread will eventually time out (because you set `socket.settimeout()` on the underlying client, *right*?) and finish. This is the **biggest reason threads are not always the right answer for low-latency aggregation** — asyncio's cooperative cancellation is cleaner here.
5. **Required vs optional pattern** — almost every aggregator has this distinction. Bake it in or you'll bolt it on later via if-spaghetti.
6. **Always set client-level timeouts** — `requests.get(..., timeout=...)`, gRPC deadlines, etc. Without them, "timeout the future" is a lie because the thread never returns.

---

<a id="scenario-6-stampede"></a>
## 10. Real-World Scenario 6: Cache Stampede Prevention (Single-flight)

**Problem:** 1,000 concurrent requests miss the cache simultaneously for the same key. Without coordination, all 1,000 hit the upstream — a *thundering herd*. You want exactly one of them to fetch, and the other 999 to wait for that result.

Go has [`golang.org/x/sync/singleflight`](https://pkg.go.dev/golang.org/x/sync/singleflight). Python doesn't ship one, but it's ~30 lines:

```python
import threading
from typing import Callable, Any
from dataclasses import dataclass, field

@dataclass
class _Call:
    event: threading.Event = field(default_factory=threading.Event)
    result: Any = None
    error: BaseException | None = None

class SingleFlight:
    def __init__(self):
        self._calls: dict[str, _Call] = {}
        self._lock = threading.Lock()

    def do(self, key: str, fn: Callable[[], Any]) -> Any:
        with self._lock:
            existing = self._calls.get(key)
            if existing is not None:
                pass  # we will wait below
            else:
                existing = _Call()
                self._calls[key] = existing
                am_leader = True
        try:
            am_leader  # NameError if we didn't take the leader path
        except NameError:
            am_leader = False

        if am_leader:
            try:
                existing.result = fn()
            except BaseException as e:
                existing.error = e
            finally:
                existing.event.set()
                with self._lock:
                    self._calls.pop(key, None)
        else:
            existing.event.wait()

        if existing.error is not None:
            raise existing.error
        return existing.result
```

Cleaner version using a flag rather than NameError gymnastics:

```python
class SingleFlight:
    def __init__(self):
        self._calls: dict[str, _Call] = {}
        self._lock = threading.Lock()

    def do(self, key, fn):
        with self._lock:
            call = self._calls.get(key)
            am_leader = call is None
            if am_leader:
                call = _Call()
                self._calls[key] = call

        if am_leader:
            try:
                call.result = fn()
            except BaseException as e:
                call.error = e
            finally:
                call.event.set()
                with self._lock:
                    self._calls.pop(key, None)
        else:
            call.event.wait()

        if call.error is not None:
            raise call.error
        return call.result
```

### Staff-level callouts

1. **The lock is short** — only protects the dictionary. The actual `fn()` runs lock-free, so independent keys don't serialize.
2. **Errors propagate to all waiters**. Whether this is what you want is a product decision — Go's `singleflight` does the same. Some people prefer "n-1 waiters retry" but that's just a thundering herd of retries.
3. **Use case beyond cache:** anywhere you have idempotent expensive operations triggered by concurrent requests — token refresh (OAuth), config reload, schema fetch, leader election warm-up. **In any system serving > a few RPS, you should be reaching for this pattern by reflex.**
4. **Don't combine with an LRU cache without thought**. `lru_cache` is itself thread-safe (uses an `RLock` internally) but does **not** prevent stampede — two threads both miss, both compute, second one's result wins. Use `lru_cache` *outside* singleflight for correctness; use singleflight when computation is expensive enough to warrant deduplication.

---

<a id="scenario-7-rate-limit"></a>
## 11. Real-World Scenario 7: Token-Bucket Rate Limiter

**Problem:** limit calls to a third-party API to 100 requests per second, smoothly. Bursty traffic OK up to a small ceiling.

Token buckets are the workhorse rate-limiting algorithm. Threaded implementation:

```python
import threading
import time

class TokenBucket:
    def __init__(self, rate_per_sec: float, capacity: int | None = None):
        self.rate = rate_per_sec
        self.capacity = capacity if capacity is not None else int(rate_per_sec)
        self._tokens = float(self.capacity)
        self._last = time.monotonic()
        self._lock = threading.Lock()
        self._cv = threading.Condition(self._lock)

    def _refill(self) -> None:
        now = time.monotonic()
        delta = now - self._last
        self._tokens = min(self.capacity, self._tokens + delta * self.rate)
        self._last = now

    def acquire(self, n: int = 1, timeout: float | None = None) -> bool:
        deadline = None if timeout is None else time.monotonic() + timeout
        with self._cv:
            while True:
                self._refill()
                if self._tokens >= n:
                    self._tokens -= n
                    self._cv.notify(1)              # next waiter may now make progress
                    return True
                if deadline is not None:
                    remaining = deadline - time.monotonic()
                    if remaining <= 0:
                        return False
                    wait_for = min(remaining, (n - self._tokens) / self.rate)
                else:
                    wait_for = (n - self._tokens) / self.rate
                self._cv.wait(timeout=wait_for)
```

### Staff-level callouts

1. **`time.monotonic()`, never `time.time()`** for elapsed-duration math. `time.time()` can jump backwards (NTP, leap seconds, clock adjustments).
2. **`_refill` lazily computes tokens** — no background thread tick. Fewer threads, fewer wakes, lower steady-state cost.
3. **`_cv.wait(timeout=...)` with computed sleep** — instead of a tight polling loop. The wake-up time is exactly when enough tokens *would* have refilled.
4. **`notify(1)` on success**, not `notify_all()` — wakes one waiter who may now be able to proceed; mass notify thrashes when the bucket has only a few tokens.
5. **Caveat — fairness**: this is not a strict-FIFO queue. A late-arriving small request can grab a token before a long-waiting big one. For most rate-limiting that's fine; if you need fairness, layer a queue on top.
6. **Distributed rate limiting is not this** — across multiple processes/hosts, you need Redis with `INCR + EXPIRE` patterns, or a token-bucket service. In-process token bucket is the per-pod local enforcer; cross-pod limits require coordination.

---

<a id="scenario-8-shutdown"></a>
## 12. Real-World Scenario 8: Graceful Shutdown of a Multithreaded Daemon

**Problem:** SIGTERM arrives. You have N worker threads, K acceptor threads, an ingest queue. You have ~25 seconds (typical k8s `terminationGracePeriodSeconds`) to drain cleanly. After that, SIGKILL.

The mistake almost every team makes: signal handlers and threads. Python's signal handlers **only run on the main thread**. If your main thread is doing real work and not yielding, signals queue indefinitely.

The robust pattern:

```python
import signal
import threading
import logging
import sys

log = logging.getLogger(__name__)

class Application:
    def __init__(self):
        self.stop = threading.Event()
        self.threads: list[threading.Thread] = []

    def start(self):
        # Register signal handlers BEFORE spawning workers.
        signal.signal(signal.SIGTERM, self._on_signal)
        signal.signal(signal.SIGINT, self._on_signal)

        # Spawn workers as non-daemon threads so we can join cleanly.
        for i in range(8):
            t = threading.Thread(target=self._worker, args=(i,),
                                 name=f"worker-{i}", daemon=False)
            t.start()
            self.threads.append(t)

        # Main thread: park, but stay responsive to signals.
        # DON'T do `for t in threads: t.join()` — that blocks signal handling
        # in some Python versions/configs.
        while not self.stop.is_set():
            self.stop.wait(timeout=1.0)

        log.info("shutting down: waiting for workers")
        deadline = 25.0
        for t in self.threads:
            t.join(timeout=deadline)
            deadline = max(0.5, deadline - 1.0)

        # Anything still alive: log it. We're about to be SIGKILLed.
        stuck = [t.name for t in self.threads if t.is_alive()]
        if stuck:
            log.error("threads did not drain in time: %s", stuck)
            sys.exit(1)

    def _on_signal(self, signum, frame):
        log.warning("received signal %s; initiating shutdown", signum)
        self.stop.set()

    def _worker(self, idx):
        while not self.stop.is_set():
            try:
                item = work_queue.get(timeout=1.0)        # short timeout — see note
            except Empty:
                continue
            self._handle(item)
```

### Staff-level callouts

1. **Signal handlers run on the main thread only.** The main thread must regularly enter the bytecode interpreter for the handler to run. A `t.join()` in CPython 3.x *does* allow signal handling on Linux (the join is interruptible), but the safer idiom is the `Event.wait` loop above.
2. **Non-daemon worker threads.** Daemon threads are terminated abruptly at interpreter shutdown — finally blocks may not run, mid-write files corrupt, sockets close ungracefully. For workers you want to drain, **daemon=False**.
3. **Short `get(timeout=...)`** in workers, not `get()` blocking forever. A thread blocked on a queue cannot check `self.stop`. Either short-poll, or push sentinels into the queue at shutdown — both work, the trade-off is responsiveness vs cleanliness.
4. **Per-thread join deadline**, declining over time. Without a deadline, one wedged thread holds up shutdown indefinitely.
5. **Log threads that didn't drain.** Forensic gold when you investigate "why did k8s SIGKILL my pod" later.
6. **`SIGINT` on Windows is special** — only the main thread can be interrupted, and `signal.signal(SIGINT, ...)` is the *only* way (no `pthread_sigmask` equivalent). Cross-platform daemons should test on Windows.

---

<a id="13-anti-patterns"></a>
## 13. Anti-Patterns and Production Pitfalls

### 13.1 Holding a lock while doing I/O

```python
# WRONG
with self.cache_lock:
    if key not in self.cache:
        self.cache[key] = expensive_http_call(key)   # blocks all readers
    return self.cache[key]
```

You serialized every cache read behind one slow HTTP call. The pattern is to acquire, check, release, do work, re-acquire — or use single-flight (§10).

### 13.2 Race conditions on "atomic" Python operations

`+=`, `-=`, `dict[k] = v`, `list.append(x)` are *not* atomic in the language sense. They happen to be atomic *in CPython today* because of the GIL (for some of them), but:

- The GIL contract is a CPython implementation detail, not a language guarantee. PyPy, IronPython, and the 3.13 free-threaded build break it.
- `counter += 1` is `LOAD, LOAD_CONST, BINARY_ADD, STORE` — multiple bytecodes, GIL can switch between them. **It is not atomic, even in classic CPython.** Use `threading.Lock` or `itertools.count()`.

```python
counter = itertools.count()
def increment(): next(counter)        # truly atomic, in C
```

### 13.3 Sharing mutable state without locks because "it's fine"

It's fine until traffic doubles, the GIL switch interval changes, you move to PyPy, you upgrade Python, or someone adds a `time.sleep` mid-critical-section. **If your correctness depends on the GIL, document it; if not, lock it.**

### 13.4 `time.sleep` in a lock

The GIL is released during `sleep`, but **your `Lock` is not**. Waiters on that lock now block for the sleep duration. Always sleep outside locks.

### 13.5 Daemon threads that do important work

Daemon threads are killed at interpreter shutdown without finally blocks running. Logger threads, metrics threads, file writers should not be daemon if they buffer state.

### 13.6 `ThreadPoolExecutor` + `fork()`

If you call `os.fork()` (or use `multiprocessing` with `fork` start method) after creating a `ThreadPoolExecutor`, the child process inherits a *broken* pool — the threads don't exist in the child, but their queue and locks do. Submitting to it hangs. **Create executors after fork, not before.** Same applies to gevent/eventlet monkey-patching.

### 13.7 Lock convoys and priority inversion

When many threads contend on one lock, the OS scheduler can produce a *convoy* — threads queue, run briefly, release, get pre-empted, queue again. Throughput collapses to roughly serial.

Mitigations:
- Reduce critical-section size (do less under the lock)
- Shard the lock (one lock per partition, hash the key)
- Use lock-free structures where possible (`queue.Queue` is well-tuned)

### 13.8 `logging` inside signal handlers

Python `logging` uses locks. If a signal fires while a thread holds the logging lock, and the handler also tries to log, you deadlock. Signal handlers should do as little as possible — set a flag, write to an `os.write(fd, ...)` self-pipe, and return. Real handling happens on the main thread later.

### 13.9 Forgetting that `print()` is not atomic across threads

`print("a", "b")` involves multiple syscalls. Two threads printing simultaneously interleave. Use a `Lock`-wrapped logger, not `print`.

### 13.10 `Thread.join()` without timeout in shutdown paths

A wedged thread + `join()` without timeout = your shutdown hangs forever, the orchestrator SIGKILLs you, and you lose in-flight state you could have flushed. Always join with a deadline.

### 13.11 Re-entrant locks hiding design problems

If you're reaching for `RLock`, ask: is the same call stack acquiring this lock twice? Why? Often it means a public method and a private method both lock — the cleaner fix is to have private methods *assume* the lock is held (and document it).

### 13.12 Using threads for fairness ordering

Thread scheduling order is not FIFO. If you submit A, B, C to a pool, they may complete C, A, B. If order matters, use a `PriorityQueue` with sequence numbers, not thread-submission order.

---

<a id="14-decision-framework"></a>
## 14. Threading vs Multiprocessing vs Asyncio — Decision Framework

The framework I'd present at a tech-strategy review:

```
┌─────────────────────────────────────────────────────────────────────┐
│ Q1: Is the workload CPU-bound (no GIL release) or I/O-bound?        │
└─────────────────────────────────────────────────────────────────────┘
        │                                 │
   I/O-bound                          CPU-bound
        │                                 │
        ▼                                 ▼
┌────────────────────────┐    ┌────────────────────────────────────┐
│ Q2: How many concurrent│    │ Q5: Does the C extension you use   │
│ tasks at peak?         │    │     release the GIL? (NumPy, Pillow,│
└────────────────────────┘    │     hashlib, lxml, polars)          │
   │           │              └────────────────────────────────────┘
  ≤1k        ≥10k                 │                       │
   │           │                 yes                     no
   ▼           ▼                  │                       │
THREADING  ASYNCIO                ▼                       ▼
                              THREADING               MULTIPROCESSING
                              (or 3.13 free-           or 3.12 sub-
                               threaded)               interpreters
                                                       or rewrite

Q3 (threading branch): does the existing codebase use sync libraries?
   yes → THREADING. asyncio rewrite is rarely worth the cost.
   no  → ASYNCIO is more efficient at scale.

Q4 (asyncio branch): do you need to call sync libraries?
   yes → run them in a `loop.run_in_executor()` thread pool
         — i.e., asyncio + threading, hybrid.
```

### Specific guidance

- **Hybrid asyncio + thread pool** for I/O is the modern default for *new* services. `asyncio` for the network layer, `loop.run_in_executor(...)` for the few sync libraries you can't avoid (legacy DB drivers, file I/O on most platforms, anything that doesn't have an async equivalent).
- **`multiprocessing.Pool`** for embarrassingly parallel CPU work (image processing, ML preprocessing, data ETL on Pandas frames). Watch for serialization cost — `pickle` round-trips a 1 GB DataFrame are slower than the work.
- **Subinterpreters / free-threaded** are still maturing. Reach for them when other options have failed and you've measured.
- **Native rewrite (Cython, Rust via PyO3, Go)** for the hot path is often the right answer when CPU is the bottleneck. Don't be precious about Python-only.

---

<a id="15-modern-python"></a>
## 15. Modern Python: Sub-interpreters (3.12) and Free-Threaded Mode (3.13+)

### Sub-interpreters (PEP 684, 3.12)

Each sub-interpreter has its own GIL. You get true parallelism *between interpreters*. Memory is mostly isolated (some sharing for immortal objects). Communication is via channels (PEP 554, still provisional).

The current state (as of early 2026):
- `interpreters` module ships in 3.13's stdlib (was `_xxsubinterpreters` in 3.12)
- Channel API is workable but not as ergonomic as `multiprocessing.Queue`
- C extensions need to be sub-interpreter-safe (many aren't yet — NumPy support landed in 1.26)

**When to consider it today:** you need parallel CPU and the memory cost of `multiprocessing` is too high (because each process duplicates code, libraries, model weights, etc.). Sub-interpreters share the binary code segment.

### Free-threaded build (PEP 703, 3.13 experimental, 3.14 default-on)

Removes the GIL entirely. Single interpreter, true thread parallelism for pure Python. Trade-offs:
- All your locking assumptions need to be correct now — code that "worked because of the GIL" can race
- ~10–15% slower single-threaded performance currently (improving)
- C extensions need `Py_mod_gil = Py_MOD_GIL_NOT_USED` declaration; many don't yet
- `dict`, `list`, `set` operations got new internal locking — atomic single-op behavior is preserved, but correctness across multiple ops still requires user locks (just like in any other language)

**When to consider it today:** you have CPU-bound Python that you can't easily move to NumPy/multiprocess, and you're willing to live on the bleeding edge. **For most production code in 2026, classic CPython + threading-for-I/O + multiprocessing-for-CPU is still the right default.**

---

<a id="16-testing-observability"></a>
## 16. Testing, Observability, and Debugging

### Testing

- **Don't write tests that rely on `time.sleep()` for synchronization.** They are flaky on slow CI. Use `Event`s, condition variables, or `threading.Barrier` to coordinate.
- **Force schedule interleavings** by setting `sys.setswitchinterval(0.000001)` in test setup. This dramatically increases the rate of context switches and surfaces races.
- **Stress-loop tests**: `for _ in range(1000): test()` for any test that exercises threading. Most race conditions are 1-in-1000 timing windows.
- **Static analysis**: `mypy` is limited but `pyright` catches some thread-safety hints. There's no Python equivalent to Go's `-race` flag — you compensate with stress + careful code review.

### Observability

- **Name your threads.** `threading.current_thread().name` shows up in py-spy, in stack traces, in logs. Free signal-to-noise.
- **`py-spy dump --pid <pid>`** dumps every thread's Python stack. Indispensable when a process is wedged. No need to install a debugger; py-spy attaches non-invasively.
- **`faulthandler.dump_traceback_later(60, repeat=True)`** in production: every 60 seconds, dump all thread stacks to stderr. Cheap, catches deadlocks in real-time.
- **Per-thread metrics**: count active threads, queue depths, lock wait times. Prometheus + custom collectors. Without these, you fly blind under load.
- **Structured logging with thread context**: `%(threadName)s` in the log format string at minimum.

### Debugging deadlocks

1. `py-spy dump --pid <PID>` — see all stacks. Two threads each waiting on `acquire()`? Cross-locking deadlock.
2. `gdb -p <PID>` then `py-bt` (with python-debug installed) — same, lower-level.
3. `faulthandler.register(signal.SIGUSR1)` — send `kill -USR1 <pid>` to dump stacks on demand.

### Debugging "why is this slow"

- `cProfile` aggregates across threads — useful for total CPU but hides per-thread structure.
- `pyinstrument` has thread-aware modes.
- `py-spy record --threads` produces a per-thread flamegraph. Shows GIL contention as fat "wait" frames.

---

<a id="17-cheat-sheet"></a>
## 17. Cheat Sheet

| Need | Use |
|---|---|
| Concurrent I/O, ≤1k concurrency, sync libs | `ThreadPoolExecutor` |
| Concurrent I/O, ≥10k concurrency | `asyncio` |
| CPU-bound pure Python | `multiprocessing.Pool` |
| CPU-bound NumPy/Pandas/Polars | `ThreadPoolExecutor` (releases GIL) |
| Producer-consumer | `queue.Queue` (bounded) |
| Wait until condition true | `Condition` + `while predicate:` |
| Shutdown signal | `Event` |
| Limit concurrent resource use | `BoundedSemaphore` |
| Connection pool | `BoundedSemaphore` + `Queue` |
| Per-thread state | `threading.local()` (reset between pool tasks!) |
| Cache miss thundering herd | Single-flight pattern (§10) |
| Rate limit (in-process) | Token bucket (§11) |
| Rate limit (distributed) | Redis + Lua, *not* in-process |
| Atomic counter | `itertools.count()` or `Lock` |
| Job worker with redelivery | `at-least-once + idempotent handler` |
| Aggregate fan-out | One process-wide pool, single deadline |

### Numbers a staff engineer should know

- Python thread default stack: **8 MB** (Linux), tunable via `threading.stack_size()`
- Default GIL switch interval: **5 ms** (`sys.getswitchinterval()`)
- `ThreadPoolExecutor` default workers: `min(32, os.cpu_count() + 4)` (3.8+)
- Practical thread count ceiling for a single Python process: **~5,000** before you're better off with asyncio
- CPython source: `Python/ceval_gil.c` — read it once. It's only ~700 lines and it'll change how you think about Python concurrency forever.

---

## Closing thought

Threading in Python is a tool for **overlapping waiting**, with sharp edges. Its strengths are simplicity, ecosystem compatibility, and shared memory. Its weaknesses are the GIL for CPU work and the OS-thread cost ceiling for very high concurrency.

A staff engineer's job here isn't to memorize APIs — it's to:
1. Recognize the workload's blocking profile within the first 5 minutes of a design discussion.
2. Pick the right concurrency model for the trade-offs that matter (memory, latency, ops complexity, team skill).
3. Identify the 2–3 patterns (single-flight, bounded queues, graceful drain) that compound across many problems and reach for them by reflex.
4. Know where the bodies are buried (GIL semantics, lock convoys, signal handling, fork+threads) so you don't have to learn them by outage.

Everything else is library lookups.