# Part 25: Microservices Security — API Versioning & Deprecation Security

> API versioning ka core — **sirf breaking changes par naya version** — deprecation announce karne ke liye **`Deprecation` (RFC 9745) + `Sunset` (RFC 8594) headers**, aur version-specific OAuth2 scopes se old version ko safely sunset karna. [1830][1835][1840][1842]

---

## 1. API Versioning Mental Model

```text
Breaking Change?
   |
   +--> Nahi -> Same version me add (backward compatible)
   |
   +--> Haan -> Naya version (v2)
                |
                +--> v1 deprecate -> Sunset -> Remove
```

### Breaking vs Non-Breaking

| Change | Breaking? | Action |
|---|---|---|
| Naya endpoint add | Nahi | Same version |
| Naya optional field | Nahi | Same version |
| Field remove/rename | Haan | Naya version |
| Required field add | Haan | Naya version |
| Auth change | Haan | Naya version |
| Response structure change | Haan | Naya version |

Sirf breaking changes par naya version — non-breaking changes same version me. [1835][1842]

---

## 2. Versioning Strategies

| Strategy | Example | Pros | Cons |
|---|---|---|---|
| URL Path | `/v1/users` | Simple, visible, cacheable | URL change |
| Custom Header | `X-API-Version: 2` | Clean URLs | Testing hard |
| Media Type | `Accept: application/vnd.api.v2+json` | REST-compliant | Complex |
| Query Param | `?version=2` | Simple | Caching issues |

### Recommendation

```text
Microservices: URL Path (Gateway routing easy)
Public APIs: URL Path (industry standard)
Enterprise internal: Header / Media Type
```

URL path versioning — industry standard (GitHub, Stripe, Google), Gateway routing easy. [1837][1839][1841][1844]

---

## 3. URL Path Versioning

```text
GET /v1/orders
GET /v2/orders
```

### Spring Boot

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderControllerV1 {

    @GetMapping
    public List<OrderV1> getOrders() {
        // V1 response
    }
}

@RestController
@RequestMapping("/api/v2/orders")
public class OrderControllerV2 {

    @GetMapping
    public List<OrderV2> getOrders() {
        // V2 response (naya structure)
    }
}
```

### Gateway Routing

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-v1
          uri: http://order-service-v1:8081
          predicates:
            - Path=/api/v1/orders/**
        - id: order-v2
          uri: http://order-service-v2:8082
          predicates:
            - Path=/api/v2/orders/**
```

URL versioning — Gateway alag service versions route karta hai. [1837][1841]

---

## 4. Deprecation Lifecycle

```text
1. v2 Release
2. v1 Deprecation Announce (Deprecation header)
3. Migration Period (6-12 months)
4. Sunset Date (Sunset header)
5. v1 Remove (410 Gone)
```

### Headers

| Header | RFC | Meaning |
|---|---|---|
| `Deprecation` | RFC 9745 | Deprecated date |
| `Sunset` | RFC 8594 | Removal date |
| `Link` | — | Migration guide |

### Example

```text
HTTP/1.1 200 OK
Deprecation: @1735689600
Sunset: Wed, 01 Jan 2027 00:00:00 GMT
Link: <https://docs.example.com/migrate-v2>; rel="deprecation"
```

`Deprecation` (RFC 9745) + `Sunset` (RFC 8594) — machine-readable deprecation. [1830][1833][1838][1840]

---

## 5. Deprecation Implementation (Spring)

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderControllerV1 {

    @GetMapping
    public ResponseEntity<List<OrderV1>> getOrders() {
        List<OrderV1> orders = orderService.getOrdersV1();

        return ResponseEntity.ok()
            .header("Deprecation", "@1735689600")
            .header("Sunset", "Wed, 01 Jan 2027 00:00:00 GMT")
            .header("Link", "<https://docs.example.com/migrate-v2>; rel=\"deprecation\"")
            .body(orders);
    }
}
```

### Gateway Filter (Centralized)

```java
@Component
public class DeprecationFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(
        ServerWebExchange exchange,
        GatewayFilterChain chain
    ) {
        String path = exchange.getRequest().getPath().value();

        if (path.startsWith("/api/v1/")) {
            exchange.getResponse().getHeaders().add(
                "Deprecation", "@1735689600"
            );
            exchange.getResponse().getHeaders().add(
                "Sunset", "Wed, 01 Jan 2027 00:00:00 GMT"
            );
        }

        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return -50;
    }
}
```

Gateway filter — centralized deprecation headers (har service me duplicate nahi). [1835][1840]

---

## 6. Version-Specific Security

```text
v1: Old auth (API Key)
v2: New auth (OAuth2 + JWT)
```

### Migration Strategy

```text
1. v2 release (OAuth2)
2. v1 deprecated (API Key still works)
3. Migration period
4. v1 sunset
5. v1 remove (API Key invalid)
```

### Gateway Auth per Version

```yaml
routes:
  - id: order-v1
    uri: http://order-v1:8081
    predicates:
      - Path=/api/v1/orders/**
    filters:
      - name: ApiKeyAuth
  - id: order-v2
    uri: http://order-v2:8082
    predicates:
      - Path=/api/v2/orders/**
    filters:
      - TokenRelay=
```

Version-specific auth — v1 (API Key) vs v2 (OAuth2) — migration smooth. [1832][1836]

---

## 7. OAuth2 Scopes per Version

```text
v1 scopes: orders:read, orders:write
v2 scopes: orders:v2:read, orders:v2:write
```

### Kyu?

```text
v1 sunset -> v1 scopes revoke
v2 scopes unaffected
Clean migration
```

Version-specific scopes — v1 sunset par v1 scopes revoke, v2 unaffected. [1831][1834][1836]

---

## 8. Complete Flow

```text
1. v2 release
2. v1 Deprecation header
3. Clients migrate (6-12 months)
4. Usage metrics monitor
5. v1 Sunset date
6. v1 remove (410 Gone)
7. v1 scopes revoke
```

### 410 Gone Response

```text
HTTP/1.1 410 Gone
Link: <https://docs.example.com/migrate-v2>; rel="deprecation"
```

Sunset ke baad — `410 Gone` + migration link. [1830][1835][1840]

---

## 9. Real-World Patterns

| Pattern | Use Case |
|---|---|
| URL versioning | Public APIs, microservices |
| Header versioning | Enterprise internal |
| Media type versioning | REST purist |
| Gateway routing | Version isolation |
| Scope-based sunset | Clean migration |

---

## 10. Common Gotchas

### Gotcha 1 — No Deprecation Notice

```text
Problem:
Clients surprise (breaking)

Fix:
Deprecation + Sunset headers
```

### Gotcha 2 — Sunset Too Early

```text
Problem:
Clients migrate nahi hue

Fix:
Usage metrics + extension
```

### Gotcha 3 — Version Sprawl

```text
Problem:
v1-v5 saare alive

Fix:
Sunset policy enforce
```

### Gotcha 4 — Auth Change Without Version

```text
Problem:
Breaking change same version me

Fix:
Naya version + migration
```

---

## 11. Interview Questions

| Question | Answer |
|---|---|
| API versioning kya hai? | Breaking changes manage karne ka pattern. |
| Breaking vs non-breaking? | Field remove vs add. |
| Versioning strategies? | URL, Header, Media Type, Query. |
| Deprecation vs Sunset? | Announce vs removal date. |
| RFC 8594? | Sunset header. |
| Version-specific scopes? | Clean migration (v1 revoke, v2 safe). |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Breaking change | Naya version |
| Deprecation announce | Deprecation + Sunset headers |
| Migration tracking | Usage metrics |
| Auth migration | Version-specific scopes |
| Sunset ke baad | 410 Gone + migration link |

---

## 12. Key Takeaways

```text
1. Sirf breaking changes par naya version.

2. URL versioning = industry standard.

3. Deprecation (RFC 9745) + Sunset (RFC 8594).

4. Gateway = centralized deprecation headers.

5. Version-specific scopes = clean migration.

6. Usage metrics = migration tracking.

7. Sunset ke baad = 410 Gone.

8. Migration period = 6-12 months.

9. Version sprawl = avoid.

10. Auth change = breaking = naya version.
```

---

## 13. Next Part Preview

**Part 26: Circuit Breaker & Security**

Isme hum cover karenge:

```text
- Circuit breaker deep dive
- Resilience4j
- Security implications
- Fail-safe vs fail-open
- Real-world patterns
```