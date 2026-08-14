# 06 · NoSQL vs SQL

"NoSQL" isn't one technology — it's an umbrella for document stores
(MongoDB), key-value stores (Redis), wide-column stores (Cassandra), and
graph databases (Neo4j), each trading away parts of the relational model
for something else: flexible schema, horizontal scale, or a data shape
that fits their problem better. This module compares the trade-offs
conceptually, and uses SQLite's JSON support (Level 3) to show concretely
where "schema-flexible" and "relational" meet in the middle.

## The core trade-off

| | SQL (relational) | NoSQL (document store, e.g. MongoDB) |
|---|---|---|
| Schema | Fixed, enforced by `CREATE TABLE` | Flexible — each document can have different fields |
| Relationships | First-class — `JOIN` across normalized tables | Usually denormalized — related data embedded in one document |
| Consistency | Strong (ACID transactions) is the default | Varies — many default to eventual consistency for scale |
| Query language | SQL — declarative, standardized-ish | Varies per database — often a JSON-based query API |
| Horizontal scaling | Harder — traditionally scales up (bigger server) | Often designed to scale out (more servers) from day one |
| Best fit | Data with real relationships, need for consistency | High write volume, evolving schema, denormalized reads |

## Schema flexibility, shown with JSON in SQLite

```sql
CREATE TABLE customers (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    profile TEXT NOT NULL  -- JSON blob, schema-flexible like a document store
);

INSERT INTO customers (name, profile) VALUES
    ('Priya', '{"tier":"gold","preferences":{"newsletter":true},"tags":["vip"]}'),
    ('Marco', '{"tier":"silver","tags":["new"]}');

SELECT name, json_extract(profile, '$.tier') AS tier FROM customers;
```

```text
name   tier
-----  ------
Priya  gold
Marco  silver
```

Priya's profile has a `preferences` object that Marco's doesn't — no schema
migration needed to add fields per-row, which is exactly the flexibility a
document store like MongoDB offers as a first-class feature (every document
in a collection can have different fields, with no `ALTER TABLE` required).
The difference is that here it's one `TEXT` column holding JSON as an
escape hatch inside an otherwise relational table — the norm, not the
exception, in most SQL schemas that need some flexible fields alongside
structured ones.

## Where relational SQL wins: real joins

```sql
CREATE TABLE orders (id INTEGER PRIMARY KEY, customer_id INTEGER REFERENCES customers(id), amount REAL);
INSERT INTO orders (customer_id, amount) VALUES (1, 50), (1, 30), (2, 90);

SELECT c.name, SUM(o.amount) FROM customers c
JOIN orders o ON o.customer_id = c.id
GROUP BY c.id;
```

```text
name   SUM(o.amount)
-----  -------------
Priya  80.0
Marco  90.0
```

This is a one-line query because the data is normalized — `orders`
references `customers` by ID, and the database itself guarantees that
reference is valid (`REFERENCES customers(id)`). A document database
without native joins typically either embeds each customer's orders inside
their document (fine until an order needs its own independent lifecycle —
refunds, status changes, being queried across all customers) or requires
the application to fetch customers, fetch their orders separately, and join
them in code. Neither is wrong, but it's work SQL's relational model does
for you, backed by real constraint enforcement.

## When document/NoSQL genuinely wins

- **Write-heavy, loosely-structured data at huge scale** — logging,
  event streams, sensor data — where enforcing a rigid schema up front
  costs more than it's worth, and horizontal scaling across many cheap
  servers matters more than complex queries.
- **Data that's naturally document-shaped** — a user's full profile with
  nested, variable settings — where you almost always read/write the whole
  document as a unit and rarely need to query *across* documents by their
  internal structure.
- **Key-value access patterns** — a cache, a session store — where you
  always fetch by a single key and the "query language" is really just
  "get" and "set."
- **Massive horizontal scale is a hard requirement from day one** —
  systems like Cassandra are built around partitioning data across many
  nodes as the primary design goal, something traditional single-writer
  relational databases (SQLite very much included) don't do at all.

## When relational SQL wins

- **Data has real relationships that need to stay consistent** —
  financial transactions, inventory, anything where "these two facts must
  never disagree" matters (an order should never reference a deleted
  customer).
- **You need ad-hoc queries across the data's structure** — "which
  customers in Germany spent over $500 last quarter, broken down by
  category" is a `JOIN` + `GROUP BY` away in SQL; the document-store
  equivalent often means either an aggregation pipeline that mirrors SQL's
  complexity anyway, or restructuring your documents around the queries you
  expect to run.
- **Strong consistency (ACID) is a requirement, not a nice-to-have** —
  transactions across multiple rows/tables with guaranteed all-or-nothing
  behavior are relational databases' foundational strength.

## The realistic answer: most systems use both

A typical production system might use Postgres for orders and billing
(needs consistency and joins), Redis for session/cache data (needs raw
speed, key-value access), and Elasticsearch for full-text search (needs
relevance ranking across large text). "SQL vs NoSQL" is rarely a
one-database-for-everything decision in practice — it's picking the right
tool per data shape and access pattern, the same instinct that led SQLite
itself to bolt on JSON support (this module) and FTS5 (Level 3) rather than
forcing every use case through pure relational tables.

## Cheat sheet

| Question | Leans SQL | Leans NoSQL |
|---|---|---|
| Do records reference each other? | Yes → relational joins | No, mostly standalone documents |
| Does the schema change often, per-record? | No, fairly fixed | Yes, frequently and per-document |
| Do you need multi-row ACID transactions? | Yes | Often not required |
| Is horizontal scale across many servers a hard requirement? | Not the default strength | Often designed in from the start |
| Do you need ad-hoc queries across many fields? | Yes, that's SQL's strength | Harder — often needs a specialized query layer |

## Exercise

1. Take the `customers`/`orders` example above and sketch what the same
   data would look like as two MongoDB-style JSON documents — one
   customer, with orders embedded as a nested array. What breaks if an
   order needs to be independently queried across all customers?
2. Using SQLite's JSON functions, write a query that finds every customer
   whose `profile` has `"tier": "gold"` — this is the SQL-plus-JSON hybrid
   approach in action.
3. Describe a real system (pick one you know or imagine one) that would
   plausibly use both a relational database and a document/key-value store
   together, and explain which data goes where and why.
