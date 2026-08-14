# 10 · Project: Normalized Schema & Analytics

This project builds a small e-commerce analytics database from scratch,
applying the normalization, indexing, and query techniques from this
level: a normalized multi-table schema, seed data, and a set of real
analytics queries against it — including a look at how the query planner
executes one of them.

## Schema

```sql
PRAGMA foreign_keys = ON;

CREATE TABLE customers (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
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
    price REAL NOT NULL CHECK (price > 0)
);

CREATE TABLE orders (
    id INTEGER PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(id),
    placed_at TEXT NOT NULL
);

CREATE TABLE order_items (
    id INTEGER PRIMARY KEY,
    order_id INTEGER NOT NULL REFERENCES orders(id),
    product_id INTEGER NOT NULL REFERENCES products(id),
    quantity INTEGER NOT NULL CHECK (quantity > 0)
);

CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_order_items_product_id ON order_items(product_id);
CREATE INDEX idx_products_category_id ON products(category_id);
```

This is the same shape covered in "Normalization & Schema Design":
`customers` and `products` each store their own facts once, `orders` links
a customer to a point in time, and `order_items` is the junction table for
the many-to-many between orders and products, carrying `quantity` as data
specific to that pairing. Every foreign key column is indexed, since every
one of them gets joined on below.

## Seed data

```sql
INSERT INTO customers (name, country, signed_up_at) VALUES
    ('Amara Osei', 'GH', '2025-01-10'),
    ('Bo Lindqvist', 'SE', '2025-02-14'),
    ('Chen Wei', 'CN', '2025-01-22'),
    ('Dara Kim', 'KR', '2025-03-05'),
    ('Elin Vartan', 'FI', '2025-02-28');

INSERT INTO categories (name) VALUES ('Electronics'), ('Books'), ('Home');

INSERT INTO products (name, category_id, price) VALUES
    ('Wireless Mouse', 1, 25.99),
    ('Mechanical Keyboard', 1, 89.50),
    ('SQL for Dummies', 2, 19.99),
    ('Novel: The Long Road', 2, 14.50),
    ('Ceramic Mug', 3, 9.99),
    ('Desk Lamp', 3, 34.00);

INSERT INTO orders (customer_id, placed_at) VALUES
    (1, '2025-03-01'), (1, '2025-04-15'), (2, '2025-03-10'), (3, '2025-03-20'),
    (3, '2025-05-01'), (4, '2025-04-02'), (5, '2025-04-20'), (2, '2025-05-15');

INSERT INTO order_items (order_id, product_id, quantity) VALUES
    (1, 1, 2), (1, 3, 1),
    (2, 2, 1),
    (3, 4, 3), (3, 5, 2),
    (4, 6, 1), (4, 1, 1),
    (5, 2, 1), (5, 3, 1),
    (6, 5, 4),
    (7, 6, 2), (7, 4, 1),
    (8, 1, 1), (8, 2, 1);
```

## Running it

### Revenue per customer, ranked

```sql
SELECT c.name, ROUND(SUM(oi.quantity * p.price), 2) AS total_spent
FROM customers c
JOIN orders o ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
GROUP BY c.id
ORDER BY total_spent DESC;
```

```text
name          total_spent
------------  -----------
Bo Lindqvist  178.97
Chen Wei      169.48
Amara Osei    161.47
Elin Vartan   82.5
Dara Kim      39.96
```

### Revenue per category

```sql
SELECT cat.name, ROUND(SUM(oi.quantity * p.price), 2) AS revenue
FROM categories cat
JOIN products p ON p.category_id = cat.id
JOIN order_items oi ON oi.product_id = p.id
GROUP BY cat.id
ORDER BY revenue DESC;
```

```text
name         revenue
-----------  -------
Electronics  372.46
Home         161.94
Books        97.98
```

### Monthly revenue trend

```sql
SELECT strftime('%Y-%m', o.placed_at) AS month,
       ROUND(SUM(oi.quantity * p.price), 2) AS revenue
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
GROUP BY month
ORDER BY month;
```

```text
month    revenue
-------  -------
2025-03  195.44
2025-04  211.96
2025-05  224.98
```

### Best-selling product by units

```sql
SELECT p.name, SUM(oi.quantity) AS units
FROM order_items oi
JOIN products p ON p.id = oi.product_id
GROUP BY p.id
ORDER BY units DESC
LIMIT 3;
```

```text
name                   units
---------------------  -----
Ceramic Mug            6
Wireless Mouse         4
Novel: The Long Road   4
```

### Above-average spenders, with the average alongside each row

```sql
WITH spend AS (
    SELECT c.id, c.name, SUM(oi.quantity * p.price) AS total
    FROM customers c
    JOIN orders o ON o.customer_id = c.id
    JOIN order_items oi ON oi.order_id = o.id
    JOIN products p ON p.id = oi.product_id
    GROUP BY c.id
)
SELECT name, ROUND(total, 2) AS total, ROUND(AVG(total) OVER (), 2) AS avg_spend
FROM spend
WHERE total > (SELECT AVG(total) FROM spend);
```

```text
name          total   avg_spend
------------  ------  ---------
Amara Osei    161.47  169.97
Bo Lindqvist  178.97  169.97
Chen Wei      169.48  169.97
```

The CTE (`spend`) computes each customer's total once; the outer query
filters against a scalar subquery average while the window function
`AVG(total) OVER ()` shows that same average on every surviving row without
a second `GROUP BY` — Dara Kim and Elin Vartan, the two below-average
spenders, are correctly excluded.

### Checking the plan on the customer-revenue query

```sql
EXPLAIN QUERY PLAN
SELECT c.name, SUM(oi.quantity * p.price) AS total_spent
FROM customers c
JOIN orders o ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
GROUP BY c.id;
```

```text
QUERY PLAN
|--SCAN oi
|--SEARCH o USING INTEGER PRIMARY KEY (rowid=?)
|--SEARCH c USING INTEGER PRIMARY KEY (rowid=?)
|--SEARCH p USING INTEGER PRIMARY KEY (rowid=?)
`--USE TEMP B-TREE FOR GROUP BY
```

The planner drives the join from `order_items` (the largest table, and the
one with no equality filter to search by) and reaches every other table
through its `INTEGER PRIMARY KEY` — the fastest possible lookup in SQLite,
since the primary key *is* the table's row storage order. `USE TEMP B-TREE
FOR GROUP BY` shows the grouping needs its own sort step because the driving
scan (`oi`) isn't already ordered by `customer_id`; on a much larger dataset
that's the step worth watching if this query got slow.

## Stretch goals

1. Add a `reviews` table (`product_id`, `customer_id`, `rating` 1-5,
   `body`) and write a query ranking products by average rating, excluding
   products with fewer than 2 reviews.
2. Add a `discounts` table keyed by `category_id` with a `percent_off`
   column, and rewrite the revenue-per-category query to apply the
   discount before summing.
3. Rewrite the monthly revenue trend query to also show each month's
   percent change from the previous month, using `LAG()`.
4. Create a covering index that lets the customer-revenue query's `GROUP
   BY` avoid the `TEMP B-TREE` step, and confirm the change in `EXPLAIN
   QUERY PLAN` output.
