# 04 · String & Date Functions

Real-world data is messy — inconsistent capitalization, stray whitespace,
dates stored as text. SQL's built-in string and date functions let you clean
up, extract, and compute over that data directly in a query instead of
pulling everything into application code first.

## Sample schema

```sql
CREATE TABLE customers (
    id INTEGER PRIMARY KEY,
    full_name TEXT NOT NULL,
    email TEXT NOT NULL,
    signup_date TEXT NOT NULL
);

INSERT INTO customers (full_name, email, signup_date) VALUES
    ('  Alice Nguyen  ', 'ALICE@Example.com', '2025-11-03'),
    ('Bob Smith', 'bob@example.com', '2026-01-15'),
    ('Carol Diaz', 'carol@example.com', '2026-02-20');
```

Note the deliberately messy data: Alice's name has extra spaces and her
email has inconsistent capitalization — exactly the kind of thing these
functions exist to handle.

## String cleanup: TRIM, UPPER, LOWER

```sql
SELECT full_name, TRIM(full_name) AS cleaned, LOWER(email) AS email_lower
FROM customers;
```

```text
full_name         cleaned       email_lower
----------------  ------------  ------------------
  Alice Nguyen    Alice Nguyen  alice@example.com
Bob Smith         Bob Smith     bob@example.com
Carol Diaz        Carol Diaz    carol@example.com
```

`TRIM()` strips leading/trailing whitespace (use `LTRIM`/`RTRIM` for one side
only). `LOWER()`/`UPPER()` normalize case — essential before comparing
user-entered text, since `'Alice'` and `'alice'` are different strings as far
as `=` is concerned in most databases (SQLite string comparison is
case-sensitive by default, aside from ASCII letters in `LIKE`).

## LENGTH and SUBSTR

```sql
SELECT full_name, LENGTH(full_name) AS raw_len, LENGTH(TRIM(full_name)) AS trimmed_len
FROM customers;
```

```text
full_name         raw_len  trimmed_len
----------------  -------  -----------
  Alice Nguyen    16       12
Bob Smith         9        9
Carol Diaz        10       10
```

Alice's untrimmed name is 4 characters longer than it looks — a reminder
that whitespace is invisible but very real to `LENGTH()`.

`SUBSTR(string, start, length)` extracts a portion of a string (1-indexed).
Combined with `INSTR` (find the position of a substring), you can pull out a
first name from a full name:

```sql
SELECT full_name, SUBSTR(TRIM(full_name), 1, INSTR(TRIM(full_name), ' ') - 1) AS first_name
FROM customers;
```

```text
full_name         first_name
----------------  ----------
  Alice Nguyen    Alice
Bob Smith         Bob
Carol Diaz        Carol
```

`INSTR(TRIM(full_name), ' ')` finds the position of the first space (after
trimming), and `SUBSTR` takes everything before it. This is fragile — it
breaks on single-word names or multiple middle names — which is exactly why
production systems usually store first/last name in separate columns rather
than parsing them out of a combined field.

## REPLACE — and a case-sensitivity trap

```sql
SELECT email, REPLACE(email, 'example.com', 'newdomain.com') AS migrated_email
FROM customers;
```

```text
email               migrated_email
------------------  --------------------
ALICE@Example.com   ALICE@Example.com
bob@example.com     bob@newdomain.com
carol@example.com   carol@newdomain.com
```

Alice's email was **not** migrated. `REPLACE()` does an exact, case-sensitive
substring match — it looked for the literal text `example.com`, but Alice's
address has `Example.com` (capital E), so no match was found and the string
came back unchanged with no error or warning. The safe fix is to normalize
case before comparing or replacing:

```sql
SELECT email, LOWER(email) AS normalized
FROM customers
WHERE LOWER(email) LIKE '%example.com%';
```

## Concatenation with `||`

```sql
SELECT TRIM(full_name) || ' <' || LOWER(email) || '>' AS display
FROM customers;
```

```text
display
--------------------------------
Alice Nguyen <alice@example.com>
Bob Smith <bob@example.com>
Carol Diaz <carol@example.com>
```

`||` is the standard SQL string concatenation operator (SQLite, PostgreSQL,
Oracle). MySQL and SQL Server differ: MySQL uses `CONCAT(a, b, c)` by default,
and SQL Server uses `+`. Check your target engine before relying on `||` in
production code.

## Date functions: DATE, STRFTIME, julianday

SQLite stores dates as plain `TEXT` in ISO-8601 format (`YYYY-MM-DD`), which
sorts and compares correctly as a string — no special date type required for
basic use.

```sql
SELECT full_name, signup_date, DATE(signup_date, '+30 days') AS trial_end
FROM customers;
```

```text
full_name         signup_date  trial_end
----------------  -----------  ----------
  Alice Nguyen    2025-11-03   2025-12-03
Bob Smith         2026-01-15   2026-02-14
Carol Diaz        2026-02-20   2026-03-22
```

`DATE(date_string, modifier, ...)` applies one or more modifiers like
`'+30 days'`, `'-1 month'`, or `'start of year'` to compute a new date.

```sql
SELECT full_name, STRFTIME('%Y', signup_date) AS signup_year, STRFTIME('%m', signup_date) AS signup_month
FROM customers;
```

```text
full_name         signup_year  signup_month
----------------  -----------  -------------
  Alice Nguyen    2025         11
Bob Smith         2026         01
Carol Diaz        2026         02
```

`STRFTIME(format, date_string)` formats a date using `strftime`-style
placeholders (`%Y` = 4-digit year, `%m` = 2-digit month, `%d` = day, `%H:%M:%S`
= time). It's SQLite's general-purpose date formatting and extraction tool.
PostgreSQL's equivalent is `TO_CHAR`/`EXTRACT`; MySQL has its own
`DATE_FORMAT`.

```sql
SELECT full_name, signup_date,
       CAST(julianday('2026-03-01') - julianday(signup_date) AS INTEGER) AS days_since_signup
FROM customers;
```

```text
full_name         signup_date  days_since_signup
----------------  -----------  ------------------
  Alice Nguyen    2025-11-03   118
Bob Smith         2026-01-15   45
Carol Diaz        2026-02-20   9
```

`julianday()` converts a date into a continuous day count, so subtracting two
of them gives the number of days between them directly — the standard SQLite
idiom for date arithmetic that a simple string comparison can't do.

## Cheat sheet

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

## Exercise

Using the `customers` table above:

1. Write a query that produces a cleaned, lowercase, trimmed version of every
   email address, and use it to check for duplicate accounts differing only
   by case or whitespace.
2. Write a query using `STRFTIME` that groups customers by signup month and
   counts how many signed up in each.
3. Write a query that flags any customer whose `signup_date` is more than 60
   days before today's date (hint: compare against `DATE('now', '-60 days')`).
