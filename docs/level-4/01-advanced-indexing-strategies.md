# 01 · Advanced Indexing Strategies

Level 3 covered single-column indexes and `EXPLAIN QUERY PLAN`. This module
goes further into composite indexes, covering indexes, partial indexes, and
expression indexes — all real, runnable SQLite features. Where SQLite's
behavior differs from MySQL/Postgres (which have their own index types like
hash or GIN indexes), that's called out explicitly; everything here is
SQLite's native B-tree index with different shapes of key.

## Sample schema

```sql
CREATE TABLE employees (
    id INTEGER PRIMARY KEY,
    dept TEXT NOT NULL,
    last_name TEXT NOT NULL,
    first_name TEXT NOT NULL,
    salary REAL NOT NULL,
    active INTEGER NOT NULL DEFAULT 1
);
-- 3000 rows, ~4 departments, random salaries, 10% inactive
```

## Composite indexes and the leftmost-prefix rule

```sql
CREATE INDEX idx_dept_lastname ON employees(dept, last_name);

EXPLAIN QUERY PLAN
SELECT * FROM employees WHERE dept = 'Eng' AND last_name = 'Last100';
```

```text
QUERY PLAN
`--SEARCH employees USING INDEX idx_dept_lastname (dept=? AND last_name=?)
```

```sql
EXPLAIN QUERY PLAN
SELECT * FROM employees WHERE last_name = 'Last100';
```

```text
QUERY PLAN
`--SCAN employees
```

A composite index on `(dept, last_name)` is usable for lookups on `dept`
alone, or `dept` + `last_name` together — but **not** for `last_name` alone,
because a B-tree index is sorted by its first column first. Filtering on
just the second column is like trying to look someone up in a phone book
sorted by city-then-name when you only know the name — you'd have to check
every city. This is the leftmost-prefix rule: put the column your queries
filter on *most consistently* first in a composite index.

## Covering indexes — skip the table entirely

```sql
CREATE INDEX idx_dept_salary ON employees(dept, salary);

EXPLAIN QUERY PLAN
SELECT dept, salary FROM employees WHERE dept = 'Sales';
```

```text
QUERY PLAN
`--SEARCH employees USING COVERING INDEX idx_dept_salary (dept=?)
```

Because the query only asks for `dept` and `salary` — exactly the columns
in the index — SQLite never has to open the table itself; the index alone
"covers" the query. This is the fastest possible read: one B-tree traversal,
zero table lookups. As soon as you `SELECT *` or add a column not in the
index, the plan reverts to a normal (non-covering) `SEARCH` that also visits
the table row.

```sql
EXPLAIN QUERY PLAN
SELECT dept, COUNT(*) FROM employees GROUP BY dept;
```

```text
QUERY PLAN
`--SCAN employees USING COVERING INDEX idx_dept_salary
```

Even a full `GROUP BY` scan benefits — SQLite scans the *index* instead of
the table, since `idx_dept_salary` already has `dept` and doesn't need
anything else from the row to count it.

## Partial indexes — index only the rows you query

```sql
CREATE INDEX idx_active_salary ON employees(salary) WHERE active = 1;

EXPLAIN QUERY PLAN
SELECT * FROM employees WHERE active = 1 AND salary > 100000;
```

```text
QUERY PLAN
`--SEARCH employees USING INDEX idx_active_salary (salary>?)
```

```sql
EXPLAIN QUERY PLAN
SELECT * FROM employees WHERE salary > 100000;
```

```text
QUERY PLAN
`--SCAN employees
```

`WHERE active = 1` in the `CREATE INDEX` statement makes this a **partial
index** — it only contains the 90% of rows that are active, skipping the
inactive 10% entirely, so it's both smaller and cheaper to maintain than
indexing every row. The query without `active = 1` in its own `WHERE`
clause can't use it (the index simply doesn't have the inactive rows to
give back), so a partial index is a deliberate trade: fast for the query
shape you built it for, unusable for others.

## Expression indexes — indexing a computed value

```sql
CREATE INDEX idx_lower_lastname ON employees(lower(last_name));

EXPLAIN QUERY PLAN
SELECT * FROM employees WHERE lower(last_name) = 'last50';
```

```text
QUERY PLAN
`--SEARCH employees USING INDEX idx_lower_lastname (<expr>=?)
```

This is the fix for the trap from Level 3's optimization module — wrapping
a column in a function normally disables index use, but indexing the
*expression itself* (`lower(last_name)`) gives SQLite something to search
directly, as long as your query uses the exact same expression.

## What SQLite doesn't have

MySQL and Postgres offer index types SQLite doesn't: hash indexes for
pure equality lookups, GIN/GiST indexes for full-text and geometric data,
and multiple index algorithms selectable per index. SQLite has one index
structure — the B-tree — used for every case, including its `rowid` and
`INTEGER PRIMARY KEY` lookups, `UNIQUE` constraints, and FTS5 (which uses a
specialized B-tree-backed structure internally, covered in Level 3). The
composite/partial/expression variations above are how far SQLite's single
index type stretches to cover what other engines split across index types.

## Cheat sheet

| Index type | Syntax | Use when |
|---|---|---|
| Composite | `CREATE INDEX ix ON t(a, b)` | Queries filter on `a`, or `a` and `b` together — not `b` alone |
| Covering | Any index containing every column a query needs | Read-heavy query on a fixed set of columns |
| Partial | `CREATE INDEX ix ON t(col) WHERE cond` | Queries always filter by `cond`, and `cond` excludes most rows |
| Expression | `CREATE INDEX ix ON t(expr(col))` | Queries always filter using that exact expression |
| Leftmost-prefix rule | — | A composite index only helps if the query's filter starts with the index's first column |

## Exercise

Using the `employees` table above:

1. Create a composite index that speeds up `WHERE dept = ? AND active = ?`
   and confirm with `EXPLAIN QUERY PLAN` that both conditions are used in
   the index search.
2. Create a partial index for `WHERE active = 0` (the inactive minority)
   and explain why a partial index built for the *minority* case is
   usually more valuable than one built for the majority.
3. Explain, without running anything, why an index on `(dept, last_name)`
   would *not* help a query filtering only on `WHERE last_name LIKE
   'Last1%'` even though `last_name` is in the index.
