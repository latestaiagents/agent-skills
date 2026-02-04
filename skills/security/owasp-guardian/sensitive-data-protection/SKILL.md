---
name: sensitive-data-protection
description: |
  OWASP A03 - Sensitive Data Exposure Prevention. Use this skill when handling PII,
  passwords, credit cards, API keys, or any sensitive information. Activate when:
  encryption, PII, personal data, credit card, SSN, password storage, HTTPS, TLS,
  data at rest, data in transit, GDPR, compliance, data masking.
---

# Sensitive Data Protection (OWASP A03)

**Protect sensitive data in transit and at rest through proper encryption, masking, and access controls.**

## When to Use

- Handling user personal information (PII)
- Storing or transmitting passwords
- Processing payment card data
- Managing API keys and secrets
- Implementing data encryption
- Ensuring compliance (GDPR, PCI-DSS, HIPAA)

## Sensitive Data Categories

| Category | Examples | Protection Level |
|----------|----------|------------------|
| Credentials | Passwords, API keys, tokens | CRITICAL |
| Financial | Credit cards, bank accounts | CRITICAL |
| Personal (PII) | SSN, passport, driver's license | HIGH |
| Health (PHI) | Medical records, prescriptions | HIGH |
| Contact | Email, phone, address | MEDIUM |
| Behavioral | Browsing history, preferences | MEDIUM |

## Detection Patterns

### Exposed Sensitive Data

```javascript
// VULNERABLE - Logging sensitive data
console.log('User login:', { email, password });
logger.info(`Payment processed: ${creditCardNumber}`);

// VULNERABLE - Sensitive data in URL
res.redirect(`/reset?token=${token}&email=${email}`);

// VULNERABLE - Sensitive data in error messages
throw new Error(`Invalid password for user ${email}`);

// VULNERABLE - Storing unencrypted
user.ssn = req.body.ssn;
user.creditCard = req.body.cardNumber;
```

### Weak Encryption

```javascript
// VULNERABLE - Weak algorithms
const encrypted = crypto.createCipher('des', key); // DES is broken
const hash = crypto.createHash('md5').update(data); // MD5 is broken

// VULNERABLE - ECB mode
crypto.createCipheriv('aes-256-ecb', key, ''); // ECB leaks patterns

// VULNERABLE - Hardcoded keys
const ENCRYPTION_KEY = 'my-secret-key-123';

// VULNERABLE - Predictable IV
const iv = Buffer.alloc(16, 0); // All zeros IV
```

## Secure Implementation

### 1. Encryption at Rest

```javascript
const crypto = require('crypto');

const ALGORITHM = 'aes-256-gcm';
const KEY = Buffer.from(process.env.ENCRYPTION_KEY, 'hex'); // 256-bit key

function encrypt(plaintext) {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv(ALGORITHM, KEY, iv);

  let encrypted = cipher.update(plaintext, 'utf8', 'hex');
  encrypted += cipher.final('hex');

  const authTag = cipher.getAuthTag();

  // Return IV + AuthTag + Ciphertext
  return iv.toString('hex') + ':' + authTag.toString('hex') + ':' + encrypted;
}

function decrypt(encryptedData) {
  const [ivHex, authTagHex, ciphertext] = encryptedData.split(':');

  const iv = Buffer.from(ivHex, 'hex');
  const authTag = Buffer.from(authTagHex, 'hex');

  const decipher = crypto.createDecipheriv(ALGORITHM, KEY, iv);
  decipher.setAuthTag(authTag);

  let decrypted = decipher.update(ciphertext, 'hex', 'utf8');
  decrypted += decipher.final('utf8');

  return decrypted;
}

// Usage
user.encryptedSSN = encrypt(ssn);
const ssn = decrypt(user.encryptedSSN);
```

### 2. Field-Level Encryption for Database

```javascript
const mongoose = require('mongoose');
const { encrypt, decrypt } = require('./encryption');

// Mongoose plugin for automatic encryption
function encryptedField(schema, options) {
  const { fields } = options;

  schema.pre('save', function(next) {
    fields.forEach(field => {
      if (this.isModified(field) && this[field]) {
        this[field] = encrypt(this[field]);
      }
    });
    next();
  });

  fields.forEach(field => {
    schema.methods[`getDecrypted${capitalize(field)}`] = function() {
      return this[field] ? decrypt(this[field]) : null;
    };
  });
}

// Usage
const userSchema = new mongoose.Schema({
  email: String,
  ssn: String,        // Will be encrypted
  taxId: String       // Will be encrypted
});

userSchema.plugin(encryptedField, {
  fields: ['ssn', 'taxId']
});
```

### 3. Data Masking

```javascript
// Mask credit card: **** **** **** 1234
function maskCreditCard(cardNumber) {
  const last4 = cardNumber.slice(-4);
  return `**** **** **** ${last4}`;
}

// Mask email: j***@example.com
function maskEmail(email) {
  const [local, domain] = email.split('@');
  const maskedLocal = local[0] + '*'.repeat(Math.max(local.length - 1, 2));
  return `${maskedLocal}@${domain}`;
}

// Mask phone: ***-***-5678
function maskPhone(phone) {
  const digits = phone.replace(/\D/g, '');
  const last4 = digits.slice(-4);
  return `***-***-${last4}`;
}

// Mask SSN: ***-**-6789
function maskSSN(ssn) {
  const digits = ssn.replace(/\D/g, '');
  const last4 = digits.slice(-4);
  return `***-**-${last4}`;
}

// Usage in API responses
function sanitizeUserForResponse(user) {
  return {
    id: user.id,
    email: maskEmail(user.email),
    phone: user.phone ? maskPhone(user.phone) : null,
    // Never include: password, ssn, full credit card
  };
}
```

### 4. HTTPS/TLS Configuration

```javascript
const https = require('https');
const fs = require('fs');

// Strong TLS configuration
const server = https.createServer({
  key: fs.readFileSync('private-key.pem'),
  cert: fs.readFileSync('certificate.pem'),

  // TLS 1.2+ only
  minVersion: 'TLSv1.2',

  // Strong cipher suites
  ciphers: [
    'ECDHE-ECDSA-AES128-GCM-SHA256',
    'ECDHE-RSA-AES128-GCM-SHA256',
    'ECDHE-ECDSA-AES256-GCM-SHA384',
    'ECDHE-RSA-AES256-GCM-SHA384'
  ].join(':'),

  // Prefer server cipher order
  honorCipherOrder: true
}, app);

// Force HTTPS with HSTS
app.use((req, res, next) => {
  res.setHeader(
    'Strict-Transport-Security',
    'max-age=31536000; includeSubDomains; preload'
  );
  next();
});
```

### 5. Secure Logging

```javascript
const sensitiveFields = [
  'password', 'token', 'secret', 'key', 'apiKey',
  'creditCard', 'cardNumber', 'cvv', 'ssn', 'taxId'
];

function sanitizeForLogging(obj, seen = new WeakSet()) {
  if (obj === null || typeof obj !== 'object') {
    return obj;
  }

  if (seen.has(obj)) {
    return '[Circular]';
  }
  seen.add(obj);

  if (Array.isArray(obj)) {
    return obj.map(item => sanitizeForLogging(item, seen));
  }

  const sanitized = {};
  for (const [key, value] of Object.entries(obj)) {
    if (sensitiveFields.some(f => key.toLowerCase().includes(f))) {
      sanitized[key] = '[REDACTED]';
    } else if (typeof value === 'object') {
      sanitized[key] = sanitizeForLogging(value, seen);
    } else {
      sanitized[key] = value;
    }
  }

  return sanitized;
}

// Custom logger that auto-sanitizes
const logger = {
  info: (message, data) => {
    console.log(message, sanitizeForLogging(data));
  },
  error: (message, data) => {
    console.error(message, sanitizeForLogging(data));
  }
};

// Usage
logger.info('User registered', { email, password });
// Output: User registered { email: 'user@example.com', password: '[REDACTED]' }
```

### 6. Key Management

```javascript
// Use environment variables (minimum)
const ENCRYPTION_KEY = process.env.ENCRYPTION_KEY;

// Better: Use a secrets manager
const { SecretManagerServiceClient } = require('@google-cloud/secret-manager');

async function getSecret(secretName) {
  const client = new SecretManagerServiceClient();
  const [version] = await client.accessSecretVersion({
    name: `projects/my-project/secrets/${secretName}/versions/latest`
  });
  return version.payload.data.toString();
}

// Best: Use a KMS for key encryption
const { KMSClient, DecryptCommand } = require('@aws-sdk/client-kms');

async function decryptDataKey(encryptedKey) {
  const client = new KMSClient({ region: 'us-east-1' });
  const command = new DecryptCommand({
    CiphertextBlob: Buffer.from(encryptedKey, 'base64'),
    KeyId: process.env.KMS_KEY_ID
  });
  const response = await client.send(command);
  return response.Plaintext;
}
```

## Data Classification Policy

```javascript
const DataClassification = {
  PUBLIC: {
    level: 0,
    encryption: false,
    logging: true,
    retention: 'indefinite'
  },
  INTERNAL: {
    level: 1,
    encryption: false,
    logging: true,
    retention: '7 years'
  },
  CONFIDENTIAL: {
    level: 2,
    encryption: true,
    logging: 'sanitized',
    retention: '3 years'
  },
  RESTRICTED: {
    level: 3,
    encryption: true,
    logging: false,
    retention: 'minimum required',
    accessControl: 'need-to-know'
  }
};

// Field classifications
const fieldClassifications = {
  userId: 'INTERNAL',
  email: 'CONFIDENTIAL',
  password: 'RESTRICTED',
  ssn: 'RESTRICTED',
  creditCard: 'RESTRICTED',
  address: 'CONFIDENTIAL',
  preferences: 'INTERNAL'
};
```

## Code Review Checklist

- [ ] All sensitive data encrypted at rest (AES-256-GCM)
- [ ] TLS 1.2+ enforced for data in transit
- [ ] HSTS header enabled
- [ ] No sensitive data in logs
- [ ] No sensitive data in URLs
- [ ] No sensitive data in error messages
- [ ] Encryption keys stored securely (KMS/secrets manager)
- [ ] Data masking for display/API responses
- [ ] Proper data classification implemented
- [ ] Data retention policies defined
- [ ] PII inventory documented

## Best Practices

1. **Minimize Data Collection**: Don't collect what you don't need
2. **Encrypt Everything Sensitive**: Both at rest and in transit
3. **Use Strong Algorithms**: AES-256, RSA-2048+, SHA-256+
4. **Rotate Keys Regularly**: Automated key rotation
5. **Audit Access**: Log who accesses sensitive data
6. **Data Retention**: Delete data when no longer needed
7. **Incident Response**: Plan for data breach scenarios
