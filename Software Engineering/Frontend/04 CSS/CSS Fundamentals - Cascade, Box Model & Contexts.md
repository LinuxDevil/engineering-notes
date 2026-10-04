---
title: "CSS Fundamentals - Cascade, Box Model & Contexts"
tags: [frontend, frontend/css]
aliases: [CSS Fundamentals, Cascade, Specificity, Box Model, Stacking Context, BFC, Margin Collapsing]
updated: 2026-09-27
---
# CSS Fundamentals - Cascade, Box Model & Contexts

<Note>
**TL;DR**

Every CSS bug is one of three things: the **cascade** picked a different declaration than you expected (origin, `@layer`, specificity, order), the **box** is sized or positioned against something you didn't expect (box model, containing block, formatting context, margin collapsing), or the **paint order** is not what you think (stacking contexts). Learn the rules behind those three and CSS stops being "magic".
</Note>

{/* diagram source: /diagrams/diagram-249.mmd */}
![diagram](/diagrams/diagram-249.svg)

See [How Browsers Render - Critical Rendering Path](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/How%20Browsers%20Render%20-%20Critical%20Rendering%20Path) for how the CSSOM is built and where layout, paint and composite happen in the pipeline.

---

## 1. Selectors (quick map)

| Kind | Examples | Specificity bucket |
|---|---|---|
| ID | `#main` | A (ids) |
| Class, attribute, pseudo-class | `.btn`, `[type="email"]`, `:hover`, `:nth-child(2)` | B |
| Type, pseudo-element | `button`, `::before`, `::placeholder` | C |
| Universal, combinators | `*`, `>`, `+`, `~`, ` ` (descendant) | none (0) |
| Functional pseudo-classes | `:is()`, `:not()`, `:has()` | the **most specific argument** |
| Zero-specificity wrapper | `:where()` | always 0 |

Combinators: `A B` (descendant), `A > B` (child), `A + B` (next sibling), `A ~ B` (any following sibling). `:has()` finally gives a "parent/previous sibling" selector: `label:has(+ input:invalid)`. See [Modern CSS 2025-2026](/Software%20Engineering/Frontend/04%20CSS/Modern%20CSS%202025-2026).

## 2. Specificity (with worked examples)

Specificity is a 3-part tuple **(A, B, C)** = (IDs, classes/attributes/pseudo-classes, types/pseudo-elements). Compare left to right; the first bigger column wins. It is **not** base-10: 11 classes never beat 1 ID.

<Info>
**Worked examples**

| Selector | Specificity | Why |
|---|---|---|
| `*` | (0,0,0) | universal has none |
| `li` | (0,0,1) | one type |
| `ul li a` | (0,0,3) | three types |
| `a:hover` | (0,1,1) | pseudo-class + type |
| `.btn.primary` | (0,2,0) | two classes |
| `input[type="text"]` | (0,1,1) | attribute counts like a class |
| `p::first-line` | (0,0,2) | pseudo-element counts like a type |
| `#nav .item > a` | (1,1,1) | 1 id, 1 class, 1 type |
| `:is(#a, .b) p` | (1,0,1) | `:is()` takes its most specific argument (`#a`) even if `.b` matched |
| `:where(#a, .b) p` | (0,0,1) | `:where()` is always zero, great for library defaults |
| `a:not(.x, #y)` | (1,0,1) | `:not()` also takes its most specific argument |
| `li:nth-child(2 of .hot)` | (0,2,1) | the pseudo-class (1) plus its selector argument (1) |
| `.card:has(> img)` | (0,1,1) | `.card` + the argument `img` |
| `style="color:red"` | beats any selector | element-attached style, a separate cascade step |
| `color: red !important` | beats all normal declarations | importance is decided before specificity |
</Info>

<Info>
**Which colour wins?**

```html
<nav id="nav"><a class="link active" href="#">Home</a></nav>
```
```css
#nav a            { color: blue; }   /* (1,0,1) */
.link.active      { color: green; }  /* (0,2,0) */
nav .link:hover   { color: orange; } /* (0,2,1) */
```
**Blue.** (1,0,1) beats (0,2,x) because the ID column is compared first. On hover it is still blue. Fixes, from best to worst: lower the ID rule (`:where(#nav) a`), put the rules in ordered `@layer`s, or stop styling by ID. Using `!important` only starts an arms race.
</Info>

<Tip>
**Keeping specificity flat**

- Style with single classes (BEM, CSS Modules, utilities). See [Styling Approaches](/Software%20Engineering/Frontend/04%20CSS/Styling%20Approaches).
- Wrap resets and library defaults in `:where()` or a low `@layer` so they are easy to override.
- Avoid IDs and deep descendant chains in stylesheets.
</Tip>

## 3. The cascade: full sorting order

When several declarations set the same property on the same element, the cascade sorts them by these criteria **in order**. It moves to the next step only on a tie:

{/* diagram source: /diagrams/diagram-250.mmd */}
![diagram](/diagrams/diagram-250.svg)

**Step 1, origin and importance**, from weakest to strongest:
1. User-agent (browser defaults), normal
2. User styles, normal
3. **Author (your CSS), normal**
4. CSS animations (`@keyframes`)
5. Author `!important`
6. User `!important`
7. User-agent `!important`
8. CSS transitions (always win while running)

Note that `!important` **inverts** the origin order: that is how a user's accessibility stylesheet can beat yours.

### Cascade layers (`@layer`)
Layers let you decide priority by **architecture** instead of specificity. They have been Baseline in all major browsers since 2022.

```css
/* declare the order once, up front: later = stronger */
@layer reset, base, components, utilities;

@layer reset      { *, *::before, *::after { box-sizing: border-box; margin: 0; } }
@layer components { #hero .title { margin-top: 2rem; } }   /* (1,1,0) */
@layer utilities  { .mt-0 { margin-top: 0; } }              /* (0,1,0) */

/* <h1 id="hero" class="title mt-0"> ... gets margin-top: 0 */
```

`.mt-0` wins even with lower specificity, because `utilities` is a later layer. Specificity only matters **inside** the same layer.

Layer rules to remember:
- **Unlayered styles beat all layered styles** (for normal declarations).
- For `!important` the order **flips**: an earlier layer's `!important` beats a later layer's, and layered `!important` beats unlayered `!important`. This lets a reset layer lock something down.
- Import third-party CSS into a low layer: `@import url(lib.css) layer(vendor);`.
- `revert-layer` rolls a property back to the value from the previous layer.
- Tailwind v4 emits its CSS into real cascade layers (`theme`, `base`, `components`, `utilities`).

## 4. Inheritance and the "value pipeline"

If no declaration wins for a property, then:
- **Inherited properties** take the parent's computed value. Examples: `color`, `font-*`, `line-height`, `letter-spacing`, `text-align`, `visibility`, `cursor`, `list-style`, `direction`, and **all custom properties** (`--x`).
- **Non-inherited properties** take their **initial** value. Examples: `margin`, `padding`, `border`, `background`, `width`/`height`, `display`, `position`, `overflow`, `opacity`, `transform`.

Global keywords work on every property:

| Keyword | Meaning |
|---|---|
| `inherit` | take the parent's computed value |
| `initial` | the spec's initial value (`display: initial` is `inline`, not `block`) |
| `unset` | `inherit` if the property is inherited, otherwise `initial` |
| `revert` | roll back to the user-agent or user origin value (e.g. restores `display: block` on a `div`) |
| `revert-layer` | roll back to the previous cascade layer |

`all: unset` or `all: revert` resets every property at once. This is handy for a "clean" button.

**How a value is resolved** ("Properties" and "Values" in my checklist):
1. **Declared**: every matching declaration.
2. **Cascaded**: the one the cascade picks.
3. **Specified**: the cascaded value, or the inherited or initial value if there is none.
4. **Computed**: relative units resolved where possible (`em` becomes `px`, `inherit` is resolved). This is what children inherit, and what `getComputedStyle()` mostly returns.
5. **Used**: after layout (`width: 50%` becomes `412px`).
6. **Actual**: after device rounding.

<Info>
**Why `line-height: 1.5` beats `line-height: 1.5em`**

```css
.parent { font-size: 16px; line-height: 1.5em; } /* computed: 24px, children inherit 24px */
.child  { font-size: 32px; }                     /* line-height is still 24px: lines overlap */
.parent { line-height: 1.5; }                    /* unitless: children inherit the factor, so 48px */
```
</Info>

## 5. Box model and `box-sizing`

Every element generates a box: **content → padding → border → margin**. Margin is outside the border and is transparent.

```
┌──────────── margin ────────────┐
│ ┌────────── border ──────────┐ │
│ │ ┌──────── padding ───────┐ │ │
│ │ │        content         │ │ │
│ │ └────────────────────────┘ │ │
│ └────────────────────────────┘ │
└────────────────────────────────┘
```

| `box-sizing` | What `width` means | Rendered width of `width:200px; padding:20px; border:5px` |
|---|---|---|
| `content-box` (default) | content only | 200 + 40 + 10 = **250px** |
| `border-box` | content + padding + border | **200px** (content shrinks to 150px) |

```css
/* the universal reset almost everyone uses */
*, *::before, *::after { box-sizing: border-box; }
```

Other box facts:
- Percentage `padding` and `margin`, **even vertical ones**, resolve against the containing block's **width**. That is the old "padding-top: 56.25%" 16:9 hack; use `aspect-ratio: 16 / 9` today.
- `outline` and `box-shadow` do not take up layout space. That is why outlines are good focus indicators: they don't shift layout.
- `min-width`/`max-width` beat `width`; `min-*` beats `max-*` when they conflict.
- Intrinsic sizing keywords: `min-content`, `max-content`, `fit-content`, and the newer `stretch` keyword (Safari 27 added it in Sept 2026).

### Replaced elements
**Replaced elements** have content that comes from outside CSS: `<img>`, `<video>`, `<iframe>`, `<embed>`, `<object>`, and in practice form controls such as `<input>`. They:
- have an **intrinsic size and aspect ratio**, so set `width` and `height` attributes to avoid CLS (see [Core Web Vitals](/Software%20Engineering/Frontend/10%20Performance/Core%20Web%20Vitals));
- are controlled with `object-fit` (`cover`, `contain`) and `object-position`;
- cannot have `::before`/`::after` (there is no content box for CSS to insert into);
- are inline by default. The gap under an image is the text baseline gap: fix it with `display: block` or `vertical-align: middle`.

## 6. Display types and the visual formatting model

The **visual formatting model** is how the browser turns the element tree into boxes:
1. Each element generates zero or more boxes, depending on `display`.
2. The boxes are laid out by the **formatting context** they participate in: block, inline, flex, grid, or table.
3. The positioning scheme applies: normal flow, float, or absolute.

`display` really has two parts:
- **Outer display**: how the box behaves toward its siblings (`block` or `inline`).
- **Inner display**: how it lays out its children (`flow`, `flow-root`, `flex`, `grid`, `table`).

| Value | Two-value form | Notes |
|---|---|---|
| `block` | `block flow` | full width, starts on a new line |
| `inline` | `inline flow` | `width`/`height` ignored; vertical margins don't push lines |
| `inline-block` | `inline flow-root` | sits in a line but has a box you can size; creates a BFC |
| `flex` / `inline-flex` | `block flex` / `inline flex` | see [Layout - Flexbox, Grid & Container Queries](/Software%20Engineering/Frontend/04%20CSS/Layout%20-%20Flexbox%2C%20Grid%20%26%20Container%20Queries) |
| `grid` / `inline-grid` | `block grid` / `inline grid` | |
| `flow-root` | `block flow-root` | a block that creates a new BFC (the modern clearfix) |
| `contents` | none | the box disappears and its children are promoted. Older browsers dropped the element's role from the accessibility tree, so test buttons and lists |
| `none` | none | no box; removed from the accessibility tree as well |

**Hiding things** (a classic interview question):

| Technique | Takes space? | Clickable? | Screen reader? | Animatable? |
|---|---|---|---|---|
| `display: none` | no | no | hidden | no (but see `@starting-style` and `transition-behavior: allow-discrete`) |
| `visibility: hidden` | yes | no | hidden | yes (discrete) |
| `opacity: 0` | yes | **yes** | **still read** | yes |
| `.sr-only` clip pattern | no | no | **read** | n/a |
| `content-visibility: hidden` | keeps size if `contain-intrinsic-size` is set | no | hidden | n/a |

## 7. Position types

| `position` | In normal flow? | Offsets (`top`, `inset`) are relative to | Creates stacking context? |
|---|---|---|---|
| `static` (default) | yes | offsets are ignored | no |
| `relative` | yes (its original space is kept) | its own normal position | only with a `z-index` other than `auto` |
| `absolute` | no | the **containing block** (nearest positioned ancestor's padding box) | only with a `z-index` other than `auto` |
| `fixed` | no | the viewport (unless an ancestor traps it, see below) | **always** |
| `sticky` | yes, until the threshold | its nearest **scrolling ancestor** | **always** |

<Warning>
**Why `position: sticky` "doesn't work"**

- There is no threshold: you need `top: 0` (or another inset).
- An ancestor has `overflow: hidden`/`auto`/`scroll`, so it becomes the scroll container and the element sticks inside it, not the page. `overflow: clip` doesn't create a scroll container.
- The parent is only as tall as the sticky element, so it has no room to stick.
- In a flex or grid row, the item is stretched to full height. Add `align-self: start`.
</Warning>

## 8. Containing block

Percentages, `inset` values and absolute positioning all resolve against the **containing block**. It is *not* always the parent.

| Element's `position` | Containing block is… |
|---|---|
| `static`, `relative`, `sticky` | the **content box** of the nearest block container ancestor (or of the ancestor that forms a formatting context) |
| `absolute` | the **padding box** of the nearest ancestor with `position` other than `static`… |
| `fixed` | the **viewport**… |
| …**unless** | an ancestor has `transform`, `perspective`, `filter`, `backdrop-filter`, `contain: layout/paint`, `container-type` (size queries), or a `will-change` naming one of these. That ancestor then becomes the containing block for **both** absolute and fixed descendants |

<Info>
**The "position: fixed is stuck inside a card" bug**

```css
.card  { transform: translateY(0); }        /* harmless-looking animation leftover */
.modal { position: fixed; inset: 0; }        /* now covers only .card, not the viewport */
```
**Fixes:**
- Render the modal outside the card with a portal (`createPortal` in React).
- Or use `<dialog>.showModal()` or `popover`, which render in the **top layer** and ignore ancestors entirely. See [Accessibility Essentials](/Software%20Engineering/Frontend/12%20Accessibility/Accessibility%20Essentials).
</Info>

## 9. Block formatting context (BFC)

A **BFC** is an isolated layout region in which block boxes stack vertically. Things inside can't affect things outside. A BFC:
- **contains its floats** (the parent grows to wrap floated children);
- **does not overlap external floats** (you can build a "media object" without clearfix);
- **stops margin collapsing** between itself and its children.

**What creates a BFC:**
- the root `<html>`
- floats
- `position: absolute` or `fixed`
- `display: inline-block`, `table-cell`, `table-caption`, or `flow-root`
- `overflow` other than `visible` or `clip`
- flex and grid **items** (flex and grid containers create their own flex/grid formatting contexts)
- `contain: layout`, `content` or `paint`
- multi-column containers
- `column-span: all`

<Info>
**Collapsed parent with floated children**

```css
.gallery        { border: 1px solid; }
.gallery img    { float: left; width: 33%; }   /* the parent collapses to 0 height */
.gallery        { display: flow-root; }         /* modern fix: new BFC contains floats */
/* old clearfix: .gallery::after { content: ""; display: table; clear: both; } */
```
</Info>

Related contexts: an **inline formatting context** (line boxes, `vertical-align`, `line-height`), and flex and grid formatting contexts. Flex and grid items **never** margin-collapse and **ignore `float`**.

## 10. Margin collapsing

**Vertical** margins between block boxes in normal flow can combine into one margin. The result is the **larger** of the two. With one negative margin, they are summed; with both negative, the most negative wins. Horizontal margins never collapse.

It happens in three cases:
1. **Adjacent siblings**: `h2 { margin-bottom: 20px }` + `p { margin-top: 30px }` gives **30px**, not 50px.
2. **Parent and first/last child**: if nothing separates them (no border, no padding, no inline content, no BFC, no `height` for the bottom case), the child's `margin-top` "leaks" out of the parent.
3. **Empty blocks**: an empty block's own top and bottom margins collapse together.

<Info>
**The "margin leaks out of my card" bug**

```css
.card    { background: #eef; }
.card h2 { margin-top: 24px; }  /* the gap appears ABOVE the card, not inside it */
```
**Fixes (any one):**
- `.card { padding-top: 1px }`, or a border.
- `.card { display: flow-root }` (creates a new BFC).
- Make `.card` a flex or grid container.

A common modern habit: use `gap` in flex/grid and only single-direction margins (`margin-block-end`) so collapsing never surprises you.
</Info>

It does **not** collapse for floats, absolutely positioned elements, flex/grid items, inline-block elements, or across a BFC boundary.

## 11. Stacking context and z-index debugging

The page is painted in layers along the z-axis. A **stacking context** is a group that is painted **as one unit**: its children's `z-index` values only compete **inside** it. A child with `z-index: 9999` cannot escape a parent context that sits below a sibling with `z-index: 1`.

**Paint order inside one stacking context**, from back to front:
1. the context's own background and borders
2. children with negative `z-index`
3. in-flow block boxes
4. floats
5. inline content
6. positioned children with `z-index: auto` or `0`, in DOM order
7. children with positive `z-index`

**What creates a stacking context** (memorise the common ones):
- the root element
- `position: relative` or `absolute` **with a `z-index` other than `auto`**
- `position: fixed` or `sticky` (always)
- flex or grid **items** with a `z-index` other than `auto` (even without `position`)
- `opacity` < 1
- `transform`, `scale`, `rotate`, `translate`, `filter`, `backdrop-filter`, `perspective`, `clip-path`, `mask` other than none
- `mix-blend-mode` other than `normal`
- `isolation: isolate` (the clean, explicit way)
- `will-change` naming any of the above
- `contain: layout`, `paint`, `strict` or `content`; `container-type: size` or `inline-size`
- `view-transition-name` other than none
- elements in the **top layer** (modal `<dialog>`, popovers, fullscreen), which sit above everything regardless of z-index

{/* diagram source: /diagrams/diagram-251.mmd */}
![diagram](/diagrams/diagram-251.svg)

<Info>
**z-index debugging checklist**

1. Is the element **positioned** (or a flex/grid item)? Otherwise `z-index` does nothing.
2. Walk **up** the tree. Which ancestor creates a stacking context (`transform`, `opacity`, `filter`, `will-change`, sticky…)? Compare z-index **at the level of the sibling contexts**, not the leaves.
3. Chrome DevTools: the **Layers** panel and the Elements > Computed pane show what forms contexts and layers. Edge's 3D View helps too.
4. Fix it structurally: raise the ancestor's context, add `isolation: isolate` to contain a component, portal the overlay to `body`, or use the top layer (`popover`, `<dialog>`).
5. Use a **z-index scale** as tokens (`--z-dropdown: 100; --z-sticky: 200; --z-modal: 300; --z-toast: 400`) instead of magic numbers.
</Info>

## 12. Units

| Group | Units | Use for |
|---|---|---|
| Absolute | `px` (1/96 in, a CSS pixel, not a device pixel) | borders, shadows, fine details |
| Font-relative | `rem` (root font size), `em` (the element's own font size; for `font-size` itself, the parent's), `ch` (width of "0"), `ex`, `cap`, `lh` (element line height), `rlh` (root line height) | type scale and spacing (`rem`), component-relative padding (`em`), readable measure (`max-width: 65ch`), vertical rhythm (`lh`/`rlh`) |
| Percent | `%` | relative to the containing block (or the font size, for `font-size`) |
| Viewport | `vw`, `vh`, `vmin`, `vmax`, `vi`, `vb` | hero sections, fluid type |
| **Small / large / dynamic viewport** | `svh`, `lvh`, `dvh` (plus `sv*`, `lv*`, `dv*` for all axes) | full-height mobile layouts |
| Container | `cqw`, `cqh`, `cqi`, `cqb`, `cqmin`, `cqmax` | sizing relative to a query container |
| Grid | `fr` | a share of the free space in grid tracks |
| Angles and time | `deg`, `turn`, `rad`; `s`, `ms` | transforms, animations |

### Dynamic viewport units (the mobile `100vh` bug)
On mobile, browser UI (the URL bar and toolbars) expands and retracts. Classic `100vh` equals the **largest** viewport, so bottom content hides behind the toolbar.

| Unit | Equals | Behaviour |
|---|---|---|
| `svh` | the **small** viewport (toolbars shown) | stable, never hidden behind UI. **Best default** |
| `lvh` | the **large** viewport (toolbars hidden) | same as the old `vh` |
| `dvh` | the **current** viewport | updates as the toolbars move. Can cause layout jank and reflows while scrolling, so use it for fixed full-screen UIs (modals, app shells) |

```css
.hero      { min-height: 100svh; }              /* never hidden behind the URL bar */
.app-shell { height: 100dvh; }                  /* fills exactly what's visible */
.fallback  { min-height: 100vh; min-height: 100svh; } /* progressive enhancement */
```
These units have worked in all major browsers since late 2022.

### Fluid values
```css
h1 { font-size: clamp(1.75rem, 1.2rem + 2.5vw, 3rem); }   /* min, preferred, max */
.stack > * + * { margin-block-start: 1lh; }                 /* vertical rhythm from line height */
```
Always mix `rem` into fluid font sizes (as above). Pure `vw` ignores the user's zoom and font size, which fails WCAG 1.4.4 Resize Text.

**Vertical rhythm** means keeping vertical spacing on a consistent baseline grid (multiples of the line height). The modern tools for it are `lh`/`rlh` units, a spacing-token scale, and `text-box: trim-both cap alphabetic` (see [Modern CSS 2025-2026](/Software%20Engineering/Frontend/04%20CSS/Modern%20CSS%202025-2026)), which removes the extra leading above and below text so spacing matches the design.

## 13. Logical properties (RTL-ready CSS)

Physical properties (`left`, `margin-right`) are tied to the screen. **Logical properties** follow the **writing mode and direction**:
- `inline` = the text direction (horizontal in English and Arabic).
- `block` = the direction blocks stack in (vertical).
- `start`/`end` flip automatically in RTL.

| Physical | Logical |
|---|---|
| `width` / `height` | `inline-size` / `block-size` |
| `margin-left` / `margin-right` | `margin-inline-start` / `margin-inline-end` |
| `padding-top` + `padding-bottom` | `padding-block` |
| `top` / `left` | `inset-block-start` / `inset-inline-start` |
| `border-top-left-radius` | `border-start-start-radius` |
| `text-align: left` | `text-align: start` |
| `float: left` | `float: inline-start` |

<Info>
**One stylesheet for English and Arabic**

```css
.flight-card__icon { margin-inline-end: 0.75rem; }      /* right gap in LTR, left gap in RTL */
.badge             { inset-inline-end: 0; inset-block-start: 0; }
.arrow             { transform: scaleX(1); }
[dir="rtl"] .arrow { transform: scaleX(-1); }           /* mirror directional icons only */
```
Set `<html lang="ar" dir="rtl">` and the logical properties flip on their own. Flexbox and grid are already logical: `justify-content: flex-start` follows `dir`.
</Info>

## 14. Transforms, transitions, animations, filters and GPU compositing

- **Transforms** (`transform`, or the individual `translate`, `rotate`, `scale` properties) move boxes visually **without affecting layout**. They create a stacking context and a containing block.
- **Transitions** animate between two states when a property changes: `transition: transform 200ms ease-out`. To transition from `display: none`, use `@starting-style` plus `transition-behavior: allow-discrete`.
- **Animations** use `@keyframes` for multi-step or looping motion: `animation: spin 1s linear infinite`.
- **Filters**: `filter: blur() brightness() drop-shadow()` applies to the element; `backdrop-filter: blur(8px)` applies to what is behind it (glass effects). Both are expensive on large areas.

### "How do I make the GPU do the work instead of the CPU?"
The owner's interview note says "use `transform` for GPU acceleration". That is right, and here is the full reasoning.
- Changing **geometry** (`width`, `top`, `margin`) triggers **layout, then paint, then composite** on the main thread.
- Changing colours or `background` triggers **paint, then composite**.
- Changing only **`transform` and `opacity`** (and usually `filter`) can skip layout and paint. The element sits on its own **compositor layer**, and the GPU compositor just moves or blends that texture, even while the main thread is busy.
- `will-change: transform` (or an existing transform animation) asks the browser to **promote** the element to its own layer ahead of time.

<Warning>
**Layer promotion is not free**

Every layer costs GPU memory, and too many layers cause "layer explosion". Add `will-change` shortly before an animation and remove it afterwards. Never put `* { will-change: transform }` or `translateZ(0)` everywhere. Profile in DevTools (Performance, Layers, and the rendering option "Paint flashing").
</Warning>

<Info>
**Animate a drawer the cheap way**

```css
.drawer { transform: translateX(100%); transition: transform 250ms ease; }
.drawer.open { transform: translateX(0); }          /* compositor-only */
/* avoid: .drawer { right: -300px; transition: right 250ms } which re-runs layout every frame */
@media (prefers-reduced-motion: reduce) { .drawer { transition: none; } }
```
</Info>

## Interview questions

<Accordion title="What is CSS specificity and how is it calculated?">
It is a weight (A, B, C) = (ID selectors, class/attribute/pseudo-class selectors, type/pseudo-element selectors), compared column by column. `:is()`, `:not()` and `:has()` take their most specific argument; `:where()` is 0; `*` and combinators are 0. Inline `style` beats any selector, and `!important` beats normal declarations. Specificity only decides between declarations with the same origin, importance and layer. Order of appearance breaks exact ties.
</Accordion>

<Accordion title="Walk me through the cascade. Where do @layer and !important fit?">
Origin and importance come first (user-agent, user, author, animations, then the important origins in reverse, then transitions). Then shadow context, then inline style, then **cascade layers** (later layers win for normal declarations, unlayered beats layered, and the order flips for `!important`), then specificity, then source order. Layers let a design system give utilities priority over components without escalating specificity.
</Accordion>

<Accordion title="What is a stacking context, and why doesn't my z-index: 9999 work?">
A stacking context is a group painted atomically. Its children's z-index only competes inside it. If an ancestor creates a context (transform, opacity < 1, filter, sticky/fixed, a positioned element with z-index, `isolation`, `will-change`…) and that ancestor sits below a sibling, nothing inside can rise above that sibling. Also check that `z-index` is on a positioned element or a flex/grid item. Fix it at the ancestor level, portal the element, or use the top layer.
</Accordion>

<Accordion title="What is a block formatting context? Name ways to create one.">
An independent layout region where blocks stack vertically. It contains floats, avoids overlapping outside floats, and prevents margin collapsing with its children. It is created by `display: flow-root`, `overflow` other than visible/clip, floats, absolute/fixed positioning, inline-block, table cells, flex/grid items, `contain: layout/paint`, and multicol.
</Accordion>

<Accordion title="Explain margin collapsing and how to prevent it.">
Adjacent vertical margins of block boxes in normal flow merge into the larger one. It happens between siblings, between a parent and its first/last child when nothing separates them, and within empty blocks. Prevent it with padding or a border on the parent, a new BFC (`flow-root`), using flex/grid (items don't collapse), or by spacing with `gap` and single-direction margins.
</Accordion>

<Accordion title="content-box vs border-box?">
With `content-box`, `width` sets the content only, and padding and border are added on top. With `border-box`, `width` includes padding and border. Most codebases set `border-box` globally because it makes sizing predictable, especially with percentage widths.
</Accordion>

<Accordion title="What is the containing block of an absolutely positioned element? And of a fixed one?">
Absolute: the padding box of the nearest ancestor whose `position` isn't `static`. Fixed: the viewport. For **both**, an ancestor with `transform`, `filter`, `perspective`, `contain: paint/layout`, `container-type`, or a matching `will-change` becomes the containing block instead. That is the classic "fixed modal trapped in a card" bug.
</Accordion>

<Accordion title="100vh vs 100dvh vs 100svh?">
On mobile, `vh` is the large viewport (toolbars hidden), so content can end up under the URL bar. `svh` is the small viewport (always fully visible), `lvh` is the large one, and `dvh` tracks the current size and updates as the toolbars move. Default to `svh` for sections, and use `dvh` for full-screen app shells or modals when you accept the resize cost.
</Accordion>

<Accordion title="Which properties are cheap to animate, and why?">
`transform` and `opacity` (and often `filter`). They can be handled by the compositor on the GPU without layout or paint. Animating `width`, `top` or `margin` forces layout on every frame. Use `will-change` sparingly to pre-promote layers.
</Accordion>

## Related
- [How Browsers Render - Critical Rendering Path](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/How%20Browsers%20Render%20-%20Critical%20Rendering%20Path)
- [Layout - Flexbox, Grid & Container Queries](/Software%20Engineering/Frontend/04%20CSS/Layout%20-%20Flexbox%2C%20Grid%20%26%20Container%20Queries)
- [Modern CSS 2025-2026](/Software%20Engineering/Frontend/04%20CSS/Modern%20CSS%202025-2026)
- [Styling Approaches](/Software%20Engineering/Frontend/04%20CSS/Styling%20Approaches)
- [Accessibility Essentials](/Software%20Engineering/Frontend/12%20Accessibility/Accessibility%20Essentials)
- [Core Web Vitals](/Software%20Engineering/Frontend/10%20Performance/Core%20Web%20Vitals)
- [Performance Optimization Playbook](/Software%20Engineering/Frontend/10%20Performance/Performance%20Optimization%20Playbook)

## Sources
- MDN: [Specificity](https://developer.mozilla.org/en-US/docs/Web/CSS/Specificity), [Cascade](https://developer.mozilla.org/en-US/docs/Web/CSS/Cascade), [@layer](https://developer.mozilla.org/en-US/docs/Web/CSS/@layer), [Inheritance](https://developer.mozilla.org/en-US/docs/Web/CSS/Inheritance), [Box model](https://developer.mozilla.org/en-US/docs/Learn/CSS/Building_blocks/The_box_model), [Containing block](https://developer.mozilla.org/en-US/docs/Web/CSS/Containing_block), [Block formatting context](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_display/Block_formatting_context), [Margin collapsing](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_box_model/Mastering_margin_collapsing), [Stacking context](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_positioned_layout/Understanding_z-index/Stacking_context), [Visual formatting model](https://developer.mozilla.org/en-US/docs/Web/CSS/Visual_formatting_model), [Replaced elements](https://developer.mozilla.org/en-US/docs/Web/CSS/Replaced_element), [Logical properties](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_logical_properties_and_values), [Length units](https://developer.mozilla.org/en-US/docs/Web/CSS/length)
- web.dev: [Large, small and dynamic viewport units](https://web.dev/blog/viewport-units), [Cascade layers](https://web.dev/learn/css/cascade), [Animations and performance](https://web.dev/articles/animations-guide)
- Specificity calculator: [polypane.app/css-specificity-calculator](https://polypane.app/css-specificity-calculator/)
- Checklist origin: the owner's `JS Concepts` note (CSS section) and `Front-End System Design/Core Fundamentals` (box model, positioning, formatting context, stacking context, rendering, composition layers, GPU acceleration)
