Embeddings are the representation layer. They compress semantic meaning into vectors. RAG is the architecture that uses those vectors to fetch relevant context at inference time, grounding LLM outputs in external knowledge without retraining.

These two concepts are deeply coupled. RAG only works as well as the embeddings powering its retrieval.

---

## Embeddings
A function `E: text → ℝᵈ` that maps a chunk of text to a dense vector in `d`-dimensional space (typically `d = 768, 1024, or 1536`), where **semantic similarity ≈ geometric proximity**.

```python
# Conceptually:
E("king") - E("man") + E("woman") ≈ E("queen")  # the classic demo
```

The model learns this geometry during contrastive pretraining — similar texts are pulled together, dissimilar ones pushed apart.

### Similarity Metrics

| Metric | Formula | When to use |
|---|---|---|
| **Cosine similarity** | `cos(θ) = (A·B) / (‖A‖‖B‖)` | Most common; magnitude-invariant |
| **Dot product** | `A·B` | Faster; used when vectors are normalized |
| **L2 / Euclidean** | `‖A - B‖₂` | Less common in NLP; sensitive to magnitude |

Most vector DBs default to cosine or dot product.

### Embedding Models

| Model | Notes |
|---|---|
| `text-embedding-3-large` (OpenAI) | Strong general-purpose, 3072-dim |
| `e5-large`, `bge-large` (BAAI) | Open-source SOTA, often competitive |
| `all-MiniLM-L6-v2` (SBERT) | Lightweight, great for prototyping |
| Domain-specific fine-tuned | Best for medical, legal, code retrieval |

**Key insight:** The embedding model is often the bottleneck in RAG quality, not the LLM. A weak retriever means irrelevant context, and the LLM can't reason its way out of that.

---

## RAG

### Core Pipeline

```
Query
  │
  ▼
[Embed query] ──────────────────────────────────-┐
                                                 │
[Offline: Chunk → Embed → Store in Vector DB]    │
                                                 ▼
                                     [ANN Search → Top-k chunks]
                                                 │
                                                 ▼
                               [Inject into LLM prompt as context]
                                                 │
                                                 ▼
                                           [Generate answer]
```

### Chunking

How you split documents before embedding has an outsized effect on retrieval quality.

| Strategy | Trade-off |
|---|---|
| **Fixed-size** (e.g., 512 tokens) | Simple; can split mid-sentence or mid-concept |
| **Sentence/paragraph splitting** | Respects natural boundaries; variable size |
| **Semantic chunking** | Splits where topic shifts; best quality, most expensive |
| **Hierarchical (small-to-big)** | Store small chunks, retrieve parent context — good for precision + completeness |

**Common mistake:** Chunks that are too large dilute the signal; chunks that are too small lose context. Typical sweet spot: **256–512 tokens with ~20% overlap**.

---

## Advanced RAG Techniques

This is where it gets interesting. Naive RAG fails in practice — here's the upgrade path:

### Hybrid Search

Combine **dense retrieval** (embedding similarity) with **sparse retrieval** (BM25 keyword matching), then merge results.

```
score_hybrid = α × score_dense + (1 - α) × score_sparse
```

Why: Dense retrieval misses exact keyword matches ("GPT-4o" as a specific term). Sparse retrieval misses semantic paraphrases. Hybrid gets both.

### Reranking

After retrieving top-k chunks (say, k=20), run a **cross-encoder** reranker to re-score and keep top-n (say, n=5).

```
Bi-encoder (fast, ANN) → top-20 candidates
    → Cross-encoder (slower, more accurate) → top-5 final context
```

Cross-encoders (e.g., `cross-encoder/ms-marco-MiniLM`) jointly encode query + chunk — far more accurate than bi-encoders but too slow for full-corpus search.

### Query Transformations

Don't just embed the raw user query. Smarter options:

- **HyDE** (Hypothetical Document Embedding): Ask the LLM to *hallucinate* an ideal answer, embed *that*, then retrieve against it. The hypothetical answer is closer in embedding space to real answers than the question itself.
- **Multi-query**: Generate 3–5 paraphrases of the query, retrieve for each, union the results.
- **Step-back prompting**: Abstract the query to a higher-level concept before retrieving.

### RAG vs. Fine-tuning

| Situation | Recommendation |
|---|---|
| Knowledge changes frequently | RAG — no retraining needed |
| Knowledge is static + performance matters | Fine-tune |
| Need citations / grounding | RAG |
| Need to change *style or behavior* | Fine-tune |
| Need both | RAG + fine-tuned base model |

---

## Connection to Agentic Workflows

RAG is essentially a tool in an agentic system. The retriever is one callable action. In more advanced setups:

- The agent decides when to retrieve (not every turn needs RAG)
- It can retrieve multiple times with refined queries mid-task
- Memory modules in agents are often RAG pipelines over conversation history

```
Agent loop:
  Reason → "I need to look this up" → [RAG tool call] → Observe → Continue
```
