# 08 · Creating Tables

`CREATE TABLE` defines a table's structure: its columns, their types, and
basic rules about what data is allowed in them.

## Basic CREATE TABLE

```sql
CREATE TABLE books (
    id INTEGER PRIMARY KEY,
    title TEXT NOT NULL,
    author TEXT,
    price REAL,
    published_year INTEGER
);
```

Running this creates an empty table — no rows yet, just the shape.

## SQLite's core data types

| Type | Stores |
|------|--------|
| `INTEGER` | Whole numbers |
| `REAL` | Floating-point numbers |
| `TEXT` | Strings of any length |
| `BLOB` | Raw binary data |
| `NUMERIC` | Flexible numeric storage (also used for dates/booleans) |

SQLite uses "type affinity" rather than strict typing — a column declared
`TEXT` will still technically accept a number, though your application code
shouldn't rely on that. Standard SQL databases like PostgreSQL and MySQL
enforce column types strictly; treat SQLite's leniency as a convenience for
learning, not a habit to carry into production schemas on other engines.

## PRIMARY KEY

```sql
CREATE TABLE books (
    id INTEGER PRIMARY KEY,   -- auto-increments in SQLite when INTEGER PRIMARY KEY
    title TEXT NOT NULL
);

INSERT INTO books (title) VALUES ('Dune');
INSERT INTO books (title) VALUES ('Foundation');

SELECT * FROM books;
```

```text
id  title
--  ----------
1   Dune
2   Foundation
```

`INTEGER PRIMARY KEY` in SQLite is special: it becomes an alias for the
table's hidden internal row ID and auto-increments automatically — you don't
need a separate `AUTOINCREMENT` keyword for the common case (it exists for
edge cases where you need IDs to never be reused).

## NOT NULL

```sql
CREATE TABLE books (
    id INTEGER PRIMARY KEY,
    title TEXT NOT NULL,      -- every book MUST have a title
    price REAL                -- price is optional (defaults to NULL)
);

INSERT INTO books (title) VALUES ('Dune');       -- fine, price is NULL
INSERT INTO books (price) VALUES (9.99);          -- FAILS: title is required
```

```text
Error: NOT NULL constraint failed: books.title
```

## DEFAULT values

```sql
CREATE TABLE books (
    id INTEGER PRIMARY KEY,
    title TEXT NOT NULL,
    in_stock INTEGER NOT NULL DEFAULT 1,
    added_at TEXT DEFAULT (datetime('now'))
);

INSERT INTO books (title) VALUES ('Dune');
SELECT * FROM books;
```

```text
id  title  in_stock  added_at
--  -----  --------  -------------------
1   Dune   1         2026-07-18 10:00:00
```

Any column with a `DEFAULT` can be omitted entirely from an `INSERT` — SQLite
fills it in automatically.

## UNIQUE

```sql
CREATE TABLE authors (
    id INTEGER PRIMARY KEY,
    email TEXT UNIQUE
);

INSERT INTO authors (email) VALUES ('ada@example.com');
INSERT INTO authors (email) VALUES ('ada@example.com');   -- FAILS
```

```text
Error: UNIQUE constraint failed: authors.email
```

## Modifying an existing table

```sql
ALTER TABLE books ADD COLUMN genre TEXT;

-- SQLite's ALTER TABLE is limited compared to Postgres/MySQL: it can add a
-- column or rename a table/column, but cannot drop a column or change a
-- column's type without recreating the table (a common workaround: create a
-- new table with the desired shape, copy data with INSERT ... SELECT, drop
-- the old table, rename the new one).
```

## Dropping a table

```sql
DROP TABLE IF EXISTS temp_import;   -- IF EXISTS avoids an error if it's already gone
```

## Cheat sheet

| Clause | Purpose |
|--------|---------|
| `PRIMARY KEY` | Uniquely identifies each row (auto-increments if `INTEGER`) |
| `NOT NULL` | Column must always have a value |
| `UNIQUE` | No two rows may share the same value in this column |
| `DEFAULT value` | Value used automatically when not specified on insert |
| `ALTER TABLE ... ADD COLUMN` | Add a new column to an existing table |
| `DROP TABLE IF EXISTS` | Delete a table, without erroring if it's already gone |

## Exercise

Create a `students` table with: an auto-incrementing `id`, a required `name`,
a `unique` `email`, an `enrolled` integer column defaulting to `1`, and an
optional `graduation_year`. Insert two students (omitting `enrolled` for one
of them to confirm the default kicks in), then try inserting a third student
with a duplicate email and confirm it fails.
