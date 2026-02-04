---
name: codebase-context-builder
description: |
  Use this skill when preparing context for AI coding assistants. Activate when the user wants to help
  AI understand their codebase, provide better context for code generation, improve AI responses,
  create context files, set up CLAUDE.md or similar context documents, or optimize prompts for code tasks.
---

# Codebase Context Builder

Prepare optimal context so AI assistants understand your codebase deeply.

## When to Use

- Setting up a new project for AI-assisted development
- AI assistant generates code that doesn't fit your patterns
- Creating CLAUDE.md, AGENTS.md, or similar context files
- Preparing context for complex code generation tasks
- Improving AI understanding of your architecture

## The Context Hierarchy

```
┌─────────────────────────────────────────┐
│ Level 1: Project Context (Always Load)  │
│ - Tech stack, conventions, structure    │
│ - CLAUDE.md / AGENTS.md files           │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ Level 2: Domain Context (Task-Specific) │
│ - Related files, types, interfaces      │
│ - Existing patterns to follow           │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ Level 3: Task Context (Per-Request)     │
│ - Specific requirements                 │
│ - Examples of expected output           │
└─────────────────────────────────────────┘
```

## Creating CLAUDE.md

Place in project root. AI assistants read this automatically.

```markdown
# Project: [Name]

## Tech Stack
- **Frontend:** React 19, TypeScript 5.4, Tailwind CSS 4.0
- **Backend:** Node.js 22, Express 5, PostgreSQL 16
- **Testing:** Vitest, Playwright
- **Package Manager:** pnpm 9

## Project Structure
```
src/
├── components/     # React components (PascalCase)
├── hooks/          # Custom hooks (use* prefix)
├── services/       # API clients and business logic
├── types/          # TypeScript interfaces
├── utils/          # Pure utility functions
└── pages/          # Route components
```

## Code Conventions

### Naming
- Components: PascalCase (`UserProfile.tsx`)
- Hooks: camelCase with `use` prefix (`useAuth.ts`)
- Utils: camelCase (`formatDate.ts`)
- Types: PascalCase with `I` prefix for interfaces (`IUser`)

### Patterns
- Use functional components with hooks
- Prefer `const` arrow functions for components
- Use named exports, not default exports
- Error handling with custom `AppError` class

### Imports Order
1. External packages
2. Internal modules (absolute paths)
3. Relative imports
4. Styles

## API Conventions
- Base URL: `/api/v1`
- Auth: Bearer token in Authorization header
- Errors: `{ error: { code, message, details } }`

## Testing
- Unit tests: `*.test.ts` colocated with source
- E2E tests: `e2e/*.spec.ts`
- Run: `pnpm test` (unit), `pnpm test:e2e` (e2e)

## Common Commands
```bash
pnpm dev          # Start development
pnpm build        # Production build
pnpm test         # Run tests
pnpm lint         # Lint code
```

## Important Files
- `src/services/api.ts` - API client with interceptors
- `src/hooks/useAuth.ts` - Authentication hook
- `src/types/index.ts` - Shared type definitions
```

## Building Task-Specific Context

### Step 1: Identify Relevant Files

```bash
# Find files related to your task
# For a "user profile" feature:

# Find existing user-related code
grep -r "user" --include="*.ts" -l src/

# Find related types
grep -r "interface.*User" --include="*.ts" src/types/

# Find similar components
ls src/components/ | grep -i profile
```

### Step 2: Include Key Dependencies

When asking AI to modify code, include:

1. **The file to modify**
2. **Types/interfaces used**
3. **Related utilities**
4. **Similar existing implementations**

**Example prompt structure:**
```markdown
## Context Files

### src/types/user.ts
[paste file content]

### src/services/userService.ts
[paste file content]

### src/components/UserCard.tsx (similar component)
[paste file content]

## Task
Create a UserProfile component that displays user details
and allows editing. Follow the patterns in UserCard.
```

### Step 3: Provide Examples

```markdown
## Expected Output Example

The component should look similar to this pattern:

```tsx
export const UserProfile: React.FC<UserProfileProps> = ({ userId }) => {
  const { data, isLoading } = useUser(userId);

  if (isLoading) return <Skeleton />;

  return (
    <Card>
      {/* Component content */}
    </Card>
  );
};
```

## Context Strategies by Task Type

### For New Features
```markdown
Include:
1. CLAUDE.md (project context)
2. Similar existing feature (as reference)
3. Relevant types and interfaces
4. API endpoint documentation
5. Design requirements or mockups
```

### For Bug Fixes
```markdown
Include:
1. The buggy code
2. Error message / stack trace
3. Expected vs actual behavior
4. Related test files
5. Recent changes to the file (git log)
```

### For Refactoring
```markdown
Include:
1. Current implementation
2. Target patterns/architecture
3. All usages of the code being refactored
4. Migration examples if changing APIs
```

### For Tests
```markdown
Include:
1. Code to be tested
2. Existing test patterns in the project
3. Test utilities and mocks
4. Coverage requirements
```

## Context File Templates

### AGENTS.md (For Multi-Agent Systems)
```markdown
# Agent Configuration

## Available Agents
- **code-reviewer**: Reviews code for best practices
- **test-writer**: Generates comprehensive tests
- **doc-generator**: Creates documentation

## Shared Context
All agents have access to:
- Project structure in CLAUDE.md
- Type definitions in src/types/
- Coding standards in .eslintrc

## Agent-Specific Instructions

### code-reviewer
Focus on:
- Security vulnerabilities
- Performance issues
- Code style violations
- Missing error handling

### test-writer
Requirements:
- Minimum 80% coverage
- Include edge cases
- Mock external dependencies
- Use existing test utilities
```

### .cursorrules / .claude/settings.json
```json
{
  "contextFiles": [
    "CLAUDE.md",
    "src/types/index.ts",
    "docs/API.md"
  ],
  "excludePatterns": [
    "node_modules",
    "dist",
    "*.test.ts"
  ]
}
```

## Optimizing Context Size

### Token Budget Guidelines (2026)
- Claude: 200K context, but 10-20K is optimal for speed
- GPT-4: 128K context
- Keep routine context under 5K tokens

### Strategies
1. **Summarize large files** instead of including full content
2. **Use file references** when AI can read files directly
3. **Include only relevant sections** of large files
4. **Create focused context files** per feature area

### Example: Summarized Context
```markdown
## UserService Summary
- Location: src/services/userService.ts
- Methods: getUser, updateUser, deleteUser, listUsers
- Uses: apiClient, UserSchema validation
- Auth: Requires authenticated user context
- Key pattern: All methods return Promise<Result<T, AppError>>
```

## Verifying Context Quality

### Checklist
- [ ] Tech stack and versions specified
- [ ] File naming conventions documented
- [ ] Import patterns shown
- [ ] Error handling patterns included
- [ ] Existing similar code referenced
- [ ] Types and interfaces available
- [ ] Test patterns documented

### Test Your Context
Ask the AI a simple question about your codebase conventions. If it answers correctly, your context is working.

```
"What naming convention should I use for a new React hook in this project?"

Expected: "Use camelCase with 'use' prefix, like useAuth.ts"
```
