---
date: '2025-07-22T22:39:19+05:00'
draft: false
title: 'Automating Business Reports with Google Apps Script and Telegram'
tags: ["apps-script", "automation", "telegram-bot"]
---

Google Apps Script is easy to dismiss as a toy. Over two freelance projects it turned out to be a genuinely capable serverless platform for automation work built around Google Sheets and Telegram. This post covers the second of those projects — a reporting system for a US-market logistics business — and how I got there.

## First encounter

I came across Apps Script on an earlier freelance job. A consulting firm needed a Telegram bot that generated templated Word contracts. They had started the work themselves in Apps Script but hadn't finished it, so I picked it up and rebuilt the bot in Python with [aiogram](https://docs.aiogram.dev/), rendering `.docx` files from templates using [docxtpl](https://docxtpl.readthedocs.io/) and passing files around as base64. Apps Script was only incidental to that project, but it was my first look at the platform.

## The reporting project

Later, a university contact who runs a business serving the US market asked me to automate their reporting. Their operation ran almost entirely through Telegram groups — orders and updates flowed as messages between team members. The goal was to capture that data automatically and turn it into a clean, single-page report.

The design came down to two decisions: where the data would live, and how it would get there.

### Google Sheets as the datastore

I chose Google Sheets as the backing store rather than a traditional database. For this client it was the right trade-off:

- **Accessible.** The client can open, read, and edit the data directly, with no extra tooling or credentials.
- **Easy to manipulate.** Sheets is a comfortable interface for non-technical users and has a full scripting API.
- **Free.** No hosting or database costs.

### Capturing data through a Telegram bot

A Telegram bot, running on Apps Script, served as the ingestion layer. I set up a webhook handler to receive updates, parsed the relevant messages from the groups, and wrote structured rows into the sheet. The technical details of the webhook and message handling are in a companion post: [Setting Up a Telegram Bot on Google Apps Script](/posts/setting-up-telegram-bot-on-apps-script/).

With the data source in place, the bulk of the work was the report itself — a single-page summary sheet designed together with the client from a shared template.

## What was hard

The main friction was the runtime. Apps Script executes vanilla JavaScript, and I work in Python and Go day to day, so the language itself took some adjustment.

Timezones were the real difficulty. I stored timestamps in UTC alongside EST display strings, and filtering the data meant parsing requests back on the EST boundary. Getting JavaScript's date parsing to behave consistently across that boundary took the most trial and error of the whole project. I leaned on LLMs — mainly Claude, with Gemini and DeepSeek — to work through the rougher parts.

After delivery, the client requested a few additions and formatting changes, all of which were straightforward to fold in.

## When Apps Script fits

Apps Script is a low-code, serverless platform from Google that integrates tightly with Google Workspace — Sheets, Docs, and the rest — through first-class APIs. Its `UrlFetchApp` service makes outbound HTTP requests, and a deployed script can receive `GET`/`POST` requests, so it can act as a lightweight webhook server.

That combination makes it a strong fit when your data already lives in Google Workspace, your users are non-technical, and your traffic is modest. It's not the tool for high-throughput services or anything needing fine-grained control over the HTTP layer — but for gluing Telegram to a spreadsheet and shipping a working automation quickly, it's hard to beat.
