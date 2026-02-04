---
description: Generate clear explanations for complex code sections
---

# /explain-code

Generate clear explanations for complex code sections.

## What to Explain

Share the code you want explained, or point me to a file.

You can ask for:
- **Overview** - What does this code do at a high level?
- **Line-by-line** - Explain each part in detail
- **Concept** - Explain the underlying pattern or algorithm
- **Why** - Explain design decisions and tradeoffs

## Explanation Levels

### Level 1: Quick Summary

One paragraph explaining what the code does:

```
This function implements a debounce mechanism that delays executing
a callback until after a specified wait time has passed since the
last invocation. It's commonly used to limit the rate of API calls
triggered by user input like typing in a search box.
```

### Level 2: Annotated Code

Comments explaining each significant line:

```typescript
function debounce<T extends (...args: any[]) => any>(
  func: T,           // The function to debounce
  wait: number       // Milliseconds to wait
): (...args: Parameters<T>) => void {
  let timeoutId: NodeJS.Timeout | null = null;  // Store timeout reference

  return (...args: Parameters<T>) => {
    // Clear any existing timeout (reset the timer)
    if (timeoutId) {
      clearTimeout(timeoutId);
    }

    // Set new timeout - func will only run if no new calls come in
    timeoutId = setTimeout(() => {
      func(...args);      // Call original function with arguments
      timeoutId = null;   // Clean up reference
    }, wait);
  };
}
```

### Level 3: Concept Deep Dive

Explain the underlying pattern:

```markdown
## Debouncing Explained

**Problem:** User types in search box, triggering API call on every keystroke.
Typing "hello" causes 5 API calls (h, he, hel, hell, hello).

**Solution:** Debouncing waits until user stops typing before making the call.
Only 1 API call for "hello" (after 300ms of no typing).

**How it works:**
1. User types 'h' → Start 300ms timer
2. User types 'e' (100ms later) → Cancel timer, start new 300ms timer
3. User types 'l' (100ms later) → Cancel timer, start new 300ms timer
4. ... continues until user stops
5. 300ms passes with no typing → Execute API call with "hello"

**Visual timeline:**
```
Keystrokes: h----e----l----l----o------------------->
Timers:     [==X] [==X] [==X] [==X] [=====EXECUTE!]
                                    └── 300ms pause
```

**Related concepts:**
- **Throttling:** Execute at most once per interval (different from debounce)
- **Leading edge:** Execute immediately, then debounce subsequent calls
- **Trailing edge:** (default) Execute after the wait period
```

### Level 4: Design Decisions

Explain why the code is written this way:

```markdown
## Why This Design?

**Q: Why use generics (`<T extends ...>`)?**
A: Preserves the original function's type signature. Without generics,
TypeScript couldn't know what arguments the debounced function accepts.

**Q: Why `NodeJS.Timeout | null` instead of just `number`?**
A: TypeScript's `setTimeout` returns different types in Node.js vs browser.
Using `NodeJS.Timeout` works in both environments.

**Q: Why not just use lodash's debounce?**
A: For a utility this simple, avoiding a dependency reduces bundle size.
Lodash debounce is ~1KB, this is ~100 bytes.

**Q: What are the tradeoffs?**
- ✅ Simple, easy to understand
- ✅ No dependencies
- ❌ No cancel() method
- ❌ No leading edge option
- ❌ No maxWait option

For complex needs, consider lodash or custom implementation with more features.
```

## What I Can Explain

| Code Type | What I'll Cover |
|-----------|-----------------|
| **Functions** | Purpose, parameters, return value, side effects |
| **Classes** | Responsibilities, methods, inheritance |
| **Algorithms** | Time/space complexity, step-by-step walkthrough |
| **Regex** | Pattern breakdown, what matches/doesn't match |
| **SQL queries** | What data is selected, joins explained |
| **Config files** | What each option does |
| **Type definitions** | What the types represent |

## Example Questions

- "Explain this regex: `/^(?=.*[A-Z])(?=.*[0-9]).{8,}$/`"
- "What does this SQL query do?"
- "Walk me through this sorting algorithm"
- "Why is this useEffect causing infinite loops?"
- "Explain the dependency injection pattern in this code"

---

**Share the code and tell me what level of explanation you want.**
