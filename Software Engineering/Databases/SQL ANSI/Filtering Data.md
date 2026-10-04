---
title: "Filtering Data"
tags: [databases, databases/sql]
aliases: [WHERE, WHERE clause, Filtering, IS NULL, NULL comparison]
updated: 2026-09-27
---
# Filtering Data

<Note>
**TL;DR**

`WHERE` keeps only rows whose condition is **true**. Use comparison operators (`=`, `<>`, `<`, `>`, `<=`, `>=`), ranges (`BETWEEN`), lists (`IN`), patterns (`LIKE`) and `IS [NOT] NULL`. The big trap: `col = NULL` is never true, because any comparison with `NULL` is **unknown**; always use `IS NULL`.
</Note>

> Course: SQL in 10 Minutes, Lesson 4 · Prev: [Sorting Data](/Software%20Engineering/Databases/SQL%20ANSI/Sorting%20Data) · Next: [Advanced Filtering Data](/Software%20Engineering/Databases/SQL%20ANSI/Advanced%20Filtering%20Data)

## Using the `WHERE` clause

`WHERE` goes right after `FROM` (and before `ORDER BY`).

| Operator | Example | Description |
|---|---|---|
| `=` | `prod_name = 'Card'` | Equality |
| `<>` | `prod_name <> 'Card'` | Non-equality (standard) |
| `!=` | `prod_id != 1` | Non-equality (supported almost everywhere, not standard) |
| `<` | `prod_price < 1400` | Less than |
| `<=` | `prod_price <= 1400` | Less than or equal to |
| `>` | `prod_price > 2600` | Greater than |
| `>=` | `prod_price >= 2600` | Greater than or equal to |
| `!<` | `prod_price !< 1400` | Not less than. **SQL Server only**; elsewhere write `>=` |
| `!>` | `prod_price !> 2600` | Not greater than. **SQL Server only**; elsewhere write `<=` |
| `BETWEEN` | `prod_price BETWEEN 5 AND 10` | Between two values, **inclusive** of both ends |
| `IS NULL` | `prod_name IS NULL` | Is a `NULL` value |
| `IS NOT NULL` | `cust_email IS NOT NULL` | Has a value |
| `IN` | `vend_id IN ('DLL01','BRS01')` | Matches any value in a list, see [Advanced Filtering Data](/Software%20Engineering/Databases/SQL%20ANSI/Advanced%20Filtering%20Data) |
| `LIKE` | `prod_name LIKE 'Fish%'` | Pattern match, see [Using Wildcard Filtering](/Software%20Engineering/Databases/SQL%20ANSI/Using%20Wildcard%20Filtering) |

### Example 1: searching for equality

```sql
SELECT prod_name, prod_price
FROM Products
WHERE prod_price = 3.49;
```

**Analysis:** retrieves two columns from `Products`, but only rows whose `prod_price` is `3.49`.

### Example 2: checking against a single value

```sql
SELECT prod_name, prod_price
FROM Products
WHERE prod_price < 10;
```

**Analysis:** only rows with a `prod_price` less than `10` are returned.

### Example 3: non-matches

```sql
SELECT vend_id, prod_name
FROM Products
WHERE vend_id <> 'DLL01';
```

Strings are quoted with **single** quotes. Numbers are not quoted. Double quotes mean *identifiers* (column/table names) in standard SQL and PostgreSQL.

### Example 4: range of values

```sql
SELECT prod_name, prod_price
FROM Products
WHERE prod_price BETWEEN 5 AND 10;
```

`BETWEEN` includes both 5 and 10.

<Warning>
**`BETWEEN` with timestamps**

`order_date BETWEEN '2020-01-01' AND '2020-12-31'` misses anything on Dec 31 after midnight if the column is a timestamp. Use a half-open range: `order_date >= '2020-01-01' AND order_date < '2021-01-01'`.
</Warning>

### Example 5: no value

```sql
SELECT cust_name
FROM Customers
WHERE cust_email IS NULL;
```

## The `= NULL` pitfall

`NULL` means "unknown / no value". Any comparison with it (`=`, `<>`, `<`...) returns **UNKNOWN**, not true or false, and `WHERE` only keeps rows where the condition is **TRUE**.

```sql
SELECT * FROM Customers WHERE cust_email = NULL;   -- always 0 rows
SELECT * FROM Customers WHERE cust_email <> NULL;  -- also 0 rows
SELECT * FROM Customers WHERE cust_email IS NULL;  -- correct
```

A second consequence: `WHERE vend_id <> 'DLL01'` does **not** return rows where `vend_id` is `NULL`. If you want them, say so: `WHERE vend_id <> 'DLL01' OR vend_id IS NULL`, or in PostgreSQL/SQL Server 2022+/standard SQL: `WHERE vend_id IS DISTINCT FROM 'DLL01'`.

## Beyond the book / Senior perspective

- **Three-valued logic** (TRUE, FALSE, UNKNOWN) drives many bugs: `NOT IN` with a `NULL` in the list, outer joins filtered in `WHERE`, `CHECK` constraints that pass on `NULL`. Full treatment in [Advanced Filtering Data](/Software%20Engineering/Databases/SQL%20ANSI/Advanced%20Filtering%20Data).
- **`COALESCE(col, default)`** returns the first non-null argument, useful for display and arithmetic (`price * COALESCE(discount, 0)`), but wrapping an indexed column in a function in `WHERE` can stop the index being used.
- **Sargable predicates.** A condition is *sargable* (Search ARGument ABLE) when the database can use an index for it. `WHERE order_date >= '2020-01-01'` is sargable; `WHERE YEAR(order_date) = 2020` is not (unless you have an expression index). Keep the indexed column alone on one side of the operator. See [SQL Indexes Deep Dive](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20Indexes%20Deep%20Dive).
- **Filter in the database, not the app.** Fetching all rows and filtering in code wastes I/O and network and can't use indexes.
- **Never build `WHERE` clauses by string concatenation** with user input; use parameters (`WHERE cust_id = $1`). See [Advanced Filtering Data](/Software%20Engineering/Databases/SQL%20ANSI/Advanced%20Filtering%20Data) for SQL injection.

## Interview questions

<Accordion title="Why does `WHERE col = NULL` return no rows?">
Comparisons with `NULL` evaluate to UNKNOWN, and `WHERE` keeps only TRUE rows. Use `IS NULL` / `IS NOT NULL`, or `IS [NOT] DISTINCT FROM` for null-safe comparison.
</Accordion>

<Accordion title="Is `BETWEEN` inclusive?">
Yes, on both ends. For timestamps prefer a half-open range `>= start AND < next_start`.
</Accordion>

<Accordion title="`<>` or `!=`?">
`<>` is the SQL standard; `!=` is supported by virtually every DBMS. `!<` and `!>` are SQL Server-only.
</Accordion>

<Accordion title="Does `WHERE status <> 'done'` return rows where status is NULL?">
No. `NULL <> 'done'` is UNKNOWN. Add `OR status IS NULL`, or use `status IS DISTINCT FROM 'done'`.
</Accordion>

<Accordion title="What makes a predicate sargable?">
The indexed column appears bare on one side of a comparison (`col = ?`, `col > ?`, `col LIKE 'abc%'`). Functions on the column, implicit casts, or leading wildcards usually prevent index use.
</Accordion>

## Related
- [SQL ANSI](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20ANSI) · [Databases Index](/Software%20Engineering/Databases/Databases%20Index)
- [Advanced Filtering Data](/Software%20Engineering/Databases/SQL%20ANSI/Advanced%20Filtering%20Data) · [Using Wildcard Filtering](/Software%20Engineering/Databases/SQL%20ANSI/Using%20Wildcard%20Filtering) · [SQL Indexes Deep Dive](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20Indexes%20Deep%20Dive) · [Query Optimization & EXPLAIN](/Software%20Engineering/Databases/Core%20Concepts/Query%20Optimization%20%26%20EXPLAIN)

## Sources
- Ben Forta, *Sams Teach Yourself SQL in 10 Minutes a Day*, 5th ed., Lesson 4 ([O'Reilly](https://learning.oreilly.com/library/view/sams-teach-yourself/9780135182925/))
- [PostgreSQL docs: Comparison functions and operators (IS NULL, IS DISTINCT FROM)](https://www.postgresql.org/docs/current/functions-comparison.html)
- [SQL Server docs: Comparison operators (!&lt;, !&gt;)](https://learn.microsoft.com/en-us/sql/t-sql/language-elements/comparison-operators-transact-sql)
- [Use The Index, Luke: Functions in WHERE](https://use-the-index-luke.com/sql/where-clause/functions)
