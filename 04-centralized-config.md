# Part 4: Microservices Security — Centralized Config with Spring Cloud Config

> Config Server ka core value — **Git-backed centralized config + encrypted secrets (`{cipher}`) + dynamic refresh (`@RefreshScope` + Spring Cloud Bus)** — secrets kabhi plain text me Git me nahi jaate. [1579][1581][1583][1587]

---

## 1. Centralized Config Mental Model

```text
Without Config Server:
Har service ka apna application.properties
Secrets scattered
Change = redeploy

With Config Server:
Git repo (single source of truth)
   |
   v
Config Server
   |
   +--> Service A
   +--> Service B
   +--> Service C
```

### Benefits

```text
1. Single source of truth
2. Environment-specific config
3. Encrypted secrets
4. Dynamic refresh (no restart)
5. Audit trail (Git history)
```

Config Server externalized configuration — central place, environment-specific, encrypted secrets. [1581][1583][1587]

---

## 2. Architecture

```text
Git Repo (config-repo)
   |
   | application.yml, service-a.yml, service-b.yml
   v
Config Server (port 8888)
   |
   | HTTP (encrypted props)
   v
Service A / B / C
   |
   | @RefreshScope
   v
Runtime refresh
```

### Config Server Responsibilities

| Responsibility | Detail |
|---|---|
| Git backend | Config files Git me |
| Encryption | `{cipher}` values decrypt |
| Environment-specific | dev/staging/prod |
| Dynamic refresh | Bus + @RefreshScope |

Config Server Git se config fetch karta hai, encrypted values decrypt karke clients ko deta hai. [1579][1581][1583]

---

## 3. Config Server Setup

### Dependency

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

### Main Class

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

### application.yml

```yaml
server:
  port: 8888

spring:
  cloud:
    config:
      server:
        git:
          uri: [https://github.com/org/config-repo](https://github.com/org/config-repo)
          username: ${GIT_USER}
          password: ${GIT_TOKEN}
          default-label: main

  security:
    user:
      name: config-admin
      password: ${CONFIG_PASSWORD}
```

`@EnableConfigServer` + Git URI + Basic Auth — config server ready. [1581][1583]

---

## 4. Git Repo Structure

```text
config-repo/
   |
   +--> application.yml          (common)
   +--> service-a.yml            (service-a specific)
   +--> service-a-dev.yml        (dev env)
   +--> service-a-prod.yml       (prod env)
   +--> service-b.yml
   +--> service-b-prod.yml
```

### application.yml (common)

```yaml
logging:
  level:
    root: INFO

management:
  endpoints:
    web:
      exposure:
        include: health,info,refresh
```

### service-a-prod.yml

```yaml
jwt:
  secret: '{cipher}AQBc8H7k2mN9pQrS...'
db:
  password: '{cipher}BQXd9I8j3nO0qRsT...'
```

`{cipher}` prefix = encrypted value — Config Server decrypt karke client ko deta hai. [1579][1580][1588]

---

## 5. Encryption Setup

### Symmetric Key

```yaml
encrypt:
  key: ${ENCRYPT_KEY}
```

```bash
export ENCRYPT_KEY=my-secret-key-123
```

### Asymmetric (RSA)

```yaml
encrypt:
  key-store:
    location: classpath:config-server.jks
    password: ${KEYSTORE_PASSWORD}
    alias: config-key
```

```bash
keytool -genkeypair -alias config-key \
  -keyalg RSA -keystore config-server.jks \
  -storepass ${KEYSTORE_PASSWORD}
```

Symmetric = single key, Asymmetric = RSA key pair — asymmetric zyada secure. [1579][1582][1588][1589]

---

## 6. Encrypt / Decrypt Endpoints

```bash
# Encrypt
curl -X POST http://config-admin:${CONFIG_PASSWORD}@localhost:8888/encrypt \
  -d "my-db-password"

# Response
AQBc8H7k2mN9pQrS...

# Decrypt
curl -X POST http://config-admin:${CONFIG_PASSWORD}@localhost:8888/decrypt \
  -d "AQBc8H7k2mN9pQrS..."
```

`/encrypt` + `/decrypt` endpoints — secured hone chahiye (Basic Auth). [1579][1580][1588]

---

## 7. Config Client Setup

### Dependency

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

### application.yml

```yaml
spring:
  application:
    name: service-a
  cloud:
    config:
      uri: http://config-admin:${CONFIG_PASSWORD}@localhost:8888
      fail-fast: true
```

### Config Service

```java
@RestController
@RefreshScope
@RequiredArgsConstructor
public class ConfigController {

    @Value("${feature.flag.enabled:false}")
    private boolean featureEnabled;

    @GetMapping("/config")
    public Map<String, Object> getConfig() {
        return Map.of("featureEnabled", featureEnabled);
    }
}
```

`@RefreshScope` — bean refresh scope me, config change par re-create. [1584][1586][1587][1593]

---

## 8. Dynamic Refresh (Spring Cloud Bus)

```text
Git commit
   |
   v
Config Server detect
   |
   v
/actuator/busrefresh
   |
   v
RabbitMQ / Kafka
   |
   +--> Service A refresh
   +--> Service B refresh
   +--> Service C refresh
```

### Trigger Refresh

```bash
curl -X POST http://config-admin:${CONFIG_PASSWORD}@localhost:8888/actuator/busrefresh
```

Spring Cloud Bus — ek call se saare instances refresh. [1584][1587][1590]

---

## 9. Security Best Practices

| Practice | Detail |
|---|---|
| Config Server secured | Basic Auth / mTLS |
| Secrets encrypted | `{cipher}` + RSA |
| Git repo private | Access control |
| Encrypt endpoint secured | Basic Auth only |
| Bus secured | AMQP credentials |
| Least privilege | Git token read-only |

### Complete Secure Setup

```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: [https://github.com/org/config-repo](https://github.com/org/config-repo)
          username: ${GIT_USER}
          password: ${GIT_TOKEN}
          clone-on-start: true
        encrypt:
          enabled: true
encrypt:
  key-store:
    location: classpath:config-server.jks
    password: ${KEYSTORE_PASSWORD}
    alias: config-key
```

---

## 10. Complete Flow

```text
1. Git me config commit (encrypted secrets)
2. Config Server fetch
3. Service start -> Config Server se fetch
4. Encrypted values decrypt
5. Runtime change -> Git commit
6. Bus refresh -> Saare instances update
```

---

## 11. Real-World Considerations

| Consideration | Solution |
|---|---|
| Config Server down | Local cache / fail-fast |
| Secret rotation | Re-encrypt + bus refresh |
| Multi-env | Git branches / profiles |
| Audit | Git history |
| Rollback | Git revert |

---

## 12. Common Gotchas

### Gotcha 1 — Cipher Not Decrypting

```text
Problem:
{cipher} value plain text me

Fix:
encrypt.key / keystore correct
```

### Gotcha 2 — Refresh Not Working

```text
Problem:
@RefreshScope missing

Fix:
Bean pe @RefreshScope add
```

### Gotcha 3 — Bus Not Propagating

```text
Problem:
Sirf ek instance refresh

Fix:
Spring Cloud Bus + RabbitMQ
```

### Gotcha 4 — Git Auth Fail

```text
Problem:
401 from Git

Fix:
Token / credentials correct
```

---

## 13. Interview Questions

| Question | Answer |
|---|---|
| Config Server kya hai? | Centralized externalized configuration. |
| Git backend kyu? | Version control + audit trail. |
| `{cipher}` kya hai? | Encrypted property prefix. |
| @RefreshScope kya hai? | Runtime config refresh. |
| Spring Cloud Bus kya? | Saare instances ko refresh propagate. |
| Symmetric vs Asymmetric? | Single key vs RSA key pair. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Secrets secure | `{cipher}` + RSA |
| Dynamic config | @RefreshScope + Bus |
| Multi-env | Git profiles |
| Audit trail | Git history |
| Config Server HA | Multiple instances + Git |

---

## 14. Key Takeaways

```text
1. Config Server = centralized config.

2. Git = single source of truth.

3. {cipher} = encrypted secrets.

4. RSA = asymmetric encryption.

5. @RefreshScope = runtime refresh.

6. Spring Cloud Bus = propagate refresh.

7. /encrypt + /decrypt = secured endpoints.

8. fail-fast = startup validation.

9. Git history = audit trail.

10. Secrets kabhi plain text nahi.
```

---

## 15. Next Part Preview

**Part 5: Security Boundaries & Trust Zones**

Isme hum cover karenge:

```text
- Trust zones deep dive
- Network segmentation
- Security boundaries
- Zero trust principles
- Real-world architecture
```