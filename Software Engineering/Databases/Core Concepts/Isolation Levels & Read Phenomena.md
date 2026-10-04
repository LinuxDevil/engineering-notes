---
title: "Isolation Levels & Read Phenomena"
tags: [databases, databases/transactions, databases/postgres]
aliases: [Isolation Levels, Read Phenomena, Dirty Read, Phantom Read, Non-repeatable Read, Write Skew, Lost Update, Snapshot Isolation, SSI]
updated: 2026-09-27
---
# Isolation Levels & Read Phenomena

<Note>
**TL;DR**

Isolation levels trade correctness for concurrency. The SQL standard defines four levels by which **read phenomena** they forbid: dirty reads, non-repeatable reads and phantoms. Real databases differ: PostgreSQL's Repeatable Read is **snapshot isolation** (no phantoms, but **write skew** is possible) and its Serializable is **SSI** (true serializability with retries); MySQL InnoDB defaults to Repeatable Read with next-key locks; SQL Server and Oracle have their own twists. Know the anomalies, know your database's default (usually Read Committed), and choose per transaction.
</Note>

## The idea

Running transactions one at a time would be perfectly correct but slow. Isolation levels let transactions overlap and specify which kinds of interference you'll tolerate. The weaker the level, the more concurrency and the more anomalies.

## Read phenomena

Examples use `accounts(id, balance)` and two concurrent transactions T1 and T2.

### Dirty read

Reading data another transaction has written but **not committed** (and might roll back).

```sql
-- T2                                  -- T1
BEGIN;
UPDATE accounts SET balance = 0
 WHERE id = 1;                         BEGIN;
                                       SELECT balance FROM accounts WHERE id = 1;
                                       -- dirty read: sees 0
ROLLBACK;                              -- T1 acted on a value that never existed
```

### Non-repeatable read (fuzzy read)

Within one transaction you read the **same row twice** and get different values, because another transaction committed an update in between.

```sql
-- T1                                        -- T2
BEGIN;
SELECT balance FROM accounts WHERE id = 1;   -- 100
                                             UPDATE accounts SET balance = 50 WHERE id = 1; COMMIT;
SELECT balance FROM accounts WHERE id = 1;   -- 50 at Read Committed
COMMIT;
```

### Phantom read

A **range query** returns a different **set** of rows the second time, because another transaction inserted (or deleted) rows matching the condition.

```sql
-- T1                                                  -- T2
BEGIN;
SELECT count(*) FROM orders WHERE total > 100;         -- 5
                                                       INSERT INTO orders (total) VALUES (500); COMMIT;
SELECT count(*) FROM orders WHERE total > 100;         -- 6: a phantom row appeared
```

### Lost update

Two transactions read the same value, both compute a new value from it, both write. The first write is overwritten.

```sql
-- T1: SELECT balance → 100        -- T2: SELECT balance → 100
-- T1: UPDATE ... SET balance = 110 (deposit 10)
-- T2: UPDATE ... SET balance = 120 (deposit 20)   -- T1's deposit is lost; should be 130
```

Fixes: atomic update (`SET balance = balance + 10`), `SELECT ... FOR UPDATE`, optimistic version check, or a level that detects it (Postgres Repeatable Read/Serializable raise a serialization error).

### Write skew

Two transactions read an **overlapping set of rows**, each checks an invariant, then each writes to **different** rows. Individually fine; together they break the invariant. No row was written by both, so row locks and snapshot isolation don't catch it.

Classic example: at least one doctor must stay on call.

```sql
-- Both Alice and Bob are on call (2 rows). Each runs, concurrently:
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT count(*) FROM doctors WHERE on_call AND shift_id = 1;   -- both see 2
UPDATE doctors SET on_call = false WHERE name = 'Alice';       -- Bob's tx updates 'Bob'
COMMIT;                                                        -- both commit → 0 on call
```

At `SERIALIZABLE` in Postgres, one of them fails with `ERROR: could not serialize access due to read/write dependencies among transactions` (SQLSTATE `40001`) and must retry. Alternatives: `SELECT ... FOR UPDATE` on the rows you check, or model the invariant as a constraint.

### Read skew and others

- **Read skew**: you see part of the database before and part after another transaction's commit (e.g. reading two accounts during a transfer and seeing the total change). Snapshot isolation prevents it.
- **Serialization anomaly**: any outcome that no serial order could produce (write skew is the common case).

## The SQL standard levels

| Level | Dirty read | Non-repeatable read | Phantom |
|---|---|---|---|
| Read Uncommitted | possible | possible | possible |
| Read Committed | ❌ | possible | possible |
| Repeatable Read | ❌ | ❌ | possible |
| Serializable | ❌ | ❌ | ❌ (and no anomalies at all) |

The user's original summary, kept and corrected:

- **Read Uncommitted**: no isolation; you can see uncommitted changes.
- **Read Committed** (the default in Postgres, Oracle, SQL Server): each **statement** sees data committed before that statement started.
- **Repeatable Read**: rows you've read stay unchanged for the rest of the transaction. In Postgres this is implemented as a **snapshot** taken at the transaction's first statement.
- **Snapshot** (a separate level in SQL Server; what Postgres calls Repeatable Read): each query sees only changes committed before the transaction started.
- **Serializable**: the result equals some serial execution. It doesn't mean transactions literally run one at a time; they run concurrently and the DB prevents or aborts anything that would break serial equivalence.

The standard's definitions were criticised (Berenson et al., 1995) for missing anomalies like lost update and write skew, and for describing levels in terms of phenomena that locking implementations produce. That's why vendors' "same name" levels behave differently.

## What databases actually do

| | PostgreSQL | MySQL InnoDB | SQL Server | Oracle |
|---|---|---|---|---|
| Default | Read Committed | **Repeatable Read** | Read Committed (locking) | Read Committed |
| Read Uncommitted | behaves as Read Committed | real dirty reads | real dirty reads (`NOLOCK`) | not offered |
| Repeatable Read | snapshot isolation: no phantoms; lost updates detected (error); **write skew possible** | consistent snapshot for plain reads; locking reads/updates see latest data and use next-key (gap) locks; lost updates **possible** with plain `SELECT` then `UPDATE` | lock-based, phantoms possible | not offered |
| Snapshot | (= Repeatable Read) | n/a | `SNAPSHOT` level (needs `ALLOW_SNAPSHOT_ISOLATION`) | n/a |
| Serializable | **SSI**: optimistic, detects dangerous dependency cycles, aborts with `40001` | plain `SELECT`s become `FOR SHARE` locking reads (with autocommit off) | range locks (pessimistic) | actually snapshot isolation (write skew possible) |

SQL Server also has `READ_COMMITTED_SNAPSHOT ON`, which turns its Read Committed into a versioned (MVCC-like) read so readers don't block writers.

## How isolation is implemented

- **Pessimistic (locks)**: row, page and table locks to prevent conflicts. Repeatable Read by locking every row read is expensive when you read many rows. See [Locking & Concurrency Control](/Software%20Engineering/Databases/Core%20Concepts/Locking%20%26%20Concurrency%20Control).
- **Optimistic**: no locks; track what changed and fail the transaction at commit if there was a conflict. Postgres Serializable (SSI).
- **MVCC snapshots**: each transaction reads a consistent snapshot of committed versions; readers never block writers and vice versa. See [MVCC & Vacuum](/Software%20Engineering/Databases/Core%20Concepts/MVCC%20%26%20Vacuum).
- You can get serializable-like behaviour at a lower level pessimistically with **`SELECT ... FOR UPDATE`** on the rows your decision depends on.

## Using it

```sql
-- Postgres / standard
BEGIN ISOLATION LEVEL SERIALIZABLE;
-- or
BEGIN; SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

SHOW transaction_isolation;               -- Postgres
SELECT @@transaction_isolation;           -- MySQL 8
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;  -- MySQL
```

Read-only reporting in Postgres without blocking or being aborted:

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;
-- waits for a "safe" snapshot, then can never fail with a serialization error
```

## Trade-offs

| Level | Concurrency | Anomalies | Cost |
|---|---|---|---|
| Read Committed | highest | non-repeatable reads, phantoms, lost updates, write skew | cheapest, app must lock where needed |
| Repeatable Read / Snapshot | high | write skew (Postgres), lost updates (InnoDB plain reads) | retries on update conflicts (Postgres) |
| Serializable | lower | none | retries (SSI) or blocking (lock-based), extra bookkeeping |

## Pitfalls

<Warning>
**Common pitfalls**

- **Assuming "Repeatable Read" means the same thing everywhere.** Postgres RR prevents phantoms; MySQL RR can show different results to plain vs locking reads within one transaction.
- **Using Serializable without retry logic.** SSI will abort transactions under contention by design. Every serializable transaction must be retryable.
- **Check-then-insert at Read Committed** ("if no user with this email, insert") races. Use a `UNIQUE` constraint and `INSERT ... ON CONFLICT`.
- **Reading from replicas** removes guarantees: a replica is a lagging snapshot, so "read your own writes" breaks. See [Replication & 2PC 3PC](/Software%20Engineering/Databases/Database%20Round/Replication%20%26%202PC%203PC).
- **Mixing levels**: Postgres SSI only protects transactions that all run at Serializable.
</Warning>

## Senior perspective

- Default to Read Committed plus targeted fixes (atomic updates, `FOR UPDATE`, unique constraints), and use Serializable for invariant-heavy workflows where reasoning about races is too hard. Measure the abort rate.
- Make invariants **declarative** where you can: `UNIQUE`, `CHECK`, exclusion constraints (`EXCLUDE USING gist` for non-overlapping bookings), and in PG 18 temporal `WITHOUT OVERLAPS` keys. The database enforces them regardless of isolation level.
- Know that ORMs often wrap each request in a transaction at the default level; lost updates in "edit form" flows are common. Use version columns.
- For money, consider an append-only ledger (insert movements, derive balances) which sidesteps many update anomalies.
- Review question: "What invariant does this code protect, and what happens if two requests run it at the same time?"

<Info>
**🆕 2025–2026**

- **PostgreSQL 18 (2025-09-25)** added temporal constraints: `PRIMARY KEY`/`UNIQUE ... WITHOUT OVERLAPS` and `FOREIGN KEY ... PERIOD`, a declarative way to forbid overlapping ranges instead of relying on isolation.
- **PostgreSQL 19 (beta, GA expected ~October 2026)** adds a `WAIT FOR` command so a session can wait until a standby has replayed up to a given WAL position, enabling read-your-writes against replicas. Still beta as of Beta 4 (2026-09-24).
</Info>

## Interview questions

<Accordion title="Define dirty read, non-repeatable read and phantom read.">
Dirty: reading uncommitted data. Non-repeatable: re-reading the same row in one transaction returns a different value because another transaction committed an update. Phantom: re-running a range query returns different rows because another transaction inserted or deleted matching rows.
</Accordion>

<Accordion title="What is write skew and which isolation level prevents it?">
Two transactions read overlapping data, check an invariant, then write disjoint rows, jointly breaking the invariant. Snapshot isolation (Postgres Repeatable Read, Oracle "Serializable") allows it. True serializability (Postgres SSI, lock-based serializable) prevents it, as does explicitly locking the rows read.
</Accordion>

<Accordion title="What is the default isolation level in PostgreSQL and MySQL?">
PostgreSQL: Read Committed. MySQL InnoDB: Repeatable Read.
</Accordion>

<Accordion title="How does PostgreSQL implement Serializable?">
Serializable Snapshot Isolation: transactions run on snapshots without extra blocking; the system tracks read/write dependencies (using predicate "SIRead" locks that never block) and aborts a transaction with SQLSTATE 40001 when a cycle that could produce a non-serializable result is detected.
</Accordion>

<Accordion title="How do you prevent lost updates at Read Committed?">
Use an atomic update expression, `SELECT ... FOR UPDATE` before modifying, or optimistic concurrency with a version column and a conditional `UPDATE ... WHERE version = ?`.
</Accordion>

<Accordion title="Why might Postgres Repeatable Read throw an error on UPDATE?">
If a row you're updating was changed by a transaction that committed after your snapshot, Postgres can't apply your update on top of a version you never saw, so it raises "could not serialize access due to concurrent update". The application should retry.
</Accordion>

## Related
- [Databases Index](/Software%20Engineering/Databases/Databases%20Index) · [Database Round](/Software%20Engineering/Databases/Database%20Round/Database%20Round)
- [Transactions & ACID](/Software%20Engineering/Databases/Database%20Round/Transactions%20%26%20ACID) · [Locking & Concurrency Control](/Software%20Engineering/Databases/Core%20Concepts/Locking%20%26%20Concurrency%20Control) · [MVCC & Vacuum](/Software%20Engineering/Databases/Core%20Concepts/MVCC%20%26%20Vacuum)
- [Managing Transaction Processing](/Software%20Engineering/Databases/SQL%20ANSI/Managing%20Transaction%20Processing) · [Replication & 2PC 3PC](/Software%20Engineering/Databases/Database%20Round/Replication%20%26%202PC%203PC) · [CAP Theorem](/Software%20Engineering/Databases/Database%20Round/CAP%20Theorem)

## Sources
- PostgreSQL docs, Transaction Isolation: https://www.postgresql.org/docs/current/transaction-iso.html
- MySQL 8.4 docs, Transaction Isolation Levels: https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html
- SQL Server docs, SET TRANSACTION ISOLATION LEVEL: https://learn.microsoft.com/en-us/sql/t-sql/statements/set-transaction-isolation-level-transact-sql
- Berenson et al., *A Critique of ANSI SQL Isolation Levels* (1995): https://www.microsoft.com/en-us/research/publication/a-critique-of-ansi-sql-isolation-levels/
- Ports & Grittner, *Serializable Snapshot Isolation in PostgreSQL* (VLDB 2012): https://drkp.net/papers/ssi-vldb12.pdf
- PostgreSQL 18 release: https://www.postgresql.org/about/news/postgresql-18-released-3142/
- PostgreSQL 19 Beta 1 / Beta 4: https://www.postgresql.org/about/news/postgresql-19-beta-1-released-3313/ · https://www.postgresql.org/about/news/postgresql-19-beta-4-released-3386/
- Martin Kleppmann, *Designing Data-Intensive Applications*, ch. 7
