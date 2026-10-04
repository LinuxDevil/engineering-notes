---
title: "Managing Transaction Processing"
tags: [databases, databases/sql, databases/postgres]
aliases: [Transactions, COMMIT, ROLLBACK, SAVEPOINT, Autocommit, Transactional DDL]
updated: 2026-09-27
---
# Managing Transaction Processing

<Note>
**TL;DR**

A transaction groups statements so they all succeed (`COMMIT`) or none do (`ROLLBACK`). **Savepoints** let you undo part of a transaction. Most clients run in **autocommit** mode (each statement is its own transaction) until you `BEGIN`. What can be rolled back differs: in **PostgreSQL, DDL is transactional** (`CREATE`/`DROP`/`ALTER` roll back), while MySQL and Oracle implicitly commit on DDL.
</Note>

> Course: SQL in 10 Minutes, Lesson 20 · Prev: [Working with Stored Procedures](/Software%20Engineering/Databases/SQL%20ANSI/Working%20with%20Stored%20Procedures) · Next: [Using Cursors](/Software%20Engineering/Databases/SQL%20ANSI/Using%20Cursors)

## Understanding transaction processing

Transaction processing maintains database integrity by ensuring that batches of SQL operations execute **completely or not at all**.

The book's `Orders` tables are a good example. Orders live in two tables: `Orders` (the order) and `OrderItems` (its items), related by primary keys, and those relate to customer and product tables. Adding an order:
1. Check if the customer is in the database; if not, add them.
2. Retrieve the customer's ID.
3. Add a row to `Orders` for that customer.
4. Retrieve the new order ID.
5. Add one row to `OrderItems` per item, linked to the order (and to `Products` by product ID).

If the database fails halfway (say after step 3), you'd have an order with no items. Transactions prevent that.

**Transaction processing** manages sets of SQL operations that must run as a batch, so the database never contains the results of partial operations. If no error occurs, the whole set is committed. If an error occurs, a rollback restores the database to a known, safe state.

Vocabulary:
1. **Transaction:** a block of SQL statements.
2. **Rollback:** undoing specified SQL statements.
3. **Commit:** writing unsaved SQL statements to the database tables.
4. **Savepoint:** a temporary placeholder in a transaction that you can roll back to, instead of rolling back the whole transaction.

These are the "A" (atomicity) of ACID; see [Transactions & ACID](/Software%20Engineering/Databases/Database%20Round/Transactions%20%26%20ACID).

## Which statements can you roll back?

Transactions manage `INSERT`, `UPDATE` and `DELETE`. Rolling back a `SELECT` is meaningless (it changes nothing).

What about `CREATE` and `DROP`? The book says they can't be rolled back. **That depends on the DBMS** (corrected):

| DBMS | DDL (`CREATE`, `DROP`, `ALTER`, `TRUNCATE`) in a transaction |
|---|---|
| **PostgreSQL** | **Transactional.** `ROLLBACK` undoes it. (Exceptions: `CREATE DATABASE`, `CREATE INDEX CONCURRENTLY`, `VACUUM` can't run inside a transaction block.) |
| SQL Server | Mostly transactional |
| SQLite | Transactional |
| **MySQL / MariaDB** | **Implicit commit** before and after most DDL; can't be rolled back |
| **Oracle** | **Implicit commit**; can't be rolled back |

```sql
-- PostgreSQL: this really does nothing
BEGIN;
DROP TABLE CustCopy;
CREATE TABLE test_ddl (id int);
ROLLBACK;   -- CustCopy is back, test_ddl never existed
```

That's why PostgreSQL migrations can be wrapped in a transaction and fail cleanly, while MySQL migrations can leave the schema half-changed.

With the transaction in place, the order process becomes:
1. Check if the customer exists; if not, add them.
2. Commit the customer information.
3. Retrieve the customer's ID.
4. Add a row to `Orders`.
5. If that fails, roll back.
6. Retrieve the new order ID.
7. Add one row to `OrderItems` per item.
8. If that fails, roll back all the `OrderItems` rows and the `Orders` row.

## Autocommit

By default, most clients and drivers (psql, MySQL client, JDBC, psycopg in autocommit mode) **autocommit**: every statement is its own transaction and is committed immediately. This is the "implicit commit" the book describes.

A transaction starts when you say so:

```sql
-- PostgreSQL, MySQL, SQLite
BEGIN;            -- or START TRANSACTION (standard)
...
COMMIT;           -- or ROLLBACK;
```

```sql
-- SQL Server
BEGIN TRANSACTION
...
COMMIT TRANSACTION
```

Oracle is the exception: it **starts** a transaction implicitly with the first DML statement and waits for `COMMIT`/`ROLLBACK` (SQL*Plus commits on exit by default). Some drivers (psycopg2's default, JDBC with `setAutoCommit(false)`) also open transactions implicitly; forgetting to commit leaves an "idle in transaction" session holding locks.

## Controlling transactions

<Note>
The key to managing transactions is breaking SQL into logical chunks and stating explicitly when data should be rolled back and when not.
</Note>

In the examples below, all SQL between `BEGIN TRANSACTION` and `COMMIT TRANSACTION` must execute entirely or not at all. Other DBMSs use variations. Most don't have an explicit "end transaction": the transaction lasts until something terminates it, usually `COMMIT` or `ROLLBACK`.

## Using `ROLLBACK`

`ROLLBACK` undoes SQL statements:

```sql
BEGIN;
DELETE FROM Orders;
ROLLBACK;
```

**Analysis:** the `DELETE` is performed and then undone. Not a very useful example, but it shows that inside a transaction block, `DELETE` (like `INSERT` and `UPDATE`) is never final until committed.

## Using `COMMIT`

Normally statements are written directly to the tables: an *implicit commit* (autocommit). Inside a transaction block, commits don't happen implicitly (DBMS-specific: some treat the end of a batch as a commit).

To force an explicit commit, use `COMMIT`. SQL Server example:

```sql
BEGIN TRANSACTION
DELETE OrderItems WHERE order_num = 12345
DELETE Orders WHERE order_num = 12345
COMMIT TRANSACTION
```

**Analysis:** order `12345` is deleted completely. Because this touches two tables, a transaction ensures the order isn't partially deleted. The `COMMIT` writes the changes only if no error occurred.

<Warning>
**SQL Server gotcha**

In T-SQL, a runtime error in one statement does **not** automatically abort the transaction unless `SET XACT_ABORT ON` is set (or you use `TRY...CATCH` with `ROLLBACK`). Without it, the first `DELETE` could fail and the second still commit. Always use `SET XACT_ABORT ON` in multi-statement T-SQL transactions.
</Warning>

The PostgreSQL version:

```sql
BEGIN;
DELETE FROM OrderItems WHERE order_num = 12345;
DELETE FROM Orders     WHERE order_num = 12345;
COMMIT;
```

In PostgreSQL, any error puts the transaction into an **aborted** state ("current transaction is aborted, commands ignored until end of transaction block"); every following statement fails and `COMMIT` actually performs a `ROLLBACK`.

## Using savepoints

`ROLLBACK` and `COMMIT` work on the whole transaction. More complex transactions may need partial rollbacks. In the order example, if adding the order fails you want to roll back to before the `Orders` row, but keep the new customer.

For that you put placeholders at strategic points: **savepoints**.

| DBMS | Create | Roll back to it | Release |
|---|---|---|---|
| PostgreSQL, SQLite, MySQL, MariaDB, Oracle, DB2 | `SAVEPOINT delete1;` | `ROLLBACK TO SAVEPOINT delete1;` (`SAVEPOINT` keyword optional in PostgreSQL, MySQL and SQLite) | `RELEASE SAVEPOINT delete1;` |
| SQL Server | `SAVE TRANSACTION delete1;` | `ROLLBACK TRANSACTION delete1;` | n/a |

Each savepoint has a unique name so the DBMS knows where to roll back to. Rolling back to a savepoint keeps the transaction open; you still need a final `COMMIT` or `ROLLBACK`.

### Full example (SQL Server)

```sql
BEGIN TRANSACTION
INSERT INTO Customers(cust_id, cust_name)
VALUES(1000000010, 'Toys Emporium');
SAVE TRANSACTION StartOrder;
INSERT INTO Orders(order_num, order_date, cust_id)
VALUES(20100, '2020/12/1', 1000000010);
IF @@ERROR <> 0 ROLLBACK TRANSACTION StartOrder;
INSERT INTO OrderItems(order_num, order_item, prod_id, quantity, item_price)
VALUES(20100, 1, 'BR01', 100, 5.49);
IF @@ERROR <> 0 ROLLBACK TRANSACTION StartOrder;
INSERT INTO OrderItems(order_num, order_item, prod_id, quantity, item_price)
VALUES(20100, 2, 'BR03', 100, 10.99);
IF @@ERROR <> 0 ROLLBACK TRANSACTION StartOrder;
COMMIT TRANSACTION
```

**Analysis:** four `INSERT`s in one transaction. A savepoint is defined after the first `INSERT`, so if any later `INSERT` fails, the transaction only rolls back that far (the customer stays). `@@ERROR` is non-zero if the last statement failed. If everything succeeds, `COMMIT` saves the data. (Modern T-SQL would use `TRY...CATCH`.)

### The same in PostgreSQL

```sql
BEGIN;
INSERT INTO Customers (cust_id, cust_name) VALUES (1000000010, 'Toys Emporium');
SAVEPOINT start_order;

INSERT INTO Orders (order_num, order_date, cust_id) VALUES (20100, '2020-12-01', 1000000010);
INSERT INTO OrderItems (order_num, order_item, prod_id, quantity, item_price)
VALUES (20100, 1, 'BR01', 100, 5.49),
       (20100, 2, 'BR03', 100, 10.99);
-- If any of these failed, the client would run:
--   ROLLBACK TO SAVEPOINT start_order;
-- which clears the aborted state and keeps the customer row.

COMMIT;
```

PostgreSQL has no `IF @@ERROR` in plain SQL; the client (or a PL/pgSQL `BEGIN ... EXCEPTION ... END` block, which uses a savepoint internally) decides whether to roll back to the savepoint.

## Beyond the book / Senior perspective

- **Keep transactions short.** Locks are held until commit; long transactions block writers, and in PostgreSQL they stop `VACUUM` from cleaning dead rows (table bloat). Never wait for user input or call slow external APIs inside a transaction. Monitor `idle in transaction` sessions and set `idle_in_transaction_session_timeout`.
- **Isolation levels** decide what concurrent transactions see (dirty reads, non-repeatable reads, phantoms, write skew). PostgreSQL defaults to Read Committed, MySQL InnoDB to Repeatable Read. See [Isolation Levels & Read Phenomena](/Software%20Engineering/Databases/Core%20Concepts/Isolation%20Levels%20%26%20Read%20Phenomena).
- **Retry on serialization failures and deadlocks** (`40001`, `40P01` in PostgreSQL). Under Serializable isolation, retries are part of the contract.
- **Transactions don't span services.** Across databases or a DB plus a message broker, use the outbox pattern, sagas, or (rarely) two-phase commit ([Replication & 2PC 3PC](/Software%20Engineering/Databases/Database%20Round/Replication%20%26%202PC%203PC)).
- **Savepoints cost something.** ORMs that wrap every statement in a savepoint, or PL/pgSQL exception blocks in tight loops, add overhead; PostgreSQL also has subtransaction limits that can hurt performance at high counts.
- **Durability knobs:** PostgreSQL `synchronous_commit = off` trades a small window of possible data loss on crash for faster commits; know when that's acceptable.

## Interview questions

<Accordion title="What does ACID stand for, and which part do COMMIT/ROLLBACK provide?">
Atomicity, Consistency, Isolation, Durability. `COMMIT`/`ROLLBACK` implement atomicity (all or nothing); durability is guaranteed once `COMMIT` returns (WAL/redo log flushed).
</Accordion>

<Accordion title="Can you roll back a DROP TABLE?">
In PostgreSQL, SQLite and (mostly) SQL Server, yes, if it ran inside an open transaction. In MySQL and Oracle, no: DDL causes an implicit commit.
</Accordion>

<Accordion title="What is autocommit?">
A mode where each statement is committed automatically as its own transaction. Most clients default to it; `BEGIN` starts an explicit multi-statement transaction.
</Accordion>

<Accordion title="What is a savepoint and when would you use it?">
A named point inside a transaction you can roll back to without abandoning the whole transaction, e.g. to retry or skip one failed step in a batch. PostgreSQL syntax: `SAVEPOINT x; ... ROLLBACK TO SAVEPOINT x; RELEASE SAVEPOINT x;`.
</Accordion>

<Accordion title="What happens in PostgreSQL after an error inside a transaction?">
The transaction is marked aborted; every further statement errors until `ROLLBACK` (or `ROLLBACK TO SAVEPOINT` if a savepoint was set before the error). `COMMIT` on an aborted transaction rolls it back.
</Accordion>

<Accordion title="Why are long-running transactions dangerous in PostgreSQL?">
They hold locks, and their old snapshot prevents `VACUUM` from removing dead tuples everywhere, causing bloat and, in the extreme, transaction ID wraparound pressure.
</Accordion>

## Related
- [SQL ANSI](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20ANSI) · [Databases Index](/Software%20Engineering/Databases/Databases%20Index)
- [Transactions & ACID](/Software%20Engineering/Databases/Database%20Round/Transactions%20%26%20ACID) · [Isolation Levels & Read Phenomena](/Software%20Engineering/Databases/Core%20Concepts/Isolation%20Levels%20%26%20Read%20Phenomena) · [Replication & 2PC 3PC](/Software%20Engineering/Databases/Database%20Round/Replication%20%26%202PC%203PC)
- [Working with Stored Procedures](/Software%20Engineering/Databases/SQL%20ANSI/Working%20with%20Stored%20Procedures) · [Updating and Deleting Data](/Software%20Engineering/Databases/SQL%20ANSI/Updating%20and%20Deleting%20Data) · [Using Cursors](/Software%20Engineering/Databases/SQL%20ANSI/Using%20Cursors)

## Sources
- Ben Forta, *Sams Teach Yourself SQL in 10 Minutes a Day*, 5th ed., Lesson 20 ([O'Reilly](https://learning.oreilly.com/library/view/sams-teach-yourself/9780135182925/))
- [PostgreSQL docs: Transactions tutorial](https://www.postgresql.org/docs/current/tutorial-transactions.html), [SAVEPOINT](https://www.postgresql.org/docs/current/sql-savepoint.html), [ROLLBACK TO SAVEPOINT](https://www.postgresql.org/docs/current/sql-rollback-to.html)
- [PostgreSQL wiki: Transactional DDL in PostgreSQL](https://wiki.postgresql.org/wiki/Transactional_DDL_in_PostgreSQL:_A_Competitive_Analysis)
- [MySQL docs: Statements that cause an implicit commit](https://dev.mysql.com/doc/refman/8.4/en/implicit-commit.html)
- [SQL Server docs: SET XACT_ABORT](https://learn.microsoft.com/en-us/sql/t-sql/statements/set-xact-abort-transact-sql)
