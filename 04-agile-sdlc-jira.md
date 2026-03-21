# 📋 Module 4: Agile, SDLC & JIRA

> All examples use the **ShopEase** e-commerce microservice project.

---

## 📑 Table of Contents

- [1. SDLC — Software Development Life Cycle](#1-sdlc)
- [2. Waterfall vs Agile](#2-waterfall-vs-agile)
- [3. Agile Methodology](#3-agile-methodology)
- [4. Agile Team Structure](#4-agile-team-structure)
- [5. Agile Terminology](#5-agile-terminology)
- [6. JIRA — Project Management](#6-jira)
- [7. Communication Templates](#7-communication-templates)
- [8. Corporate Abbreviations](#8-corporate-abbreviations)

---

## 1. SDLC

SDLC (Software Development Life Cycle) is the process followed to develop software:

```
Requirements → Analysis → Design → Development → Testing → Deployment → Support
```

| Methodology | Status | Approach |
|---|---|---|
| **Waterfall** | Outdated | Sequential, one phase at a time |
| **Agile** | Trending | Iterative, continuous delivery |

---

## 2. Waterfall vs Agile

| Aspect | Waterfall | Agile |
|---|---|---|
| **Approach** | Linear / Sequential | Iterative / Incremental |
| **Direction** | Forward only | Flexible, can adapt |
| **Requirements** | Fixed upfront | Evolving |
| **Budget** | Fixed | Flexible |
| **Client Involvement** | Very low (sees project at end) | Very high (feedback every sprint) |
| **Risk** | High (money/time wasted if client unhappy) | Low (continuous feedback) |
| **Delivery** | One big release | Multiple small releases |

---

## 3. Agile Methodology

> Agile is an **iterative approach** where planning, development, testing, and deployment happen continuously.

### ShopEase Sprint Example

```
Sprint 1 (Week 1-2):  Product Service + Service Registry
Sprint 2 (Week 3-4):  Order Service + Kafka Integration
Sprint 3 (Week 5-6):  User Service + JWT Security
Sprint 4 (Week 7-8):  API Gateway + Notification Service
Sprint 5 (Week 9-10): Angular Frontend + Integration Testing
Sprint 6 (Week 11-12): Docker + K8S Deployment + Performance Testing
```

---

## 4. Agile Team Structure

| Role | Responsibility | Count |
|---|---|---|
| **Product Owner** | Client deliverables, priorities | 1 |
| **Scrum Master** | Team management, remove blockers | 1 |
| **Tech Lead** | Technical decisions, architecture | 1 |
| **Developers** | Write code, fix bugs | 3-5 |
| **Testers** | QA, write test cases | 1-2 |

> **📝 Note:** Industry standard agile team size is **7-10 members**. One project can have multiple agile teams.

---

## 5. Agile Terminology

### 📌 Backlog Grooming
Meeting to discuss pending work in the project. Every pending task becomes a **story** in JIRA.

### 📌 Story & Story Points

| Story Points | Duration | Example (ShopEase) |
|---|---|---|
| 3 points | 1 day | Add search endpoint to Product Service |
| 5 points | 2 days | Implement Kafka producer in Order Service |
| 8 points | 3 days | Build JWT authentication in User Service |

### 📌 Sprint Planning
Meeting to select priority stories for the upcoming sprint (industry standard: **2 weeks**).

### 📌 Sprint
A fixed set of stories targeted for completion in a given timeframe.

### 📌 Daily Scrum Call
Every team member shares:
1. What am I working on?
2. Status of my story?
3. When will it complete?
4. Any blockers?

### 📌 Retrospective (Monthly)
Review meeting to discuss:
- ✅ What went well
- ❌ What went wrong
- 📖 Lessons learned
- 💡 New ideas

---

## 6. JIRA

JIRA (by Atlassian) is used for:
- Creating & assigning stories
- Tracking story status (To Do → In Progress → Done)
- Sprint planning & reports
- Bug reporting

### ShopEase JIRA Story Example

```
┌──────────────────────────────────────────────────────┐
│  STORY: SE-42                                         │
│  Title: Implement product search by category          │
│  Sprint: Sprint 2                                     │
│  Story Points: 5                                      │
│  Assignee: Surya Pandey                               │
│  Status: In Progress                                  │
│  Description:                                         │
│    As a customer, I want to search products by         │
│    category so I can find items quickly.               │
│  Acceptance Criteria:                                  │
│    - GET /api/products?categoryId={id} works          │
│    - Returns paginated results                        │
│    - Unit tests added (>80% coverage)                 │
└──────────────────────────────────────────────────────┘
```

---

## 7. Communication Templates

### Daily Scrum Update Email

```
Subject: Scrum Update | Surya Pandey

Hi Steve,

Greetings for the day!

Today I am unable to join scrum call due to personal work.
Please find my work status below:

• Working on story SE-42 — Implement Product Search (IN PROGRESS)
• Targeted completion: Tomorrow EOD
• No blockers at the moment.

Thanks,
Surya
```

### Leave Request Email

```
Subject: OOO Request | Surya Pandey

Hi Steve,

I will be OOO from 10-Mar to 14-March due to a family function.
Kindly approve my leave request.  

Thanks,
Surya
```

### Jenkins Pipeline Request Email

```
To: devops-team@company.com
Subject: Jenkins Pipeline Creation Request | SE-42

Hi DevOps Team,

Please create a Jenkins CI/CD pipeline for 'order-service' microservice.
Git Hub Repo: https://github.com/shopease/order-service.git

@manager: Kindly approve this request.

Thanks,
Surya
```

---

## 8. Corporate Abbreviations

| Abbreviation | Meaning |
|---|---|
| OOO | Out of Office |
| ASAP | As Soon As Possible |
| FYI | For Your Information |
| PFB | Please Find Below |
| EOD | End of Day |
| DM | Direct Message |
| DL | Distribution List (email group) |
| KT | Knowledge Transfer |
| WFH | Work From Home |
| POC | Proof of Concept |

---

*← [03 — Git & GitHub](./03-git-and-github.md) | [05 — Logging & Monitoring →](./05-logging-and-monitoring.md)*
