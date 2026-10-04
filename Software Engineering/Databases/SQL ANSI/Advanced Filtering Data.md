---
title: "Advanced Filtering Data"
tags: [databases, databases/sql]
aliases: [AND OR NOT IN, Logical Operators, Three-Valued Logic, NOT IN NULL, SQL Injection]
updated: 2026-09-27
---
# Advanced Filtering Data

<Note>
**TL;DR**

Combine conditions with `AND`, `OR`, `NOT` and `IN`. `AND` binds tighter than `OR`, so **always use parentheses** when mixing them. SQL uses **three-valued logic** (TRUE, FALSE, UNKNOWN), which makes `NOT IN` return nothing if the list contains a `NULL`; prefer `NOT EXISTS`. And never paste user input into a `WHERE` clause: use parameterized queries.
</Note>

> Course: SQL in 10 Minutes, Lesson 5 · Prev: [Filtering Data](/Software%20Engineering/Databases/SQL%20ANSI/Filtering%20Data) · Next: [Using Wildcard Filtering](/Software%20Engineering/Databases/SQL%20ANSI/Using%20Wildcard%20Filtering)

Combining `WHERE` with the `AND`, `OR`, `IN` and `NOT` operators.

**What is an operator?** A special keyword used to join or change clauses within a `WHERE` clause. Also known as *logical operators*.

## Using the `AND` operator

`AND`: only rows matching **all** the conditions are retrieved.

### Example 1

```sql
SELECT prod_id, prod_price, prod_name
FROM Products
WHERE vend_id = 'DLL01' AND prod_price <= 4;
```

**Analysis:** retrieves products made by vendor `DLL01` that cost $4 or less. A `DLL01` product that costs more than $4 is not returned, and neither is a cheap product from another vendor.

## Using the `OR` operator

`OR`: rows matching **either** condition are retrieved.

### Example 2

```sql
SELECT prod_id, prod_price, prod_name
FROM Products
WHERE vend_id = 'DLL01' OR vend_id = 'BRS01';
```

**Analysis:** returns products made by either vendor. With `AND` here, no rows would match (a product can't have two vendor IDs).

## Understanding order of evaluation

### Example 3: the wrong way

```sql
SELECT prod_name, prod_price
FROM Products
WHERE vend_id = 'DLL01' OR vend_id = 'BRS01'
      AND prod_price >= 10;
```

**Analysis:** some returned rows cost less than $10. SQL (like most languages) evaluates `AND` before `OR`, so this reads as *any products costing $10 or more made by `BRS01`, plus any products made by `DLL01` regardless of price*.

### Example 4: the fix, parentheses

```sql
SELECT prod_name, prod_price
FROM Products
WHERE (vend_id = 'DLL01' OR vend_id = 'BRS01')
      AND prod_price >= 10;
```

**Analysis:** parentheses have higher precedence than `AND` and `OR`, so the `OR` is evaluated first. Now it reads *products from `DLL01` or `BRS01` costing $10 or more*.

<Tip>
**Rule of thumb**

Whenever a `WHERE` mixes `AND` and `OR`, add parentheses, even when you think precedence already does the right thing. It costs nothing and removes ambiguity for the next reader.
</Tip>

Precedence, highest first: parentheses, comparison operators, `NOT`, `AND`, `OR`.

## Using the `IN` operator

`IN`: specifies a list of values to match, each compared with an implicit `OR`.

### Example 5

```sql
SELECT prod_name, prod_price
FROM Products
WHERE vend_id IN ('DLL01', 'BRS01')
ORDER BY prod_name;
```

**Analysis:** returns all products made by `DLL01` and `BRS01`. The list is comma-separated and enclosed in parentheses. It's equivalent to:

```sql
SELECT prod_name, prod_price
FROM Products
WHERE vend_id = 'DLL01' OR vend_id = 'BRS01'
ORDER BY prod_name;
```

Why use `IN`?
1. With long lists, `IN` is much cleaner and easier to read.
2. The order of evaluation is easier to manage when combined with `AND`/`OR`.
3. `IN` is often at least as fast as a list of `OR`s (modern optimizers frequently rewrite one into the other).
4. `IN` can contain another `SELECT` (a subquery), enabling dynamic `WHERE` clauses. See [Working with Subqueries](/Software%20Engineering/Databases/SQL%20ANSI/Working%20with%20Subqueries).

## Using the `NOT` operator

`NOT`: **negates** the condition that follows it. (The original note copied the `IN` definition here by mistake.)

### Example 6

```sql
SELECT prod_name
FROM Products
WHERE NOT vend_id = 'DLL01'
ORDER BY prod_name;
```

**Analysis:** instead of matching `vend_id` to `DLL01`, the DBMS matches every `vend_id` that is **not** `DLL01`.

### Example 7: the same with `<>`

```sql
SELECT prod_name
FROM Products
WHERE vend_id <> 'DLL01'
ORDER BY prod_name;
```

**Analysis:** for simple clauses like these there's no advantage to `NOT`. It shines in complex clauses: `NOT IN (...)`, `NOT BETWEEN`, `NOT LIKE`, `NOT EXISTS`, or negating a parenthesized group: `WHERE NOT (vend_id = 'DLL01' AND prod_price > 5)`.

## Three-valued logic

Because of `NULL`, every predicate is TRUE, FALSE or **UNKNOWN**. `WHERE` keeps only TRUE.

| `a` | `b` | `a AND b` | `a OR b` | `NOT a` |
|---|---|---|---|---|
| TRUE | UNKNOWN | UNKNOWN | TRUE | FALSE |
| FALSE | UNKNOWN | FALSE | UNKNOWN | TRUE |
| UNKNOWN | UNKNOWN | UNKNOWN | UNKNOWN | UNKNOWN |

`NOT UNKNOWN` is still UNKNOWN, so negating a condition does **not** give you "all the other rows" when NULLs are around.

## The `NOT IN` + NULL trap

```sql
-- Customers who never ordered? Looks right...
SELECT cust_id FROM Customers
WHERE cust_id NOT IN (SELECT cust_id FROM Orders);
```

If **any** `Orders.cust_id` is `NULL`, this returns **zero rows**. `x NOT IN (1, 2, NULL)` expands to `x <> 1 AND x <> 2 AND x <> NULL`; the last part is UNKNOWN, so the whole thing is never TRUE.

Safe versions:

```sql
-- NOT EXISTS ignores NULLs correctly and is usually planned as an anti-join
SELECT c.cust_id
FROM Customers c
WHERE NOT EXISTS (SELECT 1 FROM Orders o WHERE o.cust_id = c.cust_id);

-- or filter the NULLs out explicitly
WHERE cust_id NOT IN (SELECT cust_id FROM Orders WHERE cust_id IS NOT NULL);
```

## `EXISTS` vs `IN`

| | `IN (subquery)` | `EXISTS (correlated subquery)` |
|---|---|---|
| Meaning | value is in the returned set | at least one matching row exists |
| NULL behaviour | `NOT IN` breaks with NULLs | `NOT EXISTS` is safe |
| Performance | Modern PostgreSQL/SQL Server/Oracle plan both as semi-joins; old MySQL (< 5.6) ran `IN` subqueries badly | Stops at the first match |
| Use when | short literal lists, or a clean subquery on a NOT NULL column | "has any / has none" checks, especially negated |

## Parameterized queries and SQL injection

Filters usually contain user input. Building them with string concatenation is the classic **SQL injection** hole:

```python
# NEVER do this
cur.execute("SELECT * FROM users WHERE email = '" + email + "'")
# email = "x' OR '1'='1"  -> returns every user
```

Use placeholders; the driver sends the value separately from the SQL text, so it can never change the query's structure:

```python
cur.execute("SELECT * FROM users WHERE email = %s", (email,))       # psycopg
```
```js
await client.query('SELECT * FROM users WHERE email = $1', [email]); // node-postgres
```

For `IN` lists, pass an array (`WHERE id = ANY($1)` in PostgreSQL) instead of concatenating. Identifiers (table/column names) can't be parameters; whitelist them in code.

## Beyond the book / Senior perspective

- **`OR` across different columns** (`WHERE a = 1 OR b = 2`) often can't use a single index; databases may use a bitmap OR of two indexes, or you can rewrite as `UNION`. PostgreSQL 18 improved turning `OR` clauses into index-friendly array lookups.
- **Long `IN` lists** (thousands of literals) bloat parse time and plan caches. Use `= ANY($1::int[])`, a temp table, or a `VALUES` list joined in.
- **`NOT`/`<>` predicates are rarely selective**; an index won't help "everything except X". Consider a partial index on the rare case instead ([SQL Indexes Deep Dive](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20Indexes%20Deep%20Dive)).
- In code review, flag any SQL built with `+`, template strings or `format()` using request data.

## Interview questions

<Accordion title="What's the precedence of AND and OR?">
`AND` binds tighter than `OR` (and `NOT` tighter than both). `a OR b AND c` means `a OR (b AND c)`. Use parentheses.
</Accordion>

<Accordion title="Why can `NOT IN (subquery)` return no rows?">
If the subquery returns a `NULL`, `x <> NULL` is UNKNOWN, so `x NOT IN (...)` is never TRUE. Use `NOT EXISTS` or filter `IS NOT NULL` in the subquery.
</Accordion>

<Accordion title="EXISTS or IN?">
Semantically similar for positive checks, and modern optimizers plan both as semi-joins. For negation, `NOT EXISTS` is the safe default because of NULLs. `EXISTS` also reads naturally for correlated "has any" checks.
</Accordion>

<Accordion title="How do you prevent SQL injection?">
Parameterized queries / prepared statements for all values; whitelist dynamic identifiers; least-privilege DB users; ORMs/query builders that parameterize. Escaping by hand is fragile.
</Accordion>

<Accordion title="What is three-valued logic?">
SQL predicates evaluate to TRUE, FALSE or UNKNOWN (when `NULL` is involved). `WHERE`, `HAVING` and `ON` keep only TRUE; `CHECK` constraints reject only FALSE, so `NULL` passes a check.
</Accordion>

## Related
- [SQL ANSI](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20ANSI) · [Databases Index](/Software%20Engineering/Databases/Databases%20Index)
- [Filtering Data](/Software%20Engineering/Databases/SQL%20ANSI/Filtering%20Data) · [Working with Subqueries](/Software%20Engineering/Databases/SQL%20ANSI/Working%20with%20Subqueries) · [Using Wildcard Filtering](/Software%20Engineering/Databases/SQL%20ANSI/Using%20Wildcard%20Filtering) · [SQL Indexes Deep Dive](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20Indexes%20Deep%20Dive)
- Security: [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types) (the same "data vs code" idea on the client)

## Sources
- Ben Forta, *Sams Teach Yourself SQL in 10 Minutes a Day*, 5th ed., Lesson 5 ([O'Reilly](https://learning.oreilly.com/library/view/sams-teach-yourself/9780135182925/))
- [PostgreSQL docs: Logical operators](https://www.postgresql.org/docs/current/functions-logical.html) and [Subquery expressions (EXISTS, IN, NOT IN)](https://www.postgresql.org/docs/current/functions-subquery.html)
- [OWASP: SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [PostgreSQL 18 release announcement (OR-clause index use)](https://www.postgresql.org/about/news/postgresql-18-released-3142/)
