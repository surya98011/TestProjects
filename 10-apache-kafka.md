# 📨 Module 10: Apache Kafka — Messaging

> All examples use the **ShopEase** e-commerce microservice project.

---

## 📑 Table of Contents

- [1. What is Kafka?](#1-what-is-kafka)
- [2. Kafka Architecture](#2-kafka-architecture)
- [3. Kafka Setup](#3-kafka-setup)
- [4. ShopEase Producer — Order Service](#4-shopease-producer--order-service)
- [5. ShopEase Consumer — Notification Service](#5-shopease-consumer--notification-service)
- [6. End-to-End Flow](#6-end-to-end-flow)

---

## 1. What is Kafka?

| Aspect | Details |
|---|---|
| **Type** | Distributed streaming / messaging platform |
| **Model** | Publish-Subscribe (Pub/Sub) |
| **Use Cases** | Real-time data processing, event-driven architectures |
| **ShopEase Use** | Order placed → Kafka event → Notification service sends email |

> **Analogy:** Kafka is like a **newspaper distribution center**. Publishers (order-service) drop messages into topics, and subscribers (notification-service) pick them up independently.

---

## 2. Kafka Architecture

```
┌─────────────┐     ┌──────────────────────────────┐     ┌──────────────┐
│  Producer   │────▶│  Kafka Broker                 │────▶│  Consumer    │
│(order-svc)  │     │  ┌──────────────────────────┐ │     │(notif-svc)   │
│             │     │  │ Topic: order-events       │ │     │              │
│ POST /order │     │  │  Partition 0: [msg1,msg2] │ │     │ @KafkaListener│
└─────────────┘     │  │  Partition 1: [msg3,msg4] │ │     └──────────────┘
                    │  └──────────────────────────┘ │
                    └──────────────────────────────┘
                              ▲
                    ┌─────────┴─────────┐
                    │   Zookeeper       │  (Manages Kafka cluster)
                    └───────────────────┘
```

---

## 3. Kafka Setup

```bash
# Step 1: Start Zookeeper
zookeeper-server-start.bat zookeeper.properties

# Step 2: Start Kafka Server
kafka-server-start.bat server.properties

# Step 3: Create Topic
kafka-topics.bat --create --bootstrap-server localhost:9092 \
    --replication-factor 1 --partitions 1 --topic shopease-order-events

# Step 4: List Topics
kafka-topics.bat --list --bootstrap-server localhost:9092
```

---

## 4. ShopEase Producer — Order Service

### Dependencies (pom.xml)

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

### OrderEvent Model

```java
package com.shopease.order.event;

import lombok.Data;

@Data
public class OrderEvent {
    private String orderId;
    private String productName;
    private Double totalAmount;
    private String customerEmail;
    private String status;
}
```

### Kafka Producer Config

```java
@Configuration
public class KafkaProducerConfig {

    @Bean
    public ProducerFactory<String, OrderEvent> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        return new DefaultKafkaProducerFactory<>(config);
    }

    @Bean
    public KafkaTemplate<String, OrderEvent> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}
```

### OrderService — Publish Event

```java
@Service
@Slf4j
public class OrderService {

    private static final String TOPIC = "shopease-order-events";

    @Autowired private OrderRepository orderRepo;
    @Autowired private KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public Order createOrder(OrderRequest request) {
        // Save order to DB
        Order order = new Order();
        order.setProductName(request.getProductName());
        order.setTotalAmount(request.getTotalAmount());
        order.setEmail(request.getEmail());
        order.setStatus("PLACED");
        Order saved = orderRepo.save(order);

        // ✅ Publish event to Kafka
        OrderEvent event = new OrderEvent();
        event.setOrderId(saved.getId().toString());
        event.setProductName(saved.getProductName());
        event.setTotalAmount(saved.getTotalAmount());
        event.setCustomerEmail(saved.getEmail());
        event.setStatus("PLACED");

        kafkaTemplate.send(TOPIC, event);
        log.info("Order event published to Kafka: {}", event.getOrderId());

        return saved;
    }
}
```

---

## 5. ShopEase Consumer — Notification Service

### Kafka Consumer Config

```java
@Configuration
public class KafkaConsumerConfig {

    @Bean
    public ConsumerFactory<String, OrderEvent> consumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        config.put(JsonDeserializer.TRUSTED_PACKAGES, "*");
        return new DefaultKafkaConsumerFactory<>(config, new StringDeserializer(),
                new JsonDeserializer<>(OrderEvent.class));
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, OrderEvent> kafkaListenerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, OrderEvent> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        return factory;
    }
}
```

### Notification Listener

```java
@Service
@Slf4j
public class NotificationService {

    @KafkaListener(topics = "shopease-order-events", groupId = "shopease-notifications")
    public void handleOrderEvent(OrderEvent event) {
        log.info("Received order event: orderId={}, email={}",
                 event.getOrderId(), event.getCustomerEmail());

        // ✅ Send email notification
        sendEmail(event.getCustomerEmail(),
                  "Order Confirmed: " + event.getOrderId(),
                  "Your order for " + event.getProductName() +
                  " of ₹" + event.getTotalAmount() + " has been placed!");

        log.info("Email notification sent to {}", event.getCustomerEmail());
    }

    private void sendEmail(String to, String subject, String body) {
        // Email sending logic (JavaMailSender)
    }
}
```

---

## 6. End-to-End Flow

```
Customer places order
        │
        ▼
POST /api/orders ──▶ Order Service
        │
        ├── Save to MySQL (orders table)
        └── Publish OrderEvent to Kafka topic
                    │
                    ▼
        Notification Service (consumer) picks up event
                    │
                    └── Sends email to customer
```

### Test with Postman

```json
POST http://localhost:8080/api/orders
Content-Type: application/json

{
    "productName": "Dell Laptop",
    "totalAmount": 59999.00,
    "email": "surya@gmail.com"
}

Response: { "id": 1, "status": "PLACED", ... }
// → Check notification-service console for: "Email notification sent to surya@gmail.com"
```

---

*← [09 — Spring Security](./09-spring-security.md) | [11 — Redis Cache →](./11-redis-cache.md)*
