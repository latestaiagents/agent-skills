---
name: hybrid-retrieval
description: |
  Implement hybrid search combining vector and keyword retrieval for RAG systems.
  Use this skill when building RAG retrieval, combining semantic search with BM25,
  implementing reciprocal rank fusion (RRF), or optimizing retrieval accuracy.
  Activate when: vector search, keyword search, BM25, semantic search, hybrid RAG,
  retrieval optimization, search relevance, reranking.
---

# Hybrid Retrieval for RAG

**Combine vector similarity with keyword matching for superior retrieval accuracy.**

## Why Hybrid is Mandatory in 2026

Vector search alone misses:
- Exact matches (product codes, IDs, names)
- Rare terms not well-represented in embeddings
- Keyword-specific queries ("error code E-5012")

Keyword search alone misses:
- Semantic similarity ("car" vs "automobile")
- Context and meaning
- Paraphrased content

**Hybrid search combines both for 15-25% better recall.**

## Core Patterns

### Pattern 1: Reciprocal Rank Fusion (RRF)

The standard for combining ranked results from multiple retrievers:

```python
def reciprocal_rank_fusion(
    results_lists: list[list[dict]],
    k: int = 60
) -> list[dict]:
    """
    Combine multiple ranked result lists using RRF.

    Args:
        results_lists: List of ranked results from different retrievers
        k: Ranking constant (default 60, higher = more weight to lower ranks)

    Returns:
        Fused and re-ranked results
    """
    fused_scores = {}

    for results in results_lists:
        for rank, doc in enumerate(results):
            doc_id = doc["id"]
            if doc_id not in fused_scores:
                fused_scores[doc_id] = {"doc": doc, "score": 0}
            # RRF formula: 1 / (k + rank)
            fused_scores[doc_id]["score"] += 1 / (k + rank + 1)

    # Sort by fused score
    sorted_results = sorted(
        fused_scores.values(),
        key=lambda x: x["score"],
        reverse=True
    )
    return [item["doc"] for item in sorted_results]
```

### Pattern 2: LangChain Ensemble Retriever

```python
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever
from langchain_community.vectorstores import Chroma

# Create vector retriever
vectorstore = Chroma.from_documents(documents, embeddings)
vector_retriever = vectorstore.as_retriever(search_kwargs={"k": 10})

# Create BM25 retriever
bm25_retriever = BM25Retriever.from_documents(documents)
bm25_retriever.k = 10

# Combine with weights
ensemble_retriever = EnsembleRetriever(
    retrievers=[bm25_retriever, vector_retriever],
    weights=[0.4, 0.6]  # Tune based on your data
)

# Use in RAG chain
results = ensemble_retriever.invoke("your query here")
```

### Pattern 3: LlamaIndex Hybrid Search

```python
from llama_index.core import VectorStoreIndex, StorageContext
from llama_index.core.retrievers import QueryFusionRetriever
from llama_index.retrievers.bm25 import BM25Retriever

# Build index
index = VectorStoreIndex.from_documents(documents)

# Create retrievers
vector_retriever = index.as_retriever(similarity_top_k=10)
bm25_retriever = BM25Retriever.from_defaults(
    nodes=index.docstore.docs.values(),
    similarity_top_k=10
)

# Fusion retriever with query expansion
retriever = QueryFusionRetriever(
    retrievers=[vector_retriever, bm25_retriever],
    similarity_top_k=10,
    num_queries=4,  # Generate 4 query variations
    mode="reciprocal_rerank",
    use_async=True,
)
```

### Pattern 4: Direct Implementation with Qdrant

```python
from qdrant_client import QdrantClient
from qdrant_client.models import SparseVector, NamedSparseVector, NamedVector

client = QdrantClient(url="http://localhost:6333")

# Hybrid search with both dense and sparse vectors
results = client.query_points(
    collection_name="documents",
    prefetch=[
        # Dense vector search
        models.Prefetch(
            query=dense_embedding,  # [0.1, 0.2, ...]
            using="dense",
            limit=20
        ),
        # Sparse vector search (BM25-style)
        models.Prefetch(
            query=SparseVector(
                indices=[1, 42, 123],  # Token IDs
                values=[0.5, 0.8, 0.3]  # Token weights
            ),
            using="sparse",
            limit=20
        ),
    ],
    query=models.FusionQuery(fusion=models.Fusion.RRF),
    limit=10
)
```

## Reranking for Better Precision

After hybrid retrieval, rerank for final ordering:

### Cross-Encoder Reranking

```python
from sentence_transformers import CrossEncoder

# Load reranker model
reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

def rerank_results(query: str, documents: list[str], top_k: int = 5):
    """Rerank documents using cross-encoder."""
    # Create query-document pairs
    pairs = [[query, doc] for doc in documents]

    # Score all pairs
    scores = reranker.predict(pairs)

    # Sort by score
    scored_docs = list(zip(documents, scores))
    scored_docs.sort(key=lambda x: x[1], reverse=True)

    return [doc for doc, score in scored_docs[:top_k]]
```

### Cohere Rerank API

```python
import cohere

co = cohere.Client("your-api-key")

def cohere_rerank(query: str, documents: list[str], top_k: int = 5):
    """Rerank using Cohere's rerank endpoint."""
    response = co.rerank(
        model="rerank-english-v3.0",
        query=query,
        documents=documents,
        top_n=top_k,
        return_documents=True
    )
    return [result.document.text for result in response.results]
```

## Weight Tuning Guidelines

| Data Type | Vector Weight | Keyword Weight |
|-----------|---------------|----------------|
| Technical docs | 0.5 | 0.5 |
| Legal/compliance | 0.4 | 0.6 |
| Creative content | 0.7 | 0.3 |
| Product catalogs | 0.3 | 0.7 |
| Code repositories | 0.4 | 0.6 |

## Best Practices

1. **Always benchmark** - Test vector-only, keyword-only, and hybrid on your data
2. **Tune weights empirically** - Start at 0.5/0.5, adjust based on evaluation
3. **Use reranking** - Hybrid retrieval + reranking = best results
4. **Consider query type** - Route exact-match queries to keyword, semantic to vector
5. **Monitor latency** - Hybrid adds overhead; cache where possible

## Quick Decision Tree

```
Is the query an exact match (ID, code, name)?
├─ Yes → Keyword-heavy (0.3 vector / 0.7 keyword)
└─ No → Is it conceptual/semantic?
         ├─ Yes → Vector-heavy (0.7 vector / 0.3 keyword)
         └─ Mixed → Balanced (0.5 / 0.5) + reranking
```
