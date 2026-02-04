---
name: agent-checkpointing
description: |
  Implement checkpointing for agent recovery, debugging, and replay.
  Use this skill when building recoverable agents, implementing replay,
  debugging agent failures, or creating resumable workflows.
  Activate when: agent checkpoint, agent recovery, resume agent, agent restart,
  workflow replay, agent debugging, failure recovery, state snapshot.
---

# Agent Checkpointing

**Save, restore, replay, and debug agent execution with checkpoints.**

## Why Checkpointing?

- **Recovery**: Resume from failure without losing progress
- **Debugging**: Replay exact execution path
- **Branching**: Try different paths from same checkpoint
- **Audit**: Complete history of agent decisions
- **Testing**: Reproduce specific scenarios

## Checkpoint Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Checkpoint Timeline                       │
│                                                              │
│   CP-1        CP-2        CP-3        CP-4                  │
│    │           │           │           │                     │
│    ▼           ▼           ▼           ▼                     │
│  ┌───┐       ┌───┐       ┌───┐       ┌───┐                  │
│  │ S │──────▶│ S │──────▶│ S │──────▶│ S │                  │
│  └───┘       └───┘       └───┘       └───┘                  │
│  State       State       State       State                   │
│    +           +           +           +                     │
│  Metadata   Metadata   Metadata   Metadata                  │
│                              │                               │
│                              │  Rewind here                  │
│                              ▼                               │
│                            ┌───┐                             │
│                            │ S │──────▶ New Branch          │
│                            └───┘                             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Basic Checkpointing

### Setup

```python
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.sqlite import SqliteSaver
from langgraph.checkpoint.postgres import PostgresSaver

# Development: SQLite
checkpointer = SqliteSaver.from_conn_string("./checkpoints.db")

# Production: PostgreSQL
checkpointer = PostgresSaver.from_conn_string(
    "postgresql://user:pass@host/db"
)

# Compile graph with checkpointer
app = workflow.compile(checkpointer=checkpointer)
```

### Automatic Checkpointing

```python
# Every node completion creates a checkpoint automatically
config = {"configurable": {"thread_id": "my-thread"}}

# Run agent
result = app.invoke({"input": "Process this"}, config)

# Checkpoints created at each step
```

### Inspecting Checkpoints

```python
# Get current state
current_state = app.get_state(config)
print(f"Current checkpoint: {current_state.config}")
print(f"State values: {current_state.values}")
print(f"Next node: {current_state.next}")

# Get full checkpoint history
for checkpoint in app.get_state_history(config):
    print(f"""
    Checkpoint ID: {checkpoint.config['configurable']['checkpoint_id']}
    Created: {checkpoint.metadata.get('created_at')}
    Node: {checkpoint.metadata.get('source')}
    State: {checkpoint.values}
    """)
```

## Recovery Patterns

### Resume After Failure

```python
async def run_with_recovery(
    app,
    input_state: dict,
    thread_id: str,
    max_retries: int = 3
) -> dict:
    """Run agent with automatic recovery on failure."""
    config = {"configurable": {"thread_id": thread_id}}

    for attempt in range(max_retries):
        try:
            # Check for existing state
            existing = app.get_state(config)

            if existing and existing.values and existing.next:
                # Resume from checkpoint
                logger.info(f"Resuming from checkpoint, attempt {attempt + 1}")
                return await app.ainvoke(None, config)
            else:
                # Start fresh
                logger.info(f"Starting fresh, attempt {attempt + 1}")
                return await app.ainvoke(input_state, config)

        except Exception as e:
            logger.error(f"Attempt {attempt + 1} failed: {e}")
            if attempt == max_retries - 1:
                raise

            # Wait before retry
            await asyncio.sleep(2 ** attempt)

    raise RuntimeError("Max retries exceeded")
```

### Manual Recovery

```python
def recover_from_checkpoint(
    app,
    thread_id: str,
    checkpoint_id: str = None
) -> dict:
    """Manually recover from a specific checkpoint."""

    if checkpoint_id:
        # Resume from specific checkpoint
        config = {
            "configurable": {
                "thread_id": thread_id,
                "checkpoint_id": checkpoint_id
            }
        }
    else:
        # Resume from latest
        config = {"configurable": {"thread_id": thread_id}}

    # Get state at checkpoint
    state = app.get_state(config)
    logger.info(f"Recovering from: {state.config}")
    logger.info(f"State: {state.values}")
    logger.info(f"Next node: {state.next}")

    # Continue execution
    return app.invoke(None, config)
```

## Replay and Debugging

### Replay Execution

```python
def replay_execution(app, thread_id: str):
    """Replay entire execution history."""
    config = {"configurable": {"thread_id": thread_id}}

    # Get all checkpoints in order
    history = list(app.get_state_history(config))
    history.reverse()  # Oldest first

    print("=== Execution Replay ===\n")

    for i, checkpoint in enumerate(history):
        print(f"Step {i + 1}: {checkpoint.metadata.get('source', 'start')}")
        print(f"  Checkpoint: {checkpoint.config['configurable']['checkpoint_id']}")
        print(f"  State: {json.dumps(checkpoint.values, indent=4)}")
        print()
```

### Branch from Checkpoint

```python
def branch_from_checkpoint(
    app,
    source_thread_id: str,
    checkpoint_id: str,
    new_thread_id: str,
    modified_state: dict = None
) -> dict:
    """Create a new branch from an existing checkpoint."""

    # Get state at checkpoint
    source_config = {
        "configurable": {
            "thread_id": source_thread_id,
            "checkpoint_id": checkpoint_id
        }
    }
    state = app.get_state(source_config)

    # Create new thread with same state
    new_config = {"configurable": {"thread_id": new_thread_id}}

    # Optionally modify state
    if modified_state:
        merged_state = {**state.values, **modified_state}
    else:
        merged_state = state.values

    # Initialize new branch
    app.update_state(new_config, merged_state)

    logger.info(f"Created branch {new_thread_id} from {checkpoint_id}")
    return app.get_state(new_config)
```

### Compare Branches

```python
def compare_branches(
    app,
    thread_id_a: str,
    thread_id_b: str
) -> dict:
    """Compare final states of two branches."""
    config_a = {"configurable": {"thread_id": thread_id_a}}
    config_b = {"configurable": {"thread_id": thread_id_b}}

    state_a = app.get_state(config_a)
    state_b = app.get_state(config_b)

    # Find differences
    differences = {}
    all_keys = set(state_a.values.keys()) | set(state_b.values.keys())

    for key in all_keys:
        val_a = state_a.values.get(key)
        val_b = state_b.values.get(key)
        if val_a != val_b:
            differences[key] = {"branch_a": val_a, "branch_b": val_b}

    return {
        "branch_a": thread_id_a,
        "branch_b": thread_id_b,
        "differences": differences,
        "identical": len(differences) == 0
    }
```

## Checkpoint Management

### Cleanup Old Checkpoints

```python
from datetime import datetime, timedelta

async def cleanup_checkpoints(
    checkpointer,
    max_age_days: int = 30,
    max_per_thread: int = 100
):
    """Clean up old checkpoints to save storage."""

    # Age-based cleanup
    cutoff = datetime.now() - timedelta(days=max_age_days)

    if hasattr(checkpointer, 'conn'):  # Postgres
        await checkpointer.conn.execute("""
            DELETE FROM checkpoints
            WHERE created_at < $1
        """, cutoff)

    # Keep only latest N per thread
    await checkpointer.conn.execute("""
        DELETE FROM checkpoints
        WHERE id NOT IN (
            SELECT id FROM (
                SELECT id,
                       ROW_NUMBER() OVER (
                           PARTITION BY thread_id
                           ORDER BY created_at DESC
                       ) as rn
                FROM checkpoints
            ) ranked
            WHERE rn <= $1
        )
    """, max_per_thread)

    logger.info(f"Checkpoint cleanup complete")
```

### Export/Import Checkpoints

```python
import json

def export_checkpoint(app, thread_id: str, checkpoint_id: str) -> str:
    """Export checkpoint to JSON for backup or transfer."""
    config = {
        "configurable": {
            "thread_id": thread_id,
            "checkpoint_id": checkpoint_id
        }
    }

    state = app.get_state(config)

    export_data = {
        "thread_id": thread_id,
        "checkpoint_id": checkpoint_id,
        "values": state.values,
        "metadata": state.metadata,
        "exported_at": datetime.now().isoformat()
    }

    return json.dumps(export_data, default=str)


def import_checkpoint(
    app,
    export_json: str,
    new_thread_id: str = None
) -> dict:
    """Import checkpoint from JSON."""
    data = json.loads(export_json)

    thread_id = new_thread_id or data["thread_id"]
    config = {"configurable": {"thread_id": thread_id}}

    # Restore state
    app.update_state(config, data["values"])

    logger.info(f"Imported checkpoint to thread {thread_id}")
    return app.get_state(config)
```

## Testing with Checkpoints

```python
import pytest

class TestAgentWithCheckpoints:
    """Test suite using checkpoint replay."""

    @pytest.fixture
    def app(self):
        return workflow.compile(checkpointer=MemorySaver())

    def test_recovery_from_failure(self, app):
        """Test that agent recovers correctly."""
        config = {"configurable": {"thread_id": "test-1"}}

        # Run partially
        app.invoke({"input": "start"}, config)

        # Simulate failure at node 2
        state = app.get_state(config)

        # Recovery should continue from checkpoint
        result = app.invoke(None, config)
        assert result["status"] == "complete"

    def test_branch_produces_different_result(self, app):
        """Test branching with modified input."""
        config_a = {"configurable": {"thread_id": "test-a"}}

        # Run original
        result_a = app.invoke({"input": "original"}, config_a)

        # Get checkpoint before final step
        history = list(app.get_state_history(config_a))
        mid_checkpoint = history[1].config["configurable"]["checkpoint_id"]

        # Branch with different input
        branch_from_checkpoint(
            app, "test-a", mid_checkpoint, "test-b",
            {"input": "modified"}
        )

        result_b = app.invoke(None, {"configurable": {"thread_id": "test-b"}})

        # Results should differ
        assert result_a != result_b
```

## Best Practices

1. **Checkpoint at decision points** - Not after every micro-step
2. **Include metadata** - Timestamp, source node, version
3. **Size limits** - Don't checkpoint huge objects
4. **Regular cleanup** - Old checkpoints consume storage
5. **Test recovery** - Verify resume actually works
6. **Versioning** - Handle schema changes gracefully
