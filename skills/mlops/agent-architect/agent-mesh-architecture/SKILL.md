---
name: agent-mesh-architecture
description: |
  Use this skill when designing peer-to-peer multi-agent systems. Activate when the user needs agents
  that collaborate without central control, wants resilient agent networks, needs swarm-like agent behavior,
  or is building decentralized agent architectures.
---

# Agent Mesh Architecture

Design resilient peer-to-peer agent networks where agents collaborate without central orchestration.

## When to Use

- Resilience is critical (no single point of failure)
- Agents need to collaborate dynamically
- Tasks benefit from emergent behavior
- Scale varies (agents join/leave dynamically)
- Different perspectives improve outcomes

## Architecture Overview

```
    ┌─────────┐     ┌─────────┐
    │ Agent A │◄───►│ Agent B │
    └────┬────┘     └────┬────┘
         │               │
         │   ┌───────┐   │
         └──►│ Agent │◄──┘
             │   C   │
         ┌──►│       │◄──┐
         │   └───────┘   │
    ┌────┴────┐     ┌────┴────┐
    │ Agent D │◄───►│ Agent E │
    └─────────┘     └─────────┘
```

## Core Components

### 1. Agent Node

Each agent is an independent node with communication capabilities.

```typescript
interface AgentNode {
  id: string;
  capabilities: string[];
  state: AgentState;

  // Communication
  broadcast(message: Message): Promise<void>;
  sendTo(agentId: string, message: Message): Promise<void>;
  onMessage(handler: MessageHandler): void;

  // Discovery
  discoverPeers(): Promise<AgentNode[]>;
  announceCapability(capability: string): void;

  // Task handling
  canHandle(task: Task): boolean;
  handle(task: Task): Promise<Result>;
}

interface Message {
  type: 'task_request' | 'task_result' | 'collaboration_invite' |
        'capability_query' | 'heartbeat' | 'consensus_vote';
  from: string;
  to: string | 'broadcast';
  payload: unknown;
  timestamp: Date;
  correlationId: string;
}
```

### 2. Message Bus / Communication Layer

```typescript
interface MessageBus {
  publish(topic: string, message: Message): Promise<void>;
  subscribe(topic: string, handler: MessageHandler): Subscription;
  request(agentId: string, message: Message): Promise<Message>;
}

// In-memory implementation for local agents
class LocalMessageBus implements MessageBus {
  private subscribers = new Map<string, MessageHandler[]>();

  async publish(topic: string, message: Message) {
    const handlers = this.subscribers.get(topic) || [];
    await Promise.all(handlers.map(h => h(message)));
  }

  subscribe(topic: string, handler: MessageHandler) {
    const handlers = this.subscribers.get(topic) || [];
    handlers.push(handler);
    this.subscribers.set(topic, handlers);

    return {
      unsubscribe: () => {
        const idx = handlers.indexOf(handler);
        if (idx >= 0) handlers.splice(idx, 1);
      }
    };
  }
}
```

### 3. Service Discovery

```typescript
interface ServiceRegistry {
  register(agent: AgentNode): Promise<void>;
  deregister(agentId: string): Promise<void>;
  findByCapability(capability: string): Promise<AgentNode[]>;
  getAll(): Promise<AgentNode[]>;
  onAgentJoin(handler: (agent: AgentNode) => void): void;
  onAgentLeave(handler: (agentId: string) => void): void;
}

class AgentRegistry implements ServiceRegistry {
  private agents = new Map<string, AgentNode>();
  private capabilities = new Map<string, Set<string>>();

  async findByCapability(capability: string): Promise<AgentNode[]> {
    const agentIds = this.capabilities.get(capability) || new Set();
    return Array.from(agentIds)
      .map(id => this.agents.get(id))
      .filter(Boolean) as AgentNode[];
  }
}
```

## Communication Patterns

### Pattern 1: Request-Response

```typescript
// Agent A needs help from specialized agent
async function requestHelp(task: Task): Promise<Result> {
  // Find capable agents
  const candidates = await registry.findByCapability(task.requiredCapability);

  if (candidates.length === 0) {
    throw new Error(`No agent can handle: ${task.requiredCapability}`);
  }

  // Select best candidate (load balancing, proximity, etc.)
  const selected = selectBestAgent(candidates, task);

  // Send request and await response
  const response = await messageBus.request(selected.id, {
    type: 'task_request',
    payload: task
  });

  return response.payload as Result;
}
```

### Pattern 2: Collaborative Consensus

```typescript
// Multiple agents vote on a decision
async function reachConsensus(question: string, voters: AgentNode[]): Promise<Decision> {
  const correlationId = generateId();

  // Broadcast question to all voters
  const votePromises = voters.map(voter =>
    messageBus.request(voter.id, {
      type: 'consensus_vote',
      correlationId,
      payload: { question }
    })
  );

  const votes = await Promise.all(votePromises);

  // Aggregate votes
  const tally = new Map<string, number>();
  for (const vote of votes) {
    const choice = vote.payload.choice;
    tally.set(choice, (tally.get(choice) || 0) + 1);
  }

  // Determine winner
  const winner = [...tally.entries()]
    .sort((a, b) => b[1] - a[1])[0];

  return {
    choice: winner[0],
    confidence: winner[1] / voters.length,
    votes: votes.map(v => ({ agent: v.from, choice: v.payload.choice }))
  };
}
```

### Pattern 3: Task Auction

```typescript
// Agents bid on tasks
async function auctionTask(task: Task): Promise<AgentNode> {
  const correlationId = generateId();

  // Broadcast task availability
  await messageBus.publish('tasks', {
    type: 'task_auction',
    correlationId,
    payload: { task, deadline: Date.now() + 5000 }
  });

  // Collect bids within deadline
  const bids: Bid[] = [];

  return new Promise((resolve) => {
    const sub = messageBus.subscribe(`bids:${correlationId}`, (msg) => {
      bids.push(msg.payload as Bid);
    });

    setTimeout(() => {
      sub.unsubscribe();

      // Select winning bid
      const winner = bids.sort((a, b) => {
        // Consider: capability match, current load, past performance
        return scoreBid(b, task) - scoreBid(a, task);
      })[0];

      resolve(winner.agent);
    }, 5000);
  });
}
```

### Pattern 4: Gossip Protocol

```typescript
// Agents share state updates through gossip
class GossipProtocol {
  private state = new Map<string, StateEntry>();

  async gossip() {
    // Select random peers
    const peers = selectRandomPeers(this.registry, 3);

    for (const peer of peers) {
      // Exchange state digests
      const theirDigest = await this.requestDigest(peer);
      const myDigest = this.getDigest();

      // Find differences
      const theyNeed = this.findMissing(myDigest, theirDigest);
      const iNeed = this.findMissing(theirDigest, myDigest);

      // Exchange missing entries
      if (theyNeed.length) await this.sendUpdates(peer, theyNeed);
      if (iNeed.length) this.applyUpdates(await this.requestUpdates(peer, iNeed));
    }
  }

  // Run gossip periodically
  startGossipLoop(intervalMs: number) {
    setInterval(() => this.gossip(), intervalMs);
  }
}
```

## Agent Behavior Patterns

### Self-Organizing Task Distribution

```typescript
const agentBehavior = {
  onTaskReceived: async (task: Task) => {
    // Can I handle this?
    if (this.canHandle(task)) {
      if (this.currentLoad < this.capacity) {
        return this.handle(task);
      } else {
        // Overloaded, find help
        return this.delegateToCapablePeer(task);
      }
    }

    // Find someone who can
    const capable = await this.findCapablePeer(task);
    if (capable) {
      return this.forwardTo(capable, task);
    }

    // No one can handle
    return { error: 'No capable agent found' };
  }
};
```

### Collaborative Problem Solving

```typescript
async function collaborativeSolve(problem: Problem): Promise<Solution> {
  // Phase 1: Brainstorm
  const ideas = await broadcastAndCollect<Idea>('brainstorm', {
    problem,
    timeout: 10000
  });

  // Phase 2: Critique and refine
  const refinedIdeas = await broadcastAndCollect<Idea>('critique', {
    ideas,
    timeout: 15000
  });

  // Phase 3: Vote on best solution
  const decision = await reachConsensus(
    'Which solution should we implement?',
    this.getAllAgents()
  );

  // Phase 4: Collaborate on implementation
  return collaborativeImplement(decision.choice);
}
```

## Resilience Patterns

### Agent Failure Detection

```typescript
class HeartbeatMonitor {
  private lastSeen = new Map<string, number>();
  private TIMEOUT_MS = 30000;

  recordHeartbeat(agentId: string) {
    this.lastSeen.set(agentId, Date.now());
  }

  checkHealth(): string[] {
    const now = Date.now();
    const failed: string[] = [];

    for (const [agentId, lastTime] of this.lastSeen) {
      if (now - lastTime > this.TIMEOUT_MS) {
        failed.push(agentId);
      }
    }

    return failed;
  }

  async handleFailures(failed: string[]) {
    for (const agentId of failed) {
      // Remove from registry
      await registry.deregister(agentId);

      // Reassign pending tasks
      const pendingTasks = await getPendingTasks(agentId);
      for (const task of pendingTasks) {
        await redistributeTask(task);
      }
    }
  }
}
```

### Work Redistribution

```typescript
async function redistributeTask(task: Task) {
  // Find available capable agents
  const candidates = await registry.findByCapability(task.capability);
  const available = candidates.filter(a => a.state.load < a.state.capacity);

  if (available.length === 0) {
    // Queue for later
    await taskQueue.push(task);
    return;
  }

  // Assign to least loaded
  const target = available.sort((a, b) => a.state.load - b.state.load)[0];
  await assignTask(target, task);
}
```

## When to Use Mesh vs Supervisor

| Aspect | Mesh | Supervisor |
|--------|------|------------|
| Failure tolerance | High | Single point of failure |
| Visibility | Distributed | Centralized |
| Coordination overhead | Higher | Lower |
| Emergent behavior | Yes | Controlled |
| Scaling | Natural | Requires design |
| Debugging | Harder | Easier |

## Best Practices

1. **Define clear protocols** - Message formats, timeouts, retries
2. **Implement idempotency** - Messages may be delivered multiple times
3. **Handle partitions** - Agents may be temporarily unreachable
4. **Log everything** - Distributed debugging is hard
5. **Set boundaries** - Prevent infinite message chains
6. **Monitor gossip** - Ensure state converges
