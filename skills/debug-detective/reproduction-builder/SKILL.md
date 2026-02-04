---
name: reproduction-builder
description: |
  Use this skill when creating bug reproductions. Activate when the user needs to create a minimal
  reproduction case, report a bug with steps to reproduce, isolate a bug to specific conditions,
  or help others understand how to trigger an issue.
---

# Reproduction Builder

Create minimal, reliable reproduction cases for bugs.

## When to Use

- Reporting bugs to maintainers
- Isolating intermittent issues
- Documenting bugs for your team
- Creating test cases from bugs
- Helping others understand an issue

## The Minimal Reproduction

A minimal reproduction (repro) is:
- **Minimal** - Remove everything not needed to trigger the bug
- **Complete** - Contains everything needed to see the bug
- **Verifiable** - Others can run it and see the same issue
- **Isolated** - No external dependencies if possible

## Building a Repro

### Step 1: Identify the Symptom

```markdown
## Bug Symptom

**What I expected:**
When I click "Submit", the form should save and show a success message.

**What actually happened:**
The page refreshes but nothing is saved. No error in UI.

**Error in console (if any):**
```
TypeError: Cannot read properties of undefined (reading 'id')
    at saveForm (form.js:45)
```
```

### Step 2: Find the Trigger

Ask these questions:
- What action causes the bug?
- Does it happen every time?
- What conditions are required?

```markdown
## Trigger Conditions

**Always happens when:**
- [ ] Form field "email" is empty
- [ ] User is not logged in
- [ ] Data was previously saved

**Sometimes happens when:**
- [ ] Network is slow
- [ ] Multiple tabs open
- [ ] After being idle for a while

**Never happens when:**
- [ ] All fields filled
- [ ] Fresh browser session
```

### Step 3: Isolate Variables

Remove things one at a time until you find the minimum needed.

```typescript
// Original code with bug
async function saveForm(data: FormData) {
  const validated = await validateForm(data);
  const enriched = await enrichWithUserData(validated);
  const formatted = formatForAPI(enriched);
  const response = await api.post('/forms', formatted);
  await updateLocalCache(response);
  showSuccessMessage();
  analytics.track('form_saved');
  return response;
}

// Isolated - which step fails?
async function saveFormDebug(data: FormData) {
  console.log('1. Input:', data);

  const validated = await validateForm(data);
  console.log('2. Validated:', validated);

  const enriched = await enrichWithUserData(validated);
  console.log('3. Enriched:', enriched);  // Bug here: enriched.user is undefined

  const formatted = formatForAPI(enriched);
  console.log('4. Formatted:', formatted);

  // ... rest
}
```

### Step 4: Create Standalone Repro

```typescript
// Minimal reproduction
// Run with: npx ts-node repro.ts

interface User {
  id: string;
  name: string;
}

interface FormData {
  user?: User;
  email: string;
}

function formatForAPI(data: FormData) {
  // BUG: Crashes when user is undefined
  return {
    userId: data.user.id,  // <- TypeError here
    email: data.email
  };
}

// Reproduction case
const buggyData: FormData = {
  email: 'test@example.com'
  // user is undefined
};

formatForAPI(buggyData);  // Throws!
```

## Repro Templates

### For JavaScript/TypeScript

```markdown
## Bug Report

**Environment:**
- Node.js: v20.x
- Package: example-lib@2.3.4
- OS: macOS 14

**Minimal Reproduction:**

```javascript
// Save as repro.js and run: node repro.js
const { buggyFunction } = require('example-lib');

// This should work but throws
const result = buggyFunction({
  input: 'valid input',
  options: { flag: true }
});

console.log(result);
```

**Expected:** Returns processed result
**Actual:** Throws TypeError

**Steps:**
1. npm init -y
2. npm install example-lib@2.3.4
3. Create repro.js with above code
4. Run: node repro.js
5. See error
```

### For React

```jsx
// CodeSandbox/StackBlitz link preferred
// Or minimal component:

import { useState } from 'react';
import { BuggyComponent } from 'library';

export default function Repro() {
  const [data, setData] = useState(null);

  // Bug: Component crashes when data is null
  return (
    <div>
      <button onClick={() => setData({ value: 1 })}>
        Load Data
      </button>
      {/* Crashes before clicking button */}
      <BuggyComponent data={data} />
    </div>
  );
}
```

### For API Issues

```bash
# Minimal curl reproduction
curl -X POST https://api.example.com/endpoint \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "field": "value",
    "nested": {
      "trigger": true
    }
  }'

# Expected: 200 OK with response
# Actual: 500 Internal Server Error
```

### For Database Issues

```sql
-- Reproduction schema
CREATE TABLE repro_users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255),
  created_at TIMESTAMP DEFAULT NOW()
);

-- Sample data that triggers bug
INSERT INTO repro_users (email) VALUES
  ('valid@example.com'),
  (NULL),  -- This row triggers the bug
  ('another@example.com');

-- Query that fails
SELECT * FROM repro_users
WHERE email LIKE '%@%';  -- Fails on NULL row
```

## Repro Quality Checklist

### Essential
- [ ] Can be run by copying and pasting
- [ ] Includes all necessary dependencies/versions
- [ ] Shows exact error message
- [ ] Describes expected vs actual behavior

### Good to Have
- [ ] Hosted on CodeSandbox/StackBlitz/GitHub
- [ ] Includes sample data
- [ ] Notes which line triggers the bug
- [ ] Lists what you've already tried

### Avoid
- [ ] Private/proprietary code
- [ ] Real credentials or PII
- [ ] Unnecessary dependencies
- [ ] External service dependencies

## Handling Intermittent Bugs

For bugs that don't happen every time:

```typescript
// Stress test reproduction
async function reproduceRaceCondition() {
  let failures = 0;
  const attempts = 1000;

  for (let i = 0; i < attempts; i++) {
    try {
      await buggyAsyncOperation();
    } catch (error) {
      failures++;
      console.log(`Failed on attempt ${i}:`, error.message);
    }
  }

  console.log(`Failed ${failures}/${attempts} times (${(failures/attempts*100).toFixed(1)}%)`);
}

reproduceRaceCondition();
```

## Converting Repro to Test

```typescript
// From reproduction
function formatForAPI(data: FormData) {
  return { userId: data.user.id };  // Bug
}
const buggyData = { email: 'test@example.com' };
formatForAPI(buggyData);  // Throws

// To test
describe('formatForAPI', () => {
  it('handles missing user gracefully', () => {
    const data = { email: 'test@example.com' };

    // Current behavior (bug)
    expect(() => formatForAPI(data)).toThrow();

    // After fix, should be:
    // expect(formatForAPI(data)).toEqual({ userId: null, email: 'test@example.com' });
  });
});
```

## AI-Assisted Repro Building

```markdown
Help me create a minimal reproduction for this bug:

**Symptoms:**
[Describe what's happening]

**Error:**
```
[Paste error message/stack trace]
```

**Code involved:**
```
[Paste relevant code]
```

Please:
1. Identify the minimum code needed
2. Create a standalone repro file
3. Suggest what data triggers the bug
4. Explain why the bug occurs
```

## Best Practices

1. **Start minimal** - Don't copy entire files
2. **Use online playgrounds** - Easy for others to run
3. **Include versions** - Bugs are often version-specific
4. **Test your repro** - Make sure it actually fails
5. **Remove red herrings** - Extra code confuses
6. **Document workarounds** - If you found one
