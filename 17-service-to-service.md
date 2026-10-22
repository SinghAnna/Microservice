# Part 17: Microservices Security — Service-to-Service Auth Patterns

> Service-to-service auth ke 4 patterns — **mTLS (transport identity), Client Credentials (app identity), API Keys (simple), SPIFFE (runtime identity)** — production me **mTLS + OAuth2 Client Credentials combined** recommended, static secrets se bachna. [1729][1730][1734][1742]

---

## 1. Service-to-Service Mental Model

```text
Service A -> Service B
   |
   | "Main kaun hoon?" (Authentication)
   | "Main kya kar sakta hoon?" (Authorization)
   v
Service B verify
```

### Core Question

```text
Service B ko kaise pata:
1. Request Service A se aayi (authentication)
2. Service A ko ye action allowed hai (authorization)
```

Service-to-service auth — machine identity verify + access control. [1729][1730][1740]

---

## 2. Pattern Comparison

| Pattern | Identity | Credential | Rotation | Security | Complexity |
|---|---|---|---|---|---|
| API Key | Static | Shared secret | Manual | Low | Low |
| Client Credentials | App | client_id + secret | Manual/automated | Medium | Medium |
| mTLS | Certificate | X.509 cert | Automated (SPIFFE) | High | High |
| SPIFFE | Runtime (SVID) | Short-lived cert | Automatic | Highest | High |

### Recommendation

```text
Simple internal: Client Credentials
High security: mTLS + Client Credentials
Kubernetes at scale: SPIFFE/SPIRE + mTLS
Legacy/simple: API Keys (avoid production)
```

API Keys simplest lekin weakest — no expiry, copy-able. [1729][1735][1740]

---

## 3. Pattern 1 — API Keys

```text
Service A -> Service B
Header: X-API-Key: sk_live_abc123
```

### Problems

```text
- No expiry (permanent)
- Copy-able (leak = compromise)
- No fine-grained scopes
- No standard revocation
```

### Kab Use Karein

```text
- Third-party integrations (simple)
- Low-security internal tools
- Quick prototypes
```

API Keys — simple lekin weak — production microservices me avoid. [1729][1735]

---

## 4. Pattern 2 — OAuth2 Client Credentials

```text
Service A -> Auth Server
   client_id + client_secret
   |
   v
Access Token (JWT, short-lived)
   |
   v
Service A -> Service B (Bearer token)
```

### Advantages

```text
- Short-lived tokens (15 min)
- Scope-based authorization
- Standard (OAuth2)
- Revocation support
- Gateway/gateway-friendly
```

### Spring Implementation

```java
OAuth2AuthorizeRequest request = OAuth2AuthorizeRequest
    .withClientRegistrationId("product-service")
    .principal("order-service")
    .build();

OAuth2AuthorizedClient client = manager.authorize(request);
String token = client.getAccessToken().getTokenValue();
```

Client Credentials — short-lived, scoped, standard — recommended default. [1730][1741][1742]

---

## 5. Pattern 3 — mTLS

```text
Service A -> Service B
   |
   | TLS handshake
   | Client cert + Server cert
   v
Mutual verification
```

### How It Works

```text
1. Service A -> Client certificate present
2. Service B -> Certificate verify (CA)
3. Service B -> Client identity (CN/SAN)
4. Mutual trust established
5. Encrypted channel
```

### Spring Boot Config

```yaml
server:
  ssl:
    client-auth: need
    key-store: classpath:server-keystore.p12
    trust-store: classpath:server-truststore.p12
```

mTLS — transport-level identity, dono parties verify. [1731][1736][1738]

### mTLS + OAuth2 (Combined)

```text
mTLS = Transport identity (kaun connect kar raha)
OAuth2 = Application identity + scopes (kya kar sakta)
```

mTLS + OAuth2 — transport + application layer, highest security. [1729][1731][1738]

---

## 6. Pattern 4 — SPIFFE/SPIRE (Runtime Identity)

```text
Traditional:
Static secret config me -> Copy-able -> Leak risk

SPIFFE:
Runtime attestation -> SVID issue -> Short-lived -> Auto-rotate
```

### SPIFFE ID

```text
spiffe://trust-domain/ns/backend/sa/order-service
```

### SVID (X.509)

```text
SAN URI: spiffe://trust-domain/ns/backend/sa/order-service
Lifetime: 1 hour (auto-rotate)
```

SPIFFE — workload identity runtime-issued, static secrets eliminate. [1732][1734][1739][1742]

### SPIFFE vs Static Secrets

| Aspect | Static Secret | SPIFFE |
|---|---|---|
| Credential | Config me | Runtime issued |
| Lifetime | Permanent | 1 hour |
| Rotation | Manual | Automatic |
| Leak impact | High | Low (short-lived) |
| Copy risk | High | Low (attestation-bound) |

SPIFFE — "pod kya hai" se identity, "pod kya jaanta hai" nahi. [1734][1739][1742]

---

## 7. Decision Framework

```text
Kya user context chahiye?
   |
   +--> Haan -> Token Exchange
   |
   +--> Nahi
         |
         Kya high security?
            |
            +--> Haan -> mTLS + Client Credentials
            |            (ya SPIFFE + mTLS)
            |
            +--> Nahi -> Client Credentials
```

### Decision Table

| Requirement | Pattern |
|---|---|
| Simple, low security | API Key (avoid prod) |
| Standard M2M | Client Credentials |
| High security | mTLS + Client Credentials |
| K8s at scale | SPIFFE + mTLS |
| User context | Token Exchange |

---

## 8. Complete Flow (mTLS + Client Credentials)

```text
1. Service A -> Service B (mTLS handshake)
2. Certificates exchange + verify
3. Service A -> Auth Server (client credentials)
4. Access token (JWT)
5. Service A -> Service B (Bearer token over mTLS)
6. Service B -> JWT validate + scope check
7. Response
```

### Defense in Depth

```text
Layer 1: mTLS (transport identity)
Layer 2: OAuth2 token (application identity)
Layer 3: Scope check (authorization)
Layer 4: NetworkPolicy (network isolation)
```

---

## 9. Real-World Examples

| Scenario | Pattern |
|---|---|
| Order -> Product | Client Credentials |
| Payment -> Bank API | mTLS + Client Credentials |
| K8s microservices | SPIFFE + mTLS |
| Third-party integration | API Key / OAuth2 |
| Internal batch job | Client Credentials |

---

## 10. Common Gotchas

### Gotcha 1 — Static Secrets Everywhere

```text
Problem:
Secrets config me, leak risk

Fix:
Vault / SPIFFE (runtime identity)
```

### Gotcha 2 — API Keys in Production

```text
Problem:
No expiry, no scopes

Fix:
Client Credentials / mTLS
```

### Gotcha 3 — mTLS Without AuthZ

```text
Problem:
Cert valid but koi bhi action

Fix:
mTLS + OAuth2 scopes
```

### Gotcha 4 — Long-Lived Certs

```text
Problem:
Compromise = long window

Fix:
Short-lived certs (SPIFFE)
```

---

## 11. Interview Questions

| Question | Answer |
|---|---|
| Service-to-service patterns? | API Key, Client Credentials, mTLS, SPIFFE. |
| mTLS vs Client Credentials? | Transport identity vs app identity. |
| SPIFFE kya hai? | Runtime workload identity (SVID). |
| API Keys kyu avoid? | No expiry, no scopes, copy-able. |
| Best production pattern? | mTLS + Client Credentials. |
| Least privilege? | Sirf zaroori scopes/permissions. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Standard M2M | Client Credentials |
| High security | mTLS + Client Credentials |
| K8s scale | SPIFFE + mTLS |
| User context | Token Exchange |
| Third-party | API Key / OAuth2 |

---

## 12. Key Takeaways

```text
1. 4 patterns: API Key, Client Credentials, mTLS, SPIFFE.

2. API Keys = simple but weak.

3. Client Credentials = standard M2M.

4. mTLS = transport identity.

5. SPIFFE = runtime identity (SVID).

6. mTLS + OAuth2 = best combo.

7. Short-lived > static.

8. Least privilege = kam scopes.

9. NetworkPolicy = extra layer.

10. Zero trust = identity-based.
```

---

## 13. Next Part Preview

**Part 18: mTLS (Mutual TLS) Between Services**

Isme hum cover karenge:

```text
- mTLS deep dive
- Certificate generation
- Spring Boot config
- Keystore/truststore
- Cert rotation
- Real-world setup
```