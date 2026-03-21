# 📘 Module 1: Introduction & Software Industry Overview

> **JRTP — Java Real-Time Project (Fullstack)**
> All code examples reference the **ShopEase** e-commerce microservice platform.

---

## 📑 Table of Contents

- [1. What is JRTP?](#1-what-is-jrtp)
- [2. Types of Software Companies](#2-types-of-software-companies)
- [3. Types of Software Projects](#3-types-of-software-projects)
- [4. Project Teams: Onshore vs Offshore](#4-project-teams-onshore-vs-offshore)
- [5. Real-Time Tools Used in Projects](#5-real-time-tools-used-in-projects)
- [6. ShopEase — Our Reference Project](#6-shopease--our-reference-project)

---

## 1. What is JRTP?

JRTP stands for **Java Real-Time Project** — a fullstack training program that simulates real corporate software development workflows.

### What You'll Master

| Area | Topics |
|---|---|
| **Build & Dependency** | Maven, Gradle |
| **Version Control** | Git, GitHub, Branching strategies |
| **Code Quality** | SonarQube, JUnit, Mockito, JaCoCo |
| **Logging** | SLF4J, Logback |
| **Microservices** | Eureka, API Gateway, FeignClient, Circuit Breaker, Config Server |
| **Messaging** | Apache Kafka |
| **Caching** | Redis |
| **Security** | Spring Security, JWT, OAuth 2.0 |
| **DevOps** | Docker, Kubernetes, Jenkins CI/CD |
| **Cloud** | AWS (EC2, S3, RDS, IAM) |
| **Frontend** | Angular |
| **Testing** | JUnit, JMeter (performance) |
| **Management** | JIRA, Agile/Scrum |

---

## 2. Types of Software Companies

### 🏢 Product-Based Companies

> Build their own products and sell them to customers.

**Examples:** Google, Microsoft, Amazon, Facebook, Oracle, IBM

| Aspect | Details |
|---|---|
| **Interview Focus** | Coding tests, Data Structures, Algorithms, System Design |
| **Fresher Salary** | ₹15–20 LPA |
| **Work Style** | R&D focused, innovation-driven |

### 🏗️ Service-Based Companies

> Develop projects based on **client requirements**.

**Examples:** TCS, Infosys, Cognizant, Capgemini, Tech Mahindra, Wipro, HCL, Deloitte

| Aspect | Details |
|---|---|
| **Interview Focus** | Core Java, Spring Boot, Microservices, Cloud, Tools, Project experience |
| **Fresher Salary** | ₹3.6 LPA |
| **Experienced Formula** | ~Years × 5-6 LPA |

### 🤝 Outsourcing/Staffing Companies

> Provide trained employees to other companies.

| Aspect | Details |
|---|---|
| **Fresher Salary** | ₹2–3 LPA |
| **Process** | 2-3 months on-job training → placed at client site for higher rate |

> **💡 Pro Tip:** Most Java developers start at service-based companies and later transition to product companies after gaining 2-3 years of experience.

---

## 3. Types of Software Projects

| Project Type | Frequency | Description |
|---|---|---|
| 🆕 **Scratch Development** | ~15% | Build from ground zero (e.g., building ShopEase from scratch) |
| 🔧 **Support / Enhancement** | ~75% | Bug fixing, change requests on existing code — *most common* |
| 🔄 **Migration** | ~10% | Upgrade from old to new tech (e.g., monolith → microservices) |

---

## 4. Project Teams: Onshore vs Offshore

### Team Structure

```
┌─────────────────────────────────────────────────────────────┐
│                     🏢 CLIENT                                │
│   (Provides requirements, approves deliverables)             │
└──────────────────────┬──────────────────────────────────────┘
                       │
         ┌─────────────▼──────────────┐
         │   🌎 ONSHORE TEAM          │
         │   (At client location)      │
         │   • Understand business     │
         │   • Prepare BRD / FDD       │
         │   • Get client approvals    │
         │   • Explain reqs to offshore│
         └─────────────┬──────────────┘
                       │
         ┌─────────────▼──────────────┐
         │   💻 OFFSHORE TEAM         │
         │   (Company office)          │
         │   • Design & Develop        │
         │   • Test & Debug            │
         │   • Deploy & Support        │
         └────────────────────────────┘
```

### Key Visa Types (for reference)

| Visa | Purpose | Duration |
|---|---|---|
| B1 | Business visit | 2–3 months |
| L1 | Intra-company transfer | 2 years |
| H1B | Work permit (lottery) | 3 years |
| F1 | Student | Until studies complete |

---

## 5. Real-Time Tools Used in Projects

| # | Category | Tools | Purpose |
|---|---|---|---|
| 1 | **Build Tools** | Maven, Gradle | Automate compile → test → package |
| 2 | **Version Control** | Git, GitHub, Bitbucket | Source code management |
| 3 | **Code Review** | SonarQube | Static analysis, code quality |
| 4 | **Unit Testing** | JUnit 5, Mockito | Test individual components |
| 5 | **Logging** | SLF4J + Logback | Runtime behavior tracking |
| 6 | **Log Monitoring** | ELK Stack, Splunk | Centralized log analysis |
| 7 | **Performance Test** | JMeter, LoadRunner | Stability & throughput testing |
| 8 | **Project Management** | JIRA | Story tracking, sprint management |
| 9 | **Containerization** | Docker | Package app + dependencies |
| 10 | **Orchestration** | Kubernetes (K8S) | Manage containers at scale |
| 11 | **CI/CD** | Jenkins | Automate build & deployment |
| 12 | **Artifact Storage** | Nexus, JFrog | Store build artifacts (JARs/WARs) |
| 13 | **Messaging** | Apache Kafka | Event-driven, async communication |
| 14 | **Caching** | Redis | Reduce DB calls, faster reads |
| 15 | **API Testing** | Postman | Manual REST API testing |
| 16 | **API Docs** | Swagger / OpenAPI | Auto-generate API documentation |
| 17 | **Reports** | Apache POI, iText | Generate Excel & PDF files |

---

## 6. ShopEase — Our Reference Project

Throughout this entire documentation, every code example comes from **ShopEase** — a real-time e-commerce microservice platform.

### Architecture Overview

```
                    ┌───────────────────────┐
                    │  🌐 Angular Frontend  │
                    │      Port: 4200       │
                    └──────────┬────────────┘
                               │
                    ┌──────────▼────────────┐
                    │   🚪 API Gateway      │
                    │   Port: 8080          │
                    │   (JWT Filter)        │
                    └──┬────────┬───────┬───┘
                       │        │       │
          ┌────────────▼┐  ┌────▼────┐  ┌▼───────────┐
          │ 📦 Product  │  │🛒 Order │  │👤 User     │
          │ Service     │  │ Service │  │ Service    │
          │ Port: 8081  │  │Port:8082│  │Port: 8083  │
          └──────┬──────┘  └────┬────┘  └────────────┘
                 │              │
          ┌──────▼──────┐  ┌───▼──────────────────┐
          │ ⚡ Redis    │  │📧 Notification Service│
          │ Cache       │  │   Port: 8084          │
          └─────────────┘  │   (Kafka Consumer)    │
                           └───────────────────────┘
       ┌──────────────────────────────────────────────┐
       │  🔍 Service Registry (Eureka) — Port: 8761   │
       │  ⚙️ Config Server — Port: 8888                │
       └──────────────────────────────────────────────┘
```

### Tech Stack

| Layer | Technology |
|---|---|
| **Backend** | Java 17, Spring Boot 3.x, Spring Cloud |
| **Database** | MySQL 8 |
| **Cache** | Redis |
| **Messaging** | Apache Kafka |
| **Security** | Spring Security + JWT |
| **Frontend** | Angular 17 |
| **DevOps** | Docker, Kubernetes, Jenkins |
| **Cloud** | AWS (EC2, S3, RDS) |

> **📝 Note:** Every subsequent documentation file uses ShopEase classes, configurations, and workflows as examples. This ensures you learn concepts through one cohesive project rather than disconnected snippets.

---

*Next: [02 — Maven Build Tool →](./02-maven-build-tool.md)*
