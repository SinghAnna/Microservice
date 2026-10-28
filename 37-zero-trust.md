# Part 37: Microservices Security — Zero Trust Architecture

> Zero trust ka core — **"Never trust, always verify"** — NIST SP 800-207 ke 7 tenets + identity-centric access + micro-segmentation + continuous verification; network location = trust nahi, har request re-evaluate. [1768][2008][2012][2013]

---

## 1. Zero Trust Mental Model

```text
Traditional (Perimeter):
Inside network = trusted
Outside = untrusted
Problem: Lateral movement

Zero Trust:
Inside = untrusted bhi
Outside = untrusted
Har request = verify
```

### Core Principle

```text
"Never trust, always verify"
"Assume breach"
"Least privilege"
```

Zero trust — implicit trust remove, har request verify, assume breach. [2008][2013][2017]

---

## 2. NIST SP 800-207 Tenets

```text
1. All resources = protected assets
2. Secure communication (location-independent)
3. Per-session access
4. Dynamic policy
5. Monitor asset integrity
6. Dynamic authN + authZ
7. Data-driven improvement
```

### Tenets Explained

| Tenet | Meaning |
|---|---|
| Resources | Sab kuch protect (apps, DBs, APIs) |
| Location-independent | LAN bhi untrusted |
| Per-session | Har session fresh evaluate |
| Dynamic policy | Identity + device + behavior |
| Monitor | Asset posture check |
| Dynamic authN/authZ | Har request re-evaluate |
| Data-driven | Improve from telemetry |

NIST 7 tenets — resource protection, location-independent security, per-session access, dynamic policy, monitoring, dynamic authN/authZ, data-driven. [1768][2015][2017]

---

## 3. Zero Trust Components

```text
PEP (Policy Enforcement Point)
   |
   v
PDP (Policy Decision Point)
   |
   +--> IdP (Identity Provider)
   +--> Device Trust Engine
   +--> Risk Engine
   +--> Data Classification
```

### Components

| Component | Role |
|---|---|
| PEP | Request intercept + enforce |
| PDP | Decision (allow/deny) |
| IdP | User/device identity |
| Device Trust | Posture check |
| Risk Engine | Behavior + anomaly |
| Data Classification | Sensitivity tags |

PEP (enforce) + PDP (decide) + IdP (identity) + device trust + risk engine. [2017][2012]

### Access Flow

```text
1. User -> PEP
2. PEP -> PDP (query)
3. PDP -> IdP (identity)
4. PDP -> Device Trust (posture)
5. PDP -> Risk Engine (behavior)
6. PDP -> Decision (allow/deny)
7. PEP -> Enforce
8. Next request -> Re-evaluate
```

Access flow — PEP intercept → PDP evaluate → enforce → re-evaluate. [2017][2012]

---

## 4. Identity-Centric Security

```text
Identity = New Perimeter
```

### Identity Types

| Type | Example |
|---|---|
| Human | Employee, contractor |
| Device | Laptop, mobile |
| Workload | Service account, pod |
| Non-human | API, bot |

Identity — human + device + workload + non-human (SPIFFE). [2013][2017]

### Strong Authentication

```text
- MFA (mandatory)
- SSO (federated)
- Passwordless (FIDO2)
- Short-lived tokens
```

MFA + SSO + passwordless + short-lived tokens. [2012][2017][2019]

### Workload Identity (SPIFFE)

```text
SPIFFE ID = spiffe://trust-domain/ns/namespace/sa/service-account
SVID = X.509 cert (short-lived)
mTLS = service-to-service auth
```

SPIFFE/SPIRE — workload identity + short-lived SVID + mTLS. [2017][2013]

---

## 5. Micro-Segmentation

```text
Traditional segmentation:
Zone-level (DMZ, internal)

Micro-segmentation:
Workload-level (pod-to-pod)
```

### Micro-Segmentation Benefits

```text
- Lateral movement block
- Blast radius minimize
- Least privilege enforce
- East-west traffic control
```

Micro-segmentation — workload-level isolation, lateral movement block, blast radius minimize. [2008][2013][2015]

### Implementation (Kubernetes)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: order-to-payment
spec:
  podSelector:
    matchLabels:
      app: payment-service
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: order-service
      ports:
        - port: 8080
```

K8s NetworkPolicy — label-based micro-segmentation. [2013][2015]

### Service Mesh (Istio)

```text
Istio sidecar -> mTLS + policy
   |
   +--> Identity (SPIFFE)
   +--> Authorization (RBAC)
   +--> Traffic control
```

Istio — sidecar-based mTLS + authorization policy. [2013][2017]

---

## 6. Continuous Verification

```text
Traditional:
Login once -> Access forever

Zero Trust:
Har request -> Re-evaluate
```

### Continuous Checks

```text
- Session re-evaluation (periodic)
- Device posture (EDR agent)
- Behavior anomaly (UEBA)
- Step-up auth (sensitive ops)
- Token expiry (short-lived)
```

Continuous verification — session re-eval, device posture, behavior, step-up auth. [2012][2017][2019]

### Step-Up Authentication

```text
Normal op: Password + MFA
Sensitive op: + Biometric / re-auth
```

Step-up — sensitive operations pe extra auth. [2017][2019]

---

## 7. BeyondCorp Model

```text
Google BeyondCorp:
- No privileged intranet
- Apps on Internet
- Access proxy (Identity-Aware Proxy)
- Device inventory + trust
```

### BeyondCorp Components

| Component | Purpose |
|---|---|
| Access Proxy | Policy enforcement |
| Device Inventory | Trust state |
| Trust Engine | Risk evaluate |
| Control Plane | Policy manage |

BeyondCorp — access proxy + device inventory + trust engine + control plane. [2009][2011][2019]

### BeyondCorp Flow

```text
1. User -> Access Proxy
2. Proxy -> Trust Engine (device + user)
3. Trust Engine -> Decision
4. Proxy -> Backend (if allowed)
5. No VPN needed
```

BeyondCorp — access proxy replaces VPN, device + user trust evaluate. [2009][2011][2019]

---

## 8. Complete Flow

```text
1. User/Service -> PEP
2. PEP -> PDP
3. PDP -> IdP + Device Trust + Risk
4. PDP -> Decision
5. PEP -> Enforce (mTLS + policy)
6. Micro-segmentation (east-west)
7. Continuous monitoring
8. Re-evaluate (next request)
```

---

## 9. Real-World Patterns

| Pattern | Use Case |
|---|---|
| BeyondCorp | Remote workforce |
| Service mesh (Istio) | Microservices mTLS |
| SPIFFE/SPIRE | Workload identity |
| ZTNA | VPN replacement |
| Micro-segmentation | East-west control |

### Decision Framework

```text
Remote-first? -> BeyondCorp / ZTNA
Microservices? -> Istio + SPIFFE
High security? -> Micro-segmentation + continuous verify
Legacy? -> Phased rollout (identity first)
```

---

## 10. Common Gotchas

### Gotcha 1 — Network Location = Trust

```text
Problem:
LAN = trusted assumption

Fix:
Location-independent verification
```

### Gotcha 2 — One-Time Auth

```text
Problem:
Login once, access forever

Fix:
Continuous re-evaluation
```

### Gotcha 3 — No Workload Identity

```text
Problem:
Service-to-service implicit trust

Fix:
SPIFFE + mTLS
```

### Gotcha 4 — Big Bang Migration

```text
Problem:
Disruption

Fix:
Phased rollout (identity → segmentation → verify)
```

---

## 11. Interview Questions

| Question | Answer |
|---|---|
| Zero trust kya? | Never trust, always verify. |
| NIST 7 tenets? | Resources, location-independent, per-session, dynamic, monitor, authN/authZ, data-driven. |
| PEP vs PDP? | Enforce vs decide. |
| SPIFFE kya? | Workload identity standard. |
| Micro-segmentation? | Workload-level isolation. |
| BeyondCorp? | Google's zero trust implementation. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Remote workforce | BeyondCorp / ZTNA |
| Microservices mTLS | Istio + SPIFFE |
| Lateral movement | Micro-segmentation |
| Sensitive op | Step-up auth |
| Legacy migration | Phased rollout |

---

## 12. Key Takeaways

```text
1. Zero trust = never trust, always verify.

2. NIST = 7 tenets.

3. Identity = new perimeter.

4. PEP = enforce, PDP = decide.

5. SPIFFE = workload identity.

6. Micro-segmentation = east-west control.

7. Continuous = re-evaluate every request.

8. BeyondCorp = no VPN, access proxy.

9. Step-up = sensitive ops.

10. Phased = identity → segmentation → verify.
```

---

## 13. Next Part Preview

**Part 38: API Security (OWASP API Top 10)**

Isme hum cover karenge:

```text
- API security deep dive
- OWASP API Top 10
- BOLA/IDOR
- Rate limiting
- API gateway security
```