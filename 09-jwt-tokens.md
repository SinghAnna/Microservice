# Part 9: Microservices Security — JWT Tokens (Structure, Claims, Validation)

> JWT ka core — **Header + Payload (claims) + Signature** — RFC 7519 ke 7 registered claims (`iss`, `sub`, `aud`, `exp`, `nbf`, `iat`, `jti`), aur microservices me **RS256/ES256 recommended** (HS256 nahi). [1638][1639][1143][1647]

---

## 1. JWT Mental Model

```text
JWT = Header.Payload.Signature (Base64URL encoded)

Header:
{
  "alg": "RS256",
  "kid": "key-1",
  "typ": "JWT"
}

Payload (Claims):
{
  "iss": "[https://auth.example.com](https://auth.example.com)",
  "sub": "user-123",
  "aud": "order-service",
  "exp": 1735689600,
  "iat": 1735686000,
  "jti": "unique-id",
  "scope": "orders:read"
}

Signature:
RS256(base64(header) + "." + base64(payload), private_key)
```

JWT 3 parts — header (metadata), payload (claims), signature (integrity). [1638][1640][1642]

---

## 2. Registered Claims (RFC 7519)

| Claim | Name | Purpose | Validation |
|---|---|---|---|
| `iss` | Issuer | Kaun issue kiya | Expected issuer match |
| `sub` | Subject | Kaun (user/service ID) | — |
| `aud` | Audience | Kis ke liye | Service-specific match |
| `exp` | Expiration | Kab expire | Not expired |
| `nbf` | Not Before | Kab se valid | Not before |
| `iat` | Issued At | Kab issue | — |
| `jti` | JWT ID | Unique ID | Replay prevention |

### Critical Claims

```text
iss  -> "Kaun issue kiya" (trust anchor)
aud  -> "Kis ke liye" (audience validation)
exp  -> "Kab expire" (mandatory check)
jti  -> "Unique ID" (replay prevention)
```

RFC 7519 ke 7 registered claims — `aud` present ho to MUST reject agar aap audience nahi. [1638][1639][1643]

---

## 3. Custom Claims

```json
{
  "roles": ["USER", "ADMIN"],
  "tenant_id": "tenant-1",
  "department": "engineering",
  "permissions": ["orders:read", "orders:write"]
}
```

### Custom Claims Best Practices

```text
1. Chhote rakho (token size)
2. Sensitive data nahi (JWT readable)
3. Standard names use karo
4. Namespacing (custom namespace)
```

Custom claims — roles, tenant, permissions — lekin sensitive data nahi (JWT base64 readable). [1639][1640]

---

## 4. Signing Algorithms

| Algorithm | Type | Key | Use Case |
|---|---|---|---|
| HS256 | Symmetric (HMAC) | Shared secret | Single app |
| RS256 | Asymmetric (RSA) | Private + Public | Microservices |
| ES256 | Asymmetric (ECDSA) | Private + Public | Microservices (smaller) |
| EdDSA | Asymmetric (Ed25519) | Private + Public | Modern |

### HS256 vs RS256

| Aspect | HS256 | RS256 |
|---|---|---|
| Key | Same secret (sign + verify) | Private (sign) + Public (verify) |
| Key distribution | Har service ko secret | Sirf public key (JWKS) |
| Key rotation | Har service redeploy | JWKS update |
| Recommendation | Single app | Microservices |

RS256/ES256 asymmetric — sirf public key share, private key sirf Auth Server. [1143][1144][1641][1647]

### ES256 vs RS256

| Aspect | RS256 | ES256 |
|---|---|---|
| Key size | 2048-bit | 256-bit |
| Signature size | 256 bytes | 64 bytes |
| Performance | Slower | Faster |
| Recommendation | Common | Modern (smaller tokens) |

ES256 — chhote keys + signatures, faster — modern recommendation. [1144][1646]

---

## 5. JWT Validation Steps

```text
1. Structure check (3 parts)
2. Header parse (alg, kid)
3. alg whitelist (RS256/ES256, never none)
4. Signature verify (JWKS public key)
5. iss check (expected issuer)
6. aud check (service-specific)
7. exp check (not expired)
8. nbf check (not before)
9. scope/roles check (authorization)
```

### Spring Security Validation

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: [https://auth.example.com](https://auth.example.com)
          audience: order-service
```

```java
@Bean
public JwtDecoder jwtDecoder() {
    NimbusJwtDecoder decoder = JwtDecoders.fromIssuerLocation(
        "[https://auth.example.com](https://auth.example.com)"
    );
    decoder.setJwtValidator(new DelegatingOAuth2TokenValidator<>(
        new JwtTimestampValidator(),
        new JwtIssuerValidator("[https://auth.example.com](https://auth.example.com)"),
        new JwtAudienceValidator("order-service")
    ));
    return decoder;
}
```

Spring Security — signature + iss + exp automatic, `aud` explicitly configure karna hota hai. [1614][1645][1647]

---

## 6. Audience Validation (Critical)

```text
Auth Server:
- order-service token: aud = ["order-service"]
- product-service token: aud = ["product-service"]

Order Service:
- aud must contain "order-service"
- Product token reject (audience mismatch)
```

### Kyu Zaroori

```text
Bina audience validation:
Order token Product service me use ho sakta hai
Token replay across services

With audience validation:
Token sirf intended service ke liye
```

Audience validation — token replay across services prevent. [1639][1645][1647]

---

## 7. JWKS (JSON Web Key Set)

```text
Auth Server -> /oauth2/jwks
   |
   v
Public Keys (RSA/EC)
   |
   v
Services -> Cache -> Verify
```

### JWKS Response

```json
{
  "keys": [
    {
      "kty": "RSA",
      "kid": "key-1",
      "use": "sig",
      "n": "...",
      "e": "AQAB"
    }
  ]
}
```

JWKS — public keys endpoint, services cache + verify. [1629][1634]

### Key Rotation

```text
1. Auth Server new key generate
2. JWKS me add (kid: key-2)
3. New tokens kid: key-2
4. Services JWKS refresh
5. Old key retire
```

---

## 8. Best Practices

| Practice | Detail |
|---|---|
| Short-lived access tokens | 5-15 min |
| RS256/ES256 | HS256 nahi (microservices) |
| Audience validation | Har service |
| `alg: none` block | Whitelist algorithms |
| Sensitive data nahi | JWT readable |
| `jti` for revocation | Replay prevention |
| JWKS caching | Performance |
| Key rotation | Regular |

---

## 9. Common Gotchas

### Gotcha 1 — `alg: none` Accepted

```text
Problem:
Unsigned token accept

Fix:
Algorithm whitelist (RS256 only)
```

### Gotcha 2 — Audience Not Validated

```text
Problem:
Token cross-service replay

Fix:
JwtAudienceValidator
```

### Gotcha 3 — Sensitive Data in JWT

```text
Problem:
Password/PII in payload

Fix:
Sirf claims (roles, tenant), sensitive data nahi
```

### Gotcha 4 — Long-Lived Tokens

```text
Problem:
Compromise = long window

Fix:
Short-lived (5-15 min) + refresh
```

---

## 10. Interview Questions

| Question | Answer |
|---|---|
| JWT structure? | Header + Payload + Signature. |
| Registered claims? | iss, sub, aud, exp, nbf, iat, jti. |
| HS256 vs RS256? | Symmetric vs Asymmetric. |
| Audience validation kyu? | Token replay prevent. |
| JWKS kya hai? | Public keys endpoint. |
| `alg: none` attack? | Unsigned token forgery. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Microservices JWT | RS256/ES256 + JWKS |
| Token replay | Audience validation + jti |
| Key rotation | JWKS + kid |
| Sensitive data | JWT me nahi (encryption) |
| Short-lived tokens | 5-15 min + refresh |

---

## 11. Key Takeaways

```text
1. JWT = Header + Payload + Signature.

2. 7 registered claims (RFC 7519).

3. aud = audience validation critical.

4. RS256/ES256 = microservices.

5. HS256 = single app only.

6. JWKS = public keys.

7. alg whitelist (never none).

8. Short-lived + refresh.

9. Sensitive data nahi JWT me.

10. jti = replay prevention.
```

---

## 12. Next Part Preview

**Part 10: Token Lifecycle — Issue, Refresh, Revoke**

Isme hum cover karenge:

```text
- Token lifecycle deep dive
- Token issuance
- Refresh token rotation
- Token revocation
- Blacklist / whitelist
- Real-world strategies
```