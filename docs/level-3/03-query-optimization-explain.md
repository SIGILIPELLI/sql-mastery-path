# 03 · Query Optimization & EXPLAIN QUERY PLAN

Every query has a *plan* — the sequence of steps SQLite's engine actually
takes to produce your result: which tables it scans, which indexes it uses,
in what order it joins. `EXPLAIN QUERY PLAN` shows you that plan without
running the query, so you can tell whether SQLite is scanning a million rows
or using an index to jump straight to the ones you want.

## Sample schema

```sql
CREATE TABLE customers (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    country TEXT NOT NULL
);

CREATE TABLE orders (
    id INTEGER PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(id),
    amount REAL NOT NULL,
    placed_at TEXT NOT NULL
);

INSERT INTO customers (name, country) VALUES
    ('Amara Osei', 'GH'), ('Bo Lindqvist', 'SE'), ('Chen Wei', 'CN'),
    ('Dara Kim', 'KR'), ('Elin Vartan', 'FI');

-- 2000 orders spread randomly across the 5 customers
INSERT INTO orders (customer_id, amount, placed_at)
SELECT (abs(random()) % 5) + 1,
       (abs(random()) % 500) + 10,
       date('2025-01-01', '+' || (abs(random()) % 300) || ' days')
FROM (WITH RECURSIVE seq(x) AS (
          SELECT 1 UNION ALL SELECT x + 1 FROM seq WHERE x < 2000
      ) SELECT x FROM seq);
```

## Reading a plan: SCAN vs SEARCH

```sql
EXPLAIN QUERY PLAN
SELECT * FROM orders WHERE customer_id = 3;
```

```text
QUERY PLAN
`--SCAN orders
```

`SCAN orders` means SQLite walks every row in the table, checking the
`WHERE` clause on each one — with 2000 rows and no index on `customer_id`,
that's 2000 comparisons to find the handful that match.

## Adding an index changes SCAN to SEARCH

```sql
CREATE INDEX idx_orders_customer_id ON orders(customer_id);

EXPLAIN QUERY PLAN
SELECT * FROM orders WHERE customer_id = 3;
```

```text
QUERY PLAN
`--SEARCH orders USING INDEX idx_orders_customer_id (customer_id=?)
```

`SEARCH ... USING INDEX` means SQLite uses a B-tree lookup instead of a full
scan — it jumps directly to the matching rows via `idx_orders_customer_id`
instead of reading the whole table. This is the single most common
optimization: index the columns your `WHERE` clauses filter on.

## Joins: plan order matters

```sql
EXPLAIN QUERY PLAN
SELECT o.* FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE c.country = 'SE';
```

```text
QUERY PLAN
|--SCAN c
`--SEARCH o USING INDEX idx_orders_customer_id (customer_id=?)
```

SQLite picked `customers` (`c`) as the *outer* loop — scanning its 5 rows —
and for each matching customer, searches `orders` by the indexed
`customer_id`. That's 5 scans plus 5 indexed lookups, versus scanning all
2000 orders. The query planner chooses join order based on table sizes and
available indexes; it isn't necessarily the order you wrote the tables in.

## Indexing the filtered column too

```sql
CREATE INDEX idx_customers_country ON customers(country);

EXPLAIN QUERY PLAN
SELECT o.* FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE c.country = 'SE';
```

```text
QUERY PLAN
|--SEARCH c USING COVERING INDEX idx_customers_country (country=?)
`--SEARCH o USING INDEX idx_orders_customer_id (customer_id=?)
```

Now `customers` is searched by the new `country` index instead of scanned —
"COVERING INDEX" means the index alone (columns `country`, `id`) has enough
data to answer that step without touching the table itself, which is even
cheaper than a normal index search.

## GROUP BY can piggyback on an index

```sql
EXPLAIN QUERY PLAN
SELECT customer_id, COUNT(*) FROM orders GROUP BY customer_id;
```

```text
QUERY PLAN
`--SCAN orders USING COVERING INDEX idx_orders_customer_id
```

Because `idx_orders_customer_id` already stores rows sorted by
`customer_id`, SQLite can scan the index in order and count consecutive
runs — no separate sort step needed, and it never touches the underlying
table rows at all.

## The trap: wrapping a column breaks index use

```sql
CREATE INDEX idx_orders_amount ON orders(amount);

EXPLAIN QUERY PLAN
SELECT * FROM orders WHERE amount + 0 > 400;
```

```text
QUERY PLAN
`--SCAN orders
```

```sql
EXPLAIN QUERY PLAN
SELECT * FROM orders WHERE amount > 400;
```

```text
QUERY PLAN
`--SEARCH orders USING INDEX idx_orders_amount (amount>?)
```

Identical result set, wildly different plans. `amount + 0 > 400` computes an
expression on every row before comparing it, so SQLite can't use the index —
it falls back to a full scan. `amount > 400` compares the raw column, so the
index applies directly. This is the most common way developers accidentally
disable an index: wrapping the indexed column in a function or arithmetic
(`WHERE lower(name) = 'x'`, `WHERE date(placed_at) = '2025-01-01'`, etc.).

## ANALYZE: give the planner real statistics

```sql
ANALYZE;
SELECT * FROM sqlite_stat1;
```

```text
tbl        idx                     stat
---------  ----------------------  --------
orders     idx_orders_amount       2000 5
orders     idx_orders_customer_id  2000 400
customers  idx_customers_country   5 1
```

`ANALYZE` samples each index and records row-count statistics in
`sqlite_stat1` — here, `idx_orders_customer_id` has 2000 rows in the table
with roughly 400 rows per distinct `customer_id` value. The query planner
uses these numbers to decide whether an index search is actually cheaper
than a scan (an index on a column with only 2 distinct values across a
million rows usually isn't worth using). Without `ANALYZE`, SQLite guesses;
run it after loading real data or major changes, since stale statistics can
lead to worse plans than no statistics at all.

## Cheat sheet

| Plan output | Meaning |
|---|---|
| `SCAN table` | Full table scan — every row checked |
| `SEARCH table USING INDEX ix (col=?)` | Indexed lookup — jumps to matching rows |
| `SEARCH table USING COVERING INDEX ix` | Index alone answers the query, no table access |
| `SCAN table USING COVERING INDEX ix` | Ordered scan of the index, often replaces a sort |
| Wrapping a column (`col + 0`, `lower(col)`) | Disables index use on that column |
| `ANALYZE` | Refreshes planner statistics in `sqlite_stat1` |

## Exercise

Using the `orders`/`customers` schema above:

1. Run `EXPLAIN QUERY PLAN` on `SELECT * FROM orders WHERE placed_at > '2025-06-01'` before and after creating an index on `placed_at`. Confirm the plan switches from `SCAN` to `SEARCH`.
2. Write a query that filters on `date(placed_at) = '2025-06-15'` and explain why an index on `placed_at` won't help it, even though one exists.
3. Run `ANALYZE` and inspect `sqlite_stat1` for the index you created in step 1 — what does the row count tell you about how selective that index is?
