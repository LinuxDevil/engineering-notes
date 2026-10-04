---
title: "Interview Questions Angular"
tags: [interviews, interviews/angular, interviews/dotnet]
aliases: [Angular Interview Questions (original), Angular and .NET Interview Questions]
updated: 2026-09-27
---
# Interview Questions Angular

<Note>
**TL;DR**

The questions from a real full-stack interview: Angular (pipes, directives, lazy loading, HTTP, component communication, libraries, JWT) and ASP.NET Core / C# (authorization policies, EF Core code first, middleware, DI lifetimes, exception filters, JWT). Answers are corrected and updated for modern Angular (standalone components, signals, v17+ control flow) and .NET 8+. The complete Angular answer set lives in [Angular Interview Q&A](/Software%20Engineering/Frontend/08%20Angular/Angular%20Interview%20Q%26A).
</Note>

<Info>
**🆕 2025–2026 context**

Modern Angular answers should mention **standalone components** (default since v19, Nov 2024), **signals** and the signal-based `input()`, `output()` and `model()` APIs, the built-in control flow (`@if`, `@for`), and zoneless change detection (stable in v20.2, default for new apps in v21). Details and dates: [Modern Angular v17 to v22](/Software%20Engineering/Frontend/08%20Angular/Modern%20Angular%20v17%20to%20v22).
</Info>

## Angular

### Pipes
<Accordion title="What are pipes and how do you create a custom pipe?">
A pipe transforms a value inside a template expression: `{{ birthday | date }}`. Built-in pipes include `date`, `currency`, `async`, `json`, `titlecase`.
```ts
@Component({
  selector: 'app-birthday',
  template: `<p>Birthday is {{ birthday | date }}</p>`,
})
export class BirthdayComponent {
  birthday = new Date(1987, 5, 18); // months are 0-based: 5 = June
}
```
**(fixed)** The original used `new Date(1987, 6, 18)` with the comment "June 18"; month 6 is **July**.

A custom pipe is a class with `@Pipe` that implements `PipeTransform.transform(value, ...args)`. Pipes are standalone by default in modern Angular, so you add them to a component's `imports`.
```ts
@Pipe({ name: 'truncate' })
export class TruncatePipe implements PipeTransform {
  transform(value: string, max = 20): string {
    return value.length > max ? value.slice(0, max) + '…' : value;
  }
}
// template: {{ title | truncate: 40 }}
```
</Accordion>

<Accordion title="Pure vs impure pipes?">
A **pure** pipe (default) runs only when the input value (primitive) or reference (object/array) changes, so Angular can cache the result. An **impure** pipe (`pure: false`) runs on **every** change detection cycle, which can be every keystroke, so keep it cheap. Mutating an array in place does not re-run a pure pipe; create a new array instead. `async` is an impure built-in pipe.
</Accordion>

<Accordion title="What is the difference between pipes and directives?">
A pipe transforms **data** for display. A directive changes the **DOM element or component** it is attached to (behaviour, attributes, structure).
</Accordion>

### Directives and components
<Accordion title="What are directives?">
Classes that add behaviour to elements. Three kinds: **components** (directives with a template), **attribute directives** (change look/behaviour: `ngClass`, a custom highlight), and **structural directives** (change DOM structure: `*ngIf`, `*ngFor`, now mostly replaced by `@if`/`@for` blocks).
</Accordion>

<Accordion title="How do you create a custom directive?">
```ts
@Directive({ selector: '[appHighlight]' })
export class HighlightDirective {
  color = input('yellow');                         // signal input
  private el = inject(ElementRef);
  @HostListener('mouseenter') on()  { this.el.nativeElement.style.background = this.color(); }
  @HostListener('mouseleave') off() { this.el.nativeElement.style.background = ''; }
}
// <p appHighlight color="lightblue">Hover me</p>
```
Prefer `host: { '(mouseenter)': 'on()' }` metadata or `Renderer2` over touching `nativeElement` directly if you need SSR safety.
</Accordion>

<Accordion title="Component vs directive?">
A component is a directive **with a template**. Only one component can match an element, but many attribute directives can. Components build UI; directives add behaviour to existing elements or components.
**(fixed)** The original table mentioned an `@View` decorator; it was removed before Angular 2 final. The line "directives are used to design reusable components" is also misleading: both are reusable.
</Accordion>

<Accordion title="How does Angular find components, directives and pipes?">
At compile time the compiler matches element and attribute **selectors** in the template against the components/directives in the component's compilation scope (its `imports` for standalone components, or the declaring NgModule). Pipes are matched by **name** in pipe syntax. Unknown elements cause a compile error.
</Accordion>

### Routing and lazy loading
<Accordion title="How does lazy loading work in Angular?">
The router loads a route's code only when it is visited, so the initial bundle is smaller. Modern (standalone) form:
```ts
export const routes: Routes = [
  { path: 'customers', loadComponent: () => import('./customers/customers.component').then(m => m.CustomersComponent) },
  { path: 'orders', loadChildren: () => import('./orders/orders.routes').then(m => m.ORDERS_ROUTES) },
  { path: '', redirectTo: 'customers', pathMatch: 'full' },
];
```
**(fixed)** The original redirected `''` to `''`, which does nothing useful; redirect to a real route. `loadChildren` with NgModules still works. Inside templates, `@defer` lazy-loads parts of a page.
</Accordion>

### HTTP and auth
<Accordion title="How do you make HTTP requests and handle responses?">
Inject `HttpClient` (provided with `provideHttpClient()`), call `get/post/...` to get a cold Observable, and subscribe (or use the `async` pipe, `toSignal`, or `httpResource` in newer versions). Handle errors with `catchError`, retries with `retry`.
</Accordion>

<Accordion title="How do you handle a JWT in Angular?">
Attach it in a functional **interceptor**:
```ts
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).accessToken();
  return next(token ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } }) : req);
};
// provideHttpClient(withInterceptors([authInterceptor]))
```
**(fixed / nuance)** The original said to store the JWT in `sessionStorage` or IndexedDB. Any storage readable by JS can be stolen by XSS. Better: keep the access token **in memory** with a short lifetime and keep the refresh token in an `HttpOnly; Secure; SameSite` cookie, or use a BFF so the browser only has a session cookie. See [Authentication & Token Storage](/Software%20Engineering/Frontend/11%20Security/Authentication%20%26%20Token%20Storage).
</Accordion>

### Component communication
<Accordion title="How do you share data between components?">
1. Parent → child: inputs (`input()` signal or `@Input`).
2. Child → parent: outputs (`output()` or `@Output` + `EventEmitter`).
3. Two-way: `model()` signal or the `[(x)]` "banana in a box" pattern.
4. Parent reads child: `viewChild()` / `@ViewChild`.
5. Siblings or unrelated components: a shared injectable service holding a signal or `BehaviorSubject`, or a store (NgRx SignalStore).
6. Via the router: route params, query params, resolvers.
</Accordion>

<Accordion title="How does a child update data that came from the parent?">
Inputs are one-way. For two-way binding, pair an input `x` with an output `xChange`, and bind with `[(x)]` in the parent. Modern version:
```ts
// child
searchStatus = model<boolean>(false);
toggle() { this.searchStatus.update(v => !v); }  // parent is notified automatically
```
```html
<!-- parent -->
<app-search [(searchStatus)]="isSearching" />
```
Classic version: `@Input() getSearchStatus` + `@Output() getSearchStatusChange = new EventEmitter<boolean>()`, emit with `this.getSearchStatusChange.emit(newValue)`, bind with `[(getSearchStatus)]="myBoundProperty"`.
</Accordion>

### Libraries and reuse
<Accordion title="What is an Angular library?">
A project that cannot run on its own and is imported by applications (`ng generate library`), built with ng-packagr into the Angular Package Format and published to npm or used in a monorepo. Example: `@angular/service-worker` adds PWA support.
</Accordion>

<Accordion title="How do you reuse components?">
Within one app: standalone components imported where needed (formerly a shared NgModule). Across apps: an internal component library (in a monorepo with Nx or published to a registry), documented and tested visually with Storybook. See [Design Systems](/Software%20Engineering/Frontend/09%20Architecture/Design%20Systems) and [Monorepos](/Software%20Engineering/Frontend/14%20Tooling/Monorepos).
</Accordion>

<Accordion title="Can libraries include CSS?">
Yes: component styles are bundled with components, and global styles/theme files can be shipped as assets (declared in `ng-package.json`) or Sass partials consumers import.
</Accordion>

## ASP.NET Core / C#
The original notes listed these topics without answers; answers added below.

<Accordion title="Authorization policies vs roles (and &quot;rules&quot;)?">
**Roles** are simple group checks (`[Authorize(Roles = "Admin")]`). **Policies** are named sets of **requirements** evaluated by handlers, so they can express any rule (claims, age, ownership of a resource):
```csharp
builder.Services.AddAuthorization(o =>
    o.AddPolicy("CanEditInvoices", p => p.RequireClaim("permission", "invoices:edit")));
[Authorize(Policy = "CanEditInvoices")]
```
Use roles for coarse access and policies (plus resource-based authorization with `IAuthorizationService`) for fine-grained or data-dependent checks. See [API Security](/Software%20Engineering/Security/API%20Security).
</Accordion>

<Accordion title="What is the EF Core code-first approach?">
You write C# entity classes and a `DbContext`; EF Core generates the schema through **migrations** (`dotnet ef migrations add Init`, `dotnet ef database update`). Configure the model in `OnModelCreating` (keys, indexes, relationships) or with attributes. Database-first is the opposite: scaffold classes from an existing schema.
</Accordion>

<Accordion title="How do you set `CreatedAt` / `UpdatedAt` in one place?">
Override `SaveChanges`/`SaveChangesAsync` (or use a `SaveChangesInterceptor`) and set timestamps for tracked entities:
```csharp
public override Task<int> SaveChangesAsync(CancellationToken ct = default) {
    var now = DateTime.UtcNow;
    foreach (var e in ChangeTracker.Entries<IAuditable>()) {
        if (e.State == EntityState.Added) e.Entity.CreatedAt = now;
        if (e.State is EntityState.Added or EntityState.Modified) e.Entity.UpdatedAt = now;
    }
    return base.SaveChangesAsync(ct);
}
```
</Accordion>

<Accordion title="What is middleware in ASP.NET Core?">
Components in the request pipeline; each receives `HttpContext`, may do work, and calls `next()` or short-circuits. Order matters: exception handling → HTTPS redirection → static files → routing → CORS → authentication → authorization → endpoints.
```csharp
app.Use(async (ctx, next) => {
    var sw = Stopwatch.StartNew();
    await next();
    logger.LogInformation("{Path} took {Ms} ms", ctx.Request.Path, sw.ElapsedMilliseconds);
});
```
</Accordion>

<Accordion title="Singleton vs scoped vs transient services?">
- **Singleton**: one instance for the app lifetime (must be thread-safe; caches, config).
- **Scoped**: one per request (the `DbContext` is scoped).
- **Transient**: a new instance every time it is resolved (lightweight stateless services).
Pitfall: injecting a scoped service into a singleton ("captive dependency") keeps one request's `DbContext` forever; the default service provider throws for this in Development.
</Accordion>

<Accordion title="What are exception filters? What should you use today?">
MVC filters (`IExceptionFilter`) that handle exceptions thrown by controller actions. For the whole app, prefer the exception-handling middleware with `IExceptionHandler` (.NET 8+) and `AddProblemDetails()`, which returns RFC 9457 Problem Details responses and does not leak stack traces.
</Accordion>

<Accordion title="How is JWT authentication set up in ASP.NET Core?">
`AddAuthentication().AddJwtBearer(o => { o.Authority = "..."; o.Audience = "..."; })`, then `UseAuthentication()` and `UseAuthorization()`. Always validate issuer, audience, lifetime and signature; keep tokens short-lived. Prefer an identity provider over hand-rolled token issuing. See [OAuth 2.x & OpenID Connect](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect).
</Accordion>

## Related
- [Interviews Index](/Software%20Engineering/Interviews/Interviews%20Index)
- [Angular Interview Q&A](/Software%20Engineering/Frontend/08%20Angular/Angular%20Interview%20Q%26A) (full answer set) · [Angular Index](/Software%20Engineering/Frontend/08%20Angular/Angular%20Index) · [Modern Angular v17 to v22](/Software%20Engineering/Frontend/08%20Angular/Modern%20Angular%20v17%20to%20v22) · [Angular Change Detection, Zones & Signals](/Software%20Engineering/Frontend/08%20Angular/Angular%20Change%20Detection%2C%20Zones%20%26%20Signals)
- [Authentication & Token Storage](/Software%20Engineering/Frontend/11%20Security/Authentication%20%26%20Token%20Storage) · [API Security](/Software%20Engineering/Security/API%20Security) · [Backend Interview Questions](/Software%20Engineering/Interviews/Backend%20Interview%20Questions)

## Sources
- [angular.dev: Pipes](https://angular.dev/guide/templates/pipes), [Directives](https://angular.dev/guide/directives), [Lazy loading routes](https://angular.dev/guide/routing/define-routes), [Signal inputs](https://angular.dev/guide/components/inputs), [Model inputs](https://angular.dev/guide/components/inputs#model-inputs)
- [sudheerj/angular-interview-questions](https://github.com/sudheerj/angular-interview-questions)
- [Microsoft Learn: Policy-based authorization in ASP.NET Core](https://learn.microsoft.com/aspnet/core/security/authorization/policies)
- [Microsoft Learn: ASP.NET Core middleware](https://learn.microsoft.com/aspnet/core/fundamentals/middleware/)
- [Microsoft Learn: Dependency injection service lifetimes](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection#service-lifetimes)
- [Microsoft Learn: Handle errors in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/error-handling)
- [Microsoft Learn: EF Core migrations](https://learn.microsoft.com/ef/core/managing-schemas/migrations/)
