# 05 — Apache Kafka (Architecture, Producer/Consumer, Exactly-Once, DLQ, Schema Registry)

> **Priority**: 🔴 Critical  
> **Estimated Study Time**: 2 days  
> Markers: 🔥 = Frequently Asked | 💎 = Differentiator

---

## Section 1: Architecture

### 🔥 Q1. Explain Kafka's architecture. What are the core components?

**Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                      Kafka Cluster                               │
│                                                                  │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                  │
│  │ Broker 0  │    │ Broker 1  │    │ Broker 2  │                  │
│  │           │    │           │    │           │                  │
│  │ Topic: payments                                               │
│  │ ┌───────┐ │    │ ┌───────┐ │    │ ┌───────┐ │                  │
│  │ │ P0    │ │    │ │ P1    │ │    │ │ P2    │ │  ← Leaders      │
│  │ │Leader │ │    │ │Leader │ │    │ │Leader │ │                  │
│  │ └───────┘ │    │ └───────┘ │    │ └───────┘ │                  │
│  │ ┌───────┐ │    │ ┌───────┐ │    │ ┌───────┐ │                  │
│  │ │ P1    │ │    │ │ P2    │ │    │ │ P0    │ │  ← Replicas     │
│  │ │Replica│ │    │ │Replica│ │    │ │Replica│ │                  │
│  │ └───────┘ │    │ └───────┘ │    │ └───────┘ │                  │
│  └──────────┘    └──────────┘    └──────────┘                  │
│                                                                  │
│  ┌──────────────────────────────────────────┐                   │
│  │ ZooKeeper / KRaft (metadata management)   │                   │
│  └──────────────────────────────────────────┘                   │
└─────────────────────────────────────────────────────────────────┘

Producers ──→ Broker (Leader) ──→ Replicas
                    │
                    ▼
              Consumer Groups
              ┌─────────────────┐
              │ Group: payment-  │
              │ processor        │
              │ ┌─────┐ ┌─────┐ │
              │ │ C0  │ │ C1  │ │  C0 reads P0, P1
              │ │     │ │     │ │  C1 reads P2
              │ └─────┘ └─────┘ │
              └─────────────────┘
```

**Core Components:**

| Component | Role |
|-----------|------|
| **Broker** | Server that stores data and serves clients |
| **Topic** | Logical channel for messages (like a table) |
| **Partition** | Ordered, immutable sequence of records within a topic |
| **Offset** | Unique sequential ID for each record in a partition |
| **Producer** | Publishes records to topics |
| **Consumer** | Reads records from topics |
| **Consumer Group** | Set of consumers that cooperatively consume a topic |
| **Replica** | Copy of a partition for fault tolerance |
| **Leader** | The replica that handles all reads/writes for a partition |
| **ISR (In-Sync Replicas)** | Replicas that are caught up with the leader |
| **ZooKeeper/KRaft** | Cluster metadata, leader election (KRaft replaces ZK in newer versions) |

**Key Properties:**
- Messages within a partition are **strictly ordered**
- A partition can only be consumed by **one consumer** in a group
- Kafka retains messages for a configurable period (default 7 days), not until consumed
- Kafka is a **distributed commit log**, not a message queue

---

### 🔥 Q2. How does Kafka ensure message ordering?

**Answer:**

Kafka guarantees ordering **only within a partition**, not across partitions.

```
Topic: payment-events (3 partitions)

Partition 0: [msg1] [msg4] [msg7] → Ordered within P0
Partition 1: [msg2] [msg5] [msg8] → Ordered within P1
Partition 2: [msg3] [msg6] [msg9] → Ordered within P2

Global order across partitions? NOT guaranteed.
```

**How to ensure ordering for related events:**

Use the **same key** for related messages → they go to the same partition.

```java
// All events for the same payment go to the same partition
kafkaTemplate.send("payment-events", payment.getTxnId(), paymentEvent);
// Key = txnId → hash(txnId) % numPartitions → always same partition

// Ordering guarantee:
// INITIATED → PROCESSING → COMPLETED for txn-001 → all in same partition → ordered
```

**Partition Assignment:**
```
key = "txn-001" → hash("txn-001") % 3 = 1 → Partition 1
key = "txn-002" → hash("txn-002") % 3 = 0 → Partition 0
key = "txn-001" → hash("txn-001") % 3 = 1 → Partition 1 (same!)
```

**⚠️ Gotcha: Adding partitions breaks key-based ordering!**
- If you increase partitions from 3 to 6, `hash(key) % 6` gives different results
- Existing keys may map to different partitions
- Solution: Plan partition count upfront, or use custom partitioner

---

## Section 2: Producer Internals

### 🔥 Q3. Explain Kafka Producer internals. What are acks, retries, and idempotent producer?

**Answer:**

```
Producer Architecture:
┌─────────────────────────────────────────────────┐
│                   Producer                       │
│                                                  │
│  send() → Serializer → Partitioner → RecordAccumulator
│                                          │       │
│                                    ┌─────┴─────┐ │
│                                    │ Batch for  │ │
│                                    │ Partition 0│ │
│                                    ├───────────┤ │
│                                    │ Batch for  │ │
│                                    │ Partition 1│ │
│                                    └─────┬─────┘ │
│                                          │       │
│                                    Sender Thread  │
│                                    (background)   │
│                                          │       │
└──────────────────────────────────────────┼───────┘
                                           │
                                           ▼
                                      Kafka Broker
```

**Acknowledgment Levels (acks):**

| acks | Behavior | Durability | Latency | Use Case |
|------|----------|-----------|---------|----------|
| `0` | Fire and forget | ❌ None | ⚡ Lowest | Metrics, logs (loss acceptable) |
| `1` | Leader acknowledges | 🟡 Medium | 🔵 Low | General use |
| `all` (-1) | All ISR replicas acknowledge | ✅ Highest | 🔴 Higher | Payments, financial data |

**Idempotent Producer (exactly-once within a partition):**

```java
// Producer configuration for payment events
@Configuration
public class KafkaProducerConfig {
    
    @Bean
    public ProducerFactory<String, PaymentEvent> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        
        // Durability settings for payments
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        config.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);
        config.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5); // Safe with idempotent
        
        // Idempotent producer — prevents duplicates on retry
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        
        // Batching for throughput
        config.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);       // 16KB batch
        config.put(ProducerConfig.LINGER_MS_CONFIG, 10);           // Wait 10ms to fill batch
        config.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");
        
        return new DefaultKafkaProducerFactory<>(config);
    }
}
```

**How Idempotent Producer Works:**
- Each producer gets a **Producer ID (PID)** from the broker
- Each message gets a **sequence number** per partition
- Broker deduplicates: if it sees same PID + sequence number → ignores duplicate
- Prevents duplicates caused by **producer retries** (network timeout, broker restart)

```
Producer (PID=1) sends:
  Seq 0 → Broker receives ✅
  Seq 1 → Network timeout, producer retries
  Seq 1 → Broker sees PID=1, Seq=1 already exists → DEDUP ✅
  Seq 2 → Broker receives ✅
```

---

## Section 3: Consumer Internals

### 🔥 Q4. Explain Consumer Groups and partition assignment. How does rebalancing work?

**Answer:**

**Consumer Group Rules:**
1. Each partition is assigned to **exactly one consumer** in a group
2. A consumer can read from **multiple partitions**
3. If consumers > partitions → some consumers are **idle**
4. Different consumer groups read **independently** (each gets all messages)

```
Topic: payments (4 partitions)

Consumer Group A (3 consumers):
  Consumer 0 → P0, P1
  Consumer 1 → P2
  Consumer 2 → P3

Consumer Group B (2 consumers):  ← Independent!
  Consumer 0 → P0, P1
  Consumer 1 → P2, P3
```

**Rebalancing** occurs when:
- Consumer joins/leaves the group
- Consumer crashes (heartbeat timeout)
- New partitions added to topic
- Consumer takes too long to process (`max.poll.interval.ms` exceeded)

**Rebalancing Strategies:**

| Strategy | Behavior | Downtime |
|----------|----------|----------|
| **Eager (Range/RoundRobin)** | Revoke ALL partitions, reassign | ❌ Full stop-the-world |
| **Cooperative Sticky** | Only revoke partitions that need to move | ✅ Minimal disruption |

```java
// Consumer configuration
@Configuration
public class KafkaConsumerConfig {
    
    @Bean
    public ConsumerFactory<String, PaymentEvent> consumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        config.put(ConsumerConfig.GROUP_ID_CONFIG, "payment-processor");
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        
        // Cooperative rebalancing (recommended)
        config.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
            CooperativeStickyAssignor.class.getName());
        
        // Offset management
        config.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        config.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false); // Manual commit!
        
        // Tuning
        config.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 100);
        config.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300000); // 5 min
        config.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 30000);    // 30s
        config.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, 10000); // 10s
        
        return new DefaultKafkaConsumerFactory<>(config);
    }
}
```

**⚠️ Common Rebalancing Issues:**
- **Long processing time** → exceeds `max.poll.interval.ms` → consumer kicked out → rebalance loop
- **Fix**: Increase `max.poll.interval.ms` or reduce `max.poll.records`
- **Slow deserialization** → same issue
- **GC pauses** → missed heartbeats → consumer considered dead

---

### 🔥 Q5. How do you handle consumer offset management?

**Answer:**

**Offset = position of the consumer in a partition.** Stored in internal topic `__consumer_offsets`.

| Strategy | How | Risk |
|----------|-----|------|
| **Auto commit** | Commits every `auto.commit.interval.ms` | Data loss (committed but not processed) |
| **Manual sync commit** | `consumer.commitSync()` after processing | Blocks, slower |
| **Manual async commit** | `consumer.commitAsync()` after processing | Non-blocking, may lose on failure |
| **Per-record commit** | Commit after each record | Slowest, safest |

```java
// Spring Kafka — Manual acknowledgment (recommended for payments)
@KafkaListener(
    topics = "payment-events",
    groupId = "payment-processor",
    containerFactory = "kafkaListenerContainerFactory"
)
public void processPayment(
        @Payload PaymentEvent event,
        @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
        @Header(KafkaHeaders.OFFSET) long offset,
        Acknowledgment ack) {
    
    try {
        log.info("Processing payment event: txnId={}, partition={}, offset={}", 
            event.getTxnId(), partition, offset);
        
        paymentService.process(event);
        
        ack.acknowledge(); // ✅ Commit offset only after successful processing
        
    } catch (RetryableException e) {
        // Don't ack — message will be redelivered
        throw e;
    } catch (NonRetryableException e) {
        log.error("Non-retryable error for txnId={}", event.getTxnId(), e);
        // Send to DLQ
        deadLetterPublisher.publish(event, e);
        ack.acknowledge(); // Ack to move past this message
    }
}

// Container factory with manual ack
@Bean
public ConcurrentKafkaListenerContainerFactory<String, PaymentEvent> kafkaListenerContainerFactory() {
    ConcurrentKafkaListenerContainerFactory<String, PaymentEvent> factory = 
        new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory());
    factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL);
    factory.setConcurrency(3); // 3 consumer threads
    return factory;
}
```

---

## Section 4: Exactly-Once Semantics

### 🔥 Q6. Explain Kafka's exactly-once semantics end-to-end.

**Answer:**

**Delivery Guarantees:**

| Guarantee | Mechanism | Duplicates? | Data Loss? |
|-----------|-----------|-------------|------------|
| **At-most-once** | Auto-commit before processing | No | ✅ Yes |
| **At-least-once** | Commit after processing | ✅ Yes | No |
| **Exactly-once** | Transactions + idempotent producer | No | No |

**Exactly-Once = Idempotent Producer + Transactions + Consumer read_committed**

```
┌──────────┐    Transactional     ┌──────────┐    read_committed    ┌──────────┐
│ Source   │    Producer          │  Kafka   │    Consumer          │  Sink    │
│ Topic    │ ──────────────────→  │  Broker  │ ──────────────────→  │  Topic   │
│          │    (atomic write +   │          │    (only reads        │          │
│          │     offset commit)   │          │     committed msgs)  │          │
└──────────┘                      └──────────┘                      └──────────┘
```

```java
// Transactional Producer Configuration
@Bean
public ProducerFactory<String, PaymentEvent> producerFactory() {
    Map<String, Object> config = new HashMap<>();
    config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
    config.put(ProducerConfig.ACKS_CONFIG, "all");
    config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
    config.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "payment-txn-"); // Enables transactions
    
    return new DefaultKafkaProducerFactory<>(config);
}

@Bean
public KafkaTransactionManager<String, PaymentEvent> kafkaTransactionManager(
        ProducerFactory<String, PaymentEvent> producerFactory) {
    return new KafkaTransactionManager<>(producerFactory);
}

// Exactly-once: Read from topic → Process → Write to topic (atomic)
@Transactional("kafkaTransactionManager")
@KafkaListener(topics = "raw-payments", groupId = "payment-enricher")
public void enrichAndForward(PaymentEvent rawEvent, Acknowledgment ack) {
    // 1. Process
    EnrichedPaymentEvent enriched = enrichmentService.enrich(rawEvent);
    
    // 2. Write to output topic (within same transaction)
    kafkaTemplate.send("enriched-payments", enriched.getTxnId(), enriched);
    
    // 3. Offset commit happens atomically with the produce
    // If any step fails, everything rolls back (no partial writes)
}
```

**Exactly-Once in Practice (Consume-Transform-Produce):**
1. Consumer reads from input topic
2. Application processes the message
3. Producer writes to output topic
4. Consumer offset is committed
5. Steps 3 & 4 happen **atomically** in a Kafka transaction

**⚠️ Limitations:**
- Exactly-once is **within Kafka** only (Kafka → Kafka)
- For Kafka → External DB: Use **idempotent consumers** (dedup at application level)
- Transactions add latency (~50-100ms overhead)

---

## Section 5: Dead Letter Queue (DLQ)

### 🔥 Q7. How do you implement a Dead Letter Queue in Kafka?

**Answer:**

A DLQ captures messages that **fail processing** after all retries are exhausted.

```
                    ┌─────────────┐
                    │ Main Topic   │
                    │ payment-     │
                    │ events       │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  Consumer    │
                    │  (process)   │
                    └──────┬──────┘
                           │
                    ┌──────┴──────┐
                    │             │
               Success        Failure
                    │             │
                    ▼             ▼
              ack()        Retry (3x)
                                 │
                           ┌─────┴─────┐
                           │           │
                      Retry OK    All retries
                           │      exhausted
                           ▼           │
                      ack()           ▼
                              ┌──────────────┐
                              │ DLQ Topic     │
                              │ payment-      │
                              │ events.DLT    │
                              └──────────────┘
```

**Spring Kafka DLQ with Retry:**

```java
@Configuration
public class KafkaRetryConfig {
    
    @Bean
    public DefaultErrorHandler errorHandler(KafkaTemplate<String, Object> kafkaTemplate) {
        // Retry 3 times with exponential backoff
        FixedBackOff backOff = new FixedBackOff(1000L, 3); // 1s interval, 3 attempts
        
        // Dead letter publisher
        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
            kafkaTemplate,
            (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition())
        );
        
        DefaultErrorHandler handler = new DefaultErrorHandler(recoverer, backOff);
        
        // Don't retry these exceptions (non-retryable)
        handler.addNotRetryableExceptions(
            DeserializationException.class,
            ValidationException.class,
            NullPointerException.class
        );
        
        return handler;
    }
}

// Advanced: Custom retry with exponential backoff
@Configuration
@EnableRetryTopic
public class KafkaRetryTopicConfig {
    
    @RetryableTopic(
        attempts = "4",
        backoff = @Backoff(delay = 1000, multiplier = 2, maxDelay = 10000),
        topicSuffixingStrategy = TopicSuffixingStrategy.SUFFIX_WITH_INDEX_VALUE,
        dltStrategy = DltStrategy.FAIL_ON_ERROR,
        include = {RetryableException.class, TimeoutException.class}
    )
    @KafkaListener(topics = "payment-events", groupId = "payment-processor")
    public void processPayment(PaymentEvent event) {
        paymentService.process(event);
    }
    
    @DltHandler
    public void handleDlt(PaymentEvent event, 
                          @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
                          @Header(KafkaHeaders.EXCEPTION_MESSAGE) String errorMsg) {
        log.error("DLT received: topic={}, txnId={}, error={}", topic, event.getTxnId(), errorMsg);
        
        // Store in DB for manual review
        dlqRepository.save(DlqRecord.builder()
            .txnId(event.getTxnId())
            .payload(objectMapper.writeValueAsString(event))
            .errorMessage(errorMsg)
            .sourceTopic(topic)
            .createdAt(Instant.now())
            .status(DlqStatus.PENDING_REVIEW)
            .build());
        
        // Alert operations team
        alertService.sendAlert("Payment DLQ", "Failed payment: " + event.getTxnId());
    }
}
```

**Retry Topics Created:**
```
payment-events              ← Main topic
payment-events-retry-0      ← 1st retry (1s delay)
payment-events-retry-1      ← 2nd retry (2s delay)
payment-events-retry-2      ← 3rd retry (4s delay)
payment-events.DLT          ← Dead letter topic
```

---

## Section 6: Schema Registry

### 💎 Q8. What is Schema Registry? Why is it important?

**Answer:**

Schema Registry manages **Avro/JSON/Protobuf schemas** for Kafka topics, ensuring producers and consumers agree on data format.

```
┌──────────┐     1. Register schema     ┌─────────────────┐
│ Producer  │ ──────────────────────────→│ Schema Registry  │
│           │     2. Get schema ID       │                  │
│           │ ←──────────────────────────│ Schemas:         │
│           │                            │  v1: {txnId,amt} │
│           │     3. Send data with      │  v2: {txnId,amt, │
│           │        schema ID           │       currency}  │
└─────┬─────┘                            └────────┬────────┘
      │                                           │
      │  [schemaId=2][data]                       │
      ▼                                           │
┌──────────┐                                      │
│  Kafka   │                                      │
│  Broker  │                                      │
└─────┬────┘                                      │
      │                                           │
      ▼                                           │
┌──────────┐     4. Fetch schema by ID   ┌────────┘
│ Consumer  │ ──────────────────────────→│
│           │     5. Deserialize with     │
│           │        schema               │
└──────────┘                              
```

**Schema Evolution & Compatibility:**

| Mode | Rule | Example |
|------|------|---------|
| **BACKWARD** | New schema can read old data | Add field with default |
| **FORWARD** | Old schema can read new data | Remove optional field |
| **FULL** | Both backward and forward | Add/remove optional fields with defaults |
| **NONE** | No compatibility check | Breaking changes allowed |

```java
// Avro schema for PaymentEvent
// payment-event.avsc
{
  "type": "record",
  "name": "PaymentEvent",
  "namespace": "com.shopease.payments",
  "fields": [
    {"name": "txnId", "type": "string"},
    {"name": "amount", "type": "double"},
    {"name": "currency", "type": "string", "default": "INR"},  // Added in v2 with default
    {"name": "status", "type": {"type": "enum", "name": "Status", 
      "symbols": ["INITIATED", "PROCESSING", "COMPLETED", "FAILED"]}},
    {"name": "timestamp", "type": "long"}
  ]
}
```

```yaml
# application.yml
spring:
  kafka:
    properties:
      schema.registry.url: http://schema-registry:8081
    producer:
      value-serializer: io.confluent.kafka.serializers.KafkaAvroSerializer
    consumer:
      value-deserializer: io.confluent.kafka.serializers.KafkaAvroDeserializer
      properties:
        specific.avro.reader: true
```

---

## Section 7: Production Patterns

### 💎 Q9. How do you handle Kafka consumer lag in production?

**Answer:**

**Consumer lag** = difference between the latest offset (log-end) and the consumer's committed offset.

**Monitoring:**
```bash
# Check consumer lag
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --describe --group payment-processor

# Output:
# TOPIC          PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# payment-events 0          1000            1500            500
# payment-events 1          2000            2100            100
# payment-events 2          1500            1500            0
```

**Causes & Solutions:**

| Cause | Solution |
|-------|----------|
| Slow processing | Optimize processing logic, async I/O |
| Too few consumers | Add consumers (up to partition count) |
| Too few partitions | Increase partitions (plan ahead!) |
| Large messages | Compress, or use claim-check pattern |
| GC pauses | Tune JVM, use ZGC |
| External dependency slow | Circuit breaker, async calls |
| Rebalancing storms | Use CooperativeStickyAssignor |

```java
// Scaling consumers dynamically
@KafkaListener(
    topics = "payment-events",
    groupId = "payment-processor",
    concurrency = "${kafka.consumer.concurrency:3}" // Configurable
)
public void process(PaymentEvent event, Acknowledgment ack) {
    paymentService.process(event);
    ack.acknowledge();
}

// Batch processing for higher throughput
@KafkaListener(topics = "payment-events", groupId = "payment-processor")
public void processBatch(List<PaymentEvent> events, Acknowledgment ack) {
    // Process in batch — much faster than one-by-one
    paymentService.processBatch(events);
    ack.acknowledge();
}
```

**Alerting:**
```yaml
# Prometheus alert for consumer lag
- alert: KafkaConsumerLagHigh
  expr: kafka_consumer_group_lag > 10000
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Kafka consumer lag is high"
    description: "Consumer group {{ $labels.group }} has lag {{ $value }} on topic {{ $labels.topic }}"
```

---

### 💎 Q10. Explain the Outbox Pattern with Kafka.

**Answer:**

The Outbox Pattern solves the **dual-write problem**: How to atomically update a database AND publish a Kafka event?

**Problem:**
```java
// ❌ Dual-write problem
@Transactional
public void processPayment(PaymentRequest request) {
    paymentRepository.save(payment);        // 1. DB write ✅
    kafkaTemplate.send("payments", event);  // 2. Kafka write — what if this fails?
    // DB committed but Kafka didn't get the event → inconsistency!
}
```

**Solution — Outbox Pattern:**

```
┌─────────────────────────────────────────────────────────┐
│                    Single DB Transaction                  │
│                                                          │
│  1. INSERT INTO payments (...)                           │
│  2. INSERT INTO outbox_events (topic, key, payload, ...) │
│                                                          │
│  COMMIT (atomic — both or neither)                       │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────┐
│  Outbox Poller / CDC (Debezium)                          │
│                                                          │
│  Reads outbox_events table → Publishes to Kafka          │
│  Marks events as published                               │
└──────────────────────────────────────────────────────────┘
```

```java
// Outbox table entity
@Entity
@Table(name = "outbox_events")
public class OutboxEvent {
    @Id @GeneratedValue
    private Long id;
    
    private String aggregateType;  // "Payment"
    private String aggregateId;    // txnId
    private String eventType;      // "PaymentCompleted"
    private String topic;          // "payment-events"
    
    @Column(columnDefinition = "CLOB")
    private String payload;        // JSON
    
    private Instant createdAt;
    private boolean published;
}

// Service — atomic DB write
@Service
public class PaymentService {
    
    @Transactional
    public PaymentResult processPayment(PaymentRequest request) {
        // 1. Business logic + DB write
        Payment payment = createAndSavePayment(request);
        
        // 2. Write to outbox (same transaction!)
        OutboxEvent event = OutboxEvent.builder()
            .aggregateType("Payment")
            .aggregateId(payment.getTxnId())
            .eventType("PaymentCompleted")
            .topic("payment-events")
            .payload(objectMapper.writeValueAsString(toEvent(payment)))
            .createdAt(Instant.now())
            .published(false)
            .build();
        outboxRepository.save(event);
        
        return toResult(payment);
        // Both payment and outbox event committed atomically
    }
}

// Outbox Poller — publishes to Kafka
@Component
@Slf4j
public class OutboxPoller {
    
    @Scheduled(fixedDelay = 1000) // Poll every second
    @Transactional
    public void publishPendingEvents() {
        List<OutboxEvent> events = outboxRepository
            .findByPublishedFalseOrderByCreatedAtAsc();
        
        for (OutboxEvent event : events) {
            try {
                kafkaTemplate.send(event.getTopic(), event.getAggregateId(), event.getPayload())
                    .get(5, TimeUnit.SECONDS); // Wait for ack
                
                event.setPublished(true);
                outboxRepository.save(event);
            } catch (Exception e) {
                log.error("Failed to publish outbox event: {}", event.getId(), e);
                break; // Stop processing to maintain order
            }
        }
    }
}
```

**Alternative: Use Debezium CDC** (Change Data Capture) to tail the outbox table's transaction log → no polling needed, lower latency.

---

## Quick Revision Checklist

- [ ] Architecture: Broker, Topic, Partition, Offset, Consumer Group, ISR
- [ ] Ordering: Only within a partition. Use same key for related events
- [ ] Producer: acks=all, idempotent=true, transactional for exactly-once
- [ ] Consumer Groups: 1 partition → 1 consumer per group, rebalancing
- [ ] Rebalancing: CooperativeStickyAssignor, tune max.poll.interval.ms
- [ ] Offset Management: Manual commit after processing (MANUAL ack mode)
- [ ] Exactly-Once: Idempotent producer + Transactions + read_committed
- [ ] DLQ: Retry topics + DLT topic, @RetryableTopic, @DltHandler
- [ ] Schema Registry: Avro schemas, BACKWARD compatibility, schema evolution
- [ ] Consumer Lag: Monitor, scale consumers, batch processing
- [ ] Outbox Pattern: Atomic DB + outbox write, poller/CDC publishes to Kafka
- [ ] Key configs: acks, retries, max.in.flight, batch.size, linger.ms, compression