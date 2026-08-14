# 05 · Triggers

A trigger is SQL code that runs automatically when a row is inserted,
updated, or deleted — no application code has to remember to call it.
Unlike stored procedures (previous module), **SQLite fully supports
triggers** with real, runnable `CREATE TRIGGER` syntax. This module uses
that native support directly.

## Sample schema

```sql
CREATE TABLE products (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    stock INTEGER NOT NULL,
    updated_at TEXT
);

INSERT INTO products (name, stock) VALUES ('Widget', 50), ('Gadget', 10);

CREATE TABLE stock_log (
    id INTEGER PRIMARY KEY,
    product_id INTEGER NOT NULL,
    old_stock INTEGER,
    new_stock INTEGER,
    changed_at TEXT NOT NULL
);
```

## AFTER trigger — audit log on every stock change

```sql
CREATE TRIGGER trg_stock_audit
AFTER UPDATE OF stock ON products
FOR EACH ROW
BEGIN
    INSERT INTO stock_log (product_id, old_stock, new_stock, changed_at)
    VALUES (OLD.id, OLD.stock, NEW.stock, datetime('now'));

    UPDATE products SET updated_at = datetime('now') WHERE id = NEW.id;
END;

UPDATE products SET stock = stock - 5 WHERE name = 'Widget';

SELECT * FROM products;
```

```text
id  name    stock  updated_at
--  ------  -----  -------------------
1   Widget  45     2026-08-14 04:31:10
2   Gadget  10
```

```sql
SELECT product_id, old_stock, new_stock FROM stock_log;
```

```text
product_id  old_stock  new_stock
----------  ---------  ---------
1           50         45
```

`UPDATE OF stock` scopes the trigger to fire only when the `stock` column
specifically changes — updating `name` alone wouldn't fire it. Inside the
trigger body, `OLD` refers to the row before the change and `NEW` to the row
after; `AFTER` means the trigger body runs once the triggering statement has
already committed its change, so `NEW.stock` in the `INSERT` reflects the
already-updated value. Only `Widget`'s row appears in the log because only
its `stock` was touched — `Gadget` is untouched by this `UPDATE`.

## BEFORE trigger with a guard condition — blocking bad writes

```sql
CREATE TRIGGER trg_no_negative_stock
BEFORE UPDATE OF stock ON products
FOR EACH ROW
WHEN NEW.stock < 0
BEGIN
    SELECT RAISE(ABORT, 'stock cannot go negative');
END;

UPDATE products SET stock = stock - 100 WHERE name = 'Gadget';
```

```text
Runtime error: stock cannot go negative
```

`BEFORE` runs the trigger *before* the row is written, so it can veto the
change. The `WHEN NEW.stock < 0` clause limits the trigger to only the rows
where the incoming value would be invalid — most updates skip the trigger
body entirely. `RAISE(ABORT, 'message')` cancels the triggering statement
and rolls back the change with the given error message; `Gadget`'s stock is
never touched, and the earlier `trg_stock_audit` trigger doesn't fire either
because the `UPDATE` never completes.

## Cascading effects — deleting a parent cleans up children

```sql
CREATE TABLE orders (
    id INTEGER PRIMARY KEY,
    customer TEXT NOT NULL
);
CREATE TABLE order_items (
    id INTEGER PRIMARY KEY,
    order_id INTEGER NOT NULL,
    sku TEXT NOT NULL
);
INSERT INTO orders (customer) VALUES ('Nadia');
INSERT INTO order_items (order_id, sku) VALUES (1, 'A1'), (1, 'B2');

CREATE TRIGGER trg_cascade_delete_items
AFTER DELETE ON orders
FOR EACH ROW
BEGIN
    DELETE FROM order_items WHERE order_id = OLD.id;
END;

DELETE FROM orders WHERE id = 1;
SELECT * FROM order_items;
```

```text
(zero rows)
```

Deleting the order automatically deleted its line items — `OLD.id` inside
an `AFTER DELETE` trigger still refers to the row that just disappeared,
since SQLite captures its values before removal. This is a manual
alternative to `ON DELETE CASCADE` foreign keys; either approach works, but
a trigger makes the cascade logic explicit and lets you add extra steps
(like writing to an audit table) beyond a plain cascade.

## Trap: triggers can fire recursively

By default SQLite does **not** let a trigger's own statements re-fire the
same trigger (`recursive_triggers` is off by default), but a trigger on
table A that modifies table B, which has a trigger that modifies table A
again, *can* create an infinite loop across tables. Always trace what each
trigger writes to before adding another trigger on a table it touches.

## Cheat sheet

| Concept | Syntax | Notes |
|---|---|---|
| Fire after a change | `AFTER INSERT/UPDATE/DELETE ON table` | Row is already written; good for audit logs, cascades |
| Fire before a change | `BEFORE INSERT/UPDATE/DELETE ON table` | Can inspect and reject via `RAISE(ABORT, ...)` |
| Scope to one column | `UPDATE OF col ON table` | Fires only when that column is in the `UPDATE` |
| Old/new row values | `OLD.col` / `NEW.col` | `OLD` unavailable on `INSERT`, `NEW` unavailable on `DELETE` |
| Conditional firing | `WHEN condition` | Skips the body when the condition is false |
| Cancel + roll back | `SELECT RAISE(ABORT, 'msg')` | Only meaningful in `BEFORE` triggers |
| Drop a trigger | `DROP TRIGGER name;` | — |

## Exercise

Using the `products`/`stock_log` schema above:

1. Write an `AFTER INSERT` trigger on `products` that inserts a `stock_log`
   row with `old_stock = NULL` and `new_stock` equal to the initial stock,
   so new products appear in the audit trail too.
2. Write a `BEFORE UPDATE` trigger that prevents `name` from ever being set
   to an empty string, using `RAISE(ABORT, ...)`.
3. Explain, in your own words, why `BEFORE` triggers are the right choice
   for validation/rejection, while `AFTER` triggers are the right choice for
   audit logging and cascades.
