# Part 32: Microservices Security — Kubernetes Security Best Practices

> Kubernetes security ka core — **restricted Pod Security Standard + default-deny NetworkPolicy + least-privilege RBAC + Falco runtime detection** — namespace-level enforcement, zero-trust networking, syscall-level threat detection. [1934][1936][1940][1942]

---

## 1. Kubernetes Security Mental Model

```text
Layers:
1. Cluster (API server, etcd)
2. Namespace (isolation)
3. Pod (runtime)
4. Network (traffic)
5. RBAC (access)
6. Runtime (detection)
```

### Security Layers

| Layer | Control |
|---|---|
| Cluster | API server auth, etcd encryption |
| Namespace | Pod Security Standards |
| Pod | securityContext |
| Network | NetworkPolicy |
| Access | RBAC |
| Runtime | Falco / eBPF |

Kubernetes security — layered defense (cluster → namespace → pod → network → RBAC → runtime). [1933][1941][1944]

---

## 2. Pod Security Standards (PSS)

```text
3 Profiles:
1. Privileged  -> No restrictions
2. Baseline    -> Minimal restrictions
3. Restricted  -> Hardened (production)
```

### Restricted Profile

```text
- runAsNonRoot: true
- allowPrivilegeEscalation: false
- seccompProfile: RuntimeDefault
- capabilities: drop ALL
- Volume types limited
```

Restricted — non-root + no privilege escalation + seccomp + limited volumes. [1934][1936][1945]

### Namespace Label

```bash
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest
```

Pod Security Admission — namespace-level enforcement. [1936][1942][1946]

---

## 3. Pod SecurityContext

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: order-service
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
```

securityContext — non-root, read-only FS, drop capabilities, seccomp. [1934][1941][1946]

### Why Each Field

| Field | Purpose |
|---|---|
| `runAsNonRoot` | Root avoid |
| `readOnlyRootFilesystem` | Tampering prevent |
| `allowPrivilegeEscalation: false` | setuid/sudo block |
| `capabilities.drop: ALL` | Linux capabilities remove |
| `seccompProfile` | Syscall filter |

---

## 4. NetworkPolicy (Default Deny)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}   # All pods
  policyTypes:
    - Ingress
    - Egress
```

Default deny — all traffic block, phir explicit allow. [1594][1937][1940]

### Allow DNS (Egress)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
```

DNS egress allow — default deny ke baad zaroori. [1940][1943]

### Allow Service-to-Service

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-order-to-payment
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: payment-service
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

NetworkPolicy — L3/L4 allow-list, label-based selection. [1935][1943]

---

## 5. RBAC (Least Privilege)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: order-service-reader
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]
***
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: order-service-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: order-service-sa
    namespace: production
roleRef:
  kind: Role
  name: order-service-reader
  apiGroup: rbac.authorization.k8s.io
```

RBAC — namespace-level Role (not ClusterRole), specific resources + verbs. [1933][1942]

### RBAC Best Practices

```text
- Role (namespace) > ClusterRole (cluster-wide)
- Wildcard (*) avoid
- ServiceAccount per service
- kubectl auth can-i --list audit
```

Least privilege — namespace RoleBindings, no wildcards, audit with `kubectl auth can-i`. [1933][1942]

---

## 6. Runtime Security (Falco)

```text
Falco (eBPF) -> Syscall monitoring -> Alert
   |
   +--> Shell spawn in container
   +--> /etc write
   +--> Privilege escalation
   +--> Unexpected outbound connection
```

Falco — syscall-level (eBPF), shell spawn / file write / privilege escalation detect. [1941][1942][1944]

### Falco Rule Example

```yaml
- rule: Shell in Container
  desc: Shell spawned in container
  condition: spawned_process and container and proc.name in (bash, sh)
  output: "Shell spawned (user=%user.name container=%container.name)"
  priority: WARNING
```

Custom Falco rule — shell spawn alert. [1942][1944]

### Falco -> SIEM

```text
Falco -> Alert -> SIEM / Slack / PagerDuty
```

---

## 7. Complete Flow

```text
1. Namespace label (restricted PSS)
2. Pod securityContext (non-root, seccomp)
3. NetworkPolicy (default deny + allow)
4. RBAC (least privilege)
5. Falco (runtime detection)
6. Alert -> SIEM
```

---

## 8. Real-World Patterns

| Pattern | Use Case |
|---|---|
| Restricted PSS | Production namespace |
| Default-deny NetworkPolicy | Zero-trust |
| Namespace-level RBAC | Multi-tenant |
| Falco | Runtime detection |
| Service mesh (Istio) | mTLS + policy |

### Decision Framework

```text
Production? -> Restricted PSS + default-deny
Multi-tenant? -> Namespace isolation + RBAC
Runtime threats? -> Falco
mTLS? -> Istio / Linkerd
```

---

## 9. Common Gotchas

### Gotcha 1 — Default Allow Network

```text
Problem:
All pods talk to all

Fix:
Default-deny + explicit allow
```

### Gotcha 2 — ClusterRole Everywhere

```text
Problem:
Over-permissive

Fix:
Namespace Role + RoleBinding
```

### Gotcha 3 — No PSS

```text
Problem:
Root containers allowed

Fix:
Restricted PSS label
```

### Gotcha 4 — No Runtime Detection

```text
Problem:
Compromise invisible

Fix:
Falco / eBPF
```

---

## 10. Interview Questions

| Question | Answer |
|---|---|
| Pod Security Standards? | Privileged / Baseline / Restricted. |
| Restricted profile? | Non-root + no escalation + seccomp. |
| NetworkPolicy default? | Allow all (unless policy). |
| Default deny? | podSelector: {} + no rules. |
| RBAC best practice? | Namespace Role, no wildcard. |
| Falco kya? | eBPF runtime security. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Production hardening | Restricted PSS |
| Zero-trust network | Default-deny + allow-list |
| Multi-tenant | Namespace + RBAC |
| Runtime threat | Falco |
| mTLS | Service mesh |

---

## 11. Key Takeaways

```text
1. Restricted PSS = production standard.

2. securityContext = non-root + seccomp.

3. Default-deny NetworkPolicy = zero-trust.

4. RBAC = namespace Role, no wildcard.

5. Falco = syscall detection.

6. Layered defense = PSS + Network + RBAC.

7. Audit = kubectl auth can-i.

8. Service mesh = mTLS.

9. Secrets = Vault / K8s Secrets.

10. Runtime = detect + alert.
```

---

## 12. Next Part Preview

**Part 33: Supply Chain Security (SBOM/Sigstore)**

Isme hum cover karenge:

```text
- Supply chain security deep dive
- SBOM (CycloneDX/SPDX)
- Sigstore / Cosign
- SLSA framework
- Dependency scanning
```