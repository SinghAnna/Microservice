# Part 1: Microservices Security — Architecture Overview

> Microservices security ka core challenge — **single app ki jagah distributed trust boundaries** — isliye defense in depth (Gateway + Service + Service-to-Service + Data) aur Zero Trust approach zaroori hai. [1537][1541][1542][1550]

---

## 1. Mental Model

```text
Monolith:
Client -> [Single App] -> DB
         (1 trust boundary)

Microservices:
Client -> Gateway -> Service A -> Service B -> DB
         |          |            |
         +----------+------------+--> Multiple trust boundaries
```

### Core Challenge

```text
Monolith: 1 app, 1 boundary, 1 security config
Microservices: N services, N boundaries, N security configs
```

Microservices me har service apna trust boundary hai — security distributed problem ban jati hai. [1537][1544][1550]

---

## 2. Full Architecture

```text
Internet
   |
   v
Load Balancer / WAF
   |
   v
API Gateway (Spring Cloud Gateway)
   |
   +--> Auth Server (Keycloak / Spring Authorization Server)
   |
   +--> User Service
   |
   +--> Order Service
   |
   +--> Product Service
   |
   +--> Notification Service
   |
   +--> Config Server
   |
   +--> Discovery Server (Eureka)
   |
   v
Databases / Message Queues
```

### Component Responsibilities

| Component | Security Responsibility |
|---|---|
| WAF / LB | DDoS, IP filtering |
| API Gateway | Auth, rate limit, token relay |
| Auth Server | Token issue, user identity |
| Services | JWT validate, method security |
| Config Server | Secrets distribution |
| Discovery | Secure service registry |

API Gateway single entry point hai — security, routing, rate limiting centralize karta hai. [1538][1539][1540][1545]

---

## 3. Security Layers (Defense in Depth)

```text
Layer 1: Edge (WAF, Gateway)
   |
Layer 2: Gateway (Auth, Rate Limit, Token Relay)
   |
Layer 3: Service (JWT validate, Method Security)
   |
Layer 4: Service-to-Service (mTLS, Client Credentials)
   |
Layer 5: Data (Encryption, Access Control)
```

### Har Layer Ka Kaam

| Layer | Kya Rokta Hai |
|---|---|
| Edge | DDoS, bot traffic |
| Gateway | Unauthorized access, rate abuse |
| Service | Business logic bypass |
| Service-to-Service | Lateral movement |
| Data | Data breach |

Defense in depth — har layer pe security check, single point of failure nahi. [1541][1542][1547][1550]

---

## 4. Trust Boundaries

```text
Trust Zone 1: Internet (Untrusted)
   |
   | Gateway (Boundary)
   v
Trust Zone 2: DMZ (Gateway)
   |
   | Service Network (Boundary)
   v
Trust Zone 3: Services (Semi-trusted)
   |
   | Data Network (Boundary)
   v
Trust Zone 4: Data (Trusted)
```

### Boundary Crossings

```text
Internet -> Gateway: TLS + Auth
Gateway -> Service: mTLS + JWT
Service -> Service: mTLS + Client Credentials
Service -> DB: Encrypted connection + least privilege
```

Trust boundaries identify karna + har crossing ka justification — OWASP recommended approach. [1537][1544][1547]

---

## 5. North-South vs East-West Traffic

```text
North-South (Client -> Service):
Client -> Gateway -> Service
Security: OAuth2/OIDC, JWT, WAF

East-West (Service -> Service):
Service A -> Service B
Security: mTLS, Client Credentials, JWT propagation
```

| Traffic Type | Direction | Security |
|---|---|---|
| North-South | Client -> Service | OAuth2, JWT, WAF |
| East-West | Service -> Service | mTLS, Client Credentials |

North-South = user-facing, East-West = internal — dono ke alag security patterns. [1541][1543][1550]

---

## 6. Zero Trust Mental Model

```text
Traditional (Perimeter):
Inside network = trusted
Outside = untrusted

Zero Trust:
Kuch bhi trusted nahi
Har request verify
Har service authenticate
```

### Zero Trust Principles

```text
1. Never trust, always verify
2. Least privilege access
3. Micro-segmentation
4. Continuous verification
5. Assume breach
```

Zero Trust — "inside network trusted" assumption drop, har request verify. [1529][1541][1550]

---

## 7. Authentication vs Authorization in Microservices

```text
Authentication:
"Kaun hai?" — Identity verify
Gateway + Service dono karte hain

Authorization:
"Kya kar sakta hai?" — Access check
Gateway (coarse) + Service (fine-grained)
```

| Concern | Gateway | Service |
|---|---|---|
| Authentication | JWT validate | JWT validate (double check) |
| Authorization | Coarse (path-based) | Fine-grained (business logic) |

Gateway coarse authorization, service fine-grained — dono zaroori. [1542][1543][1547]

---

## 8. Common Security Challenges

| Challenge | Solution |
|---|---|
| Token propagation | Token Relay / Token Exchange |
| Service identity | mTLS / Client Credentials |
| Secrets distribution | Vault / K8s Secrets |
| Network segmentation | NetworkPolicies / Service Mesh |
| Observability | Distributed tracing |
| Config security | Encrypted config / Vault |

---

## 9. Real-World Example

```text
E-Commerce:
Client -> Gateway
Gateway -> Auth Server (login)
Gateway -> Order Service (JWT relay)
Order Service -> Product Service (Client Credentials)
Order Service -> Notification Service (async, message queue)
```

### Security Flow

```text
1. User login -> JWT
2. Gateway validate -> Relay
3. Order Service validate -> Business logic
4. Order -> Product (Client Credentials)
5. Order -> Notification (async, encrypted)
```

---

## 10. Common Gotchas

### Gotcha 1 — Gateway-Only Security

```text
Problem:
Sirf gateway pe auth, services open

Fix:
Har service bhi validate (zero trust)
```

### Gotcha 2 — No Service-to-Service Auth

```text
Problem:
Internal network = trusted assumption

Fix:
mTLS / Client Credentials
```

### Gotcha 3 — Secrets in Config Files

```text
Problem:
Secrets distributed insecurely

Fix:
Vault / K8s Secrets
```

### Gotcha 4 — No Observability

```text
Problem:
Security incident trace nahi

Fix:
Distributed tracing + audit logs
```

---

## 11. Interview Questions

| Question | Answer |
|---|---|
| Microservices security challenge? | Distributed trust boundaries. |
| Defense in depth? | Multiple security layers. |
| Zero trust kya hai? | Never trust, always verify. |
| North-South vs East-West? | Client->Service vs Service->Service. |
| Gateway ka role? | Central auth + rate limit + relay. |
| mTLS kyu? | Service identity verify. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Single entry point | API Gateway |
| Service identity | mTLS / Client Credentials |
| User context forward | Token Relay |
| Secrets | Vault / K8s Secrets |
| Incident trace | Distributed tracing |

---

## 12. Key Takeaways

```text
1. Microservices = distributed trust boundaries.

2. Defense in depth = multiple layers.

3. Zero trust = never trust, always verify.

4. Gateway = central security edge.

5. North-South = user-facing.

6. East-West = service-to-service.

7. mTLS = service identity.

8. Token Relay = user context.

9. Secrets = Vault / K8s.

10. Observability = incident response.
```

---

## 13. Next Part Preview

**Part 2: API Gateway Pattern & Spring Cloud Gateway**

Isme hum cover karenge:

```text
- API Gateway pattern deep dive
- Spring Cloud Gateway setup
- Routing, filters, predicates
- Security integration
- Real-world gateway config
```