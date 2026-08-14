# 02 · Performance Tuning at Scale

Indexing (previous module) fixes slow reads. This module covers the other
half: write throughput, connection-level tuning via `PRAGMA`, and the
general principles that carry over to any RDBMS at scale — even though the
specific knobs shown here are SQLite's own.

## The single biggest win: batch writes in one transaction

```python
import sqlite3, time, os

conn = sqlite3.connect("a.db")
conn.execute("CREATE TABLE t(id INTEGER PRIMARY KEY, v INTEGER)")
start = time.time()
for i in range(2000):
    conn.execute("INSERT INTO t (v) VALUES (?)", (i,))
    conn.commit()
row_by_row = time.time() - start
```

```python
conn2 = sqlite3.connect("b.db")
conn2.execute("CREATE TABLE t(id INTEGER PRIMARY KEY, v INTEGER)")
start = time.time()
conn2.execute("BEGIN")
for i in range(2000):
    conn2.execute("INSERT INTO t (v) VALUES (?)", (i,))
conn2.commit()
batched = time.time() - start

print(f"row-by-row commit: {row_by_row:.3f}s")
print(f"single transaction: {batched:.3f}s")
```

```text
row-by-row commit (2000 inserts, on-disk): 0.470s
single transaction (2000 inserts, on-disk): 0.002s
```

**A 239x difference**, from 2000 tiny transactions doing 2000 separate disk
syncs, versus one transaction doing one. Every `COMMIT` in SQLite's default
mode forces the write to be durably flushed to disk before returning — fine
once, ruinous 2000 times. This is the single highest-leverage performance
fix available in almost any database: batch related writes into as few
transactions as reasonably possible.

## WAL mode: readers don't block writers

```sql
PRAGMA journal_mode;
```

```text
journal_mode
------------
delete
```

```sql
PRAGMA journal_mode=WAL;
PRAGMA journal_mode;
```

```text
journal_mode
------------
wal
```

SQLite's default journal mode (`delete` — a rollback journal file per
transaction) locks the whole database file for the duration of a write,
blocking concurrent readers. `WAL` (write-ahead logging) mode instead
appends changes to a separate `-wal` file and lets readers keep reading the
last-committed state while a write is in progress — the standard setting
for any SQLite database handling concurrent read/write access. Note this
`PRAGMA` has no effect on an in-memory database (there's no file to
journal); it matters for file-backed databases under real concurrent load.

## cache_size: how much of the database stays in memory

```sql
PRAGMA cache_size;
```

```text
cache_size
----------
-2000
```

`cache_size` controls how many pages SQLite keeps cached in memory (a
negative value means "kilobytes of cache," so `-2000` is roughly 2MB by
default). For a large, frequently-queried database, raising this
(`PRAGMA cache_size = -64000` for ~64MB) reduces disk reads for hot data —
the general principle any database tuning guide repeats: memory is orders
of magnitude faster than disk, so keep the working set in memory whenever
you can afford the RAM.

## Reading a slow query's actual cost, not just its plan

`EXPLAIN QUERY PLAN` (Level 3) shows the *strategy*; to see real cost at
scale, generate representative data volume and measure:

```sql
EXPLAIN QUERY PLAN SELECT COUNT(*) FROM logs WHERE level = 'ERROR';
```

```text
QUERY PLAN
`--SCAN logs
```

A `SCAN` over 50,000 rows to count a filtered subset is the kind of plan
that's invisible in a 20-row dev table and very visible once real data
volume shows up — the fix is the same as Level 3's optimization lesson: add
an index on the filtered column (`level`) and confirm the plan flips to
`SEARCH`.

## General principles beyond SQLite's specific knobs

These carry over to MySQL, Postgres, and most RDBMSes, even though the
exact commands differ:

- **Batch writes.** Fewer, larger transactions beat many small ones — the
  fsync-per-commit cost above is universal, not SQLite-specific.
- **Measure with real data volume.** A query that looks instant against
  200 rows can be a full scan against 200 million; always test against a
  representative row count before trusting a plan.
- **Index what you filter and join on, not what you display.** Extra
  indexes speed up reads but slow down every write, since each index needs
  updating too — don't index columns nothing filters on.
- **Cache what's hot.** Whether it's SQLite's `cache_size`, Postgres'
  `shared_buffers`, or an application-level cache — keeping frequently-read
  data in memory is the standard fix once disk I/O becomes the bottleneck.
- **Concurrency needs a plan.** SQLite's WAL mode, Postgres' MVCC, and
  MySQL's InnoDB isolation levels all solve the same underlying problem —
  readers and writers needing to coexist without blocking each other or
  seeing inconsistent data.

## Cheat sheet

| Lever | SQLite command | Effect |
|---|---|---|
| Batch writes | Wrap many `INSERT`/`UPDATE` in one `BEGIN`/`COMMIT` | Avoids per-statement fsync cost — often 100x+ |
| Concurrent readers/writers | `PRAGMA journal_mode=WAL` | Readers don't block on an in-progress write |
| More in-memory cache | `PRAGMA cache_size = -N` (N in KB) | Fewer disk reads for hot pages |
| Confirm an index is used | `EXPLAIN QUERY PLAN` | `SCAN` → full table; `SEARCH` → index used |
| Durability vs speed trade-off | `PRAGMA synchronous=NORMAL` (with WAL) | Fewer fsyncs, small durability window on crash |

## Exercise

1. Reproduce the batching benchmark above with 5000 rows instead of 2000
   and confirm the gap grows, not shrinks, with more rows.
2. Set `PRAGMA journal_mode=WAL` on a file-backed database and explain, in
   your own words, what concurrent workload it specifically helps —
   contrast with a single-connection, read-only workload where it wouldn't
   matter.
3. Given a `logs` table with 50,000 rows and no index on `level`, write the
   `CREATE INDEX` statement that would fix the `SCAN` shown above, and
   predict (before running it) what the new `EXPLAIN QUERY PLAN` output
   would say.
