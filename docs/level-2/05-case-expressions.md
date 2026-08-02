# 05 · CASE Expressions

`CASE` is SQL's inline if/else — it evaluates conditions and returns a value,
right inside a `SELECT`, `WHERE`, or `ORDER BY` clause. It's how you turn raw
data into human-readable labels, bucket values into ranges, or build
pivot-style summaries without leaving SQL.

## Sample schema

```sql
CREATE TABLE orders (
    id INTEGER PRIMARY KEY,
    amount REAL NOT NULL,
    status_code INTEGER NOT NULL
);

INSERT INTO orders (amount, status_code) VALUES
    (15.00, 1),
    (85.00, 2),
    (250.00, 3),
    (40.00, 1),
    (600.00, 2);
```

## Searched CASE — arbitrary conditions

The most common form checks a different condition per branch, evaluated top
to bottom, stopping at the first match:

```sql
SELECT id, amount,
    CASE
        WHEN amount < 50 THEN 'small'
        WHEN amount < 200 THEN 'medium'
        ELSE 'large'
    END AS order_tier
FROM orders;
```

```text
id  amount  order_tier
--  ------  ----------
1   15.0    small
2   85.0    medium
3   250.0   large
4   40.0    small
5   600.0   large
```

Order matters: `WHEN amount < 200` only gets checked for rows that already
failed `amount < 50`, so it effectively means "between 50 and 200" without
needing to write that range explicitly. Reversing the branch order would
change the result — always put the narrowest or most specific condition
first.

## Simple CASE — comparing one expression against several values

When every branch compares the *same* column against a specific value, the
simple form is more compact:

```sql
SELECT id, status_code,
    CASE status_code
        WHEN 1 THEN 'pending'
        WHEN 2 THEN 'shipped'
        WHEN 3 THEN 'delivered'
        ELSE 'unknown'
    END AS status_label
FROM orders;
```

```text
id  status_code  status_label
--  -----------  -------------
1   1            pending
2   2            shipped
3   3            delivered
4   1            pending
5   2            shipped
```

This is equivalent to `CASE WHEN status_code = 1 THEN ... WHEN status_code =
2 THEN ...`, just shorter when it's always the same column being tested.

## Missing ELSE means NULL

```sql
SELECT id, status_code,
    CASE status_code
        WHEN 1 THEN 'pending'
        WHEN 2 THEN 'shipped'
    END AS status_label
FROM orders;
```

```text
id  status_code  status_label
--  -----------  -------------
1   1            pending
2   2            shipped
3   3            NULL
4   1            pending
5   2            shipped
```

Order 3 has `status_code = 3`, which matches no `WHEN` branch, and there's no
`ELSE` — so the whole expression evaluates to `NULL` rather than raising an
error. This is easy to miss in testing if your sample data happens to cover
every case; always add an explicit `ELSE` (even `ELSE 'unknown'`) so unmapped
values are visibly flagged instead of silently becoming `NULL`.

## Conditional aggregation — pivoting with SUM(CASE ...)

Combining `CASE` with an aggregate function is the standard way to build a
pivot-style summary — one row, one column per category — without a dedicated
`PIVOT` clause (which SQLite, and standard SQL generally, doesn't have):

```sql
SELECT
    SUM(CASE WHEN amount < 50 THEN 1 ELSE 0 END) AS small_count,
    SUM(CASE WHEN amount >= 50 AND amount < 200 THEN 1 ELSE 0 END) AS medium_count,
    SUM(CASE WHEN amount >= 200 THEN 1 ELSE 0 END) AS large_count
FROM orders;
```

```text
small_count  medium_count  large_count
-----------  ------------  -----------
2            1             2
```

Each `CASE` contributes `1` to its `SUM` when the condition matches and `0`
otherwise — so each `SUM(CASE ...)` becomes a conditional counter. This
pattern (sometimes written `SUM(CASE WHEN cond THEN amount ELSE 0 END)` to
total a value instead of counting rows) is one of the most useful tricks in
reporting SQL: it turns several separate filtered queries into a single pass
over the data.

## CASE in ORDER BY — custom sort order

```sql
SELECT id, status_code
FROM orders
ORDER BY CASE status_code WHEN 3 THEN 0 ELSE 1 END, id;
```

```text
id  status_code
--  -----------
3   3
1   1
2   2
4   1
5   2
```

Here, delivered orders (`status_code = 3`) are pulled to the front regardless
of their `id`, because the `CASE` maps them to sort key `0` and everything
else to `1`; the trailing `, id` breaks ties within each group. This is the
standard way to express a sort order that doesn't match any column's natural
alphabetical or numeric ordering — like "pending, then shipped, then
delivered" instead of alphabetical.

## Cheat sheet

| Form | Syntax | When to use |
|------|--------|-------------|
| Searched `CASE` | `CASE WHEN cond1 THEN a WHEN cond2 THEN b ELSE c END` | Different conditions per branch, ranges |
| Simple `CASE` | `CASE col WHEN v1 THEN a WHEN v2 THEN b END` | Same column compared to several exact values |
| No `ELSE` | — | Unmatched rows become `NULL` — always add `ELSE` deliberately |
| `SUM(CASE WHEN cond THEN 1 ELSE 0 END)` | — | Conditional counting / pivot-style summaries |
| `CASE` in `ORDER BY` | — | Custom sort order that isn't alphabetical/numeric |

## Exercise

Using the `orders` table above:

1. Write a searched `CASE` that labels orders as `'refund risk'` when
   `amount > 500`, `'watch'` when `amount > 100`, and `'normal'` otherwise.
2. Write a query using conditional aggregation to count how many orders fall
   into each `status_code`, all in a single row (one column per status).
3. Write a query that sorts orders so `status_code = 1` (pending) always
   comes last, everything else in its normal numeric order.
