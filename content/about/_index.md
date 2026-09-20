---
date: '2025-07-22T17:46:16+05:00'
lastmod: '2026-09-20'
draft: false
title: 'About'
description: "Senior Go developer at Grid Dynamics, MSc student at PJAIT in Warsaw, and the author of ursus and logd."
weight: 1
---

I'm **Asadbek (Asad) Kurbonov** — a software engineer working on backend systems,
data pipelines, and the storage engines underneath them. I've been writing code
since 2020, and most of what I enjoy is turning messy, manual processes into
systems that run themselves.

My work sits at the intersection of Go and Python: Python for data and fast
delivery, Go for services that have to stay up. I care about the unglamorous
parts — stability, predictable resource use, and clean boundaries between
components — because that's what keeps production quiet.

Lately I've been working a level lower down. Both of the things I'm building now
are engines written from scratch rather than services assembled out of
libraries, and the problems worth the time have been the ones you only meet down
there: how a validity bitmap breaks when the vector width changes, what a log
index costs if you want range queries over raw text, and how you prove that a
fast path returns exactly what the slow path would.

## What I'm building

**[ursus](/projects/ursus/)** — a dataframe library for Go, modelled on Polars.
Lazy execution with a query optimizer, Arrow memory layout, SIMD kernels, and
streaming execution that spills to disk rather than falling over. Pure Go: no
cgo, no C++ toolchain, no Python runtime, so `go get` is the whole install. It
passes 22/22 PDS-H (TPC-H) and 15/15 h2o.ai queries, every result validated
against a DuckDB reference — and it is still 4–10x slower than Polars. The
reason to reach for it is the deployment story, not the speed. Currently v0.2,
built with Claude Code.

**[logd](/projects/logd/)** — a single-node log storage engine with a
Loki-compatible API, so Grafana can point straight at it. The differentiator is
range queries on typed values pulled out of unstructured log text: given a line
like `took 247ms`, it answers `duration > 200` in O(log n). Presence-only
indexes can't compare values at all, and column stores can only do it if you
structured the data in advance. Nine phases in, it's a complete daemon —
durable ingest, typed-range and label pushdown, LogQL including metric queries,
live tail, retention, and shared-nothing sharding. It's also my master's thesis,
under prof. Krzysztof Stencel, with *Database Internals* by Alex Petrov behind
most of the design reading.

Earlier work — the production data pipeline, a Telegram bot at 100K+ users, and
a logistics CRM — is on the [projects page](/projects/).

## Currently

Since August 2026 I've been a **Senior Software Developer (Golang)** at Grid
Dynamics' Poland office.

I'm also pursuing an **MSc in Computer Science** at the Polish-Japanese Academy
of Information Technology in Warsaw, which I began in the 2025 winter semester.

I learned to program by self-study — the *One Million Uzbek Coders* program,
[Stepik](https://stepik.org/67), and a lot of building — and sharpened it
professionally at [MyTaxi.uz](https://mytaxi.uz) and through freelance work.
High-load distributed systems are what I'm drawn to. Go is my main language, and
I'm learning Rust.

## Technical stack

- **Languages:** Go, Python, JavaScript, SQL
- **Backend:** fiber, net/http, FastAPI, Django, asyncio
- **Data:** Polars, Pandas, Arrow, Parquet, ursus
- **Infrastructure:** Docker, Kubernetes, NATS, Cloudflare Workers, Google Apps Script

## Connect

- GitHub — [github.com/advenn](https://github.com/advenn)
- LinkedIn — [linkedin.com/in/bek-kurbonov](https://www.linkedin.com/in/bek-kurbonov/)
- Telegram — [t.me/code_journeys](https://t.me/code_journeys)
- Email — [qurbonovasadbek@gmail.com](mailto:qurbonovasadbek@gmail.com)
- [Résumé (PDF)](/files/Kurbonov-Asadbek-CV.pdf)
