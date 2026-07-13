# Database-Query-Optimization
# PostgreSQL Query Optimization

A hands-on project demonstrating how to identify slow SQL queries and fix them using
proven optimization techniques in PostgreSQL — indexing, query rewriting, and
`EXPLAIN ANALYZE` diagnostics — with every step measured and benchmarked.

## Why this project

A slow query isn't just annoying — at scale, it can degrade or crash a production
database. Query optimization is one of the highest-leverage skills in data
engineering: the same fix that shaves 200ms off a query on a laptop can be the
difference between a responsive app and a timed-out request in production.

This project works through the full diagnose-and-fix cycle on a dataset large enough
(millions of rows) that the improvements are actually measurable, not theoretical.

## Tech Stack

- **PostgreSQL** — database engine
- **Python** — pandas, SQLAlchemy, psycopg2
- **Jupyter Notebook** — for a step-by-step, documented workflow
- **matplotlib** — before/after benchmark visualization
- **pgAdmin** — optional GUI for browsing tables and query results

## Dataset

Three related tables loaded from CSV into a `customers` / `products` / `orders`
schema:

| Table | Columns | Approx. Rows |
|---|---|---|
| `customers` | id, name, email, country, signup_date | 500,000 |
| `products` | id, name, category, price, stock | 10,000 |
| `orders` | id, customer_id, product_id, amount, order_date, status | 2,000,000 |

`orders.customer_id` and `orders.product_id` are foreign keys referencing
`customers.id` and `products.id`.

## Project Structure

```
.
├── data/
│   ├── customers.csv
│   ├── products.csv
│   └── orders.csv
├── Query_optimization.ipynb      # main notebook - all 7 steps
├── benchmark_results.csv         # exported before/after timing results
├── benchmark_chart.png           # before/after bar chart
└── README.md
```

## Setup

**Prerequisites:** Python 3.10+, PostgreSQL 15+, pgAdmin (optional).

1. Clone this repo and install dependencies:
   ```bash
   pip install psycopg2-binary pandas sqlalchemy matplotlib
   ```
2. Create the database in `psql` or pgAdmin's Query Tool:
   ```sql
   CREATE DATABASE query_optimization;
   ```
3. Place `customers.csv`, `products.csv`, and `orders.csv` in a `data/` folder.
4. Open `Query_optimization.ipynb` in Jupyter and update the connection variables
   in the credentials cell (`DB_USER`, `DB_PASSWORD`, etc.) and the `PROJECT_DIR` /
   `DATA_DIR` paths to match your local setup.
5. Run all cells in order, top to bottom.

> Database credentials in this notebook are for local development only. See
> [Security](#security) below before pushing to a public repo.

## Results

| Query | Before (avg, sec) | After (avg, sec) | Speedup |
|---|---|---|---|
| Country filter (JOIN + filter) | _pending_ | _pending_ | _pending_ |
| Monthly revenue (aggregation) | _pending_ | _pending_ | _pending_ |
| Top customers (JOIN + GROUP BY) | _pending_ | _pending_ | _pending_ |

*(Fill this in from `benchmark_results.csv` once you've run the full notebook —
this is the actual portfolio evidence for the project.)*

![Before vs After Benchmark](benchmark_chart.png)

---

# The 7-Step Roadmap: Code Walkthrough

Every step below includes the actual code from `Query_optimization.ipynb`, with a
line-by-line explanation underneath.

## Step 1: Environment Setup

**Set the working folder**

```python
import os
import sys

PROJECT_DIR = r"C:\Query optimization"

if os.path.isdir(PROJECT_DIR):
    os.chdir(PROJECT_DIR)
    sys.path.insert(0, PROJECT_DIR)
    print("Working directory set to:", os.getcwd())
else:
    print("PROJECT_DIR does not exist yet - create the folder or update the path above.")
    print("Current working directory:", os.getcwd())
```

`os` lets Python interact with the operating system (check folders, list files);
`sys` controls settings of the running Python process itself. `PROJECT_DIR` is
stored as a **raw string** (the `r` prefix) so Windows backslashes aren't
misread as escape codes. `os.chdir()` changes the active working directory so
relative file paths behave predictably, and `sys.path.insert()` makes that folder
importable if custom scripts are added later. The `if/else` just confirms the
folder actually exists before switching to it.

**Load the toolkits**

```python
import time
import pandas as pd
import numpy as np
from sqlalchemy import create_engine, text

pd.set_option("display.max_columns", None)
print("Libraries loaded OK")
```

`pandas` is the core library for working with tabular data (rows/columns) in
Python. `numpy` handles fast numeric operations underneath pandas. `sqlalchemy`
is what lets Python talk to PostgreSQL — `create_engine` builds the connection,
and `text()` wraps raw SQL strings so SQLAlchemy sends them to the database
as-is. `pd.set_option(...)` just tells pandas never to hide columns when
displaying a wide table.

**Database credentials**

```python
DB_USER = "postgres"
DB_PASSWORD = "postgres"
DB_HOST = "localhost"
DB_PORT = "5432"
DB_NAME = "query_optimization"

DB_URL = f"postgresql+psycopg2://{DB_USER}:{DB_PASSWORD}@{DB_HOST}:{DB_PORT}/{DB_NAME}"

engine = create_engine(DB_URL)
print("Engine created for database:", DB_NAME)
```

Credentials are kept as separate variables rather than one hardcoded string, so
any one piece (like the password) can change without touching the rest. The
f-string (`f"..."`) plugs each variable into a standard connection URL format:
`driver://user:password@host:port/database`. `create_engine()` doesn't connect
yet — it just prepares a reusable connection factory used by every later cell.

**Test the connection**

```python
with engine.connect() as conn:
    result = conn.execute(text("SELECT current_database(), version();"))
    row = result.fetchone()
    print("Connected to database:", row[0])
    print("Server version:", row[1])
```

`with ... as conn:` opens a connection and guarantees it closes automatically
afterward, even if an error occurs. The query asks Postgres which database it's
connected to and which version it's running — a direct proof the connection
actually works, not just that the engine object was created. `.fetchone()`
grabs the single result row; `row[0]` / `row[1]` access its two columns.

---

## Step 2: Create Tables and Load Sample Data

**Locate the CSV files**

```python
DATA_DIR = r"C:\Query optimization\Data"

customers_path = os.path.join(DATA_DIR, "customers.csv")
products_path = os.path.join(DATA_DIR, "products.csv")
orders_path = os.path.join(DATA_DIR, "orders.csv")

for p in [customers_path, products_path, orders_path]:
    status = "found" if os.path.exists(p) else "MISSING"
    print(status, "-", p)
```

`os.path.join()` safely combines a folder and filename into a full path without
manual slash-handling bugs. The loop checks each of the three files actually
exists on disk before anything tries to load them.

**Preview before loading**

```python
preview_customers = pd.read_csv(customers_path, nrows=5)
preview_products = pd.read_csv(products_path, nrows=5)
preview_orders = pd.read_csv(orders_path, nrows=5)

print("customers columns:", list(preview_customers.columns))
print("products columns:", list(preview_products.columns))
print("orders columns:", list(preview_orders.columns))

preview_orders
```

`nrows=5` reads only the first 5 rows — a cheap sanity check before committing
to loading a potentially multi-million-row file. `.columns` lists the detected
column names, which is exactly where a mismatch (e.g. a CSV column named
`customer_id` instead of the expected `id`) would be caught early. The final
bare `preview_orders` line uses Jupyter's auto-display feature to render it as
a table.

**Load `customers`**

```python
customers = pd.read_csv(customers_path, parse_dates=["signup_date"])
print("customers loaded:", customers.shape)

customers.to_sql("customers", engine, if_exists="replace", index=False, method="multi", chunksize=5000)
print("customers table written to query_optimization")
```

`parse_dates=["signup_date"]` converts that column from plain text into a real
date type. `.shape` returns `(rows, columns)` as a quick load confirmation.
`.to_sql()` writes the DataFrame to Postgres:
- `if_exists="replace"` drops and recreates the table if it already exists
  (versus `"append"`, which would add on top of existing rows)
- `index=False` skips writing pandas's internal row index as an extra column
- `method="multi"` batches many rows per `INSERT` statement for speed
- `chunksize=5000` caps each batch at 5,000 rows

**Load `products`** follows the identical pattern (no `parse_dates` needed).

**Load `orders` in chunks**

```python
orders_chunksize = 50_000
first_chunk = True
rows_loaded = 0
t0 = time.time()

for chunk in pd.read_csv(orders_path, parse_dates=["order_date"], chunksize=orders_chunksize):
    chunk.to_sql(
        "orders",
        engine,
        if_exists="replace" if first_chunk else "append",
        index=False,
        method="multi",
        chunksize=5000,
    )
    first_chunk = False
    rows_loaded += len(chunk)
    print("Rows loaded so far:", rows_loaded)

print("Done. Total orders rows loaded:", rows_loaded, "in", round(time.time() - t0, 1), "seconds")
```

`orders` is the largest table, so `read_csv(..., chunksize=...)` returns an
iterator that yields 50,000 rows at a time instead of loading the whole file
into memory at once. The key logic is
`if_exists="replace" if first_chunk else "append"` — the **first** batch wipes
and recreates the table; every batch after that **appends**, so the full file
builds up correctly across many small writes. `t0`/`time.time()` measure total
load duration.

**Check row counts**

```python
with engine.connect() as conn:
    counts = {
        "customers": conn.execute(text("SELECT COUNT(*) FROM customers;")).scalar(),
        "products": conn.execute(text("SELECT COUNT(*) FROM products;")).scalar(),
        "orders": conn.execute(text("SELECT COUNT(*) FROM orders;")).scalar(),
    }

expected = {"customers": 500_000, "products": 10_000, "orders": 2_000_000}

for table, expected_count in expected.items():
    actual_count = counts[table]
    status = "OK" if actual_count == expected_count else "MISMATCH"
    print(f"{status} - {table}: expected {expected_count}, got {actual_count}")
```

`COUNT(*)` returns the number of rows in a table; `.scalar()` pulls that single
number directly out of the result. Two dictionaries — actual counts vs.
expected counts — are compared row by row to catch loading problems (like a
table accidentally loaded twice) before moving forward.

**Add primary keys**

```python
with engine.connect() as conn:
    conn.execute(text("ALTER TABLE customers ADD PRIMARY KEY (id);"))
    conn.execute(text("ALTER TABLE products ADD PRIMARY KEY (id);"))
    conn.execute(text("ALTER TABLE orders ADD PRIMARY KEY (id);"))
    conn.commit()

print("Primary keys added on customers, products, orders")
```

`pandas.to_sql()` creates plain columns with no constraints, so primary keys
are added manually here. A primary key enforces uniqueness on `id` and lets
Postgres build a fast lookup structure on it automatically — important since
`id` columns get used constantly in joins. `conn.commit()` makes the structural
change permanent. Note: re-running this cell after it's already succeeded will
error, since a table can only have one primary key.

---

## Step 3: Write and Measure Slow Queries (Baseline)

**A reusable timing function**

```python
def time_query(label, sql, engine, runs=3):
    """Run a query `runs` times and return the average wall-clock time in seconds."""
    times = []
    with engine.connect() as conn:
        for _ in range(runs):
            t0 = time.perf_counter()
            conn.execute(text(sql)).fetchall()
            times.append(time.perf_counter() - t0)
    avg = sum(times) / len(times)
    print(f"{label}: {avg:.3f}s avg over {runs} runs (individual runs: {[round(t,3) for t in times]})")
    return avg
```

Defined once and reused for every timing measurement in the rest of the
notebook. `time.perf_counter()` is a high-precision timer suited for measuring
short code durations. `.fetchall()` forces Postgres to actually return every
matching row, so the timing reflects real work done, not just query dispatch.
The function returns the average across `runs` executions so a single slow/fast
outlier doesn't skew the result.

**The three baseline queries**

```python
SLOW_QUERY_1 = """
SELECT o.id, o.amount, o.order_date, c.name, c.email
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE c.country = 'United States'
ORDER BY o.order_date DESC;
"""

SLOW_QUERY_2 = """
SELECT DATE_TRUNC('month', order_date) AS month,
       SUM(amount) AS revenue,
       COUNT(*) AS num_orders
FROM orders
GROUP BY 1
ORDER BY 1;
"""

SLOW_QUERY_3 = """
SELECT c.name, SUM(o.amount) AS total_spend
FROM customers c
JOIN orders o ON c.id = o.customer_id
GROUP BY c.name
ORDER BY total_spend DESC
LIMIT 10;
"""

baseline_results = {}
baseline_results["Q1 Country Filter"] = time_query("Q1 Before (country filter)", SLOW_QUERY_1, engine)
baseline_results["Q2 Monthly Revenue"] = time_query("Q2 Before (monthly revenue)", SLOW_QUERY_2, engine)
baseline_results["Q3 Top Customers"] = time_query("Q3 Before (top customers)", SLOW_QUERY_3, engine)
```

- **Q1** joins orders to customers and filters by country — a common
  "customer-facing report" pattern.
- **Q2** uses `DATE_TRUNC('month', ...)` to bucket every order into its month,
  then aggregates revenue and order count per month.
- **Q3** joins and groups by customer to find the top 10 by total spend.

These three become the "before" numbers the whole rest of the project compares
against.

---

## Step 4: Read the Query Execution Plan (EXPLAIN ANALYZE)

```python
def explain_query(sql, engine):
    with engine.connect() as conn:
        result = conn.execute(text("EXPLAIN ANALYZE " + sql))
        for row in result:
            print(row[0])
```

`EXPLAIN ANALYZE` doesn't just describe a query plan — it actually executes the
query and reports exactly how Postgres processed it: which scan method was
used, how many rows were expected vs. found, and how long each step took. The
function prints the multi-line plan output row by row.

```python
explain_query(SLOW_QUERY_1, engine)
explain_query(SLOW_QUERY_2, engine)
explain_query(SLOW_QUERY_3, engine)
```

**How to read the output:**
- `Seq Scan` = a full table scan (reads every row) — the main thing indexing
  fixes
- `Index Scan` / `Bitmap Index Scan` = uses an index to jump directly to
  matching rows
- `cost=X..Y` = the planner's internal *estimate*, in arbitrary units — not a
  real time measurement
- `actual time=X..Y ms` = the real, measured time in milliseconds — the number
  worth trusting

---

## Step 5: Create and Test Indexes

**Create the indexes**

```python
with engine.connect().execution_options(isolation_level="AUTOCOMMIT") as conn:
    conn.execute(text("""
        CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_customers_country
        ON customers(country);
    """))
    conn.execute(text("""
        CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_cust_date
        ON orders(customer_id, order_date DESC);
    """))
    conn.execute(text("""
        CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_pending
        ON orders(order_date)
        WHERE status = 'pending';
    """))
    conn.execute(text("""
        CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_products_hash
        ON products USING HASH (id);
    """))

print("Indexes created (or already existed)")
```

`CREATE INDEX CONCURRENTLY` builds an index without locking the table for other
users — standard practice in production. It cannot run inside a transaction,
so `isolation_level="AUTOCOMMIT"` tells SQLAlchemy to execute each statement
immediately rather than batching them. `IF NOT EXISTS` makes the cell safe to
re-run. Four index types are demonstrated:

| Index | Type | Use case |
|---|---|---|
| `idx_customers_country` | B-Tree (default) | Equality/range filters, e.g. `WHERE country = ...` |
| `idx_orders_cust_date` | Composite | Multi-column filters/sorts together |
| `idx_orders_pending` | Partial | Indexes only rows matching a condition — smaller, faster |
| `idx_products_hash` | Hash | Fast exact-match (`=`) lookups only |

**Verify the indexes**

```python
verify_sql = """
SELECT tablename, indexname, indexdef
FROM pg_indexes
WHERE tablename IN ('customers', 'orders', 'products')
ORDER BY tablename, indexname;
"""
with engine.connect() as conn:
    idx_df = pd.read_sql(text(verify_sql), conn)
idx_df
```

`pg_indexes` is a built-in Postgres system view listing every index on every
table. `pd.read_sql()` runs the query and returns the result directly as a
DataFrame in one step.

**Refresh planner statistics**

```python
with engine.connect() as conn:
    conn.execute(text("ANALYZE customers;"))
    conn.execute(text("ANALYZE orders;"))
    conn.execute(text("ANALYZE products;"))
    conn.commit()
```

`ANALYZE` updates Postgres's internal statistics about table size and data
distribution. The query planner relies on these stats to decide whether an
index will actually help — without running `ANALYZE`, a brand-new index might
be ignored simply because the stats are stale.

**Re-check plans and timings**

```python
explain_query(SLOW_QUERY_1, engine)

after_index_results = {}
after_index_results["Q1 Country Filter"] = time_query("Q1 After index", SLOW_QUERY_1, engine)
after_index_results["Q2 Monthly Revenue"] = time_query("Q2 After index", SLOW_QUERY_2, engine)
after_index_results["Q3 Top Customers"] = time_query("Q3 After index", SLOW_QUERY_3, engine)
```

Same queries, same functions, run again now that indexes exist — this is the
direct "before vs. after indexing" comparison.

---

## Step 6: Rewrite Query for Performance

**SELECT * + subquery vs. JOIN + named columns**

```python
SLOW_REWRITE = """
SELECT * FROM orders
WHERE customer_id IN (
    SELECT id FROM customers WHERE country = 'United States'
)
ORDER BY order_date DESC;
"""

FAST_REWRITE = """
SELECT o.id, o.amount, o.order_date, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE c.country = 'United States'
ORDER BY o.order_date DESC;
"""

rewrite_before = time_query("Rewrite - before (SELECT * + IN subquery)", SLOW_REWRITE, engine)
rewrite_after = time_query("Rewrite - after (JOIN + named columns)", FAST_REWRITE, engine)
print(f"Speedup: {rewrite_before / rewrite_after:.1f}x faster")
```

`SELECT *` pulls every column, including ones not needed. `IN (subquery)`
first runs an inner query, then filters the outer query against that result —
often less efficient than an equivalent `JOIN`, which Postgres can optimize as
a single combined operation. `FAST_REWRITE` fixes both: named columns only,
and a `JOIN` instead of a subquery.

**Rewriting with a CTE**

```python
CTE_QUERY = """
WITH customer_orders AS (
    SELECT customer_id, SUM(amount) AS total_spend
    FROM orders
    GROUP BY customer_id
)
SELECT c.name, co.total_spend
FROM customer_orders co
JOIN customers c ON c.id = co.customer_id
ORDER BY co.total_spend DESC
LIMIT 10;
"""

cte_time = time_query("Q3 rewritten as CTE", CTE_QUERY, engine)
```

`WITH customer_orders AS (...)` defines a **CTE** — a temporary, named result
set usable later in the same query as if it were a real table. Here, spend is
summed per customer first, then joined to `customers` for names. The benefit
is mainly readability — breaking a dense query into clearly separated logical
steps — though it can also help the planner in some cases.

---

## Step 7: Benchmark and Document Improvements

```python
FAST_QUERY_1 = FAST_REWRITE
FAST_QUERY_2 = SLOW_QUERY_2  # same query, now backed by the orders indexes
FAST_QUERY_3 = CTE_QUERY

benchmark_rows = []

for label, before_sql, after_sql in [
    ("Q1 Country Filter", SLOW_QUERY_1, FAST_QUERY_1),
    ("Q2 Monthly Revenue", SLOW_QUERY_2, FAST_QUERY_2),
    ("Q3 Top Customers", SLOW_QUERY_3, FAST_QUERY_3),
]:
    before_time = time_query(f"{label} - before", before_sql, engine, runs=5)
    after_time = time_query(f"{label} - after", after_sql, engine, runs=5)
    speedup = before_time / after_time if after_time > 0 else float("inf")
    benchmark_rows.append({
        "query": label,
        "before_sec": round(before_time, 4),
        "after_sec": round(after_time, 4),
        "speedup_x": round(speedup, 1),
    })

benchmark_df = pd.DataFrame(benchmark_rows)
benchmark_df
```

Each query pair runs 5 times (`runs=5`) and averages the result to smooth out
noise. `speedup = before_time / after_time` (guarded against divide-by-zero)
gives a simple multiplier, e.g. `12.4x faster`. The results collect into a list
of dictionaries, then convert into a single summary DataFrame.

**Export the results**

```python
output_path = os.path.join(PROJECT_DIR, "benchmark_results.csv") if os.path.isdir(PROJECT_DIR) else "benchmark_results.csv"
benchmark_df.to_csv(output_path, index=False)
print("Benchmark results saved to:", output_path)
```

Saves the final before/after table as a CSV — the actual portfolio artifact
referenced in the [Results](#results) section above.

**Chart the results**

```python
import matplotlib.pyplot as plt

fig, ax = plt.subplots(figsize=(8, 5))
x = range(len(benchmark_df))
width = 0.35

ax.bar([i - width/2 for i in x], benchmark_df["before_sec"], width, label="Before")
ax.bar([i + width/2 for i in x], benchmark_df["after_sec"], width, label="After")

ax.set_xticks(list(x))
ax.set_xticklabels(benchmark_df["query"], rotation=15)
ax.set_ylabel("Seconds (avg of 5 runs)")
ax.set_title("Query Performance: Before vs After Optimization")
ax.legend()
plt.tight_layout()

chart_path = os.path.join(PROJECT_DIR, "benchmark_chart.png") if os.path.isdir(PROJECT_DIR) else "benchmark_chart.png"
plt.savefig(chart_path, dpi=150)
plt.show()
print("Chart saved to:", chart_path)
```

Builds a grouped bar chart: two bars per query (before/after), positioned
side-by-side by offsetting each set by half the bar width (`± width/2`). Saved
as `benchmark_chart.png`, the image referenced above.

---

## Key Takeaways

- Indexes dramatically speed up reads on large tables, but every index also
  slows down writes — index only the columns actually used in `WHERE`,
  `JOIN`, and `ORDER BY` clauses.
- `EXPLAIN ANALYZE`'s `cost` values are the planner's estimate in arbitrary
  units; `actual time` (in milliseconds) is the real, trustworthy number.
- `SELECT *` and `IN (subquery)` are common but costly patterns — named
  columns and `JOIN`s are usually faster and clearer.
- You can't prove an improvement without a baseline measurement taken first.
- `if_exists="replace"` vs. `"append"` matters a lot when re-running load
  cells — using the wrong one is a common source of duplicated data.

## Security

This project uses local, disposable credentials for a learning database. If
you fork or publish this repo:
- Never commit real database passwords
- Consider using environment variables or a `.env` file (excluded via
  `.gitignore`) instead of hardcoding credentials in the notebook

## Next Steps

Production-scale topics worth exploring beyond this project:
- Table partitioning (range/list) for very large tables
- Materialized views for pre-computed aggregations
- Connection pooling with PgBouncer
- Query result caching (Redis, `pg_prewarm`)
- `pg_stat_statements` for tracking slow queries in a live database

## License

MIT
