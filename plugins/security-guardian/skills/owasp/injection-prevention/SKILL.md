---
name: injection-prevention
description: |
  OWASP A01 - Injection Prevention. Use this skill when reviewing code for SQL injection,
  NoSQL injection, command injection, LDAP injection, or any user input that reaches
  databases, shells, or interpreters. Activate when: SQL query, database query, user input,
  command execution, shell command, exec, eval, system call, parameterized query.
---

# Injection Prevention (OWASP A01)

**Prevent SQL, NoSQL, Command, and other injection attacks by validating and sanitizing all user input.**

## When to Use

- Reviewing code that builds SQL/NoSQL queries
- Code that executes shell commands
- Any place user input reaches an interpreter
- Building APIs that accept user data
- Migrating from string concatenation to parameterized queries

## Injection Types

| Type | Danger | Common Locations |
|------|--------|------------------|
| SQL Injection | CRITICAL | Database queries, ORMs with raw queries |
| NoSQL Injection | CRITICAL | MongoDB, Redis, Elasticsearch queries |
| Command Injection | CRITICAL | Shell exec, system calls, child_process |
| LDAP Injection | HIGH | Directory service queries |
| XPath Injection | HIGH | XML document queries |
| Expression Language | HIGH | Template engines, eval() |

## Detection Patterns

### SQL Injection Red Flags

```javascript
// VULNERABLE - String concatenation
const query = "SELECT * FROM users WHERE id = " + userId;
const query = `SELECT * FROM users WHERE name = '${userName}'`;

// VULNERABLE - Format strings
query = "SELECT * FROM users WHERE id = %s" % user_id

// VULNERABLE - String interpolation in ORM
User.where("name = '#{params[:name]}'")
```

### Command Injection Red Flags

```javascript
// VULNERABLE - Direct user input in commands
exec(`ls ${userInput}`);
system("ping " + ipAddress);
child_process.exec(`convert ${filename} output.png`);

// VULNERABLE - eval with user data
eval(userCode);
new Function(userInput)();
```

## Prevention Techniques

### 1. Parameterized Queries (SQL)

```javascript
// SAFE - Node.js with parameterized query
const result = await db.query(
  'SELECT * FROM users WHERE id = $1 AND status = $2',
  [userId, status]
);

// SAFE - Using ORM properly
const user = await User.findOne({ where: { id: userId } });

// SAFE - Prepared statements
const stmt = db.prepare('SELECT * FROM users WHERE email = ?');
const user = stmt.get(email);
```

```python
# SAFE - Python with parameterized query
cursor.execute(
    "SELECT * FROM users WHERE id = %s AND status = %s",
    (user_id, status)
)

# SAFE - SQLAlchemy ORM
user = session.query(User).filter(User.id == user_id).first()
```

```php
// SAFE - PHP PDO with prepared statements
$stmt = $pdo->prepare('SELECT * FROM users WHERE id = :id');
$stmt->execute(['id' => $userId]);
```

### 2. NoSQL Injection Prevention

```javascript
// VULNERABLE - MongoDB with user object
db.users.find({ username: req.body.username, password: req.body.password });
// Attack: { "username": "admin", "password": { "$ne": "" } }

// SAFE - Validate and sanitize input
const username = String(req.body.username).slice(0, 50);
const password = String(req.body.password);

// SAFE - Use mongoose with schema validation
const userSchema = new Schema({
  username: { type: String, required: true, maxlength: 50 },
  password: { type: String, required: true }
});
```

### 3. Command Injection Prevention

```javascript
// VULNERABLE
exec(`convert ${filename} output.png`);

// SAFE - Use array arguments (no shell)
execFile('convert', [filename, 'output.png']);

// SAFE - Whitelist allowed values
const ALLOWED_FORMATS = ['png', 'jpg', 'gif'];
if (!ALLOWED_FORMATS.includes(format)) {
  throw new Error('Invalid format');
}

// SAFE - Use library instead of shell
const sharp = require('sharp');
await sharp(inputFile).toFile(outputFile);
```

```python
# VULNERABLE
os.system(f"ping {ip_address}")

# SAFE - Use subprocess with list arguments
import subprocess
subprocess.run(['ping', '-c', '4', ip_address], check=True)

# SAFE - Use library instead of shell
import socket
socket.gethostbyname(hostname)
```

### 4. Input Validation

```javascript
// Validation helper
function validateInput(input, options = {}) {
  const { maxLength = 255, pattern, allowedValues } = options;

  // Type check
  if (typeof input !== 'string') {
    throw new ValidationError('Input must be a string');
  }

  // Length check
  if (input.length > maxLength) {
    throw new ValidationError(`Input exceeds ${maxLength} characters`);
  }

  // Pattern check
  if (pattern && !pattern.test(input)) {
    throw new ValidationError('Input contains invalid characters');
  }

  // Whitelist check
  if (allowedValues && !allowedValues.includes(input)) {
    throw new ValidationError('Input value not allowed');
  }

  return input;
}

// Usage
const userId = validateInput(req.params.id, {
  maxLength: 36,
  pattern: /^[a-f0-9-]+$/i  // UUID pattern
});
```

## Code Review Checklist

### Before Approving Code, Verify:

- [ ] No string concatenation in SQL queries
- [ ] All database queries use parameterized statements
- [ ] User input is validated before use
- [ ] Shell commands use array arguments, not string interpolation
- [ ] No eval(), new Function(), or similar with user data
- [ ] Input length limits are enforced
- [ ] Type checking is performed on all inputs
- [ ] Error messages don't expose query structure

## Testing for Injection

### Manual Test Payloads

```bash
# SQL Injection tests
' OR '1'='1
'; DROP TABLE users; --
' UNION SELECT * FROM passwords --
1; SELECT * FROM users

# NoSQL Injection tests
{"$gt": ""}
{"$ne": null}
{"$where": "sleep(5000)"}

# Command Injection tests
; ls -la
| cat /etc/passwd
`whoami`
$(id)
```

### Automated Tools

```bash
# SQLMap for SQL injection
sqlmap -u "http://target.com/page?id=1" --dbs

# NoSQLMap for NoSQL injection
nosqlmap -u "http://target.com/api/users"

# Commix for command injection
commix -u "http://target.com/ping?ip=127.0.0.1"
```

## Framework-Specific Guidance

### Express.js / Node.js
- Use `express-validator` for input validation
- Use Sequelize/Prisma ORM with proper escaping
- Never use `eval()` or `child_process.exec()` with user input

### Django / Python
- Use Django ORM (automatic parameterization)
- Use `subprocess.run()` with list arguments
- Enable `SECURE_CONTENT_TYPE_NOSNIFF`

### Rails / Ruby
- Use ActiveRecord properly (automatic parameterization)
- Avoid `where("column = '#{value}'")`
- Use `Shellwords.escape()` for shell arguments

### Laravel / PHP
- Use Eloquent ORM or Query Builder
- Use PDO prepared statements
- Use `escapeshellarg()` for shell arguments

## Best Practices

1. **Defense in Depth**: Combine input validation + parameterized queries + output encoding
2. **Least Privilege**: Database accounts should have minimum required permissions
3. **Whitelist Over Blacklist**: Define what's allowed, not what's forbidden
4. **Escape Output**: Even with safe queries, escape data when displaying
5. **Log Suspicious Input**: Monitor for injection attempts
6. **Regular Security Audits**: Use automated scanners in CI/CD pipeline
