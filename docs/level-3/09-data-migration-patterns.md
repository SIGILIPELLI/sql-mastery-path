# 09 · Data Migration Patterns

Schemas change after data already exists in production — adding a column,
renaming one, tightening a constraint. SQLite supports a growing subset of
`ALTER TABLE` directly, but for anything it doesn't support (like adding a
`CHECK` constraint to an existing table), the standard workaround is the
"12-step" rebuild pattern: create a new table with the shape you want, copy
the data across, then swap names.

## Sample schema

```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT
);

INSERT INTO users (name, email) VALUES
    ('Priya', 'priya@example.com'), ('Marco', NULL);
```

## Simple migration: adding a column

```sql
ALTER TABLE users ADD COLUMN status TEXT NOT NULL DEFAULT 'active';

SELECT * FROM users;
```

```text
id  name   email               status
--  -----  ------------------  ------
1   Priya  priya@example.com  active
2   Marco                     active
```

`ADD COLUMN` with a `NOT NULL` column requires a `DEFAULT`, since SQLite
has to backfill existing rows with *something* — every existing row gets
`'active'` immediately, and the operation is fast because SQLite doesn't
have to rewrite the whole table for a simple column addition.

## Simple migration: renaming a column

```sql
ALTER TABLE users RENAME COLUMN name TO full_name;

SELECT * FROM users;
```

```text
id  full_name  email               status
--  ---------  ------------------  ------
1   Priya      priya@example.com  active
2   Marco                         active
```

SQLite (3.25+) supports `RENAME COLUMN` directly — no table rebuild needed.
Application code that referenced `name` must be updated at the same time,
or reads/writes against that column will start failing the moment this
migration runs.

## Simple migration: dropping a column

```sql
ALTER TABLE users DROP COLUMN email;

SELECT * FROM users;
```

```text
id  full_name  status
--  ---------  ------
1   Priya      active
2   Marco      active
```

`DROP COLUMN` (3.35+) is also directly supported now — earlier SQLite
versions needed the full rebuild pattern below even for this.

## The rebuild pattern — for what ALTER TABLE can't do

SQLite's `ALTER TABLE` still can't add a `CHECK` constraint, change a
column's type, or add a `UNIQUE`/`FOREIGN KEY` constraint to an existing
table. For those, rebuild:

```sql
PRAGMA foreign_keys=OFF;
BEGIN TRANSACTION;

CREATE TABLE users_new (
    id INTEGER PRIMARY KEY,
    full_name TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'inactive'))
);

INSERT INTO users_new SELECT id, full_name, status FROM users;

DROP TABLE users;
ALTER TABLE users_new RENAME TO users;

COMMIT;
PRAGMA foreign_keys=ON;

SELECT * FROM users;
```

```text
id  full_name  status
--  ---------  ------
1   Priya      active
2   Marco      active
```

The steps: turn off foreign key enforcement (so the temporary absence of
`users` doesn't break other tables' references mid-migration), build the
new table with the constraint you actually want, copy every row across,
drop the old table, rename the new one into its place, and turn foreign
keys back on — all wrapped in one transaction so a failure partway through
leaves the original table untouched instead of a half-migrated database.

## Confirming the constraint actually took effect

```sql
INSERT INTO users (full_name, status) VALUES ('Test', 'bogus');
```

```text
Runtime error: CHECK constraint failed: status IN ('active','inactive')
```

The rebuilt table now rejects a `status` value the original schema would
have silently accepted — this is the whole point of the rebuild: getting a
constraint SQLite's `ALTER TABLE` alone can't add.

## Migration checklist for production data

1. **Back up first** — `.backup` or copy the file before any structural
   change; the rebuild pattern is safe *within* a transaction, but a bug in
   your migration script itself isn't protected by that.
2. **Wrap in a transaction** — as shown above, so a mid-migration failure
   rolls back cleanly rather than leaving two half-tables.
3. **Handle NULLs and defaults explicitly** — a new `NOT NULL` column
   needs either a `DEFAULT` or an explicit backfill `UPDATE` before the
   constraint can apply to existing rows.
4. **Test the copy step against production-shaped data** — a rebuild that
   works on a 3-row dev table can still fail on real data with edge-case
   values (empty strings, unexpected NULLs, duplicate values that violate a
   new `UNIQUE`).
5. **Recreate indexes and triggers** — `DROP TABLE` on the old table drops
   its indexes and triggers too; the rebuild only recreates what you
   explicitly write into `users_new` and afterward.

## Cheat sheet

| Change | Supported directly? | How |
|---|---|---|
| Add column | Yes | `ALTER TABLE t ADD COLUMN c ...` |
| Rename column | Yes (3.25+) | `ALTER TABLE t RENAME COLUMN old TO new` |
| Rename table | Yes | `ALTER TABLE t RENAME TO new_name` |
| Drop column | Yes (3.35+) | `ALTER TABLE t DROP COLUMN c` |
| Add CHECK / UNIQUE / FK constraint | No | Rebuild pattern |
| Change column type | No | Rebuild pattern |
| Any rebuild | — | `PRAGMA foreign_keys=OFF` → transaction → create new → copy → drop old → rename → `COMMIT` → `PRAGMA foreign_keys=ON` |

## Exercise

Using the `users` table above:

1. Write a migration that adds a `UNIQUE` constraint on `full_name` using
   the rebuild pattern, and confirm a duplicate insert now fails.
2. Write a migration that changes `status` from `TEXT` to an `INTEGER`
   code (`0` = inactive, `1` = active), including the data transformation
   in the `INSERT ... SELECT` step.
3. List, in order, every step you'd take before running a rebuild migration
   against a production database with a million rows — not just the SQL,
   but the surrounding process (backup, testing, rollback plan).
