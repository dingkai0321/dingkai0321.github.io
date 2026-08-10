---
layout: page
title: FIFA Brand Compliance Review System
description: Multimodal RAG compliance review system for Lenovo FIFA marketing materials.
img:
importance: 2
category: Industry
---

This project was built during my AI engineering internship at Lenovo (Beijing). The goal was to automate pre-release compliance checks for co-branded FIFA marketing assets and replace slow, inconsistent manual review.

The system covered four linked stages:

- Parse Lenovo Brand World pages and FIFA branding PDFs into structured, traceable rules.
- Index those rules with hybrid retrieval over vector and keyword signals.
- Detect logos, spacing, proportions, colors, and text usage from submitted images.
- Produce rule-grounded compliance judgments with evidence and remediation suggestions.

The implementation stack included `LangChain`, `Milvus`, `BGE-Reranker`, `PyMuPDF`, `PaddleOCR`, `FastAPI`, and multimodal vision models. Structured outputs were validated with `Pydantic`, and every decision was linked back to source rule IDs, page numbers, and annotated evidence.

Measured outcomes from the project:

- Recall@10 improved from `76.8%` to `92.4%` after hybrid retrieval and reranking.
- Faithfulness increased to `95.1%` for rule-grounded answers.
- Single-image review `P50` latency dropped from `74.2s` to `32.8s` with caching, dynamic top-k compression, and parallel orchestration.

This work pushed me to treat RAG not as a demo pattern, but as an evidence system where retrieval quality, visual measurement, and rule traceability all matter to the final answer.
