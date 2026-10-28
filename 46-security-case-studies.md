# Part 46: Microservices Security — Security Case Studies

> Case studies ka core — **real incidents (Capital One, Uber, Equifax, SolarWinds, Log4Shell) → root cause → lessons → controls** — har incident ek specific security gap expose karta hai, aur modern controls (IMDSv2, Vault, SBOM, patch SLA) usi gap ka direct answer hain. [2138][2139][2145][2148]

---

## 1. Case Study Mental Model

```text
Incident -> Root Cause -> Lesson -> Control

Pattern:
Har breach = ek specific gap
Modern control = usi gap ka fix
```

### Case Study Framework

```text
1. What happened (timeline)
2. Root cause (technical + process)
3. Impact (data + financial + reputational)
4. Lessons (what to learn)
5. Controls (what to implement)
```

Case study framework — what → root cause → impact → lessons → controls. [2138][2145]

---

## 2. Capital One (2019) — SSRF + IAM

```text
Incident: 106M records exposed
Attack: SSRF -> Metadata service -> IAM creds -> S3
```

### Attack Chain

```text
1. WAF misconfiguration (SSRF possible)
2. SSRF -> Metadata service (169.254.169.254)
3. IAM role creds fetch
4. Over-permissive role (S3 broad access)
5. S3 bucket enumerate + exfiltrate
```

Capital One — WAF misconfig → SSRF → metadata → IAM → S3. [2138][2141][2146]

### Root Cause

```text
1. WAF misconfiguration (SSRF possible)
2. IMDSv1 (no session token)
3. IAM role over-permissive (wildcard S3)
4. No egress restriction
5. Monitoring gap (normal-looking API calls)
```

Root cause — WAF misconfig + IMDSv1 + over-permissive IAM + no egress restriction + monitoring gap. [2138][2140][2146]

### Lessons + Controls

| Lesson | Control |
|---|---|
| SSRF hardening | IMDSv2 + egress restriction |
| Least privilege | Scoped IAM (no wildcard) |
| Metadata protection | Hop limit + token required |
| Cloud monitoring | CloudTrail + GuardDuty |
| WAF change mgmt | Peer review + staged rollout |

Lessons — IMDSv2, scoped IAM, hop limit, CloudTrail/GuardDuty, WAF change mgmt. [2140][2141][2146]

---

## 3. Uber (2016) — Secrets in Git

```text
Incident: 57M users/drivers exposed
Attack: GitHub creds -> S3 access -> Data exfil
```

### Attack Chain

```text
1. AWS access key hardcoded in GitHub repo
2. Attacker finds key (repo scan)
3. S3 bucket access (over-permissive)
4. Data exfiltrate (57M records)
5. Breach hidden 1 year (ransom paid)
```

Uber — GitHub creds → S3 → exfil → hidden breach. [2137][2139][2149]

### Root Cause

```text
1. Secrets in Git (hardcoded)
2. No secrets scanning
3. Over-permissive S3 access
4. No MFA on GitHub
5. Breach concealment (ransom paid)
```

Root cause — secrets in Git, no scan, over-permissive S3, no MFA, concealment. [2137][2139][2149]

### Lessons + Controls

| Lesson | Control |
|---|---|
| Secrets in Git | Vault / Secrets Manager |
| Secrets scanning | gitleaks / trufflehog (CI) |
| Least privilege | Scoped S3 policy |
| MFA | GitHub + AWS mandatory |
| Transparency | Breach disclosure policy |

Lessons — Vault, gitleaks, scoped S3, MFA, disclosure policy. [2139][2149]

---

## 4. Equifax (2017) — Unpatched CVE

```text
Incident: 147M records exposed
Attack: Known CVE (Apache Struts) unpatched
```

### Attack Chain

```text
1. Known CVE (CVSS 9.8) public
2. Patch available, not applied
3. Exploit -> Web shell
4. Data exfiltrate (76 days undetected)
5. Monitoring cert expired (blind spot)
```

Equifax — known CVE unpatched → exploit → 76 days undetected. [2145][2148]

### Root Cause

```text
1. Patch management failure
2. Asset inventory gap (unknown exposure)
3. Monitoring cert expired
4. No patch SLA
5. Slow incident response
```

Root cause — patch failure, inventory gap, cert expired, no SLA, slow response. [2145][2148]

### Lessons + Controls

| Lesson | Control |
|---|---|
| Patch SLA | Critical: 24-48h |
| Asset inventory | CMDB + scanning |
| Monitoring health | Cert expiry alerts |
| Incident readiness | IR plan + tabletop |
| Transparency | Timely disclosure |

Lessons — patch SLA, CMDB, cert alerts, IR plan, disclosure. [2145][2148]

---

## 5. SolarWinds (2020) — Supply Chain

```text
Incident: 18,000+ orgs affected
Attack: Build compromise -> Malicious update
```

### Attack Chain

```text
1. Build system compromise
2. Malicious code in update
3. Signed update (trust bypass)
4. Customers install
5. Backdoor activate (dormant)
```

SolarWinds — build compromise → malicious update → signed → backdoor. [2145][2148]

### Root Cause

```text
1. Build pipeline compromise
2. No build provenance
3. No artifact signing verification
4. Trust-based update (signature only)
5. No behavior monitoring
```

Root cause — build compromise, no provenance, no verification, trust-based, no behavior monitoring. [2145][2148]

### Lessons + Controls

| Lesson | Control |
|---|---|
| Build integrity | SLSA provenance |
| Artifact verification | Cosign + policy |
| Supply chain visibility | SBOM |
| Behavior detection | UEBA + anomaly |
| Vendor risk | Third-party assessment |

Lessons — SLSA, Cosign, SBOM, UEBA, vendor risk. [2145][2148]

---

## 6. Log4Shell (2021) — Dependency Risk

```text
Incident: CVSS 10.0, global impact
Attack: JNDI lookup in log message
```

### Attack Chain

```text
1. Log4j vulnerability (JNDI)
2. Attacker sends crafted log message
3. JNDI lookup -> Remote code execution
4. Widespread exploitation
```

Log4Shell — JNDI lookup → RCE. [2148]

### Root Cause

```text
1. Transitive dependency (unknown usage)
2. No SBOM (visibility gap)
3. Slow patching
4. Widespread exposure
```

Root cause — transitive dep, no SBOM, slow patching, widespread. [2148]

### Lessons + Controls

| Lesson | Control |
|---|---|
| Dependency visibility | SBOM (CycloneDX/SPDX) |
| Fast patching | Patch SLA + automation |
| Transitive deps | Dependency tree scan |
| Input validation | Log injection prevention |

Lessons — SBOM, patch SLA, dep tree scan, input validation. [2148]

---

## 7. Cross-Case Patterns

```text
Common Themes:
1. Identity = new perimeter (Capital One, Uber)
2. Secrets = high-value target (Uber)
3. Patching = basic but critical (Equifax)
4. Supply chain = systemic risk (SolarWinds)
5. Visibility = detection gap (all)
```

### Pattern Matrix

| Incident | Primary Gap | Modern Control |
|---|---|---|
| Capital One | Cloud misconfig | IMDSv2 + IAM least privilege |
| Uber | Secrets in Git | Vault + gitleaks |
| Equifax | Unpatched CVE | Patch SLA + inventory |
| SolarWinds | Build compromise | SLSA + Cosign + SBOM |
| Log4Shell | Transitive dep | SBOM + dep scan |

Pattern matrix — incident → gap → control. [2138][2139][2145][2148]

---

## 8. Interview Scenarios

| Scenario | Answer |
|---|---|
| "Capital One se kya seekha?" | SSRF + IAM least privilege + IMDSv2 |
| "Uber se kya seekha?" | Secrets management + transparency |
| "Equifax se kya seekha?" | Patch SLA + asset inventory |
| "SolarWinds se kya seekha?" | Supply chain + SBOM + provenance |
| "Log4Shell se kya seekha?" | Dependency visibility + fast patching |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Cloud misconfig | IMDSv2 + IAM review |
| Secrets leak | Vault + scanning |
| Unpatched CVE | Patch SLA |
| Supply chain | SBOM + signing |
| Dependency risk | SBOM + dep tree |

---

## 9. Key Takeaways

```text
1. Capital One = SSRF + IAM least privilege.

2. Uber = Secrets in Git = disaster.

3. Equifax = Patch SLA critical.

4. SolarWinds = Supply chain integrity.

5. Log4Shell = Dependency visibility.

6. Identity = new perimeter.

7. Secrets = high-value target.

8. Visibility = detection gap.

9. Modern controls = direct answers.

10. Lessons = proactive defense.
```

---

## 10. Next Part Preview

**Part 47: Security Interview Prep (Advanced)**

Isme hum cover karenge:

```text
- Security interview deep dive
- System design (security)
- Behavioral questions
- Coding (secure code)
- Mock interview scenarios
```