Ingestion Backpressure Handling                                                                                                                                                                               
Your Kafka-based real-time ingestion is falling behind during peak hours (500K events/sec). Design a solution considering:

Tuning taskCount, maxRowsPerSegment, maxTotalRows                                                                                                                                                                
When to scale horizontally vs vertically                                                                                                                                                                         
Handling poison pills and malformed data                                                                                                                                                                         
Monitoring and alerting strategy

⏺ Druid Kafka Ingestion Backpressure — Complete Solution

  ---                                                                                                                                                                                                            
Understanding Why You're Falling Behind

Before tuning, diagnose the exact bottleneck:

**500K events/sec** pipeline:

Kafka Partition → Druid Peon (parsing) → In-memory Index → Segment Push → Deep Storage                                                                                                                         
↑                   ↑                      ↑                ↑                                                                                                                                             
Lag here?         CPU bound?             Memory pressure?  I/O bound?

Each bottleneck has a different fix — wrong diagnosis = wrong cure.

# Diagnose in 60 seconds

# 1. Kafka consumer lag per partition
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \                                                                                                                                                       
--describe --group druid-supervisor-events_raw

# 2. Druid task throughput (events/sec being parsed)
curl http://overlord:8090/druid/indexer/v1/runningTasks | jq \                                                                                                                                                 
'.[] | {id, type, status, location}'

# 3. MiddleManager resource saturation
curl http://middlemanager:8091/druid/worker/v1/enabled

# 4. Active task count vs worker capacity
curl http://overlord:8090/druid/indexer/v1/workers | jq \                                                                                                                                                      
'.[] | {host, capacity, availableCapacity}'

# 5. JVM GC pressure on peons (leading cause of lag spikes)
curl http://middlemanager:8091/status/health
# Check GC pause time in logs: grep "GC overhead" /var/log/druid/peon.log
                                                                                                                                                                                                                 
---                                                                                                                                                                                                            
1. Tuning taskCount, maxRowsPerSegment, maxTotalRows

The Relationship Between These Parameters

taskCount          = parallelism (how many Kafka partitions you consume simultaneously)
maxRowsInMemory    = flush trigger (rows in memory before flushing to disk)                                                                                                                                    
maxRowsPerSegment  = segment size target (rows before pushing a segment)                                                                                                                                       
maxTotalRows       = total rows across ALL in-memory indexes before rejecting new data                                                                                                                         
↑ THIS is your backpressure valve

If maxTotalRows is hit → task pauses ingestion → Kafka lag grows

Baseline Calculation for 500K events/sec

# Sizing worksheet
events_per_sec     = 500_000                                                                                                                                                                                   
avg_row_bytes      = 500          # bytes per raw event                                                                                                                                                        
rollup_ratio       = 0.3          # assume 70% reduction after rollup                                                                                                                                          
mm_heap_gb         = 32           # MiddleManager JVM heap                                                                                                                                                     
task_heap_gb       = 4            # per peon JVM heap                                                                                                                                                          
mm_count           = 8            # MiddleManager nodes

# How many tasks can run simultaneously?
tasks_per_mm       = mm_heap_gb // task_heap_gb        # = 8 tasks/node                                                                                                                                        
total_tasks        = mm_count * tasks_per_mm            # = 64 tasks max

# How much data per task per second?
kafka_partitions   = 64                                                                                                                                                                                        
events_per_task    = events_per_sec / kafka_partitions  # = 7,812 events/sec/task

# Memory per task for in-flight rows
rows_in_memory_target = 1_500_000                       # 1.5M rows                                                                                                                                            
memory_for_rows    = rows_in_memory_target * avg_row_bytes * rollup_ratio
# = 1.5M * 500 * 0.3 = 225MB per task → fits in 4GB heap with headroom

# Segment push frequency
rows_per_segment   = 5_000_000                                                                                                                                                                                 
push_interval_sec  = rows_per_segment / events_per_task # = 640s ≈ 10 min
# Good: segments pushed every ~10 min means low push overhead

Supervisor Spec — Fully Tuned

{                                                                                                                                                                                                              
"type": "kafka",
    "dataSchema": {                                                                                                                                                                                              
    "dataSource": "events_raw",
    "timestampSpec": { "column": "event_time", "format": "iso" },                                                                                                                                              
    "dimensionsSpec": {                                                                                                                                                                                        
    "dimensions": ["tenant_id", "user_id", "event_type", "country"],                                                                                                                                         
    "dimensionExclusions": ["__time", "raw_payload"]                                                                                                                                                         
},                                                                                                                                                                                                         
    "metricsSpec": [                                                                                                                                                                                           
        { "type": "count",        "name": "event_count" },                                                                                                                                                       
        { "type": "doubleSum",    "name": "revenue",       "fieldName": "revenue" },                                                                                                                             
        { "type": "HLLSketchMerge", "name": "unique_users","fieldName": "user_id_hll" }                                                                                                                          
    ],                                                                                                                                                                                                         
    "granularitySpec": {                                                                                                                                                                                       
        "type": "uniform",                                                                                                                                                                                       
        "segmentGranularity": "HOUR",                                                                                                                                                                            
        "queryGranularity": "MINUTE",                                                                                                                                                                            
        "rollup": true                                                                                                                                                                                           
    }                                                                                                                                                                                                          
},                                                                                                                                                                                                           
"tuningConfig": {                                                                                                                                                                                            
"type": "kafka",

      "maxRowsInMemory": 1500000,                                                                                                                                                                                
      // ↑ Flush to disk when 1.5M rows in memory (per index sink)                                                                                                                                               
      // Too low → too many disk flushes → I/O bound                                                                                                                                                             
      // Too high → GC pressure → latency spikes                                                                                                                                                                 
                                                                                                                                                                                                                 
      "maxRowsPerSegment": 5000000,                                                                                                                                                                              
      // ↑ Target segment size — 5M rows ≈ 300-500MB compressed                                                                                                                                                  
      // Larger = better query performance (fewer segments to open)                                                                                                                                              
      // Smaller = faster push, less memory per task                                                                                                                                                             
                                                                                                                                                                                                                 
      "maxTotalRows": 20000000,                                                                                                                                                                                  
      // ↑ CRITICAL: total rows across all sinks before task pauses                                                                                                                                              
      // = maxRowsPerSegment * number_of_active_sinks                                                                                                                                                            
      // With HOUR granularity, you have at most 2 active hours → 2 sinks                                                                                                                                        
      // 2 * 5M * 2x buffer = 20M rows                                                                                                                                                                           
      // If hit → pause ingestion → Kafka lag grows intentionally                                                                                                                                                
                                                                                                                                                                                                                 
      "intermediaryPersistPeriod": "PT3M",                                                                                                                                                                       
      // ↑ Flush in-memory index to disk every 3 min regardless of row count                                                                                                                                     
      // Prevents memory buildup during low-volume periods                                                                                                                                                       
                                                                                                                                                                                                                 
      "maxPendingPersists": 3,                                                                                                                                                                                   
      // ↑ Max concurrent persist operations (memory → disk)                                                                                                                                                     
      // Higher = more parallelism but more memory used simultaneously                                                                                                                                           
                                                                                                                                                                                                                 
      "pushTimeout": 0,                                                                                                                                                                                          
      // ↑ 0 = wait forever for segment push (don't fail tasks on slow deep storage)                                                                                                                             
                                                                                                                                                                                                                 
      "handoffConditionTimeout": 0,                                                                                                                                                                              
      // ↑ 0 = wait indefinitely for coordinator to acknowledge handoff                                                                                                                                          
                                                                                                                                                                                                                 
      "indexSpec": {                                                                                                                                                                                             
        "bitmap": { "type": "roaring", "compressRunOnSerialization": true },                                                                                                                                     
        "dimensionCompression": "lz4",                                                                                                                                                                           
        "metricCompression": "lz4",                                                                                                                                                                              
        "longEncoding": "auto"                                                                                                                                                                                   
        // lz4 over zstd at ingest: faster compression, slightly larger files                                                                                                                                    
        // Switch to zstd for cold segments via compaction                                                                                                                                                       
      },                                                                                                                                                                                                         
                                                                                                                                                                                                                 
      "partitionsSpec": {                                                                                                                                                                                        
        "type": "dynamic",                                                                                                                                                                                       
        "maxRowsPerSegment": 5000000,                                                                                                                                                                            
        "maxTotalRows": 20000000                                                                                                                                                                                 
      }                                                                                                                                                                                                          
    },                                                                                                                                                                                                           
    "ioConfig": {                                                                                                                                                                                                
      "type": "kafka",                                                                                                                                                                                           
      "consumerProperties": {                                                                                                                                                                                    
        "bootstrap.servers": "kafka-1:9092,kafka-2:9092,kafka-3:9092",                                                                                                                                           
        "fetch.min.bytes": "1048576",                                                                                                                                                                            
        // ↑ 1MB: wait for 1MB of data before fetching (reduces round trips)                                                                                                                                     
        "fetch.max.wait.ms": "500",                                                                                                                                                                              
        "max.partition.fetch.bytes": "10485760",                                                                                                                                                                 
        // ↑ 10MB max per partition per fetch                                                                                                                                                                    
        "max.poll.records": "10000"                                                                                                                                                                              
        // ↑ Process 10K records per poll loop                                                                                                                                                                   
      },                                                                                                                                                                                                         
      "topic": "events_raw",                                                                                                                                                                                     
                                                                                                                                                                                                                 
      "taskCount": 16,                                                                                                                                                                                           
      // ↑ Number of parallel indexing tasks                                                                                                                                                                     
      // Rule: taskCount <= kafka_partitions / 2 (leave headroom for reassignment)                                                                                                                               
      // Rule: taskCount <= total_mm_capacity / 2 (leave room for handoff tasks)                                                                                                                                 
      // Start at 16, scale to 32 if lag persists                                                                                                                                                                
                                                                                                                                                                                                                 
      "replicas": 2,                                                                                                                                                                                             
      // ↑ Each task has a replica = fault tolerance                                                                                                                                                             
      // replicas * taskCount tasks running simultaneously                                                                                                                                                       
                                                                                                                                                                                                                 
      "taskDuration": "PT30M",                                                                                                                                                                                   
      // ↑ Tasks live 30 min then gracefully hand off                                                                                                                                                            
      // Shorter = more frequent handoffs = fresher segments but more overhead                                                                                                                                   
      // Longer = fewer handoffs but longer recovery if task dies                                                                                                                                                
                                                                                                                                                                                                                 
      "completionTimeout": "PT30M",                                                                                                                                                                              
                                                                                                                                                                                                                 
      "lateMessageRejectionPeriod": "PT1H",                                                                                                                                                                      
      // ↑ Reject events >1 hour late — prevents old data from holding segments open
      // Without this: one late event forces a segment to stay "active" forever                                                                                                                                  
                                                                                                                                                                                                                 
      "earlyMessageRejectionPeriod": "PT1H",                                                                                                                                                                     
      // ↑ Reject events >1 hour in the future — clock skew protection                                                                                                                                           
                                                                                                                                                                                                                 
      "useEarliestOffset": false                                                                                                                                                                                 
    }                                                                                                                                                                                                            
}

Dynamic taskCount Scaling via API

# Scale up during peak (don't restart supervisor — just update)
curl -X POST http://overlord:8090/druid/indexer/v1/supervisor/events_raw \                                                                                                                                     
-H 'Content-Type: application/json' \                                                                                                                                                                        
-d '{                                                                                                                                                                                                        
...same spec...                                                                                                                                                                                            
"taskCount": 32    ← doubled                                                                                                                                                                               
}'

# Supervisor will gracefully drain current tasks and spawn new ones
# No data loss, no restart required
                                                                                                                                                                                                                 
---             
2. Horizontal vs Vertical Scaling

Decision Framework

Symptom                          Root Cause              Fix
─────────────────────────────────────────────────────────────────────                                                                                                                                          
Kafka lag growing steadily   →   Not enough tasks    →   Scale H: +taskCount                                                                                                                                   
Kafka lag spiky, recovers    →   GC pauses           →   Scale V: +heap per task                                                                                                                               
Tasks OOMing                 →   Heap too small      →   Scale V: bigger MM nodes                                                                                                                              
Persist queue backing up     →   Disk I/O bound      →   Scale V: faster NVMe                                                                                                                                  
Segment push slow            →   Deep storage I/O    →   Scale H: +MM nodes for pushes                                                                                                                         
All MMs at 100% CPU          →   Parse bound         →   Scale H: +MM nodes                                                                                                                                    
Coordinator overwhelmed      →   Too many segments   →   Fix: increase maxRowsPerSegment

Horizontal Scaling — Add MiddleManager Nodes

When: taskCount is maxed out (all MM slots occupied), CPU utilization > 70% across all MMs, disk I/O saturated on existing MMs.

# k8s HPA for MiddleManager based on Kafka lag
apiVersion: autoscaling/v2                                                                                                                                                                                     
kind: HorizontalPodAutoscaler                                                                                                                                                                                  
metadata:                                                                                                                                                                                                      
name: druid-middlemanager-hpa                                                                                                                                                                                
spec:                                                                                                                                                                                                          
scaleTargetRef:                                                                                                                                                                                              
apiVersion: apps/v1                                                                                                                                                                                        
kind: Deployment                                                                                                                                                                                           
name: druid-middlemanager                                                                                                                                                                                  
minReplicas: 4                                                                                                                                                                                               
maxReplicas: 16                                                                                                                                                                                              
metrics:                                                                                                                                                                                                     
- type: External                                                                                                                                                                                             
external:                                                                                                                                                                                                  
metric:   
name: kafka_consumer_group_lag_sum                                                                                                                                                                     
selector:                                                                                                                                                                                              
matchLabels:                                                                                                                                                                                         
consumer_group: druid-supervisor-events_raw                                                                                                                                                        
target:                                                                                                                                                                                                  
type: Value                                                                                                                                                                                            
value: "5000000"      # scale up if lag > 5M messages                                                                                                                                                  
behavior:                                                                                                                                                                                                    
scaleUp:                                                                                                                                                                                                   
stabilizationWindowSeconds: 120   # wait 2 min before scaling up                                                                                                                                         
policies:                                                                                                                                                                                                
- type: Pods                                                                                                                                                                                             
value: 2                        # add 2 MMs at a time                                                                                                                                                  
periodSeconds: 300              # at most every 5 min                                                                                                                                                  
scaleDown:                                                                                                                                                                                                 
stabilizationWindowSeconds: 600   # wait 10 min before scaling down

Vertical Scaling — Tune JVM Per MiddleManager

When: GC pause times > 200ms, tasks failing with OOM, maxTotalRows hit frequently despite enough tasks.

# MiddleManager JVM — the peon launcher
DRUID_JVM_ARGS="-server                                                                                                                                                                                        
-Xms6g -Xmx6g                                                                                                                                                                                                
-XX:+UseG1GC                                                                                                                                                                                                 
-XX:MaxGCPauseMillis=100                                                                                                                                                                                     
-XX:G1HeapRegionSize=32m                                                                                                                                                                                     
-XX:InitiatingHeapOccupancyPercent=35                                                                                                                                                                        
-XX:G1ReservePercent=15                                                                                                                                                                                      
-XX:+ParallelRefProcEnabled                                                                                                                                                                                  
-XX:+DisableExplicitGC                                                                                                                                                                                       
-Duser.timezone=UTC"

# Peon JVM (each task process) — in supervisor tuningConfig
"indexerJavaOpts": [                                                                                                                                                                                           
"-server",                                                                                                                                                                                                   
"-Xms3g",                                                                                                                                                                                                    
"-Xmx3g",                                                                                                                                                                                                    
"-XX:+UseG1GC",                                                                                                                                                                                              
"-XX:MaxGCPauseMillis=200",                                                                                                                                                                                  
"-XX:G1HeapRegionSize=16m",                                                                                                                                                                                  
"-XX:InitiatingHeapOccupancyPercent=40",                                                                                                                                                                     
"-XX:+PrintGCDetails",                                                                                                                                                                                       
"-Xloggc:/var/log/druid/peon-gc.log"                                                                                                                                                                         
]

Scaling Decision Matrix

Current: 500K events/sec falling behind

Step 1: Check task utilization                                                                                                                                                                                 
→ taskCount=16, all 16 tasks at >80% CPU                                                                                                                                                                     
→ Action: increase taskCount to 32 (H scale within existing MMs)                                                                                                                                             
→ Cost: $0 (use existing capacity)

Step 2: MMs out of capacity                                                                                                                                                                                    
→ 8 MMs × 8 slots = 64 capacity, 32 tasks × 2 replicas = 64 used                                                                                                                                             
→ Action: add 4 more MM nodes (H scale)                                                                                                                                                                      
→ Cost: 4 × instance cost

Step 3: Individual tasks GC'ing heavily                                                                                                                                                                        
→ peon heap 2GB → OOM or long GC pauses                                                                                                                                                                      
→ Action: increase peon heap to 4GB (V scale, may need bigger MM instances)                                                                                                                                  
→ OR: reduce maxRowsInMemory to 750K (less memory per task, more flushes)

Step 4: Disk I/O on MMs saturated during persist                                                                                                                                                               
→ iostat shows 100% disk utilization                                                                                                                                                                         
→ Action: NVMe drives on MM nodes (V scale) OR                                                                                                                                                               
spread tasks across more MMs (H scale)
                                                                                                                                                                                                                 
---                                                                                                                                                                                                            
3. Poison Pills and Malformed Data

Failure Taxonomy

Malformed data types:
1. Schema mismatch       → wrong type for column (string where int expected)                                                                                                                                 
2. Null timestamp        → event has no __time → Druid rejects entire batch                                                                                                                                  
3. Oversized events      → 100MB JSON payload → OOM in parse phase                                                                                                                                           
4. Invalid UTF-8         → parser exception → task hangs                                                                                                                                                     
5. Future timestamps     → events 10 years ahead → pollutes recent segments                                                                                                                                  
6. Poison pill           → structured but semantically wrong (revenue = -999999)

Defense Layer 1 — Kafka Streams Pre-Processor

// Kafka Streams app sitting between raw topic and Druid input topic                                                                                                                                           
public class IngestionPreProcessor {

      private static final int MAX_EVENT_BYTES = 1_048_576; // 1MB                                                                                                                                               
      private static final long MAX_FUTURE_MS  = 3_600_000; // 1 hour ahead                                                                                                                                      
      private static final long MAX_PAST_MS    = 86_400_000 * 7; // 7 days behind                                                                                                                                
                                                                                                                                                                                                                 
      public ProcessorResult validate(ConsumerRecord<String, byte[]> record) {                                                                                                                                   
          // 1. Size guard                                                                                                                                                                                       
          if (record.value().length > MAX_EVENT_BYTES) {                                                                                                                                                         
              return reject(record, "OVERSIZED", record.value().length + " bytes");                                                                                                                              
          }                                                                                                                                                                                                      
                                                                                                                                                                                                                 
          JsonNode event;                                                                                                                                                                                        
          try {                                                                                                                                                                                                  
              event = mapper.readTree(record.value());                                                                                                                                                           
          } catch (JsonParseException e) {                                                                                                                                                                       
              return reject(record, "INVALID_JSON", e.getMessage());                                                                                                                                             
          }                                                                                                                                                                                                      
                                                                                                                                                                                                                 
          // 2. Timestamp validation                                                                                                                                                                             
          JsonNode ts = event.get("event_time");
          if (ts == null || ts.isNull()) {                                                                                                                                                                       
              return reject(record, "NULL_TIMESTAMP", "missing event_time");                                                                                                                                     
          }                                                                                                                                                                                                      
                                                                                                                                                                                                                 
          long eventMs = parseTimestamp(ts.asText());                                                                                                                                                            
          long now     = System.currentTimeMillis();                                                                                                                                                             
          if (eventMs > now + MAX_FUTURE_MS) {                                                                                                                                                                   
              return reject(record, "FUTURE_TIMESTAMP", "ts=" + eventMs);                                                                                                                                        
          }                                                                                                                                                                                                      
          if (eventMs < now - MAX_PAST_MS) {                                                                                                                                                                     
              return reject(record, "STALE_EVENT", "ts=" + eventMs);                                                                                                                                             
          }                                                                                                                                                                                                      
                                                                                                                                                                                                                 
          // 3. Required fields                                                                                                                                                                                  
          for (String required : List.of("tenant_id", "user_id", "event_type")) {
              if (!event.has(required) || event.get(required).isNull()) {                                                                                                                                        
                  return reject(record, "MISSING_FIELD", "field=" + required);                                                                                                                                   
              }                                                                                                                                                                                                  
          }                                                                                                                                                                                                      
                                                                                                                                                                                                                 
          // 4. Semantic validation (poison pill detection)                                                                                                                                                      
          if (event.has("revenue")) {
              double revenue = event.get("revenue").asDouble();                                                                                                                                                  
              if (revenue < -1_000_000 || revenue > 1_000_000_000) {                                                                                                                                             
                  return reject(record, "SUSPICIOUS_VALUE",                                                                                                                                                      
                      "revenue=" + revenue + " out of bounds");                                                                                                                                                  
              }                                                                                                                                                                                                  
          }                                                                                                                                                                                                      
                                                                                                                                                                                                                 
          return accept(record);
      }                                                                                                                                                                                                          
                                                                                                                                                                                                                 
      private ProcessorResult reject(ConsumerRecord<String, byte[]> r,                                                                                                                                           
                                     String reason, String detail) {                                                                                                                                             
          // Route to DLQ with metadata                                                                                                                                                                          
          producer.send(new ProducerRecord<>("events_dlq",                                                                                                                                                       
              null,                                                                                                                                                                                              
              Json.object()                                                                                                                                                                                      
                  .put("original_topic",     r.topic())                                                                                                                                                          
                  .put("original_partition", r.partition())                                                                                                                                                      
                  .put("original_offset",    r.offset())                                                                                                                                                         
                  .put("rejection_reason",   reason)                                                                                                                                                             
                  .put("rejection_detail",   detail)                                                                                                                                                             
                  .put("rejected_at",        Instant.now().toString())                                                                                                                                           
                  .put("payload",            new String(r.value()))                                                                                                                                              
                  .toString()                                                                                                                                                                                    
          ));                                                                                                                                                                                                    
                                                                                                                                                                                                                 
          metrics.increment("events.rejected", "reason", reason);                                                                                                                                                
          return ProcessorResult.REJECTED;                                                                                                                                                                       
      }                                                                                                                                                                                                          
}

Defense Layer 2 — Druid Transform + Filter Spec

// In supervisor dataSchema — second line of defense                                                                                                                                                           
"transformSpec": {                                                                                                                                                                                             
"transforms": [                                                                                                                                                                                              
{                                                                                                                                                                                                          
"type": "expression",                                                                                                                                                                                    
"name": "revenue_safe",                                                                                                                                                                                  
"expression": "case_searched(revenue < -1000000 || revenue > 1e9, null, revenue)"                                                                                                                        
// Null out bad revenue rather than rejecting the whole row                                                                                                                                              
},                                                                                                                                                                                                         
{                                                                                                                                                                                                          
"type": "expression",                                                                                                                                                                                    
"name": "event_type_clean",                                                                                                                                                                              
"expression": "lower(trim(event_type))"                                                                                                                                                                  
// Normalize before ingestion                                                                                                                                                                            
}                                                                                                                                                                                                          
],                                                                                                                                                                                                           
"filter": {                                                                                                                                                                                                  
"type": "and",                                                                                                                                                                                             
"fields": [                                                                                                                                                                                                
{ "type": "not",      "field": { "type": "null", "column": "tenant_id" } },                                                                                                                              
{ "type": "not",      "field": { "type": "null", "column": "event_type" } },                                                                                                                             
{ "type": "interval", "dimension": "__time",                                                                                                                                                             
"intervals": ["2020-01-01/2028-01-01"] }                                                                                                                                                               
// Hard drop rows outside valid time range at ingest                                                                                                                                                     
]                                                                                                                                                                                                          
}                                                                                                                                                                                                            
}

Defense Layer 3 — Task Auto-Recovery

# Supervisor watchdog — runs every 60 seconds
class SupervisorWatchdog:

      STUCK_TASK_THRESHOLD_MIN = 10  # task producing 0 rows for 10 min = stuck                                                                                                                                  
      MAX_CONSECUTIVE_FAILURES = 5                                                                                                                                                                               
                                                                                                                                                                                                                 
      def check(self, supervisor_id: str):                                                                                                                                                                       
          status = druid.get_supervisor_status(supervisor_id)                                                                                                                                                    
                                                                                                                                                                                                                 
          for task in status['activeTasks']:                                                                                                                                                                     
              task_id = task['id']                                                                                                                                                                               
              rows_processed = self.get_rows_delta(task_id, window_minutes=10)                                                                                                                                   
                                                                                                                                                                                                                 
              # Detect stuck task (parsing loop on poison pill)                                                                                                                                                  
              if rows_processed == 0:                                                                                                                                                                            
                  log.warning(f"Task {task_id} processed 0 rows in 10 min — killing")                                                                                                                            
                  druid.shutdown_task(task_id)                                                                                                                                                                   
                  metrics.increment('tasks.killed.stuck')                                                                                                                                                        
                  continue                                                                                                                                                                                       
                                                                                                                                                                                                                 
              # Detect excessive failure rate                                                                                                                                                                    
              error_rate = self.get_parse_error_rate(task_id)
              if error_rate > 0.01:  # >1% parse errors                                                                                                                                                          
                  log.error(f"Task {task_id} parse error rate {error_rate:.2%}")                                                                                                                                 
                  alert.fire('HIGH_PARSE_ERROR_RATE', task_id=task_id,                                                                                                                                           
                             rate=error_rate)                                                                                                                                                                    
                                                                                                                                                                                                                 
          # Check consecutive supervisor failures                                                                                                                                                                
          failures = status.get('recentErrors', [])                                                                                                                                                              
          if len(failures) >= self.MAX_CONSECUTIVE_FAILURES:                                                                                                                                                     
              alert.fire('SUPERVISOR_REPEATED_FAILURES',                                                                                                                                                         
                         supervisor=supervisor_id,                                                                                                                                                               
                         errors=failures[-3:])                                                                                                                                                                   
                                                                                                                                                                                                                 
      def get_rows_delta(self, task_id: str, window_minutes: int) -> int:                                                                                                                                        
          now   = self.metrics_store.get_rows_ingested(task_id, at=datetime.now())                                                                                                                               
          past  = self.metrics_store.get_rows_ingested(task_id,                                                                                                                                                  
                      at=datetime.now() - timedelta(minutes=window_minutes))                                                                                                                                     
          return now - past                                                                                                                                                                                      

Dead Letter Queue Processing

// Async DLQ processor — retry eligible events, alert on patterns                                                                                                                                              
type DLQProcessor struct {                                                                                                                                                                                     
consumer    kafka.Consumer                                                                                                                                                                                 
producer    kafka.Producer                                                                                                                                                                                 
alerter     Alerter                                                                                                                                                                                        
store       DLQStore                                                                                                                                                                                       
}

func (p *DLQProcessor) Process(ctx context.Context) {                                                                                                                                                          
// Track rejection reasons in a rolling window                                                                                                                                                             
reasonCounts := make(map[string]*RollingCounter)

      for msg := range p.consumer.Messages() {                                                                                                                                                                   
          var dlqEvent DLQEvent                                                                                                                                                                                  
          json.Unmarshal(msg.Value, &dlqEvent)                                                                                                                                                                   
                                                                                                                                                                                                                 
          // Store for inspection                                                                                                                                                                                
          p.store.Save(dlqEvent)                                                                                                                                                                                 
                                                                                                                                                                                                                 
          // Count by reason                                                                                                                                                                                     
          rc := reasonCounts[dlqEvent.Reason]                                                                                                                                                                    
          rc.Increment()                                                                                                                                                                                         
                                                                                                                                                                                                                 
          // Alert on sudden spikes — indicates upstream bug or attack                                                                                                                                           
          if rc.Rate1Min() > 1000 {                                                                                                                                                                              
              p.alerter.Fire(Alert{                                                                                                                                                                              
                  Severity: "HIGH",                                                                                                                                                                              
                  Message:  fmt.Sprintf("DLQ spike: %s at %.0f/min", dlqEvent.Reason, rc.Rate1Min()),                                                                                                            
              })                                                                                                                                                                                                 
          }                                                                                                                                                                                                      
                                                                                                                                                                                                                 
          // Retry stale events that are now within window                                                                                                                                                       
          if dlqEvent.Reason == "STALE_EVENT" {
              age := time.Since(dlqEvent.RejectedAt)                                                                                                                                                             
              if age < 6*time.Hour {                                                                                                                                                                             
                  // Event arrived late but within Druid's lateMessageRejectionPeriod                                                                                                                            
                  p.producer.Send("events_retry", dlqEvent.Payload)                                                                                                                                              
              }                                                                                                                                                                                                  
          }                                                                                                                                                                                                      
      }                                                                                                                                                                                                          
}
                  
---                                                                                                                                                                                                            
4. Monitoring and Alerting Strategy

Metrics Collection Architecture

Druid JMX → Prometheus JMX Exporter → Prometheus → Grafana
Kafka JMX  → Prometheus JMX Exporter ↗                                                                                                                                                                         
Custom app metrics → Prometheus pushgateway ↗

Critical Metrics Hierarchy

Tier 1 — Page immediately (ingestion broken)                                                                                                                                                                   
kafka_consumer_group_lag_sum > 10M for 5 min                                                                                                                                                                 
druid_ingest_events_unparseable rate > 1%                                                                                                                                                                    
druid_supervisor_state != RUNNING for 2 min                                                                                                                                                                  
druid_task_failed_count increase > 3 in 10 min

Tier 2 — Alert, investigate within 30 min                                                                                                                                                                      
kafka_consumer_group_lag_sum > 2M for 10 min                                                                                                                                                                 
druid_ingest_persist_time_p99 > 30s                                                                                                                                                                          
druid_jvm_gc_pause_p99 > 500ms                                                                                                                                                                               
druid_middlemanager_available_capacity < 20%

Tier 3 — Slack notification, investigate during business hours                                                                                                                                                 
druid_ingest_rows_per_second < 80% of baseline for 15 min                                                                                                                                                    
druid_segment_handoff_delay > 10 min                                                                                                                                                                         
DLQ rejection rate increasing trend

Prometheus Alert Rules

groups:
- name: druid_ingestion_critical                                                                                                                                                                               
  rules:

    - alert: KafkaLagCritical                                                                                                                                                                                    
      expr: |                                                                                                                                                                                                    
      sum(kafka_consumer_group_lag{                                                                                                                                                                            
      consumer_group=~"druid-supervisor-.*"                                                                                                                                                                  
      }) by (consumer_group) > 10000000                                                                                                                                                                        
      for: 5m                                                                                                                                                                                                    
      labels:                                                                                                                                                                                                    
      severity: page                                                                                                                                                                                           
      annotations:                                                                                                                                                                                               
      summary: "Druid ingestion lag {{ $value | humanize }} events"                                                                                                                                            
      runbook: "https://wiki/druid-lag-runbook"

    - alert: DruidTaskFailureSpike                                                                                                                                                                               
      expr: |                                                                                                                                                                                                    
      increase(druid_task_failed_total[10m]) > 3                                                                                                                                                               
      labels:                                                                                                                                                                                                    
      severity: page                                                                                                                                                                                           
      annotations:                                                                                                                                                                                               
      summary: "{{ $value }} Druid tasks failed in 10 min"

    - alert: HighParseErrorRate                                                                                                                                                                                  
      expr: |                                                                                                                                                                                                    
      rate(druid_ingest_events_unparseable_total[5m]) /                                                                                                                                                        
      rate(druid_ingest_events_processed_total[5m]) > 0.01                                                                                                                                                     
      for: 3m                                                                                                                                                                                                    
      labels:                                                                                                                                                                                                    
      severity: page                                                                                                                                                                                           
      annotations:                                                                                                                                                                                               
      summary: "Parse error rate {{ $value | humanizePercentage }}"

    - alert: MiddleManagerCapacityLow                                                                                                                                                                            
      expr: |                                                                                                                                                                                                    
      sum(druid_worker_available_capacity) /                                                                                                                                                                   
      sum(druid_worker_capacity) < 0.20                                                                                                                                                                        
      for: 5m                                                                                                                                                                                                    
      labels:                                                                                                                                                                                                    
      severity: warning                                                                                                                                                                                        
      annotations:                                                                                                                                                                                               
      summary: "Only {{ $value | humanizePercentage }} MM capacity available"

    - alert: PeonGCPressure                                                                                                                                                                                      
      expr: |                                                                                                                                                                                                    
      druid_jvm_gc_collection_seconds{gc="G1 Young Generation", quantile="0.99"} > 0.5                                                                                                                         
      for: 5m                                                                                                                                                                                                    
      labels:                                                                                                                                                                                                    
      severity: warning                                                                                                                                                                                        
      annotations:                                                                                                                                                                                               
      summary: "Peon P99 GC pause {{ $value }}s — risking task timeout"

    - alert: SegmentHandoffDelayed                                                                                                                                                                               
      expr: |                                                                                                                                                                                                    
      time() - druid_segment_last_handoff_timestamp > 600                                                                                                                                                      
      labels:                                                                                                                                                                                                    
      severity: warning                                                                                                                                                                                        
      annotations:                                                                                                                                                                                               
      summary: "No segment handoff in {{ $value | humanizeDuration }}"

Grafana Dashboard — Key Panels

Row 1: Pipeline Health                                                                                                                                                                                         
┌─────────────────────┬─────────────────────┬─────────────────────┐                                                                                                                                            
│  Kafka Lag (total)  │ Events/sec ingested │  Parse error rate   │                                                                                                                                            
│  [time series]      │  [gauge + sparkline]│  [%age over time]   │                                                                                                                                            
└─────────────────────┴─────────────────────┴─────────────────────┘

Row 2: Task Health                                                                                                                                                                                             
┌─────────────────────┬─────────────────────┬─────────────────────┐                                                                                                                                            
│ Running tasks       │  MM capacity used   │  Task duration p99  │                                                                                                                                            
│  by supervisor      │  [stacked bar]      │  [histogram]        │                                                                                                                                            
└─────────────────────┴─────────────────────┴─────────────────────┘

Row 3: Resource Pressure                                                                                                                                                                                       
┌─────────────────────┬─────────────────────┬─────────────────────┐                                                                                                                                            
│  JVM heap per MM    │  GC pause p99       │  Disk I/O (persist) │                                                                                                                                            
│  [multi-line]       │  [heat map]         │  [area chart]       │                                                                                                                                            
└─────────────────────┴─────────────────────┴─────────────────────┘

Row 4: Data Quality                                                                                                                                                                                            
┌─────────────────────┬─────────────────────┬─────────────────────┐                                                                                                                                            
│  DLQ rejection rate │ Rejection by reason │  Late event rate    │                                                                                                                                            
│  [rate over time]   │  [pie chart]        │  [% of total]       │                                                                                                                                            
└─────────────────────┴─────────────────────┴─────────────────────┘

Lag-Based Auto-Tuning Script

# Runs every 5 minutes — auto-adjusts taskCount based on lag trend
class IngestionAutoTuner:

      MIN_TASKS = 8                                                                                                                                                                                              
      MAX_TASKS = 48                                                                                                                                                                                             
      LAG_SCALE_UP_THRESHOLD   = 2_000_000   # lag > 2M → scale up                                                                                                                                               
      LAG_SCALE_DOWN_THRESHOLD =   500_000   # lag < 500K for 15 min → scale down                                                                                                                                
      SCALE_UP_STEP   = 4                                                                                                                                                                                        
      SCALE_DOWN_STEP = 2                                                                                                                                                                                        
      COOLDOWN_MIN    = 10  # don't scale again within 10 min                                                                                                                                                    
                                                                                                                                                                                                                 
      def evaluate(self, supervisor_id: str):                                                                                                                                                                    
          lag     = kafka.get_consumer_lag(f"druid-supervisor-{supervisor_id}")                                                                                                                                  
          spec    = druid.get_supervisor_spec(supervisor_id)                                                                                                                                                     
          current = spec['ioConfig']['taskCount']                                                                                                                                                                
                                                                                                                                                                                                                 
          if self.in_cooldown(supervisor_id):                                                                                                                                                                    
              return                                                                                                                                                                                             
                                                                                                                                                                                                                 
          if lag > self.LAG_SCALE_UP_THRESHOLD:                                                                                                                                                                  
              new_count = min(current + self.SCALE_UP_STEP, self.MAX_TASKS)                                                                                                                                      
              if new_count != current:                                                                                                                                                                           
                  log.info(f"Scaling up {supervisor_id}: {current} → {new_count} tasks (lag={lag:,})")                                                                                                           
                  self.update_task_count(supervisor_id, spec, new_count)                                                                                                                                         
                  self.record_scale_event(supervisor_id, 'UP', current, new_count, lag)                                                                                                                          
                                                                                                                                                                                                                 
          elif lag < self.LAG_SCALE_DOWN_THRESHOLD:                                                                                                                                                              
              # Only scale down if consistently low (check 3 consecutive readings)                                                                                                                               
              if self.consistently_low_lag(supervisor_id, readings=3):                                                                                                                                           
                  new_count = max(current - self.SCALE_DOWN_STEP, self.MIN_TASKS)                                                                                                                                
                  if new_count != current:                                                                                                                                                                       
                      log.info(f"Scaling down {supervisor_id}: {current} → {new_count} tasks")                                                                                                                   
                      self.update_task_count(supervisor_id, spec, new_count)                                                                                                                                     
                                                                                                                                                                                                                 
      def update_task_count(self, supervisor_id, spec, new_count):                                                                                                                                               
          spec['ioConfig']['taskCount'] = new_count                                                                                                                                                              
          druid.update_supervisor(supervisor_id, spec)                                                                                                                                                           
          self.set_cooldown(supervisor_id)                                                                                                                                                                       
                                                                                                                                                                                                                 
---                                                                                                                                                                                                            
Runbook — Peak Hour Response

Alert: KafkaLagCritical fired (lag > 10M)

Step 1 (< 2 min): Verify it's real                                                                                                                                                                             
curl overlord:8090/druid/indexer/v1/supervisor/events_raw/status                                                                                                                                             
→ Check "activeTasks" count matches expected taskCount                                                                                                                                                       
→ Check "recentErrors" for parse failures

Step 2 (< 5 min): Quick wins                                                                                                                                                                                   
a. If tasks < expected: supervisor may have lost tasks                                                                                                                                                       
POST overlord:8090/druid/indexer/v1/supervisor/events_raw/resume                                                                                                                                          
b. If parse errors spiking: bad deployment pushed malformed events                                                                                                                                           
→ Roll back producer, let DLQ absorb bad events

Step 3 (< 10 min): Scale up tasks                                                                                                                                                                              
→ Update supervisor spec: taskCount = current * 2                                                                                                                                                            
→ Verify new tasks appear: watch overlord:8090/druid/indexer/v1/runningTasks

Step 4 (< 20 min): If lag still growing                                                                                                                                                                        
→ Add MiddleManager nodes (k8s: kubectl scale deploy druid-mm --replicas=12)                                                                                                                                 
→ After nodes register, increase taskCount again

Step 5 (< 30 min): If tasks OOMing                                                                                                                                                                             
→ Check peon GC logs: grep "OutOfMemory" /var/log/druid/peon*.log                                                                                                                                            
→ Reduce maxRowsInMemory: 1500000 → 750000 (halve memory per task)                                                                                                                                           
→ Increase maxTotalRows proportionally

Step 6 (post-incident)                                                                                                                                                                                         
→ Identify root cause: volume spike? schema change? upstream bug?                                                                                                                                            
→ Update auto-tuner thresholds                                                                                                                                                                               
→ Add DLQ patterns as new validation rules                                                                                                                                                                   
→ Document in incident log
                                                                                                                                                                                                                 
---                                                                                                                                                                                                            
Summary: Decision Tree

500K/sec falling behind?
│                                                                                                                                                                                                              
├─ Parse errors > 1%? ──────────────────────→ Fix upstream, add pre-processor                                                                                                                                  
│                                                                                                                                                                                                              
├─ GC pauses > 200ms? ──────────────────────→ Increase peon heap (vertical)                                                                                                                                    
│                                              Reduce maxRowsInMemory                                                                                                                                          
│                                                                                                                                                                                                              
├─ All MM slots full? ──────────────────────→ Add MM nodes (horizontal)                                                                                                                                        
│                                                                                                                                                                                                              
├─ Tasks < taskCount? ──────────────────────→ Increase taskCount (free, do first)                                                                                                                              
│                                                                                                                                                                                                              
├─ Persist queue backing up? ───────────────→ Faster disks (NVMe) on MMs                                                                                                                                       
│                                                                                                                                                                                                              
├─ Handoff slow? ───────────────────────────→ Deep storage I/O issue
│                                              Check S3/HDFS throughput                                                                                                                                        
│                                                                                                                                                                                                              
└─ Everything fine but lag growing? ────────→ Kafka partition count < taskCount                                                                                                                                
Scale Kafka partitions first          