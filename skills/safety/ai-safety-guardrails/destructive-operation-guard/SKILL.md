---
name: destructive-operation-guard
description: |
  CRITICAL SAFETY SKILL - Always active. Use this skill to prevent destructive operations without explicit
  user confirmation. Activate BEFORE any operation that deletes data, resets state, overwrites files,
  force pushes, drops tables, truncates data, runs migrations, or performs any irreversible action.
  This skill MUST be consulted before executing potentially harmful commands.
---

# Destructive Operation Guard

**CRITICAL: This skill defines mandatory safety protocols for AI agents.**

## Core Principle

**NEVER execute a destructive operation without EXPLICIT user confirmation.**

"Destructive" means any operation that:
- Deletes data
- Overwrites existing data
- Resets state
- Cannot be easily undone
- Affects production systems
- Modifies data belonging to others

## Destructive Operations Checklist

### ALWAYS ASK BEFORE:

#### Database Operations
- [ ] `DROP TABLE`, `DROP DATABASE`
- [ ] `TRUNCATE TABLE`
- [ ] `DELETE` without specific WHERE clause
- [ ] `DELETE` affecting more than 10 rows
- [ ] `UPDATE` without specific WHERE clause
- [ ] `UPDATE` affecting more than 10 rows
- [ ] Running migrations on production
- [ ] Running migrations that delete columns/tables
- [ ] Seeding that overwrites existing data
- [ ] Any `--force` flag on database commands

#### File System Operations
- [ ] `rm -rf` on any directory
- [ ] `rm` on multiple files
- [ ] Overwriting existing files
- [ ] Moving files that might overwrite destinations
- [ ] Clearing directories
- [ ] Any recursive delete operation

#### Git Operations
- [ ] `git push --force` or `git push -f`
- [ ] `git reset --hard`
- [ ] `git clean -fd`
- [ ] `git checkout .` (discards all changes)
- [ ] `git stash drop`
- [ ] `git branch -D` (force delete)
- [ ] `git rebase` on shared branches
- [ ] Amending pushed commits

#### Deployment Operations
- [ ] Deploying to production
- [ ] Rolling back deployments
- [ ] Scaling down to zero
- [ ] Deleting cloud resources
- [ ] Modifying environment variables in production

#### API/Service Operations
- [ ] Calling DELETE endpoints
- [ ] Bulk update operations
- [ ] Cache invalidation (full flush)
- [ ] Queue purging
- [ ] Session/token invalidation for all users

## Mandatory Confirmation Protocol

### Step 1: Identify the Risk

Before ANY potentially destructive operation, assess:

```markdown
## Risk Assessment

**Operation:** [What you're about to do]
**Scope:** [What will be affected]
**Reversibility:** [Can this be undone? How?]
**Data at Risk:** [What data could be lost]
**Affected Users:** [Who will be impacted]
```

### Step 2: Present to User

ALWAYS show the user:

```markdown
⚠️ **DESTRUCTIVE OPERATION WARNING**

I'm about to perform an operation that could result in data loss.

**Action:** [Specific command/operation]
**Impact:**
- [Specific impact 1]
- [Specific impact 2]

**Data that will be affected:**
- [Specific data 1]
- [Specific data 2]

**This action is [REVERSIBLE/IRREVERSIBLE].**

[If reversible: "Recovery method: [how to recover]"]
[If irreversible: "⛔ THIS CANNOT BE UNDONE"]

Do you want me to proceed? Please confirm with "yes" or "proceed".
```

### Step 3: Require Explicit Confirmation

**ONLY proceed if user explicitly says:**
- "yes"
- "proceed"
- "confirm"
- "do it"
- "go ahead"

**DO NOT proceed if user says:**
- "ok" (ambiguous)
- "sure" (ambiguous)
- Anything that could be interpreted as casual agreement
- Nothing (silence is NOT consent)

### Step 4: Create Safety Net BEFORE Execution

```bash
# For database operations
pg_dump database_name > backup_$(date +%Y%m%d_%H%M%S).sql

# For file operations
cp -r target_directory target_directory.backup.$(date +%Y%m%d_%H%M%S)

# For git operations
git stash
git branch backup-$(date +%Y%m%d_%H%M%S)
```

### Step 5: Execute with Logging

```bash
# Log the operation
echo "[$(date)] DESTRUCTIVE OP: [command]" >> ~/.destructive_ops.log
echo "[$(date)] USER CONFIRMED: yes" >> ~/.destructive_ops.log
echo "[$(date)] BACKUP LOCATION: [backup_path]" >> ~/.destructive_ops.log

# Execute
[actual command]

# Log result
echo "[$(date)] RESULT: $?" >> ~/.destructive_ops.log
```

## Examples of Proper Guardrails

### Example 1: Database Migration

**WRONG:**
```
User: "Run the migration"
AI: [Runs migration immediately]
```

**RIGHT:**
```
User: "Run the migration"
AI: "Before running the migration, I need to check a few things:

1. **Checking migration status...**
   - 3 pending migrations found
   - Migration `2024_01_15_drop_users_table` will DROP the `legacy_users` table

2. **Checking existing data...**
   - `legacy_users` table has 1,247 records

⚠️ **WARNING: This migration will permanently delete 1,247 user records.**

Before proceeding, I recommend:
- [ ] Creating a backup: `pg_dump -t legacy_users db > legacy_users_backup.sql`
- [ ] Verifying these records are not needed
- [ ] Confirming you have a rollback plan

Do you want me to:
A) Create a backup first, then run migration
B) Show me the migration file content first
C) Run migration without backup (NOT RECOMMENDED)
D) Cancel

Please choose an option."
```

### Example 2: Resetting Development Environment

**WRONG:**
```
User: "Reset the database"
AI: [Runs db:reset immediately]
```

**RIGHT:**
```
User: "Reset the database"
AI: "I'll help you reset the database. First, let me check the current state:

**Current database:** myapp_development
**Tables:** 47 tables
**Total records:** 15,234 records

**Records by table (top 5):**
- users: 523 records
- orders: 8,921 records
- products: 1,245 records
- logs: 4,102 records
- sessions: 443 records

⚠️ **A database reset will:**
1. Drop ALL tables
2. Delete ALL 15,234 records
3. Recreate schema from migrations
4. Run seeders (if configured)

**Is this a development or production database?**

If development, I can proceed with your confirmation.
If production, I STRONGLY recommend against this operation.

Type 'yes, reset development database' to confirm, or 'cancel' to abort."
```

### Example 3: Git Force Push

**WRONG:**
```
User: "Force push to fix the history"
AI: git push --force
```

**RIGHT:**
```
User: "Force push to fix the history"
AI: "Before force pushing, I need to verify this is safe:

**Current branch:** feature/user-auth
**Remote tracking:** origin/feature/user-auth
**Local commits ahead:** 3
**Remote commits that will be OVERWRITTEN:** 2

⚠️ **Force pushing will:**
- Overwrite 2 commits on the remote
- These commits may be lost if not backed up elsewhere
- Other developers pulling this branch will have conflicts

**Safety checks:**
- [ ] Is this a shared branch? (Others may have based work on it)
- [ ] Do you have the overwritten commits backed up?
- [ ] Have you notified team members?

I recommend using `--force-with-lease` instead of `--force`:
```bash
git push --force-with-lease origin feature/user-auth
```
This will fail if the remote has commits you haven't seen, preventing accidental data loss.

Proceed with:
A) `--force-with-lease` (safer)
B) `--force` (I understand the risks)
C) Cancel"
```

## Recovery Information

Always provide recovery information BEFORE the operation:

```markdown
## Recovery Plan

**If something goes wrong:**

1. **Backup location:** [path]
2. **Restore command:** [command]
3. **Point of no return:** [when recovery becomes impossible]
4. **Who to contact:** [if applicable]
```

## Red Flags - STOP Immediately

If you encounter these, STOP and ask for clarification:

1. **Production indicators:**
   - Database URL contains "prod", "production", "live"
   - Environment variables show `NODE_ENV=production`
   - Server hostname contains production identifiers

2. **Scale indicators:**
   - More than 1000 records to be affected
   - Multiple tables involved
   - Operation affects shared resources

3. **Ambiguous requests:**
   - "Clean up the database"
   - "Reset everything"
   - "Start fresh"
   - "Remove all the old data"

4. **Missing context:**
   - No clear scope defined
   - No backup mentioned
   - No rollback plan discussed

## Best Practices Summary

1. **ASK FIRST** - Always confirm before destructive operations
2. **BACKUP FIRST** - Create safety net before execution
3. **SCOPE CLEARLY** - Show exactly what will be affected
4. **PROVIDE OPTIONS** - Give user choices, including "cancel"
5. **LOG EVERYTHING** - Record what was done for audit trail
6. **VERIFY ENVIRONMENT** - Double-check prod vs dev
7. **SMALLEST SCOPE** - Do minimum necessary, not maximum possible
8. **REVERSIBILITY** - Always explain if/how to undo
