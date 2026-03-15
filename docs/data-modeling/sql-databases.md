---
title: "SQL Databases"
layout: default
parent: Data Modeling
nav_order: 1
---

# SQL Databases — Deep Dive

> A comprehensive guide to SQL databases — why they exist, how they store data, how queries execute, indexing internals, and why relational databases have dominated for 50 years. Written for system design interviews and real-world engineering decisions.

---

## Why Do Databases Exist?

Before databases, programs stored data in flat files. This caused real problems:

```
users.txt:
  1,Alice,alice@email.com,NYC
  2,Bob,bob@email.com,SF
  3,Charlie,charlie@email.com,NYC

orders.txt:
  101,1,2025-01-15,150.00
  102,2,2025-01-16,89.99
  103,1,2025-01-17,200.00
```

**Problems with flat files:**
- **No concurrent access** — two programs writing simultaneously corrupt the file
- **No crash recovery** — power failure mid-write = corrupted data
- **No query language** — want "all orders from NYC users over $100"? Write custom parsing code
- **No relationships** — linking users to orders requires manual cross-referencing
- **No integrity** — nothing prevents order 104 referencing non-existent user 999
- **No security** — can't give read-only access to some users

Databases solve all of these. SQL databases specifically solve them with a **relational model** — data organized into tables with rows and columns, connected by relationships.

---

## What is the Relational Model?

Proposed by Edgar F. Codd at IBM in 1970. The key insight: separate the **logical representation** of data (tables, rows, columns) from the **physical storage** (how bytes are arranged on disk).

### Core Concepts

**Relations (Tables):**
```sql
CREATE TABLE users (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    email       VARCHAR(255) UNIQUE NOT NULL,
    city        VARCHAR(100),
    created_at  TIMESTAMP DEFAULT NOW()
);

CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,
    user_id     INTEGER REFERENCES users(id),  -- foreign key
    total       DECIMAL(10,2) NOT NULL,
    status      VARCHAR(20) DEFAULT 'pending',
    created_at  TIMESTAMP DEFAULT NOW()
);
```

**Tuples (Rows):** Each row is a single record — one user, one order.

**Attributes (Columns):** Each column has a name and a type. Types are enforced — you can't put "hello" in an INTEGER column.

**Keys:**
- **Primary Key** — uniquely identifies each row (`id`)
- **Foreign Key** — references a primary key in another table (`user_id → users.id`)
- **Unique Key** — ensures no duplicates (`email`)
- **Composite Key** — multiple columns together form the key

**Schema:** The structure definition. Unlike NoSQL, the schema is enforced by the database — every row must conform.

### Why Relations Matter

The power is in **joins** — combining data from multiple tables in a single query:

```sql
-- "All orders from NYC users over $100"
-- This is trivial in SQL, painful with flat files
SELECT u.name, o.total, o.created_at
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.city = 'NYC' AND o.total > 100
ORDER BY o.created_at DESC;
```

### Normalization — Eliminating Redundancy

Normalization organizes data to reduce duplication:

```
❌ Denormalized (redundant):
  orders table:
    order_id | user_name | user_email      | user_city | total
    101      | Alice     | alice@email.com | NYC       | 150.00
    103      | Alice     | alice@email.com | NYC       | 200.00
    -- Alice's info repeated! If she changes email, update EVERY row

✅ Normalized (3NF):
  users: id | name  | email           | city
         1  | Alice | alice@email.com | NYC

  orders: id  | user_id | total
          101 | 1       | 150.00
          103 | 1       | 200.00
    -- Alice's info stored once. Change email in one place.
```

**Normal Forms (simplified):**
- **1NF:** No repeating groups, atomic values (no arrays in cells)
- **2NF:** 1NF + every non-key column depends on the entire primary key
- **3NF:** 2NF + no transitive dependencies (non-key columns don't depend on other non-key columns)

**Interview Tip 💡**
> In practice, most production systems use 3NF for transactional data (OLTP) but intentionally denormalize for read-heavy analytics (OLAP). Know when to break normalization rules — it's a tradeoff between write consistency and read performance.

---

## How SQL Databases Store Data — Postgres Internals

Understanding storage internals helps you make better indexing and schema decisions.

### Pages — The Fundamental Unit

Postgres doesn't read individual rows from disk. It reads **pages** (8 KB blocks by default).

```
Disk:
  ┌──────────┐┌──────────┐┌──────────┐┌──────────┐
  │  Page 0  ││  Page 1  ││  Page 2  ││  Page 3  │  ...
  │  (8 KB)  ││  (8 KB)  ││  (8 KB)  ││  (8 KB)  │
  │          ││          ││          ││          │
  │ Row 1    ││ Row 15   ││ Row 30   ││ Row 45   │
  │ Row 2    ││ Row 16   ││ Row 31   ││ Row 46   │
  │ ...      ││ ...      ││ ...      ││ ...      │
  │ Row 14   ││ Row 29   ││ Row 44   ││ Row 59   │
  └──────────┘└──────────┘└──────────┘└──────────┘
```

Each page contains:
- **Page header** (24 bytes) — metadata, free space pointer
- **Item pointers** (4 bytes each) — array pointing to row locations within the page
- **Rows (tuples)** — actual data, stored from the end of the page backward
- **Free space** — between item pointers and rows

### Heap — The Table Storage

In Postgres, a table is stored as a **heap** — an unordered collection of pages. Rows are inserted wherever there's free space.

```
Table "users" on disk:
  File: base/16384/16385  (OID of the table)
  
  Page 0: [Alice, Bob, Charlie, ...]
  Page 1: [Dave, Eve, Frank, ...]
  Page 2: [Grace, Heidi, Ivan, ...]
  ...
```

**Key insight:** Without an index, finding a specific row requires scanning EVERY page (sequential scan). For a 1 GB table with 8 KB pages, that's 131,072 pages to read.

### MVCC — How Postgres Handles Concurrency

Postgres uses **Multi-Version Concurrency Control**. Instead of locking rows, it keeps multiple versions of each row.

```
Row "Alice" versions:
  Version 1: { name: "Alice", city: "NYC" }     xmin=100, xmax=200
  Version 2: { name: "Alice", city: "SF" }      xmin=200, xmax=∞

Transaction 150 (started before 200): sees Version 1 (NYC)
Transaction 250 (started after 200):  sees Version 2 (SF)
```

Each row has hidden system columns:
- `xmin` — transaction ID that created this version
- `xmax` — transaction ID that deleted/updated this version (∞ if current)
- `ctid` — physical location (page number, offset within page)

**This is why Postgres tables bloat over time** — old row versions aren't immediately removed. `VACUUM` cleans them up.

### Indexes — B-Tree Internals

The default index in Postgres is a **B-Tree** (balanced tree).

```sql
CREATE INDEX idx_users_email ON users(email);
```

```
B-Tree for email index:
                    ┌─────────────────────┐
                    │   [frank@, mike@]   │  ← Root node
                    └──┬──────┬──────┬────┘
                       │      │      │
            ┌──────────┘      │      └──────────┐
            ↓                 ↓                  ↓
    ┌───────────────┐ ┌───────────────┐ ┌───────────────┐
    │ [alice@,      │ │ [frank@,      │ │ [mike@,       │
    │  bob@,        │ │  grace@,      │ │  nancy@,      │
    │  charlie@,    │ │  heidi@,      │ │  oscar@,      │
    │  dave@, eve@] │ │  ivan@, ...]  │ │  pat@, ...]   │
    └───────────────┘ └───────────────┘ └───────────────┘
         Leaf nodes contain: (email_value, ctid pointer to heap)
```

**How a lookup works:**
```sql
SELECT * FROM users WHERE email = 'grace@email.com';

1. Start at root: grace@ > frank@ → go right
2. At internal node: grace@ is in [frank@, ...] range → go to that leaf
3. At leaf: find grace@ → get ctid (page 5, offset 3)
4. Go to heap page 5, read row at offset 3
5. Return row ✅

Pages read: 3 (root + internal + leaf) + 1 (heap) = 4 pages
Without index: scan all 131,072 pages 😬
```

**B-Tree properties:**
- Balanced — all leaf nodes at the same depth
- Sorted — enables range queries (`WHERE email > 'f' AND email < 'h'`)
- O(log n) lookups — 1 billion rows ≈ 10 levels deep ≈ 10 page reads
- Each node is one page (8 KB), fits hundreds of keys

### Other Index Types in Postgres

| Index Type | Use Case | Example |
|------------|----------|---------|
| B-Tree | Equality, range, sorting (default) | `WHERE age > 25` |
| Hash | Equality only (faster than B-Tree for =) | `WHERE id = 123` |
| GIN | Full-text search, arrays, JSONB | `WHERE tags @> '{python}'` |
| GiST | Geometric, range types, nearest-neighbor | `WHERE location <-> point(40.7, -74.0) < 1000` |
| BRIN | Very large tables with natural ordering | `WHERE created_at > '2025-01-01'` on append-only tables |

**Interview Tip 💡**
> When an interviewer asks "how would you optimize this query?", the answer almost always starts with indexing. Know B-Tree for most cases, GIN for JSONB/full-text, and BRIN for time-series data. Also mention `EXPLAIN ANALYZE` — the tool that shows you exactly how Postgres executes a query.

---

## How a SQL Query Executes — Postgres Query Pipeline

```sql
SELECT u.name, COUNT(o.id) as order_count
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE u.city = 'NYC'
GROUP BY u.name
HAVING COUNT(o.id) > 5
ORDER BY order_count DESC
LIMIT 10;
```

### The Pipeline

```
SQL Text
  ↓
1. Parser → Abstract Syntax Tree (AST)
  ↓
2. Analyzer → Resolves table/column names, checks types
  ↓
3. Rewriter → Applies rules (views, row-level security)
  ↓
4. Planner/Optimizer → Generates execution plan (the most complex step)
  ↓
5. Executor → Runs the plan, returns results
```

### The Planner — Where the Magic Happens

The planner considers multiple strategies and picks the cheapest one:

```
EXPLAIN ANALYZE SELECT * FROM users WHERE city = 'NYC';

Option A: Sequential Scan
  Read every page, check each row → Cost: 131,072 page reads

Option B: Index Scan (if index on city exists)
  B-Tree lookup → heap fetch → Cost: ~100 page reads

Option C: Bitmap Index Scan (many matching rows)
  B-Tree → build bitmap of matching pages → read those pages → Cost: ~5,000 page reads

Planner picks the cheapest based on:
  - Table statistics (row count, value distribution, null fraction)
  - Available indexes
  - Estimated selectivity (what % of rows match the WHERE clause)
  - Disk I/O cost vs CPU cost
```

### Join Strategies

```
Nested Loop Join:
  For each row in users WHERE city='NYC':
    Scan orders WHERE user_id = user.id
  Best for: small outer table, indexed inner table
  Cost: O(n × m) worst case, O(n × log m) with index

Hash Join:
  1. Build hash table from smaller table (users WHERE city='NYC')
  2. Scan larger table (orders), probe hash table for matches
  Best for: large tables, no useful index, equality joins
  Cost: O(n + m)

Merge Join:
  1. Sort both tables on join key
  2. Merge sorted streams (like merge sort's merge step)
  Best for: both tables already sorted (index), large datasets
  Cost: O(n log n + m log m) for sort, O(n + m) for merge
```

**Interview Tip 💡**
> If asked "why is this query slow?", walk through: (1) check `EXPLAIN ANALYZE`, (2) look for sequential scans on large tables, (3) check if indexes exist for WHERE/JOIN columns, (4) check for missing statistics (`ANALYZE` command). This shows you debug systematically, not guess.

---

## Transactions and Isolation Levels

See ACID Properties for the deep dive on transactions. Here's the Postgres-specific behavior:

### Postgres Default: Read Committed

```sql
-- Transaction A                    -- Transaction B
BEGIN;                              BEGIN;
SELECT balance FROM accounts        
WHERE id = 1;  -- sees $1000       
                                    UPDATE accounts SET balance = 500
                                    WHERE id = 1;
                                    COMMIT;
SELECT balance FROM accounts        
WHERE id = 1;  -- sees $500 ← changed within same transaction!
COMMIT;
```

In Read Committed, each statement sees the latest committed data. This means two SELECTs in the same transaction can return different results.

### Postgres Serializable: Full Isolation

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;
-- Now this transaction behaves as if it's the only one running
-- Postgres uses Serializable Snapshot Isolation (SSI)
-- If a conflict is detected, one transaction is aborted
```

---

## Connection Pooling — Why It Matters

Each Postgres connection = 1 OS process (~10 MB RAM). 1000 connections = 10 GB just for connection overhead.

```
Without pooling:
  App Server (1000 requests) → 1000 Postgres connections → 💀 OOM

With pooling (PgBouncer):
  App Server (1000 requests) → PgBouncer (50 connections) → Postgres
  Requests queue and share connections
```

**PgBouncer modes:**
- **Session pooling** — connection assigned for entire session (safest)
- **Transaction pooling** — connection assigned per transaction (most efficient)
- **Statement pooling** — connection assigned per statement (most aggressive, breaks multi-statement transactions)

**Production rule:** Use transaction pooling with PgBouncer. Set Postgres `max_connections` to ~100-200, let PgBouncer handle thousands of app connections.

---

## Postgres-Specific Features That Matter

### JSONB — Best of Both Worlds

```sql
CREATE TABLE events (
    id      SERIAL PRIMARY KEY,
    type    VARCHAR(50),
    data    JSONB  -- structured + flexible
);

INSERT INTO events (type, data) VALUES 
('purchase', '{"user_id": 123, "amount": 99.99, "items": ["book", "pen"]}');

-- Query JSON fields
SELECT data->>'user_id' as user_id, data->>'amount' as amount
FROM events
WHERE data->>'amount'::numeric > 50;

-- Index JSON fields
CREATE INDEX idx_events_user ON events USING GIN (data);
```

JSONB gives you schema flexibility within a relational database. Use it for semi-structured data that varies between rows (event payloads, user preferences, feature flags).

### Partitioning — Scaling Large Tables

```sql
-- Partition by range (time-series data)
CREATE TABLE logs (
    id          BIGSERIAL,
    created_at  TIMESTAMP NOT NULL,
    message     TEXT
) PARTITION BY RANGE (created_at);

CREATE TABLE logs_2025_01 PARTITION OF logs
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
CREATE TABLE logs_2025_02 PARTITION OF logs
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

-- Query only touches relevant partition
SELECT * FROM logs WHERE created_at >= '2025-02-15';
-- Postgres only scans logs_2025_02, skips logs_2025_01 ✅
```

### CTEs and Window Functions

```sql
-- Common Table Expression (CTE) — readable complex queries
WITH monthly_revenue AS (
    SELECT DATE_TRUNC('month', created_at) as month,
           SUM(total) as revenue
    FROM orders
    GROUP BY 1
)
SELECT month, revenue,
       LAG(revenue) OVER (ORDER BY month) as prev_month,
       revenue - LAG(revenue) OVER (ORDER BY month) as growth
FROM monthly_revenue;
```

---

## When to Use SQL Databases

### ✅ Use SQL When
- Data has clear relationships (users → orders → items)
- You need ACID transactions (financial systems, inventory)
- Complex queries with joins, aggregations, window functions
- Data integrity is critical (foreign keys, constraints, types)
- You need strong consistency (read-after-write guarantees)
- Your data model is well-defined and doesn't change frequently

### ❌ Don't Use SQL When
- Schema changes constantly (rapid prototyping with unknown structure)
- You need horizontal write scaling beyond a single node
- Data is hierarchical/document-shaped (deeply nested JSON)
- You need sub-millisecond latency at massive scale (caching layer)
- Write throughput exceeds what a single master can handle

---

## Real-World Production Examples

### 1. Stripe — Financial Transactions
- **Postgres** for all payment data
- ACID transactions ensure money never appears or disappears
- Sharded Postgres for horizontal scaling
- **Why SQL:** Financial data demands absolute consistency

### 2. Instagram — User Data
- **Postgres** for user profiles, relationships, media metadata
- Sharded across many Postgres instances
- Django ORM on top
- **Why SQL:** Complex queries (feed generation, search, analytics)

### 3. GitHub — Repository Metadata
- **MySQL** for repositories, issues, pull requests, users
- Vitess for horizontal sharding
- **Why SQL:** Relational data (repos → issues → comments → users)

### 4. Uber — Trip Data
- **Postgres** (originally, then migrated to MySQL + Schemaless)
- Needed to shard beyond single Postgres capacity
- **Why they moved:** Write scaling limits of single-node Postgres

### 5. Netflix — Billing and Account Data
- **MySQL** for billing, subscriptions, account management
- **Why SQL:** Financial transactions need ACID guarantees

---

## Further Reading
- ACID Properties — the guarantees that make SQL databases reliable
- SQL vs NoSQL — when to choose which
- Common API Patterns — pagination patterns that interact with SQL queries
- Rate Limiting — using SQL/Redis for distributed rate limiting

---

*Last updated: 2025-03-14*
