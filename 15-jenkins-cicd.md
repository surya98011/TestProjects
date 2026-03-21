# 🔄 Module 15: Jenkins CI/CD

> All examples use the **ShopEase** e-commerce microservice project.

---

## 1. What is CI/CD?

| Term | Meaning | What Happens |
|---|---|---|
| **CI** | Continuous Integration | Auto build + test on every commit |
| **CD** | Continuous Deployment | Auto deploy to servers |

### Manual Build Problems

| Problem | Impact |
|---|---|
| Daily manual deployments | Time-consuming |
| Multiple environments | Repeated work |
| Human errors | Deployments break |

> Jenkins automates the entire process: **Git pull → Maven build → Docker image → Deploy container**

---

## 2. Jenkins Pipeline for ShopEase

### Declarative Pipeline — product-service

```groovy
pipeline {

    agent any

    tools {
        maven "M3"    // Maven installation configured in Jenkins
    }

    stages {
        stage('Git Clone') {
            steps {
                git branch: 'develop',
                    url: 'https://github.com/shopease/product-service.git'
            }
        }

        stage('Maven Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Run Tests') {
            steps {
                sh 'mvn test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                sh 'mvn sonar:sonar'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t shopease/product-service:${BUILD_NUMBER} .'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub-creds',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS')]) {
                    sh 'docker login -u $USER -p $PASS'
                    sh 'docker push shopease/product-service:${BUILD_NUMBER}'
                }
            }
        }

        stage('Deploy') {
            steps {
                sh 'docker stop product-svc || true'
                sh 'docker rm product-svc || true'
                sh 'docker run -d -p 8081:8081 --name product-svc shopease/product-service:${BUILD_NUMBER}'
            }
        }
    }
}
```

---

## 3. CI/CD Flow

```
Developer pushes code to Git
        │
        ▼
Jenkins detects change (webhook / polling)
        │
        ▼
┌──────────────────────────────────┐
│     Jenkins Pipeline Stages      │
│                                  │
│  1. Git Clone                    │
│  2. Maven Build (compile+test)   │
│  3. SonarQube Analysis           │
│  4. Build Docker Image           │
│  5. Push Image to Docker Hub     │
│  6. Deploy Container             │
└──────────────────────────────────┘
        │
        ▼
Application running and accessible!
```

---

## 4. Real-Time Workflow

| Who | Does What |
|---|---|
| **DevOps Team** | Sets up Jenkins, creates pipelines, manages users |
| **Development Team** | Runs pipelines, checks console output on failures |
| **Release Team** | Handles production deployments (separate Jenkins server) |

> **📝 Note:** If a CI/CD job fails, check the **Console Output** in Jenkins to see the error logs and fix the issue.

---

*← [14 — Kubernetes](./14-kubernetes.md) | [16 — Nexus Repository →](./16-nexus-repository.md)*
