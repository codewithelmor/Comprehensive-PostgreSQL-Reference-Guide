# Comprehensive PostgreSQL Reference Guide
**Audience:** Intermediate backend developer working with a Postgres-backed API
**Target version:** PostgreSQL 18

Throughout this guide we use one consistent sample schema, a small **sales** database:

```
customers(customer_id PK, first_name, last_name, email, created_at)
employees(employee_id PK, first_name, last_name, manager_id FK->employees, hire_date)
products(product_id PK, product_name, unit_price, category_name)
orders(order_id PK, customer_id FK, employee_id FK, order_date, status)
order_items(order_item_id PK, order_id FK, product_id FK, quantity, unit_price)
```

---

## 1. Database & Server Basics

- **Cluster / instance**: one running `postgres` server process (`postmaster`) managing a shared set of databases, all sharing one data directory, one set of background processes (WAL writer, autovacuum, checkpointer), and one listening port. A single cluster can host many databases.
- **Database**: a top-level container of schemas, tables, and other objects. Unlike MySQL, a client connects to exactly **one** database per session — you cannot query across databases in a single SQL statement (only across schemas within the same database, or via `postgres_fdw`/`dblink`).
- **Schema**: a namespace *inside* a database (e.g. `public`, `sales`) used to group and secure objects. **"Database" and "schema" are NOT synonyms in Postgres** — a database contains one or more schemas, and every fresh database starts with a default `public` schema.

### Creating a database and connecting to it

```sql
-- Run while connected to any existing database (e.g. the default 'postgres' db)
CREATE DATABASE sales
    WITH OWNER = postgres
         ENCODING = 'UTF8'
         TEMPLATE = template0;
```

```
-- In psql, switch your session to the new database:
\c sales
```

### Creating a schema within a database

```sql
CREATE SCHEMA sales_data AUTHORIZATION postgres;

-- Fully qualified object name: schema.table
CREATE TABLE sales_data.customers (customer_id INT);

-- search_path controls which schema unqualified names resolve against
SHOW search_path;                 -- default: "$user", public
SET search_path TO sales_data, public;
```

### Inspecting objects (psql meta-commands and catalogs)

```
\l                      -- list databases
\dt                     -- list tables in the current schema/search_path
\dt sales_data.*        -- list tables in a specific schema
\d customers            -- describe a table: columns, types, indexes, constraints
\d+ customers           -- same, with storage/size details
\dn                     -- list schemas
\du                     -- list roles
```

```sql
-- Equivalent catalog queries (portable across tools, not just psql)
SELECT schema_name FROM information_schema.schemata;

SELECT table_schema, table_name
FROM information_schema.tables
WHERE table_type = 'BASE TABLE';

SELECT column_name, data_type, is_nullable
FROM information_schema.columns
WHERE table_name = 'customers';

-- pg_catalog is Postgres's native (faster, more detailed) system catalog
SELECT relname, relkind FROM pg_catalog.pg_class WHERE relnamespace = 'sales_data'::regnamespace;
```

**Summary:** A cluster hosts databases; each database is organized into schemas (default `public`); `\d`-family psql commands are the fastest way to inspect objects, while `information_schema` gives you a portable, standards-based alternative.

---

## 2. All PostgreSQL Data Types

| Category | Type | Size / Range | Notes / Use case |
|---|---|---|---|
| Exact numeric | `smallint` | 2 bytes, ±32,767 | Rarely used except for small enumerated codes |
| | `integer` / `int` | 4 bytes, ±2.1 billion | Default integer choice |
| | `bigint` | 8 bytes, ±9.2×10¹⁸ | Use for IDs that may exceed `int` range |
| | `decimal` / `numeric(p,s)` | variable, up to 131,072 digits | Exact — always use for money math |
| | `smallserial` | 2 bytes | Auto-increment `smallint` via a sequence |
| | `serial` | 4 bytes | Auto-increment `int` via a sequence (legacy — prefer `GENERATED ... AS IDENTITY`) |
| | `bigserial` | 8 bytes | Auto-increment `bigint` via a sequence |
| Approximate numeric | `real` | 4 bytes | ~6 decimal digits precision |
| | `double precision` | 8 bytes | ~15 decimal digits precision |
| Date/time | `date` | 4 bytes | Date only, no time |
| | `time [without time zone]` | 8 bytes | Time only, no date |
| | `time with time zone` | 12 bytes | Rarely useful — offset is fixed, not zone-aware |
| | `timestamp [without time zone]` | 8 bytes | Date + time, no zone info (stored as-is) |
| | `timestamp with time zone` (`timestamptz`) | 8 bytes | Stored in UTC internally, converted on display — **preferred for most apps** |
| | `interval` | 16 bytes | A span of time (`'3 days 4 hours'`), supports arithmetic |
| Character strings | `char(n)` | fixed, padded | Rarely needed; padding wastes space/confuses comparisons |
| | `varchar(n)` | variable, capped at n | Enforces a max length |
| | `text` | variable, unlimited | **No performance penalty vs `varchar` in Postgres** — many teams default to `text` everywhere and enforce length with a `CHECK` constraint instead |
| Binary | `bytea` | variable | Raw binary data (files, hashes) |
| Boolean | `boolean` | 1 byte | `true` / `false` / `NULL` — a real distinct type, not an int alias |
| Identifier | `uuid` | 16 bytes | Universally unique identifier; good for distributed key generation |
| Semi-structured | `json` | variable | Stores an exact text copy of the input, re-parses on every read |
| | `jsonb` | variable | Stores a decomposed binary form — supports indexing (GIN) and `@>` containment; **preferred over `json`** for anything you'll query |
| | `xml` | variable | Native XML type with `xpath()` support |
| Collection | `ARRAY` | variable | Any type can be an array, e.g. `integer[]`, `text[]` |
| Enumerated | `enum` | 4 bytes | User-defined via `CREATE TYPE ... AS ENUM (...)` |
| Currency | `money` | 8 bytes | Locale-dependent formatting; most teams avoid it in favor of `numeric` |
| Network | `cidr`, `inet`, `macaddr` | variable | Validated IP/network/MAC address types with built-in operators |
| Full-text search | `tsvector`, `tsquery` | variable | Preprocessed document / search query for `@@` full-text matching |
| Extension type | `hstore` | variable | Simple flat key-value store (requires `CREATE EXTENSION hstore`) |
| Geometric | `point`, `line`, `lseg`, `box`, `path`, `polygon`, `circle` | variable | 2D geometric primitives with built-in operators (`@>`, `<->`, etc.) |
| Range | `int4range`, `int8range`, `numrange`, `tsrange`, `tstzrange`, `daterange` | variable | Represents a range of values, e.g. a booking's date span, with overlap operators (`&&`) |
| Composite | user-defined | variable | `CREATE TYPE address AS (street text, city text, zip text)` — a structured row-like value |

**A note on extensibility:** `CREATE TYPE` lets you define enums, composite types, and full custom base types. Combined with the extension system (`CREATE EXTENSION`), this is what lets Postgres grow entirely new type systems — `postgis` (geospatial), `pgcrypto` (encryption functions), `pg_trgm` (fuzzy text search), and hundreds of others — without touching the core engine. This extensibility is one of Postgres's most distinguishing features versus MySQL/SQL Server.

```sql
-- Example: enum and array in the sample schema
CREATE TYPE order_status AS ENUM ('pending', 'shipped', 'cancelled');

ALTER TABLE orders ALTER COLUMN status TYPE order_status USING status::order_status;

-- Example: jsonb column for flexible product attributes
ALTER TABLE products ADD COLUMN attributes JSONB;
UPDATE products SET attributes = '{"color": "black", "weight_kg": 1.2}' WHERE product_id = 1;
```

**Summary:** Postgres has a far richer built-in type system than most engines — default to `text`, `timestamptz`, `numeric`, and `jsonb` unless you have a specific reason to reach for something more specialized, and lean on `CREATE TYPE`/extensions when the built-ins don't fit.

---

## 3. DDL — Data Definition Language

```sql
-- DATABASE / SCHEMA
CREATE DATABASE sales;
ALTER DATABASE sales SET timezone TO 'UTC';
DROP DATABASE sales;

CREATE SCHEMA sales_data;
DROP SCHEMA sales_data CASCADE;  -- CASCADE also drops objects inside it

-- TABLE
CREATE TABLE customers (
    customer_id  INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    first_name   TEXT NOT NULL,
    last_name    TEXT NOT NULL,
    email        TEXT NOT NULL UNIQUE,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE customers ADD COLUMN phone TEXT;
ALTER TABLE customers DROP COLUMN phone;
ALTER TABLE customers RENAME COLUMN first_name TO given_name;
DROP TABLE customers;
TRUNCATE TABLE customers;  -- fast delete-all; resets identity sequences by default only with RESTART IDENTITY

-- VIEW
CREATE VIEW customer_orders AS
SELECT c.customer_id, c.first_name, c.last_name, o.order_id, o.order_date
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id;

CREATE OR REPLACE VIEW customer_orders AS
SELECT c.customer_id, c.first_name, c.last_name, o.order_id, o.order_date, o.status
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id;

DROP VIEW customer_orders;

-- MATERIALIZED VIEW: like a view, but stores results physically until refreshed
CREATE MATERIALIZED VIEW customer_lifetime_value AS
SELECT c.customer_id, SUM(oi.quantity * oi.unit_price) AS lifetime_spend
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
JOIN order_items oi ON oi.order_id = o.order_id
GROUP BY c.customer_id;

REFRESH MATERIALIZED VIEW customer_lifetime_value;              -- blocks reads while refreshing
REFRESH MATERIALIZED VIEW CONCURRENTLY customer_lifetime_value;  -- requires a unique index; doesn't block reads
DROP MATERIALIZED VIEW customer_lifetime_value;

-- INDEX
CREATE INDEX idx_orders_customer_id ON orders (customer_id);
CREATE INDEX idx_orders_customer_id_covering ON orders (customer_id) INCLUDE (order_date, status);
ALTER INDEX idx_orders_customer_id RENAME TO idx_orders_cust;
DROP INDEX idx_orders_cust;
```

### Constraints

```sql
CREATE TABLE orders (
    order_id     INT GENERATED ALWAYS AS IDENTITY,
    customer_id  INT NOT NULL,
    employee_id  INT NOT NULL,
    order_date   TIMESTAMPTZ NOT NULL DEFAULT now(),
    status       TEXT NOT NULL DEFAULT 'pending',
    CONSTRAINT pk_orders PRIMARY KEY (order_id),
    CONSTRAINT fk_orders_customers FOREIGN KEY (customer_id)
        REFERENCES customers (customer_id),
    CONSTRAINT fk_orders_employees FOREIGN KEY (employee_id)
        REFERENCES employees (employee_id),
    CONSTRAINT ck_orders_status CHECK (status IN ('pending', 'shipped', 'cancelled'))
);
```

### Identity columns and sequences

```sql
-- GENERATED ... AS IDENTITY: SQL-standard, preferred over the legacy SERIAL pseudo-type
-- ALWAYS: application inserts can't override the value (unless OVERRIDING SYSTEM VALUE is used)
-- BY DEFAULT: closer to SERIAL's old behavior — an explicit INSERT value is allowed
CREATE TABLE log_entries (
    log_id   INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    message  TEXT
);

-- A sequence is an independent, reusable object — not tied to one table/column,
-- and it supports CYCLE/CACHE options for high-throughput generation.
CREATE SEQUENCE order_number_seq
    START WITH 100000 INCREMENT BY 1 NO CYCLE CACHE 20;

INSERT INTO orders (order_id, customer_id, employee_id)
VALUES (nextval('order_number_seq'), 1, 1);
```

**Summary:** DDL statements are largely portable ANSI SQL in Postgres, with a few distinctive extras — `CREATE MATERIALIZED VIEW`, `INCLUDE` columns on indexes, and `GENERATED ... AS IDENTITY` as the modern replacement for `SERIAL`.

---

## 4. DML — Data Manipulation Language

### SELECT

```sql
-- WHERE, ORDER BY, LIMIT/OFFSET
SELECT order_id, order_date, status
FROM orders
WHERE status = 'shipped'
ORDER BY order_date DESC
LIMIT 10 OFFSET 20;

-- GROUP BY / HAVING
SELECT customer_id, COUNT(*) AS order_count
FROM orders
GROUP BY customer_id
HAVING COUNT(*) > 5;

-- JOIN types
SELECT o.order_id, c.first_name
FROM orders o INNER JOIN customers c ON c.customer_id = o.customer_id;

SELECT c.first_name, o.order_id
FROM customers c LEFT JOIN orders o ON o.customer_id = c.customer_id;   -- keeps all customers, even with no orders

SELECT c.first_name, o.order_id
FROM customers c RIGHT JOIN orders o ON o.customer_id = c.customer_id;  -- keeps all orders, even orphaned ones

SELECT c.first_name, o.order_id
FROM customers c FULL JOIN orders o ON o.customer_id = c.customer_id;   -- keeps unmatched rows from both sides

SELECT p1.product_name, p2.product_name
FROM products p1 CROSS JOIN products p2
WHERE p1.product_id < p2.product_id;  -- every pairing, filtered to avoid duplicates/self-pairs

-- SELF JOIN: classic employee/manager example
SELECT e.first_name AS employee, m.first_name AS manager
FROM employees e
LEFT JOIN employees m ON m.employee_id = e.manager_id;
```

### INSERT

```sql
-- Single row
INSERT INTO customers (first_name, last_name, email)
VALUES ('Ana', 'Reyes', 'ana@example.com');

-- Multi-row
INSERT INTO customers (first_name, last_name, email) VALUES
    ('Ben', 'Cruz', 'ben@example.com'),
    ('Cara', 'Dizon', 'cara@example.com');

-- INSERT ... SELECT
INSERT INTO customer_orders_archive (customer_id, order_id, order_date)
SELECT customer_id, order_id, order_date FROM orders WHERE order_date < '2024-01-01';
```

### Upsert: INSERT ... ON CONFLICT

```sql
INSERT INTO products (product_id, product_name, unit_price)
VALUES (1, 'Wireless Mouse', 24.99)
ON CONFLICT (product_id) DO NOTHING;

INSERT INTO products (product_id, product_name, unit_price)
VALUES (1, 'Wireless Mouse', 22.99)
ON CONFLICT (product_id)
DO UPDATE SET unit_price = EXCLUDED.unit_price;  -- EXCLUDED refers to the row that would have been inserted
```

### UPDATE (including join-style updates with FROM)

```sql
UPDATE orders SET status = 'shipped' WHERE order_id = 1001;

-- FROM lets an UPDATE reference another table, like a join
UPDATE order_items oi
SET unit_price = p.unit_price
FROM products p
WHERE p.product_id = oi.product_id
  AND oi.unit_price IS DISTINCT FROM p.unit_price;
```

### DELETE vs TRUNCATE

```sql
DELETE FROM orders WHERE status = 'cancelled';   -- row-by-row, fires triggers, can be filtered, transactional/rollback-able
TRUNCATE TABLE orders RESTART IDENTITY;           -- instant, minimal logging, resets identity, no WHERE clause, needs table-level lock
```

### RETURNING (a distinctive Postgres feature)

```sql
-- Get generated values back without a second round-trip query
INSERT INTO customers (first_name, last_name, email)
VALUES ('Dana', 'Flores', 'dana@example.com')
RETURNING customer_id, created_at;

UPDATE orders SET status = 'shipped' WHERE order_id = 1001 RETURNING order_id, status;

DELETE FROM orders WHERE status = 'cancelled' RETURNING order_id;
```

**Summary:** Postgres's DML is standard SQL at its core, but `INSERT ... ON CONFLICT` (upserts), `UPDATE ... FROM` (join-style updates), and `RETURNING` on every write statement are ergonomic wins you won't find in every engine.

---

## 5. DCL — Data Control Language

In Postgres, a **user is just a role with the `LOGIN` privilege** — there's no separate "user" object at the catalog level. `CREATE USER` is literally shorthand for `CREATE ROLE ... LOGIN`.

```sql
CREATE ROLE reporting;                           -- a "group" role, no login
CREATE ROLE app_user WITH LOGIN PASSWORD 'change_me';  -- a login-capable role ("user")
-- Equivalent shorthand:
CREATE USER app_user2 WITH PASSWORD 'change_me';
```

### GRANT / REVOKE at each level

```sql
-- Database-level: must connect before anything else works
GRANT CONNECT ON DATABASE sales TO app_user;

-- Schema-level: required before table-level grants take effect
GRANT USAGE ON SCHEMA public TO app_user;

-- Table-level
GRANT SELECT, INSERT, UPDATE ON orders TO app_user;

-- Column-level
GRANT SELECT (customer_id, first_name) ON customers TO reporting;

-- Revoking mirrors granting
REVOKE UPDATE ON orders FROM app_user;
```

### Default privileges for future objects

```sql
-- Without this, newly created tables do NOT automatically inherit earlier grants
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT ON TABLES TO reporting;
```

### Role membership / inheritance

```sql
-- Postgres's equivalent of MySQL/SQL Server "roles": grant a group role to a login role
GRANT reporting TO app_user;   -- app_user now inherits everything reporting can do
REVOKE reporting FROM app_user;
```

### Worked example: two purpose-built roles

```sql
-- A read-only "reporting" role, granted SELECT on every table in the schema
CREATE ROLE reporting;
GRANT CONNECT ON DATABASE sales TO reporting;
GRANT USAGE ON SCHEMA public TO reporting;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO reporting;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO reporting;

-- An "app" role, scoped to only the tables the application actually touches
CREATE ROLE app_user WITH LOGIN PASSWORD 'change_me';
GRANT CONNECT ON DATABASE sales TO app_user;
GRANT USAGE ON SCHEMA public TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON customers, orders, order_items TO app_user;
```

### Verifying effective privileges

```
\dp customers        -- (or \z) shows the access privileges (ACL) on an object
```

```sql
SELECT grantee, privilege_type
FROM information_schema.table_privileges
WHERE table_name = 'customers';
```

### Removing access

```sql
REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA public FROM app_user;
DROP ROLE app_user;  -- fails if the role still owns objects or has remaining grants elsewhere
```

**Summary:** Grant order matters — `CONNECT` on the database, then `USAGE` on the schema, *then* table/column privileges — and `ALTER DEFAULT PRIVILEGES` is the easy-to-forget step that keeps future tables from silently becoming inaccessible.

---

## 6. TCL — Transaction Control Language

```sql
BEGIN;
UPDATE orders SET status = 'shipped' WHERE order_id = 1001;
SAVEPOINT before_risky_update;
UPDATE orders SET status = 'invalid_status' WHERE order_id = 1002;  -- suppose this violates a CHECK constraint
ROLLBACK TO SAVEPOINT before_risky_update;   -- undoes just the failed step, keeps the first UPDATE
COMMIT;
```

### Isolation levels and MVCC

Postgres uses **MVCC** (Multi-Version Concurrency Control): writers never block readers and vice versa, because each transaction sees a consistent snapshot of the data rather than acquiring row locks for reads.

| Isolation level | Dirty read | Non-repeatable read | Phantom read | Notes |
|---|---|---|---|---|
| Read Uncommitted | — | — | — | **Not distinct in Postgres** — behaves exactly like Read Committed |
| Read Committed *(default)* | Prevented | Possible | Possible | Each statement sees a fresh snapshot as of when it starts |
| Repeatable Read | Prevented | Prevented | Prevented* | One snapshot for the whole transaction; Postgres's implementation also prevents phantom reads, going beyond the SQL standard's minimum |
| Serializable | Prevented | Prevented | Prevented | Full serializable isolation via predicate locking; may abort transactions with a serialization-failure error that the app must retry |

- **Dirty read**: reading another transaction's uncommitted change.
- **Non-repeatable read**: re-reading the same row within a transaction and getting a different value because another transaction committed a change in between.
- **Phantom read**: re-running the same query and getting a different *set* of rows because another transaction inserted/deleted matching rows in between.

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- ... statements ...
COMMIT;
```

### Exception handling inside PL/pgSQL

Plain SQL transactions don't have "exception handling" — a failed statement aborts the whole transaction. PL/pgSQL functions, however, can catch errors and continue:

```sql
CREATE OR REPLACE FUNCTION place_order(p_customer_id INT, p_employee_id INT)
RETURNS INT AS $$
DECLARE
    v_order_id INT;
BEGIN
    INSERT INTO orders (customer_id, employee_id)
    VALUES (p_customer_id, p_employee_id)
    RETURNING order_id INTO v_order_id;

    RETURN v_order_id;
EXCEPTION
    WHEN foreign_key_violation THEN
        RAISE NOTICE 'Invalid customer_id or employee_id: %, %', p_customer_id, p_employee_id;
        RETURN NULL;
    WHEN OTHERS THEN
        RAISE NOTICE 'Unexpected error: %', SQLERRM;
        RETURN NULL;
END;
$$ LANGUAGE plpgsql;
```

**Summary:** Postgres defaults to Read Committed and implements true MVCC, so reads never block writes; reach for Repeatable Read or Serializable only when your application logic genuinely can't tolerate the default's anomalies, and be ready to retry on serialization failures.

---

## 7. Creating a Role and Granting Database Permissions (Step-by-Step)

The full, correct sequence for standing up a new database with a new login role:

```sql
-- 1. Create the database (if it doesn't exist yet)
CREATE DATABASE sales;

-- 2. Create the login role
CREATE ROLE report_reader WITH LOGIN PASSWORD 'change_me';

-- 3. Grant CONNECT — without this, the role can't even open a session
GRANT CONNECT ON DATABASE sales TO report_reader;

-- 4. Grant USAGE on the schema — required before table-level grants have any effect
GRANT USAGE ON SCHEMA public TO report_reader;

-- 5. Grant specific table privileges, then set defaults so future tables inherit them
GRANT SELECT ON ALL TABLES IN SCHEMA public TO report_reader;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO report_reader;
```

### 6. Example: a read-only "reporting" role

```sql
CREATE ROLE reporting WITH LOGIN PASSWORD 'change_me';
GRANT CONNECT ON DATABASE sales TO reporting;
GRANT USAGE ON SCHEMA public TO reporting;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO reporting;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO reporting;
```

### 7. Example: an "app" role scoped to specific tables

```sql
CREATE ROLE app WITH LOGIN PASSWORD 'change_me';
GRANT CONNECT ON DATABASE sales TO app;
GRANT USAGE ON SCHEMA public TO app;
GRANT SELECT, INSERT, UPDATE, DELETE ON customers, orders, order_items TO app;
-- Deliberately NOT granted on employees — app doesn't need it
```

### 8. Verifying effective privileges

```
\dp                                          -- all objects in the current schema, with ACLs
\z customers                                 -- alias for \dp, older name
```

```sql
SELECT table_name, grantee, privilege_type
FROM information_schema.table_privileges
WHERE grantee = 'app';
```

### 9. Removing or revoking access

```sql
REVOKE SELECT, INSERT, UPDATE, DELETE ON customers, orders, order_items FROM app;
REVOKE USAGE ON SCHEMA public FROM app;
REVOKE CONNECT ON DATABASE sales FROM app;
DROP ROLE app;  -- role must first have all its privileges/ownership cleared
```

**Summary:** The sequence is always database → schema → table, and it's easy to grant table-level privileges and then wonder why a role still can't query — check `CONNECT` and schema `USAGE` first.

---

## 8. Common Table Expressions (CTEs)

### Basic (non-recursive) CTEs

```sql
WITH big_spenders AS (
    SELECT customer_id, SUM(quantity * unit_price) AS total_spend
    FROM order_items oi
    JOIN orders o ON o.order_id = oi.order_id
    GROUP BY customer_id
    HAVING SUM(quantity * unit_price) > 500
)
SELECT c.first_name, c.last_name, bs.total_spend
FROM big_spenders bs
JOIN customers c ON c.customer_id = bs.customer_id
ORDER BY bs.total_spend DESC;
```

CTEs improve readability by naming an intermediate result and letting the main query reference it like a table.

### Materialization: an important behavior change in Postgres 12+

- **Before Postgres 12**: every CTE was always materialized (computed once, in isolation) — an "optimization fence" the planner couldn't see through.
- **Postgres 12+**: the planner may *inline* a non-recursive CTE referenced only once, as if it were a subquery, unless you force one behavior explicitly:

```sql
WITH regional_totals AS MATERIALIZED (   -- force materialization: compute once, don't inline
    SELECT category_name, SUM(quantity) AS total_qty
    FROM order_items oi JOIN products p ON p.product_id = oi.product_id
    GROUP BY category_name
)
SELECT * FROM regional_totals WHERE total_qty > 100;

WITH cheap_products AS NOT MATERIALIZED (  -- hint the planner to inline/optimize freely
    SELECT * FROM products WHERE unit_price < 20
)
SELECT * FROM cheap_products WHERE category_name = 'Accessories';
```

### Recursive CTEs: employee/manager hierarchy

```sql
WITH RECURSIVE org_chart AS (
    -- Anchor member: top-level employees (no manager)
    SELECT employee_id, first_name, manager_id, 0 AS level
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive member: join back to org_chart to walk down one level at a time
    SELECT e.employee_id, e.first_name, e.manager_id, oc.level + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.employee_id
)
SELECT * FROM org_chart
ORDER BY level, first_name;
-- Postgres has no built-in MAXRECURSION guard like SQL Server's OPTION (MAXRECURSION);
-- guard against infinite recursion with a LIMIT or an explicit level check in the recursive member instead.
```

### CTE vs subquery vs temp table vs view vs materialized view

| Approach | Scope | Materialized? | Reusable elsewhere? | Best for |
|---|---|---|---|---|
| CTE | Single statement | Sometimes (planner-dependent since PG12; force with `MATERIALIZED`) | No | Readability, recursion, breaking a query into named steps |
| Subquery | Single expression | No | No | Small, one-off filters/lookups |
| Temp table | Session | Yes — real rows, can be indexed | Within the session | Large intermediate results reused multiple times |
| View | Database object | No | Yes, by anyone with permission | Stable, reusable business definition of a query |
| Materialized view | Database object | Yes — until `REFRESH`ed | Yes, by anyone with permission | Expensive aggregations queried often, where slightly stale data is acceptable |

### Multiple chained CTEs

```sql
WITH order_totals AS (
    SELECT order_id, SUM(quantity * unit_price) AS total
    FROM order_items
    GROUP BY order_id
),
customer_totals AS (
    SELECT o.customer_id, SUM(ot.total) AS lifetime_spend
    FROM orders o
    JOIN order_totals ot ON ot.order_id = o.order_id
    GROUP BY o.customer_id
)
SELECT c.last_name, ct.lifetime_spend
FROM customer_totals ct
JOIN customers c ON c.customer_id = ct.customer_id
ORDER BY ct.lifetime_spend DESC;
```

### Writable CTEs

A distinctive Postgres feature: a CTE can wrap an `INSERT`/`UPDATE`/`DELETE ... RETURNING` and feed its result into the outer query.

```sql
WITH shipped AS (
    UPDATE orders
    SET status = 'shipped'
    WHERE status = 'pending' AND order_date < now() - INTERVAL '2 days'
    RETURNING order_id, customer_id
)
INSERT INTO order_notifications (order_id, customer_id, message)
SELECT order_id, customer_id, 'Your order has shipped!'
FROM shipped;
```

**Summary:** Use a CTE for readability and recursion, but don't assume it's an optimization fence anymore — force `MATERIALIZED` when you specifically need one, and reach for a temp table or materialized view when a result needs to be indexed or reused across multiple statements.

---

## 9. Built-in SQL Functions

### String functions

| Function | Example | Result |
|---|---|---|
| `LENGTH` | `LENGTH('Manila')` | `6` |
| `SUBSTRING` | `SUBSTRING('Manila' FROM 1 FOR 3)` | `'Man'` |
| `CONCAT` | `CONCAT(first_name, ' ', last_name)` | `'Ana Reyes'` |
| `\|\|` operator | `first_name \|\| ' ' \|\| last_name` | `'Ana Reyes'` |
| `TRIM` | `TRIM('  hi  ')` | `'hi'` |
| `LTRIM`/`RTRIM` | `LTRIM('  hi')` | `'hi'` |
| `REPLACE` | `REPLACE('a-b-c', '-', '_')` | `'a_b_c'` |
| `UPPER`/`LOWER` | `UPPER('sale')` | `'SALE'` |
| `POSITION` | `POSITION('@' IN 'ana@x.com')` | `4` |
| `STRING_AGG` | `STRING_AGG(product_name, ', ')` | `'Mouse, Keyboard'` |
| `FORMAT` | `FORMAT('%s costs $%s', product_name, unit_price)` | `'Mouse costs $24.99'` |

```sql
SELECT STRING_AGG(p.product_name, ', ' ORDER BY p.product_name) AS products
FROM order_items oi
JOIN products p ON p.product_id = oi.product_id
WHERE oi.order_id = 1001;
```

### Date/time functions

```sql
SELECT
    NOW(),                                        -- current timestamptz
    CURRENT_DATE,                                 -- current date
    AGE(NOW(), order_date) AS order_age,           -- interval since order_date
    DATE_TRUNC('month', order_date) AS order_month,
    EXTRACT(YEAR FROM order_date) AS order_year,
    TO_CHAR(order_date, 'YYYY-MM-DD') AS formatted_date,
    order_date + INTERVAL '7 days' AS due_date     -- interval arithmetic
FROM orders;

SELECT TO_DATE('2026-09-26', 'YYYY-MM-DD');
```

### Aggregate functions

```sql
SELECT COUNT(*) AS order_count, SUM(oi.quantity*oi.unit_price) AS revenue,
       AVG(oi.unit_price) AS avg_price, MIN(o.order_date) AS first_order, MAX(o.order_date) AS last_order,
       ARRAY_AGG(DISTINCT oi.product_id) AS distinct_products
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id;

-- GROUPING SETS / ROLLUP / CUBE: multiple grouping levels in one result set
SELECT c.customer_id, p.category_name, SUM(oi.quantity*oi.unit_price) AS revenue
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
JOIN products p ON p.product_id = oi.product_id
JOIN customers c ON c.customer_id = o.customer_id
GROUP BY ROLLUP (c.customer_id, p.category_name);  -- subtotals + grand total
```

### Window / analytic functions

```sql
SELECT
    o.order_id, o.customer_id, o.order_date,
    ROW_NUMBER() OVER (PARTITION BY o.customer_id ORDER BY o.order_date) AS order_seq,
    RANK()       OVER (PARTITION BY o.customer_id ORDER BY o.order_date) AS order_rank,
    DENSE_RANK() OVER (PARTITION BY o.customer_id ORDER BY o.order_date) AS order_dense_rank,
    NTILE(4)     OVER (ORDER BY o.order_date) AS quartile,
    LAG(o.order_date)  OVER (PARTITION BY o.customer_id ORDER BY o.order_date) AS prev_order_date,
    LEAD(o.order_date) OVER (PARTITION BY o.customer_id ORDER BY o.order_date) AS next_order_date,
    SUM(oi.quantity*oi.unit_price) OVER (PARTITION BY o.customer_id ORDER BY o.order_date) AS running_total,
    COUNT(*) FILTER (WHERE o.status = 'shipped') OVER (PARTITION BY o.customer_id) AS shipped_count
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id;
```

### Conversion functions

```sql
SELECT CAST('2026-09-26' AS DATE);
SELECT '2026-09-26'::DATE;              -- Postgres's :: shorthand for CAST
SELECT TO_CHAR(123456.789, 'FM999,999.00');
SELECT TO_NUMBER('1,234.50', '999,999.99');
```

### Logical / conditional functions

```sql
SELECT
    CASE WHEN status = 'shipped' THEN 'Done' ELSE 'In progress' END AS status_label,
    COALESCE(phone, 'N/A') AS phone_or_default,
    NULLIF(status, '') AS status_or_null
FROM customers c JOIN orders o ON o.customer_id = c.customer_id;
```

### JSON/JSONB functions

```sql
-- ->  returns JSON, ->> returns text, #> and #>> do the same for a path of keys
SELECT attributes -> 'color' AS color_json,
       attributes ->> 'color' AS color_text,
       attributes #>> '{dimensions,height}' AS height
FROM products;

SELECT jsonb_set(attributes, '{color}', '"red"') FROM products WHERE product_id = 1;
SELECT jsonb_build_object('id', product_id, 'name', product_name) FROM products;

-- @> containment: does the left JSON contain the right JSON as a subset?
SELECT * FROM products WHERE attributes @> '{"color": "black"}';
```

### Array functions

```sql
SELECT ARRAY_AGG(product_id) AS product_ids FROM order_items WHERE order_id = 1001;

SELECT * FROM products WHERE product_id = ANY(ARRAY[1,2,3]);

SELECT unnest(ARRAY['a','b','c']) AS item;  -- expands an array into rows
```

### System / session functions

```sql
SELECT CURRENT_USER, SESSION_USER;
SELECT LASTVAL();                                       -- last value from any sequence in this session
SELECT currval('order_number_seq');                     -- last value from a specific sequence in this session
SELECT pg_typeof(order_date) FROM orders LIMIT 1;        -- introspect a runtime value's type
```

**Summary:** Reach for `jsonb` operators and `STRING_AGG`/window functions before hand-rolling logic in application code — Postgres's function library covers most reporting and transformation needs natively.

---

## 10. Best Practices & Gotchas

**Indexing**
- B-tree is the default and right choice for most equality/range lookups (`=`, `<`, `BETWEEN`, `ORDER BY`).
- Use **GIN** indexes for `jsonb` containment queries (`@>`), array membership, and full-text search (`tsvector`).
- Use **GiST** for geometric types, range types, and exclusion constraints.
- Add indexes on foreign key columns — Postgres does **not** create them automatically (unlike the primary key side, which always gets one).

**Common pitfalls**
- **Unquoted identifiers are folded to lowercase.** `CREATE TABLE Orders (...)` creates a table named `orders`; referencing `"Orders"` (quoted) afterward will fail with "relation does not exist." Stick to lowercase snake_case and avoid quoting entirely.
- **NULL comparisons**: `WHERE status = NULL` never matches anything — use `IS NULL` / `IS NOT NULL`. Similarly, `NULL <> NULL` is `NULL` (unknown), not `true`.
- **Implicit casting differences**: comparing `text` to `integer` columns, or relying on operator-level implicit casts, can silently prevent an index from being used — match types explicitly.
- **`SELECT *`** couples callers to schema changes and defeats "covering" indexes — list columns explicitly in production code.
- **VACUUM / autovacuum**: because of MVCC, an `UPDATE` or `DELETE` doesn't overwrite a row in place — it marks the old row version as dead and writes a new one. Autovacuum reclaims that dead space; if it falls behind (long-running transactions holding back the cleanup horizon, autovacuum tuned too conservatively), tables and indexes bloat and query performance degrades. Monitor `pg_stat_user_tables.n_dead_tup` and tune `autovacuum_vacuum_scale_factor` on hot tables rather than disabling autovacuum.

**Security**
- Follow least privilege (see Section 7) — grant only what a role needs, never blanket `ALL PRIVILEGES`.
- Never run application workloads as the `postgres` superuser — create a dedicated login role per application.
- For fine-grained, per-tenant access control beyond table/column grants, consider **row-level security** (`ALTER TABLE ... ENABLE ROW LEVEL SECURITY` + `CREATE POLICY`), which lets Postgres itself filter which rows a role can see or modify — useful for multi-tenant SaaS schemas.

**Summary:** Most production Postgres problems trace back to one of three things — a missing index (especially on foreign keys), an implicit type mismatch defeating an index, or autovacuum falling behind under MVCC — so make all three a deliberate part of code review and monitoring, not an afterthought.
