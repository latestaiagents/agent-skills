---
name: secure-headers
description: |
  HTTP security headers configuration guide. Use this skill when hardening web
  applications, configuring CSP, or setting up security headers. Activate when:
  security headers, CSP, Content-Security-Policy, HSTS, X-Frame-Options, CORS headers,
  clickjacking prevention, helmet.
---

# Secure HTTP Headers

**Configure HTTP security headers to protect against common web attacks.**

## When to Use

- Setting up a new web application
- Hardening existing applications
- Fixing security scanner findings
- Implementing Content Security Policy
- Preventing clickjacking

## Essential Security Headers

| Header | Purpose | Priority |
|--------|---------|----------|
| Content-Security-Policy | XSS prevention | HIGH |
| Strict-Transport-Security | Force HTTPS | HIGH |
| X-Frame-Options | Clickjacking | HIGH |
| X-Content-Type-Options | MIME sniffing | MEDIUM |
| Referrer-Policy | Privacy | MEDIUM |
| Permissions-Policy | Feature control | MEDIUM |

## Implementation

### Express.js with Helmet

```javascript
const helmet = require('helmet');

app.use(helmet({
  // Content Security Policy
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "https://cdn.example.com"],
      styleSrc: ["'self'", "'unsafe-inline'"],  // Consider removing unsafe-inline
      imgSrc: ["'self'", "data:", "https:"],
      connectSrc: ["'self'", "https://api.example.com"],
      fontSrc: ["'self'", "https://fonts.gstatic.com"],
      objectSrc: ["'none'"],
      mediaSrc: ["'self'"],
      frameSrc: ["'none'"],
      frameAncestors: ["'none'"],
      formAction: ["'self'"],
      baseUri: ["'self'"],
      upgradeInsecureRequests: []
    }
  },

  // HTTP Strict Transport Security
  hsts: {
    maxAge: 31536000,  // 1 year
    includeSubDomains: true,
    preload: true
  },

  // Prevent clickjacking
  frameguard: { action: 'deny' },

  // Prevent MIME type sniffing
  noSniff: true,

  // Referrer Policy
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },

  // Permissions Policy
  permittedCrossDomainPolicies: { permittedPolicies: 'none' }
}));

// Additional headers
app.use((req, res, next) => {
  // Permissions Policy (formerly Feature-Policy)
  res.setHeader('Permissions-Policy',
    'geolocation=(), microphone=(), camera=(), payment=()');

  // Cross-Origin policies
  res.setHeader('Cross-Origin-Embedder-Policy', 'require-corp');
  res.setHeader('Cross-Origin-Opener-Policy', 'same-origin');
  res.setHeader('Cross-Origin-Resource-Policy', 'same-origin');

  next();
});
```

### Nginx Configuration

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    # SSL configuration
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    # Security Headers
    add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self'; object-src 'none'; frame-ancestors 'none';" always;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    add_header X-Frame-Options "DENY" always;

    add_header X-Content-Type-Options "nosniff" always;

    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;

    add_header Cross-Origin-Embedder-Policy "require-corp" always;
    add_header Cross-Origin-Opener-Policy "same-origin" always;
    add_header Cross-Origin-Resource-Policy "same-origin" always;

    # Hide server version
    server_tokens off;
}
```

### Apache Configuration

```apache
<VirtualHost *:443>
    ServerName example.com

    # Security Headers
    Header always set Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline';"

    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"

    Header always set X-Frame-Options "DENY"

    Header always set X-Content-Type-Options "nosniff"

    Header always set Referrer-Policy "strict-origin-when-cross-origin"

    Header always set Permissions-Policy "geolocation=(), microphone=(), camera=()"

    # Hide server version
    ServerTokens Prod
    ServerSignature Off
</VirtualHost>
```

## Content Security Policy Deep Dive

### Building a CSP

```javascript
// Start restrictive, then loosen as needed
const cspDirectives = {
  // Default fallback for all resource types
  defaultSrc: ["'self'"],

  // JavaScript sources
  scriptSrc: [
    "'self'",
    // "'unsafe-inline'",  // Avoid if possible
    // "'unsafe-eval'",    // Never use
    "'nonce-${nonce}'",    // Better than unsafe-inline
    "https://trusted-cdn.com"
  ],

  // CSS sources
  styleSrc: [
    "'self'",
    "'unsafe-inline'",  // Often needed for frameworks
    "https://fonts.googleapis.com"
  ],

  // Image sources
  imgSrc: [
    "'self'",
    "data:",           // For inline images
    "https:"           // Any HTTPS image
  ],

  // Font sources
  fontSrc: [
    "'self'",
    "https://fonts.gstatic.com"
  ],

  // AJAX/WebSocket/Fetch destinations
  connectSrc: [
    "'self'",
    "https://api.example.com",
    "wss://ws.example.com"
  ],

  // <object>, <embed>, <applet>
  objectSrc: ["'none'"],

  // <frame>, <iframe>
  frameSrc: ["'none'"],

  // Who can frame this page
  frameAncestors: ["'none'"],

  // Form submission targets
  formAction: ["'self'"],

  // <base> tag restrictions
  baseUri: ["'self'"],

  // Upgrade HTTP to HTTPS
  upgradeInsecureRequests: []
};
```

### CSP with Nonces (for inline scripts)

```javascript
// Generate nonce per request
app.use((req, res, next) => {
  res.locals.nonce = crypto.randomBytes(16).toString('base64');
  next();
});

// Set CSP with nonce
app.use((req, res, next) => {
  res.setHeader('Content-Security-Policy',
    `default-src 'self'; script-src 'self' 'nonce-${res.locals.nonce}'`);
  next();
});
```

```html
<!-- In template -->
<script nonce="<%= nonce %>">
  // Inline script that will execute
</script>
```

### CSP Reporting

```javascript
// Report-only mode (doesn't block, just reports)
const cspReportOnly = {
  ...cspDirectives,
  reportUri: '/csp-report'
};

app.use(helmet.contentSecurityPolicy({
  directives: cspReportOnly,
  reportOnly: true  // Test before enforcing
}));

// CSP report endpoint
app.post('/csp-report', express.json({ type: 'application/csp-report' }), (req, res) => {
  console.log('CSP Violation:', req.body);
  res.status(204).end();
});
```

## Header Reference

### Strict-Transport-Security (HSTS)

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload

max-age: Time in seconds browser should remember HTTPS-only
includeSubDomains: Apply to all subdomains
preload: Allow inclusion in browser preload list
```

### X-Frame-Options

```
X-Frame-Options: DENY           # Never allow framing
X-Frame-Options: SAMEORIGIN     # Only same origin can frame
```

### Referrer-Policy

```
Referrer-Policy: no-referrer                    # Never send referrer
Referrer-Policy: same-origin                    # Only same origin
Referrer-Policy: strict-origin-when-cross-origin  # Recommended
```

### Permissions-Policy

```
Permissions-Policy: geolocation=(), microphone=(), camera=(), payment=(), usb=()

# Allow specific origins
Permissions-Policy: geolocation=(self "https://maps.example.com")
```

## Testing Security Headers

```bash
# Check headers with curl
curl -I https://example.com

# Online tools
# https://securityheaders.com
# https://observatory.mozilla.org

# Check CSP specifically
# https://csp-evaluator.withgoogle.com
```

## Common Issues & Fixes

```javascript
// Issue: "Refused to execute inline script"
// Fix: Use nonce or move to external file
scriptSrc: ["'self'", `'nonce-${nonce}'`]

// Issue: "Refused to load stylesheet"
// Fix: Add the source
styleSrc: ["'self'", "https://cdn.example.com"]

// Issue: "Blocked frame with origin"
// Fix: Adjust frame-ancestors or X-Frame-Options
frameAncestors: ["'self'", "https://trusted-site.com"]

// Issue: Breaking third-party widgets
// Fix: Add specific sources (not *)
scriptSrc: ["'self'", "https://widget.example.com"]
```

## Best Practices

1. **Start Restrictive**: Begin with strict CSP, loosen as needed
2. **Use Report-Only First**: Test CSP before enforcing
3. **Avoid unsafe-inline**: Use nonces or hashes instead
4. **Never use unsafe-eval**: Refactor code that needs it
5. **Enable HSTS Preload**: Maximum HTTPS enforcement
6. **Test Thoroughly**: Security headers can break functionality
7. **Monitor Reports**: Set up CSP report collection
8. **Review Regularly**: Update as dependencies change
