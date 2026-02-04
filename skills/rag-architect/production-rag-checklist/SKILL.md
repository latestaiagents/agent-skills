---
name: production-rag-checklist
description: |
  Deploy RAG systems to production with proper architecture, monitoring, and operations.
  Use this skill when deploying RAG to production, scaling retrieval systems,
  implementing RAG monitoring, or building enterprise RAG infrastructure.
  Activate when: production RAG, RAG deployment, enterprise RAG, RAG architecture,
  RAG monitoring, RAG scaling, RAG infrastructure, RAG ops.
---

# Production RAG Checklist

**Everything you need to deploy RAG systems that are reliable, scalable, and maintainable.**

## Pre-Production Checklist

### Data & Indexing

- [ ] **Document pipeline defined**
  - Ingestion source(s) identified
  - Parsing strategy per document type
  - Chunking strategy tested and benchmarked
  - Metadata extraction configured

- [ ] **Index refresh strategy**
  - [ ] Real-time (streaming)
  - [ ] Scheduled (hourly/daily)
  - [ ] On-demand (manual trigger)
  - Stale document detection
  - Incremental updates (not full rebuild)

- [ ] **Data quality**
  - Duplicate detection/removal
  - Empty/corrupt document handling
  - PII detection and handling
  - Source verification

### Retrieval

- [ ] **Retrieval strategy selected**
  - [ ] Vector-only
  - [ ] Hybrid (vector + keyword)
  - [ ] GraphRAG
  - Reranking configured

- [ ] **Embeddings**
  - Model selected and tested
  - Embedding dimension appropriate
  - Batch embedding for bulk operations
  - Embedding versioning strategy

- [ ] **Vector store**
  - Production-grade store selected (Pinecone, Weaviate, Qdrant, etc.)
  - Scaling strategy (sharding, replicas)
  - Backup and recovery tested

### Generation

- [ ] **LLM configuration**
  - Model selected (cost vs quality tradeoff)
  - Temperature and parameters tuned
  - Context window sized appropriately
  - Fallback model configured

- [ ] **Prompt engineering**
  - System prompt tested
  - Few-shot examples included
  - Output format specified
  - Guardrails in place

## Architecture Patterns

### Basic Production Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Load Balancer                         │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│   API Server    │ │   API Server    │ │   API Server    │
│   (Stateless)   │ │   (Stateless)   │ │   (Stateless)   │
└─────────────────┘ └─────────────────┘ └─────────────────┘
              │               │               │
              └───────────────┼───────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│   Redis Cache   │ │  Vector Store   │ │   LLM Gateway   │
│  (Query Cache)  │ │  (Retrieval)    │ │  (Rate Limit)   │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

### LLM-Agnostic Design

```python
from abc import ABC, abstractmethod

class LLMProvider(ABC):
    @abstractmethod
    async def generate(self, prompt: str, **kwargs) -> str:
        pass

class OpenAIProvider(LLMProvider):
    async def generate(self, prompt: str, **kwargs) -> str:
        # OpenAI implementation
        pass

class AnthropicProvider(LLMProvider):
    async def generate(self, prompt: str, **kwargs) -> str:
        # Anthropic implementation
        pass

class RAGService:
    def __init__(self, llm_provider: LLMProvider, retriever):
        self.llm = llm_provider
        self.retriever = retriever

    async def query(self, question: str) -> str:
        docs = await self.retriever.ainvoke(question)
        context = self._format_context(docs)
        return await self.llm.generate(
            f"Context: {context}\n\nQuestion: {question}"
        )
```

## Caching Strategies

### Query Cache

```python
import hashlib
import redis
import json

class RAGCache:
    def __init__(self, redis_client: redis.Redis, ttl: int = 3600):
        self.redis = redis_client
        self.ttl = ttl

    def _cache_key(self, query: str) -> str:
        """Generate cache key from query."""
        return f"rag:query:{hashlib.md5(query.encode()).hexdigest()}"

    async def get_or_compute(
        self,
        query: str,
        compute_fn: callable
    ) -> dict:
        """Get from cache or compute and store."""
        key = self._cache_key(query)

        # Try cache
        cached = self.redis.get(key)
        if cached:
            return json.loads(cached)

        # Compute
        result = await compute_fn(query)

        # Cache
        self.redis.setex(key, self.ttl, json.dumps(result))
        return result
```

### Embedding Cache

```python
class EmbeddingCache:
    """Cache embeddings to avoid recomputation."""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    def get_embedding(self, text: str) -> list[float] | None:
        key = f"emb:{hashlib.md5(text.encode()).hexdigest()}"
        cached = self.redis.get(key)
        return json.loads(cached) if cached else None

    def set_embedding(self, text: str, embedding: list[float]):
        key = f"emb:{hashlib.md5(text.encode()).hexdigest()}"
        self.redis.setex(key, 86400 * 7, json.dumps(embedding))  # 7 days
```

## Error Handling & Fallbacks

```python
class ResilientRAG:
    """RAG with comprehensive error handling."""

    def __init__(self, primary_llm, fallback_llm, retriever):
        self.primary_llm = primary_llm
        self.fallback_llm = fallback_llm
        self.retriever = retriever

    async def query(self, question: str) -> dict:
        try:
            # Retrieval with timeout
            docs = await asyncio.wait_for(
                self.retriever.ainvoke(question),
                timeout=5.0
            )
        except asyncio.TimeoutError:
            logger.warning("Retrieval timeout, using empty context")
            docs = []
        except Exception as e:
            logger.error(f"Retrieval error: {e}")
            docs = []

        if not docs:
            return {
                "answer": "I couldn't find relevant information to answer your question.",
                "sources": [],
                "status": "no_results"
            }

        context = self._format_context(docs)

        # Try primary LLM
        try:
            answer = await asyncio.wait_for(
                self.primary_llm.generate(question, context),
                timeout=30.0
            )
            return {"answer": answer, "sources": docs, "status": "success"}

        except Exception as e:
            logger.warning(f"Primary LLM failed: {e}, trying fallback")

            # Try fallback LLM
            try:
                answer = await self.fallback_llm.generate(question, context)
                return {"answer": answer, "sources": docs, "status": "fallback"}
            except Exception as e:
                logger.error(f"Fallback LLM failed: {e}")
                return {
                    "answer": "I'm having trouble generating a response. Please try again.",
                    "sources": docs,
                    "status": "error"
                }
```

## Observability & Monitoring

### Logging Structure

```python
import structlog

logger = structlog.get_logger()

async def monitored_query(rag, query: str, user_id: str) -> dict:
    """Query with comprehensive logging."""
    request_id = str(uuid.uuid4())

    logger.info(
        "rag_query_started",
        request_id=request_id,
        user_id=user_id,
        query_length=len(query)
    )

    start = time.perf_counter()

    try:
        # Retrieval
        retrieval_start = time.perf_counter()
        docs = await rag.retriever.ainvoke(query)
        retrieval_ms = (time.perf_counter() - retrieval_start) * 1000

        logger.info(
            "rag_retrieval_complete",
            request_id=request_id,
            num_docs=len(docs),
            latency_ms=retrieval_ms
        )

        # Generation
        gen_start = time.perf_counter()
        answer = await rag.generate(query, docs)
        gen_ms = (time.perf_counter() - gen_start) * 1000

        total_ms = (time.perf_counter() - start) * 1000

        logger.info(
            "rag_query_complete",
            request_id=request_id,
            retrieval_ms=retrieval_ms,
            generation_ms=gen_ms,
            total_ms=total_ms,
            answer_length=len(answer)
        )

        return {"answer": answer, "request_id": request_id}

    except Exception as e:
        logger.error(
            "rag_query_failed",
            request_id=request_id,
            error=str(e),
            error_type=type(e).__name__
        )
        raise
```

### Key Metrics to Track

```python
# Prometheus metrics example
from prometheus_client import Counter, Histogram, Gauge

# Counters
rag_queries_total = Counter(
    'rag_queries_total',
    'Total RAG queries',
    ['status']  # success, no_results, error
)

# Histograms
rag_latency = Histogram(
    'rag_latency_seconds',
    'RAG query latency',
    ['stage'],  # retrieval, generation, total
    buckets=[0.1, 0.5, 1.0, 2.0, 5.0, 10.0]
)

# Gauges
rag_index_size = Gauge(
    'rag_index_document_count',
    'Number of documents in RAG index'
)
```

### Dashboard Essentials

| Metric | Alert Threshold |
|--------|-----------------|
| P95 latency | >5 seconds |
| Error rate | >5% |
| Cache hit rate | <50% |
| Empty results rate | >20% |
| LLM fallback rate | >10% |

## Security Checklist

- [ ] **Input validation**
  - Query length limits
  - Prompt injection detection
  - Rate limiting per user

- [ ] **Data protection**
  - PII filtering in responses
  - Access control on documents
  - Audit logging

- [ ] **API security**
  - Authentication required
  - HTTPS only
  - API key rotation

## Cost Optimization

```python
class CostAwareRAG:
    """RAG with cost controls."""

    def __init__(self, rag, monthly_budget: float):
        self.rag = rag
        self.budget = monthly_budget
        self.month_spend = 0

    async def query(self, question: str) -> dict:
        # Check budget
        if self.month_spend >= self.budget:
            return {"error": "Monthly budget exceeded"}

        # Estimate cost before query
        estimated_cost = self._estimate_cost(question)

        if self.month_spend + estimated_cost > self.budget:
            # Use cheaper model
            return await self.rag.query_cheap(question)

        result = await self.rag.query(question)
        self.month_spend += self._actual_cost(result)
        return result

    def _estimate_cost(self, question: str) -> float:
        # Estimate based on typical token usage
        input_tokens = len(question.split()) * 1.3
        output_tokens = 500  # Assume average
        return (input_tokens * 0.00001) + (output_tokens * 0.00003)
```

## Deployment Checklist

### Pre-Deploy
- [ ] Load testing completed
- [ ] Evaluation metrics meet thresholds
- [ ] Rollback plan documented
- [ ] On-call rotation set up

### Deploy
- [ ] Canary deployment (10% traffic)
- [ ] Monitor error rates
- [ ] Monitor latency
- [ ] Gradual rollout

### Post-Deploy
- [ ] Verify metrics stable
- [ ] Check cache hit rates
- [ ] Monitor user feedback
- [ ] Schedule first review (1 week)
