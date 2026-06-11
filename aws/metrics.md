# Observability Metrics: RED & USE Methods — Staff SWE Reference

## 1. Framework Overview

### RED Method (Tom Wilkie, Weaveworks) — Service Perspective
Applies to **request-driven services** (APIs, microservices, queues). Ask: "Is my service healthy for the caller?"

| Signal | What it measures | Typical unit |
|--------|-----------------|-------------|
| **Rate** | Throughput — how many requests per unit time | req/s, events/s |
| **Errors** | Proportion or count of failed requests | % or count/s |
| **Duration** | Latency distribution of completed requests | p50/p95/p99 ms |

> **Failure mode RED catches:** A service is receiving traffic and returning errors or responding slowly — invisible to infrastructure-level dashboards.

---

### USE Method (Brendan Gregg) — Resource Perspective
Applies to **hardware and OS-level resources** (CPU, memory, disk, network interfaces). Ask: "Is my resource exhausted?"

| Signal | What it measures | Typical unit |
|--------|-----------------|-------------|
| **Utilization** | % of time resource is busy doing work | 0–100% |
| **Saturation** | Work that cannot be served now (queue depth, wait) | queue len, wait time |
| **Errors** | Hardware/driver error events | count/s, cumulative |

> **Failure mode USE catches:** A resource is at or near its capacity ceiling — causing latency spikes before the error rate climbs.

---

### When to Use Which

| Scenario | Method |
|----------|--------|
| Debugging slow API endpoint | RED |
| CPU pinned at 100% on a node | USE |
| DB connection pool exhaustion | USE (saturation) + RED (duration) |
| Payment service returns 5xx | RED |
| Disk IOPS hitting hardware limit | USE |
| Message queue consumer falling behind | RED (rate/duration) + USE (saturation) |

---

## 2. RED Method — Deep Dive

### 2.1 Rate

**What to instrument:**
- HTTP request count via Prometheus `http_requests_total` counter
- gRPC `grpc_server_started_total`
- Kafka consumer `kafka_consumer_records_consumed_total`

**Useful derived metrics:**
```
# Requests per second (5m window)
rate(http_requests_total[5m])

# Traffic split by status class
sum by(status_code_class) (rate(http_requests_total[5m]))
```

**Alerting heuristic:** Alert on rate dropping below expected baseline (traffic drop often signals upstream issue or misrouted traffic), not just rate spikes.

---

### 2.2 Errors

**What to count as errors:**
- HTTP 5xx (server-side errors) — always errors
- HTTP 4xx — context-dependent (429 = rate-limited client, usually an error from reliability POV; 404 on a search endpoint may be legitimate)
- Timeouts at the caller — even if the server logs 200
- Business-logic errors returned inside 200 (e.g., `{"status":"failed"}`)

**Prometheus pattern:**
```
# Error rate as a ratio
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```

**SLO framing:** Google SRE defines error budget against error rate. A 99.9% availability SLO allows ~43 min/month of errors. Burn rate alerting fires when the error rate is consuming that budget too fast.

---

### 2.3 Duration

**Always use histograms, never averages in isolation.**

Why averages lie: if 99% of requests take 10 ms but 1% take 10 s, average ≈ 110 ms — the tail suffering is invisible.

```
# p95 latency
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# p99 latency
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
```

**Latency budget decomposition (useful at staff level):**
```
Total latency = network transit
              + load balancer overhead
              + service compute (CPU)
              + downstream call latency (DB, cache, external API)
              + serialization/deserialization
              + GC pauses (JVM/Go)
```
Instrument each segment with spans (distributed tracing) to pinpoint which contributes to p99 growth.

---

## 3. USE Method — Deep Dive

### 3.1 CPU

| Signal | Linux metric | Tool |
|--------|-------------|------|
| Utilization | `%usr + %sys` in `mpstat` | `top`, `htop`, Prometheus `node_cpu_seconds_total` |
| Saturation | Run queue length > CPU count | `vmstat r` column, `sar -q` |
| Errors | Machine check exceptions | `mcelog`, `dmesg` |

**Prometheus:**
```
# CPU utilization per core
1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (cpu)

# Run-queue saturation (load average normalized)
node_load1 / count without(cpu, mode)(node_cpu_seconds_total{mode="idle"})
```

**Saturation threshold:** Run queue > 2× CPU count means threads are consistently waiting. Latency increases even when utilization hasn't hit 100%.

---

### 3.2 Memory

| Signal | Linux metric |
|--------|-------------|
| Utilization | `MemUsed / MemTotal` |
| Saturation | Swap activity, major page faults, OOM kill events |
| Errors | ECC memory errors, `edac` driver events |

```
# Memory utilization
1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)

# Swap saturation
rate(node_vmstat_pswpout[5m]) > 0  # pages being swapped out = memory pressure
```

**Key insight:** A system can show 90% memory utilization without saturation if it's using page cache effectively. Watch swap I/O and major page faults, not just `MemUsed`.

---

### 3.3 Disk I/O

| Signal | Linux metric |
|--------|-------------|
| Utilization | `%util` in `iostat` (time disk was busy) |
| Saturation | `avgqu-sz` (average queue length), `await` (total wait including queue) vs `svctm` (service time) |
| Errors | I/O errors in `dmesg`, SMART failures |

```
# Disk utilization
rate(node_disk_io_time_seconds_total[5m])

# Disk saturation — queue depth
node_disk_io_time_weighted_seconds_total  # integrates queue depth × time
```

**Interpretation:** `await >> svctm` means requests are spending time waiting in the OS I/O scheduler queue — disk is saturated even if `%util` < 100% (because `%util` only tracks time when at least one request is in flight, not queue depth).

---

### 3.4 Network

| Signal | Metric |
|--------|--------|
| Utilization | bytes_in+out / interface capacity |
| Saturation | TX/RX drops, retransmits, socket recv-Q/send-Q overflows |
| Errors | CRC errors, carrier drops, hardware errors |

```
# Interface utilization (GbE = 1e9 bits)
rate(node_network_receive_bytes_total[5m]) * 8 / 1e9

# TCP retransmit rate (saturation proxy)
rate(node_netstat_Tcp_RetransSegs[5m])
```

---

## 4. Production Case Studies

---

### Case Study 1: Latency Spike with No Error Rate Change (RED)

**Symptom:** p99 API latency for `/checkout` spiked from 120 ms to 4.2 s. Error rate held at 0.1%. On-call gets paged by the latency SLO burn alert.

**Investigation path:**
1. **RED — Duration breakdown by endpoint:** `checkout` spiked; `/health` and `/catalog` unaffected → isolated to checkout service.
2. **Distributed trace on a slow request:** 3.9 s spent inside `payment-service.AuthorizePayment` RPC.
3. **RED on payment-service:** Rate unchanged. Error rate 0%. Duration p99 = 3.9 s. Confirmed: payment service is the culprit.
4. **USE on payment-service pod:** CPU utilization 12%, memory fine. No resource constraint.
5. **USE on the external payment gateway connection pool:** `http_client_pool_waiting` (custom metric) = 18 goroutines waiting for a connection. Pool max = 10.
6. **Root cause:** A deployment 30 min prior doubled the number of checkout replicas, doubling connections to the payment gateway. The gateway enforced a per-client connection limit, causing connection pool saturation (USE: saturation), which appeared as latency, not errors (RED: duration).
7. **Fix:** Increased gateway API key's connection quota; added pool saturation alerting.

**Key lesson:** Latency-only spikes with healthy error rate often point to **resource saturation** downstream, not bugs. USE method on connection pools, thread pools, and DB connection limits is required.

---

### Case Study 2: Cascading Failure from DB Connection Pool Exhaustion (USE + RED)

**Symptom:** At 14:32 UTC, 40% of HTTP requests to the monolith returned 500. Alert fired on RED error rate > 1%.

**Investigation:**
1. **RED — Error rate by endpoint:** All endpoints degraded simultaneously — suggests shared dependency.
2. **Logs:** `context deadline exceeded acquiring DB connection` — connection pool wait timeout.
3. **USE on PostgreSQL connection pool:**
    - Utilization: 100% (pool_size=100, active=100)
    - Saturation: `pgbouncer_cl_waiting` = 340 clients waiting
    - Errors: 0 hardware errors — pure software saturation
4. **Trace back to cause:** A slow background job started at 14:30 — a report generation query scanning 50M rows without an index, holding 80 connections open for 8+ minutes.
5. **Root cause:** Single long-running query starved the pool, causing a request avalanche of timeout errors (RED: errors ↑).
6. **Fix:**
    - Kill long-running query immediately
    - Add statement_timeout to background jobs (separate pool)
    - Add `pgbouncer_cl_waiting` alert at threshold > 20

**Key lesson:** Use saturation metrics on shared resources as **leading indicators** — they precede error rate increases by minutes, giving you time to act before user impact.

---

### Case Study 3: Ghost Traffic — Rate Drop as a Failure Signal (RED)

**Symptom:** No alerts fired. But a weekly business review noticed Monday's order count was 30% below the previous Monday.

**Investigation:**
1. **RED — Rate over time:** A deploy at 09:15 caused a 32% drop in `POST /orders` requests. Other endpoints unaffected.
2. **Hypothesis:** 4xx client error causing silent failure?
3. **RED — Error breakdown:** `POST /orders` returning 422 Unprocessable Entity for requests with `currency: "EUR"` — a new validation added in the deploy accidentally rejected all non-USD orders.
4. **No alert fired because:** Error rate stayed below SLO threshold (total order volume is small), and 422 was not categorized as a server error in the dashboard.

**Fix:**
- Add alerting on **rate drops** (> 20% below 7-day rolling baseline)
- Classify 422 as billable errors for business logic failures
- Canary deploys with traffic comparison before full rollout

**Key lesson:** RED Rate is not just a scale metric — a **rate drop is a failure signal**. Combine with 4xx error classification.

---

### Case Study 4: CPU Saturation Before Errors Appear (USE)

**Symptom:** During a flash sale, response times climbed gradually. No error SLO breach yet, but engineers wanted to understand headroom.

**Investigation:**
1. **USE — CPU utilization:** Average 82%, but individual cores hitting 97-100% (not visible in cluster average).
2. **USE — CPU saturation:** `node_load1 / vCPU_count` = 2.1 — run queue 2× CPU count, meaning every thread waited on average for one other thread before getting scheduled.
3. **RED — Duration:** p99 = 450 ms (SLO = 500 ms). Headroom = 50 ms. With utilization at 82% and saturation already present, any further traffic spike would breach the SLO.
4. **Capacity projection:** At current traffic growth rate, saturation would push latency above SLO in ~18 minutes.
5. **Action:** Triggered horizontal pod autoscaler manually; scaled from 8 to 14 pods. CPU utilization dropped to 55%, run queue normalized, p99 returned to 180 ms.

**Key lesson:** USE saturation is a **predictive signal** for future RED degradation. Alert on saturation before utilization reaches 100% — not after errors begin.

---

### Case Study 5: Memory Leak Causing GC-Induced Latency (USE + RED)

**Symptom:** A Java service showed gradually increasing p99 latency over 48 hours, correlating with pod restarts every ~24 hours.

**Investigation:**
1. **RED — Duration trend:** p99 grew from 90 ms to 780 ms linearly over 24 hours, resetting sharply after each pod restart.
2. **USE — Memory utilization:** Heap utilization climbed steadily 60% → 95% over the same window.
3. **USE — Memory saturation:** JVM GC pause time metric (`jvm_gc_pause_seconds`) correlated exactly with p99 spikes. At 95% heap, GC triggered full stop-the-world collections lasting 600-700 ms.
4. **USE — Errors:** OOMKilled events in k8s logs — pod killed when heap hit 100%.
5. **Root cause:** A cache inside the service held references to response objects without TTL eviction. Each request grew the cache by ~2 KB.
6. **Fix:** Add TTL eviction to cache; add `jvm_memory_used_bytes / jvm_memory_max_bytes > 0.85` alert.

**Key lesson:** **USE memory saturation (GC pressure) manifests as RED latency spikes** — not errors. The error (OOMKill) only appears at the extreme end. Fix the saturation signal early.

---

### Case Study 6: Network Saturation Causing Intermittent Packet Loss (USE)

**Symptom:** A real-time analytics pipeline showed intermittent 1–2 second processing gaps. RED metrics showed no errors; durations were bimodal — most events processed in 5 ms, occasional batches taking 1.8 s.

**Investigation:**
1. **USE — Network utilization:** Kafka broker NIC at 94% capacity during peak ingestion.
2. **USE — Network saturation:** `node_network_transmit_drop_total` incrementing — TX buffer drops at the NIC level.
3. **USE — Errors:** No hardware errors — pure software buffer overflow.
4. **Effect on RED:** Dropped TCP segments caused producer retransmits, stalling a full batch until TCP retransmit timer (1 s default) expired — explaining the bimodal latency distribution.
5. **Fix:** Migrated Kafka brokers to 10 GbE interfaces; tuned `net.core.wmem_max` and Kafka producer batch settings; added NIC utilization and drop-rate alerts.

**Key lesson:** NIC saturation (USE) causes **invisible TCP retransmit storms** that appear as latency outliers in application-layer metrics (RED). Always check NIC drops when debugging bimodal latency distributions.

---

### Case Study 7: Noisy Neighbor on Shared Infrastructure (USE)

**Symptom:** A search service in a multi-tenant Kubernetes cluster showed random p99 degradation every few hours, unrelated to its own traffic patterns.

**Investigation:**
1. **RED on search service:** Duration p99 spikes; rate and error rate flat. Random timing → external cause.
2. **USE on the underlying node's disk I/O:** `node_disk_io_time_weighted_seconds_total` showed I/O saturation coinciding exactly with search service latency spikes.
3. **USE — who is consuming disk?** `kubectl top pods --containers` + cAdvisor `container_fs_io_time_seconds_total` identified a batch ETL job on the same node performing 800 MB/s sequential writes, saturating the shared EBS volume.
4. **Root cause:** No resource limits on the ETL job's disk I/O. Kubernetes had no disk I/O QoS in place (unlike CPU/memory).
5. **Fix:**
    - Applied node affinity rules to separate ETL and latency-sensitive workloads onto different node pools
    - Added cgroups blkio limits to ETL pods
    - Instrumented per-container disk I/O as a standard dashboard panel

**Key lesson:** USE method must be applied **per workload**, not just per host. Shared-node saturation from a noisy neighbor never appears in the victim service's RED metrics.

---

### Case Study 8: Thundering Herd on Cache Expiry (RED + USE)

**Symptom:** Every 5 minutes at :00 and :30, the origin database received a 40× spike in query load, causing CPU saturation and latency spikes for 15–20 seconds.

**Investigation:**
1. **RED on DB proxy:** Rate spiked to 4,200 queries/s (baseline: 100 q/s) at :00/:30. Duration p99 jumped from 3 ms to 2.1 s. Error rate 0.3%.
2. **USE on DB CPU:** Utilization hit 100%, saturation run queue = 18 (server has 8 vCPUs).
3. **Pattern:** Timing exactly correlated with Redis cache TTL of 5 minutes (TTL set uniformly to `300s` at service startup).
4. **Root cause:** Cache stampede — all cache keys set at service restart with identical TTL expired simultaneously. Every cache miss hit the DB concurrently.
5. **Fix:**
    - Jitter TTL: `TTL = base_ttl + rand(0, base_ttl * 0.1)` — spread expiry over a window
    - Probabilistic early recomputation (XFetch algorithm): recompute cache before expiry proportional to recomputation cost
    - Add `cache_hit_rate` and `cache_miss_rate` as RED-equivalent metrics for the cache tier

**Key lesson:** **Coordinated expiry creates periodic USE saturation spikes** even when per-request behavior looks fine. Add jitter to all TTL-based caches.

---

## 5. Alerting Strategy

### SLO-Based Alerting (RED-Driven)

Prefer burn-rate alerts over static thresholds:

```yaml
# 1-hour burn rate (fast burn — critical)
# If error budget would exhaust in < 1 hour
alert: ErrorBudgetFastBurn
expr: |
  (
    rate(http_requests_total{status=~"5.."}[1h])
    /
    rate(http_requests_total[1h])
  ) > (14.4 * 0.001)  # 14.4× burn for 99.9% SLO
for: 2m

# 6-hour burn rate (slow burn — warning)
alert: ErrorBudgetSlowBurn
expr: |
  (
    rate(http_requests_total{status=~"5.."}[6h])
    /
    rate(http_requests_total[6h])
  ) > (6 * 0.001)
for: 15m
```

### USE-Driven Alerts (Before RED Degrades)

```yaml
# CPU saturation — alert before errors start
alert: NodeCPUSaturated
expr: node_load1 / count without(cpu, mode)(node_cpu_seconds_total{mode="idle"}) > 1.5
for: 10m

# Memory pressure
alert: NodeMemoryPressure
expr: 1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) > 0.90
for: 5m

# Connection pool saturation
alert: DBConnectionPoolSaturated
expr: pgbouncer_cl_waiting > 20
for: 2m
```

### Alert Priority Matrix

| Condition | Severity | Response time |
|-----------|----------|---------------|
| Error rate burning SLO budget fast (>14.4×) | Page immediately | < 5 min |
| p99 latency > SLO threshold | Page | < 15 min |
| Resource saturation (USE) | Ticket + watch | < 4 hours |
| Error budget < 10% remaining | Ticket | Next sprint |
| Rate anomaly (> 3σ deviation) | Ticket | Next business day |

---

## 6. Golden Signals vs RED vs USE

Google SRE's **Four Golden Signals** subsume both frameworks:

| Golden Signal | Maps to |
|---------------|---------|
| Latency | RED: Duration |
| Traffic | RED: Rate |
| Errors | RED: Errors |
| Saturation | USE: Saturation (resource-level) |

USE adds **Utilization** and **Errors** at the resource layer, which the Golden Signals framework doesn't prescribe explicitly. In practice:
- Start with RED for services (user-facing impact)
- Drill into USE for the resource causing RED degradation
- Combine for root cause: "**what's degraded**" (RED) → "**why**" (USE)

---

## 7. Tooling Reference

| Layer | Tool | Key metrics |
|-------|------|-------------|
| Service (RED) | Prometheus + Grafana | `http_requests_total`, `http_request_duration_seconds` |
| Service (RED) | Datadog APM / New Relic | Automatic RED instrumentation per service |
| Tracing | Jaeger / Tempo / X-Ray | Latency breakdown by span |
| Resource (USE) | Prometheus node_exporter | CPU, memory, disk, network |
| Resource (USE) | cAdvisor | Per-container USE metrics in k8s |
| Resource (USE) | `iostat`, `vmstat`, `sar` | Ad hoc CLI investigation |
| DB (USE) | `pg_stat_activity`, pgbouncer | Connection pool saturation |
| Alerting | Alertmanager + PagerDuty | Multi-window burn rate rules |

---

## 8. Staff-Level Interview Angle

When asked about metrics/observability at staff level, structure answers as:

1. **Instrument at the right layer:** RED at every service boundary; USE at every shared resource.
2. **Prioritize by user impact:** RED SLO breach = user sees it. USE saturation = user will see it in N minutes.
3. **Alert on burn rate, not snapshots:** A 5% error rate for 1 minute is different from 0.1% sustained for 2 hours.
4. **Use tracing to bridge the gap:** RED tells you *what* is slow. USE tells you *which resource* is under pressure. Distributed traces tell you *exactly where* in the call graph.
5. **Capacity planning from USE:** Utilization trending toward 70% with saturation appearing at 80% → scale before you hit 90%.
6. **Cardinality discipline:** High-cardinality labels (user_id, request_id) on metrics cause Prometheus memory explosion — use traces for high-cardinality attribution, metrics for aggregation.
