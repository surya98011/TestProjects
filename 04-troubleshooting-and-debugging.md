# 🛠️ Real-Time Troubleshooting & System Debugging Guide

This document contains expert-level, real-world solutions to the 15 critical production scenarios.

## 1. CPU Spikes to 90% in Production
**Investigation:**
* **Identify the process/thread:** Use `top -H -p <pid>` to find which specific thread is consuming CPU. Convert the thread ID to Hex.
* **Thread Dump:** Generate a thread dump using `jstack <pid>`. Map the Hex ID from `top` to the thread dump to see exactly which line of code is executing.
* **APM/Metrics:** Check Grafana/Datadog to see if this correlates with a traffic spike. Check memory usage—a high CPU could actually be the JVM stuck in continuous Garbage Collection ("GC Thrashing") due to low memory.

**Fix:**
* **Immediate:** Scale horizontally (add more pods/instances) to shed load.
* **Root Cause:** Fix memory leaks, optimize slow code (regex, infinite loops), or move heavy synchronous tasks to Kafka.

## 2. Intermittent 500 Errors
**Investigation:**
* **Logs & Traces:** Check centralized logs (ELK/Datadog) filtering by `status: 500`. Look for patterns—do they happen during specific endpoints, specific payloads, or at a specific time?
* **Dependencies:** Are downstream services or databases timing out or dropping connections?

**Fix:**
* Fix underlying `NullPointerException`s or unhandled exceptions. 
* Add resilience: Use retry mechanisms for transient network glitches, and Circuit Breakers for external API failures.

## 3. Microservice Chain Failure
**Prevention:**
* **Circuit Breakers (Resilience4j):** If Service B is down, Service A should trip its circuit breaker and fail fast instead of waiting and exhausting its own threads.
* **Bulkhead Pattern:** Isolate resources. Give the connection pool for Service B its own dedicated threads, so if it fails, it doesn't take down the entire application.
* **Aggressive Timeouts:** Never make an external HTTP call without a strict read/connect timeout.

## 4. API Response Time Increased (200ms -> 3s)
**Investigation:**
* **Distributed Tracing (Jaeger/Zipkin):** Look at the trace for a slow request to see exactly which span (service call, DB query) took the longest.
* **Database Checks:** Look for slow queries, missing indexes, or table locks. 

**Fix:**
* Add missing DB indexes. Resolve JPA `N+1` query problems.
* Implement Redis caching for read-heavy endpoints. 
* If caused by a new release, initiate an immediate rollback.

## 5. Database Connections Exhausted
**Investigation:**
* **Connection Leaks:** Code failing to close connections.
* **Long Transactions:** External API calls being made inside `@Transactional` blocks, holding the connection open for seconds instead of milliseconds.
* **Traffic Spikes:** Legitimate load exceeding `maximumPoolSize`.

**Fix:**
* Move external HTTP calls/heavy logic *outside* of `@Transactional` blocks.
* Temporarily increase connection pool size (e.g., HikariCP).
* Add database Read Replicas and route `@Transactional(readOnly = true)` traffic to them.

## 6. Third-Party Service Timing Out
**Handling:**
* **Circuit Breaker:** Open the circuit if the error rate exceeds a threshold (e.g., 50%).
* **Fallbacks:** When the circuit is open, return a graceful response (e.g., cached data, empty list, or "Please try again later").
* **Async Processing:** If immediate response isn't required, queue the request in Kafka to be processed when the third-party service recovers.

## 7. Duplicate Transactions
**Prevention:**
* **Idempotency Keys:** Require API clients to send an `Idempotency-Key` header.
* **Validation:** Before processing, check Redis/DB if this key has already been processed. If yes, return the cached result.
* **Database Constraints:** Add a `UNIQUE` constraint on `(user_id, idempotency_key)` to prevent race conditions at the data layer.
* **Distributed Locks:** Use Redisson (Redis lock) around the user's transaction execution block.

## 8. Logs Too Large & Distributed
**Improvement:**
* **Centralization:** Pipe all logs to ELK (Elasticsearch, Logstash, Kibana), Loki, or Datadog.
* **Structured Logging:** Log in JSON format so tools can parse fields easily.
* **Correlation IDs:** Inject a `traceId` via SLF4J MDC (Mapped Diagnostic Context). Every log line for a single user request across all microservices will share this `traceId`.

## 9. Memory Leaks
**Detection & Fix:**
* **Detect:** APM dashboards will show a "saw-tooth" memory graph where the baseline memory continuously rises after Garbage Collection, eventually hitting `OutOfMemoryError`.
* **Analyze:** Capture a heap dump (`-XX:+HeapDumpOnOutOfMemoryError` or using `jmap`). 
* **Fix:** Open the dump in Eclipse MAT or VisualVM. Look for the "GC Roots". Common culprits: Unbounded `HashMap` caches (use Caffeine instead), unclosed Streams, and ThreadLocals.

## 10. Works Locally, Fails in Prod
**Approach:**
* **Configuration:** Check for environment-specific properties (DB URLs, feature flags, secret keys).
* **Data Volume:** Queries that run fast on a local DB with 10 rows will time out on a Prod DB with 10 million rows.
* **Network & Infra:** Prod might have strict Firewalls, Proxy restrictions, or tight K8s CPU/Memory limits causing OOM Kills.

## 11. Deployment Breaks One Feature
**Safe Rollback:**
* **Immediate:** If using Kubernetes, execute `kubectl rollout undo deployment/<name>` to instantly revert to the previous ReplicaSet.
* **Future Prevention:** Implement **Feature Flags** (e.g., LaunchDarkly). Wrap new features in toggles. If a feature breaks in prod, you simply turn the toggle off in a dashboard—no redeployment required.

## 12. Traffic Spikes 5x
**Scaling:**
* **Compute:** Horizontal Pod Autoscaler (HPA) in K8s to automatically add instances based on CPU utilization.
* **Database:** Scale up DB instance size, use Read Replicas.
* **Offload:** Heavily cache read APIs with Redis and CloudFront/CDN. Implement Rate Limiting to protect the backend. 

## 13. Inter-Service Network Latency
**Optimization:**
* **Placement:** Ensure chatty microservices are deployed in the same Availability Zone.
* **Protocol:** Switch from REST/JSON over HTTP/1.1 to **gRPC/Protobuf** over HTTP/2 for multiplexed, binary, high-speed communication.
* **Architecture:** Shift from synchronous requests to an Event-Driven Architecture (Kafka).

## 14. Tracing a Single Request Across Services
**Implementation:**
* Use **OpenTelemetry** or **Micrometer Tracing**.
* The API Gateway generates a unique `traceId` and passes it downstream via HTTP headers (like `traceparent` or `b3`).
* Each service creates a `spanId` for its specific chunk of work.
* The `[traceId, spanId]` is injected into the logging MDC. Send this data to Jaeger or Zipkin to visualize the entire request flow as a Gantt chart.

## 15. Inconsistent Data Across Services
**Handling:**
* **Never use 2PC:** Distributed transactions (Two-Phase Commit) lock DBs and destroy performance.
* **Saga Pattern:** Implement Orchestration or Choreography.
* **Compensating Transactions:** If Service A succeeds but Service B fails, Service B publishes an event that triggers Service A to execute a rollback (compensating) transaction.
* **Reconciliation:** Run nightly background cron jobs to compare databases, find mismatches, and flag them for manual or automated resolution.
