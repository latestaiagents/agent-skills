---
description: Create optimal CLAUDE.md context file for your codebase
---

# /build-context

Create an optimal CLAUDE.md context file for your codebase.

## What is CLAUDE.md?

A `CLAUDE.md` file in your project root gives AI assistants essential context about your codebase. It helps AI:

- Understand your project structure
- Follow your coding conventions
- Use the right frameworks and patterns
- Avoid common mistakes specific to your project

## I'll Analyze Your Codebase

Let me scan your project to generate a CLAUDE.md. I'll look at:

1. **Project type** - Framework, language, architecture
2. **Directory structure** - Where things live
3. **Key files** - Entry points, config, important modules
4. **Dependencies** - Libraries you're using
5. **Conventions** - Code style, naming patterns

## Generated CLAUDE.md Template

```markdown
# Project Context

## Overview
[One paragraph describing what this project does]

## Tech Stack
- **Language:** [TypeScript/Python/etc.]
- **Framework:** [Next.js/Django/etc.]
- **Database:** [PostgreSQL/MongoDB/etc.]
- **Key libraries:** [List important dependencies]

## Project Structure
```
src/
├── components/    # React components
├── pages/         # Next.js pages (file-based routing)
├── lib/           # Shared utilities
├── api/           # API routes
└── types/         # TypeScript definitions
```

## Key Files
- `src/lib/db.ts` - Database connection and queries
- `src/lib/auth.ts` - Authentication logic
- `src/types/index.ts` - Shared type definitions

## Coding Conventions
- Use TypeScript strict mode
- Prefer functional components with hooks
- Use named exports, not default exports
- Error messages should be user-friendly

## Commands
- `npm run dev` - Start development server
- `npm run test` - Run tests
- `npm run build` - Production build

## Common Patterns

### API calls
```typescript
// Use the fetcher from lib/api
import { fetcher } from '@/lib/api';
const data = await fetcher('/api/users');
```

### Database queries
```typescript
// Use Prisma client from lib/db
import { prisma } from '@/lib/db';
const users = await prisma.user.findMany();
```

## Things to Avoid
- Don't use `any` type - define proper interfaces
- Don't commit .env files
- Don't use console.log in production code
- Don't make database calls in components (use API routes)

## Testing
- Tests are in `__tests__/` directories
- Use React Testing Library for components
- Mock external APIs in tests
```

## Customization Questions

To make your CLAUDE.md more useful, tell me:

1. **What are the most important files in your project?**
2. **What mistakes do developers commonly make?**
3. **What patterns should AI follow?**
4. **Any project-specific terminology?**

---

**Let me scan your project and I'll generate a customized CLAUDE.md.**
