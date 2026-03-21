# ☸️ Module 14: Kubernetes (K8S)

> All examples use the **ShopEase** e-commerce microservice project.

---

## 1. What is Kubernetes?

| Aspect | Details |
|---|---|
| **Type** | Open-source container orchestration platform |
| **Created by** | Google (donated to CNCF) |
| **Purpose** | Manage, scale, and heal containers automatically |

### K8S Advantages

| Feature | Description |
|---|---|
| **Orchestration** | Manage hundreds of containers |
| **Auto Scaling** | Scale up/down based on load |
| **Self Healing** | Restart crashed pods automatically |
| **Load Balancing** | Distribute traffic across pod replicas |

---

## 2. K8S Architecture

```
┌─────────────────── Control Plane (Master) ───────────────────┐
│  API Server │ Scheduler │ Controller Manager │ ETCD (DB)      │
└──────────────────────────┬───────────────────────────────────┘
                           │
    ┌──────────────────────┼──────────────────────┐
    │                      │                      │
┌───▼────────────┐  ┌──────▼───────────┐  ┌──────▼───────────┐
│  Worker Node 1 │  │  Worker Node 2   │  │  Worker Node 3   │
│  ┌───────────┐ │  │  ┌───────────┐   │  │  ┌───────────┐   │
│  │   POD     │ │  │  │   POD     │   │  │  │   POD     │   │
│  │ product   │ │  │  │ order     │   │  │  │ product   │   │
│  │ service   │ │  │  │ service   │   │  │  │ service   │   │
│  └───────────┘ │  │  └───────────┘   │  │  │ (replica) │   │
│  Kubelet       │  │  Kubelet         │  │  └───────────┘   │
│  Kube-Proxy    │  │  Kube-Proxy      │  │  Kubelet         │
│  Docker Engine │  │  Docker Engine   │  │  Kube-Proxy      │
└────────────────┘  └──────────────────┘  └──────────────────┘
```

| Component | Purpose |
|---|---|
| **API Server** | Receives all requests (kubectl → API Server) |
| **ETCD** | Cluster database — stores all state |
| **Scheduler** | Assigns pods to available worker nodes |
| **Controller Manager** | Monitors and maintains desired state |
| **Kubelet** | Worker node agent |
| **Kube-Proxy** | Network communication within cluster |
| **POD** | Smallest deployable unit (runs your container) |

---

## 3. K8S Services

| Service Type | Purpose | Use Case |
|---|---|---|
| **ClusterIP** | Internal access only | Service-to-service communication |
| **NodePort** | Access via Node's Public IP | Development / testing |
| **LoadBalancer** | External traffic + load balancing | Production deployments |

---

## 4. ShopEase K8S Manifest

### product-service-deployment.yml

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  labels:
    app: product-service
spec:
  replicas: 2                         # ← Run 2 instances
  strategy:
    type: RollingUpdate               # ← Zero-downtime deployment
  selector:
    matchLabels:
      app: product-service
  template:
    metadata:
      labels:
        app: product-service
    spec:
      containers:
        - name: product-service
          image: shopease/product-service:1.0
          ports:
            - containerPort: 8081
          env:
            - name: SPRING_DATASOURCE_URL
              value: jdbc:mysql://mysql-service:3306/shopease
---
apiVersion: v1
kind: Service
metadata:
  name: product-service-svc
spec:
  type: LoadBalancer
  selector:
    app: product-service
  ports:
    - port: 80
      targetPort: 8081
```

---

## 5. Essential kubectl Commands

```bash
# Apply manifest
kubectl apply -f product-service-deployment.yml

# Check pods
kubectl get pods
kubectl get pods -o wide          # with node info

# Check services
kubectl get svc

# Check deployments
kubectl get deployment

# View pod logs
kubectl logs <pod-name>

# Delete a pod (K8S will auto-restart — self healing!)
kubectl delete pod <pod-name>

# Delete all resources
kubectl delete all --all
```

---

*← [13 — Docker](./13-docker.md) | [15 — Jenkins CI/CD →](./15-jenkins-cicd.md)*
