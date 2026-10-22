# Part 27: Microservices Security — Distributed Tracing (Micrometer Tracing)

> Distributed tracing ka core — **Trace ID + Span ID + W3C `traceparent` header** — Spring Boot 3 me **Micrometer Tracing** (Sleuth ka successor), security context (user ID, tenant) spans me enrich karke audit trail banta hai. [1860][1862][1864][1869]

---

## 1. Distributed Tracing Mental Model

```text
Without Tracing:
User complaint: "Order slow"
-> Kaunsi service? Kaunsa DB query? Pata nahi

With Tracing:
User complaint: "Order slow"
-> Trace ID: abc123
-> Gateway (50ms) -> Order Service (200ms) -> DB (150ms)
-> Bottleneck: DB query
```

### Core Concepts

| Concept | Meaning |
|---|---|
| Trace | Pura request journey (end-to-end) |
| Span | Ek operation (service call, DB query) |
| Trace ID | Unique request ID |
| Span ID | Individual operation ID |
| Parent Span | Caller span |

Trace = pura journey, Span = individual operation — Trace ID se request follow. [1860][1864][1868]

---

## 2. W3C Trace Context

```text
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             |  |                                |                |
           version  trace-id                    parent-span-id    flags
```

### Headers

| Header | Content |
|---|---|
| `traceparent` | version-trace-id-parent-span-id-flags |
| `tracestate` | Vendor-specific data |

W3C `traceparent` — standard propagation format, Spring Boot 3 default. [1860][1866][1868]

### Propagation Flow

```text
Client -> Gateway (traceparent generate)
Gateway -> Service A (traceparent forward)
Service A -> Service B (traceparent forward)
Service B -> DB (span)
```

---

## 3. Spring Boot 3 Setup (Micrometer Tracing)

### Dependencies

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### application.yml

```yaml
management:
  tracing:
    sampling:
      probability: 1.0  # Dev (prod: 0.1)
  otlp:
    tracing:
      endpoint: http://jaeger:4318/v1/traces
```

Micrometer Tracing — Sleuth ka successor, Spring Boot 3 built-in. [1861][1862][1864]

### Sleuth -> Micrometer Migration

| Sleuth (Boot 2) | Micrometer (Boot 3) |
|---|---|
| `spring-cloud-starter-sleuth` | `micrometer-tracing-bridge-otel` |
| `spring.sleuth.*` | `management.tracing.*` |
| B3 format | W3C format (default) |
| Zipkin Brave | OTLP / Brave dono |

Sleuth Boot 3 me kaam nahi karta — Micrometer Tracing migration mandatory. [1861][1862][1870][1872]

---

## 4. Security Context in Traces

```java
@Component
public class SecurityTracingFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(
        ServerWebExchange exchange,
        GatewayFilterChain chain
    ) {
        return exchange.getPrincipal()
            .flatMap(principal -> {
                Span currentSpan = Span.current();
                currentSpan.setAttribute("user.id", principal.getName());
                currentSpan.setAttribute("user.roles", getRoles(principal));
                return chain.filter(exchange);
            })
            .switchIfEmpty(chain.filter(exchange));
    }

    @Override
    public int getOrder() {
        return -50;
    }
}
```

Security context — user ID + roles spans me enrich — audit trail. [1864][1869]

### Custom Attributes

```java
Span.current().setAttribute("user.id", userId);
Span.current().setAttribute("tenant.id", tenantId);
Span.current().setAttribute("auth.method", "jwt");
Span.current().setAttribute("risk.score", riskScore);
```

---

## 5. Sensitive Data Warning

```text
WRONG:
Span.setAttribute("password", password)
Span.setAttribute("token", jwt)
Span.setAttribute("ssn", ssn)

RIGHT:
Span.setAttribute("user.id", userId)  // Non-sensitive
Span.setAttribute("tenant.id", tenantId)
```

**Rule:** PII / credentials / tokens spans me kabhi nahi — tracing backend accessible. [1864]

### Sensitive Data Handling

| Data | Span Me? |
|---|---|
| User ID | Haan (non-sensitive) |
| Tenant ID | Haan |
| Password | Nahi |
| JWT | Nahi |
| PII (SSN, email) | Nahi / masked |

---

## 6. Jaeger Setup

```bash
docker run -d --name jaeger \
  -p 16686:16686 \
  -p 4318:4318 \
  jaegertracing/all-in-one:latest
```

```text
Jaeger UI: http://localhost:16686
OTLP endpoint: http://localhost:4318
```

### Trace Search

```text
1. Jaeger UI open
2. Trace ID search (user complaint se)
3. Waterfall view
4. Bottleneck identify
5. Security events trace
```

Jaeger — trace search + waterfall visualization. [1864][1869]

---

## 7. Complete Flow

```text
1. User -> Gateway
2. Gateway -> traceparent generate
3. Gateway -> Service A (traceparent forward)
4. Service A -> Span + user attributes
5. Service A -> Service B (traceparent forward)
6. Service B -> Span
7. Service B -> DB (span)
8. Jaeger -> Complete trace
```

### Security Incident Trace

```text
Incident: Unauthorized access attempt
   |
   v
Trace ID search
   |
   v
Gateway -> Auth fail -> Service -> 403
Full journey visible
```

---

## 8. Async + Tracing

```text
@Async me context loss (ThreadLocal)

Solution:
Context propagation library
```

```java
@Bean
public Executor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setTaskDecorator(new ContextPropagatingTaskDecorator());
    return executor;
}
```

Context propagation — async me bhi trace context. [1864][1867]

---

## 9. Real-World Patterns

| Pattern | Use Case |
|---|---|
| 100% sampling | Dev |
| 10% sampling | Prod (cost) |
| Tail-based sampling | Errors 100%, success 10% |
| Security events | Always sample |

### Sampling Strategy

| Environment | Sampling |
|---|---|
| Dev | 100% |
| Staging | 100% |
| Prod | 1-10% |
| Security events | 100% |

Sampling — cost vs visibility trade-off. [1869]

---

## 10. Common Gotchas

### Gotcha 1 — Sleuth with Boot 3

```text
Problem:
Sleuth kaam nahi karta

Fix:
Micrometer Tracing migration
```

### Gotcha 2 — Sensitive Data in Spans

```text
Problem:
PII/tokens traces me

Fix:
Sirf non-sensitive attributes
```

### Gotcha 3 — Sampling Miss

```text
Problem:
Security event trace nahi

Fix:
Security events always sample
```

### Gotcha 4 — Async Context Loss

```text
Problem:
@Async me trace break

Fix:
Context propagation decorator
```

---

## 11. Interview Questions

| Question | Answer |
|---|---|
| Distributed tracing kya hai? | Request journey across services. |
| Trace vs Span? | Pura journey vs individual operation. |
| W3C traceparent? | Standard propagation header. |
| Micrometer Tracing? | Sleuth successor (Boot 3). |
| Security context in traces? | User ID + roles (non-sensitive). |
| Sampling? | Cost vs visibility trade-off. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Slow request debug | Trace ID + waterfall |
| Security incident | Trace + user attributes |
| Boot 3 migration | Micrometer Tracing |
| Cost control | Sampling (prod 10%) |
| Async tracing | Context propagation |

---

## 12. Key Takeaways

```text
1. Tracing = request journey visibility.

2. Trace ID + Span ID + W3C.

3. Micrometer Tracing = Boot 3 standard.

4. Sleuth = deprecated (Boot 2 only).

5. Security context = spans me enrich.

6. Sensitive data = kabhi nahi.

7. Jaeger = visualization.

8. Sampling = cost control.

9. Async = context propagation.

10. Security events = always sample.
```

---

## 13. Next Part Preview

**Part 28: Security Logging & Audit Trail**

Isme hum cover karenge:

```text
- Security logging deep dive
- Audit trail design
- Structured logging
- SIEM integration
- Compliance (GDPR, PCI-DSS)
```