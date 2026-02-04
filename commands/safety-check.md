---
description: Run safety checks before any destructive operation (database, files, git)
---

# /safety-check

Run comprehensive safety checks before any destructive operation.

## What Are You About To Do?

Tell me the operation and I'll assess the risks:

- **Database**: DELETE, DROP, TRUNCATE, migrations, resets
- **Files**: rm, mv, bulk operations, directory cleanup
- **Git**: force push, reset --hard, rebase, clean
- **Deployment**: production deploys, rollbacks, scaling
- **API**: DELETE endpoints, bulk updates, cache flush

## Safety Check Protocol

### 1. Environment Verification

```bash
# What environment are we in?
echo $NODE_ENV
echo $RAILS_ENV
echo $APP_ENV

# What database are we connected to?
echo $DATABASE_URL
```

**RED FLAG**: If you see "production", "prod", or "live" - STOP and verify intent.

### 2. Impact Assessment

I'll analyze:
- **Scope**: How many records/files affected?
- **Reversibility**: Can this be undone?
- **Dependencies**: What else will break?
- **Users affected**: Who will notice?

### 3. Pre-Operation Checklist

Before ANY destructive operation:

- [ ] Environment confirmed (not production unless intended)
- [ ] Backup created
- [ ] Impact scope understood
- [ ] Recovery plan documented
- [ ] Team notified (if shared resources)

### 4. Backup Creation

```bash
# Database
pg_dump database_name > backup_$(date +%Y%m%d_%H%M%S).sql

# Files
cp -r target_directory target_directory.backup.$(date +%Y%m%d_%H%M%S)

# Git
git stash push -m "safety backup"
git branch backup-$(date +%Y%m%d_%H%M%S)
```

### 5. Confirmation Protocol

I will ALWAYS show you:

```
⚠️ DESTRUCTIVE OPERATION WARNING

Operation: [specific command]
Environment: [dev/staging/prod]
Impact: [exactly what will be affected]
Reversibility: [can this be undone?]

Type "[specific confirmation phrase]" to proceed.
```

## Quick Reference

| Operation | Risk Level | Requires |
|-----------|------------|----------|
| `DROP DATABASE` | CRITICAL | Full backup + explicit confirmation |
| `DELETE` without WHERE | CRITICAL | Explicit confirmation |
| `rm -rf` on directory | CRITICAL | Path verification + confirmation |
| `git push --force` | HIGH | Branch check + team notification |
| `git reset --hard` | HIGH | Stash uncommitted changes first |
| Database migration | MEDIUM | Check existing data + backup |
| `rm` single file | LOW | Verify file path |

---

**What operation do you want me to check?**
