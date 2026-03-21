# 📦 Module 16: Nexus Repository

> All examples use the **ShopEase** e-commerce microservice project.

---

## 1. What is Nexus?

| Aspect | Details |
|---|---|
| **Type** | Artifactory server (stores build artifacts) |
| **Stores** | JAR files, WAR files, Docker images |
| **Also used as** | Maven Remote Repository (shared company libraries) |

### Nexus vs GitHub

| | GitHub | Nexus |
|---|---|---|
| **Stores** | Source code | Build artifacts (JARs/WARs) |
| **Type** | Version control | Artifact repository |
| **Purpose** | Code management | Artifact backup & shared libs |

---

## 2. Repository Types

| Type | Purpose | ShopEase Example |
|---|---|---|
| **Snapshot** | Dev builds (in progress) | `product-service-1.0-SNAPSHOT.jar` |
| **Release** | Production builds | `product-service-1.0-RELEASE.jar` |
| **Remote (Hosted)** | Shared company libraries | `shopease-commons.jar` |

---

## 3. Upload Artifacts from ShopEase

### Configure in `pom.xml`

```xml
<distributionManagement>
    <repository>
        <id>nexus</id>
        <name>ShopEase Release Repo</name>
        <url>http://nexus-server:8081/repository/shopease-release/</url>
    </repository>
    <snapshotRepository>
        <id>nexus</id>
        <name>ShopEase Snapshot Repo</name>
        <url>http://nexus-server:8081/repository/shopease-snapshot/</url>
    </snapshotRepository>
</distributionManagement>
```

### Configure in Maven `settings.xml`

```xml
<server>
    <id>nexus</id>
    <username>admin</username>
    <password>admin123</password>
</server>
```

### Deploy Command

```bash
mvn clean deploy
# → Uploads JAR to Nexus based on <version> in pom.xml:
#   1.0-SNAPSHOT → Snapshot repo
#   1.0-RELEASE  → Release repo
```

> `mvn deploy` internally runs: `compile → test → package → install → deploy`

---

## 4. Using Shared Libraries

### Add Remote Repo to `pom.xml`

```xml
<repositories>
    <repository>
        <id>nexus</id>
        <url>http://nexus-server:8081/repository/shopease-remote/</url>
    </repository>
</repositories>

<dependencies>
    <dependency>
        <groupId>com.shopease</groupId>
        <artifactId>shopease-commons</artifactId>
        <version>1.0</version>
    </dependency>
</dependencies>
```

---

*← [15 — Jenkins CI/CD](./15-jenkins-cicd.md) | [17 — AWS Cloud →](./17-aws-cloud.md)*
