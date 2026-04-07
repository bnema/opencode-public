# Server-Side Bugs — Reference

## SSRF Bypass Techniques

| Technique | Example | Effect |
|-----------|---------|--------|
| Decimal IP | `http://2130706433` | 127.0.0.1 as decimal |
| Octal IP | `http://0177.0.0.1` | Octal representation |
| Hex IP | `http://0x7f.0x0.0x0.0x1` | Hexadecimal |
| IPv6 localhost | `http://[::1]` | IPv6 loopback |
| IPv6 mapped IPv4 | `http://[::ffff:127.0.0.1]` | IPv4-mapped IPv6 |
| Short IPv6 | `http://[::]` | All zeros |
| DNS rebinding | Attacker DNS returns internal IP | First resolve external, second internal |
| CNAME to internal | Attacker domain CNAMEs to internal host | DNS points to internal hostname |
| URL parser confusion | `http://attacker.com#@internal` | Libraries parse differently |
| Redirect chains | External URL redirects to internal | Follows redirects to internal |
| IPv6 scope ID | `http://[fe80::1%25eth0]` | Interface-scoped IPv6 |
| Shortened IP | `http://127.1` | Rare notation |

### DNS Rebinding Prevention

1. Resolve DNS before making the request
2. Reject private/internal resolved IPs
3. Pin the resolved IP — never re-resolve
4. Alternative: resolve twice with delay, reject if results differ

### Cloud Metadata Endpoints to Block

- AWS: `169.254.169.254`
- GCP: `metadata.google.internal`, `169.254.169.254`
- Azure: `169.254.169.254`
- DigitalOcean: `169.254.169.254`

### SSRF Checklist

- [ ] Restrict URL scheme to HTTP/HTTPS
- [ ] Resolve DNS and reject private/internal IPs
- [ ] Block cloud metadata IPs
- [ ] Limit or disable redirect following
- [ ] Validate each redirect hop if following
- [ ] Set request timeout and response size limit
- [ ] Use network isolation where possible

## File Upload Bypass Techniques

| Attack | Example | Prevention |
|--------|---------|------------|
| Extension bypass | `shell.php.jpg` | Check full extension; use allowlist |
| Null byte | `shell.php%00.jpg` | Sanitize filename; check for null bytes |
| Double extension | `shell.jpg.php` | Allow only single extension |
| MIME spoofing | Content-Type set to `image/jpeg` | Validate magic bytes |
| Magic byte injection | Valid header + malicious body | Validate entire file structure |
| Polyglot files | Valid as both JPEG and JS | Parse as expected type; reject if invalid |
| SVG with JS | `<svg onload="alert(1)">` | Sanitize SVG or disallow |
| XXE via upload | Malicious DOCX/XLSX (XML inside) | Disable external entities in parser |
| ZIP slip | `../../../etc/passwd` in archive | Validate extracted paths |
| ImageMagick exploits | Crafted images | Keep updated; use policy.xml |
| Filename injection | `; rm -rf /` in filename | Sanitize filenames; use random names |
| Content-type confusion | Browser MIME sniffing | Set `X-Content-Type-Options: nosniff` |

### Magic Bytes Reference

| Type | Magic Bytes (hex) |
|------|-------------------|
| JPEG | `FF D8 FF` |
| PNG | `89 50 4E 47 0D 0A 1A 0A` |
| GIF | `47 49 46 38` |
| PDF | `25 50 44 46` |
| ZIP/DOCX/XLSX | `50 4B 03 04` |

### Secure Upload Handling

1. Rename to random UUID; discard original filename
2. Store outside webroot or on a separate domain
3. Serve with `Content-Disposition: attachment`, correct `Content-Type`, and `nosniff`
4. Set restrictive permissions — uploaded files must not be executable

## SQL Injection

### Parameterized Queries

```sql
-- VULNERABLE
query = "SELECT * FROM users WHERE id = " + userId

-- SECURE
query = "SELECT * FROM users WHERE id = ?"
execute(query, [userId])
```

### Injection Points

- WHERE, IN, LIKE clauses (escape `%` and `_` wildcards)
- ORDER BY — cannot parameterize; whitelist allowed columns
- Table/column names — cannot parameterize; whitelist
- LIMIT/OFFSET values
- INSERT values, UPDATE SET values

### Additional Defenses

- Grant DB user minimum required permissions
- Disable dangerous functions (`xp_cmdshell` in SQL Server)
- Never expose SQL errors to users

## XXE Prevention by Language

**Java:**
```java
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
dbf.setFeature("http://xml.org/sax/features/external-general-entities", false);
dbf.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
dbf.setExpandEntityReferences(false);
```

**Python (lxml):**
```python
from lxml import etree
parser = etree.XMLParser(resolve_entities=False, no_network=True)
# Or use defusedxml
```

**PHP:**
```php
libxml_disable_entity_loader(true);
```

**Node.js:**
```javascript
// Use libraries that disable DTD by default
// libxmljs: { noent: false, dtdload: false }
```

**.NET:**
```csharp
XmlReaderSettings settings = new XmlReaderSettings();
settings.DtdProcessing = DtdProcessing.Prohibit;
settings.XmlResolver = null;
```

### Vulnerable Scenarios

**Direct XML:** SOAP APIs, XML-RPC, XML uploads, config parsing, RSS/Atom feeds

**Indirect XML:** Office documents (DOCX/XLSX/PPTX are ZIP+XML), SVG files, SAML assertions, PDF with XFA forms

### XXE Checklist

- [ ] Disable DTD processing entirely if possible
- [ ] Disable external entity resolution
- [ ] Disable external DTD loading
- [ ] Disable XInclude processing
- [ ] Keep XML parsers updated
- [ ] Consider JSON instead of XML where possible

## Path Traversal

### Vulnerable Patterns

```python
# VULNERABLE
file_path = "/uploads/" + user_input
file_path = base_dir + request.params['file']
```

### Safe File Access

**Prefer indirect references:**
```python
files = {'report': '/reports/q1.pdf', 'invoice': '/invoices/2024.pdf'}
file_path = files.get(user_input)  # None if invalid
```

**Canonicalize and validate:**
```python
import os

def safe_join(base_directory, user_path):
    base = os.path.abspath(os.path.realpath(base_directory))
    target = os.path.abspath(os.path.realpath(os.path.join(base, user_path)))

    if os.path.commonpath([base, target]) != base:
        raise ValueError("Path traversal blocked")

    return target
```

### Input Sanitization

- Reject `..` sequences and absolute path indicators (`/`, `C:`)
- Whitelist allowed characters (alphanumeric, dash, underscore)
- Validate file extension if applicable

### Path Traversal Checklist

- [ ] Never use user input directly in file paths
- [ ] Canonicalize paths and validate against base directory
- [ ] Restrict file extensions if applicable
- [ ] Test with encoded and bypass variants
