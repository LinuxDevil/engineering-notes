---
title: "Job Control API - Pause, Resume, Run Now & Stop"
tags: [backend, backend/jobs, backend/node, devops/kubernetes]
aliases: [Job Control API, jobs-api, Job Control Plane, JobBackend, Pause Resume Run Now Stop, K8sCronJobBackend, BullMqBackend]
updated: 2026-09-27
---
# Job Control API - Pause, Resume, Run Now & Stop

<Note>
**TL;DR**

**`jobs-api`** is a small Fastify service that gives operators one API for every scheduled job, whether it runs on **BullMQ** or as a **Kubernetes CronJob**. A `JobBackend` interface hides the difference: *pause* removes the BullMQ scheduler or sets `spec.suspend: true`; *resume* re-creates it without backfill; *run now* enqueues a job whose id comes from the `Idempotency-Key` or creates a Job from the CronJob's `jobTemplate`; *stop* cancels cooperatively (BullMQ `AbortSignal` via Redis pub/sub) or deletes the K8s Job with foreground propagation. Around it: OIDC JWT verification with `jose`, per-route RBAC, idempotency keys, RFC 9457 Problem Details, an append-only audit table, rate limits, SSE for logs and live events, and a least-privilege K8s Role.
</Note>

It's the control plane of Acme Ops. The worker side is in [BullMQ Job Schedulers in Practice](/Software%20Engineering/Backend/Scheduled%20Jobs/BullMQ%20Job%20Schedulers%20in%20Practice), the CronJobs in [Kubernetes CronJobs in Practice](/Software%20Engineering/Backend/Scheduled%20Jobs/Kubernetes%20CronJobs%20in%20Practice), the UI in [Job Scheduler Dashboard with Next.js](/Software%20Engineering/Backend/Scheduled%20Jobs/Job%20Scheduler%20Dashboard%20with%20Next.js), and the reasoning (idempotency, overlap, misfires) in [Cron with Message Queues - Architecture](/Software%20Engineering/Backend/Scheduled%20Jobs/Cron%20with%20Message%20Queues%20-%20Architecture).

## 1. Resource model

| Resource | Identity | Owned by | Mutable via API |
|---|---|---|---|
| **Job** (definition) | `id`, e.g. `sync-inventory` | Git: the registry (`@acme/jobs-core`) and the CronJob manifests | no (schedule changes go through Git review) |
| **Job state** | `active` / `paused` | `jobs-api` (Redis flag for BullMQ, `spec.suspend` for K8s) | pause / resume |
| **Run** | `runId`, opaque string | the backend (BullMQ job, K8s Job) | create (run now), stop |
| **Audit event** | bigint id | Postgres `job_audit_log` | append only |

`runId` values are **opaque** to clients: BullMQ scheduled runs look like `repeat:sync-inventory:1790500500000`, manual BullMQ runs like `manual-3f9c…`, K8s runs are Job names like `cleanup-expired-carts-29316000`. Clients must `encodeURIComponent` them in URLs.

### The API contract

All under `/api/jobs`, JSON, `Authorization: Bearer <OIDC access token>`. Reads need `ops:jobs:read`; mutations need `ops:jobs:write` and are audited.

| Method & path | Body / query | Response |
|---|---|---|
| `GET /api/jobs` | | `200 JobSummary[]` |
| `GET /api/jobs/:id` | | `200 JobDetail` |
| `POST /api/jobs/:id/pause` | optional `{ reason? }` | `204` |
| `POST /api/jobs/:id/resume` | optional `{ reason? }` | `204` |
| `POST /api/jobs/:id/runs` | `{ input? }`, header **`Idempotency-Key`** (required) | `202 { runId }` |
| `POST /api/jobs/:id/runs/:runId/stop` | optional `{ reason? }` | `202` (status → `cancelling` → `cancelled`) |
| `GET /api/jobs/:id/runs?limit=&cursor=` | `limit` 1–100 (default 20) | `200 { items: JobRun[], nextCursor }` |
| `GET /api/jobs/:id/runs/:runId/logs` | | `text/event-stream`: one `data:` event per log line, then `event: end` |
| `GET /api/jobs/events` | header `Last-Event-ID` (optional) | `text/event-stream`: `data: { type: 'run.updated', jobId, runId, status?, progress? }` or `{ type: 'job.updated', jobId, state? }` |

```ts
type JobSummary = {
  id: string; name: string; backend: 'bullmq' | 'k8s';
  schedule: string; timezone: string; state: 'active' | 'paused';
  nextRunAt: string | null;                        // ISO 8601; null while paused
  lastRun?: { runId: string; status: RunStatus; startedAt: string; finishedAt: string | null };
};
type JobDetail = JobSummary & {
  description: string; owner: string;
  concurrencyPolicy: 'Allow' | 'Forbid' | 'Replace'; timeoutSeconds: number; retries: number;
};
type RunStatus = 'queued' | 'running' | 'succeeded' | 'failed' | 'cancelling' | 'cancelled';
type JobRun = {
  runId: string; status: RunStatus; trigger: 'schedule' | 'manual';
  startedAt: string;                               // enqueue/creation time while queued, start time once running
  finishedAt?: string; durationMs?: number; error?: string; attempts: number;
};
type Page<T> = { items: T[]; nextCursor: string | null };
```

## 2. State machines

{/* diagram source: /diagrams/diagram-062.mmd */}
![diagram](/diagrams/diagram-062.svg)

{/* diagram source: /diagrams/diagram-063.mmd */}
![diagram](/diagrams/diagram-063.svg)

Rules that every note and the dashboard rely on:

- **Pause** stops *new scheduled* runs only. In-flight runs continue; manual runs still work.
- **Resume** re-enables the schedule; missed ticks are **not** backfilled (BullMQ: the scheduler is re-created from the next pattern match; K8s: `startingDeadlineSeconds` bounds catch-up, see [Kubernetes CronJobs in Practice](/Software%20Engineering/Backend/Scheduled%20Jobs/Kubernetes%20CronJobs%20in%20Practice)).
- **Stop** is **cooperative** and asynchronous: `202` means "cancellation requested". A run can still end as `succeeded` if it finished first.
- **Overlap** is `Forbid` by default (K8s `concurrencyPolicy`; BullMQ global concurrency 1 + misfire grace).

## 3. Project layout

```text
apps/jobs-api/
├── src/
│   ├── server.ts                 build the Fastify app (plugins, routes); exported for tests
│   ├── main.ts                   listen, graceful shutdown
│   ├── schemas.ts                TypeBox schemas = the contract above
│   ├── problems.ts               RFC 9457 errors
│   ├── auth.ts                   OIDC JWT verification (jose) + RBAC
│   ├── idempotency.ts            Idempotency-Key store (Postgres)
│   ├── audit.ts                  append-only audit log
│   ├── sse.ts                    SSE helper
│   ├── events.ts                 EventHub: QueueEvents + K8s watch + Redis fan-out
│   ├── backends/
│   │   ├── backend.ts            JobBackend interface + JobService router
│   │   ├── bullmq.ts             BullMqBackend
│   │   └── k8s.ts                K8sCronJobBackend
│   └── routes/jobs.ts
├── migrations/001_init.sql
├── k8s/{serviceaccount,role,rolebinding,admission-policy}.yaml
└── test/
```

```bash
pnpm add fastify @fastify/type-provider-typebox typebox @fastify/rate-limit jose pg ioredis bullmq \
         @kubernetes/client-node cron-parser pino
pnpm add -D vitest @testcontainers/postgresql @testcontainers/redis typescript @types/node @types/pg
```

Versions at the time of writing: Fastify 5.12, `@fastify/type-provider-typebox` 6.1 (uses the `typebox` 1.x package), jose 6.2, `@kubernetes/client-node` 2.0, BullMQ 6.3, Node 24 LTS.

## 4. Schemas (TypeBox)

```ts
// src/schemas.ts
import Type from 'typebox';

const IsoDate = Type.String({ format: 'date-time' });
export const RunStatus = Type.Union(
  ['queued', 'running', 'succeeded', 'failed', 'cancelling', 'cancelled'].map((s) => Type.Literal(s)),
);
export const JobRun = Type.Object({
  runId: Type.String(),
  status: RunStatus,
  trigger: Type.Union([Type.Literal('schedule'), Type.Literal('manual')]),
  startedAt: IsoDate,
  finishedAt: Type.Optional(IsoDate),
  durationMs: Type.Optional(Type.Integer({ minimum: 0 })),
  error: Type.Optional(Type.String()),
  attempts: Type.Integer({ minimum: 0 }),
});
export const JobSummary = Type.Object({
  id: Type.String(), name: Type.String(),
  backend: Type.Union([Type.Literal('bullmq'), Type.Literal('k8s')]),
  schedule: Type.String(), timezone: Type.String(),
  state: Type.Union([Type.Literal('active'), Type.Literal('paused')]),
  nextRunAt: Type.Union([IsoDate, Type.Null()]),
  lastRun: Type.Optional(Type.Object({
    runId: Type.String(), status: RunStatus, startedAt: IsoDate, finishedAt: Type.Union([IsoDate, Type.Null()]),
  })),
});
export const JobDetail = Type.Intersect([JobSummary, Type.Object({
  description: Type.String(), owner: Type.String(),
  concurrencyPolicy: Type.Union([Type.Literal('Allow'), Type.Literal('Forbid'), Type.Literal('Replace')]),
  timeoutSeconds: Type.Integer(), retries: Type.Integer(),
})]);
export const RunsPage = Type.Object({ items: Type.Array(JobRun), nextCursor: Type.Union([Type.String(), Type.Null()]) });

export const JobParams = Type.Object({ id: Type.String({ pattern: '^[a-z0-9-]{1,52}$' }) });
export const RunParams = Type.Object({ id: Type.String({ pattern: '^[a-z0-9-]{1,52}$' }), runId: Type.String({ minLength: 1, maxLength: 200 }) });
// pause/resume/stop may be sent with NO body (the dashboard does), so no body schema: parse it leniently
export function reasonOf(body: unknown): string | undefined {
  const r = (body as { reason?: unknown } | null | undefined)?.reason;
  return typeof r === 'string' ? r.slice(0, 500) : undefined;
}
export const RunNowBody = Type.Object({ input: Type.Optional(Type.Unknown()) }, { additionalProperties: false });
export const RunsQuery = Type.Object({
  limit: Type.Integer({ minimum: 1, maximum: 100, default: 20 }),
  cursor: Type.Optional(Type.String({ maxLength: 500 })),
});

export type JobRunT = Type.Static<typeof JobRun>;
export type JobSummaryT = Type.Static<typeof JobSummary>;
export type JobDetailT = Type.Static<typeof JobDetail>;
export type RunStatusT = Type.Static<typeof RunStatus>;
export type Page<T> = { items: T[]; nextCursor: string | null };
```

The same schemas produce the OpenAPI document (via `@fastify/swagger`) that the Next.js dashboard can generate its client types from.

## 5. RFC 9457 Problem Details

```ts
// src/problems.ts
export class ProblemError extends Error {
  constructor(
    public status: number,
    public slug: string,                 // becomes https://ops.acme.dev/problems/<slug>
    public title: string,
    public detail?: string,
    public extra: Record<string, unknown> = {},
  ) { super(detail ?? title); }
}
export const notFound = (what: string) => new ProblemError(404, 'not-found', 'Resource not found', what);
export const forbidden = (role: string) => new ProblemError(403, 'forbidden', 'Missing role', `Requires ${role}`, { requiredRole: role });
export const unauthorized = (detail: string) => new ProblemError(401, 'unauthorized', 'Authentication required', detail);
export const runNotActive = (runId: string, status: string) =>
  new ProblemError(409, 'run-not-active', 'Run is not active', `Run ${runId} is ${status}`, { runId, runStatus: status });
```

```ts
// in server.ts
app.setErrorHandler((err, req, reply) => {
  const p =
    err instanceof ProblemError ? err
    : (err as { validation?: unknown }).validation
      ? new ProblemError(400, 'validation', 'Invalid request', err.message, { errors: (err as any).validation })
    : (err as { statusCode?: number }).statusCode === 429
      ? new ProblemError(429, 'rate-limited', 'Too many requests', err.message)
    : isBackendDown(err)
      ? new ProblemError(503, 'backend-unavailable', 'A job backend is unavailable')
      : new ProblemError(500, 'internal', 'Unexpected error');
  if (p.status >= 500) req.log.error({ err }, 'request failed');
  if (p.status === 401) reply.header('WWW-Authenticate', 'Bearer realm="jobs-api"');
  reply.code(p.status).type('application/problem+json').send({
    type: `https://ops.acme.dev/problems/${p.slug}`,
    title: p.title, status: p.status, detail: p.detail,
    instance: req.url, requestId: req.id, ...p.extra,
  });
});
app.setNotFoundHandler((req) => { throw notFound(`${req.method} ${req.url}`); });

const isBackendDown = (err: unknown) =>
  /ECONNREFUSED|Connection is closed|ETIMEDOUT/.test(String((err as Error)?.message));
```

```http
HTTP/1.1 409 Conflict
Content-Type: application/problem+json

{ "type": "https://ops.acme.dev/problems/run-not-active", "title": "Run is not active",
  "status": 409, "detail": "Run manual-3f9c... is succeeded", "instance": "/api/jobs/sync-inventory/runs/manual-3f9c.../stop",
  "requestId": "req-7", "runId": "manual-3f9c...", "runStatus": "succeeded" }
```

Problem types used: `unauthorized` 401, `forbidden` 403, `not-found` 404, `validation` 400, `idempotency-key-missing` 400, `run-not-active` 409, `idempotency-key-in-flight` 409, `idempotency-key-reused` 422, `rate-limited` 429, `backend-unavailable` 503, `internal` 500.

## 6. The `JobBackend` interface

```ts
// src/backends/backend.ts
import { createHash } from 'node:crypto';
import type { JobDefinition } from '@acme/jobs-core';
import type { JobSummaryT, JobDetailT, JobRunT, Page } from '../schemas.js';
import { ProblemError } from '../problems.js';

export type JobSummary = JobSummaryT;
export type JobDetail = JobDetailT;
export type JobRun = JobRunT;
export { type Page };

/** Extension to the spec'd signature: an optional third argument carries idempotency and actor. */
export interface RunNowOptions { idempotencyKey?: string; actor?: string }
export interface StopOptions { reason?: string; actor?: string }

export interface JobBackend {
  list(): Promise<JobSummary[]>;
  get(jobId: string): Promise<JobDetail>;
  pause(jobId: string): Promise<void>;       // stop future scheduled runs
  resume(jobId: string): Promise<void>;
  runNow(jobId: string, input?: unknown, opts?: RunNowOptions): Promise<{ runId: string }>;
  stopRun(jobId: string, runId: string, opts?: StopOptions): Promise<void>; // cancel an in-flight run
  listRuns(jobId: string, opts: { limit: number; cursor?: string }): Promise<Page<JobRun>>;
  getRunLogs(jobId: string, runId: string): AsyncIterable<string>;
}

/**
 * Routes each job id to a backend: BullMQ jobs come from the registry; anything else is looked up
 * as a CronJob labelled acme.dev/managed-by=jobs-api (the K8s backend 404s if there's none).
 */
export class JobService {
  constructor(private defs: readonly JobDefinition[], private backends: Record<'bullmq' | 'k8s', JobBackend>) {}
  backendFor(jobId: string): JobBackend {
    const def = this.defs.find((d) => d.id === jobId);
    return this.backends[def?.backend ?? 'k8s'];
  }
  async list() {
    const all = await Promise.all(Object.values(this.backends).map((b) => b.list()));
    return all.flat().sort((a, b) => a.id.localeCompare(b.id));
  }
}

export const hash = (s: string) => createHash('sha256').update(s).digest('hex');

export const encodeCursor = (o: unknown) => Buffer.from(JSON.stringify(o)).toString('base64url');
export function decodeCursor<T>(c: string | undefined, fallback: T): T {
  if (!c) return fallback;
  try { return JSON.parse(Buffer.from(c, 'base64url').toString()) as T; }
  catch { throw new ProblemError(400, 'validation', 'Invalid cursor'); }
}
```

## 7. `BullMqBackend`

```ts
// src/backends/bullmq.ts
import { Queue, type Job } from 'bullmq';
import type { Redis, RedisOptions } from 'ioredis';
import { randomUUID } from 'node:crypto';
import { KEYS, scheduledTimeOf, type JobDefinition, type ControlMessage } from '@acme/jobs-core';
import { templateFor } from '@acme/jobs-core';                    // same template the worker uses
import { hash, encodeCursor, decodeCursor, type JobBackend, type JobRun, type Page, type RunNowOptions, type StopOptions } from './backend.js';
import { notFound, runNotActive } from '../problems.js';
import type { RunStatusT } from '../schemas.js';

const PAUSED_SPEC = (id: string) => `acme-jobs:paused-spec:${id}`;   // scheduler stash while paused
const iso = (ms?: number) => (ms ? new Date(ms).toISOString() : undefined);
const FINAL = new Set(['completed', 'failed']);

export class BullMqBackend implements JobBackend {
  private queues = new Map<string, Queue>();

  constructor(
    private defs: readonly JobDefinition[],
    private redis: Redis,                    // producer-style connection: fails fast
    private connection: RedisOptions,
  ) {}

  private def(id: string) {
    const d = this.defs.find((x) => x.id === id && x.backend === 'bullmq');
    if (!d) throw notFound(`job ${id}`);
    return d;
  }
  queue(id: string) {
    let q = this.queues.get(id);
    if (!q) { q = new Queue(id, { connection: this.connection }); this.queues.set(id, q); }
    return q;
  }

  async list() {
    return Promise.all(this.defs.filter((d) => d.backend === 'bullmq').map((d) => this.summary(d)));
  }

  async get(id: string) {
    const d = this.def(id);
    return {
      ...(await this.summary(d)),
      description: d.description, owner: d.owner,
      concurrencyPolicy: d.concurrencyPolicy, timeoutSeconds: d.timeoutSeconds, retries: d.retries,
    };
  }

  private async summary(d: JobDefinition) {
    const q = this.queue(d.id);
    const [sched, paused, last] = await Promise.all([
      q.getJobScheduler(d.id),
      this.redis.sismember(KEYS.paused, d.id),
      this.listRuns(d.id, { limit: 1 }),
    ]);
    const lastRun = last.items[0];
    return {
      id: d.id, name: d.name, backend: 'bullmq' as const,
      schedule: sched?.pattern ?? d.schedule, timezone: sched?.tz ?? d.timezone,
      state: paused ? ('paused' as const) : ('active' as const),
      nextRunAt: !paused && sched?.next ? new Date(sched.next).toISOString() : null,
      ...(lastRun && { lastRun: { runId: lastRun.runId, status: lastRun.status, startedAt: lastRun.startedAt, finishedAt: lastRun.finishedAt ?? null } }),
    };
  }

  /** Pause: stash the live scheduler, mark paused, remove it (and its pending delayed job). */
  async pause(id: string) {
    this.def(id);
    const q = this.queue(id);
    const sched = await q.getJobScheduler(id);
    const tx = this.redis.multi().sadd(KEYS.paused, id);
    if (sched) tx.set(PAUSED_SPEC(id), JSON.stringify(sched));
    await tx.exec();
    await q.removeJobScheduler(id);                         // in-flight runs are untouched
  }

  /** Resume: re-create the scheduler exactly as it was; the first job waits for the next pattern match. */
  async resume(id: string) {
    const d = this.def(id);
    const stashed = await this.redis.get(PAUSED_SPEC(id));
    const s = stashed ? JSON.parse(stashed) : undefined;
    await this.redis.multi().srem(KEYS.paused, id).del(PAUSED_SPEC(id)).exec();
    const fallback = templateFor(d);
    await this.queue(id).upsertJobScheduler(
      id,
      { pattern: s?.pattern ?? d.schedule, tz: s?.tz ?? d.timezone },
      { name: s?.name ?? fallback.name, data: s?.template?.data ?? fallback.data, opts: s?.template?.opts ?? fallback.opts },
    );
  }

  async runNow(id: string, input?: unknown, opts?: RunNowOptions) {
    const d = this.def(id);
    // Deterministic id from the Idempotency-Key: a retried request can't enqueue twice.
    // BullMQ custom ids must not contain ':' and must not be all digits.
    const runId = opts?.idempotencyKey
      ? `manual-${hash(`${opts.actor ?? 'anon'}|${opts.idempotencyKey}`).slice(0, 32)}`
      : `manual-${randomUUID()}`;
    const t = templateFor(d);
    const job = await this.queue(id).add(d.id, { trigger: 'manual', input, requestedBy: opts?.actor }, { ...t.opts, jobId: runId });
    return { runId: job.id! };                               // existing job returned if the id already exists
  }

  async stopRun(id: string, runId: string, opts?: StopOptions) {
    this.def(id);
    const job = await this.queue(id).getJob(runId);
    if (!job) throw notFound(`run ${runId}`);
    const state = await job.getState();
    if (FINAL.has(state)) throw runNotActive(runId, this.statusOf(job, state, false));

    const reason = `${opts?.reason ?? 'stopped via jobs-api'} (by ${opts?.actor ?? 'unknown'})`;
    // 1. durable flag: covers queued runs, waiting-children parents and missed pub/sub messages
    await this.redis.set(KEYS.cancel(runId), reason, 'EX', 24 * 3600);
    // 2. fast path: the worker that holds the job aborts its AbortSignal
    if (state === 'active') {
      const msg: ControlMessage = { action: 'cancel', queue: id, runId, reason };
      await this.redis.publish(KEYS.control, JSON.stringify(msg));
    }
  }

  statusOf(job: Job, state: string, cancelFlag: boolean): RunStatusT {
    switch (state) {
      case 'completed': return 'succeeded';
      case 'failed': return job.failedReason?.startsWith('cancelled:') ? 'cancelled' : 'failed';
      case 'active':
      case 'waiting-children': return cancelFlag ? 'cancelling' : 'running';
      default: return cancelFlag ? 'cancelling' : 'queued';     // waiting, prioritized, delayed (retry backoff)
    }
  }

  toRun(job: Job, state: string, cancelFlag: boolean): JobRun {
    const status = this.statusOf(job, state, cancelFlag);
    return {
      runId: job.id!,
      status,
      trigger: job.id!.startsWith('repeat:') ? 'schedule' : 'manual',
      startedAt: iso(job.processedOn ?? scheduledTimeOf(job.id!) ?? job.timestamp)!,
      ...(FINAL.has(state) && job.finishedOn && { finishedAt: iso(job.finishedOn) }),
      ...(job.finishedOn && job.processedOn && { durationMs: job.finishedOn - job.processedOn }),
      ...(job.failedReason && { error: job.failedReason }),
      attempts: job.attemptsMade + (state === 'active' ? 1 : 0),
    };
  }

  /** Used by the event hub to turn a QueueEvents id into a full run. */
  async getRun(id: string, runId: string): Promise<JobRun | undefined> {
    const job = await this.queue(id).getJob(runId);
    if (!job) return undefined;
    const [state, flag] = await Promise.all([job.getState(), this.redis.exists(KEYS.cancel(runId))]);
    return this.toRun(job, state, flag === 1);
  }

  /**
   * Newest first. First page: in-flight runs (bounded: global concurrency 1 keeps these few),
   * then completed + failed merged by finishedOn with a two-offset cursor { c, f }.
   */
  async listRuns(id: string, { limit, cursor }: { limit: number; cursor?: string }): Promise<Page<JobRun>> {
    this.def(id);
    const q = this.queue(id);
    const cur = decodeCursor(cursor, { c: 0, f: 0, first: true });
    const items: JobRun[] = [];

    if (cur.first) {
      for (const state of ['active', 'waiting-children', 'waiting', 'prioritized', 'delayed'] as const) {
        for (const job of await q.getJobs([state], 0, 49, false)) {
          // skip the scheduler's *next* tick (delayed, never attempted): that's nextRunAt, not a run
          if (state === 'delayed' && job.attemptsMade === 0 && job.id?.startsWith('repeat:')) continue;
          items.push(this.toRun(job, state, (await this.redis.exists(KEYS.cancel(job.id!))) === 1));
        }
      }
    }

    const need = Math.max(limit - items.length, 0);
    const [completed, failed] = await Promise.all([
      q.getJobs(['completed'], cur.c, cur.c + need - 1, false),
      q.getJobs(['failed'], cur.f, cur.f + need - 1, false),
    ]);
    let ci = 0, fi = 0;
    while (items.length < limit && (ci < completed.length || fi < failed.length)) {
      const c = completed[ci], f = failed[fi];
      if (c && (!f || (c.finishedOn ?? 0) >= (f.finishedOn ?? 0))) { items.push(this.toRun(c, 'completed', false)); ci++; }
      else { items.push(this.toRun(f!, 'failed', false)); fi++; }
    }
    // need === 0: the page was filled by in-flight runs, finished ones start on the next page
    const more = need === 0 || ci < completed.length || fi < failed.length || completed.length === need || failed.length === need;
    return {
      items: items.slice(0, limit),
      nextCursor: more ? encodeCursor({ c: cur.c + ci, f: cur.f + fi, first: false }) : null,
    };
  }

  /** Tails job.log() lines; ends shortly after the run reaches a final state. */
  async *getRunLogs(id: string, runId: string): AsyncIterable<string> {
    this.def(id);
    const q = this.queue(id);
    const job = await q.getJob(runId);
    if (!job) throw notFound(`run ${runId}`);
    let offset = 0;
    let idleAfterFinal = 0;
    while (idleAfterFinal < 2) {
      const { logs } = await q.getJobLogs(runId, offset, offset + 499, true);
      for (const line of logs) yield line;
      offset += logs.length;
      const state = await q.getJobState(runId);
      if (FINAL.has(state) || state === 'unknown') idleAfterFinal += logs.length === 0 ? 1 : 0;
      if (logs.length === 0) await new Promise((r) => setTimeout(r, 1_000));
    }
  }
}
```

Notes:

- `getJobs([type], start, end, asc)` pages within one state; the merge keeps the list globally ordered by finish time without loading everything.
- Stopping a **queued** run sets only the flag; the worker fails it with `cancelled:` the moment it starts, so it stays `cancelling` until then (with Forbid, possibly until the current run ends). Removing the job instead would be instant but would also erase it from history.
- Stopping a parent in **`waiting-children`** sets its flag; active children finish (or are stopped individually) and the parent fails as cancelled when it resumes.

## 8. `K8sCronJobBackend`

Uses `@kubernetes/client-node` with the **object-parameter** style introduced in 1.0 and kept in 2.0 (`readNamespacedCronJob({ name, namespace })`, not positional arguments). Dates in returned models are `Date` objects.

The backend **discovers** CronJobs by the label `acme.dev/managed-by: jobs-api` and refuses (404) to touch any CronJob without it. Owner and description come from the CronJob's `acme.dev/owner` and `acme.dev/description` annotations; the registry is only a fallback. Runs are the Jobs labelled `acme.dev/job-id=<id>` (set on `jobTemplate.metadata.labels`).

```ts
// src/backends/k8s.ts
import * as k8s from '@kubernetes/client-node';
import { PassThrough } from 'node:stream';
import { createInterface } from 'node:readline';
import { randomBytes } from 'node:crypto';
import { CronExpressionParser } from 'cron-parser';
import type { JobDefinition } from '@acme/jobs-core';
import { hash, encodeCursor, decodeCursor, type JobBackend, type JobRun, type Page, type RunNowOptions, type StopOptions } from './backend.js';
import { notFound, runNotActive, ProblemError } from '../problems.js';
import type { RunTombstones } from '../tombstones.js';

const MANAGED_BY = 'acme.dev/managed-by=jobs-api';   // CronJobs opt in to jobs-api with this label
const JOB_ID_LABEL = 'acme.dev/job-id';               // on jobTemplate.metadata.labels → copied to every Job
const MANUAL = 'cronjob.kubernetes.io/instantiate';
const iso = (d?: Date | string | null) => (d ? new Date(d).toISOString() : undefined);
const mergePatch = k8s.setHeaderOptions('Content-Type', k8s.PatchStrategy.MergePatch);

type Deps = { batch: k8s.BatchV1Api; core: k8s.CoreV1Api; log: k8s.Log };

export class K8sCronJobBackend implements JobBackend {
  constructor(
    private defs: readonly JobDefinition[],
    private namespace: string,
    private deps: Deps,
    private tombstones: RunTombstones,          // Postgres: remembers cancelled runs after the Job is deleted
  ) {}

  static fromCluster(defs: readonly JobDefinition[], namespace: string, tombstones: RunTombstones) {
    const kc = new k8s.KubeConfig();
    kc.loadFromDefault();                       // in-cluster ServiceAccount token, or ~/.kube/config locally
    return new K8sCronJobBackend(defs, namespace, {
      batch: kc.makeApiClient(k8s.BatchV1Api),
      core: kc.makeApiClient(k8s.CoreV1Api),
      log: new k8s.Log(kc),
    }, tombstones);
  }

  /** Optional registry entry; the CronJob itself (label + annotations) is the source of truth. */
  private def(id: string) {
    return this.defs.find((x) => x.id === id && x.backend === 'k8s');
  }

  private async call<T>(what: string, fn: () => Promise<T>): Promise<T> {
    try { return await fn(); }
    catch (e) {
      if (e instanceof k8s.ApiException && e.code === 404) throw notFound(what);
      if (e instanceof k8s.ApiException && e.code === 403) throw new ProblemError(503, 'backend-unavailable', 'Kubernetes RBAC denied the request', what);
      throw e;
    }
  }

  /** Reads a CronJob and 404s unless it carries acme.dev/managed-by=jobs-api (no control over unmanaged ones). */
  private async cronJob(id: string) {
    const cj = await this.call(`cronjob ${id}`, () => this.deps.batch.readNamespacedCronJob({ name: id, namespace: this.namespace }));
    if (cj.metadata?.labels?.['acme.dev/managed-by'] !== 'jobs-api') throw notFound(`job ${id}`);
    return cj;
  }
  /** Guard used by mutations and run reads: the job must be a managed CronJob. */
  private async assertManaged(id: string) { await this.cronJob(id); }

  // ---------- reads ----------
  async list() {
    const [cjs, jobs] = await Promise.all([
      this.deps.batch.listNamespacedCronJob({ namespace: this.namespace, labelSelector: MANAGED_BY }),
      this.deps.batch.listNamespacedJob({ namespace: this.namespace, labelSelector: JOB_ID_LABEL }),
    ]);
    return cjs.items.map((cj) =>
      this.summary(cj, jobs.items.filter((j) => j.metadata?.labels?.[JOB_ID_LABEL] === cj.metadata!.name)));
  }

  async get(id: string) {
    const d = this.def(id);
    const [cj, jobs] = await Promise.all([
      this.cronJob(id),
      this.deps.batch.listNamespacedJob({ namespace: this.namespace, labelSelector: `${JOB_ID_LABEL}=${id}` }),
    ]);
    const tmpl = cj.spec!.jobTemplate.spec!;
    const ann = cj.metadata?.annotations ?? {};
    return {
      ...this.summary(cj, jobs.items),
      description: ann['acme.dev/description'] ?? d?.description ?? '',
      owner: ann['acme.dev/owner'] ?? d?.owner ?? 'unknown',
      concurrencyPolicy: (cj.spec!.concurrencyPolicy ?? 'Allow') as 'Allow' | 'Forbid' | 'Replace',
      timeoutSeconds: tmpl.activeDeadlineSeconds ?? d?.timeoutSeconds ?? 3600,
      retries: tmpl.backoffLimit ?? 6,                 // Kubernetes default backoffLimit is 6
    };
  }

  private summary(cj: k8s.V1CronJob, jobs: k8s.V1Job[]) {
    const id = cj.metadata!.name!;
    const tz = cj.spec!.timeZone ?? 'Etc/UTC';        // unset = controller-manager's zone; Acme always sets it
    const paused = cj.spec!.suspend === true;
    const last = jobs.map((j) => this.toRun(j)).sort((a, b) => b.startedAt.localeCompare(a.startedAt))[0];
    return {
      id, name: this.def(id)?.name ?? cj.metadata?.labels?.['app.kubernetes.io/name'] ?? id, backend: 'k8s' as const,
      schedule: cj.spec!.schedule, timezone: tz,
      state: paused ? ('paused' as const) : ('active' as const),
      nextRunAt: paused ? null : CronExpressionParser.parse(cj.spec!.schedule, { tz, currentDate: new Date() }).next().toDate().toISOString(),
      ...(last && { lastRun: { runId: last.runId, status: last.status, startedAt: last.startedAt, finishedAt: last.finishedAt ?? null } }),
    };
  }

  toRun(j: k8s.V1Job): JobRun {
    const cond = (type: string) => j.status?.conditions?.find((c) => c.type === type && c.status === 'True');
    const failed = cond('Failed');
    const status: JobRun['status'] =
      j.metadata?.deletionTimestamp ? 'cancelling'
      : cond('Complete') ? 'succeeded'
      : failed ? 'failed'
      : (j.status?.active ?? 0) > 0 ? 'running'
      : 'queued';
    const started = j.status?.startTime ?? j.metadata?.creationTimestamp;
    const finished = j.status?.completionTime ?? failed?.lastTransitionTime;
    return {
      runId: j.metadata!.name!,
      status,
      trigger: j.metadata?.annotations?.[MANUAL] === 'manual' ? 'manual' : 'schedule',
      startedAt: iso(started)!,
      ...(finished && { finishedAt: iso(finished) }),
      ...(finished && started && { durationMs: new Date(finished).getTime() - new Date(started).getTime() }),
      ...(failed && { error: `${failed.reason ?? 'Failed'}: ${failed.message ?? ''}`.trim() }),
      attempts: (j.status?.failed ?? 0) + (j.status?.succeeded ?? 0) + (j.status?.active ?? 0),
    };
  }

  // ---------- pause / resume: spec.suspend ----------
  async pause(id: string) {
    // 404 unless the CronJob carries acme.dev/managed-by=jobs-api
    await this.assertManaged(id);
    await this.call(`cronjob ${id}`, () => this.deps.batch.patchNamespacedCronJob(
      { name: id, namespace: this.namespace, body: { spec: { suspend: true } } }, mergePatch));
  }

  async resume(id: string) {
    await this.assertManaged(id);
    // No backfill: the CronJob sets startingDeadlineSeconds, so ticks missed while suspended are
    // outside the deadline and skipped instead of fired immediately on unsuspend.
    await this.call(`cronjob ${id}`, () => this.deps.batch.patchNamespacedCronJob(
      { name: id, namespace: this.namespace, body: { spec: { suspend: false } } }, mergePatch));
  }

  // ---------- run now: the API equivalent of `kubectl create job --from=cronjob/<id>` ----------
  async runNow(id: string, input?: unknown, opts?: RunNowOptions) {
    const cj = await this.cronJob(id);
    const suffix = opts?.idempotencyKey
      ? hash(`${opts.actor ?? 'anon'}|${opts.idempotencyKey}`).slice(0, 8)
      : randomBytes(4).toString('hex');
    const name = `${id.slice(0, 47)}-manual-${suffix}`;          // Job names must be ≤ 63 chars

    const spec = structuredClone(cj.spec!.jobTemplate.spec!);
    if (input !== undefined) {
      for (const c of spec.template.spec!.containers) {
        c.env = [...(c.env ?? []).filter((e) => e.name !== 'JOB_INPUT'), { name: 'JOB_INPUT', value: JSON.stringify(input) }];
      }
    }
    const tmplMeta = cj.spec!.jobTemplate.metadata ?? {};
    const body: k8s.V1Job = {
      apiVersion: 'batch/v1',
      kind: 'Job',
      metadata: {
        name,
        namespace: this.namespace,
        labels: { ...tmplMeta.labels, [JOB_ID_LABEL]: id, 'acme.dev/trigger': 'manual' },
        annotations: {
          ...tmplMeta.annotations,
          [MANUAL]: 'manual',                                     // same annotation kubectl sets
          'acme.dev/requested-by': opts?.actor ?? 'unknown',
        },
        // Owned by the CronJob: history limits clean it up, Forbid treats it as active,
        // deleting the CronJob deletes it. No blockOwnerDeletion: that would need extra RBAC
        // (update on cronjobs/finalizers) under the OwnerReferencesPermissionEnforcement plugin.
        ownerReferences: [{ apiVersion: 'batch/v1', kind: 'CronJob', name: id, uid: cj.metadata!.uid!, controller: true }],
      },
      spec,
    };
    try {
      await this.deps.batch.createNamespacedJob({ namespace: this.namespace, body });
    } catch (e) {
      // same Idempotency-Key ⇒ same name ⇒ 409 AlreadyExists: treat as a replay
      if (!(e instanceof k8s.ApiException && e.code === 409)) throw e;
    }
    return { runId: name };
  }

  // ---------- stop: foreground delete → pods get SIGTERM → grace period → Job removed ----------
  async stopRun(id: string, runId: string, opts?: StopOptions) {
    await this.assertManaged(id);
    const job = await this.call(`run ${runId}`, () => this.deps.batch.readNamespacedJob({ name: runId, namespace: this.namespace }));
    if (job.metadata?.labels?.[JOB_ID_LABEL] !== id) throw notFound(`run ${runId}`);
    const run = this.toRun(job);
    if (['succeeded', 'failed'].includes(run.status)) throw runNotActive(runId, run.status);

    await this.tombstones.markCancelling({ jobId: id, run, actor: opts?.actor, reason: opts?.reason });
    await this.call(`run ${runId}`, () => this.deps.batch.deleteNamespacedJob({
      name: runId,
      namespace: this.namespace,
      propagationPolicy: 'Foreground',   // Job stays (with deletionTimestamp) until its pods are gone
    }));
  }

  async listRuns(id: string, { limit, cursor }: { limit: number; cursor?: string }): Promise<Page<JobRun>> {
    await this.assertManaged(id);
    const { offset } = decodeCursor(cursor, { offset: 0 });
    const jobs = await this.deps.batch.listNamespacedJob({ namespace: this.namespace, labelSelector: `${JOB_ID_LABEL}=${id}` });
    const live = jobs.items.map((j) => this.toRun(j));
    const liveIds = new Set(live.map((r) => r.runId));
    // Runs we deleted: once the Job object is gone, the tombstone becomes "cancelled"
    const gone = await this.tombstones.resolveMissing(id, liveIds);
    const all = [...live, ...gone].sort((a, b) => b.startedAt.localeCompare(a.startedAt));
    const items = all.slice(offset, offset + limit);
    return { items, nextCursor: offset + limit < all.length ? encodeCursor({ offset: offset + limit }) : null };
  }

  // ---------- logs: follow the newest pod of the Job ----------
  async *getRunLogs(id: string, runId: string): AsyncIterable<string> {
    await this.assertManaged(id);
    const pod = await this.waitForPod(runId);
    if (!pod) { yield '[jobs-api] no pods left for this run in the cluster; search Loki for run_id=' + runId; return; }
    const container = pod.spec!.containers[0].name;
    const stream = new PassThrough();
    const ctrl = await this.deps.log.log(this.namespace, pod.metadata!.name!, container, stream, {
      follow: true, tailLines: 2000, timestamps: true,
    });
    const lines = createInterface({ input: stream, crlfDelay: Infinity });
    try {
      for await (const line of lines) yield line;               // ends when the container exits
    } finally {
      ctrl.abort();                                            // client went away: close the kube-apiserver stream
      lines.close();
    }
  }

  private async waitForPod(runId: string, timeoutMs = 120_000) {
    const deadline = Date.now() + timeoutMs;
    while (Date.now() < deadline) {
      const pods = await this.deps.core.listNamespacedPod({
        namespace: this.namespace, labelSelector: `batch.kubernetes.io/job-name=${runId}`,
      });
      const newest = pods.items.sort((a, b) =>
        new Date(b.metadata!.creationTimestamp!).getTime() - new Date(a.metadata!.creationTimestamp!).getTime())[0];
      if (!newest) {
        try { await this.deps.batch.readNamespacedJob({ name: runId, namespace: this.namespace }); }
        catch { return undefined; }                             // Job gone and no pods: nothing to stream
      } else if (newest.status?.phase !== 'Pending') {
        return newest;                                          // Running, Succeeded or Failed: logs available
      }
      await new Promise((r) => setTimeout(r, 2_000));
    }
    throw new ProblemError(504, 'pod-not-started', 'Pod did not start in time', runId);
  }
}
```

```ts
// src/tombstones.ts: minimal Postgres-backed memory for deleted (cancelled) K8s runs
import type pg from 'pg';
import type { JobRun } from './backends/backend.js';

export class RunTombstones {
  constructor(private db: pg.Pool) {}
  async markCancelling(a: { jobId: string; run: JobRun; actor?: string; reason?: string }) {
    await this.db.query(
      `INSERT INTO job_run_tombstones (job_id, run_id, trigger, started_at, attempts, requested_by, reason)
       VALUES ($1,$2,$3,$4,$5,$6,$7) ON CONFLICT (job_id, run_id) DO NOTHING`,
      [a.jobId, a.run.runId, a.run.trigger, a.run.startedAt, a.run.attempts, a.actor ?? null, a.reason ?? null]);
  }
  async resolveMissing(jobId: string, liveIds: Set<string>): Promise<JobRun[]> {
    const { rows } = await this.db.query(
      `UPDATE job_run_tombstones SET finished_at = COALESCE(finished_at, now())
       WHERE job_id = $1 AND NOT (run_id = ANY($2::text[]))
       RETURNING run_id, trigger, started_at, finished_at, attempts, reason`,
      [jobId, [...liveIds]]);
    return rows.map((r) => ({
      runId: r.run_id, status: 'cancelled' as const, trigger: r.trigger,
      startedAt: r.started_at.toISOString(), finishedAt: r.finished_at.toISOString(),
      durationMs: r.finished_at.getTime() - r.started_at.getTime(),
      error: `cancelled: ${r.reason ?? 'stopped via jobs-api'}`, attempts: r.attempts,
    }));
  }
}
```

The contract with the CronJob manifests (see [Kubernetes CronJobs in Practice](/Software%20Engineering/Backend/Scheduled%20Jobs/Kubernetes%20CronJobs%20in%20Practice) and [Playwright Automation as a Cron Job](/Software%20Engineering/Backend/Scheduled%20Jobs/Playwright%20Automation%20as%20a%20Cron%20Job)):

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-expired-carts                 # = job id (≤ 52 chars)
  labels:
    acme.dev/managed-by: jobs-api             # opt-in: jobs-api lists and controls only these
  annotations:
    acme.dev/owner: team-checkout             # → JobDetail.owner
    acme.dev/description: "Delete carts not updated for 7 days"   # → JobDetail.description
spec:
  timeZone: Europe/Berlin                     # → JobSummary.timezone
  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 300                # resume = no backfill
  jobTemplate:
    metadata:
      labels:
        acme.dev/job-id: cleanup-expired-carts   # copied to every Job: run history and events select on it
    spec:
      activeDeadlineSeconds: 1200             # → JobDetail.timeoutSeconds
      backoffLimit: 2                         # → JobDetail.retries
      template:
        spec:
          terminationGracePeriodSeconds: 60   # the container must stop cleanly on SIGTERM within this
```

<Tip>
**Stop without losing the Job object**

An alternative to deleting is patching the **Job's** `spec.suspend: true`: Kubernetes terminates the active pods but keeps the Job (with a `Suspended` condition) for history. Acme deletes with foreground propagation to match `kubectl delete job` semantics and uses tombstones for history.
</Tip>

## 9. Postgres: audit log, idempotency keys, tombstones

```sql
-- migrations/001_init.sql
CREATE TABLE job_audit_log (
  id               bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  occurred_at      timestamptz NOT NULL DEFAULT now(),
  actor_sub        text        NOT NULL,             -- OIDC subject
  actor_email      text,
  actor_roles      text[]      NOT NULL,
  action           text        NOT NULL CHECK (action IN ('job.pause','job.resume','run.create','run.stop')),
  job_id           text        NOT NULL,
  run_id           text,
  outcome          text        NOT NULL CHECK (outcome IN ('accepted','denied','failed')),
  reason           text,
  request_id       text        NOT NULL,
  idempotency_key  text,
  source_ip        inet,
  user_agent       text,
  details          jsonb       NOT NULL DEFAULT '{}'
);
CREATE INDEX job_audit_log_job_time ON job_audit_log (job_id, occurred_at DESC);
CREATE INDEX job_audit_log_actor_time ON job_audit_log (actor_sub, occurred_at DESC);

-- append-only for the application role
REVOKE UPDATE, DELETE, TRUNCATE ON job_audit_log FROM jobs_api;
GRANT INSERT, SELECT ON job_audit_log TO jobs_api;

CREATE TABLE idempotency_keys (
  principal        text        NOT NULL,
  idem_key         text        NOT NULL CHECK (length(idem_key) BETWEEN 8 AND 255),
  route            text        NOT NULL,             -- e.g. POST /api/jobs/sync-inventory/runs
  request_hash     text        NOT NULL,             -- sha256 of the canonical body
  response_status  int,                              -- NULL while in flight
  response_body    jsonb,
  created_at       timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (principal, idem_key)
);
CREATE INDEX idempotency_keys_created ON idempotency_keys (created_at);   -- nightly purge > 24 h

CREATE TABLE job_run_tombstones (
  job_id       text        NOT NULL,
  run_id       text        NOT NULL,
  trigger      text        NOT NULL,
  started_at   timestamptz NOT NULL,
  finished_at  timestamptz,
  attempts     int         NOT NULL DEFAULT 0,
  requested_by text,
  reason       text,
  PRIMARY KEY (job_id, run_id)
);
```

```ts
// src/audit.ts
import type pg from 'pg';
import type { FastifyRequest } from 'fastify';

export type AuditAction = 'job.pause' | 'job.resume' | 'run.create' | 'run.stop';
export async function audit(db: pg.Pool, req: FastifyRequest, e: {
  action: AuditAction; jobId: string; runId?: string; outcome: 'accepted' | 'denied' | 'failed';
  reason?: string; details?: Record<string, unknown>;
}) {
  await db.query(
    `INSERT INTO job_audit_log (actor_sub, actor_email, actor_roles, action, job_id, run_id, outcome, reason,
                                request_id, idempotency_key, source_ip, user_agent, details)
     VALUES ($1,$2,$3,$4,$5,$6,$7,$8,$9,$10,$11,$12,$13)`,
    [req.user?.sub ?? 'anonymous', req.user?.email ?? null, req.user?.roles ?? [], e.action, e.jobId, e.runId ?? null,
     e.outcome, e.reason ?? null, req.id, req.headers['idempotency-key'] ?? null, req.ip, req.headers['user-agent'] ?? null,
     e.details ?? {}]);
}
```

```ts
// src/idempotency.ts: IETF draft "The Idempotency-Key HTTP Header Field" semantics
import type pg from 'pg';
import type { FastifyRequest } from 'fastify';
import { ProblemError } from './problems.js';
import { hash } from './backends/backend.js';

export async function withIdempotency<T>(
  db: pg.Pool, req: FastifyRequest, fn: (key: string) => Promise<{ status: number; body: T }>,
): Promise<{ status: number; body: T; replayed: boolean }> {
  const key = req.headers['idempotency-key'];
  if (typeof key !== 'string' || key.length < 8 || key.length > 255)
    throw new ProblemError(400, 'idempotency-key-missing', 'Idempotency-Key header required', 'Send a unique key (e.g. a UUID) per logical request');
  const principal = req.user!.sub;
  const route = `${req.method} ${req.url.split('?')[0]}`;
  const requestHash = hash(JSON.stringify(req.body ?? {}));

  const inserted = await db.query(
    `INSERT INTO idempotency_keys (principal, idem_key, route, request_hash) VALUES ($1,$2,$3,$4)
     ON CONFLICT (principal, idem_key) DO NOTHING`, [principal, key, route, requestHash]);

  if (inserted.rowCount === 0) {
    const { rows: [row] } = await db.query(
      'SELECT route, request_hash, response_status, response_body FROM idempotency_keys WHERE principal=$1 AND idem_key=$2',
      [principal, key]);
    if (row.route !== route || row.request_hash !== requestHash)
      throw new ProblemError(422, 'idempotency-key-reused', 'Idempotency-Key reused with a different request');
    if (row.response_status === null)
      throw new ProblemError(409, 'idempotency-key-in-flight', 'A request with this Idempotency-Key is still being processed');
    return { status: row.response_status, body: row.response_body as T, replayed: true };
  }

  try {
    const res = await fn(key);
    await db.query('UPDATE idempotency_keys SET response_status=$3, response_body=$4 WHERE principal=$1 AND idem_key=$2',
      [principal, key, res.status, JSON.stringify(res.body)]);
    return { ...res, replayed: false };
  } catch (err) {
    await db.query('DELETE FROM idempotency_keys WHERE principal=$1 AND idem_key=$2', [principal, key]); // allow a retry
    throw err;
  }
}
```

Two layers of idempotency: the Postgres record replays the **HTTP response**, and the backend derives the **run id** from the key, so even if Postgres loses the record the queue or the API server rejects the duplicate.

## 10. Authentication and RBAC

```ts
// src/auth.ts
import fp from 'fastify-plugin';
import { createRemoteJWKSet, jwtVerify, type JWTVerifyGetKey } from 'jose';
import { forbidden, unauthorized } from './problems.js';

export type User = { sub: string; email?: string; roles: string[] };
declare module 'fastify' {
  interface FastifyRequest { user: User | null }
  interface FastifyContextConfig { requiredRole?: 'ops:jobs:read' | 'ops:jobs:write' | 'ops:jobs:admin'; public?: boolean }
}

export const authPlugin = fp<{ issuer: string; audience: string; jwks?: JWTVerifyGetKey; jwksUrl?: string }>(async (app, opts) => {
  const jwks = opts.jwks ?? createRemoteJWKSet(new URL(opts.jwksUrl!), { cooldownDuration: 30_000 }); // caches keys, refetches on unknown kid
  app.decorateRequest('user', null);

  app.addHook('onRequest', async (req) => {
    if (req.routeOptions.config?.public) return;
    const h = req.headers.authorization;
    if (!h?.startsWith('Bearer ')) throw unauthorized('Missing bearer token');
    try {
      const { payload } = await jwtVerify(h.slice(7), jwks, {
        issuer: opts.issuer,
        audience: opts.audience,                  // tokens minted for other APIs are rejected
        algorithms: ['RS256', 'ES256'],           // never accept "none" or HS* with a public key
        clockTolerance: 30,
      });
      const roles = Array.isArray(payload.roles) ? (payload.roles as string[]) : [];
      req.user = { sub: payload.sub!, email: payload.email as string | undefined, roles };
    } catch {
      throw unauthorized('Invalid or expired token');
    }
  });

  app.addHook('preHandler', async (req) => {
    const role = req.routeOptions.config?.requiredRole;
    if (!role || req.routeOptions.config?.public) return;
    const roles = req.user?.roles ?? [];
    const ok = roles.includes(role) || roles.includes('ops:jobs:admin') ||
      (role === 'ops:jobs:read' && roles.includes('ops:jobs:write'));   // write implies read
    if (!ok) throw forbidden(role);
  });
});
```

Per-job policy on top of the role (who can run what):

| Job | `ops:jobs:read` | Pause/Resume | Run now | Stop |
|---|---|---|---|---|
| `sync-inventory` | all engineers | `ops:jobs:write` | `ops:jobs:write` | `ops:jobs:write` |
| `nightly-sales-report` | all engineers | `ops:jobs:write` + member of `team-finance-eng` | same | same |
| `cleanup-expired-carts` | all engineers | `ops:jobs:write` | `ops:jobs:write` | `ops:jobs:write` |
| `playwright-checkout-smoke` | all engineers | `ops:jobs:write` | `ops:jobs:write` (safe: read-only check) | `ops:jobs:write` |

```ts
// src/policy.ts
const EXTRA_ROLE: Record<string, string> = { 'nightly-sales-report': 'team:finance-eng' };
export function assertJobPolicy(user: User, jobId: string) {
  const extra = EXTRA_ROLE[jobId];
  if (extra && !user.roles.includes(extra) && !user.roles.includes('ops:jobs:admin')) throw forbidden(extra);
}
```

## 11. Routes

```ts
// src/routes/jobs.ts
import type { FastifyPluginAsyncTypebox } from '@fastify/type-provider-typebox';
import Type from 'typebox';
import type pg from 'pg';
import { JobSummary, JobDetail, JobParams, RunParams, RunNowBody, RunsQuery, RunsPage, reasonOf } from '../schemas.js';
import type { JobService } from '../backends/backend.js';
import type { EventHub } from '../events.js';
import { withIdempotency } from '../idempotency.js';
import { audit, type AuditAction } from '../audit.js';
import { assertJobPolicy } from '../policy.js';
import { openSse } from '../sse.js';

export const jobRoutes: FastifyPluginAsyncTypebox<{ jobs: JobService; db: pg.Pool; hub: EventHub }> = async (app, { jobs, db, hub }) => {
  const read = { requiredRole: 'ops:jobs:read' as const };
  const write = { requiredRole: 'ops:jobs:write' as const, rateLimit: { max: 20, timeWindow: '1 minute' } };

  /** Every mutation: per-job policy, then action, then audit (denials and failures too). */
  async function mutate<T>(req: any, action: AuditAction, jobId: string, fn: () => Promise<T & { runId?: string }>) {
    try { assertJobPolicy(req.user, jobId); }
    catch (e) { await audit(db, req, { action, jobId, runId: req.params.runId, outcome: 'denied' }); throw e; }
    try {
      const res = await fn();
      await audit(db, req, { action, jobId, runId: res?.runId ?? req.params.runId, outcome: 'accepted', reason: reasonOf(req.body) });
      return res;
    } catch (e) {
      await audit(db, req, { action, jobId, runId: req.params.runId, outcome: 'failed', details: { error: String(e) } });
      throw e;
    }
  }

  app.get('/', { config: read, schema: { response: { 200: Type.Array(JobSummary) } } }, () => jobs.list());

  app.get('/events', { config: read }, async (req, reply) => {
    const sse = openSse(req, reply);
    const off = hub.subscribe((evt, id) => sse.send(evt, { id }), req.headers['last-event-id'] as string | undefined);
    await sse.closed;
    off();
  });

  app.get('/:id', { config: read, schema: { params: JobParams, response: { 200: JobDetail } } },
    (req) => jobs.backendFor(req.params.id).get(req.params.id));

  app.post('/:id/pause', { config: write, schema: { params: JobParams } }, async (req, reply) => {
    const { id } = req.params;
    await mutate(req, 'job.pause', id, () => jobs.backendFor(id).pause(id) as Promise<any>);
    await hub.publishJobUpdated(id, 'paused');
    reply.code(204).send();
  });

  app.post('/:id/resume', { config: write, schema: { params: JobParams } }, async (req, reply) => {
    const { id } = req.params;
    await mutate(req, 'job.resume', id, () => jobs.backendFor(id).resume(id) as Promise<any>);
    await hub.publishJobUpdated(id, 'active');
    reply.code(204).send();
  });

  app.post('/:id/runs', {
    config: { ...write, rateLimit: { max: 10, timeWindow: '1 minute' } },
    schema: { params: JobParams, body: RunNowBody, response: { 202: Type.Object({ runId: Type.String() }) } },
  }, async (req, reply) => {
    const { id } = req.params;
    const backend = jobs.backendFor(id);                          // 404 before touching the idempotency store
    const res = await withIdempotency(db, req, (key) =>
      mutate(req, 'run.create', id, async () => {
        const { runId } = await backend.runNow(id, req.body.input, { idempotencyKey: key, actor: req.user!.sub });
        return { status: 202, body: { runId }, runId };
      }));
    if (res.replayed) reply.header('Idempotent-Replayed', 'true');
    reply.code(res.status).send(res.body);
  });

  app.post('/:id/runs/:runId/stop', { config: write, schema: { params: RunParams } }, async (req, reply) => {
    const { id, runId } = req.params;
    await mutate(req, 'run.stop', id, () =>
      jobs.backendFor(id).stopRun(id, runId, { reason: reasonOf(req.body), actor: req.user!.sub }) as Promise<any>);
    reply.code(202).send();
  });

  app.get('/:id/runs', { config: read, schema: { params: JobParams, querystring: RunsQuery, response: { 200: RunsPage } } },
    (req) => jobs.backendFor(req.params.id).listRuns(req.params.id, { limit: req.query.limit, cursor: req.query.cursor }));

  app.get('/:id/runs/:runId/logs', { config: read, schema: { params: RunParams } }, async (req, reply) => {
    const { id, runId } = req.params;
    const lines = jobs.backendFor(id).getRunLogs(id, runId)[Symbol.asyncIterator]();
    const first = await lines.next();                             // surface 404s as problem+json before hijacking
    const sse = openSse(req, reply);
    let aborted = false;
    sse.closed.then(() => { aborted = true; void lines.return?.(); });
    for (let r = first; !r.done && !aborted; r = await lines.next()) sse.sendRaw(r.value);
    if (!aborted) { sse.send({ runId }, { event: 'end' }); sse.end(); }
  });
};
```

Fastify's router ranks static segments above parameters, so `/events` never matches `/:id`.

### SSE helper

```ts
// src/sse.ts
import type { FastifyReply, FastifyRequest } from 'fastify';

export function openSse(req: FastifyRequest, reply: FastifyReply) {
  reply.hijack();                                    // we write the raw response ourselves
  const res = reply.raw;
  res.writeHead(200, {
    'content-type': 'text/event-stream; charset=utf-8',
    'cache-control': 'no-cache, no-transform',
    connection: 'keep-alive',
    'x-accel-buffering': 'no',                       // disable proxy buffering (nginx, some gateways)
  });
  res.write('retry: 5000\n\n');                      // client reconnect delay
  const heartbeat = setInterval(() => res.write(': ping\n\n'), 15_000);  // keeps idle proxies from closing
  const closed = new Promise<void>((resolve) => req.raw.on('close', () => { clearInterval(heartbeat); resolve(); }));
  return {
    closed,
    /** JSON event; unnamed (onmessage) unless `event` is given */
    send(data: unknown, o: { event?: string; id?: string } = {}) {
      res.write(`${o.id ? `id: ${o.id}\n` : ''}${o.event ? `event: ${o.event}\n` : ''}data: ${JSON.stringify(data)}\n\n`);
    },
    /** raw log line: split on newlines so one line can never break the SSE framing */
    sendRaw(line: string) { res.write(line.split(/\r?\n/).map((l) => `data: ${l}`).join('\n') + '\n\n'); },
    end() { clearInterval(heartbeat); res.end(); },
  };
}
```

### Event hub

`GET /api/jobs/events` merges BullMQ `QueueEvents`, a Kubernetes watch on Jobs, and `job.updated` messages. Because several `jobs-api` replicas run behind the Service, every event goes through a Redis channel so each replica's SSE clients see everything.

The payloads are **flat, partial updates**, the same shape the worker-side listeners in [BullMQ Job Schedulers in Practice](/Software%20Engineering/Backend/Scheduled%20Jobs/BullMQ%20Job%20Schedulers%20in%20Practice#11-progress-logs-and-live-status) publish and the [Job Scheduler Dashboard with Next.js](/Software%20Engineering/Backend/Scheduled%20Jobs/Job%20Scheduler%20Dashboard%20with%20Next.js) parses:

```ts
type JobEvent =
  | { type: 'run.updated'; jobId: string; runId: string; status?: RunStatus; progress?: unknown }
  | { type: 'job.updated'; jobId: string; state?: 'active' | 'paused' };
```

A client that needs the full object refetches it (`GET /api/jobs/:id`, `GET /api/jobs/:id/runs`). Each SSE message carries an `id:`; on reconnect the browser sends `Last-Event-ID` and the hub replays what it still has in a small ring buffer.

```ts
// src/events.ts
import { QueueEvents } from 'bullmq';
import * as k8s from '@kubernetes/client-node';
import type { Redis, RedisOptions } from 'ioredis';
import type { K8sCronJobBackend } from './backends/k8s.js';
import type { RunStatusT } from './schemas.js';

export type JobEvent =
  | { type: 'run.updated'; jobId: string; runId: string; status?: RunStatusT; progress?: unknown }
  | { type: 'job.updated'; jobId: string; state?: 'active' | 'paused' };
type Listener = (e: JobEvent, id: string) => void;

const CHANNEL = 'acme-jobs:events';
const BUFFER = 500;

export class EventHub {
  private listeners = new Set<Listener>();
  private recent: { id: string; e: JobEvent }[] = [];      // replay buffer for Last-Event-ID
  private seq = 0;
  constructor(private pub: Redis, private sub: Redis) {}

  async start(opts: { bullJobIds: string[]; connection: RedisOptions; k8s: K8sCronJobBackend; kc: k8s.KubeConfig; namespace: string }) {
    await this.sub.subscribe(CHANNEL);
    this.sub.on('message', (_c, raw) => this.deliver(JSON.parse(raw) as JobEvent));

    // BullMQ → flat run.updated. Every replica receives every QueueEvent, so dedupe by event id.
    for (const jobId of opts.bullJobIds) {
      const qe = new QueueEvents(jobId, { connection: opts.connection });
      const emit = async (eventId: string, e: JobEvent) => {
        if (await this.pub.set(`acme-jobs:evt:${jobId}:${eventId}`, '1', 'EX', 60, 'NX')) await this.publish(e);
      };
      qe.on('waiting',   ({ jobId: runId }, id) => void emit(id, { type: 'run.updated', jobId, runId, status: 'queued' }));
      qe.on('active',    ({ jobId: runId }, id) => void emit(id, { type: 'run.updated', jobId, runId, status: 'running' }));
      qe.on('progress',  ({ jobId: runId, data }, id) => void emit(id, { type: 'run.updated', jobId, runId, progress: data }));
      qe.on('completed', ({ jobId: runId }, id) => void emit(id, { type: 'run.updated', jobId, runId, status: 'succeeded' }));
      qe.on('failed',    ({ jobId: runId, failedReason }, id) => void emit(id, {
        type: 'run.updated', jobId, runId, status: failedReason?.startsWith('cancelled:') ? 'cancelled' : 'failed',
      }));
    }

    // Kubernetes: watch Jobs of managed CronJobs; the Job's status maps to the same run statuses
    const watch = new k8s.Watch(opts.kc);
    const startWatch = () => void watch.watch(
      `/apis/batch/v1/namespaces/${opts.namespace}/jobs`,
      { labelSelector: 'acme.dev/job-id' },
      async (phase, obj: k8s.V1Job) => {
        const jobId = obj.metadata?.labels?.['acme.dev/job-id'];
        if (!jobId) return;
        const key = `acme-jobs:evt:k8s:${obj.metadata!.uid}:${obj.metadata!.resourceVersion}`;
        if (!(await this.pub.set(key, '1', 'EX', 60, 'NX'))) return;
        const status = phase === 'DELETED' ? 'cancelled' : opts.k8s.toRun(obj).status;   // deleted = stopped via jobs-api
        await this.publish({ type: 'run.updated', jobId, runId: obj.metadata!.name!, status });
      },
      () => setTimeout(startWatch, 5_000),                                 // reconnect after errors/timeouts
    );
    startWatch();
  }

  publish(e: JobEvent) { return this.pub.publish(CHANNEL, JSON.stringify(e)); }
  publishJobUpdated(jobId: string, state: 'active' | 'paused') { return this.publish({ type: 'job.updated', jobId, state }); }

  private deliver(e: JobEvent) {
    const id = `${Date.now()}-${this.seq++}`;
    this.recent.push({ id, e });
    if (this.recent.length > BUFFER) this.recent.shift();
    for (const l of this.listeners) l(e, id);
  }

  /** Replays buffered events after lastEventId (if still buffered), then streams live ones. */
  subscribe(fn: Listener, lastEventId?: string) {
    if (lastEventId) {
      const i = this.recent.findIndex((r) => r.id === lastEventId);
      if (i >= 0) for (const r of this.recent.slice(i + 1)) fn(r.e, r.id);
    }
    this.listeners.add(fn);
    return () => { this.listeners.delete(fn); };
  }
}
```

A `DELETED` watch event for a run that finished normally (history limits, TTL) would also look like a stop; in practice the dashboard already saw its `succeeded`/`failed` update and refetches the run list, where tombstones decide. If the replay buffer no longer has the client's `Last-Event-ID` (or the client hit another replica), the client should refetch. For durable replay across replicas, write events to a Redis Stream and read from the client's last id.

### Wiring it together

```ts
// src/server.ts
import Fastify from 'fastify';
import type { TypeBoxTypeProvider } from '@fastify/type-provider-typebox';
import rateLimit from '@fastify/rate-limit';
import pg from 'pg';
import { Redis } from 'ioredis';
import { JOBS, producerConnection } from '@acme/jobs-core';
import { authPlugin } from './auth.js';
import { JobService } from './backends/backend.js';
import { BullMqBackend } from './backends/bullmq.js';
import { K8sCronJobBackend } from './backends/k8s.js';
import { RunTombstones } from './tombstones.js';
import { EventHub } from './events.js';
import { jobRoutes } from './routes/jobs.js';
import { ProblemError } from './problems.js';

export async function buildServer(deps?: Partial<{ jobs: JobService; db: pg.Pool; hub: EventHub; jwks: any }>) {
  const app = Fastify({
    logger: { level: 'info', redact: ['req.headers.authorization'] },
    genReqId: () => crypto.randomUUID(),
    trustProxy: true,
  }).withTypeProvider<TypeBoxTypeProvider>();

  const db = deps?.db ?? new pg.Pool({ connectionString: process.env.DATABASE_URL, max: 10 });
  const redis = deps?.jobs && deps?.hub ? undefined : new Redis(producerConnection);   // tests inject fakes
  const jobs = deps?.jobs ?? new JobService(JOBS, {
    bullmq: new BullMqBackend(JOBS, redis!, producerConnection),
    k8s: K8sCronJobBackend.fromCluster(JOBS, 'shop-prod', new RunTombstones(db)),
  });
  const hub = deps?.hub ?? new EventHub(redis!, new Redis(producerConnection));
  // in main.ts: await hub.start({ bullJobIds: bullJobs().map((j) => j.id), connection: workerConnection,
  //                                k8s: k8sBackend, kc, namespace: 'shop-prod' });

  // ... setErrorHandler / setNotFoundHandler from §5 ...
  await app.register(authPlugin, {
    issuer: process.env.OIDC_ISSUER ?? 'https://id.acme.dev/realms/acme',
    audience: 'jobs-api',
    jwksUrl: process.env.OIDC_JWKS_URL, jwks: deps?.jwks,
  });
  await app.register(rateLimit, {
    global: true, max: 300, timeWindow: '1 minute',
    hook: 'preHandler',                                   // after auth, so we can key by user
    keyGenerator: (req) => req.user?.sub ?? req.ip,
    errorResponseBuilder: (_req, ctx) => new ProblemError(429, 'rate-limited', 'Too many requests', `Retry in ${ctx.after}`),
  });
  app.get('/healthz', { config: { public: true } }, async () => ({ ok: true }));
  await app.register(jobRoutes, { prefix: '/api/jobs', jobs, db, hub });
  return app;
}
```

`jobs-api` also serves `/metrics` (public to the cluster network only) with a `jobs_paused{job}` gauge (1 while paused), refreshed from `jobs.list()` every 30 s. The missed-schedule alert in [Cron with Message Queues - Architecture](/Software%20Engineering/Backend/Scheduled%20Jobs/Cron%20with%20Message%20Queues%20-%20Architecture#14-observability) uses it to stay quiet for jobs someone paused on purpose.

## 12. Kubernetes RBAC for the `jobs-api` ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata: { name: jobs-api, namespace: shop-prod }
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: jobs-api, namespace: shop-prod }
rules:
  # read CronJobs. RBAC can't select by label: jobs-api itself filters on acme.dev/managed-by=jobs-api,
  # and list/watch can't be restricted by name either
  - apiGroups: ["batch"]
    resources: ["cronjobs"]
    verbs: ["get", "list", "watch"]
  # pause/resume only the CronJobs we manage (keep this list in sync with the labelled CronJobs;
  # a CI check can compare `kubectl get cronjobs -l acme.dev/managed-by=jobs-api` with resourceNames)
  - apiGroups: ["batch"]
    resources: ["cronjobs"]
    resourceNames: ["cleanup-expired-carts", "playwright-checkout-smoke"]
    verbs: ["patch"]
  # run now (create), stop (delete), history (list/get), live events (watch)
  - apiGroups: ["batch"]
    resources: ["jobs"]
    verbs: ["get", "list", "watch", "create", "delete"]
  # find the pods of a Job, then stream their logs
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["list"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: jobs-api, namespace: shop-prod }
subjects: [{ kind: ServiceAccount, name: jobs-api, namespace: shop-prod }]
roleRef: { apiGroup: rbac.authorization.k8s.io, kind: Role, name: jobs-api }
```

```bash
# verify least privilege
kubectl auth can-i patch cronjobs/cleanup-expired-carts -n shop-prod --as=system:serviceaccount:shop-prod:jobs-api   # yes
kubectl auth can-i patch cronjobs/some-other -n shop-prod --as=system:serviceaccount:shop-prod:jobs-api             # no
kubectl auth can-i delete deployments -n shop-prod --as=system:serviceaccount:shop-prod:jobs-api                    # no
kubectl auth can-i get secrets -n shop-prod --as=system:serviceaccount:shop-prod:jobs-api                           # no
```

<Warning>
**`create jobs` is a powerful permission**

A principal that can create Jobs can run **any image with any ServiceAccount** in the namespace, which is a privilege-escalation path if `jobs-api` is compromised. Contain it: run CronJobs in a dedicated namespace (e.g. `shop-jobs`) with no other workloads, and add an admission policy that only lets `jobs-api` create Jobs owned by a CronJob and using Acme images. `ValidatingAdmissionPolicy` is GA since Kubernetes 1.30:
</Warning>

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata: { name: jobs-api-manual-jobs-only }
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - { apiGroups: ["batch"], apiVersions: ["v1"], operations: ["CREATE"], resources: ["jobs"] }
  matchConditions:
    - name: only-jobs-api
      expression: request.userInfo.username == "system:serviceaccount:shop-prod:jobs-api"
  validations:
    - expression: >-
        has(object.metadata.ownerReferences) &&
        object.metadata.ownerReferences.exists(o, o.kind == "CronJob" && o.apiVersion == "batch/v1")
      message: jobs-api may only create Jobs owned by a CronJob
    - expression: object.spec.template.spec.containers.all(c, c.image.startsWith("ghcr.io/acme/"))
      message: only ghcr.io/acme images are allowed
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata: { name: jobs-api-manual-jobs-only }
spec:
  policyName: jobs-api-manual-jobs-only
  validationActions: [Deny]
  matchResources:
    namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: shop-prod } }
```

## 13. Security

- **Authentication**: OIDC access tokens verified locally with `jose` (issuer, audience `jobs-api`, algorithm allow-list, 30 s clock tolerance). The dashboard calls `jobs-api` **server-side** with the user's token (or a token exchanged for this audience); tokens never reach browser JavaScript. See [OAuth 2.x & OpenID Connect](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect) and [Authentication vs Authorization](/Software%20Engineering/APIs%20%26%20Auth/Authentication%20vs%20Authorization).
- **Authorization**: route roles (`ops:jobs:read`, `ops:jobs:write`, `ops:jobs:admin`) plus per-job policy. Deny by default: a route without `requiredRole` fails review.
- **Destructive actions**: stop and pause are confirmed in the UI (type the job id, show what's running and since when) and carry a `reason` that lands in the audit log. `Replace`-style actions (stop then run) are two audited calls, never one silent one.
- **Rate limits**: 300 req/min per user overall, 20 mutations/min, 10 manual runs/min per user. Manual runs also respect Forbid, so spamming "run now" queues at most one extra run per job.
- **Input**: `input` is untrusted JSON; each job validates it with its own schema, and it's size-limited (Fastify `bodyLimit`). K8s passes it as an env var, never as a shell string.
- **Blast radius**: least-privilege Role, admission policy, network policy allowing only `ops-console` → `jobs-api`, Bull Board read-only.
- **Audit**: append-only table (no UPDATE/DELETE grants), ship it to long-term storage, and alert on unusual patterns (many stops by one user, mutations outside business hours). See [API Security](/Software%20Engineering/Security/API%20Security).

## 14. Tests

```ts
// test/api.test.ts
import { beforeAll, afterAll, expect, test, vi } from 'vitest';
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { SignJWT, generateKeyPair, exportJWK, createLocalJWKSet } from 'jose';
import { readFile } from 'node:fs/promises';
import pg from 'pg';
import { buildServer } from '../src/server.js';
import { JobService, type JobBackend } from '../src/backends/backend.js';
import { JOBS } from '@acme/jobs-core';
import { runNotActive } from '../src/problems.js';

let pgc: StartedPostgreSqlContainer, db: pg.Pool, app: Awaited<ReturnType<typeof buildServer>>;
let sign: (roles: string[]) => Promise<string>;
const fake: JobBackend = {
  list: vi.fn(async () => []), get: vi.fn(), pause: vi.fn(async () => {}), resume: vi.fn(async () => {}),
  runNow: vi.fn(async (_id, _in, o) => ({ runId: `manual-${o!.idempotencyKey}` })),
  stopRun: vi.fn(async (_id, runId) => { throw runNotActive(runId, 'succeeded'); }),
  listRuns: vi.fn(async () => ({ items: [], nextCursor: null })),
  getRunLogs: async function* () { yield 'hello'; },
};

beforeAll(async () => {
  pgc = await new PostgreSqlContainer('postgres:18-alpine').start();
  db = new pg.Pool({ connectionString: pgc.getConnectionUri() });
  await db.query('CREATE ROLE jobs_api');
  await db.query(await readFile(new URL('../migrations/001_init.sql', import.meta.url), 'utf8'));

  const { publicKey, privateKey } = await generateKeyPair('ES256');
  const jwk = { ...(await exportJWK(publicKey)), kid: 'test', alg: 'ES256' };
  sign = (roles) => new SignJWT({ roles, email: 'ana@acme.dev' })
    .setProtectedHeader({ alg: 'ES256', kid: 'test' }).setSubject('user-ana')
    .setIssuer('https://id.acme.dev/realms/acme').setAudience('jobs-api').setExpirationTime('5m').sign(privateKey);

  app = await buildServer({
    db, jwks: createLocalJWKSet({ keys: [jwk] }),
    jobs: new JobService(JOBS, { bullmq: fake, k8s: fake }),
    hub: { subscribe: () => () => {}, publishJobUpdated: async () => {} } as any,
  });
}, 90_000);
afterAll(async () => { await app.close(); await db.end(); await pgc.stop(); });

test('401 problem+json without a token', async () => {
  const res = await app.inject({ method: 'GET', url: '/api/jobs' });
  expect(res.statusCode).toBe(401);
  expect(res.headers['content-type']).toContain('application/problem+json');
  expect(res.json().type).toBe('https://ops.acme.dev/problems/unauthorized');
});

test('read-only users cannot pause, and the denial is not executed', async () => {
  const res = await app.inject({
    method: 'POST', url: '/api/jobs/sync-inventory/pause',
    headers: { authorization: `Bearer ${await sign(['ops:jobs:read'])}` },
  });
  expect(res.statusCode).toBe(403);
  expect(fake.pause).not.toHaveBeenCalled();
});

test('run now is idempotent per Idempotency-Key', async () => {
  const headers = { authorization: `Bearer ${await sign(['ops:jobs:write'])}`, 'idempotency-key': 'k-12345678' };
  const a = await app.inject({ method: 'POST', url: '/api/jobs/sync-inventory/runs', headers, payload: { input: { full: true } } });
  const b = await app.inject({ method: 'POST', url: '/api/jobs/sync-inventory/runs', headers, payload: { input: { full: true } } });
  expect(a.statusCode).toBe(202);
  expect(b.json()).toEqual(a.json());
  expect(b.headers['idempotent-replayed']).toBe('true');
  expect(fake.runNow).toHaveBeenCalledTimes(1);

  const c = await app.inject({ method: 'POST', url: '/api/jobs/sync-inventory/runs', headers, payload: { input: { full: false } } });
  expect(c.statusCode).toBe(422);                                    // same key, different body

  const { rows } = await db.query(`SELECT action, outcome FROM job_audit_log WHERE action = 'run.create'`);
  expect(rows).toEqual([{ action: 'run.create', outcome: 'accepted' }]);
});

test('missing Idempotency-Key is a 400', async () => {
  const res = await app.inject({ method: 'POST', url: '/api/jobs/sync-inventory/runs',
    headers: { authorization: `Bearer ${await sign(['ops:jobs:write'])}` }, payload: {} });
  expect(res.statusCode).toBe(400);
  expect(res.json().type).toMatch(/idempotency-key-missing$/);
});

test('stopping a finished run is a 409 run-not-active', async () => {
  const res = await app.inject({ method: 'POST', url: '/api/jobs/sync-inventory/runs/manual-x/stop',
    headers: { authorization: `Bearer ${await sign(['ops:jobs:write'])}` }, payload: { reason: 'test' } });
  expect(res.statusCode).toBe(409);
  expect(res.json()).toMatchObject({ type: 'https://ops.acme.dev/problems/run-not-active', status: 409 });
});
```

```ts
// test/k8s-backend.test.ts: the K8s adapter against fakes (no cluster needed)
import { expect, test, vi } from 'vitest';
import * as k8s from '@kubernetes/client-node';
import { K8sCronJobBackend } from '../src/backends/k8s.js';
import { JOBS } from '@acme/jobs-core';

test('runNow creates a Job from the jobTemplate with owner ref and manual annotation', async () => {
  const cj: k8s.V1CronJob = {
    metadata: { name: 'cleanup-expired-carts', uid: 'uid-1', labels: { 'acme.dev/managed-by': 'jobs-api' } },
    spec: { schedule: '0 * * * *', timeZone: 'Europe/Berlin', jobTemplate: {
      metadata: { labels: { 'acme.dev/job-id': 'cleanup-expired-carts' } },
      spec: { template: { spec: { restartPolicy: 'Never', containers: [{ name: 'cleanup', image: 'ghcr.io/acme/cleanup:1' }] } } },
    } },
  };
  const batch = {
    readNamespacedCronJob: vi.fn(async () => cj),
    createNamespacedJob: vi.fn(async ({ body }: { body: k8s.V1Job }) => body),
  } as unknown as k8s.BatchV1Api;
  const backend = new K8sCronJobBackend(JOBS, 'shop-prod', { batch, core: {} as any, log: {} as any }, {} as any);

  const { runId } = await backend.runNow('cleanup-expired-carts', { dryRun: true }, { idempotencyKey: 'k-1', actor: 'ana' });

  const body = (batch.createNamespacedJob as any).mock.calls[0][0].body as k8s.V1Job;
  expect(runId).toMatch(/^cleanup-expired-carts-manual-[0-9a-f]{8}$/);
  expect(body.metadata!.annotations!['cronjob.kubernetes.io/instantiate']).toBe('manual');
  expect(body.metadata!.ownerReferences![0]).toMatchObject({ kind: 'CronJob', uid: 'uid-1', controller: true });
  expect(body.spec!.template.spec!.containers[0].env).toContainEqual({ name: 'JOB_INPUT', value: '{"dryRun":true}' });

  // same key again → 409 from the API server is treated as success with the same runId
  (batch.createNamespacedJob as any).mockRejectedValueOnce(new k8s.ApiException(409, 'AlreadyExists', {}, {}));
  expect((await backend.runNow('cleanup-expired-carts', { dryRun: true }, { idempotencyKey: 'k-1', actor: 'ana' })).runId).toBe(runId);
});
```

For end-to-end coverage, run `jobs-api` against a `kind` cluster with the real CronJob manifests and a Redis container in CI: pause → assert `spec.suspend`, run now → wait for `Complete`, stop → assert pods got SIGTERM and the run ends as `cancelled`.

## 15. Build vs buy

| Option | You get | You give up | Pick it when |
|---|---|---|---|
| **Build `jobs-api`** (this note) | one API/UI over BullMQ + K8s CronJobs, your RBAC and audit, your domain language | ~2–4 weeks to build, and you maintain it | jobs already live in several runtimes and you need a single ops surface |
| **Temporal (Schedules + Web UI)** | pause/unpause/trigger/backfill, full run history, retries and durable state, cancellation | a Temporal cluster (or Cloud) and rewriting jobs as workflows | multi-step business processes, long waits, strong history needs |
| **Argo Workflows (CronWorkflows + UI)** | suspend/resume, submit from a CronWorkflow, DAGs, artifacts, K8s-native RBAC | everything becomes a workflow CRD; BullMQ jobs don't fit | K8s-heavy batch/data pipelines |
| **Apache Airflow** | DAG pause toggle, manual trigger with config, backfills, a large operator ecosystem | Python DAGs, a scheduler + metadata DB to run, slower feedback | data engineering / ETL teams |
| **Dagster** | schedules and sensors, asset lineage, launchpad for manual runs, good UI | asset-centric model; Python | data platforms that think in datasets, not tasks |
| **Bull Board / Taskforce.sh** | BullMQ-only admin UI in minutes | no K8s jobs, weak domain RBAC/audit (Taskforce has more) | BullMQ is your only runtime |

A senior answer is usually "buy the engine, build the thin layer": keep the native tools (Bull Board read-only, Argo CD for manifests, Grafana for history) and build only the missing piece, a small audited control API with the semantics your org needs. Re-evaluate when the number of jobs, runtimes or dependencies between jobs grows; that's the point where Temporal or Argo Workflows pays for itself.

## Pitfalls

<Warning>
**Common mistakes**

- Pausing a BullMQ job with `queue.pause()` (blocks manual runs, fires a catch-up on resume) instead of removing the scheduler.
- Unsuspending a CronJob without `startingDeadlineSeconds`: Kubernetes schedules a catch-up run immediately.
- Deleting a K8s Job with the default `Background` or `Orphan` propagation: the API returns before pods stop, or leaves pods running.
- Treating `202` from stop as "stopped". It means "cancellation requested"; the dashboard must follow the run to `cancelled`.
- No `Idempotency-Key`: a double click or a retrying proxy starts two runs.
- `runId`s with `:` or `/` put raw into URLs; always encode them.
- SSE through a proxy that buffers or times out idle connections; send heartbeats and `X-Accel-Buffering: no`.
- Giving `jobs-api` a ClusterRole or `pods/exec` "just in case".
- Logging tokens (Fastify logs headers unless redacted).
</Warning>

## Senior perspective

- **The interface is the product.** `JobBackend` lets you move a job from a K8s CronJob to BullMQ (or to Temporal) without changing the dashboard, the audit trail or the runbooks. Keep the semantics (pause, resume, run now, stop, overlap) identical across backends, and document where they can't be (BullMQ stop is cooperative; K8s stop is SIGTERM).
- **Control plane ≠ data plane.** If `jobs-api` is down, jobs must keep running on schedule. Nothing in the run path depends on it.
- **Audit is a feature.** Who paused the inventory sync during the incident, and why, is the first question in the postmortem. Make the reason mandatory for pause/stop in the UI.
- **Idempotency everywhere**: HTTP (Idempotency-Key), queue (job id), cluster (Job name), side effects (business keys).
- **Least privilege is testable**: `kubectl auth can-i` checks in CI, admission policies, and an alert if the Role changes.
- Review questions: *What does pause do to a run that's already queued? What if two operators click stop at once? Can a user run a job they can't see? What happens if Redis is down: do reads still work for K8s jobs? Where is the audit log retained and who can delete it?*

<Info>
**🆕 2025–2026**

- **2025-02-22 · jose 6.0.0**; 6.2.x current. `createRemoteJWKSet` + `jwtVerify` as used here.
- **2025-07-09 · Zod 4.0.0** (an alternative to TypeBox for schemas; Fastify has type providers for both).
- **2025-09-09 · TypeBox 1.0.0** published as the `typebox` package; **2025-09-20 · `@fastify/type-provider-typebox` 6.0.0** switched to it (peer `typebox ^1.0.13`). Old code imports `@sinclair/typebox`.
- **2025-10-03 · `@kubernetes/client-node` 1.4.0:** Kubernetes 1.34 APIs, AbortSignal support. **2026-08-12 · 2.0.0:** HTTP layer moved from node-fetch to **undici**, dropped Node 18, 20 and 23, generated against Kubernetes 1.36; the object-parameter API style from 1.0 is unchanged.
- **2026-04-18:** the IETF *Idempotency-Key HTTP Header Field* draft reached **-07**; still a draft, but its semantics (400 missing, 409 in flight, 422 reused with a different payload) are what this API implements.
- **2026-07-30 · BullMQ 6.0.0** removed repeatable jobs; the BullMQ backend uses only Job Scheduler APIs (see [BullMQ Job Schedulers in Practice](/Software%20Engineering/Backend/Scheduled%20Jobs/BullMQ%20Job%20Schedulers%20in%20Practice)).
</Info>

## Interview questions

<Accordion title="How do you give one API over jobs running on different schedulers?">
Define a backend-neutral resource model (job, state, run) and an adapter interface (`list`, `get`, `pause`, `resume`, `runNow`, `stopRun`, `listRuns`, `getRunLogs`). Each backend maps the semantics: BullMQ removes/re-creates the Job Scheduler, K8s patches `spec.suspend`, run now is `queue.add` vs creating a Job from the CronJob template, stop is AbortSignal vs deleting the Job. A router picks the backend by job id.
</Accordion>

<Accordion title="What exactly does &quot;pause&quot; mean, and how is it different on BullMQ and Kubernetes?">
Stop creating new scheduled runs; in-flight runs continue and manual runs still work. BullMQ: remove the Job Scheduler (after stashing its definition) and record a paused flag so deploy-time reconciliation doesn't re-add it; not `queue.pause()`, which blocks all processing. Kubernetes: `spec.suspend: true` via a merge patch; resume with `startingDeadlineSeconds` set to avoid an immediate catch-up run.
</Accordion>

<Accordion title="How is &quot;run now&quot; implemented for a CronJob, and why the owner reference?">
Read the CronJob, copy `spec.jobTemplate` into a new Job with the annotation `cronjob.kubernetes.io/instantiate: manual` and a controller owner reference to the CronJob, which is what `kubectl create job --from=cronjob/x` does. The owner reference means history limits clean it up, `Forbid` counts it as active, and deleting the CronJob cascades. A deterministic name from the Idempotency-Key makes retries return 409, which we treat as success.
</Accordion>

<Accordion title="How do you stop an in-flight run safely?">
Cooperatively. BullMQ: set a durable cancel flag and publish a cancel message; the worker holding the job calls `cancelJob`, the processor's `AbortSignal` fires, it cleans up and throws `UnrecoverableError('cancelled: …')`. Kubernetes: delete the Job with `propagationPolicy: Foreground`, so pods get SIGTERM and `terminationGracePeriodSeconds` before SIGKILL, and the Job object disappears only after its pods. The API returns 202 and the status moves `cancelling` → `cancelled`.
</Accordion>

<Accordion title="How do you implement Idempotency-Key correctly?">
Store `(principal, key)` with a hash of the request and, once done, the response. On repeat: same hash and a stored response → replay it; different hash → 422; still in flight → 409; missing key → 400. Delete the record if the operation failed so the client can retry. Also derive downstream ids (queue job id, K8s Job name) from the key so duplicates are rejected even if the store is lost.
</Accordion>

<Accordion title="What K8s permissions does this service need, and what's risky about them?">
`get/list/watch/patch` on CronJobs (patch pinned with `resourceNames`), `get/list/watch/create/delete` on Jobs, `list` on pods and `get` on `pods/log`, in one namespace via a Role. Creating Jobs means creating Pods with any image and ServiceAccount in that namespace, so isolate the namespace and add an admission policy restricting what `jobs-api` may create.
</Accordion>

<Accordion title="Why RFC 9457 Problem Details?">
A standard, machine-readable error shape (`type`, `title`, `status`, `detail`, `instance` plus extensions) with its own media type `application/problem+json`. Clients branch on `type` instead of parsing messages, and every error from every route looks the same.
</Accordion>

<Accordion title="Build vs buy for a job control plane?">
Buy the engine (BullMQ, K8s, Temporal, Argo Workflows, Airflow, Dagster) and prefer its native UI when you have one runtime. Build a thin control API only when you need one audited surface over several runtimes with your own RBAC and semantics. Revisit when jobs gain dependencies and history needs: that's when a workflow engine is cheaper than more glue.
</Accordion>

## Related
- [Scheduled Jobs Index](/Software%20Engineering/Backend/Scheduled%20Jobs/Scheduled%20Jobs%20Index) · [Backend Index](/Software%20Engineering/Backend/Backend%20Index) · [APIs & Auth Index](/Software%20Engineering/APIs%20%26%20Auth/APIs%20%26%20Auth%20Index)
- [Cron with Message Queues - Architecture](/Software%20Engineering/Backend/Scheduled%20Jobs/Cron%20with%20Message%20Queues%20-%20Architecture) · [BullMQ Job Schedulers in Practice](/Software%20Engineering/Backend/Scheduled%20Jobs/BullMQ%20Job%20Schedulers%20in%20Practice) · [Kubernetes CronJobs in Practice](/Software%20Engineering/Backend/Scheduled%20Jobs/Kubernetes%20CronJobs%20in%20Practice) · [Job Scheduler Dashboard with Next.js](/Software%20Engineering/Backend/Scheduled%20Jobs/Job%20Scheduler%20Dashboard%20with%20Next.js) · [Playwright Automation as a Cron Job](/Software%20Engineering/Backend/Scheduled%20Jobs/Playwright%20Automation%20as%20a%20Cron%20Job)
- [REST API Design](/Software%20Engineering/APIs%20%26%20Auth/REST%20API%20Design) · [API Versioning, Idempotency & Backward Compatibility](/Software%20Engineering/APIs%20%26%20Auth/API%20Versioning%2C%20Idempotency%20%26%20Backward%20Compatibility) · [Rate Limiting & Throttling](/Software%20Engineering/APIs%20%26%20Auth/Rate%20Limiting%20%26%20Throttling) · [Webhooks, Polling, SSE & WebSockets](/Software%20Engineering/APIs%20%26%20Auth/Webhooks%2C%20Polling%2C%20SSE%20%26%20WebSockets)
- [OAuth 2.x & OpenID Connect](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect) · [Authentication vs Authorization](/Software%20Engineering/APIs%20%26%20Auth/Authentication%20vs%20Authorization) · [API Security](/Software%20Engineering/Security/API%20Security)
- [Deploying Node.js Apps to Kubernetes](/Software%20Engineering/DevOps/Delivery%20Platform/Deploying%20Node.js%20Apps%20to%20Kubernetes) · [Kubernetes Operations - Logs, Debugging & Scheduling](/Software%20Engineering/DevOps/Delivery%20Platform/Kubernetes%20Operations%20-%20Logs%2C%20Debugging%20%26%20Scheduling)

## Sources
- [RFC 9457: Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457)
- [IETF draft: The Idempotency-Key HTTP Header Field](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/)
- [WHATWG HTML: Server-sent events](https://html.spec.whatwg.org/multipage/server-sent-events.html)
- [Fastify docs](https://fastify.dev/docs/latest/) · [Type providers](https://fastify.dev/docs/latest/Reference/Type-Providers/) · [@fastify/type-provider-typebox](https://github.com/fastify/fastify-type-provider-typebox) · [@fastify/rate-limit](https://github.com/fastify/fastify-rate-limit) · [TypeBox](https://github.com/sinclairzx81/typebox)
- [jose: jwtVerify and createRemoteJWKSet](https://github.com/panva/jose)
- [@kubernetes/client-node README and examples](https://github.com/kubernetes-client/javascript) (incl. `in-cluster-create-job-from-cronjob.js`, `follow-logs.js`, `patch-example.js`) · [Releases (1.4.0, 2.0.0)](https://github.com/kubernetes-client/javascript/releases)
- [Kubernetes: CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) · [Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/) · [Garbage collection and propagation policies](https://kubernetes.io/docs/concepts/architecture/garbage-collection/) · [Using RBAC authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) · [Validating Admission Policy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/)
- [kubectl create job --from](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_job/)
- [BullMQ: Job Schedulers](https://docs.bullmq.io/guide/job-schedulers) · [Cancelling jobs](https://docs.bullmq.io/guide/workers/cancelling-jobs) · [Events](https://docs.bullmq.io/guide/events)
- [Temporal: Schedules](https://docs.temporal.io/schedule) · [Argo Workflows: CronWorkflows](https://argo-workflows.readthedocs.io/en/latest/cron-workflows/) · [Airflow: DAG runs](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dag-run.html) · [Dagster: Schedules](https://docs.dagster.io/guides/automate/schedules)
- [Testcontainers for Node](https://node.testcontainers.org/)
