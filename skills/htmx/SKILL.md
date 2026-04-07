---
name: htmx
description: Use when writing, reviewing, or debugging htmx code -- building hypermedia-driven UIs with hx-get, hx-post, hx-swap, hx-trigger, hx-target, hx-boost, hx-partial, hx-status, hx-action, morph swaps, SSE, WebSockets, htmx events, htmx extensions, or any project using htmx attributes. Covers htmx 4.x API.
---

# htmx 4.x Complete Reference

htmx gives access to AJAX, CSS Transitions, WebSockets, and Server-Sent Events directly in HTML using attributes. No JavaScript needed for most interactive patterns.

**htmx 4 uses `fetch()` (not XMLHttpRequest), has explicit attribute inheritance, swaps all HTTP responses by default, and includes a built-in morph algorithm.**

## Installation

```html
<!-- CDN -->
<script src="https://cdn.jsdelivr.net/npm/htmx.org@next/dist/htmx.min.js"></script>

<!-- NPM -->
npm install htmx.org@next
```

Extensions are loaded as separate scripts (no `hx-ext` attribute needed):
```html
<script src="https://cdn.jsdelivr.net/npm/htmx.org@next/dist/htmx.min.js"></script>
<script src="/path/to/ext/hx-sse.js"></script>
```

Restrict allowed extensions via config:
```html
<meta name="htmx-config" content='{"extensions": "sse, ws"}'>
```

## Core Pattern

Any element can issue HTTP requests and swap content into the DOM:

```html
<button hx-get="/api/data" hx-target="#result" hx-swap="innerHTML" hx-trigger="click">
  Load
</button>
<div id="result"></div>
```

## HTTP Verbs

| Attribute | Method |
|-----------|--------|
| `hx-get` | GET |
| `hx-post` | POST |
| `hx-put` | PUT |
| `hx-patch` | PATCH |
| `hx-delete` | DELETE |

**New in v4:** `hx-action` + `hx-method` to separate URL from method:
```html
<button hx-action="/api/item/1" hx-method="PATCH">Update</button>
```

**Note:** `hx-get` and `hx-delete` do NOT include enclosing form inputs. Use `hx-include="closest form"` if needed.

## Targeting

- `hx-target="#id"` -- CSS selector for swap target
- `hx-target="this"` -- the element itself (default for non-boosted)
- `hx-target="closest div"` -- nearest ancestor matching selector
- `hx-target="find .child"` -- first descendant matching selector
- `hx-target="next .sibling"` / `hx-target="previous .sibling"`
- `hx-target="body"` / `hx-target="document"` / `hx-target="window"`
- `hx-target="host"` -- shadow DOM host element
- `hx-select="#content"` -- swap only matching portion of response

## Swapping (`hx-swap`)

**Styles:**

| Style | Effect |
|-------|--------|
| `innerHTML` | Replace children (default) |
| `outerHTML` | Replace entire element |
| `beforebegin` / `before` | Insert before element |
| `afterbegin` / `prepend` | Insert as first child |
| `beforeend` / `append` | Insert as last child |
| `afterend` / `after` | Insert after element |
| `innerMorph` | Morph children (preserves DOM state) |
| `outerMorph` | Morph entire element |
| `textContent` | Set text content (no HTML parsing) |
| `delete` | Remove target element |
| `none` | No swap |

**Modifiers** (space-separated after value, `key:value` syntax):
- `swap:<delay>` / `settle:<delay>` -- timing for CSS transitions
- `scroll:top` / `scroll:bottom` -- scroll target after swap
- `scrollTarget:<selector>` -- element to scroll
- `show:top` / `show:bottom` -- scroll target into viewport
- `showTarget:<selector>` -- element to scroll into view
- `transition:true|false` -- enable/disable View Transitions for this swap
- `ignoreTitle:true` -- don't update document title
- `focusScroll:true` -- scroll focused element into view
- `target:<selector>` -- override target for this swap

```html
<div hx-get="/data" hx-swap="innerHTML swap:300ms settle:100ms scroll:top transition:true">
```

## Triggers (`hx-trigger`)

**Default triggers:** `input`/`textarea`/`select` -> `change`, `form` -> `submit`, `input[type=submit]` -> `click`, everything else -> `click`

**Special triggers:**
- `load` -- on element initialization
- `revealed` -- when scrolled into viewport (fires once)
- `intersect` -- IntersectionObserver-based (`opts.root:<selector>`, `opts.threshold:<0.0-1.0>`)
- `every <interval>` -- polling (`every 1s`, `every 5s`)

**Modifiers** (space-separated `key:value` syntax):
- `once` -- fire only once
- `changed` -- fire only if value changed
- `delay:<time>` -- debounce (`delay:500ms`)
- `throttle:<time>` -- throttle (`throttle:1s`)
- `from:<selector>` -- listen on different element (`from:body`, `from:document`, `from:window`, `from:closest form`)
- `target:<selector>` -- filter by event.target
- `consume` -- stop event propagation

**Filters:** `hx-trigger="click[ctrlKey]"`, `hx-trigger="keyup[key=='Enter']"`

**Multiple triggers:** `hx-trigger="load, click delay:1s, keyup changed delay:500ms"`

## Attribute Inheritance

**htmx 4 uses explicit inheritance.** Attributes do NOT inherit to children by default.

Use `:inherited` suffix to opt in:
```html
<div hx-confirm:inherited="Are you sure?">
    <button hx-delete="/item/1">Delete</button> <!-- inherits hx-confirm -->
</div>

<div hx-target:inherited="#result">
    <button hx-get="/a">A</button> <!-- targets #result -->
    <button hx-get="/b">B</button> <!-- targets #result -->
</div>
```

Use `:append` to extend an inherited value:
```html
<div hx-include:inherited="#global-fields">
    <form hx-include:inherited:append=".extra">...</form>
</div>
```

Revert to v2 implicit inheritance: `htmx.config.implicitInheritance = true`

## Synchronization (`hx-sync`)

Control concurrent requests: `hx-sync="<selector>:<strategy>"`

| Strategy | Behavior |
|----------|----------|
| `queue first` | Queue first request only (default) |
| `drop` | Ignore if request in flight |
| `abort` | Drop if in flight; abort this if new request arrives |
| `replace` | Abort current request, replace with this one |
| `queue last` | Queue last request only |
| `queue all` | Queue all requests |

```html
<input hx-get="/search" hx-trigger="keyup changed delay:300ms" hx-sync="this:replace">
```

## Multi-Target Responses

### `<hx-partial>` (new in v4)

Target multiple elements from one response. Cleaner alternative to OOB swaps:

```html
<!-- Server response -->
<hx-partial hx-target="#messages" hx-swap="beforeend">
    <div>New message</div>
</hx-partial>

<hx-partial hx-target="#count">
    <span>5</span>
</hx-partial>
```

Each `<hx-partial>` has its own `hx-target` and `hx-swap`. The main content swaps first, then partials in document order.

### Out-of-Band Swaps (`hx-swap-oob`)

```html
<!-- Server response -->
<div id="main">Main content</div>
<div id="sidebar" hx-swap-oob="true">Updated sidebar</div>
<div id="notifications" hx-swap-oob="afterbegin">New notification</div>
```

Client-side with `hx-select-oob`:
```html
<button hx-get="/data" hx-select="#main" hx-select-oob="#sidebar,#notifications:afterbegin">
```

**Note:** In v4, main content swaps first, then OOB/partials (reversed from v2).

## Status Code Handling (`hx-status`)

**htmx 4 swaps ALL responses by default** (only 204 and 304 skip swap).

Control behavior per HTTP status code:

```html
<form hx-post="/save"
      hx-status:422="swap:innerHTML target:#errors select:#validation-errors"
      hx-status:5xx="swap:none push:false">
</form>
```

Available config keys: `swap:`, `target:`, `select:`, `push:`, `replace:`, `transition:`

Wildcards: exact (`404`), single-digit (`50x`), range (`5xx`). Evaluated by specificity.

Revert to v2 behavior: `htmx.config.noSwap = [204, 304, '4xx', '5xx']`

## Request Configuration

- `hx-vals='{"key": "value"}'` -- add JSON values
- `hx-vals="js:{key: computeValue()}"` -- dynamic values via `js:` prefix
- `hx-headers='{"X-Custom": "value"}'` -- custom headers (supports `js:` prefix)
- `hx-include="[name='token']"` -- include extra inputs
- `hx-encoding="multipart/form-data"` -- for file uploads
- `hx-config="credentials:include, timeout:5000"` -- per-element request config
- `hx-validate="true"` -- force HTML5 form validation

## Indicators

```html
<button hx-get="/api" hx-indicator="#spinner">Load</button>
<span id="spinner" class="htmx-indicator">Loading...</span>
```

CSS class `htmx-request` is added during requests. Default CSS fades in `.htmx-indicator` elements.

`hx-disable="this"` -- disable element during request (was `hx-disabled-elt` in v2).

## Boosting

`hx-boost="true"` on a container converts `<a>` and `<form>` children to AJAX with history support.

Advanced config overrides:
```html
<div hx-boost="swap:outerMorph target:#content push:true">
```

## History

- `hx-push-url="true"` / `hx-push-url="/custom-url"` -- push to browser history
- `hx-replace-url="true"` -- replace current URL (no history entry)
- `hx-preserve` -- keep element unchanged across swaps (requires `id`)

**No localStorage cache in v4.** Back/forward triggers a re-fetch (or `location.reload()` with `htmx.config.history = "reload"`).

## Confirmation

```html
<button hx-delete="/item/1" hx-confirm="Are you sure?">Delete</button>

<!-- Async/custom confirm with js: prefix -->
<button hx-delete="/item/1" hx-confirm="js:myCustomDialog('Delete this?')">Delete</button>
```

The `htmx:confirm` event fires with `issueRequest()` / `dropRequest()` callbacks for fully custom dialogs.

## Ignore Processing

`hx-ignore` disables htmx processing on an element and its children (was `hx-disable` in v2).

## Inline Event Handling (`hx-on`)

```html
<button hx-get="/api" hx-on:htmx:before:request="showSpinner()">
<div hx-on:click="this.classList.toggle('active')">
```

**Note:** The `hx-on::` double-colon shorthand from v2 no longer works. Use full event names.

## Etag / Conditional Requests

Use the `ptag` extension for per-element conditional polling:

```html
<div hx-get="/news" hx-trigger="every 3s" hx-ptag="initial-value">Latest News</div>
```

Server responds with `HX-PTag` header. Next request sends `HX-PTag` request header. Server returns 304 if unchanged.

## JSX Compatibility

Replace `:` in attribute names with a custom character:
```js
htmx.config.metaCharacter = "-";
// hx-confirm-inherited instead of hx-confirm:inherited
```

## CSS Transitions

htmx applies CSS classes during the swap lifecycle:

| Class | When |
|-------|------|
| `htmx-request` | During request (on element or indicator) |
| `htmx-swapping` | Before swap, removed after swap |
| `htmx-settling` | After swap, removed after settle |
| `htmx-added` | On new content before settle |

```css
.fade-me-in.htmx-added { opacity: 0; }
.fade-me-in { opacity: 1; transition: opacity 300ms ease-in; }
```

View Transitions API (disabled by default): `htmx.config.transitions = true`

## Events

All events follow pattern: **`htmx:phase:action[:sub-action]`**

Key events: `htmx:config:request`, `htmx:before:request`, `htmx:after:request`, `htmx:before:swap`, `htmx:after:swap`, `htmx:before:settle`, `htmx:after:settle`, `htmx:error`, `htmx:finally:request`

All errors consolidated in `htmx:error`. All events provide a `ctx` object.

Consult `references/events-and-api.md` for the complete list.

## HTTP Headers

Consult `references/headers-and-config.md` for complete request/response headers and config options.

Key request headers: `HX-Request`, `HX-Source` (`tagName#id`), `HX-Target` (`tagName#id`), `HX-Request-Type` (`full`/`partial`), `HX-Current-URL`, `Accept: text/html`

Key response headers: `HX-Redirect`, `HX-Refresh`, `HX-Retarget`, `HX-Reswap`, `HX-Reselect`, `HX-Trigger`, `HX-Push-Url`, `HX-Replace-Url`

## JavaScript API

| Method | Purpose |
|--------|---------|
| `htmx.ajax(method, url, config)` | Programmatic AJAX request |
| `htmx.on(elt, event, handler)` | Add event listener (returns callback) |
| `htmx.trigger(elt, event, detail)` | Trigger event |
| `htmx.process(elt)` | Initialize htmx on dynamic content |
| `htmx.find(selector)` / `htmx.findAll(selector)` | DOM queries |
| `htmx.swap(ctx)` | Programmatic swap |
| `htmx.onLoad(callback)` | Run on new content (listens `htmx:after:process`) |
| `htmx.takeClass(elt, class, container)` | Remove class from siblings, add to element |
| `htmx.forEvent(event, timeout, on)` | Promise that resolves on event |
| `htmx.timeout(time)` | Promise that resolves after delay |
| `htmx.parseInterval(str)` | Parse time string (`500ms`, `2s`, `1m`) |
| `htmx.registerExtension(name, ext)` | Register extension |

**Removed from v2:** `htmx.addClass/removeClass/toggleClass/closest/remove/off/logAll/logNone/defineExtension` -- use native DOM equivalents.

## Extensions

htmx 4 ships with core extensions. Consult `references/extensions.md` for full documentation.

| Extension | Description |
|-----------|-------------|
| `hx-sse.js` | Server-Sent Events (fetch-based, supports headers/body) |
| `hx-ws.js` | WebSocket bidirectional communication |
| `hx-head.js` | Merge `<head>` content from responses |
| `hx-alpine-compat.js` | Alpine.js compatibility |
| `htmx-2-compat.js` | htmx 2.x backward compatibility layer |
| `hx-optimistic.js` | Optimistic UI from templates |
| `hx-preload.js` | Prefetch on mouseover/mousedown |
| `hx-upsert.js` | Update existing elements by ID, insert new |
| `hx-download.js` | File download with progress events |
| `hx-browser-indicator.js` | Browser native loading indicator |
| `hx-ptag.js` | Per-element conditional polling tags |
| `hx-targets.js` | Multi-error targeting by HTTP status |

## Error Handling

Consult `references/error-handling.md` for patterns covering status codes, network errors, and timeouts.

## Additional Resources

- **`references/extensions.md`** -- Complete extension docs (SSE, WebSocket, download, upsert, optimistic, all extensions)
- **`references/events-and-api.md`** -- All events with ctx properties, JS API details
- **`references/headers-and-config.md`** -- HTTP headers, htmx.config options
- **`references/error-handling.md`** -- Error handling patterns, hx-status, network errors
