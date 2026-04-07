# htmx 4.x Error Handling Reference

## Key Difference from v2

**htmx 4 swaps ALL HTTP responses by default.** Only `204` and `304` skip swap.

This means 4xx/5xx responses get swapped into the target -- design your error responses as swap content.

## Error Types

### 1. HTTP Error Responses (4xx, 5xx)

**Option A: Design error responses as HTML (recommended)**

Since v4 swaps all responses, return proper HTML for error codes:

```python
# Server returns HTML with 422 status
@app.post("/submit")
def submit():
    if not valid:
        return render_template("validation_errors.html"), 422
    return render_template("success.html")
```

The error HTML swaps directly into the target -- no extra client config needed.

**Option B: `hx-status` attribute (per-code control)**

```html
<form hx-post="/save"
      hx-target="#result"
      hx-status:422="swap:innerHTML target:#errors select:#validation-errors"
      hx-status:5xx="swap:none push:false"
      hx-status:401="swap:none">
</form>
```

Available config keys: `swap:`, `target:`, `select:`, `push:`, `replace:`, `transition:`

Wildcard support: exact (`404`), single-digit (`50x`), range (`5xx`). Evaluated by specificity.

**Option C: Global noSwap config**

Restore v2 behavior where errors don't swap:
```javascript
htmx.config.noSwap = [204, 304, '4xx', '5xx'];
```

Or be selective:
```javascript
htmx.config.noSwap = [204, 304, '5xx']; // swap 4xx, skip 5xx
```

**Option D: htmx:before:swap event**

```javascript
document.addEventListener('htmx:before:swap', function(evt) {
    let status = evt.detail.ctx.response.status;
    if (status === 401) {
        evt.preventDefault(); // cancel swap
        window.location.href = '/login';
    }
});
```

### 2. Network Errors

Fires `htmx:error` when the request fails (fetch rejects):

```javascript
document.addEventListener('htmx:error', function(evt) {
    let ctx = evt.detail.ctx;
    if (ctx.status.startsWith('error:')) {
        showToast('Network error. Check your connection.', 'error');
    }
});
```

### 3. Timeout Errors

Default timeout is 60 seconds. Configure per-element or globally:

```html
<!-- Per element -->
<div hx-get="/slow-api" hx-config="timeout:120000">Load</div>
```

```javascript
// Global
htmx.config.defaultTimeout = 120000; // 2 minutes
// Or disable: htmx.config.defaultTimeout = 0;
```

Timeouts fire `htmx:error` with the abort error.

---

## Comprehensive Error Handler

```javascript
// Unified error handler (replaces v2's separate sendError/responseError/timeout events)
document.addEventListener('htmx:error', function(evt) {
    let ctx = evt.detail.ctx;
    let error = evt.detail.error;
    console.error('htmx error:', error, ctx);
    showToast('Something went wrong. Please try again.', 'error');
});

// After every request (success or failure)
document.addEventListener('htmx:after:request', function(evt) {
    let ctx = evt.detail.ctx;
    let status = ctx.response?.status;
    if (status && status >= 400) {
        console.warn('Request failed:', status);
    }
});

// Always runs (like finally block)
document.addEventListener('htmx:finally:request', function(evt) {
    // Cleanup, analytics, etc.
});
```

---

## Server-Side Error Patterns

### Return Partial HTML for Errors

Server returns error HTML with appropriate status code -- htmx v4 swaps it automatically:

```html
<form hx-post="/api/create" hx-target="#result"
      hx-status:422="target:#form-errors">
```

Server (422 response):
```html
<ul class="errors">
    <li>Email is required</li>
    <li>Password must be 8+ characters</li>
</ul>
```

### Use HX-Retarget Header

Server can override the target element:

```
HTTP/1.1 422 Unprocessable Entity
HX-Retarget: #form-errors
HX-Reswap: innerHTML

<ul><li>Validation failed</li></ul>
```

### Use HX-Trigger for Toast Notifications

```
HTTP/1.1 200 OK
HX-Trigger: {"showToast": {"message": "Saved!", "type": "success"}}

<div>Updated content</div>
```

```javascript
document.addEventListener('showToast', function(evt) {
    showToast(evt.detail.message, evt.detail.type);
});
```

### Redirect on Auth Errors

```
HTTP/1.1 401 Unauthorized
HX-Redirect: /login
```

Or with HX-Location for SPA-style:
```
HX-Location: path:/login
```

---

## Retry Pattern

htmx doesn't have built-in retry. Implement via events:

```javascript
document.addEventListener('htmx:error', function(evt) {
    let elt = evt.detail.ctx.sourceElement;
    let retries = parseInt(elt.dataset.retries || '0');
    if (retries < 3) {
        elt.dataset.retries = retries + 1;
        setTimeout(() => htmx.trigger(elt, 'retry'), 1000 * (retries + 1));
    } else {
        elt.dataset.retries = '0';
        showToast('Failed after 3 retries', 'error');
    }
});
```

---

## Loading/Error State CSS

```css
/* Show spinner during request */
.htmx-request .loading { display: inline-block; }
.loading { display: none; }

/* Dim target during request */
[hx-get].htmx-request, [hx-post].htmx-request {
    opacity: 0.5;
    pointer-events: none;
}

/* Error state styling */
.error-container:not(:empty) {
    padding: 1rem;
    border: 1px solid #dc3545;
    border-radius: 4px;
    background: #f8d7da;
    color: #721c24;
}
```

---

## Migration Notes from v2

| v2 Pattern | v4 Equivalent |
|------------|---------------|
| `response-targets` extension | `hx-status:CODE` attribute or just let errors swap |
| `htmx:beforeSwap` + `shouldSwap` | `hx-status:CODE` or `htmx:before:swap` + preventDefault |
| `htmx.config.responseHandling` | `htmx.config.noSwap` + `hx-status` |
| `htmx:responseError` event | `htmx:error` event |
| `htmx:sendError` event | `htmx:error` event |
| `htmx:timeout` event | `htmx:error` event |
| `htmx:swapError` event | `htmx:error` event |
| `detail.xhr` | `detail.ctx.response.raw` (native Response object, not XHR) |
| `detail.xhr.status` | `detail.ctx.response.status` |
