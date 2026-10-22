# Part 19: Microservices Security — Service Mesh Security (Istio)

> Istio ka core — **Envoy sidecar (data plane) + Istiod (control plane) + automatic mTLS + AuthorizationPolicy** — app code unchanged, security transparently enforced. [1753][1756][1757]

---

## 1. Service Mesh Mental Model

```text
Without Mesh:
Service A -> Service B
(App code me: mTLS config, retry, circuit breaker, tracing)

With Mesh:
Service A -> [Sidecar] -> [Sidecar] -> Service B
             (Envoy)      (Envoy)
(App code unchanged — mesh sab handle)
```

### Mesh Responsibilities

```text
- mTLS (automatic)
- Traffic management (routing, retries)
- Policy enforcement
- Observability (tracing, metrics)
- Resilience (circuit breaker)
```

Service mesh — networking concerns app se alag, sidecar proxies handle. [1753][1759][1764]

---

## 2. Istio Architecture

```text
Control Plane (Istiod)
   |
   | Config (routes, policies, certs)
   v
Data Plane (Envoy Sidecars)
   |
   +--> Service A pod [App + Envoy]
   +--> Service B pod [App + Envoy]
   +--> Service C pod [App + Envoy]
```

### Components

| Component | Plane | Role |
|---|---|---|
| Envoy | Data | Proxy (traffic, mTLS, policy) |
| Istiod | Control | Config + CA + discovery |
| Citadel (in Istiod) | Control | Certificate authority |
| Pilot (in Istiod) | Control | Route config |

Istiod = Pilot + Citadel + Galley unified — config + CA + discovery. [1753][1759][1760][1766]

### Sidecar Pattern

```text
Pod:
+---------------------------+
|  App Container            |
|  (Spring Boot)            |
+---------------------------+
|  Envoy Sidecar            |
|  (mTLS, policy, tracing)  |
+---------------------------+
```

Sidecar — har pod me Envoy, app ke saath, traffic intercept. [1753][1764]

---

## 3. Automatic mTLS

```text
Istiod (Citadel) -> SPIFFE identity + cert issue
   |
   v
Envoy sidecar -> mTLS handshake (transparent)
   |
   v
App code unchanged
```

### PeerAuthentication

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system  # Mesh-wide
spec:
  mtls:
    mode: STRICT
```

### mTLS Modes

| Mode | Behavior |
|---|---|
| `STRICT` | Sirf mTLS (plaintext reject) |
| `PERMISSIVE` | mTLS + plaintext dono |
| `DISABLE` | No mTLS |

`STRICT` — mesh-wide mTLS enforced, plaintext reject. [1757][1758][1765][1767]

### Migration Path

```text
PERMISSIVE (start) -> STRICT (after all services onboard)
```

Migration — pehle PERMISSIVE, saare services onboard hone ke baad STRICT. [1758][1765]

---

## 4. AuthorizationPolicy

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: order-to-product
  namespace: backend
spec:
  selector:
    matchLabels:
      app: product-service
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/backend/sa/order-service"
      to:
        - operation:
            methods: ["GET"]
            paths: ["/api/products/*"]
```

### Policy Structure

```yaml
selector:   -> Kis workload pe
from:       -> Kaun (principal/namespace)
to:         -> Kya (method/path)
when:       -> Conditions (JWT claims)
action:     -> ALLOW / DENY / CUSTOM
```

`AuthorizationPolicy` — identity-based access control, service account + method + path. [1755][1756][1761]

### Default Deny

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: backend
spec:
  selector: {}
  action: DENY
  rules:
    - {}
```

Default deny + explicit allow — Zero Trust pattern. [1754][1755][1767]

---

## 5. JWT Validation (RequestAuthentication)

```yaml
apiVersion: security.istio.io/v1
kind: RequestAuthentication
metadata:
  name: jwt-validation
  namespace: backend
spec:
  selector:
    matchLabels:
      app: order-service
  jwtRules:
    - issuer: "[https://auth.example.com](https://auth.example.com)"
      jwksUri: "[https://auth.example.com/oauth2/jwks](https://auth.example.com/oauth2/jwks)"
      audiences:
        - order-service
```

### JWT + AuthorizationPolicy

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
  namespace: backend
spec:
  selector:
    matchLabels:
      app: order-service
  action: ALLOW
  rules:
    - when:
        - key: request.auth.claims[iss]
          values: ["[https://auth.example.com](https://auth.example.com)"]
        - key: request.auth.claims[scope]
          values: ["orders:read"]
```

`RequestAuthentication` (JWT validate) + `AuthorizationPolicy` (claims-based authz). [1755][1762]

---

## 6. Traffic Management + Security

```text
Gateway -> [Envoy] -> Service A -> [Envoy] -> Service B
              |                       |
              +--> Retry              +--> mTLS
              +--> Circuit breaker    +--> AuthZ policy
              +--> Rate limit         +--> Tracing
```

### VirtualService (Retry + Timeout)

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
    - order-service
  http:
    - route:
        - destination:
            host: order-service
      retries:
        attempts: 3
        perTryTimeout: 2s
      timeout: 10s
```

Traffic management + security — mesh me combined. [1753][1764]

---

## 7. Complete Flow

```text
1. User -> Gateway
2. Gateway -> Envoy sidecar
3. Envoy -> mTLS handshake (SPIFFE identity)
4. Envoy -> AuthorizationPolicy check
5. Envoy -> Service A (mTLS)
6. Service A -> Envoy -> Service B
7. Envoy -> mTLS + policy
8. Service B -> Response
```

### Observability

```text
Envoy -> Tracing (Jaeger/Zipkin)
Envoy -> Metrics (Prometheus)
Envoy -> Access logs
```

Envoy — tracing, metrics, access logs automatic. [1753][1764]

---

## 8. Real-World Setup

```yaml
# 1. Mesh-wide STRICT mTLS
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
***
# 2. Namespace isolation
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: backend-isolation
  namespace: backend
spec:
  action: ALLOW
  rules:
    - from:
        - source:
            namespaces: ["backend"]
***
# 3. JWT validation
apiVersion: security.istio.io/v1
kind: RequestAuthentication
metadata:
  name: jwt
  namespace: backend
spec:
  jwtRules:
    - issuer: "[https://auth.example.com](https://auth.example.com)"
      jwksUri: "[https://auth.example.com/oauth2/jwks](https://auth.example.com/oauth2/jwks)"
```

---

## 9. Common Gotchas

### Gotcha 1 — STRICT Mode Too Early

```text
Problem:
Non-mesh services fail

Fix:
PERMISSIVE start -> STRICT after onboard
```

### Gotcha 2 — AuthorizationPolicy Order

```text
Problem:
DENY + ALLOW conflict

Fix:
DENY pehle evaluate hota hai
```

### Gotcha 3 — Sidecar Missing

```text
Problem:
Policy apply nahi

Fix:
Namespace label istio-injection=enabled
```

### Gotcha 4 — JWKS Unreachable

```text
Problem:
JWT validation fail

Fix:
NetworkPolicy allow JWKS endpoint
```

---

## 10. Interview Questions

| Question | Answer |
|---|---|
| Service mesh kya hai? | Networking concerns app se alag — sidecar proxies. |
| Istio architecture? | Control plane (Istiod) + data plane (Envoy). |
| Envoy kya hai? | Sidecar proxy — traffic, mTLS, policy. |
| PeerAuthentication? | mTLS mode (STRICT/PERMISSIVE). |
| AuthorizationPolicy? | Identity-based access control. |
| Citadel? | Certificate authority (SPIFFE identity). |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Automatic mTLS | Istio + PeerAuthentication |
| Zero trust | STRICT + default deny |
| JWT validation | RequestAuthentication |
| Identity-based access | AuthorizationPolicy |
| Transparent security | Service Mesh |

---

## 11. Key Takeaways

```text
1. Service mesh = sidecar pattern.

2. Envoy = data plane proxy.

3. Istiod = control plane (config + CA).

4. mTLS automatic (app unchanged).

5. STRICT = plaintext reject.

6. AuthorizationPolicy = identity-based.

7. Default deny + explicit allow.

8. RequestAuthentication = JWT.

9. Citadel = SPIFFE identity.

10. Observability built-in.
```

---

## 12. Next Part Preview

**Part 20: Zero Trust Architecture**

Isme hum cover karenge:

```text
- Zero trust deep dive
- Principles (never trust, always verify)
- Implementation patterns
- Identity-based access
- Real-world architecture
```