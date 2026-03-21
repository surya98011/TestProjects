# ⚡ Module 11: Redis Cache

> All examples use the **ShopEase** product-service.

---

## 1. Why Caching?

| Table Type | Operations | Cache? |
|---|---|---|
| **Transactional** | CRUD (Create, Read, Update, Delete) | Usually no |
| **Non-Transactional** | SELECT only (rarely changes) | ✅ Yes — perfect for caching |

> **ShopEase Example:** Product categories rarely change → Cache them in Redis instead of hitting MySQL every time.

---

## 2. Redis Overview

| Aspect | Details |
|---|---|
| **Type** | In-memory key-value data store |
| **Speed** | Sub-millisecond reads (vs ~10ms for MySQL) |
| **Data Format** | Key-Value pairs |
| **Use Case** | Cache frequently-read, rarely-changed data |

---

## 3. Setup

### pom.xml

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

### application.yml

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
  cache:
    type: redis
    redis:
      time-to-live: 3600000   # 1 hour TTL
```

---

## 4. ShopEase Product Service with Redis

### Enable Caching

```java
@SpringBootApplication
@EnableCaching   // ← Enable Redis caching
public class ProductServiceApplication { ... }
```

### ProductService with Cache Annotations

```java
@Service
@Slf4j
public class ProductService {

    @Autowired
    private ProductRepository productRepo;

    @Cacheable(value = "products", key = "'all'")   // ← Cache result
    public List<Product> getAllProducts() {
        log.info("Fetching from DATABASE (not cached yet)");
        return productRepo.findAll();
    }

    @Cacheable(value = "products", key = "#id")      // ← Cache by ID
    public Product getProductById(Long id) {
        log.info("Fetching product {} from DATABASE", id);
        return productRepo.findById(id)
                .orElseThrow(() -> new ProductNotFoundException("Not found: " + id));
    }

    @CacheEvict(value = "products", allEntries = true)  // ← Clear cache on update
    public Product createProduct(Product product) {
        log.info("Creating product — clearing cache");
        return productRepo.save(product);
    }

    @CacheEvict(value = "products", allEntries = true)
    public void deleteProduct(Long id) {
        productRepo.deleteById(id);
    }
}
```

### Cache Behavior

```
1st call: GET /api/products  → Hits MySQL → Stores in Redis → Returns data
2nd call: GET /api/products  → Reads from Redis (no DB call!) → Returns data
POST /api/products (create)  → Clears Redis cache
Next GET /api/products       → Hits MySQL again (cache was cleared)
```

| Annotation | Purpose |
|---|---|
| `@Cacheable` | Read from cache; if miss, call method & cache result |
| `@CacheEvict` | Remove entries from cache |
| `@CachePut` | Always call method, update cache with result |

---

*← [10 — Kafka](./10-apache-kafka.md) | [12 — Multi Threading →](./12-multithreading.md)*
