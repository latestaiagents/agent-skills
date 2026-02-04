---
name: logging-monitoring
description: |
  OWASP A10 - Insufficient Logging and Monitoring. Use this skill when implementing
  audit logs, security monitoring, or incident detection. Activate when: logging,
  audit trail, security events, monitoring, alerting, SIEM, incident detection,
  log analysis, security logging, breach detection.
---

# Security Logging & Monitoring (OWASP A10)

**Implement comprehensive logging and monitoring to detect and respond to security incidents.**

## When to Use

- Setting up application logging
- Implementing audit trails
- Configuring security alerting
- Building incident detection
- Compliance requirements (SOC2, GDPR)
- Post-incident forensics

## Critical Events to Log

| Event Category | Examples | Priority |
|---------------|----------|----------|
| Authentication | Login success/failure, logout, MFA events | HIGH |
| Authorization | Access denied, privilege changes | HIGH |
| Data Access | Sensitive data reads, exports | HIGH |
| Data Modification | Create, update, delete operations | MEDIUM |
| Security Events | Input validation failures, rate limits | HIGH |
| System Events | Startup, shutdown, errors | MEDIUM |
| Admin Actions | Config changes, user management | HIGH |

## What NOT to Log

```javascript
// NEVER log these:
const NEVER_LOG = [
  'passwords',
  'credit_card_numbers',
  'ssn',
  'api_keys',
  'tokens',
  'session_ids',
  'private_keys',
  'health_information'
];
```

## Secure Logging Implementation

### 1. Structured Security Logger

```javascript
const winston = require('winston');

// Security event types
const SecurityEventType = {
  AUTH_SUCCESS: 'AUTH_SUCCESS',
  AUTH_FAILURE: 'AUTH_FAILURE',
  AUTH_LOGOUT: 'AUTH_LOGOUT',
  ACCESS_DENIED: 'ACCESS_DENIED',
  PRIVILEGE_ESCALATION: 'PRIVILEGE_ESCALATION',
  DATA_ACCESS: 'DATA_ACCESS',
  DATA_EXPORT: 'DATA_EXPORT',
  INPUT_VALIDATION_FAILURE: 'INPUT_VALIDATION_FAILURE',
  RATE_LIMIT_EXCEEDED: 'RATE_LIMIT_EXCEEDED',
  SUSPICIOUS_ACTIVITY: 'SUSPICIOUS_ACTIVITY',
  ADMIN_ACTION: 'ADMIN_ACTION',
  CONFIG_CHANGE: 'CONFIG_CHANGE'
};

// Security logger configuration
const securityLogger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp({ format: 'ISO' }),
    winston.format.json()
  ),
  defaultMeta: {
    service: 'my-app',
    environment: process.env.NODE_ENV
  },
  transports: [
    // Security events to dedicated file
    new winston.transports.File({
      filename: 'logs/security.log',
      level: 'info'
    }),
    // Critical events to separate file
    new winston.transports.File({
      filename: 'logs/security-critical.log',
      level: 'warn'
    }),
    // Send to SIEM (example: Splunk HEC)
    new winston.transports.Http({
      host: 'splunk.example.com',
      port: 8088,
      path: '/services/collector',
      ssl: true
    })
  ]
});

// Log security event function
function logSecurityEvent(eventType, details) {
  const event = {
    eventType,
    timestamp: new Date().toISOString(),
    ...sanitizeForLogging(details)
  };

  // Determine log level based on event type
  const criticalEvents = [
    SecurityEventType.AUTH_FAILURE,
    SecurityEventType.ACCESS_DENIED,
    SecurityEventType.PRIVILEGE_ESCALATION,
    SecurityEventType.SUSPICIOUS_ACTIVITY
  ];

  if (criticalEvents.includes(eventType)) {
    securityLogger.warn(event);
  } else {
    securityLogger.info(event);
  }

  return event;
}
```

### 2. Authentication Event Logging

```javascript
// Login attempt logging
async function handleLogin(req, res) {
  const { email, password } = req.body;
  const clientIp = req.ip;
  const userAgent = req.headers['user-agent'];

  try {
    const user = await authenticateUser(email, password);

    logSecurityEvent(SecurityEventType.AUTH_SUCCESS, {
      userId: user.id,
      email: maskEmail(email),
      ip: clientIp,
      userAgent,
      method: 'password',
      mfaUsed: false
    });

    // Continue with login...
  } catch (error) {
    logSecurityEvent(SecurityEventType.AUTH_FAILURE, {
      email: maskEmail(email),
      ip: clientIp,
      userAgent,
      reason: error.code,  // e.g., 'INVALID_PASSWORD', 'USER_NOT_FOUND'
      attemptCount: await getFailedAttemptCount(email)
    });

    // Don't reveal if user exists
    res.status(401).json({ error: 'Invalid credentials' });
  }
}

// Logout logging
function handleLogout(req, res) {
  logSecurityEvent(SecurityEventType.AUTH_LOGOUT, {
    userId: req.user.id,
    sessionDuration: Date.now() - req.session.loginTime,
    ip: req.ip
  });

  req.session.destroy();
  res.json({ success: true });
}
```

### 3. Authorization Event Logging

```javascript
// Access control logging middleware
function logAccessControl(req, res, next) {
  const originalJson = res.json.bind(res);

  res.json = (body) => {
    // Log access denied
    if (res.statusCode === 403) {
      logSecurityEvent(SecurityEventType.ACCESS_DENIED, {
        userId: req.user?.id,
        path: req.path,
        method: req.method,
        ip: req.ip,
        requiredRole: req.requiredRole,
        userRole: req.user?.role
      });
    }

    return originalJson(body);
  };

  next();
}

// Privilege change logging
async function changeUserRole(adminId, targetUserId, newRole) {
  const oldRole = await getUserRole(targetUserId);

  await updateUserRole(targetUserId, newRole);

  logSecurityEvent(SecurityEventType.ADMIN_ACTION, {
    action: 'ROLE_CHANGE',
    adminId,
    targetUserId,
    oldRole,
    newRole,
    timestamp: new Date().toISOString()
  });
}
```

### 4. Data Access Logging

```javascript
// Sensitive data access logging
async function getSensitiveUserData(requesterId, targetUserId) {
  const data = await db.users.findById(targetUserId, {
    include: ['ssn', 'taxId', 'bankAccount']
  });

  logSecurityEvent(SecurityEventType.DATA_ACCESS, {
    requesterId,
    targetUserId,
    dataType: 'sensitive_pii',
    fields: ['ssn', 'taxId', 'bankAccount'],
    reason: 'customer_support_request',
    ticketId: 'TICKET-123'
  });

  return data;
}

// Data export logging
async function exportUserData(adminId, filters) {
  const recordCount = await db.users.count(filters);

  logSecurityEvent(SecurityEventType.DATA_EXPORT, {
    adminId,
    exportType: 'user_data',
    filters: sanitizeForLogging(filters),
    recordCount,
    format: 'csv'
  });

  return generateExport(filters);
}
```

### 5. Input Validation Failure Logging

```javascript
// Log validation failures (potential attack indicators)
function validateAndLog(schema, data, req) {
  const result = schema.validate(data);

  if (result.error) {
    logSecurityEvent(SecurityEventType.INPUT_VALIDATION_FAILURE, {
      path: req.path,
      method: req.method,
      ip: req.ip,
      userId: req.user?.id,
      validationErrors: result.error.details.map(d => ({
        field: d.path.join('.'),
        message: d.message,
        // Don't log the actual invalid value (might be attack payload)
        valueLength: String(d.context?.value).length
      }))
    });
  }

  return result;
}
```

### 6. Rate Limiting & Suspicious Activity

```javascript
// Rate limit exceeded logging
const rateLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  handler: (req, res) => {
    logSecurityEvent(SecurityEventType.RATE_LIMIT_EXCEEDED, {
      ip: req.ip,
      path: req.path,
      userId: req.user?.id,
      windowMs: 15 * 60 * 1000,
      requestCount: req.rateLimit.current
    });

    res.status(429).json({ error: 'Too many requests' });
  }
});

// Suspicious activity detection
function detectSuspiciousActivity(req) {
  const indicators = [];

  // Multiple failed logins
  if (req.failedLoginCount > 5) {
    indicators.push('multiple_failed_logins');
  }

  // Unusual user agent
  if (!req.headers['user-agent'] || req.headers['user-agent'].includes('sqlmap')) {
    indicators.push('suspicious_user_agent');
  }

  // Geographic anomaly
  if (req.geoLocation !== req.user?.lastKnownLocation) {
    indicators.push('geographic_anomaly');
  }

  if (indicators.length > 0) {
    logSecurityEvent(SecurityEventType.SUSPICIOUS_ACTIVITY, {
      userId: req.user?.id,
      ip: req.ip,
      indicators,
      path: req.path
    });
  }
}
```

### 7. Alerting Configuration

```javascript
// Alert on critical security events
const alertThresholds = {
  AUTH_FAILURE: { count: 10, windowMinutes: 5 },
  ACCESS_DENIED: { count: 20, windowMinutes: 10 },
  RATE_LIMIT_EXCEEDED: { count: 5, windowMinutes: 1 },
  SUSPICIOUS_ACTIVITY: { count: 1, windowMinutes: 1 }  // Immediate
};

async function checkAlertThreshold(eventType, identifier) {
  const threshold = alertThresholds[eventType];
  if (!threshold) return;

  const count = await redis.incr(`alert:${eventType}:${identifier}`);
  await redis.expire(`alert:${eventType}:${identifier}`, threshold.windowMinutes * 60);

  if (count >= threshold.count) {
    await sendSecurityAlert({
      eventType,
      identifier,
      count,
      windowMinutes: threshold.windowMinutes,
      severity: count > threshold.count * 2 ? 'CRITICAL' : 'HIGH'
    });
  }
}

async function sendSecurityAlert(alert) {
  // Send to Slack
  await slack.send({
    channel: '#security-alerts',
    text: `🚨 Security Alert: ${alert.eventType}`,
    attachments: [{
      color: alert.severity === 'CRITICAL' ? 'danger' : 'warning',
      fields: [
        { title: 'Event', value: alert.eventType },
        { title: 'Count', value: String(alert.count) },
        { title: 'Window', value: `${alert.windowMinutes} minutes` }
      ]
    }]
  });

  // Send to PagerDuty for critical
  if (alert.severity === 'CRITICAL') {
    await pagerduty.trigger({
      routing_key: process.env.PAGERDUTY_KEY,
      event_action: 'trigger',
      payload: {
        summary: `Security Alert: ${alert.eventType}`,
        severity: 'critical',
        source: 'security-monitoring'
      }
    });
  }
}
```

## Log Retention & Compliance

```javascript
// Log retention policy
const retentionPolicy = {
  security: '1 year',      // Security events
  audit: '7 years',        // Compliance audit trail
  application: '90 days',  // General application logs
  debug: '7 days'          // Debug logs
};

// GDPR-compliant log anonymization
function anonymizeOldLogs(olderThanDays) {
  // Replace PII with anonymized values
  // Keep event structure for security analysis
}
```

## Code Review Checklist

- [ ] Authentication events logged (success and failure)
- [ ] Authorization failures logged
- [ ] Sensitive data access logged
- [ ] Admin actions logged
- [ ] Input validation failures logged
- [ ] No sensitive data in logs
- [ ] Logs are tamper-evident (signed/append-only)
- [ ] Log retention policy defined
- [ ] Alerting configured for critical events
- [ ] Logs sent to centralized SIEM
- [ ] Log access is restricted and audited

## Best Practices

1. **Log All Security Events**: Don't skip auth, access, and admin actions
2. **Sanitize Log Data**: Never log passwords, tokens, or PII
3. **Use Structured Logging**: JSON format for easy parsing
4. **Centralize Logs**: Send to SIEM for correlation
5. **Set Up Alerts**: Don't just collect, actively monitor
6. **Protect Log Integrity**: Prevent tampering
7. **Define Retention**: Balance security needs with privacy
8. **Test Detection**: Regularly test if you'd catch an attack
