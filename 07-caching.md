# 07 — Caching (Redis, Caffeine, Cache Patterns, Distributed Caching)

> **Priority**: 🟡 High  
> **Estimated Study Time**: 1 day  
> Markers: 🔥 = Frequently Asked | 💎 = Differentiator

---

## Section 1: Caching Patterns

### 🔥 Q1. Explain Cache-aside vs Write-through vs Write-behind vs Read-through.

**Answer:**

**1. Cache-Aside (Lazy Loading)** — Most common

```
Read:                              Write:
App ──→ Cache hit? ──→ Return      App ──→ Write to DB
  │         │                        │
  │    Cache miss                    └──→ Invalidate/Delete cache
  │         │
  └──→ Read from DB
  └──→ Put in cache
  └──→ Return
```

```java
// Cache-Aside Implementation
@Service
public class MerchantService {
    
    @Cacheable(value = "merchants", key = "#merchantId", unless = "#result == null")
    public Merchant getMerchant(String merchantId) {
        // Called only on cache miss
        return merchantRepository.findById(merchantId).orElse(null);
    }
    
    @CacheEvict(value = "merchants", key = "#merchant.id")
    public Merchant updateMerchant(Merchant merchant) {
        return merchantRepository.save(merchant);
    }
}
```

| Pros | Cons |
|------|------|
| Simple, widely used | Cache miss penalty (extra DB call) |
| Only caches what's needed | Stale data possible |
| Cache failure doesn't break reads | Application manages cache logic |

**2. Read-Through** — Cache loads data itself

```
App ──→ Cache ──→ Cache hit? ──→ Return
                      │
                 Cache miss
                      │
                 Cache loads from DB (cache manages this)
                      │
                 Return
```

**3. Write-Through** — Write to cache AND DB synchronously

```
App ──→ Write to Cache ──→ Cache writes to DB ──→ Return
```

| Pros | Cons |
|------|------|
| Cache always consistent with DB | Higher write latency (2 writes) |
| No stale data | Writes to cache for data that may never be read |

**4. Write-Behind (Write-Back)** — Write to cache, async write to DB

```
App ──→ Write to Cache ──→ Return immediately
              │
              └──→ Async batch write to DB (later)
```

| Pros | Cons |
|------|------|
| Very fast writes | Data loss risk if cache crashes before DB write |
| Batch DB writes (efficient) | Complex, eventual consistency |

**Comparison:**

| Pattern | Read Perf | Write Perf | Consistency | Complexity | Best For |
|---------|-----------|------------|-------------|------------|----------|
| Cache-Aside | Good (after warm) | Good | Eventual | Low | General purpose |
| Read-Through | Good | N/A | Eventual | Medium | Read-heavy |
| Write-Through | Good | Slower | Strong | Medium | Consistency critical |
| Write-Behind | Good | Fastest | Eventual | High | Write-heavy |

**For Payment Service:**
- **Merchant data** → Cache-Aside (read-heavy, rarely changes)
- **Exchange rates** → Read-Through with TTL (refresh periodically)
- **Session data** → Write-Through (consistency needed)
- **Analytics counters** → Write-Behind (high write volume)

---

### 🔥 Q2. What is cache stampede (thundering herd)? How do you prevent it?

**Answer:**

**Cache Stampede**: When a popular cache key expires, hundreds of concurrent requests all miss the cache simultaneously and hit the database.

```
Time T: Cache key "popular-merchant" expires
  
Thread 1 ──→ Cache MISS ──→ Query DB ──→ (slow)
Thread 2 ──→ Cache MISS ──→ Query DB ──→ (slow)
Thread 3 ──→ Cache MISS ──→ Query DB ──→ (slow)
...
Thread 100 ──→ Cache MISS ──→ Query DB ──→ 💥 DB overwhelmed!
```

**Solutions:**

**1. Locking (Mutex)**
```java
// Only one thread fetches from DB, others wait
public Merchant getMerchantWithLock(String merchantId) {
    String cacheKey = "merchant:" + merchantId;
    Merchant cached = redisTemplate.opsForValue().get(cacheKey);
    
    if (cached != null) return cached;
    
    String lockKey = "lock:" + cacheKey;
    boolean locked = redisTemplate.opsForValue()
        .setIfAbsent(lockKey, "1", Duration.ofSeconds(10));
    
    if (locked) {
        try {
            // Double-check after acquiring lock
            cached = redisTemplate.opsForValue().get(cacheKey);
            if (cached != null) return cached;
            
            Merchant merchant = merchantRepository.findById(merchantId).orElseThrow();
            redisTemplate.opsForValue().set(cacheKey, merchant, Duration.ofHours(1));
            return merchant;
        } finally {
            redisTemplate.delete(lockKey);
        }
    } else {
        // Wait and retry
        Thread.sleep(50);
        return getMerchantWithLock(merchantId);
    }
}
```

**2. Stale-While-Revalidate (Background Refresh)**
```java
// Caffeine cache with async refresh
@Bean
public Cache<String, Merchant> merchantCache() {
    return Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(Duration.ofMinutes(30))
        .refreshAfterWrite(Duration.ofMinutes(25)) // Refresh 5 min before expiry
        .build(merchantId -> merchantRepository.findById(merchantId).orElse(null));
    // When refresh triggers, serves stale data while fetching new data in background
}
```

**3. Probabilistic Early Expiration (Jitter)**
```java
// Add random jitter to TTL to prevent all keys expiring at once
Duration baseTtl = Duration.ofHours(1);
Duration jitter = Duration.ofMinutes(ThreadLocalRandom.current().nextInt(0, 10));
Duration ttl = baseTtl.plus(jitter); // 60-70 minutes

redisTemplate.opsForValue().set(key, value, ttl);
```

**4. Pre-warming Cache**
```java
// Warm cache on application startup
@EventListener(ApplicationReadyEvent.class)
public void warmCache() {
    List<String> topMerchants = merchantRepository.findTopMerchantIds(1000);
    topMerchants.parallelStream().forEach(id -> {
        Merchant m = merchantRepository.findById(id).orElse(null);
        if (m != null) {
            redisTemplate.opsForValue().set("merchant:" + id, m, Duration.ofHours(1));
        }
    });
    log.info("Cache warmed with {} merchants", topMerchants.size());
}
```

---

## Section 2: Redis

### 🔥 Q3. Explain Redis data structures and their use cases in a payment system.

**Answer:**

| Data Structure | Commands | Use Case in Payments |
|---------------|----------|---------------------|
| **String** | GET, SET, INCR, SETNX | Idempotency keys, session tokens, rate limit counters |
| **Hash** | HGET, HSET, HGETALL | Payment details, user profiles |
| **List** | LPUSH, RPOP, LRANGE | Recent transactions, notification queue |
| **Set** | SADD, SMEMBERS, SISMEMBER | Unique merchant IDs, blocked cards |
| **Sorted Set** | ZADD, ZRANGEBYSCORE | Leaderboards, scheduled retries, time-based expiry |
| **Stream** | XADD, XREAD, XREADGROUP | Event streaming, audit log |
| **HyperLogLog** | PFADD, PFCOUNT | Unique visitor count (approximate) |

```java
// Redis in Payment Service — Spring Data Redis

// 1. Idempotency Key (String with TTL)
public boolean checkAndSetIdempotencyKey(String key) {
    Boolean isNew = redisTemplate.opsForValue()
        .setIfAbsent("idem:" + key, "1", Duration.ofHours(24));
    return Boolean.TRUE.equals(isNew); // true = new request, false = duplicate
}

// 2. Rate Limiting (String with INCR)
public boolean isRateLimited(String merchantId) {
    String key = "rate:" + merchantId + ":" + Instant.now().getEpochSecond();
    Long count = redisTemplate.opsForValue().increment(key);
    if (count == 1) {
        redisTemplate.expire(key, Duration.ofSeconds(2)); // TTL
    }
    return count > 100; // 100 requests per second
}

// 3. Payment Cache (Hash)
public void cachePayment(Payment payment) {
    String key = "payment:" + payment.getTxnId();
    Map<String, String> hash = Map.of(
        "txnId", payment.getTxnId(),
        "amount", payment.getAmount().toString(),
        "status", payment.getStatus().name(),
        "merchantId", payment.getMerchantId()
    );
    redisTemplate.opsForHash().putAll(key, hash);
    redisTemplate.expire(key, Duration.ofHours(2));
}

// 4. Blocked Cards (Set)
public boolean isCardBlocked(String cardHash) {
    return Boolean.TRUE.equals(
        redisTemplate.opsForSet().isMember("blocked-cards", cardHash));
}

// 5. Scheduled Retries (Sorted Set — score = retry timestamp)
public void scheduleRetry(String txnId, Instant retryAt) {
    redisTemplate.opsForZSet().add("retry-queue", txnId, retryAt.toEpochMilli());
}

public List<String> getDueRetries() {
    double now = Instant.now().toEpochMilli();
    Set<String> txnIds = redisTemplate.opsForZSet().rangeByScore("retry-queue", 0, now);
    // Remove from sorted set after fetching
    if (txnIds != null && !txnIds.isEmpty()) {
        redisTemplate.opsForZSet().removeRangeByScore("retry-queue", 0, now);
    }
    return txnIds != null ? new ArrayList<>(txnIds) : List.of();
}
```

---

### 🔥 Q4. Redis persistence: RDB vs AOF. Which one for payments?

**Answer:**

| Feature | RDB (Snapshotting) | AOF (Append-Only File) |
|---------|-------------------|----------------------|
| How | Point-in-time snapshots | Logs every write operation |
| Data loss | Up to last snapshot interval | Minimal (configurable: every sec / every write) |
| Recovery speed | Fast (load snapshot) | Slower (replay all commands) |
| File size | Compact | Larger (can be compacted with rewrite) |
| Performance | Better (no per-write I/O) | Slightly slower |

**For Payments — Use Both (RDB + AOF):**
```
# redis.conf
# RDB snapshots
save 900 1      # Snapshot if 1 key changed in 900 seconds
save 300 10     # Snapshot if 10 keys changed in 300 seconds
save 60 10000   # Snapshot if 10000 keys changed in 60 seconds

# AOF for durability
appendonly yes
appendfsync everysec  # Fsync every second (good balance)
# appendfsync always  # Fsync every write (safest, slowest)

# AOF rewrite to compact
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

**Redis Cluster for High Availability:**
```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ Master 1     │    │ Master 2     │    │ Master 3     │
│ Slots 0-5460 │    │ Slots 5461-  │    │ Slots 10923- │
│              │    │ 10922        │    │ 16383        │
└──────┬──────┘    └──────┬──────┘    └──────┬──────┘
       │                  │                  │
┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐
│ Replica 1    │    │ Replica 2    │    │ Replica 3    │
└─────────────┘    └─────────────┘    └─────────────┘
```

---

## Section 3: Caffeine (Local Cache)

### 🔥 Q5. When would you use Caffeine (local cache) vs Redis (distributed cache)?

**Answer:**

| Aspect | Caffeine (Local) | Redis (Distributed) |
|--------|-----------------|---------------------|
| Location | In-process (JVM heap) | External server |
| Latency | ~nanoseconds | ~1-5 milliseconds |
| Shared | No (per instance) | Yes (all instances) |
| Capacity | Limited by JVM heap | Large (dedicated server) |
| Consistency | Per-instance only | Shared across instances |
| Failure impact | None (in-process) | Network issues, Redis down |
| Best for | Hot data, config, reference data | Session, shared state, large datasets |

**Two-Level Cache (L1 + L2):**

```
Request ──→ L1 (Caffeine, in-process) ──→ HIT → Return
                    │
               L1 MISS
                    │
                    ▼
            L2 (Redis, distributed) ──→ HIT → Put in L1, Return
                    │
               L2 MISS
                    │
                    ▼
            Database ──→ Put in L2 ──→ Put in L1 ──→ Return
```

```java
// Two-Level Cache Implementation
@Service
public class MerchantCacheService {
    
    private final Cache<String, Merchant> localCache; // Caffeine L1
    private final RedisTemplate<String, Merchant> redisTemplate; // Redis L2
    private final MerchantRepository repository;
    
    public MerchantCacheService(RedisTemplate<String, Merchant> redisTemplate,
                                 MerchantRepository repository) {
        this.redisTemplate = redisTemplate;
        this.repository = repository;
        this.localCache = Caffeine.newBuilder()
            .maximumSize(1_000)
            .expireAfterWrite(Duration.ofMinutes(5)) // Short TTL for L1
            .recordStats()
            .build();
    }
    
    public Merchant getMerchant(String merchantId) {
        // L1: Check local cache
        Merchant cached = localCache.getIfPresent(merchantId);
        if (cached != null) return cached;
        
        // L2: Check Redis
        cached = redisTemplate.opsForValue().get("merchant:" + merchantId);
        if (cached != null) {
            localCache.put(merchantId, cached); // Promote to L1
            return cached;
        }
        
        // DB: Load from database
        Merchant merchant = repository.findById(merchantId).orElseThrow();
        redisTemplate.opsForValue().set("merchant:" + merchantId, merchant, Duration.ofHours(1));
        localCache.put(merchantId, merchant);
        return merchant;
    }
    
    // Invalidate both levels
    public void evict(String merchantId) {
        localCache.invalidate(merchantId);
        redisTemplate.delete("merchant:" + merchantId);
        // Publish invalidation event for other instances
        redisTemplate.convertAndSend("cache-invalidation", merchantId);
    }
}
```

**Caffeine Configuration:**
```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CaffeineCacheManager cacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        manager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofMinutes(10))
            .expireAfterAccess(Duration.ofMinutes(5))
            .recordStats() // Enable stats for monitoring
        );
        return manager;
    }
}
```

---

## Section 4: Cache Invalidation

### 💎 Q6. How do you handle cache invalidation in a distributed system?

**Answer:**

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

**Strategies:**

| Strategy | How | Consistency | Complexity |
|----------|-----|-------------|------------|
| **TTL-based** | Cache expires after time | Eventual (stale for TTL duration) | Low |
| **Event-based** | Publish invalidation event on write | Near real-time | Medium |
| **Write-through** | Update cache on every write | Strong | Medium |
| **Version-based** | Include version in cache key | Strong | Medium |

**Event-Based Invalidation (Recommended for Microservices):**

```java
// When merchant data is updated
@Service
public class MerchantService {
    
    @Transactional
    public Merchant updateMerchant(Merchant merchant) {
        Merchant saved = merchantRepository.save(merchant);
        
        // Publish invalidation event
        kafkaTemplate.send("cache-invalidation", 
            new CacheInvalidationEvent("merchant", merchant.getId()));
        
        return saved;
    }
}

// All instances listen for invalidation events
@KafkaListener(topics = "cache-invalidation", groupId = "${spring.application.name}-${random.uuid}")
public void handleCacheInvalidation(CacheInvalidationEvent event) {
    switch (event.entityType()) {
        case "merchant" -> {
            localCache.invalidate(event.entityId());
            redisTemplate.delete("merchant:" + event.entityId());
        }
        case "exchange-rate" -> {
            localCache.invalidate("rate:" + event.entityId());
        }
    }
    log.info("Cache invalidated: type={}, id={}", event.entityType(), event.entityId());
}
```

**Redis Pub/Sub for Local Cache Invalidation:**
```java
// Publish invalidation to all instances via Redis Pub/Sub
@Component
public class CacheInvalidationListener implements MessageListener {
    
    @Autowired
    private Cache<String, Object> localCache;
    
    @Override
    public void onMessage(Message message, byte[] pattern) {
        String key = new String(message.getBody());
        localCache.invalidate(key);
        log.debug("Local cache invalidated for key: {}", key);
    }
}

@Configuration
public class RedisSubscriberConfig {
    @Bean
    public RedisMessageListenerContainer container(RedisConnectionFactory factory,
                                                    CacheInvalidationListener listener) {
        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(factory);
        container.addMessageListener(listener, new ChannelTopic("cache-invalidation"));
        return container;
    }
}
```

---

## Section 5: Common Cache Problems

### 💎 Q7. Explain cache penetration, cache breakdown, and cache avalanche.

**Answer:**

| Problem | Description | Solution |
|---------|-------------|----------|
| **Cache Penetration** | Queries for non-existent data bypass cache every time | Cache null values, Bloom filter |
| **Cache Breakdown** | Hot key expires, all requests hit DB | Mutex lock, never-expire + background refresh |
| **Cache Avalanche** | Many keys expire simultaneously | Jittered TTL, multi-level cache, circuit breaker |

**Cache Penetration:**
```java
// Problem: Attacker queries non-existent IDs → always misses cache → hammers DB
// GET /merchants/non-existent-id → cache miss → DB query → null → not cached → repeat

// Solution 1: Cache null values
@Cacheable(value = "merchants", key = "#id", unless = "false") // Cache even null
public Merchant getMerchant(String id) {
    return merchantRepository.findById(id).orElse(null);
    // null is cached with short TTL
}

// Solution 2: Bloom Filter (probabilistic, memory-efficient)
@Component
public class MerchantBloomFilter {
    private final BloomFilter<String> filter;
    
    @PostConstruct
    public void init() {
        filter = BloomFilter.create(Funnels.stringFunnel(Charset.defaultCharset()), 
            1_000_000, 0.01); // 1M entries, 1% false positive rate
        merchantRepository.findAllIds().forEach(filter::put);
    }
    
    public boolean mightExist(String merchantId) {
        return filter.mightContain(merchantId);
        // If returns false → definitely doesn't exist (skip DB query)
        // If returns true → might exist (proceed with normal flow)
    }
}
```

**Cache Avalanche Prevention:**
```java
// Add jitter to prevent mass expiration
public Duration jitteredTtl(Duration baseTtl) {
    long jitterMs = ThreadLocalRandom.current().nextLong(0, baseTtl.toMillis() / 10);
    return baseTtl.plusMillis(jitterMs);
}

// Pre-warm critical data before peak hours
@Scheduled(cron = "0 0 8 * * *") // 8 AM daily
public void prewarmCache() {
    List<String> hotMerchants = analyticsService.getTopMerchants(1000);
    hotMerchants.forEach(id -> getMerchant(id)); // Warm cache
}
```

---

## Quick Revision Checklist

- [ ] Cache-Aside: App manages cache, most common pattern
- [ ] Write-Through: Cache + DB sync write, strong consistency
- [ ] Write-Behind: Cache write, async DB write, fast but risky
- [ ] Cache Stampede: Mutex lock, stale-while-revalidate, jittered TTL
- [ ] Redis Data Structures: String (counters), Hash (objects), Set (membership), Sorted Set (scheduling)
- [ ] Redis Persistence: RDB (snapshots) + AOF (append log) for payments
- [ ] Caffeine vs Redis: Local (ns latency) vs Distributed (ms latency, shared)
- [ ] Two-Level Cache: L1 Caffeine → L2 Redis → DB
- [ ] Cache Invalidation: TTL, event-based (Kafka), Redis Pub/Sub
- [ ] Cache Penetration: Bloom filter, cache null values
- [ ] Cache Breakdown: Mutex, never-expire + background refresh
- [ ] Cache Avalanche: Jittered TTL, pre-warming, circuit breaker
