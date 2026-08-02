# 01 · Advanced Joins

Level 1 covered `INNER JOIN` and `LEFT JOIN` between two tables. Real schemas
need more: joining a table to itself, chaining several joins together, and
knowing exactly where to put a filter so a `LEFT JOIN` doesn't quietly turn
into an `INNER JOIN`. These patterns show up constantly in reporting queries
and org-chart-style data.

## Sample schema

```sql
CREATE TABLE employees (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    manager_id INTEGER,
    department TEXT,
    FOREIGN KEY (manager_id) REFERENCES employees(id)
);

INSERT INTO employees (name, manager_id, department) VALUES
    ('Alice', NULL, 'Executive'),
    ('Bob', 1, 'Engineering'),
    ('Carol', 1, 'Sales'),
    ('Dave', 2, 'Engineering'),
    ('Erin', 2, 'Engineering'),
    ('Frank', 3, 'Sales');
```

## Self-joins — a table joined to itself

Each employee's `manager_id` points back at another row in the *same* table.
To show a manager's name next to each employee, join `employees` to itself
using two aliases:

```sql
SELECT e.name AS employee, m.name AS manager
FROM employees AS e
LEFT JOIN employees AS m ON e.manager_id = m.id;
```

```text
employee  manager
--------  -------
Alice     NULL
Bob       Alice
Carol     Alice
Dave      Bob
Erin      Bob
Frank     Carol
```

`LEFT JOIN` (not `INNER JOIN`) is essential here — Alice has no manager
(`manager_id IS NULL`), and an `INNER JOIN` would silently drop her from the
results entirely. Self-joins always need two distinct aliases (`e` and `m`
above); without them, SQL can't tell which copy of `employees` a column
reference belongs to.

## The WHERE-vs-ON trap

This is the single most common `LEFT JOIN` bug. Add a `departments` table
with an `active` flag:

```sql
CREATE TABLE departments (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    active INTEGER NOT NULL DEFAULT 1
);

INSERT INTO departments (name, active) VALUES
    ('Engineering', 1),
    ('Sales', 0),
    ('Marketing', 1);
```

Suppose you want every employee, plus their department's row *if* that
department is active. Putting the `active` check in `WHERE` looks reasonable
but is wrong:

```sql
-- WRONG: filters out unmatched rows, defeating the LEFT JOIN
SELECT e.name, d.name AS department
FROM employees AS e
LEFT JOIN departments AS d ON e.department = d.name
WHERE d.active = 1;
```

```text
name  department
----  -----------
Bob   Engineering
Dave  Engineering
Erin  Engineering
```

Alice, Carol, and Frank vanished. Here's why: `LEFT JOIN` first produces rows
for every employee (filling `NULL` where there's no matching department), but
`WHERE d.active = 1` then runs *after* the join and throws away any row where
`d.active` is `NULL` (unmatched departments) or `0` — which discards exactly
the "keep everything on the left" rows the `LEFT JOIN` was meant to preserve.

The fix: put the extra condition in the `ON` clause instead, so it's applied
*during* the join, not after:

```sql
-- RIGHT: condition lives inside the join itself
SELECT e.name, d.name AS department
FROM employees AS e
LEFT JOIN departments AS d ON e.department = d.name AND d.active = 1;
```

```text
name   department
-----  -----------
Alice  NULL
Bob    Engineering
Carol  NULL
Dave   Engineering
Erin   Engineering
Frank  NULL
```

Now every employee still appears; Carol and Frank (Sales, which is inactive)
just get `NULL` for department instead of disappearing. **Rule of thumb:**
conditions on the *right-hand* (optional) table belong in `ON`; conditions on
the *left-hand* (required) table are safe in `WHERE`.

## RIGHT JOIN and FULL OUTER JOIN

SQLite added native `RIGHT JOIN` and `FULL OUTER JOIN` support in version
3.39 (2022). If your SQLite is older, or you're on a database that lacks
`FULL OUTER JOIN` (MySQL, notably), you can emulate it — see below.

```sql
SELECT e.name AS employee, d.name AS department
FROM employees AS e
RIGHT JOIN departments AS d ON e.department = d.name;
```

```text
employee  department
--------  -----------
Bob       Engineering
Carol     Sales
Dave      Engineering
Erin      Engineering
Frank     Sales
NULL      Marketing
```

`RIGHT JOIN` is the mirror image of `LEFT JOIN`: it keeps every row from the
right-hand table (`departments`), filling `NULL` on the left where there's no
matching employee — here, nobody is in Marketing yet.

```sql
SELECT e.name AS employee, d.name AS department
FROM employees AS e
FULL OUTER JOIN departments AS d ON e.department = d.name;
```

```text
employee  department
--------  -----------
Alice     NULL
Bob       Engineering
Carol     Sales
Dave      Engineering
Erin      Engineering
Frank     Sales
NULL      Marketing
```

`FULL OUTER JOIN` keeps everything from both sides — Alice (Executive, no
matching department) and Marketing (a department, no matching employee) both
appear. On engines without native `FULL OUTER JOIN`, emulate it with a
`LEFT JOIN` plus a `RIGHT JOIN`-flavored `LEFT JOIN` combined via `UNION`
(which also de-duplicates any rows that appear in both):

```sql
SELECT e.name AS employee, d.name AS department
FROM employees AS e
LEFT JOIN departments AS d ON e.department = d.name
UNION
SELECT e.name AS employee, d.name AS department
FROM departments AS d
LEFT JOIN employees AS e ON e.department = d.name;
```

## Joining more than two tables

Chain joins by adding another `JOIN` clause — SQLite evaluates them in order,
each one working against the row set built by the ones before it:

```sql
SELECT e.name AS employee, m.name AS manager, d.name AS department
FROM employees AS e
LEFT JOIN employees AS m ON e.manager_id = m.id
LEFT JOIN departments AS d ON e.department = d.name;
```

Each additional join can only add columns or fan out rows (if the join
condition matches more than one row on the other side) — it never removes
rows that a preceding `LEFT JOIN` already preserved, as long as you keep
using `LEFT JOIN` throughout the chain.

## Cheat sheet

| Pattern | What it's for |
|---------|----------------|
| Self-join (`t AS a JOIN t AS b`) | Relating rows in a table to other rows in the same table (managers, parent categories) |
| Filter in `ON` | Keeps unmatched left-side rows even when the filter targets the right-side table |
| Filter in `WHERE` on a `LEFT JOIN`'d table | Silently degrades the `LEFT JOIN` into an `INNER JOIN` |
| `RIGHT JOIN` | Keep everything from the right table (or swap table order and use `LEFT JOIN`) |
| `FULL OUTER JOIN` | Keep everything from both tables; emulate with `LEFT JOIN ... UNION ... LEFT JOIN` if unsupported |

## Exercise

Using the `employees` and `departments` schema above:

1. Write a self-join that lists every employee alongside their manager's
   department (not their own).
2. Write a `LEFT JOIN` between `employees` and `departments` that keeps all
   employees but only matches *active* departments — putting the condition
   in the correct clause.
3. Write a query that finds departments with zero employees currently
   assigned to them (hint: start from `departments`, join to `employees`,
   and check for `NULL` on the employee side).
