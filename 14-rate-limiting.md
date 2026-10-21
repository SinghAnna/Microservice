# Part 14: Microservices Security — Rate Limiting & Throttling at Gateway

> Rate limiting ka core — **Token Bucket (bursts allow) vs Sliding Window (smooth)** — distributed ke liye **Redis + Lua script (atomic)**, Spring Cloud Gateway me `RequestRateLimiter` + `KeyResolver`. [1697][1698][1699][1705]

---

## 1. Rate Limiting Mental Model

```text
Without Rate Limiting:
Attacker -> 10,000 req/sec -> Service overload
Brute force -> Password cracking
Scraping -> Data theft

With Rate Limiting:
Attacker -> 10,000 req/sec -> 429 (Too Many Requests)
Legit user -> 10 req/sec -> 200 OK
```

### Kya Rokta Hai

```text
- Brute force attacks
- DDoS / resource exhaustion
- API scraping
- Credential stuffing
- Cost control (per-tier limits)
```

Rate limiting — abuse, brute force, DDoS, cost control ke liye. [1698][1699][1706]

---

## 2. Rate Limiting Algorithms

| Algorithm | Bursts | Memory | Accuracy | Use Case |
|---|---|---|---|---|
| Token Bucket | Allow (controlled) | Low | Good | APIs (standard) |
| Leaky Bucket | No (smooth) | Low | Perfect | Strict rate |
| Fixed Window | Boundary spike | Low | Poor | Simple |
| Sliding Window Log | No | High | Perfect | Exact |
| Sliding Window Counter | Minor smoothing | Low | Good | Recommended default |

### Token Bucket

```text
Bucket capacity: 20 tokens
Refill rate: 10 tokens/sec

Request -> Token available? -> Yes: consume, process
                            -> No: 429
```

Token Bucket — bursts allow (capacity tak), long-term average rate maintain. [1697][1698][1701]

### Sliding Window Counter

```text
Current window + Previous window (weighted)
= Smooth approximation
```

Sliding Window Counter — low memory + good accuracy — recommended default. [1700][1706]

---

## 3. Spring Cloud Gateway Rate Limiter

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
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
                redis-rate-limiter.requestedTokens: 1
                key-resolver: "#{@userKeyResolver}"
```

### Options

| Option | Meaning |
|---|---|
| `replenishRate` | Sustained rate (tokens/sec) |
| `burstCapacity` | Max burst (bucket size) |
| `requestedTokens` | Cost per request (default 1) |
| `key-resolver` | Kaun limit (user/IP/API) |

`RequestRateLimiter` — Token Bucket based, Redis backend. [1697][1703]

---

## 4. KeyResolver (Per-User / Per-IP)

```java
@Configuration
public class RateLimiterConfig {

    // Per-user (JWT sub)
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> {
            String userId = exchange.getRequest().getHeaders()
                .getFirst("X-User-Id");
            if (userId != null) {
                return Mono.just("user:" + userId);
            }
            return Mono.just("anonymous");
        };
    }

    // Per-IP
    @Bean
    public KeyResolver ipKeyResolver() {
        return exchange -> Mono.just(
            "ip:" + exchange.getRequest().getRemoteAddress()
                .getAddress().getHostAddress()
        );
    }

    // Per-API + User
    @Bean
    public KeyResolver apiUserKeyResolver() {
        return exchange -> {
            String path = exchange.getRequest().getPath().value();
            String userId = exchange.getRequest().getHeaders()
                .getFirst("X-User-Id");
            return Mono.just(path + ":" + userId);
        };
    }
}
```

KeyResolver — per-user, per-IP, per-API+user — granularity decide. [1697][1703][1704]

### Key Strategy

```text
Authenticated: user:{sub}
Anonymous: ip:{ip}
Premium: tier:{tier}:{user}
API-specific: {path}:{user}
```

---

## 5. Redis + Lua (Atomic)

```lua
-- Token Bucket Lua Script (atomic)
local key = KEYS
local rate = tonumber(ARGV)
local capacity = tonumber(ARGV)
local now = tonumber(ARGV)
local requested = tonumber(ARGV)

local bucket = redis.call('HMGET', key, 'tokens', 'timestamp')
local tokens = tonumber(bucket) or capacity
local last = tonumber(bucket) or now

-- Refill
local elapsed = math.max(0, now - last)
tokens = math.min(capacity, tokens + elapsed * rate)

local allowed = tokens >= requested
if allowed then
    tokens = tokens - requested
end

redis.call('HMSET', key, 'tokens', tokens, 'timestamp', now)
redis.call('EXPIRE', key, math.ceil(capacity / rate) * 2)

return {allowed, tokens}
```

Lua script — Redis single-threaded, atomic check + update (race condition nahi). [1696][1702][1705][1707]

### Kyu Lua?

```text
GET + check + SET = race condition (multi-instance)
Lua script = atomic (single operation)
```

---

## 6. Distributed Rate Limiting

```text
Gateway Instance 1 ----\
                         \
Gateway Instance 2 --------> Redis (Shared State)
                         /
Gateway Instance 3 ----/
```

### Kyu Redis?

```text
In-memory (per-instance):
3 instances = 3x limit (30 req/sec actual)

Redis (shared):
3 instances = 1x limit (10 req/sec actual)
```

Distributed rate limiting — Redis shared state, saare instances same limit. [1696][1705][1707]

---

## 7. Response Headers

```text
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 10
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1735689660
Retry-After: 1
```

### Headers

| Header | Meaning |
|---|---|
| `X-RateLimit-Limit` | Total limit |
| `X-RateLimit-Remaining` | Bacha hua |
| `X-RateLimit-Reset` | Reset time |
| `Retry-After` | Kitni der baad try |

Rate limit headers — client ko transparency, Retry-After se backoff. [1697][1698]

---

## 8. Tiered Rate Limiting

```text
Free Tier:    10 req/min
Pro Tier:     100 req/min
Enterprise:   1000 req/min
```

### Implementation

```java
@Bean
public KeyResolver tieredKeyResolver() {
    return exchange -> {
        String tier = exchange.getRequest().getHeaders()
            .getFirst("X-Tier");
        String userId = exchange.getRequest().getHeaders()
            .getFirst("X-User-Id");
        return Mono.just(tier + ":" + userId);
    };
}
```

```yaml
routes:
  - id: free-tier
    uri: http://localhost:8081
    filters:
      - name: RequestRateLimiter
        args:
          redis-rate-limiter.replenishRate: 1
          redis-rate-limiter.burstCapacity: 10
```

Tiered limits — subscription-based, alag routes/filters. [1697][1706]

---

## 9. Complete Flow

```text
1. Request -> Gateway
2. KeyResolver -> Key (user/IP/API)
3. Redis Lua -> Atomic check
4. Token available? -> Yes: forward
                    -> No: 429 + Retry-After
5. Response headers
```

---

## 10. Real-World Patterns

| Pattern | Use Case |
|---|---|
| Per-user | Fair usage |
| Per-IP | Anonymous / brute force |
| Per-API | Endpoint-specific |
| Tiered | Subscription plans |
| Adaptive | Load-based dynamic |

### Adaptive Rate Limiting

```text
Normal load: 100 req/sec
High load: 50 req/sec
Overload: 10 req/sec
```

Adaptive — system load ke basis par dynamic limits. [1707]

---

## 11. Common Gotchas

### Gotcha 1 — In-Memory Rate Limiter

```text
Problem:
Multi-instance = limit x N

Fix:
Redis distributed limiter
```

### Gotcha 2 — KeyResolver Null

```text
Problem:
Saare requests deny

Fix:
Default key (IP) return karo
```

### Gotcha 3 — Race Condition

```text
Problem:
GET + SET separate = bypass

Fix:
Lua script (atomic)
```

### Gotcha 4 — 429 Without Retry-After

```text
Problem:
Client retry storm

Fix:
Retry-After header
```

---

## 12. Interview Questions

| Question | Answer |
|---|---|
| Rate limiting kya hai? | Request rate control — abuse prevent. |
| Token Bucket vs Sliding Window? | Bursts allow vs smooth. |
| KeyResolver kya? | Kaun limit (user/IP/API). |
| Distributed rate limiting? | Redis shared state. |
| Lua script kyu? | Atomic (race condition prevent). |
| 429 + Retry-After? | Client backoff. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Brute force | Per-IP + per-user |
| API fairness | Per-user |
| Subscription tiers | Tiered limits |
| Multi-instance | Redis distributed |
| Burst