---
name: agentic-rag
description: |
  Build agentic RAG systems with autonomous planning, reflection, and tool use.
  Use this skill when implementing agent-driven retrieval, query decomposition,
  iterative refinement, or multi-source RAG orchestration.
  Activate when: agentic RAG, A-RAG, autonomous retrieval, query planning,
  iterative RAG, multi-step retrieval, agent retrieval, RAG agent.
---

# Agentic RAG Patterns

**Let agents autonomously plan, retrieve, reflect, and refine for complex queries.**

## What Makes RAG "Agentic"

Traditional RAG: Query → Retrieve → Generate (one-shot)

Agentic RAG adds:
- **Planning**: Decompose complex queries into sub-queries
- **Tool Use**: Choose between multiple retrieval sources
- **Reflection**: Evaluate if retrieved context is sufficient
- **Iteration**: Refine queries and re-retrieve if needed

## Core Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      User Query                              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Query Planner                             │
│         "Break this into retrievable sub-queries"            │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        ┌──────────┐   ┌──────────┐   ┌──────────┐
        │ SubQ 1   │   │ SubQ 2   │   │ SubQ 3   │
        └──────────┘   └──────────┘   └──────────┘
              │               │               │
              ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────┐
│                    Tool Router                               │
│         "Which source for each sub-query?"                   │
└─────────────────────────────────────────────────────────────┘
              │               │               │
              ▼               ▼               ▼
        ┌──────────┐   ┌──────────┐   ┌──────────┐
        │ VectorDB │   │ Web API  │   │ SQL DB   │
        └──────────┘   └──────────┘   └──────────┘
              │               │               │
              └───────────────┼───────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Reflection                                │
│         "Is this context sufficient to answer?"              │
│         → No: Refine query, re-retrieve                     │
│         → Yes: Generate answer                               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Response Generation                       │
└─────────────────────────────────────────────────────────────┘
```

## Pattern 1: Query Decomposition

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate

DECOMPOSITION_PROMPT = """You are a query planner. Break down complex queries into simpler sub-queries.

Original query: {query}

Rules:
1. Each sub-query should be answerable independently
2. Sub-queries should cover all aspects of the original query
3. Return 2-5 sub-queries maximum
4. Return as JSON array

Example:
Query: "Compare Tesla and Ford's EV strategies and market performance"
Sub-queries: [
  "What is Tesla's electric vehicle strategy?",
  "What is Ford's electric vehicle strategy?",
  "What is Tesla's market performance and stock price?",
  "What is Ford's market performance and stock price?"
]

Sub-queries:"""

async def decompose_query(query: str, llm: ChatOpenAI) -> list[str]:
    """Break complex query into sub-queries."""
    prompt = ChatPromptTemplate.from_template(DECOMPOSITION_PROMPT)
    chain = prompt | llm
    result = await chain.ainvoke({"query": query})
    return json.loads(result.content)
```

## Pattern 2: Tool-Based Routing

```python
from langchain.agents import AgentExecutor, create_openai_tools_agent
from langchain.tools import Tool

# Define retrieval tools
tools = [
    Tool(
        name="company_docs",
        description="Search internal company documentation. Use for policies, procedures, internal info.",
        func=lambda q: vector_store_company.similarity_search(q)
    ),
    Tool(
        name="product_catalog",
        description="Search product database. Use for product specs, pricing, availability.",
        func=lambda q: sql_product_search(q)
    ),
    Tool(
        name="web_search",
        description="Search the web. Use for current events, external information.",
        func=lambda q: tavily_search(q)
    ),
    Tool(
        name="knowledge_graph",
        description="Query knowledge graph. Use for relationships between entities.",
        func=lambda q: graph_query(q)
    ),
]

# Create agent that routes to appropriate tools
agent = create_openai_tools_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

# Agent decides which tool(s) to use
result = executor.invoke({
    "input": "What's our return policy and how does it compare to Amazon's?"
})
# Agent uses: company_docs for policy, web_search for Amazon comparison
```

## Pattern 3: LangGraph Agentic RAG

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
import operator

class AgentState(TypedDict):
    query: str
    sub_queries: list[str]
    retrieved_docs: Annotated[list, operator.add]
    reflection: str
    answer: str
    iteration: int

def plan_node(state: AgentState) -> AgentState:
    """Decompose query into sub-queries."""
    sub_queries = decompose_query(state["query"])
    return {"sub_queries": sub_queries}

def retrieve_node(state: AgentState) -> AgentState:
    """Retrieve docs for each sub-query."""
    docs = []
    for sq in state["sub_queries"]:
        results = retriever.invoke(sq)
        docs.extend(results)
    return {"retrieved_docs": docs}

def reflect_node(state: AgentState) -> AgentState:
    """Evaluate if context is sufficient."""
    prompt = f"""
    Query: {state['query']}
    Retrieved context: {state['retrieved_docs'][:5]}

    Is this context sufficient to answer the query?
    If not, what additional information is needed?

    Return JSON: {{"sufficient": true/false, "missing": "..."}}
    """
    reflection = llm.invoke(prompt)
    return {"reflection": reflection.content}

def should_continue(state: AgentState) -> str:
    """Decide whether to refine or generate."""
    reflection = json.loads(state["reflection"])
    if reflection["sufficient"] or state["iteration"] >= 3:
        return "generate"
    return "refine"

def refine_node(state: AgentState) -> AgentState:
    """Refine query based on reflection."""
    reflection = json.loads(state["reflection"])
    refined = f"{state['query']} Specifically: {reflection['missing']}"
    return {
        "sub_queries": [refined],
        "iteration": state["iteration"] + 1
    }

def generate_node(state: AgentState) -> AgentState:
    """Generate final answer."""
    context = "\n".join([d.page_content for d in state["retrieved_docs"]])
    prompt = f"Context: {context}\n\nQuery: {state['query']}\n\nAnswer:"
    answer = llm.invoke(prompt)
    return {"answer": answer.content}

# Build graph
workflow = StateGraph(AgentState)
workflow.add_node("plan", plan_node)
workflow.add_node("retrieve", retrieve_node)
workflow.add_node("reflect", reflect_node)
workflow.add_node("refine", refine_node)
workflow.add_node("generate", generate_node)

workflow.set_entry_point("plan")
workflow.add_edge("plan", "retrieve")
workflow.add_edge("retrieve", "reflect")
workflow.add_conditional_edges("reflect", should_continue, {
    "generate": "generate",
    "refine": "refine"
})
workflow.add_edge("refine", "retrieve")
workflow.add_edge("generate", END)

app = workflow.compile()
```

## Pattern 4: Multi-Source Orchestration

```python
class MultiSourceRAG:
    """Orchestrate retrieval across multiple sources."""

    def __init__(self, sources: dict):
        self.sources = sources  # {"name": retriever}
        self.router_llm = ChatOpenAI(model="gpt-4")

    async def retrieve(self, query: str) -> dict:
        # 1. Determine which sources to query
        routing_prompt = f"""
        Query: {query}

        Available sources:
        {list(self.sources.keys())}

        Which sources should be queried? Return as JSON array.
        """
        sources_to_query = json.loads(
            (await self.router_llm.ainvoke(routing_prompt)).content
        )

        # 2. Query selected sources in parallel
        tasks = []
        for source_name in sources_to_query:
            if source_name in self.sources:
                tasks.append(self._query_source(source_name, query))

        results = await asyncio.gather(*tasks)

        # 3. Combine and deduplicate
        combined = self._fuse_results(results)
        return combined

    async def _query_source(self, name: str, query: str):
        retriever = self.sources[name]
        docs = await retriever.ainvoke(query)
        return {"source": name, "docs": docs}

    def _fuse_results(self, results: list) -> list:
        # Deduplicate and rank
        seen = set()
        fused = []
        for r in results:
            for doc in r["docs"]:
                doc_hash = hash(doc.page_content[:100])
                if doc_hash not in seen:
                    seen.add(doc_hash)
                    doc.metadata["source"] = r["source"]
                    fused.append(doc)
        return fused
```

## Pattern 5: Self-Reflective RAG

```python
def self_reflective_rag(query: str, retriever, llm, max_iterations: int = 3):
    """RAG with self-reflection loop."""

    for i in range(max_iterations):
        # Retrieve
        docs = retriever.invoke(query)
        context = "\n".join([d.page_content for d in docs])

        # Generate with reflection request
        response = llm.invoke(f"""
        Context: {context}

        Query: {query}

        Provide your answer, then reflect:
        1. Does the context fully support your answer?
        2. What information might be missing?
        3. Confidence level (high/medium/low)?

        Format:
        ANSWER: ...
        REFLECTION: ...
        CONFIDENCE: ...
        MISSING: ...
        """)

        # Parse reflection
        if "CONFIDENCE: high" in response.content:
            return extract_answer(response.content)

        if "MISSING:" in response.content:
            # Refine query with missing info
            missing = extract_missing(response.content)
            query = f"{query} Also include: {missing}"

    return extract_answer(response.content)
```

## Best Practices

1. **Limit iterations** - Cap at 3-5 to prevent infinite loops
2. **Parallelize sub-queries** - Use async for multiple retrievals
3. **Cache intermediate results** - Don't re-retrieve same content
4. **Track token usage** - Agentic loops can be expensive
5. **Provide fallbacks** - If reflection fails, fall back to standard RAG

## When to Use Agentic RAG

| Query Type | Standard RAG | Agentic RAG |
|------------|--------------|-------------|
| Simple factual | ✅ | Overkill |
| Multi-part questions | ⚠️ | ✅ |
| Comparison queries | ❌ | ✅ |
| Research tasks | ❌ | ✅ |
| Multi-source needs | ❌ | ✅ |

## Performance Considerations

- **Latency**: 2-5x slower than standard RAG
- **Cost**: 3-10x more tokens consumed
- **Quality**: 15-30% better on complex queries

Use agentic patterns selectively for complex queries, not for everything.
