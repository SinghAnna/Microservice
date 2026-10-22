# Part 23: Microservices Security — Distributed Session Management

> Distributed sessions ka core — **Spring Session + Redis (shared session store)** — saare instances same Redis se session read/write karte hain; BFF pattern me session cookie + server-side tokens, stateless APIs me JWT. [177][1803][1809][1812]

---

## 1. Distributed Session Mental Model

```text
Problem (In-Memory Sessions):
Load Balancer
   |
   +--> Instance 1 (Session A)
   +--> Instance 2 (Session B)  <- User request yahan = Session not found!
   +--> Instance 3 (Session C)

Solution (Redis):
Load Balancer
   |
   +--> Instance 1 --\
   +--> Instance 2 ----> Redis (Shared Sessions)
   +--> Instance 3 --/
```

### Kaise Kaam Karta Hai

```text
1. User login -> Session Redis me store
2. Request -> Koi bhi instance
3. Instance -> Redis se session read
4. Response -> Session Redis me update
```

Spring Session — `HttpSession` ko Redis-backed bana deta hai, saare instances share. [175][1804][1809]

---

## 2. Session Strategies

| Strategy | Storage | Scalability | Complexity |
|---|---|---|---|
| Sticky Sessions | Instance memory | Limited | Low |
| Session Replication | Instance memory (replicated) | Medium | Medium |
| Centralized (Redis) | Redis | High | Medium |
| Stateless (JWT) | Token (client) | Highest | Medium |

### Comparison

| Aspect | Sticky | Replication | Redis | JWT |
|---|---|---|---|---|
| Session location | Instance | All instances | Redis | Client |
| Failover | Poor | Good | Excellent | Excellent |
| Scaling | Limited | Medium | High | Highest |
| Revocation | Easy | Easy | Easy | Hard (blacklist) |
| Recommendation | Avoid | Legacy | BFF apps | APIs |

Sticky = avoid, Redis = BFF apps, JWT = stateless APIs. [1803][1806][1813]

---

## 3. Spring Session + Redis Setup

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

### application.yml

```yaml
spring:
  session:
    redis:
      namespace: myapp:sessions
      flush-mode: on-save
    timeout: 30m
  data:
    redis:
      host: localhost
      port: 6379
      password: ${REDIS_PASSWORD}
```

Spring Boot auto-configures `RedisSessionRepository` — extra Java config nahi chahiye. [1812][1805]

### Namespace (Multi-App)

```text
Same Redis, multiple apps:
- myapp:sessions (Service A)
- otherapp:sessions (Service B)

Key collision prevent
```

`spring.session.redis.namespace` — multi-app Redis me key collision prevent. [177][1805]

---

## 4. Spring Security Integration

```text
Login -> SecurityContext -> Session (Redis)
   |
   v
SPRING_SECURITY_CONTEXT attribute
```

### Flow

```text
1. User login
2. SecurityContext create
3. Session attribute: SPRING_SECURITY_CONTEXT
4. Redis me persist
5. Next request -> Redis se read
6. SecurityContext restore
```

Spring Security `SecurityContextPersistenceFilter` — SecurityContext ko session (Redis) me save. [175][1809]

### Session Cookie

```yaml
server:
  servlet:
    session:
      cookie:
        name: SESSION
        http-only: true
        secure: true
        same-site: strict
```

Cookie name `SESSION` (JSESSIONID nahi) — Spring Session ka default. [1814][1812]

---

## 5. BFF Pattern + Distributed Sessions

```text
SPA -> BFF (session cookie)
   |
   v
BFF Session (Redis)
   |
   +--> Access Token
   +--> Refresh Token
   +--> User Info
   |
   v
TokenRelay -> Services
```

### Kaise Kaam Karta Hai

```text
1. Login -> Tokens Redis session me
2. SPA -> Cookie -> BFF
3. BFF -> Redis se tokens
4. TokenRelay -> Service (Bearer JWT)
5. Service -> JWT validate
```

BFF + Redis session — tokens server-side, browser sirf cookie. [1789][1806]

---

## 6. Session Expiry & Cleanup

```text
Session TTL: 30 min (idle)
Redis TTL: Auto-expire
```

### Configuration

```yaml
spring:
  session:
    timeout: 30m
    redis:
      flush-mode: on-save
```

### Flush Modes

| Mode | Behavior |
|---|---|
| `ON_SAVE` | Response commit par save (default) |
| `IMMEDIATE` | Har change turant save |

`ON_SAVE` — response commit par Redis write (efficient). [1805][1812]

---

## 7. Session Security Best Practices

| Practice | Detail |
|---|---|
| HttpOnly cookie | JS access block |
| Secure flag | HTTPS only |
| SameSite=Strict | CSRF |
| Short TTL | 30 min idle |
| Rotation on login | Session fixation |
| Redis TLS | Encryption in transit |
| Namespace isolation | Multi-app |

### Session Fixation Protection

```java
http.sessionManagement(session -> session
    .sessionFixation(fixation -> fixation.changeSessionId())
);
```

Session fixation — login par session ID change. [1806]

---

## 8. Complete Flow

```text
1. User -> BFF /login
2. BFF -> Auth Server (tokens)
3. BFF -> Redis session (tokens + SecurityContext)
4. BFF -> SPA (HttpOnly SESSION cookie)
5. SPA -> BFF (cookie)
6. BFF -> Redis se session read
7. TokenRelay -> Service (Bearer JWT)
8. Service -> JWT validate
9. Response
```

---

## 9. Real-World Patterns

| Pattern | Use Case |
|---|---|
| Redis + BFF | Web SPA + session |
| JWT stateless | REST APIs |
| Hybrid | BFF (session) + Services (JWT) |
| Multi-region | Redis cluster + geo-replication |

### Hybrid Pattern (Recommended)

```text
SPA -> BFF (Redis session) -> Services (JWT stateless)
```

Hybrid — BFF session (user context) + services stateless (JWT). [1803][1806]

---

## 10. Common Gotchas

### Gotcha 1 — In-Memory Sessions

```text
Problem:
Multi-instance = session loss

Fix:
Spring Session + Redis
```

### Gotcha 2 — Namespace Missing

```text
Problem:
Multi-app key collision

Fix:
spring.session.redis.namespace
```

### Gotcha 3 — Serialization Issues

```text
Problem:
Class version mismatch

Fix:
JSON serialization / namespace per app
```

### Gotcha 4 — Redis Down

```text
Problem:
Saare sessions lost

Fix:
Redis HA (cluster + replication)
```

Serialization — Java serialization problematic multi-app — JSON / namespace. [177]

---

## 11. Interview Questions

| Question | Answer |
|---|---|
| Distributed session kya hai? | Sessions shared store me (Redis). |
| Spring Session kya karta hai? | HttpSession ko Redis-backed banata hai. |
| Sticky vs Redis? | Instance-bound vs shared. |
| Namespace kyu? | Multi-app key collision prevent. |
| BFF + Redis? | Tokens server-side, cookie browser. |
| JWT vs Session? | Stateless vs stateful (revocable). |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Multi-instance web app | Spring Session + Redis |
| Stateless API | JWT |
| BFF + tokens | Redis session + TokenRelay |
| Multi-app Redis | Namespace |
| Redis HA | Cluster + replication |

---

## 12. Key Takeaways

```text
1. Distributed session = shared store (Redis).

2. Spring Session = HttpSession -> Redis.

3. Namespace = multi-app isolation.

4. SPRING_SECURITY_CONTEXT = session attribute.

5. BFF + Redis = tokens server-side.

6. Hybrid = BFF session + JWT services.

7. HttpOnly + Secure + SameSite.

8. ON_SAVE = efficient flush.

9. Redis HA = cluster.

10. Stateless = JWT (APIs).
```

---

## 13. Next Part Preview

**Part 24: Multi-Tenancy Security**

Isme hum cover karenge:

```text
- Multi-tenancy deep dive
- Tenant isolation patterns
- Tenant context propagation
- Database isolation
- Real-world strategies
```