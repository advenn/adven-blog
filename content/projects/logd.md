---
title: "logd"
tagline: "A log storage engine with typed-range indexing"
status: active
state: "Phases 1–9 complete · active development"
weight: 2
date: 2026-09-20
language: "Go"
repo: "https://github.com/advenn/logd"
install: "go run ./cmd/logd -config config.yaml"
summary: >-
  A single-node log daemon with a Loki-compatible API. Its differentiator is
  range queries on typed values pulled out of unstructured log text — answer
  `duration > 200` in O(log n) without pre-structuring the data.
facts:
  - "Loki-compatible: Grafana points straight at it"
  - "O(log n) typed range queries over raw log text"
  - "Also my MSc thesis, under prof. Krzysztof Stencel"
---

A lightweight, single-node log storage daemon with a Loki-compatible HTTP API.
Grafana can point at it without knowing the difference.

## The differentiator

Range queries on **typed values extracted from unstructured log text.** Given a
line like `request took 247ms`, logd pulls `247` out at ingest and answers
`duration > 200` in O(log n).

Loki and VictoriaLogs index presence, not value — they can find lines
*containing* a number but not compare it. ClickHouse and Quickwit can compare,
but only over columns you structured in advance. logd does it against the raw
text, with the extraction declared as a template:

```
took {ms:int}ms
```

That compiles to literal fragments plus typed captures. One Aho-Corasick
automaton finds every literal position in a single pass, and hand-written byte
scanners parse int/float/str/uuid out of the anchors.

## How it stores things

Keys are 16 bytes, encoded so that **byte order equals value order** —
sign-flipped ints, IEEE-754 total-order floats with −0 normalized, longest valid
UTF-8 prefix for strings, raw uuid. Each sealed segment gets its own sorted flat
index file (`.tidx`), CRC-headered, written atomically at seal. No global merge,
no compaction: per-segment files mean retention is a delete, not a rewrite.

Pages are 4 KiB with a CRC32 over the whole page, not just the header — which is
what makes torn-page recovery possible. On restart the active segment is
rescanned, its real time bounds republished to the manifest, and any torn
trailing page truncated.

## Correctness

The contract is enforced by a **differential oracle**: the indexed path
`Execute(q)` must return exactly what the brute-force `ExecuteScan(q)` returns,
for a large battery of queries across multiple segments — typed ranges, lossy
string ranges beyond 16 bytes, negative bounds, labels, intersections, limits,
directions, cross-kind predicates, config changes.

Everything that could be wrong degrades to a scan instead: a non-indexed
segment, a `!=`, a field absent from a segment's schema, a missing or corrupt
`.tidx`. Never a wrong answer — just a slower one.

## What's done

Durable ingest with group-commit fsync and crash recovery; typed-range and label
pushdown with a seq-scan-vs-index cost guard; Loki push (protobuf and JSON); a
hand-written LogQL lexer, parser and AST translator, so a Grafana `| latency_ms >
200` transparently drives the typed-range index; metric queries
(`count_over_time`, `rate`, the `unwrap` aggregations, vector aggregation with
`by`/`without`); Grafana label discovery; WebSocket live tail on a hand-rolled
RFC 6455 server; retention; a RAM budget that seals segments early; and
shared-nothing multi-writer sharding where fan-in provably equals single-shard.
