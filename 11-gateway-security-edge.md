# Part 11: Microservices Security — Gateway as Security Edge

> Gateway security edge ka core — **authentication (JWT validate) + coarse authorization + token relay + rate limiting + WAF** — sab gateway pe centralize, services sirf fine-grained authorization karti hain. [1663][1664][1665][1668]

---

## 1. Gateway Security Edge Mental Model

```text
Internet
   |
   v
WAF (L7 attacks)
   |
   v
API Gateway
   |
   +--> Authentication (JWT validate)
   +--> Authorization (coarse, path-based)
   +--> Rate Limiting
   +--> Token Relay
   |
   v
Services (fine-grained authz only)
```

### Security Responsibilities

| Concern | Gateway | Service |
|---|---|---|
| Authentication | JWT validate | JWT validate (defense in depth) |
| Authorization | Coarse (path/role) | Fine-grained (business) |
| Rate Limiting | Yes | No |
| Token Relay | Yes | N/A |
| WAF | Yes | No |

Gateway = policy enforcement point (PEP), Auth Server = policy decision point (PDP). [1663][1665][1668]

---

## 2. Authentication at Gateway

```java
@Configuration
@EnableWebSecurity
public class GatewaySecurityConfig {

    @Bean
    public SecurityWebFilterChain filterChain(ServerHttpSecurity http) {
        http
            .csrf(ServerHttpSecurity.CsrfSpec::disable)
            .authorizeExchange(exchange -> exchange
                .pathMatchers("/api/public/**").permitAll()
                .pathMatchers("/api/admin/**").hasAuthority("SCOPE_admin")
                .anyExchange().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(Customizer.withDefaults())
            );
        return http.build();
    }
}
```

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: [https://auth.example.com](https://auth.example.com)
```

Gateway JWT validate — signature, iss, exp, scope — unauthorized requests service tak nahi pahunchte. [1663][1667][1670]

---

## 3. Coarse Authorization at Gateway

```text
Gateway:
- Path-based (admin vs user)
- Role-based (SCOPE_admin)
- Coarse-grained

Service:
- Business logic
- Resource ownership
- Fine-grained
```

### Example

```text
Gateway:
/api/admin/** -> SCOPE_admin required
/api/orders/** -> authenticated

Order Service:
GET /api/orders/{id} -> Order owner check
DELETE /api/orders/{id} -> Owner OR Admin
```

Gateway coarse (path/role), service fine-grained (ownership) — dono zaroori. [1664][1665][1670]

---

## 4. Token Relay

```yaml
spring:
  cloud:
    gateway:
      default-filters:
        - TokenRelay=
```

### Flow

```text
1. Client -> Gateway (Bearer JWT)
2. Gateway validate
3. TokenRelay filter
4. Gateway -> Service (Same JWT)
5. Service validate (defense in depth)
```

`TokenRelay=` — validated JWT downstream forward. [1663][1668]

---

## 5. Custom Global Filter (JWT Validation)

```java
@Component
@RequiredArgsConstructor
public class JwtValidationFilter implements GlobalFilter, Ordered {

    private final JwtService jwtService;

    @Override
    public Mono<Void> filter(
        ServerWebExchange exchange,
        GatewayFilterChain chain
    ) {
        String path = exchange.getRequest().getPath().value();

        if (isPublicEndpoint(path)) {
            return chain.filter(exchange);
        }

        String token = extractToken(exchange.getRequest());
        if (token == null) {
            return unauthorized(exchange, "Missing token");
        }

        return jwtService.validateToken(token)
            .flatMap(claims -> {
                ServerHttpRequest request = exchange.getRequest().mutate()
                    .header("X-User-Id", claims.getSubject())
                    .header("X-User-Roles", getRoles(claims))
                    .build();
                return chain.filter(exchange.mutate().request(request).build());
            })
            .onErrorResume(e -> unauthorized(exchange, "Invalid token"));
    }

    @Override
    public int getOrder() {
        return -100;  // Early in chain
    }
}
```

Custom `GlobalFilter` — JWT validate + user info headers downstream. [1662][1669][1671][1672]

### Header Propagation

```text
Gateway -> Service:
X-User-Id: user-123
X-User-Roles: USER,ADMIN
X-Tenant-Id: tenant-1
```

**Warning:** Headers trust nahi karna — service bhi JWT validate karti hai (headers spoof ho sakte hain agar network compromised). [1671]

---

## 6. Rate Limiting

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: http://localhost:8081
          predicates:
            - Path=/api/orders/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
                key-resolver: "#{@userKeyResolver}"
```

```java
@Bean
public KeyResolver userKeyResolver() {
    return exchange -> Mono.just(
        exchange.getRequest().getHeaders()
            .getFirst("X-User-Id") != null
            ? exchange.getRequest().getHeaders().getFirst("X-User-Id")
            : exchange.getRequest().getRemoteAddress().getAddress().getHostAddress()
    );
}
```

Rate limiting per-user (JWT sub) ya per-IP — Redis distributed. [1551][1664][1670][1673]

### Rate Limit Options

| Option | Meaning |
|---|---|
| `replenishRate` | Sustained rate (req/sec) |
| `burstCapacity` | Max burst |
| `key-resolver` | Per-user / per-IP |

---

## 7. WAF Integration

```text
WAF (ModSecurity / AWS WAF / Cloudflare)
   |
   | L7 attack detection
   v
API Gateway
   |
   v
Services
```

### WAF Blocks

```text
- SQL injection patterns
- XSS patterns
- Path traversal
- Known attack signatures
- Bot traffic
- DDoS
```

WAF gateway se pehle — L7 attacks filter, gateway business security. [1664][1667]

---

## 8. Complete Gateway Config

```yaml
server:
  port: 8080

spring:
  cloud:
    gateway:
      default-filters:
        - TokenRelay=
      routes:
        - id: order-service
          uri: http://localhost:8081
          predicates:
            - Path=/api/orders/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: [https://auth.example.com](https://auth.example.com)
```

---

## 9. Complete Flow

```text
1. Client -> WAF (L7 filter)
2. WAF -> Gateway
3. Gateway -> JWT validate
4. Gateway -> Coarse authz (path/role)
5. Gateway -> Rate limit check
6. Gateway -> TokenRelay
7. Gateway -> Service
8. Service -> JWT validate (defense in depth)
9. Service -> Fine-grained authz
10. Response back
```

---

## 10. Real-World Patterns

| Pattern | Use Case |
|---|---|
| Gateway-only auth | Simple (not recommended) |
| Gateway + Service auth | Standard (zero trust) |
| Gateway + WAF | High security |
| Gateway + Service Mesh | Advanced |

---

## 11. Common Gotchas

### Gotcha 1 — Gateway-Only Auth

```text
Problem:
Service direct access = bypass

Fix:
Service bhi validate (zero trust)
```

### Gotcha 2 — Header Trust

```text
Problem:
X-User-Id header trust

Fix:
Service bhi JWT validate
```

### Gotcha 3 — Rate Limit Bypass

```text
Problem:
Direct service access

Fix:
NetworkPolicies (sirf gateway se)
```

### Gotcha 4 — Filter Order Wrong

```text
Problem:
Auth filter baad me

Fix:
getOrder() = -100 (early)
```

---

## 12. Interview Questions

| Question | Answer |
|---|---|
| Gateway security edge kya? | Central auth + rate limit + relay. |
| PEP vs PDP? | Enforcement vs Decision point. |
| Coarse vs fine-grained? | Path/role vs business logic. |
| Token Relay? | JWT downstream forward. |
| Rate limiting kaise? | RequestRateLimiter + Redis + KeyResolver. |
| WAF kya karta hai? | L7 attacks filter. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Central auth | Gateway JWT validate |
| User context forward | Token Relay / headers |
| Rate limiting | Redis + KeyResolver |
| L7 attacks | WAF |
| Zero trust | Gateway + Service dono validate |

---

## 13. Key Takeaways

```text
1. Gateway = security edge (PEP).

2. Auth + coarse authz + rate limit + relay.

3. Service = fine-grained authz.

4. TokenRelay = JWT forward.

5. Custom GlobalFilter = JWT validation.

6. Headers = user context (trust nahi).

7. Rate limiting = per-user/IP.

8. WAF = L7 attacks.

9. Zero trust = dono validate.

10. NetworkPolicies = direct access block.
```

---

## 14. Next Part Preview

**Part 12: Token Relay Pattern Deep Dive**

Isme hum cover karenge:

```text
- Token Relay deep dive
- TokenRelay filter internals
- Token Exchange vs Relay
- Custom token propagation
- Real-world scenarios
```