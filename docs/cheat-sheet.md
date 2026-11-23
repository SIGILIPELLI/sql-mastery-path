---
title: "SQL Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free SQL cheat sheet: 30 quick-reference tables from every level of the SQL Mastery Path, each linked to its full lesson."
---

# SQL Cheat Sheet

Quick-reference tables from **30 lessons** of the free SQL Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [06 · Joins Basics](level-1/06-joins-basics.md)

| Join type | Keeps |
|-----------|-------|
| `INNER JOIN` | Only rows with a match on both sides |
| `LEFT JOIN` | All left-side rows, `NULL`-filled right side if no match |
| (SQLite 3.39+) `RIGHT JOIN` | All right-side rows, `NULL`-filled left side if no match — or just swap table order and use `LEFT JOIN`, which works everywhere |

### [07 · Working with NULL](level-1/07-working-with-null.md)

| Task | Correct approach |
|------|-------------------|
| Check for NULL | `WHERE col IS NULL` (never `= NULL`) |
| Check for NOT NULL | `WHERE col IS NOT NULL` |
| Provide a default | `COALESCE(col, 'default')` |
| Turn a sentinel into NULL | `NULLIF(col, sentinel_value)` |
| Count non-NULL values | `COUNT(col)` (not `COUNT(*)`) |
| NULLs in aggregates | Silently skipped, not treated as 0 |
| NULLs in arithmetic | Any operation with NULL produces NULL |

### [08 · Creating Tables](level-1/08-creating-tables.md)

| Clause | Purpose |
|--------|---------|
| `PRIMARY KEY` | Uniquely identifies each row (auto-increments if `INTEGER`) |
| `NOT NULL` | Column must always have a value |
| `UNIQUE` | No two rows may share the same value in this column |
| `DEFAULT value` | Value used automatically when not specified on insert |
| `ALTER TABLE ... ADD COLUMN` | Add a new column to an existing table |
| `DROP TABLE IF EXISTS` | Delete a table, without erroring if it's already gone |

### [09 · Inserting, Updating, Deleting Data](level-1/09-insert-update-delete.md)

| Operation | Syntax | Danger |
|-----------|--------|--------|
| Insert one row | `INSERT INTO t (cols) VALUES (...)` | — |
| Insert many rows | `INSERT INTO t (cols) VALUES (...), (...)` | — |
| Update rows | `UPDATE t SET col = val WHERE ...` | No `WHERE` = updates everything |
| Delete rows | `DELETE FROM t WHERE ...` | No `WHERE` = deletes everything |
| Upsert (SQLite) | `INSERT OR REPLACE INTO t (...) VALUES (...)` | — |
| Group changes | `BEGIN; ... COMMIT;` or `ROLLBACK;` | — |

## Level 2 · Intermediate

### [01 · Advanced Joins](level-2/01-advanced-joins.md)

| Pattern | What it's for |
|---------|----------------|
| Self-join (`t AS a JOIN t AS b`) | Relating rows in a table to other rows in the same table (managers, parent categories) |
| Filter in `ON` | Keeps unmatched left-side rows even when the filter targets the right-side table |
| Filter in `WHERE` on a `LEFT JOIN`'d table | Silently degrades the `LEFT JOIN` into an `INNER JOIN` |
| `RIGHT JOIN` | Keep everything from the right table (or swap table order and use `LEFT JOIN`) |
| `FULL OUTER JOIN` | Keep everything from both tables; emulate with `LEFT JOIN ... UNION ... LEFT JOIN` if unsupported |

### [02 · Subqueries](level-2/02-subqueries.md)

| Subquery type | Returns | Typical use |
|---------------|---------|--------------|
| Scalar | One row, one column | Comparisons (`> (SELECT ...)`) |
| `IN (...)` | A list of values | Membership filtering |
| Correlated | Re-evaluated per outer row | Per-row lookups (latest, running total) |
| `EXISTS` | True/false per outer row | "At least one related row" checks — safe with `NULL` |
| `NOT IN` with a nullable list | — | **Avoid** — a `NULL` in the list breaks the whole query; use `NOT EXISTS` instead |

### [03 · Set Operations](level-2/03-set-operations.md)

| Operator | Keeps | De-duplicates? |
|----------|-------|-----------------|
| `UNION` | Rows from either query | Yes |
| `UNION ALL` | Rows from either query | No — keeps duplicates, faster |
| `INTERSECT` | Rows in both queries | Yes |
| `EXCEPT` | Rows in the first query only, absent from the second | Yes |

### [04 · String &amp; Date Functions](level-2/04-string-date-functions.md)

| Function | Purpose |
|----------|---------|
| `TRIM`/`LTRIM`/`RTRIM` | Strip whitespace |
| `UPPER`/`LOWER` | Normalize case (do this before comparing user text) |
| `LENGTH` | Character count |
| `SUBSTR(s, start, len)` | Extract part of a string |
| `INSTR(s, sub)` | Position of a substring (0 if not found) |
| `REPLACE(s, old, new)` | Case-sensitive substring replacement |
| `\|\|` | Concatenation (SQLite/Postgres/Oracle; MySQL uses `CONCAT`, SQL Server uses `+`) |
| `DATE(d, modifier)` | Compute a new date from a base date |
| `STRFTIME(fmt, d)` | Format/extract parts of a date |
| `julianday(d)` | Convert a date to a number for arithmetic |

### [05 · CASE Expressions](level-2/05-case-expressions.md)

| Form | Syntax | When to use |
|------|--------|-------------|
| Searched `CASE` | `CASE WHEN cond1 THEN a WHEN cond2 THEN b ELSE c END` | Different conditions per branch, ranges |
| Simple `CASE` | `CASE col WHEN v1 THEN a WHEN v2 THEN b END` | Same column compared to several exact values |
| No `ELSE` | — | Unmatched rows become `NULL` — always add `ELSE` deliberately |
| `SUM(CASE WHEN cond THEN 1 ELSE 0 END)` | — | Conditional counting / pivot-style summaries |
| `CASE` in `ORDER BY` | — | Custom sort order that isn't alphabetical/numeric |

### [06 · Views](level-2/06-views.md)

| Fact | Detail |
|------|--------|
| Storage | None — a view is a saved query, re-run on every reference |
| Freshness | Always current; no refresh step needed |
| Writable in SQLite? | No, unless you add an `INSTEAD OF` trigger |
| Writable in Postgres/MySQL? | Simple single-table views often are, by default |
| `DROP VIEW IF EXISTS` | Removes the definition only, never the base table's rows |
| Good use case | Hiding a recurring join/aggregate behind a simple name |

### [07 · Indexes Basics](level-2/07-indexes-basics.md)

| Situation | Index helps? |
|-----------|---------------|
| `WHERE col = value` on an indexed column | Yes — `SEARCH` instead of `SCAN` |
| `WHERE func(col) = value` | No — the function hides the raw value from the index |
| `WHERE col LIKE 'prefix%'` | Yes — prefix search fits the sorted index |
| `WHERE col LIKE '%suffix'` | No — leading wildcard can't use a sorted index |
| Composite index `(a, b)`, filtering on `a` only | Yes |
| Composite index `(a, b)`, filtering on `b` only | No — wrong leading column |
| Heavy `INSERT`/`UPDATE` workload, low-selectivity column | Often not worth it |

### [08 · Constraints](level-2/08-constraints.md)

| Constraint | Enforces | SQLite gotcha |
|------------|----------|-----------------|
| `PRIMARY KEY` | Unique, non-`NULL` row identifier (single or composite) | Composite form: `PRIMARY KEY (col_a, col_b)` |
| `FOREIGN KEY ... REFERENCES` | Referenced row must exist | **Not enforced unless `PRAGMA foreign_keys = ON`** |
| `ON DELETE CASCADE` | Delete children when parent is deleted | Easy to lose data unintentionally — use deliberately |
| `ON DELETE SET NULL` | Clear the reference when parent is deleted | Column must be nullable |
| `UNIQUE` | No duplicate values in a column (or column combination) | |
| `CHECK (expr)` | Any custom boolean rule | Can reference multiple columns in the same row |
| `NOT NULL` | Column must always have a value | |

### [09 · Transactions &amp; ACID Basics](level-2/09-transactions-acid.md)

| Command | Effect |
|---------|--------|
| `BEGIN` | Start an explicit transaction |
| `COMMIT` | Make all changes since `BEGIN` permanent |
| `ROLLBACK` | Discard all changes since `BEGIN` |
| Implicit transaction | Each standalone statement outside `BEGIN`/`COMMIT` is its own transaction |
| Failed statement mid-transaction (SQLite) | Aborts that statement only — you must `ROLLBACK` explicitly to undo the whole transaction |

## Level 3 · Advanced

### [01 · Window Functions](level-3/01-window-functions.md)

| Function | Behavior on ties | Typical use |
|----------|-------------------|-------------|
| `ROW_NUMBER()` | Unique, sequential — ties broken arbitrarily | Paginate, dedupe, pick exactly N rows |
| `RANK()` | Ties share a rank; next rank is skipped | Leaderboards where gaps after ties are expected |
| `DENSE_RANK()` | Ties share a rank; no gap afterward | Leaderboards where ranks must stay consecutive |
| `LAG(col, n, default)` | — | Previous row's value; period-over-period change |
| `LEAD(col, n, default)` | — | Next row's value |
| `SUM()`/`AVG()`/`COUNT()` `OVER(...)` | — | Running totals, moving averages, percent-of-group |
| No `ORDER BY` in `OVER(...)` | — | Frame defaults to the whole partition |
| `ORDER BY` present, no explicit frame | — | **Trap:** frame defaults to `UNBOUNDED PRECEDING` through current row |

### [02 · CTEs &amp; Recursive Queries](level-3/02-ctes-recursive-queries.md)

| Concept | Detail |
|---------|--------|
| `WITH name AS (...)` | Defines a non-recursive CTE, usable once in the query that follows |
| `WITH RECURSIVE name(...) AS (...)` | Required keyword in SQLite for a self-referencing CTE |
| Anchor query | The first `SELECT`, before `UNION ALL` — the starting row(s) |
| Recursive query | The second `SELECT`, which references the CTE's own name |
| Termination | Recursion stops automatically once the recursive step returns zero new rows |
| `UNION ALL` vs `UNION` | Always use `UNION ALL` in recursion — `UNION`'s deduplication is wasted work and can hide legitimate duplicate rows |
| Multiple CTEs | Separate with commas: `WITH a AS (...), b AS (...) SELECT ...` |

### [03 · Query Optimization &amp; EXPLAIN QUERY PLAN](level-3/03-query-optimization-explain.md)

| Plan output | Meaning |
|---|---|
| `SCAN table` | Full table scan — every row checked |
| `SEARCH table USING INDEX ix (col=?)` | Indexed lookup — jumps to matching rows |
| `SEARCH table USING COVERING INDEX ix` | Index alone answers the query, no table access |
| `SCAN table USING COVERING INDEX ix` | Ordered scan of the index, often replaces a sort |
| Wrapping a column (`col + 0`, `lower(col)`) | Disables index use on that column |
| `ANALYZE` | Refreshes planner statistics in `sqlite_stat1` |

### [05 · Triggers](level-3/05-triggers.md)

| Concept | Syntax | Notes |
|---|---|---|
| Fire after a change | `AFTER INSERT/UPDATE/DELETE ON table` | Row is already written; good for audit logs, cascades |
| Fire before a change | `BEFORE INSERT/UPDATE/DELETE ON table` | Can inspect and reject via `RAISE(ABORT, ...)` |
| Scope to one column | `UPDATE OF col ON table` | Fires only when that column is in the `UPDATE` |
| Old/new row values | `OLD.col` / `NEW.col` | `OLD` unavailable on `INSERT`, `NEW` unavailable on `DELETE` |
| Conditional firing | `WHEN condition` | Skips the body when the condition is false |
| Cancel + roll back | `SELECT RAISE(ABORT, 'msg')` | Only meaningful in `BEFORE` triggers |
| Drop a trigger | `DROP TRIGGER name;` | — |

### [06 · Normalization &amp; Schema Design](level-3/06-normalization-schema-design.md)

| Symptom in a flat table | Fix |
|---|---|
| Same customer's name/email repeated per order | Move to a `customers` table, reference by `customer_id` |
| Same product's name/price repeated per order line | Move to a `products` table, reference by `product_id` |
| One order needs many products | Junction table (`order_items`) with a foreign key to each side |
| Updating one fact requires touching many rows | Sign of a normalization gap — that fact belongs in its own table |
| Read-heavy reporting query, source data static | Deliberate denormalization can be acceptable — document the trade-off |

### [07 · JSON in SQL](level-3/07-json-in-sql.md)

| Function | Purpose |
|---|---|
| `json_extract(col, '$.path')` | Pull a value out, JSON-typed if it's an object/array |
| `col ->> '$.path'` | Same, but unquotes scalar strings/numbers |
| `col -> '$.path'` | Same as `json_extract`, keeps JSON typing |
| `json_valid(text)` | `1`/`0` — is this valid JSON |
| `json_each(col, '$.path')` | Table-valued function — one row per array/object element |
| `json_set(col, '$.path', val)` | Returns modified JSON (doesn't persist without `UPDATE`) |
| `json_group_array(col)` / `json_group_object(k, v)` | Aggregate rows into a JSON array/object |
| `CREATE INDEX ix ON t(col ->> '$.path')` | Index a specific JSON path for fast filtering |

### [08 · Full-Text Search](level-3/08-full-text-search.md)

| Task | Syntax |
|---|---|
| Create an FTS5 table | `CREATE VIRTUAL TABLE t USING fts5(col1, col2)` |
| Search | `SELECT * FROM t WHERE t MATCH 'word'` |
| Relevance order | `ORDER BY rank` (ascending = best match first) |
| Explicit BM25 with column weights | `bm25(t, weight1, weight2, ...)` |
| Highlighted excerpt | `snippet(t, col_idx, start, end, ellipsis, max_tokens)` |
| Phrase match | `MATCH '"exact phrase"'` |
| Prefix match | `MATCH 'prefix*'` |
| Keep FTS in sync with a real table | Triggers on the source table's INSERT/UPDATE/DELETE |

### [09 · Data Migration Patterns](level-3/09-data-migration-patterns.md)

| Change | Supported directly? | How |
|---|---|---|
| Add column | Yes | `ALTER TABLE t ADD COLUMN c ...` |
| Rename column | Yes (3.25+) | `ALTER TABLE t RENAME COLUMN old TO new` |
| Rename table | Yes | `ALTER TABLE t RENAME TO new_name` |
| Drop column | Yes (3.35+) | `ALTER TABLE t DROP COLUMN c` |
| Add CHECK / UNIQUE / FK constraint | No | Rebuild pattern |
| Change column type | No | Rebuild pattern |
| Any rebuild | — | `PRAGMA foreign_keys=OFF` → transaction → create new → copy → drop old → rename → `COMMIT` → `PRAGMA foreign_keys=ON` |

## Level 4 · Master

### [01 · Advanced Indexing Strategies](level-4/01-advanced-indexing-strategies.md)

| Index type | Syntax | Use when |
|---|---|---|
| Composite | `CREATE INDEX ix ON t(a, b)` | Queries filter on `a`, or `a` and `b` together — not `b` alone |
| Covering | Any index containing every column a query needs | Read-heavy query on a fixed set of columns |
| Partial | `CREATE INDEX ix ON t(col) WHERE cond` | Queries always filter by `cond`, and `cond` excludes most rows |
| Expression | `CREATE INDEX ix ON t(expr(col))` | Queries always filter using that exact expression |
| Leftmost-prefix rule | — | A composite index only helps if the query's filter starts with the index's first column |

### [02 · Performance Tuning at Scale](level-4/02-performance-tuning-at-scale.md)

| Lever | SQLite command | Effect |
|---|---|---|
| Batch writes | Wrap many `INSERT`/`UPDATE` in one `BEGIN`/`COMMIT` | Avoids per-statement fsync cost — often 100x+ |
| Concurrent readers/writers | `PRAGMA journal_mode=WAL` | Readers don't block on an in-progress write |
| More in-memory cache | `PRAGMA cache_size = -N` (N in KB) | Fewer disk reads for hot pages |
| Confirm an index is used | `EXPLAIN QUERY PLAN` | `SCAN` → full table; `SEARCH` → index used |
| Durability vs speed trade-off | `PRAGMA synchronous=NORMAL` (with WAL) | Fewer fsyncs, small durability window on crash |

### [03 · Partitioning Concepts](level-4/03-partitioning-concepts.md)

| Concept | Real partitioning (Postgres/MySQL) | SQLite |
|---|---|---|
| Split table by range/hash | `PARTITION BY RANGE/HASH` | Separate tables you create and name yourself |
| Unified query interface | Automatic — one table name | Manual — a `UNION ALL` view |
| Partition pruning | Automatic, planner-driven | Manual — application picks which table to query |
| Drop old data | `ALTER TABLE ... DROP PARTITION` | `DROP TABLE partition_name` |
| Routing new rows | Automatic | Application logic or a trigger |
| Cross-partition constraints | Supported (with caveats) | Not enforced — application's responsibility |

### [04 · Replication &amp; Backup Concepts](level-4/04-replication-backup-concepts.md)

| Concept | SQLite | Server RDBMS (Postgres/MySQL) |
|---|---|---|
| Point-in-time snapshot | `.backup file.db` or backup API | `pg_dump`, `mysqldump`, or filesystem snapshot |
| Portable SQL export | `.dump` | `pg_dump --format=plain`, `mysqldump` |
| Multi-machine redundancy | Not supported — copy the file yourself | Primary/replica replication |
| Read scaling across servers | Not applicable — single file | Route reads to replicas |
| Automatic failover | Not applicable | Replica promotion on primary failure |
| Concurrent access on one machine | WAL mode + file locking | Built into the server's connection handling |

### [05 · Security &amp; SQL Injection](level-4/05-security-sql-injection.md)

| Practice | Vulnerable | Safe |
|---|---|---|
| Building a query | f-string / string concatenation with input | `?` placeholders + `execute(query, params)` |
| Table/column names from input | Concatenated directly | Validate against an allow-list — parameters can't cover identifiers |
| Error handling | Query errors shown to the user | Generic user-facing message, detailed log server-side |
| DB account permissions | App connects as an admin/superuser | App connects with only the privileges it needs |
| Input validation | Trust the input's shape | Validate type/format before it reaches SQL |

### [06 · NoSQL vs SQL](level-4/06-nosql-vs-sql.md)

| Question | Leans SQL | Leans NoSQL |
|---|---|---|
| Do records reference each other? | Yes → relational joins | No, mostly standalone documents |
| Does the schema change often, per-record? | No, fairly fixed | Yes, frequently and per-document |
| Do you need multi-row ACID transactions? | Yes | Often not required |
| Is horizontal scale across many servers a hard requirement? | Not the default strength | Often designed in from the start |
| Do you need ad-hoc queries across many fields? | Yes, that's SQL's strength | Harder — often needs a specialized query layer |

### [07 · SQL from Application Code](level-4/07-sql-from-application-code.md)

| Task | Python `sqlite3` |
|---|---|
| Connect | `sqlite3.connect("file.db")` |
| Parameterized query | `conn.execute("... WHERE x = ?", (value,))` |
| Dict-like rows | `conn.row_factory = sqlite3.Row` |
| Batch insert | `conn.executemany(sql, list_of_tuples)` |
| Auto commit/rollback | `with conn: ...` |
| Explicit cursor, memory-safe iteration | `cur = conn.cursor(); for row in cur:` |
| Manual transaction control | `conn.execute("BEGIN")` / `conn.commit()` / `conn.rollback()` |

### [08 · Data Warehousing Concepts](level-4/08-data-warehousing-concepts.md)

| Term | Meaning |
|---|---|
| Fact table | One row per event, holds numeric measures + foreign keys to dimensions |
| Dimension table | Describes one axis to slice by (customer, product, date) |
| Surrogate key | An artificial integer key (`customer_key`) used for fast joins, distinct from any natural/business key |
| Star schema | One fact table, several dimension tables, no dimension-to-dimension joins |
| Snowflake schema | A star schema where dimensions are further normalized into sub-dimensions |
| ETL / ELT | The batch pipeline that populates a warehouse from OLTP sources |
| Grain | What one fact row represents (e.g. "one line item," not "one order") — defines what you can and can't aggregate to |

### [09 · Testing Database Code](level-4/09-testing-database-code.md)

| Testing concern | Approach |
|---|---|
| Fast, isolated test database | `sqlite3.connect(":memory:")`, fresh per test |
| Consistent starting state | Seed data in `setUp`/a fixture, not per-test copy-paste |
| Prove a rollback actually rolled back | Re-query the data after the failure, don't just check the exception |
| Test constraints independently of app logic | A test that violates the constraint directly, bypassing your code |
| Avoid mocking the database itself | Real (in-memory) database > mocked cursor — catches real constraint/transaction bugs |

---

Go deeper with the full [SQL Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
