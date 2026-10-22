# Part 28: Microservices Security — Security Logging & Audit Trail

> Security logging ka core — **OWASP: "when, where, who, what" har security event ke liye** — structured JSON logs + MDC (trace ID) + SIEM integration; PCI-DSS me 12 months retention, 3 months immediately available. [1873][1874][1877][1885]

---

## 1. Security Logging Mental Model

```text
Without Security Logging:
Attacker brute force -> Pata nahi chala
Data breach -> Forensics impossible
Compliance audit -> Fail

With Security Logging:
Attacker brute force -> Alert + block
Data breach -> Timeline reconstruct
Compliance audit -> Pass
```

### Kya Log Karein

```text
- Auth success + failure
- AuthZ failures (403)
- Input validation failures
- Session anomalies
- Admin actions
- Config changes
- High-risk operations
```

OWASP — auth success/failure, authz failures, validation failures, session anomalies, admin actions log. [1873][1881][1883]

---

## 2. Log Entry Structure (When, Where, Who, What)

```json
{
  "timestamp": "2026-10-22T12:19:32Z",
  "level": "WARN",
  "event": "AUTHENTICATION_FAILURE",
  "user_id": "user-123",
  "tenant_id": "tenant-1",
  "ip": "203.0.113.5",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "action": "LOGIN",
  "object": "/api/auth/login",
  "result": "FAILURE",
  "reason": "INVALID_CREDENTIALS",
  "service": "auth-service"
}
```

### Required Fields (OWASP)

| Field | Purpose |
|---|---|
| `timestamp` | When |
| `ip` / `service` | Where |
| `user_id` | Who |
| `event` + `action` + `result` | What |
| `trace_id` + `span_id` | Correlation |
| `severity` | Priority |

"when, where, who, what" + trace correlation — OWASP mandatory fields. [1873][1874][1881]

---

## 3. Events to Log

| Event | Level | Example |
|---|---|---|
| Auth success | INFO | `LOGIN_SUCCESS` |
| Auth failure | WARN | `LOGIN_FAILURE` |
| AuthZ failure | WARN | `ACCESS_DENIED` |
| Validation failure | WARN | `INPUT_VALIDATION_FAIL` |
| Session anomaly | WARN | `SESSION_FIXATION_ATTEMPT` |
| Admin action | INFO | `USER_CREATED` |
| Config change | INFO | `SECURITY_CONFIG_CHANGED` |
| Token revocation | INFO | `TOKEN_REVOKED` |
| Rate limit hit | WARN | `RATE_LIMIT_EXCEEDED` |

OWASP — auth, authz, validation, session, admin, config, high-risk operations. [1873][1881][1883]

---

## 4. Structured Logging (Spring Boot 3.4+)

```yaml
logging:
  structured:
    format:
      console: ecs  # Elastic Common Schema
  level:
    root: INFO
    security: WARN
```

### Logback JSON

```xml
<appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <customFields>
            {"service":"order-service","env":"prod"}
        </customFields>
    </encoder>
</appender>
```

Spring Boot 3.4+ — built-in structured logging (ECS, GELF, Logstash JSON). [1876][1878][1882]

### MDC (Trace Correlation)

```java
MDC.put("trace_id", traceId);
MDC.put("user_id", userId);
MDC.put("tenant_id", tenantId);

try {
    log.warn("AUTHENTICATION_FAILURE");
} finally {
    MDC.clear();
}
```

MDC — trace ID + user ID har log entry me automatic. [1876][1879]

---

## 5. Audit Trail Design

```text
Audit Log:
- Immutable (tamper-proof)
- Separate storage (app logs se alag)
- Retention: 12 months (PCI-DSS)
- 3 months immediately available
```

### Audit Event Example

```json
{
  "event": "DATA_ACCESS",
  "user_id": "user-123",
  "tenant_id": "tenant-1",
  "resource": "order-456",
  "action": "READ",
  "result": "SUCCESS",
  "timestamp": "2026-10-22T12:19:32Z",
  "ip": "203.0.113.5",
  "trace_id": "abc123"
}
```

Audit trail — immutable, separate storage, 12 months retention (PCI-DSS). [1875][1877][1885]

### PCI-DSS Requirements

```text
Req 10.2: Log all access to cardholder data
Req 10.5.1: 12 months retention, 3 months immediately available
Req 10.7: Daily log review
```

PCI-DSS Req 10 — cardholder data access log + 12 months retention. [1875][1877][1885][1887]

---

## 6. What NOT to Log

```text
WRONG:
- Passwords
- JWT tokens
- API keys
- Credit card numbers (full)
- SSN / PII
- Session IDs (full)

RIGHT:
- Masked: **-**-****-1234
- User ID (non-sensitive)
- Event type + result
```

**Rule:** Credentials, tokens, PII logs me kabhi nahi — masking / redaction. [1884][1886]

### Masking Example

```java
public String maskCard(String card) {
    return "**-**-****-" + card.substring(card.length() - 4);
}
```

---

## 7. SIEM Integration

```text
Services -> JSON logs -> Log Shipper (Fluent Bit) -> SIEM (Splunk/ELK)
   |
   v
SIEM -> Correlation -> Alerts -> SOC
```

### SIEM Use Cases

| Use Case | Detection |
|---|---|
| Brute force | 5+ failures / min / IP |
| Credential stuffing | Many IPs, same user |
| Privilege escalation | Admin action anomaly |
| Data exfiltration | Unusual data access volume |
| Lateral movement | Cross-service anomaly |

SIEM — log correlation + real-time alerts + SOC. [1875][1877][1885]

### Alert Example

```text
Alert: Brute Force Detected
Condition: 5+ LOGIN_FAILURE from same IP in 1 min
Action: Block IP + notify SOC
```

---

## 8. Complete Flow

```text
1. Security event (login failure)
2. MDC (trace + user + tenant)
3. Structured JSON log
4. Log shipper (Fluent Bit)
5. SIEM (Elastic/Splunk)
6. Correlation + alert
7. SOC response
```

---

## 9. Real-World Patterns

| Pattern | Use Case |
|---|---|
| Centralized logging | All services -> one place |
| Audit trail (separate) | Compliance |
| Real-time alerting | SOC |
| Log retention | 12 months (PCI) |
| Immutable logs | Tamper-proof |

---

## 10. Common Gotchas

### Gotcha 1 — Sensitive Data in Logs

```text
Problem:
Password/token logs me

Fix:
Masking + redaction + review
```

### Gotcha 2 — No Trace Correlation

```text
Problem:
Logs scattered, incident trace nahi

Fix:
MDC (trace ID + user ID)
```

### Gotcha 3 — Logs in App Container

```text
Problem:
Container restart = logs lost

Fix:
Centralized (ELK/Splunk)
```

### Gotcha 4 — No Retention Policy

```text
Problem:
Compliance fail

Fix:
12 months retention (PCI-DSS)
```

---

## 11. Interview Questions

| Question | Answer |
|---|---|
| Security logging kya? | Security events capture + correlate. |
| "When, where, who, what"? | OWASP mandatory fields. |
| MDC kya hai? | Mapped Diagnostic Context — trace correlation. |
| PCI-DSS retention? | 12 months, 3 months immediately available. |
| SIEM kya hai? | Security Information & Event Management. |
| Kya log nahi karna? | Passwords, tokens, PII. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Brute force detect | SIEM correlation + alert |
| Compliance audit | Audit trail + retention |
| Incident forensics | Trace ID + timeline |
| PII protection | Masking + redaction |
| Log loss | Centralized + immutable |

---

## 12. Key Takeaways

```text
1. Security logging = when, where, who, what.

2. Structured JSON + MDC (trace).

3. Audit trail = immutable + separate.

4. PCI-DSS = 12 months retention.

5. SIEM = correlation + alerts.

6. Sensitive data = kabhi nahi.

7. Masking = PII protection.

8. Centralized = container loss prevent.

9. Real-time alerting = SOC.

10. Compliance = GDPR + PCI-DSS.
```

---

## 13. Next Part Preview

**Part 29: Metrics & Alerting (Prometheus/Grafana)**

Isme hum cover karenge:

```text
- Metrics deep dive
- Micrometer + Prometheus
- Security metrics
- Grafana dashboards
- Alerting rules
```