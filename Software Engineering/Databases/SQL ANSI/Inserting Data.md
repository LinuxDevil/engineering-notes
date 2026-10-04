---
title: "Inserting Data"
tags: [databases, databases/sql]
aliases: [INSERT, INSERT SELECT, Upsert, ON CONFLICT, MERGE, RETURNING, CREATE TABLE AS]
updated: 2026-09-27
---
# Inserting Data

<Note>
**TL;DR**

`INSERT INTO t (cols) VALUES (...)` adds rows; always list the columns. `INSERT ... SELECT` copies query results into an existing table, `CREATE TABLE ... AS SELECT` creates a new one. In practice you'll also need **upserts** (`INSERT ... ON CONFLICT` in PostgreSQL/SQLite, `ON DUPLICATE KEY UPDATE` in MySQL, standard `MERGE`) and **`RETURNING`** to get generated IDs back without a second query.
</Note>

> Course: SQL in 10 Minutes, Lesson 15 · Prev: [Combining Queries](/Software%20Engineering/Databases/SQL%20ANSI/Combining%20Queries) · Next: [Updating and Deleting Data](/Software%20Engineering/Databases/SQL%20ANSI/Updating%20and%20Deleting%20Data)

## What is `INSERT`

`INSERT` adds rows to a table. It can be used to insert:
1. a single complete row,
2. a single partial row,
3. the results of a query.

## Inserting a complete row

### Example 1

```sql
INSERT INTO Customers(cust_id,
                      cust_name,
                      cust_address,
                      cust_city,
                      cust_state,
                      cust_zip,
                      cust_country,
                      cust_contact,
                      cust_email)
VALUES(1000000006,
       'Toy Land',
       '123 Any Street',
       'New York',
       'NY',
       '11111',
       'USA',
       NULL,
       NULL);
```

**Analysis:** inserts a new customer. `VALUES` supplies a value for each listed column, **in the order of the column list**. Columns with no value (`cust_contact`, `cust_email`) get `NULL`, assuming the table allows it.

<Warning>
**Always name the columns**

`INSERT INTO Customers VALUES (...)` without a column list also works, but then values must match the table's column order exactly. It breaks the moment someone adds or reorders a column. Always list the columns.
</Warning>

## Inserting partial rows

### Example 2

```sql
INSERT INTO Customers(cust_id,
                      cust_name,
                      cust_address,
                      cust_city,
                      cust_state,
                      cust_zip,
                      cust_country)
VALUES(1000000006,
       'Toy Land',
       '123 Any Street',
       'New York',
       'NY',
       '11111',
       'USA');
```

**Analysis:** `cust_contact` and `cust_email` had no values, so they're simply left out of the statement. Omitted columns get their `DEFAULT` (or `NULL`); omitting a `NOT NULL` column without a default is an error.

## Multi-row insert

```sql
INSERT INTO Products (prod_id, vend_id, prod_name, prod_price)
VALUES ('TOY01', 'DLL01', 'Yo-yo',  2.99),
       ('TOY02', 'DLL01', 'Kite',   7.49),
       ('TOY03', 'BRS01', 'Marbles', 1.99);
```

One statement, one round trip, much faster than three single inserts. For bulk loads use `COPY` (PostgreSQL), `LOAD DATA` (MySQL) or `BULK INSERT` (SQL Server).

## Inserting retrieved data: `INSERT SELECT`

### Example 3

```sql
INSERT INTO Customers(cust_id,
                      cust_contact,
                      cust_email,
                      cust_name,
                      cust_address,
                      cust_city,
                      cust_state,
                      cust_zip,
                      cust_country)
SELECT cust_id,
       cust_contact,
       cust_email,
       cust_name,
       cust_address,
       cust_city,
       cust_state,
       cust_zip,
       cust_country
FROM CustNew;
```

**Analysis:** imports all rows from `CustNew` into `Customers`. The `SELECT` provides the values; columns are matched **by position**, not by name. If `CustNew` is empty, zero rows are inserted and no error is raised.

## Copying from one table to another

`INSERT SELECT` appends to an existing table. `CREATE TABLE ... AS SELECT` (CTAS) creates a **new** table and fills it.

### Example 4

```sql
CREATE TABLE CustCopy AS SELECT * FROM Customers;
```

**Analysis:** creates `CustCopy` with every column of `Customers` and copies all rows. List columns instead of `*` to copy a subset. SQL Server uses `SELECT * INTO CustCopy FROM Customers;`.

<Note>
CTAS copies columns and data, but **not** primary keys, indexes, defaults or foreign keys. In PostgreSQL, `CREATE TABLE t (LIKE Customers INCLUDING ALL)` copies the structure.
</Note>

## Getting generated values back: `RETURNING`

```sql
-- PostgreSQL, SQLite 3.35+, MariaDB 10.5+ (SQL Server: OUTPUT inserted.*)
INSERT INTO Orders (order_date, cust_id)
VALUES (CURRENT_DATE, 1000000001)
RETURNING order_num;
```

No race-prone `SELECT MAX(id)` afterwards, no second round trip. MySQL has `LAST_INSERT_ID()` instead.

## Upserts: insert or update

```sql
-- PostgreSQL 9.5+ and SQLite 3.24+
INSERT INTO Products (prod_id, vend_id, prod_name, prod_price)
VALUES ('TOY01', 'DLL01', 'Yo-yo', 3.49)
ON CONFLICT (prod_id)
DO UPDATE SET prod_price = EXCLUDED.prod_price;   -- EXCLUDED = the row we tried to insert

-- Ignore duplicates instead
INSERT INTO Products (prod_id, vend_id, prod_name, prod_price)
VALUES ('TOY01', 'DLL01', 'Yo-yo', 3.49)
ON CONFLICT (prod_id) DO NOTHING;

-- MySQL / MariaDB
INSERT INTO Products (prod_id, vend_id, prod_name, prod_price)
VALUES ('TOY01', 'DLL01', 'Yo-yo', 3.49)
ON DUPLICATE KEY UPDATE prod_price = VALUES(prod_price);  -- 8.0.20+: use a row alias instead
```

`ON CONFLICT` needs a unique index or constraint on the conflict columns. It's atomic, unlike "SELECT, then INSERT or UPDATE" in application code, which races under concurrency.

### `MERGE` (standard SQL)

```sql
-- PostgreSQL 15+, SQL Server, Oracle, DB2
MERGE INTO Products p
USING ProductFeed f ON p.prod_id = f.prod_id
WHEN MATCHED AND f.discontinued THEN DELETE
WHEN MATCHED THEN UPDATE SET prod_price = f.prod_price
WHEN NOT MATCHED THEN INSERT (prod_id, vend_id, prod_name, prod_price)
                      VALUES (f.prod_id, f.vend_id, f.prod_name, f.prod_price);
```

`MERGE` syncs a table from a source in one statement. In PostgreSQL, prefer `ON CONFLICT` for concurrent single-row upserts: `MERGE` doesn't use the unique index as an arbiter and can fail with a unique violation if another session inserts the same key at the same moment.

<Info>
**🆕 2025–2026**

- **PostgreSQL 17 (2024-09-26)** added `RETURNING` to `MERGE` (with `merge_action()` telling you whether each row was inserted, updated or deleted).
- **PostgreSQL 18 (2025-09-25)**: `RETURNING` can reference both `OLD` and `NEW` values in `INSERT`, `UPDATE`, `DELETE` and `MERGE`. For an upsert that means you can see what the row looked like before the update:
  ```sql
  INSERT INTO Products (prod_id, vend_id, prod_name, prod_price)
  VALUES ('TOY01', 'DLL01', 'Yo-yo', 3.49)
  ON CONFLICT (prod_id) DO UPDATE SET prod_price = EXCLUDED.prod_price
  RETURNING old.prod_price AS before, new.prod_price AS after;  -- old.* is NULL for a fresh insert
  ```
- **PostgreSQL 18** also added `uuidv7()` for time-ordered UUID keys (see [Creating and Manipulating Tables](/Software%20Engineering/Databases/SQL%20ANSI/Creating%20and%20Manipulating%20Tables)).
- **PostgreSQL 19 (beta, not yet released; GA expected around October 2026)**: the v19 release notes list `INSERT ... ON CONFLICT DO SELECT ... RETURNING`, which returns the **existing** conflicting row (optionally locked `FOR UPDATE`/`SHARE`) instead of doing nothing. This is the "get or create" pattern in one statement. Check the final release notes before relying on it.
</Info>

## Beyond the book / Senior perspective

- **Identity columns over `SERIAL`.** Use `id BIGINT GENERATED ALWAYS AS IDENTITY` (standard, PostgreSQL 10+). `SERIAL` is a legacy shorthand for a sequence with looser ownership and permissions. Use `bigint`: `int` IDs overflowing at 2.1 billion is a real outage story.
- **Batch your inserts.** Multi-row `VALUES`, `COPY`, or driver batch APIs. Thousands of single-row autocommit inserts each pay a round trip and a WAL flush.
- **Idempotency.** Retried requests (network timeouts, queue redelivery) should not create duplicates. Use a unique key (e.g. an idempotency key column) plus `ON CONFLICT DO NOTHING`.
- **Parameters, always.** Values from users go in as bind parameters, never concatenated ([Advanced Filtering Data](/Software%20Engineering/Databases/SQL%20ANSI/Advanced%20Filtering%20Data)).
- **Big `INSERT ... SELECT`s** hold locks and generate lots of WAL/redo in one transaction. Chunk them for large migrations.

## Interview questions

<Accordion title="Why should INSERT always specify a column list?">
Without it, values map to the current physical column order, so schema changes (added or reordered columns) break the statement or put data in the wrong columns.
</Accordion>

<Accordion title="How do you implement an upsert safely?">
A single atomic statement backed by a unique constraint: `INSERT ... ON CONFLICT (key) DO UPDATE` (PostgreSQL/SQLite), `ON DUPLICATE KEY UPDATE` (MySQL), or `MERGE` (SQL Server/Oracle, with care about concurrency). Not "select then insert" in app code, which races.
</Accordion>

<Accordion title="How do you get the ID of the row you just inserted?">
`RETURNING id` (PostgreSQL, SQLite, MariaDB), `OUTPUT inserted.id` (SQL Server), `LAST_INSERT_ID()` (MySQL), or the driver's generated-keys API. Never `SELECT MAX(id)`.
</Accordion>

<Accordion title="INSERT SELECT vs CREATE TABLE AS?">
`INSERT SELECT` appends to an existing table. `CREATE TABLE AS SELECT` creates a new table from the query; it copies data and column types, not constraints or indexes.
</Accordion>

<Accordion title="SERIAL vs IDENTITY in PostgreSQL?">
Both use a sequence. `IDENTITY` is SQL-standard, ties the sequence tightly to the column, and with `GENERATED ALWAYS` prevents accidental manual values. Prefer it for new tables.
</Accordion>

## Related
- [SQL ANSI](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20ANSI) · [Databases Index](/Software%20Engineering/Databases/Databases%20Index)
- [Updating and Deleting Data](/Software%20Engineering/Databases/SQL%20ANSI/Updating%20and%20Deleting%20Data) · [Creating and Manipulating Tables](/Software%20Engineering/Databases/SQL%20ANSI/Creating%20and%20Manipulating%20Tables) · [Managing Transaction Processing](/Software%20Engineering/Databases/SQL%20ANSI/Managing%20Transaction%20Processing) · [Transactions & ACID](/Software%20Engineering/Databases/Database%20Round/Transactions%20%26%20ACID)

## Sources
- Ben Forta, *Sams Teach Yourself SQL in 10 Minutes a Day*, 5th ed., Lesson 15 ([O'Reilly](https://learning.oreilly.com/library/view/sams-teach-yourself/9780135182925/))
- [PostgreSQL docs: INSERT (ON CONFLICT, RETURNING)](https://www.postgresql.org/docs/current/sql-insert.html)
- [PostgreSQL docs: MERGE](https://www.postgresql.org/docs/current/sql-merge.html)
- [PostgreSQL 18 released (RETURNING OLD/NEW, uuidv7)](https://www.postgresql.org/about/news/postgresql-18-released-3142/)
- [PostgreSQL 19 release notes (ON CONFLICT DO SELECT)](https://www.postgresql.org/docs/19/release-19.html) · [PostgreSQL 19 Beta 4 announcement](https://www.postgresql.org/about/news/postgresql-19-beta-4-released-3386/)
- [MySQL docs: INSERT ... ON DUPLICATE KEY UPDATE](https://dev.mysql.com/doc/refman/8.4/en/insert-on-duplicate.html)
- [SQLite docs: UPSERT](https://www.sqlite.org/lang_upsert.html)
