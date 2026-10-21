# Part 3: Microservices Security — Service Discovery & Secure Service Registry

> Service discovery ka core idea — services **registry me register** hoti hain, doosri services **name se lookup** karti hain — Eureka me security ke liye **Basic Auth + TLS + CSRF ignore for `/eureka/**`** configure karna hota hai. [1564][1567][1572][1578]

---

## 1. Service Discovery Mental Model

```text
Without Discovery:
Service A -> http://192.168.1.10:8081 (hardcoded)
(Scaling / IP change = break)

With Discovery:
Service A -> "order-service" (name)
         -> Registry lookup
         -> Actual instance
```

### Kaise Kaam Karta Hai

```text
1. Service start -> Registry me register (name, host, port)
2. Heartbeat har 30 sec -> "main alive hoon"
3. Doosri service -> Name se lookup
4. Load balancer -> Instance select
5. Request -> Instance
```

Eureka client startup par register hota hai, heartbeat bhejta hai, registry cache karta hai. [1567][1572][1578]

---

## 2. Eureka Architecture

```text
+------------------+
|  Eureka Server   |
|  (Registry)      |
+------------------+
   ^          ^
   | register | lookup
   |          |
+--+--+    +--+--+
|Svc A|    |Svc B|
+-----+    +-----+
```

### Components

| Component | Role |
|---|---|
| Eureka Server | Registry maintain |
| Eureka Client | Register + lookup |
| Heartbeat | Liveness signal |
| Registry Cache | Client-side copy |

Eureka Server registry maintain karta hai, clients register + lookup karte hain. [1564][1566][1572]

---

## 3. Eureka Server Setup

### Dependency

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

### Main Class

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

### application.yml

```yaml
server:
  port: 8761

eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
  instance:
    hostname: localhost
```

`@EnableEurekaServer` + `register-with-eureka: false` — standalone server. [1564][1569]

---

## 4. Securing Eureka Server

```java
@Configuration
@EnableWebSecurity
public class EurekaSecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf
                .ignoringRequestMatchers("/eureka/**")
            )
            .authorizeHttpRequests(auth -> auth
                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults());
        return http.build();
    }
}
```

```yaml
spring:
  security:
    user:
      name: eureka-admin
      password: ${EUREKA_PASSWORD}
```

**Critical:** `/eureka/**` ke liye CSRF ignore — Eureka clients CSRF token nahi bhejte. [1564][1569][1571][1574]

---

## 5. Eureka Client Setup

### Dependency

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

### Main Class

```java
@SpringBootApplication
@EnableDiscoveryClient
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

### application.yml

```yaml
spring:
  application:
    name: order-service

eureka:
  client:
    service-url:
      defaultZone: http://eureka-admin:${EUREKA_PASSWORD}@localhost:8761/eureka
  instance:
    prefer-ip-address: true
```

`defaultZone` me credentials embed — Basic Auth auto add hota hai. [1564][1567][1574]

---

## 6. Secure Communication (TLS)

```yaml
eureka:
  client:
    tls:
      enabled: true
      key-store: classpath:client-keystore.p12
      key-store-type: PKCS12
      key-store-password: ${KEYSTORE_PASSWORD}
      trust-store: classpath:client-truststore.p12
      trust-store-password: ${TRUSTSTORE_PASSWORD}
  instance:
    non-secure-port-enabled: false
    secure-port-enabled: true
    secure-port: 8761
```

Eureka client TLS — `eureka.client.tls.enabled=true` + keystore/truststore. [1564][1567][1568]

---

## 7. Service-to-Service Communication

```java
@Service
@RequiredArgsConstructor
public class ProductServiceClient {

    private final WebClient.Builder webClientBuilder;

    public Product getProduct(Long id) {
        return webClientBuilder.build()
            .get()
            .uri("http://product-service/api/products/" + id)
            .retrieve()
            .bodyToMono(Product.class)
            .block();
    }
}
```

### Load Balanced WebClient

```java
@Bean
@LoadBalanced
public WebClient.Builder webClientBuilder() {
    return WebClient.builder();
}
```

`@LoadBalanced` + service name — Eureka se instance resolve + load balance. [1565][1573][1575][1578]

---

## 8. Spring Cloud LoadBalancer

```text
Service A -> "product-service"
   |
   v
LoadBalancer -> Instance select (round-robin)
   |
   v
Product Service Instance
```

### Load Balancing Strategies

| Strategy | Behavior |
|---|---|
| Round Robin | Sequential (default) |
| Random | Random instance |
| Zone-aware | Same zone prefer |

`Spring Cloud LoadBalancer` — discovery-based instance list + load balancing. [1565][1573][1575]

---

## 9. Complete Flow

```text
1. Order Service start
2. Eureka Server me register
3. Heartbeat har 30 sec
4. Product Service bhi register
5. Order -> "product-service" lookup
6. LoadBalancer -> Instance select
7. Request -> Product Service
8. Response back
```

---

## 10. Real-World Considerations

| Consideration | Solution |
|---|---|
| Eureka Server down | Client-side cache |
| Service instance down | Heartbeat miss -> evict |
| Multiple instances | Load balancer |
| Security | Basic Auth + TLS |
| High availability | Eureka cluster (peer replication) |

Eureka peer replication se high availability — har server bhi client hai. [1567][1572][1576]

---

## 11. Common Gotchas

### Gotcha 1 — 401 on Register

```text
Problem:
Eureka client register fail

Fix:
defaultZone me credentials embed
```

### Gotcha 2 — CSRF Error

```text
Problem:
Eureka server CSRF block

Fix:
/eureka/** ke liye CSRF ignore
```

### Gotcha 3 — Service Not Found

```text
Problem:
Name se lookup fail

Fix:
spring.application.name match karo
```

### Gotcha 4 — Stale Instance

```text
Problem:
Dead instance call

Fix:
Heartbeat + eviction config tune
```

---

## 12. Interview Questions

| Question | Answer |
|---|---|
| Service discovery kya hai? | Services registry me register + name se lookup. |
| Eureka kya hai? | Netflix service discovery server + client. |
| Heartbeat kya hai? | Liveness signal (30 sec). |
| Load balancing kaise? | Spring Cloud LoadBalancer + @LoadBalanced. |
| Eureka secure kaise? | Basic Auth + TLS + CSRF ignore for /eureka/**. |
| Eureka down ho to? | Client-side cache se kaam chalta hai. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Dynamic scaling | Service discovery |
| Service lookup | Eureka + @LoadBalanced |
| Secure registry | Basic Auth + TLS |
| High availability | Eureka cluster |
| Dead instance | Heartbeat + eviction |

---

## 13. Key Takeaways

```text
1. Service discovery = register + lookup.

2. Eureka = server + client.

3. Heartbeat = liveness (30 sec).

4. @LoadBalanced = client-side LB.

5. Basic Auth = Eureka security.

6. /eureka/** CSRF ignore.

7. TLS = secure communication.

8. Client-side cache = resilience.

9. Peer replication = HA.

10. Service name = logical address.
```

---

## 14. Next Part Preview

**Part 4: Centralized Config with Spring Cloud Config**

Isme hum cover karenge:

```text
- Spring Cloud Config Server
- Git-backed configuration
- Encrypted properties
- Config security
- Dynamic refresh
```