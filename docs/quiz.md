---
title: "Final Quiz"
description: "Test what you have learned across all four levels of SQL Mastery Path with a 20-question final quiz."
---

# Final Quiz

20 questions drawn from all four levels of this course. You need **70% (14 out of 20)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "Use this data for the next questions: books(id, title, genre, price, author_id) holds Dune/Sci-Fi/9.99/1, Foundation/Sci-Fi/8.5/2, Snow Crash/Cyber/10.5/NULL, Hobbit/Fantasy/6.99/3, Neuromancer/Cyber/7.25/NULL; authors holds 1 Herbert, 2 Asimov, 3 Tolkien.\nWhat does `SELECT COUNT(*), COUNT(author_id) FROM books;` return?",
    "options": [
      "5 and 3",
      "3 and 3",
      "5 and 5",
      "3 and 5"
    ],
    "correctIndex": 0,
    "explanation": "COUNT(*) counts all 5 rows; COUNT(author_id) skips the two NULLs."
  },
  {
    "question": "Use this data for the next questions: books(id, title, genre, price, author_id) holds Dune/Sci-Fi/9.99/1, Foundation/Sci-Fi/8.5/2, Snow Crash/Cyber/10.5/NULL, Hobbit/Fantasy/6.99/3, Neuromancer/Cyber/7.25/NULL; authors holds 1 Herbert, 2 Asimov, 3 Tolkien.\nWhat does `SELECT genre, COUNT(*) FROM books GROUP BY genre HAVING COUNT(*) >= 2 ORDER BY genre;` return?",
    "options": [
      "Sci-Fi 2 only",
      "Cyber 2, Fantasy 1 and Sci-Fi 2",
      "Cyber 2 and Sci-Fi 2",
      "Fantasy 1 only"
    ],
    "correctIndex": 2,
    "explanation": "HAVING filters groups after aggregation, dropping Fantasy (1 book)."
  },
  {
    "question": "Use this data for the next questions: books(id, title, genre, price, author_id) holds Dune/Sci-Fi/9.99/1, Foundation/Sci-Fi/8.5/2, Snow Crash/Cyber/10.5/NULL, Hobbit/Fantasy/6.99/3, Neuromancer/Cyber/7.25/NULL; authors holds 1 Herbert, 2 Asimov, 3 Tolkien.\nWhich books come back from a LEFT JOIN of books to authors filtered with `WHERE a.id IS NULL`?",
    "options": [
      "Neuromancer and Snow Crash",
      "Dune and Foundation",
      "Hobbit",
      "All five"
    ],
    "correctIndex": 0,
    "explanation": "The two books with a NULL author_id have no matching author row."
  },
  {
    "question": "Use this data for the next questions: books(id, title, genre, price, author_id) holds Dune/Sci-Fi/9.99/1, Foundation/Sci-Fi/8.5/2, Snow Crash/Cyber/10.5/NULL, Hobbit/Fantasy/6.99/3, Neuromancer/Cyber/7.25/NULL; authors holds 1 Herbert, 2 Asimov, 3 Tolkien.\nHow many rows does `SELECT * FROM books WHERE author_id = NULL;` return?",
    "options": [
      "5",
      "2",
      "3",
      "0"
    ],
    "correctIndex": 3,
    "explanation": "Comparing to NULL with = is never true; you need IS NULL."
  },
  {
    "question": "Use this data for the next questions: books(id, title, genre, price, author_id) holds Dune/Sci-Fi/9.99/1, Foundation/Sci-Fi/8.5/2, Snow Crash/Cyber/10.5/NULL, Hobbit/Fantasy/6.99/3, Neuromancer/Cyber/7.25/NULL; authors holds 1 Herbert, 2 Asimov, 3 Tolkien.\nWhat does `SELECT title FROM books WHERE price BETWEEN 7 AND 9 ORDER BY title;` return?",
    "options": [
      "Foundation only",
      "Dune and Foundation",
      "Hobbit and Neuromancer",
      "Foundation and Neuromancer"
    ],
    "correctIndex": 3,
    "explanation": "BETWEEN is inclusive: 8.5 and 7.25 qualify; 9.99 and 6.99 do not."
  },
  {
    "question": "Use this data for the next questions: books(id, title, genre, price, author_id) holds Dune/Sci-Fi/9.99/1, Foundation/Sci-Fi/8.5/2, Snow Crash/Cyber/10.5/NULL, Hobbit/Fantasy/6.99/3, Neuromancer/Cyber/7.25/NULL; authors holds 1 Herbert, 2 Asimov, 3 Tolkien.\nWhat does `SELECT title FROM books ORDER BY price DESC LIMIT 2;` return?",
    "options": [
      "Hobbit and Neuromancer",
      "Dune and Foundation",
      "Snow Crash and Neuromancer",
      "Snow Crash and Dune"
    ],
    "correctIndex": 3,
    "explanation": "Sorted by price descending, the top two are 10.5 and 9.99."
  },
  {
    "question": "Use this data for the next questions: books(id, title, genre, price, author_id) holds Dune/Sci-Fi/9.99/1, Foundation/Sci-Fi/8.5/2, Snow Crash/Cyber/10.5/NULL, Hobbit/Fantasy/6.99/3, Neuromancer/Cyber/7.25/NULL; authors holds 1 Herbert, 2 Asimov, 3 Tolkien.\nIn `RANK() OVER (PARTITION BY genre ORDER BY price DESC)`, what rank does Neuromancer get?",
    "options": [
      "2",
      "NULL",
      "1",
      "3"
    ],
    "correctIndex": 0,
    "explanation": "Within Cyber, Snow Crash (10.5) ranks 1 and Neuromancer (7.25) ranks 2."
  },
  {
    "question": "Use this data for the next questions: books(id, title, genre, price, author_id) holds Dune/Sci-Fi/9.99/1, Foundation/Sci-Fi/8.5/2, Snow Crash/Cyber/10.5/NULL, Hobbit/Fantasy/6.99/3, Neuromancer/Cyber/7.25/NULL; authors holds 1 Herbert, 2 Asimov, 3 Tolkien.\nWhat does `SELECT SUM(CASE WHEN price > 8 THEN 1 ELSE 0 END) FROM books;` return?",
    "options": [
      "3",
      "5",
      "8",
      "2"
    ],
    "correctIndex": 0,
    "explanation": "Dune, Foundation and Snow Crash are above 8."
  },
  {
    "question": "A report must list every customer, including those with no orders. Which join do you use?",
    "options": [
      "UNION",
      "LEFT JOIN from customers to orders",
      "INNER JOIN",
      "CROSS JOIN"
    ],
    "correctIndex": 1,
    "explanation": "LEFT JOIN keeps unmatched left rows, filling NULLs for the right side."
  },
  {
    "question": "Your LEFT JOIN with `WHERE d.active = 1` silently drops employees whose department is inactive or missing. What is the fix?",
    "options": [
      "Add ORDER BY",
      "Add DISTINCT",
      "Move the condition into the ON clause",
      "Switch to INNER JOIN"
    ],
    "correctIndex": 2,
    "explanation": "A WHERE on the right table turns the join into an inner join; put the filter in ON."
  },
  {
    "question": "A bank transfer must debit one account and credit another, or do neither. What do you use?",
    "options": [
      "A transaction with BEGIN ... COMMIT",
      "An index",
      "A view",
      "Two separate statements"
    ],
    "correctIndex": 0,
    "explanation": "A transaction makes the two updates atomic; ROLLBACK undoes both on failure."
  },
  {
    "question": "In SQLite, a book row references author 999, which doesn't exist, and no error appears. What did you forget?",
    "options": [
      "PRAGMA foreign_keys = ON",
      "A NOT NULL constraint",
      "CREATE INDEX",
      "A view"
    ],
    "correctIndex": 0,
    "explanation": "SQLite parses foreign keys but only enforces them when the pragma is on for the connection."
  },
  {
    "question": "A query filtering `WHERE email = ?` on a 5-million-row table runs a full SCAN. What is the most direct fix?",
    "options": [
      "Use SELECT *",
      "Use a view",
      "Add LIMIT",
      "CREATE INDEX on email"
    ],
    "correctIndex": 3,
    "explanation": "An index turns the SCAN into a SEARCH ... USING INDEX."
  },
  {
    "question": "You have an index on (dept, last_name). Which query can NOT use it effectively?",
    "options": [
      "WHERE last_name = 'Smith'",
      "WHERE dept = 'Eng' AND last_name = 'Smith'",
      "WHERE dept = 'Eng'",
      "WHERE dept = 'Eng' ORDER BY last_name"
    ],
    "correctIndex": 0,
    "explanation": "The leftmost-prefix rule: filtering only on the second column can't use the index."
  },
  {
    "question": "Login code builds `SELECT ... WHERE username = '` + input + `'`. A user types `admin'--`. What happens?",
    "options": [
      "The user is rejected",
      "The table is dropped",
      "The query fails safely",
      "The quote closes the string and -- comments out the rest, bypassing the password check"
    ],
    "correctIndex": 3,
    "explanation": "This is SQL injection; parameterized queries pass the input as data instead."
  },
  {
    "question": "2,000 INSERTs run much faster when wrapped in one transaction instead of committing each. Why?",
    "options": [
      "Each COMMIT forces a durable disk flush, so one transaction flushes once",
      "Inserts become parallel",
      "Transactions skip constraints",
      "Indexes are disabled"
    ],
    "correctIndex": 0,
    "explanation": "Batching writes is the highest-leverage performance fix."
  },
  {
    "question": "A table stores customer name, email and product price on every order row, and an email change leaves two versions. Which problem is this and what is the cure?",
    "options": [
      "An update anomaly; normalize so each fact is stored once",
      "A deadlock; add an index",
      "A trigger bug; drop the trigger",
      "A NULL problem; add DEFAULT"
    ],
    "correctIndex": 0,
    "explanation": "Normalization stores each fact in one place, so one update stays consistent."
  },
  {
    "question": "When should you choose a view?",
    "options": [
      "To store a copy of the data",
      "To replace constraints",
      "To speed up writes",
      "To save a complex join/aggregate under a name so everyone reuses one definition"
    ],
    "correctIndex": 3,
    "explanation": "A view stores the query, not the data; it runs fresh each time."
  },
  {
    "question": "What does a trigger `AFTER UPDATE OF stock ON products` let you do?",
    "options": [
      "Automatically write an audit row recording OLD and NEW stock",
      "Block all updates",
      "Rename a column",
      "Create an index"
    ],
    "correctIndex": 0,
    "explanation": "Triggers run automatically on data changes, with OLD and NEW row references."
  },
  {
    "question": "You need to run dashboards over years of order history without slowing the live ordering system. Which design fits?",
    "options": [
      "More triggers",
      "One normalized table for both",
      "A separate OLAP star schema (fact and dimension tables) loaded in batches",
      "Store everything as JSON"
    ],
    "correctIndex": 2,
    "explanation": "OLTP is for fast small writes; OLAP star schemas are for large scans and aggregation."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### Top spender per genre

Given a `books` table (title, genre, price), write a query that returns each genre with its highest price, ordered by genre. Use the sample data from the questions above.

**Starter code**

```sql
CREATE TABLE books (id INTEGER PRIMARY KEY, title TEXT, genre TEXT, price REAL);
INSERT INTO books (title, genre, price) VALUES
  ('Dune','Sci-Fi',9.99),('Foundation','Sci-Fi',8.5),('Snow Crash','Cyber',10.5),
  ('Hobbit','Fantasy',6.99),('Neuromancer','Cyber',7.25);
-- write your SELECT here
```

**Expected output**

```text
Cyber|10.5
Fantasy|6.99
Sci-Fi|9.99
```

<details><summary>Show solution and explanation</summary>

<pre><code>SELECT genre, MAX(price) FROM books GROUP BY genre ORDER BY genre;</code></pre>

<p>GROUP BY genre makes one group per genre and MAX(price) takes the highest price in each.</p>
</details>

### Books with no author

Using `books` and `authors` (id, name), list the titles of books that have no matching author, ordered by title, using a LEFT JOIN.

**Starter code**

```sql
CREATE TABLE authors (id INTEGER PRIMARY KEY, name TEXT);
CREATE TABLE books (id INTEGER PRIMARY KEY, title TEXT, author_id INTEGER);
INSERT INTO authors VALUES (1,'Herbert'),(2,'Asimov');
INSERT INTO books (title, author_id) VALUES ('Dune',1),('Foundation',2),('Snow Crash',NULL),('Neuromancer',NULL);
-- write your SELECT here
```

**Expected output**

```text
Neuromancer
Snow Crash
```

<details><summary>Show solution and explanation</summary>

<pre><code>SELECT b.title FROM books b LEFT JOIN authors a ON a.id = b.author_id WHERE a.id IS NULL ORDER BY b.title;</code></pre>

<p>LEFT JOIN keeps every book; the unmatched ones have a NULL author id, which you test with IS NULL.</p>
</details>
