# Part 13: Microservices Security — Token Exchange Pattern Deep Dive (RFC 8693)

> Token Exchange ka core — **`subject_token` (user) + `actor_token` (service) + `act` claim (delegation chain)** — Spring Security 6.3+ me first-class support, Spring Authorization Server 1.3+ me server-side. [1685][1690][1693]

---

## 1. Token Exchange Mental Model

```text
User -> Gateway (JWT: aud=api)
   |
   v
Gateway -> Auth Server (Token Exchange)
   |
   | subject_token = user JWT
   | audience = order-service
   v
Naya Token (aud=order-service, scope=orders:read)
   |
   v
Order Service
```

### Kaise Kaam Karta Hai

```text
1. User token Gateway pe aata hai
2. Gateway Auth Server ko exchange request bhejta hai
3. Auth Server user token validate
4. Naya token issue (service-specific audience + scopes)
5. Gateway naya token downstream forward
```

Token Exchange — ek token doosre me exchange, service-specific audience + scopes. [1685][1690][1683]

---

## 2. RFC 8693 Request

```text
POST /oauth2/token
Content-Type: application/x-www-form-urlencoded
Authorization: Basic base64(gateway-client:secret)

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
subject_token=eyJhbGciOi...  (user JWT)
subject_token_type=urn:ietf:params:oauth:token-type:access_token
requested_token_type=urn:ietf:params:oauth:token-type:access_token
audience=order-service
scope=orders:read orders:write
```

### Request Parameters

| Parameter | Purpose |
|---|---|
| `grant_type` | `token-exchange` |
| `subject_token` | User token (on whose behalf) |
| `subject_token_type` | Token type hint |
| `actor_token` | Service token (optional, delegation) |
| `audience` | Target service |
| `scope` | Requested scopes (downscope) |

`subject_token` = user identity, `actor_token` = acting service, `audience` = target. [1685][1686][1693]

---

## 3. Response

```json
{
  "access_token": "eyJhbGciOi...",
  "issued_token_type": "urn:ietf:params:oauth:token-type:access_token",
  "token_type": "Bearer",
  "expires_in": 900,
  "scope": "orders:read orders:write"
}
```

### Naya JWT

```json
{
  "iss": "[https://auth.example.com](https://auth.example.com)",
  "sub": "user-123",
  "aud": "order-service",
  "scope": "orders:read orders:write",
  "act": {
    "sub": "gateway-client"
  }
}
```

Naya token — service-specific `aud` + downscoped `scope` + `act` (delegation). [1685][1690][1693]

---

## 4. Impersonation vs Delegation

```text
Impersonation:
Service user ki identity me act (indistinguishable)
sub = user, act = nahi

Delegation:
Service user ki taraf se act (own identity)
sub = user, act = { sub: service }
```

| Mode | `sub` | `act` | Meaning |
|---|---|---|---|
| Impersonation | user | Absent | Service = user |
| Delegation | user | `{sub: service}` | Service user ki taraf se |

Impersonation — service user jaisa dikhta hai; Delegation — `act` claim se service ki identity separate. [1685][1686][1689][1693]

### Delegation Chain (Nested)

```json
{
  "sub": "user-123",
  "act": {
    "sub": "order-service",
    "act": {
      "sub": "notification-service"
    }
  }
}
```

Nested `act` — delegation chain (user -> order -> notification). [1685][1689]

---

## 5. Spring Authorization Server (Server-Side)

```java
@Bean
public RegisteredClientRepository registeredClientRepository() {
    RegisteredClient gatewayClient = RegisteredClient
        .withId(UUID.randomUUID().toString())
        .clientId("gateway-client")
        .clientSecret("{bcrypt}$2a$10$...")
        .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
        .authorizationGrantType(
            new AuthorizationGrantType("urn:ietf:params:oauth:grant-type:token-exchange")
        )
        .scope("orders:read")
        .scope("orders:write")
        .build();

    return new InMemoryRegisteredClientRepository(gatewayClient);
}
```

### Token Exchange Grant Enable

```java
@Bean
@Order(1)
public SecurityFilterChain authorizationServerSecurityFilterChain(
    HttpSecurity http
) throws Exception {
    OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http);
    return http.build();
}
```

Spring Authorization Server 1.3+ — `token-exchange` grant type native support. [1691][1692][1693]

### Custom Exchange Logic

```java
@Bean
public OAuth2TokenExchangeAuthenticationProvider tokenExchangeProvider(
    RegisteredClientRepository clients,
    OAuth2AuthorizationService authorizationService
) {
    return new OAuth2TokenExchangeAuthenticationProvider(
        authorizationService, clients
    );
}
```

`OAuth2TokenExchangeAuthenticationProvider` — exchange logic (impersonation/delegation). [1693]

---

## 6. Spring Security Client (Gateway-Side)

```java
@Bean
public OAuth2AuthorizedClientManager authorizedClientManager(
    ClientRegistrationRepository clients,
    OAuth2AuthorizedClientService authorizedClients
) {
    OAuth2AuthorizedClientProvider provider =
        OAuth2AuthorizedClientProviderBuilder.builder()
            .authorizationCode()
            .refreshToken()
            .clientCredentials()
            .build();

    DefaultOAuth2AuthorizedClientManager manager =
        new DefaultOAuth2AuthorizedClientManager(clients, authorizedClients);
    manager.setAuthorizedClientProvider(provider);
    return manager;
}
```

### Exchange Request (Gateway)

```java
public Mono<String> exchangeToken(String userToken) {
    MultiValueMap<String, String> form = new LinkedMultiValueMap<>();
    form.add("grant_type",
        "urn:ietf:params:oauth:grant-type:token-exchange");
    form.add("subject_token", userToken);
    form.add("subject_token_type",
        "urn:ietf:params:oauth:token-type:access_token");
    form.add("audience", "order-service");
    form.add("scope", "orders:read");

    return webClient.post()
        .uri("[https://auth.example.com/oauth2/token](https://auth.example.com/oauth2/token)")
        .headers(h -> h.setBasicAuth("gateway-client", secret))
        .bodyValue(form)
        .retrieve()
        .bodyToMono(TokenResponse.class)
        .map(TokenResponse::getAccessToken);
}
```

Gateway — user token ko service-specific token me exchange. [1690][1694]

---

## 7. Complete Flow

```text
1. User -> Gateway (JWT: aud=api)
2. Gateway validate
3. Gateway -> Auth Server (token exchange)
4. Auth Server validate subject_token
5. Auth Server -> Naya token (aud=order-service)
6. Gateway -> Order Service (naya token)
7. Order Service validate (aud=order-service)
8. Response back
```

---

## 8. Real-World Scenarios

| Scenario | Pattern |
|---|---|
| Service-specific scopes | Token Exchange |
| Blast radius kam | Token Exchange |
| Audit trail (kaun kis ki taraf se) | Delegation (`act`) |
| Service impersonation | Impersonation |
| Multi-hop delegation | Nested `act` |
| Downscoping | Exchange with fewer scopes |

### Example: E-Commerce

```text
User -> Gateway (aud=api, scope=full)
Gateway -> Order Service (aud=order-service, scope=orders:read)
Order -> Notification (aud=notification, scope=notify)
Har hop = naya token, downscoped
```

---

## 9. Common Gotchas

### Gotcha 1 — Exchange Not Supported

```text
Problem:
unsupported_grant_type

Fix:
Spring Authorization Server 1.3+ / custom grant
```

### Gotcha 2 — Audience Mismatch

```text
Problem:
Exchange fail

Fix:
audience parameter + client registration
```

### Gotcha 3 — Scope Escalation

```text
Problem:
Zyada scopes maang raha

Fix:
Downscope only (original scopes subset)
```

### Gotcha 4 — Latency

```text
Problem:
Har request exchange call

Fix:
Token cache (short TTL)
```

---

## 10. Interview Questions

| Question | Answer |
|---|---|
| Token Exchange kya hai? | RFC 8693 — token doosre me exchange. |
| subject_token vs actor_token? | User identity vs acting service. |
| Impersonation vs Delegation? | User jaisa vs user ki taraf se (`act`). |
| `act` claim? | Delegation chain record. |
| Downscoping? | Kam scopes ka naya token. |
| Blast radius? | Service-specific token = kam. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Service-specific token | Token Exchange |
| Audit trail | Delegation (`act`) |
| Multi-hop | Nested `act` |
| Scope kam karna | Downscope |
| Blast radius kam | Per-service token |

---

## 11. Key Takeaways

```text
1. Token Exchange = RFC 8693.

2. subject_token = user, actor_token = service.

3. act claim = delegation chain.

4. Impersonation = no act.

5. Delegation = act claim.

6. Downscope = kam scopes.

7. Service-specific audience.

8. Blast radius kam.

9. Spring Security 6.3+ support.

10. Spring Authorization Server 1.3+.
```

---

## 12. Next Part Preview

**Part 14: Rate Limiting & Throttling at Gateway**

Isme hum cover karenge:

```text
- Rate limiting deep dive
- Token bucket / sliding window
- Redis rate limiter
- Per-user / per-IP / per-API
- Distributed rate limiting
```