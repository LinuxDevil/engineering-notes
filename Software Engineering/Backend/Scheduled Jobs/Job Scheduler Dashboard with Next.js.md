---
title: "Job Scheduler Dashboard with Next.js"
tags: [backend, backend/jobs, frontend/nextjs]
aliases: [ops-console, Jobs Dashboard, Scheduled Jobs UI, Job Control Dashboard, Cron Dashboard Next.js]
updated: 2026-09-27
---
# Job Scheduler Dashboard with Next.js

<Note>
**TL;DR**

`ops-console` is a Next.js 16 App Router app at `ops.acme.dev` that lists every scheduled job in Acme Ops, shows run history and live logs, and lets operators **Pause, Resume, Run now and Stop** jobs, whether they run on BullMQ or as Kubernetes CronJobs. It is a **backend-for-frontend**: Server Components read from `jobs-api` with the user's access token through a server-only, zod-validated client; Server Actions do the mutations (with `useActionState` + `useOptimistic`, a fresh `Idempotency-Key` per click, and a role check in every action); Route Handlers proxy the `jobs-api` SSE streams so the browser gets live status without ever holding a token. Accessibility (status by text and icon, live regions, a real modal dialog for Stop) and failure states (skeletons, `error.tsx`, empty state, reconnecting streams) are part of the design, not polish.
</Note>

This note builds the UI on top of the API described in [Job Control API - Pause, Resume, Run Now & Stop](/Software%20Engineering/Backend/Scheduled%20Jobs/Job%20Control%20API%20-%20Pause%2C%20Resume%2C%20Run%20Now%20%26%20Stop). The architecture of the whole system is in [Cron with Message Queues - Architecture](/Software%20Engineering/Backend/Scheduled%20Jobs/Cron%20with%20Message%20Queues%20-%20Architecture); the BullMQ side in [BullMQ Job Schedulers in Practice](/Software%20Engineering/Backend/Scheduled%20Jobs/BullMQ%20Job%20Schedulers%20in%20Practice); the Kubernetes side in [Kubernetes CronJobs in Practice](/Software%20Engineering/Backend/Scheduled%20Jobs/Kubernetes%20CronJobs%20in%20Practice). Next.js basics live in [Next.js App Router Fundamentals](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20App%20Router%20Fundamentals), and the login flow is reused from [Next.js Authentication & Authorization](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Authentication%20%26%20Authorization).

## 1. What the dashboard must do

| Need | How `ops-console` does it |
|---|---|
| See all jobs at a glance | `/jobs` table: name, backend badge, schedule in words (cronstrue), timezone, state, next run, last run status |
| Investigate one job | `/jobs/[id]`: details, paginated run history, live log viewer for the selected run |
| Control jobs | Pause / Resume / Run now / Stop run, as Server Actions calling `jobs-api` |
| Trust what you see | Live updates via SSE; optimistic UI reconciled with the server state |
| Safety | OIDC login; `ops:jobs:write` required for any mutation, checked in every action and again by `jobs-api`; confirmation for Stop; idempotency keys; audit trail in `jobs-api` |
| Operability | Loading skeletons, error boundaries, empty states, health endpoint, runs on Kubernetes next to `jobs-api` |

The four jobs it manages (Acme Ops running example):

| id | schedule (Europe/Berlin) | backend | what |
|---|---|---|---|
| `nightly-sales-report` | `0 2 * * *` | BullMQ | CSV to S3, email link |
| `sync-inventory` | `*/15 * * * *` | BullMQ | supplier API to Postgres |
| `cleanup-expired-carts` | `0 * * * *` | K8s CronJob | delete carts older than 7 days |
| `playwright-checkout-smoke` | `*/10 * * * *` | K8s CronJob | synthetic checkout check ([Playwright Automation as a Cron Job](/Software%20Engineering/Backend/Scheduled%20Jobs/Playwright%20Automation%20as%20a%20Cron%20Job)) |

### Action semantics the UI must respect

The UI does not invent semantics; it mirrors the API's state machine:

| Button | API call | Meaning | Enabled when |
|---|---|---|---|
| **Pause** | `POST /api/jobs/:id/pause` → 204 | stop creating new scheduled runs; in-flight runs continue | `state === 'active'` |
| **Resume** | `POST /api/jobs/:id/resume` → 204 | re-enable the schedule; no backfill by default | `state === 'paused'` |
| **Run now** | `POST /api/jobs/:id/runs` + `Idempotency-Key` → 202 `{ runId }` | enqueue one manual run | no run is `queued`/`running` (overlap is `Forbid` by default) |
| **Stop** | `POST /api/jobs/:id/runs/:runId/stop` → 202 | cooperative cancel: `cancelling` then `cancelled` | the selected run is `queued` or `running` |

Pausing does not stop a running job, and stopping a run does not pause the schedule. Operators confuse these under pressure, so the UI labels them explicitly ("Pause schedule", "Stop this run").

## 2. Architecture

{/* diagram source: /diagrams/diagram-064.mmd */}
![diagram](/diagrams/diagram-064.svg)

- **Reads**: Server Components call `jobs-api` directly from the server. No client-side data fetching library is needed.
- **Writes**: Server Actions. They are public POST endpoints, so each one authenticates, authorizes and validates.
- **Live**: two Route Handlers proxy SSE from `jobs-api`. The browser uses same-origin `EventSource` with its session cookie; the proxy adds the bearer token.
- **Why a BFF**: the access token stays on the server (no XSS token theft), `jobs-api` is not exposed to the internet, and CORS disappears because everything is same-origin. See [Next.js Route Handlers & Backend-for-Frontend](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Route%20Handlers%20%26%20Backend-for-Frontend).

### Wireframe

```text
/jobs
 Acme Ops ▸ Jobs                                        live ● connected    ana@acme [Sign out]

 Scheduled jobs
 ┌───────────────────────┬─────────┬──────────────────┬─────────────┬──────────┬──────────────┬─────────────┬────────────────────────────┐
 │ Job                   │ Backend │ Schedule         │ Timezone    │ State    │ Next run     │ Last run    │ Actions                    │
 ├───────────────────────┼─────────┼──────────────────┼─────────────┼──────────┼──────────────┼─────────────┼────────────────────────────┤
 │ nightly-sales-report  │ BullMQ  │ At 02:00         │ Europe/Berl │ ▶ Active │ 28 Sep 02:00 │ ✓ Succeeded │ [Pause] [Run now] [Stop]░  │
 │ sync-inventory        │ BullMQ  │ Every 15 minutes │ Europe/Berl │ ▶ Active │ 27 Sep 14:45 │ ◌ Running   │ [Pause] [Run now]░ [Stop]  │
 │ cleanup-expired-carts │ K8s     │ Every hour       │ Europe/Berl │ ⏸ Paused │ -            │ ✕ Failed    │ [Resume] [Run now] [Stop]░ │
 │ playwright-checkout-… │ K8s     │ Every 10 minutes │ Europe/Berl │ ▶ Active │ 27 Sep 14:40 │ ✓ Succeeded │ [Pause] [Run now] [Stop]░  │
 └───────────────────────┴─────────┴──────────────────┴─────────────┴──────────┴──────────────┴─────────────┴────────────────────────────┘
  ░ = disabled, with the reason in text (read-only role, run in progress, nothing to stop)
  sr-only live region: "sync-inventory: run succeeded"

/jobs/sync-inventory
┌──────────────────────────────────────────────────────────────────────────────────────┐
│ ← Jobs   sync-inventory   [BullMQ]  ▶ Active                [Pause schedule] [Run now]│
│ Pulls supplier API and upserts Postgres · owner: team-inventory                      │
│ Every 15 minutes (Europe/Berlin) · Forbid overlap · timeout 600 s · retries 3        │
├───────────────────────────────── Run history ────────────────────────────────────────┤
│ Status        Trigger    Started            Duration   Attempts   Error              │
│ ◌ Running     schedule   14:30:00           -          1                     [Stop]  │ ◀ selected
│ ✓ Succeeded   manual     14:21:13           41 s       1                              │
│ ✕ Failed      schedule   14:15:00           12 s       3          Supplier 503        │
│                                       [ Latest ]  [ Older runs → ]                   │
├───────────────────────────────── Logs: run 8f2c ─────────────────────────────────────┤
│ 14:30:00.112 fetching page 1/12                                                      │
│ 14:30:02.518 upserted 500 rows                                                       │
│ …                                                          [⏸ Pause scroll] [⤓ Latest]│
└──────────────────────────────────────────────────────────────────────────────────────┘
```

### Component tree

{/* diagram source: /diagrams/diagram-065.mmd */}
![diagram](/diagrams/diagram-065.svg)

## 3. Project structure

The layout follows [Next.js Project Structure & Architecture](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Project%20Structure%20%26%20Architecture) (`src/`, feature folders, everything with secrets under `server/` behind `import 'server-only'`).

```text
ops-console/
├── next.config.ts
├── Dockerfile
├── src/
│   ├── env.ts                               # zod-validated env (server-only)
│   ├── proxy.ts                             # optimistic cookie check (Next 16 name for middleware)
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── (auth)/login/page.tsx
│   │   ├── (console)/
│   │   │   ├── layout.tsx                   # shell, nav, Suspense(UserMenu)
│   │   │   ├── error.tsx
│   │   │   └── jobs/
│   │   │       ├── page.tsx                 # table of jobs
│   │   │       ├── loading.tsx              # skeleton
│   │   │       ├── error.tsx
│   │   │       └── [id]/
│   │   │           ├── page.tsx             # detail, run history, log viewer
│   │   │           ├── loading.tsx
│   │   │           └── not-found.tsx
│   │   └── api/
│   │       ├── auth/{login,callback}/route.ts   # from the auth note
│   │       ├── health/route.ts
│   │       └── jobs/
│   │           ├── events/route.ts                          # SSE proxy
│   │           └── [id]/runs/[runId]/logs/route.ts          # SSE proxy
│   ├── features/jobs/
│   │   ├── actions.ts                       # 'use server': pause, resume, runNow, stopRun
│   │   ├── schemas.ts                       # zod: API contract, shared with client components
│   │   ├── format.ts                        # cronstrue, dates
│   │   ├── components/
│   │   │   ├── LiveJobsTable.tsx  JobActions.tsx  StopRunDialog.tsx
│   │   │   ├── StatusBadge.tsx    RunHistory.tsx  LogViewer.tsx  LiveRefresh.tsx
│   │   │   └── EmptyJobs.tsx      JobsTableSkeleton.tsx
│   │   └── hooks/useEventSource.ts
│   └── server/
│       ├── auth/{oidc,session,permissions,tokens}.ts
│       ├── jobs-api/client.ts               # typed client (server-only)
│       └── sse-proxy.ts
└── tests/{unit,e2e}/
```

```ts
// next.config.ts
import type { NextConfig } from "next";

const config: NextConfig = {
  output: "standalone",            // small runtime image (see Deploying Next.js to Kubernetes)
  typedRoutes: true,
  poweredByHeader: false,
};
export default config;
```

Caching: every view here is per-user and live, so nothing is cached. `fetch` is uncached by default since Next 15, and in Next 16 caching is opt-in (`'use cache'`, see [Next.js Rendering & Caching](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Rendering%20%26%20Caching)). If you enable `cacheComponents`, keep anything that reads the session inside `<Suspense>` (the layout already does; `loading.tsx` wraps each page).

```ts
// src/env.ts
import "server-only";
import { z } from "zod";

export const env = z
  .object({
    JOBS_API_URL: z.url().default("http://jobs-api.shop-prod.svc.cluster.local:8080"),
    OIDC_ISSUER: z.url(),
    OIDC_CLIENT_ID: z.string().min(1),
    OIDC_CLIENT_SECRET: z.string().min(1),
  })
  .parse(process.env);
```

## 4. Auth and roles

Login is the OIDC Authorization Code + PKCE flow with a server-side session from [Next.js Authentication & Authorization](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Authentication%20%26%20Authorization) (the `ops-console` example there): `/api/auth/login`, `/api/auth/callback`, a `__Host-session` cookie holding a random ID, tokens stored server-side, and DAL helpers `getViewer()`, `requireViewer()`, `requirePermission()`. Only what the dashboard adds is shown here.

```ts
// src/server/auth/permissions.ts
export type Role = "viewer" | "operator" | "admin";
export type Permission = "ops:jobs:read" | "ops:jobs:write";

const grants: Record<Role, Permission[]> = {
  viewer: ["ops:jobs:read"],
  operator: ["ops:jobs:read", "ops:jobs:write"],
  admin: ["ops:jobs:read", "ops:jobs:write"],
};

export const can = (roles: Role[], p: Permission) => roles.some((r) => grants[r].includes(p));
```

The access token must be fresh when we forward it. Refresh on the server, and serialize refreshes per session: IdPs that rotate refresh tokens invalidate the old one, so two concurrent refreshes log the user out.

```ts
// src/server/auth/tokens.ts
import "server-only";
import * as client from "openid-client";
import { oidc } from "./oidc";
import { db } from "@/server/db";
import type { Viewer } from "./session";

export class SessionExpiredError extends Error {}

const inflight = new Map<string, Promise<string>>();   // per-process refresh lock

export async function getAccessToken(viewer: Viewer): Promise<string> {
  const s = await db.session.findUnique({ where: { idHash: viewer.sessionIdHash } });
  if (!s) throw new SessionExpiredError();
  if (s.accessTokenExpiresAt.getTime() - Date.now() > 30_000) return s.accessToken;
  if (!s.refreshToken) throw new SessionExpiredError();

  let p = inflight.get(s.idHash);
  if (!p) {
    p = (async () => {
      try {
        const tokens = await client.refreshTokenGrant(await oidc(), s.refreshToken!);
        await db.session.update({
          where: { idHash: s.idHash },
          data: {
            accessToken: tokens.access_token,
            refreshToken: tokens.refresh_token ?? s.refreshToken,   // rotation
            accessTokenExpiresAt: new Date(Date.now() + (tokens.expiresIn() ?? 300) * 1000),
          },
        });
        return tokens.access_token;
      } catch {
        throw new SessionExpiredError();
      } finally {
        inflight.delete(s.idHash);
      }
    })();
    inflight.set(s.idHash, p);
  }
  return p;
}
```

With several `ops-console` replicas the in-process lock is not enough; use a row lock (`SELECT ... FOR UPDATE`) or a short Redis lock around the refresh, or accept a retry on the rare race.

Where the role check happens:

| Place | Check | Why |
|---|---|---|
| `proxy.ts` | cookie present? else redirect to `/login` | fast, optimistic, never the only check |
| Page | `requireViewer()`; compute `canWrite` for the UI | read access + which buttons to enable |
| **Every Server Action** | `requireViewer()` + `can(roles, 'ops:jobs:write')` | actions are public POST endpoints; a read-only user can call them with curl |
| SSE Route Handlers | `getViewer()` → 401 | handlers are public endpoints too |
| `jobs-api` | validates the JWT and the role/scope again, audits the call | defense in depth; the dashboard is not the only client |

Read-only users see the buttons **disabled with a reason** ("Read-only access: ask #ops-access for the operator role"). Hiding them entirely makes people think the feature is broken; disabling with a reason teaches the permission model.

## 5. The typed API client

Rules: `server-only` (a client import fails the build), forwards the user's token (never a service account for user actions, so `jobs-api` audits the real person), a timeout, zod-parsed responses (the contract is checked at the boundary instead of crashing somewhere in a component), RFC 9457 Problem Details mapped to a typed error.

```ts
// src/features/jobs/schemas.ts  (no 'server-only': client components import the types)
import { z } from "zod";

const Timestamp = z.iso.datetime({ offset: true });

export const Backend = z.enum(["bullmq", "k8s"]);
export const JobState = z.enum(["active", "paused"]);
export const RunStatus = z.enum(["queued", "running", "succeeded", "failed", "cancelling", "cancelled"]);

export const LastRun = z.object({
  runId: z.string(),
  status: RunStatus,
  startedAt: Timestamp,
  finishedAt: Timestamp.nullish(),
});

export const JobSummary = z.object({
  id: z.string(),
  name: z.string(),
  backend: Backend,
  schedule: z.string(),                   // 5-field cron
  timezone: z.string(),                   // IANA, e.g. Europe/Berlin
  state: JobState,
  nextRunAt: Timestamp.nullable(),        // null while paused
  lastRun: LastRun.optional(),
});

export const JobDetail = JobSummary.extend({
  description: z.string(),
  owner: z.string(),
  concurrencyPolicy: z.enum(["Forbid", "Allow", "Replace"]),
  timeoutSeconds: z.number().int().positive(),
  retries: z.number().int().nonnegative(),
});

export const JobRun = z.object({
  runId: z.string(),
  status: RunStatus,
  trigger: z.enum(["schedule", "manual"]),
  startedAt: Timestamp,
  finishedAt: Timestamp.optional(),
  durationMs: z.number().int().nonnegative().optional(),
  error: z.string().optional(),
  attempts: z.number().int().nonnegative(),
});

export const RunPage = z.object({ items: z.array(JobRun), nextCursor: z.string().nullable() });
export const RunAccepted = z.object({ runId: z.string() });

// Live events from GET /api/jobs/events (flat envelope, as published by jobs-api;
// see BullMQ Job Schedulers in Practice). Partial updates: status or progress may be absent.
// .loose() keeps unknown fields instead of failing, so jobs-api can add fields first.
export const JobEvent = z.discriminatedUnion("type", [
  z.object({
    type: z.literal("run.updated"),
    jobId: z.string(),
    runId: z.string(),
    status: RunStatus.optional(),
    progress: z.unknown().optional(),
  }).loose(),
  z.object({ type: z.literal("job.updated"), jobId: z.string() }).loose(),
]);

export const ProblemDetails = z.object({
  type: z.string().optional(),
  title: z.string().optional(),
  status: z.number().optional(),
  detail: z.string().optional(),
  instance: z.string().optional(),
});

export type JobSummary = z.infer<typeof JobSummary>;
export type JobDetail = z.infer<typeof JobDetail>;
export type JobRun = z.infer<typeof JobRun>;
export type RunStatus = z.infer<typeof RunStatus>;
export type JobEvent = z.infer<typeof JobEvent>;
export type ProblemDetails = z.infer<typeof ProblemDetails>;
```

```ts
// src/server/jobs-api/client.ts
import "server-only";
import { z } from "zod";
import { env } from "@/env";
import { getAccessToken } from "@/server/auth/tokens";
import type { Viewer } from "@/server/auth/session";
import {
  JobDetail, JobSummary, ProblemDetails, RunAccepted, RunPage,
  type ProblemDetails as Problem,
} from "@/features/jobs/schemas";

export class JobsApiError extends Error {
  constructor(public readonly status: number, public readonly problem?: Problem) {
    super(problem?.detail ?? problem?.title ?? `jobs-api responded ${status}`);
  }
}

type CallInit = { method?: "GET" | "POST"; body?: unknown; idempotencyKey?: string };

const enc = encodeURIComponent;

export function jobsApi(viewer: Viewer) {
  async function call<T>(path: string, schema: z.ZodType<T>, init: CallInit = {}): Promise<T> {
    const token = await getAccessToken(viewer);
    const headers = new Headers({ Authorization: `Bearer ${token}`, Accept: "application/json" });
    if (init.body !== undefined) headers.set("Content-Type", "application/json");
    if (init.idempotencyKey) headers.set("Idempotency-Key", init.idempotencyKey);

    // Retry once on network errors, but only if the request is safe to repeat:
    // GETs, or POSTs carrying an Idempotency-Key (the server dedupes them).
    const retryable = (init.method ?? "GET") === "GET" || !!init.idempotencyKey;
    let res!: Response;
    for (let attempt = 0; ; attempt++) {
      try {
        res = await fetch(`${env.JOBS_API_URL}${path}`, {
          method: init.method ?? "GET",
          headers,
          body: init.body === undefined ? undefined : JSON.stringify(init.body),
          cache: "no-store",
          signal: AbortSignal.timeout(8_000),
        });
        break;
      } catch (err) {
        if (attempt === 0 && retryable) continue;
        throw err;
      }
    }

    const text = await res.text();
    const json = text ? JSON.parse(text) : undefined;
    if (!res.ok) {
      const problem = ProblemDetails.safeParse(json);
      throw new JobsApiError(res.status, problem.success ? problem.data : undefined);
    }
    return schema.parse(json);   // ZodError here = contract drift: alert, don't guess
  }

  return {
    listJobs: () => call("/api/jobs", z.array(JobSummary)),
    getJob: (id: string) => call(`/api/jobs/${enc(id)}`, JobDetail),
    listRuns: (id: string, o: { limit: number; cursor?: string }) => {
      const qs = new URLSearchParams({ limit: String(o.limit), ...(o.cursor ? { cursor: o.cursor } : {}) });
      return call(`/api/jobs/${enc(id)}/runs?${qs}`, RunPage);
    },
    pause: (id: string, key: string) =>
      call(`/api/jobs/${enc(id)}/pause`, z.undefined(), { method: "POST", idempotencyKey: key }),
    resume: (id: string, key: string) =>
      call(`/api/jobs/${enc(id)}/resume`, z.undefined(), { method: "POST", idempotencyKey: key }),
    runNow: (id: string, key: string, input?: unknown) =>
      call(`/api/jobs/${enc(id)}/runs`, RunAccepted, { method: "POST", body: { input }, idempotencyKey: key }),
    stopRun: (id: string, runId: string, key: string) =>
      call(`/api/jobs/${enc(id)}/runs/${enc(runId)}/stop`, z.unknown(), { method: "POST", idempotencyKey: key }),
  };
}
```

Only `POST /runs` requires an `Idempotency-Key` in the API contract. Sending one on pause, resume and stop costs nothing and lets `jobs-api` dedupe those too if it chooses to.

Formatting helpers (server side, so cronstrue never ships to the browser):

```ts
// src/features/jobs/format.ts
import cronstrue from "cronstrue";

export function describeCron(expr: string) {
  try {
    return cronstrue.toString(expr, { use24HourTimeFormat: true, verbose: false });
  } catch {
    return expr;                                   // unknown syntax: show the raw expression
  }
}
// "0 2 * * *" -> "At 02:00", "*/15 * * * *" -> "Every 15 minutes"

// Deterministic output (fixed locale + the job's own timezone), so SSR and hydration agree.
export function formatInZone(iso: string | null | undefined, timeZone: string) {
  if (!iso) return "-";
  return new Intl.DateTimeFormat("en-GB", {
    timeZone, day: "2-digit", month: "short", hour: "2-digit", minute: "2-digit",
  }).format(new Date(iso));
}
```

## 6. Pages

### Shell layout

```tsx
// src/app/(console)/layout.tsx
import { Suspense } from "react";
import Link from "next/link";
import { UserMenu } from "@/features/auth/components/UserMenu";

export default function ConsoleLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="shell">
      <a href="#main" className="skip-link">Skip to content</a>
      <header>
        <nav aria-label="Main">
          <Link href="/jobs">Jobs</Link>
        </nav>
        <Suspense fallback={<span className="avatar-skeleton" aria-hidden />}>
          <UserMenu />  {/* shows "read-only" badge for viewers */}
        </Suspense>
      </header>
      <main id="main">{children}</main>
    </div>
  );
}
```

The layout is UX only. Layouts don't re-run on every navigation and don't protect Server Actions, so every page and action checks again.

### `/jobs`: the table

```tsx
// src/app/(console)/jobs/page.tsx
import type { Metadata } from "next";
import { requireViewer } from "@/server/auth/session";
import { can } from "@/server/auth/permissions";
import { jobsApi } from "@/server/jobs-api/client";
import { describeCron, formatInZone } from "@/features/jobs/format";
import { LiveJobsTable } from "@/features/jobs/components/LiveJobsTable";
import { EmptyJobs } from "@/features/jobs/components/EmptyJobs";

export const metadata: Metadata = { title: "Jobs · Acme Ops" };

export default async function JobsPage() {
  const viewer = await requireViewer();
  const jobs = await jobsApi(viewer).listJobs();

  if (jobs.length === 0) return <EmptyJobs />;

  // Precompute display strings on the server: no cronstrue in the client bundle,
  // and no locale/timezone hydration mismatches.
  const rows = jobs.map((j) => ({
    ...j,
    scheduleText: describeCron(j.schedule),
    nextRunText: formatInZone(j.nextRunAt, j.timezone),
  }));

  return (
    <>
      <h1>Scheduled jobs</h1>
      <LiveJobsTable initialRows={rows} canWrite={can(viewer.roles, "ops:jobs:write")} />
    </>
  );
}
```

```tsx
// src/features/jobs/components/StatusBadge.tsx
import type { RunStatus } from "../schemas";

type Status = RunStatus | "active" | "paused";

// Icon + text + colour: never colour alone (WCAG 1.4.1)
const META: Record<Status, { icon: string; label: string; tone: string }> = {
  active:     { icon: "▶", label: "Active",     tone: "ok" },
  paused:     { icon: "⏸", label: "Paused",     tone: "muted" },
  queued:     { icon: "…", label: "Queued",     tone: "info" },
  running:    { icon: "◌", label: "Running",    tone: "info" },
  succeeded:  { icon: "✓", label: "Succeeded",  tone: "ok" },
  failed:     { icon: "✕", label: "Failed",     tone: "danger" },
  cancelling: { icon: "⊘", label: "Stopping",   tone: "warn" },
  cancelled:  { icon: "⊘", label: "Cancelled",  tone: "muted" },
};

export function StatusBadge({ status }: { status: Status }) {
  const m = META[status];
  return (
    <span className={`badge badge--${m.tone}`}>
      <span aria-hidden="true">{m.icon}</span> {m.label}
    </span>
  );
}

export function BackendBadge({ backend }: { backend: "bullmq" | "k8s" }) {
  return <span className="badge badge--outline">{backend === "bullmq" ? "BullMQ" : "K8s CronJob"}</span>;
}
```

```tsx
// src/features/jobs/components/LiveJobsTable.tsx
"use client";
import { useRef, useState, startTransition } from "react";
import Link from "next/link";
import { useRouter } from "next/navigation";
import { JobEvent, type JobSummary } from "../schemas";
import { useEventSource } from "../hooks/useEventSource";
import { StatusBadge, BackendBadge } from "./StatusBadge";
import { JobActions } from "./JobActions";

type Row = JobSummary & { scheduleText: string; nextRunText: string };

export function LiveJobsTable({ initialRows, canWrite }: { initialRows: Row[]; canWrite: boolean }) {
  const router = useRouter();
  const [rows, setRows] = useState(initialRows);
  const [prevInitial, setPrevInitial] = useState(initialRows);
  const [announcement, setAnnouncement] = useState("");
  const refreshTimer = useRef<ReturnType<typeof setTimeout>>(undefined);

  // New server render (after an action or router.refresh): adopt it.
  if (initialRows !== prevInitial) {
    setPrevInitial(initialRows);
    setRows(initialRows);
  }

  const connection = useEventSource("/api/jobs/events", {
    onMessage(data) {
      const parsed = JobEvent.safeParse(JSON.parse(data));
      if (!parsed.success) return;                         // unknown event: ignore, don't crash
      const ev = parsed.data;
      if (ev.type === "run.updated") {
        if (!ev.status) return;                            // progress-only event: nothing to show here
        const status = ev.status;
        setRows((rs) =>
          rs.map((r) =>
            r.id === ev.jobId
              ? {
                  ...r,
                  lastRun: {
                    runId: ev.runId,
                    status,
                    // keep the known start time for the same run; a new run starts "now"
                    startedAt: r.lastRun?.runId === ev.runId ? r.lastRun.startedAt : new Date().toISOString(),
                  },
                }
              : r,
          ),
        );
        const name = rows.find((r) => r.id === ev.jobId)?.name ?? ev.jobId;
        if (["succeeded", "failed", "cancelled"].includes(status)) {
          setAnnouncement(`${name}: run ${status}`);
        }
      } else {
        // Schedule/state changed: let the server recompute (cronstrue, next run).
        clearTimeout(refreshTimer.current);
        refreshTimer.current = setTimeout(() => startTransition(() => router.refresh()), 300);
      }
    },
  });

  return (
    <>
      <p className="live-indicator">
        Live updates: <strong>{connection === "open" ? "connected" : connection === "connecting" ? "reconnecting…" : "offline"}</strong>
      </p>
      {/* Polite live region: screen readers hear finished runs without losing focus */}
      <p role="status" aria-live="polite" className="sr-only">{announcement}</p>

      <table className="jobs-table">
        <caption className="sr-only">Scheduled jobs with state, next run and last run</caption>
        <thead>
          <tr>
            <th scope="col">Job</th><th scope="col">Backend</th><th scope="col">Schedule</th>
            <th scope="col">Timezone</th><th scope="col">State</th><th scope="col">Next run</th>
            <th scope="col">Last run</th><th scope="col"><span className="sr-only">Actions</span></th>
          </tr>
        </thead>
        <tbody>
          {rows.map((job) => (
            <tr key={job.id}>
              <th scope="row"><Link href={`/jobs/${encodeURIComponent(job.id)}`}>{job.name}</Link></th>
              <td><BackendBadge backend={job.backend} /></td>
              <td><span title={job.schedule}>{job.scheduleText}</span></td>
              <td>{job.timezone}</td>
              <td><StatusBadge status={job.state} /></td>
              <td>{job.nextRunAt ? <time dateTime={job.nextRunAt}>{job.nextRunText}</time> : "-"}</td>
              <td>{job.lastRun ? <StatusBadge status={job.lastRun.status} /> : "Never run"}</td>
              <td><JobActions job={job} canWrite={canWrite} forbidOverlap /></td>
            </tr>
          ))}
        </tbody>
      </table>
    </>
  );
}
```

### `/jobs/[id]`: detail, history, logs

```tsx
// src/app/(console)/jobs/[id]/page.tsx
import { notFound } from "next/navigation";
import Link from "next/link";
import { requireViewer } from "@/server/auth/session";
import { can } from "@/server/auth/permissions";
import { jobsApi, JobsApiError } from "@/server/jobs-api/client";
import { describeCron } from "@/features/jobs/format";
import { StatusBadge, BackendBadge } from "@/features/jobs/components/StatusBadge";
import { JobActions } from "@/features/jobs/components/JobActions";
import { RunHistory } from "@/features/jobs/components/RunHistory";
import { LogViewer } from "@/features/jobs/components/LogViewer";
import { LiveRefresh } from "@/features/jobs/components/LiveRefresh";

type Props = {
  params: Promise<{ id: string }>;
  searchParams: Promise<{ cursor?: string; run?: string }>;
};

export default async function JobPage({ params, searchParams }: Props) {
  const [{ id }, { cursor, run }] = await Promise.all([params, searchParams]);
  const viewer = await requireViewer();
  const api = jobsApi(viewer);

  // Independent requests in parallel: no waterfall.
  const [job, runs] = await Promise.all([
    api.getJob(id),
    api.listRuns(id, { limit: 20, cursor }),
  ]).catch((e) => {
    if (e instanceof JobsApiError && e.status === 404) notFound();
    throw e;                                            // -> error.tsx
  });

  const selectedRunId = run ?? runs.items[0]?.runId;
  const selectedRun = runs.items.find((r) => r.runId === selectedRunId);
  const canWrite = can(viewer.roles, "ops:jobs:write");

  return (
    <article aria-labelledby="job-title">
      <LiveRefresh jobId={job.id} />
      <p><Link href="/jobs">← Jobs</Link></p>
      <header className="job-header">
        <h1 id="job-title">{job.name}</h1>
        <BackendBadge backend={job.backend} /> <StatusBadge status={job.state} />
        <JobActions job={job} canWrite={canWrite} forbidOverlap={job.concurrencyPolicy === "Forbid"} />
      </header>

      <p>{job.description}</p>
      <dl className="job-facts">
        <dt>Schedule</dt><dd>{describeCron(job.schedule)} (<code>{job.schedule}</code>, {job.timezone})</dd>
        <dt>Owner</dt><dd>{job.owner}</dd>
        <dt>Overlap</dt><dd>{job.concurrencyPolicy}</dd>
        <dt>Timeout</dt><dd>{job.timeoutSeconds} s</dd>
        <dt>Retries</dt><dd>{job.retries}</dd>
      </dl>

      <section aria-labelledby="runs-title">
        <h2 id="runs-title">Run history</h2>
        <RunHistory job={job} runs={runs.items} nextCursor={runs.nextCursor} isFirstPage={!cursor}
          selectedRunId={selectedRunId} canWrite={canWrite} />
      </section>

      {selectedRunId && (
        <section aria-labelledby="logs-title">
          <h2 id="logs-title">Logs: run {selectedRunId.slice(0, 8)}</h2>
          {/* key: a new run gets a fresh viewer (state + connection) */}
          <LogViewer key={selectedRunId} jobId={job.id} runId={selectedRunId}
            live={selectedRun?.status === "running" || selectedRun?.status === "queued"} />
        </section>
      )}
    </article>
  );
}
```

```tsx
// src/features/jobs/components/RunHistory.tsx  (Server Component)
import Link from "next/link";
import type { JobDetail, JobRun } from "../schemas";
import { StatusBadge } from "./StatusBadge";
import { StopRunDialog } from "./StopRunDialog";

const time = (iso: string, tz: string) =>
  new Intl.DateTimeFormat("en-GB", { timeZone: tz, dateStyle: "short", timeStyle: "medium" }).format(new Date(iso));

export function RunHistory({ job, runs, nextCursor, isFirstPage, selectedRunId, canWrite }: {
  job: JobDetail; runs: JobRun[]; nextCursor: string | null; isFirstPage: boolean;
  selectedRunId?: string; canWrite: boolean;
}) {
  if (runs.length === 0) return <p>No runs yet. Use <strong>Run now</strong> to start one.</p>;
  const base = `/jobs/${encodeURIComponent(job.id)}`;

  return (
    <>
      <table>
        <thead>
          <tr><th scope="col">Status</th><th scope="col">Trigger</th><th scope="col">Started</th>
            <th scope="col">Duration</th><th scope="col">Attempts</th><th scope="col">Error</th><th scope="col"><span className="sr-only">Actions</span></th></tr>
        </thead>
        <tbody>
          {runs.map((r) => (
            <tr key={r.runId} aria-current={r.runId === selectedRunId ? "true" : undefined}>
              <td><StatusBadge status={r.status} /></td>
              <td>{r.trigger}</td>
              <td>
                <Link href={`${base}?run=${encodeURIComponent(r.runId)}`} scroll={false}>
                  <time dateTime={r.startedAt}>{time(r.startedAt, job.timezone)}</time>
                </Link>
              </td>
              <td>{r.durationMs != null ? `${Math.round(r.durationMs / 1000)} s` : "-"}</td>
              <td>{r.attempts}</td>
              <td className="error-cell">{r.error ?? ""}</td>
              <td>
                {(r.status === "running" || r.status === "queued" || r.status === "cancelling") && (
                  <StopRunDialog jobId={job.id} jobName={job.name} runId={r.runId}
                    disabled={!canWrite || r.status === "cancelling"} />
                )}
              </td>
            </tr>
          ))}
        </tbody>
      </table>
      <nav aria-label="Run history pages" className="pager">
        {!isFirstPage && <Link href={base}>Latest</Link>}
        {nextCursor && <Link href={`${base}?cursor=${encodeURIComponent(nextCursor)}`}>Older runs →</Link>}
      </nav>
    </>
  );
}
```

Cursor pagination is forward-only by design (stable under inserts: new runs arrive every 10 minutes, so offset pages would shift). "Latest" resets to page one; the browser Back button handles "newer".

```tsx
// src/features/jobs/components/LiveRefresh.tsx
"use client";
import { useRef, startTransition } from "react";
import { useRouter } from "next/navigation";
import { useEventSource } from "../hooks/useEventSource";

// Detail page: any event for this job -> re-render the Server Components (debounced).
export function LiveRefresh({ jobId }: { jobId: string }) {
  const router = useRouter();
  const t = useRef<ReturnType<typeof setTimeout>>(undefined);
  useEventSource("/api/jobs/events", {
    onMessage(data) {
      const ev = JSON.parse(data) as { type: string; jobId?: string };
      if (ev.jobId !== jobId) return;
      clearTimeout(t.current);
      t.current = setTimeout(() => startTransition(() => router.refresh()), 500);
    },
  });
  return null;
}
```

## 7. Mutations: Server Actions with optimistic UI

### The actions

```ts
// src/features/jobs/actions.ts
"use server";
import { z } from "zod";
import { revalidatePath } from "next/cache";
import { requireViewer, type Viewer } from "@/server/auth/session";
import { can } from "@/server/auth/permissions";
import { jobsApi, JobsApiError } from "@/server/jobs-api/client";
import { SessionExpiredError } from "@/server/auth/tokens";

export type ActionState =
  | { status: "idle" }
  | { status: "ok"; message: string; at: number }
  | { status: "error"; message: string; at: number };

const JobInput = z.object({
  jobId: z.string().regex(/^[a-z0-9][a-z0-9-]{0,62}$/),
  idempotencyKey: z.uuid(),
});
const StopInput = JobInput.extend({ runId: z.string().min(1).max(128) });

async function mutate<S extends z.ZodType>(
  formData: FormData,
  schema: S,
  op: (api: ReturnType<typeof jobsApi>, input: z.infer<S>, viewer: Viewer) => Promise<string>,
): Promise<ActionState> {
  const at = Date.now();
  const viewer = await requireViewer();                       // redirects if not logged in
  if (!can(viewer.roles, "ops:jobs:write")) {                 // authorization, every time
    return { status: "error", message: "Read-only access: you need the operator role.", at };
  }
  const parsed = schema.safeParse(Object.fromEntries(formData));
  if (!parsed.success) return { status: "error", message: "Invalid request.", at };

  try {
    const message = await op(jobsApi(viewer), parsed.data, viewer);
    revalidatePath("/jobs");                                   // list
    revalidatePath(`/jobs/${parsed.data.jobId}`);              // detail
    return { status: "ok", message, at };
  } catch (e) {
    if (e instanceof SessionExpiredError) return { status: "error", message: "Session expired. Reload to sign in again.", at };
    if (e instanceof JobsApiError) {
      // Expected business errors come back as values; the UI shows them inline.
      if (e.status === 403) return { status: "error", message: "jobs-api refused: missing ops:jobs:write.", at };
      if (e.status === 404) return { status: "error", message: "Job or run not found (was it removed?).", at };
      if (e.status === 409) return { status: "error", message: e.problem?.detail ?? "Conflicting state: refresh and retry.", at };
    }
    throw e;                                                   // unexpected: error.tsx + logs
  }
}

export async function pauseJob(_: ActionState, fd: FormData) {
  return mutate(fd, JobInput, async (api, i) => { await api.pause(i.jobId, i.idempotencyKey); return `Paused schedule of ${i.jobId}. Running runs continue.`; });
}

export async function resumeJob(_: ActionState, fd: FormData) {
  return mutate(fd, JobInput, async (api, i) => { await api.resume(i.jobId, i.idempotencyKey); return `Resumed ${i.jobId}. Missed runs are not backfilled.`; });
}

export async function runJobNow(_: ActionState, fd: FormData) {
  return mutate(fd, JobInput, async (api, i) => { const { runId } = await api.runNow(i.jobId, i.idempotencyKey); return `Started run ${runId.slice(0, 8)} of ${i.jobId}.`; });
}

export async function stopRun(_: ActionState, fd: FormData) {
  return mutate(fd, StopInput, async (api, i) => { await api.stopRun(i.jobId, i.runId, i.idempotencyKey); return `Stopping run ${i.runId.slice(0, 8)}…`; });
}
```

Why `revalidatePath` and not `updateTag`: nothing here is cached with tags, and the page is dynamic. `revalidatePath` inside an action makes the action's response carry the fresh RSC payload for the current route, so the table reconciles in the same round trip. `refresh()` from `next/cache` (Next 16, only callable inside Server Actions) does the same for "just re-render the current page". Auditing happens in `jobs-api`, which sees the user's own token.

### One hook per button: `useActionState` + optimistic patch + per-click key

```tsx
// src/features/jobs/components/JobActions.tsx
"use client";
import { useActionState, useOptimistic } from "react";
import { pauseJob, resumeJob, runJobNow, type ActionState } from "../actions";
import type { JobSummary, RunStatus } from "../schemas";
import { StopRunDialog } from "./StopRunDialog";

type View = { state: JobSummary["state"]; runStatus?: RunStatus };
const idle: ActionState = { status: "idle" };

function useJobAction(
  action: (s: ActionState, fd: FormData) => Promise<ActionState>,
  optimistic: () => void,
) {
  return useActionState(async (prev: ActionState, fd: FormData) => {
    optimistic();                                  // instant feedback, inside the transition
    fd.set("idempotencyKey", crypto.randomUUID()); // one key per click (generated in the browser)
    return action(prev, fd);
  }, idle);
}

export function JobActions({ job, canWrite, forbidOverlap }: {
  job: JobSummary; canWrite: boolean; forbidOverlap: boolean;
}) {
  const [view, patch] = useOptimistic<View, Partial<View>>(
    { state: job.state, runStatus: job.lastRun?.status },
    (cur, p) => ({ ...cur, ...p }),
  );

  const [pauseRes, pauseAction, pausing] = useJobAction(pauseJob, () => patch({ state: "paused" }));
  const [resumeRes, resumeAction, resuming] = useJobAction(resumeJob, () => patch({ state: "active" }));
  const [runRes, runAction, starting] = useJobAction(runJobNow, () => patch({ runStatus: "queued" }));

  const busy = pausing || resuming || starting;
  const inFlight = view.runStatus === "queued" || view.runStatus === "running";
  const readOnlyReason = canWrite ? undefined : "Read-only access";
  const latest = [pauseRes, resumeRes, runRes]
    .filter((r): r is Exclude<ActionState, { status: "idle" }> => r.status !== "idle")
    .sort((a, b) => b.at - a.at)[0];

  const reasonId = `reason-${job.id}`;

  return (
    <div className="job-actions" role="group" aria-label={`Actions for ${job.name}`}>
      <form action={view.state === "active" ? pauseAction : resumeAction}>
        <input type="hidden" name="jobId" value={job.id} />
        <button type="submit" disabled={!canWrite || busy}
          aria-describedby={readOnlyReason ? reasonId : undefined}>
          {view.state === "active" ? (pausing ? "Pausing…" : "Pause schedule") : resuming ? "Resuming…" : "Resume schedule"}
        </button>
      </form>

      <form action={runAction}>
        <input type="hidden" name="jobId" value={job.id} />
        <button type="submit" disabled={!canWrite || busy || (forbidOverlap && inFlight)}
          aria-describedby={readOnlyReason ? reasonId : inFlight && forbidOverlap ? `${reasonId}-overlap` : undefined}>
          {starting ? "Starting…" : "Run now"}
        </button>
      </form>

      {job.lastRun && (
        <StopRunDialog jobId={job.id} jobName={job.name} runId={job.lastRun.runId}
          disabled={!canWrite || !inFlight || view.runStatus === "cancelling"}
          onOptimistic={() => patch({ runStatus: "cancelling" })} />
      )}

      {readOnlyReason && <span id={reasonId} className="hint">{readOnlyReason}: ask #ops-access for the operator role.</span>}
      {inFlight && forbidOverlap && <span id={`${reasonId}-overlap`} className="sr-only">A run is already in progress.</span>}

      {/* Result of the last action: visible + announced */}
      <p role={latest?.status === "error" ? "alert" : "status"} className={`action-msg action-msg--${latest?.status ?? "idle"}`}>
        {latest?.message}
      </p>
    </div>
  );
}
```

How the pieces behave:

1. Click → the wrapper runs inside a transition, so `patch()` shows "Paused" immediately and the button shows "Pausing…".
2. The Server Action runs; `revalidatePath` returns the new server state in the same response.
3. The transition ends; `useOptimistic` drops the patch and shows the **real** `job.state` from props. If the call failed, the UI snaps back and the error message appears in a live region. No manual rollback code.

**Disabled states** come from the optimistic view, so a paused job can't be paused twice, Run now is disabled while a run is queued/running (for `Forbid` jobs), and Stop is only enabled for an in-flight run. They are UX, not safety: `jobs-api` still returns `409 Conflict` for an invalid transition, and the action turns it into a message.

**Idempotency-Key per click.** Generated in the browser at submit time (not at render, which would differ between SSR and hydration). The same key is reused if the client's `fetch` to `jobs-api` retries after a network error, so a lost response never starts two runs. A second click is a second intent and gets a new key; the disabled-while-pending button stops accidental double clicks. `jobs-api` remembers keys for a retention window and replays the first response ([Job Control API - Pause, Resume, Run Now & Stop](/Software%20Engineering/Backend/Scheduled%20Jobs/Job%20Control%20API%20-%20Pause%2C%20Resume%2C%20Run%20Now%20%26%20Stop)).

### Stop: a confirmation dialog that's actually accessible

```tsx
// src/features/jobs/components/StopRunDialog.tsx
"use client";
import { useActionState, useId, useRef } from "react";
import { stopRun, type ActionState } from "../actions";

export function StopRunDialog({ jobId, jobName, runId, disabled, onOptimistic }: {
  jobId: string; jobName: string; runId: string; disabled: boolean; onOptimistic?: () => void;
}) {
  const ref = useRef<HTMLDialogElement>(null);
  const titleId = useId();
  const descId = useId();
  const [res, action, pending] = useActionState(async (prev: ActionState, fd: FormData) => {
    onOptimistic?.();
    fd.set("idempotencyKey", crypto.randomUUID());
    const r = await stopRun(prev, fd);
    if (r.status === "ok") ref.current?.close();
    return r;
  }, { status: "idle" } as ActionState);

  return (
    <>
      <button type="button" className="danger" disabled={disabled || pending}
        onClick={() => ref.current?.showModal()}>
        {pending ? "Stopping…" : "Stop run"}
      </button>

      {/* Native modal: focus trap, Esc to close, inert background, focus restored on close */}
      <dialog ref={ref} aria-labelledby={titleId} aria-describedby={descId} className="confirm">
        <h2 id={titleId}>Stop this run of {jobName}?</h2>
        <p id={descId}>
          Run <code>{runId.slice(0, 8)}</code> will be asked to stop and finish its current step.
          Work already done is not rolled back. The schedule is <strong>not</strong> paused.
        </p>
        <form action={action}>
          <input type="hidden" name="jobId" value={jobId} />
          <input type="hidden" name="runId" value={runId} />
          {res.status === "error" && <p role="alert">{res.message}</p>}
          <div className="dialog-buttons">
            {/* autoFocus on the safe choice */}
            <button type="button" autoFocus onClick={() => ref.current?.close()}>Keep running</button>
            <button type="submit" className="danger" disabled={pending}>{pending ? "Stopping…" : "Stop run"}</button>
          </div>
        </form>
      </dialog>
    </>
  );
}
```

`<dialog>` + `showModal()` gives you the modal semantics for free in current browsers. Radix `AlertDialog` is the alternative when you need identical styling hooks everywhere or support for older browsers; it implements the WAI-ARIA alert dialog pattern. Either way: title and description wired up, focus lands on the non-destructive button, and the dialog says what will and won't happen.

## 8. Live updates

### The SSE proxy (Route Handlers)

```ts
// src/server/sse-proxy.ts
import "server-only";
import { env } from "@/env";
import { getViewer } from "@/server/auth/session";
import { getAccessToken, SessionExpiredError } from "@/server/auth/tokens";

const SSE_HEADERS = {
  "Content-Type": "text/event-stream; charset=utf-8",
  "Cache-Control": "no-cache, no-transform",   // no-transform: stop proxies from compressing/buffering
  "X-Accel-Buffering": "no",                   // nginx-style proxies: don't buffer
};

export async function proxySse(request: Request, upstreamPath: string, opts: { appendEnd?: boolean } = {}) {
  const viewer = await getViewer();
  if (!viewer) return new Response("Unauthorized", { status: 401 });

  let token: string;
  try { token = await getAccessToken(viewer); }
  catch (e) { if (e instanceof SessionExpiredError) return new Response("Session expired", { status: 401 }); throw e; }

  const headers = new Headers({ Authorization: `Bearer ${token}`, Accept: "text/event-stream" });
  const lastEventId = request.headers.get("last-event-id");   // EventSource sends it on reconnect
  if (lastEventId) headers.set("Last-Event-ID", lastEventId);

  const upstream = await fetch(`${env.JOBS_API_URL}${upstreamPath}`, {
    headers,
    cache: "no-store",
    signal: request.signal,          // browser disconnects -> upstream connection closes
  });

  if (!upstream.ok || !upstream.body) {
    return new Response(null, { status: upstream.status === 404 ? 404 : upstream.status === 403 ? 403 : 502 });
  }

  let body: ReadableStream<Uint8Array> = upstream.body;
  if (opts.appendEnd) {
    // When jobs-api closes a finished log stream, tell the browser explicitly;
    // otherwise EventSource would reconnect forever.
    const endFrame = new TextEncoder().encode("event: end\ndata: {}\n\n");
    body = body.pipeThrough(new TransformStream<Uint8Array, Uint8Array>({
      flush(controller) { controller.enqueue(endFrame); },
    }));
  }
  return new Response(body, { headers: SSE_HEADERS });
}
```

```ts
// src/app/api/jobs/events/route.ts
import { proxySse } from "@/server/sse-proxy";

export async function GET(request: Request) {
  return proxySse(request, "/api/jobs/events");
}
```

```ts
// src/app/api/jobs/[id]/runs/[runId]/logs/route.ts
import { proxySse } from "@/server/sse-proxy";

export async function GET(request: Request, { params }: { params: Promise<{ id: string; runId: string }> }) {
  const { id, runId } = await params;
  return proxySse(request, `/api/jobs/${encodeURIComponent(id)}/runs/${encodeURIComponent(runId)}/logs`, { appendEnd: true });
}
```

Notes:
- The Route Handler streams the upstream body straight through; nothing is buffered in Next.js. It needs the Node.js runtime (the default) and a server that supports streaming (`next start` / standalone does).
- **Auth happens at connect time.** An open stream outlives the access token. `jobs-api` should close streams when the token expires (or after N minutes); the browser reconnects, and the Route Handler forwards a freshly refreshed token.
- Every open tab holds one connection per stream on the Node process and one upstream. Budget for it (hundreds are fine; for thousands, fan out from one upstream connection per replica with an in-process broadcaster).

### `useEventSource`: reconnection that survives real networks

`EventSource` retries by itself after a network drop, but gives up for good when the server answers with a non-200 status or a wrong content type (for example a 401 or a 502 during a deploy). The hook adds its own backoff for that case, pauses while the tab is hidden, and reports the state for the UI.

```ts
// src/features/jobs/hooks/useEventSource.ts
"use client";
import { useEffect, useEffectEvent, useState } from "react";

type Status = "connecting" | "open" | "closed";

export function useEventSource(
  url: string,
  handlers: { onMessage: (data: string, lastEventId: string) => void; onEnd?: () => void },
  enabled = true,
): Status {
  const [status, setStatus] = useState<Status>("connecting");
  // React 19.2: latest handlers without re-subscribing on every render
  const onMessage = useEffectEvent(handlers.onMessage);
  const onEnd = useEffectEvent(() => handlers.onEnd?.());

  useEffect(() => {
    if (!enabled) { setStatus("closed"); return; }
    let es: EventSource | undefined;
    let attempt = 0;
    let timer: ReturnType<typeof setTimeout> | undefined;
    let disposed = false;

    const connect = () => {
      if (disposed || document.visibilityState === "hidden") return;
      setStatus("connecting");
      es = new EventSource(url);                           // same origin: session cookie is sent
      es.onopen = () => { attempt = 0; setStatus("open"); };
      es.onmessage = (e) => onMessage(e.data, e.lastEventId);
      es.addEventListener("end", () => { es?.close(); setStatus("closed"); onEnd(); disposed = true; });
      es.onerror = () => {
        if (es?.readyState === EventSource.CLOSED) {       // browser gave up: our turn
          setStatus("closed");
          const delay = Math.min(30_000, 1_000 * 2 ** attempt++) * (0.5 + Math.random() / 2); // backoff + jitter
          timer = setTimeout(connect, delay);
        } else {
          setStatus("connecting");                          // browser is retrying (uses `retry:` from server)
        }
      };
    };

    const onVisibility = () => {
      if (document.visibilityState === "hidden") { es?.close(); clearTimeout(timer); setStatus("closed"); }
      else if (!disposed && (!es || es.readyState === EventSource.CLOSED)) { attempt = 0; connect(); }
    };

    connect();
    document.addEventListener("visibilitychange", onVisibility);
    return () => {
      disposed = true;
      clearTimeout(timer);
      es?.close();
      document.removeEventListener("visibilitychange", onVisibility);
    };
  }, [url, enabled]);

  return status;
}
```

After a reconnect the stream may have missed events. Two defences: the server sends `id:` lines and honours `Last-Event-ID`, and the table does a `router.refresh()` whenever the connection comes back after being closed (cheap insurance, not shown).

### The log viewer

Requirements: stream `runs/:runId/logs`, strip ANSI colour codes (workers log with colours), autoscroll while the user is at the bottom, stop autoscrolling when they scroll up or press Pause, "Jump to latest", bounded memory, and no HTML injection.

```tsx
// src/features/jobs/components/LogViewer.tsx
"use client";
import { useLayoutEffect, useRef, useState } from "react";
import { useEventSource } from "../hooks/useEventSource";

// Same pattern as the `ansi-regex` package (CSI and OSC sequences)
const ANSI = /[\u001B\u009B][[\]()#;?]*(?:(?:(?:(?:;[-a-zA-Z\d/#&.:=?%@~_]+)*|[a-zA-Z\d]+(?:;[-a-zA-Z\d/#&.:=?%@~_]*)*)?\u0007)|(?:(?:\d{1,4}(?:;\d{0,4})*)?[\dA-PR-TZcf-nq-uy=><~]))/g;
const stripAnsi = (s: string) => s.replace(ANSI, "");
const MAX_LINES = 5_000;

type Line = { n: number; text: string };

export function LogViewer({ jobId, runId, live }: { jobId: string; runId: string; live: boolean }) {
  const [lines, setLines] = useState<Line[]>([]);
  const [follow, setFollow] = useState(true);
  const [ended, setEnded] = useState(false);
  const [unseen, setUnseen] = useState(0);
  const box = useRef<HTMLPreElement>(null);
  const buffer = useRef<Line[]>([]);
  const seq = useRef(0);
  const frame = useRef<number | null>(null);

  // Batch lines per animation frame: 500 lines/s must not mean 500 renders/s.
  const flush = () => {
    frame.current = null;
    const batch = buffer.current;
    buffer.current = [];
    setLines((prev) => {
      const next = prev.concat(batch);
      return next.length > MAX_LINES ? next.slice(next.length - MAX_LINES) : next;
    });
    setUnseen((u) => (follow ? 0 : u + batch.length));
  };

  const status = useEventSource(
    `/api/jobs/${encodeURIComponent(jobId)}/runs/${encodeURIComponent(runId)}/logs`,
    {
      onMessage(data) {
        for (const text of data.split("\n")) buffer.current.push({ n: seq.current++, text: stripAnsi(text) });
        frame.current ??= requestAnimationFrame(flush);
      },
      onEnd: () => setEnded(true),
    },
    !ended,
  );

  // Autoscroll after the DOM has the new lines, before paint
  useLayoutEffect(() => {
    if (follow && box.current) box.current.scrollTop = box.current.scrollHeight;
  }, [lines, follow]);

  const onScroll = () => {
    const el = box.current!;
    const atBottom = el.scrollHeight - el.scrollTop - el.clientHeight < 24;
    if (atBottom !== follow) { setFollow(atBottom); if (atBottom) setUnseen(0); }
  };

  const jumpToLatest = () => { setFollow(true); setUnseen(0); };

  return (
    <div className="log-viewer">
      <div className="log-toolbar">
        <span role="status">
          {ended ? "Run finished: full log shown" : status === "open" ? (live ? "Streaming…" : "Loading log…") : "Reconnecting…"}
          {lines.length >= MAX_LINES && ` (showing last ${MAX_LINES} lines)`}
        </span>
        <button type="button" aria-pressed={!follow} onClick={() => (follow ? setFollow(false) : jumpToLatest())}>
          {follow ? "⏸ Pause scroll" : "▶ Follow"}
        </button>
        {!follow && unseen > 0 && (
          <button type="button" onClick={jumpToLatest}>⤓ Jump to latest ({unseen} new)</button>
        )}
      </div>
      {/* role=log is a polite live region by default; a firehose would drown screen readers,
          so it is turned off and the status line above carries the important changes. */}
      <pre ref={box} role="log" aria-live="off" aria-label={`Log output of run ${runId}`}
        tabIndex={0} onScroll={onScroll} className="log-lines">
        {lines.map((l) => <div key={l.n}>{l.text}</div>)}
      </pre>
    </div>
  );
}
```

- Text is rendered as React text nodes: log lines containing `<script>` stay text. Never `dangerouslySetInnerHTML` log output (logs contain user input, supplier API bodies, stack traces).
- ANSI is stripped, not rendered. If you want colours, use a small ANSI-to-spans converter that outputs React elements, not HTML strings.
- 5,000 lines in the DOM is fine. For 100k-line logs, virtualize (`@tanstack/react-virtual`) or offer "Download full log" from Loki instead.
- Old runs: `jobs-api` replays the stored log then closes; the proxy appends `event: end`; the hook stops. Deep history lives in Loki with `job` and `run_id` labels ([Grafana & the LGTM Stack](/Software%20Engineering/DevOps/Delivery%20Platform/Grafana%20%26%20the%20LGTM%20Stack)); link out with a prebuilt Explore URL.

## 9. UI states

```tsx
// src/app/(console)/jobs/loading.tsx
import { JobsTableSkeleton } from "@/features/jobs/components/JobsTableSkeleton";
export default function Loading() {
  return (
    <div aria-busy="true" aria-live="polite">
      <h1>Scheduled jobs</h1>
      <span className="sr-only">Loading jobs…</span>
      <JobsTableSkeleton rows={4} />   {/* same column widths as the real table: no layout shift */}
    </div>
  );
}
```

```tsx
// src/app/(console)/jobs/error.tsx
"use client";
import { useEffect } from "react";

export default function JobsError({ error, reset }: { error: Error & { digest?: string }; reset: () => void }) {
  useEffect(() => { console.error(error); }, [error]);    // real apps: report to Sentry/OTel
  return (
    <div role="alert">
      <h1>Jobs are unavailable</h1>
      <p>We couldn't reach the job control plane (jobs-api). Scheduled runs are not affected by this page.</p>
      {error.digest && <p>Reference: <code>{error.digest}</code></p>}
      <button type="button" onClick={() => reset()}>Try again</button>
    </div>
  );
}
```

```tsx
// src/features/jobs/components/EmptyJobs.tsx
export function EmptyJobs() {
  return (
    <section className="empty">
      <h1>No scheduled jobs yet</h1>
      <p>Jobs appear here when they are registered with jobs-api: BullMQ job schedulers are upserted by
        <code> jobs-worker</code> at startup; Kubernetes CronJobs need the label <code>acme.dev/managed-by=jobs-api</code>.</p>
      <a href="https://runbooks.acme.dev/jobs/register">How to register a job</a>
    </section>
  );
}
```

```tsx
// src/app/(console)/jobs/[id]/not-found.tsx
import Link from "next/link";
export default function JobNotFound() {
  return (<><h1>Job not found</h1><p>It may have been renamed or removed.</p><Link href="/jobs">Back to all jobs</Link></>);
}
```

| State | Where | Notes |
|---|---|---|
| Loading | `loading.tsx` per route | skeleton matches the final layout; `aria-busy` |
| Error | `error.tsx` per segment | explain impact ("runs are not affected"), show digest, retry |
| Not found | `notFound()` → `not-found.tsx` | on 404 from `jobs-api` |
| Empty | page-level | say how to get data in |
| Partial | inline in `JobActions` | action errors as values, never a full-page error |
| Stale / offline | live indicator | "reconnecting…" when SSE is down; data may be old |

More patterns: [Next.js Error Handling, Loading & Not-Found](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Error%20Handling%2C%20Loading%20%26%20Not-Found).

## 10. Accessibility checklist

- **Status is never colour only**: icon + text + colour in `StatusBadge` (WCAG 1.4.1 Use of Color).
- **Live regions**: a polite `role="status"` announces finished runs and action results; errors use `role="alert"`. The log stream is `aria-live="off"` on purpose.
- **Dialog**: native `<dialog>` with `showModal()`, labelled and described, safe default focus, Esc closes, focus returns to the Stop button.
- **Disabled buttons explain why** via `aria-describedby` text. Screen readers and sighted users get the same reason.
- **Tables**: `<caption>`, `scope` on headers, the job name as a row header.
- **Keyboard**: scrollable log `<pre>` is focusable (`tabIndex=0`) so it can be scrolled with the keyboard; skip link to main content.
- Test with axe in Playwright (`@axe-core/playwright`) and a quick screen-reader pass per release.

## 11. Testing

### Unit and component tests (Vitest)

```ts
// vitest.config.mts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";
import tsconfigPaths from "vite-tsconfig-paths";

export default defineConfig({
  plugins: [tsconfigPaths(), react()],
  test: { environment: "jsdom", setupFiles: ["./tests/setup.ts"], include: ["tests/unit/**/*.test.{ts,tsx}"] },
});
```

```ts
// tests/setup.ts
import "@testing-library/jest-dom/vitest";
import { vi } from "vitest";
vi.mock("server-only", () => ({}));        // allow importing server modules in tests
```

```ts
// tests/unit/schemas.test.ts: the contract, with fixtures copied from jobs-api's OpenAPI examples
import { describe, it, expect } from "vitest";
import { JobSummary } from "@/features/jobs/schemas";

describe("JobSummary", () => {
  it("parses a paused K8s job without nextRunAt", () => {
    const job = JobSummary.parse({
      id: "cleanup-expired-carts", name: "cleanup-expired-carts", backend: "k8s",
      schedule: "0 * * * *", timezone: "Europe/Berlin", state: "paused", nextRunAt: null,
    });
    expect(job.state).toBe("paused");
  });
  it("rejects an unknown backend (contract drift)", () => {
    expect(() => JobSummary.parse({ id: "x", name: "x", backend: "airflow", schedule: "* * * * *",
      timezone: "UTC", state: "active", nextRunAt: null })).toThrow();
  });
});
```

```ts
// tests/unit/actions.test.ts: authorization and idempotency are enforced server side
import { describe, it, expect, vi, beforeEach } from "vitest";

const api = { pause: vi.fn(), resume: vi.fn(), runNow: vi.fn(), stopRun: vi.fn() };
vi.mock("next/cache", () => ({ revalidatePath: vi.fn() }));
vi.mock("@/server/auth/session", () => ({ requireViewer: vi.fn() }));
vi.mock("@/server/jobs-api/client", async (orig) => ({
  ...(await orig<typeof import("@/server/jobs-api/client")>()),
  jobsApi: () => api,
}));

import { requireViewer } from "@/server/auth/session";
import { pauseJob, runJobNow } from "@/features/jobs/actions";

const fd = (o: Record<string, string>) => { const f = new FormData(); for (const [k, v] of Object.entries(o)) f.set(k, v); return f; };
const key = "0f8fad5b-d9cb-469f-a165-70867728950e";

beforeEach(() => vi.clearAllMocks());

describe("job actions", () => {
  it("refuses read-only users without calling jobs-api", async () => {
    vi.mocked(requireViewer).mockResolvedValue({ userId: "u1", email: "v@acme.dev", roles: ["viewer"], sessionIdHash: "h" });
    const res = await pauseJob({ status: "idle" }, fd({ jobId: "sync-inventory", idempotencyKey: key }));
    expect(res).toMatchObject({ status: "error" });
    expect(api.pause).not.toHaveBeenCalled();
  });

  it("forwards the Idempotency-Key for Run now", async () => {
    vi.mocked(requireViewer).mockResolvedValue({ userId: "u2", email: "o@acme.dev", roles: ["operator"], sessionIdHash: "h" });
    api.runNow.mockResolvedValue({ runId: "8f2c0000-run" });
    const res = await runJobNow({ status: "idle" }, fd({ jobId: "sync-inventory", idempotencyKey: key }));
    expect(api.runNow).toHaveBeenCalledWith("sync-inventory", key);
    expect(res).toMatchObject({ status: "ok" });
  });

  it("rejects malformed job ids", async () => {
    vi.mocked(requireViewer).mockResolvedValue({ userId: "u2", email: "o@acme.dev", roles: ["operator"], sessionIdHash: "h" });
    const res = await pauseJob({ status: "idle" }, fd({ jobId: "../../admin", idempotencyKey: key }));
    expect(res).toMatchObject({ status: "error", message: "Invalid request." });
  });
});
```

```tsx
// tests/unit/JobActions.test.tsx
import { render, screen } from "@testing-library/react";
import { vi, it, expect } from "vitest";
vi.mock("@/features/jobs/actions", () => ({ pauseJob: vi.fn(), resumeJob: vi.fn(), runJobNow: vi.fn(), stopRun: vi.fn() }));
import { JobActions } from "@/features/jobs/components/JobActions";

const job = { id: "sync-inventory", name: "sync-inventory", backend: "bullmq" as const, schedule: "*/15 * * * *",
  timezone: "Europe/Berlin", state: "active" as const, nextRunAt: null,
  lastRun: { runId: "r1", status: "running" as const, startedAt: "2026-09-27T12:30:00+02:00" } };

it("disables everything for read-only users and says why", () => {
  render(<JobActions job={job} canWrite={false} forbidOverlap />);
  for (const b of screen.getAllByRole("button")) expect(b).toBeDisabled();
  expect(screen.getByText(/Read-only access/)).toBeInTheDocument();
});

it("disables Run now while a run is in flight (Forbid)", () => {
  render(<JobActions job={job} canWrite forbidOverlap />);
  expect(screen.getByRole("button", { name: "Run now" })).toBeDisabled();
  expect(screen.getByRole("button", { name: "Stop run" })).toBeEnabled();
});
```

Async Server Components themselves are best covered by E2E tests; see [Next.js Testing - Unit, Integration & E2E](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Testing%20-%20Unit%2C%20Integration%20%26%20E2E).

### E2E (Playwright) against a stub `jobs-api`

Server Actions and Route Handlers call `jobs-api` from the **Next.js server**, so `page.route()` in the browser can't intercept them. Run a small stub server instead and point `JOBS_API_URL` at it. For login, CI starts the IdP in a container (e.g. Keycloak with a realm export containing `e2e-viewer` and `e2e-operator` test users); a setup project logs in once per role and saves `storageState`.

```ts
// playwright.config.ts
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  testDir: "tests/e2e",
  retries: process.env.CI ? 2 : 0,
  use: { baseURL: "http://localhost:3000", trace: "on-first-retry" },
  webServer: [
    { command: "node tests/e2e/stub-jobs-api.mjs", port: 4010, reuseExistingServer: !process.env.CI },
    {
      command: "npm run build && npm run start",
      port: 3000,
      env: { JOBS_API_URL: "http://localhost:4010", OIDC_ISSUER: process.env.E2E_OIDC_ISSUER ?? "http://localhost:8081/realms/acme-e2e" },
      reuseExistingServer: !process.env.CI,
      timeout: 180_000,
    },
  ],
  projects: [
    { name: "setup", testMatch: /auth\.setup\.ts/ },
    { name: "operator", dependencies: ["setup"], testMatch: /operator\..*spec\.ts/, use: { ...devices["Desktop Chrome"], storageState: "playwright/.auth/operator.json" } },
    { name: "viewer", dependencies: ["setup"], testMatch: /viewer\..*spec\.ts/, use: { ...devices["Desktop Chrome"], storageState: "playwright/.auth/viewer.json" } },
  ],
});
```

```ts
// tests/e2e/operator.jobs.spec.ts
import { test, expect } from "@playwright/test";
import AxeBuilder from "@axe-core/playwright";

test("pause is instant, sends an Idempotency-Key and survives reload", async ({ page, request }) => {
  await page.goto("/jobs");
  const row = page.getByRole("row", { name: /nightly-sales-report/ });
  await row.getByRole("button", { name: "Pause schedule" }).click();
  await expect(row.getByText("Paused")).toBeVisible();                 // optimistic, then confirmed
  await expect(page.getByRole("status").filter({ hasText: "Paused schedule" })).toBeVisible();

  const calls = await (await request.get("http://localhost:4010/__calls")).json();
  expect(calls.at(-1)).toMatchObject({ path: "/api/jobs/nightly-sales-report/pause", idempotencyKey: expect.any(String) });

  await page.reload();
  await expect(row.getByRole("button", { name: "Resume schedule" })).toBeEnabled();
});

test("stop asks for confirmation and Esc cancels", async ({ page }) => {
  await page.goto("/jobs/sync-inventory");
  const stop = page.getByRole("button", { name: "Stop run" }).first();
  await stop.click();
  const dialog = page.getByRole("dialog", { name: /Stop this run of sync-inventory/ });
  await expect(dialog).toBeVisible();
  await expect(dialog.getByRole("button", { name: "Keep running" })).toBeFocused();
  await page.keyboard.press("Escape");
  await expect(dialog).toBeHidden();
  await expect(stop).toBeFocused();                                    // focus restored
});

test("live status update is announced", async ({ page, request }) => {
  await page.goto("/jobs");
  await request.post("http://localhost:4010/__emit", {
    data: { type: "run.updated", jobId: "sync-inventory", runId: "r9", status: "succeeded" },
  });
  await expect(page.getByRole("status").filter({ hasText: "sync-inventory: run succeeded" })).toBeAttached();
});

test("jobs page has no axe violations", async ({ page }) => {
  await page.goto("/jobs");
  const results = await new AxeBuilder({ page }).analyze();
  expect(results.violations).toEqual([]);
});
```

```ts
// tests/e2e/viewer.jobs.spec.ts
import { test, expect } from "@playwright/test";

test("read-only users see disabled controls, and the action is refused server side", async ({ page }) => {
  await page.goto("/jobs");
  const row = page.getByRole("row", { name: /sync-inventory/ });
  await expect(row.getByRole("button", { name: "Run now" })).toBeDisabled();
  await expect(row.getByText(/Read-only access/)).toBeVisible();
  // Server-side enforcement is covered by the actions unit test; a forged POST must also fail (manual pen test / DAST).
});
```

## 12. Deployment on Kubernetes next to `jobs-api`

Image: the standard `output: 'standalone'` multi-stage build, non-root, `node server.js` (full Dockerfile in [Deploying Next.js Micro-frontends to Kubernetes](/Software%20Engineering/DevOps/Delivery%20Platform/Deploying%20Next.js%20Micro-frontends%20to%20Kubernetes)). Manifests in `platform-gitops/apps/ops-console/`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: ops-console, namespace: shop-prod, labels: { app: ops-console } }
spec:
  replicas: 2
  selector: { matchLabels: { app: ops-console } }
  template:
    metadata: { labels: { app: ops-console } }
    spec:
      securityContext: { runAsNonRoot: true, runAsUser: 1001, fsGroup: 1001 }
      containers:
        - name: web
          image: ghcr.io/acme/ops-console:placeholder     # bumped by Jenkins
          ports: [{ containerPort: 3000, name: http }]
          env:
            - { name: JOBS_API_URL, value: "http://jobs-api.shop-prod.svc.cluster.local:8080" }
            - { name: OIDC_ISSUER, value: "https://sso.acme.dev/realms/acme" }
            - { name: OIDC_CLIENT_ID, value: "ops-console" }
            - { name: HOSTNAME, value: "0.0.0.0" }
          envFrom: [{ secretRef: { name: ops-console-secrets } }]   # OIDC_CLIENT_SECRET, DATABASE_URL
          readinessProbe: { httpGet: { path: /api/health, port: http }, periodSeconds: 5 }
          livenessProbe: { httpGet: { path: /api/health, port: http }, periodSeconds: 10, failureThreshold: 6 }
          resources:
            requests: { cpu: 100m, memory: 256Mi }
            limits: { memory: 512Mi }
          securityContext: { allowPrivilegeEscalation: false, readOnlyRootFilesystem: true, capabilities: { drop: [ALL] } }
          volumeMounts: [{ name: next-cache, mountPath: /app/.next/cache }]
      volumes: [{ name: next-cache, emptyDir: {} }]
      terminationGracePeriodSeconds: 30     # open SSE streams are cut; browsers reconnect to the other Pod
---
apiVersion: v1
kind: Service
metadata: { name: ops-console, namespace: shop-prod }
spec: { selector: { app: ops-console }, ports: [{ port: 80, targetPort: http }] }
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: { name: ops-console, namespace: shop-prod }
spec:
  parentRefs: [{ name: internal-gateway, namespace: gateway }]   # internal/VPN-only listener
  hostnames: ["ops.acme.dev"]
  rules:
    - matches: [{ path: { type: PathPrefix, value: /api/jobs } }]   # SSE: no request timeout
      timeouts: { request: "0s" }        # 0s disables the timeout per the spec; check your implementation
      backendRefs: [{ name: ops-console, port: 80 }]
    - matches: [{ path: { type: PathPrefix, value: / } }]
      timeouts: { request: 30s }
      backendRefs: [{ name: ops-console, port: 80 }]
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata: { name: ops-console-secrets, namespace: shop-prod }
spec:
  refreshInterval: 1h
  secretStoreRef: { kind: ClusterSecretStore, name: aws-secrets-manager }
  target: { name: ops-console-secrets }
  data:
    - { secretKey: OIDC_CLIENT_SECRET, remoteRef: { key: prod/ops-console/oidc, property: client_secret } }
    - { secretKey: DATABASE_URL, remoteRef: { key: prod/ops-console/db, property: url } }
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: jobs-api-from-ops-console, namespace: shop-prod }
spec:
  podSelector: { matchLabels: { app: jobs-api } }
  policyTypes: [Ingress]
  ingress:
    - from: [{ podSelector: { matchLabels: { app: ops-console } } }]
      ports: [{ port: 8080 }]
```

Deployment notes:
- **Two replicas are fine without sticky sessions**: sessions live in the DB, and each SSE connection is independent. With several replicas also set a shared `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` (and the same build), so Server Action IDs and encrypted closures are valid on every Pod.
- **Streaming through proxies**: disable response buffering and long-request timeouts only on the SSE paths. Send an SSE comment (`: ping`) every ~15 s from `jobs-api` so idle-timeout middleboxes don't cut the stream.
- **Internal only**: expose `ops.acme.dev` on an internal gateway or behind the VPN / zero-trust proxy. A control plane for production jobs is a high-value target.
- **Release**: Jenkins builds and bumps the tag in `platform-gitops`, Argo CD syncs ([Kubernetes Delivery Platform - End to End](/Software%20Engineering/DevOps/Delivery%20Platform/Kubernetes%20Delivery%20Platform%20-%20End%20to%20End)).

## 13. Build or buy: alternatives

| Tool | What it gives you | Fits when | Gaps for Acme Ops |
|---|---|---|---|
| **Bull Board** (`@bull-board/*`) | Queue UI for Bull/BullMQ: jobs by state, retry, promote, clean, pause queues; mounts into Express/Fastify/Nest/Hono | Only BullMQ, internal users, quick win | Queue-centric (not schedule-centric), no K8s CronJobs, coarse auth (you wrap it), no per-job audit |
| **Taskforce.sh** | Hosted/commercial BullMQ dashboard from the BullMQ authors | BullMQ fleets, want metrics and alerts without building | BullMQ only |
| **Temporal UI** | Workflow history, Schedules with pause, trigger, backfill | Jobs are already Temporal workflows | Only Temporal |
| **Headlamp** | Kubernetes web UI (now the SIG UI project); view CronJobs/Jobs, logs, plugins | Platform engineers who think in K8s objects | K8s only, exposes cluster permissions, not a product UI |
| **Kubernetes Dashboard** | The old K8s web UI | Legacy clusters | **Archived (Jan 2026)**; migrate to Headlamp |
| **Argo CD / Argo Workflows UI** | Resource tree, sync; CronWorkflows with suspend/resume | GitOps teams; Argo Workflows users | Not job-level operations for BullMQ |
| **Build your own (`ops-console`)** | One view across backends, your roles, your audit trail, your runbooks | Several backends, non-platform users (support, finance), strict RBAC/audit | You own it: code, security patches, on-call |

Rule of thumb: start with the backend's own UI (Bull Board, Headlamp). Build a console when you have **more than one backend**, users who shouldn't get cluster or Redis access, or compliance needs (who stopped which run, when, why). Keep the console thin: all logic lives in `jobs-api`, so a CLI or chatops bot can use the same API.

## Common pitfalls

<Warning>
**Pitfalls**

- **Auth only in the layout or `proxy.ts`.** Server Actions and Route Handlers are reachable directly. Check in every action and handler (and again in `jobs-api`).
- **Access token in the browser** (returned from a session endpoint or put in a client component's props) just to call `jobs-api` from the client. Use the BFF proxy.
- **Refresh-token races** with rotating refresh tokens: two tabs refresh at once and the session dies. Serialize refreshes.
- **SSE through default proxies**: buffering (events arrive in bursts or never), 30–60 s request timeouts, compression. Set `no-transform`, disable buffering, add heartbeats, raise timeouts on the SSE route only.
- **`EventSource` reconnect loops** on finished streams, or permanent death after one 502. Send an explicit `end` event; add your own backoff on `CLOSED`.
- **Hydration mismatches** from `new Date().toLocaleString()` or `crypto.randomUUID()` during render. Format with a fixed locale and timezone; create keys in event handlers.
- **Optimistic state that never reconciles** (a manual `setState` that ignores the server). `useOptimistic` resets to server truth automatically.
- **Idempotency key per render or per page load**: two clicks share a key and the second "Run now" is silently dropped; or a new key per retry, and a retry double-runs.
- **Confusing Pause with Stop.** Label them "Pause schedule" and "Stop run", and say in the dialog what isn't affected.
- **Strict zod enums without a plan**: `jobs-api` adds a `skipped` status and the whole page 500s. Either deploy the console first, or parse unknown values into an "Unknown" badge and alert on it.
- **Rendering logs as HTML.** Log lines are untrusted input.
</Warning>

## Senior perspective

- **The console is a control plane for production.** Treat it like one: internal-only exposure, SSO with MFA at the IdP, least-privilege roles, per-action audit in `jobs-api` (actor, job, run, idempotency key, reason), and alerts on unusual use (Stop pressed 20 times in a minute).
- **Put semantics in the API, not the UI.** The UI only reflects `state` and `status`; the `409` rules (can't resume an active job, can't run a `Forbid` job twice) live in `jobs-api`, so the CLI and chatops behave the same.
- **Optimism with humility.** Optimistic UI for reversible, fast actions (pause, resume, run now). For destructive ones (stop), confirm first and show "Stopping…" until the server says `cancelled`; cancellation is cooperative and can take the whole grace period.
- **Scale of live updates**: one SSE upstream per browser tab is fine for dozens of operators. If the console becomes a wallboard for hundreds, multiplex: one upstream connection per replica and an in-process broadcaster, or push via a managed pub/sub.
- **Failure isolation**: if `jobs-api` is down, the console must say so clearly, and scheduled runs must continue. The console is never on the critical path of job execution.
- **Questions to ask in review**: What happens if two operators press Run now at the same moment? If the SSE stream silently stalls, how does the user know the data is stale? Can a viewer reach any mutation by crafting a POST? Where is the audit trail and who reviews it? What does the page do when `jobs-api` returns a status it doesn't know?

<Info>
**🆕 2025–2026**

- **2025-09-22:** Auth.js (NextAuth) joined **Better Auth** and entered maintenance mode; new projects are pointed to Better Auth or a plain OIDC client such as `openid-client` v6 (the approach used here). See [Next.js Authentication & Authorization](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Authentication%20%26%20Authorization).
- **2025-10-01:** **React 19.2** made `useEffectEvent` stable; `useEventSource` uses it to read the latest handlers without re-subscribing.
- **2025-10-21:** **Next.js 16**: `proxy.ts` replaces `middleware.ts`, async request APIs are mandatory (`params`, `searchParams` are Promises), caching is opt-in, and `refresh()` / `updateTag()` were added for Server Actions. See [Next.js 15 to 16.3](/Software%20Engineering/Frontend/07%20Next.js/Next.js%2015%20to%2016.3).
- **2026-01-21:** the **Kubernetes Dashboard was archived**; Headlamp (under SIG UI) is the recommended web UI. The Kubernetes blog explained the transition on 2026-06-01.
- **2026-09-22:** **Next.js 16.3.6** fixed a critical RCE in `next/og` `ImageResponse` (16.2.0 to 16.3.5). An internal ops console is still worth patching the same week.
</Info>

## Interview questions

<Accordion title="Why call jobs-api from the Next.js server instead of from the browser?">
The access token stays server-side (no token theft through XSS), `jobs-api` isn't exposed publicly, there's no CORS, and responses are validated in one place. The browser only has an httpOnly session cookie. Live updates go through a Route Handler that proxies SSE and adds the bearer token.
</Accordion>

<Accordion title="How do useActionState and useOptimistic work together here?">
The function given to `useActionState` runs inside a transition when the form submits. It first applies an optimistic patch (e.g. state = paused), then awaits the Server Action. While the transition is pending, `useOptimistic` shows the patched value; when it finishes, React discards the patch and renders the real props, which `revalidatePath` refreshed in the same response. On failure it snaps back automatically and the returned error is rendered.
</Accordion>

<Accordion title="Where do you generate the Idempotency-Key and why there?">
In the browser at submit time, one per click, added to the FormData. The Server Action forwards it to `jobs-api`, and any retry of that request reuses it, so a lost response can't start a second run. Generating it during render would mismatch between SSR and hydration and would be shared by repeated clicks.
</Accordion>

<Accordion title="A read-only user's buttons are disabled. Is that enough?">
No. Disabled buttons are UX. Server Actions are public POST endpoints, so each action verifies the session and the `ops:jobs:write` permission, and `jobs-api` validates the token's role again. The UI shows why buttons are disabled so users understand the permission model.
</Accordion>

<Accordion title="How do you make an SSE-based live view reliable?">
Proxy with streaming and no buffering, `Cache-Control: no-transform`, heartbeats against idle timeouts, long timeouts only on the SSE route, `id:` + `Last-Event-ID` for resume, an explicit end event for finite streams, client backoff with jitter when `EventSource` gives up on non-200 responses, pause on hidden tabs, and a full refresh after reconnecting. Show connection state so users know when data might be stale.
</Accordion>

<Accordion title="How would you build the log viewer so it stays fast and safe?">
Batch incoming lines per animation frame, keep a bounded buffer (or virtualize), strip ANSI codes, render as text nodes (never HTML), autoscroll only when the user is at the bottom, offer pause/jump-to-latest, and turn off live-region announcements for the stream itself. Link to Loki for full history.
</Accordion>

<Accordion title="When would you use Bull Board or Headlamp instead of building a dashboard?">
When there's one backend and the users are engineers who already have access to it: Bull Board for BullMQ queues, Headlamp for Kubernetes objects. Build your own when you have several backends behind one API, non-engineering users, strict RBAC and audit requirements, or workflows (confirmation, reasons, runbooks) the generic tools don't support.
</Accordion>

<Accordion title="How do you test Server Actions that call another service?">
Unit-test the action with the session and API client mocked: read-only users are refused without an API call, inputs are validated, the idempotency key is forwarded, and expected API errors become values. E2E-test with a stub service started by Playwright's `webServer`, because server-side fetches can't be intercepted by `page.route()`.
</Accordion>

## Related
- [Scheduled Jobs Index](/Software%20Engineering/Backend/Scheduled%20Jobs/Scheduled%20Jobs%20Index) · [Backend Index](/Software%20Engineering/Backend/Backend%20Index)
- [Job Control API - Pause, Resume, Run Now & Stop](/Software%20Engineering/Backend/Scheduled%20Jobs/Job%20Control%20API%20-%20Pause%2C%20Resume%2C%20Run%20Now%20%26%20Stop) · [Cron with Message Queues - Architecture](/Software%20Engineering/Backend/Scheduled%20Jobs/Cron%20with%20Message%20Queues%20-%20Architecture) · [BullMQ Job Schedulers in Practice](/Software%20Engineering/Backend/Scheduled%20Jobs/BullMQ%20Job%20Schedulers%20in%20Practice) · [Kubernetes CronJobs in Practice](/Software%20Engineering/Backend/Scheduled%20Jobs/Kubernetes%20CronJobs%20in%20Practice) · [Playwright Automation as a Cron Job](/Software%20Engineering/Backend/Scheduled%20Jobs/Playwright%20Automation%20as%20a%20Cron%20Job)
- Next.js: [Next.js App Router Fundamentals](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20App%20Router%20Fundamentals) · [Next.js Authentication & Authorization](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Authentication%20%26%20Authorization) · [Next.js Data Fetching & Mutations Patterns](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Data%20Fetching%20%26%20Mutations%20Patterns) · [Next.js Route Handlers & Backend-for-Frontend](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Route%20Handlers%20%26%20Backend-for-Frontend) · [Next.js Error Handling, Loading & Not-Found](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Error%20Handling%2C%20Loading%20%26%20Not-Found) · [Next.js Testing - Unit, Integration & E2E](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Testing%20-%20Unit%2C%20Integration%20%26%20E2E) · [Next.js Project Structure & Architecture](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Project%20Structure%20%26%20Architecture) · [Next.js 15 to 16.3](/Software%20Engineering/Frontend/07%20Next.js/Next.js%2015%20to%2016.3)
- Platform: [Deploying Next.js Micro-frontends to Kubernetes](/Software%20Engineering/DevOps/Delivery%20Platform/Deploying%20Next.js%20Micro-frontends%20to%20Kubernetes) · [Kubernetes Delivery Platform - End to End](/Software%20Engineering/DevOps/Delivery%20Platform/Kubernetes%20Delivery%20Platform%20-%20End%20to%20End) · [Grafana & the LGTM Stack](/Software%20Engineering/DevOps/Delivery%20Platform/Grafana%20%26%20the%20LGTM%20Stack) · [Argo CD in Practice](/Software%20Engineering/DevOps/Delivery%20Platform/Argo%20CD%20in%20Practice)
- [Background Jobs & Task Queues](/Software%20Engineering/Backend/Background%20Jobs%20%26%20Task%20Queues) · [Cron Jobs](/Software%20Engineering/Backend/Cron%20Jobs) · [Frontend Testing](/Software%20Engineering/Frontend/13%20Testing/Frontend%20Testing) · [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time) · [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types)

## Sources
- Next.js docs: [Server Actions / updating data](https://nextjs.org/docs/app/getting-started/updating-data) · [Route Handlers](https://nextjs.org/docs/app/api-reference/file-conventions/route) · [refresh](https://nextjs.org/docs/app/api-reference/functions/refresh) · [revalidatePath](https://nextjs.org/docs/app/api-reference/functions/revalidatePath) · [loading.js](https://nextjs.org/docs/app/api-reference/file-conventions/loading) · [error.js](https://nextjs.org/docs/app/api-reference/file-conventions/error) · [proxy.js](https://nextjs.org/docs/app/api-reference/file-conventions/proxy) · [Authentication guide](https://nextjs.org/docs/app/guides/authentication) · [Self-hosting (Server Actions encryption key, streaming)](https://nextjs.org/docs/app/guides/self-hosting) · [Next.js 16 blog](https://nextjs.org/blog/next-16) · [Security update 2026-09-22](https://nextjs.org/blog/nextjs-security-update-september-22-2026)
- React: [useActionState](https://react.dev/reference/react/useActionState) · [useOptimistic](https://react.dev/reference/react/useOptimistic) · [useEffectEvent](https://react.dev/reference/react/useEffectEvent) · [React 19.2 release (2025-10-01)](https://react.dev/blog/2025/10/01/react-19-2)
- HTML/Web: [WHATWG: Server-sent events](https://html.spec.whatwg.org/multipage/server-sent-events.html) · [MDN: EventSource](https://developer.mozilla.org/en-US/docs/Web/API/EventSource) · [MDN: `<dialog>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/dialog) · [WAI-ARIA APG: Alert dialog pattern](https://www.w3.org/WAI/ARIA/apg/patterns/alertdialog/) · [MDN: ARIA live regions](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/ARIA_Live_Regions) · [WCAG 2.2 SC 1.4.1 Use of Color](https://www.w3.org/WAI/WCAG22/Understanding/use-of-color.html) · [Radix AlertDialog](https://www.radix-ui.com/primitives/docs/components/alert-dialog)
- [RFC 9457: Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457) · [IETF draft: The Idempotency-Key HTTP header](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/)
- Libraries: [zod](https://zod.dev/) · [cronstrue](https://github.com/bradymholt/cRonstrue) · [ansi-regex](https://github.com/chalk/ansi-regex) · [openid-client](https://github.com/panva/openid-client) · [Better Auth: Auth.js joins Better Auth (2025-09-22)](https://better-auth.com/blog/authjs-joins-better-auth) · [@axe-core/playwright](https://github.com/dequelabs/axe-core-npm/tree/develop/packages/playwright)
- Alternatives: [Bull Board](https://github.com/felixmosh/bull-board) · [Taskforce.sh](https://taskforce.sh/) · [Temporal: Schedules](https://docs.temporal.io/schedule) · [Headlamp](https://headlamp.dev/) · [Kubernetes blog: From Kubernetes Dashboard to Headlamp (2026-06-01)](https://kubernetes.io/blog/2026/06/01/dashboard-to-headlamp/)
- Kubernetes: [Gateway API HTTPRoute timeouts](https://gateway-api.sigs.k8s.io/api-types/httproute/) · [External Secrets Operator](https://external-secrets.io/)
