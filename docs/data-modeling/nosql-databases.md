---
title: "NoSQL Databases"
layout: default
parent: Data Modeling
nav_order: 4
---

# NoSQL Databases — Deep Dive

> A comprehensive guide to NoSQL internals — how MongoDB and DynamoDB actually store, index, query, and replicate data under the hood. Storage engines, consistency models, partition strategies, and production patterns. Written for system design interviews at Meta/Google level.

---

## Why This Matters

Knowing "MongoDB stores documents" is surface-level. Interviewers at top companies want to hear: How does the storage engine lay out data on disk? What happens when you write a document — what's the I/O path? How does sharding actually distribute data? What are the consistency guarantees and how are they implemented? When does it break, and what are the failure modes?

We'll cover MongoDB (self-managed document store) and DynamoDB (managed key-value/document store) because they represent two fundamentally different architectures.

---

## Part 1: MongoDB — Internals

---

### Data Model — Documents, Not Rows

MongoDB stores data as **BSON** (Binary JSON) documents in **collections** (analogous to tables).

```javascript
// A single document in the "orders" collection
{
  _id: ObjectId("65f1a2b3c4d5e6f7a8b9c0d1"),
  user_id: "user_123",
  status: "shipped",
  total: 299.99,
  items: [
    { product_id: "prod_42", name: "Keyboard", qty: 1, price: 149.99 },
    { product_id: "prod_77", name: "Mouse", qty: 2, price: 75.00 }
  ],
  shipping: {
    address: "123 Main St, NYC",
    carrier: "FedEx",
    tracking: "FX123456789"
  },
  created_at: ISODate("2025-03-14T10:30:00Z")
}
```

**Key difference from SQL:** Related data is **embedded** in the document instead of normalized across tables. The order, its items, and shipping info are one atomic unit — no joins needed.

### BSON — Why Not Just JSON?

MongoDB doesn't store raw JSON. It uses BSON (Binary JSON):

```
JSON:  {"age": 30}
       Text: 12 bytes, must parse string "30" to integer every read

BSON:  \x10 age \x00 \x1e\x00\x00\x00
       Binary: type tag (int32) + field name + 4-byte integer
       No parsing needed, direct memory access
```

| Aspect | JSON | BSON |
|--------|------|------|
| Format | Text (UTF-8) | Binary |
| Types | 6 (string, number, bool, null, array, object) | 20+ (int32, int64, decimal128, date, binary, regex, ...) |
| Traversal | Parse entire document | Skip to field by offset |
| Speed | Slow to parse | Fast to traverse |

BSON's length-prefixed fields mean MongoDB can **skip over fields** it doesn't need without parsing them.

### ObjectId — The Default Primary Key

Every document gets an `_id` field. The default type is **ObjectId** — a 12-byte value:

```
ObjectId("65f1a2b3c4d5e6f7a8b9c0d1")

Bytes:  [65f1a2b3] [c4d5e6] [f7a8] [b9c0d1]
         ↑          ↑        ↑      ↑
         Timestamp   Machine  PID    Counter
         (4 bytes)   (3 bytes)(2 b)  (3 bytes)
         seconds     random   process incrementing
         since epoch          ID
```

**Why this design matters:**
- **Timestamp embedded** — you can extract creation time from the ID itself, no separate `created_at` needed
- **Roughly sortable** — ObjectIds created later are greater, so `_id` index gives approximate insertion order
- **Globally unique without coordination** — no central ID generator needed. Each process generates unique IDs independently
- **12 bytes** — smaller than UUID (16 bytes), larger than auto-increment integer (4-8 bytes)

### Interview Tip 💡
> When asked "how would you generate unique IDs in a distributed system?", ObjectId's design is a great answer. It combines a timestamp, machine identifier, and counter to guarantee uniqueness without any coordination between nodes. Compare this to Twitter's Snowflake ID which uses a similar approach.

---

### Storage Engine — WiredTiger

Since MongoDB 3.2, the default storage engine is **WiredTiger**. This is where the real internals live.

#### B-Tree Storage (Not LSM)

WiredTiger uses **B+ Trees** for both the data and indexes (similar to Postgres, different from Cassandra's LSM trees).

```
_id Index (B+ Tree):
                    ┌─────────────────────┐
                    │   [id_500, id_1000] │  Root
                    └──┬──────┬──────┬────┘
                       │      │      │
            ┌──────────┘      │      └──────────┐
            ↓                 ↓                  ↓
    ┌───────────────┐ ┌───────────────┐ ┌───────────────┐
    │ id_1 → doc    │ │ id_501 → doc  │ │ id_1001 → doc │
    │ id_2 → doc    │ │ id_502 → doc  │ │ id_1002 → doc │
    │ ...           │ │ ...           │ │ ...            │
    │ id_500 → doc  │ │ id_1000 → doc│ │ id_1500 → doc │
    └───────────────┘ └───────────────┘ └───────────────┘
         Leaf nodes store the actual BSON documents
```

**Key difference from Postgres:** In MongoDB, the `_id` index IS the primary storage. The document data lives in the B+ Tree leaf nodes (clustered index). In Postgres, the heap stores data and indexes point to heap locations.

#### Document Storage Layout

```
WiredTiger on disk:

Collection "orders":
  File: collection-7-xxxxx.wt

  B+ Tree organized by _id:
    Internal pages: [keys only, pointers to child pages]
    Leaf pages: [keys + full BSON documents]
    
  Each page: configurable size (default ~4KB internal, ~32KB leaf)
  Pages are compressed on disk (snappy or zlib or zstd)
```

#### Compression

WiredTiger compresses data at two levels:

```
1. Block compression (entire pages):
   - snappy (default): fast, moderate compression (~50-70% ratio)
   - zlib: slower, better compression (~30-50% ratio)  
   - zstd: best balance of speed + compression

2. Prefix compression (within index pages):
   Keys: "user_123_order_001", "user_123_order_002", "user_123_order_003"
   Stored as: "user_123_order_001", "+002", "+003"
   Saves significant space in indexes with common prefixes
```

**Real impact:** A 100 GB uncompressed collection might be 30-50 GB on disk with snappy compression.

#### Write Path — What Happens on Insert

```
db.orders.insertOne({ user_id: "user_123", total: 299.99, ... })

Step 1: Client sends write to the PRIMARY node
Step 2: MongoDB validates the document (size < 16MB, valid BSON)
Step 3: Generate ObjectId if _id not provided
Step 4: Write to the JOURNAL (Write-Ahead Log)
        → Sequential write to journal file on disk
        → This is the durability guarantee (like Postgres WAL)
Step 5: Write to WiredTiger's in-memory B+ Tree (cache)
        → Document inserted into the correct leaf page
Step 6: Update all secondary indexes in memory
Step 7: Return acknowledgment to client
Step 8: (Background) Checkpoint flushes dirty pages to disk
        → Default: every 60 seconds or 2GB of journal data
```

```
Memory (WiredTiger Cache):
  ┌─────────────────────────────────────┐
  │  B+ Tree pages (modified in RAM)    │
  │  ┌──────┐ ┌──────┐ ┌──────┐       │
  │  │Page 1│ │Page 5│ │Page 9│ dirty  │
  │  │(dirty)│ │(clean)│ │(dirty)│       │
  │  └──────┘ └──────┘ └──────┘       │
  └─────────────────────────────────────┘
           ↓ checkpoint (every 60s)
  Disk:
  ┌─────────────────────────────────────┐
  │  Journal: [op1] [op2] [op3] ...    │ ← durability
  │  Data files: collection-7-xxx.wt   │ ← updated at checkpoint
  │  Index files: index-8-xxx.wt       │
  └─────────────────────────────────────┘
```

#### WiredTiger Cache — The Critical Resource

WiredTiger maintains its own cache (separate from the OS page cache):

```
Default cache size: 50% of (RAM - 1GB)
  Server with 16 GB RAM → ~7.5 GB WiredTiger cache

Cache contains:
  - Uncompressed B+ Tree pages (data + indexes)
  - Modified (dirty) pages waiting for checkpoint
  - Internal pages for tree navigation

Cache pressure:
  When cache is full → evict clean pages first
  If still full → write dirty pages to disk (eviction)
  If eviction can't keep up → operations start blocking 💀
```

**This is the #1 MongoDB performance issue.** If your working set (frequently accessed data) doesn't fit in the WiredTiger cache, every read hits disk. Monitor `wiredTiger.cache.bytes currently in the cache` vs `wiredTiger.cache.maximum bytes configured`.

### Interview Tip 💡
> When asked "how would you size a MongoDB deployment?", the answer starts with: "ensure the working set fits in the WiredTiger cache." If your hot data is 20 GB, you need at least 40 GB RAM (WiredTiger gets ~50%). This is the single most important sizing decision.

---

### Indexes in MongoDB

#### Default _id Index

Every collection automatically has a unique index on `_id`. This is the clustered index — documents are physically organized by `_id` in the B+ Tree.

#### Secondary Indexes

```javascript
// Single field index
db.orders.createIndex({ user_id: 1 })  // 1 = ascending

// Compound index (multi-field)
db.orders.createIndex({ user_id: 1, created_at: -1 })

// Multikey index (on array fields)
db.orders.createIndex({ "items.product_id": 1 })
// Creates one index entry PER array element PER document

// Text index (full-text search)
db.orders.createIndex({ "items.name": "text" })

// TTL index (auto-delete old documents)
db.sessions.createIndex({ created_at: 1 }, { expireAfterSeconds: 3600 })
// Documents deleted 1 hour after created_at — great for sessions, caches

// Partial index (index only matching documents)
db.orders.createIndex(
  { user_id: 1 },
  { partialFilterExpression: { status: "pending" } }
)
// Only indexes pending orders — smaller index, faster queries on pending
```

#### How Secondary Indexes Work

```
Secondary index on user_id (separate B+ Tree):

  B+ Tree:
    "user_100" → [ObjectId("aaa..."), ObjectId("bbb...")]
    "user_123" → [ObjectId("ccc..."), ObjectId("ddd..."), ObjectId("eee...")]
    "user_456" → [ObjectId("fff...")]

Query: db.orders.find({ user_id: "user_123" })

  1. Search secondary index B+ Tree → find "user_123"
  2. Get list of _id values: [ccc, ddd, eee]
  3. For each _id, look up the document in the _id B+ Tree (primary)
  4. Return documents

  This is TWO B+ Tree lookups per document (index + primary).
  Covered query (all fields in the index) skips step 3.
```

#### Covered Queries — The Performance Trick

```javascript
// Index: { user_id: 1, total: 1 }

// NOT covered — needs to fetch full document for "status"
db.orders.find({ user_id: "user_123" }, { total: 1, status: 1 })

// COVERED — all requested fields are in the index
db.orders.find(
  { user_id: "user_123" },
  { _id: 0, user_id: 1, total: 1 }  // _id: 0 excludes _id
)
// MongoDB returns results directly from the index
// Never touches the primary B+ Tree — much faster
```

#### The Explain Plan

```javascript
db.orders.find({ user_id: "user_123" }).explain("executionStats")

// Output (simplified):
{
  "winningPlan": {
    "stage": "FETCH",           // Had to fetch full documents
    "inputStage": {
      "stage": "IXSCAN",       // Used index scan
      "indexName": "user_id_1",
      "direction": "forward",
      "indexBounds": { "user_id": ["user_123", "user_123"] }
    }
  },
  "executionStats": {
    "nReturned": 3,             // 3 documents returned
    "totalKeysExamined": 3,     // 3 index entries scanned
    "totalDocsExamined": 3,     // 3 documents fetched (1:1 = good)
    "executionTimeMillis": 1
  }
}

// Red flags in explain:
// "stage": "COLLSCAN"  → full collection scan, no index used 💀
// totalDocsExamined >> nReturned → index not selective enough
// totalKeysExamined >> nReturned → scanning too many index entries
```

### Interview Tip 💡
> When discussing MongoDB performance, mention: (1) check `explain()` for COLLSCAN, (2) ensure compound indexes match query patterns (ESR rule: Equality → Sort → Range), (3) covered queries avoid document fetches, (4) multikey indexes on arrays create N index entries per document — watch for index bloat.

---

### Replication — Replica Sets

MongoDB uses **replica sets** for high availability: one primary, multiple secondaries.

```
┌──────────┐     sync      ┌──────────────┐
│ PRIMARY  │ ──────────────→│ SECONDARY 1  │
│          │                │ (sync replica)│
│ Reads ✅ │     sync      └──────────────┘
│ Writes ✅│ ──────────────→┌──────────────┐
│          │                │ SECONDARY 2  │
└──────────┘                │ (sync replica)│
                            └──────────────┘
```

#### Oplog — The Replication Log

The primary records every write operation in a capped collection called the **oplog** (operation log):

```javascript
// Oplog entry (simplified)
{
  ts: Timestamp(1710400000, 1),    // timestamp + ordinal
  op: "i",                          // i=insert, u=update, d=delete
  ns: "mydb.orders",               // namespace (db.collection)
  o: { _id: ObjectId("..."), user_id: "user_123", total: 299.99 }
}
```

Secondaries **tail the oplog** continuously, applying operations to stay in sync. This is similar to Postgres WAL replication, but at the logical level (operations) rather than physical level (page changes).

```
Replication flow:
  1. Client writes to primary
  2. Primary applies write + appends to oplog
  3. Secondaries poll primary's oplog for new entries
  4. Secondaries apply oplog entries in order
  5. Secondaries acknowledge to primary

Replication lag = time between primary write and secondary apply
  Healthy: < 1 second
  Concerning: > 10 seconds
  Dangerous: oplog window exceeded → secondary needs full resync 💀
```

#### Write Concern — Tunable Durability

```javascript
// w: 1 (default) — acknowledged by primary only
db.orders.insertOne(doc, { writeConcern: { w: 1 } })
// Fast, but data lost if primary crashes before replication

// w: "majority" — acknowledged by majority of replica set
db.orders.insertOne(doc, { writeConcern: { w: "majority" } })
// Slower, but survives primary failure (data on 2+ nodes)

// w: "majority", j: true — majority + journaled
db.orders.insertOne(doc, { writeConcern: { w: "majority", j: true } })
// Slowest, safest — data on disk on majority of nodes

// w: 0 — fire and forget
db.orders.insertOne(doc, { writeConcern: { w: 0 } })
// Fastest, no acknowledgment, data might be lost
```

#### Read Concern — Tunable Consistency

```javascript
// "local" (default) — read from this node, might see uncommitted data
db.orders.find().readConcern("local")

// "majority" — only read data committed on majority of nodes
db.orders.find().readConcern("majority")
// Won't see data that could be rolled back

// "linearizable" — strongest, read reflects all acknowledged writes
db.orders.find().readConcern("linearizable")
// Equivalent to Postgres default — but much slower in MongoDB
```

#### Read Preference — Where to Read

```javascript
// "primary" (default) — all reads go to primary
// "primaryPreferred" — primary, fallback to secondary
// "secondary" — read from secondaries (eventual consistency)
// "secondaryPreferred" — secondary, fallback to primary
// "nearest" — lowest latency node

db.orders.find().readPref("secondary")
// Reads from secondary — might be slightly behind primary
// Good for analytics queries that don't need latest data
```

### Interview Tip 💡
> The combination of write concern + read concern + read preference gives you a consistency spectrum. For payment systems: `w: "majority"` + `readConcern: "majority"` + `readPref: "primary"`. For analytics dashboards: `w: 1` + `readConcern: "local"` + `readPref: "secondary"`. Know how to tune these for different use cases.
