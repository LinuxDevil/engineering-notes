---
title: "Kubernetes CronJobs in Practice"
tags: [backend, backend/jobs, devops/kubernetes]
aliases: [K8s CronJob, Kubernetes CronJob, CronJob spec, kubectl create job --from, startingDeadlineSeconds, concurrencyPolicy, podFailurePolicy]
updated: 2026-09-27
---
# Kubernetes CronJobs in Practice

<Note>
**TL;DR**

A **CronJob** creates a **Job** on a cron schedule, and the Job runs **Pods** to completion. The CronJob fields decide *when* and *whether* a run starts (`schedule`, `timeZone`, `concurrencyPolicy`, `startingDeadlineSeconds`, `suspend`, history limits); the Job template decides *how* a run behaves (`backoffLimit`, `activeDeadlineSeconds`, `ttlSecondsAfterFinished`, `podFailurePolicy`, `restartPolicy`). Operations map cleanly: **pause** = `spec.suspend: true`, **resume** = `false` (set `startingDeadlineSeconds` or a missed run fires immediately), **run now** = `kubectl create job --from=cronjob/...`, **stop** = delete the Job with foreground propagation. Kubernetes promises neither exactly one Job per tick nor any Job at all, so jobs must be **idempotent**, and you alert on both failures and missing successes with kube-state-metrics.
</Note>

This note is the Kubernetes backend of Acme Ops. Cron syntax and the general idea are in [Cron Jobs](/Software%20Engineering/Backend/Cron%20Jobs); how the `jobs-api` control plane hides BullMQ vs Kubernetes behind one interface is in [Cron with Message Queues - Architecture](/Software%20Engineering/Backend/Scheduled%20Jobs/Cron%20with%20Message%20Queues%20-%20Architecture) and [Job Control API - Pause, Resume, Run Now & Stop](/Software%20Engineering/Backend/Scheduled%20Jobs/Job%20Control%20API%20-%20Pause%2C%20Resume%2C%20Run%20Now%20%26%20Stop); the UI is [Job Scheduler Dashboard with Next.js](/Software%20Engineering/Backend/Scheduled%20Jobs/Job%20Scheduler%20Dashboard%20with%20Next.js). Day-2 kubectl skills are in [Kubernetes Operations - Logs, Debugging & Scheduling](/Software%20Engineering/DevOps/Delivery%20Platform/Kubernetes%20Operations%20-%20Logs%2C%20Debugging%20%26%20Scheduling).

## 1. How it works

{/* diagram source: /diagrams/diagram-066.mmd */}
![diagram](/diagrams/diagram-066.svg)

- The **CronJob controller** (in kube-controller-manager) wakes up, computes the most recent scheduled time that hasn't been handled, and creates a Job named `<cronjob>-<scheduled time in minutes since epoch>`. The deterministic name means the same tick can't create two Jobs with the same name, but it is not a full exactly-once guarantee (see pitfalls).
- It stamps the Job with the annotation `batch.kubernetes.io/cronjob-scheduled-timestamp` (RFC 3339) and an owner reference to the CronJob.
- The **Job controller** creates Pods until the success criteria are met or the failure limits are hit.
- One CronJob is like one crontab line. Because of the name suffix (11 characters), the CronJob name should be at most **52 characters**.

{/* diagram source: /diagrams/diagram-067.mmd */}
![diagram](/diagrams/diagram-067.svg)

## 2. The full spec, field by field

### CronJob level: when and whether

| Field | Default | What it does | Acme choice |
|---|---|---|---|
| `schedule` | required | 5-field cron (`@hourly`, `@daily`... also allowed; `?` and `H` are not) | `0 * * * *` |
| `timeZone` | controller's local zone | IANA name, e.g. `Europe/Berlin`. **Stable since v1.27** (beta 1.25–1.26). `CRON_TZ=`/`TZ=` inside `schedule` is not supported and is rejected | `Europe/Berlin` |
| `concurrencyPolicy` | `Allow` | `Allow`: runs may overlap. `Forbid`: skip the new run if one is active (counts as missed). `Replace`: delete the active run, start the new one | `Forbid` |
| `startingDeadlineSeconds` | none | If the Job can't start within N s of its scheduled time, skip it. Also bounds the "missed schedules" window (below) | `300` |
| `suspend` | `false` | `true`: no new Jobs; running Jobs are untouched. Ticks during suspension count as missed | toggled by `jobs-api` |
| `successfulJobsHistoryLimit` | `3` | Finished successful Jobs to keep (0 = none) | `3` |
| `failedJobsHistoryLimit` | `1` | Failed Jobs to keep | `3` |
| `jobTemplate` | required | Metadata + spec of the Jobs it creates | below |

**The 100 missed schedules rule.** On each check the controller counts how many schedules were missed since the last scheduled time. If `startingDeadlineSeconds` is set, it only counts within that window. If more than **100** were missed, it doesn't start the Job and records "too many missed start times... Set or decrease `.spec.startingDeadlineSeconds` or check clock skew". The docs' example: a per-minute CronJob without a deadline whose controller was down from 08:29 to 10:21 does not start; with `startingDeadlineSeconds: 200` it starts at 10:22, because only about 3 schedules fall inside the last 200 s. The fix is always the same: set a `startingDeadlineSeconds` smaller than 100 × interval (and larger than 10 s, because the controller only checks every ~10 s).

**Resume behaviour.** When `suspend` flips from `true` to `false` on a CronJob **without** a starting deadline, the missed run is scheduled **immediately** (one Job for the most recent missed time, not one per missed tick). With `startingDeadlineSeconds: 300`, a run missed more than 5 minutes ago is skipped. That is how Acme Ops gets "resume = no backfill by default".

### Job level (`jobTemplate.spec`): how a run behaves

| Field | Default | What it does |
|---|---|---|
| `backoffLimit` | `6` | Pod failures before the Job is marked Failed. Retries back off exponentially (10 s, 20 s, 40 s... capped at 6 min) |
| `activeDeadlineSeconds` | none | Wall-clock limit for the whole Job, retries included. Wins over `backoffLimit`: when hit, running Pods are terminated and the Job fails with reason `DeadlineExceeded` |
| `ttlSecondsAfterFinished` | none | Delete the finished Job (and its Pods) N s after it finishes. **Stable since v1.23** |
| `podFailurePolicy` | none | Rules on exit codes or Pod conditions: `FailJob`, `Ignore` (don't count against `backoffLimit`), `Count`, `FailIndex`. **Stable since v1.31**. Requires `restartPolicy: Never` |
| `template.spec.restartPolicy` | must be set | `Never` (a failed container → a new Pod; logs of each attempt kept) or `OnFailure` (kubelet restarts the container in the same Pod; fewer Pods, attempts harder to inspect) |
| `completions` / `parallelism` / `completionMode` | 1 / 1 / `NonIndexed` | Parallel and indexed Jobs (sharding work) |
| `podReplacementPolicy` | `TerminatingOrFailed` | `Failed`: create a replacement only after the old Pod is fully gone. **Stable since v1.34** |
| `suspend` (on the Job) | `false` | Suspending a running Job terminates its active Pods but keeps the Job object |

`ttlSecondsAfterFinished` and the CronJob history limits both delete Jobs. If the TTL is shorter than your history window, the history limits never matter and `kubectl get jobs` shows nothing after a run. Pick one owner: history limits for "keep the last N runs of a CronJob", TTL for standalone Jobs (migrations, one-off scripts). Jobs created with `kubectl create job --from=cronjob/...` are owned by the CronJob, so its history limits clean them up too.

### Newer Job features (2024–2026)

| Feature | Field | Stable since | Use it for |
|---|---|---|---|
| Pod failure policy | `spec.podFailurePolicy` | v1.31 | don't retry config errors (exit 42 → `FailJob`); don't count node drains/preemption (`DisruptionTarget` → `Ignore`) |
| Backoff limit per index | `spec.backoffLimitPerIndex`, `spec.maxFailedIndexes` | **v1.33** | indexed Jobs where each shard retries independently; one bad shard doesn't fail the whole Job |
| Success policy | `spec.successPolicy` | **v1.33** | indexed Jobs that are done when specific indexes (e.g. a leader) or N indexes succeed; remaining Pods are stopped |
| Pod replacement policy | `spec.podReplacementPolicy` | **v1.34** | avoid two Pods for the same work while the old one is still terminating (important for jobs that must never overlap) |
| Managed-by | `spec.managedBy` | **v1.35** | hand reconciliation of a Job to an external controller (e.g. Kueue MultiKueue); the built-in Job controller ignores it. Immutable; the built-in value is `kubernetes.io/job-controller` |

```yaml
# Indexed sharding with per-index retries and early success
spec:
  completionMode: Indexed
  completions: 8                # 8 shards of the carts table (JOB_COMPLETION_INDEX env var = 0..7)
  parallelism: 4
  backoffLimitPerIndex: 2       # each shard may fail twice
  maxFailedIndexes: 1           # more than one shard failing -> the Job fails
  # successPolicy:              # alternative: done when shard 0 (e.g. a coordinator) succeeds
  #   rules: [{ succeededIndexes: "0" }]
  podReplacementPolicy: Failed  # no replacement until the old Pod has fully terminated
```

## 3. Worked example: `cleanup-expired-carts`

Deletes carts not updated for 7 days, every hour, in batches, safe to run twice.

```yaml
# platform-gitops/apps/jobs/base/cleanup-expired-carts.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-expired-carts
  namespace: shop-prod
  labels:
    app.kubernetes.io/name: cleanup-expired-carts
    app.kubernetes.io/part-of: acme-ops
    acme.dev/managed-by: jobs-api          # jobs-api discovers CronJobs by this label
  annotations:
    acme.dev/owner: team-checkout
    acme.dev/description: "Delete carts not updated for 7 days"
    acme.dev/runbook: https://runbooks.acme.dev/jobs/cleanup-expired-carts
spec:
  schedule: "0 * * * *"
  timeZone: Europe/Berlin
  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 300             # resume/controller outage: don't fire stale runs
  suspend: false                           # runtime-owned, see GitOps section
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    metadata:
      labels:
        app.kubernetes.io/name: cleanup-expired-carts   # copied onto every Job: easy selectors
        acme.dev/job-id: cleanup-expired-carts
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 1200          # 20 min hard stop, retries included (< 1 h interval)
      podFailurePolicy:
        rules:
          - action: FailJob                # bad config: retrying won't help
            onExitCodes: { containerName: cleanup, operator: In, values: [42] }
          - action: Ignore                 # node drain, preemption, eviction: retry for free
            onPodConditions: [{ type: DisruptionTarget }]
      template:
        metadata:
          labels:
            app.kubernetes.io/name: cleanup-expired-carts
        spec:
          restartPolicy: Never             # required by podFailurePolicy
          serviceAccountName: cleanup-expired-carts
          automountServiceAccountToken: false
          priorityClassName: batch-low
          terminationGracePeriodSeconds: 60 # time to finish the current batch after SIGTERM
          securityContext: { runAsNonRoot: true, runAsUser: 10001, seccompProfile: { type: RuntimeDefault } }
          containers:
            - name: cleanup
              image: ghcr.io/acme/orders-api:3f9c2e1           # same image, different entrypoint
              command: ["node", "dist/jobs/cleanup-expired-carts.js"]
              env:
                - name: RUN_ID                                  # Job name = cronjob + scheduled minute
                  valueFrom: { fieldRef: { fieldPath: "metadata.labels['batch.kubernetes.io/job-name']" } }
                - name: CART_TTL_DAYS
                  value: "7"
              envFrom: [{ secretRef: { name: orders-api-db } }] # ExternalSecret -> DATABASE_URL
              resources:
                requests: { cpu: 100m, memory: 128Mi }
                limits: { memory: 256Mi }
              securityContext: { allowPrivilegeEscalation: false, readOnlyRootFilesystem: true, capabilities: { drop: [ALL] } }
```

The job itself: batched, idempotent, stops cleanly on SIGTERM, and uses exit codes the `podFailurePolicy` understands.

```ts
// src/jobs/cleanup-expired-carts.ts
import pg from "pg";
import pino from "pino";

const log = pino({ base: { job: "cleanup-expired-carts", run_id: process.env.RUN_ID } }); // Loki fields
const ttlDays = Number(process.env.CART_TTL_DAYS ?? 7);
if (!process.env.DATABASE_URL || !Number.isFinite(ttlDays)) { log.error("bad config"); process.exit(42); } // -> FailJob

const stop = new AbortController();
process.on("SIGTERM", () => { log.warn("SIGTERM: finishing current batch"); stop.abort(); });

const pool = new pg.Pool({ connectionString: process.env.DATABASE_URL, max: 2 });
let total = 0;
try {
  while (!stop.signal.aborted) {
    // Idempotent: re-running deletes nothing twice; SKIP LOCKED tolerates an accidental overlap.
    const { rowCount } = await pool.query(
      `DELETE FROM carts WHERE id IN (
         SELECT id FROM carts
         WHERE updated_at < now() - make_interval(days => $1)
         ORDER BY updated_at LIMIT 1000
         FOR UPDATE SKIP LOCKED)`,
      [ttlDays],
    );
    total += rowCount ?? 0;
    if (!rowCount) break;
  }
  log.info({ deleted: total, interrupted: stop.signal.aborted }, "done");
  process.exitCode = stop.signal.aborted ? 143 : 0;   // stopped early: counts as a failed attempt
} catch (err) {
  log.error({ err }, "cleanup failed");
  process.exitCode = 1;                               // retryable: counts against backoffLimit
} finally {
  await pool.end();
}
```

RBAC for `jobs-api`, which drives these objects on behalf of the dashboard:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: jobs-api-cronjobs, namespace: shop-prod }
rules:
  - apiGroups: [batch]
    resources: [cronjobs]
    verbs: [get, list, watch, patch]           # patch = suspend/resume
  - apiGroups: [batch]
    resources: [jobs]
    verbs: [get, list, watch, create, delete]  # run now / stop
  - apiGroups: [""]
    resources: [pods, pods/log]
    verbs: [get, list, watch]                  # status and log streaming
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: jobs-api-cronjobs, namespace: shop-prod }
subjects: [{ kind: ServiceAccount, name: jobs-api, namespace: shop-prod }]
roleRef: { apiGroup: rbac.authorization.k8s.io, kind: Role, name: jobs-api-cronjobs }
```

## 4. Operations with kubectl (and the API calls behind them)

Set a default namespace for the session: `kubectl config set-context --current --namespace=shop-prod`.

### Inspect

```bash
kubectl get cronjobs                                   # SCHEDULE, TIMEZONE, SUSPEND, ACTIVE, LAST SCHEDULE
kubectl get cronjob cleanup-expired-carts -o wide
kubectl describe cronjob cleanup-expired-carts         # events: SuccessfulCreate, SawCompletedJob, missed...
kubectl events --for cronjob/cleanup-expired-carts

# Runs (Jobs) newest last, via the label copied from jobTemplate.metadata
kubectl get jobs -l app.kubernetes.io/name=cleanup-expired-carts --sort-by=.metadata.creationTimestamp
kubectl get jobs -l app.kubernetes.io/name=cleanup-expired-carts -w          # watch runs come and go
kubectl get pods -l app.kubernetes.io/name=cleanup-expired-carts -w          # watch attempts

# Logs of the last run
LAST=$(kubectl get jobs -l app.kubernetes.io/name=cleanup-expired-carts \
  --sort-by=.metadata.creationTimestamp -o name | tail -n 1)
kubectl logs "$LAST" --all-containers --timestamps
kubectl logs "$LAST" -f                                # follow a running one
kubectl get pods -l batch.kubernetes.io/job-name=${LAST#job.batch/} # all attempts of that run
```

Older runs are gone once history limits or the TTL delete them; that's why logs go to Loki with `job` and `run_id` fields ([Grafana & the LGTM Stack](/Software%20Engineering/DevOps/Delivery%20Platform/Grafana%20%26%20the%20LGTM%20Stack)):

```logql
{namespace="shop-prod", app_kubernetes_io_name="cleanup-expired-carts"} | json | run_id="cleanup-expired-carts-29315040"
```

### Pause and resume

```bash
kubectl patch cronjob cleanup-expired-carts --type=merge -p '{"spec":{"suspend":true}}'    # pause
kubectl patch cronjob cleanup-expired-carts --type=merge -p '{"spec":{"suspend":false}}'   # resume
kubectl get cronjob cleanup-expired-carts -o jsonpath='{.spec.suspend}{"\n"}'
```

API equivalent (what `K8sCronJobBackend.pause()` sends):

```http
PATCH /apis/batch/v1/namespaces/shop-prod/cronjobs/cleanup-expired-carts
Content-Type: application/merge-patch+json

{"spec":{"suspend":true}}
```

Pausing never touches a running Job. Resuming fires the most recent missed tick immediately **unless** `startingDeadlineSeconds` excludes it.

### Run now

```bash
kubectl create job --from=cronjob/cleanup-expired-carts cleanup-expired-carts-manual-$(date +%s)
```

`kubectl` copies the CronJob's `jobTemplate` (labels, annotations, spec), adds the annotation `cronjob.kubernetes.io/instantiate: manual`, and sets an owner reference to the CronJob. Things to know:
- `concurrencyPolicy` is **not checked** for manual Jobs: the controller evaluates it only when *it* creates a Job, and `kubectl` doesn't look. `jobs-api` must check for an active Job itself before creating one (and return `409`).
- The name must be unique; `jobs-api` derives it from the `Idempotency-Key` (e.g. `cleanup-expired-carts-m-<first 8 hex of sha256(key)>`), so a retried request gets `409 AlreadyExists` from the API server instead of a second run. That is idempotency for free.
- A manual Job counts toward the CronJob's history limits because it's owned by the CronJob.

API equivalent: `POST /apis/batch/v1/namespaces/shop-prod/jobs` with `metadata` (name, labels, the `instantiate: manual` annotation, `ownerReferences` pointing at the CronJob's UID) and `spec` copied from `cronjob.spec.jobTemplate.spec`.

### Stop a running run

```bash
# Foreground: the Job object stays (with deletionTimestamp) until its Pods are gone
kubectl delete job cleanup-expired-carts-29315040 --cascade=foreground
kubectl get pods -l batch.kubernetes.io/job-name=cleanup-expired-carts-29315040 -w   # Terminating -> gone
```

What happens: the Pods get **SIGTERM**, have `terminationGracePeriodSeconds` (60 s here) to finish the current batch and exit, then SIGKILL. The job code above treats SIGTERM as "finish the batch, then stop".

API equivalent:

```http
DELETE /apis/batch/v1/namespaces/shop-prod/jobs/cleanup-expired-carts-29315040
Content-Type: application/json

{"propagationPolicy":"Foreground"}
```

Alternatives and trade-offs:

| Way to stop | Effect | Keeps the Job record? |
|---|---|---|
| `delete --cascade=foreground` | Pods terminated, then the Job disappears | no (record it in `jobs-api` / Loki first) |
| `delete` (background, default) | Job disappears now, Pods cleaned up after | no |
| `patch job ... '{"spec":{"suspend":true}}'` | Active Pods terminated; Job stays suspended (can be resumed or deleted later) | yes |
| `concurrencyPolicy: Replace` | the next tick kills the running Job | no |

### Wait in scripts and CI

```bash
kubectl wait --for=condition=complete job/cleanup-expired-carts-manual-1727440000 --timeout=20m \
  || kubectl wait --for=condition=failed job/cleanup-expired-carts-manual-1727440000 --timeout=1s
```

## 5. GitOps and runtime changes

In Acme Shop, Argo CD keeps `shop-prod` equal to `platform-gitops` with `selfHeal: true` ([Argo CD in Practice](/Software%20Engineering/DevOps/Delivery%20Platform/Argo%20CD%20in%20Practice)). Toggling `suspend` from `jobs-api` or `kubectl patch` makes the live CronJob differ from Git:
- Argo CD shows the app **OutOfSync**, and with self-heal it **resumes the job within minutes** by re-applying `suspend: false`. Operators think it's paused; it isn't.

Two workable designs:

**A. Runtime owns `suspend`** (what Acme Ops does):

```yaml
# In the Argo CD Application for the jobs
spec:
  ignoreDifferences:
    - group: batch
      kind: CronJob
      jsonPointers: [/spec/suspend]
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [RespectIgnoreDifferences=true]    # without it, a sync still writes Git's value
```

Also remove `suspend` from the manifest in Git (or leave it `false` and rely on `RespectIgnoreDifferences`), so a new CronJob starts active and later syncs don't touch the live value.

**B. Pausing is a Git commit.** `jobs-api` (or a human) opens a PR/commit that sets `suspend: true`; Argo CD applies it.

| | A: runtime-owned field | B: Git commit |
|---|---|---|
| Speed | seconds | a commit + sync (typically 1–3 min, longer with PR review) |
| Audit | `jobs-api` audit log | Git history (who, when, why, reviewed) |
| Survives cluster rebuild / DR | no: a rebuilt cluster resumes everything | yes |
| Needs | ignoreDifferences on every CronJob app | `jobs-api` with Git write access, or a human in the loop |
| Good for | incident response ("stop hammering the supplier now") | planned pauses (maintenance for a week) |

Many teams combine them: pause at runtime during an incident, then either resume or commit the pause within a day. Alert on CronJobs suspended for more than 24 h (section 7) so a runtime pause doesn't become permanent by accident.

Manual Jobs created by `jobs-api` are not in Git and don't carry Argo CD's tracking annotation, so Argo CD shows them as children of the CronJob but never prunes them. The CronJob's history limits (or a TTL) clean them up.

## 6. Pitfalls

<Warning>
**Common pitfalls**

- **Assuming exactly once.** The docs say a CronJob can create multiple concurrent Jobs in some circumstances, or none (controller down, deadline passed, `Forbid` skip, clock skew). Make every job idempotent: conditional updates, `SKIP LOCKED`, and an idempotency key from the Job name (`<cronjob>-<scheduled minute>`) when calling external systems. See [Background Jobs & Task Queues](/Software%20Engineering/Backend/Background%20Jobs%20%26%20Task%20Queues).
- **Missed schedules after control plane downtime or upgrades.** Without `startingDeadlineSeconds`, a per-minute job that missed more than 100 ticks doesn't start at all. With a huge deadline, one catch-up run fires late. Set a deadline that matches the job's value ("a report started 3 hours late is useless").
- **Surprise run on resume**: without a starting deadline, unsuspending fires the most recent missed run immediately.
- **GitOps drift on `suspend`** (section 5).
- **`concurrencyPolicy: Allow` (the default)** on anything that touches shared data. Default to `Forbid`; `Replace` kills a running Job mid-work.
- **`activeDeadlineSeconds` longer than the interval** with `Forbid`: the next ticks are skipped while a stuck run hangs. Keep it below the interval.
- **`restartPolicy: OnFailure` + `backoffLimit`**: restarts happen inside one Pod; logs of earlier attempts are harder to get and `podFailurePolicy` can't be used. Prefer `Never`.
- **TTL shorter than history**: history limits look broken, `kubectl get jobs` is empty, and failure metrics vanish before alerts fire.
- **No timezone**: the schedule follows the controller's zone (usually UTC on managed clusters). With `timeZone: Europe/Berlin`, DST matters: on the spring-forward day 02:00–02:59 doesn't exist and on the autumn day it exists twice. How the controller's cron library treats those ticks is easy to get wrong, so test it or keep local-time schedules out of 02:00–03:00. The same applies to `nightly-sales-report` at 02:00 on BullMQ ([BullMQ Job Schedulers in Practice](/Software%20Engineering/Backend/Scheduled%20Jobs/BullMQ%20Job%20Schedulers%20in%20Practice)).
- **No `terminationGracePeriodSeconds` handling in the job**: Stop, node drains and `Replace` all send SIGTERM; a job that ignores it gets SIGKILLed mid-write.
- **Manual runs skip the `concurrencyPolicy` check**: `kubectl create job --from` happily overlaps a running scheduled Job. Check for an active Job first (`jobs-api` does and returns `409`).
- **CronJob name longer than 52 characters**: Job creation fails.
</Warning>

## 7. Monitoring and alerts

kube-state-metrics exposes the objects' state (stable metrics listed):

| Metric | Meaning |
|---|---|
| `kube_cronjob_info{schedule, concurrency_policy, timezone}` | static info |
| `kube_cronjob_spec_suspend` | 1 if suspended |
| `kube_cronjob_status_active` | active Jobs |
| `kube_cronjob_status_last_schedule_time` | last time a Job was created |
| `kube_cronjob_status_last_successful_time` | last successful completion |
| `kube_cronjob_next_schedule_time` | next planned run |
| `kube_job_owner{owner_kind, owner_name}` | links a Job to its CronJob |
| `kube_job_status_start_time`, `kube_job_status_completion_time` | timing |
| `kube_job_status_active`, `kube_job_status_succeeded`, `kube_job_status_failed{reason}` | Pod counts |
| `kube_job_complete{condition}`, `kube_job_failed{condition}` | Job conditions (`condition="true"`) |
| `kube_job_created` | creation timestamp |

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata: { name: acme-cronjobs, namespace: shop-prod }
spec:
  groups:
    - name: acme-cronjobs
      rules:
        # A run owned by a CronJob failed within the last hour
        - alert: CronJobRunFailed
          expr: |
            (kube_job_failed{namespace="shop-prod", condition="true"} == 1)
              * on (namespace, job_name) group_left (owner_name)
                kube_job_owner{owner_kind="CronJob"}
            and on (namespace, job_name)
              (time() - kube_job_created) < 3600
          labels: { severity: ticket, team: platform }
          annotations:
            summary: "{{ $labels.owner_name }} run {{ $labels.job_name }} failed"
            runbook_url: https://runbooks.acme.dev/jobs/{{ $labels.owner_name }}

        # Missed runs: no success in 2x the interval (cleanup runs hourly), unless paused on purpose
        - alert: CronJobNoRecentSuccess
          expr: |
            (time() - kube_cronjob_status_last_successful_time{namespace="shop-prod", cronjob="cleanup-expired-carts"}) > 2 * 3600
            and on (namespace, cronjob) kube_cronjob_spec_suspend == 0
          for: 5m
          labels: { severity: page, team: checkout }
          annotations:
            summary: "cleanup-expired-carts has not succeeded for 2 intervals"

        # The controller isn't creating Jobs at all (deadline skips, >100 missed, broken schedule)
        - alert: CronJobNotScheduling
          expr: |
            (time() - kube_cronjob_status_last_schedule_time{namespace="shop-prod"}) > 2 * 3600
            and on (namespace, cronjob) kube_cronjob_spec_suspend == 0
          for: 10m
          labels: { severity: ticket }

        # A run is taking suspiciously long (activeDeadlineSeconds should kill it at 20 min)
        - alert: CronJobRunTooLong
          expr: |
            (time() - kube_job_status_start_time{namespace="shop-prod"}
              * on (namespace, job_name) group_left (owner_name) kube_job_owner{owner_kind="CronJob"})
            > 1800
            and on (namespace, job_name) kube_job_status_active > 0
          labels: { severity: ticket }

        # Someone paused a job and forgot about it
        - alert: CronJobSuspendedTooLong
          expr: kube_cronjob_spec_suspend{namespace="shop-prod"} == 1
          for: 24h
          labels: { severity: info }
```

Notes:
- "No recent success" catches everything that "failed" alerts miss: the controller being down, deadline skips, `Forbid` skips behind a hung run, a job that exits 0 without doing its work (pair it with an app metric such as `jobs_runs_total{job,status}` or a `carts_deleted_total` counter).
- `CronJobNotScheduling` needs a per-job threshold; generate these rules from the same list that defines the jobs, or join on a label with the interval.
- `jobs-api` adds its own metrics (`jobs_runs_total{job,status}`, `jobs_run_duration_seconds`) across BullMQ and Kubernetes so the dashboard and alerts look the same for both ([Cron with Message Queues - Architecture](/Software%20Engineering/Backend/Scheduled%20Jobs/Cron%20with%20Message%20Queues%20-%20Architecture)).

## Senior perspective

- **CronJob vs a queue scheduler.** CronJobs shine for jobs that need their own image, resources, isolation or long run time (reports on big nodes, Playwright with a browser, DB maintenance). Every run pays Pod start-up (image pull, scheduling: seconds to minutes), so for "every minute, 200 ms of work" a BullMQ job scheduler inside an existing worker is cheaper ([BullMQ Job Schedulers in Practice](/Software%20Engineering/Backend/Scheduled%20Jobs/BullMQ%20Job%20Schedulers%20in%20Practice)).
- **Design the failure modes first**: what if it runs twice, never, late, or for 3 hours? The answers become `concurrencyPolicy`, `startingDeadlineSeconds`, `activeDeadlineSeconds`, idempotency and the two alerts.
- **Control plane dependency**: CronJobs need a healthy kube-controller-manager. On managed clusters, upgrades can pause controllers for minutes; critical schedules need the "no recent success" alert, not just "failed".
- **Retries: pick one layer.** `backoffLimit` × in-app retries × BullMQ retries multiply. Decide where retries live and make the others 0.
- **Treat the jobTemplate as an API.** Labels on `jobTemplate.metadata` (not only on the Pod template) make Jobs selectable; annotations for owner/runbook make the dashboard self-describing; the `acme.dev/managed-by` label is the contract with `jobs-api`.
- **Review questions**: Is `timeZone` set? Is `concurrencyPolicy` explicit? What's the deadline and why? Does the job handle SIGTERM within the grace period? Where do logs of a run from last week live? Who gets paged when it silently stops running?

<Info>
**🆕 2025–2026**

- **v1.33 (2025-04-23)**: **Job backoff limit per index** and **Job success policy** reached GA (blog posts 2025-05-13 and 2025-05-15).
- **v1.34 (2025-08-27)**: **Pod replacement policy for Jobs** reached GA (blog 2025-09-05).
- **v1.35 (2025-12-17)**: **Job `managedBy`** reached GA (blog 2025-12-18), letting external controllers such as MultiKueue own a Job's status.
- **2026-01-21**: the **Kubernetes Dashboard** was archived; **Headlamp** (SIG UI) is the recommended web UI for browsing CronJobs and Jobs.
- Earlier baselines still worth knowing: `timeZone` stable in **v1.27** (2023), `podFailurePolicy` stable in **v1.31** (2024).
</Info>

## Interview questions

<Accordion title="What's the difference between a CronJob, a Job and a Pod?">
A Pod runs containers. A Job runs Pods until a number of them succeed, retrying failures up to `backoffLimit` and within `activeDeadlineSeconds`. A CronJob creates a new Job on each tick of its schedule and manages history and overlap between those Jobs.
</Accordion>

<Accordion title="Explain concurrencyPolicy and when you'd use each value.">
`Allow` (default) lets runs overlap; fine for independent, short, idempotent work. `Forbid` skips a tick if the previous Job is still active; the safe default for anything touching shared data. `Replace` kills the running Job and starts the new one; useful when only the latest run matters (e.g. refreshing a cache) and a stuck run should be replaced.
</Accordion>

<Accordion title="What does startingDeadlineSeconds do, and what is the 100 missed schedules rule?">
It skips a run that couldn't start within N seconds of its scheduled time, and it bounds the window the controller uses to count missed schedules. Without it, the controller counts from the last scheduled time; if more than 100 schedules were missed (e.g. a per-minute job after a 2-hour controller outage), it doesn't start the Job at all. With a deadline it only counts misses inside that window and starts normally.
</Accordion>

<Accordion title="How do you pause, resume, run now and stop a CronJob?">
Pause: patch `spec.suspend: true` (running Jobs continue). Resume: `false`; without `startingDeadlineSeconds` the most recent missed run starts immediately. Run now: `kubectl create job --from=cronjob/name unique-name` (copies the jobTemplate, marks it manual, ignores concurrencyPolicy). Stop: delete the Job with foreground propagation; Pods get SIGTERM and the grace period, then SIGKILL. Suspending the Job instead stops the Pods but keeps the record.
</Accordion>

<Accordion title="Your CronJob is managed by Argo CD with self-heal. What happens when someone suspends it with kubectl?">
Argo CD sees drift and re-applies Git's `suspend: false`, so the job resumes by itself. Either let the runtime own the field (`ignoreDifferences` on `/spec/suspend` plus `RespectIgnoreDifferences=true`) or make pausing a Git commit. Runtime is fast; Git is auditable and survives a cluster rebuild.
</Accordion>

<Accordion title="Does a CronJob guarantee exactly-once execution?">
No. It can create two Jobs in rare cases, create none when the controller is down or deadlines pass, and Pods can be retried after partial work. Jobs must be idempotent: conditional writes, locks such as `SKIP LOCKED`, and idempotency keys derived from the scheduled time (the Job name encodes it).
</Accordion>

<Accordion title="How do you alert on scheduled jobs in Kubernetes?">
From kube-state-metrics: alert on recent failed Jobs owned by a CronJob (`kube_job_failed` joined with `kube_job_owner`), on "no success in 2× the interval" (`kube_cronjob_status_last_successful_time`, ignoring suspended ones), on runs that take too long, and on CronJobs suspended for too long. Add application metrics so "exit 0 but did nothing" is visible too.
</Accordion>

<Accordion title="What does podFailurePolicy add over backoffLimit?">
It classifies failures: fail the Job immediately on non-retryable exit codes (bad config), ignore disruptions like node drains or preemption so they don't consume retries, or count them normally. It requires `restartPolicy: Never` and saves compute and time on hopeless retries.
</Accordion>

## Related
- [Scheduled Jobs Index](/Software%20Engineering/Backend/Scheduled%20Jobs/Scheduled%20Jobs%20Index) · [Backend Index](/Software%20Engineering/Backend/Backend%20Index)
- [Cron Jobs](/Software%20Engineering/Backend/Cron%20Jobs) · [Background Jobs & Task Queues](/Software%20Engineering/Backend/Background%20Jobs%20%26%20Task%20Queues) · [Cron with Message Queues - Architecture](/Software%20Engineering/Backend/Scheduled%20Jobs/Cron%20with%20Message%20Queues%20-%20Architecture) · [BullMQ Job Schedulers in Practice](/Software%20Engineering/Backend/Scheduled%20Jobs/BullMQ%20Job%20Schedulers%20in%20Practice) · [Job Control API - Pause, Resume, Run Now & Stop](/Software%20Engineering/Backend/Scheduled%20Jobs/Job%20Control%20API%20-%20Pause%2C%20Resume%2C%20Run%20Now%20%26%20Stop) · [Job Scheduler Dashboard with Next.js](/Software%20Engineering/Backend/Scheduled%20Jobs/Job%20Scheduler%20Dashboard%20with%20Next.js) · [Playwright Automation as a Cron Job](/Software%20Engineering/Backend/Scheduled%20Jobs/Playwright%20Automation%20as%20a%20Cron%20Job)
- [Kubernetes Operations - Logs, Debugging & Scheduling](/Software%20Engineering/DevOps/Delivery%20Platform/Kubernetes%20Operations%20-%20Logs%2C%20Debugging%20%26%20Scheduling) · [Argo CD in Practice](/Software%20Engineering/DevOps/Delivery%20Platform/Argo%20CD%20in%20Practice) · [Grafana & the LGTM Stack](/Software%20Engineering/DevOps/Delivery%20Platform/Grafana%20%26%20the%20LGTM%20Stack) · [Kubernetes Delivery Platform - End to End](/Software%20Engineering/DevOps/Delivery%20Platform/Kubernetes%20Delivery%20Platform%20-%20End%20to%20End) · [Deploying Node.js Apps to Kubernetes](/Software%20Engineering/DevOps/Delivery%20Platform/Deploying%20Node.js%20Apps%20to%20Kubernetes)

## Sources
- Kubernetes docs: [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) · [Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/) · [Running automated tasks with a CronJob](https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/) · [Handling retriable and non-retriable Pod failures with Pod failure policy](https://kubernetes.io/docs/tasks/job/pod-failure-policy/) · [TTL-after-finished controller](https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/) · [Garbage collection (foreground/background deletion)](https://kubernetes.io/docs/concepts/architecture/garbage-collection/) · [kubectl create job](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_job/)
- [KEP-3140: TimeZone support in CronJob](https://github.com/kubernetes/enhancements/blob/master/keps/sig-apps/3140-TimeZone-support-in-CronJob/README.md)
- Kubernetes blog: [Pod failure policy GA in 1.31 (2024-08-19)](https://kubernetes.io/blog/2024/08/19/kubernetes-1-31-pod-failure-policy-for-jobs-goes-ga/) · [v1.33 release (2025-04-23)](https://kubernetes.io/blog/2025/04/23/kubernetes-v1-33-release/) · [Backoff limit per index GA (2025-05-13)](https://kubernetes.io/blog/2025/05/13/kubernetes-v1-33-jobs-backoff-limit-per-index-goes-ga/) · [Job success policy GA (2025-05-15)](https://kubernetes.io/blog/2025/05/15/kubernetes-1-33-jobs-success-policy-goes-ga/) · [Pod replacement policy GA (2025-09-05)](https://kubernetes.io/blog/2025/09/05/kubernetes-v1-34-pod-replacement-policy-for-jobs-goes-ga/) · [v1.35 release (2025-12-17)](https://kubernetes.io/blog/2025/12/17/kubernetes-v1-35-release/) · [Job managedBy GA (2025-12-18)](https://kubernetes.io/blog/2025/12/18/kubernetes-v1-35-job-managedby-for-jobs-goes-ga/) · [From Kubernetes Dashboard to Headlamp (2026-06-01)](https://kubernetes.io/blog/2026/06/01/dashboard-to-headlamp/)
- kube-state-metrics: [CronJob metrics](https://github.com/kubernetes/kube-state-metrics/blob/main/docs/metrics/workload/cronjob-metrics.md) · [Job metrics](https://github.com/kubernetes/kube-state-metrics/blob/main/docs/metrics/workload/job-metrics.md)
- Argo CD: [Diffing customization / ignoreDifferences](https://argo-cd.readthedocs.io/en/stable/user-guide/diffing/) · [Sync options: RespectIgnoreDifferences](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/)
- [PostgreSQL: SELECT ... FOR UPDATE SKIP LOCKED](https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE)
