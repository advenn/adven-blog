---
date: '2026-07-07T20:00:00+02:00'
draft: false
title: 'pon: A Compiled Python Built in a Week, Measured Against Its Claims'
tags: ['python', 'rust', 'compilers', 'jit', 'performance']
---

I came across [pon](https://github.com/can1357/pon), a project that sets out to be "the bun/v8 of Python": a from-scratch native compiler and runtime for Python 3.14, written in Rust, with no interpreter and no bytecode. The pitch is ambitious enough to be worth taking seriously — and the repository is unusual in another way. It was built in about a week: 475 commits from a single author between June 30 and July 7, with an `AGENTS.md` file and conventional-commit discipline that make it fairly clear the bulk was produced with AI coding agents.

That combination — a very hard systems project, assembled very fast, with heavy AI assistance — is what made me want to look closely. So I read the code and ran the binary. This is what I found: what the project actually is, what genuinely works today, where reality diverges from the pitch, and whether the end goal is reachable at all.

To be clear up front: none of what follows is a knock on the person behind it. A working Python compiler of any kind is genuinely hard, and this is a real one. Taking a project's claims seriously enough to test them is a form of respect. The interesting questions are how those claims hold up under measurement, and what the goal itself runs into.

## What pon is

The architecture is clean and, on paper, correct. There is one intermediate representation and two backends:

```
source.py
   │ ruff parser (pinned 0.14.0, Python 3.14)
AST ──> PON IR (one IR for every tier)
   │
   ├── pon run:   IR ──> Cranelift JIT ──> native code, in process
   └── pon build: IR ──> Cranelift object file ──> linked native executable
   │
pon-runtime (object model, builtins, native stdlib, C-ABI helpers)
pon-gc      (Green Tea tracing garbage collector)
```

Three design decisions define the project:

- **No interpreter.** Every module is parsed with ruff, lowered to a shared IR, and compiled to machine code through Cranelift — either just-in-time (`pon run`) or ahead-of-time into a standalone binary (`pon build`).
- **Tracing GC instead of reference counting.** The object model is CPython's heap layout minus the refcount header, managed by a tracing collector called Green Tea. This is the decision everything else hinges on, and I'll come back to it.
- **Byte-exact conformance.** Correctness is defined as producing output identical, byte for byte, to CPython 3.14.0 on a curated corpus, enforced by ratcheted floor files in CI.

The scope is real, not a sketch. Excluding the vendored copy of CPython used for differential testing, it is roughly 263,000 lines of Rust: a 210k-line runtime with 69 native stdlib modules (`math`, `json`, `re`, `datetime`, `socket`, `ssl`, `asyncio`, `pickle`, `unicodedata` with its 40k-line tables, and more), plus a ~21k-line CPython C-API compatibility layer. This is a lot of working machinery for a week of calendar time.

## What actually works

I ran the prebuilt release binary against a battery of programs. The JIT path is not a demo — it runs real Python:

```python
class Point:
    def __init__(self, x, y):
        self.x, self.y = x, y
    def __repr__(self):
        return f"Point({self.x}, {self.y})"

print([Point(i, i*i) for i in range(3)])   # [Point(0, 0), Point(1, 1), Point(2, 4)]
print(sum(x for x in range(100) if x % 3 == 0))   # 1683
try:
    1 / 0
except ZeroDivisionError as e:
    print("caught:", e)                    # caught: division by zero
```

Classes, dunder methods, f-strings, list and dict comprehensions, generators, generator expressions, and exceptions with correct messages all produce the right output. The stdlib modules don't just import; they compute — `math.factorial(5)`, `json.dumps(..., sort_keys=True)`, `re.findall`, `datetime.date(...).isoformat()` all behave. And the byte-exactness claim held on every spot check I threw at it, including the genuinely fiddly cases:

```python
print(0.1 + 0.2)     # 0.30000000000000004  — identical to CPython
print(10 ** 30)      # arbitrary-precision integers
print(repr("café"))  # 'café'               — correct unicode repr
```

Startup is fast, too. A trivial `print("hi")` runs in about 6 ms through the JIT — faster than CPython's ~14 ms on the same machine. The committed floor files record a couple hundred corpus modules reproduced byte-for-byte (244 under the JIT, 233 compiled ahead-of-time). Whatever else is true, there is a real, working Python compiler here.

## Where reality diverges from the pitch

The README is careful to separate "what is true today" from "where it is going," which is to its credit. Still, three gaps stand between the current state and the headline.

**It is currently slower than CPython — and how much depends heavily on the shape of the code.** The goal is a multi-tier JIT that runs "well past CPython," a stated target of ≥5× on average and ≥20× on numerics. Today it runs the other way, so rather than take one measurement I watched how the gap behaves as the work grows. (CPython here is 3.13, the version on my machine; pon targets 3.14, so read these as directional. Times are the best of two runs.)

Startup is not the bottleneck: that trivial `print("hi")` again, ~6 ms, faster than CPython. The cost is per-operation, and scaling a recursive `fib(n)` shows it plainly:

| `fib(n)` | calls | CPython | pon | pon (tier-0 only) | pon / CPython | pon ns/call |
| --- | --- | --- | --- | --- | --- | --- |
| 28 | 1.7M | 0.044 s | 0.60 s | 0.59 s | 13.5× | 358 |
| 30 | 4.4M | 0.097 s | 1.62 s | 1.63 s | 16.6× | 372 |
| 32 | 11.4M | 0.240 s | 4.06 s | 3.91 s | 16.9× | 356 |
| 34 | 29.9M | 0.607 s | 10.9 s | 11.5 s | 17.8× | 363 |
| 36 | 78.2M | 1.61 s | 33.2 s | 31.0 s | 20.6× | 425 |

Two things jump out. First, the per-call cost is essentially **flat at ~360 ns** regardless of input size — there is no warmup being amortized away, and if anything the gap to CPython *widens* as the work grows (13.5× → 20.6×). The time is going into calling machinery, boxing, arbitrary-precision integers, and GC allocation, not a one-time compile. Second, and more revealing: the `tier-0 only` column — which forces the boxed baseline and disables the optimizing tier — is identical to the full tiered run within noise. For call-heavy code, the optimizing tier currently buys *nothing*, which matches the roadmap exactly, where call and attribute specialization are still open items.

But the JIT substrate is real; it just only helps the right shape of code today. On a long-running hot loop — 60M iterations of `s += i*i` — the tiering does engage. On-stack replacement and background compilation make the tiered run about **1.4× faster** than tier-0, and pon lands ~4× behind CPython instead of ~17×:

| 60M-iteration hot loop | time | note |
| --- | --- | --- |
| CPython | 2.84 s | — |
| pon (tiered) | 11.5 s | ~4× slower than CPython |
| pon (tier-0 only) | 16.0 s | so tiering is worth ~1.4× here |

So performance isn't one number, it's a workload-shaped story: call-bound code is an order of magnitude-plus behind with no tier-up benefit yet, while loop-bound numeric code is a few times behind *and* gets faster as the optimizer kicks in. To the project's credit, this is what its own roadmap predicts — tagged small integers, unboxing, and call/attribute specialization are the listed-but-unbuilt pieces that would close the gap. The scaffolding for speed (background compilation, OSR, inline caches, type feedback) is already in place and demonstrably working on loops; making it pay off for calls is the next stretch of work. "Slower today" is a snapshot of an early tier, not a verdict on the ceiling — the ceiling is just still ahead of it.

**The single-binary path isn't robust yet.** `pon build` is the flagship feature: same IR, compiled to a standalone native executable. On my machine it failed to link — even for hello-world — with an undefined reference to `unget_wch`, a wide-character curses symbol. Because the entire runtime, including the curses module, is linked into every AoT binary, a single missing system symbol takes down builds that never touch curses. It may well link cleanly in the project's own CI with the right `ncursesw` development libraries present, and this is the kind of thing that gets shaken out with time; the honest state today is that "ship a single binary" still depends on the host environment in a way a finished product wouldn't.

**The CPython test suite hasn't started to fall yet.** The deepest correctness goal — passing CPython's own `Lib/test` suite — is tracked by a floor file that currently reads `min_pass_count: 0`, with an empty passing set. The curated differential corpus is real and passing; CPython's actual test suite is the next mountain, and the roadmap names it as the number-one item. That is the honest distance between "runs a few hundred hand-picked modules identically" and "is a drop-in Python" — a wide gap, but one the project is clear-eyed about.

## Is "the bun of Python" even possible?

Set aside the week-old snapshot and take the goal on its own terms. "The bun/v8 of Python" bundles several things that are each individually hard: byte-exact CPython compatibility, large speedups over CPython, single-binary output, batteries included, and a no-GIL runtime. History suggests you usually get *some* of these at the cost of others.

- **PyPy** proves the speed is real — a tracing JIT can beat CPython by 5–50× on pure Python. But its C-extension compatibility layer is slow and imperfect, which is exactly why so much of the ecosystem never moved.
- **Nuitka** compiles Python to C with full compatibility — by bundling and deferring to CPython itself, not by replacing it, and without the dramatic speedups.
- **Codon** gets native speed by *not* being CPython-compatible: it changes semantics and supports a subset.
- **Cinder** and **Pyston**, the production forks at Meta and Dropbox, kept full compatibility precisely by staying close to CPython and its refcounting C-API — and got modest speedups (roughly 1.5–4×) for it.

The pattern is consistent: the two things pon wants *together* — big speed and faithful compatibility including the C-extension world — have historically been a trade-off, not a package deal.

And pon has chosen the hardest possible position on the central axis. Its performance and no-GIL ambitions both flow from replacing reference counting with a tracing GC. But CPython's reference counting is *observable*, and an enormous amount of real Python — and effectively the entire C-extension ecosystem, numpy and friends — is built directly on it. `__del__` runs at a deterministic moment; `weakref` callbacks fire on a schedule; `sys.getrefcount` returns numbers programs branch on; every C extension calls `Py_INCREF`/`Py_DECREF` by hand. A tracing GC cannot reproduce that timing.

The project knows this. Its conformance harness includes a *divergence ledger* with a closed, capped taxonomy of allowed differences — and the reasons are exactly `refcount-observability`, `del-timing`, and `weakref-timing`, each annotated as behavior "that a tracing GC cannot reproduce." That is an unusually honest piece of engineering: the maintainers have written down, in the test infrastructure itself, that perfect byte-exact parity is impossible by construction, and bounded how much divergence they'll accept. It also means "the bun of Python" has a permanent asterisk. You can get remarkably close for pure-Python programs that don't inspect object lifetimes. You cannot, with this design, be a literal drop-in for the long tail that does — and the C-extension story (there is a ~21k-line C-API layer here, reaching toward `dlopen` and `PyInit_`) is the steepest climb of all, because presenting a refcounting C-API faithfully on top of a tracing GC is its own multi-year problem.

## What this says about building with AI

The most interesting thing about pon may not be the compiler but the method. One developer, seven days, ~260k lines of Rust, a working JIT, 69 native modules, a C-API layer, a differential conformance harness with governance rules — this is a genuinely impressive demonstration of how far AI-assisted development can push scope and breadth in a short time. The scaffolding is disciplined in ways that matter: centralized dependency management, ratcheted floors that fail CI on regression, a divergence ledger with an approval process and a hard cap.

But the shape of what's done versus not-done is telling. The breadth — parser wiring, IR, a Cranelift backend, dozens of stdlib modules, a test harness — is exactly where an agent that can hold a spec and generate a lot of correct, conventional code excels. The parts that are *not* there yet are the ones that resist that approach: the performance work that needs profiling, measurement, and iteration against real workloads; the AoT robustness that needs to survive messy host environments; the true-compatibility grind against CPython's own suite, where the failures are clustered and each one is its own investigation. AI and a capable engineer got this project to an astonishing starting line astonishingly fast. The remaining distance is the hard 20% that has always been the actual work of a language runtime — and that's not a criticism of anyone. Every runtime, built by every team, spends most of its life in that last stretch.

## Verdict

pon is more real than a one-week build might lead you to expect, and earlier in its life than "the bun of Python" makes it sound. The interpreter-free JIT works and runs real Python byte-for-byte; the numbers I measured say it is slower than CPython today, its ahead-of-time binaries don't yet link reliably, and its deepest compatibility goal is still at the starting line. The architecture is sound and the ambition is legitimate — and the headline goal runs into a wall the project itself has documented: you cannot be byte-exact with CPython *and* abandon reference counting, not for the programs and extensions that observe it. That's not a flaw in the execution; it's a fixed cost of the design, and the maintainer has been honest enough to write it into the test suite.

I'll be watching it — both as a language project and as a data point on what one engineer with AI agents can stand up in a week, which is a lot more than I'd have guessed. And as the project itself advises, read the committed floor files, not the README, for what's true today. That's a good instinct on its author's part, and a good one for the rest of us judging work like this: measure it before you praise or dismiss it.
