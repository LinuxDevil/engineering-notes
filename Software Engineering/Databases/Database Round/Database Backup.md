---
title: "Database Backup"
tags: [databases, databases/operations, databases/postgres]
aliases: [Backup, Backups, Restore, PITR, Point-in-Time Recovery, RPO, RTO, pg_dump, pgBackRest, WAL archiving, 3-2-1 rule, Disaster Recovery]
updated: 2026-09-27
---
# Database Backup

<Note>
**TL;DR**

A backup is a copy of your data you can **restore**; a backup you've never restored is a hope, not a backup. Two numbers drive every design: **RPO** (how much data you may lose) and **RTO** (how long you may be down). **Logical** backups (`pg_dump`) are portable but slow to restore; **physical** backups (`pg_basebackup`, pgBackRest, snapshots) plus continuous **WAL archiving** give **point-in-time recovery** (PITR). Follow **3-2-1** (3 copies, 2 media, 1 off-site, ideally one immutable), encrypt, and test restores on a schedule.
</Note>

## The idea

Backup is the process of creating copies of your database that can be restored if needed: after hardware failure, a bad migration, an accidental `DELETE` without `WHERE`, ransomware, or a region outage. **Replication is not a backup**: a `DROP TABLE` replicates to every replica within milliseconds.

## RPO and RTO

| Term | Question | Example target | Drives |
|---|---|---|---|
| **RPO** (Recovery Point Objective) | How much data (time) can we lose? | 5 minutes | Backup/WAL archiving frequency, sync replication |
| **RTO** (Recovery Time Objective) | How long can we be down? | 1 hour | Restore speed, standby replicas, automation |

{/* diagram source: /diagrams/diagram-115.mmd */}
![diagram](/diagrams/diagram-115.svg)

A nightly `pg_dump` gives RPO ≈ 24 h and RTO = hours for a big database. WAL archiving gives RPO ≈ seconds to minutes. A hot standby gives RTO ≈ minutes, but still needs backups for logical errors.

## Backup strategies (from the original notes, expanded)

1. **Full backup**: copies all data. Simple to restore; large and slow.
2. **Incremental backup**: only data changed since the **last backup of any type**. Small and fast; a restore needs the full plus every incremental in the chain.
3. **Differential backup**: all data changed since the **last full** backup. Grows over the week; a restore needs the full plus the latest differential only.
4. **Log backup**: back up the **transaction log** (WAL in Postgres, binlog in MySQL, log backups in SQL Server). Contains every change, so you can replay to any moment: **point-in-time recovery**.
5. **Snapshot backup**: a read-only copy at an instant, usually a storage/volume snapshot (EBS, ZFS, LVM). Fast; must be crash-consistent (all volumes at the same instant) and ideally coordinated with the database.
6. **Hot vs cold backup**: a **hot** (online) backup is taken while the database runs and serves users; a **cold** (offline) backup is taken with the database shut down. Cold is simplest and consistent, but means downtime.

| Type | Backup size/time | Restore needs | Restore time |
|---|---|---|---|
| Full | largest | the full | fast-ish |
| Incremental | smallest | full + all incrementals | slowest |
| Differential | medium, grows | full + last differential | medium |
| + Log/WAL | continuous | base + logs up to target time | depends on log volume |

### Logical vs physical

| | Logical (`pg_dump`, `mysqldump`, `mongodump`) | Physical (`pg_basebackup`, pgBackRest, Barman, XtraBackup, snapshots) |
|---|---|---|
| Content | SQL/archive of schema + rows | Data files + WAL |
| Portability | Across versions, platforms, selective tables | Same major version and architecture |
| Speed on large DBs | Slow to dump, very slow to restore (rebuild indexes) | Fast (copy files) |
| PITR | No | Yes, with WAL archive |
| Good for | Small DBs, migrations, exports, per-table restore | Production DR |

## Worked examples (PostgreSQL)

**Logical backup and restore**

```bash
# custom format: compressed, supports parallel & selective restore
pg_dump -Fc -d app -f app_2026-09-27.dump
# directory format with 4 parallel jobs
pg_dump -Fd -j 4 -d app -f app_dump_dir/

# restore into a new database with 4 parallel jobs
createdb app_restore
pg_restore -j 4 -d app_restore app_2026-09-27.dump
# only one table
pg_restore -d app_restore -t orders app_2026-09-27.dump

pg_dumpall --globals-only > globals.sql   # roles and tablespaces aren't in pg_dump
```

**WAL archiving + base backup = PITR**

```ini
# postgresql.conf
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /mnt/wal/%f && cp %p /mnt/wal/%f'   # toy example; use a real tool
```

```bash
pg_basebackup -D /backups/base_2026-09-27 -Ft -z -X stream -P
```

Restore to just before an accidental delete at 14:03:

```ini
# on the restored data directory: postgresql.conf (PG 12+), plus an empty recovery.signal file
restore_command = 'cp /mnt/wal/%f %p'
recovery_target_time = '2026-09-27 14:02:30+00'
recovery_target_action = 'promote'
```

**pgBackRest** (the common production tool: parallel, compressed, encrypted, S3/GCS/Azure repos, full/diff/incr, retention, verify):

```bash
pgbackrest --stanza=main stanza-create
pgbackrest --stanza=main --type=full backup      # e.g. weekly
pgbackrest --stanza=main --type=diff backup      # e.g. daily
pgbackrest --stanza=main --type=incr backup      # e.g. hourly
pgbackrest --stanza=main info
pgbackrest --stanza=main verify
# point-in-time restore
pgbackrest --stanza=main --delta --type=time \
  --target="2026-09-27 14:02:30+00" --target-action=promote restore
```

with `archive_command = 'pgbackrest --stanza=main archive-push %p'`. Barman and WAL-G are alternatives; managed services (RDS, Cloud SQL, Atlas) do this for you with a configurable retention window.

Other engines: MySQL `mysqldump`/`mysqlpump` (logical), Percona XtraBackup or MySQL Enterprise Backup (physical) + binlogs for PITR; SQL Server full/differential/log backups; MongoDB `mongodump`, filesystem snapshots, or Atlas continuous backups with PITR.

## The 3-2-1 rule

- **3** copies of the data (production + 2 backups),
- on **2** different media/storage types,
- **1** off-site (another region/account).

Modern extension, **3-2-1-1-0**: plus **1** immutable or air-gapped copy (object lock / WORM, separate account that production credentials can't delete), and **0** errors in restore verification.

## Restore testing

- Automate a **scheduled restore** (weekly/monthly) into a scratch environment: restore, start, run sanity queries (row counts, latest timestamps, checksums), record the duration. That duration is your real RTO.
- Run `pg_amcheck` / `pgbackrest verify` to detect corruption in backups.
- Practise PITR drills and document the runbook; the person on call at 3 a.m. shouldn't be reading docs for the first time.
- Monitor: last successful backup age, WAL archive lag/failures (`pg_stat_archiver`), repo size.

## Disaster recovery planning

Define RPO/RTO per system, decide the mechanisms (backups + WAL, standby in another AZ/region, cross-region backup copies), secure and encrypt backups (they contain everything, including PII), set retention in line with legal requirements (and GDPR deletion obligations), and rehearse.

## Pitfalls

<Warning>
**Common pitfalls**

- Treating **replicas** or RAID as backups.
- Backups stored in the **same account/region** with the same credentials as production (one compromise or mistake deletes both).
- `pg_dump` without roles/globals, or restoring without the right extensions/versions.
- Forgetting that a big logical restore also rebuilds every index: plan hours.
- WAL archive silently failing (disk full, permissions) while backups "succeed"; PITR then has gaps.
- Never testing restore. The classic incident report line: "the backups hadn't worked for months".
</Warning>

## Senior perspective

- Start from business RPO/RTO, then choose: nightly dumps are fine for a small internal app; money systems need PITR plus a synchronous or near-synchronous standby.
- Separate **logical-error recovery** (PITR, delayed replica with `recovery_min_apply_delay`) from **infrastructure failure** (standby failover).
- Consider tool sustainability: open-source tools depend on maintainers and sponsors (see the pgBackRest episode below). Prefer tools with active communities and have an exit plan.
- Review questions: When was the last restore test and how long did it take? Can an attacker with prod credentials delete the backups? Are backups encrypted and is the key backed up elsewhere?

<Info>
**🆕 2025–2026**

- **PostgreSQL 17 (2024-09-26)** added native **incremental backups**: `pg_basebackup --incremental=<manifest>` (needs `summarize_wal = on`) and the new `pg_combinebackup` tool to reconstruct a full backup from a chain. `pg_dump` gained `--filter`.
- **PostgreSQL 18 (2025-09-25)**: `pg_upgrade` now keeps planner statistics and has a `--swap` mode; new clusters have data checksums on by default, which helps detect corrupted pages early.
- **pgBackRest**: on **2026-04-27** its sole maintainer announced the project would no longer be maintained after losing sponsorship; on **2026-05-18** it was revived with a coalition of sponsors (including AWS, Supabase, Percona, Dalibo and others per pgbackrest.org). The latest release listed is **2.59.2 (2026-09-27)** with PostgreSQL 19 beta support. Lesson: track the health of critical open-source dependencies.
</Info>

## Interview questions

<Accordion title="What's the difference between RPO and RTO?">
RPO is the maximum acceptable data loss measured in time (how old the last recoverable point may be). RTO is the maximum acceptable downtime to restore service. RPO drives backup/log frequency; RTO drives restore speed and standby architecture.
</Accordion>

<Accordion title="Incremental vs differential backup?">
Incremental captures changes since the last backup of any type (small, but a restore needs the whole chain). Differential captures changes since the last full (grows over time, restore needs full + latest differential).
</Accordion>

<Accordion title="How does point-in-time recovery work in PostgreSQL?">
Take a physical base backup and continuously archive WAL segments. To recover, restore the base backup, configure `restore_command` and a `recovery_target_time` (or LSN/xid), and Postgres replays WAL up to that point, then promotes.
</Accordion>

<Accordion title="Why isn't replication a backup?">
Replication copies every change, including mistakes and corruption, almost instantly. Backups preserve history so you can go back to before the mistake.
</Accordion>

<Accordion title="Logical or physical backups for a 2 TB production database?">
Physical backups with WAL archiving (pgBackRest or a managed service) for DR and PITR, because logical restores of 2 TB take too long. Logical dumps can supplement for selective table restores or version migrations.
</Accordion>

<Accordion title="What is the 3-2-1 rule?">
Keep three copies of data on two different media with one copy off-site; modern practice adds one immutable/air-gapped copy and zero errors on verified restores.
</Accordion>

## Related
- [Databases Index](/Software%20Engineering/Databases/Databases%20Index) · [Database Round](/Software%20Engineering/Databases/Database%20Round/Database%20Round)
- [Replication & 2PC 3PC](/Software%20Engineering/Databases/Database%20Round/Replication%20%26%202PC%203PC) · [Clustering and Scaling](/Software%20Engineering/Databases/Database%20Round/Clustering%20and%20Scaling) · [Transactions & ACID](/Software%20Engineering/Databases/Database%20Round/Transactions%20%26%20ACID) (WAL)
- [Database Migrations & Schema Evolution](/Software%20Engineering/Databases/Core%20Concepts/Database%20Migrations%20%26%20Schema%20Evolution) · [PostgreSQL 17 to 18 - What's New](/Software%20Engineering/Databases/Core%20Concepts/PostgreSQL%2017%20to%2018%20-%20What's%20New)

## Sources
- PostgreSQL docs, Backup and Restore: https://www.postgresql.org/docs/current/backup.html
- PostgreSQL docs, Continuous Archiving and PITR: https://www.postgresql.org/docs/current/continuous-archiving.html
- PostgreSQL docs, pg_basebackup / pg_combinebackup: https://www.postgresql.org/docs/current/app-pgbasebackup.html · https://www.postgresql.org/docs/current/app-pgcombinebackup.html
- PostgreSQL 17 release: https://www.postgresql.org/about/news/postgresql-17-released-2936/
- PostgreSQL 18 release: https://www.postgresql.org/about/news/postgresql-18-released-3142/
- pgBackRest user guide and news: https://pgbackrest.org/user-guide.html · https://pgbackrest.org/news.html
- Percona Community, "pgBackRest is archived, what now?" (2026-04-28): https://percona.community/blog/2026/04/28/pgbackrest-is-archived-what-now/
- CISA, Data Backup Options (3-2-1): https://www.cisa.gov/sites/default/files/publications/data_backup_options.pdf
