---
name: chunking-strategies
description: |
  Optimize document chunking for RAG performance and retrieval quality.
  Use this skill when splitting documents, choosing chunk sizes, implementing
  semantic chunking, or improving RAG retrieval accuracy.
  Activate when: chunking, split documents, chunk size, text splitting,
  document processing, RAG performance, semantic chunking, overlap.
---

# Chunking Strategies for RAG

**Optimal chunking is the difference between good and great RAG performance.**

## Why Chunking Matters

Poor chunking causes:
- Context fragmentation (answers split across chunks)
- Irrelevant retrieval (too much noise in chunks)
- Lost relationships (parent-child content separated)
- Wasted tokens (chunks too large or too small)

## Chunking Methods Comparison

| Method | Best For | Chunk Quality | Implementation |
|--------|----------|---------------|----------------|
| Fixed-size | Simple docs, uniform content | Medium | Easy |
| Recursive | Structured docs, markdown | High | Medium |
| Semantic | Complex docs, varied content | Highest | Complex |
| Parent-child | Hierarchical docs | High | Medium |
| Late chunking | Preserving context | Highest | Complex |

## Pattern 1: Fixed-Size with Overlap

The baseline approach - simple but effective:

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

def create_fixed_chunks(
    text: str,
    chunk_size: int = 512,
    chunk_overlap: int = 50
) -> list[str]:
    """
    Split text into fixed-size chunks with overlap.

    Guidelines:
    - chunk_size: 256-1024 tokens (512 is solid default)
    - overlap: 10-20% of chunk_size
    """
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=chunk_size,
        chunk_overlap=chunk_overlap,
        length_function=len,
        separators=["\n\n", "\n", ". ", " ", ""]
    )
    return splitter.split_text(text)
```

## Pattern 2: Semantic Chunking

Group by meaning, not arbitrary boundaries:

```python
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai import OpenAIEmbeddings

def create_semantic_chunks(text: str) -> list[str]:
    """
    Split text based on semantic similarity between sentences.
    Keeps related content together.
    """
    embeddings = OpenAIEmbeddings()

    splitter = SemanticChunker(
        embeddings=embeddings,
        breakpoint_threshold_type="percentile",
        breakpoint_threshold_amount=95  # Higher = fewer, larger chunks
    )

    return splitter.split_text(text)
```

### Custom Semantic Chunking

```python
import numpy as np
from sentence_transformers import SentenceTransformer

def semantic_chunk(
    sentences: list[str],
    model_name: str = "all-MiniLM-L6-v2",
    threshold: float = 0.5
) -> list[list[str]]:
    """
    Group sentences by semantic similarity.
    """
    model = SentenceTransformer(model_name)
    embeddings = model.encode(sentences)

    chunks = []
    current_chunk = [sentences[0]]

    for i in range(1, len(sentences)):
        # Cosine similarity between consecutive sentences
        sim = np.dot(embeddings[i-1], embeddings[i]) / (
            np.linalg.norm(embeddings[i-1]) * np.linalg.norm(embeddings[i])
        )

        if sim >= threshold:
            current_chunk.append(sentences[i])
        else:
            chunks.append(current_chunk)
            current_chunk = [sentences[i]]

    chunks.append(current_chunk)
    return chunks
```

## Pattern 3: Parent-Child Chunking

Retrieve small, return with context:

```python
from llama_index.core.node_parser import (
    HierarchicalNodeParser,
    SentenceSplitter,
    get_leaf_nodes
)
from llama_index.core import StorageContext, VectorStoreIndex
from llama_index.core.retrievers import AutoMergingRetriever

def create_hierarchical_index(documents):
    """
    Create parent-child chunk hierarchy.
    Small chunks for retrieval, auto-merge to parents for context.
    """
    # Define chunk sizes for each level
    node_parser = HierarchicalNodeParser.from_defaults(
        chunk_sizes=[2048, 512, 128]  # Parent → Child → Leaf
    )

    nodes = node_parser.get_nodes_from_documents(documents)
    leaf_nodes = get_leaf_nodes(nodes)

    # Store all nodes
    storage_context = StorageContext.from_defaults()
    storage_context.docstore.add_documents(nodes)

    # Index only leaf nodes
    index = VectorStoreIndex(
        leaf_nodes,
        storage_context=storage_context
    )

    # Retriever auto-merges to parents when siblings retrieved
    retriever = AutoMergingRetriever(
        index.as_retriever(similarity_top_k=12),
        storage_context=storage_context,
        simple_ratio_thresh=0.3  # Merge if 30%+ siblings retrieved
    )

    return retriever
```

## Pattern 4: Late Chunking (2026 Technique)

Embed full document first, then chunk - preserves global context:

```python
def late_chunking(
    document: str,
    model,
    chunk_size: int = 512
) -> list[dict]:
    """
    Late chunking: embed document, then split embeddings.
    Preserves document-level context in chunk embeddings.

    Reference: Jina AI Late Chunking (2024)
    """
    # 1. Get token-level embeddings for full document
    tokens = model.tokenize(document)
    token_embeddings = model.encode_tokens(tokens)

    # 2. Split into chunks
    chunks = []
    for i in range(0, len(tokens), chunk_size):
        chunk_tokens = tokens[i:i + chunk_size]
        chunk_embeddings = token_embeddings[i:i + chunk_size]

        # 3. Pool chunk embeddings (mean pooling)
        chunk_vector = np.mean(chunk_embeddings, axis=0)

        chunks.append({
            "text": model.decode(chunk_tokens),
            "embedding": chunk_vector
        })

    return chunks
```

## Pattern 5: Markdown/Code-Aware Chunking

```python
from langchain.text_splitter import (
    MarkdownHeaderTextSplitter,
    Language,
    RecursiveCharacterTextSplitter
)

def chunk_markdown(text: str) -> list[dict]:
    """Split markdown by headers, preserving structure."""
    headers_to_split_on = [
        ("#", "h1"),
        ("##", "h2"),
        ("###", "h3"),
    ]

    splitter = MarkdownHeaderTextSplitter(
        headers_to_split_on=headers_to_split_on
    )

    return splitter.split_text(text)


def chunk_code(code: str, language: str = "python") -> list[str]:
    """Split code respecting language syntax."""
    lang_map = {
        "python": Language.PYTHON,
        "javascript": Language.JS,
        "typescript": Language.TS,
    }

    splitter = RecursiveCharacterTextSplitter.from_language(
        language=lang_map.get(language, Language.PYTHON),
        chunk_size=1000,
        chunk_overlap=100
    )

    return splitter.split_text(code)
```

## Chunk Size Guidelines

| Content Type | Recommended Size | Overlap |
|--------------|------------------|---------|
| Q&A / FAQ | 256-512 | 25-50 |
| Technical docs | 512-1024 | 50-100 |
| Legal documents | 1024-2048 | 100-200 |
| Code | 500-1000 | 50-100 |
| Conversations | 256-512 | 50-100 |

## Evaluation: How to Know If Chunking Is Good

```python
def evaluate_chunking(chunks: list[str], test_queries: list[dict]):
    """
    Evaluate chunk quality with test queries.

    test_queries format:
    [{"query": "What is X?", "expected_chunk_contains": "X is..."}]
    """
    results = {
        "avg_chunk_size": np.mean([len(c) for c in chunks]),
        "chunk_size_std": np.std([len(c) for c in chunks]),
        "total_chunks": len(chunks),
        "retrieval_hits": 0
    }

    for tq in test_queries:
        # Check if expected content is in a single chunk
        for chunk in chunks:
            if tq["expected_chunk_contains"] in chunk:
                results["retrieval_hits"] += 1
                break

    results["hit_rate"] = results["retrieval_hits"] / len(test_queries)
    return results
```

## Best Practices

1. **Match chunk size to query length** - Chunks should be similar size to expected queries
2. **Preserve meaning boundaries** - Never split mid-sentence or mid-paragraph
3. **Include metadata** - Add source, page, section info to each chunk
4. **Test with real queries** - Evaluate on your actual use cases
5. **Consider retrieval model** - Some embedding models prefer specific chunk sizes

## Quick Decision Tree

```
What type of content?
├─ Structured (headers, sections)
│   └─ Use: Markdown/recursive splitter + hierarchy
├─ Unstructured (prose, articles)
│   └─ Use: Semantic chunking
├─ Code
│   └─ Use: Language-aware splitter
└─ Mixed
    └─ Use: Parent-child with semantic leaves
```
