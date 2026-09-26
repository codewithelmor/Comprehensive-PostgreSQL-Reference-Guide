# PostgreSQL JSON Column — Comprehensive Guide

## 1. Overview

PostgreSQL offers **two** native JSON types:

| Type | Storage | Notes |
|---|---|---|
| `json` | Text, exact copy of input | Preserves whitespace, key order, and duplicate keys. Re-parses on every read. |
| `jsonb` | Decomposed binary format | Removes insignificant whitespace, dedups keys (last wins), does **not** preserve key order. Faster to query; supports indexing (GIN). |

**Recommendation:** use `jsonb` for almost everything unless you specifically
need to preserve the exact original text (e.g., an audit log of the literal
payload received).

---

## 2. Creating a Table with a JSON Column

```sql
CREATE TABLE products (
    product_id  SERIAL PRIMARY KEY,
    name        TEXT NOT NULL,
    attributes  JSONB
);
```

PostgreSQL validates JSON syntax automatically for both `json` and `jsonb` —
invalid documents are rejected at insert time.

---

## 3. Inserting JSON Data

```sql
INSERT INTO products (name, attributes)
VALUES (
    'Wireless Mouse',
    '{
        "color": "black",
        "wireless": true,
        "price": 24.99,
        "dimensions": {"width": 6.5, "height": 3.2},
        "tags": ["electronics", "peripheral", "office"]
    }'::jsonb
);
```

Building JSON with functions:

```sql
INSERT INTO products (name, attributes)
VALUES (
    'Keyboard',
    jsonb_build_object(
        'color', 'silver',
        'wireless', false,
        'price', 49.99,
        'tags', jsonb_build_array('electronics')
    )
);
```

---

## 4. Querying JSON — Operators & Extraction

### 4.1 Core operators

| Operator | Returns | Example |
|---|---|---|
| `->` | JSON object/array (jsonb) | `attributes -> 'dimensions'` |
| `->>` | text | `attributes ->> 'color'` |
| `#>` | JSON at path (array of keys) | `attributes #> '{dimensions,width}'` |
| `#>>` | text at path | `attributes #>> '{dimensions,width}'` |
| `@>` | contains (jsonb only) | `attributes @> '{"color":"black"}'` |
| `<@` | contained by | `'{"color":"black"}' <@ attributes` |
| `?` | top-level key exists | `attributes ? 'color'` |
| `?\|` | any of these keys exist | `attributes ?\| array['color','size']` |
| `?&` | all of these keys exist | `attributes ?& array['color','price']` |
| `-` | delete key/element | `attributes - 'color'` |
| `#-` | delete at path | `attributes #- '{dimensions,width}'` |

```sql
SELECT
    product_id,
    name,
    attributes->>'color'                    AS color,
    (attributes->>'price')::numeric         AS price,
    attributes#>>'{dimensions,width}'       AS width
FROM products;
```

### 4.2 Filtering with `WHERE`

```sql
-- scalar equality
SELECT * FROM products WHERE attributes->>'color' = 'black';

-- containment (uses GIN index efficiently — preferred over ->> filters)
SELECT * FROM products WHERE attributes @> '{"color": "black"}';

-- key existence
SELECT * FROM products WHERE attributes ? 'dimensions';

-- numeric comparison (cast text to numeric)
SELECT * FROM products WHERE (attributes->>'price')::numeric > 20;
```

### 4.3 Array containment / membership

```sql
-- does the tags array contain "office"?
SELECT * FROM products
WHERE attributes -> 'tags' @> '"office"';

-- alternative using jsonb_array_elements_text
SELECT * FROM products p
WHERE EXISTS (
    SELECT 1 FROM jsonb_array_elements_text(p.attributes->'tags') AS tag
    WHERE tag = 'office'
);
```

### 4.4 Expanding/shredding JSON into rows

```sql
-- expand a top-level object into key/value rows
SELECT * FROM jsonb_each(attributes) FROM products;      -- (key, value jsonb)
SELECT * FROM jsonb_each_text(attributes) FROM products;  -- (key, value text)

-- expand an array into rows
SELECT p.product_id, tag
FROM products p, jsonb_array_elements_text(p.attributes->'tags') AS tag;

-- expand nested objects with type info
SELECT * FROM jsonb_typeof(attributes) FROM products; -- 'object','array','string', etc.
```

### 4.5 `jsonb_to_record` / `jsonb_populate_record` — shred to typed columns

```sql
SELECT *
FROM products p,
LATERAL jsonb_to_record(p.attributes) AS x(color text, price numeric);
```

```sql
-- populate a full row type from JSON (useful with a matching table/type)
SELECT (jsonb_populate_record(NULL::products, attributes)).*
FROM products;
```

### 4.6 JSONPath (SQL/JSON, PostgreSQL 12+)

```sql
SELECT jsonb_path_query(attributes, '$.dimensions.width') FROM products;

SELECT * FROM products
WHERE attributes @@ '$.price > 20';

SELECT * FROM products
WHERE jsonb_path_exists(attributes, '$.tags[*] ? (@ == "office")');
```

`jsonb_path_query`, `jsonb_path_exists`, and the `@@`/`@?` operators support
the full SQL/JSON path language, including filters (`? (@ > 10)`), which is
more expressive than plain `->`/`#>` chains for conditional matching.

---

## 5. Modifying JSON

### 5.1 `jsonb_set()` — update or insert a value at a path

```sql
UPDATE products
SET attributes = jsonb_set(attributes, '{price}', '19.99', true)
WHERE product_id = 1;
-- 4th arg (create_if_missing) defaults to true
```

```sql
-- update a nested path
UPDATE products
SET attributes = jsonb_set(attributes, '{dimensions,width}', '10.0')
WHERE product_id = 1;
```

### 5.2 `jsonb_insert()` — insert only if path doesn't already hold a value

```sql
UPDATE products
SET attributes = jsonb_insert(attributes, '{discontinued}', 'false')
WHERE product_id = 1;

-- insert into an array at a specific position
UPDATE products
SET attributes = jsonb_insert(attributes, '{tags,0}', '"new"', false)
WHERE product_id = 1;
```

### 5.3 Deleting keys/elements

```sql
-- delete a top-level key
UPDATE products SET attributes = attributes - 'discontinued' WHERE product_id = 1;

-- delete a nested path
UPDATE products SET attributes = attributes #- '{dimensions,width}' WHERE product_id = 1;

-- delete an array element by index
UPDATE products SET attributes = attributes #- '{tags,0}' WHERE product_id = 1;
```

### 5.4 Merging documents (`||` concatenation operator)

```sql
-- shallow merge: right side wins on key conflicts
UPDATE products
SET attributes = attributes || '{"price": 29.99, "color": "red"}'::jsonb
WHERE product_id = 1;
```

### 5.5 Appending to an array

```sql
UPDATE products
SET attributes = jsonb_set(
    attributes,
    '{tags}',
    (attributes->'tags') || '["sale"]'::jsonb
)
WHERE product_id = 1;
```

---

## 6. Producing JSON from Relational Data

```sql
-- aggregate rows into a JSON array
SELECT jsonb_agg(jsonb_build_object('id', product_id, 'name', name))
FROM products;

-- aggregate into a JSON object keyed by id
SELECT jsonb_object_agg(product_id, name) FROM products;

-- a full row as jsonb
SELECT to_jsonb(p) FROM products p;

-- row_to_json (json, not jsonb)
SELECT row_to_json(p) FROM products p;
```

---

## 7. Indexing JSON Data

This is where `jsonb` shines — PostgreSQL supports **GIN indexes** directly on
`jsonb` columns, unlike SQL Server/MySQL which require generated/computed
columns for indexing.

### 7.1 Default GIN index (supports `@>`, `?`, `?|`, `?&`)

```sql
CREATE INDEX idx_products_attributes ON products USING GIN (attributes);
```

```sql
-- this query uses the index
SELECT * FROM products WHERE attributes @> '{"color": "black"}';
```

### 7.2 `jsonb_path_ops` (smaller, faster for `@>`, but not `?`)

```sql
CREATE INDEX idx_products_attributes_pathops
ON products USING GIN (attributes jsonb_path_ops);
```

`jsonb_path_ops` produces a smaller index and faster containment lookups,
but only supports the `@>` operator — not `?`, `?|`, `?&`.

### 7.3 Expression (B-tree) index for a specific field

```sql
CREATE INDEX idx_products_color ON products ((attributes->>'color'));
```

```sql
-- uses the expression index
SELECT * FROM products WHERE attributes->>'color' = 'black';
```

### 7.4 Generated column (PostgreSQL 12+) as an alternative

```sql
ALTER TABLE products
ADD COLUMN color TEXT GENERATED ALWAYS AS (attributes->>'color') STORED;

CREATE INDEX idx_products_color_gen ON products(color);
```

---

## 8. Validation & Constraints

```sql
-- required key
ALTER TABLE products
ADD CONSTRAINT chk_has_color CHECK (attributes ? 'color');

-- schema-like constraint on a nested value's type/range
ALTER TABLE products
ADD CONSTRAINT chk_price_positive
CHECK ((attributes->>'price')::numeric > 0);
```

For heavier schema validation, PostgreSQL doesn't ship a native JSON Schema
validator; use application-layer validation or an extension, or rely on
`CHECK` constraints for the critical invariants.

---

## 9. `json` vs `jsonb` — Choosing

| Consideration | `json` | `jsonb` |
|---|---|---|
| Preserves whitespace/key order | Yes | No |
| Duplicate keys | Kept (last wins on ->>) | Deduped (last wins) on input |
| Insert speed | Faster (no parsing) | Slightly slower (parses on write) |
| Query speed | Slower (parses on every read) | Faster (already decomposed) |
| Indexing (GIN) | Not supported | Supported |
| Containment operators (`@>`, `?`) | Not supported | Supported |

**Rule of thumb:** default to `jsonb`. Use `json` only when you must preserve
the exact textual representation (e.g., logging raw webhook payloads
verbatim).

---

## 10. Common Patterns

### 10.1 Upsert with JSON payload

```sql
INSERT INTO products (product_id, name, attributes)
VALUES (1, 'Mouse', '{"price": 24.99}')
ON CONFLICT (product_id) DO UPDATE
SET attributes = products.attributes || EXCLUDED.attributes;
```

### 10.2 Counting array elements

```sql
SELECT product_id, jsonb_array_length(attributes->'tags') AS tag_count
FROM products;
```

### 10.3 Building a nested API response directly in SQL

```sql
SELECT jsonb_build_object(
    'id', product_id,
    'name', name,
    'attributes', attributes
) FROM products;
```

---

## 11. Performance & Best Practices

1. **Default to `jsonb`**, not `json`, unless you need exact-text
   preservation.
2. **Use GIN indexes with containment (`@>`)** for flexible "document
   matches these fields" queries; use expression indexes for single-field
   equality/range filters.
3. **`jsonb_path_ops` over plain GIN** when you only need `@>` — smaller,
   faster index, at the cost of losing `?`/`?|`/`?&` support.
4. **Promote hot, frequently-filtered fields** to generated columns with
   B-tree indexes rather than relying purely on GIN + containment for
   selective, high-frequency predicates.
5. **Avoid `SELECT *` shredding patterns in hot loops** — prefer set-based
   `jsonb_to_recordset`/`jsonb_populate_recordset` over row-by-row
   application parsing.
6. **Use `||` for shallow merges** but remember it fully replaces arrays and
   top-level keys — for deep merges you need recursive logic or
   `jsonb_set` per path.
7. **Keep documents reasonably sized** — TOASTed (compressed, out-of-line)
   storage kicks in for large jsonb values, which adds I/O overhead; very
   large or deeply nested documents may benefit from being split into
   normalized tables.
8. **Leverage JSONPath (`@@`, `@?`, `jsonb_path_query`)** for conditional
   array filtering instead of correlated subqueries with
   `jsonb_array_elements`, where the path language is expressive enough.

---

## 12. Quick Reference Cheat Sheet

```sql
col -> 'key'                     -- get jsonb value
col ->> 'key'                    -- get text value
col #> '{a,b}'                   -- get jsonb value at path
col #>> '{a,b}'                  -- get text value at path
col @> '{"k":"v"}'               -- contains (GIN-indexable)
col <@ '{"k":"v"}'               -- is contained by
col ? 'key'                      -- key exists
col ?| array['a','b']            -- any key exists
col ?& array['a','b']            -- all keys exist
col - 'key'                      -- delete key
col #- '{a,b}'                   -- delete at path
col || '{"k":"v"}'               -- shallow merge / concatenate
jsonb_set(col, '{path}', val, true)
jsonb_insert(col, '{path}', val)
jsonb_build_object(k, v, ...)
jsonb_build_array(v, ...)
jsonb_agg(expr) / jsonb_object_agg(k, v)
jsonb_each(col) / jsonb_each_text(col)
jsonb_array_elements(col) / jsonb_array_elements_text(col)
jsonb_to_record(col) / jsonb_populate_record(base, col)
jsonb_path_query(col, '$.path')
jsonb_path_exists(col, '$.path ? (@ > 10)')
jsonb_typeof(col)
jsonb_array_length(col)
CREATE INDEX ... USING GIN (col [jsonb_path_ops])
```
