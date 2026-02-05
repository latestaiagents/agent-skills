---
name: access-control-audit
description: |
  OWASP A05 - Broken Access Control Detection. Use this skill when implementing
  authorization, checking permissions, or auditing who can access what resources.
  Activate when: authorization, permissions, access control, RBAC, ABAC, admin access,
  privilege escalation, IDOR, direct object reference, role check, can user access.
---

# Access Control Audit (OWASP A05)

**Detect and fix broken access control vulnerabilities including IDOR, privilege escalation, and missing authorization checks.**

## When to Use

- Implementing authorization logic
- Auditing API endpoint permissions
- Reviewing admin functionality
- Checking resource ownership
- Implementing role-based access
- Preventing privilege escalation

## Common Vulnerabilities

| Vulnerability | Risk | Example |
|--------------|------|---------|
| IDOR | HIGH | `/api/users/123` accessible by any user |
| Missing Auth Check | CRITICAL | Admin endpoints without verification |
| Privilege Escalation | CRITICAL | User can elevate to admin |
| Path Traversal | HIGH | `../../admin/config` |
| Forced Browsing | MEDIUM | Guessing admin URLs |
| Metadata Manipulation | HIGH | Changing userId in JWT |

## Detection Patterns

### Missing Authorization Checks

```javascript
// VULNERABLE - No ownership check
app.get('/api/documents/:id', async (req, res) => {
  const doc = await Document.findById(req.params.id);
  res.json(doc); // Anyone can access any document!
});

// VULNERABLE - No role check on admin route
app.delete('/api/users/:id', async (req, res) => {
  await User.findByIdAndDelete(req.params.id);
  res.json({ success: true }); // Any user can delete anyone!
});

// VULNERABLE - Client-side only checks
// Frontend hides admin button, but API has no check
if (user.role === 'admin') {
  showAdminButton();
}
```

### IDOR (Insecure Direct Object Reference)

```javascript
// VULNERABLE - Sequential IDs exposed
app.get('/api/invoices/:id', (req, res) => {
  // User can increment ID to see other invoices
  const invoice = await Invoice.findById(req.params.id);
  res.json(invoice);
});

// VULNERABLE - User ID from request body
app.post('/api/profile/update', (req, res) => {
  // Attacker can change userId to modify others
  await User.findByIdAndUpdate(req.body.userId, req.body.data);
});
```

### Privilege Escalation

```javascript
// VULNERABLE - Role from user input
app.post('/api/users', (req, res) => {
  const user = new User({
    email: req.body.email,
    role: req.body.role // Attacker sets role: 'admin'
  });
});

// VULNERABLE - Mass assignment
app.put('/api/profile', (req, res) => {
  // Attacker adds { isAdmin: true } to request
  await User.findByIdAndUpdate(userId, req.body);
});
```

## Secure Implementation

### 1. Authorization Middleware

```javascript
// Authorization middleware factory
function authorize(...allowedRoles) {
  return (req, res, next) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Authentication required' });
    }

    if (!allowedRoles.includes(req.user.role)) {
      // Log authorization failure
      logger.warn('Authorization failed', {
        userId: req.user.id,
        requiredRoles: allowedRoles,
        userRole: req.user.role,
        path: req.path
      });
      return res.status(403).json({ error: 'Insufficient permissions' });
    }

    next();
  };
}

// Usage
app.delete('/api/users/:id',
  authenticate,
  authorize('admin'),
  deleteUserHandler
);

app.get('/api/reports',
  authenticate,
  authorize('admin', 'manager'),
  getReportsHandler
);
```

### 2. Resource Ownership Verification

```javascript
// Ownership check middleware
function verifyOwnership(resourceModel, paramName = 'id') {
  return async (req, res, next) => {
    const resourceId = req.params[paramName];
    const resource = await resourceModel.findById(resourceId);

    if (!resource) {
      return res.status(404).json({ error: 'Resource not found' });
    }

    // Check ownership (adjust field name as needed)
    const isOwner = resource.userId?.toString() === req.user.id;
    const isAdmin = req.user.role === 'admin';

    if (!isOwner && !isAdmin) {
      logger.warn('Ownership check failed', {
        userId: req.user.id,
        resourceId,
        resourceOwner: resource.userId
      });
      return res.status(403).json({ error: 'Access denied' });
    }

    req.resource = resource;
    next();
  };
}

// Usage
app.get('/api/documents/:id',
  authenticate,
  verifyOwnership(Document),
  (req, res) => res.json(req.resource)
);

app.put('/api/documents/:id',
  authenticate,
  verifyOwnership(Document),
  updateDocumentHandler
);
```

### 3. Attribute-Based Access Control (ABAC)

```javascript
const { Ability, AbilityBuilder } = require('@casl/ability');

function defineAbilitiesFor(user) {
  const { can, cannot, build } = new AbilityBuilder(Ability);

  if (user.role === 'admin') {
    can('manage', 'all'); // Admin can do everything
  } else if (user.role === 'manager') {
    can('read', 'all');
    can('update', 'Document', { departmentId: user.departmentId });
    can('create', 'Document');
    cannot('delete', 'Document');
  } else {
    // Regular user
    can('read', 'Document', { userId: user.id });
    can('update', 'Document', { userId: user.id });
    can('create', 'Document');
    can('read', 'Profile', { userId: user.id });
    can('update', 'Profile', { userId: user.id });
  }

  return build();
}

// Middleware
function checkAbility(action, subject) {
  return (req, res, next) => {
    const ability = defineAbilitiesFor(req.user);

    if (ability.can(action, subject)) {
      next();
    } else {
      res.status(403).json({ error: 'Forbidden' });
    }
  };
}

// Usage
app.delete('/api/documents/:id',
  authenticate,
  checkAbility('delete', 'Document'),
  deleteHandler
);
```

### 4. Prevent Mass Assignment

```javascript
// Whitelist allowed fields
const ALLOWED_PROFILE_FIELDS = ['name', 'email', 'avatar', 'bio'];
const ALLOWED_ADMIN_FIELDS = [...ALLOWED_PROFILE_FIELDS, 'role', 'isActive'];

function filterFields(data, allowedFields) {
  return Object.keys(data)
    .filter(key => allowedFields.includes(key))
    .reduce((obj, key) => {
      obj[key] = data[key];
      return obj;
    }, {});
}

app.put('/api/profile', authenticate, async (req, res) => {
  const allowedFields = req.user.role === 'admin'
    ? ALLOWED_ADMIN_FIELDS
    : ALLOWED_PROFILE_FIELDS;

  const safeData = filterFields(req.body, allowedFields);

  await User.findByIdAndUpdate(req.user.id, safeData);
  res.json({ success: true });
});
```

### 5. Use UUIDs Instead of Sequential IDs

```javascript
const { v4: uuidv4 } = require('uuid');

// Mongoose schema with UUID
const documentSchema = new mongoose.Schema({
  _id: {
    type: String,
    default: uuidv4
  },
  title: String,
  userId: String
});

// Or use mongoose-uuid
const mongoose = require('mongoose');
require('mongoose-uuid')(mongoose);
```

### 6. API-Level Access Control Matrix

```javascript
const accessMatrix = {
  'GET /api/users': ['admin'],
  'GET /api/users/:id': ['admin', 'self'],
  'PUT /api/users/:id': ['admin', 'self'],
  'DELETE /api/users/:id': ['admin'],
  'GET /api/documents': ['admin', 'user'],
  'GET /api/documents/:id': ['admin', 'owner'],
  'POST /api/documents': ['admin', 'user'],
  'PUT /api/documents/:id': ['admin', 'owner'],
  'DELETE /api/documents/:id': ['admin', 'owner'],
  'GET /api/admin/*': ['admin'],
  'POST /api/admin/*': ['admin']
};

function checkAccess(req, resourceOwnerId = null) {
  const route = `${req.method} ${req.route.path}`;
  const allowedRoles = accessMatrix[route] || [];

  if (allowedRoles.includes(req.user.role)) {
    return true;
  }

  if (allowedRoles.includes('self') && req.params.id === req.user.id) {
    return true;
  }

  if (allowedRoles.includes('owner') && resourceOwnerId === req.user.id) {
    return true;
  }

  return false;
}
```

## Code Review Checklist

- [ ] All API endpoints have authorization checks
- [ ] Resource ownership verified before access
- [ ] No sequential/guessable IDs for sensitive resources
- [ ] Admin functions require admin role verification
- [ ] Role changes require elevated privileges
- [ ] Mass assignment prevented with field whitelisting
- [ ] Authorization logic is server-side (not just frontend)
- [ ] Failed access attempts are logged
- [ ] Default deny - explicitly grant access
- [ ] Horizontal access control (user A can't access user B's data)
- [ ] Vertical access control (user can't access admin functions)

## Testing for Access Control Issues

```bash
# Test IDOR - Try accessing other users' resources
curl -H "Authorization: Bearer $USER_A_TOKEN" \
  http://api.com/users/USER_B_ID

# Test privilege escalation
curl -X PUT http://api.com/profile \
  -H "Authorization: Bearer $USER_TOKEN" \
  -d '{"role": "admin"}'

# Test missing auth
curl http://api.com/admin/users

# Test forced browsing
curl http://api.com/admin
curl http://api.com/backup
curl http://api.com/config

# Automated testing
# Use Burp Suite's Authorize extension
# OWASP ZAP's Access Control Testing
```

## Best Practices

1. **Deny by Default**: Explicitly grant access, deny everything else
2. **Server-Side Checks**: Never trust client-side authorization
3. **Log Access Failures**: Monitor for attack patterns
4. **Use UUIDs**: Avoid predictable resource identifiers
5. **Separate Admin Routes**: Different authentication for admin
6. **Audit Regularly**: Review who can access what
7. **Principle of Least Privilege**: Grant minimum necessary access
