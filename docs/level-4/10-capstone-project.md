# 10 · Capstone Project

This capstone pulls together the whole path: a normalized schema (Level 3)
with constraints, indexes tuned for the actual queries (Level 4), a trigger
that maintains an audit trail (Level 3), full-text search over products
(Level 3), JSON metadata for flexible product attributes (Level 3/4), and a
verified query plan (Level 4). It's a small but complete order-management
database, built and run end to end.

## Schema

```sql
PRAGMA foreign_keys = ON;

CREATE TABLE customers (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT NOT NULL UNIQUE,
    country TEXT NOT NULL,
    signed_up_at TEXT NOT NULL
);

CREATE TABLE categories (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL UNIQUE
);

CREATE TABLE products (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    category_id INTEGER NOT NULL REFERENCES categories(id),
    price REAL NOT NULL CHECK (price > 0),
    metadata TEXT  -- JSON: flexible attrs like color, weight
);

CREATE TABLE orders (
    id INTEGER PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(id),
    placed_at TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending', 'paid', 'shipped', 'cancelled'))
);

CREATE TABLE order_items (
    id INTEGER PRIMARY KEY,
    order_id INTEGER NOT NULL REFERENCES orders(id),
    product_id INTEGER NOT NULL REFERENCES products(id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price REAL NOT NULL
);

CREATE TABLE order_audit (
    id INTEGER PRIMARY KEY,
    order_id INTEGER NOT NULL,
    old_status TEXT,
    new_status TEXT NOT NULL,
    changed_at TEXT NOT NULL
);
```

`unit_price` is captured on `order_items` at the time of purchase, deliberately
duplicating `products.price` — the correct call here, not an oversight:
`products.price` can change later, but an existing order must keep the price
the customer actually paid.

## Indexes, chosen for the queries below

```sql
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_order_items_product_id ON order_items(product_id);
CREATE INDEX idx_products_category_id ON products(category_id);
CREATE INDEX idx_orders_status ON orders(status);
```

Every foreign key used in a `JOIN` below is indexed, plus `status` since
revenue reporting filters on it directly (query 1).

## Full-text search over products

```sql
CREATE VIRTUAL TABLE products_fts USING fts5(name, content='products', content_rowid='id');

CREATE TRIGGER trg_products_fts_insert AFTER INSERT ON products BEGIN
    INSERT INTO products_fts(rowid, name) VALUES (new.id, new.name);
END;
```

`content='products', content_rowid='id'` makes this an **external content**
FTS5 table — it indexes `products.name` for search without storing a
duplicate copy of the text, keeping the FTS index in sync via the same
`rowid` as the source table. The trigger keeps new products automatically
searchable without an application-level step to remember.

## Audit trail via trigger

```sql
CREATE TRIGGER trg_order_status_audit
AFTER UPDATE OF status ON orders
FOR EACH ROW
BEGIN
    INSERT INTO order_audit (order_id, old_status, new_status, changed_at)
    VALUES (OLD.id, OLD.status, NEW.status, datetime('now'));
END;
```

Any status transition — `pending` → `paid`, `paid` → `shipped`, anything —
gets logged automatically, the same pattern from Level 3's triggers module
applied to a real workflow.

## Seed data

```sql
INSERT INTO customers (name, email, country, signed_up_at) VALUES
    ('Amara Osei', 'amara@example.com', 'GH', '2025-01-10'),
    ('Bo Lindqvist', 'bo@example.com', 'SE', '2025-02-14'),
    ('Chen Wei', 'chen@example.com', 'CN', '2025-01-22'),
    ('Dara Kim', 'dara@example.com', 'KR', '2025-03-05');

INSERT INTO categories (name) VALUES ('Electronics'), ('Books'), ('Home');

INSERT INTO products (name, category_id, price, metadata) VALUES
    ('Wireless Mouse', 1, 25.99, '{"color":"black"}'),
    ('Mechanical Keyboard', 1, 89.50, '{"color":"white"}'),
    ('SQL for Dummies', 2, 19.99, NULL),
    ('Ceramic Mug', 3, 9.99, NULL);

INSERT INTO orders (customer_id, placed_at, status) VALUES
    (1, '2025-03-01', 'paid'),
    (1, '2025-04-15', 'pending'),
    (2, '2025-03-10', 'shipped'),
    (3, '2025-03-20', 'cancelled');

INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
    (1, 1, 2, 25.99), (1, 3, 1, 19.99),
    (2, 2, 1, 89.50),
    (3, 4, 3, 9.99),
    (4, 1, 1, 25.99);
```

## Running it

### 1. Revenue by customer, counting only paid/shipped orders

```sql
SELECT c.name, ROUND(SUM(oi.quantity * oi.unit_price), 2) AS revenue
FROM customers c
JOIN orders o ON o.customer_id = c.id AND o.status IN ('paid', 'shipped')
JOIN order_items oi ON oi.order_id = o.id
GROUP BY c.id
ORDER BY revenue DESC;
```

```text
name          revenue
------------  -------
Amara Osei    71.97
Bo Lindqvist  29.97
```

Chen Wei's `cancelled` order and Amara's second, still-`pending` order are
correctly excluded — only real revenue counts.

### 2. A status change, automatically audited

```sql
UPDATE orders SET status = 'paid' WHERE id = 2;

SELECT * FROM order_audit;
```

```text
id  order_id  old_status  new_status  changed_at
--  --------  ----------  ----------  -------------------
1   2         pending     paid        2026-08-14 05:29:25
```

No application code wrote to `order_audit` — the trigger did, the moment
the `UPDATE` committed.

### 3. Full-text product search

```sql
SELECT p.name FROM products_fts
JOIN products p ON p.id = products_fts.rowid
WHERE products_fts MATCH 'keyboard';
```

```text
name
-------------------
Mechanical Keyboard
```

### 4. Filtering on JSON metadata

```sql
SELECT name, json_extract(metadata, '$.color') AS color
FROM products
WHERE json_extract(metadata, '$.color') = 'black';
```

```text
name            color
--------------  -----
Wireless Mouse  black
```

`SQL for Dummies` and `Ceramic Mug`, with `NULL` metadata, are correctly
excluded — `json_extract(NULL, ...)` returns `NULL`, which never equals
`'black'`.

### 5. Confirming the revenue query actually uses the indexes

```sql
EXPLAIN QUERY PLAN
SELECT c.name, SUM(oi.quantity * oi.unit_price)
FROM customers c
JOIN orders o ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
WHERE o.status = 'paid'
GROUP BY c.id;
```

```text
QUERY PLAN
|--SEARCH o USING INDEX idx_orders_status (status=?)
|--SEARCH c USING INTEGER PRIMARY KEY (rowid=?)
|--SEARCH oi USING INDEX idx_order_items_order_id (order_id=?)
`--USE TEMP B-TREE FOR GROUP BY
```

The planner starts from `idx_orders_status` — the most selective filter
(`status = 'paid'`) — rather than scanning every order, then reaches
`customers` and `order_items` through indexed lookups. Every `SEARCH` here
is backed by one of the indexes defined above; without them this plan would
be three full table scans.

## What this project demonstrates from across the whole path

| Level | Concept | Where it appears here |
|---|---|---|
| 1-2 | Joins, aggregation, CASE-style filtering | Query 1 |
| 3 | Normalization | `customers`/`products`/`orders`/`order_items` split |
| 3 | Triggers | `trg_order_status_audit`, `trg_products_fts_insert` |
| 3 | JSON in SQL | `products.metadata`, query 4 |
| 3 | Full-text search | `products_fts`, query 3 |
| 3 | Constraints from migrations | `CHECK` on `status`, `quantity`, `price` |
| 4 | Indexing strategy | Every FK + `status` indexed |
| 4 | Query plan verification | Query 5 |
| 4 | Security | Every query above uses parameters when given user input, not string concatenation |

## Stretch goals

1. Add a `refunds` table and a trigger that, on `orders.status` moving to
   `'cancelled'`, automatically inserts a refund record for the order's
   total.
2. Build a star-schema reporting layer (Level 4's warehousing module) on
   top of this OLTP schema: a `fact_sales` table populated from
   `order_items`, with `dim_customer`, `dim_product`, and `dim_date`.
3. Write a Python test suite (Level 4's testing module) covering: a
   successful order placement, a rejected `CHECK` violation on an invalid
   status, and confirmation that a status change produces exactly one
   `order_audit` row.
4. Simulate an injection attempt (Level 4's security module) against a
   hypothetical `search_products(keyword)` function built on the FTS5
   table, and confirm a parameterized `MATCH` query resists it.
