# 06 — System Design (Idempotency, Saga, Outbox, CAP, Rate Limiting, Circuit Breaker, CQRS)

> **Priority**: 🔴 Critical  
> **Estimated Study Time**: 2 days  
> Markers: 🔥 = Frequently Asked | 💎 = Differentiator

---

## Section 1: Idempotency

### 🔥 Q1. Design an idempotent payment API. How do you prevent duplicate charges?

**Answer:**

**Idempotency** means making the same request multiple times produces the same result as making it once. Critical for payments — network retries must not cause double charges.

**Design:**

```
Client                          Payment Service                    Database
  │                                   │                               │
  │  POST /payments                   │                               │
  │  Idempotency-Key: "abc-123"       │                               │
  │──────────────────────────────────→│                               │
  │                                   │  SELECT * FROM idempotency    │
  │                                   │  WHERE key = 'abc-123'        │
  │                                   │──────────────────────────────→│
  │                                   │  Not found                    │
  │                                   │←──────────────────────────────│
  │                                   │                               │
  │                                   │  INSERT INTO idempotency      │
  │                                   │  (key, status='PROCESSING')   │
  │                                   │──────────────────────────────→│
  │                                   │                               │
  │                                   │  Process payment...           │
  │                                   │                               │
  │                                   │  UPDATE idempotency           │
  │                                   │  SET status='COMPLETED',      │
  │                                   │  response='{...}'             │
  │                                   │──────────────────────────────→│
  │                                   │                               │
  │  200 OK {txnId: "TXN-001"}       │                               │
  │←──────────────────────────────────│                               │
  │                                   │                               │
  │  POST /payments (RETRY)           │                               │
  │  Idempotency-Key: "abc-123"       │                               │
  │──────────────────────────────────→│                               │
  │                                   │  SELECT * FROM idempotency    │
  │                                   │  WHERE key = 'abc-123'        │
  │                                   │──────────────────────────────→│
  │                                   │  Found! status=COMPLETED      │
  │                                   │←──────────────────────────────│
  │                                   │                               │
  │  200 OK {txnId: "TXN-001"}       │  Return cached response       │
  │←──────────────────────────────────│  (no reprocessing!)           │
```

```java
@RestController
@RequestMapping("/api/v1/payments")
public class PaymentController {
    
    @PostMapping
    public ResponseEntity<PaymentResponse> createPayment(
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            @Valid @RequestBody CreatePaymentRequest request) {
        
        return paymentService.processIdempotent(idempotencyKey, request);
    }
}

@Service
@Slf4j
public class PaymentService {
    
    @Transactional
    public ResponseEntity<PaymentResponse> processIdempotent(String idempotencyKey, CreatePaymentRequest request) {
        // 1. Check if already processed
        Optional<IdempotencyRecord> existing = idempotencyRepository.findByKey(idempotencyKey);
        
        if (existing.isPresent()) {
            IdempotencyRecord record = existing.get();
            
            if (record.getStatus() == IdempotencyStatus.PROCESSING) {
                // Still processing — return 409 Conflict
                return ResponseEntity.status(HttpStatus.CONFLICT)
                    .body(new PaymentResponse("PROCESSING", "Payment is being processed"));
            }
            
            // Already completed — return cached response
            return ResponseEntity.ok(objectMapper.readValue(record.getResponse(), PaymentResponse.class));
        }
        
        // 2. Create idempotency record (lock)
        IdempotencyRecord record = IdempotencyRecord.builder()
            .key(idempotencyKey)
            .status(IdempotencyStatus.PROCESSING)
            .requestHash(hashRequest(request))
            .createdAt(Instant.now())
            .expiresAt(Instant.now().plus(24, ChronoUnit.HOURS))
            .build();
        
        try {
            idempotencyRepository.save(record); // Unique constraint on key
        } catch (DataIntegrityViolationException e) {
            // Race condition — another thread got there first
            return processIdempotent(idempotencyKey, request); // Retry
        }
        
        // 3. Process payment
        try {
            PaymentResponse response = processPayment(request);
            
            // 4. Store response
            record.setStatus(IdempotencyStatus.COMPLETED);
            record.setResponse(objectMapper.writeValueAsString(response));
            idempotencyRepository.save(record);
            
            return ResponseEntity.status(HttpStatus.CREATED).body(response);
            
        } catch (Exception e) {
            record.setStatus(IdempotencyStatus.FAILED);
            record.setErrorMessage(e.getMessage());
            idempotencyRepository.save(record);
            throw e;
        }
    }
}

@Entity
@Table(name = "idempotency_records")
public class IdempotencyRecord {
    @Id @GeneratedValue
    private Long id;
    
    @Column(unique = true, nullable = false, length = 64)
    private String key;
    
    @Enumerated(EnumType.STRING)
    private IdempotencyStatus status;
    
    private String requestHash;
    
    @Column(columnDefinition = "CLOB")
    private String response;
    
    private String errorMessage;
    private Instant createdAt;
    private Instant expiresAt;
}
```

**Key Design Decisions:**
- Idempotency key is **client-generated** (UUID) — sent in header
- Keys expire after 24 hours (configurable)
- Request hash validates that retry has same payload
- PROCESSING state handles concurrent requests for same key

---

## Section 2: Saga Pattern

### 🔥 Q2. Explain the Saga pattern with a real payments example.

**Answer:**

A Saga is a sequence of **local transactions** where each step has a **compensating action** for rollback. Used when you can't have a distributed transaction across microservices.

**Two Types:**

| Type | Coordination | Pros | Cons |
|------|-------------|------|------|
| **Choreography** | Events (no central coordinator) | Loose coupling, simple | Hard to track, complex flows |
| **Orchestration** | Central orchestrator | Easy to understand, centralized logic | Single point of failure, coupling |

**Orchestration Saga — Payment Flow:**

```
┌──────────────┐
│  Saga        │
│  Orchestrator│
└──────┬───────┘
       │
       │ 1. Create Order
       ├──────────────────→ Order Service ──→ [Order Created]
       │
       │ 2. Reserve Inventory
       ├──────────────────→ Inventory Service ──→ [Inventory Reserved]
       │
       │ 3. Process Payment
       ├──────────────────→ Payment Service ──→ [Payment Charged]
       │
       │ 4. Confirm Delivery
       ├──────────────────→ Delivery Service ──→ [Delivery Scheduled]
       │
       │ ✅ All steps succeeded → Saga Complete
       │
       │ ❌ Step 3 fails (Payment Declined)?
       │    Compensate in reverse:
       │    3c. Refund Payment (no-op, payment failed)
       │    2c. Release Inventory ←── Inventory Service
       │    1c. Cancel Order ←── Order Service
```

```java
// Saga Orchestrator
@Service
@Slf4j
public class OrderSagaOrchestrator {
    
    public enum SagaStep { CREATE_ORDER, RESERVE_INVENTORY, PROCESS_PAYMENT, SCHEDULE_DELIVERY }
    
    @Transactional
    public OrderResult executeOrderSaga(OrderRequest request) {
        SagaContext context = new SagaContext(request);
        
        try {
            // Step 1: Create Order
            context.setStep(SagaStep.CREATE_ORDER);
            Order order = orderService.createOrder(request);
            context.setOrderId(order.getId());
            
            // Step 2: Reserve Inventory
            context.setStep(SagaStep.RESERVE_INVENTORY);
            inventoryService.reserve(order.getItems());
            
            // Step 3: Process Payment
            context.setStep(SagaStep.PROCESS_PAYMENT);
            PaymentResult payment = paymentService.charge(request.getPaymentDetails());
            context.setPaymentId(payment.getTxnId());
            
            // Step 4: Schedule Delivery
            context.setStep(SagaStep.SCHEDULE_DELIVERY);
            deliveryService.schedule(order.getId(), request.getDeliveryAddress());
            
            // All steps succeeded
            orderService.confirmOrder(order.getId());
            return OrderResult.success(order.getId());
            
        } catch (Exception e) {
            log.error("Saga failed at step {}: {}", context.getStep(), e.getMessage());
            compensate(context);
            return OrderResult.failed(e.getMessage());
        }
    }
    
    private void compensate(SagaContext context) {
        log.info("Compensating saga from step: {}", context.getStep());
        
        // Compensate in reverse order
        List<SagaStep> completedSteps = getCompletedSteps(context.getStep());
        Collections.reverse(completedSteps);
        
        for (SagaStep step : completedSteps) {
            try {
                switch (step) {
                    case SCHEDULE_DELIVERY -> deliveryService.cancel(context.getOrderId());
                    case PROCESS_PAYMENT -> paymentService.refund(context.getPaymentId());
                    case RESERVE_INVENTORY -> inventoryService.release(context.getOrderId());
                    case CREATE_ORDER -> orderService.cancelOrder(context.getOrderId());
                }
            } catch (Exception e) {
                log.error("Compensation failed for step {}: {}", step, e.getMessage());
                // Store for manual intervention
                sagaRecoveryService.recordFailedCompensation(context, step, e);
            }
        }
    }
}
```

**Choreography Saga (Event-Driven):**
```
Order Service                    Inventory Service              Payment Service
     │                                │                              │
     │ OrderCreated event             │                              │
     │──────────────────────────────→│                              │
     │                                │ InventoryReserved event      │
     │                                │─────────────────────────────→│
     │                                │                              │
     │                                │                   PaymentCharged event
     │←─────────────────────────────────────────────────────────────│
     │                                │                              │
     │ OrderConfirmed                 │                              │
     │                                │                              │
     │ ❌ PaymentFailed event?        │                              │
     │                                │←─────────────────────────────│
     │                                │ Release inventory            │
     │←──────────────────────────────│                              │
     │ Cancel order                   │                              │
```

---

## Section 3: CAP Theorem

### 🔥 Q3. Explain the CAP theorem. How does it apply to microservices?

**Answer:**

**CAP Theorem**: In a distributed system, you can only guarantee **two out of three**:

| Property | Meaning |
|----------|---------|
| **Consistency (C)** | Every read receives the most recent write |
| **Availability (A)** | Every request receives a response (not error) |
| **Partition Tolerance (P)** | System works despite network partitions |

**In practice, P is mandatory** (networks fail). So the real choice is **CP vs AP**.

| Choice | Behavior During Partition | Example |
|--------|--------------------------|---------|
| **CP** | Rejects requests to maintain consistency | Payment ledger, bank balance |
| **AP** | Serves potentially stale data | Product catalog, user profiles |

**In Microservices:**
```
Payment Service (CP)              Product Service (AP)
├── Must be consistent            ├── Can serve stale data
├── Reject if unsure              ├── Always respond
├── Use strong consistency DB     ├── Use eventual consistency
├── Synchronous replication       ├── Async replication
└── Example: Account balance      └── Example: Product price
```

**Real-World: Most systems use eventual consistency with compensation:**
- Payment is processed (strong consistency within payment DB)
- Event published to Kafka (eventual consistency across services)
- If downstream fails → compensating transaction (refund)

---

## Section 4: Rate Limiting

### 🔥 Q4. How do you implement rate limiting? Explain different algorithms.

**Answer:**

| Algorithm | How It Works | Pros | Cons |
|-----------|-------------|------|------|
| **Token Bucket** | Tokens added at fixed rate, consumed per request | Allows bursts, smooth | Memory for tokens |
| **Leaky Bucket** | Requests queue, processed at fixed rate | Smooth output | No burst handling |
| **Fixed Window** | Count requests per time window | Simple | Boundary burst problem |
| **Sliding Window Log** | Track timestamp of each request | Accurate | Memory intensive |
| **Sliding Window Counter** | Weighted count across windows | Good balance | Approximate |

**Token Bucket Implementation (most common):**

```java
// Using Resilience4j Rate Limiter
@Configuration
public class RateLimitConfig {
    
    @Bean
    public RateLimiterRegistry rateLimiterRegistry() {
        RateLimiterConfig config = RateLimiterConfig.custom()
            .limitForPeriod(100)              // 100 requests
            .limitRefreshPeriod(Duration.ofSeconds(1))  // per second
            .timeoutDuration(Duration.ofMillis(500))    // wait 500ms if limit reached
            .build();
        
        return RateLimiterRegistry.of(config);
    }
}

@RestController
public class PaymentController {
    
    private final RateLimiter rateLimiter;
    
    @PostMapping("/api/v1/payments")
    public ResponseEntity<PaymentResponse> createPayment(@RequestBody CreatePaymentRequest request) {
        return RateLimiter.decorateSupplier(rateLimiter, () -> {
            return ResponseEntity.ok(paymentService.process(request));
        }).get();
    }
}

// Redis-based distributed rate limiter (for multiple instances)
@Component
public class RedisRateLimiter {
    
    private final StringRedisTemplate redis;
    
    public boolean isAllowed(String clientId, int maxRequests, Duration window) {
        String key = "rate_limit:" + clientId;
        
        // Lua script for atomic check-and-increment
        String luaScript = """
            local current = redis.call('INCR', KEYS[1])
            if current == 1 then
                redis.call('PEXPIRE', KEYS[1], ARGV[1])
            end
            return current
            """;
        
        Long count = redis.execute(
            RedisScript.of(luaScript, Long.class),
            List.of(key),
            String.valueOf(window.toMillis())
        );
        
        return count != null && count <= maxRequests;
    }
}
```

**API Gateway Rate Limiting (per merchant):**
```yaml
# Spring Cloud Gateway rate limiting
spring:
  cloud:
    gateway:
      routes:
        - id: payment-service
          uri: lb://payment-service
          predicates:
            - Path=/api/v1/payments/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100  # 100 req/sec steady
                redis-rate-limiter.burstCapacity: 200   # 200 req/sec burst
                key-resolver: "#{@merchantKeyResolver}"
```

---

## Section 5: Circuit Breaker

### 🔥 Q5. How does a circuit breaker work? Explain Resilience4j states.

**Answer:**

A circuit breaker prevents cascading failures by **stopping calls to a failing service**.

```
                    ┌─────────┐
         success    │         │  failure rate < threshold
        ┌──────────│ CLOSED  │──────────┐
        │          │(normal) │          │
        │          └────┬────┘          │
        │               │               │
        │    failure rate >= threshold   │
        │               │               │
        │          ┌────▼────┐          │
        │          │         │          │
        │          │  OPEN   │──────────┘
        │          │(reject) │  after wait duration
        │          └────┬────┘
        │               │
        │    wait duration expires
        │               │
        │          ┌────▼────┐
        │          │ HALF-   │
        └──────────│ OPEN    │
         success   │(probe)  │
                   └────┬────┘
                        │
                   failure → back to OPEN
```

| State | Behavior |
|-------|----------|
| **CLOSED** | Normal operation, requests pass through. Monitors failure rate. |
| **OPEN** | All requests fail immediately (fast fail). Waits for timeout. |
| **HALF-OPEN** | Allows limited requests to test if service recovered. |

```java
// Resilience4j Circuit Breaker Configuration
@Configuration
public class CircuitBreakerConfig {
    
    @Bean
    public CircuitBreakerRegistry circuitBreakerRegistry() {
        io.github.resilience4j.circuitbreaker.CircuitBreakerConfig config = 
            io.github.resilience4j.circuitbreaker.CircuitBreakerConfig.custom()
                .failureRateThreshold(50)                    // Open at 50% failure rate
                .slowCallRateThreshold(80)                   // Open at 80% slow calls
                .slowCallDurationThreshold(Duration.ofSeconds(3)) // "Slow" = > 3s
                .waitDurationInOpenState(Duration.ofSeconds(30))  // Wait 30s before half-open
                .permittedNumberOfCallsInHalfOpenState(5)    // Allow 5 test calls
                .slidingWindowType(SlidingWindowType.COUNT_BASED)
                .slidingWindowSize(10)                       // Evaluate last 10 calls
                .minimumNumberOfCalls(5)                     // Need 5 calls before evaluating
                .recordExceptions(IOException.class, TimeoutException.class, 
                    ServiceUnavailableException.class)
                .ignoreExceptions(BusinessException.class)   // Don't count business errors
                .build();
        
        return CircuitBreakerRegistry.of(config);
    }
}

// Using Circuit Breaker with Payment Gateway
@Service
@Slf4j
public class PaymentGatewayService {
    
    private final CircuitBreaker circuitBreaker;
    private final PaymentGatewayClient gatewayClient;
    
    @CircuitBreaker(name = "paymentGateway", fallbackMethod = "fallbackCharge")
    @Retry(name = "paymentGateway")
    @TimeLimiter(name = "paymentGateway")
    public CompletableFuture<GatewayResponse> charge(PaymentRequest request) {
        return CompletableFuture.supplyAsync(() -> gatewayClient.charge(request));
    }
    
    // Fallback when circuit is open
    public CompletableFuture<GatewayResponse> fallbackCharge(PaymentRequest request, Throwable t) {
        log.warn("Circuit breaker fallback for txnId={}: {}", request.getTxnId(), t.getMessage());
        
        // Option 1: Queue for later processing
        pendingPaymentQueue.add(request);
        return CompletableFuture.completedFuture(
            GatewayResponse.pending("Payment queued for processing"));
        
        // Option 2: Try alternate gateway
        // return alternateGateway.charge(request);
    }
}
```

```yaml
# application.yml — Resilience4j config
resilience4j:
  circuitbreaker:
    instances:
      paymentGateway:
        failure-rate-threshold: 50
        slow-call-rate-threshold: 80
        slow-call-duration-threshold: 3s
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 5
        sliding-window-size: 10
        minimum-number-of-calls: 5
  retry:
    instances:
      paymentGateway:
        max-attempts: 3
        wait-duration: 1s
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
  timelimiter:
    instances:
      paymentGateway:
        timeout-duration: 5s
```

---

## Section 6: Event Sourcing & CQRS

### 💎 Q6. Explain Event Sourcing and CQRS. When would you use them?

**Answer:**

**Event Sourcing**: Store **events** (facts) instead of current state. Rebuild state by replaying events.

```
Traditional (State-based):
┌──────────────────────────────┐
│ Payment Table                 │
│ id=1, status=COMPLETED,      │
│ amount=100, updated_at=...   │  ← Only current state
└──────────────────────────────┘

Event Sourcing:
┌──────────────────────────────────────────────────┐
│ Payment Events                                    │
│ 1. PaymentInitiated  {txnId, amount=100, time}   │
│ 2. FraudCheckPassed  {txnId, score=0.1, time}    │
│ 3. PaymentAuthorized {txnId, authCode, time}     │
│ 4. PaymentCaptured   {txnId, capturedAmt, time}  │
│ 5. PaymentCompleted  {txnId, time}               │  ← Full history!
└──────────────────────────────────────────────────┘
Current state = replay all events → status=COMPLETED
```

**CQRS (Command Query Responsibility Segregation)**: Separate read and write models.

```
                    ┌─────────────────┐
                    │   API Gateway    │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
        ┌─────▼─────┐               ┌──────▼──────┐
        │  Command   │               │   Query     │
        │  Service   │               │   Service   │
        │ (writes)   │               │  (reads)    │
        └─────┬──────┘               └──────┬──────┘
              │                             │
        ┌─────▼──────┐               ┌──────▼──────┐
        │ Write DB   │──── Events ──→│  Read DB    │
        │ (Oracle,   │    (Kafka)    │ (Elastic,   │
        │  normalized)│               │  Redis,     │
        │            │               │  denormalized)│
        └────────────┘               └─────────────┘
```

**When to use:**

| Use Event Sourcing When | Don't Use When |
|------------------------|----------------|
| Full audit trail required (payments, compliance) | Simple CRUD |
| Need to replay/rebuild state | Low complexity domain |
| Complex business rules with temporal queries | Team unfamiliar with pattern |
| Debugging production issues (what happened?) | Small scale |

| Use CQRS When | Don't Use When |
|---------------|----------------|
| Read and write patterns are very different | Read/write ratio is balanced |
| Need different read models (search, reports) | Simple queries |
| High read:write ratio (100:1) | Small team, simple domain |
| Need independent scaling of reads vs writes | Consistency is paramount |

```java
// Event Sourcing — Payment Aggregate
public class PaymentAggregate {
    private String txnId;
    private BigDecimal amount;
    private PaymentStatus status;
    private List<DomainEvent> uncommittedEvents = new ArrayList<>();
    
    // Command handler
    public void initiate(String txnId, BigDecimal amount) {
        // Validate
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
        // Emit event (don't mutate state directly!)
        apply(new PaymentInitiated(txnId, amount, Instant.now()));
    }
    
    public void complete(String authCode) {
        if (status != PaymentStatus.AUTHORIZED) {
            throw new IllegalStateException("Cannot complete: current status=" + status);
        }
        apply(new PaymentCompleted(txnId, authCode, Instant.now()));
    }
    
    // Event handler (rebuilds state)
    private void on(PaymentInitiated event) {
        this.txnId = event.txnId();
        this.amount = event.amount();
        this.status = PaymentStatus.INITIATED;
    }
    
    private void on(PaymentCompleted event) {
        this.status = PaymentStatus.COMPLETED;
    }
    
    // Rebuild from event history
    public static PaymentAggregate fromHistory(List<DomainEvent> events) {
        PaymentAggregate aggregate = new PaymentAggregate();
        for (DomainEvent event : events) {
            aggregate.applyEvent(event); // Replay each event
        }
        return aggregate;
    }
}
```

---

## Section 7: Consistent Hashing

### 💎 Q7. Explain consistent hashing. Where is it used?

**Answer:**

Consistent hashing distributes data across nodes such that **adding/removing a node only affects ~1/N of the keys** (not all keys like modular hashing).

```
Regular Hashing: hash(key) % N
  Problem: If N changes (3→4 nodes), almost ALL keys remap!

Consistent Hashing:
  Hash ring (0 to 2^32):
  
       Node A (pos 100)
          ╱
    ─────●──────────────────●───── Node B (pos 300)
   ╱                          ╲
  ●                            ●
  Node D (pos 900)    Node C (pos 600)
   ╲                          ╱
    ──────────────────────────
    
  Key "payment-001" → hash = 250 → clockwise → Node B
  Key "payment-002" → hash = 700 → clockwise → Node D
  
  If Node B removed:
  Only keys between Node A (100) and Node B (300) remap to Node C
  Other keys unaffected!
```

**Virtual Nodes**: Each physical node gets multiple positions on the ring for better distribution.

**Used In:**
- **Redis Cluster**: Distributing keys across shards
- **Kafka**: Partition assignment
- **Load Balancers**: Sticky sessions
- **CDNs**: Content distribution
- **Cassandra/DynamoDB**: Data partitioning

---

## Section 8: Microservice Communication Patterns

### 🔥 Q8. Synchronous vs Asynchronous communication. When to use which?

**Answer:**

| Aspect | Synchronous (REST/gRPC) | Asynchronous (Kafka/RabbitMQ) |
|--------|------------------------|-------------------------------|
| Coupling | Tight (caller waits) | Loose (fire and forget) |
| Latency | Higher (chain of calls) | Lower (non-blocking) |
| Failure handling | Cascading failures | Isolated failures |
| Consistency | Immediate | Eventual |
| Debugging | Easier (request-response) | Harder (event tracing) |
| Use case | Queries, real-time needs | Events, notifications, long processes |

**Decision Guide for Payment System:**

```
User clicks "Pay" → Synchronous (needs immediate response)
    │
    ├── Payment Service → Gateway (sync — need auth code)
    │
    ├── Payment Completed → Kafka event (async)
    │       │
    │       ├── Notification Service (send email/SMS)
    │       ├── Analytics Service (update dashboards)
    │       ├── Reconciliation Service (record for settlement)
    │       └── Loyalty Service (award points)
    │
    └── Return response to user
```

**Patterns:**

| Pattern | Description | Use Case |
|---------|-------------|----------|
| **Request-Response** | Sync call, wait for response | Payment authorization |
| **Event Notification** | Publish event, no response expected | Order placed → notify warehouse |
| **Event-Carried State Transfer** | Event contains full data | Avoid callback to source |
| **Command** | Async request expecting action | "Process this refund" |
| **Saga** | Coordinated multi-service transaction | Order → Inventory → Payment |

---

## Quick Revision Checklist

- [ ] Idempotency: Client-generated key, check-before-process, cache response
- [ ] Saga: Orchestration (central coordinator) vs Choreography (events)
- [ ] Saga Compensation: Reverse order, handle compensation failures
- [ ] CAP: CP (payments, consistency) vs AP (catalog, availability)
- [ ] Rate Limiting: Token bucket (allows bursts), sliding window (accurate)
- [ ] Circuit Breaker: CLOSED → OPEN → HALF-OPEN, Resilience4j
- [ ] Event Sourcing: Store events, rebuild state by replay
- [ ] CQRS: Separate read/write models, different databases
- [ ] Consistent Hashing: Ring-based, virtual nodes, minimal remapping
- [ ] Sync vs Async: Sync for queries/real-time, Async for events/notifications
- [ ] Outbox Pattern: Atomic DB write + outbox, poll/CDC to Kafka
