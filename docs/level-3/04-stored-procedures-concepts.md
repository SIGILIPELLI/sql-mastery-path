# 04 · Stored Procedures — Concepts

In MySQL or PostgreSQL, a stored procedure is a named block of SQL
(sometimes with loops and conditionals) that lives *inside the database* and
is invoked with `CALL my_procedure(...)`. **SQLite does not have stored
procedures** — there is no `CREATE PROCEDURE` statement, and this module
won't pretend otherwise by showing syntax that doesn't run. Instead, this
lesson covers what stored procedures are for, and the two tools SQLite
actually gives you to cover the same ground: transactions driven from
application code, and user-defined functions (UDFs).

## What stored procedures are for

Across MySQL/PostgreSQL, people reach for stored procedures to:

1. **Bundle multiple statements into one atomic unit** — e.g. debit one
   account and credit another as a single all-or-nothing operation.
2. **Keep business logic close to the data**, callable from any client
   without duplicating the logic in every application.
3. **Reduce round-trips** between application and database by running
   multi-step logic in one call.

SQLite is an embedded, in-process database — there's no separate server to
send a `CALL` to, and no round-trip to save, so reason (3) mostly doesn't
apply. Reasons (1) and (2) still matter, and SQLite handles them with plain
transactions wrapped in application code instead.

## The SQLite equivalent: a transaction function in app code

```python
import sqlite3

conn = sqlite3.connect(":memory:")
conn.execute("""
CREATE TABLE accounts (
    id INTEGER PRIMARY KEY,
    owner TEXT NOT NULL,
    balance REAL NOT NULL
)
""")
conn.executemany("INSERT INTO accounts (owner, balance) VALUES (?, ?)",
                  [("Priya", 500.0), ("Marco", 120.0)])
conn.commit()

def transfer_funds(conn, from_owner, to_owner, amount):
    """App-side 'stored procedure': the transfer logic lives in Python,
    but still runs as one atomic SQL transaction."""
    cur = conn.cursor()
    try:
        cur.execute("BEGIN")
        cur.execute("SELECT balance FROM accounts WHERE owner = ?", (from_owner,))
        balance = cur.fetchone()[0]
        if balance < amount:
            raise ValueError(f"{from_owner} has insufficient funds")
        cur.execute("UPDATE accounts SET balance = balance - ? WHERE owner = ?", (amount, from_owner))
        cur.execute("UPDATE accounts SET balance = balance + ? WHERE owner = ?", (amount, to_owner))
        conn.commit()
    except Exception:
        conn.rollback()
        raise

transfer_funds(conn, "Priya", "Marco", 150.0)
for row in conn.execute("SELECT owner, balance FROM accounts"):
    print(row)
```

```text
('Priya', 350.0)
('Marco', 270.0)
```

Now the failure path — asking Marco to send money he doesn't have:

```python
try:
    transfer_funds(conn, "Marco", "Priya", 10000.0)
except ValueError as e:
    print("Rolled back:", e)
for row in conn.execute("SELECT owner, balance FROM accounts"):
    print(row)
```

```text
Rolled back: Marco has insufficient funds
('Priya', 350.0)
('Marco', 270.0)
```

The balances are unchanged — `conn.rollback()` undid the partial `UPDATE`
inside the failed `BEGIN`/`COMMIT` block, exactly the all-or-nothing
guarantee a stored procedure would give you, just expressed as a plain
Python function wrapping SQL statements instead of a database object.

## The other half: user-defined functions (UDFs)

What SQLite *does* let you register inside the engine is a **scalar
function** — logic callable directly from SQL, implemented in the host
language:

```python
def title_case(s):
    return s.title() if s else s

conn.create_function("TITLE_CASE", 1, title_case)

conn.execute("CREATE TABLE t(name TEXT)")
conn.executemany("INSERT INTO t VALUES (?)", [("priya moras",), ("MARCO diaz",)])

for row in conn.execute("SELECT name, TITLE_CASE(name) FROM t"):
    print(row)
```

```text
('priya moras', 'Priya Moras')
('MARCO diaz', 'Marco Diaz')
```

`TITLE_CASE` now behaves like a built-in SQL function — usable in any
`SELECT`, `WHERE`, or `ORDER BY` — but its implementation is ordinary Python
registered via `create_function()`, not SQL stored inside the database file.
This covers the "reusable logic invoked from SQL" half of what stored
procedures do; the "atomic multi-statement operation" half is covered by
transactions as shown above, and (for logic that must fire automatically on
data changes, not on demand) by triggers — see the next module.

## Where this leaves you

| Need | MySQL/Postgres | SQLite |
|---|---|---|
| Named, reusable multi-statement logic | `CREATE PROCEDURE` | A function in application code |
| Atomic multi-step operation | Procedure body wrapped in a transaction | `BEGIN` / `COMMIT` / `ROLLBACK` around app code |
| Custom logic callable from SQL expressions | User-defined function (`CREATE FUNCTION`) | `conn.create_function()` (host-language callback) |
| Logic that fires automatically on INSERT/UPDATE/DELETE | Trigger calling a procedure | Native `CREATE TRIGGER` (see next module) |
| Server-side execution, no round-trip | Yes — logic runs on the DB server | N/A — SQLite is embedded, there's no separate server |

## Exercise

1. Write a Python function `apply_discount(conn, order_id, percent)` that
   reads an order's `amount`, reduces it by `percent`, and updates the row —
   wrapped in `BEGIN`/`COMMIT`/`ROLLBACK` so a failure (e.g. `percent > 100`)
   leaves the row untouched.
2. Register a SQLite user-defined function `DISCOUNTED(amount, percent)`
   that returns `amount * (1 - percent/100.0)`, and use it directly inside a
   `SELECT` query.
3. In your own words, explain why SQLite's "no separate server" design is
   the main reason it skips stored procedures where MySQL/Postgres have them.
