# 06 · Views

A view is a `SELECT` query saved under a name, queryable as if it were a
table. It doesn't store data itself — every time you query a view, the
underlying query runs fresh against the real tables. Views exist to hide
complexity: wrap a gnarly join or aggregate once, then let everyone (and
every later query) reference a simple name instead.

## Sample schema

```sql
CREATE TABLE products (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    category TEXT NOT NULL,
    price REAL NOT NULL,
    stock INTEGER NOT NULL
);

INSERT INTO products (name, category, price, stock) VALUES
    ('Wireless Mouse', 'Electronics', 25.00, 40),
    ('Mechanical Keyboard', 'Electronics', 75.00, 0),
    ('Desk Lamp', 'Home', 30.00, 15),
    ('Standing Desk', 'Home', 350.00, 5);

CREATE TABLE sales (
    id INTEGER PRIMARY KEY,
    product_id INTEGER NOT NULL,
    quantity INTEGER NOT NULL,
    sale_date TEXT NOT NULL,
    FOREIGN KEY (product_id) REFERENCES products(id)
);

INSERT INTO sales (product_id, quantity, sale_date) VALUES
    (1, 3, '2026-02-01'),
    (1, 2, '2026-02-10'),
    (3, 1, '2026-02-05'),
    (4, 1, '2026-02-15');
```

## Creating and querying a simple view

```sql
CREATE VIEW in_stock_products AS
SELECT id, name, category, price, stock
FROM products
WHERE stock > 0;

SELECT * FROM in_stock_products;
```

```text
id  name            category     price  stock
--  --------------  -----------  -----  -----
1   Wireless Mouse  Electronics  25.0   40
3   Desk Lamp       Home         30.0   15
4   Standing Desk   Home         350.0  5
```

`Mechanical Keyboard` is excluded because its `stock` is `0`. Anywhere you'd
use a table name — `SELECT`, `JOIN`, another view's definition — you can use
`in_stock_products` instead, and the `WHERE stock > 0` filter is applied
automatically every time.

## A view over a join and aggregate

Views are most valuable when they hide something genuinely complex:

```sql
CREATE VIEW category_revenue AS
SELECT p.category, SUM(p.price * s.quantity) AS total_revenue
FROM sales AS s
JOIN products AS p ON s.product_id = p.id
GROUP BY p.category;

SELECT * FROM category_revenue;
```

```text
category     total_revenue
-----------  -------------
Electronics  125.0
Home         380.0
```

Anyone who needs "revenue per category" can now just `SELECT * FROM
category_revenue` instead of re-deriving the join and `GROUP BY` every time —
and if the revenue calculation logic ever changes, you fix it in one place.

## Views are not snapshots

A view has no storage of its own — it re-runs its defining query against
live data on every reference:

```sql
UPDATE products SET stock = 10 WHERE name = 'Mechanical Keyboard';

SELECT * FROM in_stock_products;
```

```text
id  name                 category     price  stock
--  -------------------  -----------  -----  -----
1   Wireless Mouse       Electronics  25.0   40
2   Mechanical Keyboard  Electronics  75.0   10
3   Desk Lamp            Home         30.0   15
4   Standing Desk        Home         350.0  5
```

`Mechanical Keyboard` now appears — the view didn't need refreshing, because
it was never a stored copy. This is the key mental model: **a view is a
named query, not a cached result.** (Some databases also offer *materialized*
views, which do cache results and require an explicit refresh — SQLite does
not have these; you'd emulate one with a real table populated by a script or
trigger.)

## Views are read-only in SQLite

Unlike PostgreSQL or MySQL, which allow direct `UPDATE`/`INSERT`/`DELETE`
through simple single-table views, SQLite treats every view as strictly
read-only:

```sql
UPDATE in_stock_products SET stock = stock - 1 WHERE name = 'Desk Lamp';
```

```text
Error: cannot modify in_stock_products because it is a view
```

This fails even though `in_stock_products` is a simple, single-table view
with no aggregation — SQLite doesn't attempt to figure out which underlying
row an update should target. To make a SQLite view writable, you'd attach an
`INSTEAD OF` trigger that translates writes against the view into an
explicit `UPDATE`/`INSERT`/`DELETE` on the base table. Without one, always
write to `products` directly and let the view reflect the change on its next
read.

## Dropping a view

```sql
DROP VIEW IF EXISTS in_stock_products;
```

Dropping a view only removes the saved query definition — it never touches
the underlying table's data.

## Cheat sheet

| Fact | Detail |
|------|--------|
| Storage | None — a view is a saved query, re-run on every reference |
| Freshness | Always current; no refresh step needed |
| Writable in SQLite? | No, unless you add an `INSTEAD OF` trigger |
| Writable in Postgres/MySQL? | Simple single-table views often are, by default |
| `DROP VIEW IF EXISTS` | Removes the definition only, never the base table's rows |
| Good use case | Hiding a recurring join/aggregate behind a simple name |

## Exercise

Using the `products`/`sales` schema above:

1. Create a view `low_stock_products` showing products with `stock < 10`.
2. Create a view `product_sales_summary` that joins `products` and `sales`
   to show each product's name alongside its total quantity sold (use
   `LEFT JOIN` so products with zero sales still appear — see
   [Level 2 · Advanced Joins](01-advanced-joins.md) for the `WHERE`-vs-`ON`
   trap if you add extra filters).
3. Confirm that inserting a new row into `products` immediately shows up (or
   doesn't) in `low_stock_products` without recreating the view.
