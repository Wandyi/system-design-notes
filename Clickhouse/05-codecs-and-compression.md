# 5 · Codecs and Compression

Compression is half the speed story: less bytes = less I/O = less decompression = faster scans. ClickHouse has a two-layer codec model: type-specific codecs (Delta, T64, Gorilla, FPC) followed by a generic compressor (LZ4, ZSTD).

## 5.1 The codec pipeline

```sql
ts DateTime CODEC(Delta, ZSTD(3))
```

Reading left to right: apply Delta first (computes deltas between consecutive values), then ZSTD level-3 over the resulting bytes.

Order matters: type-specific codecs typically improve patterns *before* general compression. ZSTD on already-Delta-encoded integers does much better than ZSTD on the raw integers.

## 5.2 Generic compressors

### LZ4 (default)

- Fast: decompresses at multiple GB/sec/core.
- Modest ratio: 2-4× on text, less on numeric.
- Always available. Set as the system default unless changed.

### LZ4HC

- Higher-compression LZ4. Slower to write, same speed to read.
- Use when storage cost > write throughput.

### ZSTD(level)

- Levels 1-22. Default level 1 ≈ LZ4 speed; level 3 is a sweet spot.
- 2-5× better compression than LZ4 on most data.
- Much slower to compress at high levels; decompression is still fast.
- Common production choice: `CODEC(ZSTD(3))`.

### ZSTD_QAT

- Hardware-accelerated ZSTD via Intel QAT cards.
- Same ratio as ZSTD, much faster compress.
- Cloud servers may or may not have it.

### Deflate, Deflate_Qpl

Less common.

## 5.3 Type-specific codecs

### Delta(N)

Stores the difference between consecutive values. Best for monotonic / slowly-varying sequences.

- `N` = byte width of the stored delta (1, 2, 4, 8). Use `Delta(4)` for `UInt32`, `Delta(8)` for `UInt64` / `DateTime64`.
- `Delta` only helps if rows are physically sorted such that adjacent values are close — usually achieved by including the relevant column early in `ORDER BY`.

Canonical use: `DateTime CODEC(Delta, ZSTD(1))` for timestamp columns when rows are sorted by `(ts, ...)`.

### DoubleDelta

Delta of delta. Best for evenly-spaced data (e.g., one event per second).

```sql
ts DateTime CODEC(DoubleDelta, ZSTD(1))
```

Storage approaches near-1-bit-per-value for perfectly regular sequences. Used by Prometheus's TSDB and Gorilla.

### Gorilla

Designed for floats that vary little between adjacent rows (sensor data, Prometheus-style metrics).

```sql
cpu_pct Float64 CODEC(Gorilla, ZSTD(1))
```

XOR-based delta encoding with variable-width significant bits.

### T64

A clever codec for integer columns: packs the actual *used* bit width per block.

```sql
bytes UInt64 CODEC(T64, ZSTD(1))
```

If all values in a block fit in 7 bits, only 7 bits per value are stored. Excellent for columns that vary in distribution (most rows small, occasional large).

### FPC

Floating-point compressor. Better than Gorilla in some workloads. Less common.

### NONE

Disable compression. Useful when you've pre-compressed yourself (e.g., raw bytes from a compressed source).

## 5.4 Compression recipe by column type

| Column type | Recommended codec |
|-------------|-------------------|
| `DateTime` / `DateTime64` (sorted) | `Delta, ZSTD(1)` |
| `DateTime` (evenly spaced) | `DoubleDelta, ZSTD(1)` |
| Monotonic counter (UInt32/UInt64) | `Delta, ZSTD(1)` or `DoubleDelta, ZSTD(1)` |
| Float metric (slowly varying) | `Gorilla, ZSTD(1)` |
| Integer with variable magnitude | `T64, ZSTD(1)` |
| Low-cardinality string | (use `LowCardinality(String)`, default codec is fine) |
| Free-form text | `ZSTD(3)` (or `LZ4` if write-throughput is critical) |
| UUID / random hash | `LZ4` (incompressible anyway) |
| Boolean / small enum | `T64, LZ4` (cheap) |

## 5.5 Cluster-default vs. per-column

You can set defaults in the config:

```xml
<compression>
    <case>
        <min_part_size>10000000</min_part_size>
        <min_part_size_ratio>0.01</min_part_size_ratio>
        <method>zstd</method>
        <level>3</level>
    </case>
</compression>
```

Per-column codecs override the default. Pick global ZSTD-3 as a baseline; specialize columns where it matters.

## 5.6 Compression vs. CPU vs. cost tradeoff

| Scenario | Pick |
|----------|------|
| Write-throughput-bound (high ingest QPS) | LZ4 default; ZSTD only on hot columns |
| Storage-cost-bound (long-retention archive) | ZSTD(5-9) + aggressive type-specific |
| Query-latency-bound (analytical reads dominate) | ZSTD(1-3); decompression is fast enough; the I/O savings win |
| Cloud / object storage backed | ZSTD higher levels (network/IO is the bottleneck, not CPU) |

In ClickHouse Cloud, where every byte read goes through S3, **higher compression pays back faster** than on-prem.

## 5.7 Measuring compression ratios

```sql
SELECT
    database, table, column,
    formatReadableSize(sum(data_compressed_bytes))   AS compressed,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed,
    sum(data_uncompressed_bytes) / sum(data_compressed_bytes) AS ratio
FROM system.parts_columns
WHERE active AND database = 'default' AND table = 'events'
GROUP BY database, table, column
ORDER BY ratio DESC;
```

This is one of the first diagnostics on a slow table — find a column with terrible ratio (1.2× when ratios are 30× elsewhere) and improve its codec or type.

## 5.8 Common mistakes

- **ZSTD on UUID columns** — wasted CPU; the data is already near-random.
- **Delta on randomly-ordered integers** — Delta needs sortedness; if rows aren't sorted by that column, Delta encodes large random differences and *hurts* compression.
- **Gorilla on integer columns** — Gorilla is a float codec; for ints, use T64.
- **Forgetting the codec on time columns** — leaving DateTime with default LZ4 wastes ~5-10× of the storage savings you could get from Delta+ZSTD.
- **Mixing column-level and table-level overrides accidentally** — use `system.columns.compression_codec` to verify what's actually applied.

## 5.9 Real-world numbers

From public ClickHouse benchmarks and engineering posts:

- A typical event-tracking table: 6–8× compression at default LZ4; 12–20× with ZSTD(3) + targeted codecs.
- A metrics table (timestamps + numeric values): 30–60× with Delta/Gorilla + ZSTD.
- A log lines table (text body): 5–10× with ZSTD(3); 12-15× with ZSTD(9).
- A web-request log (mixed): 8–12× typical.

## 5.10 Must-internalize

- Two-layer: type-specific codec → generic compressor.
- `LZ4` default, `ZSTD(1-3)` very common, `ZSTD_QAT` if hardware supports.
- `Delta, ZSTD(1)` for sorted timestamps; `DoubleDelta` for evenly-spaced.
- `Gorilla` for slowly-varying floats; `T64` for integers with variable magnitude.
- Compression ratio = storage savings *and* read-speed gains (less I/O).
- Always check `system.parts_columns` to see actual ratios — assumptions are unreliable.

---

## Sources

- [Compression and codecs](https://clickhouse.com/docs/sql-reference/statements/create/table#codecs)
- [Column codec recommendations (oneuptime CH series)](https://oneuptime.com/blog/post/2026-03-31-clickhouse-codecs/view)
- [Altinity — codec deep dive](https://altinity.com/blog/2019/7/new-encodings-to-improve-clickhouse)



## Some general recommendations #
Choosing which codec and compression algorithm to use ultimately comes down to understanding the characteristics of your data and the properties of the codecs and compression algorithms. While we encourage you to test, we also find the following general guidelines useful to act as a starting point:

ZSTD all the way - ZSTD with no codec often outperforms other options concerning compression or is at the very least competitive: especially for floating points. This is thus our default compression in ClickHouse Cloud.
Delta for integer sequences - Delta-based codecs work well whenever you have Monotonic sequences or small deltas in consecutive values. 
More specifically, the Delta codec works well, provided the derivatives yield small numbers. If not, DoubleDelta is worth trying (this typically adds little if the first-level derivative from Delta is already very small). Sequences where the monotonic increment is uniform, will compress even better - see the dramatic savings on our date field.

Maybe Gorilla and T64 for unknown patterns - If the data has an unknown pattern, it may be worth trying Gorilla and T64. Gorilla is designed principally for floating point data with small changes in value. 
It specifically calculates an XOR between the current and previous value and writes it in compact binary form: with the best results when neighboring values are the same. 
For further information, see Compressing Values in Gorilla: A Fast, Scalable, In-Memory Time Series Database. It can also be used for integers. In our tests, however, plain ZSTD outperforms these codecs even when combined with them.

T64 for sparse or small ranges - Above, we have shown T64 can be effective on sparse data or when the range in a block is small. Avoid T64 for random numbers.

Gorilla possibly for floating point and gauge data - Other posts have highlighted Gorilla's effectiveness on floating point data, specifically that which represents gauge readings, i.e., random spikes. This aligns with the algorithmic properties, although we have no fields in our above dataset to verify. Our tests above suggest, at least on Float32s, that ZSTD offers the best compression of Floats.

Delta improves ZSTD - ZSTD is an effective codec on delta data - conversely, delta encoding can improve ZSTD compression. **Compression levels above 3 rarely result in significant gains**, but we recommend testing. In the presence of ZSTD, other codecs rarely offer further improvement, as demonstrated by our results above. 
We have seen reports of LZ4 offering superior compression on DoubleDelta encoded data than ZSTD on artificial datasets, but we have yet to find evidence of this with our real datasets.

LZ4 over ZSTD if possible - if you get comparable compression between LZ4 and ZSTD, favor the former since it offers faster decompression and needs less CPU. However, ZSTD will outperform LZ4 by a significant margin in most cases. Some of these codecs may work faster in combination with LZ4 while providing similar compression compared to ZSTD without a codec. This will be data specific, however, and requires testing.