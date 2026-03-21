# 🏗️ Module 8: Microservices Architecture

> All examples use the **ShopEase** e-commerce microservice project.

---

## 📑 Table of Contents

- [1. Monolith vs Microservices](#1-monolith-vs-microservices)
- [2. ShopEase Architecture Overview](#2-shopease-architecture-overview)
- [3. Service Registry (Eureka)](#3-service-registry-eureka)
- [4. Admin Server](#4-admin-server)
- [5. Zipkin — Distributed Tracing](#5-zipkin--distributed-tracing)
- [6. FeignClient — Interservice Communication](#6-feignclient--interservice-communication)
- [7. Load Balancing](#7-load-balancing)
- [8. API Gateway](#8-api-gateway)
- [9. Config Server](#9-config-server)
- [10. Circuit Breaker (Resilience4J)](#10-circuit-breaker-resilience4j)

---

## 1. Monolith vs Microservices

### Monolith Architecture
All functionalities in a **single application** (one WAR/JAR, one database, one deployment).

### Drawbacks of Monolith

| Problem | Impact |
|---|---|
| Burden on server | Single server handles everything |
| Response delay | Slow under heavy load |
| Server crash | All features go down |
| Single point of failure | One bug can crash entire app |
| Technology dependent | Entire app must use same tech |
| Re-deploy entire app | Even for small changes |

### Microservices Architecture

| Advantage | How ShopEase Benefits |
|---|---|
| **Loosely coupled** | Product, Order, User services are independent |
| **Reduced burden** | Each service handles only its own load |
| **Easy maintenance** | Fix product-service without touching order-service |
| **No single point of failure** | If notification-service is down, orders still work |
| **Technology independent** | Can use different DBs per service |
| **Quick deliveries** | Deploy one service without affecting others |

---

## 2. ShopEase Architecture Overview

```
                    ┌─────────────────────────┐
                    │   🌐 Angular Frontend    │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │   🚪 API Gateway (8080)  │
                    │   • JWT Filter            │
                    │   • Route to services     │
                    └──┬─────────┬──────────┬──┘
                       │         │          │
          ┌────────────▼┐   ┌────▼─────┐   ┌▼───────────┐
          │📦 Product   │   │🛒 Order  │   │👤 User     │
          │Service 8081 │   │Svc  8082 │   │Service 8083│
          │• CRUD       │   │• Create  │   │• Register  │
          │• Redis cache│   │• Kafka   │   │• Login     │
          └─────────────┘   │  publish │   │• JWT token │
                            └────┬─────┘   └────────────┘
                                 │ Kafka Event
                            ┌────▼──────────┐
                            │📧 Notification│
                            │Service  8084  │
                            │• Kafka consume│
                            │• Send email   │
                            └───────────────┘

          ┌─────────────────────────────────────────┐
          │ 🔍 Service Registry (Eureka) — 8761     │
          │ ⚙️ Config Server — 8888                  │
          └─────────────────────────────────────────┘
```

---

## 3. Service Registry (Eureka)

> Maintains all APIs' information (name, status, URL, health) at one place.

### service-registry `pom.xml` dependency

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

### ServiceRegistryApplication.java

```java
package com.shopease.registry;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer    // ← Enables Eureka Server
public class ServiceRegistryApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServiceRegistryApplication.class, args);
    }
}
```

### application.yml

```yaml
server:
  port: 8761

eureka:
  client:
    register-with-eureka: false    # Don't register itself
    fetch-registry: false
```

> **Access Dashboard:** `http://localhost:8761/`

### Register a Client (e.g., product-service)

Add `eureka-client` dependency and annotate:

```java
@SpringBootApplication
@EnableDiscoveryClient    // ← Registers with Eureka
public class ProductServiceApplication { ... }
```

```yaml
# product-service application.yml
spring:
  application:
    name: PRODUCT-SERVICE    # ← Name shown in Eureka dashboard

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka
```

---

## 4. Admin Server

> Provides a beautiful UI to monitor and manage all APIs' actuator endpoints.

```java
@SpringBootApplication
@EnableAdminServer
public class AdminServerApplication { ... }
```

> **What you can monitor:** Beans, loggers, heap dump, thread dump, metrics, mappings.

---

## 5. Zipkin — Distributed Tracing

> Tracks a request across multiple microservices using a unique trace ID.

```bash
# Download & run Zipkin
java -jar zipkin-server.jar
# Dashboard: http://localhost:9411/
```

Add to each service's `pom.xml`:
```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

---

## 6. FeignClient — Interservice Communication

> When one microservice needs to call another within the same application.

### ShopEase: Order Service calls Product Service

```java
// In order-service — FeignClient interface
@FeignClient(name = "PRODUCT-SERVICE")
public interface ProductServiceClient {

    @GetMapping("/api/products/{id}")
    Product getProductById(@PathVariable("id") Long productId);
}
```

```java
// OrderService.java — Using FeignClient
@Service
public class OrderService {

    @Autowired
    private ProductServiceClient productClient;

    @Autowired
    private OrderRepository orderRepo;

    public Order createOrder(OrderRequest request) {
        // ✅ Call product-service via Feign
        Product product = productClient.getProductById(request.getProductId());

        Order order = new Order();
        order.setProductName(product.getName());
        order.setPrice(product.getPrice());
        order.setQuantity(request.getQuantity());
        order.setTotalAmount(product.getPrice() * request.getQuantity());
        order.setEmail(request.getEmail());

        return orderRepo.save(order);
    }
}
```

```java
// Enable Feign in boot start class
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients    // ← Enable FeignClient
public class OrderServiceApplication { ... }
```

---

## 7. Load Balancing

> Distribute requests across multiple instances of a service.

### Run multiple instances of product-service:

```bash
# Instance 1
java -jar product-service.jar -Dserver.port=8081

# Instance 2
java -jar product-service.jar -Dserver.port=8085
```

> FeignClient + Eureka automatically provides **client-side load balancing** (round-robin) across all registered instances.

---

## 8. API Gateway

> Single entry point for all backend APIs. Handles routing and filters.

### api-gateway `application.yml`

```yaml
server:
  port: 8080

spring:
  application:
    name: API-GATEWAY
  cloud:
    gateway:
      routes:
        - id: product-service
          uri: lb://PRODUCT-SERVICE       # lb = load balanced via Eureka
          predicates:
            - Path=/api/products/**
        - id: order-service
          uri: lb://ORDER-SERVICE
          predicates:
            - Path=/api/orders/**
        - id: user-service
          uri: lb://USER-SERVICE
          predicates:
            - Path=/api/users/**
```

### JWT Authentication Filter

```java
package com.shopease.gateway.filter;

import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.http.HttpHeaders;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

@Component
public class JwtAuthFilter implements GlobalFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        String path = request.getPath().toString();

        // ✅ Skip auth for login and register
        if (path.contains("/api/users/login") || path.contains("/api/users/register")) {
            return chain.filter(exchange);
        }

        // ✅ Check for Authorization header
        HttpHeaders headers = request.getHeaders();
        if (!headers.containsKey(HttpHeaders.AUTHORIZATION)) {
            throw new RuntimeException("Missing Authorization header");
        }

        String authHeader = headers.getFirst(HttpHeaders.AUTHORIZATION);
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            throw new RuntimeException("Invalid Authorization header");
        }

        // ✅ Validate JWT token
        String token = authHeader.substring(7);
        // ... validate token logic (see Spring Security module) ...

        return chain.filter(exchange);
    }
}
```

### Access via Gateway

```
http://localhost:8080/api/products      → routed to PRODUCT-SERVICE
http://localhost:8080/api/orders        → routed to ORDER-SERVICE
http://localhost:8080/api/users/login   → routed to USER-SERVICE
```

---

## 9. Config Server

> Externalizes application configuration — change properties without redeploying.

### Problem Config Server Solves

```
Without Config Server:
  application.yml is INSIDE the JAR
  → Change a DB password? → Rebuild + Redeploy! ❌

With Config Server:
  application.yml lives in Git repo
  → Change a DB password? → Just update Git, refresh! ✅
```

### Config Server Application

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

### Config Server `application.yml`

```yaml
server:
  port: 8888

spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/shopease/config-repo
          clone-on-start: true
```

### Config Client (product-service)

```yaml
# product-service application.yml
spring:
  config:
    import: optional:configserver:http://localhost:8888
  application:
    name: product-service    # ← Config server looks for product-service.yml in Git
```

### Use `@RefreshScope` for Dynamic Config

```java
@RestController
@RefreshScope
public class ProductController {

    @Value("${product.discount.percentage}")
    private int discountPercentage;    // ← Changes without restart!
}
```

---

## 10. Circuit Breaker (Resilience4J)

> If a service call fails, execute a **fallback** instead of crashing.

### Three States

```
CLOSED ──(failures exceed threshold)──▶ OPEN ──(wait duration)──▶ HALF_OPEN
   ▲                                                                    │
   └──────────────── (success) ◀────────────────────────────────────────┘
```

| State | Behavior |
|---|---|
| **CLOSED** | Normal — all requests go through |
| **OPEN** | Failure threshold exceeded — all requests go to fallback |
| **HALF_OPEN** | After wait period — allows few test requests |

### ShopEase: Order Service with Circuit Breaker

```java
@RestController
public class OrderController {

    @Autowired
    private ProductServiceClient productClient;

    @GetMapping("/api/orders/product/{id}")
    @CircuitBreaker(fallbackMethod = "getProductFallback", name = "productService")
    public Product getProductDetails(@PathVariable Long id) {
        // ✅ Main logic — call product service
        return productClient.getProductById(id);
    }

    // ✅ Fallback — if product service is down
    public Product getProductFallback(Long id, Throwable t) {
        Product fallback = new Product();
        fallback.setName("Product Service Unavailable");
        fallback.setPrice(0.0);
        return fallback;
    }
}
```

### Circuit Breaker Configuration

```yaml
resilience4j.circuitbreaker:
  configs:
    default:
      registerHealthIndicator: true
      slidingWindowSize: 10
      minimumNumberOfCalls: 5
      permittedNumberOfCallsInHalfOpenState: 3
      automaticTransitionFromOpenToHalfOpenEnabled: true
      waitDurationInOpenState: 30s
      failureRateThreshold: 50

management:
  endpoints.web.exposure.include: '*'
  endpoint.health.show-details: always
  health.circuitbreakers.enabled: true
```

---

*← [07 — SonarQube](./07-sonarqube-code-quality.md) | [09 — Spring Security →](./09-spring-security.md)*
