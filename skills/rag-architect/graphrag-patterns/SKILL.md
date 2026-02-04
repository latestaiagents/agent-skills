---
name: graphrag-patterns
description: |
  Implement GraphRAG combining knowledge graphs with RAG for multi-hop reasoning.
  Use this skill when building knowledge graph RAG, implementing multi-hop queries,
  using Neo4j with RAG, or connecting entities across documents.
  Activate when: GraphRAG, knowledge graph, multi-hop reasoning, Neo4j RAG,
  entity extraction, relationship queries, graph database, connected data.
---

# GraphRAG Patterns

**Combine knowledge graphs with RAG for complex reasoning over connected data.**

## When to Use GraphRAG vs Vector RAG

| Use Case | Vector RAG | GraphRAG |
|----------|------------|----------|
| Simple Q&A | ✅ | Overkill |
| Factual lookup | ✅ | ✅ |
| Multi-hop reasoning | ❌ | ✅ |
| "How is X related to Y?" | ❌ | ✅ |
| Entity relationships | ❌ | ✅ |
| Compliance/audit trails | ❌ | ✅ |
| Summarizing themes | ❌ | ✅ |

## Core Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      User Query                              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Query Analyzer                            │
│         (Determine: vector, graph, or hybrid?)               │
└─────────────────────────────────────────────────────────────┘
                    │                    │
          ┌────────┴────────┐  ┌────────┴────────┐
          ▼                 ▼  ▼                 ▼
┌─────────────────┐  ┌─────────────────┐
│  Vector Search  │  │  Graph Traverse │
│  (Semantic)     │  │  (Structured)   │
└─────────────────┘  └─────────────────┘
          │                    │
          └────────┬──────────┘
                   ▼
┌─────────────────────────────────────────────────────────────┐
│                  Context Fusion                              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    LLM Generation                            │
└─────────────────────────────────────────────────────────────┘
```

## Pattern 1: Entity Extraction → Knowledge Graph

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from neo4j import GraphDatabase

# Step 1: Extract entities and relationships from documents
EXTRACTION_PROMPT = """Extract entities and relationships from this text.

Text: {text}

Return JSON format:
{{
  "entities": [
    {{"name": "...", "type": "Person|Organization|Concept|Event|Location"}}
  ],
  "relationships": [
    {{"source": "...", "target": "...", "type": "..."}}
  ]
}}
"""

async def extract_knowledge(text: str, llm: ChatOpenAI) -> dict:
    """Extract entities and relationships from text."""
    prompt = ChatPromptTemplate.from_template(EXTRACTION_PROMPT)
    chain = prompt | llm
    result = await chain.ainvoke({"text": text})
    return json.loads(result.content)


# Step 2: Store in Neo4j
class KnowledgeGraph:
    def __init__(self, uri: str, user: str, password: str):
        self.driver = GraphDatabase.driver(uri, auth=(user, password))

    def add_entity(self, name: str, entity_type: str, properties: dict = None):
        with self.driver.session() as session:
            session.run(
                f"""
                MERGE (e:{entity_type} {{name: $name}})
                SET e += $properties
                """,
                name=name,
                properties=properties or {}
            )

    def add_relationship(self, source: str, target: str, rel_type: str):
        with self.driver.session() as session:
            session.run(
                """
                MATCH (a {name: $source})
                MATCH (b {name: $target})
                MERGE (a)-[r:""" + rel_type + """]->(b)
                """,
                source=source,
                target=target
            )
```

## Pattern 2: Graph-Enhanced Retrieval

```python
from llama_index.core import PropertyGraphIndex
from llama_index.graph_stores.neo4j import Neo4jPropertyGraphStore

def create_property_graph_index(documents):
    """Create a property graph index with LlamaIndex."""

    # Connect to Neo4j
    graph_store = Neo4jPropertyGraphStore(
        username="neo4j",
        password="password",
        url="bolt://localhost:7687",
    )

    # Build index - automatically extracts entities/relationships
    index = PropertyGraphIndex.from_documents(
        documents,
        property_graph_store=graph_store,
        show_progress=True,
    )

    return index


def query_with_graph(index, query: str):
    """Query using both vector and graph retrieval."""

    # Create retriever that uses both paths
    retriever = index.as_retriever(
        include_text=True,  # Include original text chunks
        similarity_top_k=5,
    )

    # Get results
    nodes = retriever.retrieve(query)
    return nodes
```

## Pattern 3: Text-to-Cypher for Direct Graph Queries

```python
from langchain_community.graphs import Neo4jGraph
from langchain.chains import GraphCypherQAChain

def create_text_to_cypher_chain():
    """Create a chain that converts natural language to Cypher queries."""

    # Connect to Neo4j
    graph = Neo4jGraph(
        url="bolt://localhost:7687",
        username="neo4j",
        password="password"
    )

    # Print schema for debugging
    print(graph.schema)

    # Create chain
    chain = GraphCypherQAChain.from_llm(
        llm=ChatOpenAI(model="gpt-4", temperature=0),
        graph=graph,
        verbose=True,
        validate_cypher=True,  # Validate before executing
        return_intermediate_steps=True
    )

    return chain


# Usage
chain = create_text_to_cypher_chain()
result = chain.invoke({
    "query": "What companies has John Smith worked for?"
})
# Generated Cypher: MATCH (p:Person {name: 'John Smith'})-[:WORKED_AT]->(c:Company) RETURN c.name
```

## Pattern 4: Hybrid Vector + Graph Retrieval

```python
class HybridGraphRAG:
    """Combine vector similarity with graph traversal."""

    def __init__(self, vector_store, graph_store):
        self.vector_store = vector_store
        self.graph_store = graph_store

    def retrieve(self, query: str, top_k: int = 5) -> list[dict]:
        # 1. Vector search for relevant chunks
        vector_results = self.vector_store.similarity_search(query, k=top_k)

        # 2. Extract entities from query
        query_entities = self._extract_entities(query)

        # 3. Graph traversal from those entities
        graph_context = []
        for entity in query_entities:
            # Get 1-hop neighbors
            neighbors = self.graph_store.query(f"""
                MATCH (e {{name: '{entity}'}})-[r]-(n)
                RETURN e.name, type(r), n.name, n.description
                LIMIT 10
            """)
            graph_context.extend(neighbors)

        # 4. Combine results
        combined = {
            "vector_chunks": [r.page_content for r in vector_results],
            "graph_context": graph_context,
            "entities": query_entities
        }

        return combined

    def _extract_entities(self, text: str) -> list[str]:
        # Use NER or LLM to extract entities
        # Simplified version:
        prompt = f"Extract entity names from: {text}"
        # ... LLM call
        return entities
```

## Pattern 5: Microsoft GraphRAG (Community Detection)

```python
# Microsoft's GraphRAG approach uses community detection
# for global summarization queries

from graphrag.index import run_indexing
from graphrag.query import LocalSearch, GlobalSearch

# Index documents (creates communities)
await run_indexing(
    input_dir="./documents",
    output_dir="./index",
    config={
        "llm": {"model": "gpt-4"},
        "embeddings": {"model": "text-embedding-3-small"},
        "chunks": {"size": 300, "overlap": 100},
        "community_detection": {
            "algorithm": "leiden",
            "resolution": 1.0
        }
    }
)

# Local search (specific entity questions)
local = LocalSearch(index_dir="./index")
result = local.search("What is Company X's main product?")

# Global search (summarization across communities)
global_search = GlobalSearch(index_dir="./index")
result = global_search.search("What are the main themes in these documents?")
```

## When to Use Each Pattern

| Pattern | Use When |
|---------|----------|
| Entity Extraction → KG | Building from scratch, custom schema |
| Property Graph Index | Quick setup, LlamaIndex ecosystem |
| Text-to-Cypher | Existing graph, complex queries |
| Hybrid Vector + Graph | Need both semantic + structural |
| Microsoft GraphRAG | Large corpus, summarization queries |

## Best Practices

1. **Define your schema** - Know what entities and relationships matter
2. **Start simple** - Begin with 2-3 entity types, expand as needed
3. **Validate Cypher** - Always validate generated queries before execution
4. **Cache graph queries** - Graph traversals can be expensive
5. **Combine with vector** - Pure graph misses semantic similarity
6. **Test multi-hop** - Ensure 2-3 hop queries perform acceptably

## Common Pitfalls

- **Over-extraction**: Too many entities = noisy graph
- **Missing relationships**: Entities without connections are useless
- **Schema drift**: Inconsistent entity types break queries
- **No fallback**: Graph-only fails when entities not found

## Tools & Resources

- **Neo4j**: Production graph database
- **LlamaIndex PropertyGraphIndex**: Easy Python integration
- **Microsoft GraphRAG**: Community-based approach
- **Amazon Neptune**: Managed graph database
- **LangChain GraphCypherQAChain**: Text-to-Cypher chains
