# htmx 4.x Extensions Reference

## Installing Extensions

Load htmx first, then extensions as separate scripts. No `hx-ext` attribute needed:

```html
<script src="https://cdn.jsdelivr.net/npm/htmx.org@next/dist/htmx.min.js"></script>
<script src="/path/to/ext/hx-sse.js"></script>
```

Restrict which extensions can load:
```html
<meta name="htmx-config" content='{"extensions": "sse, ws"}'>
```

Register custom extensions:
```javascript
htmx.registerExtension('my-ext', {
    init(internalAPI) { /* access internal methods */ },
    htmx_before_request(elt, detail) { /* hook into events */ },
    handle_swap(style, target, fragment, swapSpec) { /* custom swap */ }
});
```

---

## SSE (Server-Sent Events) -- `hx-sse.js`

**htmx 4 SSE uses `fetch()` + `ReadableStream`** (not `EventSource`), enabling request bodies, custom headers, and all HTTP methods.

### Attributes

| Attribute | Purpose |
|-----------|---------|
| `hx-sse:connect="<url>"` | SSE server endpoint URL |
| `hx-sse:swap="<event>"` | Swap content on named event(s) |
| `hx-sse:close="<event>"` | Close connection on this event |
| `hx-trigger="sse:<event>"` | Trigger HTTP request on SSE event |

### Basic Connection

```html
<div hx-sse:connect="/updates" hx-sse:swap="message">
    Content updates in real-time
</div>
```

### Named vs Unnamed Events

**Unnamed** (server sends `data:` without `event:`) -- use `sse-swap="message"`:
```
data: <div>New content</div>
```

**Named** (server sends with `event:`) -- match exact name:
```
event: notification
data: <div>Alert!</div>
```
```html
<div hx-sse:swap="notification"></div>
```

### Multiple Events

```html
<div hx-sse:connect="/stream">
    <div hx-sse:swap="messages">Messages here</div>
    <div hx-sse:swap="alerts">Alerts here</div>
</div>
```

### Trigger HTTP Requests from SSE

```html
<div hx-sse:connect="/events">
    <div hx-get="/chatroom" hx-trigger="sse:chatter">
        Reloads chatroom when "chatter" SSE event arrives
    </div>
</div>
```

### Reconnection & Background Behavior

Automatic reconnection on disconnect. Supports `Last-Event-ID` for resumption. Pauses when app is backgrounded (iOS Safari fix).

### SSE Events

| Event | Description |
|-------|-------------|
| `htmx:sseOpen` | Connection established |
| `htmx:sseError` | Connection error |
| `htmx:sseBeforeMessage` | Before swap (preventDefault to skip) |
| `htmx:sseMessage` | After swap |
| `htmx:sseClose` | Connection closed |

**Note:** If using `metaCharacter`, SSE attribute names adjust accordingly (e.g., `hx-sse-connect` with `-`).

---

## WebSocket -- `hx-ws.js`

### Attributes

| Attribute | Purpose |
|-----------|---------|
| `hx-ws:connect="<url>"` | WebSocket server URL |
| `hx-ws:send` | Mark form/element for sending data |

### Bidirectional Communication

```html
<div hx-ws:connect="ws://localhost:8080">
    <div id="chat-messages"></div>

    <form hx-ws:send>
        <input name="message" type="text">
        <button type="submit">Send</button>
    </form>
</div>
```

Server sends HTML responses that are swapped into the DOM. Supports raw HTML messages with cancelable events. Script tag processing is supported.

Uses `htmx.swap()` internally for consistent swap behavior.

---

## Head Support -- `hx-head.js`

Merges `<head>` content (styles, scripts, meta) from AJAX responses. Supports async head processing.

```html
<script src="/path/to/ext/hx-head.js"></script>
<!-- Then use hx-boost or full-page AJAX normally -->
<a hx-get="/page2" hx-target="body" hx-push-url="true">Page 2</a>
```

---

## Alpine.js Compatibility -- `hx-alpine-compat.js`

Initializes Alpine.js on swapped fragments before they enter the DOM. Handles morph operations correctly with Alpine reactive bindings.

```html
<script src="/path/to/ext/hx-alpine-compat.js"></script>
```

---

## htmx 2.x Compatibility -- `htmx-2-compat.js`

Restores htmx 2.x behavior for gradual migration:

- Implicit attribute inheritance
- Old event names (e.g., `htmx:afterSwap` fires alongside `htmx:after:swap`)
- 4xx/5xx responses don't swap (adds to `noSwap`)

```html
<script src="/path/to/htmx.js"></script>
<script src="/path/to/ext/htmx-2-compat.js"></script>
```

Configure via `htmx.config.compat`:
- `doNotTriggerOldEvents` -- skip re-triggering old event names
- `useExplicitInheritance` -- don't revert to implicit inheritance
- `swapErrorResponseCodes` -- don't add 4xx/5xx to noSwap
- `suppressInheritanceLogs` -- suppress console warnings about implicit inheritance

---

## Optimistic UI -- `hx-optimistic.js`

Shows expected content from a template before the server responds:

```html
<script src="/path/to/ext/hx-optimistic.js"></script>

<template id="optimistic-content">
    <div>Saving...</div>
</template>

<button hx-post="/save" hx-target="#result" hx-optimistic="#optimistic-content">
    Save
</button>
```

The template content is shown immediately, then replaced by the server response. On error, the optimistic content is removed.

---

## Preload -- `hx-preload.js`

Prefetch content on mouseover or mousedown for near-instant page loads:

```html
<script src="/path/to/ext/hx-preload.js"></script>

<a href="/page2" preload="mouseover">Page 2 (preloads on hover)</a>
<button hx-get="/data" preload>Load (preloads on mousedown)</button>
```

---

## Upsert -- `hx-upsert.js`

Updates existing elements by ID and inserts new ones, preserving unmatched elements. Useful for live lists:

```html
<script src="/path/to/ext/hx-upsert.js"></script>

<div hx-get="/items" hx-swap="upsert">
    <div id="item-1">Old item 1</div>
    <div id="item-2">Old item 2</div>
</div>
```

**Swap modifiers:**
- `key:<attr>` -- attribute name for matching (default: `id`)
- `sort` / `sort:desc` -- sort elements
- `prepend` -- prepend new elements without keys (default: append)

**As `<hx-upsert>` partial:**
```html
<hx-upsert hx-target="#list" key="data-id" sort>
    <div id="item-1" data-id="1">Updated item 1</div>
    <div id="item-3" data-id="3">New item 3</div>
</hx-upsert>
```

---

## Download -- `hx-download.js`

Triggers file download instead of DOM swap, with streaming progress events:

```html
<script src="/path/to/ext/hx-download.js"></script>

<button hx-get="/file.pdf" hx-swap="download">Download PDF</button>
```

**Events:**

| Event | Detail |
|-------|--------|
| `htmx:download:start` | `{total}` |
| `htmx:download:progress` | `{loaded, total, percent}` |
| `htmx:download:complete` | `{filename, size}` |

Works by overriding `ctx.fetch` to stream the response body, then creating a blob download link.

---

## Browser Indicator -- `hx-browser-indicator.js`

Shows the browser's native loading indicator (tab spinner) during htmx requests:

```html
<script src="/path/to/ext/hx-browser-indicator.js"></script>
```

---

## Polling Tags (PTag) -- `hx-ptag.js`

Per-element conditional polling. Server can skip swaps when content hasn't changed:

```html
<script src="/path/to/ext/hx-ptag.js"></script>

<div hx-get="/news" hx-trigger="every 3s" hx-ptag="initial-value">
    Latest News...
</div>
```

- Sends `HX-PTag` request header with stored tag value
- Server responds with `HX-PTag` response header (new tag) or `304` if unchanged
- Replaces the old etag-based behavior from earlier alphas

---

## Targets -- `hx-targets.js`

Multi-error targeting by HTTP status code (similar to v2's `response-targets` extension):

```html
<script src="/path/to/ext/hx-targets.js"></script>
```

---

## Writing Custom Extensions

Extensions register via `htmx.registerExtension()` and hook into the lifecycle via method names matching event patterns (colons replaced with underscores):

```javascript
htmx.registerExtension('my-ext', {
    // Called once with internal API access
    init(internalAPI) {
        // internalAPI provides: attributeValue, parseTriggerSpecs,
        // determineMethodAndAction, createRequestContext, collectFormData,
        // getAttributeObject, insertContent, morph, isSoftMatch,
        // onTrigger, htmxProp, triggerHtmxEvent
    },

    // Hook into any htmx event (underscores replace colons)
    htmx_before_request(elt, detail) {
        // Modify detail.ctx, return false to cancel
    },

    htmx_config_request(elt, detail) {
        // Add headers, modify request config
        detail.ctx.request.headers['X-Custom'] = 'value';
    },

    // Custom swap style
    handle_swap(style, target, fragment, swapSpec) {
        if (style === 'myCustomSwap') {
            // Perform custom DOM manipulation
            return true; // or return array of new elements
        }
        return false; // not handled
    },

    // Custom <hx-*> template tag types
    htmx_process_mytype(templateElt, detail) {
        let {ctx, tasks} = detail;
        tasks.push({
            type: 'partial',
            fragment: templateElt.content.cloneNode(true),
            target: '#my-target',
            swapSpec: {style: 'innerHTML'},
            sourceElement: ctx.sourceElement
        });
    }
});
```
