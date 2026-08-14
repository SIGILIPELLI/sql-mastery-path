# 07 · JSON in SQL

SQLite ships a built-in JSON1 extension, enabled by default in modern
builds, that lets you store semi-structured data in a `TEXT` column and
still query into it with SQL — extracting fields, filtering on nested
values, exploding arrays into rows, and rebuilding JSON from query results.

## Sample schema

```sql
CREATE TABLE events (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    payload TEXT NOT NULL
);

INSERT INTO events (name, payload) VALUES
    ('signup', '{"user":{"name":"Priya","age":29},"source":"referral","tags":["vip","beta"]}'),
    ('purchase', '{"user":{"name":"Marco","age":34},"amount":59.99,"tags":["repeat"]}');
```

`payload` is stored as plain text — SQLite doesn't have a dedicated JSON
column type, it just validates and parses the text on demand when a JSON
function touches it.

## Extracting a field: json_extract

```sql
SELECT id, name, json_extract(payload, '$.user.name') AS user_name FROM events;
```

```text
id  name      user_name
--  --------  ---------
1   signup    Priya
2   purchase  Marco
```

The `'$.user.name'` path walks into the nested `user` object and pulls out
`name`. `$` refers to the whole document; dots descend into objects.

## The shorthand: the ->> operator

```sql
SELECT id, payload ->> '$.user.age' AS age FROM events;
```

```text
id  age
--  ---
1   29
2   34
```

`->>` is shorthand for `json_extract` that also unquotes the result (so a
JSON string comes back as a plain SQL text value rather than a
quoted-JSON-string). The plain `->` operator instead returns the value
still wrapped as JSON — useful when you want to pass the result into
another JSON function.

## Validating JSON before you trust it

```sql
SELECT json_valid('{"a":1}') AS ok, json_valid('{a:1}') AS bad;
```

```text
ok  bad
--  ---
1   0
```

`{a:1}` isn't valid JSON (keys must be quoted strings), so `json_valid`
returns `0`. Since SQLite stores JSON as ordinary text with no schema
enforcement, nothing stops an `INSERT` from putting malformed JSON in a
column meant to hold it — check `json_valid()` in a `CHECK` constraint if
you need to guarantee well-formed data at write time.

## Exploding an array into rows: json_each

```sql
SELECT e.id, j.value AS tag
FROM events e, json_each(e.payload, '$.tags') j;
```

```text
id  tag
--  ------
1   vip
1   beta
2   repeat
```

`json_each` is a table-valued function — it turns the JSON array at
`$.tags` into one row per element, and the comma join (`events e,
json_each(...)`) applies it once per source row. This is the standard way
to filter or aggregate on values buried inside a JSON array, something
`json_extract` alone can't do since it returns a single value, not rows.

## Modifying JSON: json_set

```sql
SELECT json_set(payload, '$.user.age', 30) FROM events WHERE id = 1;
```

```text
{"user":{"name":"Priya","age":30},"source":"referral","tags":["vip","beta"]}
```

`json_set` returns a *new* JSON string with the given path replaced — it
doesn't mutate the stored row. To persist the change you still need an
ordinary `UPDATE events SET payload = json_set(payload, '$.user.age', 30)
WHERE id = 1`.

## Building JSON from query results: json_group_array

```sql
SELECT json_group_array(name) FROM events;
```

```text
["signup","purchase"]
```

`json_group_array` is an aggregate that collects a column's values across
rows into a JSON array — useful for producing an API-ready JSON response
directly from SQL. `json_group_object(key_col, value_col)` does the same
for key/value pairs.

## The trap: JSON columns aren't indexed by default

An index on `payload` only helps you find rows by the *entire* text value.
To make `WHERE payload ->> '$.user.name' = 'Priya'` fast, you need an
expression index on that specific path:

```sql
CREATE INDEX idx_events_user_name ON events(payload ->> '$.user.name');
```

Without it, every JSON-path filter forces a full table scan with a
`json_extract` call on every row — fine for small tables, a real cost at
scale.

## Cheat sheet

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

## Exercise

Using the `events` table above:

1. Write a query that returns every event where `$.user.age` is over 30.
2. Write a query using `json_each` that counts how many times each tag
   appears across all events.
3. Create an expression index on `payload ->> '$.source'` and confirm with
   `EXPLAIN QUERY PLAN` that a filter on that path now uses the index.
