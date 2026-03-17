---
title: "Communication Stack"
layout: default
parent: Important Concepts
nav_order: 2
---

# Communication Stack — API Styles, Protocols, and Frameworks

A clean mental model for how REST, GraphQL, gRPC, HTTP, and WebSocket relate to each other.

---

## The 3-Layer Model

```
API Style / Interaction Model
         ↓
Transport Protocol
         ↓
Data Format
```

Each layer has a different job. Most confusion comes from mixing them up.

---

## Layer 1 — Transport Protocols

These define **how data travels** across the network.

| Protocol | Key Property |
|---|---|
| HTTP | Request-response, stateless |
| HTTP/2 | Multiplexed streams, binary framing |
| WebSocket | Full-duplex, persistent connection |
| TCP | Reliable byte stream (lower level) |
| FTP | File transfer |

WebSocket belongs here. It is a network transport protocol, just like HTTP.

---

## Layer 2 — API Communication Styles

These define **how applications structure requests** — the interaction pattern, not the network transport.

| Style | How It Works |
|---|---|
| REST | Resource-based, HTTP verbs (GET, POST, PUT, DELETE) |
| GraphQL | Query language, client asks for exactly what it needs |
| RPC | Call remote functions as if they were local |

---

## Layer 3 — RPC Frameworks

Frameworks that implement the RPC communication style. They choose their transport protocol internally.

| Framework | Transport |
|---|---|
| gRPC | HTTP/2 |
| Apache Thrift | TCP / HTTP |
| Dubbo | TCP |

---

## Where WebSocket Fits

WebSocket is a transport protocol designed for **persistent connections**.

Unlike HTTP:

```
HTTP      → request-response (client asks, server replies, done)
WebSocket → full-duplex persistent connection (both sides talk freely)
```

Once connected:

```
Client ↔ Server
Client ↔ Server
Client ↔ Server
No new request each time.
```

---

## Visual Categorization

```
API Communication Styles
    ├── REST
    ├── GraphQL
    └── RPC
          └── gRPC (framework)

Transport Protocols
    ├── HTTP
    ├── HTTP/2
    ├── WebSocket
    ├── FTP
    └── TCP
```

---

## How They Combine in Real Systems

### REST API

```
REST → HTTP → JSON
```

### GraphQL API

```
GraphQL → HTTP → JSON
(sometimes WebSocket for subscriptions)
```

### gRPC Microservices

```
gRPC → HTTP/2 → Protocol Buffers
```

### Real-Time Systems (chat, gaming, live data)

```
Application Protocol → WebSocket → TCP
```

Examples: WhatsApp chat, stock market updates, multiplayer games, live dashboards.

---

## Special Case: GraphQL + WebSocket

GraphQL subscriptions often use WebSocket for server-pushed updates.

```
Client
  ↓
GraphQL subscription
  ↓
WebSocket
  ↓
Server pushes updates
```

Used for: live notifications, live chat, stock updates.

---

## Interview Answer

> "Where does WebSocket fit compared to REST and gRPC?"

WebSocket is a **transport protocol** that enables persistent bidirectional communication between client and server. REST and GraphQL are **API styles** usually built on HTTP, while gRPC is an **RPC framework** built on HTTP/2.

The key distinction: REST/GraphQL/RPC describe *how you structure interactions*. HTTP/WebSocket describe *how data travels over the network*. They live on different layers.
