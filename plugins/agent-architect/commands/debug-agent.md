---
description: Debug multi-agent system issues with systematic analysis
---

# /debug-agent

Diagnose and fix issues in your multi-agent system.

## What I Need

Tell me:
- What's the unexpected behavior?
- Which agent(s) are involved?
- Do you have traces/logs?
- Is it reproducible?

## Debug Process

### Step 1: Identify Failure Point
I'll help locate:
- Which agent failed
- At what step in the workflow
- What was the input/output

### Step 2: Analyze Root Cause
Common issues:
- Prompt ambiguity
- Tool execution failures
- State corruption
- Infinite loops
- Token limit exceeded

### Step 3: Trace Analysis
If using LangSmith:
```python
from langsmith import Client
client = Client()
runs = client.list_runs(project_name="your-project", error=True)
```

### Step 4: Fix & Verify
I'll provide:
- Specific fix for the issue
- Test case to verify
- Prevention measures

## Common Issues

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| Agent loops forever | No exit condition | Add max_iterations |
| Wrong agent selected | Poor routing | Improve router prompt |
| State lost | Missing checkpointer | Add persistence |
| Timeout | Agent stuck | Add timeout + fallback |
| Inconsistent results | Non-deterministic | Set temperature=0 |

## Debug Checklist

```
[ ] Check agent prompts for ambiguity
[ ] Verify tool definitions are correct
[ ] Inspect state at each transition
[ ] Review routing logic
[ ] Check for token limits
[ ] Verify checkpointing works
[ ] Test edge cases
```
