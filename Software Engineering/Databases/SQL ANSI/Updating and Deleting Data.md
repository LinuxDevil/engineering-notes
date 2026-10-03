---
tags: [databases, databases/sql]
aliases: [UPDATE, DELETE, TRUNCATE, Soft Delete, UPDATE FROM]
updated: 2026-09-27
---
# Updating and Deleting Data

> [!tldr] TL;DR
> `UPDATE table SET col = value WHERE ...` changes rows; `DELETE FROM table WHERE ...` removes them. Forget the `WHERE` and you change **every** row, so test the `WHERE` with a `SELECT` first and run risky changes inside a transaction. `TRUNCATE` empties a whole table fast; how it's logged and whether it can be rolled back depends on the DBMS (in PostgreSQL it's transactional and WAL-logged).

> Course: SQL in 10 Minutes, Lesson 16 · Prev: [[Inserting Data]] · Next: [[Creating and Manipulating Tables]]

## Updating data

`UPDATE` can be used:
1. to update specific rows in a table,
2. to update all rows in a table.

### Example 1

Customer `1000000005` had no email address and now has one:

```sql
UPDATE Customers
SET cust_email = 'kim@thetoystore.com'
WHERE cust_id = 1000000005;
```

**Analysis:** `UPDATE` starts with the table name, `Customers`. `SET` assigns the new value to a column:

```sql
SET cust_email = 'kim@thetoystore.com'
```

The `WHERE` tells the DBMS which row to update. Without it, **every** customer would get this email address.

### Updating several columns

```sql
UPDATE Customers
SET cust_contact = 'Sam Roberts',
    cust_email   = 'sam@toyland.com'
WHERE cust_id = 1000000006;
```

One `SET`, comma-separated `column = value` pairs.

### Clearing a value

```sql
UPDATE Customers
SET cust_email = NULL
WHERE cust_id = 1000000005;
```

Setting a column to `NULL` removes its value (if the column allows `NULL`).

### Updating from another table

The syntax isn't standardized:

```sql
-- PostgreSQL, SQL Server (similar), SQLite 3.33+
UPDATE Products p
SET prod_price = f.new_price
FROM PriceFeed f
WHERE f.prod_id = p.prod_id;

-- MySQL
UPDATE Products p
JOIN PriceFeed f ON f.prod_id = p.prod_id
SET p.prod_price = f.new_price;

-- Portable: correlated subquery
UPDATE Products
SET prod_price = (SELECT new_price FROM PriceFeed f WHERE f.prod_id = Products.prod_id)
WHERE EXISTS (SELECT 1 FROM PriceFeed f WHERE f.prod_id = Products.prod_id);
```

Without the `EXISTS` in the portable form, products with no feed row would get `NULL` prices.

## Deleting data

`DELETE` can be used:
1. to delete specific rows,
2. to delete all rows.

### Example 2

Delete the customer added in the last lesson:

```sql
DELETE FROM Customers
WHERE cust_id = 1000000006;
```

**Analysis:** `DELETE FROM` names the table; `WHERE` picks the rows. Only customer `1000000006` is deleted. Without the `WHERE`, **every** customer would be deleted.

`DELETE` removes whole rows. To remove a value from a column, `UPDATE` it to `NULL`. `DELETE` never deletes the table itself; that's `DROP TABLE`.

## Faster deletes: `TRUNCATE`

To delete all rows, `TRUNCATE TABLE` is much faster than `DELETE`: it deallocates the table's storage instead of removing rows one by one, doesn't fire row-level `DELETE` triggers, and usually resets identity counters (optional in PostgreSQL with `RESTART IDENTITY`).

> [!warning] Correction: "TRUNCATE is not logged" is not universally true
> The original note said `TRUNCATE` is faster "because data changes are not logged". That's an oversimplification:
>
> | DBMS | Logging | Can it be rolled back? |
> |---|---|---|
> | **PostgreSQL** | WAL-logged (it logs the file swap, not each row) | **Yes**, it's fully transactional inside `BEGIN ... ROLLBACK` |
> | **SQL Server** | **Minimally logged** (logs page deallocations) | Yes, inside an explicit transaction |
> | **MySQL / MariaDB** | DDL: drops and recreates the table | **No**, it causes an implicit commit |
> | **Oracle** | DDL | **No**, implicit commit |
> | SQLite | No `TRUNCATE`; `DELETE FROM t` without `WHERE` uses a fast "truncate optimization" | Yes |
>
> Other differences from `DELETE`: `TRUNCATE` needs a stronger lock (PostgreSQL takes `ACCESS EXCLUSIVE`, blocking reads), fails if other tables reference it with foreign keys (unless `CASCADE` in PostgreSQL), and in PostgreSQL isn't MVCC-safe (a concurrent transaction using an older snapshot will see the table as empty).

```sql
-- PostgreSQL: empty two related tables and reset their identity sequences
TRUNCATE TABLE OrderItems, Orders RESTART IDENTITY;
```

## Guidelines for updating and deleting data

Without a `WHERE`, `UPDATE` and `DELETE` apply to every row. Guidelines many SQL programmers follow:

1. Never run `UPDATE` or `DELETE` without a `WHERE` unless you really intend to affect every row.
2. Make sure every table has a primary key, and use it in the `WHERE` whenever possible (single keys, lists or ranges).
3. Before running an `UPDATE` or `DELETE`, test the `WHERE` with a `SELECT` to be sure it selects the right rows.
4. Use database-enforced referential integrity (foreign keys) so the DBMS refuses to delete rows that other tables depend on.
5. Some DBMSs can refuse `UPDATE`/`DELETE` without a `WHERE`. Use it if available: MySQL's `--safe-updates` / `sql_safe_updates=1`; in PostgreSQL the `pg-safeupdate` extension.
6. Wrap manual data fixes in a transaction and check the affected row count before committing:

```sql
BEGIN;
UPDATE Customers SET cust_state = 'NY' WHERE cust_city = 'New York';
-- "UPDATE 3"  <- expected 3? then:
COMMIT;        -- otherwise ROLLBACK;
```

## Beyond the book / Senior perspective

- **`RETURNING`** shows exactly what changed: `DELETE FROM sessions WHERE expires_at < now() RETURNING id;`. PostgreSQL 18 (2025-09-25) allows `RETURNING old.*, new.*` in `UPDATE`, handy for audit logs and "before/after" checks. See [[Inserting Data]].
- **Big deletes in batches.** `DELETE FROM events WHERE created_at < '2025-01-01'` on 100M rows holds locks, bloats the table (PostgreSQL MVCC leaves dead tuples for `VACUUM`) and can lag replicas. Delete in chunks, or better, **drop old partitions** ([[SQL Table Partitioning]]).
- **Soft deletes** (`deleted_at timestamptz`) keep history and allow undo, but every query must filter them, unique constraints need partial indexes (`WHERE deleted_at IS NULL`), and GDPR erasure may still require a hard delete.
- **`ON DELETE CASCADE`** is convenient but can delete far more than expected in one statement. Know which FKs cascade.
- **Lost updates.** `UPDATE accounts SET balance = 150` after reading `100` in the app overwrites concurrent changes. Update relative to the current value (`SET balance = balance + 50`), use `SELECT ... FOR UPDATE`, or optimistic locking with a version column. See [[Isolation Levels & Read Phenomena]].

## Interview questions

> [!question]- DELETE vs TRUNCATE vs DROP?
> `DELETE` removes selected rows one by one, fires triggers and is always transactional. `TRUNCATE` removes all rows by deallocating storage, is much faster, skips row triggers, and resets identity; it's rollback-able in PostgreSQL and SQL Server but auto-commits in MySQL and Oracle. `DROP` removes the table itself.

> [!question]- Is TRUNCATE logged?
> In PostgreSQL it's WAL-logged and transactional (it logs the file operation, not each row). In SQL Server it's minimally logged (page deallocations). "Unlogged" is inaccurate in both.

> [!question]- How do you delete 500 million old rows without taking the site down?
> Ideally, drop or detach time-based partitions. Otherwise delete in small batches by primary key range with pauses, monitor replication lag and vacuum, and do it off-peak.

> [!question]- How do you prevent lost updates?
> Atomic relative updates (`SET x = x + 1`), pessimistic locking (`SELECT ... FOR UPDATE`), optimistic locking (`WHERE version = $v` and check the row count), or a stricter isolation level.

> [!question]- How do you update a table using values from another table?
> `UPDATE ... FROM` (PostgreSQL/SQL Server), `UPDATE ... JOIN` (MySQL), `MERGE`, or a correlated subquery with an `EXISTS` guard.

## Related
- [[SQL ANSI]] · [[Databases Index]]
- [[Inserting Data]] · [[Managing Transaction Processing]] · [[Transactions & ACID]] · [[Isolation Levels & Read Phenomena]] · [[SQL Table Partitioning]]

## Sources
- Ben Forta, *Sams Teach Yourself SQL in 10 Minutes a Day*, 5th ed., Lesson 16 ([O'Reilly](https://learning.oreilly.com/library/view/sams-teach-yourself/9780135182925/))
- [PostgreSQL docs: TRUNCATE](https://www.postgresql.org/docs/current/sql-truncate.html)
- [SQL Server docs: TRUNCATE TABLE](https://learn.microsoft.com/en-us/sql/t-sql/statements/truncate-table-transact-sql)
- [MySQL docs: TRUNCATE TABLE](https://dev.mysql.com/doc/refman/8.4/en/truncate-table.html) and [Statements that cause an implicit commit](https://dev.mysql.com/doc/refman/8.4/en/implicit-commit.html)
- [PostgreSQL docs: UPDATE](https://www.postgresql.org/docs/current/sql-update.html)
