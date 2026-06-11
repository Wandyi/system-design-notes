Staff-Level Practices for High-Volume Writes in PostgreSQL
                                                                                                                                                                                                                      
---                                                                                                                                                                                                                 
Mental Model First

Every write in Postgres touches multiple subsystems. Understanding this determines every decision:

INSERT row
│                                                                                                                                                                                                               
├── WAL record written (pg_wal/)         ← durability
├── Heap page modified (shared_buffers)  ← data storage                                                                                                                                                         
├── Every index updated                  ← read performance cost paid at write time
├── Visibility map updated               ← VACUUM bookkeeping                                                                                                                                                   
├── FSM (free space map) updated         ← space reuse
└── autovacuum eventually cleans dead tuples

High-write volume amplifies the cost of every one of these. Bad practices hit each layer differently.
                                                                                                                                                                                                                      
---                                                                                                                                                                                                                 
1. Insert Mechanism — COPY vs INSERT

Bad practice: Single-row inserts in a loop

-- Application doing this in a loop — catastrophic at scale
INSERT INTO events (user_id, type, payload) VALUES ($1, $2, $3);                                                                                                                                                    
-- Per row: parse query, plan, lock, WAL write, heap write, index updates, ACK                                                                                                                                      
-- 1M rows = 1M round trips × all of the above

Issues:
- Each statement is parsed and planned independently (even with prepared statements, execution overhead remains)
- If autocommit is on: 1M transactions, each flushing WAL to disk (fsync per commit)
- Network round-trip per row is the dominant cost at high row counts
- At 10k rows/sec this feels fine; at 500k rows/sec it collapses

Solution 1: Multi-row INSERT batching

-- Batch 1000 rows per statement                                                                                                                                                                                    
INSERT INTO events (user_id, type, payload) VALUES                                                                                                                                                                  
($1, $2, $3),                                                                                                                                                                                                   
($4, $5, $6),
...  -- up to ~1000 rows                                                                                                                                                                                        
($2998, $2999, $3000);

// Go: build batch with pgx                                                                                                                                                                                         
batch := &pgx.Batch{}
for i, row := range rows {                                                                                                                                                                                          
batch.Queue(                                                                                                                                                                                                    
"INSERT INTO events (user_id, type, payload) VALUES ($1, $2, $3)",
row.UserID, row.Type, row.Payload,                                                                                                                                                                          
)                                                                                                                                                                                                               
if (i+1)%1000 == 0 {
results := conn.SendBatch(ctx, batch)                                                                                                                                                                       
if err := results.Close(); err != nil {                                                                                                                                                                     
return fmt.Errorf("batch flush: %w", err)
}                                                                                                                                                                                                           
batch = &pgx.Batch{}
}                                                                                                                                                                                                               
}

Solution 2: COPY protocol — fastest path into Postgres

COPY bypasses:                                                                                                                                                                                                      
- query parsing
- query planning                                                                                                                                                                                                  
- per-row constraint checks (done at end)
- per-row index updates (batched internally)

COPY does not bypass:                                                                                                                                                                                               
- WAL (unless unlogged table)                                                                                                                                                                                     
- indexes (still updated, just more efficiently)                                                                                                                                                                  
- triggers (unless COPY FROM with triggers disabled)

// pgx COPY — 5-10x faster than batched INSERTs for bulk loads
rows := [][]any{}                                                                                                                                                                                                   
for _, event := range events {                                                                                                                                                                                      
rows = append(rows, []any{event.UserID, event.Type, event.Payload, event.CreatedAt})                                                                                                                            
}

copyCount, err := conn.CopyFrom(                                                                                                                                                                                    
ctx,        
pgx.Identifier{"events"},                                                                                                                                                                                       
[]string{"user_id", "type", "payload", "created_at"},
pgx.CopyFromRows(rows),
)

Throughput comparison on same hardware (approximate):                                                                                                                                                               
Single-row INSERTs:          ~5,000 rows/sec
Batched INSERTs (1000/batch): ~80,000 rows/sec                                                                                                                                                                      
COPY (binary):               ~300,000+ rows/sec
COPY + unlogged table:       ~600,000+ rows/sec
                                                                                                                                                                                                                      
---                                                                                                                                                                                                                 
2. Transaction Design

Bad practice A: One transaction per row (autocommit)

1M inserts = 1M transactions
Each commit = WAL flush to disk (if synchronous_commit = on)                                                                                                                                                        
1M fsyncs × ~0.1ms each = ~100 seconds just in fsync overhead

Bad practice B: One giant transaction for everything

BEGIN;          
INSERT ... (row 1)
INSERT ... (row 2)                                                                                                                                                                                                  
...
INSERT ... (row 1,000,000)  -- holding locks entire time                                                                                                                                                            
COMMIT;                      -- WAL flush of 1M rows at once

Issues:
- Holds table/row locks for the entire duration
- Any failure = roll back 1M rows (expensive, generates WAL for undo)
- WAL accumulation: standby replication lag spikes while transaction is open
- pg_wal can fill disk if transaction exceeds max_wal_size
- Long-running transactions block VACUUM from reclaiming dead tuples

Solution: Right-sized transaction batches

// Optimal batch size: 1,000–10,000 rows per transaction                                                                                                                                                            
// Balance between: commit overhead vs lock duration vs rollback cost

const batchSize = 5000

tx, err := pool.Begin(ctx)                                                                                                                                                                                          
if err != nil {
return err                                                                                                                                                                                                      
}

for i, row := range rows {
if err := insertRow(ctx, tx, row); err != nil {
tx.Rollback(ctx)                                                                                                                                                                                            
return fmt.Errorf("row %d: %w", i, err)
}

      if (i+1)%batchSize == 0 {
          if err := tx.Commit(ctx); err != nil {                                                                                                                                                                      
              return fmt.Errorf("commit batch %d: %w", i/batchSize, err)
          }
          tx, err = pool.Begin(ctx)                                                                                                                                                                                   
          if err != nil {
              return err                                                                                                                                                                                              
          }       
      }
}
// commit final partial batch
return tx.Commit(ctx)

WAL durability tuning for non-critical bulk loads

-- Per-session: reduces fsync overhead for this connection's writes                                                                                                                                                 
-- Data can be lost on OS crash (not DB crash) — acceptable for reloadable data                                                                                                                                     
SET synchronous_commit = off;

-- Per-transaction: entire txn treated as synchronous_commit=off                                                                                                                                                    
-- Use for: event ingestion, analytics events, replayable logs
BEGIN;                                                                                                                                                                                                              
SET LOCAL synchronous_commit = off;
INSERT ...;                                                                                                                                                                                                         
COMMIT;

synchronous_commit = on:  commit waits for WAL flush to disk — durable                                                                                                                                              
synchronous_commit = off: commit returns after WAL sent to OS buffer — ~200ms loss window                                                                                                                           
synchronous_commit = local: flush to local disk only, not to standby
                                                                                                                                                                                                                      
---                                                                                                                                                                                                                 
3. Index Strategy During Bulk Loads

Bad practice: Bulk loading into a fully-indexed table

Every INSERT into a table with 6 indexes:                                                                                                                                                                           
- 1 heap write
- 6 index B-tree updates (each may cause page splits)                                                                                                                                                             
- At 500k rows: potentially millions of page splits across indexes                                                                                                                                                
- Index bloat accumulates rapidly

Issue: B-tree page splits under sequential load

When inserting sequential IDs (SERIAL, SEQUENCE), all inserts go to the rightmost leaf page of every B-tree index. This page is heavily contended:

Sequential insert pattern:                                                                                                                                                                                          
[1][2][3][4][5]  ← all threads writing to same page
← lock contention on single buffer page                                                                                                                                                         
← page split: [1..500][501..1000]                                                                                                                                                               
← split writes two pages, both to WAL

Solution: Drop and rebuild non-critical indexes

-- Before bulk load: drop indexes you don't need during the load                                                                                                                                                    
DROP INDEX CONCURRENTLY idx_events_user_id;
DROP INDEX CONCURRENTLY idx_events_created_at;
-- Keep only PRIMARY KEY and UNIQUE constraints (required for correctness)

-- Run bulk load

-- After bulk load: rebuild — much faster to build index from sorted data                                                                                                                                           
-- than to update it row by row during insert
CREATE INDEX CONCURRENTLY idx_events_user_id ON events(user_id);                                                                                                                                                    
CREATE INDEX CONCURRENTLY idx_events_created_at ON events(created_at);                                                                                                                                              
-- CONCURRENTLY: doesn't lock table, but slower than non-concurrent                                                                                                                                                 
-- For maintenance windows: omit CONCURRENTLY for 2-5x faster build

Why index build from scratch is faster:                                                                                                                                                                             
Row-by-row index update:  O(N log N) random page writes during insert                                                                                                                                               
Bulk index build:         Sort all values → O(N log N) sequential scan
Sequential I/O is 10-50x faster than random I/O

Solution: Partial indexes to reduce index overhead

-- Instead of indexing all rows, index only what queries actually filter on                                                                                                                                         
-- Cuts index size and update cost proportionally

-- BAD: index all events
CREATE INDEX idx_events_status ON events(status);

-- GOOD: if 95% of queries only care about 'pending' events                                                                                                                                                         
CREATE INDEX idx_events_pending ON events(id)                                                                                                                                                                       
WHERE status = 'pending';                                                                                                                                                                                       
-- 20x smaller index, 20x fewer index updates for non-pending inserts
                                                                                                                                                                                                                      
---
4. WAL and Checkpoint Tuning

Bad practice: Default checkpoint settings under heavy write load

Default max_wal_size = 1GB. Under heavy writes:

WAL fills quickly → frequent checkpoints triggered                                                                                                                                                                  
Checkpoint: flush all dirty pages from shared_buffers to disk
Untuned: checkpoint completes in burst → massive I/O spike → write latency spikes                                                                                                                                   
visible as p99 latency cliff every few minutes

Solution: Tune for sustained throughput

# postgresql.conf

# How much WAL can accumulate before a checkpoint is forced
# Higher = fewer checkpoints = less I/O pressure
# Cost: longer crash recovery time (replays more WAL)
max_wal_size = 8GB          # default: 1GB, raise for write-heavy workloads

# Spread checkpoint I/O over this fraction of checkpoint_timeout
# 0.9 = checkpoint uses 90% of the interval, leaving headroom for queries
checkpoint_completion_target = 0.9   # default: 0.9 (already good)

# How often to checkpoint even if max_wal_size not reached
checkpoint_timeout = 15min   # default: 5min — raise for write-heavy

# Shared memory for dirty pages — larger = fewer writes to disk between checkpoints
shared_buffers = 25% of RAM  # rule of thumb

# WAL writer flushes — reduce for write-heavy workloads
wal_compression = on         # compresses WAL records, reduces WAL volume ~40%

Monitoring checkpoints:                                                                                                                                                                                             
SELECT                                                                                                                                                                                                              
checkpoints_timed,       -- triggered by checkpoint_timeout                                                                                                                                                     
checkpoints_req,         -- triggered by max_wal_size (bad — WAL pressure)
checkpoint_write_time,                                                    
checkpoint_sync_time,    -- high sync_time = I/O bottleneck                                                                                                                                                     
buffers_checkpoint,                                                                                                                                                                                             
buffers_clean,           -- bgwriter activity                                                                                                                                                                   
buffers_backend          -- backends writing their own dirty pages (bad sign)                                                                                                                                   
FROM pg_stat_bgwriter;

-- If checkpoints_req >> checkpoints_timed: increase max_wal_size
-- If buffers_backend > 0: bgwriter can't keep up, increase bgwriter_lru_maxpages
                  
---                                                                                                                                                                                                                 
5. Constraint and Foreign Key Handling

Bad practice: FK constraints during bulk insert into child table

-- Table: orders references users(id)
-- Inserting 1M orders:                                                                                                                                                                                             
--   Every row checks: SELECT 1 FROM users WHERE id = $user_id                                                                                                                                                      
--   Acquires SHARE ROW EXCLUSIVE lock on users table                                                                                                                                                               
--   1M lock acquisitions on a hot table                                                                                                                                                                            
--   If users table is also being written to: deadlock risk

Solution: Defer constraint checks to end of transaction

-- Make constraints deferrable at table creation
ALTER TABLE orders                                                                                                                                                                                                  
DROP CONSTRAINT orders_user_id_fkey,
ADD CONSTRAINT orders_user_id_fkey                                                                                                                                                                              
FOREIGN KEY (user_id) REFERENCES users(id)
DEFERRABLE INITIALLY DEFERRED;

-- Now FK check happens once at COMMIT, not per row                                                                                                                                                                 
-- Dramatically reduces lock contention during insert
BEGIN;                                                                                                                                                                                                              
SET CONSTRAINTS orders_user_id_fkey DEFERRED;
-- bulk insert 1M rows                                                                                                                                                                                              
COMMIT;  -- FK check happens here, once

Solution: Staging table pattern for bulk loads

-- Load into unindexed, unconstrained staging table first                                                                                                                                                           
CREATE UNLOGGED TABLE events_staging (
LIKE events INCLUDING DEFAULTS                                                                                                                                                                                  
-- no constraints, no FK, no indexes
);

-- Bulk load into staging (fast — no indexes, no WAL)                                                                                                                                                               
COPY events_staging FROM '/data/events.csv';

-- Validate and move to production table in one statement                                                                                                                                                           
INSERT INTO events
SELECT * FROM events_staging                                                                                                                                                                                        
WHERE user_id IN (SELECT id FROM users)  -- apply FK check in bulk
ON CONFLICT (id) DO NOTHING;             -- handle duplicates

TRUNCATE events_staging;
                                                                                                                                                                                                                      
---             
6. Table Design for High Write Volume

Issue: Table bloat from UPDATE-heavy workloads

Postgres uses MVCC — UPDATEs don't modify rows in place. They write a new row version and mark the old one as dead.

UPDATE users SET last_seen = now() WHERE id = 1;

Before: [id=1, last_seen=yesterday, xmin=100, xmax=0]    ← live                                                                                                                                                     
After:  [id=1, last_seen=yesterday, xmin=100, xmax=200]  ← dead (xmax set)
[id=1, last_seen=now,       xmin=200, xmax=0]    ← new live version

Dead tuple occupies space until VACUUM reclaims it                                                                                                                                                                  
High UPDATE rate → dead tuples accumulate faster than autovacuum can clean

Solution: fill_factor for update-heavy tables

-- fill_factor=70: leave 30% of each page empty for in-place HOT updates
-- HOT (Heap Only Tuple): update stays on same page → no index update needed                                                                                                                                        
--                        only possible if updated column is NOT indexed                                                                                                                                            
--                        only possible if new row fits in the free space on same page                                                                                                                              
ALTER TABLE user_sessions SET (fillfactor = 70);

-- When to use:                                                                                                                                                                                                     
--   Tables with frequent UPDATE on non-indexed columns (e.g., last_seen, status)
--   fill_factor = 70-80 for moderate update rate                                                                                                                                                                   
--   fill_factor = 50-60 for very high update rate
-- Cost: table is 20-50% larger on disk                                                                                                                                                                             
-- Benefit: HOT updates = no index churn, far less dead tuple bloat

Solution: Partitioning for continuous time-series writes

-- Without partitioning: one massive table
-- autovacuum scans entire table, takes longer and longer                                                                                                                                                           
-- Old data's dead tuples bloat the table even if rarely queried                                                                                                                                                    
-- Dropping old data = expensive DELETE + VACUUM

-- With range partitioning by time:                                                                                                                                                                                 
CREATE TABLE events (
id          BIGINT GENERATED ALWAYS AS IDENTITY,                                                                                                                                                                
created_at  TIMESTAMPTZ NOT NULL,
user_id     BIGINT,                                                                                                                                                                                             
type        TEXT,
payload     JSONB                                                                                                                                                                                               
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2026_04 PARTITION OF events
FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');

CREATE TABLE events_2026_05 PARTITION OF events
FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

-- Benefits:
-- autovacuum works on smaller partitions — faster, more effective
-- DROP old partition = instant (vs DELETE millions of rows)                                                                                                                                                        
-- Writes always go to one partition — hot page contention concentrated but manageable                                                                                                                              
-- Partition pruning: queries on recent data don't scan old partitions

Automate partition creation:                                                                                                                                                                                        
-- pg_partman extension handles this automatically                                                                                                                                                                  
SELECT partman.create_parent(                     
p_parent_table => 'public.events',                                                                                                                                                                              
p_control => 'created_at',        
p_type => 'range',                                                                                                                                                                                              
p_interval => '1 month',
p_premake => 3  -- create 3 future partitions in advance                                                                                                                                                        
);

Unlogged tables for ephemeral high-write data

-- Unlogged: skips WAL entirely
-- 2-5x faster writes                                                                                                                                                                                               
-- Data LOST on crash (truncated on recovery)                                                                                                                                                                       
-- Use for: session state, caches, work queues, staging tables, derived data

CREATE UNLOGGED TABLE rate_limit_counters (                                                                                                                                                                         
key       TEXT PRIMARY KEY,                                                                                                                                                                                     
count     INT DEFAULT 0,                                                                                                                                                                                        
window_end TIMESTAMPTZ
);                                                                                                                                                                                                                  
-- Also: any table you can reconstruct from another source
                                                                                                                                                                                                                      
---             
7. Sequence and ID Strategy

Bad practice: SERIAL/SEQUENCE with high concurrency

CREATE TABLE events (id SERIAL PRIMARY KEY, ...);
-- 100 concurrent writers all calling nextval('events_id_seq')                                                                                                                                                      
-- nextval takes a lightweight lock on the sequence                                                                                                                                                                 
-- Contention at extreme concurrency (>1000 concurrent inserts/sec)

Worse: sequential IDs and B-tree index hot pages                                                                                                                                                                    
Sequential IDs [1, 2, 3, 4, ...] always insert at rightmost page of pk index                                                                                                                                        
All writers contend on the same buffer page lock                                                                                                                                                                    
Causes: "buffer lock contention" visible in pg_stat_activity wait events

Solution A: UUIDv7 — time-ordered, no sequence contention

-- UUIDv7: time-sortable UUID (first 48 bits = millisecond timestamp)                                                                                                                                               
-- No central sequence → no contention                                                                                                                                                                              
-- Roughly sequential within a millisecond → good B-tree locality
-- Globally unique → safe for distributed systems, merges, replication

CREATE TABLE events (                                                                                                                                                                                               
id UUID DEFAULT gen_random_uuid() PRIMARY KEY,  -- UUIDv4: fully random (worst for indexes)                                                                                                                     
-- OR with UUIDv7 (Postgres 17+ or pg_idkit extension):                                                                                                                                                         
id UUID DEFAULT uuidv7() PRIMARY KEY,                                                                                                                                                                           
...                                                                                                                                                                                                             
);

-- UUIDv4 downside: fully random → random B-tree inserts → index bloat + slow inserts                                                                                                                               
-- UUIDv7 preserves time ordering → near-sequential inserts → much better index behavior

Solution B: Sequence with cache for high throughput

-- Each backend pre-allocates 1000 IDs from the sequence                                                                                                                                                            
-- Reduces sequence lock contention by 1000x                                                                                                                                                                        
-- Gaps in IDs are acceptable in most applications
ALTER SEQUENCE events_id_seq CACHE 1000;
                                                                                                                                                                                                                      
---                                                                                                                                                                                                                 
8. ON CONFLICT (Upsert) Pitfalls

Bad practice: Upsert on every write

-- Looks harmless, but:
INSERT INTO counters (key, value) VALUES ($1, $2)
ON CONFLICT (key) DO UPDATE SET value = counters.value + EXCLUDED.value;                                                                                                                                            
-- Takes FOR UPDATE lock on conflicting row
-- Under high concurrency on same key: lock queue forms                                                                                                                                                             
-- Queue depth grows faster than it drains → latency explosion

Issue: lock queuing on hot keys                                                                                                                                                                                     
10 concurrent upserts on key='homepage_views':                                                                                                                                                                      
Thread 1: gets lock, updates                                                                                                                                                                                      
Threads 2-10: queue behind Thread 1                                                                                                                                                                               
Thread 1 commits, Thread 2 gets lock, etc.                                                                                                                                                                        
Throughput for this key = 1 update at a time regardless of parallelism

Solution: Counter aggregation pattern

// Don't upsert every event — accumulate in memory, flush periodically                                                                                                                                              
type CounterBuffer struct {                                                                                                                                                                                         
mu       sync.Mutex                                                                                                                                                                                             
counters map[string]int64                                                                                                                                                                                       
}

func (b *CounterBuffer) Increment(key string, delta int64) {                                                                                                                                                        
b.mu.Lock()
b.counters[key] += delta                                                                                                                                                                                        
b.mu.Unlock()
}

// Flush every second — one upsert per unique key per second                                                                                                                                                        
func (b *CounterBuffer) Flush(ctx context.Context, db *pgxpool.Pool) error {
b.mu.Lock()                                                                                                                                                                                                     
snapshot := b.counters
b.counters = make(map[string]int64)                                                                                                                                                                             
b.mu.Unlock()

      // Single upsert per unique key, batched in one transaction                                                                                                                                                     
      batch := &pgx.Batch{}
      for key, delta := range snapshot {                                                                                                                                                                              
          batch.Queue(`
              INSERT INTO counters (key, value) VALUES ($1, $2)                                                                                                                                                       
              ON CONFLICT (key) DO UPDATE SET value = counters.value + EXCLUDED.value`,
              key, delta,                                                                                                                                                                                             
          )
      }                                                                                                                                                                                                               
      results := db.SendBatch(ctx, batch)
      return results.Close()
}

  ---                                                                                                                                                                                                                 
9. Connection Management

Bad practice: One DB connection per application goroutine

100 goroutines → 100 DB connections
Each Postgres backend = ~5-10MB RAM + OS process overhead                                                                                                                                                           
1000 connections = 5-10GB RAM just for connection overhead                                                                                                                                                          
Postgres degrades significantly above ~300-500 active connections

Solution: PgBouncer in transaction pooling mode

Application → PgBouncer (transaction pooling) → Postgres                                                                                                                                                            
1000 app connections     ←→     25 Postgres connections

Transaction pooling: connection returned to pool after each COMMIT/ROLLBACK                                                                                                                                         
Most efficient for OLTP write workloads                                                                                                                                                                             
Limitation: session-level features don't work (SET, prepared statements, advisory locks)                                                                                                                            
→ use pgx's built-in pool for most Go services instead

// pgxpool: built-in connection pooling for Go                                                                                                                                                                      
// Respects context cancellation on pool.Acquire()                                                                                                                                                                  
pool, err := pgxpool.New(ctx, connString)                                                                                                                                                                           
pool.Config().MaxConns = 25            // match to Postgres max_connections / num_services                                                                                                                          
pool.Config().MinConns = 5             // keep warm connections ready                                                                                                                                               
pool.Config().MaxConnLifetime = 1*time.Hour                                                                                                                                                                         
pool.Config().MaxConnIdleTime = 30*time.Minute                                                                                                                                                                      
pool.Config().HealthCheckPeriod = 1*time.Minute
                                                                                                                                                                                                                      
---
10. Autovacuum Tuning for High-Write Tables

Bad practice: Default autovacuum settings on high-write tables

Default: autovacuum triggers when 20% of table is dead tuples
1M row table: triggers at 200,000 dead tuples                                                                                                                                                                       
10M row table: triggers at 2,000,000 dead tuples — by then, table bloat is significant                                                                                                                              
And query planner uses stale statistics = bad query plans

Solution: Per-table autovacuum overrides

-- For high-write tables: vacuum more aggressively
ALTER TABLE events SET (                                                                                                                                                                                            
autovacuum_vacuum_scale_factor = 0.01,   -- trigger at 1% dead tuples (default: 20%)
autovacuum_analyze_scale_factor = 0.005, -- analyze at 0.5% changed (default: 10%)                                                                                                                              
autovacuum_vacuum_cost_delay = 2,        -- less throttling (default: 2ms — already low)                                                                                                                        
autovacuum_vacuum_insert_scale_factor = 0.05  -- vacuum after 5% new inserts (for insert-only tables)                                                                                                           
);

-- For critical tables: run manual VACUUM during low-traffic windows                                                                                                                                                
VACUUM (ANALYZE, VERBOSE) events;

-- Check vacuum health:
SELECT                                                                                                                                                                                                              
schemaname,
relname,
n_dead_tup,                                                                                                                                                                                                     
n_live_tup,
round(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_pct,                                                                                                                           
last_vacuum,                                                                                                                                                                                                    
last_autovacuum,
last_analyze                                                                                                                                                                                                    
FROM pg_stat_user_tables
WHERE relname = 'events';
                                                                                                                                                                                                                      
---
11. Monitoring the Write Path

-- 1. Are checkpoints too frequent? (WAL pressure)
SELECT checkpoints_req, checkpoints_timed,                                                                                                                                                                          
round(checkpoints_req::numeric / (checkpoints_req + checkpoints_timed) * 100, 1)                                                                                                                             
AS forced_pct                                                                                                                                                                                            
FROM pg_stat_bgwriter;                                                                                                                                                                                              
-- forced_pct > 10%: increase max_wal_size

-- 2. What are writers waiting on?                                                                                                                                                                                  
SELECT wait_event_type, wait_event, count(*)                                                                                                                                                                        
FROM pg_stat_activity                                                                                                                                                                                               
WHERE state = 'active' AND wait_event IS NOT NULL
GROUP BY 1, 2                                                                                                                                                                                                       
ORDER BY 3 DESC;
-- 'Lock' waits: contention on rows/tables                                                                                                                                                                          
-- 'BufferPin' / 'Buffer': shared_buffers contention                                                                                                                                                                
-- 'WALWrite': WAL I/O bottleneck                                                                                                                                                                                   
-- 'DataFileWrite': heap/index I/O bottleneck

-- 3. Table bloat                                                                                                                                                                                                   
SELECT                                                                                                                                                                                                              
relname,    
pg_size_pretty(pg_total_relation_size(oid)) AS total_size,
pg_size_pretty(pg_relation_size(oid)) AS heap_size,                                                                                                                                                             
pg_size_pretty(pg_total_relation_size(oid) - pg_relation_size(oid)) AS index_size,
n_dead_tup,                                                                                                                                                                                                     
n_live_tup  
FROM pg_stat_user_tables                                                                                                                                                                                            
ORDER BY pg_total_relation_size(oid) DESC
LIMIT 20;

-- 4. Index bloat (from pgstattuple extension)                                                                                                                                                                      
SELECT * FROM pgstattuple('idx_events_user_id');                                                                                                                                                                    
-- free_percent > 30%: consider REINDEX CONCURRENTLY

-- 5. Replication lag (don't let writes get too far ahead of standbys)                                                                                                                                              
SELECT                                                                                                                                                                                                              
application_name,                                                                                                                                                                                               
state,      
write_lag,
flush_lag,                                                                                                                                                                                                      
replay_lag
FROM pg_stat_replication;
                  
---
Summary: Issues → Causes → Solutions
```
┌──────────────────────────────┬──────────────────────────────────────────────────────┬─────────────────────────────────────────────────────────────────┐
│            Issue             │                        Cause                         │                            Solution                             │                                                           
├──────────────────────────────┼──────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────┤
│ Low insert throughput        │ Single-row autocommit inserts                        │ COPY or batched INSERTs in explicit transactions                │
├──────────────────────────────┼──────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────┤
│ WAL I/O spike                │ Frequent forced checkpoints                          │ Increase max_wal_size, tune checkpoint_completion_target        │                                                           
├──────────────────────────────┼──────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────┤                                                           
│ Index write amplification    │ Fully indexed table during bulk load                 │ Drop/rebuild indexes around bulk loads                          │                                                           
├──────────────────────────────┼──────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────┤                                                           
│ Table bloat                  │ MVCC dead tuples accumulating faster than autovacuum │ Lower autovacuum_vacuum_scale_factor per table, use fill_factor │
├──────────────────────────────┼──────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────┤                                                           
│ Lock contention on hot rows  │ High-frequency upserts on same keys                  │ Buffer writes in application, flush aggregated deltas           │
├──────────────────────────────┼──────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────┤                                                           
│ FK check overhead            │ Row-by-row FK validation during bulk insert          │ DEFERRABLE INITIALLY DEFERRED or staging table pattern          │
├──────────────────────────────┼──────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────┤                                                           
│ Sequence contention at scale │ All writers calling nextval()                        │ SEQUENCE CACHE 1000 or UUIDv7                                   │
├──────────────────────────────┼──────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────┤                                                           
│ B-tree right-page contention │ Sequential IDs under concurrent writes               │ UUIDv7 or hash-distributed IDs                                  │
├──────────────────────────────┼──────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────┤                                                           
│ Long-txn VACUUM blockage     │ One giant bulk-load transaction                      │ Right-sized batches (5k–10k rows/txn)                           │
├──────────────────────────────┼──────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────┤                                                           
│ Connection overhead          │ Too many Postgres backends                           │ pgxpool or PgBouncer transaction pooling                        │
├──────────────────────────────┼──────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────┤                                                           
│ Partition bloat on old data  │ No partitioning strategy                             │ Time-range partitioning + DROP old partitions                   │
├──────────────────────────────┼──────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────┤                                                           
│ Stale query plans            │ Autovacuum analyze not keeping up                    │ Lower autovacuum_analyze_scale_factor on hot tables             │
├──────────────────────────────┼──────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────┤                                                           
│ Replication lag spikes       │ Large transactions holding WAL open                  │ Smaller transactions, wal_compression = on                      │
└──────────────────────────────┴──────────────────────────────────────────────────────┴─────────────────────────────────────────────────────────────────┘      
```