---
description: Structure a clear, actionable bug report with reproduction steps
---

# /bug-report

Create a well-structured bug report that developers can act on.

## What I Need

Tell me:
- What went wrong?
- What did you expect?
- Steps you took before the bug
- Environment (browser, OS, app version)
- Any error messages or screenshots?

## What I'll Create

A complete bug report with:

1. **Clear title** - Descriptive summary
2. **Severity/Priority** - Impact assessment
3. **Environment** - Where it happened
4. **Steps** - Exact reproduction path
5. **Expected vs Actual** - Clear comparison
6. **Evidence** - Screenshots, logs guidance

## Bug Report Format

```markdown
## Bug: [Title]

### Severity: [Critical/High/Medium/Low]
### Priority: [P0/P1/P2/P3]

### Environment
- OS:
- Browser:
- App Version:

### Steps to Reproduce
1. ...
2. ...
3. ...

### Expected Result
[What should happen]

### Actual Result
[What happens]

### Evidence
[Screenshots, logs, videos]
```

## Example

```
You: /bug-report

The save button doesn't work after editing profile.
Chrome on Mac. No error shown.

Claude: [Structures complete bug report with clear steps,
suggests what evidence to collect, asks clarifying
questions about frequency and workarounds]
```

## Quality Checks

I'll ensure your report:
- Has reproducible steps
- Includes environment details
- Documents expected behavior
- Suggests evidence to attach
- Uses appropriate severity
