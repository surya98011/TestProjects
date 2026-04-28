# 01 — Java Core (Java 8, Java 21, Memory, GC, Concurrency, Collections)

> **Priority**: 🔴 Critical  
> **Estimated Study Time**: 2 days  
> Markers: 🔥 = Frequently Asked | 💎 = Differentiator

---

## Section 1: Java 8 Features

### 🔥 Q1. What are Functional Interfaces? Name the key ones in java.util.function.

**Answer:**  
A Functional Interface has exactly **one abstract method** (SAM — Single Abstract Method). It can have default and static methods. Annotated with `@FunctionalInterface`.

**Key Functional Interfaces:**

| Interface | Method | Use Case |
|-----------|--------|----------|
| `Predicate<T>` | `boolean test(T t)` | Filtering |
| `Function<T,R>` | `R apply(T t)` | Transformation |
| `Consumer<T>` | `void accept(T t)` | Side effects (logging, saving) |
| `Supplier<T>` | `T get()` | Lazy creation / factory |
| `UnaryOperator<T>` | `T apply(T t)` | Same type transformation |
| `BiFunction<T,U,R>` | `R apply(T t, U u)` | Two-arg transformation |

```java
// Production example: Payment validation chain
Predicate<Payment> isAmountValid = p -> p.getAmount().compareTo(BigDecimal.ZERO) > 0;
Predicate<Payment> isCurrencySupported = p -> SUPPORTED_CURRENCIES.contains(p.getCurrency());
Predicate<Payment> isNotDuplicate = p -> !transactionCache.contains(p.getIdempotencyKey());

Predicate<Payment> fullValidation = isAmountValid
    .and(isCurrencySupported)
    .and(isNotDuplicate);

if (!fullValidation.test(payment)) {
    throw new PaymentValidationException("Validation failed");
}
```

---

### 🔥 Q2. Explain Java Streams in depth. What are intermediate vs terminal operations?

**Answer:**  
Streams provide a **declarative, pipeline-based** approach to process collections. They are **lazy** — intermediate operations don't execute until a terminal operation is invoked.

**Intermediate Operations** (return Stream, lazy):
- `filter()`, `map()`, `flatMap()`, `distinct()`, `sorted()`, `peek()`, `limit()`, `skip()`

**Terminal Operations** (trigger execution):
- `collect()`, `forEach()`, `reduce()`, `count()`, `findFirst()`, `findAny()`, `anyMatch()`, `allMatch()`, `noneMatch()`, `toArray()`

**Key Characteristics:**
- Streams are **not reusable** — once consumed, you must create a new stream
- Streams don't modify the source collection
- Short-circuiting: `findFirst()`, `anyMatch()`, `limit()` can terminate early

```java
// Production: Aggregate daily payment totals by currency
Map<Currency, BigDecimal> dailyTotals = payments.stream()
    .filter(p -> p.getStatus() == PaymentStatus.COMPLETED)
    .filter(p -> p.getCreatedAt().toLocalDate().equals(LocalDate.now()))
    .collect(Collectors.groupingBy(
        Payment::getCurrency,
        Collectors.reducing(BigDecimal.ZERO, Payment::getAmount, BigDecimal::add)
    ));
```

**Stream Pipeline Execution (Lazy Evaluation):**
```
Source → filter → map → collect
         ↑ Nothing happens until collect() is called
```

---

### 🔥 Q3. What is Optional? How do you use it correctly?

**Answer:**  
`Optional<T>` is a container that may or may not hold a non-null value. Introduced to **eliminate NullPointerException** and make APIs more expressive.

**Do's and Don'ts:**

| Do ✅ | Don't ❌ |
|-------|---------|
| Use as return type | Use as method parameter |
| Use `map()`, `flatMap()`, `orElse()` | Use `get()` without `isPresent()` |
| Use `orElseThrow()` for mandatory values | Use for class fields |
| Use `ifPresent()` for side effects | Use `Optional.of(null)` |

```java
// Production: Fetch payment with fallback
public PaymentResponse getPayment(String paymentId) {
    return paymentRepository.findById(paymentId)
        .map(this::toPaymentResponse)
        .orElseThrow(() -> new PaymentNotFoundException(
            "Payment not found: " + paymentId));
}

// Chaining Optionals
public String getCardLast4(Order order) {
    return Optional.ofNullable(order)
        .map(Order::getPayment)
        .map(Payment::getCard)
        .map(Card::getLast4Digits)
        .orElse("****");
}
```

---

### 🔥 Q4. Explain CompletableFuture vs Future. When would you use each?

**Answer:**  

| Feature | Future | CompletableFuture |
|---------|--------|-------------------|
| Blocking | `get()` blocks the thread | Non-blocking with `thenApply()`, `thenCompose()` |
| Chaining | Not possible | Full chaining support |
| Combining | Not possible | `thenCombine()`, `allOf()`, `anyOf()` |
| Exception handling | Only via `ExecutionException` | `exceptionally()`, `handle()` |
| Manual completion | Not possible | `complete()`, `completeExceptionally()` |

```java
// Production: Parallel payment processing
public CompletableFuture<PaymentResult> processPayment(PaymentRequest request) {
    CompletableFuture<FraudCheckResult> fraudCheck = CompletableFuture
        .supplyAsync(() -> fraudService.check(request), paymentExecutor);
    
    CompletableFuture<KycResult> kycCheck = CompletableFuture
        .supplyAsync(() -> kycService.verify(request.getUserId()), paymentExecutor);
    
    CompletableFuture<BalanceResult> balanceCheck = CompletableFuture
        .supplyAsync(() -> walletService.checkBalance(request), paymentExecutor);
    
    return CompletableFuture.allOf(fraudCheck, kycCheck, balanceCheck)
        .thenApplyAsync(v -> {
            if (fraudCheck.join().isFraudulent()) {
                throw new FraudDetectedException("Fraud detected");
            }
            return paymentGateway.charge(request);
        }, paymentExecutor)
        .exceptionally(ex -> {
            log.error("Payment failed for txnId={}", request.getTxnId(), ex);
            return PaymentResult.failed(ex.getMessage());
        });
}
```

**When to use:**
- **Future**: Simple async tasks where you can afford to block
- **CompletableFuture**: Complex workflows with chaining, combining multiple async operations, non-blocking pipelines (payment orchestration, parallel API calls)

---

### 💎 Q5. What is the difference between map() and flatMap() in Streams and Optional?

**Answer:**

**In Streams:**
- `map()`: One-to-one transformation. Wraps result in a Stream.
- `flatMap()`: One-to-many transformation. Flattens nested streams into a single stream.

```java
// map: Each order → one status
List<String> statuses = orders.stream()
    .map(Order::getStatus)  // Stream<String>
    .collect(Collectors.toList());

// flatMap: Each order → multiple line items
List<LineItem> allItems = orders.stream()
    .flatMap(order -> order.getLineItems().stream())  // Flattened Stream<LineItem>
    .collect(Collectors.toList());
```

**In Optional:**
- `map()`: Transforms value, wraps in Optional → `Optional<Optional<T>>` if mapper returns Optional
- `flatMap()`: Transforms and flattens → avoids nested Optionals

```java
// map gives Optional<Optional<Address>>
Optional<Optional<Address>> nested = user.map(User::getAddress);

// flatMap gives Optional<Address>
Optional<Address> flat = user.flatMap(User::getAddress);
```

---

## Section 2: Java 21 Features

### 🔥 Q6. What are Virtual Threads? How do they differ from platform threads?

**Answer:**  
Virtual Threads (Project Loom, JEP 444) are **lightweight threads managed by the JVM**, not the OS.

| Aspect | Platform Threads | Virtual Threads |
|--------|-----------------|-----------------|
| Managed by | OS kernel | JVM scheduler |
| Memory | ~1MB stack each | ~few KB |
| Count | Thousands (limited) | Millions possible |
| Blocking cost | Expensive (blocks OS thread) | Cheap (unmounts from carrier) |
| Context switch | OS-level (expensive) | JVM-level (cheap) |
| Best for | CPU-bound tasks | I/O-bound tasks |

```java
// Creating virtual threads
Thread vThread = Thread.ofVirtual().name("payment-worker").start(() -> {
    processPayment(request);
});

// Virtual thread executor (production)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<PaymentResult>> futures = payments.stream()
        .map(p -> executor.submit(() -> processPayment(p)))
        .toList();
    
    for (Future<PaymentResult> f : futures) {
        results.add(f.get());
    }
}

// Spring Boot 3.2+ configuration
// application.yml
// spring.threads.virtual.enabled: true
```

**⚠️ Gotchas:**
- Don't use `synchronized` blocks (pins the carrier thread) — use `ReentrantLock` instead
- Don't pool virtual threads — create new ones per task
- ThreadLocal works but can be memory-heavy with millions of threads — use ScopedValues instead

---

### 💎 Q7. Explain Records, Sealed Classes, and Pattern Matching in Java 21.

**Answer:**

**Records** (JEP 395): Immutable data carriers with auto-generated `equals()`, `hashCode()`, `toString()`, constructor, and accessors.

```java
public record PaymentEvent(
    String transactionId,
    BigDecimal amount,
    Currency currency,
    Instant timestamp
) {
    // Compact constructor for validation
    public PaymentEvent {
        Objects.requireNonNull(transactionId, "transactionId must not be null");
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
    }
}
```

**Sealed Classes** (JEP 409): Restrict which classes can extend/implement.

```java
public sealed interface PaymentMethod permits CreditCard, DebitCard, UPI, Wallet {
    String getIdentifier();
}

public record CreditCard(String last4, String network) implements PaymentMethod {
    public String getIdentifier() { return "CC-" + last4; }
}
public record UPI(String vpa) implements PaymentMethod {
    public String getIdentifier() { return "UPI-" + vpa; }
}
```

**Pattern Matching** (JEP 441): Enhanced switch with type patterns.

```java
public BigDecimal calculateFee(PaymentMethod method) {
    return switch (method) {
        case CreditCard cc when cc.network().equals("AMEX") -> amount.multiply(new BigDecimal("0.029"));
        case CreditCard cc -> amount.multiply(new BigDecimal("0.025"));
        case DebitCard dc -> amount.multiply(new BigDecimal("0.010"));
        case UPI upi -> BigDecimal.ZERO;  // UPI is free in India
        case Wallet w -> amount.multiply(new BigDecimal("0.015"));
    };
}
```

---

## Section 3: Memory Model & Garbage Collection

### 🔥 Q8. Explain the Java Memory Model (JMM). What are the memory areas?

**Answer:**

```
┌─────────────────────────────────────────────────┐
│                   JVM Memory                     │
├──────────────────┬──────────────────────────────┤
│   Heap (shared)  │   Non-Heap                   │
│  ┌────────────┐  │  ┌─────────────────────────┐ │
│  │ Young Gen  │  │  │ Metaspace (class meta)  │ │
│  │ ┌────────┐ │  │  │ Code Cache (JIT)        │ │
│  │ │ Eden   │ │  │  │ Thread Stacks           │ │
│  │ ├────────┤ │  │  │ Direct Memory (NIO)     │ │
│  │ │ S0/S1  │ │  │  └─────────────────────────┘ │
│  │ └────────┘ │  │                              │
│  ├────────────┤  │                              │
│  │  Old Gen   │  │                              │
│  └────────────┘  │                              │
└──────────────────┴──────────────────────────────┘
```

| Area | Stores | Shared? |
|------|--------|---------|
| **Heap — Young Gen (Eden + S0/S1)** | New objects | Yes |
| **Heap — Old Gen** | Long-lived objects | Yes |
| **Metaspace** | Class metadata, method info | Yes |
| **Thread Stack** | Local variables, method frames | No (per thread) |
| **PC Register** | Current instruction address | No (per thread) |
| **Code Cache** | JIT-compiled native code | Yes |

**Key JMM Concepts:**
- **Happens-before**: Guarantees visibility of writes across threads
- **volatile**: Ensures visibility (no caching in CPU registers), prevents reordering
- **synchronized**: Ensures atomicity + visibility + ordering

---

### 🔥 Q9. Explain different Garbage Collectors. Which one would you choose for a payment service?

**Answer:**

| GC | Algorithm | Pause Time | Throughput | Best For |
|----|-----------|-----------|------------|----------|
| **Serial GC** | Mark-Sweep-Compact | High | Low | Single-core, small heaps |
| **Parallel GC** | Parallel Mark-Sweep-Compact | Medium | High | Batch processing |
| **G1 GC** | Region-based, concurrent | Low-Medium | Good | General purpose (default since Java 9) |
| **ZGC** | Colored pointers, load barriers | Ultra-low (<1ms) | Good | Large heaps, low-latency |
| **Shenandoah** | Brooks pointers, concurrent | Ultra-low | Good | Low-latency alternative |

**For a Payment Service, I'd choose ZGC:**
- Payment APIs need **sub-10ms p99 latency**
- ZGC provides **<1ms pause times** regardless of heap size
- Handles large heaps (multi-GB) without long pauses

```bash
# Production JVM flags for payment service
java -XX:+UseZGC \
     -XX:+ZGenerational \
     -Xms4g -Xmx4g \
     -XX:MaxGCPauseMillis=5 \
     -XX:+UseStringDeduplication \
     -XX:MetaspaceSize=256m \
     -XX:MaxMetaspaceSize=256m \
     -Xlog:gc*:file=gc.log:time,uptime,level,tags:filecount=5,filesize=50m \
     -jar payment-service.jar
```

---

### 💎 Q10. How would you debug a memory leak in production?

**Answer:**

**Step-by-step approach:**

1. **Detect**: Monitor via Dynatrace/Grafana — look for steadily increasing heap usage that doesn't drop after GC
2. **Confirm**: Check GC logs — if Full GC frequency increases but free memory doesn't recover
3. **Capture Heap Dump**:
```bash
# Trigger heap dump
jmap -dump:live,format=b,file=heapdump.hprof <PID>

# Or auto-dump on OOM (set in JVM flags)
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/logs/heapdumps/
```
4. **Analyze**: Use Eclipse MAT or VisualVM
   - Look at **Dominator Tree** — which objects retain the most memory
   - Check **Leak Suspects Report**
   - Examine **GC Roots** — what's preventing collection

**Common Leak Sources in Payment Services:**
- Unbounded caches (no TTL, no max size)
- Connection pool leaks (unclosed DB/HTTP connections)
- ThreadLocal not cleaned up in thread pools
- Event listeners not deregistered
- Large collections in static fields

```java
// Common leak: ThreadLocal not cleaned
public class RequestContext {
    private static final ThreadLocal<Map<String, String>> context = new ThreadLocal<>();
    
    // ❌ Leak: Thread pool reuses threads, ThreadLocal accumulates
    public static void set(String key, String value) {
        context.get().put(key, value);
    }
    
    // ✅ Fix: Always clean up in finally block or use try-with-resources
    public static void clear() {
        context.remove();  // MUST call this
    }
}
```

---

## Section 4: Concurrency

### 🔥 Q11. Explain synchronized, ReentrantLock, and CAS. When to use which?

**Answer:**

| Feature | synchronized | ReentrantLock | CAS (Compare-And-Swap) |
|---------|-------------|---------------|------------------------|
| Type | Intrinsic lock (monitor) | Explicit lock | Lock-free |
| Fairness | No | Configurable | N/A |
| Try lock | No | `tryLock(timeout)` | N/A |
| Interruptible | No | `lockInterruptibly()` | N/A |
| Condition | `wait()/notify()` | Multiple `Condition` objects | N/A |
| Performance | Good (JVM optimized) | Slightly better under contention | Best (no blocking) |

```java
// CAS with AtomicReference — lock-free payment status update
public class PaymentStateMachine {
    private final AtomicReference<PaymentStatus> status = new AtomicReference<>(PaymentStatus.INITIATED);
    
    public boolean transition(PaymentStatus expected, PaymentStatus newStatus) {
        return status.compareAndSet(expected, newStatus);
        // Returns false if another thread already changed the status
    }
}

// ReentrantLock — when you need tryLock for deadlock avoidance
public class AccountTransfer {
    public boolean transfer(Account from, Account to, BigDecimal amount) {
        boolean fromLocked = false, toLocked = false;
        try {
            fromLocked = from.getLock().tryLock(100, TimeUnit.MILLISECONDS);
            toLocked = to.getLock().tryLock(100, TimeUnit.MILLISECONDS);
            if (fromLocked && toLocked) {
                from.debit(amount);
                to.credit(amount);
                return true;
            }
            return false; // Retry later
        } finally {
            if (fromLocked) from.getLock().unlock();
            if (toLocked) to.getLock().unlock();
        }
    }
}
```

**When to use:**
- **synchronized**: Simple mutual exclusion, no need for advanced features
- **ReentrantLock**: Need tryLock, fairness, multiple conditions, or interruptible locking
- **CAS (Atomic*)**: High-contention counters, status flags, lock-free data structures

---

### 🔥 Q12. How does ConcurrentHashMap work internally?

**Answer:**

**Java 8+ Implementation:**
- Uses **array of Nodes** (bins) + **CAS + synchronized** (per-bin locking)
- No more Segment-based locking (pre-Java 8)
- When a bin exceeds **8 entries**, it converts from linked list to **red-black tree** (treeification)
- When it drops below **6**, it converts back to linked list

```
ConcurrentHashMap Internal Structure (Java 8+):
┌─────┬─────┬─────┬─────┬─────┬─────┐
│ [0] │ [1] │ [2] │ [3] │ ... │ [n] │  ← Node array (table)
└──┬──┴──┬──┴─────┴──┬──┴─────┴─────┘
   │     │           │
   ▼     ▼           ▼
  Node  Node        TreeBin (if > 8 entries)
   │     │           │
   ▼     ▼           ▼
  Node  null      TreeNode ←→ TreeNode
   │
   ▼
  null
```

**Key Operations:**
- **put()**: CAS for empty bin, synchronized on first node for non-empty bin
- **get()**: No locking (volatile reads)
- **size()**: Uses `baseCount` + `CounterCell[]` (striped counters, like LongAdder)

```java
// Production: Thread-safe idempotency cache
private final ConcurrentHashMap<String, PaymentResult> idempotencyCache = new ConcurrentHashMap<>();

public PaymentResult processIdempotent(String idempotencyKey, Supplier<PaymentResult> processor) {
    return idempotencyCache.computeIfAbsent(idempotencyKey, key -> {
        // This lambda executes atomically for this key
        return processor.get();
    });
}
```

---

### 🔥 Q13. How does HashMap work internally? What changed in Java 8?

**Answer:**

**Structure**: Array of buckets. Each bucket is a linked list (or tree in Java 8+).

**put() flow:**
1. Calculate `hash(key)` → `hash ^ (hash >>> 16)` (spread high bits)
2. Find bucket index: `(n - 1) & hash`
3. If bucket empty → insert new Node
4. If bucket occupied → traverse linked list/tree
   - If key exists (equals + hashCode match) → update value
   - If key doesn't exist → append to end (Java 8) / insert at head (Java 7)
5. If linked list length > 8 AND table size >= 64 → **treeify** (convert to red-black tree)
6. If size > threshold (capacity × load factor 0.75) → **resize** (double capacity)

**Java 8 Changes:**
| Aspect | Java 7 | Java 8 |
|--------|--------|--------|
| Collision handling | Linked list only | Linked list → Red-black tree (>8) |
| Insertion | Head insertion | Tail insertion |
| Resize | Can cause infinite loop (multi-threaded) | Tail insertion prevents this |
| Hash function | Multiple shifts + XORs | Single `hash ^ (hash >>> 16)` |
| Worst-case get() | O(n) | O(log n) with tree |

---

### 💎 Q14. Explain ThreadLocal. What are the pitfalls with thread pools?

**Answer:**

ThreadLocal provides **thread-confined** variables — each thread has its own independent copy.

```java
// Common use: Request context propagation
public class RequestContext {
    private static final ThreadLocal<String> correlationId = new ThreadLocal<>();
    private static final ThreadLocal<String> userId = new ThreadLocal<>();
    
    public static void setCorrelationId(String id) { correlationId.set(id); }
    public static String getCorrelationId() { return correlationId.get(); }
    
    public static void clear() {
        correlationId.remove();
        userId.remove();
    }
}

// In a filter/interceptor
public class CorrelationFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
        try {
            String corrId = Optional.ofNullable(((HttpServletRequest) req).getHeader("X-Correlation-ID"))
                .orElse(UUID.randomUUID().toString());
            RequestContext.setCorrelationId(corrId);
            chain.doFilter(req, res);
        } finally {
            RequestContext.clear(); // ✅ CRITICAL: Always clean up
        }
    }
}
```

**Pitfalls with Thread Pools:**
1. **Memory Leak**: Thread pool reuses threads → ThreadLocal values persist across requests
2. **Data Leakage**: Previous request's data visible to next request on same thread
3. **Virtual Threads**: Millions of virtual threads × ThreadLocal = massive memory usage

**Solutions:**
- Always call `remove()` in `finally` block
- Use `InheritableThreadLocal` for parent-child thread propagation
- Java 21: Use `ScopedValue` instead (structured, auto-cleanup)

```java
// Java 21: ScopedValue (preferred over ThreadLocal)
private static final ScopedValue<String> CORRELATION_ID = ScopedValue.newInstance();

ScopedValue.where(CORRELATION_ID, "txn-12345").run(() -> {
    // CORRELATION_ID is available here and in all called methods
    processPayment();
    // Automatically cleaned up when scope exits
});
```

---

### 💎 Q15. Explain ExecutorService types and when to use each.

**Answer:**

| Executor | Pool Size | Queue | Use Case |
|----------|-----------|-------|----------|
| `newFixedThreadPool(n)` | Fixed n | Unbounded LinkedBlockingQueue | Known concurrency level |
| `newCachedThreadPool()` | 0 → Integer.MAX | SynchronousQueue | Short-lived burst tasks |
| `newSingleThreadExecutor()` | 1 | Unbounded LinkedBlockingQueue | Sequential task execution |
| `newScheduledThreadPool(n)` | Fixed n | DelayedWorkQueue | Periodic/delayed tasks |
| `newVirtualThreadPerTaskExecutor()` | Unlimited virtual | N/A | I/O-bound tasks (Java 21) |
| `new ThreadPoolExecutor(...)` | Custom | Custom | Production (full control) |

```java
// Production: Custom ThreadPoolExecutor for payment processing
ThreadPoolExecutor paymentExecutor = new ThreadPoolExecutor(
    10,                          // core pool size
    50,                          // max pool size
    60L, TimeUnit.SECONDS,       // keep-alive for idle threads
    new ArrayBlockingQueue<>(100), // bounded queue (backpressure!)
    new ThreadFactory() {
        private final AtomicInteger counter = new AtomicInteger(0);
        @Override
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r, "payment-worker-" + counter.incrementAndGet());
            t.setDaemon(false);
            t.setUncaughtExceptionHandler((thread, ex) -> 
                log.error("Uncaught exception in {}", thread.getName(), ex));
            return t;
        }
    },
    new ThreadPoolExecutor.CallerRunsPolicy() // Backpressure: caller thread executes
);
```

**⚠️ Never use unbounded queues in production** — they can cause OOM. Always use `ArrayBlockingQueue` with a rejection policy.

---

## Section 5: Collections Internals

### 🔥 Q16. What is the difference between HashMap, LinkedHashMap, TreeMap, and ConcurrentHashMap?

**Answer:**

| Feature | HashMap | LinkedHashMap | TreeMap | ConcurrentHashMap |
|---------|---------|---------------|---------|-------------------|
| Order | No guarantee | Insertion order | Sorted (natural/comparator) | No guarantee |
| Null keys | 1 allowed | 1 allowed | Not allowed | Not allowed |
| Null values | Allowed | Allowed | Allowed | Not allowed |
| Thread-safe | No | No | No | Yes |
| Time complexity | O(1) avg | O(1) avg | O(log n) | O(1) avg |
| Use case | General purpose | LRU cache, ordered iteration | Range queries, sorted data | Concurrent access |

```java
// LinkedHashMap as LRU Cache
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxSize;
    
    public LRUCache(int maxSize) {
        super(maxSize, 0.75f, true); // true = access-order
        this.maxSize = maxSize;
    }
    
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > maxSize;
    }
}
```

---

### 💎 Q17. How does the equals() and hashCode() contract work? What happens if you break it?

**Answer:**

**Contract:**
1. If `a.equals(b)` is true → `a.hashCode() == b.hashCode()` MUST be true
2. If `a.hashCode() == b.hashCode()` → `a.equals(b)` may or may not be true (collision)
3. `equals()` must be reflexive, symmetric, transitive, consistent

**What happens if you break it:**

```java
// ❌ BROKEN: Override equals but not hashCode
public class PaymentKey {
    private String txnId;
    private String merchantId;
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof PaymentKey)) return false;
        PaymentKey that = (PaymentKey) o;
        return txnId.equals(that.txnId) && merchantId.equals(that.merchantId);
    }
    // hashCode NOT overridden — uses Object.hashCode() (memory address)
}

Map<PaymentKey, Payment> map = new HashMap<>();
PaymentKey key1 = new PaymentKey("TXN-001", "M-100");
map.put(key1, payment);

PaymentKey key2 = new PaymentKey("TXN-001", "M-100"); // Same logical key
Payment result = map.get(key2); // ❌ Returns NULL! Different hashCode → different bucket
```

**✅ Correct Implementation:**
```java
@Override
public int hashCode() {
    return Objects.hash(txnId, merchantId);
}
```

---

## Quick Revision Checklist

- [ ] Functional Interfaces: Predicate, Function, Consumer, Supplier
- [ ] Streams: Lazy evaluation, intermediate vs terminal, collectors
- [ ] Optional: map vs flatMap, orElseThrow, never use as field
- [ ] CompletableFuture: allOf, thenCompose, exceptionally
- [ ] Virtual Threads: Lightweight, don't pool, avoid synchronized
- [ ] Records: Immutable, compact constructor, auto equals/hashCode
- [ ] Sealed Classes + Pattern Matching: Exhaustive switch
- [ ] JMM: Heap (Young/Old), Metaspace, Thread Stack
- [ ] GC: G1 (default), ZGC (low-latency payments)
- [ ] Memory Leak: Heap dump → MAT → Dominator tree
- [ ] synchronized vs ReentrantLock vs CAS
- [ ] ConcurrentHashMap: CAS + per-bin sync, treeification at 8
- [ ] HashMap: Array + LinkedList/Tree, load factor 0.75, resize at threshold
- [ ] ThreadLocal: Always remove() in finally, use ScopedValue in Java 21
- [ ] ExecutorService: Bounded queue + rejection policy in production
