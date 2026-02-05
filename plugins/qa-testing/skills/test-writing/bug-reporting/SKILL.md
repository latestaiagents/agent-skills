---
name: bug-reporting
description: |
  Write clear, actionable bug reports that help developers fix issues quickly.
  Use this skill when reporting bugs, documenting defects, or improving bug tracking processes.
  Activate when: bug report, defect, issue report, bug tracking, reproduction steps, bug template.
---

# Bug Reporting

**Write clear, actionable bug reports that get fixed faster.**

## When to Use

- Reporting a new bug
- Improving existing bug reports
- Training team on bug reporting
- Setting up bug templates
- Triaging and prioritizing issues

## Bug Report Template

```markdown
## Bug: [Clear, descriptive title]

**ID:** BUG-[number]
**Reporter:** [Name]
**Date:** [Date]

### Severity & Priority
- **Severity:** Critical / High / Medium / Low
- **Priority:** P0 / P1 / P2 / P3

### Environment
- **OS:** [e.g., macOS 14.1, Windows 11]
- **Browser:** [e.g., Chrome 120.0.6099]
- **App Version:** [e.g., 2.3.1]
- **Environment:** [Production / Staging / Dev]
- **Device:** [Desktop / Mobile / Tablet]

### Description
[1-2 sentences describing what's wrong]

### Steps to Reproduce
1. [First step]
2. [Second step]
3. [Third step]
4. [Continue until bug occurs]

### Expected Result
[What should happen]

### Actual Result
[What actually happens]

### Evidence
- Screenshot: [attachment]
- Video: [link]
- Console logs: [attachment]
- Network logs: [attachment]

### Additional Context
- Frequency: [Always / Sometimes / Rarely]
- Workaround: [If any]
- Related issues: [Links]

### Technical Notes (if known)
- Error message: [exact text]
- Stack trace: [if available]
- Related code: [file/function if known]
```

## Example Bug Reports

### Good Bug Report

```markdown
## Bug: Checkout fails with "undefined" error when applying discount code

**ID:** BUG-1234
**Reporter:** Jane Smith
**Date:** 2026-02-05

### Severity & Priority
- **Severity:** High (blocks purchases)
- **Priority:** P1

### Environment
- **OS:** macOS 14.2
- **Browser:** Chrome 121.0.6167.85
- **App Version:** 3.2.0
- **Environment:** Production

### Description
When applying a valid discount code at checkout, the page shows an "undefined" error and prevents order completion.

### Steps to Reproduce
1. Add any item to cart
2. Go to checkout (/checkout)
3. Enter discount code "SAVE20"
4. Click "Apply"
5. Error appears, "Apply" button becomes disabled

### Expected Result
- Discount of 20% applied to order total
- Updated total displayed
- User can proceed to payment

### Actual Result
- Red error message: "Error: undefined"
- Apply button grayed out
- Cannot proceed with checkout
- Must refresh page to continue (without discount)

### Evidence
- Screenshot: [checkout-error.png]
- Console error: `TypeError: Cannot read property 'value' of undefined at applyDiscount (checkout.js:142)`
- Network: POST /api/discounts returns 200 with valid response

### Additional Context
- Frequency: Always (100% reproducible)
- Workaround: Complete checkout without discount code
- Started after deploy on 2026-02-04
- Works correctly on staging environment

### Technical Notes
- Error in checkout.js:142
- Appears to be missing null check on discount response
- Related PR: #1456 (discount feature)
```

### Poor Bug Report (Don't Do This)

```markdown
## Bug: Checkout broken

Checkout doesn't work. Please fix ASAP.

Attached screenshot.
```

## Severity Guidelines

| Severity | Definition | Examples |
|----------|------------|----------|
| **Critical** | System down, data loss, security breach | App crashes on launch, payment processing fails for all users, SQL injection vulnerability |
| **High** | Major feature broken, significant user impact | Cannot complete checkout, search returns wrong results, login fails intermittently |
| **Medium** | Feature partially working, workaround exists | Filter doesn't work on one category, export missing some columns |
| **Low** | Minor issue, cosmetic, edge case | Typo in error message, button alignment off by 2px, rare edge case fails |

## Priority Guidelines

| Priority | Definition | Response |
|----------|------------|----------|
| **P0** | Drop everything | Fix immediately, hotfix to prod |
| **P1** | Current sprint | Fix within 1-2 days |
| **P2** | Next sprint | Plan for upcoming sprint |
| **P3** | Backlog | Fix when time permits |

## Reproduction Tips

### Be Specific

```markdown
❌ Bad: "Click the button"
✅ Good: "Click the blue 'Submit' button in the bottom-right corner of the form"

❌ Bad: "Enter some data"
✅ Good: "Enter 'test@example.com' in the email field"

❌ Bad: "Wait a while"
✅ Good: "Wait approximately 30 seconds until the loading spinner disappears"
```

### Include State

```markdown
### Preconditions
- Logged in as user: testuser@example.com
- Shopping cart contains 3 items totaling $150
- User has no saved payment methods
- User's account was created > 30 days ago
```

### Document Environment

```bash
# Useful commands for environment info

# Browser version (from console)
navigator.userAgent

# OS info
uname -a  # Linux/Mac
systeminfo  # Windows

# App version
cat package.json | grep version
```

## Evidence Collection

### Screenshots

```markdown
Best practices:
- Capture full screen or relevant area
- Highlight the issue (circle, arrow)
- Include URL bar when relevant
- Show error messages clearly
- Use before/after comparisons
```

### Screen Recordings

```markdown
Tools:
- macOS: Cmd+Shift+5
- Windows: Win+G (Game Bar)
- Chrome: Ctrl+Shift+R (DevTools)
- Loom, ScreenPal for sharing

Best practices:
- Keep under 60 seconds
- Narrate or add captions
- Show the steps clearly
- Include the error occurring
```

### Console Logs

```javascript
// Capture console errors
// Open DevTools (F12) > Console tab

// Copy all console output:
// Right-click > Save as... (Chrome)

// Filter errors only:
// Click 'Errors' filter in console
```

### Network Logs

```markdown
How to capture:
1. Open DevTools > Network tab
2. Enable "Preserve log"
3. Reproduce the issue
4. Right-click > "Save all as HAR"

What to look for:
- Failed requests (red)
- Status codes (4xx, 5xx)
- Response bodies
- Request payloads
```

## Bug Triage Process

```markdown
## Daily Triage Meeting

### Agenda (15 min)
1. Review new bugs (5 min)
2. Assign owners (3 min)
3. Adjust priorities (3 min)
4. Review blockers (4 min)

### Triage Questions
- Is this a duplicate?
- Is it reproducible?
- What's the user impact?
- Is there a workaround?
- What's the fix complexity?
- Who should own this?
```

## Bug Lifecycle

```
New → Triaged → In Progress → In Review → Verified → Closed
                    ↓
              Cannot Reproduce
                    ↓
              Need More Info
                    ↓
               Won't Fix
                    ↓
               Duplicate
```

## Best Practices

1. **One bug per report** - Don't combine multiple issues
2. **Search first** - Check for duplicates before filing
3. **Be objective** - Describe facts, not frustrations
4. **Include evidence** - Screenshots, logs, videos
5. **Stay updated** - Respond to questions promptly
6. **Verify fixes** - Confirm the bug is actually fixed
7. **Close the loop** - Update status when resolved
