# 04 · Sorting & Limiting Results

`ORDER BY` controls the order rows come back in; `LIMIT` (and `OFFSET`)
control how many. Without `ORDER BY`, SQL makes **no guarantee** about row
order — a database is free to return rows in whatever order is convenient
internally, and that order can change between runs.

We'll keep using the `books` table from [Module 3](03-filtering-where.md).

## ORDER BY — ascending and descending

```sql
SELECT title, price FROM books ORDER BY price;          -- ascending (default)
SELECT title, price FROM books ORDER BY price ASC;       -- explicit ascending
SELECT title, price FROM books ORDER BY price DESC;       -- descending
```

```text
title                       price
--------------------------  -----
The Hobbit                  6.99
Neuromancer                 7.25
Foundation                  8.5
The Left Hand of Darkness   8.99
Dune                        9.99
Snow Crash                  10.5
```

## Sorting by multiple columns

Ties in the first column are broken by the next column listed:

```sql
SELECT title, genre, published_year
FROM books
ORDER BY genre ASC, published_year DESC;
```

```text
title                       genre      published_year
--------------------------  ---------  --------------
Snow Crash                  Cyberpunk  1992
Neuromancer                 Cyberpunk  1984
The Hobbit                  Fantasy    1937
Dune                        Sci-Fi     1965
The Left Hand of Darkness   Sci-Fi     1969
Foundation                  Sci-Fi     1951
```

Each column can have its own direction — `ORDER BY genre ASC, price DESC` sorts
genres alphabetically, and within each genre sorts price highest-to-lowest.

## Sorting by an alias or expression

```sql
SELECT title, price * 1.08 AS price_with_tax
FROM books
ORDER BY price_with_tax DESC;

-- sorting by an expression that isn't selected also works
SELECT title FROM books
ORDER BY length(title);
```

## Sorting by column position

```sql
SELECT title, price FROM books ORDER BY 2 DESC;   -- sorts by the 2nd selected column
```

Works, but avoid it in real code — inserting a column or reordering the
`SELECT` list silently changes what `ORDER BY 2` means. Sort by name.

## NULLs in ORDER BY

SQLite sorts `NULL` values first in ascending order (before any real value)
and last in descending order. Other databases differ — Postgres defaults to
`NULLS LAST` for `ASC` and `NULLS FIRST` for `DESC`, and lets you override it
explicitly:

```sql
-- Postgres/standard SQL syntax (SQLite doesn't support NULLS FIRST/LAST directly)
SELECT title, genre FROM books ORDER BY genre NULLS LAST;
```

## LIMIT — capping the number of rows

```sql
SELECT title, price FROM books
ORDER BY price DESC
LIMIT 3;
```

```text
title         price
------------  -----
Snow Crash    10.5
Dune          9.99
The Left...   8.99
```

`LIMIT` without `ORDER BY` is close to meaningless — you'd get an arbitrary
3 rows with no guarantee of which ones. Always pair `LIMIT` with `ORDER BY`
when the specific rows matter.

## OFFSET — skipping rows (pagination)

```sql
SELECT title, price FROM books
ORDER BY price DESC
LIMIT 3 OFFSET 3;      -- skip the first 3, then take the next 3
```

This is the classic pattern behind "page 2 of results": `LIMIT page_size
OFFSET (page_number - 1) * page_size`. It's simple but has a real weakness at
scale — the database still has to walk past all the skipped rows, so `OFFSET
1000000` on a huge table is slow. Production systems commonly use
**keyset pagination** instead (e.g. `WHERE id > last_seen_id ORDER BY id LIMIT
20`), which you'll see used in later levels.

## Top-N per query — a common pattern

```sql
-- 3 cheapest books
SELECT title, price FROM books ORDER BY price ASC LIMIT 3;

-- most recently published book
SELECT title, published_year FROM books ORDER BY published_year DESC LIMIT 1;
```

## Putting it together

```sql
SELECT title, author, price
FROM books
WHERE genre = 'Sci-Fi'
ORDER BY price ASC
LIMIT 2;
```

```text
title        author         price
-----------  -------------  -----
Foundation   Isaac Asimov   8.5
The Left...  Ursula K...    8.99
```

Clause order in a query is fixed: `SELECT ... FROM ... WHERE ... ORDER BY ...
LIMIT ...` — `WHERE` always comes before `ORDER BY`, which always comes before
`LIMIT`.

## Exercise

Using the `books` table:

1. List all books ordered by `published_year` ascending (oldest first).
2. List the 3 most expensive books, most expensive first.
3. List books ordered by `genre` alphabetically, and within each genre by
   `price` descending.
4. Return "page 2" of books (3 per page) ordered by `title` alphabetically —
   i.e., books 4, 5, and 6 in that order.
