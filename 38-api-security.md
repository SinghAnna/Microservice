# Part 38: Microservices Security — API Security (OWASP API Top 10)

> API security ka core — **BOLA/IDOR (#1 risk, object-level authZ) + Broken Authentication + Rate Limiting** — har endpoint pe ownership verify, OAuth2 + short-lived tokens, gateway-level centralized enforcement. [2020][2021][2025][2030]

---

## 1. API Security Mental Model

```text
API = External attack surface
Attack = BOLA / Auth bypass / DoS

Defense:
1. AuthN (OAuth2 + JWT)
2. AuthZ (object-level)
3. Rate limit (gateway)
4. Validation (schema)
5. WAF (edge)
```

### OWASP API Top 10 (2023)

| # | Risk | Type |
|---|---|---|
| API1 | BOLA | AuthZ |
| API2 | Broken AuthN | AuthN |
| API3 | Broken Object Property AuthZ | AuthZ |
| API4 | Unrestricted Resource Consumption | DoS |
| API5 | Broken Function Level AuthZ | AuthZ |
| API6 | Unrestricted Sensitive Business Flows | Business |
| API7 | SSRF | Server |
| API8 | Security Misconfiguration | Config |
| API9 | Improper Inventory | Inventory |
| API10 | Unsafe Consumption of APIs | Third-party |

OWASP API Top 10 2023 — BOLA #1, 3/5 top = authorization. [2021][2023][2025]

---

## 2. BOLA / IDOR (API1)

```text
BOLA = Broken Object Level Authorization
IDOR = Insecure Direct Object Reference
#1 API vulnerability
```

### Vulnerable Example

```java
// WRONG: ownership check nahi
@GetMapping("/api/orders/{id}")
public Order getOrder(@PathVariable Long id) {
    return orderRepository.findById(id).orElseThrow();
}
```

### Exploit

```text
User A: GET /api/orders/12345 -> 200 (own order)
User B: GET /api/orders/12345 -> 200 (WRONG!)
```

BOLA — user-supplied ID, ownership verify nahi, enumeration possible. [2028][2029][2030]

### Fix (Object Ownership)

```java
// RIGHT: ownership check
@GetMapping("/api/orders/{id}")
public Order getOrder(@PathVariable Long id, @AuthenticationPrincipal User user) {
    return orderRepository.findByIdAndUserId(id, user.getId())
        .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));
}
```

Fix — `WHERE id = ? AND user_id = ?` (data-layer ownership). [2029][2030][2031]

### 404 vs 403

```text
403 = "Exist karta hai, access nahi" (enumeration hint)
404 = "Exist nahi karta" (enumeration block)
```

404 return — enumeration prevent. [2029][2031]

### Indirect References

```text
Sequential ID: /api/orders/12345 (guessable)
UUID: /api/orders/550e8400-e29b-41d4 (harder)
```

UUID / indirect reference — guessing harder. [2029][2031]

---

## 3. Broken Authentication (API2)

```text
AuthN flaws:
- Weak tokens
- Missing rotation
- alg: none
- No revocation
- No PKCE
```

### Fix

```text
- OAuth2 + PKCE (Authorization Code flow)
- RS256 (asymmetric, not HS256)
- Short-lived access token (15 min)
- Refresh token rotation
- Token revocation (logout)
- MFA
```

OAuth2 + PKCE, RS256, short-lived tokens, rotation, revocation. [2021][2029][2031]

### JWT Validation

```java
// Validate: signature + exp + iss + aud + scope
Jws<Claims> jws = Jwts.parserBuilder()
    .setSigningKey(publicKey)
    .requireIssuer("[https://auth.example.com](https://auth.example.com)")
    .requireAudience("order-service")
    .build()
    .parseClaimsJws(token);
```

JWT — signature + exp + iss + aud + scope validate. [2029][2031][2034]

---

## 4. Rate Limiting (API4)

```text
Unrestricted Resource Consumption = DoS
```

### Rate Limit Strategy

| Endpoint | Limit |
|---|---|
| Login | 5/min/IP |
| Password reset | 3/hour |
| General API | 120/min/user |
| Expensive query | 10/min |

Tiered limits — auth strict, general moderate, expensive strict. [2028][2030][2032]

### Redis Sliding Window

```text
Redis (shared) -> Gateway instances agree
Per-IP + per-user + per-endpoint
```

Redis-backed — distributed rate limit (multi-instance). [2029][2031][2034]

### Response Headers

```text
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 14
Retry-After: 42
429 Too Many Requests
```

Standard headers — limit, remaining, retry-after. [2029][2031]

### API Gateway Rate Limit

```yaml
# Kong / AWS API Gateway
rate_limit:
  requests_per_second: 100
  burst: 200
  key: user_id
```

Gateway-level — centralized enforcement. [2026][2027][2030]

---

## 5. API Gateway Security

```text
Gateway = Single enforcement point
```

### Gateway Controls

```text
1. TLS termination
2. WAF inspection
3. AuthN (JWT/OAuth2)
4. AuthZ (scope/role)
5. Rate limiting
6. Schema validation
7. Logging
8. DDoS protection
```

Gateway — TLS, WAF, authN, authZ, rate limit, validation, logging, DDoS. [2026][2027][2031]

### Gateway Flow

```text
Client -> Gateway
   |
   +--> WAF (inspect)
   +--> AuthN (JWT validate)
   +--> AuthZ (scope check)
   +--> Rate limit
   +--> Schema validate
   |
   v
Backend service
```

Gateway — ordered controls (WAF → authN → authZ → rate → validate). [2026][2027][2031]

---

## 6. Complete Flow

```text
1. Client -> Gateway
2. WAF inspect
3. JWT validate (signature + exp)
4. Scope/role check
5. Rate limit (Redis)
6. Schema validate
7. Backend -> Object ownership check
8. Response (404 if not owner)
```

---

## 7. Real-World Patterns

| Pattern | Use Case |
|---|---|
| BOLA guard | Every object endpoint |
| OAuth2 + PKCE | SPA / mobile |
| Redis rate limit | Multi-instance |
| API Gateway | Centralized enforcement |
| WAF | Edge protection |

### Decision Framework

```text
Object endpoints? -> BOLA guard (mandatory)
SPA/mobile? -> OAuth2 + PKCE
Multi-instance? -> Redis rate limit
Centralized? -> API Gateway
Edge? -> WAF + CDN
```

---

## 8. Common Gotchas

### Gotcha 1 — AuthN ≠ AuthZ

```text
Problem:
Authenticated = can access anything

Fix:
Object-level authorization
```

### Gotcha 2 — 403 Instead of 404

```text
Problem:
Enumeration hint

Fix:
404 (not found)
```

### Gotcha 3 — Per-Instance Rate Limit

```text
Problem:
Multi-instance bypass

Fix:
Redis (shared state)
```

### Gotcha 4 — Tokens in Query Params

```text
Problem:
Log leak

Fix:
Authorization header only
```

---

## 9. Interview Questions

| Question | Answer |
|---|---|
| BOLA kya hai? | Object-level authorization missing. |
| BOLA vs IDOR? | Same (different name). |
| OAuth2 + PKCE? | SPA/mobile auth standard. |
| Rate limit kyun? | DoS + brute force prevent. |
| Gateway kya karta? | Centralized enforcement. |
| 404 vs 403? | Enumeration prevent vs hint. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Object endpoint | BOLA guard |
| SPA auth | OAuth2 + PKCE |
| Multi-instance | Redis rate limit |
| DoS | Gateway rate limit + WAF |
| Enumeration | UUID + 404 |

---

## 10. Key Takeaways

```text
1. BOLA = #1 API risk.

2. Object ownership = mandatory.

3. 404 = enumeration block.

4. OAuth2 + PKCE = standard.

5. Short-lived tokens = 15 min.

6. Redis = distributed rate limit.

7. Gateway = centralized enforcement.

8. WAF = edge protection.

9. Schema validation = input trust nahi.

10. AuthN ≠ AuthZ.
```

---

## 11. Next Part Preview

**Part 39: Security in CI/CD (DevSecOps)**

Isme hum cover karenge:

```text
- DevSecOps deep dive
- Shift-left security
- Security gates in CI/CD
- Automated security testing
- Security champions
```