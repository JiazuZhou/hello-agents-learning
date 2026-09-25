# Day 04 — Chapter 8: Memory & Retrieval (记忆与检索)

## Why this chapter exists

LLMs have two fundamental limitations that Chapter 8 addresses:

1. **Statelessness → conversation amnesia**: every API call is independent; the model
   doesn't remember prior sessions. `SimpleAgent`'s `_history` only lives inside one
   process/instance — no persistence, no long-term retrieval, no forgetting/consolidation.
2. **Static, bounded built-in knowledge**: training-data cutoff, weak domain depth,
   hallucination, no source attribution.

The fixes: a **Memory System** (for limitation 1) and **RAG** (for limitation 2), both
implemented as standard tools (`MemoryTool`, `RAGTool`) following Chapter 7's
"everything is a tool" design principle — no new Agent classes.

---

## 1. Cognitive science foundation

Human memory model (Atkinson & Shiffrin) that the design mimics:

- **Sensory memory**: 0.5–3s, huge capacity
- **Working memory**: 15–30s, ~7±2 items
- **Long-term memory**: procedural + declarative (semantic = facts/concepts,
  episodic = personal events)

Memory lifecycle mapped to system operations:
**Encoding → Storage → Retrieval → Consolidation → Forgetting**

## 2. Memory system architecture (4 layers)

```
Infrastructure : MemoryManager / MemoryItem / MemoryConfig / BaseMemory
Memory types   : Working / Episodic / Semantic / Perceptual
Storage        : Qdrant (vectors) / Neo4j (graph) / SQLite (documents)
Embedding      : DashScope (cloud) / LocalTransformer / TF-IDF (fallback)
```

### The four memory types

| Type | Stores | Backend | Lifecycle |
|---|---|---|---|
| **Working** | current-session context | pure in-memory | capacity ~50 + TTL auto-expiry |
| **Episodic** | concrete events, time-ordered | SQLite + Qdrant | long-term, session-indexed |
| **Semantic** | abstract knowledge, concepts, user preferences | Qdrant + **Neo4j knowledge graph** (auto entity/relation extraction via spaCy) | most persistent |
| **Perceptual** | multimodal (image via CLIP, audio via CLAP) | per-modality Qdrant collections | importance/space-managed |

### Scoring formulas (worth memorizing the pattern)

All types share the shape: `relevance × importance_weight`, where
**importance_weight = 0.8 + importance × 0.4** (range [0.8, 1.2] — importance nudges
but never dominates similarity).

- Working: `(TF-IDF·0.7 + keyword·0.3) × time_decay × weight`
- Episodic: `(vector·0.8 + recency·0.2) × weight` — recency matters for events
- Semantic: `(vector·0.7 + graph·0.3) × weight` — graph reasoning supplements
- Perceptual: `(vector·0.8 + recency·0.2) × weight`; recency = exponential decay
  (forgetting curve), floor 0.1

### MemoryTool — unified `execute(action, **kwargs)` interface

Key actions:

- **add** — auto session-ID, timestamps, modality inference; `importance` ∈ [0,1]
- **search** — semantic retrieval, filter by `memory_types` / `min_importance`
- **forget** — 3 strategies: `importance_based` (below threshold),
  `time_based` (older than N days), `capacity_based` (evict least important)
- **consolidate** — promote short→long term (e.g. working→episodic when
  importance ≥ 0.7; episodic→semantic at ≥ 0.8) — mimics memory consolidation
- plus: summary / stats / update / remove / clear_all

Layering: `MemoryTool` (interface + param handling) → `MemoryManager`
(coordination, per-type enable flags) → memory type classes → storage backends.

## 3. RAG system

**Retrieval-Augmented Generation** = retrieve from external KB → inject into prompt →
generate grounded, citable answers.

Evolution: **Naive RAG** (2020–21, TF-IDF/BM25 retrieve-read) → **Advanced RAG**
(2022–23, dense embeddings + query rewriting/chunking/reranking) → **Modular RAG**
(2023–, pluggable modules, hybrid retrieval, self-reflection).

### Pipeline ("any format in, Markdown as the universal intermediate")

```
Any document → MarkItDown → Markdown → structure-aware chunking → embedding → Qdrant
```

- **MarkItDown** (Microsoft OSS) converts PDF/Office/images(OCR)/audio(transcription)
  to Markdown; enhanced path for PDFs; plain-text fallback.
- **Chunking** exploits Markdown heading hierarchy (`#`/`##`/`###`) to split on
  semantic boundaries, keeps a `heading_path` per chunk, then packs paragraphs into
  token-budgeted chunks with **overlap** for continuity. CJK-aware token estimate
  (1 CJK char ≈ 1 token; other text by whitespace split).
- **Embedding**: unified interface shared with memory system; DashScope → local
  sentence-transformers → TF-IDF fallback chain. Namespace isolation in Qdrant
  (`rag_namespace`) for multi-tenant KBs.

### Advanced retrieval strategies

1. **MQE (Multi-Query Expansion)** — LLM generates N paraphrases of the query;
   parallel retrieval, merged. Fixes vocabulary mismatch; +30–50% recall.
2. **HyDE (Hypothetical Document Embeddings)** — "answer to find the answer":
   LLM writes a hypothetical answer paragraph, whose embedding sits closer to real
   documents than the question does. Great for jargon-heavy domains.
3. **Unified expanded-search framework** — expand → retrieve per query
   (candidate pool = top_k × 4) → dedupe by max score → return top-k.
   Toggle via `enable_mqe` / `enable_hyde`; use both for recall-critical work,
   plain vector search when latency matters.

## 4. Capstone: PDF learning assistant (`11_Q&A_Assistant.py`, Gradio)

`PDFLearningAssistant` combines both tools in a closed loop:

- **load_document** → RAGTool `add_document` (chunk_size=1000, overlap=200);
  logs an *episodic* memory ("loaded doc X", importance 0.9)
- **ask** → question logged to *working* memory → RAGTool `ask` with MQE+HyDE →
  interaction logged to *episodic* memory
- **add_note** → *semantic* memory (importance 0.8, tagged by concept)
- **recall / get_stats / generate_report** → memory search + JSON learning report

Design takeaways: per-user isolation via `user_id` (memory) + `rag_namespace` (KB);
session tracking via `session_id`; RAG answers "what does the document say",
Memory answers "what happened / what do I know about the user".

## 5. Practical notes from my own run

- Chapter pins `hello-agents[all]==0.2.0`; I used **0.2.9** (last 0.2.x) on
  **Python 3.11** — 1.0.0 dropped the memory subsystem entirely.
- Embedding default is `dashscope`; without an API key the fallback chain has a
  bug (dashscope model name leaks into local/tfidf constructors). Fix: set
  `EMBED_MODEL_TYPE=local` (+ default `all-MiniLM-L6-v2`) to run fully offline.
- Full cloud setup additionally needs Qdrant (vectors) and Neo4j (graph) creds in
  `.env`; local demos work with SQLite + local embeddings.

## Key takeaways

1. Memory ≠ RAG: **memory is the agent's own experience** (write-heavy, lifecycle
   managed); **RAG is external knowledge** (read-heavy, provenance matters).
2. The universal scoring pattern `relevance × (0.8 + importance × 0.4)` keeps
   importance influential but subordinate to similarity.
3. Forgetting and consolidation are features, not bugs — unbounded memory degrades
   retrieval quality and cost.
4. Converting everything to Markdown first makes chunking structure-aware and the
   whole pipeline format-agnostic.
5. MQE and HyDE both attack the query↔document semantic gap, from opposite ends
   (diversify the question vs. approximate the answer).

## Next step

- Chapter 9: Context Engineering (`hello-agents[all]==0.2.8`) — how to feed all of
  this memory/retrieval into the prompt effectively.
