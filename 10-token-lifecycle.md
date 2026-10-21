# Part 10: Microservices Security — Token Lifecycle (Issue, Refresh, Revoke)

> Token lifecycle ka core — **short-lived access token + refresh token rotation + reuse detection + revocation (RFC 7009)** — RFC 9700 (Jan 2025) me refresh rotation mandatory hai. [1649][1656][1657][1659]

---

## 1. Token Lifecycle Mental Model

```text
Issue -> Use -> Expire -> Refresh -> Revoke

Access Token:
Issue -> Use -> Expire (5-15 min)

Refresh Token:
Issue -> Use -> Rotate -> Reuse? -> Revoke family
```

### Lifecycle Stages

| Stage | Access Token | Refresh Token |
|---|---|---|
| Issue | Login par | Login par |
| Use | Har API call | Refresh par |
| Expire | 5-15 min | Days/weeks |
| Refresh | Naya issue | Rotation (naya issue) |
| Revoke | Blacklist (jti) | DB delete / family revoke |

Access short-lived (stateless), refresh long-lived (stateful) — revocation control. [1649][1651][1659]

---

## 2. Token Issuance

```text
Login / Authorization Code
   |
   v
Auth Server
   |
   +--> Access Token (JWT, 15 min)
   +--> Refresh Token (opaque, 8 hours)
   +--> ID Token (user identity)
```

### Spring Authorization Server

```java
TokenSettings.builder()
    .accessTokenTimeToLive(Duration.ofMinutes(15))
    .refreshTokenTimeToLive(Duration.ofHours(8))
    .reuseRefreshTokens(false)  // Rotation enable
    .build();
```

`reuseRefreshTokens(false)` — rotation enable (har refresh par naya token). [1649][1656][1659]

---

## 3. Refresh Token Rotation

```text
1. Client -> Refresh Token (old)
2. Auth Server validate
3. Auth Server -> Naya Access + Naya Refresh
4. Old refresh token -> Invalidate
5. Client -> Naye tokens store
```

### Rotation Flow

```text
Login:     RT-1 issued
Refresh 1: RT-1 -> AT-2 + RT-2 (RT-1 invalid)
Refresh 2: RT-2 -> AT-3 + RT-3 (RT-2 invalid)
...
```

Rotation — har refresh par naya refresh token, purana invalidate. [1650][1656][1657][1659]

### Kyu Zaroori

```text
Bina rotation:
RT compromise -> Infinite access

With rotation:
RT compromise -> Ek refresh ke baad invalid
Reuse detection -> Family revoke
```

---

## 4. Reuse Detection

```text
Attacker steals RT-2
User bhi RT-2 use karta hai
   |
   v
Auth Server: RT-2 already used!
   |
   v
Token family revoke (RT-1, RT-2, RT-3...)
User re-login
```

### Implementation

```java
@Service
@RequiredArgsConstructor
public class RefreshTokenService {

    private final RefreshTokenRepository repository;

    public TokenResponse refresh(String oldToken) {
        RefreshToken token = repository.findByToken(oldToken)
            .orElseThrow(() -> new InvalidTokenException("Invalid refresh token"));

        if (token.isUsed()) {
            // Reuse detected -> Family revoke
            revokeFamily(token.getFamilyId());
            throw new TokenReuseException("Token reuse detected");
        }

        token.setUsed(true);
        repository.save(token);

        // Naya access + refresh issue
        return issueNewTokens(token.getUser(), token.getFamilyId());
    }

    private void revokeFamily(String familyId) {
        repository.revokeByFamilyId(familyId);
    }
}
```

Reuse detection — purana refresh token dobara use = theft signal = family revoke. [1650][1653][1656][1659]

### Token Family

```text
Login: Family-1
  RT-1 -> RT-2 -> RT-3 -> RT-4
  (Sab RT-1 family)

RT-2 reuse -> Family-1 revoke (RT-3, RT-4 bhi)
```

---

## 5. Token Revocation (RFC 7009)

```text
POST /oauth2/revoke
Content-Type: application/x-www-form-urlencoded
Authorization: Basic base64(client_id:client_secret)

token=RT-1&token_type_hint=refresh_token
```

### Response

```text
200 OK (idempotent — already revoked bhi 200)
```

### Spring Authorization Server

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

`/oauth2/revoke` — RFC 7009 standard, idempotent response. [1653][1654][1659]

### Revocation Use Cases

| Use Case | Action |
|---|---|
| Logout | Refresh token revoke |
| Password change | Saare tokens revoke |
| Account suspension | Saare tokens revoke |
| Anomaly detection | Family revoke |

---

## 6. JWT Blacklist (Access Token Revocation)

```text
Problem:
JWT stateless — revoke nahi ho sakta directly

Solution:
jti blacklist (Redis, TTL = token remaining life)
```

### Implementation

```java
@Service
@RequiredArgsConstructor
public class TokenBlacklistService {

    private final RedisTemplate<String, String> redis;

    public void blacklist(String jti, long remainingSeconds) {
        redis.opsForValue().set(
            "blacklist:" + jti,
            "revoked",
            Duration.ofSeconds(remainingSeconds)
        );
    }

    public boolean isBlacklisted(String jti) {
        return Boolean.TRUE.equals(
            redis.hasKey("blacklist:" + jti)
        );
    }
}
```

### Filter Integration

```java
@Component
@RequiredArgsConstructor
public class JwtBlacklistFilter extends OncePerRequestFilter {

    private final TokenBlacklistService blacklistService;

    @Override
    protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain chain
    ) throws ServletException, IOException {

        String token = extractToken(request);
        if (token != null) {
            String jti = extractJti(token);
            if (blacklistService.isBlacklisted(jti)) {
                response.setStatus(401);
                return;
            }
        }

        chain.doFilter(request, response);
    }
}
```

JWT blacklist — `jti` + Redis + TTL = token remaining life. [1648][1652][1655][1658]

### Trade-off

```text
JWT stateless (fast) + Blacklist (revocable)
= Best of both
```

---

## 7. Complete Lifecycle Flow

```text
1. Login -> Access (15 min) + Refresh (8 hours)
2. API calls -> Access token
3. Access expire -> Refresh token
4. Refresh -> Naya Access + Naya Refresh (rotation)
5. Old refresh invalidate
6. Reuse detect -> Family revoke
7. Logout -> Refresh revoke + Access blacklist
8. Password change -> Saare tokens revoke
```

---

## 8. Real-World Strategies

| Strategy | Use Case |
|---|---|
| Short access + rotation | Standard |
| Blacklist | Immediate revocation |
| Token family | Reuse detection |
| Introspection | Real-time validation |
| DPoP | Token binding |

### Introspection vs Blacklist

| Approach | Latency | Revocation |
|---|---|---|
| Local JWT validation | Fast | Delayed (blacklist) |
| Introspection | Slower (network) | Immediate |

Introspection — har request par Auth Server call — immediate revocation, lekin latency. [1649][1659]

---

## 9. Common Gotchas

### Gotcha 1 — No Rotation

```text
Problem:
Same refresh token reuse

Fix:
reuseRefreshTokens(false)
```

### Gotcha 2 — No Reuse Detection

```text
Problem:
Stolen token infinite use

Fix:
Token family + reuse detection
```

### Gotcha 3 — Blacklist TTL Wrong

```text
Problem:
Blacklist entry permanent / early expire

Fix:
TTL = token remaining life
```

### Gotcha 4 — Logout Sirf Access Revoke

```text
Problem:
Refresh token alive

Fix:
Refresh + Access dono revoke
```

---

## 10. Interview Questions

| Question | Answer |
|---|---|
| Token lifecycle? | Issue -> Use -> Expire -> Refresh -> Revoke. |
| Refresh rotation? | Har refresh par naya token, purana invalidate. |
| Reuse detection? | Purana token reuse = theft = family revoke. |
| Token family? | Ek login se saare refresh tokens. |
| JWT revocation? | jti blacklist (Redis + TTL). |
| RFC 7009? | Token revocation endpoint. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Token theft | Rotation + reuse detection |
| Immediate logout | Blacklist + refresh revoke |
| Password change | Saare tokens revoke |
| Stateless + revocable | JWT + jti blacklist |
| Real-time revocation | Introspection |

---

## 11. Key Takeaways

```text
1. Access short-lived, refresh long-lived.

2. Rotation = har refresh naya token.

3. Reuse detection = theft signal.

4. Token family = revoke all.

5. RFC 7009 = revocation endpoint.

6. jti blacklist = JWT revocation.

7. Redis + TTL = blacklist cleanup.

8. Introspection = real-time validation.

9. Logout = refresh + access revoke.

10. RFC 9700 = rotation mandatory.
```

---

## 12. Next Part Preview

**Part 11: Gateway as Security Edge**

Isme hum cover karenge:

```text
- Gateway security edge deep dive
- Authentication at gateway
- Authorization at gateway
- Token relay
- Rate limiting
- WAF integration
```