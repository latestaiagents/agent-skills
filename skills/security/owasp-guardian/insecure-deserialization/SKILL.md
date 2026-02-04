---
name: insecure-deserialization
description: |
  OWASP A08 - Insecure Deserialization Prevention. Use this skill when parsing serialized
  data, handling JSON with type information, or processing pickled/marshalled objects.
  Activate when: deserialization, JSON parse, pickle, marshal, serialize, unserialize,
  ObjectInputStream, yaml.load, eval JSON, prototype pollution.
---

# Insecure Deserialization (OWASP A08)

**Prevent remote code execution and object injection through safe deserialization practices.**

## When to Use

- Parsing serialized objects from untrusted sources
- Handling session data or cookies
- Processing API payloads with type information
- Working with message queues
- Importing/exporting data

## Risk Levels by Language

| Language | Serialization | Risk | Impact |
|----------|--------------|------|--------|
| Java | ObjectInputStream | CRITICAL | RCE |
| Python | pickle/marshal | CRITICAL | RCE |
| PHP | unserialize() | CRITICAL | RCE |
| Ruby | Marshal.load | CRITICAL | RCE |
| Node.js | node-serialize | HIGH | RCE |
| .NET | BinaryFormatter | CRITICAL | RCE |
| JavaScript | JSON.parse | LOW | Prototype pollution |

## Vulnerable Patterns

### Python

```python
# VULNERABLE - pickle with untrusted data
import pickle
data = pickle.loads(user_input)  # RCE possible!

# VULNERABLE - yaml.load without SafeLoader
import yaml
data = yaml.load(user_input)  # RCE possible!

# VULNERABLE - marshal
import marshal
data = marshal.loads(user_input)
```

### Java

```java
// VULNERABLE - ObjectInputStream with untrusted data
ObjectInputStream ois = new ObjectInputStream(inputStream);
Object obj = ois.readObject();  // RCE possible!

// VULNERABLE - XMLDecoder
XMLDecoder decoder = new XMLDecoder(inputStream);
Object obj = decoder.readObject();
```

### PHP

```php
// VULNERABLE - unserialize with user input
$data = unserialize($_POST['data']);  // RCE possible!

// VULNERABLE - unserialize from cookie
$session = unserialize($_COOKIE['session']);
```

### Node.js

```javascript
// VULNERABLE - node-serialize
const serialize = require('node-serialize');
const obj = serialize.unserialize(userInput);  // RCE possible!

// VULNERABLE - js-yaml without safe loading
const yaml = require('js-yaml');
const data = yaml.load(userInput);  // Code execution possible!

// VULNERABLE - prototype pollution via JSON
const obj = JSON.parse('{"__proto__": {"admin": true}}');
```

### Ruby

```ruby
# VULNERABLE - Marshal with untrusted data
data = Marshal.load(user_input)  # RCE possible!

# VULNERABLE - YAML.load
data = YAML.load(user_input)
```

## Secure Implementation

### Python

```python
# SAFE - Use JSON instead of pickle
import json
data = json.loads(user_input)

# SAFE - yaml with SafeLoader
import yaml
data = yaml.safe_load(user_input)

# If you MUST use pickle, use hmac signing
import pickle
import hmac
import hashlib

SECRET_KEY = os.environ['SECRET_KEY'].encode()

def secure_serialize(obj):
    data = pickle.dumps(obj)
    signature = hmac.new(SECRET_KEY, data, hashlib.sha256).hexdigest()
    return f"{signature}:{data.hex()}"

def secure_deserialize(signed_data):
    signature, hex_data = signed_data.split(':')
    data = bytes.fromhex(hex_data)
    expected_sig = hmac.new(SECRET_KEY, data, hashlib.sha256).hexdigest()

    if not hmac.compare_digest(signature, expected_sig):
        raise ValueError("Invalid signature")

    return pickle.loads(data)
```

### Java

```java
// SAFE - Use JSON instead
import com.fasterxml.jackson.databind.ObjectMapper;

ObjectMapper mapper = new ObjectMapper();
MyClass obj = mapper.readValue(jsonString, MyClass.class);

// If you MUST deserialize, use allowlist
import org.apache.commons.io.serialization.ValidatingObjectInputStream;

ValidatingObjectInputStream vois = new ValidatingObjectInputStream(inputStream);
vois.accept(MyAllowedClass.class, AnotherAllowedClass.class);
vois.reject("*");  // Reject everything else
Object obj = vois.readObject();

// Or use a lookup-based deserialization filter (Java 9+)
ObjectInputFilter filter = ObjectInputFilter.Config.createFilter(
    "com.myapp.model.*;!*"  // Allow only specific packages
);
ois.setObjectInputFilter(filter);
```

### PHP

```php
// SAFE - Use JSON instead
$data = json_decode($_POST['data'], true);

// If you MUST unserialize, use allowed_classes
$data = unserialize($input, [
    'allowed_classes' => ['MyClass', 'AnotherClass']
]);

// Or disallow all classes
$data = unserialize($input, ['allowed_classes' => false]);
```

### Node.js

```javascript
// SAFE - Plain JSON.parse (but watch for prototype pollution)
const data = JSON.parse(userInput);

// SAFE - Prototype pollution prevention
function safeParse(json) {
  return JSON.parse(json, (key, value) => {
    if (key === '__proto__' || key === 'constructor' || key === 'prototype') {
      return undefined;
    }
    return value;
  });
}

// SAFE - Using secure-json-parse
const sjson = require('secure-json-parse');
const data = sjson.parse(userInput);

// SAFE - js-yaml with safe schema
const yaml = require('js-yaml');
const data = yaml.load(userInput, { schema: yaml.SAFE_SCHEMA });
```

### Ruby

```ruby
# SAFE - Use JSON instead
require 'json'
data = JSON.parse(user_input)

# SAFE - YAML with safe_load
require 'yaml'
data = YAML.safe_load(user_input, permitted_classes: [Date, Time])

# SAFE - Psych with safe loading
data = Psych.safe_load(user_input)
```

### .NET

```csharp
// VULNERABLE - BinaryFormatter
BinaryFormatter bf = new BinaryFormatter();
var obj = bf.Deserialize(stream);  // Never use with untrusted data!

// SAFE - Use JSON.NET with type handling disabled
var settings = new JsonSerializerSettings {
    TypeNameHandling = TypeNameHandling.None  // Important!
};
var obj = JsonConvert.DeserializeObject<MyClass>(json, settings);

// SAFE - System.Text.Json (no type handling by default)
var obj = JsonSerializer.Deserialize<MyClass>(json);
```

## Prototype Pollution Prevention (JavaScript)

```javascript
// Freeze Object.prototype
Object.freeze(Object.prototype);

// Use Object.create(null) for dictionaries
const safeDict = Object.create(null);
safeDict[userKey] = userValue;  // No prototype chain

// Validate keys before assignment
const FORBIDDEN_KEYS = ['__proto__', 'constructor', 'prototype'];

function safeAssign(target, key, value) {
  if (FORBIDDEN_KEYS.includes(key)) {
    throw new Error('Invalid key');
  }
  target[key] = value;
}

// Use Map instead of objects for user-controlled keys
const userPrefs = new Map();
userPrefs.set(userKey, userValue);
```

## Signed Serialization Pattern

```javascript
const crypto = require('crypto');

const SECRET = process.env.SERIALIZATION_SECRET;

function signedSerialize(obj) {
  const data = JSON.stringify(obj);
  const signature = crypto
    .createHmac('sha256', SECRET)
    .update(data)
    .digest('hex');

  return Buffer.from(JSON.stringify({ data, signature })).toString('base64');
}

function signedDeserialize(input) {
  const { data, signature } = JSON.parse(
    Buffer.from(input, 'base64').toString()
  );

  const expectedSig = crypto
    .createHmac('sha256', SECRET)
    .update(data)
    .digest('hex');

  if (!crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expectedSig)
  )) {
    throw new Error('Invalid signature');
  }

  return JSON.parse(data);
}
```

## Code Review Checklist

- [ ] No pickle/marshal with untrusted data (Python)
- [ ] No ObjectInputStream with untrusted data (Java)
- [ ] No unserialize() with untrusted data (PHP)
- [ ] No Marshal.load with untrusted data (Ruby)
- [ ] JSON used instead of native serialization
- [ ] YAML uses safe_load/SafeLoader
- [ ] Type handling disabled in JSON libraries
- [ ] Prototype pollution prevented in JavaScript
- [ ] Signed serialization for session data
- [ ] Input validation before deserialization

## Best Practices

1. **Avoid Native Serialization**: Use JSON or Protocol Buffers
2. **Sign Serialized Data**: Verify integrity before deserializing
3. **Type Whitelisting**: Only allow specific classes
4. **Input Validation**: Validate structure before processing
5. **Least Privilege**: Deserialize with minimal permissions
6. **Monitor**: Log deserialization errors for attack detection
