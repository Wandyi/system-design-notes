staff level software engineer questions on etcd and it's features. give different realistic sceanrios and case studies

Same structure as before — scenario, question, what a strong answer covers, the trap. etcd separates seniors from staff at the operational layer (compaction, partitions, lease semantics) and the API-design layer (watches, leases, fencing) more than at the API-surface layer.

---
1. Sizing the cluster

You're standing up a new etcd cluster as the source of truth for a service registry. SRE asks: "how many nodes?"

Question: Pick a number, justify it.

Strong answer covers:
- Quorum is floor(N/2) + 1. Failure tolerance is floor((N-1)/2). So 3 tolerates 1, 5 tolerates 2, 7 tolerates 3.
- Even numbers buy nothing: 4 tolerates 1 (same as 3), 6 tolerates 2 (same as 5), but you pay more write-side network for the bigger quorum.
- 5 is the production sweet spot for most workloads. Survives a node failure plus a rolling restart (or one AZ down + one maintenance), and write quorum is 3 — small enough that latency stays bounded.
- 7 only if you genuinely need to survive 3 simultaneous failures, and you accept higher write latency (more peer fsyncs to wait for).
- Geographic placement: 5 nodes across 3 AZs (2-2-1). Losing the 2-node AZ still leaves 3 voters — quorum holds. Don't do 3 AZs with one node each only if your AZ availability matches the SLO.
- 1-node "cluster" is fine for dev/CI; never prod. No HA, no quorum, fsync failures lose data.

Trap: "More nodes = more durable." Past a point, more voters = slower writes (more fsyncs to wait on) and more inter-node traffic, without buying useful additional failure tolerance.

---
2. Distributed lock with a lease — the lease-loss problem

You implement a seat-hold lock on top of etcd: acquire by putting /locks/seat/123 with a lease (TTL=30s), background goroutine sends KeepAlive. Booking flow runs while holding the lock.

Question: Where's the bug, and what's the right design?

Strong answer covers:
- The lock owner can lose the lease without knowing it — GC pause, network partition between client and etcd majority, KeepAlive RPC stalled. Etcd's view: lease expired, key deleted, somebody else can acquire. Owner's view: "I still hold the lock." Now two holders.
- KeepAlive's response channel emits when the keepalive can't be maintained. The owner must watch that channel and stop work the moment it closes. Re-check with LeaseTimeToLive before any externally-visible action.
- The right primitive is fencing tokens (Kleppmann). etcd's mod_revision from the lock-acquire response IS a monotonic fencing token. Pass it downstream. The inventory DB rejects writes with a token lower than the highest seen for that resource.
- Bound work duration to "TTL minus safety margin" — don't start a 25s job with a 30s lease. Either chunk the work and re-check, or use a longer lease.
- clientv3/concurrency.Mutex packages most of this (lease + watch on predecessor + automatic release) but does not give you fencing — that's still on you at the downstream.

Trap: Believing the lock guarantees mutual exclusion of work. etcd guarantees mutual exclusion of key ownership at a given revision. The gap between those two is where outages live. Always combine an etcd lock with a fencing check at the system whose state actually matters.

---
3. Leader election — "Am I still the leader?"

A scheduled-job runner uses concurrency.Election to ensure exactly one instance runs cron jobs. The leader holds a 15s lease. A job takes 20s and writes to a downstream API.

Question: What can go wrong, and how do you design around it?

Strong answer covers:
- Same lease-loss class of bug. The leader's session can expire mid-job. Another candidate observes the campaign key delete, becomes leader, runs the same job.
- Election.Observe() and Session.Done() give signals — leader must select on ctx.Done() AND session.Done() while executing work. On either, abort immediately.
- Idempotency at the job is non-negotiable. Even with perfect leader election, retries happen, network blips happen. Every job should be safe to run twice (idempotency key, conditional writes).
- For longer jobs: chunk into shorter steps with leadership re-check between each. Each chunk records its own progress; on leader change, the new leader resumes from the recorded checkpoint.
- For "do this exactly once across the fleet, ever": leader election is necessary but not sufficient. You also need a durable record of "this job ran" written under conditional revision check (Txn().If(Compare(...)).Then(Put(...))).

Trap: Treating election as a guarantee instead of a hint. The election tells you "you probably are the leader." The actual safety property comes from idempotent operations and compare-and-swap at the data layer.

---
4. Watch storm and revision tracking

You build config distribution. 2,000 service instances each Watch("/config/"). During a feature flag rollout, config updates happen ~50/sec for 10 minutes.

Question: What blows up, and how do you design for it?

Strong answer covers:
- 50 events × 2,000 watchers = 100k watch deliveries/sec sustained. etcd handles this fine — watch events are multiplexed over the gRPC stream per client — but watchers' downstream handling becomes the bottleneck.
- Per-stream events are delivered in order. A slow handler stalls the stream. Buffer overruns disconnect the watcher with ErrCompacted if it falls too far behind compaction.
- Revision-based recovery is the contract: every watcher tracks last_processed_revision. On disconnect, reconnect with WithRev(last+1). Etcd replays missed events. If last < compactRevision, etcd returns ErrCompacted — watcher must do a full Get of current state (which returns a header.revision) and resume watch from there.
- Compaction interval and watcher disconnect-tolerance must be calibrated together. Aggressive compaction (5 min) + slow reconnect = forced full resyncs every blip.
- Coalescing: if your handler is idempotent and only cares about "current state," collapse rapid updates. Maintain a per-key debounce in the watcher.
- Server-side: tune --max-request-bytes for big payloads, --snapshot-count for snapshot frequency. For 2,000 watchers, ensure the leader has the file descriptors and CPU for the gRPC streams.

Trap: Treating watches as a reliable broadcast bus and never tracking revisions. The first network blip causes silent event loss, which surfaces hours later as "config is wrong on five instances."

---
5. The compaction + defrag + quota saga

Production etcd is at 7.2GB, approaching the default 8GB quota. Disk usage growing 250MB/day. Last week someone ran etcdctl compact and disk usage barely moved.

Question: Explain what's happening and write the runbook.

Strong answer covers:
- etcd's storage is MVCC — every key write creates a new revision; old revisions are retained until compacted. Compaction removes the logical old revisions but doesn't return space to the filesystem. The bbolt file stays the same size, with free pages internally.
- etcdctl defrag rewrites the bbolt file compactly and does return space — but it blocks the local node from serving traffic during the rewrite (can be tens of seconds on a multi-GB DB).
- Production runbook:
  a. Confirm auto-compaction is on: --auto-compaction-mode=periodic --auto-compaction-retention=1h (or revision mode). Default is OFF; many clusters silently never compact.
  b. Trigger compaction at the current revision if behind.
  c. Defrag one node at a time, never the leader first; wait for it to return to the cluster and catch up before moving to the next. For 5-node cluster, this is a 5-step rolling operation.
  d. If the cluster has tripped the quota already, you'll see a NOSPACE alarm. The cluster goes read-only. After defrag, etcdctl alarm disarm to clear it.
- Quota tuning: --quota-backend-bytes=16G for larger needs. But: etcd is a coordination store. If you're past 8GB of config-like data, you're using etcd as a database. Move bulk data elsewhere.
- Monitor: etcd_mvcc_db_total_size_in_bytes, etcd_mvcc_db_total_size_in_use_bytes. The gap between those is reclaimable via defrag.

Trap: Running defrag on all nodes simultaneously to "make it faster." Cluster loses quorum. Always rolling, leader last (or step down first).

---
6. Disaster recovery — restoring from snapshot

3-node etcd cluster underlying Kubernetes. Two disks corrupted simultaneously in a storage incident. The third node has data but won't serve writes — it's lost quorum.

Question: Recovery procedure?

Strong answer covers:
- With 2 of 3 nodes lost, quorum is gone. Even the remaining "healthy" node can't commit anything. This is the disaster recovery path, not a normal repair.
- Need a recent snapshot. etcdctl snapshot save should be running on cron, ideally every 15–60 min for Kubernetes, stored outside the cluster (S3/GCS). Snapshots include a hash; etcdctl snapshot status verifies integrity.
- Procedure:
  a. Stop the surviving node. Don't try to "promote" it — even if its data is newer than the snapshot, mixing snapshot restore with existing data leads to inconsistency.
  b. etcdctl snapshot restore <snapshot> --data-dir=/var/lib/etcd-new --initial-cluster=... on a fresh node. This generates a new cluster ID — important: members can't join an existing cluster after restore; you're forming a new cluster.
  c. Start the restored single-node cluster.
  d. Add the other two members (member add → start) so they replicate from the new leader.
  e. Repoint clients (Kubernetes API servers) at the new cluster.
- Data loss = whatever was written after the snapshot. For Kubernetes, that's recent pod state, recent leader leases, recent events. Workloads usually self-heal because controllers reconcile.
- Verify before re-opening writes: hash check, consistency check, run k8s control plane against the restored cluster in read-only mode for a few minutes.

Trap: Restoring on top of an existing data dir, or restoring different snapshots on different nodes. Different cluster IDs = nodes refuse to peer. Different snapshot timestamps = split brain at the application layer.

---
7. WAN-distributed etcd — why "just stretch it" doesn't work

Architecture review: someone proposes a single etcd cluster across us-east, eu-west, ap-south for HA. RTT between regions is 80–150ms.

Question: Push back with the right model.

Strong answer covers:
- Every etcd write is a Raft round-trip: leader proposes → majority of voters ack (with fsync) → commit. Stretched cluster means every write pays the worst inter-region RTT.
- 100ms RTT means 100ms minimum write latency, and that's the floor — at the tail you'll see 300ms+. Kubernetes API server cumulative latency becomes painful for controllers that do many writes per reconcile.
- Linearizable reads also do a ReadIndex through the leader — same RTT cost. Serializable reads from local replica skip this but can be stale.
- Failover during a partition: if us-east holds 3 of 5 voters and the trans-Atlantic link drops, eu-west and ap-south lose quorum. Writes stop in the minority side. That's by design (CP, not AP) — but operators are often surprised.
- The right pattern:
    - Regional etcd clusters per control plane. One Kubernetes cluster per region, each with its own etcd.
    - Cross-region replication at the application layer (multi-region database, async event replication), not at etcd.
    - Learner nodes (3.4+) can act as non-voting replicas — useful for read-fanout and faster promotion during failover, but they don't help write latency because they're not in the quorum.
- If you absolutely must have cross-region etcd: 3 nodes in primary region, 2 in secondary, accept the latency. On primary region loss, manually promote secondary members and accept the data divergence risk.

Trap: Thinking learner nodes solve the WAN problem. Learners receive replication but never vote, so writes still need a quorum of voters — exactly the latency floor you were trying to escape.

---
8. Lease loss during a network partition

Service A acquires a lock with a 30s lease and starts a long-running operation. The network partitions A from the etcd majority. From etcd's perspective, A's lease expires; service B acquires the same lock. A's view: still holding the lock.

Question: What's the right defense?

Strong answer covers:
- This is fundamental, not preventable at the lock-protocol layer. Asymmetric knowledge: A still thinks it owns the lock until its KeepAlive RPC fails, but by then etcd-and-B have moved on.
- The only correct mitigation is at the system A is writing to:
    - Fencing tokens. A receives mod_revision on lock acquire. Every write to the downstream system carries it. Downstream tracks max_token_seen per resource and rejects lower-token writes. B's writes carry a higher token; A's stale writes get rejected.
    - Conditional writes at the storage layer. Use the data store as the source of truth — UPDATE … WHERE held_by = $A AND held_until > NOW(). The lock becomes advisory.
- In application code, A must watch its session and abort fast on lease loss. concurrency.Session.Done() channel. Wrap any write in a "still leader?" check, but understand this is a race-narrowing measure, not a race elimination.
- Time-based safety margin: if your lease TTL is 30s and your minimum work unit is 10s, you have a 20s window where you might still attempt work after lease loss. Either shrink work units or accept the risk explicitly with fencing.

Trap: Believing KeepAlive plus "I'll exit if KeepAlive fails" is sufficient. Between the lease expiring on the server and KeepAlive detecting failure on the client, there's an unbounded window. Fencing tokens are the only reliable backstop.

---
9. Member changes — adding a node safely

You need to grow a 3-node cluster to 5 to improve availability. Naive approach: add member 4, wait, add member 5.

Question: What's the safe sequence and why?

Strong answer covers:
- Each member add to a voting member transiently changes the quorum requirement. Going from 3 → 4 voters changes quorum from 2 → 3. If the new member is slow to come up, writes need acks from 3 of 4, and you might temporarily have just 3 of 4 ready, which is fine — but if any existing member is also flaky, you can lose quorum.
- 3.4+: use learner members for adds. etcdctl member add --learner=true. Learners receive replicated log but don't vote. Add a learner → it catches up via snapshot + log → promote with member promote once RaftAppliedIndex is close to leader.
- Removing: member remove before stopping the process. If you stop first and then remove, the cluster may lose quorum during the gap.
- For 3 → 5: add learner-4, promote, add learner-5, promote. Quorum transitions are 2 → 3, never breaching availability.
- For 5 → 3 (scale down): remove voter-5 first, then voter-4. Each removal lowers quorum (3 → 3 unchanged, then → 2). Always one change at a time; let the cluster stabilize.
- Joint consensus (Raft membership change protocol) handles this safely if you do one member at a time, but operator mistakes (multiple adds in flight, stopping before removing) circumvent the safety.

Trap: Adding multiple voters at once, or adding a voter to a cluster whose remaining members are already at the edge of resource exhaustion. The new voter's catch-up traffic + the new quorum requirement can tip the cluster into write stalls.

---
10. Linearizable vs serializable reads

A dashboard polls etcd 10× per second for ~500 keys. Customer reports occasional load spikes on the etcd leader correlating with dashboard refreshes.

Question: What's happening, and how do you fix it?

Strong answer covers:
- Default read consistency is linearizable. Each read calls ReadIndex on the leader, which round-trips Raft to confirm leadership before serving. Cheaper than a write, but it's not free — it goes through the leader.
- Dashboard polling at 10Hz × 500 keys is 5,000 leader-touching reads/sec. Aggregated across multiple dashboard users, this becomes a real load.
- Switch dashboard reads to serializable: WithSerializable(). Reads serve from the local member's MVCC store. No leader round-trip. Possible staleness: a few ms in steady state, up to seconds during a leader change or partition.
- When not to do this: anything that drives decisions whose correctness depends on the latest committed state. Scheduling, leader election sanity checks, "is this lock still held?" — must be linearizable.
- For a metrics dashboard, eventual consistency is the right trade. Document it.
- If you can, replace polling with a watch: a single long-lived stream gives you all updates without repeated reads.

Trap: Switching everything to serializable to "speed up etcd," then debugging mysterious correctness bugs months later when a stale read causes a double-execute on a leader-elected job.

---
11. Schema and key-space design

You're designing the etcd key layout for a multi-tenant service registry. Keys will be queried by tenant, by service, and by instance health.

Question: Design considerations?

Strong answer covers:
- Etcd has no secondary indexes. All queries are prefix range scans on the key. Design key prefixes for your dominant query pattern.
- Hierarchical prefixes: /registry/{tenant}/{service}/{instance} lets you Range("/registry/T1/") for all of tenant 1, Range("/registry/T1/svc-foo/") for a specific service. Cheap.
- Querying by health (orthogonal dimension) requires either a secondary key (/health/unhealthy/T1/svc-foo/inst-12 → /registry/...) maintained transactionally with the primary, or scanning + filtering. Etcd transactions (Txn) support multi-key atomic writes for this.
- Per-instance lease pattern: bind each instance's key to a lease. Instance heartbeats KeepAlive its lease. Lease expires → key auto-deletes. Service registry is self-cleaning. One lease can be attached to multiple keys, so a single instance with several registered endpoints uses one lease.
- Value size: max 1.5MB per value by default, but smaller is much better. MVCC keeps copies of every revision; a 1MB key updated 1000 times = 1GB until compacted.
- Hot keys: a single key updated very frequently is a bottleneck — every update is a Raft round-trip serialized through the leader. Shard by partitioning: instead of /counter, use /counter/{shard_id} and aggregate at read time.
- Avoid putting per-revision business data in etcd. It's not a database. The 8GB default quota exists for a reason.

Trap: Using etcd's mod_revision as a per-key version number for application-layer optimistic locking. It is — and that works — but it's a cluster-global monotonic counter, not per-key. Two unrelated writes increment it. Build your CAS around Txn().If(Compare(ModRevision(key), "=", expected)) for the actual semantics.

---
12. When etcd is the bottleneck

Kubernetes cluster hits a wall at ~5,000 nodes. API server P99 climbs to 5s. Logs show "etcd: request took too long". etcd metrics show disk_wal_fsync_duration_seconds P99 at 80ms.

Question: What's happening?

Strong answer covers:
- WAL fsync is on every write commit. 80ms is terrible — should be sub-10ms on SSD. Either the disk is shared/throttled, or there's massive write volume, or fsync is being serialized across many committers.
- Investigation order:
  a. Disk subsystem. Is it actual SSD/NVMe local? Network storage (EBS gp2, gp3 without enough IOPS) hits fsync ceilings hard. Move to local NVMe or provisioned-IOPS storage. fio benchmarks should show sub-ms fsync latency.
  b. Write load. etcd_server_proposals_committed_total — derive rate. Kubernetes at 5,000 nodes commonly does 1,000+ writes/sec from leases, events, pod status updates. Reduce event volume (lifetime, types).
  c. Database size. If DB is large and compaction is behind, every write touches a bigger bbolt tree. Compact + defrag.
  d. Resource competition. Is the etcd VM also running other things? Dedicate the node.
- For Kubernetes specifically: shard the API resources across multiple etcd clusters. --etcd-servers-overrides=/events#https://etcd-events:2379 puts the high-write events resource on its own etcd cluster. This is the canonical scale-out pattern at large k8s deployments.
- Lease churn is a hidden tax. Every lease grant/revoke is a write. Lots of short-lived pods with leases → lots of etcd traffic. Tune lease TTLs upward where possible.

Trap: Adding more etcd nodes to "scale writes." Etcd writes are bottlenecked by the leader's fsync rate, not by the number of members. More nodes makes writes slower, not faster. The right axis is faster disk + sharded clusters per resource.