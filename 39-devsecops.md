# Part 39: Microservices Security — Security in CI/CD (DevSecOps)

> DevSecOps ka core — **shift-left (security early, cheap fix) + automated gates (SAST/SCA/container/DAST) + security champions (dev-security bridge)** — har pipeline stage pe security check, CRITICAL/HIGH pe block. [2037][2041][2042][2043]

---

## 1. DevSecOps Mental Model

```text
Traditional:
Code -> Build -> Deploy -> Security check (late)
= Expensive fix

DevSecOps:
Code -> Security check -> Build -> Security check -> Deploy
= Early fix (cheap)
```

### Shift-Left Principle

```text
Fix cost:
Design: 1x
Code: 10x
Test: 100x
Production: 1000x
```

Shift-left — security early = cheap fix, late = expensive. [2037][2043][2046]

### DevSecOps = Everyone Responsible

```text
- Developer: Secure code
- Security: Enable + coach
- Ops: Secure infra
- Everyone: Security culture
```

DevSecOps — security everyone's responsibility, not just security team. [2045][2039]

---

## 2. Security Gates

```text
Gate = Checkpoint (pass/fail)
```

### Gate Types

| Gate | Stage | Tool |
|---|---|---|
| Secrets scan | Pre-commit | gitleaks |
| SAST | PR | Semgrep |
| SCA | PR | Trivy/Snyk |
| Container scan | Build | Trivy |
| IaC scan | Build | Checkov |
| DAST | Staging | ZAP |
| Policy check | Deploy | OPA/Kyverno |

Security gates — secrets, SAST, SCA, container, IaC, DAST, policy. [2041][2042][2044]

### Gate Policy

```text
CRITICAL = Block
HIGH = Block (or manual approval)
MEDIUM = Warn
LOW = Info
```

Severity-based — CRITICAL/HIGH block, MEDIUM warn. [2042][2043][2044]

### False Positive Management

```text
- Baseline (known issues)
- Suppression (ticket + expiry)
- Tuning (rules refine)
```

False positives — baseline + suppression (ticket ref + expiry) + tuning. [2042][2043]

---

## 3. Complete CI/CD Pipeline

```text
1. IDE (pre-commit: secrets + lint)
2. PR (SAST + SCA)
3. Build (container scan + IaC scan)
4. Registry (SBOM + sign)
5. Staging (DAST)
6. Deploy (policy check)
7. Runtime (monitoring)
```

### GitHub Actions Example

```yaml
name: DevSecOps Pipeline
on: [pull_request, push]

jobs:
  secrets:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: gitleaks/gitleaks-action@v2

  sast:
    runs-on: ubuntu-latest
    container: semgrep/semgrep
    steps:
      - uses: actions/checkout@v4
      - run: semgrep scan --config=auto --error --severity=ERROR .

  sca:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: trivy fs --exit-code 1 --severity CRITICAL,HIGH .

  container:
    needs: [sast, sca]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t app:${{ github.sha }} .
      - run: trivy image --exit-code 1 --severity CRITICAL app:${{ github.sha }}

  dast:
    needs: [container]
    runs-on: ubuntu-latest
    steps:
      - run: docker-compose up -d
      - uses: zaproxy/action-api-scan@v0.1.0
        with:
          target: 'http://localhost:8080/api/openapi.json'
```

Full pipeline — secrets → SAST → SCA → container → DAST. [2036][2041][2042]

---

## 4. Security Champions

```text
Champion = Dev team me security focal point
Bridge = Dev <-> Security
```

### Champion Roles

```text
- Evangelize security
- Threat modeling
- Secure code review
- Tool adoption
- CTF / training
- Vulnerability management
```

Champion — evangelize, threat model, code review, tool adoption, CTF. [2035][2038][2039]

### Champion Program

```text
1. Nominate (not assign)
2. 20% time allocation
3. Training + support
4. Communication channel
5. Periodic meetings
6. Management buy-in
```

Champion program — nominate, 20% time, training, channel, meetings, buy-in. [2039][2045]

### Champion Benefits

```text
- Insider view (app knowledge)
- Faster remediation
- Security culture
- Scalable security
```

Champion — insider view, faster remediation, culture, scalable. [2038][2039][2045]

---

## 5. Pipeline Security (Meta-Security)

```text
Pipeline bhi secure hona chahiye
```

### Pipeline Threats

```text
- Malicious PR
- Compromised CI
- Secret leak
- Dependency confusion
- Artifact tampering
```

Pipeline threats — malicious PR, CI compromise, secret leak, dependency confusion, artifact tampering. [2036][2048]

### Pipeline Hardening

```text
- RBAC + MFA (pipeline access)
- Least privilege (IAM)
- Secrets (Vault / Parameter Store)
- Branch protection (PR + review)
- Signed commits
- Artifact signing (Cosign)
```

Pipeline hardening — RBAC, MFA, least privilege, secrets, branch protection, signing. [2036][2048]

### Branch Protection

```text
- PR required
- 2 approvals
- Status checks (SAST/SCA pass)
- Signed commits
- No direct push to main
```

Branch protection — PR + approvals + status checks + signed commits. [2041][2048]

---

## 6. Complete Flow

```text
1. Dev commit
2. Pre-commit: secrets scan
3. PR: SAST + SCA (gate)
4. Merge (if pass)
5. Build: container scan + IaC scan
6. Registry: SBOM + sign
7. Staging: DAST
8. Deploy: policy check (OPA)
9. Runtime: monitoring
```

---

## 7. Real-World Patterns

| Pattern | Use Case |
|---|---|
| Pre-commit hooks | Fast feedback |
| PR gates | SAST + SCA |
| Staging DAST | Runtime validation |
| Security champions | Scale security |
| Policy-as-code | Consistent enforcement |

### Decision Framework

```text
Fast feedback? -> Pre-commit + PR gates
Runtime validation? -> Staging DAST
Scale? -> Security champions
Consistency? -> Policy-as-code (OPA)
```

---

## 8. Common Gotchas

### Gotcha 1 — Security at End

```text
Problem:
Late fix, expensive

Fix:
Shift-left (gates early)
```

### Gotcha 2 — Too Many Gates

```text
Problem:
Dev velocity kill

Fix:
Start essential, add gradually
```

### Gotcha 3 — False Positive Overload

```text
Problem:
Dev ignore karta hai

Fix:
Tuning + baseline + suppression
```

### Gotcha 4 — Pipeline Itself Insecure

```text
Problem:
CI compromise

Fix:
RBAC + MFA + secrets + signing
```

---

## 9. Interview Questions

| Question | Answer |
|---|---|
| DevSecOps kya? | Security in DevOps pipeline. |
| Shift-left kya? | Security early (cheap fix). |
| Security gate? | Checkpoint (pass/fail). |
| Security champion? | Dev-security bridge. |
| Pipeline hardening? | RBAC + MFA + secrets + signing. |
| False positive handling? | Baseline + suppression + tuning. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Fast feedback | Pre-commit + PR gates |
| Runtime validation | Staging DAST |
| Scale security | Champions |
| Consistent policy | OPA / policy-as-code |
| Pipeline compromise | RBAC + MFA + signing |

---

## 10. Key Takeaways

```text
1. DevSecOps = security in pipeline.

2. Shift-left = early fix (cheap).

3. Gates = secrets, SAST, SCA, container, DAST.

4. Severity = CRITICAL/HIGH block.

5. False positives = tune + baseline.

6. Champions = dev-security bridge.

7. Pipeline = bhi secure (meta-security).

8. Policy-as-code = consistency.

9. Phased = start essential.

10. Culture = everyone responsible.
```

---

## 11. Next Part Preview

**Part 40: Security Architecture Review**

Isme hum cover karenge:

```text
- Security architecture review
- Threat modeling deep dive
- Security design patterns
- Architecture decision records
- Security review checklist
```