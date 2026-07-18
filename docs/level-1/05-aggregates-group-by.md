# 05 · Aggregate Functions & GROUP BY

Aggregate functions collapse many rows into a single summary value —
counting, summing, averaging. Combined with `GROUP BY`, they let you compute
those summaries **per category** instead of over the whole table.

We'll extend the `books` table from earlier modules with a couple more rows so
grouping is more interesting:

```sql
CREATE TABLE books (
    id INTEGER PRIMARY KEY,
    title TEXT NOT NULL,
    author TEXT NOT NULL,
    price REAL,
    published_year INTEGER,
    genre TEXT
);

INSERT INTO books (title, author, price, published_year, genre) VALUES
    ('Dune', 'Frank Herbert', 9.99, 1965, 'Sci-Fi'),
    ('Foundation', 'Isaac Asimov', 8.50, 1951, 'Sci-Fi'),
    ('Neuromancer', 'William Gibson', 7.25, 1984, 'Cyberpunk'),
    ('The Left Hand of Darkness', 'Ursula K. Le Guin', 8.99, 1969, 'Sci-Fi'),
    ('Snow Crash', 'Neal Stephenson', 10.50, 1992, 'Cyberpunk'),
    ('The Hobbit', 'J.R.R. Tolkien', 6.99, 1937, 'Fantasy'),
    ('The Fellowship of the Ring', 'J.R.R. Tolkien', 9.50, 1954, 'Fantasy');
```

## The five core aggregate functions

```sql
SELECT COUNT(*) AS total_books FROM books;
SELECT SUM(price) AS total_price FROM books;
SELECT AVG(price) AS average_price FROM books;
SELECT MIN(price) AS cheapest FROM books;
SELECT MAX(price) AS priciest FROM books;
```

```text
total_books
-----------
7
```

`COUNT(*)` counts rows regardless of `NULL`s. `COUNT(column)` counts only rows
where that column is **not** `NULL` — an important distinction covered more in
[Module 7](07-working-with-null.md):

```sql
SELECT COUNT(*) AS all_rows, COUNT(genre) AS rows_with_genre
FROM books;
```

## GROUP BY — one summary per group

`GROUP BY` splits rows into buckets by the value(s) of one or more columns,
then computes the aggregate separately for each bucket:

```sql
SELECT genre, COUNT(*) AS num_books
FROM books
GROUP BY genre;
```

```text
genre      num_books
---------  ---------
Cyberpunk  2
Fantasy    2
Sci-Fi     3
```

```sql
SELECT genre, AVG(price) AS avg_price, MIN(price) AS min_price, MAX(price) AS max_price
FROM books
GROUP BY genre;
```

```text
genre      avg_price  min_price  max_price
---------  ---------  ---------  ---------
Cyberpunk  8.875      7.25       10.5
Fantasy    8.245      6.99       9.5
Sci-Fi     9.16       8.5        9.99
```

**Rule:** every column in `SELECT` that isn't wrapped in an aggregate function
must appear in `GROUP BY`. `SELECT genre, title, AVG(price) ... GROUP BY
genre` is invalid — SQL wouldn't know which `title` to show for a genre with
multiple books. SQLite is lenient about enforcing this (it'll pick an
arbitrary row), but Postgres and MySQL (in strict mode) reject it outright —
treat it as an error everywhere.

## Grouping by multiple columns

```sql
SELECT genre, published_year, COUNT(*) AS num_books
FROM books
GROUP BY genre, published_year;
```

Each unique *combination* of `genre` and `published_year` becomes its own
group.

## HAVING — filtering groups, not rows

`WHERE` filters individual rows *before* grouping happens. `HAVING` filters
*groups* after aggregation — you need it because aggregate functions like
`COUNT(*)` don't exist yet at the point `WHERE` is evaluated.

```sql
-- WRONG: WHERE can't reference an aggregate
-- SELECT genre, COUNT(*) FROM books WHERE COUNT(*) > 2 GROUP BY genre;

-- RIGHT: HAVING filters after grouping
SELECT genre, COUNT(*) AS num_books
FROM books
GROUP BY genre
HAVING COUNT(*) >= 2;
```

```text
genre      num_books
---------  ---------
Cyberpunk  2
Fantasy    2
Sci-Fi     3
```

You can combine `WHERE` and `HAVING` in one query — `WHERE` trims rows first,
then grouping happens on what's left, then `HAVING` trims groups:

```sql
SELECT genre, AVG(price) AS avg_price
FROM books
WHERE published_year >= 1950
GROUP BY genre
HAVING AVG(price) > 8.00;
```

## Clause order (and evaluation order)

The clauses must be written in this order:

```text
SELECT ...
FROM ...
WHERE ...
GROUP BY ...
HAVING ...
ORDER BY ...
LIMIT ...
```

But SQL *evaluates* them in a different logical order: `FROM` → `WHERE` →
`GROUP BY` → `HAVING` → `SELECT` → `ORDER BY` → `LIMIT`. This is why `WHERE`
can't see aggregate results (they're computed after it runs) and why
`ORDER BY` *can* reference a `SELECT` alias (it runs after `SELECT`).

## A full example

```sql
SELECT genre, COUNT(*) AS num_books, ROUND(AVG(price), 2) AS avg_price
FROM books
WHERE published_year >= 1950
GROUP BY genre
HAVING COUNT(*) > 1
ORDER BY avg_price DESC;
```

```text
genre      num_books  avg_price
---------  ---------  ---------
Sci-Fi     3          9.16
Cyberpunk  2          8.88
```

(`Fantasy`'s `The Hobbit` was published in 1937, so `WHERE published_year >=
1950` drops it, leaving only `The Fellowship of the Ring` — one row, filtered
out by `HAVING COUNT(*) > 1`.)

## Exercise

Using the `books` table:

1. Count how many books exist per `genre`.
2. Find the average price per `genre`, rounded to 2 decimal places.
3. Find genres that have more than one book, showing genre and count.
4. Find the earliest (`MIN`) and latest (`MAX`) `published_year` per genre.
5. Find genres whose average price is above $8.50, ordered by average price
   descending.
