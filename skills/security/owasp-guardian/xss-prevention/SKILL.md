---
name: xss-prevention
description: |
  OWASP A07 - Cross-Site Scripting (XSS) Prevention. Use this skill when rendering user
  input in HTML, handling DOM manipulation, or building frontend components. Activate when:
  XSS, cross-site scripting, user input display, innerHTML, dangerouslySetInnerHTML,
  template injection, script injection, sanitize HTML, escape output.
---

# XSS Prevention (OWASP A07)

**Prevent Cross-Site Scripting attacks by properly encoding output and sanitizing user input.**

## When to Use

- Displaying user-generated content
- Building dynamic HTML
- Implementing rich text editors
- Rendering markdown or HTML
- Working with URL parameters in pages
- Building search results pages

## XSS Types

| Type | Vector | Example |
|------|--------|---------|
| Reflected | URL parameters | `?search=<script>alert(1)</script>` |
| Stored | Database content | Comment with malicious script |
| DOM-based | Client-side JS | `document.write(location.hash)` |

## Vulnerable Patterns

### Server-Side

```javascript
// VULNERABLE - Direct interpolation
app.get('/search', (req, res) => {
  res.send(`<h1>Results for: ${req.query.q}</h1>`);
});

// VULNERABLE - Template without escaping
res.render('profile', { bio: user.bio }); // If template doesn't auto-escape
```

### Client-Side

```javascript
// VULNERABLE - innerHTML with user data
element.innerHTML = userInput;
document.getElementById('output').innerHTML = data;

// VULNERABLE - document.write
document.write(location.search);

// VULNERABLE - eval with user data
eval(userCode);

// VULNERABLE - jQuery html()
$('#output').html(userData);

// VULNERABLE - React dangerouslySetInnerHTML
<div dangerouslySetInnerHTML={{__html: userContent}} />
```

## Secure Implementation

### 1. Output Encoding

```javascript
// HTML entity encoding
function escapeHtml(text) {
  const map = {
    '&': '&amp;',
    '<': '&lt;',
    '>': '&gt;',
    '"': '&quot;',
    "'": '&#x27;',
    '/': '&#x2F;'
  };
  return text.replace(/[&<>"'/]/g, char => map[char]);
}

// Usage
app.get('/search', (req, res) => {
  const safeQuery = escapeHtml(req.query.q);
  res.send(`<h1>Results for: ${safeQuery}</h1>`);
});
```

### 2. Context-Aware Encoding

```javascript
// Different contexts need different encoding
const encoders = {
  // HTML body context
  html: (str) => str
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;'),

  // HTML attribute context
  attr: (str) => str
    .replace(/&/g, '&amp;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#x27;'),

  // JavaScript string context
  js: (str) => str
    .replace(/\\/g, '\\\\')
    .replace(/'/g, "\\'")
    .replace(/"/g, '\\"')
    .replace(/\n/g, '\\n'),

  // URL parameter context
  url: (str) => encodeURIComponent(str),

  // CSS context
  css: (str) => str.replace(/[^a-zA-Z0-9]/g, char =>
    '\\' + char.charCodeAt(0).toString(16) + ' '
  )
};

// Usage based on context
`<div>${encoders.html(userInput)}</div>`
`<input value="${encoders.attr(userInput)}">`
`<script>var x = '${encoders.js(userInput)}';</script>`
`<a href="/search?q=${encoders.url(userInput)}">`
```

### 3. Content Security Policy (CSP)

```javascript
// Express with helmet
const helmet = require('helmet');

app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc: ["'self'"], // No 'unsafe-inline'!
    styleSrc: ["'self'", "'unsafe-inline'"],
    imgSrc: ["'self'", "data:", "https:"],
    connectSrc: ["'self'", "https://api.example.com"],
    fontSrc: ["'self'"],
    objectSrc: ["'none'"],
    frameAncestors: ["'none'"],
    baseUri: ["'self'"],
    formAction: ["'self'"]
  }
}));

// With nonce for inline scripts (when needed)
app.use((req, res, next) => {
  res.locals.nonce = crypto.randomBytes(16).toString('base64');
  next();
});

app.use(helmet.contentSecurityPolicy({
  directives: {
    scriptSrc: ["'self'", (req, res) => `'nonce-${res.locals.nonce}'`]
  }
}));

// In template
`<script nonce="${nonce}">...</script>`
```

### 4. HTML Sanitization

```javascript
// Using DOMPurify (recommended)
const createDOMPurify = require('dompurify');
const { JSDOM } = require('jsdom');

const window = new JSDOM('').window;
const DOMPurify = createDOMPurify(window);

function sanitizeHtml(dirty) {
  return DOMPurify.sanitize(dirty, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'br', 'ul', 'li'],
    ALLOWED_ATTR: ['href', 'target'],
    ALLOW_DATA_ATTR: false
  });
}

// For rich text editors
function sanitizeRichText(html) {
  return DOMPurify.sanitize(html, {
    ALLOWED_TAGS: [
      'h1', 'h2', 'h3', 'p', 'br', 'ul', 'ol', 'li',
      'b', 'i', 'u', 'strong', 'em', 'a', 'img',
      'blockquote', 'code', 'pre'
    ],
    ALLOWED_ATTR: ['href', 'src', 'alt', 'class'],
    FORBID_TAGS: ['script', 'style', 'iframe', 'form', 'input'],
    FORBID_ATTR: ['onerror', 'onload', 'onclick', 'onmouseover']
  });
}
```

### 5. React Safe Patterns

```jsx
// SAFE - React auto-escapes by default
function UserProfile({ user }) {
  return <div>{user.name}</div>; // Automatically escaped
}

// SAFE - Use textContent for DOM
useEffect(() => {
  document.getElementById('output').textContent = userInput;
}, [userInput]);

// When you MUST render HTML, sanitize first
import DOMPurify from 'dompurify';

function RichContent({ html }) {
  const sanitized = DOMPurify.sanitize(html);
  return <div dangerouslySetInnerHTML={{__html: sanitized}} />;
}

// SAFE - URL validation
function SafeLink({ url, children }) {
  const isValidUrl = url.startsWith('https://') || url.startsWith('/');
  if (!isValidUrl) {
    return <span>{children}</span>;
  }
  return <a href={url}>{children}</a>;
}
```

### 6. Vue.js Safe Patterns

```vue
<!-- SAFE - Vue auto-escapes -->
<template>
  <div>{{ userInput }}</div>
</template>

<!-- DANGEROUS - v-html -->
<template>
  <div v-html="userContent"></div> <!-- XSS risk! -->
</template>

<!-- SAFE - Sanitize before v-html -->
<script>
import DOMPurify from 'dompurify';

export default {
  computed: {
    safeContent() {
      return DOMPurify.sanitize(this.userContent);
    }
  }
}
</script>
<template>
  <div v-html="safeContent"></div>
</template>
```

### 7. URL Validation

```javascript
// Prevent javascript: URLs
function isSafeUrl(url) {
  try {
    const parsed = new URL(url, window.location.origin);
    return ['http:', 'https:', 'mailto:'].includes(parsed.protocol);
  } catch {
    return false;
  }
}

// Usage
function SafeAnchor({ href, children }) {
  if (!isSafeUrl(href)) {
    return <span>{children}</span>;
  }
  return <a href={href} rel="noopener noreferrer">{children}</a>;
}
```

## Framework-Specific Guidance

### Express/Node.js
- Use template engines with auto-escaping (EJS, Pug, Handlebars)
- Set `{escape: true}` in template options
- Use helmet for CSP headers

### Django
- Templates auto-escape by default
- Use `|safe` filter only with sanitized content
- Enable CSP middleware

### Rails
- ERB auto-escapes with `<%= %>`
- Use `raw()` or `html_safe` only with sanitized content
- Use `sanitize()` helper for user HTML

### Laravel
- Blade `{{ }}` auto-escapes
- Use `{!! !!}` only with sanitized content
- Use `e()` helper for manual escaping

## Code Review Checklist

- [ ] All user input encoded before output
- [ ] Context-appropriate encoding used
- [ ] No innerHTML with unsanitized data
- [ ] No dangerouslySetInnerHTML without sanitization
- [ ] No eval() or new Function() with user data
- [ ] CSP headers configured
- [ ] URL schemes validated (no javascript:)
- [ ] Rich text sanitized with DOMPurify or similar
- [ ] Template engine auto-escaping enabled
- [ ] No user data in script blocks

## Testing for XSS

```html
<!-- Test payloads -->
<script>alert('XSS')</script>
<img src=x onerror=alert('XSS')>
<svg onload=alert('XSS')>
<body onload=alert('XSS')>
"><script>alert('XSS')</script>
'-alert('XSS')-'
javascript:alert('XSS')
data:text/html,<script>alert('XSS')</script>
```

```bash
# Automated scanning
# Use Burp Suite, OWASP ZAP, or XSStrike
python xsstrike.py -u "http://target.com/search?q=test"
```

## Best Practices

1. **Encode Output**: Always encode data based on context
2. **Use CSP**: Defense in depth against XSS
3. **Sanitize HTML**: Use DOMPurify for user HTML
4. **Validate URLs**: Block javascript: and data: URLs
5. **HttpOnly Cookies**: Prevent cookie theft
6. **Use Frameworks**: Leverage auto-escaping features
