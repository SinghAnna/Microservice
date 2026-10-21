# Part 8: Microservices Security — Keycloak Integration

> Keycloak integration ka modern approach — **Spring Security built-in OAuth2/OIDC support (no Keycloak adapter)** — Gateway = OAuth2 Client, Services = Resource Server, roles `realm_access.roles` claim se extract hote hain. [1227][1216][1629][1633]

---

## 1. Keycloak Mental Model

```text
Keycloak = Identity Provider (IdP)
   |
   +--> Realm (isolated tenant)
   |       +--> Clients (apps)
   |       +--> Users
   |       +--> Roles
   |       +--> Groups
   |
   +--> Token issuance (JWT)
   +--> JWKS endpoint (public keys)
```

### Keycloak Concepts

| Concept | Meaning |
|---|---|
| Realm | Isolated tenant (users, clients, roles) |
| Client | Application (Gateway, Service) |
| User | End user |
| Realm Role | Global role (USER, ADMIN) |
| Client Role | Client-specific role |
| JWKS | Public keys endpoint |

Keycloak realm-based IAM — users, clients, roles, tokens sab manage. [1216][1219][1633]

---

## 2. Keycloak Setup (Docker)

```bash
docker run -d \
  --name keycloak \
  -p 8080:8080 \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin \
  quay.io/keycloak/keycloak:latest \
  start-dev
```

### Realm Setup

```text
1. Admin console: http://localhost:8080
2. Create Realm: my-realm
3. Create Client: gateway-client (confidential)
4. Create Client: order-service (bearer-only)
5. Create Realm Roles: USER, ADMIN
6. Create User + assign roles
```

---

## 3. Gateway Configuration (OAuth2 Client)

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
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            scope:
              - openid
              - profile
        provider:
          keycloak:
            issuer-uri: http://localhost:8080/realms/my-realm
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

Gateway = OAuth2 Client — Keycloak login page redirect, token store. [1227][1630][1631]

---

## 4. Service Configuration (Resource Server)

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:8080/realms/my-realm
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
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(keycloakConverter())
                )
            );
        return http.build();
    }
}
```

Service = Resource Server — `issuer-uri` se JWKS auto-discover, JWT validate. [1216][1629][1633]

### issuer-uri vs jwk-set-uri

| Option | Behavior |
|---|---|
| `issuer-uri` | Discovery + JWKS + issuer validation |
| `jwk-set-uri` | Direct JWKS, no discovery |

`issuer-uri` recommended — discovery + issuer validation automatic. [1629][1634]

---

## 5. Keycloak Roles Extraction

### Problem

```text
Keycloak JWT:
{
  "realm_access": {
    "roles": ["USER", "ADMIN"]
  }
}

Spring Security expects:
Flat GrantedAuthority collection
```

### Fix — Custom Converter

```java
@Bean
public JwtAuthenticationConverter keycloakConverter() {
    JwtGrantedAuthoritiesConverter authoritiesConverter =
        new JwtGrantedAuthoritiesConverter();

    JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(jwt -> {
        Collection<GrantedAuthority> authorities =
            authoritiesConverter.convert(jwt);

        // realm_access.roles extract
        Map<String, Object> realmAccess = jwt.getClaim("realm_access");
        if (realmAccess != null && realmAccess.get("roles") != null) {
            List<String> roles = (List<String>) realmAccess.get("roles");
            authorities.addAll(roles.stream()
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                .collect(Collectors.toList()));
        }

        return authorities;
    });
    return converter;
}
```

Keycloak roles nested (`realm_access.roles`) — custom converter se flat authorities. [1216][1633][1635]

---

## 6. Complete JWT Example

```json
{
  "exp": 1735689600,
  "iat": 1735686000,
  "iss": "http://localhost:8080/realms/my-realm",
  "aud": "account",
  "sub": "user-123",
  "preferred_username": "john",
  "realm_access": {
    "roles": ["USER", "ADMIN"]
  },
  "scope": "openid profile"
}
```

### Validation

```text
1. Signature (JWKS)
2. iss = http://localhost:8080/realms/my-realm
3. exp check
4. realm_access.roles -> ROLE_USER, ROLE_ADMIN
5. hasRole("ADMIN") match
```

---

## 7. Gateway + Token Relay

```yaml
spring:
  cloud:
    gateway:
      default-filters:
        - TokenRelay=
      routes:
        - id: order-service
          uri: http://localhost:8081
          predicates:
            - Path=/api/orders/**
```

Gateway Keycloak login + TokenRelay — service ko JWT forward. [1227][1630]

---

## 8. Complete Flow

```text
1. User -> Gateway
2. Gateway -> Keycloak login page
3. User login (Keycloak)
4. Keycloak -> Gateway (code)
5. Gateway -> Keycloak (code -> tokens)
6. Gateway session me tokens
7. User -> Gateway -> Service (TokenRelay)
8. Service -> Keycloak JWKS se key
9. Service -> JWT validate + roles extract
10. hasRole("ADMIN") check
```

---

## 9. Production Considerations

| Consideration | Solution |
|---|---|
| Keycloak HA | Cluster mode (Infinispan) |
| Token lifetime | Short access (5-15 min) |
| Key rotation | Keycloak auto + JWKS cache |
| HTTPS | TLS everywhere |
| Realm per env | Dev/Staging/Prod realms |
| Client secrets | Vault |

### JWKS Cache Tuning

```java
@Bean
public JwtDecoder jwtDecoder() {
    return JwtDecoders.fromIssuerLocation(
        "[https://keycloak.example.com/realms/my-realm](https://keycloak.example.com/realms/my-realm)"
    );
}
```

JWKS cache — key rotation ke baad refresh, TTL tune. [1629][1634]

---

## 10. Common Gotchas

### Gotcha 1 — Roles Not Extracted

```text
Problem:
hasRole("ADMIN") fail

Fix:
Custom converter (realm_access.roles)
```

### Gotcha 2 — Issuer Mismatch

```text
Problem:
401 invalid issuer

Fix:
issuer-uri exact realm URL
```

### Gotcha 3 — Redirect URI Mismatch

```text
Problem:
Keycloak redirect reject

Fix:
Client me valid redirect URI
```

### Gotcha 4 — Keycloak Adapter (Old)

```text
Problem:
keycloak-spring-boot-adapter deprecated

Fix:
Spring Security built-in OAuth2
```

Keycloak adapter deprecated — Spring Security built-in OAuth2 use karo. [1227][1631][1632]

---

## 11. Interview Questions

| Question | Answer |
|---|---|
| Keycloak kya hai? | Open-source IAM — realm, users, clients, roles. |
| Realm kya hai? | Isolated tenant. |
| Gateway vs Service role? | OAuth2 Client vs Resource Server. |
| Keycloak roles kaise extract? | `realm_access.roles` custom converter. |
| issuer-uri kya karta hai? | Discovery + JWKS + issuer validation. |
| Keycloak adapter status? | Deprecated — Spring Security built-in. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Centralized IAM | Keycloak |
| Role-based access | realm_access.roles converter |
| Service token validation | Resource Server + issuer-uri |
| HA Keycloak | Cluster mode |
| Key rotation | JWKS cache |

---

## 12. Key Takeaways

```text
1. Keycloak = IAM (realm, clients, users, roles).

2. Gateway = OAuth2 Client.

3. Services = Resource Server.

4. issuer-uri = discovery + validation.

5. realm_access.roles = nested roles.

6. Custom converter = flat authorities.

7. TokenRelay = JWT forward.

8. Adapter deprecated — built-in OAuth2.

9. JWKS cache = key rotation.

10. HA = Keycloak cluster.
```

---

## 13. Next Part Preview

**Part 9: JWT Tokens — Structure, Claims, Validation**

Isme hum cover karenge:

```text
- JWT structure deep dive
- Claims (registered + custom)
- Signing algorithms (HS256 vs RS256)
- Validation strategies
- Best practices
```