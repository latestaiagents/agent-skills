---
description: Debug RAG pipeline issues with systematic retrieval and generation analysis
---

# /rag-debug

Diagnose and fix RAG pipeline problems with systematic analysis.

## What I Need

Tell me:
- What query is producing poor results?
- What answer did you expect vs what you got?
- Which vector store are you using?

## How It Works

### Step 1: Query Analysis

I'll analyze your query to identify potential issues:
- Query clarity and specificity
- Potential keyword/semantic mismatch
- Query complexity (single vs multi-hop)

### Step 2: Retrieval Diagnosis

I'll check retrieval quality:
- Are relevant documents in your index?
- What's the similarity score distribution?
- Are chunks properly sized?

### Step 3: Generation Analysis

I'll examine the generation step:
- Is context being used correctly?
- Are there hallucinations?
- Is the prompt template effective?

### Step 4: Fix Recommendations

Based on findings, I'll recommend:
- Chunking adjustments
- Retrieval strategy changes
- Prompt improvements
- Hybrid search additions

## Debug Checklist

```
[ ] Query understood correctly?
[ ] Documents indexed?
[ ] Chunks contain answer?
[ ] Top-k retrieving relevant docs?
[ ] Context passed to LLM?
[ ] LLM using context?
[ ] Answer grounded?
```

## Common Issues & Fixes

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| Wrong docs retrieved | Semantic gap | Add hybrid search |
| Right docs, wrong answer | Poor prompt | Improve system prompt |
| Partial answer | Chunk boundary | Adjust overlap |
| Hallucination | Context ignored | Add grounding check |
| No answer | Missing docs | Check indexing |

## Quick Commands

```bash
# Test retrieval only
retriever.invoke("your query")

# Check similarity scores
vectorstore.similarity_search_with_score("query", k=10)

# Inspect chunk content
for doc in results: print(doc.page_content[:200])
```
