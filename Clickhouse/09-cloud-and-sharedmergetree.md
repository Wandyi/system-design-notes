# 9 · ClickHouse Cloud and SharedMergeTree

ClickHouse Cloud is where most new ClickHouse Inc. engineering happens. If you're interviewing for a cloud-side role, this file is mandatory. SharedMergeTree is the storage engine that defines the cloud's architecture.

## 9.1 The big architectural shift

| | Classic OSS (ReplicatedMergeTree) | Cloud (SharedMergeTree) |
|---|----------------------------------|--------------------------|
| Data location | Per-replica local disks | Object storage (S3 / GCS / Azure) |
| Compute | Stateful (tied to data) | **Stateless** |
| Replicas | Each owns a copy | Share data; just metadata |
| Replication | Async, replica-to-replica HTTP fetch | Implicit via shared storage |
| Coordination | Keeper | Keeper (more central role) |
| Scale-up time | Minutes (copy data) | Seconds (just metadata) |
| Replica count limit | ~10–20 practical | Hundreds |
| Cost per extra replica | ~1× data | ~0 storage; pay only compute |
| DR | Cross-region cluster or backup/restore | Object-storage replication or backup |
| Lightweight updates / deletes | Limited | First-class |

## 9.2 SharedMergeTree internals

### Data layout

Parts live on S3 as immutable objects, organized as:

```
s3://my-clickhouse-bucket/{tenant_id}/data/{db}/{table}/{part_uuid}/
    <col>.bin
    <col>.cmrk
    primary.idx
    columns.txt
    ...
```

Each part has a UUID. Replicas don't compete to write the same path; new parts are written under a new UUID, then metadata in Keeper is updated.

### Metadata in Keeper

Keeper stores:
- The list of currently-active part UUIDs for each table.
- The merge plan (which part UUIDs were merged into which new UUID).
- Mutations and their state.
- Insert blocks (for dedup).

A SELECT first reads Keeper to learn which parts are active, then fetches them from S3 (with aggressive local caching).

### Leaderless replication

Classic ReplicatedMergeTree has a soft notion of who plans merges. SharedMergeTree is genuinely leaderless:

- Any replica can ingest.
- Any replica can plan and execute a merge (with a Keeper-arbitrated lock so two replicas don't merge the same parts).
- All replicas see new parts via Keeper updates.
- "Scale up" = add a compute node, it joins, reads Keeper, starts serving. No data copy.

### The local cache (page cache + file cache)

Reading from S3 every query is too slow / too expensive. The compute nodes have an SSD-backed **file cache** (and standard OS page cache above that).

- File cache stores **decompressed** column slices (and/or compressed depending on settings).
- Sized as a fraction of the node's disk.
- LRU eviction.
- Configurable per disk: `<disks><cache><type>cache</type><disk>s3</disk><path>...</path><max_size>1Ti</max_size>...</disks>`.

Cold queries hit S3 (tens of ms per object); warm queries hit local cache (microseconds). The aspiration: 95%+ cache hit ratios on hot queries.

## 9.3 The cloud control plane

ClickHouse Cloud is a multi-tenant Kubernetes-based control plane:

- **Service** = a tenant's logical cluster — N compute pods + a shared S3 bucket + a Keeper cluster.
- **Auto-scale**: compute pods scale up/down based on CPU / memory / queue depth.
- **Idle pause**: a service that hasn't been queried for N minutes pauses compute (storage costs remain).
- **Verticality / scaling-up a single replica** — the service can upgrade individual replicas to larger sizes.
- **Per-tenant isolation** — separate namespaces, network policies, IAM-bound S3 prefixes.

## 9.4 Lightweight updates and deletes

Classic ClickHouse mutations rewrite whole data parts (expensive). SharedMergeTree + Cloud introduces lightweight DML:

### Lightweight DELETE (OSS too)

```sql
DELETE FROM events WHERE user_id = 42;
```

Marks rows with a hidden `_row_exists = 0`. 
Queries automatically filter them out. Background cleanup eventually rewrites parts to physically drop the rows.

### Lightweight UPDATE (Cloud-first)

```sql
UPDATE events SET event = 'click' WHERE user_id = 42;
```

Writes a "patch" part that overrides specific rows. Reads merge the patch in. Much cheaper than full part rewrite.

Both are designed for **occasional point edits** (GDPR deletes, fix-up of mis-ingested rows), not bulk transactions.

## 9.5 Storage tiers and policies

Beyond the basic S3, Cloud (and OSS) supports tiered storage:

```xml
<storage_configuration>
  <disks>
    <hot><type>local</type><path>/var/lib/clickhouse/data/</path></hot>
    <cold><type>s3</type><endpoint>https://...</endpoint></cold>
  </disks>
  <policies>
    <tiered>
      <volumes>
        <hot><disk>hot</disk><max_data_part_size_bytes>100000000000</max_data_part_size_bytes></hot>
        <cold><disk>cold</disk></cold>
      </volumes>
      <move_factor>0.1</move_factor>
    </tiered>
  </policies>
</storage_configuration>
```

Then in the table:
```sql
SETTINGS storage_policy = 'tiered'
PARTITION BY toYYYYMM(ts)
TTL ts + INTERVAL 30 DAY TO VOLUME 'cold';
```

Old data automatically moves to S3 (cheaper) while hot data stays local (faster). 
Cloud abstracts this — everything's effectively on S3 with a hot file cache.

## 9.6 Multi-tenancy patterns inside one Cloud service

Even within a single Cloud service, you might serve many "tenants" (your customers). Patterns:

- **`tenant_id` as leading column in ORDER BY** — physical co-location.
- **Per-tenant row-level security** — `CREATE ROW POLICY` ties a user to their tenant_id.
- **Per-tenant quotas** — `CREATE QUOTA` limits queries/sec, rows-read, etc.
- **Per-tenant materialized views** — sometimes cheaper than per-query aggregation.
- **Per-tenant projections** if hot-paths differ per tenant.
- **Per-tenant clusters** for the very largest tenants (escape hatch from shared-tenant).

## 9.7 Cost model for Cloud

You pay for:
1. **Compute** — per second per vCPU/RAM hour. Largest line item for write/query-heavy workloads.
2. **Storage** — per GB-month on S3.
3. **Egress** — per GB out of the region (cross-region replication, exports).
4. **Data transfer** within the region — typically free or cheap.
5. **Backups** — incremental snapshots; storage cost.

Optimization levers (interview-worthy):
- **Compress more aggressively** — pay CPU once, S3-cost forever.
- **Drop / TTL aggressively** — every retained row costs storage forever.
- **Use materialized views** so user queries don't re-scan raw events.
- **Right-size compute** — auto-pause for dev/test services.
- **Avoid `SELECT *` and FINAL** — both blow up read bytes from S3.

## 9.8 What's still hard in Cloud

- **Cold-query latency** — first read of cold data takes S3 hops. Mitigate with file cache and proactive cache warm-up for known dashboards.
- **Cross-region**: object storage is per-region; for multi-region you replicate either via S3 cross-region replication or backup/restore.
- **DDL across services** — you can't easily run a query that joins data in two separate Cloud services.
- **External access to internal data** — federated queries and exports must be designed.

## 9.9 The 5 cloud-specific terms to use precisely

1. **SharedMergeTree** — the engine.
2. **Stateless compute** — compute nodes have no durable data.
3. **File cache** — the local SSD layer on each compute node.
4. **Service** — a customer's logical cluster.
5. **Auto-pause / auto-scale** — the cost-saving behaviors.

Confusing these in an interview is a tell.

## 9.10 Worked design — picking SharedMergeTree vs ReplicatedMergeTree

**Prompt**: "We're running 5 ClickHouse nodes on bare metal with NVMe. Should we migrate to Cloud / SharedMergeTree?"

A staff answer weighs:
- **Read-latency sensitivity** — bare metal NVMe < cloud-file-cache hot, often by 2-3×.
- **Replication scale needs** — if you want hundreds of replicas for query QPS, Cloud wins.
- **Operational overhead** — bare metal needs you to run Keeper, replicas, monitoring, upgrades.
- **Cost** — at constant utilization, bare metal can be cheaper; at variable utilization (bursty), Cloud wins via auto-scale and pause.
- **DR** — bare metal needs explicit backups / cross-DC; Cloud gives you S3 cross-region.
- **Compliance / data residency** — both can be satisfied; Cloud has region selection.

Common answer: stay on bare metal for steady high-utilization production; use Cloud for dev/test, bursty workloads, and new customer-facing 
analytics products where you want to spin up services per-customer.

## 9.11 Must-internalize

- SharedMergeTree replaces ReplicatedMergeTree in Cloud. Stateless compute + S3 + Keeper for metadata.
- Replicas share data; scale-up is just adding a compute pod.
- File cache (local SSD) absorbs hot queries; cold queries hit S3.
- Lightweight DELETE (OSS) and lightweight UPDATE (Cloud) avoid full part rewrites.
- Multi-tenancy via row policies, quotas, ORDER BY leading-tenant_id, per-tenant projections.
- Cost levers: compress more, TTL aggressively, MVs to pre-aggregate, right-size compute.

---

## Sources

- [SharedMergeTree — official](https://clickhouse.com/docs/cloud/reference/shared-merge-tree)
- [SharedMergeTree + Lightweight Updates blog](https://clickhouse.com/blog/clickhouse-cloud-boosts-performance-with-sharedmergetree-and-lightweight-updates)
- [Stateless compute blog](https://clickhouse.com/blog/clickhouse-cloud-stateless-compute)
- [Jack Vanlightly — Serverless ClickHouse Cloud analysis](https://jack-vanlightly.com/analyses/2024/1/23/serverless-clickhouse-cloud-asds-chapter-5-part-2)
- [Altinity — MergeTree on S3 architecture](https://altinity.com/blog/clickhouse-mergetree-on-s3-intro-and-architecture)
- [ClickHouse Cloud overview](https://clickhouse.com/cloud)
