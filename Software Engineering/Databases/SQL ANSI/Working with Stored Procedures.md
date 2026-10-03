---
tags: [databases, databases/sql, databases/postgres]
aliases: [Stored Procedures, CREATE PROCEDURE, CALL, PL/pgSQL, Procedures vs Functions]
updated: 2026-09-27
---
# Working with Stored Procedures

> [!tldr] TL;DR
> A stored procedure is a named, saved block of SQL (plus procedural logic) that runs inside the database. It encapsulates multi-step operations, can enforce security and consistency, and saves round trips. Syntax is almost completely **non-portable** (T-SQL, PL/SQL, PL/pgSQL, MySQL). In PostgreSQL, **functions** return values and run inside the caller's transaction, while **procedures** (PostgreSQL 11+) are invoked with `CALL` and can `COMMIT`/`ROLLBACK` inside.

> Course: SQL in 10 Minutes, Lesson 19 · Prev: [[Using Views]] · Next: [[Managing Transaction Processing]]

## Understanding stored procedures

Most real operations need several statements. Processing an order, for example:
1. Check that the items are in stock.
2. If they are, reserve them so they aren't sold to anyone else, and reduce the available quantity.
3. Order any items not in stock, which involves the vendor.
4. Tell the customer which items ship now and which are back-ordered.

**Stored procedure:** a collection of one or more SQL statements saved for future use. Think of it as a batch file, though it's more than that: variables, conditions, loops and error handling.

> [!note] SQLite does not support stored procedures. (Use application code or triggers.)

## Why use stored procedures

1. **Simplify** complex operations by wrapping them in one easy-to-use unit.
2. **Consistency:** everyone runs the same steps, so they aren't reinvented (differently) in every app.
3. **Prevent errors:** fewer steps written by hand, fewer mistakes.
4. **Simpler change management:** if tables, columns or business rules change, only the procedure changes; callers don't need to know.
5. **Security:** users can be allowed to run a procedure without direct access to the underlying tables.
6. **Performance:** the DBMS can cache the plan, and one `CALL` replaces many client-server round trips.
7. **Power:** procedural features (variables, loops, exception handling) not available in single statements.

> [!note]
> Stored procedure syntax varies **dramatically** between DBMSs; truly portable procedures are close to impossible. Keep the *calls* (names and parameters) stable so client code survives a DBMS change.

### Advantages vs disadvantages

| Advantages | Disadvantages |
|---|---|
| Fewer network round trips for multi-step work | Vendor lock-in: code must be rewritten to switch DBMS |
| Logic enforced for every client (apps, scripts, BI tools) | Harder to version, code-review, test and debug than app code |
| Fine-grained security (grant `EXECUTE`, not table access) | Business logic split between app and DB; "hidden" behaviour |
| Plan caching; work happens next to the data | Scaling: the database is the hardest tier to scale out, and CPU spent there is expensive |
| Atomic multi-statement operations | Deployments need migrations; rolling back procedure versions is awkward |

## Executing stored procedures

The standard statement is `EXECUTE` (SQL Server: `EXEC`); PostgreSQL, MySQL and Oracle use `CALL`. It takes the procedure name and any parameters.

### Example 1

```sql
EXECUTE AddNewProduct('JTS01',
                      'Stuffed Eiffel Tower',
                      6.49,
                      'Plush stuffed toy with the text La Tour Eiffel in red white and blue');
```

**Analysis:** runs a procedure named `AddNewProduct` that adds a product to `Products`. It takes four parameters (vendor ID, product name, price, description), which map to four variables declared in the procedure.

`Products` also needs `prod_id`, the primary key. Why isn't it passed in? To make sure IDs are generated correctly, it's safer to automate that inside the procedure rather than trust callers. The procedure:
1. validates the passed data (all four parameters present);
2. generates a unique ID for the primary key;
3. inserts the new product with the generated key and the passed data.

## Creating stored procedures (SQL Server examples)

### Example 2

```sql
CREATE PROCEDURE MailingListCount
AS
DECLARE @cnt INTEGER
SELECT @cnt = COUNT(*)
FROM Customers
WHERE NOT cust_email IS NULL;
RETURN @cnt;
```

**Analysis:** takes no parameters. A local variable `@cnt` is declared with `DECLARE` (SQL Server local variables start with `@`), filled by `COUNT(*)`, and returned with `RETURN @cnt`. The caller reads it through SQL Server's return-code support.

### Example 3

```sql
DECLARE @ReturnValue INT
EXECUTE @ReturnValue = MailingListCount;
SELECT @ReturnValue;
```

**Analysis:** declares a variable for the return value, executes the procedure, and displays the value with `SELECT`.

### Example 4

```sql
CREATE PROCEDURE NewOrder @cust_id CHAR(10)
AS
-- Declare variable for order number
DECLARE @order_num INTEGER
-- Get current highest order number
SELECT @order_num = MAX(order_num)
FROM Orders
-- Determine next order number
SELECT @order_num = @order_num + 1
-- Insert new order
INSERT INTO Orders(order_num, order_date, cust_id)
VALUES(@order_num, GETDATE(), @cust_id)
-- Return order number
RETURN @order_num;
```

**Analysis:** creates a new order. It takes one parameter, the customer ID; the order number and date are generated inside. It finds the current highest order number with `MAX()`, adds 1, inserts the order with the current date (`GETDATE()`) and returns the new number. Note the comments: always comment stored procedures.

> [!warning] `MAX() + 1` is a race condition
> Two sessions running `NewOrder` at the same time can read the same `MAX` and insert the same number (one then fails on the primary key, or worse, if there's no PK). Use an identity column or sequence and return the generated value (`SCOPE_IDENTITY()` / `OUTPUT inserted.order_num` in SQL Server, `RETURNING` in PostgreSQL).

## Stored procedures in PostgreSQL

PostgreSQL long had only **functions**. Version 11 (2018) added **procedures** as a separate object, created with `CREATE PROCEDURE` and invoked with `CALL`.

### `CREATE OR REPLACE PROCEDURE`

`CREATE OR REPLACE PROCEDURE` creates a new procedure or replaces an existing definition (keeping its owner and permissions). Syntax (abridged):

```sql
CREATE [ OR REPLACE ] PROCEDURE
    name ( [ [ argmode ] [ argname ] argtype [ { DEFAULT | = } default_expr ] [, ...] ] )
  { LANGUAGE lang_name
    | TRANSFORM { FOR TYPE type_name } [, ... ]
    | [ EXTERNAL ] SECURITY INVOKER | [ EXTERNAL ] SECURITY DEFINER
    | SET configuration_parameter { TO value | = value | FROM CURRENT }
    | AS 'definition'
    | AS 'obj_file', 'link_symbol'
    | sql_body
  } ...
```

1. If a schema name is given (`finance.transfer`), the procedure is created there; otherwise in the first schema of the search path.
2. `CREATE OR REPLACE` changes an existing definition; ownership and permissions stay the same. You can't change the argument types this way (that creates a different, overloaded procedure).
3. You need `USAGE` privilege on the language and on the argument types.
4. The creator becomes the owner.

### Workflow example: transferring money

**Step 1:** create the table and some data. (The original example created `stored_Procedure_Prac` but the procedure updated `accounts`; here both use `accounts`.)

```sql
CREATE TABLE accounts (
    id      INT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    name    VARCHAR(100)  NOT NULL,
    balance DEC(15, 2)    NOT NULL CHECK (balance >= 0)
);

INSERT INTO accounts (name, balance) VALUES ('Don', 400000);
INSERT INTO accounts (name, balance) VALUES ('Bob', 400000);
```

**Step 2:** look at the table. (The screenshots below were taken with the original table name; the data is the same.)

```sql
SELECT * FROM accounts;
```

![[image-7_1600384369c.png]]

**Step 3:** create a procedure `prac_transfer` that moves an amount from one account to another.

```sql
CREATE OR REPLACE PROCEDURE prac_transfer(
    sender   INT,
    receiver INT,
    amount   DEC
)
LANGUAGE plpgsql
AS $$
BEGIN
    IF amount <= 0 THEN
        RAISE EXCEPTION 'amount must be positive, got %', amount;
    END IF;

    -- subtract the amount from the sender's account
    UPDATE accounts
    SET balance = balance - amount
    WHERE id = sender;

    IF NOT FOUND THEN
        RAISE EXCEPTION 'sender % not found', sender;
    END IF;

    -- add the amount to the receiver's account
    UPDATE accounts
    SET balance = balance + amount
    WHERE id = receiver;

    IF NOT FOUND THEN
        RAISE EXCEPTION 'receiver % not found', receiver;
    END IF;

    COMMIT;  -- allowed in a procedure (not in a function), when CALLed outside an explicit transaction block
END;
$$;
```

If either `UPDATE` fails (for example the `CHECK (balance >= 0)` constraint is violated, or an exception is raised), the whole transfer is rolled back: no half-transfers.

**Step 4:** call it.

```sql
CALL prac_transfer(1, 2, 1000);
```

![[image-9_1600406d03b.png]]

**Step 5:** verify.

```sql
SELECT * FROM accounts;
```

![[image-10_160041025f2.png]]

Don now has 399,000 and Bob 401,000.

> [!warning] Concurrency
> Two concurrent transfers from the same account are safe here because `balance = balance - amount` is evaluated against the latest committed row under a row lock. Reading the balance into a variable first and then writing it back would **not** be safe. When moving money between two rows, lock them in a consistent order (e.g. by id) to avoid deadlocks.

## PostgreSQL procedures vs functions

A function returns a result and can be used inside queries; a procedure performs an action and returns control to the caller (optionally with `INOUT` parameters).

**Procedure:**

```sql
CREATE PROCEDURE deactivate_unpaid_accounts()
LANGUAGE SQL
AS $$
  UPDATE accounts SET active = false WHERE balance < 0;
$$;

CALL deactivate_unpaid_accounts();
```

**Function:**

```sql
CREATE FUNCTION deactivate_unpaid_accounts_fn() RETURNS integer
LANGUAGE plpgsql
AS $$
DECLARE n integer;
BEGIN
  UPDATE accounts SET active = false WHERE balance < 0;
  GET DIAGNOSTICS n = ROW_COUNT;
  RETURN n;   -- how many accounts were deactivated
END;
$$;

SELECT deactivate_unpaid_accounts_fn();
```

(These assume an `active boolean` column on `accounts`.)

| | Function | Procedure (PG 11+) |
|---|---|---|
| Invoked with | `SELECT f()` or inside any expression | `CALL p()` |
| Returns | a value, a row, or a set of rows (`RETURNS TABLE`) | nothing, or values through `INOUT`/`OUT` parameters |
| Usable inside `SELECT`/`INSERT`/`UPDATE`/`DELETE` | Yes | No |
| Transaction control (`COMMIT`/`ROLLBACK` inside) | **No**, runs inside the caller's transaction | **Yes**, when called outside an explicit transaction block |
| Typical use | computations, set-returning queries, trigger functions | batch jobs that commit in chunks, multi-step maintenance |

Historically, PostgreSQL only had functions; trigger functions were sometimes called "trigger procedures", and the system catalog is still named `pg_proc`. Features people associate with procedures (output parameters, protection from SQL injection through parameters) apply to functions too.

## MySQL flavour, for comparison

```sql
DELIMITER //
CREATE PROCEDURE order_count(IN p_cust_id CHAR(10), OUT p_count INT)
BEGIN
  SELECT COUNT(*) INTO p_count FROM Orders WHERE cust_id = p_cust_id;
END //
DELIMITER ;

CALL order_count('1000000001', @n);
SELECT @n;
```

## Beyond the book / Senior perspective

- **Where should business logic live?** Most modern teams keep it in the application (easier to test, deploy and scale) and use the database for data integrity (constraints) and set-based operations. Procedures still shine for batch jobs, data-heavy transformations, and when many different clients must share one rule.
- **SQL injection still applies** inside procedures that build dynamic SQL. In PL/pgSQL use `EXECUTE format('... %I ...', ident) USING value` (`%I` quotes identifiers, `USING` passes values as parameters); never concatenate.
- **`SECURITY DEFINER`** runs with the owner's privileges: powerful for controlled access, dangerous if `search_path` isn't pinned (`SET search_path = pg_catalog, public`), because a caller could shadow functions.
- **Chunked maintenance:** a procedure that loops over batches with `COMMIT` between them avoids one giant transaction (lock time, bloat, replication lag).
- **Version them as migrations** and test them (pgTAP for PostgreSQL, tSQLt for SQL Server).

## Interview questions

> [!question]- What's the difference between a stored procedure and a function in PostgreSQL?
> Functions return values, can be used in queries, and always run inside the caller's transaction. Procedures (PG 11+) are invoked with `CALL`, can't be used in expressions, and can `COMMIT`/`ROLLBACK` internally.

> [!question]- Pros and cons of stored procedures?
> Pros: fewer round trips, shared logic for all clients, security via `EXECUTE` grants, atomic multi-step operations. Cons: vendor lock-in, harder testing/versioning/debugging, logic hidden from app developers, and it loads the hardest-to-scale tier.

> [!question]- What's wrong with generating IDs with `SELECT MAX(id) + 1`?
> Concurrent callers can read the same max and collide. Use sequences/identity columns and return the generated key.

> [!question]- How do you avoid SQL injection in dynamic SQL inside a procedure?
> Pass values as bind parameters (`EXECUTE ... USING` in PL/pgSQL, `sp_executesql` with parameters in SQL Server), quote identifiers with `format('%I')`/`QUOTENAME`, and never concatenate user input.

## Related
- [[SQL ANSI]] · [[Databases Index]]
- [[Managing Transaction Processing]] · [[SQL Trigger Functions]] · [[Understanding Advanced SQL Features]] · [[Transactions & ACID]] · [[Advanced Filtering Data]] (SQL injection)

## Sources
- Ben Forta, *Sams Teach Yourself SQL in 10 Minutes a Day*, 5th ed., Lesson 19 ([O'Reilly](https://learning.oreilly.com/library/view/sams-teach-yourself/9780135182925/))
- [PostgreSQL docs: CREATE PROCEDURE](https://www.postgresql.org/docs/current/sql-createprocedure.html) and [CALL](https://www.postgresql.org/docs/current/sql-call.html)
- [PostgreSQL docs: PL/pgSQL transaction management](https://www.postgresql.org/docs/current/plpgsql-transactions.html)
- [DBA Stack Exchange: Stored procedures vs stored functions](https://dba.stackexchange.com/questions/2357/what-are-the-differences-between-stored-procedures-and-stored-functions)
- [MySQL docs: CREATE PROCEDURE](https://dev.mysql.com/doc/refman/8.4/en/create-procedure.html)
