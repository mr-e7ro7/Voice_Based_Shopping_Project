Embeddings create vectors, and the vector stores organize and search them.

---


```
Text
 │
 │  Embedding Model (e.g., text-embedding-3-large)
 │  f: text → ℝᵈ
 ▼
Vector  ──────────────────────────► Vector Store
[0.12, -0.87, 0.34, ..., 0.61]      - Indexes the vector (HNSW/IVF)
 (1536 floats)                       - Stores metadata alongside it
                                     - Answers: "what's nearest to query?"
```



| | **Embeddings** | **Vector Store** |
|---|---|---|
| What it is | A model / function | A data system |
| Input | Raw text | Vectors + metadata |
| Output | Dense vector in ℝᵈ | Nearest neighbor results |
| Lives | In memory / API call | On disk / in-memory index |
| Cares about | *Meaning* of text | *Geometry* of vectors |
| Swappable? | Yes — swap the model | Yes — swap the DB |


## The Relationship

 **a vector store is completely model-agnostic**. It just stores and searches float arrays. It doesn't know or care what generated them.

Hence it is critical that **You use the same embedding model at index time and query time.**

If you embed documents with `bge-large` (1024-dim) but query with `text-embedding-3-large` (1536-dim), two things break:
1. Dimension mismatch causes the store to reject the query vector.
2. Even with matching dimensions, different models produce incompatible geometric spaces

```python
# Index time:
store.upsert(id="doc1", vector=bge_model.encode("cats are mammals"))

# Query time: WRONG model:
results = store.query(vector=openai_model.embed("what are cats?"))
# Dimensions may match by coincidence, but the spaces don't align
# You'll get garbage results with no error thrown
```


## End-to-End Flow With Both

```
──── OFFLINE (Index Time) ────────────────────────────────────

Raw Docs
   │
   ├─► Chunking
   │       │
   │       ▼
   │   ["Cats are mammals", "Dogs are loyal", ...]
   │       │
   │       ▼  Embedding Model
   │   [[0.12, -0.87, ...], [0.44, 0.21, ...], ...]
   │       │
   │       ▼
   └──► Vector Store (upsert vectors + metadata)


──── ONLINE (Query Time) ─────────────────────────────────────

User Query: "What animal is a cat?"
   │
   ▼  Same Embedding Model
[0.11, -0.85, ...]     ← query vector
   │
   ▼
Vector Store.query(vector, top_k=5)
   │
   ▼
["Cats are mammals", ...]   ← retrieved chunks
   │
   ▼
LLM prompt = query + retrieved chunks → Answer
```


## Quality Dependency

Vector store retrieval quality is **upper-bounded by embedding quality**. No amount of ANN optimization rescues a weak embedding model.

```
Retrieval quality = f(embedding quality, chunking strategy, index config)

If embedding model is poor:
  → Semantically similar texts map to distant vectors
  → ANN search returns irrelevant chunks
  → LLM hallucinates or gives wrong answers
  → Root cause is invisible without eval
```

This is why embedding model selection is a first-class decision in RAG system design, not an afterthought.


## They're Independently Swappable

Because the contract is just "vectors in, vectors out," you can swap either layer without touching the other — as long as you **re-embed the entire corpus** when you change the embedding model.

```
v1: bge-large + Qdrant
         ↓ swap model
v2: text-embedding-3-large + Qdrant  ← must re-index all docs
         ↓ swap store
v3: text-embedding-3-large + Pinecone  ← just migrate vectors, no re-embed
```
