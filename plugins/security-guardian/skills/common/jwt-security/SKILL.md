---
name: jwt-security
description: |
  JSON Web Token security best practices. Use this skill when implementing JWT
  authentication, validating tokens, or reviewing JWT usage. Activate when: JWT,
  JSON Web Token, token authentication, bearer token, refresh token, token validation,
  JWT secret, token expiry.
---

# JWT Security

**Secure implementation of JSON Web Tokens for authentication.**

## When to Use

- Implementing JWT authentication
- Reviewing existing JWT code
- Setting up refresh token rotation
- Debugging JWT issues
- Migrating to JWT-based auth

## JWT Vulnerabilities

| Vulnerability | Risk | Description |
|--------------|------|-------------|
| Algorithm None | CRITICAL | Accepting unsigned tokens |
| Algorithm Confusion | CRITICAL | RS256 → HS256 attack |
| Weak Secret | HIGH | Brute-forceable secrets |
| No Expiration | HIGH | Tokens valid forever |
| Sensitive Data in Payload | MEDIUM | JWT payload is base64, not encrypted |
| Token Leakage | HIGH | Exposed in logs/URLs |

## Secure Implementation

### 1. Token Generation

```javascript
const jwt = require('jsonwebtoken');

const JWT_CONFIG = {
  accessSecret: process.env.JWT_ACCESS_SECRET,   // 256+ bit random string
  refreshSecret: process.env.JWT_REFRESH_SECRET,
  accessExpiry: '15m',    // Short-lived
  refreshExpiry: '7d',    // Longer-lived
  algorithm: 'HS256',     // Or RS256 for asymmetric
  issuer: 'your-app-name',
  audience: 'your-app-users'
};

function generateAccessToken(user) {
  const payload = {
    sub: user.id,           // Subject (user ID)
    email: user.email,      // Only non-sensitive data
    role: user.role,
    // Don't include: password, SSN, credit card, etc.
  };

  return jwt.sign(payload, JWT_CONFIG.accessSecret, {
    algorithm: JWT_CONFIG.algorithm,
    expiresIn: JWT_CONFIG.accessExpiry,
    issuer: JWT_CONFIG.issuer,
    audience: JWT_CONFIG.audience,
    jwtid: crypto.randomUUID()  // Unique token ID
  });
}

function generateRefreshToken(user) {
  const payload = {
    sub: user.id,
    type: 'refresh',
    family: crypto.randomUUID()  // For refresh token rotation
  };

  return jwt.sign(payload, JWT_CONFIG.refreshSecret, {
    algorithm: JWT_CONFIG.algorithm,
    expiresIn: JWT_CONFIG.refreshExpiry,
    issuer: JWT_CONFIG.issuer
  });
}
```

### 2. Token Verification (CRITICAL)

```javascript
function verifyAccessToken(token) {
  try {
    // CRITICAL: Always specify allowed algorithms
    return jwt.verify(token, JWT_CONFIG.accessSecret, {
      algorithms: [JWT_CONFIG.algorithm],  // Whitelist!
      issuer: JWT_CONFIG.issuer,
      audience: JWT_CONFIG.audience,
      complete: true  // Returns header + payload
    });
  } catch (error) {
    if (error instanceof jwt.TokenExpiredError) {
      throw new AuthError('Token expired', 'TOKEN_EXPIRED');
    }
    if (error instanceof jwt.JsonWebTokenError) {
      throw new AuthError('Invalid token', 'INVALID_TOKEN');
    }
    throw error;
  }
}

// VULNERABLE - Never do this!
// jwt.verify(token, secret);  // Accepts any algorithm!
// jwt.decode(token);          // No verification at all!
```

### 3. Authentication Middleware

```javascript
async function authenticateJWT(req, res, next) {
  // Extract token from header
  const authHeader = req.headers.authorization;

  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'No token provided' });
  }

  const token = authHeader.substring(7);

  try {
    const decoded = verifyAccessToken(token);

    // Optional: Check if token is blacklisted
    if (await isTokenBlacklisted(decoded.payload.jti)) {
      return res.status(401).json({ error: 'Token revoked' });
    }

    // Attach user to request
    req.user = {
      id: decoded.payload.sub,
      email: decoded.payload.email,
      role: decoded.payload.role
    };

    next();
  } catch (error) {
    if (error.code === 'TOKEN_EXPIRED') {
      return res.status(401).json({
        error: 'Token expired',
        code: 'TOKEN_EXPIRED'
      });
    }
    return res.status(401).json({ error: 'Invalid token' });
  }
}
```

### 4. Refresh Token Rotation

```javascript
// Store refresh tokens in database
const refreshTokenSchema = {
  id: 'uuid',
  userId: 'uuid',
  tokenHash: 'string',      // Store hash, not token
  family: 'string',         // For rotation detection
  expiresAt: 'datetime',
  revokedAt: 'datetime?',
  replacedBy: 'uuid?'
};

async function refreshTokens(refreshToken) {
  // Verify refresh token
  let decoded;
  try {
    decoded = jwt.verify(refreshToken, JWT_CONFIG.refreshSecret, {
      algorithms: [JWT_CONFIG.algorithm]
    });
  } catch {
    throw new AuthError('Invalid refresh token');
  }

  // Find token in database
  const tokenHash = hashToken(refreshToken);
  const storedToken = await db.refreshTokens.findOne({
    tokenHash,
    revokedAt: null
  });

  if (!storedToken) {
    // Token not found or already revoked
    // Possible token reuse attack - revoke entire family
    await db.refreshTokens.updateMany(
      { family: decoded.family },
      { revokedAt: new Date() }
    );
    throw new AuthError('Refresh token reuse detected');
  }

  // Check expiration
  if (new Date() > storedToken.expiresAt) {
    throw new AuthError('Refresh token expired');
  }

  // Rotate: revoke old, create new
  const user = await db.users.findById(decoded.sub);
  const newAccessToken = generateAccessToken(user);
  const newRefreshToken = generateRefreshToken(user);

  // Revoke old refresh token
  await db.refreshTokens.update(storedToken.id, {
    revokedAt: new Date(),
    replacedBy: newRefreshToken.id
  });

  // Store new refresh token
  await db.refreshTokens.create({
    userId: user.id,
    tokenHash: hashToken(newRefreshToken),
    family: decoded.family,  // Same family for rotation tracking
    expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)
  });

  return {
    accessToken: newAccessToken,
    refreshToken: newRefreshToken
  };
}
```

### 5. Token Revocation

```javascript
// Blacklist for immediate revocation
const tokenBlacklist = new Map();  // In production, use Redis

async function revokeToken(token) {
  const decoded = jwt.decode(token);
  if (decoded && decoded.jti) {
    // Store until token would naturally expire
    const ttl = decoded.exp * 1000 - Date.now();
    await redis.setex(`blacklist:${decoded.jti}`, ttl / 1000, '1');
  }
}

async function isTokenBlacklisted(jti) {
  return await redis.exists(`blacklist:${jti}`);
}

// Logout endpoint
app.post('/logout', authenticateJWT, async (req, res) => {
  // Revoke access token
  await revokeToken(req.headers.authorization.substring(7));

  // Revoke all refresh tokens for user
  await db.refreshTokens.updateMany(
    { userId: req.user.id },
    { revokedAt: new Date() }
  );

  res.json({ success: true });
});
```

### 6. Asymmetric Keys (RS256)

```javascript
const fs = require('fs');

// For distributed systems or when verifier != issuer
const privateKey = fs.readFileSync('private.pem');
const publicKey = fs.readFileSync('public.pem');

function generateTokenRS256(user) {
  return jwt.sign(
    { sub: user.id },
    privateKey,
    {
      algorithm: 'RS256',
      expiresIn: '15m',
      issuer: 'auth-service'
    }
  );
}

function verifyTokenRS256(token) {
  return jwt.verify(token, publicKey, {
    algorithms: ['RS256'],  // CRITICAL: Only allow RS256
    issuer: 'auth-service'
  });
}
```

```bash
# Generate RSA key pair
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem
```

## Token Storage (Client-Side)

```javascript
// BEST: HttpOnly cookie (for web apps)
res.cookie('accessToken', token, {
  httpOnly: true,    // No JS access
  secure: true,      // HTTPS only
  sameSite: 'strict',
  maxAge: 900000     // 15 minutes
});

// OK: Memory (for SPAs, lost on refresh)
let accessToken = null;

function setToken(token) {
  accessToken = token;
}

// AVOID: localStorage (XSS vulnerable)
// localStorage.setItem('token', token);  // DON'T!
```

## Common Mistakes

```javascript
// MISTAKE 1: Not specifying algorithm
jwt.verify(token, secret);  // Vulnerable to algorithm switching!

// MISTAKE 2: Using decode instead of verify
const user = jwt.decode(token);  // No signature check!

// MISTAKE 3: Sensitive data in payload
jwt.sign({ password: user.password }, secret);  // NO!

// MISTAKE 4: Weak secret
jwt.sign(payload, 'secret');  // Easily brute-forced!

// MISTAKE 5: No expiration
jwt.sign(payload, secret);  // No exp = valid forever!

// MISTAKE 6: Token in URL
res.redirect(`/dashboard?token=${token}`);  // Logged everywhere!
```

## Security Checklist

- [ ] Algorithm explicitly whitelisted in verify()
- [ ] Strong secret (256+ bits of entropy)
- [ ] Short access token expiry (15 minutes)
- [ ] Refresh token rotation implemented
- [ ] No sensitive data in payload
- [ ] Tokens not stored in localStorage
- [ ] Tokens not passed in URLs
- [ ] Token revocation mechanism exists
- [ ] jti claim for unique identification
- [ ] Issuer and audience validated

## Best Practices

1. **Whitelist Algorithms**: Always specify allowed algorithms
2. **Short Expiry**: Access tokens should be short-lived
3. **Rotate Refresh Tokens**: New refresh token on each use
4. **Use HttpOnly Cookies**: For web applications
5. **Strong Secrets**: 256+ bit random secrets
6. **No Sensitive Data**: JWT payload is not encrypted
7. **Implement Revocation**: For logout and compromised tokens
8. **Validate Everything**: issuer, audience, expiry, signature
