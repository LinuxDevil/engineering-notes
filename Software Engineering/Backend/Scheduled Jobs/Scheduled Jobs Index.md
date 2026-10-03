---
tags: [backend, backend/jobs, index]
aliases: [Scheduled Jobs, Scheduled Jobs Hub, Acme Ops, Job Scheduling Index, Which scheduler should I use]
updated: 2026-09-27
---
# Scheduled Jobs Index

> [!tldr] Start here
> This hub covers everything that runs **on a schedule**: cron syntax, "cron that enqueues" architectures, BullMQ Job Schedulers, Kubernetes CronJobs, a control API to **pause, resume, run now and stop** jobs, a Next.js dashboard on top, and Playwright synthetic checks as a scheduled job. All notes share one running example, **Acme Ops**, which extends the Acme Shop system from [[Kubernetes Delivery Platform - End to End]]. Read it beginner → senior in the order below, or jump to the decision table if you only need to pick a scheduler.

```mermaid
mindmap
  root((Scheduled Jobs))
    Foundations
      Cron syntax & crontab
      Queues, retries, idempotency
    Architecture
      Scheduler enqueues, workers execute
      Leader election & dedupe keys
      Overlap, misfires, DST
      Failure modes & alerting
    Runtimes
      BullMQ Job Schedulers
      Kubernetes CronJobs
      Temporal · EventBridge · RabbitMQ
    Control plane
      Job Control API
      Next.js dashboard
      RBAC · audit · idempotency
    Use cases
      Reports & syncs
      Cleanups
      Playwright synthetics
```

## The running example: Acme Ops

| id | schedule (Europe/Berlin) | backend | what | deep dive |
|---|---|---|---|---|
| `nightly-sales-report` | `0 2 * * *` | BullMQ | builds CSV per region, uploads to S3, emails a link | [[BullMQ Job Schedulers in Practice]] |
| `sync-inventory` | `*/15 * * * *` | BullMQ | pulls supplier APIs, upserts Postgres | [[BullMQ Job Schedulers in Practice]] |
| `cleanup-expired-carts` | `0 * * * *` | Kubernetes CronJob | deletes carts older than 7 days | [[Kubernetes CronJobs in Practice]] |
| `playwright-checkout-smoke` | every 10 min (`*/10 * * * *`) | Kubernetes CronJob (Playwright image) | synthetic check of the checkout flow on prod | [[Playwright Automation as a Cron Job]] |

| Component | What it is | Deep dive |
|---|---|---|
| **Redis** (`shop-prod`) | BullMQ storage: schedulers, queues, job logs (`noeviction`, AOF) | [[BullMQ Job Schedulers in Practice]] |
| **`jobs-worker`** | Node 24 + BullMQ workers, one Deployment, KEDA on queue depth | [[BullMQ Job Schedulers in Practice]] |
| **`jobs-api`** | Fastify control plane: one REST + SSE API over both backends | [[Job Control API - Pause, Resume, Run Now & Stop]] |
| **`ops-console`** | Next.js 16 App Router dashboard at `ops.acme.dev` | [[Job Scheduler Dashboard with Next.js]] |
| **Grafana stack** | metrics (`jobs_runs_total`, `jobs_run_duration_seconds`), Loki logs with `job`/`run_id`, alerts on failures and missed schedules | [[Cron with Message Queues - Architecture]], [[Grafana & the LGTM Stack]] |

```mermaid
flowchart LR
  subgraph Schedulers
    BS["BullMQ Job Schedulers<br/>(in Redis)"]
    KC["K8s CronJob controller"]
  end
  subgraph shop-prod
    R[("Redis<br/>queues")]
    W["jobs-worker<br/>(BullMQ workers, KEDA)"]
    J["K8s Jobs → Pods<br/>cleanup · playwright"]
    API["jobs-api (Fastify)<br/>BullMqBackend · K8sCronJobBackend"]
  end
  UI["ops-console<br/>Next.js 16 · ops.acme.dev"]
  G["Prometheus · Loki · Grafana<br/>alerts"]

  BS -- "delayed job per tick" --> R --> W
  KC -- "Job per tick" --> J
  UI -- "Server Components / Actions<br/>REST + SSE proxy" --> API
  API -- "pause/resume/run/stop<br/>(Redis, pub/sub)" --> R
  API -- "patch suspend · create/delete Job<br/>pod logs" --> K["kube-apiserver"]
  K --- KC & J
  W & J & API -- "metrics · JSON logs" --> G
```

## Reading order (beginner → senior)

| Step | Level | Note | You'll learn |
|---|---|---|---|
| 1 | Beginner | [[Cron Jobs]] | cron expressions, crontab, `flock`, systemd timers, why cron in containers is a trap |
| 2 | Beginner | [[Background Jobs & Task Queues]] | queues, workers, retries with backoff and jitter, idempotency, DLQs |
| 3 | Intermediate | [[Cron with Message Queues - Architecture]] | scheduler → queue → workers; one trigger per tick; overlap, misfires, DST; RabbitMQ, SQS + EventBridge, Temporal; failure modes and alerting |
| 4 | Intermediate | [[Kubernetes CronJobs in Practice]] | CronJob and Job fields, `concurrencyPolicy`, `startingDeadlineSeconds`, suspend, run now, stop, alerts |
| 5 | Intermediate | [[BullMQ Job Schedulers in Practice]] | `upsertJobScheduler`, workers, cancellation, graceful shutdown, flows, metrics, KEDA |
| 6 | Intermediate | [[Playwright Automation as a Cron Job]] | synthetic checks as a CronJob: image, signals, reports, alerting, safety on prod |
| 7 | Senior | [[Job Control API - Pause, Resume, Run Now & Stop]] | a backend-neutral control plane: state machine, adapters, RBAC, idempotency keys, RFC 9457, audit, SSE, K8s RBAC |
| 8 | Senior | [[Job Scheduler Dashboard with Next.js]] | the ops UI: Server Components, Server Actions, SSE proxy, role-based buttons, accessibility |

> [!example]- 🧭 Suggested study path
> **Week 1 (foundations):** [[Cron Jobs]] → [[Background Jobs & Task Queues]] → [[Message Queues & Event Streaming]]. Exercise: write a crontab line with `flock`, then rewrite it as a BullMQ scheduler that enqueues.
> **Week 2 (architecture):** [[Cron with Message Queues - Architecture]]. Exercise: fill in the failure-modes table for a job you own; add a missed-schedule alert.
> **Week 3 (runtimes):** [[Kubernetes CronJobs in Practice]] and [[BullMQ Job Schedulers in Practice]] side by side. Exercise: on `kind`, pause/resume a CronJob with and without `startingDeadlineSeconds` and observe catch-up; cancel a BullMQ job from another process.
> **Week 4 (control plane):** [[Job Control API - Pause, Resume, Run Now & Stop]] → [[Job Scheduler Dashboard with Next.js]] → [[Playwright Automation as a Cron Job]]. Exercise: implement `stopRun` for one backend with a test, and check your K8s Role with `kubectl auth can-i`.
> **Interview prep:** the interview questions at the end of each note, plus the decision table below.

## Which scheduler should I use?

| Scheduler | Runs where | State & HA | Time zones | Retries / overlap | Pause / run now / stop | Best for | Watch out for |
|---|---|---|---|---|---|---|---|
| **crontab** (+ `flock`) | one Linux host | none; host dies = no runs | `CRON_TZ` on some crons, host zone otherwise | none; `flock -n` for overlap | comment out / run by hand / `kill` | single VM, small scripts | duplicates on every host, silent failures, env/PATH issues ([[Cron Jobs]]) |
| **Kubernetes CronJob** | a Pod per run | controller-manager (HA); history in Job objects | `spec.timeZone` (GA 1.27) | `backoffLimit`, `podFailurePolicy`; `concurrencyPolicy` Allow/Forbid/Replace | `suspend` / `kubectl create job --from` / delete Job | containerised batch, isolated resources per run | min 1-minute granularity, pod start latency, 100-missed-schedules rule, history limits ([[Kubernetes CronJobs in Practice]]) |
| **BullMQ Job Schedulers** | Node workers | Redis (or PostgreSQL backend since v6) | `tz` per scheduler | `attempts` + backoff; global concurrency; `cancelJob` + `AbortSignal` | remove/upsert scheduler / `queue.add` / `cancelJob` | Node shops already on Redis; high-frequency jobs; fan-out | Redis must be `noeviction` + persistent; cooperative cancellation only ([[BullMQ Job Schedulers in Practice]]) |
| **RabbitMQ + a scheduler** | any consumer | quorum queues (replicated); the scheduler is separate (CronJob, leader-elected service) | the scheduler's | delivery limit + DLX; TTL + DLX for delays; 4.3+ quorum-queue delayed retry | depends on the scheduler | polyglot consumers, existing RabbitMQ | RabbitMQ has no cron; the delayed-message plugin is unmaintained ([[Cron with Message Queues - Architecture#11. The RabbitMQ variant]]) |
| **AWS EventBridge Scheduler + SQS** | Lambda, ECS, any SQS consumer | fully managed | IANA zone per schedule; documented DST behaviour | target retry policy + DLQ; SQS redrive (`maxReceiveCount`) | `DISABLED` state / one-time `at()` schedule / consumer-side | AWS-native, many schedules, zero ops | 60 s precision, at-least-once (dedupe on `<aws.scheduler.scheduled-time>`), no run history UI |
| **Temporal Schedules** | Temporal workers | Temporal cluster/Cloud (durable history) | `timezone` in the spec | activity retry policies; overlap `SKIP`/`BUFFER_ONE`/`CANCEL_OTHER`/... ; catch-up window | `pause`/`unpause` / `trigger` / cancel workflow | multi-step, long-running business processes with waits | operating Temporal; determinism rules in workflow code |
| **pg_cron** | inside Postgres | the database (primary only) | `cron.timezone` setting (default GMT) | none built in; overlap per job is prevented | `cron.unschedule` / run the SQL / `pg_cancel_backend` | SQL maintenance: partitions, `VACUUM`, rollups, cleanups | runs as DB work on the primary; not for HTTP calls or app logic |
| **pg-boss** | Node workers | Postgres (`SKIP LOCKED`), transactional enqueue | `tz` option on `schedule()` | retries, backoff, dead letter, singleton keys | `unschedule` / `send` / cancel job | Node apps already on Postgres that don't want Redis | load lands on the primary DB; throughput below Redis queues |
| **Quartz** (Java) | in the JVM | JDBC JobStore for clustering | per trigger | misfire instructions, `@DisallowConcurrentExecution` | `pauseJob` / `triggerJob` / `interrupt` | Spring/Java monoliths with rich calendars | 6–7 field cron with seconds; cluster needs a DB and synced clocks ([[Java & Spring Index]]) |
| **GitHub Actions `schedule`** | GitHub runners | GitHub | UTC by default; `timezone` since 2026-03 | workflow-level retries only; `concurrency` groups | disable workflow / `workflow_dispatch` / cancel run | repo chores: dependency bumps, stale issues, nightly builds | runs can be delayed or dropped under load, 5-minute minimum, disabled after 60 days of repo inactivity (public repos) |
| **Jenkins cron triggers** | Jenkins agents | the Jenkins controller | `TZ=Europe/Berlin` line in the trigger | pipeline `retry`, `disableConcurrentBuilds()` | disable job / "Build now" / abort | CI-centric teams already on Jenkins | `H` hashing makes exact minutes unpredictable (by design); controller is a SPOF ([[Jenkins - Pipelines as Code]]) |

```mermaid
flowchart TD
  Q1{"Is it a multi-step business process<br/>with waits, compensation, history?"} -- yes --> T["Temporal Schedules<br/>(or Step Functions on AWS)"]
  Q1 -- no --> Q2{"Is it pure SQL inside one database?"}
  Q2 -- yes --> PC["pg_cron"]
  Q2 -- no --> Q3{"Where does your platform live?"}
  Q3 -- "AWS serverless" --> EB["EventBridge Scheduler + SQS"]
  Q3 -- "Kubernetes" --> Q4{"Needs its own image/resources per run,<br/>minutes-level frequency?"}
  Q4 -- yes --> CJ["Kubernetes CronJob"]
  Q4 -- "no: app code, frequent or fan-out" --> Q5{"Queue store you already run?"}
  Q5 -- Redis --> BM["BullMQ Job Schedulers"]
  Q5 -- Postgres --> PB["pg-boss / BullMQ Postgres backend"]
  Q5 -- RabbitMQ --> RMQ["RabbitMQ + CronJob publisher"]
  Q3 -- "one VM" --> CT["crontab + flock"]
  Q3 -- "repo chores / CI" --> GH["GitHub Actions schedule / Jenkins cron"]
```

Rule of thumb: **use the scheduler that lives next to the state you already operate**, make every run idempotent on `(job, scheduledTime)`, and put a control plane in front only when you have more than one runtime (Acme's reason for `jobs-api`).

## Senior perspective

- The hard parts aren't cron syntax: they are **duplicate and missed runs**, **overlap**, **DST**, **cancellation**, and **knowing when a job silently stopped**. Every note ends with those.
- Standardise the contract (states, run statuses, labels, metrics names, log fields) across runtimes; the runtime is then a replaceable detail.
- Treat schedules as code (Git, review, reconcile at deploy) and runtime interventions (pause, run now, stop) as audited operations.
- Count the cost of each extra scheduler you adopt: another thing to monitor, secure, upgrade and explain to on-call.

> [!info]+ 🆕 2025–2026 in this area
> - **2025-01-25 · BullMQ 5.37.0:** built-in Prometheus exporter. → [[BullMQ Job Schedulers in Practice]]
> - **2025-04-23 · Kubernetes 1.33:** Job backoff limit per index and Job success policy GA. → [[Kubernetes CronJobs in Practice]]
> - **2025-08-27 · Kubernetes 1.34:** Pod replacement policy for Jobs GA. → [[Kubernetes CronJobs in Practice]]
> - **2025-09-09 / 2025-09-20 · TypeBox 1.0 / `@fastify/type-provider-typebox` 6.0:** the `typebox` package replaces `@sinclair/typebox` in Fastify schemas. → [[Job Control API - Pause, Resume, Run Now & Stop]]
> - **2025-09-22 · Auth.js joined Better Auth** and entered maintenance mode. → [[Job Scheduler Dashboard with Next.js]]
> - **2025-10-03 · `@kubernetes/client-node` 1.4.0** (K8s 1.34 APIs); **2026-08-12 · 2.0.0** (undici, drops Node 18/20/23, K8s 1.36 APIs, same object-parameter style). → [[Job Control API - Pause, Resume, Run Now & Stop]]
> - **2025-10-21 · Next.js 16** (`proxy.ts`, async request APIs, opt-in caching). → [[Job Scheduler Dashboard with Next.js]]
> - **2025-11-20 · BullMQ 5.64.0:** built-in job cancellation (`worker.cancelJob`, `AbortSignal`); **2026-02-21 · 5.70.0** for sandboxed processors. → [[BullMQ Job Schedulers in Practice]]
> - **2025-11-25 · Playwright 1.57:** Chrome for Testing builds; **2026-09-04 · Playwright 1.63** current. → [[Playwright Automation as a Cron Job]]
> - **2025-12-17 · Kubernetes 1.35:** Job `managedBy` GA. → [[Kubernetes CronJobs in Practice]]
> - **2026-01-21 · Kubernetes Dashboard archived;** Headlamp recommended. → [[Kubernetes CronJobs in Practice]], [[Job Scheduler Dashboard with Next.js]]
> - **2026-03-19 · GitHub Actions:** `timezone` for `schedule` triggers. → [[Cron with Message Queues - Architecture]], [[Playwright Automation as a Cron Job]]
> - **2026-04-23 · RabbitMQ 4.3.0:** Mnesia removed; quorum-queue delayed retry. **2026-09:** the delayed-message exchange plugin is no longer maintained (repo archived); native delays in quorum queues announced for 4.4. → [[Cron with Message Queues - Architecture]]
> - **2026-07-30 · BullMQ 6.0.0:** pluggable backends (PostgreSQL), repeatable jobs removed, ioredis optional peer. Latest 6.3.9 (2026-09-25). → [[BullMQ Job Schedulers in Practice]]
> - **2026-09-23 · KEDA 2.21.0.** → [[BullMQ Job Schedulers in Practice]]

## Interview questions

> [!question]- How would you design scheduled jobs for a service running 10 replicas?
> Don't run the scheduler in every replica. Use a single trigger (queue-native scheduler like BullMQ Job Schedulers, a K8s CronJob, EventBridge Scheduler, or a leader-elected scheduler) that enqueues one message per tick with an idempotency key `(job, scheduledTime)`. Workers process it with retries, timeouts and a concurrency limit; alert on failures and on missing successes.

> [!question]- How do you pick between a Kubernetes CronJob and an in-app queue scheduler?
> CronJob when each run needs its own image, resources and isolation, and minute-level frequency is fine. A queue scheduler (BullMQ, pg-boss, Solid Queue) when the job is app code, runs frequently or fans out into many small jobs, and you want retries, progress and cancellation inside the app. Many systems use both, which is why a shared control API helps.

> [!question]- What does a good "run now" button need?
> An idempotency key per click so retries don't start two runs, respect for the overlap policy, an audit record (who, when, why), RBAC, and a run id the UI can follow live. On K8s it creates a Job from the CronJob's template; on a queue it adds a job whose id is derived from the key.

> [!question]- Name three ways scheduled jobs fail silently and how you'd catch them.
> The scheduler never fires (deleted, suspended, controller down): alert when the last success is older than 2× the interval. The job "succeeds" without doing work: assert on outcomes (rows, file size, freshness). DST or time zone mistakes skip a run: business-date keys plus the same missed-success alert.

## Related
- [[Backend Index]] · [[Software Engineering]]
- [[Cron Jobs]] · [[Background Jobs & Task Queues]] · [[Message Queues & Event Streaming]] · [[Distributed Systems Patterns]]
- [[Cron with Message Queues - Architecture]] · [[BullMQ Job Schedulers in Practice]] · [[Kubernetes CronJobs in Practice]] · [[Playwright Automation as a Cron Job]] · [[Job Control API - Pause, Resume, Run Now & Stop]] · [[Job Scheduler Dashboard with Next.js]]
- Platform: [[Kubernetes Delivery Platform - End to End]] · [[Deploying Node.js Apps to Kubernetes]] · [[Grafana & the LGTM Stack]] · [[Jenkins - Pipelines as Code]]
- Testing: [[Frontend Testing]]

## Sources
- [crontab(5)](https://man7.org/linux/man-pages/man5/crontab.5.html)
- [Kubernetes: CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
- [BullMQ docs: Job Schedulers](https://docs.bullmq.io/guide/job-schedulers) · [BullMQ changelog](https://docs.bullmq.io/changelog)
- [RabbitMQ: Quorum queues](https://www.rabbitmq.com/docs/quorum-queues) · [rabbitmq-delayed-message-exchange](https://github.com/rabbitmq/rabbitmq-delayed-message-exchange)
- [AWS: EventBridge Scheduler](https://docs.aws.amazon.com/scheduler/latest/UserGuide/what-is-scheduler.html)
- [Temporal: Schedules](https://docs.temporal.io/schedule)
- [pg_cron (Citus)](https://github.com/citusdata/pg_cron) · [pg-boss](https://github.com/timgit/pg-boss)
- [Quartz Scheduler](https://www.quartz-scheduler.org/documentation/)
- [GitHub Docs: `on.schedule`](https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows#schedule) · [GitHub Changelog 2026-03-19: timezone for schedules](https://github.blog/changelog/2026-03-19-github-actions-late-march-2026-updates/)
- [Jenkins: Pipeline syntax, triggers (cron, H)](https://www.jenkins.io/doc/book/pipeline/syntax/#triggers)
- [KEDA releases](https://github.com/kedacore/keda/releases)
