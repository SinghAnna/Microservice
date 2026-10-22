# Part 26: Microservices Security — Circuit Breaker & Security

> Circuit breaker ka core — **3 states (CLOSED → OPEN → HALF_OPEN)** — cascading failures rokta hai; security me **fail-closed** (auth/authz fail = deny) default hona chahiye, fail-open sirf explicitly documented degraded mode ke liye. [1847][1849][1850][1855]

---

## 1. Circuit Breaker Mental Model

```text
Without Circuit Breaker:
Service A -> Service B (down)
Service A threads block -> Thread pool exhaust
Service A bhi down -> Cascading failure

With Circuit Breaker:
Service A -> Circuit Breaker -> Service B (down)
   |
   +--> OPEN -> Fail fast -> Fallback
```

### Electrical Analogy

```text
Ghar ka breaker:
Overload -> Trip -> Circuit open -> Protection

Microservices:
Failures -> Trip -> Circuit open -> Protection
```

Circuit breaker — cascading failures prevent, failing dependency ko calls rokta hai. [1847][1849][1859]

---

## 2. Three States

```text
CLOSED (Normal)
   |
   | Failure rate > threshold
   v
OPEN (Blocking)
   |
   | Wait duration over
   v
HALF_OPEN (Testing)
   |
   +--> Success -> CLOSED
   +--> Fail -> OPEN
```

| State | Behavior |
|---|---|
| CLOSED | Normal — sab requests pass, failures monitor |
| OPEN | Blocked — fail fast, fallback |
| HALF_OPEN | Limited probe calls — recovery test |

3 states — CLOSED (normal), OPEN (block), HALF_OPEN (probe). [1847][1851][1853][1857]

---

## 3. Resilience4j Setup

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

### Configuration

```yaml
resilience4j:
  circuitbreaker:
    instances:
      productService:
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 3
        automaticTransitionFromOpenToHalfOpenEnabled: true
```

| Option | Meaning |
|---|---|
| `slidingWindowSize` | Last N calls evaluate |
| `minimumNumberOfCalls` | Minimum calls before evaluate |
| `failureRateThreshold` | % failures -> OPEN |
| `waitDurationInOpenState` | OPEN kitni der |
| `permittedNumberOfCallsInHalfOpenState` | Probe calls |

Resilience4j — count-based/time-based sliding window, failure rate threshold. [1845][1847][1854][1857]

---

## 4. @CircuitBreaker + Fallback

```java
@Service
@RequiredArgsConstructor
public class ProductService {

    @CircuitBreaker(name = "productService", fallbackMethod = "fallbackProduct")
    public Product getProduct(Long id) {
        return productClient.getProduct(id);
    }

    private Product fallbackProduct(Long id, Throwable ex) {
        // Cached data / default / graceful degradation
        return Product.builder()
            .id(id)
            .name("Unavailable")
            .available(false)
            .build();
    }
}
```

`@CircuitBreaker` + `fallbackMethod` — OPEN state me fallback execute. [1853][1854][1857]

### Fallback Signature

```text
Same return type + same args + Throwable last param
```

---

## 5. Security Implications

### Fail-Closed (Security First)

```text
Auth Server down?
   |
   +--> Fail-closed: Deny all (secure)
   +--> Fail-open: Allow all (INSECURE!)
```

| Scenario | Fail-Closed | Fail-Open |
|---|---|---|
| Auth Server down | Deny (secure) | Allow (DANGER) |
| Token validation fail | 401 | Bypass (DANGER) |
| Policy engine down | Deny | Bypass (DANGER) |

**Rule:** Auth/AuthZ components = fail-closed (security > availability). [1848][1850][1852][1855]

### Fail-Open (Availability First)

```text
Recommendation service down?
   |
   +--> Fail-open: Cached/default recommendations
   (Non-critical feature, availability > strictness)
```

| Scenario | Mode | Reason |
|---|---|---|
| Auth/AuthZ | Fail-closed | Security critical |
| Recommendations | Fail-open | Non-critical |
| Analytics | Fail-open | Non-critical |
| Payment validation | Fail-closed | Financial integrity |

Fail-closed = security-critical, fail-open = non-critical features. [1848][1850][1855]

---

## 6. Circuit Breaker + Auth (Danger)

```java
// WRONG - Fail-open auth
@CircuitBreaker(name = "authService", fallbackMethod = "allowAll")
public boolean validateToken(String token) {
    return authClient.validate(token);
}

private boolean allowAll(String token, Throwable ex) {
    return true;  // DANGER! Bypass
}
```

### Correct (Fail-Closed)

```java
// CORRECT - Fail-closed auth
@CircuitBreaker(name = "authService", fallbackMethod = "denyAll")
public boolean validateToken(String token) {
    return authClient.validate(token);
}

private boolean denyAll(String token, Throwable ex) {
    return false;  // Secure - deny
}
```

Auth circuit breaker fallback = `false` (deny) — kabhi `true` (allow) nahi. [1850][1852][1855]

---

## 7. Gateway + Circuit Breaker

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
            - name: CircuitBreaker
              args:
                name: orderCB
                fallbackUri: forward:/fallback/orders
```

```java
@RestController
public class FallbackController {

    @RequestMapping("/fallback/orders")
    public ResponseEntity<String> ordersFallback() {
        return ResponseEntity
            .status(HttpStatus.SERVICE_UNAVAILABLE)
            .body("Order service temporarily unavailable");
    }
}
```

Gateway CircuitBreaker filter + fallback URI — route-level resilience. [1845][1846]

---

## 8. Complete Flow

```text
1. Request -> Gateway
2. Circuit Breaker check (CLOSED?)
3. CLOSED -> Service call
4. Failure rate > 50% -> OPEN
5. OPEN -> Fail fast -> Fallback
6. 10 sec -> HALF_OPEN
7. Probe calls -> Success -> CLOSED
8. Fail -> OPEN again
```

---

## 9. Real-World Patterns

| Pattern | Use Case |
|---|---|
| Circuit Breaker + Retry | Transient failures |
| Circuit Breaker + Rate Limiter | Overload protection |
| Circuit Breaker + Bulkhead | Resource isolation |
| Circuit Breaker + Fallback | Graceful degradation |

### Bulkhead (Resource Isolation)

```text
Service A pool: 10 threads
Service B pool: 10 threads
(A ek down -> B unaffected)
```

Bulkhead — thread pool isolation, cascading failure prevent. [1854][1857]

---

## 10. Common Gotchas

### Gotcha 1 — Auth Fail-Open

```text
Problem:
Auth down = sab allow

Fix:
Fail-closed (deny)
```

### Gotcha 2 — No Fallback

```text
Problem:
OPEN = exception to user

Fix:
Graceful fallback
```

### Gotcha 3 — Threshold Too Low

```text
Problem:
Jaldi OPEN (false trips)

Fix:
minimumNumberOfCalls + threshold tune
```

### Gotcha 4 — No Monitoring

```text
Problem:
OPEN state unnoticed

Fix:
Actuator + Prometheus alerts
```

Circuit breaker states — Actuator + Micrometer se monitor + alert. [1854][1857]

---

## 11. Interview Questions

| Question | Answer |
|---|---|
| Circuit Breaker kya hai? | Cascading failures prevent — 3 states. |
| 3 states? | CLOSED, OPEN, HALF_OPEN. |
| Fail-open vs fail-closed? | Allow vs deny on failure. |
| Auth me kaunsa? | Fail-closed (security first). |
| Resilience4j? | Hystrix successor — CB, Retry, Bulkhead. |
| Fallback? | Graceful degradation (OPEN state me). |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Auth Server down | Fail-closed (deny) |
| Recommendations down | Fail-open (cached) |
| Cascading failure | Circuit Breaker + Bulkhead |
| Transient failure | CB + Retry |
| Monitoring | Actuator + Prometheus |

---

## 12. Key Takeaways

```text
1. Circuit Breaker = cascading failure prevent.

2. 3 states: CLOSED, OPEN, HALF_OPEN.

3. Fail-closed = security-critical.

4. Fail-open = non-critical only.

5. Auth fallback = deny (never allow).

6. Resilience4j = modern library.

7. Fallback = graceful degradation.

8. Bulkhead = resource isolation.

9. Monitoring = Actuator + alerts.

10. Gateway + CB = route-level resilience.
```

---

## 13. Next Part Preview

**Part 27: Distributed Tracing (Sleuth/Micrometer)**

Isme hum cover karenge:

```text
- Distributed tracing deep dive
- Trace ID / Span ID
- Micrometer Tracing
- Security context in traces
- Jaeger / Zipkin
```