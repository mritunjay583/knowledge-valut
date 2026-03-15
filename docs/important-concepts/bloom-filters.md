# Bloom Filters — Complete Guide for High-Scale Systems

## Why Do Bloom Filters Exist?

Imagine you have **10 billion URLs** in a database. Every request asks: *"Does this URL exist?"*

The naive approach — query the database every time — kills your system:

- 1 million queries/sec → database becomes the bottleneck
- Most requests might be for **non-existing keys** (bots, random scans)
- Cache miss → DB query, even for items that were never stored

**Caching doesn't fully solve it** because cache misses for non-existent keys still hit the DB.

**Bloom filters exist to cheaply reject impossible queries before touching disk, network, or database.**

---

## What Is a Bloom Filter?

A Bloom filter is a **bit array + multiple hash functions**.

Instead of asking *"Does this item exist?"*, it asks a cheaper question: *"Could this item possibly exist?"*

It answers one of two things:

- **Definitely NOT present** → reject immediately, skip DB
- **Maybe present** → go ahead and query the DB

> The key insight: **fast rejection of impossible items**.

```
Request
  ↓
Bloom Filter
  ↓
NOT present → Reject immediately
Maybe present → Query database
```

---

## How It Works — Step by Step

### Structure

- A **bit array** of size `m`, all initialized to 0
- `k` independent **hash functions**

```
Initial state (m = 10):
Index:  0  1  2  3  4  5  6  7  8  9
Bits:   0  0  0  0  0  0  0  0  0  0
```

### Insert Operation

To insert `"apple"`, compute all k hash positions and set those bits to 1:

```
h1(apple) → 2
h2(apple) → 5
h3(apple) → 7

Result:
Index:  0  1  2  3  4  5  6  7  8  9
Bits:   0  0  1  0  0  1  0  1  0  0
```

Insert `"banana"`:

```
h1(banana) → 3
h2(banana) → 5
h3(banana) → 8

Result:
Index:  0  1  2  3  4  5  6  7  8  9
Bits:   0  0  1  1  0  1  0  1  1  0
```

### Query Operation

**Check "apple":** hash positions → 2, 5, 7 → all bits are 1 → **Maybe present** ✓

**Check "grape":** hash positions → 1, 5, 6 → bit[1] = 0 → **Definitely NOT present** ✗

### Why One Zero Bit Proves Absence

When we insert an element, we **always** set ALL its hash positions to 1.

So if `"grape"` had been inserted, bit[1] **must** be 1. But bit[1] = 0, meaning **nobody ever set this bit**, meaning `"grape"` was never inserted.

> **If ANY bit is 0 → element definitely NOT present.**
> Bits only go 0 → 1, never 1 → 0. Insertion only adds information, never removes it.

---

## False Positives (The Tradeoff)

Sometimes different elements set overlapping bits.

Example: after inserting `"apple"` and `"banana"`, checking `"grape"` might find all its hash positions already set to 1 by other elements.

Bloom filter says "Maybe present" — but `"grape"` was never inserted. This is a **false positive**.

**But false negatives NEVER happen.** If the filter says "Not present", it's guaranteed.

### Why This Tradeoff Is Worth It

With 1000 requests for non-existent keys and 1% false positive rate:

| Outcome | Count |
|---|---|
| Rejected by bloom filter | 990 |
| False positives (unnecessary DB query) | 10 |

**100× reduction in DB load.** That's the deal.

---

## Designing a Bloom Filter — The 4 Parameters

### Step 1: Estimate number of elements (n)

> "How many items will this bloom filter store?"

| System | n |
|---|---|
| URL shortener | 100M URLs |
| Cache filter | 10M keys |
| Crawler dedup | 1B URLs |
| SSTable filter | 1M keys |

**If you underestimate n, the filter becomes useless** (false positives explode).

### Step 2: Choose false positive rate (p)

| False Positive Rate | Usage |
|---|---|
| 10% | Small apps |
| 1% | Most systems |
| 0.1% | Databases |
| 0.01% | Search engines |

### Step 3: Compute bit array size (m)

```
m = -(n × ln(p)) / (ln2)²
```

Example: n = 10M, p = 1%

```
m ≈ 95,850,000 bits ≈ 12 MB
```

12 MB for 10 million keys. Tiny.

### Step 4: Compute number of hash functions (k)

```
k = (m/n) × ln2
```

For our example: k ≈ 7

### Quick Reference Example

| Parameter | Value |
|---|---|
| Elements (n) | 100M |
| False positive rate (p) | 1% |
| Bit array size (m) | ~958M bits ≈ 120 MB |
| Hash functions (k) | 7 |

120 MB to filter 100 million keys. The actual dataset might be 100 GB.

---

## Choosing Hash Functions

You do NOT need k different hash algorithms. Use **double hashing**:

```
h1(x) = murmurhash(key, seed1)
h2(x) = murmurhash(key, seed2)

hash_i(x) = h1(x) + i × h2(x)    for i = 0...k-1
```

This generates k hashes from just two computations.

### Good hash functions

| Hash | Notes |
|---|---|
| MurmurHash3 | Most popular. Used in Cassandra, Kafka |
| xxHash | Even faster |
| CityHash / FarmHash | Developed by Google |

**Avoid** SHA-256 / MD5 — too slow unless you need cryptographic security.

---

## Where Bloom Filters Are Used in Production

### LSM Tree Databases (Cassandra, RocksDB, HBase, LevelDB)

These databases store data in immutable **SSTable** files on disk. A query might need to check 100+ files.

Each SSTable has its own Bloom filter:

```
Query key "user_12345"
  → Bloom filter SSTable1 → NOT PRESENT (skip)
  → Bloom filter SSTable2 → NOT PRESENT (skip)
  → Bloom filter SSTable3 → NOT PRESENT (skip)
  → Bloom filter SSTable4 → MAYBE PRESENT → read this file
```

**1 disk read instead of 100.** That's a 100× improvement.

### Google Bigtable

Stores petabytes of data. Bloom filters per SSTable avoid unnecessary storage scans.

### Web Crawlers

Google crawler checks Bloom filter before crawling a URL. If URL seen before → skip.

### Distributed Caches (Redis + DB)

Instead of cache miss → DB hit, check Bloom filter first. If NOT present → skip DB entirely.

### CDNs

Check whether content exists in edge cache before going to origin server.

---

## Latency Context — Why This Matters

| Operation | Time |
|---|---|
| CPU operation | ~1 ns |
| RAM read | ~100 ns |
| SSD read | ~100 μs |
| Network call | 1–5 ms |

Bloom filters use **RAM only**. Everything else is 1000× slower.

---

## Pros and Cons

### Pros

- **Extremely memory efficient** — tiny memory for huge datasets
- **Extremely fast** — O(k) per operation, k is usually ~7
- **Prevents expensive operations** — disk reads, network calls, DB queries

### Cons

- **False positives** — can say "Maybe present" when item is not
- **No deletion** (classic bloom filter) — bits overlap between elements
- **Requires parameter tuning** — must choose m, k, n carefully or false positives increase

---

## Why Bloom Filters Cannot Delete Elements

When we insert `"apple"` (bits 2, 4, 6) and `"banana"` (bits 1, 4, 6), bits 4 and 6 are **shared**.

If we delete `"apple"` by resetting bits 2, 4, 6 to 0:
- bit 4 and bit 6 go to 0
- `"banana"` now appears as "not present"
- That's a **false negative** — which Bloom filters must NEVER allow

---

## Counting Bloom Filters (Deletion Support)

Replace each bit with a **counter**:

```
Insert "apple" (positions 2, 4, 6):
[0, 0, 1, 0, 1, 0, 1, 0]

Insert "banana" (positions 1, 4, 6):
[0, 1, 1, 0, 2, 0, 2, 0]

Delete "apple" (decrement positions 2, 4, 6):
[0, 1, 0, 0, 1, 0, 1, 0]
```

`"banana"` is still correctly represented.

**Tradeoff:** 4–8 bits per counter instead of 1 bit per cell → 4–8× more memory.

---

## Cuckoo Filters (Modern Alternative)

Named after the cuckoo bird, which kicks other birds' eggs out of nests to lay its own. The filter does the same — if a slot is taken, it kicks the existing item out and relocates it.

### Structure

Instead of a bit array, you have a **table of buckets**. Each bucket holds a few small **fingerprints** (short hashes of the key, like 8 bits).

```
Bucket 0: [ ___ , ___ , ___ , ___ ]
Bucket 1: [ ___ , ___ , ___ , ___ ]
Bucket 2: [ ___ , ___ , ___ , ___ ]
Bucket 3: [ ___ , ___ , ___ , ___ ]
```

### Insert

To insert `"apple"`:

1. Compute a small fingerprint: `fingerprint(apple) = 0xA3`
2. Compute two candidate bucket positions:
   - `bucket_a = hash(apple) % num_buckets → 1`
   - `bucket_b = bucket_a XOR hash(fingerprint) → 3`
3. If bucket 1 or bucket 3 has an empty slot → store `0xA3` there

```
Bucket 0: [ ___ , ___ , ___ , ___ ]
Bucket 1: [ 0xA3, ___ , ___ , ___ ]   ← apple's fingerprint
Bucket 2: [ ___ , ___ , ___ , ___ ]
Bucket 3: [ ___ , ___ , ___ , ___ ]
```

### The "Cuckoo" Part — What If Both Buckets Are Full?

If both candidate buckets are full, we **kick out** an existing fingerprint:

```
1. Pick bucket 1
2. Evict the fingerprint already there (say 0xB7)
3. Store our 0xA3 in its place
4. Now relocate 0xB7 to ITS alternate bucket
5. If that's also full → kick again
6. Repeat until everyone has a home (or give up and resize)
```

Just like the cuckoo bird — push someone else out, they find a new spot.

### Lookup

To check if `"apple"` exists:

1. Compute fingerprint: `0xA3`
2. Compute bucket_a (1) and bucket_b (3)
3. Check if `0xA3` is in either bucket

```
bucket 1 contains 0xA3? → YES → Maybe present
```

Only two bucket lookups. Very fast.

### Delete — Why Cuckoo Filters Exist

To delete `"apple"`:

1. Compute fingerprint: `0xA3`
2. Find it in bucket 1 or bucket 3
3. Simply remove it

```
Bucket 1: [ 0xA3, ___ , ___ , ___ ]
                ↓
Bucket 1: [ ___ , ___ , ___ , ___ ]
```

No counter needed. No risk of breaking other elements because each fingerprint is independent. This is the big advantage over Bloom filters.

### Why XOR for the Second Bucket?

The formula `bucket_b = bucket_a XOR hash(fingerprint)` is **reversible**:

```
bucket_a = bucket_b XOR hash(fingerprint)
```

When we kick out a fingerprint during insertion, we can compute its alternate bucket using only the fingerprint — we don't need the original key. This is what makes the cuckoo relocation possible.

### Cuckoo vs Bloom

| Feature | Bloom Filter | Cuckoo Filter |
|---|---|---|
| Insertion | Yes | Yes |
| Lookup | Yes | Yes |
| Deletion | No | Yes |
| Memory | Good | Better (for same false positive rate) |
| Speed | Fast | Very fast (only 2 bucket lookups) |

### When to choose which

- **Bloom filter** → dataset is static or rarely deleted (SSTables, crawlers, CDN filters)
- **Cuckoo filter** → frequent insert + delete (networking, security, high-churn caches)

---

## The Deep Insight — Probabilistic Data Structures

Bloom filters are part of a bigger family:

| Structure | Used For |
|---|---|
| Bloom Filter | Membership check |
| Counting Bloom Filter | Membership with deletion |
| Cuckoo Filter | Membership with deletion (better) |
| HyperLogLog | Count unique users |
| Count-Min Sketch | Frequency estimation |

All trade **small probability of error for huge performance gain**.

---

## Interview Mental Model

Senior engineers always ask:

> **"Can we reject this request before touching disk?"**

Bloom filters enable exactly that.

```
Request
  ↓
Bloom Filter (RAM — nanoseconds)
  ↓
Disk / DB / Network (milliseconds)
```

### FAANG Interview Answer Template

> "LSM databases use Bloom filters because most SSTables do NOT contain the queried key.
> Bloom filters let the database skip disk reads for files that definitely don't contain the key,
> dramatically reducing I/O."

### Quick Design Answer

> "For 100M keys with 1% false positives:
> bit array = ~120 MB, hash functions = 7, using MurmurHash3 with double hashing."

---

## Python Implementation (Reference)

```python
import mmh3
from bitarray import bitarray
import math

class BloomFilter:
    def __init__(self, n, p):
        self.m = int(-(n * math.log(p)) / (math.log(2) ** 2))
        self.k = int((self.m / n) * math.log(2))
        self.bits = bitarray(self.m)
        self.bits.setall(0)

    def add(self, item):
        h1 = mmh3.hash(item, 42)
        h2 = mmh3.hash(item, 99)
        for i in range(self.k):
            index = (h1 + i * h2) % self.m
            self.bits[index] = 1

    def check(self, item):
        h1 = mmh3.hash(item, 42)
        h2 = mmh3.hash(item, 99)
        for i in range(self.k):
            index = (h1 + i * h2) % self.m
            if self.bits[index] == 0:
                return False  # Definitely not present
        return True  # Maybe present
```

---

## One-Line Summary

> Bloom filters let you prove something does NOT exist using almost no memory, at the cost of occasionally saying "maybe" when the answer is "no".
