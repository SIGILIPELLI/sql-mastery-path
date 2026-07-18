# 02 · SELECT Basics & Data Types

`SELECT` retrieves data. Almost everything you do in SQL revolves around
shaping and refining a `SELECT` statement, so getting comfortable with its
basic anatomy now pays off for the rest of the course.

We'll use this `books` table for every example below:

```sql
CREATE TABLE books (
    id INTEGER PRIMARY KEY,
    title TEXT NOT NULL,
    author TEXT NOT NULL,
    price REAL,
    published_year INTEGER
);

INSERT INTO books (title, author, price, published_year) VALUES
    ('Dune', 'Frank Herbert', 9.99, 1965),
    ('Foundation', 'Isaac Asimov', 8.50, 1951),
    ('Neuromancer', 'William Gibson', 7.25, 1984),
    ('The Left Hand of Darkness', 'Ursula K. Le Guin', 8.99, 1969),
    ('Snow Crash', 'Neal Stephenson', 10.50, 1992);
```

## Selecting specific columns

```sql
SELECT title, author FROM books;
```

```text
title                       author
--------------------------  ------------------
Dune                        Frank Herbert
Foundation                  Isaac Asimov
Neuromancer                 William Gibson
The Left Hand of Darkness   Ursula K. Le Guin
Snow Crash                  Neal Stephenson
```

Listing exact columns (instead of `SELECT *`) is good practice in real
applications: it's explicit about what the query needs, and it doesn't break
if someone adds a column to the table later.

## SELECT * — all columns

```sql
SELECT * FROM books;
```

Fine for quick exploration in the shell; avoid it in application code, reports,
or views, where an unexpected new column can silently change behavior
downstream.

## Column aliases with AS

```sql
SELECT title AS book_title, author AS written_by
FROM books;
```

```text
book_title                  written_by
---------------------------  ------------------
Dune                         Frank Herbert
...
```

`AS` is optional (`SELECT title book_title` works too) but including it makes
queries easier to read. Aliases are essential once you introduce expressions,
since the raw expression makes an ugly column header:

```sql
SELECT title, price * 1.08 AS price_with_tax
FROM books;
```

## Expressions in SELECT

You can compute values, not just read stored columns:

```sql
SELECT title,
       price,
       price * 0.9 AS discounted_price,
       published_year,
       2024 - published_year AS years_old
FROM books;
```

```text
title        price  discounted_price  published_year  years_old
-----------  -----  ----------------  --------------  ---------
Dune         9.99   8.991             1965            59
Foundation   8.5    7.65              1951            73
...
```

String concatenation uses `||` in SQLite/Postgres (MySQL uses `CONCAT()`
instead):

```sql
SELECT title || ' by ' || author AS citation
FROM books;
```

```text
citation
-------------------------------------
Dune by Frank Herbert
Foundation by Isaac Asimov
...
```

## DISTINCT — removing duplicates

```sql
SELECT DISTINCT author FROM books;
```

`DISTINCT` applies to the whole selected row, not just one column — so
`SELECT DISTINCT author, published_year` removes only rows that are duplicates
across *both* columns together.

## Data types in SQLite

SQLite uses **type affinity** rather than the strict, fixed column types you
may know from other databases. A column is given a *preferred* affinity, but
SQLite will still store whatever type of value you insert. The five storage
classes:

| Storage class | Example | Roughly equivalent to |
|---------------|---------|------------------------|
| `NULL`        | `NULL`  | Absence of a value |
| `INTEGER`     | `42`    | int, bigint |
| `REAL`        | `9.99`  | float, double |
| `TEXT`        | `'Dune'`| varchar, char, string |
| `BLOB`        | binary data | bytea, binary |

```sql
CREATE TABLE demo (a INTEGER, b TEXT, c REAL, d BLOB, e NUMERIC);

INSERT INTO demo VALUES (1, 'hello', 3.14, x'0102', 'still works');
SELECT typeof(a), typeof(b), typeof(c), typeof(d), typeof(e) FROM demo;
```

```text
typeof(a)  typeof(b)  typeof(c)  typeof(d)  typeof(e)
---------  ---------  ---------  ---------  ---------
integer    text       real       blob       text
```

**This flexibility is SQLite-specific.** Postgres, MySQL, and SQL Server
enforce column types strictly — inserting text into an `INTEGER` column
raises an error there, whereas SQLite will accept it under most affinities.
Don't rely on this leniency in code meant to be portable; declare types as if
they were enforced, and validate data in your application.

## CAST — explicit conversion

```sql
SELECT CAST('42' AS INTEGER) AS as_int,
       CAST(42 AS TEXT) AS as_text,
       CAST(9.99 AS INTEGER) AS truncated;
```

```text
as_int  as_text  truncated
------  -------  ---------
42      42       9
```

`CAST` is portable across nearly all SQL databases and is the safest way to
convert types explicitly rather than relying on implicit coercion.

## Exercise

Using the `books` table above:

1. Select `title` and `price`, aliasing `price` as `cost`.
2. Write a query that returns `title` and a computed column `price_in_cents`
   (price multiplied by 100, cast to `INTEGER`).
3. Select the distinct list of `author` values.
4. Write a query producing a single readable string per row like
   `"Dune (1965) — $9.99"` using `||` concatenation.
