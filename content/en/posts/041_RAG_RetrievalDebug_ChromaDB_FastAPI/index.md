+++
date = '2026-03-22T15:25:11+09:00'
draft = false
title = 'Debugging RAG Retrieval: A PoC with FastAPI + ChromaDB'
categories = ["AI"]
tags = ["RAG", "Retrieval", "ChromaDB", "FastAPI", "Debugging"]
+++

# How to Fix “Outdated Results Ranking Above Current Docs” in RAG



---

## Background

When building a knowledge bot using RAG (Retrieval-Augmented Generation), a common issue emerges:

> **Outdated documents rank higher than the latest ones**

This is not just a data issue—it is fundamentally a **retrieval design problem**.

In this PoC, using a minimal stack of FastAPI + ChromaDB, I reproduced the issue and walked through:

> **reproduction → root cause analysis → minimal fix**

---

## The Problem (Reproduced)

We prepare two documents:

* **v4 (current)**: Official documentation (correct but less query-aligned wording)
* **v3 (legacy)**: Blog content (outdated but strongly matches the query)

Query:

```
Which guidance should I prefer when wrangler v3 and v4 instructions conflict?
```

### Result (Failure Case)

* v3 (outdated) ranks higher
* v4 (correct) is pushed down

The reason is straightforward:

> **Embedding similarity reflects semantic closeness, not correctness**

---

## Root Cause Breakdown

This issue typically arises from three factors:

### 1. Semantic Similarity Bias

Older content that closely matches the query wording is ranked higher.

---

### 2. Lack of Retrieval Scope Control

The query is executed against the entire dataset without restriction.

---

### 3. Data Hygiene Issues (Duplicates / Legacy Residue)

Older versions of documents remain in the index and continue to compete.

---

## Approach: Minimal, Targeted Fixes

This PoC avoids large architectural changes and focuses on **surgical fixes**.

---

### ① Control Retrieval Scope with Metadata Filters

Use ChromaDB’s `where` clause to constrain the search space:

```json
{
  "source_type": "official_docs",
  "version_major": 4
}
```

Internally applied as an `$and` condition:

```python
where = {
  "$and": [
    {"source_type": "official_docs"},
    {"version_major": 4}
  ]
}
```

#### Effect

* Completely excludes legacy blog content (v3)
* Restricts retrieval to only valid, current documents

---

### ② Idempotent Ingestion (Prevent Duplicates)

```python
# Delete existing chunks for the same doc_id before ingest
collection.delete(where={"doc_id": doc_id})
```

#### Effect

* Prevents duplicate entries
* Avoids stale chunks remaining in the index

---

### ③ Debug Endpoint for Observability

```
POST /debug/retrieval
```

Returns:

* documents
* metadata
* distance (similarity score)

#### Example Output

```json
{
  "results": [
    {
      "doc_id": "wrangler_v4",
      "distance": 0.21,
      "metadata": {
        "source_type": "official_docs",
        "version_major": 4
      }
    }
  ],
  "where": {
    "$and": [...]
  }
}
```

#### Effect

* Makes retrieval decisions explainable
* Eliminates black-box behavior

---

## Before / After

| State                | Result                         |
| -------------------- | ------------------------------ |
| Before               | v3 (outdated) ranks higher     |
| After (with filters) | Only v4 (current) is retrieved |

---

## Key Design Insights

### 1. Retrieval Is Not Just “Search”—It’s Control

Relying purely on similarity leads to failure.

* similarity → ranking
* metadata → scope control

These must be handled separately.

---

### 2. Compensate in Application Layer When DB Falls Short

Chroma limitations:

* Limited support for comparisons like `published_at >= ...`

Workaround:

```python
# Post-filtering in application layer
if doc["published_at"] >= min_published_at:
    filtered.append(doc)
```

---

### 3. Build Debugging Capabilities Early

Adding observability later is expensive.

In this PoC:

* Debug endpoint enabled only in dev (`DEBUG_MODE=1`)
* Disabled in production

---

## Why This PoC Matters

This setup directly addresses real-world requests such as:

* “Prevent outdated information from being returned”
* “Explain why this result was selected”
* “RAG quality is inconsistent”

Most importantly:

> **It solves the issue without requiring major redesign**

---

## Summary

In one sentence:

> **Most RAG quality issues can be resolved through retrieval scope control and data hygiene**

Concretely:

* metadata filters (`where`)
* idempotent ingestion
* debug endpoint

These three elements allow you to **reproduce, diagnose, and fix** outdated retrieval issues in a consistent way.

---

## Next Steps (Extensions)

* Re-ranking (cross-encoders)
* Hybrid search (BM25 + embeddings)
* Time decay (boost newer content)

However:

> **Always verify whether simple filtering solves the issue first**

---

## Repository

[This](https://github.com/lat35n-hrt/rag_retrieval_diagnostics_poc) PoC includes:


* FastAPI (single-file implementation)
* ChromaDB (local persistence)
* Reproducible workflow using curl

---

This minimal yet practical setup demonstrates how to systematically eliminate a common RAG failure mode. Hope it helps in your own debugging and system design.