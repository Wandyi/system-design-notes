# 10 · Joins, Lookups, Subqueries

Druid's join story is intentionally narrow: **broadcast hash joins only, no shuffle**. The trade-off is the cost of full SQL ergonomics for the gain of consistent sub-second latency. A staff engineer needs to know exactly what works, what doesn't, and the alternatives.

The contrast with ClickHouse is sharp: CH has 6 join algorithms including shuffle (grace_hash, sorting merge). Druid leans on **broadcast** + **lookup** + **denormalize** as the three answers.

## 10.1 The three real answers (in order of preference)

1. **Lookup** — for tiny reference data (a few thousand to millions of rows). Loaded into Broker memory, consulted inline.
2. **Broadcast hash join** — for small to medium right-hand sides. Broker materializes the small side and sends to Historicals.
3. **Denormalize** — for everything else. Pre-join at ingest time via MSQ.

Use a real JOIN only when none of the above fits.

## 10.2 Lookups

A **lookup** is an in-memory key-value table preloaded on every Broker (and optionally on Historicals).

### Defining a lookup

```sql
INSERT INTO sys.lookups VALUES (
  '__default', 'country_lookup',
  '{"type":"map","map":{"US":"United States","CA":"Canada","DE":"Germany"}}'
);
```

Or via the Coordinator API with a more dynamic source (JDBC, Kafka, periodic refresh from URL).

### Lookup types

- **Map** — static; defined in the spec.
- **JDBC** — refreshed periodically from a relational DB.
- **URI** — refreshed from an HTTP-accessible file.
- **Kafka** — keep current via consuming a compacted Kafka topic.

### Using a lookup

```sql
SELECT
  country_code,
  LOOKUP(country_code, 'country_lookup') AS country_name,
  count(*)
FROM events
GROUP BY 1, 2;
```

This is **0-cost**: the lookup runs inline on each Historical's segment scan. No JOIN materialized, no Broker fan-out.

You can also `JOIN` a lookup:

```sql
SELECT e.country_code, l.k AS code, l.v AS name, count(*)
FROM events e LEFT JOIN lookup.country_lookup l ON e.country_code = l.k
GROUP BY 1, 2, 3;
```

Same plan internally.

### Lookup limits

- Loaded into **Broker memory** entirely. Don't put millions of rows in a lookup unless your Broker has the heap.
- Single-column → single-column. Multi-column lookups via composite keys.
- Eventually consistent — refresh interval determines staleness.

Compare to ClickHouse's **dictionary** — the same idea, the same role.

## 10.3 Broadcast hash join

For right-hand sides that won't fit in a lookup but are small enough to broadcast:

```sql
SELECT e.country, p.partner_name, sum(e.revenue)
FROM events e
LEFT JOIN partners p ON e.partner_id = p.partner_id
WHERE e.__time >= TIMESTAMP '2026-05-17'
GROUP BY 1, 2;
```

What happens:
1. Broker runs the SELECT on `partners`, materializes as an **inline datasource** (an in-memory table).
2. Broker broadcasts that inline table to every Historical that owns relevant `events` segments.
3. Each Historical runs the join locally: builds a hash table on `partner_id`, scans events, probes.
4. Per-segment results flow back; Broker merges.

Cost: the inline table size × number of Historicals. If `partners` is 100K rows × 100 Historicals = significant per-query overhead. Use **lookups** if `partners` is < ~10K rows.

### Limits

- `druid.server.http.maxSubqueryRows` caps the inline datasource size.
- Right side must be **small** relative to left.
- No nested joins more complex than the planner can flatten.

## 10.4 Denormalize at ingest (the staff answer)

If you're joining the same way every query, **pre-join at ingest**:

```sql
REPLACE INTO events_enriched
  OVERWRITE WHERE __time >= TIMESTAMP '2026-05-17'
                AND __time <  TIMESTAMP '2026-05-18'
SELECT
  TIME_PARSE(e.ts)                            AS __time,
  e.country,
  e.user_id,
  e.event_type,
  e.revenue,
  LOOKUP(e.country, 'country_lookup')         AS country_name,
  LOOKUP(e.partner_id, 'partner_lookup')      AS partner_name
FROM TABLE(EXTERN(...)) e
PARTITIONED BY DAY
CLUSTERED BY country, partner_name;
```

The output is a wider segment but every query is a single-table scan. The classic Druid pattern.

If the dimension is too big for a lookup, use a JOIN inside the MSQ INSERT (one-time cost; the output is denormalized).

## 10.5 IN-filter and semi-join optimization

A common idiom:

```sql
SELECT country, count(*) FROM events
WHERE __time >= TIMESTAMP '2026-05-17'
  AND user_id IN (SELECT user_id FROM premium_users)
GROUP BY 1;
```

The Broker:
1. Runs the subquery to produce a list of user_ids.
2. Materializes as an inline datasource.
3. Pushes down a `user_id IN (..)` filter to each Historical.
4. Each Historical uses the bitmap for `user_id` (if it exists) for fast filtering.

If the IN-list is small (a few thousand), this is fast. If large (millions), the Broker may spill or fail; use MSQ instead.

## 10.6 Subqueries materialize

A subquery in `FROM` is run on the Broker and materialized. Limited to `druid.server.http.maxSubqueryRows` (default ~100K).

For large intermediates, **use MSQ** which can shuffle intermediates across workers via disk:

```sql
INSERT INTO results
SELECT ...
FROM events e
JOIN (SELECT user_id, FIRST_VALUE(country) ... FROM events GROUP BY user_id) u
  ON e.user_id = u.user_id;
```

## 10.7 ASOF-style joins

Druid doesn't have `ASOF JOIN` as a first-class operator (unlike ClickHouse). Workaround:

```sql
SELECT
  e.ts, e.user_id,
  (SELECT MAX(s.start_ts)
   FROM sessions s
   WHERE s.user_id = e.user_id AND s.start_ts <= e.ts) AS session_start
FROM events e;
```

Slow in pure SQL on Druid. Better: **denormalize** session_id into events at ingest time via MSQ.

## 10.8 Cross-datasource joins

A join across two datasources:

```sql
SELECT a.country, sum(a.revenue), sum(b.payouts)
FROM events a
JOIN payouts b ON a.country = b.country AND a.__time = b.__time
WHERE a.__time >= TIMESTAMP '2026-05-17'
GROUP BY 1;
```

Broker runs `events` query and `payouts` query separately, joins on the merged results. Memory-bounded.

Use cases: limited. Almost always denormalize.

## 10.9 Anti-patterns

- **Big-table joins on the query path.** Either materialize the join at ingest or use a lookup.
- **JOIN on high-cardinality dim without filter on the right side.** Broadcast becomes huge.
- **Triple JOIN.** Druid's planner flattens but the cost compounds.
- **Subquery returning millions of rows.** Use MSQ or pre-aggregate.
- **Joining streaming + dimension that updates.** Lookups don't see live updates without configured refresh.

## 10.10 Compare with ClickHouse

| | ClickHouse | Druid |
|--|------------|-------|
| Algorithms | 6 (hash, parallel_hash, grace_hash, partial_merge, full_sorting_merge, direct) | Broadcast hash only |
| Shuffle | grace_hash, sorting merge | None (use MSQ for shuffle-style joins) |
| Dictionary equivalent | `Dictionary` + `dictGet` | Lookup + `LOOKUP()` |
| ASOF JOIN | First-class | Workaround |
| Big-on-big join | grace_hash | Denormalize via MSQ |
| Co-shard joins | Yes (matching shard keys) | N/A (no sharding concept) |
| Star schema | Comfortable | Use lookups + denormalize |
| Snowflake schema | Possible | Painful — denormalize |

## 10.11 Worked example — picking the right approach

Scenario: "Show me events by country name (full name, not code)" — `events` has `country_code`, we have a `countries` table mapping code → name.

| Approach | When |
|----------|------|
| **Lookup** | `countries` has < 10K rows; updated rarely. **Best.** |
| Broadcast hash join | `countries` has 100K-1M rows; updated rarely. OK. |
| **Denormalize at ingest** | We always show country name; never need the raw code. **Best for repeated query patterns.** |
| Real JOIN at query time | One-off, exploratory. |

Senior interview answer: "Lookup if it fits in Broker heap; denormalize at ingest if the join shape is fixed across queries." The denormalize option pays once at ingest and is free at query time forever.

## 10.12 Must-internalize

- Druid joins are **broadcast hash only**. No shuffle on the query path.
- **Lookups** for small reference data — in-memory on Broker, free at query time.
- **Broadcast hash** for small-to-medium right sides.
- **Denormalize at ingest** via MSQ for repeated join patterns.
- IN-filter optimization for semi-joins.
- Subqueries materialize on Broker (memory-bounded); use MSQ for large intermediates.
- No ASOF JOIN; workaround via subquery or denormalize.

---

## Sources

- [Joins — official](https://druid.apache.org/docs/latest/querying/joins/)
- [Lookups](https://druid.apache.org/docs/latest/querying/lookups/)
- [Datasource types](https://druid.apache.org/docs/latest/querying/datasource/)
- [Query execution](https://druid.apache.org/docs/latest/querying/query-execution/)
