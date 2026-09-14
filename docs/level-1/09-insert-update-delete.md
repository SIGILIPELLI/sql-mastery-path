# 09 · Inserting, Updating, Deleting Data

## 🎥 Video walkthrough

<iframe width="100%" height="400" style="max-width:720px;aspect-ratio:16/9;height:auto;" src="https://www.youtube.com/embed/fRugrsoX--8" title="Video walkthrough" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

`SELECT` reads data — `INSERT`, `UPDATE`, and `DELETE` are how you change it.
These three are collectively called DML (Data Manipulation Language).

## INSERT — adding rows

```sql
CREATE TABLE books (
    id INTEGER PRIMARY KEY,
    title TEXT NOT NULL,
    author TEXT,
    price REAL
);

-- Single row, explicit columns
INSERT INTO books (title, author, price) VALUES ('Dune', 'Frank Herbert', 9.99);

-- Multiple rows in one statement
INSERT INTO books (title, author, price) VALUES
    ('Foundation', 'Isaac Asimov', 8.50),
    ('Neuromancer', 'William Gibson', 7.25);

SELECT * FROM books;
```

```text
id  title        author           price
--  -----------  ---------------  -----
1   Dune         Frank Herbert    9.99
2   Foundation   Isaac Asimov     8.50
3   Neuromancer  William Gibson   7.25
```

Always list column names explicitly (`INSERT INTO books (title, author,
price)`) rather than relying on column order — it's self-documenting and
won't silently break if the table's column order ever changes.

## UPDATE — modifying existing rows

```sql
UPDATE books
SET price = 11.99
WHERE title = 'Dune';
```

```sql
-- Update multiple columns at once
UPDATE books
SET price = price * 0.9, author = 'F. Herbert'
WHERE id = 1;
```

**The `WHERE` clause is not optional in practice** — `UPDATE books SET price
= 0` with no `WHERE` at all sets every single row's price to `0`. Always
write and verify the `WHERE` clause (often by running the equivalent `SELECT
... WHERE ...` first to confirm exactly which rows will be affected) before
running an `UPDATE`.

```sql
-- Verify first:
SELECT * FROM books WHERE price > 9;

-- Then update the same rows:
UPDATE books SET price = price - 1 WHERE price > 9;
```

## DELETE — removing rows

```sql
DELETE FROM books WHERE title = 'Neuromancer';
```

```sql
-- DELETE FROM books;  -- DANGER: no WHERE means every row is deleted
```

Just like `UPDATE`, a `DELETE` with no `WHERE` clause removes **every** row in
the table (though the table itself still exists, now empty). The same
"verify with SELECT first" habit applies.

## INSERT OR REPLACE / UPSERT

```sql
CREATE TABLE settings (
    key TEXT PRIMARY KEY,
    value TEXT
);

INSERT INTO settings (key, value) VALUES ('theme', 'dark');

-- Running this again with the same key replaces the existing row
INSERT OR REPLACE INTO settings (key, value) VALUES ('theme', 'light');

SELECT * FROM settings;
```

```text
key    value
-----  -----
theme  light
```

Standard SQL's more portable equivalent is `INSERT ... ON CONFLICT ...`
(supported by SQLite, Postgres, and MySQL with slightly different syntax) —
`INSERT OR REPLACE` is a SQLite-specific shorthand for the common case.

## Transactions — grouping changes together

```sql
BEGIN TRANSACTION;

UPDATE books SET price = price - 1 WHERE author = 'Isaac Asimov';
DELETE FROM books WHERE price < 0;

COMMIT;   -- makes both changes permanent together
-- or: ROLLBACK;  -- undoes both changes if something looked wrong
```

Wrapping related changes in a transaction means they succeed or fail as one
unit — if anything goes wrong partway through, `ROLLBACK` undoes everything
back to the `BEGIN`. Transactions are covered in full depth in
[Level 2 · Transactions & ACID Basics](../level-2/09-transactions-acid.md).

## Cheat sheet

| Operation | Syntax | Danger |
|-----------|--------|--------|
| Insert one row | `INSERT INTO t (cols) VALUES (...)` | — |
| Insert many rows | `INSERT INTO t (cols) VALUES (...), (...)` | — |
| Update rows | `UPDATE t SET col = val WHERE ...` | No `WHERE` = updates everything |
| Delete rows | `DELETE FROM t WHERE ...` | No `WHERE` = deletes everything |
| Upsert (SQLite) | `INSERT OR REPLACE INTO t (...) VALUES (...)` | — |
| Group changes | `BEGIN; ... COMMIT;` or `ROLLBACK;` | — |

## How It Actually Works

Every `INSERT`, `UPDATE`, or `DELETE` is wrapped in an implicit transaction if
you don't start one explicitly, and each one goes through SQLite's
**rollback journal** (or WAL, depending on `journal_mode`) before touching
the main database file. In the default rollback-journal mode: before
modifying any page, SQLite copies the *original* page content into a
`-journal` file first, then modifies the in-memory page cache, then — only
at commit — flushes the dirty pages to the main file and deletes the journal.
If the process crashes mid-write, the next connection to open the database
sees the leftover journal file and uses it to restore the original pages,
guaranteeing atomicity. `UPDATE` and `DELETE` both work by first *locating*
the target rows (via a scan or index seek, exactly like a `SELECT ... WHERE`)
and then rewriting or removing those specific B-tree entries in place —
`UPDATE` may even need to relocate a row within the B-tree if its new size
no longer fits the original slot, causing a page split.

## Exercise

Using the `books` table: insert 3 new books in a single `INSERT` statement.
Write an `UPDATE` that gives a 10% discount (`price * 0.9`) to every book
priced over $8 — first write the equivalent `SELECT` to check which rows will
be affected. Then write a `DELETE` that removes any book priced under $5,
again checking with `SELECT` first. Finally, wrap an `UPDATE` and a `DELETE`
together inside a `BEGIN TRANSACTION` / `COMMIT` block.
