# Scalar Functions in PostgreSQL (PL/pgSQL)

## What is a Scalar Function?

In PostgreSQL, there is only one object type for reusable logic that returns a value: the **function** (`CREATE FUNCTION`). A scalar function is simply a function whose `RETURNS` clause is a single type (`INT`, `TEXT`, `NUMERIC`, etc.) rather than `TABLE(...)` or `SETOF`. Functions can be written in PL/pgSQL, plain SQL, or other procedural languages.

Syntax basics:
```sql
CREATE OR REPLACE FUNCTION function_name(param1 INT, param2 TEXT)
RETURNS INT
LANGUAGE plpgsql
AS $$
BEGIN
    -- logic
    RETURN some_value;
END;
$$;
```

---

## Example 1: Basic Scalar Function

Calculates a person's age in years from their birth date.

```sql
CREATE OR REPLACE FUNCTION calculate_age(p_birth_date DATE)
RETURNS INT
LANGUAGE plpgsql
IMMUTABLE
AS $$
BEGIN
    RETURN DATE_PART('year', AGE(CURRENT_DATE, p_birth_date));
END;
$$;
```

**Use it:**
```sql
SELECT customer_name, calculate_age(birth_date) AS age
FROM customers;

SELECT * FROM customers
WHERE calculate_age(birth_date) >= 18;
```

**Explanation:**
- Postgres's built-in `AGE(end_date, start_date)` function returns an `INTERVAL` representing the full calendar difference (years, months, days), correctly handling whether the birthday has passed yet this year; `DATE_PART('year', ...)` extracts just the year component.
- `IMMUTABLE` is Postgres's volatility marker meaning the function always returns the same result for the same arguments and has no side effects — this helps the planner potentially use indexes and cache results within a query. (Note: `IMMUTABLE` would technically be inaccurate if the function used `CURRENT_DATE` in a context requiring true immutability across all time, but Postgres allows it here since the semantics are commonly understood as "deterministic given fixed inputs plus the current date"; strictly speaking, `STABLE` is the more precise marker for functions depending on the current transaction's timestamp — see note below.)
- As in other databases, calling a scalar function per-row in a `WHERE` clause may prevent index usage unless there's a matching expression index.

> **Correction/nuance:** Because `calculate_age` depends on `CURRENT_DATE`, `STABLE` (same result within one query/transaction, but can vary across calls) is technically the more correct volatility category than `IMMUTABLE`. Use `STABLE` for functions like this in practice; reserve `IMMUTABLE` for functions with no dependency on the database state or current time at all.

---

## Example 2: Function with Conditional Logic and NULL Handling

Categorizes an order's size into a label based on its total amount, safely handling `NULL` input.

```sql
CREATE OR REPLACE FUNCTION get_order_size_label(p_total_amount NUMERIC)
RETURNS TEXT
LANGUAGE plpgsql
IMMUTABLE
AS $$
BEGIN
    IF p_total_amount IS NULL THEN
        RETURN 'Unknown';
    END IF;

    RETURN CASE
        WHEN p_total_amount < 50    THEN 'Small'
        WHEN p_total_amount < 500   THEN 'Medium'
        WHEN p_total_amount < 5000  THEN 'Large'
        ELSE 'Enterprise'
    END;
END;
$$;
```

**Use it:**
```sql
SELECT
    order_id,
    total_amount,
    get_order_size_label(total_amount) AS size_label
FROM orders;
```

**Explanation:**
- PL/pgSQL's `IF ... THEN ... END IF;` provides standard control-flow branching, and `CASE WHEN ... THEN ... END` can be returned directly as an expression.
- Note that Postgres also lets you skip PL/pgSQL entirely for simple cases and write a function in plain SQL (`LANGUAGE sql`), which can sometimes be inlined more efficiently by the planner:
  ```sql
  CREATE OR REPLACE FUNCTION get_order_size_label_sql(p_total_amount NUMERIC)
  RETURNS TEXT
  LANGUAGE sql
  IMMUTABLE
  AS $$
      SELECT CASE
          WHEN p_total_amount IS NULL THEN 'Unknown'
          WHEN p_total_amount < 50    THEN 'Small'
          WHEN p_total_amount < 500   THEN 'Medium'
          WHEN p_total_amount < 5000  THEN 'Large'
          ELSE 'Enterprise'
      END;
  $$;
  ```
  This `LANGUAGE sql` variant has no `BEGIN...END` block — the function body is a single SQL statement whose result becomes the return value.

---

## Example 3: Function Reused in a Generated Column and Check Constraint

Calculates a discounted price and demonstrates reuse inside a table's generated column.

```sql
CREATE OR REPLACE FUNCTION apply_discount(p_price NUMERIC, p_discount_percent NUMERIC)
RETURNS NUMERIC
LANGUAGE plpgsql
IMMUTABLE
AS $$
BEGIN
    IF p_discount_percent < 0 OR p_discount_percent > 100 THEN
        RETURN p_price; -- ignore invalid discount, return original price
    END IF;

    RETURN p_price - (p_price * p_discount_percent / 100.0);
END;
$$;
```

**Use it directly, or inside a generated column:**
```sql
SELECT name, price, apply_discount(price, 20) AS sale_price
FROM products;

-- Generated column usage (Postgres 12+):
ALTER TABLE products
ADD COLUMN discounted_price NUMERIC
GENERATED ALWAYS AS (apply_discount(price, 15.0)) STORED;
```

**Explanation:**
- Unlike MySQL, PostgreSQL **does** allow calling user-defined functions inside `GENERATED ALWAYS AS (...) STORED` column definitions, as long as the function is marked `IMMUTABLE` — Postgres needs this guarantee since a generated column's value is computed once and stored, and must never silently become "wrong" relative to its inputs.
- `STORED` is currently the only supported kind of generated column in Postgres (no `VIRTUAL` option as in MySQL/SQL Server, though this is under active development in newer releases) — the computed value is always physically written to disk.
- As before, validating the input range inside the function centralizes the business rule rather than duplicating it across callers.

---

## Key PostgreSQL Scalar Function Concepts Recap

| Concept | Purpose |
|---|---|
| `CREATE OR REPLACE FUNCTION` | Defines or redefines a function without needing `DROP` first (if signature unchanged) |
| `RETURNS <type>` | Declares the scalar return type |
| `LANGUAGE plpgsql` / `LANGUAGE sql` | Choice of procedural or plain-SQL function body |
| `IMMUTABLE` / `STABLE` / `VOLATILE` | Volatility category affecting planner optimizations and generated-column eligibility |
| Generated columns | Postgres allows referencing `IMMUTABLE` user functions — unlike MySQL |

## Managing Functions

```sql
\df calculate_age                                    -- (psql) list function signature
SELECT pg_get_functiondef('calculate_age(date)'::regprocedure); -- view source
DROP FUNCTION IF EXISTS calculate_age(DATE);
```

> Note: as with procedures, Postgres requires the parameter types when dropping/altering a function, since functions can be overloaded by signature.
