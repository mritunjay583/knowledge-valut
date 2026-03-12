---
title: "REST"
layout: default
parent: API Design
nav_order: 1
---

# REST

> A comprehensive guide to REST — principles, resource modeling, HTTP semantics, pagination, versioning, tradeoffs, and real-world production patterns. Written for system design interviews and building highly scalable systems.

---

## What is REST?

REST (Representational State Transfer) is an **architectural style** for designing networked applications. It was defined by Roy Fielding in his 2000 PhD dissertation. REST is NOT a protocol — it's a set of constraints applied on top of http.

- Uses standard HTTP methods (GET, POST, PUT, PATCH, DELETE)
- Resources identified by URLs
- Stateless — each request contains all information needed to process it
- Representations — resources can have multiple formats (JSON, XML, HTML)
- Default choice for ~90% of web APIs

---

## REST Constraints (The 6 Principles)

These are what make an API truly "RESTful." Most APIs claiming to be REST only follow some of these.

### 1. Client-Server Separation
- Client and server evolve independently
- Client doesn't know about database, server doesn't know about UI
- Connected only through the API contract

### 2. Statelessness
- Server stores NO client state between requests
- Every request must contain all information (auth token, context, etc.)
- Enables horizontal scaling — any server can handle any request

```
❌ Stateful: Server remembers "User is on page 3 of results"
✅ Stateless: Client sends "GET /events?page=3" every time
```

### 3. Cacheability
- Responses must declare themselves cacheable or non-cacheable
- Enables CDN caching, browser caching, proxy caching
- Reduces server load and latency

### 4. Uniform Interface
- Consistent way to interact with all resources
- Resources identified by URIs
- Manipulation through representations (JSON body)
- Self-descriptive messages (Content-Type, status codes)
- HATEOAS (Hypermedia as the Engine of Application State) — responses include links to related actions

### 5. Layered System
- Client doesn't know if it's talking to the actual server, a CDN, a load balancer, or a proxy
- Enables intermediaries (caching layers, security layers, load balancers)

### 6. Code on Demand (Optional)
- Server can send executable code to the client (JavaScript)
- Rarely used in APIs, more relevant for web pages

### Interview Tip 💡
> Most interviewers won't ask you to list all 6 constraints. But knowing **statelessness** (enables scaling) and **cacheability** (enables performance) deeply will set you apart. These two drive most architectural decisions.

---

## Resource Modeling — The Foundation

The core of REST design is thinking in **resources** (nouns), not actions (verbs).

### Rules
- Resources are **plural nouns**: `/events`, `/users`, `/bookings`
- URLs identify resources, HTTP methods define actions
- Hierarchical relationships expressed through nesting

### Example: Ticketmaster-like System
```
GET    /events                     # List all events
GET    /events/{id}                # Get a specific event
POST   /events                     # Create a new event
PUT    /events/{id}                # Replace an event entirely
PATCH  /events/{id}                # Update part of an event
DELETE /events/{id}                # Delete an event

GET    /events/{id}/tickets        # Get tickets for an event
POST   /events/{id}/bookings       # Create a booking for an event
GET    /bookings/{id}              # Get a specific booking
DELETE /bookings/{id}              # Cancel a booking
```

### Nested vs Flat Resources

**Nested (parent-child relationship is required):**
```
GET /events/{eventId}/tickets          # Tickets always belong to an event
GET /users/{userId}/orders             # Orders always belong to a user
```

**Flat with query params (relationship is optional/filterable):**
```
GET /tickets?event_id=123&section=VIP  # Filter tickets by event AND section
GET /orders?user_id=456&status=pending # Filter orders by user AND status
```

**Rule of thumb:**
- Path parameter → required to identify the resource
- Query parameter → optional filter/modifier

### Anti-Patterns to Avoid
```
❌ GET  /getEvents              # Verb in URL
❌ POST /createBooking          # Verb in URL
❌ GET  /events/delete/123      # Action in URL
❌ POST /events/123/cancel      # Action in URL (use DELETE or PATCH status)

✅ GET    /events               # Resource noun
✅ POST   /bookings             # Resource noun
✅ DELETE /bookings/123         # HTTP method conveys the action
✅ PATCH  /bookings/123         # Update status to "cancelled"
   { "status": "cancelled" }
```

### Interview Tip 💡
> When the action doesn't map cleanly to CRUD (like "cancel a booking" or "approve a request"), model it as a state change via PATCH, or create a sub-resource: `POST /bookings/123/cancellation`. This shows you understand REST's resource-oriented thinking.

---

## HTTP Methods — Deep Semantics

| Method | Action | Idempotent | Safe | Has Body | Cacheable |
|--------|--------|------------|------|----------|-----------|
| GET | Read | ✅ | ✅ | ❌ | ✅ |
| POST | Create | ❌ | ❌ | ✅ | ❌ |
| PUT | Replace | ✅ | ❌ | ✅ | ❌ |
| PATCH | Partial Update | ❌* | ❌ | ✅ | ❌ |
| DELETE | Remove | ✅ | ❌ | Optional | ❌ |

*PATCH can be idempotent if implemented as "set field to X" but not if "append to list."

### PUT vs PATCH — The Subtle Difference
```json
// Original resource
{ "name": "Concert", "date": "2025-06-01", "venue": "Stadium", "price": 50 }

// PUT /events/123 — replaces ENTIRE resource
{ "name": "Concert", "date": "2025-07-01", "venue": "Arena", "price": 75 }
// Missing fields would be set to null/default

// PATCH /events/123 — updates ONLY specified fields
{ "price": 75 }
// Other fields remain unchanged
```

### Idempotency — Why It Matters for Scalable Systems

```
Scenario: User clicks "Book Tickets" and network times out.
Client retries the request. What happens?

POST /bookings (NOT idempotent):
  First call → Booking #1 created
  Retry      → Booking #2 created 😬 (duplicate!)

Solution: Idempotency Key
POST /bookings
Idempotency-Key: "abc-123-unique"
  First call → Booking #1 created
  Retry      → Server sees same key, returns Booking #1 (no duplicate)
```

**How idempotency keys work:**
1. Client generates a unique key (UUID) per logical operation
2. Server stores the key + response in a cache (Redis)
3. On retry, server checks if key exists → returns cached response
4. Keys expire after a TTL (e.g., 24 hours)

This is how **Stripe, PayPal, and every payment API** handles retries safely.

---

## Passing Data — Path, Query, Body

### Path Parameters — Resource Identification
```
GET /events/123              # "123" identifies which event
GET /users/456/orders/789    # "456" = user, "789" = order
```
- Always required
- Part of the resource hierarchy
- Used for lookups by ID

### Query Parameters — Filtering, Sorting, Pagination
```
GET /events?city=NYC&date_from=2025-01-01&sort=date&order=asc&page=2&limit=20
```
- Optional modifiers
- Filtering: `?status=active&category=music`
- Sorting: `?sort=created_at&order=desc`
- Pagination: `?page=2&limit=20` or `?cursor=abc123&limit=20`
- Search: `?q=taylor+swift`
- Field selection: `?fields=id,name,date` (sparse fieldsets)

### Request Body — Data Payload
```http
POST /events/123/bookings?notify=true
Content-Type: application/json

{
  "tickets": [
    { "section": "VIP", "quantity": 2 },
    { "section": "General", "quantity": 1 }
  ],
  "payment_method": "credit_card"
}
```
- Used with POST, PUT, PATCH
- Complex data structures
- Sensitive data (never in URL — URLs get logged)

### Interview Tip 💡
> Never put sensitive data (passwords, tokens, payment info) in query parameters. URLs are logged by servers, proxies, and browsers. Always use request body or headers for sensitive data.

---

## Response Design

### Status Codes — The Important Ones

| Code | Meaning | When to Use |
|------|---------|-------------|
| 200 | OK | Successful GET, PUT, PATCH |
| 201 | Created | Successful POST (return created resource + Location header) |
| 204 | No Content | Successful DELETE (nothing to return) |
| 400 | Bad Request | Invalid input, malformed JSON |
| 401 | Unauthorized | Missing or invalid authentication |
| 403 | Forbidden | Authenticated but not authorized |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate resource, version conflict |
| 422 | Unprocessable Entity | Valid JSON but business logic validation failed |
| 429 | Too Many Requests | Rate limited |
| 500 | Internal Server Error | Server bug |
| 502 | Bad Gateway | Upstream service failed |
| 503 | Service Unavailable | Server overloaded or in maintenance |

### Response Body Patterns

**Single resource:**
```json
{
  "id": "evt_123",
  "name": "Taylor Swift Concert",
  "date": "2025-06-15T20:00:00Z",
  "venue": {
    "id": "ven_456",
    "name": "Madison Square Garden"
  }
}
```

**Collection with pagination:**
```json
{
  "data": [
    { "id": "evt_123", "name": "Concert A" },
    { "id": "evt_456", "name": "Concert B" }
  ],
  "pagination": {
    "total": 1250,
    "page": 2,
    "limit": 20,
    "next_cursor": "eyJpZCI6ImV2dF80NTYifQ=="
  }
}
```

**Error response:**
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid booking request",
    "details": [
      { "field": "tickets[0].quantity", "message": "Must be between 1 and 10" }
    ]
  }
}
```

---

## REST Tradeoffs

### Strengths
- ✅ Simple, well-understood, massive ecosystem
- ✅ HTTP caching works natively (CDN, browser, proxy)
- ✅ Stateless — easy to scale horizontally
- ✅ Human-readable (JSON + curl for debugging)
- ✅ Browser-native (fetch API, no special libraries)
- ✅ Great tooling (Postman, Swagger/OpenAPI, curl)

### Weaknesses
- ❌ Over-fetching — endpoint returns more data than client needs
- ❌ Under-fetching — client needs multiple requests to assemble data (N+1)
- ❌ No formal contract enforcement (OpenAPI is optional)
- ❌ No streaming (workarounds: SSE, chunked transfer)
- ❌ Chatty for complex data — mobile apps suffer from multiple round trips
- ❌ Versioning is painful — breaking changes affect all clients

### When to Use REST
✅ Public-facing APIs (web, mobile clients)
✅ CRUD-heavy applications
✅ When HTTP caching matters (CDN, browser)
✅ When simplicity and developer experience matter
✅ When your team is small and needs to move fast

### When NOT to Use REST
❌ Diverse clients needing different data shapes → **GraphQL**
❌ High-performance internal service communication → **gRPC**
❌ Real-time bidirectional communication → **WebSocket**
❌ Action-oriented APIs that don't map to resources → **RPC**

---

## Real-World Production Use Cases

### 1. Stripe API
- Considered the gold standard of REST API design
- Resources: `/customers`, `/charges`, `/subscriptions`, `/invoices`
- Idempotency keys on all POST requests
- Versioning via date-based headers (`Stripe-Version: 2023-10-16`)
- Pagination via cursor-based approach

### 2. GitHub API
- REST for public API (v3), GraphQL for v4
- Resources: `/repos`, `/issues`, `/pulls`, `/users`
- Pagination via Link headers
- Rate limiting: 5000 requests/hour for authenticated users

### 3. Twitter/X API
- REST for tweet CRUD, user management
- Cursor-based pagination for timelines
- Rate limiting per endpoint (different limits for read vs write)
- OAuth 2.0 for authentication

### 4. Spotify API
- REST for catalog browsing, playlist management
- Resources: `/tracks`, `/albums`, `/playlists`, `/artists`
- Field filtering: `?fields=items(track(name,artists))` to reduce payload

### 5. AWS S3 API
- REST for object storage operations
- PUT for uploads, GET for downloads, DELETE for removal
- Presigned URLs for temporary access
- Demonstrates REST at massive scale (trillions of objects)

---

## Further Reading
- http — the protocol REST is built on
- graphql — alternative when REST's fixed endpoints cause over/under-fetching
- grpc — alternative for high-performance internal APIs
- common-api-patterns — pagination, versioning, filtering patterns
- authentication-and-authorization — securing your REST APIs
- rate-limiting — protecting your APIs from abuse

---

*Last updated: 2025-03-13*
