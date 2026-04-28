# 10 — Production Scenarios (OOM Debugging, Kafka Lag, DB Spikes, Thread Dumps, Incident Response)

> **Priority**: 🔴 Critical  
> **Estimated Study Time**: 1 day  
> Markers: 🔥 = Frequently Asked | 💎 = Differentiator

---

## Section 1: OOM Debugging

### 🔥 Q1. How would you debug an OutOfMemoryError in production?

**Answer:**

**Step-by-step approach:**

```
1. DETECT → Alert fires (heap usage > 90%, OOM error in logs)
2. STABILIZE → Restart affected instance, traffic shifts to healthy nodes
3. CAPTURE → Get heap dump (auto or manual)
4. ANALYZE → Eclipse MAT / VisualVM / Dynatrace
5. FIX → Identify leak, deploy fix
6. PREVENT → Add monitoring, load test
```

**Types of OOM:**

| Error | Cause | Fix |
|-------|-------|-----|
| `java.lang.OutOfMemoryError: Java heap space` | Heap full (objects) | Increase heap or fix leak |
| `java.lang.OutOfMemoryError: Metaspace` | Too many classes loaded | Increase Metaspace, check classloader leak |
| `java.lang.OutOfMemoryError: GC overhead limit exceeded` | GC using >98% CPU, recovering <2% heap | Fix memory leak |
| `java.lang.OutOfMemoryError: Direct buffer memory` | NIO direct buffers exhausted | Increase `-XX:MaxDirectMemorySize` |
| `java.lang.OutOfMemoryError: unable to create new native thread` | Too many threads | Reduce thread count, increase ulimit |

**Capturing Heap Dump:**
```bash
# Auto-capture on OOM (set in JVM flags — ALWAYS enable in production)
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/logs/heapdumps/

# Manual capture (live = trigger GC first)
jmap -dump:live,format=b,file=heap_$(date +%Y%m%d_%H%M%S).hprof <PID>

# Using jcmd (preferred, safer)
jcmd <PID> GC.heap_dump /var/logs/heapdumps/heap.hprof
```

**Analyzing with Eclipse MAT:**
```
1. Open heap dump in Eclipse MAT
2. Run "Leak Suspects Report" → auto-detects likely leaks
3. Check "Dominator Tree" → which objects retain the most memory
4. Check "Histogram" → count and size of each class
5. Look for:
   - Unexpectedly large collections (HashMap with millions of entries)
   - Byte arrays (often from serialization or caching)
   - String arrays (often from logging or string concatenation)
   - Connection objects (unclosed connections)
```

**Common OOM Causes in Payment Services:**

| Cause | Symptom | Fix |
|-------|---------|-----|
| Unbounded cache | HashMap grows forever | Add max size + TTL (Caffeine/Redis) |
| Connection leak | Connection pool exhausted | Always close in finally/try-with-resources |
| Large query result | Loading millions of rows | Use pagination, streaming |
| ThreadLocal leak | Memory grows with thread pool | Always call `remove()` in finally |
| String concatenation in loop | Massive String objects | Use StringBuilder |
| Kafka consumer batch too large | Large batch in memory | Reduce `max.poll.records` |
| Logging large payloads | Log full request/response bodies | Truncate or log at DEBUG only |

```java
// ❌ Common leak: Unbounded in-memory cache
private static final Map<String, PaymentResult> cache = new HashMap<>(); // Grows forever!

// ✅ Fix: Bounded cache with TTL
private final Cache<String, PaymentResult> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(Duration.ofHours(1))
    .build();

// ❌ Common leak: Large query without pagination
List<Payment> allPayments = paymentRepository.findAll(); // Millions of rows!

// ✅ Fix: Pagination
Page<Payment> page = paymentRepository.findAll(PageRequest.of(0, 100));

// ✅ Fix: Streaming for batch processing
@Transactional(readOnly = true)
public void processAllPayments() {
    try (Stream<Payment> stream = paymentRepository.streamByStatus(PaymentStatus.PENDING)) {
        stream.forEach(this::processPayment);
    }
}
```

---

## Section 2: Kafka Consumer Lag

### 🔥 Q2. How would you handle Kafka consumer lag in production?

**Answer:**

**Detection:**
```bash
# Check consumer lag
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --describe --group payment-processor

# TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# payment-events  0          50000           55000           5000  ← 5K lag
# payment-events  1          48000           55000           7000  ← 7K lag
# payment-events  2          52000           55000           3000  ← 3K lag
```

**Triage Decision Tree:**

```
Consumer Lag Detected
    │
    ├── Is lag growing? ──→ YES ──→ 🔴 CRITICAL (consumers can't keep up)
    │                              │
    │                              ├── Check consumer health
    │                              │   ├── Are consumers running? (pod status)
    │                              │   ├── Are consumers in rebalancing loop?
    │                              │   └── Check consumer error logs
    │                              │
    │                              ├── Check processing time
    │                              │   ├── Is each message taking too long?
    │                              │   ├── Is external dependency slow? (DB, API)
    │                              │   └── Is there a poison pill message?
    │                              │
    │                              └── Scale up
    │                                  ├── Increase consumer instances (up to partition count)
    │                                  ├── Increase concurrency per consumer
    │                                  └── Enable batch processing
    │
    └── Is lag stable? ──→ YES ──→ 🟡 WARNING (consumers keeping up but behind)
                                   │
                                   └── Likely caused by a burst
                                       └── Will catch up naturally
```

**Immediate Actions:**

```java
// 1. Scale consumers (if fewer consumers than partitions)
// Kubernetes: scale deployment
// kubectl scale deployment payment-consumer --replicas=6

// 2. Increase concurrency
@KafkaListener(topics = "payment-events", 
    groupId = "payment-processor",
    concurrency = "6") // Increase from 3 to 6
public void process(PaymentEvent event, Acknowledgment ack) { ... }

// 3. Switch to batch processing
@KafkaListener(topics = "payment-events", groupId = "payment-processor")
public void processBatch(List<ConsumerRecord<String, PaymentEvent>> records, Acknowledgment ack) {
    // Process in parallel
    List<CompletableFuture<Void>> futures = records.stream()
        .map(record -> CompletableFuture.runAsync(() -> 
            processPayment(record.value()), processingExecutor))
        .toList();
    
    CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
    ack.acknowledge();
}

// 4. Skip/DLQ poison pill messages
@KafkaListener(topics = "payment-events", groupId = "payment-processor")
public void process(ConsumerRecord<String, PaymentEvent> record, Acknowledgment ack) {
    try {
        paymentService.process(record.value());
        ack.acknowledge();
    } catch (NonRetryableException e) {
        log.error("Poison pill detected, sending to DLQ: offset={}", record.offset(), e);
        dlqProducer.send("payment-events.DLT", record.key(), record.value());
        ack.acknowledge(); // Skip this message
    }
}
```

---

## Section 3: Database CPU Spike

### 🔥 Q3. How would you debug a sudden database CPU spike?

**Answer:**

**Step-by-step:**

```
1. DETECT → DB CPU alert fires (> 80% for 5 minutes)
2. IDENTIFY → Find the expensive queries
3. ANALYZE → Check execution plans
4. FIX → Add index / optimize query / kill session
5. PREVENT → Add monitoring, query review in PR
```

```sql
-- Step 1: Find top CPU-consuming sessions
SELECT s.sid, s.serial#, s.username, s.program, s.status,
       q.sql_text, q.elapsed_time/1000000 as elapsed_secs
FROM v$session s
JOIN v$sql q ON s.sql_id = q.sql_id
WHERE s.status = 'ACTIVE'
AND s.type = 'USER'
ORDER BY q.elapsed_time DESC
FETCH FIRST 10 ROWS ONLY;

-- Step 2: Find queries with most CPU time
SELECT sql_id, 
       cpu_time/1000000 as cpu_secs,
       elapsed_time/1000000 as elapsed_secs,
       executions,
       buffer_gets/GREATEST(executions,1) as avg_buffer_gets,
       SUBSTR(sql_text, 1, 200) as sql_preview
FROM v$sql
WHERE cpu_time > 1000000  -- > 1 second CPU
ORDER BY cpu_time DESC
FETCH FIRST 10 ROWS ONLY;

-- Step 3: Check for full table scans
SELECT sql_id, plan_hash_value, operation, options, object_name
FROM v$sql_plan
WHERE operation = 'TABLE ACCESS' AND options = 'FULL'
AND sql_id IN (SELECT sql_id FROM v$sql WHERE cpu_time > 1000000);

-- Step 4: Kill a runaway session (emergency)
ALTER SYSTEM KILL SESSION 'sid,serial#' IMMEDIATE;
```

**Common Causes & Fixes:**

| Cause | How to Identify | Fix |
|-------|----------------|-----|
| Missing index | Full table scan in plan | Create index |
| Stale statistics | Wrong cardinality estimates | `DBMS_STATS.GATHER_TABLE_STATS` |
| Lock contention | `v$lock` shows waiting sessions | Optimize transactions, reduce lock time |
| Cartesian join | Massive row count in plan | Fix JOIN conditions |
| Runaway query | Single session consuming all CPU | Kill session, add timeout |
| Sudden traffic spike | All queries slow | Scale read replicas, add caching |

```java
// Prevention: Query timeout in Spring Boot
spring:
  jpa:
    properties:
      javax.persistence.query.timeout: 5000  # 5 second query timeout
      hibernate:
        session.events.log.LOG_QUERIES_SLOWER_THAN_MS: 1000  # Log slow queries
```

---

## Section 4: Duplicate Payment Charges

### 🔥 Q4. How would you handle duplicate payment charges in production?

**Answer:**

**Scenario**: Customer charged twice for the same order.

**Investigation:**

```
1. VERIFY → Confirm duplicate in DB and gateway
2. ROOT CAUSE → Why did it happen?
3. REMEDIATE → Refund the duplicate
4. PREVENT → Fix the root cause
```

```java
// Investigation query
@Query("""
    SELECT p FROM Payment p 
    WHERE p.orderId = :orderId 
    AND p.status IN ('CAPTURED', 'SETTLED')
    ORDER BY p.createdAt
    """)
List<Payment> findCompletedPaymentsByOrder(@Param("orderId") String orderId);

// If multiple results → duplicate detected!
```

**Common Root Causes:**

| Cause | How It Happens | Prevention |
|-------|---------------|-----------|
| Missing idempotency | Retry creates new payment | Idempotency key on every request |
| Race condition | Two threads process same request | Distributed lock / DB unique constraint |
| Client retry | User clicks "Pay" twice | Disable button after click, idempotency |
| Gateway timeout | Payment succeeded but response lost, retry creates duplicate | Check payment status before retry |
| Webhook duplicate | Gateway sends webhook twice | Idempotent webhook handler |

**Automated Duplicate Detection:**

```java
@Scheduled(cron = "0 */15 * * * *") // Every 15 minutes
public void detectDuplicateCharges() {
    List<DuplicateGroup> duplicates = paymentRepository.findPotentialDuplicates();
    // Query: SELECT order_id, COUNT(*) FROM payments 
    //        WHERE status IN ('CAPTURED','SETTLED') 
    //        AND created_at > SYSDATE - 1
    //        GROUP BY order_id HAVING COUNT(*) > 1
    
    for (DuplicateGroup group : duplicates) {
        log.error("DUPLICATE DETECTED: orderId={}, count={}, txnIds={}", 
            group.getOrderId(), group.getCount(), group.getTxnIds());
        
        // Auto-refund all but the first charge
        List<Payment> payments = group.getPayments();
        payments.stream()
            .skip(1) // Keep the first one
            .forEach(payment -> {
                try {
                    refundService.initiateRefund(payment.getTxnId(), payment.getAmount(), 
                        "Automated duplicate refund");
                    log.info("Auto-refunded duplicate: txnId={}", payment.getTxnId());
                } catch (Exception e) {
                    log.error("Failed to auto-refund: txnId={}", payment.getTxnId(), e);
                    alertService.sendCriticalAlert("Manual refund needed: " + payment.getTxnId());
                }
            });
    }
}
```

---

## Section 5: Transaction Timeouts

### 🔥 Q5. How do you handle transaction timeouts in a payment system?

**Answer:**

**Scenario**: Payment gateway doesn't respond within timeout. Did the payment go through or not?

```
Payment Service                    Gateway
     │                               │
     │  POST /charge                  │
     │──────────────────────────────→│
     │                               │  Processing...
     │                               │  (takes > 30s)
     │  ⏰ TIMEOUT (30s)             │
     │  Connection closed             │
     │                               │  Payment SUCCEEDED ✅
     │                               │  (but we don't know!)
     │                               │
     │  What now? 🤔                  │
     │  - Did it succeed?             │
     │  - Did it fail?                │
     │  - Is it still processing?     │
```

**Handling Strategy:**

```java
@Service
public class PaymentTimeoutHandler {
    
    public PaymentResult handleTimeout(PaymentRequest request, TimeoutException ex) {
        log.warn("Payment timeout for txnId={}, checking status...", request.getTxnId());
        
        // Step 1: Mark as UNCERTAIN
        paymentStateMachine.transition(request.getTxnId(), PaymentStatus.TIMEOUT);
        
        // Step 2: Query gateway for actual status
        try {
            GatewayStatusResponse status = gateway.getPaymentStatus(request.getIdempotencyKey());
            
            switch (status.getStatus()) {
                case "SUCCESS" -> {
                    paymentStateMachine.transition(request.getTxnId(), PaymentStatus.CAPTURED);
                    return PaymentResult.success(status.getGatewayTxnId());
                }
                case "FAILED" -> {
                    paymentStateMachine.transition(request.getTxnId(), PaymentStatus.FAILED);
                    return PaymentResult.failed("Payment declined");
                }
                case "PENDING" -> {
                    // Still processing — schedule a check
                    scheduleStatusCheck(request.getTxnId(), request.getIdempotencyKey());
                    return PaymentResult.pending("Payment is being processed");
                }
                case "NOT_FOUND" -> {
                    // Gateway never received it — safe to retry
                    paymentStateMachine.transition(request.getTxnId(), PaymentStatus.FAILED);
                    return PaymentResult.failed("Payment not processed");
                }
            }
        } catch (Exception e) {
            log.error("Failed to check payment status: txnId={}", request.getTxnId(), e);
            scheduleStatusCheck(request.getTxnId(), request.getIdempotencyKey());
            return PaymentResult.pending("Unable to confirm payment status");
        }
        
        return PaymentResult.pending("Payment status unknown");
    }
    
    // Scheduled status check with exponential backoff
    @Scheduled(fixedDelay = 30000) // Check every 30 seconds
    public void checkPendingPayments() {
        List<Payment> timeoutPayments = paymentRepository
            .findByStatusAndUpdatedAtBefore(PaymentStatus.TIMEOUT, 
                Instant.now().minus(1, ChronoUnit.MINUTES));
        
        for (Payment payment : timeoutPayments) {
            try {
                GatewayStatusResponse status = gateway.getPaymentStatus(payment.getIdempotencyKey());
                resolvePaymentStatus(payment, status);
            } catch (Exception e) {
                log.warn("Still unable to resolve: txnId={}", payment.getTxnId());
                
                // After 24 hours, escalate for manual review
                if (payment.getUpdatedAt().isBefore(Instant.now().minus(24, ChronoUnit.HOURS))) {
                    alertService.sendCriticalAlert("Unresolved payment timeout: " + payment.getTxnId());
                }
            }
        }
    }
}
```

---

## Section 6: Thread Dumps & Heap Dumps

### 💎 Q6. How do you take and analyze thread dumps? When would you need one?

**Answer:**

**When to take a thread dump:**
- Application is **hanging/unresponsive**
- **High CPU** usage
- **Deadlock** suspected
- **Thread pool exhaustion**

**Capturing Thread Dump:**
```bash
# Method 1: jstack (most common)
jstack <PID> > thread_dump_$(date +%Y%m%d_%H%M%S).txt

# Method 2: jcmd (preferred, more reliable)
jcmd <PID> Thread.print > thread_dump.txt

# Method 3: kill -3 (sends SIGQUIT, dumps to stdout/stderr)
kill -3 <PID>

# Best practice: Take 3 dumps, 10 seconds apart
for i in 1 2 3; do
    jstack <PID> > thread_dump_$i.txt
    sleep 10
done
# Compare the 3 dumps to see which threads are stuck
```

**Reading Thread Dumps:**

```
"payment-worker-1" #42 daemon prio=5 os_prio=0 tid=0x00007f... nid=0x2a03 
    BLOCKED (on object monitor)
    at com.shopease.PaymentService.processPayment(PaymentService.java:45)
    - waiting to lock <0x00000000c0035a08> (a java.lang.Object)
    - locked by "payment-worker-2" #43

"payment-worker-2" #43 daemon prio=5 os_prio=0 tid=0x00007f... nid=0x2a04
    BLOCKED (on object monitor)
    at com.shopease.AccountService.debit(AccountService.java:30)
    - waiting to lock <0x00000000c0035b10> (a java.lang.Object)
    - locked by "payment-worker-1" #42

Found 1 deadlock.
```

**Thread States:**

| State | Meaning | Action |
|-------|---------|--------|
| `RUNNABLE` | Executing or ready to execute | Normal (unless CPU is high) |
| `BLOCKED` | Waiting for monitor lock | Check what's holding the lock |
| `WAITING` | Waiting indefinitely (wait(), join()) | Check what it's waiting for |
| `TIMED_WAITING` | Waiting with timeout (sleep(), wait(timeout)) | Usually normal |
| `NEW` | Created but not started | Unusual in production |
| `TERMINATED` | Finished execution | Normal |

**Common Patterns:**

| Pattern | Symptom | Cause |
|---------|---------|-------|
| Many threads BLOCKED on same lock | Thread contention | Reduce synchronized scope, use ConcurrentHashMap |
| Deadlock detected | Two threads waiting for each other | Consistent lock ordering |
| All threads WAITING in pool | Thread pool exhausted | Increase pool size or fix slow tasks |
| Threads stuck in DB call | TIMED_WAITING on socket read | Slow query, connection timeout |
| Threads stuck in HTTP call | TIMED_WAITING on socket read | External service down, add timeout |

---

## Section 7: Incident Response

### 💎 Q7. Walk through your incident response process for a payment outage.

**Answer:**

**Incident Severity Levels:**

| Level | Impact | Response Time | Example |
|-------|--------|--------------|---------|
| **SEV-1** | Complete outage, all payments failing | < 5 min | Payment gateway down |
| **SEV-2** | Partial outage, some payments failing | < 15 min | One payment method failing |
| **SEV-3** | Degraded performance | < 1 hour | High latency, increased errors |
| **SEV-4** | Minor issue, no user impact | Next business day | Log errors, non-critical bug |

**Incident Response Process:**

```
┌─────────────────────────────────────────────────────────────┐
│                    INCIDENT TIMELINE                         │
│                                                              │
│  T+0min   DETECT                                            │
│  ├── Alert fires (PagerDuty/OpsGenie)                       │
│  ├── On-call engineer acknowledges                          │
│  └── Create incident channel (#inc-payment-20240115)        │
│                                                              │
│  T+5min   TRIAGE                                            │
│  ├── Assess severity (SEV-1/2/3/4)                          │
│  ├── Identify blast radius (which users/merchants affected) │
│  ├── Check: Is it us or external? (gateway, DB, network)    │
│  └── Assign Incident Commander (IC)                         │
│                                                              │
│  T+10min  MITIGATE                                          │
│  ├── Can we rollback? (recent deployment?)                  │
│  ├── Can we failover? (backup gateway, read replica)        │
│  ├── Can we shed load? (rate limit, circuit breaker)        │
│  └── Communicate: Status page update, stakeholder notify    │
│                                                              │
│  T+30min  RESOLVE                                           │
│  ├── Root cause identified                                  │
│  ├── Fix deployed or workaround in place                    │
│  ├── Monitoring confirms recovery                           │
│  └── All-clear communicated                                 │
│                                                              │
│  T+48hrs  POST-MORTEM                                       │
│  ├── Blameless post-mortem document                         │
│  ├── Timeline of events                                     │
│  ├── Root cause analysis (5 Whys)                           │
│  ├── Action items with owners and deadlines                 │
│  └── Share learnings with team                              │
└─────────────────────────────────────────────────────────────┘
```

**Post-Mortem Template (5 Whys):**
```markdown
## Incident: Payment Processing Outage — Jan 15, 2024

### Summary
Payment processing was down for 23 minutes affecting ~5,000 transactions.

### Timeline
- 14:30 UTC — Deployment v2.3.1 rolled out
- 14:35 UTC — Error rate alert fires (5xx > 5%)
- 14:37 UTC — On-call acknowledges, starts investigation
- 14:42 UTC — Root cause identified: new query missing index
- 14:45 UTC — Rollback initiated
- 14:48 UTC — Rollback complete
- 14:53 UTC — Error rate back to normal, all-clear

### 5 Whys
1. Why did payments fail? → DB queries timing out
2. Why were queries timing out? → Full table scan on payments table
3. Why was there a full table scan? → New query added without index
4. Why wasn't the missing index caught? → No query plan review in PR process
5. Why no query plan review? → No automated check in CI/CD pipeline

### Action Items
| # | Action | Owner | Deadline |
|---|--------|-------|----------|
| 1 | Add index for new query | Dev Team | Jan 16 |
| 2 | Add query plan review to PR checklist | Tech Lead | Jan 22 |
| 3 | Add automated slow query detection in CI | Platform Team | Feb 1 |
| 4 | Add DB query timeout (5s max) | Dev Team | Jan 18 |
| 5 | Improve rollback automation | DevOps | Jan 30 |
```

---

## Quick Revision Checklist

- [ ] OOM: HeapDumpOnOutOfMemoryError flag, Eclipse MAT, Dominator Tree
- [ ] OOM Causes: Unbounded cache, connection leak, large queries, ThreadLocal
- [ ] Kafka Lag: Check consumer health → processing time → scale up → batch
- [ ] DB CPU Spike: v$sql → execution plan → missing index → stale stats
- [ ] Duplicate Charges: Idempotency key, check status before retry, auto-detect
- [ ] Transaction Timeout: Mark UNCERTAIN → query gateway → scheduled recheck
- [ ] Thread Dump: jstack/jcmd, take 3 dumps 10s apart, look for BLOCKED/deadlock
- [ ] Heap Dump: jmap/jcmd, Eclipse MAT, Leak Suspects Report
- [ ] Incident Response: Detect → Triage → Mitigate → Resolve → Post-mortem
- [ ] Post-mortem: Blameless, 5 Whys, action items with owners and deadlines
- [ ] Always have: Runbooks, escalation paths, status page communication
