# Client-Side Bugs — Reference

## XSS Input Sources

**Direct:** Form fields, search queries, upload filenames, rich text editors

**Indirect:** URL params/fragments, HTTP headers (Referer, User-Agent), third-party API data, WebSocket messages, postMessage from iframes, LocalStorage values rendered in DOM

**Often missed:** Error messages reflecting input, PDF generators accepting HTML, email templates with user data, admin log viewers, JSON rendered as HTML, SVG uploads (can contain JS), markdown with HTML enabled

## Output Encoding by Context

| Context | Method |
|---------|--------|
| HTML body | HTML entity encode (`<` → `&lt;`) |
| JavaScript | JS escape via framework serializers |
| URL | `encodeURIComponent()` |
| CSS | CSS escape; avoid user input in CSS |

Prefer framework built-ins: React JSX, Vue `{{ }}`, Angular templates.

## Content Security Policy

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  font-src 'self';
  connect-src 'self' https://api.yourdomain.com;
  frame-ancestors 'none';
  base-uri 'self';
  form-action 'self';
```

- Avoid `'unsafe-inline'` and `'unsafe-eval'` for scripts
- Use nonces or hashes for inline scripts when necessary
- Report violations: `report-uri /csp-report`

## CSRF Edge Cases

### Endpoints Requiring Protection

**Authenticated:** All POST/PUT/PATCH/DELETE. Any state-changing GET (fix to proper methods). File uploads, settings, payments.

**Pre-auth:** Login, signup, password reset/change, email/phone verification, OAuth callbacks.

### Protection Mechanisms

1. **CSRF Tokens** — cryptographically random, session-bound, validated every state change, regenerated after login
2. **SameSite Cookies** — `Set-Cookie: session=abc123; SameSite=Strict; Secure; HttpOnly`
3. **Double Submit Cookie** — token in both cookie and request body/header; server validates match

### Edge Cases

- Always require the token — reject requests where it's absent, not just wrong
- Scope tokens per form for sensitive operations
- JSON content-type alone does not prevent CSRF — validate Origin/Referer AND use tokens
- Overly permissive CORS bypasses SameSite cookies
- Scope CSRF tokens to prevent subdomain takeover escalation
- Never change state on GET
- Send tokens in headers (`X-CSRF-Token`), never in URLs

### CSRF Verification Checklist

- [ ] Token is cryptographically random (secure random generator)
- [ ] Token is tied to user session
- [ ] Token validated server-side on all state-changing requests
- [ ] Missing token = rejected request
- [ ] Token regenerated on authentication state change
- [ ] SameSite cookie attribute set
- [ ] Secure and HttpOnly flags on session cookies

## Secrets Exposure

**Where secrets hide:** JS bundles, source maps, HTML comments, hidden form fields, data attributes, LocalStorage, SSR hydration data, build-tool env vars (`NEXT_PUBLIC_*`, `REACT_APP_*`)

**Never expose client-side:** Third-party API keys, DB connection strings, JWT signing secrets, encryption keys, OAuth client secrets, internal service URLs, full credit card numbers, SSNs, passwords (even hashed), stack traces, server versions

**Mitigations:**
1. Store secrets in `.env` files; never commit them
2. Route secret-requiring API calls through the backend
3. Audit production bundles for accidentally included secrets
