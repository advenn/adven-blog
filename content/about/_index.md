---
date: '2025-07-22T17:46:16+05:00'
draft: false
title: 'About'
layout: "about"
menu: "main"
weight: 1
---

I'm **Asadbek (Asad) Kurbonov** — a software engineer who works mostly on backend systems, data pipelines, and
automation. I've been writing code since 2020, and I like turning messy, manual processes into systems that run
themselves.

Most of my work lives at the intersection of Go and Python: Python for data and fast delivery, Go for services that have
to stay up. I care about the unglamorous parts — stability, predictable resource use, and clean boundaries between
components — because that's what keeps production quiet.

## Selected work

- **Production data pipeline.** Migrated a real-time aggregation service from Pandas to Polars for a ~50% speedup in
  production, then re-architected it into a Go orchestrator with short-lived Python workers to eliminate a persistent
  memory leak — taking OOM restarts to zero and serving reads from memory at ~40 µs.
- **High-traffic Telegram bot (100K+ users).** Built and operated a Telegram bot end to end, from webhook handling to
  data storage.
- **Django CRM (freelance).** Full digitalization of the logistics company business. The CRM automates their accounting,
  manages income, invoices, reports, employees, salaries, KPI, etc. (in maintenance mode)
- **Native golang dataframe library.** I'm building dataframe library that uses golang 1.27's generic methods and SIMD
  capabilities using Claude Code. Currently not ready for production, but passes PDS-H benchmark, at this stage slower
  than polars though. repo: https://github.com/advenn/ursus (in active development)
- **Log storage.** I am building logd, a log storage engine with its own storage and query engine. I am trying to
  achieve more efficient storage usage than loki via custom storage engine. Also I'm trying to achieve faster queries
  using reverse indexing for pre-determined patterns and substrings. Currently, it is MVP v0.1 ready. This project is
  also my master’s thesis under prof. Krzysztof Stencel. To accomplish this, I am reading Database Internals by Alex
  Petrov and applying the knowledge I am getting (in active development)

## Currently

I'm pursuing an **MSc in Computer Science** at the Polish-Japanese Academy of Information Technology in Warsaw, which I
began in the 2025 winter semester.

I learned to program by self-study — the *One Million Uzbek Coders* program, [Stepik](https://stepik.org/67), and a lot
of building — and sharpened it professionally at [MyTaxi.uz](https://mytaxi.uz) and through freelance work.

Since August 2026 I'm working as Senior Software Developer (Golang) at Grid Dynamics Poland office.

I'm passionate about high load distributed apps. I use golang, and learning rust. For now, logd and ursus are my
attempts to create platforms that are fast and efficient.

## Technical stack

- **Languages:** Go, Python, JavaScript, SQL
- **Backend:** fiber, net/http, FastAPI, Django, asyncio
- **Data:** Polars, Pandas, ursus
- **Infrastructure:** Docker, Kubernetes, NATS, Google Apps Script

## Connect

- GitHub — [github.com/advenn](https://github.com/advenn)
- LinkedIn — [linkedin.com/in/bek-kurbonov](https://www.linkedin.com/in/bek-kurbonov/)
- Email — [qurbonovasadbek@gmail.com](mailto:qurbonovasadbek@gmail.com)
- [Résumé (PDF)](/files/Kurbonov-Asadbek-CV.pdf)
