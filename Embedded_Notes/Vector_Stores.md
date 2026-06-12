A vector store is a data system optimized for searching. Given a query vector, find the 'k' most similar vectors from a stored corpus. The core challenge is doing this at scale without exhaustively computing distance to every stored vector.

## The Core Problem

If you have `N` vectors of dimension `d`, brute-force nearest neighbor search is `O(N × d)`. At N=10M and d=1536, that's ~15 billion operations per query. Unacceptable for real-time retrieval.

Vector stores solve this with **Approximate Nearest Neighbor (ANN)** algorithms — trading a small accuracy loss for orders-of-magnitude speedup.

## How They Work Internally

### HNSW: Hierarchical Navigable Small World

The dominant algorithm in production systems (used by Pinecone, Qdrant, Weaviate).

**Mental model:** Think of it as a multi-floor building. Upper floors are a coarse "highway" graph — few nodes, long-range connections. Lower floors are dense local neighborhoods. Search starts at the top (fast, coarse), navigates down to precision.

```
Layer 2 (sparse):    A ────────── E
Layer 1 (medium):    A ── B ── D ── E
Layer 0 (dense):     A─B─C─D─E─F─G─H   ← all vectors live here
```

**Search:** Start at top layer, greedily navigate toward the query, then descend. `O(log N)` average complexity.

**Insert:** Each new vector is probabilistically assigned a max layer, then connected to its nearest neighbors at each layer it appears in.

Key parameters:

| Param | What it controls |
|---|---|
| `M` | Max connections per node per layer — higher = better recall, more memory |
| `ef_construction` | Search width during index build — higher = better index quality, slower build |
| `ef_search` | Search width at query time — tunable recall/speed trade-off |

### IVF: Inverted File Index

**Mental model:** K-means clustering over the corpus. Divide vectors into `nlist` Voronoi cells. At query time, find the nearest `nprobe` cluster centroids, then search only those clusters.

```
Query → find nearest centroids → search only those clusters → return top-k
```

Trade-off: Fast and memory-efficient, but cluster boundaries cause edge cases — a vector just outside the `nprobe` cells gets missed entirely.

### PQ: Product Quantization

A *compression* technique, not a search algorithm. Splits each vector into `m` sub-vectors and quantizes each to a codebook entry. Reduces memory by 8–32×.

Used in combination with IVF → **IVF-PQ**, the standard for billion-scale datasets (FAISS default for large corpora).

## Anatomy of a Vector Store

```
┌─────────────────────────────────────────┐
│              Vector Store               │
│                                         │
│  ┌─────────┐    ┌──────────────────┐    │
│  │  Index  │    │  Payload / Meta  │    │
│  │ (HNSW,  │◄──►│ (doc_id, text,   │    │
│  │  IVF…)  │    │  source, date…)  │    │
│  └─────────┘    └──────────────────┘    │
│                                         │
│  Operations: upsert · query · delete    │
│              filter · update            │
└─────────────────────────────────────────┘
```

Metadata filtering is critical in practice. You rarely want to search the entire corpus. Example: filter by `source == "Q3_report.pdf"` before running ANN search.

Two strategies:

- **Pre-filtering**: Apply metadata filter first, then ANN search on the subset — accurate but slow if subset is small
- **Post-filtering**: ANN search first, then filter results — fast but can return fewer than `k` results
- **Filtered HNSW** (Qdrant's approach): Integrates filter into graph traversal — best of both worlds

---

## Key Metrics to Know

| Metric | Meaning |
|---|---|
| **Recall@k** | Fraction of true top-k neighbors found — main accuracy metric |
| **QPS** | Queries per second — throughput |
| **P99 latency** | 99th percentile query time — tail latency matters in prod |
| **Index build time** | One-time cost; matters at billion scale |
| **Memory footprint** | HNSW is memory-hungry; PQ compression trades accuracy for size |

The ANN benchmarks site ([ann-benchmarks.com](https://ann-benchmarks.com)) is the standard reference for comparing these trade-offs empirically.


## Where Vector Stores Sit in the Full RAG Stack

```
Documents
    │
    ▼
Chunking → Embedding Model → [Vectors + Metadata]
                                      │
                             ┌────────▼────────┐
                             │   Vector Store  │
                             │   (HNSW index)  │
                             └────────┬────────┘
                                      │  top-k chunks
                                      ▼
                              Query + Context → LLM → Answer
```

The vector store is the **retrieval backbone** — everything upstream (chunking, embedding quality) determines what goes in; everything downstream (reranking, prompting) determines how well what comes out gets used.
