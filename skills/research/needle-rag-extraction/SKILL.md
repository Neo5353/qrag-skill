---
name: needle-rag-extraction
description: Use when you need to find specific facts or evidence chains across a very large corpus by widening retrieval, narrowing aggressively, and extracting with provenance.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [rag, retrieval, extraction, large-corpus, ranking, schema, needle-in-haystack]
    related_skills: [dspy, doc-corpus-workflows, llm-wiki, writing-plans]
---

# Needle-in-Haystack QRAG Extraction

## Overview

This skill adapts the intuition behind "superposition" and amplitude amplification to classical RAG and evidence-path discovery.

The idea is not to search one way and hope. It is to represent the user's target in many retrieval forms at once, gather weak signals from many parts of the corpus in parallel, then use staged filtering and schema-bound extraction to concentrate probability mass onto the right few documents, spans, and evidence links.

Many enterprise answers are not located in a single chunk. They live in an evidence path: for example, an invoice connected to a purchase order, connected to a contract amendment, connected to an approval record, connected to a policy rule. Miss one link and the answer is unsupported.

For million-document corpora, speed comes from doing almost all work with indexes, embeddings, lexical search, metadata filters, and rerankers. The LLM is used late and narrowly, only on shortlisted spans.

Think of the pipeline as:

① fan out many hypotheses at once
② let multiple retrieval methods vote
③ amplify agreement and rare exact matches
④ expand plausible evidence links
⑤ extract only from the top evidence paths
⑥ return structured fields with citations, confidence, and missing-evidence signals

## When to Use

Use this skill when:
- You need 5-6 exact datapoints from a very large corpus.
- The corpus is too large for naive prompt stuffing or brute-force summarization.
- Different documents may express the same fact with different wording.
- You need traceable extraction with citations, not just a generative answer.
- The answer may require multiple linked evidence objects, not a single retrieved paragraph.
- You need explicit missing-evidence signals when a required path link is absent.
- Latency matters more than fully open-ended reasoning.

Do not use this as-is when:
- The corpus is tiny enough for direct scan.
- You need broad synthesis across thousands of documents rather than targeted extraction.
- The fields are undefined or highly subjective.
- You have no indexing layer and no way to precompute chunks, metadata, or embeddings.

## The Core Mental Model

Quantum search uses superposition plus interference to boost the right answer.

Classical RAG cannot do that physically, but it can imitate the strategy operationally:

- **Superposition analogue**: generate multiple representations of the same question at once.
- **Interference analogue**: promote documents that score well across independent retrieval channels.
- **Amplitude amplification analogue**: spend more compute on candidates that repeatedly survive each stage.
- **Quantum-walk analogue**: expand along document, entity, citation, transaction, temporal, or workflow edges to discover answer-determinative paths.
- **Measurement analogue**: do final extraction only after the candidate set has collapsed to a tiny frontier of spans or chains.

In disclosure language, candidate evidence chains are the search states and the user question compiles into a query-conditioned oracle. In the current classical implementation, that oracle is a bounded scoring function over precomputed indexes and evidence-graph features. Its cost must not scale with the full corpus at query time.

## Architecture Pattern

### Stage 0: Normalize the target schema

Before retrieval, define the exact slots to extract.

Example:
- `company_name`
- `headquarters_city`
- `latest_revenue`
- `ceo_name`
- `founding_year`
- `primary_product`

For each field, write:
- aliases
- expected data type
- likely lexical patterns
- likely source document types
- hard constraints

Example field spec:

```yaml
latest_revenue:
  type: money
  aliases: [revenue, annual revenue, FY revenue, sales]
  patterns:
    - '(revenue|sales).{0,40}(USD|\$|million|billion)'
  likely_sources: [annual report, investor presentation, financial statement]
  constraints:
    - prefer latest dated mention
    - reject projected values unless explicitly requested
```

This step matters because retrieval quality is downstream of field definition quality.

### Stage 1: Create many query views in parallel

Do not retrieve with one query.

Generate several views:
- raw user query
- dense semantic paraphrase
- keyword-heavy boolean query
- entity-only query
- field-specific subqueries, one per slot
- time-constrained query if recency matters
- document-type query, e.g. `site:annual report` style logic if available

For `latest_revenue` you might issue all of these:
- `Acme latest revenue`
- `Acme annual sales FY2025`
- `Acme investor presentation revenue`
- `company:Acme field:revenue doc_type:financial`

Treat each query view like a parallel probe into the corpus.

### Stage 2: Retrieve through multiple channels

Use more than one retrieval method.

Recommended channels:
- BM25 / full-text lexical search
- dense vector retrieval
- metadata-filtered retrieval
- entity index lookup
- exact pattern / regex retrieval for structured fields
- graph or citation-neighbor expansion if available

For each query view, retrieve top-N from each channel. Then union the candidates.

A good practical pattern:
- lexical top 50
- dense top 50
- metadata-constrained top 20
- exact-pattern top 20
- dedupe by document id and chunk id

This is the "classical superposition" phase: many weak retrievals, none trusted alone.

### Stage 3: Fuse scores instead of trusting one ranker

The winning documents are often not rank-1 in any single method. They are the ones that keep showing up.

Use fusion such as:
- reciprocal rank fusion
- weighted score sum
- channel voting count
- exact-match bonus
- recency bonus
- document-type prior

Useful scoring features:
- appears in both lexical and dense retrieval
- contains named entity exact match
- contains field alias near value pattern
- source document type matches expectation
- date is recent enough
- multiple chunks from same doc support same field

This is where "interference" happens operationally: agreement strengthens candidates.

### Stage 4: Collapse to evidence spans and paths, not documents

Do not hand whole documents to the LLM if you can avoid it.

Convert candidate docs into candidate spans:
- sentence windows around exact matches
- paragraph windows around field aliases
- table rows or bullet items if structured docs exist
- neighboring chunks around reranked hits

Then rerank spans with a cross-encoder or a strong reranker using:
- query-field relevance
- answerability
- local evidence density
- contradiction risk

When the question requires more than one supporting object, expand candidates into short evidence paths:
- transaction links, such as invoice to PO to approval
- contract links, such as MSA to amendment to order form
- security links, such as alert to host to process to vulnerability
- scientific links, such as claim to method to result to citation
- policy links, such as request to rule to exception to approver

Score paths by:
- coverage of required roles
- source quality of each link
- temporal consistency
- entity consistency
- contradiction risk
- whether any required link is missing

Target outcome:
- from 1,000,000 docs → ~200 docs
- from ~200 docs → ~500 spans
- from ~500 spans → top 10-30 spans or paths per field

### Stage 5: Extract with a fixed schema, one field at a time

Extraction should be narrow and typed.

Bad prompt:
- `Read these documents and tell me everything important.`

Good prompt:
- `From the evidence spans only, extract latest_revenue as a single normalized amount. Return null if absent. Include citation ids.`

Run extraction per field or per tightly related field group.

Require output like:

```json
{
  "latest_revenue": {
    "value": "USD 12.4 billion",
    "normalized": 12400000000,
    "citation_ids": ["doc_882:chunk_14", "doc_1034:chunk_3"],
    "confidence": 0.93,
    "notes": "Two sources agree; one is annual report dated 2025-02-11"
  }
}
```

### Stage 6: Verify by contradiction search

Before finalizing, actively look for conflicting evidence.

For each extracted field:
- run a targeted contradiction query
- compare alternate dates/units/entities
- prefer primary sources over secondary mentions
- apply recency and provenance rules

This is the difference between fast extraction and trustworthy extraction.

## Fast Path for Million-Document Corpora

If speed is the requirement, the pipeline should be engineered around these rules:

### 1. Precompute aggressively
- chunk once, not per query
- store dense embeddings ahead of time
- maintain BM25/full-text index
- extract metadata and entities during ingest
- precompute document type, dates, and source quality

### 2. Search narrow before you reason
- first reduce by entity / metadata / date / doc type
- only then run semantic retrieval and reranking

### 3. Spend expensive compute late
- embeddings and lexical search first
- reranker second
- LLM extraction last

### 4. Batch by field
- query all candidate spans for `ceo_name` together
- query all candidate spans for `founding_year` together
- keep prompts schema-bound and short

### 5. Early exit when confidence is high
If two high-quality sources agree and no contradiction appears, stop searching that field.

### 6. Parallelize independent fields
The 5-6 fields can often be searched in parallel, then merged.

## Recommended Retrieval Recipe

For each user request:

1. Parse entities, time constraints, and the 5-6 target fields.
2. Generate 5-12 query variants.
3. Run retrieval across 3-5 channels.
4. Fuse and dedupe into top candidate docs.
5. Slice docs into evidence spans.
6. Rerank spans per field.
7. Extract typed values with citations.
8. Run contradiction checks.
9. Return final JSON plus evidence summary.

## Practical Scoring Formula

A simple starting score:

```text
candidate_score =
  0.30 * lexical_score +
  0.25 * dense_score +
  0.15 * reranker_score +
  0.10 * exact_entity_bonus +
  0.10 * field_pattern_bonus +
  0.05 * source_quality_bonus +
  0.05 * recency_bonus
```

Then add a survival bonus if the same candidate appears across multiple channels.

## Prompt Pattern for Extraction

Use a strict prompt shape:

```text
You are extracting one field from evidence.
Field: latest_revenue
Allowed output: JSON only
Return null if unsupported.
Prefer primary sources and latest explicit dated values.
Do not infer from partial hints.
Evidence spans:
[span_id] ...
[span_id] ...
```

Required JSON:

```json
{
  "field": "latest_revenue",
  "value": null,
  "normalized": null,
  "citation_ids": [],
  "confidence": 0.0,
  "reason": "No explicit supported value found"
}
```

## System Design Notes

For a production system, the stack usually looks like this:

- ingest pipeline → chunking, OCR, metadata, entity extraction
- lexical index → OpenSearch / Elasticsearch / Tantivy / Postgres FTS
- vector index → FAISS / pgvector / Milvus / Weaviate / Qdrant
- reranker → cross-encoder or lightweight LLM judge
- extraction layer → schema-constrained LLM or typed parser
- result store → structured JSON with provenance

If the corpus is truly large, keep hot indexes separated from raw blob storage.

## DSPy / Programmatic Pattern

If implementing with DSPy or a similar orchestration layer, split the modules into:
- `GenerateQueryViews`
- `RetrieveCandidates`
- `FuseCandidates`
- `RerankEvidenceSpans`
- `ExtractField`
- `VerifyContradictions`
- `AssembleResult`

Optimize these modules against a labeled extraction set, not generic QA metrics.

Good metrics:
- field exact match
- citation correctness
- latency per request
- contradiction catch rate
- null precision when value is absent

## Common Pitfalls

1. **Using one generic query.**
   One query misses alias and document-type variation. Fan out.

2. **Letting the LLM read too much.**
   If the model sees full documents too early, latency rises and precision drops.

3. **Skipping metadata during ingest.**
   Without entities, dates, and document type, you lose cheap narrowing.

4. **Ranking documents instead of spans.**
   The right document can still hide the wrong paragraph. Collapse to spans before extraction.

5. **Returning answers without contradiction checks.**
   Many corpora contain stale values, projections, mirrored pages, and summaries.

6. **Treating all fields the same.**
   Revenue, person names, dates, and product names need different patterns and priors.

7. **No null discipline.**
   The system must cleanly return `null` when evidence is insufficient.

8. **Optimizing only for accuracy.**
   In production, latency and evidence traceability matter just as much.

## Verification Checklist

- [ ] Target fields are explicitly defined with types and aliases
- [ ] Query fan-out is implemented, not a single prompt/search
- [ ] At least two retrieval channels are used
- [ ] Fusion rewards cross-channel agreement
- [ ] Reranking happens at span level
- [ ] Extraction is schema-bound with citations
- [ ] Contradiction search runs before final answer
- [ ] Outputs can return null cleanly
- [ ] Latency is measured separately for retrieval, reranking, and extraction
- [ ] Final output includes value, confidence, and provenance for each field

## One-Shot Recipe

When someone says: `Find 6 fields for company X across a million docs fast`

Use this plan:

1. Define the six fields as a strict JSON schema.
2. Generate multiple query views for the entity and each field.
3. Run lexical + dense + metadata retrieval in parallel.
4. Fuse results and keep only the top candidate docs.
5. Slice to local evidence spans around aliases/patterns.
6. Rerank spans per field.
7. Extract each field independently with citations.
8. Run contradiction checks and normalize units.
9. Return compact JSON plus a human-readable evidence summary.

## Output Contract

Preferred final shape:

```json
{
  "entity": "Acme Corp",
  "answer": "Latest explicit revenue is USD 12.4 billion.",
  "evidence_chain": ["doc_882:chunk_14", "doc_1034:chunk_3"],
  "missing_evidence": [],
  "fields": {
    "ceo_name": {
      "value": "Jane Smith",
      "citation_ids": ["doc_19:chunk_4"],
      "confidence": 0.98
    },
    "latest_revenue": {
      "value": "USD 12.4 billion",
      "normalized": 12400000000,
      "citation_ids": ["doc_882:chunk_14", "doc_1034:chunk_3"],
      "confidence": 0.93
    }
  },
  "latency_ms": {
    "retrieval": 420,
    "reranking": 180,
    "extraction": 640,
    "total": 1410
  }
}
```

If the user needs code next, scaffold the pipeline around this exact stage order rather than starting from a monolithic chatbot RAG design.
