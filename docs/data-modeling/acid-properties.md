---
title: "ACID Properties"
layout: default
parent: Data Modeling
nav_order: 2
---

# ACID Properties — Deep Dive

> A comprehensive guide to ACID — what each property actually means at the implementation level, how Postgres enforces them, what happens when they break, and why they matter for distributed systems. Written for system design interviews at Meta/Google level.

---

## What is ACID?

ACID is a set of four properties that guarantee database transactions are processed reliably. Without ACID, your data corrupts, money disappears, and inventory goes negative.

| Property | Question It Answers |
|----------|-------------------|
| **Atomicity** | Did the entire operation succeed or fail as one unit? |
| **Consistency** | Is the data valid according to all defined rules? |
| **Isolation** | Can concurrent transactions interfere with each other? |
| **Durability** | If the server crashes right after commit, is my data safe? |

```
Real-world analogy — Bank Transfer ($500 from Alice to Bob):

  Step 1: Deduct $500 from Alice's account
  Step 2: Add $500 to Bob's account

  Atomicity:  Both steps happen, or neither does. Never just Step 1.
  Consistency: Total money in the system stays the same (Alice + Bob = constant).
  Isolation:  A third person checking balances mid-transfer sees either
              the before state or the after state, never the in-between.
  Durability: Once the transfer is confirmed, a power failure doesn't undo it.
```

---

## Atomicity — All or Nothing

### What It Means

A transaction is an indivisible unit. If any part fails, the entire transaction is rolled back — as if it never happened.

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 500 WHERE id = 1;  -- Alice
  UPDATE accounts SET balance = balance + 500 WHERE id = 2;  -- Bob
COMMIT;

-- If the server crashes between the two UPDATEs:
-- WITHOUT atomicity: Alice loses $500, Bob gets nothing 💀
-- WITH atomicity: Both changes are rolled back. No money lost. ✅
```

### How Postgres Implements Atomicity — Write-Ahead Log (WAL)

Postgres uses a **Write-Ahead Log** (WAL) to guarantee atomicity. The rule: **write the log BEFORE writing the data**.

```
Transaction: Transfer $500 from Alice to Bob

Step 1: Write to WAL (on disk):
  WAL Record 1: "BEGIN txn 42"
  WAL Record 2: "UPDATE accounts SET balance=500 WHERE id=1 (was 1000)"
  WAL Record 3: "UPDATE accounts SET balance=1500 WHERE id=2 (was 1000)"
  WAL Record 4: "COMMIT txn 42"

Step 2: Apply changes to actual data pages (can happen later):
  Page for Alice: balance = 500
  Page for Bob: balance = 1500
```

**Crash scenarios:**

```
Crash BEFORE WAL Record 4 (COMMIT):
  → On recovery, Postgres reads WAL
  → Sees txn 42 has no COMMIT record
  → Rolls back all changes from txn 42
  → Alice still has $1000, Bob still has $1000 ✅

Crash AFTER WAL Record 4 (COMMIT):
  → On recovery, Postgres reads WAL
  → Sees txn 42 has a COMMIT record
  → Replays any changes not yet written to data pages
  → Alice has $500, Bob has $1500 ✅
```

### WAL Internals

```
WAL on disk (sequential writes — very fast):
  ┌──────────────────────────────────────────────────┐
  │ LSN 1: BEGIN txn 42                              │
  │ LSN 2: UPDATE page 5, offset 3 (old→new values) │
  │ LSN 3: UPDATE page 8, offset 1 (old→new values) │
  │ LSN 4: COMMIT txn 42                             │
  │ LSN 5: BEGIN txn 43                              │
  │ ...                                              │
  └──────────────────────────────────────────────────┘

Data pages in shared buffers (RAM):
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │  Page 5  │  │  Page 8  │  │  Page 12 │
  │ (dirty)  │  │ (dirty)  │  │ (clean)  │
  └──────────┘  └──────────┘  └──────────┘
       ↓              ↓
  Written to disk later by background writer / checkpointer
```

**LSN (Log Sequence Number):** Every WAL record has a unique, monotonically increasing LSN. Each data page tracks the LSN of the last WAL record applied to it. During recovery, Postgres only replays WAL records with LSN > page's LSN.

**Checkpoint:** Periodically, Postgres flushes all dirty pages to disk and records a checkpoint in the WAL. Recovery only needs to replay from the last checkpoint, not from the beginning of time.

### Interview Tip 💡
> When asked "how does a database guarantee atomicity?", the answer is WAL (Write-Ahead Logging). The key insight: sequential writes to a log file are much faster than random writes to data pages. WAL makes transactions fast AND safe. This same technique is used by MySQL (redo log), SQLite (journal), and even Kafka (commit log).

---

## Consistency — Valid State to Valid State

### What It Means

A transaction moves the database from one valid state to another. "Valid" means all defined rules (constraints) are satisfied.

**This is the most misunderstood ACID property.** It has two aspects:

### 1. Database-Level Consistency (Enforced by the DB)

```sql
-- Constraints define what "valid" means:
CREATE TABLE accounts (
    id      SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),     -- FK: user must exist
    balance DECIMAL(10,2) CHECK (balance >= 0), -- No negative balances
    type    VARCHAR(20) NOT NULL               -- Can't be null
);

-- This transaction violates consistency:
BEGIN;
  UPDATE accounts SET balance = -100 WHERE id = 1;
  -- CHECK constraint violated → transaction ABORTED
  -- Database remains in valid state ✅
COMMIT;  -- never reached
```

**Constraints Postgres enforces:**
- `PRIMARY KEY` — uniqueness + not null
- `FOREIGN KEY` — referential integrity (referenced row must exist)
- `UNIQUE` — no duplicate values
- `NOT NULL` — column must have a value
- `CHECK` — custom boolean expression
- `EXCLUDE` — no overlapping ranges (useful for scheduling)

### 2. Application-Level Consistency (YOUR responsibility)

The database can't enforce business rules it doesn't know about:

```sql
-- Business rule: "Total money in the system must be constant"
-- The database doesn't know this rule!

BEGIN;
  UPDATE accounts SET balance = balance - 500 WHERE id = 1;
  -- Oops, forgot to add $500 to Bob
COMMIT;
-- Database is "consistent" (no constraint violated)
-- But business logic is BROKEN — $500 vanished 💀
```

**This is why consistency is partly the application's job.** The database provides the tools (constraints, transactions), but you must use them correctly.

### How Postgres Enforces Constraints

```
During transaction execution:

1. IMMEDIATE constraints (default):
   Checked after each statement within the transaction.
   
   INSERT INTO orders (user_id, total) VALUES (999, 50.00);
   -- FK check: does user 999 exist? No → ERROR immediately

2. DEFERRED constraints:
   Checked at COMMIT time.
   
   SET CONSTRAINTS ALL DEFERRED;
   BEGIN;
     INSERT INTO orders (user_id, total) VALUES (999, 50.00);  -- OK for now
     INSERT INTO users (id, name) VALUES (999, 'Alice');        -- Create user
   COMMIT;  -- FK check passes now ✅
```

### Interview Tip 💡
> In interviews, distinguish between database consistency (constraints) and application consistency (business rules). Then connect it to distributed systems: in the CAP theorem, "C" means something different — linearizability (all nodes see the same data at the same time). These are three different meanings of "consistency" and interviewers love testing if you know the difference.

---

## Isolation — Concurrent Transactions Don't Interfere

### What It Means

Multiple transactions running simultaneously should behave as if they ran one after another (serially). In practice, full serialization is too slow, so databases offer **isolation levels** — tradeoffs between correctness and performance.

### The Concurrency Problems (Anomalies)

#### 1. Dirty Read
Reading data written by a transaction that hasn't committed yet.

```
Transaction A:                      Transaction B:
BEGIN;                              BEGIN;
UPDATE accounts SET balance = 0     
WHERE id = 1;                       
                                    SELECT balance FROM accounts
                                    WHERE id = 1;
                                    -- Reads 0 (dirty — A hasn't committed!)
ROLLBACK;  -- A rolls back!         
                                    -- B made a decision based on data
                                    -- that never actually existed 💀
```

#### 2. Non-Repeatable Read
Reading the same row twice in a transaction and getting different values.

```
Transaction A:                      Transaction B:
BEGIN;                              
SELECT balance FROM accounts        
WHERE id = 1;  -- reads $1000      
                                    BEGIN;
                                    UPDATE accounts SET balance = 500
                                    WHERE id = 1;
                                    COMMIT;
SELECT balance FROM accounts        
WHERE id = 1;  -- reads $500 😬    
-- Same query, different result within the same transaction!
COMMIT;
```

#### 3. Phantom Read
A query returns different ROWS when executed twice (new rows appeared).

```
Transaction A:                      Transaction B:
BEGIN;                              
SELECT COUNT(*) FROM orders         
WHERE status = 'pending';           
-- returns 5                        
                                    BEGIN;
                                    INSERT INTO orders (status)
                                    VALUES ('pending');
                                    COMMIT;
SELECT COUNT(*) FROM orders         
WHERE status = 'pending';           
-- returns 6 😬 (phantom row!)     
COMMIT;
```

#### 4. Write Skew
Two transactions read the same data, make decisions based on it, and write — but the combined result violates a constraint.

```
-- Rule: At least 1 doctor must be on call at all times
-- Currently: Alice and Bob are both on call

Transaction A:                      Transaction B:
BEGIN;                              BEGIN;
SELECT COUNT(*) FROM doctors        SELECT COUNT(*) FROM doctors
WHERE on_call = true;               WHERE on_call = true;
-- 2 doctors on call                -- 2 doctors on call
-- "OK, I can remove Alice"         -- "OK, I can remove Bob"
UPDATE doctors SET on_call = false  UPDATE doctors SET on_call = false
WHERE name = 'Alice';               WHERE name = 'Bob';
COMMIT;                             COMMIT;

-- Result: ZERO doctors on call 💀
-- Each transaction was individually valid, but together they broke the rule
```

### Isolation Levels

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Write Skew |
|-------|-----------|-------------------|-------------|-----------|
| Read Uncommitted | ✅ Possible | ✅ Possible | ✅ Possible | ✅ Possible |
| Read Committed | ❌ Prevented | ✅ Possible | ✅ Possible | ✅ Possible |
| Repeatable Read | ❌ Prevented | ❌ Prevented | ✅ Possible* | ✅ Possible* |
| Serializable | ❌ Prevented | ❌ Prevented | ❌ Prevented | ❌ Prevented |

*Postgres's Repeatable Read actually prevents phantom reads too (it uses snapshot isolation), but write skew is still possible.

### How Postgres Implements Each Level

#### Read Committed (Postgres Default)

```
Implementation: Each STATEMENT sees a new snapshot of committed data.

Transaction A:                      Transaction B:
BEGIN;                              BEGIN;
SELECT * FROM accounts;             
-- Snapshot at statement time       
-- Sees Alice: $1000               
                                    UPDATE accounts SET balance = 500
                                    WHERE name = 'Alice';
                                    COMMIT;
SELECT * FROM accounts;             
-- NEW snapshot (new statement)     
-- Sees Alice: $500 ← changed!     
COMMIT;
```

**Mechanism:** Before each statement, Postgres takes a snapshot of all committed transaction IDs. The statement only sees rows created by transactions in that snapshot.

#### Repeatable Read (Snapshot Isolation)

```
Implementation: The ENTIRE TRANSACTION sees one snapshot, taken at the first statement.

Transaction A:                      Transaction B:
BEGIN;                              BEGIN;
SET TRANSACTION ISOLATION LEVEL 
REPEATABLE READ;
SELECT * FROM accounts;             
-- Snapshot taken NOW               
-- Sees Alice: $1000               
                                    UPDATE accounts SET balance = 500
                                    WHERE name = 'Alice';
                                    COMMIT;
SELECT * FROM accounts;             
-- SAME snapshot as before          
-- Still sees Alice: $1000 ✅       
COMMIT;
```

**Mechanism:** Snapshot taken at the start of the first statement. All subsequent statements in the transaction use the same snapshot. If another transaction modifies a row this transaction wants to update, Postgres aborts with a serialization error.

```sql
-- Repeatable Read conflict detection:
-- Transaction A reads row, Transaction B updates and commits it,
-- Transaction A tries to update the same row:

ERROR: could not serialize access due to concurrent update
-- Transaction A must retry
```

#### Serializable (SSI — Serializable Snapshot Isolation)

```
Implementation: Snapshot Isolation + conflict detection for read-write dependencies.

Postgres tracks:
  - Which transactions read which data (rw-dependencies)
  - Which transactions wrote which data
  - If a cycle is detected in the dependency graph → abort one transaction
```

```
The doctor on-call example with Serializable:

Transaction A:                      Transaction B:
BEGIN;                              BEGIN;
SET TRANSACTION ISOLATION LEVEL     SET TRANSACTION ISOLATION LEVEL
SERIALIZABLE;                       SERIALIZABLE;

SELECT COUNT(*) FROM doctors        SELECT COUNT(*) FROM doctors
WHERE on_call = true;               WHERE on_call = true;
-- Postgres notes: A read doctors   -- Postgres notes: B read doctors

UPDATE doctors SET on_call = false  UPDATE doctors SET on_call = false
WHERE name = 'Alice';               WHERE name = 'Bob';
-- Postgres notes: A wrote doctors  -- Postgres notes: B wrote doctors

COMMIT; ✅                          COMMIT;
                                    -- ERROR: could not serialize access
                                    -- due to read/write dependencies
                                    -- among transactions ❌
                                    -- B must retry!
```

**SSI detects the dangerous pattern:**
- A read data that B wrote (and vice versa)
- This forms a cycle in the dependency graph
- One transaction must be aborted to prevent the anomaly

### Performance Impact of Isolation Levels

```
Read Committed:     Fastest. No snapshot overhead per transaction.
                    Each statement is independent.
                    Used by: 90% of production systems.

Repeatable Read:    Moderate overhead. One snapshot per transaction.
                    Possible serialization errors (must retry).
                    Used by: Financial systems needing consistent reads.

Serializable:       Highest overhead. Tracks all read/write dependencies.
                    More serialization errors (more retries).
                    Used by: Systems where correctness > performance.
```

### Interview Tip 💡
> The key insight for interviews: Postgres doesn't use locks for isolation (unlike MySQL's InnoDB which uses row-level locks + gap locks). Postgres uses MVCC + snapshots. This means readers never block writers and writers never block readers. The tradeoff is table bloat (old row versions) and the need for VACUUM. When asked "how does Postgres handle concurrent access?", say MVCC — it's the most important implementation detail.

---

## Durability — Committed Data Survives Crashes

### What It Means

Once a transaction is committed, the data is permanently saved — even if the server crashes, loses power, or the disk fails one second later.

### How Postgres Guarantees Durability

```
COMMIT execution:

1. Write all WAL records for this transaction to WAL buffer (in RAM)
2. fsync() the WAL buffer to disk ← THIS is the durability guarantee
3. Return "COMMIT OK" to the client
4. (Later) Background writer flushes dirty data pages to disk

The critical moment is step 2. Once WAL is on disk, the data is safe.
Even if the server crashes before step 4, recovery replays the WAL.
```

### fsync — The Real Guarantee

```
Without fsync:
  Application → write() → OS page cache (RAM) → "done!"
  But data is still in RAM! Power failure = data loss 💀

With fsync:
  Application → write() → OS page cache → fsync() → disk platters → "done!"
  Data is physically on disk. Power failure = data safe ✅
```

**Postgres calls `fsync()` on every COMMIT by default** (`synchronous_commit = on`). This is why commits have latency — they wait for the disk.

### Durability Tradeoffs

```
synchronous_commit = on (default):
  Every COMMIT waits for WAL fsync.
  Safest. ~1-5ms latency per commit.
  Zero data loss on crash.

synchronous_commit = off:
  COMMIT returns immediately, WAL fsynced in background (~200ms later).
  Faster commits. Risk: lose up to ~200ms of transactions on crash.
  Used for: non-critical data (session logs, analytics events).

full_page_writes = on (default):
  After each checkpoint, first modification to a page writes the ENTIRE page
  to WAL (not just the changed bytes). Protects against partial page writes
  (torn pages) if crash happens mid-write.
```

### Replication for Durability Beyond Single Machine

Single-machine durability protects against process crashes and power failures. But what about disk failure or machine destruction?

```
Synchronous Replication:
  Primary → WAL → fsync to local disk
                → send to replica → replica fsync → ACK
                → COMMIT OK to client
  
  Data exists on 2+ machines before commit confirmed.
  Survives single machine failure.
  Cost: higher latency (network round trip to replica).

Asynchronous Replication:
  Primary → WAL → fsync to local disk → COMMIT OK to client
                → send to replica (eventually)
  
  Faster commits, but replica may lag.
  If primary dies, some committed transactions may be lost.
```

### Interview Tip 💡
> Durability seems simple but has nuance. The key question is: "durable to what failure?" Process crash → WAL on local disk is enough. Machine failure → need replication. Data center failure → need cross-region replication. Each level adds latency. In interviews, mention this spectrum and let the interviewer choose the level they care about.

---

## ACID in Practice — Postgres Transaction Example

Let's trace a complete transaction through all four properties:

```sql
-- Scenario: E-commerce checkout
-- 1. Deduct inventory
-- 2. Create order
-- 3. Charge payment

BEGIN;

-- Check inventory (Isolation: snapshot sees consistent state)
SELECT quantity FROM products WHERE id = 42 FOR UPDATE;
-- FOR UPDATE: locks this row so no other transaction can modify it
-- Returns: quantity = 5

-- Deduct inventory (Consistency: CHECK constraint ensures quantity >= 0)
UPDATE products SET quantity = quantity - 2 WHERE id = 42;
-- quantity is now 3

-- Create order (Consistency: FK ensures product exists)
INSERT INTO orders (user_id, product_id, quantity, total)
VALUES (123, 42, 2, 59.98);

-- Record payment (Atomicity: if this fails, everything rolls back)
INSERT INTO payments (order_id, amount, status)
VALUES (currval('orders_id_seq'), 59.98, 'completed');

COMMIT;
-- Durability: WAL fsynced to disk. Data survives crash.
```

**What happens if the server crashes at each point:**

```
Crash before COMMIT:
  → WAL has no COMMIT record for this transaction
  → Recovery rolls back all changes
  → Inventory still 5, no order, no payment ✅ (Atomicity)

Crash after COMMIT:
  → WAL has COMMIT record
  → Recovery replays any un-flushed changes
  → Inventory is 3, order exists, payment recorded ✅ (Durability)

Another transaction tries to buy the same product simultaneously:
  → FOR UPDATE lock blocks it until this transaction completes
  → Or with MVCC: it sees the old quantity (5) but gets a
    serialization error when trying to update ✅ (Isolation)

Someone tries to set quantity to -1:
  → CHECK (quantity >= 0) prevents it
  → Transaction aborted ✅ (Consistency)
```

---

## ACID vs BASE — The Spectrum

Not all systems need full ACID. Distributed NoSQL systems often use BASE:

| | ACID | BASE |
|--|------|------|
| Full name | Atomicity, Consistency, Isolation, Durability | Basically Available, Soft state, Eventually consistent |
| Philosophy | Correctness first | Availability first |
| Consistency | Strong (immediate) | Eventual (converges over time) |
| Transactions | Multi-statement, all-or-nothing | Single-record or none |
| Scaling | Vertical (bigger machine) | Horizontal (more machines) |
| Use case | Financial, inventory, booking | Social feeds, analytics, caching |
| Examples | Postgres, MySQL, Oracle | Cassandra, DynamoDB, MongoDB* |

*MongoDB added multi-document ACID transactions in v4.0, but they're expensive.

### The CAP Connection

```
CAP Theorem: In a distributed system, you can only guarantee 2 of 3:
  C — Consistency (all nodes see the same data)
  A — Availability (every request gets a response)
  P — Partition tolerance (system works despite network failures)

Network partitions WILL happen, so you really choose between:
  CP: Consistent + Partition tolerant (sacrifice availability)
      → Postgres with synchronous replication
      → When partition happens: refuse writes to maintain consistency
  
  AP: Available + Partition tolerant (sacrifice consistency)
      → Cassandra, DynamoDB
      → When partition happens: accept writes on both sides, merge later
```

### Interview Tip 💡
> Don't say "SQL = ACID, NoSQL = BASE." It's a spectrum. Postgres can be configured for eventual consistency (async replication). DynamoDB can do strongly consistent reads. CockroachDB and Spanner are distributed AND ACID. The real question is: what consistency guarantees does your use case need, and what are you willing to pay in latency and availability?

---

## When ACID Matters Most

| Scenario | Why ACID is Critical |
|----------|---------------------|
| Financial transactions | Money can't appear or disappear |
| Inventory management | Can't sell more items than you have |
| Booking systems | Can't double-book the same seat |
| User registration | Can't have duplicate accounts |
| Order processing | Order, payment, and inventory must be atomic |

| Scenario | Where ACID is Overkill |
|----------|----------------------|
| Social media likes | Off by one like doesn't matter |
| View counters | Approximate is fine |
| Activity feeds | Eventual consistency is acceptable |
| Session storage | Losing a session is annoying, not catastrophic |
| Analytics events | Losing 0.01% of events is acceptable |

---

## Common Interview Questions

### Q: Explain ACID with a real example
Use the bank transfer: debit Alice, credit Bob. Walk through each property and what goes wrong without it. Then mention WAL for atomicity/durability, MVCC for isolation, constraints for consistency.

### Q: What isolation level would you use for a payment system?
Serializable or Repeatable Read with explicit locking (`SELECT ... FOR UPDATE`). Explain that Read Committed allows non-repeatable reads which could cause double-charging. Mention the retry logic needed for serialization errors.

### Q: How does MVCC work?
Each row has xmin/xmax transaction IDs. Each transaction has a snapshot of active transaction IDs. A row is visible if xmin is committed and in the snapshot, and xmax is not committed or not in the snapshot. Old versions are cleaned up by VACUUM.

### Q: What's the difference between optimistic and pessimistic concurrency?
- **Pessimistic (locking):** `SELECT ... FOR UPDATE` — lock the row, prevent others from modifying it. Good when conflicts are frequent.
- **Optimistic (MVCC/versioning):** Read without locking, detect conflicts at write time. Good when conflicts are rare. Postgres's MVCC is optimistic for reads, can be pessimistic with explicit locks.

### Q: How would you handle a distributed transaction across two databases?
- **Two-Phase Commit (2PC):** Coordinator asks all participants to prepare, then commit. Blocks if coordinator crashes.
- **Saga pattern:** Break into local transactions with compensating actions. If step 3 fails, run compensating actions for steps 1 and 2.
- **Outbox pattern:** Write to local DB + outbox table in one transaction. Background process publishes outbox events. Guarantees at-least-once delivery.
- In practice, avoid distributed transactions. Design services to own their data and use eventual consistency with sagas.

---

## Further Reading
- SQL Databases — storage internals, indexing, query execution
- SQL vs NoSQL — choosing the right database
- HTTP — status codes and headers used in transactional APIs
- Common API Patterns — idempotency keys for safe retries

---

*Last updated: 2025-03-14*
