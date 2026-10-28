# Part 45: Microservices Security — Security Roadmap & Maturity Model

> Security roadmap ka core — **OWASP SAMM (5 functions, 15 practices, 3 levels) + risk-based prioritization (likelihood × impact) + phased roadmap (Quick Wins → Foundational → Efficient → Optimized)** — executive reporting me business impact + risk reduction, sirf technical metrics nahi. [2122][2123][2124][2129]

---

## 1. Security Roadmap Mental Model

```text
Roadmap = "Kahan se kahan tak" (phased plan)
Maturity = "Abhi kahan hain" (current state)
Prioritization = "Pehle kya" (risk-based)

Flow:
Assess (maturity) -> Prioritize (risk) -> Plan (phases) -> Execute -> Measure
```

### Roadmap Components

| Component | Purpose |
|---|---|
| Maturity assessment | Current state |
| Gap analysis | Target vs actual |
| Prioritization | Risk-based order |
| Phased plan | Quick wins → advanced |
| Metrics | Progress measure |

Roadmap — assess → gap → prioritize → plan → execute → measure. [2122][2126][2129]

---

## 2. OWASP SAMM Maturity Model

```text
SAMM = Software Assurance Maturity Model
= 5 Business Functions × 3 Practices × 3 Levels
```

### SAMM Structure

| Function | Practices |
|---|---|
| Governance | Strategy & Metrics, Policy & Compliance, Education & Guidance |
| Design | Threat Assessment, Security Requirements, Secure Architecture |
| Implementation | Secure Build, Secure Deployment, Defect Management |
| Verification | Architecture Assessment, Requirements-driven Testing, Security Testing |
| Operations | Incident Management, Environment Management, Operational Management |

SAMM — 5 functions, 15 practices, 3 levels each (0-3). [2122][2123][2133]

### SAMM Maturity Levels

| Level | Description |
|---|---|
| 0 | Not performed |
| 1 | Initial / ad-hoc |
| 2 | Defined / structured |
| 3 | Measured / optimized |

SAMM levels — 0 (not performed) → 1 (initial) → 2 (defined) → 3 (measured). [2122][2133]

### SAMM Assessment

```text
1. Per practice: 0-3 score
2. Function score: Average of practices
3. Overall: Weighted average
4. Gap: Target - current
5. Roadmap: Gap-based
```

SAMM assessment — per-practice score → function → overall → gap → roadmap. [2122][2133]

### SAMM Example

```text
Governance:
- Strategy & Metrics: 1
- Policy & Compliance: 2
- Education & Guidance: 1
Function Score: 1.33

Design:
- Threat Assessment: 0
- Security Requirements: 1
- Secure Architecture: 1
Function Score: 0.67

Priority: Design (lowest score)
```

SAMM example — per-function score → lowest = priority. [2122][2133]

---

## 3. Risk-Based Prioritization

```text
Risk = Likelihood × Impact
```

### Prioritization Matrix

|  | Low Impact | Medium Impact | High Impact |
|---|---|---|---|
| High Likelihood | Medium | High | Critical |
| Medium Likelihood | Low | Medium | High |
| Low Likelihood | Low | Low | Medium |

Risk matrix — likelihood × impact → priority. [2124][2128][2131]

### Prioritization Factors

```text
1. Likelihood (threat + vulnerability)
2. Impact (financial + reputational + legal)
3. Velocity (how fast exploit)
4. Control effectiveness (existing controls)
5. Manageability (fix difficulty)
```

Factors — likelihood, impact, velocity, control effectiveness, manageability. [2124][2125][2129]

### Risk Scoring

```text
RPN = Likelihood × Impact
(1-5 scale each, 1-25 total)

Critical: 20-25
High: 12-19
Medium: 6-11
Low: 1-5
```

RPN — likelihood × impact (1-25). [2125][2128]

### Business Context

```text
Technical severity (CVSS) ≠ Business risk

Business context:
- Asset criticality
- Data sensitivity
- Compliance exposure
- Financial impact
```

Business context — CVSS ≠ business risk, asset + data + compliance + financial. [2129][2131]

---

## 4. Phased Roadmap

```text
Phase 1: Quick Wins (Crawl)
Phase 2: Foundational (Walk)
Phase 3: Efficient (Run)
Phase 4: Optimized (Excel)
```

### Phase 1 — Quick Wins (0-3 months)

```text
- MFA enable
- Secrets scan (pre-commit)
- Basic logging + alerting
- Critical patching
- Backup + restore test
```

Quick wins — MFA, secrets scan, logging, patching, backup. [2122][2130]

### Phase 2 — Foundational (3-9 months)

```text
- SAST + SCA in CI
- Container scanning
- RBAC + least privilege
- Encryption at rest + transit
- IR plan + runbooks
```

Foundational — SAST/SCA, container scan, RBAC, encryption, IR. [2122][2130]

### Phase 3 — Efficient (9-18 months)

```text
- DAST + fuzzing
- Zero trust (mTLS + SPIFFE)
- SOAR + UEBA
- Compliance framework (SOC2/PCI)
- Security champions
```

Efficient — DAST, fuzzing, zero trust, SOAR/UEBA, compliance, champions. [2122][2130]

### Phase 4 — Optimized (18+ months)

```text
- Threat hunting
- Purple team
- ML-based detection
- Continuous compliance
- Security culture (metrics-driven)
```

Optimized — hunting, purple team, ML, continuous compliance, culture. [2122][2130]

---

## 5. Security Budget

```text
Budget = Risk-based allocation
```

### Budget Allocation

| Category | % | Examples |
|---|---|---|
| People | 40% | Security engineers, champions |
| Tools | 30% | SIEM, SAST, WAF |
| Services | 20% | Pentest, audit, training |
| Contingency | 10% | Incident response |

Budget — people 40%, tools 30%, services 20%, contingency 10%. [2129]

### Budget Justification

```text
ROI = Risk reduction / Cost
= (Expected loss - Residual loss) / Cost

Example:
- Risk: $1M expected loss
- Control cost: $100K
- Residual: $200K
- ROI: ($1M - $200K) / $100K = 8x
```

ROI — (expected loss - residual) / cost. [2129]

---

## 6. Executive Reporting

```text
Executive = Business language (not technical)
```

### Executive Dashboard

```text
1. Risk posture (overall score)
2. Top 5 risks (business impact)
3. Trend (improving / declining)
4. Compliance status
5. Budget utilization
6. Key wins
```

Executive dashboard — risk posture, top risks, trend, compliance, budget, wins. [2129]

### Executive Metrics

| Metric | Business Language |
|---|---|
| MTTD | "Detection speed" |
| MTTR | "Response speed" |
| Risk score | "Overall exposure" |
| Compliance % | "Audit readiness" |
| Incident count | "Security incidents" |

Executive metrics — business language (detection speed, exposure, readiness). [2129][2131]

### Reporting Cadence

```text
- Weekly: Ops (SOC metrics)
- Monthly: Management (KPIs)
- Quarterly: Executive (risk + budget)
- Annual: Board (strategy + ROI)
```

Cadence — weekly ops, monthly mgmt, quarterly exec, annual board. [2129]

---

## 7. Complete Flow

```text
1. SAMM assessment (current maturity)
2. Gap analysis (target vs actual)
3. Risk register (likelihood × impact)
4. Prioritize (RPN + business context)
5. Phased roadmap (quick wins → optimized)
6. Budget (risk-based)
7. Execute (sprints)
8. Measure (metrics)
9. Report (executive)
10. Iterate (quarterly)
```

---

## 8. Real-World Patterns

| Pattern | Use Case |
|---|---|
| SAMM | Software security maturity |
| Risk matrix | Prioritization |
| Phased roadmap | Incremental progress |
| ROI-based budget | Justification |
| Executive dashboard | Board reporting |

### Decision Framework

```text
Start? -> Quick wins + SAMM baseline
Scale? -> Foundational + champions
Mature? -> Efficient + zero trust
Optimize? -> Hunting + ML + culture
```

---

## 9. Common Gotchas

### Gotcha 1 — Big Bang

```text
Problem:
Everything at once, fail

Fix:
Phased (quick wins first)
```

### Gotcha 2 — Technical-Only Metrics

```text
Problem:
Business value unclear

Fix:
Business impact + risk reduction
```

### Gotcha 3 — No Risk Prioritization

```text
Problem:
Resources wasted

Fix:
RPN + business context
```

### Gotcha 4 — Static Roadmap

```text
Problem:
Outdated plan

Fix:
Quarterly review + iterate
```

---

## 10. Interview Questions

| Question | Answer |
|---|---|
| SAMM kya? | Software Assurance Maturity Model (5 functions, 15 practices). |
| SAMM levels? | 0-3 (not performed → measured). |
| Risk prioritization? | Likelihood × impact. |
| RPN? | Risk Priority Number. |
| Phased roadmap? | Quick wins → foundational → efficient → optimized. |
| Executive reporting? | Business language + risk + ROI. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Start | Quick wins + SAMM baseline |
| Prioritize | RPN + business context |
| Budget | ROI-based |
| Board report | Business language + risk |
| Stagnation | Quarterly review + iterate |

---

## 11. Key Takeaways

```text
1. Roadmap = assess → prioritize → plan → execute.

2. SAMM = 5 functions, 15 practices, 3 levels.

3. Risk = likelihood × impact.

4. RPN = 1-25 scale.

5. Phases = quick wins → optimized.

6. Budget = risk-based (people 40%).

7. ROI = risk reduction / cost.

8. Executive = business language.

9. Metrics = MTTD/MTTR + risk score.

10. Iterate = quarterly.
```

---

## 12. Next Part Preview

**Part 46: Security Case Studies**

Isme hum cover karenge:

```text
- Real-world security incidents
- Case study analysis
- Lessons learned
- Best practices derived
- Interview scenarios
```