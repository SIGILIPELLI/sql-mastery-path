# 07 · Indexes Basics

An index is a separate, sorted data structure that lets the database find
rows matching a condition without checking every row in the table — similar
to a book's index letting you jump straight to a page instead of reading
cover to cover. Indexes are the single biggest lever for query performance,
and also the easiest thing to misuse.

## Sample schema

```sql
CREATE TABLE customers (
    id INTEGER PRIMARY KEY,
    email TEXT NOT NULL,
    country TEXT NOT NULL,
    signup_date TEXT NOT NULL
);

-- seed 5,000 rows so the query planner has enough data to make an
-- interesting decision (generate_series is a SQLite convenience for demos,
-- not standard SQL — a real dataset would come from INSERT or an import)
INSERT INTO customers (email, country, signup_date)
SELECT
    'user' || value || '@example.com',
    CASE (value % 5)
        WHEN 0 THEN 'US' WHEN 1 THEN 'UK' WHEN 2 THEN 'IN' WHEN 3 THEN 'DE' ELSE 'FR'
    END,
    DATE('2025-01-01', '+' || value || ' days')
FROM generate_series(1, 5000);
```

## Seeing the query planner's choice with EXPLAIN QUERY PLAN

Before creating any index, look up one customer by email:

```sql
EXPLAIN QUERY PLAN
SELECT * FROM customers WHERE email = 'user2500@example.com';
```

```text
QUERY PLAN
`--SCAN customers
```

`SCAN` means SQLite checked every row in the table, one at a time, until it
found (or ruled out) a match — on 5,000 rows that's cheap, but on 5 million
it would be very slow.

## Creating an index

```sql
CREATE INDEX idx_customers_email ON customers(email);

EXPLAIN QUERY PLAN
SELECT * FROM customers WHERE email = 'user2500@example.com';
```

```text
QUERY PLAN
`--SEARCH customers USING INDEX idx_customers_email (email=?)
```

Now the plan says `SEARCH ... USING INDEX` — SQLite uses the index's sorted
structure to jump almost directly to the matching row, instead of scanning
the whole table. The naming convention `idx_<table>_<column>` isn't required
syntax, just a common, readable habit.

## Composite indexes — column order matters

An index can span multiple columns. The columns' *order* in the index
determines what it's useful for:

```sql
CREATE INDEX idx_customers_country_signup ON customers(country, signup_date);

EXPLAIN QUERY PLAN
SELECT * FROM customers WHERE country = 'US' AND signup_date > '2025-06-01';
```

```text
QUERY PLAN
`--SEARCH customers USING INDEX idx_customers_country_signup (country=? AND signup_date>?)
```

This index works well for the query above because it filters `country`
first (an equality check) and `signup_date` second (a range check) — exactly
the order the index was built in. A composite index on `(country,
signup_date)` can still help a query that filters on `country` alone, but it
is **not** usable for a query that filters on `signup_date` alone, since the
index is sorted by `country` first — think of a phone book sorted by
last-name-then-first-name: useless for finding everyone born in March.

## When an index is silently ignored

Two common patterns defeat an index even when one exists on the exact
column being filtered:

```sql
-- wrapping the column in a function prevents index use
EXPLAIN QUERY PLAN
SELECT * FROM customers WHERE LOWER(email) = 'user2500@example.com';
```

```text
QUERY PLAN
`--SCAN customers
```

```sql
-- a leading wildcard means the engine can't use the index's sort order
EXPLAIN QUERY PLAN
SELECT * FROM customers WHERE email LIKE '%2500%';
```

```text
QUERY PLAN
`--SCAN customers
```

`LOWER(email)` computes a new value for every row before comparing, so the
index — which is sorted by the *raw* `email` value — can't help; the fix is
either to store a pre-normalized column or create an expression index
(`CREATE INDEX ... ON customers(LOWER(email))`, supported in SQLite and
PostgreSQL). A leading `%` in `LIKE` means "match anything before this," so
the engine can't binary-search the sorted index — it has to check every row.
`email LIKE 'user25%'` (no leading `%`) *can* use the index, since it still
constrains the start of the sorted value.

## UNIQUE indexes

```sql
CREATE UNIQUE INDEX idx_customers_email_unique ON customers(email);

INSERT INTO customers (email, country, signup_date)
VALUES ('user2500@example.com', 'US', '2026-01-01');
```

```text
Error: UNIQUE constraint failed: customers.email
```

A `UNIQUE` index does double duty: it speeds up lookups on that column *and*
enforces that no two rows share the same value — the same mechanism behind
the `UNIQUE` column constraint from
[Level 1 · Creating Tables](../level-1/08-creating-tables.md). In fact, in
most databases, adding a `UNIQUE` or `PRIMARY KEY` constraint automatically
creates a backing index for you.

## The tradeoff: indexes aren't free

Indexes speed up reads but slow down writes: every `INSERT`, `UPDATE`, or
`DELETE` on an indexed column has to update the index's structure too, and
each index consumes additional disk space. A table with ten indexes on it
pays that cost ten times over on every write. The practical rule: index
columns you frequently filter, join, or sort on — especially ones with high
*selectivity* (many distinct values, like `email`) — and avoid indexing
columns that rarely appear in `WHERE`/`JOIN`/`ORDER BY`, or that have very
few distinct values (like a boolean flag on its own), where a full scan is
often just as fast as using the index anyway.

## Cheat sheet

| Situation | Index helps? |
|-----------|---------------|
| `WHERE col = value` on an indexed column | Yes — `SEARCH` instead of `SCAN` |
| `WHERE func(col) = value` | No — the function hides the raw value from the index |
| `WHERE col LIKE 'prefix%'` | Yes — prefix search fits the sorted index |
| `WHERE col LIKE '%suffix'` | No — leading wildcard can't use a sorted index |
| Composite index `(a, b)`, filtering on `a` only | Yes |
| Composite index `(a, b)`, filtering on `b` only | No — wrong leading column |
| Heavy `INSERT`/`UPDATE` workload, low-selectivity column | Often not worth it |

## Exercise

Using the `customers` table above:

1. Run `EXPLAIN QUERY PLAN` on a query filtering by `country` alone before
   and after creating an index on `country`, and compare the plans.
2. Create a composite index on `(signup_date, country)` — the reverse order
   of the earlier example — and check with `EXPLAIN QUERY PLAN` whether a
   query filtering on `country` alone can still use it.
3. Create an expression index on `LOWER(email)` and confirm with `EXPLAIN
   QUERY PLAN` that `WHERE LOWER(email) = '...'` now uses `SEARCH` instead
   of `SCAN`.
