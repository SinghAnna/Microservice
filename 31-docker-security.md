# Part 31: Microservices Security — Docker Security Best Practices

> Docker security ka core — **non-root user + multi-stage build + distroless base + Trivy scan (CI gate)** — attack surface minimize, build tools final image se hatao, CRITICAL CVE pe build fail. [1920][1925][1926][1931]

---

## 1. Docker Security Mental Model

```text
Wrong:
root user + fat image + secrets in Dockerfile
= Attack surface high

Right:
non-root + distroless + multi-stage + scan
= Attack surface minimal
```

### Threat Model

| Threat | Mitigation |
|---|---|
| Root escape | Non-root user |
| Image bloat / CVEs | Distroless + multi-stage |
| Secrets in image | Vault / build secrets |
| Vulnerable deps | Trivy / Grype scan |
| Tampered image | Image signing (cosign) |

Docker security — least privilege, minimal image, secrets external, scan + sign. [1922][1925][1932]

---

## 2. Non-Root User

```dockerfile
# Create dedicated user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Switch to non-root
USER appuser
```

### Why Non-Root

```text
Root container:
- Host kernel access risk
- Filesystem write anywhere
- Privilege escalation

Non-root:
- Limited permissions
- Container escape harder
```

`USER` instruction — root (UID 0) avoid, non-root default. [1925][1927][1930]

### Kubernetes Enforcement

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
```

---

## 3. Multi-Stage Build

```dockerfile
# Stage 1: Build (full tools)
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Runtime (minimal)
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Multi-stage — build tools (Maven, JDK) final image me nahi, sirf artifact. [1920][1923][1925]

### Benefit

```text
Single-stage:
- Maven + JDK + source in final image
- Size: 800MB+
- CVEs: high

Multi-stage:
- Sirf JRE + jar
- Size: 200MB
- CVEs: low
```

---

## 4. Distroless Base Image

```dockerfile
FROM gcr.io/distroless/java21-debian12
COPY --from=builder /app/target/*.jar /app/app.jar
ENTRYPOINT ["/usr/bin/java", "-jar", "/app/app.jar"]
```

### Distroless = Minimal

```text
No shell (sh/bash)
No package manager (apt/yum)
No curl/wget
Sirf app + runtime deps
```

Distroless — shell, package manager, unnecessary tools removed — attack surface minimal. [1920][1923][1928]

### ENTRYPOINT Vector Form

```dockerfile
# WRONG (shell prefix)
ENTRYPOINT java -jar app.jar

# RIGHT (vector form, shell nahi)
ENTRYPOINT ["/usr/bin/java", "-jar", "/app/app.jar"]
```

Distroless me shell nahi — ENTRYPOINT vector form mandatory. [1928]

---

## 5. Image Scanning (Trivy)

### Basic Scan

```bash
trivy image myapp:latest
```

### CI Gate (Fail on CRITICAL)

```bash
trivy image --exit-code 1 --severity CRITICAL myapp:latest
```

### Full Scan (Vuln + Secret + Misconfig)

```bash
trivy image --scanners vuln,secret,misconfig myapp:latest
```

Trivy — vulnerabilities + secrets + misconfigurations, `--exit-code 1` CI fail. [1919][1921][1926]

### Grype (Alternative)

```bash
grype myapp:latest --only-fixed --fail-on high
```

Grype — `--only-fixed` (fixable CVEs only), `--fail-on high` CI gate. [1921][1926][1929]

### CI Integration

```yaml
# GitHub Actions
- name: Scan image
  run: trivy image --exit-code 1 --severity CRITICAL ${{ env.IMAGE }}
```

---

## 6. Secrets Handling

```text
WRONG:
ENV DB_PASSWORD=secret123
ARG API_KEY=abc123

RIGHT:
- Build secrets (--mount=type=secret)
- Runtime: Vault / K8s Secrets
```

### Build Secrets (Docker BuildKit)

```dockerfile
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) npm install
```

```bash
docker build --secret id=npm_token,src=.npmrc .
```

Secrets Dockerfile me kabhi nahi — build secrets ya Vault/K8s Secrets. [1922][1927][1932]

### .dockerignore

```text
.git
.env
*.pem
*.key
node_modules
target
```

`.dockerignore` — secrets + build artifacts image me na jayein. [1924][1927]

---

## 7. Complete Secure Dockerfile

```dockerfile
# syntax=docker/dockerfile:1.7
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests

FROM gcr.io/distroless/java21-debian12
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar

# Non-root user
USER nonroot:nonroot

EXPOSE 8080
HEALTHCHECK --interval=30s CMD ["/usr/bin/java", "-jar", "/app/app.jar", "--health-check"]
ENTRYPOINT ["/usr/bin/java", "-jar", "/app/app.jar"]
```

### Checklist

```text
- Multi-stage build
- Distroless base
- Non-root user
- ENTRYPOINT vector form
- No secrets in image
- .dockerignore
- Trivy scan (CI gate)
- Image signing (cosign)
```

---

## 8. Image Signing (Cosign)

```bash
# Sign image
cosign sign --key cosign.key myapp:latest

# Verify before deploy
cosign verify --key cosign.pub myapp:latest
```

Cosign — image signing + verification (supply chain security). [1922][1928]

---

## 9. Real-World Patterns

| Pattern | Use Case |
|---|---|
| Distroless + multi-stage | Production Java/Go |
| Alpine + multi-stage | Lightweight |
| Trivy CI gate | Every build |
| Cosign signing | Supply chain |
| BuildKit secrets | Build-time secrets |

### Decision Framework

```text
Java/Go production? -> Distroless + multi-stage
Lightweight? -> Alpine + multi-stage
CI gate? -> Trivy (--exit-code 1)
Supply chain? -> Cosign signing
```

---

## 10. Common Gotchas

### Gotcha 1 — Root User Default

```text
Problem:
Root container

Fix:
USER nonroot
```

### Gotcha 2 — Fat Image

```text
Problem:
Build tools in final image

Fix:
Multi-stage + distroless
```

### Gotcha 3 — Secrets in ENV

```text
Problem:
docker history me visible

Fix:
Vault / build secrets
```

### Gotcha 4 — No Scan

```text
Problem:
CVEs production me

Fix:
Trivy CI gate
```

### Gotcha 5 — latest Tag

```text
Problem:
Non-reproducible builds

Fix:
Pinned tags (e.g., 21-jre-alpine)
```

---

## 11. Interview Questions

| Question | Answer |
|---|---|
| Docker security kya? | Image + runtime hardening. |
| Non-root kyun? | Privilege escalation avoid. |
| Multi-stage build? | Build tools final image se hatao. |
| Distroless kya? | Minimal image (no shell/pkg mgr). |
| Trivy kya hai? | Vulnerability + secret + misconfig scanner. |
| Cosign kya hai? | Image signing/verification. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Production image | Distroless + multi-stage |
| CVE gate | Trivy --exit-code 1 |
| Secrets in build | BuildKit secrets |
| Supply chain | Cosign signing |
| Lightweight | Alpine + multi-stage |

---

## 12. Key Takeaways

```text
1. Non-root user = default.

2. Multi-stage = build/runtime separate.

3. Distroless = minimal attack surface.

4. Trivy = CI gate (CRITICAL fail).

5. Secrets = Vault / build secrets.

6. .dockerignore = secrets exclude.

7. ENTRYPOINT = vector form.

8. Cosign = image signing.

9. Pinned tags = reproducible.

10. Scan on every build.
```

---

## 13. Next Part Preview

**Part 32: Kubernetes Security Best Practices**

Isme hum cover karenge:

```text
- Kubernetes security deep dive
- Pod Security Standards
- Network Policies
- RBAC
- Runtime security
```