SaaS analytics platform serving 1000+ customers, each generating 10-100GB of event data daily. How would you architect a Druid-based solution to ensure tenant isolation, query performance, and cost          
efficiency? Consider:

Data partitioning strategies                                                                                                                                                                                     
Resource allocation and query prioritization                                                                                                                                                                     
Schema evolution handling                                                                                                                                                                                        
Cost attribution per tenant

**⏺ Druid-Based Multi-Tenant Analytics Architecture**

  ---                                                                                                                                                                                                            
System Overview

┌─────────────────────────────────────────────────────────────────────┐
│                         Data Ingestion Layer                        │                                                                                                                                        
│   Kafka (per-tenant topics)  →  Druid Supervisors  →  Deep Storage  │                                                                                                                                         
└─────────────────────────────────────────────────────────────────────┘                                                                                                                                        
                                    │                                                                                                                                                                          
┌─────────────────────────────────────────────────────────────────────┐                                                                                                                                        
│                         Druid Cluster                               │                                                                                                                                        
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐   │                                                                                                                                         
│  │   Overlord   │  │  Coordinator │  │        Broker Tier       │  │                                                                                                                                        
│  │ (ingest mgmt)│  │ (segment mgmt│  │  Tier-1  │  Tier-2       │  │                                                                                                                                         
│  └──────────────┘  └──────────────┘  │ (Premium)│  (Standard)   │  │                                                                                                                                         
│                                      └──────────────────────────┘  │                                                                                                                                         
│  ┌──────────────────────────────────────────────────────────────┐   │                                                                                                                                        
│  │                    Historical Tier                           │   │                                                                                                                                        
│  │  Hot (SSD NVMe)  │  Warm (SSD)  │  Cold (HDD / S3-backed)    │   │                                                                                                                                         
│  └──────────────────────────────────────────────────────────────┘   │                                                                                                                                        
│  ┌──────────────────────────────────────────────────────────────┐   │                                                                                                                                        
│  │               MiddleManager / Peon Pool                      │   │                                                                                                                                        
│  │  Premium Pool  │  Standard Pool  │  Bulk/Background Pool     │   │                                                                                                                                        
│  └──────────────────────────────────────────────────────────────┘   │                                                                                                                                        
└─────────────────────────────────────────────────────────────────────┘                                                                                                                                        
                                    │                                                                                                                                                                          
┌─────────────────────────────────────────────────────────────────────┐                                                                                                                                        
│                      Query Gateway (custom)                         │                                                                                                                                        
│   Auth → Tenant Resolution → Query Rewrite → Router → Audit Log     │                                                                                                                                         
└─────────────────────────────────────────────────────────────────────┘
                                                                                                                                                                                                                 
---                                                                                                                                                                                                            
1. Data Partitioning Strategy

Tenant Isolation Model — Tiered by Volume

Not all tenants are equal. Use a hybrid model based on data volume:

┌─────────────────────────────────────────────────────────┐                                                                                                                                                    
│  Tier A: Large tenants (>10GB/day)  → dedicated datasource                                                                                                                                                   
│  Tier B: Mid tenants (1-10GB/day)   → shared datasource, isolated segments                                                                                                                                   
│  Tier C: Small tenants (<1GB/day)   → fully shared datasource, tagged rows                                                                                                                                   
└─────────────────────────────────────────────────────────┘

Tier A — Dedicated Datasource Per Tenant

// Kafka supervisor spec per large tenant                                                                                                                                                                      
        {                                                                                                                                                                                                              
            "type": "kafka",                                                                                                                                                                                             
            "dataSchema": {                                                                                                                                                                                              
                "dataSource": "events_tenant_acme",        // dedicated datasource                                                                                                                                         
                "timestampSpec": { "column": "event_time", "format": "iso" },                                                                                                                                              
                "dimensionsSpec": {                                                                                                                                                                                        
                "dimensions": ["user_id", "event_type", "session_id", "country", "platform"]                                                                                                                             
            },                                                                                                                                                                                                         
            "metricsSpec": [                                                                                                                                                                                           
                { "type": "count",    "name": "event_count" },                                                                                                                                                           
                { "type": "doubleSum","name": "revenue",    "fieldName": "revenue" },                                                                                                                                    
                { "type": "HLLSketchMerge", "name": "unique_users", "fieldName": "user_id_hll" }                                                                                                                         
            ],                                                                                                                                                                                                         
            "granularitySpec": {                                                                                                                                                                                       
                "segmentGranularity": "HOUR",                                                                                                                                                                            
                "queryGranularity": "MINUTE",                                                                                                                                                                            
                "rollup": true                            // pre-aggregate at ingest                                                                                                                                     
            }                                                                                                                                                                                                          
            },                                                                                                                                                                                                           
            "tuningConfig": {                                                                                                                                                                                            
                "type": "kafka",                                                                                                                                                                                           
                "maxRowsPerSegment": 5000000,                                                                                                                                                                              
                "partitionsSpec": {                                                                                                                                                                                        
                    "type": "hashed",                                                                                                                                                                                        
                    "numShards": 8,                           // tune per tenant volume                                                                                                                                      
                    "partitionDimensions": ["user_id"]        // co-locate same user's data                                                                                                                                  
            }                                                                                                                                                                                                          
            },                                                                                                                                                                                                           
            "ioConfig": {                                                                                                                                                                                                
                "topic": "events.tenant.acme",                                                                                                                                                                             
                "taskCount": 4,                             // parallelism                                                                                                                                                 
                "replicas": 2                                                                                                                                                                                              
            }                                                                                                                                                                                                            
        }

Tier B/C — Shared Datasource with Tenant Dimension

    {                                                                                                                                                                                                              
        "dataSource": "events_shared_tier_b",                                                                                                                                                                        
        "dimensionsSpec": {                                                                                                                                                                                          
        "dimensions": [                                                                                                                                                                                            
        { "type": "string", "name": "tenant_id" },   // partition key                                                                                                                                            
        "user_id", "event_type", "country"                                                                                                                                                                       
        ]                                                                                                                                                                                                          
    },                                                                                                                                                                                                           
    "granularitySpec": {                                                                                                                                                                                         
        "segmentGranularity": "DAY",                                                                                                                                                                               
        "rollup": true                                                                                                                                                                                             
    },                                                                                                                                                                                                           
    "partitionsSpec": {                                                                                                                                                                                          
        "type": "single_dim",                                                                                                                                                                                      
        "partitionDimension": "tenant_id",             // segments aligned to tenants                                                                                                                              
        "targetRowsPerSegment": 3000000                                                                                                                                                                            
    }                                                                                                                                                                                                            
    }

Why single_dim partitioning on tenant_id: Druid's segment pruning ensures tenant queries only touch segments containing their data — query isolation without separate datasources.

Segment Tiering — Hot/Warm/Cold

// Coordinator dynamic config                                                                                                                                                                                  
{                                                                                                                                                                                                              
"millisToWaitBeforeDeleting": 900000,                                                                                                                                                                        
"mergeBytesLimit": 524288000,                                                                                                                                                                                
"mergeSegmentsLimit": 100,                                                                                                                                                                                   
"tiers": {                                                                                                                                                                                                   
"_default_tier": { "replicants": 2 },                                                                                                                                                                      
"hot":           { "replicants": 2 },                                                                                                                                                                      
"warm":          { "replicants": 1 },                                                                                                                                                                      
"cold":          { "replicants": 1 }                                                                                                                                                                       
}                                                                                                                                                                                                            
}

// Tiering rule per datasource — large tenant                                                                                                                                                                  
[                                                                                                                                                                                                              
{ "type": "loadByPeriod",  "period": "P3D",  "tieredReplicants": { "hot":  2 } },                                                                                                                            
{ "type": "loadByPeriod",  "period": "P30D", "tieredReplicants": { "warm": 1 } },                                                                                                                            
{ "type": "loadByPeriod",  "period": "P365D","tieredReplicants": { "cold": 1 } },                                                                                                                            
{ "type": "dropForever" }                                                                                                                                                                                    
]

Hot  tier: NVMe SSD, last 3 days   → sub-second queries                                                                                                                                                        
Warm tier: SSD,      last 30 days  → 1-5s queries                                                                                                                                                              
Cold tier: HDD/S3,   last 1 year   → 10-60s queries, acceptable for historical
                                                                                                                                                                                                                 
---                                                                                                                                                                                                            
2. Resource Allocation & Query Prioritization

Broker Tier — Tenant-Aware Routing

Deploy dedicated broker pools per service tier:

# docker-compose / k8s: broker-premium
druid.broker.http.numConnections: 50                                                                                                                                                                           
druid.processing.numThreads: 16                                                                                                                                                                                
druid.processing.numMergeBuffers: 8                                                                                                                                                                            
druid.processing.buffer.sizeBytes: 1073741824   # 1GB merge buffer                                                                                                                                             
druid.server.http.numThreads: 100

# broker-standard
druid.processing.numThreads: 8                                                                                                                                                                                 
druid.processing.numMergeBuffers: 4                                                                                                                                                                            
druid.processing.buffer.sizeBytes: 536870912    # 512MB                                                                                                                                                        
druid.server.http.numThreads: 50

Query Gateway — The Critical Layer

This is where tenant isolation actually happens at query time:

type QueryGateway struct {                                                                                                                                                                                     
    tenantRegistry TenantRegistry                                                                                                                                                                              
    quotaManager   QuotaManager                                                                                                                                                                                
    router         BrokerRouter                                                                                                                                                                                
    auditor        AuditLogger                                                                                                                                                                                 
}

type TenantConfig struct {                                                                                                                                                                                     
    TenantID      string                                                                                                                                                                                       
    Tier          ServiceTier  // Premium, Standard, Basic                                                                                                                                                     
    DatasourcePfx string       // "events_tenant_acme" or "events_shared_tier_b"                                                                                                                               
    QueryTimeout  time.Duration                                                                                                                                                                                
    MaxConcurrent int                                                                                                                                                                                          
    BytesPerDay   int64                                                                                                                                                                                        
    RowsPerQuery  int64                                                                                                                                                                                        
}

func (gw *QueryGateway) Execute(ctx context.Context, raw QueryRequest) (*QueryResult, error) {                                                                                                                 
tenant := gw.tenantRegistry.Resolve(ctx)

      // 1. Quota check                                                                                                                                                                                          
      if err := gw.quotaManager.CheckAndRecord(tenant, raw); err != nil {                                                                                                                                        
          return nil, ErrQuotaExceeded                                                                                                                                                                           
      }                                                                                                                                                                                                          
                                                                                                                                                                                                                 
      // 2. Query rewrite — inject tenant filter for shared datasources                                                                                                                                          
      rewritten := gw.rewriteQuery(raw, tenant)
                                                                                                                                                                                                                 
      // 3. Route to appropriate broker tier                                                                                                                                                                     
      broker := gw.router.SelectBroker(tenant.Tier)                                                                                                                                                              
                                                                                                                                                                                                                 
      // 4. Apply timeout                                                                                                                                                                                        
      ctx, cancel := context.WithTimeout(ctx, tenant.QueryTimeout)                                                                                                                                               
      defer cancel()                                                                                                                                                                                             
                                                                                                                                                                                                                 
      result, err := broker.Execute(ctx, rewritten)                                                                                                                                                              
                                                                                                                                                                                                                 
      // 5. Audit every query with cost metadata                                                                                                                                                                 
      gw.auditor.Log(AuditEntry{
          TenantID:     tenant.TenantID,                                                                                                                                                                         
          Query:        rewritten,                                                                                                                                                                               
          BytesScanned: result.BytesScanned,                                                                                                                                                                     
          RowsScanned:  result.RowsScanned,                                                                                                                                                                      
          DurationMS:   result.DurationMS,                                                                                                                                                                       
          Timestamp:    time.Now(),                                                                                                                                                                              
      })                                                                                                                                                                                                         
                                                                                                                                                                                                                 
      return result, err                                                                                                                                                                                         
}

// For shared datasources: inject tenant_id filter automatically                                                                                                                                               
// Tenants cannot query other tenants' data even if they try                                                                                                                                                   
func (gw *QueryGateway) rewriteQuery(q QueryRequest, t *TenantConfig) QueryRequest {                                                                                                                           
if !t.IsSharedDatasource() {                                                                                                                                                                               
return q  // dedicated datasource — no rewrite needed                                                                                                                                                  
}

      q.Filter = &AndFilter{                                                                                                                                                                                     
          Fields: []Filter{
              {Type: "selector", Dimension: "tenant_id", Value: t.TenantID},                                                                                                                                     
              q.Filter,  // preserve original filter                                                                                                                                                             
          },                                                                                                                                                                                                     
      }                                                                                                                                                                                                          
      return q                                                                                                                                                                                                   
}

Druid Query Priority Lanes

// runtime.properties per MiddleManager pool                                                                                                                                                                   
// Premium pool                                                                                                                                                                                                
druid.worker.category=premium                                                                                                                                                                                  
druid.worker.capacity=32

// Standard pool                                                                                                                                                                                               
druid.worker.category=standard                                                                                                                                                                                 
druid.worker.capacity=16

// Bulk pool (backfills, exports)                                                                                                                                                                              
druid.worker.category=bulk                                                                                                                                                                                     
druid.worker.capacity=8

// Task spec routes to correct pool by category                                                                                                                                                                
{                                                                                                                                                                                                              
"context": {                                                                                                                                                                                                 
"priority": 75,              // higher = more priority in broker merge threads                                                                                                                             
"workerCategory": "premium", // targets premium MiddleManager pool                                                                                                                                         
"queryId": "tenant_acme_q1",                                                                                                                                                                               
"timeout": 30000                                                                                                                                                                                           
}                                                                                                                                                                                                            
}

Concurrency Limiting Per Tenant — Token Bucket

type QuotaManager struct {                                                                                                                                                                                     
buckets sync.Map  // tenantID -> *rate.Limiter                                                                                                                                                             
redis   *redis.Client                                                                                                                                                                                      
}

func (qm *QuotaManager) CheckAndRecord(tenant *TenantConfig, q QueryRequest) error {                                                                                                                           
// 1. Concurrent query limit (in-memory)
limiter := qm.getOrCreate(tenant.TenantID, tenant.MaxConcurrent)                                                                                                                                           
if !limiter.Allow() {                                                                                                                                                                                      
return fmt.Errorf("concurrent query limit %d reached", tenant.MaxConcurrent)                                                                                                                           
}

      // 2. Daily bytes quota (distributed via Redis)                                                                                                                                                            
      key := fmt.Sprintf("quota:bytes:%s:%s", tenant.TenantID, time.Now().Format("2006-01-02"))
      used, err := qm.redis.IncrBy(ctx, key, estimateQueryCost(q)).Result()                                                                                                                                      
      if err == nil && used == estimateQueryCost(q) {                                                                                                                                                            
          qm.redis.Expire(ctx, key, 25*time.Hour)                                                                                                                                                                
      }                                                                                                                                                                                                          
                                                                                                                                                                                                                 
      if used > tenant.BytesPerDay {                                                                                                                                                                             
          return fmt.Errorf("daily bytes quota %d exceeded", tenant.BytesPerDay)
      }                                                                                                                                                                                                          
                  
      return nil                                                                                                                                                                                                 
}
                                                                                                                                                                                                                 
---                                                                                                                                                                                                            
3. Schema Evolution

Druid segments are immutable — schema changes require careful coordination.

Strategy Matrix

Change Type                   → Strategy                                                                                                                                                                       
──────────────────────────────────────────────────────────────                                                                                                                                                 
Add new dimension             → New ingestion spec (zero downtime)                                                                                                                                             
Add new metric                → New ingestion spec + backfill old segments                                                                                                                                     
Rename dimension              → Add new + deprecate old + reindex                                                                                                                                              
Change metric aggregation     → Reindex affected time ranges                                                                                                                                                   
Remove dimension              → Reindex (storage savings) or just stop ingesting                                                                                                                               
Data type change (str→long)   → Reindex mandatory — Druid stores typed columns

Schema Registry

type SchemaRegistry struct {                                                                                                                                                                                   
db *sql.DB                                                                                                                                                                                                 
}

type DatasourceSchema struct {                                                                                                                                                                                 
DatasourceID string
Version      int                                                                                                                                                                                           
Dimensions   []DimensionSpec                                                                                                                                                                               
Metrics      []MetricSpec                                                                                                                                                                                  
EffectiveFrom time.Time                                                                                                                                                                                    
EffectiveTo   *time.Time  // nil = current                                                                                                                                                                 
MigrationSQL  string      // how to derive new columns from old                                                                                                                                            
}

// On query: resolve which schema version applies to a time range                                                                                                                                              
func (sr *SchemaRegistry) SchemaForRange(ds string, start, end time.Time) []DatasourceSchema {
// Segments from different eras may have different schemas                                                                                                                                                 
// Broker fan-out handles this — each historical returns what it has                                                                                                                                       
// Missing columns return null, not errors                                                                                                                                                                 
}

Zero-Downtime Dimension Addition

-- Step 1: Update ingestion spec to include new dimension                                                                                                                                                      
-- New segments from this point have 'new_dim'                                                                                                                                                                 
-- Old segments simply return null for 'new_dim'

-- Step 2 (optional): Backfill old segments via kill + reindex task                                                                                                                                            
{                                                                                                                                                                                                              
"type": "index_parallel",                                                                                                                                                                                    
"spec": {                                                                                                                                                                                                    
"dataSchema": {                                                                                                                                                                                            
"dataSource": "events_tenant_acme",                                                                                                                                                                      
"dimensionsSpec": {                                                                                                                                                                                      
"dimensions": ["user_id", "event_type", "country", "new_dim"]  -- added                                                                                                                                
}                                                                                                                                                                                                        
},                                                                                                                                                                                                         
"ioConfig": {                                                                                                                                                                                              
"type": "index_parallel",                                                                                                                                                                                
"inputSource": {                                                                                                                                                                                         
"type": "druid",                                                                                                                                                                                       
"dataSource": "events_tenant_acme",                                                                                                                                                                    
"interval": "2026-01-01/2026-03-01"  -- backfill range                                                                                                                                                 
}                                                                                                                                                                                                        
}                                                                                                                                                                                                          
}                                                                                                                                                                                                            
}

Handling Type Changes Safely

// Cannot change string → long in place.                                                                                                                                                                       
// Strategy: dual-write period → validate → drop old.

// Phase 1: Add new typed column alongside old                                                                                                                                                                 
"dimensionsSpec": {                                                                                                                                                                                            
"dimensions": [                                                                                                                                                                                              
{ "type": "string", "name": "response_code" },      // old — keep for compat                                                                                                                               
{ "type": "long",   "name": "response_code_int" }   // new — populated at ingest via transform                                                                                                             
]                                                                                                                                                                                                            
},                                                                                                                                                                                                             
"transformSpec": {                                                                                                                                                                                             
"transforms": [                                                                                                                                                                                              
{ "type": "expression", "name": "response_code_int",                                                                                                                                                       
"expression": "CAST(response_code, 'LONG')" }                                                                                                                                                            
]                                                                                                                                                                                                            
}

// Phase 2: After backfill confirmed, stop writing old column                                                                                                                                                  
// Phase 3: Reindex to reclaim storage
                                                                                                                                                                                                                 
---                                                                                                                                                                                                            
4. Cost Attribution Per Tenant

Multi-Dimensional Cost Model

Cost = IngestCost + StorageCost + QueryCost + ReplicationCost

type CostRecord struct {                                                                                                                                                                                       
TenantID        string                                                                                                                                                                                     
Date            time.Time                                                                                                                                                                                  
// Ingest                                                                                                                                                                                                  
RowsIngested    int64                                                                                                                                                                                      
BytesIngested   int64                                                                                                                                                                                      
IngestTaskHours float64                                                                                                                                                                                    
// Storage                                                                                                                                                                                                 
SegmentBytes    int64   // across hot/warm/cold tiers                                                                                                                                                      
HotTierBytes    int64                                                                                                                                                                                      
WarmTierBytes   int64                                                                                                                                                                                      
ColdTierBytes   int64                                                                                                                                                                                      
// Query                                                                                                                                                                                                   
QueryCount      int64                                                                                                                                                                                      
RowsScanned     int64                                                                                                                                                                                      
BytesScanned    int64                                                                                                                                                                                      
QueryCPUSecs    float64                                                                                                                                                                                    
// Replication                                                                                                                                                                                             
ReplicaBytes    int64                                                                                                                                                                                      
}

type CostCalculator struct {                                                                                                                                                                                   
// Price per unit (configurable per cloud region)                                                                                                                                                          
IngestPricePerGB   float64  // $0.10/GB ingested                                                                                                                                                           
HotPricePerGBMonth float64  // $0.50/GB/month (NVMe)                                                                                                                                                       
WarmPricePerGBMonth float64 // $0.20/GB/month (SSD)                                                                                                                                                        
ColdPricePerGBMonth float64 // $0.02/GB/month (HDD/S3)                                                                                                                                                     
QueryPricePerTBScan float64 // $5.00/TB scanned                                                                                                                                                            
ComputePricePerHour float64 // $0.10/CPU-hour                                                                                                                                                              
}

func (cc *CostCalculator) Calculate(r CostRecord) TenantCost {                                                                                                                                                 
gb := float64(1 << 30)                                                                                                                                                                                     
tb := float64(1 << 40)

      return TenantCost{                                                                                                                                                                                         
          TenantID: r.TenantID,                                                                                                                                                                                  
          IngestCost: (float64(r.BytesIngested) / gb) * cc.IngestPricePerGB,                                                                                                                                     
          StorageCost: (float64(r.HotTierBytes)  / gb / 30) * cc.HotPricePerGBMonth +                                                                                                                            
                       (float64(r.WarmTierBytes) / gb / 30) * cc.WarmPricePerGBMonth +                                                                                                                           
                       (float64(r.ColdTierBytes) / gb / 30) * cc.ColdPricePerGBMonth,                                                                                                                            
          QueryCost:   (float64(r.BytesScanned) / tb) * cc.QueryPricePerTBScan +                                                                                                                                 
                       r.QueryCPUSecs / 3600 * cc.ComputePricePerHour,                                                                                                                                           
      }                                                                                                                                                                                                          
}

Cost Collection Pipeline

Druid Audit Log → Kafka → Flink/Spark → Cost DB (TimescaleDB) → Billing API

// Collect from Druid's request log (emitted per query)                                                                                                                                                        
type DruidRequestLog struct {                                                                                                                                                                                  
Query         json.RawMessage `json:"query"`                                                                                                                                                               
Host          string          `json:"host"`                                                                                                                                                                
Duration      int64           `json:"duration"`     // ms                                                                                                                                                  
BytesScanned  int64           `json:"bytesScanned"`                                                                                                                                                        
RowsScanned   int64           `json:"rowsScanned"`                                                                                                                                                         
// Injected by our gateway                                                                                                                                                                                 
TenantID      string          `json:"tenantId"`                                                                                                                                                            
ServiceTier   string          `json:"serviceTier"`                                                                                                                                                         
QueryID       string          `json:"queryId"`                                                                                                                                                             
}

// Emit Druid request log to Kafka for async cost attribution                                                                                                                                                  
druid.request.logging.type=compositeRequestLogger                                                                                                                                                              
druid.request.logging.loggerProviders[0].type=file                                                                                                                                                             
druid.request.logging.loggerProviders[1].type=kafka                                                                                                                                                            
druid.request.logging.loggerProviders[1].producer.bootstrap.servers=kafka:9092                                                                                                                                 
druid.request.logging.loggerProviders[1].topic=druid-request-log

Segment Storage Attribution

# Daily job: query Druid's sys.segments to attribute storage costs
import requests

def collect_storage_costs():                                                                                                                                                                                   
query = {                                                                                                                                                                                                  
"query": """                                                                                                                                                                                           
SELECT                                                                                                                                                                                             
datasource,                                                                                                                                                                                    
SUM(size) as total_bytes,                                                                                                                                                                      
COUNT(*) as segment_count,                                                                                                                                                                     
SUM(num_rows) as total_rows                                                                                                                                                                    
FROM sys.segments                                                                                                                                                                                  
WHERE is_active = 1                                                                                                                                                                                
GROUP BY datasource                                                                                                                                                                                
"""                                                                                                                                                                                                    
}

      segments = requests.post("http://broker:8082/druid/v2/sql", json=query).json()                                                                                                                             
                  
      for row in segments:                                                                                                                                                                                       
          tenant_id = resolve_tenant(row['datasource'])  # from tenant registry
          tier      = resolve_tier(row['datasource'])    # hot/warm/cold from coordinator                                                                                                                        
                                                                                                                                                                                                                 
          cost_db.upsert(CostRecord(                                                                                                                                                                             
              tenant_id    = tenant_id,                                                                                                                                                                          
              date         = today,                                                                                                                                                                              
              segment_bytes= row['total_bytes'],                                                                                                                                                                 
              tier         = tier,                                                                                                                                                                               
          ))                                                                                                                                                                                                     

Cost Dashboard Query

-- Per-tenant cost breakdown for billing period                                                                                                                                                                
SELECT                                                                                                                                                                                                         
tenant_id,
SUM(ingest_cost)  AS ingest_usd,                                                                                                                                                                           
SUM(storage_cost) AS storage_usd,                                                                                                                                                                          
SUM(query_cost)   AS query_usd,                                                                                                                                                                            
SUM(ingest_cost + storage_cost + query_cost) AS total_usd,                                                                                                                                                 
SUM(rows_ingested)  / 1e9 AS billion_rows_ingested,                                                                                                                                                        
SUM(bytes_scanned)  / 1e12 AS tb_scanned,                                                                                                                                                                  
SUM(query_count)    AS queries_executed                                                                                                                                                                    
FROM tenant_daily_costs                                                                                                                                                                                        
WHERE date BETWEEN '2026-03-01' AND '2026-03-31'                                                                                                                                                               
GROUP BY tenant_id                                                                                                                                                                                             
ORDER BY total_usd DESC;
                                                                                                                                                                                                                 
---                                                                                                                                                                                                            
Putting It All Together — Decision Flow

New Tenant Onboarded
│                                                                                                                                                                                                      
▼                                                                                                                                                                                                      
┌───────────────────────┐                                                                                                                                                                                      
│ Volume estimate?       │                                                                                                                                                                                     
│ >10GB/day → Tier A    │  ── Dedicated datasource, hot segments, premium broker                                                                                                                               
│ 1-10GB/day → Tier B   │  ── Shared datasource + single_dim partition, standard broker                                                                                                                        
│ <1GB/day → Tier C     │  ── Fully shared, rollup aggressive, standard broker                                                                                                                                 
└───────────────────────┘                                                                                                                                                                                      
│                                                                                                                                                                                                      
▼                                                                                                                                                                                                      
Assign Kafka topic (A: dedicated, B/C: shared with tenant_id key)                                                                                                                                              
│                                                                                                                                                                                                      
▼                                                                                                                                                                                                      
Supervisor created with correct partition spec + tiering rules                                                                                                                                                 
│                                                                                                                                                                                                      
▼                                                                                                                                                                                                      
Query Gateway enforces: timeout + quota + filter injection + audit                                                                                                                                             
│                                                                                                                                                                                                      
▼                                                                                                                                                                                                      
Cost pipeline attributes all bytes/CPU/storage → daily billing record
                                                                                                                                                                                                                 
---                                                                                                                                                                                                            
Key Trade-offs Summary

┌────────────────────────┬────────────────────────────────────┬──────────────────────────────────────────────────────────────────────────────────────┐
│        Decision        │               Choice               │                                      Rationale                                       │                                                         
├────────────────────────┼────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────┤                                                         
│ Isolation model        │ Tiered (dedicated vs shared)       │ Full isolation per tenant doesn't scale to 1000+ at reasonable cost                  │                                                         
├────────────────────────┼────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────┤                                                         
│ Partition key (shared) │ single_dim on tenant_id            │ Segment pruning gives near-dedicated isolation without dedicated datasource overhead │                                                         
├────────────────────────┼────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────┤                                                         
│ Rollup                 │ Enabled at ingest                  │ 10-100GB → 1-10GB typical after rollup; massive storage and query savings            │                                                         
├────────────────────────┼────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────┤                                                         
│ Schema evolution       │ Dual-write + backfill              │ Zero downtime; old segments return null for new columns — not errors                 │
├────────────────────────┼────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────┤                                                         
│ Query routing          │ Priority-tiered brokers            │ Premium tenants get dedicated merge buffers; noisy neighbour contained               │
├────────────────────────┼────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────┤                                                         
│ Cost attribution       │ Audit log + sys.segments           │ Real usage data, not estimates; enables accurate per-query and per-byte billing      │
├────────────────────────┼────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────┤                                                         
│ Cold storage           │ S3-backed deep storage             │ Druid can query directly via S3Select; no need to load into historical nodes         │
├────────────────────────┼────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────┤                                                         
│ Concurrency            │ Token bucket per tenant at gateway │ Prevents one tenant's burst from impacting others before it reaches Druid            │
└────────────────────────┴────────────────────────────────────┴──────────────────────────────────────────────────────────────────────────────────────┘    