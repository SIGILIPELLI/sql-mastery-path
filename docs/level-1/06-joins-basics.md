# 06 · Joins Basics

Real data lives in multiple related tables. Joins let you combine rows from
two (or more) tables based on a matching column — usually a foreign key
pointing back to another table's primary key.

## Sample schema

```sql
CREATE TABLE authors (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE books (
    id INTEGER PRIMARY KEY,
    title TEXT NOT NULL,
    author_id INTEGER,
    price REAL,
    FOREIGN KEY (author_id) REFERENCES authors(id)
);

INSERT INTO authors (name) VALUES
    ('Frank Herbert'), ('Isaac Asimov'), ('Ursula K. Le Guin');

INSERT INTO books (title, author_id, price) VALUES
    ('Dune', 1, 9.99),
    ('Dune Messiah', 1, 8.99),
    ('Foundation', 2, 8.50),
    ('The Left Hand of Darkness', 3, 8.99),
    ('The Lathe of Heaven', NULL, 7.50);   -- no matching author on file
```

## INNER JOIN — only matching rows

```sql
SELECT books.title, authors.name
FROM books
INNER JOIN authors ON books.author_id = authors.id;
```

```text
title                       name
--------------------------  ------------------
Dune                        Frank Herbert
Dune Messiah                Frank Herbert
Foundation                  Isaac Asimov
The Left Hand of Darkness   Ursula K. Le Guin
```

Notice `The Lathe of Heaven` is missing — its `author_id` is `NULL`, so it has
no matching row in `authors`, and `INNER JOIN` only returns rows where **both**
sides match.

## LEFT JOIN — keep everything on the left

```sql
SELECT books.title, authors.name
FROM books
LEFT JOIN authors ON books.author_id = authors.id;
```

```text
title                       name
--------------------------  ------------------
Dune                        Frank Herbert
Dune Messiah                Frank Herbert
Foundation                  Isaac Asimov
The Left Hand of Darkness   Ursula K. Le Guin
The Lathe of Heaven         NULL
```

`LEFT JOIN` keeps every row from the left table (`books`), filling in `NULL`
for any columns from the right table (`authors`) when there's no match. This
is exactly how you find "orphaned" or unmatched rows — filter for `WHERE
authors.id IS NULL` to see only books with no known author.

## Table aliases (essential once queries get longer)

```sql
SELECT b.title, a.name
FROM books AS b
INNER JOIN authors AS a ON b.author_id = a.id
WHERE b.price < 9.00;
```

Aliasing tables (`books AS b`) keeps multi-join queries readable and is
required once two tables share a column name (both having an `id`, for
instance — `b.id` vs `a.id` disambiguates).

## Joining more than two tables

```sql
CREATE TABLE genres (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL
);

ALTER TABLE books ADD COLUMN genre_id INTEGER;

INSERT INTO genres (name) VALUES ('Sci-Fi');
UPDATE books SET genre_id = 1;

SELECT b.title, a.name AS author, g.name AS genre
FROM books AS b
INNER JOIN authors AS a ON b.author_id = a.id
INNER JOIN genres AS g ON b.genre_id = g.id;
```

Chain as many joins as you need — SQLite (and every other engine) processes
them left to right, each one narrowing or extending the row set.

## Cheat sheet

| Join type | Keeps |
|-----------|-------|
| `INNER JOIN` | Only rows with a match on both sides |
| `LEFT JOIN` | All left-side rows, `NULL`-filled right side if no match |
| (SQLite 3.39+) `RIGHT JOIN` | All right-side rows, `NULL`-filled left side if no match — or just swap table order and use `LEFT JOIN`, which works everywhere |

## Exercise

Using the `books`/`authors` schema above:

1. Write an `INNER JOIN` listing every book with its author's name.
2. Write a `LEFT JOIN` that also includes books with no known author.
3. Using that `LEFT JOIN`, add a `WHERE` clause to find only the books with a
   missing author (hint: check for `NULL` on the joined column).
4. Add the `genres` table from the multi-join example and write a query
   joining all three tables to show title, author, and genre together.
