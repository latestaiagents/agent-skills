---
name: doc-sync-automation
description: |
  Use this skill when keeping documentation synchronized with code. Activate when the user needs to update
  docs after code changes, generate API documentation, create README updates, keep JSDoc/TSDoc in sync,
  or automate documentation workflows.
---

# Doc Sync Automation

Keep documentation synchronized with code changes automatically.

## When to Use

- Code changed and docs need updating
- Generating API documentation from code
- Creating or updating README files
- Keeping inline comments/JSDoc current
- Setting up documentation automation

## Documentation Types

| Type | Location | Sync Trigger |
|------|----------|--------------|
| Inline (JSDoc/TSDoc) | In code files | Code changes |
| API Reference | `/docs/api/` | Endpoint changes |
| README | Root or feature folders | Feature changes |
| Changelog | `CHANGELOG.md` | Any release |
| Architecture | `/docs/architecture/` | Structure changes |

## Inline Documentation (JSDoc/TSDoc)

### AI Prompt for JSDoc Generation

```markdown
Generate JSDoc comments for this function:

```typescript
async function createUser(
  email: string,
  password: string,
  options?: { role?: string; sendWelcome?: boolean }
): Promise<User> {
  // implementation
}
```

Include:
- Description
- @param for each parameter
- @returns description
- @throws if applicable
- @example usage
```

### Generated Output
```typescript
/**
 * Creates a new user account with the specified credentials.
 *
 * @param email - The user's email address (must be unique)
 * @param password - The user's password (min 8 characters)
 * @param options - Optional configuration
 * @param options.role - User role, defaults to 'user'
 * @param options.sendWelcome - Send welcome email, defaults to true
 * @returns The newly created user object
 * @throws {ValidationError} If email is invalid or password too short
 * @throws {DuplicateError} If email already exists
 *
 * @example
 * const user = await createUser('john@example.com', 'securepass123');
 *
 * @example
 * const admin = await createUser('admin@example.com', 'adminpass', {
 *   role: 'admin',
 *   sendWelcome: false
 * });
 */
```

### Batch Update JSDoc

```markdown
Update JSDoc for all exported functions in this file.
Keep existing descriptions if accurate, update if code changed.

```[paste file content]```

For each function, ensure:
- Description matches current behavior
- All parameters documented
- Return type documented
- Examples are valid
```

## API Documentation

### OpenAPI/Swagger from Code

```markdown
Generate OpenAPI 3.1 specification for this Express route:

```typescript
router.post('/users',
  authenticate,
  validate(createUserSchema),
  async (req, res) => {
    const user = await userService.create(req.body);
    res.status(201).json(user);
  }
);
```

Request schema:
```[paste validation schema]```

Response type:
```[paste User type]```
```

### Generated OpenAPI
```yaml
paths:
  /users:
    post:
      summary: Create a new user
      description: Creates a new user account. Requires authentication.
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
            example:
              email: "john@example.com"
              password: "securepassword123"
              role: "user"
      responses:
        '201':
          description: User created successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '400':
          description: Validation error
        '401':
          description: Unauthorized
        '409':
          description: Email already exists
```

## README Generation

### AI Prompt for README Updates

```markdown
Update this README section based on code changes:

**Current README section:**
```markdown
## Installation
npm install mypackage
```

**Code changes made:**
- Added pnpm support
- Added peer dependencies
- Changed minimum Node version to 20

Generate updated section reflecting these changes.
```

### README Sync Checklist

When code changes, check these README sections:

- [ ] **Installation** - Dependencies changed?
- [ ] **Quick Start** - API changed?
- [ ] **Configuration** - New options?
- [ ] **Examples** - Still valid?
- [ ] **Requirements** - Version requirements?

## Changelog Generation

### AI Prompt for Changelog

```markdown
Generate changelog entry for these commits:

```
abc123 feat(auth): add OAuth2 support for Google
def456 fix(api): handle null response from payment gateway
ghi789 chore(deps): update React to 19.0
jkl012 docs(readme): add deployment section
```

Follow Keep a Changelog format. Categorize as:
- Added, Changed, Deprecated, Removed, Fixed, Security
```

### Generated Changelog
```markdown
## [1.2.0] - 2026-02-04

### Added
- OAuth2 authentication support for Google accounts (#123)
- Deployment documentation in README

### Fixed
- Null response handling in payment gateway integration (#456)

### Changed
- Updated React from 18.x to 19.0
```

## Automation Setup

### Pre-commit Hook for Doc Sync

```json
// package.json
{
  "husky": {
    "hooks": {
      "pre-commit": "npm run docs:check"
    }
  },
  "scripts": {
    "docs:check": "node scripts/check-docs-sync.js",
    "docs:generate": "typedoc --out docs/api src/"
  }
}
```

### GitHub Action for Docs

```yaml
# .github/workflows/docs.yml
name: Documentation

on:
  push:
    branches: [main]
    paths:
      - 'src/**'
      - 'docs/**'

jobs:
  update-docs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Generate API docs
        run: npm run docs:generate

      - name: Check for changes
        id: changes
        run: |
          git diff --quiet docs/ || echo "changed=true" >> $GITHUB_OUTPUT

      - name: Commit docs
        if: steps.changes.outputs.changed == 'true'
        run: |
          git config user.name 'Documentation Bot'
          git config user.email 'docs@example.com'
          git add docs/
          git commit -m 'docs: auto-update API documentation'
          git push
```

## Doc-Code Sync Patterns

### Pattern 1: Colocated Docs
```
src/
├── users/
│   ├── userService.ts
│   ├── userService.test.ts
│   └── README.md  # Feature-specific docs
```

### Pattern 2: Central Docs with References
```
docs/
├── api/
│   └── users.md  # References src/users/
src/
├── users/
│   └── userService.ts  # @see docs/api/users.md
```

### Pattern 3: Generated from Code
```typescript
// Code with JSDoc
/**
 * @module UserService
 * @description User management functionality
 */

// Auto-generated to docs/api/UserService.md
```

## Verification

After documentation updates:

```bash
# Check for broken links
npx markdown-link-check docs/**/*.md

# Validate OpenAPI spec
npx @redocly/cli lint openapi.yaml

# Check code examples still work
npx ts-node scripts/test-doc-examples.ts
```
