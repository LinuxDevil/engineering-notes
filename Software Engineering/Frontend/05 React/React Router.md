---
title: "React Router"
tags: [frontend, frontend/react, frontend/routing]
aliases: [React Router, RR, react-router-dom, Remix, React Router v7, React Router v8]
updated: 2026-09-27
---
# React Router

<Note>
**TL;DR**

React Router maps URLs to components. **v6** gave us `<Routes>/<Route element>`, nested routes with `<Outlet>`, and hooks (`useParams`, `useSearchParams`, `useNavigate`). **v6.4** added **data routers**: route `loader`s and `action`s that fetch and mutate *before* rendering. **v7 (late 2024) merged Remix into React Router**, so it now has three modes: declarative, data and **framework** (Vite plugin, SSR, type-safe route modules). **v8 (2026-06-17)** is ESM-only, needs React 19.2.7+, makes middleware the default and drops the `react-router-dom` package.
</Note>

## 1. The routers (where the location lives)

| Router | Keeps location in | Use for |
|---|---|---|
| `BrowserRouter` / `createBrowserRouter` | The browser URL + **History API** (`pushState`) | Normal web apps |
| `HashRouter` / `createHashRouter` | `#/path` in the URL | Static hosting that can't rewrite URLs (avoid if you can) |
| `MemoryRouter` / `createMemoryRouter` | An in-memory history stack | **Tests**, Storybook, embedded widgets, non-browser environments |
| `StaticRouter` / `createStaticHandler` | A fixed location | **Server rendering** a single URL to HTML, where the location never changes |

- **`BrowserRouter`** uses the **history** library (inlined into React Router since 6.4) together with **React Context**.
  - The history layer tracks navigation with the **browser's built-in history stack**.
  - React Context makes the router state **available wherever** React Router needs it.
- **`MemoryRouter`** keeps the history in memory instead of the URL.
- **`StaticRouter`** is for environments where the app's location never changes, such as rendering one route to static HTML on the server.

<Warning>
**Correction**

The old note said to use `MemoryRouter` "if you're developing a React Native application". React Native used a separate `NativeRouter` from `react-router-native` (v6). Most RN apps now use React Navigation or Expo Router. `MemoryRouter` is mainly for **tests and non-URL environments**.
</Warning>

## 2. Declarative routing (v6 style, still supported)
- **`<Route>`** maps a location to a component (`element`).
- **`<Routes>`** is the conductor. It looks at all its child `<Route>`s and **picks the best match** by ranking. In v6+ the **order doesn't matter**, and there's no `exact` or `Switch` any more.

```jsx
import { BrowserRouter, Routes, Route, Link } from 'react-router'; // v7+: import from 'react-router'

function App() {
  return (
    <BrowserRouter>
      <nav><Link to="/">Home</Link> <Link to="/messages">Messages</Link></nav>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/messages/*" element={<Messages />} />
        <Route path="/settings" element={<Settings />} />
        <Route path="*" element={<NotFound />} />   {/* 404: matches anything not matched above */}
      </Routes>
    </BrowserRouter>
  );
}
```

### Catching 404s
`path="*"` is the lowest-ranked match, so **where you put it doesn't matter**:
```jsx
<Routes>
  <Route path="*" element={<NotFound />} />
  <Route path="/" element={<Home />} />
  <Route path="/about" element={<About />} />
  <Route path="/settings" element={<Settings />} />
</Routes>
```

## 3. Nested routes and `<Outlet>`
Nested routes let a parent route **act as a wrapper** (a layout) and control where the child route renders.

**Option A: descendant `<Routes>` with `/*`.** Adding `/*` to `/messages` tells React Router that `Messages` has its own nested `<Routes>`. The parent path then matches `/messages` **and** anything under `/messages/*`.

```jsx
// App.js
<Routes>
  <Route path="/" element={<Home />} />
  <Route path="/messages/*" element={<Messages />} />
  <Route path="/settings" element={<Settings />} />
</Routes>

// Messages.js
function Messages() {
  return (
    <Container>
      <Conversations />
      <Routes>
        <Route path=":id" element={<Chat />} />   {/* relative: /messages/:id */}
      </Routes>
    </Container>
  );
}
```

**Option B: nested `<Route>` config + `<Outlet>`.** This is preferred, and it's required for data routers.
```jsx
// App.js
<Route path="/messages" element={<Messages />}>
  <Route index element={<PickAConversation />} />   {/* /messages */}
  <Route path=":id" element={<Chat />} />           {/* /messages/42 */}
</Route>

// Messages.js
import { Outlet } from 'react-router';
function Messages() {
  return (
    <Container>
      <Conversations />
      <Outlet />                                     {/* the matched child renders here */}
    </Container>
  );
}
```
Layout routes (a `<Route element>` with no `path`) wrap children without adding URL segments. Use them for auth guards and shared chrome.

{/* diagram source: /diagrams/diagram-266.mmd */}
![diagram](/diagrams/diagram-266.svg)

## 4. Hooks for reading the URL

| Hook | Returns |
|---|---|
| `useParams()` | An object mapping **URL params to values**: `/messages/:id` → `{ id: '42' }` |
| `useSearchParams()` | `[searchParams, setSearchParams]` for the **query string** (`URLSearchParams`) |
| `useLocation()` | `{ pathname, search, hash, state, key }` |
| `useNavigate()` | The imperative `navigate(to, { replace, state })` function |
| `useMatch(pattern)` / `useMatches()` | Match info (breadcrumbs, handles) |

```jsx
function FlightResults() {
  const { from, to } = useParams();                       // /flights/:from/:to
  const [searchParams, setSearchParams] = useSearchParams();
  const cabin = searchParams.get('cabin') ?? 'economy';   // ?cabin=business&sort=price
  return (
    <select value={cabin} onChange={e => setSearchParams(p => { p.set('cabin', e.target.value); return p; })}>
      <option>economy</option><option>business</option>
    </select>
  );
}
```
Keep filters, sorting and pagination in the URL. It makes them shareable, lets you bookmark them, and the back button works. See [URL as State](/Software%20Engineering/Frontend/06%20State%20Management/URL%20as%20State).

### Passing state through navigation
Data passed in a link's **`state`** prop shows up on **`location.state`**, which you read with **`useLocation`**. It isn't in the URL: it lives in the history entry, and it's lost if the user opens the link in a new tab.
```jsx
<Link to="/checkout" state={{ from: 'search', flightId }}>Book</Link>
// in Checkout
const { state } = useLocation(); // { from: 'search', flightId }
```

## 5. Navigating programmatically: `navigate` vs `<Navigate>`
React Router offers two ways to navigate from code:
- **Imperative:** `useNavigate()` returns `navigate()`. Use it in event handlers and after async work.
- **Declarative:** `<Navigate to>` navigates when it renders. Use it for redirects in JSX.

```jsx
const navigate = useNavigate();
async function onSubmit(data) { await save(data); navigate('/trips', { replace: true }); }
navigate(-1); // back

function RequireAuth({ children }) {
  const user = useUser();
  const location = useLocation();
  return user ? children : <Navigate to="/login" replace state={{ from: location }} />;
}
```
In data routers, prefer `redirect()` from a loader or action. It happens before render, so there's no flash of the protected page.

## 6. Data routers (v6.4+): loaders and actions
Fetching in effects gives you **waterfalls**: the parent renders, then fetches, then the child renders, then fetches. Data routers **start every matched route's loader in parallel as soon as navigation starts**.

{/* diagram source: /diagrams/diagram-267.mmd */}
![diagram](/diagrams/diagram-267.svg)

```jsx
import { createBrowserRouter, RouterProvider, useLoaderData, Form, useNavigation, redirect } from 'react-router';

const router = createBrowserRouter([
  {
    path: '/', element: <Root />, errorElement: <RootError />,
    children: [
      {
        path: 'trips/:id',
        loader: async ({ params, request }) => {
          const res = await fetch(`/api/trips/${params.id}`, { signal: request.signal });
          if (res.status === 404) throw new Response('Not found', { status: 404 });
          return res.json();
        },
        action: async ({ request, params }) => {
          const fd = await request.formData();
          await fetch(`/api/trips/${params.id}`, { method: 'PATCH', body: fd });
          return redirect(`/trips/${params.id}`);
        },
        element: <Trip />,
        // code-split alternative: lazy: () => import('./trip-route') returning { loader, action, Component }
      },
    ],
  },
]);

function Trip() {
  const trip = useLoaderData();
  const navigation = useNavigation();              // 'idle' | 'loading' | 'submitting'
  return (
    <Form method="post">                           {/* submits to this route's action, then revalidates loaders */}
      <input name="notes" defaultValue={trip.notes} />
      <button disabled={navigation.state === 'submitting'}>Save</button>
    </Form>
  );
}

createRoot(el).render(<RouterProvider router={router} />);
```
Other data APIs:
- **`useFetcher`**: mutations and loads that **don't navigate** (like buttons, combobox search).
- **`errorElement`/`ErrorBoundary`** per route, with `useRouteError`.
- **`defer`/streaming:** return promises and use `<Await>` or `use()`.
- **`shouldRevalidate`**.

## 7. v7: Remix merged in, three modes
**React Router v7** (released late 2024) is the next version of **Remix v2** too. The features of the two projects merged, and you import everything from `react-router`. `react-router-dom` was only a re-export in v7.

| Mode | Setup | You get |
|---|---|---|
| **Declarative** | `<BrowserRouter>` + `<Routes>` | v6-style routing only |
| **Data** | `createBrowserRouter` + `RouterProvider` | Loaders, actions, fetchers, pending UI |
| **Framework** | `@react-router/dev` Vite plugin + `routes.ts` | Everything above plus SSR/SSG/SPA modes, type-safe route modules (generated `Route.*` types), code splitting per route, `clientLoader`/`clientAction`, `meta`/`links` |

<Info>
**Framework mode**

```ts
// app/routes.ts
import { type RouteConfig, route, index, layout } from '@react-router/dev/routes';
export default [
  index('routes/home.tsx'),
  layout('routes/app-layout.tsx', [
    route('trips/:id', 'routes/trip.tsx'),
  ]),
] satisfies RouteConfig;
```
```tsx
// app/routes/trip.tsx
import type { Route } from './+types/trip';          // generated types

export async function loader({ params }: Route.LoaderArgs) {   // runs on the server (SSR) or at build (prerender)
  return getTrip(params.id);
}
export async function action({ request }: Route.ActionArgs) {
  const fd = await request.formData(); await updateTrip(fd); return { ok: true };
}
export function meta({ data }: Route.MetaArgs) { return [{ title: `Trip to ${data.city}` }]; }

export default function Trip({ loaderData }: Route.ComponentProps) {
  return <h1>{loaderData.city}</h1>;                  // fully typed from loader's return
}
```
</Info>

<Info>
**🆕 2025–2026**

- **React Router v8 (2026-06-17):**
  - **ESM-only.** Needs **React 19.2.7+, Node 22.22+ and Vite 7+**.
  - **Middleware** and the **Vite Environment API** are now defaults.
  - The **`react-router-dom` package was removed**, so import from `react-router`.
  - **RSC support is still unstable.**
  - Majors now ship **yearly**. [blog](https://remix.run/blog/react-router-v8)
- **Remix 3** is now a **separate, non-React** full-stack framework (RC 2026-08-31, GA planned for 2026-10-02). Don't confuse it with "Remix = React Router framework mode". [blog](https://remix.run/blog/remix-3-release-candidate)
- **Security:** the Dec 2025 RSC RCE advisory (CVE-2025-55182) also affected React Router's RSC support. See [React Server Components & Server Functions](/Software%20Engineering/Frontend/05%20React/React%20Server%20Components%20%26%20Server%20Functions).
- **Platform:** the **Navigation API** is Baseline Newly available (2026-01-13), and **URLPattern** is Newly available (2025-09-15). See [Modern Web Platform APIs 2025-2026](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Modern%20Web%20Platform%20APIs%202025-2026).
</Info>

**Upgrade path:**
1. v6 → v7: turn on the v6 `future` flags first, then change imports to `react-router`.
2. v7 → v8: adopt the v8 future flags (middleware), move to ESM/Vite 7, and remove `react-router-dom`.

**Alternatives:**
- **TanStack Router**: fully type-safe, search-param state.
- **Next.js App Router**: file-system routing with RSC. See [Next.js App Router Fundamentals](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20App%20Router%20Fundamentals).

## Interview questions
<Accordion title="`BrowserRouter` vs `HashRouter` vs `MemoryRouter` vs `StaticRouter`?">
Browser uses real URLs through the History API (needs a server fallback to `index.html`). Hash stores the path after `#` (no server config needed). Memory keeps history in memory (tests, embedded). Static renders a fixed location on the server.
</Accordion>

<Accordion title="How do nested routes work?">
Child `<Route>`s are nested inside a parent route. The parent's component renders `<Outlet />` where the matched child should appear. Index routes render at the parent's exact path, and layout routes wrap children without a path segment.
</Accordion>

<Accordion title="`useNavigate` vs `<Navigate>`?">
`useNavigate` returns an imperative function for event handlers and after async work. `<Navigate>` is a component that redirects when rendered (declarative guards). In data routers, prefer `redirect()` inside loaders and actions.
</Accordion>

<Accordion title="What problem do loaders solve?">
They move data fetching out of component effects, so it starts in parallel for all matched routes when navigation begins. That avoids render-then-fetch waterfalls, gives consistent pending/error UI, and lets actions automatically revalidate the data after mutations.
</Accordion>

<Accordion title="What changed in React Router v7 and v8?">
v7 merged Remix into React Router, with declarative, data and framework modes, type-safe route modules, and SSR/prerendering through a Vite plugin. v8 (June 2026) is ESM-only, requires React 19.2.7+ / Node 22.22+ / Vite 7+, makes middleware the default and removes `react-router-dom`.
</Accordion>

<Accordion title="How do you pass data between routes?">
Use URL params or search params for anything that should be shareable. Use `location.state` for transient data (lost on refresh in a new tab). Loaders can refetch by ID.
</Accordion>

## Related
- [React Index](/Software%20Engineering/Frontend/05%20React/React%20Index) · [URL as State](/Software%20Engineering/Frontend/06%20State%20Management/URL%20as%20State) · [Forms & Actions in React](/Software%20Engineering/Frontend/05%20React/Forms%20%26%20Actions%20in%20React) · [React Server Components & Server Functions](/Software%20Engineering/Frontend/05%20React/React%20Server%20Components%20%26%20Server%20Functions)
- [Next.js App Router Fundamentals](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20App%20Router%20Fundamentals) · [Rendering Patterns - CSR, SSR, SSG, ISR, Streaming & Islands](/Software%20Engineering/Frontend/09%20Architecture/Rendering%20Patterns%20-%20CSR%2C%20SSR%2C%20SSG%2C%20ISR%2C%20Streaming%20%26%20Islands)
- [Modern Web Platform APIs 2025-2026](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Modern%20Web%20Platform%20APIs%202025-2026) (Navigation API, URLPattern)

## Sources
- [reactrouter.com docs](https://reactrouter.com/) · [Picking a mode](https://reactrouter.com/start/modes) · [Upgrading from v6](https://reactrouter.com/upgrading/v6)
- [React Router v8 (remix.run blog, 2026-06-17)](https://remix.run/blog/react-router-v8) · [Remix 3 RC](https://remix.run/blog/remix-3-release-candidate)
- Owner's original links: [history library](https://github.com/ReactTraining/history) · [React Context (ui.dev)](https://ui.dev/react-context) · [BrowserRouter source (v6)](https://github.com/ReactTraining/react-router/blob/dev/packages/react-router-dom/index.tsx#L98) · [Router context source (v6)](https://github.com/ReactTraining/react-router/blob/dev/packages/react-router/index.tsx#L245) · [Imperative vs declarative (ui.dev)](https://ui.dev/imperative-vs-declarative-programming)
- [CVE-2025-55182 advisory (react.dev)](https://react.dev/blog/2025/12/03/critical-security-vulnerability-in-react-server-components)
