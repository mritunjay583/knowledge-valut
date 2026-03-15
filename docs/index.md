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
| [HTTP](protocols/http.md) | HTTP/1.1 through HTTP/3, caching, CORS, TLS, connection lifecycle |
| [gRPC](protocols/grpc.md) | Protocol Buffers, 4 communication patterns, streaming, deadlines |
| [WebSockets](protocols/websockets.md) | Full-duplex communication, scaling stateful connections, pub/sub |
| [WebRTC](protocols/webrtc.md) | Peer-to-peer media, ICE/STUN/TURN, SFU vs MCU, DataChannel |

## API Design

| Topic | What's Covered |
|:------|:---------------|
| [REST](api-design/rest.md) | Resource modeling, HTTP semantics, idempotency, response design |
| [GraphQL](api-design/graphql.md) | Schema design, resolvers, N+1 problem, DataLoader, pagination |
| [RPC](api-design/rpc.md) | RPC paradigm, gRPC vs REST vs GraphQL, production patterns |
| [Common API Patterns](api-design/common-api-patterns.md) | Pagination, filtering, versioning, idempotency, webhooks |
| [Auth & Authorization](api-design/authentication-and-authorization.md) | JWT, OAuth 2.0, sessions, RBAC, ABAC, mTLS |
| [Rate Limiting](api-design/rate-limiting.md) | Token bucket, sliding window, distributed rate limiting |
| [API Security](api-design/api-security.md) | OWASP API Top 10, injection attacks, CORS, security headers |

## Data Modeling

| Topic | What's Covered |
|:------|:---------------|
| [SQL Databases](data-modeling/sql-databases.md) | Postgres internals, storage engine, indexing, query execution, MVCC |
| [ACID Properties](data-modeling/acid-properties.md) | Atomicity, Consistency, Isolation, Durability — how Postgres implements each |
| [SQL vs NoSQL](data-modeling/sql-vs-nosql.md) | Document, key-value, wide-column, graph — when to use which and why |

## Important Concepts

| Topic | What's Covered |
|:------|:---------------|
| [Bloom Filters](important-concepts/bloom-filters.md) | How they work, math, tuning, LSM trees, Cassandra/Bigtable usage, Cuckoo filters |
