---
title: Bunori
description: Open source reader implementing an wasm bridge for sandboxed extension runs + automated job scheduling and sequencing
date: 2026-06-01
tags:
  - wasm
  - c
  - kotlin
  - android
  - systems
  - rust
---

# Bunori: Novel Reader for Android

[Bunori](https://github.com/bunoriapp/bunori) is an open-source Android reader application designed to parse and render web books and light novels in a distraction-free environment while also supporting exports to various formats.

### Architecture Highlights

- **WebAssembly Runtime for Extensions:** Embeds WAMR through native C (`wamr_bridge.c`) and JNI, enforcing strict memory boundaries (128 KB stack, 512 KB heap) so third-party scraper plugins cannot trigger out-of-memory faults or compromise the host Android process.
- **Auto Job Scheduler:** Consists of an automated Job Scheduler that can prioritises tasks based on the importance `manual actions are prioritised over in application actions`.
- **Inverted Network Callbacks:** Bridges network calls back to Android (`host_http`), enabling sandboxed scrapers to leverage the app's centralized OkHttpClient with DNS-over-HTTPS, connection pooling, and Cloudflare challenge detection.
- **Crash Auto Resume:** Download states are stored in `database` giving the ability to resume from crashes in case any happens where the scheduler auto discards the failed task to avoid a failure loop.
- **Tech Stack:** Kotlin, C, WebAssembly, Android NDK, JNI, Rust.
- **Repository:** [github.com/bunoriapp/bunori](https://github.com/bunoriapp/bunori)
### Engineering Deep Dives:
- [[blog/bunori-job-scheduler|Read how the scheduler works in Bunori]]