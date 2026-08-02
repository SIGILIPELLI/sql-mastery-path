# 02 · Subqueries

A subquery is a `SELECT` nested inside another query — used to compute a
value, produce a list to filter against, or check whether related rows
exist. They let you express "compared to the average," "any customer who
has," or "the latest one for each row" without a separate script.

## Sample schema

```sql
CREATE TABLE customers (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE orders (
    id INTEGER PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    amount REAL NOT NULL,
    order_date TEXT NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);

INSERT INTO customers (name) VALUES ('Alice'), ('Bob'), ('Carol'), ('Dave');

INSERT INTO orders (customer_id, amount, order_date) VALUES
    (1, 120.00, '2026-01-05'),
    (1, 45.50, '2026-02-14'),
    (2, 300.00, '2026-01-20'),
    (3, 75.25, '2026-03-01'),
    (3, 60.00, '2026-03-10');
```

## Scalar subqueries — a single value

A scalar subquery returns exactly one row and one column, so it can be used
anywhere a single value is expected — like the right-hand side of a
comparison:

```sql
SELECT id, amount
FROM orders
WHERE amount > (SELECT AVG(amount) FROM orders);
```

```text
id  amount
--  ------
3   300.0
```

The inner query runs once, computes the average (`120.15`), and the outer
query compares every order's `amount` against that single number. If the
subquery ever returned more than one row, SQLite would raise an error — use
scalar subqueries only when the result is guaranteed to be one value.

## IN subqueries — filtering against a list

```sql
SELECT name
FROM customers
WHERE id IN (SELECT customer_id FROM orders);
```

```text
name
-----
Alice
Bob
Carol
```

The inner query returns a list of `customer_id` values (with duplicates,
which `IN` doesn't care about); the outer query keeps any customer whose `id`
appears in that list. Dave has never ordered, so he's excluded. This is
equivalent to an `INNER JOIN` between `customers` and a de-duplicated list of
ordering customers, but reads more directly as "customers who have an
order."

## Correlated subqueries — one per outer row

A **correlated** subquery references a column from the outer query, so it
can't be evaluated once up front — it runs again for every row the outer
query considers:

```sql
SELECT c.name,
       (SELECT o.amount
        FROM orders AS o
        WHERE o.customer_id = c.id
        ORDER BY o.order_date DESC
        LIMIT 1) AS latest_order_amount
FROM customers AS c;
```

```text
name   latest_order_amount
-----  --------------------
Alice  45.5
Bob    300.0
Carol  60.0
Dave   NULL
```

For each customer row (`c`), the subquery re-runs with `c.id` plugged in,
finding that specific customer's most recent order. Dave gets `NULL` because
his subquery finds no matching rows at all — a scalar subquery with zero
results evaluates to `NULL`, not an error.

**Performance trap:** a correlated subquery executes once *per outer row*. On
a small table like this it's invisible, but against a million-row `orders`
table, a correlated subquery inside a `SELECT` list can mean a million
mini-queries. Where possible, prefer a `JOIN` combined with `GROUP BY`, or a
window function (a topic for a later level), over a correlated subquery in
performance-sensitive queries.

## EXISTS — checking for a match without caring about values

```sql
SELECT c.name
FROM customers AS c
WHERE EXISTS (
    SELECT 1 FROM orders AS o WHERE o.customer_id = c.id AND o.amount > 100
);
```

```text
name
-----
Alice
Bob
```

`EXISTS` is also correlated (`o.customer_id = c.id` references the outer
row), but it only checks whether the inner query returns *any* row at all —
it doesn't care what columns you select inside it (`SELECT 1` is a common
convention, since the value is discarded). Query planners can often stop as
soon as they find one matching row, which makes `EXISTS` a reliably efficient
choice for "does at least one related row match" checks.

## The NOT IN + NULL trap

This is one of the sharpest edges in SQL. Suppose you track customer IDs
that returned an order, and one entry is `NULL` (an unresolved case):

```sql
CREATE TABLE returned_customer_ids (customer_id INTEGER);
INSERT INTO returned_customer_ids VALUES (2), (NULL);

SELECT name FROM customers WHERE id NOT IN (SELECT customer_id FROM returned_customer_ids);
```

```text
(0 rows)
```

Zero rows — for **every** customer, including ones clearly not `2`. Here's
why: `NOT IN (2, NULL)` expands conceptually to `id != 2 AND id != NULL`. As
covered in [Level 1 · Working with NULL](../level-1/07-working-with-null.md),
`id != NULL` is never `true` — it's always unknown — so the `AND` can never
be `true` either, no matter what `id` is. **A single `NULL` in a `NOT IN`
list silently poisons the entire query.**

The safe alternative is `NOT EXISTS`, which doesn't have this problem because
it compares rows, not raw values against a list:

```sql
SELECT name FROM customers AS c
WHERE NOT EXISTS (SELECT 1 FROM returned_customer_ids AS r WHERE r.customer_id = c.id);
```

```text
name
-----
Alice
Carol
Dave
```

## Cheat sheet

| Subquery type | Returns | Typical use |
|---------------|---------|--------------|
| Scalar | One row, one column | Comparisons (`> (SELECT ...)`) |
| `IN (...)` | A list of values | Membership filtering |
| Correlated | Re-evaluated per outer row | Per-row lookups (latest, running total) |
| `EXISTS` | True/false per outer row | "At least one related row" checks — safe with `NULL` |
| `NOT IN` with a nullable list | — | **Avoid** — a `NULL` in the list breaks the whole query; use `NOT EXISTS` instead |

## Exercise

Using the `customers`/`orders` schema above:

1. Write a scalar subquery that finds customers whose *total* spend (sum of
   their `amount`s) exceeds the average total spend per customer.
2. Write a correlated subquery that returns, for each customer, the number
   of orders they've placed (hint: `COUNT(*)` inside a correlated subquery
   in the `SELECT` list).
3. Rewrite question 2 using a `LEFT JOIN` and `GROUP BY` instead, and confirm
   both approaches give the same counts.
