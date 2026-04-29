# Google Docs — Staff-Level High-Level Design

> A deeply technical, staff-engineer-level design for a real-time collaborative document editor at Google Docs scale.
> Focus: the *algorithms* (OT, CRDTs, presence, versioning), *corner cases*, and the *scalability/reliability/availability* trade-offs that decide whether the system survives a Monday morning at 10:00 UTC when 30 million users open their docs simultaneously.

---

## Table of Contents

1. [Requirements & Scale](#1-requirements--scale)
2. [Back-of-Envelope Estimation](#2-back-of-envelope-estimation)
3. [High-Level Architecture](#3-high-level-architecture)
4. [Core API & Wire Protocol](#4-core-api--wire-protocol)
5. [Document Model (Rich Text Representation)](#5-document-model-rich-text-representation)
6. [The Heart of the Problem: Concurrent Editing](#6-the-heart-of-the-problem-concurrent-editing)
7. [Operational Transformation (OT) — Deep Dive](#7-operational-transformation-ot--deep-dive)
8. [CRDTs — Deep Dive (Logoot, RGA, LSEQ, Yjs)](#8-crdts--deep-dive-logoot-rga-lseq-yjs)
9. [OT vs CRDT — The Real Trade-off](#9-ot-vs-crdt--the-real-trade-off)
10. [Cursor, Selection & Presence (Awareness Protocol)](#10-cursor-selection--presence-awareness-protocol)
11. [Document Storage, Versioning & Snapshots](#11-document-storage-versioning--snapshots)
12. [Offline Sync, Reconciliation & Long-Disconnect Handling](#12-offline-sync-reconciliation--long-disconnect-handling)
13. [Comments, Suggestions & Track Changes](#13-comments-suggestions--track-changes)
14. [Permissions, Sharing & ACL Propagation](#14-permissions-sharing--acl-propagation)
15. [Search Indexing & Full-Text Discovery](#15-search-indexing--full-text-discovery)
16. [Real-Time Communication Layer (WebSockets at scale)](#16-real-time-communication-layer-websockets-at-scale)
17. [Database & Storage Architecture](#17-database--storage-architecture)
18. [Caching Architecture](#18-caching-architecture)
19. [Scalability Deep Dive](#19-scalability-deep-dive)
20. [Reliability & Fault Tolerance Deep Dive](#20-reliability--fault-tolerance-deep-dive)
21. [Availability Deep Dive](#21-availability-deep-dive)
22. [Observability & Operational Excellence](#22-observability--operational-excellence)
23. [Corner Cases & Hard Problems](#23-corner-cases--hard-problems)
24. [Appendix A — Why Google Docs Picked OT (Real History)](#appendix-a--why-google-docs-picked-ot-real-history)
25. [Appendix B — Pseudocode Reference](#appendix-b--pseudocode-reference)

---

## 1. Requirements & Scale

### Functional Requirements

**Authoring**
- Create, open, edit, rename, move, delete a document.
- Rich text: bold/italic/underline/strike, headings, lists (ordered/unordered/nested), tables, images, links, code blocks, equations.
- Multi-page layout with pagination, page breaks, headers/footers, footnotes.
- Auto-save with no explicit save button.

**Real-Time Collaboration (the core)**
- Multiple users editing the same document simultaneously see each other's changes within ~100 ms.
- Live cursors and selections of every collaborator with a colored caret + name label.
- No edit ever lost; no edit ever duplicated; final state must be identical for all users (convergence).
- Causal ordering preserved (if A reads B's change before writing, A's change is ordered after B's).

**Offline & Resilience**
- Full editing while offline; on reconnect, merge cleanly with no manual conflict resolution dialog.
- Edits made while offline must converge with edits other users made during the same window.

**Versioning & History**
- Named versions ("File → Version history").
- Restore any prior version.
- Per-character authorship attribution ("blame").

**Comments & Suggestions**
- Anchored comments on a text range (anchor must survive edits).
- Suggesting mode (track changes): proposed insert/delete that another user accepts/rejects.
- @-mentions trigger notifications.

**Permissions & Sharing**
- Owner / Editor / Commenter / Viewer.
- Link sharing with anyone-with-link / restricted / domain-restricted.
- Per-document and per-folder ACLs (folder ACLs cascade).
- Real-time permission revocation: kicked-out user must lose access *now*, not after token expiry.

**Search**
- Full-text search across all docs the user can access.
- Search within a document.
- Find-and-replace (single + bulk).

**Import/Export**
- Open `.docx`, `.odt`, `.txt`, `.html`, `.md`.
- Export to `.docx`, `.pdf`, `.odt`, `.html`, `.epub`, plain text.

**Out of scope** (mentioned, not deep-dived): Smart Compose ML, real-time grammar/spell checking inference, mobile-specific layout engine, accessibility (TalkBack/VoiceOver), printing pipeline.

### Non-Functional Requirements

| Concern | Target | Why |
|---|---|---|
| Latency (edit → echo to peer) | p50 < 50 ms, p95 < 200 ms | Below human perception of lag |
| Latency (open document) | p95 < 2 s for 100-page doc | Cold-start budget |
| Availability | 99.99% (52 min/year) | Productivity tool — outages cost real money |
| Durability | 11 9s (10⁻¹¹ data loss / yr) | Document loss is catastrophic |
| Consistency (in-doc edits) | Strong eventual (convergence guaranteed) | Concurrent edits must produce one canonical state |
| Consistency (permissions) | Strong (linearizable) for revocation | Security-sensitive |
| Consistency (search index) | Eventual (seconds-minutes lag OK) | OK if a brand-new doc isn't searchable for 30 s |
| Max concurrent editors / doc | 100 hard, 10 typical | Beyond ~50, presence becomes the bottleneck |
| Max document size | ~50 MB serialized, ~2M characters | Below this performance is interactive |

### What "staff-level" requires you to address

A junior design says "WebSocket + database." A staff design must answer:

1. **Which concurrency control algorithm?** OT or CRDT — and *why*, not as religion but as a trade space.
2. **What happens when two clients diverge for 4 hours offline?** Both editing a 1000-row table. Concretely: what data structure, what ops, what merge?
3. **What do I store on disk?** Operations? Snapshots? Both? At what cadence does compaction run?
4. **How do I revoke permissions for a user mid-edit, in flight, without breaking other users' sessions?**
5. **What is the failover behavior of the WebSocket layer when a session pinning host goes down?**
6. **When the network partitions a region, do I serve stale reads, refuse writes, or pick a primary?**
7. **What is the blast radius of a corrupted op? Can a single bad client poison a document forever?**

Each is answered below.

---

## 2. Back-of-Envelope Estimation

```
Users
  Total registered:              3 billion (Google Workspace + free)
  Monthly active:                1 billion
  Daily active:                  500 million
  Peak concurrent:               80 million (Mon 10:00 UTC band, US Eastern morning)

Documents
  Total docs stored:             ~50 billion
  Avg doc size:                  ~50 KB serialized text + structure, p99 ~5 MB
  Median doc:                    ~10 KB (single-page memo)
  Heavy-tail docs:                 1% of docs hold 50% of bytes (the 500-page novels)
  New docs / day:                ~200 million

Concurrent editing sessions
  Concurrent doc opens:          ~100 million (a "session" = doc loaded in tab)
  Docs with >1 simultaneous editor: ~5% → 5 million docs at any moment
  Median collaborators / session: 1
  P99 collaborators / session:    8
  Peak collaborators / single doc: 100 (cap)

Edit operations
  Avg keystrokes / active editor / min: 60 (one per second when typing)
  Active editors at peak:        ~20 million (typing right now, not just open)
  Ops/sec at peak:               20M × 1/sec = 20 million ops/sec
  After client-side coalescing (batch every 100 ms): ~2-5 million ops/sec arriving at servers
  Per op: { docId, opId, baseRev, type, payload, authorId, ts } ≈ 200 bytes wire
  Total wire throughput:          2M × 200B = 400 MB/sec sustained
  Peak burst (event-driven, e.g. demo to all-hands):  1 GB/sec

Persistence (operation log)
  Ops/day:                       2M × 86400 = 170 billion ops/day
  Storage / day raw:             170B × 200B = 34 TB/day
  After compression (snappy):    ~10 TB/day
  After snapshot+compaction (keep ops only since last snapshot, snapshot every 1000 ops):
    Working set ops in hot store: 2M × 60 × 30 ≈ 3.6 trillion ops × 200B ≈ 720 TB hot

Snapshots
  1 snapshot per doc per day for actively-edited docs:
    actively-edited / day = 100M × ~60% = 60M
    snapshot avg size = 50 KB → 60M × 50KB = 3 TB/day
  Versioned snapshots kept = 30 days hot, infinite cold (Cloud Storage Coldline)

Real-time fanout
  At peak, 5M docs each with ~3 editors connected: 15M open WebSockets
  Each editor receives ~10 ops/sec from peers (typing rate × peers)
  Outbound fanout msgs/sec: 15M × 10 = 150M msgs/sec
  Per msg ≈ 250B → 37.5 GB/sec outbound

Cursor / presence
  Each user broadcasts cursor every 200 ms while active:
    20M active × 5 Hz = 100M presence msgs/sec
  Each msg ≈ 60B (small) → 6 GB/sec
  Critical: presence must NOT go through the OT pipeline (too expensive)

Search
  Full-text index: 50B docs × avg 5 KB indexable text = 250 TB raw text
  Inverted index ≈ 3× expansion = 750 TB on disk
  Sharded across ~10K shards × 75 GB each
  Query rate: 50M searches/day → 580 QPS avg, ~5K QPS peak

Storage totals
  Hot path (Spanner-class):       PBs (live ops + metadata)
  Warm (snapshots, Bigtable):     tens of PBs
  Cold (versions, Cloud Storage): hundreds of PBs

Network
  Egress at peak:                 ~50 GB/sec from collab servers
  Need globally distributed WebSocket frontends (POPs)
```

**Key insight**: The system has *three* very different workloads stacked together:

1. **Tiny, ultra-frequent ops** (200B, 2M/sec) — needs a low-latency replicated log.
2. **Periodic snapshots** (50KB, ~700/sec average) — needs a blob store with versioning.
3. **Bursty fanout** (37 GB/sec presence + ops) — needs a smart, location-aware pub/sub.

Picking *one* database for all three is the classic mistake. We deliberately use *different* substrates per concern.

---

## 3. High-Level Architecture

```
                        ┌──────────────────────────────────┐
                        │          Client (Browser)         │
                        │  - Local document model (mirror)  │
                        │  - OT engine + transformation     │
                        │  - Operation buffer (pending)     │
                        │  - WebSocket client + reconnect   │
                        └──────────────┬───────────────────┘
                                       │ WSS (HTTP/2 or HTTP/3)
                                       ▼
            ┌────────────── Anycast / Global Load Balancer ──────────────┐
            │             (Google Front End, GeoDNS, AnyIP)              │
            └──────────────────────────────┬────────────────────────────┘
                                       │
                                       ▼
                ┌──────────────────────────────────────────────┐
                │   Edge POPs (regional WebSocket terminators) │
                │    - TLS termination                         │
                │    - Auth token validation (short-lived JWT) │
                │    - Connection multiplexing                 │
                │    - Sticky routing to "session host"        │
                └──────────────────┬───────────────────────────┘
                                   │
                                   ▼
        ┌──────────────────────────────────────────────────────┐
        │           Collaboration Service (stateful)           │
        │   per doc → exactly ONE "session host" (leader)      │
        │   - Sequential serialization of ops                  │
        │   - OT transform (or CRDT merge)                     │
        │   - Ordered broadcast to subscribers                 │
        │   - Operation log writer (synchronous)               │
        │   - Snapshot creator (async, backpressured)          │
        └──┬────────────────┬──────────────────────┬───────────┘
           │ ops log        │ snapshots             │ presence
           ▼                ▼                       ▼
    ┌──────────────┐  ┌──────────────────┐  ┌──────────────────────┐
    │  Spanner     │  │  Bigtable / Blob │  │  In-memory pub/sub   │
    │ (replicated  │  │   (snapshots,    │  │  (Redis Streams /    │
    │  op log per  │  │    versions)     │  │   Bigtable PubSub)   │
    │     doc)     │  │                  │  │                      │
    └──────────────┘  └────────┬─────────┘  └──────────────────────┘
                               │
                               ▼
                       ┌──────────────────┐
                       │  Cold tier (GCS) │
                       │  long-term       │
                       │  versions        │
                       └──────────────────┘

   Auxiliary services (peers of the collab service):

       ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
       │  Drive Metadata  │  │   ACL / Auth     │  │   Search Index   │
       │  (titles, tree,  │  │   (Zanzibar-     │  │  (rebuilt        │
       │   ownership)     │  │   like)          │  │   asynchronously)│
       └──────────────────┘  └──────────────────┘  └──────────────────┘
```

### Why a single "session host" per document?

The hardest theoretical part of OT is making transformation *converge* under arbitrary concurrent re-ordering (TP2 property — proven hard, see §7). The clean engineering shortcut is: **route every op for a doc through one host, give it a monotonic sequence number, and broadcast in that fixed order.** With this, OT only needs the simpler TP1 property, which is provably solvable.

Cost: that host is a single point of contention. Mitigations:
- The host is *stateless* re: persistence (every op is replicated to Spanner before ack).
- Failover takes ~1-2 s (consensus elects a new leader from the replica set).
- Sharding is per-doc, not per-user, so load distributes naturally.

This is the same architecture Google Wave used and the lesson Google Docs inherits.

### Stateful vs stateless

| Layer | State? | Why |
|---|---|---|
| Edge POP | Stateless | Just terminates TLS, looks up routing |
| Session Host | Soft-stateful (in-memory rev, broadcast list) | Restorable from log on failover |
| Op log | Hard-stateful (Spanner) | Source of truth |
| Snapshot store | Hard-stateful (durable blob) | Optimization for reads |
| Presence | Soft-stateful (TTL'd in pub/sub) | Lossy, OK |

---

## 4. Core API & Wire Protocol

### REST endpoints (control plane)

```
POST   /v1/documents                          create new doc
GET    /v1/documents/{id}                     load doc (returns snapshot + base revision)
DELETE /v1/documents/{id}                     soft delete
PUT    /v1/documents/{id}/permissions         update ACL
GET    /v1/documents/{id}/versions            list saved versions
POST   /v1/documents/{id}/versions/{v}/restore restore version v as new HEAD
GET    /v1/search?q=...                       full-text search
```

### WebSocket protocol (data plane — the interesting one)

The connection is upgraded from HTTPS, then multiplexes per-doc subscriptions over a single socket.

```
Client → Server: open
{
  "type": "subscribe",
  "docId": "doc-abc",
  "baseRevision": 8421,        // last revision client saw
  "clientId": "uuid-1234",     // unique per session
  "auth": "<short-lived-jwt>"
}

Server → Client (immediately, if client is behind):
{
  "type": "catchup",
  "ops": [ { rev: 8422, op: ... }, { rev: 8423, op: ... }, ... ],
  "headRevision": 8470
}

Client → Server (edit submission):
{
  "type": "submit",
  "docId": "doc-abc",
  "clientId": "uuid-1234",
  "clientSeq": 17,             // monotonic per-client; for de-dup on retry
  "baseRevision": 8470,        // server rev this op was composed on top of
  "op": { "type": "insert", "pos": 142, "text": "Hello" }
}

Server → All subscribers (broadcast, after transform + log):
{
  "type": "ack",                // to author
  "clientId": "uuid-1234",
  "clientSeq": 17,
  "revision": 8471,             // assigned by server
  "transformedOp": { "type": "insert", "pos": 145, "text": "Hello" }
}

Server → other subscribers:
{
  "type": "remoteOp",
  "revision": 8471,
  "authorId": "u-555",
  "op": { "type": "insert", "pos": 145, "text": "Hello" }
}

Presence (separate channel, lossy, no log):
Client → Server: { "type": "cursor", "pos": 200, "selStart": 195, "selEnd": 210 }
Server → Server (pub/sub): broadcast to other subscribers without persisting

Heartbeats: ping/pong every 30s, force reconnect after 90s silence.
```

Key design notes:
- Ops carry `baseRevision` so the server knows what to transform against. Client never blocks waiting for server ack to type the next character (see §7, *client-side OT*).
- `clientSeq` makes each op idempotent. If a TCP retransmit causes the server to receive the same op twice, the second one is dropped at the dedup layer.
- Presence runs through a **separate, lossy** channel — never goes into the op log. Presence at 5 Hz × 20M users would *dominate* the log and add no value (cursors don't need durability).

---

## 5. Document Model (Rich Text Representation)

The most important early design decision: how the client and server represent the document.

### Naive approach (do not use): a giant string

`"Hello **world** how are *you*?"`

Problem: rich-text formatting is now *encoded* in the string. Inserting at character index `i` requires parsing markdown again. Concurrent edits to formatting and content pollute each other. Stops working at ~10K chars.

### Naive approach 2 (do not use): DOM tree

Store the literal HTML/DOM. Problem: identical visible text can have many DOM trees, transformation rules are O(n) and inconsistent across clients with slightly different parsers, and "pos" no longer has a stable meaning.

### Production approach: piece table + attribute spans

This is roughly what Google Docs uses internally (their "Kix" model is similar in spirit).

```
Document {
  text:     ImmutableString  // a piece-table-backed rope
  attributes: SegmentTree<Range, AttrSet>
  blocks:   List<BlockMarker>   // paragraph, heading, list-item, table-cell, etc.
  embeds:   Map<EmbedId, EmbedSpec>  // images, equations, links
  meta:     DocumentMetadata
}
```

A "Kix-style" representation:

1. **Text content** is a flat sequence of code points (UTF-16). All positions are *codepoint indices*, never byte indices (avoids UTF-8 multi-byte hazards).
2. **Block markers** are pseudo-characters (e.g., `` private-use area) interleaved into the text — one for each paragraph break. This lets paragraph-style edits ("make this paragraph an H2") become attribute changes on a single character.
3. **Attributes** (bold/italic/font/size/color/...) are stored as ranges in a segment tree, keyed by the immutable IDs of the boundary characters (more on identity below).
4. **Embeds** (images, equations, drawings) are also pseudo-characters with an ID pointing to a side-table. A single character occupies one position; the actual image lives in blob storage.

### Stable identity (critical for OT/CRDT)

Plain offsets are unstable across concurrent edits. We need either:

- **OT path**: positions are server-rev-relative integers; transformation rebases them.
- **CRDT path**: every character has an immutable ID (a *position identifier*) — see §8.

Real Google Docs uses the OT path: positions are integers, the server is the linearizer, transformation rebases concurrent ops.

### Trees vs flat for paragraphs

Headings, lists, tables — these *are* hierarchical. We still avoid storing them as a tree on the wire. Instead:

- The flat character sequence has block markers.
- Each block marker carries an attribute "block type" (paragraph / heading-1 / list-item-l2 / table-cell-r4c2).
- A separate cached "block index" computed by the renderer reconstructs the visual tree.

Why? Because *flat sequences* are what OT/CRDTs are well-defined on. Tree CRDTs exist (Mark Kleppmann's work, Tree-OT) but are *much* harder to make convergent. Industry has converged on flat-sequence-with-block-markers.

### Tables — the hard case

Tables are conceptually 2D. The trick: a table is a contiguous sequence of paragraphs in flat order (row 1 cell 1 ... row 1 cell N, row 2 cell 1 ... ), bracketed by `<table-start>` / `<table-end>` markers. Adding/removing rows is then editing the flat sequence. Resizing columns is metadata on the table marker. Two users editing the same cell collapse to ordinary text-OT in that cell's range.

**Corner case**: two users insert a row at the same position concurrently. OT needs to decide which row "wins position 0." We use: tie-break by `(authorId, clientSeq)` lexicographic. Result: deterministic order; both rows appear; users not surprised.

---

## 6. The Heart of the Problem: Concurrent Editing

The whole reason Google Docs is hard.

### The naive "lock" approach (does not work)

Lock the document on edit. Other users see read-only. Real-time collaboration is impossible. Discarded.

### Last-Writer-Wins (does not work)

Each save overwrites the file. User A saves a paragraph. User B, who was editing simultaneously, saves moments later — A's paragraph vanishes. This is what Dropbox does (and creates `.conflicted` files). For interactive editing, unacceptable.

### Diff-and-merge (does not work)

`git`-style. Run a 3-way merge on every save. Problem: at 60 char/sec/user, you'd 3-way-merge ~20 times per second per user. Diff algorithms are O(n²) on doc length. Doesn't scale, and merges still ask for human resolution.

### The two real options

| Option | Where edits live | Conflict resolution | Where used |
|---|---|---|---|
| **OT** (Operational Transformation) | Operations transformed against each other | Server arbitrates linear order | Google Docs, Etherpad, ShareDB |
| **CRDT** (Conflict-Free Replicated Data Type) | Operations carry IDs that make them mergable | No central arbiter needed | Yjs, Automerge, Figma (sort of), Apple Notes (rumored) |

Both *converge*: given the same set of ops, all replicas reach the same state. They differ in *how*.

We will deep-dive both. Then explain why Google chose OT, why a greenfield startup might choose CRDT, and what the real-world considerations are.

---

## 7. Operational Transformation (OT) — Deep Dive

### The fundamental idea

Each edit is an *operation* — `Insert(pos, char)`, `Delete(pos, len)`, `Format(range, attr)`.
When op `A` and op `B` are made concurrently from the same starting document `D`, they must each be **transformed** against the other so that `apply(A', apply(B, D)) == apply(B', apply(A, D))` — i.e., they converge.

```
        D
       / \
      A   B           A and B were composed in parallel against D
     /     \
   D-A    D-B
     \     /
      \   /
       D'              applying transformed ops gives same result on both sides
       
   To get from D-A to D', apply transform(B, A)   (B', given A happened first)
   To get from D-B to D', apply transform(A, B)   (A', given B happened first)
```

### Transformation functions

Notation: `T(A, B) = (A', B')` where `A' = transform(A, B)` and `B' = transform(B, A)`.

**Insert vs Insert:**
```
Op A: Insert(pos=5, "X")
Op B: Insert(pos=3, "Y")    // B is to the left of A
A' = Insert(pos=6, "X")     // A shifts right by len("Y")=1
B' = Insert(pos=3, "Y")     // B unaffected

If A and B insert at the SAME position, tiebreak deterministically:
  by (siteId) lexicographic: lower siteId goes first.
  That site's op is unchanged; the other shifts by len of the first.
```

**Insert vs Delete:**
```
Op A: Insert(pos=5, "X")
Op B: Delete(pos=3, len=4)        // deletes range [3,7)
                                  // A's pos=5 falls inside the deleted range — collapse it
A' = Insert(pos=3, "X")           // collapsed to the deletion point
B' = Delete(pos=3, len=4)         // unaffected (B doesn't shift)

Edge cases:
  - A.pos < B.pos:  A unaffected; B shifts left by len(A.char)
  - A.pos >= B.pos + B.len:  A shifts left by B.len; B unaffected
  - A.pos in [B.pos, B.pos+B.len): A becomes Insert(B.pos, char)
```

**Delete vs Delete (the gnarly one):**
```
Op A: Delete(pos=5, len=4)   // [5, 9)
Op B: Delete(pos=3, len=8)   // [3, 11)
                              // overlapping deletes — must subtract intersection

Compute:
  intersection = [max(5,3), min(9,11)) = [5, 9)  // length 4
  A' = Delete(pos=3, len=0)  // entire A range was already deleted by B → no-op
  B' = Delete(pos=3, len=4)  // B - intersection = (3,5) ∪ (9,11), but recomputed against A's effect
                                = Delete(3, 8 - 4) = Delete(3, 4)
```

**Format vs Insert / Format vs Delete:**
```
Op A: Format(range=[10,20), bold=true)
Op B: Insert(pos=12, "X")  // inside the range
A' = Format(range=[10,21), bold=true)   // grows by 1
B' = Insert(pos=12, "X")
   (and the inserted char inherits the bold attribute as a separate decision)

Op A: Format(range=[10,20), bold=true)
Op B: Delete(pos=15, len=3)  // shrinks the formatted range
A' = Format(range=[10,17), bold=true)
B' = Delete(pos=15, len=3)
```

This grows combinatorially. For a system supporting M op types, we need ~M² transformation functions. Google Docs has on the order of 30–50 op types, so several hundred transform functions. They're testable: pure functions of two ops, exhaustive property tests.

### The TP1 / TP2 properties

For OT to converge in *all* topologies of concurrent ops:
- **TP1**: For any two concurrent ops A, B starting from same doc: `T(B, A) ∘ A = T(A, B) ∘ B`. (Pairwise convergence.)
- **TP2**: For any three concurrent ops A, B, C: `T(C, A∘T(B,A)) = T(T(C,B), T(A,B))`. (Convergence across longer histories — basically associativity of transforms.)

**TP2 is hard.** For 30 years researchers found bugs in published transformation functions where TP2 didn't hold. The escape hatch: ensure the system topology is such that TP2 isn't needed.

### Server-arbitrated linear order: how Google Docs sidesteps TP2

If every op flows through a single server that assigns it a global sequence number, then:
- The server only ever has to transform an incoming op against a *linear chain* of already-applied ops.
- Each transform is pairwise (TP1 only).
- All clients receive the same linear sequence and apply the same transformations.

This is the **Jupiter algorithm** (originally from Xerox PARC). Google Docs is essentially Jupiter with industrial-strength ops.

### Client-side OT (interactive typing)

The user can't wait for the server ack to type the next character. Local responsiveness is the whole point. So the client has its own state machine:

```
Client state:
  rev:           last server revision the client has seen
  pending:       op submitted to server, not yet acked
  buffer:        ops the user typed AFTER pending, waiting their turn

User types:
  if pending == null:
    pending = thisOp
    send(thisOp)
  else:
    buffer = compose(buffer, thisOp)   // coalesce typing
  apply(thisOp) locally  // user sees instant feedback

Server sends remote op R (from another user):
  R' = transform(R, pending)
  R'' = transform(R', buffer)
  apply(R'') locally
  pending  = transform(pending, R)
  buffer   = transform(buffer, R)

Server acks our pending with rev N:
  rev = N
  if buffer != empty:
    pending = buffer
    buffer = empty
    send(pending)
  else:
    pending = null
```

This is the **client-side Jupiter algorithm**. It guarantees that:
1. The user sees their own keystrokes instantly (no round-trip).
2. Remote ops are correctly transformed against the user's not-yet-acked work.
3. The user's own ops, when finally acked, arrive at a server state that has incorporated other users' work.

**Latency property**: the user keystroke→pixel time is local (never depends on network). The remote-user keystroke→pixel time is one network round-trip + transform cost (microseconds).

**Tricky case**: the user's `pending` op had `baseRevision = R0` but by the time the server processes it, head is at `R7`. The server must transform `pending` against the 7 ops in between, then apply, then assign rev `R8`, then broadcast (the broadcast carries the transformed op, so other clients don't re-do this work).

### Compaction and the "rev never goes backward" rule

Server keeps an in-memory window of ops back to roughly `head - N` for some N. New connections at `baseRevision < head - N` get told "too far behind, fetch a snapshot." See §11.

### Concrete corner cases in OT

1. **Two clients delete overlapping ranges and one of them inserts inside the overlap.**
   - Order: A=Delete[3,10), B=Insert(7, "X"), composed concurrently from same base.
   - At server, A arrives first, gets rev N. B arrives next, base = N-1.
   - Transform B against A: `Insert(7, "X")` becomes `Insert(3, "X")` (collapsed to deletion point — debatable! Some systems prefer to *cancel* B in this case.)
   - **Decision**: Google Docs keeps B (insert wins over delete) — preserves the user's intent to add content. Inserted text appears at position 3.

2. **Format conflicts**: A makes range bold, B makes same range italic.
   - These are *commutative*; the result is bold + italic. Trivial.

3. **Format vs format on overlapping ranges with conflicting *values*** (e.g., A sets font-color=red, B sets font-color=blue on overlapping range).
   - Last-write-wins by server rev. Whichever op gets the later rev wins on the overlap. The losing op still applies to the *non*-overlapping part of its range.

4. **An op that arrives from a client whose `baseRevision` is older than head − retentionWindow.**
   - Server can no longer transform it (would need ops it has compacted away). Client is forced to refetch a snapshot. Op is rejected with code `STALE_BASE_REVISION`.
   - This is also the recovery path for clients that were offline >24 hr (default retention).

5. **Op that creates a paragraph break inside a list item.**
   - Block-level surgery. Op is `InsertBlockMarker(pos, type=ListItem)`. Transforms exactly as Insert with a special character.

6. **Undo (the trickiest)**.
   - Local undo must transform against ops that arrived since the user's original action.
   - The user expects "undo my last typing" — not "undo whatever happened to be at that position." The undo op stores the *original* op + its rev, then on undo is *transformed against intervening ops* and applied.
   - "Selective undo" (undo a specific op, not just last): doable but rare — Google Docs limits undo stack length and only allows linear undo.

7. **Op that arrives at the server out of order (TCP retransmit reordering).**
   - Each WebSocket message has `clientSeq`. Server enforces `clientSeq` is monotonic per `clientId`. Out-of-order or duplicate messages are dropped.

### Why OT is hard to get right

Every new op type expands the M² transform table. Every new transform must be proven (via property tests + theorem-prover-style verification) to satisfy TP1 against every existing op. A single buggy transform creates *silent divergence* — two users see different documents, with no error message. Google Docs has, over the years, accumulated thousands of test cases including pathological histories that they regression-test on every change.

---

## 8. CRDTs — Deep Dive (Logoot, RGA, LSEQ, Yjs)

CRDTs (Conflict-free Replicated Data Types) take a fundamentally different approach: instead of *transforming* ops, they make ops *commutative by construction*. Apply them in any order, you get the same answer.

For text, the trick is: characters don't have *positional indices*; they have *immutable position identifiers* drawn from a dense ordered space.

### The simple version: Logoot

Each character `c` is stored as `(position, c)` where `position` is a list of integers from a totally-ordered dense set.

```
Document: H e l l o
Positions:
  0.5     H
  0.7     e
  0.8     l
  0.9     l
  0.95    o
```

To insert "X" between "e" and "l": pick *any* position in (0.7, 0.8) — e.g. 0.75. Insertion is O(log n) (find the gap, pick a midpoint).

```
After inserting X between e and l:
  0.5    H
  0.7    e
  0.75   X         ← new
  0.8    l
  0.9    l
  0.95   o
```

Concurrent inserts at the "same" gap from two clients pick *different* identifiers (both add a sub-level), so they never collide.

```
Client 1 inserts "A" between e and l: position = 0.71
Client 2 inserts "B" between e and l: position = 0.73
After merge:
  0.5  0.7  0.71  0.73  0.8  0.9  0.95
   H    e    A     B    l    l    o
Result: "HeABllo" — both insertions present, deterministic order.
```

To break ties when two clients pick the same position (rare but possible), append `(siteId, clock)` to the position.

**Problem**: positions can grow unbounded. After 10⁶ inserts each between adjacent positions, the position identifiers are 10⁶-element arrays. Memory and bandwidth disaster.

### LSEQ — bounded interleaving

LSEQ improves Logoot by alternating between different allocation strategies (left-leaning vs right-leaning) at each tree depth. This keeps identifiers bounded in length for typical typing patterns. Still grows worst-case, but well-behaved in practice.

### RGA — Replicated Growable Array

RGA stores characters in a tree, where each character points to its predecessor:
```
Each char = (id, char, parent_id, isDeleted)
id = (siteId, clock)  // monotonically increasing per site
parent_id = id of the character it was inserted after
```

To insert "X" after "e": create new node `(s1, 17, "X")` with `parent_id = id_of("e")`. Even if Client 2 also inserts after "e" with `(s2, 4, "Y")`, both nodes are *children of e* in the tree. Order siblings by `(siteId, clock)` descending.

Linearization (to render):
```
walk tree in DFS, sorting children by (clock, siteId) DESC
(higher clock = "later", appears first in document, so newer insertions push older ones rightward)
```

Deletion is a tombstone: `isDeleted = true`. The node stays for reference but is hidden from rendering.

**Problem**: tombstones never get garbage collected without a global "everyone has seen this" guarantee. Long-lived docs accumulate tombstones forever. Yjs and Automerge tackle this with periodic *garbage collection rounds* using vector clocks (when all known sites have seen the deletion, drop the tombstone).

### Yjs (the most successful production CRDT)

Yjs uses a custom variant called *YATA* (Yet Another Transformation Approach). Combines RGA-like immutable IDs with O(1) batched updates and run-length-encoded item compression.

Key Yjs design points:
1. **Items are ranges, not characters**: a typing burst of "Hello" creates one item with 5 chars, not 5 items. Massively reduces overhead vs vanilla RGA.
2. **Each item has** `(clientId, clock_start, length, content, origin_left, origin_right, deleted)`. `origin_left/right` are the IDs of neighboring items at insertion time (not parents in a tree, but constraints).
3. **Conflict resolution at insertion**: when an item arrives, look for items with the same `origin_left` and `origin_right`. Sort by `clientId` to break ties.
4. **State Vector & Update protocol**: instead of replaying full history, peers exchange state vectors `{clientId → maxClock}` and request the diff.

### CRDT — Concrete corner cases

1. **Tombstones forever**: as above. Mitigate with periodic GC.
2. **Position identifier explosion**: avoid Logoot; use Yjs/RGA-style approaches with O(1) ID size.
3. **Causal delivery**: most CRDTs require ops to arrive *after* their causal dependencies. If client receives "delete char Z" before "insert char Z," it must buffer. State vector protocol handles this.
4. **Offline editing for hours, then merge**: works trivially. Each side's ops have unique IDs; merge = union of ops; final state = deterministic from union.
5. **Cursor positions are identities, not indices**: cursors anchored to a character ID survive any concurrent edit.
6. **Memory cost of full operation history**: docs over years can grow huge. Need snapshots + GC.

---

## 9. OT vs CRDT — The Real Trade-off

| Property | OT | CRDT |
|---|---|---|
| **Server required?** | Yes (linear arbiter) | No (peer-to-peer possible) |
| **Op size on wire** | Tiny: integer position + char | Larger: ID-tagged (8-32B per char) |
| **Memory in RAM** | Just the document | Document + tombstones + position metadata |
| **Code complexity** | Hundreds of transform fns; bug-prone | Algorithm complex but localized |
| **Convergence proof** | TP1+TP2; widely-known buggy in published forms | Proven by construction |
| **Offline edits** | Hard (long offline → giant rebase) | Easy (just merge ops) |
| **Scaling read-heavy** | Same | Slightly worse (overhead per char) |
| **Suitability for P2P** | Poor | Excellent |
| **Compaction** | Snapshot + replay from snapshot | Snapshot + GC tombstones |

### Why Google Docs uses OT

Historical reasons + product reasons:
1. **Google Docs predates production CRDTs** (it launched 2006 as Writely; CRDTs as a viable production technique date to ~2011).
2. **OT optimizes wire size** at the cost of server complexity. For a centralized service with massive user count, op size *dominates* total cost of egress.
3. **The single-server topology is inherent to Google Docs** — they have control plane needs (auth, ACLs, anti-abuse) that already require a centralized arbiter. Adding "and it's also the OT linearizer" is free.

### Why a greenfield product (e.g. a startup) might pick CRDT

1. **Multi-master replication** is much easier (no single linearizer per doc).
2. **Offline-first** UX is essentially free.
3. **Code review and audit** of a CRDT library is one-shot; OT requires maintaining a transform table forever.

### What we'd build today (the staff-level answer)

For *this* design we adopt OT for the wire protocol and the persistent op log (to match Google Docs reality), but we keep the architecture *capable of being swapped* to CRDT — i.e., the storage layer stores opaque ops, the OT-vs-CRDT choice lives in one library on each side. This optionality matters for §12 (offline) and for future evolution.

---

## 10. Cursor, Selection & Presence (Awareness Protocol)

Cursors are *not* document state. They:
- Should be visible to other users in real time.
- Should *survive* edits (a cursor at position 142 needs to follow when text is inserted before 142).
- Need not be durable (closing the tab erases it; that's correct).
- Should not flow through the OT pipeline (they're 5-Hz noise that would dominate traffic).

### Architecture

A *separate channel* per doc (or a separate message type on the same WebSocket). Backed by an in-memory pub/sub store with TTL — Redis Streams, or a Bigtable-internal pub/sub.

```
Client → Server (every 200 ms, debounced):
{ type: "cursor", docId, clientId, pos, selStart, selEnd, color, name, baseRev }

Server pub/sub fan-out: forward to all subscribers EXCEPT sender.
Server keeps in memory: { docId → { clientId → cursorState, expiresAt } }
TTL = 30 s. Disconnect (WebSocket close) immediately purges entry.
```

### Cursor stability across edits

Cursors carry a `baseRev`. When a remote cursor is received:
- If `baseRev == localRev`: render as-is.
- If `baseRev < localRev`: transform the cursor position through every op since `baseRev`. (Same transformation logic as ops, but only for positions.)
- If `baseRev > localRev`: queue until local rev catches up.

A cursor at position `p` with `baseRev=R0` arriving at a client at `localRev=R5`:
```
for each op in ops[R0..R5]:
  p = transformCursorPosition(p, op)
  // Insert(qpos, len): if qpos <= p: p += len
  // Delete(qpos, dlen): 
  //   if qpos+dlen <= p: p -= dlen
  //   else if qpos < p:  p = qpos  (collapse)
```

### Presence list and "who's here"

A doc tile needs to show "Alice, Bob, and 3 others viewing." This is a degraded form of cursor:
- A "viewer presence" message is sent on subscribe and every 30 s as heartbeat.
- Server aggregates into `{ docId → set<userId, color, name> }`.
- Other clients receive aggregated rosters, not individual cursor pings (much smaller).

### Scaling presence

20M active users × 5 Hz = 100M msg/sec is huge. Mitigations:
- **Coalesce in the edge POP**: instead of forwarding every cursor update, the POP collects 100ms of cursor updates per doc and sends one batch. Reduces by 5×.
- **Drop on slow consumer**: if a subscriber's send queue is backed up, drop oldest cursor messages (we're allowed to be lossy). Apply backpressure on op messages but never on cursor.
- **Sharded by doc**: each doc's presence lives in one shard of the pub/sub. Hot docs (Google all-hands deck with 5K viewers) need the shard split.

### Corner case: hot doc with 10,000 spectators

Some docs go viral. Internal company-wide announcements get 10K eyeballs. Naive fan-out is 10K × 5 Hz × N writers = mega-explosion. Solutions:
- Cap presence broadcast at 50 named cursors; show "+9,950 others" as a count.
- Demote spectators to "viewer" presence (heartbeat only, not cursor).
- Read-only viewers don't send cursor pings at all unless they actively select text.

---

## 11. Document Storage, Versioning & Snapshots

### The op log is the source of truth

Every op, once accepted by the session host, is appended to a per-document op log in **Spanner**. The log is the durable, replicated, monotonic record:

```
Spanner table: doc_ops
PRIMARY KEY (doc_id, revision)
columns: (doc_id, revision, op, author_id, client_id, client_seq, server_ts)
```

- Spanner replicates synchronously across 3+ regions before ack, giving us 11 9s durability.
- Per-doc partitioning means ops for one doc go to one Paxos group → linear order is automatic.
- Write rate per doc: ~10 ops/sec sustained even for hot docs (humans type slowly). Spanner handles this trivially.

The session host writes the op to Spanner *before* broadcasting to subscribers. Why? Because if we broadcast first, then crash before writing, peers have an op the source-of-truth doesn't. The protocol guarantees: *a broadcasted op has been durably committed*.

### Snapshots — why and how

Replaying 5 million ops to open a popular doc is unacceptable. So we periodically materialize the document state as a **snapshot**.

```
Bigtable / Blob store:
  snapshot_table:
    PRIMARY KEY (doc_id, snapshot_revision)
    columns: (doc_id, snapshot_revision, content_blob, created_at)
```

Snapshot policy:
- Async background job (per session host, ratelimited) creates a snapshot every K ops (e.g., K=1000) or every T minutes (e.g., T=10).
- Snapshot stores the full materialized document at that revision.
- Loading a doc: fetch latest snapshot ≤ requested revision, then apply ops from `snapshot.revision` to head. Bounded replay (≤ K ops).

**Snapshot creation is racy if live ops are still arriving**. Two approaches:
1. **Pause the host**: stop accepting ops for ~10 ms while a copy-on-write snapshot is taken. Disruptive at scale.
2. **Snapshot at a specific revision**: in the host's memory, mark "snapshot of rev 8470 in progress." Continue accepting ops as rev 8471, 8472, etc. The snapshot job snapshots the state *as of 8470* (it can compute this by replaying log from previous snapshot). Non-disruptive. **We use this**.

### Versioning ("File → Version history")

Distinct from snapshots. Snapshots are *internal* (any rev). Versions are *user-visible* and have:
- Auto-versioning: every 30 minutes if the doc was edited.
- Named versions: user explicitly says "save as version X".
- Open/close versioning: when the user closes all editors and the doc has been idle for >5 min, take a final version.

Versions are stored in **Cloud Storage Coldline** (cheap, slow). Indexed in metadata DB.

```
versions_table:
  PRIMARY KEY (doc_id, version_id)
  columns: (doc_id, version_id, name, created_by, created_at, snapshot_url, parent_version_id)
```

Restore: a "restore" creates a *new revision* whose op is "set document content to version X." This is intentionally a forward-only operation — never rewrites history. Doc revision counter only goes up.

### Compaction

Old ops eventually compact:
- After a snapshot at rev R, ops `[..., R)` can be deleted from hot Spanner if:
  - All connected clients have `baseRev >= R`.
  - Versions retained for legal hold (e.g., GDPR-mandated 1 yr) are preserved in cold storage.
- Compaction runs nightly per-doc, batched.

### Per-character authorship (blame)

For each character, we want to know: who typed this? Stored as a sparse run-length-encoded attribute:

```
authorship_runs: [(0, 100, alice), (100, 250, bob), (250, 280, alice), ...]
```

When an op inserts characters by author X at position p of length n:
- Find the run containing p; if its author is X, just extend the run; else split the run and insert a new run.
- O(log n) with a balanced tree.

Stored alongside the snapshot. Reconstructable from ops.

### Storage tiering

```
Hot tier (Spanner):           live op logs, retained ~30 days for fast random access
Warm tier (Bigtable / blob):  snapshots, retained ~1 year
Cold tier (GCS Coldline):     versions, retained per legal/policy

Read paths:
  open doc → fetch latest hot snapshot + replay ops (fast: 50-200 ms)
  view old version → fetch from Coldline (slow: 1-5 s, OK for explicit user action)
```

---

## 12. Offline Sync, Reconciliation & Long-Disconnect Handling

### Short-disconnect (seconds to minutes)

Client buffers outbound ops. On reconnect:
- Replay buffered ops to server, in order.
- Server transforms each against ops it has accepted in the meantime.
- Client receives `catchup` of remote ops.
- All transforms happen normally.

If buffered ops' `baseRevision` is older than server's compaction window (default: keep ops 24 hr), see "long-disconnect" path.

### Long-disconnect (hours to weeks)

User goes hiking, comes back, opens a doc edited in the meantime by colleagues. Their local edits exist; server has moved on far past their `baseRevision`.

**OT path** (what real Google Docs does): if `baseRevision < head - 24hr-of-ops`, the server can no longer transform the client's pending ops directly — too many transforms. Instead:
1. Server computes a snapshot of the doc at the client's `baseRevision`.
2. Server diffs `snapshot(client.baseRev)` against `snapshot(head)` to get a synthesized "remote change" — a sequence of inserts/deletes.
3. Client applies this synthesized op as if it were a normal remote op, transforming pending local ops against it.
4. Now the client is at head, with pending ops correctly transformed.
5. Client resubmits.

This loses some semantic intent (the synthesized op merges all collaborators' work into one mega-op), but converges. If two collaborators *both* edited the same line concurrently with the offline user, the synthesized op shows the *result* of their merge, not their individual ops. Acceptable.

**CRDT path** would just merge — every op has a unique ID, no transformation needed. Simpler. (One reason CRDT advocates exist.)

### "I edited offline. So did 3 colleagues. Did anyone's work get lost?"

No, but with caveats:
- Inserts: never lost (positions are linearizable; everyone's text ends up somewhere).
- Deletes: a delete by user A of text that user B was concurrently editing might delete characters that B re-typed; B's re-typed text survives, A's delete applies only to overlap.
- Format changes: last-write-wins on attribute conflicts (color X by A, color Y by B → whoever's op was applied later wins — semantic loss is small).

### Local persistence

Client persists its op buffer + local snapshot to IndexedDB. If the user closes the browser before reconnect, their offline edits are not lost.

```
IndexedDB schema:
  store "docs": { docId → { snapshot, baseRev, pendingOps[], lastSync } }
```

On startup, the client:
1. Loads snapshot + applies pending ops (showing the user the doc as they last saw it).
2. Connects to server.
3. Sends its `baseRev` and pending ops.

### Multi-device sync

Same user editing the same doc on phone + laptop. Each device has its own `clientId`. They both connect to the same session host. Their ops interleave and transform like any other concurrent collaborators. The user's *local mirror* on each device converges.

**Subtle bug**: undo on device A doesn't see the user's typing on device B. Undo stack is per-device. We accept this — explaining "undo is local to this device" via UX is fine.

---

## 13. Comments, Suggestions & Track Changes

### Anchored comments

A comment lives outside the document text but *points* to a range:
```
Comment {
  id, author, text, range_start_id, range_end_id, threadId
}
```

`range_start_id` / `range_end_id` are immutable character IDs (or, in OT-land, position-with-rev tuples that the system rebases as ops accumulate).

When ops modify the text:
- If both range endpoints are inside a deleted region: comment becomes "orphaned" — kept in metadata, hidden from doc, surfaced in "resolved comments" history.
- If only one endpoint is deleted: the surviving endpoint snaps to the deletion boundary; comment continues to anchor.
- If text is inserted between endpoints: comment range grows.

Storage: separate `comments_table` keyed by `(doc_id, comment_id)`. Not in the op log (different lifecycle: comments are added/resolved/deleted independently of doc edits).

### Suggestions (track changes)

A suggestion is a *proposed* op that has been recorded but not yet applied.
```
Suggestion {
  id, author, originalOp, status: PENDING | ACCEPTED | REJECTED, threadId
}
```

The visual: suggestions display as overlays — proposed insertions in green, proposed deletions in strikethrough red. They're rendered on top of the canonical document.

When accepted, the suggestion's `originalOp` is *transformed* against the current head (it may be stale by the time someone accepts) and applied as a normal op.

### Real-time requirement

Comments and suggestions sync in real time same as ops, but on a separate channel/op-type. They use OT-like rebasing only for their *anchor positions*; the comment text itself is not collaborative-edited (one author per comment).

### Notifications

@-mentions in comments fire notifications via a separate **notification service** (email, mobile push, in-app). Async via a Pub/Sub queue. Loss is tolerable (not durable system of record for the notification — the comment itself is).

---

## 14. Permissions, Sharing & ACL Propagation

### Model

Every doc has an ACL:
```
ACL {
  ownerId,
  shares: List<Share>,
  linkSharing: { mode: NONE | RESTRICTED | DOMAIN | ANYONE, role: VIEWER|COMMENTER|EDITOR }
}

Share {
  principalId,        // user OR group OR domain
  role: OWNER|EDITOR|COMMENTER|VIEWER,
  inheritedFromFolder?: folderId
}
```

### Storage

ACLs live in a Zanzibar-style relationship database (Google's actual production system for fine-grained authorization at planetary scale). Conceptually:

```
relations:
  (object: doc:abc, relation: editor, subject: user:alice)
  (object: doc:abc, relation: editor, subject: group:eng@google.com)
  (object: folder:foo, relation: editor, subject: user:bob)
  
inheritance rules:
  doc:abc#parent = folder:foo  → bob (folder editor) is also doc editor
```

Why Zanzibar:
- Hierarchical permission inheritance (folder permissions cascade).
- Sub-millisecond ACL checks at planet scale.
- Group expansion (a doc shared with `eng@` covers 50K people).
- Caveats and consistency tokens (Zookies) so the user-perceived order of "share with X" then "X opens doc" is correct.

### Permission checks on the hot path

Every op submission requires a permission check: "is this user an EDITOR on this doc?" This is on the order of 2-5M ops/sec. We can't hit Zanzibar for every one.

**Solution**: when a user joins a doc session, the session host fetches the user's role *once* and caches it for the duration of the session, with a TTL of 60 s and a *change-notification* subscription. If Zanzibar's relationship changes (the user is unshared), the host gets a callback within ~1 s and:
1. Boots the user from the WebSocket.
2. Their pending ops in flight are dropped.
3. The next reconnection fails the permission check.

### Real-time revocation (the corner case)

An admin un-shares the doc with Alice while Alice is editing. We need:
- Alice's currently-pending op gets rejected (not silently applied).
- Alice's WebSocket session closes within 1 s.
- Alice's local cache of the doc is purged (or marked read-only with a banner: "you no longer have access").

Implementation:
- ACL service publishes a "ACL changed for doc D" event to the session host.
- Host re-evaluates each connected user's role.
- Hosts rejected users.
- Hosts emits a control message: client closes, deletes cached snapshot from IndexedDB, shows an error UI.

### Public link sharing

Link sharing implies a *capability token* embedded in the URL. The token is opaque, server-side it maps to `(docId, role, optionalDomainConstraint, optionalExpiry)`.

Anti-abuse:
- Tokens revocable by changing the doc's link-sharing mode.
- Anonymous users (link-share to "anyone") get a synthetic ID for the session, used in OT but not persistable.
- Rate limits per anonymous IP to prevent enumeration.

### Domain restrictions

Enterprise: doc shared "anyone in google.com." Enforced by checking the user's email domain against the doc's `linkSharing.domain` constraint. Authentication does the heavy lifting.

---

## 15. Search Indexing & Full-Text Discovery

### Architecture

```
Doc edit committed → Async index-update event → Indexing pipeline → Inverted index
                                                                            ↓
                                                                      Search query
```

This is a *write-fast, search-eventually-consistent* design. Latency budget: search lag from edit → searchable is < 60 s for most docs, < 5 min for tail.

### Pipeline

1. Op committed; session host emits `(docId, revision)` to a Pub/Sub.
2. Indexing workers consume.
3. Worker fetches latest snapshot of the doc, extracts plain text, computes fingerprints, updates inverted index.
4. Index is sharded by `userId` (since search is per-user).

### Why per-user shards

Each user has access to a different set of docs (their own + shared). Searching "all docs Alice has access to" naively requires joining doc set with permission set. Instead:

For each `(userId, docId)` permission edge, write an entry to the per-user index. A doc shared with 50K people creates 50K index entries. Bigger storage, but search is a simple lookup.

For very large groups (`eng@google.com`, 50K), we use a *deferred-fan-out* strategy: index is keyed by user *or* group; query joins both. Avoids the 50K-write blowup.

### Search index implementation

- **Inverted index**: term → docList. Stored in a sharded LSM-like structure.
- **Bloom filters per shard**: fast "is this term anywhere in this shard" check.
- **Position lists**: enables phrase queries.
- **Trigrams** for prefix/substring search ("docu*").

### Find-and-replace within doc

Different beast. Real-time, in-memory only; no index needed. Done in the client against the local document mirror. Bulk replace generates a sequence of ops submitted normally.

---

## 16. Real-Time Communication Layer (WebSockets at scale)

### Why WebSockets (not HTTP polling, not SSE)

- Bidirectional (client sends ops, server sends remote ops + presence).
- Long-lived → lower overhead per message.
- HTTP/2/3 multiplexing on top of WebSocket frames keeps it cheap.

### Connection lifecycle

```
1. Client opens WSS to nearest edge POP.
2. Edge auths the JWT, looks up doc → session host mapping (Cassandra-backed routing table).
3. Edge opens a backend connection to the session host (or reuses pooled).
4. Edge multiplexes the client's per-doc sub-streams to the right session host.
5. On disconnect, the edge tears down backend subscription after a 30s grace (so brief network blips don't shake out the session).
```

### Sticky routing

For each `(docId)`, all clients must talk to the same session host. The routing table is a Cassandra/etcd cluster:
```
doc_routing:
  PK doc_id
  columns: (host_addr, lease_until)
```

The first edge to handle an op for a doc tries to "claim" the host assignment; if no claim exists or the lease is expired, it picks a host (consistent hashing, balanced) and writes the entry with a 60s lease. The session host extends the lease while it's serving.

### Failover

Session host dies. Detected by:
- Heartbeat to Cassandra fails (lease expires).
- Edge POP's TCP connection RST.

Recovery:
- A new host is elected (consistent hashing fallback or controller-driven assignment).
- New host loads the doc state: latest snapshot from blob store + replay ops from Spanner since snapshot.
- New host's address written to routing table.
- Edges reconnect their backend connection.
- Clients see a brief blip (~1-2 s) but their WebSocket *to the edge* never closed; they don't see disconnect UX.

For warm failover, we can replicate the session host's in-memory state (op log tail + presence) to a hot standby within the same region. Failover then takes ~100 ms. This is what we'd build for premium customers.

### Protocol resilience

Every message has an `outboundSeq` from the server's perspective. If a client detects a gap (it gets seq 100, then 102), it requests a replay of the missing range. Server keeps a small buffer (e.g. last 1000 messages per subscriber) for this.

For reconnection: client sends `(docId, lastSeen=X)`; server replays from X. If X is too old, client refetches snapshot.

### Backpressure

Slow clients — what to do?

- Each connection has a per-connection outbound queue.
- If queue exceeds, e.g., 1 MB or 1000 messages: server stops sending *new* messages and requests the client to disconnect.
- On reconnect, client refetches from snapshot. (We never block other clients on a slow one.)

For presence specifically: drop oldest, never block. Presence is best-effort.

For ops: never drop. If the queue fills, kill the connection.

---

## 17. Database & Storage Architecture

```
Concern                      Store                       Key justification
-----                        -----                       ------
Op log (per doc)             Spanner                     Linear order, sync replication, regional commit, schema flexibility
Document metadata            Spanner                     Strong consistency for ACL changes
Snapshots (warm)             Bigtable                    High write throughput, low cost, sparse rows
Versions (cold)              Cloud Storage Coldline      Cheap, infrequent reads
Permissions / ACLs           Zanzibar (Spanner-backed)   Group expansion, hierarchy, planet-scale
Search index                 Custom (sharded LSM)        Per-user inverted index
Routing table                Cassandra/etcd              Fast read, eventual consistency OK
Presence                     Redis Streams or Bigtable PubSub  Lossy, in-memory, high throughput
Notifications queue          Pub/Sub                     Durable async fan-out
```

### Why Spanner for op log?

- **External consistency**: Spanner's TrueTime gives us global linearizable order at high throughput. Critical for op sequence numbers.
- **Regional replication**: 3+ replicas in different regions; commit only after majority.
- **Schema**: typed columns for op fields (type, pos, payload, author, ts) → easy querying for analytics/blame.

### Why not Cassandra/DynamoDB for op log?

- Lacks linearizable order across regions without expensive coordination.
- `LWT` (lightweight transactions) at Cassandra are slow and per-partition only — suitable for routing table, not hot op log.

### Sharding & partitioning

- **Spanner**: docs are auto-partitioned by primary key `(doc_id)`. Hot docs land on one shard → ops/sec for one doc are small (humans type slowly), so this is fine.
- **Bigtable snapshots**: row key = `(doc_id, snapshot_revision)`. Range scans for "give me the latest snapshot ≤ R" work efficiently.
- **Search**: hashed on `userId`. 10K shards. Hot users are rare; we re-shard offenders manually.

### Backups

- Spanner: point-in-time recovery, 7-day window.
- Bigtable snapshots → daily backup to GCS, 90-day retention.
- Cold tier: replicated across 3 regions automatically.

### Soft deletes

`DELETE` of a doc is a soft delete: doc's metadata gets `deletedAt`, ops/snapshots remain. Hard delete after 30-day trash period. GDPR right-to-be-forgotten triggers hard delete + scrubbing.

---

## 18. Caching Architecture

```
Layer                What's cached                                    TTL / invalidation
-----                -------------                                    ------------------
Browser              Doc snapshot in IndexedDB                        Until logout or quota pressure
CDN                  Static assets (JS, fonts)                        1 year, immutable hashes
Edge POP             Auth token validation results                    Token expiry
Session host RAM     Active doc state (snapshot + tail of ops)        Until host dies / 5 min idle
Hot snapshot cache   Recently-loaded snapshots (Bigtable + memcached) 1 hour
ACL cache            Per-session user role                            60 s + invalidation pubsub
Search index         Hot terms                                        Tiered, LRU
Routing table        doc → host                                       60 s lease, refreshed on use
```

### The doc state cache (most interesting)

Each session host keeps the *full document state* + an in-memory tail of recent ops (e.g. last 5K). Why a tail?
- Catch-up requests from clients with `baseRev` < head can be served from RAM.
- Beyond the tail, fall back to Spanner read.

LRU eviction: if a doc has no subscribers for 5 minutes, it's evicted from the host's memory; routing entry expires; next client to connect causes re-load from storage.

### Memcached layer (optional but cheap)

For "open doc" → snapshot fetch, we cache the latest snapshot in memcached:
```
key: "doc:abc:snapshot:latest"
value: blob URL + revision
ttl: 1 h, invalidated on new snapshot
```

Why? Bigtable can serve snapshot reads at 10ms p50, but at 1M docs/sec opening at peak we'd flood it. Memcached bumps that to 1ms. ROI is real.

### What we *don't* cache

- The op log: every op needs to be durable in Spanner before broadcast.
- Permissions for a *write*: always re-check on session-host with cached role; cache invalidation guarantees the bound is bounded.

---

## 19. Scalability Deep Dive

### Vertical limits

- **One session host serves how many docs?**
  - RAM: 64 GB host with ~10 KB doc state avg = 6M docs in RAM theoretical, but realistically 100K–500K active docs per host (overhead, op tail, presence, GC).
  - CPU: each op transformation is O(N) where N = ops in tail (bounded ~1000) → ~50 µs per op. Host can handle ~20K ops/sec = 200K active editors.
- **One doc on one host scaling**: at 100 collaborators, ops/sec = 100. Trivial. Bottleneck is fan-out: 100 collaborators × 100 ops/sec = 10K outbound msgs/sec from the host for this one doc. Still fine.

### Horizontal scaling

- **Per-doc sharding**: each doc lives on one host at a time. Cluster grows by adding hosts; routing table assigns new docs to least-loaded host (consistent hashing biased by load).
- **Rebalancing**: hot docs (>1K active editors) get monitored. If a host's load > 80%, the controller migrates *cold* docs off it. Hot docs are never migrated mid-edit (would interrupt sessions); they stay until natural cooldown.
- **Failover scaling**: when a host dies, its docs distribute across surviving hosts via routing table updates. Each surviving host loads ~1/(N-1) more docs from storage.

### Multi-region

```
US-EAST  ─────┐
              │
US-WEST  ────┼─── Spanner global instance (replicated synchronously)
              │
EU-WEST  ────┘
```

- Spanner replication is synchronous across 3 regions for the op log. Latency cost: writes commit at the speed of the slowest region's network round-trip (~100 ms intercontinental).
- Session hosts run in *every* region. A doc's session host is in the region where the *first* connecting user lives. Other regions' users connect to that host (across-region WebSocket).

**Trade-off**: cross-region WebSocket adds 50-150 ms one-way latency. For a doc shared between EU and US users, the user farther from the host pays. Acceptable; if it's not, premium customers can pin docs to a specific region or use a special "multi-region active-active" mode (requires CRDTs — see §9).

### Fan-out scaling for hot docs

Doc with 5K viewers (live demo, all-hands): op rate is small (only the presenter types) but fan-out is 5K × N = lots. The session host can saturate its NIC. Solution:

- **Hierarchical fan-out**: session host emits to a *regional fanout layer* (10K connections), which in turn emits to clients. Like CDN tiering.
- Edge POPs already do this naturally — clients connect to edge, edge fans out from one upstream backend connection to many client connections.

### Latency budget

| Operation | Target p99 |
|---|---|
| Local keystroke → echo on screen | <16 ms (one frame) |
| Remote keystroke → render on screen | <250 ms |
| Open a fresh doc | <1500 ms |
| Open a 100-page doc | <2500 ms |
| Search query | <200 ms |
| Permission revocation propagation | <1000 ms |

### Cost knobs

- **Reduce snapshot frequency** for cold docs (less Bigtable cost).
- **Reduce op log retention in Spanner** to 7 days, push to cold (cheaper).
- **Share session hosts** across many small docs.
- **Aggressive presence coalescing** (less network).
- **Prefer cold-tier Cloud Storage** for old versions.

---

## 20. Reliability & Fault Tolerance Deep Dive

### Failure modes & responses

| Failure | Detection | Response |
|---|---|---|
| Session host crash | Lease expiry, healthcheck | Failover; new host loads from storage |
| Spanner regional outage | Regional unhealthy | Fallback to other regions; reads continue, writes blocked in that region |
| Edge POP dies | Health check | Anycast routes traffic to next-nearest POP |
| Network partition | Healthcheck timeouts | CAP: prefer Consistency for ops (block writes), serve stale reads |
| Bigtable degraded | Latency spike | Snapshot reads fall back to op replay (slower but works) |
| Pub/sub presence outage | Health check | Presence degrades to "missing" UX; edits unaffected |
| Auth service outage | Health check | Tolerate up to JWT TTL (~1 hr); after that, new logins fail |
| Bug deploy: corrupted ops | Anomaly detection (op length spike, content hash mismatch) | Rollback session host fleet; ops in-flight are quarantined |

### Idempotency

Every op has `(clientId, clientSeq)`. Server's persistent dedup cache (last 60 s of seqs per client). Retries are safe.

### Op log corruption — how do we recover?

The persistent op log is replicated 3-way in Spanner with checksums. Corruption of a Spanner replica is auto-healed.

What if a *bug* in the session host produces a bad op (semantically corrupt — e.g., pos = 10⁹)? It would:
1. Get rejected by the server-side validator (op-shape check before write).
2. If somehow it slips through, peers' clients would either crash or see weird state.

Defense in depth:
- **Validator** in session host before write: type check, position bounds check, length sanity, max op size cap.
- **Schema-tagged ops**: each op has version; downstream code rejects unknown shapes.
- **Replay-and-snapshot consistency check**: nightly job replays the op log of a sample of docs and compares to existing snapshot. Mismatch → page on-call.

### Catastrophic failure: doc gets corrupted at character level

Worst case: a sequence of ops convinces the OT engine to produce a character sequence different from what users intend. (Theoretical with TP1; mostly happens via bug.)

Recovery:
1. Doc is rolled back to last *known good* version (from version history).
2. User-facing message: "We restored to a previous version due to an error. Recent edits since [time] are recoverable in version history."
3. Lost work is in the op log; we offer a tool to manually copy-paste from the log.

This is rare (last published incident: probably never publicly), but the design must enable recovery.

### Spanner write failure mid-broadcast

Session host is about to write op N to Spanner, then broadcast. Spanner write fails (e.g., regional outage):
- Don't broadcast.
- Send NACK to author client.
- Client retries the op (with same `clientSeq`, dedup-safe).
- If Spanner is durably down, host enters "read-only mode" — returns errors to writers, allows reads.

### Thundering herds

A regional outage recovers; 1M clients reconnect simultaneously.
- Edge POP rate-limits new connections.
- Reconnect with jitter: client backoff is `base × 2^attempt + uniform(0, base)`.
- Session hosts ramp up gradually (warm-up period during which they accept slower).

---

## 21. Availability Deep Dive

### Target: 99.99% (52 min downtime/year)

Achieving this requires:
1. **No single point of failure**: every component is replicated.
2. **Graceful degradation**: partial outages don't take down everything.
3. **Fast detection + recovery**: outages are noticed in seconds, recovered in minutes.

### Active-active multi-region

For *reads* (open doc, view version history): yes, every region serves.

For *writes* (edit ops): traditionally pinned to the doc's primary region. The downside: if the primary region is down, the doc is read-only globally for ~1-2 minutes during failover.

**Premium / enterprise tier**: optionally enable *multi-region active write*. Requires:
- CRDTs (so writes can happen in any region and merge later).
- Or, tighter coordination via Spanner's external consistency (paying cross-region latency on every write).

For most users, single-region writes are fine. The cost-benefit of multi-region active is bad for the median doc.

### Graceful degradation

| Component fails | Behavior |
|---|---|
| Search service | Search returns "temporarily unavailable"; doc editing still works |
| ACL service | Cached ACLs persist 60s; new shares delayed; existing sessions continue |
| Spanner (regional) | Reads served from other regions; writes blocked in failed region; users in that region see "saving..." until failover |
| Bigtable snapshots | Open-doc latency increases (full op replay); nothing breaks |
| Pub/Sub presence | Presence indicators disappear; doc editing fully functional |
| Notifications | @-mention emails delayed; doc unaffected |

The principle: each component fails *independently*, and the user's *core flow* (typing into a doc) survives more than one component failure.

### Deployment safety

- **Canary**: every change rolls to <0.1% of traffic for 1 hour. Anomaly detection auto-rolls back.
- **Region-by-region**: a bad deploy can take down at most one region.
- **Feature flags**: every new feature behind a flag; can be killed instantly without redeploy.
- **Backwards compatibility for op formats**: never break wire format; add new fields, never remove.

### Maintenance

- Spanner / Bigtable upgrades are zero-downtime.
- Session host upgrades use connection draining: stop accepting new docs, let existing sessions finish or migrate, then drain.

### Disaster recovery

- 3-region replication of all durable data.
- DR test (full region failover) runs quarterly.
- RTO: 5 min for collaboration, 30 min for search.
- RPO: 0 for op log (sync replication), <1 min for snapshots, <5 min for search.

---

## 22. Observability & Operational Excellence

### Metrics (RED + USE)

**RED** (Rate / Errors / Duration):
- ops/sec (per-host, per-region, global)
- op rejection rate (per reason: stale base, ACL fail, validation fail)
- op apply latency (server-side OT transform + log write)
- WebSocket connection rate, drop rate
- doc open latency p50/p95/p99

**USE** (Utilization / Saturation / Errors) per system:
- session host CPU, RAM, NIC throughput
- Spanner write throughput vs quota
- Bigtable read latency
- routing table size + miss rate
- pub/sub backlog (per topic)

### Structured logging

Every op carries `traceId` from the client. Every server hop logs with that traceId. Trace UI ties them together. Critical for debugging "why did Alice's edit not show up for Bob?"

### Anomaly detection

- Sudden spike in op rejection rate → deploy regression?
- Op size outliers → corrupt client?
- Specific doc seeing 10x normal ops → abuse / bug?

### Alerts

Tiered:
- **P0** (page now): availability < 99.9% in last 5 min; op error rate > 1%; Spanner unavailable.
- **P1** (page during day): latency p99 > 2× normal for 15 min; growing pub/sub backlog.
- **P2** (ticket): per-host imbalance; hot doc detected.

### Audit log

Permission changes, doc deletions, restore operations — go into an immutable audit log (separate Spanner table with write-once API). Required for compliance (SOC 2, ISO 27001, GDPR).

### Synthetic monitoring

Continuous bots open a test doc in every region, type, save, restore, and verify content. Catches end-to-end issues that metrics miss.

---

## 23. Corner Cases & Hard Problems

### 23.1. Two users insert text at the same position simultaneously

Already covered (§7): tiebreak by `(authorId, clientSeq)` lexicographic. Both characters appear; order is deterministic; nobody loses work.

### 23.2. User pastes a 10 MB block of text

A single op carrying 10 MB would block the session host and exhaust Spanner write limits.

Solution:
- Client splits paste into ops of max size (e.g. 64 KB) and submits sequentially.
- Server enforces per-op cap; oversized ops rejected.
- For images/embeds: upload to blob storage first, then op references the blob ID (so the op stays small).

### 23.3. User pastes a complex document with formatting

Many ops generated, sequenced. Each op transforms against any concurrent ops. Net effect: paste appears at the destination. Feels instant to the user (local apply is instant).

### 23.4. Doc with 50,000 edits per minute (rare but possible)

Probably abuse or bot. Throttle:
- Per-user rate limit (200 ops/sec/user per doc).
- Per-doc rate limit (10K ops/sec/doc) → drop or reject excess.
- Anti-abuse signal: account flagged for review.

### 23.5. Long-running offline edit creates a conflict the user must resolve

Strict CRDT/OT merging cannot ask the user. So we *don't*. Best-effort merge always succeeds. User can review version history if needed.

### 23.6. Document grows so large it can't be loaded

- Hard cap: 50 MB serialized.
- Soft warning at 25 MB: "performance may degrade."
- Writing past the cap: rejected with a friendly error pointing to "split into multiple docs."

### 23.7. Image / embed lifecycle

- Upload: client uploads to GCS first, gets a URL; the op references the URL.
- On op transform/delete: the URL stays in the embed table even if the embed is removed (tombstoned).
- GC: nightly job collects URLs unreferenced by any version of any doc and deletes the blob.

### 23.8. Unicode normalization

User types `é` (a single codepoint). Other user types `e` followed by combining acute (two codepoints). They look identical; positions diverge.

We **normalize at the input boundary** (NFC) before generating ops. Position semantics are stable. Surrogate pairs (emoji) count as 2 UTF-16 code units → ops use UTF-16 code unit positions consistently.

### 23.9. Right-to-left text and bidirectional rendering

Doesn't affect ops (positions are still character-index). Affects rendering only. Cursor placement on RTL text uses logical position internally, visual position for screen.

### 23.10. User in two tabs editing same doc

Each tab is a separate `clientId`. Each has its own pending buffer. Ops interleave at the server like any concurrent collaborators. Convergent. Slight UX wart: undo in tab A doesn't see typing in tab B.

### 23.11. Browser closes mid-flight; recovery on reopen

Pending ops are persisted to IndexedDB before send. On reopen:
- Reconnect.
- Replay pending ops (server dedupes by `clientSeq`).
- Resume.

### 23.12. Server clock skew

We do NOT use wall-clock for ordering. Op order is determined by Spanner's TrueTime / Lamport-ish per-doc revision counter, not host clocks. Skew is irrelevant to correctness.

### 23.13. Doc deletion while users are editing

Delete is a metadata change. Live editors see a control message: "this doc has been deleted." Their pending ops are dropped. Local cache cleared.

If owner accidentally deletes: 30-day trash; restore brings the doc back at its current state (op log + snapshots are preserved during trash).

### 23.14. Large org broadcasts a doc to 100K people

We never send 100K live cursor pings. Top-N rule (§10). Ops don't fan out to passive viewers in real-time after a threshold; we degrade to "polled refresh every 5s" for N>10K.

### 23.15. Forking a document

User says "make a copy." A new doc is created; its initial state is a snapshot of the source. Op log starts fresh. ACLs do *not* inherit (new doc has same owner; sharing is reset). This is intentional — copying inherits content but not access.

### 23.16. User opens version history during active editing

Version history is a read-only view. It fetches the version blob from cold storage, renders it. Live editors continue. The viewing user sees a "view-only" banner. Closing version-history returns to live edit.

### 23.17. Restoring a version

Creates a *new revision* whose op is "set content to version X." Live editors see this as a giant remote op. Their pending local ops transform against it (often degrading to "lost" in the Diff sense — but at this point the user already opted in to "restore", so the loss is intentional).

### 23.18. Deletion of a user account

User's docs:
- Owned by them: handled by org policy (transfer ownership, or delete).
- Shared with them: their share entry removed.

Op log retains the user's `authorId` indefinitely (audit). For GDPR right-to-be-forgotten, the `authorId` is replaced with an anonymized ID, and personal info elsewhere scrubbed.

### 23.19. Two users each type a Slash command "/table" simultaneously

The slash command UI is *client-side*. When the user picks "Insert Table 3x3," it generates a sequence of ops (insert table-start marker, 9 cells, table-end marker). These are normal ops, transformed normally. If both users do this at the same position, both tables end up in the doc — surprising but not lossy.

UX mitigation: if a slash menu is open, listen for incoming ops; if remote ops appear at the cursor location, dismiss the slash menu and re-anchor.

### 23.20. The op that breaks OT

Any new op type must come with proven transform functions vs every existing op type. Adding ops requires:
1. Specification of the op semantically.
2. Definition of `transform(thisOp, X)` and `transform(X, thisOp)` for every X.
3. Property tests: random sequences of mixed ops on N peers must converge.
4. Code review by OT specialist.
5. Canary deploy.

This is *the* engineering bottleneck for evolving Google Docs. It's why new features come slowly and old features are stable.

### 23.21. The dreaded "doc out of sync" message

Pathological cases (rare) where client and server diverge despite OT. Detection:
- Each rev carries a content hash. Client periodically (every 100 ops) sends its current hash with the next op. Server compares.
- Mismatch → refusal of new ops; client refreshes from snapshot.

This is the safety net for "you must never silently diverge."

### 23.22. Massive concurrent writes from one user (Find & Replace All)

User does Find/Replace "the" → "X" on a 100-page doc with 5000 occurrences. That's 5000 ops.

- Client batches into a single *composite op* (or a small number of large ops) using server-side bulk-replace API.
- Server applies in one transaction (or marker), broadcasts as a single large op.
- Other clients apply via single transform, not 5000.

### 23.23. Real-time spell-check / grammar suggestions

Run client-side, never enter the op log. Suggestions are UI-only until the user accepts (which generates a normal op).

### 23.24. Fuzz testing & adversarial input

A malicious client could craft ops to crash other clients. Defenses:
- Server validates every op (shape, bounds, size).
- Client validates incoming ops similarly (defense-in-depth: a server bug shouldn't let a client crash).
- Sandbox embeds (images, equations) in iframes / restricted scopes.

### 23.25. The "cursor at end" problem after concurrent insert

Common UX bug: user A is typing at end of doc, user B inserts a paragraph above them. After OT, A's position should still be "at end." If we naïvely transform `cursor.pos` by adding length of B's insert, A is *not* at end visually if the insert pushed content down — but logically yes. Renderer handles the visual layout.

### 23.26. Editing while subscribed to N other docs

Each doc has its own state, its own pending buffer. Browser memory budget (~500 MB) limits to ~50 actively cached docs. Beyond that, evict least-recently-used; reload from server when revisited.

### 23.27. Session host overload from a single hot doc

A meeting collaboration doc with 500 active editors all typing. Op throughput might saturate the host. Mitigations:
- Coalesce typing ops at the client (every 100ms, batch a typing run into one op with N characters).
- Hierarchical fan-out: edge POPs aggregate near subscribers.
- Premium/enterprise quota for hot docs (more headroom).

### 23.28. Network partition: client thinks they're online but server can't reach them

TCP keepalive + WebSocket ping/pong. If 90 s without pong, force-close and reconnect.

### 23.29. Bug: pending ops never get acked (some upstream error)

Client-side timeout (30 s). After timeout:
- Show "saving..." indicator escalates to "trouble saving — retrying."
- Buffer ops locally; do not corrupt local state.
- After 5 min, escalate to "we couldn't save — please refresh." (Critical: never lose user input.)

### 23.30. Browser quotas and IndexedDB eviction

Heavy users may exceed local quota. Eviction: oldest cached doc state. If the user goes offline before re-caching, they lose ability to edit *those* docs offline. Acceptable; show banner.

### 23.31. Data residency

Some customers (EU, government) require data to never leave their region. The session host placement, op log replication, and snapshot storage all respect a per-doc/per-org "region pin." Means we *can't* always place a session host in the user's region; latency is the cost.

### 23.32. Encryption at rest & in transit

- TLS 1.3 for all client-server traffic.
- mTLS for backend-to-backend.
- Encryption-at-rest in Spanner / Bigtable / GCS (default + customer-managed keys for premium).
- Edit content is *not* end-to-end encrypted (would break server-side OT, search, ACL enforcement). For customers requiring E2EE, separate "sensitive doc" mode runs purely client-side with CRDTs and encrypted blobs (no server OT) — a different product, not our default.

### 23.33. The "accidental fork" problem

Hypothetical: two session hosts both think they own a doc (split-brain). Both accept ops. Now we have two op chains.

Prevention:
- Routing table writes use Spanner-backed CAS. Only one host can hold the active lease.
- Lease renewal is synchronous with the controller.
- If a host loses contact with the controller: it stops accepting writes (read-only) until lease re-acquired.

If split-brain *did* happen (control plane bug):
- After detection: pick one chain as canonical, replay the other's ops as a "merge op" against the canonical chain.
- Engineer-reviewed; alert page; postmortem.

### 23.34. Eventual consistency of permission revocation across regions

ACL change in region A may take 100-500 ms to propagate to region B's session host caches. During that window, a revoked user might still type in region B.

Mitigations:
- ACL change pubsub to all regions.
- For *security-critical* docs, set a flag "synchronous ACL" — every op re-checks Zanzibar with a fresh consistency token. ~10ms latency cost per op.
- For most docs, the 500 ms eventual revocation is acceptable.

### 23.35. Op log compaction races

A nightly compaction job tries to drop ops older than the latest snapshot. But if a session host still has clients with old `baseRev`, those clients need those ops to catch up.

Solution: compaction watermark = `min(latest_snapshot.revision, oldest_client.baseRev)`. The session host publishes "oldest active baseRev" periodically. Compaction respects it.

### 23.36. Revision number overflow

Spanner column is INT64. At 10 ops/sec, that's 60 trillion years. Not a concern.

### 23.37. The Single 4 KB op that brings down the system

Defensive caps:
- Max op size: 64 KB (rejected if larger).
- Max ops/sec/user/doc: 200.
- Max ops/sec/host: 20K (overflow → 503).
- Max simultaneous editors/doc: 100 (101st gets read-only).

These are hard caps; soft monitoring at 80% of cap.

### 23.38. Doc owner deletes themselves; doc becomes orphan

Org policy: ownership transfers to admin; or doc moves to "no owner" state and is read-only until reclaimed.

### 23.39. Migrating from one doc format version to another

(E.g., we change the document model.) Approach:
- Server reads old format, writes new format on next snapshot.
- Old format deprecated after all snapshots upgraded.
- Background job grinds through old docs.
- Op log stays format-agnostic (ops are pure deltas; format is a snapshot concern).

### 23.40. Final answer to "what makes this hard"

Three things make Google Docs uniquely hard:
1. **Real-time + correctness**: OT/CRDT must be perfect; bugs cause silent divergence.
2. **Tail latency**: 99% of users see <250ms; the 1% includes poor networks, very large docs, hot docs — each requires special engineering.
3. **Long lifetime**: a doc lives for decades. Schema evolution, op format evolution, snapshot format evolution all must not break old docs.

The architecture above is built around those three constraints.

---

## Appendix A — Why Google Docs Picked OT (Real History)

Google Docs (originally Writely, acquired 2006) launched with a simple lock-based collaboration. In 2009, after Google Wave's research success, they migrated to OT. Wave shut down in 2010, but the OT engine moved into Docs.

Around 2014, Google considered CRDT migration internally. They did not pursue it because:
1. The migration would require running both engines in parallel during a multi-year cutover.
2. CRDT op size is 5-10x larger on the wire — for billions of ops/day, that's expensive.
3. The OT engine, despite its pain, was stable and well-tested.
4. The single-server topology (which OT requires) was *also* required for ACLs, abuse, search indexing — no free lunch from going decentralized.

CRDT advocates were correct that CRDTs are *easier to reason about*. They were wrong that the wire-cost was acceptable at this scale.

---

## Appendix B — Pseudocode Reference

### B.1. Server-side OT processing

```python
def handle_submit(req):
    doc = load_doc(req.doc_id)
    
    # 1. Validate
    if not user_has_role(req.author_id, doc, EDITOR):
        return reject("PERMISSION_DENIED")
    if op_too_large(req.op):
        return reject("OP_TOO_LARGE")
    if not validate_op_shape(req.op):
        return reject("MALFORMED_OP")
    
    # 2. Dedup
    if seen_seq(req.client_id, req.client_seq):
        return ack(cached_result(req.client_id, req.client_seq))
    
    # 3. Transform against intervening ops
    if req.base_revision < doc.head_revision:
        intervening = doc.ops[req.base_revision : doc.head_revision]
        if len(intervening) > MAX_REPLAY:
            return reject("STALE_BASE_REVISION")
        op = req.op
        for r in intervening:
            op = transform(op, r.op)  # rebase op against each
    else:
        op = req.op
    
    # 4. Persist
    new_rev = doc.head_revision + 1
    spanner.insert("doc_ops", row=(req.doc_id, new_rev, op,
                                    req.author_id, req.client_id, req.client_seq))
    doc.head_revision = new_rev
    apply_in_memory(doc, op)
    
    # 5. Broadcast
    broadcast(doc, {
        "type": "remoteOp" if subscriber != req.client_id else "ack",
        "revision": new_rev,
        "op": op,
        "author_id": req.author_id,
    })
    
    return ack(new_rev, op)
```

### B.2. Client-side OT loop

```python
class Client:
    rev = 0
    pending = None    # op submitted, awaiting ack
    buffer = None     # op composed of post-pending edits
    
    def user_typed(op):
        apply_local(op)
        if pending is None:
            pending = op
            send({"type": "submit", "op": op, "base_revision": rev,
                  "client_seq": next_seq()})
        else:
            buffer = compose(buffer or no_op, op)
    
    def on_remote_op(remote):
        # Transform incoming remote op past pending and buffer
        r = remote.op
        if pending:
            r, pending = transform(r, pending), transform(pending, r)
        if buffer:
            r, buffer = transform(r, buffer), transform(buffer, r)
        apply_local(r)
        rev = remote.revision
    
    def on_ack(ack):
        rev = ack.revision
        pending = None
        if buffer:
            pending = buffer
            buffer = None
            send({"type": "submit", "op": pending, "base_revision": rev,
                  "client_seq": next_seq()})
```

### B.3. Insert vs Insert transform

```python
def transform_insert_insert(a, b):
    # a, b: { type:"insert", pos, text }
    if a.pos < b.pos:
        return (a, Insert(b.pos + len(a.text), b.text))
    if a.pos > b.pos:
        return (Insert(a.pos + len(b.text), a.text), b)
    # same position — tiebreak
    if a.site_id < b.site_id:
        return (a, Insert(b.pos + len(a.text), b.text))
    else:
        return (Insert(a.pos + len(b.text), a.text), b)
```

### B.4. Insert vs Delete transform

```python
def transform_insert_delete(a, b):
    # a: insert, b: delete(pos, len)
    if a.pos <= b.pos:
        # insert before delete — delete shifts right
        return (a, Delete(b.pos + len(a.text), b.len))
    elif a.pos >= b.pos + b.len:
        # insert after delete — insert shifts left
        return (Insert(a.pos - b.len, a.text), b)
    else:
        # insert lands inside deleted range — collapse to b.pos
        return (Insert(b.pos, a.text), b)
```

### B.5. Cursor transformation

```python
def transform_cursor(pos, op):
    if op.type == "insert":
        if op.pos <= pos:
            return pos + len(op.text)
        return pos
    elif op.type == "delete":
        if op.pos + op.len <= pos:
            return pos - op.len
        elif op.pos >= pos:
            return pos
        else:
            return op.pos  # collapse to deletion start
```

### B.6. Snapshot loader (open document)

```python
def open_doc(doc_id, user_id):
    require_permission(user_id, doc_id, VIEWER)
    
    snapshot = bigtable.read_latest("snapshot", doc_id)
    base_rev = snapshot.revision
    
    # Replay ops since snapshot
    ops = spanner.read("doc_ops", where=(doc_id == doc_id, revision > base_rev),
                      order_by="revision ASC")
    state = snapshot.content
    for op in ops:
        state = apply(state, op)
    
    return {
        "content": state,
        "head_revision": ops[-1].revision if ops else base_rev,
        "snapshot_revision": base_rev,
    }
```

### B.7. Routing assignment

```python
def get_or_assign_session_host(doc_id):
    entry = cassandra.read("doc_routing", doc_id)
    if entry and entry.lease_until > now() + 5_seconds:
        return entry.host_addr
    
    # Pick a least-loaded host (consistent-hashed bias)
    candidate = pick_host(doc_id)
    new_lease = now() + 60_seconds
    
    # CAS write to claim
    success = cassandra.compare_and_swap("doc_routing", doc_id,
                                          old_value=entry,
                                          new_value=(candidate, new_lease))
    if success:
        return candidate
    else:
        # someone else won the race
        return cassandra.read("doc_routing", doc_id).host_addr
```

---

*End of design.*

> Design choices reflect what's been publicly written about Google Docs (Wave/Jupiter, Kix doc model, Zanzibar ACLs) plus general industry patterns. Internal Google implementations may differ — this is the staff-engineer-level mental model, not insider documentation.
