# Authentication & Authorization — Deep Dive

> A comprehensive guide to authn vs authz, JWT, OAuth 2.0, API keys, session management, RBAC, and production patterns. Written for system design interviews and building secure, scalable systems.

---

## Authentication vs Authorization

These are two different questions:

| | Authentication (AuthN) | Authorization (AuthZ) |
|--|------------------------|----------------------|
| Question | **Who are you?** | **What can you do?** |
| Verifies | Identity | Permissions |
| Example | Login with email/password | Can this user delete this post? |
| Happens | First | After authentication |
| Failure | 401 Unauthorized | 403 Forbidden |

```
Request: DELETE /bookings/456
  
1. Authentication: Is this a valid user? (check JWT/session)
   ├── No → 401 Unauthorized
   └── Yes → User is john@example.com

2. Authorization: Can john delete booking 456?
   ├── No (not his booking) → 403 Forbidden
   └── Yes (his booking) → 204 No Content ✅
```

### Interview Tip 💡
> 401 means "unauthenticated" (despite the misleading name "Unauthorized"). 403 means "authenticated but not authorized." Getting this distinction right in interviews shows you understand security fundamentals.

---

## Authentication Methods

### 1. Session-Based Authentication (Traditional)

```
1. Client → POST /login { email, password }
2. Server → Validates credentials
           → Creates session in DB/Redis: { session_id: "abc", user_id: "123" }
           → Returns Set-Cookie: session_id=abc; HttpOnly; Secure

3. Client → GET /bookings (Cookie: session_id=abc)
4. Server → Looks up session_id "abc" in DB/Redis
           → Finds user_id "123"
           → Returns bookings for user 123
```

**Pros:**
- Simple to implement
- Easy to revoke (delete session from DB)
- Server has full control

**Cons:**
- **Stateful** — server must store sessions (DB/Redis lookup on every request)
- **Scaling pain** — sticky sessions or shared session store needed
- **Not great for microservices** — every service needs access to session store
- **CSRF vulnerable** — cookies sent automatically on cross-site requests

### 2. JWT (JSON Web Tokens) — Stateless Authentication

```
1. Client → POST /login { email, password }
2. Server → Validates credentials
           → Creates JWT: header.payload.signature
           → Returns { "token": "eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoiMTIzIn0.signature" }

3. Client → GET /bookings
             Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
4. Server → Verifies JWT signature (NO database lookup)
           → Reads user_id from payload
           → Returns bookings
```

**JWT Structure:**
```
Header (algorithm + type):
{
  "alg": "RS256",
  "typ": "JWT"
}

Payload (claims):
{
  "sub": "user_123",           // Subject (user ID)
  "email": "john@example.com",
  "role": "customer",
  "iat": 1710000000,           // Issued at
  "exp": 1710003600,           // Expires (1 hour)
  "iss": "auth.myapp.com"      // Issuer
}

Signature:
  RS256(base64(header) + "." + base64(payload), private_key)
```

**Pros:**
- **Stateless** — no DB lookup needed, any service can verify
- **Scales perfectly** — no shared session store
- **Great for microservices** — each service verifies independently with the public key
- **Cross-domain** — works across different services/domains
- **Carries context** — user info embedded in the token

**Cons:**
- **Can't revoke easily** — token is valid until it expires (no server-side invalidation)
- **Size** — JWTs are larger than session IDs (~800 bytes vs ~32 bytes)
- **Secret management** — signing key must be protected
- **Payload is NOT encrypted** — just base64 encoded (anyone can read it)

### JWT Revocation Strategies
Since JWTs can't be "deleted" like sessions, you need workarounds:

```
1. Short expiry (15 min) + Refresh tokens
   - Access token: 15 min TTL (used for API calls)
   - Refresh token: 7 days TTL (stored in DB, used to get new access tokens)
   - To "revoke": delete the refresh token from DB
   - Worst case: compromised access token works for 15 min

2. Token blacklist
   - Store revoked token IDs in Redis
   - Check blacklist on every request (adds a lookup, partially defeats stateless benefit)
   - Only needed for critical revocations (password change, account compromise)

3. Token versioning
   - Store a "token_version" per user in DB
   - Include version in JWT payload
   - Increment version to invalidate all existing tokens
```

### 3. API Keys

```
GET /events
Authorization: Bearer sk_live_abc123def456...
```

- Long-lived, randomly generated strings
- Stored in DB with associated permissions
- **Use for:** Server-to-server communication, third-party developer access
- **Don't use for:** User-facing authentication (no expiry, no user context)

**API Key vs JWT:**
| | API Key | JWT |
|--|---------|-----|
| Contains user info | ❌ (just an opaque string) | ✅ (payload has claims) |
| Stateless verification | ❌ (DB lookup required) | ✅ (signature check only) |
| Revocation | ✅ Easy (delete from DB) | ❌ Hard (wait for expiry) |
| Best for | Service-to-service, developer APIs | User sessions, microservices |

### 4. OAuth 2.0 — Delegated Authorization

OAuth lets users grant third-party apps limited access to their data without sharing passwords.

```
Example: "Login with Google" or "Connect your Spotify"

1. User clicks "Login with Google" on your app
2. Your app redirects to Google's auth page
3. User logs in to Google, grants permission
4. Google redirects back to your app with an authorization code
5. Your server exchanges the code for an access token (server-to-server)
6. Your server uses the access token to fetch user info from Google
7. Your server creates a session/JWT for the user
```

**OAuth 2.0 Flows:**

| Flow | Use Case | Security |
|------|----------|----------|
| Authorization Code | Web apps with backend | Most secure (code exchanged server-side) |
| Authorization Code + PKCE | Mobile/SPA apps | Secure (no client secret needed) |
| Client Credentials | Service-to-service | Machine-to-machine, no user involved |
| Device Code | Smart TVs, CLI tools | User authorizes on another device |

**Authorization Code Flow (most common):**
```
┌────────┐     1. Redirect to Google      ┌────────────┐
│  User  │ ──────────────────────────────→ │  Google    │
│Browser │                                 │  Auth Page │
│        │ ←── 2. User logs in + consents  │            │
│        │                                 └─────┬──────┘
│        │ ←── 3. Redirect with auth code ───────┘
│        │
│        │ ──── 4. Send code to your server ────→ ┌────────────┐
│        │                                        │ Your Server│
│        │                                        │            │
│        │      5. Server exchanges code          │ ──→ Google │
│        │         for access token               │ ←── token  │
│        │                                        │            │
│        │ ←── 6. Server creates JWT/session ──── │            │
└────────┘                                        └────────────┘
```

### Interview Tip 💡
> OAuth 2.0 is for **authorization** (granting access), not authentication. OpenID Connect (OIDC) is a layer on top of OAuth that adds authentication (proving identity). When someone says "Login with Google," they're using OIDC, not raw OAuth. But in interviews, saying "OAuth" is usually fine.

---

## Authorization Models

### 1. Role-Based Access Control (RBAC)

Assign roles to users, permissions to roles.

```
Roles:
  customer     → [read:events, create:bookings, read:own_bookings]
  venue_manager → [create:events, read:events, read:venue_sales, update:own_events]
  admin        → [*] (everything)

Users:
  john@example.com    → customer
  manager@venue.com   → venue_manager
  admin@company.com   → admin
```

**Implementation:**
```javascript
// Middleware checks role + permission
function authorize(requiredPermission) {
  return (req, res, next) => {
    const userRole = req.user.role;  // from JWT
    const permissions = rolePermissions[userRole];
    
    if (!permissions.includes(requiredPermission) && !permissions.includes('*')) {
      return res.status(403).json({ error: "Forbidden" });
    }
    next();
  };
}

// Usage
app.delete('/bookings/:id', authenticate, authorize('delete:bookings'), handler);
```

**Pros:** Simple, widely understood, easy to audit
**Cons:** Role explosion (too many roles), coarse-grained

### 2. Attribute-Based Access Control (ABAC)

Decisions based on attributes of the user, resource, and environment.

```
Policy: "Users can cancel bookings they own, within 24 hours of creation"

Attributes checked:
  - user.id == booking.user_id          (ownership)
  - booking.created_at > now() - 24h    (time constraint)
  - booking.status != "completed"       (state constraint)
```

**Pros:** Fine-grained, flexible, context-aware
**Cons:** Complex to implement and audit, harder to reason about

### 3. Relationship-Based Access Control (ReBAC)

Authorization based on relationships between entities. Used by Google Zanzibar.

```
# Define relationships
document:readme#owner@user:alice       # Alice owns readme
document:readme#viewer@group:eng       # Engineering group can view readme
group:eng#member@user:bob              # Bob is in engineering group

# Check: Can Bob view readme?
# Bob → member of eng → eng can view readme → YES ✅
```

**Used by:** Google (Zanzibar), Airbnb, Carta
**Pros:** Models real-world relationships naturally, scales to complex hierarchies
**Cons:** Complex infrastructure (graph-based authorization engine)

### Comparison

| Model | Granularity | Complexity | Best For |
|-------|-------------|------------|----------|
| RBAC | Coarse | Low | Most applications, admin panels |
| ABAC | Fine | Medium | Context-dependent policies |
| ReBAC | Fine | High | Document sharing, social networks, org hierarchies |

### Interview Tip 💡
> Default to RBAC in interviews — it covers 90% of use cases. Mention ABAC if the problem involves complex policies (e.g., "users can only edit documents in their department during business hours"). Mention Zanzibar/ReBAC if the problem involves sharing and permissions (e.g., "design Google Docs permissions").

---

## Mutual TLS (mTLS) — Service-to-Service Auth

In microservices, services need to authenticate each other (not just users).

```
Normal TLS:
  Client verifies server's certificate (one-way)

Mutual TLS:
  Client verifies server's certificate
  Server verifies client's certificate (both ways)
```

```
┌──────────┐                    ┌──────────┐
│ Service A│ ── presents cert → │ Service B│
│          │ ← presents cert ── │          │
│          │                    │          │
│ Verifies │                    │ Verifies │
│ B's cert │                    │ A's cert │
└──────────┘                    └──────────┘
    Both sides trust each other ✅
```

**Used by:** Kubernetes service mesh (Istio, Linkerd), AWS internal services
**Why:** No tokens to manage, no secrets to rotate (certificates handle everything)

---

## Auth Architecture at Scale

### Centralized Auth Service Pattern
```
┌────────────┐     JWT          ┌──────────────┐
│ Client     │ ──────────────→  │ API Gateway  │
│            │                  │ (validates   │
│            │                  │  JWT)        │
│            │                  └──────┬───────┘
│            │                         │ Forward request + user context
│            │                  ┌──────┼──────────┐
│            │                  ↓      ↓          ↓
│            │            ┌────────┐ ┌────────┐ ┌────────┐
│            │            │User Svc│ │Order   │ │Payment │
│            │            │        │ │Service │ │Service │
│            │            └────────┘ └────────┘ └────────┘
│            │
│            │  Login flow:
│            │ ──── POST /auth/login ────→ ┌────────────┐
│            │ ←─── JWT (access + refresh)  │ Auth       │
│            │                              │ Service    │
│            │ ──── POST /auth/refresh ──→  │ (issues    │
│            │ ←─── New JWT                 │  JWTs)     │
└────────────┘                              └────────────┘
```

**Flow:**
1. Client authenticates with Auth Service → gets JWT
2. Client sends JWT with every request to API Gateway
3. API Gateway validates JWT signature (no DB call)
4. API Gateway forwards request + user context to internal services
5. Internal services trust the gateway (mTLS between services)

**This is the pattern used by most large-scale systems (Google, Netflix, Uber).**

---

## Common Interview Questions

### Q: How would you design an auth system for a microservices architecture?
1. **Auth Service** issues JWTs (short-lived access + long-lived refresh tokens)
2. **API Gateway** validates JWTs on every request (stateless, no DB)
3. **Internal services** communicate via mTLS (no user tokens needed)
4. **Refresh flow:** Client uses refresh token to get new access token when expired
5. **Revocation:** Short access token TTL (15 min) + revocable refresh tokens in Redis

### Q: JWT vs Session — when would you pick each?
- **JWT:** Microservices, distributed systems, mobile apps, cross-domain auth
- **Session:** Simple monolith, when you need instant revocation, when token size matters

### Q: How do you handle "Login with Google"?
- OAuth 2.0 Authorization Code flow (with PKCE for mobile/SPA)
- Exchange auth code for Google access token (server-side)
- Fetch user profile from Google
- Create or link user in your DB
- Issue your own JWT for subsequent requests

### Q: How do you secure service-to-service communication?
- **mTLS** for transport-level authentication
- **API keys** or **service tokens** for application-level auth
- **Service mesh** (Istio/Linkerd) automates mTLS certificate management
- **Network policies** restrict which services can talk to each other

---

## Further Reading
- [[api-security]] — broader security considerations (CORS, injection, etc.)
- [[rate-limiting]] — protecting authenticated endpoints from abuse
- [[rest]] — applying auth to REST APIs
- [[graphql]] — field-level authorization in GraphQL
- [[http]] — HTTP auth headers, cookies, TLS

---

*Last updated: 2025-03-13*
