---
description: "Setup & First Queries — CREATE TABLE, INSERT, and SELECT are all covered in depth in later modules — this is just enough to confirm your setup works end…"
---

# 01 · Setup & First Queries

## 🎥 Video walkthrough

<iframe width="100%" height="400" style="max-width:720px;aspect-ratio:16/9;height:auto;" src="https://www.youtube.com/embed/0YmNvwWD1vE" title="Video walkthrough" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

SQL (Structured Query Language) is how you talk to a relational database:
create tables, insert data, and ask questions of it. This course uses
**SQLite** via the `sqlite3` command-line tool — no server to install, no
accounts, no network. A SQLite database is a single file on disk, and every
query you run here works (with small dialect notes) against Postgres, MySQL,
and SQL Server too.

## Installing sqlite3

```bash
# macOS (Homebrew) -- also ships pre-installed on macOS
brew install sqlite

# Ubuntu/Debian
sudo apt install sqlite3

# Windows: download the "sqlite-tools" zip from https://sqlite.org/download.html
```

Verify the install:

```bash
sqlite3 --version
# 3.4x.x 2024-... (version varies)
```

## Opening a database

```bash
sqlite3 school.db
```

If `school.db` doesn't exist yet, SQLite creates it the moment you write data
to it (opening the file alone doesn't create anything on disk). You're now in
the `sqlite3` shell, which accepts two kinds of input:

- **Dot-commands** — shell-specific, no semicolon, e.g. `.tables`, `.quit`.
- **SQL statements** — end with a semicolon `;`, e.g. `SELECT 1;`.

## Useful dot-commands

```text
.help                 -- list all dot-commands
.tables               -- list tables in the current database
.schema students      -- show the CREATE TABLE statement for a table
.mode column          -- pretty-print query results in aligned columns
.headers on           -- show column names above results
.quit                 -- exit the shell
```

Turn on column mode and headers right away — the default output is hard to
read:

```text
sqlite> .mode column
sqlite> .headers on
```

## Your first table and query

```sql
CREATE TABLE students (
    id INTEGER PRIMARY KEY,
    name TEXT,
    grade INTEGER
);

INSERT INTO students (id, name, grade) VALUES
    (1, 'Amara', 9),
    (2, 'Ben', 10),
    (3, 'Chidi', 9);

SELECT * FROM students;
```

```text
id  name   grade
--  -----  -----
1   Amara  9
2   Ben    10
3   Chidi  9
```

`CREATE TABLE`, `INSERT`, and `SELECT` are all covered in depth in later
modules — this is just enough to confirm your setup works end to end.

## Comments and formatting

```sql
-- a single-line comment

/* a
   multi-line
   comment */

SELECT name
FROM students     -- keywords are case-insensitive, but UPPERCASE is convention
WHERE grade = 9;
```

SQL statements can span multiple lines; the semicolon (not the newline) is
what ends a statement. Indentation and line breaks are purely for humans — the
database ignores whitespace.

## Running SQL from a file

For anything longer than a one-off query, keep your SQL in a `.sql` file and
run it non-interactively:

```bash
sqlite3 school.db < setup.sql
sqlite3 school.db ".read setup.sql"     # equivalent, from inside the shell
```

## Exiting and reopening

```text
sqlite> .quit
```

Because the database is a plain file, closing the shell doesn't lose your
data — reopen the same file and everything you created is still there:

```bash
sqlite3 school.db
sqlite> SELECT * FROM students;
```

## How It Actually Works

When you run `sqlite3 mydb.db`, SQLite doesn't start a server process — it
opens the file directly in your process's address space and memory-maps its
header. The file is organized into fixed-size **pages** (default 4096 bytes).
Page 1 holds the database header (magic string, page size, schema version,
text encoding) plus the start of the `sqlite_master` table, which is itself
just an ordinary B-tree table that happens to store the schema (your `CREATE
TABLE` statements as text). Every `.tables` or `.schema` command is really a
`SELECT` against that table. When you type a query, SQLite doesn't execute it
directly against the text — it compiles it into **bytecode** for an internal
virtual machine (VDBE, the Virtual Database Engine). Even a query as simple
as `SELECT 1;` produces a handful of opcodes (`Init`, `Integer`, `ResultRow`,
`Halt`) that you can see by prefixing a query with `EXPLAIN`. There is no
client/server round trip, no network stack, and no connection pool — the
"connection" is just an in-process handle (`sqlite3*`) wrapping open file
descriptors and a page cache in RAM.

## 🔀 See this in another language

- [Shell/Bash — Setup & First Script](https://sigilipelli.github.io/shell-mastery-path/level-1/01-setup/)
- [C — Setup & First Program](https://sigilipelli.github.io/c-mastery-path/level-1/01-setup/)
- [PHP — Setup & First Program](https://sigilipelli.github.io/php-mastery-path/level-1/01-setup/)

## Exercise

1. Create a database file called `practice.db`.
2. Turn on `.mode column` and `.headers on`.
3. Create a table `books` with columns `id`, `title`, and `year`.
4. Insert three rows of your choosing.
5. Run `SELECT * FROM books;` and confirm the output looks right.
6. Quit and reopen `practice.db` to confirm the data persisted.
