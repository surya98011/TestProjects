# 🐳 Module 13: Docker & Containerization

> All examples use the **ShopEase** e-commerce microservice project.

---

## 📑 Table of Contents

- [1. What is Docker?](#1-what-is-docker)
- [2. Docker Architecture](#2-docker-architecture)
- [3. Docker Commands Cheat Sheet](#3-docker-commands-cheat-sheet)
- [4. Dockerfile Keywords](#4-dockerfile-keywords)
- [5. ShopEase Dockerfiles](#5-shopease-dockerfiles)
- [6. Docker Compose — Multi-Container](#6-docker-compose--multi-container)

---

## 1. What is Docker?

> **Containerization** = Packaging application code + all dependencies as a single unit and running it as a container.

| Aspect | Details |
|---|---|
| **Problem** | "Works on my machine but not in production" |
| **Solution** | Package app + Java + configs into one Docker image |
| **Container** | Lightweight virtual machine running your app |

---

## 2. Docker Architecture

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Dockerfile  │───▶│ Docker Image │───▶│  Docker Hub  │───▶│  Container   │
│ (recipe)     │    │ (snapshot)   │    │  (registry)  │    │ (running app)│
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
   docker build        docker push         docker pull        docker run
```

---

## 3. Docker Commands Cheat Sheet

| Command | Purpose |
|---|---|
| `docker images` | List all images |
| `docker pull <image>` | Download image |
| `docker build -t <name> .` | Build image from Dockerfile |
| `docker run -d -p 8081:8081 <image>` | Run container (detached, port mapped) |
| `docker ps` | Running containers |
| `docker ps -a` | All containers (running + stopped) |
| `docker logs <container-id>` | View container logs |
| `docker stop <container-id>` | Stop container |
| `docker start <container-id>` | Restart stopped container |
| `docker rm <container-id>` | Remove stopped container |
| `docker rmi <image-id>` | Remove image |
| `docker system prune -a` | Clean up unused images + containers |
| `docker login` | Login to Docker Hub |
| `docker push <image>` | Push image to registry |

---

## 4. Dockerfile Keywords

| Keyword | Purpose | Example |
|---|---|---|
| `FROM` | Base image | `FROM openjdk:17-slim` |
| `MAINTAINER` | Author | `MAINTAINER surya@shopease.com` |
| `COPY` | Copy files host → container | `COPY target/app.jar /app/` |
| `RUN` | Execute during **image build** | `RUN apt-get update` |
| `CMD` | Execute during **container start** | `CMD ["java", "-jar", "app.jar"]` |
| `ENTRYPOINT` | Main command (not overridable) | `ENTRYPOINT ["java", "-jar"]` |
| `EXPOSE` | Document container port | `EXPOSE 8081` |
| `WORKDIR` | Set working directory | `WORKDIR /app` |

> **RUN vs CMD:** RUN executes during image build (can have multiple). CMD executes when container starts (only last CMD is used).

---

## 5. ShopEase Dockerfiles

### Product Service Dockerfile

```dockerfile
FROM openjdk:17-slim

MAINTAINER surya@shopease.com

COPY target/product-service.jar /app/product-service.jar

WORKDIR /app

EXPOSE 8081

ENTRYPOINT ["java", "-jar", "product-service.jar"]
```

### Build & Run

```bash
cd ShopEase/product-service
mvn clean package
docker build -t shopease/product-service:1.0 .
docker run -d -p 8081:8081 --name product-svc shopease/product-service:1.0

# Push to Docker Hub
docker login
docker push shopease/product-service:1.0
```

---

## 6. Docker Compose — Multi-Container

> Manage all ShopEase services with a single `docker-compose.yml`.

```yaml
version: "3.8"

services:

  # --- Infrastructure ---
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: shopease
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql

  redis:
    image: redis:7
    ports:
      - "6379:6379"

  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
    ports:
      - "9092:9092"

  # --- Microservices ---
  service-registry:
    image: shopease/service-registry:1.0
    ports:
      - "8761:8761"

  config-server:
    image: shopease/config-server:1.0
    ports:
      - "8888:8888"
    depends_on:
      - service-registry

  product-service:
    image: shopease/product-service:1.0
    ports:
      - "8081:8081"
    depends_on:
      - mysql
      - redis
      - service-registry
      - config-server

  order-service:
    image: shopease/order-service:1.0
    ports:
      - "8082:8082"
    depends_on:
      - mysql
      - kafka
      - service-registry

  user-service:
    image: shopease/user-service:1.0
    ports:
      - "8083:8083"
    depends_on:
      - mysql
      - service-registry

  notification-service:
    image: shopease/notification-service:1.0
    ports:
      - "8084:8084"
    depends_on:
      - kafka
      - service-registry

  api-gateway:
    image: shopease/api-gateway:1.0
    ports:
      - "8080:8080"
    depends_on:
      - service-registry
      - product-service
      - order-service
      - user-service

volumes:
  mysql-data:
```

### Docker Compose Commands

```bash
docker-compose up -d       # Start all services
docker-compose ps          # Check status
docker-compose logs -f     # Follow all logs
docker-compose down        # Stop and remove all
```

---

*← [12 — Multi Threading](./12-multithreading.md) | [14 — Kubernetes →](./14-kubernetes.md)*
