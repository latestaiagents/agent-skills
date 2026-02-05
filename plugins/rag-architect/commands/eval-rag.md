---
description: Evaluate RAG pipeline quality with comprehensive metrics
---

# /eval-rag

Run comprehensive evaluation on your RAG pipeline.

## What I Need

Tell me:
- Do you have a test dataset? (queries + expected answers)
- What metrics matter most? (accuracy, latency, cost)
- Current retrieval/generation setup?

## Evaluation Steps

### Step 1: Test Dataset

I'll help you create or validate your test dataset:
- Query variety (simple, complex, edge cases)
- Ground truth answers
- Relevant document mappings

### Step 2: Retrieval Metrics

Measure retrieval quality:
- **Recall@k** - Are relevant docs in top-k?
- **MRR** - How high do relevant docs rank?
- **NDCG** - Quality of ranking order

### Step 3: Generation Metrics

Assess answer quality:
- **Faithfulness** - Is answer grounded in context?
- **Relevance** - Does answer address the question?
- **Correctness** - Is answer factually correct?

### Step 4: System Metrics

Track operational performance:
- **Latency** - p50, p95, p99 response times
- **Cost** - Per-query LLM/embedding costs
- **Throughput** - Queries per second

## Quick Evaluation

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy

# Prepare data
eval_data = {
    "question": [...],
    "answer": [...],
    "contexts": [...],
    "ground_truth": [...]
}

# Run evaluation
results = evaluate(Dataset.from_dict(eval_data),
                   metrics=[faithfulness, answer_relevancy])
print(results)
```

## Benchmark Targets

| Metric | Good | Great |
|--------|------|-------|
| Recall@5 | >0.7 | >0.85 |
| MRR | >0.6 | >0.75 |
| Faithfulness | >3.5/5 | >4.0/5 |
| Relevance | >3.5/5 | >4.0/5 |
| Latency p95 | <2s | <500ms |

## Output

I'll provide:
- Metric scores with comparisons
- Failure case analysis
- Specific improvement recommendations
