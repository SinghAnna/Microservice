# Part 20: Microservices Security — Zero Trust Architecture

> Zero Trust ka core — **NIST SP 800-207: "never trust, always verify"** — network location se trust nahi, har request continuously verify (identity + device + context), least privilege + micro-segmentation + assume breach. [1768][1769][1770][1775]

---

## 1. Zero Trust Mental Model

```text
Traditional (Castle & Moat):
[ Internet ]---[ Firewall ]---[ Trusted Inside ]
                                    |
                                    +--> Sab trust (ek baar verify)

Zero Trust:
[ Internet ]---[ Gateway ]---[ Service A ]---[ Service B ]
                  |              |              |
                  +--------------+--------------+--> Har request verify
```

### Core Shift

```text
Traditional: "Inside = trusted"
Zero Trust:  "Kuch bhi trusted nahi — har baar verify"
```

Zero Trust — implicit trust nahi, har access request continuously evaluate. [1768][1769][1776]

---

## 2. NIST SP 800-207 Principles

| Principle | Meaning |
|---|---|
| Per-session access | Har session alag verify |
| Dynamic policy | Identity + device + context + risk |
| Continuous monitoring | Real-time posture check |
| Least privilege | Sirf zaroori access |
| Assume breach | Attacker andar maan ke design |

### 7 Tenets (NIST)

```text
1. Sab resources = protected resources
2. Sab communication secure (location irrelevant)
3. Per-session access
4. Dynamic policy (observable state)
5. Enterprise monitor + measure integrity
6. AuthN + AuthZ dynamic + strictly enforced
7. Information collection = attack surface reduce
```

NIST SP 800-207 — 7 tenets, resource protection focused. [1768][1769][1776][1779]

---

## 3. Core Components

```text
Policy Engine (PE)     -> Decision (allow/deny)
Policy Administrator   -> Session establish/terminate
Policy Enforcement Point (PEP) -> Enforce (Gateway/Proxy)
```

### Architecture

```text
Subject (User/Service)
   |
   v
PEP (Gateway/Envoy)
   |
   | Signal (identity, device, context)
   v
Policy Engine (PE)
   |
   | Decision
   v
PEP -> Resource (allow) / Block (deny)
```

| Component | Role |
|---|---|
| Policy Engine | Decision (identity + context + risk) |
| Policy Administrator | Session control |
| PEP | Enforcement (Gateway/Envoy) |

Policy Engine decision leta hai, PEP enforce karta hai — NIST ZTA core. [1775][1776]

---

## 4. Zero Trust in Microservices

```text
1. Identity: SPIFFE/SVID (service identity)
2. Authentication: mTLS (har connection)
3. Authorization: Fine-grained (per-request)
4. Segmentation: NetworkPolicies (default deny)
5. Monitoring: Tracing + audit logs
```

### Implementation Stack

| Layer | Tool |
|---|---|
| Identity | SPIFFE/SPIRE |
| Transport | mTLS (Istio/Linkerd) |
| Authorization | OPA / Istio AuthorizationPolicy |
| Segmentation | Kubernetes NetworkPolicies |
| Observability | Jaeger + Prometheus |

Zero Trust microservices — identity + mTLS + fine-grained authz + segmentation. [1772][1775][1780]

---

## 5. Identity-Based Access

```text
Traditional:
IP whitelist -> 10.0.0.5 allowed

Zero Trust:
SPIFFE ID -> spiffe://prod/ns/backend/sa/order-service
+ JWT claims -> scope, tenant
+ Context -> time, location, device
```

### Policy Example (ABAC)

```text
ALLOW order-service -> product-service
IF:
- SPIFFE ID = order-service
- JWT scope = product:read
- Time = business hours
- Device posture = healthy
```

Identity-based — "kaun" + "kya" + "kahan" + "kab" — dynamic policy. [1770][1776][1779]

---

## 6. Continuous Verification

```text
Traditional:
Login ek baar -> Session valid (hours)

Zero Trust:
Har request -> Re-evaluate
- Token valid?
- Device healthy?
- Behavior normal?
- Risk score low?
```

### Risk-Based Access

```text
Low risk:  Normal access
Medium:    Step-up MFA
High:      Block / re-auth
```

Continuous verification — har request re-evaluate, risk-based step-up. [1770][1774][1776]

---

## 7. Assume Breach

```text
Maan lo attacker andar hai:
- Lateral movement block (micro-segmentation)
- Blast radius kam (least privilege)
- Detection fast (monitoring)
- Recovery fast (backups, rotation)
```

### Breach Scenario

```text
Attacker compromises Service A
   |
   v
Service A -> Service B try
   |
   v
NetworkPolicy: Block (sirf specific allow)
mTLS: Identity verify fail
Blast radius: Service A only
```

Assume breach — lateral movement block, blast radius minimize. [1768][1770][1778]

---

## 8. Complete Architecture

```text
Internet
   |
   v
WAF + Gateway (PEP)
   |
   | OAuth2/OIDC + JWT
   v
Service Mesh (mTLS + SPIFFE)
   |
   +--> Service A (PEP: sidecar)
   |       |
   |       +--> Policy Engine (OPA/Istio)
   |
   +--> Service B (PEP: sidecar)
   |
   v
Data (encrypted, least privilege)
```

### Security Controls

| Layer | Control |
|---|---|
| Edge | WAF + Gateway + Rate limit |
| Identity | SPIFFE + OIDC |
| Transport | mTLS (STRICT) |
| Authorization | OPA / AuthorizationPolicy |
| Network | NetworkPolicies (default deny) |
| Data | Encryption + least privilege |
| Monitoring | Tracing + audit + alerts |

---

## 9. Real-World Example

```text
E-Commerce Zero Trust:

1. User -> Gateway (OIDC login)
2. Gateway -> Order Service (mTLS + JWT relay)
3. Order -> Product (mTLS + Client Credentials + SPIFFE)
4. NetworkPolicy: Order -> Product only
5. Policy Engine: scope + tenant + time check
6. Assume breach: Service A compromise = Service A only
```

---

## 10. Common Gotchas

### Gotcha 1 — "Zero Trust = Product"

```text
Problem:
Ek tool se zero trust nahi hota

Fix:
Architecture + process + tools
```

### Gotcha 2 — Big Bang Migration

```text
Problem:
Sab ek saath change = breakage

Fix:
Incremental (permissive -> strict)
```

### Gotcha 3 — Network-Only Focus

```text
Problem:
Sirf segmentation, identity ignore

Fix:
Identity + device + context + network
```

### Gotcha 4 — No Monitoring

```text
Problem:
Breach detect nahi

Fix:
Tracing + audit + alerting
```

---

## 11. Interview Questions

| Question | Answer |
|---|---|
| Zero Trust kya hai? | Never trust, always verify — NIST SP 800-207. |
| Core principles? | Least privilege, micro-segmentation, assume breach. |
| PEP vs PE? | Enforcement vs Decision point. |
| Identity-based access? | SPIFFE + JWT claims + context. |
| Continuous verification? | Har request re-evaluate. |
| Assume breach? | Lateral movement block + blast radius kam. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Zero trust implement | Identity + mTLS + fine-grained authz |
| Lateral movement | Micro-segmentation |
| Dynamic access | ABAC + risk-based |
| Breach containment | Assume breach + blast radius |
| Monitoring | Tracing + audit + alerts |

---

## 12. Key Takeaways

```text
1. Zero Trust = never trust, always verify.

2. NIST SP 800-207 = standard.

3. Network location = trust nahi.

4. Identity = SPIFFE + JWT.

5. Least privilege = minimal access.

6. Micro-segmentation = default deny.

7. Assume breach = blast radius kam.

8. Continuous verification.

9. PEP + PE + PA = core components.

10. Zero Trust = journey, not product.
```

---

## 13. Next Part Preview

**Part 21: BFF Pattern (Backend for Frontend)**

Isme hum cover karenge:

```text
- BFF pattern deep dive
- Token handling (server-side)
- SPA + Mobile + Web
- Security benefits
- Real-world implementation
```