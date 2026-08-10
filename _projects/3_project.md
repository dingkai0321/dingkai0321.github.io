---
layout: page
title: otto-agent
description: A transparent, local-first agent runtime with memory, skills, tasks, subagents, and context governance.
img:
importance: 3
category: AI Systems
github: https://github.com/dingkai0321/otto-agent
---

`otto-agent` is a personal AI assistant runtime I built around the idea that useful assistants need durable memory, structured tool use, and explicit operational constraints, not only model capability.

The assistant supports three entry points:

- CLI for fast local workflows
- Web dashboard for visibility and control
- Voice entry for lightweight interaction

Key system capabilities:

- Long-term memory and Knowledge RAG backed by `PostgreSQL`, `pgvector`, full-text search, and `HNSW`.
- Progressive Skill loading and `MCP`-based tool extension for controlled capability growth.
- Layered context governance with observation compression, history trimming, and summary-based context recovery.
- Permission-aware file and terminal operations with trace hooks, evaluation harnesses, and runtime telemetry export.

From an engineering standpoint, the project is where I concentrated work on agent harnesses, context engineering, memory design, and evaluation infrastructure. It is less about a single model and more about turning model calls into a system that can be inspected, constrained, and improved over time.

The stack includes `Python 3.11+`, `Anthropic/OpenAI SDKs`, `PostgreSQL`, `pgvector`, `Pytest`, and `uv`.

[GitHub Repository](https://github.com/dingkai0321/otto-agent)
