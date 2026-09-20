---
date: '2026-09-20T19:45:00+02:00'
draft: false
title: 'ursus: A Dataframe Library for Go, and What It Is Honestly Worth'
tags: ['go', 'dataframe', 'polars', 'arrow', 'simd', 'performance']
categories: ['Building ursus']
---

I have spent the last weeks building [ursus](https://github.com/advenn/ursus) — a dataframe library for Go modelled
on Polars. Lazy execution with a query optimizer, Arrow memory layout, SIMD kernels, and streaming execution that spills
to disk instead of falling over.

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

The projection reaches the Parquet reader, the filter becomes a row-group predicate, and the group-by runs on every
core. None of that shows up in the query.

## The one reason to use it

It is pure Go. No cgo, no C++ toolchain, no Python runtime, no sidecar process. It cross-compiles and links into a
static binary like any other dependency, and `go get` is the whole install.

That is the reason to pick it, and I would rather state the cost in the same breath: **ursus is slower than the serious
engines.** On the h2o.ai benchmark at ten million rows it is about 4x Polars. On TPC-H at scale factor 1 it is about
10.8x. CSV parsing is worse than either. It is not going to beat Polars or DuckDB, and I do not think it ever will.

| | ursus | Polars | |
|---|--:|--:|---|
| h2o.ai, 10M rows | 2,401 ms | 601 ms | 4.0x |
| TPC-H SF=1 | 936 ms | 86 ms | 10.8x |
| TPC-H SF=0.1 | 160 ms | 76 ms | 2.1x |
| TPC-H SF=1 peak memory | 2.26 GB | 0.81 GB | 2.8x |

The gap narrows as the data gets smaller, which is exactly the shape of the trade. If your data fits comfortably in
memory — up to tens of millions of rows — the difference is half a second against a tenth of a second and nobody is
waiting. If you are doing interactive analytics over hundreds of millions of rows, use DuckDB. If you can link cgo
freely, `duckdb-go` is roughly 10x faster than this and is a binding to a mature engine.

The case for ursus is narrow and I think it is real: Go has not had a dataframe library of this shape, and one that is
correct and a few times slower is more useful than none — at the sizes most services actually handle.

## Why Go 1.27 specifically

Two changes in 1.27 are load-bearing. Generic methods are why `Series[T].Map[U]` and `df.Column[T](name)` exist at all.
And because a generic method still cannot satisfy an interface, every public type ended up a concrete struct with all
the polymorphism pushed down into unexported interfaces. That constraint shaped the entire public API.

`GOEXPERIMENT=simd` is optional. With it, the SIMD kernels switch on; without it every kernel falls back to a scalar
twin behind a build tag. That is not a claim I am asking anyone to take on faith — CI runs the whole suite with the
experiment off on every push. The flag buys speed, not correctness.

## The part I care about more than the benchmarks

1,610 test cases, run across four SIMD widths (512/256/128/0) plus the experiment off entirely.

The matrix is not decoration. Vector width is a *runtime* property, so a single-width run proves very little. 512-bit
gives you 8 float64 lanes, which happens to be exactly one bitmap byte — a coincidence that hides an entire class of
sub-byte validity-bitmap bug. The 128-bit leg is where those surface, and they did.

Correctness is also checked against other engines. The benchmark suite validates every result against a DuckDB
reference, and a disagreement is struck through in the report rather than quietly published as a fast number. ursus
currently passes 22/22 TPC-H queries and 15/15 h2o.ai queries.

Two days ago I found that Duration aggregates were accumulating in float64. On a one-row group, `sum` disagreed with
`min` sitting next to it in the same `Agg()` — nothing was ever summed, the value was rounded on the way in. A total of
`MaxInt64` came back negative on amd64 and positive on arm64. Worse, `cum_sum` was not imprecision at all but type
confusion: float64 bits published as a tick count, so one hour read back as 152 years. It survived because the column
builder type-checked values on the way *out* but not on the way *in*, and float64 and int64 being the same width meant
neither the dtype nor the row count could ever disagree.

I mention it because that is what most of these two weeks actually were. Not writing kernels — finding out which of the
ones I had written were lying.

## What works, and what does not

Working today: Parquet and CSV both directions, Arrow in and out in pure Go (export is zero-copy), the full scalar type
set including 128-bit Decimal and Enum, three-valued Kleene logic, `.str` and `.dt` namespaces, 20 aggregates, window
functions, all seven equi-join kinds plus non-equi `JoinWhere`/`WhereExists` and as-of joins, `GroupByDynamic` and
`Rolling`, predicate and projection pushdown (including through joins), parallel hash aggregation, and spilling for
sort, hash aggregation and hash join.

Not there: `Pivot` — its output columns are the distinct values of a column, so its schema would depend on data and no
plan node here does that. No SQL, no common subexpression elimination, no `MapGroups`. Nested types are half-done: List
and Struct read from Parquet and Arrow with `Explode`, `Unnest` and a `.list` namespace, but nested columns cannot be
written back to Parquet yet.

One thing ursus can genuinely beat Polars at: a per-element UDF in Go is a function call, not a Python interpreter
round trip.

```go
lf.Select(ursus.Col("celsius").MapElements("to_fahrenheit", ursus.Float64,
    func(c float64) (float64, error) { return c*9/5 + 32, nil }))
```

Nulls pass through untouched, so your function never receives a zero value it cannot distinguish from a real one.

## How it was built

93,000 lines of Go in about ten days, with Claude Code. I want to be plain about that rather than let anyone infer it
from the commit velocity. What made it work was not the code generation — it was refusing to let any step end without
an as-built document. There are 62 of them in [`context_files/`](https://github.com/advenn/ursus/tree/master/context_files),
one per step, each written to be authoritative over the ones before it. They record the defects found, the measurements
taken, the tests that turned out to be vacuous, and the optimisations that were tried and reverted for being slower.

Packages are arranged in strict import levels, and a generator fails the build if a package imports one at or above its
own level. That is what keeps the dependency graph a DAG rather than a suggestion — and with a codebase growing this
fast, a structural rule the build can check beats a convention every time.

## Status

v0.2 is complete. The API still moves, nothing is promised, and I would not put it anywhere production-critical today.
The docs are on [pkg.go.dev](https://pkg.go.dev/github.com/advenn/ursus); the [runnable examples](https://github.com/advenn/ursus/blob/master/example_test.go)
are checked by `go test`, so an example that stops being true breaks the build instead of misleading someone quietly.

MIT licensed. If you have a Go service that needs real dataframe work and cannot take on cgo, I would like to hear how
it goes.
