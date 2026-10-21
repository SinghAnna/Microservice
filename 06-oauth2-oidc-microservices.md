# Part 6: Microservices Security — OAuth2 & OIDC in Microservices

> Microservices me OAuth2/OIDC ka core pattern — **Gateway OAuth2 login + Token Relay + har service JWT validate** — token types (access/refresh/ID) ka clear separation aur har request pe validation. [1172][1611][1613][1615]

---

## 1. OAuth2/OIDC Mental Model

```text
OAuth2 = Authorization ("kya access de sakte hain")
OIDC   = Authentication ("kaun hai") + OAuth2

Microservices me:
Gateway = OAuth2 Client (login handle)
Services = Resource Servers (JWT validate)
Auth Server = Token issue (Keycloak/Spring Auth Server)
```

### Token Flow

```text
User -> Gateway (login)
Gateway -> Auth Server (auth code)
Gateway -> Tokens (access + refresh + ID)
Gateway -> Service (access token relay)
Service -> JWT validate
```

OAuth2 authorization, OIDC authentication + identity — microservices me Gateway client, services resource servers. [1172][1612][1615]

---

## 2. Token Types

| Token | Format | Purpose | Lifetime |
|---|---|---|---|
| Access Token | JWT (usually) | API access | Short (5-15 min) |
| Refresh Token | Opaque / JWT | New access token | Long (days/weeks) |
| ID Token | JWT | User identity (client only) | Short |

### Kaun Kaun Use Karta Hai

| Token | Gateway | Service | Client |
|---|---|---|---|
| Access | Relay | Validate | Store |
| Refresh | Store | Nahi | Nahi |
| ID | User info | Nahi | User profile |

Access token = API calls, Refresh = renewal, ID = user identity (sirf client). [1172][1608][1609][1612]

---

## 3. JWT Structure

```text
Header.Payload.Signature

Header:
{
  "alg": "RS256",
  "kid": "key-id-1",
  "typ": "JWT"
}

Payload (Claims):
{
  "iss": "[https://auth.example.com](https://auth.example.com)",
  "sub": "user-123",
  "aud": "order-service",
  "exp": 1735689600,
  "iat": 1735686000,
  "scope": "orders:read orders:write",
  "roles": ["USER"]
}

Signature:
RS256(header.payload, private_key)
```

### Critical Claims

| Claim | Meaning | Validation |
|---|---|---|
| `iss` | Issuer | Expected issuer match |
| `sub` | Subject (user ID) | — |
| `aud` | Audience | Service-specific |
| `exp` | Expiry | Not expired |
| `iat` | Issued at | — |
| `scope` | Permissions | Endpoint check |
| `roles` | Roles | Role check |

JWT = header + payload (claims) + signature — `iss`, `aud`, `exp`, `scope` validate karna zaroori. [1611][1612][1614][1615]

---

## 4. Gateway OAuth2 Login

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          gateway:
            client-id: gateway-client
            client-secret: ${GATEWAY_SECRET}
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            scope:
              - openid
              - profile
        provider:
          gateway:
            issuer-uri: [https://auth.example.com](https://auth.example.com)
```

```java
@Configuration
@EnableWebSecurity
public class GatewaySecurityConfig {

    @Bean
    public SecurityWebFilterChain filterChain(ServerHttpSecurity http) {
        http
            .csrf(ServerHttpSecurity.CsrfSpec::disable)
            .authorizeExchange(exchange -> exchange
                .pathMatchers("/api/public/**").permitAll()
                .anyExchange().authenticated()
            )
            .oauth2Login(Customizer.withDefaults());
        return http.build();
    }
}
```

Gateway OAuth2 client — login handle karta hai, tokens store karta hai. [1611][1613][1615]

---

## 5. Token Relay

```yaml
spring:
  cloud:
    gateway:
      default-filters:
        - TokenRelay=
```

### Flow

```text
1. User -> Gateway (Bearer access token)
2. Gateway validate
3. TokenRelay filter
4. Gateway -> Service (Same access token)
5. Service validate
```

`TokenRelay=` — current user ka access token downstream forward. [1613][1615]

---

## 6. Service JWT Validation

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
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class ServiceSecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/orders/**")
                .hasAuthority("SCOPE_orders:read")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(Customizer.withDefaults())
            );
        return http.build();
    }
}
```

Service JWT validate — signature (JWKS), `iss`, `aud`, `exp`, `scope`. [1611][1614][1615]

### Validation Steps

```text
1. Signature verify (JWKS se public key)
2. alg check (RS256, never none)
3. iss check (expected issuer)
4. aud check (service-specific)
5. exp check (not expired)
6. scope/roles check (endpoint)
```

---

## 7. Audience Validation

```text
Auth Server:
- order-service token: aud = "order-service"
- product-service token: aud = "product-service"
```

### Kyu Zaroori

```text
Order token Product service me use ho sakta hai (audience mismatch)
Audience validation = token sirf intended service ke liye
```

`aud` claim — token kis service ke liye hai — audience validation critical. [1612][1614]

---

## 8. Refresh Token Strategy

```text
Gateway/BFF:
- Refresh token server-side store (session/DB)
- Browser ko sirf session cookie
- Access token short-lived

SPA:
- Refresh token HttpOnly cookie
- ya BFF pattern
```

### BFF Pattern (Recommended)

```text
SPA -> BFF (Gateway) -> Auth Server
   |                        |
   +-- session cookie ------+
   
BFF tokens server-side store
Browser me token nahi
```

Refresh token server-side (BFF) — browser me expose nahi. [1613][1615]

---

## 9. Complete Flow

```text
1. User -> Gateway /login
2. Gateway -> Auth Server (auth code)
3. Gateway -> Tokens (access + refresh + ID)
4. Gateway session me store
5. User -> Gateway -> Service (Token Relay)
6. Service JWT validate (signature, iss, aud, exp, scope)
7. Service -> Response
8. Access token expire -> Gateway refresh token se naya
9. User ko re-login nahi karna
```

---

## 10. Real-World Setup

```text
Auth Server: Keycloak
Gateway: Spring Cloud Gateway (OAuth2 client + Token Relay)
Services: Spring Boot Resource Servers (JWT validate)
Frontend: SPA (session cookie with BFF)
```

### Keycloak Integration

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          keycloak:
            client-id: gateway-client
            client-secret: ${GATEWAY_SECRET}
            authorization-grant-type: authorization_code
            scope:
              - openid
        provider:
          keycloak:
            issuer-uri: [https://keycloak.example.com/realms/my-realm](https://keycloak.example.com/realms/my-realm)
```

---

## 11. Common Gotchas

### Gotcha 1 — Audience Mismatch

```text
Problem:
401 on service

Fix:
aud claim + audience config match
```

### Gotcha 2 — Clock Skew

```text
Problem:
Token expire jaldi lagta hai

Fix:
Clock skew tolerance (30-60 sec)
```

### Gotcha 3 — JWKS Cache

```text
Problem:
Key rotation ke baad fail

Fix:
JWKS cache refresh config
```

### Gotcha 4 — Refresh Token in Browser

```text
Problem:
XSS se token steal

Fix:
BFF pattern / HttpOnly cookie
```

---

## 12. Interview Questions

| Question | Answer |
|---|---|
| OAuth2 vs OIDC? | Authorization vs +identity. |
| Access vs Refresh vs ID token? | API access / renewal / user identity. |
| JWT validation steps? | Signature, iss, aud, exp, scope. |
| Audience validation kyu? | Token sirf intended service ke liye. |
| Token Relay kya? | User token downstream forward. |
| Refresh token kahan store? | BFF/server-side (browser me nahi). |

### Scenario-Based

| Scenario | Solution |
|---|---|
| User login | Gateway OAuth2 client |
| Token forward | TokenRelay |
| Service validation | Resource Server + JWT |
| Token renewal | Refresh token (BFF) |
| Service-specific token | Audience validation |

---

## 13. Key Takeaways

```text
1. OAuth2 = authorization, OIDC = +identity.

2. Access = API, Refresh = renewal, ID = identity.

3. JWT = header + claims + signature.

4. Validate: signature, iss, aud, exp, scope.

5. Audience validation critical.

6. TokenRelay = gateway se service.

7. BFF = refresh token server-side.

8. Har service validate (zero trust).

9. Short-lived access tokens.

10. JWKS = public keys.
```

---

## 14. Next Part Preview

**Part 7: Spring Authorization Server Setup**

Isme hum cover karenge:

```text
- Spring Authorization Server setup
- Client registration
- Token issuance
- JWK endpoint
- Custom claims
- Real-world config
```