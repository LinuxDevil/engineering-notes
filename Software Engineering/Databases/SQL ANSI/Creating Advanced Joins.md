---
tags: [databases, databases/sql]
aliases: [Outer Joins, LEFT JOIN, RIGHT JOIN, FULL OUTER JOIN, Self Join, Natural Join, Table Aliases, Anti-join]
updated: 2026-09-27
---
# Creating Advanced Joins

> [!tldr] TL;DR
> Table aliases shorten queries and make self joins possible. **Outer joins** keep unmatched rows from one side (`LEFT`/`RIGHT`) or both (`FULL`), filling the other side with `NULL`. A condition on the outer table's columns belongs in `ON`, not `WHERE`, or you silently turn the outer join back into an inner join. SQLite supports `RIGHT` and `FULL OUTER JOIN` since 3.39.0 (2022); MySQL/MariaDB still have no `FULL OUTER JOIN`.

> Course: SQL in 10 Minutes, Lesson 13 · Prev: [[Joining Tables]] · Next: [[Combining Queries]]

## Using table aliases

### Example 1

```sql
SELECT cust_name, cust_contact
FROM Customers AS C, Orders AS O, OrderItems AS OI
WHERE C.cust_id = O.cust_id
  AND OI.order_num = O.order_num
  AND prod_id = 'RGAN01';
```

**Analysis:** the three tables have aliases: `Customers AS C` lets you write `C` instead of `Customers`. Here aliases are used only in `WHERE`, but you can use them in the `SELECT` list, `ORDER BY` and anywhere else in the statement.

> [!note]
> Table aliases exist only during query execution. Unlike column aliases, they are **never returned to the client**. Oracle does not accept the `AS` keyword for table aliases: write `Customers C`.

## Using different join types

Self join, natural join and outer join.

### Self joins

A self join joins a table **to itself**. You need a column that acts as a key (call it X) and another column (Y) whose values can be matched to X. In a given row, X and Y needn't be equal, and Y may even be `NULL` (the classic example: `employees.manager_id` pointing at `employees.id`).

#### Example 2

Send a mailing to all contacts who work for the same company as Jim Jones. First find Jim Jones's company, then the customers at that company:

```sql
SELECT c1.cust_id, c1.cust_name, c1.cust_contact
FROM Customers AS c1, Customers AS c2
WHERE c1.cust_name = c2.cust_name
  AND c2.cust_contact = 'Jim Jones';
```

**Analysis:** both tables are `Customers`, so it appears twice in `FROM`. That's legal, but any reference to `Customers` would be ambiguous, so aliases `c1` and `c2` are used as table names. The `SELECT` uses `c1.` to say which copy's columns to return; without it you'd get an ambiguity error (there are two of each `cust_id`, `cust_name`, `cust_contact`, even though they're the same table). The `WHERE` joins the two copies and filters `c2` to Jim Jones.

The subquery version:

```sql
SELECT cust_id, cust_name, cust_contact
FROM Customers
WHERE cust_name = (SELECT cust_name
                   FROM Customers
                   WHERE cust_contact = 'Jim Jones');
```

> [!note]
> Self joins are often used instead of subqueries that read from the same table. The book says many DBMSs process joins faster than subqueries; today's optimizers often produce the same plan for both, so write the clearer one and check `EXPLAIN`.

A hierarchy example (employee and manager):

```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON m.id = e.manager_id;   -- LEFT so the CEO (no manager) appears
```

For whole hierarchies (all levels), use a recursive CTE: [[Understanding Advanced SQL Features]].

### Natural joins

The book defines a natural join as one where you select only non-duplicated columns: typically `SELECT *` for one table and an explicit column list for the others, so each common column appears once.

The SQL keyword `NATURAL JOIN` is something slightly different: it joins two tables on **all** columns that have the same name, and returns each common column once.

#### INNER vs NATURAL joins

| `NATURAL JOIN` | `INNER JOIN` |
|---|---|
| Joins on every column with the same name in both tables | Joins on the columns you specify in `ON` (or `USING`) |
| Common columns appear once in the result | All columns of both tables, including both copies of the join columns (with `SELECT *`) |
| If there are no common column names, it becomes a cross join | Without `ON`, it's a syntax error (except MySQL, which treats it as a cross join) |
| `SELECT * FROM table1 NATURAL JOIN table2;` | `SELECT * FROM table1 INNER JOIN table2 ON table1.col = table2.col;` |

> [!warning] Avoid `NATURAL JOIN` in real code
> Add a column called `updated_at` or `name` to both tables and the join condition silently changes. Be explicit with `ON` or `USING (col)`.

#### Example 3

```sql
SELECT C.*, O.order_num, O.order_date,
       OI.prod_id, OI.quantity, OI.item_price
FROM Customers AS C, Orders AS O, OrderItems AS OI
WHERE C.cust_id = O.cust_id
  AND OI.order_num = O.order_num
  AND prod_id = 'RGAN01';
```

**Analysis:** the wildcard is used for the first table only; all other columns are listed explicitly, so no duplicate columns are returned. In the book's sense, nearly every inner join you write is a natural join.

## Outer joins

Most joins relate rows in one table to rows in another. Sometimes you also want rows that have **no** related rows, e.g. every customer including those with no orders. That's an **outer join**.

### Example 4

```sql
SELECT Customers.cust_id, Orders.order_num
FROM Customers
LEFT OUTER JOIN Orders ON Customers.cust_id = Orders.cust_id;
```

**Analysis:** like an inner join, but it also includes rows with no related rows. With `OUTER JOIN` you must say `LEFT` or `RIGHT` to name the table whose rows are all kept (`LEFT` = the table before the keyword, `RIGHT` = the one after). Here all `Customers` rows are kept; customers without orders get `NULL` in `order_num`. `OUTER` is optional: `LEFT JOIN` is the same thing.

A `RIGHT JOIN` is a `LEFT JOIN` with the tables swapped; most people write only `LEFT JOIN`s for consistency.

> [!note] SQLite outer joins (corrected)
> The book (and the original note) says SQLite supports `LEFT OUTER JOIN` but not `RIGHT`. That was true until **SQLite 3.39.0 (2022-07-21)**, which added `RIGHT` and `FULL OUTER JOIN`. On older SQLite, flip the table order and use `LEFT JOIN`.

### Anti-join: rows with **no** match

```sql
-- Customers who have never ordered
SELECT c.cust_id
FROM Customers c
LEFT JOIN Orders o ON o.cust_id = c.cust_id
WHERE o.cust_id IS NULL;

-- Equivalent and often clearer
SELECT c.cust_id
FROM Customers c
WHERE NOT EXISTS (SELECT 1 FROM Orders o WHERE o.cust_id = c.cust_id);
```

### The `ON` vs `WHERE` trap

```sql
-- WRONG: customers with no 2020 orders disappear
SELECT c.cust_id, o.order_num
FROM Customers c
LEFT JOIN Orders o ON o.cust_id = c.cust_id
WHERE o.order_date >= '2020-01-01';

-- RIGHT: keep every customer, attach only their 2020+ orders
SELECT c.cust_id, o.order_num
FROM Customers c
LEFT JOIN Orders o ON o.cust_id = c.cust_id
                  AND o.order_date >= '2020-01-01';
```

For customers with no matching orders, `o.order_date` is `NULL`, the `WHERE` condition is UNKNOWN, and the row is dropped, which is exactly what an inner join would do. Conditions on the **optional** side go in `ON`; conditions on the **preserved** side go in `WHERE`.

## Full outer joins

`FULL OUTER JOIN` returns all rows from both tables, matched where possible and padded with `NULL`s where not.

### Example 5

```sql
SELECT Customers.cust_id, Orders.order_num
FROM Customers
FULL OUTER JOIN Orders ON Customers.cust_id = Orders.cust_id;
```

**Analysis:** a rarely used variant. It keeps customers with no orders **and** orders with no customer. Left/right outer joins keep unrelated rows from one table; a full outer join keeps them from both.

> [!note] Support (corrected)
> - PostgreSQL, SQL Server, Oracle, DB2: supported.
> - SQLite: supported since **3.39.0** (2022-07-21).
> - **MySQL and MariaDB: still not supported.** Emulate it with a `UNION` of a left and a right join:
>
> ```sql
> SELECT c.cust_id, o.order_num
> FROM Customers c LEFT JOIN Orders o ON c.cust_id = o.cust_id
> UNION ALL
> SELECT c.cust_id, o.order_num
> FROM Customers c RIGHT JOIN Orders o ON c.cust_id = o.cust_id
> WHERE c.cust_id IS NULL;      -- only the right-side rows with no match
> ```
>
> (Plain `UNION` of the two joins also works but removes legitimately duplicate rows.)

Typical real use: reconciling two data sources ("which records exist only in system A, only in B, or in both but differ?").

## Using joins with aggregate functions

### Example 6

```sql
SELECT Customers.cust_id,
       COUNT(Orders.order_num) AS num_ord
FROM Customers
INNER JOIN Orders ON Customers.cust_id = Orders.cust_id
GROUP BY Customers.cust_id;
```

**Analysis:** `INNER JOIN` relates customers to orders, `GROUP BY` groups by customer, and `COUNT(Orders.order_num)` counts each customer's orders as `num_ord`.

### Example 7

```sql
SELECT Customers.cust_id,
       COUNT(Orders.order_num) AS num_ord
FROM Customers
LEFT OUTER JOIN Orders ON Customers.cust_id = Orders.cust_id
GROUP BY Customers.cust_id;
```

**Analysis:** the left outer join includes all customers, even those with no orders: customer `1000000002` now appears with `0`. Note it's `COUNT(Orders.order_num)`, not `COUNT(*)`: `COUNT(*)` would count the `NULL`-padded row and report 1.

## Joins summary

![[Untitled 35.png]]

There are four main join types, depending on which rows you want to keep when combining tables on common columns: **inner** (only matching rows in both), **left** (all rows from the left table plus matches from the right), **right** (all rows from the right plus matches from the left), and **full** (all rows from both, matched or not).

**Inner join:** only the data common to both tables (area 3 in the diagram): all rows from both tables that satisfy the join condition.

```sql
SELECT column_names FROM table1 INNER JOIN table2 ON table1.col = table2.col WHERE condition;
```

![[Untitled 36.png]]

**Left join:** all rows from the left table, and from the right table only those matching the condition (areas 1 + 3).

```sql
SELECT column_names FROM table1 LEFT JOIN table2 ON table1.col = table2.col WHERE condition;
```

![[Untitled 37.png]]

**Right join:** all rows from the right table, and from the left only matching rows (areas 3 + 2).

```sql
SELECT column_names FROM table1 RIGHT JOIN table2 ON table1.col = table2.col WHERE condition;
```

![[Untitled 38.png]]

**Full join:** all rows from both tables, whether the join condition matched or not (areas 1 + 2 + 3).

```sql
SELECT column_names FROM table1 FULL JOIN table2 ON table1.col = table2.col WHERE condition;
```

![[Untitled 39.png]]

![[Untitled 40.png]]

![[Untitled 41.png]]

![[Screen_Shot_2022-07-02_at_6.59.18_PM.png]]

![[Screen_Shot_2022-07-02_at_6.59.37_PM.png]]

> [!tip] Venn diagrams are a simplification
> Joins produce **combinations of rows**, not set intersections. If one customer has three orders, the inner join returns three rows for that customer. Venn diagrams hide this fan-out, which is where most join bugs come from.

## Beyond the book / Senior perspective

- **Outer joins constrain the planner.** The optimizer can reorder inner joins freely but has fewer options with outer joins. Don't write `LEFT JOIN` "just in case"; use it when you need the unmatched rows.
- **`NULL`-padded columns** from outer joins flow into expressions and aggregates. Use `COALESCE` for display and `COUNT(col)` rather than `COUNT(*)`.
- **Semi- and anti-joins** (`EXISTS`, `NOT EXISTS`) often express "has / has no related rows" better than joins plus `DISTINCT` or `IS NULL` checks.
- **Top-N per row** (latest 3 orders per customer): `LATERAL` / `CROSS APPLY` ([[Working with Subqueries]]) or window functions ([[Grouping Data]]).

## Interview questions

> [!question]- LEFT JOIN vs INNER JOIN?
> Inner returns only rows with a match on both sides. Left returns every row from the left table; where there's no match, the right side's columns are `NULL`.

> [!question]- Why did my LEFT JOIN behave like an INNER JOIN?
> A `WHERE` condition on a right-table column rejects the `NULL`-padded rows. Move that condition into the `ON` clause (or allow `IS NULL` explicitly).

> [!question]- How do you do a FULL OUTER JOIN in MySQL?
> `LEFT JOIN` ... `UNION ALL` ... `RIGHT JOIN` restricted to rows where the left key `IS NULL` (or `UNION` of both joins if duplicates don't matter).

> [!question]- Find customers with no orders, three ways.
> `LEFT JOIN orders ... WHERE orders.cust_id IS NULL`; `WHERE NOT EXISTS (SELECT 1 FROM orders WHERE ...)`; `WHERE cust_id NOT IN (SELECT cust_id FROM orders WHERE cust_id IS NOT NULL)`. `NOT EXISTS` is the safest.

> [!question]- What is a self join? Give a use case.
> A table joined to itself with two aliases, e.g. employees to their managers via `manager_id`, or finding pairs of rows that share a value.

> [!question]- Why avoid NATURAL JOIN?
> It joins on every same-named column, so adding an unrelated common column (like `created_at`) changes results silently.

## Related
- [[SQL ANSI]] · [[Databases Index]]
- [[Joining Tables]] · [[Combining Queries]] · [[Working with Subqueries]] · [[Advanced Filtering Data]] · [[Query Optimization & EXPLAIN]]

## Sources
- Ben Forta, *Sams Teach Yourself SQL in 10 Minutes a Day*, 5th ed., Lesson 13 ([O'Reilly](https://learning.oreilly.com/library/view/sams-teach-yourself/9780135182925/))
- [SQLite release 3.39.0 (2022-07-21): RIGHT and FULL OUTER JOIN](https://www.sqlite.org/releaselog/3_39_0.html)
- [MySQL docs: JOIN clause](https://dev.mysql.com/doc/refman/8.4/en/join.html)
- [PostgreSQL docs: Joined tables](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN)
