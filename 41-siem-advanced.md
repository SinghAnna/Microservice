# Part 41: Microservices Security — Security Monitoring & SIEM (Advanced)

> Advanced security monitoring ka core — **SIEM (log correlation) + UEBA (behavior baseline + anomaly) + SOAR (automated response) + Threat Hunting (proactive)** — MTTD/MTTR metrics se effectiveness measure, rule-based + behavior-based detection combine. [2063][2065][2066][2072]

---

## 1. Advanced Monitoring Mental Model

```text
SIEM = Log correlation (rule-based)
UEBA = Behavior anomaly (ML-based)
SOAR = Automated response
Threat Hunting = Proactive search

Combined:
SIEM (what) + UEBA (who/how) + SOAR (respond) + Hunting (find unknown)
```

### Detection Maturity

| Level | Capability |
|---|---|
| Basic | Log collection + alerts |
| Intermediate | Correlation + dashboards |
| Advanced | UEBA + SOAR |
| Expert | Threat hunting + ML |

Detection maturity — logs → correlation → UEBA/SOAR → hunting/ML. [2065][2069][2072]

---

## 2. SIEM Deep Dive

```text
SIEM = Security Information & Event Management
= Log aggregation + correlation + alerting
```

### SIEM Architecture

```text
Data Sources -> Collectors -> SIEM Engine -> Dashboards/Alerts
   |                                    |
   +--> Rules Engine                    +--> Case Management
   +--> Correlation                     +--> Reporting
```

### SIEM Use Cases

| Use Case | Detection |
|---|---|
| Brute force | 5+ failures/min/IP |
| Credential stuffing | Many IPs, same user |
| Privilege escalation | Admin action anomaly |
| Data exfiltration | Unusual data access volume |
| Lateral movement | Cross-service anomaly |

SIEM — correlation rules + real-time alerts + case management. [2065][2072]

### Correlation Rule Example

```yaml
rule: brute_force
condition: |
  count(security_auth_failures_total) by (source_ip) > 5
  within 1 minute
action: alert + block IP
severity: HIGH
```

Correlation — count + time window + action. [2065][2072]

---

## 3. UEBA (User & Entity Behavior Analytics)

```text
UEBA = Behavior baseline + anomaly detection
= "Normal kya hai?" -> "Abnormal kya hai?"
```

### UEBA Flow

```text
1. Baseline build (normal behavior)
2. Current activity compare
3. Anomaly detect (deviation)
4. Risk score assign
5. Alert (if threshold)
```

UEBA — baseline → compare → anomaly → risk score → alert. [2063][2069][2072]

### UEBA Detects

```text
- Compromised credentials
- Insider threat
- Lateral movement
- Data exfiltration
- Privilege escalation
```

UEBA — compromised creds, insider, lateral, exfil, escalation. [2063][2069][2075]

### UEBA Example

```text
User X:
Normal: 9-5, office IP, 50 API calls/day
Anomaly: 2 AM, foreign IP, 5000 API calls
Risk Score: 95/100
Alert: Possible compromise
```

UEBA — time + location + volume deviation = risk score. [2063][2069][2072]

### Risk Scoring

```text
Risk Score = f(anomaly severity, entity criticality, context)
```

Risk score — anomaly + entity + context. [2063][2069][2075]

---

## 4. SOAR (Automated Response)

```text
SOAR = Security Orchestration, Automation, Response
= Playbook-driven automation
```

### SOAR Components

| Component | Purpose |
|---|---|
| Orchestration | Tool integration |
| Automation | Repetitive task automate |
| Response | Playbook execution |

SOAR — orchestration + automation + response. [2062][2066][2068]

### SOAR Playbook Example

```text
Trigger: Brute force alert
   |
   v
1. Enrich (IP reputation)
2. Check (known malicious?)
3. Block IP (WAF)
4. Notify SOC (Slack)
5. Create ticket
6. Monitor (24h)
```

Playbook — enrich → check → block → notify → ticket → monitor. [2062][2064][2066]

### Playbook Design

```text
- Trigger (alert condition)
- Enrichment (context add)
- Decision (if/else)
- Action (block/notify/ticket)
- Feedback (close/escalate)
```

Playbook — trigger → enrich → decide → act → feedback. [2064][2066][2068]

### Automation Benefits

```text
- MTTD reduce (faster detect)
- MTTR reduce (faster respond)
- Analyst time free (deep investigation)
- Consistency (no human error)
```

SOAR — MTTD/MTTR reduce, analyst time free, consistency. [2066][2068]

---

## 5. Threat Hunting

```text
Threat Hunting = Proactive search (alert nahi aaya, phir bhi dhoondo)
```

### Hunting Types

| Type | Approach |
|---|---|
| Hypothesis-driven | "Kya X ho raha hai?" |
| IOC-based | Known indicator search |
| TTP-based | MITRE technique search |
| Data-driven | Anomaly explore |

Hunting — hypothesis, IOC, TTP, data-driven. [2065][2067][2069]

### Hunting Hypothesis Example

```text
Hypothesis: "Lateral movement ho raha hai?"
Query: Cross-service access anomalies
Data: Service-to-service logs
Result: Unexpected service A -> B access
```

Hunting — hypothesis → query → data → result. [2067][2069]

### MITRE ATT&CK Mapping

```text
Tactic: Lateral Movement
Technique: T1021 (Remote Services)
Detection: Cross-service anomaly
```

MITRE ATT&CK — tactic + technique mapping. [2067][2069][2072]

### Hunting Loop

```text
1. Hypothesis
2. Data collect
3. Analyze
4. Findings
5. Detection rule create
6. Repeat
```

Hunting loop — hypothesis → data → analyze → findings → rule → repeat. [2065][2069]

---

## 6. Security Metrics

```text
Metrics = SOC effectiveness measure
```

### Key Metrics

| Metric | Definition | Target |
|---|---|---|
| MTTD | Detect time | <24h (advanced: 1-4h) |
| MTTR | Respond time | <4h (critical: <1h) |
| MTTC | Contain time | <72h |
| False Positive Rate | % false alarms | <30% |
| Coverage | MITRE technique % | >70% |

MTTD, MTTR, MTTC, false positive rate, coverage. [2065][2070][2071][2076]

### Metric Calculation

```text
MTTD = Σ(detect_time - occur_time) / incidents
MTTR = Σ(first_action - detect_time) / incidents
```

MTTD/MTTR formula — timeline-based. [2070][2071][2073]

### Metric by Severity

```text
Critical: MTTR < 1h
High: MTTR < 2h
Medium: MTTR < 4h
Low: MTTR < 8h
```

Severity-based MTTR targets. [2071][2076]

### Metric Pitfalls

```text
- MTTD/MTTR definition inconsistent
- False positive ignore
- Coverage gap ignore
- Vanity metrics (alert count)
```

Metric pitfalls — inconsistent definition, FP ignore, coverage gap, vanity. [2073][2076]

---

## 7. Complete Flow

```text
1. Log ingest (SIEM)
2. Rule correlation (known threats)
3. UEBA baseline + anomaly (unknown threats)
4. Alert -> SOAR playbook
5. Automated response (block/notify)
6. Threat hunting (proactive)
7. Metrics (MTTD/MTTR)
8. Improve (rules + playbooks)
```

---

## 8. Real-World Patterns

| Pattern | Use Case |
|---|---|
| SIEM + UEBA | Full detection |
| SOAR | Automated response |
| Threat hunting | Proactive |
| MITRE mapping | Coverage |
| Metrics | Effectiveness |

### Decision Framework

```text
Basic? -> SIEM + rules
Intermediate? -> UEBA
Advanced? -> SOAR
Expert? -> Hunting + ML
```

---

## 9. Common Gotchas

### Gotcha 1 — SIEM Only

```text
Problem:
Unknown threats miss

Fix:
UEBA + hunting
```

### Gotcha 2 — SOAR Over-Automation

```text
Problem:
False positive auto-block

Fix:
Human approval for high-impact
```

### Gotcha 3 — No Metrics

```text
Problem:
Effectiveness unknown

Fix:
MTTD/MTTR + coverage
```

### Gotcha 4 — Hunting Without Focus

```text
Problem:
Random search

Fix:
Hypothesis-driven + MITRE
```

---

## 10. Interview Questions

| Question | Answer |
|---|---|
| SIEM vs UEBA? | Rule-based vs behavior-based. |
| SOAR kya? | Orchestration + automation + response. |
| Threat hunting? | Proactive search. |
| MTTD? | Mean time to detect. |
| MTTR? | Mean time to respond. |
| MITRE ATT&CK? | Tactic + technique framework. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Known threat | SIEM rule |
| Unknown threat | UEBA + hunting |
| Repetitive response | SOAR playbook |
| Effectiveness measure | MTTD/MTTR |
| Coverage gap | MITRE mapping |

---

## 11. Key Takeaways

```text
1. SIEM = correlation (known).

2. UEBA = behavior (unknown).

3. SOAR = automated response.

4. Hunting = proactive.

5. MTTD/MTTR = effectiveness.

6. Risk score = anomaly + context.

7. Playbook = trigger → act → feedback.

8. MITRE = coverage mapping.

9. Severity-based = MTTR targets.

10. Combined = full detection.
```

---

## 12. Next Part Preview

**Part 42: Security Testing Automation (Advanced)**

Isme hum cover karenge:

```text
- Security testing automation deep dive
- Fuzzing
- Chaos engineering (security)
- Red team / Blue team
- Purple team
```