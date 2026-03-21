# ⚡ Module 19: JMeter — Performance Testing

> All examples test the **ShopEase** product-service REST APIs.

---

## 1. What is JMeter?

| Aspect | Details |
|---|---|
| **Type** | Open-source performance testing tool |
| **Developed by** | Apache Software Foundation |
| **Purpose** | Test application speed, stability under load |
| **When used** | Before production release to validate throughput |

---

## 2. How JMeter Works

```
JMeter sends concurrent HTTP requests (simulated users)
        │
        ▼
┌──────────────────────────────────────┐
│  ShopEase Product Service            │
│  GET /api/products                   │
│                                      │
│  Response times, throughput, errors   │
└──────────────────────────────────────┘
        │
        ▼
JMeter generates report:
  ✅ Avg response time
  ✅ Throughput (requests/sec)
  ✅ Error %
  ✅ Min/Max/Median response times
```

---

## 3. JMeter Setup

```bash
# 1. Download JMeter: https://jmeter.apache.org/download_jmeter.cgi
# 2. Extract ZIP
# 3. Run jmeter.bat (GUI) or jmeter (CLI)

# 4. In GUI mode:
#    - Create Test Plan
#    - Add Thread Group (simulated users)
#    - Add HTTP Request
#    - Add Listeners (for results)
```

---

## 4. Testing ShopEase APIs

### Test Plan Setup

| Setting | Value |
|---|---|
| **Thread Group** | Number of Users: 100, Ramp-Up: 10 sec, Loop: 5 |
| **HTTP Request** | Method: GET, URL: `http://localhost:8081/api/products` |
| **Listeners** | Summary Report, View Results Tree, Graph |

### What the Numbers Mean

| Metric | Good Value | Bad Value |
|---|---|---|
| **Avg Response** | < 200ms | > 2000ms |
| **Throughput** | > 100 req/sec | < 10 req/sec |
| **Error %** | 0% | > 5% |
| **90th Percentile** | < 500ms | > 3000ms |

---

## 5. JMeter Results — ShopEase Example

```
┌──────────────────────────────────────────────────────────┐
│              JMeter Summary Report                        │
├──────────────┬───────┬──────────┬──────────┬─────────────┤
│ Label        │ #Req  │ Avg (ms) │ Error%   │ Throughput  │
├──────────────┼───────┼──────────┼──────────┼─────────────┤
│ GET Products │ 500   │ 145      │ 0.00%    │ 87.5/sec    │
│ GET Product/1│ 500   │ 89       │ 0.00%    │ 112.3/sec   │
│ POST Product │ 500   │ 234      │ 0.20%    │ 65.7/sec    │
│ Search       │ 500   │ 312      │ 0.00%    │ 53.2/sec    │
└──────────────┴───────┴──────────┴──────────┴─────────────┘
```

> **💡 Pro Tip:** If response times are high, check DB indexing, enable Redis caching, and check server resources. JMeter helps validate improvements.

---

*← [18 — Angular](./18-angular-frontend.md) | [20 — Resume & Interview →](./20-resume-and-interview.md)*
