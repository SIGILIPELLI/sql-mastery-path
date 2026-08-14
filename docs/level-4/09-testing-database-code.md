# 09 · Testing Database Code

Code that touches a database needs tests just like any other code — but
the database itself is part of what's under test. The standard approach:
run tests against a real (in-memory) instance of the same database engine,
not a mock, so the tests catch real constraint violations and transaction
behavior. This module builds a small transfer function and a full test
suite for it, run and passing.

## The code under test

```python
def transfer(conn, from_owner, to_owner, amount):
    cur = conn.cursor()
    cur.execute("BEGIN")
    try:
        cur.execute("SELECT balance FROM accounts WHERE owner = ?", (from_owner,))
        row = cur.fetchone()
        if row is None or row[0] < amount:
            raise ValueError("insufficient funds")
        cur.execute("UPDATE accounts SET balance = balance - ? WHERE owner = ?", (amount, from_owner))
        cur.execute("UPDATE accounts SET balance = balance + ? WHERE owner = ?", (amount, to_owner))
        conn.commit()
    except Exception:
        conn.rollback()
        raise
```

The schema it runs against:

```sql
CREATE TABLE accounts (
    id INTEGER PRIMARY KEY,
    owner TEXT NOT NULL,
    balance REAL NOT NULL CHECK (balance >= 0)
);
```

## Why an in-memory SQLite database, not a mock

Mocking the database (faking `execute`/`fetchone` to return canned values)
tests that your code *calls the right methods* — it can't tell you whether
`transfer` actually respects the `CHECK (balance >= 0)` constraint, or
whether the transaction really rolls back on failure. An in-memory SQLite
database (`sqlite3.connect(":memory:")`) is a real, fully-functional
database that exists only for the test process and disappears when the
connection closes — fast (no disk I/O) and completely isolated between test
runs, while still enforcing every constraint and transaction guarantee the
production database would.

## The full test suite, run and passing

```python
import sqlite3, unittest

SCHEMA = """
CREATE TABLE accounts (
    id INTEGER PRIMARY KEY,
    owner TEXT NOT NULL,
    balance REAL NOT NULL CHECK (balance >= 0)
);
"""

class TestTransfer(unittest.TestCase):
    def setUp(self):
        self.conn = sqlite3.connect(":memory:")
        self.conn.executescript(SCHEMA)
        self.conn.executemany("INSERT INTO accounts (owner, balance) VALUES (?, ?)",
                               [("Priya", 100.0), ("Marco", 20.0)])
        self.conn.commit()

    def test_successful_transfer_moves_money(self):
        transfer(self.conn, "Priya", "Marco", 30.0)
        balances = dict(self.conn.execute("SELECT owner, balance FROM accounts").fetchall())
        self.assertEqual(balances["Priya"], 70.0)
        self.assertEqual(balances["Marco"], 50.0)

    def test_insufficient_funds_raises_and_rolls_back(self):
        with self.assertRaises(ValueError):
            transfer(self.conn, "Marco", "Priya", 1000.0)
        balances = dict(self.conn.execute("SELECT owner, balance FROM accounts").fetchall())
        self.assertEqual(balances["Marco"], 20.0)
        self.assertEqual(balances["Priya"], 100.0)

    def test_check_constraint_blocks_negative_balance_directly(self):
        with self.assertRaises(sqlite3.IntegrityError):
            self.conn.execute("UPDATE accounts SET balance = -5 WHERE owner = 'Priya'")

if __name__ == "__main__":
    unittest.main(verbosity=2)
```

```text
test_check_constraint_blocks_negative_balance_directly ... ok
test_insufficient_funds_raises_and_rolls_back ... ok
test_successful_transfer_moves_money ... ok

----------------------------------------------------------------------
Ran 3 tests in 0.001s

OK
```

`setUp` runs before every test method, giving each one a fresh, identically
seeded database — tests don't leak state into each other, which is exactly
what lets `test_insufficient_funds_raises_and_rolls_back` assert Marco's
balance is untouched after a failed transfer, regardless of what the
previous test did to the (different, disposed-of) in-memory database.

## What each test is actually proving

- **`test_successful_transfer_moves_money`** — the happy path: money moved
  correctly, both balances updated in the same transaction.
- **`test_insufficient_funds_raises_and_rolls_back`** — the interesting
  one. It doesn't just check that `ValueError` is raised; it re-queries the
  database afterward to confirm the failed transfer left *no partial
  effect* — proving the `try`/`except`/`rollback()` actually works, not
  just that the exception propagates.
- **`test_check_constraint_blocks_negative_balance_directly`** — tests the
  database's own defense, independent of the application code. Even if
  `transfer`'s validation logic had a bug and let a negative amount through,
  the schema's `CHECK` constraint is a second layer that stops it — this
  test exists to confirm that layer works on its own, not just as a
  side-effect of `transfer` being correct.

## Fixtures and isolation, beyond a single test file

For a larger test suite:

- **One fresh connection per test** (as `setUp` does here) is the simplest
  and safest isolation strategy — no test can see another's data.
- **Seed data as a fixture function**, not copy-pasted `INSERT` statements
  in every test — change the seed once, every test picks it up.
- **Test against the same schema file production uses**, not a hand-typed
  copy that can drift out of sync — load it from the actual `schema.sql` /
  migration files your application ships.
- **For Postgres/MySQL** (not SQLite, which has no server), the equivalent
  pattern is usually a disposable test database created and torn down per
  test run, or a transaction per test that's always rolled back at the end
  — tools like `pytest-postgresql` or Django's `TestCase` automate this.

## Cheat sheet

| Testing concern | Approach |
|---|---|
| Fast, isolated test database | `sqlite3.connect(":memory:")`, fresh per test |
| Consistent starting state | Seed data in `setUp`/a fixture, not per-test copy-paste |
| Prove a rollback actually rolled back | Re-query the data after the failure, don't just check the exception |
| Test constraints independently of app logic | A test that violates the constraint directly, bypassing your code |
| Avoid mocking the database itself | Real (in-memory) database > mocked cursor — catches real constraint/transaction bugs |

## Exercise

1. Add a test `test_transfer_to_nonexistent_owner_rolls_back` that calls
   `transfer(conn, "Priya", "NoSuchPerson", 10.0)` and confirms Priya's
   balance is unchanged (the second `UPDATE` silently affects zero rows —
   is that actually caught by the current `transfer` implementation, or is
   this a bug the test uncovers?).
2. Write a test that seeds 0 accounts and confirms `transfer` raises
   cleanly rather than crashing on `row is None`.
3. Extend the schema with a `transfer_log` table and a trigger (Level 3)
   that records every balance change, then write a test asserting the log
   has exactly one row after a successful transfer and zero after a failed
   one.
