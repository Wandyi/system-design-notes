# 12 · Mutations, Deletes, Updates, and TTL

ClickHouse is append-mostly. Any "update" or "delete" works against the immutable-parts grain. The interview will probe: how do you handle deletes in an immutable engine, what's the difference between mutations and lightweight deletes, and when does TTL save you.

## 12.1 The mutation model

`ALTER TABLE ... UPDATE` and `ALTER TABLE ... DELETE` are **mutations**: heavy background operations that rewrite affected parts.

```sql
ALTER TABLE events
UPDATE event = 'click'
WHERE user_id = 42 AND ts > '2026-01-01';

ALTER TABLE events
DELETE WHERE ts < now() - INTERVAL 90 DAY;
```

What happens:
1. The mutation is enqueued in `system.mutations` (Keeper if Replicated).
2. For each *affected* part, ClickHouse rewrites the part with the change applied.
3. The new part replaces the old; old is removed.
4. Each replica applies the mutation independently.

### Cost

- **A single ALTER UPDATE touching a column on N parts** rewrites all N parts. If N parts total 1 TB, you rewrite 1 TB.
- The rewrite uses the merger thread pool; running too many simultaneously throttles inserts.
- **You cannot mutate ORDER BY / PRIMARY KEY columns.** They define the physical sort; ClickHouse refuses.

### Async / sync

Default is async. The ALTER returns immediately; the work happens in background.

```sql
ALTER TABLE events DELETE WHERE ... SETTINGS mutations_sync = 1;  -- wait for local replica
ALTER TABLE events DELETE WHERE ... SETTINGS mutations_sync = 2;  -- wait for all replicas
```

`mutations_sync = 2` is the "make me actually wait" mode; use it in scripted DDL when you depend on completion.

### Monitoring

```sql
SELECT
    database, table, mutation_id,
    create_time, parts_to_do, is_done, latest_failed_part, latest_fail_reason
FROM system.mutations
WHERE NOT is_done
ORDER BY create_time;
```

Look for stuck mutations (`is_done = 0`, `latest_fail_reason != ''`). Common cause: bad expression (NULL handling, type mismatch).

### Killing a mutation

```sql
KILL MUTATION WHERE mutation_id = '...' AND table = 'events';
```

The partially-rewritten parts are abandoned. The mutation is removed from the queue. Safe.

## 12.2 Lightweight DELETE

```sql
DELETE FROM events WHERE user_id = 42;
```

Available since 22.8. Doesn't rewrite parts. Instead:
- Marks rows with `_row_exists = 0` (a hidden column that defaults to 1).
- All subsequent queries automatically filter rows where `_row_exists = 0`.
- A background cleanup eventually rewrites parts to physically drop the marked rows.

### When to prefer over ALTER DELETE

- Frequent small deletes (GDPR-style).
- Tactical fixes that you want to be visible immediately without waiting for part rewrites.
- Mass deletes where you'd rather amortize the rewrite cost.

### Caveats

- The `_row_exists` filter is automatically applied — but it costs a tiny per-row check.
- Until cleanup happens, the storage is still there.
- Lightweight DELETE doesn't update materialized views' targets. If the MV is `count()`, the count is now wrong relative to the source. Plan accordingly.
- `OPTIMIZE TABLE ... CLEANUP` (or specific settings) can be triggered to materialize physical deletion.

## 12.3 Lightweight UPDATE (Cloud-first; OSS coming)

```sql
UPDATE events SET event = 'click' WHERE user_id = 42;
```

Writes a small "patch" part with the new values for affected rows. Reads merge the patch in transparently.

Same caveats as lightweight DELETE plus:
- Many small patches accumulate; periodic compaction folds them in.
- Not always faster than ALTER UPDATE for very large change sets; the patch can become large.

## 12.4 Alternatives to mutations (the "engine choice" answers)

For most "update" or "delete" requirements you should *not* use mutations. Pick an engine that handles it at merge time:

| Requirement | Engine + pattern |
|-------------|-------------------|
| "Latest state" per entity | **ReplacingMergeTree** + version column |
| "Add then cancel" state events | **CollapsingMergeTree** with Sign |
| "Latest state" but state changes arrive out of order | **VersionedCollapsingMergeTree** |
| Sum/count of repeated keys | **SummingMergeTree** |
| Pre-aggregated complex stats | **AggregatingMergeTree** + MV |
| Time-bound: drop after N days | **TTL DELETE** (per-row or per-part) |
| Time-bound: move cold to S3 | **TTL TO VOLUME / DISK** |
| GDPR delete-by-user | Lightweight DELETE |
| Schema change (add column) | `ALTER ADD COLUMN` (cheap; no rewrite) |
| Change column type | `ALTER MODIFY COLUMN` (expensive, full rewrite) |

## 12.5 TTL — automatic deletes and moves

The most-loved data-lifecycle feature.

### TTL DELETE

```sql
CREATE TABLE events (
    ts        DateTime,
    user_id   UInt64,
    event     String
) ENGINE = MergeTree
ORDER BY (ts, user_id)
TTL ts + INTERVAL 90 DAY;
```

When a part's `max(ts) + 90 days < now()`, the entire part is dropped at merge time. **TTL DELETE works at part granularity**, so partition-by-time is what makes it efficient.

### Per-column TTL

```sql
column_name UInt64 TTL ts + INTERVAL 30 DAY
```

After 30 days, the column is reset to default. Useful for redacting expensive fields while keeping the row.

### TTL TO DISK / VOLUME

```sql
TTL ts + INTERVAL 30 DAY TO DISK 'cold',
    ts + INTERVAL 365 DAY DELETE
```

Move old parts to a cheaper disk (e.g., S3), drop after a year. Tiered storage in one declarative pattern.

### TTL with GROUP BY (aggregating TTL)

```sql
TTL ts + INTERVAL 30 DAY GROUP BY id SET sum_x = sum(x)
```

Roll up old data into aggregates rather than dropping it. Niche but powerful for retention with summary.

### TTL with WHERE

```sql
TTL ts + INTERVAL 30 DAY DELETE WHERE event_type = 'debug'
```

Drop only matching rows. Implemented by rewriting; more expensive than whole-part drops.

### TTL gotchas

- `TTL` is enforced at **merge time**. If no merge happens, no TTL runs. `OPTIMIZE TABLE ... FINAL` forces merges (don't do this on hot tables).
- `MATERIALIZE TTL` is the explicit "apply now" command:
  ```sql
  ALTER TABLE events MATERIALIZE TTL;
  ```
- TTL changes via `ALTER TABLE MODIFY TTL`. Need to `MATERIALIZE TTL` to apply to existing parts.

## 12.6 DROP PARTITION — the fastest delete

```sql
ALTER TABLE events DROP PARTITION '202404';
```

Removes an entire partition's parts. O(small) — just deletes directories. **This is the cheapest "delete" by orders of magnitude.** Design partitions to align with deletion granularity (often monthly).

For Replicated tables, runs on all replicas. Confirm with `system.replication_queue`.

## 12.7 Truncate and detach

```sql
TRUNCATE TABLE events;       -- delete all parts; table remains
DETACH TABLE events;         -- mark the table inactive; data preserved
ATTACH TABLE events;         -- bring back
```

`DETACH PARTITION 'X'` is the partition-level version — move the partition to `detached/` directory; later re-attach or delete.

## 12.8 Schema migrations

### Cheap (no rewrite)

- `ALTER TABLE ... ADD COLUMN c T DEFAULT ...` — adds a column. Old parts get the default at read time.
- `ALTER TABLE ... DROP COLUMN c` — just removes; old data stays in part files but is ignored.
- `ALTER TABLE ... COMMENT COLUMN`.
- `ALTER TABLE ... MODIFY COMMENT`.
- `ALTER TABLE ... ADD INDEX / DROP INDEX` (with optional MATERIALIZE).
- `ALTER TABLE ... MODIFY ORDER BY` — only if extending (adding columns at the end).
- `ALTER TABLE ... MODIFY SETTING`.
- `ALTER TABLE ... ADD PROJECTION` (then MATERIALIZE).

### Expensive (full rewrite of parts)

- `ALTER TABLE ... MODIFY COLUMN c NEW_TYPE` — type change rewrites all parts.
- `ALTER TABLE ... MATERIALIZE COLUMN` after changing default.

### Pattern: change ORDER BY (which you can't do directly)

You can't change the existing ORDER BY in a meaningful way. Pattern:

1. Create `events_new` with the desired ORDER BY.
2. `INSERT INTO events_new SELECT * FROM events;`
3. Switch reads (and writes) to `events_new`.
4. `RENAME TABLE events TO events_old, events_new TO events;`
5. Drop `events_old` after grace period.

For ReplicatedMergeTree, do this on each shard; the Distributed table picks up the new local table automatically.

## 12.9 Worked example — "we need to update one user's data per GDPR request"

Bad answer: `ALTER TABLE events UPDATE SET email = '' WHERE user_id = 42` — rewrites every part touching that user.

Better:
1. **Lightweight DELETE** + insert a redacted row (if you must keep history with redaction):
   ```sql
   DELETE FROM events WHERE user_id = 42;
   INSERT INTO events SELECT *, '' AS email FROM events WHERE user_id = 42;
   ```
   Wait, that's circular. Better:
   ```sql
   -- step 1: capture data needed for an audit
   SELECT * INTO TABLE gdpr_audit FROM events WHERE user_id = 42;
   -- step 2: delete
   DELETE FROM events WHERE user_id = 42;
   -- step 3: optionally re-insert redacted (without PII)
   INSERT INTO events SELECT ts, user_id, event, '' AS email, ... FROM gdpr_audit;
   ```

If your schema has PII in one column and rest is anonymous, **drop the PII column** entirely (declare it with TTL on a per-column basis or `ALTER DROP COLUMN`).

If the deletion volume is huge, **partition-aligned drops** (DROP PARTITION) are vastly cheaper than per-row DELETE.

## 12.10 ON CLUSTER, distributed mutations

```sql
ALTER TABLE events ON CLUSTER my_cluster DELETE WHERE user_id = 42;
```

Distributes the mutation across all nodes. Each node applies locally.

For Replicated tables, the mutation enters Keeper's mutation log; each replica applies in order.

## 12.11 OPTIMIZE TABLE

`OPTIMIZE TABLE ... FINAL` forces merges to completion, including dedup for ReplacingMergeTree etc.

**Don't run on a hot table.** It blocks merging of new parts, can take hours, and uses heavy CPU/IO.

Exception: you've just done a one-time bulk re-ingest and want everything compacted. Even then, `OPTIMIZE TABLE ... DEDUPLICATE` (without FINAL) or partition-by-partition optimization is safer.

## 12.12 Must-internalize

- Mutations rewrite parts. Expensive but correct. Async by default; use `mutations_sync = 2` to wait.
- Lightweight DELETE marks rows; cheap and immediate.
- Lightweight UPDATE writes patches; Cloud-first.
- Prefer engine-level alternatives (Replacing/Aggregating/Collapsing) over runtime mutations.
- TTL DELETE / MOVE for lifecycle; works at part granularity, so partition by time.
- DROP PARTITION is the cheapest delete; design partitions for it.
- Schema migrations: ADD/DROP COLUMN cheap, MODIFY type expensive, ORDER BY changes require a new table.
- Avoid OPTIMIZE TABLE FINAL on production tables.

---

## Sources

- [Mutations guide — official](https://clickhouse.com/docs/guides/developer/mutations)
- [Handling updates and deletes in ClickHouse](https://clickhouse.com/blog/handling-updates-and-deletes-in-clickhouse)
- [Lightweight delete (OSS)](https://clickhouse.com/docs/sql-reference/statements/delete)
- [Lightweight UPDATE / SharedMergeTree perf blog](https://clickhouse.com/blog/clickhouse-cloud-boosts-performance-with-sharedmergetree-and-lightweight-updates)
- [TTL operations](https://clickhouse.com/docs/sql-reference/statements/alter/ttl)
- [MODIFY (ADD) TTL — Altinity KB](https://kb.altinity.com/altinity-kb-queries-and-syntax/ttl/modify-ttl/)
