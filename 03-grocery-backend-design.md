# 🛒 Grocery Backend Service — System Design & Implementation Plan

## 1. Understanding the Problem (Before I Design Anything)
Before choosing any tech or pattern, I asked myself these questions — because the "best" design depends entirely on the answers:

### Functional Questions:
* **What's the scale?** Is this a neighborhood grocer app or a Blinkit/Zepto/BigBasket competitor?
* **Is it B2C** (customers buying) or **B2B** (stores managing inventory) or both?
* **Are we handling real-time inventory** (stock decrements on order) or catalog-only?
* **Multi-store / multi-location** or single warehouse?
* **Do we need search** (by name, category, brand, price range)?
* **Delivery/logistics involved** or just product management?

### Non-Functional Questions:
* **Expected QPS?** 100 RPS (small) vs 10K RPS (Zepto-scale) drastically changes design.
* **Read:Write ratio?** Grocery is typically 90:10 read-heavy (browse >> order).
* **Consistency needs?** Inventory MUST be strongly consistent (can't oversell eggs). Product catalog can be eventually consistent.
* **Latency SLO?** P99 < 200ms for browse, < 500ms for checkout?

> **My Assumption** (since you said "highly concurrent and scalable"): I'm designing for a Zepto/Blinkit-scale quick-commerce platform — millions of SKUs, thousands of concurrent users, real-time inventory, multi-store (dark stores), read-heavy with bursty write load during flash sales.

If my assumption is wrong, tell me and I'll recalibrate.

## 2. The Core Design Philosophy (My Thought Process)
> *"Premature optimization is the root of all evil, but building a monolith when you need Netflix-scale is also evil."* — My pragmatic take

I follow 3 principles when designing:

1. **Start simple, design for evolution** — Don't build 15 microservices on day 1. Build a modular monolith that can be strangled into microservices when bottlenecks appear.
2. **Optimize for the critical path** — Identify the hot path (product browse + add-to-cart + checkout). Every millisecond here matters. Non-critical paths (admin, reports) can be slow.
3. **Data is king** — The hardest thing to change later is your data model. Spend 40% of design time here.

## 3. High-Level Architecture
Here's the architecture I'm proposing:

```text
┌─────────────────┐
│   CloudFront    │  ← CDN for images/static
│      (CDN)      │
└────────┬────────┘
         │
┌────────▼────────┐
│   API Gateway   │  ← Kong / AWS API Gateway
│  (Rate Limit,   │     - Auth validation
│   Auth, CORS)   │     - Request routing
└────────┬────────┘
         │
         ├───────────────────────────────┐
         │                               │
┌────────▼───────┐               ┌───────▼───────┐
│ Load Balancer  │               │   ...         │
│    (ALB)       │               │               │
└────────┬───────┘               └───────────────┘
         │
┌────────┴────────┬───────────────────┐
│                 │                   │
┌────▼─────┐ ┌────▼─────┐       ┌─────▼────┐
│ App Node │ │ App Node │  ...  │ App Node │  ← Stateless
│    1     │ │    2     │       │    N     │     (K8s pods)
└────┬─────┘ └────┬─────┘       └─────┬────┘
     │            │                   │
     └────────────┴─────────┬─────────┘
                            │
      ┌─────────────┬───────┴─────┬──────────────┬─────────────┐
      │             │             │              │             │
┌─────▼────┐  ┌─────▼────┐  ┌─────▼────┐   ┌─────▼────┐  ┌─────▼────┐
│  Redis   │  │PostgreSQL│  │ MongoDB  │   │  Kafka   │  │ Elastic- │
│ (Cache,  │  │ (Orders, │  │ (Product │   │ (Events, │  │  search  │
│ Session, │  │  Users,  │  │ Catalog) │   │  Async)  │  │ (Search) │
│ RateLim) │  │Inventory)│  │          │   │          │  │          │
└──────────┘  └──────────┘  └──────────┘   └─────┬────┘  └──────────┘
                                                 │
                                     ┌───────────┴───────────┐
                                     │           │           │
                               ┌─────▼────┐ ┌────▼────┐ ┌────▼────┐
                               │Notifica- │ │Analytics│ │Inventory│
                               │tion Svc  │ │   Svc   │ │  Sync   │
                               │(Email/   │ │ (Read   │ │ Worker  │
                               │  SMS)    │ │ Models) │ │         │
                               └──────────┘ └─────────┘ └─────────┘
┌─────────────┐         ┌──────────────┐
│     S3      │         │  Prometheus  │
│(Product Img,│         │  + Grafana   │
│ Invoices)   │         │ (Monitoring) │
└─────────────┘         └──────────────┘
```

## 4. Topic-by-Topic Design Decisions (My Reasoning)

### 4.1 API Design & HTTP Fundamentals
**What I'll build:**
* **RESTful API** following resource-oriented design (`/api/v1/products`, `/api/v1/cart`, `/api/v1/orders`).
* **URI versioning** (`/v1/`) over header versioning — simpler for clients, easier to debug in logs.
* **Idempotency keys** for `POST /orders` (critical for payments — prevents double-charge on retry).
* **Cursor-based pagination** for product listings (not offset — breaks at scale when page 50000).
* **HATEOAS?** NO. Over-engineered for our use case. Clients know their URLs.

**Trade-offs I considered:**

| Option | Pros | Cons | My Choice |
| :--- | :--- | :--- | :--- |
| **REST** | Simple, cached by CDN, tooling mature | Over-fetching, multiple round trips | ✅ Chosen |
| **GraphQL** | Single endpoint, client picks fields | Caching hard, N+1 risk, complexity | Later for mobile app |
| **gRPC** | Fast, typed, streaming | Browser unfriendly, harder debugging | Internal service-to-service only |

**Standard response envelope:**
```json
{
    "success": true,
    "data": {
        // ...
    },
    "meta": {
        "page": 1,
        "nextCursor": "eyJpZCI6MTIzfQ=="
    },
    "errors": null,
    "timestamp": "2026-04-27T10:00:00Z",
    "traceId": "abc-123"
}
```

### 4.2 Authentication & Authorization
**My Choice: JWT (access) + Refresh Token (rotating) + RBAC**

* **Access Token:** JWT, 15-min expiry, stateless, contains `userId`, `roles`, `storeId`.
* **Refresh Token:** Opaque random string, 30-day expiry, stored in Redis with rotation. Revocable.
* **Password hashing:** bcrypt with cost factor 12 (argon2id would be better but bcrypt is battle-tested and sufficient).
* **RBAC roles:** `CUSTOMER`, `DELIVERY_PARTNER`, `STORE_MANAGER`, `ADMIN`, `SUPER_ADMIN`.
* **OAuth2:** Google/Facebook login via `/auth/oauth/callback` using Authorization Code flow with PKCE.

> **Why NOT sessions?** Sessions require sticky load balancing OR a shared session store. JWT scales horizontally with zero coordination. Refresh tokens give us revocation power back.

*Trade-off Acknowledged:* JWTs can't be instantly revoked. Mitigation: short access token TTL + refresh token rotation + a Redis blocklist for compromised tokens.

### 4.3 Database Design (The Most Critical Section)
**My Choice: Polyglot Persistence**

| Data Type | Store | Why |
| :--- | :--- | :--- |
| **Users, Orders, Payments, Inventory** | **PostgreSQL** | ACID for money & stock, joins, relational |
| **Product Catalog** (SKUs, descriptions, variants) | **MongoDB** | Flexible schema — every product category has different attributes (milk has fat%, rice has weight, electronics has warranty) |
| **Product Search** | **Elasticsearch** | Full-text search, faceted filters, typo tolerance |
| **Cart, Sessions, Rate Limits, Inventory Cache** | **Redis** | Sub-ms latency, TTL built-in |
| **Product Images, Invoices** | **S3** | Cheap, durable, CDN-friendly |
| **Events** (order placed, stock updated) | **Kafka** | Durable event log, async processing |

> **Why NOT single database?** One relational DB for everything would work at 1K QPS, but:
> * Product catalog queries (flexible filters) kill relational queries.
> * Full-text search on Postgres `ILIKE` is slow beyond 100K products.
> * Cart in Postgres means write amplification on every "add to cart".

**Schema Design (PostgreSQL — key tables):**
```sql
-- Normalized for write integrity, denormalized views for reads
CREATE TABLE users (id UUID, email VARCHAR, phone VARCHAR, password_hash VARCHAR, created_at TIMESTAMP, ...);
CREATE TABLE addresses (id UUID, user_id UUID, line1 VARCHAR, city VARCHAR, pincode VARCHAR, lat DECIMAL, lng DECIMAL, is_default BOOLEAN);
CREATE TABLE stores (id UUID, name VARCHAR, lat DECIMAL, lng DECIMAL, pincode_serving VARCHAR[], is_active BOOLEAN);
CREATE TABLE inventory (store_id UUID, sku_id UUID, quantity INT, reserved_quantity INT, version INT); -- optimistic locking
CREATE TABLE orders (id UUID, user_id UUID, store_id UUID, status VARCHAR, total DECIMAL, idempotency_key VARCHAR UNIQUE, created_at TIMESTAMP);
CREATE TABLE order_items (order_id UUID, sku_id UUID, quantity INT, price_at_purchase DECIMAL);
CREATE TABLE payments (id UUID, order_id UUID, gateway_txn_id VARCHAR, status VARCHAR, amount DECIMAL);
```

**Indexing strategy:**
* Composite index on `orders(user_id, created_at DESC)` for order history
* Unique index on `orders(idempotency_key)` for duplicate prevention
* Composite index on `inventory(store_id, sku_id)` — covers our hottest query
* Partial index on `orders(status) WHERE status IN ('PENDING', 'PROCESSING')` — small, fast for dashboards

**Avoiding N+1:**
* Use `JOIN FETCH` / batch loading (DataLoader pattern)
* For order list with items: single query with `LEFT JOIN order_items` + application-side grouping

**Transactions & Isolation:**
* `READ_COMMITTED` as default (PostgreSQL default)
* `SERIALIZABLE` for checkout (prevent phantom reads on inventory)
* `SELECT ... FOR UPDATE` for stock deduction — pessimistic lock on hot inventory rows

### 4.4 Caching Strategy (Critical for Scale)
**Multi-layer cache:**
Browser Cache (1 min) → CDN (1 hour, product images) → Redis (5 min, product details) → DB

**What to cache:**

| Data | Strategy | TTL | Invalidation |
| :--- | :--- | :--- | :--- |
| **Product details** | Cache-aside | 5 min | On product update event |
| **Product listings** (by category) | Cache-aside | 2 min | TTL-based |
| **User session** | Write-through | 15 min (access token TTL) | On logout |
| **Inventory count** | Write-through | N/A (source of truth in Redis, synced to DB) | Real-time |
| **Cart** | Write-through | 7 days | User clears |
| **Rate limits** | Redis counter | 1 min window | TTL-based |

**Cache Invalidation — The Hard Problem:**
I'll use event-driven invalidation via Kafka:
* When product is updated → publish `product.updated` event → cache invalidator consumer deletes the key.
* For inventory, I use write-through: Redis is the source of truth for reads, DB for durability (synced every 5 sec via CDC).

**Cache Stampede Prevention:**
* Probabilistic early expiration (re-cache before TTL hits)
* Distributed lock (Redlock) on cache miss — only one pod re-fetches from DB
* Stale-while-revalidate — serve stale data, async refresh

### 4.5 System Design — Scale, LB, Rate Limiting
**Scaling approach:**
* Horizontal scaling for app servers (stateless pods behind ALB)
* Vertical scaling for Postgres primary, with read replicas for analytics queries
* Sharding (future): shard orders by `user_id % N` when single DB hits limits

**Load Balancing:**
* L7 ALB with round-robin for stateless services
* Consistent hashing on the inventory service (route same SKU to same pod for in-memory lock affinity)

**Rate Limiting:**
* Token bucket algorithm in Redis (using Lua script for atomicity)
* Tiered limits: Anonymous (20 req/min), Authenticated (100 req/min), Premium (500 req/min)
* Per-endpoint limits: `/checkout` (5 req/min), `/search` (60 req/min)

**CAP Theorem position:**
* Product catalog: **AP** (availability over consistency — slightly stale products OK)
* Inventory & Orders: **CP** (consistency over availability — NEVER oversell)
*This is why we split stores: each optimizes for its CAP trade-off.*

### 4.6 Runtime — Language & Concurrency Model
**My Choice: Java 21 with Spring Boot 3** (since your background is Java/Spring from prior docs)

**Why Java over Node.js:**
* Your team skillset (from our prior conversation)
* Virtual Threads (Project Loom) give us Node.js-like concurrency without callback hell
* Better tooling for payments/financial domain (BigDecimal, JPA, mature Kafka clients)
* JVM's JIT outperforms Node for CPU-bound tasks (pricing calculations, cart total)

**Virtual Threads usage:**
```java
// Handle 10K concurrent requests per pod with ~100MB heap
@Bean
TomcatProtocolHandlerCustomizer<?> protocolHandlerVirtualThreadExecutorCustomizer() {
    return protocolHandler -> protocolHandler.setExecutor(
        Executors.newVirtualThreadPerTaskExecutor()
    );
}
```

**Blocking vs Non-blocking:**
* Virtual threads make blocking I/O cheap — write synchronous-looking code, JVM handles scheduling
* For truly async flows (send email, update analytics) → push to Kafka, don't block the request

### 4.7 Error Handling & Logging
**Global exception handler using `@ControllerAdvice`:**
* Map domain exceptions to HTTP status codes
* Never leak stack traces to clients
* Log with correlation ID (every request gets `X-Trace-Id` header, propagated via MDC)

**Structured logging (JSON):**
```json
{
    "timestamp": "2026-04-27T10:00:00Z",
    "level": "ERROR",
    "traceId": "abc",
    "userId": "u123",
    "msg": "Stock check failed",
    "skuId": "s456"
}
```

**Log levels:**
* **ERROR** — unexpected failures requiring investigation
* **WARN** — handled failures (retries, fallbacks triggered)
* **INFO** — business events (order placed, payment success)
* **DEBUG** — off in prod, toggled per-request via header for debugging

**Monitoring stack:**
* **Metrics:** Micrometer → Prometheus → Grafana
* **Tracing:** OpenTelemetry → Jaeger/Dynatrace
* **Logs:** Loki or ELK stack
* **Alerts:** Prometheus Alertmanager → PagerDuty/Slack

### 4.8 Security Fundamentals
**Non-negotiables:**
* HTTPS everywhere (TLS 1.3, HSTS header)
* Passwords: bcrypt cost 12, never logged
* SQL Injection: Prepared statements only (JPA/Hibernate handles this, but audit raw queries)
* XSS: Output encoding, CSP header (`default-src 'self'`)
* CSRF: Not an issue for stateless JWT APIs with Authorization header, but enforce `SameSite=Strict` on refresh token cookie
* Secrets: AWS Secrets Manager / HashiCorp Vault — NEVER in code or env files
* CORS: Whitelist specific origins (no `*` in production)
* Brute-force: Rate limit login (5 attempts/15 min per IP + per email)
* PII: Encrypt at rest (column-level for phone/address), TLS in transit
* Audit log: Immutable log of admin actions (who changed price, deleted product)

### 4.9 Architecture Pattern — The Big Decision
**My Choice: Modular Monolith → Evolve to Microservices**

Here's my honest reasoning:

| Approach | When to Use | My Assessment |
| :--- | :--- | :--- |
| **Monolith** | < 20 devs, < 10K RPS, fast iteration | ❌ Won't scale to Zepto-level |
| **Modular Monolith** | Starting out, < 50 devs | ✅ Phase 1 |
| **Microservices** | > 50 devs, independent scaling needed | ✅ Phase 2 (after 6-12 months) |

**Phase 1 Modules (all in one deployable, but with strict boundaries):**
```text
com.grocery/
├── api/                    ← Controllers (thin layer)
├── modules/
│   ├── catalog/            ← Products, categories, search
│   │   ├── api/            ← Internal API (other modules call this)
│   │   ├── domain/         ← Entities, value objects
│   │   ├── application/    ← Use cases, services
│   │   └── infrastructure/ ← Repositories, adapters
│   ├── inventory/          ← Stock management
│   ├── cart/               ← Shopping cart
│   ├── order/              ← Order lifecycle (Saga)
│   ├── payment/            ← Payment gateway integration
│   ├── user/               ← Auth, profile
│   ├── notification/       ← Email, SMS, push
│   └── delivery/           ← Logistics integration
├── shared/                 ← Cross-cutting (security, logging, exceptions)
└── config/
```

**Rules I enforce:**
* Modules talk to each other only via their `api/` package (no reaching into internals)
* Each module owns its database tables (no FK across modules — use IDs)
* Events over direct calls when possible (publish `OrderPlacedEvent`, inventory listens)

**Design Patterns I'll use:**
* **Hexagonal Architecture (Ports & Adapters)** — domain logic doesn't know about DB or HTTP
* **Repository Pattern** — abstract data access
* **Strategy Pattern** — payment gateway selection (Razorpay/Stripe/PayU)
* **Saga Pattern** — distributed checkout (reserve stock → charge → confirm)
* **Outbox Pattern** — reliable event publishing (write event to DB in same txn, publish async)
* **Circuit Breaker (Resilience4j)** — when payment gateway is down, fail fast
* **CQRS (light)** — separate read models for product listings (from Elasticsearch) vs write model (MongoDB)

### 4.10 Message Queues & Async Processing
**My Choice: Kafka over RabbitMQ**

**Why Kafka:**
* Event log is replayable (critical for rebuilding analytics / fixing bugs)
* Higher throughput (millions/sec vs RabbitMQ's hundreds of thousands)
* Natural fit for event-driven architecture
* Partitioning gives us ordering guarantees per user/order

**Topics:**
* `order.events` (placed, confirmed, shipped, delivered, cancelled)
* `inventory.events` (stock_updated, low_stock_alert)
* `user.events` (signup, login — for analytics)
* `notification.requests` (send email/SMS)
* `dead-letter-queue` (failed processing)

**Patterns:**
* Outbox pattern for reliable publishing
* Idempotent consumers (store processed event IDs in Redis)
* DLQ after 3 retries with exponential backoff

**Background jobs:**
* Order confirmation emails
* Inventory restock alerts
* Abandoned cart notifications (after 2 hours)
* Nightly reports

### 4.11 File Handling & Storage
**Product images:**
* Direct-to-S3 upload via presigned URLs (client uploads directly, backend never sees the file)
* Images resized by Lambda on upload (thumbnail 200x200, medium 600x600, original)
* Served via CloudFront CDN

**Large files (invoices, reports):**
* Stream, don't load into memory
* Generated asynchronously, link delivered via email

### 4.12 Testing & Deployment
**Testing pyramid:**
* **70% Unit tests** — JUnit 5, Mockito, pure domain logic
* **20% Integration tests** — Testcontainers (real Postgres, Redis, Kafka in Docker)
* **10% E2E tests** — Postman/Newman runs on CI, hits a staging env

**CI/CD pipeline:**
`Git push` → `GitHub Actions` → `Build` → `Unit Test` → `Integration Test` → `SonarQube` → `Build Docker image` → `Push to ECR` → `Deploy to staging` → `Smoke test` → `Manual approval` → `Canary deploy to 10% prod` → `Monitor` → `Full rollout`

**Docker & K8s:**
* Multi-stage Dockerfile (build stage + slim runtime, final image ~150MB)
* Health checks: `/actuator/health/liveness` and `/actuator/health/readiness`
* HPA (Horizontal Pod Autoscaler) on CPU + custom metric (requests/sec)
* Rolling deployment with `maxSurge=25%`, `maxUnavailable=0`

## 5. Critical Flow Walkthrough — "Place Order" (The Hot Path)
This shows how it all comes together:

1. **[Client]** `POST /v1/orders`
   ```json
   { "items": [...], "idempotencyKey": "uuid" }
   ```
   *Headers: Authorization: Bearer <jwt>*

2. **[API Gateway]**
   * Validates JWT signature
   * Rate limit check (Redis: 5 orders/min per user)
   * Forwards to app

3. **[OrderController]**
   * Check idempotency key in Redis → if exists, return cached response
   * Delegate to `OrderService`

4. **[OrderService]** — Saga orchestration begins
   * **a. `ReserveInventoryCommand` → `InventoryService`**
     * `SELECT ... FOR UPDATE` on inventory rows
     * Check quantity, decrement, commit
     * Publish `InventoryReserved` event
   * **b. `CreatePaymentCommand` → `PaymentService`**
     * Call Razorpay API (with circuit breaker, 3s timeout)
     * On success: store payment record
     * On failure: compensating action — release inventory
   * **c. `CreateOrderRecord` → `OrderRepository`**
     * `INSERT` order (`status = CONFIRMED`)
     * `INSERT` order_items
     * `INSERT` into outbox table (`OrderPlacedEvent`)
     * Commit transaction

5. **[Outbox Publisher]** (async)
   * Polls outbox → publishes `OrderPlacedEvent` to Kafka

6. **[Async consumers]**
   * `NotificationService` → sends SMS/email
   * `AnalyticsService` → updates dashboards
   * `DeliveryService` → assigns to dark store

7. **[Response to client]**
   ```json
   {
     "success": true,
     "data": { "orderId": "ord_123", "status": "CONFIRMED" }
   }
   ```
   * Cache response against idempotency key (TTL 24h)

**Failure scenarios handled:**
* **Inventory insufficient** → 409 Conflict, no payment attempted
* **Payment fails** → inventory released via compensating transaction
* **App crashes mid-flow** → outbox ensures event will eventually publish
* **Client retries** → idempotency key prevents duplicate order

## 6. Implementation Roadmap (Phased)

| Phase | Duration | Deliverables |
| :--- | :--- | :--- |
| **Phase 0 — Foundation** | 1 week | Project skeleton, Docker setup, CI/CD pipeline, base modules |
| **Phase 1 — Auth & User** | 1 week | JWT auth, RBAC, user profile, OAuth |
| **Phase 2 — Catalog** | 1.5 weeks | Product CRUD, categories, MongoDB, Elasticsearch sync |
| **Phase 3 — Inventory** | 1 week | Stock management, pessimistic locking, Redis cache |
| **Phase 4 — Cart & Checkout** | 1.5 weeks | Cart in Redis, Saga orchestration, idempotency |
| **Phase 5 — Payment** | 1 week | Razorpay integration, circuit breaker, webhook handling |
| **Phase 6 — Async & Events** | 1 week | Kafka setup, Outbox pattern, notification service |
| **Phase 7 — Observability** | 3 days | Prometheus, Grafana, distributed tracing, runbooks |
| **Phase 8 — Hardening** | 1 week | Load testing (Gatling), security audit, chaos tests |

**Total:** ~10 weeks to production-ready MVP

## 7. Key Trade-offs I'm Making (Be Honest About Them)

| Decision | Trade-off | Why I'm OK with it |
| :--- | :--- | :--- |
| **Modular monolith over microservices** | Slower to scale org-wise | Faster delivery, easier to evolve |
| **Polyglot persistence** | More ops complexity | Right tool for right job, not optional at scale |
| **JWT over sessions** | Can't instantly revoke | Short TTL + blocklist mitigates |
| **Pessimistic locking on inventory** | Lower concurrency | Correctness > throughput for stock |
| **Kafka over RabbitMQ** | Steeper learning curve | Future-proof for analytics |
| **Java/Spring over Node.js** | Higher memory footprint | Virtual threads + team fit |

## 8. What I Need From You Before Coding
Please confirm / answer:

1. **Scale assumption correct?** Zepto-scale (10K+ RPS, millions of SKUs, multi-store)? Or smaller?
2. **Tech stack:** Java 21 + Spring Boot 3 OK? Or you prefer Node.js/Python?
3. **Database:** OK with polyglot (Postgres + Mongo + Redis + Elasticsearch), or prefer single DB for v1?
4. **Cloud:** AWS, GCP, Azure, or on-prem?
5. **Payment gateway:** Razorpay, Stripe, or mock for now?
6. **Scope for v1:** Everything above, or MVP (catalog + cart + checkout only) first?
7. **Team size & timeline:** Solo/team? Aggressive or measured delivery?

Once you confirm, I'll:
* Create the full project skeleton with all modules
* Write production-grade code for each module (following the patterns above)
* Include tests, Dockerfile, K8s manifests, CI/CD config
* Document every API in Postman collection with examples

Ready when you give the green light. Let me know any pushback on the design too — I'd rather debate it now than refactor later.
