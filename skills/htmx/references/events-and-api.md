# htmx 4.x Events & JavaScript API Reference

## Event Naming Convention

All events follow: **`htmx:phase:action[:sub-action]`**

All errors are consolidated into `htmx:error`. All events provide a `ctx` object in `detail`.

## Events (Complete List)

### Request Lifecycle

| Event | When | Key Detail Properties |
|-------|------|----------------------|
| `htmx:config:request` | Before request, modify config | `ctx` (full request context) -- preventDefault to cancel |
| `htmx:confirm` | After trigger, before request | `ctx`, `issueRequest()`, `dropRequest()` for async confirm |
| `htmx:before:request` | Before fetch send | `ctx` -- preventDefault to cancel |
| `htmx:before:response` | Before response body is read | `ctx` -- preventDefault to cancel |
| `htmx:after:request` | After response text read | `ctx` -- preventDefault to cancel swap |
| `htmx:finally:request` | Always fires after request (success or failure) | `ctx` |
| `htmx:error` | Any error (network, timeout, swap) | `ctx`, `error` |
| `htmx:abort` | Send to element to abort its request | (send this, don't listen for it) |

### Swap Lifecycle

| Event | When | Key Detail Properties |
|-------|------|----------------------|
| `htmx:before:swap` | Before DOM swap | `ctx`, `tasks` (array of swap tasks) -- preventDefault to cancel |
| `htmx:after:swap` | After all swaps complete | `ctx` |
| `htmx:before:settle` | Before settle phase (CSS transitions) | `task`, `newContent`, `settleTasks` |
| `htmx:after:settle` | After settle phase | `task`, `newContent`, `settleTasks` |

### View Transitions

| Event | When |
|-------|------|
| `htmx:before:viewTransition` | Before a view transition starts |
| `htmx:after:viewTransition` | After a view transition completes |

### Node Processing

| Event | When |
|-------|------|
| `htmx:before:process` | Before htmx processes a node tree |
| `htmx:after:process` | After htmx processes a node tree |
| `htmx:before:init` | Before htmx initializes a single element |
| `htmx:after:init` | After htmx initializes a single element |
| `htmx:before:cleanup` | Before htmx removes event listeners from element |
| `htmx:after:cleanup` | After htmx cleanup completes |

### History

| Event | When |
|-------|------|
| `htmx:before:history:update` | Before push/replace URL |
| `htmx:after:history:update` | After push/replace URL |
| `htmx:after:history:push` | After URL pushed to history |
| `htmx:after:history:replace` | After URL replaced in history |
| `htmx:before:history:restore` | Before history restoration (back/forward) |

### Morph

| Event | When |
|-------|------|
| `htmx:before:morph:node` | Before morphing a single node -- return false to skip |

---

## The `ctx` Object

All request/response events share a unified `ctx` object:

```javascript
ctx = {
    sourceElement,      // Element that triggered the request
    sourceEvent,        // Original DOM event
    status,             // "created" | "issuing" | "queued" | "dropped" | "response received" | "swapped" | "error: ..."
    select,             // hx-select value
    selectOOB,          // hx-select-oob value
    target,             // Resolved target element
    swap,               // hx-swap value
    push,               // hx-push-url value
    replace,            // hx-replace-url value
    transition,         // View Transitions enabled
    confirm,            // hx-confirm value
    vals,               // Parsed hx-vals object
    title,              // Extracted <title> text
    text,               // Raw response text
    request: {
        action,         // URL
        method,         // HTTP method
        headers,        // Request headers object
        body,           // FormData or URLSearchParams
        signal,         // AbortController signal
        abort(),        // Abort the request
        credentials,    // fetch credentials mode
        mode,           // fetch mode (same-origin, cors, etc.)
        timeout,        // Timeout value
        form,           // Associated form element
        submitter,      // Submit button element
        validate        // Whether to validate
    },
    response: {         // Only after response received
        raw,            // Native Response object
        status,         // HTTP status code
        headers         // Response headers
    },
    hx: {               // Parsed HX-* response headers
        trigger,        // HX-Trigger value
        redirect,       // HX-Redirect value
        refresh,        // HX-Refresh value
        pushurl,        // HX-Push-Url value
        replaceurl,     // HX-Replace-Url value
        retarget,       // HX-Retarget value
        reswap,         // HX-Reswap value
        reselect,       // HX-Reselect value
        location        // HX-Location value
    },
    fetch               // Override to intercept fetch (used by download extension)
}
```

---

## Event Handling Patterns

### Listen with JavaScript

```javascript
// On specific element
htmx.on('#my-form', 'htmx:before:request', function(evt) {
    console.log('Request context:', evt.detail.ctx);
});

// On document (catch all)
document.addEventListener('htmx:after:swap', function(evt) {
    console.log('Swapped:', evt.detail.ctx.target);
});
```

### Inline with hx-on

```html
<!-- Full event name required (no :: shorthand in v4) -->
<button hx-get="/api"
        hx-on:htmx:before:request="showSpinner()"
        hx-on:htmx:after:swap="hideSpinner()"
        hx-on:htmx:error="showError(event)">
    Load
</button>
```

### Modify Requests via config:request

```javascript
document.addEventListener('htmx:config:request', function(evt) {
    let ctx = evt.detail.ctx;
    ctx.request.headers['X-CSRF-Token'] = getCsrfToken();
});
```

### Custom Confirm Dialog

```javascript
document.addEventListener('htmx:confirm', function(evt) {
    evt.preventDefault();
    myCustomDialog(evt.detail.ctx.confirm).then(confirmed => {
        if (confirmed) evt.detail.issueRequest();
        else evt.detail.dropRequest();
    });
});
```

### Abort a Request

```javascript
htmx.trigger(document.getElementById('my-element'), 'htmx:abort');
```

---

## JavaScript API (Complete)

### AJAX Requests

Returns a **Promise** that resolves after content is swapped.

```javascript
// Simple
htmx.ajax('GET', '/api/data', '#target');

// With config
htmx.ajax('POST', '/api/submit', {
    target: '#result',
    swap: 'innerHTML',
    values: { key: 'value' },
    headers: { 'X-Custom': 'header' },
    source: '#form-element',
    event: originalEvent
});

// Await completion
await htmx.ajax('GET', '/content', '#target');
```

### DOM Queries

```javascript
htmx.find('#id');                    // Single element
htmx.find(parentElt, '.selector');   // Within parent
htmx.findAll('.class');              // All matching
htmx.findAll(parentElt, '.class');   // All within parent
```

### Event Management

```javascript
// Add listener (returns callback for removal via removeEventListener)
let handler = htmx.on('#elt', 'click', (e) => { ... });
htmx.on('htmx:after:swap', (e) => { ... }); // document-level

// Run callback when new content is processed
htmx.onLoad(function(elt) {
    initializeWidgets(elt);
});

// Trigger custom event
htmx.trigger('#elt', 'myCustomEvent', { key: 'value' });
```

### Promises

```javascript
// Wait for event
let evt = await htmx.forEvent('htmx:after:swap', 5000); // 5s timeout
let evt = await htmx.forEvent('htmx:after:swap', 5000, myElement);

// Delay
await htmx.timeout('500ms');
await htmx.timeout(1000);
```

### CSS Class Helpers

```javascript
htmx.takeClass(element, 'selected');           // removes from siblings, adds to element
htmx.takeClass(element, 'active', container);  // within specific container
```

### Processing & Utilities

```javascript
htmx.process(elt);              // Initialize htmx on dynamically added content
htmx.parseInterval('500ms');    // 500
htmx.parseInterval('2s');       // 2000
htmx.parseInterval('1m');       // 60000
```

### Extensions

```javascript
htmx.registerExtension('my-ext', {
    init(internalAPI) { ... },

    // Extension hook methods (use underscores, match event names):
    htmx_before_request(elt, detail) { ... },
    htmx_after_swap(elt, detail) { ... },
    htmx_config_request(elt, detail) { ... },
    htmx_before_settle(elt, detail) { ... },

    // Custom swap style handler
    handle_swap(style, target, fragment, swapSpec) {
        if (style === 'mySwap') { /* ... */ return true; }
        return false;
    },

    // Custom <hx-*> template tag handler
    htmx_process_mytype(templateElt, detail) { ... }
});
```

### Debugging

```javascript
htmx.config.logAll = true;   // Log every htmx event to console
```

---

## Event Name Mapping (v2 -> v4)

| htmx 2.x | htmx 4.x |
|-----------|-----------|
| `htmx:afterOnLoad` | `htmx:after:init` |
| `htmx:afterProcessNode` | `htmx:after:init` |
| `htmx:afterRequest` | `htmx:after:request` |
| `htmx:afterSettle` | `htmx:after:swap` |
| `htmx:afterSwap` | `htmx:after:swap` |
| `htmx:beforeCleanupElement` | `htmx:before:cleanup` |
| `htmx:beforeHistorySave` | `htmx:before:history:update` |
| `htmx:beforeHistoryUpdate` | `htmx:before:history:update` |
| `htmx:beforeOnLoad` | `htmx:before:init` |
| `htmx:beforeProcessNode` | `htmx:before:process` |
| `htmx:beforeRequest` | `htmx:before:request` |
| `htmx:beforeSwap` | `htmx:before:swap` |
| `htmx:beforeTransition` | `htmx:before:viewTransition` |
| `htmx:configRequest` | `htmx:config:request` |
| `htmx:historyCacheMiss` | `htmx:before:history:restore` |
| `htmx:historyRestore` | `htmx:before:history:restore` |
| `htmx:load` | `htmx:after:init` |
| `htmx:pushedIntoHistory` | `htmx:after:history:push` |
| `htmx:replacedInHistory` | `htmx:after:history:replace` |
| `htmx:responseError` | `htmx:error` |
| `htmx:sendError` | `htmx:error` |
| `htmx:swapError` | `htmx:error` |
| `htmx:targetError` | `htmx:error` |
| `htmx:timeout` | `htmx:error` |

**Removed events (no replacement):**
- `htmx:validation:validate/failed/halted` -- use native form validation
- `htmx:xhr:loadstart/progress/loadend/abort` -- htmx uses fetch(), not XHR
- `htmx:oobBeforeSwap/oobAfterSwap` -- use `htmx:before:swap`/`htmx:after:swap`
- `htmx:prompt` -- use `hx-confirm` with `js:` prefix
