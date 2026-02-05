---
description: Design a multi-agent system architecture for your use case
---

# /design-agent

Get expert guidance on designing multi-agent architectures.

## What I Need

Tell me:
- What problem are you trying to solve?
- How many distinct tasks/roles are involved?
- Do you need human approval at any step?
- What's your scale (requests/day)?

## Design Process

### Step 1: Analyze Requirements
I'll identify:
- Task decomposition opportunities
- Parallelization potential
- Human oversight needs
- State management requirements

### Step 2: Select Pattern
Based on your needs, I'll recommend:

| Pattern | Best For |
|---------|----------|
| Single Agent | Simple, linear tasks |
| Supervisor | Task delegation with oversight |
| Hierarchical | Complex multi-level workflows |
| Mesh/P2P | Collaborative problem-solving |
| Pipeline | Sequential processing stages |

### Step 3: Design Architecture
I'll provide:
- Agent role definitions
- Communication flows
- State management strategy
- Error handling approach

### Step 4: Implementation Plan
You'll get:
- LangGraph workflow code
- Agent prompt templates
- Checkpoint configuration
- Testing strategy

## Quick Patterns

```
Supervisor Pattern:
┌─────────────┐
│  Supervisor │
└──────┬──────┘
       │
┌──────┴──────┐
│   │   │   │
▼   ▼   ▼   ▼
A1  A2  A3  A4  (Worker Agents)

Hierarchical Pattern:
       ┌───────┐
       │ Chief │
       └───┬───┘
     ┌─────┴─────┐
     ▼           ▼
┌─────────┐ ┌─────────┐
│Manager 1│ │Manager 2│
└────┬────┘ └────┬────┘
   ┌─┴─┐      ┌──┴──┐
   ▼   ▼      ▼     ▼
  W1   W2    W3    W4
```
