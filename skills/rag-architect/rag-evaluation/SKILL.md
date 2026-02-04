---
name: rag-evaluation
description: |
  Test and benchmark RAG systems with proper metrics and evaluation frameworks.
  Use this skill when evaluating RAG quality, measuring retrieval performance,
  using RAGAS metrics, or building RAG test suites.
  Activate when: RAG evaluation, RAG testing, RAGAS, retrieval metrics,
  faithfulness, relevance, context precision, RAG benchmarking.
---

# RAG Evaluation & Testing

**Measure what matters: retrieval quality, answer faithfulness, and end-to-end performance.**

## Key Metrics Overview

| Metric | What It Measures | Good Score |
|--------|------------------|------------|
| **Context Precision** | Are retrieved docs relevant? | >0.8 |
| **Context Recall** | Did we get all relevant docs? | >0.7 |
| **Faithfulness** | Is answer grounded in context? | >0.9 |
| **Answer Relevancy** | Does answer address the question? | >0.8 |
| **Answer Correctness** | Is the answer factually correct? | >0.8 |

## RAGAS Framework

RAGAS (Retrieval Augmented Generation Assessment) is the standard for RAG evaluation.

### Installation & Setup

```python
pip install ragas langchain_openai

from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_precision,
    context_recall,
    answer_correctness
)
from datasets import Dataset
```

### Creating Evaluation Dataset

```python
def create_eval_dataset(test_cases: list[dict]) -> Dataset:
    """
    Create evaluation dataset from test cases.

    Each test case needs:
    - question: The query
    - answer: Generated answer from your RAG
    - contexts: Retrieved documents (list of strings)
    - ground_truth: Expected correct answer (for some metrics)
    """
    return Dataset.from_dict({
        "question": [tc["question"] for tc in test_cases],
        "answer": [tc["answer"] for tc in test_cases],
        "contexts": [tc["contexts"] for tc in test_cases],
        "ground_truth": [tc.get("ground_truth", "") for tc in test_cases]
    })

# Example test cases
test_cases = [
    {
        "question": "What is the return policy?",
        "answer": "Items can be returned within 30 days with receipt.",
        "contexts": [
            "Our return policy allows returns within 30 days of purchase. A receipt is required.",
            "Refunds are processed within 5-7 business days."
        ],
        "ground_truth": "30-day return policy with receipt required"
    },
    # Add more test cases...
]

eval_dataset = create_eval_dataset(test_cases)
```

### Running Evaluation

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_precision,
    context_recall
)

# Run evaluation
results = evaluate(
    eval_dataset,
    metrics=[
        faithfulness,
        answer_relevancy,
        context_precision,
        context_recall
    ],
    llm=ChatOpenAI(model="gpt-4"),
    embeddings=OpenAIEmbeddings()
)

# View results
print(results)
# {'faithfulness': 0.92, 'answer_relevancy': 0.88, ...}

# Detailed per-question results
df = results.to_pandas()
print(df)
```

## Understanding Each Metric

### Faithfulness (Is answer grounded?)

Measures if the answer can be inferred from the retrieved context.

```python
from ragas.metrics import faithfulness

# Faithfulness checks:
# 1. Extracts claims from the answer
# 2. Verifies each claim against context
# 3. Score = supported claims / total claims

# Example:
# Answer: "The product costs $99 and ships free"
# Context: "Product price: $99. Standard shipping: $5"
# Faithfulness: 0.5 (only price claim is supported)
```

### Context Precision (Are retrieved docs relevant?)

Measures if retrieved documents are actually relevant to the question.

```python
from ragas.metrics import context_precision

# High precision: Retrieved docs are all relevant
# Low precision: Retrieved docs contain irrelevant noise

# Scored by checking if each context chunk is needed
# to answer the question
```

### Context Recall (Did we get everything?)

Measures if all information needed to answer is in the retrieved context.

```python
from ragas.metrics import context_recall

# Requires ground_truth answer
# Checks if context contains info needed for ground truth

# High recall: Context has everything needed
# Low recall: Missing relevant documents
```

### Answer Relevancy (Does it answer the question?)

Measures how well the answer addresses the original question.

```python
from ragas.metrics import answer_relevancy

# Generates questions from the answer
# Compares generated questions to original
# Higher similarity = more relevant answer
```

## Custom Evaluation Metrics

### Retrieval Hit Rate

```python
def calculate_hit_rate(
    test_cases: list[dict],
    retriever,
    k: int = 5
) -> float:
    """
    Calculate retrieval hit rate.
    Hit = relevant doc in top-k results.
    """
    hits = 0
    for tc in test_cases:
        results = retriever.invoke(tc["question"])[:k]
        retrieved_texts = [r.page_content for r in results]

        # Check if any relevant doc was retrieved
        for relevant_doc in tc["relevant_docs"]:
            if any(relevant_doc in text for text in retrieved_texts):
                hits += 1
                break

    return hits / len(test_cases)
```

### Mean Reciprocal Rank (MRR)

```python
def calculate_mrr(
    test_cases: list[dict],
    retriever
) -> float:
    """
    Mean Reciprocal Rank.
    Higher = relevant docs appear earlier in results.
    """
    reciprocal_ranks = []

    for tc in test_cases:
        results = retriever.invoke(tc["question"])
        retrieved_texts = [r.page_content for r in results]

        # Find rank of first relevant doc
        for rank, text in enumerate(retrieved_texts, 1):
            if any(rel in text for rel in tc["relevant_docs"]):
                reciprocal_ranks.append(1 / rank)
                break
        else:
            reciprocal_ranks.append(0)

    return sum(reciprocal_ranks) / len(reciprocal_ranks)
```

### Latency Tracking

```python
import time
from dataclasses import dataclass

@dataclass
class RAGMetrics:
    retrieval_latency_ms: float
    generation_latency_ms: float
    total_latency_ms: float
    num_docs_retrieved: int
    num_tokens_generated: int

def measure_rag_performance(rag_chain, query: str) -> RAGMetrics:
    """Measure RAG pipeline performance."""

    # Measure retrieval
    start = time.perf_counter()
    docs = rag_chain.retriever.invoke(query)
    retrieval_time = (time.perf_counter() - start) * 1000

    # Measure generation
    start = time.perf_counter()
    response = rag_chain.generate(query, docs)
    generation_time = (time.perf_counter() - start) * 1000

    return RAGMetrics(
        retrieval_latency_ms=retrieval_time,
        generation_latency_ms=generation_time,
        total_latency_ms=retrieval_time + generation_time,
        num_docs_retrieved=len(docs),
        num_tokens_generated=len(response.split())
    )
```

## Building a Test Suite

```python
class RAGTestSuite:
    """Comprehensive RAG test suite."""

    def __init__(self, rag_chain, test_cases: list[dict]):
        self.rag = rag_chain
        self.test_cases = test_cases

    def run_all(self) -> dict:
        """Run all evaluation metrics."""
        results = {
            "ragas_metrics": self._run_ragas(),
            "retrieval_metrics": self._run_retrieval_metrics(),
            "latency_metrics": self._run_latency_metrics(),
            "failure_analysis": self._analyze_failures()
        }
        return results

    def _run_ragas(self) -> dict:
        # Generate answers for test cases
        eval_data = []
        for tc in self.test_cases:
            answer = self.rag.query(tc["question"])
            contexts = self.rag.retriever.invoke(tc["question"])
            eval_data.append({
                "question": tc["question"],
                "answer": answer,
                "contexts": [c.page_content for c in contexts],
                "ground_truth": tc.get("expected_answer", "")
            })

        dataset = Dataset.from_dict({
            "question": [d["question"] for d in eval_data],
            "answer": [d["answer"] for d in eval_data],
            "contexts": [d["contexts"] for d in eval_data],
            "ground_truth": [d["ground_truth"] for d in eval_data]
        })

        return evaluate(dataset, metrics=[
            faithfulness, answer_relevancy,
            context_precision, context_recall
        ])

    def _run_retrieval_metrics(self) -> dict:
        return {
            "hit_rate@5": calculate_hit_rate(self.test_cases, self.rag.retriever, k=5),
            "hit_rate@10": calculate_hit_rate(self.test_cases, self.rag.retriever, k=10),
            "mrr": calculate_mrr(self.test_cases, self.rag.retriever)
        }

    def _run_latency_metrics(self) -> dict:
        latencies = []
        for tc in self.test_cases[:10]:  # Sample for latency
            metrics = measure_rag_performance(self.rag, tc["question"])
            latencies.append(metrics)

        return {
            "avg_total_ms": sum(l.total_latency_ms for l in latencies) / len(latencies),
            "avg_retrieval_ms": sum(l.retrieval_latency_ms for l in latencies) / len(latencies),
            "p95_total_ms": sorted([l.total_latency_ms for l in latencies])[int(len(latencies) * 0.95)]
        }

    def _analyze_failures(self) -> list[dict]:
        """Identify and categorize failures."""
        failures = []
        for tc in self.test_cases:
            answer = self.rag.query(tc["question"])
            if tc.get("expected_answer") and tc["expected_answer"].lower() not in answer.lower():
                failures.append({
                    "question": tc["question"],
                    "expected": tc["expected_answer"],
                    "actual": answer,
                    "failure_type": self._classify_failure(tc, answer)
                })
        return failures

    def _classify_failure(self, tc: dict, answer: str) -> str:
        """Classify failure type."""
        contexts = self.rag.retriever.invoke(tc["question"])
        context_text = " ".join([c.page_content for c in contexts])

        if tc.get("expected_answer") and tc["expected_answer"] in context_text:
            return "generation_failure"  # Right docs, wrong answer
        else:
            return "retrieval_failure"  # Wrong docs retrieved
```

## Continuous Evaluation

```python
# In production, log and monitor
import logging

logger = logging.getLogger("rag_eval")

def monitored_rag_query(rag, query: str) -> dict:
    """RAG query with monitoring."""
    start = time.perf_counter()

    # Get results
    docs = rag.retriever.invoke(query)
    answer = rag.generate(query, docs)

    # Log metrics
    logger.info({
        "query": query,
        "num_docs": len(docs),
        "latency_ms": (time.perf_counter() - start) * 1000,
        "answer_length": len(answer)
    })

    return {"answer": answer, "sources": docs}
```

## Best Practices

1. **Build golden dataset** - 50-100 human-validated Q&A pairs
2. **Test edge cases** - Empty results, ambiguous queries, out-of-domain
3. **Automate in CI/CD** - Run eval on every change
4. **Track over time** - Monitor metric trends
5. **Segment by query type** - Different metrics for different use cases
