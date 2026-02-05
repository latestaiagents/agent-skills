---
name: a2a-protocols
description: |
  Implement Agent-to-Agent (A2A) communication for cross-framework interoperability.
  Use this skill when building multi-agent communication, implementing agent protocols,
  connecting agents across frameworks, or standardizing agent interfaces.
  Activate when: agent to agent, A2A, agent communication, agent protocol,
  cross-framework agents, agent interoperability, MCP, agent discovery.
---

# Agent-to-Agent (A2A) Protocols

**Enable agents to discover, communicate, and collaborate across frameworks and systems.**

## The Interoperability Challenge

Different agent frameworks:
- LangChain/LangGraph
- AutoGen
- CrewAI
- Custom implementations

**Problem**: They can't talk to each other natively.

**Solution**: Standard protocols for agent communication.

## Protocol Landscape (2026)

| Protocol | Purpose | Adoption |
|----------|---------|----------|
| **MCP** (Model Context Protocol) | Tool/resource sharing | High (Anthropic-backed) |
| **A2A** (Agent-to-Agent) | Agent coordination | Growing |
| **OpenAI Agents Protocol** | Agent invocation | OpenAI ecosystem |
| **Custom REST/gRPC** | Point-to-point | Common |

## MCP Integration

### MCP Overview

MCP standardizes how agents access tools and resources:

```
┌─────────────────┐          ┌─────────────────┐
│     Agent       │          │   MCP Server    │
│  (MCP Client)   │◀────────▶│  (Tools/Data)   │
└─────────────────┘          └─────────────────┘
         │
         │  MCP Protocol
         │  - List tools
         │  - Call tools
         │  - Access resources
         ▼
┌─────────────────┐
│  Another Agent  │
│  (MCP Client)   │
└─────────────────┘
```

### Creating an MCP Server

```python
from mcp.server import Server
from mcp.types import Tool, TextContent

# Create MCP server
server = Server("my-agent-tools")

@server.tool()
async def search_database(query: str) -> str:
    """Search the internal database."""
    results = await db.search(query)
    return json.dumps(results)

@server.tool()
async def send_notification(
    recipient: str,
    message: str
) -> str:
    """Send a notification to a user."""
    await notifications.send(recipient, message)
    return "Notification sent"

# Run server
if __name__ == "__main__":
    server.run()
```

### Connecting Agent to MCP

```python
from mcp import ClientSession, StdioServerParameters
from langchain_mcp import MCPToolkit

async def create_mcp_agent():
    """Create agent with MCP tools."""

    # Connect to MCP server
    server_params = StdioServerParameters(
        command="python",
        args=["mcp_server.py"]
    )

    async with ClientSession(server_params) as session:
        # Get tools from MCP
        toolkit = MCPToolkit(session=session)
        tools = toolkit.get_tools()

        # Create LangChain agent with MCP tools
        agent = create_react_agent(llm, tools)

        return agent
```

## Agent-to-Agent Communication

### Message-Based Protocol

```python
from dataclasses import dataclass
from enum import Enum
from typing import Any
import uuid

class MessageType(Enum):
    REQUEST = "request"
    RESPONSE = "response"
    BROADCAST = "broadcast"
    HANDOFF = "handoff"

@dataclass
class AgentMessage:
    """Standard message format for A2A communication."""
    id: str
    type: MessageType
    sender: str
    recipient: str | None  # None for broadcasts
    content: dict
    reply_to: str | None = None
    timestamp: str = None

    def __post_init__(self):
        if not self.id:
            self.id = str(uuid.uuid4())
        if not self.timestamp:
            self.timestamp = datetime.now().isoformat()

    def to_dict(self) -> dict:
        return {
            "id": self.id,
            "type": self.type.value,
            "sender": self.sender,
            "recipient": self.recipient,
            "content": self.content,
            "reply_to": self.reply_to,
            "timestamp": self.timestamp
        }

    @classmethod
    def from_dict(cls, data: dict) -> "AgentMessage":
        return cls(
            id=data["id"],
            type=MessageType(data["type"]),
            sender=data["sender"],
            recipient=data.get("recipient"),
            content=data["content"],
            reply_to=data.get("reply_to"),
            timestamp=data.get("timestamp")
        )
```

### Agent Registry

```python
class AgentRegistry:
    """Central registry for agent discovery."""

    def __init__(self, redis_client):
        self.redis = redis_client

    async def register(
        self,
        agent_id: str,
        capabilities: list[str],
        endpoint: str,
        metadata: dict = None
    ):
        """Register an agent with its capabilities."""
        agent_info = {
            "agent_id": agent_id,
            "capabilities": capabilities,
            "endpoint": endpoint,
            "metadata": metadata or {},
            "registered_at": datetime.now().isoformat(),
            "last_heartbeat": datetime.now().isoformat()
        }

        # Store agent info
        await self.redis.hset(
            "agents",
            agent_id,
            json.dumps(agent_info)
        )

        # Index by capability
        for cap in capabilities:
            await self.redis.sadd(f"capability:{cap}", agent_id)

    async def find_by_capability(self, capability: str) -> list[dict]:
        """Find agents with a specific capability."""
        agent_ids = await self.redis.smembers(f"capability:{capability}")
        agents = []
        for aid in agent_ids:
            info = await self.redis.hget("agents", aid)
            if info:
                agents.append(json.loads(info))
        return agents

    async def heartbeat(self, agent_id: str):
        """Update agent heartbeat."""
        info = await self.redis.hget("agents", agent_id)
        if info:
            data = json.loads(info)
            data["last_heartbeat"] = datetime.now().isoformat()
            await self.redis.hset("agents", agent_id, json.dumps(data))
```

### Inter-Agent Communication

```python
class AgentCommunicator:
    """Handle agent-to-agent communication."""

    def __init__(self, agent_id: str, registry: AgentRegistry):
        self.agent_id = agent_id
        self.registry = registry
        self.message_handlers = {}

    def on_message(self, message_type: MessageType):
        """Decorator to register message handlers."""
        def decorator(func):
            self.message_handlers[message_type] = func
            return func
        return decorator

    async def send(
        self,
        recipient: str,
        content: dict,
        message_type: MessageType = MessageType.REQUEST
    ) -> AgentMessage:
        """Send message to another agent."""
        # Get recipient endpoint
        recipient_info = await self.registry.get(recipient)
        if not recipient_info:
            raise ValueError(f"Agent {recipient} not found")

        message = AgentMessage(
            id=str(uuid.uuid4()),
            type=message_type,
            sender=self.agent_id,
            recipient=recipient,
            content=content
        )

        # Send via HTTP
        async with aiohttp.ClientSession() as session:
            async with session.post(
                f"{recipient_info['endpoint']}/messages",
                json=message.to_dict()
            ) as response:
                return await response.json()

    async def request(
        self,
        capability: str,
        content: dict,
        timeout: float = 30.0
    ) -> dict:
        """Find agent by capability and send request."""
        agents = await self.registry.find_by_capability(capability)
        if not agents:
            raise ValueError(f"No agent with capability: {capability}")

        # Simple: pick first available
        target = agents[0]

        response = await self.send(
            target["agent_id"],
            content,
            MessageType.REQUEST
        )

        return response

    async def broadcast(self, content: dict, capability: str = None):
        """Broadcast message to multiple agents."""
        if capability:
            agents = await self.registry.find_by_capability(capability)
        else:
            agents = await self.registry.get_all()

        tasks = [
            self.send(a["agent_id"], content, MessageType.BROADCAST)
            for a in agents
        ]

        return await asyncio.gather(*tasks, return_exceptions=True)
```

## Handoff Protocol

```python
@dataclass
class HandoffRequest:
    """Request to hand off a task to another agent."""
    task_id: str
    task_type: str
    context: dict
    state: dict
    reason: str
    required_capabilities: list[str]

@dataclass
class HandoffResponse:
    """Response to handoff request."""
    accepted: bool
    agent_id: str
    reason: str = None

class HandoffProtocol:
    """Protocol for handing off tasks between agents."""

    def __init__(self, communicator: AgentCommunicator):
        self.comm = communicator

    async def request_handoff(
        self,
        request: HandoffRequest
    ) -> HandoffResponse:
        """Request another agent to take over a task."""

        # Find capable agents
        candidates = []
        for cap in request.required_capabilities:
            agents = await self.comm.registry.find_by_capability(cap)
            candidates.extend(agents)

        # Deduplicate
        candidates = {a["agent_id"]: a for a in candidates}.values()

        # Request handoff from each until accepted
        for candidate in candidates:
            response = await self.comm.send(
                candidate["agent_id"],
                {
                    "type": "handoff_request",
                    "request": asdict(request)
                },
                MessageType.HANDOFF
            )

            if response.get("accepted"):
                return HandoffResponse(
                    accepted=True,
                    agent_id=candidate["agent_id"]
                )

        return HandoffResponse(
            accepted=False,
            agent_id=None,
            reason="No agent accepted the handoff"
        )

    async def accept_handoff(
        self,
        request: HandoffRequest
    ) -> bool:
        """Accept a handoff and begin processing."""
        # Validate we can handle it
        # ... capability check

        # Restore state
        # ... state restoration

        return True
```

## REST API for Agents

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()

class MessageRequest(BaseModel):
    sender: str
    content: dict
    message_type: str = "request"

@app.post("/agents/{agent_id}/messages")
async def receive_message(agent_id: str, message: MessageRequest):
    """Receive message from another agent."""
    if agent_id != MY_AGENT_ID:
        raise HTTPException(404, "Agent not found")

    # Process message
    result = await agent.process_message(
        AgentMessage(
            id=str(uuid.uuid4()),
            type=MessageType(message.message_type),
            sender=message.sender,
            recipient=agent_id,
            content=message.content
        )
    )

    return {"status": "processed", "result": result}

@app.get("/agents/{agent_id}/capabilities")
async def get_capabilities(agent_id: str):
    """Get agent capabilities."""
    return {
        "agent_id": agent_id,
        "capabilities": ["research", "summarization", "coding"],
        "status": "active"
    }

@app.post("/agents/{agent_id}/handoff")
async def handle_handoff(agent_id: str, request: dict):
    """Handle handoff request."""
    handoff = HandoffRequest(**request)

    # Check if we can accept
    can_accept = await agent.can_handle(handoff)

    if can_accept:
        await agent.accept_handoff(handoff)
        return {"accepted": True}
    else:
        return {"accepted": False, "reason": "Capacity exceeded"}
```

## Best Practices

1. **Version your protocol** - Include version in messages
2. **Timeout all requests** - Agents may be unavailable
3. **Idempotent handlers** - Messages may be retried
4. **Capability-based routing** - Don't hardcode agent IDs
5. **Health checks** - Remove dead agents from registry
6. **Rate limiting** - Prevent agent flooding
7. **Authentication** - Verify agent identity

## Security Considerations

```python
# Sign messages
import hmac
import hashlib

def sign_message(message: AgentMessage, secret: str) -> str:
    """Sign message for authenticity."""
    payload = json.dumps(message.to_dict(), sort_keys=True)
    return hmac.new(
        secret.encode(),
        payload.encode(),
        hashlib.sha256
    ).hexdigest()

def verify_signature(message: AgentMessage, signature: str, secret: str) -> bool:
    """Verify message signature."""
    expected = sign_message(message, secret)
    return hmac.compare_digest(signature, expected)
```
