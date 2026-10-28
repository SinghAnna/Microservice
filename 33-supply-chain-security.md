# Part 33: Microservices Security — Supply Chain Security (SBOM/Sigstore/SLSA)

> Supply chain security ka core — **SBOM (dependency inventory) + Sigstore keyless signing (identity-based, no long-lived keys) + SLSA provenance (build integrity)** — image sign karo, SBOM attach karo, deploy se pehle verify karo. [1947][1949][1951][1956]

---

## 1. Supply Chain Security Mental Model

```text
Attack:
Malicious dependency -> Build -> Deploy -> Compromise

Defense:
1. SBOM (kya andar hai)
2. Dependency scan (CVE check)
3. Image signing (kisne banaya)
4. Provenance (kaise bana)
5. Verify at deploy (policy gate)
```

### Supply Chain Threats

| Threat | Example |
|---|---|
| Dependency confusion | Typosquatting package |
| Malicious dependency | event-stream incident |
| Build tampering | CI compromise |
| Image tampering | Registry substitution |
| Artifact substitution | Post-build swap |

Supply chain — dependency + build + artifact integrity. [1956][1959][1961]

---

## 2. SBOM (Software Bill of Materials)

```text
SBOM = "Image me kya-kya hai" (inventory)
```

### SBOM Standards

| Standard | Body | Use Case |
|---|---|---|
| CycloneDX | OWASP | Security-focused |
| SPDX | Linux Foundation | Compliance (ISO/IEC 5962) |

CycloneDX — security-focused BOM; SPDX — ISO/IEC 5962 standard. [1949][1952][1954]

### Generate SBOM (Syft)

```bash
syft packages myapp:latest -o cyclonedx-json > sbom.cdx.json
syft packages myapp:latest -o spdx-json > sbom.spdx.json
```

Syft — SBOM generate (CycloneDX/SPDX format). [1955][1956]

### SBOM Contents

```json
{
  "components": [
    {
      "name": "spring-boot",
      "version": "3.4.0",
      "purl": "pkg:maven/org.springframework.boot/spring-boot@3.4.0"
    }
  ]
}
```

SBOM — components + versions + purl (package URL). [1952][1954]

### SBOM Use Cases

```text
- Vulnerability correlation (CVE -> affected component)
- License compliance
- Incident response (Log4Shell jaisa)
- Policy enforcement
```

SBOM — Log4Shell jaise incidents me fast impact analysis. [1954][1956]

---

## 3. Sigstore / Cosign (Keyless Signing)

```text
Traditional:
Private key manage -> Leak risk

Keyless:
OIDC identity -> Fulcio cert (short-lived) -> Rekor log
```

### Keyless Flow

```text
1. CI OIDC token
2. Fulcio -> short-lived cert
3. Sign image
4. Rekor -> transparency log
5. Verify (identity-based)
```

Keyless — OIDC identity, Fulcio short-lived cert, Rekor transparency log, no long-lived key. [1947][1953][1958]

### Sign Image

```bash
cosign sign --yes ghcr.io/org/myapp@sha256:abc123...
```

### Verify Image

```bash
cosign verify \
  --certificate-identity-regexp "^[https://github.com/org/repo/.github/workflows/build.yml@refs/heads/main$](https://github.com/org/repo/.github/workflows/build.yml@refs/heads/main$)" \
  --certificate-oidc-issuer "[https://token.actions.githubusercontent.com](https://token.actions.githubusercontent.com)" \
  ghcr.io/org/myapp@sha256:abc123...
```

`cosign verify` — identity + OIDC issuer match check. [1950][1960]

### Attest SBOM

```bash
cosign attest --yes \
  --predicate sbom.spdx.json \
  --type spdxjson \
  ghcr.io/org/myapp@sha256:abc123...
```

SBOM attestation — image ke saath attach. [1956][1960]

---

## 4. SLSA Framework

```text
SLSA = Supply-chain Levels for Software Artifacts

Build L1: Provenance exists
Build L2: Hosted build + signed provenance
Build L3: Hardened, isolated builder
```

### SLSA Levels

| Level | Requirement |
|---|---|
| L1 | Provenance exists (self-generated OK) |
| L2 | Hosted platform + signed provenance |
| L3 | Isolated builder, no persistent creds |

SLSA — L1 (provenance exists) → L2 (hosted + signed) → L3 (hardened builder). [1951][1959][1961]

### Provenance Contents

```json
{
  "buildDefinition": {
    "externalParameters": {
      "source": "github.com/org/repo@refs/heads/main"
    }
  },
  "runDetails": {
    "builder": { "id": "[https://github.com/actions/runner](https://github.com/actions/runner)" }
  },
  "resolvedDependencies": [...]
}
```

Provenance — source, builder, dependencies, artifact digest. [1948][1951][1957]

### Generate Provenance (GitHub Actions)

```yaml
- uses: actions/attest-build-provenance@v1
  with:
    subject-name: ghcr.io/org/myapp
    subject-digest: ${{ steps.build.outputs.digest }}
    push-to-registry: true
```

`actions/attest-build-provenance` — SLSA provenance generate + sign. [1956][1959]

### Verify Provenance

```bash
slsa-verifier verify-image \
  ghcr.io/org/myapp@sha256:abc123... \
  --source-uri github.com/org/repo \
  --source-tag v1.0.0
```

`slsa-verifier` — provenance signature + builder identity + digest binding check. [1959][1961]

---

## 5. Complete CI/CD Flow

```text
1. Code push -> CI trigger
2. Dependency scan (Trivy/Grype)
3. Build image (multi-stage)
4. SBOM generate (Syft)
5. Image sign (Cosign keyless)
6. SBOM attest (Cosign)
7. Provenance attest (SLSA)
8. Push to registry
9. Deploy gate: verify signature + SBOM + provenance
```

### GitHub Actions Example

```yaml
- name: Build image
  run: docker build -t ghcr.io/org/myapp:${{ github.sha }} .

- name: Generate SBOM
  run: syft packages ghcr.io/org/myapp:${{ github.sha }} -o spdx-json > sbom.spdx.json

- name: Sign image
  run: cosign sign --yes ghcr.io/org/myapp@${{ github.sha }}

- name: Attest SBOM
  run: cosign attest --yes --predicate sbom.spdx.json --type spdxjson ghcr.io/org/myapp@${{ github.sha }}

- name: Attest provenance
  uses: actions/attest-build-provenance@v1
  with:
    subject-name: ghcr.io/org/myapp
    subject-digest: ${{ steps.build.outputs.digest }}
    push-to-registry: true
```

Full pipeline — scan → build → SBOM → sign → attest → push. [1953][1956][1960]

---

## 6. Deploy-Time Verification

```text
Admission Controller (Kyverno/OPA)
   |
   +--> Check: cosign signature valid?
   +--> Check: SBOM attached?
   +--> Check: SLSA provenance L2+?
   |
   +--> Fail -> Pod reject
```

Admission policy — unsigned / unattested image reject. [1956][1959][1961]

### Kyverno Policy Example

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-signature
      match:
        resources:
          kinds: [Pod]
      verifyImages:
        - imageReferences: ["ghcr.io/org/*"]
          attestors:
            - entries:
                - keyless:
                    subject: "[https://github.com/org/repo/.github/workflows/build.yml@refs/heads/main](https://github.com/org/repo/.github/workflows/build.yml@refs/heads/main)"
                    issuer: "[https://token.actions.githubusercontent.com](https://token.actions.githubusercontent.com)"
```

Kyverno — signature verify at admission, fail = reject. [1956][1961]

---

## 7. Dependency Scanning

```bash
# Trivy (SBOM + image + repo)
trivy sbom sbom.cdx.json
trivy fs --scanners vuln,secret .

# Grype (SBOM-based)
grype sbom:sbom.cdx.json
```

Trivy/Grype — SBOM se vulnerability scan. [1919][1926][1929]

### Dependency Pinning

```text
- package-lock.json / pom.xml lock
- Digest-pinned base images
- Renovate/Dependabot auto-update
```

---

## 8. Real-World Patterns

| Pattern | Use Case |
|---|---|
| Syft + CycloneDX | SBOM generation |
| Cosign keyless | CI signing |
| SLSA L3 | High-assurance builds |
| Kyverno verify | Deploy gate |
| Renovate | Dependency updates |

### Decision Framework

```text
Compliance-heavy? -> SBOM + SLSA L3
Standard? -> SBOM + Cosign keyless
GitOps? -> Kyverno verify at admission
Fast? -> SBOM + Trivy scan
```

---

## 9. Common Gotchas

### Gotcha 1 — SBOM Generate but Not Attest

```text
Problem:
SBOM alag file, image se link nahi

Fix:
cosign attest (image ke saath)
```

### Gotcha 2 — Long-Lived Signing Keys

```text
Problem:
Key leak risk

Fix:
Keyless signing (OIDC + Fulcio)
```

### Gotcha 3 — No Deploy Gate

```text
Problem:
Unsigned image deploy

Fix:
Kyverno/OPA admission policy
```

### Gotcha 4 — Provenance Self-Generated

```text
Problem:
Build script khud provenance banaye

Fix:
Hosted platform + platform-signed (SLSA L2+)
```

---

## 10. Interview Questions

| Question | Answer |
|---|---|
| SBOM kya hai? | Software Bill of Materials — dependency inventory. |
| CycloneDX vs SPDX? | Security-focused vs ISO compliance. |
| Sigstore keyless? | OIDC identity + Fulcio + Rekor, no long-lived key. |
| SLSA kya hai? | Supply-chain Levels for Software Artifacts. |
| SLSA L2 vs L3? | Hosted+signed vs isolated builder. |
| Kyverno verify? | Admission-time signature check. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Log4Shell impact | SBOM query |
| Image tampering | Cosign verify |
| Build integrity | SLSA provenance |
| Unsigned deploy | Kyverno policy |
| Dependency CVE | Trivy sbom scan |

---

## 11. Key Takeaways

```text
1. SBOM = dependency inventory.

2. CycloneDX = security, SPDX = compliance.

3. Syft = SBOM generate.

4. Cosign keyless = OIDC + Fulcio + Rekor.

5. SLSA = build integrity levels.

6. Provenance = signed build attestation.

7. Kyverno = deploy-time verify.

8. Trivy = SBOM-based scan.

9. Dependency pinning = reproducible.

10. Full chain = SBOM + sign + attest + verify.
```

---

## 12. Next Part Preview

**Part 34: Security Testing (DAST/SAST)**

Isme hum cover karenge:

```text
- Security testing deep dive
- SAST (static analysis)
- DAST (dynamic testing)
- Penetration testing
- Security testing in CI/CD
```