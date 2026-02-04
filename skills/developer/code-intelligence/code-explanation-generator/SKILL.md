---
name: code-explanation-generator
description: |
  Use this skill when generating explanations for code. Activate when the user needs to understand complex code,
  wants to document how something works, needs to explain code to others, is onboarding to a new codebase,
  or wants to create educational content about code.
---

# Code Explanation Generator

Generate clear explanations for complex code to aid understanding and knowledge transfer.

## When to Use

- Understanding unfamiliar or complex code
- Creating documentation for code
- Onboarding new team members
- Code review explanations
- Educational content creation

## Explanation Levels

| Level | Audience | Focus |
|-------|----------|-------|
| Beginner | Junior devs, non-technical | What it does, simple terms |
| Intermediate | Mid-level devs | How it works, patterns used |
| Expert | Senior devs, architects | Why decisions made, trade-offs |

## AI Prompts by Level

### Beginner Explanation

```markdown
Explain this code for someone new to programming:

```[paste code]```

- Use simple, non-technical language
- Explain what the code accomplishes
- Use real-world analogies
- Avoid jargon or define terms used
```

### Intermediate Explanation

```markdown
Explain how this code works:

```[paste code]```

Include:
- Step-by-step execution flow
- Design patterns used
- Key functions and their purposes
- How components interact
```

### Expert Explanation

```markdown
Provide an architectural analysis of this code:

```[paste code]```

Cover:
- Design decisions and alternatives
- Performance characteristics
- Trade-offs made
- Potential improvements
- Edge cases and limitations
```

## Explanation Formats

### Line-by-Line Comments

**AI Prompt:**
```markdown
Add explanatory comments to each significant line of this code:

```[paste code]```

Comments should explain WHY, not WHAT (the code shows what).
```

**Output:**
```typescript
// Debounce user input to prevent excessive API calls
const debouncedSearch = useMemo(
  // Create a new debounced function only when delay changes
  () => debounce((query: string) => {
    // Skip empty queries to avoid unnecessary requests
    if (!query.trim()) return;
    // Trigger the actual search with the debounced value
    onSearch(query);
  }, delay),
  [delay, onSearch] // Recreate when these dependencies change
);
```

### Flowchart Description

**AI Prompt:**
```markdown
Describe the flow of this code in a way that could be drawn as a flowchart:

```[paste code]```

Use this format:
1. [Start] → description
2. [Decision] condition? → Yes: action / No: action
3. [Process] what happens
4. [End] final state
```

**Output:**
```
1. [Start] User submits login form
2. [Process] Extract email and password from form
3. [Decision] Is email valid format?
   → No: Display email error, return to form
   → Yes: Continue
4. [Decision] Is password >= 8 characters?
   → No: Display password error, return to form
   → Yes: Continue
5. [Process] Send credentials to auth API
6. [Decision] Did API return success?
   → No: Display error message, return to form
   → Yes: Continue
7. [Process] Store auth token in secure storage
8. [Process] Redirect to dashboard
9. [End] User is logged in
```

### Concept Map

**AI Prompt:**
```markdown
Identify the key concepts in this code and how they relate:

```[paste code]```

Format as:
- Main Concept
  - Related concept 1 (relationship)
  - Related concept 2 (relationship)
```

**Output:**
```
- AuthContext (Central state management)
  - User object (stores current user data)
  - isAuthenticated (derived from user presence)
  - login() (updates user, triggers redirect)
  - logout() (clears user, clears storage)

- AuthProvider (React context provider)
  - Wraps application (provides context to children)
  - Manages state (useState for user)
  - Handles persistence (localStorage sync)

- useAuth hook (Consumer interface)
  - Accesses context (useContext internally)
  - Provides type safety (typed return value)
```

## Specialized Explanations

### Algorithm Explanation

```markdown
Explain this algorithm:

```[paste algorithm code]```

Include:
1. What problem it solves
2. Time and space complexity
3. Step-by-step walkthrough with example
4. When to use vs alternatives
```

### React Component Explanation

```markdown
Explain this React component:

```[paste component]```

Cover:
1. Component purpose
2. Props and their uses
3. State management approach
4. Lifecycle (effects, cleanup)
5. Render logic
```

### API Endpoint Explanation

```markdown
Explain this API endpoint:

```[paste route handler]```

Include:
1. What the endpoint does
2. Request format (method, body, params)
3. Response format
4. Error cases
5. Authentication/authorization
```

### Database Query Explanation

```markdown
Explain this SQL/ORM query:

```[paste query]```

Cover:
1. What data it retrieves/modifies
2. Tables involved and relationships
3. Filtering and sorting logic
4. Performance considerations
5. Potential issues (N+1, etc.)
```

## Documentation Templates

### Function Documentation

```markdown
## `functionName(params)`

### Purpose
[What this function does and why it exists]

### Parameters
| Name | Type | Description |
|------|------|-------------|
| param1 | string | Description |
| param2 | number | Description |

### Returns
`ReturnType` - Description of return value

### Example
```javascript
const result = functionName('value', 42);
// result: { ... }
```

### Notes
- Important consideration 1
- Important consideration 2
```

### Component Documentation

```markdown
## ComponentName

### Purpose
[What this component renders and its role in the app]

### Props
| Prop | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| prop1 | string | Yes | - | Description |
| prop2 | boolean | No | false | Description |

### Usage
```jsx
<ComponentName prop1="value" prop2={true} />
```

### States
- **Loading**: Shows spinner
- **Error**: Shows error message with retry
- **Success**: Renders main content

### Accessibility
- Keyboard navigable
- Screen reader labels
- Focus management
```

## Onboarding Guides

### AI Prompt for Onboarding Doc

```markdown
Create an onboarding guide for this module:

```[paste module code]```

The guide should help a new developer:
1. Understand the module's purpose
2. Know the key files and their roles
3. Understand how to make common changes
4. Know what to test after changes
```

## Best Practices

### Good Explanations

- **Start with the "what"** - What does this accomplish?
- **Explain the "why"** - Why was it built this way?
- **Show examples** - Concrete usage scenarios
- **Note gotchas** - Common mistakes or edge cases
- **Link related code** - Where else is this used?

### Avoid

- Restating the code in English
- Over-explaining simple things
- Assuming too much context
- Using undefined acronyms
- Outdated explanations

## Verification

After generating explanations:

- [ ] Technical accuracy (code matches explanation)
- [ ] Appropriate level for audience
- [ ] No undefined jargon
- [ ] Examples are valid and runnable
- [ ] Covers edge cases and errors
