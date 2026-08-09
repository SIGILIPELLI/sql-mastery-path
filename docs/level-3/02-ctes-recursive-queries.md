# 02 · CTEs & Recursive Queries

A Common Table Expression (CTE) is a named, temporary result set defined with
a `WITH` clause and used just like a table for the rest of the query. CTEs
exist to make complex queries readable — instead of nesting subqueries three
levels deep, you name each step and read the query top to bottom. A
**recursive** CTE goes further: it can reference itself, which is the
standard SQL way to walk hierarchical data (org charts, category trees,
folder structures) of unknown depth.

## Sample schema

```sql
CREATE TABLE employees (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    manager_id INTEGER,
    salary REAL NOT NULL,
    FOREIGN KEY (manager_id) REFERENCES employees(id)
);

INSERT INTO employees (id, name, manager_id, salary) VALUES
    (1, 'Grace (CEO)', NULL, 220000),
    (2, 'Heidi (VP Eng)', 1, 180000),
    (3, 'Ivan (VP Sales)', 1, 170000),
    (4, 'Judy (Eng Mgr)', 2, 140000),
    (5, 'Kevin (Engineer)', 4, 110000),
    (6, 'Liam (Engineer)', 4, 105000),
    (7, 'Mia (Sales Mgr)', 3, 130000),
    (8, 'Noah (Sales Rep)', 7, 90000);
```

`manager_id` is a **self-referencing foreign key** — it points to another row
in the same table, which is exactly what makes this data hierarchical rather
than flat.

## A basic CTE

```sql
WITH high_earners AS (
    SELECT id, name, salary
    FROM employees
    WHERE salary > 120000
)
SELECT name, salary FROM high_earners ORDER BY salary DESC;
```

```text
name             salary
---------------  --------
Grace (CEO)      220000.0
Heidi (VP Eng)   180000.0
Ivan (VP Sales)  170000.0
Judy (Eng Mgr)   140000.0
Mia (Sales Mgr)  130000.0
```

This is equivalent to wrapping the same `SELECT` as a subquery in `FROM (...)
AS high_earners`, but the `WITH` form names it *before* the main query, so
you read the intent ("here's what a high earner is") before you read what's
done with it.

## Multiple CTEs, chained together

CTEs can reference each other — this is where they really outperform nested
subqueries, because each step gets its own name instead of another
indentation level:

```sql
WITH dept_size AS (
    SELECT manager_id, COUNT(*) AS num_reports
    FROM employees
    WHERE manager_id IS NOT NULL
    GROUP BY manager_id
),
managers_with_size AS (
    SELECT e.name AS manager_name, ds.num_reports
    FROM dept_size ds
    JOIN employees e ON e.id = ds.manager_id
)
SELECT * FROM managers_with_size ORDER BY num_reports DESC;
```

```text
manager_name     num_reports
---------------  -----------
Grace (CEO)      2
Judy (Eng Mgr)   2
Heidi (VP Eng)   1
Ivan (VP Sales)  1
Mia (Sales Mgr)  1
```

`managers_with_size` references `dept_size` by name, and both are just
`SELECT`s under the same `WITH`, separated by commas — no nesting required.

## Recursive CTEs — walking up a hierarchy

A recursive CTE has two parts, joined by `UNION ALL`: an **anchor** query
(the starting row(s)) and a **recursive** query that references the CTE's
own name, repeatedly joining one more step outward until nothing new
matches. SQLite requires the `RECURSIVE` keyword explicitly:

```sql
WITH RECURSIVE chain_of_command(id, name, manager_id, depth) AS (
    -- anchor: the starting employee
    SELECT id, name, manager_id, 0
    FROM employees
    WHERE name = 'Kevin (Engineer)'

    UNION ALL

    -- recursive step: walk one level up to the manager
    SELECT e.id, e.name, e.manager_id, cc.depth + 1
    FROM employees e
    JOIN chain_of_command cc ON e.id = cc.manager_id
)
SELECT * FROM chain_of_command ORDER BY depth;
```

```text
id  name              manager_id  depth
--  ----------------  ----------  -----
5   Kevin (Engineer)  4           0
4   Judy (Eng Mgr)    2           1
2   Heidi (VP Eng)    1           2
1   Grace (CEO)                   3
```

Each pass through the recursive step takes the *previous* pass's result
(`chain_of_command`, referring to itself) and joins one more level up the
`manager_id` chain. The recursion terminates naturally: once it reaches
Grace, whose `manager_id` is `NULL`, the join finds no matching row and adds
nothing more, so the next iteration produces zero rows and recursion stops.

## Recursive CTEs — walking down a hierarchy

The same technique works in reverse — start at a manager and walk down to
every person under them, at any depth:

```sql
WITH RECURSIVE org_tree(id, name, manager_id, depth) AS (
    SELECT id, name, manager_id, 0
    FROM employees
    WHERE name = 'Heidi (VP Eng)'

    UNION ALL

    SELECT e.id, e.name, e.manager_id, ot.depth + 1
    FROM employees e
    JOIN org_tree ot ON e.manager_id = ot.id
)
SELECT id, name, depth FROM org_tree ORDER BY depth, id;
```

```text
id  name              depth
--  ----------------  -----
2   Heidi (VP Eng)    0
4   Judy (Eng Mgr)    1
5   Kevin (Engineer)  2
6   Liam (Engineer)   2
```

The only difference from the "walk up" version is which side of the join
condition drives the recursion: `e.manager_id = ot.id` walks *down* to direct
reports of the previous level, instead of `e.id = cc.manager_id` walking
*up* to the previous level's manager.

## Recursive CTEs without any table at all

A recursive CTE doesn't need to reference a real table — it can generate a
sequence purely from constants, which is useful for date ranges, counters,
or filling gaps in reporting data:

```sql
WITH RECURSIVE counter(n) AS (
    SELECT 1
    UNION ALL
    SELECT n + 1 FROM counter WHERE n < 5
)
SELECT n FROM counter;
```

```text
n
-
1
2
3
4
5
```

## The infinite-loop trap

The recursive step's `WHERE` clause (or an equivalent stopping condition) is
what makes recursion terminate — leave it out or get the join condition
backwards, and the query can spin close to forever, generating rows until it
hits SQLite's built-in safety limit or exhausts memory:

```sql
-- DANGEROUS: no upper bound, would run until SQLite's internal limit kicks in
-- WITH RECURSIVE counter(n) AS (
--     SELECT 1
--     UNION ALL
--     SELECT n + 1 FROM counter
-- )
-- SELECT * FROM counter;
```

Two other common mistakes to watch for:

- **Using `UNION` instead of `UNION ALL`.** `UNION` deduplicates every row
  before continuing, which is slower and can be outright wrong for
  hierarchies where two different rows legitimately share the same values —
  `UNION ALL` is almost always what you want in a recursive CTE.
- **Cyclic data.** If `manager_id` ever pointed back down the chain (a data
  bug, since a real org chart shouldn't cycle), a "walk up" recursive CTE
  would loop forever even with a correct join, because it would never run
  out of new matching rows. Recursive CTEs trust the data to be acyclic;
  they don't detect cycles for you.

## Cheat sheet

| Concept | Detail |
|---------|--------|
| `WITH name AS (...)` | Defines a non-recursive CTE, usable once in the query that follows |
| `WITH RECURSIVE name(...) AS (...)` | Required keyword in SQLite for a self-referencing CTE |
| Anchor query | The first `SELECT`, before `UNION ALL` — the starting row(s) |
| Recursive query | The second `SELECT`, which references the CTE's own name |
| Termination | Recursion stops automatically once the recursive step returns zero new rows |
| `UNION ALL` vs `UNION` | Always use `UNION ALL` in recursion — `UNION`'s deduplication is wasted work and can hide legitimate duplicate rows |
| Multiple CTEs | Separate with commas: `WITH a AS (...), b AS (...) SELECT ...` |

## Exercise

Using the `employees` schema above:

1. Write a non-recursive CTE that finds the average salary per manager (via
   `GROUP BY manager_id`), then a second CTE that joins it back to
   `employees` to show each manager's name alongside their team's average
   salary.
2. Write a recursive CTE that lists every employee under Ivan (VP Sales),
   at any depth, along with their depth in the hierarchy.
3. Add a `SUM` of `salary` to the "everyone under Heidi" query to find the
   total salary cost of Heidi's entire reporting chain, including Heidi
   herself.
4. Modify the number-sequence example to generate the numbers 10 through 20
   instead of 1 through 5.
