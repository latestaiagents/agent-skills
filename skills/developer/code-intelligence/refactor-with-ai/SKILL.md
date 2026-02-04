---
name: refactor-with-ai
description: |
  Use this skill when refactoring code with AI assistance. Activate when the user wants to improve code
  structure, extract functions, reduce complexity, modernize legacy code, apply design patterns,
  clean up technical debt, or restructure code while preserving behavior.
---

# Refactor with AI

Safely refactor code with AI assistance while preserving behavior.

## When to Use

- Cleaning up messy or complex code
- Extracting reusable functions/components
- Modernizing legacy patterns
- Reducing cyclomatic complexity
- Applying design patterns
- Preparing code for new features

## The Safe Refactoring Process

```
┌─────────────────────────────────────────┐
│ 1. CHARACTERIZE - Understand current    │
│    behavior with tests                  │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ 2. IDENTIFY - Find specific smells      │
│    and improvement opportunities        │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ 3. REFACTOR - Small, incremental        │
│    changes with AI assistance           │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ 4. VERIFY - Tests still pass,           │
│    behavior unchanged                   │
└─────────────────────────────────────────┘
```

## Step 1: Characterize with Tests

Before ANY refactoring, ensure behavior is captured.

```typescript
// If no tests exist, create characterization tests
describe('OrderProcessor (characterization)', () => {
  it('processes valid order', () => {
    const result = processor.process(validOrder);
    // Capture current behavior, even if not ideal
    expect(result).toMatchSnapshot();
  });

  it('handles edge cases', () => {
    expect(processor.process(null)).toBeNull();
    expect(processor.process(emptyOrder)).toEqual({ status: 'empty' });
  });
});
```

## Step 2: Identify Code Smells

### Common Smells to Look For

| Smell | Sign | Refactoring |
|-------|------|-------------|
| Long Method | >20 lines | Extract Method |
| Long Parameter List | >3 params | Introduce Parameter Object |
| Duplicate Code | Copy-paste | Extract and reuse |
| Nested Conditionals | >3 levels deep | Guard clauses, early return |
| Feature Envy | Uses other class's data | Move method |
| Data Clumps | Same params together | Create class/type |
| Primitive Obsession | Strings for everything | Value objects |
| Switch Statements | Type checking | Polymorphism |

### AI Prompt for Identification

```markdown
Analyze this code for refactoring opportunities. List:
1. Code smells present
2. Complexity hotspots
3. Potential extractions
4. Modernization opportunities

```[paste code]```

Focus on actionable improvements, not stylistic preferences.
```

## Step 3: Refactor Incrementally

### Pattern: Extract Function

**Before:**
```typescript
function processOrder(order: Order) {
  // 50 lines of validation
  if (!order.items) return { error: 'No items' };
  if (order.items.length === 0) return { error: 'Empty order' };
  if (!order.customer) return { error: 'No customer' };
  // ... more validation

  // 30 lines of calculation
  let total = 0;
  for (const item of order.items) {
    total += item.price * item.quantity;
    if (item.discount) {
      total -= item.discount;
    }
  }
  // ... more calculation

  // 20 lines of persistence
  // ...
}
```

**AI Prompt:**
```markdown
Extract the validation logic from this function into a separate
`validateOrder` function. Keep the same behavior.

```[paste function]```
```

**After:**
```typescript
function validateOrder(order: Order): ValidationResult {
  if (!order.items) return { valid: false, error: 'No items' };
  if (order.items.length === 0) return { valid: false, error: 'Empty order' };
  if (!order.customer) return { valid: false, error: 'No customer' };
  return { valid: true };
}

function calculateTotal(items: OrderItem[]): number {
  return items.reduce((total, item) => {
    const itemTotal = item.price * item.quantity;
    return total + itemTotal - (item.discount || 0);
  }, 0);
}

function processOrder(order: Order) {
  const validation = validateOrder(order);
  if (!validation.valid) return { error: validation.error };

  const total = calculateTotal(order.items);
  // persistence...
}
```

### Pattern: Replace Conditionals with Guard Clauses

**Before:**
```typescript
function getPaymentStatus(payment: Payment) {
  if (payment) {
    if (payment.status) {
      if (payment.status === 'completed') {
        if (payment.amount > 0) {
          return 'success';
        } else {
          return 'zero-amount';
        }
      } else {
        return payment.status;
      }
    } else {
      return 'unknown';
    }
  } else {
    return 'no-payment';
  }
}
```

**AI Prompt:**
```markdown
Refactor this function using guard clauses and early returns
to reduce nesting. Keep exact same behavior.

```[paste function]```
```

**After:**
```typescript
function getPaymentStatus(payment: Payment) {
  if (!payment) return 'no-payment';
  if (!payment.status) return 'unknown';
  if (payment.status !== 'completed') return payment.status;
  if (payment.amount <= 0) return 'zero-amount';
  return 'success';
}
```

### Pattern: Introduce Parameter Object

**Before:**
```typescript
function createUser(
  firstName: string,
  lastName: string,
  email: string,
  phone: string,
  address: string,
  city: string,
  country: string,
  role: string
) {
  // ...
}
```

**AI Prompt:**
```markdown
Refactor this function to use a parameter object instead of
individual parameters. Create appropriate TypeScript interface.

```[paste function]```
```

**After:**
```typescript
interface CreateUserParams {
  name: {
    first: string;
    last: string;
  };
  contact: {
    email: string;
    phone: string;
  };
  address: {
    street: string;
    city: string;
    country: string;
  };
  role: string;
}

function createUser(params: CreateUserParams) {
  // ...
}
```

## Step 4: Verify Behavior

After each refactoring step:

```bash
# Run existing tests
npm test

# Run type checking
npx tsc --noEmit

# Run linting
npm run lint

# If available, run mutation testing
npx stryker run
```

### Verification Checklist

- [ ] All existing tests pass
- [ ] No new TypeScript errors
- [ ] No new linting errors
- [ ] Manual smoke test of affected feature
- [ ] Performance not degraded (if applicable)

## Refactoring Prompts Library

### Simplify Complex Logic
```markdown
Simplify this code while preserving exact behavior.
Focus on readability and reducing complexity.
```

### Modernize Patterns
```markdown
Modernize this code to use current JavaScript/TypeScript patterns:
- async/await instead of callbacks/promises
- Optional chaining and nullish coalescing
- Array methods instead of loops where appropriate

Preserve all existing behavior.
```

### Extract Component (React)
```markdown
Extract the [specific part] into a separate reusable component.
- Keep the same props interface visible
- Maintain all existing behavior
- Follow the project's component patterns
```

### Apply Design Pattern
```markdown
Refactor this code to use the [Strategy/Factory/Observer] pattern.
Current code: ```[paste]```
Goal: Make it easier to [add new types/extend behavior/etc]
```

## Common Mistakes to Avoid

1. **Refactoring without tests** - You can't verify behavior preservation
2. **Too many changes at once** - Small steps, verify each
3. **Changing behavior "while we're at it"** - Separate commits
4. **Not running tests after each step** - Bugs compound
5. **Over-engineering** - Extract only what's needed now

## When NOT to Refactor

- Code works and rarely changes
- No tests and can't add them
- Under time pressure for unrelated feature
- Major rewrite is actually needed
