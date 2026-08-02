# 08 · Constraints

Constraints are rules the database enforces on your behalf — they reject bad
data at insert/update time instead of letting it slip in and cause bugs
later. [Level 1 · Creating Tables](../level-1/08-creating-tables.md) briefly
introduced `NOT NULL`, `DEFAULT`, and `UNIQUE`. This module goes deeper on
foreign keys, `CHECK`, composite keys, and a SQLite-specific trap that catches
almost everyone.

## Foreign keys are OFF by default in SQLite

This is the biggest surprise for anyone coming from Postgres or MySQL: SQLite
parses and stores `FOREIGN KEY` clauses, but does **not enforce** them unless
you explicitly turn enforcement on for the connection.

```sql
CREATE TABLE authors (id INTEGER PRIMARY KEY, name TEXT NOT NULL);
CREATE TABLE books (
    id INTEGER PRIMARY KEY,
    title TEXT NOT NULL,
    author_id INTEGER,
    FOREIGN KEY (author_id) REFERENCES authors(id)
);

INSERT INTO authors (name) VALUES ('Frank Herbert');
INSERT INTO books (title, author_id) VALUES ('Ghost Book', 999);   -- author 999 doesn't exist

SELECT * FROM books;
```

```text
id  title       author_id
--  ----------  ---------
1   Ghost Book  999
```

No error — a book referencing a nonexistent author was inserted without
complaint. Fix it by enabling enforcement at the start of every connection:

```sql
PRAGMA foreign_keys = ON;

INSERT INTO books (title, author_id) VALUES ('Another Ghost', 999);
```

```text
Error: FOREIGN KEY constraint failed
```

`PRAGMA foreign_keys = ON` is not persisted in the database file — it's a
per-connection setting that most SQLite client libraries let you set on
every connect. **Always turn it on explicitly**; relying on the default
silently disables one of the most important data-integrity checks in a
relational database.

## ON DELETE actions

A foreign key can specify what happens to child rows when the referenced
parent row is deleted:

```sql
CREATE TABLE authors2 (id INTEGER PRIMARY KEY, name TEXT NOT NULL);
CREATE TABLE books2 (
    id INTEGER PRIMARY KEY,
    title TEXT NOT NULL,
    author_id INTEGER,
    FOREIGN KEY (author_id) REFERENCES authors2(id) ON DELETE CASCADE
);

INSERT INTO authors2 (name) VALUES ('Isaac Asimov');
INSERT INTO books2 (title, author_id) VALUES ('Foundation', 1), ('I, Robot', 1);

DELETE FROM authors2 WHERE id = 1;
SELECT COUNT(*) FROM books2;
```

```text
COUNT(*)
--------
0
```

`ON DELETE CASCADE` deleted both of Asimov's books automatically when his
author row was removed. The alternative, `ON DELETE SET NULL`, keeps the
child rows but clears the reference instead:

```sql
CREATE TABLE authors3 (id INTEGER PRIMARY KEY, name TEXT NOT NULL);
CREATE TABLE books3 (
    id INTEGER PRIMARY KEY,
    title TEXT NOT NULL,
    author_id INTEGER,
    FOREIGN KEY (author_id) REFERENCES authors3(id) ON DELETE SET NULL
);

INSERT INTO authors3 (name) VALUES ('Ursula K. Le Guin');
INSERT INTO books3 (title, author_id) VALUES ('The Dispossessed', 1);

DELETE FROM authors3 WHERE id = 1;
SELECT * FROM books3;
```

```text
id  title             author_id
--  ----------------  ---------
1   The Dispossessed  NULL
```

Without either clause, the default behavior (`ON DELETE NO ACTION` /
`RESTRICT`, depending on the engine) simply **rejects** the `DELETE` if
matching child rows exist — often the safest default for production data you
don't want silently cascaded away.

## CHECK constraints — arbitrary validation rules

```sql
CREATE TABLE products (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    price REAL NOT NULL CHECK (price > 0),
    discount_pct INTEGER NOT NULL DEFAULT 0 CHECK (discount_pct BETWEEN 0 AND 100)
);

INSERT INTO products (name, price) VALUES ('Widget', 9.99);   -- fine
INSERT INTO products (name, price) VALUES ('Bad Widget', -5);
```

```text
Error: CHECK constraint failed: price > 0
```

`CHECK` accepts any boolean expression — comparisons, `BETWEEN`, even
multi-column expressions referencing other columns in the same row (`CHECK
(discount_pct = 0 OR price > 10)`, for instance). It's the general-purpose
tool for business rules that don't fit `NOT NULL`, `UNIQUE`, or a foreign
key.

## Composite primary keys

A primary key can span more than one column, useful for join/junction tables
where no single column is naturally unique:

```sql
CREATE TABLE enrollments (
    student_id INTEGER NOT NULL,
    course_id INTEGER NOT NULL,
    enrolled_on TEXT NOT NULL,
    PRIMARY KEY (student_id, course_id)
);

INSERT INTO enrollments VALUES (1, 101, '2026-01-10');
INSERT INTO enrollments VALUES (1, 101, '2026-02-01');
```

```text
Error: UNIQUE constraint failed: enrollments.student_id, enrollments.course_id
```

The *combination* of `student_id` and `course_id` must be unique — student 1
can enroll in many courses, and course 101 can have many students, but
student 1 can't enroll in course 101 twice. Neither column alone is `UNIQUE`.

## Cheat sheet

| Constraint | Enforces | SQLite gotcha |
|------------|----------|-----------------|
| `PRIMARY KEY` | Unique, non-`NULL` row identifier (single or composite) | Composite form: `PRIMARY KEY (col_a, col_b)` |
| `FOREIGN KEY ... REFERENCES` | Referenced row must exist | **Not enforced unless `PRAGMA foreign_keys = ON`** |
| `ON DELETE CASCADE` | Delete children when parent is deleted | Easy to lose data unintentionally — use deliberately |
| `ON DELETE SET NULL` | Clear the reference when parent is deleted | Column must be nullable |
| `UNIQUE` | No duplicate values in a column (or column combination) | |
| `CHECK (expr)` | Any custom boolean rule | Can reference multiple columns in the same row |
| `NOT NULL` | Column must always have a value | |

## Exercise

Using the schemas above:

1. Recreate the `authors`/`books` tables with `PRAGMA foreign_keys = ON` set
   from the start, and confirm that inserting a book with a bad `author_id`
   fails immediately.
2. Add a `CHECK` constraint to `enrollments` ensuring `enrolled_on` isn't in
   the future compared to a fixed reference date (e.g. `CHECK (enrolled_on
   <= '2026-12-31')`).
3. Create a `order_items` table with a composite primary key on
   `(order_id, product_id)`, a `FOREIGN KEY` on each column, and a `CHECK`
   ensuring `quantity > 0`.
