# Open Redirect — Reference

## Protection Strategies

**1. Allowlist validation:**
```
allowed_domains = ['yourdomain.com', 'app.yourdomain.com']

function isValidRedirect(url):
    parsed = parseUrl(url)
    return parsed.hostname in allowed_domains
```

**2. Relative URLs only** — accept paths (`/dashboard`), reject full URLs. Validate the path starts with `/` and contains no `//`.

**3. Indirect references** — map keys to URLs: `?redirect=dashboard` → lookup to `/dashboard`.

## Bypass Techniques

| Technique | Example | Effect |
|-----------|---------|--------|
| @ symbol | `https://legit.com@evil.com` | Browser navigates to evil.com |
| Subdomain abuse | `https://legit.com.evil.com` | evil.com owns the subdomain |
| Protocol tricks | `javascript:alert(1)` | XSS via redirect |
| Double URL encoding | `%252f%252fevil.com` | Decodes to `//evil.com` after double decode |
| Backslash | `https://legit.com\@evil.com` | Some parsers normalize `\` to `/` |
| Null byte | `https://legit.com%00.evil.com` | Some parsers truncate at null |
| Tab/newline | `https://legit.com%09.evil.com` | Whitespace confusion |
| Unicode homograph | `https://legіt.com` (Cyrillic і) | IDN homograph attack |
| Data URLs | `data:text/html,<script>...` | Direct payload execution |
| Protocol-relative | `//evil.com` | Uses current page's protocol |
| Fragment abuse | `https://legit.com#@evil.com` | Libraries parse differently |

## IDN Homograph Protection

- Convert URLs to Punycode before validation
- Consider blocking non-ASCII domains for sensitive redirects
