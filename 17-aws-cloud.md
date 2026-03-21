# ☁️ Module 17: AWS Cloud

> All examples use the **ShopEase** e-commerce microservice project.

---

## 📑 Table of Contents

- [1. Cloud Computing Overview](#1-cloud-computing-overview)
- [2. AWS Services Map](#2-aws-services-map)
- [3. EC2 — Virtual Machines](#3-ec2--virtual-machines)
- [4. Linux Commands Quick Reference](#4-linux-commands-quick-reference)
- [5. Deploy ShopEase to EC2](#5-deploy-shopease-to-ec2)
- [6. AWS RDS — Managed Database](#6-aws-rds--managed-database)
- [7. AWS S3 — Storage](#7-aws-s3--storage)
- [8. AWS IAM — User Management](#8-aws-iam--user-management)
- [9. Load Balancer & Auto Scaling](#9-load-balancer--auto-scaling)

---

## 1. Cloud Computing Overview

> Delivering IT resources over the internet with **pay-as-you-go** pricing.

### On-Premises vs Cloud

| Aspect | On-Premises | Cloud (AWS) |
|---|---|---|
| **Hardware** | Buy & maintain servers | Rent virtual servers |
| **Cost** | Large upfront investment | Pay per hour/usage |
| **Scaling** | Buy more hardware (weeks) | Click a button (minutes) |
| **Maintenance** | Your responsibility | AWS manages |
| **Security** | Your responsibility | Shared responsibility model |

---

## 2. AWS Services Map

| # | Service | Purpose | ShopEase Usage |
|---|---|---|---|
| 1 | **EC2** | Virtual servers | Run microservices |
| 2 | **RDS** | Managed database | MySQL for ShopEase |
| 3 | **S3** | Unlimited storage | Product images |
| 4 | **IAM** | User & permissions | Team access control |
| 5 | **ELB** | Load balancing | Distribute traffic |
| 6 | **Route 53** | DNS / Domain mapping | `shopease.com` |
| 7 | **EKS** | Managed Kubernetes | K8S cluster |

---

## 3. EC2 — Virtual Machines

### Key Components

| Component | Description |
|---|---|
| **AMI** | OS template (Amazon Linux, Ubuntu, Windows) |
| **Instance Type** | Machine size (`t2.micro` = 1GB free tier) |
| **Key Pair** | SSH authentication (public key on AWS, private key with you) |
| **Security Group** | Firewall rules (inbound/outbound) |
| **EBS Volume** | Hard disk (8GB default for Linux, 30GB for Windows) |

### Common Security Group Ports

| Protocol | Port | Use |
|---|---|---|
| SSH | 22 | Linux terminal access |
| RDP | 3389 | Windows remote desktop |
| HTTP | 80 | Web server |
| HTTPS | 443 | Secure web |
| MySQL | 3306 | Database |
| Custom | 8080-8084 | ShopEase services |

### IP Types

| Type | Behavior | Use |
|---|---|---|
| **Private IP** | Fixed, internal only | Service-to-service within VPC |
| **Public IP** | Changes on restart | External access (temporary) |
| **Elastic IP** | Fixed public IP (paid) | Production servers |

---

## 4. Linux Commands Quick Reference

| Command | Purpose |
|---|---|
| `whoami` | Display current username |
| `pwd` | Present working directory |
| `ls -ltr` | List files (latest last) |
| `cd <dir>` | Change directory |
| `mkdir <name>` | Create directory |
| `touch <file>` | Create empty file |
| `cat <file>` | Display file content |
| `cp <src> <dst>` | Copy file |
| `mv <old> <new>` | Rename/move |
| `rm -r <dir>` | Delete directory |
| `head -n 20 file` | First 20 lines |
| `tail -f file` | Follow file (live logs) |
| `grep -i "error" file` | Search pattern |
| `vi <file>` | Edit file (press `i` to edit, `:wq` to save) |
| `sudo yum install <pkg>` | Install package (Amazon Linux) |
| `sudo apt install <pkg>` | Install package (Ubuntu) |

---

## 5. Deploy ShopEase to EC2

### Step-by-step

```bash
# 1. Connect to EC2 via SSH
ssh -i shopease-key.pem ec2-user@<public-ip>

# 2. Install dependencies
sudo yum install git -y
sudo yum install maven -y
sudo yum install docker -y
sudo service docker start

# 3. Clone and build
git clone https://github.com/shopease/product-service.git
cd product-service
mvn clean package

# 4. Run directly (option A)
java -jar target/product-service.jar

# 5. Or run with Docker (option B)
docker build -t shopease/product-service .
docker run -d -p 8081:8081 shopease/product-service

# 6. Enable port 8081 in EC2 Security Group Inbound Rules!

# 7. Access:
# http://<ec2-public-ip>:8081/api/products
```

### User Data (Auto-run script on EC2 launch)

```bash
#!/bin/bash
sudo yum install docker -y
sudo service docker start
docker run -d -p 8081:8081 shopease/product-service:1.0
```

> **📝 Note:** User Data script runs **only once** when the EC2 instance first starts.

---

## 6. AWS RDS — Managed Database

### Create MySQL for ShopEase

| Setting | Value |
|---|---|
| Engine | MySQL 8.0 |
| Template | Free Tier |
| DB Instance ID | `shopease-db` |
| Username | `admin` |
| Password | `ShopEase2024` |
| Public Access | Yes (for dev) |

### ShopEase `application.yml` with RDS

```yaml
spring:
  datasource:
    url: jdbc:mysql://shopease-db.xxxx.ap-south-1.rds.amazonaws.com:3306/shopease
    username: admin
    password: ShopEase2024
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
```

> **⚠️ Important:** Always delete RDS instances when not in use to avoid billing!

---

## 7. AWS S3 — Storage

| Aspect | Details |
|---|---|
| **S3** | Simple Storage Service — unlimited object storage |
| **Structure** | Buckets → Objects (files) |
| **ShopEase Use** | Store product images |
| **Pricing** | Pay for storage + retrieval |

### Static Website Hosting with S3

1. Create bucket → Enable ACL → Allow Public Access
2. Upload `index.html` and `error.html`
3. Enable Static Website Hosting in bucket properties
4. Access via bucket URL

---

## 8. AWS IAM — User Management

| Concept | Description |
|---|---|
| **Root User** | Super admin (has access to everything) |
| **IAM User** | Limited permissions (for daily work) |
| **Policies** | Permission rules (EC2FullAccess, S3FullAccess) |
| **Groups** | Collection of users with same permissions |

> **Best Practice:** Never use root account for daily activities. Create IAM users with specific policies.

---

## 9. Load Balancer & Auto Scaling

### Application Load Balancer (ALB)

```
Incoming requests ──▶ Load Balancer (ALB)
                          │
                    ┌─────┼─────┐
                    ▼     ▼     ▼
                 EC2-1  EC2-2  EC2-3
              (product) (product) (product)
```

| Advantage | Description |
|---|---|
| Traffic distributed | Round-robin across servers |
| Reduced burden | No single server overloaded |
| High availability | If one server crashes, others handle traffic |
| Faster responses | Less load per server |

### Auto Scaling

> Automatically adds/removes EC2 instances based on demand.

| Feature | Benefit |
|---|---|
| **Fault tolerance** | Replace unhealthy instances |
| **Cost management** | Scale down when traffic drops |
| **High availability** | Scale up during peak hours |

---

*← [16 — Nexus](./16-nexus-repository.md) | [18 — Angular Frontend →](./18-angular-frontend.md)*
