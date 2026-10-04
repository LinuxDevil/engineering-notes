---
title: "Core Web Vitals"
tags: [frontend, frontend/performance]
aliases: [CWV, Web Vitals, LCP, INP, CLS]
updated: 2026-09-27
---
# Core Web Vitals

<Note>
**TL;DR**

Core Web Vitals are Google's three user-centric metrics for **loading (LCP)**, **responsiveness (INP)** and **visual stability (CLS)**. They are judged on **real-user (field) data at the 75th percentile**, collected by the Chrome UX Report (CrUX), and they feed Google Search's page experience signals. Lab tools (Lighthouse, DevTools) help you *debug*; field data (RUM, CrUX) tells you what users actually *feel*.
</Note>

## The three metrics and their thresholds

| Metric | Measures | Good | Needs improvement | Poor |
|---|---|---|---|---|
| **LCP** – Largest Contentful Paint | When the largest image/text block in the viewport is rendered | ≤ 2.5 s | 2.5–4 s | > 4 s |
| **INP** – Interaction to Next Paint | Latency from a user interaction to the next frame painted (roughly the worst interaction of the visit) | ≤ 200 ms | 200–500 ms | > 500 ms |
| **CLS** – Cumulative Layout Shift | How much visible content unexpectedly moves | ≤ 0.1 | 0.1–0.25 | > 0.25 |

- A page "passes" when **the 75th percentile** of page loads is "good" for all three, evaluated separately for mobile and desktop.
- **INP replaced FID** (First Input Delay) as a Core Web Vital in **March 2024**. FID only measured the *input delay* of the *first* interaction; INP measures the *whole* latency (input delay + processing + presentation) of *all* interactions.
- The thresholds are unchanged in 2025–2026 (see 🆕 box below).

### Diagnostic (non-core) metrics

| Metric | What it tells you | Good |
|---|---|---|
| **TTFB** – Time to First Byte | Server + network latency (redirects, DNS, TCP/TLS, server think time) | ≤ 0.8 s |
| **FCP** – First Contentful Paint | First time *any* text/image paints | ≤ 1.8 s |
| TBT – Total Blocking Time (lab only) | Sum of main-thread "long task" time over 50 ms between FCP and TTI; the lab proxy for INP | ≤ 200 ms |

TTFB and FCP are not Core Web Vitals, but they sit *inside* LCP: a slow TTFB makes a good LCP impossible.

{/* diagram source: /diagrams/diagram-317.mmd */}
![diagram](/diagrams/diagram-317.svg)

## How each metric is measured

### LCP
- Candidates: `<img>`, `<image>` inside SVG, `<video>` poster/first frame, elements with a CSS `background-image` via `url()`, and block-level text nodes.
- The browser emits a new `largest-contentful-paint` entry each time a *larger* candidate paints; reporting stops at the first user input (tap, click, key press) because interaction changes what the user sees.
- Size is the *visible* area in the viewport (clipped, not the intrinsic size). Elements with opacity 0 or low-entropy placeholder images are ignored.

LCP breaks down into four **sub-parts** — this is how you debug it:

| Sub-part | Meaning | Typical share of a good LCP |
|---|---|---|
| TTFB | Until the HTML's first byte arrives | ~40% |
| Resource load delay | TTFB → browser *starts* fetching the LCP resource | < 10% |
| Resource load duration | Time to download the LCP resource | ~40% |
| Element render delay | Resource loaded → element actually painted | < 10% |

### INP
Every click, tap and key press (not scroll or hover) is an interaction. Each has three phases:

{/* diagram source: /diagrams/diagram-318.mmd */}
![diagram](/diagrams/diagram-318.svg)

- INP is the **longest** interaction on the page, except that with many interactions, one highest outlier is ignored per 50 interactions (so it approximates the 98th percentile for busy pages).
- It is reported when the page is hidden/unloaded, so it covers the *full* visit — that is why lab tools cannot measure it directly (they use TBT as a proxy).

### CLS
- Each unexpected shift gets a score = **impact fraction × distance fraction** (how much of the viewport moved × how far it moved).
- Shifts within **500 ms after a user input** are excluded (`hadRecentInput`), because the user caused them.
- Shifts are grouped into **session windows** (gaps < 1 s, window max 5 s). CLS = the **largest** window's sum, not the sum over the whole page life. This keeps long-lived SPAs from accumulating unfairly.
- CSS `transform` animations don't cause layout shifts; animating `top/left/width/height` does.

## Field vs lab data

| | Field (RUM) | Lab (synthetic) |
|---|---|---|
| Source | Real users, real devices, real networks | One scripted load on a controlled machine |
| Tools | CrUX, PageSpeed Insights (top section), `web-vitals` library → your analytics, RUM vendors (SpeedCurve, Datadog, Sentry…) | Lighthouse, DevTools Performance panel, WebPageTest, Lighthouse CI |
| Measures INP? | Yes | No (TBT as a proxy) |
| Good for | "Are users happy? Which pages/segments hurt?" | "Why is it slow? Did this PR regress?" — reproducible, runs in CI |
| Weakness | Noisy, delayed, needs traffic | Doesn't reflect real device/network mix; misses interactions, cache state, personalisation |

<Tip>
**Rule of thumb**

**Field data decides whether you have a problem; lab data helps you find and fix it.** A 100 Lighthouse score with failing CrUX data is still failing.
</Tip>

### CrUX (Chrome UX Report)
- Anonymised field data from **opted-in Chrome users** (desktop and Android), aggregated over a **rolling 28-day window**.
- Available per **origin** and (for popular pages) per **URL**; pages without enough traffic fall back to origin-level data.
- Access it via **PageSpeed Insights**, the **CrUX API / History API**, **CrUX Dashboard (Looker Studio)**, **BigQuery** (monthly dataset), and Search Console's Core Web Vitals report.
- Only Chrome: Safari/Firefox users never appear in CrUX, so run your own RUM for the full picture.

## Measuring in code

### The `web-vitals` library (recommended)
Handles all the edge cases (bfcache restores, prerendered pages, background tabs, CLS session windows, INP outliers) that a raw `PerformanceObserver` gets wrong.

<Info>
**Send Core Web Vitals to analytics**

```js
import { onLCP, onINP, onCLS, onFCP, onTTFB } from 'web-vitals';

function sendToAnalytics(metric) {
  const body = JSON.stringify({
    name: metric.name,      // 'LCP' | 'INP' | 'CLS' | ...
    value: metric.value,
    rating: metric.rating,  // 'good' | 'needs-improvement' | 'poor'
    id: metric.id,          // unique per page load, for dedupe
    navigationType: metric.navigationType,
    page: location.pathname,
  });
  // sendBeacon survives page unload; fall back to keepalive fetch
  (navigator.sendBeacon && navigator.sendBeacon('/analytics', body)) ||
    fetch('/analytics', { body, method: 'POST', keepalive: true });
}

onLCP(sendToAnalytics);
onINP(sendToAnalytics);
onCLS(sendToAnalytics);
onFCP(sendToAnalytics);
onTTFB(sendToAnalytics);
```
</Info>

The **attribution build** (`import { onINP } from 'web-vitals/attribution'`) adds a `metric.attribution` object that tells you *why*: for LCP the element and the four sub-parts, for CLS the largest shifted element, for INP the target element, interaction type, the three phases and the related Long Animation Frame entries.

<Info>
**Debug INP in production with attribution**

```js
import { onINP } from 'web-vitals/attribution';

onINP(({ value, attribution }) => {
  const { interactionTarget, interactionType,
          inputDelay, processingDuration, presentationDelay,
          longAnimationFrameEntries } = attribution;
  // Which scripts were running during the slow frame?
  const scripts = (longAnimationFrameEntries || [])
    .flatMap(loaf => loaf.scripts)
    .map(s => ({ src: s.sourceURL, fn: s.sourceFunctionName, ms: s.duration }));
  report({ value, interactionTarget, interactionType,
           inputDelay, processingDuration, presentationDelay, scripts });
});
```
</Info>

### Raw Performance APIs
Useful to understand what the library does, and for custom metrics.

```js
// LCP candidates
new PerformanceObserver((list) => {
  const last = list.getEntries().at(-1);
  console.log('LCP candidate', last.startTime, last.element);
}).observe({ type: 'largest-contentful-paint', buffered: true });

// Layout shifts
new PerformanceObserver((list) => {
  for (const e of list.getEntries()) {
    if (!e.hadRecentInput) console.log('shift', e.value, e.sources);
  }
}).observe({ type: 'layout-shift', buffered: true });

// Custom timing with the Performance object
performance.mark('search:start');
await runSearch();
performance.mark('search:end');
performance.measure('search', 'search:start', 'search:end'); // shows in DevTools
```

### Long Animation Frames (LoAF) for INP attribution
- The `long-animation-frame` entry replaces the old Long Tasks API for diagnosis: it reports **frames taking > 50 ms**, including rendering work, and lists the **scripts** that ran (URL, function name, invoker such as `click` handler or `setTimeout`, duration, forced layout time) plus `blockingDuration`.
- It answers "which third-party or first-party script blocked this interaction?" — the single most useful INP debugging signal in the field.
- Status: **Chromium only** (Chrome 123+), Baseline Limited.

```js
new PerformanceObserver((list) => {
  for (const loaf of list.getEntries()) {
    if (loaf.blockingDuration > 100) {
      console.table(loaf.scripts.map(s => ({ src: s.sourceURL, invoker: s.invoker, ms: s.duration })));
    }
  }
}).observe({ type: 'long-animation-frame', buffered: true });
```

## Debugging and fixing each metric

### Fixing LCP
1. **Find the LCP element** (DevTools Performance panel "LCP" marker, Lighthouse "Largest Contentful Paint element", or web-vitals attribution).
2. **Look at the sub-parts** and attack the biggest one:

| Biggest sub-part | Fixes |
|---|---|
| TTFB | CDN / edge caching of HTML, fewer redirects, faster backend, streaming SSR, `103 Early Hints`, avoid heavy middleware |
| Resource load delay | Make the LCP image discoverable in the **initial HTML** (not injected by JS, not a CSS background), `fetchpriority="high"`, `<link rel="preload">` if it must be CSS/JS-discovered, **never `loading="lazy"` on the LCP image**, `preconnect` to the image CDN |
| Resource load duration | Smaller image: AVIF/WebP, correct `srcset`/`sizes`, compression, CDN; fewer competing downloads |
| Element render delay | Remove render-blocking CSS/JS, inline critical CSS, avoid client-side rendering the hero (SSR/SSG it), don't hide content behind A/B-test scripts or hydration, `font-display` for text LCP |

<Info>
**LCP-friendly hero image**

```html
<link rel="preconnect" href="https://img.example-cdn.com">
<img src="https://img.example-cdn.com/hero-1200.avif"
     srcset="https://img.example-cdn.com/hero-600.avif 600w,
             https://img.example-cdn.com/hero-1200.avif 1200w"
     sizes="100vw" width="1200" height="600"
     fetchpriority="high" alt="Dubai skyline at sunset">
```
</Info>

### Fixing INP
- **Input delay high** → the main thread was busy: break up long tasks, defer/remove third-party scripts, avoid heavy work during hydration, move work to a Web Worker.
- **Processing high** → your handler is slow: do the minimum needed for visual feedback first, then **yield** (`await scheduler.yield()` with a `setTimeout` fallback) before the rest; debounce; avoid synchronous `localStorage`/layout reads; in React use `startTransition`/`useDeferredValue` and fix unnecessary re-renders (see [React Performance](/Software%20Engineering/Frontend/05%20React/React%20Performance)).
- **Presentation delay high** → too much rendering: huge DOM (virtualize — [Virtualization & Large Lists](/Software%20Engineering/Frontend/10%20Performance/Virtualization%20%26%20Large%20Lists)), layout thrashing, expensive CSS, `content-visibility: auto` for off-screen sections.

<Info>
**Give feedback first, then yield**

```js
button.addEventListener('click', async () => {
  button.classList.add('loading');         // cheap visual update
  await yieldToMain();                     // let the browser paint it
  const results = filterHugeList(query);   // heavy work after the paint
  render(results);
});

function yieldToMain() {
  if (globalThis.scheduler?.yield) return scheduler.yield();
  return new Promise((r) => setTimeout(r, 0));
}
```
</Info>

### Fixing CLS
- Always set **`width`/`height`** (or `aspect-ratio`) on images, videos, iframes and ad slots.
- **Reserve space** for late content (banners, cookie bars, ads, embeds) with `min-height`; insert new content *below* the viewport or in response to a user action.
- Web fonts: `font-display: optional` or metric-matched fallbacks (`size-adjust`, `ascent-override`) so the text doesn't reflow when the font swaps.
- Animate with `transform`/`opacity` instead of layout properties.
- Keep pages **bfcache-eligible** — back/forward restores have zero CLS and near-zero LCP.
- Skeletons must match the final layout size.

### Fixing TTFB / FCP
- Cache HTML at the edge (SSG/ISR or `Cache-Control` + `stale-while-revalidate`), minimise redirects, use HTTP/2 or HTTP/3, stream the response so the `<head>` flushes early.
- Remove render-blocking resources in `<head>`; `defer` scripts; inline critical CSS. See [How Browsers Render - Critical Rendering Path](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/How%20Browsers%20Render%20-%20Critical%20Rendering%20Path).

<Warning>
**Common pitfalls**

- Optimising the Lighthouse score instead of field data.
- Lazy-loading the hero image (hurts LCP badly).
- Measuring only on fast office laptops — test with CPU throttling (4× or more) and a mid-range Android profile.
- Forgetting that SPA route changes are not new page loads for CrUX (see soft navigations below).
- Using `unload` handlers, which block bfcache.
</Warning>

<Info>
**🆕 2025–2026**

- **Thresholds unchanged:** LCP ≤ 2.5 s, INP ≤ 200 ms, CLS ≤ 0.1.
- **LCP and Event Timing are Baseline Newly available (2025-12-12)** now that Safari 26.2 supports them (Firefox had LCP since 122). Your *own* RUM can therefore collect LCP and interaction latency across all major engines — but **CrUX is still Chrome-only data**.
- **Soft Navigations API shipped in Chrome 151 (2026-07-28)** after a final origin trial in Chrome 147–149. It adds a `soft-navigation` PerformanceEntry and a new `interaction-contentful-paint` entry, so SPA route changes (History/Navigation API URL change after a user interaction + DOM paint) can be measured like page loads.
- **web-vitals v6 (2026-07-21)** supports soft navigations.
- **Still undecided:** how CrUX and Core Web Vitals will use SPA soft-navigation data. Google said it would decide after launch; there's no confirmation that CrUX counts it yet.
- **LoAF** remains Chromium only — use it for INP attribution.
- Framework traces in DevTools: **React 19.2 Performance Tracks** and Angular's Chrome DevTools profiling integration (v20).

```js
new PerformanceObserver((l) => l.getEntries().forEach(console.log))
  .observe({ type: 'soft-navigation', buffered: true });
```
</Info>

## Interview questions

<Accordion title="What are the Core Web Vitals and their &quot;good&quot; thresholds?">
LCP ≤ 2.5 s (loading), INP ≤ 200 ms (responsiveness), CLS ≤ 0.1 (visual stability), measured at the 75th percentile of real page loads, separately for mobile and desktop.
</Accordion>

<Accordion title="Why did INP replace FID?">
FID only measured the input delay of the first interaction, ignoring handler time and rendering, and most sites passed it. INP measures the full latency (input delay + processing + presentation delay) of all interactions across the visit, so it reflects real sluggishness.
</Accordion>

<Accordion title="Lighthouse says 98 but Search Console says my LCP is poor. Why?">
Lighthouse is one synthetic load on a fast machine with a cold cache and simulated throttling. Search Console uses CrUX field data from real users on real devices and networks over 28 days. Differences come from slow devices, geography/CDN misses, personalisation, ads, logged-in states, and different LCP elements. Trust field data; use lab to reproduce.
</Accordion>

<Accordion title="How would you debug a poor LCP?">
Identify the LCP element, then split LCP into TTFB, resource load delay, load duration and render delay. Fix the biggest: cache HTML at the edge for TTFB; make the image discoverable in HTML with `fetchpriority="high"` / preload for load delay; compress and resize (AVIF, srcset) for duration; remove render-blocking CSS/JS and avoid client-only rendering for render delay.
</Accordion>

<Accordion title="How do you find what causes a poor INP in production?">
Collect INP with the `web-vitals/attribution` build: it gives the target element, interaction type, the three phases and Long Animation Frame entries listing the scripts that ran. Then fix by phase: break up long tasks and yield (input delay/processing), reduce render work and DOM size (presentation delay).
</Accordion>

<Accordion title="How is CLS calculated, and why doesn't it just keep growing in a long-lived SPA?">
Each shift scores impact fraction × distance fraction; shifts within 500 ms of user input are excluded. Shifts are grouped into session windows (max 5 s, gaps under 1 s) and CLS is the largest window, so a long session doesn't accumulate forever.
</Accordion>

<Accordion title="Are TTFB and FCP Core Web Vitals?">
No. They're diagnostic metrics. TTFB is part of LCP (the first sub-part) and FCP shows when anything first appears; both help explain a bad LCP.
</Accordion>

<Accordion title="How are SPA route changes measured?">
Historically they weren't — CrUX only counts hard navigations. Chrome 151 shipped the Soft Navigations API (`soft-navigation` and `interaction-contentful-paint` entries) and web-vitals v6 supports it, so RUM can measure them. Whether CrUX will count them is still undecided.
</Accordion>

## Related
- [Performance Optimization Playbook](/Software%20Engineering/Frontend/10%20Performance/Performance%20Optimization%20Playbook)
- [Virtualization & Large Lists](/Software%20Engineering/Frontend/10%20Performance/Virtualization%20%26%20Large%20Lists)
- [How Browsers Render - Critical Rendering Path](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/How%20Browsers%20Render%20-%20Critical%20Rendering%20Path)
- [React Performance](/Software%20Engineering/Frontend/05%20React/React%20Performance)
- [Rendering Patterns - CSR, SSR, SSG, ISR, Streaming & Islands](/Software%20Engineering/Frontend/09%20Architecture/Rendering%20Patterns%20-%20CSR%2C%20SSR%2C%20SSG%2C%20ISR%2C%20Streaming%20%26%20Islands)
- [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time)
- [Observer APIs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Observer%20APIs)

## Sources
- [web.dev – Web Vitals](https://web.dev/articles/vitals)
- [web.dev – LCP](https://web.dev/articles/lcp), [Optimize LCP](https://web.dev/articles/optimize-lcp)
- [web.dev – INP](https://web.dev/articles/inp), [Optimize INP](https://web.dev/articles/optimize-inp)
- [web.dev – CLS](https://web.dev/articles/cls)
- [Chrome UX Report docs](https://developer.chrome.com/docs/crux)
- [GoogleChrome/web-vitals](https://github.com/GoogleChrome/web-vitals)
- [Long Animation Frames API](https://developer.chrome.com/docs/web-platform/long-animation-frames)
- [MDN – Performance API](https://developer.mozilla.org/en-US/docs/Web/API/Performance_API)
- [New in Chrome 151](https://developer.chrome.com/blog/new-in-chrome-151)
- [Final soft navigations origin trial](https://developer.chrome.com/blog/final-soft-navigations-origin-trial)
- [CSS Wizardry – Native SPA metrics have arrived](https://csswizardry.com/2026/08/web-perf-wednesday-003-native-spa-metrics-have-arrived/)
