# AWS S3 — Realistic Scenarios at Staff-Engineer Depth

> A practical, opinionated reference covering the features of S3 and how to combine them to solve real problems at scale. 
> Each scenario presents the naive approach, the S3-native approach, the trade-offs, what alternatives exist, and what I would actually do. 
> Written for engineers who already know "S3 stores objects" and now need to make architecture decisions where 100 PB, $1M/yr, and regulatory penalties are on the line.

> Companion to the other system-design docs in this folder. S3 is the most-deployed storage primitive at AWS scale; it underpins data lakes, backups, content delivery, ML training, log archival, immutable audit, and effectively every modern AWS architecture. Knowing it deeply pays compounding dividends.

---

## 0. The Staff-Level Frame

S3 is not "files in the cloud." At staff level, the questions stop being "how do I put a file there" and become:

1. **What's the access pattern?** (Sequential? Random? Read-heavy? Write-heavy? Burst? Cold?)
2. **What's the durability and availability budget?** (Standard 11×9? IA 11×9 but lower availability? One-zone — happy to lose data on AZ failure?)
3. **What's the cost shape?** (Storage? Requests? Egress? Retrieval? Lifecycle? Cross-region?)
4. **What's the consistency model and freshness budget?** (Strong read-after-write since 2020. Eventual for replication. Inventory has hours-of-lag.)
5. **What's the security and governance model?** (IAM? Bucket policies? Object Lock? Cross-account? Encryption keys?)
6. **What does failure look like?** (Region outage? Account compromise? Accidental deletion? Ransomware?)
7. **What's the operational lifecycle?** (Schema migrations on object naming? Lifecycle transitions? Restore drills?)

A staff engineer chooses S3 features the same way they'd choose a database engine: precisely matching feature to invariant. The wrong combination of versioning + replication + encryption + lifecycle has cost millions of dollars in extra storage, hidden bills, and audit failures across the industry.

---

## 1. S3 Mental Model — What S3 Is and Isn't

### 1.1 What S3 is

- **A flat object store** keyed by `(bucket, key)`. The "/" in keys is a convention — there are no real directories.
- **Strong read-after-write consistency** for new PUTs and overwrites since December 2020. (Pre-2020 it was eventually consistent for overwrites and DELETEs.)
- **Eleven 9s of durability** (Standard, Standard-IA, Intelligent-Tiering, Glacier classes). Replicated across ≥3 AZs in a region. One Zone-IA is a single AZ.
- **High availability** (99.99% Standard, 99.9% Standard-IA, 99.5% One Zone-IA).
- **Effectively unlimited scale** — petabytes per bucket; 100 TPS on a single prefix automatically scales (no longer needs prefix-randomization since 2018).
- **HTTP-based** with REST API, with consistent semantics across regions.

### 1.2 What S3 isn't

- **Not a filesystem**. No POSIX semantics (no atomic rename across keys; no directory locking). Mountpoint for S3 fakes some of this, with caveats.
- **Not a database**. No transactions across keys. No strong indexing. No conditional writes (until 2024's `If-None-Match`/`If-Match`; still limited).
- **Not low-latency**. Single-object GET ≈ 50–200 ms p50 from EC2 in same region; longer cross-region. S3 Express One Zone is ~10× faster but with caveats.
- **Not a queue**. People misuse it as one. Don't.
- **Not a cache**. S3 Transfer Acceleration helps for large global uploads; CloudFront is the actual cache layer.

### 1.3 The pricing model — internalize this

```
PAID DIMENSIONS:
  Storage      ($/GB/month)        — varies dramatically by class
  Requests     ($/1K requests)     — PUT/GET/LIST priced differently
  Egress       ($/GB)              — to internet, to other regions, to other VPCs
  Retrieval    ($/GB)              — IA classes, Glacier (separate from request fee)
  Replication  ($/GB transferred)
  Inventory / Analytics / Lens     — usually negligible at small scale, real at PB scale
  Lifecycle    (small per-object fee for transitions)

FREE DIMENSIONS:
  Same-region transfer to AWS service (mostly)
  In-VPC via Gateway Endpoint
  Intra-AZ transfer
```

A staff-level mistake: forgetting that **listing** PB of objects costs real money. A `LIST` over 1B objects = 1M LIST calls × $0.005/1K = $5,000. Inventory reports cost a fraction of that.

### 1.4 Storage classes — the cheat sheet

| Class | $/GB/mo* | Min duration | Min size | First-byte latency | Use when |
|---|---|---|---|---|---|
| **Standard** | $0.023 | none | none | ms | hot data, user-facing |
| **Intelligent-Tiering** | $0.023 → $0.0036 | 30d | none | ms (auto-promoted from cold) | unknown access pattern |
| **Standard-IA** | $0.0125 | 30d | 128 KB | ms | infrequent, predictable |
| **One Zone-IA** | $0.01 | 30d | 128 KB | ms | re-creatable infrequent (single-AZ risk) |
| **Glacier Instant Retrieval** | $0.004 | 90d | 128 KB | ms | rarely accessed but instant when needed |
| **Glacier Flexible Retrieval** | $0.0036 | 90d | 40 KB | minutes-to-hours | archive, occasional access |
| **Glacier Deep Archive** | $0.00099 | 180d | 40 KB | hours | true cold archive |
| **S3 Express One Zone** | ~$0.16 | none | none | sub-10ms | high-RPS hot workloads |

*us-east-1 prices, approximate; check current.

The 23×–230× cost differential between Standard and Deep Archive is the single biggest lever in any S3 design.

### 1.5 Limits that matter

- **Object size**: 5 GB single PUT, 5 TB max with multipart.
- **Bucket**: ~100 per account default (raisable). Don't shard data across buckets unless you have isolation reasons.
- **Key length**: 1024 bytes.
- **Object metadata**: 2 KB (user-defined headers).
- **Versions**: unlimited per key.
- **Bucket policy**: 20 KB.
- **Request rate**: 3,500 PUT/COPY/POST/DELETE per second per prefix; 5,500 GET/HEAD per second per prefix.
 Auto-scales beyond. No "prefix randomization" needed since 2018, but high prefixes still help in extreme cases.

---

## 2. Scenario 1 — Static Website / SPA Hosting

### 2.1 The problem

A React/Vue/Next.js (static export) frontend. Needs to serve `index.html`, JS bundles, CSS, images. Globally accessed. Tens of thousands of users per second at peak.

### 2.2 The naive S3 approach

Enable "Static Website Hosting" on a bucket; point DNS at it.

```
mybucket.s3-website-us-east-1.amazonaws.com
```

Issues:
- **No HTTPS** at the bucket-website endpoint (only HTTP).
- **No CDN** = slow for users far from us-east-1, and high egress cost.
- **No custom domain TLS** without CloudFront.
- **No edge caching** = every request hits S3 = $0.0004/1K GET + egress. At 100M req/day that's $40/day for requests alone, plus egress.

### 2.3 The real approach: CloudFront + S3 Origin

```
User → CloudFront (edge, 400+ PoPs)
       → cache hit (95%+):       served at edge in ms
       → cache miss:             OAI/OAC fetch from S3
S3 bucket: PRIVATE (block public access).
           Bucket policy allows only CloudFront's Origin Access Control (OAC).
```

- **HTTPS via ACM cert** (free for CloudFront).
- **Cache hit ratio 90–98%** for static assets → S3 origin sees ~5% of traffic.
- **Egress at CDN rate** (cheaper at edge in many regions; tier discounts at scale).
- **Custom domains, custom errors, custom routing** via CloudFront Functions / Lambda@Edge for SPA routing.

### 2.4 SPA routing trick

```
CloudFront error responses:
  403 → /index.html  (200)
  404 → /index.html  (200)
```

Because S3 returns 403 (or 404 if list permitted) for unknown paths, and SPA routing wants every path to load `index.html`. This is the standard pattern.

### 2.5 Cache invalidation

Bundles named with content hash: `app.a3f9c.js`, `vendor.b7e21.js`. New deploy → new filenames; old cache entries naturally expire. Only `index.html` gets a `Cache-Control: no-store` (or short TTL) — it's the entry point that changes between deploys.

```
index.html       Cache-Control: no-cache, must-revalidate
app.a3f9c.js     Cache-Control: public, max-age=31536000, immutable
```

This is the "cache-busting via filename" pattern. Don't `aws cloudfront create-invalidation` on every deploy unless absolutely needed; invalidations cost money and propagation takes minutes.

### 2.6 Trade-offs and alternatives

| Approach | Pros | Cons |
|---|---|---|
| **S3 + CloudFront + OAC** | Standard, cheap, scalable | Initial setup; SPA error mapping; deploy CI |
| **AWS Amplify Hosting** | Managed; auto-deploy from Git | More expensive; less control; vendor-locked workflow |
| **Vercel / Netlify** | Best DX; zero AWS config | Egress out of AWS region; vendor lock-in |
| **CloudFlare Pages + R2** | Cheap egress | Requires moving to CloudFlare ecosystem |
| **EC2 + Nginx serving from EBS** | Full control | Pet servers; you operate them; doesn't scale |

### 2.7 What I'd actually do

For most teams: **S3 + CloudFront + OAC + ACM + Route53**. The Terraform for this is ~50 lines and ships in an afternoon. At any meaningful scale this is what it ends up being even if you start with Amplify or similar.

---

## 3. Scenario 2 — User-Uploaded Media (Photos, Videos)

### 3.1 The problem

Mobile app users upload photos and videos. Tens of thousands per minute peak. Files 100 KB – 100 MB. Each must be processed (thumbnail generation, virus scan, transcoding) and served back via CDN.

### 3.2 The naive approach

```
Mobile → API server → API server uploads to S3
```

Issues:
- API server proxies bytes — bandwidth/latency through the EC2 fleet.
- API server's network and memory bottleneck.
- Egress from mobile → API → S3 doubles the network cost.
- API server is a stateful holder of partial uploads.

### 3.3 The pre-signed URL pattern

```
Mobile  → API server: "I want to upload, here's metadata"
         API server  → returns pre-signed PUT URL (valid 5 min)
Mobile  → S3 directly with the pre-signed URL
S3      → emits ObjectCreated event → SQS / Lambda / EventBridge
Lambda  → process (thumbnail, scan, etc.)
```

```python
# API server generates the URL:
url = s3.generate_presigned_url(
    'put_object',
    Params={'Bucket': 'uploads', 'Key': f'users/{user_id}/{uuid}.jpg',
            'ContentType': 'image/jpeg'},
    ExpiresIn=300,
)
return {'upload_url': url}
```

- API server never holds bytes.
- S3 absorbs the upload throughput natively.
- Asynchronous processing pipeline picks up via events.

### 3.4 Validation and safety

The pre-signed URL needs constraints:

```python
# Force content-type and size constraints
fields = {'Content-Type': 'image/jpeg'}
conditions = [
    {'Content-Type': 'image/jpeg'},
    ['content-length-range', 1, 10 * 1024 * 1024],  # 1B–10MB
]
post = s3.generate_presigned_post(
    Bucket='uploads', Key=key, Fields=fields, Conditions=conditions, ExpiresIn=300
)
```

S3 rejects uploads that violate constraints. Without this, attackers can upload arbitrary files.

### 3.5 Event-driven processing

```
S3 ObjectCreated event → SNS topic → fanout to:
  - SQS queue (image processor)
  - SQS queue (virus scan)
  - SQS queue (analytics ingestion)
  - Lambda (immediate notification)
```

Or via **EventBridge** for richer routing (filter by metadata, prefix, suffix). EventBridge is typically the better choice now — single bus, complex rules.

Why fanout via SNS/EventBridge, not direct S3 → Lambda?
- S3 → Lambda is 1:1; can only have one direct subscription per event type per bucket.
- SNS fanout supports multiple consumers without coupling.
- EventBridge supports content-based routing, archive, replay.

### 3.6 The processing pipeline

```
Original upload   → s3://uploads/users/{user_id}/orig/{uuid}.jpg
Thumbnail Lambda  → s3://uploads/users/{user_id}/thumb/{uuid}.jpg
Resize variants   → s3://uploads/users/{user_id}/{size}/{uuid}.jpg
Video transcode   → s3://uploads/users/{user_id}/hls/{uuid}/master.m3u8
                    + segments/* (for HLS streaming)
```

Some teams use **S3 Object Lambda** to generate variants on-the-fly (no pre-processing), but that's runtime cost vs storage cost trade — almost always pre-process when you know what variants you need.

### 3.7 Serving back via CloudFront

Public read URLs through CloudFront (signed cookies / signed URLs if private). The original raw upload bucket stays private (only Lambda + admins can read); 
the **distribution bucket** holds the processed variants and is what CloudFront serves.

Two-bucket pattern:
```
Bucket A: uploads (private, short retention, lifecycle to delete after 30d)
Bucket B: distribution (private, served via CloudFront with OAC, long retention)
```

This separates raw / processed / distribution concerns. Common at production scale.

### 3.8 Trade-offs and alternatives

| Approach | Trade |
|---|---|
| **API proxy upload** | Server sees all bytes; bandwidth bottleneck; simple |
| **Pre-signed URL (PUT)** | Direct upload; lighter API server; less validation |
| **Pre-signed POST (with conditions)** | Direct + validated; HTML form-friendly |
| **Multipart with pre-signed parts** | Required for >100 MB; complex client |
| **AWS Transfer Family** | If clients are SFTP/FTP-only; expensive |
| **Mountpoint for S3** | EC2 sees S3 as a local filesystem; not for browser uploads |

### 3.9 At MAANG scale

Instagram-like service: **100M uploads/day = ~1,150 uploads/sec average, ~10K/sec peak**. Pre-signed URL pattern is mandatory — proxying through any service tier doubles cost and creates a bottleneck. 
Lambda or Fargate or EKS jobs process; SQS handles bursts; EventBridge routes.

---

## 4. Scenario 3 — Large File Uploads (Multipart)

### 4.1 The problem

Backup / video / dataset / VM image uploads of 1 GB – 5 TB. Single PUT max is 5 GB; you need multipart for everything bigger, and multipart is *better* even for files >100 MB due to parallelism and retry granularity.

### 4.2 Multipart mechanics

```
1. CreateMultipartUpload    → returns UploadId
2. UploadPart × N            (5MB–5GB per part; up to 10,000 parts)
3. CompleteMultipartUpload   → S3 stitches parts into final object
   OR
3. AbortMultipartUpload      → discards parts
```

- Parts are stored as separate objects internally; you pay storage on parts until completed or aborted.
- Parts can be uploaded in parallel — drives the throughput.
- Parts can be retried independently — retry on failure is just one part, not the whole file.

### 4.3 The classic forgotten cost

```
Forgot to call **CompleteMultipartUpload or AbortMultipartUpload?**
The parts sit there. You pay for them. Forever.
A single failed multipart upload of 5 TB = ~$115/month forever.
```

**Lifecycle rule everyone needs**:
```yaml
LifecycleRules:
  - Status: Enabled
    AbortIncompleteMultipartUpload:
      DaysAfterInitiation: 7
```

This auto-aborts incomplete uploads after 7 days. Set this on every bucket. Audit with S3 Storage Lens.

### 4.4 Pre-signed multipart for client uploads

```
Server: CreateMultipartUpload, returns UploadId.
Server: For each part (10MB chunks), generate pre-signed UploadPart URL.
Client: PUT each part in parallel.
Client: POST CompleteMultipartUpload with ETag list.
```

This pattern is what S3 Console, AWS CLI, and most SDKs use under the hood. For browser uploads of large files (5–100 GB), this is the only sane approach.

### 4.5 Concurrency tuning

```
For 1 GB file with 10 MB parts:  100 parts.
At 8 parallel uploaders, ~12 parts each in series.
Throughput: ~8 × per-part-throughput (network-bound).

Aurora-class instances: ~100 MB/s per part; total ~800 MB/s.
1 GB → ~1.5 seconds.
```

Tune part size for your network: small parts = many requests = high overhead; big parts = fewer retries but slower failure recovery. Typical sweet spot: **8–64 MB**.

### 4.6 Transfer Acceleration

S3 Transfer Acceleration uses CloudFront edge to upload to a nearby PoP, then private network to S3 region. Useful for:
- Global users uploading to a single region.
- Cross-continent transfers (NYC user → EU bucket).

Cost: $0.04/GB extra. Worth it only when measured improvement justifies it. AWS provides a comparison tool: `s3-accelerate-speedtest`.

### 4.7 Mountpoint for S3 / S3 File Gateway

For "I want to use S3 from my application as if it were a filesystem":

- **Mountpoint for S3** (released 2023): high-throughput read/write, but **no atomic renames, no random writes mid-file, eventual visibility**. Best for ML training (read), batch jobs (read), append-only logs.
- **AWS Storage Gateway / S3 File Gateway**: NFS/SMB on-prem cache backed by S3. For lift-and-shift legacy.
- **goofys, s3fs-fuse**: community FUSE drivers. Slower; not for production.

Don't try to use S3 as a real filesystem unless you've fully internalized its semantics. Many bugs come from "I assumed renames are atomic."

### 4.8 What I'd actually do

For uploads >100 MB: always multipart with parallelism. For large dataset transfer in/out: AWS DataSync (managed, throttle-able, resumable, scheduled) or Snowball (physical for PB-scale).

For browser uploads: pre-signed multipart with chunk-resume on failure. Libraries like `@aws-sdk/lib-storage` handle this well.

---

## 5. Scenario 4 — Backup and Disaster Recovery

### 5.1 The problem

Backing up production databases, file shares, on-prem data, and other AWS service state (EBS snapshots, RDS dumps) to S3, with:
- 11×9 durability.
- 1-day RPO, 1-hour RTO.
- 7-year retention for compliance.
- Multi-region for region failure.
- Protection against accidental deletion and ransomware.

### 5.2 The lifecycle policy

```yaml
LifecycleRules:
  - Filter:
      Prefix: backups/db/
    Transitions:
      - Days: 30
        StorageClass: STANDARD_IA
      - Days: 90
        StorageClass: GLACIER_FLEXIBLE_RETRIEVAL
      - Days: 365
        StorageClass: DEEP_ARCHIVE
    Expiration:
      Days: 2555   # 7 years
```

A 100 TB backup retained for 7 years across these tiers:
```
Year 1 (Standard 30d, IA 60d, Glacier 275d):
  30 × $0.023 / 30 + 60 × $0.0125/30 + 275 × $0.0036 / 30 ≈ $0.058/GB
  = $5,800 for first year (rough)

Year 2-7 (Deep Archive):
  6 × 365 × $0.00099 / 30 ≈ $0.072/GB
  = $7,200 for remaining 6 years

Total: ~$13,000 over 7 years.

Vs all-Standard for 7 years:
  7 × 365 × $0.023 / 30 ≈ $1.96/GB → $196,000.
  ~15× more expensive.
```

The lifecycle policy is the single biggest cost lever in S3 backup design.

### 5.3 Object Lock / WORM for ransomware protection

```yaml
ObjectLockConfiguration:
  ObjectLockEnabled: Enabled
  Rule:
    DefaultRetention:
      Mode: COMPLIANCE       # cannot be removed by anyone (incl. root) until expiration
      Years: 7
```

- **GOVERNANCE mode**: removable by users with `s3:BypassGovernanceRetention`.
- **COMPLIANCE mode**: cannot be deleted or modified by anyone, including root, until retention expires.
- **Legal hold**: indefinite retention until explicitly released.

For ransomware: if a backup bucket has Object Lock COMPLIANCE for 30+ days, even a compromised root credential cannot delete the backups. **This is the gold standard for backup-of-backups** in regulated industries.

### 5.4 Versioning + MFA Delete

```yaml
VersioningConfiguration:
  Status: Enabled
  MFADelete: Enabled
```

Versioning keeps every overwrite as a separate version. Combined with MFA Delete (required to permanently delete versions, only configurable by root account), it's a strong defense against accidental or malicious deletion.

Caveats:
- Versioning **doubles or worse** your storage if you're constantly overwriting; combine with lifecycle to expire old versions.
- MFA Delete is somewhat operationally annoying (only the root account can configure or use). Many teams skip it; risk-tradeoff.

### 5.5 Cross-region replication (CRR)

```yaml
ReplicationConfiguration:
  Role: arn:aws:iam::123:role/replication-role
  Rules:
    - Status: Enabled
      Destination:
        Bucket: arn:aws:s3:::backup-eu-west-1
        StorageClass: GLACIER_FLEXIBLE_RETRIEVAL  # tier directly
        ReplicationTime:
          Status: Enabled
          Time:
            Minutes: 15      # SLA: 99.99% of objects replicated within 15 min
        Metrics:
          Status: Enabled
        EncryptionConfiguration:
          ReplicaKmsKeyID: arn:...
```

- **Asynchronous**: typical lag minutes; SLA-bound 15-min option (S3 Replication Time Control, RTC) costs extra.
- **Same-region replication (SRR)**: useful for compliance (must keep two copies, even in same region).
- **Replicates only new objects** by default; for existing objects, use S3 Batch Replication (separate operation, paid per object).
- **Doesn't replicate deletes by default** (good — protects against accidental delete propagation). Can opt in with delete marker replication.

### 5.6 The 3-2-1-1-0 backup rule (cloud era)

```
3 copies of data
2 different storage media (still relevant — S3 + Glacier counts)
1 offsite copy (different region)
1 immutable copy (Object Lock)
0 errors after recovery test (the rule everyone forgets)
```

Last point: **untested backups are not backups**. Quarterly restore drills.

### 5.7 Trade-offs and alternatives

| Approach | Trade |
|---|---|
| **S3 Lifecycle to Glacier** | Cheapest long-term; restore latency hours |
| **S3 + CRR** | Multi-region; doubles storage cost |
| **Object Lock** | Ransomware protection; cannot delete even legitimately |
| **AWS Backup** | Managed cross-service backup orchestration (RDS, EBS, EFS, etc.) |
| **3rd-party (Veeam, Rubrik, etc.)** | Compliance & UI features; expensive |
| **Self-managed scripts** | Cheapest; fragile; no audit |

### 5.8 What I'd actually do

For production backup at MAANG scale:
- AWS Backup orchestration (handles RDS snapshots, EBS, EFS).
- Daily snapshots → S3 Standard for first 30 days.
- Lifecycle to Standard-IA (60 days), then Glacier Flexible (1 year), then Deep Archive (7 years).
- CRR to second region (also lifecycle-tiered).
- _Object Lock COMPLIANCE on the secondary region's backup bucket_ — that's your "cannot be deleted" copy.
- _Monthly automated restore drill:_ pick a random backup, restore, verify integrity, notify if fail.
- Alarms on backup-bucket size deviating from baseline (catches both missing backups and accidental over-write).

---

## 6. Scenario 5 — Data Lake / Analytics Storage

### 6.1 The problem

Hundreds of TB to PB of data: clickstream events, application logs, transactional CDC, batch ETL outputs. Analyzed by Athena, Spark/EMR, Redshift Spectrum, ML pipelines.

### 6.2 The partitioning strategy

```
s3://data-lake/events/year=2026/month=04/day=28/hour=15/part-00000.parquet
s3://data-lake/orders/cdc/year=2026/month=04/day=28/part-00001.parquet
```

Hive-style partitions. Athena, Spark, and Redshift Spectrum prune by partition automatically with `WHERE year = 2026 AND month = 4`. Without partitioning, every query scans every object — fast path to bankruptcy at PB scale.

**Right-sized partitions**: _aim for ~128 MB – 1 GB per file, ~1-100 files per partition. Too small (1MB files_) = high LIST/GET overhead; too big (10 GB) = cannot parallelize. 
Compaction jobs maintain this.

### 6.3 File format — Parquet with care

```
Parquet (columnar, compressed):
  - 5–10× smaller than JSON or CSV for the same data.
  - Athena/Spark can prune columns and row groups.
  - Standard at AWS scale.

ORC: alternate columnar; comparable performance.
Avro: row-oriented; good for Kafka pipelines, less for analytics.
JSON/CSV: avoid except for ingestion staging.
```

Always Parquet for warehouse data. Compression: Snappy (fast) or Zstd (smaller). Column statistics (min/max/null count) enable predicate pushdown.

    Predicate pushdown is a query optimization technique that improves performance by moving filter conditions ("WHERE" clauses) as close to the data source as possible. 
    It reduces data processing by filtering rows at the storage level, minimizing data transfer to the query engine and lowering memory usage.

### 6.4 Table formats — Iceberg, Hudi, Delta

Plain "Hive on S3" has problems:
- No atomic commits (a half-written partition is visible).
- No schema evolution that survives readers.
- No time-travel queries.
- No efficient point updates / deletes.
- Listing partitions is slow as the table grows.

Modern table formats fix this with metadata layers:

| Format | Strength |
|---|---|
| **Apache Iceberg** | Best-of-breed; AWS native support (Athena, EMR, Glue); large ecosystem |
| **Apache Hudi** | Best for streaming upserts; CDC-friendly |
| **Delta Lake** | Strongest if you're Databricks-native; Spark-first |

Iceberg has emerged as the de-facto standard at AWS for new data lakes. Athena, EMR Spark, Snowflake, Trino all support it natively.

### 6.5 Glue Data Catalog

```
Bucket holds files. Glue Data Catalog holds the schema, partitions, table format.
Athena, Redshift Spectrum, EMR query via Glue Catalog.
```

- Glue is a Hive Metastore-compatible catalog.
- AWS Lake Formation layers ABAC permissions on top (column-level, row-level filtering).
- Glue Crawlers auto-detect schema (use sparingly — slow, costly, can wreck schemas).

### 6.6 Cost levers in a data lake

1. **Compress aggressively** (Zstd over Snappy when CPU permits).
2. **Partition correctly** (avoid over-partitioning; <10K partitions per table).
3. **Compact small files** weekly (5,000 small files → 50 large files, 100× scan-time improvement).
4. **Lifecycle to IA / Glacier** for partitions older than X. Athena can query Glacier Instant Retrieval. Glacier Flexible/Deep cannot (must restore first).
5. **Set up S3 Intelligent-Tiering** for unpredictable access (typical for analytics workloads).
6. **VPC Endpoints** for compute → S3 to avoid NAT Gateway costs.
7. **Athena query result location** in cheap class with short TTL.

### 6.7 At PB scale

- **S3 Storage Lens** to monitor at account level.
- **S3 Inventory** (daily/weekly CSV/Parquet of all objects with metadata) — cheaper than `LIST` for analyzing storage.
- **S3 Batch Operations** for re-tagging, copying, lifecycle changes across millions of objects.

### 6.8 Alternatives

| Approach | Trade |
|---|---|
| **S3 + Iceberg + Athena** | Cheap, decoupled compute, vendor-agnostic |
| **S3 + EMR Spark** | Heavier compute, more flexibility |
| **Snowflake on S3** | Best DX, performance; very expensive at PB |
| **Redshift (managed cluster)** | Best for BI workloads; cluster overhead |
| **Databricks on S3** | Best Spark experience; vendor lock-in |

### 6.9 What I'd actually do

Greenfield at MAANG scale: **S3 + Iceberg + Glue Catalog + Athena/Spark + Lake Formation** is the pragmatic default. Workflow:
- Ingest via Kinesis Firehose / MSK to S3.
- Compact + partition with EMR/Glue jobs into Iceberg tables.
- Query via Athena (ad-hoc) and Spark (heavy).
- ML feature stores (SageMaker Feature Store) on top.
- Lifecycle for older partitions.

---

## 7. Scenario 6 — Log Aggregation Pipeline

### 7.1 The problem

Tens of thousands of pods generating GB/sec of logs. Need to collect, store, query (within seconds for hot queries; minutes for older), retain 90 days, archive 7 years.

### 7.2 The architecture

```
Pod stdout/stderr → Fluent Bit / Vector → Kinesis Firehose / Kafka
  → Firehose buffers (1–15 min) → writes Parquet to S3:
       s3://logs/service=foo/year=2026/month=04/day=28/hour=15/part-...

Hot path:        OpenSearch (last 7 days)
Warm path:       Athena on S3 Parquet (7–90 days)
Cold path:       Glacier Deep Archive (90 days+)
```

Firehose batches → writes Parquet at intervals; partitions by date/hour. OpenSearch (managed Elasticsearch) handles low-latency search on recent data.

### 7.3 Why not put everything in OpenSearch

- 1 PB OpenSearch cluster: $$$. Storage on EBS gp3 ~10× cost of S3.
- Indexing cost: significant CPU per log line.
- Operational cost: cluster ops, version upgrades, scaling.

S3 + Athena is **10–50× cheaper** for warm/cold queries; OpenSearch reserved for the hot 7 days where ms-latency search matters.

### 7.4 Firehose → S3 best practices

- **Buffer**: 5–15 minutes or 5–128 MB, whichever first. Smaller = lower latency, more files; bigger = better Parquet, slower visibility.
- **Compression**: built-in Snappy / Zstd / Gzip.
- **Format conversion**: Firehose can convert JSON → Parquet inline (with Glue Catalog as schema).
- **Partition**: dynamic partitioning (Firehose 2021+) extracts fields and routes to prefix.
- **Error output**: separate prefix for failed records; investigate before retention drops.

### 7.5 Querying with Athena

```sql
SELECT count(*), service
FROM logs
WHERE year = 2026 AND month = 4 AND day = 28
  AND level = 'ERROR'
GROUP BY service;
```

Athena scans only the day's partition; with Parquet column pruning, only the `service` and `level` columns. Cost: ~$5/TB scanned. A targeted query of one day across 100 services scans ~10 GB → costs $0.05.

Without partitioning, the same query would scan the full table (100s of TB) → $500+ per query. **Partitioning is the difference between a $500 query and a $0.05 query.**

### 7.6 Cost optimizations

- **Sample at high volume**: top 1% of paths might generate 80% of logs. Sample non-error logs at 10%; keep 100% of errors.
- **Drop redundant fields** at the agent before forwarding.
- **Compact small Parquet files**: Firehose's 5-minute buffer creates many small files; nightly job merges into hour-aggregated.
- **Tier partitions to Glacier IR after 30 days**: still queryable by Athena.

### 7.7 Alternatives

| Approach | Trade |
|---|---|
| **CloudWatch Logs only** | Easy; expensive at scale (~$0.50/GB ingested); slow query |
| **Firehose → S3 + Athena** | Cheapest; partition-aware queries |
| **Firehose → OpenSearch** | Best search UX; operational cost |
| **Hybrid (hot OpenSearch + cold S3)** | Best of both; complexity |
| **Datadog / New Relic / Splunk** | Best UX; eye-watering cost at PB |

### 7.8 What I'd actually do    

At 1 PB/year of logs:
- Hot 7 days in OpenSearch (sized for ms search).
- Warm 30 days in S3 Standard, queryable via Athena.
- Tiered to Glacier IR for 90 days, then Glacier Flexible to 1 year, Deep Archive to 7 years.
- Sampled for ingestion cost (errors 100%; info logs 10%; debug 1%).
- Glue Catalog with Iceberg for schema evolution as log fields change.

---

## 8. Scenario 7 — ML Training Data and Checkpoints

### 8.1 The problem

Training deep models on large datasets:
- 100 TB image dataset.
- 10K GPUs reading concurrently.
- Model checkpoints (10–100 GB) saved every 30 min.
- Multi-region training-data availability.

### 8.2 The read pattern

```
Training jobs need:
  - Sequential GB/s read throughput per worker
  - 1000s of workers reading concurrently
  - p99 stable; tail latency spike causes slow GPU-utilization
```

S3 Standard delivers ~5,500 GET/sec per prefix, auto-scales beyond. For 10K workers reading at 100 MB/s each = 1 TB/s aggregate. S3 Standard handles it if you spread across prefixes.

### 8.3 Mountpoint for S3 vs FSx for Lustre

```
Mountpoint for S3:
  + Direct S3 access; no extra cost.
  + Stream large files efficiently.
  - No POSIX semantics; no random write.
  - High overhead for many small reads.

FSx for Lustre (with S3 backing):
  + POSIX filesystem; great for random + small reads.
  + 1 TB/s+ aggregate; sub-ms latency.
  + Lazy-loads from S3 backing on first read.
  - More expensive ($$/GB-hour while mounted).
  - Operational complexity.

S3 Express One Zone (single AZ):
  + Sub-10ms latency, 100K+ TPS.
  + 50% cheaper than EBS gp3 for hot data at scale.
  - Single AZ: lose AZ, lose data.
  - More expensive than Standard for storage.
```

### 8.4 The training-data lifecycle

```
Ingest                → S3 Standard
Active training       → Standard or Express One Zone
Inactive datasets     → Standard-IA (still ms-accessible)
Archive               → Glacier (must restore for re-training)
```

For active datasets accessed in a training run: Standard. After training: lifecycle to IA. Archive when no longer in active use.

### 8.5 Checkpoints

```
Save every 30 min during training.
Each checkpoint: 50 GB.
24-hour training: 48 checkpoints × 50 GB = 2.4 TB per run.
1000 runs/month: 2.4 PB.
```

Strategies:
- Keep last N checkpoints (e.g., 3) per run; delete intermediates.
- Compressed checkpoints (~30% reduction).
- Multipart upload for 50 GB checkpoints (parallel parts → seconds to upload).
- Standard-IA for older checkpoints if you need to roll back.

### 8.6 Multi-region training

```
Primary region (us-east-1): training cluster.
Replicate dataset CRR → eu-west-1.
EU training clusters can read locally.
```

Egress savings: 10K workers each reading 100 GB = 1 PB. Cross-region egress at $0.02/GB = $20K per training run. Replicating once and reading locally: 1 PB × $0.02 once + free reads.

### 8.7 Trade-offs and alternatives

| Approach | Trade |
|---|---|
| **S3 directly via SDK** | No mounting overhead; SDK in code |
| **Mountpoint for S3** | Filesystem feel; large sequential reads |
| **FSx for Lustre + S3** | Best perf; expensive |
| **EFS** | POSIX; slower than Lustre; managed |
| **EBS / instance store** | Fastest; doesn't scale; operational |

### 8.8 What I'd actually do

Modern recipe:
- Datasets in S3 Standard, Iceberg-cataloged.
- Training: Mountpoint for S3 if sequential I/O dominates; FSx for Lustre for random I/O models (recommendation systems, GNNs).
- Checkpoints: S3 Standard with multipart upload.
- Inactive: S3 Standard-IA.
- Multi-region replication for global training fleet.

---

## 9. Scenario 8 — Compliance and Immutable Storage (WORM)

### 9.1 The problem

Regulated data: SEC 17a-4 (financial records, 7-year), HIPAA (medical, 6-year), GDPR (right-to-be-forgotten with retention exceptions), FINRA, state regulations.

Requirements:
- Write Once Read Many (WORM).
- Tamper-evident (auditable that records weren't altered).
- Retention enforcement (cannot delete before X).
- Audit trail of access.

### 9.2 Object Lock

```yaml
ObjectLockConfiguration:
  ObjectLockEnabled: Enabled
  Rule:
    DefaultRetention:
      Mode: COMPLIANCE
      Years: 7
```

**COMPLIANCE mode**:
- Cannot be deleted until retention expires.
- Cannot be modified.
- Cannot be removed even by root account.
- Cannot be reduced (only extended).

**GOVERNANCE mode**: same, but can be overridden by `s3:BypassGovernanceRetention` permission.

For SEC 17a-4: COMPLIANCE. AWS has SEC certification of S3 Object Lock COMPLIANCE.

### 9.3 Versioning required

Object Lock requires versioning enabled. Each version can have its own retention.

### 9.4 Legal Hold

Indefinite retention regardless of expiration. Used for:
- Litigation discovery.
- Investigation.
- Manual override for important data.

Removed only by users with `s3:PutObjectLegalHold`. Operationally controlled.

### 9.5 The immutable audit log pattern

```
Application writes audit records → S3 bucket with Object Lock COMPLIANCE 7 years.
Each record signed/hashed; chained.
Auditor can verify chain integrity without trusting the application.
```

This is the standard pattern for SOX, HIPAA, PCI audit trails. Cheaper than dedicated WORM storage appliances.

### 9.6 GDPR right-to-be-forgotten conflict

GDPR Article 17 right-to-be-forgotten: subject can request deletion. SEC 17a-4: must retain 7 years. Conflict.

Resolution:
- **Pseudonymize** data: store user data keyed by pseudonym; keep mapping in a separate "deletable" store.
- **Right-to-be-forgotten** = delete the mapping; data still retained but unlinkable.
- AWS calls this pattern "crypto-shredding": encrypt with per-user KMS keys; "delete" = destroy the user's KMS key.

This is the only way to reconcile contradictory regulations.

### 9.7 S3 Glacier Vault Lock (separate from Object Lock)

For Glacier vaults (note: distinct from S3 Glacier storage classes), Vault Lock provides immutable archive policies. Older API; mostly subsumed by Object Lock on S3.

### 9.8 Trade-offs and alternatives

| Approach | Trade |
|---|---|
| **S3 Object Lock COMPLIANCE** | True WORM; SEC-certified; cannot be undone |
| **S3 Object Lock GOVERNANCE** | WORM with override path |
| **S3 Versioning + MFA Delete** | Cannot accidentally delete; can intentionally |
| **AWS Backup Vault Lock** | For backup data specifically |
| **3rd-party WORM appliance** | Legacy; don't choose new |

### 9.9 What I'd actually do

For new compliance workloads: S3 Object Lock COMPLIANCE on a dedicated bucket; cross-region replication to a second-region bucket also in COMPLIANCE mode; 
separate AWS account for the compliance bucket so production permissions can't reach it; dedicated KMS key with restricted access.

---

## 10. Scenario 9 — Cross-Account Data Sharing

### 10.1 The problem

Producer team in Account A generates data. Consumer team in Account B (or 50 different accounts) needs to read it. Cannot use IAM users — they're cross-account.

### 10.2 The bucket policy approach

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::222222222222:root" },
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::shared-data",
        "arn:aws:s3:::shared-data/*"
      ]
    }
  ]
}
```

Account A's bucket policy grants access to Account B principals. Account B's IAM still must allow s3 actions on its side ("two-key" model for cross-account: *both* must allow).

### 10.3 The KMS encryption gotcha

If the bucket is encrypted with a KMS key in Account A, Account B users also need permission on **the KMS key**. Otherwise: read fails with cryptic "access denied."

```json
// In Account A's KMS key policy:
{
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::222222222222:root" },
  "Action": ["kms:Decrypt"],
  "Resource": "*"
}
```

This is the most common cross-account S3 bug at staff-level review.

### 10.4 The bucket owner enforced setting (2021+)

If consumers in Account B PUT objects to Account A's bucket, by default the **uploader** (B) owns the object, not the bucket owner (A). Account A may not be able to read it.

Fix: enable **Bucket owner enforced** ACL setting. Object ACLs are disabled; bucket policy is the sole access mechanism. AWS recommends this for all new buckets.

### 10.5 Access Points — for many consumers

```
Bucket: data-lake (Account A, complex bucket policy)
Access Point: api-users-ap (granted to Account B with limited prefix and IP filter)
Access Point: ml-users-ap  (granted to Account C with different prefix)
```

Access Points are **named access policies** with their own ARNs. Each has:
- Its own policy.
- Optional VPC restriction.
- Optional network origin (internet vs VPC).

Cleaner than one giant bucket policy. Critical when sharing one bucket with 100s of consumers (hits 20 KB bucket policy limit).

### 10.6 S3 Access Grants (newer, 2023)

Manage permissions per-prefix with IAM Identity Center integration. Useful for human-driven access (data scientists with SSO). Less useful for service-to-service.

### 10.7 Lake Formation for column/row-level

For data-lake style sharing where Account B should see only some columns or rows:
- Glue Data Catalog + Lake Formation policies.
- Lake Formation enforces column/row filtering at query time.
- Account A's data; Account B queries via Athena/Spark with filtered results.

This is the modern way to share analytics data without copying.

### 10.8 Requester Pays

```
Bucket configured as Requester Pays.
GET requests must include `x-amz-request-payer: requester`.
The requesting account pays for the request and egress.
```

Useful when you publish data others want, but don't want to pay their egress (open datasets, public datasets).

### 10.9 Trade-offs and alternatives

| Approach | Trade |
|---|---|
| **Bucket policy** | Simple; limited (20KB) for many principals |
| **Access Points** | Scales to many consumers; per-AP policy |
| **Cross-account replication** | Decouples producer/consumer accounts; doubles storage |
| **Lake Formation** | Fine-grained; analytics-focused |
| **S3 Access Grants** | SSO-integrated; new, less mature |
| **Pre-signed URLs** | Time-bounded; per-request |

### 10.10 What I'd actually do

For B2B/cross-team:
- Bucket owner enforced.
- KMS key shared to consumer accounts via key policy.
- Access Points per consumer team.
- VPC restriction on Access Point if cross-account flow is via PrivateLink.
- Audit via CloudTrail data events on the bucket.

---

## 11. Scenario 10 — Software Artifact Distribution

### 11.1 The problem

Distribute installers, container images, Helm charts, ML models, ZIP releases to:
- Internal developers (high-trust)
- External customers (medium-trust, signed URLs)
- Partner integrations (cross-account)

### 11.2 The CDN-fronted pattern

```
Build pipeline → S3 (versioned: artifacts/v1.2.3/binary.tar.gz)
              → CloudFront distribution
                → Customers download via CDN
```

- CloudFront edge caching.
- Signed URLs / signed cookies for time-bounded access.
- Geo-restriction if needed.
- Custom domain (`releases.mycompany.com`).

### 11.3 Versioning vs immutable keys

Two patterns:

**Immutable keys** (preferred):
```
artifacts/v1.2.3/binary.tar.gz  (never changes)
artifacts/v1.2.4/binary.tar.gz  (new version)
```

- Caching is trivial: `Cache-Control: immutable, max-age=31536000`.
- No invalidations needed.
- "Latest" is a separate small file pointing to the current version.

```aiexclude

        The HTTP response header Cache-Control: immutable, max-age=31536000 is a modern best practice for caching static assets that will never change, such as versioned JavaScript, CSS, or images. 
 
Breakdown of Directives
max-age=31536000: This tells the browser (and other caches) to store the file for exactly 31,536,000 seconds, which is one year. This is the standard "forever" value recommended by RFC 2616 for assets that should not expire.
immutable: This is a powerful optimization that tells the browser the content at this specific URL will never change. Under normal circumstances, if a user clicks "Refresh" (F5), browsers typically revalidate cached content with the server. Adding immutable prevents this revalidation, saving the server from processing thousands of "304 Not Modified" requests and reducing latency for the user. 

When to Use This Header
You should only use this header for "cache-busted" or versioned resources where a content change results in a completely different filename. For example: 
/assets/styles.f4fa2b.css
/js/app.v1.2.js
/images/logo-2024.png 
Comparison with Other Strategies
Directive 	When to Use	Browser Behaviour
no-store	Sensitive or highly dynamic data	Never caches; always downloads fresh.
no-cache	Content that changes often	Caches but must revalidate with the server every time.
max-age (only)	Standard static files	Caches for the duration but revalidates on user refresh.
immutable	Versioned/Hashed assets	Caches for the duration and ignores refresh revalidation.
```
**Mutable + versioning**:
```
artifacts/binary.tar.gz  (overwrite each release)
S3 Versioning enabled to keep history.
```

Worse because:
- Cache invalidation needed on each release.
- "Latest" is implicit; tooling has to know.

Always prefer immutable keys for distributable artifacts.

### 11.4 Container images — ECR vs S3

S3 can serve container layer blobs (skopeo, oras), but **ECR (Elastic Container Registry)** is purpose-built. ECR uses S3 underneath; you don't manage S3 directly.

For non-container artifacts (binaries, ZIPs): S3 + CloudFront is right.

### 11.5 Signed URLs for paid / restricted downloads

```python
# Generate URL valid for 1 hour
url = s3.generate_presigned_url(
    'get_object',
    Params={'Bucket': 'releases', 'Key': 'enterprise/v1.2.3.tar.gz'},
    ExpiresIn=3600,
)
```

Use for:
- Customer-specific downloads.
- One-time tokens after license verification.
- Audit-trail of who downloaded what.

For CloudFront-fronted: use **CloudFront signed URLs / signed cookies** instead — the signature is verified at edge; S3 stays private (OAC).

### 11.6 OpenAPI / SDK distribution

Same pattern: build → S3 (versioned) → CloudFront. Use Pages-like custom domain for documentation site.

### 11.7 What I'd actually do

For a SaaS product distributing CLI binaries / SDKs / Docker images:
- Containers → ECR (with cross-region replication).
- Binaries / ZIPs → S3 with immutable keys, fronted by CloudFront, signed URLs for paid/restricted.
- Documentation → S3 + CloudFront (the static-site pattern from §2).
- Build pipeline writes to S3 with `Cache-Control: immutable`.

---

## 12. Scenario 11 — Event-Driven Processing

### 12.1 The problem

When an object lands in S3, trigger a workflow: thumbnail generation, virus scan, ETL, ML inference, downstream replication. At 100K events/sec.

### 12.2 The event sources

S3 emits events for:
- `s3:ObjectCreated:Put` / `Post` / `CompleteMultipartUpload`
- `s3:ObjectRemoved:Delete`
- `s3:ObjectRestore:Completed`
- `s3:Replication:OperationFailedReplication`

Destinations:
- **SNS topic** (fanout to multiple subscribers)
- **SQS queue** (one consumer)
- **Lambda function** (direct, but limits)
- **EventBridge bus** (rich routing)

### 12.3 Direct → Lambda

```
S3 → Lambda (synchronous trigger)

Pros: simple, low-latency.
Cons:
  - 1 trigger per event-type; can't fanout natively.
  - Lambda concurrency limit (1000 default; raise for high event rate).
  - Failed Lambdas → DLQ if configured; otherwise lost.
```

For >10K events/sec, Lambda concurrency tuning becomes a job.

### 12.4 S3 → SQS → Lambda (or EKS / Fargate)

```
S3 → SQS (queue absorbs burst) → Lambda batch-pulls → process

Pros: resilient to bursts; bounded concurrency; replays on failure.
Cons: extra latency (poll-based); **SQS retention bounded (14 days max).**
```

This is the **standard pattern for production-grade async processing**. SQS smooths bursts; Lambda scales pull rate.

### 12.5 S3 → EventBridge → multiple targets

```
S3 → EventBridge bus → rules:
  rule 1: prefix=images/ → Lambda thumbnail
  rule 2: suffix=.csv    → Step Functions ETL
  rule 3: contains=PII   → SQS for human review

Pros: single source; rich routing; archive + replay; cross-account easy.
Cons: extra hop; EventBridge cost ($1/M events).
```

EventBridge is the AWS-recommended event bus now; SNS-based fanout is "legacy but still works."

### 12.6 Event filtering on S3 side

```yaml
NotificationConfiguration:
  TopicConfigurations:
    - Topic: arn:aws:sns:...:images-topic
      Events:
        - s3:ObjectCreated:*
      Filter:
        S3Key:
          FilterRules:
            - Name: prefix
              Value: uploads/
            - Name: suffix
              Value: .jpg
```

Filter at the S3 source so you don't pay for events you'd discard. Saves Lambda invocations and downstream cost.

### 12.7 Ordering and exactly-once

S3 events are **at-least-once** and **not strictly ordered**. Multiple events for the same object can arrive in any order — including a "delete" before a corresponding 
"create" (you wouldn't think this is possible, but in retry scenarios it is).

Make consumers idempotent:
- Use object's ETag or version ID for deduplication.
- Don't assume "first event = creation, second = update."
- Re-fetch from S3 to verify state.

### 12.8 Failure handling

```
Lambda fails 3 times → DLQ (SQS DLQ).
Process DLQ manually or via auto-replay job.
Alarm on DLQ depth.
```

Without DLQ, failed events disappear. Always configure.

### 12.9 The cost shape at high volume

```
1M events/day:
  S3 PUT requests:     $5/M
  SNS:                 $0.50/M
  SQS:                 $0.40/M (or $0.20 for FIFO)
  EventBridge:         $1/M
  Lambda invocations:  $0.20/M + duration

Direct S3 → Lambda is cheapest in pure trigger cost.
S3 → SQS → Lambda is cheapest in resilience cost.
EventBridge is most expensive but most flexible.
```

### 12.10 What I'd actually do

For most production:
- S3 → EventBridge for routing (single bus, multiple consumers).
- EventBridge → SQS → Lambda for heavy async work.
- Direct Lambda for lightweight low-volume triggers.
- DLQs always configured.
- CloudWatch alarms on DLQ depth and Lambda errors.

---

## 13. Scenario 12 — Cost Optimization at Scale

### 13.1 The problem

S3 bill is $500K/month. Engineering needs to cut 30%. Where to start.

### 13.2 The diagnostic — S3 Storage Lens

```
S3 Storage Lens (free dashboard, more detailed paid):
  - Storage breakdown by bucket / prefix / class
  - Request patterns
  - Incomplete multipart uploads
  - Non-current versions taking up space
  - Objects older than X
```

This is **the first thing** to look at. Often surfaces:
- Forgotten incomplete multiparts (millions of dollars left over).
- Versioned buckets without lifecycle (history is 5× the live size).
- Wrong storage class for access pattern.
- Cross-region replication of data that's never read in the secondary region.

### 13.3 The high-impact levers (in order)

1. **Delete incomplete multipart uploads** (lifecycle rule for AbortIncompleteMultipartUpload after 7 days). One-time saving, can be millions of dollars.
2. **Lifecycle non-current versions** (versioning on heavy-write buckets accumulates). Move non-current to IA after 7 days; expire after 90.
3. **Lifecycle to IA / Glacier** for old data with predictable access drop-off.
4. **Intelligent-Tiering** for unpredictable access. Auto-tiers; small monitoring fee. **Default for unknown patterns.**
5. **Compress / reformat** to Parquet for analytics (5–10× reduction).
6. **Compact small files** (millions of 1KB files cost more in metadata + IA-min-size penalty than fewer 1MB files).
7. **Negotiate Compute Savings on egress** (Direct Connect for predictable cross-region/on-prem flow).
8. **Cross-region replication: needed?** Often replicated for "DR" but never actually fails over. Do you really need it?
9. **Request reduction** (cache HEAD/GET via CloudFront).
10. **VPC Endpoints** (Gateway endpoint is free; eliminates NAT Gateway $0.045/GB).

### 13.4 Intelligent-Tiering

```
Auto-monitors object access patterns; moves between tiers without retrieval cost:
  Frequent → Infrequent → Archive Instant → Archive Access (opt-in) → Deep Archive (opt-in)

Cost: $0.0025/1K objects/mo monitoring fee + per-tier storage.
For >128KB objects accessed unpredictably, this is usually the cheapest.
```

Enable Intelligent-Tiering on any bucket where:
- Access pattern is unpredictable.
- Objects are >128 KB (smaller objects don't get monitoring benefit).
- You don't have manual lifecycle policies that already handle it.

### 13.5 The hidden cost: requests

```
1B objects accessed once/month:
  Standard:  1B × $0.0004/1K = $400 in GET requests + $0.023/GB storage.
  IA:        1B × $0.001/1K  = $1,000 in GET requests + $0.0125/GB + retrieval.
  Glacier IR:1B × $0.01/1K   = $10,000 in GET requests.

If access frequency is high relative to storage size, IA can be MORE expensive than Standard.
```

Always model the **total** cost including request and retrieval, not just storage.

### 13.6 The egress trap

```
Cross-AZ:                $0.01/GB
Cross-region:            $0.02/GB
Egress to internet:      $0.09/GB (reduces with volume)
NAT Gateway hop:         $0.045/GB additional
```

A common cost surprise: services in private subnet → S3 via NAT Gateway. Each GB transferred pays $0.045. Fix: VPC Gateway Endpoint for S3 (free).

### 13.7 At MAANG scale: the unit economics

```
Storage:        $0.005–$0.023/GB/month (mix)
Replication:    +50–100%
Requests:       hugely variable
Egress:         often 30%+ of bill; cross-region the surprise
```

A staff-level S3 cost engineer carries the bucket-level breakdown and re-runs it monthly.

### 13.8 What I'd actually do

A 90-day S3 cost optimization project:
- Week 1: enable Storage Lens advanced metrics; audit existing.
- Week 2: lifecycle for incomplete multiparts; lifecycle for non-current versions.
- Week 3-4: bucket-by-bucket review of access patterns; move to IA / Intelligent-Tiering.
- Week 5-6: compaction job for small-file accumulations.
- Week 7-8: VPC endpoints; review CRR necessity.
- Week 9-12: Parquet conversion for analytics buckets.

Typical 30-50% saving on a previously-unmanaged S3 footprint.

---

## 14. Scenario 13 — Direct-to-S3 Browser Uploads

### 14.1 The problem

Browser uploads of files to S3 without proxying through your API tier. Tens of MB to GB. No CORS gotchas, no leaked credentials, secure.

### 14.2 The pre-signed POST policy

```python
post = s3.generate_presigned_post(
    Bucket='uploads',
    Key=f'users/{user_id}/${{filename}}',  # ${filename} variable
    Fields={'acl': 'private', 'Content-Type': 'image/jpeg'},
    Conditions=[
        {'acl': 'private'},
        {'Content-Type': 'image/jpeg'},
        ['content-length-range', 1, 100 * 1024 * 1024],   # 100MB max
        ['starts-with', '$key', f'users/{user_id}/'],     # constrain key prefix
    ],
    ExpiresIn=300,
)
return post  # has 'url' and 'fields'
```

Browser uses an HTML form:

```html
<form action={post.url} method="POST" enctype="multipart/form-data">
  {Object.entries(post.fields).map(([k,v]) =>
    <input type="hidden" name={k} value={v}/>)}
  <input type="file" name="file"/>
</form>
```

Key benefits:
- Server enforces constraints; browser cannot violate.
- No SDK on the client.
- No bytes through your tier.

### 14.3 CORS on the bucket

```yaml
CorsConfiguration:
  CorsRules:
    - AllowedOrigins: ["https://app.mycompany.com"]
      AllowedMethods: [POST, PUT, GET]
      AllowedHeaders: ["*"]
      ExposeHeaders: [ETag]
      MaxAgeSeconds: 3000
```

Without this, browser preflight fails. Tighten origins to your domains; never `*` for write methods.

### 14.4 Multipart from browser

For files >100 MB, multi-part upload:

```
1. POST /api/uploads/start   → server: CreateMultipartUpload, returns UploadId + key
2. For each chunk:
   POST /api/uploads/sign-part?uploadId&partNumber → returns pre-signed UploadPart URL
   Client: PUT chunk to URL
3. POST /api/uploads/complete  → server: CompleteMultipartUpload with parts list
```

This is a small dance but well-supported by libraries (e.g., `@aws-sdk/lib-storage`).

### 14.5 Resume on failure

If chunk upload fails: retry the chunk's pre-signed URL. Multipart upload supports this naturally — if part 5 of 100 fails, only retry part 5. Resume across browser refreshes by persisting the UploadId.

### 14.6 Server-side scanning

After upload, S3 event triggers a scan Lambda. If malicious: move to quarantine bucket; notify user. Common for SaaS with file upload.

### 14.7 What I'd actually do

For browser uploads:
- Pre-signed POST for files <100 MB.
- Multipart with pre-signed URLs for >100 MB.
- Server side validates content-type, size; tags object with user_id.
- S3 event → Lambda scan + thumbnail.
- Bucket lifecycle: move "scanned-clean" to distribution bucket; expire raw uploads after 30d.

---

## 15. Scenario 14 — High-Performance / Hot Data (S3 Express One Zone)

### 15.1 The problem

ML training reading 100K small parameter files per second. Or: a session-state store that's hotter than DynamoDB cost makes sense for. Or: high-RPS feature lookups for online inference.

### 15.2 Where Standard hits limits

```
S3 Standard:
  - 5,500 GET/sec/prefix; auto-scales beyond.
  - First-byte latency 50–200 ms.
  - Cross-AZ replication adds latency.

For a workload doing 100K random GET/sec at <10 ms: Standard struggles.
```

### 15.3 S3 Express One Zone

Released 2023. Single AZ. Sub-10 ms first-byte latency. 100K+ GET/sec single bucket. 50% cheaper request cost than Standard.

Trade-offs:
- **Single AZ**: lose AZ, lose data. Use only for **re-creatable** or **low-durability-need** data.
- **Storage cost**: ~7× Standard ($0.16/GB/mo vs $0.023). Only worth for hot data.
- **Object min size 0** (no min like IA classes).
- **Different bucket type**: directory bucket; different ARN format; different SDK calls.

### 15.4 When to use

- ML training scratch storage (re-derivable from S3 Standard backing).
- Online inference feature reads (already cached; re-fillable).
- High-RPS analytics intermediates.
- Session-state with "lose AZ = lose state" acceptable.

### 15.5 When not to use

- Anything that must survive AZ failure.
- Low-RPS data (storage cost dominates; not worth it).
- General-purpose user data.

### 15.6 Alternatives

| Approach | Trade |
|---|---|
| **S3 Standard** | 11×9 durability; ms latency; high cost at high RPS |
| **S3 Express One Zone** | sub-10ms; cheap requests; single AZ |
| **DynamoDB** | sub-10ms; multi-AZ; expensive at PB |
| **Redis / ElastiCache** | sub-ms; in-memory; expensive |
| **FSx for Lustre** | High throughput; POSIX; expensive |

### 15.7 What I'd actually do

S3 Express is niche. For most "hot data" needs, DynamoDB or ElastiCache is better. Reach for Express when:
- You need S3-style API (already integrated).
- Data is large per object (KB+, not bytes — DynamoDB has 400KB cap).
- Re-fillable from a Standard backing copy.

---

## 16. Scenario 15 — Multi-Tenant Data Isolation

### 16.1 The problem

SaaS: 10K customers; each has their own data; need:
- Customer can read only their own data.
- Customer A's data leak into Customer B's view = catastrophic.
- Per-customer encryption keys (regulatory).
- Per-customer cost allocation.
- Per-customer retention policies.

### 16.2 The bucket-per-tenant question

Pros:
- Strongest isolation (separate bucket policies).
- Easy per-tenant lifecycle, encryption, replication.
- Easy cost allocation.

Cons:
- Default 100 buckets per account (raisable).
- Operationally heavy (per-bucket lifecycle, replication, audit).
- 10K buckets ≠ scalable.

**Verdict**: bucket-per-tenant works for ≤ 100 large tenants. Beyond that, prefix-per-tenant.

### 16.3 The prefix-per-tenant pattern

```
s3://shared-bucket/tenants/{tenant_id}/...

IAM policy:
  s3:GetObject on s3://shared-bucket/tenants/${aws:userid}/*
```

Per-tenant access via dynamic IAM conditions. Single bucket, single set of bucket-level policies, per-tenant logical separation.

### 16.4 S3 Access Points per tenant

```
Bucket: shared-bucket
Access Point: tenant-a-ap   → policy granting Customer A access to /tenants/a/*
Access Point: tenant-b-ap   → policy granting Customer B access to /tenants/b/*
```

- Each AP has its own policy and ARN.
- AP-scoped access; cleaner than monolithic bucket policy.
- Cost: free.

### 16.5 Per-tenant encryption

```
SSE-KMS with customer-managed keys (CMK):
  Tenant A → key-A → encrypts only Tenant A's objects.
  Tenant B → key-B → encrypts only Tenant B's objects.

If Tenant A's key is revoked: their data becomes unreadable.
"Crypto-shredding" satisfies right-to-be-forgotten.
```

But: KMS calls cost. ~$0.03/10K calls. At 1M reads/day per tenant × 10K tenants → millions of KMS calls/day → real cost.

Alternative: **Bucket Keys** — KMS DEK (data encryption key) cached per bucket; KMS hit only on key rotation. ~99% reduction in KMS calls.

### 16.6 Cost allocation

Tag every object with `tenant_id`. Cost Allocation Reports break down storage and request cost by tag. Bill back per tenant if relevant.

### 16.7 Per-tenant retention

Lifecycle rules can be filter-based:
```yaml
Rules:
  - Filter:
      Tag: { Key: tenant_id, Value: a }
    Expiration: { Days: 365 }
  - Filter:
      Tag: { Key: tenant_id, Value: b }
    Expiration: { Days: 90 }
```

Or per-object-tag retention via Object Lock + tag-based rules.

### 16.8 What I'd actually do

For SaaS at 10K+ tenants:
- Single shared bucket with strict prefix-per-tenant.
- Bucket Owner Enforced (no ACLs).
- IAM policies + Access Points with `${aws:PrincipalTag/tenant}` conditions.
- SSE-KMS with bucket key per bucket; per-tenant KMS keys for high-tier customers needing crypto-shredding.
- Object tags with tenant_id for cost reporting.
- Lifecycle rules per tag.

---

## 17. Scenario 16 — Object Lambda for Dynamic Transformation

### 17.1 The problem

Different consumers want different views of the same object:
- Mobile clients: low-res image, JSON API.
- Web clients: high-res image, full data.
- Reporting: PII-masked CSV.
- Internal: full unredacted XML.

Without Object Lambda: pre-compute every variant and store separately (large storage cost; fixed variants).

### 17.2 Object Lambda pattern

```
Client → S3 Object Lambda Access Point → Lambda function → S3 origin

Lambda intercepts the GET, reads from S3, transforms, returns to client.
```

```python
def handler(event, context):
    s3url = event['getObjectContext']['inputS3Url']
    raw = http.get(s3url)
    transformed = redact_pii(raw)
    s3.write_get_object_response(
        Body=transformed,
        RequestRoute=event['getObjectContext']['outputRoute'],
        RequestToken=event['getObjectContext']['outputToken'],
    )
```

### 17.3 Use cases

- PII redaction by audience.
- Resize / watermark images on-demand.
- Format conversion (JSON → XML, Parquet → CSV).
- Tokenization / masking.
- Data transformation per access role.

### 17.4 Trade-offs

- **Cost**: Lambda invocation per GET (~$0.20/M + duration). At scale ($/M GET via Lambda is 10–100× S3 GET cost), only worth it for transformations that would otherwise need many pre-stored variants.
- **Latency**: adds Lambda cold start + execution time (typically +50–500 ms).
- **Caching**: Cache the transformed response in CloudFront in front of the OL Access Point.

### 17.5 Alternatives

| Approach | Trade |
|---|---|
| **Pre-compute variants** | Storage cost; fixed variants |
| **Object Lambda** | Compute cost; flexible; cacheable |
| **API tier transforming on the fly** | Same as Object Lambda but in your service |
| **S3 Select** (limited) | SQL-like over CSV/JSON/Parquet at S3 layer |

### 17.6 What I'd actually do

Reach for Object Lambda when:
- You have **many consumer roles** with different views.
- Variants are **derivable** from a single source.
- Variants are **cacheable** at CloudFront.

Skip when:
- Only 1–3 variants exist (just pre-compute).
- Transformation is heavy (Lambda 15-min limit; size constraints).

---

## 18. Scenario 17 — Time-Series / Partitioned Tables

### 18.1 The problem

IoT telemetry, financial tick data, monitoring metrics. Billions of rows/day. Time-range queries dominate ("last 30 days," "Q4 2025").

### 18.2 The partitioning by time

```
s3://telemetry/year=2026/month=04/day=28/hour=15/device-{id}.parquet
```

Hourly partitions for 1B+/day workloads. Daily for 100M/day. Monthly for low-volume.

### 18.3 Lifecycle for time-series

```yaml
Rules:
  - Filter: { Prefix: "year=2024/" }
    Transition: [{Days: 0, StorageClass: GLACIER_FLEXIBLE_RETRIEVAL}]
  - Filter: { Prefix: "year=2025/" }
    Transition: [{Days: 0, StorageClass: STANDARD_IA}]
  # year=2026 stays in Standard
```

Tier old years to cheap class. Old data still queryable (Athena on IA / Glacier IR).

### 18.4 Compaction job

Hour-partitions can have 1000s of small files (one per writer). Daily/weekly compaction job:

```
For each hour=*/day=*/month=*/year=*:
  Read all files in partition.
  Write a single file.
  Atomically replace.
```

Dramatically improves Athena/Spark scan performance.

### 18.5 Compression

Zstd-3 or higher for cold time-series. Often 10–20× compression for repetitive numeric data. Saves storage AND query I/O.

### 18.6 Iceberg time-travel

Iceberg tables have snapshot history. "What was the table state on April 1?" queryable directly via:

```sql
SELECT * FROM telemetry FOR TIMESTAMP AS OF '2026-04-01 00:00:00';
```

Useful for debugging, regulatory replay, audit.

### 18.7 Alternatives

| Approach | Trade |
|---|---|
| **S3 + Athena** | Cheap; cold queries are slow |
| **Timestream** | Managed time-series DB; can be expensive |
| **InfluxDB / TimescaleDB** | Operational; richer functions |
| **Druid / Pinot** | Sub-second queries on TB; complex to operate |
| **CloudWatch Metrics** | Best for ops monitoring; expensive at app metric scale |

### 18.8 What I'd actually do

For high-volume time-series (>1M events/sec):
- Kinesis → Firehose → S3 (Parquet, Iceberg).
- Hot 7 days in Druid/Pinot for sub-second querying.
- 30 days in Standard for Athena warm queries.
- Tier to IA / Glacier for older.
- Compaction job nightly.

---

## 19. Scenario 18 — Migration from On-Prem (PB scale)

### 19.1 The problem

Move 5 PB of on-prem data (NAS, NFS, Hadoop) to S3 within 6 months.

### 19.2 The transfer math

```
Over the internet:
  1 Gbps = 125 MB/s = ~10 TB/day = ~3.6 PB/year.
  10 Gbps = 36 PB/year = ~6 months for 5 PB. Saturated link, no other use.

You need parallel circuits or a different approach.
```

### 19.3 AWS Snow family

```
Snowcone:    14 TB; ruggedized; small.
Snowball Edge: 80 TB; with compute optionally.
Snowmobile:  100 PB shipping container; literally driven by truck.
                (Discontinued in 2024 in favor of multiple Snowballs.)
```

For 5 PB: ~60 Snowball Edge devices (80 TB each), shipped in batches. Customer fills, ships back, AWS uploads to S3 in their facility.

```
Per device: ~1 week to fill on-site (depends on local network).
Shipping + AWS ingestion: 1–2 weeks.
Parallel: 10 devices/batch → 6 batches → ~3 months.
Cost: ~$300/device + shipping = ~$20K total.

Compare: 6 months of dedicated 10 Gbps link = $50–100K.
```

### 19.4 AWS DataSync

```
Managed agent runs on-prem; pulls from NFS/SMB/HDFS, pushes to S3.
Includes:
  - Bandwidth throttling (don't saturate user network during day).
  - Scheduled transfers (overnight).
  - Verification.
  - Incremental (only changed files).
```

For ongoing sync (not one-time bulk): DataSync. For PB one-shot: Snow first, then DataSync for incremental.

### 19.5 Direct Connect

Dedicated fiber from on-prem to AWS region:
- 1, 10, or 100 Gbps.
- Lower egress cost than internet egress.
- Predictable latency.

For ongoing data flow (not one-time migration): Direct Connect. Setup is weeks; cost is monthly.

### 19.6 Storage Gateway (transitional)

```
On-prem File Gateway → caches and tiers to S3.
Looks like SMB/NFS to legacy apps.
S3 is the truth; cache is for hot files.
```

For lift-and-shift legacy apps that can't be modified to use S3 SDK directly. Buy time while you re-architect.

### 19.7 Combining: the realistic plan

```
Phase 1 (months 1–3): Bulk historical data via Snowballs.
                     ~5 PB shipped in batches.
Phase 2 (months 2–6): Direct Connect installed (overlap with Phase 1).
Phase 3 (months 4–6): DataSync over Direct Connect for ongoing delta.
Phase 4 (months 5–6): Cutover application reads to S3.
Phase 5 (months 6+):   Decommission on-prem storage.
```

### 19.8 What I'd actually do

5 PB migration:
- Snowballs for bulk historical (parallelize 5–10 devices).
- Direct Connect provisioned in month 1 (lead time).
- DataSync for incremental/delta.
- Per-app migration: Storage Gateway as bridge for legacy; rewrite for cloud-native.
- Test cutover in non-production first.
- Run dual-write for 30 days to catch divergence.
- Final cutover with documented rollback.

---

## 20. Scenario 19 — Anti-Patterns: Using S3 as a Queue / Cache / DB

### 20.1 As a queue

People do this. S3 receives objects; consumer LISTs and processes. Don't:
- LIST is paginated, not real-time.
- No exactly-once; dedup is your problem.
- Slow polling; expensive at high LIST rate.
- Eventual consistency on LIST historically (now strongly consistent for reads but LIST has subtle ordering).

**Right tool**: SQS, Kinesis, MSK.

### 20.2 As a cache

Storing computed results in S3 to avoid re-compute:
- 50–200 ms first-byte latency.
- Per-request cost.
- No TTL eviction (you must lifecycle).

**Right tool**: ElastiCache (Redis/Memcached) for sub-ms; CloudFront for HTTP-friendly results.

S3 *can* be a cache for very large items (multi-MB) where Redis would be expensive. But it's a tier of cache, not the cache.

### 20.3 As a database

Storing per-key state in S3:
- No transactions across keys.
- No conditional writes (until 2024's `If-None-Match`; even then limited).
- Slow first-byte latency.
- LIST not optimized for relational queries.

**Right tool**: DynamoDB, RDS, Aurora.

S3's `If-None-Match: *` (2024) lets you do "create if not exists" atomic. Useful for distributed locks-by-object-creation. But still: not a DB.

### 20.4 The legitimate "S3 as state" cases

- **Source of truth for blobs** (images, video, documents): S3 is the right answer.
- **Append-only event store**: S3 + Iceberg for high-throughput, low-frequency-access events. Good.
- **Snapshot store**: ZFS snapshots, DB dumps, ML model checkpoints. Good.
- **Distributed lock via conditional create**: niche but works.

The rule: S3 holds **objects** (large, immutable-ish). Anything mutable, fast, or relational: use a different service.

---

## 21. Performance and Scaling Deep Dive

### 21.1 Request rate scaling

```
Per prefix:  3,500 PUT/COPY/POST/DELETE/sec
             5,500 GET/HEAD/sec
Auto-scales beyond — but warmup needed.

For massive read fan-out (1M+ GET/sec):
  Spread across many prefixes.
  Pre-warm by predictive request.
  Or contact AWS for capacity planning.
```

The 2018 change made prefix-randomization unnecessary for *moderate* loads. At extreme scale (100K+ RPS sustained), prefixing still helps.

### 21.2 Read parallelism

```
Single S3 GET: ~50–200 ms first-byte; 80 MB/s sustained throughput.
Parallel GETs (or byte-range GETs of single object): aggregate higher.

Range GETs for parallel read of single large object:
  Range: bytes=0-1048575       (first MB)
  Range: bytes=1048576-2097151 (second MB)
  ...
  
Parallelism N: throughput ~ N × 80 MB/s (until network-limited).
```

For high-throughput read of large objects: parallel range GETs. SDKs auto-do this for downloads above threshold.

### 21.3 Write parallelism

Multipart upload (already covered §4):
- Each part uploads in parallel.
- Up to 10,000 parts.
- Aggregate speed = N × per-part throughput, until network-limited.

### 21.4 Latency

```
First-byte (single GET):
  Same-region EC2 → S3:        50–200 ms p50; 200-500 ms p99
  Cross-region:                100–300 ms p50
  Internet:                    150–500 ms p50

S3 Express One Zone:
  Same-AZ:                     <10 ms p50; 20–50 ms p99
```

For latency-sensitive reads, CloudFront (edge cached) or DynamoDB.

### 21.5 Warming up

For known-incoming high traffic:
- AWS can pre-scale a bucket if notified in advance (ticket, "request rate increase").
- Pre-warm via low-rate traffic ramping up over hours.
- Or design with prefix-spreading to start.

---

## 22. Security and Access Control Deep Dive

### 22.1 The 3 access mechanisms

```
1. IAM policy (identity-based)        — what THIS USER can do across S3.
2. Bucket policy (resource-based)     — what THESE PRINCIPALS can do on THIS BUCKET.
3. ACL (legacy, object-level)         — disabled by Bucket Owner Enforced.

Effective access = union of allows across mechanisms, MINUS any explicit denies.
Explicit deny always wins.
```

### 22.2 The deny-by-default approach

```
Block Public Access at account + bucket level (default for new buckets):
  - BlockPublicAcls
  - IgnorePublicAcls
  - BlockPublicPolicy
  - RestrictPublicBuckets
```

These are the *protective* settings. AWS now enables all four by default for new buckets. Disable only with explicit business reason.

### 22.3 The encryption choices

| Mode | Key managed by | Use when |
|---|---|---|
| **SSE-S3** | AWS, AES-256 | Default; transparent |
| **SSE-KMS** (AWS-managed key) | AWS (customer-visible KMS key) | Audit trail of key usage |
| **SSE-KMS** (customer-managed key, CMK) | You, in KMS | Key control, multi-account, crypto-shredding |
| **SSE-C** (customer-provided key per request) | You, on every request | Niche; complex |
| **Client-side** | You, before upload | Zero AWS trust; complex |

Default for compliance: SSE-KMS with CMK. Worth the small per-request cost.

### 22.4 Bucket Keys (must enable)

```
SSE-KMS without bucket key:
  Every PUT/GET hits KMS for a key encryption.
  At 1B requests/day → $30K/month KMS cost.

SSE-KMS WITH bucket key:
  KMS hit once per X minutes per bucket.
  ~99% cost reduction.
```

Always enable bucket keys when using SSE-KMS. Pure cost win, no security cost.

### 22.5 Audit via CloudTrail Data Events

```
S3 management events (CreateBucket, DeleteBucket): always logged.
S3 data events (GetObject, PutObject): NOT logged by default.

Enable CloudTrail data events on sensitive buckets:
  Cost: $0.10/100K events.
  Necessary for audit.
```

Without data events, you can't answer "who downloaded this file" — required for compliance investigations.

### 22.6 Pre-signed URL leakage

Pre-signed URLs are bearer tokens — anyone with the URL can use it. Risks:
- Logged in webserver access logs → leaks.
- Captured by network proxy → leaks.
- Bookmarked → re-accessed.

Mitigations:
- Short expiry (5 minutes typical).
- IP restriction (`s3:RemoteAddress` condition).
- VPC restriction (Access Point with VPC binding).
- Don't log full URL with query string.

### 22.7 Cross-account audit & DLP

For PII / regulated data:
- Macie scans buckets; alerts on PII discovery.
- Access Analyzer flags buckets accessible cross-account / publicly.
- GuardDuty S3 protection alerts on unusual access patterns.

These services are inexpensive and required for any serious enterprise S3 deployment.

---

## 23. Anti-Patterns — Staff-Level Red Flags

### 23.1 "We'll just put it in S3 with public read"

Public buckets are the most-leaked attack surface in cloud. Block Public Access; use CloudFront + signed URLs for distribution.

### 23.2 No lifecycle for incomplete multipart

Fixed cost waste. Always set `AbortIncompleteMultipartUpload` lifecycle rule.

### 23.3 No lifecycle for non-current versions

Versioning + heavy writes = exponential storage growth. Always lifecycle non-current.

### 23.4 "We'll list to find files"

LIST is paginated, expensive, and slow at scale. Use S3 Inventory for analysis; database index for lookup; partition prefixes for query.

### 23.5 Random keys for scaling

Pre-2018 advice: prefix with random hash for scaling. Now obsolete; auto-scales. Random keys hurt operations (can't browse, can't lifecycle by prefix). Use logical, hierarchical keys.

### 23.6 Bucket per object class

Hundreds of buckets quickly becomes operational nightmare. Use prefixes within a bucket.

### 23.7 Storing secrets in S3

S3 is not Secrets Manager. KMS-encrypt + bucket-policy + audit ≠ Secrets Manager rotation, IAM integration, lambda triggers. Use Secrets Manager for secrets.

### 23.8 Forgetting cross-account KMS permissions

Bucket grants Account B access; KMS key doesn't. Symptoms: cryptic "access denied" with no clear cause. Remember: **encryption key permissions are independent**.

### 23.9 Cross-region replication of everything

CRR doubles your bill. Often replicated "for DR" but never fails over. Replicate only what truly needs DR.

### 23.10 Glacier for "occasional access"

Glacier Flexible / Deep is hours to retrieve. If you might need it within minutes: Glacier Instant Retrieval. If you might access it once a quarter at known times: Flexible. If "we shouldn't ever need it": Deep Archive.

### 23.11 SSE-KMS without bucket keys

10–100× KMS cost. Enable bucket keys.

### 23.12 No CloudTrail data events on sensitive buckets

You'll need it for the audit you didn't anticipate.

### 23.13 No restore drills

Backup that's never restored isn't a backup. Quarterly drills.

### 23.14 LIST in hot path

LIST is slow and expensive. Cache the result; or use S3 Inventory for offline analysis; or use a side-channel index (DynamoDB / database) for lookups.

### 23.15 Misunderstanding "11 nines"

11 nines = expected loss of ~1 object per 10B/year. **Doesn't protect against**: software bugs (code deletes the wrong object), human error (wrong bucket), ransomware, regional disaster. Versioning + Object Lock + CRR + Backup are protections against those.

### 23.16 Letting CloudFront fetch directly without OAC

OAI (Origin Access Identity, legacy) and OAC (Origin Access Control, current) ensure S3 only accepts requests from your specific CloudFront distribution. Without it, the S3 URL is exposed; bypasses CloudFront cache and signed URLs.

### 23.17 Trusting object metadata for security decisions

Object metadata (`x-amz-meta-*`) is editable. Don't put security-bearing claims (user_id, role) in metadata and trust it. Use bucket/prefix policies bound to authenticated identity.

---

## 24. Decision Framework

For a new S3-using design:

### 24.1 Step 0: Establish the workload

- Object size: avg, max, distribution.
- Read pattern: random or sequential? hot or cold? RPS?
- Write pattern: append-only? overwrite-heavy? versioned?
- Retention: how long, regulated?
- Geography: single region, multi-region, global?
- Consumers: same-account, cross-account, public?
- Cost target: $/GB-mo, $/M req?

### 24.2 Step 1: Pick the storage class

```
Hot, predictable             → Standard
Hot, unpredictable           → Intelligent-Tiering
Warm, predictable            → Standard-IA
Warm, unpredictable          → Intelligent-Tiering (auto-tiers there)
Re-creatable, hot            → One Zone-IA or Express
Cold, occasional access      → Glacier Instant Retrieval
Archive, hours-to-retrieve   → Glacier Flexible / Deep Archive
```

### 24.3 Step 2: Pick the access pattern

```
Public website          → CloudFront + OAC
User uploads            → Pre-signed URL / pre-signed POST
Service-to-service      → IAM + VPC Endpoint
Cross-account           → Bucket policy + Access Point
Many tenants            → Access Points + IAM conditions
Compliance archive      → Object Lock + KMS
```

### 24.4 Step 3: Pick the encryption

```
Default                 → SSE-KMS + Bucket Key
Compliance              → SSE-KMS with CMK + key rotation
Zero AWS trust          → Client-side encryption
Per-tenant crypto-shred → KMS CMK per tenant
```

### 24.5 Step 4: Plan the lifecycle

```
Always: AbortIncompleteMultipartUpload after 7 days.
If versioned: lifecycle non-current versions.
Tier old data: Standard → IA → Glacier.
Define expiration if data has TTL.
```

### 24.6 Step 5: Plan the events

```
Real-time triggers      → S3 → EventBridge → consumers
Async batch             → S3 Inventory + Batch Operations
Replication             → S3 Replication (RTC if SLA-bound)
Audit                   → CloudTrail data events
```

### 24.7 Step 6: Plan the failure modes

- Region failure: CRR; failover plan.
- Bucket compromise: Object Lock + MFA Delete + dedicated account.
- Accidental delete: Versioning + lifecycle for old versions.
- Cost runaway: Budget alarms; Storage Lens; lifecycle tuning.

### 24.8 Step 7: Plan observability

- CloudTrail data events on sensitive buckets.
- Storage Lens metrics.
- CloudWatch alarms on bucket size, request rates.
- Cost alarms.

---

## 25. Mental Models a Staff Engineer Carries

1. **S3 is a flat key-value store.** Hierarchy is convention; "/" is a character.

2. **Storage class is the biggest cost lever.** 23×–230× ratio between Standard and Deep Archive.

3. **Lifecycle is not optional.** Set up the abort-incomplete-multipart rule day 1.

4. **Versioning without lifecycle is unbounded growth.**

5. **Object Lock = real WORM.** SEC-compliant. Cannot be undone by anyone.

6. **CRR is for DR only when you've drilled it.** Otherwise it's a cost.

7. **Public buckets are leaks waiting to happen.** Block Public Access by default.

8. **Pre-signed URLs offload the bytes.** Don't proxy through your tier.

9. **CloudFront in front of S3 is a 90%+ cost saver** for distributable assets.

10. **VPC endpoints save NAT-gateway egress.** Free; enable always.

11. **Bucket Keys save 99% of KMS cost.** Enable always with SSE-KMS.

12. **LIST is expensive and slow.** Don't put it in hot paths.

13. **Cross-account requires KMS permissions, not just bucket policy.**

14. **S3 events are at-least-once, unordered.** Make consumers idempotent.

15. **Strong consistency since 2020.** PUT then GET sees the new version. But replication is still async.

16. **11 nines protects against hardware loss.** Not against your code, your users, your attackers.

17. **Multipart upload is mandatory for files >100 MB.** Single PUT max is 5 GB.

18. **Athena's bill scales with bytes scanned.** Partitioning + Parquet = 100× cost reduction.

19. **Iceberg is the modern data-lake table format.** Standard at AWS now.

20. **S3 Express is niche.** DynamoDB or Redis is usually right for "hot data."

21. **Object Lambda is for variants.** Pre-compute when variants are few and known.

22. **CDN edges hold most of the read traffic.** S3 is the origin, not the serving layer.

23. **Mountpoint / FSx for Lustre give POSIX-feel.** Each has caveats.

24. **One AZ class loses on AZ failure.** Use only for re-creatable data.

25. **Cost optimization is a 90-day project, not a 1-day fix.** Storage Lens → lifecycle → tier → compress → audit.

26. **Audit data events on sensitive buckets.** Free is ignorance, not safety.

27. **Restore drills make backups real.** Quarterly. No exceptions.

28. **Boring is a feature.** A 1 PB S3 footprint that quietly serves traffic and stays under budget is the highest engineering accomplishment.

---

## 26. Closing Notes

S3 is the most underestimated AWS service: deceptively simple, infinitely scalable, full of subtle expensive surprises. The staff-level skill is not memorizing every feature — it's developing the **judgment** to combine them appropriately.

The pattern across every scenario:
- Pick the **right storage class** for the access pattern.
- Define a **lifecycle policy** to control cost over time.
- Use **CloudFront** for distributable reads.
- Use **events** for asynchronous processing.
- Use **encryption** matched to compliance.
- Use **Object Lock / Versioning** for ransomware protection.
- Use **Access Points / Bucket Policies** for sharing.
- **Audit, observe, and drill** the recovery paths.

Done well, S3 is the most reliable PB-scale storage tier in the industry, at unmatched price. Done poorly, it's a $500K/month bill with publicly-accessible PII and no recoverable backups.

The art is doing it well.

> Companion docs:
> - `statefulSystemsAtMAANGScale.md` — designing the stateful tier S3 often plays in.
> - `awsCostOptimization.md` — broader cost engineering.
> - `lambdaStepFunctionsAtScale.md` — event-driven processing of S3 events.
> - `databaseTransactionScenarios.md` — when transactions vs S3.