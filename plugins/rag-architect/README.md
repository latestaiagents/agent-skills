# RAG Architect Plugin for Claude Code

**The complete toolkit for building production-grade RAG systems.**

From basic retrieval to advanced patterns like GraphRAG, Agentic RAG, and Corrective RAG - everything you need to build, evaluate, and deploy RAG pipelines.

## Quick Install

```bash
# Install via Claude Code CLI
/plugin install latestaiagents/agent-skills/plugins/rag-architect

# Or using npx
npx skills add latestaiagents/agent-skills/plugins/rag-architect
```

> **Note**: For complete plugin documentation, see the [official Claude Code plugins guide](https://code.claude.com/docs/en/plugins).

## What's Included

### 3 Slash Commands

| Command | Description |
|---------|-------------|
| `/build-rag` | Step-by-step guidance to build a RAG pipeline from scratch |
| `/rag-debug` | Diagnose retrieval and generation issues systematically |
| `/eval-rag` | Run comprehensive evaluation with RAGAS and custom metrics |

### 7 Professional Skills

| Skill | Description |
|-------|-------------|
| **Chunking Strategies** | Fixed, recursive, semantic, and document-aware splitting |
| **Hybrid Retrieval** | Combine dense vectors with BM25/keyword search |
| **Agentic RAG** | Multi-step retrieval with reasoning and tool use |
| **GraphRAG Patterns** | Knowledge graphs + vector search for relationship-aware RAG |
| **Corrective RAG** | Self-correcting pipelines with validation and fallbacks |
| **RAG Evaluation** | RAGAS metrics, LLM-as-judge, retrieval benchmarks |
| **Production RAG Checklist** | Deploy with caching, monitoring, and scaling |

## MCP Tool Integrations

Optional connections to popular RAG infrastructure:

| Tool | What It Enables |
|------|-----------------|
| **Pinecone** | Managed vector database queries |
| **Weaviate** | Hybrid search with native BM25 |
| **Qdrant** | High-performance vector operations |
| **Neo4j** | Graph database for GraphRAG |
| **Elasticsearch** | Keyword + vector hybrid search |
| **OpenAI** | Embeddings and generation |

## Usage Examples

### Building a RAG Pipeline

```
You: /build-rag

Claude: I'll help you build a RAG pipeline. What documents will you be querying?

You: Internal company docs, mostly markdown and PDFs

Claude: For mixed document types, I recommend...
[Provides chunking strategy, embedding model selection,
vector store setup, and complete implementation code]
```

### Debugging Poor Results

```
You: /rag-debug

Claude: Let's diagnose the issue. What query is producing poor results?

You: "What's our refund policy?" returns unrelated product docs

Claude: I'll analyze this systematically...
[Checks retrieval scores, chunk boundaries, examines
top results, identifies semantic gap, recommends hybrid search]
```

### Evaluating Quality

```
You: /eval-rag

Claude: I'll run a comprehensive evaluation. Do you have a test dataset?

You: Yes, 50 query-answer pairs

Claude: Running evaluation...
[Calculates Recall@5, MRR, faithfulness, relevance,
identifies weak queries, provides improvement plan]
```

## Skills Auto-Activation

Skills activate based on context:

- Mention "chunking" or "text splitting" → Chunking Strategies
- Ask about "hybrid search" or "BM25" → Hybrid Retrieval
- Discuss "GraphRAG" or "knowledge graph" → GraphRAG Patterns
- Say "RAG evaluation" or "RAGAS" → RAG Evaluation
- Mention "production RAG" or "deploy RAG" → Production Checklist

## Architecture Patterns Covered

```
Simple RAG:
Query → Retrieve → Generate → Answer

Hybrid RAG:
Query → [Vector + BM25] → Fuse → Rerank → Generate

Agentic RAG:
Query → Plan → [Retrieve → Analyze]*n → Synthesize

Corrective RAG:
Query → Retrieve → Grade → [Use/Search/Fallback] → Generate

GraphRAG:
Query → Entity Extract → Graph Traverse → Vector Search → Generate
```

## Requirements

- Claude Code CLI (latest version)
- Node.js 18+ (for npx)
- Python 3.9+ (for RAG implementations)
- Optional: Vector database access (Pinecone, Weaviate, etc.)

## Security & Trust

This plugin includes MCP server configurations for external tool integrations. Before enabling:

- **Review credentials**: Only add API keys for tools you trust
- **Principle of least privilege**: Use read-only keys where possible
- **Audit MCP servers**: Review `.mcp.json` before copying to your config

All MCP integrations are marked `optional: true`.

## Contributing

Found an issue or want to add a pattern?

1. Fork the repository
2. Create a feature branch
3. Submit a pull request

See [CLAUDE.md](../../CLAUDE.md) for development guidelines.

## License

MIT License - See [LICENSE](../../LICENSE) for details.

---

**Built by [latestaiagents](https://github.com/latestaiagents)** | Part of the [Agent Skills](https://github.com/latestaiagents/agent-skills) collection
