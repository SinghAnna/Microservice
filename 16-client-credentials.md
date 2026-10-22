# Part 16: Microservices Security — Client Credentials Flow

> Client Credentials ka core — **service apni identity (client_id + secret) se token leta hai, user involvement nahi** — `OAuth2AuthorizedClientManager` token acquire + cache + auto-refresh (expiry se pehle) karta hai. [1722][1723][1728]

---

## 1. Client Credentials Mental Model

```text
Service A -> Auth Server
   |
   | client_id + client_secret
   v
Access Token (service-specific)
   |
   v
Service A -> Service B (Bearer token)
```

### Kab Use Karein

| Use Case | Example |
|---|---|
| Service-to-service | Order -> Product |
| Background jobs | Scheduler -> API |
| Machine-to-machine | Batch processing |
| No user context | Internal APIs |

Client Credentials — M2M auth, user involvement nahi, service apni identity se. [1722][1728]

---

## 2. Flow Deep Dive

```text
1. Service A -> Auth Server /oauth2/token
   client_id + client_secret + grant_type=client_credentials

2. Auth Server validate credentials

3. Auth Server -> Access Token
   {
     "iss": "[https://auth.example.com](https://auth.example.com)",
     "sub": "order-service",     (service = subject)
     "aud": "product-service",
     "scope": "product:read",
     "exp": ...
   }

4. Service A -> Service B (Bearer token)

5. Service B validate (signature, iss, aud, scope)
```

### Token Claims

```json
{
  "iss": "[https://auth.example.com](https://auth.example.com)",
  "sub": "order-service",
  "aud": "product-service",
  "scope": "product:read",
  "exp": 1735689600
}
```

`sub` = service identity (user nahi), `aud` = target service, `scope` = permissions. [1722][1728]

---

## 3. Auth Server Config

```java
@Bean
public RegisteredClientRepository registeredClientRepository() {
    RegisteredClient orderService = RegisteredClient
        .withId(UUID.randomUUID().toString())
        .clientId("order-service")
        .clientSecret("{bcrypt}$2a$10$...")
        .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
        .authorizationGrantType(AuthorizationGrantType.CLIENT_CREDENTIALS)
        .scope("product:read")
        .scope("product:write")
        .tokenSettings(TokenSettings.builder()
            .accessTokenTimeToLive(Duration.ofMinutes(15))
            .build())
        .build();

    return new InMemoryRegisteredClientRepository(orderService);
}
```

Client registration — `CLIENT_CREDENTIALS` grant + scopes + short TTL. [1722]

---

## 4. Service A Config (Client)

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          product-service:
            client-id: order-service
            client-secret: ${ORDER_SERVICE_SECRET}
            authorization-grant-type: client_credentials
            scope:
              - product:read
          provider:
            auth-server:
              token-uri: [https://auth.example.com/oauth2/token](https://auth.example.com/oauth2/token)
```

### AuthorizedClientManager

```java
@Bean
public OAuth2AuthorizedClientManager authorizedClientManager(
    ClientRegistrationRepository clients,
    OAuth2AuthorizedClientService authorizedClients
) {
    OAuth2AuthorizedClientProvider provider =
        OAuth2AuthorizedClientProviderBuilder.builder()
            .clientCredentials()
            .build();

    DefaultOAuth2AuthorizedClientManager manager =
        new DefaultOAuth2AuthorizedClientManager(clients, authorizedClients);
    manager.setAuthorizedClientProvider(provider);
    return manager;
}
```

`OAuth2AuthorizedClientManager` — token acquire + cache + refresh (expiry se pehle). [1722][1723][1725]

---

## 5. Service A Code (Calling Service B)

```java
@Service
@RequiredArgsConstructor
public class ProductServiceClient {

    private final OAuth2AuthorizedClientManager manager;
    private final WebClient.Builder webClientBuilder;

    public Product getProduct(Long id) {
        OAuth2AuthorizeRequest authorizeRequest = OAuth2AuthorizeRequest
            .withClientRegistrationId("product-service")
            .principal("order-service")
            .build();

        OAuth2AuthorizedClient client = manager.authorize(authorizeRequest);
        String token = client.getAccessToken().getTokenValue();

        return webClientBuilder.build()
            .get()
            .uri("http://product-service/api/products/" + id)
            .header("Authorization", "Bearer " + token)
            .retrieve()
            .bodyToMono(Product.class)
            .block();
    }
}
```

`manager.authorize()` — token cached hai to reuse, expire hua to refresh. [1722][1723][1726]

---

## 6. Token Caching

```text
Token acquire:
1. Cache check (valid token?)
2. Valid -> Reuse
3. Expired / Missing -> Auth Server se naya
4. Cache update
```

### Caching Behavior

| Scenario | Action |
|---|---|
| Valid token cached | Reuse (no call) |
| Token expiring soon | Proactive refresh |
| Token expired | New acquire |
| 401 from service | Retry with new token |

Proactive refresh — expiry se 1-2 min pehle, failed requests avoid. [1723][1724][1728]

### Buffer Time

```text
Token expiry: 15 min
Refresh buffer: 1-2 min pehle
= 13-14 min me refresh
```

---

## 7. Service B Config (Resource Server)

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: [https://auth.example.com](https://auth.example.com)
          audience: product-service
```

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .csrf(AbstractHttpConfigurer::disable)
        .sessionManagement(session -> session
            .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
        )
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/products/**")
            .hasAuthority("SCOPE_product:read")
            .anyRequest().authenticated()
        )
        .oauth2ResourceServer(oauth2 -> oauth2
            .jwt(Customizer.withDefaults())
        );
    return http.build();
}
```

Service B — JWT validate (signature, iss, aud, scope). [1722][1728]

---

## 8. Scope-Based Authorization

```text
order-service token:
scope = product:read

Product Service:
GET /api/products -> SCOPE_product:read required ✓
POST /api/products -> SCOPE_product:write required ✗ (403)
```

### Least Privilege

```text
Order service ko sirf read chahiye:
scope = product:read (write nahi)

Compromise = read-only damage
```

Least privilege — sirf zaroori scopes, compromise impact kam. [1722][1728]

---

## 9. Complete Flow

```text
1. Order Service -> Product Service call
2. AuthorizedClientManager -> Cache check
3. Valid token -> Reuse
4. Expired -> Auth Server se naya
5. Bearer token -> Product Service
6. Product Service -> JWT validate
7. Scope check (product:read)
8. Response
```

---

## 10. Real-World Patterns

| Pattern | Use Case |
|---|---|
| Client Credentials | Service-to-service |
| Token Exchange | User context + service |
| mTLS | Service identity (cert) |
| Client Credentials + mTLS | Highest security |

### Client Credentials vs Token Exchange

| Aspect | Client Credentials | Token Exchange |
|---|---|---|
| User context | Nahi | Haan |
| Identity | Service | User (on behalf) |
| Use Case | M2M | User-facing flow |

Client Credentials = no user, Token Exchange = user context preserve. [1722][1728]

---

## 11. Common Gotchas

### Gotcha 1 — Token Fetch Har Request

```text
Problem:
Performance hit

Fix:
AuthorizedClientManager (cache built-in)
```

### Gotcha 2 — Concurrent Refresh

```text
Problem:
Multiple threads refresh simultaneously

Fix:
Manager synchronization / Caffeine cache
```

### Gotcha 3 — 401 After Revocation

```text
Problem:
Cached token revoked

Fix:
401 retry with new token
```

### Gotcha 4 — Scope Escalation

```text
Problem:
Zyada scopes maangna

Fix:
Least privilege (sirf zaroori)
```

Concurrent refresh — manager synchronize karta hai, ek hi refresh. [1726]

---

## 12. Interview Questions

| Question | Answer |
|---|---|
| Client Credentials kya hai? | M2M auth — service apni identity se token. |
| Kab use? | Service-to-service, background jobs. |
| Token caching? | AuthorizedClientManager built-in. |
| Proactive refresh? | Expiry se 1-2 min pehle. |
| sub claim? | Service identity (user nahi). |
| Least privilege? | Sirf zaroori scopes. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Service-to-service | Client Credentials |
| User context | Token Exchange |
| Performance | Token caching |
| Security | Least privilege scopes |
| Revoked token | 401 retry |

---

## 13. Key Takeaways

```text
1. Client Credentials = M2M auth.

2. sub = service identity.

3. AuthorizedClientManager = cache + refresh.

4. Proactive refresh = buffer time.

5. Least privilege scopes.

6. Audience validation.

7. Short-lived tokens (15 min).

8. Concurrent refresh = synchronized.

9. 401 retry = resilience.

10. mTLS = extra layer.
```

---

## 14. Next Part Preview

**Part 17: Service-to-Service Auth Patterns**

Isme hum cover karenge:

```text
- Service-to-service patterns deep dive
- mTLS vs Client Credentials vs API Keys
- Service identity
- Network segmentation
- Real-world comparison
```