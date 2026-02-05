---
description: Generate comprehensive test cases from requirements or user stories
---

# /write-test-cases

Generate comprehensive test cases for a feature or user story.

## What I Need

Tell me:
- Feature or user story to test
- Type of testing (functional, regression, E2E)
- Format preference (table, BDD/Gherkin, standard)
- Priority focus (smoke, full coverage)

## What I'll Create

Comprehensive test cases covering:

1. **Happy path** - Normal successful flows
2. **Negative cases** - Error handling, validation
3. **Edge cases** - Boundary values, limits
4. **Security** - Input validation, auth checks
5. **Performance** - Load considerations

## Output Formats

### Table Format
```
| ID | Title | Steps | Expected |
```

### BDD/Gherkin
```gherkin
Feature: ...
Scenario: ...
Given/When/Then
```

### Standard
```
Preconditions, Steps, Expected, Postconditions
```

## Example

```
You: /write-test-cases

Feature: User registration with email verification
Format: BDD

Claude: [Generates comprehensive Gherkin scenarios covering
happy path, validation errors, duplicate email, expired links,
resend verification, etc.]
```

## Techniques Applied

- Equivalence partitioning
- Boundary value analysis
- Decision tables
- State transitions
- Error guessing
