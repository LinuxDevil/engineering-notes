---
tags: [databases, databases/sql]
aliases: [Subqueries, Subquery, Correlated Subquery, Derived Table, LATERAL, CTE]
updated: 2026-09-27
---
# Working with Subqueries

> [!tldr] TL;DR
> A subquery is a `SELECT` nested inside another statement. Use it in `WHERE ... IN (...)` / `EXISTS (...)` to filter, in the `SELECT` list as a calculated field, or in `FROM` as a derived table. A **correlated** subquery references the outer row and logically runs once per outer row. Modern optimizers often turn subqueries into joins, but for readability of multi-step queries use **CTEs** (`WITH`), and for "top N per row" use `LATERAL`.

> Course: SQL in 10 Minutes, Lesson 11 · Prev: [[Grouping Data]] · Next: [[Joining Tables]]

## Understanding subqueries

So far every `SELECT` was a simple query: one statement reading from one table. SQL also lets you nest queries inside queries.

**Query:** any SQL statement, though the term usually means a `SELECT`.

## Filtering by subquery

### Example 1: customers who ordered item `RGAN01`

`Orders` stores one row per order (number, customer ID, date). Order items are in `OrderItems`. Customer details are in `Customers`; `Orders` only has the customer ID.

Steps:
1. Get the order numbers of all orders containing `RGAN01`.
2. Get the customer IDs for those orders.
3. Get the customer details for those IDs.

Each step can be its own query, feeding the next one's `WHERE`:

```sql
SELECT order_num
FROM OrderItems
WHERE prod_id = 'RGAN01';
```

That returns orders `20007` and `20008`. Next, the customers for those orders:

```sql
SELECT cust_id
FROM Orders
WHERE order_num IN (20007, 20008);
```

Now combine them, turning the first query into a subquery:

```sql
SELECT cust_id
FROM Orders
WHERE order_num IN (SELECT order_num
                    FROM OrderItems
                    WHERE prod_id = 'RGAN01');
```

And finally fetch the customer details:

```sql
SELECT cust_name, cust_contact
FROM Customers
WHERE cust_id IN (SELECT cust_id
                  FROM Orders
                  WHERE order_num IN (SELECT order_num
                                      FROM OrderItems
                                      WHERE prod_id = 'RGAN01'));
```

**Analysis:** logically, subqueries are processed from the innermost `SELECT` outward. First `SELECT order_num FROM OrderItems WHERE prod_id = 'RGAN01'` returns `20007` and `20008`. Those are passed, comma-delimited, to the `IN` of the next query, which becomes `SELECT cust_id FROM Orders WHERE order_num IN (20007, 20008)`. Its customer IDs then feed the top-level query, which returns the data. Three `SELECT`s, one statement.

> [!note] Subquery rules
> - A subquery used with `IN` or `=` must return a **single column**.
> - A subquery used with `=`, `<`, `>` (a *scalar* subquery) must return **at most one row**, or you get "more than one row returned by a subquery used as an expression".
> - Nesting has no hard limit, but deep nesting hurts readability. Use joins or CTEs.

The same result with joins (often clearer, see [[Joining Tables]]):

```sql
SELECT DISTINCT c.cust_name, c.cust_contact
FROM Customers c
JOIN Orders o      ON o.cust_id = c.cust_id
JOIN OrderItems oi ON oi.order_num = o.order_num
WHERE oi.prod_id = 'RGAN01';
```

Note the `DISTINCT`: a customer who ordered `RGAN01` twice would appear twice with the join, but once with `IN`. That's why "does a matching row exist?" questions are more naturally written as `IN`/`EXISTS`.

## Using subqueries as calculated fields

### Example 2: number of orders per customer

Steps:
1. Retrieve the list of customers from `Customers`.
2. For each customer, count their orders in `Orders`.

```sql
SELECT cust_name,
       cust_state,
       (SELECT COUNT(*)
        FROM Orders
        WHERE Orders.cust_id = Customers.cust_id) AS orders
FROM Customers
ORDER BY cust_name;
```

**Analysis:** three columns per customer; `orders` is a calculated field set by the subquery in parentheses. The subquery is **correlated**: it refers to `Customers.cust_id` from the outer query, so logically it runs once per customer (five times here).

The subquery's `WHERE` uses **fully qualified** column names (`Orders.cust_id`, `Customers.cust_id`): table name, period, column name. You must do this whenever a column name is ambiguous. Without qualification, the DBMS resolves `cust_id` to the nearest table (`Orders`) on both sides:

```sql
SELECT COUNT(*) FROM Orders WHERE cust_id = cust_id
```

That compares `Orders.cust_id` with itself, which is always true (for non-null values), so every customer would get the **total** number of orders in the table.

> [!note]
> Subqueries are very useful for this kind of `SELECT`, but always qualify ambiguous column names.

The join + `GROUP BY` equivalent (note the `LEFT JOIN` so customers with no orders get 0):

```sql
SELECT c.cust_name, c.cust_state, COUNT(o.order_num) AS orders
FROM Customers c
LEFT JOIN Orders o ON o.cust_id = c.cust_id
GROUP BY c.cust_id, c.cust_name, c.cust_state
ORDER BY c.cust_name;
```

## Subquery types at a glance

| Type | Where | Returns | Example |
|---|---|---|---|
| Scalar | `SELECT` list, `WHERE x = (...)` | one value | `(SELECT MAX(prod_price) FROM Products)` |
| Multi-row | `IN`, `ANY`, `ALL` | one column, many rows | `WHERE id IN (SELECT ...)` |
| Existence | `EXISTS`, `NOT EXISTS` | true/false | `WHERE EXISTS (SELECT 1 FROM ... WHERE ...)` |
| Derived table | `FROM (...) AS t` | a table | `FROM (SELECT ... GROUP BY ...) AS t` |
| Correlated | any of the above | references the outer row | Example 2 |

## Derived tables and CTEs

A subquery in `FROM` is a *derived table*. A **CTE** (common table expression, `WITH`) is the same idea with a name, written up front:

```sql
WITH order_totals AS (
  SELECT order_num, SUM(quantity * item_price) AS total
  FROM OrderItems
  GROUP BY order_num
),
big_orders AS (
  SELECT order_num FROM order_totals WHERE total > 1000
)
SELECT c.cust_name, o.order_num
FROM big_orders b
JOIN Orders o    ON o.order_num = b.order_num
JOIN Customers c ON c.cust_id = o.cust_id;
```

CTEs read top-to-bottom like steps, and each can be referenced several times. Recursive CTEs (hierarchies, graphs) are covered in [[Understanding Advanced SQL Features]].

> [!note] CTE performance in PostgreSQL
> Before PostgreSQL 12, every CTE was an "optimization fence" (computed separately, conditions not pushed in). Since 12, a non-recursive CTE referenced once is inlined like a subquery. Force the old behaviour with `WITH x AS MATERIALIZED (...)`.

## `LATERAL`: a subquery per outer row, returning many rows

A normal derived table can't reference other tables in the same `FROM`. `LATERAL` can, like a correlated subquery that returns a whole table:

```sql
-- Each customer's 3 most recent orders (PostgreSQL, MySQL 8.0.14+)
SELECT c.cust_name, r.order_num, r.order_date
FROM Customers c
CROSS JOIN LATERAL (
  SELECT o.order_num, o.order_date
  FROM Orders o
  WHERE o.cust_id = c.cust_id
  ORDER BY o.order_date DESC
  LIMIT 3
) r;
```

Use `LEFT JOIN LATERAL (...) r ON true` to keep customers with no orders. SQL Server and Oracle 12c+ spell it `CROSS APPLY` / `OUTER APPLY`. With an index on `Orders (cust_id, order_date DESC)`, this is often the fastest top-N-per-group query.

## Beyond the book / Senior perspective

- **"Subqueries are slow" is mostly folklore now.** PostgreSQL, SQL Server and Oracle rewrite `IN`/`EXISTS` subqueries into semi-joins and anti-joins. Correlated scalar subqueries in the `SELECT` list, though, can still run once per row; check the plan ([[Query Optimization & EXPLAIN]]).
- **`NOT IN` + NULL** returns nothing; use `NOT EXISTS` (details in [[Advanced Filtering Data]]).
- **Pick the form that states intent**: `EXISTS` for "has any", a join for "combine columns", a CTE for multi-step logic, `LATERAL` for "top N per row", window functions for "rank/compare within a group" ([[Grouping Data]]).
- **Scalar subqueries must return one row.** Guard with `LIMIT 1` plus an `ORDER BY`, or an aggregate, otherwise data growth turns a working query into a runtime error.

## Interview questions

> [!question]- What's a correlated subquery?
> A subquery that references columns of the outer query, so its result depends on the current outer row. Logically it runs once per outer row; optimizers may decorrelate it into a join.

> [!question]- Subquery or join, which is faster?
> It depends on the plan, and modern optimizers often produce the same plan for `IN`/`EXISTS` and semi-joins. Write the clearest form, then check `EXPLAIN`. Watch for correlated scalar subqueries in the `SELECT` list on large result sets.

> [!question]- What is a CTE and when would you use one?
> A named temporary result defined with `WITH`, visible to the rest of the statement. Use it to break complex queries into readable steps, reuse an intermediate result, or write recursive queries.

> [!question]- What does LATERAL do?
> Lets a subquery in `FROM` reference columns from tables listed before it, so it runs per outer row and can return multiple rows (e.g. top 3 orders per customer with `ORDER BY ... LIMIT 3`). SQL Server calls it `APPLY`.

> [!question]- Why must you qualify column names in a correlated subquery?
> Unqualified names bind to the nearest scope. `WHERE cust_id = cust_id` inside a subquery on `Orders` compares the column with itself, silently returning wrong results.

## Related
- [[SQL ANSI]] · [[Databases Index]]
- [[Advanced Filtering Data]] (EXISTS vs IN, NOT IN trap) · [[Joining Tables]] · [[Grouping Data]] (window functions) · [[Understanding Advanced SQL Features]] (recursive CTEs) · [[Query Optimization & EXPLAIN]]

## Sources
- Ben Forta, *Sams Teach Yourself SQL in 10 Minutes a Day*, 5th ed., Lesson 11 ([O'Reilly](https://learning.oreilly.com/library/view/sams-teach-yourself/9780135182925/))
- [PostgreSQL docs: Subquery expressions](https://www.postgresql.org/docs/current/functions-subquery.html)
- [PostgreSQL docs: WITH queries (CTEs)](https://www.postgresql.org/docs/current/queries-with.html)
- [PostgreSQL docs: LATERAL subqueries](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-LATERAL)
- [MySQL docs: Lateral derived tables](https://dev.mysql.com/doc/refman/8.4/en/lateral-derived-tables.html)
