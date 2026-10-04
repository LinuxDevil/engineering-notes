---
title: "Creating Calculated Fields"
tags: [databases, databases/sql]
aliases: [Calculated Fields, Concatenation, Column Aliases, AS, CASE expression, COALESCE]
updated: 2026-09-27
---
# Creating Calculated Fields

<Note>
**TL;DR**

A calculated field is a column computed in the `SELECT` (concatenation, arithmetic, `CASE`, functions) rather than stored in the table. Name it with an alias (`AS`). Concatenation syntax differs: `||` (standard, PostgreSQL, Oracle, SQLite), `+` (SQL Server), `CONCAT()` (MySQL, and most others). Watch out: `NULL` in arithmetic or `||` makes the whole result `NULL`; use `COALESCE`.
</Note>

> Course: SQL in 10 Minutes, Lesson 7 · Prev: [Using Wildcard Filtering](/Software%20Engineering/Databases/SQL%20ANSI/Using%20Wildcard%20Filtering) · Next: [Using Data Manipulation Functions](/Software%20Engineering/Databases/SQL%20ANSI/Using%20Data%20Manipulation%20Functions)

**Field:** essentially the same thing as *column*. Database columns are usually called columns; "field" is mostly used for calculated fields.

Why calculate in SQL? The database can format, combine and compute over millions of rows close to the data, and it sends back only the result.

## Concatenating fields

**Concatenate:** join values together (append them) to form a single longer value.

| DBMS | Operator / function |
|---|---|
| PostgreSQL, Oracle, DB2, SQLite (standard) | `\|\|` |
| SQL Server | `+` (and `CONCAT()` since 2012) |
| MySQL, MariaDB | `CONCAT()` (`\|\|` means OR unless `PIPES_AS_CONCAT` mode) |

### Example 1

```sql
/* DB2, Oracle, PostgreSQL, SQLite */
SELECT vend_name || ' (' || vend_country || ')'
FROM Vendors
ORDER BY vend_name;

/* SQL Server */
SELECT vend_name + ' (' + vend_country + ')'
FROM Vendors
ORDER BY vend_name;

/* MySQL, MariaDB */
SELECT CONCAT(vend_name, ' (', vend_country, ')')
FROM Vendors
ORDER BY vend_name;
```

**Analysis:** the statements concatenate four elements:
1. The name stored in `vend_name`.
2. A string containing a space and an open parenthesis.
3. The country stored in `vend_country`.
4. A string containing the close parenthesis.

The result is a single calculated column containing all four as one value.

## The `RTRIM()` function

Trims trailing spaces so the output looks right (the book's tables use `CHAR`, which pads values with spaces).

### Example 2

```sql
/* SQL Server */
SELECT RTRIM(vend_name) + ' (' + RTRIM(vend_country) + ')'
FROM Vendors
ORDER BY vend_name;

/* Standard / PostgreSQL / Oracle / SQLite */
SELECT RTRIM(vend_name) || ' (' || RTRIM(vend_country) || ')'
FROM Vendors
ORDER BY vend_name;
```

**Analysis:** `RTRIM()` removes all spaces from the right of a value. Most DBMSs also have `LTRIM()` (left side) and `TRIM()` (both sides).

## Using aliases (derived columns)

An alias is an alternate name for a field or value, assigned with `AS`. Without one, the calculated column has no usable name (or a generated one like `?column?` in PostgreSQL).

### Example 3

```sql
SELECT RTRIM(vend_name) || ' (' || RTRIM(vend_country) || ')'
       AS vend_title
FROM Vendors
ORDER BY vend_name;
```

**Analysis:** same statement, with `AS vend_title` after the calculated field. The result column is now named `vend_title`, and client code can refer to it like any real column.

<Note>
**Alias rules**

- `AS` is optional for column aliases in most DBMSs, but writing it is clearer. (Oracle does not allow `AS` for *table* aliases.)
- Aliases with spaces need quoting: `AS "Vendor Title"`. Avoid them.
- You can use a select-list alias in `ORDER BY`, but **not** in `WHERE`, because `WHERE` is evaluated before the select list (see the logical order in [Grouping Data](/Software%20Engineering/Databases/SQL%20ANSI/Grouping%20Data)).
</Note>

## Performing mathematical calculations

Operators: `+`, `-`, `*`, `/` (and `%` for modulo in most DBMSs). Parentheses control precedence.

### Example 4

```sql
SELECT prod_id,
       quantity,
       item_price,
       quantity * item_price AS expanded_price
FROM OrderItems
WHERE order_num = 20008;
```

**Analysis:** `expanded_price` is a calculated field: `quantity * item_price`. The client can use it like any other column.

<Warning>
**Integer division**

In PostgreSQL, SQL Server and SQLite, `7 / 2` is `3` (both integers). Cast one side: `7 / 2.0` or `CAST(a AS numeric) / b`. MySQL returns `3.5000`.
</Warning>

## Conditional fields with `CASE`

```sql
SELECT prod_name,
       prod_price,
       CASE
         WHEN prod_price < 5  THEN 'budget'
         WHEN prod_price < 10 THEN 'standard'
         ELSE 'premium'
       END AS price_band
FROM Products;
```

`CASE` is standard SQL and works everywhere. The first matching `WHEN` wins; without `ELSE`, unmatched rows get `NULL`.

## NULL in calculations

`NULL` is contagious: `5 + NULL` is `NULL`, and `'abc' || NULL` is `NULL` in PostgreSQL/SQLite/standard SQL (Oracle treats `NULL` as an empty string here; `CONCAT()` in PostgreSQL and SQL Server skips NULLs; MySQL's `CONCAT()` returns `NULL` if any argument is `NULL`).

```sql
SELECT cust_name || ' <' || COALESCE(cust_email, 'no email') || '>' AS label
FROM Customers;

SELECT order_num, quantity * item_price * (1 - COALESCE(discount, 0)) AS net
FROM OrderItems;   -- assuming a nullable discount column
```

`COALESCE(a, b, c)` returns the first non-null argument. `NULLIF(a, b)` returns `NULL` when `a = b`, handy to avoid division by zero: `total / NULLIF(count, 0)`.

## Beyond the book / Senior perspective

- **Generated columns** store (or compute) a calculated field in the table itself: `total NUMERIC GENERATED ALWAYS AS (quantity * item_price) STORED`. You can index them. See [Creating and Manipulating Tables](/Software%20Engineering/Databases/SQL%20ANSI/Creating%20and%20Manipulating%20Tables).
- **Formatting belongs at the edge.** Concatenating display strings in SQL is fine for reports; for APIs, return raw columns and let the client format (locales, currencies).
- **Money:** use `NUMERIC`/`DECIMAL`, never floating point, for currency calculations.
- **Calculated fields in `WHERE`** (e.g. `WHERE quantity * item_price > 100`) can't use a plain index. If it's a hot filter, add an expression index or a generated column.

## Interview questions

<Accordion title="Why can't I use a column alias in WHERE?">
Logically, `WHERE` runs before the `SELECT` list is computed, so the alias doesn't exist yet. Repeat the expression, or wrap the query in a subquery/CTE and filter outside. `ORDER BY` runs after `SELECT`, so aliases work there.
</Accordion>

<Accordion title="What does `COALESCE` do? How does it differ from `ISNULL`/`IFNULL`/`NVL`?">
Returns the first non-null argument; it's standard and takes any number of arguments. `ISNULL` (SQL Server), `IFNULL` (MySQL/SQLite) and `NVL` (Oracle) are two-argument dialect versions.
</Accordion>

<Accordion title="What is `5 / 2` in PostgreSQL?">
`2`. Integer divided by integer is integer division. Use `5 / 2.0` or a cast.
</Accordion>

<Accordion title="How do you concatenate strings portably?">
`CONCAT()` exists in PostgreSQL, MySQL, SQL Server 2012+ and Oracle (two args). `||` is the standard operator but means OR in MySQL by default.
</Accordion>

## Related
- [SQL ANSI](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20ANSI) · [Databases Index](/Software%20Engineering/Databases/Databases%20Index)
- [Using Data Manipulation Functions](/Software%20Engineering/Databases/SQL%20ANSI/Using%20Data%20Manipulation%20Functions) · [Summarizing Data](/Software%20Engineering/Databases/SQL%20ANSI/Summarizing%20Data) · [Creating and Manipulating Tables](/Software%20Engineering/Databases/SQL%20ANSI/Creating%20and%20Manipulating%20Tables) · [Using Views](/Software%20Engineering/Databases/SQL%20ANSI/Using%20Views)

## Sources
- Ben Forta, *Sams Teach Yourself SQL in 10 Minutes a Day*, 5th ed., Lesson 7 ([O'Reilly](https://learning.oreilly.com/library/view/sams-teach-yourself/9780135182925/))
- [PostgreSQL docs: Conditional expressions (CASE, COALESCE, NULLIF)](https://www.postgresql.org/docs/current/functions-conditional.html)
- [PostgreSQL docs: String functions and operators](https://www.postgresql.org/docs/current/functions-string.html)
- [MySQL docs: CONCAT()](https://dev.mysql.com/doc/refman/8.4/en/string-functions.html#function_concat)
