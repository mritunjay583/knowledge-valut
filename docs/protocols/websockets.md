---
title: "WebSockets"
layout: default
parent: Protocols
nav_order: 3
---

# WebSockets — Deep Dive

> A comprehensive guide to WebSockets — why they exist, what problems they solve, how they compare to HTTP and gRPC, architecture internals, and real-world production use cases. Written for system design interviews and engineering decisions.

---

## What are WebSockets?

WebSocket is a **full-duplex, persistent communication protocol** that enables real-time, bidirectional data exchange between a client and server over a single, long-lived TCP connection.

- Defined in **RFC 6455**
- Starts as an HTTP request, then **upgrades** to a WebSocket connection
- Runs on ports **80** (ws://) and **443** (wss:// — encrypted)
- Sits at **Layer 7** (Application Layer) but behaves more like a raw TCP pipe
- Supported natively in **all modern browsers**
- Lightweight framing — minimal overhead per message (~2-14 bytes)

---

## Why WebSockets? What Problem Do They Solve?

### The Problem with HTTP for Real-Time Communication

HTTP is request-response. The client always initiates. The server can never push data to the client on its own. This creates real pain for real-time features:

**Polling (the naive approach):**
```
Client → GET /messages (any new?) → Server: "No"     (wasted request)
Client → GET /messages (any new?) → Server: "No"     (wasted request)
Client → GET /messages (any new?) → Server: "Yes, here's 1 message"
Client → GET /messages (any new?) → Server: "No"     (wasted request)
```
- Wastes bandwidth and server resources
- Latency = polling interval (if you poll every 3s, messages are delayed up to 3s)
- Doesn't scale — 100K users polling every 3s = 33K requests/second for nothing

**Long Polling (slightly better):**
```
Client → GET /messages (hold connection open...)
         ... server waits until there's data ...
Server → "Here's a message"
Client → GET /messages (hold again...)
```
- Better latency, but still one-directional
- Each message requires a new HTTP request (headers, cookies, overhead)
- Connection management is complex (timeouts, reconnection)
- Still doesn't scale well

**Server-Sent Events (SSE — one-way streaming):**
```
Client → GET /events (Accept: text/event-stream)
Server → data: message 1
Server → data: message 2
Server → data: message 3
```
- Server can push, but client can't send through the same connection
- Text-only (no binary)
- Built on HTTP — works with proxies and CDNs
- Good for notifications, but not for chat or collaboration

### What WebSockets Solve
```
Client ←→ Server (both can send anytime, no request needed)

Client: "Hello"
Server: "Hi there"
Server: "New notification"    ← server pushes without client asking
Client: "Typing..."
Server: "User X is typing..."
Client: "Send message"
Server: "Message delivered"
```

| Problem | HTTP | WebSocket |
|---------|------|-----------|
| Server push | ❌ Client must ask | ✅ Server sends anytime |
| Latency | Polling interval or new request | Near-zero (persistent connection) |
| Overhead per message | ~800 bytes (headers) | ~2-14 bytes (frame header) |
| Direction | One-way (request → response) | Full-duplex (both ways simultaneously) |
| Connection | New or reused per request | Single persistent connection |
| Scalability for real-time | Poor (polling wastes resources) | Good (idle connections are cheap) |

### Interview Tip 💡
> WebSockets don't replace HTTP — they solve a specific problem HTTP can't: **low-latency, bidirectional, server-initiated communication**. If you don't need the server to push data, plain HTTP is simpler and better.

---

## WebSocket Architecture

### The Handshake — HTTP Upgrade

WebSocket connections start as a normal HTTP request, then upgrade:

**Client Request:**
```http
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Sec-WebSocket-Protocol: chat, superchat
Origin: http://example.com
```

**Server Response:**
```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
```

After this handshake, the HTTP connection is **upgraded** to a WebSocket connection. The TCP socket stays open and both sides can send data freely.

### Key Handshake Details
- `Sec-WebSocket-Key` — random base64 value from client (prevents caching proxies from replaying)
- `Sec-WebSocket-Accept` — server hashes the key with a magic GUID to prove it understands WebSocket
- `101 Switching Protocols` — the only valid success response
- After 101, HTTP is done — the connection is now a raw WebSocket pipe

### Connection Lifecycle
```
┌────────┐         HTTP Upgrade          ┌────────┐
│ Client │ ──────────────────────────────→│ Server │
│        │ ←─── 101 Switching Protocols ──│        │
│        │                                │        │
│        │ ══════ WebSocket Open ═════════│        │
│        │                                │        │
│        │ ──── Text/Binary Frame ──────→ │        │
│        │ ←─── Text/Binary Frame ─────── │        │
│        │ ──── Text/Binary Frame ──────→ │        │
│        │ ←─── Ping ──────────────────── │        │
│        │ ──── Pong ──────────────────→  │        │
│        │ ←─── Text/Binary Frame ─────── │        │
│        │                                │        │
│        │ ──── Close Frame ────────────→ │        │
│        │ ←─── Close Frame ──────────── │        │
│        │                                │        │
│        │ ══════ TCP Closed ════════════ │        │
└────────┘                                └────────┘
```

### WebSocket Frame Format
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |           (16/64)             |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+-------------------------------+
|     Masking-key (0 or 4 bytes)                                |
+---------------------------------------------------------------+
|     Payload Data                                              |
+---------------------------------------------------------------+
```

- **FIN** — is this the final fragment? (1 bit)
- **Opcode** — frame type: `0x1` text, `0x2` binary, `0x8` close, `0x9` ping, `0xA` pong
- **MASK** — client-to-server frames MUST be masked (security against cache poisoning)
- **Payload length** — 7 bits (0-125), or 16 bits (126), or 64 bits (127)
- **Overhead** — just 2-14 bytes per frame vs ~800 bytes for HTTP headers

### Interview Tip 💡
> The masking in client→server frames isn't for encryption — it's to prevent malicious JavaScript from crafting bytes that look like valid HTTP to confuse transparent proxies. Server→client frames are NOT masked.

---

## WebSocket vs HTTP vs gRPC — Complete Three-Way Comparison

| Aspect | HTTP/REST | gRPC | WebSocket |
|--------|-----------|------|-----------|
| Transport | TCP (HTTP/1.1) or TCP (HTTP/2) | TCP (HTTP/2) | TCP (upgraded from HTTP) |
| Connection | Short-lived or keep-alive | Persistent (HTTP/2 stream) | Persistent (single TCP) |
| Direction | Request → Response | 4 patterns (unary + streaming) | Full-duplex (both ways anytime) |
| Who initiates? | Always the client | Client initiates, streaming varies | Either side, anytime |
| Data format | JSON/XML (text) | Protobuf (binary) | Any (text or binary) |
| Schema/Contract | Optional (OpenAPI) | Required (.proto) | None — raw messages |
| Overhead per msg | ~800 bytes (headers) | ~20-50 bytes (HTTP/2 + Protobuf) | ~2-14 bytes (frame header) |
| Browser support | ✅ Native | ❌ Needs grpc-web | ✅ Native |
| Server push | ❌ (polling/SSE workaround) | ✅ (server streaming) | ✅ Native |
| Type safety | ❌ Runtime | ✅ Compile-time | ❌ Runtime |
| Built-in features | Caching, redirects, auth | Deadlines, cancellation, LB | Ping/pong heartbeat only |
| Scalability model | Stateless (easy to scale) | Stateless per RPC | Stateful (connection affinity) |
| Load balancing | Simple (any L7 LB) | Needs gRPC-aware L7 LB | Sticky sessions needed |
| Reconnection | N/A (new request) | Channel handles it | Manual (you implement it) |
| Multiplexing | HTTP/2 only | Yes (HTTP/2 streams) | No (one connection = one channel) |
| Caching | ✅ HTTP caching | ❌ No native caching | ❌ No caching |
| Use case | Public APIs, CRUD, web pages | Internal microservices, streaming RPC | Real-time UI: chat, gaming, collab |

### Similarities Across All Three
- All use TCP as the underlying transport
- All support TLS encryption (HTTPS, gRPC+TLS, WSS)
- All work at the application layer (Layer 7)
- All can carry JSON data (though gRPC prefers Protobuf)
- All support authentication via tokens/headers
- All can be proxied through Nginx/Envoy

### Key Differences That Matter

**1. Statefulness**
- HTTP: Stateless — each request is independent, easy to scale horizontally
- gRPC: Stateless per RPC, but HTTP/2 connection is persistent
- WebSocket: **Stateful** — the server must remember each connection. This is the biggest scaling challenge

**2. Message Structure**
- HTTP: Structured (method, path, headers, body) — the protocol defines semantics
- gRPC: Structured (service, method, typed messages) — the contract defines semantics
- WebSocket: **Unstructured** — just raw text or binary frames. YOU define the message format

**3. When the Server Needs to Talk First**
- HTTP: Impossible (client must ask)
- gRPC: Server streaming (but client must initiate the stream)
- WebSocket: Server sends anytime — true push

---

## Where WebSockets Win

### Over HTTP
| Scenario | Why WebSocket Wins |
|----------|--------------------|
| Chat applications | Server pushes messages instantly, no polling |
| Live notifications | Zero-latency push, no wasted requests |
| Collaborative editing (Google Docs) | Both sides send changes in real-time |
| Live sports scores / stock tickers | Continuous server push, minimal overhead |
| Online gaming | Low-latency bidirectional state sync |
| Live dashboards | Server streams metrics as they change |

### Over gRPC
| Scenario | Why WebSocket Wins |
|----------|--------------------|
| Browser-based real-time apps | Native browser support, no proxy needed |
| Unstructured messaging | No schema needed, flexible message formats |
| Simple real-time features | Lower complexity, no protoc/codegen |
| When you need raw speed in browsers | Direct TCP-like connection, minimal overhead |

## Where WebSockets Lose

### To HTTP
| Scenario | Why HTTP Wins |
|----------|---------------|
| CRUD APIs | Request-response is simpler, cacheable |
| Public APIs | Stateless, scalable, well-understood |
| SEO / web pages | HTTP is the web's native protocol |
| When you need caching | HTTP caching (CDN, browser) works natively |
| Infrequent updates | Polling every 30s is simpler than maintaining a connection |

### To gRPC
| Scenario | Why gRPC Wins |
|----------|---------------|
| Microservice communication | Type safety, deadlines, cancellation |
| Polyglot backends | Code generation from .proto |
| Structured streaming | Typed messages, flow control, backpressure |
| When you need both streaming + RPC | gRPC has 4 patterns built-in |

### Interview Tip 💡
> The decision isn't "WebSocket vs HTTP vs gRPC" — they serve different purposes. A real system uses all three: HTTP for public APIs, gRPC for internal services, WebSocket for real-time browser features. Know when to pick each.

---

## Scaling WebSockets — The Hard Part

WebSockets are **stateful**. This is the single biggest challenge. Here's why and how to solve it.

### The Problem
```
User A connects to Server 1 → WebSocket open on Server 1
User B connects to Server 2 → WebSocket open on Server 2

User A sends message to User B...
But Server 1 doesn't know about User B's connection on Server 2 😬
```

### Solution 1: Sticky Sessions
```
Load Balancer (sticky by user ID or IP)
  ├── User A → always Server 1
  ├── User B → always Server 2
  └── User C → always Server 1
```
- Simple but limits horizontal scaling
- If Server 1 dies, User A and C lose their connections

### Solution 2: Pub/Sub Backbone (The Standard Approach)
```
┌──────────┐     ┌──────────┐     ┌──────────┐
│ Server 1 │     │ Server 2 │     │ Server 3 │
│ (User A) │     │ (User B) │     │ (User C) │
└────┬─────┘     └────┬─────┘     └────┬─────┘
     │                │                │
     └────────┬───────┴────────┬───────┘
              │                │
         ┌────┴────┐     ┌────┴────┐
         │  Redis  │ or  │  Kafka  │
         │  Pub/Sub│     │         │
         └─────────┘     └─────────┘
```

1. User A sends a message on Server 1
2. Server 1 publishes to Redis Pub/Sub channel
3. Server 2 (subscribed) receives it and pushes to User B
4. All servers subscribe to relevant channels

### Solution 3: Dedicated Connection Servers
```
┌─────────────┐         ┌──────────────────┐
│ API Servers  │ ←─────→ │ Connection Servers│ (WebSocket only)
│ (stateless)  │  gRPC   │ (stateful)        │
└─────────────┘         └────────┬─────────┘
                                 │
                          ┌──────┴──────┐
                          │ Redis/Kafka │
                          └─────────────┘
```
- Separate WebSocket handling from business logic
- Connection servers are stateful but simple (just routing)
- API servers remain stateless and easy to scale

### Connection Limits
- Each WebSocket = 1 TCP connection = 1 file descriptor
- Linux default: ~1024 file descriptors per process
- Tuned servers: **1M+ concurrent connections** per server (with epoll/kqueue)
- Memory: ~10-50 KB per connection (buffers, state)
- A single server with 32GB RAM can handle ~500K-1M connections

### Interview Tip 💡
> When an interviewer asks "how would you scale a chat system," the answer is: WebSocket servers + Redis Pub/Sub for cross-server messaging + sticky sessions or consistent hashing for connection routing. This is the industry-standard pattern used by Slack, Discord, and WhatsApp Web.

---

## WebSocket Subprotocols and Message Design

WebSocket gives you a raw pipe. You need to define your own message protocol on top.

### Common Patterns

**JSON-based protocol (most common):**
```json
// Client → Server
{"type": "message", "to": "user_123", "content": "Hello!"}
{"type": "typing", "channel": "room_456"}
{"type": "subscribe", "channels": ["room_456", "room_789"]}

// Server → Client
{"type": "message", "from": "user_789", "content": "Hi!", "timestamp": 1710000000}
{"type": "presence", "user": "user_789", "status": "online"}
{"type": "error", "code": 4001, "message": "Not authorized for this channel"}
```

**Binary protocol (for performance):**
```
[1 byte: message type][4 bytes: length][payload]
```
Used in gaming, financial trading, or high-throughput scenarios.

### Subprotocol Negotiation
```http
// Client offers subprotocols
Sec-WebSocket-Protocol: chat-v2, chat-v1

// Server picks one
Sec-WebSocket-Protocol: chat-v2
```

### STOMP (Simple Text Oriented Messaging Protocol)
A popular subprotocol for message brokers:
```
SUBSCRIBE
destination:/topic/chat/room123
id:sub-0

^@

MESSAGE
destination:/topic/chat/room123
content-type:application/json

{"user": "alice", "text": "Hello!"}
^@
```

---

## Heartbeat and Reconnection

### Ping/Pong (Protocol Level)
```
Server → Ping frame (opcode 0x9)
Client → Pong frame (opcode 0xA) — automatic in browsers

If no Pong received within timeout → connection is dead → close it
```

### Application-Level Heartbeat
```json
// Every 30 seconds
Client → {"type": "ping"}
Server → {"type": "pong"}
```
Why both? Protocol-level ping/pong detects dead TCP connections. Application-level heartbeat detects dead application logic.

### Reconnection Strategy
```javascript
class ReconnectingWebSocket {
  connect() {
    this.ws = new WebSocket(this.url);
    this.ws.onclose = () => this.reconnect();
    this.ws.onerror = () => this.reconnect();
  }

  reconnect() {
    // Exponential backoff: 1s, 2s, 4s, 8s, 16s... max 30s
    const delay = Math.min(1000 * Math.pow(2, this.retryCount), 30000);
    // Add jitter to prevent thundering herd
    const jitter = delay * 0.3 * Math.random();
    setTimeout(() => {
      this.retryCount++;
      this.connect();
    }, delay + jitter);
  }

  // After reconnect, re-subscribe and fetch missed messages
  onReconnected() {
    this.resubscribe();
    this.fetchMissedMessages(this.lastMessageTimestamp);
  }
}
```

### Interview Tip 💡
> Always mention **exponential backoff with jitter** for reconnection. Without jitter, if a server restarts and 100K clients reconnect simultaneously, they'll all retry at the same intervals creating a thundering herd that crashes the server again.

---

## Security Considerations

### Authentication
WebSocket has NO built-in auth mechanism. Common approaches:

**1. Token in query string (simple but less secure):**
```javascript
new WebSocket("wss://server.com/ws?token=jwt_token_here");
// ⚠️ Token visible in server logs and browser history
```

**2. Cookie-based (if same origin):**
```javascript
// Browser automatically sends cookies during handshake
new WebSocket("wss://server.com/ws");
// Server validates session cookie in the Upgrade request
```

**3. Auth during handshake + first message (recommended):**
```javascript
const ws = new WebSocket("wss://server.com/ws");
ws.onopen = () => {
  ws.send(JSON.stringify({ type: "auth", token: "jwt_token_here" }));
};
// Server validates token, closes connection if invalid
```

### Common Attacks and Mitigations

| Attack | Description | Mitigation |
|--------|-------------|------------|
| Cross-Site WebSocket Hijacking | Malicious page opens WS to your server using victim's cookies | Check `Origin` header during handshake |
| DoS (connection flooding) | Attacker opens thousands of connections | Rate limit connections per IP, max connections per user |
| Message flooding | Attacker sends messages at extreme rate | Rate limit messages per connection |
| Large payload attack | Attacker sends huge frames to exhaust memory | Set max frame size limit |
| Unencrypted traffic | Data readable in transit | Always use `wss://` (WebSocket over TLS) |

---

## Real-World Production Use Cases

### 1. Slack
- **WebSocket for real-time messaging** in the browser/desktop client
- Every channel update, typing indicator, presence change goes through WebSocket
- Backend uses a gateway service that manages millions of concurrent connections
- Falls back to long polling if WebSocket fails
- **Scale:** Millions of concurrent WebSocket connections

### 2. Discord
- **WebSocket is the core protocol** for all real-time features
- Voice channels use WebSocket for signaling + WebRTC for audio
- Custom binary protocol on top of WebSocket for efficiency
- Gateway servers handle connection management, separate from business logic
- **Scale:** Millions of concurrent users, billions of messages per day

### 3. Figma / Google Docs (Collaborative Editing)
- WebSocket carries **operational transforms (OT)** or **CRDTs** between collaborators
- Every cursor move, text edit, shape drag is a WebSocket message
- Server acts as the source of truth, broadcasting changes to all connected clients
- **Why WebSocket:** Sub-50ms latency needed for smooth collaboration

### 4. Binance / Coinbase (Financial Trading)
- **WebSocket for real-time price feeds** — prices update multiple times per second
- Order book updates, trade executions, portfolio changes
- Binary WebSocket frames for maximum throughput
- **Why WebSocket:** HTTP polling would be too slow and too expensive at this frequency

### 5. Uber / Lyft (Live Location Tracking)
- Driver app sends GPS coordinates via WebSocket every few seconds
- Rider app receives real-time driver location via WebSocket
- **Why WebSocket:** Continuous bidirectional updates, low overhead per message

### 6. Multiplayer Games (Fortnite Web, Agar.io)
- Game state synchronization between players
- Player actions sent to server, server broadcasts world state
- Often use binary WebSocket frames for minimal latency
- **Why WebSocket:** Full-duplex, low overhead, browser-native

### 7. Live Sports / Streaming Platforms
- ESPN, Cricbuzz — live score updates
- Twitch — chat alongside video streams
- **Why WebSocket:** Server pushes updates instantly to millions of viewers

### Production Architecture Pattern
```
                        ┌─────────────────┐
    Browser/Mobile ────→│  Load Balancer   │ (L7, sticky sessions)
                        │  (Nginx/Envoy)   │
                        └────────┬────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ↓                  ↓                  ↓
     ┌────────────────┐ ┌────────────────┐ ┌────────────────┐
     │ WS Gateway 1   │ │ WS Gateway 2   │ │ WS Gateway 3   │
     │ (connections)   │ │ (connections)   │ │ (connections)   │
     └───────┬────────┘ └───────┬────────┘ └───────┬────────┘
             │                  │                  │
             └──────────┬───────┴──────────┬───────┘
                        │                  │
                   ┌────┴────┐      ┌──────┴──────┐
                   │  Redis  │      │  API Servers │
                   │ Pub/Sub │      │  (stateless) │
                   └─────────┘      └─────────────┘
```

- **WS Gateways:** Stateful, manage connections, minimal logic
- **API Servers:** Stateless, handle business logic, publish events to Redis
- **Redis Pub/Sub:** Bridges messages across gateway servers
- **Load Balancer:** Sticky sessions (by connection ID or user ID)

---

## Common Interview Questions

### Q: Design a real-time chat system (like Slack/WhatsApp Web)
1. **Connection:** Client opens WebSocket to a gateway server
2. **Auth:** First message is auth token, server validates and maps connection to user ID
3. **Sending:** Client sends message → gateway → API server → store in DB → publish to Redis
4. **Receiving:** Redis Pub/Sub → recipient's gateway server → push via WebSocket
5. **Offline users:** Messages stored in DB, delivered when they reconnect
6. **Scaling:** Multiple gateway servers + Redis Pub/Sub for cross-server routing
7. **Presence:** Heartbeat every 30s, mark offline after 2 missed heartbeats

### Q: WebSocket vs SSE — when would you pick each?
| | WebSocket | SSE |
|--|-----------|-----|
| Direction | Bidirectional | Server → Client only |
| Protocol | Custom binary/text | HTTP (text/event-stream) |
| Reconnection | Manual | Automatic (built-in) |
| Browser support | ✅ All modern | ✅ All modern (except old IE) |
| Through proxies/CDN | Can be tricky | Works naturally (it's HTTP) |
| Binary data | ✅ Yes | ❌ Text only |
| **Pick when** | Chat, gaming, collab editing | Notifications, live feeds, dashboards |

**Rule of thumb:** If the client needs to send frequent messages → WebSocket. If only the server pushes → SSE is simpler.

### Q: How do you handle message ordering and delivery guarantees?
- **Ordering:** Assign sequence numbers to messages. Client buffers and reorders if needed
- **At-least-once delivery:** Server stores unacknowledged messages, resends on reconnect
- **Exactly-once:** Use message IDs + client-side deduplication
- **Missed messages on reconnect:** Client sends last received sequence number, server replays from there
```json
// Message with sequence number
{"seq": 42, "type": "message", "content": "Hello"}

// Client reconnects
{"type": "resume", "last_seq": 41}
// Server replays messages from seq 42 onwards
```

### Q: How would you handle 10 million concurrent WebSocket connections?
- **Horizontal scaling:** Multiple gateway servers behind a load balancer
- **Connection limits:** Each server handles ~500K-1M connections (tune file descriptors, use epoll)
- **Cross-server messaging:** Redis Pub/Sub or Kafka for routing
- **Consistent hashing:** Route users to specific servers to reduce Pub/Sub fan-out
- **Connection state:** Store in Redis (user → server mapping)
- **Graceful shutdown:** Drain connections to other servers before shutting down a node
- **Estimated infra:** ~15-20 gateway servers + Redis cluster + API server fleet

### Q: What happens when a WebSocket server crashes?
1. All connections on that server are dropped
2. Clients detect disconnect (onclose event)
3. Clients reconnect with exponential backoff + jitter
4. Load balancer routes them to healthy servers
5. Clients send last received sequence number
6. Server replays missed messages from persistent storage
7. **Key insight:** Messages must be persisted (DB/Kafka) before being sent via WebSocket, so nothing is lost

### Q: How is WebSocket different from TCP sockets?
- WebSocket runs **on top of TCP** (it's an application-layer protocol)
- WebSocket adds framing (message boundaries), TCP is a byte stream
- WebSocket works through HTTP proxies and firewalls (starts as HTTP)
- WebSocket has built-in ping/pong for keepalive
- Raw TCP sockets are not available in browsers — WebSocket is the browser's "TCP"

### Q: Can you use WebSocket with a CDN?
- Most CDNs now support WebSocket (Cloudflare, AWS CloudFront, Fastly)
- The CDN terminates TLS and proxies the WebSocket connection to your origin
- No caching benefit (WebSocket is real-time, nothing to cache)
- Benefit is mainly **global edge presence** — users connect to nearest CDN node, reducing latency for the initial handshake

---

## Decision Framework

```
Does the server need to push data to the client?
├── No → Use HTTP/REST
└── Yes → How frequently?
          ├── Rarely (every few minutes) → HTTP polling or SSE
          └── Frequently (sub-second) → Does the client also send frequently?
                                        ├── No → SSE (simpler, HTTP-native)
                                        └── Yes → Is it browser-based?
                                                  ├── Yes → WebSocket ✅
                                                  └── No (service-to-service) →
                                                      Need type safety? → gRPC bidi streaming
                                                      Just raw speed? → WebSocket or raw TCP
```

---

## Further Reading
- http — the protocol WebSocket upgrades from
- grpc — alternative for typed streaming between services
- sse — simpler alternative for server-push only scenarios
- tcp — the transport layer WebSocket runs on
- redis-pubsub — the backbone for scaling WebSocket across servers
- load-balancing — sticky sessions and connection-aware routing

---

*Last updated: 2025-03-13*
