---
description: "Transactions & ACID Basics — A transaction groups multiple statements into a single all-or-nothing unit of work. Without transactions, a crash or error…"
---

# 09 · Transactions & ACID Basics

A transaction groups multiple statements into a single all-or-nothing unit
of work. Without transactions, a crash or error halfway through a multi-step
change (like moving money between two accounts) can leave your data in an
inconsistent state — money debited from one account but never credited to
the other. Transactions exist specifically to prevent that.

## Sample schema

```sql
CREATE TABLE accounts (
    id INTEGER PRIMARY KEY,
    owner TEXT NOT NULL,
    balance REAL NOT NULL CHECK (balance >= 0)
);

INSERT INTO accounts (owner, balance) VALUES ('Alice', 500.00), ('Bob', 100.00);
```

## BEGIN / COMMIT — grouping statements

```sql
BEGIN;
UPDATE accounts SET balance = balance - 200 WHERE owner = 'Alice';
UPDATE accounts SET balance = balance + 200 WHERE owner = 'Bob';
COMMIT;

SELECT * FROM accounts;
```

```text
id  owner  balance
--  -----  -------
1   Alice  300.0
2   Bob    300.0
```

`BEGIN` starts a transaction; every statement after it is provisional until
`COMMIT` makes them permanent together. Outside an explicit `BEGIN`, SQLite
(like most databases) runs each statement in its own implicit
one-statement transaction — fine for single updates, insufficient for a
multi-step change like this transfer that must succeed or fail as a unit.

## ROLLBACK — undoing an in-progress transaction

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE owner = 'Alice';

SELECT * FROM accounts;   -- mid-transaction, before rollback
```

```text
id  owner  balance
--  -----  -------
1   Alice  200.0
2   Bob    300.0
```

```sql
ROLLBACK;

SELECT * FROM accounts;   -- after rollback
```

```text
id  owner  balance
--  -----  -------
1   Alice  300.0
2   Bob    300.0
```

`ROLLBACK` discards every change made since `BEGIN`, as if the transaction
never happened. Note that other connections to the same database can't see
Alice's `200.0` balance during the open transaction either — SQLite doesn't
expose uncommitted changes to other readers.

## Atomicity in practice — a failing statement doesn't auto-rollback

This is a real trap: a constraint violation mid-transaction does **not**
automatically undo the transaction in SQLite's default mode — it aborts just
the failing statement, leaving the transaction open with earlier changes
still pending.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE owner = 'Bob';   -- succeeds

UPDATE accounts SET balance = balance - 10000 WHERE owner = 'Alice';   -- fails
```

```text
Error: CHECK constraint failed: balance >= 0
```

```sql
SELECT * FROM accounts;   -- still inside the open transaction
```

```text
id  owner  balance
--  -----  -------
1   Alice  300.0
2   Bob    250.0
```

Bob's debit is still sitting there, uncommitted but not undone either. If
your application code doesn't catch the error and explicitly issue
`ROLLBACK`, that transaction can be left open indefinitely, or accidentally
`COMMIT`'d with only half the intended change applied — silently breaking
the atomicity guarantee the transaction was supposed to provide. **Always
wrap transactional code so that any error triggers an explicit `ROLLBACK`.**

```sql
ROLLBACK;

SELECT * FROM accounts;   -- Bob's debit is gone too, as intended
```

```text
id  owner  balance
--  -----  -------
1   Alice  300.0
2   Bob    300.0
```

## ACID, briefly

| Property | Meaning | Seen above |
|----------|---------|-------------|
| **Atomicity** | A transaction's statements all succeed or all fail together | The transfer example — but only if you remember to `ROLLBACK` on error |
| **Consistency** | The database moves from one valid state to another, never violating constraints | The `CHECK (balance >= 0)` constraint blocking an overdraft |
| **Isolation** | Concurrent transactions don't see each other's uncommitted changes | Other connections can't see Alice's `200.0` mid-transaction |
| **Durability** | Once committed, changes survive a crash or power loss | SQLite's rollback journal / WAL file, `fsync`'d to disk on `COMMIT` |

## Isolation levels — how much concurrency you allow

Isolation is a spectrum, not a single switch. From loosest to strictest:

- **Read uncommitted** — can see other transactions' uncommitted changes
  ("dirty reads"). Rare in practice; SQLite doesn't support this.
- **Read committed** — only sees committed data, but a value re-read within
  the same transaction can change if another transaction commits in between
  ("non-repeatable reads"). PostgreSQL's default.
- **Repeatable read** — the same row read twice within a transaction is
  guaranteed to return the same value, but new rows matching a range query
  can still appear ("phantom reads").
- **Serializable** — transactions behave as if run one at a time in some
  order, no exceptions. SQLite's default: a writer takes an exclusive lock on
  the whole database file for the duration of the transaction, which
  sidesteps concurrency anomalies entirely at the cost of write concurrency
  (only one writer at a time, though `WAL` journal mode lets readers proceed
  unblocked).

Stricter isolation means fewer surprises but less concurrency — the right
level depends on whether your application needs many simultaneous writers or
mainly needs correctness guarantees.

## Cheat sheet

| Command | Effect |
|---------|--------|
| `BEGIN` | Start an explicit transaction |
| `COMMIT` | Make all changes since `BEGIN` permanent |
| `ROLLBACK` | Discard all changes since `BEGIN` |
| Implicit transaction | Each standalone statement outside `BEGIN`/`COMMIT` is its own transaction |
| Failed statement mid-transaction (SQLite) | Aborts that statement only — you must `ROLLBACK` explicitly to undo the whole transaction |

## How It Actually Works

SQLite guarantees **atomicity** and **durability** primarily through its
rollback journal (or WAL) mechanism: on `BEGIN`, no journal exists yet;
on the first write, SQLite copies original page images to the journal file
*before* modifying the live page cache; on `COMMIT`, it calls `fsync()` to
force the journal to durable storage, then writes the actual dirty pages to
the main database file, `fsync()`s again, and finally deletes (or truncates)
the journal — that final deletion is the atomic "commit point." If the
process or machine crashes at any point before that deletion, the next
connection to open the file finds a stale journal and automatically replays
it to roll back the incomplete transaction, restoring exactly the pre-transaction
state — this is what gives you atomicity without a separate log-replay
service. **Isolation** in SQLite's default (non-WAL) mode is coarse: a writer
holds an exclusive lock on the *entire database file* for the whole
transaction, so concurrent writers simply block/fail with `SQLITE_BUSY`
rather than being interleaved — there's no per-row locking or MVCC snapshot
isolation unless you switch to `PRAGMA journal_mode=WAL`, which lets readers
see a consistent snapshot from the WAL file while a single writer appends
new pages concurrently.

## Exercise

Using the `accounts` schema above:

1. Write a transaction that transfers `150` from Bob to Alice, and confirm
   both balances update correctly after `COMMIT`.
2. Write a transaction that would overdraw an account (violating the `CHECK`
   constraint), observe the error, then confirm with `SELECT` that an
   explicit `ROLLBACK` is required to fully undo any earlier successful
   statement in that same transaction.
3. Research (or check your SQLite version's docs) what `PRAGMA journal_mode
   = WAL;` changes about concurrent readers and writers compared to the
   default rollback-journal mode.
