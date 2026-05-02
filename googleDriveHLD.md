# Google Drive — Staff-Level High-Level Design

A deep dive into how to design a globally-distributed file storage and sync platform like Google Drive / Dropbox / OneDrive. The focus is on the **algorithms** (chunking, dedup, delta-sync, Merkle reconciliation, conflict resolution), the **corner cases** of multi-device sync, and the **scalability / reliability / availability** posture you would expect at staff level.

---

## 1. Requirements

### 1.1 Functional
1. **Upload / download** of arbitrary binary files (text, photos, videos, archives) up to 5 TB per file.
2. **Folder hierarchy** with rename / move / delete (soft delete + trash + version history).
3. **Multi-device sync**: changes on device A propagate to all of the user's other devices (laptop, phone, web) within seconds.
4. **Selective sync** — user picks which folders to materialize on a given device.
5. **Sharing** — file/folder ACLs (viewer, commenter, editor), public links, link expiry, domain-restricted sharing.
6. **Versioning** — every save retained for N days; restore arbitrary version.
7. **Search** — full-text + metadata + OCR + content-type filters.
8. **Real-time collaboration** for Google-Docs-style files (separate problem; addressed briefly).
9. **Offline edit** with eventual reconciliation when connectivity returns.

### 1.2 Non-Functional
- **Scale**: ~3 B users, ~1 EB stored, 100s of millions of mutations/sec at peak.
- **Availability**: 99.99 % for read, 99.95 % for write (slightly lower because writes hit the consensus path).
- **Durability**: 11 nines (10⁻¹¹ annual object loss) — same target as S3 / GCS.
- **Latency**: metadata p99 < 100 ms, first-byte download p99 < 300 ms in-region, sync detection < 5 s.
- **Consistency**: read-your-writes for the owner; eventual consistency across devices and regions; strong consistency for ACL changes (security-critical).
- **Cost**: storage cost dominates — must dedupe, compress, and tier aggressively.

### 1.3 Capacity Math (back-of-envelope)
- 3 B users × 15 GB free + paid → ~1 EB raw, ~400 PB after dedup + compression.
- Average file 1 MB, so ~10¹⁵ logical objects → metadata DB has trillions of rows → must shard.
- 100 M DAU × 50 file ops/day → ~58 K writes/sec, peak ~250 K writes/sec.
- Read:write ≈ 10:1 → ~2.5 M reads/sec at peak.
- Egress: average download 5 MB × 1 B downloads/day → ~60 GB/s sustained, multi-Tbps at peak.

---

## 2. High-Level Architecture

```
                ┌────────────────────────────────────────────┐
   Clients ────▶│           Edge / CDN (Anycast)             │
 (web, mobile,  └─────────────────┬──────────────────────────┘
  desktop sync)                   │ TLS, signed URLs
                                  ▼
                       ┌────────────────────┐
                       │   API Gateway      │ ◀── auth (OAuth2 / mTLS for desktop client)
                       └─────────┬──────────┘
                                 │
       ┌──────────────┬──────────┼─────────────┬──────────────────┐
       ▼              ▼          ▼             ▼                  ▼
 ┌──────────┐  ┌────────────┐ ┌──────────┐ ┌──────────────┐ ┌────────────┐
 │ Metadata │  │  Block     │ │ Sync /   │ │ Notification │ │  Sharing/  │
 │ Service  │  │  Service   │ │ Reconcile│ │ (long poll / │ │  ACL Svc   │
 │ (Spanner)│  │  (CAS)     │ │ Service  │ │   gRPC stream│ │ (Spanner)  │
 └─────┬────┘  └─────┬──────┘ └─────┬────┘ └──────┬───────┘ └─────┬──────┘
       │             │              │             │               │
       │             ▼              │             ▼               │
       │      ┌────────────┐        │       ┌──────────┐          │
       │      │ Object     │        │       │ Pub/Sub  │          │
       │      │ Store      │        │       │ (Kafka)  │          │
       │      │ (Colossus/ │◀───────┘       └────┬─────┘          │
       │      │  S3)       │                     │                 │
       │      └────────────┘                     ▼                 │
       │                              ┌────────────────────┐       │
       └─────────────────────────────▶│ Indexer / Search   │◀──────┘
                                      │ (Inverted index +  │
                                      │  Vector store)     │
                                      └────────────────────┘
```

### Component Responsibilities

| Service | Responsibility | Storage |
|---|---|---|
| **API Gateway** | AuthN/Z, rate limit, request routing, schema validation | stateless |
| **Metadata Service** | File/folder tree, version chain, chunk → file mapping | Spanner / sharded MySQL |
| **Block Service** | Chunk upload/download, dedup, compression, encryption | KV layer over object store |
| **Object Store** | Immutable chunk blobs, erasure-coded across DCs | Colossus / S3 |
| **Sync Service** | Compute delta, drive Merkle reconciliation, push events | Stateful, sharded by user |
| **Notification Service** | Long-lived gRPC streams to clients; fan out change events | In-mem + Kafka |
| **Sharing/ACL** | Permission graph, link tokens, audit | Spanner (strong consistency) |
| **Search/Indexer** | Async inverted index + embeddings | Elasticsearch + vector DB |

---

## 3. Data Model

### 3.1 Logical schema (Metadata DB — Spanner-style)

```
users(user_id PK, ...)

namespaces(ns_id PK, owner_user_id, quota_bytes, used_bytes)
   -- A namespace = a root folder. Personal drive = 1 ns; shared drives = N ns.

nodes(ns_id, node_id PK, parent_id, name, type{file|folder},
      size, mime, created_at, modified_at, version_id, is_deleted, ...)
   -- Directory entry. Path is reconstructed by walking parent_id chain.

versions(node_id, version_id PK, manifest_id, author_id, created_at, size,
         is_current BOOL)
   -- Every save creates a new version row.

manifests(manifest_id PK, chunk_ids JSON, total_size, hash)
   -- Ordered list of content-addressed chunks that reconstitute the file.

chunks(chunk_id PK = sha256(content), ref_count, size_after_compression,
       storage_locator, encryption_dek_wrapped)
   -- Content-addressable, dedup-friendly.

acl(node_id, principal_id, role{viewer|commenter|editor|owner}, inherited BOOL)

cursors(user_id, device_id, last_seen_change_seq)
   -- High-water mark for sync, per device.

changes(ns_id, seq BIGINT PK, node_id, op{create|update|delete|rename|move}, ...)
   -- Append-only change log per namespace; drives sync.
```

Key choices:
- **Content-addressed chunks**: dedup falls out for free.
- **Per-namespace change log with monotonic `seq`**: gives clients a single value to checkpoint against → trivial resume.
- **Manifest indirection**: file content = ordered chunk list; lets us version cheaply (new manifest, mostly the same chunks).
- **Soft delete + version chain**: undelete and version restore are O(1) metadata ops.

### 3.2 Sharding strategy
- **Metadata** sharded by `ns_id` (a user's whole drive lives on one shard → most ops are single-shard, fast). Cross-namespace ops (sharing) are 2PC across shards.
- **Chunks** sharded by `chunk_id` (hash) with consistent hashing → uniform load, no hotspot from one celebrity user.
- **Change log** co-located with metadata shard (same Paxos group) → single-shard append, transactionally consistent with the node mutation that generated it.

---

## 4. Core Algorithms (the meat)

### 4.1 Content-Defined Chunking (CDC) — Rabin fingerprinting

**Why not fixed-size chunks?** Inserting one byte at the start shifts every byte → every fixed-size chunk changes → zero dedup, full re-upload.

**Rabin / FastCDC algorithm**:
```
slide a window of W bytes across the file
compute a rolling hash h(window)         # Rabin polynomial hash
if (h & MASK) == MAGIC:                  # MASK has e.g. 13 bits set → avg chunk ≈ 8 KiB
    cut a chunk boundary here
else if chunk_len >= MAX:                # cap at e.g. 128 KiB to bound metadata
    cut here anyway
```

Properties:
- **Shift-resistant**: inserting bytes only changes the chunks containing the insertion point; surrounding chunks keep the same boundaries → still deduped.
- **Average size tunable** via mask width. Tradeoff: smaller chunks = better dedup, more metadata; larger = opposite. ~4–16 KiB is typical.
- **MIN/MAX bounds** prevent pathological tiny chunks (degenerate text files) and oversized chunks (long binary runs).

**FastCDC** improves on Rabin with gear hashing and "normalized chunking" (vary the mask near MIN/MAX) — ~10× faster, tighter size distribution. Use FastCDC.

### 4.2 Deduplication — content-addressable storage

After chunking:
```
chunk_id = SHA-256(chunk_bytes)   # 256 bits, collision probability ≈ 0
```

Upload protocol:
1. Client chunks the file locally, sends list of `chunk_id`s to Block Service.
2. Server replies with the **subset** the cluster doesn't already have.
3. Client uploads only the missing chunks (often a tiny fraction).
4. Server bumps `ref_count` on every chunk in the manifest.

This gives:
- **Cross-user dedup** (everyone has the same iOS update.dmg → stored once).
- **Cross-version dedup** (editing 1 KB of a 1 GB file uploads ~1 chunk).
- **Resumable upload** for free — failed uploads leave already-stored chunks; retry is incremental.

**Privacy / encryption tension**: naive dedup leaks information ("does user X have file Y?"). Two mitigations:
- **Per-tenant dedup scope** (dedupe within a single org or user only) — costs storage, restores privacy.
- **Convergent encryption with proof-of-ownership**: encrypt chunk with key = HKDF(content), but require client to prove they have the plaintext before granting access — defeats confirmation-of-file attacks.

Garbage collection: when `ref_count` hits 0 → mark for deletion → tombstone for grace period (handles races with in-flight uploads referencing the chunk) → reclaim.

### 4.3 Delta Sync — rsync rolling-hash algorithm

When a client edits a file, we don't want to send the whole file. Two cases:

**(a) Client has the old version** (the common case — desktop sync):
- Client computes new manifest locally → diffs chunk lists → uploads only new chunks.
- This is the chunking + dedup pipeline above; nothing else needed.

**(b) Client doesn't have the old version** (web editor, "save as"):
- Server runs rsync algorithm:
  1. Server splits old file into fixed chunks of size B; computes for each chunk a *weak* (rolling Adler-32) and *strong* (SHA-256) hash → ships hash list to client.
  2. Client slides a B-byte window across new file; at every position, computes Adler-32 (cheap, rolling). If it matches one of the server's weak hashes, verify with SHA-256.
  3. Client sends a stream of `(matched_chunk_index | literal_bytes)` instructions. Server reconstructs.
- O(n) on each side, network = literal bytes only.

Drive uses (a) almost always — clients keep manifests cached.

### 4.4 Merkle-Tree Reconciliation (for folder-tree sync)

Problem: a client comes online after a week. Walking the change log for that week may be huge, and we want to detect what's actually different *now* (intervening churn may have cancelled out).

Approach: each folder has a **Merkle hash** = hash(sorted children's (name, type, content_hash | folder_hash)). Client and server exchange root hashes, then descend only into subtrees that disagree:

```
sync(client_root, server_root):
    if hash(client_root) == hash(server_root): return    # whole subtree in sync
    for entry in zip(client.entries, server.entries):
        if entry is folder:
            recurse
        elif entry is file:
            if hashes differ:  enqueue file for chunk-level sync
        elif missing on one side: schedule create/delete
```

- O(log N) network when only a few files changed in a deep tree, even if the tree has millions of files.
- Same idea Cassandra uses for anti-entropy and Git uses for `fetch`.

### 4.5 Conflict Resolution

Two devices edit the same file while offline.

**Binary files (most of Drive)**: server cannot merge bytes. Strategy = **divergence + manual reconciliation**:
- Server detects causally-concurrent versions via per-file vector clocks: each version carries `{device_id → counter}` of its parent. If neither dominates the other → conflict.
- Server keeps both, names the loser `foo (conflicted copy from <device> at <time>).docx`, surfaces a notification. **Never silently lose data** — first principle.
- "Loser" choice: typically the one that arrived second (server timestamp tiebreak), but the *content* of both is preserved.

**Text-like files (Docs, Sheets)**: real-time collaboration via:
- **Operational Transformation (OT)** — every edit is an `Op` (insert/delete at index); concurrent ops are *transformed* against each other so each side reaches the same state. Requires a central server to linearize and broadcast transformed ops (Google Docs).
- **CRDTs** (e.g., RGA, Yjs) — ops are commutative by construction; can converge peer-to-peer without a central authority. Cleaner semantics, larger metadata overhead per char (~10–20 bytes). Modern systems trend CRDT.
- Either way, persisted to a separate doc-collab service (out of scope for the binary file store).

**Folder-level conflicts** (rename loop, A→B on device 1, B→A on device 2): apply Lamport ordering on the change log; client replays ops in server order; if a rename collides, the loser becomes `name (1)`.

### 4.6 Compression

Per-chunk compression *after* CDC (not before — would defeat the rolling hash).
- **zstd level 3** as default — best ratio per CPU at scale.
- **Skip compression** when the chunk's entropy is high (already-compressed media). Cheap detection: compress the first 4 KiB; if ratio < 0.95, abort and store raw. Saves CPU on the 60 % of bytes that are MP4/JPEG/ZIP.
- Compression happens server-side after dedup check (so clients don't waste CPU compressing chunks the server already has).

### 4.7 Encryption

**Envelope encryption**:
- **DEK** (Data Encryption Key) per chunk → AES-256-GCM-encrypts the chunk content.
- DEK is wrapped by a **KEK** (Key Encryption Key) held in KMS, scoped per namespace.
- KEK rotation is metadata-only: re-wrap DEKs; chunks themselves never re-encrypted (huge cost saving).
- For dedup-friendliness: convergent-encryption variant uses `DEK = HKDF(content)` so the same plaintext encrypts to the same ciphertext → cross-user dedup still works. Trade-off: well-known plaintexts have known ciphertexts (info leak). Drive opts out of cross-user dedup for sensitive tenants.

**At rest**: AES-256-GCM. **In transit**: TLS 1.3, mTLS for desktop client.
**Client-side encryption** option (CSE / E2EE): client wraps DEK with a key the server never sees → server has zero-knowledge of content. Loses server-side features (search, preview). Drive offers CSE for enterprise.

### 4.8 Erasure Coding (durability + cost)

Replication is expensive (3× for 6-nines). For long-tail / cold chunks:
- **Reed-Solomon (k, m)** = e.g. (6, 3) → 6 data shards + 3 parity shards across 9 failure domains. 1.5× overhead, tolerates 3 simultaneous failures.
- Encode at chunk-write time; decode on read by fetching any 6 of 9.
- **Hot data** stays 3× replicated (fast reads, no decode CPU). Move to EC after 30 days idle (lifecycle policy).
- Read amplification: EC decode requires fetching from k nodes → k× the IOPS of replicated read. Mitigated by caching and by serving hot reads from the still-extant replica copy.

---

## 5. Multi-Device Sync — full corner-case treatment

This is the section interviewers probe hardest at staff level.

### 5.1 The sync loop (per device)

```
loop:
    cursor = local.cursor                              # last seq we saw
    events = SyncSvc.poll(ns_id, cursor)               # gRPC long-stream
    for evt in events:
        apply_to_local_fs(evt)                         # respects local edits
        local.cursor = evt.seq

    on local FS change (fsnotify / FSEvents / inotify):
        new_manifest = chunk_and_hash(file)
        SyncSvc.push(node_id, base_version, new_manifest)
        # base_version is the parent in the version DAG → server detects forks
```

- Server's `seq` is **monotonic per namespace** (Spanner commit timestamp or Paxos slot).
- Notification path is push-via-stream when online, fall back to long-poll (better through corporate proxies) every ~30 s when stream is dead.

### 5.2 Corner Cases

| Case | What goes wrong | How we handle it |
|---|---|---|
| **Two-way edit while offline** | Both devices upload divergent versions when they reconnect | Vector-clock detect; keep both; rename loser `foo (conflicted copy …).ext` |
| **Rename + edit collision** | Device 1 renames `a.txt → b.txt`; Device 2 edits `a.txt` | Server resolves by `node_id` (stable across renames). Edit applies to renamed node; no loss. |
| **Delete + edit collision** | Device 1 deletes; Device 2 edits | "Resurrection-on-edit": treat edit as undelete; restore from soft-delete. Notify user. |
| **Rename loops (A→B, B→A on different devices)** | Cycle in rename graph | Topological sort fails → break with `(1)` suffix on the loser; one user-visible conflict. |
| **Clock skew between devices** | Wall-clock timestamps misorder | Use **server `seq`** (monotonic logical clock) for ordering, never client wall clock. |
| **Partial upload, client crashes** | Some chunks uploaded, manifest never committed | Two-phase commit: (1) upload all chunks (refcounted by manifest_id staging row), (2) atomic manifest insert + version row. Stage TTLs garbage-collect orphans after 24 h. |
| **Same file uploaded twice concurrently from one device** | fsnotify can fire twice for one save (atomic-write tools rename a temp file) | De-dupe at client by debounce window (~500 ms) + content hash; coalesce into one upload. |
| **Network partition mid-sync** | Half the changes applied, then connection dies | Cursor-based resume — client retries from last `seq` it durably wrote. Server-side ops are idempotent (keyed by `(node_id, version_id)`). |
| **Disk full on receiving device** | Sync would corrupt local FS | Pre-flight `statvfs` check; if insufficient, mark folder "out of sync — needs space"; do not delete remote. |
| **Case-insensitive vs case-sensitive FS** | macOS HFS+ collapses `Foo.txt` and `foo.txt`; Linux doesn't | Detect FS, store original case in metadata, expose only one of the conflicting names locally with a `(case conflict)` warning. |
| **Path-length limits (Windows 260)** | Deeply nested folders fail to materialize | Detect, skip with surfaced error; offer "use long path support" toggle. |
| **Symlinks / hardlinks** | Cycles, escape-from-root | Don't follow symlinks; store as metadata-only "shortcut" entries. |
| **Permission change races** | User loses access mid-download | ACL check at every chunk request, not just at file open. Cancel stream with 403 if access revoked. |
| **Selective sync churn** | User unselects 10 GB folder; we mustn't delete server copy | Selective-sync deletion is *local-only*; server-side version of folder untouched. |
| **Quota exceeded mid-upload** | Manifest commit would push user over | Reject at manifest-commit time; pre-uploaded chunks are GC'd by refcount = 0. Pre-flight quota check is best-effort (race-prone) but improves UX. |
| **Time-travel restore** | User restores a version from 30 days ago that references chunks GC'd by quota cleanup | Versions hold strong refs (refcount); GC only when *no version* references the chunk. Trash retention drives chunk lifetime. |
| **Large-file resumable upload across week-long laptop hibernation** | TCP died, session token expired | Upload session is durable (manifest staging row), client resumes by listing already-uploaded chunk IDs. |
| **Mobile data savings** | Don't sync 4 K video on cellular | Client-side policy: pause uploads/downloads on metered networks unless user opts in. |

### 5.3 Notification fan-out

User has N devices online. A change on device 1 must reach the others within ~2 s.
- Sync service maintains gRPC streams keyed by `(user_id, device_id)`, sharded by `user_id`.
- On commit: write change log row → publish `(ns_id, seq)` to Kafka → subscribers fanout to relevant device streams.
- For **shared files**: change touches multiple namespaces (owner's ns + every collaborator's "shared with me" view) → publish to all affected namespaces.
- Backpressure: if a stream is slow, drop to "you have changes, repoll" signal — never buffer per-device unbounded.

---

## 6. Sharing and Permissions

### 6.1 Permission model

- **Direct grants**: ACL row on a node.
- **Inheritance**: child nodes inherit parent ACL unless explicitly overridden.
- **Public links**: capability tokens — opaque, signed, optionally bound to an account / IP / expiry.
- **Domain restriction**: `acl.audience = domain:example.com` enforced at API gateway via ID token claims.

### 6.2 Algorithm — permission check at scale

Naive: walk up the parent chain checking ACLs. With deep folder trees (1000+) this is slow.

Solution: **denormalized effective ACL** materialized at write time.
- Each node has `effective_acl = computed(parent.effective_acl, self.acl_overrides)`.
- ACL change at folder F → background job propagates new `effective_acl` to descendants. Reads see eventual consistency for permission *grants*, but **revocations are strongly consistent**: revoke writes a tombstone in a fast-path table that the auth check consults synchronously. (Security-critical: never grant access via stale cache; always revoke instantly.)
- For very wide trees (1 M descendants) propagation is async + paginated; the fast-path tombstone protects correctness while propagation catches up.

This is essentially **Google Zanzibar** under the hood — relation tuples, consistency tokens (`Zookies`) to ensure reads observe at-or-after a given ACL write.

### 6.3 Public-link tokens
```
token = HMAC(secret, node_id || expiry || requester_class) | b64
```
Stateless, revocable by rotating the per-link secret. Kept in a small KV store; lookup on every fetch.

---

## 7. Search

Two-stage:
1. **Sync indexer** — on file commit, enqueue a job. Worker pulls plaintext (text extraction for PDF/Office, OCR for images, Whisper for audio), pushes to Elasticsearch.
2. **Query path** — combine inverted index (BM25) + embeddings (vector ANN, e.g. ScaNN/HNSW) for semantic search; merge with permission filter.

**Permission-aware search** is hard. Two approaches:
- **Filter at query time**: fetch top-K then filter by ACL → may return < K results, requires over-fetching.
- **Index ACL** as a field per doc and AND it into the query. Faster, but ACL changes require reindexing → only practical because effective_acl is denormalized (§6.2).

Consistency: search index is eventually consistent (lag ~seconds); the source-of-truth list ("My recent files") reads from metadata DB directly.

---

## 8. Scalability

### 8.1 Sharding & data placement
- **Metadata shard key = `ns_id`**: keeps a user's tree on one shard → most ops single-shard. Resharding via Spanner-style auto-split when shard exceeds size threshold.
- **Chunk shard key = `chunk_id`**: consistent hashing across cells; rebalance via virtual nodes.
- **Hot-key mitigation**: a viral public file (chunk_id) reads from many users → cache it at edge CDN; origin sees ≤ 1 RPS regardless of viral load.

### 8.2 Caching tiers
| Tier | What | TTL |
|---|---|---|
| Edge CDN | Public/shared file bytes | hours |
| Regional read-through cache (in front of Block Svc) | Hot chunks for that region | minutes |
| Metadata cache (per-user-recent) | "My recent files", folder listings | seconds, invalidated on write |
| Negative cache | "chunk not found" | seconds (defends scrapers) |

Cache invalidation: writes publish to a per-user invalidation channel; clients & caches subscribe.

### 8.3 Storage tiering
Lifecycle: **Hot SSD (3× replicated)** → **Cold HDD (EC 6+3)** at 30 days idle → **Archive tape / cold object** at 1 year idle. Promotion on access. Saves an order of magnitude on cost without breaking durability targets.

### 8.4 Geographic distribution
- **Metadata**: globally-replicated Spanner with leader pinned to user's home region. Cross-region writes pay a round-trip; 99 % of users read locally.
- **Chunks**: written first to home region (fast ack), async replicated to a paired region for DR. Optionally synchronously replicated for enterprise tier.
- **Edge POPs** terminate TLS, serve cached reads, forward cache misses to nearest cell.

### 8.5 Backpressure & quotas
- Per-user write QPS cap (token bucket) — protects against a runaway sync client.
- Per-ns storage quota — checked at manifest-commit time.
- Global circuit breaker on Block Svc → if upload pipeline is saturated, push back at the API gateway with `503 + Retry-After`; clients honor.

---

## 9. Reliability

### 9.1 Durability stack
- **At least 3 copies** until cold, **EC (6,3) or (10,4)** when cold.
- Copies span at least 3 **failure domains** (rack / DC / region depending on tier).
- **Background scrubber** reads every chunk on a 30-day cadence, verifies SHA-256, repairs from peers if mismatch (silent corruption defense — rotting drives flip bits).
- **Anti-entropy** Merkle compare across replicas catches missed writes.

### 9.2 Failure handling
| Failure | Response |
|---|---|
| Single drive | Hot-spare; rebuild from replicas in background |
| Single node | Cordoned; replicas served from peers; manifest jobs reroute |
| Single rack | EC degrades gracefully (k-of-n still readable); replicated tier loses 1 of 3, repairs |
| Single DC | Region-paired replica takes over; metadata leader fails over (Paxos elects new leader in seconds) |
| Region | Failover to paired region (RPO ≤ 5 s for sync replication, ≤ 60 s for async); customer-visible blip during DNS shift |
| Corrupted chunk (bit-flip) | Scrubber detects via SHA-256 mismatch; repair from peer; if all peers corrupt → escalate to tape backup |

### 9.3 Backups separate from replicas
Replication ≠ backup. Replication propagates a `DELETE` instantly. Drive runs a **separate PITR system** (point-in-time recovery, e.g. snapshots every 6 h, retained 30 days) so we can restore from operator error / ransomware.

---

## 10. Availability

### 10.1 Read path SLO 99.99 %
- Stateless API tier behind L7 load balancer with 5+ availability zones.
- Reads can serve from any replica (eventual consistency acceptable for content; chunks are immutable so actually strongly consistent on read).
- Edge CDN absorbs ~80 % of read traffic — even if origin is degraded, cached files keep flowing.

### 10.2 Write path SLO 99.95 %
- Writes need consensus (Paxos/Raft) on the metadata shard → fewer 9s (one majority-quorum failure stalls writes for that shard).
- Mitigation: shards are small (1000s of users each), so a stuck shard affects 0.0001 % of users.
- Client buffers writes locally; pending mutations replay when service returns. UI degrades gracefully to "saving…".

### 10.3 Multi-region active-active for metadata
Spanner gives us this: writes go to leader, but leader can be in any region; read replicas everywhere. For Drive's read-heavy mix, 5-replica Paxos groups across 3 regions are typical (survives full region loss).

### 10.4 Graceful degradation
- If search index is down → fall back to filename-only query against metadata DB.
- If notification pipe is down → clients fall back to 30 s polling. Slower sync but no data loss.
- If a chunk is temporarily unreadable → return partial-content error; client retries other replicas, escalates to repair queue.
- If sharing service is down → cached ACLs serve reads, **all ACL writes fail closed** (security over availability for permission changes).

---

## 11. Observability

- **Metrics** (Prometheus / Monarch): per-API p50/p99 latency, error rate, dedup ratio, EC overhead, sync lag (commit-to-device), chunk repair queue depth.
- **Distributed tracing** (Dapper / OpenTelemetry): every upload/download has a trace; sample 1 % normal, 100 % on error.
- **Audit log** (immutable, separate cluster): every ACL change, share creation, permission grant — required for compliance.
- **Sync-lag dashboard**: P50 / P99 of `change_committed → device_received` per region. The single most important UX metric.

---

## 12. Trade-offs and Things Staff-Level Should Be Ready to Argue

| Decision | Alternative | Why we picked this |
|---|---|---|
| **Content-defined chunking (FastCDC)** | Fixed-size chunks | 5–10× better dedup ratio; small CPU cost; insertion-resilient |
| **Spanner for metadata** | Sharded MySQL + Vitess | Spanner gives global strong consistency, transparent resharding, paid for in cost. MySQL+Vitess is cheaper but cross-shard sharing transactions are painful. |
| **Server-side dedup (post-encryption)** | Client-side dedup | Server-side keeps the keying simple and supports server features (preview, search). E2EE tier opts out, accepting loss of cross-user dedup. |
| **Conflict-as-divergence (binary files)** | Server-side merge / last-write-wins | Never silently lose user data; let humans reconcile |
| **CRDT for collab docs, OT for legacy Docs** | Pick one | OT was incumbent (Google Docs); CRDTs win on correctness for new builds |
| **Eventual consistency for reads, strong for ACLs** | Strong everywhere | Strong everywhere kills tail latency; ACL is the one place we can't compromise |
| **Erasure coding for cold data** | 3× replication everywhere | 50 % cost reduction with same durability; small read latency hit acceptable for cold tier |
| **Per-namespace change log** | Global change stream | Per-ns is shard-local → cheap; global stream is a thunderherd |
| **Long-poll / streaming notifications** | Client polling | Cuts mean sync lag from ~30 s to ~2 s; cost is connection state |

---

## 13. Open Questions / Extensions

- **Smart compression by mime** (different codecs for image vs text) — speculative, complicates pipeline; usually not worth it.
- **Block-level dedup across compressed archives** — peeking inside zip/tar to dedup contents. High value (CI artifacts, package mirrors) but complex; out of scope for v1.
- **ML-driven prefetch** ("you usually open this folder Monday morning, prewarm it"). Cheap win on cold-tier promotion.
- **Blast-radius isolation** — one runaway script shouldn't be able to take down the whole sync stream for a user; per-device fairness in the notification fanout.
- **Regulatory data residency** (GDPR, India DPDPA) — pin namespace to a region; cross-region access denied or audited.

---

## 14. Quick recap of "what makes this staff-level"

1. **Algorithms named, not waved at**: FastCDC, rsync rolling hash, Merkle reconciliation, Reed-Solomon (k,m), Paxos, Zanzibar-style ACLs, OT vs CRDT.
2. **Corner cases enumerated** for multi-device sync — interviewer's favorite probing ground.
3. **Consistency boundaries explicit**: eventual for content, strong for ACLs, monotonic per-ns ordering for sync.
4. **Quantified trade-offs**: dedup ratios, EC overhead, replication factor, expected sync lag.
5. **Security baked in**: envelope encryption, KEK rotation, optional CSE/E2EE, permission-aware search.
6. **Failure modes mapped** end-to-end (drive → rack → DC → region) with concrete RPO/RTO targets.
7. **Operational reality**: backups separate from replicas, scrubber + anti-entropy, audit log, sync-lag SLO.
