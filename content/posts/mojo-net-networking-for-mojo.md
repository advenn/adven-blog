---
date: '2026-10-30T10:30:00+01:00'
draft: false
title: 'mojo_net: Network stack in Mojo and http benchmark against rust and go'
tags: [ 'mojo', 'networking', 'linux', 'epoll', 'http', 'tls', 'performance', 'ai' ]
categories: [ 'Building mojo_net' ]
---

Trigger Warning: AI!

I have been watching Mojo language development by Modular, a Qualcomm company for years. This August they released 1.0
stable version and open sourced it later. The BDFL is Chris Lattner, the creator of LLVM and Swift language.

Mojo feels like a child of rust and python. On one hand it looks like a python syntax, but it has rust like
internals. Its memory model is like rust, it is low level, compiles to machine code and very fast.

The purpose of Mojo is to be single language for AI. Currently, they use CUDA, C++ for fast accelerated calculation and
python to orchestrate them. Mojo tries to unify them to single place, and the current state of mojo is designed for AI.

Because of that, currently it has no network stack, not even native TCP lib exists. Some wraps C, Modular themselves use
python (fastapi). There was a project named lightbug, but they archived their repositories on May this year.

Why not write network stack for mojo from scratch and benchmark it?

## What got built

Claude Code wrote it in one week, 22 to 29 October: about 75 commits, ~45,500 lines of Mojo in the libraries and
~55,000 lines of tests. I chose what to build and in what order. The code is on
[GitHub](https://github.com/advenn/mojo_net), and the detailed results are on the
[project page](https://advenn.github.io/mojo_net/).

- **linux**: every syscall is inline assembly. No libc on the I/O path.
- **net**: TCP, UDP and Unix sockets with deadlines, its own DNS resolver, connect by host name with Happy Eyeballs.
- **tls**: TLS 1.2 and 1.3 through OpenSSL 3.5. I did not let it write its own TLS. It only passes bytes between OpenSSL
  and the socket.
- **http**: HTTP/1.1 parser, server-sent events, a client for `http://` and `https://`.
- **server**: HTTP/1.1 server with timeouts, backpressure, streaming, HTTPS, a router, static files, graceful shutdown
  and worker processes.
- Also a PostgreSQL driver, a Redis client (half done), JSON, and clients for OpenAI and Anthropic APIs.

The parsers do no I/O. They take bytes in and give bytes out. So the same HTTP code works in a blocking client, in the
server's loop, and in whatever async runtime Mojo gets later.

A server looks like this:

```mojo
from net.addr import SocketAddr
from server import Handler, Request, Response, Server


struct App(Handler):
    var requests: Int

    def __init__(out self):
        self.requests = 0

    def handle(mut self, req: Request, mut res: Response) raises:
        self.requests += 1
        if req.path() == "/":
            res.text(200, "Hello, world!\n")
        else:
            res.text(404, "not found\n")


def main() raises:
    var server = Server(App())
    server.bind(SocketAddr.parse("127.0.0.1:8080"))
    server.run()
```

## No threads, no async

The first commit had an async runtime in the style of Rust's tokio. I froze it the same day. Mojo's async/await is not
stable yet, and a runtime should follow that design, not guess it.

Instead, the server is one epoll loop per process. For more CPUs you run one process per CPU, all on the same port with
`SO_REUSEPORT`, like nginx. Processes share nothing, so there are no locks. The cost: a handler must not block, because
one slow handler stalls every connection in its process.

Before any library was written, this model was tested with an echo server: 10,000 idle and 1,000 busy connections on
one CPU. Mojo held all 11,000 in every run and matched C and tokio at about 180k messages per second. That test also
found that every Mojo process has runtime threads, even if you never start one, which makes the kernel stall for
17 to 47 ms each time the fd table grows. The fix is to grow it at startup.

More on the model: [how it handles concurrency without threads](https://advenn.github.io/mojo_net/#concurrency).

## Results

TechEmpower's plaintext test: `GET /plaintext` returns `Hello, World!`. The client is wrk, on my laptop's i5-1145G7
(4 cores, 8 threads). Each server ran in its fastest usual setup: hyper with fat LTO, fasthttp in prefork mode, Go with
PGO, nginx with one worker per CPU. The server and wrk never shared a physical core. Each run was 60 seconds, and the
whole set ran twice, the second time in reverse order. The full method is on the
[project page](https://advenn.github.io/mojo_net/#method).

A "CPU" below is a hardware thread. 1 CPU is one hardware thread with its sibling left idle. 4 CPUs are two physical
cores with both of their threads.

Requests per second, the mean of the two runs:

| Scenario                              |   mojo |  hyper | fasthttp | nginx | Go net/http |
|---------------------------------------|-------:|-------:|---------:|------:|------------:|
| 1 CPU, 64 keep-alive connections      |   168k |   172k |     162k |  156k |         82k |
| 1 CPU, 256 connections, 16 pipelined  | 1,185k | 1,193k |   1,123k |  200k |        103k |
| 1 CPU, 1024 keep-alive connections    |   144k |   146k |     123k |  141k |         67k |
| 4 CPUs, 256 keep-alive connections    |   350k |   340k |     317k |  322k |        154k |
| 4 CPUs, 256 connections, 16 pipelined | 2,153k | 2,073k |   1,950k |  403k |        180k |

In short:

- **Pipelined, mojo is level with hyper.** On 1 CPU it spends 0.84 µs of CPU per request, hyper 0.83, fasthttp 0.88.
  On 4 CPUs mojo is ahead, 2.15M against 2.07M.
- **Without pipelining, the top four are within a few percent.** Most of each request there is the kernel's loopback
  TCP, the same for every server. On 4 CPUs wrk itself used 87 to 92% of its CPU, so those rows mostly measure wrk.
- **Go's net/http** uses about twice the CPU per request of the others and gains little from pipelining. fasthttp, also
  Go, is in the top group.

Throughput is not the whole story. Here is one scenario with CPU and latency (1 CPU, 64 keep-alive connections):

| Server      | req/s | CPU per request |    p50 |     p99 |
|-------------|------:|----------------:|-------:|--------:|
| hyper       |  172k |         5.77 µs | 360 µs |  612 µs |
| mojo        |  168k |         5.94 µs | 368 µs |  749 µs |
| fasthttp    |  162k |         6.16 µs | 398 µs |  992 µs |
| nginx       |  156k |         6.40 µs | 394 µs |  764 µs |
| Go net/http |   82k |        12.07 µs | 793 µs | 2.08 ms |

Mojo's median is close to hyper's, but its p99 is about 20% longer. This scenario was also the noisiest: mojo's two runs
were 153k and 183k. In the quieter run, mojo and hyper were even, 183,059 and 183,042 requests per second. Every
scenario's numbers, with both runs, are in the
[full results](https://advenn.github.io/mojo_net/#results).

Caveats: one laptop, loopback, no real network. wrk is closed-loop, so its latencies miss real queueing. A desktop
session was running at the same time. And "Hello, World!" tests the HTTP machinery, not an application.
[All the caveats](https://advenn.github.io/mojo_net/#caveats).

## A day earlier it was 35% behind

The first full run was on 27 October. Then the HTTP and server code went through reviews driven by a profile, and the
same benchmark ran again on the 28th:

| Scenario                              | mojo, 27 Oct | mojo, 28 Oct | change | hyper, 27 / 28 Oct |
|---------------------------------------|-------------:|-------------:|-------:|-------------------:|
| 1 CPU, 64 keep-alive connections      |         156k |         168k |    +8% |        181k / 172k |
| 1 CPU, 256 connections, 16 pipelined  |         762k |        1.19M |   +56% |      1.17M / 1.19M |
| 1 CPU, 1024 keep-alive connections    |         132k |         144k |    +9% |        144k / 146k |
| 4 CPUs, 256 keep-alive connections    |         301k |         350k |   +16% |        350k / 340k |
| 4 CPUs, 256 connections, 16 pipelined |        1.33M |        2.15M |   +62% |      2.12M / 2.07M |

Kernel time per request was about the same for mojo and hyper, because they make the same syscalls. So the gap was in mojo's
own code. Callgrind counted the user-space instructions per request:

- mojo before the reviews: **9,644**
- after the HTTP review: **6,417**
- after the server review: **4,227**
- hyper: **4,565**

Most of the gap was HTTP header handling: each request's headers were walked several times, byte by byte, and each
layer checked again what the layer above had already checked. Two costs came from Mojo itself. An error type passed in
registers cost ~85 instructions on every call that could fail, even when nothing failed. And creating an empty `Error()`
in advance took a stack trace on every request.

Every hot spot, with file and line: [where the time went](https://advenn.github.io/mojo_net/#profile).

## Mojo problems found on the way

Mojo is young, and the project hit a few traps. Some of them:

- A value is destroyed right after its last use. In `fcntl(owner.raw_fd(), ...)` the file descriptor is closed before
  the syscall runs.
- `String.splitlines()` also splits on tabs. Bad for HTTP parsing.
- A compiler bug in 1.2.0: reassigning a field in a loop can corrupt the heap.
- `__deinit__` is skipped when `__init__` raises halfway, which leaked an OpenSSL context.

The full list, each with a reproduction, is in the repo's `CLAUDE.md`. The main ones are on the
[project page](https://advenn.github.io/mojo_net/#learned).

## How it was checked

Not all of the code has been read by a human, including me. What was done instead:

- Other AI agents reviewed each library as attackers, against the RFCs and against how Go, nginx, hyper and Kestrel
  behave. TLS, TCP/UDP and HTTP got three rounds, the server two.
- Every finding first got a test that fails on the old code, then the fix.
- Then each fix was undone on a copy, one at a time, to check that its test catches it. In the last server review, all
  62 planted bugs were caught, 3 of them only after their tests were made stricter.
- Outputs were compared with other implementations: Python's standard library, OpenSSL's own tools, a real PostgreSQL
  and Valkey, and the x509-limbo test suite for certificates.

x509-limbo found a bug that two earlier rounds of AI review missed: a wildcard certificate accepted for a name its CA
was not allowed to issue for, the same shape as CVE-2025-61727. It is fixed. But no human TLS expert has reviewed the
TLS code, and three rounds of AI review are not the same thing.

## Status

It is an experiment, not production code. The package is not published, on purpose. The Redis client is half done.

The AI wrote the code. My job was to decide what to build, what to leave out (its own TLS, an async runtime), and which
numbers not to trust.

- Detailed results, the method and the profile: [advenn.github.io/mojo_net](https://advenn.github.io/mojo_net/)
- How to run the benchmark yourself: [Reproduce it](https://advenn.github.io/mojo_net/#reproduce)
- The code: [github.com/advenn/mojo_net](https://github.com/advenn/mojo_net)
