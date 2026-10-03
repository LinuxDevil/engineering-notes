---
tags: [databases, databases/sql]
aliases: [SELECT, Retrieving Data, SELECT DISTINCT, LIMIT, FETCH FIRST]
updated: 2026-09-27
---
# Retrieving Information

> [!tldr] TL;DR
> `SELECT columns FROM table` reads data. `DISTINCT` removes duplicate rows from the result, and limiting the number of rows is dialect-specific: `LIMIT n` (PostgreSQL, MySQL, SQLite), `TOP n` (SQL Server), or the **standard** `FETCH FIRST n ROWS ONLY` (PostgreSQL, Oracle 12c+, DB2, SQL Server with `OFFSET`). Without `ORDER BY`, the order of rows is not guaranteed, so "first 5" means nothing.

> Course: SQL in 10 Minutes, Lesson 2 · Prev: [[Introduction to SQL]] · Next: [[Sorting Data]]

## The `SELECT` statement

Basic structure: `SELECT <what> FROM <where>`.

### Example 1: one column

```sql
SELECT prod_name
FROM Products;
```

**Analysis:** retrieves a single column, `prod_name`, from the `Products` table. The column name comes right after `SELECT`, and `FROM` names the table.

### Example 2: several columns

```sql
SELECT prod_id, prod_name, prod_price
FROM Products;
```

**Analysis:** same idea, three columns separated by commas. There is no comma after the last column.

### All columns: `SELECT *`

```sql
SELECT * FROM Products;
```

Handy while exploring. Avoid it in application code: it fetches columns you don't need, breaks when columns are added or reordered, and prevents index-only scans.

## The `DISTINCT` keyword

Returns only unique rows.

### Example 3

```sql
SELECT DISTINCT vend_id
FROM Products;
```

**Analysis:** tells the DBMS to return only distinct `vend_id` values, so only three rows come back. `DISTINCT` goes directly before the column list and applies to **all** selected columns together: `SELECT DISTINCT vend_id, prod_price` returns distinct *pairs*.

> [!tip] PostgreSQL `DISTINCT ON`
> `SELECT DISTINCT ON (cust_id) cust_id, order_date FROM Orders ORDER BY cust_id, order_date DESC;` keeps the first row per `cust_id` (here: each customer's latest order). It's a PostgreSQL extension; the portable version uses `ROW_NUMBER()` (see [[Understanding Advanced SQL Features]]).

## Limiting results

### Example 4: `LIMIT`

```sql
SELECT prod_name
FROM Products
LIMIT 5;
```

**Analysis:** `LIMIT 5` tells supported DBMSs to return no more than five rows.

The syntax differs by DBMS:

| DBMS | Syntax |
|---|---|
| PostgreSQL, MySQL, MariaDB, SQLite | `SELECT ... LIMIT 5 OFFSET 10` |
| SQL Server | `SELECT TOP 5 ...` or `ORDER BY ... OFFSET 10 ROWS FETCH NEXT 5 ROWS ONLY` |
| Oracle 12c+, DB2, PostgreSQL | `ORDER BY ... OFFSET 10 ROWS FETCH FIRST 5 ROWS ONLY` (SQL:2008 standard) |
| Oracle (old) | `WHERE ROWNUM <= 5` |

```sql
-- Standard SQL, works in PostgreSQL, Oracle 12c+, DB2 and SQL Server 2012+
SELECT prod_name, prod_price
FROM Products
ORDER BY prod_price DESC
FETCH FIRST 5 ROWS ONLY;

-- PostgreSQL 13+ and Oracle: include rows tied with the 5th
SELECT prod_name, prod_price
FROM Products
ORDER BY prod_price DESC
FETCH FIRST 5 ROWS WITH TIES;
```

## Comments

```sql
SELECT prod_name  -- a single-line comment
FROM Products;
/* a multi-line
   comment */
```

`#` comments work in MySQL only.

## Beyond the book / Senior perspective

- **No `ORDER BY`, no order.** Tables are unordered sets. A query may return rows in insertion order today and in a different order after a `VACUUM`, a parallel plan or an index change. Always pair `LIMIT`/`FETCH FIRST` with an `ORDER BY` on a unique key.
- **`OFFSET` pagination gets slow.** `OFFSET 100000` still reads and throws away 100,000 rows. Use keyset (seek) pagination: `WHERE (created_at, id) < ($1, $2) ORDER BY created_at DESC, id DESC LIMIT 20`. See [[Keyset Pagination vs Offset]].
- **Select only what you need.** Narrow column lists allow covering indexes and index-only scans ([[SQL Indexes Deep Dive]]) and send less data over the network.
- **`DISTINCT` is often a smell.** If you need `DISTINCT` to fix duplicates, check whether a join is fanning out rows ([[Joining Tables]]); `EXISTS` is usually the right fix.

## Interview questions

> [!question]- How do you return the top 10 rows in standard SQL?
> `ORDER BY <cols> FETCH FIRST 10 ROWS ONLY`. `LIMIT` is a widely supported extension (PostgreSQL, MySQL, SQLite), `TOP` is SQL Server. Always include an `ORDER BY`, ideally ending on a unique column so ties are deterministic.

> [!question]- Why is `SELECT *` discouraged in production code?
> It reads and transfers unused columns, prevents index-only scans, and makes the code fragile when the schema changes (new or reordered columns, name clashes in joins).

> [!question]- Does `DISTINCT` apply to one column or the whole row?
> The whole selected row. `SELECT DISTINCT a, b` removes rows where both `a` and `b` are equal.

> [!question]- Why is `LIMIT ... OFFSET` pagination slow on deep pages, and what's the alternative?
> The database must produce and discard all skipped rows. Keyset pagination filters on the last seen sort key (`WHERE id > $last ORDER BY id LIMIT n`), which uses an index and costs the same on every page.

## Related
- [[SQL ANSI]] · [[Databases Index]]
- [[Sorting Data]] · [[Filtering Data]] · [[Keyset Pagination vs Offset]] · [[Query Optimization & EXPLAIN]]

## Sources
- Ben Forta, *Sams Teach Yourself SQL in 10 Minutes a Day*, 5th ed., Lesson 2 ([O'Reilly](https://learning.oreilly.com/library/view/sams-teach-yourself/9780135182925/))
- [PostgreSQL docs: LIMIT and OFFSET](https://www.postgresql.org/docs/current/queries-limit.html) and [SELECT (FETCH FIRST, WITH TIES)](https://www.postgresql.org/docs/current/sql-select.html)
- [SQL Server docs: ORDER BY with OFFSET-FETCH](https://learn.microsoft.com/en-us/sql/t-sql/queries/select-order-by-clause-transact-sql)
- [Use The Index, Luke: Paging through results](https://use-the-index-luke.com/sql/partial-results/fetch-next-page)
