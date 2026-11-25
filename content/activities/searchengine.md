---
title: "searchengine"
description: "Small book search engine experiment built to understand crawling, inverted indexing in PostgreSQL, and BM25 ranking."
tags: ["python", "search", "postgresql", "redis"]
---

# searchengine

[searchengine](https://github.com/Binit06/search-engine) is a small search engine I built to explore what actually happens behind a search box—from crawling pages to building inverted indexes and ranking results. Not trying to build a commercial product, just wanted to understand search systems under the hood.

---

### How It Works

1. **URL Frontier & Workers:**
   * Uses Redis to queue URLs and manage round-robin domain scheduling.
   * Graceful fallback: if Redis isn't running locally, it seamlessly falls back to in-memory Python queues and sets so it runs anywhere without setup hassle.
2. **Rate Limiting:**
   * Uses atomic Lua scripts on Redis for sliding-window rate limiting to respect robots.txt and prevent IP bans.
3. **Inverted Index in PostgreSQL:**
   * Tokenizes scraped content into positional inverted indexes and N-grams stored in PostgreSQL for fast lookups.
4. **Ranking:**
   * Implements BM25 combined with TF-IDF scoring to rank relevance for multi-term queries.

* **Repository:** [github.com/Binit06/search-engine](https://github.com/Binit06/search-engine)
* **Tech Stack:** Python, PostgreSQL, Redis, Lua.
