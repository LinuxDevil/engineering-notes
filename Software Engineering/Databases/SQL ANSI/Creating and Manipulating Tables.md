---
tags: [databases, databases/sql, databases/postgres]
aliases: [CREATE TABLE, ALTER TABLE, DROP TABLE, DDL, Identity Columns, Generated Columns, uuidv7]
updated: 2026-09-27
---
# Creating and Manipulating Tables

> [!tldr] TL;DR
> `CREATE TABLE` defines columns, types, `NULL`/`NOT NULL` and `DEFAULT`s; `ALTER TABLE` changes the definition; `DROP TABLE` deletes it with no undo (outside a transaction). Modern practice adds **identity columns** (or UUIDv7) for keys, constraints from day one, and **generated columns** for derived values. On a live system, schema changes are about **locks**: know which `ALTER`s rewrite the table.

> Course: SQL in 10 Minutes, Lesson 17 · Prev: [[Updating and Deleting Data]] · Next: [[Using Views]]

## Creating tables

SQL is used not only to manipulate data but also to create and change the tables themselves (DDL, Data Definition Language). Two ways to create tables:
1. An administration tool (pgAdmin, DBeaver, SSMS, MySQL Workbench) that creates and manages tables interactively.
2. SQL statements written directly.

Interactive tools also generate and run SQL behind the scenes. In real projects, DDL lives in version-controlled **migration** files (Flyway, Liquibase, Alembic, Prisma, Rails, etc.), not in clicks.

## Basic table creation

`CREATE TABLE` needs:
1. the new table's name, after `CREATE TABLE`;
2. the name and definition of each column, separated by commas;
3. in some DBMSs, the location (which database or schema).

### Example 1: the book's `Products` table

```sql
CREATE TABLE Products
(
    prod_id      CHAR(10)        NOT NULL,
    vend_id      CHAR(10)        NOT NULL,
    prod_name    CHAR(254)       NOT NULL,
    prod_price   DECIMAL(8,2)    NOT NULL,
    prod_desc    VARCHAR(1000)   NULL
);
```

**Analysis:** the table name follows `CREATE TABLE`, and the column definitions are inside parentheses, separated by commas. Five columns; each starts with a name (unique within the table) followed by its datatype.

> [!tip] Datatype choices the book doesn't discuss
> - `CHAR(n)` pads with spaces (hence all the `RTRIM()` calls in earlier lessons). Prefer `VARCHAR(n)` or PostgreSQL `TEXT`.
> - Money: `DECIMAL`/`NUMERIC`, never `FLOAT`.
> - Instants: `TIMESTAMPTZ` in PostgreSQL (`DATETIME` is SQL Server/MySQL, and doesn't exist in PostgreSQL).
> - IDs: `BIGINT` rather than `INT`.

## Working with `NULL` values

Every column is either `NULL` (allowed to be empty) or `NOT NULL`, set at creation time.

### Example 2

```sql
CREATE TABLE Orders
(
    order_num     INTEGER     NOT NULL,
    order_date    DATETIME    NOT NULL,   -- TIMESTAMP / TIMESTAMPTZ in PostgreSQL
    cust_id       CHAR(10)    NOT NULL
);
```

**Analysis:** all three columns are required, so each is `NOT NULL`. Inserting a row without a value for one of them fails with an error.

### Example 3: a mix of `NULL` and `NOT NULL`

```sql
CREATE TABLE Vendors
(
    vend_id        CHAR(10)    NOT NULL,
    vend_name      CHAR(50)    NOT NULL,
    vend_address   CHAR(50)    ,
    vend_city      CHAR(50)    ,
    vend_state     CHAR(5)     ,
    vend_zip       CHAR(10)    ,
    vend_country   CHAR(50)
);
```

**Analysis:** vendor ID and name are required; the other five allow `NULL`. `NULL` is the default in most DBMSs, so if `NOT NULL` isn't specified, `NULL` is assumed. (Exception: SQL Server can be configured otherwise, so being explicit is safer.)

> [!warning] `NULL` is not an empty string
> `''` is a value; `NULL` is the absence of one. `NOT NULL` columns accept `''`. (Oracle is the odd one out: it treats `''` as `NULL`.)

## Specifying default values

`DEFAULT` gives a value to use when none is supplied on insert.

### Example 4

```sql
CREATE TABLE OrderItems
(
    order_num     INTEGER         NOT NULL,
    order_item    INTEGER         NOT NULL,
    prod_id       CHAR(10)        NOT NULL,
    quantity      INTEGER         NOT NULL    DEFAULT 1,
    item_price    DECIMAL(8,2)    NOT NULL
);
```

**Analysis:** `DEFAULT 1` makes the quantity 1 when no quantity is given.

Defaults are often used for date/timestamp columns, with the system date. The function differs by DBMS:

| DBMS | Default for the current date |
|---|---|
| PostgreSQL | `CURRENT_DATE` (or `now()` / `CURRENT_TIMESTAMP` for a timestamp) |
| MySQL / MariaDB | `CURRENT_DATE` (as `(CURRENT_DATE)` in 8.0.13+), `CURRENT_TIMESTAMP` |
| Oracle | `SYSDATE` |
| SQL Server | `GETDATE()` |
| SQLite | `date('now')` / `CURRENT_DATE` |
| DB2 | `CURRENT_DATE` |

> [!tip] Defaults vs `NULL`
> Many developers prefer `DEFAULT` values over `NULL` columns, especially for columns used in calculations or grouping.

## Primary keys, identity columns and UUIDs

The book's examples add keys in [[Understanding Advanced SQL Features]]. Modern PostgreSQL:

```sql
CREATE TABLE orders (
  order_num   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  cust_id     BIGINT      NOT NULL REFERENCES customers (cust_id),
  order_date  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

| Option | Pros | Cons |
|---|---|---|
| `IDENTITY` (`BIGINT`) | Standard, compact (8 bytes), sequential inserts into the B-tree | Guessable; must be generated by the DB |
| `SERIAL` / `BIGSERIAL` | Works everywhere in PostgreSQL | Legacy shorthand; weaker ownership of the sequence |
| `AUTO_INCREMENT` (MySQL), `IDENTITY(1,1)` (SQL Server) | Native | Dialect-specific |
| UUIDv4 (`gen_random_uuid()`) | Generate anywhere, unguessable | 16 bytes; **random** inserts scatter across the index (page splits, cache misses) |
| **UUIDv7** | Generate anywhere, **time-ordered**, so inserts stay at the index's right edge | Embeds creation time (can leak it) |

## Generated columns

A column computed from other columns of the same row:

```sql
CREATE TABLE order_items (
  order_num      BIGINT        NOT NULL,
  order_item     INT           NOT NULL,
  quantity       INT           NOT NULL DEFAULT 1,
  item_price     NUMERIC(8,2)  NOT NULL,
  expanded_price NUMERIC(10,2) GENERATED ALWAYS AS (quantity * item_price) STORED,
  PRIMARY KEY (order_num, order_item)
);
```

`STORED` computes on write and stores the value (indexable). `VIRTUAL` computes on read and uses no space. MySQL 5.7+ supports both; SQL Server has computed columns (`PERSISTED` or not).

> [!info]+ 🆕 2025–2026
> - **PostgreSQL 18 (released 2025-09-25)** added **virtual generated columns**, and virtual is now the **default** kind: `GENERATED ALWAYS AS (expr)` without `STORED` is virtual. Before 18, PostgreSQL only had `STORED`. Write `STORED` explicitly if you need to index the value or want the old behaviour.
> - **PostgreSQL 18** added **`uuidv7()`**: timestamp-ordered UUIDs, much better for B-tree primary key locality than random `gen_random_uuid()` (v4).
>   ```sql
>   CREATE TABLE events (
>     id         uuid PRIMARY KEY DEFAULT uuidv7(),
>     payload    jsonb NOT NULL,
>     created_at timestamptz NOT NULL DEFAULT now()
>   );
>   ```
> - **PostgreSQL 18** also added temporal constraints: `PRIMARY KEY`/`UNIQUE ... WITHOUT OVERLAPS` on a range column and `PERIOD` in foreign keys (e.g. "a room can't have two overlapping bookings").
> - **MySQL 8.0 reached end of life on 2026-04-30.** Current LTS lines are **8.4** (2024-04-30) and **9.7** (2026-04-21); plan upgrades of any DDL/migration tooling accordingly.

## Updating tables: `ALTER TABLE`

All DBMSs support `ALTER TABLE`, but what you can alter varies a lot. Points to consider:
1. Ideally, tables shouldn't need altering once they contain data; spend time on design up front. (In reality schemas evolve; the skill is changing them safely.)
2. All DBMSs let you add columns, though some restrict types, `NULL` and `DEFAULT` usage.
3. Some DBMSs don't let you remove or change columns (SQLite only gained `DROP COLUMN` in 3.35.0).
4. Most DBMSs allow renaming columns.
5. Many restrict changes on populated columns more than on empty ones.

To alter a table, specify:
1. the table name after `ALTER TABLE` (it must exist),
2. the list of changes.

### Example 5

```sql
ALTER TABLE Vendors
ADD vend_phone CHAR(20);
```

**Analysis:** adds a `vend_phone` column to `Vendors`; the datatype is required. Other operations use similar syntax:

```sql
ALTER TABLE Vendors DROP COLUMN vend_phone;
ALTER TABLE Vendors ALTER COLUMN vend_name TYPE VARCHAR(100);        -- PostgreSQL
ALTER TABLE Vendors MODIFY vend_name VARCHAR(100);                  -- MySQL
ALTER TABLE Vendors RENAME COLUMN vend_zip TO vend_postcode;
ALTER TABLE Vendors ADD CONSTRAINT vendors_pk PRIMARY KEY (vend_id);
```

> [!note] SQLite
> SQLite's `ALTER TABLE` is limited. It can't add primary or foreign keys after the fact; define them in the initial `CREATE TABLE`. The standard workaround for bigger changes: create a new table, copy data, drop the old one, rename the new one (inside a transaction).

## Deleting tables

Deleting a table (the whole table, not just its contents) is easy, arguably too easy:

### Example 6

```sql
DROP TABLE CustCopy;
```

**Analysis:** deletes the `CustCopy` table. No confirmation, and no undo, unless you're in PostgreSQL inside an open transaction (DDL is transactional there, so `ROLLBACK` brings it back; see [[Managing Transaction Processing]]). Use `DROP TABLE IF EXISTS` in scripts, and note that tables referenced by foreign keys can't be dropped without `CASCADE`.

## Renaming tables

No hard standard:

| DBMS | Syntax |
|---|---|
| PostgreSQL, SQLite | `ALTER TABLE CustCopy RENAME TO CustBackup;` |
| MySQL, MariaDB | `RENAME TABLE CustCopy TO CustBackup;` (or `ALTER TABLE ... RENAME TO`) |
| Oracle, DB2 | `RENAME CustCopy TO CustBackup;` (DB2: `RENAME TABLE`) |
| SQL Server | `EXEC sp_rename 'CustCopy', 'CustBackup';` |

(The original note listed PostgreSQL under `RENAME`; PostgreSQL has no standalone `RENAME` statement.)

## Beyond the book / Senior perspective

- **Schema changes lock tables.** In PostgreSQL most `ALTER TABLE` forms take an `ACCESS EXCLUSIVE` lock. Cheap ones (add nullable column, add column with a constant default since PG 11, rename) are instant; expensive ones rewrite the table (change type, add a volatile default). Any of them can queue behind a long-running query and then block everyone behind *them*. Set `lock_timeout` (e.g. `SET lock_timeout = '3s'`) and retry.
- **Safe migration patterns**: add constraints as `NOT VALID` then `VALIDATE CONSTRAINT` separately; create indexes `CONCURRENTLY` ([[SQL Indexes Deep Dive]]); expand-and-contract for renames (add new column, dual-write, backfill in batches, switch reads, drop old). MySQL has online DDL (`ALGORITHM=INSTANT/INPLACE`) and tools like gh-ost / pt-online-schema-change.
- **Constraints are documentation that can't lie.** `NOT NULL`, `CHECK`, `UNIQUE` and FKs catch bugs the application misses.
- **Temporary and unlogged tables**: `CREATE TEMP TABLE` (session-local) and PostgreSQL `UNLOGGED` tables (fast, not crash-safe, not replicated) for scratch data.

## Interview questions

> [!question]- IDENTITY, SERIAL or UUID for primary keys?
> Identity `BIGINT` is compact and index-friendly. UUIDs let clients generate IDs and avoid guessable URLs; use time-ordered UUIDv7 (native `uuidv7()` in PostgreSQL 18) rather than random v4 to keep B-tree inserts local.

> [!question]- What's a generated column? STORED vs VIRTUAL?
> A column computed from others in the same row. STORED computes on write and takes space (indexable); VIRTUAL computes on read. PostgreSQL 18 added virtual and made it the default.

> [!question]- Why can a simple ALTER TABLE take down production?
> It needs an exclusive lock. If a long query holds a conflicting lock, the ALTER waits, and all new queries queue behind the ALTER. Some ALTERs also rewrite the whole table. Use `lock_timeout`, off-peak windows and online patterns.

> [!question]- How do you rename a column with zero downtime?
> Expand/contract: add the new column, write to both, backfill, switch reads, stop writing the old one, then drop it. A direct rename breaks any running code that uses the old name.

> [!question]- Does CREATE TABLE AS copy constraints?
> No, only column definitions and data. Add keys and indexes afterwards, or use `CREATE TABLE ... (LIKE src INCLUDING ALL)` in PostgreSQL.

## Related
- [[SQL ANSI]] · [[Databases Index]]
- [[Understanding Advanced SQL Features]] (constraints) · [[Inserting Data]] · [[Managing Transaction Processing]] · [[SQL Indexes Deep Dive]] · [[Data Modeling & Normalization]] · [[Primary vs Secondary Key]]

## Sources
- Ben Forta, *Sams Teach Yourself SQL in 10 Minutes a Day*, 5th ed., Lesson 17 ([O'Reilly](https://learning.oreilly.com/library/view/sams-teach-yourself/9780135182925/))
- [PostgreSQL docs: CREATE TABLE](https://www.postgresql.org/docs/current/sql-createtable.html) and [Generated columns](https://www.postgresql.org/docs/current/ddl-generated-columns.html)
- [PostgreSQL docs: ALTER TABLE (lock levels)](https://www.postgresql.org/docs/current/sql-altertable.html)
- [PostgreSQL 18 released (virtual generated columns, uuidv7, temporal constraints)](https://www.postgresql.org/about/news/postgresql-18-released-3142/)
- [endoflife.date: MySQL](https://endoflife.date/mysql)
- [SQLite docs: ALTER TABLE](https://www.sqlite.org/lang_altertable.html)
