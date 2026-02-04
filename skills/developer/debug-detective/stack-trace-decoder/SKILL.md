---
name: stack-trace-decoder
description: |
  Use this skill when analyzing stack traces. Activate when the user has a stack trace to understand,
  needs to decode error traces, wants to find the root cause from a stack trace, is debugging crashes,
  or needs help interpreting exception traces.
---

# Stack Trace Decoder

Parse, understand, and extract actionable insights from stack traces.

## When to Use

- Understanding where an error originated
- Decoding minified/obfuscated traces
- Identifying the call chain leading to failure
- Comparing stack traces across incidents
- Explaining traces to team members

## Stack Trace Anatomy

### JavaScript/Node.js

```
Error: Cannot read properties of undefined (reading 'map')
    at processItems (/app/src/services/processor.js:45:12)
    at async handleRequest (/app/src/handlers/api.js:23:5)
    at async Router.handle (/app/node_modules/express/router.js:156:3)
    at async Layer.handle_request (/app/node_modules/express/layer.js:95:5)
```

**Components:**
```
Error: [Error Type]: [Error Message]
    at [Function Name] ([File Path]:[Line]:[Column])
    │        │                 │        │      │
    │        │                 │        │      └─ Column number
    │        │                 │        └─ Line number
    │        │                 └─ File path
    │        └─ Function where error occurred
    └─ "at" indicates stack frame
```

### Python

```
Traceback (most recent call last):
  File "/app/main.py", line 45, in process_data
    result = transform(data['items'])
  File "/app/utils.py", line 23, in transform
    return [item.upper() for item in items]
TypeError: 'NoneType' object is not iterable
```

**Note:** Python traces read bottom-to-top (error at bottom).

### Java

```
java.lang.NullPointerException: Cannot invoke method on null object
    at com.example.service.UserService.getProfile(UserService.java:89)
    at com.example.controller.UserController.showProfile(UserController.java:45)
    at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
    at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:897)
Caused by: java.sql.SQLException: Connection refused
    at com.mysql.jdbc.ConnectionImpl.createNewIO(ConnectionImpl.java:2181)
    ... 15 more
```

**Note:** Java traces can have "Caused by" chains showing root cause.

## Decoding Strategy

### Step 1: Find the Entry Point

The first frame (or last in Python) in YOUR code is usually most important.

```typescript
function findRelevantFrame(trace: string, appPaths: string[]): Frame | null {
  const frames = parseStackTrace(trace);

  // Skip framework/library frames
  for (const frame of frames) {
    if (appPaths.some(p => frame.file.includes(p))) {
      return frame;
    }
  }

  return frames[0]; // Fallback to first frame
}

// Example: Find first frame in 'src/' directory
const relevantFrame = findRelevantFrame(trace, ['src/', 'app/']);
```

### Step 2: Parse the Trace

```typescript
interface StackFrame {
  functionName: string;
  file: string;
  line: number;
  column?: number;
  isNative: boolean;
  isAsync: boolean;
}

function parseJavaScriptTrace(trace: string): StackFrame[] {
  const lines = trace.split('\n');
  const frames: StackFrame[] = [];

  for (const line of lines) {
    // Match: "at functionName (file:line:col)"
    const match = line.match(/at\s+(?:(async)\s+)?(\S+)\s+\((.+):(\d+):(\d+)\)/);

    if (match) {
      frames.push({
        isAsync: match[1] === 'async',
        functionName: match[2],
        file: match[3],
        line: parseInt(match[4]),
        column: parseInt(match[5]),
        isNative: match[3].includes('native') || match[3].includes('node:')
      });
    }
  }

  return frames;
}
```

### Step 3: Identify the Error Type

```typescript
const ERROR_PATTERNS = {
  // JavaScript
  'TypeError.*undefined': {
    cause: 'Accessing property on undefined value',
    fix: 'Add null check or optional chaining'
  },
  'TypeError.*not a function': {
    cause: 'Calling something that is not a function',
    fix: 'Check if the method exists and is spelled correctly'
  },
  'RangeError': {
    cause: 'Value out of allowed range (array size, recursion)',
    fix: 'Check for infinite recursion or large data'
  },
  'SyntaxError': {
    cause: 'Invalid JavaScript syntax',
    fix: 'Check for JSON parse errors or eval issues'
  },

  // Python
  'KeyError': {
    cause: 'Dictionary key does not exist',
    fix: 'Use .get() with default or check key existence'
  },
  'AttributeError.*NoneType': {
    cause: 'Calling method on None value',
    fix: 'Add None check before method call'
  },

  // Java
  'NullPointerException': {
    cause: 'Calling method on null reference',
    fix: 'Add null check or use Optional'
  },
  'ClassCastException': {
    cause: 'Invalid type cast',
    fix: 'Check object type before casting'
  }
};

function identifyErrorPattern(trace: string): ErrorInfo | null {
  for (const [pattern, info] of Object.entries(ERROR_PATTERNS)) {
    if (new RegExp(pattern, 'i').test(trace)) {
      return info;
    }
  }
  return null;
}
```

## Common Patterns

### The Null Chain

```javascript
// Error: Cannot read properties of undefined (reading 'email')
const email = user.profile.contact.email;
//            ↑      ↑       ↑
//            Any of these could be undefined
```

**Solution:**
```javascript
const email = user?.profile?.contact?.email;
// Or
const email = user && user.profile && user.profile.contact?.email;
```

### The Async Trace Gap

```javascript
Error: Something went wrong
    at processData (app.js:10)  // Where error was thrown
    // Gap - async boundary
    at async Promise.all        // No intermediate frames
```

**Solution:** Use `Error.captureStackTrace` or async stack traces:
```javascript
// Enable in Node.js
Error.stackTraceLimit = 50;
// Or use --async-stack-traces flag
```

### The Minified Trace

```javascript
Error: t is not a function
    at e.render (main.a3f2b1c.js:1:45678)
    at t.update (main.a3f2b1c.js:1:23456)
```

**Solution:** Use source maps:
```bash
# Decode with source-map-cli
source-map resolve main.a3f2b1c.js.map 1 45678
```

### The Caused-By Chain (Java)

```java
ServiceException: Failed to process user
    at UserService.process(UserService.java:50)
Caused by: DatabaseException: Query failed
    at Database.query(Database.java:120)
Caused by: SQLException: Connection refused
    at Driver.connect(Driver.java:80)
```

**Reading:** Start from the innermost "Caused by" - that's the root cause.

## Source Map Decoding

```typescript
import { SourceMapConsumer } from 'source-map';

async function decodeMinifiedTrace(
  trace: string,
  sourceMap: string
): Promise<string> {
  const consumer = await new SourceMapConsumer(JSON.parse(sourceMap));
  const frames = parseJavaScriptTrace(trace);

  const decoded = frames.map(frame => {
    if (frame.file.includes('.min.') || frame.file.includes('.bundle.')) {
      const original = consumer.originalPositionFor({
        line: frame.line,
        column: frame.column || 0
      });

      return {
        ...frame,
        file: original.source || frame.file,
        line: original.line || frame.line,
        column: original.column || frame.column,
        functionName: original.name || frame.functionName
      };
    }
    return frame;
  });

  consumer.destroy();
  return formatFrames(decoded);
}
```

## AI-Assisted Decoding

```markdown
Decode and explain this stack trace:

```
[paste stack trace]
```

Provide:
1. **Error Summary** - What went wrong in plain English
2. **Root Cause Location** - The file and line to look at
3. **Call Chain** - How the code got there
4. **Likely Cause** - What probably triggered this
5. **Suggested Fix** - How to resolve it

Also note if:
- This looks like a common pattern
- The error is in our code vs library code
- Additional context would help
```

## Trace Comparison

```typescript
function compareTraces(trace1: string, trace2: string): TraceComparison {
  const frames1 = parseStackTrace(trace1);
  const frames2 = parseStackTrace(trace2);

  // Find common frames
  const common = frames1.filter(f1 =>
    frames2.some(f2 => f1.file === f2.file && f1.line === f2.line)
  );

  // Find divergence point
  let divergeIndex = 0;
  for (let i = 0; i < Math.min(frames1.length, frames2.length); i++) {
    if (frames1[i].file !== frames2[i].file || frames1[i].line !== frames2[i].line) {
      divergeIndex = i;
      break;
    }
  }

  return {
    areSameError: common.length > frames1.length * 0.8,
    commonFrames: common,
    divergencePoint: divergeIndex,
    uniqueToFirst: frames1.filter(f => !common.includes(f)),
    uniqueToSecond: frames2.filter(f => !common.includes(f))
  };
}
```

## Best Practices

1. **Read from your code first** - Skip framework frames initially
2. **Check the error message** - Often tells you exactly what's wrong
3. **Look for "Caused by"** - The real error is often nested
4. **Enable source maps** - Minified traces are useless
5. **Increase stack limit** - Default 10 frames may not be enough
6. **Check async boundaries** - Errors can lose context
7. **Compare with working state** - What changed?
