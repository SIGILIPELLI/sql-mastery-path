---
title: "SQL Interview Questions and Answers (180 Questions)"
description: "180 SQL interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the SQL Mastery Path."
---

# SQL Interview Questions and Answers

**180 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free SQL Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · Setup &amp; First Queries](level-1/01-setup-first-queries.md)

**Q1.** How does the sqlite3 shell treat dot-commands such as .tables versus SQL statements?

**Answer:** Dot-commands need no semicolon; SQL statements end with a semicolon

Dot-commands are shell-specific; SQL statements are ended by the semicolon.

**Q2.** What is a SQLite database?

**Answer:** A single file on disk

There is no server to install and no accounts or network needed.

**Q3.** Which dot-command shows the CREATE TABLE statement for the students table?

**Answer:** .schema students

.tables lists tables; .schema shows how a table was defined.

**Q4.** What ends a SQL statement in the shell?

**Answer:** The semicolon, not the newline

Statements can span multiple lines; indentation is only for human readability.

### [02 · SELECT Basics &amp; Data Types](level-1/02-select-basics-data-types.md)

**Q5.** Why list exact columns instead of SELECT * in application code?

**Answer:** It's explicit and doesn't break or change behavior if a column is added

SELECT * is fine for quick exploration in the shell.

**Q6.** What does AS do in SELECT title AS book_title?

**Answer:** Gives the column an alias in the result

AS is optional but improves readability, and aliases are essential for expressions.

**Q7.** Which query computes a 10% discounted price column?

**Answer:** SELECT title, price * 0.9 AS discounted_price FROM books;

You can compute values in SELECT, not just read stored columns.

**Q8.** What does the lesson say SELECT retrieves?

**Answer:** Data, and most of SQL revolves around shaping a SELECT

Getting comfortable with SELECT's anatomy pays off for the rest of the course.

### [03 · Filtering with WHERE](level-1/03-filtering-where.md)

**Q9.** Which operator has higher precedence, AND or OR?

**Answer:** AND binds tighter than OR

Use parentheses whenever you mix them, since relying on precedence is a common source of bugs.

**Q10.** Is BETWEEN 1960 AND 1990 inclusive?

**Answer:** Yes, both endpoints are included

It equals &gt;= 1960 AND &lt;= 1990.

**Q11.** Which clause matches a column against a set of values?

**Answer:** IN ('Fantasy', 'Cyberpunk')

IN is a shorthand for several OR comparisons on the same column.

**Q12.** When is WHERE evaluated?

**Answer:** Per row, before any grouping or sorting

WHERE narrows the result set down to rows that match.

### [04 · Sorting &amp; Limiting Results](level-1/04-sorting-limiting.md)

**Q13.** Without ORDER BY, what does SQL guarantee about row order?

**Answer:** Nothing; the order can vary between runs

The database may return rows in whatever order is convenient internally.

**Q14.** What does ORDER BY genre ASC, price DESC do?

**Answer:** Sorts genres A–Z, and within each genre sorts price highest to lowest

Ties in the first column are broken by the next, each with its own direction.

**Q15.** Why avoid ORDER BY 2 in real code?

**Answer:** Adding or reordering selected columns silently changes what it sorts by

Sort by column name instead.

**Q16.** Where does SQLite place NULLs in ascending order?

**Answer:** First, before any real value

Other databases differ: Postgres defaults to NULLS LAST for ASC.

### [05 · Aggregate Functions &amp; GROUP BY](level-1/05-aggregates-group-by.md)

**Q17.** What is the difference between COUNT(*) and COUNT(genre)?

**Answer:** COUNT(*) counts all rows; COUNT(genre) counts only rows where genre is not NULL

COUNT(column) ignores NULLs in that column.

**Q18.** What rule applies to non-aggregated columns in a SELECT with GROUP BY?

**Answer:** Each must appear in the GROUP BY

SELECT genre, title, AVG(price) ... GROUP BY genre is invalid because SQL wouldn't know which title to show.

**Q19.** What does GROUP BY genre with COUNT(*) return?

**Answer:** One row per genre with the number of books in it

It splits rows into buckets by value and computes the aggregate for each.

**Q20.** Which function returns the highest price?

**Answer:** MAX(price)

The five core aggregates are COUNT, SUM, AVG, MIN and MAX.

### [06 · Joins Basics](level-1/06-joins-basics.md)

**Q21.** Why is The Lathe of Heaven missing from the INNER JOIN result?

**Answer:** Its author_id is NULL, so it has no match in authors

INNER JOIN returns only rows where both sides match.

**Q22.** What does LEFT JOIN do with a left row that has no match?

**Answer:** Keeps it and fills the right table's columns with NULL

It keeps every row from the left table.

**Q23.** How do you find only books with no known author after a LEFT JOIN?

**Answer:** WHERE authors.id IS NULL

Filtering for NULL on the right table's key exposes the unmatched rows.

**Q24.** When are table aliases required?

**Answer:** When two tables share a column name such as id, and they keep long queries readable

For example, books AS b and authors AS a.

### [07 · Working with NULL](level-1/07-working-with-null.md)

**Q25.** What does SELECT * FROM books WHERE price = NULL return?

**Answer:** Zero rows, always

Comparing unknown to anything yields unknown, which SQL treats as not true.

**Q26.** What is the correct way to find rows with no price?

**Answer:** WHERE price IS NULL

Use IS NULL and IS NOT NULL.

**Q27.** What does COALESCE(discontinued_reason, 'still in print') return for a NULL reason?

**Answer:** 'still in print'

COALESCE returns the first argument that isn't NULL.

**Q28.** What does price + 1 give when price is NULL?

**Answer:** NULL

NULL propagates through arithmetic instead of being treated as zero.

### [08 · Creating Tables](level-1/08-creating-tables.md)

**Q29.** What does CREATE TABLE produce?

**Answer:** An empty table: the structure with no rows

It defines columns, types and rules.

**Q30.** What happens on INSERT INTO books (price) VALUES (9.99) when title is NOT NULL?

**Answer:** It fails with a NOT NULL constraint error

Error: NOT NULL constraint failed: books.title.

**Q31.** What is special about INTEGER PRIMARY KEY in SQLite?

**Answer:** It auto-increments and aliases the internal row ID

AUTOINCREMENT is only for edge cases where IDs must never be reused.

**Q32.** What does a DEFAULT value allow?

**Answer:** Omitting the column from INSERT so SQLite fills it in

For example in_stock INTEGER NOT NULL DEFAULT 1.

### [09 · Inserting, Updating, Deleting Data](level-1/09-insert-update-delete.md)

**Q33.** What happens if you run UPDATE books SET price = 0 with no WHERE clause?

**Answer:** Every row's price becomes 0

Verify with an equivalent SELECT ... WHERE first.

**Q34.** Why list column names explicitly in INSERT?

**Answer:** It is self-documenting and doesn't break if column order changes

INSERT INTO books (title, author, price) is preferred over relying on column order.

**Q35.** What does DELETE FROM books; do?

**Answer:** Removes every row, but the table itself still exists

The table remains, now empty.

**Q36.** What does INSERT OR REPLACE do when the primary key already exists?

**Answer:** Replaces the existing row

In the settings example, theme becomes 'light'.

### [10 · Project — Library/Bookstore Database](level-1/10-project-library-database.md)

**Q37.** Why does query 2 use LEFT JOIN?

**Answer:** Books with no known author still appear

'Unknown Author Book' has a NULL author_id.

**Q38.** What does HAVING COUNT(*) &gt;= 2 do in query 4?

**Answer:** Keeps only authors with two or more books

HAVING filters groups after aggregation.

**Q39.** Which tables make up the project database?

**Answer:** authors and books

It's a two-table library database.

**Q40.** What does in_stock INTEGER NOT NULL DEFAULT 1 mean?

**Answer:** Every book must have a value, and it defaults to 1 (in stock)

The seed data uses 0 for out-of-stock books.

## Level 2 · Intermediate

### [01 · Advanced Joins](level-2/01-advanced-joins.md)

**Q41.** Why does the employee/manager self-join use LEFT JOIN?

**Answer:** Alice has no manager, and INNER JOIN would drop her

manager_id IS NULL for Alice.

**Q42.** What does a self-join require?

**Answer:** Two distinct aliases for the same table

Without aliases SQL can't tell which copy a column belongs to.

**Q43.** Why is WHERE d.active = 1 wrong in the LEFT JOIN example?

**Answer:** It filters out unmatched rows, turning the LEFT JOIN into an INNER JOIN

Alice, Carol and Frank vanished; put the active check in ON instead.

**Q44.** What is the single most common LEFT JOIN bug per the lesson?

**Answer:** The WHERE-vs-ON trap

Conditions on the right table belong in ON to preserve unmatched left rows.

### [02 · Subqueries](level-2/02-subqueries.md)

**Q45.** What is a scalar subquery?

**Answer:** One that returns exactly one row and one column

If it returns more than one row, SQLite raises an error.

**Q46.** What does WHERE id IN (SELECT customer_id FROM orders) return?

**Answer:** Customers who have at least one order

Dave has never ordered, so he's excluded.

**Q47.** What makes a subquery correlated?

**Answer:** It references a column from the outer query, so it runs for each outer row

It can't be evaluated once up front.

**Q48.** In the AVG example, how many times does the inner query run?

**Answer:** Once; the outer query compares every amount to that single value

It's uncorrelated, so it computes the average once.

### [03 · Set Operations](level-2/03-set-operations.md)

**Q49.** How do set operations combine results compared with joins?

**Answer:** Vertically, stacking rows of two SELECTs; joins combine horizontally

Set operations are the SQL equivalent of union, intersection and difference.

**Q50.** What is the rule for SELECTs combined with a set operator?

**Answer:** They must return the same number of columns with compatible types

The result uses the first query's column names; mismatched counts fail with a parse error.

**Q51.** How do UNION and UNION ALL differ?

**Answer:** UNION removes duplicates; UNION ALL keeps every row

UNION ALL gives accurate row counts and is faster on large sets.

**Q52.** Where must ORDER BY go in a UNION query?

**Answer:** After the last SELECT, applying to the combined result

It is not attached to an individual query.

### [04 · String &amp; Date Functions](level-2/04-string-date-functions.md)

**Q53.** Why does LENGTH('  Alice Nguyen  ') return 16 while the name looks like 12 characters?

**Answer:** Whitespace is invisible but counted

TRIM brings it back to 12.

**Q54.** What does SUBSTR(string, start, length) do, and how is it indexed?

**Answer:** Extracts part of a string, 1-indexed

Combined with INSTR you can pull out a first name.

**Q55.** Why normalize case with LOWER() before comparing user-entered text?

**Answer:** 'Alice' and 'alice' are different strings for = in most databases

SQLite comparison is case-sensitive by default.

**Q56.** Which function strips leading and trailing whitespace?

**Answer:** TRIM

LTRIM and RTRIM do one side only.

### [05 · CASE Expressions](level-2/05-case-expressions.md)

**Q57.** How is a searched CASE evaluated?

**Answer:** Top to bottom, stopping at the first matching WHEN

Order matters; put the narrowest condition first.

**Q58.** What does CASE return when no WHEN matches and there is no ELSE?

**Answer:** NULL

Order 3 with status_code 3 shows NULL in the example.

**Q59.** When is the simple CASE form useful?

**Answer:** When every branch compares the same expression against specific values

CASE status_code WHEN 1 THEN ... is equivalent to repeated equality tests.

**Q60.** Where can CASE be used?

**Answer:** In SELECT, WHERE or ORDER BY

It is SQL's inline if/else.

### [06 · Views](level-2/06-views.md)

**Q61.** What is a view?

**Answer:** A SELECT query saved under a name, queryable like a table

It doesn't store data itself.

**Q62.** When does a view's underlying query run?

**Answer:** Every time you query the view

It runs fresh against the real tables.

**Q63.** What is a key benefit of the category_revenue view?

**Answer:** The join and GROUP BY logic is defined once and can be fixed in one place

Views hide complexity.

**Q64.** Why is Mechanical Keyboard missing from in_stock_products?

**Answer:** Its stock is 0 and the view filters WHERE stock &gt; 0

The WHERE filter is applied automatically every time.

### [07 · Indexes Basics](level-2/07-indexes-basics.md)

**Q65.** What does a SCAN in EXPLAIN QUERY PLAN mean?

**Answer:** SQLite checked every row in the table

On 5 million rows that would be very slow; SEARCH ... USING INDEX jumps almost directly to the match.

**Q66.** What is an index, per the lesson?

**Answer:** A separate sorted structure that lets the database find rows without checking every row

It is like a book's index.

**Q67.** Does the idx_&lt;table&gt;_&lt;column&gt; naming pattern affect how an index works?

**Answer:** No, it's just a common readable habit

Any valid name works.

**Q68.** What does the order of columns in a composite index determine?

**Answer:** What the index is useful for

The country, signup_date index suits filtering by country first, then signup_date.

### [08 · Constraints](level-2/08-constraints.md)

**Q69.** What is the SQLite surprise about foreign keys?

**Answer:** They're parsed but not enforced unless you turn enforcement on

The Ghost Book with author_id 999 was inserted without error.

**Q70.** How do you enable foreign key enforcement?

**Answer:** PRAGMA foreign_keys = ON; on every connection

The setting isn't persisted in the database file.

**Q71.** What does ON DELETE CASCADE do?

**Answer:** Deletes child rows automatically when the parent row is deleted

Deleting Asimov removed both of his books.

**Q72.** What do constraints do in general?

**Answer:** Reject bad data at insert or update time

They stop bad data causing bugs later.

### [09 · Transactions &amp; ACID Basics](level-2/09-transactions-acid.md)

**Q73.** What does a transaction provide?

**Answer:** An all-or-nothing unit of work across several statements

It prevents inconsistent states such as money debited but never credited.

**Q74.** What does ROLLBACK do?

**Answer:** Discards every change since BEGIN

It is as if the transaction never happened.

**Q75.** In SQLite's default mode, what happens when a statement fails mid-transaction due to a constraint?

**Answer:** Only that statement is aborted; the transaction stays open with earlier changes pending

You must issue ROLLBACK (or handle it) yourself.

**Q76.** What happens outside an explicit BEGIN?

**Answer:** Each statement runs in its own implicit one-statement transaction

This is insufficient for multi-step changes like a transfer.

### [10 · Project — E-commerce Reporting](level-2/10-project-ecommerce-reporting.md)

**Q77.** Why does the schema have an order_items table?

**Answer:** One order has many products and a product appears in many orders: a many-to-many relationship via a junction table

It uses a composite PRIMARY KEY of (order_id, product_id).

**Q78.** Why is unit_price stored on order_items instead of looked up from products?

**Answer:** Historical orders stay accurate if a product's price later changes

The price at the time of sale is preserved.

**Q79.** Which four tables does the project use?

**Answer:** customers, products, orders and order_items

The schema is realistic enough to need joins, subqueries, views, CASE and transactions.

**Q80.** What does CHECK (price &gt; 0) enforce?

**Answer:** Prices must be positive

Constraints reject bad data at insert or update time.

## Level 3 · Advanced

### [01 · Window Functions](level-3/01-window-functions.md)

**Q81.** How does a window function differ from GROUP BY?

**Answer:** Each input row keeps its own line while gaining a computed column

Window functions handle top-N per group, running totals and period comparisons that GROUP BY cannot express.

**Q82.** What does PARTITION BY region do?

**Answer:** Splits rows into independent windows, one per region, each restarting numbering

ORDER BY inside OVER decides the order within each partition.

**Q83.** For Bob's two tied 800.0 rows in West, what do RANK and DENSE_RANK show?

**Answer:** 4 and 4 for both

ROW_NUMBER still gives 4 and 5, since it always assigns unique sequential numbers.

**Q84.** What syntax marks a function as a window function?

**Answer:** OVER (...)

Every window function is a normal function followed by OVER (...).

### [02 · CTEs &amp; Recursive Queries](level-3/02-ctes-recursive-queries.md)

**Q85.** What is a CTE?

**Answer:** A named temporary result set defined with WITH and used like a table in that query

CTEs make complex queries readable, naming each step instead of nesting subqueries.

**Q86.** What can a recursive CTE do that a normal one cannot?

**Answer:** Reference itself, walking hierarchies of unknown depth

It is the standard SQL way to walk org charts, category trees and folders.

**Q87.** What does the self-referencing manager_id make the employees data?

**Answer:** Hierarchical rather than flat

It points to another row in the same table.

**Q88.** Can CTEs reference each other in one query?

**Answer:** Yes, chained with commas, each with its own name

This avoids another indentation level for each step.

### [03 · Query Optimization &amp; EXPLAIN QUERY PLAN](level-3/03-query-optimization-explain.md)

**Q89.** What does EXPLAIN QUERY PLAN do?

**Answer:** Shows the plan SQLite would use without running the query

It tells you whether SQLite scans many rows or uses an index.

**Q90.** What is the single most common optimization per the lesson?

**Answer:** Index the columns your WHERE clauses filter on

It changes SCAN into SEARCH ... USING INDEX.

**Q91.** What does SEARCH ... USING INDEX mean?

**Answer:** A B-tree lookup instead of a full scan

SQLite jumps directly to the matching rows.

**Q92.** In the join example, why did SQLite make customers the outer loop?

**Answer:** It has only 5 rows, so it scans them and searches orders by index for each match

Plan order matters in joins.

### [04 · Stored Procedures — Concepts](level-3/04-stored-procedures-concepts.md)

**Q93.** Does SQLite have stored procedures?

**Answer:** No, there is no CREATE PROCEDURE

The lesson refuses to show syntax that doesn't run.

**Q94.** What does SQLite offer instead, per the lesson?

**Answer:** Transactions driven from application code and user-defined functions

The transfer_funds example does this in Python.

**Q95.** Why does the 'reduce round-trips' reason mostly not apply to SQLite?

**Answer:** It is embedded and in-process, with no separate server

There's no server to send a CALL to.

**Q96.** What does the transfer_funds example guarantee?

**Answer:** The debit and credit run as one atomic SQL transaction

It checks the balance, updates both accounts, and commits (rolling back on failure).

### [05 · Triggers](level-3/05-triggers.md)

**Q97.** Inside a trigger body, what do OLD and NEW refer to?

**Answer:** The row before the change and the row after it

In the audit trigger, OLD.stock is 50 and NEW.stock is 45.

**Q98.** What does AFTER UPDATE OF stock mean?

**Answer:** The trigger fires only when the stock column changes, after the change is made

Updating name alone wouldn't fire it.

**Q99.** How does the BEFORE trigger block a bad write?

**Answer:** SELECT RAISE(ABORT, 'message') when the WHEN condition is true

BEFORE runs before the row is written, so it can veto the change.

**Q100.** Does SQLite support triggers?

**Answer:** Yes, fully, with real CREATE TRIGGER syntax

Unlike stored procedures, triggers are natively supported.

### [06 · Normalization &amp; Schema Design](level-3/06-normalization-schema-design.md)

**Q101.** What is the update anomaly?

**Answer:** One real-world fact needing multiple easy-to-miss writes, leaving inconsistent copies

Nadia ended up with two different emails depending on the row.

**Q102.** What does 1NF require?

**Answer:** Each column holds a single atomic value

No comma-separated lists crammed into one field.

**Q103.** What is normalization's goal per the lesson?

**Answer:** Store each fact in exactly one place

The payoff is that updates, inserts and deletes can't silently corrupt data.

**Q104.** What does 3NF require of non-key columns?

**Answer:** They depend on the key and nothing but the key

2NF requires dependence on the whole primary key.

### [07 · JSON in SQL](level-3/07-json-in-sql.md)

**Q105.** How does SQLite store JSON?

**Answer:** As plain TEXT, parsed on demand by JSON functions

SQLite has no dedicated JSON column type.

**Q106.** What does the -&gt;&gt; operator return compared with -&gt;?

**Answer:** -&gt;&gt; unquotes the result into plain SQL text; -&gt; keeps it as JSON

-&gt; is useful when passing the result into another JSON function.

**Q107.** How can you guarantee well-formed JSON at write time?

**Answer:** Use json_valid() in a CHECK constraint

SQLite stores text with no schema enforcement otherwise.

**Q108.** What does json_each do?

**Answer:** Turns a JSON array into one row per element

It's a table-valued function, used with a comma join per source row.

### [08 · Full-Text Search](level-3/08-full-text-search.md)

**Q109.** Why is LIKE '%database%' a poor choice for text search?

**Answer:** It can't rank by relevance and a leading % forces a full scan

FTS5 provides word indexing, ranking, phrase and prefix queries.

**Q110.** How do you create a full-text searchable table?

**Answer:** CREATE VIRTUAL TABLE ... USING fts5(...)

It looks like a normal table but is backed by an inverted index.

**Q111.** How do you query an FTS5 table for a word?

**Answer:** WHERE articles MATCH 'database'

MATCH performs a real word-index lookup.

**Q112.** In FTS5's rank column, which values are more relevant?

**Answer:** More negative values

ORDER BY rank without DESC gives best-match-first (BM25).

### [09 · Data Migration Patterns](level-3/09-data-migration-patterns.md)

**Q113.** Why does ADD COLUMN with NOT NULL require a DEFAULT in SQLite?

**Answer:** SQLite must backfill existing rows with something

Existing rows immediately get the default (e.g. 'active').

**Q114.** What must you do when renaming a column with RENAME COLUMN?

**Answer:** Update application code that references the old name at the same time

Reads and writes against the old column name fail once the migration runs.

**Q115.** Which change still needs the table-rebuild pattern?

**Answer:** Adding a CHECK constraint to an existing table

ALTER TABLE can't add a CHECK, change a column type, or add UNIQUE/FOREIGN KEY to an existing table.

**Q116.** What is the rebuild pattern's outline?

**Answer:** Create a new table in the desired shape, copy the data across, then swap names

The lesson runs it inside a transaction with foreign keys temporarily off.

### [10 · Project: Normalized Schema &amp; Analytics](level-3/10-project-normalized-schema-analytics.md)

**Q117.** Why does the project index every foreign key column?

**Answer:** Every one of them gets joined on

Examples: idx_orders_customer_id and idx_order_items_product_id.

**Q118.** What is order_items' role in the schema?

**Answer:** Junction table for the many-to-many between orders and products, carrying quantity

quantity is data specific to that pairing.

**Q119.** Why does the schema have a separate categories table?

**Answer:** Normalization: each category fact is stored once

products reference it with category_id.

**Q120.** What does categories.name UNIQUE enforce?

**Answer:** No two categories share a name

It is a constraint on the data.

## Level 4 · Master

### [01 · Advanced Indexing Strategies](level-4/01-advanced-indexing-strategies.md)

**Q121.** Can an index on (dept, last_name) speed up a query filtering only on last_name?

**Answer:** No, it falls back to a SCAN (leftmost-prefix rule)

A B-tree is sorted by its first column first, like a phone book sorted by city then name.

**Q122.** What is a covering index?

**Answer:** An index containing all the columns the query needs, so the table is never opened

SELECT dept, salary with an index on (dept, salary) shows USING COVERING INDEX.

**Q123.** What breaks a covering index?

**Answer:** Selecting a column not in the index, e.g. SELECT *

The plan reverts to a normal SEARCH that also visits the table row.

**Q124.** Which column should come first in a composite index?

**Answer:** The one your queries filter on most consistently

That is the leftmost-prefix rule.

### [02 · Performance Tuning at Scale](level-4/02-performance-tuning-at-scale.md)

**Q125.** Why was one transaction about 239x faster than 2000 separate commits?

**Answer:** Each COMMIT forces a durable disk flush, and one transaction does it once

Batching related writes is the single highest-leverage performance fix.

**Q126.** What does WAL mode change for concurrency?

**Answer:** Readers can keep reading the last-committed state while a write happens

The default delete journal mode locks the database file during a write, blocking readers.

**Q127.** Does PRAGMA journal_mode=WAL matter for an in-memory database?

**Answer:** No, there is no file to journal

It matters for file-backed databases under concurrent load.

**Q128.** What does cache_size control?

**Answer:** How many pages SQLite keeps cached in memory

A negative value is interpreted as an amount of memory rather than a page count.

### [03 · Partitioning Concepts](level-4/03-partitioning-concepts.md)

**Q129.** Does SQLite have built-in partitioning?

**Answer:** No; the manual pattern is separate tables presented via a view

Postgres and MySQL have native support; SQLite does not.

**Q130.** Why is dropping a partition better than DELETE ... WHERE for old data?

**Answer:** It avoids finding and removing matching rows one at a time

Retention becomes a drop-the-partition operation.

**Q131.** What is partition pruning?

**Answer:** The planner skipping partitions irrelevant to the query

A query for this month's orders only touches this month's partition.

**Q132.** In the manual SQLite pattern, how does application code see the data?

**Answer:** As one events view built with UNION ALL over the separate tables

The view hides that two tables back it, but you manage routing yourself.

### [04 · Replication &amp; Backup Concepts](level-4/04-replication-backup-concepts.md)

**Q133.** What is the difference between replication and backup?

**Answer:** Replication keeps live copies in sync for availability and read scaling; backup is a restorable snapshot against data loss

SQLite has no replication but supports backup well.

**Q134.** What does .backup produce?

**Answer:** A consistent point-in-time copy of the database file, safe while it's in use

Rows inserted afterward do not appear in backup.db.

**Q135.** How does .dump differ from .backup?

**Answer:** It produces portable, readable SQL, but restores slower for large databases

Restore with sqlite3 new.db &lt; dump.sql.

**Q136.** Which statement fits the checklist?

**Answer:** Test restores, not just backups

An untested backup is an assumption, not a guarantee.

### [05 · Security &amp; SQL Injection](level-4/05-security-sql-injection.md)

**Q137.** How did the attacker log in as admin with the input admin'--?

**Answer:** The quote closed the string and -- commented out the password check

The database never evaluated the AND password = ... part.

**Q138.** What is the fix demonstrated?

**Answer:** Parameterized queries with ? placeholders

The same malicious input returns an empty result.

**Q139.** Why does string concatenation make injection possible?

**Answer:** Input becomes literal SQL text instead of a value

Anything a caller passes becomes part of the query itself.

**Q140.** Which are other injection shapes the lesson mentions?

**Answer:** ' OR '1'='1 and '; DROP TABLE users;--

One matches every row; the other runs a second destructive statement.

### [06 · NoSQL vs SQL](level-4/06-nosql-vs-sql.md)

**Q141.** Which describes NoSQL accurately?

**Answer:** An umbrella for document, key-value, wide-column and graph databases

Each trades away parts of the relational model for something else.

**Q142.** How do relationships typically differ between SQL and document stores?

**Answer:** SQL uses JOINs across normalized tables; document stores usually embed related data (denormalized)

From the lesson's comparison table.

**Q143.** What does Priya's extra preferences object show in the JSON example?

**Answer:** Per-row fields without a schema migration

It's the flexibility of a document store, achieved via a JSON TEXT column.

**Q144.** Which is generally easier to scale horizontally?

**Answer:** Many NoSQL systems, designed to scale out from day one

Relational systems traditionally scale up on a bigger server.

### [07 · SQL from Application Code](level-4/07-sql-from-application-code.md)

**Q145.** What does setting conn.row_factory = sqlite3.Row allow?

**Answer:** Accessing columns by name (and dict(row) for JSON-ready output)

By default sqlite3 returns plain tuples, which get unreadable fast.

**Q146.** How should values be passed into SQL from application code?

**Answer:** With ? placeholders, never string formatting

It's the same parameterization from the security module.

**Q147.** What does executemany do?

**Answer:** Runs one INSERT per tuple in a list, the idiomatic way to batch inserts

Still wrap it in a single transaction rather than committing after each.

**Q148.** Why does the lesson use Python's sqlite3 as the example?

**Answer:** Its patterns map onto other languages' drivers (psycopg2, JDBC, mysql2)

Parameterized queries, connections/cursors and transaction handling are common patterns.

### [08 · Data Warehousing Concepts](level-4/08-data-warehousing-concepts.md)

**Q149.** What is OLTP optimized for compared to OLAP?

**Answer:** Fast, small, frequent reads/writes; OLAP is for large scans over history

OLTP: 'record this order'; OLAP: 'total revenue by region, last 8 quarters'.

**Q150.** What is the fact table in a star schema?

**Answer:** One row per measurable event, holding numeric measures and foreign keys to dimensions

fact_sales holds quantity and revenue with keys to dim_customer, dim_product and dim_date.

**Q151.** What does a dimension table describe?

**Answer:** One axis you'd want to slice by, such as customer, product or date

They surround the fact table, like the points of a star.

**Q152.** How does schema shape differ between OLTP and OLAP?

**Answer:** OLTP is normalized for fast writes; OLAP is denormalized (star) for scanning and aggregation

Data freshness also differs: warehouses often load in batches.

### [09 · Testing Database Code](level-4/09-testing-database-code.md)

**Q153.** Why use an in-memory SQLite database rather than a mock to test database code?

**Answer:** It's a real database that enforces constraints and transactions, so tests catch real violations

A mock only tests that you call the right methods.

**Q154.** What does an in-memory database do when the connection closes?

**Answer:** It disappears, giving isolation between test runs

It's fast (no disk I/O) and isolated.

**Q155.** What does transfer() do when funds are insufficient?

**Answer:** Raises ValueError after rolling back

The except block calls conn.rollback() and re-raises.

**Q156.** Why does the test's setUp create the schema and seed rows each time?

**Answer:** Each test starts from a known clean state

It uses sqlite3.connect(":memory:") with executescript.

### [10 · Capstone Project](level-4/10-capstone-project.md)

**Q157.** Why does unit_price on order_items deliberately duplicate products.price?

**Answer:** products.price can change later, but an order must keep the price the customer paid

The lesson calls this the correct call, not an oversight.

**Q158.** What does the CHECK on orders.status enforce?

**Answer:** Only pending, paid, shipped or cancelled are allowed

It is a constraint with an IN list.

**Q159.** What is order_audit maintained by?

**Answer:** A trigger recording status changes

The capstone includes a trigger that maintains an audit trail.

**Q160.** Which pieces does the capstone combine?

**Answer:** Normalized schema, indexes, trigger, full-text search, JSON metadata and a verified query plan

It is a complete small order-management database run end to end.

## Mixed questions across all levels

**Q161.** Use this data for the next questions: books(id, title, genre, price, author_id) holds Dune/Sci-Fi/9.99/1, Foundation/Sci-Fi/8.5/2, Snow Crash/Cyber/10.5/NULL, Hobbit/Fantasy/6.99/3, Neuromancer/Cyber/7.25/NULL; authors holds 1 Herbert, 2 Asimov, 3 Tolkien.
What does `SELECT COUNT(*), COUNT(author_id) FROM books;` return?

**Answer:** 5 and 3

COUNT(*) counts all 5 rows; COUNT(author_id) skips the two NULLs.

**Q162.** Use this data for the next questions: books(id, title, genre, price, author_id) holds Dune/Sci-Fi/9.99/1, Foundation/Sci-Fi/8.5/2, Snow Crash/Cyber/10.5/NULL, Hobbit/Fantasy/6.99/3, Neuromancer/Cyber/7.25/NULL; authors holds 1 Herbert, 2 Asimov, 3 Tolkien.
What does `SELECT genre, COUNT(*) FROM books GROUP BY genre HAVING COUNT(*) >= 2 ORDER BY genre;` return?

**Answer:** Cyber 2 and Sci-Fi 2

HAVING filters groups after aggregation, dropping Fantasy (1 book).

**Q163.** Use this data for the next questions: books(id, title, genre, price, author_id) holds Dune/Sci-Fi/9.99/1, Foundation/Sci-Fi/8.5/2, Snow Crash/Cyber/10.5/NULL, Hobbit/Fantasy/6.99/3, Neuromancer/Cyber/7.25/NULL; authors holds 1 Herbert, 2 Asimov, 3 Tolkien.
Which books come back from a LEFT JOIN of books to authors filtered with `WHERE a.id IS NULL`?

**Answer:** Neuromancer and Snow Crash

The two books with a NULL author_id have no matching author row.

**Q164.** Use this data for the next questions: books(id, title, genre, price, author_id) holds Dune/Sci-Fi/9.99/1, Foundation/Sci-Fi/8.5/2, Snow Crash/Cyber/10.5/NULL, Hobbit/Fantasy/6.99/3, Neuromancer/Cyber/7.25/NULL; authors holds 1 Herbert, 2 Asimov, 3 Tolkien.
How many rows does `SELECT * FROM books WHERE author_id = NULL;` return?

**Answer:** 0

Comparing to NULL with = is never true; you need IS NULL.

**Q165.** Use this data for the next questions: books(id, title, genre, price, author_id) holds Dune/Sci-Fi/9.99/1, Foundation/Sci-Fi/8.5/2, Snow Crash/Cyber/10.5/NULL, Hobbit/Fantasy/6.99/3, Neuromancer/Cyber/7.25/NULL; authors holds 1 Herbert, 2 Asimov, 3 Tolkien.
What does `SELECT title FROM books WHERE price BETWEEN 7 AND 9 ORDER BY title;` return?

**Answer:** Foundation and Neuromancer

BETWEEN is inclusive: 8.5 and 7.25 qualify; 9.99 and 6.99 do not.

**Q166.** Use this data for the next questions: books(id, title, genre, price, author_id) holds Dune/Sci-Fi/9.99/1, Foundation/Sci-Fi/8.5/2, Snow Crash/Cyber/10.5/NULL, Hobbit/Fantasy/6.99/3, Neuromancer/Cyber/7.25/NULL; authors holds 1 Herbert, 2 Asimov, 3 Tolkien.
What does `SELECT title FROM books ORDER BY price DESC LIMIT 2;` return?

**Answer:** Snow Crash and Dune

Sorted by price descending, the top two are 10.5 and 9.99.

**Q167.** Use this data for the next questions: books(id, title, genre, price, author_id) holds Dune/Sci-Fi/9.99/1, Foundation/Sci-Fi/8.5/2, Snow Crash/Cyber/10.5/NULL, Hobbit/Fantasy/6.99/3, Neuromancer/Cyber/7.25/NULL; authors holds 1 Herbert, 2 Asimov, 3 Tolkien.
In `RANK() OVER (PARTITION BY genre ORDER BY price DESC)`, what rank does Neuromancer get?

**Answer:** 2

Within Cyber, Snow Crash (10.5) ranks 1 and Neuromancer (7.25) ranks 2.

**Q168.** Use this data for the next questions: books(id, title, genre, price, author_id) holds Dune/Sci-Fi/9.99/1, Foundation/Sci-Fi/8.5/2, Snow Crash/Cyber/10.5/NULL, Hobbit/Fantasy/6.99/3, Neuromancer/Cyber/7.25/NULL; authors holds 1 Herbert, 2 Asimov, 3 Tolkien.
What does `SELECT SUM(CASE WHEN price > 8 THEN 1 ELSE 0 END) FROM books;` return?

**Answer:** 3

Dune, Foundation and Snow Crash are above 8.

**Q169.** A report must list every customer, including those with no orders. Which join do you use?

**Answer:** LEFT JOIN from customers to orders

LEFT JOIN keeps unmatched left rows, filling NULLs for the right side.

**Q170.** Your LEFT JOIN with `WHERE d.active = 1` silently drops employees whose department is inactive or missing. What is the fix?

**Answer:** Move the condition into the ON clause

A WHERE on the right table turns the join into an inner join; put the filter in ON.

**Q171.** A bank transfer must debit one account and credit another, or do neither. What do you use?

**Answer:** A transaction with BEGIN ... COMMIT

A transaction makes the two updates atomic; ROLLBACK undoes both on failure.

**Q172.** In SQLite, a book row references author 999, which doesn't exist, and no error appears. What did you forget?

**Answer:** PRAGMA foreign_keys = ON

SQLite parses foreign keys but only enforces them when the pragma is on for the connection.

**Q173.** A query filtering `WHERE email = ?` on a 5-million-row table runs a full SCAN. What is the most direct fix?

**Answer:** CREATE INDEX on email

An index turns the SCAN into a SEARCH ... USING INDEX.

**Q174.** You have an index on (dept, last_name). Which query can NOT use it effectively?

**Answer:** WHERE last_name = 'Smith'

The leftmost-prefix rule: filtering only on the second column can't use the index.

**Q175.** Login code builds `SELECT ... WHERE username = '` + input + `'`. A user types `admin'--`. What happens?

**Answer:** The quote closes the string and -- comments out the rest, bypassing the password check

This is SQL injection; parameterized queries pass the input as data instead.

**Q176.** 2,000 INSERTs run much faster when wrapped in one transaction instead of committing each. Why?

**Answer:** Each COMMIT forces a durable disk flush, so one transaction flushes once

Batching writes is the highest-leverage performance fix.

**Q177.** A table stores customer name, email and product price on every order row, and an email change leaves two versions. Which problem is this and what is the cure?

**Answer:** An update anomaly; normalize so each fact is stored once

Normalization stores each fact in one place, so one update stays consistent.

**Q178.** When should you choose a view?

**Answer:** To save a complex join/aggregate under a name so everyone reuses one definition

A view stores the query, not the data; it runs fresh each time.

**Q179.** What does a trigger `AFTER UPDATE OF stock ON products` let you do?

**Answer:** Automatically write an audit row recording OLD and NEW stock

Triggers run automatically on data changes, with OLD and NEW row references.

**Q180.** You need to run dashboards over years of order history without slowing the live ordering system. Which design fits?

**Answer:** A separate OLAP star schema (fact and dimension tables) loaded in batches

OLTP is for fast small writes; OLAP star schemas are for large scans and aggregation.

## Practical coding questions

### Top spender per genre

Given a `books` table (title, genre, price), write a query that returns each genre with its highest price, ordered by genre. Use the sample data from the questions above.

Starter code:

```
CREATE TABLE books (id INTEGER PRIMARY KEY, title TEXT, genre TEXT, price REAL);
INSERT INTO books (title, genre, price) VALUES
  ('Dune','Sci-Fi',9.99),('Foundation','Sci-Fi',8.5),('Snow Crash','Cyber',10.5),
  ('Hobbit','Fantasy',6.99),('Neuromancer','Cyber',7.25);
-- write your SELECT here
```

Expected output:

```text
Cyber|10.5
Fantasy|6.99
Sci-Fi|9.99
```

??? success "Show a solution"

    ```
    SELECT genre, MAX(price) FROM books GROUP BY genre ORDER BY genre;
    ```

    GROUP BY genre makes one group per genre and MAX(price) takes the highest price in each.

### Books with no author

Using `books` and `authors` (id, name), list the titles of books that have no matching author, ordered by title, using a LEFT JOIN.

Starter code:

```
CREATE TABLE authors (id INTEGER PRIMARY KEY, name TEXT);
CREATE TABLE books (id INTEGER PRIMARY KEY, title TEXT, author_id INTEGER);
INSERT INTO authors VALUES (1,'Herbert'),(2,'Asimov');
INSERT INTO books (title, author_id) VALUES ('Dune',1),('Foundation',2),('Snow Crash',NULL),('Neuromancer',NULL);
-- write your SELECT here
```

Expected output:

```text
Neuromancer
Snow Crash
```

??? success "Show a solution"

    ```
    SELECT b.title FROM books b LEFT JOIN authors a ON a.id = b.author_id WHERE a.id IS NULL ORDER BY b.title;
    ```

    LEFT JOIN keeps every book; the unmatched ones have a NULL author id, which you test with IS NULL.

---

More free practice: the [SQL Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
