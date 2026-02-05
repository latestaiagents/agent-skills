---
name: api-security
description: |
  Comprehensive API security for REST and GraphQL APIs. Use this skill when building
  or reviewing API endpoints, implementing authentication, or securing data transfer.
  Activate when: API security, REST security, GraphQL security, API authentication,
  API rate limiting, API versioning, secure endpoint, API design.
---

# API Security

**Secure your REST and GraphQL APIs against common attacks and vulnerabilities.**

## When to Use

- Designing new API endpoints
- Implementing API authentication
- Setting up rate limiting
- Reviewing API security
- Building public APIs
- Implementing webhooks

## API Security Checklist

| Area | Controls |
|------|----------|
| Authentication | OAuth 2.0, API keys, JWT |
| Authorization | Scopes, RBAC, resource ownership |
| Input Validation | Schema validation, type checking |
| Rate Limiting | Per-user, per-endpoint limits |
| Transport | HTTPS only, certificate pinning |
| Output | No sensitive data leakage |

## Authentication Patterns

### API Key Authentication

```javascript
// API key middleware
function apiKeyAuth(req, res, next) {
  const apiKey = req.headers['x-api-key'];

  if (!apiKey) {
    return res.status(401).json({ error: 'API key required' });
  }

  // Constant-time comparison to prevent timing attacks
  const validKey = await getApiKey(apiKey);
  if (!validKey || !crypto.timingSafeEqual(
    Buffer.from(apiKey),
    Buffer.from(validKey.key)
  )) {
    return res.status(401).json({ error: 'Invalid API key' });
  }

  req.apiClient = validKey.client;
  next();
}

// API key generation
function generateApiKey() {
  const prefix = 'sk_live_';  // Identifiable prefix
  const key = crypto.randomBytes(32).toString('hex');
  return prefix + key;
}

// Store hashed keys
async function createApiKey(clientId) {
  const key = generateApiKey();
  const hash = crypto.createHash('sha256').update(key).digest('hex');

  await db.apiKeys.create({
    clientId,
    keyHash: hash,
    keyPrefix: key.substring(0, 12),  // For identification
    createdAt: new Date()
  });

  return key;  // Only returned once
}
```

### OAuth 2.0 Implementation

```javascript
// OAuth 2.0 token endpoint
app.post('/oauth/token', async (req, res) => {
  const { grant_type, client_id, client_secret, code, refresh_token } = req.body;

  // Validate client
  const client = await validateClient(client_id, client_secret);
  if (!client) {
    return res.status(401).json({ error: 'invalid_client' });
  }

  switch (grant_type) {
    case 'authorization_code':
      return handleAuthorizationCode(req, res, client, code);
    case 'refresh_token':
      return handleRefreshToken(req, res, client, refresh_token);
    case 'client_credentials':
      return handleClientCredentials(req, res, client);
    default:
      return res.status(400).json({ error: 'unsupported_grant_type' });
  }
});

// Token response
function generateTokenResponse(user, client, scopes) {
  const accessToken = jwt.sign(
    { sub: user.id, client_id: client.id, scopes },
    process.env.JWT_SECRET,
    { expiresIn: '15m' }
  );

  const refreshToken = crypto.randomBytes(64).toString('hex');

  return {
    access_token: accessToken,
    token_type: 'Bearer',
    expires_in: 900,
    refresh_token: refreshToken,
    scope: scopes.join(' ')
  };
}
```

## Request Validation

### Schema Validation

```javascript
const Joi = require('joi');

// Define schemas
const schemas = {
  createUser: Joi.object({
    email: Joi.string().email().required(),
    name: Joi.string().min(1).max(100).required(),
    age: Joi.number().integer().min(0).max(150)
  }),

  updateUser: Joi.object({
    name: Joi.string().min(1).max(100),
    age: Joi.number().integer().min(0).max(150)
  }).min(1)  // At least one field required
};

// Validation middleware
function validate(schemaName) {
  return (req, res, next) => {
    const schema = schemas[schemaName];
    const { error, value } = schema.validate(req.body, {
      abortEarly: false,
      stripUnknown: true  // Remove unknown fields
    });

    if (error) {
      return res.status(400).json({
        error: 'Validation failed',
        details: error.details.map(d => ({
          field: d.path.join('.'),
          message: d.message
        }))
      });
    }

    req.validatedBody = value;
    next();
  };
}

// Usage
app.post('/users', validate('createUser'), createUserHandler);
```

### GraphQL Validation

```javascript
const { ApolloServer } = require('@apollo/server');
const depthLimit = require('graphql-depth-limit');
const { createComplexityLimitRule } = require('graphql-validation-complexity');

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [
    // Prevent deep queries
    depthLimit(5),
    // Prevent complex queries
    createComplexityLimitRule(1000)
  ],
  plugins: [
    // Rate limiting plugin
    {
      async requestDidStart() {
        return {
          async didResolveOperation(ctx) {
            await checkRateLimit(ctx.request);
          }
        };
      }
    }
  ]
});
```

## Rate Limiting

```javascript
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');

// Different limits for different endpoints
const rateLimits = {
  // Standard API endpoints
  standard: rateLimit({
    store: new RedisStore({ client: redis }),
    windowMs: 60 * 1000,  // 1 minute
    max: 100,
    keyGenerator: (req) => req.apiClient?.id || req.ip,
    standardHeaders: true,
    legacyHeaders: false
  }),

  // Authentication endpoints (stricter)
  auth: rateLimit({
    store: new RedisStore({ client: redis }),
    windowMs: 15 * 60 * 1000,  // 15 minutes
    max: 10,
    keyGenerator: (req) => req.ip,
    skipSuccessfulRequests: true  // Only count failures
  }),

  // Expensive operations
  expensive: rateLimit({
    store: new RedisStore({ client: redis }),
    windowMs: 60 * 60 * 1000,  // 1 hour
    max: 10,
    keyGenerator: (req) => req.apiClient?.id || req.ip
  })
};

// Apply to routes
app.use('/api/', rateLimits.standard);
app.use('/api/auth/', rateLimits.auth);
app.post('/api/export', rateLimits.expensive, exportHandler);
```

## Response Security

```javascript
// Secure response headers
app.use((req, res, next) => {
  // No caching for API responses
  res.setHeader('Cache-Control', 'no-store');
  res.setHeader('Pragma', 'no-cache');

  // Prevent MIME sniffing
  res.setHeader('X-Content-Type-Options', 'nosniff');

  next();
});

// Filter sensitive fields from responses
function sanitizeResponse(data, allowedFields) {
  if (Array.isArray(data)) {
    return data.map(item => sanitizeResponse(item, allowedFields));
  }

  if (typeof data === 'object' && data !== null) {
    return Object.keys(data)
      .filter(key => allowedFields.includes(key))
      .reduce((obj, key) => {
        obj[key] = data[key];
        return obj;
      }, {});
  }

  return data;
}

// Usage
app.get('/api/users/:id', async (req, res) => {
  const user = await User.findById(req.params.id);
  const safeUser = sanitizeResponse(user, ['id', 'name', 'email', 'createdAt']);
  res.json(safeUser);
});
```

## Error Handling

```javascript
// Secure error handler
app.use((err, req, res, next) => {
  // Log full error internally
  logger.error('API Error', {
    error: err.message,
    stack: err.stack,
    path: req.path,
    method: req.method,
    clientId: req.apiClient?.id
  });

  // Return safe error to client
  const statusCode = err.statusCode || 500;
  const response = {
    error: {
      code: err.code || 'INTERNAL_ERROR',
      message: statusCode === 500
        ? 'An internal error occurred'
        : err.message
    }
  };

  // Include request ID for support
  response.error.requestId = req.id;

  res.status(statusCode).json(response);
});
```

## Best Practices

1. **Always Use HTTPS**: Never transmit credentials over HTTP
2. **Validate All Input**: Schema validation on every endpoint
3. **Rate Limit Everything**: Protect against abuse
4. **Version Your API**: `/v1/`, `/v2/` for breaking changes
5. **Use Standard Auth**: OAuth 2.0 for third-party access
6. **Log Security Events**: Monitor for anomalies
7. **Don't Expose Stack Traces**: Generic error messages
8. **Implement Pagination**: Prevent data dumps
