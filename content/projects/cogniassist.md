---
title: "CogniAssist"
description: "Visual agentic canvas with an ahead-of-time Python workflow compiler and distributed RabbitMQ/Redis execution workers."
date: "2025-09-01"
tags: ["ai", "agentic", "distributed-systems", "nodejs", "python", "redis"]
aliases: ["/projects/terabot/"]
---

# CogniAssist: Visual Agent Workflow Compiler & Orchestrator

[CogniAssist](https://www.cogniwide.com/products/cogniassist) is an enterprise agent orchestration platform built at Cogniwide that lets teams visually compose, connect, and execute multi-step AI agent workflows.

### Architecture Highlights
- **Ahead-of-Time Workflow Compiler:** Compiles visual React Flow node graphs into self-contained, standalone Python scripts. Eliminates inter-node network roundtrips by resolving dependencies, inlining tool classes, and embedding DFS topological traversal directly into the generated code.
- **Asynchronous Execution Queues:** Decouples API ingestion from execution using RabbitMQ task queues and Redis state caching, supporting long-running LLM tools without HTTP connection exhaustion.
- **Process Isolation & Tracebacks:** Executes Python tool tasks in isolated scratch directories under unprivileged system users with strict timeouts and fine-grained error parsing (`ModuleNotFoundError`, `SyntaxError`, `TimeoutExpired`).
- **Tech Stack:** Node.js, Python 3, RabbitMQ, Redis, React Flow, Supabase.
