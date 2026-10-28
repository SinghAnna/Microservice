# Part 43: Microservices Security — Security Automation & Orchestration

> Security automation ka core — **SOAR (orchestration + automation + response) + Security as Code + Policy as Code (OPA/Kyverno) + Automated Remediation** — frequent + deterministic + low-blast-radius tasks automate, high-impact pe human approval. [2092][2094][2100][2102]

---

## 1. Security Automation Mental Model

```text
Manual:
Alert -> Analyst investigate -> Manual fix -> Ticket
= Slow, inconsistent

Automated:
Alert -> SOAR playbook -> Auto remediate -> Audit log
= Fast, consistent
```

### Automation Spectrum

```text
Manual -> Semi-Automated (ChatOps) -> Fully Automated (Self-healing)
```

Automation spectrum — manual → semi (ChatOps) → fully automated. [2093][2094]

### Automation Eligibility

```text
Automate if:
- Frequent (>5/month)
- Deterministic (clear path)
- Low blast radius
- Validatable
- Rollback possible

Human approval if:
- High impact (account disable, isolation)
- Ambiguous (unclear cause)
- Compliance-sensitive
```

Automation eligibility — frequent + deterministic + low blast radius + rollback. [2102][2096]

---

## 2. SOAR Advanced Patterns

```text
SOAR = Orchestration + Automation + Response
```

### Advanced Playbook Patterns

| Pattern | Use Case |
|---|---|
| Enrichment | Context add (IP rep, user info) |
| Conditional | Severity-based path |
| Parallel | Multi-tool simultaneous |
| Human-in-loop | Approval gate |
| Feedback | Close/escalate |
| Correlation | Cross-incident pattern |

Advanced patterns — enrichment, conditional, parallel, human-in-loop, feedback, correlation. [2094][2095][2096]

### Playbook Example (Full)

```text
Trigger: Malware detected (EDR)
   |
   v
1. Enrich (file hash + IP rep)
2. Conditional:
   - Known malicious? -> Isolate endpoint
   - Unknown? -> Sandbox + analyst review
3. Parallel:
   - Block C2 IP (firewall)
   - Notify SOC (Slack)
   - Create ticket
4. Human approval (if isolate prod)
5. Feedback (close / escalate)
```

Full playbook — enrich → conditional → parallel → approval → feedback. [2095][2096][2098]

### Playbook as Code

```yaml
# SOAR Playbook (YAML)
name: malware-response
trigger:
  event: malware_detected
  source: edr
steps:
  - enrich:
      - ip_reputation
      - file_hash_lookup
  - condition:
      if: known_malicious
      then:
        - isolate_endpoint
        - block_c2_ip
      else:
        - sandbox_analysis
        - notify_analyst
  - notify:
      - slack: '#security-alerts'
      - ticket: create
  - approval:
      required_if: production_isolation
```

Playbook as code — YAML, version-controlled, testable. [2094][2095][2102]

---

## 3. Security as Code

```text
Security as Code = Security controls in code (versioned + tested)
```

### Components

```text
- IaC (Terraform) + security modules
- Policy as Code (OPA/Kyverno)
- Secrets as Code (Vault)
- Detection as Code (SIEM rules)
- Playbook as Code (SOAR)
```

Security as Code — IaC + policy + secrets + detection + playbook. [2100][2103][2106]

### IaC Security Module

```hcl
# Terraform module: secure-s3-bucket
module "secure_s3" {
  source = "./modules/secure-s3"

  bucket_name = "app-data"
  encryption  = "AES256"
  versioning  = true
  public_access_block = true
  logging     = true
}
```

IaC module — secure defaults baked in. [2100][2103]

### Detection as Code

```yaml
# SIEM rule (versioned)
name: brute_force
query: |
  count(security_auth_failures_total) by (source_ip) > 5
  within 1 minute
severity: HIGH
action: alert + block_ip
```

Detection as code — SIEM rules in Git. [2094][2100]

---

## 4. Policy as Code (OPA/Kyverno)

```text
Policy as Code = Rules as machine-readable code
```

### OPA vs Kyverno

| Aspect | OPA | Kyverno |
|---|---|---|
| Language | Rego | YAML |
| Scope | Multi-system | K8s-native |
| Learning curve | Steeper | Easier |
| Use case | Unified governance | K8s-focused |

OPA (Rego, multi-system) vs Kyverno (YAML, K8s-native). [2100][2103][2106]

### OPA Policy (Rego)

```rego
package kubernetes.admission

deny[msg] {
  input.request.kind.kind == "Pod"
  input.request.object.spec.containers[_].securityContext.runAsNonRoot != true
  msg := "Container must run as non-root"
}
```

OPA Rego — deny rule (non-root enforce). [2100][2103]

### Kyverno Policy (YAML)

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-non-root
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-non-root
      match:
        resources:
          kinds: [Pod]
      validate:
        message: "runAsNonRoot required"
        pattern:
          spec:
            securityContext:
              runAsNonRoot: true
```

Kyverno YAML — validation rule (non-root). [2100][2103][2106]

### Policy Types

```text
- Validate (block non-compliant)
- Mutate (auto-fix)
- Generate (create default)
```

Kyverno — validate, mutate, generate. [2100][2106]

### Policy Rollout

```text
1. Audit mode (detect only)
2. Report violations
3. Fix violations
4. Enforce mode (block)
```

Rollout — audit → report → fix → enforce. [2100][2103]

---

## 5. Automated Remediation

```text
Remediation = Auto-fix (safe, validated)
```

### Remediation Patterns

```text
- Pre-check (precondition verify)
- Action (idempotent)
- Post-check (success verify)
- Rollback (if fail)
```

Remediation — pre-check → action → post-check → rollback. [2102]

### Remediation Example

```yaml
# Auto-remediate: Public S3 bucket
trigger: s3_public_access_detected
steps:
  - pre_check:
      - bucket_exists
      - not_production_critical
  - action:
      - block_public_access
  - post_check:
      - verify_private
  - rollback:
      - restore_previous_config
  - notify:
      - slack: '#security-alerts'
      - ticket: create
```

Remediation — pre-check → block → verify → rollback → notify. [2102][2093]

### Remediation Safety

```text
- Idempotent (re-run safe)
- Dry-run mode
- Approval gate (high-impact)
- Audit log (all actions)
- Rate limit (remediation storm prevent)
```

Remediation safety — idempotent, dry-run, approval, audit, rate limit. [2093][2102]

---

## 6. Complete Flow

```text
1. Security event (alert)
2. SOAR playbook trigger
3. Enrichment (context)
4. Conditional logic
5. Auto-remediation (if eligible)
6. Human approval (if high-impact)
7. Post-check + rollback (if fail)
8. Audit log + ticket
9. Metrics (MTTR)
```

---

## 7. Real-World Patterns

| Pattern | Use Case |
|---|---|
| SOAR + SIEM | Detection → response |
| OPA/Kyverno | K8s policy |
| Vault + Terraform | Secrets as code |
| Auto-remediation | Frequent issues |
| ChatOps | Semi-automated |

### Decision Framework

```text
Frequent + deterministic? -> Auto-remediate
High-impact? -> Human approval
K8s? -> Kyverno
Multi-system? -> OPA
Semi-auto? -> ChatOps
```

---

## 8. Common Gotchas

### Gotcha 1 — Over-Automation

```text
Problem:
False positive auto-block

Fix:
Approval gate + dry-run
```

### Gotcha 2 — No Rollback

```text
Problem:
Bad remediation stuck

Fix:
Rollback + post-check
```

### Gotcha 3 — Policy Without Audit Mode

```text
Problem:
Sudden breakage

Fix:
Audit → enforce gradual
```

### Gotcha 4 — Playbook Not Versioned

```text
Problem:
No review / test

Fix:
Playbook as code (PR + test)
```

---

## 9. Interview Questions

| Question | Answer |
|---|---|
| SOAR kya? | Orchestration + automation + response. |
| Policy as Code? | Rules as machine-readable code. |
| OPA vs Kyverno? | Rego multi-system vs YAML K8s-native. |
| Remediation safety? | Idempotent + dry-run + rollback. |
| Automation eligibility? | Frequent + deterministic + low blast radius. |
| Approval gate? | High-impact actions pe human review. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Frequent issue | Auto-remediate |
| High-impact | Approval gate |
| K8s policy | Kyverno |
| Multi-system policy | OPA |
| Semi-auto | ChatOps |

---

## 10. Key Takeaways

```text
1. SOAR = orchestration + automation + response.

2. Automation = frequent + deterministic + low blast.

3. Approval gate = high-impact.

4. Security as Code = versioned + tested.

5. OPA = Rego, multi-system.

6. Kyverno = YAML, K8s-native.

7. Remediation = pre-check + action + post-check.

8. Rollback = mandatory.

9. Audit mode = gradual rollout.

10. Playbook = code (PR + test).
```

---

## 11. Next Part Preview

**Part 44: Security Culture & Training**

Isme hum cover karenge:

```text
- Security culture deep dive
- Security training programs
- Security champions (advanced)
- Security awareness
- Metrics for culture
```