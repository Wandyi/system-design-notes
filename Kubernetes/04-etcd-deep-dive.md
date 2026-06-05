# 04 · etcd Deep Dive

The single point of consistency for the cluster. Every other component is stateless; etcd is the source of truth. Staff candidates should know its Raft model, watch protocol, scaling limits, and the ops procedures (compaction, defrag, backup).

## 4.1 The big picture

```
              ┌──────── kube-apiserver ────────┐
              │ (the only client)              │
              └───────────────┬────────────────┘
                              │ gRPC over TLS (port 2379)
                              ▼
              ┌────────────────────────────────┐
              │ etcd Raft cluster (3 or 5)     │
              │                                │
              │   [leader]  ───── log ───────► │
              │      │       replicates to     │
              │      ▼       ┌──────────────┐  │
              │   [follower] │  [follower]  │  │
              │      │       └──────────────┘  │
              │      ▼  state machine (boltdb) │
              │   key/value MVCC store         │
              └────────────────────────────────┘
                              │
                              ▼
                       Disk (NVMe SSD)
                       /var/lib/etcd
```

## 4.2 Raft in one diagram

```
   client ────PUT k=v────► [leader]
                              │ appends to local log
                              │ broadcasts AppendEntries
                              ▼
              ┌────────────┐ ┌────────────┐ ┌────────────┐
              │ follower 1 │ │ follower 2 │ │ follower 3 │
              └────────────┘ └────────────┘ └────────────┘
                              │
                  followers ack; once majority,
                  leader commits, replies to client
```

Properties:
- **Strongly consistent** (linearizable reads via leader; serializable reads from followers possible but rare).
- **Tolerates ⌊n/2⌋ failures** (5-node tolerates 2; 3-node tolerates 1).
- **Even cluster sizes (2, 4) are bad** — same write quorum (2 of 2, 3 of 4) but less fault tolerance.

3 is the standard for small clusters; 5 for large (more failure tolerance + better tail latency on reads). 7 is rare (write quorum cost outweighs gain).

## 4.3 MVCC and watch

Every write creates a new **revision** (monotonic int64). etcd stores all revisions up to the **compaction point** (older ones discarded).

```
   k=v1 (rev=10)
   k=v2 (rev=15)
   k=v3 (rev=20)
   k=v4 (rev=25)

   compact(rev=18) → discards rev 10, 15
```

### Watch

```
   client: watch /registry/pods/ from rev=20
   server: streams events starting at rev=20:
              {PUT key=pod1 rev=21}
              {DELETE key=pod2 rev=22}
              ...
```

If client's start revision is below compaction → error → client must re-list.

The apiserver's watch cache layers on top: it watches etcd, buffers events, serves multiple clients from the in-memory ring.

## 4.4 Leases — ephemeral keys

A **lease** has a TTL. Keys attached to it disappear when the lease expires (TTL not renewed).

Used for:
- **Node heartbeats** — kubelet writes a Lease object (in `coordination.k8s.io/v1` API), apiserver translates to etcd lease. Stops renewing → node marked unhealthy.
- **Leader election** — controllers acquire a Lease (or write to a ConfigMap with annotations, legacy).
- **Service-account-token refresh** — projected volume tokens have short TTL.

Lease TTL in etcd is independent of kubernetes Lease object; apiserver maps between.

## 4.5 The 8GB DB limit and how to live with it

`--quota-backend-bytes` defaults to 2GB; recommended max 8GB. Past 8GB:
- Defrag becomes slow (full DB rewrite).
- WAL replay on restart slow.
- Memory consumption (etcd loads index in memory).
- Watch performance degrades.

What fills the DB:
- Number of objects × average size × revision history (~3 latest revisions retained per key after compaction).
- Events resource is the most common bloat — verbose, frequent updates.

### Mitigations

1. **Events to a separate etcd cluster** — `--etcd-servers-overrides=/events#http://events-etcd:2379`. The kube-apiserver flag splits Events to its own etcd.
2. **Compact aggressively** — `etcd --auto-compaction-mode=periodic --auto-compaction-retention=8h`.
3. **Defragment periodically** — `etcdctl defrag` per member. (After compaction, the DB has free space; defrag reclaims to disk.)
4. **Audit object count** — `kubectl get --all-namespaces` per resource; identify the bloat.
5. **Use TTL-after-finished controller** to clean up old Jobs.

## 4.6 etcd HA and disk requirements

Recommended:
- **Dedicated nodes** (not co-located with kube-apiserver) — etcd is disk-IO bound; apiserver is CPU-bound; co-locating causes mutual interference.
- **Local NVMe SSD** — etcd fsyncs every write. Network-attached storage (EBS gp3, even gp2) has too much latency for >1000-node clusters.
- **Provisioned IOPS** — at least 5000 random write IOPS.
- **Separate disk from OS** — etcd's WAL + snapshot writes shouldn't compete with OS logs.

Sysctl tunings:
- `vm.swappiness=0` — never swap etcd.
- `vm.dirty_background_ratio=5` — start flushing early.
- `vm.dirty_ratio=10` — limit dirty pages.

## 4.7 etcd metrics that matter

Scrape `/metrics` on each etcd member. Key ones:

```
etcd_disk_wal_fsync_duration_seconds_bucket    # fsync latency; p99 < 10ms is healthy
etcd_disk_backend_commit_duration_seconds_*    # commit latency to boltdb; < 25ms p99
etcd_server_proposals_pending                   # backlog
etcd_server_leader_changes_seen_total           # should be ~0 except during restart
etcd_server_has_leader                          # 1 = healthy
etcd_mvcc_db_total_size_in_bytes               # the 8GB watch
etcd_mvcc_db_total_size_in_use_in_bytes        # after compaction (post-defrag)
```

The **wal_fsync** and **backend_commit** are the two big ones. Anything above 50ms at p99 means etcd is degraded; the cluster will feel slow.

## 4.8 Backup and restore

`etcdctl snapshot save` produces a single file representing the entire state:

```bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%F).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/ssl/etcd/ca.crt \
  --cert=/etc/ssl/etcd/peer.crt \
  --key=/etc/ssl/etcd/peer.key
```

Restore: every member must restore from the same snapshot:

```bash
etcdctl snapshot restore /backup/etcd-2026-06-01.db \
  --name=member1 \
  --initial-cluster=member1=https://10.0.0.1:2380,member2=...,member3=... \
  --initial-advertise-peer-urls=https://10.0.0.1:2380 \
  --data-dir=/var/lib/etcd-restored
```

Then start each member pointing at the new data-dir.

**Caveat**: restore loses any change since the snapshot. Backups are typically every 30 min or hourly.

## 4.9 Compaction and defrag — the ops dance

```
compact (logical):
  etcdctl compact <revision>
  - removes old revisions from MVCC
  - DB physical size unchanged
  - returns immediately

defrag (physical):
  etcdctl defrag --endpoints=<each member>
  - rewrites DB file to remove fragmentation
  - reclaims disk
  - BLOCKS the member during the operation (seconds to minutes)
  - run sequentially across members to keep quorum
```

Auto-compaction (configurable in etcd 3.x): `--auto-compaction-retention=8h` retains 8 hours.

Defrag is **never** automated by kube; you must script it. Run once a week, or after a known burst of writes.

## 4.10 etcd alternatives at scale

For very large managed-k8s offerings:
- **Sharded apiserver** — events to separate etcd; secrets to separate etcd; nodes to separate etcd. Kubernetes supports per-resource etcd overrides.
- **k3s + kine** — kine is a translation layer; runs etcd-protocol APIs against PostgreSQL/MySQL/SQLite. k3s defaults to SQLite. Trade-off: simpler ops, but consensus is now on the DB level (Postgres replication).
- **GKE / EKS / AKS** internal stores — hyperscalers run customized etcd (or replacement) at scales beyond what upstream supports. AWS EKS uses managed etcd with separation across availability zones. GKE used to use Bigtable-backed alternative for the most extreme scale.

## 4.11 Hot questions in production

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `etcd is slow` (apiserver 504s) | Disk fsync slow | Move to NVMe; provisioned IOPS |
| `etcdserver: mvcc: database space exceeded` | DB > quota | Compact + defrag; raise `--quota-backend-bytes` |
| `etcd cluster has lost quorum` | Two of three members down | Restore one; if all gone, restore from snapshot |
| Frequent leader changes | Network instability between members; CPU starvation | Dedicated network; CPU pinning |
| Watch falls behind | Apiserver overloaded; etcd write rate high | Tune apiserver watch cache; reduce event spam |
| Memory growth on etcd nodes | Many open watches | Cap clients; restart |

## 4.12 The KMS provider

For Secrets encryption at rest, integrate with cloud KMS:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources: ["secrets"]
  providers:
  - kms:
      apiVersion: v2
      name: aws-kms
      endpoint: unix:///var/run/kmsplugin/socket.sock
      cachesize: 1000
      timeout: 3s
  - identity: {}
```

kube-apiserver calls a KMS plugin (Unix socket) for envelope encryption: DEK encrypts data; KEK (in KMS) encrypts DEK. Rotating KEK is cheap (only DEKs need re-encryption).

KMS v2 (1.27+) supports rotation of the DEK cache.

## 4.13 Common interview probes

- **"What's the etcd quorum write story?"** Leader receives → appends to log → replicates → majority ack → commits → applies to state machine → responds to client.
- **"What if a follower is partitioned?"** Lags behind. Leader replicates eventually. If majority can still write, cluster is fine. If leader is partitioned and minority, a new leader is elected on the majority side; partitioned leader steps down on next term-bump.
- **"How do you handle etcd disk full?"** Add capacity if possible; compact + defrag; reduce object count; consider event split.
- **"Why is etcd disk performance so important?"** Every write fsyncs; slow fsync = slow apiserver = slow cluster. P99 > 50ms = unhealthy.
- **"Can you run etcd across regions?"** No — Raft latency would be the RTT. Etcd needs LAN. Multi-region k8s = multiple clusters.

## 4.14 Corner cases

- **Two clients race to write the same key**: one wins (revision incremented); the other gets a conflict error if it specified preconditions. Kubernetes uses `resourceVersion` for this.
- **Slow disk causes leader to fall behind on heartbeats** → leadership flip-flop. Pin etcd to fast disk.
- **Snapshot mid-write** doesn't corrupt — it's MVCC; snapshot captures a revision.
- **NTP skew across members** causes lease expiry weirdness. Run chrony/NTP.
- **Defrag during peak** causes apiserver latency spikes. Schedule off-hours; or do rolling per-member.

## Must-internalize

- Raft: leader replicates log, majority ack commits. 3 voters tolerate 1 fail; 5 tolerate 2.
- MVCC: every write a new revision; compaction discards old ones.
- Watch from a revision; if too old → 410 Gone → re-list.
- 8GB DB limit; events are the common bloat.
- NVMe SSD + dedicated disk + provisioned IOPS = etcd happy.
- `etcdctl snapshot save` for backup; restore replaces every member.
- KMS v2 for encryption at rest.
- Compaction is logical; defrag is physical and blocks the member.
