---
title: "API Security"
layout: default
parent: API Design
nav_order: 7
---

# API Security Considerations — Deep Dive

> A comprehensive guide to securing APIs — common attacks, defense strategies, OWASP API risks, HTTPS, input validation, and production security patterns. Written for system design interviews and building production-grade systems.

---

## Why API Security Matters

APIs are the #1 attack surface for modern applications. Every public endpoint is a door that attackers will try to open.

```
Your API handles:
  - User credentials (passwords, tokens)
  - Payment information (credit cards, bank accounts)
  - Personal data (emails, addresses, phone numbers)
  - Business logic (pricing, inventory, permissions)

One vulnerability = data breach, financial loss, regulatory fines
```

---

## OWASP API Security Top 10 (2023)

### 1. Broken Object Level Authorization (BOLA)
The #1 API vulnerability. User A can access User B's data by changing an ID.

```
# User A is authenticated, has booking 123
GET /bookings/123 → 200 OK ✅ (their booking)

# User A tries User B's booking
GET /bookings/456 → 200 OK 😱 (not their booking, but server returned it!)
```

**Fix:** Always check ownership in your authorization logic:
```python
def get_booking(booking_id, current_user):
    booking = db.bookings.find(booking_id)
    if booking.user_id != current_user.id and current_user.role != 'admin':
        raise ForbiddenError()  # 403
    return booking
```

### 2. Broken Authentication
Weak login flows, missing rate limits on login, token leaks.

**Common mistakes:**
- No rate limiting on `/login` → brute force attacks
- JWT secret is weak or hardcoded → token forgery
- Tokens in URL query params → leaked in logs and browser history
- No token expiry → stolen token works forever

**Fix:**
- Rate limit login attempts (5 per minute per IP)
- Use strong JWT secrets (RS256 with key rotation)
- Tokens in `Authorization` header, never in URLs
- Short-lived access tokens (15 min) + refresh tokens

### 3. Broken Object Property Level Authorization
API returns fields the user shouldn't see.

```json
// User requests their profile
GET /users/me

// Response includes internal fields 😱
{
  "id": "123",
  "name": "John",
  "email": "john@example.com",
  "password_hash": "$2b$10$...",     // ❌ NEVER expose
  "internal_credit_score": 750,       // ❌ Internal data
  "is_flagged_for_fraud": false       // ❌ Internal data
}
```

**Fix:** Explicit response serialization — whitelist fields per role:
```python
def serialize_user(user, requester_role):
    response = { "id": user.id, "name": user.name }
    if requester_role == 'admin':
        response["email"] = user.email
        response["internal_credit_score"] = user.credit_score
    return response
```

### 4. Unrestricted Resource Consumption
No limits on request size, query complexity, or rate.

```
# Attacker sends massive payload
POST /upload
Content-Length: 10737418240  # 10 GB file 😱

# Attacker sends expensive query
GET /events?limit=1000000    # Returns millions of records

# GraphQL complexity attack
query { users { friends { friends { friends { ... } } } } }
```

**Fix:**
- Max request body size (e.g., 10 MB)
- Max pagination limit (e.g., 100 per page)
- GraphQL query depth and complexity limits
- Rate limiting (see rate-limiting)

### 5. Broken Function Level Authorization
User accesses admin endpoints.

```
# Regular user discovers admin endpoint
DELETE /admin/users/456 → 200 OK 😱 (should be 403)
POST /admin/events/bulk-delete → 200 OK 😱
```

**Fix:** Role-based middleware on every endpoint:
```python
@app.route('/admin/users/<id>', methods=['DELETE'])
@require_auth
@require_role('admin')  # Explicit role check
def delete_user(id):
    ...
```

### 6. Server-Side Request Forgery (SSRF)
API fetches a URL provided by the user, attacker points it to internal services.

```
# Legitimate use: fetch user's avatar from URL
POST /profile/avatar
{ "url": "https://example.com/photo.jpg" }

# Attack: fetch internal service data
POST /profile/avatar
{ "url": "http://169.254.169.254/latest/meta-data/" }  # AWS metadata 😱
{ "url": "http://internal-db:5432/" }                    # Internal DB 😱
```

**Fix:**
- Whitelist allowed domains
- Block private IP ranges (10.x, 172.16.x, 192.168.x, 169.254.x)
- Use a proxy/sandbox for external URL fetching
- Don't return raw responses from fetched URLs

---

## Transport Security

### Always HTTPS
```
❌ http://api.example.com/users    → Data readable by anyone on the network
✅ https://api.example.com/users   → Encrypted in transit
```

**Enforce with:**
- HSTS header: `Strict-Transport-Security: max-age=31536000; includeSubDomains`
- Redirect HTTP → HTTPS at load balancer level
- TLS 1.2 minimum (prefer TLS 1.3)

### Certificate Pinning (Mobile Apps)
```
Mobile app hardcodes the expected server certificate fingerprint.
If a proxy (MITM) presents a different cert → connection refused.
```
Prevents man-in-the-middle attacks even if attacker has a trusted CA cert.

---

## Input Validation

### Validate Everything
```python
# ❌ Trust user input
name = request.body["name"]
db.execute(f"SELECT * FROM users WHERE name = '{name}'")
# Attacker sends: name = "'; DROP TABLE users; --"  → SQL Injection 😱

# ✅ Parameterized queries
db.execute("SELECT * FROM users WHERE name = ?", [name])

# ✅ Input validation
if len(name) > 100:
    raise ValidationError("Name too long")
if not re.match(r'^[a-zA-Z\s]+$', name):
    raise ValidationError("Invalid characters")
```

### Common Injection Attacks

| Attack | Vector | Prevention |
|--------|--------|------------|
| SQL Injection | User input in SQL queries | Parameterized queries, ORM |
| NoSQL Injection | User input in MongoDB queries | Input validation, sanitization |
| XSS (Cross-Site Scripting) | User input rendered in HTML | Output encoding, CSP headers |
| Command Injection | User input in shell commands | Never use shell commands with user input |
| Path Traversal | `../../etc/passwd` in file paths | Whitelist allowed paths, sanitize |

### Content-Type Validation
```
# Only accept expected content types
if request.content_type != 'application/json':
    return 415 Unsupported Media Type

# Validate JSON schema
schema = {
  "type": "object",
  "required": ["name", "email"],
  "properties": {
    "name": { "type": "string", "maxLength": 100 },
    "email": { "type": "string", "format": "email" }
  },
  "additionalProperties": false  # Reject unknown fields
}
```

---

## Security Headers

```http
# Prevent XSS
Content-Security-Policy: default-src 'self'; script-src 'self'

# Prevent MIME sniffing
X-Content-Type-Options: nosniff

# Prevent clickjacking
X-Frame-Options: DENY

# Force HTTPS
Strict-Transport-Security: max-age=31536000; includeSubDomains

# Control referrer information
Referrer-Policy: strict-origin-when-cross-origin

# Prevent browser features
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

---

## CORS (Cross-Origin Resource Sharing)

Browsers block requests from different origins by default. CORS headers tell the browser which cross-origin requests are allowed.

```
# Restrictive (recommended)
Access-Control-Allow-Origin: https://myapp.com
Access-Control-Allow-Methods: GET, POST
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 86400

# ❌ NEVER in production
Access-Control-Allow-Origin: *  # Allows any website to call your API
```

**Key points:**
- CORS is enforced by the **browser**, not the server
- Server-to-server calls ignore CORS entirely
- CORS is NOT a security mechanism for your API — it's browser protection
- Preflight (OPTIONS) requests add latency — cache with `Max-Age`

See http for detailed CORS explanation.

---

## Secrets Management

```
❌ Hardcoded secrets
const API_KEY = "sk_live_abc123..."  // In source code 😱
const DB_PASSWORD = "password123"     // In source code 😱

✅ Environment variables
const API_KEY = process.env.API_KEY

✅ Secrets manager (production)
const API_KEY = await secretsManager.getSecret("api-key")
// AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager
```

**Rules:**
- Never commit secrets to git (use `.gitignore`, pre-commit hooks)
- Rotate secrets regularly (automated rotation)
- Use different secrets per environment (dev, staging, prod)
- Audit secret access (who accessed what, when)

---

## API Security Checklist

```
Transport:
  ✅ HTTPS everywhere (TLS 1.2+)
  ✅ HSTS header
  ✅ Certificate pinning for mobile apps

Authentication:
  ✅ Strong password hashing (bcrypt, argon2)
  ✅ Rate limit login attempts
  ✅ Short-lived JWTs + refresh tokens
  ✅ Tokens in Authorization header, not URLs

Authorization:
  ✅ Check object ownership on every request (prevent BOLA)
  ✅ Role-based access control on every endpoint
  ✅ Whitelist response fields per role

Input:
  ✅ Validate and sanitize all input
  ✅ Parameterized queries (prevent SQL injection)
  ✅ Max request body size
  ✅ Max pagination limits

Headers:
  ✅ CORS properly configured (not *)
  ✅ Security headers (CSP, X-Content-Type-Options, etc.)
  ✅ Remove server version headers (X-Powered-By)

Monitoring:
  ✅ Log all authentication events
  ✅ Alert on unusual patterns (brute force, scraping)
  ✅ Rate limiting with 429 responses
```

---

## Interview Tip 💡
> In system design interviews, you don't need to design a full security system. But mentioning 2-3 security considerations shows production awareness. Good ones to mention: "I'd use JWT with short expiry for auth, rate limiting to prevent abuse, and always validate object ownership to prevent BOLA." That's usually enough to impress.

---

## Further Reading
- authentication-and-authorization — deep dive into auth mechanisms
- rate-limiting — protecting APIs from abuse
- http — HTTPS, TLS, CORS, security headers
- rest — securing REST endpoints
- graphql — GraphQL-specific security (query depth, introspection)

---

*Last updated: 2025-03-13*
