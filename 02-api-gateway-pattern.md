# Part 2: Microservices Security — API Gateway Pattern & Spring Cloud Gateway

> API Gateway microservices ka **single entry point** hai — routing, security, rate limiting, token relay sab yahan centralize hote hain; Spring Cloud Gateway me **Route + Predicate + Filter** teen core building blocks hain. [1538][1539][1554][1561]

---

## 1. API Gateway Mental Model

```text
Without Gateway:
Client -> Service A
Client -> Service B
Client -> Service C
(Clients ko har service ka pata)

With Gateway:
Client -> Gateway -> Service A/B/C
(Clients sirf gateway ko jaante hain)
```

### Gateway Responsibilities

```text
1. Routing (path -> service)
2. Security (auth, token validation)
3. Rate Limiting
4. Token Relay
5. Request/Response Transformation
6. Observability (logging, tracing)
```

API Gateway single entry point — clients ko internal services ka pata nahi, security centralize. [1539][1540][1545][1548]

---

## 2. Spring Cloud Gateway Overview

```text
Spring Cloud Gateway
   |
   +--> Route (ID + URI + Predicates + Filters)
   |
   +--> Predicate (Request match condition)
   |
   +--> Filter (Request/Response modify)
```

### Core Concepts

| Concept | Purpose |
|---|---|
| Route | Path -> Service mapping |
| Predicate | Request match condition |
| Filter | Request/Response modify |
| URI | Target service |

Spring Cloud Gateway reactive (WebFlux) based hai — non-blocking, high throughput. [1554][1556][1561][1563]

---

## 3. Setup

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

---

## 4. Routes Configuration (YAML)

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
            - StripPrefix=0

        - id: product-service
          uri: http://localhost:8082
          predicates:
            - Path=/api/products/**

        - id: user-service
          uri: http://localhost:8083
          predicates:
            - Path=/api/users/**
```

### Route Anatomy

```text
- id: order-service          (unique name)
  uri: http://localhost:8081 (target)
  predicates:                (match condition)
    - Path=/api/orders/**
  filters:                   (modification)
    - StripPrefix=0
```

Route = ID + URI + Predicates + Filters — request match hone par target service. [1552][1554][1561][1563]

---

## 5. Predicates (Deep Dive)

```yaml
predicates:
  - Path=/api/orders/**
  - Method=GET,POST
  - Header=Authorization, Bearer .*
  - Query=version, v1
  - Host=api.example.com
```

### Common Predicates

| Predicate | Match |
|---|---|
| `Path` | URL path |
| `Method` | HTTP method |
| `Header` | Header pattern |
| `Query` | Query param |
| `Host` | Host header |
| `After/Before/Between` | Time-based |

Predicate = "kab route hoga" — path, method, header, query, host, time. [1552][1558][1561]

---

## 6. Filters (Deep Dive)

```yaml
filters:
  - StripPrefix=1
  - AddRequestHeader=X-Gateway, true
  - AddResponseHeader=X-Response, from-gateway
  - RewritePath=/api/(?<path>.*), /$\{path}
  - name: RequestRateLimiter
    args:
      redis-rate-limiter.replenishRate: 10
      redis-rate-limiter.burstCapacity: 20
```

### Common Filters

| Filter | Purpose |
|---|---|
| `StripPrefix` | Path prefix remove |
| `RewritePath` | Path rewrite |
| `AddRequestHeader` | Header add |
| `AddResponseHeader` | Response header |
| `RequestRateLimiter` | Rate limiting |
| `TokenRelay` | JWT forward |
| `CircuitBreaker` | Resilience |

Filter = "request/response kaise modify hoga" — headers, path, rate limit, token relay. [1551][1553][1554][1561]

---

## 7. Java DSL Configuration

```java
@Configuration
public class GatewayConfig {

    @Bean
    public RouteLocator routes(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("order-service", r -> r
                .path("/api/orders/**")
                .filters(f -> f
                    .stripPrefix(0)
                    .requestRateLimiter(c -> c
                        .setRateLimiter(redisRateLimiter()))
                )
                .uri("http://localhost:8081")
            )
            .route("product-service", r -> r
                .path("/api/products/**")
                .uri("http://localhost:8082")
            )
            .build();
    }
}
```

Java DSL — programmatic routes, dynamic config ke liye. [1556][1560][1561]

---

## 8. Gateway + Security Integration

```java
@Configuration
@EnableWebSecurity
public class GatewaySecurityConfig {

    @Bean
    public SecurityWebFilterChain springSecurityFilterChain(
        ServerHttpSecurity http
    ) {
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

Gateway pe JWT validate — services ko sirf valid requests. [1555][1556][1559]

---

## 9. Rate Limiting at Gateway

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
                redis-rate-limiter.requestedTokens: 1
```

### Rate Limiter Options

| Option | Meaning |
|---|---|
| `replenishRate` | Requests per second (sustained) |
| `burstCapacity` | Max burst |
| `requestedTokens` | Cost per request |

`RequestRateLimiter` + Redis — distributed rate limiting. [1551][1553][1557][1560]

---

## 10. Complete Gateway Config

```yaml
server:
  port: 8080

spring:
  application:
    name: api-gateway
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
        - id: product-service
          uri: http://localhost:8082
          predicates:
            - Path=/api/products/**
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: [https://auth.example.com](https://auth.example.com)
```

---

## 11. Real-World Gateway Patterns

| Pattern | Use Case |
|---|---|
| Single Gateway | Simple microservices |
| BFF per client | Web + Mobile alag |
| Gateway + Service Mesh | Advanced |
| Gateway + WAF | High security |

### BFF Pattern

```text
Web Client -> Web BFF -> Services
Mobile Client -> Mobile BFF -> Services
```

BFF — har client type ke liye alag gateway, optimized responses. [1540][1548]

---

## 12. Common Gotchas

### Gotcha 1 — Route Not Matching

```text
Problem:
404 from gateway

Fix:
Predicate path check karo
```

### Gotcha 2 — Rate Limiter Not Working

```text
Problem:
Redis connection missing

Fix:
Redis dependency + config
```

### Gotcha 3 — Token Relay Missing

```text
Problem:
Downstream token null

Fix:
TokenRelay= default filter
```

### Gotcha 4 — CORS Issue

```text
Problem:
Browser block

Fix:
Gateway pe CORS config
```

---

## 13. Interview Questions

| Question | Answer |
|---|---|
| API Gateway kya hai? | Single entry point — routing + security. |
| Route kya hai? | Path -> Service mapping. |
| Predicate kya hai? | Request match condition. |
| Filter kya hai? | Request/Response modify. |
| Token Relay kya hai? | JWT downstream forward. |
| Rate limiting kaise? | RequestRateLimiter + Redis. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Single entry point | API Gateway |
| Path-based routing | Predicate Path |
| Rate limiting | RequestRateLimiter + Redis |
| JWT forward | TokenRelay filter |
| Per-client gateway | BFF pattern |

---

## 14. Key Takeaways

```text
1. Gateway = single entry point.

2. Route = ID + URI + Predicates + Filters.

3. Predicate = match condition.

4. Filter = modification.

5. TokenRelay = JWT forward.

6. RequestRateLimiter = rate limiting.

7. Gateway pe JWT validate.

8. BFF = per-client gateway.

9. Reactive (WebFlux) based.

10. Security centralize at gateway.
```

---

## 15. Next Part Preview

**Part 3: Service Discovery & Secure Service Registry**

Isme hum cover karenge:

```text
- Service discovery pattern
- Eureka setup
- Secure service registry
- Service-to-service communication
- Load balancing
```