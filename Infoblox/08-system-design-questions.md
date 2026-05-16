# 8 · System Design Questions — Tuned to Infoblox

Twelve realistic system-design prompts. For each: a quick clarifying-question pass, a sketch architecture, the key tradeoffs, and the failure modes you must surface. At staff level the interviewer cares more about *what you'd ask* and *what you'd defend* than about the diagram.

A general rubric to apply to every question:

1. **Clarify scope and scale** — read vs. write QPS, tenants, geo distribution, latency SLOs, durability targets.
2. **Sketch the dataflow first**, then the storage.
3. **State the consistency model** explicitly — strong / linearizable / read-your-writes / eventual.
4. **Name your failure modes** — what happens if the master dies, the network partitions, the cache empties, the queue backs up.
5. **Name your bottleneck**. Don't say "we can scale horizontally" without identifying which component first.
6. **Multi-tenancy** — if this is for a SaaS product, talk about isolation early.
7. **Observability** — "and we'd track these 4 metrics with alerts at these thresholds".

---

## Q1 · Design a Global Recursive DNS Resolver (à la 1.1.1.1)

**Clarifications**:
- Public service (any client) or enterprise (auth'd)? Affects abuse/scale.
- Latency SLO? p99 < 20 ms globally is the standard.
- DoH, DoT, DoQ support? Yes, probably all three.
- DNSSEC validation? Yes, validating resolver.
- Throughput? 1M QPS aggregate, 100K QPS per PoP.

**Architecture**:

- **Anycast** the service IP (e.g., `9.9.9.9`) from 100+ PoPs via BGP.
- Each PoP: a fleet of stateless resolver pods behind L4 LB (or XDP/eBPF kernel-bypass for top-end perf).
- Per-PoP **sharded LRU cache** keyed by `(qname, qtype, qclass, ECS-prefix)`. Each shard a goroutine + lock-free map.
- Cache miss: standard iterative recursion, with **prefetch** for popular entries before TTL expiry.
- **DNSSEC validator** in-process; trust anchor = root KSK.
- DoT/DoH/DoQ: TLS termination at the edge; same internal cache shared with UDP/TCP.

**Tradeoffs**:
- Anycast vs. unicast + GeoDNS: anycast is simpler, BGP handles failover; GeoDNS lets you split traffic by region but has caching issues.
- Per-PoP cache vs. globally shared: a global cache (e.g., Redis cluster) cuts upstream load but adds latency and is a SPOF. Most operators do per-PoP local + shared "tier-2" cache.
- DNSSEC validation cost: real but small; cache the validation result with the answer.

**Failure modes**:
- A PoP fails → BGP withdrawal, anycast routes traffic elsewhere.
- Authoritative server for `.com` is slow → falls back to alternative roots; TTL keeps users served.
- Cache poisoning attempt → DNSSEC blocks, EDNS source-port + 0x20 mitigate non-DNSSEC.
- DDoS amplification (others reflecting through us) → rate limit per source IP; serve only the trusted ECS sources.

**Numbers**: 1M QPS × ~100 bytes ≈ 100 MB/sec ingest; with cache-hit 85%, upstream ≈ 150K QPS. Cache size for ~10M entries × 1 KB ≈ 10 GB per PoP — fits in memory comfortably.

---

## Q2 · Design a Distributed DHCP Service for an ISP

**Clarifications**:
- Subscribers? 10M.
- Geographic distribution? 100 PoPs.
- Lease time? 24h.
- Static vs. dynamic? Mostly dynamic with reservation by option 82.

**Architecture**:

- **Per-region DHCP cluster** — 2–4 Kea servers per region in **load-balancing HA** mode.
- **Shared lease state per region** via MySQL/Postgres or Kea's in-built HA sync.
- **Anycast** the DHCP relay target IP — though most ISPs hard-code helper-addresses on CMTSes, so this is optional.
- **Lease backend partitioning** by option-82 circuit ID, so leases stay local to their region's DB.
- **Option-82 → RADIUS** policy lookup for which pool to use (rate plan, type of customer).
- **DDNS** updates to per-region authoritative DNS via TSIG-signed `RFC 2136` updates.
- **Audit pipeline** — every DORA → Kafka → ClickHouse.

**Tradeoffs**:
- Centralized lease DB simplifies failover but creates a regional blast radius; you accept that.
- Hot-standby vs. load-balancing: load-balancing doubles throughput but partitioning has corner cases on partner-down.
- RADIUS round-trip adds latency on every grant — cache subscriber profile aggressively, TTL ≤ 5 min.

**Failure modes**:
- DB master fails → standby promotion (5–30s); during failover, Kea queues requests up to a configurable depth.
- Both Kea HA peers partition from DB → enter "partner-down" → both serve from local pool → reconcile on rejoin (rare but real, document recovery).
- Lease pool exhaustion → alert at 80%, page at 95%; auto-extend pool from adjacent block if IPAM allows.
- RADIUS outage → fall back to default profile; alert.

**Numbers**: 10M leases / 24h = ~115 ops/sec average, ~600/sec peak. Easy. Concurrency bottleneck is per-pool allocation, not aggregate throughput.

---

## Q3 · Design IPAM for Hybrid (On-Prem + AWS + GCP)

**Clarifications**:
- Source of truth? Infoblox IPAM is the master.
- Cloud objects flow in via discovery; on-prem and cloud allocations both managed.
- Scale? 10M IP allocations, 50K subnets.
- Latency? Internal users / automation; p95 < 500 ms is fine.

**Architecture**:

- **Core service**: IPAM API (gRPC + REST), Postgres for canonical state, Redis for read cache and per-subnet allocation locks.
- **Patricia trie** rebuilt in memory per shard; persisted as rows in Postgres (parent_id + cidr).
- **Cloud discovery agents** — workers running per cloud account; poll the cloud API every N minutes, reconcile to IPAM state.
- **Event ingress** — cloud event streams (AWS EventBridge, GCP Pub/Sub) for near-real-time updates.
- **Audit log** in Kafka → cold storage.
- **Search** via Elasticsearch / OpenSearch for free-text + tag queries.

**Tradeoffs**:
- Bi-directional sync vs. observe-only — start with cloud-as-read; allowing IPAM to push into cloud is brittle (Terraform usually owns it).
- Strong vs. eventual consistency — strong within a subnet (allocation locks), eventual across regions.
- Single DB vs. sharded — start single, shard by tenant when needed.

**Failure modes**:
- DB primary down → readonly failover; allocations blocked but lookups continue.
- Cloud API rate-limited → backoff, miss real-time accuracy, periodic full sweep fills gaps.
- Stale discovery data → reconciliation diff alerts on drift.

**Specifically test for**: "two engineers both ask for next-available /24 in the same /16 simultaneously" — use SELECT FOR UPDATE on the parent block, or an external lock with TTL.

---

## Q4 · Design a Threat-Intelligence Distribution Platform (TIDE)

**Clarifications**:
- Volume? Millions of new indicators per day.
- Subscribers? Tens of thousands of DNS appliances pulling feeds.
- Update latency? Some feeds real-time, some 1h refresh acceptable.
- Formats? STIX/TAXII, RPZ, custom JSON.

**Architecture**:

- **Ingest tier**: connectors per source (internal research, partners, customer submissions, ML pipeline). Normalize to common schema.
- **Curation tier**: deduplication, confidence scoring, expiry, FP-list scrubbing.
- **Storage**: Postgres + S3 for indicators; ClickHouse for time-series of indicator activity (when was a domain blocked, how often).
- **Distribution**:
  - REST API for batch fetch.
  - Kafka topics for streaming consumers.
  - RPZ feeds via DNS zone transfer (IXFR/TSIG).
  - CDN-cached blob downloads for huge feeds.
- **Per-customer policy filter** — high-confidence to all, medium based on opt-in.

**Tradeoffs**:
- Push vs. pull: pushing via Kafka gets to consumers fastest; pull via REST is more compatible with firewalled customers.
- Single global feed vs. per-tenant — almost always single global with tenant-side opt-in filters.

**Failure modes**:
- Bad indicator slips through (FP that takes down legit site) — must have **kill switch** to retract: publish a "negative" indicator that overrides, propagate within minutes. Document and rehearse.
- Distribution channel down → degraded protection, not protection failure (consumers have last known list).

---

## Q5 · Design Real-Time DNS Tunneling Detection

**Clarifications**:
- Query rate? 1M QPS fleet-wide.
- Detection latency target? Seconds.
- False positive tolerance? Very low (alerting on legit Microsoft AD traffic is unacceptable).

**Architecture**:

- **Ingress**: every DNS query event → Kafka, partitioned by `(tenant, source_ip)`.
- **Per-source aggregation window** (e.g., 60s sliding) in a Flink job:
  - query count
  - unique-domain count
  - mean label length
  - entropy of subdomain
  - ratio of unusual qtypes (TXT/NULL/CNAME-chains)
- **Scoring** — combination of statistical + ML model trained on labeled tunneling samples (iodine, dnscat2, ML-detected).
- **Decisioning** — score > threshold → emit alert event; correlate with asset inventory; auto-block at the recursive resolver via RPZ update.

**Tradeoffs**:
- Detect at the resolver (fast) vs. detect in batch (more context) → do both. Resolver-side blocks known patterns; batch catches subtler ones with cross-correlation.
- Window size: short windows miss slow-tunneling; long windows delay detection. Use multi-scale windows (1m, 5m, 1h).

**Failure modes**:
- Model drift → periodic retraining + monitored precision/recall.
- A noisy tenant generating false alerts → tenant-specific allowlists, per-source baselining.

---

## Q6 · Design a Multi-Region Configuration Push for 100K Customer Appliances

**Clarifications**:
- Devices? 100K NIOS-X hosts.
- Config size? 100 KB–10 MB per host.
- Update frequency? Every few minutes for some, on-demand for big changes.
- Networks? Many behind NAT, firewalls; outbound-only connections.

**Architecture**:

- Each NIOS-X opens a **long-lived mTLS gRPC stream** to the closest region of the cloud control plane.
- Control plane publishes config changes to **Kafka**, partitioned by `host_id`.
- A **per-region pusher service** consumes Kafka, fans out to the gRPC streams it owns.
- Hosts ACK applied config with version numbers; cloud reconciles desired vs. actual state.
- Periodic full-state reconciliation (every hour) to catch drift from missed events.

**Tradeoffs**:
- Push vs. pull: outbound-only constraint forces pull (long-lived stream is effectively pull-with-server-events).
- Config delivery contract: incremental deltas (smaller) vs. full snapshots (idempotent). Use snapshots with a delta layer; full snapshot on connect.
- Backpressure: if a host can't keep up, queue and drop oldest non-essential events; critical events (threat intel) get priority queue.

**Failure modes**:
- A region's pusher dies → host reconnects to neighbor region; possible config-version skew during cutover.
- A host disconnects for days → reconnects, downloads full snapshot, catches up.
- Bad config breaks fleet → staged rollout + automatic rollback on health degradation.

---

## Q7 · Design a Multi-Tenant DNS Analytics Platform

**Clarifications**:
- Tenants? 5K.
- Volume? 10B queries/day, 100B retained 90 days.
- Query patterns? Time-series rollups (top N domains, query-rate trends), forensic search (find all queries to `evil.example` last 30 days).
- Tenant isolation? Strict.

**Architecture**:

- **Ingest**: NIOS-X → telemetry endpoint → Kafka, partitioned by tenant.
- **Hot store**: ClickHouse cluster, table partitioned by day, sorted by `(tenant, timestamp, domain)`.
  - Pre-aggregated rollups: hourly top-domain, query-rate by view, NXDOMAIN rate, etc.
- **Cold store**: Parquet on S3, partition `tenant/yyyy/mm/dd/`, queryable via Trino.
- **Tenant query API**: routes tenant ID into every query; row-level filter; per-tenant rate limit.

**Tradeoffs**:
- Per-tenant ClickHouse cluster (clean isolation, costly) vs. shared cluster with row-level isolation (cheaper, weaker boundary). Most start shared, peel off whales.
- Columnar vs. document store: ClickHouse wins for time-series; ES for full-text. Possibly both.
- Pre-aggregation depth: more rollups = fast common queries, more storage and ingest complexity. Pick the top 10 queries you'll always run.

**Failure modes**:
- ClickHouse node fails → replica takes over; per-table replication keeps reads working.
- Kafka backs up → consumer lag alerts; scale out consumers; do not drop data silently.
- Noisy tenant scans terabytes → query-cost limits, kill long-running queries.

**Cardinality concerns**: tenants with many subdomains can blow up the dictionary. Use `lowCardinality(String)` selectively; encode high-cardinality fields with codecs.

---

## Q8 · Design a Global Anycast DNS for Authoritative Zones (CDN-style)

**Clarifications**:
- Zones served? 1M zones.
- QPS? 5M aggregate.
- Latency SLO? p99 < 30 ms globally.
- Updates? Customers update zones via API; propagation < 1 min.

**Architecture**:

- **Anycast** the authoritative service IP from 50 PoPs via BGP.
- **Per-PoP authoritative server** (e.g., NSD, Knot, or CoreDNS with `file`/`auto` plugins).
- **Zone-storage tier**: zones canonical in a control plane; pushed to PoPs.
- **Push mechanism**: signed zone snapshots via S3 + per-PoP fetcher, or DNS NOTIFY + IXFR from a hidden primary.
- **DNSSEC**: sign at the control plane; PoPs serve pre-signed RRSIGs.

**Tradeoffs**:
- Hidden primary + AXFR/IXFR vs. push from object store: AXFR is the standard, well-understood; object-store push is simpler at scale.
- On-the-fly DNSSEC signing vs. pre-signed: pre-signed is faster, but rotation is more involved; on-the-fly burns CPU per query.
- Per-zone TTL minimums to prevent customers from setting too-low TTLs that hammer your servers.

**Failure modes**:
- A PoP corrupts a zone after sync error → PoP serves stale; health checks should detect; failover withdraws BGP route.
- DDoS on a single zone → anycast spreads it geographically; PoP-level rate limit on per-zone basis.
- Customer publishes a bad zone (SERVFAIL'd by DNSSEC validators) → validate before publishing; offer a "publish but warn" mode.

---

## Q9 · Design a Subnet-Allocation API with Strict Concurrency Guarantees

**Clarifications**:
- "Find next available /N inside /M". Concurrent requests must never overlap.
- Throughput? 1K ops/sec, bursts to 10K.

**Architecture**:

- **Storage**: Postgres row per CIDR. Index on `(parent_id, start_ip, end_ip)`. GiST index on `cidr` for overlap queries.
- **Allocation flow** (in a single transaction):
  ```
  BEGIN;
  SELECT * FROM blocks WHERE id = $parent FOR UPDATE;
  -- compute next available /N by walking children
  INSERT INTO blocks (parent_id, cidr, ...) VALUES (...);
  COMMIT;
  ```
- **Optimization**: cache the "next-free hint" in Redis per parent block to skip the scan when contention is low. Validate against DB before commit.
- **Idempotency**: `Idempotency-Key` header maps to a 24h memo of the response — repeated POST returns the same allocation.

**Tradeoffs**:
- Per-parent lock (this design) vs. global lock (simpler, doesn't scale) vs. lock-free with CAS retries (complex, but scales).
- Reservation TTL: allow soft-allocate with 60s TTL so workflows can roll back without leaking IPs.

**Failure modes**:
- Client dies mid-transaction → DB releases lock; no allocation made.
- Network partition between client and DB → potential double-write on retry; idempotency key saves us.
- Cache says "next free is X" but DB has it allocated → CAS retry; invalidate cache entry.

---

## Q10 · Design Migration from NIOS Grid to BloxOne (the real Infoblox problem)

**Clarifications**:
- Customer's NIOS data: zones, fixed addresses, IPAM tree, user permissions.
- They want zero downtime, gradual migration, ability to roll back.
- Their automation uses NIOS WAPI.

**Architecture**:

- **Coexistence period**: NIOS Grid and BloxOne both operational; one is "primary" per resource.
- **Bridge service** in the cloud control plane:
  - Polls NIOS WAPI (or accepts NIOS webhooks) for changes.
  - Pushes mirrored data into BloxOne IPAM/DNS/DHCP services.
  - Bi-directional during dual-write phase.
- **WAPI compatibility shim** on BloxOne — exposes a NIOS-compatible REST API so existing automation works unchanged. Translates calls into native BloxOne ops.
- **Per-resource cutover** — a zone, subnet, or DHCP scope can be flipped one at a time, with metadata recording which side is primary.
- **Rollback** — keep the old data live in NIOS for N days post-cutover.

**Tradeoffs**:
- Dual-write conflicts: define a precedence rule (e.g., "BloxOne wins after cutover"); audit any divergence.
- WAPI shim coverage — must cover the calls customers actually make; build a usage telemetry layer in NIOS first to know what to support.
- Cost: running both for N months is expensive; communicate to customer and shape pricing.

**Failure modes**:
- Bridge falls behind → resources out of sync; alert; halt cutover until catchup.
- Shim semantic drift — a NIOS call has subtle behavior the shim mis-emulates; canary every new shim path with real customer traffic in parallel-validation mode.
- Customer rolls back after data has diverged → preserve audit log; offer reconciliation tool.

This is the highest-signal staff-level discussion at Infoblox. They live this problem.

---

## Q11 · Design a Rate Limiter for DNS Queries per Source

**Clarifications**:
- Per source IP / per tenant / per (source, qname) — usually all three layers.
- Throughput? 1M QPS through the system.
- Action when over limit? Drop or REFUSED response code.

**Architecture**:

- **Token bucket per (tenant, source_ip)** state in memory at each resolver.
- For multi-instance correctness, use a shared token-bucket via Redis with Lua atomic scripts.
- **Hierarchical limits**: per-source < per-tenant < per-region.
- **Sliding window** for analytics; **token bucket** for enforcement.
- **Burst**: allow short bursts above sustained rate.

**Tradeoffs**:
- Local-only bucket: fast but per-resolver enforcement is loose (a client with 4 connections to 4 resolvers gets 4× the limit). Acceptable for soft limits.
- Shared bucket: accurate but adds latency and a SPOF. Use only for hard caps.
- Algorithms: token bucket > leaky bucket for DNS (bursty), avoid fixed-window (boundary effects).

**Failure modes**:
- Redis down → local fallback bucket with conservative rate.
- Memory pressure from per-(IP, qname) buckets → coarsen to per-source only; sample for per-(IP, qname).

---

## Q12 · Design a System to Detect Rogue DHCP Servers on a Network

**Clarifications**:
- Network has VLANs; rogue server could be on any.
- Detection time? Minutes.
- Action? Alert + optionally block via L2 switch ACL push.

**Architecture**:

- **Sensor agents** on each VLAN (NIOS-X or dedicated probe).
- Each sensor periodically sends a DHCPDISCOVER (or passive-listens for broadcasts).
- Sensor logs every DHCPOFFER it sees. Each offer carries server-ID (option 54).
- Sensor reports to BloxOne: VLAN, source IP/MAC of offer, server-ID, options offered.
- Cloud correlator: compares observed server-IDs against the known DHCP server inventory in IPAM. Unknown server → alert.
- Optional: push ACL to switch (via NETCONF / RESTCONF) to block the rogue MAC.

**Tradeoffs**:
- Active probe vs. passive listen: active is reliable; passive depends on real client activity. Run both.
- False positives from authorized but unregistered servers — keep an allowlist editable by ops.

**Failure modes**:
- Sensor disconnected → blind on that VLAN; alert on sensor health.
- Adversary that mimics a legitimate server-ID → check MAC/L2 source.

---

## Cross-cutting answers to expect

- "What's your bottleneck?" Always have a confident answer with a number.
- "How would you observe this?" Name 3–5 SLI/SLO metrics and the alert thresholds.
- "What changes for 10× scale?" Pick the component that breaks first; explain the next move.
- "What if a region fails?" Articulate the blast radius and the recovery time.
- "How is this multi-tenant?" Bring it up before they ask.