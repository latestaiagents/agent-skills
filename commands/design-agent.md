---
description: Design a multi-agent system architecture with proven patterns
---

# /design-agent

Design a multi-agent system architecture with proven patterns.

## Tell Me About Your System

To recommend the right architecture, I need to know:

1. **What problem are you solving?**
   - Customer support automation
   - Code generation pipeline
   - Research and analysis
   - Content creation workflow
   - Data processing pipeline

2. **How many agents do you need?**
   - 2-3 agents (simple delegation)
   - 4-7 agents (specialized team)
   - 8+ agents (complex swarm)

3. **How should they coordinate?**
   - One agent controls others (Supervisor)
   - Agents talk to each other (Mesh)
   - Strict pipeline (Sequential)
   - Dynamic based on task (Adaptive)

## Architecture Patterns

### Pattern 1: Supervisor (Hub-and-Spoke)

Best for: Clear hierarchy, centralized control, audit requirements

```
         ┌─────────────┐
         │  Supervisor │  ← Makes decisions, delegates
         └──────┬──────┘
        ┌───────┼───────┐
        ▼       ▼       ▼
    ┌───────┐┌───────┐┌───────┐
    │Agent A││Agent B││Agent C│  ← Specialized workers
    └───────┘└───────┘└───────┘
```

**Use when:**
- You need clear accountability
- Tasks can be cleanly separated
- You want predictable behavior

### Pattern 2: Mesh (Peer-to-Peer)

Best for: Complex tasks, emergent behavior, resilience

```
    ┌───────┐     ┌───────┐
    │Agent A│◄───►│Agent B│
    └───┬───┘     └───┬───┘
        │     ╲   ╱   │
        │      ╲ ╱    │
        ▼       ╳     ▼
    ┌───────┐  ╱ ╲┌───────┐
    │Agent C│◄───►│Agent D│
    └───────┘     └───────┘
```

**Use when:**
- Tasks require collaboration
- Agents have overlapping knowledge
- System needs fault tolerance

### Pattern 3: Pipeline (Sequential)

Best for: Ordered workflows, data transformation, quality gates

```
    Input → [Agent A] → [Agent B] → [Agent C] → Output
              │            │            │
              ▼            ▼            ▼
           Validate     Process      Format
```

**Use when:**
- Tasks have natural order
- Each stage transforms data
- You need checkpoints

### Pattern 4: Router (Dynamic)

Best for: Variable tasks, cost optimization, load balancing

```
                ┌─────────┐
    Request ───►│ Router  │
                └────┬────┘
           ┌────────┼────────┐
           ▼        ▼        ▼
       ┌───────┐┌───────┐┌───────┐
       │Simple ││Medium ││Complex│
       │ Agent ││ Agent ││ Agent │
       └───────┘└───────┘└───────┘
         Haiku   Sonnet    Opus
```

**Use when:**
- Tasks vary in complexity
- You want to optimize costs
- Different skills needed for different requests

## Implementation Template

```typescript
interface Agent {
  name: string;
  role: string;
  model: string;
  systemPrompt: string;
  tools: Tool[];
}

interface AgentSystem {
  agents: Agent[];
  topology: 'supervisor' | 'mesh' | 'pipeline' | 'router';
  coordinator: Coordinator;
  memory: SharedMemory;
}

// Example: Code Review System
const codeReviewSystem: AgentSystem = {
  topology: 'pipeline',
  agents: [
    {
      name: 'syntax-checker',
      role: 'Validate code syntax and formatting',
      model: 'claude-3-haiku',  // Fast, cheap
      tools: ['lint', 'format']
    },
    {
      name: 'security-reviewer',
      role: 'Check for security vulnerabilities',
      model: 'claude-3.5-sonnet',  // Balanced
      tools: ['security-scan', 'dependency-check']
    },
    {
      name: 'logic-reviewer',
      role: 'Review business logic and architecture',
      model: 'claude-3-opus',  // Deep reasoning
      tools: ['codebase-search', 'test-runner']
    }
  ]
};
```

## Key Decisions I'll Help You Make

1. **Agent boundaries** - What does each agent own?
2. **Communication protocol** - How do agents share information?
3. **Memory architecture** - What state is shared vs private?
4. **Error handling** - What happens when an agent fails?
5. **Cost budget** - How to allocate tokens across agents?

---

**Describe your use case and I'll design a complete architecture.**
