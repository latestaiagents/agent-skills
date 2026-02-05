---
name: test-planning
description: |
  Create comprehensive test strategies and test plans for software projects.
  Use this skill when planning testing for new features, releases, or projects.
  Activate when: test plan, test strategy, testing approach, QA planning, test coverage, release testing.
---

# Test Planning

**Create comprehensive test strategies that ensure quality and coverage.**

## When to Use

- Planning testing for a new feature or release
- Creating a test strategy document
- Defining test coverage requirements
- Estimating testing effort
- Setting up QA processes for a new project

## Test Strategy Document Template

```markdown
# Test Strategy: [Project/Feature Name]

**Version:** 1.0
**Author:** [Name]
**Date:** [Date]
**Status:** Draft / In Review / Approved

## 1. Introduction

### 1.1 Purpose
[Brief description of what this test strategy covers]

### 1.2 Scope
**In Scope:**
- [Feature/component 1]
- [Feature/component 2]

**Out of Scope:**
- [Explicitly excluded items]

### 1.3 References
- PRD: [link]
- Technical Design: [link]
- API Specs: [link]

## 2. Test Objectives

- Verify [functional requirement 1]
- Validate [performance requirement]
- Ensure [security requirement]
- Confirm [compatibility requirement]

## 3. Test Approach

### 3.1 Test Levels

| Level | Scope | Owner | Tools |
|-------|-------|-------|-------|
| Unit | Individual functions | Dev | Jest, pytest |
| Integration | API contracts | Dev/QA | Postman, SuperTest |
| E2E | User workflows | QA | Playwright, Cypress |
| Performance | Load/stress | QA | k6, Artillery |

### 3.2 Test Types

- [ ] Functional testing
- [ ] Regression testing
- [ ] Smoke testing
- [ ] Integration testing
- [ ] Performance testing
- [ ] Security testing
- [ ] Accessibility testing
- [ ] Compatibility testing

## 4. Test Environment

| Environment | Purpose | Data |
|-------------|---------|------|
| Dev | Unit/Integration | Mocked |
| Staging | E2E/Regression | Sanitized prod |
| Pre-prod | Performance/Security | Prod-like |

## 5. Test Data

### Requirements
- [Data requirement 1]
- [Data requirement 2]

### Sources
- Test data generators
- Sanitized production data
- Synthetic data sets

## 6. Entry/Exit Criteria

### Entry Criteria
- [ ] Code complete and merged
- [ ] Unit tests passing (>80% coverage)
- [ ] Environment deployed and stable
- [ ] Test data prepared

### Exit Criteria
- [ ] All critical test cases passed
- [ ] No P0/P1 bugs open
- [ ] Performance benchmarks met
- [ ] Security scan passed
- [ ] Sign-off from stakeholders

## 7. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| [Risk 1] | High/Med/Low | High/Med/Low | [Action] |
| [Risk 2] | | | |

## 8. Schedule

| Phase | Start | End | Owner |
|-------|-------|-----|-------|
| Test Planning | | | |
| Test Case Design | | | |
| Test Execution | | | |
| Bug Fixing | | | |
| Regression | | | |
| Sign-off | | | |

## 9. Deliverables

- [ ] Test cases in [TestRail/Xray]
- [ ] Automated test suite
- [ ] Test execution report
- [ ] Bug report summary
- [ ] Sign-off document
```

## Test Coverage Matrix

```markdown
## Feature Coverage Matrix

| Feature | Unit | Integration | E2E | Performance | Security |
|---------|------|-------------|-----|-------------|----------|
| User Login | ✓ | ✓ | ✓ | ✓ | ✓ |
| Checkout | ✓ | ✓ | ✓ | ✓ | ✓ |
| Search | ✓ | ✓ | ✓ | ✓ | |
| Profile | ✓ | ✓ | ✓ | | |

## Risk-Based Priority

| Priority | Description | Coverage Target |
|----------|-------------|-----------------|
| P0 | Critical path, revenue impact | 100% |
| P1 | Core functionality | 90% |
| P2 | Important features | 70% |
| P3 | Nice-to-have | 50% |
```

## Test Estimation

### Estimation Factors

```markdown
| Factor | Multiplier | Notes |
|--------|------------|-------|
| New feature | 1.5x | More exploration needed |
| Complex logic | 1.3x | More test cases |
| External integrations | 1.4x | More failure scenarios |
| Security-critical | 1.5x | Additional security tests |
| Performance-critical | 1.3x | Load testing needed |
```

### Estimation Template

```markdown
## Test Effort Estimation

| Activity | Hours | Notes |
|----------|-------|-------|
| Test case design | X | |
| Test data setup | X | |
| Manual execution | X | |
| Automation development | X | |
| Bug verification | X | |
| Regression testing | X | |
| **Total** | **X** | |
```

## Release Testing Checklist

```markdown
## Pre-Release Checklist

### Functional
- [ ] All user stories tested
- [ ] Acceptance criteria verified
- [ ] Edge cases covered
- [ ] Error handling verified

### Regression
- [ ] Core workflows passing
- [ ] No new failures in existing tests
- [ ] Cross-browser verification

### Non-Functional
- [ ] Performance benchmarks met
- [ ] Security scan completed
- [ ] Accessibility audit passed

### Documentation
- [ ] Release notes prepared
- [ ] Known issues documented
- [ ] Rollback plan in place

### Sign-off
- [ ] QA sign-off
- [ ] Product sign-off
- [ ] Engineering sign-off
```

## Best Practices

1. **Risk-based testing** - Focus effort on highest-risk areas
2. **Shift left** - Involve QA early in development
3. **Automate wisely** - Automate stable, repetitive tests
4. **Data independence** - Tests shouldn't depend on specific data
5. **Environment parity** - Test environments should mirror production
6. **Clear criteria** - Define pass/fail criteria upfront
7. **Continuous feedback** - Report status early and often
