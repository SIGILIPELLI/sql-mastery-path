# 10 · Project — Library/Bookstore Database

A small end-to-end project combining everything from Level 1: creating
tables, inserting data, filtering, sorting, joins, aggregates, and handling
`NULL`.

## What you'll build

A two-table library database — `authors` and `books` — that you'll design,
populate, and query to answer realistic questions.

## Setting up

Run `sqlite3 library.db` to open a new database file, then create the schema:

```sql
CREATE TABLE authors (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    country TEXT
);

CREATE TABLE books (
    id INTEGER PRIMARY KEY,
    title TEXT NOT NULL,
    author_id INTEGER,
    genre TEXT,
    price REAL,
    published_year INTEGER,
    in_stock INTEGER NOT NULL DEFAULT 1,
    FOREIGN KEY (author_id) REFERENCES authors(id)
);
```

## Seeding data

```sql
INSERT INTO authors (name, country) VALUES
    ('Frank Herbert', 'USA'),
    ('Isaac Asimov', 'USA'),
    ('Ursula K. Le Guin', 'USA'),
    ('Haruki Murakami', 'Japan');

INSERT INTO books (title, author_id, genre, price, published_year, in_stock) VALUES
    ('Dune', 1, 'Sci-Fi', 9.99, 1965, 1),
    ('Dune Messiah', 1, 'Sci-Fi', 8.99, 1969, 0),
    ('Foundation', 2, 'Sci-Fi', 8.50, 1951, 1),
    ('I, Robot', 2, 'Sci-Fi', 7.99, 1950, 1),
    ('The Left Hand of Darkness', 3, 'Sci-Fi', 8.99, 1969, 1),
    ('Norwegian Wood', 4, 'Literary Fiction', 10.99, 1987, 1),
    ('Kafka on the Shore', 4, 'Literary Fiction', 11.99, 2002, 0),
    ('Unknown Author Book', NULL, 'Mystery', 5.99, 2010, 1);
```

## Queries to answer real questions

**1. List every book currently in stock, cheapest first:**

```sql
SELECT title, price FROM books WHERE in_stock = 1 ORDER BY price ASC;
```

**2. List each book with its author's name (books with no known author still
appear, thanks to `LEFT JOIN`):**

```sql
SELECT b.title, a.name AS author
FROM books AS b
LEFT JOIN authors AS a ON b.author_id = a.id;
```

**3. Count books per genre:**

```sql
SELECT genre, COUNT(*) AS num_books
FROM books
GROUP BY genre
ORDER BY num_books DESC;
```

**4. Average book price per author, only for authors with 2+ books:**

```sql
SELECT a.name, COUNT(*) AS num_books, ROUND(AVG(b.price), 2) AS avg_price
FROM books AS b
INNER JOIN authors AS a ON b.author_id = a.id
GROUP BY a.name
HAVING COUNT(*) >= 2
ORDER BY avg_price DESC;
```

**5. Find books published before 1970 that are currently out of stock:**

```sql
SELECT title, published_year
FROM books
WHERE published_year < 1970 AND in_stock = 0;
```

**6. Find books with no author on file (using the `NULL` check from
Module 7):**

```sql
SELECT title FROM books WHERE author_id IS NULL;
```

## Expected results (sanity check)

Query 3 should show `Sci-Fi` with 5 books and `Literary Fiction` with 2 —
plus `Mystery` with 1 (the unknown-author book). Query 4 should only include
`Frank Herbert` and `Isaac Asimov` (each with 2+ books) — `Ursula K. Le Guin`
and `Haruki Murakami` each have exactly 2 as well, so all four authors
should actually appear; if you seeded fewer rows for one of them, recheck
your `INSERT` statements above.

## Stretch goals

- Add a `reviews` table (`book_id`, `rating`, `comment`) and write a query
  showing each book's average rating alongside its title.
- Add a `genres` lookup table (instead of a free-text `genre` column) and
  rewrite the schema and queries to join through it.
- Write a query that finds the most expensive book per genre using a
  subquery (a preview of [Level 2 · Subqueries](../level-2/02-subqueries.md)).

Completing this project means you're ready for **Level 2 · Intermediate**.
