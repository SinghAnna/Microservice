# Part 34: Microservices Security — Security Testing (SAST/DAST/Pentest)

> Security testing ka core — **SAST (code, fast, pre-deploy) + DAST (running app, black-box) + SCA (dependencies) + Pentest (manual, business logic)** — layered approach, har layer alag vulnerability class pakadta hai. [1963][1964][1973][1975]

---

## 1. Security Testing Mental Model

```text
Layer 1: SAST  -> Code me bug? (white-box)
Layer 2: SCA   -> Dependencies me CVE? 
Layer 3: DAST  -> Running app me bug? (black-box)
Layer 4: Pentest -> Business logic / chain exploit? (manual)
```

### Testing Types

| Type | What | When | Speed |
|---|---|---|---|
| SAST | Source code | Dev/CI | Fast (min) |
| SCA | Dependencies | Dev/CI | Fast (min) |
| DAST | Running app | Staging/Pre-release | Slow (hours) |
| Pentest | Full app + logic | Pre-release/Annual | Days-weeks |

SAST (white-box, code) vs DAST (black-box, runtime) vs Pentest (manual, deep). [1964][1973][1975]

---

## 2. SAST (Static Analysis)

```text
SAST = Source code scan (execute nahi karta)
White-box = Code access hai
```

### Detects

```text
- SQL injection (string concat)
- XSS (unescaped output)
- Hardcoded secrets
- Insecure crypto
- Path traversal
- SSRF
```

SAST — SQL injection, XSS, hardcoded secrets, insecure crypto patterns. [1964][1969][1976]

### Semgrep Example

```bash
semgrep scan --config=auto --error --severity=ERROR .
```

`--error` — findings pe exit code 1 (CI block). [1969][1976]

### Custom Rule

```yaml
rules:
  - id: sql-injection-concat
    patterns:
      - pattern: $QUERY + $INPUT
    message: "SQL injection risk - use parameterized query"
    severity: ERROR
    languages: [java]
```

Custom Semgrep rule — pattern-based detection. [1969][1970]

### SAST Limitations

```text
- False positives high (30-50%)
- Business logic nahi samajhta
- Runtime-only bugs miss
```

SAST — high false positive rate, business logic coverage zero. [1973][1975]

---

## 3. DAST (Dynamic Analysis)

```text
DAST = Running app pe attack simulate
Black-box = Code access nahi
```

### Detects

```text
- SQL injection (runtime)
- XSS (runtime)
- Misconfigurations (headers, TLS)
- Auth bypass
- SSRF
- Insecure direct object reference
```

DAST — runtime injection, misconfig, auth bypass. [1963][1967][1968]

### OWASP ZAP

```bash
zap-baseline.py -t [https://staging.example.com](https://staging.example.com) -r report.html
```

ZAP baseline scan — passive, quick. [1967][1971][1974]

### ZAP API Scan (OpenAPI)

```bash
zap-api-scan.py \
  -t [https://staging.example.com/api/openapi.json](https://staging.example.com/api/openapi.json) \
  -f openapi \
  -r report.html
```

API scan — OpenAPI spec se endpoints fuzz. [1967][1971]

### DAST Advantage

```text
- Runtime context = fewer false positives
- Infra-level bugs (headers, TLS) catch
- SAST findings validate (exploitable?)
```

DAST — runtime context se false positives kam, SAST findings validate. [1964][1973]

---

## 4. SCA (Software Composition Analysis)

```text
SCA = Third-party dependencies CVE check
```

### Tools

| Tool | Purpose |
|---|---|
| Trivy | SBOM + CVE |
| Snyk | Dependency + fix PR |
| Dependabot | Auto-update PR |
| OWASP Dependency-Check | CVE match |

SCA — Trivy, Snyk, Dependabot, Dependency-Check. [1972][1974][1975]

### Trivy SBOM Scan

```bash
trivy sbom sbom.cdx.json
```

SBOM-based scan — dependency CVE correlation. [1919][1975]

---

## 5. Penetration Testing

```text
Pentest = Manual, expert-led, full attack simulation
```

### Pentest Phases (PTES)

```text
1. Scoping
2. Intelligence gathering
3. Threat modeling
4. Vulnerability analysis
5. Exploitation
6. Post-exploitation
7. Reporting
```

PTES — 7 phases (scoping → reporting). [1968]

### Pentest Finds (Automated Miss)

```text
- Business logic flaws
- IDOR (chained)
- Auth bypass (multi-step)
- Privilege escalation chains
- Social engineering
```

Pentest — business logic, IDOR chains, auth bypass — automated tools miss. [1965][1968][1973]

### DAST vs Pentest

| Aspect | DAST | Pentest |
|---|---|---|
| Method | Automated | Manual + automated |
| Scope | Broad, shallow | Narrow, deep |
| Speed | Fast, repeatable | Slow, point-in-time |
| Business logic | No | Yes |
| Cost | Low (after setup) | High |

DAST = broad/shallow/automated; Pentest = narrow/deep/manual. [1962][1965][1968]

---

## 6. CI/CD Integration

```text
Pipeline:
1. Commit -> Secrets scan + SAST (fast)
2. PR -> SCA + container scan
3. Staging deploy -> DAST (baseline)
4. Nightly -> DAST (full active)
5. Pre-release -> Pentest
```

### GitHub Actions — SAST

```yaml
name: SAST
on: [pull_request]
jobs:
  semgrep:
    runs-on: ubuntu-latest
    container:
      image: semgrep/semgrep
    steps:
      - uses: actions/checkout@v4
      - run: semgrep scan --config=auto --error --severity=ERROR .
```

Semgrep CI — `--error` pe fail. [1969][1976]

### GitHub Actions — DAST

```yaml
name: DAST
on: [push]
jobs:
  zap:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Start app
        run: docker-compose up -d
      - name: ZAP API Scan
        uses: zaproxy/action-api-scan@v0.1.0
        with:
          target: 'http://localhost:8080/api/openapi.json'
```

ZAP API scan — OpenAPI spec target. [1971][1974]

### Vulnerability SLA

```text
CRITICAL = 24h fix
HIGH = 7d
MEDIUM = 30d
LOW = Next release
```

SLA policy — severity-based fix timeline. [1974]

---

## 7. Complete Flow

```text
1. Dev writes code
2. Pre-commit: secrets scan
3. PR: SAST + SCA
4. Build: container scan
5. Staging: DAST baseline
6. Nightly: DAST full
7. Pre-release: Pentest
8. Production: monitoring + WAF
```

---

## 8. Real-World Patterns

| Pattern | Use Case |
|---|---|
| Semgrep | Fast SAST in CI |
| ZAP baseline | Staging DAST |
| ZAP API scan | OpenAPI-driven |
| Trivy | SCA + container |
| Pentest | Annual / major release |

### Decision Framework

```text
Fast feedback? -> SAST (Semgrep)
Runtime validation? -> DAST (ZAP)
Dependencies? -> SCA (Trivy/Snyk)
Deep assurance? -> Pentest
Full coverage? -> All 4 layers
```

---

## 9. Common Gotchas

### Gotcha 1 — SAST Only

```text
Problem:
Runtime bugs miss

Fix:
SAST + DAST dono
```

### Gotcha 2 — DAST on Production

```text
Problem:
Data corruption / DoS

Fix:
Staging environment only
```

### Gotcha 3 — False Positive Overload

```text
Problem:
Dev ignore karta hai

Fix:
Tuning + baseline + severity filter
```

### Gotcha 4 — No Business Logic Test

```text
Problem:
Automated tools miss

Fix:
Pentest + manual security review
```

---

## 10. Interview Questions

| Question | Answer |
|---|---|
| SAST kya hai? | Static code analysis (white-box). |
| DAST kya hai? | Dynamic runtime testing (black-box). |
| SCA kya hai? | Dependency CVE check. |
| Pentest vs DAST? | Manual deep vs automated broad. |
| ZAP kya hai? | OWASP DAST tool. |
| False positive kaunsa zyada? | SAST (30-50%). |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Fast CI feedback | Semgrep SAST |
| Runtime validation | ZAP DAST |
| Dependency CVE | Trivy/Snyk |
| Business logic | Pentest |
| Full coverage | SAST + SCA + DAST + Pentest |

---

## 11. Key Takeaways

```text
1. SAST = code, fast, white-box.

2. DAST = runtime, black-box.

3. SCA = dependencies CVE.

4. Pentest = manual, business logic.

5. Semgrep = fast SAST.

6. ZAP = DAST (baseline + API scan).

7. Trivy = SCA + container.

8. Layered = SAST + SCA + DAST + Pentest.

9. CI gate = severity-based.

10. Staging = DAST target (production nahi).
```

---

## 12. Next Part Preview

**Part 35: Incident Response & Forensics**

Isme hum cover karenge:

```text
- Incident response deep dive
- IR lifecycle (NIST)
- Forensics (log analysis)
- Post-mortem
- Runbooks
```