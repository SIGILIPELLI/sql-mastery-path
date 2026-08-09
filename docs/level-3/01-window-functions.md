# 01 · Window Functions

A window function computes a value across a set of rows *related to the
current row* — a "window" — without collapsing those rows into one, the way
`GROUP BY` does. Each input row keeps its own line in the output, but gets an
extra column computed from its neighbors: its rank within a group, a running
total, the value from the previous row, and so on. This is the tool for
"top N per category," running totals, and period-over-period comparisons —
things `GROUP BY` alone cannot express because it only produces one row per
group.

## Sample schema

```sql
CREATE TABLE sales (
    id INTEGER PRIMARY KEY,
    salesperson TEXT NOT NULL,
    region TEXT NOT NULL,
    amount REAL NOT NULL,
    sale_date TEXT NOT NULL
);

INSERT INTO sales (salesperson, region, amount, sale_date) VALUES
    ('Alice', 'West', 1200.00, '2026-01-05'),
    ('Alice', 'West', 900.00,  '2026-01-12'),
    ('Alice', 'West', 1500.00, '2026-01-20'),
    ('Bob',   'West', 800.00,  '2026-01-08'),
    ('Bob',   'West', 800.00,  '2026-01-15'),
    ('Carol', 'East', 2000.00, '2026-01-03'),
    ('Carol', 'East', 1100.00, '2026-01-18'),
    ('Dave',  'East', 500.00,  '2026-01-22');
```

## The anatomy of `OVER (...)`

Every window function is a normal function followed by `OVER (...)`, which
defines the window:

```sql
SELECT salesperson, region, amount,
       ROW_NUMBER() OVER (PARTITION BY region ORDER BY amount DESC) AS row_num,
       RANK()       OVER (PARTITION BY region ORDER BY amount DESC) AS rnk,
       DENSE_RANK() OVER (PARTITION BY region ORDER BY amount DESC) AS dense_rnk
FROM sales
ORDER BY region, amount DESC;
```

```text
salesperson  region  amount  row_num  rnk  dense_rnk
-----------  ------  ------  -------  ---  ---------
Carol        East    2000.0  1        1    1
Carol        East    1100.0  2        2    2
Dave         East    500.0   3        3    3
Alice        West    1500.0  1        1    1
Alice        West    1200.0  2        2    2
Alice        West    900.0   3        3    3
Bob          West    800.0   4        4    4
Bob          West    800.0   5        4    4
```

`PARTITION BY region` splits the rows into independent windows (one per
region) — each restarts its numbering from 1. `ORDER BY amount DESC` decides
the order *within* each partition. The three ranking functions only differ
when there's a tie, visible on Bob's two `800.0` rows:

- **`ROW_NUMBER()`** always assigns unique, sequential numbers (4, 5) — it
  doesn't care about ties at all.
- **`RANK()`** gives ties the same rank (4, 4) and then **skips** the next
  rank — there is no rank 5 in this partition.
- **`DENSE_RANK()`** gives ties the same rank (4, 4) too, but never skips —
  the next distinct value would get rank 5, not 6.

Picking the wrong one is a real bug: using `ROW_NUMBER()` for "top 3 per
region" arbitrarily breaks ties instead of including all of them, while
`RANK()` can make a "top 3" list return more or fewer than 3 rows if there's
a tie at the boundary.

## Aggregate window functions — a running total

Aggregate functions you already know (`SUM`, `AVG`, `COUNT`, `MIN`, `MAX`)
also work as window functions — the difference from `GROUP BY` is that the
underlying rows aren't collapsed:

```sql
SELECT salesperson, sale_date, amount,
       SUM(amount) OVER (PARTITION BY salesperson ORDER BY sale_date
                         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM sales
WHERE salesperson = 'Alice'
ORDER BY sale_date;
```

```text
salesperson  sale_date   amount  running_total
-----------  ----------  ------  -------------
Alice        2026-01-05  1200.0  1200.0
Alice        2026-01-12  900.0   2100.0
Alice        2026-01-20  1500.0  3600.0
```

`ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` is the **frame clause** —
it says "sum every row from the start of this partition up to and including
the current one," which is exactly a running total.

## Frame clauses — a moving average

The frame doesn't have to start at the beginning. `ROWS BETWEEN 1 PRECEDING
AND 1 FOLLOWING` looks at the current row plus one neighbor on each side — a
3-row moving average:

```sql
SELECT salesperson, sale_date, amount,
       ROUND(AVG(amount) OVER (ORDER BY sale_date
             ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING), 2) AS moving_avg
FROM sales
WHERE region = 'West'
ORDER BY sale_date;
```

```text
salesperson  sale_date   amount  moving_avg
-----------  ----------  ------  ----------
Alice        2026-01-05  1200.0  1000.0
Bob          2026-01-08  800.0   966.67
Alice        2026-01-12  900.0   833.33
Bob          2026-01-15  800.0   1066.67
Alice        2026-01-20  1500.0  1150.0
```

Note there's no `PARTITION BY` here — with none, the whole result set is one
window, ordered by `sale_date` across every salesperson in the West region.
The first and last rows only average 2 values (they have no left/right
neighbor respectively) rather than erroring.

## `LAG` and `LEAD` — looking at neighboring rows

`LAG(col)` reads a column's value from a previous row in the same window;
`LEAD(col)` reads it from a following row. Both are the standard tool for
period-over-period comparisons:

```sql
SELECT salesperson, sale_date, amount,
       LAG(amount) OVER (PARTITION BY salesperson ORDER BY sale_date) AS prev_amount,
       LEAD(amount) OVER (PARTITION BY salesperson ORDER BY sale_date) AS next_amount,
       amount - LAG(amount) OVER (PARTITION BY salesperson ORDER BY sale_date) AS change
FROM sales
WHERE salesperson = 'Alice'
ORDER BY sale_date;
```

```text
salesperson  sale_date   amount  prev_amount  next_amount  change
-----------  ----------  ------  -----------  -----------  ------
Alice        2026-01-05  1200.0               900.0
Alice        2026-01-12  900.0   1200.0       1500.0       -300.0
Alice        2026-01-20  1500.0  900.0                     600.0
```

The first row has no `prev_amount` (nothing precedes it) and the last has no
`next_amount` — both come back `NULL` rather than erroring, so any
arithmetic built on them (like `change`) is also `NULL` at the edges. Both
functions accept an optional second argument, `LAG(amount, 1, 0)`, to supply
a default instead of `NULL`.

## Percent of a group's total

Combining a window aggregate with the row's own value is how you compute
"this row's share of its group" without a self-join:

```sql
SELECT region, salesperson, amount,
       ROUND(100.0 * amount / SUM(amount) OVER (PARTITION BY region), 1) AS pct_of_region
FROM sales
ORDER BY region, amount DESC;
```

```text
region  salesperson  amount  pct_of_region
------  -----------  ------  -------------
East    Carol        2000.0  55.6
East    Carol        1100.0  30.6
East    Dave         500.0   13.9
West    Alice        1500.0  28.8
West    Alice        1200.0  23.1
West    Alice        900.0   17.3
West    Bob          800.0   15.4
West    Bob          800.0   15.4
```

`SUM(amount) OVER (PARTITION BY region)` — with no `ORDER BY` inside the
`OVER(...)` — computes the total for the *entire* partition on every row,
which is exactly what's needed as the denominator.

## The default frame trap

Leaving out an explicit frame clause is dangerous the moment you add an
`ORDER BY` to the window, because SQL then silently defaults to `RANGE
BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` instead of covering every row
in the partition:

```sql
SELECT salesperson, region, amount,
       SUM(amount) OVER (PARTITION BY region ORDER BY amount) AS default_frame_sum
FROM sales
WHERE region = 'West'
ORDER BY amount;
```

```text
salesperson  region  amount  default_frame_sum
-----------  ------  ------  -----------------
Bob          West    800.0   1600.0
Bob          West    800.0   1600.0
Alice        West    900.0   2500.0
Alice        West    1200.0  3700.0
Alice        West    1500.0  5200.0
```

This looks like a running total, and mostly is — except it's a **trap**: as
soon as you add `ORDER BY` inside `OVER(...)` without writing an explicit
`ROWS`/`RANGE` frame, the default frame changes from "the whole partition" to
"start of partition through current row," turning what looks like "West's
total" into a running sum instead. Both Bob rows show `1600.0` (tied rows
under the default `RANGE` frame are treated as one peer group), which can
easily be mistaken for "the region total" if you aren't watching for it. The
fix is the same rule from the running-total example above: **if you want the
whole partition's total on every row, either drop `ORDER BY` from the
`OVER(...)` entirely, or write the frame explicitly.**

## Cheat sheet

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

## Exercise

Using the `sales` schema above:

1. Find each salesperson's rank by `amount` within their own region using
   `RANK()`, and explain in your own words why `Bob`'s tie produces the
   result it does.
2. Write a running total of `amount` per region, ordered by `sale_date`,
   using an explicit `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`
   frame.
3. Using `LAG()`, find the change in `amount` between each salesperson's
   consecutive sales, and identify which sale had the biggest drop.
4. Compute each region's total revenue as a window function (no explicit
   `ORDER BY` inside `OVER(...)`) and use it to find what percentage each
   individual sale contributed to its region's total.
