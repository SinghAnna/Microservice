# Part 24: Microservices Security — Multi-Tenancy Security

> Multi-tenancy ka core rule — **ek tenant kabhi doosre tenant ka data nahi dekh sakta** — 3 isolation models (shared schema + `tenant_id`, schema-per-tenant, database-per-tenant) aur tenant context JWT claim se propagate hota hai. [1816][1815][1821]

---

## 1. Multi-Tenancy Mental Model

```text
Single-Tenant:
App -> Tenant A data (dedicated)

Multi-Tenant:
App -> Tenant A data
    -> Tenant B data
    -> Tenant C data
    (Isolation critical!)
```

### Golden Rule

```text
"Ek tenant kabhi doosre tenant ka data nahi dekh sakta"
```

Multi-tenancy — ek app instance, multiple tenants, data isolation mandatory. [1816][1821]

---

## 2. Isolation Models

| Model | Isolation | Cost | Complexity | Use Case |
|---|---|---|---|---|
| Shared DB + Shared Schema | Row-level (`tenant_id`) | Low | Medium | Most SaaS |
| Shared DB + Schema-per-Tenant | Schema-level | Medium | High | Mid-tier |
| Database-per-Tenant | Full isolation | High | High | Enterprise/regulated |

### Comparison

| Aspect | Shared Schema | Schema-per-Tenant | DB-per-Tenant |
|---|---|---|---|
| Isolation | Row-level | Schema-level | Full |
| Cost | Low | Medium | High |
| Scaling | Easy | Medium | Hard |
| Noisy neighbor | Yes | Partial | No |
| Compliance | RLS needed | Good | Best |

Shared schema + RLS = most common new SaaS; DB-per-tenant = regulated industries. [1815][1818][1821][1828]

---

## 3. Model 1 — Shared Schema + tenant_id

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    tenant_id VARCHAR(50) NOT NULL,
    user_id BIGINT NOT NULL,
    amount DECIMAL(10,2),
    created_at TIMESTAMP
);

CREATE INDEX idx_orders_tenant ON orders(tenant_id);
```

### Query Filtering

```java
// Har query me tenant filter
@Query("SELECT o FROM Order o WHERE o.tenantId = :tenantId")
List<Order> findByTenant(@Param("tenantId") String tenantId);
```

### Risk

```text
Missing WHERE tenant_id = ?
= Data leak across tenants!
```

Shared schema — har table `tenant_id` column, har query filter — miss = leak. [1818][1821][1827]

---

## 4. Row-Level Security (RLS)

```sql
-- PostgreSQL RLS
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON orders
USING (tenant_id = current_setting('app.current_tenant')::uuid);
```

### Kaise Kaam Karta Hai

```text
1. App -> SET app.current_tenant = 'tenant-1'
2. Query -> SELECT * FROM orders
3. DB automatically -> WHERE tenant_id = 'tenant-1'
4. Doosre tenant rows = invisible
```

RLS — database-level enforcement, app-level filter miss ho to bhi safe. [1816][1821][1826][1827]

### Spring + RLS

```java
@Bean
public DataSource dataSource() {
    // Connection interceptor: SET app.current_tenant
}
```

---

## 5. Model 2 — Schema-per-Tenant

```text
Database: saas_db
   |
   +--> Schema: tenant_acme
   |       +--> orders, users, products
   |
   +--> Schema: tenant_beta
           +--> orders, users, products
```

### Hibernate Multi-Tenancy

```java
public class SchemaMultiTenantConnectionProvider
    implements MultiTenantConnectionProvider {

    @Override
    public Connection getConnection(String tenantId) {
        Connection connection = dataSource.getConnection();
        connection.setSchema("tenant_" + tenantId);
        return connection;
    }
}
```

Schema-per-tenant — `search_path` / schema set per connection, query filter nahi chahiye. [1818][1824][1826]

---

## 6. Model 3 — Database-per-Tenant

```text
Tenant A -> DB-A
Tenant B -> DB-B
Tenant C -> DB-C
```

### Routing DataSource

```java
public class TenantRoutingDataSource
    extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        return TenantContext.getTenantId();
    }
}
```

Database-per-tenant — `AbstractRoutingDataSource` se tenant-specific DB route. [1818][1820][1825]

---

## 7. Tenant Context Propagation

```text
JWT claim: tenant_id
   |
   v
Filter -> TenantContext (ThreadLocal)
   |
   v
Service / Repository / DB
```

### TenantContext

```java
public class TenantContext {

    private static final ThreadLocal<String> CURRENT_TENANT =
        new ThreadLocal<>();

    public static void setTenantId(String tenantId) {
        CURRENT_TENANT.set(tenantId);
    }

    public static String getTenantId() {
        return CURRENT_TENANT.get();
    }

    public static void clear() {
        CURRENT_TENANT.remove();
    }
}
```

### JWT Filter

```java
@Component
public class TenantFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain chain
    ) throws ServletException, IOException {

        String token = extractToken(request);
        String tenantId = extractTenantClaim(token);

        TenantContext.setTenantId(tenantId);
        try {
            chain.doFilter(request, response);
        } finally {
            TenantContext.clear();  // Thread reuse = leak prevent
        }
    }
}
```

Tenant context — JWT claim -> ThreadLocal -> har layer, finally me clear. [1817][1819][1823]

### Hibernate Filter

```java
@Entity
@FilterDef(name = "tenantFilter", parameters = @ParamDef(name = "tenantId", type = String.class))
@Filter(name = "tenantFilter", condition = "tenant_id = :tenantId")
public class Order {
    // ...
}

// Session me enable
session.enableFilter("tenantFilter")
    .setParameter("tenantId", TenantContext.getTenantId());
```

Hibernate `@Filter` — automatic tenant filtering, query miss impossible. [1817][1820][1822]

---

## 8. Tenant Identification

| Method | Example |
|---|---|
| JWT claim | `tenant_id: tenant-1` |
| Subdomain | `tenant1.app.com` |
| Header | `X-Tenant-Id: tenant-1` |
| URL path | `/tenant-1/api/...` |

### Best Practice

```text
JWT claim (signed) > Header (spoof-able) > Subdomain
```

Tenant identification — JWT claim (signed) best, header spoof-able. [1820][1823]

---

## 9. Complete Flow

```text
1. User -> Gateway (JWT with tenant_id)
2. Gateway -> Service (Token Relay)
3. Service -> TenantFilter -> TenantContext
4. Hibernate Filter / RLS -> Auto tenant filter
5. Query -> Sirf tenant data
6. Response
```

### Defense in Depth

```text
Layer 1: JWT tenant claim (identity)
Layer 2: TenantContext (propagation)
Layer 3: Hibernate Filter (query)
Layer 4: RLS (database)
```

Defense in depth — JWT + context + Hibernate filter + RLS. [1816][1821][1827]

---

## 10. Common Gotchas

### Gotcha 1 — Missing Tenant Filter

```text
Problem:
Query me tenant_id nahi = leak

Fix:
Hibernate Filter / RLS (automatic)
```

### Gotcha 2 — ThreadLocal Leak

```text
Problem:
Thread reuse = purana tenant

Fix:
finally me TenantContext.clear()
```

### Gotcha 3 — Header Trust

```text
Problem:
X-Tenant-Id spoof

Fix:
JWT claim (signed) use karo
```

### Gotcha 4 — Noisy Neighbor

```text
Problem:
Ek tenant sab resources kha raha

Fix:
Rate limiting per tenant + resource quotas
```

ThreadLocal leak — finally me clear, warna thread reuse se cross-tenant leak. [1817][1819]

---

## 11. Interview Questions

| Question | Answer |
|---|---|
| Multi-tenancy kya hai? | Ek app, multiple tenants, data isolation. |
| 3 isolation models? | Shared schema, schema-per-tenant, DB-per-tenant. |
| RLS kya hai? | Row-Level Security — DB-level tenant filter. |
| Tenant context kaise? | JWT claim -> ThreadLocal -> Hibernate filter. |
| ThreadLocal leak? | finally me clear. |
| Noisy neighbor? | Rate limiting per tenant. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Most SaaS | Shared schema + RLS |
| Enterprise/regulated | DB-per-tenant |
| Tenant isolation | RLS + Hibernate Filter |
| Tenant identification | JWT claim |
| Noisy neighbor | Per-tenant rate limiting |

---

## 12. Key Takeaways

```text
1. Multi-tenancy = tenant isolation.

2. 3 models: shared schema, schema-per-tenant, DB-per-tenant.

3. RLS = DB-level enforcement.

4. Tenant context = JWT -> ThreadLocal.

5. Hibernate @Filter = automatic.

6. finally clear = leak prevent.

7. JWT claim > header (security).

8. Noisy neighbor = rate limiting.

9. Defense in depth = JWT + context + filter + RLS.

10. Enterprise = DB-per-tenant.
```

---

## 13. Next Part Preview

**Part 25: API Versioning & Deprecation Security**

Isme hum cover karenge:

```text
- API versioning strategies
- Deprecation security
- Breaking changes
- Version-specific auth
- Real-world patterns
```