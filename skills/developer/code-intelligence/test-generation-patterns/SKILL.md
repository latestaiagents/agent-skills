---
name: test-generation-patterns
description: |
  Use this skill when generating tests with AI assistance. Activate when the user wants to create unit tests,
  integration tests, generate test cases, improve test coverage, write tests for existing code, or set up
  testing patterns for their project.
---

# Test Generation Patterns

Generate comprehensive tests with AI assistance following 2026 best practices.

## When to Use

- Adding tests to existing code
- Creating test suites for new features
- Improving test coverage
- Setting up testing patterns for a project
- Generating edge case tests

## The Testing Pyramid (2026)

```
        ┌─────────┐
        │   E2E   │  Few, slow, high confidence
        ├─────────┤
        │ Integr. │  Some, medium speed
        ├─────────┤
        │  Unit   │  Many, fast, isolated
        └─────────┘
```

## Unit Test Generation

### Step 1: Identify Test Cases

For any function, test:
1. **Happy path** - Normal expected input
2. **Edge cases** - Boundaries, empty, null
3. **Error cases** - Invalid input, failures
4. **State changes** - If function has side effects

### AI Prompt Template

```markdown
Generate comprehensive unit tests for this function:

```[paste function]```

Include tests for:
1. Happy path with typical inputs
2. Edge cases (empty, null, boundary values)
3. Error handling
4. Any state changes or side effects

Use [Vitest/Jest/pytest] with [testing-library if applicable].
Follow AAA pattern (Arrange, Act, Assert).
```

### Example: Testing a Utility Function

**Code to test:**
```typescript
function parseAmount(input: string): number | null {
  if (!input || typeof input !== 'string') return null;
  const cleaned = input.replace(/[$,]/g, '').trim();
  const num = parseFloat(cleaned);
  return isNaN(num) ? null : Math.round(num * 100) / 100;
}
```

**Generated tests:**
```typescript
import { describe, it, expect } from 'vitest';
import { parseAmount } from './parseAmount';

describe('parseAmount', () => {
  describe('happy path', () => {
    it('parses plain number', () => {
      expect(parseAmount('123.45')).toBe(123.45);
    });

    it('parses number with dollar sign', () => {
      expect(parseAmount('$99.99')).toBe(99.99);
    });

    it('parses number with commas', () => {
      expect(parseAmount('1,234.56')).toBe(1234.56);
    });

    it('parses number with dollar sign and commas', () => {
      expect(parseAmount('$1,234.56')).toBe(1234.56);
    });

    it('rounds to 2 decimal places', () => {
      expect(parseAmount('123.456')).toBe(123.46);
    });
  });

  describe('edge cases', () => {
    it('handles zero', () => {
      expect(parseAmount('0')).toBe(0);
    });

    it('handles negative numbers', () => {
      expect(parseAmount('-50.00')).toBe(-50);
    });

    it('trims whitespace', () => {
      expect(parseAmount('  123.45  ')).toBe(123.45);
    });

    it('handles integer input', () => {
      expect(parseAmount('100')).toBe(100);
    });
  });

  describe('error cases', () => {
    it('returns null for empty string', () => {
      expect(parseAmount('')).toBeNull();
    });

    it('returns null for null input', () => {
      expect(parseAmount(null as any)).toBeNull();
    });

    it('returns null for undefined', () => {
      expect(parseAmount(undefined as any)).toBeNull();
    });

    it('returns null for non-string input', () => {
      expect(parseAmount(123 as any)).toBeNull();
    });

    it('returns null for non-numeric string', () => {
      expect(parseAmount('abc')).toBeNull();
    });

    it('returns null for only symbols', () => {
      expect(parseAmount('$,')).toBeNull();
    });
  });
});
```

## React Component Testing

### AI Prompt Template

```markdown
Generate tests for this React component:

```[paste component]```

Test:
1. Renders correctly with required props
2. Renders correctly with optional props
3. User interactions (clicks, inputs)
4. Loading/error states if applicable
5. Accessibility (basic a11y)

Use React Testing Library and Vitest.
Focus on user behavior, not implementation details.
```

### Example: Component Tests

**Component:**
```tsx
interface ButtonProps {
  label: string;
  onClick: () => void;
  disabled?: boolean;
  loading?: boolean;
}

export function Button({ label, onClick, disabled, loading }: ButtonProps) {
  return (
    <button
      onClick={onClick}
      disabled={disabled || loading}
      aria-busy={loading}
    >
      {loading ? 'Loading...' : label}
    </button>
  );
}
```

**Generated tests:**
```typescript
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect, vi } from 'vitest';
import { Button } from './Button';

describe('Button', () => {
  const defaultProps = {
    label: 'Click me',
    onClick: vi.fn(),
  };

  describe('rendering', () => {
    it('renders with label', () => {
      render(<Button {...defaultProps} />);
      expect(screen.getByRole('button', { name: 'Click me' })).toBeInTheDocument();
    });

    it('renders loading state', () => {
      render(<Button {...defaultProps} loading />);
      expect(screen.getByRole('button', { name: 'Loading...' })).toBeInTheDocument();
      expect(screen.getByRole('button')).toHaveAttribute('aria-busy', 'true');
    });

    it('renders disabled state', () => {
      render(<Button {...defaultProps} disabled />);
      expect(screen.getByRole('button')).toBeDisabled();
    });
  });

  describe('interactions', () => {
    it('calls onClick when clicked', async () => {
      const onClick = vi.fn();
      render(<Button {...defaultProps} onClick={onClick} />);

      await userEvent.click(screen.getByRole('button'));

      expect(onClick).toHaveBeenCalledTimes(1);
    });

    it('does not call onClick when disabled', async () => {
      const onClick = vi.fn();
      render(<Button {...defaultProps} onClick={onClick} disabled />);

      await userEvent.click(screen.getByRole('button'));

      expect(onClick).not.toHaveBeenCalled();
    });

    it('does not call onClick when loading', async () => {
      const onClick = vi.fn();
      render(<Button {...defaultProps} onClick={onClick} loading />);

      await userEvent.click(screen.getByRole('button'));

      expect(onClick).not.toHaveBeenCalled();
    });
  });

  describe('accessibility', () => {
    it('is focusable', async () => {
      render(<Button {...defaultProps} />);

      await userEvent.tab();

      expect(screen.getByRole('button')).toHaveFocus();
    });

    it('can be activated with Enter key', async () => {
      const onClick = vi.fn();
      render(<Button {...defaultProps} onClick={onClick} />);

      await userEvent.tab();
      await userEvent.keyboard('{Enter}');

      expect(onClick).toHaveBeenCalled();
    });
  });
});
```

## API/Integration Testing

### AI Prompt Template

```markdown
Generate integration tests for this API endpoint:

```[paste route handler]```

Test:
1. Successful requests with valid data
2. Validation errors (400)
3. Authentication/authorization (401/403)
4. Not found cases (404)
5. Server errors (500)

Use [supertest/axios] with proper mocking.
```

## Test Coverage Patterns

### Coverage Checklist

| Area | Target | Priority |
|------|--------|----------|
| Business logic | 90%+ | High |
| Utilities | 100% | High |
| API routes | 80%+ | High |
| Components | 70%+ | Medium |
| Integrations | 60%+ | Medium |

### Generating Tests for Coverage Gaps

```markdown
Here's my current code and test file. Identify untested paths
and generate tests to cover them:

Code:
```[paste code]```

Existing tests:
```[paste tests]```

Coverage report shows these lines uncovered: [lines]
```

## Test Data Generation

### AI Prompt for Test Fixtures

```markdown
Generate test fixtures for this type:

```typescript
interface Order {
  id: string;
  customerId: string;
  items: OrderItem[];
  status: 'pending' | 'processing' | 'shipped' | 'delivered';
  total: number;
  createdAt: Date;
}
```

Create:
1. A valid complete order
2. A minimal valid order
3. Orders in each status
4. Edge case orders (empty items, zero total, etc.)
```

## Common Testing Patterns

### Factory Pattern
```typescript
function createOrder(overrides: Partial<Order> = {}): Order {
  return {
    id: 'order-123',
    customerId: 'customer-456',
    items: [{ productId: 'prod-1', quantity: 1, price: 10 }],
    status: 'pending',
    total: 10,
    createdAt: new Date('2024-01-01'),
    ...overrides,
  };
}

// Usage in tests
const pendingOrder = createOrder();
const shippedOrder = createOrder({ status: 'shipped' });
const emptyOrder = createOrder({ items: [], total: 0 });
```

### Mock Patterns
```typescript
// Service mock
vi.mock('./userService', () => ({
  getUser: vi.fn().mockResolvedValue({ id: '1', name: 'John' }),
}));

// Spy on method
const logSpy = vi.spyOn(console, 'log').mockImplementation(() => {});

// Mock module partially
vi.mock('./utils', async () => {
  const actual = await vi.importActual('./utils');
  return {
    ...actual,
    formatDate: vi.fn().mockReturnValue('2024-01-01'),
  };
});
```

## Verification After Generation

- [ ] Tests actually test the code (not just pass)
- [ ] No hardcoded values that match implementation
- [ ] Edge cases are meaningful
- [ ] Mocks don't hide real bugs
- [ ] Tests are readable and maintainable
