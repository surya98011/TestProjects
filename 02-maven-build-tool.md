# 🔨 Module 2: Apache Maven — Build Tool

> All examples use the **ShopEase** e-commerce microservice project.

---

## 📑 Table of Contents

- [1. What is Maven?](#1-what-is-maven)
- [2. Maven Setup](#2-maven-setup)
- [3. Maven Terminology](#3-maven-terminology)
- [4. Maven Project Structure](#4-maven-project-structure)
- [5. The POM File — ShopEase Example](#5-the-pom-file--shopease-example)
- [6. Maven Dependencies & Transitive Deps](#6-maven-dependencies--transitive-deps)
- [7. Maven Repositories](#7-maven-repositories)
- [8. Maven Goals & Build Lifecycle](#8-maven-goals--build-lifecycle)
- [9. Dependency Scopes](#9-dependency-scopes)
- [10. Multi-Module Project](#10-multi-module-project)
- [11. Cheat Sheet](#11-cheat-sheet)

---

## 1. What is Maven?

Maven is a **build automation tool** for Java projects. It handles the entire build lifecycle:

```
📥 Download Dependencies → 🔨 Compile → 🧪 Test → 📦 Package → 🚀 Deploy
```

> **Analogy:** Maven is like a **recipe manager** — you list the ingredients (dependencies) in `pom.xml`, and Maven downloads them, compiles your code, runs tests, and packages everything into a JAR/WAR.

---

## 2. Maven Setup

```bash
# Step 1: Set JAVA_HOME
JAVA_HOME = C:\Program Files\Java\jdk-17

# Step 2: Download Maven from https://maven.apache.org/download.cgi
# Step 3: Set MAVEN_HOME
MAVEN_HOME = C:\apache-maven-3.9.6

# Step 4: Add to PATH
Path = %JAVA_HOME%\bin;%MAVEN_HOME%\bin

# Step 5: Verify
mvn -v
```

---

## 3. Maven Terminology

| Term | Meaning | ShopEase Example |
|---|---|---|
| **Archetype** | Project template | `maven-archetype-quickstart` |
| **groupId** | Organization identifier | `com.shopease` |
| **artifactId** | Project/module name | `product-service` |
| **version** | Release stage | `1.0-SNAPSHOT` / `1.0-RELEASE` |
| **packaging** | Output type | `jar` (microservice) / `war` (legacy web) |
| **dependency** | External library | `spring-boot-starter-web` |
| **repository** | Where deps are stored | Central / Remote / Local |
| **goal** | Build action | `clean`, `compile`, `test`, `package` |

---

## 4. Maven Project Structure

```
product-service/
├── pom.xml                              ← Project Object Model
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/shopease/product/
│   │   │       ├── ProductServiceApplication.java
│   │   │       ├── controller/
│   │   │       │   └── ProductController.java
│   │   │       ├── service/
│   │   │       │   └── ProductService.java
│   │   │       ├── repository/
│   │   │       │   └── ProductRepository.java
│   │   │       └── entity/
│   │   │           └── Product.java
│   │   └── resources/
│   │       └── application.yml
│   └── test/
│       └── java/
│           └── com/shopease/product/
│               └── service/
│                   └── ProductServiceTest.java
└── target/                              ← Build output (auto-generated)
    └── product-service.jar
```

---

## 5. The POM File — ShopEase Example

Here's the `pom.xml` for ShopEase's **product-service**:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <!-- ✅ Parent: Spring Boot Starter Parent -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.3</version>
    </parent>

    <!-- ✅ Project Coordinates -->
    <groupId>com.shopease</groupId>
    <artifactId>product-service</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <properties>
        <java.version>17</java.version>
        <spring-cloud.version>2023.0.0</spring-cloud.version>
    </properties>

    <!-- ✅ Dependencies -->
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>

    <!-- ✅ Spring Cloud BOM (Bill of Materials) -->
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <!-- ✅ Custom Build Config -->
    <build>
        <finalName>product-service</finalName>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## 6. Maven Dependencies & Transitive Deps

### How Dependencies Resolve

```
pom.xml declares dependency
        │
        ▼
📁 Local Repo (~/.m2) ──(not found)──▶ 🌐 Maven Central
        │                                      │
        │                               (downloads)
        ◀──────────────────────────────────────┘
        │
    Added to classpath → 🚀 Application runs
```

### Transitive Dependencies (Auto-Downloaded)

```
spring-boot-starter-web
├── spring-boot-starter           (auto)
├── spring-boot-starter-tomcat    (auto)
├── spring-web                    (auto)
└── spring-webmvc                 (auto)
```

### Excluding Unwanted Transitive Dependencies

```xml
<!-- ShopEase API Gateway: Exclude Tomcat to use Netty -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

---

## 7. Maven Repositories

| Type | Description | Location |
|---|---|---|
| **Local** | Cache on your machine | `C:\Users\<name>\.m2\repository` |
| **Central** | Public Apache repo | `repo.maven.apache.org` |
| **Remote** | Company-private repo | Nexus / JFrog (see Module 16) |

> **💡 Pro Tip:** In real projects, companies use **Nexus/JFrog** remote repos. You configure access in Maven's `settings.xml` to share internal libraries across teams.

---

## 8. Maven Goals & Build Lifecycle

```bash
# Individual goals
mvn clean          # Delete /target folder
mvn compile        # Compile source code
mvn test           # Run unit tests
mvn package        # Create JAR/WAR
mvn install        # Install to local repo (~/.m2)
mvn deploy         # Upload to remote repo (Nexus)

# Combined (each includes all prior phases)
mvn clean package  # clean → compile → test → package
mvn clean install  # clean → compile → test → package → install
mvn clean deploy   # clean → compile → test → package → install → deploy
```

### ShopEase Build Example

```bash
cd ShopEase/product-service
mvn clean package
# Output: target/product-service.jar

java -jar target/product-service.jar
```

---

## 9. Dependency Scopes

| Scope | When Available | ShopEase Example |
|---|---|---|
| `compile` (default) | Compile + Test + Runtime | `spring-boot-starter-web` |
| `runtime` | Test + Runtime only | `mysql-connector-j` |
| `test` | Test only | `spring-boot-starter-test` |
| `provided` | Compile + Test (not packaged) | `lombok` |
| `system` | Like provided, from local path | Rarely used |
| `import` | For BOM dependency management | `spring-cloud-dependencies` |

---

## 10. Multi-Module Project

ShopEase as a Maven multi-module project:

```xml
<!-- shopease-parent/pom.xml -->
<groupId>com.shopease</groupId>
<artifactId>shopease-parent</artifactId>
<version>1.0-SNAPSHOT</version>
<packaging>pom</packaging>

<modules>
    <module>service-registry</module>
    <module>config-server</module>
    <module>api-gateway</module>
    <module>product-service</module>
    <module>order-service</module>
    <module>user-service</module>
    <module>notification-service</module>
</modules>
```

> Running `mvn clean package` on the parent builds **all** modules in dependency order automatically.

---

## 11. Cheat Sheet

```
┌──────────────────────────────────────────────────────────┐
│                   MAVEN CHEAT SHEET                       │
├──────────────────────────────────────────────────────────┤
│ mvn -v                    → Check Maven version           │
│ mvn clean package         → Full build cycle              │
│ mvn clean install         → Build + install to .m2        │
│ mvn clean deploy          → Build + upload to Nexus       │
│ mvn dependency:tree       → Show dependency tree          │
│ mvn sonar:sonar           → Run SonarQube analysis        │
├──────────────────────────────────────────────────────────┤
│ Key POM Tags:                                             │
│   <groupId>      → com.shopease                           │
│   <artifactId>   → product-service                        │
│   <version>      → 1.0-SNAPSHOT / 1.0-RELEASE             │
│   <packaging>    → jar / war / pom                        │
│   <finalName>    → Custom JAR name                        │
│   <dependencies> → Required libraries                     │
│   <build>        → Plugins & config                       │
└──────────────────────────────────────────────────────────┘
```

---

*← [01 — Introduction](./01-introduction-and-industry.md) | [03 — Git & GitHub →](./03-git-and-github.md)*
