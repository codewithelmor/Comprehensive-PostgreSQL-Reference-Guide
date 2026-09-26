# SQL Views in PostgreSQL

## What is a View?

A view in PostgreSQL is a named, stored `SELECT` query that acts as a virtual table. Postgres additionally offers **materialized views**, which physically store the query's result set on disk and must be explicitly refreshed — a genuine alternative to the "logical only" views found in MySQL.

Syntax basics:
```sql
CREATE VIEW view_name AS
SELECT column1, column2
FROM some_table
WHERE condition;
```

---

## Example 1: Basic View for Simplifying a Join

Combines orders with customer names for repeated querying.

```sql
CREATE VIEW order_details AS
SELECT
    o.order_id,
    o.order_date,
    c.customer_name,
    o.total_amount
FROM orders o
INNER JOIN customers c
    ON o.customer_id = c.customer_id;
```

**Query it:**
```sql
SELECT * FROM order_details
WHERE order_date >= '2026-01-01'
ORDER BY order_date DESC;
```

**Explanation:**
- The view is queried like a normal table; Postgres's planner typically inlines the view's definition into the outer query (similar in spirit to MySQL's `MERGE` algorithm), so filters and joins against the view are usually optimized as if written directly against the base tables.
- `CREATE OR REPLACE VIEW` can be used instead of `CREATE VIEW` to redefine a view without dropping it first — as long as the new definition doesn't change existing column names/types/order (you can only append new columns at the end).

---

## Example 2: Materialized View for Precomputed Aggregation

Produces a per-customer sales summary, but — unlike a plain view — physically stores the computed rows until explicitly refreshed.

```sql
CREATE MATERIALIZED VIEW customer_sales_summary AS
SELECT
    o.customer_id,
    COUNT(*)            AS order_count,
    SUM(o.total_amount)  AS total_spent
FROM orders o
GROUP BY o.customer_id
WITH DATA;

-- Recommended: create a unique index to allow CONCURRENT refresh
CREATE UNIQUE INDEX idx_css_customer_id
ON customer_sales_summary (customer_id);
```

**Query and refresh it:**
```sql
SELECT customer_id, order_count, total_spent
FROM customer_sales_summary
WHERE total_spent > 1000;

-- Data is stale until refreshed — nothing updates it automatically
REFRESH MATERIALIZED VIEW customer_sales_summary;

-- Or refresh without blocking concurrent reads (requires the unique index above)
REFRESH MATERIALIZED VIEW CONCURRENTLY customer_sales_summary;
```

**Explanation:**
- `MATERIALIZED VIEW` computes and **stores** the query result physically, making subsequent reads fast — ideal for expensive aggregations queried often but not needing up-to-the-second freshness.
- Data becomes stale after the underlying tables change; you must run `REFRESH MATERIALIZED VIEW` (typically on a schedule via `pg_cron`, a cron job, or an application task) to bring it up to date.
- By default, `REFRESH` takes an exclusive lock, blocking reads during the refresh; `CONCURRENTLY` avoids this but requires at least one unique index on the materialized view.
- This is Postgres's closest equivalent to SQL Server's indexed views, though it's refreshed explicitly rather than being kept continuously in sync by the engine.

---

## Example 3: Updatable View with `WITH CHECK OPTION` and Security

Exposes only active customers in a specific region, restricts inserts/updates to matching rows, and demonstrates Postgres's security-focused view options.

```sql
CREATE VIEW active_west_customers
WITH (security_invoker = true)
AS
SELECT customer_id, customer_name, region, is_active
FROM customers
WHERE region = 'West' AND is_active = 1
WITH CHECK OPTION;
```

**Use it:**
```sql
-- Reads work like a normal table
SELECT * FROM active_west_customers;

-- Updates are allowed because the view is a simple single-table SELECT
UPDATE active_west_customers
SET customer_name = 'Acme West Corp'
WHERE customer_id = 55;

-- This INSERT FAILS: ERROR: new row violates check option for view "active_west_customers"
INSERT INTO active_west_customers (customer_id, customer_name, region, is_active)
VALUES (999, 'Test Co', 'East', 1);
```

**Explanation:**
- Postgres automatically allows `INSERT`/`UPDATE`/`DELETE` through "simple" views — those selecting from exactly one table/view with no aggregates, `DISTINCT`, `GROUP BY`, `HAVING`, `UNION`, or set-returning functions.
- `WITH CHECK OPTION` (optionally `LOCAL` or `CASCADED`, `CASCADED` being default) prevents writes through the view from creating rows the view itself wouldn't show.
- `WITH (security_invoker = true)` (PostgreSQL 15+) makes the view execute with the **querying user's** privileges and row-level security policies, rather than the view owner's — an important option when views are used as a security boundary. Without it, views historically run with the definer's (owner's) privileges, which can inadvertently bypass row-level security on the base table.

---

## Key PostgreSQL View Concepts Recap

| Concept | Purpose |
|---|---|
| `CREATE VIEW` / `CREATE OR REPLACE VIEW` | Defines or redefines a virtual table |
| `CREATE MATERIALIZED VIEW` | Physically stores query results; refreshed on demand |
| `REFRESH MATERIALIZED VIEW [CONCURRENTLY]` | Updates a materialized view's stored data |
| Updatable simple views | Single-table views auto-support `INSERT`/`UPDATE`/`DELETE` |
| `WITH CHECK OPTION` (`LOCAL` / `CASCADED`) | Restricts writes to rows matching the view's `WHERE` clause |
| `security_invoker` | Runs the view with the caller's privileges instead of the owner's |

## Managing Views

```sql
\d+ order_details                                  -- (psql) view definition & details
SELECT pg_get_viewdef('order_details', true);      -- view the SQL definition
DROP VIEW IF EXISTS order_details;                  -- delete a view
DROP MATERIALIZED VIEW IF EXISTS customer_sales_summary;
SELECT * FROM information_schema.views
WHERE table_schema = 'public';                      -- list all views
```
