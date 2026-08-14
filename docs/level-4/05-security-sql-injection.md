# 05 · Security & SQL Injection

SQL injection happens when untrusted input is concatenated directly into a
SQL string instead of being passed as data. This module demonstrates a real
injection against a real login query, then the fix — parameterized
queries — proven against the exact same attack string.

## Sample schema

```python
import sqlite3

conn = sqlite3.connect(":memory:")
conn.execute("CREATE TABLE users (id INTEGER PRIMARY KEY, username TEXT, password TEXT, is_admin INTEGER)")
conn.executemany("INSERT INTO users VALUES (?,?,?,?)", [
    (1, 'alice', 'hunter2', 0),
    (2, 'admin', 's3cret', 1),
])
conn.commit()
```

## The vulnerable version: string concatenation

```python
def login_vulnerable(username, password):
    query = f"SELECT * FROM users WHERE username = '{username}' AND password = '{password}'"
    print("QUERY:", query)
    return conn.execute(query).fetchall()

print(login_vulnerable("alice", "hunter2"))
```

```text
QUERY: SELECT * FROM users WHERE username = 'alice' AND password = 'hunter2'
[(1, 'alice', 'hunter2', 0)]
```

Looks fine with normal input. The problem is that `username` and `password`
become literal SQL text, not values — anything a caller passes in becomes
part of the query itself.

## The exploit, actually run

```python
print(login_vulnerable("admin'--", "wrongpassword"))
```

```text
QUERY: SELECT * FROM users WHERE username = 'admin'--' AND password = 'wrongpassword'
[(2, 'admin', 's3cret', 1)]
```

The attacker logged in as `admin` **without knowing the password.**
`admin'--` closes the `username` string early with `'`, then `--` starts a
SQL comment that swallows the rest of the line — including the entire
`AND password = '...'` check. The database receives, and executes:

```sql
SELECT * FROM users WHERE username = 'admin'--' AND password = 'wrongpassword'
```

Everything after `--` is a comment, so the password check never happens at
all. This is the classic "comment out the rest of the query" injection —
one of dozens of shapes injection can take (others: `' OR '1'='1` to match
every row, `'; DROP TABLE users;--` to run a second destructive statement).

## The fix: parameterized queries

```python
def login_safe(username, password):
    query = "SELECT * FROM users WHERE username = ? AND password = ?"
    return conn.execute(query, (username, password)).fetchall()

print(login_safe("admin'--", "wrongpassword"))
```

```text
[]
```

Same malicious input, empty result — login correctly rejected. The `?`
placeholders tell SQLite's driver "these values are data, never SQL
syntax" — the `'` and `--` in the attacker's string are treated as literal
characters to compare against, not as characters that alter the query
structure. This is true regardless of database — Postgres uses `%s` or
`$1`, MySQL uses `%s`, SQLite uses `?` — the mechanism (send SQL and values
separately, let the driver bind them) is universal and is *the* fix for
injection, not an optional best practice.

## Why "just escape the quotes" isn't the fix

A tempting-but-wrong fix is manually escaping quotes (replacing `'` with
`''`) before concatenating. This is fragile: it's easy to miss an edge
case (backslash handling, encoding tricks, numeric fields that don't get
quoted at all so `1 OR 1=1` needs no quote-escaping whatsoever), and every
place in the codebase that builds SQL has to remember to do it correctly,
every time. Parameterization removes the entire class of bug instead of
patching individual symptoms of it.

## Other SQL-level security practices

- **Least privilege** — a web app's database user should only have the
  permissions it actually needs (no `DROP TABLE` rights for a read-mostly
  reporting account); SQLite itself has no user/permission system since
  it's embedded, but the *application* connecting to it should still run
  with restricted file permissions on the `.db` file.
- **Never build identifiers (table/column names) from user input**, even
  with parameters — `?` placeholders only work for values, not for table or
  column names, since those are part of the query's structure. If a table
  name must be dynamic, validate it against a fixed allow-list before use.
- **Don't leak query errors to end users** — the error message from a
  failed query (as printed above) is exactly what an attacker uses to
  refine an injection attempt; log detailed errors server-side, show users
  a generic message.
- **Validate input types at the boundary** — expecting an integer ID and
  receiving `"1; DROP TABLE users"` should fail type validation before it
  ever reaches a query, as a second layer behind parameterization.

## Cheat sheet

| Practice | Vulnerable | Safe |
|---|---|---|
| Building a query | f-string / string concatenation with input | `?` placeholders + `execute(query, params)` |
| Table/column names from input | Concatenated directly | Validate against an allow-list — parameters can't cover identifiers |
| Error handling | Query errors shown to the user | Generic user-facing message, detailed log server-side |
| DB account permissions | App connects as an admin/superuser | App connects with only the privileges it needs |
| Input validation | Trust the input's shape | Validate type/format before it reaches SQL |

## Exercise

1. Using the `login_vulnerable` function above, craft an injection string
   for `username` that would return **every** row in `users` regardless of
   password (hint: make the `WHERE` clause always true).
2. Rewrite a hypothetical `search_products(keyword)` function that
   currently does `f"SELECT * FROM products WHERE name LIKE '%{keyword}%'"`
   to use a parameterized query instead, keeping the `LIKE` wildcard
   behavior intact.
3. Explain why parameterizing a table name (e.g. `conn.execute("SELECT *
   FROM ?", (table_name,))`) doesn't work, and describe the allow-list
   approach you'd use instead if the table genuinely needs to be dynamic.
