# Part 29: Microservices Security — Metrics & Alerting (Prometheus/Grafana)

> Security metrics ka core — **auth failures rate, RBAC denials, circuit breaker state, rate limit hits** — Micrometer counters/gauges + Prometheus scrape + Grafana dashboard + alert rules (`for: 2m` se flapping avoid). [1889][1890][1894][1899]

---

## 1. Metrics Mental Model

```text
Logs = "Kya hua" (events)
Metrics = "Kitna hua" (aggregates)
Traces = "Kahan hua" (journey)

Security Metrics:
- Auth failures rate
- RBAC denials
- Rate limit hits
- Circuit breaker state
- Token validation failures
```

### Metrics Types

| Type | Purpose | Example |
|---|---|---|
| Counter | Monotonic increase | Auth failures total |
| Gauge | Current value | Active sessions |
| Timer | Duration | Auth latency |
| Histogram | Distribution | Request size |

Micrometer — Counter (events), Gauge (current), Timer (duration), Histogram (distribution). [1889][1895][1898]

---

## 2. Micrometer Setup

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

### application.yml

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  metrics:
    tags:
      application: ${spring.application.name}
      environment: ${ENV:dev}
```

`/actuator/prometheus` — Prometheus scrape endpoint. [1889][1897][1899]

---

## 3. Security Metrics (Custom)

```java
@Service
@RequiredArgsConstructor
public class SecurityMetricsService {

    private final MeterRegistry registry;

    private final Counter authSuccess;
    private final Counter authFailure;
    private final Counter accessDenied;
    private final Counter rateLimitHit;

    public SecurityMetricsService(MeterRegistry registry) {
        this.registry = registry;
        this.authSuccess = Counter.builder("security.auth.success")
            .description("Successful authentications")
            .tag("result", "success")
            .register(registry);
        this.authFailure = Counter.builder("security.auth.failures")
            .description("Failed authentications")
            .tag("result", "failure")
            .register(registry);
        this.accessDenied = Counter.builder("security.access.denied")
            .description("Authorization denials")
            .register(registry);
        this.rateLimitHit = Counter.builder("security.ratelimit.hits")
            .description("Rate limit hits")
            .register(registry);
    }

    public void recordAuthSuccess() {
        authSuccess.increment();
    }

    public void recordAuthFailure(String reason) {
        Counter.builder("security.auth.failures")
            .tag("reason", reason)
            .register(registry)
            .increment();
    }

    public void recordAccessDenied() {
        accessDenied.increment();
    }

    public void recordRateLimitHit() {
        rateLimitHit.increment();
    }
}
```

Custom counters — auth success/failure, access denied, rate limit hits — tags se dimensions. [1894][1895][1898]

### Active Sessions (Gauge)

```java
@Component
public class SessionMetrics {

    private final AtomicInteger activeSessions = new AtomicInteger(0);

    public SessionMetrics(MeterRegistry registry) {
        Gauge.builder("security.sessions.active", activeSessions,
            AtomicInteger::get)
            .description("Current active sessions")
            .register(registry);
    }

    public void sessionCreated() {
        activeSessions.incrementAndGet();
    }

    public void sessionDestroyed() {
        activeSessions.decrementAndGet();
    }
}
```

Gauge — active sessions (current value). [1895][1898]

---

## 4. Prometheus Queries

```promql
# Auth failure rate (per minute)
sum(rate(security_auth_failures_total[5m])) * 60

# Auth failures by IP (brute force)
sum by (source_ip) (rate(security_auth_failures_total[5m]))

# Access denied rate
sum(rate(security_access_denied_total[5m])) * 60

# Circuit breaker OPEN count
sum(resilience4j_circuitbreaker_state{state="open"})

# Rate limit hits
sum(rate(security_ratelimit_hits_total[5m])) * 60

# 5xx error rate
sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
```

PromQL — `rate()` + `sum by()` — per-IP, per-service aggregation. [1891][1894][1899]

---

## 5. Alerting Rules

```yaml
groups:
  - name: security-alerts
    interval: 30s
    rules:
      # Brute force
      - alert: BruteForceDetected
        expr: sum by (source_ip) (rate(security_auth_failures_total[5m])) > 0.5
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Brute force: {{ $labels.source_ip }}"
          description: "{{ $value }} auth failures/sec"

      # Circuit breaker OPEN
      - alert: CircuitBreakerOpen
        expr: resilience4j_circuitbreaker_state{state="open"} == 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Circuit breaker OPEN: {{ $labels.name }}"

      # High 5xx rate
      - alert: HighErrorRate
        expr: sum(rate(http_requests_total{status=~"5.."}[5m])) by (job) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High 5xx: {{ $labels.job }}"
```

`for: 2m` — flapping avoid (transient spikes ignore). [1890][1893][1894]

### Alert Design

| Alert | Condition | For | Severity |
|---|---|---|---|
| Brute force | >0.5 failures/sec/IP | 2m | Warning |
| Circuit OPEN | state=open | 1m | Critical |
| High 5xx | >5% | 5m | Critical |
| Rate limit abuse | >50 hits/min | 5m | Warning |

---

## 6. Grafana Dashboard

```text
Dashboard: Security Overview
   |
   +--> Panel 1: Auth success/failure rate
   +--> Panel 2: Access denied by service
   +--> Panel 3: Rate limit hits
   +--> Panel 4: Circuit breaker states
   +--> Panel 5: Active sessions
   +--> Panel 6: 5xx error rate
```

### Panel Query Example

```promql
# Auth failures (time series)
sum(rate(security_auth_failures_total[5m])) * 60

# By reason
sum by (reason) (rate(security_auth_failures_total[5m])) * 60
```

Grafana — time series, pie, gauge, table panels + alerting. [1888][1891][1897]

---

## 7. Complete Flow

```text
1. Security event (auth failure)
2. Micrometer counter increment
3. /actuator/prometheus expose
4. Prometheus scrape (15s)
5. Grafana dashboard
6. Alert rule evaluate
7. Alert firing -> Alertmanager -> Slack/PagerDuty
```

### Alertmanager

```yaml
route:
  receiver: slack
receivers:
  - name: slack
    slack_configs:
      - channel: '#security-alerts'
```

Prometheus -> Alertmanager -> Slack/PagerDuty. [1890][1901]

---

## 8. Real-World Metrics

| Metric | Alert Threshold |
|---|---|
| Auth failures | >30/min/IP |
| Access denied | >10/min/service |
| Circuit OPEN | Any |
| Rate limit hits | >50/min |
| 5xx rate | >5% |
| Active sessions | Sudden spike |

---

## 9. Common Gotchas

### Gotcha 1 — High Cardinality Tags

```text
Problem:
user_id tag = explosion

Fix:
Low-cardinality tags only (service, reason)
```

### Gotcha 2 — No `for` Clause

```text
Problem:
Flapping alerts

Fix:
for: 2m (sustained condition)
```

### Gotcha 3 — Prometheus Endpoint Public

```text
Problem:
Metrics expose

Fix:
Internal network + auth
```

### Gotcha 4 — No Alert Runbook

```text
Problem:
Alert aaye, kya karein?

Fix:
runbook_url annotation
```

High cardinality — user_id tag avoid, low-cardinality (service, reason). [1894][1900]

---

## 10. Interview Questions

| Question | Answer |
|---|---|
| Metrics vs Logs vs Traces? | Kitna vs kya hua vs kahan. |
| Counter vs Gauge? | Monotonic vs current value. |
| PromQL rate()? | Per-second rate. |
| `for` clause? | Sustained condition (flapping avoid). |
| High cardinality? | Tag explosion — avoid. |
| Security metrics? | Auth failures, denials, CB state. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Brute force detect | Auth failure rate + per-IP |
| Circuit OPEN alert | State metric + `for: 1m` |
| Dashboard | Grafana + PromQL |
| Flapping | `for` clause |
| Cardinality | Low-cardinality tags |

---

## 11. Key Takeaways

```text
1. Metrics = aggregates (counters, gauges).

2. Micrometer = facade, Prometheus = backend.

3. Security metrics = auth, authz, CB, rate limit.

4. PromQL = rate() + sum by().

5. Alert rules = `for` clause.

6. Grafana = dashboards + alerting.

7. High cardinality = avoid.

8. Runbook = alert annotations.

9. Alertmanager = routing.

10. Prometheus endpoint = internal only.
```

---

## 12. Next Part Preview

**Part 30: Secrets Management (Vault/K8s)**

Isme hum cover karenge:

```text
- Secrets management deep dive
- HashiCorp Vault
- Kubernetes Secrets
- Secret rotation
- Spring Cloud Vault
```