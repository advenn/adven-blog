---
title: "ursus"
tagline: "A dataframe library for Go, modelled on Polars"
status: active
state: "v0.2 · active development"
weight: 1
date: 2026-09-20
language: "Go 1.27"
repo: "https://github.com/advenn/ursus"
docs: "https://pkg.go.dev/github.com/advenn/ursus"
install: "go get github.com/advenn/ursus"
summary: >-
  Lazy execution with a query optimizer, Arrow memory layout, SIMD kernels, and
  streaming execution that spills to disk rather than falling over. Pure Go — no
  cgo, no C++ toolchain, no Python runtime.
facts:
  - "22/22 TPC-H · 15/15 h2o.ai, validated against DuckDB"
  - "1,610 tests across four SIMD widths"
  - "Pure Go — no cgo, static binary"
---

A dataframe library for Go with the shape of Polars: you describe the query, and
the engine decides how to run it.

```go
df, err := ursus.ScanParquet("events.parquet").
    Filter(ursus.Col("price").Gt(5)).
    GroupBy(ursus.Col("region")).
    Agg(
        ursus.Col("price").Sum().Alias("revenue"),
        ursus.Col("qty").Mean().Alias("avg_qty"),
    ).
    Sort(ursus.Desc(ursus.Col("revenue"))).
    Collect(ctx)
```

The projection reaches the Parquet reader, the filter becomes a row-group
predicate, and the group-by runs on every core. None of that is visible in the
query.

## The trade

It is pure Go. No cgo, no C++ toolchain, no Python runtime, no sidecar process.
It cross-compiles and links into a static binary like any other dependency, and
`go get` is the whole install.

That is the reason to pick it, and the cost should be just as plain: **ursus is
slower than the serious analytical engines.**

| | ursus | Polars | |
|---|--:|--:|---|
| h2o.ai, 10M rows | 2,401 ms | 601 ms | 4.0x |
| TPC-H SF=1 | 936 ms | 86 ms | 10.8x |
| TPC-H SF=0.1 | 160 ms | 76 ms | 2.1x |
| TPC-H SF=1 peak memory | 2.26 GB | 0.81 GB | 2.8x |

The gap narrows as the data gets smaller, which is the shape of the trade. For
data that fits comfortably in memory, the difference is half a second against a
tenth of a second and nobody is waiting. For interactive analytics over hundreds
of millions of rows, use DuckDB. Where cgo is free, `duckdb-go` is roughly 10x
faster and is a binding to a mature engine.

## What works

Parquet and CSV both directions; Arrow in and out in pure Go with zero-copy
export; the full scalar type set including 128-bit Decimal and Enum; three-valued
Kleene logic; `.str` and `.dt` namespaces; 20 aggregates; window functions; all
seven equi-join kinds plus non-equi `JoinWhere`/`WhereExists` and as-of joins;
`GroupByDynamic` and `Rolling`; predicate and projection pushdown including
through joins; parallel hash aggregation; and spilling for sort, hash
aggregation and hash join.

Not there yet: `Pivot` (its output columns are the distinct values of a column,
so its schema would depend on data), SQL, common subexpression elimination, and
writing nested columns back to Parquet.

## Correctness

The test matrix runs every case at four SIMD widths — 512, 256, 128 and scalar —
plus the `GOEXPERIMENT=simd` build off entirely. That is not decoration. Vector
width is a *runtime* property, and 512-bit gives 8 float64 lanes, which happens
to be exactly one bitmap byte. That coincidence hides an entire class of
sub-byte validity-bitmap bug; the 128-bit leg is where they surface.

Every benchmark result is validated against a DuckDB reference. A disagreement
is struck through in the report rather than published as a fast number.

[Read the write-up →](/posts/ursus-dataframes-in-go/)
