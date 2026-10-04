---
title: "Summarizing Data"
tags: [databases, databases/sql]
aliases: [Aggregate Functions, COUNT, SUM, AVG, MIN, MAX, FILTER clause]
updated: 2026-09-27
---
# Summarizing Data

<Note>
**TL;DR**

Aggregate functions (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`) collapse many rows into one value. All of them **ignore `NULL`s**, except `COUNT(*)`, which counts rows. `DISTINCT` inside an aggregate works on unique values only. On an empty set, `COUNT` returns `0` but `SUM`/`AVG`/`MIN`/`MAX` return `NULL`.
</Note>

> Course: SQL in 10 Minutes, Lesson 9 · Prev: [Using Data Manipulation Functions](/Software%20Engineering/Databases/SQL%20ANSI/Using%20Data%20Manipulation%20Functions) · Next: [Grouping Data](/Software%20Engineering/Databases/SQL%20ANSI/Grouping%20Data)

## Aggregate functions

**Aggregate function:** a function that operates on a set of rows and returns a single value.

| Function | Description |
|---|---|
| `AVG()` | Average of a column's values |
| `MAX()` | Highest value |
| `MIN()` | Lowest value |
| `COUNT()` | Number of rows (or of non-null values) |
| `SUM()` | Sum of the values |

It's often necessary to **summarize** data without retrieving all of it (how many customers, the average price, the latest order). Aggregates do this inside the database, which is far cheaper than shipping rows to the application.

<Note>
Aggregates can work on expressions, not just columns: `AVG(prod_price * quantity)`.
</Note>

## The `AVG()` function

Returns the average of a column by summing the values and dividing by the number of non-null values. It can average all rows or, with `WHERE`, specific rows.

`NULL` values are **ignored** by `AVG()` (they're neither added nor counted).

### Example 1

```sql
SELECT AVG(prod_price) AS avg_price
FROM Products;
```

**Analysis:** returns one value, `avg_price`: the average price of all products.

## The `COUNT()` function

Counts rows. Two forms:
1. `COUNT(*)` counts **all rows**, whatever the columns contain.
2. `COUNT(column)` counts rows that have a **non-null** value in that column.

### Example 2

```sql
SELECT COUNT(*) AS num_cust
FROM Customers;
```

**Analysis:** counts all customers, regardless of values.

### Example 3

```sql
SELECT COUNT(cust_email) AS num_cust
FROM Customers;
```

**Analysis:** counts only customers with an email address: `3` of the 5 customers.

## The `MAX()` function

Returns the highest value in a column; a column (or expression) must be specified.

<Note>
**`MAX()` with non-numeric types**

Most DBMSs allow `MAX()` on text and dates. On text it returns the value that would sort **last** (per the collation). `NULL`s are ignored.
</Note>

### Example 4

```sql
SELECT MAX(prod_price) AS max_price
FROM Products;
```

**Analysis:** the price of the most expensive item.

## The `MIN()` function

The opposite of `MAX()`: the lowest value. On text it returns the value that would sort **first**. `NULL`s are ignored.

### Example 5

```sql
SELECT MIN(prod_price) AS min_price
FROM Products;
```

**Analysis:** the price of the cheapest item.

## The `SUM()` function

Returns the total of the values in a column. `NULL`s are ignored.

### Example 6

```sql
SELECT SUM(item_price * quantity) AS total_price
FROM OrderItems
WHERE order_num = 20005;
```

**Analysis:** sums the expanded prices of all items in order `20005`; the `WHERE` makes sure only that order's items are included.

## Aggregates on distinct values

All five aggregates can be used two ways:
1. On all rows: specify `ALL` or nothing (`ALL` is the default).
2. On unique values only: specify `DISTINCT`.

<Warning>
**`DISTINCT` rules**

`DISTINCT` can only be used with `COUNT()` when a column is named: `COUNT(DISTINCT col)` works, `COUNT(DISTINCT *)` does not. `DISTINCT` with `MIN()`/`MAX()` is legal but pointless.
</Warning>

Some DBMSs support extra arguments (e.g. SQL Server's `TOP`/`TOP PERCENT`) for calculations on subsets; check your docs.

### Example 7

```sql
SELECT AVG(DISTINCT prod_price) AS avg_price
FROM Products
WHERE vend_id = 'DLL01';
```

**Analysis:** the average is higher with `DISTINCT` because several items share the same low price; counting that price once raises the average.

## Combining aggregate functions

### Example 8

```sql
SELECT COUNT(*)        AS num_items,
       MIN(prod_price) AS price_min,
       MAX(prod_price) AS price_max,
       AVG(prod_price) AS price_avg
FROM Products;
```

**Analysis:** four aggregates in one statement, returning four values in one row.

## Conditional aggregates

Count or sum only some rows, in one pass, without separate queries:

```sql
-- Standard SQL:2003 FILTER clause (PostgreSQL 9.4+, SQLite 3.30+)
SELECT COUNT(*)                                   AS all_customers,
       COUNT(*) FILTER (WHERE cust_email IS NULL) AS without_email,
       COUNT(*) FILTER (WHERE cust_state = 'IL')  AS in_illinois
FROM Customers;

-- Portable version (MySQL, SQL Server, Oracle)
SELECT COUNT(*) AS all_customers,
       SUM(CASE WHEN cust_email IS NULL THEN 1 ELSE 0 END) AS without_email,
       SUM(CASE WHEN cust_state = 'IL' THEN 1 ELSE 0 END)  AS in_illinois
FROM Customers;
```

## NULL and empty-set behaviour

| Input | `COUNT(*)` | `COUNT(col)` | `SUM(col)` | `AVG(col)` | `MAX(col)` |
|---|---|---|---|---|---|
| No rows | 0 | 0 | `NULL` | `NULL` | `NULL` |
| Rows, all `col` NULL | n | 0 | `NULL` | `NULL` | `NULL` |

So `SELECT SUM(amount) FROM payments WHERE user_id = 42` can return `NULL`, not `0`. Use `COALESCE(SUM(amount), 0)`.

`AVG` ignoring NULLs can surprise you: if 2 of 4 ratings are NULL, the average is over 2 values. If NULL should mean 0, use `AVG(COALESCE(rating, 0))`.

## Beyond the book / Senior perspective

- **Aggregates per group** need `GROUP BY` ([Grouping Data](/Software%20Engineering/Databases/SQL%20ANSI/Grouping%20Data)). Aggregates that keep every row (running totals, ranks, "percent of total") are **window functions**, also covered there.
- **`COUNT(*)` on a huge table is slow** in PostgreSQL (MVCC means it must check row visibility; an index-only scan helps if the visibility map is fresh). For dashboards, an estimate is often enough: `SELECT reltuples FROM pg_class WHERE relname = 'orders';`.
- **`COUNT(DISTINCT ...)` is expensive** at scale (sort or hash of all values). Approximate algorithms (HyperLogLog, e.g. the `hll` extension, BigQuery `APPROX_COUNT_DISTINCT`) trade exactness for speed.
- **Integer overflow and precision:** `SUM` of `integer` returns `bigint` in PostgreSQL, and `AVG` of integers returns `numeric`. In SQL Server, `AVG` of an `int` column returns an **int** (truncated); cast first.
- Other useful aggregates: `STRING_AGG` (PostgreSQL, SQL Server 2017+), `GROUP_CONCAT` (MySQL/SQLite), `ARRAY_AGG`, `JSON_AGG`, `BOOL_AND`/`BOOL_OR`, `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY x)` for medians.

## Interview questions

<Accordion title="`COUNT(*)` vs `COUNT(col)` vs `COUNT(1)`?">
`COUNT(*)` and `COUNT(1)` both count rows (identical performance in modern DBMSs). `COUNT(col)` counts rows where `col` is not NULL.
</Accordion>

<Accordion title="What does `SUM` return when no rows match?">
`NULL`, not 0. Wrap with `COALESCE(SUM(x), 0)`. `COUNT` returns 0.
</Accordion>

<Accordion title="How do aggregates treat NULLs?">
They ignore them (except `COUNT(*)`). That changes averages: `AVG` divides by the number of non-null values.
</Accordion>

<Accordion title="How do you count several conditions in one query?">
Conditional aggregation: `COUNT(*) FILTER (WHERE ...)` in PostgreSQL/SQLite, or `SUM(CASE WHEN ... THEN 1 ELSE 0 END)` everywhere.
</Accordion>

<Accordion title="Why is `SELECT COUNT(*) FROM big_table` slow in PostgreSQL?">
MVCC: each row version's visibility must be checked, so it scans the table or an index (index-only scans help when the visibility map is up to date). Use estimates from `pg_class.reltuples` or maintained counters when exact numbers aren't required.
</Accordion>

## Related
- [SQL ANSI](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20ANSI) · [Databases Index](/Software%20Engineering/Databases/Databases%20Index)
- [Grouping Data](/Software%20Engineering/Databases/SQL%20ANSI/Grouping%20Data) (GROUP BY, HAVING, window functions) · [Creating Calculated Fields](/Software%20Engineering/Databases/SQL%20ANSI/Creating%20Calculated%20Fields) · [Query Optimization & EXPLAIN](/Software%20Engineering/Databases/Core%20Concepts/Query%20Optimization%20%26%20EXPLAIN)

## Sources
- Ben Forta, *Sams Teach Yourself SQL in 10 Minutes a Day*, 5th ed., Lesson 9 ([O'Reilly](https://learning.oreilly.com/library/view/sams-teach-yourself/9780135182925/))
- [PostgreSQL docs: Aggregate functions](https://www.postgresql.org/docs/current/functions-aggregate.html)
- [PostgreSQL docs: Aggregate expressions (FILTER)](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-AGGREGATES)
- [PostgreSQL wiki: Slow counting](https://wiki.postgresql.org/wiki/Slow_Counting)
- [SQL Server docs: AVG (return types)](https://learn.microsoft.com/en-us/sql/t-sql/functions/avg-transact-sql)
