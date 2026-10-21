# Part 15: Microservices Security — Request Validation & WAF at Gateway

> Request validation ka core — **allowlist validation (server-side) + WAF (signature/anomaly detection)** — OWASP CRS anomaly scoring (threshold 5) se SQLi/XSS/LFI block, lekin WAF bypass-able hai isliye parameterized queries + output encoding bhi zaroori. [1708][1712][1713][1718]

---

## 1. Request Validation Mental Model

```text
Request -> WAF (signature/anomaly) -> Gateway -> Service -> Validation -> Business Logic
              |                          |            |
              +--> Known attacks block   |            +--> Allowlist check
                                         +--> Schema validation
```

### Defense Layers

| Layer | Kya Rokta Hai |
|---|---|
| WAF | Known attack signatures |
| Gateway | Schema violations, size limits |
| Service | Business rule violations |
| DB | Parameterized queries (last line) |

WAF first line, validation deep defense — WAF bypass ho to bhi service safe. [1708][1712][1718]

---

## 2. Input Validation Principles

```text
1. Allowlist > Blocklist
2. Server-side validation (client-side bypass-able)
3. Type + length + range + format check
4. Canonicalize before validate
5. Fail closed (invalid = reject)
```

### Allowlist vs Blocklist

```text
Blocklist (bad):
Block: <script>, SELECT, ../
Attacker: <scr<script>ipt> (bypass)

Allowlist (good):
Allow: [a-zA-Z0-9]{1,50}
Baqi sab reject
```

Allowlist — sirf known-good patterns, baaki reject — OWASP recommended. [1708][1712]

---

## 3. Common Attack Patterns

### SQL Injection

```text
' OR '1'='1
'; DROP TABLE users;--
1' UNION SELECT password FROM users--
' AND SLEEP(5)--
```

### XSS

```text
<script>alert(1)</script>
<img src=x onerror=alert(1)>
javascript:alert(1)
<svg onload=alert(1)>
```

### Path Traversal

```text
../../etc/passwd
..\..\windows\system32
....//....//etc/passwd
%2e%2e%2f%2e%2e%2f
```

### Common Patterns

| Attack | Pattern |
|---|---|
| SQLi | `'`, `--`, `UNION`, `SLEEP()` |
| XSS | `<script>`, `onerror=`, `javascript:` |
| Path Traversal | `../`, `..\\`, `%2e%2e` |
| Command Injection | `;`, `&&`, `|`, backtick |
| SSRF | `http://169.254.169.254`, `file://` |

---

## 4. Spring Cloud Gateway Validation Filter

```java
@Component
public class RequestValidationFilter implements GlobalFilter, Ordered {

    private static final Pattern SQLI = Pattern.compile(
        "(?i)(union\\s+select|drop\\s+table|or\\s+1\\s*=\\s*1|sleep\\()"
    );
    private static final Pattern XSS = Pattern.compile(
        "(?i)(<script|onerror\\s*=|javascript:)"
    );
    private static final Pattern TRAVERSAL = Pattern.compile(
        "(\\.\\./|\\.\\\\|%2e%2e)"
    );

    @Override
    public Mono<Void> filter(
        ServerWebExchange exchange,
        GatewayFilterChain chain
    ) {
        String path = exchange.getRequest().getPath().value();
        String query = exchange.getRequest().getURI().getQuery();

        if (containsAttack(path) || containsAttack(query)) {
            return blocked(exchange, "Malicious pattern detected");
        }

        return chain.filter(exchange);
    }

    private boolean containsAttack(String input) {
        if (input == null) return false;
        return SQLI.matcher(input).find()
            || XSS.matcher(input).find()
            || TRAVERSAL.matcher(input).find();
    }

    private Mono<Void> blocked(ServerWebExchange exchange, String reason) {
        exchange.getResponse().setStatusCode(HttpStatus.FORBIDDEN);
        return exchange.getResponse().setComplete();
    }

    @Override
    public int getOrder() {
        return -200;  // WAF se baad, auth se pehle
    }
}
```

Custom `GlobalFilter` — signature-based validation, early in chain. [1714][1715][1716][1720]

---

## 5. Bean Validation (Service Layer)

```java
public class OrderRequest {

    @NotBlank
    @Size(min = 1, max = 100)
    @Pattern(regexp = "^[a-zA-Z0-9\\s-]+$")  // Allowlist
    private String productName;

    @NotNull
    @DecimalMin("0.01")
    @DecimalMax("100000.00")
    private BigDecimal amount;

    @Email
    @Size(max = 255)
    private String email;
}

@PostMapping("/api/orders")
public Order create(@Valid @RequestBody OrderRequest request) {
    // Validated
}
```

Bean Validation — type, length, range, format — allowlist regex. [1708]

---

## 6. OWASP CRS (ModSecurity)

```text
CRS = Core Rule Set
Anomaly Scoring:
- Critical: 5 points
- Error: 4 points
- Warning: 3 points
- Notice: 2 points
- Threshold: 5 (block)
```

### CRS Rule Categories

| Rule Range | Category |
|---|---|
| 920-921 | Protocol abuse |
| 930-931 | File inclusion (LFI/RFI) |
| 932 | RCE |
| 941 | XSS |
| 942 | SQL Injection |
| 943 | Session fixation |

### CRS Example

```text
Request: /api/products?id=1' UNION SELECT--
CRS Rule 942100 (libinjection) -> 5 points
Threshold 5 -> Block (403)
```

OWASP CRS — anomaly scoring, threshold 5 pe block. [1709][1713][1718]

### ModSecurity + CRS Setup

```text
Nginx/Envoy + ModSecurity + OWASP CRS
   |
   v
Spring Cloud Gateway
```

---

## 7. False Positives Tuning

```text
Problem:
Legit request block (false positive)

Solution:
1. Paranoia Level adjust (PL1 = low FP)
2. Rule exclusion (specific path)
3. Anomaly threshold increase
```

### Rule Exclusion

```text
SecRule REQUEST_URI "@beginsWith /api/search" \
  "id:1010,ctl:ruleRemoveByTag=attack-sqli"
```

False positives — paranoia level + rule exclusion + threshold tuning. [1709][1711]

---

## 8. Complete Flow

```text
1. Request -> WAF (CRS rules)
2. WAF -> Anomaly score
3. Score >= 5 -> 403 block
4. Score < 5 -> Gateway
5. Gateway -> Custom validation filter
6. Gateway -> Service
7. Service -> Bean validation
8. Service -> Parameterized query
9. Response
```

---

## 9. Real-World Rules

| Rule | Pattern | Action |
|---|---|---|
| SQLi | `UNION SELECT` | Block |
| XSS | `<script` | Block |
| Traversal | `../` | Block |
| SSRF | `169.254.169.254` | Block |
| Size | Body > 1MB | Block |
| Content-Type | Unexpected | Block |

---

## 10. Common Gotchas

### Gotcha 1 — WAF-Only Security

```text
Problem:
WAF bypass (encoding/obfuscation)

Fix:
WAF + validation + parameterized queries
```

### Gotcha 2 — Blocklist Validation

```text
Problem:
Bypass (obfuscation)

Fix:
Allowlist validation
```

### Gotcha 3 — False Positives

```text
Problem:
Legit traffic block

Fix:
Paranoia level + exclusions
```

### Gotcha 4 — Client-Side Validation Only

```text
Problem:
Bypass (direct API call)

Fix:
Server-side validation
```

---

## 11. Interview Questions

| Question | Answer |
|---|---|
| WAF kya hai? | Web Application Firewall — L7 attack filter. |
| Allowlist vs Blocklist? | Known-good allow vs known-bad block. |
| CRS kya hai? | OWASP Core Rule Set — anomaly scoring. |
| Anomaly scoring? | Points accumulate, threshold pe block. |
| False positive? | Legit request block. |
| WAF bypass? | Encoding/obfuscation — isliye layered defense. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| SQLi | WAF + parameterized queries |
| XSS | WAF + output encoding |
| Path traversal | WAF + path normalization |
| False positives | Paranoia level + exclusions |
| Defense in depth | WAF + Gateway + Service |

---

## 12. Key Takeaways

```text
1. WAF = first line (signature/anomaly).

2. Allowlist > blocklist.

3. Server-side validation mandatory.

4. CRS = anomaly scoring (threshold 5).

5. Parameterized queries = last line.

6. False positives = tuning.

7. WAF bypass-able = layered defense.

8. Bean validation = service layer.

9. Canonicalize before validate.

10. Fail closed.
```

---

## 13. Next Part Preview

**Part 16: Client Credentials Flow**

Isme hum cover karenge:

```text
- Client Credentials deep dive
- Service-to-service auth
- Token caching
- Scope-based authorization
- Real-world implementation
```