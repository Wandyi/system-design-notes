# 4 · Data Types — Pick the Right One or Pay 10×

Type choice in ClickHouse drives storage cost, compression, scan speed, and index effectiveness. Wrong types are the easiest way to make a fast database slow.

## 4.1 Integers

| Type | Bytes | Range |
|------|-------|-------|
| Int8 / UInt8 | 1 | -128..127 / 0..255 |
| Int16 / UInt16 | 2 | ~±32K / 0..65K |
| Int32 / UInt32 | 4 | ~±2.1B / 0..4.3B |
| Int64 / UInt64 | 8 | ±9.2E18 / 0..1.8E19 |
| Int128 / Int256 / UInt128 / UInt256 | 16 / 32 | for hashes, blockchain |

**Rule**: pick the smallest that fits *with future growth headroom*. UserID UInt64 if you might exceed 4B; event_id UUID if global; bytes UInt32 unless individual values exceed 4GB.

`UInt32` for unix-second timestamps will overflow in 2106; use `DateTime` instead (also 4 bytes).

## 4.2 Decimal — money and exact arithmetic

```sql
price       Decimal64(2)         -- 18 digits, 2 after the point
volume      Decimal128(8)        -- 38 digits, 8 after the point
```

Don't use Float for money. Float for measurements, Decimal for accounting / billing.

## 4.3 Floats

`Float32`, `Float64`. Standard IEEE. For measurements (metrics, sensor data). Compresses *much* worse than integers; use Gorilla codec for time-series floats.

## 4.4 Strings

- **String** — variable-length, UTF-8 by convention. No length limit (well, 1 GiB practical).
- **FixedString(N)** — exactly N bytes; padded if shorter. Rarely useful except for hashes / IDs of known size.

### LowCardinality — the most important string optimization

```sql
country  LowCardinality(String)
```

Stores the column as a dictionary of distinct values + a small integer index per row. Pays for itself when the column has < a few million distinct values.

- 10-20× smaller on real-world enum-like columns ("US", "CA", "DE"...).
- Faster grouping, faster filtering.
- Free `IN`-based filter speedups.

Use for: country, status, browser, OS, event_type, log level, region, service-name, environment.

Don't use for: random hashes (cardinality near the row count), free-form text bodies, URLs.

There's also `LowCardinality(Nullable(String))` — works but adds an extra null path.

### Enum — fixed set, even smaller

```sql
status  Enum8('active'=1, 'inactive'=2, 'banned'=3)
```

Stored as the integer. Fastest filter / group on a known small set. Cost: schema change to add values.

Modern preference: `LowCardinality(String)` is usually friendlier; Enum only when you really want type-system enforcement.

## 4.5 Date and time

| Type | Bytes | Range / precision |
|------|-------|--------------------|
| Date | 2 | 1970-01-01 .. 2149-06-06 (day) |
| Date32 | 4 | 1900-01-01 .. 2299-12-31 (day) |
| DateTime | 4 | unix-second; up to 2106 |
| DateTime64(p, [tz]) | 8 | sub-second precision, ~year 2299; e.g., DateTime64(3) = milliseconds |

**Timezone**: store in UTC; carry timezone in the column type (`DateTime('UTC')`) or apply at query time (`toTimezone(ts, 'America/New_York')`). Mixing zones is a frequent bug.

**Codecs**: `DateTime CODEC(Delta, ZSTD(1))` for time-series; `DateTime64(3) CODEC(DoubleDelta, ZSTD)` for high-frequency with sub-second.

## 4.6 UUID

16 bytes. `UUID` type. Default codec is fine since UUIDs are random.

If you can choose **sequential** IDs (ULID, snowflake), prefer those — they compress and sort better. UUIDs disable Delta and similar; their entropy is near max.

## 4.7 IPv4 and IPv6

```sql
src_ip  IPv4   -- stored as UInt32 internally
dst_ip  IPv6   -- 16 bytes
```

Functions like `IPv4StringToNum`, `IPv6CIDRToRange`, `isIPAddressInRange` work on these types.

## 4.8 Arrays

```sql
tags  Array(String)
```

Stored as two parallel columns: an offsets column and a flat values column. Operations:
- `has(tags, 'sport')`, `arrayElement(tags, 1)`, `arrayMap(x -> ..., tags)`, `arrayFilter(x -> ..., tags)`.
- `arrayJoin(tags)` — explodes into rows (like Postgres `unnest`).

Very efficient. Hugely useful for sparse fields, multi-value tags, label sets.

## 4.9 Tuples and Maps

```sql
coords  Tuple(Float64, Float64)
attrs   Map(LowCardinality(String), String)
```

**Tuple**: positional, fixed schema; cheap. Use for paired values.

**Map**: variable schema, more expensive. Each row has its own keys. Useful for event metadata that varies per row. Stored internally as two arrays (`Map.keys`, `Map.values`) so you can use array functions on it.

## 4.10 Nested

```sql
nested_field Nested(
    name  String,
    value Float64
)
```

Two parallel arrays under the hood (`nested_field.name`, `nested_field.value`). Insert via parallel arrays:
```sql
INSERT INTO t (nested_field.name, nested_field.value) VALUES (['a','b'], [1,2]);
```

When to use vs. `Map`:
- `Nested` if the keys are a fixed set across rows.
- `Map` if the keys are open-ended.

## 4.11 JSON / Dynamic / Variant (production in 25.x)

The new semi-structured trio.

### Variant — discriminated union

```sql
val Variant(UInt64, String, Array(UInt64))
```

A column whose value at each row is one of the listed types. Cheaper than serializing JSON. Useful for heterogeneous fields with a small known type set.

### Dynamic — open type

```sql
val Dynamic
```

Variant with unbounded types — store anything; ClickHouse tracks which types appeared. Useful for "we don't know the schema yet" columns.

### JSON — fully typed semi-structured

```sql
data JSON
```

Parses JSON into a columnar form where each path becomes a subcolumn with inferred type. Subcolumns can be used in ORDER BY, in skip indexes, and benefit from columnar storage / compression.

You can hint known paths to keep them efficient:
```sql
data JSON(user_id UInt64, country LowCardinality(String))
```

This combines structured fields (the hints) with arbitrary additional JSON (other paths discovered automatically).

The old `Object('json')` type is **deprecated**; migrate to `JSON`.

## 4.12 Geo

- `Point` = `Tuple(Float64, Float64)` — long/lat.
- `Ring` = `Array(Point)`.
- `Polygon` = `Array(Ring)`.
- `MultiPolygon` = `Array(Polygon)`.

Spatial functions: `pointInPolygon`, `geoDistance`, `S2`-prefixed functions, `H3`-prefixed functions for hexagonal grids.

## 4.13 Nullable — handle with care

```sql
score  Nullable(UInt32)
```

Adds a parallel "is null" mask column. Costs ~12 bytes per row at minimum + breaks some optimizations (e.g., LowCardinality can wrap Nullable but path is slower).

**Rule**: prefer a sentinel value (0, empty string, `1970-01-01`) over `Nullable`. Use `Nullable` only when the distinction "no value" vs. "zero value" matters semantically.

## 4.14 Special string types

- **UUID** (16 bytes), **IPv4** / **IPv6** as above.
- **String** with hex-encoded data — consider `FixedString(N)` if length is constant.

## 4.15 Type modifiers

- `LowCardinality(T)` — dictionary-encode.
- `Nullable(T)` — wrap to allow NULL.
- `Array(T)`, `Tuple(...)`, `Map(K,V)`, `Nested(...)` — composites.
- Codec: `T CODEC(...)` — see [05](05-codecs-and-compression.md).

## 4.16 Common type-selection mistakes (corner cases the interview will probe)

### Storing dates as `String`

Doubles+ storage, breaks date functions, breaks Delta compression. Always use `Date`/`DateTime`/`DateTime64`.

### `String` for IPs

Use `IPv4` / `IPv6` — 4 / 16 bytes vs. 15+ bytes; built-in network functions.

### `Float` for money

Use `Decimal*`.

### `String` for categorical with low cardinality

Wrap in `LowCardinality`.

### `Nullable` everywhere

A team migrating from Postgres often makes everything Nullable. Reset that habit: use sentinels.

### `Map` when keys are fixed

If you know the keys, `Nested` (or just multiple typed columns) is cheaper.

### `JSON` (new) for known schemas

If you know the schema, declare typed columns. JSON type is for unknown / open schemas. Otherwise you pay parse overhead for nothing.

### Wide ID (`UInt64`) when `UInt32` fits

Twice the bytes for nothing. Look at the distribution; UInt32 holds 4.3B.

### `String` for booleans

Use `UInt8` (0/1) or `Bool` (alias for UInt8 in newer versions).

## 4.17 Sizing example

Pricing event schema, raw vs. optimized:

```sql
-- BAD: 80–100 bytes/row uncompressed
CREATE TABLE bad_events (
    ts      String,
    user    String,
    country String,
    event   String,
    price   Float64,
    is_paid String
);

-- GOOD: 25-35 bytes/row uncompressed; ~3-5 bytes after codecs
CREATE TABLE good_events (
    ts       DateTime CODEC(Delta, ZSTD(1)),
    user     UInt64,
    country  LowCardinality(String),
    event    LowCardinality(String),
    price    Decimal64(2),
    is_paid  UInt8
);
```

The difference compounds: at 1B rows/day, the bad schema costs ~30 GB/day raw vs. 3-5 GB for the good one. Over a year: 1 TB vs. 11 TB.

## 4.18 Must-internalize

- Pick the smallest int that fits with headroom; `UInt32` over `UInt64` when possible.
- `LowCardinality(String)` for any column with < a few million distinct values.
- `Date` / `DateTime` / `DateTime64`, never `String`, for time.
- `Decimal*` for money.
- `IPv4` / `IPv6` for IPs.
- Avoid `Nullable` unless you really need NULL semantics.
- `JSON` / `Dynamic` / `Variant` for genuinely unknown schemas; typed columns otherwise.
- Composite types are first-class and cheap: prefer arrays/maps/nested over string-encoding lists.

---

## Sources

- [Data types — official](https://clickhouse.com/docs/sql-reference/data-types)
- [LowCardinality](https://clickhouse.com/docs/sql-reference/data-types/lowcardinality)
- [JSON type — official](https://clickhouse.com/docs/sql-reference/data-types/json)
- [How we built a new JSON data type — blog](https://clickhouse.com/blog/a-new-powerful-json-data-type-for-clickhouse)
- [ClickHouse Native JSON Support 2026 (substack)](https://dataanalyticsguide.substack.com/p/clickhouse-native-json-support-2026)
