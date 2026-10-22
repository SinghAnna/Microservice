# Part 30: Microservices Security — Secrets Management (Vault/K8s)

> Secrets management ka core — **Vault dynamic secrets (short-lived, auto-rotate) + Kubernetes auth (no static credentials)** — pod apne Service Account token se Vault authenticate karta hai, Vault role-based policy se sirf apne secrets read karta hai. [1903][1904][1914][1917]

---

## 1. Secrets Management Mental Model

```text
Wrong:
application.properties me password
Git me commit -> Leak -> Compromise

Right:
Vault / K8s Secrets
Runtime fetch
Short-lived / rotated
```

### Secret Types

| Type | Examples | Lifetime |
|---|---|---|
| Static | DB password, API key | Long (rotated) |
| Dynamic | DB credentials (per-request) | Short (minutes/hours) |
| Certificates | TLS certs | 90 days |
| Tokens | OAuth2, JWT | Minutes/hours |

Vault dynamic secrets — DB credentials on-demand, short TTL, auto-expire. [1917][1910]

---

## 2. Vault Architecture

```text
Vault Server
   |
   +--> Secrets Engines (KV, Database, PKI)
   +--> Auth Methods (Kubernetes, OIDC, AppRole)
   +--> Policies (path-based access)
   +--> Audit Log
```

### Components

| Component | Purpose |
|---|---|
| Secrets Engine | Secrets store/generate |
| Auth Method | Identity verify |
| Policy | Access control |
| Audit Log | Tamper-proof trail |

Vault — secrets engines + auth methods + policies + audit. [1903][1917]

---

## 3. Kubernetes Auth Method

```text
Pod -> Service Account Token (auto-mounted)
   |
   v
Vault -> TokenReview API (K8s verify)
   |
   v
Vault Role -> Policy -> Secrets
```

### Vault Setup

```bash
# Enable Kubernetes auth
vault auth enable kubernetes

# Configure K8s connection
vault write auth/kubernetes/config \
  kubernetes_host=[https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT](https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT)

# Create role (bound to service account)
vault write auth/kubernetes/role/order-service \
  bound_service_account_names=order-service-sa \
  bound_service_account_namespaces=backend \
  policies=order-service-policy \
  ttl=1h
```

Kubernetes auth — Service Account token se Vault login, role-bound policy. [1903][1905][1914]

### Policy

```hcl
# order-service-policy.hcl
path "secret/data/order-service/*" {
  capabilities = ["read"]
}
```

Least privilege — sirf apne path ke secrets read. [1914][1917]

---

## 4. Spring Cloud Vault Integration

### Dependency

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-vault-config</artifactId>
</dependency>
```

### bootstrap.yml

```yaml
spring:
  application:
    name: order-service
  cloud:
    vault:
      uri: [https://vault.example.com:8200](https://vault.example.com:8200)
      authentication: KUBERNETES
      kubernetes:
        role: order-service
        kubernetes-path: kubernetes
        service-account-token-file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kv:
        backend: secret
        default-context: order-service
```

Spring Cloud Vault — `KUBERNETES` auth, Service Account token auto-mounted. [1904][1908][1909]

### Secret Usage

```java
@Value("${db.password}")
private String dbPassword;  // Vault se inject

@Value("${jwt.secret}")
private String jwtSecret;
```

`@Value` — Vault secrets automatic inject (Environment me). [1904][1914]

---

## 5. Dynamic Secrets (Database)

```text
App -> Vault (DB credentials request)
   |
   v
Vault -> DB (temporary user create)
   |
   v
Credentials (TTL: 1 hour)
   |
   v
App -> DB (temporary credentials)
   |
   v
TTL expire -> Vault auto-revoke
```

### Vault DB Engine

```bash
vault secrets enable database

vault write database/config/mydb \
  plugin_name=postgresql-database-plugin \
  connection_url=postgresql://{{username}}:{{password}}@db:5432/mydb \
  allowed_roles=order-service

vault write database/roles/order-service \
  db_name=mydb \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';" \
  default_ttl=1h \
  max_ttl=24h
```

Dynamic DB credentials — per-request create, TTL expire, auto-revoke. [1917][1910]

### Benefit

```text
Static password:
Leak = permanent compromise

Dynamic credentials:
Leak = 1 hour window, auto-revoke
```

---

## 6. Kubernetes Secrets (Native)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: order-service-secrets
type: Opaque
stringData:
  db-password: ${DB_PASSWORD}
  jwt-secret: ${JWT_SECRET}
```

### Pod Mount

```yaml
spec:
  containers:
    - name: order-service
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: order-service-secrets
              key: db-password
```

### K8s Secrets Limitations

```text
- Base64 (encryption nahi, encoding)
- etcd encryption at rest chahiye
- RBAC lock down
- Rotation manual (ya ESO)
```

K8s Secrets — base64 encoding (encryption nahi), etcd encryption + RBAC zaroori. [1911][1913][1915]

### External Secrets Operator (ESO)

```text
Vault (source of truth) -> ESO -> K8s Secret (sync)
```

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: order-service
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
  target:
    name: order-service-secrets
  data:
    - secretKey: db-password
      remoteRef:
        key: order-service/db-password
```

ESO — Vault source of truth, K8s Secret sync, auto-refresh. [1913][1915]

---

## 7. Secret Rotation

| Credential | Rotation |
|---|---|
| DB passwords | 30-90 days (dynamic = auto) |
| API keys | 90 days |
| TLS certs | 90 days (cert-manager) |
| JWT signing keys | 90 days |

### Rotation Strategy

```text
1. Naya secret issue
2. Dual-validity period (old + new)
3. Consumers update
4. Old secret revoke
```

Rotation — dual-validity period, zero downtime. [1910][1911][1913]

---

## 8. Complete Flow

```text
1. Pod start -> Service Account token
2. Vault login (Kubernetes auth)
3. Vault role -> Policy check
4. Secrets fetch (dynamic/static)
5. Environment inject
6. App use
7. TTL expire -> Re-fetch / auto-revoke
```

---

## 9. Real-World Patterns

| Pattern | Use Case |
|---|---|
| Vault + K8s auth | Production standard |
| Vault dynamic secrets | DB credentials |
| ESO + Vault | GitOps-friendly |
| K8s Secrets + etcd encryption | Simple setups |
| Vault + SPIFFE | Highest security |

### Decision Framework

```text
Compliance-heavy / multi-cloud? -> Vault
GitOps? -> ESO + Vault
Simple? -> K8s Secrets + etcd encryption
K8s at scale? -> Vault + SPIFFE
```

---

## 10. Common Gotchas

### Gotcha 1 — Secrets in Git

```text
Problem:
Password commit

Fix:
Vault + secret scanning
```

### Gotcha 2 — K8s Secrets Unencrypted

```text
Problem:
etcd me plain text

Fix:
Encryption at rest enable
```

### Gotcha 3 — Over-Permissive Policy

```text
Problem:
Sab secrets readable

Fix:
Least privilege (path-based)
```

### Gotcha 4 — No Rotation

```text
Problem:
Static secrets forever

Fix:
Dynamic secrets / scheduled rotation
```

---

## 11. Interview Questions

| Question | Answer |
|---|---|
| Secrets management kya? | Secrets secure store + distribute + rotate. |
| Vault dynamic secrets? | Per-request credentials, short TTL, auto-revoke. |
| Kubernetes auth? | Service Account token se Vault login. |
| K8s Secrets limitation? | Base64 encoding, encryption nahi. |
| ESO kya hai? | External Secrets Operator — Vault -> K8s sync. |
| Rotation strategy? | Dual-validity + revoke old. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Production secrets | Vault + K8s auth |
| DB credentials | Vault dynamic secrets |
| GitOps | ESO + Vault |
| Simple setup | K8s Secrets + etcd encryption |
| Compliance | Vault + audit log |

---

## 12. Key Takeaways

```text
1. Secrets = Vault / K8s Secrets (never Git).

2. Dynamic secrets = short TTL + auto-revoke.

3. Kubernetes auth = no static credentials.

4. Policy = least privilege (path-based).

5. ESO = GitOps-friendly sync.

6. K8s Secrets = etcd encryption chahiye.

7. Rotation = dual-validity.

8. Audit log = tamper-proof.

9. Spring Cloud Vault = @Value inject.

10. Vault + SPIFFE = highest security.
```

---

## 13. Next Part Preview

**Part 31: Docker Security Best Practices**

Isme hum cover karenge:

```text
- Docker security deep dive
- Image hardening
- Non-root user
- Multi-stage builds
- Image scanning
```