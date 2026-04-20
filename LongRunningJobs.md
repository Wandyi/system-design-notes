Long-Running Job System in Go

  ---
Architecture Overview

┌──────────────────────────────────────────────────────────────────┐
│                         Worker Process                            │
│                                                                   │
│  ┌─────────────┐    ┌──────────────┐    ┌──────────────────────┐ │
│  │  Poll Loop  │───▶│ Claim (Lock) │───▶│  Execute Job         │ │
│  │             │    │  (atomic SQL)│    │  ┌────────────────┐  │ │
│  └─────────────┘    └──────────────┘    │  │ Heartbeat Loop │  │ │
│                                         │  │ (goroutine)    │  │ │
│                                         │  └───────┬────────┘  │ │
│                                         │          │ cancel on  │ │
│                                         │          │ lock loss  │ │
│                                         │  ┌───────▼────────┐  │ │
│                                         │  │ Process Batches│  │ │
│                                         │  │ + Checkpoint   │  │ │
│                                         │  └────────────────┘  │ │
│                                         └──────────────────────┘ │
└─────────────────────────────────┬────────────────────────────────┘
│
┌─────────────▼─────────────┐
│         PostgreSQL          │
│                            │
│  jobs table                │
│  ├── status                │
│  ├── worker_id             │
│  ├── heartbeat_at  ◀───────┼── renewed every 5s
│  ├── locked_until          │
│  └── checkpoint (JSONB) ◀──┼── saved every batch
└────────────────────────────┘

The 6 properties and where they live:
Single owner         → FOR UPDATE SKIP LOCKED in claim query
Prove liveness       → heartbeat_at renewed every 5s by heartbeat goroutine
Forward progress     → checkpoint JSONB column updated after every batch
Safe stop            → context.WithCancelCause + graceful drain
Resume after failure → checkpoint loaded on claim, process resumes from offset
Recover if crashed   → stale heartbeat_at detected in next worker's claim query

  ---
1. Database Schema

CREATE TABLE jobs (
id            TEXT        PRIMARY KEY,
name          TEXT        NOT NULL,
status        TEXT        NOT NULL DEFAULT 'pending',
-- pending | running | completed | failed

      -- Ownership
      worker_id     TEXT,                  -- which worker holds the lock right now
      locked_until  TIMESTAMPTZ,           -- lock hard expiry (backup safety net)
      heartbeat_at  TIMESTAMPTZ,           -- last proof of life from the worker

      -- Progress
      checkpoint    JSONB       NOT NULL DEFAULT '{}',
                    -- opaque blob: the job defines its own structure
                    -- e.g. {"offset": 42000, "cursor": "2026-01-15T10:00:00Z"}

      -- Lifecycle
      attempt       INT         NOT NULL DEFAULT 0,
      max_attempts  INT         NOT NULL DEFAULT 3,
      started_at    TIMESTAMPTZ,
      completed_at  TIMESTAMPTZ,
      error         TEXT,
      created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Efficient scan for claimable jobs
CREATE INDEX idx_jobs_claimable
ON jobs (name, status, heartbeat_at)
WHERE status IN ('pending', 'running', 'failed');

  ---
2. Core Types and Executor Interface

package job

import (
"context"
"database/sql"
"errors"
"fmt"
"log/slog"
"time"

      "github.com/google/uuid"
)

var (
// ErrLockLost is returned when the worker's lock was taken by another worker.
// This is not a job failure — the job will be resumed by whoever took the lock.
ErrLockLost    = errors.New("job lock lost to another worker")
ErrNoJobsFound = errors.New("no claimable jobs")
)

// BatchResult is returned by the Executor after processing one batch.
type BatchResult struct {
Checkpoint []byte // serialized state — passed back as-is on next Process call
Processed  int64  // items processed in this batch (for progress reporting)
Total      int64  // total items, 0 if unknown
Done       bool   // true = job is fully complete, no more batches
Message    string // human-readable status: "imported 42,000 / 100,000 rows"
}

// Executor is the interface the application implements.
// The framework handles everything else: locking, heartbeating, checkpointing.
type Executor interface {
// Process is called repeatedly until result.Done == true or an error occurs.
// checkpoint is nil on the very first run, or the value from the last saved BatchResult.
// Process must respect ctx cancellation — return promptly when ctx is cancelled.
Process(ctx context.Context, checkpoint []byte) (BatchResult, error)
}

// Config holds tuning parameters.
type Config struct {
JobName  string
DB       *sql.DB
Executor Executor

      HeartbeatInterval time.Duration // how often to renew the lock (default: 5s)
      LockTTL           time.Duration // lock expires after this if heartbeat stops (default: 30s)
      StaleLockAge      time.Duration // steal lock if heartbeat older than this (default: 60s)
      PollInterval      time.Duration // how often to scan for claimable jobs (default: 10s)

      // OnProgress is called after every batch for metrics/logging.
      // Safe to leave nil.
      OnProgress func(jobID string, processed, total int64, message string)
}

func (c *Config) applyDefaults() {
if c.HeartbeatInterval == 0 { c.HeartbeatInterval = 5 * time.Second }
if c.LockTTL == 0           { c.LockTTL = 30 * time.Second }
if c.StaleLockAge == 0      { c.StaleLockAge = 60 * time.Second }
if c.PollInterval == 0      { c.PollInterval = 10 * time.Second }
if c.OnProgress == nil      { c.OnProgress = func(string, int64, int64, string) {} }
}

  ---
3. Worker — Main Loop

// Worker manages the full lifecycle of a single job type.
type Worker struct {
id  string // unique per process instance, generated at startup
cfg Config
}

func NewWorker(cfg Config) *Worker {
cfg.applyDefaults()
return &Worker{id: uuid.New().String(), cfg: cfg}
}

// Run is the entry point. It polls for claimable jobs, runs them, and loops.
// It exits cleanly when ctx is cancelled (e.g. SIGTERM received).
func (w *Worker) Run(ctx context.Context) error {
slog.Info("worker started", "worker_id", w.id, "job", w.cfg.JobName)
defer slog.Info("worker stopped", "worker_id", w.id)

      ticker := time.NewTicker(w.cfg.PollInterval)
      defer ticker.Stop()

      for {
          record, err := w.claim(ctx)
          switch {
          case err == nil:
              // Got a job — run it
              slog.Info("job claimed",
                  "worker_id", w.id,
                  "job_id", record.id,
                  "attempt", record.attempt,
              )
              if err := w.execute(ctx, record); err != nil && !errors.Is(err, ErrLockLost) {
                  // Real failure: mark the job as failed (or pending for retry)
                  if merr := w.markFailed(context.Background(), record.id, err); merr != nil {
                      slog.Error("could not mark job failed — requires manual intervention",
                          "job_id", record.id, "exec_err", err, "mark_err", merr,
                      )
                  }
              }

          case errors.Is(err, ErrNoJobsFound):
              // Nothing to do — wait for next poll tick or shutdown
              select {
              case <-ctx.Done():
                  return nil
              case <-ticker.C:
              }

          case errors.Is(err, context.Canceled), errors.Is(err, context.DeadlineExceeded):
              return nil // clean shutdown

          default:
              return fmt.Errorf("claim: %w", err)
          }
      }
}

  ---
4. Property 1 — Single Owner: Atomic Lock Acquisition

The claim query is a single atomic UPDATE ... WHERE ... FOR UPDATE SKIP LOCKED.

- FOR UPDATE SKIP LOCKED: if two workers race, one gets the row, the other skips it — no blocking, no double-claim.
- The WHERE clause also detects stale locks (crashed workers) and failed-but-retryable jobs.

type claimedJob struct {
id         string
checkpoint []byte
attempt    int
}

// claim atomically acquires ownership of a claimable job.
//
// A job is claimable if:
//   status = 'pending'                                           (new job)
//   status = 'running' AND heartbeat_at < now() - StaleLockAge  (crashed worker)
//   status = 'failed'  AND attempt < max_attempts               (retry eligible)
func (w *Worker) claim(ctx context.Context) (*claimedJob, error) {
lockSecs  := w.cfg.LockTTL.Seconds()
staleSecs := w.cfg.StaleLockAge.Seconds()

      var job claimedJob
      err := w.cfg.DB.QueryRowContext(ctx, `
          UPDATE jobs
          SET
              worker_id    = $1,
              locked_until = NOW() + make_interval(secs => $2),
              heartbeat_at = NOW(),
              status       = 'running',
              attempt      = attempt + 1,
              started_at   = COALESCE(started_at, NOW()),
              error        = NULL
          WHERE id = (
              SELECT id FROM jobs
              WHERE name = $3
                AND (
                    status = 'pending'
                    OR (status = 'running'
                        AND heartbeat_at < NOW() - make_interval(secs => $4))
                    OR (status = 'failed'
                        AND attempt < max_attempts)
                )
              ORDER BY created_at ASC
              LIMIT 1
              FOR UPDATE SKIP LOCKED   -- concurrent workers skip each other's rows
          )
          RETURNING id, checkpoint, attempt
      `, w.id, lockSecs, w.cfg.JobName, staleSecs,
      ).Scan(&job.id, &job.checkpoint, &job.attempt)

      if errors.Is(err, sql.ErrNoRows) {
          return nil, ErrNoJobsFound
      }
      if err != nil {
          return nil, fmt.Errorf("claim query: %w", err)
      }
      return &job, nil
}

  ---
5. Property 2 — Liveness: Heartbeat Goroutine

The heartbeat goroutine runs concurrently with the job. Every HeartbeatInterval it updates heartbeat_at. If the update affects 0 rows, the lock was stolen — it cancels the job context via
context.WithCancelCause.

// heartbeatLoop runs as a goroutine alongside the job execution.
// On failure it cancels the job context with ErrLockLost.
func (w *Worker) heartbeatLoop(
ctx context.Context,
cancelWithCause context.CancelCauseFunc,
jobID string,
) {
ticker := time.NewTicker(w.cfg.HeartbeatInterval)
defer ticker.Stop()

      consecutive := 0 // consecutive renewal failures

      for {
          select {
          case <-ctx.Done():
              return
          case <-ticker.C:
              if err := w.renewLock(ctx, jobID); err != nil {
                  if errors.Is(err, context.Canceled) || errors.Is(err, context.DeadlineExceeded) {
                      return
                  }
                  consecutive++
                  slog.Warn("heartbeat renewal failed",
                      "worker_id", w.id,
                      "job_id", jobID,
                      "attempt", consecutive,
                      "error", err,
                  )
                  // Tolerate one transient failure (network blip)
                  // Cancel after two consecutive failures
                  if consecutive >= 2 {
                      slog.Error("heartbeat failed — cancelling job execution",
                          "worker_id", w.id, "job_id", jobID,
                      )
                      cancelWithCause(ErrLockLost)
                      return
                  }
                  continue
              }
              consecutive = 0
              slog.Debug("heartbeat ok", "worker_id", w.id, "job_id", jobID)
          }
      }
}

// renewLock extends the lock TTL. Returns ErrLockLost if this worker no longer owns it.
func (w *Worker) renewLock(ctx context.Context, jobID string) error {
res, err := w.cfg.DB.ExecContext(ctx, `
          UPDATE jobs
          SET
              heartbeat_at = NOW(),
              locked_until = NOW() + make_interval(secs => $1)
          WHERE id       = $2
            AND worker_id = $3
            AND status    = 'running'
      `, w.cfg.LockTTL.Seconds(), jobID, w.id)
if err != nil {
return fmt.Errorf("renew: %w", err)
}
n, err := res.RowsAffected()
if err != nil {
return err
}
if n == 0 {
return ErrLockLost // lock was stolen or job was cancelled externally
}
return nil
}

  ---
6. Properties 3, 4, 5 — Progress, Safe Stop, Resume: Execute Loop

// execute drives the job from its last checkpoint to completion.
// It wires together: heartbeat goroutine, batch processing, checkpointing, and shutdown.
func (w *Worker) execute(parentCtx context.Context, record *claimedJob) error {
// CancelCause lets the heartbeat goroutine cancel with a specific reason
// so we can distinguish "context cancelled externally" from "lock was stolen"
jobCtx, cancel := context.WithCancelCause(parentCtx)
defer cancel(nil)

      // Start heartbeat — keeps the lock alive while we work
      hbDone := make(chan struct{})
      go func() {
          defer close(hbDone)
          w.heartbeatLoop(jobCtx, cancel, record.id)
      }()

      defer func() {
          cancel(nil)  // stop the heartbeat goroutine
          <-hbDone     // wait for clean exit before releasing resources
      }()

      checkpoint := record.checkpoint
      var totalProcessed int64

      for {
          // Property 4 — SAFE STOP:
          // Check before starting each batch. If context was cancelled
          // (SIGTERM, lock loss, or timeout), save checkpoint and exit cleanly.
          if err := jobCtx.Err(); err != nil {
              cause := context.Cause(jobCtx)
              if errors.Is(cause, ErrLockLost) {
                  return ErrLockLost // another worker will resume from the last checkpoint
              }
              // Graceful shutdown requested — checkpoint already saved from last batch
              slog.Info("job paused for shutdown",
                  "job_id", record.id,
                  "processed", totalProcessed,
              )
              return nil
          }

          // Property 5 — RESUME:
          // checkpoint is nil on first run, or the last saved value.
          // The executor resumes exactly where it left off.
          result, err := w.cfg.Executor.Process(jobCtx, checkpoint)
          if err != nil {
              if errors.Is(context.Cause(jobCtx), ErrLockLost) {
                  return ErrLockLost
              }
              return fmt.Errorf("process: %w", err)
          }

          totalProcessed += result.Processed

          // Property 3 — FORWARD PROGRESS VISIBLE:
          // Checkpoint is saved atomically to DB after every batch.
          // Combined with heartbeat renewal to reduce DB round-trips.
          if err := w.saveCheckpoint(jobCtx, record.id, result.Checkpoint); err != nil {
              if errors.Is(context.Cause(jobCtx), ErrLockLost) {
                  return ErrLockLost
              }
              return fmt.Errorf("save checkpoint: %w", err)
          }
          checkpoint = result.Checkpoint

          // Report progress externally (metrics, logs, dashboard)
          w.cfg.OnProgress(record.id, totalProcessed, result.Total, result.Message)

          if result.Done {
              return w.markCompleted(jobCtx, record.id)
          }
      }
}

// saveCheckpoint persists the job's state AND renews the heartbeat in one round-trip.
// Returns ErrLockLost if this worker no longer owns the job.
func (w *Worker) saveCheckpoint(ctx context.Context, jobID string, cp []byte) error {
res, err := w.cfg.DB.ExecContext(ctx, `
          UPDATE jobs
          SET
              checkpoint   = $1,
              heartbeat_at = NOW(),
              locked_until = NOW() + make_interval(secs => $2)
          WHERE id       = $3
            AND worker_id = $4
            AND status    = 'running'
      `, cp, w.cfg.LockTTL.Seconds(), jobID, w.id)
if err != nil {
return fmt.Errorf("save checkpoint: %w", err)
}
n, err := res.RowsAffected()
if err != nil {
return err
}
if n == 0 {
return ErrLockLost
}
return nil
}

  ---
7. Property 6 — Automatic Recovery: Terminal State Transitions

func (w *Worker) markCompleted(ctx context.Context, jobID string) error {
_, err := w.cfg.DB.ExecContext(ctx, `
          UPDATE jobs
          SET
              status       = 'completed',
              completed_at = NOW(),
              worker_id    = NULL,
              locked_until = NULL
          WHERE id       = $1
            AND worker_id = $2
      `, jobID, w.id)
if err != nil {
return fmt.Errorf("mark completed: %w", err)
}
slog.Info("job completed", "job_id", jobID, "worker_id", w.id)
return nil
}

// markFailed transitions the job based on remaining attempts:
//   - attempts exhausted → 'failed' (requires manual intervention or re-queue)
//   - attempts remaining → 'pending' (automatically re-queued for the next worker)
//
// The checkpoint is preserved — the next attempt resumes from where this one stopped.
func (w *Worker) markFailed(ctx context.Context, jobID string, jobErr error) error {
_, err := w.cfg.DB.ExecContext(ctx, `
          UPDATE jobs
          SET
              status       = CASE
                                 WHEN attempt >= max_attempts THEN 'failed'
                                 ELSE 'pending'
                             END,
              error        = $1,
              worker_id    = NULL,
              locked_until = NULL
              -- checkpoint is intentionally NOT reset: next attempt resumes from here
          WHERE id       = $2
            AND worker_id = $3
      `, jobErr.Error(), jobID, w.id)
if err != nil {
return fmt.Errorf("mark failed: %w", err)
}
slog.Error("job failed",
"job_id", jobID,
"error", jobErr,
)
return err
}

  ---
8. Concrete Executor: Database Row Export

package examples

import (
"context"
"database/sql"
"encoding/json"
"errors"
"fmt"
"io"
"os"
)

// ExportCheckpoint is the job-specific progress state.
// The framework stores this as opaque bytes — only the executor interprets it.
type ExportCheckpoint struct {
LastID     int64  `json:"last_id"`      // cursor — resume from here
Exported   int64  `json:"exported"`     // total rows exported so far
TotalRows  int64  `json:"total_rows"`   // estimated total (for progress %)
}

type DBExportExecutor struct {
SourceDB  *sql.DB
OutputPath string
BatchSize int
}

func (e *DBExportExecutor) Process(ctx context.Context, raw []byte) (job.BatchResult, error) {
// Deserialize checkpoint — nil on first run
var cp ExportCheckpoint
if raw != nil {
if err := json.Unmarshal(raw, &cp); err != nil {
return job.BatchResult{}, fmt.Errorf("unmarshal checkpoint: %w", err)
}
}

      // On first run: estimate total for progress reporting
      if cp.TotalRows == 0 {
          _ = e.SourceDB.QueryRowContext(ctx, `SELECT COUNT(*) FROM events`).
              Scan(&cp.TotalRows)
      }

      // Open output file in append mode — idempotency-safe: rows are written
      // only if they pass the cursor, duplicates are impossible
      f, err := os.OpenFile(e.OutputPath, os.O_CREATE|os.O_APPEND|os.O_WRONLY, 0644)
      if err != nil {
          return job.BatchResult{}, fmt.Errorf("open output: %w", err)
      }
      defer f.Close()

      // Fetch next batch using keyset pagination — stable across retries
      rows, err := e.SourceDB.QueryContext(ctx, `
          SELECT id, user_id, type, payload, created_at
          FROM events
          WHERE id > $1
          ORDER BY id ASC
          LIMIT $2
      `, cp.LastID, e.BatchSize)
      if err != nil {
          return job.BatchResult{}, fmt.Errorf("query batch: %w", err)
      }
      defer rows.Close()

      var processed int64
      var lastID int64

      for rows.Next() {
          // Check for cancellation between rows
          if ctx.Err() != nil {
              break
          }
          var row EventRow
          if err := rows.Scan(&row.ID, &row.UserID, &row.Type, &row.Payload, &row.CreatedAt); err != nil {
              return job.BatchResult{}, fmt.Errorf("scan: %w", err)
          }
          if _, err := fmt.Fprintf(f, "%d,%s,%s,%s\n",
              row.ID, row.UserID, row.Type, row.CreatedAt.Format(time.RFC3339),
          ); err != nil {
              return job.BatchResult{}, fmt.Errorf("write row: %w", err)
          }
          lastID = row.ID
          processed++
      }
      if err := rows.Err(); err != nil {
          return job.BatchResult{}, fmt.Errorf("rows: %w", err)
      }

      // Update cursor
      if lastID > 0 {
          cp.LastID = lastID
      }
      cp.Exported += processed

      // If we got fewer rows than batch size, we've reached the end
      done := processed < int64(e.BatchSize)

      cpBytes, err := json.Marshal(cp)
      if err != nil {
          return job.BatchResult{}, fmt.Errorf("marshal checkpoint: %w", err)
      }

      return job.BatchResult{
          Checkpoint: cpBytes,
          Processed:  processed,
          Total:      cp.TotalRows,
          Done:       done,
          Message:    fmt.Sprintf("exported %d / %d rows (last_id=%d)", cp.Exported, cp.TotalRows, cp.LastID),
      }, nil
}

  ---
9. Wiring It All Together

package main

import (
"context"
"database/sql"
"fmt"
"log/slog"
"os"
"os/signal"
"syscall"

      _ "github.com/lib/pq"
)

func main() {
db, err := sql.Open("postgres", os.Getenv("DATABASE_URL"))
if err != nil {
slog.Error("db connect", "err", err)
os.Exit(1)
}
defer db.Close()

      // Seed the job if it doesn't exist
      _, _ = db.Exec(`
          INSERT INTO jobs (id, name, status, max_attempts)
          VALUES ('export-001', 'event-export', 'pending', 5)
          ON CONFLICT (id) DO NOTHING
      `)

      executor := &examples.DBExportExecutor{
          SourceDB:   db,
          OutputPath: "/data/events-export.csv",
          BatchSize:  1000,
      }

      worker := job.NewWorker(job.Config{
          JobName:  "event-export",
          DB:       db,
          Executor: executor,

          HeartbeatInterval: 5 * time.Second,
          LockTTL:           30 * time.Second,
          StaleLockAge:      60 * time.Second,
          PollInterval:      10 * time.Second,

          OnProgress: func(jobID string, processed, total int64, msg string) {
              pct := 0.0
              if total > 0 {
                  pct = float64(processed) / float64(total) * 100
              }
              slog.Info("progress",
                  "job_id", jobID,
                  "message", msg,
                  "pct", fmt.Sprintf("%.1f%%", pct),
              )
          },
      })

      // Graceful shutdown on SIGTERM / SIGINT
      // signal.NotifyContext cancels ctx when signal arrives
      ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM, os.Interrupt)
      defer stop()

      if err := worker.Run(ctx); err != nil {
          slog.Error("worker error", "err", err)
          os.Exit(1)
      }
}

  ---
10. Timing Relationships — How the Properties Hold Together

HeartbeatInterval (5s) < LockTTL (30s) < StaleLockAge (60s)

Timeline of a worker crash:

t=0s:   Worker A claims job, sets heartbeat_at = now()
t=5s:   Worker A renews heartbeat_at
t=10s:  Worker A renews heartbeat_at
t=11s:  Worker A CRASHES — heartbeat stops
t=30s:  locked_until expires (backup safety net)
t=71s:  heartbeat_at is 60s old → StaleLockAge threshold crossed
t=71s:  Worker B's claim query matches (status='running' AND heartbeat_at < now()-60s)
t=71s:  Worker B atomically takes ownership, loads checkpoint from t=10s
t=71s:  Worker B resumes from exactly where Worker A left off

Relationship constraints:

HeartbeatInterval  must be < LockTTL / 2
(ensures renewal happens before lock can expire)

LockTTL            must be > HeartbeatInterval × 2
(tolerates one missed heartbeat without false-positive lock theft)

StaleLockAge       must be > LockTTL
(don't steal until lock has actually expired)
set to 2× LockTTL to be conservative

Checkpoint timing: save after every batch, not every row
batches should be sized so a crash loses < 1s of work

  ---
11. Operational Queries

-- What is the job doing right now?
SELECT
id,
status,
worker_id,
attempt,
heartbeat_at,
NOW() - heartbeat_at            AS heartbeat_age,
checkpoint->>'last_id'          AS cursor,
checkpoint->>'exported'         AS rows_exported,
checkpoint->>'total_rows'       AS total_rows,
ROUND(
(checkpoint->>'exported')::numeric /
NULLIF((checkpoint->>'total_rows')::numeric, 0) * 100, 1
)                               AS pct_complete
FROM jobs
WHERE name = 'event-export';

-- Are any jobs stuck? (heartbeat not renewed for > 2 minutes)
SELECT id, worker_id, heartbeat_at, NOW() - heartbeat_at AS stale_for
FROM jobs
WHERE status = 'running'
AND heartbeat_at < NOW() - INTERVAL '2 minutes';

-- Manually re-queue a permanently failed job
UPDATE jobs
SET status = 'pending', attempt = 0, error = NULL
WHERE id = 'export-001' AND status = 'failed';

-- Force-release a lock (if worker is known dead and you can't wait for StaleLockAge)
UPDATE jobs
SET status = 'pending', worker_id = NULL, locked_until = NULL
WHERE id = 'export-001' AND worker_id = 'worker-uuid-that-is-dead';

  ---
Summary: How Each Property Is Achieved

Property                  Mechanism
──────────────────────────────────────────────────────────────────────────
Single owner              FOR UPDATE SKIP LOCKED in claim query
→ atomic, no two workers claim the same row

Prove liveness            heartbeat_at updated every HeartbeatInterval
→ operator can query staleness; framework steals stale locks

Forward progress          checkpoint column updated after every batch
→ visible in DB at all times via monitoring query

Safe stop                 context.WithCancelCause propagated into Process()
→ executor respects ctx, batch completes, checkpoint saved

Resume after failure      checkpoint loaded on claim, passed to first Process() call
→ executor resumes cursor/offset, no work repeated

Recover if crashed        claim query matches stale running jobs
→ new worker takes lock, loads last checkpoint, continues

✻ Churned for 3m 45s

❯ basic principles for handling validations for api calls across different use cases in industry

⏺ API Validation — Core Principles Across Industry Use Cases

  ---
The Validation Hierarchy

Every API call passes through distinct validation layers. Each layer has a specific responsibility — mixing them creates bugs and poor error messages.

Request arrives
│
▼
┌─────────────────────────────────────────────────┐
│  L1: Transport / Schema Validation               │
│  Is the request well-formed?                     │
│  - Valid JSON/Protobuf/XML                       │
│  - Required fields present                       │
│  - Field types match schema                      │
│  - Size limits respected                         │
│  Returns: 400 Bad Request                        │
└────────────────────────┬────────────────────────┘
│
▼
┌─────────────────────────────────────────────────┐
│  L2: Semantic Validation                         │
│  Do the values make sense in isolation?          │
│  - Email format valid                            │
│  - Date is not in the past                       │
│  - start_date < end_date                         │
│  - Enum value is recognised                      │
│  Returns: 400 Bad Request                        │
└────────────────────────┬────────────────────────┘
│
▼
┌─────────────────────────────────────────────────┐
│  L3: Security Validation                         │
│  Is the caller allowed to do this?               │
│  - Authentication (who are you?)                 │
│  - Authorisation (are you allowed?)              │
│  - Rate limiting                                 │
│  - Replay attack prevention                      │
│  Returns: 401, 403, 429                          │
└────────────────────────┬────────────────────────┘
│
▼
┌─────────────────────────────────────────────────┐
│  L4: Business Rule Validation                    │
│  Does this make sense in the domain?             │
│  - Account has sufficient balance                │
│  - Order status allows cancellation              │
│  - User hasn't exceeded subscription quota       │
│  - Resource exists and belongs to caller         │
│  Returns: 409 Conflict / 422 Unprocessable       │
└────────────────────────┬────────────────────────┘
│
▼
Execute Logic

Why layering matters: An authentication failure should never reach business rule validation. A malformed request should never reach the database. Validate at the earliest layer that has enough context.

  ---
Core Principles

Principle 1: Collect All Errors, Don't Stop at the First

// BAD: stops at first error — user fixes one problem, submits, finds another
func validate(req CreateUserRequest) error {
if req.Email == "" {
return errors.New("email required")
}
if req.Age < 18 {
return errors.New("must be 18 or older")
}
if req.Password == "" {
return errors.New("password required")
}
return nil
}

// GOOD: collects all errors — one round trip to fix everything
type ValidationErrors []FieldError

type FieldError struct {
Field   string `json:"field"`
Code    string `json:"code"`    // machine-readable — client can branch on this
Message string `json:"message"` // human-readable — client can display this
}

func (ve ValidationErrors) Error() string {
msgs := make([]string, len(ve))
for i, e := range ve {
msgs[i] = fmt.Sprintf("%s: %s", e.Field, e.Message)
}
return strings.Join(msgs, "; ")
}

func validate(req CreateUserRequest) error {
var errs ValidationErrors

      if req.Email == "" {
          errs = append(errs, FieldError{"email", "required", "email is required"})
      } else if !isValidEmail(req.Email) {
          errs = append(errs, FieldError{"email", "invalid_format", "email format is invalid"})
      }

      if req.Age < 18 {
          errs = append(errs, FieldError{"age", "below_minimum", "must be 18 or older"})
      }

      if req.Password == "" {
          errs = append(errs, FieldError{"password", "required", "password is required"})
      } else if len(req.Password) < 8 {
          errs = append(errs, FieldError{"password", "too_short", "minimum 8 characters"})
      }

      if len(errs) > 0 {
          return errs
      }
      return nil
}

Exception: For security-sensitive validation (auth checks, rate limits) — fail immediately and return nothing specific. Collect-all is for user-facing input errors, not security rejections.

  ---
Principle 2: Validate at the Boundary, Trust Inside

External input (user, partner API, webhook)  → validate everything, trust nothing
Internal service calls (same trust domain)   → validate structure, trust values
Database reads                               → trust (you wrote it, schema enforces it)

// External boundary: validate thoroughly
func (h *Handler) CreateOrder(w http.ResponseWriter, r *http.Request) {
var req CreateOrderRequest
if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
respondError(w, 400, "INVALID_JSON", "request body is not valid JSON")
return
}
if errs := validateCreateOrder(req); errs != nil {
respondValidationErrors(w, errs)
return
}
// From here: req is trusted
order, err := h.svc.CreateOrder(r.Context(), req)
}

// Internal service method: light validation — trust your own callers
// If an internal caller passes garbage, that's a programming error (panic or assert)
func (s *OrderService) CreateOrder(ctx context.Context, req CreateOrderRequest) (*Order, error) {
// Basic nil/zero guards — not full validation
if req.UserID == "" {
panic("CreateOrder: UserID must not be empty")  // programming error, not user error
}
// Business logic follows
}

  ---
Principle 3: Separate Structural from Business Validation

Structural validation is stateless and fast. Business validation requires I/O (DB reads, service calls). Keep them separate — run structural first, only hit the database if structure is valid.

// Structural: no I/O, runs first
func validateStructure(req TransferRequest) ValidationErrors {
var errs ValidationErrors
if req.FromAccountID == "" {
errs = append(errs, FieldError{"from_account_id", "required", "required"})
}
if req.ToAccountID == "" {
errs = append(errs, FieldError{"to_account_id", "required", "required"})
}
if req.FromAccountID != "" && req.ToAccountID != "" && req.FromAccountID == req.ToAccountID {
errs = append(errs, FieldError{"to_account_id", "same_as_source", "cannot transfer to same account"})
}
if req.Amount.LessThanOrEqual(decimal.Zero) {
errs = append(errs, FieldError{"amount", "must_be_positive", "amount must be greater than zero"})
}
if req.Amount.GreaterThan(decimal.NewFromInt(1_000_000)) {
errs = append(errs, FieldError{"amount", "exceeds_limit", "single transfer limit is 1,000,000"})
}
return errs
}

// Business rules: requires I/O — only called after structural passes
func (s *TransferService) validateBusiness(ctx context.Context, req TransferRequest) error {
from, err := s.accounts.Get(ctx, req.FromAccountID)
if err != nil {
if errors.Is(err, ErrNotFound) {
return ValidationErrors{{"from_account_id", "not_found", "account not found"}}
}
return fmt.Errorf("get source account: %w", err)
}

      if from.Balance.LessThan(req.Amount) {
          return ValidationErrors{{"amount", "insufficient_funds",
              fmt.Sprintf("balance %.2f is less than transfer amount %.2f",
                  from.Balance, req.Amount)}}
      }

      if from.Status != AccountActive {
          return ValidationErrors{{"from_account_id", "account_inactive", "account is not active"}}
      }

      return nil
}

// Handler wires them in order
func (h *Handler) Transfer(w http.ResponseWriter, r *http.Request) {
var req TransferRequest
json.NewDecoder(r.Body).Decode(&req)

      // 1. Structure first — no I/O
      if errs := validateStructure(req); len(errs) > 0 {
          respondValidationErrors(w, errs)
          return
      }

      // 2. Business rules — I/O only if structure is valid
      if err := h.svc.validateBusiness(r.Context(), req); err != nil {
          respondError(w, 422, err)
          return
      }

      // 3. Execute
      h.svc.Transfer(r.Context(), req)
}

  ---
Principle 4: Error Responses Must Be Actionable

// BAD: generic, client can't do anything with this
{
"error": "validation failed"
}

// BAD: exposing internals — security risk + unstable
{
"error": "pq: duplicate key value violates unique constraint \"users_email_key\""
}

// GOOD: structured, machine-readable, human-readable, traceable
{
"error": {
"code": "VALIDATION_FAILED",          // stable code client switches on
"message": "request validation failed", // human summary
"trace_id": "abc-123-def",             // for support lookup
"fields": [                            // field-level detail
{
"field": "email",
"code": "already_in_use",
"message": "this email address is already registered"
},
{
"field": "phone",
"code": "invalid_format",
"message": "phone must be in E.164 format, e.g. +14155552671"
}
]
}
}

HTTP status codes used consistently:

400 Bad Request         Malformed input, schema mismatch, missing required fields
401 Unauthorized        Missing or invalid authentication
403 Forbidden           Authenticated but not authorised
404 Not Found           Resource doesn't exist (or intentionally hidden)
409 Conflict            State conflict (duplicate, already exists, wrong state)
410 Gone                Resource permanently deleted
422 Unprocessable       Structurally valid but fails business rules
429 Too Many Requests   Rate limit hit (include Retry-After header)
500 Internal Error      Never expose details — log internally, return trace ID

  ---
Use Case 1: REST API — Form/Request Validation

// Reusable validator using go-playground/validator for structural checks
// plus custom methods for business rules

type CreateProductRequest struct {
Name        string   `json:"name"        validate:"required,min=1,max=200"`
Price       float64  `json:"price"       validate:"required,gt=0"`
Currency    string   `json:"currency"    validate:"required,oneof=USD EUR GBP"`
Categories  []string `json:"categories"  validate:"required,min=1,max=5,dive,uuid4"`
Description string   `json:"description" validate:"max=5000"`
SKU         string   `json:"sku"         validate:"required,alphanum,max=50"`
}

var validate = validator.New()

func init() {
// Register tag name to use JSON names in error messages
validate.RegisterTagNameFunc(func(fld reflect.StructField) string {
name := strings.SplitN(fld.Tag.Get("json"), ",", 2)[0]
if name == "-" {
return ""
}
return name
})
}

func validateCreateProduct(req CreateProductRequest) ValidationErrors {
var errs ValidationErrors

      // Structural validation via tags
      if err := validate.Struct(req); err != nil {
          var validationErrs validator.ValidationErrors
          if errors.As(err, &validationErrs) {
              for _, e := range validationErrs {
                  errs = append(errs, FieldError{
                      Field:   e.Field(),
                      Code:    validationCode(e.Tag()),
                      Message: humanMessage(e),
                  })
              }
          }
      }

      // Cross-field validation — tags can't express these
      if req.Price > 0 && req.Currency == "" {
          errs = append(errs, FieldError{"currency", "required_with_price",
              "currency is required when price is specified"})
      }

      return errs
}

func validationCode(tag string) string {
codes := map[string]string{
"required": "required",
"min":      "too_short",
"max":      "too_long",
"gt":       "must_be_positive",
"oneof":    "invalid_value",
"uuid4":    "invalid_uuid",
"alphanum": "invalid_format",
}
if code, ok := codes[tag]; ok {
return code
}
return tag
}

  ---
Use Case 2: gRPC — Interceptor-Based Validation

// Validation as a server interceptor — runs before every RPC handler
func ValidationInterceptor(
ctx context.Context,
req interface{},
info *grpc.UnaryServerInfo,
handler grpc.UnaryHandler,
) (interface{}, error) {
// If the request implements Validator, validate it
type Validator interface {
Validate() error
}
if v, ok := req.(Validator); ok {
if err := v.Validate(); err != nil {
var valErrs ValidationErrors
if errors.As(err, &valErrs) {
// Convert to gRPC status with details
st := status.New(codes.InvalidArgument, "validation failed")
br := &errdetails.BadRequest{}
for _, e := range valErrs {
br.FieldViolations = append(br.FieldViolations,
&errdetails.BadRequest_FieldViolation{
Field:       e.Field,
Description: e.Message,
},
)
}
st, _ = st.WithDetails(br)
return nil, st.Err()
}
return nil, status.Errorf(codes.InvalidArgument, "invalid request: %v", err)
}
}
return handler(ctx, req)
}

// Each proto message type implements Validate()
type CreateOrderRequest struct {
UserId  string
Items   []*OrderItem
// ...
}

func (r *CreateOrderRequest) Validate() error {
var errs ValidationErrors
if r.UserId == "" {
errs = append(errs, FieldError{"user_id", "required", "required"})
}
if len(r.Items) == 0 {
errs = append(errs, FieldError{"items", "required", "at least one item required"})
}
for i, item := range r.Items {
if item.Quantity <= 0 {
errs = append(errs, FieldError{
Field:   fmt.Sprintf("items[%d].quantity", i),
Code:    "must_be_positive",
Message: "quantity must be greater than zero",
})
}
}
if len(errs) > 0 {
return errs
}
return nil
}

  ---
Use Case 3: Async / Event-Driven — Schema + Poison Pill Handling

// Events entering an async system need validation before processing
// Invalid events should not retry infinitely — route to dead letter queue

type EventValidator struct {
schemaRegistry SchemaRegistry
dlq            DeadLetterQueue
}

func (v *EventValidator) Process(ctx context.Context, msg RawMessage) error {
// Layer 1: schema validation — is this a valid event structure?
schema, err := v.schemaRegistry.Get(msg.EventType, msg.SchemaVersion)
if err != nil {
// Unknown event type — might be from a newer producer version
// Don't fail — route to DLQ with reason for human inspection
return v.dlq.Send(ctx, msg, DeadLetterReason{
Code:    "UNKNOWN_SCHEMA",
Message: fmt.Sprintf("no schema for %s v%d", msg.EventType, msg.SchemaVersion),
})
}

      if err := schema.Validate(msg.Payload); err != nil {
          // Schema mismatch — retrying won't help, route to DLQ
          return v.dlq.Send(ctx, msg, DeadLetterReason{
              Code:    "SCHEMA_VALIDATION_FAILED",
              Message: err.Error(),
          })
      }

      // Layer 2: business validation — is the event semantically valid?
      event, _ := schema.Unmarshal(msg.Payload)
      if err := v.validateEventSemantics(ctx, event); err != nil {
          var permanent *PermanentError
          if errors.As(err, &permanent) {
              // Will never be valid — DLQ
              return v.dlq.Send(ctx, msg, DeadLetterReason{
                  Code:    "PERMANENT_VALIDATION_FAILURE",
                  Message: err.Error(),
              })
          }
          // Transient (e.g. referenced entity not yet visible) — allow retry
          return fmt.Errorf("transient validation failure: %w", err)
      }

      return v.processor.Handle(ctx, event)
}

Key difference from REST: async messages can't return errors to the sender. Classify failures as:
Retryable:    network blip, dependency temporarily down → retry with backoff
Permanent:    invalid schema, unknown type, business rule always violated → DLQ
Idempotent:   message already processed → ack and discard (check idempotency key)

  ---
Use Case 4: Webhooks — Authenticity + Replay Prevention

// Webhook validation has a unique concern: proving the payload came from who you expect
// and wasn't replayed from a previous request

type WebhookValidator struct {
secret        []byte
maxAgeSeconds int64 // reject events older than this
}

func (v *WebhookValidator) Validate(r *http.Request) error {
// 1. Timestamp check — prevent replay attacks
timestampStr := r.Header.Get("X-Webhook-Timestamp")
if timestampStr == "" {
return &WebhookError{"missing_timestamp", "X-Webhook-Timestamp header required"}
}
ts, err := strconv.ParseInt(timestampStr, 10, 64)
if err != nil {
return &WebhookError{"invalid_timestamp", "X-Webhook-Timestamp must be Unix epoch"}
}
age := time.Now().Unix() - ts
if age < 0 || age > v.maxAgeSeconds {
// Reject both future timestamps AND old timestamps
return &WebhookError{"timestamp_out_of_range",
fmt.Sprintf("timestamp must be within %ds of current time", v.maxAgeSeconds)}
}

      // 2. HMAC signature check — prove payload wasn't tampered with
      body, err := io.ReadAll(r.Body)
      if err != nil {
          return fmt.Errorf("read body: %w", err)
      }
      r.Body = io.NopCloser(bytes.NewBuffer(body)) // restore for handler

      sig := r.Header.Get("X-Webhook-Signature")
      if sig == "" {
          return &WebhookError{"missing_signature", "X-Webhook-Signature header required"}
      }

      // Sign: HMAC-SHA256(secret, timestamp + "." + body)
      mac := hmac.New(sha256.New, v.secret)
      mac.Write([]byte(timestampStr + "."))
      mac.Write(body)
      expected := "sha256=" + hex.EncodeToString(mac.Sum(nil))

      // Constant-time comparison — prevents timing attacks
      if !hmac.Equal([]byte(sig), []byte(expected)) {
          return &WebhookError{"invalid_signature", "signature verification failed"}
      }

      return nil
}

  ---
Use Case 5: Internal Microservice Calls — Trust Levels and Service Identity

// Internal calls don't need full user-facing validation
// but DO need to validate the caller's identity and intent

type InternalMiddleware struct {
allowedServices map[string][]string  // service → allowed operations
}

func (m *InternalMiddleware) Validate(ctx context.Context, req *http.Request) error {
// 1. Service identity via mTLS or service token
serviceID := req.Header.Get("X-Service-ID")
if serviceID == "" {
return errors.New("X-Service-ID required for internal calls")
}

      // 2. Operation-level authorisation
      // Not all services can call all operations
      operation := req.Header.Get("X-Operation")
      allowed, ok := m.allowedServices[serviceID]
      if !ok {
          return fmt.Errorf("unknown service: %s", serviceID)
      }
      if !contains(allowed, operation) {
          return fmt.Errorf("service %s not authorised for operation %s", serviceID, operation)
      }

      // 3. Structural validation still applies
      // Internal services can send bad data due to bugs
      // Validate structure, but error messages can be more technical
      return nil
}

// Don't skip structural validation even for internal callers
// The calling service may have a bug or be running an old version
func (s *UserService) GetUser(ctx context.Context, id string) (*User, error) {
if id == "" {
// Return a clear error — this is a caller bug, not user error
return nil, status.Errorf(codes.InvalidArgument, "user id must not be empty")
}
if !isValidUUID(id) {
return nil, status.Errorf(codes.InvalidArgument, "user id must be a valid UUID, got: %q", id)
}
// ...
}

  ---
Use Case 6: Idempotency Key Validation

// Clients retry on network failure — the server must handle duplicate requests
// without duplicate effects. Idempotency keys are themselves validated.

func (h *Handler) CreateCharge(w http.ResponseWriter, r *http.Request) {
// 1. Validate idempotency key is present and well-formed
iKey := r.Header.Get("Idempotency-Key")
if iKey == "" {
respondError(w, 400, "IDEMPOTENCY_KEY_REQUIRED",
"Idempotency-Key header is required for this operation")
return
}
if len(iKey) > 255 {
respondError(w, 400, "IDEMPOTENCY_KEY_TOO_LONG",
"Idempotency-Key must be 255 characters or fewer")
return
}

      // 2. Check if we've seen this key before
      prev, err := h.store.GetIdempotentResult(r.Context(), iKey)
      if err == nil {
          // Seen before: replay the original response
          if prev.RequestHash != hashRequest(r) {
              // Same key, different request body — this is a client bug
              respondError(w, 422, "IDEMPOTENCY_KEY_REUSED",
                  "Idempotency-Key was previously used with different request parameters")
              return
          }
          // Identical request — return cached response
          w.WriteHeader(prev.StatusCode)
          w.Write(prev.Body)
          return
      }

      // 3. Process and store result atomically
      result, err := h.svc.CreateCharge(r.Context(), req)
      h.store.SaveIdempotentResult(r.Context(), iKey, result)
}

  ---
Security Validation — Across All Use Cases

// 1. Input size limits — before any parsing
func SizeLimitMiddleware(maxBytes int64) func(http.Handler) http.Handler {
return func(next http.Handler) http.Handler {
return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
r.Body = http.MaxBytesReader(w, r.Body, maxBytes)
next.ServeHTTP(w, r)
})
}
}

// 2. Sanitize before storage — context-specific
func sanitize(input string) string {
// Strip null bytes (PostgreSQL injection via \x00)
input = strings.ReplaceAll(input, "\x00", "")
// Normalize unicode (prevent homograph attacks)
input = norm.NFC.String(input)
return strings.TrimSpace(input)
}

// 3. Parameterized queries — never interpolate user input into SQL
// BAD:
query := fmt.Sprintf("SELECT * FROM users WHERE email = '%s'", req.Email)

// GOOD:
query := "SELECT * FROM users WHERE email = $1"
db.QueryRowContext(ctx, query, req.Email)

// 4. Enumeration prevention — don't confirm resource existence to unauthorised users
func (h *Handler) GetOrder(w http.ResponseWriter, r *http.Request) {
order, err := h.svc.GetOrder(r.Context(), orderID)
if err != nil {
if errors.Is(err, ErrNotFound) || errors.Is(err, ErrForbidden) {
// Return the same 404 for both — don't reveal whether the order exists
http.Error(w, "not found", 404)
return
}
}
// Only reach here if authenticated AND authorised AND found
if order.UserID != authenticatedUserID(r) {
http.Error(w, "not found", 404)  // 404 not 403 — don't confirm existence
return
}
}

  ---
Domain Object Validation — Validation at Construction

The most robust approach: make invalid states unrepresentable. Objects can only be constructed in a valid state.

// BAD: struct can be created in any state, validation is scattered
type Order struct {
ID       string
UserID   string
Amount   float64
Currency string
}
// Any code can do: Order{} — missing all required fields, no validation

// GOOD: validation happens at construction — once, at the boundary
type Order struct {
id       string          // unexported — only set through constructor
userID   string
amount   decimal.Decimal
currency Currency
}

// Private field access via methods — never a raw struct
func (o Order) ID()       string          { return o.id }
func (o Order) Amount()   decimal.Decimal { return o.amount }
func (o Order) Currency() Currency        { return o.currency }

// NewOrder is the only way to create a valid Order
func NewOrder(userID string, amount decimal.Decimal, currency string) (Order, error) {
var errs ValidationErrors

      if userID == "" {
          errs = append(errs, FieldError{"user_id", "required", "required"})
      }
      if amount.LessThanOrEqual(decimal.Zero) {
          errs = append(errs, FieldError{"amount", "must_be_positive", "must be > 0"})
      }
      cur, ok := parseCurrency(currency)
      if !ok {
          errs = append(errs, FieldError{"currency", "unsupported",
              fmt.Sprintf("%q is not a supported currency", currency)})
      }
      if len(errs) > 0 {
          return Order{}, errs
      }

      return Order{
          id:       uuid.New().String(),
          userID:   userID,
          amount:   amount,
          currency: cur,
      }, nil
}

// Every Order that exists is valid — no guards needed inside domain methods
func (o Order) ApplyDiscount(pct decimal.Decimal) Order {
// No need to check if o is valid — it can't be invalid
return Order{
id:       o.id,
userID:   o.userID,
amount:   o.amount.Mul(decimal.NewFromInt(1).Sub(pct.Div(decimal.NewFromInt(100)))),
currency: o.currency,
}
}

  ---
Summary: Principle Reference

Principle                         Rule
────────────────────────────────────────────────────────────────────────
Layer ordering                    Schema → Semantic → Security → Business
Never reach business layer with malformed input

Collect all errors                For user-facing input: gather all field errors
For security: fail fast, return nothing specific

Validate at the boundary          Thorough at external entry points
Structural-only for internal trusted callers

Separate structural from business Structural = stateless, no I/O, runs first
Business = requires I/O, runs second

Domain construction               Invalid objects cannot be created
Validation runs once at NewXxx(), not on every use

Error responses                   Always: code (machine) + message (human) + trace_id
Never: stack traces, SQL errors, internal paths

Async / events                    Classify as retryable vs permanent
Permanent failures → dead letter queue, not infinite retry

Webhooks                          Validate timestamp (replay) + HMAC (authenticity)
Constant-time comparison for signatures

Security inputs                   Size limits before parsing
Parameterized queries always
Same error for not-found vs forbidden (enumeration)

Idempotency                       Validate key presence + format
Reject same key with different body
Replay identical response for duplicate keys