# Part 21: Microservices Security — BFF Pattern (Backend for Frontend)

> BFF ka core — **tokens server-side (BFF session), browser ko sirf `HttpOnly` session cookie** — XSS se token theft impossible, har frontend (Web/Mobile) ka apna optimized BFF. [1782][1784][1785][1789]

---

## 1. BFF Mental Model

```text
Without BFF:
SPA -> Tokens in browser (localStorage)
   |
   +--> XSS = Token theft = Full compromise

With BFF:
SPA -> BFF (session cookie only)
   |
   +--> Tokens server-side
   +--> XSS = Sirf session cookie (HttpOnly, JS read nahi kar sakta)
```

### Core Idea

```text
"Never hand a JWT to the browser"

Browser <-> BFF: Session cookie (opaque ID)
BFF <-> Services: Bearer JWT
```

BFF — tokens server-side, browser me sirf opaque session cookie. [1782][1784][1793]

---

## 2. Architecture

```text
Web SPA ----\
             \
Mobile App ---+--> BFF (per frontend) --> Microservices
             /
Desktop ----/
```

### Per-Frontend BFF

| Frontend | BFF | Optimization |
|---|---|---|
| Web SPA | Web BFF | JSON, chhote payloads |
| Mobile | Mobile BFF | Compressed, aggregated |
| Desktop | Desktop BFF | Full payloads |

BFF — har frontend ke liye dedicated backend, UX + security optimized. [1783][1786]

---

## 3. OAuth2 Flow with BFF

```text
1. SPA -> BFF /login
2. BFF -> Auth Server (Authorization Code + PKCE)
3. User -> Auth Server login
4. Auth Server -> BFF (code)
5. BFF -> Auth Server (code -> tokens)
6. BFF -> Tokens server-side store (session/Redis)
7. BFF -> SPA (HttpOnly session cookie)
8. SPA -> BFF (cookie) -> Service (Bearer JWT)
```

### Token Storage

| Token | Storage | Browser Access |
|---|---|---|
| Access | BFF session (Redis) | Nahi |
| Refresh | BFF session (Redis) | Nahi |
| ID | BFF session (Redis) | Nahi |
| Session ID | HttpOnly cookie | JS read nahi kar sakta |

BFF confidential client — tokens server-side, browser sirf session ID. [1784][1785][1790]

---

## 4. Spring Cloud Gateway BFF

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
  cloud:
    gateway:
      default-filters:
        - SaveSession
        - TokenRelay=
      routes:
        - id: order-service
          uri: http://localhost:8081
          predicates:
            - Path=/api/orders/**
```

`SaveSession` + `TokenRelay=` — BFF pattern ke 2 cornerstone filters. [1789][1791][1794]

### Kaise Kaam Karta Hai

```text
SaveSession:
- OAuth2 login ke baad session persist
- Tokens session me store

TokenRelay:
- Session cookie -> Session lookup
- Access token extract
- Bearer header downstream
```

`SaveSession` tokens persist, `TokenRelay` cookie -> Bearer conversion. [1789][1791][1795]

---

## 5. Session Cookie Security

```yaml
server:
  servlet:
    session:
      cookie:
        http-only: true
        secure: true
        same-site: strict
        name: SESSION
```

| Attribute | Purpose |
|---|---|
| `HttpOnly` | JS access block (XSS protection) |
| `Secure` | Sirf HTTPS |
| `SameSite=Strict` | CSRF protection |
| Short lifetime | Session hijack window kam |

`HttpOnly` + `Secure` + `SameSite=Strict` — XSS + CSRF + hijack protection. [1784][1787][1790]

---

## 6. CSRF with BFF

```text
Cookie-based auth = CSRF risk
BFF = CSRF token + SameSite cookie
```

```java
@Bean
public SecurityWebFilterChain filterChain(ServerHttpSecurity http) {
    http
        .csrf(csrf -> csrf
            .csrfTokenRepository(CookieServerCsrfTokenRepository.withHttpOnlyFalse())
        )
        .authorizeExchange(exchange -> exchange
            .anyExchange().authenticated()
        )
        .oauth2Login(Customizer.withDefaults());
    return http.build();
}
```

CSRF token + `SameSite=Strict` cookie — double protection. [1788][1790]

---

## 7. Complete Flow

```text
1. SPA -> BFF /login
2. BFF -> Keycloak (Authorization Code + PKCE)
3. Keycloak -> BFF (tokens)
4. BFF -> Redis session (tokens)
5. BFF -> SPA (HttpOnly cookie)
6. SPA -> BFF /api/orders (cookie)
7. BFF -> TokenRelay -> Order Service (Bearer JWT)
8. Order Service -> JWT validate
9. Response back
```

### XSS Scenario

```text
Attacker XSS inject
   |
   v
document.cookie read try
   |
   v
HttpOnly = Fail (JS access nahi)
Tokens server-side = Safe
```

XSS attack — HttpOnly cookie + server-side tokens = token theft impossible. [1785][1793]

---

## 8. BFF vs Direct Token (SPA)

| Aspect | Direct Token (SPA) | BFF |
|---|---|---|
| Token storage | localStorage/memory | Server-side |
| XSS impact | Token theft = full compromise | Session cookie (limited) |
| Refresh token | Browser me (risky) | Server-side |
| Confidential client | Nahi (public client) | Haan |
| Complexity | Low | Medium |
| Recommendation | Simple apps | Production SPAs |

BFF — production SPA ke liye recommended (token theft risk eliminate). [1784][1785][1793]

---

## 9. Real-World Example

```text
E-Commerce:
Web SPA -> Web BFF -> Order/Product/User Services
Mobile App -> Mobile BFF -> Same Services

Web BFF:
- Session cookie (HttpOnly)
- Tokens Redis
- TokenRelay downstream

Mobile BFF:
- PKCE + native token storage
- Compressed responses
```

---

## 10. Common Gotchas

### Gotcha 1 — Tokens in localStorage

```text
Problem:
XSS = token theft

Fix:
BFF pattern (server-side tokens)
```

### Gotcha 2 — Missing SaveSession

```text
Problem:
Tokens lost between requests

Fix:
SaveSession filter
```

### Gotcha 3 — SameSite=Lax

```text
Problem:
CSRF partial protection

Fix:
SameSite=Strict + CSRF token
```

### Gotcha 4 — Session Fixation

```text
Problem:
Session ID reuse

Fix:
Login par session ID rotate
```

---

## 11. Interview Questions

| Question | Answer |
|---|---|
| BFF kya hai? | Backend for Frontend — per-frontend backend. |
| Token kahan store? | Server-side (BFF session/Redis). |
| Browser me kya? | HttpOnly session cookie only. |
| XSS protection? | HttpOnly + server-side tokens. |
| CSRF protection? | SameSite=Strict + CSRF token. |
| SaveSession + TokenRelay? | BFF ke 2 cornerstone filters. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Production SPA | BFF pattern |
| Token theft prevent | HttpOnly + server-side |
| Mobile + Web | Separate BFFs |
| CSRF | SameSite + token |
| Session persistence | SaveSession + Redis |

---

## 12. Key Takeaways

```text
1. BFF = per-frontend backend.

2. Tokens server-side (session/Redis).

3. Browser = HttpOnly cookie only.

4. XSS = token theft impossible.

5. SaveSession + TokenRelay = core.

6. SameSite=Strict + CSRF token.

7. Confidential client (BFF).

8. Mobile = PKCE + native storage.

9. Production SPA = BFF recommended.

10. "Never hand JWT to browser".
```

---

## 13. Next Part Preview

**Part 22: Token Propagation Strategies**

Isme hum cover karenge:

```text
- Token propagation deep dive
- Header-based propagation
- Context propagation
- Async + token
- Message queues + token
```