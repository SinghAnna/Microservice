# Part 35: Microservices Security — Incident Response & Forensics

> Incident response ka core — **NIST 4-phase lifecycle (Preparation → Detection/Analysis → Containment/Eradication/Recovery → Post-Incident)** — blameless postmortem + chain of custody + runbook-driven response. [1979][1981][1983][1988]

---

## 1. Incident Response Mental Model

```text
Incident = Security breach / attack / compromise

Wrong:
Panic -> Ad-hoc fix -> Repeat

Right:
Playbook -> Contain -> Eradicate -> Learn
```

### Incident Types

| Type | Example |
|---|---|
| Data breach | PII leak |
| Ransomware | Encryption + ransom |
| Account compromise | Stolen credentials |
| DDoS | Service unavailability |
| Insider threat | Malicious employee |

Incident types — breach, ransomware, compromise, DDoS, insider. [1981][1985][1989]

---

## 2. NIST IR Lifecycle

```text
Phase 1: Preparation
Phase 2: Detection & Analysis
Phase 3: Containment, Eradication & Recovery
Phase 4: Post-Incident Activity
```

### Phase 1 — Preparation

```text
- IR team + roles
- Playbooks / runbooks
- SIEM + EDR deploy
- Tabletop exercises
- Contact list + escalation
```

Preparation — team, playbooks, tools, tabletop drills. [1979][1981][1984]

### Phase 2 — Detection & Analysis

```text
- SIEM alert
- Triage (true positive?)
- Scope (kitne systems?)
- Severity classify
- Evidence collect
```

Detection — SIEM alert → triage → scope → severity → evidence. [1981][1989]

### Phase 3 — Containment, Eradication & Recovery

```text
Short-term: Isolate system
Long-term: Patch + disable account
Eradication: Malware remove
Recovery: Restore + validate
```

Containment → eradication → recovery (short-term isolate, long-term patch, restore). [1979][1981][1987]

### Phase 4 — Post-Incident

```text
- Root cause analysis
- Lessons learned
- Playbook update
- Control improvement
```

Post-incident — RCA + lessons learned + playbook update. [1979][1981][1988]

---

## 3. Forensics (Evidence Collection)

```text
Order of Volatility:
1. Memory (RAM)
2. Network connections
3. Disk
4. Backups
```

### Chain of Custody

```text
- Who collected
- When collected
- How collected
- Hash (integrity)
- Secure storage
```

Chain of custody — who/when/how + hash + secure storage. [1977][1984][1987]

### Evidence Collection Steps

```bash
# Memory dump (volatile first)
volatility -f memory.dmp --profile=Linux

# Disk image
dd if=/dev/sda of=disk.img bs=4M status=progress

# Hash (integrity)
sha256sum disk.img > disk.img.sha256
```

Volatile first (memory) → disk → hash → secure store. [1984][1987][1991]

### Forensics Tools

| Tool | Purpose |
|---|---|
| Volatility | Memory analysis |
| Autopsy | Disk forensics |
| Wireshark | Network capture |
| Velociraptor | Endpoint forensics |

Volatility (memory), Autopsy (disk), Wireshark (network), Velociraptor (endpoint). [1984][1987][1991]

---

## 4. SIEM-Based Investigation

```text
SIEM query:
"Last 24h me user X ne kya kiya?"

Query:
user_id="X" AND timestamp > now-24h
```

### Investigation Queries

```promql
# Auth failures by IP
sum by (source_ip) (rate(security_auth_failures_total[5m]))

# Data access by user
sum by (user_id) (rate(data_access_total[5m]))

# Privilege escalation
sum(rate(privilege_escalation_total[5m]))
```

SIEM — log correlation + timeline reconstruction. [1984][1985][1989]

### Timeline Reconstruction

```text
1. Initial access (login anomaly)
2. Lateral movement (cross-service)
3. Privilege escalation
4. Data exfiltration
5. Persistence (backdoor)
```

Kill chain — initial access → lateral → escalation → exfil → persistence. [1984][1989]

---

## 5. Runbook Example

```markdown
# Runbook: Brute Force Attack

## Trigger
- 5+ auth failures/min from same IP

## Steps
1. Verify (SIEM query)
2. Block IP (WAF/firewall)
3. Notify SOC
4. Monitor (next 24h)
5. Post-incident review
```

Runbook — trigger + steps + escalation. [1986][1990]

### Runbook Template

```markdown
# Runbook: <Scenario>

## Trigger
<Alert condition>

## Impact
<What's affected>

## Steps
1. <Action>
2. <Action>

## Escalation
<Who to contact>

## Recovery
<Restore steps>
```

Runbook — on-call engineer bina senior escalation ke use kar sake. [1986][1990]

---

## 6. Blameless Postmortem

```text
Blameless = "Kya hua" not "Kisne kiya"
```

### Postmortem Template

```markdown
# Postmortem: <Title>

## Summary
<2-3 sentences>

## Impact
- Users affected: X
- Duration: Y
- Data loss: None/Yes

## Timeline
| Time | Event |
|---|---|

## Root Cause
<Why>

## Contributing Factors
<What enabled it>

## Action Items
| Action | Owner | Due |
|---|---|---|

## Lessons Learned
<What to improve>
```

Blameless — systemic cause, not individual blame; specific + verifiable action items. [1980][1983][1988]

### Action Item Rules

```text
- Specific + verifiable
- One owner (person, not team)
- Due date
- Tracked in backlog
- Prioritized (prevent/detect/mitigate)
```

Action items — specific, one owner, due date, backlog-tracked. [1988]

---

## 7. Complete Flow

```text
1. SIEM alert (brute force)
2. Triage (true positive?)
3. Evidence collect (memory + logs)
4. Contain (block IP)
5. Eradicate (remove backdoor)
6. Recover (restore + validate)
7. Postmortem (blameless)
8. Improve (playbook + controls)
```

---

## 8. Real-World Patterns

| Pattern | Use Case |
|---|---|
| SIEM + SOAR | Automated response |
| EDR | Endpoint telemetry |
| Tabletop exercise | Readiness |
| Blameless postmortem | Learning culture |
| Runbook library | On-call empowerment |

### Decision Framework

```text
High severity? -> Immediate containment + exec notify
Low severity? -> Standard playbook
Recurring? -> Automation (SOAR)
Complex? -> External forensics
```

---

## 9. Common Gotchas

### Gotcha 1 — No Preparation

```text
Problem:
Incident pe panic

Fix:
Playbooks + tabletop drills
```

### Gotcha 2 — Evidence Contaminated

```text
Problem:
Chain of custody break

Fix:
Volatile first + hash + secure store
```

### Gotcha 3 — Blame Culture

```text
Problem:
Log hiding

Fix:
Blameless postmortem
```

### Gotcha 4 — No Action Items

```text
Problem:
Repeat incident

Fix:
Specific + tracked action items
```

---

## 10. Interview Questions

| Question | Answer |
|---|---|
| NIST IR lifecycle? | 4 phases (Prep → Detect → Contain → Post). |
| Chain of custody? | Evidence integrity (who/when/how + hash). |
| Order of volatility? | Memory → network → disk → backup. |
| Blameless postmortem? | Systemic cause, not individual blame. |
| Runbook kya? | Step-by-step recovery procedure. |
| SIEM role? | Log correlation + timeline. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Brute force | SIEM → block IP → monitor |
| Data breach | Forensics → notify → postmortem |
| Ransomware | Isolate → eradicate → restore |
| Insider threat | Audit log → investigate → action |
| Repeat incident | Postmortem → action items |

---

## 11. Key Takeaways

```text
1. NIST = 4-phase lifecycle.

2. Preparation = playbooks + drills.

3. Detection = SIEM + triage.

4. Containment = isolate first.

5. Eradication = remove threat.

6. Recovery = restore + validate.

7. Forensics = volatile first + hash.

8. Chain of custody = integrity.

9. Blameless = learning culture.

10. Action items = specific + tracked.
```

---

## 12. Next Part Preview

**Part 36: Compliance (GDPR/PCI-DSS/SOC2)**

Isme hum cover karenge:

```text
- Compliance deep dive
- GDPR (data protection)
- PCI-DSS (payment)
- SOC2 (trust services)
- Audit preparation
```