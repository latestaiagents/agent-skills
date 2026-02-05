---
name: broken-auth-detector
description: |
  OWASP A02 - Broken Authentication Detection. Use this skill when reviewing login systems,
  session management, password handling, or authentication flows. Activate when: login,
  authentication, password, session, token, JWT, OAuth, credentials, sign in, logout,
  remember me, forgot password, password reset, MFA, 2FA.
---

# Broken Authentication Detector (OWASP A02)

**Identify and fix authentication vulnerabilities including weak passwords, session hijacking, and credential stuffing.**

## When to Use

- Reviewing login/signup implementations
- Auditing session management
- Implementing password reset flows
- Adding OAuth/SSO integration
- Setting up JWT authentication
- Implementing MFA/2FA

## Common Vulnerabilities

| Vulnerability | Risk | Impact |
|--------------|------|--------|
| Weak password policy | HIGH | Easy brute force |
| No rate limiting | HIGH | Credential stuffing |
| Session fixation | HIGH | Account takeover |
| Insecure password storage | CRITICAL | Mass credential theft |
| Missing MFA | MEDIUM | Single point of failure |
| Predictable tokens | CRITICAL | Session hijacking |

## Detection Patterns

### Weak Password Handling

```javascript
// VULNERABLE - Plain text password storage
user.password = req.body.password;
await user.save();

// VULNERABLE - Weak hashing
const hash = md5(password);
const hash = sha1(password);

// VULNERABLE - No salt
const hash = bcrypt.hashSync(password); // Missing salt rounds

// VULNERABLE - Weak password policy
if (password.length >= 4) { // Too short!
  // accept password
}
```

### Session Vulnerabilities

```javascript
// VULNERABLE - Predictable session ID
const sessionId = `session_${Date.now()}`;
const sessionId = `user_${userId}`;

// VULNERABLE - Session in URL
res.redirect(`/dashboard?sessionId=${sessionId}`);

// VULNERABLE - No session expiry
req.session.cookie.maxAge = null; // Never expires

// VULNERABLE - Session not invalidated on logout
app.post('/logout', (req, res) => {
  res.redirect('/login'); // Session still valid!
});
```

### JWT Vulnerabilities

```javascript
// VULNERABLE - No algorithm verification
const decoded = jwt.decode(token); // Doesn't verify!

// VULNERABLE - Weak secret
const token = jwt.sign(payload, 'secret123');

// VULNERABLE - Algorithm confusion
const decoded = jwt.verify(token, secret); // Accepts 'none' algorithm

// VULNERABLE - No expiration
const token = jwt.sign({ userId }, secret); // No exp claim
```

## Secure Implementation

### 1. Password Hashing

```javascript
const bcrypt = require('bcrypt');
const SALT_ROUNDS = 12;

// Hash password
async function hashPassword(password) {
  return bcrypt.hash(password, SALT_ROUNDS);
}

// Verify password
async function verifyPassword(password, hash) {
  return bcrypt.compare(password, hash);
}

// Usage
const hashedPassword = await hashPassword(req.body.password);
user.password = hashedPassword;
await user.save();
```

```python
# Python with argon2 (recommended)
from argon2 import PasswordHasher

ph = PasswordHasher()

# Hash
hashed = ph.hash(password)

# Verify
try:
    ph.verify(hashed, password)
except VerifyMismatchError:
    raise InvalidCredentials()
```

### 2. Strong Password Policy

```javascript
const passwordPolicy = {
  minLength: 12,
  requireUppercase: true,
  requireLowercase: true,
  requireNumbers: true,
  requireSpecial: true,
  preventCommon: true,
  preventUserInfo: true
};

function validatePassword(password, userInfo = {}) {
  const errors = [];

  if (password.length < 12) {
    errors.push('Password must be at least 12 characters');
  }

  if (!/[A-Z]/.test(password)) {
    errors.push('Password must contain uppercase letter');
  }

  if (!/[a-z]/.test(password)) {
    errors.push('Password must contain lowercase letter');
  }

  if (!/[0-9]/.test(password)) {
    errors.push('Password must contain a number');
  }

  if (!/[!@#$%^&*(),.?":{}|<>]/.test(password)) {
    errors.push('Password must contain a special character');
  }

  // Check against common passwords
  if (COMMON_PASSWORDS.includes(password.toLowerCase())) {
    errors.push('Password is too common');
  }

  // Check if contains user info
  const userValues = Object.values(userInfo).map(v => v?.toLowerCase());
  if (userValues.some(v => password.toLowerCase().includes(v))) {
    errors.push('Password cannot contain personal information');
  }

  return { valid: errors.length === 0, errors };
}
```

### 3. Secure Session Management

```javascript
const session = require('express-session');
const RedisStore = require('connect-redis').default;

app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET, // Strong, random secret
  name: '__Host-sessionId', // Secure cookie prefix
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: true,        // HTTPS only
    httpOnly: true,      // No JavaScript access
    sameSite: 'strict',  // CSRF protection
    maxAge: 3600000,     // 1 hour
    domain: 'example.com',
    path: '/'
  },
  genid: () => crypto.randomUUID() // Cryptographically random
}));

// Regenerate session on login
app.post('/login', async (req, res) => {
  const user = await authenticateUser(req.body);

  // Regenerate session ID to prevent fixation
  req.session.regenerate((err) => {
    req.session.userId = user.id;
    req.session.loginTime = Date.now();
    res.redirect('/dashboard');
  });
});

// Destroy session on logout
app.post('/logout', (req, res) => {
  req.session.destroy((err) => {
    res.clearCookie('__Host-sessionId');
    res.redirect('/login');
  });
});
```

### 4. Secure JWT Implementation

```javascript
const jwt = require('jsonwebtoken');

const JWT_CONFIG = {
  secret: process.env.JWT_SECRET, // 256+ bit random string
  algorithm: 'HS256',
  expiresIn: '15m',      // Short-lived access tokens
  issuer: 'your-app',
  audience: 'your-app-users'
};

// Generate token
function generateAccessToken(user) {
  return jwt.sign(
    {
      sub: user.id,
      email: user.email,
      role: user.role
    },
    JWT_CONFIG.secret,
    {
      algorithm: JWT_CONFIG.algorithm,
      expiresIn: JWT_CONFIG.expiresIn,
      issuer: JWT_CONFIG.issuer,
      audience: JWT_CONFIG.audience
    }
  );
}

// Verify token (with algorithm enforcement)
function verifyToken(token) {
  return jwt.verify(token, JWT_CONFIG.secret, {
    algorithms: [JWT_CONFIG.algorithm], // IMPORTANT: Whitelist algorithms
    issuer: JWT_CONFIG.issuer,
    audience: JWT_CONFIG.audience
  });
}

// Refresh token (longer-lived, stored securely)
function generateRefreshToken(user) {
  const token = crypto.randomBytes(64).toString('hex');
  // Store in database with user association and expiry
  return token;
}
```

### 5. Rate Limiting

```javascript
const rateLimit = require('express-rate-limit');

// Login rate limiting
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // 5 attempts per window
  message: 'Too many login attempts, please try again later',
  standardHeaders: true,
  legacyHeaders: false,
  keyGenerator: (req) => req.body.email || req.ip, // Rate limit by email
  handler: (req, res) => {
    // Log failed attempts for monitoring
    logger.warn('Rate limit exceeded', {
      ip: req.ip,
      email: req.body.email
    });
    res.status(429).json({ error: 'Too many attempts' });
  }
});

app.post('/login', loginLimiter, loginHandler);

// Account lockout after repeated failures
async function handleFailedLogin(email) {
  const key = `failed_login:${email}`;
  const attempts = await redis.incr(key);
  await redis.expire(key, 900); // 15 minutes

  if (attempts >= 5) {
    await lockAccount(email);
    await sendLockoutNotification(email);
  }
}
```

### 6. Multi-Factor Authentication

```javascript
const speakeasy = require('speakeasy');
const qrcode = require('qrcode');

// Setup MFA
async function setupMFA(user) {
  const secret = speakeasy.generateSecret({
    name: `YourApp:${user.email}`,
    length: 32
  });

  // Store secret (encrypted) in database
  user.mfaSecret = encrypt(secret.base32);
  user.mfaEnabled = false; // Enable after verification
  await user.save();

  // Generate QR code
  const qrCodeUrl = await qrcode.toDataURL(secret.otpauth_url);

  return { qrCodeUrl, manualKey: secret.base32 };
}

// Verify MFA token
function verifyMFA(user, token) {
  const secret = decrypt(user.mfaSecret);

  return speakeasy.totp.verify({
    secret: secret,
    encoding: 'base32',
    token: token,
    window: 1 // Allow 1 step tolerance
  });
}
```

## Code Review Checklist

- [ ] Passwords hashed with bcrypt/argon2 (cost factor >= 12)
- [ ] Session IDs are cryptographically random
- [ ] Sessions regenerated after login
- [ ] Sessions destroyed on logout
- [ ] Cookies have Secure, HttpOnly, SameSite flags
- [ ] JWT algorithms explicitly whitelisted
- [ ] JWT tokens have short expiration
- [ ] Rate limiting on authentication endpoints
- [ ] Account lockout after failed attempts
- [ ] Password reset tokens are single-use and expire
- [ ] MFA option available for users
- [ ] No credentials in URLs or logs

## Best Practices

1. **Use established libraries**: Don't roll your own crypto
2. **Implement MFA**: Especially for sensitive accounts
3. **Monitor authentication**: Log and alert on anomalies
4. **Secure password reset**: Time-limited, single-use tokens
5. **Credential stuffing protection**: Rate limiting + CAPTCHA
6. **Session timeout**: Both idle and absolute timeouts
