---
name: vibesec
description: Use when writing, reviewing, or modifying web application code that handles user input, authentication, file uploads, URL redirects, database queries, or serves HTTP responses. Also use when adding API endpoints, webhooks, or any feature where untrusted data crosses a trust boundary.
---

# Secure Web Application Coding

Think like a bug hunter. Secure applications without breaking functionality.

## Key Principles

- **Defense in depth**: Never rely on a single security control
- **Fail closed**: When something fails, deny access
- **Least privilege**: Grant minimum permissions necessary
- **Validate server-side**: Never trust client input
- **Encode for context**: HTML, JS, URL, CSS need different encoding

## Per-Feature Checklist

- [ ] All input validated server-side
- [ ] Parameterized queries (never string concatenation)
- [ ] Output encoded for rendering context
- [ ] Auth check on every endpoint
- [ ] Authorization verifies user owns the resource
- [ ] Errors don't leak internals (stack traces, DB schemas)
- [ ] Security headers set (see below)
- [ ] No secrets in client-side code

## Access Control

Every authenticated action needs:

1. **User-level authorization** — verify ownership at the data layer, not just routes
2. **UUIDs over sequential IDs** — prevent enumeration (unless user requests sequential)
3. **Account lifecycle** — revoke tokens/sessions immediately on removal or deactivation

**Common vulnerabilities:** IDOR, privilege escalation (horizontal and vertical), mass assignment.

**Return 404 (not 403)** for unauthorized resources to prevent enumeration.

See @access-control.md for implementation patterns and checklist.

## Client-Side Bugs

### XSS Prevention

Sanitize every user-controllable input — direct (forms, search) and indirect (URL params, headers, third-party API data, WebSocket messages).

**Often overlooked:** Error messages reflecting input, PDF generators accepting HTML, SVG uploads, markdown rendering, JSON rendered as HTML.

**Defenses:**
1. Context-specific output encoding (use framework built-ins)
2. Content Security Policy — avoid `'unsafe-inline'` and `'unsafe-eval'` for scripts
3. Input sanitization with established libraries (DOMPurify)
4. `X-Content-Type-Options: nosniff`

### CSRF Prevention

Every state-changing endpoint needs protection, including pre-auth (login, signup, password reset, OAuth callbacks).

**Defenses:**
1. Cryptographically random CSRF tokens tied to session, validated on every state change
2. `SameSite=Strict` (or `Lax`) + `Secure` + `HttpOnly` on cookies
3. Missing token = rejected request (never optional)

**Key mistake:** Don't assume JSON content-type prevents CSRF — validate Origin/Referer AND use tokens.

### Secrets Exposure

Never expose in client code: API keys, DB strings, JWT secrets, encryption keys, OAuth secrets.

**Where secrets hide:** JS bundles, source maps, HTML comments, hidden fields, `NEXT_PUBLIC_*`/`REACT_APP_*` env vars, SSR hydration data.

See @client-side-bugs.md for detailed input sources, CSP configuration, CSRF edge cases, and checklists.

## Server-Side Bugs

### SSRF

Any feature that fetches user-provided URLs — webhooks, URL previews, importers, PDF generators — opens an SSRF vector.

**Defenses:** Allowlist domains. Resolve DNS before requesting. Reject private/internal IPs. Block cloud metadata (`169.254.169.254`). Limit redirects.

### File Upload

Validate type (extension + magic bytes), content, and size. Never rely on one check.

**Key attacks:** Extension bypass (`shell.php.jpg`), MIME spoofing, SVG with JS, ZIP slip, polyglot files.

**Secure handling:** Rename to UUID, store outside webroot, serve with `Content-Disposition: attachment` and `nosniff`.

### SQL Injection

Use parameterized queries. Always. ORM raw query methods are still vulnerable.

**Can't parameterize:** ORDER BY, table/column names — must whitelist. Also escape LIKE wildcards (`%`, `_`).

### XXE

Disable DTD processing and external entity resolution in XML parsers. Applies to SOAP, XML uploads, DOCX/XLSX, SVG, SAML.

### Path Traversal

Never use user input directly in file paths. Canonicalize with `realpath`, validate result starts with base directory. Prefer indirect references (map keys to paths).

See @server-side-bugs.md for bypass technique tables, per-language XXE configs, and implementation checklists.

## Open Redirect

Validate redirect URLs against an allowlist, or accept only relative paths.

**Common bypasses:** `@` symbol, subdomain abuse, protocol tricks, double encoding, unicode homographs, data URLs.

See @open-redirect.md for the full bypass table and prevention patterns.

## Password Security

- Minimum 8 chars (12+ recommended), no max (or 128)
- Allow all characters, don't require specific types
- Hash with Argon2id, bcrypt, or scrypt — never MD5/SHA1/SHA256

## Security Headers

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; connect-src 'self' https://api.yourdomain.com; frame-ancestors 'none'; base-uri 'self'; form-action 'self'
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: strict-origin-when-cross-origin
Cache-Control: no-store  (for sensitive pages)
```

When unsure, choose the more restrictive option and document why in a comment.
