# 16 · Voldemort, Espresso, Ambry — LinkedIn's Online Stores

LinkedIn's online (member-facing) state has lived in three primary stores for a decade-plus:

- **Voldemort** — distributed KV store (Dynamo-style). The original home of derived & some online state.
- **Espresso** — distributed document store with MySQL-per-partition. Today's primary OLTP for member/profile/message state.
- **Ambry** — distributed immutable blob store (photos, attachments).

A staff candidate is expected to know all three: what each is good at, how they differ, and the migrations between them.

## 16.1 Voldemort

Open-sourced 2009. The intellectual heir of Amazon Dynamo (2007 paper).

### Data model

- Key-value. Keys are typed (often longs or strings); values are bytes (serialized JSON / Avro).
- Multiple stores in a cluster, each with its own schema.

### Distribution

- **Consistent hashing** over a virtual ring.
- Each store has a **replication factor** (typically 3).
- Each key is replicated to N consecutive nodes on the ring.
- **Sloppy quorum**: writes go to any N reachable replicas; reads similarly. Vector clocks for conflict.

### Read-only mode (the killer feature)

A unique Voldemort use case: **read-only stores built in Hadoop**.

- A nightly Hadoop job materializes the entire store as a set of sorted chunk files.
- Files pushed to Voldemort RO nodes via HDFS.
- Nodes atomically swap to the new version after distribution.
- Queries hit the in-memory hash index → mmap'd chunk file → result.

This is *exactly* the pattern modernized in Venice. Voldemort RO was LinkedIn's first answer to "serve derived data at low latency with batch refresh".

### Read-write mode

- Eventual consistency.
- Vector clocks for versioning.
- Read repair on inconsistent quorum.
- Operations: get, getAll, put, delete, getVersion.

### When it's used today

- Older online stores haven't been migrated yet.
- New work tends toward Espresso (for OLTP) or Venice (for derived data).
- Some specialized use cases where the simplicity is valued.

### What to know for the interview

- The Dynamo lineage (hash ring, vector clocks, sloppy quorum).
- The RO/RW split.
- Why Venice replaces Voldemort RO.
- Why Espresso replaces Voldemort RW for OLTP.

## 16.2 Espresso

LinkedIn's primary online document store. Built ~2012, open-sourced partially.

### Mental model

> A distributed sharded MySQL with online schema evolution, multi-region replication, and a richer query model than KV.

### Data model

- **Database** → **Table** → **Document**.
- Each document has a primary key + arbitrary fields (versioned schema).
- Documents are JSON-ish; Avro-defined schema.
- Multi-column primary keys supported.
- Secondary indexes (with caveats).
- Tables can be in different "consistency policies" (strong leader-write vs. eventual).

### Distribution

- A database is split into **partitions** (e.g., 1024 partitions per DB).
- Each partition has a leader + followers (3-way replication typical).
- Partitions assigned to physical Espresso nodes by **Helix**.
- Routing: client computes `hash(primary_key) % numPartitions`, looks up leader via D2/Helix.

### Underlying storage

- Each partition is a MySQL instance (or one MySQL instance per node hosting many partitions as separate DBs).
- Writes go to MySQL (with InnoDB), replicated via MySQL replication for in-DC redundancy, and via **Brooklin** CDC for cross-DC.

### Replication and consistency

- **In-DC**: synchronous leader → at-least-one-follower (or async, configurable).
- **Cross-DC**: asynchronous via Brooklin tailing MySQL binlogs into Kafka.
- Consistency: read-your-writes for the leader; eventual for follower / cross-DC reads.

### Schema evolution

- One of Espresso's strongest features.
- Schemas evolve forward-compatibly; old code reads new docs, new code reads old docs.
- Avro evolution rules apply.
- Schema registry tracks compatibility.

### Routing & failover

- D2-based routing.
- Leader fails → Helix elects a new leader from the followers.
- Recovery from MySQL binlog + Brooklin's WAL of in-flight writes.

### What it's used for

- **Member profile**.
- **Settings**.
- **Mailbox** (conversation + message metadata).
- **Job postings**.
- **Connections** (the canonical edge store; LIquid is a derived view).
- Anything that wants OLTP semantics + cross-region replication.

### Common interview points

- **Why MySQL underneath?** Mature, performant, well-understood. Schema enforcement helps. Same skill set for DBAs.
- **Sharding key choice** — generally by primary-key hash; for tables joining to a parent entity, by parent key for co-location.
- **Cross-partition queries** — not directly supported; do at the application or via a derived store (Venice).
- **Hot partition** — a few options: re-shard, denormalize, cache aggressively.

## 16.3 Ambry

Open-sourced 2016. LinkedIn's "S3-but-for-our-shape-of-data" before they had cheap S3.

### Mental model

> A distributed, immutable, append-only blob store. Each blob is a sealed unit; once written, never modified. Reads are very fast; deletes are tombstones.

### Data model

- **Blob ID** = a 32-byte structured identifier including the partition, account, container, and a unique ID.
- Each blob carries metadata (size, content-type, retention class).
- No filesystem hierarchy.

### Distribution

- Blobs are written to **partitions**.
- Each partition is replicated to multiple data nodes.
- A partition contains many blobs in append-only segment files.

### Storage layout

- Each data node has a per-partition log of blob payloads.
- An in-memory hash index (`blob_id → byte_offset_in_log`) is the only index.
- No global metadata index (avoids that being a bottleneck).
- Cleaning / compaction: GC old tombstoned blobs from segments.

### Write path

- Client requests a blob upload → router service routes to a partition.
- Multi-replica write: client posts the blob to multiple data nodes (or to a coordinator that fans out).
- Quorum ack.

### Read path

- Client requests `blob_id` → router computes partition → routes to data node.
- Data node looks up offset in hash index → reads bytes → returns.

### Why Ambry instead of S3?

When Ambry was designed:
- LinkedIn was in own DCs; no S3 nearby.
- LinkedIn-controlled hardware → cheaper at their scale.
- Network locality important for media-heavy fanout.

Today: many workloads have moved or are eligible to move to Azure Blob (the Microsoft equivalent). Ambry's role is shrinking in the cloud era.

### Failure modes

- Replica disk failure → re-replicate from peers.
- Partition leader failure → no concept of leader; any replica can serve reads; writes use coordinator.
- Tombstone leak (slow GC) → disk fills; auto-compactor and alerts.

## 16.4 Comparison table

| Property | Voldemort | Espresso | Ambry |
|---|---|---|---|
| Data model | KV | Document | Immutable blob |
| Schema | Bytes (app-managed) | Avro-defined | Just bytes + metadata |
| Consistency | Eventual | Strong (leader) | Eventual |
| Sharding | Consistent hash | Partition by key | Hash → partition |
| Underlying engine | BDB-JE, RocksDB | MySQL | Custom log-structured |
| Cross-DC | Built-in | Brooklin CDC | Built-in |
| Queries | get / put | Rich (range, secondary index) | get by blob_id |
| Sweet spot | Simple KV at scale | OLTP, profile-like data | Photos, attachments, videos |
| Current status | Maintenance | Active primary | Active but cloud-displaced |

## 16.5 Common interview questions

> **"When would you choose Espresso vs. Voldemort vs. Venice?"**
- Espresso: OLTP, transactional updates, secondary indexes, schema evolution.
- Voldemort RW: legacy or simple KV semantics that don't need Espresso's richness.
- Venice: derived/precomputed data; built offline or near-real-time; read-heavy serving.

> **"How would you scale Espresso for 10x writes?"**
Increase partition count (offline reshuffle — non-trivial). Lighten work in the MySQL layer (less indexing, smaller rows). Consider sharding by a different key for hotspots.

> **"How does cross-DC replication tolerate a 10-minute outage?"**
Brooklin queues / Kafka mirror buffers events; on recovery, drains the backlog. Eventual catch-up. Read-your-writes during outage limited to source DC.

> **"How would you migrate a table from Voldemort to Espresso?"**
Dual-write phase from app → both stores. Verify Espresso is correct via shadow reads. Migrate read traffic 1% → 10% → 50% → 100%. Decommission Voldemort once stable. Months-long process.

> **"How do you handle a Hot Partition in Espresso?"**
Several options: split the partition (offline reshuffle), denormalize so the hot key isn't all in one partition, add a per-key cache layer, or batch writes via an intermediate accumulator.

> **"How does Ambry handle blob deletion under GDPR?"**
Logical tombstone immediately + replica fanout. Physical removal happens at next compaction (within hours typically). If immediate physical deletion is required, force compaction on the partition.

> **"Why no global secondary index in Ambry?"**
Because of the trade-off: maintaining a global index would require coordination on every blob write, eroding the append-only simplicity. Application maintains its own metadata in Espresso or similar.

> **"How would you replace Ambry with Azure Blob?"**
Dual-write to Ambry + Azure Blob. Verify durability. Migrate reads 1% → 100%. Optimize: object-naming convention to encode the LinkedIn shard semantics; lifecycle policies for tiering.