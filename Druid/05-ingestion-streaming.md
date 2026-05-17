# 5 · Streaming Ingestion — Kafka / Kinesis Supervisors

Streaming ingest with exactly-once semantics is Druid's biggest single competitive advantage over ClickHouse and most other OLAPs. The supervisor model is the why.

## 5.1 The supervisor pattern

A **supervisor** is a long-running coordination object (state stored in the metadata DB) that manages a fleet of **indexing tasks** for a streaming source. The supervisor is owned by the **Overlord**.

For each Kafka topic (or Kinesis stream), you submit one supervisor spec. The Overlord:
1. Submits N indexing tasks (one per Kafka partition group).
2. Each task consumes a subset of partitions for a finite time interval (a "task duration", typically PT1H).
3. When the task duration elapses, the task closes — its segments publish to deep storage and are handed off to Historicals.
4. The supervisor immediately spawns the next task to resume from where the previous left off.
5. Periodic checkpoints ensure exactly-once even across task failures.

## 5.2 The Kafka supervisor in detail

A minimal Kafka supervisor spec:

```json
{
  "type": "kafka",
  "spec": {
    "dataSchema": {
      "dataSource": "events",
      "timestampSpec": { "column": "ts", "format": "auto" },
      "dimensionsSpec": { "useSchemaDiscovery": true, "includeAllDimensions": true },
      "metricsSpec": [
        { "type": "count", "name": "events" },
        { "type": "doubleSum", "name": "revenue", "fieldName": "revenue" },
        { "type": "HLLSketchBuild", "name": "distinct_users", "fieldName": "user_id" }
      ],
      "granularitySpec": {
        "type": "uniform",
        "segmentGranularity": "HOUR",
        "queryGranularity": "MINUTE",
        "rollup": true
      }
    },
    "ioConfig": {
      "topic": "events",
      "consumerProperties": {
        "bootstrap.servers": "kafka:9092"
      },
      "inputFormat": { "type": "json" },
      "taskCount": 4,
      "replicas": 2,
      "taskDuration": "PT1H",
      "useEarliestOffset": true
    },
    "tuningConfig": {
      "type": "kafka",
      "maxRowsInMemory": 1000000,
      "maxRowsPerSegment": 5000000,
      "maxTotalRows": 20000000,
      "intermediatePersistPeriod": "PT10M"
    }
  }
}
```

Notable knobs:
- **`taskCount`** — number of tasks per replica group. Each task gets a contiguous subset of Kafka partitions.
- **`replicas`** — each partition group runs `replicas` times in parallel. With `replicas=2`, you have 2× tasks; only one publishes per group (the others are "shadow" for fault tolerance).
- **`taskDuration`** — how long each task runs before rolling over. Shorter = more segments; longer = bigger segments.
- **`maxRowsInMemory`** — when to spill from in-memory IncrementalIndex to disk.
- **`intermediatePersistPeriod`** — how often to spill.

## 5.3 Exactly-once semantics

Druid stores Kafka consumer offsets in its **metadata DB**, not in Kafka. The flow:

1. Task consumes messages, builds segment in memory.
2. At end of `taskDuration` (or `maxRowsPerSegment` hit), task publishes segment to deep storage.
3. Within the **same transaction** in the metadata DB, the task writes both:
   - The new segment metadata.
   - The end Kafka offset for each partition.
4. Next task starts from those committed offsets.

If a task crashes mid-flight, no segment is published and the offsets aren't advanced. The new task re-consumes from the last committed offsets, producing **exactly-once** rather than at-least-once.

This is materially different from the Kafka-engine + MV approach in ClickHouse, which relies on Kafka's consumer offsets and ClickHouse block-dedup hash — at-least-once with dedup-based at-most-once.

## 5.4 Replicas and shadow tasks

`replicas: 2` doubles the cost but provides fault tolerance:
- Two parallel sets of tasks per partition group.
- They both consume the same offsets.
- Only one publishes (the "leader"); the other's segments are discarded.
- If the leader fails, the shadow becomes leader instantly; no missed data.

For high-stakes streams (billing), use `replicas: 2`. For non-critical (telemetry), `replicas: 1`.

## 5.5 Late data handling

Real streams have **late events**: a message timestamped `12:00` arrives at `12:30`. What Druid does depends on the rollup window:

- If the segment for `12:00` is still being ingested (the task hasn't closed yet), the late event joins normally.
- If the segment has already published, the late event goes into a **new** segment that overlaps the same time interval. Coordinator merges the two via compaction later (segments at the same interval are merged).

To prevent the late-event-causes-new-segment problem, set `lateMessageRejectionPeriod` or `earlyMessageRejectionPeriod` to drop late messages outside a window.

Or just accept that late data will appear in a separate small segment that gets compacted later. This is normal.

## 5.6 Schema discovery in streaming

With `useSchemaDiscovery: true`, Druid learns dimensions from each incoming message:

- A new field appears → Druid adds it as a dimension automatically.
- Existing fields' types are tracked; type mismatches are coerced or null'd.
- Newer Druid versions support `auto` type for fully dynamic typed dimensions.

Use case: an event-tracking platform where product engineers add new event fields without coordinating with the data team.

Pair with **JSON columns** for the highest flexibility:

```json
"dimensionsSpec": {
  "dimensions": [
    "country",
    "event_type",
    { "name": "props", "type": "json" }
  ]
}
```

`props` stores the entire JSON payload; subsequent queries can drill into paths.

## 5.7 The Kinesis supervisor

Identical pattern; reads from Kinesis instead of Kafka. Offset management uses Kinesis sequence numbers.

```json
{
  "type": "kinesis",
  "spec": {
    "ioConfig": {
      "stream": "events",
      "endpoint": "kinesis.us-east-1.amazonaws.com",
      "useEarliestSequenceNumber": true,
      "taskCount": 4,
      "replicas": 2,
      "taskDuration": "PT1H"
    },
    ...
  }
}
```

Same exactly-once via metadata-DB-tracked sequence numbers.

## 5.8 Real-time query semantics

While a task is still running, **its in-memory IncrementalIndex is queryable**. The Broker discovers ingestion tasks via Overlord and fans queries out to them.

Path:
- Historical → published segments (most data).
- MiddleManager Peon (or Indexer thread) → in-flight segments still being built.

This is why Druid can show data ingested 2 seconds ago in the same query that shows data ingested 2 years ago. The Broker stitches together.

## 5.9 Backpressure and ingest throughput

Ingest rate per partition group is bounded by:
- CPU (parsing, dictionary building, rollup, bitmap construction).
- Memory (`maxRowsInMemory` controls spill).
- Disk IO (persisting intermediate segments).

Scaling levers:
- **Increase `taskCount`** — more parallelism (limited by Kafka partition count).
- **Increase `replicas`** — only helps fault tolerance, not throughput.
- **Add MiddleManager / Indexer nodes** — more slots for tasks.
- **Tune `maxRowsInMemory`** — bigger = fewer persists, more memory.
- **Reduce schema discovery overhead** — explicit dims are faster than inferred.

Realistic per-task throughput: **100K-500K rows/sec** for a typical schema, more for narrow schemas with strong rollup.

## 5.10 Task lifecycle states

```
PENDING → RUNNING → FAILED
                  → SUCCESS
WAITING (waiting for slot)
```

Visible in `system.tasks` and the Overlord web console (`/druid/indexer/v1/tasks`).

Common diagnoses:
- Task PENDING for long time → no MiddleManager slot available (scale up).
- Task FAILED with OOM → reduce `maxRowsInMemory` or scale up Peon JVM heap.
- Task FAILED on publish → metadata DB or deep storage unavailable.
- Task running past `taskDuration` → still flushing; allow it.

## 5.11 Supervisor "auto-recovery"

If a supervisor detects task failure, it spawns a replacement. If many failures happen quickly, the supervisor enters a backoff state. You can:
- **Reset** the supervisor to clear bad state (`POST /druid/indexer/v1/supervisor/<id>/reset`).
- **Suspend** the supervisor to stop ingestion temporarily.
- **Terminate** to stop permanently.

## 5.12 The "Two Druid Tasks vs One Flink Job" question

A common architecture question: stream a Kafka topic through Flink → write to Druid via batch, or just supervise the Kafka topic in Druid.

- **Druid supervisor**: simpler, fewer moving parts, exactly-once, lower latency.
- **Flink → Druid**: useful when you need complex stream processing (joins, windowed agg, ML enrichment) before ingest. Flink writes via MSQ batch or via push.

Modern recommendation: use Druid supervisors directly for raw events; use Flink for complex pipelines. Don't add Flink if you don't need it.

## 5.13 The "Two Kafka Topics, One Datasource" pattern

If you have multiple Kafka topics that should write to the same Druid datasource (e.g., page_views, button_clicks), you can:
- Run two supervisors writing to the same datasource. Each has its own offsets, tasks.
- Or normalize upstream into one topic.

Druid allows multi-source ingest into one datasource — segments from different supervisors coexist.

## 5.14 Worked design — exactly-once at 1M events/sec

**Target**: 1M events/sec from Kafka, exactly-once into Druid, sub-second tail dashboard latency.

- Kafka cluster: 200 partitions of `events` topic.
- Druid supervisor: `taskCount = 100`, `replicas = 2`, `taskDuration = PT15M`.
- That's 200 simultaneous tasks (100 per replica).
- 5 MiddleManagers × 40 task slots each = 200 slots. Add 1-2 for headroom.
- Schema: explicit dimensions for performance.
- Rollup at minute granularity: realistic 10× rollup ratio → ~100K stored rows/sec post-rollup.
- Historicals receive ~300-700 MB segments every 15 minutes = ~96 segments/day. Healthy.

## 5.15 Must-internalize

- Supervisor = long-running orchestration object for streaming ingest.
- Exactly-once via metadata-DB-stored Kafka offsets, committed atomically with segments.
- `replicas` is for fault tolerance only (cost doubles, throughput doesn't).
- `taskCount` is for throughput (bounded by partition count).
- In-flight segments are queryable on MiddleManager Peons / Indexers.
- Late data goes into overlapping new segments; compaction merges later.
- Schema discovery + JSON columns for flexible schemas.
- Realistic per-task throughput: 100-500K rows/sec.

---

## Sources

- [Kafka ingestion supervisor — official](https://druid.apache.org/docs/latest/ingestion/kafka-ingestion/)
- [Kinesis ingestion supervisor — official](https://druid.apache.org/docs/latest/ingestion/kinesis-ingestion/)
- [Schema discovery (auto type)](https://druid.apache.org/docs/latest/ingestion/schema-design/)
- [Tuning config reference](https://druid.apache.org/docs/latest/ingestion/native-batch/)
- [Streaming ingestion overview](https://druid.apache.org/docs/latest/ingestion/streaming/)
