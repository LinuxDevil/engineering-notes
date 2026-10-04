---
title: "BullMQ Job Schedulers in Practice"
tags: [backend, backend/jobs, backend/node, devops/kubernetes]
aliases: [BullMQ Job Schedulers, upsertJobScheduler, BullMQ in Production, jobs-worker, BullMQ Cancellation, BullMQ v6]
updated: 2026-09-27
---
# BullMQ Job Schedulers in Practice

<Note>
**TL;DR**

BullMQ runs recurring work with **Job Schedulers**: `queue.upsertJobScheduler(id, { pattern, tz }, template)` keeps exactly one delayed job for the next tick with a deterministic id `repeat:<id>:<millis>`, so N replicas upserting the same scheduler never duplicate ticks. Manage them with `getJobSchedulers()`, `getJobScheduler(id)` and `removeJobScheduler(id)`. Since **BullMQ 6.0.0 (2026-07-30)** the old repeatable-jobs API is **removed**. Build workers with concurrency, rate limits, `attempts` + backoff, `removeOnComplete`/`removeOnFail`, and sandboxed processors for CPU work; stop runs with the built-in **`worker.cancelJob()` + `AbortSignal`** (since 5.64.0), reached across pods via Redis pub/sub; shut down on SIGTERM with `worker.close()`; export metrics with `queue.exportPrometheusMetrics()`; and run Redis with **`maxmemory-policy noeviction`** and AOF.
</Note>

This note builds the **`jobs-worker`** of Acme Ops (Node 24 + TypeScript) for the two BullMQ jobs, `nightly-sales-report` and `sync-inventory`. The architecture and the reasoning behind idempotency keys, overlap and misfires are in [Cron with Message Queues - Architecture](/Software%20Engineering/Backend/Scheduled%20Jobs/Cron%20with%20Message%20Queues%20-%20Architecture); the control plane that pauses, triggers and stops these jobs is [Job Control API - Pause, Resume, Run Now & Stop](/Software%20Engineering/Backend/Scheduled%20Jobs/Job%20Control%20API%20-%20Pause%2C%20Resume%2C%20Run%20Now%20%26%20Stop).

## 1. Versions and API status (checked 2026-09-27)

| Version | Date | What matters here |
|---|---|---|
| 5.16.0 | 2024-10-06 | `upsertJobScheduler`, `getJobSchedulers`, `removeJobScheduler` added; Job Schedulers replace repeatable jobs |
| 5.19.0 | 2024-10-11 | `immediately` deprecated: a **new** `every` scheduler always fires its first job right away |
| 5.37.0 | 2025-01-25 | built-in **Prometheus exporter** (`queue.exportPrometheusMetrics()`) |
| 5.64.0 | 2025-11-20 | **job cancellation**: `worker.cancelJob(id, reason)`, `worker.cancelAllJobs()`, `AbortSignal` as the processor's 3rd argument |
| 5.70.0 | 2026-02-21 | cancellation also reaches **sandboxed** processors |
| 6.0.0 | 2026-07-30 | pluggable backends (Redis default, **PostgreSQL** optional); legacy repeatable APIs removed; `ioredis` becomes an optional peer dependency; `Queue#client` and other Redis internals removed; `debounce` removed (use `deduplication`); `paused` state removed from `getJobCounts()` |
| 6.3.9 | 2026-09-25 | latest at the time of writing |

Removed in v6 and still common in blog posts: `queue.add(name, data, { repeat })`, `getRepeatableJobs()`, `removeRepeatable()`, `removeRepeatableByKey()`, `repeat.utc` (use `tz: 'UTC'`). Migrate on the latest 5.x first; v6 **throws** if it finds legacy repeatable metadata in Redis.

## 2. Project layout

```text
acme/jobs                      (pnpm monorepo)
├── packages/jobs-core/        shared by jobs-worker and jobs-api
│   └── src/{registry.ts, keys.ts, redis.ts, templates.ts, index.ts}
├── apps/jobs-worker/
│   ├── src/
│   │   ├── main.ts                    boot, reconcile schedulers, workers, shutdown
│   │   ├── run-context.ts             misfire, timeout, cancellation wrapper
│   │   ├── control.ts                 Redis pub/sub cancel channel
│   │   ├── metrics.ts                 prom-client + BullMQ exporter on :9464
│   │   ├── processors/sync-inventory.ts
│   │   ├── processors/nightly-sales-report.ts
│   │   └── processors/report-part.sandbox.ts   (sandboxed, CPU heavy)
│   └── test/
└── apps/jobs-api/             see [[Job Control API - Pause, Resume, Run Now & Stop]]
```

```bash
pnpm add bullmq ioredis pino prom-client cron-parser pg @aws-sdk/client-s3 @aws-sdk/s3-request-presigner
pnpm add -D typescript vitest @testcontainers/redis @types/node
```

## 3. The shared registry and Redis keys

The registry is the source of truth for **what** runs **when**. The worker reconciles it into Redis on boot; `jobs-api` reads the same package.

```ts
// packages/jobs-core/src/registry.ts
export type Backend = 'bullmq' | 'k8s';
export type ConcurrencyPolicy = 'Allow' | 'Forbid' | 'Replace';

export interface JobDefinition {
  id: string;                 // also the BullMQ queue name and scheduler id (no ':' allowed)
  name: string;
  description: string;
  owner: string;
  backend: Backend;
  schedule: string;           // 5-field cron
  timezone: string;           // IANA
  concurrencyPolicy: ConcurrencyPolicy;
  timeoutSeconds: number;
  retries: number;            // retries after the first attempt
  misfireGraceSeconds?: number;
  expectedIntervalSeconds: number;  // feeds the "missed schedule" alert
}

export const JOBS: readonly JobDefinition[] = [
  {
    id: 'nightly-sales-report', name: 'Nightly sales report', owner: 'team-finance-eng',
    description: "Builds yesterday's sales CSV per region, uploads it to S3 and emails a signed link.",
    backend: 'bullmq', schedule: '0 2 * * *', timezone: 'Europe/Berlin',
    concurrencyPolicy: 'Forbid', timeoutSeconds: 1800, retries: 3,
    misfireGraceSeconds: 4 * 3600, expectedIntervalSeconds: 86_400,
  },
  {
    id: 'sync-inventory', name: 'Sync inventory', owner: 'team-catalog',
    description: 'Pulls stock levels from supplier APIs and upserts them into Postgres.',
    backend: 'bullmq', schedule: '*/15 * * * *', timezone: 'Europe/Berlin',
    concurrencyPolicy: 'Forbid', timeoutSeconds: 600, retries: 5,
    misfireGraceSeconds: 300, expectedIntervalSeconds: 900,
  },
  {
    id: 'cleanup-expired-carts', name: 'Cleanup expired carts', owner: 'team-checkout',
    description: 'Deletes carts older than 7 days.',
    backend: 'k8s', schedule: '0 * * * *', timezone: 'Europe/Berlin',
    concurrencyPolicy: 'Forbid', timeoutSeconds: 900, retries: 2, expectedIntervalSeconds: 3600,
  },
  {
    id: 'playwright-checkout-smoke', name: 'Checkout smoke test', owner: 'team-qa',
    description: 'Synthetic Playwright check of the checkout flow on production.',
    backend: 'k8s', schedule: '*/10 * * * *', timezone: 'Europe/Berlin',
    concurrencyPolicy: 'Forbid', timeoutSeconds: 300, retries: 0, expectedIntervalSeconds: 600,
  },
];

export const bullJobs = () => JOBS.filter((j) => j.backend === 'bullmq');
export const getJob = (id: string) => JOBS.find((j) => j.id === id);
```

```ts
// packages/jobs-core/src/keys.ts   (our own keys; ':' is fine here, only BullMQ job ids forbid it)
export const KEYS = {
  paused: 'acme-jobs:paused',                         // SET of paused BullMQ job ids
  cancel: (runId: string) => `acme-jobs:cancel:${runId}`, // STRING reason, TTL 24 h
  control: 'acme-jobs:control',                       // pub/sub channel for cancel requests
} as const;

export type ControlMessage = { action: 'cancel'; queue: string; runId: string; reason: string };

// Scheduler-produced ids look like repeat:<schedulerId>:<millis>
export function scheduledTimeOf(runId: string): number | undefined {
  const m = /^repeat:.+:(\d+)$/.exec(runId);
  return m ? Number(m[1]) : undefined;
}
```

## 4. Redis and connections

### Redis settings BullMQ needs

```conf
# redis.conf (or Helm values for the Redis in shop-prod)
maxmemory 2gb
maxmemory-policy noeviction     # the ONLY safe policy: evicting a key corrupts queues
appendonly yes                  # AOF persistence
appendfsync everysec            # ~1 s of writes at risk, good default
```

With any `allkeys-*`/`volatile-*` policy, Redis silently deletes job hashes or scheduler keys under memory pressure: jobs vanish, schedulers stop. With `noeviction`, writes fail loudly instead, and your memory alert fires. Size memory for `removeOnComplete`/`removeOnFail` retention, not "forever".

### Connection options

BullMQ creates its connections with ioredis (install it explicitly in v6). Producers should **fail fast**; workers should **wait forever**:

```ts
// packages/jobs-core/src/redis.ts
import type { RedisOptions } from 'ioredis';

const base: RedisOptions = {
  host: process.env.REDIS_HOST ?? 'redis-master.shop-prod.svc',
  port: Number(process.env.REDIS_PORT ?? 6379),
  password: process.env.REDIS_PASSWORD,
  // reconnect forever with capped exponential backoff (BullMQ's own default looks similar)
  retryStrategy: (times) => Math.max(Math.min(Math.exp(times), 20_000), 1_000),
};

/** For Worker/QueueEvents: block and retry, never throw on a transient disconnect. */
export const workerConnection: RedisOptions = { ...base, maxRetriesPerRequest: null };

/** For Queue in HTTP handlers: fail fast so the API returns 503 instead of hanging. */
export const producerConnection: RedisOptions = { ...base, enableOfflineQueue: false };
```

## 5. Reconciling schedulers from the registry

```ts
// packages/jobs-core/src/templates.ts   (shared: jobs-api's Resume and Run now use the same options)
import type { JobDefinition } from './registry.js';

export function templateFor(def: JobDefinition) {
  return {
    name: def.id,
    data: { trigger: 'schedule' as const },
    opts: {
      attempts: def.retries + 1,
      backoff: { type: 'exponential' as const, delay: 30_000 }, // 30 s, 60 s, 120 s ...
      removeOnComplete: { age: 14 * 24 * 3600, count: 500 },   // keep 14 days / 500 runs
      removeOnFail: { age: 30 * 24 * 3600 },                    // keep failures 30 days
    },
  };
}
```

```ts
// apps/jobs-worker/src/schedulers.ts
import { Queue } from 'bullmq';
import type { Redis } from 'ioredis';
import { KEYS, templateFor, type JobDefinition } from '@acme/jobs-core';

export async function reconcileScheduler(queue: Queue, def: JobDefinition, redis: Redis) {
  // Forbid ⇒ at most one run of this job at a time, across ALL worker pods
  if (def.concurrencyPolicy === 'Forbid') await queue.setGlobalConcurrency(1);
  else await queue.removeGlobalConcurrency();

  if (await redis.sismember(KEYS.paused, def.id)) {
    await queue.removeJobScheduler(def.id);           // paused: keep it removed
    return 'paused';
  }
  await queue.upsertJobScheduler(def.id, { pattern: def.schedule, tz: def.timezone }, templateFor(def));

  // close the race with a concurrent "pause" from jobs-api
  if (await redis.sismember(KEYS.paused, def.id)) await queue.removeJobScheduler(def.id);
  return 'active';
}

/** Remove schedulers that exist in Redis but no longer in code (orphans). */
export async function removeOrphans(queue: Queue, def: JobDefinition) {
  for (const s of await queue.getJobSchedulers(0, -1, true)) {
    if (s.key !== def.id) await queue.removeJobScheduler(s.key);
  }
}
```

`getJobScheduler(id)` returns a `JobSchedulerJson`: `{ key, name, pattern?, every?, tz?, next?, iterationCount?, limit?, startDate?, endDate?, template? }`. `next` (epoch ms) is what `jobs-api` shows as `nextRunAt`.

<Warning>
**Scheduler gotchas**

- **`every` vs `pattern`**: since 5.19 a *new* `every` scheduler fires immediately, and `every` aligns to the clock (every 2000 ms fires at even seconds). Resume-by-reinsert of an `every` scheduler would therefore run at once. Acme uses cron `pattern`s, whose first job waits for the next match.
- **Next job is created when the current one starts**. If the queue is busy or paused, ticks are produced *less* often than the pattern, never more.
- **Custom job ids are not allowed** on scheduler jobs; use the job `name` to tell them apart.
- **Patterns** use cron-parser: 5 fields, or 6 with seconds first (`'0 15 3 * * *'` = 03:15:00). Don't copy Quartz expressions.
</Warning>

## 6. Workers: concurrency, rate limits, retries, retention

```ts
// apps/jobs-worker/src/main.ts
import { Queue, Worker, QueueEvents } from 'bullmq';
import { Redis } from 'ioredis';
import pino from 'pino';
import { bullJobs, workerConnection, KEYS } from '@acme/jobs-core';
import { reconcileScheduler } from './schedulers.js';
import { withRunContext } from './run-context.js';
import { syncInventory } from './processors/sync-inventory.js';
import { nightlySalesReport } from './processors/nightly-sales-report.js';
import { startControlChannel } from './control.js';
import { startMetricsServer, observeWorker } from './metrics.js';

const log = pino({ base: { app: 'jobs-worker' } });
const redis = new Redis(workerConnection);          // our own keys (paused set, cancel flags)

const processors = {
  'sync-inventory': syncInventory,
  'nightly-sales-report': nightlySalesReport,
} as const;

const workers: Worker[] = [];
const queues: Queue[] = [];

for (const def of bullJobs()) {
  const queue = new Queue(def.id, { connection: workerConnection });
  queues.push(queue);
  log.info({ job: def.id, state: await reconcileScheduler(queue, def, redis) }, 'scheduler reconciled');

  const worker = new Worker(def.id, withRunContext(def, processors[def.id as keyof typeof processors], redis), {
    connection: workerConnection,
    concurrency: 1,                    // per pod; global concurrency (1) caps it across pods
    lockDuration: 60_000,              // lock renewed automatically while the event loop is free
    maxStalledCount: 1,
    autorun: false,
  });
  worker.on('error', (err) => log.error({ err, job: def.id }, 'worker error'));
  worker.on('failed', (job, err) => log.warn({ job: def.id, run_id: job?.id, err: err.message }, 'run failed'));
  observeWorker(def, worker);
  workers.push(worker);
}

// Fan-out queue for report parts: CPU heavy → sandboxed processor in worker threads
const partsWorker = new Worker(
  'report-parts',
  new URL('./processors/report-part.sandbox.js', import.meta.url),   // a file URL (recommended on Windows too)
  {
    connection: workerConnection,
    concurrency: 4,
    useWorkerThreads: true,
    limiter: { max: 20, duration: 1_000 },   // ≤ 20 part jobs started per second per queue
  },
);
workers.push(partsWorker);

const control = await startControlChannel(workers, log);   // cancel requests from jobs-api
const metricsServer = startMetricsServer(queues, log);

for (const w of workers) void w.run();
log.info('jobs-worker started');

// ---------- graceful shutdown (see §9) ----------
let shuttingDown = false;
async function shutdown(signal: string) {
  if (shuttingDown) return;
  shuttingDown = true;
  log.info({ signal }, 'shutting down: finishing active jobs');
  const hardStop = setTimeout(() => {
    log.error('grace period exceeded, cancelling active jobs');
    for (const w of workers) w.cancelAllJobs('worker shutdown');   // processors abort and fail → retried
  }, 50_000);                                                        // < terminationGracePeriodSeconds (60)
  await Promise.allSettled(workers.map((w) => w.close()));           // stop fetching, wait for active jobs
  clearTimeout(hardStop);
  await Promise.allSettled([...queues.map((q) => q.close()), control.close(), redis.quit()]);
  metricsServer.close();
  process.exit(0);
}
process.on('SIGTERM', () => void shutdown('SIGTERM'));
process.on('SIGINT', () => void shutdown('SIGINT'));
```

Options cheat sheet:

| Option | Where | Meaning |
|---|---|---|
| `concurrency` | Worker | jobs processed in parallel **by this worker instance** |
| `queue.setGlobalConcurrency(n)` | Queue | cap across **all** workers (Acme's Forbid = 1) |
| `limiter: { max, duration }` | Worker | at most `max` jobs started per `duration` ms for the queue (protects downstream APIs) |
| `attempts`, `backoff` | job opts | retries; `{ type: 'exponential', delay }` doubles each attempt; custom strategies via `settings.backoffStrategy` |
| `UnrecoverableError` | thrown in processor | fail now, don't retry (permanent error, cancellation) |
| `removeOnComplete` / `removeOnFail` | job opts | `true`, a count, or `{ age, count }`; default is keep forever, which fills Redis |
| `lockDuration`, `maxStalledCount` | Worker | lock TTL (renewed automatically) and how often a job may stall before failing |
| `deduplication: { id, ttl? }` | job opts | ignore/replace duplicates while a job with that id is pending (simple, throttle, debounce modes) |

## 7. The run context: misfire, timeout and cancellation

One wrapper gives every processor the same semantics as the Control API: **Forbid** via a misfire grace, a **hard timeout**, and **cooperative cancellation**.

```ts
// apps/jobs-worker/src/run-context.ts
import { UnrecoverableError, type Job } from 'bullmq';
import type { Redis } from 'ioredis';
import { KEYS, scheduledTimeOf, type JobDefinition } from '@acme/jobs-core';

export interface RunContext {
  job: Job;
  signal: AbortSignal;                        // aborts on cancel OR timeout
  log: (msg: string) => Promise<void>;        // visible in the dashboard via job logs
  progress: (done: number, total: number) => Promise<void>;
  scheduledAt: Date;
}
export type Processor<R> = (ctx: RunContext, token?: string) => Promise<R>;

export function withRunContext<R>(def: JobDefinition, fn: Processor<R>, redis: Redis) {
  return async (job: Job, token?: string, signal?: AbortSignal): Promise<R | { skipped: string }> => {
    const runId = job.id!;
    const scheduledMs = scheduledTimeOf(runId) ?? job.timestamp;

    // 1. Forbid semantics: a scheduled tick that starts too late is skipped, not run
    const lateBy = Date.now() - scheduledMs;
    if (job.data?.trigger === 'schedule' && def.misfireGraceSeconds && lateBy > def.misfireGraceSeconds * 1000) {
      await job.log(`skipped: started ${Math.round(lateBy / 1000)}s late (misfire grace ${def.misfireGraceSeconds}s)`);
      return { skipped: 'misfire' };
    }

    // 2. Cancelled while it was still queued?
    const pre = await redis.get(KEYS.cancel(runId));
    if (pre !== null) throw new UnrecoverableError(`cancelled: ${pre}`);

    // 3. Combine: BullMQ's cancel signal + hard timeout + durable cancel flag (fallback if pub/sub was missed)
    const flag = new AbortController();
    const timeout = AbortSignal.timeout(def.timeoutSeconds * 1000);
    const combined = AbortSignal.any([timeout, flag.signal, ...(signal ? [signal] : [])]);
    const poll = setInterval(async () => {
      const reason = await redis.get(KEYS.cancel(runId)).catch(() => null);
      if (reason !== null) flag.abort(reason);
    }, 5_000);

    try {
      return await fn({
        job, signal: combined, scheduledAt: new Date(scheduledMs),
        log: (msg) => job.log(`${new Date().toISOString()} ${msg}`).then(() => undefined),
        progress: (done, total) => job.updateProgress({ done, total }),
      }, token);
    } catch (err) {
      if (combined.aborted && !timeout.aborted) {
        // user/operator cancellation: permanent, no retry; jobs-api maps this prefix to status "cancelled"
        throw new UnrecoverableError(`cancelled: ${String(combined.reason ?? 'no reason')}`);
      }
      if (timeout.aborted) throw new Error(`timeout after ${def.timeoutSeconds}s`); // retryable
      throw err;
    } finally {
      clearInterval(poll);
    }
  };
}
```

### The cancel channel

`worker.cancelJob()` only reaches jobs **held by that worker instance**. `jobs-api` doesn't know which pod runs the job, so it (1) sets a durable flag and (2) broadcasts on Redis pub/sub; every pod tries `cancelJob`, and the one that owns the job aborts its signal. This is the pattern BullMQ's own cancellation guide recommends for cross-process cancellation.

```ts
// apps/jobs-worker/src/control.ts
import type { Worker } from 'bullmq';
import { Redis } from 'ioredis';
import type { Logger } from 'pino';
import { KEYS, workerConnection, type ControlMessage } from '@acme/jobs-core';

export async function startControlChannel(workers: Worker[], log: Logger) {
  const sub = new Redis(workerConnection);           // a subscribed connection can't run other commands
  await sub.subscribe(KEYS.control);
  sub.on('message', (_channel, raw) => {
    const msg = JSON.parse(raw) as ControlMessage;
    if (msg.action !== 'cancel') return;
    for (const w of workers) {
      if (w.name === msg.queue && w.cancelJob(msg.runId, msg.reason)) {
        log.info({ job: msg.queue, run_id: msg.runId, reason: msg.reason }, 'cancel signalled');
      }
    }
  });
  return { close: () => sub.quit() };
}
```

Cancellation is **cooperative**: code that ignores the signal keeps running. Pass `signal` to everything that accepts one (`fetch`, `pg` query cancellation via `AbortSignal`-aware wrappers, AWS SDK `abortSignal`), and check `signal.throwIfAborted()` between steps.

## 8. The processors

### `sync-inventory`: I/O bound, inline

```ts
// apps/jobs-worker/src/processors/sync-inventory.ts
import pg from 'pg';
import type { Processor } from '../run-context.js';

const pool = new pg.Pool({ connectionString: process.env.DATABASE_URL, max: 5 });
const SUPPLIERS = ['alpha', 'bravo', 'charlie'] as const;

type Item = { sku: string; quantity: number; updatedAt: string };

export const syncInventory: Processor<{ upserted: number }> = async ({ signal, log, progress }) => {
  let upserted = 0;
  for (const [i, supplier] of SUPPLIERS.entries()) {
    signal.throwIfAborted();
    const res = await fetch(`https://api.${supplier}.example/v1/stock?changedSince=PT1H`, {
      headers: { authorization: `Bearer ${process.env[`SUPPLIER_${supplier.toUpperCase()}_TOKEN`]}` },
      signal,                                             // cancellation aborts the HTTP request
    });
    if (res.status === 429) throw new Error(`${supplier}: rate limited`);   // retry with backoff
    if (!res.ok) throw new Error(`${supplier}: HTTP ${res.status}`);
    const items = (await res.json()) as Item[];

    // idempotent upsert: re-running the same tick converges to the same rows
    const { rowCount } = await pool.query(
      `INSERT INTO inventory (sku, supplier, quantity, supplier_updated_at)
       SELECT x.sku, $1, x.quantity, x.updated_at
       FROM jsonb_to_recordset($2::jsonb) AS x(sku text, quantity int, updated_at timestamptz)
       ON CONFLICT (sku) DO UPDATE
         SET quantity = EXCLUDED.quantity, supplier_updated_at = EXCLUDED.supplier_updated_at
         WHERE inventory.supplier_updated_at < EXCLUDED.supplier_updated_at`,
      [supplier, JSON.stringify(items.map((x) => ({ sku: x.sku, quantity: x.quantity, updated_at: x.updatedAt })))],
    );
    upserted += rowCount ?? 0;
    await log(`${supplier}: ${items.length} items, ${rowCount} upserted`);
    await progress(i + 1, SUPPLIERS.length);
  }
  return { upserted };
};
```

### `nightly-sales-report`: parent with sandboxed children

The parent fans out one child per region into `report-parts`, waits for them with `moveToWaitingChildren`, then merges the CSV parts. Heavy CSV work runs in worker threads, so the parent's lock renewal is never blocked.

```ts
// apps/jobs-worker/src/processors/nightly-sales-report.ts
import { Queue, WaitingChildrenError } from 'bullmq';
import { S3Client, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { workerConnection } from '@acme/jobs-core';
import type { Processor } from '../run-context.js';

const parts = new Queue('report-parts', { connection: workerConnection });
const s3 = new S3Client({});
const REGIONS = ['DE', 'AT', 'CH'] as const;
enum Step { FanOut = 'fan-out', Wait = 'wait', Assemble = 'assemble' }

/** "Yesterday" in Europe/Berlin relative to the SCHEDULED time, never "now" (DST/misfire safe). */
function reportDate(scheduledAt: Date): string {
  const d = new Date(scheduledAt.getTime() - 24 * 3600 * 1000);
  return new Intl.DateTimeFormat('en-CA', { timeZone: 'Europe/Berlin' }).format(d); // YYYY-MM-DD
}

export const nightlySalesReport: Processor<{ key: string }> = async ({ job, signal, log, scheduledAt }, token) => {
  const date = job.data.reportDate ?? reportDate(scheduledAt);
  let step: Step = job.data.step ?? Step.FanOut;

  while (true) {
    signal.throwIfAborted();
    switch (step) {
      case Step.FanOut: {
        await parts.addBulk(REGIONS.map((region) => ({
          name: 'report-part',
          data: { date, region, parentRunId: job.id },
          opts: {
            jobId: `part-${date}-${region}-${job.id!.replaceAll(':', '_')}`, // no ':' in custom ids
            parent: { id: job.id!, queue: job.queueQualifiedName },
            attempts: 3, backoff: { type: 'exponential', delay: 10_000 },
            removeOnComplete: { age: 7 * 24 * 3600 },
          },
        })));
        await job.updateData({ ...job.data, reportDate: date, step: Step.Wait });
        step = Step.Wait;
        await log(`fanned out ${REGIONS.length} parts for ${date}`);
        break;
      }
      case Step.Wait: {
        if (await job.moveToWaitingChildren(token!)) throw new WaitingChildrenError(); // parent sleeps
        await job.updateData({ ...job.data, step: Step.Assemble });
        step = Step.Assemble;
        break;
      }
      case Step.Assemble: {
        const values = await job.getChildrenValues<{ csv: string }>();
        const csv = ['region,order_id,total_eur', ...Object.values(values).map((v) => v.csv)].join('\n');
        const key = `reports/sales/${date}.csv`;               // same key on re-run: overwrite is harmless
        await s3.send(new PutObjectCommand({ Bucket: process.env.REPORTS_BUCKET, Key: key, Body: csv }), { abortSignal: signal });
        const url = await getSignedUrl(s3, new GetObjectCommand({ Bucket: process.env.REPORTS_BUCKET, Key: key }), { expiresIn: 7 * 24 * 3600 });
        await sendReportEmailOnce(date, url);                  // INSERT INTO report_emails(date) ON CONFLICT DO NOTHING, then send
        await log(`uploaded s3://${process.env.REPORTS_BUCKET}/${key}`);
        return { key };
      }
    }
  }
};

declare function sendReportEmailOnce(date: string, url: string): Promise<void>;
```

```ts
// apps/jobs-worker/src/processors/report-part.sandbox.ts   (runs in a worker thread)
import type { SandboxedJob } from 'bullmq';
import pg from 'pg';

export default async function (job: SandboxedJob<{ date: string; region: string }>, _token?: string, signal?: AbortSignal) {
  const pool = new pg.Pool({ connectionString: process.env.DATABASE_URL, max: 1 });
  try {
    const { rows } = await pool.query(
      `SELECT order_id, total_eur FROM orders
       WHERE region = $1 AND (created_at AT TIME ZONE 'Europe/Berlin')::date = $2::date`,
      [job.data.region, job.data.date],
    );
    const lines: string[] = [];
    for (const [i, r] of rows.entries()) {
      if (i % 10_000 === 0) signal?.throwIfAborted();          // cancellation checkpoint
      lines.push(`${job.data.region},${r.order_id},${r.total_eur}`);
    }
    await job.updateProgress(100);
    return { csv: lines.join('\n') };
  } finally {
    await pool.end();
  }
}
```

## 9. Pausing and stopping: which knob does what

| Action | Effect on **future** scheduled runs | Effect on **manual** runs | Effect on **active** runs | Use it for |
|---|---|---|---|---|
| `removeJobScheduler(id)` | stops them (removes the scheduler **and** its pending delayed job) | still run | continue | Acme **Pause** (store the paused flag so reconcile doesn't re-add it) |
| `upsertJobScheduler(id, …)` | resumes from the next pattern match, no backfill | n/a | n/a | Acme **Resume** |
| `queue.pause()` | jobs are still produced but nobody processes them; they pile up as waiting | blocked too | continue | emergency brake for a whole queue (e.g. a downstream outage) |
| `worker.pause()` | this worker stops fetching; other pods continue | same | waits for them (or `pause(true)` to not wait) | draining one pod, debugging |
| `worker.close()` | this worker stops for good | same | waits for them to finish | graceful shutdown on SIGTERM |
| `worker.cancelJob(id)` | n/a | n/a | aborts that job's `AbortSignal` | Acme **Stop** (via pub/sub) |

Why Acme's Pause removes the scheduler instead of pausing the queue: the spec says *Pause stops creating new scheduled runs* while **Run now** keeps working. `queue.pause()` would also block manual runs, and the tick that was waiting would fire the moment you resume (an unwanted catch-up).

**Graceful shutdown** (in `main.ts` above): on SIGTERM, `worker.close()` stops fetching and waits for active jobs. It never times out by itself, so the process sets its own deadline below `terminationGracePeriodSeconds` and then calls `cancelAllJobs()`; the aborted jobs fail with a retryable error and another pod picks them up. If the pod is SIGKILLed anyway, the lock expires and the **stalled checker** moves the job back to waiting (default check every 30 s).

## 10. Flows - parent and child jobs

Section 8 adds children **at runtime**. When the whole tree is known upfront, use `FlowProducer`: the parent is processed only after every child completes, and it reads their results with `getChildrenValues()`.

```ts
import { FlowProducer } from 'bullmq';
import { producerConnection } from '@acme/jobs-core';

// ops tooling: rebuild reports for a date range as one flow per day
const flows = new FlowProducer({ connection: producerConnection });
await flows.add({
  name: 'nightly-sales-report',
  queueName: 'nightly-sales-report',
  data: { trigger: 'manual', reportDate: '2026-09-20', step: 'wait' },   // children exist, go straight to waiting
  opts: { jobId: 'manual-rebuild-2026-09-20' },
  children: ['DE', 'AT', 'CH'].map((region) => ({
    name: 'report-part', queueName: 'report-parts',
    data: { date: '2026-09-20', region },
    opts: { attempts: 3, failParentOnFailure: true },
  })),
});
```

- `failParentOnFailure: true` fails the parent when a child fails for good; `ignoreDependencyOnFailure` lets the parent continue without it.
- In v6, flow nodes without an explicit `jobId` get UUIDs, and deduplication is not allowed on parent nodes.
- **Stopping a flow**: a parent in `waiting-children` is not active in any worker, so `cancelJob` can't reach it. `jobs-api` sets the cancel flag on the parent and broadcasts cancel for its active children; children check the parent's flag via `parentRunId`, and the parent fails with `cancelled:` when it resumes.

## 11. Progress, logs and live status

```ts
// in a processor
await job.updateProgress({ done: 3, total: 10 });   // number or object
await job.log('supplier alpha: 1203 items');        // appended to the job's log list in Redis

// in jobs-api
const { logs, count } = await queue.getJobLogs(runId, 0, -1, true);    // oldest first

// live events for dashboards (one QueueEvents per queue; uses a Redis stream, survives reconnects)
import { QueueEvents } from 'bullmq';
const events = new QueueEvents('sync-inventory', { connection: workerConnection });
events.on('active',    ({ jobId }) => publish({ type: 'run.updated', jobId: 'sync-inventory', runId: jobId, status: 'running' }));
events.on('progress',  ({ jobId, data }) => publish({ type: 'run.updated', jobId: 'sync-inventory', runId: jobId, progress: data }));
events.on('completed', ({ jobId }) => publish({ type: 'run.updated', jobId: 'sync-inventory', runId: jobId, status: 'succeeded' }));
events.on('failed',    ({ jobId, failedReason }) => publish({
  type: 'run.updated', jobId: 'sync-inventory', runId: jobId,
  status: failedReason.startsWith('cancelled:') ? 'cancelled' : 'failed',
}));
```

`jobs-api` turns these into the `GET /api/jobs/events` SSE stream that the [Job Scheduler Dashboard with Next.js](/Software%20Engineering/Backend/Scheduled%20Jobs/Job%20Scheduler%20Dashboard%20with%20Next.js) consumes. Job logs live in Redis with the job and disappear with `removeOnComplete`; for long-term logs also write JSON to stdout (Loki) with `job` and `run_id` fields.

## 12. Admin UIs

| Tool | What | When |
|---|---|---|
| **Bull Board** (`@bull-board/*`, v9.x supports BullMQ 5 and 6) | open-source UI: queues, job states, data, logs, retry/remove/promote | internal ops UI behind SSO; quick to embed in Fastify/Express |
| **Taskforce.sh** | commercial SaaS/on-prem dashboard from the BullMQ authors; connects to your Redis via a connector | multiple teams/clusters, alerting, no UI to maintain |
| **Your own** (`ops-console`) | domain view: jobs not queues, pause/resume/run/stop with RBAC and audit | when the dashboard must cover BullMQ **and** K8s CronJobs, like Acme |

```ts
// Bull Board mounted inside jobs-api at /admin/queues (protect it: ops:jobs:admin only)
import { createBullBoard } from '@bull-board/api';
import { BullMQAdapter } from '@bull-board/api/bullMQAdapter';
import { FastifyAdapter } from '@bull-board/fastify';

const serverAdapter = new FastifyAdapter();
serverAdapter.setBasePath('/admin/queues');
createBullBoard({ queues: queues.map((q) => new BullMQAdapter(q, { readOnlyMode: true })), serverAdapter });
await app.register(serverAdapter.registerPlugin(), { prefix: '/admin/queues' });
```

Run Bull Board **read-only** when a control plane owns mutations; otherwise people bypass audit and RBAC by clicking "retry" in the side door.

## 13. Metrics

Two sources: BullMQ's exporter for queue state, and prom-client for run outcomes.

```ts
// apps/jobs-worker/src/metrics.ts
import http from 'node:http';
import type { Queue, Worker } from 'bullmq';
import client from 'prom-client';
import type { Logger } from 'pino';
import type { JobDefinition } from '@acme/jobs-core';

const registry = new client.Registry();
client.collectDefaultMetrics({ register: registry });

const runs = new client.Counter({ name: 'jobs_runs_total', help: 'Finished runs', labelNames: ['job', 'status'], registers: [registry] });
const duration = new client.Histogram({
  name: 'jobs_run_duration_seconds', help: 'Run duration', labelNames: ['job'],
  buckets: [1, 5, 15, 30, 60, 120, 300, 600, 1800, 3600], registers: [registry],
});
const lastSuccess = new client.Gauge({ name: 'jobs_last_success_timestamp_seconds', help: 'Last successful run', labelNames: ['job'], registers: [registry] });
const expected = new client.Gauge({ name: 'jobs_expected_interval_seconds', help: 'Expected interval', labelNames: ['job'], registers: [registry] });

export function observeWorker(def: JobDefinition, worker: Worker) {
  expected.set({ job: def.id }, def.expectedIntervalSeconds);
  worker.on('completed', (job, result) => {
    const skipped = (result as { skipped?: string } | undefined)?.skipped;
    runs.inc({ job: def.id, status: skipped ? 'skipped' : 'succeeded' });
    if (!skipped) lastSuccess.set({ job: def.id }, Date.now() / 1000);
    if (job.processedOn && job.finishedOn) duration.observe({ job: def.id }, (job.finishedOn - job.processedOn) / 1000);
  });
  worker.on('failed', (job, err) => {
    if (!job) return;
    const final = job.attemptsMade >= (job.opts.attempts ?? 1) || err.name === 'UnrecoverableError';
    if (final) runs.inc({ job: def.id, status: err.message.startsWith('cancelled:') ? 'cancelled' : 'failed' });
  });
}

export function startMetricsServer(queues: Queue[], log: Logger) {
  const server = http.createServer(async (req, res) => {
    if (req.url !== '/metrics') { res.writeHead(404).end(); return; }
    const bull = (await Promise.all(queues.map((q) => q.exportPrometheusMetrics({ env: 'prod' })))).join('\n');
    res.writeHead(200, { 'content-type': registry.contentType });
    res.end(`${await registry.metrics()}\n${bull}\n`);
  });
  server.listen(9464, () => log.info('metrics on :9464/metrics'));
  return server;
}
```

`jobs_last_success_timestamp_seconds` is per pod; alert on `max by (job)` of it (or push it to Redis and export from one place). `jobs_paused{job}` (1/0) is exported by `jobs-api`, which owns pause state. The alert rules are in [Cron with Message Queues - Architecture](/Software%20Engineering/Backend/Scheduled%20Jobs/Cron%20with%20Message%20Queues%20-%20Architecture#14-observability).

## 14. Testing workers

Unit-test processors as plain functions with a fake `RunContext`; integration-test the queue wiring against a real Redis in a container.

```ts
// apps/jobs-worker/test/scheduler.int.test.ts
import { afterAll, beforeAll, expect, test } from 'vitest';
import { RedisContainer, type StartedRedisContainer } from '@testcontainers/redis';
import { Queue, Worker, QueueEvents } from 'bullmq';
import { Redis } from 'ioredis';
import { reconcileScheduler } from '../src/schedulers.js';
import { withRunContext } from '../src/run-context.js';
import { KEYS, getJob } from '@acme/jobs-core';

let redisC: StartedRedisContainer;
let connection: { host: string; port: number; maxRetriesPerRequest: null };

beforeAll(async () => {
  redisC = await new RedisContainer('redis:8-alpine').start();
  connection = { host: redisC.getHost(), port: redisC.getPort(), maxRetriesPerRequest: null };
}, 60_000);
afterAll(async () => { await redisC.stop(); });

test('reconcile creates one scheduler and respects the paused flag', async () => {
  const def = getJob('sync-inventory')!;
  const queue = new Queue(def.id, { connection });
  const redis = new Redis(connection);

  await reconcileScheduler(queue, def, redis);
  await reconcileScheduler(queue, def, redis);                 // idempotent
  expect(await queue.getJobSchedulersCount()).toBe(1);
  expect((await queue.getJobScheduler(def.id))?.pattern).toBe('*/15 * * * *');

  await redis.sadd(KEYS.paused, def.id);
  await reconcileScheduler(queue, def, redis);
  expect(await queue.getJobScheduler(def.id)).toBeUndefined();
  await Promise.all([queue.close(), redis.quit()]);
});

test('a cancelled run fails with a non-retryable "cancelled:" reason', async () => {
  const def = { ...getJob('sync-inventory')!, id: 'cancel-test' };
  const queue = new Queue(def.id, { connection });
  const events = new QueueEvents(def.id, { connection });
  const redis = new Redis(connection);
  const worker = new Worker(def.id, withRunContext(def, async ({ signal }) => {
    await new Promise((_, reject) => signal.addEventListener('abort', () => reject(signal.reason)));
    return 'unreachable';
  }, redis), { connection });
  await events.waitUntilReady();                               // don't miss the 'active' event

  const job = await queue.add(def.id, { trigger: 'manual' }, { jobId: 'manual-test-1', attempts: 3 });
  await new Promise<void>((r) => events.once('active', () => r()));
  expect(worker.cancelJob(job.id!, 'test')).toBe(true);

  const failed = await new Promise<string>((r) => events.once('failed', ({ failedReason }) => r(failedReason)));
  expect(failed).toBe('cancelled: test');
  expect((await queue.getJob(job.id!))?.attemptsMade).toBe(1);  // UnrecoverableError: no retries
  await Promise.all([worker.close(), events.close(), queue.close(), redis.quit()]);
});
```

Also test: DST behaviour of your patterns (compute the next ticks with `cron-parser` for 2026-03-29 and 2026-10-25 in Europe/Berlin), idempotency (run a processor twice, assert one email and identical rows), and graceful shutdown (send SIGTERM to a child process with an active job, assert it completes).

## 15. Deploying `jobs-worker` on Kubernetes

The base setup (image, probes, JSON logs, Kustomize overlays, Argo CD) is the same as `orders-api` in [Deploying Node.js Apps to Kubernetes](/Software%20Engineering/DevOps/Delivery%20Platform/Deploying%20Node.js%20Apps%20to%20Kubernetes). Worker-specific parts:

```yaml
# apps/jobs-worker/base/deployment.yaml (excerpt)
apiVersion: apps/v1
kind: Deployment
metadata: { name: jobs-worker, namespace: shop-prod, labels: { app: jobs-worker } }
spec:
  replicas: 2
  selector: { matchLabels: { app: jobs-worker } }
  template:
    metadata:
      labels: { app: jobs-worker }
      annotations: { prometheus.io/scrape: "true", prometheus.io/port: "9464" }
    spec:
      terminationGracePeriodSeconds: 60      # > the 50 s drain deadline in main.ts
      containers:
        - name: jobs-worker
          image: ghcr.io/acme/jobs-worker:3f9c2e1
          args: ["--enable-source-maps", "dist/main.js"]
          envFrom: [{ secretRef: { name: jobs-worker-env } }]   # REDIS_*, DATABASE_URL, supplier tokens (ESO)
          ports: [{ name: metrics, containerPort: 9464 }]
          readinessProbe: { httpGet: { path: /metrics, port: metrics }, periodSeconds: 10 }
          resources:
            requests: { cpu: 250m, memory: 384Mi }
            limits: { memory: 768Mi }
```

No Service or HTTPRoute is needed: workers pull from Redis. No `preStop` sleep either (nothing routes traffic to it).

### KEDA: scale on queue depth

There is no BullMQ-specific KEDA scaler. Two options that work with KEDA 2.x (2.21.0 was released 2026-09-23):

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: { name: jobs-worker, namespace: shop-prod }
spec:
  scaleTargetRef: { name: jobs-worker }
  minReplicaCount: 2            # schedulers still need a live worker to process ticks
  maxReplicaCount: 10
  cooldownPeriod: 300
  triggers:
    # Preferred: BullMQ's own exporter (works for the Postgres backend too)
    - type: prometheus
      metadata:
        serverAddress: http://kube-prometheus-stack-prometheus.monitoring:9090
        query: sum(bullmq_job_count{queue="report-parts", state=~"waiting|prioritized"})
        threshold: "50"          # target waiting jobs per replica
    # Alternative: the Redis Lists scaler on BullMQ's wait list (Redis backend internals; may change)
    # - type: redis
    #   metadata: { addressFromEnv: REDIS_ADDRESS, passwordFromEnv: REDIS_PASSWORD,
    #               listName: "bull:report-parts:wait", listLength: "50" }
```

Scaling only helps queues whose global concurrency is above 1. With Forbid (`setGlobalConcurrency(1)`), more pods don't make `sync-inventory` faster; the fan-out queue `report-parts` is what benefits.

## Pitfalls

<Warning>
**Common mistakes**

- Using v5 tutorials on v6: `repeat` options and `getRepeatableJobs()` are gone; v6 throws on legacy repeatable data in Redis.
- `maxmemory-policy allkeys-lru` on the Redis that holds queues (a cache config copied over).
- Keeping finished jobs forever (no `removeOnComplete`): Redis fills up, then `noeviction` makes writes fail.
- Custom job ids with `:` or all digits (`"123"`): rejected or mis-parsed. Use `-`/`_` and a prefix.
- CPU-heavy processors inline: the lock isn't renewed, the job "stalls" and runs twice. Use sandboxed processors.
- `worker.close()` without a deadline, then SIGKILL: jobs stall and re-run after ~30 s.
- Expecting `cancelJob` to stop a job on another pod, or to stop code that never checks the signal.
- Treating `queue.pause()` as "pause the schedule": manual runs stop too, and a catch-up tick fires on resume.
- `every` schedulers for daily work: a new `every` scheduler fires immediately and aligns to the epoch, not local midnight.
- Scaling workers for a queue with global concurrency 1 and wondering why nothing got faster.
</Warning>

## Senior perspective

- **Redis is now a database, not a cache.** Give the queue Redis its own instance (or at least its own DB with `noeviction`), persistence, memory alerts, backups, and a failover story (Sentinel or a managed service). Since v6 you can instead choose the **PostgreSQL backend** if you'd rather not run Redis; it's newer and less battle-tested, so load-test it first.
- **Reconcile, don't mutate.** Schedules come from the registry at deploy time; runtime state (paused) is a separate, audited flag. That keeps Git as the source of truth and makes drift visible.
- **Separate queues per job** give per-job pause, global concurrency, metrics and KEDA triggers. The cost is more Redis connections (each Worker and QueueEvents holds blocking connections); plan connection limits.
- **Cancellation is a contract**: every processor must document where it checks the signal and what partial state it leaves. Prefer steps that are idempotent so a cancelled run can simply be re-run.
- **Know your failure budget**: how long can `sync-inventory` be down before customers see wrong stock? That number sets the alert thresholds, retry counts and whether you need a hot standby Redis.
- Review questions: *What happens on a double tick? Where does this processor check `signal`? What's retained in Redis and for how long? What's the drain deadline versus `terminationGracePeriodSeconds`? Is this Redis shared with a cache?*

<Info>
**🆕 2025–2026**

- **2025-01-25 · BullMQ 5.37.0:** built-in Prometheus exporter, `queue.exportPrometheusMetrics(globalLabels?)`, gauge `bullmq_job_count{queue,state}`.
- **2025-11-20 · BullMQ 5.64.0:** job cancellation with `worker.cancelJob(jobId, reason?)`, `worker.cancelAllJobs(reason?)` and an `AbortSignal` passed as the processor's third argument. **2026-02-21 · 5.70.0:** also for sandboxed processors.
- **2026-03-25:** BullMQ's guide *Properly cancelling jobs* recommends a Redis pub/sub control channel for cancelling from another process.
- **2026-07-30 · BullMQ 6.0.0:** pluggable backends with an optional **PostgreSQL backend**, legacy repeatable jobs removed, `ioredis` an optional peer dependency (install it yourself), Redis internals (`Queue#client`, `Worker#blockingClient`) removed, `debounce` → `deduplication`, `Worker#resume()`/`Queue#resume()` must be awaited, flow ids default to UUIDs. Latest is **6.3.9 (2026-09-25)**.
- **2026-07-31 · ioredis 6.0.0:** adds RESP3; BullMQ's peer range accepts ioredis ≥ 5.
- **2026-09-23 · KEDA 2.21.0** (no dedicated BullMQ scaler; use the Prometheus or Redis Lists scaler).
</Info>

## Interview questions

<Accordion title="How does a BullMQ Job Scheduler avoid duplicate ticks when several processes call `upsertJobScheduler`?">
The scheduler is stored in Redis under its id, and upsert creates or updates it atomically. It keeps a single delayed job whose id is derived from the scheduler id and the next timestamp (`repeat:<id>:<millis>`). A second process computing the same tick produces the same id, so no duplicate is added.
</Accordion>

<Accordion title="What replaced repeatable jobs and what changed in BullMQ 6?">
Job Schedulers (`upsertJobScheduler`, `getJobSchedulers`, `getJobScheduler`, `removeJobScheduler`) since 5.16. BullMQ 6.0.0 removed the legacy `repeat` option and repeatable APIs entirely, removed `repeat.utc` (use `tz: 'UTC'`), made ioredis an optional peer dependency and added pluggable backends including PostgreSQL.
</Accordion>

<Accordion title="`queue.pause()` vs `worker.pause()` vs `removeJobScheduler()`?">
`queue.pause()` stops all workers from taking any job from that queue (scheduled or manual) while producers can still add jobs. `worker.pause()` stops one worker instance only. `removeJobScheduler()` stops future scheduled jobs but leaves the queue working for manual jobs. Active jobs finish in all three cases.
</Accordion>

<Accordion title="How do you cancel a job that's running on another pod?">
Use BullMQ's built-in cancellation: the processor receives an `AbortSignal`, and `worker.cancelJob(id, reason)` aborts it, but only on the worker that holds the job. So the API sets a durable cancel flag and publishes a cancel message on Redis pub/sub; every worker calls `cancelJob`, the owner aborts. The processor throws `UnrecoverableError` so the job fails as cancelled without retries. The flag also covers queued jobs and missed pub/sub messages.
</Accordion>

<Accordion title="Why must Redis use `maxmemory-policy noeviction` for BullMQ?">
Any eviction policy can delete arbitrary keys (job hashes, lists, scheduler entries) under memory pressure, silently corrupting queues. `noeviction` makes writes fail instead, which is visible and alertable. Pair it with AOF persistence and retention limits.
</Accordion>

<Accordion title="What causes stalled jobs and how do you prevent them?">
A worker must renew the job lock periodically; if it crashes or blocks the event loop longer than the lock duration, the stalled checker moves the job back to waiting (or fails it after `maxStalledCount`). Prevent with sandboxed processors for CPU work, graceful shutdown on SIGTERM, and idempotent processors for the cases you can't prevent.
</Accordion>

<Accordion title="How would you autoscale BullMQ workers on Kubernetes?">
KEDA with a Prometheus trigger on `bullmq_job_count{state="waiting"}` from BullMQ's exporter (or the Redis Lists scaler on the queue's wait list). Keep a minimum replica count so scheduled ticks are processed, and remember that global concurrency caps throughput regardless of replicas.
</Accordion>

<Accordion title="How do you model &quot;parent runs after all children finish&quot; in BullMQ?">
Flows: `FlowProducer.add({ ..., children: [...] })` when the tree is known upfront, or add children at runtime with the `parent` option and call `job.moveToWaitingChildren(token)`, throwing `WaitingChildrenError` so the parent sleeps. When children complete, the parent is processed again and reads `getChildrenValues()`.
</Accordion>

## Related
- [Scheduled Jobs Index](/Software%20Engineering/Backend/Scheduled%20Jobs/Scheduled%20Jobs%20Index) · [Backend Index](/Software%20Engineering/Backend/Backend%20Index) · [Node.js Index](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Index)
- [Cron with Message Queues - Architecture](/Software%20Engineering/Backend/Scheduled%20Jobs/Cron%20with%20Message%20Queues%20-%20Architecture) · [Job Control API - Pause, Resume, Run Now & Stop](/Software%20Engineering/Backend/Scheduled%20Jobs/Job%20Control%20API%20-%20Pause%2C%20Resume%2C%20Run%20Now%20%26%20Stop) · [Job Scheduler Dashboard with Next.js](/Software%20Engineering/Backend/Scheduled%20Jobs/Job%20Scheduler%20Dashboard%20with%20Next.js) · [Kubernetes CronJobs in Practice](/Software%20Engineering/Backend/Scheduled%20Jobs/Kubernetes%20CronJobs%20in%20Practice)
- [Background Jobs & Task Queues](/Software%20Engineering/Backend/Background%20Jobs%20%26%20Task%20Queues) · [Cron Jobs](/Software%20Engineering/Backend/Cron%20Jobs) · [Message Queues & Event Streaming](/Software%20Engineering/System%20Design/Message%20Queues%20%26%20Event%20Streaming)
- [Deploying Node.js Apps to Kubernetes](/Software%20Engineering/DevOps/Delivery%20Platform/Deploying%20Node.js%20Apps%20to%20Kubernetes) · [Grafana & the LGTM Stack](/Software%20Engineering/DevOps/Delivery%20Platform/Grafana%20%26%20the%20LGTM%20Stack) · [Kubernetes Workloads in Production](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Workloads%20in%20Production)

## Sources
- [BullMQ docs: Job Schedulers](https://docs.bullmq.io/guide/job-schedulers) · [Repeat options](https://docs.bullmq.io/guide/job-schedulers/repeat-options) · [Manage Job Schedulers](https://docs.bullmq.io/guide/job-schedulers/manage-job-schedulers)
- [BullMQ docs: Migrate from v5 to v6](https://docs.bullmq.io/guide/migrations/migrate-from-v5-to-v6) · [Changelog (v6)](https://docs.bullmq.io/changelog) · [Changelog v5 (GitHub)](https://github.com/taskforcesh/bullmq/blob/master/docs/gitbook/changelogs/changelog-v5.md) · [Releases](https://github.com/taskforcesh/bullmq/releases)
- [BullMQ docs: Cancelling jobs](https://docs.bullmq.io/guide/workers/cancelling-jobs) · [BullMQ blog: Properly cancelling jobs (2026-03-25)](https://bullmq.io/articles/guides/properly-cancelling-jobs/)
- [BullMQ docs: Going to production](https://docs.bullmq.io/guide/going-to-production) · [Graceful shutdown](https://docs.bullmq.io/guide/workers/graceful-shutdown) · [Pausing queues](https://docs.bullmq.io/guide/workers/pausing-queues) · [Sandboxed processors](https://docs.bullmq.io/guide/workers/sandboxed-processors)
- [BullMQ docs: Job ids](https://docs.bullmq.io/guide/jobs/job-ids) · [Deduplication](https://docs.bullmq.io/guide/jobs/deduplication) · [Global concurrency](https://docs.bullmq.io/guide/queues/global-concurrency) · [Rate limiting](https://docs.bullmq.io/guide/rate-limiting) · [Auto-removal](https://docs.bullmq.io/guide/queues/auto-removal-of-jobs)
- [BullMQ docs: Flows](https://docs.bullmq.io/guide/flows) · [Process step jobs / waiting children](https://docs.bullmq.io/patterns/process-step-jobs) · [Events](https://docs.bullmq.io/guide/events) · [Prometheus](https://docs.bullmq.io/guide/metrics/prometheus) · [PostgreSQL backend](https://docs.bullmq.io/guide/postgresql)
- [Bull Board](https://github.com/felixmosh/bull-board) · [Taskforce.sh](https://taskforce.sh/)
- [KEDA: Prometheus scaler](https://keda.sh/docs/latest/scalers/prometheus/) · [Redis Lists scaler](https://keda.sh/docs/latest/scalers/redis-lists/) · [KEDA releases](https://github.com/kedacore/keda/releases)
- [Redis: Eviction policies](https://redis.io/docs/latest/develop/reference/eviction/) · [Persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
- [ioredis releases](https://github.com/redis/ioredis/releases) · [Testcontainers for Node: Redis module](https://node.testcontainers.org/modules/redis/)
