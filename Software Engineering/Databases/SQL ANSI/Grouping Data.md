---
title: "Grouping Data"
tags: [databases, databases/sql]
aliases: [GROUP BY, HAVING, Window Functions, OVER, PARTITION BY, ROW_NUMBER, ROLLUP, GROUPING SETS]
updated: 2026-09-27
---
# Grouping Data

<Note>
**TL;DR**

`GROUP BY` splits rows into groups and computes aggregates **per group**; `HAVING` filters groups the way `WHERE` filters rows. `WHERE` runs before grouping, `HAVING` after. **Window functions** (`... OVER (PARTITION BY ... ORDER BY ...)`) compute aggregates, ranks and running totals **without collapsing rows**, and they're one of the most useful things a senior engineer can know in SQL.
</Note>

> Course: SQL in 10 Minutes, Lesson 10 · Prev: [Summarizing Data](/Software%20Engineering/Databases/SQL%20ANSI/Summarizing%20Data) · Next: [Working with Subqueries](/Software%20Engineering/Databases/SQL%20ANSI/Working%20with%20Subqueries)

## Understanding data grouping

Grouping lets you divide data into logical sets so you can run aggregate calculations on each set: products per vendor, orders per customer, revenue per month.

## Creating groups

Groups are created with the `GROUP BY` clause of a `SELECT`.

### Example 1

```sql
SELECT vend_id, COUNT(*) AS num_prods
FROM Products
GROUP BY vend_id;
```

**Analysis:** two columns: `vend_id` and `num_prods`, a calculated field. `GROUP BY` groups rows by `vend_id`, so `COUNT(*)` is computed once per vendor instead of once for the whole table: `BRS01` has 3 products, `DLL01` 4 and `FNG01` 2. You don't list the groups yourself; they're created automatically.

## Rules of `GROUP BY`

1. `GROUP BY` can contain as many columns as you want, so you can nest groups for finer control.
2. With several grouping columns, data is summarized at the combination of all of them (you don't get subtotals per level; for that see `ROLLUP` below).
3. Every `GROUP BY` item must be a retrieved column or a valid expression (not an aggregate). If you group by an expression, repeat the same expression. Output-column **aliases** in `GROUP BY` are not standard but PostgreSQL, MySQL and SQLite accept them; SQL Server and Oracle (before 23ai) do not.
4. Some DBMSs don't allow grouping on large-object types (SQL Server `text`/`ntext`/`image`, Oracle `CLOB`). Normal `VARCHAR`/`TEXT` columns are fine in modern DBMSs.
5. Every non-aggregate column in the `SELECT` must appear in `GROUP BY`. Exception (standard SQL:1999, PostgreSQL 9.1+, MySQL 5.7+): columns **functionally dependent** on the grouped columns, e.g. `GROUP BY customers.cust_id` lets you select `customers.cust_name` because `cust_id` is the primary key.
6. `NULL` values in the grouping column form **one group** together.
7. `GROUP BY` comes after `WHERE` and before `ORDER BY`.

<Warning>
**MySQL and `ONLY_FULL_GROUP_BY`**

Old MySQL let you select non-grouped columns and returned an arbitrary value from the group. Since 5.7 the `ONLY_FULL_GROUP_BY` mode (on by default) rejects that. Don't turn it off; fix the query (group by the column, aggregate it, or use `ANY_VALUE()` deliberately).
</Warning>

## Filtering groups: `HAVING`

`HAVING` is like `WHERE`, and accepts the same kinds of conditions. The difference: **`WHERE` filters rows, `HAVING` filters groups.**

### Example 2

```sql
SELECT cust_id, COUNT(*) AS orders
FROM Orders
GROUP BY cust_id
HAVING COUNT(*) >= 2;
```

**Analysis:** the `HAVING` clause keeps only groups with two or more orders. `WHERE` can't do this, because the filter uses the aggregate value of the group, not a value in a row.

### The difference between `HAVING` and `WHERE`

`WHERE` filters **before** data is grouped; `HAVING` filters **after**. Rows eliminated by `WHERE` are not in any group, which changes the aggregates and therefore which groups `HAVING` keeps.

<Note>
**Using `HAVING` and `WHERE`**

Without `GROUP BY`, most DBMSs treat `HAVING` like `WHERE` over one big group. Keep the distinction anyway: `HAVING` only with `GROUP BY`, `WHERE` for row-level filtering. Also put every condition that doesn't need an aggregate in `WHERE`; filtering early is cheaper.
</Note>

### Example 3

```sql
SELECT vend_id, COUNT(*) AS num_prods
FROM Products
WHERE prod_price >= 4
GROUP BY vend_id
HAVING COUNT(*) >= 2;
```

**Analysis:** `WHERE` keeps products priced at least 4, rows are grouped by `vend_id`, then `HAVING` keeps groups with 2 or more products. Without the `WHERE`, vendor `DLL01` (four products, all under 4) would also appear.

## Grouping and sorting

`GROUP BY` and `ORDER BY` are different, even though grouped output often looks sorted.

| `ORDER BY` | `GROUP BY` |
|---|---|
| Sorts the generated output | Groups rows. The output is **not guaranteed** to be in group order |
| Any columns may be used (even ones not selected) | Only selected columns or expressions may be used, and every selected non-aggregate column must be used |
| Never required | Required when mixing columns with aggregates |

If you want the groups sorted, add `ORDER BY`; never rely on `GROUP BY` to sort (a hash aggregate returns groups in any order).

### Example 4

```sql
SELECT order_num, COUNT(*) AS items
FROM OrderItems
GROUP BY order_num
HAVING COUNT(*) >= 3
ORDER BY items, order_num;
```

**Analysis:** groups by order number so `COUNT(*)` gives the items per order, `HAVING` keeps orders with 3 or more items, and `ORDER BY` sorts the output.

## `SELECT` clause ordering

Written order:

| Clause | Description | Required |
|---|---|---|
| `SELECT` | Columns or expressions to return | Yes |
| `FROM` | Table to retrieve data from | Only if selecting from a table |
| `WHERE` | Row-level filtering | No |
| `GROUP BY` | Group specification | Only if calculating aggregates by group |
| `HAVING` | Group-level filtering | No |
| `ORDER BY` | Output sort order | No |

**Logical processing order** (why aliases work in some clauses and not others):

```mermaid
flowchart LR
  F[FROM / JOIN] --> W[WHERE] --> G[GROUP BY] --> H[HAVING] --> S[SELECT<br/>+ window functions] --> D[DISTINCT] --> O[ORDER BY] --> L[LIMIT / FETCH]
```

## Subtotals: `ROLLUP`, `CUBE`, `GROUPING SETS`

```sql
-- Revenue per vendor per product, plus a subtotal per vendor and a grand total
SELECT p.vend_id, p.prod_id, SUM(oi.quantity * oi.item_price) AS revenue
FROM OrderItems oi
JOIN Products p ON p.prod_id = oi.prod_id
GROUP BY ROLLUP (p.vend_id, p.prod_id)
ORDER BY p.vend_id, p.prod_id;
```

Subtotal rows have `NULL` in the rolled-up columns; `GROUPING(col)` tells you whether a `NULL` is a subtotal marker or real data. `CUBE (a, b)` gives all combinations; `GROUPING SETS ((a), (b), ())` gives exactly the ones you list. Supported in PostgreSQL 9.5+, SQL Server, Oracle, DB2; MySQL has `WITH ROLLUP` only.

## Window functions

A window function computes over a set of rows **related to the current row**, and returns a value for **every** row. `GROUP BY` collapses; windows don't.

```sql
function(...) OVER (
  PARTITION BY ...   -- split into groups (optional)
  ORDER BY ...       -- order inside each partition (optional)
  ROWS/RANGE ...     -- the frame: which rows around the current one (optional)
)
```

### Ranking

```sql
SELECT prod_id, vend_id, prod_price,
       ROW_NUMBER() OVER (PARTITION BY vend_id ORDER BY prod_price DESC) AS rn,
       RANK()       OVER (PARTITION BY vend_id ORDER BY prod_price DESC) AS rnk,
       DENSE_RANK() OVER (PARTITION BY vend_id ORDER BY prod_price DESC) AS drnk
FROM Products;
```

| Price | `ROW_NUMBER` | `RANK` | `DENSE_RANK` |
|---|---|---|---|
| 11.99 | 1 | 1 | 1 |
| 8.99 | 2 | 2 | 2 |
| 8.99 | 3 | 2 | 2 |
| 4.99 | 4 | 4 | 3 |

### Top-N per group (classic interview question)

```sql
-- Most expensive 2 products per vendor
SELECT *
FROM (
  SELECT p.*,
         ROW_NUMBER() OVER (PARTITION BY vend_id ORDER BY prod_price DESC, prod_id) AS rn
  FROM Products p
) ranked
WHERE rn <= 2;
```

You can't put a window function in `WHERE` (it's computed after `WHERE`), hence the subquery. (Some engines, e.g. Snowflake, DuckDB, BigQuery, offer `QUALIFY rn <= 2`.)

### Aggregates as windows: share of total, running total

```sql
SELECT order_num,
       order_date,
       order_total,
       SUM(order_total) OVER ()                         AS grand_total,
       ROUND(100.0 * order_total / SUM(order_total) OVER (), 2) AS pct_of_total,
       SUM(order_total) OVER (ORDER BY order_date, order_num
                              ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total,
       AVG(order_total) OVER (ORDER BY order_date
                              ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)         AS moving_avg_7
FROM (
  SELECT o.order_num, o.order_date, SUM(oi.quantity * oi.item_price) AS order_total
  FROM Orders o JOIN OrderItems oi ON oi.order_num = o.order_num
  GROUP BY o.order_num, o.order_date
) t;
```

### Looking at neighbours: `LAG` / `LEAD`

```sql
-- Days between a customer's consecutive orders
SELECT cust_id, order_num, order_date,
       order_date - LAG(order_date) OVER (PARTITION BY cust_id ORDER BY order_date) AS days_since_prev
FROM Orders;
```

Other useful ones: `FIRST_VALUE`, `LAST_VALUE`, `NTH_VALUE`, `NTILE(4)` (quartiles), `PERCENT_RANK`, `CUME_DIST`.

<Warning>
**The default frame trap**

With `ORDER BY` in the window and no frame clause, the default frame is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. That makes `LAST_VALUE(x)` return the current row's value, and with `RANGE`, rows tied on the sort key are summed together in a "running" total. Specify `ROWS BETWEEN ...` explicitly.
</Warning>

Supported in PostgreSQL 8.4+, SQL Server 2012+ (full), Oracle, MySQL 8.0+, MariaDB 10.2+, SQLite 3.25+.

## Beyond the book / Senior perspective

- **Filter before grouping.** Conditions in `HAVING` that don't use aggregates should move to `WHERE`, so fewer rows are grouped.
- **Hash vs sort aggregation.** PostgreSQL chooses `HashAggregate` or `GroupAggregate` (which needs sorted input, e.g. from an index). Large hash aggregates can spill to disk; check `EXPLAIN ANALYZE` ([Query Optimization & EXPLAIN](/Software%20Engineering/Databases/Core%20Concepts/Query%20Optimization%20%26%20EXPLAIN)).
- **Joins before grouping multiply rows.** Joining orders to items and then to payments and summing both double-counts. Aggregate each child table in a subquery/CTE first, then join.
- **Window functions replace self-joins and correlated subqueries** for "previous row", "rank within group" and "running total" problems, usually with a single sort.
- **Pre-aggregate for dashboards**: materialized views or rollup tables ([Using Views](/Software%20Engineering/Databases/SQL%20ANSI/Using%20Views)) instead of grouping billions of rows per request.

## Interview questions

<Accordion title="WHERE vs HAVING?">
`WHERE` filters rows before grouping and can't use aggregates. `HAVING` filters groups after aggregation and can. Put non-aggregate conditions in `WHERE` for performance.
</Accordion>

<Accordion title="Does GROUP BY sort the output?">
No guarantee. Some plans produce sorted groups, hash aggregation doesn't. Use `ORDER BY`.
</Accordion>

<Accordion title="What's the difference between GROUP BY and a window function?">
`GROUP BY` returns one row per group. A window function returns a value for every input row, computed over its partition/frame, so you can show a row alongside its group's total, rank or running sum.
</Accordion>

<Accordion title="ROW_NUMBER vs RANK vs DENSE_RANK?">
On ties: `ROW_NUMBER` gives distinct numbers (1,2,3), `RANK` gives equal ranks and skips (1,2,2,4), `DENSE_RANK` gives equal ranks without gaps (1,2,2,3).
</Accordion>

<Accordion title="How do you get the top 3 rows per category?">
`ROW_NUMBER() OVER (PARTITION BY category ORDER BY score DESC)` in a subquery/CTE, then `WHERE rn <= 3`. Alternatives: `LATERAL` join with `LIMIT 3` (PostgreSQL), `CROSS APPLY` (SQL Server).
</Accordion>

<Accordion title="Why can I select `cust_name` when I only `GROUP BY cust_id` in PostgreSQL?">
Functional dependency: if `cust_id` is the primary key of the customers table, every other column of that table is determined by it, and the standard allows selecting them.
</Accordion>

## Related
- [SQL ANSI](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20ANSI) · [Databases Index](/Software%20Engineering/Databases/Databases%20Index)
- [Summarizing Data](/Software%20Engineering/Databases/SQL%20ANSI/Summarizing%20Data) · [Working with Subqueries](/Software%20Engineering/Databases/SQL%20ANSI/Working%20with%20Subqueries) · [Understanding Advanced SQL Features](/Software%20Engineering/Databases/SQL%20ANSI/Understanding%20Advanced%20SQL%20Features) (CTEs) · [Using Views](/Software%20Engineering/Databases/SQL%20ANSI/Using%20Views) · [Query Optimization & EXPLAIN](/Software%20Engineering/Databases/Core%20Concepts/Query%20Optimization%20%26%20EXPLAIN)

## Sources
- Ben Forta, *Sams Teach Yourself SQL in 10 Minutes a Day*, 5th ed., Lesson 10 ([O'Reilly](https://learning.oreilly.com/library/view/sams-teach-yourself/9780135182925/))
- [PostgreSQL docs: Window functions tutorial](https://www.postgresql.org/docs/current/tutorial-window.html) and [Window function calls](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS)
- [PostgreSQL docs: GROUPING SETS, CUBE, ROLLUP](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-GROUPING-SETS)
- [MySQL docs: Handling of GROUP BY (ONLY_FULL_GROUP_BY)](https://dev.mysql.com/doc/refman/8.4/en/group-by-handling.html)
- [SQLite docs: Window functions](https://www.sqlite.org/windowfunctions.html)
