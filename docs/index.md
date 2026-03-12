---
title: Home
layout: default
nav_order: 1
---

# Networking Essentials

Deep-dive notes on protocols and API design for system design interviews and real-world engineering.
{: .fs-6 .fw-300 }

---

## Protocols

| Topic | What's Covered |
|:------|:---------------|
| [HTTP]({% link protocols/http.md %}) | HTTP/1.1 through HTTP/3, caching, CORS, TLS, connection lifecycle |
| [gRPC]({% link protocols/grpc.md %}) | Protocol Buffers, 4 communication patterns, streaming, deadlines |
| [WebSockets]({% link protocols/websockets.md %}) | Full-duplex communication, scaling stateful connections, pub/sub |
| [WebRTC]({% link protocols/webrtc.md %}) | Peer-to-peer media, ICE/STUN/TURN, SFU vs MCU, DataChannel |

## API Design

| Topic | What's Covered |
|:------|:---------------|
| [REST]({% link api-design/rest.md %}) | Resource modeling, HTTP semantics, idempotency, response design |
| [GraphQL]({% link api-design/graphql.md %}) | Schema design, resolvers, N+1 problem, DataLoader, pagination |
| [RPC]({% link api-design/rpc.md %}) | RPC paradigm, gRPC vs REST vs GraphQL, production patterns |
| [Common API Patterns]({% link api-design/common-api-patterns.md %}) | Pagination, filtering, versioning, idempotency, webhooks |
| [Auth & Authorization]({% link api-design/authentication-and-authorization.md %}) | JWT, OAuth 2.0, sessions, RBAC, ABAC, mTLS |
| [Rate Limiting]({% link api-design/rate-limiting.md %}) | Token bucket, sliding window, distributed rate limiting |
| [API Security]({% link api-design/api-security.md %}) | OWASP API Top 10, injection attacks, CORS, security headers |
