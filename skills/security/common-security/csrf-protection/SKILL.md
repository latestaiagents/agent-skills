---
name: csrf-protection
description: |
  Cross-Site Request Forgery prevention techniques. Use this skill when implementing
  forms, state-changing operations, or reviewing CSRF protections. Activate when:
  CSRF, cross-site request forgery, form security, token validation, same-site cookie,
  state changing request, POST request security.
---

# CSRF Protection

**Prevent Cross-Site Request Forgery attacks on your web application.**

## When to Use

- Implementing forms that change state
- Building APIs consumed by browsers
- Setting up session cookies
- Reviewing authentication flows
- Any state-changing POST/PUT/DELETE requests

## How CSRF Works

```html
<!-- Attacker's malicious page -->
<html>
  <body onload="document.forms[0].submit()">
    <form action="https://bank.com/transfer" method="POST">
      <input name="to" value="attacker" />
      <input name="amount" value="10000" />
    </form>
  </body>
</html>
<!-- Victim visits this page while logged into bank.com -->
<!-- Their session cookie is sent automatically! -->
```

## Protection Methods

### 1. SameSite Cookies (Primary Defense)

```javascript
// Express session with SameSite
app.use(session({
  secret: process.env.SESSION_SECRET,
  cookie: {
    httpOnly: true,
    secure: true,  // HTTPS only
    sameSite: 'strict',  // Or 'lax' for better UX
    maxAge: 3600000
  }
}));

// Set-Cookie header result:
// Set-Cookie: sessionId=abc123; HttpOnly; Secure; SameSite=Strict
```

**SameSite Options:**

| Value | Behavior |
|-------|----------|
| `Strict` | Cookie never sent cross-site |
| `Lax` | Sent on top-level navigation (default) |
| `None` | Always sent (requires Secure) |

### 2. CSRF Tokens (Defense in Depth)

```javascript
const csrf = require('csurf');

// Setup CSRF middleware
const csrfProtection = csrf({
  cookie: {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict'
  }
});

// Apply to routes
app.get('/form', csrfProtection, (req, res) => {
  res.render('form', { csrfToken: req.csrfToken() });
});

app.post('/submit', csrfProtection, (req, res) => {
  // Token automatically validated by middleware
  // Process form...
});
```

```html
<!-- In your form template -->
<form method="POST" action="/submit">
  <input type="hidden" name="_csrf" value="<%= csrfToken %>">
  <!-- Other form fields -->
  <button type="submit">Submit</button>
</form>
```

### 3. Double Submit Cookie Pattern

```javascript
// Generate CSRF token
function generateCsrfToken() {
  return crypto.randomBytes(32).toString('hex');
}

// Set token in cookie and return for form
app.get('/form', (req, res) => {
  const token = generateCsrfToken();

  res.cookie('csrf-token', token, {
    httpOnly: false,  // JS needs to read this
    secure: true,
    sameSite: 'strict'
  });

  res.render('form', { csrfToken: token });
});

// Validate both match
app.post('/submit', (req, res) => {
  const cookieToken = req.cookies['csrf-token'];
  const bodyToken = req.body._csrf || req.headers['x-csrf-token'];

  if (!cookieToken || !bodyToken ||
      !crypto.timingSafeEqual(Buffer.from(cookieToken), Buffer.from(bodyToken))) {
    return res.status(403).json({ error: 'Invalid CSRF token' });
  }

  // Process request...
});
```

### 4. Custom Header Verification (for APIs)

```javascript
// Require custom header that can't be set cross-origin
function csrfHeaderCheck(req, res, next) {
  // Browsers block cross-origin custom headers
  const csrfHeader = req.headers['x-requested-with'];

  if (csrfHeader !== 'XMLHttpRequest') {
    return res.status(403).json({ error: 'CSRF validation failed' });
  }

  next();
}

// Apply to API routes
app.use('/api', csrfHeaderCheck);
```

```javascript
// Client-side
fetch('/api/data', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-Requested-With': 'XMLHttpRequest'  // Custom header
  },
  body: JSON.stringify(data),
  credentials: 'include'
});
```

### 5. Origin/Referer Validation

```javascript
function validateOrigin(req, res, next) {
  const origin = req.headers.origin || req.headers.referer;

  if (!origin) {
    // Might be same-origin request - additional checks needed
    return next();
  }

  try {
    const url = new URL(origin);
    const allowedOrigins = [
      'https://example.com',
      'https://www.example.com'
    ];

    if (!allowedOrigins.includes(url.origin)) {
      return res.status(403).json({ error: 'Invalid origin' });
    }
  } catch {
    return res.status(403).json({ error: 'Invalid origin header' });
  }

  next();
}
```

## Framework-Specific Implementation

### React (with fetch)

```jsx
// Get CSRF token from meta tag or cookie
function getCsrfToken() {
  return document.querySelector('meta[name="csrf-token"]')?.content
    || document.cookie.match(/csrf-token=([^;]+)/)?.[1];
}

// Custom fetch wrapper
async function secureFetch(url, options = {}) {
  const csrfToken = getCsrfToken();

  return fetch(url, {
    ...options,
    credentials: 'include',
    headers: {
      ...options.headers,
      'X-CSRF-Token': csrfToken
    }
  });
}

// Usage
await secureFetch('/api/update', {
  method: 'POST',
  body: JSON.stringify(data)
});
```

### Django

```python
# settings.py - CSRF is enabled by default
MIDDLEWARE = [
    'django.middleware.csrf.CsrfViewMiddleware',
    # ...
]

# In templates
<form method="post">
    {% csrf_token %}
    <!-- form fields -->
</form>

# For AJAX
<script>
const csrftoken = document.querySelector('[name=csrfmiddlewaretoken]').value;

fetch('/api/endpoint', {
    method: 'POST',
    headers: {
        'X-CSRFToken': csrftoken
    },
    body: JSON.stringify(data)
});
</script>
```

### Rails

```ruby
# ApplicationController
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception
end

# In views
<%= form_with url: '/submit' do |f| %>
  <!-- CSRF token automatically included -->
<% end %>

# For AJAX (Rails UJS handles this automatically)
# Or manually:
headers: {
  'X-CSRF-Token': document.querySelector('meta[name="csrf-token"]').content
}
```

### Laravel

```php
// In Blade templates
<form method="POST" action="/submit">
    @csrf
    <!-- form fields -->
</form>

// For AJAX - token in meta tag
<meta name="csrf-token" content="{{ csrf_token() }}">

// JavaScript
fetch('/api/endpoint', {
    method: 'POST',
    headers: {
        'X-CSRF-TOKEN': document.querySelector('meta[name="csrf-token"]').content
    }
});
```

## Common Mistakes

```javascript
// MISTAKE 1: CSRF token in URL (visible in logs/history)
<a href="/delete?csrf=abc123">Delete</a>  // BAD

// MISTAKE 2: Not validating on state-changing requests
app.get('/delete/:id', deleteHandler);  // GET shouldn't change state

// MISTAKE 3: Accepting token from any location
const token = req.query.csrf || req.body.csrf;  // Don't check query!

// MISTAKE 4: SameSite=None without understanding
cookie: { sameSite: 'none', secure: true }  // Opens CSRF risk

// MISTAKE 5: Not regenerating token after login
// Token should change when auth state changes
```

## Testing CSRF Protection

```html
<!-- Test page (host on different domain) -->
<!DOCTYPE html>
<html>
<body>
  <h1>CSRF Test</h1>

  <!-- Test 1: Form submission -->
  <form id="test1" action="http://target.com/api/update" method="POST">
    <input name="data" value="malicious">
  </form>

  <!-- Test 2: Fetch request -->
  <script>
    fetch('http://target.com/api/update', {
      method: 'POST',
      credentials: 'include',
      body: JSON.stringify({ data: 'malicious' })
    }).then(r => console.log('Fetch result:', r.status));
  </script>

  <!-- Test 3: Image tag (for GET requests) -->
  <img src="http://target.com/api/delete?id=1" />
</body>
</html>
```

## Code Review Checklist

- [ ] SameSite cookie attribute set (Strict or Lax)
- [ ] CSRF tokens on all state-changing forms
- [ ] Tokens validated server-side
- [ ] Tokens regenerated on authentication changes
- [ ] No state changes on GET requests
- [ ] Origin/Referer validated for sensitive operations
- [ ] CORS properly configured
- [ ] Custom headers required for API calls

## Best Practices

1. **Use SameSite Cookies**: First line of defense
2. **Add CSRF Tokens**: Defense in depth
3. **Validate Origin**: Extra layer for sensitive ops
4. **No State Changes on GET**: REST properly
5. **Regenerate Tokens**: After login/privilege change
6. **Use Framework Defaults**: Don't disable built-in protection
