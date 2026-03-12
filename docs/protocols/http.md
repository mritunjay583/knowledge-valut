---
title: "HTTP Protocol"
layout: default
parent: Protocols
nav_order: 1
---

# HTTP Protocol

> A comprehensive guide to HTTP — from fundamentals to optimization strategies. Written for system design interviews and real-world engineering decisions.

---

## What is HTTP?

HTTP (HyperText Transfer Protocol) is an **application-layer protocol** built on top of TCP (and now UDP with HTTP/3). It follows a **request-response model** where a client sends a request and a server returns a response.

- Stateless by design — each request is independent
- Text-based in HTTP/1.x, binary in HTTP/2+
- Default port: **80** (HTTP), **443** (HTTPS)
- Sits at **Layer 7** (Application Layer) of the OSI model

---

## HTTP Message Structure

### Request

```
METHOD /path HTTP/1.1
Host: example.com
Headers...

Body (optional)
```

### Response

```
HTTP/1.1 200 OK
Headers...

Body
```

Key parts:
- **Start line** — method + path + version (request) or version + status code (response)
- **Headers** — key-value metadata pairs
- **Body** — optional payload (present in POST, PUT, PATCH, and most responses)

---

## HTTP Methods

| Method | Safe | Idempotent | Has Body | Use Case |
|---------|------|------------|----------|----------|
| GET | ✅ | ✅ | ❌ | Retrieve a resource |
| HEAD | ✅ | ✅ | ❌ | Same as GET but no body (check if resource exists) |
| POST | ❌ | ❌ | ✅ | Create a resource, submit data |
| PUT | ❌ | ✅ | ✅ | Replace a resource entirely |
| PATCH | ❌ | ❌ | ✅ | Partial update of a resource |
| DELETE | ❌ | ✅ | Optional | Remove a resource |
| OPTIONS | ✅ | ✅ | ❌ | Discover allowed methods (used in CORS preflight) |
| TRACE | ✅ | ✅ | ❌ | Loop-back test (usually disabled for security) |
| CONNECT | ❌ | ❌ | ❌ | Establish a tunnel (used for HTTPS through proxy) |

### Interview Tip 💡
> **Safe** = doesn't modify server state. **Idempotent** = calling it N times has the same effect as calling it once. POST is neither safe nor idempotent — that's why retrying POST requests is dangerous without idempotency keys.

---

## Status Codes

### 1xx — Informational
- `100 Continue` — server received headers, client should send body
- `101 Switching Protocols` — upgrading to WebSocket or HTTP/2
- `103 Early Hints` — preload resources before final response

### 2xx — Success
- `200 OK` — standard success
- `201 Created` — resource created (return in POST)
- `204 No Content` — success but no body (common for DELETE)
- `206 Partial Content` — range request fulfilled (video streaming, resumable downloads)

### 3xx — Redirection
- `301 Moved Permanently` — resource moved, update your bookmarks (cacheable)
- `302 Found` — temporary redirect (historically misused)
- `304 Not Modified` — use your cached version
- `307 Temporary Redirect` — like 302 but preserves HTTP method
- `308 Permanent Redirect` — like 301 but preserves HTTP method

### 4xx — Client Error
- `400 Bad Request` — malformed request
- `401 Unauthorized` — authentication required (misleading name — it means "unauthenticated")
- `403 Forbidden` — authenticated but not authorized
- `404 Not Found` — resource doesn't exist
- `405 Method Not Allowed` — wrong HTTP method
- `409 Conflict` — conflict with current state (e.g., duplicate resource)
- `429 Too Many Requests` — rate limited

### 5xx — Server Error
- `500 Internal Server Error` — generic server failure
- `502 Bad Gateway` — upstream server returned invalid response
- `503 Service Unavailable` — server overloaded or in maintenance
- `504 Gateway Timeout` — upstream server didn't respond in time

### Interview Tip 💡
> Know the difference between **401 vs 403**. 401 = "who are you?" (not authenticated). 403 = "I know who you are, but you can't do this" (not authorized).

---

## HTTP Headers — The Important Ones

### Request Headers
- `Host` — required in HTTP/1.1, identifies the target server (enables virtual hosting)
- `Authorization` — carries credentials (Bearer token, Basic auth)
- `Accept` — what content types the client can handle (`application/json`, `text/html`)
- `Accept-Encoding` — compression algorithms supported (`gzip`, `br`, `deflate`)
- `Content-Type` — MIME type of the request body (`application/json`, `multipart/form-data`)
- `Cookie` — sends stored cookies to server
- `User-Agent` — identifies the client software
- `If-None-Match` / `If-Modified-Since` — conditional requests for caching

### Response Headers
- `Content-Type` — MIME type of the response body
- `Content-Length` — size of the response body in bytes
- `Set-Cookie` — instructs client to store a cookie
- `Cache-Control` — caching directives
- `ETag` — unique identifier for a specific version of a resource
- `Last-Modified` — when the resource was last changed
- `Location` — redirect target URL (used with 3xx)
- `Retry-After` — when to retry (used with 429, 503)
- `Transfer-Encoding: chunked` — body sent in chunks (size unknown upfront)

### Security Headers
- `Strict-Transport-Security (HSTS)` — force HTTPS
- `Content-Security-Policy (CSP)` — prevent XSS by controlling resource loading
- `X-Content-Type-Options: nosniff` — prevent MIME type sniffing
- `X-Frame-Options` — prevent clickjacking (DENY, SAMEORIGIN)
- `Access-Control-Allow-Origin` — CORS header

---

## HTTP Connection Lifecycle

### HTTP/1.0 — One Request Per Connection
```
Client → TCP Handshake → Request → Response → Connection Closed
Client → TCP Handshake → Request → Response → Connection Closed
```
Every request opens a new TCP connection. Extremely wasteful.

### HTTP/1.1 — Persistent Connections (Keep-Alive)
```
Client → TCP Handshake → Request 1 → Response 1 → Request 2 → Response 2 → ... → Close
```
- Connections are **kept alive by default** (`Connection: keep-alive`)
- Supports **pipelining** (send multiple requests without waiting) — but responses must come back **in order** → **Head-of-Line (HOL) blocking**
- Browsers open **6 parallel TCP connections** per domain to work around HOL blocking

### Interview Tip 💡
> HOL blocking in HTTP/1.1 is at the **application layer** — a slow response blocks all subsequent responses on that connection. This is the primary motivation for HTTP/2.

---

## HTTP/2 — Multiplexing Revolution

### Key Features
1. **Binary framing** — messages split into binary frames instead of text
2. **Multiplexing** — multiple streams over a single TCP connection, no HOL blocking at HTTP layer
3. **Header compression (HPACK)** — compresses headers using a static + dynamic table
4. **Server Push** — server can proactively send resources the client will need
5. **Stream prioritization** — client can hint which resources are more important

### How Multiplexing Works
```
Single TCP Connection:
  Stream 1: [Frame] [Frame] [Frame]     → GET /index.html
  Stream 2: [Frame] [Frame]             → GET /style.css
  Stream 3: [Frame] [Frame] [Frame]     → GET /app.js
```
Frames from different streams are interleaved on the same connection. Each stream is independent.

### The Remaining Problem
HTTP/2 solves HOL blocking at the HTTP layer, but **TCP still has HOL blocking**. If a single TCP packet is lost, ALL streams are blocked until retransmission completes. This is why HTTP/3 exists.

---

## HTTP/3 — QUIC and UDP

### Why HTTP/3?
TCP's HOL blocking can't be fixed without changing the transport layer. HTTP/3 replaces TCP with **QUIC** (built on UDP).

### Key Features
1. **QUIC transport** — UDP-based, each stream has independent loss recovery
2. **0-RTT connection setup** — combine transport + TLS handshake (vs 2-3 RTTs for TCP + TLS)
3. **No HOL blocking** — packet loss in one stream doesn't affect others
4. **Connection migration** — connections survive IP changes (great for mobile switching WiFi ↔ cellular)
5. **Built-in encryption** — TLS 1.3 is mandatory, not optional

### Connection Setup Comparison
```
HTTP/1.1 + TLS 1.2:  TCP (1 RTT) + TLS (2 RTT) = 3 RTT before first byte
HTTP/2 + TLS 1.3:    TCP (1 RTT) + TLS (1 RTT) = 2 RTT before first byte
HTTP/3 (QUIC):       QUIC + TLS (1 RTT) = 1 RTT before first byte
                      Resumed connection = 0 RTT ✨
```

### Interview Tip 💡
> HTTP/3 doesn't just "use UDP" — QUIC reimplements reliability, flow control, and congestion control on top of UDP. It's essentially a better TCP that's not limited by OS kernel TCP stacks and middlebox ossification.

---

## HTTP Caching — Deep Dive

Caching is one of the most important HTTP optimization mechanisms. Understanding it deeply is critical for system design.

### Cache-Control Directives

| Directive | Meaning |
|-----------|---------|
| `public` | Any cache (CDN, proxy, browser) can store it |
| `private` | Only the browser can cache (not CDN/proxy) — use for user-specific data |
| `no-cache` | Cache can store it, but MUST revalidate with server before using |
| `no-store` | Don't cache at all — for sensitive data |
| `max-age=N` | Fresh for N seconds from the time of request |
| `s-maxage=N` | Like max-age but only for shared caches (CDN/proxy) |
| `must-revalidate` | Once stale, MUST revalidate — don't serve stale content |
| `stale-while-revalidate=N` | Serve stale content while revalidating in background for N seconds |
| `immutable` | Content will never change — don't revalidate even on refresh |

### Caching Flow
```
Client Request
    ↓
Is response in cache?
    ├── No → Forward to origin server
    └── Yes → Is it fresh? (check max-age / Expires)
              ├── Yes → Serve from cache ✅
              └── No (stale) → Revalidate with origin
                               ├── 304 Not Modified → Serve cached version ✅
                               └── 200 OK → Update cache, serve new version
```

### Validation Mechanisms

**ETag-based (strong validation):**
```
Server → ETag: "abc123"
Client → If-None-Match: "abc123"
Server → 304 Not Modified (or 200 with new content)
```

**Time-based (weak validation):**
```
Server → Last-Modified: Thu, 01 Jan 2025 00:00:00 GMT
Client → If-Modified-Since: Thu, 01 Jan 2025 00:00:00 GMT
Server → 304 Not Modified (or 200 with new content)
```

### Real-World Caching Strategy
```
# Static assets with content hash in filename (app.a1b2c3.js)
Cache-Control: public, max-age=31536000, immutable

# HTML pages
Cache-Control: no-cache

# API responses (user-specific)
Cache-Control: private, max-age=0, must-revalidate

# API responses (public, can be slightly stale)
Cache-Control: public, max-age=60, stale-while-revalidate=300
```

### Interview Tip 💡
> `no-cache` does NOT mean "don't cache." It means "cache it, but always check with the server first." If you want zero caching, use `no-store`. This is a classic interview gotcha.

---

## HTTPS and TLS

HTTPS = HTTP over TLS. TLS provides:
1. **Encryption** — data can't be read in transit
2. **Authentication** — server proves its identity via certificates
3. **Integrity** — data can't be tampered with

### TLS 1.3 Handshake (Simplified)
```
Client → ClientHello (supported ciphers + key share)
Server → ServerHello (chosen cipher + key share + certificate + finished)
Client → Finished
         ↓
      Encrypted data flows
```
Only **1 RTT** (vs 2 RTT in TLS 1.2). Resumed sessions can do **0-RTT**.

### Certificate Chain
```
Root CA (trusted by OS/browser)
  └── Intermediate CA
        └── Server Certificate (your domain)
```
The browser validates the chain up to a trusted root CA.

---

## CORS (Cross-Origin Resource Sharing)

When a browser makes a request to a different origin (protocol + domain + port), CORS rules apply.

### Simple Requests (no preflight)
- Methods: GET, HEAD, POST
- Only simple headers (Accept, Content-Type with limited values, etc.)

### Preflight Request
For "non-simple" requests, the browser sends an OPTIONS request first:
```
OPTIONS /api/data HTTP/1.1
Origin: https://app.example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: Content-Type, Authorization
```

Server responds:
```
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, PUT, POST, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400
```

### Interview Tip 💡
> CORS is enforced by the **browser**, not the server. The server just sets the headers. A curl request or server-to-server call ignores CORS entirely. This is why CORS is not a security mechanism for APIs — it's a browser security feature.

---

## Cookies and Session Management

HTTP is stateless, so cookies provide state persistence.

### Cookie Attributes
| Attribute | Purpose |
|-----------|---------|
| `Domain` | Which domains receive the cookie |
| `Path` | URL path scope |
| `Expires` / `Max-Age` | When the cookie expires (session cookie if omitted) |
| `Secure` | Only sent over HTTPS |
| `HttpOnly` | Not accessible via JavaScript (prevents XSS theft) |
| `SameSite` | Controls cross-site sending: `Strict`, `Lax`, `None` |

### SameSite Explained
- `Strict` — never sent on cross-site requests (breaks "Login with Google" flows)
- `Lax` (default in modern browsers) — sent on top-level navigations (clicking a link) but NOT on cross-site POST/AJAX
- `None` — always sent (requires `Secure` flag) — needed for third-party cookies

---

## HTTP in System Design — When and How to Optimize

### When to Use HTTP
✅ Request-response APIs (REST, GraphQL)
✅ Static content delivery (with CDN)
✅ Microservice communication (with HTTP/2 for efficiency)
✅ Webhooks

### When NOT to Use HTTP
❌ Real-time bidirectional communication → use **WebSockets** or **SSE**
❌ High-throughput internal service communication → use **gRPC** (HTTP/2 + Protobuf)
❌ Streaming large data between services → use **message queues** (Kafka, SQS)
❌ Low-latency gaming/trading → use raw **TCP/UDP**

### HTTP Optimization Strategies

**1. Reduce Round Trips**
- Use HTTP/2 or HTTP/3 (multiplexing, fewer connections)
- Bundle API calls (GraphQL, batch endpoints)
- Use `103 Early Hints` to preload critical resources
- Connection prewarming (`preconnect`, `dns-prefetch`)

**2. Reduce Payload Size**
- Enable compression (`Content-Encoding: gzip` or `br` — Brotli is ~20% better)
- Use efficient serialization (Protobuf, MessagePack instead of JSON for internal services)
- Paginate large responses
- Use sparse fieldsets / field selection (GraphQL, `?fields=id,name`)

**3. Caching**
- Set proper `Cache-Control` headers
- Use CDN for static and semi-static content
- Implement `ETag` / `Last-Modified` for conditional requests
- Use `stale-while-revalidate` for better perceived performance

**4. Connection Optimization**
- HTTP/2 eliminates need for domain sharding
- Use connection pooling in backend services
- Keep-alive with appropriate timeouts
- Consider HTTP/3 for high-latency or lossy networks (mobile)

**5. API Design**
- Use proper status codes (clients can make smart retry decisions)
- Implement idempotency keys for POST requests
- Use `ETag` for optimistic concurrency control
- Compress request bodies for large uploads

---

## HTTP vs gRPC vs WebSocket vs SSE

| Feature | HTTP/REST | gRPC | WebSocket | SSE |
|---------|-----------|------|-----------|-----|
| Protocol | HTTP/1.1 or HTTP/2 | HTTP/2 | TCP (upgraded from HTTP) | HTTP/1.1+ |
| Data Format | JSON/XML (text) | Protobuf (binary) | Any | Text (event stream) |
| Direction | Request-Response | Unary, Streaming (both) | Full Duplex | Server → Client only |
| Browser Support | ✅ Native | ❌ Needs grpc-web proxy | ✅ Native | ✅ Native |
| Use Case | Public APIs, CRUD | Internal microservices | Chat, gaming, live collab | Notifications, live feeds |
| Overhead | Medium | Low | Low (after handshake) | Low |
| Connection | Short-lived or keep-alive | Persistent (HTTP/2) | Persistent | Persistent |

---

## Common Interview Questions

### Q: What happens when you type a URL in the browser?
1. DNS resolution (browser cache → OS cache → resolver → root → TLD → authoritative)
2. TCP handshake (SYN → SYN-ACK → ACK)
3. TLS handshake (if HTTPS)
4. HTTP request sent
5. Server processes request
6. HTTP response received
7. Browser parses HTML, discovers sub-resources
8. Additional requests for CSS, JS, images (HTTP/2 multiplexes these)
9. DOM construction → CSSOM → Render tree → Paint

### Q: How would you design an API rate limiter?
- Use `429 Too Many Requests` status code
- Include `Retry-After` header
- Implement with token bucket or sliding window algorithm
- Track by API key, IP, or user ID
- Return `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` headers

### Q: How do you handle large file uploads over HTTP?
- Use `multipart/form-data` for the upload
- Support resumable uploads with `Content-Range` header
- Return `206 Partial Content` for range requests
- Implement chunked upload protocol (like tus.io)
- Set appropriate timeouts and max body size limits

### Q: HTTP/2 vs HTTP/3 — when would you choose each?
- **HTTP/2**: When you need broad compatibility, internal services behind a load balancer, or when packet loss is minimal
- **HTTP/3**: Mobile-first applications, high-latency networks, users on unreliable connections, when you need connection migration (WiFi ↔ cellular)

---

## Version Comparison Summary

| Feature | HTTP/1.0 | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|----------|----------|--------|--------|
| Year | 1996 | 1997 | 2015 | 2022 |
| Transport | TCP | TCP | TCP | QUIC (UDP) |
| Connections | One per request | Persistent (keep-alive) | Single multiplexed | Single multiplexed |
| HOL Blocking | N/A | ✅ (application layer) | ✅ (TCP layer) | ❌ Solved |
| Header Format | Text | Text | Binary (HPACK) | Binary (QPACK) |
| Server Push | ❌ | ❌ | ✅ | ✅ |
| Encryption | Optional | Optional | Practically required | Mandatory (TLS 1.3) |
| Header Compression | ❌ | ❌ | HPACK | QPACK |

---

## Further Reading
- websockets — for real-time bidirectional communication
- grpc — for efficient internal service communication
- dns — how domain names resolve before HTTP even starts
- tcp — the transport layer HTTP/1.1 and HTTP/2 rely on
- tls — encryption layer that makes HTTPS possible

---

*Last updated: 2025-03-13*
