# 08 — Observability & Dynatrace (Distributed Tracing, OpenTelemetry, SLI/SLO, Alerting)

> **Priority**: 🟡 High  
> **Estimated Study Time**: 1 day  
> Markers: 🔥 = Frequently Asked | 💎 = Differentiator

---

## Section 1: Observability Fundamentals

### 🔥 Q1. What are the three pillars of observability? How do they work together?

**Answer:**

```
┌─────────────────────────────────────────────────────────────┐
│                    Three Pillars of Observability             │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │    Logs       │  │   Metrics    │  │   Traces     │      │
│  │              │  │              │  │              │      │
│  │ What happened│  │ How much/    │  │ Request flow │      │
│  │ (events)     │  │ how fast     │  │ across       │      │
│  │              │  │ (numbers)    │  │ services     │      │
│  │ Structured   │  │ Counters,    │  │ Spans,       │      │
│  │ JSON logs    │  │ Gauges,      │  │ TraceID,     │      │
│  │ with context │  │ Histograms   │  │ SpanID       │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│         │                  │                  │              │
│         └──────────────────┼──────────────────┘              │
│                            │                                 │
│                   Correlation ID / Trace ID                  │
│                   (links all three together)                 │
└─────────────────────────────────────────────────────────────┘
```

| Pillar | What | Tool Examples | Use Case |
|--------|------|---------------|----------|
| **Logs** | Discrete events with context | ELK Stack, Loki, Splunk | Debugging specific errors |
| **Metrics** | Numeric measurements over time | Prometheus, Grafana, Dynatrace | Dashboards, alerting, trends |
| **Traces** | Request journey across services | Jaeger, Zipkin, Dynatrace | Latency analysis, bottleneck detection |

**How they work together:**
1. **Alert fires** (metric: p99 latency > 500ms)
2. **Check dashboard** (metric: which endpoint is slow?)
3. **Find trace** (trace: what's the request flow? which service is slow?)
4. **Read logs** (log: what error occurred in that service?)

```java
// Structured logging with trace context
@Slf4j
@Service
public class PaymentService {
    
    public PaymentResult processPayment(PaymentRequest request) {
        log.info("Processing payment: txnId={}, amount={}, merchantId={}, traceId={}",
            request.getTxnId(),
            request.getAmount(),
            request.getMerchantId(),
            MDC.get("traceId")); // Automatically set by OpenTelemetry/Sleuth
        
        // Process...
        
        // Custom metric
        meterRegistry.counter("payments.processed", 
            "status", "success", 
            "merchant", request.getMerchantId()).increment();
        
        return result;
    }
}
```

---

### 🔥 Q2. Explain distributed tracing. How does it work across microservices?

**Answer:**

Distributed tracing tracks a request as it flows through multiple microservices.

**Key Concepts:**
- **Trace**: The entire journey of a request (one trace per user request)
- **Span**: A single operation within a trace (one span per service call)
- **Trace ID**: Unique ID for the entire request (propagated across services)
- **Span ID**: Unique ID for each operation
- **Parent Span ID**: Links child spans to parent

```
User Request (Trace ID: abc-123)
│
├── Span 1: API Gateway (SpanID: s1, Parent: none)
│   Duration: 250ms
│   │
│   ├── Span 2: Payment Service (SpanID: s2, Parent: s1)
│   │   Duration: 200ms
│   │   │
│   │   ├── Span 3: Fraud Check (SpanID: s3, Parent: s2)
│   │   │   Duration: 50ms
│   │   │
│   │   ├── Span 4: DB Query (SpanID: s4, Parent: s2)
│   │   │   Duration: 30ms
│   │   │
│   │   └── Span 5: Kafka Publish (SpanID: s5, Parent: s2)
│   │       Duration: 10ms
│   │
│   └── Span 6: Notification Service (SpanID: s6, Parent: s1)
│       Duration: 20ms

Timeline:
|------ API Gateway (250ms) ------|
  |---- Payment Service (200ms) ----|
    |-- Fraud (50ms) --|
                        |-- DB (30ms) --|
                                         |- Kafka (10ms) -|
  |-- Notification (20ms) --|
```

**Context Propagation:**
```
Service A                          Service B
┌─────────────┐                   ┌─────────────┐
│ Create Span  │                   │ Extract     │
│ TraceID: abc │  HTTP Header:     │ TraceID: abc│
│ SpanID: s1   │  traceparent:     │ SpanID: s2  │
│              │  00-abc-s1-01     │ ParentID: s1│
│              │ ────────────────→ │             │
└─────────────┘                   └─────────────┘
```

**Spring Boot + OpenTelemetry Setup:**
```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  tracing:
    sampling:
      probability: 1.0  # 100% sampling in dev, lower in prod (0.1 = 10%)
  otlp:
    tracing:
      endpoint: http://otel-collector:4318/v1/traces

logging:
  pattern:
    level: "%5p [${spring.application.name},%X{traceId},%X{spanId}]"
```

---

## Section 2: SLI, SLO, SLA

### 🔥 Q3. Explain SLI, SLO, and SLA. Give examples for a payment service.

**Answer:**

| Term | Definition | Example |
|------|-----------|---------|
| **SLI** (Service Level Indicator) | A metric that measures service quality | Request latency, error rate, availability |
| **SLO** (Service Level Objective) | Target value for an SLI | p99 latency < 500ms, availability > 99.95% |
| **SLA** (Service Level Agreement) | Contract with consequences if SLO is breached | 99.9% uptime or credits issued |

```
SLI (What we measure) → SLO (What we target) → SLA (What we promise)
```

**Payment Service SLIs and SLOs:**

| SLI | SLO | Measurement |
|-----|-----|-------------|
| **Availability** | 99.95% (26 min downtime/month) | Successful responses / Total requests |
| **Latency (p50)** | < 100ms | 50th percentile response time |
| **Latency (p99)** | < 500ms | 99th percentile response time |
| **Error Rate** | < 0.1% | 5xx errors / Total requests |
| **Payment Success Rate** | > 98% | Successful payments / Total attempts |
| **Throughput** | > 1000 TPS | Transactions per second |

**Error Budget:**
```
SLO: 99.95% availability
Error Budget: 0.05% = 21.6 minutes/month

If you've used 15 minutes of downtime this month:
  Remaining budget: 6.6 minutes
  Action: Slow down deployments, focus on stability

If budget exhausted:
  Action: Feature freeze, focus on reliability
```

**Implementing SLO Monitoring:**
```java
// Custom SLI metrics
@Component
public class PaymentSLIMetrics {
    
    private final MeterRegistry registry;
    
    @EventListener
    public void onPaymentProcessed(PaymentProcessedEvent event) {
        // Availability SLI
        registry.counter("payment.requests.total",
            "status", event.isSuccess() ? "success" : "failure",
            "endpoint", "/api/v1/payments"
        ).increment();
        
        // Latency SLI
        registry.timer("payment.latency",
            "endpoint", "/api/v1/payments"
        ).record(event.getDuration());
        
        // Payment success rate SLI
        registry.counter("payment.outcome",
            "result", event.getOutcome().name() // SUCCESS, DECLINED, ERROR
        ).increment();
    }
}
```

```yaml
# Prometheus alerting rules for SLO
groups:
  - name: payment-slo
    rules:
      - alert: PaymentLatencySLOBreach
        expr: histogram_quantile(0.99, rate(payment_latency_seconds_bucket[5m])) > 0.5
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Payment p99 latency exceeds 500ms SLO"
          
      - alert: PaymentAvailabilitySLOBreach
        expr: |
          (1 - (rate(payment_requests_total{status="failure"}[1h]) / 
                rate(payment_requests_total[1h]))) < 0.9995
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Payment availability below 99.95% SLO"
```

---

## Section 3: Dynatrace

### 🔥 Q4. How does Dynatrace work? Explain PurePath and Smartscape.

**Answer:**

Dynatrace is an **AI-powered observability platform** that provides full-stack monitoring with automatic instrumentation.

**Key Components:**

| Component | What It Does |
|-----------|-------------|
| **OneAgent** | Auto-instruments applications (Java, .NET, Node.js, etc.) |
| **PurePath** | End-to-end distributed trace of every transaction |
| **Smartscape** | Auto-discovered topology map of your entire environment |
| **Davis AI** | AI engine that detects anomalies and finds root cause |
| **Grail** | Data lakehouse for logs, metrics, traces, events |

**PurePath (Distributed Tracing):**
```
PurePath for: POST /api/v1/payments

┌─ API Gateway (12ms)
│  ├─ Authentication Filter (2ms)
│  └─ Route to Payment Service
│
├─ Payment Service (180ms)
│  ├─ PaymentController.createPayment (1ms)
│  ├─ PaymentService.process (175ms)
│  │  ├─ FraudService.check (45ms)  ← HTTP call to Fraud Service
│  │  │  └─ ML Model inference (40ms)
│  │  ├─ PaymentRepository.save (25ms)
│  │  │  └─ Oracle DB: INSERT INTO payments (20ms)  ← SQL captured
│  │  ├─ GatewayClient.charge (90ms)  ← HTTP call to external gateway
│  │  │  └─ Razorpay API (85ms)
│  │  └─ KafkaTemplate.send (10ms)
│  │     └─ Topic: payment-events, Partition: 2
│  └─ Response serialization (4ms)
│
└─ Total: 192ms
   Hotspot: GatewayClient.charge (47% of time)
```

**Smartscape (Topology Discovery):**
```
Smartscape automatically discovers:

┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ Load Balancer│────→│ API Gateway  │────→│ Payment Svc  │
│ (nginx)      │     │ (Spring)     │     │ (Spring Boot)│
└─────────────┘     └─────────────┘     └──────┬───────┘
                                               │
                    ┌──────────────────────────┤
                    │                          │
              ┌─────▼─────┐            ┌──────▼──────┐
              │ Oracle DB  │            │ Kafka Cluster│
              │ (payments) │            │ (3 brokers)  │
              └────────────┘            └─────────────┘

All connections, dependencies, and versions auto-discovered!
```

**Dynatrace OneAgent Setup (Spring Boot):**
```bash
# Download and install OneAgent
wget -O Dynatrace-OneAgent.sh "https://{your-env}.live.dynatrace.com/api/v1/deployment/installer/agent/unix/default/latest?Api-Token={token}"
sh Dynatrace-OneAgent.sh

# Or via Docker
docker run -d --name dt-oneagent \
  -e DT_API_URL="https://{env}.live.dynatrace.com/api" \
  -e DT_API_TOKEN="{token}" \
  -e DT_ONEAGENT_OPTIONS="flavor=default&include=java" \
  dynatrace/oneagent
```

```yaml
# application.yml — Dynatrace integration
management:
  metrics:
    export:
      dynatrace:
        enabled: true
        api-token: ${DT_API_TOKEN}
        uri: https://{env}.live.dynatrace.com
        v2:
          metric-key-prefix: payment.service
```

---

### 💎 Q5. How does Dynatrace Davis AI detect problems automatically?

**Answer:**

Davis AI uses **causal AI** (not just threshold-based alerting) to:
1. **Detect anomalies** — Learns baseline behavior, detects deviations
2. **Find root cause** — Traces the problem through the dependency graph
3. **Assess impact** — Determines which users/services are affected

**Example: Payment Latency Spike**

```
Davis AI Problem Card:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔴 PROBLEM: Response time degradation

Impact:
  - Payment Service: p99 latency increased from 200ms to 2.5s
  - Affected: 15,000 users in last 30 minutes
  - Error rate increased from 0.1% to 5.2%

Root Cause (auto-detected):
  ├── Payment Service response time ↑
  │   └── Oracle DB query time ↑ (from 20ms to 800ms)
  │       └── Missing index on payments table
  │           └── Query: SELECT * FROM payments WHERE merchant_id = ? 
  │               AND created_at > ? (FULL TABLE SCAN detected)
  │
  └── Triggered by: Deployment v2.3.1 at 14:30 UTC
      (new query added without index)

Remediation Suggestion:
  CREATE INDEX idx_payment_merchant_date ON payments(merchant_id, created_at);
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Davis AI vs Traditional Monitoring:**

| Aspect | Traditional (Threshold) | Davis AI |
|--------|------------------------|----------|
| Alert trigger | Static threshold (latency > 500ms) | Dynamic baseline deviation |
| False positives | High (during deployments, traffic spikes) | Low (understands context) |
| Root cause | Manual investigation | Automatic causal analysis |
| Correlation | Manual (check multiple dashboards) | Automatic (follows dependency graph) |
| Seasonality | Not handled | Learns daily/weekly patterns |

---

## Section 4: Log Correlation & Structured Logging

### 🔥 Q6. How do you implement log correlation across microservices?

**Answer:**

Log correlation links logs from different services for the same request using a **Correlation ID / Trace ID**.

```java
// 1. Filter to extract/generate correlation ID
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CorrelationIdFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                     HttpServletResponse response, 
                                     FilterChain chain) throws ServletException, IOException {
        String correlationId = request.getHeader("X-Correlation-ID");
        if (correlationId == null || correlationId.isBlank()) {
            correlationId = UUID.randomUUID().toString();
        }
        
        // Set in MDC for logging
        MDC.put("correlationId", correlationId);
        MDC.put("service", "payment-service");
        
        // Set in response header
        response.setHeader("X-Correlation-ID", correlationId);
        
        try {
            chain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}

// 2. Logback configuration for structured JSON logging
// logback-spring.xml
```

```xml
<configuration>
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyName>correlationId</includeMdcKeyName>
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <includeMdcKeyName>spanId</includeMdcKeyName>
            <includeMdcKeyName>service</includeMdcKeyName>
        </encoder>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="JSON" />
    </root>
</configuration>
```

**Resulting log output (JSON):**
```json
{
  "timestamp": "2024-01-15T10:30:45.123Z",
  "level": "INFO",
  "service": "payment-service",
  "correlationId": "abc-123-def-456",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "spanId": "00f067aa0ba902b7",
  "logger": "com.shopease.PaymentService",
  "message": "Payment processed successfully",
  "txnId": "TXN-001",
  "amount": 100.00,
  "merchantId": "M-001",
  "duration_ms": 185
}
```

**Searching correlated logs in ELK/Dynatrace:**
```
# Find all logs for a specific request across all services
correlationId: "abc-123-def-456"

# Results:
10:30:45.100 [api-gateway]      Received POST /api/v1/payments
10:30:45.105 [payment-service]  Processing payment: txnId=TXN-001
10:30:45.150 [fraud-service]    Fraud check passed: score=0.05
10:30:45.180 [payment-service]  Gateway charge successful: authCode=AUTH-789
10:30:45.190 [payment-service]  Payment completed: txnId=TXN-001
10:30:45.195 [notification-svc] Sending payment confirmation email
```

---

## Section 5: Alerting Strategy

### 💎 Q7. How do you design an alerting strategy for a payment service?

**Answer:**

**Alerting Principles:**
1. **Alert on symptoms, not causes** (alert on high latency, not high CPU)
2. **Every alert must be actionable** (if you can't act on it, remove it)
3. **Use severity levels** (critical = page, warning = ticket, info = dashboard)
4. **Avoid alert fatigue** (too many alerts = all alerts ignored)

**Payment Service Alert Matrix:**

| Alert | SLI | Threshold | Severity | Action |
|-------|-----|-----------|----------|--------|
| Payment latency high | p99 latency | > 500ms for 5min | 🔴 Critical | Page on-call |
| Payment error rate high | Error rate | > 1% for 5min | 🔴 Critical | Page on-call |
| Payment success rate low | Success rate | < 95% for 10min | 🔴 Critical | Page on-call |
| DB connection pool exhausted | Active connections | > 90% for 2min | 🔴 Critical | Page on-call |
| Kafka consumer lag high | Consumer lag | > 10,000 for 5min | 🟡 Warning | Create ticket |
| Disk space low | Disk usage | > 85% | 🟡 Warning | Create ticket |
| Certificate expiring | Days to expiry | < 30 days | 🟡 Warning | Create ticket |
| Deployment completed | Deployment event | Any | 🔵 Info | Dashboard |

**Runbook Template:**
```markdown
## Alert: Payment Latency High (p99 > 500ms)

### Impact
- Users experiencing slow payment processing
- Potential timeout errors and failed payments

### Investigation Steps
1. Check Dynatrace PurePath for slow transactions
2. Identify bottleneck service/component
3. Check DB query performance (slow queries?)
4. Check external gateway latency
5. Check JVM metrics (GC pauses? heap usage?)
6. Check recent deployments

### Remediation
- If DB slow → Check execution plans, add missing indexes
- If gateway slow → Enable circuit breaker, switch to backup gateway
- If GC pauses → Increase heap or tune GC
- If recent deployment → Rollback

### Escalation
- If not resolved in 15 min → Escalate to Tech Lead
- If not resolved in 30 min → Escalate to Engineering Manager
```

---

## Quick Revision Checklist

- [ ] Three Pillars: Logs (events), Metrics (numbers), Traces (request flow)
- [ ] Distributed Tracing: Trace ID + Span ID, context propagation via headers
- [ ] OpenTelemetry: Vendor-neutral, W3C traceparent header
- [ ] SLI: What you measure (latency, error rate, availability)
- [ ] SLO: Target for SLI (p99 < 500ms, 99.95% availability)
- [ ] SLA: Contract with consequences (99.9% or credits)
- [ ] Error Budget: SLO gap = budget for experimentation/risk
- [ ] Dynatrace PurePath: End-to-end trace with code-level visibility
- [ ] Dynatrace Smartscape: Auto-discovered topology map
- [ ] Davis AI: Causal AI, auto root cause, dynamic baselines
- [ ] Log Correlation: MDC + Correlation ID + structured JSON logging
- [ ] Alerting: Symptoms not causes, actionable, severity levels, runbooks
