# 03 · Set Operations

Joins combine tables *horizontally* — matching rows side by side into wider
rows. Set operations combine query results *vertically* — stacking the rows
of two `SELECT` statements into one result set, the way you'd combine two
lists. They're the SQL equivalent of set math: union, intersection, and
difference.

## Sample schema

```sql
CREATE TABLE newsletter_subscribers (email TEXT);
CREATE TABLE webinar_signups (email TEXT);

INSERT INTO newsletter_subscribers VALUES
    ('alice@example.com'), ('bob@example.com'), ('carol@example.com');

INSERT INTO webinar_signups VALUES
    ('bob@example.com'), ('carol@example.com'), ('dave@example.com'), ('bob@example.com');
```

Note `bob@example.com` appears twice in `webinar_signups` (he registered for
two sessions) — useful for seeing the difference between `UNION` and
`UNION ALL` below.

## The rule for all set operations

Every `SELECT` combined with a set operator must return **the same number of
columns**, with compatible types in the same positions. Column *names* don't
have to match — the result set uses the first query's column names:

```sql
SELECT email FROM newsletter_subscribers
UNION
SELECT email, 1 FROM webinar_signups;
```

```text
Parse error: SELECTs to the left and right of UNION do not have the same
number of result columns
```

Mismatched column counts fail immediately — there's no implicit padding or
truncation.

## UNION — combine and de-duplicate

```sql
SELECT email FROM newsletter_subscribers
UNION
SELECT email FROM webinar_signups
ORDER BY email;
```

```text
email
------------------
alice@example.com
bob@example.com
carol@example.com
dave@example.com
```

`UNION` stacks both result sets and removes duplicate rows — `bob@example.com`
appears in both source tables (and twice in `webinar_signups`), but shows up
only once here. `ORDER BY` applies to the *combined* result and must come
after the last `SELECT`, not attached to either individual query.

## UNION ALL — combine and keep everything

```sql
SELECT email FROM newsletter_subscribers
UNION ALL
SELECT email FROM webinar_signups
ORDER BY email;
```

```text
email
------------------
alice@example.com
bob@example.com
bob@example.com
bob@example.com
carol@example.com
carol@example.com
dave@example.com
```

`UNION ALL` keeps every row, duplicates included. This matters for two
reasons: it's the only one of the two that gives you an accurate row *count*
(e.g. "total signups across both channels, including repeats"), and it's
significantly faster on large tables — plain `UNION` has to sort or hash the
combined set to find and remove duplicates, while `UNION ALL` just
concatenates. **Default to `UNION ALL` unless you specifically need
de-duplication.**

## INTERSECT — only rows in both

```sql
SELECT email FROM newsletter_subscribers
INTERSECT
SELECT email FROM webinar_signups
ORDER BY email;
```

```text
email
------------------
bob@example.com
carol@example.com
```

`INTERSECT` returns only rows that appear in *both* result sets — here, the
subscribers who are also registered for the webinar. Like `UNION`, it
de-duplicates automatically.

## EXCEPT — rows in the first, not in the second

```sql
SELECT email FROM newsletter_subscribers
EXCEPT
SELECT email FROM webinar_signups
ORDER BY email;
```

```text
email
------------------
alice@example.com
```

`EXCEPT` returns rows from the first query that have **no** match in the
second — subscribers who never signed up for the webinar. Order matters:
swapping the two queries changes the answer (it would instead return webinar
registrants who never subscribed to the newsletter — `dave@example.com`).
Some databases (notably MySQL before 8.0.31, and Oracle) call this operator
`MINUS` instead of `EXCEPT`; SQLite, PostgreSQL, and SQL Server all use
`EXCEPT`.

## Cheat sheet

| Operator | Keeps | De-duplicates? |
|----------|-------|-----------------|
| `UNION` | Rows from either query | Yes |
| `UNION ALL` | Rows from either query | No — keeps duplicates, faster |
| `INTERSECT` | Rows in both queries | Yes |
| `EXCEPT` | Rows in the first query only, absent from the second | Yes |

## Exercise

Using the `newsletter_subscribers`/`webinar_signups` schema above:

1. Write a `UNION ALL` query that counts the total number of *signal
   events* (subscriptions plus webinar registrations, counting Bob's two
   registrations separately) using `COUNT(*)` over the combined result.
2. Write an `EXCEPT` query showing webinar registrants who are not on the
   newsletter list.
3. Add a `source` column to each branch (a literal `'newsletter'` or
   `'webinar'` string) and combine them with `UNION ALL`, so the combined
   result shows which list each email came from and Bob's two webinar
   registrations both survive. Then switch it to plain `UNION` and explain
   why Bob's two `('bob@example.com', 'webinar')` rows collapse into one.
