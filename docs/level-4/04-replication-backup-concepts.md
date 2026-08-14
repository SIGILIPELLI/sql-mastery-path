# 04 · Replication & Backup Concepts

Replication (keeping copies of a database in sync across multiple servers)
and backup (taking a point-in-time snapshot you can restore from) solve
different problems — replication is about availability and read scaling
across a live system, backup is about surviving data loss. **SQLite has no
replication** — it's an embedded, single-file, single-writer database, not
a client/server system with a replication protocol. Backup, on the other
hand, it supports directly and well. This module covers what SQLite
actually gives you, and what real replication looks like in server
databases for contrast.

## Backup: the real thing, using SQLite's `.backup` command

```sql
CREATE TABLE t(id INTEGER PRIMARY KEY, v TEXT);
INSERT INTO t (v) VALUES ('a'), ('b'), ('c');
.backup backup.db
```

```sql
INSERT INTO t (v) VALUES ('d');
SELECT * FROM t;
```

```text
1|a
2|b
3|c
4|d
```

```sql
-- querying the separate backup.db file
SELECT * FROM t;
```

```text
1|a
2|b
3|c
```

`.backup` (or the underlying `sqlite3_backup_*` C API, or Python's
`sqlite3.Connection.backup()`) makes a consistent, complete copy of the
database file at that instant. The row `'d'` inserted afterward correctly
does not appear in `backup.db` — it's a true point-in-time snapshot, safe
to take even while the live database is being read or written, since
SQLite's backup API handles the consistency internally rather than
requiring you to stop writes first.

## Logical backup: .dump for portable, human-readable SQL

```sql
.dump
```

```text
PRAGMA foreign_keys=OFF;
BEGIN TRANSACTION;
CREATE TABLE t(id INTEGER PRIMARY KEY, v TEXT);
INSERT INTO t VALUES(1,'a');
INSERT INTO t VALUES(2,'b');
INSERT INTO t VALUES(3,'c');
INSERT INTO t VALUES(4,'d');
COMMIT;
```

`.dump` produces plain SQL text — every `CREATE TABLE` and `INSERT`
needed to rebuild the database from scratch (`sqlite3 new.db < dump.sql`).
Unlike `.backup`'s binary file copy, a `.dump` is portable across SQLite
versions and human-readable, at the cost of being slower to restore for
large databases (replaying millions of `INSERT` statements versus copying
raw pages).

## Backup strategy checklist (applies to any database, SQLite included)

- **Automate it.** A backup you have to remember to run is a backup you'll
  forget to run right before the day you need it.
- **Test restores, not just backups.** A backup file nobody has ever
  restored from is an assumption, not a guarantee.
- **Keep backups off the same disk/machine as the original.** A backup
  sitting next to the live file doesn't survive a disk failure or an
  accidental `rm -rf`.
- **Decide on a retention window.** Daily backups kept forever grow
  without bound — most teams keep recent daily backups plus a handful of
  older weekly/monthly ones.
- **For a live SQLite file under write load**, prefer `.backup` (or the
  backup API) over copying the raw file with `cp` — a plain file copy taken
  mid-write can capture an inconsistent, corrupt snapshot; the backup API is
  specifically designed to avoid that.

## What real replication looks like (Postgres/MySQL, conceptually)

Since SQLite has none of this natively, here's what the words mean when
you meet them in a server database:

- **Primary/replica (leader/follower) replication** — one server accepts
  writes (the primary); one or more replicas continuously receive a stream
  of changes and apply them, staying a few milliseconds to seconds behind.
  Reads can be spread across replicas to scale read throughput; writes
  still all go to the primary.
- **Synchronous vs asynchronous replication** — synchronous waits for a
  replica to confirm the write before acknowledging it to the client
  (safer, slower); asynchronous acknowledges immediately and ships the
  change to replicas afterward (faster, risks losing the last few writes if
  the primary fails before replicating them).
- **Failover** — if the primary goes down, a replica is promoted to
  become the new primary. This is the mechanism that turns replication into
  high availability, not just read scaling.
- **Logical vs physical replication** — physical replication ships raw
  disk-page changes (fast, but replica must match the primary's
  version/architecture); logical replication ships row-level changes as
  data (slower, but flexible — can replicate between different versions or
  even different table structures).

## The closest thing SQLite has to multi-writer coordination

SQLite's own answer to "many processes touching one database" isn't
replication — it's file locking and WAL mode (Level 4's tuning module):
multiple processes on the *same machine* can share one SQLite file safely,
with WAL mode letting readers proceed during a write. That's concurrency on
a single file, not redundancy across machines — if the disk holding that
one file fails, there is no automatic failover, which is precisely why
backups (not replication) are SQLite's actual resilience story.

## Cheat sheet

| Concept | SQLite | Server RDBMS (Postgres/MySQL) |
|---|---|---|
| Point-in-time snapshot | `.backup file.db` or backup API | `pg_dump`, `mysqldump`, or filesystem snapshot |
| Portable SQL export | `.dump` | `pg_dump --format=plain`, `mysqldump` |
| Multi-machine redundancy | Not supported — copy the file yourself | Primary/replica replication |
| Read scaling across servers | Not applicable — single file | Route reads to replicas |
| Automatic failover | Not applicable | Replica promotion on primary failure |
| Concurrent access on one machine | WAL mode + file locking | Built into the server's connection handling |

## Exercise

1. Write a shell one-liner using `sqlite3 mydb.db ".backup backup-$(date +%F).db"` you could put in a cron job for nightly backups, and describe how you'd verify a given backup file is actually restorable.
2. Compare `.backup` and `.dump` — for a 10GB SQLite database, which would you choose for a nightly automated backup, and which for archiving a copy you might need to open with a different SQLite version years later? Justify both.
3. In your own words, explain why "SQLite has excellent backup support but no replication" is not a contradiction — what problem does each solve, and why does an embedded single-file database only need one of them.
