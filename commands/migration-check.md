---
description: Analyze database migrations for risks before running them
---

# /migration-check

Analyze database migrations for risks before running them.

## Step 1: Environment Check

```bash
# Verify environment
echo "Environment: $NODE_ENV"
echo "Database: $DATABASE_URL"
```

**⚠️ If this is production, proceed with extra caution.**

## Step 2: Check Pending Migrations

```bash
# Laravel
php artisan migrate:status

# Rails
rails db:migrate:status

# Prisma
npx prisma migrate status

# Knex
npx knex migrate:status
```

## Step 3: Analyze Each Migration

For each pending migration, I'll assess:

| Check | What I Look For |
|-------|-----------------|
| **Destructive DDL** | DROP TABLE, DROP COLUMN, TRUNCATE |
| **Data modifications** | DELETE, UPDATE affecting existing data |
| **Constraint changes** | Adding NOT NULL to populated columns |
| **Index changes** | Dropping indexes, adding unique constraints |
| **Foreign keys** | CASCADE DELETE risks |

### Risk Classification

- **CRITICAL**: Will delete/modify existing data
- **HIGH**: May fail or lock tables
- **MEDIUM**: Adds constraints, may need backfill
- **LOW**: Safe additive changes (CREATE, ADD COLUMN nullable)

## Step 4: Check Existing Data

Before migrations that DROP or modify:

```sql
-- Check row counts in affected tables
SELECT 'users' as table_name, COUNT(*) as rows FROM users
UNION ALL
SELECT 'orders', COUNT(*) FROM orders;

-- Check if column being dropped has data
SELECT COUNT(*) FROM users WHERE deprecated_column IS NOT NULL;
```

## Step 5: Pre-Migration Backup

**Required for CRITICAL/HIGH risk migrations:**

```bash
# Full database backup
pg_dump -Fc database_name > backup_pre_migration_$(date +%Y%m%d_%H%M%S).dump

# Single table backup
pg_dump -t affected_table database_name > table_backup.sql
```

## Step 6: Migration Decision

I'll present:

```
## Migration Risk Assessment

### Migration: 2024_01_15_drop_legacy_users
**Risk Level:** CRITICAL
**Action:** DROP TABLE legacy_users
**Data at Risk:** 1,247 records

**Recommendation:**
1. Create backup: `pg_dump -t legacy_users db > legacy_users_backup.sql`
2. Verify these records are not needed
3. Proceed only after confirmation

### Migration: 2024_01_16_add_email_index
**Risk Level:** LOW
**Action:** ADD INDEX on email column
**Data at Risk:** None

**Recommendation:** Safe to proceed
```

## Step 7: Safe Execution

```bash
# Run with verbose output
php artisan migrate --pretend  # Preview only (Laravel)
rails db:migrate:status        # Check status (Rails)

# If safe, execute
php artisan migrate
rails db:migrate
npx prisma migrate deploy
```

## Rollback Plan

Document before running:

```markdown
## Rollback Instructions

If migration fails:
1. Run: `php artisan migrate:rollback`
2. Restore from backup: `pg_restore -d database backup.dump`
3. Notify team: [contact info]
```

---

**Show me your pending migrations or migration files, and I'll analyze the risks.**
