---
title: "SQL vs NoSQL"
layout: default
parent: Data Modeling
nav_order: 3
---

# SQL vs NoSQL — Deep Dive

> A comprehensive guide to SQL vs NoSQL — not "which is better" but "which is right for THIS problem." Covers data models, consistency tradeoffs, scaling patterns, and real-world production decisions at companies like Meta, Google, Netflix, and Uber. Written for system design interviews.

---

## Why Does NoSQL Exist?

SQL databases dominated for 40 years. Then three things happened in the 2000s:

**1. Data volume exploded**
- Google indexed the entire web (petabytes)
- Facebook stored billions of social connections
- A single Postgres/MySQL server couldn't hold it all

**2. Write throughput hit limits**
- SQL databases scale reads (replicas) but writes go to ONE master
- Twitter: 500M tweets/day → single master can't handle it
- Sharding SQL is possible but painful (cross-shard joins, distributed transactions)

**3. Data shapes changed**
- Not everything fits neatly into tables with fixed columns
- User profiles with varying fields, nested documents, graph relationships
- Schema migrations on billion-row tables take hours/days

**Google's response:** BigTable (2006) — distributed, column-family store
**Amazon's response:** Dynamo (2007) — distributed, key-value store with eventual consistency
**Facebook's response:** Cassandra (2008) — hybrid of BigTable + Dynamo

These papers launched the NoSQL movement. The core idea: **sacrifice some SQL guarantees to gain horizontal scalability and flexible schemas.**

---

## The Five NoSQL Data Models

### 1. Key-Value Stores

The simplest model. A hash map on steroids.

```
Key → Value (opaque blob — the DB doesn't know what's inside)

"user:123"        → {"name": "Alice", "email": "alice@example.com"}
"session:abc"     → {"user_id": 123, "expires": 1710000000}
"cache:product:42" → "<html>...rendered product page...</html>"
```

**Examples:** Redis, Memcached, Amazon DynamoDB (also document), Riak

**Strengths:**
- Blazing fast — O(1) lookups by key
- Horizontally scalable — partition by key hash
- Simple — no query language to learn

**Weaknesses:**
- No queries by value — can't do "find all users in NYC"
- No relationships — no joins
- No secondary indexes (some KV stores add them, but it's bolted on)

**Use when:** Caching, session storage, rate limiting counters, feature flags, leaderboards

**Real-world:** Redis at Twitter (timeline caching), Memcached at Facebook (query cache), DynamoDB at Amazon (shopping cart)

### 2. Document Stores

Like key-value, but the database **understands the value's structure** (JSON/BSON). You can query and index fields inside the document.

```json
// Collection: users
{
  "_id": "user_123",
  "name": "Alice",
  "email": "alice@example.com",
  "address": {
    "city": "NYC",
    "zip": "10001"
  },
  "orders": [
    { "id": "ord_1", "total": 150.00, "items": ["book", "pen"] },
    { "id": "ord_2", "total": 89.99, "items": ["laptop"] }
  ]
}
```

```javascript
// Query by nested field — the DB understands the structure
db.users.find({ "address.city": "NYC", "orders.total": { $gt: 100 } })
```

**Examples:** MongoDB, Couchbase, Amazon DocumentDB, Firestore

**Strengths:**
- Flexible schema — each document can have different fields
- Natural for hierarchical data (embed related data in one document)
- Rich queries on document fields
- Horizontal scaling via sharding

**Weaknesses:**
- No joins (you denormalize — embed data or do multiple queries)
- Data duplication (Alice's address stored in every order if denormalized)
- Transactions were limited (MongoDB added multi-doc ACID in v4.0, but expensive)

**Use when:** Content management, user profiles, product catalogs, event logging, any data that's naturally document-shaped

**Real-world:** MongoDB at Forbes (CMS), Couchbase at LinkedIn (profile data), Firestore at Google (mobile app backends)

### 3. Wide-Column (Column-Family) Stores

Data organized by columns rather than rows. Each row can have different columns. Optimized for writes and range scans.

```
Row Key    | Column Family: profile      | Column Family: activity
-----------+-----------------------------+---------------------------
user_123   | name: "Alice"               | last_login: "2025-03-14"
           | email: "alice@example.com"  | login_count: 42
           | city: "NYC"                 |
-----------+-----------------------------+---------------------------
user_456   | name: "Bob"                 | last_login: "2025-03-13"
           | email: "bob@example.com"    | login_count: 7
           |                             | last_purchase: "2025-03-10"
```

**Examples:** Apache Cassandra, HBase, Google Bigtable, ScyllaDB

**Strengths:**
- Massive write throughput (append-only LSM trees)
- Linear horizontal scaling (add nodes, data rebalances)
- Tunable consistency (per-query: ONE, QUORUM, ALL)
- Great for time-series data (row key = sensor_id, columns = timestamps)

**Weaknesses:**
- No joins, no subqueries
- Limited query patterns — must design around partition key
- Denormalization required (data modeling is hard)
- No ACID transactions across partitions

**Use when:** Time-series data, IoT sensor data, activity logs, messaging systems, anything with massive write volume

**Real-world:** Cassandra at Netflix (viewing history — trillions of rows), Cassandra at Discord (messages), Bigtable at Google (Search index, Maps, Gmail)

### 4. Graph Databases

Data modeled as nodes (entities) and edges (relationships). Optimized for traversing connections.

```
(Alice)--[FRIENDS_WITH]-->(Bob)
(Alice)--[PURCHASED]-->(Product_42)
(Bob)--[REVIEWED]-->(Product_42)
(Product_42)--[CATEGORY]-->(Electronics)
```

```cypher
// Cypher query (Neo4j): "Friends of Alice who bought electronics"
MATCH (alice:User {name: "Alice"})-[:FRIENDS_WITH]->(friend)
      -[:PURCHASED]->(product)-[:CATEGORY]->(cat:Category {name: "Electronics"})
RETURN friend.name, product.name
```

**Examples:** Neo4j, Amazon Neptune, JanusGraph, Dgraph

**Strengths:**
- Relationship traversal is O(1) per hop (not O(n) like SQL joins)
- Natural for connected data (social graphs, recommendations)
- Flexible schema — add new relationship types anytime

**Weaknesses:**
- Not great for aggregations or bulk analytics
- Harder to shard (graph partitioning is NP-hard)
- Smaller ecosystem than SQL or document stores

**Use when:** Social networks, recommendation engines, fraud detection, knowledge graphs, network topology

**Real-world:** Neo4j at eBay (product recommendations), Neptune at Amazon (knowledge graph), custom graph at Facebook (social graph — TAO)

### 5. Search Engines (Inverted Index)

Not a traditional database, but often used as one. Optimized for full-text search and analytics.

```json
// Document indexed in Elasticsearch
{
  "title": "Introduction to Distributed Systems",
  "body": "Distributed systems are collections of...",
  "tags": ["distributed", "systems", "architecture"],
  "published": "2025-03-14"
}

// Inverted index (simplified):
"distributed" → [doc_1, doc_5, doc_12]
"systems"     → [doc_1, doc_3, doc_5]
"architecture" → [doc_1, doc_7]

// Search: "distributed systems" → intersection → [doc_1, doc_5]
```

**Examples:** Elasticsearch, Apache Solr, Meilisearch, Typesense

**Use when:** Full-text search, log analytics, product search with facets, autocomplete

**Real-world:** Elasticsearch at GitHub (code search), Elasticsearch at Uber (trip search), Elasticsearch at Netflix (log analysis)

---

## SQL vs NoSQL — Complete Comparison

| Aspect | SQL (Postgres, MySQL) | Document (MongoDB) | Key-Value (Redis) | Wide-Column (Cassandra) | Graph (Neo4j) |
|--------|----------------------|-------------------|-------------------|------------------------|---------------|
| Data Model | Tables, rows, columns | JSON documents | Key → blob | Row key → column families | Nodes + edges |
| Schema | Fixed, enforced | Flexible per document | None (opaque) | Flexible per row | Flexible |
| Query Language | SQL | MQL / JSON queries | GET/SET commands | CQL (SQL-like) | Cypher / Gremlin |
| Joins | Native, powerful | None (embed or $lookup) | None | None | Native (traversals) |
| Transactions | Full ACID | Multi-doc ACID (v4.0+) | Single-key atomic | Single-partition | ACID (single node) |
| Consistency | Strong (default) | Strong or eventual | Strong (single node) | Tunable (ONE→ALL) | Strong (single node) |
| Scaling | Vertical (+ read replicas) | Horizontal (sharding) | Horizontal (cluster) | Horizontal (linear) | Vertical mostly |
| Write Throughput | 10K-50K/sec (single node) | 50K-100K/sec | 100K-500K/sec | 100K-1M/sec | 10K-50K/sec |
| Read Latency | 1-10ms (indexed) | 1-10ms | <1ms | 1-10ms | 1-10ms (traversal) |
| Best For | Relational data, ACID | Flexible documents | Caching, sessions | Time-series, high write | Connected data |

---

## The Real Decision Framework

Forget "SQL vs NoSQL." Ask these questions:

### 1. What does the data look like?

```
Highly relational (users → orders → items → reviews)?
  → SQL. Joins are your friend.

Document-shaped (user profiles with varying fields)?
  → Document store. Embed related data.

Simple key-based access (cache, sessions, counters)?
  → Key-value store. Fastest possible.

Time-series / append-heavy (logs, metrics, events)?
  → Wide-column store. Optimized for writes.

Heavily connected (social graph, recommendations)?
  → Graph database. Traversals are O(1) per hop.
```

### 2. What are the access patterns?

```
Complex queries with joins, aggregations, GROUP BY?
  → SQL. No other model handles this as well.

Lookup by ID, maybe a few secondary indexes?
  → Document or key-value. Simpler, faster.

"Find all X related to Y within N hops"?
  → Graph. SQL joins become exponentially expensive at depth.

Full-text search with relevance ranking?
  → Search engine (Elasticsearch). SQL LIKE is terrible for this.
```

### 3. What consistency do you need?

```
"Money can never be wrong" (financial, inventory, booking)?
  → SQL with ACID transactions. Non-negotiable.

"Eventually correct is fine" (likes, views, feeds)?
  → NoSQL with eventual consistency. Better availability.

"Correct within a partition, eventual across partitions"?
  → Cassandra with QUORUM reads/writes.
```

### 4. What scale do you need?

```
< 1 TB data, < 10K writes/sec?
  → Single Postgres instance handles this easily. Don't over-engineer.

1-10 TB, 10K-100K writes/sec?
  → Postgres with read replicas + partitioning, OR
  → MongoDB/Cassandra if data model fits.

> 10 TB, > 100K writes/sec?
  → Distributed NoSQL (Cassandra, DynamoDB), OR
  → Sharded SQL (Vitess, Citus, CockroachDB, Spanner).

> 1 PB?
  → You're Google/Meta. Custom solutions.
```

### Interview Tip 💡
> Never say "I'd use NoSQL because it scales better." That's a red flag. Instead say: "I'd start with Postgres because the data is relational and we need ACID for payments. For the activity feed, I'd use Cassandra because it's write-heavy, time-ordered, and eventual consistency is acceptable. For caching, Redis." Show you pick the right tool for each part of the system.

---

## The Polyglot Persistence Pattern

Real production systems use MULTIPLE databases. Each optimized for its use case.

```
┌─────────────────────────────────────────────────────────┐
│                    Application Layer                     │
└──────┬──────────┬──────────┬──────────┬────────────────┘
       │          │          │          │
       ↓          ↓          ↓          ↓
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐
│ Postgres │ │ Redis    │ │Cassandra │ │Elasticsearch │
│          │ │          │ │          │ │              │
│ Users    │ │ Sessions │ │ Activity │ │ Search       │
│ Orders   │ │ Cache    │ │ Feed     │ │ Index        │
│ Payments │ │ Rate     │ │ Messages │ │ Logs         │
│ Inventory│ │ Limits   │ │ Metrics  │ │ Analytics    │
└──────────┘ └──────────┘ └──────────┘ └──────────────┘
  ACID          Speed       Write scale   Full-text
  Relational    Sub-ms      Time-series   Relevance
```

### How Big Companies Do It

**Meta (Facebook):**
- **MySQL** — user accounts, social graph metadata (TAO cache on top)
- **Cassandra** — inbox search, messaging
- **RocksDB** — embedded KV store inside many services
- **Custom graph** — TAO for social graph traversals
- **Elasticsearch** — search

**Google:**
- **Spanner** — globally distributed SQL (AdWords, Google Play)
- **Bigtable** — massive scale KV/wide-column (Search, Maps, Gmail)
- **Firestore** — document store for mobile/web apps
- **Memorystore (Redis)** — caching

**Netflix:**
- **Cassandra** — viewing history, bookmarks (trillions of rows)
- **MySQL** — billing, account data (ACID needed)
- **Elasticsearch** — search, log analysis
- **EVCache (Memcached)** — caching layer

**Uber:**
- **MySQL (Schemaless)** — trip data, custom sharding layer
- **Cassandra** — real-time analytics
- **Redis** — caching, geospatial indexes
- **Elasticsearch** — search

---

## Common Misconceptions

### "NoSQL is faster than SQL"
**Wrong.** Redis is faster than Postgres for key lookups because it's in-memory, not because it's NoSQL. Postgres with proper indexing returns queries in <1ms. The speed difference comes from the data model and access pattern, not the SQL vs NoSQL label.

### "NoSQL scales better"
**Partially true.** NoSQL databases were designed for horizontal scaling from day one. But SQL can scale too — Postgres with Citus, MySQL with Vitess, Google Spanner, CockroachDB. The difference is that NoSQL makes scaling easier by giving up joins and multi-record transactions.

### "SQL can't handle big data"
**Wrong.** Google Spanner handles petabytes with full ACID. Postgres with partitioning and Citus handles terabytes. The question is whether you need the relational model at that scale.

### "NoSQL means no schema"
**Misleading.** The schema moves from the database to the application. Your code still needs to know the data structure. "Schema-on-read" means you validate when reading, not when writing. This can be a feature (flexibility) or a bug (data quality issues).

### "MongoDB lost data"
**Historical.** Early MongoDB (pre-3.0) had weak durability defaults (no journaling, no write concern). Modern MongoDB with `w: majority` and journaling is durable. The reputation stuck, but the product improved.

---

## Common Interview Questions

### Q: Design a social media platform — what databases would you use?
- **Postgres:** User accounts, authentication, settings (relational, ACID)
- **Cassandra:** News feed, activity timeline (write-heavy, time-ordered)
- **Redis:** Feed cache, session store, rate limiting (speed)
- **Elasticsearch:** User search, post search (full-text)
- **S3 + CDN:** Media storage (images, videos)

### Q: When would you choose Cassandra over Postgres?
When you need: massive write throughput (>100K/sec), linear horizontal scaling, time-series or append-heavy data, and can accept eventual consistency. Example: storing billions of IoT sensor readings.

When you'd keep Postgres: data is relational, you need joins, you need ACID transactions, data fits on one machine (<1TB).

### Q: How do you migrate from SQL to NoSQL (or vice versa)?
1. **Dual-write:** Write to both old and new DB simultaneously
2. **Backfill:** Copy historical data to new DB
3. **Shadow read:** Read from both, compare results, serve from old
4. **Cutover:** Switch reads to new DB
5. **Decommission:** Stop writing to old DB

Key: never big-bang migrate. Always incremental with rollback capability.

### Q: Can you have ACID with NoSQL?
- **Single-document:** MongoDB, DynamoDB — atomic at document/item level
- **Multi-document:** MongoDB 4.0+ — multi-doc ACID (with performance cost)
- **Distributed SQL:** CockroachDB, Spanner, YugabyteDB — full ACID + horizontal scaling
- **The trend:** The line between SQL and NoSQL is blurring. NewSQL databases offer both.

---

## Further Reading
- SQL Databases — Postgres internals, indexing, query execution
- ACID Properties — deep dive into transaction guarantees
- Common API Patterns — pagination patterns across different databases
- Rate Limiting — Redis-based distributed rate limiting

---

*Last updated: 2025-03-14*
