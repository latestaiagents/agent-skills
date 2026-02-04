---
name: human-in-loop-agents
description: |
  Build agents that pause for human approval, review, and intervention.
  Use this skill when implementing approval workflows, human oversight,
  agent interrupts, or review-before-execute patterns.
  Activate when: human in the loop, HITL, agent approval, human oversight,
  interrupt agent, pause agent, review workflow, agent supervision.
---

# Human-in-the-Loop Agents

**Build agents that know when to stop and ask for human judgment.**

## Why Human-in-the-Loop?

Critical for:
- **High-stakes actions**: Financial transactions, data deletion
- **Compliance**: Audit requirements, approval workflows
- **Quality control**: Review before publishing, sending
- **Edge cases**: When agent confidence is low
- **Trust building**: Users control what agents do

## Core Patterns

### Pattern 1: Interrupt Before Action

```python
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.memory import MemorySaver
from langgraph.types import interrupt

class AgentState(TypedDict):
    messages: list
    pending_action: dict | None
    approved: bool

def plan_action(state: AgentState) -> dict:
    """Agent plans what to do."""
    # Determine action based on messages
    action = {
        "type": "send_email",
        "to": "user@example.com",
        "subject": "Important Update",
        "body": "..."
    }
    return {"pending_action": action, "approved": False}

def request_approval(state: AgentState) -> dict:
    """Interrupt and wait for human approval."""
    action = state["pending_action"]

    # This pauses execution and waits for human input
    approved = interrupt({
        "message": f"Approve this action?",
        "action": action,
        "options": ["approve", "reject", "modify"]
    })

    return {"approved": approved == "approve"}

def execute_action(state: AgentState) -> dict:
    """Execute the approved action."""
    if state["approved"]:
        result = execute(state["pending_action"])
        return {"messages": [{"role": "system", "content": f"Executed: {result}"}]}
    else:
        return {"messages": [{"role": "system", "content": "Action rejected"}]}

def should_execute(state: AgentState) -> str:
    return "execute" if state["approved"] else "end"

# Build graph
workflow = StateGraph(AgentState)
workflow.add_node("plan", plan_action)
workflow.add_node("approve", request_approval)
workflow.add_node("execute", execute_action)

workflow.set_entry_point("plan")
workflow.add_edge("plan", "approve")
workflow.add_conditional_edges("approve", should_execute, {
    "execute": "execute",
    "end": END
})
workflow.add_edge("execute", END)

# Compile with checkpointer (required for interrupts)
app = workflow.compile(checkpointer=MemorySaver())
```

### Using the Interrupt

```python
# Start the agent
config = {"configurable": {"thread_id": "user-123"}}
result = app.invoke({"messages": [{"role": "user", "content": "Send the report"}]}, config)

# Agent pauses at interrupt - check state
state = app.get_state(config)
print(state.next)  # Shows we're waiting at 'approve' node
print(state.tasks)  # Shows the interrupt details

# Human reviews and approves
app.update_state(
    config,
    {"approved": True},
    as_node="approve"  # Resume from this node
)

# Continue execution
final_result = app.invoke(None, config)  # None = continue from checkpoint
```

### Pattern 2: Edit Before Continue

```python
def generate_draft(state: AgentState) -> dict:
    """Generate content for review."""
    draft = llm.invoke("Generate a response...")
    return {"draft": draft.content}

def human_edit(state: AgentState) -> dict:
    """Allow human to edit the draft."""
    current_draft = state["draft"]

    # Interrupt with editable content
    edited = interrupt({
        "type": "edit",
        "content": current_draft,
        "instructions": "Review and edit this draft before sending"
    })

    return {"draft": edited}

def send_response(state: AgentState) -> dict:
    """Send the (possibly edited) response."""
    send_message(state["draft"])
    return {"sent": True}
```

### Pattern 3: Confidence-Based Interrupt

```python
def agent_with_confidence(state: AgentState) -> dict:
    """Agent that knows when to ask for help."""
    response = llm.invoke(
        state["messages"],
        # Request confidence score
        response_format={"type": "json_object"}
    )

    result = json.loads(response.content)
    return {
        "answer": result["answer"],
        "confidence": result["confidence"]
    }

def check_confidence(state: AgentState) -> str:
    """Route based on confidence."""
    if state["confidence"] < 0.7:
        return "human_review"
    return "respond"

def human_review(state: AgentState) -> dict:
    """Low confidence - get human help."""
    review = interrupt({
        "type": "review",
        "agent_answer": state["answer"],
        "confidence": state["confidence"],
        "question": "Is this answer correct? Edit if needed."
    })

    return {"answer": review.get("edited_answer", state["answer"])}
```

### Pattern 4: Batch Approval

```python
class BatchApprovalState(TypedDict):
    items: list
    approved_items: list
    rejected_items: list

def collect_items(state: BatchApprovalState) -> dict:
    """Collect items that need approval."""
    items = get_pending_items()
    return {"items": items}

def batch_approve(state: BatchApprovalState) -> dict:
    """Present batch for approval."""
    decisions = interrupt({
        "type": "batch_approval",
        "items": state["items"],
        "instructions": "Review each item. Select items to approve."
    })

    # decisions = {"approved": [id1, id2], "rejected": [id3]}
    return {
        "approved_items": decisions["approved"],
        "rejected_items": decisions["rejected"]
    }

def process_approved(state: BatchApprovalState) -> dict:
    """Process only approved items."""
    for item_id in state["approved_items"]:
        process_item(item_id)
    return {}
```

## Timeout and Escalation

```python
import asyncio
from datetime import datetime, timedelta

class TimedApprovalState(TypedDict):
    action: dict
    requested_at: str
    approved: bool | None
    escalated: bool

async def request_with_timeout(state: TimedApprovalState) -> dict:
    """Request approval with timeout."""
    state["requested_at"] = datetime.now().isoformat()

    # Interrupt for approval
    try:
        result = interrupt({
            "type": "approval",
            "action": state["action"],
            "timeout_minutes": 30
        })
        return {"approved": result == "approve"}
    except TimeoutError:
        return {"approved": None, "escalated": True}

def check_approval_status(state: TimedApprovalState) -> str:
    if state["approved"] is True:
        return "execute"
    elif state["approved"] is False:
        return "reject"
    elif state["escalated"]:
        return "escalate"
    return "wait"

def escalate_to_manager(state: TimedApprovalState) -> dict:
    """Escalate after timeout."""
    notify_manager(state["action"])
    return {"messages": [{"content": "Escalated to manager for approval"}]}
```

## UI Integration

### Webhook-Based Approval

```python
from fastapi import FastAPI, HTTPException
import uuid

app = FastAPI()
pending_approvals = {}

@app.post("/agent/start")
async def start_agent(request: dict):
    """Start agent, returns approval_id if interrupt."""
    thread_id = str(uuid.uuid4())
    config = {"configurable": {"thread_id": thread_id}}

    result = agent_app.invoke(request["input"], config)

    # Check if waiting for approval
    state = agent_app.get_state(config)
    if state.next:  # Interrupted
        approval_id = str(uuid.uuid4())
        pending_approvals[approval_id] = {
            "thread_id": thread_id,
            "interrupt_data": state.tasks[0].interrupts[0]
        }
        return {
            "status": "pending_approval",
            "approval_id": approval_id,
            "details": state.tasks[0].interrupts[0]
        }

    return {"status": "complete", "result": result}

@app.post("/agent/approve/{approval_id}")
async def approve_action(approval_id: str, decision: dict):
    """Handle approval decision."""
    if approval_id not in pending_approvals:
        raise HTTPException(404, "Approval not found")

    pending = pending_approvals.pop(approval_id)
    config = {"configurable": {"thread_id": pending["thread_id"]}}

    # Update state with decision
    agent_app.update_state(
        config,
        {"approved": decision["approved"]},
        as_node="approve"
    )

    # Continue execution
    result = agent_app.invoke(None, config)
    return {"status": "complete", "result": result}
```

## Best Practices

1. **Clear context**: Tell humans exactly what they're approving
2. **Reasonable defaults**: Pre-fill with agent's best guess
3. **Timeout handling**: Don't block forever
4. **Audit logging**: Record all approvals/rejections
5. **Batch when possible**: Don't interrupt for every small thing
6. **Graceful degradation**: Have fallback if no response

## When to Require Approval

| Action Type | Approval Needed | Reason |
|------------|-----------------|---------|
| Read data | No | Low risk |
| Send email | Yes | External impact |
| Delete data | Yes | Irreversible |
| Financial transaction | Yes | High stakes |
| Publish content | Yes | Reputation risk |
| Internal logging | No | Low risk |

## Monitoring HITL Metrics

```python
# Track approval patterns
metrics = {
    "total_interrupts": 0,
    "approved": 0,
    "rejected": 0,
    "edited": 0,
    "timed_out": 0,
    "avg_response_time_seconds": 0
}

# Analyze to improve agent
# High rejection rate = agent needs improvement
# High edit rate = agent close but not quite
# High timeout rate = approval process too slow
```
