# 03 · Filtering with WHERE

`WHERE` narrows a result set down to the rows that match a condition. It's
evaluated per row, before any grouping or sorting happens.

We'll reuse the `books` table from [Module 2](02-select-basics-data-types.md):

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
    ('The Hobbit', 'J.R.R. Tolkien', 6.99, 1937, 'Fantasy');
```

## Comparison operators

```sql
SELECT title, price FROM books WHERE price > 8.50;
SELECT title, price FROM books WHERE price >= 8.50;
SELECT title FROM books WHERE published_year = 1965;
SELECT title FROM books WHERE published_year != 1965;   -- or <> 1965
```

| Operator | Meaning |
|----------|---------|
| `=`      | equal to |
| `!=` / `<>` | not equal to |
| `>`, `<` | greater / less than |
| `>=`, `<=` | greater-or-equal / less-or-equal |

## Combining conditions — AND, OR, NOT

```sql
-- AND: both conditions must be true
SELECT title FROM books
WHERE genre = 'Sci-Fi' AND price < 9.00;

-- OR: at least one condition must be true
SELECT title FROM books
WHERE genre = 'Fantasy' OR genre = 'Cyberpunk';

-- NOT: negates a condition
SELECT title FROM books
WHERE NOT genre = 'Sci-Fi';
```

`AND` binds tighter than `OR`, exactly like `*` binds tighter than `+` in
arithmetic — use parentheses whenever you mix them, since relying on
precedence rules is a common source of bugs:

```sql
-- Ambiguous-looking without parens; means: (genre = 'Sci-Fi') AND (price < 8 OR price > 10)
SELECT title FROM books
WHERE genre = 'Sci-Fi' AND (price < 8 OR price > 10);

-- vs a totally different meaning if you drop the parens:
SELECT title FROM books
WHERE genre = 'Sci-Fi' AND price < 8 OR price > 10;
```

## BETWEEN — inclusive range

```sql
SELECT title, published_year FROM books
WHERE published_year BETWEEN 1960 AND 1990;
```

Equivalent to `published_year >= 1960 AND published_year <= 1990` — both
endpoints are included.

## IN — matching a set of values

```sql
SELECT title, genre FROM books
WHERE genre IN ('Fantasy', 'Cyberpunk');

-- equivalent to, but shorter than:
SELECT title, genre FROM books
WHERE genre = 'Fantasy' OR genre = 'Cyberpunk';
```

`NOT IN` negates it:

```sql
SELECT title FROM books
WHERE genre NOT IN ('Fantasy', 'Cyberpunk');
```

## LIKE — pattern matching

`LIKE` matches text against a pattern using two wildcards:

- `%` matches any sequence of characters (including zero characters)
- `_` matches exactly one character

```sql
SELECT title FROM books WHERE title LIKE 'The%';        -- starts with "The"
SELECT title FROM books WHERE title LIKE '%Crash';       -- ends with "Crash"
SELECT title FROM books WHERE title LIKE '%Dune%';       -- contains "Dune"
SELECT title FROM books WHERE author LIKE '_ea%';        -- 2nd char is 'e', 3rd is 'a'
```

`LIKE` is case-insensitive for ASCII in SQLite by default, but this varies —
Postgres's `LIKE` is case-sensitive (use `ILIKE` there for case-insensitive
matching). Always test the behavior of the database you're actually using.

## IS NULL / IS NOT NULL

```sql
SELECT title FROM books WHERE genre IS NULL;
SELECT title FROM books WHERE genre IS NOT NULL;
```

You cannot test for `NULL` with `= NULL` — it silently matches nothing,
because `NULL` represents "unknown," and nothing is known to equal an unknown
value. This is explored fully in [Module 7](07-working-with-null.md).

## Combining everything

```sql
SELECT title, author, price, published_year
FROM books
WHERE genre = 'Sci-Fi'
  AND published_year BETWEEN 1950 AND 1970
  AND price < 10.00;
```

```text
title        author           price  published_year
-----------  ---------------  -----  --------------
Dune         Frank Herbert    9.99   1965
Foundation   Isaac Asimov     8.5    1951
```

## Exercise

Using the `books` table above, write queries to find:

1. All books priced under $8.00.
2. All books that are either `'Sci-Fi'` or `'Fantasy'`, using `IN`.
3. All books with a title containing the word `"the"` (any case), using `LIKE`.
4. All books published between 1960 and 2000 that cost more than $7.
5. All books whose author's name starts with a letter in `('F', 'N')` — hint:
   combine `LIKE` patterns with `OR`, or use `substr()`.
