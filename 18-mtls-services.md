# Part 18: Microservices Security — mTLS (Mutual TLS) Between Services

> mTLS ka core — **dono sides certificates exchange + verify** — server `client-auth: need` set karta hai, client keystore me apna cert + truststore me CA cert; Spring Boot 3.1+ me **SSL Bundles** se clean config. [1743][1744][1747][1749]

---

## 1. mTLS Mental Model

```text
Normal TLS:
Client verify Server cert (one-way)

mTLS:
Client verify Server cert
Server verify Client cert
(mutual verification)
```

### Handshake Flow

```text
1. Client -> Server: ClientHello + Client Certificate
2. Server -> Client: Server Certificate
3. Server -> Client verify (CA chain)
4. Client -> Server verify (CA chain)
5. Key exchange -> Encrypted channel
```

mTLS — dono parties certificates exchange + verify, encrypted channel. [1743][1747][1749]

---

## 2. Certificate Hierarchy

```text
Root CA (self-signed)
   |
   +--> Server Cert (Service B)
   |
   +--> Client Cert (Service A)
```

### Certificate Files

| File | Content | Kahan |
|---|---|---|
| `ca-cert.pem` | Root CA public cert | Truststore (dono) |
| `server-cert.pem` | Service B cert | Server keystore |
| `server-key.pem` | Service B private key | Server keystore |
| `client-cert.pem` | Service A cert | Client keystore |
| `client-key.pem` | Service A private key | Client keystore |

---

## 3. Certificate Generation (OpenSSL)

```bash
# 1. Root CA
openssl req -x509 -newkey rsa:4096 \
  -keyout ca-key.pem -out ca-cert.pem \
  -days 365 -nodes \
  -subj "/CN=Internal-Root-CA"

# 2. Server cert (Service B)
openssl req -newkey rsa:2048 \
  -keyout server-key.pem -out server-csr.pem \
  -nodes -subj "/CN=product-service"
openssl x509 -req -in server-csr.pem \
  -CA ca-cert.pem -CAkey ca-key.pem \
  -out server-cert.pem -days 365 \
  -extfile <(echo "subjectAltName=DNS:product-service")

# 3. Client cert (Service A)
openssl req -newkey rsa:2048 \
  -keyout client-key.pem -out client-csr.pem \
  -nodes -subj "/CN=order-service"
openssl x509 -req -in client-csr.pem \
  -CA ca-cert.pem -CAkey ca-key.pem \
  -out client-cert.pem -days 365

# 4. PKCS12 conversion
openssl pkcs12 -export \
  -in server-cert.pem -inkey server-key.pem \
  -out server-keystore.p12 \
  -name product-service -passout pass:${PASSWORD}

openssl pkcs12 -export \
  -in client-cert.pem -inkey client-key.pem \
  -out client-keystore.p12 \
  -name order-service -passout pass:${PASSWORD}

# 5. Truststore (CA cert)
keytool -importcert -file ca-cert.pem \
  -keystore truststore.p12 \
  -storetype PKCS12 -storepass ${PASSWORD} \
  -alias internal-ca
```

CA -> Server cert + Client cert -> PKCS12 keystore + truststore. [1746][1748][1750]

---

## 4. Server Config (Service B)

```yaml
server:
  port: 8443
  ssl:
    enabled: true
    key-store-type: PKCS12
    key-store: classpath:server-keystore.p12
    key-store-password: ${SSL_PASSWORD}
    key-alias: product-service
    trust-store-type: PKCS12
    trust-store: classpath:truststore.p12
    trust-store-password: ${SSL_PASSWORD}
    client-auth: need
    protocol: TLS
    enabled-protocols: TLSv1.3
```

### Critical Setting

```text
client-auth: need
= Client certificate MANDATORY
= Bina cert connection fail
```

`client-auth: need` — mTLS enforced, bina client cert connection fail. [1744][1747][1749]

### Client Auth Modes

| Mode | Behavior |
|---|---|
| `need` | Cert required (mTLS) |
| `want` | Request but optional |
| `none` | No client cert |

---

## 5. Client Config (Service A)

### SSL Bundle (Spring Boot 3.1+)

```yaml
spring:
  ssl:
    bundle:
      jks:
        product-service:
          truststore:
            location: classpath:truststore.p12
            password: ${SSL_PASSWORD}
          keystore:
            location: classpath:client-keystore.p12
            password: ${SSL_PASSWORD}
```

```java
@Bean
public WebClient productWebClient(
    WebClient.Builder builder,
    WebClientSsl ssl
) {
    return builder
        .apply(ssl.fromBundle("product-service"))
        .baseUrl("[https://product-service:8443](https://product-service:8443)")
        .build();
}
```

SSL Bundles — keystore + truststore ek bundle, WebClient/RestTemplate me reuse. [1743][1744][1752]

### Traditional Config

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          product-service:
            ...
```

```java
@Bean
public RestTemplate mtlsRestTemplate() throws Exception {
    SSLContext sslContext = SSLContexts.custom()
        .loadKeyMaterial(
            new File("classpath:client-keystore.p12"),
            password.toCharArray(), password.toCharArray())
        .loadTrustMaterial(
            new File("classpath:truststore.p12"),
            password.toCharArray())
        .build();

    SSLConnectionSocketFactory socketFactory =
        new SSLConnectionSocketFactory(sslContext);

    return RestTemplateBuilder
        .basicAuthentication(...)
        .setSslContext(sslContext)
        .build();
}
```

---

## 6. X.509 Authentication (Spring Security)

```java
@Configuration
@EnableWebSecurity
public class MtlsSecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .x509(x509 -> x509
                .subjectPrincipalRegex("CN=(.*?)(?:,|$)")
                .userDetailsService(userDetailsService)
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            );
        return http.build();
    }
}
```

`x509()` DSL — cert CN se user load, Spring Security authentication. [1749]

---

## 7. Complete Flow

```text
1. Service A -> Service B (mTLS handshake)
2. Service A cert present
3. Service B cert verify (CA chain)
4. Service A cert verify (CA chain)
5. Encrypted channel
6. Service A -> Bearer token (OAuth2)
7. Service B -> JWT validate
8. Response
```

### Defense in Depth

```text
Layer 1: mTLS (transport identity)
Layer 2: OAuth2 JWT (application identity)
Layer 3: Scope check (authorization)
```

---

## 8. Certificate Rotation

```text
Problem:
Certs expire (1 year) -> Rotation zaroori

Solutions:
1. Manual: New cert + restart
2. Automated: cert-manager (K8s)
3. SPIFFE: Auto-rotate (1 hour)
```

### Cert-Manager (K8s)

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: product-service
spec:
  secretName: product-service-tls
  duration: 2160h  # 90 days
  renewBefore: 360h
  issuerRef:
    name: internal-ca
    kind: ClusterIssuer
```

Cert rotation — cert-manager (K8s) ya SPIFFE (auto), manual restart avoid. [1749]

---

## 9. Real-World Patterns

| Pattern | Use Case |
|---|---|
| mTLS only | Internal trust |
| mTLS + OAuth2 | High security |
| mTLS + SPIFFE | K8s at scale |
| Service Mesh mTLS | Istio/Linkerd managed |

### Service Mesh (Istio)

```text
Istio sidecar:
- mTLS automatic
- Cert rotation automatic
- App code unchanged
```

Service Mesh — mTLS transparent, app code unchanged. [1749]

---

## 10. Common Gotchas

### Gotcha 1 — Handshake Failure

```text
Problem:
SSL handshake exception

Fix:
Truststore me CA cert + client-auth: need
```

### Gotcha 2 — Cert Expired

```text
Problem:
Connection fail

Fix:
Rotation + monitoring
```

### Gotcha 3 — CN Mismatch

```text
Problem:
Hostname verification fail

Fix:
SAN (subjectAltName) correct
```

### Gotcha 4 — Password in Config

```text
Problem:
Keystore password hardcoded

Fix:
Vault / env vars
```

---

## 11. Interview Questions

| Question | Answer |
|---|---|
| mTLS kya hai? | Mutual TLS — dono parties verify. |
| client-auth: need? | Client cert mandatory. |
| Keystore vs Truststore? | Own cert+key vs trusted CAs. |
| SSL Bundles? | Spring Boot 3.1+ — clean SSL config. |
| Cert rotation? | cert-manager / SPIFFE. |
| mTLS + OAuth2? | Transport + application identity. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Service identity | mTLS |
| High security | mTLS + OAuth2 |
| K8s | SPIFFE + mTLS |
| Cert rotation | cert-manager / SPIFFE |
| Transparent mTLS | Service Mesh (Istio) |

---

## 12. Key Takeaways

```text
1. mTLS = mutual certificate verification.

2. client-auth: need = enforced.

3. Keystore = own cert + key.

4. Truststore = trusted CAs.

5. SSL Bundles = Spring Boot 3.1+.

6. SAN = hostname verification.

7. Rotation = cert-manager / SPIFFE.

8. mTLS + OAuth2 = defense in depth.

9. Service Mesh = transparent mTLS.

10. TLS 1.3 = modern protocol.
```

---

## 13. Next Part Preview

**Part 19: Service Mesh Security (Istio)**

Isme hum cover karenge:

```text
- Service mesh deep dive
- Istio architecture
- mTLS automatic
- Authorization policies
- Traffic management
```