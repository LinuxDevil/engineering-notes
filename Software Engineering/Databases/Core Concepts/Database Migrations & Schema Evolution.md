---
title: "Database Migrations & Schema Evolution"
tags: [databases, databases/operations]
aliases: [Database Migrations, Schema Migrations, Schema Evolution, Zero-Downtime Migrations, Expand and Contract, Parallel Change, Online Schema Change]
updated: 2026-09-27
---
# Database Migrations & Schema Evolution

<Note>
**TL;DR**

A **migration** is a versioned, reviewed script that changes the schema (and sometimes data) so every environment evolves the same way. In production the hard part is not the SQL but **doing it without downtime**: avoid long exclusive locks, keep old and new app versions compatible, and split breaking changes into **expand → migrate → contract** steps. Set `lock_timeout`, backfill in batches, build indexes concurrently, and test on production-sized data before you ship.
</Note>

## The idea

- The schema is code. It lives in the repo, changes through pull requests, and is applied by a tool, never by hand in a SQL console.
- The tool records which migrations ran (e.g. `flyway_schema_history`, `alembic_version`, `schema_migrations`) so each runs exactly once per database.
- Two parts of every change: **the DDL itself** (can it lock or rewrite a big table?) and **the compatibility window** (old and new app code run at the same time during a deploy).

## Migration tools

| Tool | Ecosystem | Style | Notes |
|---|---|---|---|
| **Flyway** | JVM, CLI | Versioned SQL (`V3__add_email.sql`) | Simple, SQL-first; undo migrations are a paid feature |
| **Liquibase** | JVM, CLI | Changelogs (XML/YAML/SQL) | Rollback blocks, preconditions, diff tools |
| **Alembic** | Python / SQLAlchemy | Versioned Python scripts, autogenerate | `upgrade()` / `downgrade()` |
| **Rails migrations** | Ruby | Ruby DSL, `change` reversible | `strong_migrations` gem for safety checks |
| **Django migrations** | Python | Auto-generated from models | `sqlmigrate` to view SQL |
| **Prisma Migrate** | Node/TS | Generated SQL from `schema.prisma` | `migrate dev` vs `migrate deploy` |
| **golang-migrate** | Go, CLI | Up/down SQL files | Minimal |
| **Atlas** | Any, CLI | Declarative (desired state) or versioned | Diffing, linting, drift detection |
| **sqitch** | Any, CLI | Change plan with deploy/revert/verify scripts | Dependency-based ordering, no ORM |

### Versioned vs declarative (state-based)

- **Versioned (imperative)**: you write each step (`ALTER TABLE ... ADD COLUMN`). Full control, explicit ordering, easy to review for locking behavior. Most common.
- **Declarative**: you describe the desired schema; the tool diffs it against the live DB and generates the steps (Atlas, Prisma's generator, SQL Server DACPAC, Skeema for MySQL). Less boilerplate, but **always review the generated plan**: a rename can come out as `DROP COLUMN` + `ADD COLUMN`, which loses data.

### Forward-only vs down migrations

- **Down migrations** look like safety but rarely are: once new data has been written in the new shape, "down" may lose it, and they are almost never tested.
- Many teams run **forward-only**: to undo, write a new forward migration. Rollback safety comes from designing changes to be backward compatible, not from `downgrade()`.
- Keep down migrations for local development convenience if the tool makes them cheap.

## Zero-downtime principles

1. **Never break the running version.** During a rolling deploy, old and new code hit the same schema. Every migration must work with both.
2. **Additive first, destructive last.** Add columns/tables freely (safely); remove only after no code reads or writes them.
3. **Short locks only.** Anything that holds `ACCESS EXCLUSIVE` on a hot table must finish in milliseconds.
4. **Separate schema changes from data backfills.** DDL in a migration; big data updates in a batched background job.
5. **One risky thing per deploy.** Easier to diagnose, easier to roll back.

## Expand / contract (parallel change)

Example: rename `users.fullname` to `users.display_name` on a busy table. A direct `RENAME COLUMN` is instant in Postgres, but the old app version still queries `fullname` and breaks. So:

{/* diagram source: /diagrams/diagram-104.mmd */}
![diagram](/diagrams/diagram-104.svg)

Each step is independently deployable and reversible up to the final `DROP`. The same pattern applies to splitting a table, changing a type, moving data to a new service, or changing an enum.

Alternative for the dual-write step: a trigger that copies `fullname` into `display_name` on write, so app code changes once.

## Postgres DDL and locks

Most `ALTER TABLE` forms take an **`ACCESS EXCLUSIVE`** lock: nothing can read or write the table until it's done. The question is whether it's held for milliseconds (catalog-only change) or for a full table rewrite.

| Operation | Lock | Cost | Safe pattern |
|---|---|---|---|
| `ADD COLUMN` nullable, no default | ACCESS EXCLUSIVE | instant (catalog only) | Safe with `lock_timeout` |
| `ADD COLUMN ... DEFAULT <constant>` | ACCESS EXCLUSIVE | instant since **PG 11** (non-volatile default stored in catalog) | Safe. A volatile default like `random()` or `clock_timestamp()` still rewrites |
| `ADD COLUMN ... GENERATED ... STORED` | ACCESS EXCLUSIVE | full rewrite | Avoid on big tables, or use a virtual generated column (PG 18) |
| `ALTER COLUMN TYPE` | ACCESS EXCLUSIVE | usually **full rewrite** + index rebuilds (some binary-compatible changes like `varchar(50)` → `varchar(100)` or `text` are catalog-only) | Expand/contract with a new column |
| `DROP COLUMN` | ACCESS EXCLUSIVE | instant (marks column dropped) | Ensure no code references it first |
| `RENAME COLUMN / TABLE` | ACCESS EXCLUSIVE | instant | Breaks old code: use expand/contract or a view |
| `CREATE INDEX` | SHARE (blocks writes) | full build | Use `CREATE INDEX CONCURRENTLY` |
| `CREATE INDEX CONCURRENTLY` | SHARE UPDATE EXCLUSIVE | slower, two scans, can't run in a transaction | Check for `INVALID` index on failure and drop/retry |
| `ADD FOREIGN KEY` | SHARE ROW EXCLUSIVE on both tables + full validation scan | long | `ADD CONSTRAINT ... NOT VALID`, then `VALIDATE CONSTRAINT` |
| `ADD CHECK` | ACCESS EXCLUSIVE + scan | long | `NOT VALID` then `VALIDATE` (SHARE UPDATE EXCLUSIVE, writes continue) |
| `SET NOT NULL` | ACCESS EXCLUSIVE + full scan | long | PG 12+: add validated `CHECK (col IS NOT NULL)` first, then `SET NOT NULL` skips the scan |
| `ADD PRIMARY KEY / UNIQUE` | ACCESS EXCLUSIVE + index build | long | `CREATE UNIQUE INDEX CONCURRENTLY`, then `ADD CONSTRAINT ... USING INDEX` |

Safe recipes:

```sql
-- Foreign key without blocking for a full scan
ALTER TABLE orders ADD CONSTRAINT orders_customer_fk
  FOREIGN KEY (customer_id) REFERENCES customer(id) NOT VALID;   -- quick
ALTER TABLE orders VALIDATE CONSTRAINT orders_customer_fk;        -- scans, but allows reads/writes

-- NOT NULL without a long exclusive lock (PG 12+)
ALTER TABLE users ADD CONSTRAINT users_display_name_nn
  CHECK (display_name IS NOT NULL) NOT VALID;
ALTER TABLE users VALIDATE CONSTRAINT users_display_name_nn;
ALTER TABLE users ALTER COLUMN display_name SET NOT NULL;         -- uses the CHECK, no scan
ALTER TABLE users DROP CONSTRAINT users_display_name_nn;

-- Unique constraint backed by a concurrently-built index
CREATE UNIQUE INDEX CONCURRENTLY users_email_uq ON users (email);
ALTER TABLE users ADD CONSTRAINT users_email_uq UNIQUE USING INDEX users_email_uq;
```

### The lock queue problem and `lock_timeout`

Even an "instant" `ALTER TABLE` must **wait** for its ACCESS EXCLUSIVE lock. If a long-running query holds a conflicting lock, the ALTER waits in the queue, and **every new query on that table queues behind the ALTER**. A 1 ms migration can cause a multi-minute outage.

{/* diagram source: /diagrams/diagram-105.mmd */}
![diagram](/diagrams/diagram-105.svg)

Fix: fail fast and retry.

```sql
SET lock_timeout = '2s';          -- give up if the lock isn't granted quickly
SET statement_timeout = '30s';
ALTER TABLE users ADD COLUMN display_name text;
-- on "canceling statement due to lock timeout": sleep with backoff, retry N times
```

Most migration frameworks let you set these per migration; `strong_migrations`, `django-pg-zero-downtime-migrations` and similar libraries add them automatically. Also kill or avoid long transactions during migration windows (`idle_in_transaction_session_timeout`). See [Locking & Concurrency Control](/Software%20Engineering/Databases/Core%20Concepts/Locking%20%26%20Concurrency%20Control).

### Batched backfills

One `UPDATE users SET display_name = fullname;` on 200M rows holds row locks for the whole run, generates a huge WAL spike (replica lag), and bloats the table. Do it in batches:

```sql
-- run repeatedly (from a job/loop) until it updates 0 rows
WITH batch AS (
  SELECT id FROM users
  WHERE display_name IS NULL
  ORDER BY id
  LIMIT 5000
  FOR UPDATE SKIP LOCKED
)
UPDATE users u
SET display_name = u.fullname
FROM batch
WHERE u.id = batch.id;
```

Or key-range batches (`WHERE id >= $start AND id < $start + 10000`) for a predictable cursor. Commit per batch, sleep between batches, watch replication lag and autovacuum, and make the job idempotent and resumable.

## MySQL: online schema change

- **MySQL 8.0+ `ALGORITHM=INSTANT`**: `ADD COLUMN` (any position since 8.0.29), `DROP COLUMN` (8.0.29+), rename column, and some default changes are metadata-only. Specify `ALGORITHM=INSTANT` explicitly so MySQL errors instead of silently copying.
- **`ALGORITHM=INPLACE, LOCK=NONE`**: online for many index operations, but still rebuilds and can cause replica lag.
- **gh-ost** (GitHub): creates a shadow table, copies rows in chunks, tails the **binlog** to apply ongoing changes, then atomically swaps tables. Triggerless, throttleable, pausable.
- **pt-online-schema-change** (Percona Toolkit): same shadow-table idea using **triggers** to capture changes. Simpler setup, more write overhead.

```bash
gh-ost --host=db1 --database=shop --table=users \
  --alter="ADD COLUMN display_name VARCHAR(255) NULL" \
  --max-load=Threads_running=25 --chunk-size=1000 --execute
```

## MongoDB: schema versioning pattern

No DDL, but the schema still evolves. The **schema versioning pattern**: add `schemaVersion` to each document, make the app read all versions, write only the newest, and upgrade old documents lazily on read/write or with a background job.

```js
// v1: { _id, name: "Ann Lee" }
// v2: { _id, schemaVersion: 2, firstName: "Ann", lastName: "Lee" }
function normalize(doc) {
  if ((doc.schemaVersion ?? 1) === 1) {
    const [firstName, ...rest] = doc.name.split(" ");
    return { ...doc, schemaVersion: 2, firstName, lastName: rest.join(" ") };
  }
  return doc;
}
```

Use `$jsonSchema` validators with `validationLevel: "moderate"` so existing non-conforming documents are not rejected on unrelated updates. Indexes still need building (rolling index builds on replica sets).

## Deploy ordering with application code

| Change | Order |
|---|---|
| Add column / table | Migrate **before** deploying code that uses it |
| Remove column / table | Deploy code that stops using it **first**, then migrate (for ORMs, also remove it from the model, e.g. Rails `ignored_columns`) |
| Rename / change type | Expand/contract across several deploys |
| Add NOT NULL | Backfill + code always writes it, then add the constraint |
| Add constraint that current code might violate | Fix code first, then `NOT VALID` + `VALIDATE` |

- **Backward compatible** migration: old code still works after it runs (needed because migrations usually run before or during rollout).
- **Forward compatible** code: new code tolerates the old schema (needed if you might roll the code back).
- **Feature flags** decouple deploy from release: ship dual-write code behind a flag, flip reads to the new column gradually, and switch back instantly if something is wrong.

<Warning>
**Common pitfalls**

- Running migrations automatically on app startup across many pods: races, and a failure blocks every instance from booting. Run them as a single deploy step/job with a lock.
- `CREATE INDEX CONCURRENTLY` inside a transaction-wrapped migration (it errors); many tools need a "no transaction" flag.
- A failed concurrent index build leaves an `INVALID` index that still slows writes. Check `pg_index.indisvalid`.
- ORM default values or "change column" helpers that silently trigger table rewrites.
- Backfilling in one statement and taking down replicas with WAL lag.
- Dropping a column while an old version (or a cron job, BI tool, other service) still selects it. `SELECT *` in prepared statements can also break.
- Editing a migration that already ran somewhere. Checksums fail (Flyway) or environments drift. Always add a new migration.
- Testing only on an empty dev database where every ALTER is instant.
</Warning>

## Testing and CI

- **Test on production-sized data**: restore a recent (anonymized) snapshot to staging and time each migration. Note which statements rewrite or scan.
- **Lint migrations in CI**: **squawk** (Postgres) flags unsafe patterns such as non-concurrent index creation, adding a NOT NULL column, or changing types. Also `strong_migrations` (Rails), Atlas lint, `django-migration-linter`.
- **Check drift**: compare the migrated schema with production (`pg_dump --schema-only` diff, Atlas `schema diff`).
- **Run migrations up from zero in CI** so a fresh environment always builds.
- **Show the SQL in review**: for ORM-generated migrations, paste the SQL (`sqlmigrate`, `prisma migrate diff`) into the PR.

## Rollback strategy

- Prefer **roll forward**: a new migration or a code revert, both of which are safe when changes are backward compatible.
- Destructive steps (`DROP`, type changes that lose precision) are the point of no return. Delay them days after the expand step, and take a backup or keep the data in a renamed/archived table first ([Database Backup](/Software%20Engineering/Databases/Database%20Round/Database%20Backup)).
- For large data transformations, write to a new table/column and keep the old one until verified.
- Know your restore time. If a bad migration corrupts data, point-in-time recovery (PITR) to a new instance and copying rows back is often the real rollback.

## Senior perspective

- **Classify every migration by risk** in review: catalog-only, scan, rewrite, or data backfill. Ask: which lock, for how long, on how big a table, at what time of day?
- **Budget time for contract steps.** Teams often expand and never contract, leaving dead columns and dual writes forever. Track cleanup as real work.
- **Large-scale changes are projects**: changing a PK from `int` to `bigint` on a multi-billion-row table (a classic when IDs approach 2^31) needs a new column, trigger-based sync, batched backfill, concurrent index, and a short swap window. Monitor ID headroom before it's urgent.
- **Many databases, one migration**: with schema-per-tenant or sharded setups, migrations run N times; you need orchestration, idempotency and partial-failure handling.
- **Replicas and CDC consumers** see DDL too. Logical replication does not replicate DDL in Postgres; Debezium and downstream warehouses must handle the new columns.
- **Upgrades are migrations too**: major-version upgrades (pg_upgrade, blue/green, logical replication) need rehearsals, post-upgrade `ANALYZE`, and extension compatibility checks.

<Info>
**🆕 2025–2026**

- **PostgreSQL 18** (released 2025-09-25):
  - `pg_upgrade` now **preserves optimizer statistics** across major-version upgrades (extended statistics excepted; `--no-statistics` disables it), so the painful post-upgrade window of bad plans before `ANALYZE` finishes is largely gone.
  - New `pg_upgrade --swap` mode swaps data directories instead of copying, cloning or linking files; the docs call it potentially the fastest mode.
  - `ALTER TABLE` can set **`NOT VALID` on `NOT NULL` constraints**, so a NOT NULL can be added without an immediate full-table scan and validated later (verified in the PG 18 release notes).
  - **Virtual generated columns** (now the default kind) can be added without rewriting the table.
- **MySQL 8.0 reached end of life on 2026-04-30.** Upgrade to **8.4 LTS**, or to **9.7 LTS** (released 2026-04-21). Rehearse the upgrade like any other migration.
</Info>

## Interview questions

<Accordion title="How do you rename a column with zero downtime?">
Expand/contract: add the new column, deploy code that writes both, backfill in batches, deploy code that reads the new column, deploy code that writes only the new column, then drop the old column. A plain RENAME is instant in Postgres but breaks the old app version still running during the deploy.
</Accordion>

<Accordion title="Why can a 1 ms ALTER TABLE cause an outage?">
It needs an ACCESS EXCLUSIVE lock. If a long query holds a conflicting lock, the ALTER waits, and all new queries on the table queue behind it. Set `lock_timeout` (a few seconds), retry with backoff, and avoid long transactions during migrations.
</Accordion>

<Accordion title="How do you add a NOT NULL constraint or foreign key to a large Postgres table safely?">
Add the FK or a `CHECK (col IS NOT NULL)` with `NOT VALID` (fast, brief lock), then `VALIDATE CONSTRAINT` which scans without blocking writes. For NOT NULL, PG 12+ then lets `SET NOT NULL` use the validated CHECK and skip the scan; drop the CHECK afterwards. PG 18 also allows NOT NULL itself to be NOT VALID.
</Accordion>

<Accordion title="Versioned vs declarative migrations?">
Versioned: you write ordered change scripts; explicit and reviewable for locks. Declarative: you declare the desired schema and a tool generates the diff; less boilerplate, but generated plans must be reviewed (renames can become drop + add).
</Accordion>

<Accordion title="Do you write down migrations?">
Often not for production. After new data is written, a down migration can lose it, and they are rarely tested. Prefer backward-compatible changes and roll forward; delay destructive steps and keep backups.
</Accordion>

<Accordion title="How do gh-ost and pt-online-schema-change work?">
Both build a shadow copy of the table with the new schema, copy rows in chunks, keep it in sync with ongoing writes (gh-ost by tailing the binlog, pt-osc via triggers), then atomically swap table names. They can throttle on load or replica lag.
</Accordion>

<Accordion title="Should the app run migrations on startup?">
Usually not in multi-instance deployments: instances race, a slow migration delays every boot, and failures take down all pods. Run migrations as a dedicated deploy step with a single runner and a lock, before (for additive changes) or after (for removals) the code rollout.
</Accordion>

## Related
- [Databases Index](/Software%20Engineering/Databases/Databases%20Index)
- [Data Modeling & Normalization](/Software%20Engineering/Databases/Core%20Concepts/Data%20Modeling%20%26%20Normalization) · [Creating and Manipulating Tables](/Software%20Engineering/Databases/SQL%20ANSI/Creating%20and%20Manipulating%20Tables)
- [Locking & Concurrency Control](/Software%20Engineering/Databases/Core%20Concepts/Locking%20%26%20Concurrency%20Control) · [Database Backup](/Software%20Engineering/Databases/Database%20Round/Database%20Backup)
- [Vector Databases & pgvector](/Software%20Engineering/Databases/Core%20Concepts/Vector%20Databases%20%26%20pgvector) (re-embedding is a migration too) · [MongoDB](/Software%20Engineering/Databases/MongoDB/MongoDB)

## Sources
- [PostgreSQL docs: ALTER TABLE](https://www.postgresql.org/docs/current/sql-altertable.html)
- [PostgreSQL docs: Explicit locking](https://www.postgresql.org/docs/current/explicit-locking.html)
- [PostgreSQL docs: CREATE INDEX (CONCURRENTLY)](https://www.postgresql.org/docs/current/sql-createindex.html)
- [PostgreSQL 18 release notes](https://www.postgresql.org/docs/18/release-18.html)
- [PostgreSQL 18 released (2025-09-25)](https://www.postgresql.org/about/news/postgresql-18-released-3142/)
- [endoflife.date: MySQL](https://endoflife.date/mysql)
- [MySQL docs: Online DDL operations](https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-operations.html)
- [gh-ost (GitHub)](https://github.com/github/gh-ost)
- [Percona: pt-online-schema-change](https://docs.percona.com/percona-toolkit/pt-online-schema-change.html)
- [squawk: linter for Postgres migrations](https://squawkhq.com/)
- [strong_migrations (Rails)](https://github.com/ankane/strong_migrations)
- [Martin Fowler: ParallelChange](https://martinfowler.com/bliki/ParallelChange.html)
- [MongoDB blog: Building with Patterns, the Schema Versioning Pattern](https://www.mongodb.com/blog/post/building-with-patterns-the-schema-versioning-pattern)
- [Flyway docs](https://documentation.red-gate.com/flyway) · [Liquibase docs](https://docs.liquibase.com/) · [Alembic docs](https://alembic.sqlalchemy.org/) · [Atlas docs](https://atlasgo.io/docs) · [sqitch](https://sqitch.org/)
