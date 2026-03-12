# Common API Patterns — Deep Dive

> Patterns that apply across REST, GraphQL, and RPC — pagination, filtering, sorting, versioning, idempotency, bulk operations, and more. These come up in every system design interview.

---

## Pagination

You can't return millions of records in one response. Pagination breaks large datasets into manageable chunks.

### Offset-Based Pagination

```
GET /events?offset=20&limit=10
→ Skip 20 records, return next 10 (records 21-30)
```

```json
{
  "data": [...],
  "pagination": {
    "total": 5000,
    "offset": 20,
    "limit": 10
  }
}
```

**Pros:**
- Simple to implement (`SELECT * FROM events LIMIT 10 OFFSET 20`)
- Supports "jump to page N"
- Easy for clients to understand

**Cons:**
- **Inconsistent results** — if a new record is inserted while paginating, you see duplicates or miss records
- **Slow for large offsets** — `OFFSET 1000000` still scans 1M rows in most databases
- **Inaccurate total count** — `COUNT(*)` is expensive on large tables

### Cursor-Based Pagination (Keyset Pagination)

```
GET /events?limit=10
→ Returns first 10 + cursor pointing to last record

GET /events?cursor=eyJpZCI6MTB9&limit=10
→ Returns next 10 after the cursor
```

```json
{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6MjB9",
    "has_next": true
  }
}
```

**How it works under the hood:**
```sql
-- Cursor decodes to: { "id": 10 }
-- Instead of OFFSET, use WHERE clause:
SELECT * FROM events WHERE id > 10 ORDER BY id LIMIT 10;
-- This uses the index, always fast regardless of page number
```

**Pros:**
- **Consistent results** — not affected by inserts/deletes during pagination
- **Fast at any depth** — uses index, no scanning
- **Scales to billions of records**

**Cons:**
- Can't "jump to page 5" — must traverse sequentially
- Cursor is opaque to clients
- More complex to implement (especially with multiple sort fields)

### Which to Choose?

| Scenario | Choice | Why |
|----------|--------|-----|
| Admin dashboard with page numbers | Offset | Users expect "Page 1, 2, 3..." |
| Infinite scroll feed (Twitter, Instagram) | Cursor | Consistent, fast, no duplicates |
| Real-time data (live events, chat) | Cursor | Data changes constantly |
| Small datasets (<10K records) | Either | Performance doesn't matter |
| Large datasets (millions+) | Cursor | Offset becomes unusably slow |

### Interview Tip 💡
> Default to cursor-based pagination in interviews. It's more scalable and handles real-time data correctly. If the interviewer asks "how do you paginate the news feed?" — cursor-based is always the right answer. Mention that the cursor is typically a base64-encoded JSON of the sort key.

---

## Filtering and Search

### Simple Filtering (Query Parameters)
```
GET /events?city=NYC&category=music&status=active
GET /events?date_from=2025-01-01&date_to=2025-12-31
GET /events?price_min=50&price_max=200
```

### Complex Filtering
```
# Multiple values for one field (OR)
GET /events?category=music,sports,theater

# Operators
GET /events?price[gte]=50&price[lte]=200
GET /events?name[contains]=swift

# Full-text search
GET /events?q=taylor+swift+concert
```

### Filtering in GraphQL
```graphql
query {
  events(filter: {
    city: "NYC"
    category: MUSIC
    dateRange: { from: "2025-01-01", to: "2025-12-31" }
    priceRange: { min: 50, max: 200 }
  }, limit: 20) {
    name
    date
    price
  }
}
```

---

## Sorting

```
GET /events?sort=date&order=asc
GET /events?sort=-date              # Prefix "-" means descending
GET /events?sort=date,-price        # Multi-field: date asc, then price desc
```

### Sorting + Cursor Pagination
When combining sorting with cursor pagination, the cursor must encode the sort field:
```
Sort by date:
  Cursor = base64({ "date": "2025-06-15", "id": "evt_123" })
  
SQL: WHERE (date, id) > ('2025-06-15', 'evt_123') ORDER BY date, id LIMIT 10
```
Always include a unique field (like `id`) as a tiebreaker to ensure deterministic ordering.

---

## Sparse Fieldsets (Field Selection)

Let clients request only the fields they need (REST's answer to GraphQL's flexibility):

```
GET /events?fields=id,name,date
GET /events/123?fields=id,name,venue.name,venue.address
```

```json
// Without field selection — returns everything
{ "id": "123", "name": "Concert", "date": "...", "description": "...", 
  "venue": {...}, "organizer": {...}, "tickets": [...], ... }

// With fields=id,name,date — returns only requested
{ "id": "123", "name": "Concert", "date": "..." }
```

Used by: Google APIs, Spotify API, Facebook Graph API.

---

## Versioning Strategies

### URL Versioning (Most Common)
```
GET /v1/events/123
GET /v2/events/123
```
- Explicit, easy to understand
- Easy to route at load balancer level
- Used by: Twitter, Stripe, GitHub

### Header Versioning
```
GET /events/123
Accept-Version: v2
// or
API-Version: 2
```
- Cleaner URLs
- Harder to test in browser
- Used by: GitHub (Accept header)

### Date-Based Versioning (Stripe's Approach)
```
GET /events/123
Stripe-Version: 2024-10-28
```
- Each API change gets a date
- Clients pin to a date, upgrade when ready
- Most granular, best for public APIs with many consumers

### Query Parameter Versioning
```
GET /events/123?version=2
```
- Simple but pollutes the query string
- Rarely used in production

### No Versioning — Schema Evolution
- GraphQL: deprecate fields, add new ones
- gRPC/Protobuf: add new field numbers, old clients ignore them
- Works when you control all clients

### Interview Tip 💡
> In interviews, URL versioning (`/v1/`) is the safest answer. It's the most widely used and easiest to explain. Mention that Stripe uses date-based versioning if you want to show depth. Most interviewers don't care deeply about versioning — just show you've thought about it.

---

## Idempotency

Ensuring that retrying a request doesn't cause duplicate side effects.

### Why It Matters
```
Client → POST /payments (charge $100) → Network timeout
Client → POST /payments (retry)       → Did the first one go through? 🤔

Without idempotency: Customer charged $200
With idempotency: Customer charged $100 (retry returns cached result)
```

### Implementation Pattern
```
POST /payments
Idempotency-Key: "pay_abc123_unique_uuid"
Content-Type: application/json

{ "amount": 100, "currency": "USD" }
```

**Server logic:**
```
1. Receive request with Idempotency-Key
2. Check Redis/DB: does this key exist?
   ├── Yes → Return the stored response (no re-execution)
   └── No → Execute the operation
            → Store key + response in Redis (TTL: 24h)
            → Return response
```

**Used by:** Stripe, PayPal, Square, AWS — any system where duplicate operations are dangerous.

### Which Methods Need Idempotency Keys?
- GET, PUT, DELETE — already idempotent by definition
- **POST** — needs idempotency keys (creates new resources)
- **PATCH** — depends on implementation (append operations need keys)

---

## Bulk/Batch Operations

When clients need to create, update, or delete many resources at once.

### Batch Create
```json
POST /events/batch
{
  "events": [
    { "name": "Concert A", "date": "2025-06-01" },
    { "name": "Concert B", "date": "2025-06-02" },
    { "name": "Concert C", "date": "2025-06-03" }
  ]
}

// Response — partial success is possible
{
  "results": [
    { "status": 201, "data": { "id": "evt_1", "name": "Concert A" } },
    { "status": 201, "data": { "id": "evt_2", "name": "Concert B" } },
    { "status": 400, "error": { "message": "Invalid date format" } }
  ]
}
```

### Batch in GraphQL
```graphql
mutation {
  createEvents(input: [
    { name: "Concert A", date: "2025-06-01" }
    { name: "Concert B", date: "2025-06-02" }
  ]) {
    id
    name
  }
}
```

---

## Webhooks — Server-to-Server Notifications

Instead of clients polling for changes, the server pushes events to a client-registered URL.

```
1. Client registers: POST /webhooks
   { "url": "https://myapp.com/hooks", "events": ["booking.created", "booking.cancelled"] }

2. When event occurs, YOUR server calls THEIR server:
   POST https://myapp.com/hooks
   {
     "event": "booking.created",
     "data": { "booking_id": "123", "user_id": "456" },
     "timestamp": "2025-03-13T10:00:00Z"
   }
```

**Production considerations:**
- Retry with exponential backoff on failure (3 retries over 24h)
- Include a signature header for verification (`X-Webhook-Signature`)
- Idempotency — receivers should handle duplicate deliveries
- Timeout — don't wait more than 5-10s for receiver response

**Used by:** Stripe, GitHub, Shopify, Twilio — any platform with third-party integrations.

---

## Long-Running Operations (Async APIs)

Some operations take minutes or hours (video transcoding, report generation, data export).

```
# 1. Client starts the operation
POST /reports/generate
{ "type": "annual_sales", "year": 2024 }

# Response: 202 Accepted (not 200 — it's not done yet)
{
  "operation_id": "op_abc123",
  "status": "processing",
  "status_url": "/operations/op_abc123"
}

# 2. Client polls for status
GET /operations/op_abc123
{
  "operation_id": "op_abc123",
  "status": "completed",
  "result_url": "/reports/rpt_xyz789",
  "progress": 100
}

# 3. Client fetches the result
GET /reports/rpt_xyz789
```

**Alternative:** Use webhooks to notify when complete instead of polling.

---

## ETag-Based Optimistic Concurrency

Prevent lost updates when multiple clients edit the same resource.

```
# 1. Client A reads the event
GET /events/123
→ ETag: "v5"
→ { "name": "Concert", "price": 50 }

# 2. Client B also reads the event
GET /events/123
→ ETag: "v5"
→ { "name": "Concert", "price": 50 }

# 3. Client A updates price
PUT /events/123
If-Match: "v5"
{ "name": "Concert", "price": 75 }
→ 200 OK, ETag: "v6"

# 4. Client B tries to update (stale ETag)
PUT /events/123
If-Match: "v5"          ← stale!
{ "name": "Concert", "price": 60 }
→ 409 Conflict ❌ (Client B must re-read and retry)
```

This prevents the "last write wins" problem where Client B's change silently overwrites Client A's.

---

## Further Reading
- [[rest]] — REST-specific patterns and resource modeling
- [[graphql]] — GraphQL-specific patterns (connections, DataLoader)
- [[rpc]] — RPC-specific patterns (deadlines, interceptors)
- [[rate-limiting]] — protecting APIs from abuse
- [[authentication-and-authorization]] — securing API access

---

*Last updated: 2025-03-13*
