# Interview Preparation — Master Index & Study Plan

## About This Guide

- **Comprehensive interview prep** for a **Senior Java Backend Developer (8 YOE)** in the **Payments domain**
- **Target companies**: FAANG (Google, Amazon, Meta, Apple, Netflix), Indian product companies (Flipkart, PhonePe, Razorpay, Paytm, Google Pay, CRED, Swiggy, Zerodha, etc.)
- **Tech stack covered**: Java 8/21, Spring Boot, Spring Data JPA, Hibernate, Oracle DB, Apache Kafka, Dynatrace, Payments domain
- **Philosophy**: This is your **one-stop resource** — no need to juggle multiple tutorials, YouTube playlists, or scattered notes. Everything you need is organized, indexed, and prioritized right here.

---

## Document Index

| # | File | Topics Covered | Priority |
|---|------|---------------|----------|
| 01 | [01-java-core.md](01-java-core.md) | Java 8 (Streams, Lambdas, Optional, CompletableFuture), Java 21 (Virtual Threads, Records, Pattern Matching, Sealed Classes), Memory Model, GC internals, Concurrency (locks, CAS, ThreadLocal, ExecutorService), Collections internals (HashMap, ConcurrentHashMap) | 🔴 Critical |
| 02 | [02-spring-boot.md](02-spring-boot.md) | IoC/DI internals, Auto-configuration, Starters, Actuator, Profiles, Security (JWT, OAuth2), Exception Handling, Validation, Filters/Interceptors, Testing | 🔴 Critical |
| 03 | [03-spring-data-jpa-hibernate.md](03-spring-data-jpa-hibernate.md) | Entity lifecycle, 1st/2nd level cache, N+1 problem, Fetch strategies, Locking (optimistic/pessimistic), @Transactional pitfalls, Batch inserts, JPQL vs Criteria | 🔴 Critical |
| 04 | [04-oracle-db.md](04-oracle-db.md) | Indexing, Execution plans, Partitioning, Materialized views, Query tuning, Deadlocks, HikariCP, PL/SQL | 🟡 High |
| 05 | [05-kafka.md](05-kafka.md) | Architecture, Producer/Consumer internals, Exactly-once, Idempotent producer, Transactions, Consumer groups, Rebalancing, DLQ, Schema Registry | 🔴 Critical |
| 06 | [06-system-design.md](06-system-design.md) | Idempotency, Saga/Outbox pattern, CAP, Rate limiting, Circuit breakers (Resilience4j), Consistent hashing, Event sourcing, CQRS | 🔴 Critical |
| 07 | [07-caching.md](07-caching.md) | Redis patterns, Cache-aside/Write-through/Write-behind, Cache stampede, TTL, Caffeine, Distributed caching | 🟡 High |
| 08 | [08-observability-dynatrace.md](08-observability-dynatrace.md) | Distributed tracing, OpenTelemetry, SLI/SLO/SLA, Dynatrace (PurePath, Smartscape), Log correlation, Alerting | 🟡 High |
| 09 | [09-payments-domain.md](09-payments-domain.md) | Payment lifecycle, PCI-DSS, Tokenization, 3DS, Reconciliation, Chargebacks, Idempotency in payments, Retry strategies | 🔴 Critical |
| 10 | [10-production-scenarios.md](10-production-scenarios.md) | OOM debugging, Kafka consumer lag, DB CPU spike, Duplicate charges, Transaction timeouts, Thread dumps, Heap dumps | 🔴 Critical |
| 11 | [11-coding-problems.md](11-coding-problems.md) | DSA patterns (sliding window, two pointers, graphs, DP), Java concurrency problems (producer-consumer, rate limiter, LRU cache, thread-safe singleton) | 🟡 High |
| 12 | [12-behavioral-leadership.md](12-behavioral-leadership.md) | STAR answers for 8 YOE: design decisions, mentoring, incident handling, tech debt, cross-team collaboration | 🟡 High |

---

## 4-Week Study Plan

### Week 1: Core Foundations

- **Day 1-2**: Java Core ([01-java-core.md](01-java-core.md)) — Java 8 features, Streams deep dive, functional interfaces
- **Day 3-4**: Java Core continued — Java 21 features, Memory model, GC, Concurrency
- **Day 5**: Spring Boot ([02-spring-boot.md](02-spring-boot.md)) — IoC/DI, auto-config, starters
- **Day 6**: Spring Boot continued — Security, testing, actuator
- **Day 7**: Review + practice coding problems from [11-coding-problems.md](11-coding-problems.md) (easy/medium)

### Week 2: Data Layer & Messaging

- **Day 1-2**: JPA/Hibernate ([03-spring-data-jpa-hibernate.md](03-spring-data-jpa-hibernate.md)) — full coverage
- **Day 3**: Oracle DB ([04-oracle-db.md](04-oracle-db.md)) — indexing, query tuning, execution plans
- **Day 4-5**: Kafka ([05-kafka.md](05-kafka.md)) — architecture, producer/consumer, exactly-once
- **Day 6**: Caching ([07-caching.md](07-caching.md)) — Redis, Caffeine, patterns
- **Day 7**: Review + coding problems (medium)

### Week 3: System Design & Domain

- **Day 1-2**: System Design ([06-system-design.md](06-system-design.md)) — patterns, idempotency, Saga
- **Day 3-4**: Payments Domain ([09-payments-domain.md](09-payments-domain.md)) — full lifecycle, PCI-DSS
- **Day 5**: Observability ([08-observability-dynatrace.md](08-observability-dynatrace.md)) — tracing, Dynatrace
- **Day 6**: Production Scenarios ([10-production-scenarios.md](10-production-scenarios.md)) — debugging walkthroughs
- **Day 7**: Review + system design practice (design a payment gateway)

### Week 4: Polish & Mock Interviews

- **Day 1-2**: Coding Problems ([11-coding-problems.md](11-coding-problems.md)) — hard problems, concurrency
- **Day 3**: Behavioral ([12-behavioral-leadership.md](12-behavioral-leadership.md)) — prepare STAR stories
- **Day 4**: Mock system design interview (pick 2 problems from [06-system-design.md](06-system-design.md))
- **Day 5**: Mock coding interview (pick 3 problems from [11-coding-problems.md](11-coding-problems.md))
- **Day 6**: Weak areas review — revisit flagged questions
- **Day 7**: Rest + light review of index

---

## How to Use These Documents

- Each document has questions organized: **Conceptual → Internals → Production/Practical → Code Snippets**
- Questions marked with 🔥 are **most frequently asked**
- Questions marked with 💎 are **differentiators** — knowing these sets you apart
- Mermaid diagrams are included — render them in any Markdown viewer (VS Code, GitHub, etc.)
- Code snippets are production-grade Java — you can discuss them as "code I've written in my projects"

---

## Quick Reference: Top 20 Questions You MUST Know

1. How does HashMap work internally? What changed in Java 8?
2. Explain CompletableFuture vs Future. When would you use each?
3. What are Virtual Threads? How do they differ from platform threads?
4. How does Spring Boot auto-configuration work?
5. Explain @Transactional propagation levels with real examples
6. What is the N+1 problem? How do you solve it?
7. Optimistic vs Pessimistic locking — when to use which?
8. How do you tune a slow Oracle query? Walk through your approach.
9. Explain Kafka's exactly-once semantics end-to-end
10. How do you handle consumer rebalancing in Kafka?
11. Design an idempotent payment API
12. Explain the Saga pattern with a real payments example
13. How does circuit breaker work? Explain Resilience4j states
14. Cache-aside vs Write-through — tradeoffs?
15. How does Dynatrace PurePath tracing work?
16. What is PCI-DSS? How does tokenization work?
17. How would you debug an OOM error in production?
18. How would you handle duplicate payment charges?
19. Implement a thread-safe LRU cache in Java
20. Tell me about a time you made a critical design decision (STAR)
