---
name: debug-with-ai
description: |
  Use this skill when debugging code with AI assistance. Activate when the user has a bug, error,
  unexpected behavior, needs to understand why code isn't working, wants to analyze stack traces,
  or is troubleshooting issues in their application.
---

# Debug with AI

Structured debugging workflows using AI as your debugging partner.

## When to Use

- Code throws an error you don't understand
- Feature works sometimes but not always
- Unexpected behavior that's hard to trace
- Performance issues or memory leaks
- Complex stack traces to decipher

## The Debugging Framework

```
┌───────────────────────────────────────────┐
│ 1. REPRODUCE - Can you trigger it?        │
├───────────────────────────────────────────┤
│ 2. ISOLATE - Where exactly is the issue?  │
├───────────────────────────────────────────┤
│ 3. UNDERSTAND - Why does it happen?       │
├───────────────────────────────────────────┤
│ 4. FIX - Make minimal change to resolve   │
├───────────────────────────────────────────┤
│ 5. VERIFY - Confirm fix, add test         │
└───────────────────────────────────────────┘
```

## Step 1: Reproduce

Before asking AI for help, gather information:

### Collect This Information
```markdown
## Bug Report

**What I expected:**
[Expected behavior]

**What actually happened:**
[Actual behavior]

**Error message (if any):**
```
[Full error message and stack trace]
```

**Steps to reproduce:**
1. [Step 1]
2. [Step 2]
3. [Step 3]

**Environment:**
- Node/Browser version:
- OS:
- Relevant packages:

**Code involved:**
```[paste relevant code]```

**What I've already tried:**
- [Attempt 1]
- [Attempt 2]
```

## Step 2: Isolate

### Binary Search for Bugs

If you don't know where the bug is:

1. **Comment out half the code**
2. **Does bug still occur?**
   - Yes → Bug is in remaining code
   - No → Bug is in commented code
3. **Repeat with that half**

### AI Prompt for Isolation

```markdown
I have a bug somewhere in this code flow:

1. Function A calls Function B
2. Function B processes data and calls Function C
3. Function C returns result

The bug manifests as [description].

Here's the code:
```[paste relevant functions]```

Help me identify which function is most likely causing the issue
and what I should check first.
```

## Step 3: Understand with AI

### For Error Messages

```markdown
Explain this error and likely causes:

```
TypeError: Cannot read properties of undefined (reading 'map')
    at processItems (processor.js:45:12)
    at async handleRequest (handler.js:23:5)
```

Code at processor.js:45:
```[paste surrounding code]```

What are the most likely causes and how do I fix them?
```

### For Unexpected Behavior

```markdown
This code doesn't behave as expected:

```[paste code]```

**Expected:** When input is X, output should be Y
**Actual:** Output is Z

Walk me through the code execution step by step
and identify where the logic diverges from my expectation.
```

### For Intermittent Bugs

```markdown
This bug only happens sometimes:

```[paste code]```

**Symptoms:** [description]
**Frequency:** About 1 in 10 times
**Conditions:** Seems more common when [observation]

What could cause intermittent failures? Consider:
- Race conditions
- Timing issues
- External dependencies
- State management
- Caching
```

## Step 4: Fix

### AI Prompt for Fix

```markdown
I've identified the bug:

**Problem:** [description of root cause]

**Current code:**
```[paste buggy code]```

**Context/constraints:**
- [Any constraints on the fix]
- [Backward compatibility needs]

Provide a minimal fix that:
1. Solves the immediate problem
2. Handles edge cases
3. Doesn't introduce new issues
```

### Fix Quality Checklist

- [ ] Fixes root cause, not just symptom
- [ ] Handles related edge cases
- [ ] Doesn't break existing functionality
- [ ] Is the minimal necessary change
- [ ] Is clear and maintainable

## Step 5: Verify

### Create Regression Test

```markdown
Generate a test that would have caught this bug:

**Bug:** [description]
**Root cause:** [what was wrong]
**Fix:** [how it was fixed]

**Fixed code:**
```[paste fixed code]```

Create a test that:
1. Fails with the old buggy code
2. Passes with the fix
3. Covers the edge case that caused this
```

## Common Bug Patterns

### Null/Undefined Errors

**Symptom:** `Cannot read property 'x' of undefined`

**AI Prompt:**
```markdown
This code throws "Cannot read property 'items' of undefined":

```javascript
function processOrder(order) {
  return order.items.map(item => item.name);
}
```

Identify all paths where `order` or `order.items` could be
undefined and provide defensive code.
```

### Async/Await Issues

**Symptom:** Function returns Promise instead of value, or undefined

**AI Prompt:**
```markdown
This async code doesn't work as expected:

```javascript
async function getData() {
  const items = fetchItems(); // forgot await
  return items.filter(i => i.active);
}
```

Error: items.filter is not a function

Find async/await issues and fix them.
```

### Race Conditions

**Symptom:** Works sometimes, fails other times

**AI Prompt:**
```markdown
This code has a race condition:

```javascript
let cache = null;

async function getData() {
  if (!cache) {
    cache = await fetchData(); // Multiple calls can start fetch
  }
  return cache;
}
```

Identify the race condition and provide a thread-safe solution.
```

### State Management Bugs

**Symptom:** UI shows stale data, updates don't reflect

**AI Prompt:**
```markdown
React component doesn't update when it should:

```jsx
function Counter() {
  const [counts, setCounts] = useState({a: 0, b: 0});

  const increment = (key) => {
    counts[key]++; // Mutating state directly!
    setCounts(counts);
  };
}
```

Identify the state management issue and fix it.
```

## Debugging Tools Integration

### Console Debugging
```javascript
// Strategic logging
console.log('Function entry:', { input, state });
console.log('After processing:', { result });
console.trace('Call stack'); // Show call stack
console.table(arrayOfObjects); // Tabular display
```

### Browser DevTools
```javascript
debugger; // Pause execution here

// Conditional breakpoint (in DevTools)
// Right-click line → Add conditional breakpoint
// Condition: user.id === 'problematic-id'
```

### Performance Debugging
```javascript
console.time('operation');
// ... code to measure
console.timeEnd('operation');

// Or use Performance API
const start = performance.now();
// ... code
console.log(`Took ${performance.now() - start}ms`);
```

## When to Ask AI for Help

**Good candidates for AI debugging:**
- Understanding error messages
- Explaining complex stack traces
- Identifying patterns in intermittent bugs
- Reviewing fix approaches
- Generating regression tests

**Better to debug manually:**
- UI layout issues (AI can't see your screen)
- Environment-specific issues
- Real-time performance profiling
- Memory leak investigation (needs tools)
