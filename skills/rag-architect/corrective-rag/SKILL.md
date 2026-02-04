---
name: corrective-rag
description: |
  Implement Corrective RAG (CRAG) and Self-RAG for reliable, self-healing retrieval systems.
  Use this skill when building reliable RAG, preventing hallucinations, implementing
  retrieval evaluation, or adding self-correction to RAG pipelines.
  Activate when: corrective RAG, CRAG, self-RAG, hallucination prevention,
  retrieval evaluation, RAG reliability, self-healing RAG, document grading.
---

# Corrective RAG (CRAG) & Self-RAG

**Build RAG systems that detect and correct retrieval failures automatically.**

## The Problem with Standard RAG

Standard RAG blindly trusts retrieved documents:
- Irrelevant docs → hallucinated answers
- Partially relevant docs → incomplete answers
- Outdated docs → incorrect answers

**Corrective RAG adds evaluation and correction loops.**

## CRAG Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      User Query                              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Initial Retrieval                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│               Retrieval Evaluator (Grade Documents)          │
│                                                              │
│   For each document, classify as:                           │
│   • CORRECT - Relevant and supports answer                  │
│   • INCORRECT - Not relevant to query                       │
│   • AMBIGUOUS - Partially relevant, needs more context      │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        ┌──────────┐   ┌──────────┐   ┌──────────┐
        │ CORRECT  │   │ AMBIGUOUS│   │ INCORRECT│
        │          │   │          │   │          │
        │ Use docs │   │ Refine + │   │ Web      │
        │ as-is    │   │ Retrieve │   │ Search   │
        └──────────┘   └──────────┘   └──────────┘
              │               │               │
              └───────────────┼───────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  Knowledge Refinement                        │
│         (Strip irrelevant parts, keep key info)             │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Generate Answer                           │
└─────────────────────────────────────────────────────────────┘
```

## Pattern 1: Document Grading

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from enum import Enum

class RetrievalGrade(Enum):
    CORRECT = "correct"
    INCORRECT = "incorrect"
    AMBIGUOUS = "ambiguous"

GRADING_PROMPT = """You are a retrieval evaluator. Grade if a document is relevant to a query.

Query: {query}

Document: {document}

Grade the document:
- CORRECT: Document directly answers or supports answering the query
- INCORRECT: Document is not relevant to the query at all
- AMBIGUOUS: Document is partially relevant but may need more context

Return only one word: CORRECT, INCORRECT, or AMBIGUOUS"""

async def grade_document(
    query: str,
    document: str,
    llm: ChatOpenAI
) -> RetrievalGrade:
    """Grade a single document's relevance."""
    prompt = ChatPromptTemplate.from_template(GRADING_PROMPT)
    chain = prompt | llm
    result = await chain.ainvoke({"query": query, "document": document})
    grade = result.content.strip().upper()
    return RetrievalGrade(grade.lower())


async def grade_documents(
    query: str,
    documents: list[str],
    llm: ChatOpenAI
) -> dict[str, list]:
    """Grade multiple documents in parallel."""
    tasks = [grade_document(query, doc, llm) for doc in documents]
    grades = await asyncio.gather(*tasks)

    categorized = {
        "correct": [],
        "incorrect": [],
        "ambiguous": []
    }

    for doc, grade in zip(documents, grades):
        categorized[grade.value].append(doc)

    return categorized
```

## Pattern 2: Full CRAG Pipeline

```python
from langchain_community.tools import TavilySearchResults

class CorrectiveRAG:
    def __init__(self, retriever, llm, web_search_tool=None):
        self.retriever = retriever
        self.llm = llm
        self.web_search = web_search_tool or TavilySearchResults()

    async def query(self, question: str) -> str:
        # Step 1: Initial retrieval
        docs = await self.retriever.ainvoke(question)
        documents = [d.page_content for d in docs]

        # Step 2: Grade documents
        graded = await grade_documents(question, documents, self.llm)

        # Step 3: Decide action based on grades
        if len(graded["correct"]) >= 2:
            # Sufficient correct docs - proceed
            context_docs = graded["correct"]
            source = "knowledge_base"

        elif len(graded["ambiguous"]) > len(graded["correct"]):
            # Mostly ambiguous - refine and re-retrieve
            refined_query = await self._refine_query(question, graded["ambiguous"])
            new_docs = await self.retriever.ainvoke(refined_query)
            context_docs = [d.page_content for d in new_docs]
            source = "refined_retrieval"

        else:
            # Mostly incorrect - fall back to web search
            web_results = await self.web_search.ainvoke(question)
            context_docs = [r["content"] for r in web_results]
            source = "web_search"

        # Step 4: Refine knowledge (strip irrelevant parts)
        refined_context = await self._refine_knowledge(question, context_docs)

        # Step 5: Generate answer
        answer = await self._generate(question, refined_context)

        return {
            "answer": answer,
            "source": source,
            "docs_used": len(context_docs)
        }

    async def _refine_query(self, query: str, ambiguous_docs: list) -> str:
        """Refine query based on ambiguous results."""
        prompt = f"""
        Original query: {query}

        These documents were partially relevant:
        {ambiguous_docs[:2]}

        Rewrite the query to be more specific and get better results.
        Return only the refined query.
        """
        result = await self.llm.ainvoke(prompt)
        return result.content

    async def _refine_knowledge(self, query: str, docs: list) -> str:
        """Extract only relevant parts from documents."""
        prompt = f"""
        Query: {query}

        Documents:
        {chr(10).join(docs)}

        Extract only the sentences/paragraphs that directly help answer the query.
        Remove any irrelevant information.
        """
        result = await self.llm.ainvoke(prompt)
        return result.content

    async def _generate(self, query: str, context: str) -> str:
        """Generate final answer."""
        prompt = f"""
        Context: {context}

        Question: {query}

        Answer based only on the provided context. If the context doesn't
        contain enough information, say so.
        """
        result = await self.llm.ainvoke(prompt)
        return result.content
```

## Pattern 3: Self-RAG with Reflection Tokens

```python
class SelfRAG:
    """
    Self-RAG: Learning to Retrieve, Generate, and Critique through Self-Reflection

    Uses special tokens to decide:
    - [Retrieve]: Should we retrieve? (yes/no)
    - [IsRel]: Is retrieved doc relevant? (yes/no)
    - [IsSup]: Does doc support the response? (fully/partially/no)
    - [IsUse]: Is the response useful? (1-5)
    """

    def __init__(self, retriever, llm):
        self.retriever = retriever
        self.llm = llm

    async def query(self, question: str) -> str:
        # Step 1: Decide if retrieval is needed
        needs_retrieval = await self._should_retrieve(question)

        if not needs_retrieval:
            # Generate without retrieval
            return await self._generate_direct(question)

        # Step 2: Retrieve and evaluate
        docs = await self.retriever.ainvoke(question)
        relevant_docs = []

        for doc in docs:
            # Check relevance
            is_relevant = await self._is_relevant(question, doc.page_content)
            if is_relevant:
                relevant_docs.append(doc)

        if not relevant_docs:
            # No relevant docs - generate without or try web
            return await self._generate_direct(question)

        # Step 3: Generate with relevant docs
        context = "\n".join([d.page_content for d in relevant_docs])
        response = await self._generate_with_context(question, context)

        # Step 4: Self-critique
        is_supported = await self._is_supported(response, context)
        usefulness = await self._rate_usefulness(question, response)

        if not is_supported or usefulness < 3:
            # Response not good enough - refine
            response = await self._refine_response(question, context, response)

        return response

    async def _should_retrieve(self, question: str) -> bool:
        """Decide if retrieval is needed for this question."""
        prompt = f"""
        Question: {question}

        Should external knowledge be retrieved to answer this question?
        Consider:
        - Is this factual or opinion-based?
        - Does it require specific/current information?
        - Can it be answered from general knowledge?

        Return: YES or NO
        """
        result = await self.llm.ainvoke(prompt)
        return "YES" in result.content.upper()

    async def _is_relevant(self, question: str, document: str) -> bool:
        """Check if document is relevant to question."""
        prompt = f"""
        Question: {question}
        Document: {document[:500]}

        Is this document relevant to answering the question?
        Return: YES or NO
        """
        result = await self.llm.ainvoke(prompt)
        return "YES" in result.content.upper()

    async def _is_supported(self, response: str, context: str) -> bool:
        """Check if response is supported by context."""
        prompt = f"""
        Context: {context}
        Response: {response}

        Is the response fully supported by the context?
        Return: FULLY, PARTIALLY, or NO
        """
        result = await self.llm.ainvoke(prompt)
        return "FULLY" in result.content.upper()

    async def _rate_usefulness(self, question: str, response: str) -> int:
        """Rate response usefulness 1-5."""
        prompt = f"""
        Question: {question}
        Response: {response}

        Rate how useful this response is (1-5):
        5 = Completely answers the question
        4 = Mostly answers with minor gaps
        3 = Partially answers
        2 = Barely relevant
        1 = Not useful

        Return only the number.
        """
        result = await self.llm.ainvoke(prompt)
        try:
            return int(result.content.strip())
        except:
            return 3
```

## Pattern 4: LangGraph CRAG Implementation

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Literal

class CRAGState(TypedDict):
    question: str
    documents: list
    grades: dict
    web_results: list
    generation: str

def retrieve(state: CRAGState) -> CRAGState:
    """Retrieve documents."""
    docs = retriever.invoke(state["question"])
    return {"documents": docs}

def grade_documents_node(state: CRAGState) -> CRAGState:
    """Grade retrieved documents."""
    grades = {"correct": [], "incorrect": [], "ambiguous": []}
    for doc in state["documents"]:
        grade = grade_single_doc(state["question"], doc.page_content)
        grades[grade].append(doc)
    return {"grades": grades}

def decide_action(state: CRAGState) -> Literal["generate", "web_search"]:
    """Route based on grades."""
    if len(state["grades"]["correct"]) >= 2:
        return "generate"
    return "web_search"

def web_search(state: CRAGState) -> CRAGState:
    """Fallback to web search."""
    results = tavily.invoke(state["question"])
    return {"web_results": results}

def generate(state: CRAGState) -> CRAGState:
    """Generate answer from context."""
    if state.get("web_results"):
        context = "\n".join([r["content"] for r in state["web_results"]])
    else:
        context = "\n".join([d.page_content for d in state["grades"]["correct"]])

    response = llm.invoke(f"Context: {context}\n\nQuestion: {state['question']}")
    return {"generation": response.content}

# Build graph
workflow = StateGraph(CRAGState)
workflow.add_node("retrieve", retrieve)
workflow.add_node("grade", grade_documents_node)
workflow.add_node("web_search", web_search)
workflow.add_node("generate", generate)

workflow.set_entry_point("retrieve")
workflow.add_edge("retrieve", "grade")
workflow.add_conditional_edges("grade", decide_action)
workflow.add_edge("web_search", "generate")
workflow.add_edge("generate", END)

app = workflow.compile()
```

## Evaluation Thresholds

| Metric | Good | Needs Work |
|--------|------|------------|
| Correct docs ratio | >60% | <40% |
| Web fallback rate | <20% | >50% |
| Answer support rate | >80% | <60% |
| Usefulness score | >4.0 | <3.0 |

## Best Practices

1. **Grade in parallel** - Don't block on sequential grading
2. **Cache grades** - Same doc + query = same grade
3. **Tune thresholds** - Start strict, loosen if too many fallbacks
4. **Log everything** - Track which documents fail and why
5. **A/B test** - Compare CRAG vs standard RAG quality

## When to Use

- **High-stakes applications** - Medical, legal, financial
- **Low-quality knowledge bases** - Outdated or noisy docs
- **User-facing products** - Where hallucinations damage trust
- **Compliance requirements** - Need audit trail of sources
