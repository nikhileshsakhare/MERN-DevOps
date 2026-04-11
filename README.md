# MERN DevOps - Full-Stack E-Commerce on Kubernetes

> A production-style DevOps pipeline for a MERN e-commerce application - containerised with Docker, orchestrated with Kubernetes (Minikube), and delivered through a Jenkins CI/CD pipeline on AWS EC2.

**Application forked from:** [HuXn-WebDev/MERN-E-Commerce-Store](https://github.com/HuXn-WebDev/MERN-E-Commerce-Store)

---

## What This Project Does

The application is a full-stack MERN e-commerce store. This repository adds a complete DevOps layer on top of it:

- **Dockerised** frontend (React + nginx) and backend (Node.js + Express)
- **Kubernetes manifests** for all three tiers - frontend, backend, and MongoDB
- **Horizontal Pod Autoscalers** on both the frontend and backend
- **Jenkins CI/CD pipeline** that builds, pushes, and deploys on every git push
- **AWS EC2** as the single host running Jenkins (in Docker) and Minikube side-by-side
- **socat** to forward the EC2 public IP to the Minikube nginx Ingress, making the app accessible without a load balancer

---

## Architecture

![Architecture Diagram](mern_stack_architecture.png)

Everything runs on a **single AWS EC2 instance** (t3.medium, Amazon Linux 2023).

---

## Repository Structure

```
MERN-DevOps/
├── Jenkinsfile                  # Declarative CI/CD pipeline
├── docker-compose.yml           # Local development stack
│
├── backend/
│   ├── Dockerfile               # Node.js 20 Alpine, non-root user
│   ├── .env.example             # Environment variable template
│   └── .dockerignore
│
├── frontend/
│   ├── Dockerfile               # Multi-stage: Node build → nginx serve
│   ├── nginx.conf               # SPA routing + /api proxy to backend
│   └── .dockerignore
│
├── K8s/
│   ├── namespace.yaml           # mern-app namespace
│   ├── secrets.yaml             # MONGO_URI + JWT_SECRET
│   │
│   ├── mongo/
│   │   ├── statefulset.yaml     # mongo:7, mounts PVC
│   │   ├── service.yaml         # Headless ClusterIP on 27017
│   │   └── pvc.yaml             # 2Gi persistent storage
│   │
│   ├── server/
│   │   ├── deployment.yaml      # 2 replicas, readiness + liveness probes
│   │   ├── service.yaml         # ClusterIP on 5000
│   │   └── hpa.yaml             # 2–10 pods, CPU 70% / Memory 80%
│   │
│   └── client/
│       ├── deployment.yaml      # 2 replicas
│       ├── service.yaml         # ClusterIP on 80
│       ├── hpa.yaml             # 2–6 pods, CPU 70%
│       └── ingress.yaml         # nginx Ingress routing rules
│
└── Terminal Steps.txt           # All EC2 terminal commands in order
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React (Vite), served by nginx |
| Backend | Node.js, Express |
| Database | MongoDB 7 |
| Containerisation | Docker, Docker Hub |
| Orchestration | Kubernetes (Minikube) |
| Ingress | nginx Ingress controller |
| CI/CD | Jenkins (LTS, Dockerised) |
| Cloud | AWS EC2 (t3.medium, Amazon Linux 2023) |
| Port forwarding | socat |

---

## Prerequisites

Before starting, have the following ready:

- An AWS account with permission to launch EC2 instances
- A Docker Hub account - create a PAT at **Account Settings → Personal access tokens** (Read, Write, Delete)
- A GitHub account - create a PAT at **Settings → Developer settings → Personal access tokens → Tokens (classic)** with `repo` and `admin:repo_hook` scopes

---

## Deployment Guide

A full step-by-step deployment guide (14 steps, AWS Console through to browser access) is included in this repository as `mern_stack_deployment_guide.pdf`.

### Quick summary of steps

**1. AWS Console** - Launch an EC2 instance (t3.medium, Amazon Linux 2023, 20 GB gp3). Open inbound ports 22, 80, and 8080 in the security group.

**2. SSH into EC2**
```bash
chmod 400 your-key.pem
ssh -i your-key.pem ec2-user@<EC2_PUBLIC_IP>
```

**3. Install Docker**
```bash
sudo yum install docker -y
sudo usermod -aG docker $USER && newgrp docker
sudo systemctl start docker && sudo systemctl enable docker
```

**4. Install kubectl and Minikube**
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
```

**5. Start Minikube and enable Ingress**
```bash
minikube start --driver=docker
kubectl config view --minify --flatten > ~/.kube/config-flat
minikube addons enable ingress
sudo yum install -y socat
```

**6. Run Jenkins**
```bash
DOCKER_GID=$(getent group docker | cut -d: -f3)
docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /home/ec2-user/.kube:/home/ec2-user/.kube:ro \
  -v /usr/local/bin/kubectl:/usr/local/bin/kubectl \
  --group-add $DOCKER_GID --restart=always \
  jenkins/jenkins:lts

docker exec -it --user root jenkins bash -c "apt-get update && apt-get install -y docker.io"
sudo chmod 666 /var/run/docker.sock
sudo chmod 644 /home/ec2-user/.kube/config
docker network connect minikube jenkins
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

**7. Configure Jenkins** - Open `http://<EC2_PUBLIC_IP>:8080`, install suggested plugins plus NodeJS Plugin and Docker Pipeline. Register the NodeJS tool as `NodeJS 24.14.1`, add your DockerHub PAT as `dockerhub-credentials`, and add your MongoDB URI as `mongo-uri`.

**8. Create pipeline job** - New Item → Pipeline → Pipeline script from SCM → Git → your repo URL → Script Path: `Jenkinsfile`. Trigger with Build Now.

**9. Forward port 80 to Minikube**
```bash
sudo nohup socat TCP-LISTEN:80,fork TCP:$(minikube ip):80 > /tmp/socat.log 2>&1 &
```

**10. Access the app**
```
http://<EC2_PUBLIC_IP>
```

---

## CI/CD Pipeline Stages

The `Jenkinsfile` runs these stages in order on every triggered build:

| Stage | Description |
|---|---|
| Checkout | Clones the repository |
| Install Dependencies | `npm ci` for frontend and backend in parallel |
| Lint Frontend | ESLint with max-warnings 100 |
| Build React App | `npm run build` — produces the production `dist/` |
| Build Docker Images | Builds `mern-app-frontend:$BUILD_NUMBER` and `mern-app-backend:$BUILD_NUMBER` |
| Push to Docker Hub | Logs in with stored credentials and pushes both images |
| Setup Kubeconfig | Copies the flattened kubeconfig into the Jenkins container |
| Deploy to Kubernetes | `kubectl apply` all manifests, waits for rollout to complete |

---

## Environment Variables

Copy `backend/.env.example` to `backend/.env` for local development:

```
PORT=5000
MONGO_URI=mongodb://mongo:27017/merndb
JWT_SECRET=your-secret-here
NODE_ENV=production
```

In Kubernetes, these are injected from `K8s/secrets.yaml` — update the values there before applying.

---

## Local Development (Docker Compose)

To run the full stack locally without Kubernetes:

```bash
docker compose up --build
```

The frontend will be available at `http://localhost` and the backend at `http://localhost:5000`.

---

## Autoscaling

| Component | Min pods | Max pods | Scale trigger |
|---|---|---|---|
| Frontend (client) | 2 | 6 | CPU > 70% |
| Backend (server) | 2 | 10 | CPU > 70% or Memory > 80% |

---

## Credits

- Original MERN e-commerce application by [HuXn-WebDev](https://github.com/HuXn-WebDev/MERN-E-Commerce-Store)
- DevOps layer (Docker, Kubernetes, Jenkins CI/CD, AWS deployment) by [nikhileshsakhare](https://github.com/nikhileshsakhare)
