# 07 · SQL from Application Code

Every technique so far ran through the `sqlite3` CLI. Real applications
talk to a database through a driver/DB-API instead — this module covers
Python's built-in `sqlite3` module as the concrete example, since its
patterns (parameterized queries, connection/cursor objects, transaction
handling) map directly onto other languages' drivers (`psycopg2` for
Postgres, JDBC for Java, `mysql2` for Node).

## Connecting and running parameterized queries

```python
import sqlite3

conn = sqlite3.connect(":memory:")
conn.execute("CREATE TABLE tasks (id INTEGER PRIMARY KEY, title TEXT NOT NULL, done INTEGER NOT NULL DEFAULT 0)")

conn.execute("INSERT INTO tasks (title) VALUES (?)", ("Write report",))
conn.execute("INSERT INTO tasks (title, done) VALUES (?, ?)", ("Review PR", 1))
conn.commit()

cur = conn.execute("SELECT id, title, done FROM tasks")
print(cur.fetchall())
```

```text
[(1, 'Write report', 0), (2, 'Review PR', 1)]
```

`conn.execute()` is shorthand for creating a cursor and calling
`cursor.execute()` — fine for one-off statements. The `?` placeholders are
the same parameterization from the security module: always pass values
this way, never with string formatting.

## Row factory: getting dict-like rows instead of tuples

```python
conn.row_factory = sqlite3.Row
row = conn.execute("SELECT * FROM tasks WHERE id = 1").fetchone()
print(dict(row))
```

```text
{'id': 1, 'title': 'Write report', 'done': 0}
```

By default, `sqlite3` returns plain tuples — `row[0]`, `row[1]` — which
gets unreadable fast. Setting `row_factory = sqlite3.Row` makes rows
accessible by column name (`row['title']`) while still supporting index
access, and `dict(row)` converts cleanly to JSON-ready output. Most drivers
in most languages have an equivalent setting; check for it rather than
manually zipping column names to values.

## executemany: batch inserts without a manual loop

```python
conn.executemany("INSERT INTO tasks (title) VALUES (?)",
                  [("Task A",), ("Task B",), ("Task C",)])
conn.commit()

print(conn.execute("SELECT COUNT(*) FROM tasks").fetchone())
```

```text
(5,)
```

`executemany` runs one `INSERT` per tuple in the list, inside the same
underlying mechanics as a loop of individual `execute()` calls — but it's
the idiomatic way to express "insert this batch," and pairs with the
batching-for-performance lesson from Level 4's tuning module: still wrap
it (or let the driver wrap it) in a single transaction rather than
committing after each row.

## Transactions as a context manager

```python
try:
    with conn:
        conn.execute("INSERT INTO tasks (title) VALUES (?)", ("Will be rolled back",))
        raise RuntimeError("boom")
except RuntimeError:
    pass

print(conn.execute("SELECT COUNT(*) FROM tasks WHERE title = 'Will be rolled back'").fetchone())
```

```text
(0,)
```

Using `conn` as a context manager (`with conn:`) wraps the block in a
transaction automatically — a successful block commits when it exits, an
exception rolls it back. The row never persisted because the `RuntimeError`
inside the `with` block triggered a rollback; this is the app-code
equivalent of the `BEGIN`/`COMMIT`/`ROLLBACK` pattern from Level 3's
migration module, just handled by the driver instead of written by hand.

## Cursors vs the connection object

A `Cursor` tracks the state of one particular query's results (position for
`fetchone()`, the full result set for `fetchall()`); a `Connection` is the
actual link to the database file and owns transaction state. `conn.execute()`
implicitly creates a throw-away cursor for you — for anything you need to
iterate incrementally, keep an explicit cursor:

```python
cur = conn.cursor()
cur.execute("SELECT * FROM tasks")
for row in cur:  # iterates lazily, doesn't load everything into memory at once
    pass
```

For large result sets, iterating a cursor directly (or using
`fetchmany(n)`) avoids pulling the entire result into memory the way
`fetchall()` does.

## Connection pooling and where it matters

SQLite is embedded and file-based — there's no network round-trip to a
separate server, so a fresh `sqlite3.connect()` is cheap, and long-lived
connection pools matter far less than they do for Postgres/MySQL. In a
server-based RDBMS, opening a new TCP connection per request is expensive
(handshake, auth, server-side resource allocation), so applications keep a
pool of already-open connections and borrow/return them per request — a
pattern SQLAlchemy, HikariCP, and most ORMs implement for you. The
principle (don't pay connection setup cost repeatedly) carries over even
though SQLite's own cost for it is much lower.

## Cheat sheet

| Task | Python `sqlite3` |
|---|---|
| Connect | `sqlite3.connect("file.db")` |
| Parameterized query | `conn.execute("... WHERE x = ?", (value,))` |
| Dict-like rows | `conn.row_factory = sqlite3.Row` |
| Batch insert | `conn.executemany(sql, list_of_tuples)` |
| Auto commit/rollback | `with conn: ...` |
| Explicit cursor, memory-safe iteration | `cur = conn.cursor(); for row in cur:` |
| Manual transaction control | `conn.execute("BEGIN")` / `conn.commit()` / `conn.rollback()` |

## Exercise

1. Write a function `add_task(conn, title)` that inserts a task and returns
   its new `id` using `cursor.lastrowid`.
2. Write a function `mark_done(conn, task_id)` wrapped in a `with conn:`
   block, and demonstrate that an invalid `task_id` (that violates some
   check you add) rolls back cleanly.
3. Rewrite the `executemany` example to insert 1000 rows two ways — inside
   one `with conn:` block, and as 1000 separate auto-committed statements —
   and time the difference (see Level 4's performance tuning module for
   the pattern).
