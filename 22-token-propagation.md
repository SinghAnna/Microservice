# Part 22: Microservices Security — Token Propagation Strategies

> Token propagation ka core — **user identity ko service boundaries ke across carry karna** — 4 patterns: **Token Relay (header)**, **Token Exchange (STS)**, **Signed Assertion (self-signed)**, aur **Async/MQ propagation (message headers)**. [1797][1798][1796][1800]

---

## 1. Token Propagation Mental Model

```text
User -> Gateway -> Service A -> Service B -> Service C
         |           |            |
         +-----------+------------+--> User context propagate
```

### Problem

```text
Service A me user context hai (JWT)
Service B ko bhi chahiye (authorization ke liye)
Kaise carry karein?
```

Token propagation — authenticated user context services ke beech carry. [1797][1798]

---

## 2. Propagation Patterns

| Pattern | Mechanism | Trust | Use Case |
|---|---|---|---|
| Token Relay | Same JWT forward | High (original token) | Simple relay |
| Token Exchange | STS se naya token | Highest (per-service) | Zero trust |
| Signed Assertion | Service khud sign | Medium | Internal trust |
| Header Attributes | Claims as headers | Low (trust required) | Internal only |

### OWASP Recommendations

```text
1. Token Relay — recipient validate kar sake
2. Token Exchange — token restricted to intended resource
3. Signed Assertion — service signs assertion
4. Header Attributes — sirf trusted network
```

OWASP — har pattern ka trust model alag, recipient validate kar sake. [1797]

---

## 3. Pattern 1 — Token Relay (Header)

```text
Service A -> Service B
Authorization: Bearer <original-jwt>
```

### Implementation (Spring)

```java
@Bean
public WebClient productWebClient(
    WebClient.Builder builder,
    TokenRelayFilter filter
) {
    return builder
        .filter(filter)
        .baseUrl("http://product-service")
        .build();
}
```

### Flow

```text
1. Service A -> JWT receive (Authorization header)
2. Service A -> Service B call
3. Same JWT forward (Authorization header)
4. Service B validate
```

Token Relay — original JWT unchanged forward, har service validate karti hai. [1798][1800][1802]

### Pros/Cons

| Pro | Con |
|---|---|
| Simple | Same token everywhere (blast radius) |
| No extra calls | Audience mismatch possible |
| User context intact | Token expiry chain issue |

---

## 4. Pattern 2 — Token Exchange (STS)

```text
Service A -> STS (subject_token + audience)
   |
   v
Naya Token (service-specific)
   |
   v
Service A -> Service B (naya token)
```

### Implementation

```java
public Mono<String> exchangeToken(String userToken, String audience) {
    return webClient.post()
        .uri("[https://auth.example.com/oauth2/token](https://auth.example.com/oauth2/token)")
        .bodyValue(Map.of(
            "grant_type", "urn:ietf:params:oauth:grant-type:token-exchange",
            "subject_token", userToken,
            "audience", audience,
            "scope", "orders:read"
        ))
        .retrieve()
        .bodyToMono(TokenResponse.class)
        .map(TokenResponse::getAccessToken);
}
```

Token Exchange — STS se naya service-specific token, blast radius kam. [1797][1798]

### Pros/Cons

| Pro | Con |
|---|---|
| Per-service token | Extra STS call (latency) |
| Downscoping | Complexity |
| Blast radius kam | STS availability dependency |

---

## 5. Pattern 3 — Signed Assertion

```text
Service A -> Service B
X-Signed-Assertion: <service-signed JWT>
```

### Implementation

```java
public String createAssertion(UserContext user) {
    return Jwts.builder()
        .subject(user.getUserId())
        .claim("roles", user.getRoles())
        .claim("tenant_id", user.getTenantId())
        .issuer("order-service")           // Service A signs
        .audience().add("product-service")
        .expiration(new Date(System.currentTimeMillis() + 60000))
        .signWith(serviceAPrivateKey)
        .compact();
}
```

### Verification (Service B)

```java
Jwts.parser()
    .verifyWith(serviceAPublicKey)
    .requireIssuer("order-service")
    .build()
    .parseSignedClaims(assertion);
```

Signed Assertion — Service A khud sign karta hai, Service B public key se verify. [1797]

### Pros/Cons

| Pro | Con |
|---|---|
| No STS dependency | Trust model complex |
| Fast (no extra call) | Key management |
| Service identity clear | Standard nahi |

---

## 6. Pattern 4 — Header Attributes (Internal Only)

```text
Gateway -> Service
X-User-Id: user-123
X-User-Roles: USER,ADMIN
X-Tenant-Id: tenant-1
```

### Warning

```text
Headers spoof ho sakte hain
Sirf trusted network (mTLS + NetworkPolicy)
Production me JWT bhi validate karo
```

Header attributes — sirf trusted network, JWT bhi validate. [1797][1800]

---

## 7. Async + Message Queues

```text
Service A -> Kafka/RabbitMQ -> Service B
   |
   | User context kaise?
   v
Message headers me token/claims
```

### Spring Integration

```java
@MessagingGateway(asyncExecutor = "securityContextExecutor")
public interface OrderGateway {
    @Gateway(requestChannel = "orderChannel")
    Future<String> send(String payload);
}

@Bean
public AsyncTaskExecutor securityContextExecutor() {
    return new DelegatingSecurityContextAsyncTaskExecutor(
        new SimpleAsyncTaskExecutor()
    );
}
```

### Message Headers

```java
Message<Order> message = MessageBuilder
    .withPayload(order)
    .setHeader("X-User-Id", user.getId())
    .setHeader("X-Tenant-Id", user.getTenantId())
    .setHeader("X-Auth-Token", jwtToken)
    .build();
```

Async propagation — `DelegatingSecurityContextAsyncTaskExecutor` + message headers. [1796][1799]

### Consumer Side

```java
@KafkaListener(topics = "orders")
public void consume(Order order,
    @Header("X-User-Id") String userId,
    @Header("X-Tenant-Id") String tenantId) {

    // SecurityContext set (consumer thread)
    SecurityContextHolder.getContext().setAuthentication(
        createAuthentication(userId, tenantId)
    );
    try {
        process(order);
    } finally {
        SecurityContextHolder.clearContext();
    }
}
```

---

## 8. Complete Flow

```text
Token Relay:
User -> A -> B -> C (same JWT)

Token Exchange:
User -> A -> STS -> B (naya token)

Signed Assertion:
User -> A -> B (A-signed assertion)

Async:
A -> MQ (headers) -> B (context restore)
```

---

## 9. Decision Framework

```text
User context chahiye?
   |
   +--> Haan
   |     |
   |     Zero trust? -> Token Exchange
   |     Simple? -> Token Relay
   |     Internal trust? -> Signed Assertion
   |
   +--> Nahi -> Client Credentials
```

| Requirement | Pattern |
|---|---|
| Simple relay | Token Relay |
| Zero trust | Token Exchange |
| Internal trust | Signed Assertion |
| Async/MQ | Message headers |
| No user | Client Credentials |

---

## 10. Real-World Examples

| Scenario | Pattern |
|---|---|
| Gateway -> Service | Token Relay |
| Service -> Service (zero trust) | Token Exchange |
| Internal microservices | Signed Assertion |
| Order -> Notification (async) | MQ headers |
| Batch job | Client Credentials |

---

## 11. Common Gotchas

### Gotcha 1 — Header Trust

```text
Problem:
X-User-Id spoof

Fix:
mTLS + JWT bhi validate
```

### Gotcha 2 — Async Context Loss

```text
Problem:
@Async me context null

Fix:
DelegatingSecurityContextAsyncTaskExecutor
```

### Gotcha 3 — Token Expiry in Chain

```text
Problem:
Long chain me expire

Fix:
Token Exchange / refresh
```

### Gotcha 4 — MQ Consumer Context

```text
Problem:
Consumer me context null

Fix:
Message headers + SecurityContext set
```

---

## 12. Interview Questions

| Question | Answer |
|---|---|
| Token propagation kya hai? | User identity service boundaries across. |
| 4 patterns? | Relay, Exchange, Assertion, Headers. |
| Token Relay vs Exchange? | Same token vs naya token. |
| Signed Assertion? | Service khud sign karta hai. |
| Async propagation? | DelegatingSecurityContextAsyncTaskExecutor. |
| MQ propagation? | Message headers + consumer context. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Simple relay | Token Relay |
| Zero trust | Token Exchange |
| Async | MQ headers + context |
| Internal trust | Signed Assertion |
| Header spoof | mTLS + JWT validate |

---

## 13. Key Takeaways

```text
1. Token propagation = user context across services.

2. 4 patterns: Relay, Exchange, Assertion, Headers.

3. Relay = simple, Exchange = zero trust.

4. Assertion = service-signed.

5. Headers = internal only.

6. Async = DelegatingSecurityContextAsyncTaskExecutor.

7. MQ = message headers.

8. Header trust = dangerous.

9. Token expiry = chain issue.

10. Har service validate (zero trust).
```

---

## 14. Next Part Preview

**Part 23: Distributed Session Management**

Isme hum cover karenge:

```text
- Distributed sessions deep dive
- Spring Session + Redis
- Session replication
- Sticky vs stateless
- Real-world strategies
```