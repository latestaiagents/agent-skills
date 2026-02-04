---
name: database-safety
description: |
  CRITICAL SAFETY SKILL - Use this skill before any database operation that modifies or deletes data.
  Activate when the user wants to DELETE, UPDATE, TRUNCATE, DROP, or reset database. This skill
  prevents accidental data loss by enforcing verification, scoping, and confirmation requirements.
---

# Database Safety

**CRITICAL: Database operations can be irreversible. This skill prevents data catastrophes.**

## Danger Levels

| Level | Operations | Requires |
|-------|------------|----------|
| ⛔ CRITICAL | DROP DATABASE, DROP TABLE, TRUNCATE | Backup + explicit confirmation |
| 🔴 HIGH | DELETE without WHERE, UPDATE without WHERE | Explicit confirmation + scope verification |
| 🟠 MEDIUM | DELETE with WHERE, UPDATE with WHERE | Scope verification + preview |
| 🟢 LOW | SELECT, INSERT, CREATE | Standard execution |

## Pre-Operation Protocol

### Before ANY Data Modification

**Step 1: Identify the scope**

```sql
-- Before DELETE: Count affected rows
SELECT COUNT(*) FROM users WHERE status = 'inactive';

-- Before UPDATE: Preview changes
SELECT id, current_value, 'new_value' as proposed_value
FROM table
WHERE condition;

-- Before DROP: Check foreign keys and dependencies
SELECT
    tc.table_name,
    kcu.column_name,
    ccu.table_name AS foreign_table_name
FROM information_schema.table_constraints AS tc
JOIN information_schema.key_column_usage AS kcu
    ON tc.constraint_name = kcu.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY'
    AND ccu.table_name = 'table_to_drop';
```

**Step 2: Present impact to user**

```markdown
## Database Operation Impact Assessment

**Operation:** DELETE FROM users WHERE last_login < '2023-01-01'
**Database:** production_db
**Environment:** production ⚠️

**Impact:**
- Rows to be deleted: 8,234
- Table total rows: 45,678
- Percentage affected: 18%

**Related data that may be affected:**
- orders table: 12,456 rows reference these users (CASCADE DELETE)
- sessions table: 3,234 active sessions will be invalidated
- audit_logs: Will retain 8,234 orphaned references

**This operation is IRREVERSIBLE without a backup.**

Do you want to proceed? Please confirm with the exact count: "yes, delete 8234 rows"
```

### For Critical Operations

#### DROP TABLE

```markdown
⛔ **CRITICAL: DROP TABLE REQUESTED**

**Command:** DROP TABLE legacy_users

**Current state:**
- Table exists: ✓
- Row count: 15,234
- Size on disk: 45.2 MB
- Foreign key references: 3 tables
- Last backup: 2024-01-15 09:00:00 (2 days ago)

**This will permanently delete:**
- 15,234 records
- All indexes (4)
- All constraints (7)
- All triggers (2)

**Cascade impact:**
- orders.user_id: 23,456 rows will have NULL user_id (SET NULL)
- comments.author_id: 8,901 rows will be deleted (CASCADE)

**Before proceeding, I will:**
1. Create a backup of this table
2. Export data to JSON/CSV for recovery
3. Document the table schema for recreation if needed

Proceed with backup and drop? Type "yes, drop legacy_users permanently" to confirm.
```

#### DROP DATABASE

```markdown
⛔⛔⛔ **EXTREME DANGER: DROP DATABASE REQUESTED**

**Command:** DROP DATABASE myapp_production

**⚠️ THIS WILL DESTROY THE ENTIRE DATABASE ⚠️**

**Database statistics:**
- Tables: 47
- Total rows: 1,234,567
- Size: 2.3 GB
- Created: 2022-03-15
- Last backup: UNKNOWN ⚠️

**I will NOT execute this command unless:**
1. You confirm this is NOT a production database
2. You have a verified backup
3. You provide the exact command with database name

This is almost never the right approach. Consider alternatives:
- Drop specific tables instead
- Use a fresh database for testing
- Restore from backup to a different database

If you truly need this, type: "I confirm DROP DATABASE myapp_production is intentional and I have backups"
```

#### TRUNCATE TABLE

```markdown
⛔ **CRITICAL: TRUNCATE TABLE REQUESTED**

**Command:** TRUNCATE TABLE orders

**What TRUNCATE does:**
- Removes ALL rows instantly (no WHERE clause possible)
- Resets auto-increment counters
- Cannot be rolled back in most databases
- Faster than DELETE but more dangerous

**Current state:**
- Row count: 89,234
- Estimated recovery time: Not possible without backup

**Alternative if you want to delete specific rows:**
```sql
DELETE FROM orders WHERE created_at < '2023-01-01';
```

Confirm truncation by typing: "yes, truncate orders and delete 89234 rows"
```

## DELETE Safety

### DELETE Without WHERE Clause

**NEVER execute without warning:**

```markdown
🔴 **DANGEROUS: DELETE WITHOUT WHERE CLAUSE**

**Command:** DELETE FROM users

**This will delete ALL 45,678 rows in the users table.**

This is almost always a mistake. Did you mean:
- DELETE FROM users WHERE status = 'deleted'
- TRUNCATE TABLE users (if you really want all rows gone)
- DELETE FROM users WHERE id = specific_id

If you truly want to delete all rows, use TRUNCATE instead (it's faster and more explicit about intent).

I will NOT execute DELETE without WHERE. Please provide a WHERE clause or confirm TRUNCATE.
```

### DELETE With WHERE Clause

```markdown
## DELETE Operation Preview

**Command:** DELETE FROM users WHERE status = 'inactive' AND last_login < '2023-01-01'

**Rows to be deleted:** 1,234

**Sample of rows that will be deleted:**
| id | email | status | last_login |
|----|-------|--------|------------|
| 456 | user1@example.com | inactive | 2022-05-15 |
| 789 | user2@example.com | inactive | 2022-11-20 |
| ... | ... | ... | ... |

**Verification query (run this first):**
```sql
SELECT COUNT(*), MIN(last_login), MAX(last_login)
FROM users
WHERE status = 'inactive' AND last_login < '2023-01-01';
```

Proceed with deletion? Type "yes, delete 1234 inactive users" to confirm.
```

## UPDATE Safety

### UPDATE Without WHERE Clause

```markdown
🔴 **DANGEROUS: UPDATE WITHOUT WHERE CLAUSE**

**Command:** UPDATE users SET role = 'user'

**This will update ALL 45,678 rows in the users table.**

This will:
- Set ALL users to role 'user'
- Including admin accounts (currently 23)
- Including system accounts (currently 5)

**You may have meant:**
```sql
UPDATE users SET role = 'user' WHERE role IS NULL;
UPDATE users SET role = 'user' WHERE id = specific_id;
```

I will NOT execute UPDATE without WHERE. Please provide a WHERE clause.
```

### UPDATE With WHERE Clause

```markdown
## UPDATE Operation Preview

**Command:** UPDATE products SET price = price * 1.1 WHERE category = 'electronics'

**Rows to be updated:** 234

**Sample of changes:**
| id | name | current_price | new_price |
|----|------|---------------|-----------|
| 101 | iPhone | 999.00 | 1098.90 |
| 102 | MacBook | 1299.00 | 1428.90 |
| ... | ... | ... | ... |

**Total price impact:** $23,456.00 increase across all products

Proceed with update? Type "yes, update 234 products" to confirm.
```

## Batch Operations Safety

For operations affecting many rows:

```typescript
// Safe batch deletion pattern
async function safeBatchDelete(table: string, condition: string, batchSize: number = 1000) {
  // Get total count first
  const total = await db.query(`SELECT COUNT(*) FROM ${table} WHERE ${condition}`);

  console.log(`Will delete ${total} rows in batches of ${batchSize}`);
  console.log('Press Ctrl+C within 10 seconds to abort...');
  await sleep(10000);

  let deleted = 0;
  while (deleted < total) {
    const result = await db.query(`
      DELETE FROM ${table}
      WHERE ${condition}
      LIMIT ${batchSize}
    `);

    deleted += result.rowCount;
    console.log(`Deleted ${deleted}/${total} rows`);

    // Allow interruption between batches
    await sleep(100);
  }

  return deleted;
}
```

## Transaction Safety

```sql
-- ALWAYS use transactions for multi-step operations
BEGIN;

-- Your operations here
DELETE FROM order_items WHERE order_id = 123;
DELETE FROM orders WHERE id = 123;

-- Preview the result before committing
SELECT * FROM orders WHERE id = 123; -- Should be empty

-- If everything looks good:
COMMIT;

-- If something went wrong:
ROLLBACK;
```

## Environment Protection

### Production Safeguards

```typescript
// Add to your application
function isDangerousQuery(sql: string): boolean {
  const dangerous = [
    /DROP\s+(TABLE|DATABASE)/i,
    /TRUNCATE/i,
    /DELETE\s+FROM\s+\w+\s*$/i,  // DELETE without WHERE
    /UPDATE\s+\w+\s+SET\s+.*$/i, // UPDATE without WHERE
  ];

  return dangerous.some(pattern => pattern.test(sql));
}

function executeQuery(sql: string) {
  if (process.env.NODE_ENV === 'production' && isDangerousQuery(sql)) {
    throw new Error(`Dangerous query blocked in production: ${sql.substring(0, 50)}...`);
  }

  return db.query(sql);
}
```

### Read Replica Routing

```typescript
// Route dangerous queries awareness
function getConnection(sql: string) {
  const isWrite = /INSERT|UPDATE|DELETE|DROP|TRUNCATE|ALTER/i.test(sql);

  if (isWrite) {
    console.log('⚠️ Write operation detected, using primary database');
    return primaryConnection;
  }

  return replicaConnection;
}
```

## Backup Before Dangerous Operations

```bash
# Quick backup commands

# PostgreSQL - specific table
pg_dump -t table_name database > table_backup_$(date +%Y%m%d_%H%M%S).sql

# MySQL - specific table
mysqldump database table_name > table_backup_$(date +%Y%m%d_%H%M%S).sql

# Full database backup
pg_dump -Fc database > full_backup_$(date +%Y%m%d_%H%M%S).dump
```

## Verification Queries

Always run these BEFORE destructive operations:

```sql
-- Verify DELETE scope
EXPLAIN DELETE FROM users WHERE condition;
SELECT COUNT(*) FROM users WHERE condition;

-- Verify UPDATE scope
SELECT * FROM table WHERE condition LIMIT 10;
SELECT COUNT(*) FROM table WHERE condition;

-- Verify no cascade surprises
SELECT
    tc.table_name,
    tc.constraint_name,
    rc.delete_rule
FROM information_schema.referential_constraints rc
JOIN information_schema.table_constraints tc
    ON rc.constraint_name = tc.constraint_name
WHERE rc.delete_rule = 'CASCADE';
```

## Best Practices Summary

| DO | DON'T |
|----|-------|
| Always use WHERE clause | Run DELETE/UPDATE without WHERE |
| Preview with SELECT first | Trust row counts blindly |
| Use transactions | Commit automatically |
| Backup before destructive ops | Assume you can recover |
| Verify environment | Run production commands casually |
| Use batch operations | Delete millions in one query |
| Check cascade effects | Ignore foreign key relationships |
