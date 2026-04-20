Q) Design a log aggregation service that handles 10TB/hr of data from various services. The system should support real-time data viewing for debugging, allow regex queries for exact  
data retrieval in real-time, and provide the analytics team with data from the past 24-48 hours in the form of a tar file.

10TB/hr of data being sent from a lot of services. there are 3 main flows -

teams should be able to see the data in real time for debugging                                                                                                                     
The team can get exact data by issuing a regex query to the logs in real time.                                                                                                      
The analytics team can look at data from 24 - 48 hrs, which they receive in form of a tar file.

⏺ Log Aggregation Service — 10 TB/hr

Numbers First

Ingestion:          10 TB/hr  =  2.78 GB/s  =  ~170 GB/min
Avg log line:       ~500 bytes (mixed structured + unstructured)
Lines per second:   ~5.5 million
24-hr storage:      240 TB raw  →  ~80 TB compressed (zstd ~3x)
48-hr storage:      480 TB raw  →  ~160 TB compressed

With LZ4 on the wire (producer → Kafka), effective ingestion bandwidth drops to ~1.4 GB/s. Compression is your single biggest lever.

  ---
High-Level Architecture

┌──────────────────────────────────────────────────────────────────────────┐
│                          PRODUCER LAYER                                  │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐         ┌─────────┐                 │
│  │Service A│ │Service B│ │Service C│   ...   │Service N│                  │   
│  └────┬────┘ └────┬────┘ └────┬────┘         └────┬────┘                  │
│       │            │            │                    │                    │
│  ┌────▼────┐ ┌────▼────┐ ┌────▼────┐         ┌────▼────┐                  │
│  │  Agent  │ │  Agent  │ │  Agent  │   ...   │  Agent  │                  │
│  │(Vector) │ │(Vector) │ │(Vector) │         │(Vector) │                  │
│  └────┬────┘ └────┬────┘ └────┬────┘         └────┬────┘                  │
└───────┼────────────┼────────────┼──────────────────┼─────────────────────┘
│            │            │                    │
▼            ▼            ▼                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                         KAFKA CLUSTER                                    │
│                                                                          │
│   Topic: logs.service-a  [P0][P1][P2][P3]     Retention: 6 hrs         │
│   Topic: logs.service-b  [P0][P1][P2][P3]     Replication: 3           │
│   Topic: logs.service-c  [P0][P1][P2][P3]     Compression: LZ4        │
│   ...                                                                    │
│   20-25 brokers, NVMe SSDs, 10Gbps+ NICs                               │
│                                                                          │
└─────┬──────────────────┬──────────────────┬──────────────────────────────┘
      │                  │                    │
      ▼                  ▼                        ▼
┌───────────┐    ┌──────────────┐    ┌──────────────┐
│  FLOW 1   │    │   FLOW 2     │    │   FLOW 3     │
│  Live Tail│    │ Regex Query  │    │ Analytics    │
│  Service  │    │   Engine     │    │ Export       │
└───────────┘    └──────────────┘    └──────────────┘

Three independent consumer paths off the same Kafka topics. They don't interfere with each other (separate consumer groups). Let me go deep on each.

  ---
Ingestion Layer

Log Agents (per host)

┌─────────────────────────────────────────────┐
│              Application Host                │
│                                              │
│  ┌──────────┐    stdout/file                 │
│  │ Service  │──────────────┐                 │
│  └──────────┘              │                 │
│                       ┌────▼──────────────┐  │
│                       │   Vector Agent     │  │
│                       │                    │  │
│                       │ • Read from file/  │  │
│                       │   stdout/journald  │  │
│                       │ • Parse + enrich:  │  │
│                       │   service_name,    │  │
│                       │   host, timestamp, │  │
│                       │   log_level, env   │  │
│                       │ • Batch (64KB-1MB) │  │
│                       │ • Compress (LZ4)   │  │
│                       │ • Disk buffer      │  │
│                       │   (5GB spillover)  │  │
│                       └────────┬───────────┘  │
│                                │              │
└────────────────────────────────┼──────────────┘
│  TCP/gRPC
▼
Kafka

Why Vector over Filebeat/Fluentd:
- Rust-based, single binary, ~10x less memory than Fluentd
- Built-in disk buffer (critical — if Kafka is unreachable, logs spill to disk, not lost)
- Transforms at the edge: parse JSON, extract fields, sample, filter — reduce what hits Kafka

Log envelope format (what the agent sends):

{
"ts":        "2026-04-19T14:23:01.847Z",   // event timestamp (from service)
"ingest_ts": "2026-04-19T14:23:01.952Z",   // agent receive time
"service":   "payment-api",
"host":      "ip-10-0-3-47",
"env":       "production",
"level":     "ERROR",
"trace_id":  "abc123def456",
"msg":       "charge failed: card_declined for user_id=98712"
}

Two timestamps because they solve different problems. ts is what the developer cares about (event ordering). ingest_ts is what the system cares about (data placement, late-arrival
detection).

Kafka Cluster

Sizing:

Write throughput:     ~1.4 GB/s compressed (2.78 GB/s raw × 0.5 LZ4 ratio)
Replication factor:   3 → 4.2 GB/s total cluster write
Per broker (NVMe):    ~600 MB/s sustained write
Brokers needed:       4.2 / 0.6 = 7 minimum → 20 brokers (for headroom + reads)
Retention:            6 hours = ~30 TB compressed on-cluster
Partitions:           Topic per service. 4-16 partitions per topic.
If 200 services → ~1600 partitions total. Well within limits.

Why topic-per-service (not a single logs topic):
- Flow 1 (live tail) needs to consume only one service's logs. With a single topic, you'd read all 2.78 GB/s just to filter for one service.
- Topic-per-service lets consumers read only the relevant partitions.
- Partition count per topic is tuned to that service's throughput.

Kafka topic creation policy:

High-volume service  (>100 MB/s):  16 partitions
Medium service       (10-100 MB/s): 8 partitions
Low-volume service   (<10 MB/s):    4 partitions

Why 6-hour retention (not 48):
- 48 hours at 1.4 GB/s compressed = ~240 TB on Kafka brokers. That's 20 brokers × 12 TB each — doable but expensive (NVMe).
- Kafka is a transit buffer, not long-term storage. S3 is 10x cheaper per GB.
- 6 hours gives consumers enough runway to recover from failures without data loss.
- Long-term data lives in S3 (Flow 3 writes it there within minutes of ingestion).

  ---
Flow 1: Real-Time Tail (Live Debugging)

┌──────────┐       ┌──────────────────────────────────────┐
│  Kafka   │       │         Tail Service (cluster)       │
│  Topic:  │       │                                      │
│  logs.   │──────▶│  ┌──────────────────────────────┐    │
│  payment │       │  │   Kafka Consumer (per topic)  │   │
│  -api    │       │  │   reads at tail of partition   │   │
│          │       │  └──────────┬─────────────────────┘   │
└──────────┘       │             │                         │
│    ┌────────▼────────┐                                   │
│    │  Subscription   │                 │
│    │  Router         │                 │
│    │                 │                 │
│    │  Evaluates each │                 │
│    │  line against   │                 │
│    │  subscriber's   │                 │
│    │  filters:       │                 │
│    │  • log level    │                 │
│    │  • keyword      │                 │
│    │  • regex        │                 │
│    │  • trace_id     │                 │
│    └──┬─────┬────┬───┘                 │
│       │     │    │                      │
│       ▼     ▼    ▼                      │
│     Sub1  Sub2  Sub3                    │
│    (WSS) (WSS) (SSE)                   │
└──────────────────────────────────────── │
│     │    │
▼     ▼    ▼
Browser/CLI/Dashboard

How it works:

1. User opens the debug UI or runs the CLI: logtail --service payment-api --level ERROR
2. The Tail Service registers the subscription: {service: "payment-api", filter: level=ERROR}
3. If no Kafka consumer exists for that topic, one is created. Consumer reads from the latest offset (no need for historical data — that's Flow 2).
4. Each message is evaluated against all active subscriptions for that topic.
5. Matching lines are pushed over WebSocket/SSE to the subscriber.

Key design decisions:

- Consumer sharing. If 20 developers are tailing the same service, you don't create 20 Kafka consumers. One consumer reads the topic and fans out to 20 WebSockets. This keeps Kafka
  consumer count manageable.
- Backpressure per subscriber. A slow WebSocket client (bad network) must not block other subscribers. Each subscriber gets a bounded ring buffer (e.g., 10,000 lines). If it fills,
  oldest lines are dropped for that subscriber only. The client sees a "X lines dropped" marker.
- Filter evaluation order. Cheapest filters first: log level (enum comparison) → keyword (string contains) → regex (expensive). Short-circuit: if level doesn't match, skip regex
  evaluation.

Scaling:

20 Tail Service instances behind a load balancer.
Each instance handles subscriptions for a subset of topics.
Sticky routing: all subscribers for topic X go to the same instance
(so only one Kafka consumer per topic exists cluster-wide).

If a single topic has so many subscribers that one instance can't
fan out fast enough → shard by partition: instance 1 handles
partitions [0,1], instance 2 handles [2,3], etc.

Edge cases:

┌─────────────────────────────────────────────────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                      Case                       │                                                          Handling                                                           │
├─────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Service produces 500 MB/s, subscriber's regex   │ Regex timeout: 1ms per line. Lines exceeding timeout are skipped. Use RE2 (linear-time guarantee).                          │
│ is slow                                         │                                                                                                                             │
├─────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ 1000 subscribers on one service                 │ Fan-out via broadcast. Ring buffers per subscriber. If still bottlenecked, shard by partition.                              │
├─────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Subscriber asks for a service that doesn't      │ Return error immediately (topic doesn't exist in Kafka).                                                                    │
│ exist                                           │                                                                                                                             │
├─────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Subscriber disconnects mid-stream               │ Detect via WebSocket close/heartbeat timeout. Remove subscription. If last subscriber for topic, stop Kafka consumer after  │
│                                                 │ 60s grace period.                                                                                                           │
└─────────────────────────────────────────────────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

  ---
Flow 2: Real-Time Regex Query

This is the hardest flow. Two sub-modes:

Mode A: Forward-Looking (Live Grep)

Identical to Flow 1 with a regex filter. User issues:
logquery --service payment-api --regex "card_declined.*user_id=\d+" --mode stream
The system subscribes to the live stream with the regex. Matching lines flow in real-time. Already covered by Flow 1.

Mode B: Historical Search (Backward-Looking)

User issues:
logquery --service payment-api --regex "card_declined.*user_id=\d+" \
--from "2h ago" --to "now"

This is a search over 2 hours of ingested data. For a service producing 1% of total traffic: 2 hours × 100 GB/hr = 200 GB compressed to scan. This requires a purpose-built query
engine.

┌─────────────────────────────────────────────────────────────────────┐
│                     REGEX QUERY ENGINE                               │
│                                                                      │
│  ┌──────────┐     ┌──────────────┐     ┌─────────────────────────┐  │
│  │  Query   │     │   Chunk      │     │     S3 Object Store     │  │
│  │  API     │────▶│   Planner    │────▶│                         │  │
│  │          │     │              │     │  s3://logs/             │  │
│  │  Accepts │     │  • Resolves  │     │    payment-api/         │  │
│  │  regex + │     │    service + │     │      2026/04/19/        │  │
│  │  service │     │    time range│     │        12/              │  │
│  │  + time  │     │    → list of │     │          chunk_001.zst  │  │
│  │  range   │     │    S3 chunks │     │          chunk_002.zst  │  │
│  │          │     │              │     │          chunk_003.zst  │  │
│  └──────────┘     │  • Estimates │     │        13/              │  │
│       ▲           │    scan cost │     │          chunk_001.zst  │  │
│       │           │              │     │          ...            │  │
│       │           └──────┬───────┘     └─────────────────────────┘  │
│       │                  │                          ▲                 │
│       │                  ▼                          │                 │
│       │           ┌──────────────┐                  │                 │
│       │           │   Scatter    │    Download +    │                 │
│       │           │   Controller │    decompress +  │                 │
│       │           │              │    regex scan    │                 │
│       │           │  Dispatches  │──────────────────┘                 │
│       │           │  chunks to   │                                    │
│       │           │  scan workers│                                    │
│       │           └──────┬───────┘                                    │
│       │                  │                                            │
│       │         ┌────────┼────────┐                                   │
│       │         ▼        ▼        ▼                                   │
│       │    ┌────────┐┌────────┐┌────────┐                            │
│       │    │Worker 1││Worker 2││Worker N│    50-100 stateless        │
│       │    │        ││        ││        │    workers (spot instances) │
│       │    │ zstd   ││ zstd   ││ zstd   │                            │
│       │    │ decomp ││ decomp ││ decomp │    Each: 8 vCPU, 16GB RAM │
│       │    │ + RE2  ││ + RE2  ││ + RE2  │    Scans ~500MB/s/core    │
│       │    │ scan   ││ scan   ││ scan   │                            │
│       │    └───┬────┘└───┬────┘└───┬────┘                            │
│       │        │         │         │                                  │
│       │        ▼         ▼         ▼                                  │
│       │    ┌────────────────────────────┐                             │
│       │    │   Result Aggregator        │                             │
│       └────│   • Merge + sort by ts     │                             │
│            │   • Stream to client       │                             │
│            │   • Apply LIMIT if set     │                             │
│            └────────────────────────────┘                             │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘

Chunk Layout on S3

Chunks are written by a Chunk Writer — a Kafka consumer group that reads from all topics and writes compressed chunks to S3.

s3://log-chunks/
└── {service_name}/
└── {year}/{month}/{day}/{hour}/
└── {minute_bucket}_{partition}_{uuid}.log.zst

Example:
s3://log-chunks/payment-api/2026/04/19/14/00-04_p3_a1b2c3.log.zst
^^
5-minute bucket (00-04 means minutes 0-4)

Chunk sizing target: 64-128 MB compressed (200-400 MB raw).

Why this size:
- Small enough that a single worker scans one chunk in <1 second
- Large enough that S3 PUT overhead is amortized
- Aligns with S3 multipart upload parts

Chunk writer mechanics:

Kafka consumer → In-memory buffer (per service, per partition)
│
├── Buffer reaches 128 MB compressed → flush to S3
├── Buffer age reaches 5 minutes     → flush to S3
└── Flush is idempotent (deterministic S3 key)

Consumer group: "chunk-writer"
Parallelism:    One consumer per partition. ~100 consumer instances.

Query Execution Walkthrough

User query: service=payment-api, regex=card_declined.*, from=2h ago, to=now

Step 1 — Chunk Planning:
# Resolve time range to S3 prefixes
prefixes = []
for hour in [12, 13, 14]:  # 2h ago to now
prefixes.append(f"s3://log-chunks/payment-api/2026/04/19/{hour:02d}/")

# List all chunks in those prefixes
chunks = s3.list_objects(prefixes)
# Result: 150 chunks, ~64 MB each, ~9.6 TB total (for a high-volume service)
# For a 1%-of-traffic service: ~96 GB total

Step 2 — Cost Estimation & Guard:
total_scan_bytes = sum(c.size for c in chunks)

if total_scan_bytes > MAX_SCAN_LIMIT:  # e.g., 500 GB
return error("Query would scan {total_scan_bytes}. "
"Narrow your time range or service filter.")

This prevents a user from accidentally scanning 10 TB.

Step 3 — Scatter to Workers:
# Distribute chunks across available workers
worker_assignments = distribute(chunks, num_workers=50)

# Each worker gets ~3 chunks (150 / 50)
for worker, assigned_chunks in worker_assignments:
dispatch(worker, {
"chunks": assigned_chunks,
"regex": compiled_regex,  # pre-validated RE2
"max_matches": 10000,     # per-worker limit
})

Step 4 — Worker Scan:
def scan_chunk(chunk_key, regex, max_matches):
matches = []
stream = s3.get_object(chunk_key).stream()
decompressor = zstd.StreamDecompressor()

      for line in decompressor.decompress_lines(stream):
          if regex.match(line):
              matches.append(parse_log_line(line))
              if len(matches) >= max_matches:
                  break

      return matches

Step 5 — Gather & Stream:
Results from all workers are merge-sorted by timestamp and
streamed to the client as they arrive (not waiting for all
workers to finish). Client sees first results in ~2-3 seconds.

Performance Estimate

Scenario: Service produces 1% of traffic (100 GB/hr)
Query:    Last 2 hours → 200 GB compressed
Workers:  50
Per worker: 200 GB / 50 = 4 GB
S3 read:   ~500 MB/s per stream
Decompress: zstd at ~1 GB/s (CPU-bound on 8-core worker)
RE2 scan:  ~800 MB/s per core

Wall clock: ~4 GB / 500 MB/s = 8 seconds (S3 download dominates)
First results: ~2 seconds (streaming as chunks complete)

For a smaller service (0.1% of traffic, 10 GB/hr), the same query completes in < 1 second.

Critical Safeguards

┌───────────────────────────────────────────────────┐
│              QUERY GUARDRAILS                       │
│                                                     │
│  1. RE2 only (no PCRE backtracking)                │
│     → guarantees linear-time regex execution        │
│     → rejects lookaheads, backreferences            │
│                                                     │
│  2. Scan budget per query: 500 GB max               │
│     → "narrow your time range" error if exceeded    │
│                                                     │
│  3. Result limit: 100,000 lines per query           │
│     → prevents OOM on aggregator                    │
│                                                     │
│  4. Timeout: 60 seconds per query                   │
│     → partial results returned with "truncated" flag│
│                                                     │
│  5. Concurrency quota per team: 5 concurrent queries│
│     → prevents one team starving the worker pool    │
│                                                     │
│  6. Regex pre-validation at API layer               │
│     → reject invalid regex before dispatching       │
│     → estimate selectivity if possible              │
└───────────────────────────────────────────────────┘

  ---
Flow 3: Analytics Export (24-48hr Tar)

┌──────────────────────────────────────────────────────────────────┐
│                    ANALYTICS EXPORT PIPELINE                      │
│                                                                   │
│   ┌─────────┐     ┌──────────────┐     ┌──────────────────────┐ │
│   │  Export  │     │  Manifest    │     │   S3 Chunk Store     │ │
│   │  API /   │────▶│  Builder     │────▶│                      │ │
│   │  Cron    │     │              │     │  Lists all chunks    │ │
│   │          │     │  Resolves:   │     │  for the requested   │ │
│   │  POST    │     │  service(s)  │     │  service + time      │ │
│   │  /export │     │  + time      │     │  range               │ │
│   │  {       │     │  range       │     │                      │ │
│   │  service,│     │  → chunk     │     └──────────┬───────────┘ │
│   │  from,   │     │    manifest  │                │              │
│   │  to      │     │              │                │              │
│   │  }       │     └──────────────┘                │              │
│   └─────────┘                                      │              │
│                                                     ▼              │
│                                    ┌────────────────────────────┐ │
│                                    │   Tar Builder (batch job)  │ │
│                                    │                            │ │
│                                    │  For each chunk:           │ │
│                                    │   1. S3 GET (streaming)    │ │
│                                    │   2. Decompress zstd       │ │
│                                    │   3. Append to tar stream  │ │
│                                    │   4. Recompress with gzip  │ │
│                                    │      (analytics tools      │ │
│                                    │       expect .tar.gz)      │ │
│                                    │   5. S3 multipart PUT to   │ │
│                                    │      output bucket         │ │
│                                    │                            │ │
│                                    │  Streams directly:         │ │
│                                    │  S3 GET → decompress →     │ │
│                                    │  tar → gzip → S3 PUT      │ │
│                                    │  (never holds full dataset │ │
│                                    │   in memory)               │ │
│                                    └──────────────┬─────────────┘ │
│                                                   │               │
│                                                   ▼               │
│                                    ┌────────────────────────────┐ │
│                                    │  Output Bucket             │ │
│                                    │                            │ │
│                                    │  s3://log-exports/         │ │
│                                    │    payment-api/            │ │
│                                    │      2026-04-18.tar.gz     │ │
│                                    │      2026-04-19.tar.gz     │ │
│                                    │                            │ │
│                                    │  Presigned URL generated   │ │
│                                    │  → sent to analytics team  │ │
│                                    │    via Slack/email/API     │ │
│                                    └────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘

Export Granularity

A single 48-hour tar of ALL services would be ~160 TB compressed. That's not practical as one file. The export is per-service (or per-service-group):

POST /api/v1/export
{
"service": "payment-api",           // or ["payment-api", "order-api"]
"from":    "2026-04-17T00:00:00Z",
"to":      "2026-04-19T00:00:00Z",  // 48 hours
"format":  "tar.gz",
"notify":  "analytics-team@company.com"
}

Response:
{
"export_id": "exp_abc123",
"status":    "queued",
"estimated_size": "45 GB",
"estimated_time": "~20 minutes",
"poll_url": "/api/v1/export/exp_abc123"
}

Streaming Tar Construction

The tar builder never holds the full dataset in memory. It streams:

def build_tar_export(manifest, output_key):
chunks = manifest.chunks  # ordered by time

      with S3MultipartUpload(output_key) as upload:
          with gzip.GzipWriter(upload) as gz:
              with tarfile.open(fileobj=gz, mode='w|') as tar:
                  for chunk in chunks:
                      # Stream from S3, decompress, add as tar member
                      stream = s3.get_object_stream(chunk.key)
                      decompressed = zstd.decompress_stream(stream)

                      # Add to tar as: payment-api/2026-04-19/14/chunk_001.log
                      member = tarfile.TarInfo(name=chunk.logical_path)
                      member.size = chunk.raw_size
                      tar.addfile(member, decompressed)

      # Memory usage: ~128 MB buffer regardless of export size

Scheduling

Option A: On-demand (API call from analytics team)
Option B: Cron (daily at 02:00 UTC, export previous 24h for all services)
Option C: Both — cron for standard exports, API for ad-hoc

Cron approach for 200 services:
- 200 parallel export jobs (one per service)
- Each reads from S3, builds tar, writes to output bucket
- Total read: ~80 TB (24hr compressed)
- At 200 parallel streams × 500 MB/s each = 100 GB/s aggregate read
- Wall time: ~80 TB / 100 GB/s = ~800 seconds ≈ 13 minutes
- (S3 handles this easily — it's designed for this)

Tar Internal Structure

payment-api-2026-04-18.tar.gz
│
├── payment-api/
│   ├── 2026-04-18/
│   │   ├── 00/
│   │   │   ├── 00-04_p0.log      # 5-min bucket, partition 0
│   │   │   ├── 00-04_p1.log
│   │   │   ├── 05-09_p0.log
│   │   │   └── ...
│   │   ├── 01/
│   │   │   └── ...
│   │   └── 23/
│   │       └── ...
│   └── manifest.json              # metadata: line counts, byte counts,
│                                  #   time range, schema version

The analytics team can extract specific hours without decompressing the entire archive: tar -xzf payment-api-2026-04-18.tar.gz payment-api/2026-04-18/14/

  ---
Chunk Writer — The Shared Component

The chunk writer serves both Flow 2 and Flow 3. It's the most critical pipeline component.

┌───────────────────────────────────────────────────────────────┐
│                      CHUNK WRITER                              │
│                                                                │
│  Kafka Consumer Group: "chunk-writer"                         │
│  Instances: 80-120 (one consumer per Kafka partition)         │
│                                                                │
│  Per partition:                                                │
│  ┌────────────────────────────────────────────────────┐       │
│  │  In-memory buffer (per service × time bucket)       │       │
│  │                                                     │       │
│  │  Key: (service=payment-api, bucket=14:00-14:04)    │       │
│  │  Value: CompressedBuffer (zstd streaming)           │       │
│  │                                                     │       │
│  │  Flush when:                                        │       │
│  │    buffer.compressed_size >= 64 MB  OR              │       │
│  │    buffer.age >= 5 minutes          OR              │       │
│  │    process is shutting down                          │       │
│  │                                                     │       │
│  │  On flush:                                          │       │
│  │    1. Finalize zstd frame                           │       │
│  │    2. S3 PUT to deterministic key                   │       │
│  │    3. Commit Kafka offset                           │       │
│  │    4. Reset buffer                                  │       │
│  └────────────────────────────────────────────────────┘       │
│                                                                │
│  Ordering guarantee:                                           │
│    Within a chunk, lines are in Kafka offset order.           │
│    Chunks are non-overlapping in time (by flush boundary).    │
│                                                                │
│  Failure handling:                                             │
│    • S3 PUT fails → retry 3x with backoff → DLQ the batch    │
│    • Consumer rebalance → flush open buffers before revoking  │
│    • Process crash → uncommitted offsets mean Kafka replays   │
│      → duplicate data in S3, but chunk key is deterministic   │
│      → S3 PUT overwrites with identical data (idempotent)     │
│                                                                │
└───────────────────────────────────────────────────────────────┘

Why 5-minute buckets:
- Granular enough for time-range queries (2-hour query scans ~24 buckets, not 2 huge files)
- Coarse enough that S3 LIST operations return a manageable number of objects
- Aligns with typical debugging time windows ("what happened in the last 5 minutes?")

  ---
Data Lifecycle & Retention

┌──────────┐    ┌──────────┐    ┌──────────────┐    ┌────────────┐
│  Kafka   │    │  S3 Hot  │    │  S3 Glacier  │    │  Deleted   │
│  6 hours │───▶│  48 hours│───▶│  90 days     │───▶│            │
│          │    │          │    │  (optional)  │    │            │
│ NVMe SSD │    │ Standard │    │  $0.004/GB   │    │            │
│ ~$0.10/GB│    │ $0.023/GB│    │              │    │            │
└──────────┘    └──────────┘    └──────────────┘    └────────────┘

S3 Lifecycle Policy:
- Chunks older than 48hr → transition to S3 Glacier Instant Retrieval
- Chunks older than 90 days → delete
- Export tar files → delete after 7 days (analytics team has had time to fetch)

  ---
Failure Modes & Mitigations

Agent Layer

┌────────────────────────────┬───────────────────────────────────┬─────────────────────────────────────────────────────────────────────────────────────────────┐
│          Failure           │              Impact               │                                         Mitigation                                          │
├────────────────────────────┼───────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────┤
│ Agent crashes              │ Logs accumulate on disk           │ Container runtime (Docker/K8s) captures stdout. Agent restarts, reads from file checkpoint. │
├────────────────────────────┼───────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────┤
│ Host disk full             │ Agent can't buffer                │ Alert on disk usage > 80%. Agent drops oldest buffered data (configurable).                 │
├────────────────────────────┼───────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────┤
│ Kafka unreachable          │ Logs back up in agent disk buffer │ 5 GB disk buffer = ~30 minutes at moderate volume. Alert if buffer > 50%.                   │
├────────────────────────────┼───────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────┤
│ Agent can't parse log line │ Line lost or mangled              │ Ship raw unparsed line with parse_error=true tag. Don't drop data.                          │
└────────────────────────────┴───────────────────────────────────┴─────────────────────────────────────────────────────────────────────────────────────────────┘

Kafka Layer

┌───────────────────────┬─────────────────────────────────────────────┬──────────────────────────────────────────────────────────────────────┐
│        Failure        │                   Impact                    │                              Mitigation                              │
├───────────────────────┼─────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────┤
│ Single broker down    │ No data loss (RF=3)                         │ ISR handles automatically. Alert if under-replicated partitions > 0. │
├───────────────────────┼─────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────┤
│ Multiple brokers down │ Possible unavailability                     │ Min ISR=2 with RF=3 means 2 brokers can fail before data loss.       │
├───────────────────────┼─────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────┤
│ Disk full on broker   │ Broker rejects writes                       │ Retention policy (6hr) + monitoring. Alert at 70% disk.              │
├───────────────────────┼─────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────┤
│ Consumer lag growing  │ Data at risk of expiring before consumption │ Alert when lag > 2 hours. Auto-scale consumer group.                 │
└───────────────────────┴─────────────────────────────────────────────┴──────────────────────────────────────────────────────────────────────┘

Chunk Writer

┌─────────────────────────────────────────┬───────────────────────────────┬─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                 Failure                 │            Impact             │                                             Mitigation                                              │
├─────────────────────────────────────────┼───────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Writer crash mid-flush                  │ Partial chunk on S3           │ On restart, replay from last committed Kafka offset. Overwrite partial chunk (idempotent key).      │
├─────────────────────────────────────────┼───────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ S3 outage                               │ Chunks can't be written       │ Kafka retention (6hr) buffers the gap. Alert immediately. Writer retries indefinitely.              │
├─────────────────────────────────────────┼───────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Consumer rebalance storm                │ Temporary pause in chunk      │ cooperative-sticky assignor. Flush buffers on partition revocation.                                 │
│                                         │ writing                       │                                                                                                     │
├─────────────────────────────────────────┼───────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Late-arriving logs (30+ min old         │ Logs in wrong time bucket     │ Use ingestion timestamp for bucket placement, event timestamp for sorting within chunk. Metadata    │
│ timestamps)                             │                               │ records both.                                                                                       │
└─────────────────────────────────────────┴───────────────────────────────┴─────────────────────────────────────────────────────────────────────────────────────────────────────┘

Query Engine

┌────────────────────────────────────────────────┬───────────────────────┬─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                    Failure                     │        Impact         │                                             Mitigation                                              │
├────────────────────────────────────────────────┼───────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Pathological regex (catastrophic backtracking) │ Worker hangs          │ RE2 only (linear-time). No PCRE. Impossible by construction.                                        │
├────────────────────────────────────────────────┼───────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Worker OOM during scan                         │ Query fails partially │ Workers are stateless + auto-restarted. Scatter controller retries failed chunks on another worker. │
├────────────────────────────────────────────────┼───────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Too many concurrent queries                    │ Worker pool exhausted │ Per-team concurrency quota. Queue excess queries with position indicator.                           │
├────────────────────────────────────────────────┼───────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ S3 throttling (GET rate limit)                 │ Slow scans            │ Spread reads across S3 prefixes (already done by time-bucketing). Use S3 request rate partitioning. │
└────────────────────────────────────────────────┴───────────────────────┴─────────────────────────────────────────────────────────────────────────────────────────────────────┘

Export Pipeline

┌──────────────────────────────┬──────────────────────────────────────┬────────────────────────────────────────────────────────────────────────────────────────────────────┐
│           Failure            │                Impact                │                                             Mitigation                                             │
├──────────────────────────────┼──────────────────────────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Export job crashes mid-tar   │ Partial tar on S3                    │ Idempotent: restart from manifest. S3 multipart upload is aborted, restarted.                      │
├──────────────────────────────┼──────────────────────────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Chunks deleted before export │ Incomplete export                    │ Export runs within retention window (48hr). Alert if export latency approaches retention boundary. │
├──────────────────────────────┼──────────────────────────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Output tar too large (>5 TB) │ S3 multipart limit (5 TB per object) │ Split into daily tars. Or split by hour. Manifest lists all parts.                                 │
└──────────────────────────────┴──────────────────────────────────────┴────────────────────────────────────────────────────────────────────────────────────────────────────┘

  ---
Monitoring the Monitoring

The log aggregation service itself needs monitoring. This creates a meta-problem: where do the log service's own logs go?

┌─────────────────────────────────────────────────────────┐
│  SELF-MONITORING (avoid circular dependency)             │
│                                                          │
│  The log service's own metrics go to a SEPARATE           │
│  lightweight Prometheus/Grafana stack.                    │
│  NOT through the log aggregation pipeline itself.         │
│                                                          │
│  Key metrics:                                             │
│  • kafka.consumer_lag (per consumer group, per partition) │
│  • chunk_writer.flush_latency_p99                        │
│  • chunk_writer.s3_put_errors                            │
│  • query_engine.scan_bytes_per_query                     │
│  • query_engine.active_queries                           │
│  • tail_service.active_subscriptions                     │
│  • tail_service.dropped_lines_per_subscriber             │
│  • export.job_duration                                    │
│  • export.bytes_exported                                  │
│  • agent.buffer_utilization (per host)                   │
│                                                          │
│  Alerts:                                                  │
│  • Consumer lag > 2 hours         → P1                   │
│  • S3 PUT errors > 0 for 5 min   → P1                   │
│  • Agent buffer > 80%            → P2                    │
│  • Query P99 > 30s               → P3                    │
│  • Export job failed              → P2                   │
└─────────────────────────────────────────────────────────┘

  ---
Cost Estimate (AWS, Monthly)

┌─────────────────────────────────────────────────────────┐
│  COMPONENT                  MONTHLY COST                 │
│                                                          │
│  Kafka (20 × i3en.2xlarge)                               │
│    20 × $1.488/hr × 730hr              ~$21,700          │
│                                                          │
│  Chunk Writers (100 × c6g.xlarge)                        │
│    100 × $0.136/hr × 730hr             ~$9,900           │
│                                                          │
│  Query Workers (50 × c6g.2xlarge, spot)                  │
│    50 × $0.08/hr × 730hr               ~$2,900           │
│                                                          │
│  Tail Service (20 × c6g.large)                           │
│    20 × $0.068/hr × 730hr              ~$1,000           │
│                                                          │
│  S3 Storage (48hr hot ≈ 160TB)                           │
│    160,000 GB × $0.023/GB              ~$3,700           │
│                                                          │
│  S3 Requests                                             │
│    PUTs: ~4M/day × 30 × $0.005/1000   ~$600             │
│    GETs: ~20M/day × 30 × $0.0004/1000 ~$240             │
│                                                          │
│  Data Transfer (within region)          ~$0              │
│                                                          │
│  TOTAL                                  ~$40,000/month   │
│                                                          │
│  Per GB ingested: ~$0.0055/GB                            │
│  (Datadog at this volume: ~$300K+/month)                 │
└─────────────────────────────────────────────────────────┘

  ---
Summary: How the Three Flows Map to Components

                                      ┌─────────────────┐
                                ┌────▶│ FLOW 1: Tail    │  Kafka → WS fan-out
                                │     │ Real-time view  │  Latency: <1s
                                │     └─────────────────┘
                                │
    Services → Agents → Kafka ──┤     ┌─────────────────┐
                                │     │ FLOW 2: Query   │  Kafka (live) + S3 (historical)
                                ├────▶│ Regex search    │  scatter-gather scan
                                │     │                 │  Latency: 2-30s
                                │     └─────────────────┘
                                │
                                │     ┌─────────────────┐
                                └────▶│ FLOW 3: Export  │  S3 → tar builder → output S3
                                      │ Analytics tar   │  Latency: 10-30 min
                                      └─────────────────┘

Shared infra:
• Kafka:        transit buffer (6hr), decouples all three flows
• Chunk Writer: Kafka → S3, serves Flow 2 (scan) and Flow 3 (export)
• S3:           source of truth for anything beyond real-time

