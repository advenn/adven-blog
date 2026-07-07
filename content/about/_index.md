---
date: '2025-07-22T17:46:16+05:00'
draft: false
title: 'About'
layout: "about"
menu: "main"
weight: 1
---

I'm **Asadbek Kurbonov** — a software engineer who works mostly on backend systems, data pipelines, and automation. I've been writing code since 2020, and I like turning messy, manual processes into systems that run themselves. Online I go by *adven* (from the Latin *advena*, "stranger").

Most of my work lives at the intersection of Python and Go: Python for data and fast delivery, Go for services that have to stay up. I care about the unglamorous parts — stability, predictable resource use, and clean boundaries between components — because that's what keeps production quiet.

## Selected work

- **Production data pipeline (2 CPU / 3 GB pod).** Migrated a real-time aggregation service from Pandas to Polars for a ~50% speedup in production, then re-architected it into a Go orchestrator with short-lived Python workers to eliminate a persistent memory leak — taking OOM restarts to zero and serving reads from memory at ~40 µs.
- **High-traffic Telegram bot (100K+ users).** Built and operated a Telegram bot end to end, from webhook handling to data storage.
- **Django CRM (freelance).** Automated a company's weekly reporting workflow — from a multi-day manual process down to a few clicks.
- **Report automation (freelance).** Delivered a Telegram-to-Google-Sheets reporting system on Google Apps Script for a US-market logistics business, and a Dockerized LibreOffice service that turns wide Excel sheets into clean single-page PDFs.

## Currently

I'm pursuing an **MSc in Computer Science** at the Polish-Japanese Academy of Information Technology in Warsaw, which I began in the 2025 winter semester.

I learned to program by self-study — the *One Million Uzbek Coders* program, [Stepik](https://stepik.org/67), and a lot of building — and sharpened it professionally at [MyTaxi.uz](https://mytaxi.uz) and through freelance work.

## Technical stack

- **Languages:** Python, Go, JavaScript, SQL
- **Backend:** FastAPI, Django, asyncio
- **Data:** Polars, Pandas
- **Infrastructure:** Docker, Kubernetes, NATS, Google Apps Script

## Connect

- GitHub — [github.com/advenn](https://github.com/advenn)
- LinkedIn — [linkedin.com/in/bek-kurbonov](https://www.linkedin.com/in/bek-kurbonov/)
- Email — [qurbonovasadbek@gmail.com](mailto:qurbonovasadbek@gmail.com)
- [Résumé (PDF)](/files/Kurbonov-Asadbek-CV.pdf)
