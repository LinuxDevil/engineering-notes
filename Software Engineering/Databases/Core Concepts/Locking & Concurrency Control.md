---
tags: [databases, databases/transactions, databases/postgres]
aliases: [Locks, Database Locks, Two-Phase Locking, 2PL, Deadlocks, Optimistic Locking, Pessimistic Locking, SELECT FOR UPDATE, Advisory Locks]
updated: 2026-09-27
---
# Locking & Concurrency Control

> [!tldr] TL;DR
> Concurrency control keeps the database correct when many transactions touch the same data at once. **Pessimistic** control takes **locks** (shared for reading, exclusive for writing) and makes others wait; **two-phase locking (2PL)** guarantees serializability, and **strict 2PL** (hold locks until commit) is what real databases do. **Optimistic** control lets everyone proceed and aborts on conflict at commit. Postgres combines **MVCC** (readers never block writers) with row locks for writers, eight table-level lock modes, and advisory locks for application-level mutual exclusion.

This note merges the locking parts of the original *Overview* (Silberschatz ch. 18, Concurrency Control), *ACID Overview* and *Transactions & ACID*.

## The idea

**Locking** prevents the inconsistencies that happen when several transactions read and modify shared data at the same time. A **lock** is a flag the database attaches to a data item (row, page, table) that says who may read or write it right now.

Two basic modes:

| Lock | Also called | Allows holder to | Compatible with |
|---|---|---|---|
| **Shared (S)** | read lock | read | other S locks |
| **Exclusive (X)** | write lock | read and write | nothing |

- **Shared**: "I'm reading this; nobody change it until I'm done." Many transactions can hold S on the same item as long as nobody holds X.
- **Exclusive**: "I'm changing this; nobody else touches it." Only one X lock, and no S locks alongside.

```text
          requested →   S      X
held ↓        S         ✅     ❌
              X         ❌     ❌
```

## Two-phase locking (2PL)

2PL is the protocol for acquiring and releasing locks that guarantees **conflict serializability**. Each transaction has two phases that must not overlap:

1. **Growing phase**: the transaction may acquire locks but not release any.
2. **Shrinking phase**: it may release locks but not acquire new ones.

```mermaid
flowchart LR
  A[BEGIN] --> G[Growing: acquire S/X locks] --> P((lock point)) --> S[Shrinking: release locks] --> E[END]
```

**Strict 2PL**: a transaction releases **exclusive** locks only at commit or abort (rigorous 2PL: all locks). This prevents other transactions from reading uncommitted data, which avoids **cascading aborts**. Practically every lock-based DBMS uses strict 2PL.

### The double-booking problem (worked example)

Two users try to book seat 14 at the same time. Both read `isbooked = 0`, both update, both commit: the last commit wins and one user silently loses. The fix is to take an exclusive row lock when reading:

```sql
BEGIN;
SELECT * FROM seats WHERE id = 14 FOR UPDATE;      -- growing phase: X lock on row 14
-- any other transaction doing SELECT ... FOR UPDATE / UPDATE on row 14 now waits
UPDATE seats SET isbooked = 1, name = 'Ali' WHERE id = 14 AND isbooked = 0;
COMMIT;                                            -- shrinking phase: lock released
```

The second transaction unblocks after the first commits. Under Read Committed it re-reads the row, sees `isbooked = 1`, and can report "already booked".

> [!tip] Lock waits and timeouts
> By default a Postgres transaction waits indefinitely for a row lock. You can bound it with `SET lock_timeout = '2s'`, or skip the wait with `FOR UPDATE NOWAIT` (error immediately) or `FOR UPDATE SKIP LOCKED` (ignore locked rows, great for job queues). Oracle offers `FOR UPDATE WAIT n`.

## Deadlocks

A **deadlock** happens when transactions wait for each other in a cycle: T1 holds A and wants B; T2 holds B and wants A. Neither can proceed.

```sql
-- T1                                   -- T2
BEGIN;                                  BEGIN;
UPDATE accounts SET ... WHERE id = 1;   UPDATE accounts SET ... WHERE id = 2;
UPDATE accounts SET ... WHERE id = 2;   UPDATE accounts SET ... WHERE id = 1;
-- waits for T2                         -- waits for T1 → deadlock
```

How databases handle it:

| Strategy | How |
|---|---|
| **Detection** (Postgres, MySQL, SQL Server) | Build a wait-for graph; on a cycle, abort one victim. Postgres checks after `deadlock_timeout` (1s default) and raises SQLSTATE `40P01`. |
| **Prevention** | Acquire locks in a fixed global order (e.g. always lower id first), or lock everything up front. |
| **Avoidance with timestamps** | *wait-die* / *wound-wait*: older transactions get priority, younger ones abort. |
| **Timeouts** | Give up after a lock wait limit. |

App-side rule: **always touch rows in a consistent order** and **retry** on deadlock errors.

## Optimistic vs pessimistic

| | Pessimistic | Optimistic |
|---|---|---|
| Assumption | Conflicts are likely | Conflicts are rare |
| Mechanism | Lock before access (`FOR UPDATE`) | Check at write/commit that nothing changed; abort/retry otherwise |
| Cost | Waiting, deadlocks | Wasted work on retry |
| Good for | Hot rows, short critical sections | Mostly-read data, long user "think time" (forms) |

Optimistic locking in the application with a version column:

```sql
-- read
SELECT id, title, version FROM docs WHERE id = 7;         -- version = 3
-- write only if nobody changed it in between
UPDATE docs SET title = 'New', version = version + 1
WHERE id = 7 AND version = 3;
-- 0 rows updated → someone else won; reload and retry or show a conflict
```

Inside the database, **optimistic concurrency control** (validation-based protocol) runs transactions without locks and validates at commit. Postgres's `SERIALIZABLE` level (Serializable Snapshot Isolation, SSI) works this way: it tracks read/write dependencies and aborts one transaction with `40001` if a dangerous pattern appears.

## Concurrency control without locking (textbook)

- **Timestamp-based protocols**: each transaction gets a timestamp; conflicting operations must happen in timestamp order, otherwise the late transaction is aborted and restarted.
- **Validation-based (optimistic) protocol**: read phase, validation phase, write phase.
- **Multiversion concurrency control (MVCC)**: keep several versions of each item so readers get a consistent snapshot without blocking writers. See [[MVCC & Vacuum]].

## Lock granularity

Granularity is the size of the thing locked: database, table, page, row, or even a field.

| Granularity | Overhead | Concurrency |
|---|---|---|
| Coarse (table) | Few locks to manage, cheap | Low: whole table blocked |
| Fine (row) | Many locks, more memory/CPU | High: only touched rows blocked |

**Multiple-granularity locking** uses *intention* locks (IS, IX, SIX) on the parent (table) so that a transaction wanting a table lock can quickly see that someone holds row locks inside it. SQL Server and InnoDB use intention locks; SQL Server also does **lock escalation** (many row locks become one table lock).

**How locks are implemented**: a **lock manager** keeps a **lock table** (a hash table keyed by the locked item, with a queue of granted and waiting requests). Postgres keeps table-level locks in shared memory, but stores row locks **in the row itself** (the tuple's `xmax` and infomask bits), so it can lock millions of rows without a huge lock table. When a lock is released, waiting transactions at Read Committed or above re-check the latest row version.

## PostgreSQL in practice

### Explicit vs implicit locking

> [!note]+ Implicit locking
> The system acquires and releases locks automatically, hiding the complexity.
> 1. **Row-level**: any `UPDATE` or `DELETE` locks the affected rows until the transaction ends.
> 2. **Table-level**: every command takes a table lock mode appropriate to it. `SELECT` takes ACCESS SHARE; `UPDATE`/`DELETE`/`INSERT` take ROW EXCLUSIVE, which conflicts with things like `ALTER TABLE` but not with each other.
> 3. **Page-level**: short-term share/exclusive locks on buffer pages while reading/writing them (released right after the access, not held to commit).
> 4. **Tuple-level**: an update marks the old row version as locked/deleted by the updating transaction.
> 5. **Shared and exclusive**: shared for reads, exclusive for writes.

> [!note]+ Explicit locking
> You acquire the lock yourself to override default behaviour. Finer control, more responsibility.
> ```sql
> BEGIN;
> LOCK TABLE table_name IN SHARE ROW EXCLUSIVE MODE;
> -- CODE
> COMMIT;   -- table locks are always released at transaction end
> ```

### Table-level lock modes (weakest → strongest)

![[Screenshot_2023-07-23_at_3.25.01_PM.png]]

| Mode | Taken by | Notes |
|---|---|---|
| **ACCESS SHARE** | plain `SELECT` | Only conflicts with ACCESS EXCLUSIVE. An open reading transaction blocks `TRUNCATE`, `DROP`, `VACUUM FULL`. |
| **ROW SHARE** | `SELECT ... FOR UPDATE / NO KEY UPDATE / SHARE / KEY SHARE` | |
| **ROW EXCLUSIVE** | `UPDATE`, `INSERT`, `DELETE`, `MERGE` | DML writers don't block each other at table level. |
| **SHARE UPDATE EXCLUSIVE** | `VACUUM` (not FULL), `ANALYZE`, `CREATE INDEX CONCURRENTLY`, `CREATE STATISTICS`, `COMMENT ON`, `REINDEX CONCURRENTLY`, some `ALTER TABLE` | Protects against concurrent schema changes and concurrent VACUUM runs. |
| **SHARE** | `CREATE INDEX` (non-concurrent) | Blocks data changes; reads OK. |
| **SHARE ROW EXCLUSIVE** | `CREATE TRIGGER`, some `ALTER TABLE` | Like SHARE, but only one session can hold it. |
| **EXCLUSIVE** | `REFRESH MATERIALIZED VIEW CONCURRENTLY` | Only ACCESS SHARE (plain reads) can run alongside. |
| **ACCESS EXCLUSIVE** | `DROP TABLE`, `TRUNCATE`, `VACUUM FULL`, `CLUSTER`, `REINDEX` (non-concurrent), most `ALTER TABLE`, `LOCK TABLE` default | Nothing else can touch the table, not even `SELECT`. |

### Row-level lock modes

![[Screenshot_2023-07-23_at_3.25.29_PM.png]]

| Mode | Blocks | Typical use |
|---|---|---|
| `FOR UPDATE` | all other row locks and writers | You'll modify the row, including its key. Also taken by `DELETE` and by `UPDATE` of columns in a unique index usable by a foreign key. |
| `FOR NO KEY UPDATE` | like FOR UPDATE but **not** `FOR KEY SHARE` | Taken by ordinary `UPDATE`s that don't change key columns, so FK checks from child inserts aren't blocked. |
| `FOR SHARE` | `UPDATE`, `DELETE`, `FOR UPDATE`, `FOR NO KEY UPDATE` | Many readers want the row to stay unchanged. |
| `FOR KEY SHARE` | `DELETE` and key-changing `UPDATE`s only | Taken by foreign-key checks on the parent row. |

```sql
BEGIN;
SELECT * FROM table_name WHERE <condition> FOR UPDATE;
-- CODE
COMMIT;
```

### Advisory locks (application locks)

Locks on an arbitrary number that the database doesn't associate with any row. It's up to the application to use them consistently, for example "only one worker may run the nightly job".

```sql
-- session level: held until unlocked or the session ends
SELECT pg_advisory_lock(42);
-- ... critical section ...
SELECT pg_advisory_unlock(42);

-- transaction level: released automatically at COMMIT/ROLLBACK
BEGIN;
SELECT pg_advisory_xact_lock(42);
-- ... critical section ...
COMMIT;

-- non-blocking: returns true/false
SELECT pg_try_advisory_lock(42);
```

Family: `pg_advisory_lock`, `pg_try_advisory_lock`, `pg_advisory_lock_shared`, `pg_try_advisory_lock_shared`, their `_xact_` variants, and `pg_advisory_unlock`, `pg_advisory_unlock_shared`, `pg_advisory_unlock_all`. Any other session asking for the same key blocks until it's released.

> [!warning] Advisory locks and connection poolers
> Session-level advisory locks are tied to the **server connection**. Behind PgBouncer in transaction pooling mode, your next statement may run on a different server connection, so use `pg_advisory_xact_lock` there. See [[Connection Pooling]].

### Seeing who blocks whom

```sql
SELECT pid, state, wait_event_type, wait_event,
       pg_blocking_pids(pid) AS blocked_by, left(query, 60) AS query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;

SELECT locktype, relation::regclass, mode, granted, pid
FROM pg_locks WHERE NOT granted;
```

## Pitfalls

> [!warning] Common pitfalls
> - **Lock queue pile-up on DDL**: `ALTER TABLE` needs ACCESS EXCLUSIVE. If a long `SELECT` holds ACCESS SHARE, the `ALTER` waits, and **every new query queues behind the ALTER**. Always run migrations with `SET lock_timeout = '3s'` and retry. See [[Database Migrations & Schema Evolution]].
> - **Lost updates at Read Committed**: read-modify-write in the app without `FOR UPDATE` or a version check.
> - **Deadlocks from inconsistent ordering**, including from foreign keys and triggers you forgot about.
> - **Locking too much**: `FOR UPDATE` on a large range, or `LOCK TABLE` for convenience, destroys concurrency.
> - **Long transactions** hold row locks until the end, not until the statement ends.

## Senior perspective

- Prefer **atomic single statements** (`UPDATE stock SET qty = qty - 1 WHERE id = $1 AND qty > 0 RETURNING qty`) over read-then-write. They're shorter and need no explicit lock.
- For job queues in Postgres, `SELECT ... FOR UPDATE SKIP LOCKED LIMIT n` is the standard pattern.
- Measure contention: `pg_stat_activity` wait events, `log_lock_waits = on`, deadlock counts in `pg_stat_database`. PostgreSQL 19 (in beta as of September 2026) adds a `pg_stat_lock` view with per-lock-type statistics.
- In review, ask: which rows/tables does this lock, in what order, for how long, and what happens to other requests meanwhile?

> [!info]+ 🆕 2025–2026
> - **PostgreSQL 19 Beta 1 (2026-06-04)** adds a `pg_stat_lock` view with per-lock-type statistics. PG 19 is not released yet (Beta 4 on 2026-09-24, RC planned for early October 2026), so treat this as upcoming.

## Interview questions

> [!question]- What is two-phase locking and why does it guarantee serializability?
> Each transaction acquires locks in a growing phase and releases them in a shrinking phase, never acquiring after releasing. The moment of the last acquisition (lock point) gives a total order of transactions; any conflicting operations follow that order, so the schedule is conflict-equivalent to the serial order by lock points. Strict 2PL holds write locks to commit to avoid cascading aborts.

> [!question]- How would you prevent double-booking a seat?
> Lock the row while deciding: `SELECT ... FOR UPDATE` then update, or a single conditional `UPDATE seats SET booked = true WHERE id = 14 AND booked = false` and check the row count, or a unique constraint on (event, seat) in a bookings table. Serializable isolation with retries also works.

> [!question]- How do databases deal with deadlocks, and what should the application do?
> They detect cycles in the wait-for graph and abort a victim (Postgres raises 40P01 after `deadlock_timeout`). The app should acquire locks in a consistent order, keep transactions short, and retry the aborted transaction.

> [!question]- Optimistic vs pessimistic locking: when would you use each?
> Pessimistic for hot rows and short critical sections where conflicts are likely. Optimistic (version column, or SERIALIZABLE with retries) when conflicts are rare or the "read then write" spans user think time, since holding DB locks across user interaction is unacceptable.

> [!question]- What's the difference between `FOR UPDATE` and `FOR NO KEY UPDATE` in Postgres?
> Both lock the row exclusively against writers, but `FOR NO KEY UPDATE` doesn't conflict with `FOR KEY SHARE`, which foreign-key checks take. Normal updates that don't change key columns take the weaker lock so inserting child rows referencing the parent isn't blocked.

> [!question]- Why did my `ALTER TABLE` take the whole site down?
> It waited for ACCESS EXCLUSIVE behind a long-running query, and all subsequent queries on the table queued behind it. Use `lock_timeout` and retries, and run DDL at quiet times or with online-safe variants.

> [!question]- What are advisory locks good for?
> Application-defined mutual exclusion using the database as the coordinator: singleton cron jobs, migrations, per-tenant serialization. Use transaction-level variants behind transaction poolers.

## Related
- [[Databases Index]] · [[Database Round]]
- [[Transactions & ACID]] · [[Isolation Levels & Read Phenomena]] · [[MVCC & Vacuum]]
- [[Database Migrations & Schema Evolution]] · [[Connection Pooling]] · [[Replication & 2PC 3PC]]
- Original notes: `_Archive/Overview - Locking & Distributed DB (original)`, `_Archive/ACID Overview (original)`

## Sources
- PostgreSQL docs, Explicit Locking: https://www.postgresql.org/docs/current/explicit-locking.html
- PostgreSQL docs, Advisory lock functions: https://www.postgresql.org/docs/current/functions-admin.html#FUNCTIONS-ADVISORY-LOCKS
- PostgreSQL docs, `SELECT` locking clause: https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE
- PostgreSQL 19 Beta 1 announcement: https://www.postgresql.org/about/news/postgresql-19-beta-1-released-3313/
- Silberschatz, Korth, Sudarshan, *Database System Concepts*, ch. 18 (Concurrency Control): https://www.db-book.com/
- Hussein Nasser, *Fundamentals of Database Engineering*
