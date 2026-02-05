---
name: test-case-design
description: |
  Design comprehensive test cases with proper coverage and clear documentation.
  Use this skill when writing test cases, creating test suites, or improving test coverage.
  Activate when: test case, test scenario, test coverage, test design, write tests, BDD, gherkin.
---

# Test Case Design

**Write comprehensive, maintainable test cases that catch bugs early.**

## When to Use

- Writing test cases for new features
- Improving test coverage
- Creating regression test suites
- Documenting test scenarios
- Training new QA team members

## Test Case Structure

### Standard Format

```markdown
## Test Case: TC-[ID]

**Title:** [Clear, action-oriented title]

**Priority:** P0 / P1 / P2 / P3
**Type:** Functional / Regression / Smoke / Edge Case

### Preconditions
- [System state required before test]
- [User state/data required]

### Test Data
- [Specific data needed]

### Steps
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | [Do this] | [See this] |
| 2 | [Do that] | [Observe that] |
| 3 | [Verify] | [Confirm] |

### Postconditions
- [Expected system state after test]

### Notes
- [Edge cases, known issues, etc.]
```

### Example Test Case

```markdown
## Test Case: TC-LOGIN-001

**Title:** Successful login with valid credentials

**Priority:** P0
**Type:** Smoke

### Preconditions
- User account exists with email: test@example.com
- User is not currently logged in
- Application is accessible

### Test Data
- Email: test@example.com
- Password: ValidPassword123!

### Steps
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Navigate to /login | Login page displays with email and password fields |
| 2 | Enter email "test@example.com" | Email field populated |
| 3 | Enter password "ValidPassword123!" | Password field shows masked input |
| 4 | Click "Sign In" button | Loading indicator appears |
| 5 | Wait for redirect | User redirected to /dashboard |

### Postconditions
- User is logged in
- Session cookie is set
- Welcome message shows username

### Notes
- Also verify with uppercase email variant
- Test "Remember me" checkbox separately
```

## BDD/Gherkin Format

### Feature File

```gherkin
# features/login.feature
Feature: User Login
  As a registered user
  I want to log into my account
  So that I can access my personalized content

  Background:
    Given I am on the login page
    And I have a valid account

  @smoke @p0
  Scenario: Successful login with valid credentials
    When I enter my email "user@example.com"
    And I enter my password "ValidPassword123"
    And I click the "Sign In" button
    Then I should be redirected to the dashboard
    And I should see "Welcome back" message

  @p1
  Scenario: Login fails with incorrect password
    When I enter my email "user@example.com"
    And I enter my password "WrongPassword"
    And I click the "Sign In" button
    Then I should see an error message "Invalid credentials"
    And I should remain on the login page

  @p1
  Scenario Outline: Login validation errors
    When I enter my email "<email>"
    And I enter my password "<password>"
    And I click the "Sign In" button
    Then I should see an error message "<error>"

    Examples:
      | email           | password | error                    |
      |                 | pass123  | Email is required        |
      | invalid-email   | pass123  | Invalid email format     |
      | user@test.com   |          | Password is required     |
      | user@test.com   | short    | Password too short       |

  @p2
  Scenario: Account locked after multiple failed attempts
    Given I have failed login 4 times
    When I enter my email "user@example.com"
    And I enter my password "WrongPassword"
    And I click the "Sign In" button
    Then I should see "Account locked" message
    And I should see "Try again in 15 minutes"
```

## Test Design Techniques

### Equivalence Partitioning

```markdown
## Feature: Age Verification

### Valid Partitions:
- 18-25 (young adult)
- 26-64 (adult)
- 65+ (senior)

### Invalid Partitions:
- < 0 (negative)
- 0-17 (minor)
- > 150 (unrealistic)

### Test Cases:
| Partition | Input | Expected |
|-----------|-------|----------|
| Valid young | 21 | Accepted |
| Valid adult | 40 | Accepted |
| Valid senior | 70 | Accepted |
| Invalid minor | 15 | Rejected |
| Invalid negative | -5 | Error |
| Boundary min | 18 | Accepted |
| Boundary max | 17 | Rejected |
```

### Boundary Value Analysis

```markdown
## Feature: Quantity Input (1-100)

### Boundaries:
| Value | Type | Expected |
|-------|------|----------|
| 0 | Below min | Error |
| 1 | Min boundary | Valid |
| 2 | Above min | Valid |
| 50 | Nominal | Valid |
| 99 | Below max | Valid |
| 100 | Max boundary | Valid |
| 101 | Above max | Error |
```

### Decision Table

```markdown
## Feature: Shipping Cost Calculator

### Conditions:
- C1: Order > $50
- C2: Member
- C3: Express shipping

### Actions:
- A1: Free shipping
- A2: Standard rate
- A3: Express rate
- A4: Member discount

| Rule | C1 | C2 | C3 | Actions |
|------|----|----|----|----|
| R1 | Y | Y | N | A1 |
| R2 | Y | Y | Y | A3, A4 |
| R3 | Y | N | N | A1 |
| R4 | Y | N | Y | A3 |
| R5 | N | Y | N | A2, A4 |
| R6 | N | Y | Y | A3, A4 |
| R7 | N | N | N | A2 |
| R8 | N | N | Y | A3 |
```

### State Transition

```markdown
## Feature: Order Status

### States:
- Created → Pending Payment
- Pending Payment → Paid / Cancelled
- Paid → Processing
- Processing → Shipped / Cancelled
- Shipped → Delivered / Returned
- Delivered → (end)
- Cancelled → (end)
- Returned → Refunded
- Refunded → (end)

### Test Cases:
1. Happy path: Created → Paid → Processing → Shipped → Delivered
2. Cancellation: Created → Pending → Cancelled
3. Return flow: Delivered → Returned → Refunded
4. Invalid: Shipped → Pending (should fail)
```

## Test Coverage Checklist

```markdown
## Feature: [Feature Name]

### Functional
- [ ] Happy path scenarios
- [ ] Alternative flows
- [ ] Error handling
- [ ] Validation messages
- [ ] Default values

### Data
- [ ] Valid data combinations
- [ ] Invalid data handling
- [ ] Boundary values
- [ ] Empty/null values
- [ ] Special characters
- [ ] Max length inputs

### UI/UX
- [ ] Layout and styling
- [ ] Responsive design
- [ ] Loading states
- [ ] Error states
- [ ] Empty states
- [ ] Accessibility

### Integration
- [ ] API calls
- [ ] Database changes
- [ ] External services
- [ ] Event triggers

### Security
- [ ] Authentication required
- [ ] Authorization checked
- [ ] Input sanitization
- [ ] Session handling

### Performance
- [ ] Load time acceptable
- [ ] No memory leaks
- [ ] Handles concurrent users
```

## Test Suite Organization

```markdown
## Test Suite Structure

tests/
├── smoke/                 # Critical path tests (~10 min)
│   ├── login.spec.ts
│   ├── checkout.spec.ts
│   └── search.spec.ts
├── regression/            # Full regression (~2 hours)
│   ├── users/
│   ├── orders/
│   └── products/
├── integration/           # API/service tests
│   ├── api/
│   └── webhooks/
└── e2e/                   # End-to-end workflows
    ├── purchase-flow.spec.ts
    └── onboarding.spec.ts
```

## Best Practices

1. **One assertion focus** - Test one thing per test case
2. **Independent tests** - No dependencies between tests
3. **Clear naming** - Name describes expected behavior
4. **Maintainable** - Easy to update when requirements change
5. **Traceable** - Link to requirements/user stories
6. **Prioritized** - Know which tests matter most
7. **Reviewed** - Peer review test cases like code
