---
title: "Combining Queries"
tags: [databases, databases/sql]
aliases: [UNION, UNION ALL, INTERSECT, EXCEPT, MINUS, Compound Queries, Set Operators]
updated: 2026-09-27
---
# Combining Queries

<Note>
**TL;DR**

Set operators stack the results of several `SELECT`s **vertically**: `UNION` (all rows, duplicates removed), `UNION ALL` (all rows, duplicates kept, faster), `INTERSECT` (rows in both), `EXCEPT`/`MINUS` (rows in the first but not the second). Each `SELECT` must return the same number of columns with compatible types, and one `ORDER BY` at the end sorts the whole result. Default to `UNION ALL` unless you really need de-duplication.
</Note>

> Course: SQL in 10 Minutes, Lesson 14 · Prev: [Creating Advanced Joins](/Software%20Engineering/Databases/SQL%20ANSI/Creating%20Advanced%20Joins) · Next: [Inserting Data](/Software%20Engineering/Databases/SQL%20ANSI/Inserting%20Data)

## Understanding combined queries

Most queries have one `SELECT`. SQL also lets you run several `SELECT`s and return their results as one result set. These are called **unions** or **compound queries**.

Combining two queries on the same table usually does the same as one query with several `WHERE` conditions; any `SELECT` with multiple `WHERE` conditions can be written as a combined query.

When to use them:
1. To return similarly structured data from **different tables** in one query.
2. To run several queries against one table and return one result.

## Creating combined queries with `UNION`

### Using `UNION`

You need a report of all customers in Illinois, Indiana and Michigan, plus all `Fun4All` locations regardless of state. The individual statements:

```sql
SELECT cust_name, cust_contact, cust_email
FROM Customers
WHERE cust_state IN ('IL','IN','MI');
```

```sql
SELECT cust_name, cust_contact, cust_email
FROM Customers
WHERE cust_name = 'Fun4All';
```

**Analysis:** the first returns customers in the three states; the second finds all `Fun4All` locations. One row appears in both outputs because it meets both conditions.

#### Example 1

```sql
SELECT cust_name, cust_contact, cust_email
FROM Customers
WHERE cust_state IN ('IL','IN','MI')
UNION
SELECT cust_name, cust_contact, cust_email
FROM Customers
WHERE cust_name = 'Fun4All';
```

**Analysis:** both statements separated by `UNION`. The DBMS runs both and combines the output into one result set. The equivalent single query is `WHERE cust_state IN ('IL','IN','MI') OR cust_name = 'Fun4All'`.

<Note>
**`UNION` limits**

There's no standard limit on how many `SELECT`s you can combine.
</Note>

## `UNION` rules

1. A `UNION` has two or more `SELECT`s separated by `UNION` (four statements need three `UNION` keywords).
2. Each query must return the **same number of columns**, expressions or aggregates, and they're matched **by position**, not by name.
3. Column types must be **compatible** (the DBMS can implicitly convert them, e.g. different numeric types). They needn't have the same names.
4. The result's column names come from the **first** `SELECT`.

## Including or eliminating duplicate rows

Run separately, the first `SELECT` returns three rows and the second two. Combined with `UNION`, you get four, not five: `UNION` **removes duplicates** (the Fun4All location in Indiana matched both). That's the same as multiple `WHERE` conditions in one query would do.

To keep all occurrences, use `UNION ALL`.

### Example 2

```sql
SELECT cust_name, cust_contact, cust_email
FROM Customers
WHERE cust_state IN ('IL','IN','MI')
UNION ALL
SELECT cust_name, cust_contact, cust_email
FROM Customers
WHERE cust_name = 'Fun4All';
```

**Analysis:** with `UNION ALL` duplicates are not removed, so five rows come back, one of them twice. `UNION ALL` does something `WHERE` can't: return every occurrence of every match.

<Tip>
**Performance**

`UNION` must sort or hash the whole result to remove duplicates. `UNION ALL` just appends. If the branches can't overlap (different tables, disjoint conditions) or duplicates are fine, use `UNION ALL`.
</Tip>

## Sorting combined query results

Only **one** `ORDER BY` is allowed, after the last `SELECT`. Sorting parts of a result differently makes no sense.

```sql
SELECT cust_name, cust_contact, cust_email
FROM Customers
WHERE cust_state IN ('IL','IN','MI')
UNION
SELECT cust_name, cust_contact, cust_email
FROM Customers
WHERE cust_name = 'Fun4All'
ORDER BY cust_name, cust_contact;
```

**Analysis:** although it looks like part of the last `SELECT`, the DBMS uses this `ORDER BY` to sort the entire combined result. To sort or limit an individual branch, wrap it in parentheses or a subquery: `(SELECT ... ORDER BY x LIMIT 5) UNION ALL (SELECT ...)`.

<Note>
**Other set operators**

`EXCEPT` (Oracle: `MINUS`) returns rows in the first query but not the second; `INTERSECT` returns rows in both. The book says they're rarely used because joins can do the same. They're more useful than that: they compare **whole rows**, treat `NULL`s as equal (joins don't), and are concise for data reconciliation. MySQL only added them in **8.0.31** (2022).
</Note>

<Note>
`UNION` is especially useful for combining data from several tables, even with mismatched column names, using aliases to give one set of result columns.
</Note>

## Set operators summary

![Untitled 42](/Attachments/Untitled%2042.png)

### `UNION`

Returns rows from both queries. `UNION` alone returns distinct rows; `UNION ALL` returns everything. Useful for one combined, sorted list from two sources, e.g. a list of names from `Vendor` and `Customer` tagged with their origin:

```sql
SELECT 'Vendor' AS source, V.Name
FROM Vendor V
UNION
SELECT 'Customer', C.Name
FROM Customer C
ORDER BY Name;
```

(The original used curly quotes `‘Vendor’`, which SQL won't accept; string literals need straight single quotes.)

### `INTERSECT`

Returns distinct rows that appear in **both** queries. Vendors that are also customers:

```sql
SELECT V.Name
FROM Vendor V
INTERSECT
SELECT C.Name
FROM Customer C
ORDER BY Name;
```

The same with an `INNER JOIN`:

```sql
SELECT DISTINCT V.Name
FROM Vendor V
INNER JOIN Customer C ON V.Name = C.Name
ORDER BY V.Name;
```

### `EXCEPT`

Returns distinct rows from the left query that are **not** in the right query. Vendors that are not customers:

```sql
SELECT V.Name
FROM Vendor V
EXCEPT
SELECT C.Name
FROM Customer C
ORDER BY Name;
```

The equivalent with an outer join (anti-join):

```sql
SELECT DISTINCT V.Name
FROM Vendor V
LEFT OUTER JOIN Customer C ON V.Name = C.Name
WHERE C.Name IS NULL
ORDER BY V.Name;
```

Precedence: `INTERSECT` binds tighter than `UNION` and `EXCEPT`; use parentheses when mixing.

## Unions vs joins

| JOIN | UNION |
|---|---|
| Combines data into new **columns** (side by side) | Combines data into new **rows** (stacked) |
| Columns from each table can have different types | Corresponding columns must have compatible types |
| Combines tables based on a matching condition | Combines the result sets of two or more `SELECT`s |
| Number of columns from each table can differ | Each `SELECT` must return the same number of columns |
| May return duplicate rows | `UNION` returns distinct rows; `UNION ALL` keeps duplicates |

{/* diagram source: /diagrams/diagram-132.mmd */}
![diagram](/diagrams/diagram-132.svg)

## Beyond the book / Senior perspective

- **Partitioned or archived data** is a common `UNION ALL` use: `SELECT ... FROM orders_2025 UNION ALL SELECT ... FROM orders_2026`. Declarative partitioning ([SQL Table Partitioning](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20Table%20Partitioning)) does this for you.
- **Rewriting `OR` as `UNION ALL`** can help when the `OR` spans columns with separate indexes (each branch uses its own index). Check whether your planner already does a bitmap OR.
- **Data diffing:** `(A EXCEPT B) UNION ALL (B EXCEPT A)` lists rows that differ between two tables or environments.
- Recursive CTEs use `UNION [ALL]` to join the anchor and recursive parts ([Understanding Advanced SQL Features](/Software%20Engineering/Databases/SQL%20ANSI/Understanding%20Advanced%20SQL%20Features)).

## Interview questions

<Accordion title="UNION vs UNION ALL?">
`UNION` removes duplicate rows (costing a sort or hash); `UNION ALL` keeps them and is faster. Use `UNION ALL` unless de-duplication is required.
</Accordion>

<Accordion title="What are the rules for a UNION?">
Same number of columns in each `SELECT`, matched by position, with compatible types. Names come from the first `SELECT`. One `ORDER BY`, at the end.
</Accordion>

<Accordion title="JOIN vs UNION?">
A join combines columns from related rows horizontally; a union stacks rows from several queries vertically.
</Accordion>

<Accordion title="How do you find rows present in table A but not in table B?">
`SELECT ... FROM A EXCEPT SELECT ... FROM B` (whole-row comparison, NULL-safe), or `NOT EXISTS` / `LEFT JOIN ... IS NULL` on a key.
</Accordion>

## Related
- [SQL ANSI](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20ANSI) · [Databases Index](/Software%20Engineering/Databases/Databases%20Index)
- [Creating Advanced Joins](/Software%20Engineering/Databases/SQL%20ANSI/Creating%20Advanced%20Joins) · [Working with Subqueries](/Software%20Engineering/Databases/SQL%20ANSI/Working%20with%20Subqueries) · [SQL Table Partitioning](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20Table%20Partitioning) · [Understanding Advanced SQL Features](/Software%20Engineering/Databases/SQL%20ANSI/Understanding%20Advanced%20SQL%20Features)

## Sources
- Ben Forta, *Sams Teach Yourself SQL in 10 Minutes a Day*, 5th ed., Lesson 14 ([O'Reilly](https://learning.oreilly.com/library/view/sams-teach-yourself/9780135182925/))
- [PostgreSQL docs: Combining queries (UNION, INTERSECT, EXCEPT)](https://www.postgresql.org/docs/current/queries-union.html)
- [MySQL docs: Set operations (INTERSECT/EXCEPT since 8.0.31)](https://dev.mysql.com/doc/refman/8.4/en/set-operations.html)
