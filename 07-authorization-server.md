# Part 7: Microservices Security — Spring Authorization Server Setup

> Spring Authorization Server ka core — **`RegisteredClientRepository` (clients) + `JWKSource` (signing keys) + `OAuth2TokenCustomizer` (custom claims)** — teeno beans se complete OAuth2/OIDC provider banta hai. [1616][1620][1621]

---

## 1. Authorization Server Mental Model

```text
Auth Server Responsibilities:
1. Client registration (kaun access maang sakta hai)
2. User authentication (kaun hai)
3. Token issuance (access + refresh + ID)
4. Token signing (JWKS)
5. Token introspection / revocation
6. OIDC discovery (/.well-known/openid-configuration)
```

### Endpoints

| Endpoint | Purpose |
|---|---|
| `/oauth2/authorize` | Authorization endpoint |
| `/oauth2/token` | Token endpoint |
| `/oauth2/introspect` | Token introspection |
| `/oauth2/revoke` | Token revocation |
| `/oauth2/jwks` | Public keys (JWKS) |
| `/.well-known/openid-configuration` | OIDC discovery |

Spring Authorization Server — OAuth2 + OIDC provider, endpoints auto-configured. [1617][1620][1621]

---

## 2. Setup

### Dependency

```xml
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-oauth2-authorization-server</artifactId>
</dependency>
```

### Main Class

```java
@SpringBootApplication
public class AuthServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(AuthServerApplication.class, args);
    }
}
```

---

## 3. Client Registration (RegisteredClientRepository)

```java
@Bean
public RegisteredClientRepository registeredClientRepository() {

    // Gateway (Authorization Code + Refresh)
    RegisteredClient gatewayClient = RegisteredClient
        .withId(UUID.randomUUID().toString())
        .clientId("gateway-client")
        .clientSecret("{bcrypt}$2a$10$...")
        .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
        .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
        .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
        .redirectUri("http://localhost:8080/login/oauth2/code/gateway")
        .scope(OidcScopes.OPENID)
        .scope(OidcScopes.PROFILE)
        .clientSettings(ClientSettings.builder()
            .requireAuthorizationConsent(true)
            .build())
        .tokenSettings(TokenSettings.builder()
            .accessTokenTimeToLive(Duration.ofMinutes(15))
            .refreshTokenTimeToLive(Duration.ofHours(8))
            .build())
        .build();

    // Service-to-Service (Client Credentials)
    RegisteredClient serviceClient = RegisteredClient
        .withId(UUID.randomUUID().toString())
        .clientId("order-service")
        .clientSecret("{bcrypt}$2a$10$...")
        .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
        .authorizationGrantType(AuthorizationGrantType.CLIENT_CREDENTIALS)
        .scope("product:read")
        .build();

    return new InMemoryRegisteredClientRepository(
        gatewayClient, serviceClient
    );
}
```

`RegisteredClientRepository` — clients register, grant types, scopes, redirect URIs. [1620][1621]

### Client Settings

| Setting | Purpose |
|---|---|
| `clientId` | Unique ID |
| `clientSecret` | bcrypt hashed |
| `authorizationGrantType` | Code / Client Credentials / Refresh |
| `redirectUri` | Callback URL |
| `scope` | Requested permissions |
| `accessTokenTimeToLive` | Access token lifetime |
| `refreshTokenTimeToLive` | Refresh token lifetime |

---

## 4. JWK Source (Signing Keys)

```java
@Bean
public JWKSource<SecurityContext> jwkSource() {
    RSAKey rsaKey = generateRsa();
    JWKSet jwkSet = new JWKSet(rsaKey);
    return new ImmutableJWKSet<>(jwkSet);
}

private RSAKey generateRsa() {
    KeyPair keyPair = generateKeyPair();
    RSAPublicKey publicKey = (RSAPublicKey) keyPair.getPublic();
    RSAPrivateKey privateKey = (RSAPrivateKey) keyPair.getPrivate();

    return new RSAKey.Builder(publicKey)
        .privateKey(privateKey)
        .keyID(UUID.randomUUID().toString())
        .build();
}

private KeyPair generateKeyPair() {
    try {
        KeyPairGenerator generator = KeyPairGenerator.getInstance("RSA");
        generator.initialize(2048);
        return generator.generateKeyPair();
    } catch (Exception e) {
        throw new IllegalStateException(e);
    }
}
```

`JWKSource` — token signing keys, `/oauth2/jwks` endpoint se public keys expose. [1620][1628]

### Key Rotation

```text
Multiple keys in JWKSet
kid (key ID) header me
Old key retire, new key add
Clients JWKS cache refresh
```

---

## 5. Custom Claims (OAuth2TokenCustomizer)

```java
@Bean
public OAuth2TokenCustomizer<JwtEncodingContext> jwtTokenCustomizer(
    UserService userService
) {
    return context -> {
        if (OAuth2TokenType.ACCESS_TOKEN.equals(context.getTokenType())) {

            String username = context.getPrincipal().getName();
            User user = userService.findByUsername(username);

            context.getClaims().claims(claims -> {
                claims.put("roles", user.getRoles().stream()
                    .map(Role::getName).collect(Collectors.toList()));
                claims.put("tenant_id", user.getTenantId());
                claims.put("department", user.getDepartment());
            });
        }
    };
}
```

`OAuth2TokenCustomizer<JwtEncodingContext>` — access token me custom claims (roles, tenant). [1616][1622][1623][1624]

### Example JWT

```json
{
  "iss": "[https://auth.example.com](https://auth.example.com)",
  "sub": "user-123",
  "aud": "order-service",
  "roles": ["ROLE_USER"],
  "tenant_id": "tenant-1",
  "scope": "orders:read"
}
```

---

## 6. Security Config

```java
@Bean
@Order(1)
public SecurityFilterChain authorizationServerSecurityFilterChain(
    HttpSecurity http
) throws Exception {
    OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http);

    http.getConfigurer(OAuth2AuthorizationServerConfigurer.class)
        .oidc(Customizer.withDefaults());  // OIDC enable

    http
        .exceptionHandling(e -> e
            .defaultAuthenticationEntryPointFor(
                new LoginUrlAuthenticationEntryPoint("/login"),
                new MediaTypeRequestMatcher(MediaType.TEXT_HTML)
            )
        )
        .oauth2ResourceServer(rs -> rs.jwt(Customizer.withDefaults()));

    return http.build();
}
```

`applyDefaultSecurity` + `oidc()` — OAuth2 + OIDC dono enable. [1617][1621]

---

## 7. User Authentication (Form Login)

```java
@Bean
@Order(2)
public SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http)
    throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/login", "/error").permitAll()
            .anyRequest().authenticated()
        )
        .formLogin(Customizer.withDefaults());
    return http.build();
}

@Bean
public UserDetailsService userDetailsService() {
    UserDetails user = User.builder()
        .username("user")
        .password("{bcrypt}$2a$10$...")
        .roles("USER")
        .build();
    return new InMemoryUserDetailsManager(user);
}
```

User login (form) — authorization server user authenticate karta hai. [1619][1621]

---

## 8. Complete Flow

```text
1. User -> Gateway /login
2. Gateway -> Auth Server /oauth2/authorize
3. User -> Auth Server login + consent
4. Auth Server -> Gateway (code)
5. Gateway -> Auth Server /oauth2/token (code + secret)
6. Auth Server -> Tokens (access + refresh + ID)
7. Gateway -> Service (Token Relay)
8. Service -> /oauth2/jwks se public key
9. Service -> JWT validate
```

---

## 9. OIDC Discovery

```bash
curl http://localhost:9000/.well-known/openid-configuration
```

```json
{
  "issuer": "http://localhost:9000",
  "authorization_endpoint": "http://localhost:9000/oauth2/authorize",
  "token_endpoint": "http://localhost:9000/oauth2/token",
  "jwks_uri": "http://localhost:9000/oauth2/jwks",
  "userinfo_endpoint": "http://localhost:9000/userinfo"
}
```

OIDC discovery — clients auto-configure (issuer-uri se). [1617][1621]

---

## 10. Production Considerations

| Consideration | Solution |
|---|---|
| Client storage | DB (JdbcRegisteredClientRepository) |
| Key storage | KMS / Vault |
| Key rotation | Multiple keys + kid |
| Token storage | DB (JdbcOAuth2AuthorizationService) |
| Consent | Custom consent page |
| HA | Multiple instances + shared DB |

### JDBC Client Repository

```java
@Bean
public RegisteredClientRepository registeredClientRepository(
    JdbcTemplate jdbcTemplate
) {
    return new JdbcRegisteredClientRepository(jdbcTemplate);
}
```

Production me in-memory nahi — JDBC (DB-backed) repositories. [1620][1621]

---

## 11. Common Gotchas

### Gotcha 1 — Redirect URI Mismatch

```text
Problem:
Invalid redirect URI

Fix:
RegisteredClient redirectUri exact match
```

### Gotcha 2 — JWKS Not Accessible

```text
Problem:
Service token validate fail

Fix:
/oauth2/jwks permitAll (public)
```

### Gotcha 3 — Custom Claims Missing

```text
Problem:
Token me roles nahi

Fix:
OAuth2TokenCustomizer bean
```

### Gotcha 4 — In-Memory in Production

```text
Problem:
Restart = clients gone

Fix:
JDBC repositories
```

---

## 12. Interview Questions

| Question | Answer |
|---|---|
| Authorization Server kya karta hai? | Token issue, client manage, user auth. |
| RegisteredClientRepository? | Clients register + query. |
| JWKSource kya hai? | Token signing keys. |
| OAuth2TokenCustomizer? | Custom claims add. |
| OIDC discovery kya? | `/.well-known/openid-configuration`. |
| Key rotation? | Multiple keys + kid. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Client register | RegisteredClientRepository |
| Token sign | JWKSource (RSA) |
| Roles in token | OAuth2TokenCustomizer |
| Production storage | JDBC repositories |
| Key rotation | Multiple keys + kid |

---

## 13. Key Takeaways

```text
1. Auth Server = OAuth2 + OIDC provider.

2. RegisteredClientRepository = clients.

3. JWKSource = signing keys.

4. OAuth2TokenCustomizer = custom claims.

5. applyDefaultSecurity + oidc().

6. /oauth2/jwks = public keys.

7. OIDC discovery = auto-config.

8. Production = JDBC storage.

9. Key rotation = multiple keys.

10. bcrypt = client secrets.
```

---

## 14. Next Part Preview

**Part 8: Keycloak Integration**

Isme hum cover karenge:

```text
- Keycloak setup
- Realm, clients, roles
- Spring Boot integration
- Token validation
- Real-world config
```