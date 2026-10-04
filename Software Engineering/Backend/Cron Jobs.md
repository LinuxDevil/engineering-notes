---
title: "Cron Jobs"
tags: [backend, backend/jobs]
aliases: [Cron, Crontab, Cron Expression, Scheduled Tasks, Kubernetes CronJob, whenever gem, systemd timers]
updated: 2026-09-27
---
# Cron Jobs

<Note>
**TL;DR**

**Cron** is the Unix scheduler that runs commands at fixed times defined by a five-field **cron expression** (`minute hour day-of-month month day-of-week`). It is perfect for "run this script nightly" on one machine, but on a fleet of servers or containers you must handle **duplicates, overlaps, missed runs, time zones and observability**, which is why production systems move to Kubernetes CronJobs, systemd timers, or a job framework's scheduler that enqueues work into a queue ([Background Jobs & Task Queues](/Software%20Engineering/Backend/Background%20Jobs%20%26%20Task%20Queues)). Same idea as scheduled tasks in Spring Boot (`@Scheduled`).
</Note>

## What is a cron job?

A **cron job** is a scheduled command that the `cron` daemon runs in the future, usually periodically, e.g. send a report every morning, clean temp files hourly, renew certificates daily. Some applications (Drupal, WHMCS, WordPress without real traffic) need cron jobs to run their maintenance tasks.

In a nutshell: *a cron job is a scheduling rule that triggers an action at set times.*

Most cron jobs have three parts:
1. **The script** to execute (a shell script, `rake` task, `node` script, `curl` to an endpoint).
2. **The schedule** telling cron when to run it (the crontab entry; on shared hosting often set through cPanel).
3. **The effect/output**: frequently scripts modify files or databases, or perform tasks that don't change data, like sending email notifications. Output goes to mail/logs unless redirected.

## Crontab: where jobs live

Jobs are written in a file called the **crontab** ("cron table").

- Per-user crontabs: edited with `crontab -e`, stored by the system in `/var/spool/cron/` (e.g. `/var/spool/cron/crontabs/<user>` on Debian). Don't edit those files directly. (The original version of this note said `/usr/bin/`; that's where the `crontab` *binary* lives, not the tables.)
- System-wide: `/etc/crontab` and files in `/etc/cron.d/` (these have an extra **user** field).
- Drop-in folders: `/etc/cron.hourly`, `/etc/cron.daily`, `/etc/cron.weekly`, `/etc/cron.monthly`.

```bash
crontab -e      # edit current user's crontab
crontab -l      # list it
crontab -r      # remove it (careful!)
```

## Cron expression syntax

```text
┌───────────── minute        (0–59)
│ ┌─────────── hour          (0–23)
│ │ ┌───────── day of month  (1–31)
│ │ │ ┌─────── month         (1–12 or JAN–DEC)
│ │ │ │ ┌───── day of week   (0–7, 0 and 7 = Sunday, or SUN–SAT)
│ │ │ │ │
* * * * *  command
```

| Expression | Meaning |
|---|---|
| `*/5 * * * *` | every 5 minutes |
| `0 * * * *` | at minute 0 of every hour |
| `30 2 * * *` | daily at 02:30 |
| `0 9 * * 1-5` | 09:00 Monday–Friday |
| `0 0 1 * *` | midnight on the 1st of each month |
| `15 14 1 * *` | 14:15 on the 1st of each month |
| `0 22 * * 1,3,5` | 22:00 on Mon, Wed, Fri |
| `@reboot` | once at startup |
| `@daily` / `@hourly` / `@weekly` | shortcuts |

Gotcha: if both day-of-month and day-of-week are restricted, classic cron runs when **either** matches. Spring and Quartz use a **six/seven-field** syntax with seconds first (`0 0 9 * * MON-FRI`), so don't copy expressions blindly between tools. Test with crontab.guru.

### A robust crontab line

```cron
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin
MAILTO=ops@example.com
CRON_TZ=UTC

# nightly cleanup, never overlapping, with logs
30 2 * * * flock -n /tmp/cleanup.lock /opt/app/bin/cleanup.sh >> /var/log/app/cleanup.log 2>&1
```

- Cron runs with a **minimal environment** (short `PATH`, no shell profile): use absolute paths.
- `flock -n` prevents overlap if the previous run is still going.
- Redirect output, or rely on `MAILTO`; silent failures are the norm otherwise.
- Be explicit about **time zones** and DST (a 02:30 job may run twice or never on DST change days in local time; prefer UTC).

## Cron jobs in Rails: the `whenever` gem

[`whenever`](https://github.com/javan/whenever) lets you write the crontab in Ruby. Instead of editing the crontab yourself, you describe jobs in `config/schedule.rb` and `whenever` updates the crontab for you (typically on deploy).

```ruby
# config/schedule.rb
set :output, "log/cron.log"
set :environment, "production"

every 1.day, at: "4:30 am" do
  rake "reports:send_daily"
end

every :hour do
  runner "Cleanup.expired_sessions"
end
```

```bash
bundle exec whenever                  # preview generated crontab
bundle exec whenever --update-crontab # write it
```

**Note:** the actual task (e.g. sending batch messages) should **not** be written in the crontab or in `schedule.rb`. Write it separately as a **Rake task** (or a job class), and only *schedule* that task. This keeps it testable and runnable by hand:

```ruby
# lib/tasks/reports.rake
namespace :reports do
  desc "Send the daily report"
  task send_daily: :environment do
    DailyReportJob.perform_later    # enqueue; the job does the work
  end
end
```

Modern Rails (8.x) apps often skip system cron altogether and use **Solid Queue recurring tasks** (`config/recurring.yml`), or `sidekiq-cron`: see [Background Jobs & Task Queues](/Software%20Engineering/Backend/Background%20Jobs%20%26%20Task%20Queues).

## Other schedulers

| Tool | Where | Notes |
|---|---|---|
| **systemd timers** | Linux hosts | logs in journald, `Persistent=true` runs missed jobs after downtime, dependencies, resource limits |
| **Kubernetes CronJob** | K8s | runs a pod per schedule; `concurrencyPolicy`, `startingDeadlineSeconds`, `timeZone` |
| **Spring `@Scheduled`** | Java apps | in-process; add **ShedLock** so only one instance runs it ([Java & Spring Index](/Software%20Engineering/Backend/Java/Java%20%26%20Spring%20Index)) |
| **Job framework schedulers** | Sidekiq-cron, Solid Queue recurring, BullMQ job schedulers, Celery beat | enqueue into the job system; retries and dashboards for free |
| **Cloud schedulers** | EventBridge Scheduler, Cloud Scheduler | call an HTTP endpoint or queue on a schedule, managed |

```yaml
# Kubernetes CronJob
apiVersion: batch/v1
kind: CronJob
metadata: { name: nightly-report }
spec:
  schedule: "30 2 * * *"
  timeZone: "Etc/UTC"
  concurrencyPolicy: Forbid          # don't start if the previous run is still running
  startingDeadlineSeconds: 600       # if missed by >10 min, skip
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 3600
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: report
              image: ghcr.io/acme/app:1.42.0
              args: ["bin/rails", "reports:send_daily"]
```

### Cron inside Docker containers

Running `cron` inside an app container is usually an anti-pattern: one process per container, env vars aren't passed to cron, logs don't reach `stdout`, and scaling the container duplicates the jobs. Prefer a separate scheduler container/CronJob, or the host's/platform's scheduler, that runs your image with a command.

## Pitfalls

<Warning>
**Common mistakes**

- **Same crontab on every server** → the job runs N times. Run it on one designated node, use a distributed lock, or move scheduling into a central scheduler that enqueues one job.
- **Overlapping runs** when a job takes longer than its interval.
- **Missed runs** during downtime or deploys go unnoticed.
- Non-idempotent jobs (a retry or double-run sends the newsletter twice). Make jobs idempotent ([API Versioning, Idempotency & Backward Compatibility](/Software%20Engineering/APIs%20%26%20Auth/API%20Versioning%2C%20Idempotency%20%26%20Backward%20Compatibility)).
- No monitoring: nobody knows the backup job has failed for 3 weeks. Use a "dead man's switch" (Healthchecks.io, Cronitor, Sentry Crons) that alerts when a ping *doesn't* arrive.
- Local time zone + DST.
</Warning>

## Senior perspective

- Cron should **trigger**, not **do**: the schedule enqueues a job; workers with retries, timeouts, concurrency limits and metrics do the work.
- Every scheduled job needs: owner, expected duration, alert on failure *and* on absence, idempotency, and a manual "run now" path.
- For complex dependencies (job B after job A, backfills, data pipelines) use a workflow orchestrator (Airflow, Dagster, Temporal) instead of chained cron times.
- Review questions: *What happens if this runs twice? If it doesn't run for a day? If it runs for longer than its interval? In which time zone?*

## Interview questions

<Accordion title="Explain the cron expression `0 9 * * 1-5`.">
Minute 0, hour 9, any day of month, any month, Monday to Friday: 09:00 on weekdays.
</Accordion>

<Accordion title="How do you avoid a cron job running on every instance of a horizontally scaled app?">
Don't schedule it in every instance. Use a single scheduler (Kubernetes CronJob with concurrencyPolicy Forbid, a cloud scheduler, or the job framework's scheduler that enqueues one job), or a distributed lock (ShedLock, Redis lock, DB advisory lock) so only one instance executes.
</Accordion>

<Accordion title="Cron vs a background job queue?">
Cron decides *when* something runs; a queue decides *how* it runs reliably (workers, retries with backoff, concurrency, visibility). Best practice is cron/scheduler enqueuing a job into the queue.
</Accordion>

<Accordion title="How do you monitor scheduled jobs?">
Log start/finish with duration and outcome, emit metrics, and use a heartbeat/dead man's switch that alerts when the expected success ping doesn't arrive; alert on failures and on runs exceeding expected duration.
</Accordion>

<Accordion title="Why is running cron inside an application container discouraged?">
It breaks one-process-per-container, loses environment variables and stdout logs, complicates health checks, and every replica runs the jobs. Use a separate CronJob or scheduler that runs the image with a command.
</Accordion>

## Related
- Scheduled jobs deep dive: [Scheduled Jobs Index](/Software%20Engineering/Backend/Scheduled%20Jobs/Scheduled%20Jobs%20Index) · [Cron with Message Queues - Architecture](/Software%20Engineering/Backend/Scheduled%20Jobs/Cron%20with%20Message%20Queues%20-%20Architecture) · [BullMQ Job Schedulers in Practice](/Software%20Engineering/Backend/Scheduled%20Jobs/BullMQ%20Job%20Schedulers%20in%20Practice) · [Kubernetes CronJobs in Practice](/Software%20Engineering/Backend/Scheduled%20Jobs/Kubernetes%20CronJobs%20in%20Practice) · [Job Control API - Pause, Resume, Run Now & Stop](/Software%20Engineering/Backend/Scheduled%20Jobs/Job%20Control%20API%20-%20Pause%2C%20Resume%2C%20Run%20Now%20%26%20Stop) · [Job Scheduler Dashboard with Next.js](/Software%20Engineering/Backend/Scheduled%20Jobs/Job%20Scheduler%20Dashboard%20with%20Next.js)
- [Backend Index](/Software%20Engineering/Backend/Backend%20Index)
- [Background Jobs & Task Queues](/Software%20Engineering/Backend/Background%20Jobs%20%26%20Task%20Queues) · [Rails Clean Architecture](/Software%20Engineering/Backend/Rails%20Clean%20Architecture) · [API Versioning, Idempotency & Backward Compatibility](/Software%20Engineering/APIs%20%26%20Auth/API%20Versioning%2C%20Idempotency%20%26%20Backward%20Compatibility)
- [Java & Spring Index](/Software%20Engineering/Backend/Java/Java%20%26%20Spring%20Index) (Spring `@Scheduled`) · [Node.js Index](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Index)

## Sources
- [Hivelocity: What is a cron job](https://www.hivelocity.net/kb/what-is-cron-job/)
- [dev.to: Cron jobs in Rails, a simple guide to the whenever gem](https://dev.to/risafj/cron-jobs-in-rails-a-simple-guide-to-actually-using-the-whenever-gem-now-with-tasks-2omi)
- [adminschoice: Crontab quick reference](https://www.adminschoice.com/crontab-quick-reference)
- [whenever (GitHub)](https://github.com/javan/whenever)
- [Airplane: Docker cron jobs, how to run cron inside containers](https://www.airplane.dev/blog/docker-cron-jobs-how-to-run-cron-inside-containers) (the original link; the site may no longer be online)
- [crontab(5) man page](https://man7.org/linux/man-pages/man5/crontab.5.html) · [crontab.guru](https://crontab.guru/)
- [Kubernetes: CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
- [systemd.timer](https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html)
- [ShedLock](https://github.com/lukas-krecan/ShedLock)
