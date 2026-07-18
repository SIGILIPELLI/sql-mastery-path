# 07 · Working with NULL

`NULL` represents "unknown" or "absent" — it is not the same as zero, an
empty string, or false. It has special comparison rules that trip up almost
everyone the first time.

## NULL is not equal to anything, including itself

```sql
CREATE TABLE books (
    id INTEGER PRIMARY KEY,
    title TEXT NOT NULL,
    price REAL,
    discontinued_reason TEXT
);

INSERT INTO books (title, price, discontinued_reason) VALUES
    ('Dune', 9.99, NULL),
    ('Old Title', NULL, 'out of print'),
    ('Another Old One', NULL, NULL);

SELECT * FROM books WHERE price = NULL;    -- returns ZERO rows, always
SELECT * FROM books WHERE price != NULL;   -- also returns ZERO rows, always
```

Both queries above are silently wrong but produce no error — `= NULL` and `!=
NULL` never match anything because comparing "unknown" to anything, even
`NULL` itself, yields "unknown" (which SQL treats as not-true). This is one
of the most common real-world SQL bugs.

## IS NULL and IS NOT NULL — the correct check

```sql
SELECT * FROM books WHERE price IS NULL;
```

```text
id  title             price  discontinued_reason
--  ----------------  -----  --------------------
2   Old Title         NULL   out of print
3   Another Old One   NULL   NULL
```

```sql
SELECT * FROM books WHERE price IS NOT NULL;
```

```text
id  title  price  discontinued_reason
--  -----  -----  --------------------
1   Dune   9.99   NULL
```

## COALESCE — first non-NULL value

```sql
SELECT title, COALESCE(discontinued_reason, 'still in print') AS status
FROM books;
```

```text
title             status
----------------  --------------
Dune              still in print
Old Title         out of print
Another Old One   still in print
```

`COALESCE(a, b, c, ...)` returns the first argument that isn't `NULL` — the
standard way to provide a fallback/default display value.

## NULL in arithmetic and aggregates

```sql
SELECT price + 1 FROM books WHERE title = 'Old Title';   -- NULL, not 1
```

Any arithmetic expression involving `NULL` produces `NULL` — it propagates
through calculations rather than being treated as zero.

```sql
SELECT COUNT(*) AS all_rows, COUNT(price) AS rows_with_price, AVG(price) AS avg_price
FROM books;
```

```text
all_rows  rows_with_price  avg_price
--------  ---------------  ---------
3         1                9.99
```

`COUNT(*)` counts every row. `COUNT(column)` and aggregate functions like
`AVG`, `SUM`, `MIN`, `MAX` all silently **skip** `NULL` values rather than
erroring or counting them as zero — `AVG(price)` here is `9.99`, not `3.33`.

## NULL in ORDER BY

```sql
SELECT title, price FROM books ORDER BY price;
```

In SQLite, `NULL` sorts first in ascending order (before any real number) and
last in descending order. Other databases vary on this default (Postgres also
puts `NULL` last on `DESC`, but treats it as largest by default on some
configurations) — always check your specific engine's docs, or force it
explicitly:

```sql
SELECT title, price FROM books ORDER BY price IS NULL, price;
-- non-NULL prices first (in ascending order), NULLs pushed to the end
```

## NULLIF — turn a specific value into NULL

```sql
SELECT title, NULLIF(price, 0) AS price_or_null FROM books;
```

`NULLIF(a, b)` returns `NULL` if `a` equals `b`, otherwise returns `a` —
useful for turning sentinel values like `0` or `''` into a proper `NULL`
before further processing (e.g. before passing into `AVG`, so a placeholder
zero doesn't wrongly drag down the average).

## Cheat sheet

| Task | Correct approach |
|------|-------------------|
| Check for NULL | `WHERE col IS NULL` (never `= NULL`) |
| Check for NOT NULL | `WHERE col IS NOT NULL` |
| Provide a default | `COALESCE(col, 'default')` |
| Turn a sentinel into NULL | `NULLIF(col, sentinel_value)` |
| Count non-NULL values | `COUNT(col)` (not `COUNT(*)`) |
| NULLs in aggregates | Silently skipped, not treated as 0 |
| NULLs in arithmetic | Any operation with NULL produces NULL |

## Exercise

Using the `books` table above: write a query using `COALESCE` that shows each
book's price, defaulting to `0.00` when the price is `NULL`. Then write a
query that counts how many books have a `NULL` price using `IS NULL`, and a
separate query using `NULLIF` that treats a price of exactly `0` the same as
`NULL` when computing the average price.
