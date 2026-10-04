---
title: "Modern CSS 2025-2026"
tags: [frontend, frontend/css, frontend/new]
aliases: [Modern CSS, New CSS, CSS 2026]
updated: 2026-09-27
---
# Modern CSS 2025-2026

<Note>
**TL;DR**

CSS has taken over work that used to need JavaScript or preprocessors: nesting, parent selection (`:has`), scoping (`@scope`), cascade control (`@layer`), enter animations (`@starting-style`), tooltips and menus placed next to their trigger (anchor positioning), page transitions, scroll-linked animation, and even conditionals and functions (`if()`, `@function`). The skill now is knowing **what is Baseline** (safe everywhere) versus **what is Limited** (progressive enhancement only).

All statuses below come from webstatus.dev / web-features as of **2026-09-27**. "Newly" means it works in all core browsers since the date shown. "Widely" means it has been in all core browsers for at least 30 months. "Limited" means at least one core browser is missing it.
</Note>

---

## Status table

| Feature | Status (2026-09-27) | Use in production? |
|---|---|---|
| CSS nesting | **Widely**, 2026-06-11 | ✅ yes |
| `:has()` | **Widely**, 2026-06-19 | ✅ yes |
| `@layer` (cascade layers) | Widely (all browsers since 2022) | ✅ yes |
| Container size queries | **Widely**, 2025-08-14 | ✅ yes |
| `@starting-style` | Newly, 2024-08-06 | ✅ yes |
| `light-dark()` | Newly, 2024-05-13 (image values Newly 2026-09-14) | ✅ yes |
| Popover API | Newly (date re-set to 2025-01-27) | ✅ yes |
| `dialog.requestClose()` | Newly, 2025-05-27 | ✅ yes |
| Same-document View Transitions (+ `view-transition-class`) | **Newly, 2025-10-14** (Firefox 144) | ✅ yes, as enhancement |
| Invoker commands (`command` / `commandfor`) | **Newly, 2025-12-12** | ✅ yes |
| `@scope` | **Newly, 2026-03-24** (Safari 26.4) | ✅ yes (check your support matrix) |
| `contrast-color()` | **Newly, 2026-04-10** | ✅ yes, with a fallback for older browsers |
| `:open` | Newly, 2026-05-11 | ✅ yes |
| Name-only container queries | Newly, 2026-05-07 | ✅ yes |
| Container style queries | **Newly, 2026-05-19** | ✅ yes |
| `field-sizing` | **Newly, 2026-06-16** | ✅ yes |
| `sibling-index()` / `sibling-count()` | **Newly, 2026-08-18** | ⚠️ very new; enhancement only |
| Anchor positioning | Shipped in Chrome 125, Safari 26, **Firefox 147 (Jan 2026)**. webstatus still says Limited (the redefined feature counts only Safari 27 as complete) | ⚠️ yes, with `@supports` and a fallback |
| Cross-document View Transitions | Limited: Chrome 126, Safari 18.2; no Firefox | ⚠️ enhancement (it simply doesn't animate elsewhere) |
| Scroll-driven animations | Limited: Chrome 115, Safari 26; no Firefox | ⚠️ enhancement |
| Customizable `<select>` (`appearance: base-select`) | Limited: Chrome 135, **Safari 27 (2026-09-17)**; no Firefox | ⚠️ enhancement (falls back to the native select) |
| `text-box` (trim) | Limited: Chrome 133, Safari 18.2 | ⚠️ enhancement |
| Typed `attr()` | Limited: Chrome 133, Firefox 155 | ⚠️ enhancement |
| Scroll-state container queries | Limited: Chrome 133 only | ❌ experiment |
| `interpolate-size` / `calc-size()` | Limited: Chrome 129 only | ⚠️ harmless enhancement |
| Scroll markers / buttons (CSS carousels) | Limited: Chrome 135 | ❌ experiment |
| `if()` | Limited: Chrome 137 only | ❌ experiment |
| `reading-flow` | Limited: Chrome 137 only | ❌ experiment |
| `@function` (custom functions) | Limited: Chrome 139 only | ❌ experiment |
| `corner-shape` | Limited: Chrome 139 only | ⚠️ harmless enhancement (falls back to `border-radius`) |

Other items that became Newly available recently: `shape()` (2026-02-24), custom highlights (2026-03-24), `progress()` (2026-09-01), `alpha()` and `overflow-anchor` (2026-09-14), `scrollbar-color` (2025-12-12), `::details-content` and `content-visibility` (Sept 2025).

**Interop 2026** (announced 2026-02-12) focuses on anchor positioning, container style queries, dialogs and popovers (`closedby`, `:open`, `popover="hint"`), scroll-driven animations, view transitions (including cross-document), `attr()`, `contrast-color()`, custom highlights, scroll snap, `shape()` and `zoom`. Expect the Limited items in that list to move toward Baseline.

---

## Selectors and architecture

### Nesting (Widely, 2026-06-11)
```css
.card {
  padding: 1rem;
  &:hover { box-shadow: var(--shadow-2); }       /* & = the parent selector */
  .title { font-weight: 600; }                    /* descendant; works without & */
  &.is-selected { outline: 2px solid var(--brand); }
  @media (width >= 48rem) { padding: 2rem; }      /* nested at-rules */
}
```
Differences from Sass:
- **No string concatenation**: `&__title` (the BEM suffix trick) does **not** work in native CSS.
- A nested selector behaves like `:is()`, so it takes the **most specific** of the parent's selectors.

### `:has()`, the "parent selector" (Widely, 2026-06-19)
```css
form:has(:invalid) button[type="submit"] { opacity: .5; }
.field:has(input:focus-visible) { outline: 2px solid var(--focus); }
.card:has(> img) { grid-template-rows: 12rem auto; }    /* layout depends on content */
h2:has(+ p) { margin-block-end: .25rem; }               /* "previous sibling" selection */
body:has(dialog[open]) { overflow: hidden; }            /* scroll lock without JS */
```
Keep `:has()` anchored to a specific element. `body:has(...)` checks are fine, but very broad `:has()` rules on huge DOMs can cost style recalculation time.

### `@scope` (Newly, 2026-03-24)
Scopes styles to a DOM subtree, with an optional **lower boundary** (a "donut scope"):
```css
@scope (.card) to (.card__content) {
  img { border-radius: 8px; }   /* images in the card chrome, NOT inside the slotted content */
  :scope { padding: 1rem; }     /* :scope = the .card root */
}
```
It adds **scoping proximity** to the cascade: when specificity ties, the rule from the **closer** scope root wins. That is useful for nested light/dark themes. It is a native, lighter-weight alternative to CSS Modules or BEM for isolation. See [Styling Approaches](/Software%20Engineering/Frontend/04%20CSS/Styling%20Approaches).

### `@layer`
Layers give you priority by architecture. See [CSS Fundamentals - Cascade, Box Model & Contexts](/Software%20Engineering/Frontend/04%20CSS/CSS%20Fundamentals%20-%20Cascade%2C%20Box%20Model%20%26%20Contexts) for the full rules.
```css
@layer reset, vendor, base, components, utilities;
@import url("datepicker.css") layer(vendor);
```

---

## Motion and transitions

### `@starting-style` (Newly, 2024-08-06)
Defines the "before" state for elements that just appeared (were inserted, or changed from `display: none`), so **enter animations work with CSS only**.
```css
.toast {
  opacity: 1; translate: 0 0;
  transition: opacity .3s, translate .3s, display .3s allow-discrete, overlay .3s allow-discrete;
  @starting-style { opacity: 0; translate: 0 1rem; }
}
.toast[hidden] { display: none; opacity: 0; }   /* exit animation also works via allow-discrete */
```
The same pattern animates `[popover]:popover-open` and `dialog[open]`. `transition-behavior: allow-discrete` lets `display` and `overlay` flip at the right moment.

### View Transitions (CSS side)
- **Same-document** (SPA): Newly, 2025-10-14. JS triggers it with `document.startViewTransition(() => update())`. React 19.3 made `<ViewTransition>` stable (see [React 19 to 19.3 & React Compiler](/Software%20Engineering/Frontend/05%20React/React%2019%20to%2019.3%20%26%20React%20Compiler)).
- **Cross-document** (MPA): Limited (Chrome 126, Safari 18.2).
```css
@view-transition { navigation: auto; }                       /* opt in to MPA transitions (both pages) */
.flight-card img { view-transition-name: var(--vt-name); }   /* shared-element morph: unique name per element */
.list-item { view-transition-class: card; }                  /* style many at once */
::view-transition-old(root) { animation: fade-out .2s; }
::view-transition-new(root) { animation: fade-in .2s; }
::view-transition-group(*.card) { animation-duration: .35s; }
@media (prefers-reduced-motion: reduce) { ::view-transition-group(*) { animation: none; } }
```

### Scroll-driven animations (Limited: Chrome 115, Safari 26; no Firefox)
They tie animation progress to scroll position instead of time, and run off the main thread.
```css
/* reading progress bar */
.progress { transform-origin: 0 50%; animation: grow linear; animation-timeline: scroll(root); }
@keyframes grow { from { scale: 0 1; } to { scale: 1 1; } }

/* reveal cards as they enter the viewport */
.card { animation: fade-up linear both; animation-timeline: view(); animation-range: entry 0% cover 40%; }
```
Wrap them in `@supports (animation-timeline: view())` and respect `prefers-reduced-motion`.

### `interpolate-size` / `calc-size()` (Limited: Chrome 129)
Lets you animate to intrinsic sizes such as `height: auto`, finally.
```css
:root { interpolate-size: allow-keywords; }   /* opt in for the whole page */
details::details-content { height: 0; overflow: clip; transition: height .3s, content-visibility .3s allow-discrete; }
details[open]::details-content { height: auto; }
```
Browsers without support just snap open. It is a safe enhancement.

---

## Positioning and UI primitives

### Anchor positioning (Chrome 125, Safari 26, Firefox 147; webstatus: Limited)
Tethers an element (tooltip, menu) to another element with **no JS positioning library** (Popper or Floating UI).
```css
.btn  { anchor-name: --more-btn; }
.menu {
  position: absolute;
  position-anchor: --more-btn;
  position-area: bottom span-right;          /* place below, extend to the right */
  position-try-fallbacks: flip-block;        /* flip above if there's no room below */
  min-width: anchor-size(width);
}
```
It pairs naturally with `popover`, which gives top layer, light dismiss and Esc. HTML: `<button popovertarget="m" style="anchor-name:--more-btn">` and `<div id="m" popover class="menu">`. Guard it with `@supports (anchor-name: --a)` and keep a JS fallback if you must support older browsers.

### Popover, invoker commands, dialogs
```html
<button popovertarget="filters">Filters</button>
<div id="filters" popover>…</div>                   <!-- Newly: top layer, light dismiss, Esc -->

<button commandfor="confirm" command="show-modal">Book</button>   <!-- Newly 2025-12-12, no JS -->
<dialog id="confirm" closedby="any">…</dialog>       <!-- closedby: Limited (Chrome 134, Firefox 141) -->
```
`popover="hint"` (tooltips) and interest invokers (`interestfor`) are still Limited. See [Accessibility Essentials](/Software%20Engineering/Frontend/12%20Accessibility/Accessibility%20Essentials) for the accessibility behaviour.

### Customizable `<select>` (Limited: Chrome 135, Safari 27; no Firefox)
```css
select, ::picker(select) { appearance: base-select; }   /* opt in to a fully stylable select */
select::picker-icon { rotate: 0deg; transition: rotate .2s; }
select:open::picker-icon { rotate: 180deg; }
option:checked { font-weight: 600; }
```
You can put rich HTML (icons, flags) inside `<option>`. Browsers without support show a normal native select, so the fallback is free and it keeps native keyboard and screen-reader behaviour, unlike custom JS dropdowns.

### `field-sizing` (Newly, 2026-06-16)
```css
textarea, input.auto { field-sizing: content; min-block-size: 3lh; max-block-size: 10lh; }  /* auto-grow, no JS */
```

---

## Queries

### Container size and style queries
Size queries are Widely available, style queries Newly (2026-05-19). See [Layout - Flexbox, Grid & Container Queries](/Software%20Engineering/Frontend/04%20CSS/Layout%20-%20Flexbox%2C%20Grid%20%26%20Container%20Queries).

### Scroll-state container queries (Limited: Chrome 133 only)
```css
.header { position: sticky; top: 0; container-type: scroll-state; }
@container scroll-state(stuck: top) {
  .header > nav { box-shadow: var(--shadow-2); }   /* style only while actually stuck */
}
```
Other states: `snapped` (the current carousel item) and `scrollable` (show "more" arrows).

---

## Logic and functions in CSS

### `if()` (Limited: Chrome 137 only)
Inline conditionals on `style()`, `media()` or `supports()`:
```css
.button {
  color: if(style(--theme: dark): white; else: black);
  padding: if(media(width < 40rem): .5rem; else: 1rem);
}
```

### `@function` (Limited: Chrome 139 only)
Native custom functions, like Sass functions but evaluated live in the browser:
```css
@function --neg(--v) { result: calc(-1 * var(--v)); }
@function --fluid(--min, --max) { result: clamp(var(--min), 1rem + 2vw, var(--max)); }
.pull { margin-inline-start: --neg(1rem); }
h1    { font-size: --fluid(1.5rem, 3rem); }
```

### Typed `attr()` (Limited: Chrome 133, Firefox 155)
`attr()` used to work only in `content`. Now it can return typed values for any property:
```css
.bar { width: attr(data-w type(<length>), 0px); }       /* <div class="bar" data-w="120px"> */
.swatch { background: attr(data-color type(<color>), gray); }
```

### `sibling-index()` / `sibling-count()` (Newly, 2026-08-18)
```css
li { transition-delay: calc(sibling-index() * 50ms); }             /* staggered animation, no nth-child list */
.avatar { z-index: calc(sibling-count() - sibling-index()); }       /* first avatar on top */
```

---

## Colour and typography

### `light-dark()` (Newly, 2024-05-13)
```css
:root { color-scheme: light dark; }                      /* required: enables light-dark() and dark form controls */
body  { background: light-dark(#fff, #0f1115); color: light-dark(#111, #eee); }
.theme-dark { color-scheme: dark; }                     /* force a subtree */
```

### `contrast-color()` (Newly, 2026-04-10)
Returns black or white, whichever contrasts better with the given colour. It is ideal for user-chosen or brand-driven badge colours.
```css
.badge { background: var(--airline-color); color: contrast-color(var(--airline-color)); }
```
It picks between black and white only. It does not guarantee a WCAG AA ratio for mid-tone backgrounds, so check your token palette.

### `text-box` (Limited: Chrome 133, Safari 18.2)
Trims the built-in leading above cap height and below the baseline, so text sits optically centred in buttons and spacing matches Figma.
```css
h1, .btn { text-box: trim-both cap alphabetic; }
```

### `corner-shape` (Limited: Chrome 139)
```css
.card { border-radius: 24px; corner-shape: squircle; }   /* other values: bevel, scoop, notch… */
```

### `reading-flow` (Limited: Chrome 137)
It makes keyboard (Tab) and reading order follow the **visual** order of a flex or grid layout, fixing the classic `order`/`grid-auto-flow: dense` accessibility problem.
```css
.grid { display: grid; grid-auto-flow: dense; reading-flow: grid-rows; }
```

### Scroll markers (CSS carousels, Limited: Chrome 135)
```css
.carousel { overflow-x: auto; scroll-snap-type: x mandatory; scroll-marker-group: after; }
.carousel > .slide { scroll-snap-align: center; }
.carousel > .slide::scroll-marker { content: ""; width: .6rem; aspect-ratio: 1; border-radius: 50%; background: gray; }
.carousel > .slide::scroll-marker:target-current { background: black; }   /* the active dot */
.carousel::scroll-button(right) { content: "→"; }
.carousel::scroll-button(left)  { content: "←"; }
```

---

## What you can use today in production (Sept 2026)

<Check>
**Safe everywhere (Baseline): just use it**

- **Nesting**, **`:has()`**, **`@layer`**, **container size queries** and `cq*` units (all Widely available).
- **`@starting-style`** + `transition-behavior: allow-discrete` for enter/exit animations.
- **Popover API**, **invoker commands** (`commandfor`/`command`), `<dialog>` + `requestClose()`, `:open`.
- **Same-document View Transitions** (as progressive motion).
- **`@scope`**, **container style queries**, name-only container queries (all Newly in 2026. Fine unless you support browser versions from before 2026; check your analytics).
- **`light-dark()`**, **`contrast-color()`**, **`field-sizing: content`**.
- Plus the older staples: subgrid, `aspect-ratio`, `gap` in flex, logical properties, `clamp()`, `svh`/`dvh`, media range syntax, `:focus-visible`, `accent-color`, `color-mix()`, `oklch()`.
</Check>

<Warning>
**Progressive enhancement only (wrap in `@supports`, ship a fallback)**

- **Anchor positioning**: in all three engines now, but webstatus still says Limited. Use it with a fallback for older versions.
- **Customizable select**, **cross-document view transitions**, **scroll-driven animations**, `text-box`, `interpolate-size`, `corner-shape`, typed `attr()`, `sibling-index()` (very new).
- These are fine because a missing feature means "no animation" or "a native select", not a broken page.
</Warning>

<Warning>
**Not yet (Chrome-only experiments)**

`if()`, `@function`, `reading-flow`, scroll-state queries, scroll markers, `popover="hint"`, interest invokers. Prototype with them, but don't depend on them.
</Warning>

<Info>
**Feature-detect pattern**

```css
.menu { /* fallback: JS-positioned by Floating UI */ }
@supports (anchor-name: --a) {
  .menu { position-anchor: --trigger; position-area: bottom; position-try-fallbacks: flip-block; }
}
@supports not (animation-timeline: view()) {
  .card { opacity: 1; }   /* make sure content is visible without the animation */
}
```
</Info>

## Interview questions

<Accordion title="What is Baseline, and how do you decide whether a CSS feature is production-ready?">
Baseline (from the WebDX group, shown on MDN, caniuse and webstatus.dev) marks features as **Newly available** (in all core browsers: Chrome, Edge, Firefox and Safari, desktop and mobile) and **Widely available** (30 months after that). Compare it with your own browser analytics. Limited features can still ship as progressive enhancement when the fallback is acceptable (no animation, native select).
</Accordion>

<Accordion title="How does :has() change how we write CSS?">
It selects an element based on its descendants or following siblings, which gives CSS a parent and previous-sibling selector. That removes lots of JS class toggling: form state styling (`form:has(:invalid)`), layout variants based on content (`.card:has(img)`), and page state (`body:has(dialog[open])`). Its specificity is that of its most specific argument. It is Widely available as of June 2026.
</Accordion>

<Accordion title="@scope vs CSS Modules vs BEM?">
All three avoid style leakage. BEM is a naming convention (human discipline). CSS Modules hash class names at build time (tooling). `@scope` is native: it limits selectors to a subtree, supports a lower boundary (donut scope) so styles don't leak into slotted content, and adds scoping proximity to the cascade. It became Newly available in March 2026.
</Accordion>

<Accordion title="How do you animate an element appearing from display: none?">
Use `@starting-style` to define the initial state, plus `transition-behavior: allow-discrete` (or `allow-discrete` in the shorthand) on `display` (and `overlay` for top-layer elements) so the discrete property flips at the right time. That gives enter and exit animations without JS or keeping the element mounted.
</Accordion>

<Accordion title="How would you build a tooltip or dropdown in 2026?">
Use `popover` (top layer, light dismiss, Esc, no z-index fights) triggered by `popovertarget` or invoker commands, positioned with CSS anchor positioning (`position-anchor`, `position-area`, `position-try-fallbacks`). Anchor positioning is in Chrome, Safari 26 and Firefox 147, but still flagged Limited by webstatus, so keep a fallback (Floating UI) behind `@supports` if you support older versions. For a select-like control, try the customizable `<select>` first, since it keeps native accessibility.
</Accordion>

<Accordion title="What's new in CSS for theming?">
`light-dark()` with `color-scheme` switches values per scheme (Baseline). `contrast-color()` picks a readable black or white text colour (Newly, April 2026). Container style queries let variants be driven by a custom property (Newly, May 2026). `if()` and `@function` (Chrome-only) will allow conditional tokens and computed values in the future.
</Accordion>

## Related
- [What's New in Frontend 2025-2026](/Software%20Engineering/Frontend/What's%20New%20in%20Frontend%202025-2026)
- [CSS Fundamentals - Cascade, Box Model & Contexts](/Software%20Engineering/Frontend/04%20CSS/CSS%20Fundamentals%20-%20Cascade%2C%20Box%20Model%20%26%20Contexts)
- [Layout - Flexbox, Grid & Container Queries](/Software%20Engineering/Frontend/04%20CSS/Layout%20-%20Flexbox%2C%20Grid%20%26%20Container%20Queries)
- [Styling Approaches](/Software%20Engineering/Frontend/04%20CSS/Styling%20Approaches)
- [Accessibility Essentials](/Software%20Engineering/Frontend/12%20Accessibility/Accessibility%20Essentials)
- [Modern Web Platform APIs 2025-2026](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Modern%20Web%20Platform%20APIs%202025-2026)
- [React 19 to 19.3 & React Compiler](/Software%20Engineering/Frontend/05%20React/React%2019%20to%2019.3%20%26%20React%20Compiler)
- [Design Systems](/Software%20Engineering/Frontend/09%20Architecture/Design%20Systems)

## Sources
- Baseline data: [webstatus.dev](https://webstatus.dev), [web-features explorer: newly available](https://web-platform-dx.github.io/web-features-explorer/newly-available/)
- [Interop 2026 (web.dev)](https://web.dev/blog/interop-2026)
- [MDN: Firefox 147 release notes](https://developer.mozilla.org/en-US/docs/Mozilla/Firefox/Releases/147) (anchor positioning)
- [WebKit: Safari 27 features](https://webkit.org/blog/18325/webkit-features-for-safari-27-0/)
- MDN: [CSS nesting](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_nesting), [:has()](https://developer.mozilla.org/en-US/docs/Web/CSS/:has), [@scope](https://developer.mozilla.org/en-US/docs/Web/CSS/@scope), [@starting-style](https://developer.mozilla.org/en-US/docs/Web/CSS/@starting-style), [Anchor positioning](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_anchor_positioning), [View Transition API](https://developer.mozilla.org/en-US/docs/Web/API/View_Transition_API), [Scroll-driven animations](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_scroll-driven_animations), [light-dark()](https://developer.mozilla.org/en-US/docs/Web/CSS/color_value/light-dark)
- Chrome for Developers: [CSS Wrapped](https://developer.chrome.com/blog/) (yearly round-up of new CSS)
