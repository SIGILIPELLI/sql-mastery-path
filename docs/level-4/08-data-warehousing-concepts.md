# 08 · Data Warehousing Concepts

An OLTP (online transaction processing) database is optimized for fast,
small, frequent read/write operations — "record this order," "update this
row." A data warehouse (OLAP — online analytical processing) is optimized
for the opposite: large scans across historical data to answer "how much
revenue by quarter, by category." The schema shape that supports OLAP well
— the **star schema** — is different on purpose, and this module builds
one directly, runnable in SQLite.

## OLTP vs OLAP, side by side

| | OLTP (Level 3's normalized schema) | OLAP (data warehouse) |
|---|---|---|
| Typical query | "Get this customer's current orders" | "Total revenue by region, last 8 quarters" |
| Schema shape | Normalized — minimize redundancy, fast writes | Denormalized (star schema) — optimized for scanning and aggregation |
| Data freshness | Live, up-to-the-second | Often loaded in batches (hourly/daily ETL) |
| Row volume touched per query | Few | Many — often the whole table or large ranges |
| Write pattern | Frequent, small | Bulk loads, rare updates to historical facts |

Level 3's normalized `customers`/`orders`/`order_items` schema is a
textbook OLTP design. This module builds the OLAP counterpart for the same
kind of data.

## Star schema: one fact table, several dimension tables

```sql
CREATE TABLE dim_customer (
    customer_key INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    country TEXT NOT NULL,
    segment TEXT NOT NULL
);

CREATE TABLE dim_product (
    product_key INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    category TEXT NOT NULL
);

CREATE TABLE dim_date (
    date_key INTEGER PRIMARY KEY,  -- YYYYMMDD
    full_date TEXT NOT NULL,
    year INTEGER NOT NULL,
    quarter INTEGER NOT NULL,
    month INTEGER NOT NULL
);

CREATE TABLE fact_sales (
    id INTEGER PRIMARY KEY,
    customer_key INTEGER REFERENCES dim_customer(customer_key),
    product_key INTEGER REFERENCES dim_product(product_key),
    date_key INTEGER REFERENCES dim_date(date_key),
    quantity INTEGER NOT NULL,
    revenue REAL NOT NULL
);
```

`fact_sales` is the **fact table** — one row per measurable event (a sale),
holding numeric measures (`quantity`, `revenue`) and foreign keys out to
each **dimension table** (`dim_customer`, `dim_product`, `dim_date`), which
each describe one axis you'd want to slice by. Drawn out, the fact table
sits in the middle with dimension tables surrounding it — that shape is
literally why it's called a "star" schema.

## The date dimension — a deliberately denormalized table

```sql
INSERT INTO dim_date VALUES
    (20250301, '2025-03-01', 2025, 1, 3),
    (20250415, '2025-04-15', 2025, 2, 4);
```

Notice `year` and `quarter` are stored as plain columns, even though both
are derivable from `full_date` — a normalized OLTP schema would compute
them on the fly (`strftime('%Y', full_date)`). A warehouse pre-computes and
stores them instead, because OLAP queries filter and group by them
constantly, and recomputing `strftime` across millions of fact rows on
every query is exactly the cost a warehouse is built to avoid. This is
denormalization used deliberately — the trade-off flagged back in Level
3's normalization module, now applied on purpose.

## Loading and querying

```sql
INSERT INTO dim_customer VALUES (1, 'Amara Osei', 'GH', 'Enterprise'), (2, 'Bo Lindqvist', 'SE', 'SMB');
INSERT INTO dim_product VALUES (1, 'Widget', 'Electronics'), (2, 'SQL Book', 'Books');
INSERT INTO fact_sales (customer_key, product_key, date_key, quantity, revenue) VALUES
    (1, 1, 20250301, 3, 77.97),
    (1, 2, 20250301, 1, 19.99),
    (2, 1, 20250415, 2, 51.98);
```

```sql
SELECT dd.year, dd.quarter, dp.category, SUM(fs.revenue) AS revenue
FROM fact_sales fs
JOIN dim_date dd ON dd.date_key = fs.date_key
JOIN dim_product dp ON dp.product_key = fs.product_key
GROUP BY dd.year, dd.quarter, dp.category;
```

```text
year  quarter  category     revenue
----  -------  -----------  -------
2025  1        Books        19.99
2025  1        Electronics  77.97
2025  2        Electronics  51.98
```

This is the shape a business dashboard actually wants: revenue broken down
by time period and category, in one pass. Every dimension is joined by its
surrogate key (`date_key`, `product_key`) — cheap integer joins — rather
than joining on natural keys like the actual date string.

## ETL — how the warehouse actually gets its data

A warehouse doesn't write directly the way an OLTP system does; it's
populated by an **ETL** (Extract, Transform, Load) or **ELT** pipeline:

1. **Extract** — pull raw data from the OLTP source (the live `orders`
   table from Level 3's project).
2. **Transform** — reshape it: join `orders` + `order_items` + `products`
   into flat sale-event rows, look up or generate the surrogate keys for
   each dimension, compute `revenue = quantity * price`.
3. **Load** — bulk-insert the transformed rows into `fact_sales` and
   upsert into the dimension tables.

This typically runs on a schedule (nightly is common) rather than in
real time, which is why warehouse data is usually described as "as of last
night's load" rather than live — an acceptable trade for the query speed
gained by not touching the live transactional system for every analytics
query.

## Why not just query the OLTP database for reports?

Two reasons this module's approach (Level 3's project queries directly
against the normalized schema) doesn't scale to a real warehouse workload:

1. **Contention** — a heavy analytical scan competing for the same tables
   and locks as live transaction processing can slow down or block the
   production system taking real orders.
2. **Shape mismatch** — normalized OLTP schemas need many joins to answer
   analytical questions; a star schema answers the same question with
   fewer, cheaper joins against pre-aggregated, denormalized dimensions.

## Cheat sheet

| Term | Meaning |
|---|---|
| Fact table | One row per event, holds numeric measures + foreign keys to dimensions |
| Dimension table | Describes one axis to slice by (customer, product, date) |
| Surrogate key | An artificial integer key (`customer_key`) used for fast joins, distinct from any natural/business key |
| Star schema | One fact table, several dimension tables, no dimension-to-dimension joins |
| Snowflake schema | A star schema where dimensions are further normalized into sub-dimensions |
| ETL / ELT | The batch pipeline that populates a warehouse from OLTP sources |
| Grain | What one fact row represents (e.g. "one line item," not "one order") — defines what you can and can't aggregate to |

## Exercise

1. Add a `dim_store` dimension and a `store_key` column to `fact_sales`,
   then write a query for revenue by store and quarter.
2. Using the `fact_sales`/`dim_*` tables above, write a query for total
   quantity sold per customer segment (`dim_customer.segment`).
3. Explain, in your own words, why `dim_date` stores `year` and `quarter`
   as plain integer columns instead of computing them from `full_date` at
   query time, connecting it back to the OLTP-vs-OLAP trade-off from the
   top of this module.
