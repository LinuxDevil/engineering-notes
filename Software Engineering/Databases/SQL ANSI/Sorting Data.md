---
title: "Sorting Data"
tags: [databases, databases/sql]
aliases: [ORDER BY, Sorting, NULLS FIRST, NULLS LAST]
updated: 2026-09-27
---
# Sorting Data

<Note>
**TL;DR**

`ORDER BY` sorts the result; it's the **only** way to guarantee row order. You can sort by several columns (later columns break ties), by column position (`ORDER BY 2`), and per column direction (`DESC` applies only to the column right before it). Where `NULL`s go and how text is compared (collation) depend on the DBMS, so be explicit with `NULLS FIRST/LAST` when it matters.
</Note>

> Course: SQL in 10 Minutes, Lesson 3 · Prev: [Retrieving Information](/Software%20Engineering/Databases/SQL%20ANSI/Retrieving%20Information) · Next: [Filtering Data](/Software%20Engineering/Databases/SQL%20ANSI/Filtering%20Data)

## The `ORDER BY` clause

**What is a clause?** SQL statements are made of clauses, some required and some optional. A clause is usually a keyword plus supplied data, e.g. the `FROM` clause of a `SELECT`.

`ORDER BY` must be the **last** clause of a `SELECT` (before `LIMIT`/`FETCH`).

### Example 1: sorting by one column

```sql
SELECT prod_name
FROM Products
ORDER BY prod_name;
```

**Analysis:** same as the earlier statement, plus an `ORDER BY` that sorts the rows by `prod_name`. You can sort by a column you don't select (most DBMSs allow it, except with `DISTINCT`).

### Example 2: sorting by multiple columns

```sql
SELECT prod_id, prod_price, prod_name
FROM Products
ORDER BY prod_price, prod_name;
```

**Analysis:** the sort sequence is exactly as specified. Rows are sorted by `prod_name` **only** when several rows have the same `prod_price`. If all prices were unique, `prod_name` would never be used.

### Example 3: sorting by column position

```sql
SELECT prod_id, prod_price, prod_name
FROM Products
ORDER BY 2, 3;
```

**Analysis:** same output as Example 2. `ORDER BY 2` means "the second column in the `SELECT` list" (`prod_price`), then the third (`prod_name`). Handy in ad-hoc queries, fragile in code: reorder the select list and the sort silently changes.

### Example 4: specifying sort direction

Ascending (`ASC`, A to Z) is the default. Use `DESC` for descending.

```sql
-- Most expensive first, then by product name
SELECT prod_id, prod_price, prod_name
FROM Products
ORDER BY prod_price DESC, prod_name;
```

**Analysis:** `DESC` only applies to the column name directly before it. `prod_price` is descending, `prod_name` (within each price) is still ascending. To sort several columns descending, write `DESC` after each.

<Note>
**Comment syntax**

The original example used `#` comments, which only MySQL accepts. Standard SQL uses `--` and `/* ... */`.
</Note>

## Where do NULLs go?

| DBMS | Default for `ASC` |
|---|---|
| PostgreSQL, Oracle | `NULL`s **last** (treated as larger than any value) |
| MySQL, SQL Server, SQLite | `NULL`s **first** |

```sql
-- Standard SQL, supported by PostgreSQL, Oracle, SQLite 3.30+
SELECT cust_name, cust_email
FROM Customers
ORDER BY cust_email NULLS LAST;
```

## Beyond the book / Senior perspective

- **Collation decides text order.** Is `'a' < 'B'`? Is `'é'` next to `'e'`? That's set by the column or database collation (`COLLATE "C"`, ICU collations in PostgreSQL, `utf8mb4_0900_ai_ci` in MySQL). Case-insensitive collations also change equality. Upgrading the OS C library can change collation order and corrupt text indexes in PostgreSQL; that is why PostgreSQL 17 added a built-in, stable `C.UTF-8`-style provider.
- **Make the order deterministic.** `ORDER BY created_at` with duplicate timestamps can return ties in a different order each run, which breaks pagination. Add a unique tiebreaker: `ORDER BY created_at DESC, id DESC`.
- **Indexes can avoid a sort.** A B-tree index on `(prod_price, prod_name)` returns rows already sorted, so `ORDER BY prod_price, prod_name LIMIT 10` can stop after 10 index entries instead of sorting the table. Mixed directions need a matching index (`(prod_price DESC, prod_name ASC)`). See [SQL Indexes Deep Dive](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20Indexes%20Deep%20Dive).
- **Big sorts spill to disk.** In PostgreSQL, `EXPLAIN ANALYZE` shows `Sort Method: external merge Disk: ...` when `work_mem` is too small. See [Query Optimization & EXPLAIN](/Software%20Engineering/Databases/Core%20Concepts/Query%20Optimization%20%26%20EXPLAIN).

## Interview questions

<Accordion title="If I don't use ORDER BY, what order are rows returned in?">
Undefined. It often looks like insertion or primary key order, but it can change with the plan, parallelism, updates or maintenance. Only `ORDER BY` guarantees order.
</Accordion>

<Accordion title="In `ORDER BY a DESC, b`, how is `b` sorted?">
Ascending. `DESC` applies only to the column directly before it.
</Accordion>

<Accordion title="How do you control where NULLs appear?">
`NULLS FIRST` / `NULLS LAST` (PostgreSQL, Oracle, SQLite). In MySQL/SQL Server, sort on an expression first, e.g. `ORDER BY (col IS NULL), col` in MySQL or `ORDER BY CASE WHEN col IS NULL THEN 1 ELSE 0 END, col`.
</Accordion>

<Accordion title="How can an index make ORDER BY ... LIMIT fast?">
If an index is ordered on the sort columns (and satisfies the WHERE), the database reads the index in order and stops after N rows, with no sort step at all.
</Accordion>

## Related
- [SQL ANSI](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20ANSI) · [Databases Index](/Software%20Engineering/Databases/Databases%20Index)
- [Retrieving Information](/Software%20Engineering/Databases/SQL%20ANSI/Retrieving%20Information) · [Filtering Data](/Software%20Engineering/Databases/SQL%20ANSI/Filtering%20Data) · [SQL Indexes Deep Dive](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20Indexes%20Deep%20Dive) · [Query Optimization & EXPLAIN](/Software%20Engineering/Databases/Core%20Concepts/Query%20Optimization%20%26%20EXPLAIN)

## Sources
- Ben Forta, *Sams Teach Yourself SQL in 10 Minutes a Day*, 5th ed., Lesson 3 ([O'Reilly](https://learning.oreilly.com/library/view/sams-teach-yourself/9780135182925/))
- [PostgreSQL docs: Sorting rows (ORDER BY)](https://www.postgresql.org/docs/current/queries-order.html)
- [PostgreSQL docs: Collation support](https://www.postgresql.org/docs/current/collation.html)
- [Use The Index, Luke: Indexing ORDER BY](https://use-the-index-luke.com/sql/sorting-grouping/indexed-order-by)
