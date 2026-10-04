---
title: "Using Views"
tags: [databases, databases/sql, databases/postgres]
aliases: [Views, CREATE VIEW, Materialized Views, Updatable Views, WITH CHECK OPTION, Indexed Views]
updated: 2026-09-27
---
# Using Views

<Note>
**TL;DR**

A **view** is a saved query you can use like a table. It stores no data (the query runs each time), so it simplifies complex joins, reformats data and limits what users can see. Simple views are **updatable**, and `WITH CHECK OPTION` stops writes that would fall outside the view. A **materialized view** stores the result for fast reads and must be refreshed; in PostgreSQL `REFRESH MATERIALIZED VIEW CONCURRENTLY` needs a unique index.
</Note>

> Course: SQL in 10 Minutes, Lesson 18 · Prev: [Creating and Manipulating Tables](/Software%20Engineering/Databases/SQL%20ANSI/Creating%20and%20Manipulating%20Tables) · Next: [Working with Stored Procedures](/Software%20Engineering/Databases/SQL%20ANSI/Working%20with%20Stored%20Procedures)

## Understanding views

Views are virtual tables. Tables contain data; views contain **queries** that retrieve data dynamically when used.

<Note>
**SQLite**

SQLite views are read-only: you can create and query them, but not write through them (unless you add `INSTEAD OF` triggers).
</Note>

Take this query, which retrieves data from three tables:

```sql
SELECT cust_name, cust_contact
FROM Customers, Orders, OrderItems
WHERE Customers.cust_id = Orders.cust_id
  AND OrderItems.order_num = Orders.order_num
  AND prod_id = 'RGAN01';
```

It returns the customers who ordered a specific product. Anyone needing this must understand the table structure and the joins, and to query another product they must edit the last condition.

Now imagine wrapping the whole query in a virtual table called `ProductCustomers`:

```sql
SELECT cust_name, cust_contact
FROM ProductCustomers
WHERE prod_id = 'RGAN01';
```

That's a view. `ProductCustomers` contains no columns or data, only the query that joins the tables correctly.

## Why use views

Common uses:
1. To **reuse** SQL statements.
2. To **simplify** complex SQL. Once written, the query is reused without knowing its details.
3. To expose **parts** of a table instead of the whole table.
4. To **secure** data: grant users access to specific subsets instead of whole tables.
5. To change data **formatting and representation**.
6. To keep a **stable interface** while the underlying tables are refactored.

<Note>
After creation, views are used like tables: `SELECT`, filter, sort, join to other views or tables, and sometimes insert and update (with restrictions, see below).
</Note>

<Note>
Views contain no data themselves; they return data from other tables. When those tables change, the view returns the changed data.
</Note>

## View rules and restrictions

1. Like tables, views must be uniquely named (no clash with any other table or view in the schema).
2. There's no limit on the number of views.
3. Creating views requires privileges, usually granted by a DBA.
4. Views can be nested (a view built on another view). The allowed depth varies by DBMS. Nesting can seriously degrade performance and hides complexity; test it before production.
5. Many DBMSs prohibit `ORDER BY` in a view's query (SQL Server allows it only with `TOP`; PostgreSQL allows it). Even where allowed, sort in the query that **uses** the view.
6. Some DBMSs require every column to be named, so calculated fields need aliases.
7. The book says views can't be indexed or have triggers or defaults. **That's not true everywhere** (corrected):
   - **SQL Server indexed views** (`WITH SCHEMABINDING` + a unique clustered index) are stored and indexed.
   - **PostgreSQL materialized views** can be indexed like tables.
   - Views can have `INSTEAD OF` triggers (PostgreSQL, SQL Server, Oracle, SQLite), and PostgreSQL views can have column defaults (`ALTER VIEW v ALTER COLUMN c SET DEFAULT ...`).
   A plain (non-materialized) view can't have its own index; it uses the base tables' indexes.
8. Some DBMSs (SQLite) treat views as read-only.
9. Some DBMSs let you prevent inserts or updates that would make a row fall out of the view. Example: a view of customers with email addresses; updating a customer to remove the email would make the row disappear from the view. By default this is allowed; `WITH CHECK OPTION` forbids it.

## Creating views

Views are created with `CREATE VIEW`. Like `CREATE TABLE`, it only creates a view that doesn't exist. To change one, drop and recreate it, or use `CREATE OR REPLACE VIEW` (PostgreSQL, MySQL, Oracle; `CREATE OR ALTER VIEW` in SQL Server 2016 SP1+).

### Using views to simplify complex joins

#### Example 1

```sql
CREATE VIEW ProductCustomers AS
SELECT cust_name, cust_contact, prod_id
FROM Customers, Orders, OrderItems
WHERE Customers.cust_id = Orders.cust_id
  AND OrderItems.order_num = Orders.order_num;
```

**Analysis:** creates `ProductCustomers`, joining three tables to list every customer who ordered anything. `SELECT * FROM ProductCustomers` lists all of them.

#### Example 2

```sql
SELECT cust_name, cust_contact
FROM ProductCustomers
WHERE prod_id = 'RGAN01';
```

**Analysis:** retrieves specific data from the view with a `WHERE`. The DBMS merges this `WHERE` with the view's own conditions, so data is filtered correctly (and the planner can still use indexes on the base tables). Write the SQL once, reuse it as needed.

<Tip>
Create views that are **not** tied to specific data. `ProductCustomers` returns customers for all products, so it's reusable; a view hard-coded to `RGAN01` would not be.
</Tip>

### Using views to reformat retrieved data

This `SELECT` (from [Creating Calculated Fields](/Software%20Engineering/Databases/SQL%20ANSI/Creating%20Calculated%20Fields)) returns vendor name and location in one combined calculated column:

```sql
SELECT RTRIM(vend_name) || ' (' || RTRIM(vend_country) || ')'
       AS vend_title
FROM Vendors
ORDER BY vend_name;
```

If you need this format regularly, make it a view:

```sql
CREATE VIEW VendorLocations AS
SELECT RTRIM(vend_name) || ' (' || RTRIM(vend_country) || ')'
       AS vend_title
FROM Vendors;
```

**Analysis:** the same query, minus the `ORDER BY`. To get all the mailing labels: `SELECT * FROM VendorLocations;`. (SQL Server: use `+` instead of `||`.)

### Using views to filter unwanted data

#### Example 3

```sql
CREATE VIEW CustomerEMailList AS
SELECT cust_id, cust_name, cust_email
FROM Customers
WHERE cust_email IS NOT NULL;
```

**Analysis:** when sending email to a mailing list you'd want to skip customers without an address. The `WHERE` filters out rows with `NULL` in `cust_email`. Querying the view with its own `WHERE` combines both conditions.

### Using views with calculated fields

#### Example 4

This `SELECT` (from [Creating Calculated Fields](/Software%20Engineering/Databases/SQL%20ANSI/Creating%20Calculated%20Fields)) retrieves the items of one order and calculates each expanded price:

```sql
SELECT prod_id,
       quantity,
       item_price,
       quantity * item_price AS expanded_price
FROM OrderItems
WHERE order_num = 20008;
```

Turn it into a view, **without** the order-specific condition so it works for every order:

```sql
CREATE VIEW OrderItemsExpanded AS
SELECT order_num,
       prod_id,
       quantity,
       item_price,
       quantity * item_price AS expanded_price
FROM OrderItems;
```

Then:

```sql
SELECT *
FROM OrderItemsExpanded
WHERE order_num = 20008;
```

**Analysis:** the calculation lives in one place. `order_num` is included in the view so callers can filter on it; a view can only be filtered on columns it exposes.

## Updatable views and `WITH CHECK OPTION`

A view is usually **automatically updatable** when it reads from a single table, with no `DISTINCT`, `GROUP BY`, aggregates, window functions, set operations or `LIMIT` (PostgreSQL 9.3+, MySQL, SQL Server, Oracle have similar rules). Anything more complex needs an `INSTEAD OF` trigger.

```sql
CREATE VIEW ILCustomers AS
SELECT cust_id, cust_name, cust_state
FROM Customers
WHERE cust_state = 'IL'
WITH CHECK OPTION;

UPDATE ILCustomers SET cust_name = 'Fun4All Chicago' WHERE cust_id = 1000000003;  -- OK
UPDATE ILCustomers SET cust_state = 'NY' WHERE cust_id = 1000000003;
-- ERROR: new row violates check option for view "ilcustomers"
INSERT INTO ILCustomers (cust_id, cust_name, cust_state) VALUES (1000000099, 'X', 'CA');
-- ERROR as well
```

`WITH CHECK OPTION` rejects inserts/updates that would produce rows the view can't see. `LOCAL` checks only this view's condition; `CASCADED` (the default) also checks the views it's built on.

## Materialized views

A materialized view **stores** the query result, like a cached table:

```sql
-- PostgreSQL
CREATE MATERIALIZED VIEW vendor_sales AS
SELECT p.vend_id,
       date_trunc('month', o.order_date) AS month,
       SUM(oi.quantity * oi.item_price)  AS revenue
FROM OrderItems oi
JOIN Orders o   ON o.order_num = oi.order_num
JOIN Products p ON p.prod_id   = oi.prod_id
GROUP BY p.vend_id, date_trunc('month', o.order_date);

-- A unique index is required for CONCURRENTLY, and speeds up lookups
CREATE UNIQUE INDEX vendor_sales_pk ON vendor_sales (vend_id, month);

-- Recompute. Plain REFRESH locks out readers while it runs;
-- CONCURRENTLY keeps the old data readable and swaps in a diff.
REFRESH MATERIALIZED VIEW CONCURRENTLY vendor_sales;
```

| | View | Materialized view |
|---|---|---|
| Stores data | No | Yes |
| Freshness | Always current | As of the last refresh |
| Read cost | Runs the query every time | Reads stored rows (indexable) |
| Write cost | None | Refresh cost (PostgreSQL recomputes the whole thing) |
| Support | Everywhere | PostgreSQL, Oracle (with fast/incremental refresh), SQL Server indexed views (maintained automatically on every write), not MySQL |

Refresh on a schedule (cron, `pg_cron`) or after batch loads. For incremental maintenance in PostgreSQL, people build summary tables updated by triggers or jobs, or use extensions.

## Beyond the book / Senior perspective

- **Views as an API layer.** Point reporting tools or other teams at views, not raw tables; you can then refactor tables behind a stable interface.
- **Security:** in PostgreSQL, a view runs with the **owner's** privileges by default, which can bypass row-level security on the base tables. PostgreSQL 15+ has `WITH (security_invoker = true)`. Use `security_barrier` views when filtering rows for security, so user-supplied functions can't leak filtered rows.
- **Nested views hurt.** Views on views on views hide expensive joins and make plans hard to read. Flatten them for hot paths.
- **Dependencies:** you can't drop or change the type of a column that a view uses. Migrations must drop and recreate dependent views (tools like `pg_depend` queries help).
- **Materialized view staleness is a product decision.** Show "as of" timestamps on dashboards.

## Interview questions

<Accordion title="What's the difference between a view and a table?">
A table stores data; a view stores a query and returns fresh data from the base tables each time it's used.
</Accordion>

<Accordion title="View vs materialized view?">
A view is re-executed on every use and always current. A materialized view stores the result, is fast to read and can be indexed, but is only as fresh as its last refresh.
</Accordion>

<Accordion title="What does REFRESH MATERIALIZED VIEW CONCURRENTLY need, and why use it?">
A unique index on the materialized view. It lets readers keep querying the old contents during the refresh instead of being blocked; the cost is a slower refresh.
</Accordion>

<Accordion title="Can you insert into a view?">
Yes if it's a simple, automatically updatable view (single table, no aggregates/DISTINCT/GROUP BY etc.), or if it has `INSTEAD OF` triggers. SQLite views are read-only without triggers.
</Accordion>

<Accordion title="What does WITH CHECK OPTION do?">
Rejects inserts and updates through the view that would create rows not visible through the view's `WHERE` condition.
</Accordion>

<Accordion title="Can views be indexed?">
Plain views can't, but they use the base tables' indexes. SQL Server indexed views and PostgreSQL/Oracle materialized views can be indexed.
</Accordion>

## Related
- [SQL ANSI](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20ANSI) · [Databases Index](/Software%20Engineering/Databases/Databases%20Index)
- [Creating Calculated Fields](/Software%20Engineering/Databases/SQL%20ANSI/Creating%20Calculated%20Fields) · [Grouping Data](/Software%20Engineering/Databases/SQL%20ANSI/Grouping%20Data) · [Working with Stored Procedures](/Software%20Engineering/Databases/SQL%20ANSI/Working%20with%20Stored%20Procedures) · [SQL Trigger Functions](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20Trigger%20Functions) (INSTEAD OF triggers) · [Query Optimization & EXPLAIN](/Software%20Engineering/Databases/Core%20Concepts/Query%20Optimization%20%26%20EXPLAIN)

## Sources
- Ben Forta, *Sams Teach Yourself SQL in 10 Minutes a Day*, 5th ed., Lesson 18 ([O'Reilly](https://learning.oreilly.com/library/view/sams-teach-yourself/9780135182925/))
- [PostgreSQL docs: CREATE VIEW (updatable views, CHECK OPTION, security_invoker)](https://www.postgresql.org/docs/current/sql-createview.html)
- [PostgreSQL docs: Materialized views](https://www.postgresql.org/docs/current/rules-materializedviews.html) and [REFRESH MATERIALIZED VIEW](https://www.postgresql.org/docs/current/sql-refreshmaterializedview.html)
- [SQL Server docs: Create indexed views](https://learn.microsoft.com/en-us/sql/relational-databases/views/create-indexed-views)
- [MySQL docs: Updatable and insertable views](https://dev.mysql.com/doc/refman/8.4/en/view-updatability.html)
