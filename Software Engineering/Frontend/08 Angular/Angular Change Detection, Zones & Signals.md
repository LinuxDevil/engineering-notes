---
title: "Angular Change Detection, Zones & Signals"
tags: [frontend, frontend/angular, frontend/performance]
aliases: [Angular change detection, zone.js, NgZone, Angular signals, zoneless Angular, OnPush]
updated: 2026-09-27
---
# Angular Change Detection, Zones & Signals

<Note>
**TL;DR**

**Change detection (CD)** is how Angular keeps the DOM in sync with component state. For years Angular used **zone.js** to find out *when* something might have changed, then checked the whole component tree (the old `Default` strategy) or only "dirty" branches (`OnPush`). **Signals** tell Angular exactly *what* changed, which enables **zoneless** apps (default for new apps since v21) and makes `OnPush` the default in v22. Knowing both worlds matters, because most production apps are somewhere in between.
</Note>

## 1. What change detection does

Each component's template compiles into instructions with two modes: **create** (build DOM once) and **update** (re-evaluate bindings, patch only what differs). A CD pass walks the component tree **top-down** and runs the update block of each view it visits.

{/* diagram source: /diagrams/diagram-299.mmd */}
![diagram](/diagrams/diagram-299.svg)

In dev mode, Angular runs a second check pass. If a binding's value differs between the two passes you get **`ExpressionChangedAfterItHasBeenCheckedError`**, which usually means a child mutates parent state during CD (for example in `ngAfterViewInit`).

## 2. zone.js and NgZone

A **Zone** is an execution context that persists across async tasks. `zone.js` monkey-patches browser async APIs (`setTimeout`, `Promise`, `addEventListener`, XHR/`fetch`, `requestAnimationFrame`...), so Angular is told whenever an async callback runs.

**NgZone** is Angular's service wrapping a zone named `angular`. When a task (a click handler, a timer callback, an HTTP callback) finishes **and the microtask queue is empty**, NgZone emits `onMicrotaskEmpty`, and Angular calls `ApplicationRef.tick()` to run CD from the root.

{/* diagram source: /diagrams/diagram-300.mmd */}
![diagram](/diagrams/diagram-300.svg)

Pros: "it just works", you can mutate plain fields anywhere. Cons:
- Angular does not know **what** changed, so it checks broadly.
- Every patched event (e.g. `mousemove`, `scroll`) can trigger a full CD pass. Escape hatch: `ngZone.runOutsideAngular(() => ...)` then `ngZone.run()` to re-enter.
- zone.js is extra bytes at startup and makes async stack traces harder to read.
- It cannot patch native `async/await` (Angular's build downlevels it for zone apps).

### "Noop zone" (old way to remove zone.js)
Earlier versions let you remove zone.js by deleting `import 'zone.js'` from `polyfills.ts` and bootstrapping with `platformBrowserDynamic().bootstrapModule(AppModule, { ngZone: 'noop' })`. You then had to trigger CD yourself. **Updated 2026:** use the official zoneless provider instead (section 6).

## 3. Change detection strategies: Eager (Default) vs OnPush

| | `Eager` (called `Default` before v22) | `OnPush` |
|---|---|---|
| Checked when | Every CD pass that reaches it | Only when the component is marked dirty |
| Marked dirty by | n/a | A new **input reference** from the parent's template; an **event** handled in its template (or a child's); `async` pipe emission; a **signal read in its template** changing; manual `markForCheck()` |
| Mutation `this.items.push(x)` in parent | Shows up | **Does not** show up in an OnPush child (same reference) |
| Performance | Checks everything | Skips clean subtrees |

```ts
@Component({
  selector: 'app-fare-card',
  changeDetection: ChangeDetectionStrategy.OnPush,   // default in v22, no need to write it
  template: `{{ fare().price | currency }}`,
})
export class FareCardComponent {
  fare = input.required<Fare>();
}
```

<Info>
**Why OnPush "doesn't update"**

```ts
// Parent
fares: Fare[] = [];
ngOnInit() {
  this.api.load().subscribe(list => this.fares.push(...list));  // mutation: OnPush child with [fares] won't refresh
  this.api.load().subscribe(list => this.fares = list);          // new reference: works
}
```
With signals the problem disappears: `fares = signal<Fare[]>([])` then `this.fares.set(list)` marks every template that reads `fares()`.
</Info>

`ChangeDetectorRef` tools: `markForCheck()` (mark this view and ancestors dirty for the next pass), `detectChanges()` (check this view and its children now, synchronously), `detach()`/`reattach()`.

## 4. Signals

A **signal** is a value wrapper that tracks who reads it. Reading inside a reactive context (a template, `computed`, `effect`) registers a dependency; writing notifies exactly those consumers.

```ts
const price = signal(100);                         // WritableSignal<number>
const qty = signal(2);
const total = computed(() => price() * qty());     // lazy + memoized, read-only
effect(() => console.log('total is', total()));   // side effect, runs after changes settle

price.set(120);
qty.update(q => q + 1);
untracked(() => price());                          // read without subscribing
```

| Primitive | Purpose | Status |
|---|---|---|
| `signal()` | Writable state | Stable since v17 |
| `computed()` | Derived, memoized, lazy | Stable since v17 |
| `effect()` | Side effects (logging, `localStorage`, imperative DOM/3rd-party APIs) | **Stable in v20** |
| `linkedSignal()` | Writable signal that **resets** when a source changes (e.g. selected option resets when the options list changes) | **Stable in v20** |
| `toSignal()` / `toObservable()` | RxJS interop | `toSignal` **stable in v20** |
| `input()`, `model()`, `viewChild()`, `contentChild()` | Signal-based component APIs | Stable (v19) |
| `resource()`, `rxResource()`, `httpResource()` | Async data as signals | Experimental in v20, **stable in v22** |

```ts
// linkedSignal: user can override, but it resets when the source changes
options = input.required<Seat[]>();
selected = linkedSignal(() => this.options()[0]);
```

<Warning>
**effect() pitfalls**

- Don't use `effect` to copy one signal into another; that's what `computed` or `linkedSignal` are for.
- Effects run asynchronously as part of CD, not synchronously on `set()`.
- Signals are **glitch-free**: a `computed` never sees a half-updated state, and consumers are notified once per batch.
</Warning>

### How signals drive change detection
Each component template is a reactive consumer. When a signal it read changes, Angular marks **that** view for refresh and marks its ancestors as "has a child to refresh" without re-checking them. For OnPush components this means a signal change refreshes **only the components that read it**, not the whole path from the root. This is often called local or targeted change detection.

{/* diagram source: /diagrams/diagram-301.mmd */}
![diagram](/diagrams/diagram-301.svg)

## 5. resource and httpResource (async data as signals)

```ts
city = signal('Dubai');

// generic resource: any async loader, re-runs when params change, aborts stale requests
weather = resource({
  params: () => ({ city: this.city() }),
  loader: ({ params, abortSignal }) => fetch(`/api/wx/${params.city}`, { signal: abortSignal }).then(r => r.json()),
});

// HttpClient-based (uses interceptors), stable in v22
wx = httpResource<{ temp: number }>(() => `/api/wx/${this.city()}`);
```

```html
@if (wx.isLoading()) { <app-spinner /> }
@else if (wx.error()) { <p>Could not load weather</p> }
@else { <p>{{ wx.value()?.temp }}°C</p> }
```

A resource exposes `value()`, `status()`, `isLoading()`, `error()`, `hasValue()` and `reload()`. When `city` changes, the previous request is cancelled, which is the signal-world equivalent of RxJS `switchMap`.

<Info>
**🆕 2025–2026**

- v20 (May 28, 2025): `resource`/`httpResource` experimental.
- v22 (June 3, 2026): `resource`, `rxResource` and `httpResource` **stable**, with a new `chain()` API and an SSR cache `id`. v22 also ships an experimental `debounced()` helper.
- v22.2 (Sept 2026): router resources in developer preview.
</Info>

## 6. Zoneless Angular

Without zone.js, Angular schedules CD only when it is **told** something changed:
- a signal read by a template is written
- `markForCheck()` is called (the `async` pipe does this for you)
- an event listener bound in a template fires
- `ComponentRef.setInput()`, attaching/detaching views, and similar framework APIs

```ts
bootstrapApplication(AppComponent, {
  providers: [provideZonelessChangeDetection()],
});
// and remove "zone.js" from the polyfills in angular.json
```

<Warning>
**What breaks when you go zoneless**

- Code that mutates a plain field in `setTimeout`, a manual `subscribe`, or a third-party callback and expects the view to update. Fix: store it in a signal, or call `markForCheck()`.
- Code that relies on `NgZone.onStable`/`onMicrotaskEmpty`. Use `afterNextRender` / `afterEveryRender` instead.
- SSR needs to know when the app is "stable". Angular tracks pending HTTP requests itself; for your own async work use the `PendingTasks` service.
</Warning>

<Info>
**🆕 2025–2026 timeline**

- v18: experimental zoneless.
- v20 (May 2025): developer preview; **stable in v20.2**.
- v21 (Nov 19, 2025): **new apps are zoneless by default**.
- v22 (June 3, 2026): **OnPush is the default** change detection strategy, and `ChangeDetectionStrategy.Default` is renamed **`Eager`**.
</Info>

## 7. What the v22 OnPush default changes

- A component with **no** `changeDetection` setting now behaves as `OnPush`. Components that relied on "Angular checks everything" (mutating arrays, updating fields in subscriptions/timers) will stop refreshing.
- The fix is the same as for zoneless: put state in signals (or use `async` pipe / `markForCheck()`), and pass new references instead of mutating.
- If you truly need the old behaviour, opt in with `ChangeDetectionStrategy.Eager`.
- Combined with zoneless and signals, the mental model becomes: **"the view updates when a signal it reads changes, or when an event in its template fires."** This is close to how Vue, Solid and Svelte 5 work.
- Check the official v22 update guide (`ng update`) for the automated migration before upgrading.

## 8. Angular vs React rendering

| | Angular (zone + Eager) | Angular (signals + OnPush + zoneless) | React |
|---|---|---|---|
| What triggers an update | Any async task finishing | Signal write, template event, `markForCheck` | `setState`/`useReducer` dispatch, context change, parent re-render |
| Unit of work | Check bindings of every view in the tree | Refresh only views that read changed signals | Re-run the component function and its children, diff the virtual DOM |
| Diffing | Compares previous vs new binding values (no VDOM) | Same, but far fewer views | Virtual DOM reconciliation (Fiber) |
| Opt-out of work | `OnPush`, `runOutsideAngular` | Already fine-grained | `memo`, `useMemo`, `useCallback`, or React Compiler (1.0, Oct 2025) |
| Async scheduling | Synchronous tick after the task | Scheduled (microtask/rAF) | Batched, concurrent rendering with priorities (transitions) |
| Mutation of state | Works (Eager), breaks OnPush | Use `set`/`update` | Must be immutable |

Key interview takeaway: React re-runs **components** and diffs output; Angular re-checks **bindings**. Signals move Angular towards fine-grained reactivity (like Solid), while React stays with component re-rendering but automates memoization with the Compiler. See [How React Renders - Reconciliation & Fiber](/Software%20Engineering/Frontend/05%20React/How%20React%20Renders%20-%20Reconciliation%20%26%20Fiber).

## Interview questions

<Accordion title="What is zone.js and why is Angular moving away from it?">
It patches async browser APIs so Angular knows when a task finished and can run `ApplicationRef.tick()`. It can't tell Angular *what* changed, so CD is broad; it costs bundle size and startup time, fires CD for noisy events, and complicates debugging. Signals give precise change notifications, so zoneless is stable since v20.2 and the default for new apps since v21.
</Accordion>

<Accordion title="When is an OnPush component checked?">
When its input references change via the template, when an event fires in its template (or a child's), when the `async` pipe emits, when a signal read in its template changes, or when `markForCheck()` is called. Mutating an object passed as input does not mark it dirty.
</Accordion>

<Accordion title="`markForCheck()` vs `detectChanges()`?">
`markForCheck()` marks the view and its ancestors dirty so the **next** CD pass checks them. `detectChanges()` synchronously checks this view and its children **now**.
</Accordion>

<Accordion title="`computed` vs `effect` vs `linkedSignal`?">
`computed`: read-only derived value, lazy and memoized. `effect`: side effects outside the signal graph (logging, storage, imperative APIs). `linkedSignal`: a writable signal that resets from a source when it changes (e.g. a default selection).
</Accordion>

<Accordion title="What causes `ExpressionChangedAfterItHasBeenCheckedError`?">
In dev mode Angular checks twice. If a binding's value changes between the passes, usually because a child or a lifecycle hook (`ngAfterViewInit`) updates parent state during CD, it throws. Fix by moving the update earlier, deriving with `computed`, or deferring it.
</Accordion>

<Accordion title="How would you migrate a large app to zoneless?">
Adopt `OnPush` everywhere first and fix what breaks; move component state to signals; replace manual subscriptions with `toSignal`/`async`; replace `NgZone.onStable` usage with `afterNextRender`; add `provideZonelessChangeDetection()`, remove `zone.js` from polyfills; test SSR stability with `PendingTasks`.
</Accordion>

<Accordion title="How does Angular's change detection compare with React's?">
See the table above: Angular checks bindings (no VDOM) and with signals refreshes only views that read changed state; React re-runs component functions and reconciles a virtual DOM, optimised by memoization or React Compiler.
</Accordion>

## Related
- [Angular Fundamentals](/Software%20Engineering/Frontend/08%20Angular/Angular%20Fundamentals)
- [Modern Angular v17 to v22](/Software%20Engineering/Frontend/08%20Angular/Modern%20Angular%20v17%20to%20v22)
- [Angular Interview Q&A](/Software%20Engineering/Frontend/08%20Angular/Angular%20Interview%20Q%26A)
- [How React Renders - Reconciliation & Fiber](/Software%20Engineering/Frontend/05%20React/How%20React%20Renders%20-%20Reconciliation%20%26%20Fiber)
- [React Performance](/Software%20Engineering/Frontend/05%20React/React%20Performance)
- [Event Loop - Microtasks & Macrotasks](/Software%20Engineering/Frontend/01%20JavaScript/Event%20Loop%20-%20Microtasks%20%26%20Macrotasks)
- [Other Frameworks - Vue, Svelte, Solid, Astro](/Software%20Engineering/Frontend/09%20Architecture/Other%20Frameworks%20-%20Vue%2C%20Svelte%2C%20Solid%2C%20Astro)

## Sources
- https://indepth.dev/posts/1064/what-every-front-end-developer-should-know-about-change-detection-in-angular-and-react
- https://angular.dev/guide/signals
- https://angular.dev/guide/signals/resource
- https://angular.dev/guide/zoneless
- https://angular.dev/best-practices/skipping-subtrees
- https://angular.dev/roadmap
- https://blog.angular.dev/announcing-angular-v20-b5c9c06cf301
- https://blog.angular.dev/announcing-angular-v21-57946c34f14b
- https://blog.angular.dev/announcing-angular-v22-c52bb83a4664
- https://blog.ninja-squad.com/2026/06/03/what-is-new-angular-22.0
- https://blog.ninja-squad.com/2026/09/23/what-is-new-angular-22.2
