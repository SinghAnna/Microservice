# Part 36: Microservices Security — Compliance (GDPR/PCI-DSS/SOC2)

> Compliance ka core — **GDPR (data protection, 7 principles + 8 rights) + PCI-DSS (12 requirements, cardholder data) + SOC2 (5 Trust Services Criteria)** — har framework ka alag scope, lekin security controls overlap karte hain. [1993][1996][1997][2005]

---

## 1. Compliance Mental Model

```text
GDPR    = EU personal data protection
PCI-DSS = Cardholder data (payments)
SOC2    = Service organization controls (B2B trust)

Common:
- Encryption
- Access control
- Logging
- Incident response
```

### Framework Comparison

| Framework | Scope | Applies To |
|---|---|---|
| GDPR | Personal data (EU) | Any org processing EU data |
| PCI-DSS | Cardholder data | Payment processors |
| SOC2 | Service controls | SaaS / B2B vendors |

GDPR (EU personal data), PCI-DSS (cardholder data), SOC2 (service controls). [1993][1996][1997]

---

## 2. GDPR (Data Protection)

```text
GDPR = General Data Protection Regulation
Scope = EU citizens' personal data
```

### 7 Principles

```text
1. Lawfulness, fairness, transparency
2. Purpose limitation
3. Data minimisation
4. Accuracy
5. Storage limitation
6. Integrity & confidentiality
7. Accountability
```

GDPR 7 principles — lawfulness, purpose limitation, minimisation, accuracy, storage limitation, security, accountability. [1993][1994][1995]

### 8 Data Subject Rights

| Right | Meaning |
|---|---|
| Be informed | Kya data, kyun |
| Access | Data dekho |
| Rectification | Correct karo |
| Erasure | Delete karo (right to be forgotten) |
| Restriction | Processing limit |
| Portability | Data export |
| Object | Processing roko |
| Automated decisions | Human review |

8 rights — informed, access, rectify, erase, restrict, portability, object, automated decisions. [1992][1994]

### GDPR Implementation

```text
- Data mapping (kya data, kahan)
- Consent management
- Retention policy (auto-delete)
- DSR (Data Subject Request) workflow
- Privacy by design
- DPIA (high-risk processing)
```

GDPR implementation — data mapping, consent, retention, DSR workflow, privacy by design. [1992][1993][1994]

### DSR Workflow

```text
1. Request receive
2. Identity verify
3. Data locate (all systems)
4. Action (export/delete/correct)
5. Response (30 days)
```

DSR — 30 days response deadline. [1992][1994]

---

## 3. PCI-DSS (Payment Security)

```text
PCI-DSS = Payment Card Industry Data Security Standard
Scope = Cardholder Data Environment (CDE)
```

### 12 Requirements (6 Goals)

| Goal | Requirements |
|---|---|
| Build secure network | 1-2 |
| Protect cardholder data | 3-4 |
| Vulnerability management | 5-6 |
| Access control | 7-8 |
| Monitor + test | 9-11 |
| Policy | 12 |

PCI-DSS — 12 requirements, 6 goals. [1996][1998][2003]

### Key Requirements

```text
Req 1: Firewall + network segmentation
Req 3: Encrypt stored data (tokenize)
Req 4: TLS 1.2+ in transit
Req 7: Least privilege access
Req 8: MFA + unique IDs
Req 10: Log + monitor all access
Req 11: Pen test annually
Req 12: IR plan + security policy
```

Key PCI requirements — segmentation, encryption, least privilege, MFA, logging, pen test, IR. [1996][1999][2003]

### PCI Implementation

```text
- CDE segmentation (VLAN/firewall)
- Tokenization (PAN replace)
- TLS 1.2+ (all transit)
- MFA (all access)
- Log retention (12 months)
- Quarterly scans + annual pen test
```

PCI implementation — segmentation, tokenization, TLS, MFA, 12-month logs, annual pen test. [1999][2002][2003]

### Tokenization Example

```text
Card: 4111-1111-1111-1111
Token: tok_abc123 (vault me real PAN)
```

Tokenization — PAN replace with token, real value vault me. [1999][2002]

---

## 4. SOC2 (Service Organization Controls)

```text
SOC2 = Trust Services Criteria (AICPA)
Scope = SaaS / B2B service providers
```

### 5 Trust Services Criteria

| Category | Code | Criteria | Purpose |
|---|---|---|---|
| Security | CC | 33 | Unauthorized access protect |
| Availability | A | 3 | Uptime + recovery |
| Confidentiality | C | 2 | Confidential data protect |
| Processing Integrity | PI | 1 | Accurate processing |
| Privacy | P | 8 | PII protect |

SOC2 — 5 TSC, Security mandatory (33 criteria), baaki optional. [1997][2000][2004][2005]

### SOC2 Report Types

```text
Type I: Controls design (point in time)
Type II: Controls operating effectiveness (6-12 months)
```

Type I = design, Type II = operating effectiveness (6-12 months). [1997][2005]

### SOC2 Implementation

```text
- Access control (RBAC + MFA)
- Change management (PR + approval)
- Monitoring (SIEM + alerts)
- Incident response (playbook)
- Vendor management
- Backup + DR
```

SOC2 implementation — access, change mgmt, monitoring, IR, vendor, DR. [1997][2000][2005]

---

## 5. Overlapping Controls

```text
Common Controls (All 3):
- Encryption (at rest + transit)
- Access control (RBAC + MFA)
- Logging + monitoring
- Incident response
- Vendor management
- Data retention
```

### Control Mapping

| Control | GDPR | PCI-DSS | SOC2 |
|---|---|---|---|
| Encryption | Yes | Yes (Req 3,4) | Yes (CC) |
| MFA | Yes | Yes (Req 8) | Yes (CC) |
| Logging | Yes | Yes (Req 10) | Yes (CC) |
| IR plan | Yes | Yes (Req 12) | Yes (CC) |
| Retention | Yes | Yes (Req 3) | Yes (C) |

Overlapping controls — ek control, teen frameworks satisfy. [1993][1999][1997]

---

## 6. Audit Preparation

```text
1. Scope define (systems + data)
2. Gap assessment
3. Remediate gaps
4. Evidence collect (screenshots, logs, policies)
5. Audit (internal / external)
6. Report + attestation
```

### Evidence Examples

```text
- Access review screenshots
- MFA enablement proof
- Log retention config
- Pen test report
- IR plan + test results
- Vendor SOC2 reports
```

Audit prep — scope → gap → remediate → evidence → audit → report. [1997][2000][2005]

### Continuous Compliance

```text
- Automated evidence collection
- Policy-as-code (OPA/Kyverno)
- Continuous monitoring
- Quarterly access review
```

Continuous compliance — automated evidence + policy-as-code + monitoring. [1997][2005]

---

## 7. Complete Flow

```text
1. Framework select (GDPR/PCI/SOC2)
2. Scope define
3. Gap assessment
4. Implement controls
5. Evidence collect
6. Audit
7. Attestation
8. Continuous monitoring
```

---

## 8. Real-World Patterns

| Pattern | Use Case |
|---|---|
| GDPR + PCI | Fintech (EU) |
| SOC2 Type II | SaaS B2B |
| PCI + SOC2 | Payment SaaS |
| GDPR + SOC2 | Global SaaS |

### Decision Framework

```text
EU data? -> GDPR
Payments? -> PCI-DSS
B2B SaaS? -> SOC2 Type II
Multiple? -> Overlapping controls
```

---

## 9. Common Gotchas

### Gotcha 1 — GDPR = EU Only

```text
Problem:
Global data ignore

Fix:
Extraterritorial scope (EU citizens)
```

### Gotcha 2 — PCI = Only Payment Team

```text
Problem:
CDE scope unclear

Fix:
Data flow mapping
```

### Gotcha 3 — SOC2 = One-Time

```text
Problem:
Annual audit only

Fix:
Continuous monitoring + evidence
```

### Gotcha 4 — No Data Mapping

```text
Problem:
DSR fail (data locate nahi)

Fix:
Data catalog + lineage
```

---

## 10. Interview Questions

| Question | Answer |
|---|---|
| GDPR 7 principles? | Lawfulness, purpose, minimisation, accuracy, storage, security, accountability. |
| GDPR 8 rights? | Informed, access, rectify, erase, restrict, portability, object, automated. |
| PCI-DSS 12 requirements? | Network, data, vuln, access, monitor, policy. |
| PCI tokenization? | PAN replace with token. |
| SOC2 5 TSC? | Security, Availability, Confidentiality, PI, Privacy. |
| SOC2 Type I vs II? | Design vs operating effectiveness. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| EU user data | GDPR (DSR + retention) |
| Payment processing | PCI-DSS (segmentation + tokenization) |
| B2B SaaS | SOC2 Type II |
| Multi-framework | Overlapping controls |
| Audit prep | Evidence + continuous monitoring |

---

## 11. Key Takeaways

```text
1. GDPR = EU personal data.

2. PCI-DSS = cardholder data.

3. SOC2 = service controls (B2B).

4. GDPR = 7 principles + 8 rights.

5. PCI = 12 requirements, 6 goals.

6. SOC2 = 5 TSC, Security mandatory.

7. Overlap = encryption, access, logging, IR.

8. DSR = 30 days.

9. Tokenization = PAN replace.

10. Continuous = evidence + monitoring.
```

---

## 12. Next Part Preview

**Part 37: Zero Trust Architecture**

Isme hum cover karenge:

```text
- Zero trust deep dive
- Identity-centric security
- Micro-segmentation
- Continuous verification
- BeyondCorp model
```