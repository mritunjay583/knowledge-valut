# gRPC — Deep Dive

> A comprehensive guide to gRPC — why it exists, what problems it solves, how it compares to HTTP/REST, architecture internals, and real-world production use cases. Written for system design interviews and engineering decisions.

---

## What is gRPC?

gRPC (gRPC Remote Procedure Call) is a high-performance, open-source **RPC framework** originally developed by Google. It lets you call methods on a remote server as if they were local function calls.

- Built on top of **HTTP/2**
- Uses **Protocol Buffers (Protobuf)** as the default serialization format
- Supports **4 communication patterns** (unary, server streaming, client streaming, bidirectional)
- Strongly typed via `.proto` contract files
- Auto-generates client and server code in **30+ languages**

---

## Why gRPC? What Problem Does It Solve?

### The Problem with REST/HTTP for Internal Services

When microservices talk to each other, REST/HTTP has real friction:

1. **JSON is slow** — text-based, no schema enforcement, parsing is CPU-intensive at scale
2. **No formal contract** — OpenAPI/Swagger exists but is optional and often out of sync
3. **No streaming** — HTTP/1.1 request-response is one-shot; workarounds (SSE, long polling) are hacky
4. **Code generation is an afterthought** — you write HTTP clients manually or use codegen tools that feel bolted on
5. **Chattiness** — REST resources often require multiple round trips to assemble data (N+1 problem)
6. **No built-in deadlines/cancellation** — you implement timeouts yourself, inconsistently across services

### What gRPC Solves

| Problem | REST/HTTP | gRPC |
|---------|-----------|------|
| Serialization speed | JSON (slow, text) | Protobuf (fast, binary, ~10x smaller) |
| Contract enforcement | Optional (OpenAPI) | Mandatory (.proto files) |
| Code generation | Bolted on | First-class, multi-language |
| Streaming | Hacks (SSE, WebSocket) | Native, 4 patterns built-in |
| Type safety | Runtime errors | Compile-time errors |
| Deadlines & cancellation | DIY | Built into the framework |
| Multiplexing | HTTP/2 optional | HTTP/2 always |

### Interview Tip 💡
> gRPC isn't a replacement for REST — it solves a different problem. REST is great for public-facing APIs (browser-friendly, human-readable). gRPC is great for **internal service-to-service communication** where performance, type safety, and streaming matter.

---

## gRPC Architecture

### High-Level Architecture
```
┌──────────────┐                          ┌──────────────┐
│  gRPC Client │                          │  gRPC Server │
│              │                          │              │
│  Generated   │  ── HTTP/2 + Protobuf ─→ │  Generated   │
│  Client Stub │  ←─ HTTP/2 + Protobuf ── │  Server Stub │
│              │                          │              │
│  Application │                          │  Service     │
│  Code        │                          │  Implementation│
└──────────────┘                          └──────────────┘
        ↑                                        ↑
        └──────── Both generated from ───────────┘
                    .proto file
```

### Core Components

**1. Protocol Buffers (.proto file)** — The Contract
```protobuf
syntax = "proto3";

package order;

// Service definition — defines the RPC methods
service OrderService {
  rpc CreateOrder (CreateOrderRequest) returns (CreateOrderResponse);
  rpc GetOrder (GetOrderRequest) returns (Order);
  rpc StreamOrderUpdates (GetOrderRequest) returns (stream OrderUpdate);
}

// Message definitions — defines the data structures
message CreateOrderRequest {
  string user_id = 1;
  repeated OrderItem items = 2;
}

message OrderItem {
  string product_id = 1;
  int32 quantity = 2;
  double price = 3;
}

message CreateOrderResponse {
  string order_id = 1;
  OrderStatus status = 2;
}

enum OrderStatus {
  PENDING = 0;
  CONFIRMED = 1;
  SHIPPED = 2;
  DELIVERED = 3;
}
```

**2. Code Generation (protoc compiler)**
```bash
# Generates client and server code from .proto
protoc --go_out=. --go-grpc_out=. order.proto    # Go
protoc --java_out=. --grpc-java_out=. order.proto # Java
protoc --python_out=. --grpc_python_out=. order.proto # Python
```

This generates:
- **Client stub** — call remote methods like local functions
- **Server interface** — implement the service methods
- **Message classes** — serialize/deserialize data structures

**3. Channel** — The Connection
- A channel represents a connection to a gRPC server
- Handles connection pooling, load balancing, reconnection
- Can be shared across multiple RPCs

**4. Interceptors** — Middleware for gRPC
- Like HTTP middleware but for RPC calls
- Used for logging, auth, metrics, retries
- Both client-side and server-side interceptors

### How a gRPC Call Works Internally
```
1. Client calls stub method: orderService.CreateOrder(request)
2. Client stub serializes request → Protobuf binary
3. HTTP/2 frame created with:
   - Path: /order.OrderService/CreateOrder
   - Content-Type: application/grpc
   - Protobuf binary as body
4. HTTP/2 sends frame to server
5. Server receives frame, routes to correct service/method
6. Server stub deserializes Protobuf → request object
7. Server executes business logic
8. Server serializes response → Protobuf binary
9. HTTP/2 sends response frame back
10. Client stub deserializes → response object
11. Client receives typed response
```

### Interview Tip 💡
> gRPC uses HTTP/2 under the hood, but you never interact with HTTP directly. The path format is `/package.ServiceName/MethodName`. This is important to know because load balancers and proxies need to understand this routing pattern.

---

## The 4 Communication Patterns

### 1. Unary RPC (Request-Response)
```protobuf
rpc GetUser (GetUserRequest) returns (User);
```
```
Client ──Request──→ Server
Client ←─Response── Server
```
Like a normal HTTP request. One request, one response.

**Use case:** CRUD operations, simple queries

### 2. Server Streaming RPC
```protobuf
rpc ListOrders (ListOrdersRequest) returns (stream Order);
```
```
Client ──Request──→ Server
Client ←─Response 1── Server
Client ←─Response 2── Server
Client ←─Response 3── Server
Client ←─ (done) ──── Server
```
Client sends one request, server sends back a stream of responses.

**Use case:** Real-time stock prices, log tailing, downloading large datasets in chunks

### 3. Client Streaming RPC
```protobuf
rpc UploadFile (stream FileChunk) returns (UploadResponse);
```
```
Client ──Request 1──→ Server
Client ──Request 2──→ Server
Client ──Request 3──→ Server
Client ── (done) ───→ Server
Client ←──Response─── Server
```
Client sends a stream of requests, server sends back one response.

**Use case:** File uploads, sending sensor data, batch writes

### 4. Bidirectional Streaming RPC
```protobuf
rpc Chat (stream ChatMessage) returns (stream ChatMessage);
```
```
Client ──Message──→ Server
Client ←─Message── Server
Client ──Message──→ Server
Client ──Message──→ Server
Client ←─Message── Server
```
Both sides send streams independently. They don't have to alternate.

**Use case:** Chat applications, real-time collaboration, multiplayer gaming

### Interview Tip 💡
> Bidirectional streaming in gRPC is NOT the same as WebSockets. gRPC bidi streaming runs over HTTP/2 with typed messages and built-in flow control. WebSockets are raw TCP with no structure. gRPC gives you type safety, deadlines, and cancellation for free.

---

## Protocol Buffers (Protobuf) — Deep Dive

Protobuf is the default serialization format for gRPC. Understanding it is essential.

### Why Protobuf over JSON?

| Aspect | JSON | Protobuf |
|--------|------|----------|
| Format | Text | Binary |
| Size | Large (field names repeated) | Small (~3-10x smaller) |
| Parse Speed | Slow (string parsing) | Fast (~5-100x faster) |
| Schema | Optional | Required (.proto) |
| Type Safety | No (everything is string/number) | Yes (int32, string, enum, etc.) |
| Human Readable | ✅ Yes | ❌ No |
| Browser Support | ✅ Native | ❌ Needs library |
| Backward Compatibility | Fragile | Built-in (field numbers) |

### How Protobuf Achieves Small Size
```protobuf
message User {
  string name = 1;    // field number 1
  int32 age = 2;      // field number 2
  string email = 3;   // field number 3
}
```

In JSON:
```json
{"name": "Alice", "age": 30, "email": "alice@example.com"}
// ~52 bytes — field names included every time
```

In Protobuf:
```
[field 1, type string] [length] [Alice] [field 2, type varint] [30] [field 3, type string] [length] [alice@example.com]
// ~35 bytes — field numbers (1-2 bytes) instead of names
```

### Backward & Forward Compatibility
```protobuf
// Version 1
message User {
  string name = 1;
  int32 age = 2;
}

// Version 2 — added email (field 3), old clients still work
message User {
  string name = 1;
  int32 age = 2;
  string email = 3;  // new field — old clients ignore it
}
```

Rules for safe evolution:
- ✅ Add new fields (with new field numbers)
- ✅ Remove fields (but never reuse the field number)
- ❌ Never change a field's type
- ❌ Never change a field's number

### Interview Tip 💡
> The field numbers in Protobuf are what get serialized, not the field names. That's why you must NEVER reuse a field number — old clients would misinterpret the data. This is also why Protobuf is so compact compared to JSON.

---

## HTTP/REST vs gRPC — Complete Comparison

| Aspect | HTTP/REST | gRPC |
|--------|-----------|------|
| Transport | HTTP/1.1 or HTTP/2 | HTTP/2 (always) |
| Data Format | JSON (text) | Protobuf (binary) |
| Contract | Optional (OpenAPI) | Required (.proto) |
| Code Generation | Optional, external tools | Built-in, first-class |
| Streaming | Limited (SSE, chunked) | 4 native patterns |
| Browser Support | ✅ Native | ❌ Needs grpc-web proxy |
| Human Readable | ✅ Easy to debug with curl | ❌ Binary, needs tools |
| Performance | Good | Great (~10x faster serialization) |
| Deadlines | DIY (timeouts) | Built-in propagation |
| Cancellation | Not standard | Built-in, propagates across services |
| Load Balancing | L7 (HTTP path) | L7 (needs gRPC-aware LB) |
| Error Handling | HTTP status codes | gRPC status codes (richer) |
| Caching | ✅ Built-in (Cache-Control, ETags) | ❌ No native caching |
| Tooling | curl, Postman, browser | grpcurl, Postman (limited), BloomRPC |
| Learning Curve | Low | Medium-High |

### Similarities
- Both are request-response at their core
- Both run over HTTP/2 (gRPC always, REST optionally)
- Both support TLS encryption
- Both can use interceptors/middleware
- Both support authentication (tokens, mTLS)
- Both can be load balanced at L7

### Key Differences That Matter in Interviews

**1. Contract-First vs Implementation-First**
- REST: You build the API, then maybe document it
- gRPC: You define the contract first (.proto), then generate code

**2. Caching**
- REST: HTTP caching works naturally (CDN, browser, proxy)
- gRPC: POST-based, so HTTP caching doesn't work. You need application-level caching

**3. Error Model**
- REST: HTTP status codes (limited to ~40 meaningful codes)
- gRPC: 16 status codes + rich error details (error type, retry info, debug info)

```
gRPC Status Codes:
OK (0), CANCELLED (1), UNKNOWN (2), INVALID_ARGUMENT (3),
DEADLINE_EXCEEDED (4), NOT_FOUND (5), ALREADY_EXISTS (6),
PERMISSION_DENIED (7), RESOURCE_EXHAUSTED (8), FAILED_PRECONDITION (9),
ABORTED (10), OUT_OF_RANGE (11), UNIMPLEMENTED (12),
INTERNAL (13), UNAVAILABLE (14), DATA_LOSS (15), UNAUTHENTICATED (16)
```

**4. Deadline Propagation**
```
REST:
  Service A (timeout: 5s) → Service B (timeout: 5s) → Service C (timeout: 5s)
  Total possible wait: 15s 😬

gRPC:
  Service A (deadline: 5s) → Service B (remaining: 4.2s) → Service C (remaining: 3.1s)
  Deadline propagates automatically, all services respect the same deadline ✅
```

---

## Where gRPC Wins Over HTTP/REST

### 1. Microservice-to-Microservice Communication
- Binary serialization = less CPU, less bandwidth
- Type safety catches breaking changes at compile time
- Streaming enables real-time data flow between services

### 2. Polyglot Environments
- One `.proto` file generates clients in Go, Java, Python, C++, etc.
- All services speak the same contract regardless of language
- No hand-written HTTP clients that drift out of sync

### 3. Real-Time Streaming
- Server streaming: live dashboards, price feeds, log streaming
- Bidirectional: chat, collaborative editing, gaming
- Built-in flow control prevents fast producers from overwhelming slow consumers

### 4. Low-Latency, High-Throughput Systems
- Protobuf is ~10x faster to serialize than JSON
- HTTP/2 multiplexing = fewer connections, less overhead
- Connection pooling and keep-alive by default

### 5. Mobile Applications (with caveats)
- Smaller payloads = less data usage
- HTTP/2 = fewer connections = better battery life
- But needs grpc-web or a proxy for browser clients

## Where HTTP/REST Wins Over gRPC

### 1. Public-Facing APIs
- Browsers natively speak HTTP + JSON
- Easy to test with curl, Postman, browser
- Human-readable payloads for debugging

### 2. Caching
- HTTP caching (CDN, browser, proxy) works out of the box
- gRPC uses POST for everything — no HTTP caching

### 3. Simplicity
- Lower learning curve
- No protoc compiler, no code generation step
- Easier onboarding for new developers

### 4. Ecosystem & Tooling
- Massive ecosystem of HTTP tools, libraries, middleware
- API gateways, rate limiters, WAFs all understand HTTP natively
- gRPC needs specialized infrastructure

---

## gRPC Built-in Features

### Deadlines & Timeouts
```go
// Client sets a deadline
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

response, err := client.GetOrder(ctx, &GetOrderRequest{OrderId: "123"})
if err != nil {
    // Could be DEADLINE_EXCEEDED
}
```
The deadline propagates through the entire call chain. If Service A calls B calls C, and the deadline expires, ALL of them get cancelled.

### Cancellation
```go
// Client cancels the call
ctx, cancel := context.WithCancel(context.Background())

go func() {
    time.Sleep(2 * time.Second)
    cancel() // Cancel the RPC
}()

response, err := client.GetOrder(ctx, request)
// err will be CANCELLED
```

### Metadata (like HTTP Headers)
```go
// Client sends metadata
md := metadata.Pairs("authorization", "Bearer token123")
ctx := metadata.NewOutgoingContext(context.Background(), md)
response, err := client.GetOrder(ctx, request)

// Server reads metadata
md, ok := metadata.FromIncomingContext(ctx)
token := md["authorization"][0]
```

### Interceptors (Middleware)
```go
// Unary client interceptor — runs on every RPC call
func loggingInterceptor(
    ctx context.Context,
    method string,
    req, reply interface{},
    cc *grpc.ClientConn,
    invoker grpc.UnaryInvoker,
    opts ...grpc.CallOption,
) error {
    start := time.Now()
    err := invoker(ctx, method, req, reply, cc, opts...)
    log.Printf("RPC: %s, Duration: %s, Error: %v", method, time.Since(start), err)
    return err
}
```

### Load Balancing
gRPC supports two models:
- **Proxy-based (L7):** Envoy, NGINX, AWS ALB — the proxy understands gRPC and distributes requests
- **Client-side:** The client knows about all server instances and picks one (round-robin, weighted, etc.)

```
Proxy-based:
  Client → [Envoy/NGINX] → Server 1
                          → Server 2
                          → Server 3

Client-side:
  Client (knows all servers) → Server 1
                              → Server 2
                              → Server 3
```

### Interview Tip 💡
> Standard L4 load balancers (TCP-level) don't work well with gRPC because HTTP/2 uses a single long-lived connection. The LB sends all traffic to one server. You need either an L7 (application-aware) load balancer or client-side load balancing.

---

## Real-World Production Use Cases

### 1. Google (Creator of gRPC)
- **Internally:** All Google services communicate via Stubby (gRPC's predecessor), handling **billions of RPCs per second**
- **Externally:** Google Cloud APIs offer gRPC endpoints alongside REST
- **Why:** Polyglot environment (C++, Java, Go, Python), need for extreme performance

### 2. Netflix
- Uses gRPC for **inter-service communication** between microservices
- Replaced their custom RPC framework with gRPC
- **Why:** Type safety across 1000+ microservices, streaming for real-time data pipelines

### 3. Uber
- Uses gRPC for their **microservice mesh** (thousands of services)
- Built custom load balancing and service discovery on top of gRPC
- **Why:** Low latency for ride matching, real-time location streaming

### 4. Slack
- Uses gRPC for **backend service communication**
- REST for public API, gRPC internally
- **Why:** Performance at scale, type safety for rapid development

### 5. Dropbox
- Migrated from REST to gRPC for internal services
- Reported **~50% reduction in latency** and significant bandwidth savings
- **Why:** Courier (their gRPC framework) provides observability, retries, and deadline propagation

### 6. Square/Block
- Uses gRPC for **payment processing** between services
- **Why:** Type safety is critical for financial transactions, deadline propagation prevents hung payments

### 7. Spotify
- Uses gRPC for **backend microservices**
- Streaming RPCs for real-time music playback coordination
- **Why:** Low latency, streaming support, polyglot services

### Production Architecture Pattern
```
                    ┌─────────────┐
  Browser/Mobile ──→│ API Gateway │──→ REST/GraphQL (public)
                    │ (Envoy)     │
                    └──────┬──────┘
                           │ gRPC (internal)
              ┌────────────┼────────────┐
              ↓            ↓            ↓
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ Service A│ │ Service B│ │ Service C│
        │ (Go)     │ │ (Java)   │ │ (Python) │
        └────┬─────┘ └────┬─────┘ └────┬─────┘
             │ gRPC        │ gRPC       │ gRPC
             ↓             ↓            ↓
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ Service D│ │ Service E│ │ Service F│
        └──────────┘ └──────────┘ └──────────┘
```

**Pattern:** REST/GraphQL at the edge (for browsers), gRPC internally (for performance).

---

## Common Interview Questions

### Q: When would you choose gRPC over REST?
- Internal microservice communication where performance matters
- Polyglot environments needing shared contracts
- Real-time streaming requirements (live data, events)
- When type safety and compile-time checks are critical (financial systems)
- When you need deadline propagation across service chains

### Q: When would you NOT use gRPC?
- Public APIs consumed by browsers (no native browser support)
- When you need HTTP caching (CDN, browser cache)
- Simple CRUD apps where REST is sufficient
- When your team is small and the overhead of protoc/codegen isn't worth it
- When human-readable debugging is important (JSON is easier to inspect)

### Q: How does gRPC handle backward compatibility?
- Protobuf field numbers are the key — never reuse them
- New fields are ignored by old clients (forward compatible)
- Removed fields are ignored by new clients (backward compatible)
- Never change field types or numbers
- Use `reserved` keyword to prevent accidental reuse of deleted field numbers

### Q: How would you migrate from REST to gRPC?
1. Start with the most performance-critical internal service
2. Define `.proto` files that mirror your existing API
3. Run both REST and gRPC endpoints simultaneously (dual-stack)
4. Migrate internal callers to gRPC one by one
5. Keep REST for external/public APIs
6. Use an API gateway (Envoy) to translate REST ↔ gRPC at the edge

### Q: How do you handle authentication in gRPC?
- **Token-based:** Pass JWT/Bearer tokens via metadata (like HTTP headers)
- **mTLS:** Mutual TLS — both client and server present certificates
- **Interceptors:** Validate auth in a server interceptor (middleware)
- **Per-RPC credentials:** Attach credentials to individual calls
```go
// Token-based auth via interceptor
func authInterceptor(ctx context.Context, req interface{}, 
    info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    md, _ := metadata.FromIncomingContext(ctx)
    token := md["authorization"][0]
    if !validateToken(token) {
        return nil, status.Errorf(codes.Unauthenticated, "invalid token")
    }
    return handler(ctx, req)
}
```

### Q: gRPC uses HTTP/2, so why can't browsers use gRPC directly?
- Browsers don't expose raw HTTP/2 frames to JavaScript
- gRPC uses HTTP/2 trailers for status codes — browsers don't support reading trailers via fetch/XHR
- gRPC's binary framing (length-prefixed Protobuf) doesn't map to browser APIs
- **Solution:** grpc-web — a modified protocol that works with browser limitations, requires an Envoy proxy to translate

### Q: How do you handle errors in gRPC?
```go
// Server returns rich error
st := status.New(codes.InvalidArgument, "invalid order ID")
st, _ = st.WithDetails(&errdetails.BadRequest{
    FieldViolations: []*errdetails.BadRequest_FieldViolation{
        {Field: "order_id", Description: "must be a valid UUID"},
    },
})
return nil, st.Err()

// Client handles error
st := status.Convert(err)
if st.Code() == codes.InvalidArgument {
    for _, detail := range st.Details() {
        if badReq, ok := detail.(*errdetails.BadRequest); ok {
            // Handle field violations
        }
    }
}
```

### Q: Design a real-time notification system — would you use gRPC?
- **Yes, server streaming gRPC** is a great fit
- Client opens a stream: `rpc StreamNotifications(User) returns (stream Notification)`
- Server pushes notifications as they arrive
- Built-in flow control prevents overwhelming the client
- Deadline/cancellation for cleanup
- **Alternative:** If clients are browsers, use SSE or WebSockets instead (no grpc-web overhead)

### Q: How does gRPC handle service discovery and load balancing?
- **Service discovery:** Integrate with Consul, etcd, Kubernetes DNS, or Eureka
- **Client-side LB:** gRPC has a built-in resolver + balancer API
  - `dns:///service-name` — DNS-based discovery
  - Custom resolvers for Consul/etcd
  - Round-robin, pick-first, or custom policies
- **Proxy-based LB:** Envoy, Linkerd, Istio — L7 load balancing that understands gRPC
- **Why L7 matters:** HTTP/2 multiplexes on one connection, so L4 (TCP) LB sends all requests to one backend

---

## gRPC Ecosystem & Tooling

| Tool | Purpose |
|------|---------|
| `protoc` | Protocol Buffer compiler — generates code from .proto files |
| `grpcurl` | Like curl but for gRPC — CLI tool for testing |
| `BloomRPC` / `Postman` | GUI tools for testing gRPC services |
| `Envoy` | L7 proxy with native gRPC support, gRPC-JSON transcoding |
| `grpc-web` | Enables browser clients to call gRPC services |
| `grpc-gateway` | Generates REST reverse proxy from gRPC service definitions |
| `buf` | Modern Protobuf toolchain (linting, breaking change detection, registry) |
| `Connect` | Buf's gRPC-compatible framework that also works with curl and browsers |

---

## Decision Framework

```
Is the client a browser?
├── Yes → Use REST/GraphQL (or grpc-web with proxy if you really need gRPC)
└── No → Is it service-to-service?
          ├── Yes → Do you need streaming?
          │         ├── Yes → gRPC ✅
          │         └── No → Is performance critical?
          │                   ├── Yes → gRPC ✅
          │                   └── No → REST is fine, gRPC is better if polyglot
          └── No → Is it a mobile app?
                    ├── Yes → gRPC (smaller payloads, less battery) or REST
                    └── No → REST for simplicity
```

---

## Further Reading
- [[http]] — the protocol gRPC is built on top of
- [[protobuf]] — deep dive into Protocol Buffers serialization
- [[websockets]] — alternative for browser-based real-time communication
- [[service-mesh]] — Envoy, Istio, Linkerd — infrastructure that powers gRPC at scale
- [[load-balancing]] — L4 vs L7 and why it matters for gRPC

---

*Last updated: 2025-03-13*
