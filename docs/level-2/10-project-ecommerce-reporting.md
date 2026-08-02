# 10 · Project — E-commerce Reporting

A capstone project combining everything from Level 2: advanced joins,
subqueries, views, `CASE` expressions, and transactions, applied to a
four-table e-commerce schema that's realistic enough to need all of them at
once.

## What you'll build

A small store database — `customers`, `products`, `orders`, and
`order_items` — that you'll populate and then query to answer the kinds of
questions a real store's reporting dashboard would ask.

## Setting up

```sql
PRAGMA foreign_keys = ON;

CREATE TABLE customers (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT NOT NULL UNIQUE,
    country TEXT NOT NULL
);

CREATE TABLE products (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    category TEXT NOT NULL,
    price REAL NOT NULL CHECK (price > 0),
    stock INTEGER NOT NULL CHECK (stock >= 0)
);

CREATE TABLE orders (
    id INTEGER PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    order_date TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'completed',
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);

-- a junction table: each row is one product line within one order
CREATE TABLE order_items (
    order_id INTEGER NOT NULL,
    product_id INTEGER NOT NULL,
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price REAL NOT NULL,
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (order_id) REFERENCES orders(id),
    FOREIGN KEY (product_id) REFERENCES products(id)
);
```

`order_items` exists because one order can contain several products, and one
product appears in many orders — a classic many-to-many relationship
resolved with a junction table, composite `PRIMARY KEY` and all (see
[Level 2 · Constraints](08-constraints.md)). `unit_price` is stored on the
line item itself, not looked up fresh from `products`, so historical orders
stay accurate even if a product's price later changes.

## Seeding data

```sql
INSERT INTO customers (name, email, country) VALUES
    ('Alice Nguyen', 'alice@example.com', 'US'),
    ('Bob Smith', 'bob@example.com', 'UK'),
    ('Carol Diaz', 'carol@example.com', 'US'),
    ('Dave Okafor', 'dave@example.com', 'NG'),
    ('Erin Kelly', 'erin@example.com', 'IE'),
    ('Frank Ito', 'frank@example.com', 'JP');

INSERT INTO products (name, category, price, stock) VALUES
    ('Wireless Mouse', 'Electronics', 25.00, 60),
    ('Mechanical Keyboard', 'Electronics', 75.00, 8),
    ('USB-C Hub', 'Electronics', 40.00, 3),
    ('Desk Lamp', 'Home', 30.00, 25),
    ('Standing Desk', 'Home', 350.00, 4),
    ('Notebook Set', 'Office', 12.00, 100);

INSERT INTO orders (customer_id, order_date, status) VALUES
    (1, '2026-01-05', 'completed'),
    (1, '2026-02-14', 'completed'),
    (2, '2026-01-20', 'completed'),
    (3, '2026-01-25', 'completed'),
    (3, '2026-02-02', 'completed'),
    (4, '2026-02-10', 'completed'),
    (5, '2026-02-18', 'completed'),
    (2, '2026-02-25', 'cancelled');

INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
    (1, 1, 2, 25.00),
    (1, 4, 1, 30.00),
    (2, 2, 1, 75.00),
    (3, 5, 1, 350.00),
    (4, 1, 1, 25.00),
    (4, 6, 3, 12.00),
    (5, 2, 1, 75.00),
    (5, 3, 1, 40.00),
    (6, 6, 5, 12.00),
    (7, 4, 2, 30.00),
    (8, 1, 1, 25.00);
```

Note order 8 is `cancelled` and Frank has no orders at all — both deliberate,
to make sure the reporting queries below handle them correctly instead of
just working on tidy data.

## Reporting queries

**1. Top customers by total spend (completed orders only):**

```sql
SELECT c.name,
       ROUND(SUM(oi.quantity * oi.unit_price), 2) AS total_spent
FROM customers AS c
JOIN orders AS o ON o.customer_id = c.id
JOIN order_items AS oi ON oi.order_id = o.id
WHERE o.status = 'completed'
GROUP BY c.id
ORDER BY total_spent DESC
LIMIT 3;
```

```text
name          total_spent
------------  -----------
Bob Smith     350.0
Carol Diaz    176.0
Alice Nguyen  155.0
```

Filtering `o.status = 'completed'` is what correctly excludes Bob's
cancelled order from his total — without it, cancelled orders would inflate
revenue numbers that never actually happened.

**2. Monthly revenue:**

```sql
SELECT STRFTIME('%Y-%m', o.order_date) AS month,
       ROUND(SUM(oi.quantity * oi.unit_price), 2) AS revenue
FROM orders AS o
JOIN order_items AS oi ON oi.order_id = o.id
WHERE o.status = 'completed'
GROUP BY month
ORDER BY month;
```

```text
month    revenue
-------  -------
2026-01  491.0
2026-02  310.0
```

`STRFTIME('%Y-%m', ...)` (see
[Level 2 · String & Date Functions](04-string-date-functions.md)) collapses
every order date down to its year-month, so `GROUP BY month` aggregates
across the whole month regardless of the day.

**3. Low-stock products (fewer than 10 in stock):**

```sql
SELECT name, category, stock
FROM products
WHERE stock < 10
ORDER BY stock;
```

```text
name                 category     stock
-------------------  -----------  -----
USB-C Hub            Electronics  3
Standing Desk        Home         4
Mechanical Keyboard  Electronics  8
```

**4. Customers who spent above the average customer total (nested subquery):**

```sql
SELECT name, total_spent FROM (
    SELECT c.name AS name, SUM(oi.quantity * oi.unit_price) AS total_spent
    FROM customers AS c
    JOIN orders AS o ON o.customer_id = c.id
    JOIN order_items AS oi ON oi.order_id = o.id
    WHERE o.status = 'completed'
    GROUP BY c.id
)
WHERE total_spent > (
    SELECT AVG(customer_total) FROM (
        SELECT SUM(oi2.quantity * oi2.unit_price) AS customer_total
        FROM customers AS c2
        JOIN orders AS o2 ON o2.customer_id = c2.id
        JOIN order_items AS oi2 ON oi2.order_id = o2.id
        WHERE o2.status = 'completed'
        GROUP BY c2.id
    )
)
ORDER BY total_spent DESC;
```

```text
name        total_spent
----------  -----------
Bob Smith   350.0
Carol Diaz  176.0
```

This uses a subquery as a "derived table" (the outer `SELECT ... FROM (
... )` pattern) twice: once to compute each customer's total, and again,
nested one level deeper, to average those per-customer totals — the average
customer spent 160.20, so Alice's 155.0 falls just short and doesn't make the
list (see [Level 2 · Subqueries](02-subqueries.md)).

**5. A view combining products with units sold:**

```sql
CREATE VIEW product_sales AS
SELECT p.id, p.name, p.category, p.stock,
       COALESCE(SUM(oi.quantity), 0) AS units_sold
FROM products AS p
LEFT JOIN order_items AS oi ON oi.product_id = p.id
LEFT JOIN orders AS o ON o.id = oi.order_id AND o.status = 'completed'
GROUP BY p.id;

SELECT name, stock, units_sold FROM product_sales ORDER BY units_sold DESC;
```

```text
name                 stock  units_sold
-------------------  -----  ----------
Notebook Set         100    8
Wireless Mouse       60     4
Desk Lamp            25     3
Mechanical Keyboard  8      2
USB-C Hub            3      1
Standing Desk        4      1
```

The `LEFT JOIN`s (see
[Level 2 · Advanced Joins](01-advanced-joins.md)) keep every product even if
it has never sold, and the `o.status = 'completed'` filter lives in the `ON`
clause rather than `WHERE` — putting it in `WHERE` would have silently
dropped any product with zero completed sales, defeating the `LEFT JOIN`.
Once created, `product_sales` can be queried directly by any future report
(see [Level 2 · Views](06-views.md)).

**6. Customers with no completed orders:**

```sql
SELECT c.name
FROM customers AS c
WHERE NOT EXISTS (
    SELECT 1 FROM orders AS o WHERE o.customer_id = c.id AND o.status = 'completed'
);
```

```text
name
---------
Frank Ito
```

`NOT EXISTS` is the safe choice here — a `NOT IN` subquery would silently
return zero rows if any `orders.customer_id` were ever `NULL` (see the
[NOT IN + NULL trap](02-subqueries.md#the-not-in-null-trap)).

## Running it: placing an order inside a transaction

Placing a real order means inserting into `orders`, inserting into
`order_items`, and decrementing `products.stock` — three statements that
must succeed or fail together (see
[Level 2 · Transactions & ACID](09-transactions-acid.md)):

```sql
BEGIN;
INSERT INTO orders (customer_id, order_date, status) VALUES (2, '2026-03-01', 'completed');
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
    VALUES (last_insert_rowid(), 3, 2, 40.00);
UPDATE products SET stock = stock - 2 WHERE id = 3;
SELECT stock FROM products WHERE id = 3;
COMMIT;
```

```text
stock
-----
1
```

`last_insert_rowid()` grabs the `id` SQLite just assigned to the new order,
so `order_items` can reference it without a separate lookup. The USB-C Hub's
stock correctly drops from 3 to 1.

Now try to sell 50 units of a product with only 1 left in stock:

```sql
BEGIN;
INSERT INTO orders (customer_id, order_date, status) VALUES (3, '2026-03-02', 'completed');
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
    VALUES (last_insert_rowid(), 3, 50, 40.00);
UPDATE products SET stock = stock - 50 WHERE id = 3;
```

```text
Error: CHECK constraint failed: stock >= 0
```

```sql
ROLLBACK;
SELECT stock FROM products WHERE id = 3;
```

```text
stock
-----
1
```

The `CHECK (stock >= 0)` constraint from
[Level 2 · Constraints](08-constraints.md) caught the overselling attempt.
Because the failing `UPDATE` happened inside an open transaction, and the
application explicitly issued `ROLLBACK`, the stray order row that was
already inserted got discarded too — stock stayed at `1` and no orphaned
order exists for a sale that never really happened. Skipping that `ROLLBACK`
would have left a phantom `orders` row with no corresponding stock deducted,
exactly the kind of inconsistency transactions are meant to prevent.

## Stretch goals

- Add a `reviews` table (`product_id`, `customer_id`, `rating`, `comment`)
  and extend `product_sales` (or a new view) to show each product's average
  rating alongside its units sold.
- Rewrite query 4 (above-average spenders) using a `CASE` expression instead
  of a second nested subquery, bucketing customers into `'above average'` /
  `'below average'` labels in a single pass (see
  [Level 2 · CASE Expressions](05-case-expressions.md)).
- Write a query using `UNION ALL` that produces a single combined feed of
  "new customer" events and "order placed" events, each tagged with an
  `event_type` column and sorted by date (see
  [Level 2 · Set Operations](03-set-operations.md)).
- Add a `discount_pct` column to `order_items` with a `CHECK (discount_pct
  BETWEEN 0 AND 100)` constraint, and update the revenue queries to account
  for it.

Completing this project means you're ready for **Level 3 · Advanced**.
