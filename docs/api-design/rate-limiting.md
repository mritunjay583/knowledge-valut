---
title: "Rate Limiting"
layout: default
parent: API Design
nav_order: 6
---

# Rate Limiting

> A comprehensive guide to rate limiting — algorithms, implementation strategies, distributed rate limiting, and production patterns. Written for system design interviews and building resilient, scalable APIs.

---

## What is Rate Limiting?

Rate limiting controls how many requests a client can make to your API within a given time window. It protects your system from abuse, ensures fair usage, and prevents cascading failures.

```
Without rate limiting:
  Attacker → 1,000,000 requests/second → Your API → 💀 Server crash

With rate limiting:
  Attacker → 1,000,000 requests/second → Rate Limiter → 429 Too Many Requests
  Legitimate user → 10 requests/second → Rate Limiter → 200 OK ✅
```

---

## Why Rate Limiting?

| Threat | Without Rate Limiting | With Rate Limiting |
|--------|----------------------|-------------------|
| DDoS attack | Server overwhelmed, all users affected | Attacker blocked, legitimate users served |
| Brute force login | Attacker tries millions of passwords | Limited to 5 attempts/minute |
| Scraping | Competitor scrapes all your data | Throttled to reasonable speed |
| Accidental abuse | Buggy client sends infinite loop | Client gets 429, bug is surfaced |
| Resource fairness | One heavy user starves others | Everyone gets fair share |
| Cost control | Runaway usage = massive cloud bill | Usage capped, costs predictable |

---

## Rate Limiting Algorithms

### 1. Fixed Window Counter

Divide time into fixed windows (e.g., 1-minute intervals). Count requests per window.

```
Window: 12:00:00 - 12:00:59 → Limit: 100 requests
  Request 1   → Count: 1   → ✅ Allowed
  Request 50  → Count: 50  → ✅ Allowed
  Request 100 → Count: 100 → ✅ Allowed
  Request 101 → Count: 101 → ❌ 429 Too Many Requests

Window: 12:01:00 - 12:01:59 → Counter resets to 0
  Request 1   → Count: 1   → ✅ Allowed
```

**Implementation:**
```python
def is_allowed(user_id, limit=100, window_seconds=60):
    key = f"rate:{user_id}:{int(time.time() / window_seconds)}"
    count = redis.incr(key)
    if count == 1:
        redis.expire(key, window_seconds)
    return count <= limit
```

**Pros:** Simple, low memory (one counter per user per window)
**Cons:** **Burst at window boundary** — user can send 100 requests at 12:00:59 and 100 more at 12:01:00 = 200 requests in 2 seconds

### 2. Sliding Window Log

Store the timestamp of every request. Count requests within the sliding window.

```
Limit: 100 requests per 60 seconds
Current time: 12:01:30

Check: How many requests between 12:00:30 and 12:01:30?
  → Count timestamps in that range
  → If < 100 → ✅ Allowed
  → If >= 100 → ❌ Rejected
```

**Implementation:**
```python
def is_allowed(user_id, limit=100, window_seconds=60):
    now = time.time()
    key = f"rate:{user_id}"
    
    # Remove old entries outside the window
    redis.zremrangebyscore(key, 0, now - window_seconds)
    
    # Count entries in window
    count = redis.zcard(key)
    
    if count < limit:
        redis.zadd(key, {str(now): now})
        redis.expire(key, window_seconds)
        return True
    return False
```

**Pros:** Precise, no boundary burst problem
**Cons:** **High memory** — stores every request timestamp. 1M users × 100 requests = 100M entries

### 3. Sliding Window Counter (Hybrid)

Combines fixed window efficiency with sliding window accuracy. Estimates the count using weighted averages of current and previous windows.

```
Previous window (12:00-12:01): 80 requests
Current window  (12:01-12:02): 30 requests
Current time: 12:01:15 (25% into current window)

Estimated count = (previous × overlap%) + current
                = (80 × 75%) + 30
                = 60 + 30 = 90

Limit: 100 → ✅ Allowed (90 < 100)
```

**Pros:** Low memory (two counters), good accuracy, no boundary burst
**Cons:** Approximate (not exact), slightly more complex

### 4. Token Bucket

A bucket holds tokens. Each request consumes a token. Tokens are added at a fixed rate. If the bucket is empty, requests are rejected.

```
Bucket capacity: 10 tokens
Refill rate: 1 token/second

Time 0:  Bucket has 10 tokens
  5 requests → 5 tokens consumed → Bucket: 5 tokens
  
Time 5s: 5 tokens refilled → Bucket: 10 tokens (capped at capacity)
  12 requests → 10 allowed, 2 rejected → Bucket: 0 tokens

Time 6s: 1 token refilled → Bucket: 1 token
  1 request → ✅ Allowed → Bucket: 0 tokens
```

**Implementation:**
```python
def is_allowed(user_id, capacity=10, refill_rate=1):
    key = f"bucket:{user_id}"
    now = time.time()
    
    bucket = redis.hgetall(key)
    tokens = float(bucket.get('tokens', capacity))
    last_refill = float(bucket.get('last_refill', now))
    
    # Refill tokens based on elapsed time
    elapsed = now - last_refill
    tokens = min(capacity, tokens + elapsed * refill_rate)
    
    if tokens >= 1:
        tokens -= 1
        redis.hset(key, mapping={'tokens': tokens, 'last_refill': now})
        redis.expire(key, int(capacity / refill_rate) + 1)
        return True
    return False
```

**Pros:** Allows controlled bursts (up to bucket capacity), smooth rate limiting
**Cons:** Slightly more complex, two values to store per user

### 5. Leaky Bucket

Requests enter a queue (bucket). They're processed at a fixed rate. If the queue is full, new requests are dropped.

```
Queue capacity: 10
Processing rate: 1 request/second

Requests arrive in burst: 15 requests
  → 10 enter the queue
  → 5 are dropped (429)
  → Queue processes 1 per second
```

**Pros:** Smooth, constant output rate (great for APIs with strict throughput limits)
**Cons:** Doesn't allow bursts (even legitimate ones), adds latency (queuing)

### Algorithm Comparison

| Algorithm | Memory | Accuracy | Burst Handling | Complexity |
|-----------|--------|----------|----------------|------------|
| Fixed Window | Low | Low (boundary burst) | ❌ Double burst at boundary | Simple |
| Sliding Window Log | High | Exact | ✅ No burst issue | Medium |
| Sliding Window Counter | Low | Good (approximate) | ✅ Mostly smooth | Medium |
| Token Bucket | Low | Good | ✅ Controlled bursts | Medium |
| Leaky Bucket | Low | Good | ❌ No bursts allowed | Medium |

### Interview Tip 💡
> **Token bucket** is the most commonly asked and most widely used in production (used by AWS, Stripe, and most API gateways). It's the best default answer. Mention that it allows controlled bursts (good UX) while maintaining an average rate limit. If asked about alternatives, explain the fixed window boundary problem and how sliding window solves it.

---

## Rate Limiting Dimensions

### What to Limit By

| Dimension | Use Case | Example |
|-----------|----------|---------|
| User/API Key | Fair usage per customer | 1000 req/hour per user |
| IP Address | Unauthenticated abuse prevention | 100 req/hour per IP |
| Endpoint | Protect expensive operations | 10 POST /bookings per minute |
| Global | Protect overall system capacity | 50K req/second total |
| Tenant | Multi-tenant SaaS fair usage | 10K req/hour per organization |

### Tiered Rate Limits
```
Free tier:    100 requests/hour
Basic tier:   1,000 requests/hour
Pro tier:     10,000 requests/hour
Enterprise:   100,000 requests/hour (or custom)
```

### Per-Endpoint Limits
```
GET  /events         → 100 req/min  (reads are cheap)
POST /bookings       → 10 req/min   (writes are expensive)
POST /login          → 5 req/min    (prevent brute force)
POST /payments       → 3 req/min    (prevent fraud)
GET  /search         → 30 req/min   (expensive query)
```

---

## Response Headers

When rate limiting, always tell the client about their limits:

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100          # Max requests allowed in window
X-RateLimit-Remaining: 42       # Requests remaining in current window
X-RateLimit-Reset: 1710003600   # Unix timestamp when window resets

# When limit exceeded:
HTTP/1.1 429 Too Many Requests
Retry-After: 30                 # Seconds until client can retry
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1710003600

{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Too many requests. Please retry after 30 seconds.",
    "retry_after": 30
  }
}
```

---

## Distributed Rate Limiting

### The Problem
With multiple API servers, each server has its own counter. A user can bypass limits by hitting different servers.

```
Limit: 100 req/min

Server 1: User A → 100 requests → "limit reached"
Server 2: User A → 100 requests → "limit reached"
Server 3: User A → 100 requests → "limit reached"

Total: 300 requests 😬 (should be 100)
```

### Solution: Centralized Counter (Redis)

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│ Server 1 │     │ Server 2 │     │ Server 3 │
└────┬─────┘     └────┬─────┘     └────┬─────┘
     │                │                │
     └────────┬───────┴────────┬───────┘
              │                │
         ┌────┴────────────────┴────┐
         │         Redis            │
         │  rate:user_123 → 85      │
         │  (single source of truth)│
         └──────────────────────────┘
```

All servers check/increment the same counter in Redis. Redis is fast enough (~100K ops/sec) for this.

### Redis + Lua for Atomicity
```lua
-- Atomic rate limit check in Redis (Lua script)
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local current = redis.call('INCR', key)
if current == 1 then
    redis.call('EXPIRE', key, window)
end

if current > limit then
    return 0  -- rejected
else
    return 1  -- allowed
end
```

### What If Redis Goes Down?

Options:
1. **Fail open** — allow all requests (risk: no rate limiting temporarily)
2. **Fail closed** — reject all requests (risk: legitimate users blocked)
3. **Local fallback** — each server maintains local counters (risk: limits are per-server, not global)

**Most systems fail open** — a brief period without rate limiting is better than blocking all users.

---

## Where to Implement Rate Limiting

```
┌────────┐     ┌─────────┐     ┌──────────────┐     ┌──────────┐
│ Client │ ──→ │   CDN   │ ──→ │ API Gateway  │ ──→ │ App      │
│        │     │(L3/L4)  │     │ (L7)         │     │ Server   │
└────────┘     └─────────┘     └──────────────┘     └──────────┘
                  ↑                    ↑                   ↑
              DDoS protection    Per-user/key limits   Business logic
              IP-based limits    Per-endpoint limits    limits
```

| Layer | What It Handles | Tools |
|-------|----------------|-------|
| CDN/Edge | DDoS, IP-based blocking | Cloudflare, AWS Shield |
| API Gateway | Per-user, per-key, per-endpoint limits | Kong, AWS API Gateway, Envoy |
| Application | Business logic limits (e.g., 3 bookings/day) | Custom middleware |

### Interview Tip 💡
> In interviews, say "I'd implement rate limiting at the API gateway level using a token bucket algorithm with Redis as the centralized counter." Then mention per-user and per-endpoint limits. This covers 90% of what interviewers want to hear. If they ask about DDoS, mention CDN-level protection (Cloudflare).

---

## Real-World Production Examples

### 1. Stripe
- Per-API-key rate limits (100 req/sec for live mode)
- Separate limits for read vs write operations
- Returns `Retry-After` header with 429 responses
- Token bucket algorithm

### 2. GitHub API
- 5,000 requests/hour for authenticated users
- 60 requests/hour for unauthenticated (by IP)
- Returns `X-RateLimit-*` headers on every response
- Separate limits for search API (30 req/min)

### 3. Twitter/X API
- Tiered by API access level (Free, Basic, Pro, Enterprise)
- Per-endpoint limits (e.g., 300 tweets/3 hours for posting)
- 15-minute sliding windows
- App-level and user-level limits

### 4. AWS API Gateway
- Default: 10,000 requests/second per region
- Configurable per-API, per-stage, per-method
- Token bucket algorithm
- Burst capacity for short spikes

### 5. Google Cloud APIs
- Per-project quotas (e.g., 100 requests/100 seconds)
- Per-user quotas within projects
- Quota increase requests for higher limits
- Exponential backoff recommended for 429 responses

---

## Common Interview Questions

### Q: Design a rate limiter for a distributed system
1. **Algorithm:** Token bucket (allows bursts, simple to implement)
2. **Storage:** Redis (centralized, fast, atomic operations with Lua)
3. **Dimensions:** Per-user + per-endpoint
4. **Response:** 429 with `Retry-After` and `X-RateLimit-*` headers
5. **Failure mode:** Fail open if Redis is down
6. **Placement:** API gateway layer

### Q: How do you handle rate limiting with multiple data centers?
- **Option 1:** Global Redis cluster (cross-region replication) — accurate but adds latency
- **Option 2:** Per-region rate limiting — faster but user can get 2x limit by hitting both regions
- **Option 3:** Eventual consistency — local counters synced periodically — good balance
- Most systems use Option 2 with slightly lower per-region limits

### Q: Fixed window vs sliding window vs token bucket?
- **Fixed window:** Simple but has boundary burst problem (200 requests in 2 seconds at window edge)
- **Sliding window:** Accurate but high memory (stores all timestamps)
- **Token bucket:** Best balance — allows controlled bursts, low memory, widely used in production

### Q: How do you prevent a single user from consuming all resources?
- Per-user rate limits (not just global)
- Tiered limits based on plan/role
- Separate limits for expensive operations (writes, search)
- Queue-based processing for heavy operations (async with 202 Accepted)
- Circuit breaker pattern for downstream service protection

---

## Further Reading
- api-security — broader security considerations
- authentication-and-authorization — identifying who to rate limit
- common-api-patterns — other API patterns (pagination, idempotency)
- http — HTTP 429 status code, Retry-After header
- rest — applying rate limiting to REST APIs

---

*Last updated: 2025-03-13*
