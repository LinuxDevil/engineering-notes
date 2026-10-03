---
tags: [frontend, frontend/browser, frontend/performance]
aliases: [Observers, IntersectionObserver, MutationObserver, ResizeObserver, PerformanceObserver, Observer API]
updated: 2026-09-27
---
# Observer APIs

> [!tldr] TL;DR
> The browser's observer APIs let you **react to changes without polling or scroll/resize listeners**: `IntersectionObserver` (is an element visible?), `MutationObserver` (did the DOM change?), `ResizeObserver` (did an element's size change?) and `PerformanceObserver` (new performance entries: LCP, long tasks, resources). They all follow the same pattern: create with a callback, `observe()` targets, `disconnect()` to clean up. They are the building blocks of lazy loading, infinite scroll, sticky headers, auto-resizing components and RUM.

## The common pattern

```js
const observer = new XxxObserver((entries, observer) => {
  for (const entry of entries) { /* react */ }
}, options);

observer.observe(target, observeOptions);
observer.unobserve(target);   // Intersection/Resize: stop one target
observer.disconnect();        // stop everything (always do this on unmount)
```

- Callbacks are **batched** and run asynchronously, so you get a list of entries, not one call per change.
- They're the Observer design pattern built into the platform ([[Frontend Design Patterns]]).

| Observer | Fires when | Callback timing | Typical uses |
|---|---|---|---|
| `IntersectionObserver` | Target's visibility vs a root (viewport or scroll container) crosses a threshold | Asynchronously during the rendering steps (low priority, doesn't block scrolling) | Lazy loading, infinite scroll, analytics impressions, sticky states, pausing videos |
| `MutationObserver` | DOM nodes/attributes/text change | **Microtask** after the mutation | Reacting to third-party DOM changes, editors, testing utilities, polyfills |
| `ResizeObserver` | Element's content/border box size changes | In the rendering steps, after layout, before paint | Responsive components, auto-resizing textareas/charts, virtualization |
| `PerformanceObserver` | New performance entries are recorded | Asynchronously (idle-ish) | RUM: LCP, CLS, INP, long tasks, resource timing, navigation timing |

## 1. IntersectionObserver

```js
const io = new IntersectionObserver(callback, {
  root: null,              // null = viewport; or a scrollable ancestor element
  rootMargin: '200px 0px', // grow/shrink the root box (start loading 200px early)
  threshold: [0, 0.5, 1],  // ratios at which to fire
});
```

Entry fields: `isIntersecting`, `intersectionRatio`, `boundingClientRect`, `intersectionRect`, `rootBounds`, `target`, `time`.

Why not `scroll` listeners + `getBoundingClientRect()`? Scroll handlers fire very often on the main thread and each rect read can force layout. IntersectionObserver is computed by the browser during rendering with no forced layouts.

> [!example] Lazy-loading images (when `loading="lazy"` isn't enough)
> ```js
> // <img data-src="/deals/paris.avif" alt="Paris" class="lazy">
> const io = new IntersectionObserver((entries, obs) => {
>   entries.forEach((entry) => {
>     if (!entry.isIntersecting) return;
>     const img = entry.target;
>     img.src = img.dataset.src;
>     obs.unobserve(img);               // load once
>   });
> }, { rootMargin: '300px' });
>
> document.querySelectorAll('img.lazy').forEach((img) => io.observe(img));
> ```
> For plain images/iframes prefer native `loading="lazy"`. Use IO for backgrounds, components, videos or analytics.

> [!example] Infinite scroll with a sentinel element (React)
> ```jsx
> function Results({ items, loadMore, hasMore }) {
>   const sentinelRef = useRef(null);
>
>   useEffect(() => {
>     const el = sentinelRef.current;
>     if (!el || !hasMore) return;
>     const io = new IntersectionObserver(([entry]) => {
>       if (entry.isIntersecting) loadMore();   // fetch next page
>     }, { rootMargin: '400px' });              // prefetch before the user hits the bottom
>     io.observe(el);
>     return () => io.disconnect();
>   }, [loadMore, hasMore]);
>
>   return (
>     <>
>       <ul>{items.map((f) => <FlightRow key={f.id} flight={f} />)}</ul>
>       {hasMore && <div ref={sentinelRef} aria-hidden="true" style={{ height: 1 }} />}
>     </>
>   );
> }
> ```
> Guard `loadMore` against duplicate calls while a request is in flight. Long lists should combine this with virtualization ([[Virtualization & Large Lists]], [[Case Study - News Feed]]).

> [!example] Sticky header shadow and "seen" analytics
> ```js
> // Sentinel at the top of the page: when it leaves the viewport, the header is "stuck"
> new IntersectionObserver(([e]) => header.classList.toggle('stuck', !e.isIntersecting))
>   .observe(document.querySelector('#top-sentinel'));
>
> // Count an ad/deal impression when 50% visible
> const seen = new IntersectionObserver((entries) => entries.forEach((e) => {
>   if (e.intersectionRatio >= 0.5) { track('impression', e.target.dataset.id); seen.unobserve(e.target); }
> }), { threshold: 0.5 });
> ```

Notes:
- The callback fires **once immediately** on `observe()` with the initial state.
- `rootMargin` only works with the viewport when the document isn't in a cross-origin iframe.
- IntersectionObserver v2 (`trackVisibility`) can detect occlusion (Chromium only).
- Pure-CSS alternatives exist for some cases: scroll-driven animations (`animation-timeline: view()`) and scroll-state container queries for "stuck" styles (see [[Modern CSS 2025-2026]]).

## 2. MutationObserver

Watches DOM changes. It replaced the deprecated, slow Mutation Events (`DOMNodeInserted`…), which fired synchronously on every change.

```js
const mo = new MutationObserver((mutations) => {
  for (const m of mutations) {
    if (m.type === 'childList') console.log('added', m.addedNodes, 'removed', m.removedNodes);
    if (m.type === 'attributes') console.log(m.attributeName, 'was', m.oldValue);
    if (m.type === 'characterData') console.log('text changed');
  }
});

mo.observe(target, {
  childList: true,          // direct children added/removed
  subtree: true,            // whole descendant tree
  attributes: true,
  attributeFilter: ['class', 'aria-expanded'],
  attributeOldValue: true,
  characterData: true,
});

const pending = mo.takeRecords(); // flush queued records synchronously (e.g. before disconnect)
mo.disconnect();
```

> [!example] Wait for a third-party widget to render, then enhance it
> ```js
> function waitForElement(selector, root = document.body) {
>   return new Promise((resolve) => {
>     const found = root.querySelector(selector);
>     if (found) return resolve(found);
>     const mo = new MutationObserver(() => {
>       const el = root.querySelector(selector);
>       if (el) { mo.disconnect(); resolve(el); }
>     });
>     mo.observe(root, { childList: true, subtree: true });
>   });
> }
> const chat = await waitForElement('#vendor-chat-button');
> chat.setAttribute('aria-label', 'Open support chat');
> ```

Use cases: reacting to DOM you don't control (third-party scripts, CMS content), rich-text editors, auto-linking/highlighting inserted content, Testing Library's `waitFor`/`findBy*` (uses it under the hood), detecting injected elements (security monitoring).

> [!warning] Pitfalls
> - Changing the DOM inside the callback on the same subtree can cause infinite loops. Disconnect first or check for your own changes.
> - `subtree: true` on `document.body` in a busy app produces a flood of records; scope it tightly.
> - In a framework app you rarely need it for your own components: the framework already knows when it changes the DOM.

## 3. ResizeObserver

Tells you when an **element's** size changes, for any reason (content, CSS, parent layout, font load), not just the window. Window `resize` events don't fire for those.

```js
const ro = new ResizeObserver((entries) => {
  for (const entry of entries) {
    const { inlineSize, blockSize } = entry.contentBoxSize[0]; // writing-mode aware
    // also: entry.borderBoxSize, entry.devicePixelContentBoxSize, entry.contentRect (legacy)
  }
});
ro.observe(el, { box: 'content-box' }); // or 'border-box' | 'device-pixel-content-box'
```

> [!example] Auto-resize: a chart that fits its container
> ```jsx
> function useElementSize() {
>   const ref = useRef(null);
>   const [size, setSize] = useState({ width: 0, height: 0 });
>   useLayoutEffect(() => {
>     const ro = new ResizeObserver(([entry]) => {
>       const { inlineSize: width, blockSize: height } = entry.contentBoxSize[0];
>       setSize({ width, height });
>     });
>     ro.observe(ref.current);
>     return () => ro.disconnect();
>   }, []);
>   return [ref, size];
> }
>
> function PriceChart({ data }) {
>   const [ref, { width, height }] = useElementSize();
>   return <div ref={ref} style={{ height: 240 }}>{width > 0 && <Chart data={data} width={width} height={height} />}</div>;
> }
> ```

> [!example] Auto-growing textarea (and the CSS alternative)
> ```js
> const ta = document.querySelector('textarea');
> ta.addEventListener('input', () => { ta.style.height = 'auto'; ta.style.height = ta.scrollHeight + 'px'; });
> ```
> ```css
> /* Modern: no JS. field-sizing is Baseline Newly available since 2026-06-16 */
> textarea { field-sizing: content; min-height: 3lh; max-height: 12lh; }
> ```

Notes:
- If you change sizes inside the callback and cause another resize in the same frame, the browser defers it and logs **"ResizeObserver loop completed with undelivered notifications"**. Usually harmless, but avoid feedback loops (e.g. set size from observed size + padding).
- For **styling** based on container size, use CSS **container queries** instead of JS ([[Layout - Flexbox, Grid & Container Queries]]). Use ResizeObserver when JS needs the number (canvas, charts, virtualization, truncation logic).

## 4. PerformanceObserver

Subscribes to entries from the Performance Timeline. This is how RUM libraries (and Google's `web-vitals`) measure real users.

```js
const po = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) console.log(entry.entryType, entry.name, entry.startTime, entry.duration);
});
po.observe({ type: 'largest-contentful-paint', buffered: true }); // buffered: include entries before observing
console.log(PerformanceObserver.supportedEntryTypes);
```

| Entry type | What it measures |
|---|---|
| `navigation` | Navigation Timing: DNS, TCP, TLS, TTFB, `domContentLoaded`, `load` |
| `resource` | Resource Timing: each script/image/fetch's timings and sizes |
| `paint` | First Paint, First Contentful Paint |
| `largest-contentful-paint` | LCP candidates |
| `layout-shift` | Individual shifts, used to compute CLS |
| `event` / `first-input` | Event Timing, used to compute INP |
| `longtask` | Main-thread tasks > 50 ms (Long Tasks API) |
| `long-animation-frame` | LoAF: slow frames with script attribution (Chromium only) |
| `mark` / `measure` | Your own `performance.mark()` / `performance.measure()` |
| `soft-navigation` | SPA route changes (Chrome 151+) |

> [!example] Measure a custom interaction + report LCP
> ```js
> performance.mark('search:start');
> await runSearch();
> performance.mark('search:end');
> performance.measure('search', 'search:start', 'search:end');
>
> new PerformanceObserver((list) => {
>   const entries = list.getEntries();
>   const lcp = entries.at(-1);                 // the last candidate is the current LCP
>   navigator.sendBeacon('/rum', JSON.stringify({ lcp: lcp.startTime, el: lcp.element?.tagName }));
> }).observe({ type: 'largest-contentful-paint', buffered: true });
> ```
> In production, use the `web-vitals` library (`onLCP`, `onINP`, `onCLS`) which handles the edge cases (bfcache, visibility, final values).

> [!info]+ 🆕 2025–2026
> - **LCP and Event Timing entries are Baseline Newly available (2025-12-12)** now that Safari 26.2 supports them, so cross-browser LCP/INP field data is possible.
> - **Soft Navigations API shipped in Chrome 151 (2026-07-28)**: new `soft-navigation` and `interaction-contentful-paint` entries for SPAs; **web-vitals v6** (2026-07-21) supports it. How CrUX will use SPA soft-nav data is still undecided.
>   ```js
>   new PerformanceObserver((l) => l.getEntries().forEach(console.log))
>     .observe({ type: 'soft-navigation', buffered: true });
>   ```
> - **Long Animation Frames (LoAF)** remains Chromium only (123+); use it for INP attribution.
> - **Observable** (`el.when('click')`) is a separate Chromium-only proposal (Chrome 135), see [[Modern Web Platform APIs 2025-2026]].
> - React 19.3 Fragment refs have `observeUsing(observer)` to attach an Intersection/Resize observer to all of a fragment's children.

## Interview questions

> [!question]- How would you implement infinite scroll?
> Render a sentinel element after the list, observe it with IntersectionObserver (with a generous `rootMargin`), fetch the next page when it intersects, guard against concurrent fetches, stop when there's no more data, and virtualize the list to keep the DOM small. Provide a "Load more" button fallback for accessibility.

> [!question]- Why is IntersectionObserver better than a scroll listener?
> Scroll listeners fire many times per frame on the main thread and usually call `getBoundingClientRect()` (forced layout). IO is computed by the browser during rendering, batched, and async.

> [!question]- When would you use MutationObserver in a React app?
> Rarely for your own components. Useful for third-party DOM (widgets, CMS HTML), content-editable editors, or tests. Its callbacks run as microtasks, batched.

> [!question]- ResizeObserver vs window resize vs container queries?
> Window `resize` only fires on viewport changes. ResizeObserver fires for any element size change. Container queries handle *styling* by container size in pure CSS; use ResizeObserver when JS needs the dimensions.

> [!question]- How do you measure Core Web Vitals in the field?
> PerformanceObserver on `largest-contentful-paint`, `layout-shift`, `event` entries (or the `web-vitals` library), send with `navigator.sendBeacon` on `visibilitychange` to hidden.

> [!question]- What does `buffered: true` do?
> Delivers entries recorded before the observer was created (e.g. LCP that happened before your analytics script loaded).

## Related
- [[DOM, Events & Event Delegation]]
- [[Virtualization & Large Lists]]
- [[Core Web Vitals]]
- [[Performance Optimization Playbook]]
- [[Case Study - News Feed]]
- [[Layout - Flexbox, Grid & Container Queries]]
- [[Modern Web Platform APIs 2025-2026]]

## Sources
- MDN: [Intersection Observer API](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API), [MutationObserver](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver), [Resize Observer API](https://developer.mozilla.org/en-US/docs/Web/API/Resize_Observer_API), [PerformanceObserver](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceObserver), [Navigation timing](https://developer.mozilla.org/en-US/docs/Web/API/Performance_API/Navigation_timing)
- web.dev: [Web Vitals](https://web.dev/articles/vitals), [Custom metrics](https://web.dev/articles/custom-metrics)
- [web-vitals library](https://github.com/GoogleChrome/web-vitals)
- Chrome: [New in Chrome 151](https://developer.chrome.com/blog/new-in-chrome-151), [Final soft navigations origin trial](https://developer.chrome.com/blog/final-soft-navigations-origin-trial)
- [CSS Wizardry: native SPA metrics have arrived](https://csswizardry.com/2026/08/web-perf-wednesday-003-native-spa-metrics-have-arrived/)
- React 19.3: [react.dev/blog/2026/09/09/react-19-3](https://react.dev/blog/2026/09/09/react-19-3)
