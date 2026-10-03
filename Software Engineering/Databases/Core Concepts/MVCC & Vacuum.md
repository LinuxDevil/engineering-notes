---
tags: [databases, databases/internals, databases/postgres]
aliases: [MVCC, Multi-Version Concurrency Control, Vacuum, VACUUM, Autovacuum, Bloat, Vacuuming in SQL, Dead Tuples, Buffer Pool, XID Wraparound]
updated: 2026-09-27
---
# MVCC & Vacuum

> [!tldr] TL;DR
> **MVCC** keeps several versions of each row so readers see a consistent snapshot while writers create new versions: readers never block writers and writers never block readers. PostgreSQL stores old versions **in the table itself** (an `UPDATE` writes a new tuple and marks the old one dead), so something must clean up: **VACUUM** marks dead tuples' space reusable, updates the visibility map and **freezes** old rows to prevent transaction-ID wraparound. **Autovacuum** does this automatically; when it can't keep up (long transactions, bad settings, huge tables) you get **bloat**. `VACUUM FULL` reclaims disk space but locks the table; `pg_repack` (and `REPACK` in PG 19 beta) do it online.

## The idea

Without MVCC, a reader must lock a row so nobody changes it mid-read, and a writer must wait for readers. With MVCC, a transaction that wants to **read** a record sees the most recent version **committed before its snapshot was taken**, and a transaction that **updates** a record creates a **new version**. Others keep seeing the old version until the new one commits.

The cost: every version must be stored and managed, especially with frequent updates or long-lived transactions, and the database must periodically **garbage collect** versions nobody can see anymore. In Postgres that's **vacuuming**.

### Two ways to keep old versions

| Design | Where old versions live | Readers of old data | Cleanup | Examples |
|---|---|---|---|---|
| **Append in the heap** | New tuple written into the table; old tuple stays in place, marked dead | Read the original table pages | `VACUUM` removes dead tuples | PostgreSQL |
| **Undo log / version store** | Row updated in place; previous version copied to an undo segment | Reconstruct old version from undo | Purge thread trims undo | MySQL InnoDB, Oracle, SQL Server (tempdb version store with snapshot isolation / RCSI) |

Databases with MVCC: PostgreSQL, MySQL InnoDB, Oracle, SQLite (WAL mode gives snapshot reads), SQL Server when Snapshot Isolation or Read Committed Snapshot is on. Without it: MySQL MyISAM (table locks), SQL Server by default (lock-based Read Committed).

> The user's original note said MVCC databases "read the original table" and non-MVCC ones "read from an undo table". More precisely: Postgres readers find old versions in the heap; InnoDB and Oracle are MVCC too but rebuild old versions from **undo** logs.

## How Postgres MVCC works

Every tuple has hidden system columns:

| Column | Meaning |
|---|---|
| `xmin` | ID of the transaction that created this version ("birth") |
| `xmax` | ID of the transaction that deleted/updated it ("death"), or 0; also used for row locks |
| `ctid` | Physical location `(page, item)` of this version |

```sql
CREATE TABLE products (id int PRIMARY KEY, name text);
INSERT INTO products VALUES (1, 'Cake');
SELECT ctid, xmin, xmax, * FROM products WHERE id = 1;
--  ctid  | xmin | xmax | id | name
--  (0,1) |  750 |    0 |  1 | Cake

UPDATE products SET name = 'Cupcake' WHERE id = 1;
SELECT ctid, xmin, xmax, * FROM products WHERE id = 1;
--  (0,2) |  751 |    0 |  1 | Cupcake    -- new tuple; (0,1) now has xmax = 751 and is dead once 751 commits
```

(The screenshots of this experiment are in [[Database Internals]].)

A **snapshot** records which transactions had committed when it was taken. A tuple is visible if its `xmin` committed before the snapshot and its `xmax` is empty, aborted, or not yet committed in that snapshot.

Consequences:

- `UPDATE` = insert new version + mark old version dead. Updating one column rewrites the whole row.
- `DELETE` doesn't free space; it sets `xmax`.
- A rolled-back `INSERT` leaves a dead tuple behind.
- Indexes point to tuple versions, so a normal update also inserts new index entries in **every** index... unless it's a **HOT** update.

### HOT (heap-only tuple) updates

If the update changes **no indexed column** and the new version **fits in the same page**, Postgres chains the old item pointer to the new tuple and skips index updates. Leaving free space per page (`ALTER TABLE t SET (fillfactor = 80)`) raises the HOT rate for update-heavy tables. Check `n_tup_hot_upd` vs `n_tup_upd` in `pg_stat_user_tables`. Page layout details: [[Database Pages]].

## VACUUM

VACUUM is the operation that removes old tuples that are **not visible to anyone** anymore, freeing space **inside** pages for reuse.

What plain `VACUUM` does:

1. Scans pages (skipping all-visible ones via the **visibility map**).
2. Removes dead tuples and their index entries; the space becomes reusable by future inserts/updates in that table.
3. Updates the **free space map** and **visibility map** (which also enables **index-only scans**).
4. **Freezes** old tuples so their `xmin` is no longer compared (see wraparound).
5. May truncate empty pages at the **end** of the file.

It takes a SHARE UPDATE EXCLUSIVE lock: reads and writes continue.

**Plain VACUUM usually doesn't shrink the file.** A page full of dead tuples becomes free space inside the file, but the file stays the same size.

**`VACUUM FULL`** rewrites the whole table into a new file, physically removing dead tuples and returning space to the OS, and rebuilds indexes. It takes an **ACCESS EXCLUSIVE** lock (no reads or writes) for the duration and needs extra disk space for the copy. Use it rarely; prefer `pg_repack`.

```sql
VACUUM (VERBOSE, ANALYZE) orders;   -- clean + refresh planner statistics
VACUUM FULL orders;                 -- rewrite; blocks everything
ANALYZE orders;                     -- statistics only
```

### Autovacuum

A background launcher starts workers when a table has enough dead tuples:

```text
vacuum if  n_dead_tup > autovacuum_vacuum_threshold (50)
                       + autovacuum_vacuum_scale_factor (0.2) × reltuples
```

On a 100-million-row table the default waits for 20 million dead rows, which is too late. Tune per table:

```sql
ALTER TABLE events SET (autovacuum_vacuum_scale_factor = 0.01,
                        autovacuum_vacuum_cost_limit = 2000);
```

Also governed by `autovacuum_max_workers`, `autovacuum_naptime`, and cost-based throttling (`autovacuum_vacuum_cost_limit`/`_delay`). Insert-only tables are vacuumed based on inserts too (`autovacuum_vacuum_insert_threshold`, PG 13+) so the visibility map stays current.

### Transaction ID wraparound and freezing

Transaction IDs are 32-bit and compared modulo 2³², so about 2 billion transactions into the "future" looks like the "past". Old tuples must be **frozen** (marked as visible to everyone) before that happens. If freezing falls behind, Postgres warns, then eventually refuses new writes to protect data. Autovacuum runs an **aggressive / anti-wraparound** vacuum when `age(relfrozenxid)` exceeds `autovacuum_freeze_max_age` (200 million by default), even if autovacuum is otherwise disabled.

```sql
SELECT relname, age(relfrozenxid) AS xid_age
FROM pg_class WHERE relkind = 'r' ORDER BY 2 DESC LIMIT 10;
```

## Bloat

**Bloat** is space occupied by dead tuples or empty page space that the table/index keeps holding. Causes:

- Autovacuum can't keep up (update-heavy tables, conservative thresholds, throttling).
- **Something holds back the "xmin horizon"**: long-running or `idle in transaction` sessions, old replication slots, `hot_standby_feedback` from a replica with long queries, forgotten prepared transactions. VACUUM can only remove tuples older than the oldest snapshot still in use.
- Mass `DELETE`/`UPDATE` in one go.

Effects: more pages to scan, worse cache hit rate, bigger backups, bloated indexes.

```sql
-- dead tuples and last autovacuum per table
SELECT relname, n_live_tup, n_dead_tup, last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 10;

-- what's holding back cleanup
SELECT pid, state, xact_start, backend_xmin, left(query, 50)
FROM pg_stat_activity WHERE backend_xmin IS NOT NULL ORDER BY age(backend_xmin) DESC;
```

Fixes: kill/avoid long transactions (`idle_in_transaction_session_timeout`), drop unused replication slots, tune autovacuum per table, batch big deletes, use **partitioning** and drop old partitions instead of deleting ([[SQL Table Partitioning]]), `REINDEX CONCURRENTLY` for index bloat, `pg_repack` for table bloat.

## Buffer pool vs cache (from the original notes)

- **Buffer pool** (Postgres `shared_buffers`): the DB's own memory area that holds **pages** read from disk. Buffer management moves pages between disk and memory and decides which to evict. It's the low-level mechanism for reading and writing data efficiently.
- **Cache**: focuses on keeping **frequently accessed** data in RAM after it has been read, so repeated requests skip disk. In practice, the buffer pool *is* the database's page cache; Postgres also relies on the **OS page cache** (double buffering), which is why `shared_buffers` is typically ~25% of RAM and `effective_cache_size` tells the planner about the rest.
- Dirty pages in the buffer pool are written back by the **checkpointer** and **background writer**; durability comes from the WAL, not from these writes. See [[Database Pages]] and [[Transactions & ACID]].

## Pitfalls

> [!warning] Common pitfalls
> - **Disabling autovacuum** "for performance". You trade a small steady cost for bloat and an eventual forced anti-wraparound vacuum at the worst time.
> - **Running `VACUUM FULL` in production** during traffic: it's an outage for that table.
> - **Long transactions and abandoned replication slots** silently stop cleanup for the whole cluster.
> - **Assuming `DELETE` frees disk space.** It doesn't, until VACUUM makes space reusable, and only `VACUUM FULL`/repack returns it to the OS.
> - **Mass updates** double table size temporarily (every row gets a new version). Batch them.

## Senior perspective

- Monitor: dead tuples, autovacuum run times (`pg_stat_progress_vacuum`), `age(datfrozenxid)`, xmin horizon holders, table/index size growth. Alert on XID age well before 1 billion.
- Design for MVCC: avoid "hot row" counters updated thousands of times per second (use append + aggregate, or shard counters); keep indexes minimal on update-heavy tables to get HOT updates; set `fillfactor`.
- Prefer partition drops over deletes for retention; they create no dead tuples.
- Upgrades matter: PG 17 cut vacuum memory use dramatically, PG 18 added eager freezing and a max dead-tuple threshold, PG 19 adds parallel autovacuum and REPACK.

> [!info]+ 🆕 2025–2026
> - **PostgreSQL 17 (2024-09-26)**: a new internal memory structure for dead-tuple tracking cuts VACUUM memory use by up to 20x and removes the old 1 GB limit, so big tables need fewer index passes.
> - **PostgreSQL 18 (2025-09-25)**:
>   - `autovacuum_vacuum_max_threshold`: a fixed cap on dead tuples that triggers autovacuum, so huge tables are no longer stuck waiting for the 20% scale factor.
>   - `autovacuum_worker_slots`: `autovacuum_max_workers` can be changed without a restart (up to this many slots).
>   - Eager freezing during normal vacuums (`vacuum_max_eager_freeze_failure_rate`) spreads out freeze work and reduces expensive aggressive vacuums.
>   - `vacuum_truncate` as a server-level setting; vacuum timing columns in `pg_stat_all_tables`; asynchronous I/O can speed up vacuum reads.
> - **PostgreSQL 19 (beta; Beta 4 on 2026-09-24, GA expected ~October 2026)**: parallel autovacuum workers (`autovacuum_max_parallel_workers`), a new autovacuum scoring/prioritisation system, and a core **`REPACK`** command with a **`CONCURRENTLY`** option that rewrites a table to remove bloat without blocking reads and writes, the built-in answer to `pg_repack`.

## Interview questions

> [!question]- What is MVCC and what problem does it solve?
> Multi-version concurrency control keeps multiple versions of rows so each transaction reads a consistent snapshot. Readers don't block writers and writers don't block readers, which greatly improves concurrency compared with lock-based reads.

> [!question]- What happens physically when you UPDATE a row in PostgreSQL?
> A new tuple version is written (possibly on another page) with a new `xmin`; the old tuple gets `xmax` set to the updating transaction. Indexes get new entries unless it's a HOT update. The old version becomes dead once no snapshot needs it and VACUUM later reclaims its space.

> [!question]- VACUUM vs VACUUM FULL?
> VACUUM marks dead-tuple space reusable, updates the visibility and free space maps and freezes old rows, without blocking reads/writes; it rarely shrinks files. VACUUM FULL rewrites the table into a compact new file, returns space to the OS, but needs an ACCESS EXCLUSIVE lock and extra disk space.

> [!question]- What causes table bloat and how do you fix it?
> Dead tuples accumulating faster than vacuum reclaims them, usually due to update-heavy workloads with default thresholds or something holding back the xmin horizon (long transactions, stale replication slots). Fix the root cause, tune autovacuum per table, then compact with pg_repack/REPACK CONCURRENTLY or VACUUM FULL in a maintenance window.

> [!question]- What is transaction ID wraparound?
> Postgres XIDs are 32-bit and compared circularly. Rows must be frozen before their XID is about 2 billion transactions old, otherwise they'd appear to be in the future and vanish. Autovacuum forces anti-wraparound vacuums; if it falls too far behind, Postgres stops accepting writes.

> [!question]- How is InnoDB's MVCC different from Postgres's?
> InnoDB updates rows in place and keeps previous versions in undo logs; readers reconstruct old versions from undo and a purge thread cleans up. Postgres writes new versions into the heap and relies on VACUUM. InnoDB has no heap bloat from updates but can suffer long undo history ("history list length") with long transactions.

## Related
- [[Databases Index]] · [[Database Round]]
- [[Transactions & ACID]] · [[Isolation Levels & Read Phenomena]] · [[Locking & Concurrency Control]]
- [[Database Internals]] · [[Database Pages]] · [[Query Optimization & EXPLAIN]] · [[SQL Table Partitioning]]
- [[PostgreSQL 17 to 18 - What's New]]

## Sources
- PostgreSQL docs, Concurrency Control / MVCC: https://www.postgresql.org/docs/current/mvcc-intro.html
- PostgreSQL docs, Routine Vacuuming: https://www.postgresql.org/docs/current/routine-vacuuming.html
- PostgreSQL docs, Automatic Vacuuming settings: https://www.postgresql.org/docs/current/runtime-config-autovacuum.html
- PostgreSQL docs, Heap-Only Tuples: https://www.postgresql.org/docs/current/storage-hot.html
- PostgreSQL 17 release: https://www.postgresql.org/about/news/postgresql-17-released-2936/
- PostgreSQL 18 release notes: https://www.postgresql.org/docs/18/release-18.html
- PostgreSQL 19 Beta 1 / Beta 4: https://www.postgresql.org/about/news/postgresql-19-beta-1-released-3313/ · https://www.postgresql.org/about/news/postgresql-19-beta-4-released-3386/
- MySQL docs, InnoDB Multi-Versioning: https://dev.mysql.com/doc/refman/8.4/en/innodb-multi-versioning.html
- pg_repack: https://reorg.github.io/pg_repack/
