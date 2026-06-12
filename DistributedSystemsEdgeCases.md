list of all edge cases sceanrios that arise in distributed systems across industry, at the staff level.

⏺ Distributed Systems Edge Cases — Staff-Level Reference

1. Network Partitions & Partial Failures

- Asymmetric partition: Node A can reach B, but B cannot reach A. Heartbeats succeed one-way, causing split-brain where both sides believe the other is down.
- Partial partition: Node A can reach B and C, but B cannot reach C. Consensus protocols (Raft/Paxos) may still form a quorum that excludes a reachable node, causing confusing "I'm
  healthy but rejected" states.
- Grey failure / limp mode: A node is reachable but degraded — 50% packet loss, 2s latency spikes. Health checks pass, load balancer keeps routing, but every request through that node
  is slow. Harder to detect than full failure.
- Network partition heals mid-transaction: A two-phase commit coordinator sends PREPARE, partition occurs, coordinator times out and aborts, partition heals, and participants receive
  the late PREPARE — now they have stale/conflicting state.
- DNS partition: Nodes can reach each other by IP but DNS resolution fails. Service discovery breaks while the physical network is fine.
- One-way network degradation: Outbound packets work fine, but ACKs are lost. TCP retransmits pile up, connections appear hung, timeouts fire inconsistently.

2. Clock & Time

- Clock skew across nodes: Node A's clock is 500ms ahead of Node B. Timestamp-based conflict resolution (last-write-wins) silently drops Node B's writes even when they're causally
  later.
- Leap second: The clock goes from 23:59:59 to 23:59:60 (or Google-style "smears" it over hours). Interval calculations break, negative durations appear, rate limiters either block or
  permit double traffic.
- NTP step correction: NTP daemon jumps clock backward by 3 seconds. Monotonic-clock-unaware code creates log entries out of order, cache TTLs expire prematurely or extend, and
  sleep(5s) actually sleeps 8s.
- Time-of-check to time-of-use (TOCTOU) across nodes: You check a lock's expiry on node A, but by the time you act on it, node B's slightly-ahead clock has already expired and
  re-acquired the lock.
- Garbage collection pauses masquerading as time jumps: A 10-second GC pause means the process "wakes up" and thinks 10 seconds passed instantly. Lease-based locks expire during the
  pause, but the holder doesn't know.
- TrueTime unavailability (Spanner-like systems): If the uncertainty interval grows large, commits must wait longer, causing latency spikes. Systems that assume bounded clock
  uncertainty degrade when that bound is violated.
- DST / timezone transitions: Scheduled jobs run twice or not at all during "fall back" / "spring forward". A cron job at 2:30 AM doesn't exist on the day clocks jump from 2:00 to 3:00.

3. Consensus & Coordination

- Split brain during leader election: Old leader hasn't realized it's been deposed (its lease expired during a GC pause). It continues accepting writes while the new leader also accepts
  writes. Fencing tokens solve this — but only if every downstream system checks them.
- Quorum achieved but not durable: A write is acknowledged after 2-of-3 replicas respond, but both acknowledgments were buffered in OS page cache, not fsynced. Power failure loses the
  "committed" write.
- Phantom reads in consensus: A linearizable read sees value X, then a concurrent reconfiguration changes the quorum membership, and a second read (hitting the new quorum) sees a stale
  value Y < X.
- Membership change during consensus (Raft joint consensus): Adding/removing nodes to a cluster mid-operation. If done naively, two independent majorities can form temporarily.
- Pre-vote protocol failure: Raft pre-vote prevents disruption from partitioned nodes, but if all nodes enable pre-vote simultaneously during a rolling upgrade, the partially-upgraded
  cluster can deadlock elections.
- Witness replica promotion: A witness (non-voting, log-only) replica is promoted to voter during an emergency, but it has a truncated log. It votes for a candidate missing recent
  entries.

4. Replication & Consistency

- Replication lag reads: Write to primary, immediately read from replica — get stale data. The user sees their own write disappear. "Read-your-writes" consistency isn't guaranteed by
  default in most systems.
- Causality violation in multi-leader: User A sends message "What time?" in datacenter-1. User B reads it in datacenter-2, replies "3 PM". Replication lag causes datacenter-3 to see "3
  PM" before "What time?" — the answer appears before the question.
- Conflict resolution divergence: Two replicas apply conflicting writes. LWW resolves them differently due to clock skew. Now replicas have permanently divergent state with no mechanism
  to detect it.
- Tombstone resurrection: A deleted record's tombstone is garbage-collected. A replica that was offline during the delete comes back and replicates the record — the "ghost" record
  reappears.
- Replication slot / WAL retention bloat: A replica goes down for hours. The primary retains WAL/binlog segments waiting for it, filling the disk. Primary crashes from OOM or disk full
  — now both are down.
- Schema change replication: A DDL change (ALTER TABLE) on the primary causes a full table lock on the replica (MySQL < 8.0). Replication stalls, lag grows unboundedly.
- Auto-increment collision in multi-leader: Two primaries both assign ID 42 to different rows. Foreign keys, unique constraints, or application logic downstream now have silent
  corruption.
- Read-after-write across partitions: A multi-partition transaction writes to partitions A and B. A reader hits partition A (committed) and partition B (not yet committed). It sees an
  inconsistent snapshot.

5. Distributed Transactions

- Coordinator crash during 2PC: Participants have voted YES (prepared) and are waiting for COMMIT/ABORT. The coordinator crashes. Participants hold locks indefinitely — "blocking"
  problem of 2PC.
- Heuristic transaction resolution: An operator manually resolves the above by forcing a COMMIT on one participant and ABORT on another. The database is now permanently inconsistent,
  and the only fix is manual reconciliation.
- Saga compensation failure: Step 3 of a saga fails. Compensation for step 2 also fails (the external service is down). The system is in a partially compensated state with no automatic
  recovery path.
- Idempotency key collision: Two logically different requests happen to share the same idempotency key (reuse bug, hash collision). The second request silently returns the first
  request's response.
- Idempotency key replay after partial success: A request with key K partially succeeds (payment captured, but inventory not decremented). Retry with key K hits the idempotency cache
  and returns "success" without completing the inventory step.
- Distributed deadlock: Service A holds lock X, waiting on service B's lock Y. Service B holds lock Y, waiting on service A's lock X. No single system sees the cycle. Unlike database
  deadlocks, there's no global deadlock detector.

6. Message Queues & Event Streaming

- Exactly-once delivery myth: Consumer processes a message, commits offset, crashes before persisting the side effect. Message is not redelivered. The side effect is lost. "Exactly
  once" is actually "at-least-once + idempotent consumers."
- Poison pill / poison message: A malformed message causes the consumer to crash on deserialization. Consumer restarts, re-reads the message, crashes again. Infinite crash loop blocks
  the entire partition.
- Consumer rebalance storm: A slow consumer triggers a rebalance (exceeds max.poll.interval.ms). Rebalance causes other consumers to temporarily pause. The additional pause makes more
  consumers slow, triggering cascading rebalances.
- Offset commit race: Consumer reads message at offset 10, processes it, then reads offset 11. Offset 11 processing finishes first and commits offset 11. Consumer crashes. Message 10 is
  never reprocessed because the committed offset is 11.
- Partition ordering violation: Events for entity X are published to partition 3 (by hash), then the topic is repartitioned to 6 partitions. Entity X's events now go to partition 1.
  Historical events are on partition 3, new events on partition 1 — ordering guarantees are broken for consumers reading both.
- Compacted topic key collision: Two logically different entities hash to the same key in a compacted topic. Compaction retains only the latest — the other entity's event is silently
  deleted.
- Dead letter queue overflow: DLQ accumulates unprocessable messages. No one monitors it. Eventually it becomes a source of data loss, compliance violations, or storage exhaustion.
  - Message ordering vs. retry: A message fails and is retried later. Meanwhile, subsequent messages for the same key are processed. The retry succeeds and overwrites the newer state with
    old state.

7. Caching

- Thundering herd / cache stampede: A hot key expires. 10,000 concurrent requests all miss the cache simultaneously, all hit the database, all try to populate the cache. Database
  buckles under sudden load.
- Stale cache after write (race condition): Thread 1 writes DB, invalidates cache. Thread 2 (slower query, started before Thread 1) reads old DB value, writes it to cache. Cache now has
  stale data indefinitely until next TTL.
- Double-delete race: You use "delete cache → update DB → delete cache again" pattern. But between the two deletes, a concurrent read repopulates cache from a replica with replication
  lag. The second delete hasn't fired yet. Cache is stale.
- Hot key concentration: A single cache key gets millions of reads per second (celebrity profile, flash sale item). Even if the cache is distributed, that single shard is overloaded
  while others are idle.
- Cache inconsistency across nodes: Local in-process caches on nodes A and B expire at different times. For a brief window, A serves new data and B serves old data. Requests that hit
  both (via different microservices) see inconsistency.
- Negative caching of transient failure: A cache-aside read fails (database timeout). The code caches the empty result. Now the key returns empty for the entire TTL, even though the
  data exists.
- Serialization format change: You deploy a new version that serializes cache values in a new format. Old-format entries in the cache cause deserialization failures. Rolling deploys
  mean old and new code coexist, making this intermittent.

8. Load Balancing & Routing

- Connection pooling imbalance after scaling: A new backend instance starts. Existing long-lived connections remain on old instances. The new instance gets almost no traffic. Load is
  imbalanced until old connections close (which may be never for keep-alive/gRPC).
- Retry amplification (retry storm): Service A retries failed requests to service B. Service B retries to service C. A single failure at C causes retries_A × retries_B × retries_C total
  requests. 3 retries at each layer = 27x amplification.
- Health check false positive: The health endpoint returns 200, but the service can't actually serve requests (database connection pool exhausted, dependent service down). Load balancer
  keeps routing to a broken instance.
- Hash ring hotspot after node removal: Consistent hashing distributes keys. A node goes down; its keys are redistributed to the next node on the ring. If the failed node held a
  disproportionate number of hot keys, the successor node is now overloaded and cascades.
- Subsetting + correlated failure: Each client connects to a subset of backends. If the subset is chosen deterministically (e.g., hash-based), a single backend failure affects all
  clients in that subset — correlated blast radius.
- DNS TTL caching beyond control: An upstream DNS record changes (failover to DR). But intermediate resolvers, JVM DNS caching (networkaddress.cache.ttl), or OS-level caching hold the
  old IP for minutes to hours. Traffic still routes to the dead endpoint.
- Head-of-line blocking in HTTP/2 multiplexing: A single slow stream on a multiplexed HTTP/2 connection blocks all other streams on that connection at the TCP level (TCP HOL blocking).
  Worst-case latency is bounded by the slowest concurrent request.

9. Data Storage & Durability

- Write amplification under compaction: An LSM-tree database (Cassandra, RocksDB) receives a burst of writes. Compaction kicks in, consuming all disk I/O. Read latency spikes by 10x
  because reads compete for the same I/O bandwidth.
- Fsync vs. write confirmation: Application calls write() and gets success. Data is in the OS page cache, not on disk. Power failure loses it. Only fsync() guarantees durability — and
  even then, some disks lie about fsyncing.
- RAID controller write-back cache with BBU failure: The RAID controller's battery backup unit fails silently. Write-back cache is no longer protected. Power outage loses writes that
  the application believed were durable.
- SSD wear-out cliff: SSDs perform well until they hit their write endurance limit. Performance doesn't degrade gradually — it falls off a cliff. One node in a cluster suddenly becomes
  100x slower, causing tail latency for any request that touches it.
- Backup corruption discovered during restore: Backups run daily without error. During a disaster recovery event, the restore fails — the backup process was silently producing corrupt
  snapshots for months. No one tested restores.
- Foreign key constraint in distributed databases: You enforce referential integrity across microservices via application logic. Service A creates a parent record. Service B creates a
  child pointing to it. But A's write hasn't replicated yet. The "foreign key" points to nothing.
- Large transaction log / WAL overflow: A long-running transaction pins the WAL. New transactions' WAL entries accumulate. Disk fills. New transactions fail with "no space left." The
  single long transaction has effectively DoS'd the database.
- Auto-vacuum / compaction priority inversion: PostgreSQL autovacuum can't keep up with update rate. Transaction ID wraparound protection kicks in, forcing a full-table vacuum that
  blocks writes. This can last hours on large tables.

10. Service Mesh & Microservice Interactions

- Cascading timeout: Service A timeout = 10s. A calls B (timeout 10s), which calls C (timeout 10s). If C is slow, B waits 10s, then A also waits 10s. But A's total time is already 10s
  by the time B times out, so A times out simultaneously. All three services' error budgets burn at once.
- Deadline propagation failure: Service A sets a 5-second deadline. It takes 2 seconds to reach service B. Service B should have 3 seconds left but doesn't propagate the deadline — it
  sets its own fresh 5-second timeout. The original caller has long since timed out, but work continues downstream, wasting resources.
- Distributed circuit breaker inconsistency: Service A has 10 instances, each with an independent circuit breaker to service B. Instance 1's breaker opens (saw failures). Instance 2's
  breaker is closed (hasn't seen enough failures yet). Traffic shifts to instance 2, which now overloads service B.
- Service mesh sidecar version skew: During a rolling upgrade of the service mesh (Istio/Envoy), some pods have sidecar v1.14, others v1.15. A protocol change in v1.15 causes v1.14
  sidecars to drop headers. mTLS renegotiation fails intermittently.
- Dependency cycle: Service A calls B, B calls C, C calls A for enrichment. Under load, all three services deadlock their thread pools waiting for each other.
- Polyglot serialization incompatibility: Service A (Go) serializes a protobuf with default values omitted. Service B (Java) deserializes and treats missing fields as null instead of
  default. Subtle data corruption.

11. Deployment & Rollout

- Rolling deploy observability gap: You deploy v2 canary to 1% of traffic. P99 latency increases by 200ms. But your metrics aggregate across all instances — the 1% signal is invisible
  in the aggregate. The canary auto-promotes. You've now deployed a regression to 100%.
- Database migration + old code: You add a NOT NULL column. During rolling deploy, old-code instances try to INSERT without the new column. Every write fails. You needed a three-phase
  migration: (1) add column as nullable, (2) deploy code that writes it, (3) add the NOT NULL constraint.
- Feature flag evaluation inconsistency: Feature flag service has replication lag. During a rollout, 30% of nodes see flag=ON and 70% see flag=OFF. A single user's requests,
  load-balanced across nodes, see inconsistent behavior within the same session.
- Blue-green deploy with in-flight requests: You switch the load balancer from blue to green. Existing TCP connections to blue are still active and processing long-running requests. You
  tear down blue — those requests fail. You need connection draining.
- Rollback with irreversible side effects: v2 sent emails/webhooks/Kafka messages. You rollback to v1. The side effects are already in the wild. Downstream systems have state that v1
  doesn't understand.
- Config drift during deploy: Deploy starts with config version 5. Mid-deploy, someone pushes config version 6. Half the fleet has v5, half has v6. The interaction between old-code+v6
  or new-code+v5 is untested.
- Shadow traffic mutation: Shadow/dark traffic is sent to a test environment. But the test environment shares a dependency (queue, database, external API) with production. Shadow
  traffic causes real mutations.

12. Rate Limiting & Backpressure

- Distributed rate limiter inconsistency: Rate limit of 100 req/s enforced locally per instance. With 10 instances, the effective limit is 1000 req/s. Autoscaling adds more instances,
  and the rate limit effectively disappears.
- Token bucket refill drift: Distributed rate limiter uses a token bucket in Redis. Multiple nodes simultaneously read the bucket, all see "tokens available," all proceed. The bucket
  goes negative. Actual rate exceeds the limit.
- Backpressure signal inversion: You add backpressure (reject requests when queue depth > N). Clients retry rejected requests. Now the queue has retries AND new requests. Queue depth
  grows faster. More rejections. More retries. Positive feedback loop.
- Rate limiting by the wrong dimension: You rate-limit by IP address. A corporate NAT puts 10,000 users behind one IP. Legitimate users are throttled. Meanwhile, an attacker with a
  botnet has thousands of unique IPs and is untouched.
- Goodput collapse under overload: At 80% utilization, P99 = 50ms. At 100% utilization, P99 = 5s. The system technically handles all requests but useful work (goodput) drops because
  most responses arrive after the client has already timed out and retried.
- Adaptive throttling oscillation: An auto-tuning rate limiter reduces the limit when latency is high, increases when latency is low. This creates oscillation: limit drops → traffic
  drops → latency improves → limit rises → traffic surges → latency spikes → repeat.

13. Distributed Locking

- Lock expiry during processing: Process acquires a lock with a 30-second TTL. Processing takes 35 seconds (due to GC, slow I/O, etc.). Lock expires at 30s. Another process acquires it.
  Both processes now operate concurrently on the shared resource.
- Redlock safety violation: A Redlock across 5 Redis nodes acquires the lock on 3 nodes (majority). One node's clock jumps forward, expiring the lock early. A second client acquires the
  lock on that node + 2 others = 3 nodes. Two clients hold the "lock" simultaneously.
- Lock starvation: A high-priority process continuously acquires and releases a lock. A low-priority process is perpetually contending but never wins the lock. Its requests pile up or
  time out.
- Lock-free algorithm ABA problem: A CAS (compare-and-swap) loop reads value A, computes a new value, and CAS'es expecting A. But the value changed A → B → A between the read and the
  CAS. CAS succeeds, but the semantic context is different. Memory was freed and reallocated.
- Lease-based lock with network asymmetry: Node A holds a lease and can send heartbeats. The response path is broken — A doesn't receive the lease renewal confirmation. A thinks the
  lease expired and releases the resource. B acquires. A receives the delayed confirmation and thinks it still has the lease.

14. Observability & Monitoring

- Metric aggregation masking: Average latency is 50ms. But the distribution is bimodal: 90% at 10ms and 10% at 410ms. The average hides a severe problem for 10% of users. P50 (10ms)
  also looks fine. Only P99 (410ms) reveals the issue.
- Alert fatigue / missing alert: Too many alerts fire. On-call ignores them. The real incident alert fires among 50 noisy alerts and is missed. Alternatively: no one set up an alert for
  the new critical path.
- Trace sampling bias: You sample 1% of traces. But errors only occur 0.01% of the time. Your sampling is unlikely to capture error traces. When you debug an incident, there are no
  traces to examine.
- Observer effect / Heisenbug: Adding detailed logging to debug an issue changes the timing. The race condition disappears. Remove logging; issue returns.
- Cardinality explosion: A metric label includes user_id or request_id. The time-series database now has millions of unique series. Query performance collapses. The monitoring system
  itself becomes a scalability problem.
- Monitoring dependency cycle: Your alerting depends on the same infrastructure it monitors. The database goes down → the alerting service (which queries the same database) also goes
  down → no alert fires.
- Log timestamp ordering: Distributed logs collected from multiple nodes arrive out of order. A grep for ERROR followed by the "next" log line gives you a log line from a different
  node, from a different request. Investigation follows the wrong trail.

15. Security in Distributed Systems

- Confused deputy across services: Service A has permission to write to S3 bucket X. Service B (untrusted) calls Service A with a request to write to bucket Y. Service A relays the
  request using its own credentials, writing to a bucket B shouldn't access.
- Token replay across regions: A JWT is issued in region US-EAST. Region EU-WEST has a 5-minute token revocation propagation delay. A compromised token is revoked in US-EAST but still
  accepted in EU-WEST for 5 minutes.
- Certificate rotation race: During mTLS cert rotation, node A gets the new cert but node B still has the old CA bundle. B rejects A's connections. This lasts until B gets the update —
  during which inter-service communication is partially broken.
- Secrets in environment variables after rotation: Secrets are rotated in the vault. But running processes still have the old secret in their environment. Without a restart or dynamic
  refresh, they use the old secret until they're redeployed — which might be weeks.
- Cross-tenant data leak in shared infrastructure: A caching layer doesn't include tenant_id in the cache key. Tenant A's data is cached, then served to Tenant B's request for the same
  logical resource.

16. Autoscaling & Resource Management

- Autoscale feedback loop: Latency increases → autoscaler adds instances → new instances do cold-start (load caches, warm JIT) → they're slower than existing instances → latency
  increases further → more instances added → resource exhaustion.
- Scale-down during traffic dip: A brief traffic dip triggers scale-down. Instances are terminated. Traffic returns 2 minutes later. Now you're under-provisioned and it takes 5 minutes
  to scale up. Users experience degraded service.
- JVM / runtime warmup: A new instance starts and immediately receives traffic. The JIT hasn't compiled hot paths yet. First 10,000 requests are 5x slower. These slow requests trigger
  more scaling. Cold instances add capacity that isn't actually warm.
- Resource starvation from noisy neighbor: In a shared Kubernetes cluster, a batch job on the same node consumes all CPU/IO. Your latency-sensitive service shares the node and becomes
  slow. Resource limits aren't set, or are set but burstable.
- Connection pool exhaustion: Each instance opens N connections to the database. Autoscaling adds 50 instances → 50N new connections. The database has a max_connections limit. All new
  instances fail to connect. The database rejects connections and existing instances start failing too.
- Memory leak masked by scaling: A slow memory leak causes instances to OOM after 24 hours. Autoscaler replaces them. Everything "looks fine" because individual instances are always
  young. But you're wasting resources and one day the leak rate exceeds the replacement rate.

17. Data Migration & Schema Evolution

- Dual-write inconsistency: Migrating from DB A to DB B by writing to both. Write to A succeeds, write to B fails. Now they're inconsistent. You need event sourcing or CDC, not dual
  writes.
- Backfill race condition: You backfill data from old system to new. During backfill, live traffic writes new data to old system. Backfill copies stale data over the new data. You need
  to backfill only records not modified after a cutover timestamp.
- Wire format versioning: Service A sends protobuf v2 with a new field. Service B still uses proto v1 and ignores the new field. B processes the message, modifies it, sends it back. The
  new field is silently stripped. Data loss.
- Zero-downtime migration intermediate state: Migrating a column from string to integer. During the migration window, some rows have strings, some have integers. Application code must
  handle both. If it doesn't, 500 errors for half the rows.
- Foreign key breaks during migration: Table A migrated to new system; table B still in old system. Joins that used to be database-level now require cross-system lookups.
  Previously-atomic operations are now distributed transactions.

18. Orchestration & Scheduling (Kubernetes, etc.)

- Pod eviction cascade: A node runs low on resources. kubelet evicts pods. Evicted pods reschedule to other nodes, making those nodes resource-constrained. Those nodes evict pods.
  Cascade continues until the cluster stabilizes (or doesn't).
- PodDisruptionBudget + node drain deadlock: PDB says "at least 2 of 3 replicas must be available." You drain a node hosting 1 of the 3 replicas. The pod can't be evicted because only 2
  would remain, but one of the remaining 2 is already unhealthy. Drain hangs indefinitely.
- Init container dependency on not-yet-ready service: Pod A's init container waits for Service B to be ready. Service B's init container waits for Service A. Both pods hang in Init
  state forever. Deadlock.
- Persistent volume reattachment delay: A node fails. The PV's detach takes 6 minutes (cloud API delay + force-detach timeout). The new pod can't mount the volume for 6 minutes.
  StatefulSet is unavailable.
- Resource request vs. limit mismatch: Requests = 100m CPU, limits = 4 CPU. Scheduler packs pods tightly (based on requests). All pods burst simultaneously. Node has 4 cores but 10 pods
  all want 4 CPU. Severe CPU throttling, massive tail latency.
- CrashLoopBackOff with exponential backoff: A pod crashes immediately. Kubernetes retries with exponential backoff up to 5 minutes. Fix is deployed. But the pod is in a 5-minute
  backoff window and won't restart for another 4 minutes.

19. Consistency Anomalies (Isolation-Level-Specific)

- Write skew: Two doctors are on call. Each checks "is the other on call?" (yes), then removes themselves. Both succeed. Now no one is on call. This is allowed under Snapshot Isolation
- Resource request vs. limit mismatch: Requests = 100m CPU, limits = 4 CPU. Scheduler packs pods tightly (based on requests). All pods burst simultaneously. Node has 4 cores but 10 pods
  all want 4 CPU. Severe CPU throttling, massive tail latency.
- CrashLoopBackOff with exponential backoff: A pod crashes immediately. Kubernetes retries with exponential backoff up to 5 minutes. Fix is deployed. But the pod is in a 5-minute
  backoff window and won't restart for another 4 minutes.

19. Consistency Anomalies (Isolation-Level-Specific)

- Write skew: Two doctors are on call. Each checks "is the other on call?" (yes), then removes themselves. Both succeed. Now no one is on call. This is allowed under Snapshot Isolation
  but prevented by Serializable.
- Read skew across shards: Transaction reads shard A at time T1, then reads shard B at time T2. Between T1 and T2, a cross-shard transaction modifies both. The reader sees a state that
  never existed: A-before and B-after.
- Phantom read in range queries: Transaction 1 reads "all accounts with balance > 1000" (gets 5 rows). Transaction 2 inserts a new account with balance 2000 and commits. Transaction 1
  re-reads: now gets 6 rows. The set of rows changed mid-transaction.
- Lost update: Two transactions read the same row, each adds 1 to a counter. Both read 5, both write 6. One update is lost. Requires SELECT FOR UPDATE or atomic increment.
- Causal reverse: In an eventually consistent system, node A sees event X then Y. Node B sees event Y then X. If Y causally depends on X (e.g., "delete the file" then "file deleted"),
  node B sees the effect before the cause.

20. Miscellaneous / Cross-Cutting

- Metastable failure: A system enters a self-sustaining failure mode. Example: overload causes retries, retries cause more overload. Even after the initial trigger is removed, the      
  system stays in the failure state because the retries are now the load.
- Correlated failure across "independent" replicas: Three replicas on three different machines — but all three machines share the same ToR switch, or the same power strip, or the same
  kernel version with a latent bug. A single event takes all three down.
- Capacity cliff: System works perfectly at 95% load. At 96%, a nonlinear effect (hash table resize, compaction, GC) kicks in and throughput drops to 20% of normal. There's no graceful
  degradation — it's a cliff.
- Large message / payload bomb: A single 100MB message enters a queue designed for 1KB messages. Every consumer that touches it OOMs. Dead-letter reprocessing also fails. The message is
  stuck in the system, blocking the partition.
- UUID collision in high-throughput systems: UUIDv4 has 2^122 bits of randomness. At 1 billion IDs/day, collision probability is negligible — unless the PRNG is poorly seeded (e.g., VMs
  cloned from the same snapshot share the same seed).
- Zombie process / stale registration: An instance crashes but its service registry entry remains (TTL hasn't expired). Clients route to the dead instance. Connections fail. The  
  "unhealthy but registered" window can be 30-90 seconds.
- Request amplification through fan-out: A single API call fans out to 100 backend shards. Each shard call has a 1% error rate. Probability of at least one failure: 63%. The API
  endpoint has a 63% error rate even though every individual component is at 99%.
- Semantic versioning break in internal APIs: A "minor" version bump changes the semantics of an enum value. Consumers that don't update interpret old meaning. No deserialization error
  — just wrong behavior.
- Graceful degradation that isn't: The system is designed to degrade gracefully by disabling feature X under load. But feature X was the rate limiter. Disabling it makes the overload
  worse.
- Upgrade ordering dependency: System has components A, B, C. A must be upgraded before B, B before C. Someone upgrades C first. C uses a new protocol that B (old version) doesn't
  understand. Silent data corruption.
- Entropy exhaustion: VMs running cryptographic operations drain /dev/random. Key generation blocks. TLS handshakes hang. System appears healthy but new connections stall.
- Ephemeral port exhaustion: A service opens and closes thousands of connections per second. Each closed connection enters TIME_WAIT for 60 seconds. The OS runs out of ephemeral ports.
  New connections fail with "address already in use."
- Boolean trap in configuration: A configuration flag enable_safety_check=false is deployed to one environment. It gets copied to production via a config management tool. A critical
  safety check is disabled in production for weeks before anyone notices.

  ---
This covers the major categories. Each of these has caused real production incidents at scale. The common thread: distributed systems fail in ways that are partial, non-deterministic,  
and emergent from the interaction of components that individually work correctly.