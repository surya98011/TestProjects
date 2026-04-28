# 03 — Spring Data JPA & Hibernate (Entity Lifecycle, Caching, N+1, Locking, Transactions)

> **Priority**: 🔴 Critical  
> **Estimated Study Time**: 2 days  
> Markers: 🔥 = Frequently Asked | 💎 = Differentiator

---

## Section 1: Entity Lifecycle & Persistence Context

### 🔥 Q1. Explain the JPA Entity Lifecycle states.

**Answer:**

```
                    persist()
  NEW/TRANSIENT ──────────────→ MANAGED
       │                          │  │
       │                          │  │ flush()/commit()
       │                          │  ▼
       │                          │  DATABASE
       │                          │  │
       │              find()/     │  │ query
       │              getReference│  │
       │                          │  │
       │                    ┌─────┘  │
       │                    ▼        │
       │               MANAGED ←─────┘
       │                    │
       │         detach()/  │  merge()
       │         clear()/   │  ──────→ MANAGED
       │         close()    │
       │                    ▼
       │               DETACHED
       │
       │         remove()
       └──────────────────→ REMOVED
```

| State | In Persistence Context? | In Database? | Description |
|-------|------------------------|-------------|-------------|
| **Transient (New)** | No | No | Just created with `new`, not yet persisted |
| **Managed** | Yes | Yes (after flush) | Tracked by EntityManager, changes auto-synced |
| **Detached** | No | Yes | Was managed, now disconnected (session closed) |
| **Removed** | Yes (marked for deletion) | Yes (until flush) | Scheduled for deletion |

```java
// Lifecycle in action
@Transactional
public Payment processPayment(PaymentRequest request) {
    // 1. TRANSIENT — new object, not managed
    Payment payment = new Payment();
    payment.setTxnId(UUID.randomUUID().toString());
    payment.setAmount(request.getAmount());
    payment.setStatus(PaymentStatus.INITIATED);
    
    // 2. MANAGED — after persist(), tracked by persistence context
    entityManager.persist(payment);
    
    // 3. Dirty checking — changes are auto-detected
    payment.setStatus(PaymentStatus.PROCESSING); // No explicit save needed!
    
    // 4. At @Transactional commit → flush() → SQL UPDATE executed
    return payment;
}

// DETACHED example
public void updatePaymentStatus(Long paymentId, PaymentStatus newStatus) {
    Payment payment = entityManager.find(Payment.class, paymentId); // MANAGED
    entityManager.detach(payment); // Now DETACHED
    
    payment.setStatus(newStatus); // This change is NOT tracked!
    
    Payment merged = entityManager.merge(payment); // Back to MANAGED
    // merged is the managed copy, not the original 'payment' object
}
```

**Key Concept — Dirty Checking:**
- Hibernate takes a **snapshot** of entity state when it becomes managed
- At flush time, compares current state with snapshot
- If different → generates UPDATE SQL automatically
- No need to call `save()` on already-managed entities

---

### 💎 Q2. What is the difference between persist(), merge(), save(), and saveAndFlush()?

**Answer:**

| Method | Source | Behavior |
|--------|--------|----------|
| `persist()` | JPA EntityManager | Makes transient entity managed. Throws if entity already has ID (detached) |
| `merge()` | JPA EntityManager | Copies state of detached entity to a new managed copy. Returns the managed copy |
| `save()` | Spring Data JPA | Calls `persist()` if new (no ID), `merge()` if existing (has ID) |
| `saveAndFlush()` | Spring Data JPA | `save()` + immediate `flush()` to DB |

```java
// persist vs merge
@Transactional
public void demonstrateDifference() {
    // persist — for NEW entities
    Payment newPayment = new Payment();
    newPayment.setAmount(new BigDecimal("100.00"));
    entityManager.persist(newPayment); // newPayment IS the managed entity
    
    // merge — for DETACHED entities
    Payment detached = getDetachedPayment(); // From another session/API call
    detached.setStatus(PaymentStatus.COMPLETED);
    Payment managed = entityManager.merge(detached);
    // ⚠️ 'detached' is still detached! 'managed' is the managed copy
    // Always use the returned reference
    
    // Spring Data save() — handles both cases
    Payment payment = new Payment(); // No ID → persist()
    paymentRepository.save(payment);
    
    payment.setStatus(PaymentStatus.COMPLETED); // Has ID → merge()
    paymentRepository.save(payment);
}
```

**⚠️ Common Mistake:**
```java
// ❌ Wrong: Ignoring merge() return value
Payment detached = fetchFromCache();
detached.setAmount(newAmount);
entityManager.merge(detached); // Returns managed copy, but we ignore it!
detached.setStatus(COMPLETED); // This change is LOST!

// ✅ Correct
Payment managed = entityManager.merge(detached);
managed.setStatus(COMPLETED); // This change IS tracked
```

---

## Section 2: N+1 Problem

### 🔥 Q3. What is the N+1 problem? How do you solve it?

**Answer:**

The N+1 problem occurs when fetching a parent entity triggers **N additional queries** to load associated child entities.

**Example:**
```java
@Entity
public class Order {
    @Id private Long id;
    
    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    private List<OrderItem> items; // LAZY loaded
}

// N+1 Problem:
List<Order> orders = orderRepository.findAll(); // 1 query: SELECT * FROM orders
for (Order order : orders) {
    order.getItems().size(); // N queries: SELECT * FROM order_items WHERE order_id = ?
}
// Total: 1 + N queries! If 100 orders → 101 queries!
```

**Solutions:**

**1. JOIN FETCH (JPQL):**
```java
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.status = :status")
List<Order> findByStatusWithItems(@Param("status") OrderStatus status);
// Single query with JOIN
```

**2. @EntityGraph:**
```java
@EntityGraph(attributePaths = {"items", "items.product"})
@Query("SELECT o FROM Order o WHERE o.status = :status")
List<Order> findByStatusWithItems(@Param("status") OrderStatus status);
```

**3. @BatchSize (Hibernate):**
```java
@Entity
public class Order {
    @OneToMany(mappedBy = "order")
    @BatchSize(size = 25) // Loads items for 25 orders in one query
    private List<OrderItem> items;
}
// Instead of N queries, does ceil(N/25) queries
```

**4. Subselect Fetch:**
```java
@OneToMany(mappedBy = "order")
@Fetch(FetchMode.SUBSELECT)
private List<OrderItem> items;
// Uses: SELECT * FROM order_items WHERE order_id IN (SELECT id FROM orders WHERE ...)
```

**Comparison:**

| Solution | Queries | Cartesian Product Risk | Best For |
|----------|---------|----------------------|----------|
| JOIN FETCH | 1 | ⚠️ Yes (with multiple collections) | Single collection fetch |
| @EntityGraph | 1 | ⚠️ Yes | Declarative, reusable |
| @BatchSize | ceil(N/batch) | No | Multiple collections |
| Subselect | 2 | No | Large result sets |
| DTO Projection | 1 | No | Read-only views |

**Best Practice — DTO Projection (avoids N+1 entirely):**
```java
public record OrderSummaryDTO(Long orderId, String customerName, BigDecimal totalAmount, int itemCount) {}

@Query("""
    SELECT new com.shopease.dto.OrderSummaryDTO(
        o.id, o.customerName, o.totalAmount, SIZE(o.items))
    FROM Order o WHERE o.status = :status
    """)
List<OrderSummaryDTO> findOrderSummaries(@Param("status") OrderStatus status);
```

---

## Section 3: Fetch Strategies

### 🔥 Q4. Explain EAGER vs LAZY loading. What are the defaults?

**Answer:**

| Mapping | Default Fetch | Recommendation |
|---------|--------------|----------------|
| `@ManyToOne` | EAGER | Change to LAZY |
| `@OneToOne` | EAGER | Change to LAZY |
| `@OneToMany` | LAZY | Keep LAZY |
| `@ManyToMany` | LAZY | Keep LAZY |

**Rule: Always use LAZY loading, then fetch eagerly when needed via JOIN FETCH or @EntityGraph.**

```java
@Entity
public class Payment {
    @Id @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;
    
    // ✅ LAZY — loaded only when accessed
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "merchant_id")
    private Merchant merchant;
    
    // ✅ LAZY by default
    @OneToMany(mappedBy = "payment", fetch = FetchType.LAZY)
    private List<Refund> refunds;
}
```

**LazyInitializationException:**
```java
// ❌ This fails outside a transaction
Payment payment = paymentRepository.findById(1L).orElseThrow();
// Transaction is closed here
payment.getMerchant().getName(); // LazyInitializationException!

// ✅ Solution 1: Use @Transactional
@Transactional(readOnly = true)
public PaymentDTO getPaymentWithMerchant(Long id) {
    Payment payment = paymentRepository.findById(id).orElseThrow();
    return new PaymentDTO(payment, payment.getMerchant().getName()); // Works!
}

// ✅ Solution 2: JOIN FETCH in query
@Query("SELECT p FROM Payment p JOIN FETCH p.merchant WHERE p.id = :id")
Optional<Payment> findByIdWithMerchant(@Param("id") Long id);
```

---

## Section 4: Caching (1st Level & 2nd Level)

### 🔥 Q5. Explain Hibernate 1st Level and 2nd Level Cache.

**Answer:**

```
┌─────────────────────────────────────────────────────────┐
│                    Application                           │
│                                                          │
│  Session 1          Session 2          Session 3         │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐      │
│  │ 1st Level│      │ 1st Level│      │ 1st Level│      │
│  │  Cache   │      │  Cache   │      │  Cache   │      │
│  │(per sess)│      │(per sess)│      │(per sess)│      │
│  └────┬─────┘      └────┬─────┘      └────┬─────┘      │
│       │                  │                  │            │
│       └──────────────────┼──────────────────┘            │
│                          │                               │
│                 ┌────────▼────────┐                      │
│                 │  2nd Level Cache │                      │
│                 │  (SessionFactory)│                      │
│                 │  Ehcache/Redis   │                      │
│                 └────────┬────────┘                      │
│                          │                               │
│                 ┌────────▼────────┐                      │
│                 │  Query Cache     │                      │
│                 │  (query → IDs)   │                      │
│                 └────────┬────────┘                      │
└──────────────────────────┼───────────────────────────────┘
                           │
                  ┌────────▼────────┐
                  │    Database      │
                  └─────────────────┘
```

| Feature | 1st Level Cache | 2nd Level Cache |
|---------|----------------|-----------------|
| Scope | Per Session/EntityManager | Per SessionFactory (shared) |
| Enabled by default | Yes (always) | No (must configure) |
| Eviction | Session close/clear | TTL, size-based, manual |
| Thread-safe | No (session is not thread-safe) | Yes |
| Storage | In-memory (HashMap) | Ehcache, Redis, Hazelcast |

**Configuring 2nd Level Cache:**
```java
@Entity
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE, region = "merchants")
public class Merchant {
    @Id private Long id;
    private String name;
    private String mcc; // Merchant Category Code
}
```

```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          use_query_cache: true
          region:
            factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
        javax:
          cache:
            provider: org.ehcache.jsr107.EhcacheCachingProvider
```

**Cache Concurrency Strategies:**

| Strategy | Use Case | Consistency |
|----------|----------|-------------|
| `READ_ONLY` | Reference data (countries, currencies) | Strict |
| `NONSTRICT_READ_WRITE` | Rarely updated data | Eventual |
| `READ_WRITE` | Frequently read, sometimes updated | Strong (soft locks) |
| `TRANSACTIONAL` | Full transactional consistency | Strict (JTA required) |

---

## Section 5: Locking

### 🔥 Q6. Optimistic vs Pessimistic Locking — when to use which?

**Answer:**

| Aspect | Optimistic Locking | Pessimistic Locking |
|--------|-------------------|---------------------|
| Mechanism | Version column (`@Version`) | DB row lock (`SELECT ... FOR UPDATE`) |
| Conflict detection | At commit time | At read time |
| Performance | Better (no DB locks) | Worse (holds DB locks) |
| Contention | Low contention scenarios | High contention scenarios |
| Exception | `OptimisticLockException` | Blocks other transactions |
| Use case | Most CRUD operations | Account balance updates, inventory |

**Optimistic Locking:**
```java
@Entity
public class Payment {
    @Id @GeneratedValue
    private Long id;
    
    @Version  // Hibernate auto-increments on each update
    private Long version;
    
    private BigDecimal amount;
    private PaymentStatus status;
}

// How it works:
// UPDATE payment SET status='COMPLETED', version=2 WHERE id=1 AND version=1
// If version doesn't match → OptimisticLockException

// Handling the exception
@Retryable(value = OptimisticLockException.class, maxAttempts = 3, backoff = @Backoff(delay = 100))
@Transactional
public Payment updatePaymentStatus(Long paymentId, PaymentStatus newStatus) {
    Payment payment = paymentRepository.findById(paymentId).orElseThrow();
    payment.setStatus(newStatus);
    return paymentRepository.save(payment);
}
```

**Pessimistic Locking:**
```java
public interface AccountRepository extends JpaRepository<Account, Long> {
    
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT a FROM Account a WHERE a.id = :id")
    Optional<Account> findByIdForUpdate(@Param("id") Long id);
    // Generates: SELECT ... FROM account WHERE id = ? FOR UPDATE
}

// Use case: Transfer money (high contention on balance)
@Transactional
public void transfer(Long fromId, Long toId, BigDecimal amount) {
    // Always lock in consistent order to prevent deadlocks
    Long firstId = Math.min(fromId, toId);
    Long secondId = Math.max(fromId, toId);
    
    Account first = accountRepository.findByIdForUpdate(firstId).orElseThrow();
    Account second = accountRepository.findByIdForUpdate(secondId).orElseThrow();
    
    Account from = fromId.equals(firstId) ? first : second;
    Account to = toId.equals(firstId) ? first : second;
    
    if (from.getBalance().compareTo(amount) < 0) {
        throw new InsufficientBalanceException("Insufficient funds");
    }
    
    from.setBalance(from.getBalance().subtract(amount));
    to.setBalance(to.getBalance().add(amount));
}
```

**Decision Guide:**
- **Payment status updates** → Optimistic (low contention, retryable)
- **Wallet balance deduction** → Pessimistic (high contention, must be accurate)
- **Inventory decrement** → Pessimistic (flash sales, high contention)
- **User profile update** → Optimistic (rarely concurrent)

---

## Section 6: @Transactional Deep Dive

### 🔥 Q7. Explain @Transactional propagation levels with real examples.

**Answer:**

| Propagation | Behavior | Use Case |
|-------------|----------|----------|
| `REQUIRED` (default) | Join existing txn, or create new | Most service methods |
| `REQUIRES_NEW` | Always create new txn (suspend current) | Audit logging, notifications |
| `NESTED` | Savepoint within current txn | Partial rollback |
| `MANDATORY` | Must run within existing txn, else exception | Repository methods |
| `SUPPORTS` | Use txn if exists, else run without | Read-only queries |
| `NOT_SUPPORTED` | Suspend current txn, run without | Long-running non-DB operations |
| `NEVER` | Must NOT run within txn, else exception | Validation-only methods |

```java
@Service
public class PaymentService {
    
    // REQUIRED (default) — joins existing or creates new
    @Transactional
    public PaymentResult processPayment(PaymentRequest request) {
        Payment payment = createPayment(request);
        GatewayResponse response = chargeGateway(request);
        updatePaymentStatus(payment, response);
        
        // If this fails, everything rolls back
        sendNotification(payment); // What if we don't want this to affect the txn?
        
        return toResult(payment);
    }
    
    // REQUIRES_NEW — audit log should persist even if payment fails
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void auditLog(String action, String details) {
        auditRepository.save(new AuditLog(action, details, Instant.now()));
        // This commits independently — survives parent rollback
    }
    
    // REQUIRES_NEW — notification failure shouldn't rollback payment
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendNotification(Payment payment) {
        try {
            notificationService.send(payment);
        } catch (Exception e) {
            log.warn("Notification failed for txn={}", payment.getTxnId(), e);
            // Swallow exception — payment should still succeed
        }
    }
}
```

**⚠️ Common @Transactional Pitfalls:**

```java
// ❌ Pitfall 1: Self-invocation (proxy bypass)
@Service
public class PaymentService {
    
    @Transactional
    public void processPayment() {
        // ...
        this.sendNotification(); // ❌ Calls method directly, NOT through proxy!
        // @Transactional on sendNotification is IGNORED
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendNotification() { ... }
}

// ✅ Fix: Inject self or use separate service
@Service
public class PaymentService {
    @Autowired @Lazy
    private PaymentService self; // Inject proxy
    
    @Transactional
    public void processPayment() {
        self.sendNotification(); // ✅ Goes through proxy
    }
}

// ❌ Pitfall 2: Catching exception inside @Transactional
@Transactional
public void process() {
    try {
        repository.save(entity);
        externalService.call(); // Throws RuntimeException
    } catch (Exception e) {
        log.error("Failed", e);
        // ❌ Transaction is already marked for rollback!
        // Commit will fail with UnexpectedRollbackException
    }
}

// ❌ Pitfall 3: @Transactional on private method
@Transactional // ❌ IGNORED — Spring AOP can't proxy private methods
private void updateStatus() { ... }

// ❌ Pitfall 4: Checked exceptions don't trigger rollback by default
@Transactional // Only rolls back on RuntimeException and Error
public void process() throws PaymentException {
    throw new PaymentException("Failed"); // ❌ No rollback! (checked exception)
}

// ✅ Fix
@Transactional(rollbackFor = PaymentException.class)
public void process() throws PaymentException { ... }
```

---

### 💎 Q8. What is @Transactional(readOnly = true) and why should you use it?

**Answer:**

`readOnly = true` is a **hint** to the persistence provider and database:

**Benefits:**
1. **Hibernate skips dirty checking** — no snapshot comparison at flush time → faster
2. **No flush at commit** — read-only entities are not flushed
3. **DB optimization** — some databases route to read replicas
4. **JDBC driver optimization** — can use read-only connection mode

```java
@Service
@Transactional(readOnly = true) // Default for all methods in this service
public class PaymentQueryService {
    
    public PaymentDTO findByTxnId(String txnId) {
        return paymentRepository.findByTxnId(txnId)
            .map(this::toDTO)
            .orElseThrow(() -> new PaymentNotFoundException(txnId));
    }
    
    public Page<PaymentDTO> findByMerchant(String merchantId, Pageable pageable) {
        return paymentRepository.findByMerchantId(merchantId, pageable)
            .map(this::toDTO);
    }
    
    @Transactional // Override: this method needs write
    public Payment updateStatus(Long id, PaymentStatus status) {
        Payment payment = paymentRepository.findById(id).orElseThrow();
        payment.setStatus(status);
        return payment;
    }
}
```

**With Read Replicas (Spring Boot):**
```java
@Configuration
public class DataSourceConfig {
    
    @Bean
    public DataSource routingDataSource() {
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put("primary", primaryDataSource());
        targetDataSources.put("replica", replicaDataSource());
        
        AbstractRoutingDataSource routingDataSource = new AbstractRoutingDataSource() {
            @Override
            protected Object determineCurrentLookupKey() {
                return TransactionSynchronizationManager.isCurrentTransactionReadOnly() 
                    ? "replica" : "primary";
            }
        };
        routingDataSource.setTargetDataSources(targetDataSources);
        routingDataSource.setDefaultTargetDataSource(primaryDataSource());
        return routingDataSource;
    }
}
```

---

## Section 7: Batch Operations & Performance

### 💎 Q9. How do you optimize batch inserts in Hibernate?

**Answer:**

By default, Hibernate inserts one row at a time. For bulk operations, this is extremely slow.

**Configuration:**
```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
          batch_versioned_data: true
        order_inserts: true
        order_updates: true
        generate_statistics: true  # Monitor batch effectiveness
```

```java
// ❌ Slow: 10,000 individual INSERTs
@Transactional
public void importPayments(List<PaymentDTO> dtos) {
    for (PaymentDTO dto : dtos) {
        paymentRepository.save(toEntity(dto)); // 10,000 INSERT statements
    }
}

// ✅ Fast: Batched INSERTs (50 at a time)
@Transactional
public void importPaymentsBatched(List<PaymentDTO> dtos) {
    int batchSize = 50;
    for (int i = 0; i < dtos.size(); i++) {
        entityManager.persist(toEntity(dtos.get(i)));
        
        if (i > 0 && i % batchSize == 0) {
            entityManager.flush();  // Execute batch INSERT
            entityManager.clear();  // Free memory (detach all entities)
        }
    }
    entityManager.flush();
    entityManager.clear();
}
```

**⚠️ IDENTITY generation strategy disables batching!**
```java
// ❌ Disables batch inserts (needs DB roundtrip for each ID)
@Id @GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;

// ✅ Allows batch inserts
@Id @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "payment_seq")
@SequenceGenerator(name = "payment_seq", sequenceName = "PAYMENT_SEQ", allocationSize = 50)
private Long id;
```

**Spring Data JPA `saveAll()` with batching:**
```java
// saveAll() uses batch inserts IF:
// 1. hibernate.jdbc.batch_size is set
// 2. ID generation is SEQUENCE (not IDENTITY)
// 3. order_inserts = true
paymentRepository.saveAll(payments); // Batched!
```

---

### 💎 Q10. JPQL vs Criteria API vs Native Query — when to use which?

**Answer:**

| Approach | Type Safety | Dynamic | Readability | Use Case |
|----------|------------|---------|-------------|----------|
| **JPQL** | No (string) | No | ✅ High | Simple, static queries |
| **Criteria API** | ✅ Yes | ✅ Yes | Low | Dynamic filters, search |
| **Native SQL** | No | No | Medium | DB-specific features, complex joins |
| **Querydsl** | ✅ Yes | ✅ Yes | ✅ High | Best of both worlds |
| **Specification** | ✅ Yes | ✅ Yes | Medium | Spring Data dynamic queries |

```java
// JPQL — Simple, readable
@Query("SELECT p FROM Payment p WHERE p.merchantId = :merchantId AND p.status = :status")
List<Payment> findByMerchantAndStatus(@Param("merchantId") String merchantId, 
                                       @Param("status") PaymentStatus status);

// Specification — Dynamic filters (Spring Data)
public class PaymentSpecifications {
    
    public static Specification<Payment> withFilters(PaymentSearchCriteria criteria) {
        return (root, query, cb) -> {
            List<Predicate> predicates = new ArrayList<>();
            
            if (criteria.getMerchantId() != null) {
                predicates.add(cb.equal(root.get("merchantId"), criteria.getMerchantId()));
            }
            if (criteria.getStatus() != null) {
                predicates.add(cb.equal(root.get("status"), criteria.getStatus()));
            }
            if (criteria.getFromDate() != null) {
                predicates.add(cb.greaterThanOrEqualTo(root.get("createdAt"), criteria.getFromDate()));
            }
            if (criteria.getToDate() != null) {
                predicates.add(cb.lessThanOrEqualTo(root.get("createdAt"), criteria.getToDate()));
            }
            if (criteria.getMinAmount() != null) {
                predicates.add(cb.greaterThanOrEqualTo(root.get("amount"), criteria.getMinAmount()));
            }
            
            return cb.and(predicates.toArray(new Predicate[0]));
        };
    }
}

// Usage
Page<Payment> results = paymentRepository.findAll(
    PaymentSpecifications.withFilters(searchCriteria), 
    PageRequest.of(0, 20, Sort.by("createdAt").descending())
);

// Native Query — Oracle-specific features
@Query(value = """
    SELECT /*+ INDEX(p IDX_PAYMENT_MERCHANT_DATE) */ 
           p.* FROM payments p 
    WHERE p.merchant_id = :merchantId 
    AND p.created_at BETWEEN :fromDate AND :toDate
    AND ROWNUM <= 1000
    """, nativeQuery = true)
List<Payment> findRecentPayments(@Param("merchantId") String merchantId,
                                  @Param("fromDate") Instant fromDate,
                                  @Param("toDate") Instant toDate);
```

---

## Section 8: Auditing & Soft Deletes

### 💎 Q11. How do you implement auditing and soft deletes?

**Answer:**

**Auditing with Spring Data JPA:**
```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {
    
    @CreatedDate
    @Column(updatable = false)
    private Instant createdAt;
    
    @LastModifiedDate
    private Instant updatedAt;
    
    @CreatedBy
    @Column(updatable = false)
    private String createdBy;
    
    @LastModifiedBy
    private String updatedBy;
}

@Configuration
@EnableJpaAuditing
public class JpaConfig {
    
    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
            .map(Authentication::getName)
            .or(() -> Optional.of("SYSTEM"));
    }
}

@Entity
@Table(name = "payments")
public class Payment extends BaseEntity {
    @Id @GeneratedValue
    private Long id;
    private String txnId;
    private BigDecimal amount;
}
```

**Soft Deletes:**
```java
@Entity
@Table(name = "payments")
@SQLDelete(sql = "UPDATE payments SET deleted = true, deleted_at = CURRENT_TIMESTAMP WHERE id = ?")
@SQLRestriction("deleted = false")  // Hibernate 6.x (replaces @Where)
public class Payment extends BaseEntity {
    
    @Id @GeneratedValue
    private Long id;
    
    private boolean deleted = false;
    
    private Instant deletedAt;
}

// Usage — delete() now does soft delete
paymentRepository.delete(payment);
// Executes: UPDATE payments SET deleted = true, deleted_at = CURRENT_TIMESTAMP WHERE id = ?

// findAll() automatically filters deleted records
paymentRepository.findAll();
// Executes: SELECT * FROM payments WHERE deleted = false

// To include deleted records (admin use case)
@Query(value = "SELECT * FROM payments WHERE txn_id = :txnId", nativeQuery = true)
Optional<Payment> findByTxnIdIncludingDeleted(@Param("txnId") String txnId);
```

---

## Quick Revision Checklist

- [ ] Entity Lifecycle: Transient → Managed → Detached → Removed
- [ ] Dirty Checking: Hibernate auto-detects changes on managed entities
- [ ] persist() vs merge(): persist for new, merge for detached (use return value!)
- [ ] N+1 Problem: JOIN FETCH, @EntityGraph, @BatchSize, DTO Projection
- [ ] Fetch Strategy: Always LAZY, fetch eagerly when needed
- [ ] 1st Level Cache: Per session, always on. 2nd Level: Shared, configure explicitly
- [ ] Optimistic Lock: @Version column, good for low contention
- [ ] Pessimistic Lock: SELECT FOR UPDATE, good for high contention (balance, inventory)
- [ ] @Transactional: REQUIRED (default), REQUIRES_NEW (independent), self-invocation pitfall
- [ ] readOnly = true: Skips dirty checking, can route to read replica
- [ ] Batch Inserts: batch_size + SEQUENCE strategy + order_inserts
- [ ] JPQL vs Specification: JPQL for static, Specification for dynamic queries
- [ ] Auditing: @CreatedDate, @LastModifiedDate, AuditorAware
- [ ] Soft Deletes: @SQLDelete + @SQLRestriction
