---
title: "Real-time aggregation pipeline"
tagline: "Go orchestrator, short-lived Python workers"
status: shipped
state: "In production"
weight: 3
language: "Go · Python · NATS"
metric: "OOM restarts → zero"
summary: >-
  A 24/7 aggregation service on 2 CPUs and 3 GB of RAM. Migrated from Pandas to
  Polars for a ~50% speedup, then re-architected into a Go orchestrator with
  short-lived Python workers to kill a memory leak I could not find.
---

A data aggregation service running as a Kubernetes pod on 2 CPUs and 3 GB of
RAM, executing every 2–2.5 minutes over 5,000–7,000 rows × 100–150 columns,
combining database calls, API requests, joins and group-by aggregations into one
frame.

Two changes, in order:

**Pandas → Polars.** The pipeline was CPU-bound and every operation was already
vectorized, so the remaining win was the engine itself. Roughly 50% off the
production runtime.

**Monolith → Go orchestrator + Python workers.** Memory kept climbing and I
could not find the leak. Rather than keep hunting, I made it structural: a Go
service owns the schedule, the HTTP surface and the in-memory read path; Python
computation runs in short-lived worker processes that exit and take their heap
with them. OOM restarts went to zero, and reads are served from memory at
~40 µs.

Written up in two parts: [Pandas vs Polars in
production](/posts/pandas-vs-polars-in-production/) and [architecting around a
bug you can't find](/posts/go-python-architecture/).
