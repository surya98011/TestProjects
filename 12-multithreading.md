# 🧵 Module 12: Multi Threading

> All examples use the **ShopEase** notification-service.

---

## 1. Why Multi Threading?

Multi threading enables **parallel processing** — critical for batch operations:

| Scenario | Without Threading | With 10 Threads |
|---|---|---|
| 1 notification = 1 sec | 3,600/hour | 36,000/hour |
| 86,400 notifications | ~24 hours | ~2.4 hours |

### ShopEase Use Cases
- Sending order delivery notifications (email + WhatsApp)
- Generating monthly invoices
- Bulk promotional emails

---

## 2. Three Ways to Create Threads

### Approach 1: Extend Thread Class

```java
public class NotificationThread extends Thread {
    @Override
    public void run() {
        System.out.println("Sending notification on thread: "
                           + Thread.currentThread().getName());
    }
}
// Usage:
new NotificationThread().start();
```

> ⚠️ **Limitation:** Java doesn't support multiple inheritance. If you extend Thread, you can't extend any other class.

### Approach 2: Implement Runnable

```java
public class NotificationTask implements Runnable {
    @Override
    public void run() {
        System.out.println("Sending notification...");
    }
}
// Usage:
new Thread(new NotificationTask()).start();
```

### Approach 3: Implement Callable + ExecutorService (Recommended ✅)

```java
ExecutorService executor = Executors.newFixedThreadPool(10);

executor.submit(new Callable<String>() {
    @Override
    public String call() throws Exception {
        sendEmail("surya@gmail.com");
        return "Email sent";
    }
});
```

---

## 3. ShopEase: Bulk Notification with ThreadPool

```java
@Service
@Slf4j
public class BulkNotificationService {

    @Autowired
    private OrderRepository orderRepo;

    public void sendDeliveryNotifications() {
        List<Order> todayOrders = orderRepo.findByDeliveryDate(LocalDate.now());
        log.info("Total deliveries today: {}", todayOrders.size());

        // ✅ Create thread pool with 10 threads
        ExecutorService executor = Executors.newFixedThreadPool(10);

        for (Order order : todayOrders) {
            executor.submit(() -> {
                try {
                    sendEmail(order.getEmail(),
                              "Your order " + order.getId() + " is out for delivery!");
                    sendWhatsApp(order.getPhone(),
                                 "Delivery update for order " + order.getId());
                    log.info("Notification sent for order {} on thread {}",
                             order.getId(), Thread.currentThread().getName());
                } catch (Exception e) {
                    log.error("Failed to notify order {}: {}",
                              order.getId(), e.getMessage());
                }
            });
        }

        executor.shutdown();
        log.info("All notification tasks submitted");
    }

    private void sendEmail(String to, String message) { /* email logic */ }
    private void sendWhatsApp(String phone, String message) { /* WhatsApp logic */ }
}
```

### Performance Comparison

```
┌──────────────────┬──────────────┬───────────────┬───────────────┐
│ Scenario         │ 1 Thread     │ 10 Threads    │ 20 Threads    │
├──────────────────┼──────────────┼───────────────┼───────────────┤
│ 1 sec/notif      │              │               │               │
│ Per minute       │ 60           │ 600           │ 1,200         │
│ Per hour         │ 3,600        │ 36,000        │ 72,000        │
│ 72,000 notifs    │ 20 hours     │ 2 hours       │ 1 hour        │
└──────────────────┴──────────────┴───────────────┴───────────────┘
```

> **💡 Pro Tip:** In Spring Boot, use `@Async` with `@EnableAsync` for simpler async processing. For massive batch jobs, consider **Spring Batch**.

---

*← [11 — Redis Cache](./11-redis-cache.md) | [13 — Docker →](./13-docker.md)*
