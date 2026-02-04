---
name: migration-safety
description: |
  CRITICAL SAFETY SKILL - Use this skill before running ANY database migration. Activate when the user wants
  to run migrations, rollback migrations, re-run migrations, or modify database schema. This skill prevents
  data loss from migrations by enforcing backup requirements, checking existing data, and requiring explicit
  confirmation before destructive changes.
---

# Migration Safety

**CRITICAL: Migrations can destroy production data. This skill prevents catastrophic data loss.**

## The Problem This Solves

Real disaster scenarios:
- Migration fails halfway, rollback deletes all records
- Re-running migration truncates existing data
- Column drop removes 50,000 user records
- Foreign key constraint fails, cascade deletes everything
- "Fresh migration" on wrong database wipes production

## Golden Rules

1. **NEVER run migrations without checking for existing data first**
2. **ALWAYS create a backup before ANY migration**
3. **NEVER re-run a migration that previously partially completed without investigation**
4. **ALWAYS verify you're on the correct database/environment**
5. **NEVER trust `migrate:fresh` or `migrate:reset` on any database with data**

## Pre-Migration Checklist

### Step 1: Environment Verification

```bash
# Check current environment
echo $NODE_ENV
echo $RAILS_ENV
echo $APP_ENV

# Check database connection
# Laravel
php artisan db:show

# Rails
rails db:version

# Node.js (check connection string)
echo $DATABASE_URL
```

**RED FLAGS - STOP IMMEDIATELY:**
- Environment shows "production", "prod", "live"
- Database URL contains production hostname
- You're not 100% certain which database you're connected to

### Step 2: Check Existing Data

```sql
-- Before ANY migration, check what exists
SELECT table_name,
       (SELECT COUNT(*) FROM information_schema.columns
        WHERE table_name = t.table_name) as columns,
       (SELECT reltuples::bigint FROM pg_class
        WHERE relname = t.table_name) as estimated_rows
FROM information_schema.tables t
WHERE table_schema = 'public'
ORDER BY table_name;
```

**Present this to user:**
```markdown
## Current Database State

**Database:** [name]
**Environment:** [env]
**Tables:** [count]
**Total estimated rows:** [count]

| Table | Columns | Estimated Rows |
|-------|---------|----------------|
| users | 15 | 12,450 |
| orders | 23 | 89,234 |
| ... | ... | ... |

⚠️ This migration will affect a database with EXISTING DATA.
```

### Step 3: Analyze Migration Impact

Before running, analyze each pending migration:

```markdown
## Migration Analysis

### Migration: 2024_01_15_create_orders_table
**Type:** CREATE TABLE
**Risk Level:** LOW
**Existing Data Impact:** None (new table)

### Migration: 2024_01_16_drop_legacy_users
**Type:** DROP TABLE
**Risk Level:** CRITICAL ⛔
**Existing Data Impact:** Will DELETE 1,247 records permanently
**Recommendation:** BACKUP REQUIRED before proceeding

### Migration: 2024_01_17_add_email_to_users
**Type:** ADD COLUMN
**Risk Level:** LOW
**Existing Data Impact:** None (nullable column)
```

### Step 4: Create Backup

**MANDATORY before any migration that:**
- Drops tables
- Drops columns
- Modifies column types
- Adds NOT NULL constraints to existing columns
- Runs on a database with >100 rows

```bash
# PostgreSQL
pg_dump -Fc database_name > backup_$(date +%Y%m%d_%H%M%S).dump

# MySQL
mysqldump -u user -p database_name > backup_$(date +%Y%m%d_%H%M%S).sql

# SQLite
cp database.sqlite database.sqlite.backup.$(date +%Y%m%d_%H%M%S)

# Verify backup
ls -la backup_*.{dump,sql} 2>/dev/null || ls -la *.backup.* 2>/dev/null
```

## Migration Execution Protocol

### For Safe Migrations (CREATE, ADD COLUMN nullable)

```bash
# Run with verbose output
php artisan migrate --pretend  # Laravel: see SQL first
rails db:migrate:status        # Rails: check status
npx prisma migrate deploy      # Prisma: production deploy
```

### For Risky Migrations (DROP, MODIFY, CONSTRAINTS)

```markdown
⚠️ **DESTRUCTIVE MIGRATION WARNING**

**Migration:** 2024_01_16_drop_legacy_users
**Action:** DROP TABLE `legacy_users`
**Records that will be PERMANENTLY DELETED:** 1,247

**Backup created:** backup_20240116_143022.dump
**Backup verified:** ✓ (15.2 MB)

**Recovery command if needed:**
```bash
pg_restore -d database_name backup_20240116_143022.dump
```

Do you want to proceed? Type 'yes, delete 1247 records' to confirm.
```

### For Fresh/Reset Migrations

**NEVER suggest `migrate:fresh` or `migrate:reset` without this warning:**

```markdown
⛔ **CRITICAL WARNING: COMPLETE DATA DESTRUCTION**

You requested: `migrate:fresh` / `migrate:reset`

This will:
1. DROP ALL TABLES in the database
2. DELETE ALL DATA (estimated 156,789 records)
3. Re-run all migrations from scratch

**This is IRREVERSIBLE without a backup.**

Current database: [name]
Environment: [env]
Tables to be dropped: [count]
Records to be deleted: [count]

**Is this a development database with test data you can recreate?**

If YES and you're certain: Type 'yes, destroy all data in [database_name]'
If NO or UNSURE: Type 'cancel' and I'll help you with a safer approach.
```

## Handling Failed Migrations

### Scenario: Migration Failed Halfway

**WRONG approach:**
```bash
# DO NOT just re-run!
php artisan migrate
```

**RIGHT approach:**

```markdown
## Migration Failure Analysis

**Failed migration:** 2024_01_17_modify_orders
**Error:** Column 'status' cannot be dropped because of foreign key constraint
**State:** PARTIALLY APPLIED

**Current situation:**
- Migration started but did not complete
- Database may be in inconsistent state
- Re-running may cause data loss

**Safe recovery steps:**

1. **Check current state:**
   ```sql
   SELECT * FROM migrations ORDER BY id DESC LIMIT 5;
   ```

2. **Check what was applied:**
   ```sql
   -- Verify table structure
   \d orders
   ```

3. **Options:**
   A) Fix the migration and run manually
   B) Rollback to known good state (if possible)
   C) Restore from backup (safest)

**Recommendation:** Option C - Restore from backup

Would you like me to proceed with restoring from the backup at `backup_20240117_091534.dump`?
```

### Scenario: Need to Re-run Migration

**NEVER re-run without investigation:**

```markdown
## Re-run Migration Request

You want to re-run: `2024_01_15_seed_products`

**Analysis:**
- This migration inserts seed data
- Table `products` currently has 5,234 records
- Re-running may:
  - Duplicate records (if no unique constraints)
  - Fail on conflicts (if unique constraints exist)
  - Delete existing records (if migration truncates first)

**Migration file check:**
```php
// Found in migration:
DB::table('products')->truncate();  // ⛔ DANGEROUS - Will delete 5,234 records!
```

**Safe alternatives:**
1. Create a new migration that only adds missing records
2. Run specific seed commands instead
3. Manually insert only the records you need

**Do NOT re-run this migration unless you want to delete 5,234 records.**
```

## Framework-Specific Safety

### Laravel

```php
// Safe migration patterns

// GOOD: Check before truncate
public function up()
{
    if (Schema::hasTable('products') && DB::table('products')->count() > 0) {
        throw new Exception('Cannot run: products table has existing data');
    }
    // ... migration code
}

// GOOD: Soft delete instead of drop
public function up()
{
    Schema::rename('legacy_users', 'legacy_users_archived_' . date('Ymd'));
}

// GOOD: Backup before destructive operation
public function up()
{
    // Export existing data first
    $users = DB::table('legacy_users')->get();
    Storage::put('backups/legacy_users_' . date('Ymd') . '.json', $users->toJson());

    Schema::dropIfExists('legacy_users');
}
```

### Rails

```ruby
# Safe migration patterns

# GOOD: Reversible migrations with safety checks
class DropLegacyUsers < ActiveRecord::Migration[7.0]
  def up
    count = LegacyUser.count
    if count > 0
      raise "Cannot drop legacy_users: #{count} records exist. Backup first!"
    end
    drop_table :legacy_users
  end

  def down
    create_table :legacy_users do |t|
      # ... columns
    end
  end
end

# GOOD: Use safety_assured for intentional destructive changes
class RemoveEmailFromUsers < ActiveRecord::Migration[7.0]
  def change
    safety_assured { remove_column :users, :deprecated_email }
  end
end
```

### Prisma

```typescript
// Safe migration patterns

// Check before migrate
async function safeMigrate() {
  const count = await prisma.user.count();

  if (count > 0) {
    console.error(`Database has ${count} users. Create backup first!`);
    process.exit(1);
  }

  // Run migration
  execSync('npx prisma migrate deploy');
}

// Never use reset in production
if (process.env.NODE_ENV === 'production') {
  console.error('prisma migrate reset is DISABLED in production');
  process.exit(1);
}
```

## Recovery Procedures

### Quick Recovery from Backup

```bash
# PostgreSQL
pg_restore -c -d database_name backup_file.dump

# MySQL
mysql -u user -p database_name < backup_file.sql

# SQLite
cp database.sqlite.backup database.sqlite
```

### Point-in-Time Recovery (if available)

```sql
-- PostgreSQL with WAL archiving
SELECT pg_create_restore_point('before_migration');

-- After disaster
-- Restore to the named point
```

## Best Practices Summary

| Practice | Why |
|----------|-----|
| Always backup before migrations | Recovery from mistakes |
| Check existing data first | Awareness of impact |
| Use `--pretend` / dry-run | See changes before applying |
| Never use fresh/reset on data | Prevents accidental deletion |
| Verify environment | Prevents production disasters |
| Keep migrations reversible | Enables rollback |
| Test migrations on copy first | Catches issues early |
| Log migration runs | Audit trail |

## Red Flags Checklist

Stop immediately if:
- [ ] Not 100% sure which database you're connected to
- [ ] Environment might be production
- [ ] No backup exists
- [ ] Migration includes DROP or TRUNCATE
- [ ] Previous migration failed partway
- [ ] Someone else might be using the database
- [ ] You're tired or distracted

**When in doubt, DON'T migrate. Ask first.**
