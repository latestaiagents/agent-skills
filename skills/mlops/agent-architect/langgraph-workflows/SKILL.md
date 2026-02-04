---
name: langgraph-workflows
description: |
  Build agent workflows with LangGraph 1.0 state machines and graph patterns.
  Use this skill when creating agent graphs, implementing state machines,
  building multi-step agent workflows, or using LangGraph.
  Activate when: LangGraph, agent graph, state graph, agent workflow,
  graph nodes, conditional edges, agent state machine, ReAct agent.
---

# LangGraph Workflows (1.0)

**Build production-ready agent workflows with LangGraph's state machine architecture.**

## LangGraph 1.0 Overview

LangGraph is the standard for building stateful, multi-step agent applications:
- **Durable execution**: Survive failures and restarts
- **Human-in-the-loop**: Pause for approval, resume later
- **Streaming**: First-class support for token streaming
- **Debugging**: Full execution traces and replay

## Core Concepts

```
┌─────────────────────────────────────────────────────────────┐
│                        StateGraph                            │
│                                                              │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐                │
│   │  Node   │───▶│  Node   │───▶│  Node   │                │
│   └─────────┘    └─────────┘    └─────────┘                │
│       │              │              │                       │
│       │         Conditional         │                       │
│       │           Edge              │                       │
│       │              │              │                       │
│       │              ▼              │                       │
│       │         ┌─────────┐        │                       │
│       └────────▶│  Node   │◀───────┘                       │
│                 └─────────┘                                 │
│                                                              │
│   State flows through nodes, edges control routing          │
└─────────────────────────────────────────────────────────────┘
```

## Pattern 1: Basic State Graph

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
import operator

# Define state schema
class AgentState(TypedDict):
    messages: Annotated[list, operator.add]  # Append messages
    current_step: str
    result: str

# Define nodes (functions that transform state)
def process_input(state: AgentState) -> dict:
    """First node: process user input."""
    user_message = state["messages"][-1]
    return {
        "current_step": "processed",
        "messages": [{"role": "system", "content": f"Processing: {user_message}"}]
    }

def generate_response(state: AgentState) -> dict:
    """Second node: generate response."""
    # Call LLM here
    response = llm.invoke(state["messages"])
    return {
        "result": response.content,
        "messages": [{"role": "assistant", "content": response.content}]
    }

# Build graph
workflow = StateGraph(AgentState)

# Add nodes
workflow.add_node("process", process_input)
workflow.add_node("generate", generate_response)

# Add edges
workflow.set_entry_point("process")
workflow.add_edge("process", "generate")
workflow.add_edge("generate", END)

# Compile
app = workflow.compile()

# Run
result = app.invoke({
    "messages": [{"role": "user", "content": "Hello!"}],
    "current_step": "",
    "result": ""
})
```

## Pattern 2: Conditional Routing

```python
from typing import Literal

def should_use_tools(state: AgentState) -> Literal["tools", "respond"]:
    """Decide whether to use tools or respond directly."""
    last_message = state["messages"][-1]

    if "tool_calls" in last_message:
        return "tools"
    return "respond"

def call_tools(state: AgentState) -> dict:
    """Execute tool calls."""
    tool_calls = state["messages"][-1]["tool_calls"]
    results = []
    for call in tool_calls:
        result = execute_tool(call["name"], call["args"])
        results.append({"role": "tool", "content": result})
    return {"messages": results}

def respond(state: AgentState) -> dict:
    """Generate final response."""
    response = llm.invoke(state["messages"])
    return {"messages": [{"role": "assistant", "content": response.content}]}

# Build graph with conditional edges
workflow = StateGraph(AgentState)

workflow.add_node("agent", call_agent)
workflow.add_node("tools", call_tools)
workflow.add_node("respond", respond)

workflow.set_entry_point("agent")

# Conditional edge based on agent output
workflow.add_conditional_edges(
    "agent",
    should_use_tools,
    {
        "tools": "tools",
        "respond": "respond"
    }
)

# Tools loop back to agent
workflow.add_edge("tools", "agent")
workflow.add_edge("respond", END)

app = workflow.compile()
```

## Pattern 3: ReAct Agent

```python
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent

# Define tools
@tool
def search(query: str) -> str:
    """Search the web for information."""
    return tavily_client.search(query)

@tool
def calculator(expression: str) -> str:
    """Evaluate a math expression."""
    return str(eval(expression))

# Create ReAct agent (prebuilt)
llm = ChatOpenAI(model="gpt-4")
tools = [search, calculator]

agent = create_react_agent(llm, tools)

# Run
result = agent.invoke({
    "messages": [{"role": "user", "content": "What is 25 * 4 + the population of France?"}]
})
```

## Pattern 4: Parallel Execution

```python
from langgraph.graph import StateGraph

class ParallelState(TypedDict):
    query: str
    search_results: str
    db_results: str
    combined: str

def search_web(state: ParallelState) -> dict:
    """Search web in parallel."""
    results = web_search(state["query"])
    return {"search_results": results}

def search_database(state: ParallelState) -> dict:
    """Search database in parallel."""
    results = db_search(state["query"])
    return {"db_results": results}

def combine_results(state: ParallelState) -> dict:
    """Combine parallel results."""
    combined = f"Web: {state['search_results']}\nDB: {state['db_results']}"
    return {"combined": combined}

workflow = StateGraph(ParallelState)

workflow.add_node("search_web", search_web)
workflow.add_node("search_db", search_database)
workflow.add_node("combine", combine_results)

workflow.set_entry_point("search_web")

# Fan out: both searches start from entry
workflow.add_edge("__start__", "search_web")
workflow.add_edge("__start__", "search_db")

# Fan in: both must complete before combine
workflow.add_edge("search_web", "combine")
workflow.add_edge("search_db", "combine")

workflow.add_edge("combine", END)

app = workflow.compile()
```

## Pattern 5: Subgraphs

```python
# Define a subgraph for research
def create_research_subgraph():
    class ResearchState(TypedDict):
        topic: str
        sources: list
        summary: str

    def find_sources(state):
        sources = search_sources(state["topic"])
        return {"sources": sources}

    def summarize(state):
        summary = summarize_sources(state["sources"])
        return {"summary": summary}

    workflow = StateGraph(ResearchState)
    workflow.add_node("find", find_sources)
    workflow.add_node("summarize", summarize)
    workflow.set_entry_point("find")
    workflow.add_edge("find", "summarize")
    workflow.add_edge("summarize", END)

    return workflow.compile()

# Use subgraph in main graph
research_agent = create_research_subgraph()

class MainState(TypedDict):
    query: str
    research: str
    response: str

def do_research(state: MainState) -> dict:
    result = research_agent.invoke({"topic": state["query"]})
    return {"research": result["summary"]}

main_workflow = StateGraph(MainState)
main_workflow.add_node("research", do_research)
main_workflow.add_node("respond", generate_response)
# ... continue building
```

## State Reducers

Control how state updates are merged:

```python
from typing import Annotated
import operator

class AgentState(TypedDict):
    # Append mode: new values added to list
    messages: Annotated[list, operator.add]

    # Overwrite mode: new value replaces old (default)
    current_step: str

    # Custom reducer
    counter: Annotated[int, lambda old, new: old + new]

# Custom reducer function
def merge_dicts(old: dict, new: dict) -> dict:
    """Custom reducer that merges dictionaries."""
    return {**old, **new}

class StateWithMerge(TypedDict):
    metadata: Annotated[dict, merge_dicts]
```

## Streaming

```python
# Stream tokens
async for chunk in app.astream(
    {"messages": [{"role": "user", "content": "Hello"}]},
    stream_mode="values"
):
    print(chunk)

# Stream events (more granular)
async for event in app.astream_events(
    {"messages": [{"role": "user", "content": "Hello"}]},
    version="v2"
):
    if event["event"] == "on_chat_model_stream":
        print(event["data"]["chunk"].content, end="")
```

## Best Practices

1. **Keep nodes focused** - Each node does one thing well
2. **Use TypedDict for state** - Type safety and documentation
3. **Prefer Annotated reducers** - Explicit state merge behavior
4. **Add checkpointing early** - Don't retrofit durability
5. **Test edges** - Conditional routing is where bugs hide

## Common Patterns Cheat Sheet

| Pattern | Use Case |
|---------|----------|
| Linear | Simple pipelines |
| Conditional | Branching logic |
| Cycle | Agent loops (ReAct) |
| Fan-out/in | Parallel execution |
| Subgraph | Reusable components |

## Debugging

```python
# Enable debug mode
app = workflow.compile(debug=True)

# Get graph visualization
print(app.get_graph().draw_mermaid())

# Inspect state at each step
for step in app.stream(input_state, stream_mode="debug"):
    print(f"Node: {step['node']}")
    print(f"State: {step['state']}")
```
