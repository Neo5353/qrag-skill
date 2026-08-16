# qrag-skill

A public scaffold for QRAG: quantum-inspired retrieval over evidence paths, not just chunks.

The core idea is simple: many high-value enterprise answers are not contained in one paragraph. They live in connected evidence chains across heterogeneous documents: an invoice, a purchase order, an amendment, an approval record, a policy, a rule, or a log event. This repo frames that as an evidence-path discovery problem.

## What this repo contains

- `skills/research/needle-rag-extraction/SKILL.md`
  - a reusable Hermes/OpenClaw-style skill for million-document, needle-in-haystack extraction
  - focused on extracting exact facts with citations, confidence, and null/missing-evidence discipline
  - built around query fan-out, multi-channel retrieval, score fusion, evidence-span reranking, contradiction checks, and path-style provenance

The repo is intentionally early. It currently contains the retrieval and extraction skill first; implementation code, benchmarks, and corpus adapters can come after.

## Core idea

Traditional search retrieves similar text. QRAG treats the target as a path-finding problem over evidence-bearing objects.

Instead of searching one way and hoping, the system:

1. generates multiple query views in parallel
2. retrieves through lexical, dense, metadata, entity, and exact-pattern channels
3. amplifies candidates that survive across channels
4. expands and scores connected evidence spans as candidate chains
5. collapses from documents to answer-determinative evidence
6. extracts into a strict schema with provenance, confidence, and missing-evidence signals

It borrows the intuition of superposition, interference, amplitude amplification, and quantum walks without pretending the current implementation is running on quantum hardware.

## QRAG architecture frame

In the patent/disclosure language, the architecture is:

- candidate evidence chains are the search objects
- the user question compiles into a query-conditioned scoring/oracle function
- amplitude-amplification-style iteration spends more compute on candidates that repeatedly survive independent retrieval channels
- graph or neighbor expansion behaves like a quantum-walk analogue over evidence links
- destructive interference is modeled classically by suppressing candidates with contradictions, stale provenance, weak source quality, or missing required links

The practical system today is classical and quantum-inspired. The quantum formulation is useful because it forces the design to search over chains and provenance, not isolated chunks.

## Output contract

QRAG should return:

- an answer
- a citable evidence chain
- a confidence score
- explicit missing-evidence or null signals when the corpus does not support the answer
- contradiction notes when competing evidence exists

Example shape:

```json
{
  "answer": "Approved under Amendment 4 after invoice INV-1042 matched PO-7781.",
  "evidence_chain": [
    {"id": "invoice:INV-1042", "role": "invoice", "supports": "amount and vendor"},
    {"id": "po:PO-7781", "role": "purchase_order", "supports": "authorized spend"},
    {"id": "contract:MSA-2024-A4", "role": "amendment", "supports": "expanded scope"},
    {"id": "approval:AP-991", "role": "approval", "supports": "final authorization"}
  ],
  "confidence": 0.91,
  "missing_evidence": []
}
```

## Intended use

Use this when someone needs exact answers across a very large corpus, especially when:

- the answer is split across multiple documents
- wording varies across systems
- citations and provenance matter
- missing evidence must be reported instead of guessed
- contradiction checks matter as much as retrieval recall

Mapped domains include procurement, contracts, compliance, cybersecurity attack-chain reconstruction, industrial root-cause analysis, clinical decision support, and scientific literature synthesis.

## What this is not claiming

- It does not claim exponential speedups.
- It does not claim a production quantum hardware implementation.
- It does not claim the oracle is free.
- It does not ignore error-correction overhead.

The honest ceiling for Grover-style search is quadratic, and real quantum hardware overhead can erase that advantage. The current value is the architecture: a quantum-inspired way to reason about evidence chains as first-class retrieval objects.

If a purely classical algorithm matches the full formulation, that is a good outcome. The system can ship classically.
