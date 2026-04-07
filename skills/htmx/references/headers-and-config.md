# htmx 4.x Headers & Configuration Reference

## HTTP Request Headers (sent by htmx)

| Header | Value | Description |
|--------|-------|-------------|
| `HX-Request` | `true` | Always present on htmx requests |
| `HX-Source` | `tagName#id` | Identifier of triggering element (e.g. `button#submit`) |
| `HX-Target` | `tagName#id` | Identifier of target element |
| `HX-Request-Type` | `full` or `partial` | `full` if target is body or hx-select is used |
| `HX-Current-URL` | URL string | Current browser URL |
| `HX-Boosted` | `true` | Request via `hx-boost` element |
| `HX-History-Restore-Request` | `true` | History restoration after back/forward |
| `Accept` | `text/html` | Explicitly set on all requests |

### Server-Side Detection

```python
# Python/Flask
if request.headers.get('HX-Request'):
    return render_template('partial.html')
return render_template('full_page.html')
```

```go
// Go
if r.Header.Get("HX-Request") == "true" {
    if r.Header.Get("HX-Request-Type") == "partial" {
        // Return partial HTML
    }
}
```

**Removed from v2:** `HX-Trigger` (element id -- replaced by `HX-Source`), `HX-Trigger-Name`, `HX-Prompt`

---

## HTTP Response Headers (sent by server)

| Header | Value | Description |
|--------|-------|-------------|
| `HX-Location` | JSON or URL | Client-side redirect via htmx.ajax (no full reload) |
| `HX-Push-Url` | URL or `false` | Push URL to browser history |
| `HX-Redirect` | URL | Full-page client-side redirect |
| `HX-Refresh` | `true` | Full page refresh |
| `HX-Replace-Url` | URL or `false` | Replace current URL (no history entry) |
| `HX-Reswap` | swap value | Override `hx-swap` on the triggering element |
| `HX-Retarget` | CSS selector | Override `hx-target` on the triggering element |
| `HX-Reselect` | CSS selector | Override `hx-select` on the triggering element |
| `HX-Trigger` | event name or JSON | Trigger client-side event(s) |
| `HX-PTag` | string | Polling tag for conditional requests (ptag extension) |

### HX-Location (SPA-style redirect)

```
HX-Location: /new-page
```

With options (parsed via htmx config syntax):
```
HX-Location: path:/new-page, push:true
```

### HX-Trigger (fire client events from server)

Simple event:
```
HX-Trigger: myEvent
```

Multiple events:
```
HX-Trigger: event1, event2
```

Events with data (JSON or config syntax):
```
HX-Trigger: {"showMessage": {"level": "info", "message": "Saved!"}}
```

Events can target specific elements:
```
HX-Trigger: {"showMessage": {"target": "#toast", "message": "Done"}}
```

Listen on client:
```javascript
document.addEventListener('showMessage', function(e) {
    alert(e.detail.message);
});
```

**Removed from v2:** `HX-Trigger-After-Swap`, `HX-Trigger-After-Settle` -- use `HX-Trigger` instead.

---

## htmx.config (Complete)

Set via JavaScript or `<meta>` tag:

```html
<meta name="htmx-config" content='{"extensions": "sse, ws", "transitions": true}'>
```

Or in JavaScript (before htmx loads, or after):
```javascript
htmx.config.implicitInheritance = true;
htmx.config.noSwap = [204, 304, '4xx', '5xx'];
```

### Core

| Config | Default | Description |
|--------|---------|-------------|
| `logAll` | `false` | Log all events to console |
| `prefix` | `""` | Custom prefix for `hx-` attributes |
| `extensions` | `""` | Comma-separated list of allowed extension names |

### Request Behavior

| Config | Default | Description |
|--------|---------|-------------|
| `defaultTimeout` | `60000` | Request timeout in ms (v2 was `0`) |
| `mode` | `"same-origin"` | Fetch mode (replaces `selfRequestsOnly`) |

### Swap Behavior

| Config | Default | Description |
|--------|---------|-------------|
| `defaultSwap` | `"innerHTML"` | Default swap style (was `defaultSwapStyle` in v2) |
| `defaultSettleDelay` | `1` | Delay before settle in ms (v2 was `20`) |
| `defaultFocusScroll` | `false` | Scroll focused element into view after swap |
| `noSwap` | `[204, 304]` | Status codes that skip swap. Supports wildcards: `'4xx'`, `'5xx'` |
| `transitions` | `false` | Enable View Transitions API globally (was `globalViewTransitions`) |

### CSS Classes

| Config | Default | Description |
|--------|---------|-------------|
| `indicatorClass` | `"htmx-indicator"` | Class for indicator elements |
| `requestClass` | `"htmx-request"` | Applied during request |
| `includeIndicatorCSS` | `true` | Include default indicator styles (was `includeIndicatorStyles`) |

### History

| Config | Default | Description |
|--------|---------|-------------|
| `history` | `true` | Enable history support. Set to `"reload"` for full page reload on back/forward. `false` to disable. (Was `historyEnabled`) |

### Inheritance

| Config | Default | Description |
|--------|---------|-------------|
| `implicitInheritance` | `false` | When `true`, all attributes inherit like v2 |

### Security / CSP

| Config | Default | Description |
|--------|---------|-------------|
| `inlineScriptNonce` | `""` | Nonce for inline scripts |
| `inlineStyleNonce` | `""` | Nonce for inline styles |

### Morphing

| Config | Default | Description |
|--------|---------|-------------|
| `morphIgnore` | `["data-htmx-powered"]` | Attributes to ignore during morph |
| `morphScanLimit` | `10` | Max sibling elements to scan for match during morph |
| `morphSkip` | `""` | CSS selector for elements to skip during morph |
| `morphSkipChildren` | `""` | CSS selector for elements whose children to skip during morph |

### Other

| Config | Default | Description |
|--------|---------|-------------|
| `metaCharacter` | `":"` | Separator character in attribute/event names (for JSX compat) |

---

## Config Changes from v2

### Renamed

| v2 | v4 |
|----|-----|
| `defaultSwapStyle` | `defaultSwap` |
| `globalViewTransitions` | `transitions` |
| `historyEnabled` | `history` |
| `includeIndicatorStyles` | `includeIndicatorCSS` |
| `timeout` | `defaultTimeout` |

### Changed Defaults

| Config | v2 | v4 |
|--------|----|----|
| `defaultTimeout` | `0` (none) | `60000` (60s) |
| `defaultSettleDelay` | `20` | `1` |
| `implicitInheritance` | `true` (implicit) | `false` (explicit) |
| `noSwap` | `[204]` + responseHandling | `[204, 304]` |
| `transitions` | `false` | `false` |

### Removed

`addedClass`, `allowEval`, `allowNestedOobSwaps`, `allowScriptTags`, `attributesToSettle`,
`defaultSwapDelay`, `disableSelector`, `getCacheBusterParam`, `historyCacheSize`, `ignoreTitle`
(per-swap only via `hx-swap`), `methodsThatUseUrlParams`, `refreshOnHistoryMiss`, `responseHandling`
(use `hx-status` and `noSwap`), `scrollBehavior`, `scrollIntoViewOnBoost`, `selfRequestsOnly`
(use `mode`), `settlingClass`, `swappingClass`, `triggerSpecsCache`, `useTemplateFragments`,
`withCredentials` (use `hx-config`), `wsBinaryType`, `wsReconnectDelay`, `disableInheritance`

---

## CSS Classes Applied by htmx

| Class | Applied To | When | Removed |
|-------|-----------|------|---------|
| `htmx-request` | Triggering element or `hx-indicator` target | During request | When request completes |
| `htmx-swapping` | Target element | Before swap | After swap |
| `htmx-settling` | Target element | After swap | After settle delay |
| `htmx-added` | New content elements | After insertion | After settle |
| `htmx-indicator` | Indicator elements | Always (hidden by default CSS) | Visible when ancestor has `htmx-request` |

### CSS Transition Pattern

```css
/* Fade in new content */
.my-element.htmx-added { opacity: 0; }
.my-element { opacity: 1; transition: opacity 0.5s ease-in; }

/* Fade out before swap */
.my-element.htmx-swapping {
    opacity: 0;
    transition: opacity 0.3s ease-out;
}
```

### Indicator Pattern

```css
/* Default: htmx includes these styles automatically */
.htmx-indicator { opacity: 0; visibility: hidden; }
.htmx-request .htmx-indicator,
.htmx-request.htmx-indicator {
    opacity: 1;
    visibility: visible;
    transition: opacity 200ms ease-in;
}
```
