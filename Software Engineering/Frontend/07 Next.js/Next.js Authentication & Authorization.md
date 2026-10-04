---
title: "Next.js Authentication & Authorization"
tags: [frontend, frontend/nextjs, frontend/security, auth]
aliases: [Next.js Auth, Next.js Authentication, Next.js RBAC, Next.js OIDC, Better Auth, Auth.js, NextAuth, CVE-2025-29927, forbidden(), unauthorized()]
updated: 2026-09-27
---
# Next.js Authentication & Authorization

<Note>
**TL;DR**

Auth in Next.js has three parts: **authentication** (who are you: OIDC, passkeys, password), **session management** (a signed/encrypted cookie, or a cookie holding an opaque ID for a DB session) and **authorization** (what can you do). `proxy.ts` is only for **optimistic** checks (cookie present? redirect to `/login`); the **real** checks belong in a `server-only` Data Access Layer called by pages, Server Actions and Route Handlers. The March 2025 middleware bypass (CVE-2025-29927) is the reason. In 2026 the usual choices are **Better Auth** (Auth.js/NextAuth moved into its care and maintenance mode in Sept 2025), a hosted provider such as **Clerk/Auth0/WorkOS**, or **custom OIDC** with `openid-client` when you already run an IdP. `forbidden()`/`unauthorized()` exist but are still **experimental** (`authInterrupts`).
</Note>

General theory lives elsewhere: [Authentication vs Authorization](/Software%20Engineering/APIs%20%26%20Auth/Authentication%20vs%20Authorization), [OAuth 2.x & OpenID Connect](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect), [JWT vs Cookies](/Software%20Engineering/APIs%20%26%20Auth/JWT%20vs%20Cookies), [Authentication & Token Storage](/Software%20Engineering/Frontend/11%20Security/Authentication%20%26%20Token%20Storage), [Passkeys, WebAuthn & MFA](/Software%20Engineering/APIs%20%26%20Auth/Passkeys%2C%20WebAuthn%20%26%20MFA). This note is about doing it correctly in the App Router.

## The mental model

{/* diagram source: /diagrams/diagram-283.mmd */}
![diagram](/diagrams/diagram-283.svg)

Every entry point (page render, action, route handler) reaches data only through the DAL, and the DAL checks every time. Proxy is a convenience and a performance filter, not a security boundary.

Why several entry points matter: a page and the Server Actions it renders are **separate requests**. Layouts don't re-render on client navigation and don't stop child segments from rendering. A proxy `matcher` can skip paths (and therefore the actions posted to them).

## Session strategies

| | Stateless (token in cookie) | Database session (opaque ID in cookie) |
|---|---|---|
| Cookie holds | Signed (JWS) or encrypted (JWE) payload: user ID, role, expiry | Random session ID (store a hash of it in the DB) |
| Validation | Verify signature/decrypt, no DB hit | DB/Redis lookup per request (cache per request with `React.cache`) |
| Revocation / "log out everywhere" | Hard: valid until expiry (short TTL + refresh, or a denylist) | Easy: delete the row |
| Role changes | Stale until token refresh | Immediate |
| Size | Grows with claims (4 KB cookie limit) | Tiny |
| Infra | None | A store (Postgres, Redis) |
| Good for | Simple apps, edge-heavy setups | Internal tools, B2B, anything with admin revocation or compliance needs |

The Next.js guide shows both, recommends a session library (`iron-session` or `jose`) rather than hand-rolled crypto, and recommends these cookie options: `HttpOnly`, `Secure`, `SameSite` (`lax` is a sensible default), `Max-Age`/`Expires`, `Path=/`. Set cookies only on the server. Keep the payload minimal (IDs and roles, no PII).

<Tip>
**Cookie naming**

Prefix the cookie with `__Host-` (e.g. `__Host-session`). Browsers then require `Secure`, `Path=/` and no `Domain`, which blocks subdomain cookie injection. Browsers treat `http://localhost` as secure, so this works in development too.
</Tip>

"JWT vs cookies" is a false choice: a JWT can live in a cookie. The real questions are where the token is stored (httpOnly cookie, not `localStorage`) and whether you need server-side revocation ([JWT vs Cookies](/Software%20Engineering/APIs%20%26%20Auth/JWT%20vs%20Cookies)).

## Libraries in 2026

The Next.js authentication guide lists (alphabetically): Auth0, Better Auth, Clerk, Descope, Kinde, Logto, NextAuth.js (Auth.js), Ory, Stack Auth, Supabase, Stytch, WorkOS; and for sessions iron-session and jose. It recommends a library over a custom implementation.

| Option | Model | Status (Sept 2026) | Choose when |
|---|---|---|---|
| **Better Auth** | Open-source library, your DB, plugins (OAuth/OIDC via `genericOAuth`, organizations, 2FA, passkeys, RBAC) | Actively developed. Took over Auth.js maintenance on **2025-09-22** and recommends new projects start on Better Auth | You want to own user data and sessions without writing crypto |
| **Auth.js / NextAuth.js** | Open-source library, OAuth providers, JWT or DB sessions | **Maintenance mode**: security patches and urgent fixes only; the main contributor left in Jan 2025 | Existing apps; migrate when convenient |
| **Clerk** (also Auth0, WorkOS, Stytch, Kinde, Descope) | Hosted identity, prebuilt UI, Next SDK with a proxy helper and `auth()` on the server | Commercial SaaS | Speed, B2B SSO/SCIM, you'd rather not run identity |
| **Custom OIDC** (`openid-client` v6) | Your Next app is an OIDC client/BFF against an IdP you run (Keycloak, Entra ID, Okta, Zitadel) | `openid-client` v6 is ESM-only, Node 20+ | Enterprise/internal apps with an existing IdP; you want tokens only on the server |
| Hand-rolled password auth | `bcrypt`/`argon2` + sessions | Discouraged | Learning only |

Better Auth essentials (verified against its Next.js docs):

```ts
// src/server/auth/better-auth.ts
import "server-only";
import { betterAuth } from "better-auth";
import { nextCookies } from "better-auth/next-js";
import { genericOAuth } from "better-auth/plugins";

export const auth = betterAuth({
  database: /* your adapter, e.g. a pg Pool or Drizzle/Prisma adapter */ undefined as never,
  plugins: [
    genericOAuth({
      config: [{
        providerId: "acme-sso",
        clientId: process.env.OIDC_CLIENT_ID!,
        clientSecret: process.env.OIDC_CLIENT_SECRET!,
        discoveryUrl: `${process.env.OIDC_ISSUER}/.well-known/openid-configuration`,
        scopes: ["openid", "email", "profile"],
      }],
    }),
    nextCookies(),     // lets Server Actions that call auth APIs set cookies
  ],
});

// app/api/auth/[...all]/route.ts
//   import { toNextJsHandler } from "better-auth/next-js";
//   export const { GET, POST } = toNextJsHandler(auth);

// In a Server Component / action / DAL:
//   const session = await auth.api.getSession({ headers: await headers() });
// Client: authClient.signIn.social({ provider: "acme-sso", callbackURL: "/jobs" })
```

## Where checks go

| Layer | Check type | What to do | What not to do |
|---|---|---|---|
| `proxy.ts` | **Optimistic** | Read the cookie, redirect unauthenticated users away from protected sections, redirect logged-in users away from `/login`, add security headers | DB queries (it runs on every request including prefetches), full authorization, being the only check |
| Layout | UX | Load the user for the nav (`getUser()` via DAL); wrap in Suspense | Rely on it to protect children or actions |
| Page / Server Component | **Secure** | `requireViewer()`/`requireRole()` from the DAL; render role-specific UI | Trust `searchParams` like `?admin=true` |
| DAL functions | **Secure** | Verify session, scope queries by user/tenant, return DTOs | Return raw rows |
| Server Action | **Secure** | Re-verify session + permission for *this* resource, validate input | Assume the page check covered it |
| Route Handler | **Secure** | 401 if no session, 403 if not permitted | Assume proxy covered it |
| Client Component | Cosmetic | Hide buttons the user can't use | Consider hidden = protected |

### `proxy.ts`: optimistic only

```ts
// src/proxy.ts
import { NextResponse, type NextRequest } from "next/server";

const PUBLIC = ["/login", "/api/auth/login", "/api/auth/callback", "/api/health"];

export function proxy(req: NextRequest) {
  const { pathname } = req.nextUrl;
  const hasSession = req.cookies.has("__Host-session");     // presence only, no DB
  const isPublic = PUBLIC.some((p) => pathname === p || pathname.startsWith(`${p}/`));

  if (!hasSession && !isPublic) {
    const url = new URL("/login", req.url);
    url.searchParams.set("next", pathname);                  // validate on use (open redirect!)
    return NextResponse.redirect(url);
  }
  if (hasSession && pathname === "/login") {
    return NextResponse.redirect(new URL("/jobs", req.url));
  }
  return NextResponse.next();
}

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico|robots.txt).*)"],
};
```

Proxy runs on the Node.js runtime in Next 16 (the `runtime` option isn't allowed in proxy files). It can technically query a DB, but the docs advise against it because it runs for every matched request, including prefetches.

## The CVE-2025-29927 lesson (March 2025)

- **What:** Next.js used an internal header, `x-middleware-subrequest`, to stop middleware recursion. An attacker could send it from outside and make the server **skip middleware entirely**.
- **Timeline:** reported 2025-02-27; patched releases 14.2.25 (Mar 17), 15.2.3 (Mar 18), 13.5.9 (Mar 22), 12.3.5 (Mar 23); CVE published Mar 18 and made public Mar 21, 2025.
- **Who:** self-hosted apps (`next start`, `output: 'standalone'`) that relied on middleware for auth. Vercel, Netlify and Cloudflare Workers deployments were not affected because their routing is decoupled.
- **Workaround** if you couldn't upgrade: strip the header at the load balancer/reverse proxy.

The lessons, which also explain the rename to `proxy.ts` in Next 16:
1. **Defense in depth.** Authorization must be enforced next to the data (DAL), so bypassing one layer doesn't expose anything.
2. **Strip internal headers at the edge.** Your ingress should drop framework-internal and hop-by-hop headers from untrusted clients ([API Gateways & Reverse Proxies](/Software%20Engineering/Backend/API%20Gateways%20%26%20Reverse%20Proxies)).
3. **Patch fast.** Subscribe to Next.js/React advisories; automate dependency updates ([Supply Chain Security](/Software%20Engineering/Frontend/11%20Security/Supply%20Chain%20Security)).

## RBAC in Server Components and Actions

Model permissions, not just roles. Roles map to permissions; code checks permissions.

```ts
// src/server/auth/permissions.ts
import "server-only";

export const ROLE_PERMISSIONS = {
  viewer:   ["ops:jobs:read"],
  operator: ["ops:jobs:read", "ops:jobs:write"],
  admin:    ["ops:jobs:read", "ops:jobs:write", "ops:audit:read"],
} as const;

export type Role = keyof typeof ROLE_PERMISSIONS;
export type Permission = (typeof ROLE_PERMISSIONS)[Role][number];

export const can = (roles: readonly Role[], p: Permission) =>
  roles.some((r) => (ROLE_PERMISSIONS[r] as readonly string[]).includes(p));
```

In a Server Component:

```tsx
const viewer = await requireViewer();
return (
  <JobDetail job={job}>
    {can(viewer.roles, "ops:jobs:write") && <RunNowButton jobId={job.id} />}  {/* UI only */}
  </JobDetail>
);
```

In a Server Action: call `requirePermission("ops:jobs:write")` first (full example below). For resource-level rules (ownership, tenant), check in the query itself (`WHERE id = $1 AND org_id = $2`). For complex policies, consider a policy engine (OPA/Cedar/OpenFGA) behind the DAL.

## `forbidden()` and `unauthorized()` (experimental)

- Introduced in **15.1** (Dec 2024); still marked **experimental** in the 16.3.6 docs. Enable with `experimental: { authInterrupts: true }`.
- `unauthorized()` renders `unauthorized.tsx` (401), `forbidden()` renders `forbidden.tsx` (403). Import from `next/navigation`. Callable in Server Components, Server Functions and Route Handlers; not in the root layout.
- They work by **throwing**, like `notFound()`/`redirect()`: don't wrap them in `try/catch` (or use `unstable_rethrow`), and always `await` the function that may call them.
- Status code caveat: if the check runs inside a `<Suspense>` boundary, the response already started streaming with 200, so the user sees the 403 UI but the HTTP status is 200. For a real 403 status under Cache Components (which always streams a shell first), do the check in `proxy`.

```tsx
// app/(console)/audit/page.tsx
import { forbidden } from "next/navigation";
import { requireViewer } from "@/server/auth/session";
import { can } from "@/server/auth/permissions";

export default async function AuditPage() {
  const viewer = await requireViewer();
  if (!can(viewer.roles, "ops:audit:read")) forbidden();
  return <AuditLog />;
}
// app/(console)/audit/forbidden.tsx -> "You need the auditor role"
```

Without the flag, use `redirect('/login')` for unauthenticated and `notFound()` (hide existence) or an inline "no access" UI for unauthorized.

## CSRF and Server Actions

- **Server Actions:** only `POST`; Next compares `Origin` with `Host`/`X-Forwarded-Host` and rejects mismatches. Combined with `SameSite=Lax` cookies this blocks classic CSRF. Behind a proxy/CDN on another host, list it in `serverActions.allowedOrigins`. Make sure your reverse proxy sets `X-Forwarded-Host` correctly and doesn't let clients spoof it.
- **Route Handlers:** **no automatic CSRF protection.** For cookie-authenticated, state-changing handlers, check `Origin` yourself (or require a custom header / CSRF token), keep `SameSite=Lax` or `Strict`, and never change state on `GET`.
- **Rendering must not mutate.** Next blocks setting cookies and revalidating during render; log out through an action, not `?logout=1`.
- More: [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types).

## Full example: OIDC login, DB session, protected layout, role-gated action

The `ops-console` (see [Job Scheduler Dashboard with Next.js](/Software%20Engineering/Backend/Scheduled%20Jobs/Job%20Scheduler%20Dashboard%20with%20Next.js)) logs in against the company IdP with OIDC Authorization Code + PKCE. Tokens never reach the browser: the app is a **BFF** that stores tokens in a server-side session and calls `jobs-api` with the access token ([Next.js Route Handlers & Backend-for-Frontend](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Route%20Handlers%20%26%20Backend-for-Frontend)).

```text
sequenceDiagram
  participant U as Browser
  participant N as ops-console (Next.js)
  participant I as IdP (OIDC)
  participant S as Session store (Postgres)
  U->>N: GET /api/auth/login?next=/jobs
  N->>N: PKCE verifier, state, nonce -> short-lived httpOnly cookie
  N-->>U: 302 to IdP /authorize (code_challenge, state)
  U->>I: login + consent
  I-->>U: 302 /api/auth/callback?code&state
  U->>N: GET /api/auth/callback
  N->>I: POST /token (code, code_verifier)
  I-->>N: id_token, access_token, refresh_token
  N->>S: INSERT session (hash(id), user, roles, tokens, expiry)
  N-->>U: Set-Cookie __Host-session=<random id>; 302 /jobs
  U->>N: GET /jobs (cookie)
  N->>S: lookup session (React.cache per request)
  N-->>U: HTML
```

### 1. OIDC client

```ts
// src/server/auth/oidc.ts
import "server-only";
import * as client from "openid-client";
import { env } from "@/env";

let configPromise: Promise<client.Configuration> | undefined;

export function oidc() {
  configPromise ??= client.discovery(new URL(env.OIDC_ISSUER), env.OIDC_CLIENT_ID, env.OIDC_CLIENT_SECRET);
  return configPromise;
}
```

### 2. Login route: start the flow

```ts
// src/app/api/auth/login/route.ts
import { NextResponse, type NextRequest } from "next/server";
import * as client from "openid-client";
import { oidc } from "@/server/auth/oidc";
import { sealFlow } from "@/server/auth/flow-cookie";   // encrypts {verifier,state,nonce,next} with jose

export async function GET(req: NextRequest) {
  const config = await oidc();
  const code_verifier = client.randomPKCECodeVerifier();
  const code_challenge = await client.calculatePKCECodeChallenge(code_verifier);
  const state = client.randomState();
  const nonce = client.randomNonce();
  const next = safeNext(req.nextUrl.searchParams.get("next"));

  const url = client.buildAuthorizationUrl(config, {
    redirect_uri: new URL("/api/auth/callback", req.url).toString(),
    scope: "openid profile email offline_access",
    code_challenge,
    code_challenge_method: "S256",
    state,
    nonce,
  });

  const res = NextResponse.redirect(url);
  res.cookies.set("__Host-oidc-flow", await sealFlow({ code_verifier, state, nonce, next }), {
    httpOnly: true, secure: true, sameSite: "lax", path: "/", maxAge: 600,
  });
  return res;
}

// Only allow same-origin relative paths (prevents open redirects)
function safeNext(v: string | null) {
  return v && v.startsWith("/") && !v.startsWith("//") ? v : "/jobs";
}
```

### 3. Callback route: exchange the code, create the session

```ts
// src/app/api/auth/callback/route.ts
import { NextResponse, type NextRequest } from "next/server";
import * as client from "openid-client";
import { oidc } from "@/server/auth/oidc";
import { openFlow } from "@/server/auth/flow-cookie";
import { createSession } from "@/server/auth/session";

export async function GET(req: NextRequest) {
  const flow = await openFlow(req.cookies.get("__Host-oidc-flow")?.value);
  if (!flow) return new Response("Login expired, please retry", { status: 400 });

  const config = await oidc();
  const tokens = await client.authorizationCodeGrant(config, new URL(req.url), {
    pkceCodeVerifier: flow.code_verifier,
    expectedState: flow.state,
    expectedNonce: flow.nonce,
  });
  const claims = tokens.claims();                       // validated ID token claims
  if (!claims) return new Response("Missing ID token", { status: 400 });

  const roles = mapGroupsToRoles(claims["groups"]);     // e.g. IdP group "ops-operators" -> "operator"
  const sessionId = await createSession({
    userId: claims.sub,
    email: String(claims.email ?? ""),
    roles,
    accessToken: tokens.access_token,
    refreshToken: tokens.refresh_token,
    idToken: tokens.id_token,
    accessTokenExpiresAt: new Date(Date.now() + (tokens.expiresIn() ?? 300) * 1000),
  });

  const res = NextResponse.redirect(new URL(flow.next, req.url));
  res.cookies.delete("__Host-oidc-flow");
  res.cookies.set("__Host-session", sessionId, {
    httpOnly: true, secure: true, sameSite: "lax", path: "/", maxAge: 60 * 60 * 8,  // 8h workday session
  });
  return res;
}

function mapGroupsToRoles(groups: unknown): ("viewer" | "operator" | "admin")[] {
  const g = Array.isArray(groups) ? groups.map(String) : [];
  if (g.includes("ops-admins")) return ["admin"];
  if (g.includes("ops-operators")) return ["operator"];
  return ["viewer"];
}
```

### 4. Session + DAL helpers

```ts
// src/server/auth/session.ts
import "server-only";
import { cache } from "react";
import { cookies } from "next/headers";
import { redirect, forbidden } from "next/navigation";
import { createHash, randomBytes } from "node:crypto";
import { db } from "@/server/db";
import { can, type Permission, type Role } from "./permissions";

const hash = (id: string) => createHash("sha256").update(id).digest("hex");

export async function createSession(data: {
  userId: string; email: string; roles: Role[];
  accessToken: string; refreshToken?: string; idToken?: string; accessTokenExpiresAt: Date;
}) {
  const id = randomBytes(32).toString("base64url");
  await db.session.create({
    data: { idHash: hash(id), ...data, expiresAt: new Date(Date.now() + 8 * 3600_000) },
    // store tokens encrypted at rest in real deployments
  });
  return id;
}

export type Viewer = { userId: string; email: string; roles: Role[]; sessionIdHash: string };

// One DB lookup per request, however many components call it
export const getViewer = cache(async (): Promise<Viewer | null> => {
  const id = (await cookies()).get("__Host-session")?.value;
  if (!id) return null;
  const s = await db.session.findUnique({ where: { idHash: hash(id) } });
  if (!s || s.expiresAt < new Date()) return null;
  return { userId: s.userId, email: s.email, roles: s.roles as Role[], sessionIdHash: s.idHash };
});

export async function requireViewer() {
  const v = await getViewer();
  if (!v) redirect("/login");
  return v;
}

export async function requirePermission(p: Permission) {
  const v = await requireViewer();
  if (!can(v.roles, p)) forbidden();          // needs authInterrupts; else throw new Error("Forbidden")
  return v;
}
```

### 5. Protected layout (UX) with streaming

```tsx
// src/app/(console)/layout.tsx
import { Suspense } from "react";
import { UserMenu } from "@/features/auth/components/UserMenu";

export default function ConsoleLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="shell">
      <nav>
        <a href="/jobs">Jobs</a>
        <Suspense fallback={<span className="avatar-skeleton" />}>
          <UserMenu />            {/* awaits getViewer(); doesn't block the shell */}
        </Suspense>
      </nav>
      <main>{children}</main>     {/* pages still call requireViewer()/requirePermission() */}
    </div>
  );
}
```

The layout makes the experience nice; it is not what protects `/jobs`. Proxy redirects anonymous users early, and every page, action and handler calls the DAL.

### 6. Role-gated Server Action

```ts
// src/features/jobs/actions.ts
"use server";
import { z } from "zod";
import { updateTag } from "next/cache";
import { requirePermission } from "@/server/auth/session";
import { jobsApi } from "@/server/jobs-api";          // BFF client: adds the user's access token
import { audit } from "@/server/audit";

const StopRunInput = z.object({ jobId: z.string().min(1), runId: z.string().min(1) });

export async function stopRunAction(input: { jobId: string; runId: string }) {
  const viewer = await requirePermission("ops:jobs:write");   // authn + authz, every call
  const { jobId, runId } = StopRunInput.parse(input);         // validate

  const res = await jobsApi(viewer).post(`/api/jobs/${encodeURIComponent(jobId)}/runs/${encodeURIComponent(runId)}/stop`);
  if (res.status === 404) return { ok: false as const, message: "Run not found" };
  if (!res.ok) throw new Error(`jobs-api ${res.status}`);     // unexpected -> error boundary

  await audit({ actor: viewer.userId, action: "job.run.stop", target: `${jobId}/${runId}` });
  updateTag(`job:${jobId}`);
  return { ok: true as const };
}
```

`jobs-api` also checks the bearer token's `ops:jobs:write` scope/role: defense in depth across services.

### 7. Logout

```ts
// src/features/auth/actions.ts
"use server";
import { cookies } from "next/headers";
import { redirect } from "next/navigation";
import * as client from "openid-client";
import { oidc } from "@/server/auth/oidc";
import { getViewer, deleteSession, getIdToken } from "@/server/auth/session";

export async function logout() {
  const viewer = await getViewer();
  const idToken = viewer ? await getIdToken(viewer.sessionIdHash) : undefined;
  if (viewer) await deleteSession(viewer.sessionIdHash);        // server-side revocation
  (await cookies()).delete("__Host-session");
  const endSession = client.buildEndSessionUrl(await oidc(), {
    id_token_hint: idToken ?? "",
    post_logout_redirect_uri: "https://ops.acme.dev/login",
  });
  redirect(endSession.href);                                      // RP-initiated logout at the IdP
}
```

## Common pitfalls

<Warning>
**Pitfalls**

- Auth only in `proxy.ts`/middleware (the CVE-2025-29927 failure mode), or only in a layout.
- `return null` in a layout for unauthorized users, SPA-style. Child segments and actions are still reachable.
- Checking authentication but not authorization per resource (IDOR): "logged in" is not "owns this job".
- Tokens in `localStorage` or in a non-httpOnly cookie.
- PII or large claims in a stateless session cookie (4 KB limit, and it's readable if only signed).
- Open redirects via `?next=` / `callbackUrl`.
- DB queries in proxy for every request, including prefetches.
- `forbidden()`/`redirect()` inside `try/catch`.
- Caching per-user data with `'use cache'` under a key that doesn't include the user or tenant.
- Starting a new project on Auth.js in 2026 without considering its maintenance status.
</Warning>

## Senior perspective

- **Threat model first:** who are the actors, what are the assets, which entry points exist (pages, actions, route handlers, webhooks, proxy)? Then map each to a DAL check. Pen-test actions like any API.
- **Session policy is a product decision:** idle vs absolute timeout, step-up auth (re-authenticate before destructive actions), "sign out all devices", session listing. DB sessions make these easy.
- **Token handling in a BFF:** refresh access tokens server-side before expiry (with a lock to avoid parallel refreshes), encrypt tokens at rest, never forward them to the browser, and scope audience per downstream API.
- **Multi-instance deploys:** share session storage, `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` and signing keys across instances; plan key rotation (support two keys during rollover).
- **Auditability:** log security-relevant events (login, logout, role changes, destructive actions) with actor, target and request ID; ship to your log pipeline ([Observability - Logs, Metrics & Traces](/Software%20Engineering/DevOps/Observability%20-%20Logs%2C%20Metrics%20%26%20Traces)).
- **Caching and auth interact:** with Cache Components, read cookies outside `'use cache'` and pass IDs in; Next has a dedicated guide on authentication with Cache Components. A shared static route protected only in proxy is fine for paywalled-but-identical content, never for per-user data.
- Questions in review: "Which DAL function enforces this?", "What does an unauthenticated POST to this action do?", "How is a revoked user kicked out, and how fast?", "Which headers does our ingress strip?"

<Info>
**🆕 2025–2026**

- **15.1 (Dec 2024)**: experimental `forbidden()` / `unauthorized()` (`authInterrupts`); still experimental in the 16.3.6 docs (Sept 2026).
- **2025-03-21**: **CVE-2025-29927** middleware bypass made public (patched 15.2.3 / 14.2.25 / 13.5.9 / 12.3.5).
- **15.5 (Aug 18, 2025)**: Node.js runtime for middleware stable (full Node crypto/DB drivers available).
- **2025-09-22**: **Auth.js joins Better Auth**; Auth.js in maintenance mode, new projects advised to use Better Auth.
- **16.0 (Oct 21, 2025)**: `middleware.ts` renamed to **`proxy.ts`**, Node.js runtime by default; the docs call proxy a last resort and position it for optimistic checks.
- **2025-12-03**: RSC Server Function RCE (CVE-2025-55182): another reminder that every action is an unauthenticated entry point until your code says otherwise.
</Info>

## Interview questions

<Accordion title="Why shouldn't you rely on middleware/proxy for authorization?">
It's a single layer that can be bypassed (CVE-2025-29927 let attackers skip it with a header), its matcher may not cover every path or Server Action, and it runs before you know which data will be accessed. Use it for optimistic redirects; enforce authn/authz in a DAL called by every page, action and route handler.
</Accordion>

<Accordion title="Stateless JWT sessions vs database sessions?">
Stateless: signed/encrypted payload in a cookie, no DB hit, but hard to revoke and stale on role changes. Database: opaque ID in a cookie, lookup per request (cacheable per request), easy revocation and immediate role changes. Many production apps use DB sessions for internal/B2B tools and short-lived tokens with refresh where scale or edge execution matter.
</Accordion>

<Accordion title="How do you implement RBAC in the App Router?">
Map roles to permissions in a server-only module; a DAL helper like `requirePermission(p)` verifies the session and permission and redirects/throws (`forbidden()` if `authInterrupts` is on). Call it in pages for role-specific UI and in every Server Action/Route Handler before mutating. Resource-level rules go into the query (`WHERE id AND org_id`). Client-side hiding is cosmetic only.
</Accordion>

<Accordion title="How does Next.js protect Server Actions from CSRF, and what's left to you?">
Actions are POST-only and Next rejects requests whose `Origin` doesn't match `Host`/`X-Forwarded-Host` (configurable via `serverActions.allowedOrigins`); SameSite cookies add protection. You still must authenticate and authorize inside each action. Route Handlers get no automatic CSRF check, so verify `Origin` or use tokens for cookie-authenticated mutations.
</Accordion>

<Accordion title="What happened to NextAuth/Auth.js?">
In Sept 2025 Auth.js became part of Better Auth. Auth.js is in maintenance mode (security and urgent fixes), and the maintainers recommend Better Auth for new projects unless you need something it lacks, such as stateless DB-less sessions.
</Accordion>

<Accordion title="Why is checking auth in a layout not enough?">
Layouts don't re-render on client-side navigation between their children, so the check doesn't re-run; and a layout doesn't control whether child segments or parallel slots render or whether actions posted to those routes execute. Check near the data instead.
</Accordion>

<Accordion title="Describe a secure OIDC login in a Next.js BFF.">
Authorization Code + PKCE, with state and nonce stored in a short-lived encrypted httpOnly cookie. The callback route exchanges the code server-side, validates the ID token, creates a server-side session holding tokens, and sets an httpOnly `__Host-` session cookie with an opaque ID. The browser never sees access tokens; server code calls APIs with them and refreshes them. Logout deletes the session and redirects to the IdP's end-session endpoint.
</Accordion>

## Related
- [Next.js Index](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Index)
- [Next.js Project Structure & Architecture](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Project%20Structure%20%26%20Architecture) · [Next.js Data Fetching & Mutations Patterns](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Data%20Fetching%20%26%20Mutations%20Patterns) · [Next.js Route Handlers & Backend-for-Frontend](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Route%20Handlers%20%26%20Backend-for-Frontend) · [Next.js Error Handling, Loading & Not-Found](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Error%20Handling%2C%20Loading%20%26%20Not-Found)
- [Next.js 15 to 16.3](/Software%20Engineering/Frontend/07%20Next.js/Next.js%2015%20to%2016.3) · [Next.js App Router Fundamentals](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20App%20Router%20Fundamentals)
- [Authentication vs Authorization](/Software%20Engineering/APIs%20%26%20Auth/Authentication%20vs%20Authorization) · [OAuth 2.x & OpenID Connect](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect) · [JWT vs Cookies](/Software%20Engineering/APIs%20%26%20Auth/JWT%20vs%20Cookies) · [Passkeys, WebAuthn & MFA](/Software%20Engineering/APIs%20%26%20Auth/Passkeys%2C%20WebAuthn%20%26%20MFA)
- [Authentication & Token Storage](/Software%20Engineering/Frontend/11%20Security/Authentication%20%26%20Token%20Storage) · [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types) · [Supply Chain Security](/Software%20Engineering/Frontend/11%20Security/Supply%20Chain%20Security)
- [API Gateways & Reverse Proxies](/Software%20Engineering/Backend/API%20Gateways%20%26%20Reverse%20Proxies) · [Job Scheduler Dashboard with Next.js](/Software%20Engineering/Backend/Scheduled%20Jobs/Job%20Scheduler%20Dashboard%20with%20Next.js)

## Sources
- [Next.js docs: How to implement authentication](https://nextjs.org/docs/app/guides/authentication) (v16.3.6: sessions, DAL, optimistic proxy checks, auth libraries list)
- [Next.js docs: Data security](https://nextjs.org/docs/app/guides/data-security) (allowed origins, IDOR, audit checklist)
- [Next.js docs: forbidden](https://nextjs.org/docs/app/api-reference/functions/forbidden), [unauthorized](https://nextjs.org/docs/app/api-reference/functions/unauthorized), [authInterrupts](https://nextjs.org/docs/app/api-reference/config/next-config-js/authInterrupts)
- [Next.js docs: proxy.js](https://nextjs.org/docs/app/api-reference/file-conventions/proxy) (runtime, migration from middleware)
- [Vercel: Postmortem on Next.js middleware bypass (CVE-2025-29927)](https://vercel.com/blog/postmortem-on-next-js-middleware-bypass), Mar 25, 2025
- [Better Auth: Auth.js is now part of Better Auth](https://better-auth.com/blog/authjs-joins-better-auth), Sept 22, 2025
- [Better Auth: Next.js integration](https://www.better-auth.com/docs/integrations/next), [Generic OAuth plugin](https://www.better-auth.com/docs/plugins/generic-oauth)
- [openid-client (panva) README](https://github.com/panva/openid-client) (v6 API)
- [MDN: Set-Cookie, cookie prefixes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie)
- [OWASP: Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html), [IDOR Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html)
- [RFC 9700: OAuth 2.0 Security Best Current Practice](https://www.rfc-editor.org/rfc/rfc9700)
- [The Copenhagen Book](https://thecopenhagenbook.com/)
