# DevOps CI/CD Internship Project — Node.js → Docker → Kubernetes (Local + EKS/AKS)

A hands-on project to build a complete CI/CD workflow that **builds**, **containerizes**, and **deploys** a simple Node.js web app with **Jenkins** to **Kubernetes**. Primary target is **local Kubernetes (Docker Desktop)** with an optional bonus track for **Amazon EKS** or **Azure AKS**.

> TL;DR: You’ll run a Node.js app locally, dockerize it, deploy it to a local K8s cluster, and automate everything via a Jenkins pipeline.


## Table of Contents

- [Architecture & Flow](#architecture--flow)
- [Tech Stack](#tech-stack)
- [Prerequisites (Windows 11)](#prerequisites-windows-11)
- [Quick Start](#quick-start)
- [Project Structure](#project-structure)
- [Application (Node.js Express)](#application-nodejs-express)
- [Containerization (Docker)](#containerization-docker)
- [Kubernetes (Local)](#kubernetes-local)
- [Jenkins Pipeline](#jenkins-pipeline)
- [Cloud Bonus: EKS / AKS](#cloud-bonus-eks--aks)
- [Validation & Scaling](#validation--scaling)
- [Deliverables Checklist](#deliverables-checklist)
- [Screenshots (placeholders)](#screenshots-placeholders)


## Architecture & Flow

```
Developer → GitHub → Jenkins → Docker Image → Kubernetes (Local)
                                          ↘ (optional) EKS / AKS
Service (NodePort) → http://localhost:30080 → Browser / curl
```

**Pipeline stages:** Git Checkout → Docker Build → K8s Deploy → (Scale Test)


## Tech Stack

- **App:** Node.js (Express)
- **CI/CD:** Jenkins (Pipeline)
- **Container:** Docker
- **Orchestration:** Kubernetes (Docker Desktop)
- **OS:** Windows 11
- **Cloud (Bonus):** Amazon EKS (AWS CLI + eksctl) or Azure AKS (Azure CLI)


## Prerequisites (Windows 11)

Install the following:

- Git — <https://git-scm.com/downloads>  
- Node.js LTS — <https://nodejs.org/en/download>  
- VS Code — <https://code.visualstudio.com/Download>  
- Docker Desktop — <https://www.docker.com/products/docker-desktop> (Enable **Kubernetes** in *Settings → Kubernetes → Enable*)  
- `kubectl` — via **Chocolatey** `choco install kubernetes-cli` or download binary  
- Jenkins — <https://www.jenkins.io/download/> (Windows service installer)  
- **Optional (Cloud):** AWS CLI + `eksctl` or Azure CLI

**Verify installs**
```bash
node -v
npm -v
git --version
docker version
kubectl version --client
```

**Enable Kubernetes in Docker Desktop**
- Docker Desktop → Settings → Kubernetes → **Enable Kubernetes** → Apply & restart.
- Confirm:
```bash
kubectl get nodes
kubectl get pods -A
```

**Jenkins (Local)**
1) Install Jenkins and open <http://localhost:8080>  
2) Unlock with the initial admin password from  
   `%ProgramData%\Jenkins\.jenkins\secrets\initialAdminPassword`  
3) Create admin user; install suggested plugins.  
4) Add plugins: **Docker Pipeline**, **Kubernetes CLI**, **Git**.


## Quick Start

```bash
# 1) Clone your project (replace with your repo)
git clone https://github.com/<your-username>/sample-node-app.git
cd sample-node-app

# 2) Run locally
npm install
npm start
# Open http://localhost:3000
```

Build & run container:
```bash
docker build -t sample-node-app:v1.0 .
docker run -p 3000:3000 sample-node-app:v1.0
```

Deploy to local Kubernetes:
```bash
kubectl apply -f k8s-deployment.yaml
kubectl apply -f k8s-service.yaml
kubectl get pods
kubectl get svc
curl http://localhost:30080
```


## Project Structure

```
sample-node-app/
├─ app.js
├─ package.json
├─ Dockerfile
├─ Jenkinsfile
├─ k8s-deployment.yaml
└─ k8s-service.yaml
```


## Application (Node.js Express)

`package.json`
```json
{
  "name": "sample-node-app",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

`app.js`
```js
const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

app.get('/', (req, res) => {
  res.send('Hello from Node.js app!');
});

app.listen(port, () => {
  console.log(`App listening at http://localhost:${port}`);
});
```

**Local Test**
```bash
npm install
npm start
# http://localhost:3000
```


## Containerization (Docker)

`Dockerfile`
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```

**Build & Run**
```bash
docker build -t sample-node-app:v1.0 .
docker run -p 3000:3000 sample-node-app:v1.0
```


## Kubernetes (Local)

`k8s-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-app-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: node-app
  template:
    metadata:
      labels:
        app: node-app
    spec:
      containers:
      - name: node-app
        image: sample-node-app:v1.0
        ports:
        - containerPort: 3000
```

`k8s-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: node-app-service
spec:
  type: NodePort
  selector:
    app: node-app
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
      nodePort: 30080
```

**Apply & Verify**
```bash
kubectl apply -f k8s-deployment.yaml
kubectl apply -f k8s-service.yaml
kubectl get pods
kubectl get svc
curl http://localhost:30080
```


## Jenkins Pipeline

**Required Plugins:** Docker Pipeline, Kubernetes CLI, Git

> If you plan to push images to a registry, add the appropriate credentials in **Jenkins → Manage Credentials**.

`Jenkinsfile`
```groovy
pipeline {
  agent any

  environment {
    IMAGE_NAME = 'sample-node-app'
    IMAGE_TAG  = 'v1.0'
    // Ensure this file exists and points to your local K8s config (Docker Desktop)
    KUBECONFIG = 'C:\\\\ProgramData\\\\Jenkins\\\\.kube\\\\config'
  }

  stages {
    stage('Checkout Code') {
      steps {
        git branch: 'main', url: 'https://github.com/<your-username>/sample-node-app.git'
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        bat '''
          REM --- Make sure kubectl bypasses any proxies in Jenkins env ---
          set "HTTP_PROXY="
          set "HTTPS_PROXY="
          set "NO_PROXY=localhost,127.0.0.1,::1,kubernetes.docker.internal,*.local,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16"

          kubectl apply -f k8s-deployment.yaml
          kubectl apply -f k8s-service.yaml
          kubectl scale deployment node-app-deployment --replicas=3
        '''
      }
    }
  }

  post {
    success {
      echo 'Pipeline completed successfully!'
    }
  }
}
```

**Set up the Pipeline job**
1. Create a **Pipeline** job in Jenkins.
2. Point the pipeline to your Jenkinsfile in SCM (Git).
3. Build the job and watch the stages go green.


## Cloud Bonus: EKS / AKS

The same Jenkins pipeline can target a managed cluster:

- **EKS:** Install **AWS CLI** + **`eksctl`**, create an EKS cluster, and update `KUBECONFIG` to point to the EKS context.  
- **AKS:** Install **Azure CLI**, create an AKS cluster, and set `KUBECONFIG` accordingly.

> Keep the manifests portable; only the cluster context should change.


## Validation & Scaling

Scale the deployment to 3 replicas:
```bash
kubectl scale deployment node-app-deployment --replicas=3
kubectl get pods -l app=node-app
```

Validate service:
```bash
curl http://localhost:30080
```


## Deliverables Checklist

- [ ] Source code (Node.js)
- [ ] Dockerfile
- [ ] Jenkinsfile
- [ ] Kubernetes YAMLs (local + cloud service, if used)
- [ ] Screenshots (pipeline, pods, services, browser)
- [ ] README with setup/run instructions


## Screenshots (placeholders)

- ✅ *Versions output* (`node -v`, `npm -v`, `git --version`, `docker version`, `kubectl version --client`)
- ✅ *Docker Desktop → Kubernetes enabled*
- ✅ `kubectl get nodes` / `kubectl get pods -A`
- ✅ Jenkins UI (unlocked, plugins installed)
- ✅ Successful pipeline run (all stages green)
- ✅ Browser showing app (`http://localhost:3000` and `http://localhost:30080`)
- ✅ `kubectl get pods` and `kubectl get svc` after deploy
```

# Tips
- If Jenkins can’t find `KUBECONFIG`, verify the path exists: `C:\ProgramData\Jenkins\.kube\config`.
- If `kubectl` hits proxy issues in Jenkins, use the `NO_PROXY` settings shown above.
- Ensure Docker Desktop’s Kubernetes is enabled before deploying locally.
