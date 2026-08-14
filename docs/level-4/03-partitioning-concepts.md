# 03 · Partitioning Concepts

Partitioning splits one logically-huge table into physically-separate
chunks — by date range, by region, by hash of a key — so operations can
target just the relevant chunk instead of the whole table. **SQLite has no
built-in partitioning feature** — no `PARTITION BY` clause like Postgres,
no partition management like MySQL. This module covers the concept
generally, and shows the manual pattern SQLite users actually reach for:
splitting data across separate tables yourself and presenting them as one
via a view.

## Why partition at all

At large scale, three problems partitioning solves:

1. **Queries that only need recent data** shouldn't have to scan years of
   history — a query for "orders this month" against a partitioned table
   only touches this month's partition.
2. **Deleting old data** (a common retention requirement) becomes "drop the
   partition" instead of a slow `DELETE ... WHERE` that has to find and
   remove matching rows one at a time.
3. **Maintenance operations** (rebuilding an index, vacuuming) can run
   against one partition without locking the entire table.

In Postgres, this is `PARTITION BY RANGE (created_at)` with the engine
routing rows to the right partition automatically and a query planner smart
enough to skip irrelevant partitions ("partition pruning"). MySQL has
similar native support. SQLite has neither — there is no single "big table"
the engine splits up for you.

## The manual pattern in SQLite: separate tables, unioned by a view

```sql
CREATE TABLE events_2025_01 (id INTEGER PRIMARY KEY, occurred_at TEXT NOT NULL, payload TEXT);
CREATE TABLE events_2025_02 (id INTEGER PRIMARY KEY, occurred_at TEXT NOT NULL, payload TEXT);

INSERT INTO events_2025_01 (occurred_at, payload) VALUES ('2025-01-05', 'a'), ('2025-01-20', 'b');
INSERT INTO events_2025_02 (occurred_at, payload) VALUES ('2025-02-03', 'c');

CREATE VIEW events AS
    SELECT * FROM events_2025_01
    UNION ALL
    SELECT * FROM events_2025_02;

SELECT * FROM events ORDER BY occurred_at;
```

```text
id  occurred_at  payload
--  -----------  -------
1   2025-01-05   a
2   2025-01-20   b
1   2025-02-03   c
```

Application code that only cares about "all events" queries the `events`
view and doesn't need to know it's backed by two tables. This is exactly
what real partitioning does under the hood — the difference is that in
Postgres/MySQL the engine manages routing and pruning for you; here, you
manage it by hand.

## Targeting one partition directly for speed

```sql
SELECT * FROM events_2025_02 WHERE occurred_at >= '2025-02-01';
```

```text
id  occurred_at  payload
--  -----------  -------
1   2025-02-03   c
```

A query that knows it only wants February data can skip the view entirely
and hit `events_2025_02` directly — no scanning January's rows at all.
This is manual "partition pruning": the application (or a query-building
layer) decides which table(s) to touch based on the date range requested,
rather than relying on the engine to figure it out from a `WHERE` clause
against one big table.

## Dropping an old partition is instant

```sql
DROP TABLE events_2025_01;

SELECT name FROM sqlite_master WHERE type = 'table';
```

```text
name
--------------
events_2025_02
```

`DROP TABLE` removes the whole table (and its data) as a metadata
operation — no row-by-row deletion, no `WHERE` clause to evaluate. This is
the core operational win partitioning offers for time-series or log data
with a retention policy: "delete everything older than 90 days" becomes
"drop the partitions older than 90 days," a near-instant operation instead
of a slow bulk `DELETE`.

## The real trade-offs of the manual approach

- **You write the routing logic.** The engine won't automatically send an
  `INSERT` to the right monthly table — your application code (or a
  trigger with conditional logic) has to pick it.
- **Cross-partition queries cost a UNION.** The `events` view above scans
  every underlying table for any query that doesn't target a specific one —
  fine for 2 tables, unwieldy for 60 months of history without additional
  tooling to generate/maintain the view.
- **Constraints don't span partitions.** A `UNIQUE` constraint on
  `events_2025_01` says nothing about `events_2025_02` — global uniqueness
  across "partitions" has to be enforced by the application, not the schema.
- **This is usually a sign SQLite has grown past its use case.** SQLite is
  designed for embedded, single-file use — a workload large enough to need
  real partitioning (billions of rows, automatic partition pruning, online
  partition maintenance) is often a sign to migrate to Postgres or a
  purpose-built time-series database rather than simulate partitioning by
  hand indefinitely.

## Cheat sheet

| Concept | Real partitioning (Postgres/MySQL) | SQLite |
|---|---|---|
| Split table by range/hash | `PARTITION BY RANGE/HASH` | Separate tables you create and name yourself |
| Unified query interface | Automatic — one table name | Manual — a `UNION ALL` view |
| Partition pruning | Automatic, planner-driven | Manual — application picks which table to query |
| Drop old data | `ALTER TABLE ... DROP PARTITION` | `DROP TABLE partition_name` |
| Routing new rows | Automatic | Application logic or a trigger |
| Cross-partition constraints | Supported (with caveats) | Not enforced — application's responsibility |

## Exercise

1. Extend the pattern above to a third table `events_2025_03` and update
   the `events` view to include it.
2. Write the application-side logic (pseudocode or Python) that decides
   which monthly table to `INSERT` a new event into, based on its
   `occurred_at` date.
3. Explain why "drop the partition" is so much cheaper than "DELETE FROM
   events WHERE occurred_at < X" on a large table, in terms of what work
   each operation actually does.
