# **Deploying a Next.js and Node.js 3-Tier Application Using Docker, Kubernetes, Terraform, and AWS**

---

## **Overview**

This document provides a step-by-step guide to deploy a **Next.js and Node.js 3-tier web application** using the following technologies:

- **Docker**: Containerization for the frontend and backend.
- **Kubernetes**: Orchestration of containers using AWS Elastic Kubernetes Service (EKS).
- **Terraform**: Infrastructure as Code (IaC) for AWS resources.
- **AWS Services**:
   - **EKS**: Kubernetes managed service for running containers.
   - **RDS**: Managed relational database.
   - **ALB**: Application Load Balancer for frontend load balancing.

---

## **Prerequisites**

Ensure the following tools are installed before proceeding:

1. **AWS CLI**: [Install AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
2. **Terraform**: [Install Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli)
3. **Docker**: [Install Docker](https://docs.docker.com/get-docker/)
4. **kubectl**: [Install kubectl](https://kubernetes.io/docs/tasks/tools/)
5. **eksctl**: [Install eksctl](https://eksctl.io/introduction/install/)
6. **Git**: [Install Git](https://git-scm.com/downloads)

---

## **Folder Structure**

```
project-root/
│
├── infra/                     # Terraform infrastructure code
├── frontend/                  # Next.js frontend code
├── backend/                   # Node.js backend code
├── k8s/                       # Kubernetes manifests
│   ├── frontend-deployment.yaml
│   ├── backend-deployment.yaml
│   └── service.yaml
└── README.md
```

---

## **Step 1: Clone the Repository**

```bash
git clone <your-repository-url>
cd project-root
```

---

## **Step 2: Dockerize the Application**

### **Frontend (Next.js)**

Create a `Dockerfile` in the `frontend` folder:

```dockerfile
# Dockerfile for Next.js
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]
```

### **Backend (Node.js)**

Create a `Dockerfile` in the `backend` folder:

```dockerfile
# Dockerfile for Node.js Backend
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN npm install
EXPOSE 5000
CMD ["node", "index.js"]
```

### **Build and Push Docker Images**

Replace `<dockerhub-user>` with your Docker Hub username:

```bash
docker build -t <dockerhub-user>/frontend ./frontend
docker build -t <dockerhub-user>/backend ./backend
docker push <dockerhub-user>/frontend
docker push <dockerhub-user>/backend
```

---

## **Step 3: Infrastructure Setup with Terraform**

### **Create Terraform Files**

#### `infra/main.tf`:

```hcl
provider "aws" {
  region = "us-west-2"  # Replace with your desired region
}

# VPC Module
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "3.5"
  name    = "3-tier-vpc"
  cidr    = "10.0.0.0/16"
}

# EKS Module
module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = "3-tier-cluster"
  subnet_ids      = module.vpc.public_subnets
  vpc_id          = module.vpc.vpc_id
}

# RDS Instance
resource "aws_db_instance" "db" {
  allocated_storage = 20
  engine            = "mysql"
  engine_version    = "8.0"
  instance_class    = "db.t3.micro"
  name              = "mydb"
  username          = "admin"
  password          = "password123"
  publicly_accessible = false
}
```

### **Initialize and Apply Terraform**

Run the following commands to provision infrastructure:

```bash
cd infra/
terraform init
terraform apply
```

---

## **Step 4: Deploy to Kubernetes (EKS)**

### **Connect to EKS Cluster**

```bash
aws eks --region us-west-2 update-kubeconfig --name 3-tier-cluster
```

### **Create Kubernetes Manifests**

#### **Frontend Deployment (`k8s/frontend-deployment.yaml`)**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: <dockerhub-user>/frontend
        ports:
        - containerPort: 3000
```

#### **Backend Deployment (`k8s/backend-deployment.yaml`)**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: <dockerhub-user>/backend
        ports:
        - containerPort: 5000
```

#### **Service (`k8s/service.yaml`)**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 5000
      targetPort: 5000
```

### **Apply Kubernetes Manifests**

```bash
kubectl apply -f k8s/frontend-deployment.yaml
kubectl apply -f k8s/backend-deployment.yaml
kubectl apply -f k8s/service.yaml
```

### **Verify Services**

```bash
kubectl get services
```

---

## **Step 5: Monitoring and Scaling**

### **Monitor Deployments**

Use the following command to check pods and deployments:

```bash
kubectl get pods
kubectl get deployments
```

### **Scale Deployments**

To scale the frontend or backend, run:

```bash
kubectl scale deployment frontend --replicas=4
kubectl scale deployment backend --replicas=4
```

---

## **Step 6: Clean Up Resources**

To destroy all infrastructure and Kubernetes resources, run:

```bash
terraform destroy
kubectl delete -f k8s/
```

---

## **Conclusion**

You now have a fully deployed 3-tier architecture application using Docker, Kubernetes, Terraform, and AWS.

Happy Deploying! 🎉


