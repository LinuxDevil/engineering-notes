---
title: "Angular Fundamentals"
tags: [frontend, frontend/angular]
aliases: [Angular basics, Angular DI, Angular building blocks]
updated: 2026-09-27
---
# Angular Fundamentals

<Note>
**TL;DR**

An Angular app is a tree of **components** (class + template + styles) wired together by a hierarchical **dependency injection** system. Templates use a small binding language (`{{ }}`, `[prop]`, `(event)`, `[(model)]`) and are compiled **ahead of time** into efficient instructions. Since v19 components are **standalone by default**, so NgModules are optional. Routing, HttpClient, forms, pipes and directives are all first-party.
</Note>

## 1. Building blocks

![Untitled 43](/Attachments/Untitled%2043.png)

| Block | What it is | Modern form (2026) |
|---|---|---|
| **Component** | Class + template + styles that controls a piece of UI | `@Component({ selector, imports, template })`, standalone by default |
| **Template** | HTML plus Angular's binding syntax | `@if`/`@for` control flow, signals read as `count()` |
| **Directive** | Adds behaviour to an existing element | `@Directive`, often with `host: {}` bindings |
| **Pipe** | Pure transform used in templates | `@Pipe({ name })`, standalone |
| **Service** | Shared logic/state, injected via DI | `@Injectable({ providedIn: 'root' })`, or `@Service()` in v22 |
| **Module (NgModule)** | Old unit of compilation and DI configuration | Optional; kept for legacy code and some libraries |
| **Metadata / decorators** | Tells the compiler how to treat a class | `@Component`, `@Input` (now `input()`), `@HostListener` (now `host`) |

## 2. Components and templates

A component is a directive with a template. Only **one** component can match a given element, but many directives can.

```ts
import { Component, signal, computed, input, output } from '@angular/core';

@Component({
  selector: 'app-counter',
  template: `
    <h2>{{ label() }}: {{ count() }}</h2>
    <p>Double: {{ double() }}</p>
    <button (click)="inc()">+1</button>
  `,
})
export class CounterComponent {
  label = input('Count');           // signal input (v17.1+)
  changed = output<number>();       // output() (v17.3+)
  count = signal(0);
  double = computed(() => this.count() * 2);

  inc() {
    this.count.update(c => c + 1);
    this.changed.emit(this.count());
  }
}
```

The template can live inline (`template:`) or in a separate file (`templateUrl: './app.component.html'`). Both are compiled the same way.

### Binding syntax (Angular's template "DSL")

| Syntax | Direction | Example | Notes |
|---|---|---|---|
| `{{ expr }}` | component → view | `{{ user.name }}` | Interpolation, always escaped |
| `[prop]="expr"` | component → view | `[disabled]="saving()"` | Property binding (DOM property or input) |
| `[attr.x]`, `[class.x]`, `[style.x]` | component → view | `[attr.aria-label]="label"` | Attribute/class/style bindings |
| `(event)="handler($event)"` | view → component | `(click)="save()"` | Event binding (DOM event or output) |
| `[(x)]="value"` | two-way | `[(ngModel)]="name"` | "Banana in a box" = `[x]` + `(xChange)` |
| `@if` / `@for` / `@switch` | structure | `@if (user) { ... }` | Built-in control flow (v17+), replaces `*ngIf`/`*ngFor` |
| `*directive` | structure | `*ngIf="user"` | Old structural directive sugar for `<ng-template>` (deprecated for the built-ins in v20) |
| `#ref` | template variable | `<input #box>` | Reference an element or directive in the template |
| `@let x = expr;` | local variable | `@let total = price() * qty();` | Template-local variable (v18.1+) |

<Info>
**Two-way binding by hand (the owner's parent/child update question)**

`@Input()` is one-way. To let the child update the parent, pair it with an output named `<input>Change`, then the parent can use banana-in-a-box.
```ts
// child (classic decorators)
@Input() searchStatus!: boolean;
@Output() searchStatusChange = new EventEmitter<boolean>();
toggle() { this.searchStatusChange.emit(!this.searchStatus); }

// child (modern, v17.2+): model() does both in one line
searchStatus = model(false);
toggle() { this.searchStatus.update(v => !v); }
```
```html
<!-- parent -->
<app-search [(searchStatus)]="isSearching" />
<!-- or, to run a callback on change -->
<app-search [searchStatus]="isSearching" (searchStatusChange)="onChange($event)" />
```
</Info>

### Content projection
`<ng-content>` projects the children a parent passes in. Use `select="[slot=header]"` for named slots. Projected content belongs to the **parent's** view for change detection and DI purposes.

## 3. Standalone components (default since v19)

Before v14, every component had to be declared in an `NgModule`. Standalone components declare their own template dependencies in `imports`, and the app is bootstrapped with a config object instead of an `AppModule`.

```ts
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { appConfig } from './app/app.config';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent, appConfig);

// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withComponentInputBinding()),
    provideHttpClient(withInterceptors([authInterceptor])),
    provideZonelessChangeDetection(),        // default for new apps since v21
  ],
};
```

- Since **v19**, `standalone: true` is the default; you only write `standalone: false` for components still declared in a module.
- `ng generate @angular/core:standalone` migrates an NgModule app step by step.
- NgModules still work. You mostly meet them in older apps and libraries (`importProvidersFrom(SomeModule)` bridges them).

## 4. Dependency injection

**DI** is a design pattern where a class asks for its dependencies instead of creating them. Angular's injector creates instances, caches them per injector, and hands them out.

```ts
@Injectable({ providedIn: 'root' })        // app-wide singleton, tree-shakable
export class FlightsApi {
  private http = inject(HttpClient);      // inject() function (v14+), preferred over constructor params
  search(q: SearchQuery) { return this.http.get<Flight[]>('/api/flights', { params: { ...q } }); }
}
```

### Provider recipes

| Recipe | Example | Use |
|---|---|---|
| `useClass` | `{ provide: Logger, useClass: ConsoleLogger }` | Swap implementations |
| `useValue` | `{ provide: API_URL, useValue: '/api' }` | Config constants |
| `useFactory` | `{ provide: Store, useFactory: () => new Store(inject(API_URL)) }` | Computed or conditional instances |
| `useExisting` | `{ provide: OldLogger, useExisting: Logger }` | Aliases |
| `multi: true` | `{ provide: HTTP_INTERCEPTORS, useClass: X, multi: true }` | Many values under one token |

Use an `InjectionToken<T>` for non-class values: `export const API_URL = new InjectionToken<string>('API_URL');`

### Injector hierarchies

Angular has **two** injector hierarchies that are searched in order:

![injector hierarchies](/Attachments/injector%20hierarchies.png)

1. **Element injectors** (the component/directive tree). Angular creates an `ElementInjector` for every element that hosts a component or directive. You configure it with `providers` / `viewProviders` in `@Component` or `@Directive`. It mirrors the DOM tree, so it forms its own hierarchy.

   ![element injector hieracrhy](/Attachments/element%20injector%20hieracrhy.png)

2. **Environment injectors** (formerly "module injectors"):
   - **Root injector**: services with `providedIn: 'root'`, plus everything in `bootstrapApplication` providers (or eagerly imported NgModules' `providers`). Anything here is available app-wide.
   - **Route/lazy injectors**: a lazily loaded route with `providers: [...]` gets a **child** environment injector, so its services are scoped to that route subtree.
   - **Platform injector**: created by `platformBrowser()` (the old `platformBrowserDynamic()` in `main.ts`); holds platform-wide services such as `DomSanitizer`, shared if you bootstrap several apps on one page.
   - **NullInjector**: the top. It throws `NullInjectorError: No provider for X` unless the lookup is `optional`, in which case you get `null`.

![hierarchy diagram](/Attachments/hierarchy%20diagram.png)

{/* diagram source: /diagrams/diagram-302.mmd */}
![diagram](/diagrams/diagram-302.svg)

**Resolution rules:** start at the requesting element's injector, walk up the element tree, then jump to the environment injector of that component and walk up to `NullInjector`. The first match wins.

| Modifier | `inject(X, {...})` | Effect |
|---|---|---|
| `@Optional()` | `{ optional: true }` | Return `null` instead of throwing |
| `@Self()` | `{ self: true }` | Only look in the current element injector |
| `@SkipSelf()` | `{ skipSelf: true }` | Start from the parent |
| `@Host()` | `{ host: true }` | Stop at the host component's injector |

<Info>
**Scoping a service per component instance**

```ts
@Component({
  selector: 'app-booking-form',
  providers: [BookingFormState],   // new instance for every <app-booking-form>
  template: `<app-passenger-list /> <app-payment />`,
})
export class BookingFormComponent {}
// Children that inject(BookingFormState) share THIS instance,
// a second <app-booking-form> gets its own.
```
</Info>

<Warning>
**Common DI pitfalls**

- Adding a service to a lazy route's `providers` **and** `providedIn: 'root'` creates two instances. Pick one.
- `inject()` only works in an **injection context**: field initializers, constructors, factory functions, or inside `runInInjectionContext()`. Calling it in `ngOnInit` or a `setTimeout` throws.
- `providers` vs `viewProviders`: `viewProviders` are not visible to projected content (`<ng-content>`).
</Warning>

## 5. Services

A service holds logic or state that should not live in a component: API calls, caching, cross-component state, logging. Components stay thin and testable.

```ts
@Injectable({ providedIn: 'root' })
export class CartStore {
  private items = signal<CartItem[]>([]);
  readonly count = computed(() => this.items().length);
  add(item: CartItem) { this.items.update(list => [...list, item]); }
}
```

<Info>
**🆕 2025–2026**

Angular v22 (June 3, 2026) adds `@Service()` as a shorthand for `@Injectable({ providedIn: 'root' })`, and `injectAsync()` for lazy-loading a service's code only when needed.
</Info>

## 6. Sharing data between components

The five classic ways (see the levelup article in Sources):

| # | Relationship | Mechanism |
|---|---|---|
| 1 | Parent → child | `input()` / `@Input()` |
| 2 | Child → parent | `output()` / `@Output()` + `EventEmitter`, or `model()` for two-way |
| 3 | Parent reads child directly | `viewChild()` / `@ViewChild()` |
| 4 | Siblings | Shared parent state, or a shared service |
| 5 | Unrelated components | A service holding a signal or `BehaviorSubject` (or a store like NgRx SignalStore) |

## 7. Routing and lazy loading

```ts
export const routes: Routes = [
  { path: '', component: HomeComponent },
  // lazy standalone component
  { path: 'flights/:id', loadComponent: () => import('./flight-details.component').then(m => m.FlightDetailsComponent) },
  // lazy group of child routes, with its own scoped providers
  {
    path: 'orders',
    canActivate: [authGuard],
    providers: [OrdersStore],
    loadChildren: () => import('./orders/routes').then(m => m.ORDERS_ROUTES),
  },
  { path: '**', redirectTo: '' },
];

export const authGuard: CanActivateFn = () =>
  inject(AuthService).isLoggedIn() || inject(Router).createUrlTree(['/login']);
```

- **Lazy loading** splits each `loadComponent`/`loadChildren` target into its own chunk, downloaded only on navigation. The old form `loadChildren: () => import('./customers/customers.module').then(m => m.CustomersModule)` still works for NgModule apps.
- **Preloading**: `withPreloading(PreloadAllModules)` fetches lazy chunks in the background after the first render.
- **`withComponentInputBinding()`** maps route params, query params and resolver data straight into component inputs.
- **Functional guards and resolvers** (`CanActivateFn`, `ResolveFn`) replaced class-based guards (v15+).
- **RouterState** is a tree of `ActivatedRoute`s. Each node knows its URL segments, params and resolved data. Read it with `inject(Router).routerState` or `inject(ActivatedRoute)`.

### Router events (one navigation)

{/* diagram source: /diagrams/diagram-303.mmd */}
![diagram](/diagrams/diagram-303.svg)

`NavigationEnd`, `NavigationCancel` and `NavigationError` are alternative outcomes, not a sequence. There is also `NavigationSkipped` when the navigation is ignored (e.g. same URL).

## 8. HttpClient and interceptors

`HttpClient` returns **cold Observables**: nothing is sent until you subscribe (or use `async` pipe, `toSignal`, `firstValueFrom`, or `httpResource`).

```ts
provideHttpClient(withFetch(), withInterceptors([authInterceptor, loggingInterceptor]));
```

### Functional interceptors (v15+, the modern way)

```ts
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).token();
  const authReq = token ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } }) : req;
  return next(authReq).pipe(
    catchError((err: HttpErrorResponse) => {
      if (err.status === 401) inject(Router).navigate(['/login']);
      return throwError(() => err);
    }),
  );
};
```

Interceptors run in array order on the request and in reverse order on the response. Requests are immutable, so you always `clone()`.

The class-based form (`implements HttpInterceptor` + `{ provide: HTTP_INTERCEPTORS, useClass, multi: true }`) still works via `withInterceptorsFromDi()`, but functional interceptors are preferred.

<Warning>
**Where to keep the JWT**

The original note suggested storing the token in `sessionStorage` or IndexedDB and attaching it in an interceptor. That works, but anything readable by JS is exposed to XSS. Prefer an **HttpOnly, Secure, SameSite cookie** set by the server (then the interceptor only adds CSRF headers), or keep the access token in memory with a short lifetime. See [Authentication & Token Storage](/Software%20Engineering/Frontend/11%20Security/Authentication%20%26%20Token%20Storage) and [JWT vs Cookies](/Software%20Engineering/APIs%20%26%20Auth/JWT%20vs%20Cookies).
</Warning>

<Info>
**🆕 2025–2026**

- Angular v22 makes **HttpClient use `fetch` by default** (previously you opted in with `withFetch()`).
- `httpResource()` (stable in v22) wraps a request in a signal-based resource. See [Angular Change Detection, Zones & Signals](/Software%20Engineering/Frontend/08%20Angular/Angular%20Change%20Detection%2C%20Zones%20%26%20Signals).
</Info>

## 9. Pipes

A pipe transforms a value in a template: `{{ birthday | date:'fullDate' | uppercase }}` (pipes chain left to right, so this prints `THURSDAY, JUNE 18, 1987`).

Built-ins: `date`, `currency`, `number`/`decimal`, `percent`, `uppercase`/`lowercase`/`titlecase`, `json`, `slice`, `keyvalue`, `async`, `i18nPlural`.

```ts
@Pipe({ name: 'duration' })               // standalone by default
export class DurationPipe implements PipeTransform {
  transform(minutes: number, style: 'short' | 'long' = 'short'): string {
    const h = Math.floor(minutes / 60), m = minutes % 60;
    return style === 'short' ? `${h}h ${m}m` : `${h} hours ${m} minutes`;
  }
}
// {{ flight.durationMin | duration:'long' }}
```

| | Pure pipe (default) | Impure pipe (`pure: false`) |
|---|---|---|
| Runs when | Input value or argument changes (by reference for objects) | Every change detection cycle |
| Cost | Cheap, memoized per binding | Can run on every keystroke or mouse move |
| Example | `date`, `currency` | `async`, `json` |

<Warning>
**Pure pipes and mutation**

`items.push(x)` does not change the array reference, so a pure `filter` pipe will not re-run. Replace the array (`items = [...items, x]`) or use a `computed()` signal instead.
</Warning>

## 10. Directives

Three kinds:
1. **Components**: directives with a template.
2. **Attribute directives**: change the look or behaviour of an element (`ngClass`, `ngStyle`, custom `[appHighlight]`).
3. **Structural directives**: add or remove DOM by manipulating `<ng-template>` (`*ngIf`, `*ngFor`, custom `*appIfRole`). The built-in ones are replaced by `@if`/`@for`/`@switch`.

```ts
@Directive({
  selector: '[appHighlight]',
  host: { '(mouseenter)': 'on.set(true)', '(mouseleave)': 'on.set(false)', '[style.backgroundColor]': 'on() ? color() : null' },
})
export class HighlightDirective {
  color = input('yellow', { alias: 'appHighlight' });
  on = signal(false);
}
// <p appHighlight="lightblue">Highlight me!</p>
```

The original example set `el.nativeElement.style.backgroundColor` directly. It works, but host bindings (or `Renderer2`) are safer for SSR and web workers, where there is no real DOM.

**Host directives** (`hostDirectives: [HighlightDirective]`) let you compose behaviour into a component without inheritance.

| Component | Directive |
|---|---|
| `@Component` | `@Directive` |
| Has a template (required) | No template |
| Builds UI widgets, breaks the app into pieces | Adds behaviour to an existing element or component |
| One per element | Many per element |

**Pipes vs directives:** a pipe manipulates *data* for display; a directive manipulates *the DOM element* it is attached to.

**How does Angular find them?** The compiler matches a component or directive when its **selector** matches an element in a template that has it in scope (via `imports`). It finds a pipe when the pipe's **name** appears after `|` in a template expression.

## 11. Lifecycle hooks

![lifecycle](/Attachments/lifecycle.png)

| Hook | When | Typical use |
|---|---|---|
| `constructor` | Class instantiation (DI happens here) | Only field init and `inject()` |
| `ngOnChanges(changes)` | Before `ngOnInit` and whenever an `@Input` reference changes | React to input changes (with signal inputs use `computed`/`effect` instead) |
| `ngOnInit` | Once, after the first `ngOnChanges` | Start work that needs inputs |
| `ngDoCheck` | Every change detection run | Custom change detection (rarely needed) |
| `ngAfterContentInit` | Once, after projected content is initialised | Read `contentChild` |
| `ngAfterContentChecked` | After every check of projected content | |
| `ngAfterViewInit` | Once, after the component's view and children are initialised | Read `viewChild`, measure DOM |
| `ngAfterViewChecked` | After every check of the view | |
| `ngOnDestroy` | Just before destruction | Cleanup (or use `DestroyRef`) |

Modern helpers:
- `inject(DestroyRef).onDestroy(fn)` and `takeUntilDestroyed()` for cleanup without `ngOnDestroy`.
- `afterNextRender(fn)` / `afterEveryRender(fn)` (formerly `afterRender`) for DOM work that must run in the browser only, after rendering. Safe with SSR.

**Constructor vs `ngOnInit`:** the constructor is a plain TypeScript thing. Angular's injector calls it to create the instance and resolves its parameters as dependencies. Inputs are **not** set yet. `ngOnInit` is called by Angular after the first `ngOnChanges`, when inputs are available, so that is where you start real work such as an HTTP call.

```ts
export class App implements OnInit {
  private myService = inject(MyService);   // runs during construction
  ngOnInit() { /* inputs are ready: fetch data here */ }
}
```

## 12. AOT vs JIT compilation

| | JIT (Just-in-Time) | AOT (Ahead-of-Time) |
|---|---|---|
| When templates compile | In the browser at runtime | At build time |
| Default | Dev builds / `ng serve` up to v8 (prod builds already used AOT) | **Default for all builds since v9** (Ivy) |
| Ships the compiler | Yes (bigger bundle) | No |
| Template errors | At runtime | At build time |
| Security | Templates compiled from strings at runtime | No runtime template evaluation, fewer injection risks |

AOT benefits: faster rendering, no separate requests for external templates/CSS (they are inlined), smaller payload, earlier error detection, better security. The `aot` option in `angular.json` controls it and is `true` by default. Tests (TestBed) historically ran in JIT mode.

With **Ivy** (default since v9, View Engine removed in v13), each component compiles to its own static definition (`ɵcmp`) with template instructions, which enables tree-shaking and "locality" (a component can be compiled from its own file). The v22 default of `strictTemplates: true` means the compiler type-checks every template binding.

## 13. Forms overview

| Approach | How | Best for |
|---|---|---|
| **Template-driven** | `FormsModule`, `[(ngModel)]` | Simple forms |
| **Reactive** | `FormGroup`/`FormControl`/`FormBuilder`, strictly typed since v14 | Complex, dynamic forms, validation logic in code |
| **Signal Forms** | `form(model, schema)` from `@angular/forms/signals` | New code on v22+ (stable in v22) |

```ts
// Reactive (typed)
form = inject(NonNullableFormBuilder).group({
  email: ['', [Validators.required, Validators.email]],
  passengers: [1, [Validators.min(1), Validators.max(9)]],
});

// Signal Forms (experimental in v21, stable in v22)
readonly model = signal({ email: '', amount: 0 });
readonly f = form(this.model, s => { required(s.email); });
```

## 14. RxJS essentials

RxJS is a library for composing async and event-based code with **Observables**. `HttpClient`, the router (`events`, `paramMap`) and reactive forms (`valueChanges`) expose Observables.

| Observable | Promise |
|---|---|
| Lazy: nothing runs until `subscribe()` | Eager: runs on creation |
| 0..n values over time | Exactly one value (or error) |
| Cancellable (`unsubscribe()`) | Not cancellable (use `AbortController`) |
| Operators (`map`, `switchMap`, `retry`...) via `pipe()` | `.then()` chaining |
| Errors handled in `subscribe`/`catchError` | `.catch()`, errors propagate down the chain |

**Flattening operators** (the most asked RxJS question):

| Operator | Behaviour | Use for |
|---|---|---|
| `switchMap` | Cancel the previous inner, keep the latest | Typeahead search, route param → fetch |
| `mergeMap` | Run all in parallel | Independent writes |
| `concatMap` | Queue, one at a time in order | Ordered saves |
| `exhaustMap` | Ignore new while one runs | "Submit" button, login |

**Multicasting** means one execution shared by many subscribers. Use a `Subject`/`BehaviorSubject`/`ReplaySubject`, or the `share()` / `shareReplay({ bufferSize: 1, refCount: true })` operators. The old `multicast()` operator from the original note is deprecated in RxJS 7.

```ts
const search$ = this.query.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(q => this.api.search(q).pipe(catchError(() => of([])))),
  shareReplay({ bufferSize: 1, refCount: true }),
);
results = toSignal(search$, { initialValue: [] });   // RxJS → signal interop
```

<Warning>
**Subscription leaks**

Manual `subscribe()` in a component must be cleaned up. Prefer the `async` pipe, `toSignal()`, or `.pipe(takeUntilDestroyed())`. `HttpClient` completes after one response, but `valueChanges`, `interval` and router events do not.
</Warning>

## 15. Security basics

Angular treats every value as **untrusted** and sanitizes/escapes it when binding into the DOM (interpolation, property, attribute, style, class). Binding `[innerHTML]` to `'Template <script>alert("0wned")</script> <b>Syntax</b>'` drops the `<script>` (keeping safe content) and logs a warning. `<script>` tags written in a template are ignored. `DomSanitizer.bypassSecurityTrust*` opts out, so treat it as a code-review red flag. See [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types).

## Interview questions

<Accordion title="Explain Angular's injector hierarchy and how a dependency is resolved.">
Two trees: element injectors (from `providers` on components/directives, mirroring the DOM) and environment injectors (root, lazy-route children, platform, then NullInjector). Lookup starts at the requesting element, walks up the element tree, then the environment chain. First match wins; nothing found → `NullInjectorError` unless `optional: true`. `self`, `skipSelf` and `host` narrow the search.
</Accordion>

<Accordion title="What does `providedIn: 'root'` give you over listing a service in `providers`?">
An app-wide singleton that is **tree-shakable**: if nothing injects it, it is dropped from the bundle. Listing it in a component's `providers` instead gives one instance per component instance, and in a lazy route's `providers` gives one per route injector.
</Accordion>

<Accordion title="Why use `inject()` instead of constructor injection?">
Works in functions (guards, interceptors, resolvers, factory providers), plays well with inheritance (no `super(...)` chains), avoids `useDefineForClassFields` ordering issues, and keeps types inferred. It must run in an injection context.
</Accordion>

<Accordion title="How do functional interceptors work and in what order do they run?">
An interceptor is `(req, next) => Observable<HttpEvent>`. Register with `provideHttpClient(withInterceptors([a, b]))`. Requests pass `a` then `b`; responses come back through `b` then `a`. Clone the request to modify it.
</Accordion>

<Accordion title="Pure vs impure pipes?">
Pure pipes run only when the input reference or an argument changes, so they are memoized and cheap. Impure pipes run every change detection cycle (e.g. `async`). Mutating an array does not re-run a pure pipe.
</Accordion>

<Accordion title="`switchMap` vs `mergeMap` vs `concatMap` vs `exhaustMap`?">
Cancel previous / run all in parallel / queue in order / ignore new while busy. Typeahead → `switchMap`; submit button → `exhaustMap`.
</Accordion>

<Accordion title="What is the difference between `providers` and `viewProviders`?">
Both create element-injector providers, but `viewProviders` are visible only to the component's own view, not to content projected into it via `<ng-content>`.
</Accordion>

## Related
- [Angular Index](/Software%20Engineering/Frontend/08%20Angular/Angular%20Index)
- [Angular Change Detection, Zones & Signals](/Software%20Engineering/Frontend/08%20Angular/Angular%20Change%20Detection%2C%20Zones%20%26%20Signals)
- [Modern Angular v17 to v22](/Software%20Engineering/Frontend/08%20Angular/Modern%20Angular%20v17%20to%20v22)
- [Angular Interview Q&A](/Software%20Engineering/Frontend/08%20Angular/Angular%20Interview%20Q%26A)
- [Frontend Design Patterns](/Software%20Engineering/Frontend/09%20Architecture/Frontend%20Design%20Patterns) (DI, observer, decorator)
- [Web Components & Shadow DOM](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Web%20Components%20%26%20Shadow%20DOM) (Angular Elements)
- [Promises, Async-Await & Cancellation](/Software%20Engineering/Frontend/01%20JavaScript/Promises%2C%20Async-Await%20%26%20Cancellation)

## Sources
- https://angular.dev/guide/components
- https://angular.dev/guide/di/hierarchical-dependency-injection
- https://angular.dev/guide/routing
- https://angular.dev/guide/http/interceptors
- https://angular.dev/guide/templates/pipes
- https://angular.dev/tools/cli/aot-compiler
- https://rxjs.dev/guide/overview
- https://levelup.gitconnected.com/5-ways-to-share-data-between-angular-components-d656a7eb7f96
- https://stackoverflow.com/questions/41464871/update-parent-component-property-from-child-component-in-angular-2
- https://www.freecodecamp.org/news/angular-directives-learn-how-to-use-or-create-custom-directives-in-angular-c9b133c24442/
- https://codecraft.tv/courses/angular/custom-directives/creating-a-custom-directive/
- https://github.com/sudheerj/angular-interview-questions
- https://blog.angular.dev/announcing-angular-v22-c52bb83a4664
