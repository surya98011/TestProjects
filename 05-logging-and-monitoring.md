# 📝 Module 5: Logging & Monitoring

> All examples use the **ShopEase** e-commerce microservice project.

---

## 📑 Table of Contents

- [1. Why Logging?](#1-why-logging)
- [2. Logging Frameworks](#2-logging-frameworks)
- [3. Log Levels](#3-log-levels)
- [4. ShopEase Logging Configuration](#4-shopease-logging-configuration)
- [5. ShopEase Code with Logging](#5-shopease-code-with-logging)
- [6. Monitoring Log Files](#6-monitoring-log-files)

---

## 1. Why Logging?

| Purpose | Example |
|---|---|
| Understand runtime behavior | "OrderService.createOrder() was called with orderId=101" |
| Debug production issues | `grep -i "exception" order-service.log` |
| Audit trail | Track who did what and when |
| Performance monitoring | Log method execution times |

---

## 2. Logging Frameworks

| Framework | Status | Usage |
|---|---|---|
| **Log4J** | Legacy | Older projects |
| **SLF4J + Logback** | Current standard | Spring Boot default |
| **Log4J2** | Modern | High-performance needs |

> Spring Boot uses **SLF4J** as the logging facade and **Logback** as the implementation by default — no extra dependencies needed.

---

## 3. Log Levels

| Level | When to Use | ShopEase Example |
|---|---|---|
| `TRACE` | Ultra-detailed flow | Method entry/exit |
| `DEBUG` | Development debugging | `"Fetching product with id={}"` |
| `INFO` | Business milestones | `"Order created successfully: {}"` |
| `WARN` | Potential issues | `"Product stock low: {} units left"` |
| `ERROR` | Failures | `"Failed to process payment for order {}"` |

**Priority:** `TRACE < DEBUG < INFO < WARN < ERROR`

> Setting level to `INFO` means you see INFO, WARN, and ERROR — but not DEBUG or TRACE.

---

## 4. ShopEase Logging Configuration

### `application.yml` — Product Service

```yaml
# ShopEase Product Service — Logging Config
logging:
  level:
    root: INFO
    com.shopease.product: DEBUG        # Our package in DEBUG mode
    org.springframework: WARN          # Spring framework only warnings+
    org.hibernate.SQL: DEBUG           # Show SQL queries

  file:
    name: logs/product-service.log     # Log to file
    max-size: 10MB                     # Max file size before rotation
    max-history: 30                    # Keep 30 days of logs

  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
    file: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
```

---

## 5. ShopEase Code with Logging

### ProductService.java

```java
package com.shopease.product.service;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import java.util.List;

@Slf4j   // ← Lombok annotation — creates 'log' object automatically
@Service
public class ProductService {

    private final ProductRepository productRepo;

    public ProductService(ProductRepository productRepo) {
        this.productRepo = productRepo;
    }

    public List<Product> getAllProducts() {
        log.info("Fetching all products from database");
        List<Product> products = productRepo.findAll();
        log.debug("Found {} products", products.size());
        return products;
    }

    public Product getProductById(Long id) {
        log.info("Fetching product with id={}", id);
        return productRepo.findById(id)
                .orElseThrow(() -> {
                    log.error("Product not found with id={}", id);
                    return new ProductNotFoundException("Product not found: " + id);
                });
    }

    public Product createProduct(Product product) {
        log.info("Creating new product: name={}, price={}",
                 product.getName(), product.getPrice());
        Product saved = productRepo.save(product);
        log.info("Product created successfully with id={}", saved.getId());
        return saved;
    }
}
```

### Sample Log Output

```
2024-03-15 10:23:41 [http-nio-8081-exec-1] INFO  c.s.product.service.ProductService - Fetching all products from database
2024-03-15 10:23:41 [http-nio-8081-exec-1] DEBUG c.s.product.service.ProductService - Found 24 products
2024-03-15 10:23:45 [http-nio-8081-exec-2] INFO  c.s.product.service.ProductService - Fetching product with id=99
2024-03-15 10:23:45 [http-nio-8081-exec-2] ERROR c.s.product.service.ProductService - Product not found with id=99
```

---

## 6. Monitoring Log Files

### In Linux VM (Production)

```bash
# View last 100 lines of log
tail -n 100 /app/logs/product-service.log

# Follow logs in real-time
tail -f /app/logs/product-service.log

# Search for exceptions
grep -i "exception" /app/logs/product-service.log

# Search for specific order
grep "orderId=OD-101" /app/logs/order-service.log
```

### Enterprise Tools

| Tool | Purpose |
|---|---|
| **ELK Stack** (Elasticsearch + Logstash + Kibana) | Centralized log aggregation & search |
| **Splunk** | Enterprise log monitoring & alerting |
| **Grafana + Loki** | Lightweight log visualization |

> **💡 Pro Tip:** In microservices, use **Zipkin** for distributed tracing — it tracks a single request across multiple services using a unique trace ID.

---

*← [04 — Agile & JIRA](./04-agile-sdlc-jira.md) | [06 — JUnit Testing →](./06-junit-testing.md)*
