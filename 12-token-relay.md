# Part 12: Microservices Security — Token Relay Pattern Deep Dive

> Token Relay ka core — **user ka existing access token unchanged downstream forward** — jabki Token Exchange (RFC 8693) me **naya, service-specific token** issue hota hai (different audience/scopes). [1676][1677][1675][1683]

---

## 1. Token Relay Mental Model

```text
User -> Gateway (JWT: aud=api, scope=read)
   |
   v
Gateway (TokenRelay)
   |
   v
Service A (Same JWT: aud=api, scope=read)
   |
   v
Service B (Same JWT: aud=api, scope=read)
```

### Kaise Kaam Karta Hai

```text
1. User login -> Access token (Gateway session me)
2. Request -> Gateway
3. TokenRelay filter -> Access token extract
4. Authorization: Bearer <token> header set
5. Downstream service -> Same token
```

Token Relay — user ka existing token unchanged forward, koi naya token nahi. [1676][1677][1678]

---

## 2. TokenRelay Filter Internals

```java
// Spring Cloud Gateway internal (simplified)
public class TokenRelayGatewayFilterFactory
    extends AbstractGatewayFilterFactory<Object> {

    @Override
    public GatewayFilter apply(Object config) {
        return (exchange, chain) -> {
            return exchange.getPrincipal()
                .flatMap(principal -> {
                    OAuth2AuthenticationToken auth =
                        (OAuth2AuthenticationToken) principal;

                    OAuth2AuthorizedClient client = authorizedClientService
                        .loadAuthorizedClient(
                            auth.getAuthorizedClientRegistrationId(),
                            auth.getName()
                        );

                    String token = client.getAccessToken().getTokenValue();

                    ServerHttpRequest request = exchange.getRequest().mutate()
                        .header("Authorization", "Bearer " + token)
                        .build();

                    return chain.filter(
                        exchange.mutate().request(request).build()
                    );
                });
        };
    }
}
```

TokenRelay — `OAuth2AuthorizedClientService` se token extract, `Authorization: Bearer` header set. [1676][1677]

### Prerequisites

```text
1. spring.security.oauth2.client.* config (OAuth2AuthorizedClientManager)
2. oauth2Login() enabled
3. TokenRelay= filter (default ya route-specific)
```

`TokenRelay` sirf tab kaam karta hai jab `OAuth2AuthorizedClientManager` bean ho. [1676][1677]

---

## 3. TokenRelay Configuration

### Default (All Routes)

```yaml
spring:
  cloud:
    gateway:
      default-filters:
        - TokenRelay=
```

### Route-Specific

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: http://localhost:8081
          predicates:
            - Path=/api/orders/**
          filters:
            - TokenRelay=
```

### Java DSL

```java
@Bean
public RouteLocator routes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("order-service", r -> r
            .path("/api/orders/**")
            .filters(f -> f.tokenRelay())
            .uri("http://localhost:8081")
        )
        .build();
}
```

`TokenRelay=` default (saare routes) ya route-specific — dono options. [1676][1677]

---

## 4. Token Relay vs Token Exchange

| Aspect | Token Relay | Token Exchange (RFC 8693) |
|---|---|---|
| Token | Same (unchanged) | Naya (service-specific) |
| Audience | Same | Service-specific |
| Scopes | Same | Downscoped possible |
| Auth Server call | Nahi | Haan (har exchange) |
| Latency | Low | Higher (extra call) |
| Blast radius | High (same token everywhere) | Low (per-service token) |

### Token Exchange Flow

```text
User -> Gateway (JWT: aud=api)
   |
   v
Gateway -> Auth Server (token exchange)
   |
   v
Naya Token (aud=order-service, scope=orders:read)
   |
   v
Order Service
```

Token Relay same token, Token Exchange naya service-specific token — blast radius kam. [1675][1679][1680][1683]

---

## 5. Token Exchange (RFC 8693)

```text
POST /oauth2/token
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
subject_token=<user-jwt>
subject_token_type=urn:ietf:params:oauth:token-type:access_token
audience=order-service
requested_token_type=urn:ietf:params:oauth:token-type:access_token
```

### Response

```json
{
  "access_token": "eyJhbGciOi...",
  "issued_token_type": "urn:ietf:params:oauth:token-type:access_token",
  "token_type": "Bearer",
  "expires_in": 900
}
```

RFC 8693 — `token-exchange` grant type, `subject_token` + `audience`. [1675][1679][1681]

### Impersonation vs Delegation

| Mode | Meaning |
|---|---|
| Impersonation | Service user ki identity me act karta hai |
| Delegation | Service user ki taraf se act (actor claim) |

```json
{
  "sub": "user-123",
  "act": {
    "sub": "order-service"
  }
}
```

`act` claim = delegation — kaun user ki taraf se act kar raha. [1675][1680][1682]

---

## 6. Custom Token Propagation

```java
@Component
public class CustomTokenRelayFilter implements GatewayFilter {

    @Override
    public Mono<Void> filter(
        ServerWebExchange exchange,
        GatewayFilterChain chain
    ) {
        String token = exchange.getRequest().getHeaders()
            .getFirst("Authorization");

        if (token != null && token.startsWith("Bearer ")) {
            // Custom header add (internal service)
            ServerHttpRequest request = exchange.getRequest().mutate()
                .header("X-Internal-Token", transformToken(token))
                .build();
            return chain.filter(exchange.mutate().request(request).build());
        }

        return chain.filter(exchange);
    }
}
```

Custom propagation — internal token format / header transformation. [1669][1671][1672]

### Cookie Removal

```yaml
filters:
  - TokenRelay=
  - RemoveRequestHeader=Cookie
```

Cookie remove — downstream service ko sirf JWT chahiye, session cookie nahi. [1684]

---

## 7. Complete Flow

```text
Token Relay:
1. User -> Gateway (JWT)
2. Gateway validate
3. TokenRelay -> Same JWT
4. Service -> Validate

Token Exchange:
1. User -> Gateway (JWT)
2. Gateway validate
3. Gateway -> Auth Server (exchange)
4. Naya service-specific token
5. Service -> Validate (service-specific aud)
```

---

## 8. Real-World Scenarios

| Scenario | Pattern |
|---|---|
| Simple relay | Token Relay |
| Service-specific scopes | Token Exchange |
| Blast radius kam | Token Exchange |
| Low latency | Token Relay |
| Zero trust | Token Exchange |
| Legacy token format | Custom propagation |

---

## 9. Common Gotchas

### Gotcha 1 — TokenRelay Not Working

```text
Problem:
Downstream token null

Fix:
OAuth2AuthorizedClientManager + oauth2Login
```

### Gotcha 2 — Same Token Everywhere

```text
Problem:
Blast radius high

Fix:
Token Exchange (service-specific)
```

### Gotcha 3 — Cookie Forwarded

```text
Problem:
Session cookie downstream

Fix:
RemoveRequestHeader=Cookie
```

### Gotcha 4 — Exchange Latency

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
| Token Relay kya hai? | User token unchanged downstream forward. |
| Token Exchange kya hai? | RFC 8693 — naya service-specific token. |
| Relay vs Exchange? | Same token vs naya token. |
| Blast radius? | Compromise impact scope. |
| Impersonation vs Delegation? | User ki identity vs user ki taraf se. |
| `act` claim? | Delegation — kaun act kar raha. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Simple forward | Token Relay |
| Service-specific scopes | Token Exchange |
| Blast radius kam | Token Exchange |
| Low latency | Token Relay |
| Internal token format | Custom propagation |

---

## 11. Key Takeaways

```text
1. Token Relay = same token forward.

2. Token Exchange = naya service-specific token.

3. RFC 8693 = token exchange standard.

4. Blast radius = Exchange kam.

5. TokenRelay= = default filter.

6. OAuth2AuthorizedClientManager = prerequisite.

7. Cookie remove = downstream clean.

8. act claim = delegation.

9. Exchange = extra Auth Server call.

10. Relay = low latency.
```

---

## 12. Next Part Preview

**Part 13: Token Exchange Pattern Deep Dive**

Isme hum cover karenge:

```text
- Token Exchange deep dive
- RFC 8693 implementation
- Impersonation vs Delegation
- Spring Authorization Server support
- Real-world scenarios
```