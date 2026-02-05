---
description: Step-by-step guidance to build a RAG pipeline from scratch
---

# /build-rag

Build a complete RAG pipeline with best practices.

## What I Need

Tell me:
- What documents/data will you be querying?
- What's your use case (Q&A, search, chatbot)?
- Any technology preferences (LangChain, LlamaIndex, custom)?
- Scale expectations (documents, queries/day)?

## Workflow

### Step 1: Document Processing

I'll help you set up:
- Document loading (PDF, web, database)
- Chunking strategy based on content type
- Metadata extraction

### Step 2: Embedding & Indexing

We'll configure:
- Embedding model selection
- Vector store setup
- Index optimization

### Step 3: Retrieval Pipeline

I'll implement:
- Basic vector search
- Hybrid search (if needed)
- Reranking layer

### Step 4: Generation

We'll build:
- Prompt template
- Context formatting
- Citation handling

### Step 5: Evaluation

I'll set up:
- Test dataset creation
- Retrieval metrics
- Quality monitoring

## Architecture Options

**Simple RAG:**
```
Query → Embed → Search → Context → Generate → Answer
```

**Production RAG:**
```
Query → Cache Check → Hybrid Search → Rerank → Generate → Validate → Cache → Answer
```

## Quick Start Templates

**LangChain:**
```python
from langchain_community.vectorstores import Chroma
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain.chains import RetrievalQA

vectorstore = Chroma.from_documents(docs, OpenAIEmbeddings())
qa = RetrievalQA.from_chain_type(
    llm=ChatOpenAI(),
    retriever=vectorstore.as_retriever()
)
```

**LlamaIndex:**
```python
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader

documents = SimpleDirectoryReader("data").load_data()
index = VectorStoreIndex.from_documents(documents)
query_engine = index.as_query_engine()
```
