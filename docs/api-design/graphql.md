---
title: "GraphQL"
layout: default
parent: API Design
nav_order: 2
---

# GraphQL — Deep Dive

> A comprehensive guide to GraphQL — why it exists, schema design, resolvers, the N+1 problem, tradeoffs vs REST, and real-world production patterns. Written for system design interviews and engineering decisions.

---

## What is GraphQL?

GraphQL is a **query language for APIs** and a runtime for executing those queries. Developed by Facebook in 2012 (open-sourced in 2015), it provides a single endpoint where clients describe exactly what data they need.

- Single endpoint (typically `POST /graphql`)
- Client specifies the shape of the response
- Strongly typed schema defines all available data
- No over-fetching or under-fetching
- Introspectable — clients can discover the schema at runtime

---

## Why GraphQL? What Problem Does It Solve?

### The REST Problem

Facebook built GraphQL because their mobile app and web app needed different data from the same backend, and REST couldn't handle this cleanly.

**Over-fetching (REST returns too much):**
```
GET /users/123
Response: { id, name, email, bio, avatar, address, phone, preferences, 
            created_at, updated_at, last_login, ... }

Mobile app only needed: { id, name, avatar }
→ Wasted bandwidth, slower on mobile networks
```

**Under-fetching (REST returns too little, needs multiple calls):**
```
To show a user's feed with author info and comments:

1. GET /posts?user_id=123           → list of posts
2. GET /users/456                   → author of post 1
3. GET /users/789                   → author of post 2
4. GET /posts/1/comments            → comments on post 1
5. GET /posts/2/comments            → comments on post 2
... N+1 requests for N posts 😬
```

**GraphQL solves both in one request:**
```graphql
query {
  user(id: "123") {
    name
    avatar
    posts(limit: 10) {
      title
      author {
        name
        avatar
      }
      comments(limit: 3) {
        text
        author { name }
      }
    }
  }
}
```
One request. Exact data. No waste.

---

## How GraphQL Works

### Schema — The Contract

The schema defines all types, queries, mutations, and subscriptions available.

```graphql
# Types define the shape of your data
type Event {
  id: ID!
  name: String!
  date: DateTime!
  venue: Venue!
  tickets: [Ticket!]!
  totalTickets: Int!
  availableTickets: Int!
}

type Venue {
  id: ID!
  name: String!
  address: String!
  capacity: Int!
  events: [Event!]!
}

type Ticket {
  id: ID!
  section: String!
  price: Float!
  status: TicketStatus!
}

enum TicketStatus {
  AVAILABLE
  RESERVED
  SOLD
}

# Queries — read operations
type Query {
  event(id: ID!): Event
  events(city: String, limit: Int, after: String): EventConnection!
  venue(id: ID!): Venue
}

# Mutations — write operations
type Mutation {
  createBooking(input: CreateBookingInput!): Booking!
  cancelBooking(id: ID!): Booking!
}

# Input types for mutations
input CreateBookingInput {
  eventId: ID!
  tickets: [TicketInput!]!
  paymentMethod: String!
}

# Subscriptions — real-time updates
type Subscription {
  ticketAvailabilityChanged(eventId: ID!): Event!
}
```

### Queries — Reading Data
```graphql
# Simple query
query {
  event(id: "123") {
    name
    date
  }
}

# Response — exact shape requested
{
  "data": {
    "event": {
      "name": "Taylor Swift Concert",
      "date": "2025-06-15T20:00:00Z"
    }
  }
}
```

```graphql
# Complex nested query — one request replaces 5+ REST calls
query EventDetails {
  event(id: "123") {
    name
    date
    venue {
      name
      address
    }
    tickets(status: AVAILABLE) {
      section
      price
    }
  }
}
```

### Mutations — Writing Data
```graphql
mutation {
  createBooking(input: {
    eventId: "123"
    tickets: [
      { section: "VIP", quantity: 2 }
    ]
    paymentMethod: "credit_card"
  }) {
    id
    status
    totalPrice
  }
}
```

### Subscriptions — Real-Time Updates
```graphql
subscription {
  ticketAvailabilityChanged(eventId: "123") {
    availableTickets
    tickets {
      section
      price
      status
    }
  }
}
```
Subscriptions use WebSocket under the hood to push updates to clients.

---

## Resolvers — How Data Gets Fetched

Each field in the schema has a resolver function that knows how to fetch that data.

```javascript
const resolvers = {
  Query: {
    event: (parent, { id }, context) => {
      return context.db.events.findById(id);
    },
    events: (parent, { city, limit, after }, context) => {
      return context.db.events.find({ city }).paginate({ limit, after });
    }
  },
  Event: {
    venue: (event, args, context) => {
      return context.db.venues.findById(event.venueId);
    },
    tickets: (event, args, context) => {
      return context.db.tickets.find({ eventId: event.id });
    }
  }
};
```

Each type's fields resolve independently. When a client queries `event → venue`, the `Query.event` resolver runs first, then `Event.venue` resolver runs with the event as parent.

---

## The N+1 Problem — GraphQL's Biggest Gotcha

```graphql
query {
  events(limit: 100) {    # 1 query for events
    name
    venue {                # 100 queries for venues (one per event!)
      name
    }
  }
}
# Total: 101 database queries 😬
```

### Solution: DataLoader (Batching + Caching)

```javascript
// Without DataLoader
// Event 1 → SELECT * FROM venues WHERE id = 1
// Event 2 → SELECT * FROM venues WHERE id = 2
// Event 3 → SELECT * FROM venues WHERE id = 1  (duplicate!)
// ... 100 separate queries

// With DataLoader
// Collects all venue IDs in one tick: [1, 2, 1, 3, 2, ...]
// Deduplicates: [1, 2, 3]
// Single batch query: SELECT * FROM venues WHERE id IN (1, 2, 3)
// Caches results for the request lifetime

const venueLoader = new DataLoader(async (venueIds) => {
  const venues = await db.venues.findByIds(venueIds);
  // Return in same order as requested IDs
  return venueIds.map(id => venues.find(v => v.id === id));
});

// Resolver uses loader instead of direct DB call
Event: {
  venue: (event) => venueLoader.load(event.venueId)
}
```

### Interview Tip 💡
> Always mention DataLoader when discussing GraphQL. The N+1 problem is the #1 performance pitfall, and knowing the solution shows production experience. DataLoader batches requests within a single tick of the event loop, then executes one batch query.

---

## Pagination in GraphQL — Connections Pattern

The Relay-style connection pattern is the standard for GraphQL pagination:

```graphql
type Query {
  events(first: Int, after: String, last: Int, before: String): EventConnection!
}

type EventConnection {
  edges: [EventEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type EventEdge {
  node: Event!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

```graphql
# First page
query {
  events(first: 10) {
    edges {
      node { name, date }
      cursor
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}

# Next page
query {
  events(first: 10, after: "cursor_from_previous_page") {
    edges {
      node { name, date }
      cursor
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

---

## Security Concerns Unique to GraphQL

### 1. Query Depth Attack
```graphql
# Malicious deeply nested query
query {
  user(id: "1") {
    friends {
      friends {
        friends {
          friends {
            friends { ... }  # Infinite depth → server crash
          }
        }
      }
    }
  }
}
```
**Mitigation:** Set max query depth (e.g., depth limit of 10).

### 2. Query Complexity Attack
```graphql
# Expensive query — fetches millions of records
query {
  events(limit: 10000) {
    tickets(limit: 10000) {
      ... # 10000 × 10000 = 100M records
    }
  }
}
```
**Mitigation:** Assign complexity scores to fields, reject queries exceeding a threshold.

### 3. Introspection in Production
```graphql
# Attacker discovers your entire schema
query {
  __schema {
    types { name, fields { name, type { name } } }
  }
}
```
**Mitigation:** Disable introspection in production.

---

## GraphQL vs REST — Complete Comparison

| Aspect | REST | GraphQL |
|--------|------|---------|
| Endpoints | Multiple (`/users`, `/events`) | Single (`/graphql`) |
| Data fetching | Fixed response per endpoint | Client specifies exact fields |
| Over-fetching | Common problem | Solved by design |
| Under-fetching | Multiple requests needed | Single query with nesting |
| Caching | HTTP caching works natively | Complex (no URL-based caching) |
| Error handling | HTTP status codes | Always 200, errors in response body |
| File uploads | Native (multipart/form-data) | Awkward (needs special handling) |
| Versioning | URL or header versioning | Schema evolution (deprecate fields) |
| Learning curve | Low | Medium-High |
| Tooling | Massive ecosystem | Growing (Apollo, Relay, Hasura) |
| Real-time | SSE, WebSocket (separate) | Subscriptions (built-in) |
| Type safety | Optional (OpenAPI) | Built-in (schema) |
| Performance | Predictable | Unpredictable (complex queries) |

### When to Use GraphQL
✅ Multiple clients (mobile, web, TV) needing different data shapes
✅ Rapid frontend iteration without backend changes
✅ Complex, deeply nested data relationships
✅ When over-fetching is a real problem (mobile bandwidth)
✅ When you want a strongly typed API contract

### When NOT to Use GraphQL
❌ Simple CRUD APIs — REST is simpler
❌ When HTTP caching is critical — GraphQL uses POST, breaks URL caching
❌ File uploads — awkward in GraphQL
❌ Small teams — added complexity not worth it
❌ When you need predictable server load — arbitrary queries make capacity planning hard
❌ Public APIs for third-party developers — REST is more widely understood

---

## Real-World Production Use Cases

### 1. Facebook/Meta
- Created GraphQL, uses it for all mobile and web data fetching
- Relay framework for client-side GraphQL
- **Why:** Dozens of client platforms needing different data from the same backend

### 2. GitHub (API v4)
- REST for v3, GraphQL for v4
- Lets integrators fetch exactly the repo/issue/PR data they need
- **Why:** Third-party developers were making 5-10 REST calls for data that could be one GraphQL query

### 3. Shopify
- GraphQL for their Storefront API and Admin API
- Merchants build custom storefronts fetching only needed product data
- **Why:** Thousands of different storefront designs needing different data shapes

### 4. Netflix
- GraphQL as a federation layer across microservices
- Each team owns their part of the schema
- **Why:** Hundreds of microservices, one unified API for clients

### 5. Airbnb
- GraphQL for their web and mobile apps
- Schema stitching across multiple backend services
- **Why:** Complex data relationships (listings, hosts, reviews, availability, pricing)

### Interview Tip 💡
> In interviews, default to REST unless the problem specifically involves diverse clients or over/under-fetching. If you choose GraphQL, mention the N+1 problem and DataLoader — it shows you understand the real tradeoffs, not just the marketing pitch.

---

## Further Reading
- rest — the default alternative for most APIs
- grpc — for high-performance internal service communication
- common-api-patterns — pagination, filtering patterns that apply to GraphQL too
- authentication-and-authorization — field-level auth in GraphQL
- api-security — query depth limits, complexity analysis

---

*Last updated: 2025-03-13*
