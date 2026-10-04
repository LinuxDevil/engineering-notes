---
title: "Deploying Next.js Micro-frontends to Kubernetes"
tags: [devops, devops/kubernetes, frontend/nextjs, frontend/architecture]
aliases: [Next.js Multi-Zones on Kubernetes, Next.js micro-frontends on K8s, Deploying shell catalog checkout, Self-hosting Next.js on Kubernetes]
updated: 2026-09-27
---
# Deploying Next.js Micro-frontends to Kubernetes

<Note>
**TL;DR**

In 2026 the officially documented way to split a Next.js app into independently deployed micro-frontends is **Multi-Zones**: several normal Next.js apps, each owning a set of paths (`basePath` / `assetPrefix`), stitched together on one domain by a router. On Kubernetes that router is a **Gateway API `HTTPRoute`** with path prefixes. Each zone ships as an `output: 'standalone'` image run with `node server.js` as non-root, has its own Deployment/HPA/PDB and Argo CD Application, and can be released and rolled back on its own. What makes it work in production: one image promoted across environments (so avoid env-specific `NEXT_PUBLIC_*`), a shared cache handler (Redis) when you run more than one Pod, `deploymentId` plus long-lived static assets for version skew, and same-host cookies for auth. Cross-zone navigation is a full page load **by design**.
</Note>

This note deploys the frontend of the **Acme Shop** example: `shell` (`/`), `catalog` (`/catalog/*`) and `checkout` (`/checkout/*`), all Next.js 16, images `ghcr.io/acme/{shell,catalog,checkout}`, on `shop.acme.dev`. The backend `orders-api` is covered in [Deploying Node.js Apps to Kubernetes](/Software%20Engineering/DevOps/Delivery%20Platform/Deploying%20Node.js%20Apps%20to%20Kubernetes). Micro-frontend theory (why, principles, Module Federation mechanics) is in [Micro-frontends & Module Federation](/Software%20Engineering/Frontend/09%20Architecture/Micro-frontends%20%26%20Module%20Federation); Next.js caching semantics are in [Next.js Rendering & Caching](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Rendering%20%26%20Caching).

## 1. Choosing a micro-frontend approach for Next.js (2026)

| Approach | How it composes | Independent deploys | Next.js App Router support | Fits Acme Shop? |
|---|---|---|---|---|
| **Multi-Zones** | Separate Next.js apps per path prefix, routed by a proxy or the shell's `rewrites` | Yes | **Yes, official Next.js guide** | **Yes (chosen)** |
| Module Federation (`@module-federation/nextjs-mf`) | Runtime-loaded remote components inside one page | Yes | **No.** Maintenance mode since 2024-11-01, Pages Router only | No |
| Build-time packages | Zones/features published as npm packages, composed at build | No (consumer must rebuild) | Yes | For shared UI only |
| Client-side composition | Web Components, iframes, or a client-rendered shell loading bundles | Yes | Works, but gives up SSR/RSC for the embedded parts | No |
| **Vercel Microfrontends** | Managed Multi-Zones: Vercel's edge routes paths to separate Vercel projects via `microfrontends.json` | Yes | Yes | Only if hosted on Vercel |

### Multi-Zones (official)

The Next.js docs describe Multi-Zones as splitting one domain into several Next.js apps, each serving a set of paths. Key rules from the guide (docs version 16.3.6):

- A zone is a normal Next.js app. Give non-default zones a prefix so their `/_next/...` assets don't collide: the guide uses `assetPrefix` (served under `/<prefix>/_next/...`); since Next.js 15 no extra rewrite for those assets is needed. We use `basePath` instead, so pages **and** assets live under `/catalog` and a single path-prefix rule routes everything.
- The default zone (our `shell`) needs no prefix.
- Route requests with any HTTP proxy, or let one Next.js app route the whole domain with `rewrites` (proxy-based routing in `proxy.ts` is for dynamic decisions such as feature-flagged migrations).
- **Link across zones with `<a>`, not `<Link>`**: `<Link>` tries to prefetch and soft-navigate relative paths inside the current app, which cannot work for another app. Inside a zone, soft navigation keeps working.
- Navigation within a zone is soft; navigation **between** zones is a hard navigation (full document load). Put pages that users move between often in the same zone.
- Share code through a monorepo or npm packages; use feature flags to switch features on across zones at the same time.
- Server Actions need the public origin allow-listed (`experimental.serverActions.allowedOrigins`), because the user-facing domain serves several apps.

### Module Federation for Next.js: not an option for the App Router

On **2024-11-01** the Module Federation maintainers announced that Next.js support is in **maintenance mode** (issue #3153): the App Router gets no support, the Pages Router only gets minimal fixes, and the plugin is planned to be fully deprecated around mid-to-end 2026. They recommend other frameworks (Modern.js, Remix, TanStack) for MF-based micro-frontends. The package is still being published (`8.8.x` in September 2026), so existing Pages Router setups keep working, but don't start new App Router work on it. Next.js 16's switch to Turbopack by default (webpack-based plugins need `--webpack`) makes it even less attractive; see [Next.js 15 to 16.3](/Software%20Engineering/Frontend/07%20Next.js/Next.js%2015%20to%2016.3).

### Build-time packages and client-side composition

- **Build-time packages** (e.g. `@acme/ui`, `@acme/auth`) are how zones share code, not how they deploy independently: changing a package means every consumer rebuilds and redeploys. Use them for the design system and utilities.
- **Client-side composition** (Web Components, iframes, a SPA shell) makes sense when zones use different frameworks or embed third-party widgets. You lose server rendering and RSC streaming for the embedded parts, and pay for duplicated runtime JS.

### Vercel Microfrontends (managed contrast)

Vercel made its Microfrontends product **generally available on 2025-10-31**. Each micro-frontend is its own Vercel project; a `microfrontends.json` config (plus the `@vercel/microfrontends` package and a local development proxy) tells Vercel's network which paths go to which project. It is essentially managed Multi-Zones with routing at Vercel's edge, preview links across projects and toolbar overrides. Pricing at time of writing: 2 microfrontend projects included on Hobby/Pro, then $250 per extra project per month on Pro and $2 per million routed requests. On Kubernetes you build the same thing yourself: the Gateway is your router, Argo CD your per-project deployment.

<Info>
**🆕 2025–2026**

- **2024-11-01 → 2026**: Module Federation's `nextjs-mf` in maintenance mode (Pages Router only, no App Router), with full deprecation planned for mid-to-end 2026.
- **Next.js 16.0 (2025-10-21)**: Turbopack is the default bundler; **`cacheHandlers`** (plural) introduced for `'use cache'` / `'use cache: remote'`; `next/image` defaults changed (`minimumCacheTTL` 4 hours, `qualities: [75]`). See [Next.js 15 to 16.3](/Software%20Engineering/Frontend/07%20Next.js/Next.js%2015%20to%2016.3).
- **Vercel Microfrontends GA (2025-10-31)**: managed Multi-Zones routing on Vercel.
- **Next.js 16.1.7 (2026-03-16)**: `images.maximumDiskCacheSize` caps the per-Pod optimized-image disk cache.
- **Next.js 16.2.0 (2026-03-18)**: with `deploymentId` set, the build ID is constant and skew is detected from the deployment ID (response header) instead.
- **Grafana Faro Web SDK v2 (stable from Nov 2025)**: web-vitals v5 (FID removed), simpler setup, legacy packages removed.
- **ingress-nginx retired (2026-03-24)**: zones are routed with Gateway API `HTTPRoute`s (see [Kubernetes Networking - Services, Ingress & Gateway API](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Networking%20-%20Services%2C%20Ingress%20%26%20Gateway%20API)).
</Info>

### Recommendation for Acme Shop

**Multi-Zones behind the Gateway API**, because:

1. It is the approach the Next.js team documents and tests; each zone is a plain Next.js app with full App Router, RSC, streaming and Cache Components.
2. Routing belongs to the infrastructure we already run (the shared `Gateway` from [Kubernetes Networking - Services, Ingress & Gateway API](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Networking%20-%20Services%2C%20Ingress%20%26%20Gateway%20API)); no zone is a single point of failure for the others.
3. Each team owns its repo, image, overlays and Argo CD Application, and can release and roll back alone.
4. The seams are URL-shaped (`/catalog`, `/checkout`), which matches how the business splits ownership. The cost (a full page load between zones) is acceptable at those seams: browsing to checkout is a natural boundary.

## 2. Architecture

{/* diagram source: /diagrams/diagram-148.mmd */}
![diagram](/diagrams/diagram-148.svg)

Gateway API picks the **longest matching path prefix**, and `PathPrefix` matches whole path segments: `/catalog` matches `/catalog` and `/catalog/shoes`, but not `/catalogue`. So `/` catches everything no other route claims.

## 3. Zone configuration

### Repo layout (one zone)

```text
acme/catalog/
  app/
    layout.tsx
    page.tsx                 # served at /catalog
    [slug]/page.tsx          # /catalog/red-shoes
    api/health/route.ts      # /catalog/api/health (basePath applies!)
  components/FaroInit.tsx
  instrumentation.ts
  cache-handler.mjs          # ISR / route cache → Redis
  cache-handlers/remote.mjs  # 'use cache: remote' → Redis
  next.config.ts
  Dockerfile
  .dockerignore
  package.json
```

### next.config.ts (catalog)

```ts
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  output: 'standalone',                 // minimal server + traced node_modules
  basePath: '/catalog',                 // build-time; inlined in client bundles
  // Optional CDN for /_next/static. Build-time too. Upload .next/static to
  // s3://acme-static/catalog/_next/static/ in CI (see section 7).
  assetPrefix: process.env.ASSET_PREFIX, // e.g. https://static.shop.acme.dev/catalog

  // Version skew protection (section 7). Same value for every Pod of one build.
  deploymentId: process.env.GIT_SHA,

  // Multi-Pod caching (section 8)
  cacheHandler: require.resolve('./cache-handler.mjs'),
  cacheMaxMemorySize: 0,                // don't keep a per-Pod in-memory copy
  cacheHandlers: {
    remote: require.resolve('./cache-handlers/remote.mjs'), // 'use cache: remote'
  },

  images: {
    remotePatterns: [new URL('https://images.acme.dev/**')],
    minimumCacheTTL: 14400,             // the 16.x default (4h), made explicit
  },

  experimental: {
    serverActions: {
      allowedOrigins: ['shop.acme.dev', 'dev.shop.acme.dev'],
    },
  },

  async redirects() {
    return [
      // basePath: false lets a zone redirect outside its own prefix
      { source: '/catalog/cart', destination: '/checkout/cart', basePath: false, permanent: false },
    ];
  },
};

export default nextConfig;
```

`checkout` is the same with `basePath: '/checkout'`. `shell` has no `basePath` (it's the default zone) and no rewrites, because the Gateway does the routing.

### Linking between zones

```tsx
// shell/app/components/SiteHeader.tsx (also exported from @acme/ui for the other zones)
import Link from 'next/link';

export function SiteHeader() {
  return (
    <nav>
      <Link href="/">Home</Link>                 {/* same zone (shell): soft navigation */}
      <a href="/catalog">Catalog</a>             {/* other zone: hard navigation, no prefetch */}
      <a href="/checkout/cart">Cart</a>
    </nav>
  );
}

// catalog/app/[slug]/page.tsx
<Link href="/red-shoes">Red shoes</Link>        // becomes /catalog/red-shoes automatically
<a href="/checkout/cart">Go to cart</a>         // other zone: plain <a>, absolute path
<Image src="/catalog/shoe.png" alt="" width={400} height={400} />  // next/image needs basePath in src
```

A shared header component has to know which links are "home zone". A common trick is a `zone` prop: `<SiteHeader zone="catalog" />` renders `<Link>` for its own prefix and `<a>` for the others.

## 4. Build for containers

### Dockerfile (per zone, based on the official `with-docker` example)

```dockerfile
# syntax=docker/dockerfile:1
ARG NODE_VERSION=24-trixie-slim

# ---------- deps ----------
FROM node:${NODE_VERSION} AS deps
WORKDIR /app
COPY package.json package-lock.json .npmrc* ./
RUN --mount=type=cache,target=/root/.npm npm ci --no-audit --no-fund

# ---------- build ----------
FROM node:${NODE_VERSION} AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NODE_ENV=production NEXT_TELEMETRY_DISABLED=1
ARG GIT_SHA
ARG ASSET_PREFIX
ENV GIT_SHA=${GIT_SHA} ASSET_PREFIX=${ASSET_PREFIX}
# Same key for every Pod of this build (Server Functions encryption, see pitfalls)
RUN --mount=type=secret,id=sa_key,env=NEXT_SERVER_ACTIONS_ENCRYPTION_KEY \
    npm run build

# ---------- runtime ----------
FROM node:${NODE_VERSION} AS runner
WORKDIR /app
ARG GIT_SHA
ENV GIT_SHA=${GIT_SHA} \
    NODE_ENV=production \
    NEXT_TELEMETRY_DISABLED=1 \
    PORT=3000 \
    HOSTNAME=0.0.0.0
COPY --from=builder --chown=node:node /app/public ./public
COPY --from=builder --chown=node:node /app/.next/standalone ./
COPY --from=builder --chown=node:node /app/.next/static ./.next/static
USER node
EXPOSE 3000
CMD ["node", "server.js"]
```

- `output: 'standalone'` writes `.next/standalone` with a minimal `server.js` and only the traced `node_modules`, so the runtime image needs no `npm install`. It does **not** include `public/` or `.next/static`; copy them (as above) or serve them from a CDN.
- `server.js` listens on `PORT` and `HOSTNAME`. Set `HOSTNAME=0.0.0.0`; Kubernetes sets `HOSTNAME` to the Pod name by default, which would make the server bind only to the Pod's own address resolution and can break probes or port-forwarding.
- Since Next.js 15, `sharp` is used automatically with `next start` and standalone output; no manual install. On glibc Linux, sharp's memory use can grow with the default allocator (the docs point to sharp's allocator guidance); watch the Pod's working set.
- `CMD ["node", "server.js"]` in exec form, so Node is PID 1 and receives SIGTERM (never `npm start`; see the Node note). Next's server finishes in-flight requests and pending `after()` callbacks on SIGTERM/SIGINT.
- In a monorepo set `outputFileTracingRoot` to the repo root so shared packages are traced (see [Monorepos](/Software%20Engineering/Frontend/14%20Tooling/Monorepos)).
- The build needs the `.next/cache` directory writable at runtime (ISR, images). With `readOnlyRootFilesystem: true`, mount an `emptyDir` there.

```gitignore
# .dockerignore
node_modules
.next
.git
coverage
*.log
.env*
Dockerfile*
README.md
```

## 5. Build-time vs runtime configuration

`NEXT_PUBLIC_*` variables are **inlined into the client JavaScript at `next build`**. `basePath` and `assetPrefix` are build-time too. Plain server env vars (`process.env.X` in server code) are read at runtime, as long as the code runs at request time (dynamic rendering, or after `await connection()`), not during prerendering.

| Pattern | How | Pros | Cons |
|---|---|---|---|
| **One image per environment** | Build `catalog:SHA-dev`, `catalog:SHA-prod` with different `NEXT_PUBLIC_*` | Simple; values are static and tree-shakeable | Prod runs an image that was never tested in staging; N builds per commit; breaks "build once, promote" GitOps |
| **One image, runtime config** | Server reads env at request time and passes public values to the client | Same artifact in every env; promotion = tag bump | Config reads must be dynamic; a little plumbing |
| **Avoid the need** | Same-origin relative URLs (`/api/orders`, `/faro/collect`) | Nothing to configure | Only works for same-domain endpoints (which Multi-Zones on one domain gives you) |

Acme Shop uses **one image per commit, promoted unchanged**. Public, env-specific values are rare, because everything the browser calls is same-origin. The few that remain (environment name for Faro, a feature-flag client key) come from runtime env:

```tsx
// app/layout.tsx (catalog)
import { Suspense } from 'react';
import { connection } from 'next/server';
import { PublicConfigProvider } from '@acme/ui/public-config';
import { FaroInit } from '@/components/FaroInit';

async function RuntimePublicConfig({ children }: { children: React.ReactNode }) {
  await connection();                        // read env at request time, not at build
  const cfg = {
    env: process.env.ACME_ENV ?? 'dev',      // plain env var, set by the overlay
    flagsClientKey: process.env.FLAGS_CLIENT_KEY ?? '',
    version: process.env.GIT_SHA ?? 'unknown',
  };
  return <PublicConfigProvider value={cfg}><FaroInit config={cfg} />{children}</PublicConfigProvider>;
}

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <Suspense fallback={null}>
          <RuntimePublicConfig>{children}</RuntimePublicConfig>
        </Suspense>
      </body>
    </html>
  );
}
```

Trade-off: anything wrapped this way renders at request time. Keep the dynamic part small (a provider around client islands), or put the values in a tiny cached-per-deploy endpoint such as `/catalog/api/public-config` that client code fetches once. Never put secrets in anything the client receives.

## 6. Routing on Kubernetes

### Gateway API HTTPRoutes (one per zone, owned by each team)

```yaml
# apps/shell/base/httproute.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: shell
spec:
  parentRefs:
    - { name: shop-gateway, namespace: gateway }
  hostnames: ["shop.acme.dev"]          # overlays patch per env
  rules:
    - matches:
        - path: { type: PathPrefix, value: / }
      backendRefs:
        - { name: shell, port: 80 }
      timeouts: { request: 30s }
---
# apps/catalog/base/httproute.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: catalog
spec:
  parentRefs:
    - { name: shop-gateway, namespace: gateway }
  hostnames: ["shop.acme.dev"]
  rules:
    - matches:
        - path: { type: PathPrefix, value: /catalog }
      backendRefs:
        - { name: catalog, port: 80 }
      timeouts: { request: 30s }
---
# apps/checkout/base/httproute.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: checkout
spec:
  parentRefs:
    - { name: shop-gateway, namespace: gateway }
  hostnames: ["shop.acme.dev"]
  rules:
    - matches:
        - path: { type: PathPrefix, value: /checkout }
      backendRefs:
        - { name: checkout, port: 80 }
      timeouts: { request: 30s }
---
# platform/gateway/faro-route.yaml (browser telemetry, same origin, no CORS)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: faro
  namespace: monitoring
spec:
  parentRefs:
    - { name: shop-gateway, namespace: gateway }
  hostnames: ["shop.acme.dev"]
  rules:
    - matches:
        - path: { type: PathPrefix, value: /faro }
      filters:
        - type: URLRewrite
          urlRewrite:
            path: { type: ReplacePrefixMatch, replacePrefixMatch: / }
      backendRefs:
        - { name: alloy-faro, port: 12347 }
```

In `shop-prod` the checkout route's backend is `checkout-active` (the blue-green active Service managed by Argo Rollouts, see section 10), patched in by the prod overlay.

No `URLRewrite` for the zones: because of `basePath`, the `catalog` app expects to receive `/catalog/...` unchanged. The Gateway's listener must allow routes from the `shop-*` namespaces (`allowedRoutes`), and `kubectl describe httproute catalog -n shop-prod` shows whether the parent accepted it.

### Alternative: routing in the shell with `rewrites`

```ts
// shell/next.config.ts (alternative, not used by Acme Shop)
async rewrites() {
  return [
    { source: '/catalog', destination: `${process.env.CATALOG_ORIGIN}/catalog` },
    { source: '/catalog/:path+', destination: `${process.env.CATALOG_ORIGIN}/catalog/:path+` },
    { source: '/checkout', destination: `${process.env.CHECKOUT_ORIGIN}/checkout` },
    { source: '/checkout/:path+', destination: `${process.env.CHECKOUT_ORIGIN}/checkout/:path+` },
  ];
}
```

| | Gateway `HTTPRoute` | Shell `rewrites` |
|---|---|---|
| Extra hop | None | Every zone request goes through a shell Pod |
| Blast radius | Shell outage affects `/` only | Shell outage takes down every zone |
| Scaling | Each zone scales on its own traffic | Shell must scale for everyone's traffic |
| Ownership | Each team owns its route file | Shell team owns everyone's routing; changes need a shell deploy |
| Canary / weights | Built into `HTTPRoute` (weighted `backendRefs`), Argo Rollouts plugin | Not available |
| Portability | Kubernetes-specific | Works anywhere Next runs, including `next dev` locally |
| Local dev | Needs a local proxy (or use rewrites only in dev) | Works out of the box |

Common compromise: Gateway in clusters, and `rewrites` in the shell enabled only when `NODE_ENV === 'development'` to stitch local `next dev` servers together.

## 7. Shared concerns across zones

### Auth and session cookies

All zones are on the **same host** (`shop.acme.dev`), so a cookie set with `Path=/` by any zone is sent to all of them.

- Use a host-only cookie such as `__Host-acme_session` (the `__Host-` prefix requires `Secure`, `Path=/` and **no** `Domain` attribute, so it can't leak to other `*.acme.dev` subdomains), `HttpOnly`, `SameSite=Lax`.
- One zone (or a dedicated auth service) issues the session; every zone validates it on the server (verify a signed JWT against a JWKS, or look the session up in Redis). Put the check in each zone's `proxy.ts` or a shared `@acme/auth` package.
- Don't set cookies with `Path=/catalog` unless you mean it: checkout will not see them.
- `dev.shop.acme.dev` and `shop.acme.dev` are different hosts, so dev and prod sessions never mix.
- Details on token storage and CSRF: [Authentication & Token Storage](/Software%20Engineering/Frontend/11%20Security/Authentication%20%26%20Token%20Storage), [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types).

### Shared design system and version skew

Each zone bundles its own copy of `@acme/ui`. Because navigation between zones is a full page load, two versions never run on the same page, so there is no runtime conflict, but users **see** skew: a header that changes style when they move from catalog to checkout.

- Version `@acme/ui` with semver; zones may lag by at most one minor version (enforced by Renovate plus a CI check that fails on a stale major).
- Keep the cross-zone chrome (header, footer, fonts, tokens) in `@acme/ui` and change it with additive, backward-compatible releases.
- Global CSS resets must be identical, or pages jump when crossing zones.
- Feature flags that span zones (e.g. a new checkout button in catalog) must be evaluated from the same flag service so zones switch together.

### Static assets, CDN and cache headers

- With `basePath: '/catalog'`, assets are at `/catalog/_next/static/...`. Filenames are content-hashed and Next serves them with `Cache-Control: public, max-age=31536000, immutable`, so a CDN can cache them forever.
- Better for rollouts: upload `.next/static` to object storage in CI (`s3://acme-static/catalog/_next/static/`), set `assetPrefix` to the CDN URL, and **keep old builds' files** for days. Then a browser holding an old HTML page still finds its chunks after the Pods have moved on.
- `public/` files are **not** covered by `assetPrefix`; they are served by the zone under its `basePath` (and `next/image` needs the basePath in `src`).
- Dynamic pages send `private, no-cache, no-store, max-age=0, must-revalidate`; prerendered pages are `public` and CDN-cacheable. The CDN must forward `RSC`, `Next-Router-State-Tree`/`Next-Router-Prefetch`-style request headers and respect `Vary`, or it may serve an RSC payload where HTML was expected (see the Next.js CDN caching guide).
- Streaming must not be buffered anywhere between the browser and Next (CDN, Gateway). Envoy-based Gateways stream by default; check your CDN.

### Deployment skew across replicas

During a rolling update, old and new Pods of the **same** zone serve traffic together. Risks listed by the Next.js self-hosting guide: missing JS/CSS chunks, Server Function IDs the other version doesn't know, and incompatible prefetched RSC payloads.

- **`deploymentId`** (stable since 14.1.4; set via config or `NEXT_DEPLOYMENT_ID`): Next appends `?dpl=<id>` to asset URLs, sends `x-deployment-id` on client navigations, and when the client and server IDs differ it does a **hard navigation** instead of a soft one. Since **16.2** the build ID is constant when `deploymentId` is set, so `generateBuildId` has no effect then.
- Next does **not** route on `?dpl=`: a request can still reach a Pod of the other version, which then forces a reload. The asset-in-object-storage approach above removes the "chunk 404" part.
- **`generateBuildId`** (e.g. the git SHA) matters if you ever build the same commit twice (one image per environment); every Pod of a release must share it. Better: build once.
- **`NEXT_SERVER_ACTIONS_ENCRYPTION_KEY`**: each build otherwise generates its own key, and Pods from different builds fail with "Failed to find Server Action". Provide one key (base64, 16/24/32 bytes) at build time, as the Dockerfile does with a build secret.
- **Acme Shop deploys `checkout` with Argo Rollouts blue-green, not canary.** A canary would send some requests to v2 Pods while the browser still holds v1 HTML and asks for v1 JS chunks (or vice versa); mixing HTML from one version with JS assets from another is exactly the skew above, and in checkout it costs orders. Blue-green brings the new version up behind a preview Service, runs smoke tests, then flips the active Service at once and keeps blue running for a fast `undo`. The cost is double capacity during the release. `shell` and `catalog` use plain rolling updates with the protections above. Details: [Argo Rollouts - Canary & Blue-Green](/Software%20Engineering/DevOps/Delivery%20Platform/Argo%20Rollouts%20-%20Canary%20%26%20Blue-Green).

## 8. Multi-replica caching

By default Next stores ISR pages, the data cache and optimized images on **each Pod's local disk and memory**. With three `catalog` Pods, `revalidateTag('products')` on one Pod leaves the other two serving stale pages until their own entries expire.

Two separate knobs in Next.js 16 (verified in the 16.3.6 docs):

| Config | Covers | Interface |
|---|---|---|
| `cacheHandler` (singular) + `cacheMaxMemorySize: 0` | ISR / prerendered route cache, Route Handler responses, fetch data cache (the "incremental cache"); optimized images when it's set | Class with `get`, `set`, `revalidateTag`, `resetRequestCache` |
| `cacheHandlers` (plural, **new in 16.0**) | `'use cache'` (`default` key) and `'use cache: remote'` (`remote` key); you can add named handlers used as `'use cache: <name>'` | Object with `get`, `set`, `refreshTags`, `getExpiration`, `updateTags` |

`'use cache: private'` can't be customized. The default for `cacheHandlers` is an in-memory LRU per process.

### Redis-backed handlers (sketch)

```js
// cache-handlers/remote.mjs  ('use cache: remote' → Redis, with cross-Pod tag invalidation)
import { createClient } from 'redis';

const redis = createClient({ url: process.env.REDIS_URL });
await redis.connect();
const tagTimes = new Map();                        // local copy, refreshed per request

async function streamToBuffer(stream) {
  const chunks = [];
  for await (const c of stream) chunks.push(Buffer.from(c));
  return Buffer.concat(chunks);
}

export default {
  async get(key, softTags) {
    try {
      const raw = await redis.get(`uc:${key}`);
      if (!raw) return undefined;
      const e = JSON.parse(raw);
      const newestTag = Math.max(0, ...[...e.tags, ...softTags].map(t => tagTimes.get(t) ?? 0));
      if (newestTag > e.timestamp) return undefined;               // invalidated after it was cached
      if (Date.now() > e.timestamp + e.revalidate * 1000) return undefined;
      const buf = Buffer.from(e.value, 'base64');
      return { ...e, value: new ReadableStream({ start(c) { c.enqueue(buf); c.close(); } }) };
    } catch {
      return undefined;                                            // get() must never throw
    }
  },
  async set(key, pending) {
    const e = await pending;
    const value = (await streamToBuffer(e.value)).toString('base64');
    await redis.set(`uc:${key}`, JSON.stringify({ ...e, value }), { EX: e.expire });
  },
  async refreshTags() {                                            // called before requests
    const tags = await redis.sMembers('uc:tags');
    if (!tags.length) return;
    const times = await redis.mGet(tags.map(t => `uc:tag:${t}`));
    tags.forEach((t, i) => tagTimes.set(t, Number(times[i] ?? 0)));
  },
  async getExpiration(tags) {
    return Math.max(0, ...tags.map(t => tagTimes.get(t) ?? 0));
  },
  async updateTags(tags) {                                         // revalidateTag / updateTag
    const now = Date.now();
    const m = redis.multi();
    for (const t of tags) { m.set(`uc:tag:${t}`, String(now)); m.sAdd('uc:tags', t); tagTimes.set(t, now); }
    await m.exec();
  },
};
```

For `cacheHandler` (ISR), start from the official `cache-handler-redis` example in the Next.js repo rather than writing your own. Production concerns: key prefixes per zone and per `deploymentId` (so a new build doesn't read incompatible entries; Next already adds `deploymentId` to `'use cache'` keys), Redis memory limits and eviction policy (`volatile-lru`), timeouts so a slow Redis degrades to a cache miss rather than a hanging render, and HA Redis (Sentinel, cluster or a managed service).

### Image optimization cache

- `next/image` writes optimized images to `<distDir>/cache/images` on each Pod, capped by `images.maximumDiskCacheSize` (added in **16.1.7**; default: 50% of free disk measured at startup; `0` disables it). There is no invalidation API; keep `minimumCacheTTL` modest (default 4 hours since Next 16).
- On Kubernetes that directory is an `emptyDir`, so every new Pod starts cold and each Pod re-optimizes the same images. Put the CDN in front of `/catalog/_next/image` (responses are cacheable), give the `emptyDir` a `sizeLimit`, and account for sharp's CPU and memory in the Pod's requests.
- For heavy image traffic, move optimization out: a custom `loader` pointing at an image service or CDN image resizing, and keep the Next Pods for rendering.

## 9. Kubernetes manifests per zone

### Health route (there is no built-in one)

```ts
// app/api/health/route.ts  → GET /catalog/api/health (basePath applies)
import { connection } from 'next/server';

export async function GET() {
  await connection();        // always run at request time, never prerendered
  return Response.json({ status: 'ok', version: process.env.GIT_SHA ?? 'unknown' });
}
```

Keep it dependency-free (it serves liveness and readiness). A Next zone can usually render something useful when a backend is down (error boundaries, cached data), so don't make readiness depend on `orders-api` or Redis.

### Kustomize base (catalog)

```yaml
# apps/catalog/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources: [deployment.yaml, service.yaml, httproute.yaml, hpa.yaml, pdb.yaml]
images:
  - { name: ghcr.io/acme/catalog, newTag: placeholder }
```

```yaml
# apps/catalog/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalog
  labels: { app.kubernetes.io/name: catalog, app.kubernetes.io/part-of: acme-shop }
spec:
  selector: { matchLabels: { app.kubernetes.io/name: catalog } }
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxSurge: 25%, maxUnavailable: 0 }
  template:
    metadata:
      labels: { app.kubernetes.io/name: catalog }
    spec:
      automountServiceAccountToken: false
      terminationGracePeriodSeconds: 40
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000                          # the "node" user in the official image
        seccompProfile: { type: RuntimeDefault }
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector: { matchLabels: { app.kubernetes.io/name: catalog } }
      containers:
        - name: catalog
          image: ghcr.io/acme/catalog
          ports: [{ name: http, containerPort: 3000 }]
          env:
            - { name: HOSTNAME, value: "0.0.0.0" }   # override the Pod-name HOSTNAME
            - { name: PORT, value: "3000" }
            - { name: ACME_ENV, value: dev }
            - { name: ORDERS_API_URL, value: http://orders-api.shop-dev.svc }   # server-side calls stay in-cluster
            - { name: REDIS_URL, valueFrom: { secretKeyRef: { name: shop-redis, key: url } } }
            - { name: OTEL_SERVICE_NAME, value: catalog }
            - { name: OTEL_EXPORTER_OTLP_ENDPOINT, value: http://alloy.monitoring.svc:4318 }
            - { name: NODE_OPTIONS, value: --max-old-space-size-percentage=70 }
          resources:
            requests: { cpu: 500m, memory: 1Gi }
            limits:   { memory: 1Gi }
          startupProbe:
            httpGet: { path: /catalog/api/health, port: http }
            periodSeconds: 2
            failureThreshold: 30
          livenessProbe:
            httpGet: { path: /catalog/api/health, port: http }
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          readinessProbe:
            httpGet: { path: /catalog/api/health, port: http }
            periodSeconds: 5
            timeoutSeconds: 3
          lifecycle:
            preStop:
              sleep: { seconds: 10 }             # let the Gateway drop this endpoint first
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: { drop: ["ALL"] }
          volumeMounts:
            - { name: next-cache, mountPath: /app/.next/cache }
            - { name: tmp, mountPath: /tmp }
      volumes:
        - name: next-cache
          emptyDir: { sizeLimit: 2Gi }
        - name: tmp
          emptyDir: { sizeLimit: 128Mi }
```

```yaml
# apps/catalog/base/service.yaml
apiVersion: v1
kind: Service
metadata: { name: catalog }
spec:
  selector: { app.kubernetes.io/name: catalog }
  ports: [{ name: http, port: 80, targetPort: http }]
---
# apps/catalog/base/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: catalog }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: catalog }
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 60 } }
  behavior:
    scaleDown: { stabilizationWindowSeconds: 300 }
---
# apps/catalog/base/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: catalog }
spec:
  maxUnavailable: 1
  selector: { matchLabels: { app.kubernetes.io/name: catalog } }
```

```yaml
# apps/catalog/overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: shop-prod
resources: [../../base]
images:
  - { name: ghcr.io/acme/catalog, newTag: 9e41d07 }      # bumped by Jenkins
patches:
  - target: { kind: Deployment, name: catalog }
    patch: |
      - op: replace
        path: /spec/template/spec/containers/0/env/2/value
        value: prod
      - op: replace
        path: /spec/template/spec/containers/0/env/3/value
        value: http://orders-api.shop-prod.svc
  - target: { kind: HorizontalPodAutoscaler, name: catalog }
    patch: |
      - { op: replace, path: /spec/minReplicas, value: 3 }
      - { op: replace, path: /spec/maxReplicas, value: 40 }
```

(Index-based JSON patches are brittle; a strategic-merge patch on the container `env` by name is safer once the list grows.) Server-side fetches from Server Components go straight to `orders-api` inside the cluster; the browser uses the same-origin `/api/orders` through the Gateway.

`shell` and `checkout` are the same with their own names, probe paths (`/api/health` for shell, `/checkout/api/health` for checkout) and sizes. Checkout usually runs more replicas with a stricter PDB because it is revenue-critical.

## 10. Argo CD: every zone deployed independently

One `ApplicationSet` generates an Argo CD Application per zone per environment, so each zone syncs, fails and rolls back on its own.

```yaml
# platform-gitops/applicationsets/shop.yaml (trimmed; the full version with notifications,
# retry and ignoreDifferences is in [[Argo CD in Practice]])
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: shop
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    - matrix:
        generators:
          - git:                                   # one entry per app folder: shell, catalog, checkout, orders-api
              repoURL: https://github.com/acme/platform-gitops.git
              revision: main
              directories:
                - path: apps/*
          - list:                                  # one entry per environment
              elements:
                - { env: dev,     cluster: in-cluster }
                - { env: staging, cluster: in-cluster }
                - { env: prod,    cluster: in-cluster }
  syncPolicy:
    applicationsSync: create-update               # never delete apps automatically
  template:
    metadata:
      name: "{{ .path.basename }}-{{ .env }}"      # catalog-prod, checkout-dev, ...
      labels:
        team: shop
        app: "{{ .path.basename }}"
        env: "{{ .env }}"
    spec:
      project: shop
      source:
        repoURL: https://github.com/acme/platform-gitops.git
        targetRevision: main
        path: "apps/{{ .path.basename }}/overlays/{{ .env }}"
      destination:
        name: "{{ .cluster }}"
        namespace: "shop-{{ .env }}"
      syncPolicy:
        automated: { prune: true, selfHeal: true }
        syncOptions: [ServerSideApply=true, PruneLast=true]
```

Adding a fourth zone is a new directory under `apps/`; no Argo CD change. Deeper ApplicationSet patterns, RBAC per team (`AppProject` per zone) and progressive syncs are in [Argo CD in Practice](/Software%20Engineering/DevOps/Delivery%20Platform/Argo%20CD%20in%20Practice).

### Independent release cadence and rollback

- The catalog team merges → Jenkins builds `ghcr.io/acme/catalog:<sha>` → commits the tag to `apps/catalog/overlays/dev` → Argo CD syncs `catalog-dev` only. Promotion to prod is a PR touching only `apps/catalog/overlays/prod`.
- **Roll back one zone**: `git revert` the tag commit in `apps/catalog/overlays/prod`; Argo CD syncs `catalog-prod` back while shell and checkout stay on their versions. (Argo CD's UI "History and Rollback" works too but disables auto-sync until Git is fixed.) For `checkout`, while the previous (blue) ReplicaSet is still up after a promotion, `kubectl argo rollouts undo checkout -n shop-prod` flips traffic back instantly; then revert the tag in Git so Argo CD agrees.
- The contract between zones is **URLs, cookies and shared packages**. Rolling back one zone is safe as long as those stay backward compatible: never rename a path another zone links to, or change the session cookie format, in the same release that depends on it.

{/* diagram source: /diagrams/diagram-149.mmd */}
![diagram](/diagrams/diagram-149.svg)

## 11. Observability

### Server side

- **Logs**: Next's own server output is not structured. Log from your code (Server Components, Route Handlers, Server Actions) with pino to stdout as JSON with `service: 'catalog'`; Alloy ships it to Loki. If bundling pino causes issues, list it in `serverExternalPackages`.
- **Traces**: Next.js is instrumented for OpenTelemetry out of the box (root span `GET /catalog/[slug]`, render, `fetch` spans; `NEXT_OTEL_VERBOSE=1` for more). Register a provider in `instrumentation.ts`:

```ts
// instrumentation.ts (option A: @vercel/otel, works self-hosted with any OTLP collector)
import { registerOTel } from '@vercel/otel';

export function register() {
  registerOTel({ serviceName: 'catalog' });   // exporter endpoint from OTEL_EXPORTER_OTLP_ENDPOINT
}

// Report server errors (Server Components, Route Handlers, Server Actions, proxy)
export async function onRequestError(err: unknown, request: { path: string; method: string }, context: { routerKind: string; routeType: string }) {
  const { logger } = await import('./lib/logger');   // pino instance
  logger.error({ err, path: request.path, method: request.method, ...context }, 'request error');
}
```

```ts
// Option B: the OpenTelemetry Node SDK directly (Node runtime only)
// instrumentation.ts
export async function register() {
  if (process.env.NEXT_RUNTIME === 'nodejs') await import('./instrumentation.node');
}
// instrumentation.node.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { resourceFromAttributes } from '@opentelemetry/resources';
import { ATTR_SERVICE_NAME } from '@opentelemetry/semantic-conventions';
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-node';

new NodeSDK({
  resource: resourceFromAttributes({ [ATTR_SERVICE_NAME]: 'catalog' }),
  spanProcessor: new BatchSpanProcessor(new OTLPTraceExporter()),
}).start();
```

Traces continue into `orders-api` automatically (W3C `traceparent` on the server-side `fetch`), so one Tempo trace shows catalog render → orders-api → Postgres.

### Client side: Web Vitals and errors with Grafana Faro

```tsx
// components/FaroInit.tsx
'use client';
import { useEffect } from 'react';
import { getWebInstrumentations, initializeFaro, faro } from '@grafana/faro-web-sdk';
import { TracingInstrumentation } from '@grafana/faro-web-tracing';

export function FaroInit({ config }: { config: { env: string; version: string } }) {
  useEffect(() => {
    if (faro.api) return;                       // already initialised (hard nav re-runs this per zone)
    initializeFaro({
      url: '/faro/collect',                     // same origin → Gateway → Alloy faro.receiver
      app: { name: 'catalog', version: config.version, environment: config.env },
      instrumentations: [...getWebInstrumentations(), new TracingInstrumentation()],
    });
  }, [config]);
  return null;
}
```

- `getWebInstrumentations()` includes errors, console, session and **Web Vitals** (LCP, INP, CLS, TTFB, FCP). Faro v2 moved to web-vitals v5 and dropped FID.
- `TracingInstrumentation` propagates trace context on `fetch`/XHR, linking a slow click to the backend trace.
- Alloy's `faro.receiver` (default port 12347) accepts the payloads and forwards logs to Loki and traces to Tempo. Serving it through the Gateway at `/faro` keeps it same-origin, which avoids CORS and most ad blockers.
- Use a **different `app.name` per zone** so Web Vitals are comparable per team. Because each zone boundary is a hard navigation, each page load is attributed to exactly one zone.
- Lighter alternative: Next's `useReportWebVitals` hook posting to your own endpoint.

### Viewing logs per zone in Grafana

```logql
{namespace="shop-prod", app="catalog"} | json | level >= 50
sum by (app) (rate({namespace="shop-prod", app=~"shell|catalog|checkout"} | json | level >= 50 [5m]))
```

Build one dashboard with a `zone` variable (`shell|catalog|checkout`) showing request rate and p95 from the Gateway's metrics per route, error logs, Pod restarts and memory, plus the Faro Web Vitals panel. See [Grafana & the LGTM Stack](/Software%20Engineering/DevOps/Delivery%20Platform/Grafana%20%26%20the%20LGTM%20Stack).

## Pitfalls

<Warning>
**Common mistakes**

- **basePath mistakes**: probes pointing at `/api/health` instead of `/catalog/api/health`; hard-coding `/catalog` inside `<Link href>` (becomes `/catalog/catalog/...`); forgetting the basePath in `next/image` `src` for `public/` files; two zones claiming the same prefix; an `HTTPRoute` with `URLRewrite` that strips the prefix the app expects.
- **Broken assets after deploy**: an old HTML page asks for chunks the new Pods don't have → 404 → blank or broken page. Fix with `deploymentId`, assets kept in object storage behind `assetPrefix`, and `maxUnavailable: 0` rollouts.
- **Using `<Link>` across zones**: prefetches hit the wrong app and navigation breaks. A full page reload between zones is **by design**; use `<a>`.
- **Cookie scope**: `Path=/checkout` cookies invisible to shell; `Domain=.acme.dev` leaking sessions to every subdomain; different `SameSite` settings per zone.
- **Env-specific `NEXT_PUBLIC_*`** in a build-once pipeline: prod silently uses the value baked in at build.
- **No shared cache with several Pods**: users see different versions of a page depending on the Pod; `revalidateTag` "works sometimes".
- **Different Server Actions keys** across builds: "Failed to find Server Action" during canaries.
- **Memory limits for Next servers**: RSC rendering, the in-memory caches (the incremental cache defaults to 50 MB) and sharp all use memory; a 256Mi limit OOMKills under load. Start around 1Gi, set heap to ~70% of the limit, keep `cacheMaxMemorySize: 0` when Redis holds the cache, and load test.
- **`HOSTNAME` not overridden**: Kubernetes sets it to the Pod name, and `server.js` may not listen where the probes connect.
- **`readOnlyRootFilesystem` without a writable `.next/cache`**: ISR and image optimization fail at runtime.
- **Buffering proxies** in front of Next: streaming and PPR lose their benefit; the page arrives all at once.
</Warning>

## Senior perspective

**When micro-frontends are not worth it.** Multi-Zones solve an **organizational** problem: several teams that need to release independently. They cost you a full page load at each seam, duplicated JS per zone, shared-package version management, more pipelines, more dashboards and cross-zone contracts to guard. If one or two teams own the frontend, a single Next.js app in a monorepo with good module boundaries, feature flags and Turbopack-fast builds is better on every axis. Split when build times, release coordination or ownership conflicts are measurably hurting, and split along URL seams users rarely cross.

Other points a senior raises in review:

- **Where do the seams go?** Measure navigation paths first. If 40% of catalog sessions bounce to checkout and back, those hard navigations cost real conversion; maybe the cart drawer belongs in catalog.
- **What's the contract?** Paths, cookies, the header component and events (analytics) are a public API between teams. Version and test them (contract tests that crawl links across zones in staging).
- **Skew is normal, not an edge case.** With three zones deploying independently, at any moment users see mixed versions across zones and, during rollouts, within a zone. Design for it: backward-compatible cookies, additive design-system changes, `deploymentId`, retained assets.
- **Caching is now distributed systems work.** A shared Redis cache is a new dependency with its own failure modes; a cache miss storm after a Redis failover can overload `orders-api`. Timeouts and fallbacks matter.
- **Cost**: three Next servers at 1Gi with 3+ replicas each, plus Redis and a CDN. Compare with one app.
- **Questions to ask**: What happens to a user mid-checkout during a checkout rollout? How do we roll back catalog without breaking links from shell? Which dashboard shows per-zone Web Vitals? Who owns the Gateway routes and what stops two teams claiming the same prefix (admission policy, CODEOWNERS on `apps/*/base/httproute.yaml`)?

## Interview questions

<Accordion title="What options exist for Next.js micro-frontends in 2026, and which would you pick on Kubernetes?">
Multi-Zones (officially documented: separate Next.js apps per path prefix, routed by a proxy or `rewrites`), Module Federation via `nextjs-mf` (maintenance mode since Nov 2024, no App Router support), build-time packages (shared code, not independent deploys), client-side composition (Web Components/iframes, loses SSR), and Vercel Microfrontends (managed Multi-Zones, GA Oct 2025, Vercel-only). On Kubernetes: Multi-Zones with Gateway API `HTTPRoute` path prefixes, because it's supported, keeps full App Router features and puts routing in shared infrastructure rather than in one app.
</Accordion>

<Accordion title="Why is navigation between zones a full page reload, and how do you link correctly?">
Each zone is a separate Next.js app with its own router, bundles and RSC payloads; the client router of one app can't render another app's routes. So cross-zone links must be plain `<a href="/checkout">`, which does a hard navigation. `<Link>` would try to prefetch and soft-navigate inside the current app and fail. Within a zone, `<Link>` (with basePath applied automatically) keeps soft navigation.
</Accordion>

<Accordion title="NEXT_PUBLIC_API_URL differs between staging and prod. How do you handle it with a build-once, promote-everywhere pipeline?">
`NEXT_PUBLIC_*` is inlined at build, so it can't differ between environments for the same image. Options: build one image per environment (breaks artifact promotion), read plain env vars at request time on the server and pass them to the client (via a provider or a small config endpoint), or remove the need by using same-origin relative URLs such as `/api/orders` through the Gateway. I'd use relative URLs first and runtime config for the rest.
</Accordion>

<Accordion title="You run three Pods of a Next.js zone that uses ISR and 'use cache'. What breaks and how do you fix it?">
Each Pod has its own local cache, so pages differ between Pods and `revalidateTag` only invalidates the Pod that handled it. Configure `cacheHandler` (incremental/ISR cache) with `cacheMaxMemorySize: 0` and `cacheHandlers` (`default`/`remote` for `'use cache'`) backed by Redis; implement `updateTags`/`refreshTags`/`getExpiration` so tag invalidations propagate across Pods. Also share the Server Actions encryption key and a `deploymentId`, and handle the image cache (CDN in front or external optimizer).
</Accordion>

<Accordion title="What is deployment skew in Next.js and how do you mitigate it during a rolling update?">
Old and new versions serve traffic at the same time, so a client can request chunks that no longer exist, call Server Actions the other build doesn't know, or receive incompatible RSC payloads. Mitigations: set `deploymentId` (Next then hard-navigates on mismatch and busts asset caches with `?dpl=`), keep old static assets available (object storage + CDN via `assetPrefix`), use one Server Actions key across builds, and consider blue-green for risky releases.
</Accordion>

<Accordion title="Gateway routing vs rewrites in the shell app: trade-offs?">
Gateway: no extra hop, independent scaling and blast radius, per-team route ownership, built-in weighted canaries; Kubernetes-specific and needs a local proxy for dev. Shell rewrites: portable and easy locally, but every request flows through the shell, making it a bottleneck and single point of failure and coupling routing changes to shell deploys.
</Accordion>

<Accordion title="How do authentication cookies work across zones?">
Zones share one host, so a host-only cookie with `Path=/` (ideally `__Host-` prefixed, `Secure`, `HttpOnly`, `SameSite=Lax`) is sent to every zone. Each zone validates the session server-side with shared logic. Pitfalls are path-scoped cookies, broad `Domain` attributes leaking to subdomains, and changing the cookie format without coordinating all zones.
</Accordion>

<Accordion title="When would you advise against micro-frontends?">
When one or two teams own the frontend, when users constantly cross the proposed seams, or when the pain is really slow builds (fixable with Turbopack, caching, monorepo tooling). Micro-frontends trade runtime performance and operational complexity for team autonomy; only pay that when coordination costs are the actual bottleneck.
</Accordion>

## Related

- [DevOps Index](/Software%20Engineering/DevOps/DevOps%20Index) · [Kubernetes Delivery Platform - End to End](/Software%20Engineering/DevOps/Delivery%20Platform/Kubernetes%20Delivery%20Platform%20-%20End%20to%20End)
- [Deploying Node.js Apps to Kubernetes](/Software%20Engineering/DevOps/Delivery%20Platform/Deploying%20Node.js%20Apps%20to%20Kubernetes) (orders-api, graceful shutdown, Node memory)
- [Micro-frontends & Module Federation](/Software%20Engineering/Frontend/09%20Architecture/Micro-frontends%20%26%20Module%20Federation) · [Building Micro frontends with React 18](/Software%20Engineering/Frontend/Courses%20%26%20Books/Building%20Micro-frontends%20with%20React/Building%20Micro%20frontends%20with%20React%2018) · [Monorepos](/Software%20Engineering/Frontend/14%20Tooling/Monorepos)
- [Next.js App Router Fundamentals](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20App%20Router%20Fundamentals) · [Next.js Rendering & Caching](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Rendering%20%26%20Caching) · [Next.js 15 to 16.3](/Software%20Engineering/Frontend/07%20Next.js/Next.js%2015%20to%2016.3) · [Rendering Patterns - CSR, SSR, SSG, ISR, Streaming & Islands](/Software%20Engineering/Frontend/09%20Architecture/Rendering%20Patterns%20-%20CSR%2C%20SSR%2C%20SSG%2C%20ISR%2C%20Streaming%20%26%20Islands)
- [Kubernetes Workloads in Production](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Workloads%20in%20Production) · [Kubernetes Networking - Services, Ingress & Gateway API](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Networking%20-%20Services%2C%20Ingress%20%26%20Gateway%20API) · [Helm & Kustomize](/Software%20Engineering/DevOps/Kubernetes/Helm%20%26%20Kustomize) · [Kubernetes Operations - Logs, Debugging & Scheduling](/Software%20Engineering/DevOps/Delivery%20Platform/Kubernetes%20Operations%20-%20Logs%2C%20Debugging%20%26%20Scheduling)
- [Argo CD in Practice](/Software%20Engineering/DevOps/Delivery%20Platform/Argo%20CD%20in%20Practice) · [Argo Rollouts - Canary & Blue-Green](/Software%20Engineering/DevOps/Delivery%20Platform/Argo%20Rollouts%20-%20Canary%20%26%20Blue-Green) · [Jenkins - Pipelines as Code](/Software%20Engineering/DevOps/Delivery%20Platform/Jenkins%20-%20Pipelines%20as%20Code) · [Grafana & the LGTM Stack](/Software%20Engineering/DevOps/Delivery%20Platform/Grafana%20%26%20the%20LGTM%20Stack)
- [Dockerfile Best Practices & Image Security](/Software%20Engineering/DevOps/Docker/Dockerfile%20Best%20Practices%20%26%20Image%20Security) · [Authentication & Token Storage](/Software%20Engineering/Frontend/11%20Security/Authentication%20%26%20Token%20Storage) · [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time) · [Frontend System Design Index](/Software%20Engineering/Frontend/15%20System%20Design/Frontend%20System%20Design%20Index)

## Sources

- Next.js: Multi-Zones guide (docs 16.3.6): https://nextjs.org/docs/app/guides/multi-zones
- Next.js: Self-hosting guide (caching, deploymentId, encryption key, streaming, `after`): https://nextjs.org/docs/app/guides/self-hosting
- Next.js: `output: 'standalone'`: https://nextjs.org/docs/app/api-reference/config/next-config-js/output
- Next.js: `basePath`: https://nextjs.org/docs/app/api-reference/config/next-config-js/basePath
- Next.js: `assetPrefix`: https://nextjs.org/docs/app/api-reference/config/next-config-js/assetPrefix
- Next.js: `deploymentId` (version history incl. 16.2.0): https://nextjs.org/docs/app/api-reference/config/next-config-js/deploymentId
- Next.js: `cacheHandlers` (introduced in 16.0.0): https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheHandlers
- Next.js: custom `cacheHandler` (incremental cache): https://nextjs.org/docs/app/api-reference/config/next-config-js/incrementalCacheHandlerPath
- Next.js Redis cache handler example: https://github.com/vercel/next.js/tree/canary/examples/cache-handler-redis
- Next.js: Image component (`maximumDiskCacheSize` 16.1.7, `minimumCacheTTL`): https://nextjs.org/docs/app/api-reference/components/image
- Next.js: Deploying (Docker templates, adapters): https://nextjs.org/docs/app/getting-started/deploying
- Next.js `with-docker` example Dockerfile: https://github.com/vercel/next.js/tree/canary/examples/with-docker
- Next.js 15 release blog (automatic sharp for standalone): https://nextjs.org/blog/next-15
- Next.js: OpenTelemetry guide: https://nextjs.org/docs/app/guides/open-telemetry
- Next.js: Route Handlers: https://nextjs.org/docs/app/api-reference/file-conventions/route
- Module Federation: "Next.js Support is in maintenance mode" (issue #3153, 2024-11-01): https://github.com/module-federation/core/issues/3153
- Module Federation `nextjs-mf` package: https://github.com/module-federation/core/tree/main/packages/nextjs-mf
- Vercel Microfrontends docs: https://vercel.com/docs/microfrontends
- Vercel changelog, "Microfrontends now generally available" (2025-10-31): https://vercel.com/changelog/microfrontends-now-generally-available
- Gateway API HTTPRoute (path matching, URLRewrite): https://gateway-api.sigs.k8s.io/api-types/httproute/
- Argo CD ApplicationSet generators (Git, Matrix): https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators/
- Grafana Faro Web SDK: https://github.com/grafana/faro-web-sdk
- Grafana Alloy `faro.receiver`: https://grafana.com/docs/alloy/latest/reference/components/faro/faro.receiver/
- MDN: cookie prefixes (`__Host-`): https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies#cookie_prefixes
