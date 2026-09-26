# Stored Procedures in PostgreSQL (PL/pgSQL)

## What is a Stored Procedure?

PostgreSQL has two related but distinct server-side object types:

- **Functions** (`CREATE FUNCTION`) — existed for decades, can return values, and (prior to procedures) were the main way to encapsulate logic. Cannot manage their own transactions.
- **Procedures** (`CREATE PROCEDURE`, added in PostgreSQL 11) — invoked with `CALL`, do not return a value directly (though they can have `INOUT` parameters), and **can** run `COMMIT`/`ROLLBACK` internally, controlling their own transactions.

Both are most commonly written in **PL/pgSQL**, PostgreSQL's procedural language extension of SQL (other languages like PL/Python or PL/Perl are also supported).

Syntax basics:
```sql
CREATE PROCEDURE procedure_name(param1 INT, param2 TEXT)
LANGUAGE plpgsql
AS $$
BEGIN
    -- procedure body
END;
$$;
```

---

## Example 1: Basic Procedure with Input Parameters

Since procedures cannot return a result set as a single "table," this example uses a **function** for read/query logic (the idiomatic choice in Postgres for returning rows), and shows the plain procedure form afterward.

```sql
CREATE OR REPLACE FUNCTION get_customer_orders(
    p_customer_id INT,
    p_start_date  DATE,
    p_end_date    DATE
)
RETURNS TABLE (
    order_id     INT,
    order_date   DATE,
    total_amount NUMERIC
)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT o.order_id, o.order_date, o.total_amount
    FROM orders o
    WHERE o.customer_id = p_customer_id
      AND o.order_date BETWEEN p_start_date AND p_end_date
    ORDER BY o.order_date DESC;
END;
$$;
```

**Execute it:**
```sql
SELECT * FROM get_customer_orders(101, '2026-01-01', '2026-06-30');
```

**Explanation:**
- `RETURNS TABLE (...)` defines the shape of rows the function yields; `RETURN QUERY` executes a query and streams its rows out as the function's result.
- Functions are called like a table source (`SELECT * FROM function(...)`), which is why Postgres developers reach for functions — not procedures — whenever a result set needs to come back.
- `$$ ... $$` is **dollar-quoting**, letting you write a multi-line function body without escaping embedded quotes.

---

## Example 2: A True Procedure with Transaction Control and Error Handling

Inserts a new product using an actual `PROCEDURE`, demonstrating `INOUT` parameters and exception handling. This also shows why procedures exist: they can issue `COMMIT` mid-execution, which functions cannot.

```sql
CREATE OR REPLACE PROCEDURE add_product(
    p_name         VARCHAR(100),
    p_price        NUMERIC(10,2),
    p_category_id  INT,
    INOUT p_new_product_id INT DEFAULT NULL
)
LANGUAGE plpgsql
AS $$
BEGIN
    BEGIN
        INSERT INTO products (name, price, category_id)
        VALUES (p_name, p_price, p_category_id)
        RETURNING product_id INTO p_new_product_id;

        COMMIT;
    EXCEPTION
        WHEN OTHERS THEN
            ROLLBACK;
            RAISE EXCEPTION 'Failed to add product: %', SQLERRM;
    END;
END;
$$;
```

**Execute it:**
```sql
CALL add_product('Wireless Mouse', 19.99, 3, NULL);
```

**Explanation:**
- `INOUT` parameters serve as both input and output — the caller passes a value in (often `NULL` as a placeholder) and reads the resulting value from the same position.
- `INSERT ... RETURNING ... INTO ...` captures a generated value (Postgres has no global "last identity" function like `LAST_INSERT_ID()`/`SCOPE_IDENTITY()`; `RETURNING` is the idiomatic mechanism).
- The nested `BEGIN ... EXCEPTION WHEN OTHERS ... END` block is PL/pgSQL's structured exception handling; `SQLERRM` holds the current error message.
- Only **procedures** (not functions) can call `COMMIT`/`ROLLBACK` inside their body, because they can participate in and control the outer transaction directly.

---

## Example 3: Conditional Logic, Loops, and a Temporary Table

Generates a sales summary, optionally filtered by region, using PL/pgSQL control structures and a temporary table.

```sql
CREATE OR REPLACE PROCEDURE get_sales_summary(
    p_region TEXT DEFAULT NULL,
    p_year   INT
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_region      TEXT;
    v_total_sales NUMERIC;
    v_row_count   INT;
BEGIN
    CREATE TEMP TABLE IF NOT EXISTS sales_summary_temp (
        region      TEXT,
        total_sales NUMERIC
    ) ON COMMIT DROP;

    TRUNCATE sales_summary_temp;

    FOR v_region, v_total_sales IN
        SELECT s.region, SUM(s.amount)
        FROM sales s
        WHERE EXTRACT(YEAR FROM s.sale_date) = p_year
          AND (p_region IS NULL OR s.region = p_region)
        GROUP BY s.region
    LOOP
        INSERT INTO sales_summary_temp VALUES (v_region, v_total_sales);
    END LOOP;

    SELECT COUNT(*) INTO v_row_count FROM sales_summary_temp;

    IF v_row_count > 0 THEN
        RAISE NOTICE 'Found % region(s) with sales data.', v_row_count;
    ELSE
        RAISE NOTICE 'No sales data found for the given criteria.';
    END IF;
END;
$$;
```

**Execute it, then read the results in a follow-up query:**
```sql
CALL get_sales_summary(NULL, 2026);
-- Note: TEMP ... ON COMMIT DROP means the temp table only lives for the current
-- transaction; if autocommit is on, query it inside the same transaction/session
-- before it is dropped, e.g. by wrapping in BEGIN/COMMIT or removing ON COMMIT DROP.
```

**Explanation:**
- `FOR variable IN SELECT ... LOOP ... END LOOP` is PL/pgSQL's loop construct for iterating directly over a query's result set — no explicit cursor declaration needed (Postgres handles it implicitly).
- `RAISE NOTICE` outputs an informational message to the client, useful for logging/diagnostics inside procedures (there's no `PRINT` in Postgres).
- `CREATE TEMP TABLE ... ON COMMIT DROP` automatically cleans up the temp table at the end of the transaction.
- `p_region IS NULL OR s.region = p_region` is the same "optional filter" pattern used in T-SQL/MySQL, expressed identically in PL/pgSQL.

---

## Key PostgreSQL / PL/pgSQL Concepts Recap

| Concept | Purpose |
|---|---|
| `CREATE FUNCTION` vs `CREATE PROCEDURE` | Functions return values/tables; procedures are called with `CALL` and can control transactions |
| `$$ ... $$` | Dollar-quoting for multi-line code bodies |
| `RETURNS TABLE` / `RETURN QUERY` | Returns a result set from a function |
| `INOUT` | Parameter that is both input and output |
| `RETURNING ... INTO` | Captures a generated/returned column value |
| `EXCEPTION WHEN OTHERS` | Structured error handling; `SQLERRM` holds the error text |
| `FOR ... IN SELECT ... LOOP` | Implicit cursor iteration over a query |
| `RAISE NOTICE` / `RAISE EXCEPTION` | Emit messages or raise errors |

## Managing Procedures and Functions

```sql
\df get_customer_orders            -- (psql) list function/procedure signature
SELECT prosrc FROM pg_proc WHERE proname = 'get_customer_orders'; -- view source
DROP PROCEDURE IF EXISTS add_product(VARCHAR, NUMERIC, INT, INT);
DROP FUNCTION IF EXISTS get_customer_orders(INT, DATE, DATE);
```

> Note: Postgres requires the parameter signature (types) when dropping, since functions/procedures can be overloaded by parameter type.
