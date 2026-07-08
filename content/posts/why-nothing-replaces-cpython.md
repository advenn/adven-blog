---
date: '2026-07-08T00:10:00+02:00'
draft: false
title: 'Why Nothing Has Replaced CPython: A 2026 Tour of Python Runtimes'
tags: ['python', 'runtimes', 'pypy', 'cpython', 'jit', 'performance']
ShowToc: true
TocOpen: true
---

Python is the most popular programming language in the world, and one of the slowest in wide production use. That combination should be catnip for anyone building a faster runtime — and for fifteen years, people have. There is a version of Python with a tracing JIT that is genuinely several times faster. There are compilers that turn it into native code. There are ports to the JVM and the .NET CLR. Big companies have funded forks. And after all of it, the interpreter almost everyone actually runs is still plain CPython.

I wanted to understand why. So I pulled together the current runtime landscape, benchmarked the ones I could run on my own machine, checked the 2026 status of the rest against primary sources, and tried to work out the pattern: not "which is fastest," but why the fast ones keep bouncing off.

A note on method. Every number I call *measured* below I ran myself on one Linux x86-64 machine (8 cores), each runtime installed in its own isolated environment and run sequentially, best of several runs. That covers ten of them: CPython 3.13.5, CPython 3.14.4 and its free-threaded build, PyPy 7.3.23, [pon](/posts/inside-pon-compiled-python/), and — for the compiler tier — Numba, Cython, mypyc, Codon, and Travis Oliphant's brand-new post-py. These are small microbenchmarks: they measure interpreter, call, and loop overhead, not real applications, which are usually bound by I/O or C libraries where these gaps shrink or vanish. GraalPy and the abandoned forks I couldn't run cleanly here, so those I cite from project docs and 2025–2026 sources. Treat the ratios as directional, not as a leaderboard.

## Start with the benchmark

There are really two kinds of "faster Python," and they need separate tables. The first kind is a **drop-in runtime**: it runs your existing, unmodified `.py` file. I ran three small pure-Python programs — recursive `fib` (call overhead), a Mandelbrot escape-count (float-heavy nested loops), and a masked-integer loop — across five drop-in runtimes at three sizes each. Every one produced byte-identical output. Here are the large-size wall times (best of two):

| Program (large size) | CPython 3.13 | CPython 3.14 | 3.14t (no-GIL) | PyPy 7.3.23 | pon |
| --- | --- | --- | --- | --- | --- |
| `fib(34)` (recursive calls) | 0.574 s | 0.464 s | 0.505 s | **0.085 s** | 10.56 s |
| Mandelbrot 500² (float) | 1.637 s | 1.509 s | 1.972 s | **0.095 s** | *(did not finish)* |
| int loop, 30M | 2.186 s | 2.332 s | 2.194 s | **0.065 s** | 9.14 s |

Read across each row and most of the story is already there. **CPython 3.14 is a little faster than 3.13** — ~10–20% on the call- and float-heavy tasks — free, with no code change. The **free-threaded 3.14t build costs ~10–30% single-threaded** (worst on Mandelbrot); that's the price of removing the GIL, and I'll come back to what you get for it. **PyPy is in another league** — 7×, 17×, 34× faster on these three — and its margin *grows* with the workload, because its tracing JIT compiles the hot loop to native code once and then barely moves:

| Same loop, growing N | CPython 3.13 | PyPy 7.3.23 | speedup |
| --- | --- | --- | --- |
| 1M | 0.120 s | 0.027 s | 4.5× |
| 10M | 1.110 s | 0.038 s | 29× |
| 100M | 11.92 s | 0.174 s | **69×** |

CPython scales linearly with the work; PyPy flattens out. At a hundred million iterations it is sixty-nine times faster — on the *same source*, no annotations, no rewrite. (**pon**, the from-scratch newcomer in the last column, is the slow one — 15–30× behind, and it never even finished the float-heavy Mandelbrot; more on why that's still interesting later.)

So the question isn't academic. There is a mature, open-source, largely-compatible Python that runs pure-Python code several times faster, and often much more. **Why is essentially everyone still on CPython?** The answer isn't one thing; it's a set of walls that every alternative hits in a different order.

## The landscape, by strategy

Alternative runtimes cluster into a handful of strategies, and each strategy has a characteristic way of winning and losing.

### 1. Reimplement the interpreter with a JIT — PyPy, GraalPy

This is the purest attack on the problem: keep the language, replace the execution engine with one that compiles hot code at runtime.

**PyPy** is written in RPython and uses a *meta-tracing* JIT — it traces hot loops of its own bytecode interpreter and emits machine code, backed by a generational moving GC. Its homepage claims about 3× the speed of CPython on average (a geometric mean; an independent study using a harmonic mean put it closer to par, so the "3×" is workload-dependent, not a law). My numbers match the shape of that claim: excellent on long-running, compute-bound, pure-Python code; nothing to offer short scripts, because the JIT never warms up.

**GraalPy** (Oracle) takes the same idea onto GraalVM/Truffle: the interpreter is an AST that the Graal compiler specializes via partial evaluation. Oracle reports roughly 4× on `pyperformance` once warm — and, tellingly, about 4× *slower* than CPython when the JIT is unavailable, plus a heavy JVM startup and memory footprint.

Both are real engineering achievements. Both have failed to displace CPython for the same two reasons:

- **The C-extension tax.** This is the big one, and it recurs everywhere below. NumPy, pandas, PyTorch and the rest of the ecosystem are C extensions written against CPython's C-API, which is built around reference counting and CPython's exact object layout. PyPy emulates that API through a shim called `cpyext`; GraalPy emulates it too and requires extensions to be *rebuilt* (it supports the C-API but not CPython's binary ABI, so you can't reuse wheels). The emulation is expensive — PyPy's own docs warn that C-extension calls "take a very large hit." So on exactly the data-science and ML workloads where people most want speed, the JIT's gains are eroded or reversed by the boundary crossing. The intended fix, a handle-based API called **HPy** co-developed by both projects, has never reached broad ecosystem adoption. In 2024 conda-forge floated dropping PyPy support entirely.
- **Perpetual version lag.** PyPy in mid-2026 targets Python 3.11; CPython ships 3.14. GraalPy targets 3.12. If you need recent syntax or a library that requires a newer Python, the fast runtime simply isn't an option yet.

Add JIT warmup and higher memory use — which make both useless for the short-lived processes where a huge amount of Python actually runs (CLIs, serverless functions, test suites, scripts) — and you have a tool that helps a specific slice of workloads and hurts or ignores the rest.

### 2. Fork CPython and bolt on a JIT — Pyston, Cinder, Pyjion

If emulating the C-API is the problem, why not start *from* CPython, keep perfect compatibility, and add a JIT? Several serious efforts did exactly this.

**Pyston** began at Dropbox in 2014, later continued under Anaconda. It came in two forms: a full CPython 3.8 fork (~30% faster on web workloads) and *pyston-lite*, a pip-installable JIT that patched into stock CPython for ~10%. The team's hard-won lesson was that adoption friction mattered more than peak speed — recompiling your whole stack against a fork is a non-starter for most shops, and even the drop-in lite version stayed niche. They wound the project down in 2022; the repo header today reads "No longer maintained," and they set out to upstream what they could.

**Cinder** is Meta's CPython fork, powering Instagram's Django backend, now delivered as *CinderX*, a pip-installable extension adding a JIT and "Static Python" (a typed, compiled subset). It is very much alive — used in production at Meta, published to PyPI, and as of Python 3.14 it finally works against stock CPython rather than a patched fork. But it was open-sourced as a "code dump," not a supported product, and has essentially no external adoption. Its real influence is upstream: ideas like immortal objects ([PEP 683](https://peps.python.org/pep-0683/)) flowed into CPython itself.

**Pyjion** (Microsoft research, later Anthony Shaw) plugged CPython into the .NET CLR's JIT through the frame-evaluation API. It reached 1.0 in 2021, then went dormant — last release in 2022, stuck on Python 3.10.

The pattern here is brutal and clear: a CPython fork inherits perfect compatibility but also inherits an impossible job — staying current with a fast-moving upstream while justifying the switching cost of a non-standard interpreter. The moment CPython started improving itself (next section), the ~10–30% these forks offered stopped being worth the friction. Their best ideas didn't die; they got absorbed.

### 3. Compile Python instead of interpreting it — Cython, mypyc, Numba, Nuitka, Codon, post-py

A different bargain: give up on being a drop-in *runtime* and instead compile Python (or something close to it) to native code. These aren't really CPython competitors — most of them run *with* CPython — but they're how people actually get speed today, so they matter.

| Tool | What it compiles | Runtime needs CPython? | Typical speedup | Catch |
| --- | --- | --- | --- | --- |
| **Cython** | annotated Python superset → C extension | yes | 10–100× on typed numeric code | you must add C type annotations and a build step |
| **mypyc** | type-annotated Python → C extension | yes | 1.5–10× | code must fully type-check; some dynamism restricted |
| **Numba** | `@njit` numeric functions, LLVM JIT | yes | 10–100× on array math | only a narrow numeric subset of Python |
| **Nuitka** | whole program → C, bundles CPython | yes | ~1–2× | it's a packager first; speed is a side effect |
| **Codon** | a Python-*like* dialect → native, own runtime | **no** | 10–100× | not CPython-compatible; you rewrite to its subset |
| **post-py** | typed Python *subset* → C99 → shared lib / CPython ext | no (RAII, no GC) | ~60–70× on typed kernels | v0.3 alpha; subset only, no dynamic Python |

The scientific stack is quietly built on this tier — Cython underpins NumPy, SciPy, pandas and scikit-learn; mypyc compiles mypy itself and Black. But look at the "catch" column: every one of them buys speed by *removing dynamism*. Cython and mypyc want type annotations; Numba only handles numeric kernels; Codon is a separate language that happens to look like Python and, by cutting the cord to the CPython runtime entirely, also cuts itself off from the C-extension ecosystem. (Codon, worth noting, relicensed from a restrictive source-available license to Apache-2.0 in early 2025.) Nuitka keeps full compatibility precisely by *not* trying to be much faster — it bundles CPython and mostly helps you ship a single binary.

That trade — dynamism for speed — is the whole game, and it's why none of these is "faster Python" in the general sense. They're faster *subsets* of Python.

I benchmarked this tier too, holding all of them to one shared numeric kernel — the same Leibniz π loop — each in its own environment, timing the compiled kernel's compute:

| Leibniz π kernel (compute only) | 5M | 20M | 50M | vs CPython |
| --- | --- | --- | --- | --- |
| CPython 3.13 (interpreted) | 359 ms | 1356 ms | 3379 ms | 1× |
| mypyc (plain type hints) | 11.8 ms | 44.8 ms | 112 ms | ~30× |
| Numba (`@njit`) | 5.1 ms | 19.4 ms | 56 ms | ~65× |
| Cython (`cdef` types) | 5.0 ms | 19.4 ms | 49 ms | ~70× |
| Codon (native binary)† | — | — | ~52 ms | ~65× |
| **post-py** (`postpyc build`) | **4.7 ms** | **18.8 ms** | **47 ms** | **~70×** |

*† Codon compiles a standalone binary; its ~9 ms native process startup is excluded here so the compute is comparable. Every tool's output was byte-identical to CPython's.*

The takeaway isn't a winner — it's the *convergence*. Once a numeric kernel is compiled to native code, which tool you used barely matters; they all land within a small factor of each other and of hand-written C. mypyc is the instructive outlier: it asks the least of you (ordinary type hints, no `cdef` or decorator) and returns the least (~30× instead of ~70×), because it preserves more of Python's object semantics. That gradient *is* the lesson — the more you let the compiler assume, the closer to C you get.

The newest row is the interesting one. **post-py** — "Performance-Optimized Statically Typed Python," from **Travis Oliphant** (creator of NumPy and Numba), whose first alpha (`pip install postpyc`, v0.3.0) hit PyPI the day before I wrote this — is another entry in this tier with an unusual pitch. It is *spec-first*: instead of shipping yet another compiler with its own informal subset, it defines a **normative specification** for a compilable Python subset and treats the compiler as a reference implementation. The compiler emits **C99** (no LLVM), and — unlike anything else here — the compiled object model uses **single-owner RAII with no garbage collector and no GIL**, touching reference counting only at the CPython boundary. A conforming file is still valid Python you can run under CPython; stray outside the subset and the compiler rejects it with a diagnostic rather than silently changing behavior. On my one-kernel test it matched Cython and edged out Numba — but it's v0.3 alpha with a draft spec and essentially no ecosystem, and it is explicitly aimed at *replacing C and Rust for extension modules*, not at running your application. It's the same bargain as the rest of this tier — types and a subset, in exchange for C speed — dressed, ambitiously, as a standard.

The most lavishly funded bet in this whole space isn't really a Python compiler at all. **Mojo**, from Chris Lattner (creator of LLVM and Swift) and his company Modular, is a *new language* designed as a Python superset — Pythonic syntax with C++/Rust-class performance through MLIR, aimed squarely at AI and GPU workloads. It reached a feature-complete [1.0 beta in May 2026](https://www.modular.com/blog/modular-26-3-mojo-1-0-beta-max-video-gen-and-more) and is the highest-profile attempt yet to build a faster Python-family language. It is also a cautionary tale in progress: in June 2026 Modular [agreed to be acquired by **Qualcomm**](https://www.modular.com/blog/qualcomm-to-acquire-modular) for about $3.9 billion — a deal transparently about Modular's cross-vendor inference stack (MAX) and the war on Nvidia's CUDA moat, not about Mojo as a general-purpose language. Mojo was never a drop-in for CPython to begin with (it's a separate language you rewrite into, with its own young ecosystem), and now even its independent direction depends on a chipmaker's roadmap. When the best-resourced swing at "Python but fast" is, at best, *adjacent* to Python — and subject to whoever ends up owning it — that tells you something about the gravity well.

### 4. Put Python on another platform — Jython, IronPython, RustPython

These were motivated less by raw speed than by *interop*: run Python where another ecosystem lives.

**Jython** (JVM) and **IronPython** (.NET) let you call native Java or .NET libraries from Python. Both are alive but frozen in the past: Jython in 2026 still ships only **Python 2.7** — a language that's been end-of-life for years — with Python 3 existing only as a dateless roadmap. IronPython tracks roughly **Python 3.4** with a few later features backported. A decade-plus of language lag caps them at interop use cases.

**RustPython** is a Python 3 interpreter written from scratch in Rust, aiming at embedding and WebAssembly rather than replacing CPython. It's actively developed and targets 3.14 compatibility, but it's pre-1.0 and slower than CPython; its most durable contribution so far may be that its parser lineage seeded [Ruff](https://docs.astral.sh/ruff/), the fast Rust linter.

### 5. The from-scratch newcomers — pon, and the AI-assisted wave

Finally there's the boldest strategy: throw out the interpreter *and* refcounting and write a new native compiler and runtime from zero. [pon](/posts/inside-pon-compiled-python/) — which I looked at in detail recently — is a striking example, in part because it was largely built by one developer with AI agents in about a week. It compiles Python 3.14 through a Cranelift JIT and AOT, with a tracing GC and a byte-exact differential test suite against CPython.

It genuinely runs real Python — but it's the slow column in my benchmark table (6–15× behind CPython today), its per-call cost is a flat ~360 ns that the optimizing tier doesn't yet touch, and its own test infrastructure concedes the deepest wall of all: a tracing GC cannot reproduce CPython's refcount-observable semantics. It's an impressive artifact and a useful data point on how far you can get fast — and on exactly where the ceiling sits.

## Why nothing replaces CPython

Step back from the individual projects and the walls repeat. In rough order of how often they're fatal:

**1. The C-extension ecosystem is the real Python.** "Python" in production means NumPy, pandas, PyTorch, psycopg, cryptography, and ten thousand other packages — all compiled against CPython's C-API and its reference-counting object model. Any runtime that isn't CPython has two options, and both are bad: emulate the C-API (PyPy's `cpyext`, GraalPy) and pay a heavy per-call tax exactly where speed is wanted, or abandon it (Codon) and lose the ecosystem that makes Python worth using. This single fact has sunk more alternative runtimes than raw performance ever helped.

**2. Reference counting is observable, so you can't just swap the GC.** CPython frees an object the instant its last reference goes away. Real programs depend on that timing: `__del__` running at scope exit, `weakref` callbacks firing on schedule, even `sys.getrefcount`. Switch to a tracing GC — which every fast alternative wants to, because refcounting is itself a performance and concurrency drag — and those behaviors change. You can be *close* to CPython, but not byte-exact, and "close" fails the long tail.

**3. Python is extraordinarily dynamic.** `eval`, `exec`, monkeypatching, metaclasses, `sys._getframe`, runtime attribute munging — features that make Python pleasant make it hostile to ahead-of-time compilation and aggressive optimization. The compilers in tier 3 get their speed precisely by forbidding the dynamic parts.

**4. The compatibility bar is total, not partial.** Because there's no separate spec — "Python" is defined by what CPython does, undocumented corners included — being 95% compatible isn't 95% useful; it's a support nightmare. Users won't migrate a production system to a runtime that breaks one library in twenty.

**5. Most Python runs in short-lived processes.** CLIs, serverless handlers, cron jobs, test suites. JIT runtimes need seconds of warmup to pay off and cost more memory; for a script that runs for 200 ms they're pure downside. The place alternatives win — long-running compute — is a minority of where Python actually executes.

**6. CPython is a moving target that improves itself.** The "Faster CPython" project brought the 3.11 specializing adaptive interpreter ([PEP 659](https://peps.python.org/pep-0659/)) and roughly a 1.4–1.5× cumulative speedup from 3.10 to 3.14 — quietly erasing the margin the CPython forks were selling. A copy-and-patch JIT ([PEP 744](https://peps.python.org/pep-0744/)) shipped experimentally in 3.13, and free-threaded, no-GIL CPython ([PEP 703](https://peps.python.org/pep-0703/)) became officially supported (though still opt-in) in 3.14. Why bet on someone else's runtime when the default one is closing the gap for free?

That free-threading line is worth seeing measured, because it's the parallelism story people used to leave CPython *for*. Here is the same CPU-bound function run across four threads, GIL versus no-GIL:

| 4× CPU-bound work | 1 thread | 4 threads |
| --- | --- | --- |
| CPython 3.14 (GIL) | 0.56 s | 2.30 s |
| CPython 3.14t (no-GIL) | 0.63 s | **0.71 s** |

Under the GIL, four threads of Python compute take four times as long — they can't run in parallel, so they queue. Under free-threading they finish in roughly the time of one, for about a 12% single-threaded tax. Real multicore Python, in the *default* interpreter rather than a fork or a C extension — that alone removes one of the historical reasons to leave.

There's a real twist on that last point, though. In May 2025, Microsoft **disbanded the Faster CPython team** — Mark Shannon, Eric Snow, and others were laid off — and the work has since become community-led. The 3.13/3.14 JIT was, by the core devs' own admission, often *slower* than the interpreter; only in 3.15 (in beta as I write this) is it finally landing modest, real wins. A Steering-Council-blessed roadmap, [PEP 836](https://peps.python.org/pep-0836/), now shepherds the built-in JIT on a time-boxed leash — targeting roughly 5% by 3.15 and ~10% by 3.16 once it works alongside free-threading, with a hard bar it must clear by 3.17 or be reconsidered for removal. That the *default* interpreter is growing its own JIT is the most consequential development here — but even it is more fragile and under-resourced than it looks. And it *still* wins — which tells you how deep the other moats run.

**7. Inertia is a feature.** CPython is the default everywhere: every OS, every cloud image, every tutorial, every Stack Overflow answer. The effort to switch runtimes is real and immediate; the payoff is speculative and workload-specific. That math rarely closes.

## So what actually works

The honest lesson of the landscape is that **nobody replaces CPython — they augment it.** The patterns that work in production all keep CPython at the center:

- Push the hot 5% into a compiled extension — Cython, mypyc, or increasingly a Rust extension via [PyO3](https://pyo3.rs/) — and leave the other 95% as ordinary, compatible Python.
- Use Numba for numeric kernels, where a decorator buys 10–100× on the code that needs it and nothing changes elsewhere.
- Reach for PyPy on the specific service that is long-running, pure-Python, and light on C extensions — a real but narrow niche.
- Let CPython itself carry the parallelism story now that free-threading is supported, rather than escaping to another runtime for it.

This mirrors something I've run into at the deployment level: when a long-running Python service kept hitting a wall, the fix wasn't a faster Python — it was [changing the architecture around Python](/posts/go-python-architecture/), keeping it for what it's good at and moving the rest elsewhere. It's the same move one level up. You don't beat CPython; you route around its weak spots and keep its ecosystem.

## Conclusion

Nothing has replaced CPython, and it isn't because the alternatives are bad. PyPy is a genuine marvel — sixty-nine times faster than CPython on my loop, and byte-for-byte compatible on everything I ran. The reason is subtler and more permanent: **"Python" the language is inseparable from CPython the implementation and the C-API ecosystem grown on top of it.** Runtimes that keep that compatibility inherit CPython's constraints — refcounting, dynamism, the C boundary. Runtimes that shed those constraints to go fast stop being able to run the code people mean when they say "Python."

The most likely future isn't a replacement at all. It's CPython slowly absorbing the good ideas — the JIT, free-threading, immortal objects — from the very projects that tried to unseat it, now under less funding than it had a year ago. And the from-scratch newcomers like pon are valuable less as contenders than as fresh, honest maps of where the wall is. Fifteen years of alternatives have mostly served to demonstrate how much is packed into that unassuming `python3` on your PATH.
