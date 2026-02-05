---
name: durable-state-patterns
description: |
  Implement persistent agent state that survives failures and restarts.
  Use this skill when building stateful agents, implementing checkpointing,
  persisting agent memory across sessions, or recovering from failures.
  Activate when: durable state, agent persistence, checkpointing, agent recovery,
  stateful agents, state persistence, cross-session memory, agent restart.
---

# Durable State Patterns

**Build agents that remember state across failures, restarts, and sessions.**

## Why Durable State?

Without durability:
- Long-running agents lose progress on crash
- Users can't resume conversations after timeout
- No audit trail of agent decisions
- Expensive recomputation on every restart

With durability:
- Resume from any checkpoint
- Survive infrastructure failures
- Debug by replaying history
- Share state across instances

## LangGraph Checkpointing

### Basic Setup

```python
from langgraph.graph import StateGraph
from langgraph.checkpoint.memory import MemorySaver
from langgraph.checkpoint.sqlite import SqliteSaver
from langgraph.checkpoint.postgres import PostgresSaver

# In-memory (development only)
memory_checkpointer = MemorySaver()

# SQLite (single instance)
sqlite_checkpointer = SqliteSaver.from_conn_string("checkpoints.db")

# PostgreSQL (production, multi-instance)
postgres_checkpointer = PostgresSaver.from_conn_string(
    "postgresql://user:pass@localhost/db"
)

# Compile with checkpointer
app = workflow.compile(checkpointer=postgres_checkpointer)
```

### Thread-Based State

```python
from uuid import uuid4

# Each conversation gets a unique thread_id
thread_id = str(uuid4())

# First invocation
config = {"configurable": {"thread_id": thread_id}}
result1 = app.invoke(
    {"messages": [{"role": "user", "content": "My name is Alice"}]},
    config
)

# Later invocation (same thread = same state)
result2 = app.invoke(
    {"messages": [{"role": "user", "content": "What's my name?"}]},
    config
)
# Agent remembers: "Your name is Alice"

# Different thread = fresh state
other_config = {"configurable": {"thread_id": str(uuid4())}}
result3 = app.invoke(
    {"messages": [{"role": "user", "content": "What's my name?"}]},
    other_config
)
# Agent doesn't know: "I don't have that information"
```

## State Schema Design

### Versioned State

```python
from typing import TypedDict, Annotated
import operator

class AgentStateV1(TypedDict):
    """Version 1 of agent state."""
    messages: Annotated[list, operator.add]
    user_id: str

class AgentStateV2(TypedDict):
    """Version 2 with preferences."""
    messages: Annotated[list, operator.add]
    user_id: str
    preferences: dict  # New field
    state_version: int  # Track version

def migrate_v1_to_v2(old_state: AgentStateV1) -> AgentStateV2:
    """Migrate old state to new schema."""
    return {
        **old_state,
        "preferences": {},  # Default value
        "state_version": 2
    }
```

### Separate Concerns

```python
class ConversationState(TypedDict):
    """Short-term: current conversation."""
    messages: Annotated[list, operator.add]
    current_task: str

class UserProfileState(TypedDict):
    """Long-term: persists across conversations."""
    user_id: str
    name: str
    preferences: dict
    history_summary: str

class FullAgentState(TypedDict):
    """Combined state."""
    conversation: ConversationState
    profile: UserProfileState
```

## Memory Tiers

```
┌─────────────────────────────────────────────────────────────┐
│                    Memory Architecture                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │             Working Memory (In-Graph)                │   │
│   │   Current messages, tool results, intermediate       │   │
│   │   Lifetime: Single invocation                        │   │
│   └─────────────────────────────────────────────────────┘   │
│                              │                               │
│                              ▼                               │
│   ┌─────────────────────────────────────────────────────┐   │
│   │            Short-Term Memory (Thread)                │   │
│   │   Conversation history, session context              │   │
│   │   Lifetime: Single conversation/session              │   │
│   └─────────────────────────────────────────────────────┘   │
│                              │                               │
│                              ▼                               │
│   ┌─────────────────────────────────────────────────────┐   │
│   │            Long-Term Memory (User/Entity)            │   │
│   │   User preferences, facts, relationship history      │   │
│   │   Lifetime: Permanent                                │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Implementation

```python
import redis
from datetime import datetime

class ThreeTierMemory:
    """Three-tier memory system."""

    def __init__(self, redis_client: redis.Redis, checkpointer):
        self.redis = redis_client  # Short-term
        self.checkpointer = checkpointer  # Working memory via LangGraph
        self.db = PostgresDB()  # Long-term

    # Working memory: handled by LangGraph state

    # Short-term: Redis with TTL
    def get_session(self, session_id: str) -> dict:
        data = self.redis.get(f"session:{session_id}")
        return json.loads(data) if data else {}

    def save_session(self, session_id: str, data: dict, ttl: int = 3600):
        self.redis.setex(
            f"session:{session_id}",
            ttl,
            json.dumps(data)
        )

    # Long-term: Persistent database
    def get_user_profile(self, user_id: str) -> dict:
        return self.db.query(
            "SELECT * FROM user_profiles WHERE user_id = %s",
            (user_id,)
        )

    def update_user_profile(self, user_id: str, updates: dict):
        self.db.execute(
            "UPDATE user_profiles SET data = data || %s WHERE user_id = %s",
            (json.dumps(updates), user_id)
        )
```

## Checkpoint Management

### Manual Checkpoints

```python
from langgraph.checkpoint import Checkpoint

# Get current checkpoint
checkpoint = app.get_state(config)
print(f"Checkpoint ID: {checkpoint.config['configurable']['checkpoint_id']}")
print(f"State: {checkpoint.values}")

# List all checkpoints for a thread
history = list(app.get_state_history(config))
for checkpoint in history:
    print(f"{checkpoint.config['configurable']['checkpoint_id']}: {checkpoint.values}")

# Rewind to specific checkpoint
old_checkpoint_id = "some-checkpoint-id"
rewound_config = {
    "configurable": {
        "thread_id": thread_id,
        "checkpoint_id": old_checkpoint_id
    }
}
result = app.invoke(new_input, rewound_config)
```

### Checkpoint Cleanup

```python
async def cleanup_old_checkpoints(
    checkpointer,
    max_age_days: int = 30
):
    """Clean up checkpoints older than max_age_days."""
    cutoff = datetime.now() - timedelta(days=max_age_days)

    # Implementation depends on checkpointer
    # For Postgres:
    await checkpointer.conn.execute(
        "DELETE FROM checkpoints WHERE created_at < $1",
        cutoff
    )
```

## Recovery Patterns

### Automatic Retry

```python
from tenacity import retry, stop_after_attempt, wait_exponential

class ResilientAgent:
    def __init__(self, app, checkpointer):
        self.app = app
        self.checkpointer = checkpointer

    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=4, max=10)
    )
    async def invoke_with_retry(self, input_state: dict, config: dict):
        """Invoke with automatic retry on failure."""
        try:
            return await self.app.ainvoke(input_state, config)
        except Exception as e:
            # State is already checkpointed, will resume on retry
            logger.error(f"Invocation failed, retrying: {e}")
            raise
```

### Resume from Failure

```python
async def resume_or_start(
    app,
    thread_id: str,
    input_state: dict
) -> dict:
    """Resume existing thread or start new one."""
    config = {"configurable": {"thread_id": thread_id}}

    # Check for existing state
    existing = app.get_state(config)

    if existing and existing.values:
        # Resume from checkpoint
        logger.info(f"Resuming thread {thread_id}")
        return await app.ainvoke(None, config)  # None = continue from checkpoint
    else:
        # Start fresh
        logger.info(f"Starting new thread {thread_id}")
        return await app.ainvoke(input_state, config)
```

## Storage Backend Comparison

| Backend | Use Case | Pros | Cons |
|---------|----------|------|------|
| MemorySaver | Development | Fast, simple | Lost on restart |
| SqliteSaver | Single instance | Persistent, simple | No concurrency |
| PostgresSaver | Production | Scalable, durable | Setup complexity |
| RedisSaver | High throughput | Fast, distributed | Memory limits |

## Best Practices

1. **Choose thread_id wisely** - User ID, session ID, or conversation ID
2. **Version your state schema** - Plan for migrations
3. **Set checkpoint limits** - Don't keep infinite history
4. **Test recovery** - Simulate failures in staging
5. **Monitor checkpoint size** - Large state = slow operations
6. **Separate concerns** - Working vs short-term vs long-term

## Production Configuration

```python
from langgraph.checkpoint.postgres import PostgresSaver
import asyncpg

async def create_production_checkpointer():
    """Create production-ready checkpointer."""

    # Connection pool for concurrency
    pool = await asyncpg.create_pool(
        "postgresql://user:pass@localhost/db",
        min_size=5,
        max_size=20
    )

    checkpointer = PostgresSaver(pool)

    # Initialize tables
    await checkpointer.setup()

    return checkpointer

# Use in app
checkpointer = await create_production_checkpointer()
app = workflow.compile(checkpointer=checkpointer)
```
