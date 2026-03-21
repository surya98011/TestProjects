# 🔍 Module 7: SonarQube — Code Quality

> All examples use the **ShopEase** e-commerce microservice project.

---

## 📑 Table of Contents

- [1. What is SonarQube?](#1-what-is-sonarqube)
- [2. Types of Sonar Issues](#2-types-of-sonar-issues)
- [3. Setup & Integration](#3-setup--integration)
- [4. Common Code Smells & Fixes](#4-common-code-smells--fixes)
- [5. Real-Time Workflow](#5-real-time-workflow)

---

## 1. What is SonarQube?

| Aspect | Details |
|---|---|
| **Purpose** | Static code analysis, code quality check |
| **Developed in** | Java |
| **Supports** | 30+ programming languages |
| **Editions** | Community (free) / Enterprise (paid) |
| **What it finds** | Bugs, vulnerabilities, code smells, duplicates |

---

## 2. Types of Sonar Issues

| Issue Type | Description | Example |
|---|---|---|
| 🐛 **Bugs** | Code that will behave incorrectly | NullPointerException risk |
| 🔓 **Vulnerabilities** | Security weaknesses | Hardcoded passwords |
| 💨 **Code Smells** | Maintainability issues | Unused imports, string literals |
| 🔁 **Duplicates** | Repeated code blocks | Copy-pasted logic |
| 📊 **Coverage** | % of code tested by JUnit | Below 80% = needs improvement |

---

## 3. Setup & Integration

### Configure in ShopEase `pom.xml`

```xml
<properties>
    <sonar.host.url>http://sonar-server:9000/</sonar.host.url>
    <sonar.login>squ_your_token_here</sonar.login>
</properties>
```

### Run Analysis

```bash
cd ShopEase/product-service
mvn sonar:sonar
# → Check results at http://sonar-server:9000/
```

> **💡 Pro Tip:** Use Sonar **tokens** instead of username/password for security.

---

## 4. Common Code Smells & Fixes

### ShopEase Before & After

| # | Sonar Issue | ❌ Bad Code | ✅ Fixed Code |
|---|---|---|---|
| 1 | Use StringBuilder | `StringBuffer sb = new StringBuffer()` | `StringBuilder sb = new StringBuilder()` |
| 2 | Reuse Random | `new Random()` inside method | Class-level `private final Random random = new Random()` |
| 3 | Constants class | Public constructor | `private ProductConstants() {}` |
| 4 | Unused imports | `import java.util.Date;` (not used) | Remove it |
| 5 | String literals | `"ACTIVE"` repeated 5 times | `private static final String ACTIVE = "ACTIVE";` |
| 6 | Lambda braces | `list.forEach(p -> { return p; })` | `list.forEach(p -> p)` |
| 7 | Null handling | `product.getName().equals("x")` | `"x".equals(product.getName())` |

### ShopEase ProductConstants — SonarQube Compliant

```java
package com.shopease.product.constants;

public final class ProductConstants {

    // ✅ Private constructor — prevents instantiation
    private ProductConstants() {
        throw new UnsupportedOperationException("Constants class");
    }

    public static final String STATUS_ACTIVE = "ACTIVE";
    public static final String STATUS_INACTIVE = "INACTIVE";
    public static final String CACHE_KEY_ALL_PRODUCTS = "ALL_PRODUCTS";
}
```

---

## 5. Real-Time Workflow

```
Developer writes code
        │
        ▼
Pushes to Git → Jenkins pipeline triggers
        │
        ▼
Jenkins runs: mvn clean package sonar:sonar
        │
        ▼
SonarQube generates report
        │
        ▼
Developer checks Sonar Dashboard
        │
        ▼
Fixes sonar issues → Commits → Pipeline re-runs
```

> **📝 Note:** In most companies, SonarQube is integrated with Jenkins. DevOps sets up the pipeline; developers check the dashboard and fix reported issues.

---

*← [06 — JUnit Testing](./06-junit-testing.md) | [08 — Microservices Architecture →](./08-microservices-architecture.md)*
