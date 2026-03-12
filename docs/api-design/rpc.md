---
title: "RPC"
layout: default
parent: API Design
nav_order: 3
---

# RPC

> A comprehensive guide to RPC — the paradigm, gRPC, Protocol Buffers, tradeoffs vs REST and GraphQL, and when to use it. Written for system design interviews and building high-performance distributed systems.

---

## What is RPC?

RPC (Remote Procedure Call) is a **communication paradigm** where a client calls a function on a remote server as if it were a local function call. The network details are abstracted away.

```
// Feels like a local function call
result = orderService.createOrder(userId, items)

// But actually goes over the network:
// Client → serialize args → network → server → deserialize → execute → serialize result → network → client
```

- **Action-oriented** (verbs) vs REST's resource-oriented (nouns)
- Binary serialization for performance
- Strong typing via IDL (Interface Definition Language)
- Code generation for multiple languages

---

## RPC vs REST — Fundamental Mindset Difference

```
REST thinks in resources:
  GET    /events/123              → "Give me event 123"
  POST   /events/123/bookings    → "Create a booking for event 123"
  DELETE /bookings/456            → "Remove booking 456"

RPC thinks in actions:
  getEvent(eventId: "123")                    → "Run this function"
  createBooking(eventId: "123", userId: "456") → "Run this function"
  cancelBooking(bookingId: "456")              → "Run this function"
```

REST forces you to model everything as resources. Sometimes that's awkward:
```
REST (awkward):
  POST /emails/send              ← "send" is a verb, not a resource 🤔
  POST /reports/generate         ← "generate" is a verb 🤔
  POST /users/123/password-reset ← is this a resource? 🤔

RPC (natural):
  sendEmail(to, subject, body)
  generateReport(filters, format)
  resetPassword(userId)
```

---

## Major RPC Frameworks

| Framework | Creator | Serialization | Transport | Language Support |
|-----------|---------|---------------|-----------|-----------------|
| gRPC | Google | Protobuf | HTTP/2 | 30+ languages |
| Apache Thrift | Facebook | Thrift Binary/Compact | TCP | 20+ languages |
| Twirp | Twitch | Protobuf or JSON | HTTP/1.1 | Go, Python, JS |
| Connect | Buf | Protobuf or JSON | HTTP/1.1 + HTTP/2 | Go, JS, Swift, Kotlin |
| JSON-RPC | Community | JSON | HTTP or TCP | Any |
| XML-RPC | Dave Winer | XML | HTTP | Any |

gRPC dominates the modern landscape. See grpc for the full deep dive on architecture, streaming patterns, and production use cases.

---

## gRPC — The Modern Standard

### Service Definition (.proto)
```protobuf
syntax = "proto3";
package ticketing;

service TicketService {
  // Unary — standard request-response
  rpc GetEvent(GetEventRequest) returns (Event);
  rpc CreateBooking(CreateBookingRequest) returns (Booking);
  
  // Server streaming — server sends multiple responses
  rpc StreamTicketUpdates(EventRequest) returns (stream TicketUpdate);
  
  // Client streaming — client sends multiple requests
  rpc UploadBulkEvents(stream Event) returns (UploadSummary);
  
  // Bidirectional streaming
  rpc LiveAuction(stream Bid) returns (stream AuctionUpdate);
}

message GetEventRequest {
  string event_id = 1;
}

message Event {
  string id = 1;
  string name = 2;
  int64 date_unix = 3;
  Venue venue = 4;
  repeated Ticket tickets = 5;
}
```

### Generated Code Usage
```go
// Server implementation (Go)
type ticketServer struct{}

func (s *ticketServer) GetEvent(ctx context.Context, req *pb.GetEventRequest) (*pb.Event, error) {
    event, err := db.FindEvent(req.EventId)
    if err != nil {
        return nil, status.Errorf(codes.NotFound, "event not found")
    }
    return event, nil
}

// Client usage (Go) — feels like a local function call
client := pb.NewTicketServiceClient(conn)
event, err := client.GetEvent(ctx, &pb.GetEventRequest{EventId: "123"})
```

```java
// Client usage (Java) — same .proto, different language
TicketServiceGrpc.TicketServiceBlockingStub client = 
    TicketServiceGrpc.newBlockingStub(channel);
Event event = client.getEvent(GetEventRequest.newBuilder()
    .setEventId("123")
    .build());
```

---

## REST vs GraphQL vs RPC — Three-Way Comparison

| Aspect | REST | GraphQL | RPC (gRPC) |
|--------|------|---------|------------|
| Paradigm | Resource-oriented | Query-oriented | Action-oriented |
| Endpoint | Multiple URLs | Single `/graphql` | Per-service methods |
| Data format | JSON (text) | JSON (text) | Protobuf (binary) |
| Contract | Optional (OpenAPI) | Required (schema) | Required (.proto) |
| Code generation | Optional | Optional (codegen tools) | Built-in, first-class |
| Type safety | Runtime | Runtime (schema validation) | Compile-time |
| Caching | ✅ HTTP native | ❌ Complex | ❌ No HTTP caching |
| Browser support | ✅ Native | ✅ (via HTTP POST) | ❌ Needs grpc-web |
| Streaming | ❌ (SSE workaround) | Subscriptions (WebSocket) | ✅ 4 native patterns |
| Performance | Good | Good | Great (~10x faster) |
| Over-fetching | ❌ Common | ✅ Solved | ❌ Fixed response |
| Learning curve | Low | Medium | Medium-High |
| Best for | Public APIs | Flexible client data needs | Internal microservices |

### Decision Matrix

```
Who is the consumer?
├── Browser/Mobile (public) → REST (default) or GraphQL (if diverse clients)
├── Internal microservices → gRPC
├── Third-party developers → REST (most widely understood)
└── Mixed → REST at the edge, gRPC internally

What matters most?
├── Simplicity → REST
├── Flexibility → GraphQL
├── Performance → gRPC
├── Type safety → gRPC
├── Caching → REST
└── Streaming → gRPC or WebSocket
```

### The Common Production Pattern
```
┌─────────────┐     REST or GraphQL      ┌──────────────┐
│ Browser/     │ ──────────────────────→  │ API Gateway  │
│ Mobile App   │                          │ (public API) │
└─────────────┘                          └──────┬───────┘
                                                │ gRPC (internal)
                                    ┌───────────┼───────────┐
                                    ↓           ↓           ↓
                              ┌──────────┐ ┌──────────┐ ┌──────────┐
                              │ User     │ │ Booking  │ │ Payment  │
                              │ Service  │ │ Service  │ │ Service  │
                              └──────────┘ └──────────┘ └──────────┘
```

REST/GraphQL for external clients, gRPC for internal service-to-service. This is the pattern at Google, Netflix, Uber, and most large-scale systems.

### Interview Tip 💡
> In interviews, default to REST for user-facing APIs. Mention gRPC for internal service communication when performance matters. If the interviewer asks about internal APIs, say "I'd use gRPC between services for type safety and performance, with REST at the API gateway for external clients." This shows you understand the real-world pattern.

---

## When RPC Shines (and When It Doesn't)

### Use RPC When
✅ Internal microservice communication (performance + type safety)
✅ Polyglot environments (one .proto, code in any language)
✅ Actions that don't map to resources (`sendNotification`, `validatePayment`)
✅ Streaming is needed (real-time data, event streams)
✅ Low-latency requirements (binary serialization, HTTP/2)

### Don't Use RPC When
❌ Public APIs for browsers (no native browser support)
❌ When HTTP caching matters (gRPC uses POST)
❌ Simple CRUD apps (REST is simpler)
❌ When human-readable debugging matters (binary is hard to inspect)
❌ Small teams where protoc/codegen overhead isn't worth it

---

## Real-World Production Use Cases

### 1. Google — Internal Services
- All internal services communicate via Stubby (gRPC's predecessor)
- Billions of RPCs per second across their infrastructure
- **Why:** Polyglot (C++, Java, Go, Python), extreme performance needs

### 2. Netflix — Microservice Communication
- gRPC between hundreds of microservices
- REST/GraphQL for external client APIs
- **Why:** Type safety across service boundaries, streaming for data pipelines

### 3. Uber — Ride Matching
- gRPC for real-time communication between matching, pricing, and dispatch services
- **Why:** Low latency critical for ride matching, streaming for location updates

### 4. Coinbase — Financial Transactions
- gRPC between internal services handling trades and transfers
- **Why:** Type safety critical for financial data, deadline propagation prevents hung transactions

### 5. Slack — Backend Services
- gRPC for internal service mesh
- REST for public API
- **Why:** Performance at scale, type safety for rapid development

---

## Further Reading
- grpc — deep dive into gRPC architecture, streaming, and internals
- rest — the default alternative for public APIs
- graphql — alternative for flexible client data fetching
- http — the transport layer gRPC runs on (HTTP/2)
- common-api-patterns — patterns that apply across all API styles

---

*Last updated: 2025-03-13*
