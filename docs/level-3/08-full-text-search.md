# 08 · Full-Text Search

`WHERE body LIKE '%database%'` works, but it can't rank results by
relevance, can't handle "close to this phrase," and forces a full scan on
every query since `LIKE` with a leading `%` can't use a normal index.
SQLite's **FTS5** extension is a virtual table type built specifically for
searching text — it indexes words, ranks matches, and supports phrase and
prefix queries.

## Creating an FTS5 table

```sql
CREATE VIRTUAL TABLE articles USING fts5(title, body);

INSERT INTO articles (title, body) VALUES
    ('Getting started with SQLite', 'SQLite is a lightweight embedded database engine that requires no server.'),
    ('Indexing strategies', 'A good index can turn a full table scan into a fast lookup.'),
    ('JSON support in SQLite', 'SQLite has built-in JSON1 functions for querying nested data.');
```

`CREATE VIRTUAL TABLE ... USING fts5(...)` looks like a normal table but is
backed by an inverted index under the hood — SQLite manages the index
structure for you; you just insert rows and query with `MATCH`.

## Basic search: MATCH

```sql
SELECT title FROM articles WHERE articles MATCH 'database';
```

```text
title
----------------------------
Getting started with SQLite
```

Only the row whose `body` actually contains the word "database" matches —
unlike `LIKE '%database%'`, this is a real word-index lookup, not a
character scan, and it correctly ignores "SQLite" appearing in the other
rows' bodies since it doesn't contain that word.

## Ranking results: the built-in `rank` column

```sql
SELECT title, rank FROM articles WHERE articles MATCH 'SQLite' ORDER BY rank;
```

```text
title                         rank
----------------------------  ---------------------
JSON support in SQLite        -1.39280575539568e-06
Getting started with SQLite   -1.36626676076217e-06
```

Every FTS5 table exposes a hidden `rank` column using the BM25 relevance
algorithm — more negative is more relevant (yes, that's the convention;
`ORDER BY rank` without `DESC` gives you best-match-first). Here "JSON
support in SQLite" ranks slightly higher because "SQLite" makes up a larger
share of its shorter body text.

## Highlighting matches: snippet()

```sql
SELECT title, snippet(articles, 1, '[', ']', '...', 8)
FROM articles WHERE articles MATCH 'index';
```

```text
title                snippet(articles, 1, '[', ']', '...', 8)
-------------------  ------------------------------------------
Indexing strategies  A good [index] can turn a full table...
```

`snippet(table, column_index, start_mark, end_mark, ellipsis, max_tokens)`
returns a trimmed excerpt around the match with the search term wrapped in
your chosen markers (`[` `]` here — typically `<b>` `</b>` for HTML) — the
standard way to render "...found in this context..." style search results
instead of dumping the full row.

## Phrase search

```sql
SELECT title FROM articles WHERE articles MATCH '"full table"';
```

```text
title
-------------------
Indexing strategies
```

Quoting the phrase requires the words to appear adjacent and in that
order — "full table" matches, but a row that just happened to contain both
words far apart would not.

## Prefix search

```sql
SELECT title FROM articles WHERE articles MATCH 'embed*';
```

```text
title
----------------------------
Getting started with SQLite
```

`embed*` matches any word starting with "embed" — here, "embedded" in the
first row's body. Handy for autocomplete-style search without needing the
user to type a complete word.

## Ranking with an explicit function: bm25()

```sql
SELECT title, bm25(articles) FROM articles
WHERE articles MATCH 'SQLite'
ORDER BY bm25(articles);
```

```text
title                         bm25(articles)
----------------------------  ---------------------
JSON support in SQLite        -1.39280575539568e-06
Getting started with SQLite   -1.36626676076217e-06
```

`bm25(articles)` is the explicit function backing the `rank` column shown
earlier — calling it directly lets you pass per-column weights, e.g.
`bm25(articles, 10.0, 1.0)` to weight matches in `title` ten times more
heavily than matches in `body`.

## The trap: FTS5 tables are separate from your real data

FTS5 is its own virtual table, not an index bolted onto an existing table —
if `articles` also needs a normal `id`/foreign-key relationship to other
tables, you typically keep a regular table for the structured columns and a
*separate* FTS5 table (often named `articles_fts`) just for the searchable
text, kept in sync via triggers on `INSERT`/`UPDATE`/`DELETE` of the source
table. Search results then give you a rowid you join back to the real table
for the rest of the data.

## Cheat sheet

| Task | Syntax |
|---|---|
| Create an FTS5 table | `CREATE VIRTUAL TABLE t USING fts5(col1, col2)` |
| Search | `SELECT * FROM t WHERE t MATCH 'word'` |
| Relevance order | `ORDER BY rank` (ascending = best match first) |
| Explicit BM25 with column weights | `bm25(t, weight1, weight2, ...)` |
| Highlighted excerpt | `snippet(t, col_idx, start, end, ellipsis, max_tokens)` |
| Phrase match | `MATCH '"exact phrase"'` |
| Prefix match | `MATCH 'prefix*'` |
| Keep FTS in sync with a real table | Triggers on the source table's INSERT/UPDATE/DELETE |

## Exercise

Using the `articles` table above:

1. Write a query that searches for `'index OR embedded'` and explain what
   boolean operators FTS5's `MATCH` syntax supports (`AND`, `OR`, `NOT`).
2. Add a fourth article and write a trigger-based sketch (in words or SQL)
   for how you'd keep a separate `articles_fts` table in sync if `articles`
   were a normal table with an `id` primary key.
3. Use `snippet()` to produce a highlighted excerpt for a search on `'JSON'`,
   using `**` as both the start and end marker instead of `[` `]`.
