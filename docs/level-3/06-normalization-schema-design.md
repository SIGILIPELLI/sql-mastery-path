# 06 · Normalization & Schema Design

Normalization is the process of structuring tables so each fact is stored
in exactly one place. The payoff isn't academic — it's that updates,
inserts, and deletes stop being able to silently corrupt your data. This
module shows a real anomaly first, then the normalized fix.

## The problem: one flat table

```sql
CREATE TABLE orders_unnormalized (
    id INTEGER PRIMARY KEY,
    customer_name TEXT,
    customer_email TEXT,
    product_name TEXT,
    product_price REAL,
    quantity INTEGER
);

INSERT INTO orders_unnormalized VALUES
    (1, 'Nadia Farouk', 'nadia@example.com', 'Widget', 9.99, 3),
    (2, 'Nadia Farouk', 'nadia@example.com', 'Gadget', 19.99, 1),
    (3, 'Omar Rossi',   'omar@example.com',  'Widget', 9.99, 2);

SELECT * FROM orders_unnormalized;
```

```text
id  customer_name  customer_email     product_name  product_price  quantity
--  -------------  -----------------  ------------  -------------  --------
1   Nadia Farouk   nadia@example.com  Widget        9.99           3
2   Nadia Farouk   nadia@example.com  Gadget        19.99          1
3   Omar Rossi     omar@example.com   Widget        9.99           2
```

Nadia's name, email, and Widget's price are each duplicated across rows.

## The update anomaly, demonstrated

```sql
UPDATE orders_unnormalized SET customer_email = 'nadia.f@example.com' WHERE id = 1;

SELECT id, customer_name, customer_email FROM orders_unnormalized;
```

```text
id  customer_name  customer_email
--  -------------  -------------------
1   Nadia Farouk   nadia.f@example.com
2   Nadia Farouk   nadia@example.com
3   Omar Rossi     omar@example.com
```

Nadia now has two different emails on file, depending on which row you
look at — because "update Nadia's email" was really "update this one row's
copy of Nadia's email," and the `WHERE id = 1` clause only touched one of
the two rows that mention her. Rename a product and the same thing happens
to its price and name across every order row that references it. This is
the **update anomaly**: a single real-world fact needs multiple, easy-to-miss
writes to stay consistent.

## Normal forms, briefly

- **1NF** — each column holds a single atomic value (no comma-separated
  lists crammed into one field).
- **2NF** — every non-key column depends on the *whole* primary key, not
  part of it (relevant for composite keys).
- **3NF** — every non-key column depends on the key and *nothing but* the
  key — no column depends on another non-key column. `product_price`
  depending on `product_name` instead of the order is a 3NF violation: that's
  a fact about the product, not about the order.

In practice, "split repeating facts into their own table, referenced by
foreign key" gets you to 3NF for most everyday schemas.

## The fix: split into entities

```sql
CREATE TABLE customers (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT NOT NULL UNIQUE
);

CREATE TABLE products (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    price REAL NOT NULL
);

CREATE TABLE orders (
    id INTEGER PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(id)
);

CREATE TABLE order_items (
    id INTEGER PRIMARY KEY,
    order_id INTEGER NOT NULL REFERENCES orders(id),
    product_id INTEGER NOT NULL REFERENCES products(id),
    quantity INTEGER NOT NULL
);

INSERT INTO customers (name, email) VALUES
    ('Nadia Farouk', 'nadia@example.com'), ('Omar Rossi', 'omar@example.com');
INSERT INTO products (name, price) VALUES ('Widget', 9.99), ('Gadget', 19.99);
INSERT INTO orders (customer_id) VALUES (1), (1), (2);
INSERT INTO order_items (order_id, product_id, quantity) VALUES
    (1, 1, 3), (2, 2, 1), (3, 1, 2);
```

`customers` and `products` each store their facts exactly once.
`order_items` is a join between an order and a product, carrying only data
specific to *that combination* (`quantity`) — this is the standard pattern
for a many-to-many relationship (one order can have many products, one
product can appear on many orders).

## Reassembling the view with JOINs

```sql
SELECT o.id AS order_id, c.name, p.name AS product, oi.quantity, p.price
FROM orders o
JOIN customers c ON c.id = o.customer_id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id;
```

```text
order_id  name          product  quantity  price
--------  ------------  -------  --------  -----
1         Nadia Farouk  Widget   3         9.99
2         Nadia Farouk  Gadget   1         19.99
3         Omar Rossi    Widget   2         9.99
```

Same information as the flat table, produced on demand via `JOIN` instead
of stored redundantly.

## The anomaly is gone

```sql
UPDATE customers SET email = 'nadia.f@example.com' WHERE id = 1;
SELECT * FROM customers;
```

```text
id  name          email
--  ------------  -------------------
1   Nadia Farouk  nadia.f@example.com
2   Omar Rossi    omar@example.com
```

One row, one update, done — every order that joins to `customers` picks up
the new email automatically, because there's only one copy of it to update.

## When to break the rules: denormalization

Normalization optimizes for write consistency, not read speed — every
`SELECT` that needs the full picture now costs a `JOIN`. For reporting
tables, analytics snapshots, or hot read paths where the source data rarely
changes, deliberately storing a duplicated or precomputed column (a
"denormalized" column) can be the right trade, as long as you document that
you're accepting the anomaly risk in exchange for read speed.

## Cheat sheet

| Symptom in a flat table | Fix |
|---|---|
| Same customer's name/email repeated per order | Move to a `customers` table, reference by `customer_id` |
| Same product's name/price repeated per order line | Move to a `products` table, reference by `product_id` |
| One order needs many products | Junction table (`order_items`) with a foreign key to each side |
| Updating one fact requires touching many rows | Sign of a normalization gap — that fact belongs in its own table |
| Read-heavy reporting query, source data static | Deliberate denormalization can be acceptable — document the trade-off |

## Exercise

Using the `orders_unnormalized` table above:

1. Design normalized tables for a blog: `posts` have a title, body, and
   author; authors have a name and email; posts can have many tags, and a
   tag can apply to many posts. Write the `CREATE TABLE` statements.
2. Write a query that reassembles one post with its author's name and a
   comma-separated list of its tags (hint: `GROUP_CONCAT`).
3. Describe one realistic scenario where you'd deliberately denormalize —
   store a duplicate column instead of joining — and explain what anomaly
   risk you'd be accepting in exchange.
