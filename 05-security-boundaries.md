# Part 5: Microservices Security — Security Boundaries & Trust Zones

> Trust zones ka core idea — **network location se trust nahi, cryptographic identity se trust** — Zero Trust me har service apni identity (SPIFFE/SVID) se verify hoti hai, aur Kubernetes NetworkPolicies se **default-deny + explicit allow** segmentation hoti hai. [1596][1598][1604][1605]

---

## 1. Trust Zones Mental Model

```text
Traditional (Perimeter):
[ Internet ]---[ Firewall ]---[ Trusted Internal Network ]
                                    |
                                    +--> Sab services (full trust)

Zero Trust:
[ Internet ]---[ Gateway ]---[ Zone A ]---[ Zone B ]---[ Zone C ]
                  |              |           |           |
                  +--------------+-----------+-----------+--> Har boundary pe verify
```

### Zone Types

| Zone | Trust Level | Examples |
|---|---|---|
| Internet | Untrusted | Public users |
| DMZ | Low trust | Gateway, WAF |
| Application | Medium trust | Business services |
| Data | High trust | DB, message queues |

Trust zones — network ko segments me divide karna, har zone ka alag trust level. [1596][1599][1604]

---

## 2. Security Boundaries

```text
Boundary 1: Internet -> Gateway
   TLS + WAF + Rate Limit

Boundary 2: Gateway -> Services
   mTLS + JWT + AuthZ

Boundary 3: Service -> Service
   mTLS + Client Credentials + AuthZ

Boundary 4: Service -> Data
   TLS + Least Privilege + Encryption
```

### Har Boundary Ka Control

| Boundary | Authentication | Authorization | Encryption |
|---|---|---|---|
| Internet -> Gateway | OAuth2/OIDC | Coarse | TLS |
| Gateway -> Service | JWT | Coarse | mTLS |
| Service -> Service | Client Credentials / mTLS | Fine-grained | mTLS |
| Service -> Data | Service identity | Least privilege | TLS + at-rest |

Har boundary pe authentication + authorization + encryption — teeno zaroori. [1596][1600][1604]

---

## 3. Network Segmentation

```text
Kubernetes Cluster
   |
   +--> Namespace: frontend
   |       +--> Gateway pods
   |
   +--> Namespace: backend
   |       +--> Order, Product, User pods
   |
   +--> Namespace: data
   |       +--> DB, Redis pods
   |
   +--> Namespace: security
           +--> Auth Server, Vault
```

### Segmentation Levels

| Level | Scope | Example |
|---|---|---|
| Macro | Clusters / VPCs | Prod vs Dev cluster |
| Micro | Namespaces / zones | Frontend vs backend |
| Nano | Pod-to-pod | Order -> Product only |

Macro / micro / nano segmentation — cluster se pod-level tak. [1599][1601][1605]

---

## 4. Kubernetes NetworkPolicies

### Default Deny (All Namespace)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: backend
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

### Allow Order -> Product Only

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-order-to-product
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: product-service
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: order-service
      ports:
        - protocol: TCP
          port: 8080
```

**Default deny + explicit allow** — Zero Trust ka network-level implementation. [1594][1595][1605][1606]

---

## 5. Service Identity (SPIFFE/SPIRE)

```text
SPIFFE ID:
spiffe://trust-domain/ns/backend/sa/order-service

SVID (X.509 cert):
SAN me SPIFFE ID encoded
Short-lived (1-24 hours)
```

### Flow

```text
1. Service start
2. SPIRE agent attest (K8s PSA)
3. SVID issue (X.509 cert)
4. mTLS handshake (SVID exchange)
5. Identity verify
```

SPIFFE — har workload ko cryptographic identity (SVID), mTLS ke liye. [1598][1600][1603]

### SPIFFE ID Structure

```text
spiffe://<trust-domain>/<namespace>/<service-account>
```

| Part | Meaning |
|---|---|
| trust-domain | Organization |
| namespace | K8s namespace |
| service-account | Service identity |

---

## 6. Zero Trust Principles

```text
1. Never trust, always verify
2. Least privilege access
3. Micro-segmentation
4. Explicit allow (default deny)
5. Continuous verification
6. Assume breach
7. Encrypt everything
8. Monitor everything
```

### Zero Trust vs Traditional

| Aspect | Traditional | Zero Trust |
|---|---|---|
| Trust basis | Network location | Identity |
| Default | Allow | Deny |
| Verification | Once at perimeter | Continuous |
| Segmentation | Coarse | Micro |
| Monitoring | Perimeter | Everywhere |

Zero Trust — location-based trust drop, identity-based trust. [1596][1600][1604]

---

## 7. Complete Secure Architecture

```text
Internet
   |
   v
WAF + LB (TLS termination)
   |
   v
API Gateway (mTLS + JWT + Rate Limit)
   |
   +--> Auth Server (SPIFFE identity)
   |
   +--> Order Service (SPIFFE + NetworkPolicy)
   |       |
   |       +--> Product Service (mTLS + Client Credentials)
   |
   +--> User Service (SPIFFE + NetworkPolicy)
   |
   v
Database (TLS + least privilege + encryption at rest)
```

### Security Controls Per Layer

| Layer | Controls |
|---|---|
| Edge | WAF, TLS, DDoS |
| Gateway | OAuth2, JWT, Rate Limit, mTLS |
| Service | JWT validate, Method Security, NetworkPolicy |
| Service-to-Service | mTLS, Client Credentials, SPIFFE |
| Data | TLS, Encryption at rest, Least privilege |

---

## 8. Real-World Example

```text
E-Commerce Zero Trust:

1. User -> Gateway (OAuth2 login)
2. Gateway -> Order Service (mTLS + JWT relay)
3. Order -> Product (mTLS + Client Credentials)
4. Order -> Notification (mTLS + async)
5. NetworkPolicy: Order -> Product only
6. NetworkPolicy: Product -> Order deny
7. SPIFFE identity har service ka
8. Short-lived certs (1 hour)
```

---

## 9. Common Gotchas

### Gotcha 1 — Flat Network

```text
Problem:
Sab pods ek network me, koi bhi kisi se baat

Fix:
NetworkPolicies + namespaces
```

### Gotcha 2 — Long-Lived Certs

```text
Problem:
Static certs, rotation nahi

Fix:
SPIFFE/SPIRE short-lived SVIDs
```

### Gotcha 3 — Perimeter-Only Security

```text
Problem:
Andar sab trusted

Fix:
Zero Trust — har service verify
```

### Gotcha 4 — No Traffic Visibility

```text
Problem:
Lateral movement detect nahi

Fix:
Service mesh + tracing
```

---

## 10. Interview Questions

| Question | Answer |
|---|---|
| Trust zone kya hai? | Network segment with trust level. |
| Zero trust kya hai? | Never trust, always verify. |
| NetworkPolicy kya hai? | K8s pod-to-pod traffic control. |
| Default deny kya hai? | Sab block, explicit allow only. |
| SPIFFE kya hai? | Workload identity standard (SVID). |
| Micro-segmentation? | Pod-level network isolation. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Pod isolation | NetworkPolicy |
| Service identity | SPIFFE/SVID |
| Zero trust | mTLS + default deny |
| Lateral movement | Micro-segmentation |
| Cert rotation | SPIRE short-lived certs |

---

## 11. Key Takeaways

```text
1. Trust zones = segmented network.

2. Zero trust = identity-based trust.

3. Default deny + explicit allow.

4. NetworkPolicy = K8s segmentation.

5. SPIFFE = workload identity.

6. SVID = short-lived cert.

7. mTLS = service-to-service auth.

8. Micro-segmentation = pod-level.

9. Continuous verification.

10. Assume breach mindset.
```

---

## 12. Next Part Preview

**Part 6: OAuth2 & OIDC in Microservices**

Isme hum cover karenge:

```text
- OAuth2 flows in microservices
- OIDC deep dive
- Token types (access, refresh, ID)
- Token validation strategies
- Real-world setup
```