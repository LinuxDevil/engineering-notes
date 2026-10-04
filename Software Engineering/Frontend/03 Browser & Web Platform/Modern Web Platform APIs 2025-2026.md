---
title: "Modern Web Platform APIs 2025-2026"
tags: [frontend, frontend/browser, frontend/whats-new]
aliases: [Modern Web APIs, New Web APIs, Web Platform 2026, Popover API, View Transitions, Navigation API, Interop 2026]
updated: 2026-09-27
---
# Modern Web Platform APIs 2025-2026

<Note>
**TL;DR**

The platform now ships many things we used to build with JS libraries: **popovers, dialogs and invoker buttons** in plain HTML, **view transitions** for animated navigations, the **Navigation API** for SPA routing, **URLPattern** for route matching, **`scheduler.yield()`** for responsive long work, **Speculation Rules** for instant page loads, plus **Trusted Types / Sanitizer**, **ariaNotify**, **WebGPU** and more. This note lists each with its **Baseline status as of 2026-09-27** and a snippet. Rule of thumb: *Newly/Widely* = use it (maybe with a fallback for old browsers); *Limited* = progressive enhancement only, feature-detect.
</Note>

**Baseline reminder**: *Widely* = in all core browsers for 30+ months; *Newly* = in all core browsers since the date shown; *Limited* = at least one core browser missing. Status data is from webstatus.dev / web-features as of 2026-09-27.

## Status at a glance

| API | Status (2026-09-27) | Replaces / helps with |
|---|---|---|
| Popover API | **Newly** (2025-01-27) | Custom tooltip/menu/toast JS, z-index wars |
| `popover="hint"` | Limited: Chrome 151, Firefox 153; no Safari | Tooltips that don't close other popovers |
| Invoker commands (`command`/`commandfor`) | **Newly** (2025-12-12) | onclick glue for dialogs/popovers |
| Interest invokers (`interestfor`) | Limited: Chrome/Edge 142 only | Hover/focus-triggered tooltips |
| `<dialog closedby>` | Limited: Chrome 134, Firefox 141 | Light-dismiss for dialogs |
| `dialog.requestClose()` | **Newly** (2025-05-27) | Cancelable programmatic close |
| Customizable `<select>` | Limited: Chrome 135, Safari 27; no Firefox | Custom dropdown libraries |
| Navigation API | **Newly** (2026-01-13) | History API + popstate hacks |
| View Transitions (same-document) | **Newly** (2025-10-14) | FLIP animation libraries |
| Cross-document View Transitions | Limited: Chrome 126, Safari 18.2; no Firefox | SPA-only page transitions |
| URLPattern | **Newly** (2025-09-15) | path-to-regexp |
| Scheduler API (`postTask`, `yield`) | Limited: Chrome 129, Firefox 142; no Safari | `setTimeout(0)` chunking |
| Speculation Rules | Limited: Chromium only | `<link rel=prefetch>`, instant.page |
| Compression Streams | **Widely** (2025-11-09) | pako / fflate |
| WebGPU | Limited (Chrome, Safari 26, Firefox 141 Windows) | WebGL for compute/graphics |
| Observable (`el.when()`) | Limited: Chrome 135 | RxJS `fromEvent` |
| Document Picture-in-Picture | Limited: Chromium, Firefox 151 | Pop-out mini players/widgets |
| CloseWatcher | Limited: Chrome 126, Firefox 149 | Esc / Android back handling |
| Trusted Types | **Newly** (2026-02-24) | DOM XSS sink auditing |
| Sanitizer API (`setHTML`) | Limited: Firefox 148, Chrome 146; no Safari | DOMPurify (partly) |
| `ariaNotify()` | **Newly** (2026-09-14) | Hidden `aria-live` regions |
| Storage Buckets | Limited: Chrome 122 | One-bucket-per-origin storage |
| Web Locks | **Widely** (2024-09) | localStorage-based mutexes |
| Long Animation Frames | Limited: Chromium 123 | Long Tasks API for INP attribution |
| Soft Navigations | Chrome 151 (2026-07-28) | SPA route-change metrics |

## 1. Popover API (and `popover="hint"`)

Top-layer UI (menus, toasts, pickers) with **no JS**: it renders above everything (no z-index), gets **light dismiss** (click outside / Esc), and manages focus basics.

```html
<button popovertarget="filters">Filters</button>
<div id="filters" popover>
  <label><input type="checkbox"> Non-stop only</label>
</div>

<!-- manual: no light dismiss, multiple can be open (toasts) -->
<div id="toast" popover="manual">Booking saved</div>
<script>toast.showPopover(); setTimeout(() => toast.hidePopover(), 3000);</script>
```

```css
[popover] { transition: opacity .2s, display .2s allow-discrete, overlay .2s allow-discrete; opacity: 0; }
[popover]:popover-open { opacity: 1; }
@starting-style { [popover]:popover-open { opacity: 0; } }   /* animate in from display:none */
```

- JS: `showPopover()`, `hidePopover()`, `togglePopover()`, `beforetoggle`/`toggle` events, `:popover-open`.
- Pair with **CSS anchor positioning** to place it next to its button (see [Modern CSS 2025-2026](/Software%20Engineering/Frontend/04%20CSS/Modern%20CSS%202025-2026)).
- **Status**: Baseline **Newly** (date re-set to 2025-01-27).
- **`popover="hint"`**: a third type for tooltips. Opening a hint doesn't close open `auto` popovers (e.g. a tooltip inside an open menu). **Limited**: Chrome 151, Firefox 153; no Safari. Part of Interop 2026.

## 2. Invoker commands and interest invokers

**Invoker commands** let a button control a dialog/popover declaratively:

```html
<button commandfor="seat-dialog" command="show-modal">Choose seat</button>
<dialog id="seat-dialog">
  <p>Seat map…</p>
  <button commandfor="seat-dialog" command="close">Done</button>
</dialog>

<!-- custom commands start with -- and fire a "command" event on the target -->
<button commandfor="player" command="--rewind">⏪</button>
<script>
  player.addEventListener('command', (e) => { if (e.command === '--rewind') player.currentTime -= 10; });
</script>
```

Built-in commands: `show-modal`, `close`, `request-close` (dialogs), `show-popover`, `hide-popover`, `toggle-popover`. **Status**: Baseline **Newly (2025-12-12)**: Chrome 135, Firefox 144, Safari 26.2.

**Interest invokers** show UI when the user shows *interest* (hover, focus, long-press), ideal for tooltips and hover cards:

```html
<button interestfor="fare-tip">Fare rules</button>
<div id="fare-tip" popover="hint">Non-refundable, 1 free change.</div>
```
**Status**: **Limited**, Chrome/Edge 142 only. Progressive enhancement.

## 3. `<dialog>`: `closedby` and `requestClose()`

```html
<dialog id="d" closedby="any">…</dialog>
<!-- any: light dismiss + Esc + close(); closerequest: Esc/back only; none: only explicit close -->
```

```js
d.addEventListener('cancel', (e) => {
  if (form.dirty && !confirm('Discard changes?')) e.preventDefault(); // cancelable
});
d.requestClose('user-dismissed'); // like pressing Esc: fires "cancel" first, unlike close()
```

- `closedby`: **Limited** (Chrome 134, Firefox 141). Interop 2026 focus area (with `:open` and `hint`).
- `requestClose()`: Baseline **Newly (2025-05-27)**.
- `:open` pseudo-class (`details:open`, `dialog:open`): Baseline **Newly (2026-05-11)**.
- Always prefer `showModal()` for modals: it adds the backdrop, makes the rest of the page inert and traps focus. See [Accessibility Essentials](/Software%20Engineering/Frontend/12%20Accessibility/Accessibility%20Essentials).

## 4. Customizable `<select>`

```css
select, ::picker(select) { appearance: base-select; }
```
```html
<select>
  <button><selectedcontent></selectedcontent></button>
  <option value="EK"><img src="/logos/ek.svg" alt=""> Emirates</option>
  <option value="QR"><img src="/logos/qr.svg" alt=""> Qatar Airways</option>
</select>
```
Rich, fully stylable options while keeping native keyboard/a11y behaviour. **Limited**: Chrome 135, Safari 27 (2026-09-17); no Firefox. Older browsers fall back to a normal select.

## 5. Navigation API

A modern replacement for the History API for SPAs: **one `navigate` event** for every navigation (links, forms, back/forward, `navigation.navigate()`), which you can intercept.

```js
navigation.addEventListener('navigate', (e) => {
  if (!e.canIntercept || e.hashChange || e.downloadRequest) return;
  const url = new URL(e.destination.url);
  if (!url.pathname.startsWith('/flights')) return;

  e.intercept({
    async handler() {
      const data = await fetch(`/api${url.pathname}`, { signal: e.signal }).then(r => r.json()); // aborted if user navigates again
      render(data);
    },
    // focusReset / scroll behaviour handled by the browser
  });
});

await navigation.navigate('/flights/DXB-LHR', { state: { from: 'home' } }).finished;
navigation.currentEntry.getState();
navigation.entries();      // this app's history entries
navigation.back();
```

Why it's better than `pushState` + `popstate`: catches all navigation types in one place, built-in abort signal, `finished` promise, access to the entry list, proper accessibility (focus reset, loading indicators). **Status**: Baseline **Newly (2026-01-13)** (Firefox 147, Safari 26.2); Interop 2026 focus area. Framework routers wrap this for you ([React Router](/Software%20Engineering/Frontend/05%20React/React%20Router), [Next.js App Router Fundamentals](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20App%20Router%20Fundamentals)).

## 6. View Transitions (same-document and cross-document)

The browser snapshots the old state, you update the DOM, and it animates between snapshots with CSS.

```js
// Same-document (SPA)
function selectFlight(id) {
  if (!document.startViewTransition) return render(id);        // fallback
  const vt = document.startViewTransition(() => render(id));    // may be async
  vt.finished.then(() => console.log('done'));
}
```
```css
.flight-card.selected { view-transition-name: flight-hero; }  /* shared element morph */
.detail-header       { view-transition-name: flight-hero; }
::view-transition-old(root), ::view-transition-new(root) { animation-duration: 250ms; }
.card { view-transition-class: card; }                          /* style many at once */
::view-transition-group(*.card) { animation-timing-function: ease-in-out; }
@media (prefers-reduced-motion: reduce) { ::view-transition-group(*) { animation: none; } }
```

```css
/* Cross-document (MPA): opt in on BOTH pages, same origin */
@view-transition { navigation: auto; }
```

- **Same-document**: Baseline **Newly (2025-10-14)** with Firefox 144; `view-transition-class` also Newly.
- **Cross-document**: **Limited** (Chrome 126, Safari 18.2; no Firefox). Interop 2026 includes view transitions including cross-document.
- React 19.3 made **`<ViewTransition>` stable**; Next.js 16.2 added `<Link transitionTypes>`; see [React 19 to 19.3 & React Compiler](/Software%20Engineering/Frontend/05%20React/React%2019%20to%2019.3%20%26%20React%20Compiler).

## 7. URLPattern

```js
const route = new URLPattern({ pathname: '/flights/:from-:to/:date?' });
const m = route.exec('https://site.com/flights/DXB-LHR/2026-10-01');
m.pathname.groups; // { from: 'DXB', to: 'LHR', date: '2026-10-01' }
route.test(location.href);

// in a service worker: route requests
const api = new URLPattern({ pathname: '/api/*' });
self.addEventListener('fetch', (e) => { if (api.test(e.request.url)) e.respondWith(networkFirst(e)); });
```
Uses path-to-regexp-style syntax (`:param`, `*`, `?` optional, `(regex)`). **Status**: Baseline **Newly (2025-09-15)**. Pairs well with the Navigation API.

## 8. Scheduler API: `scheduler.postTask()` and `scheduler.yield()`

Break up long tasks so input and rendering can happen in between, improving **INP** ([Core Web Vitals](/Software%20Engineering/Frontend/10%20Performance/Core%20Web%20Vitals)).

```js
async function processResults(rows) {
  for (let i = 0; i < rows.length; i++) {
    enrich(rows[i]);
    if (i % 200 === 0) await yieldToMain();   // let the browser handle clicks/paint
  }
}

function yieldToMain() {
  if (globalThis.scheduler?.yield) return scheduler.yield(); // continuation keeps priority (runs before other queued tasks)
  return new Promise((r) => setTimeout(r, 0));                // fallback
}

scheduler.postTask(() => sendAnalytics(), { priority: 'background' }); // 'user-blocking' | 'user-visible' | 'background'
const ctl = new TaskController({ priority: 'user-visible' });
scheduler.postTask(work, { signal: ctl.signal }); ctl.abort();
```
**Status**: **Limited** (Chrome 129, Firefox 142; no Safari), so keep the `setTimeout` fallback. See [Event Loop - Microtasks & Macrotasks](/Software%20Engineering/Frontend/01%20JavaScript/Event%20Loop%20-%20Microtasks%20%26%20Macrotasks).

## 9. Speculation Rules (prefetch / prerender)

Declarative next-page **prefetch** or full **prerender** (the page renders in a hidden tab, so navigation is instant).

```html
<script type="speculationrules">
{
  "prerender": [{ "where": { "href_matches": "/*" }, "eagerness": "moderate" }],
  "prefetch":  [{ "where": { "selector_matches": ".result-card a" }, "eagerness": "conservative" }]
}
</script>
```
- `eagerness`: `immediate`, `eager`, `moderate` (hover ~200 ms), `conservative` (pointer down).
- Prerendered pages run JS: guard analytics with `document.prerendering` / `prerenderingchange`; exclude logout/"add to cart" URLs.
- **Status**: **Limited, Chromium only**. Other browsers ignore it, so it's safe progressive enhancement.

## 10. Compression Streams

```js
// gzip a large JSON payload before upload
const stream = new Blob([JSON.stringify(bigPayload)]).stream().pipeThrough(new CompressionStream('gzip'));
const body = await new Response(stream).blob();
await fetch('/api/logs', { method: 'POST', body, headers: { 'Content-Encoding': 'gzip' } });

// decompress
const text = await new Response(res.body.pipeThrough(new DecompressionStream('gzip'))).text();
```
Formats: `gzip`, `deflate`, `deflate-raw` (Firefox 147 added Brotli). **Status**: Baseline **Widely (2025-11-09)**.

## 11. WebGPU

Modern GPU API for graphics **and general compute** (ML inference, image processing, data viz), successor to WebGL.

```js
if (!navigator.gpu) throw new Error("WebGPU not supported: use a WebGL fallback");
const adapter = await navigator.gpu.requestAdapter();
const device = await adapter.requestDevice();
const module = device.createShaderModule({ code: `@compute @workgroup_size(64) fn main() {}` }); // WGSL
```
**Status**: **Limited** per web-features: Chrome shipped on Windows/Mac/ChromeOS in 2023 (data lists Chrome 144, Jan 2026, for the full feature), **Safari 26**, Firefox 141 on Windows (147 adds Apple Silicon). Usually used via libraries (Three.js, Babylon.js, ONNX Runtime Web, Transformers.js). **WebNN** (neural-network API) is only in an origin trial from Chrome 146, not shipped.

## 12. Observable

Push-based streams on the platform (like RxJS basics), with `EventTarget.when()`:

```js
button.when('click')
  .filter((e) => e.altKey)
  .map((e) => ({ x: e.clientX, y: e.clientY }))
  .takeUntil(document.when('keydown'))
  .subscribe({ next: console.log });
```
**Status**: **Limited**, Chrome 135 only. Keep using RxJS in Angular apps ([Angular Change Detection, Zones & Signals](/Software%20Engineering/Frontend/08%20Angular/Angular%20Change%20Detection%2C%20Zones%20%26%20Signals)).

## 13. Document Picture-in-Picture

An always-on-top window that can hold **any HTML**, not just video (a mini flight tracker, a call UI, a timer).

```js
const pip = await documentPictureInPicture.requestWindow({ width: 400, height: 300 });
[...document.styleSheets].forEach((s) => { try { pip.document.head.append(s.ownerNode.cloneNode(true)); } catch {} });
pip.document.body.append(document.querySelector('#flight-tracker'));   // move live DOM
pip.addEventListener('pagehide', () => main.append(document.querySelector('#flight-tracker')));
```
Must be triggered by a user gesture. **Status**: **Limited** (Chromium, Firefox 151).

## 14. CloseWatcher

One API for "close requests": **Esc** on desktop, the **Android back** gesture/button, etc. Used internally by `<dialog>` and popovers; use it for custom overlays.

```js
function openSheet(sheet) {
  sheet.hidden = false;
  const watcher = new CloseWatcher();
  watcher.onclose = () => { sheet.hidden = true; };
  watcher.oncancel = (e) => { if (hasUnsaved()) e.preventDefault(); };
  sheet.querySelector('.close').onclick = () => watcher.requestClose();
}
```
**Status**: **Limited** (Chrome 126, Firefox 149). Prefer `<dialog>`/popover when you can; they get this for free.

## 15. Sanitizer API (`setHTML`) and Trusted Types

```js
// Sanitizer: safe-by-default HTML insertion (strips scripts, event handlers, etc.)
commentEl.setHTML(untrustedHtml);
commentEl.setHTML(untrustedHtml, { sanitizer: { elements: ['b', 'i', 'a', 'p'] } });

// Trusted Types: make dangerous sinks (innerHTML, script.src, eval) reject plain strings
// CSP header: Content-Security-Policy: require-trusted-types-for 'script'; trusted-types default
trustedTypes.createPolicy('default', { createHTML: (s) => DOMPurify.sanitize(s) });
```
- **Trusted Types**: Baseline **Newly (2026-02-24)**, after Firefox 148. React 19.3 added Trusted Types support in React DOM.
- **Sanitizer API `setHTML()`**: **Limited** (Firefox 148, Chrome 146; no Safari), so keep DOMPurify as a fallback.
- `Document.parseHTMLUnsafe()`: Baseline (2025-09) (the name signals it does **not** sanitize).
- Full treatment: [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types).

## 16. `ariaNotify()`

Announce messages to screen readers without a hidden `aria-live` region.

```js
document.ariaNotify('Flight added to your trip');
button.ariaNotify('3 results', { priority: 'high' });
```
**Status**: Baseline **Newly (2026-09-14)**. Keep live regions as fallback for older browsers. See [Accessibility Essentials](/Software%20Engineering/Frontend/12%20Accessibility/Accessibility%20Essentials).

## 17. Other recent APIs worth a line

- **Storage Buckets** (Limited, Chrome 122) and **Web Locks** (Widely): [Browser Storage - Cookies, Web Storage & IndexedDB](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Browser%20Storage%20-%20Cookies%2C%20Web%20Storage%20%26%20IndexedDB).
- **Long Animation Frames** (Chromium 123) and **Soft Navigations** (Chrome 151): [Observer APIs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Observer%20APIs).
- **`<usermedia>` element**: a declarative camera/mic permission element, in Chrome 151.
- **Built-in AI APIs** (Chrome: Translator, Language Detector, Summarizer stable in 138; Prompt API for the web in 148) and **WebMCP** (origin trial from Chrome 149): [AI in Frontend - MCP, WebMCP & Built-in AI](/Software%20Engineering/Frontend/16%20AI%20in%20Frontend/AI%20in%20Frontend%20-%20MCP%2C%20WebMCP%20%26%20Built-in%20AI).
- New CSS (anchor positioning, `@scope`, container style queries, scroll-driven animations…): [Modern CSS 2025-2026](/Software%20Engineering/Frontend/04%20CSS/Modern%20CSS%202025-2026).

## Interop 2026

Interop 2026 (announced **2026-02-12**) is the cross-browser effort (Apple, Google, Microsoft, Mozilla, Igalia…) to make these work consistently. Its **20 focus areas**:

1. Anchor positioning
2. Container style queries
3. Dialogs and popovers (`closedby`, `:open`, `hint`)
4. Scroll-driven animations
5. View transitions (including cross-document)
6. `attr()`
7. `contrast-color()`
8. Custom highlights
9. Fetch uploads and ranges
10. IndexedDB `getAllRecords`
11. JSPI (JavaScript Promise Integration for WebAssembly)
12. Media pseudo-classes
13. Navigation API
14. Scoped custom element registries ([Web Components & Shadow DOM](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Web%20Components%20%26%20Shadow%20DOM))
15. Scroll snap
16. `shape()`
17. Web compat
18. WebRTC
19. WebTransport ([Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time))
20. `zoom`

## Classic Web APIs quick reference (from the owner's checklist)

| API | One-liner |
|---|---|
| Console API | `console.table`, `console.time/timeEnd`, `console.group`, `console.assert`, `console.trace` for debugging. |
| Geolocation API | `navigator.geolocation.getCurrentPosition(ok, err, { enableHighAccuracy })`; permission prompt, HTTPS only. |
| History API | `history.pushState/replaceState` + `popstate`; being superseded by the Navigation API (above). |
| URL API | `new URL(href)`, `url.searchParams.get('q')`, `URLSearchParams`, `URL.canParse()`. See [URL as State](/Software%20Engineering/Frontend/06%20State%20Management/URL%20as%20State). |
| Drag & Drop API | `draggable="true"`, `dragstart`/`dragover` (`preventDefault` to allow drop)/`drop`, `dataTransfer`. Poor on touch; many use pointer-event libraries. |
| Canvas API / WebGL | Immediate-mode 2D (`getContext('2d')`) and 3D (`'webgl2'`) drawing; `OffscreenCanvas` in workers; WebGPU is the successor. |
| Notifications API | `Notification.requestPermission()`, `new Notification(title)`; with a service worker + Push API for background notifications. |
| WebRTC | Peer-to-peer audio/video/data (`RTCPeerConnection`, `getUserMedia`); needs signaling + STUN/TURN. |
| Beacon API | `navigator.sendBeacon(url, data)`: reliable fire-and-forget POST on page hide (analytics). |
| Broadcast Channel | `new BroadcastChannel('auth')`: messages between same-origin tabs/workers (sync logout). |
| Channel Messaging | `new MessageChannel()`: two linked `MessagePort`s for private 1:1 messaging (iframes, workers). |
| Fullscreen API | `el.requestFullscreen()`, `document.exitFullscreen()`, `:fullscreen`. |
| Page Visibility | `document.visibilityState`, `visibilitychange`: pause polling/video, flush analytics. |
| Periodic Background Sync | Chromium-only, installed PWAs: `registration.periodicSync.register('refresh', { minInterval })`. See [Web Workers, Service Workers & PWAs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Web%20Workers%2C%20Service%20Workers%20%26%20PWAs). |
| Picture-in-Picture (video) | `video.requestPictureInPicture()`; for arbitrary HTML see Document PiP above. |
| Pointer events & Pointer Lock | Unified mouse/touch/pen (`pointerdown`, `setPointerCapture`); `el.requestPointerLock()` for games/3D. |
| Screen Wake Lock | `await navigator.wakeLock.request('screen')`: keep the screen on (boarding pass, recipe). |
| Sensor APIs | Generic Sensor API (`Accelerometer`, `Gyroscope`, Chromium) and `DeviceOrientationEvent`. |
| Vibration API | `navigator.vibrate([100, 50, 100])` (not supported on iOS). |
| Web Animations API | `el.animate(keyframes, { duration, easing })` returns an `Animation` (pause, reverse, `finished`). |
| Credential Management | `navigator.credentials.get/create` for passwords, federated login and **passkeys (WebAuthn)**. See [Authentication & Token Storage](/Software%20Engineering/Frontend/11%20Security/Authentication%20%26%20Token%20Storage). |
| WebVTT | Text tracks for video captions: `<track kind="captions" src="en.vtt">`. |
| Media Capabilities | `navigator.mediaCapabilities.decodingInfo(config)`: will this codec play smoothly and power-efficiently? |
| Long Tasks / Navigation Timing | `PerformanceObserver` entry types `longtask` and `navigation`. See [Observer APIs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Observer%20APIs). |
| Storage API | `navigator.storage.estimate()` / `persist()`. See [Browser Storage - Cookies, Web Storage & IndexedDB](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Browser%20Storage%20-%20Cookies%2C%20Web%20Storage%20%26%20IndexedDB). |
| IndexedDB, Web Storage, Service Worker, Web Workers, WebSocket, Fetch, XHR, Server-sent events, Intersection/Resize Observer, Web Components, Event phases, Attributes vs properties, DOM/BOM | Covered in the other notes of this folder. |

## Interview questions

<Accordion title="How would you build an accessible dropdown menu today with minimal JS?">
`<button popovertarget="menu">` + `<div popover id="menu">` for top-layer rendering and light dismiss, CSS anchor positioning to place it, proper roles/keyboard handling for menu items. Add `interestfor`/`hint` for tooltips as enhancement.
</Accordion>

<Accordion title="What does the Navigation API give you over pushState/popstate?">
A single `navigate` event for all navigation types, `intercept()` with async handlers and abort signals, `finished`/`committed` promises, access to history entries and state, and built-in focus/scroll handling. Baseline since Jan 2026.
</Accordion>

<Accordion title="How do View Transitions work?">
The browser captures a snapshot of the old state, runs your DOM update, captures the new state, and animates between them using `::view-transition-*` pseudo-elements. `view-transition-name` pairs elements across states for shared-element morphs. Same-document is Baseline (Oct 2025); cross-document is Chromium + Safari.
</Accordion>

<Accordion title="How do you keep a long JS task from hurting INP?">
Split the work and yield between chunks: `await scheduler.yield()` where available (it keeps the continuation's priority), with a `setTimeout(0)` fallback; move heavy work to a worker; avoid layout thrashing.
</Accordion>

<Accordion title="What are Speculation Rules and what are the risks?">
A JSON script declaring URLs to prefetch or prerender based on eagerness. Risks: wasted bandwidth, server load, analytics/side effects running on prerendered pages (check `document.prerendering`), and prerendering state-changing URLs. Chromium-only, safe to add as enhancement.
</Accordion>

<Accordion title="How does the platform help with XSS now?">
Trusted Types (Baseline Feb 2026) block string assignments to dangerous sinks unless they pass through a policy; `setHTML()` sanitizes by default (Firefox/Chrome); CSP limits script sources.
</Accordion>

<Accordion title="What is Interop and why should a front end engineer care?">
A yearly cross-browser commitment to fix/implement selected features consistently. It predicts which Limited features will become safe to use soon (e.g. Navigation API, scoped registries, anchor positioning in Interop 2026).
</Accordion>

## Related
- [What's New in Frontend 2025-2026](/Software%20Engineering/Frontend/What's%20New%20in%20Frontend%202025-2026)
- [Modern CSS 2025-2026](/Software%20Engineering/Frontend/04%20CSS/Modern%20CSS%202025-2026)
- [How Browsers Render - Critical Rendering Path](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/How%20Browsers%20Render%20-%20Critical%20Rendering%20Path)
- [Observer APIs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Observer%20APIs)
- [Web Components & Shadow DOM](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Web%20Components%20%26%20Shadow%20DOM)
- [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types)
- [Accessibility Essentials](/Software%20Engineering/Frontend/12%20Accessibility/Accessibility%20Essentials)
- [Core Web Vitals](/Software%20Engineering/Frontend/10%20Performance/Core%20Web%20Vitals)
- [React 19 to 19.3 & React Compiler](/Software%20Engineering/Frontend/05%20React/React%2019%20to%2019.3%20%26%20React%20Compiler)
- [AI in Frontend - MCP, WebMCP & Built-in AI](/Software%20Engineering/Frontend/16%20AI%20in%20Frontend/AI%20in%20Frontend%20-%20MCP%2C%20WebMCP%20%26%20Built-in%20AI)

## Sources
- Baseline data: [webstatus.dev](https://webstatus.dev/), [web-features explorer: newly available](https://web-platform-dx.github.io/web-features-explorer/newly-available/)
- [Interop 2026 (web.dev)](https://web.dev/blog/interop-2026)
- MDN: [Popover API](https://developer.mozilla.org/en-US/docs/Web/API/Popover_API), [Invoker commands](https://developer.mozilla.org/en-US/docs/Web/API/Invoker_Commands_API), [`<dialog>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/dialog), [Navigation API](https://developer.mozilla.org/en-US/docs/Web/API/Navigation_API), [View Transition API](https://developer.mozilla.org/en-US/docs/Web/API/View_Transition_API), [URLPattern](https://developer.mozilla.org/en-US/docs/Web/API/URL_Pattern_API), [Prioritized Task Scheduling](https://developer.mozilla.org/en-US/docs/Web/API/Prioritized_Task_Scheduling_API), [Speculation Rules](https://developer.mozilla.org/en-US/docs/Web/API/Speculation_Rules_API), [Compression Streams](https://developer.mozilla.org/en-US/docs/Web/API/Compression_Streams_API), [WebGPU](https://developer.mozilla.org/en-US/docs/Web/API/WebGPU_API), [Document PiP](https://developer.mozilla.org/en-US/docs/Web/API/Document_Picture-in-Picture_API), [CloseWatcher](https://developer.mozilla.org/en-US/docs/Web/API/CloseWatcher), [HTML Sanitizer API](https://developer.mozilla.org/en-US/docs/Web/API/HTML_Sanitizer_API), [Trusted Types](https://developer.mozilla.org/en-US/docs/Web/API/Trusted_Types_API)
- [MDN Firefox 147 release notes](https://developer.mozilla.org/en-US/docs/Mozilla/Firefox/Releases/147)
- [Mozilla gfx: WebGPU in Firefox 141](https://mozillagfx.wordpress.com/2025/07/15/shipping-webgpu-on-windows-in-firefox-141/)
- [WebKit: Safari 27 features](https://webkit.org/blog/18325/webkit-features-for-safari-27-0/)
- Chrome: [New in Chrome 151](https://developer.chrome.com/blog/new-in-chrome-151), [Built-in AI APIs](https://developer.chrome.com/docs/ai/built-in-apis), [Phoronix: Chrome 146 beta (WebNN OT)](https://www.phoronix.com/news/Chrome-146-Beta)
- React 19.3: [react.dev/blog/2026/09/09/react-19-3](https://react.dev/blog/2026/09/09/react-19-3)
