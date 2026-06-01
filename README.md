# qrag-skill

A small public repo for the QRAG idea: using a quantum-search-like retrieval strategy in classical RAG to find a few exact datapoints across very large corpora.

## What this repo contains

- `skills/research/needle-rag-extraction/SKILL.md`
  - a reusable Hermes skill for million-document, needle-in-haystack extraction
  - focused on 5–6 field extraction with citations and confidence
  - built around query fan-out, multi-channel retrieval, score fusion, span reranking, and contradiction checks

## Core idea

Instead of searching one way and hoping, the system:

1. generates multiple query views in parallel
2. retrieves through lexical, dense, metadata, entity, and exact-pattern channels
3. amplifies candidates that survive across channels
4. collapses from documents to evidence spans
5. extracts into a strict schema with provenance

It borrows the intuition of superposition and amplitude amplification without pretending the system is actually quantum.

## Intended use

Use this when someone needs a small number of exact fields extracted quickly from a huge corpus, especially when wording varies and the answer may be buried in only a few documents.

## Status

Early scaffold. The repo currently contains the skill definition first; code and benchmarks can come after.
