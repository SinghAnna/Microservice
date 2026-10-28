# Part 40: Microservices Security — Security Architecture Review

> Security architecture review ka core — **STRIDE threat modeling (6 categories) + DFD (data flow + trust boundaries) + ADR (decision record) + security design patterns** — design phase me review, implementation se pehle. [2054][2056][2059]

---

## 1. Security Architecture Review Mental Model

```text
SAR = Security Architecture Review
When = Design phase (before code)
Goal = Find design flaws early
```

### SAR vs Threat Model

```text
Threat Model = "Kya galat ho sakta hai?"
SAR = "Controls sahi design hue?"

SAR includes:
- Threat model
- Control validation
- Compliance check
- Risk assessment
```

SAR — threat model + control validation + compliance + risk. [2054][2059]

### Review Scope

```text
1. Authentication model
2. Authorization model
3. Data flows + trust boundaries
4. Secrets + cryptography
5. Network segmentation
```

SAR scope — authN, authZ, data flows, secrets, network. [2054][2059]

---

## 2. STRIDE Threat Modeling

```text
S - Spoofing
T - Tampering
R - Repudiation
I - Information Disclosure
D - Denial of Service
E - Elevation of Privilege
```

### STRIDE Checklist

| Threat | Question | Control |
|---|---|---|
| Spoofing | Impersonation possible? | MFA, mTLS, strong authN |
| Tampering | Data modify? | Integrity (hash, signature) |
| Repudiation | Action deny? | Audit log (tamper-proof) |
| Info Disclosure | Data leak? | Encryption, authZ |
| DoS | Unavailable? | Rate limit, WAF |
| EoP | Privilege escalate? | Least privilege, RBAC |

STRIDE — 6 categories, har component + trust boundary pe evaluate. [2054][2057][2059]

### STRIDE Example (Order Service)

```text
Component: GET /api/orders/{id}

S: Token forge? -> JWT RS256 + validation
T: Order modify? -> Integrity check
R: Action deny? -> Audit log
I: Other user's order? -> BOLA guard
D: DoS? -> Rate limit
E: Admin access? -> RBAC + ownership
```

STRIDE per-endpoint — har category pe question + control. [2054][2057]

---

## 3. Data Flow Diagram (DFD)

```text
DFD = Data kahan se kahan jaata hai
Trust Boundary = Trust level change
```

### DFD Example

```text
[User] --(HTTPS)--> [Gateway] --(mTLS)--> [Order Service] --(mTLS)--> [DB]
   |                     |                        |                    |
   v                     v                        v                    v
Trust: External       Trust: DMZ             Trust: Internal       Trust: Data
```

DFD — components + flows + trust boundaries. [2059]

### Trust Boundary Rules

```text
Har boundary crossing pe:
- AuthN enforce
- Input validate
- Encrypt in transit
- AuthZ check
```

Trust boundary — har crossing pe authN + validate + encrypt + authZ. [2054][2059]

### DFD Checklist

```text
- All components listed
- All data flows mapped
- Trust boundaries marked
- Sensitive data tagged
- External dependencies shown
```

DFD checklist — components, flows, boundaries, sensitive data, dependencies. [2059]

---

## 4. Security Design Patterns

```text
Patterns = Reusable security solutions
```

### Core Patterns

| Pattern | Purpose |
|---|---|
| Defense in Depth | Layered controls |
| Least Privilege | Minimal access |
| Fail-Safe Defaults | Default deny |
| Complete Mediation | Every access check |
| Separation of Privilege | Multi-condition |
| Least Common Mechanism | Shared mechanism minimize |

Security patterns — defense in depth, least privilege, fail-safe, complete mediation, separation, least common mechanism. [2056][2050]

### Defense in Depth

```text
Layer 1: WAF
Layer 2: Gateway (authN + rate limit)
Layer 3: Service (authZ + validation)
Layer 4: Data (encryption)
```

Defense in depth — ek control fail, doosra protect. [2050][2056]

### Least Privilege Example

```text
WRONG:
Service -> db_owner

RIGHT:
Service -> SELECT on own tables only
```

Least privilege — service-specific credentials, scoped permissions. [2056][2050]

### Fail-Safe Defaults

```text
Default = Deny
Explicit allow only
```

Fail-safe — default deny, explicit allow. [2056]

---

## 5. Architecture Decision Records (ADR)

```text
ADR = Decision record (context + decision + consequences)
```

### ADR Template

```markdown
# ADR-001: Use OAuth2 + JWT for API AuthN

## Status
Accepted

## Context
Microservices need secure service-to-service + user authN.
Requirements: short-lived tokens, revocation, multi-client.

## Decision
OAuth2 Authorization Code + PKCE for user flows.
RS256 JWT (15 min) for API access.
Refresh token rotation.

## Consequences
+ Stateless validation
+ Revocation support
+ Multi-client
- Key management complexity
- Token size overhead

## Alternatives
- Session-based: stateful, scaling issue
- API key: no user context
```

ADR — title, status, context, decision, consequences, alternatives. [2049][2053][2055]

### ADR Best Practices

```text
- 1-2 pages max
- "Why" > "What"
- Alternatives document
- Version control (docs/adr/)
- Sequential numbering
- Security ADR = threat model + compliance mapping
```

ADR — short, why-focused, alternatives, version control, sequential. [2049][2053][2055][2060]

### Security ADR

```markdown
# ADR-002: Use Vault for Secrets

## Security Implications
- Threat model: Secret leak
- Compliance: PCI-DSS Req 3, SOC2 CC
- Review: Security team sign-off
```

Security ADR — threat model + compliance mapping + security sign-off. [2055][2060]

---

## 6. Review Checklist

```markdown
# Security Architecture Review Checklist

## Authentication
- [ ] MFA enabled?
- [ ] Strong authN (OAuth2/OIDC)?
- [ ] Short-lived tokens?
- [ ] Token revocation?

## Authorization
- [ ] Object-level authZ?
- [ ] Least privilege?
- [ ] RBAC/ABAC model?
- [ ] Default deny?

## Data Protection
- [ ] Encryption at rest?
- [ ] Encryption in transit (TLS 1.2+)?
- [ ] Data classification?
- [ ] Retention policy?

## Secrets
- [ ] Vault / K8s Secrets?
- [ ] No hardcoded secrets?
- [ ] Rotation policy?

## Network
- [ ] Segmentation?
- [ ] Default deny?
- [ ] mTLS service-to-service?

## Logging
- [ ] Security events logged?
- [ ] Audit trail (immutable)?
- [ ] SIEM integration?

## Incident Response
- [ ] IR plan?
- [ ] Runbooks?
- [ ] Contact list?
```

Checklist — authN, authZ, data, secrets, network, logging, IR. [2054][2059]

---

## 7. Complete Flow

```text
1. Design doc + DFD
2. STRIDE threat model
3. SAR checklist
4. Gaps identify
5. ADR (decisions)
6. Remediate
7. Sign-off
8. Implement
9. Validate (testing)
```

---

## 8. Real-World Patterns

| Pattern | Use Case |
|---|---|
| STRIDE workshop | New feature |
| DFD | Architecture doc |
| ADR | Major decision |
| Checklist | Consistent review |
| Threat register | Track + prioritize |

### Decision Framework

```text
New feature? -> STRIDE workshop
Major change? -> ADR + SAR
High-risk? -> External review
Recurring? -> Checklist + automation
```

---

## 9. Common Gotchas

### Gotcha 1 — Review at End

```text
Problem:
Late fix, expensive

Fix:
Design phase review
```

### Gotcha 2 — No DFD

```text
Problem:
Trust boundaries unclear

Fix:
DFD mandatory
```

### Gotcha 3 — No ADR

```text
Problem:
Decision context lost

Fix:
ADR for major decisions
```

### Gotcha 4 — Checklist Ignored

```text
Problem:
Inconsistent review

Fix:
Automated + mandatory
```

---

## 10. Interview Questions

| Question | Answer |
|---|---|
| SAR kya hai? | Security architecture review. |
| STRIDE? | Spoofing, Tampering, Repudiation, Info Disclosure, DoS, EoP. |
| DFD kya? | Data flow diagram + trust boundaries. |
| ADR kya? | Architecture decision record. |
| Defense in depth? | Layered controls. |
| Least privilege? | Minimal access. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| New feature | STRIDE workshop |
| Major decision | ADR |
| Architecture doc | DFD |
| Consistent review | Checklist |
| High-risk | External review |

---

## 11. Key Takeaways

```text
1. SAR = design phase review.

2. STRIDE = 6 threat categories.

3. DFD = data flow + trust boundaries.

4. ADR = decision record.

5. Defense in depth = layered.

6. Least privilege = minimal access.

7. Fail-safe = default deny.

8. Checklist = consistency.

9. Security ADR = threat + compliance.

10. Review = before implementation.
```

---

## 12. Next Part Preview

**Part 41: Security Monitoring & SIEM (Advanced)**

Isme hum cover karenge:

```text
- SIEM deep dive
- UEBA (behavior analytics)
- SOAR (automated response)
- Threat hunting
- Security metrics
```