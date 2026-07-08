---
date: '2026-07-08T12:30:00+02:00'
draft: false
title: 'Polars vs Pandas, PyO3 vs Cython: Rust Is Rewriting Python''s Fast Path'
tags: ['python', 'rust', 'polars', 'pandas', 'pyo3', 'cython', 'performance']
categories: ['Performance Optimization in Python']
ShowToc: true
TocOpen: false
---

If you list the tools that made Python feel fast lately, a pattern jumps out. The dataframe library everyone is switching to, [Polars](https://pola.rs/), is written in Rust. The linter that replaced flake8, [Ruff](https://docs.astral.sh/ruff/), is Rust. The packaging tool eating pip and virtualenv, [uv](https://docs.astral.sh/uv/), is Rust. Pydantic v2 rewrote its core in Rust; Hugging Face's `tokenizers` is Rust; `orjson` is Rust; even `cryptography` moved its guts to Rust years ago.

Python isn't going anywhere — it's still the interface. But the *engine* underneath the performance-critical parts is quietly being rewritten in Rust. This post looks at that shift from two angles I've actually measured: **Polars vs Pandas** (the library you use) and **PyO3 vs Cython** (how you'd build such a library yourself). All numbers below I ran on one 8-core Linux x86-64 box.

## Polars vs Pandas

The two dataframe libraries make opposite architectural bets.

**Pandas** is the incumbent: a decade and a half of API, built on **NumPy** with hot paths in **C and Cython**. It's *eager* (every operation runs immediately) and mostly *single-threaded*. Its data model grew organically, with per-column dtypes and a lot of Python-level glue.

**Polars** is the challenger: written in **Rust**, built on the **Apache Arrow** columnar memory format, *multithreaded by default*, and — in its lazy API — backed by a **query optimizer** that reorders and prunes work before executing (predicate pushdown, projection pushdown, common-subexpression elimination). Where Pandas runs your steps as written, Polars can rewrite the plan.

Here's how that plays out on 10 million rows (Polars 1.42, Pandas 3.0, best of three):

| Operation (10M rows) | Pandas 3.0 | Polars 1.42 | speedup |
| --- | --- | --- | --- |
| group-by + aggregate (1000 groups) | 190 ms | 129 ms | 1.5× |
| filter + group-by | 220 ms | 69 ms | 3.2× |
| left join (10M × 50) | 631 ms | 54 ms | **11.6×** |

The honest read is more interesting than "Polars is 10× faster." On a plain group-by, the gap is only ~1.5× — **Pandas 3.0 genuinely closed a lot of ground** (Arrow-backed types, years of optimization). Where Polars pulls away is joins and multi-step queries: the join is nearly 12× faster, and the filter-then-group-by benefits from the lazy optimizer pushing the filter down and running both stages across all cores. The more your workload looks like a *pipeline* — filter, join, group, sort, chained together — the more Polars' plan optimization and parallelism compound.

**When to reach for which:**

- **Pandas** still wins for exploratory work in notebooks, small-to-medium data, and — the big one — ecosystem gravity: matplotlib, scikit-learn, statsmodels, and a thousand tutorials speak Pandas. If your data fits comfortably in memory and you're iterating interactively, Pandas is frictionless.
- **Polars** wins when the data is large, the work is a repeatable pipeline, memory matters (Arrow's columnar layout and its streaming engine are far more frugal), or performance is load-bearing. Its expression API is also, in my opinion, cleaner once it clicks.

I'm not neutral here — I [migrated a production pipeline from Pandas to Polars](/posts/pandas-vs-polars-in-production/) and cut processing time in half on constrained hardware, which is what sent me down this whole rabbit hole. But the switch isn't free (a new API to learn, and the C-extension interop story differs), and Pandas remains the right default for a lot of day-to-day analysis.

## PyO3 vs Cython: building the extension

Polars is a library you *use*. The more general question is how you'd *build* a fast extension for the hot 5% of your own code. For years the answer was **Cython** (or hand-written C). Increasingly it's **Rust via [PyO3](https://pyo3.rs/)**, built with [maturin](https://www.maturin.rs/). So I built the same numeric kernel both ways and measured it.

**On raw speed, it's a tie.** Both compile to a native CPython extension, and both land at hand-written-C performance:

| Leibniz π kernel | PyO3 / Rust | Cython |
| --- | --- | --- |
| 5M terms | 4.95 ms | 5.02 ms |
| 20M terms | 20.2 ms | 18.6 ms |
| 50M terms | 48.2 ms | 51.3 ms |

And the per-call FFI overhead — the cost of crossing from Python into the extension, which matters when you call it in a tight loop — is close, with Cython actually a hair leaner:

| Cost of one call into the extension | ns/call |
| --- | --- |
| Cython | 28 ns |
| PyO3 / Rust | 34 ns |
| pure Python (for reference) | 106 ns |

So if you were hoping Rust would make your kernel *faster* than Cython, it won't. They're both ~C, and Cython's call path is marginally shorter. **Speed is not why the ecosystem is moving.** Here's what actually differs:

**What Rust/PyO3 brings:**

- **Memory safety.** C and Cython let you write buffer overruns, use-after-frees, and undefined behavior that surface as segfaults or CVEs. Rust's borrow checker makes whole classes of those bugs a compile error. For a security-sensitive library like `cryptography`, that's the entire reason it moved.
- **Fearless concurrency.** Inside a Rust extension you're outside the GIL, and Rust's type system makes multithreading safe to write. Reach for [rayon](https://docs.rs/rayon/) and a serial loop becomes a parallel one by changing `iter()` to `par_iter()`. This is how Polars gets its parallelism.
- **Tooling and ecosystem.** `cargo` + `crates.io` + `maturin` is a genuinely better build experience than setuptools invoking a C compiler. And you get serde, ndarray, regex, and the rest of the Rust ecosystem for free.

**What Cython/C still brings:**

- **Gradual adoption.** Cython is a superset of Python — you can start with a `.py` file, add types incrementally, and never leave the language your team already knows. PyO3 asks you to write actual Rust.
- **NumPy intimacy and simple cases.** For small numeric kernels tightly coupled to NumPy buffers, Cython is often less ceremony. And it underpins so much of the scientific stack (NumPy, SciPy, pandas, scikit-learn) that it isn't going anywhere.

The evidence that the industry made this trade is everywhere: `cryptography` (2021, and it caused a minor revolt), **Pydantic v2** (core rewritten in Rust via PyO3, with large validation speedups), Hugging Face `tokenizers`, `orjson`, and Polars itself. When a marquee project wants a fast core in 2026, Rust is now the default answer where it used to be C.

## Why Rust, specifically

Python's performance problem always forced a choice between two bad options: stay in pure Python and be slow, or drop to C and inherit its footguns. Rust is the first mainstream language that offers **C-level speed *and* memory safety** at the same time, with tooling (cargo, PyO3, maturin) polished enough that the FFI boundary is no longer a research project. That combination is exactly the shape of the gap in Python's ecosystem — the hot core wants to be fast, safe, and parallel, and Rust is the first tool that delivers all three without a GIL in the way.

## The caveats

This is not "rewrite everything in Rust."

- **The learning curve is real.** The borrow checker is a genuine wall for people coming from Python, and for a simple kernel, Numba or Cython will get you 90% of the benefit for 10% of the effort.
- **Compile times and build complexity.** A Rust toolchain in your build pipeline is a heavier dependency than a C compiler, and cold Rust builds aren't fast.
- **Most code shouldn't be an extension at all.** The whole point is that you profile, find the hot 5%, and rewrite *only that*. The other 95% stays ordinary, readable Python.

And on the dataframe side, Pandas is still the correct default for a huge amount of interactive, small-data, ecosystem-bound work.

## Conclusion

The lesson from every attempt to make Python fast is that you don't replace the language — you keep Python as the interface and push the hot path into a compiled extension. What's changed, quietly, is the *language of that extension*. For thirty years it was C, later softened by Cython. Now it's Rust: same native speed, but memory-safe, ergonomically bound to Python through PyO3, and fearless about the parallelism the GIL otherwise denies you.

My benchmarks say Rust won't make your kernel faster than a well-typed Cython one — they're both just "native." Rust wins on everything *around* the kernel: safety, concurrency, and tooling. That's why the fast new thing in Python — Polars, Ruff, uv, pydantic-core — keeps turning out, on inspection, to be Rust wearing a Python interface. Python is still the language you write. Increasingly, Rust is the language it runs on.
