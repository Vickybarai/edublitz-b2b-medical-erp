
[meansrepo](https://github.com/shubhamkalsait/edublitz-b2b-medical-erp/tree/project-b22)
[meank8s_guide](https://github.com/shubhamkalsait/edublitz-b2b-medical-erp/blob/project-b22/docs/KUBERNETES_DEPLOYMENT.md)
# MedPharm ERP - Complete Deployment Guide

> **B2B Pharmaceutical ERP System** | Microservices | AWS EKS | Docker | MongoDB Atlas
>
> A step-by-step beginner-friendly guide to deploy the entire MedPharm ERP stack from scratch.

---

## Table of Contents

- [Project Overview](#-project-overview)
- [Architecture Diagram](#-architecture-diagram)
- [Tech Stack Reference](#-tech-stack-reference)
- [Phase 1 - MongoDB Atlas Setup](#phase-1--mongodb-atlas-database-setup)
- [Phase 2 - AWS EC2 Command Center](#phase-2--aws-ec2-command-center)
- [Phase 3 - EKS Cluster Setup (Detailed)](#phase-3--eks-cluster-setup-detailed)
- [Phase 4 - Install Prerequisites on EC2](#phase-4--install-prerequisites-on-ec2)
- [Phase 5 - Build & Push Docker Images](#phase-5--build--push-docker-images-to-ecr)
- [Phase 6 - SSL Certificates (ACM)](#phase-6--ssl-certificates--acm-)
- [Phase 7 - S3 Bucket & CloudFront (Frontend)](#phase-7--s3-bucket--cloudfront-frontend)
- [Phase 8 - IAM Setup for Load Balancer Controller](#phase-8--iam-setup-for-aws-load-balancer-controller-irsa)
- [Phase 9 - Kubernetes Deployment](#phase-9--kubernetes-deployment)
- [Phase 10 - DNS Configuration (Route 53)](#phase-10--dns-configuration--route-53-)
- [Phase 11 - Final Verification & Testing](#phase-11--final-verification--testing)
- [Troubleshooting Guide](#troubleshooting-guide)
- [Useful kubectl Commands](#useful-kubectl-commands)
- [Appendix - Complete YAML Reference](#appendix--complete-yaml-reference)

---

## Project Overview

**MedPharm ERP** is a B2B (Business-to-Business) Enterprise Resource Planning system designed for the pharmaceutical supply chain. It connects manufacturers, distributors, and hospital procurement departments on a single digital platform.

### What the System Does

| Capability | Description |
|---|---|
| **User Management** | Authentication (JWT), role-based access (Admin, Distributor, Hospital), organization management |
| **Product Catalog** | Product listings with SKU, categories, pricing, batch tracking |
| **Inventory Management** | Stock levels, warehouse tracking, expiry monitoring, low-stock alerts |
| **Order Processing** | Purchase orders, order tracking, status management (Pending, Confirmed, Shipped, Delivered) |

### Microservices Architecture

```
                    ┌─────────────────────────────────────────────┐
                    │              EKS Cluster                     │
                    │                                              │
   Browser ──────►  │   ALB Ingress ──── /api/v1/auth    ──► user-service:8081     │
                    │                ──── /api/v1/users   ──► user-service:8081     │
                    │                ──── /api/v1/products ──► product-service:8082 │
                    │                ──── /api/v1/orders   ──► order-service:8083  │
                    │                                              │
                    │   order-service ──► user-service (ClusterIP) │
                    │   order-service ──► product-service (ClusterIP)│
                    └─────────────────────────────────────────────┘
                                       │
                                       ▼
                              MongoDB Atlas (Cloud)
                         ┌────────────────────────┐
                         │  users_db │ products_db  │
                         │  orders_db               │
                         └────────────────────────┘
```

**Frontend** (React) is hosted separately on **S3 + CloudFront** and communicates with the backend APIs via the ALB domain.

---

## Architecture Diagram

### Traffic Flow

```
                                    AWS Cloud
  ┌──────────┐                       │
  │  Browser  │──────────────────────┤
  └────┬─────┘                       │
       │                             │
       ├──────► CloudFront (CDN) ────┤──────► S3 Bucket (Frontend React)
       │         (Frontend)          │
       │                             │
       ├──────► Route 53 ────────────┤
       │     (api.yourdomain.com)    │
       │                             │
       └──────► ALB Ingress ─────────┤──────► EKS Cluster
                  (HTTPS:443)        │         ├── user-service:8081
                                     │         ├── product-service:8082
                                     │         └── order-service:8083
                                     │
                                     ▼
                              MongoDB Atlas
                              (3 Databases)
```

### Inter-Service Communication

| Service | Port | Talks To | Purpose |
|---|---|---|---|
| **user-service** | 8081 | MongoDB (`users_db`) | Auth, users, organizations |
| **product-service** | 8082 | MongoDB (`products_db`) | Products, inventory |
| **order-service** | 8083 | MongoDB (`orders_db`) | Orders |
| **order-service** | 8083 | `user-service:8081` (internal) | Validate users/orgs |
| **order-service** | 8083 | `product-service:8082` (internal) | Validate products, check stock |

> **Key Point:** Inter-service calls happen internally via Kubernetes ClusterIP services (not through the ALB). The ConfigMap stores these internal URLs.

---

## Tech Stack Reference

| Component | Technology | Version | Purpose |
|---|---|---|---|
| **Backend Services** | Spring Boot (Java) | JDK 17 | 3 microservices |
| **Frontend** | React + TypeScript + Vite | Node 20 | SPA web application |
| **CSS** | Tailwind CSS | v3 | Utility-first styling |
| **Database** | MongoDB Atlas | 7.0 | Cloud NoSQL (3 databases) |
| **Container** | Docker | Multi-stage builds | Image packaging |
| **Registry** | AWS ECR | - | Container image storage |
| **Orchestration** | AWS EKS | Kubernetes 1.30 | Container orchestration |
| **Ingress** | AWS Load Balancer Controller | v2+ | ALB-based traffic routing |
| **Frontend Hosting** | S3 + CloudFront | - | Static hosting + CDN |
| **SSL** | AWS ACM | - | TLS certificates |
| **DNS** | Route 53 | - | Domain name management |
| **CI/CD** | Jenkins + ArgoCD (optional) | - | Automated pipelines |
| **IaC** | Terraform / eksctl | - | Infrastructure as Code |

### Port Mapping

| Service | Internal Port | Context Path | Swagger UI |
|---|---|---|---|
| user-service | 8081 | `/api/v1` | `/api/v1/swagger-ui.html` |
| product-service | 8082 | `/api/v1` | `/api/v1/swagger-ui.html` |
| order-service | 8083 | `/api/v1` | `/api/v1/swagger-ui.html` |
| frontend | 80 (nginx) | `/` | - |

---

## Phase 1 - MongoDB Atlas Database Setup

> **Goal:** Create a cloud database to store user, product, and order data.

### 1.1 Create Atlas Account

1. Go to [MongoDB Atlas](https://www.mongodb.com/atlas)
2. **Sign Up** (free) or **Log In** with your credentials

### 1.2 Build a Free Cluster

1. Click **Build a Database** (big green button on the dashboard)
2. Select **M0 Sandbox** (Free tier)
3. **Cluster Name:** `edublitz-cluster`
4. **Region:** Select the region closest to your EKS cluster (e.g., **Mumbai**, **Singapore**, or **Sydney**)
   - For `ap-southeast-2` (Sydney EKS), choose **Sydney** for lowest latency
5. Click **Create Cluster** and wait 2-3 minutes

### 1.3 Create Database User

1. Left sidebar > **Database Access** > **Add New Database User**
2. Authentication Method: **Password**
3. **Username:** `admin`
4. **Password:** Your secure password (save it securely)
5. Click **Add User**

### 1.4 Configure Network Access

1. Left sidebar > **Network Access** > **Add IP Address**
2. Select **Allow Access from Anywhere** (adds `0.0.0.0/0`)
3. Click **Confirm**

> **Note:** For production, restrict this to your EKS cluster's outbound IP ranges or VPC CIDR.

### 1.5 Create Databases and Collections

MongoDB Atlas collections are created automatically on first write. However, you need to understand the schema:

| Database | Collections | Purpose |
|---|---|---|
| `users_db` | `users`, `organizations`, `audit_logs` | User accounts, org profiles, audit trail |
| `products_db` | `products`, `inventory` | Product catalog, stock levels |
| `orders_db` | `orders` | Purchase orders |

> **Important:** Use **underscore** naming (`users_db`, NOT `users-db`). The application.yml files reference `users_db` explicitly. Using hyphens will cause connection failures.

#### Index Reference (for local Docker setup)

When running locally with Docker Compose, the `init-mongo.js` script auto-creates all databases, collections, and indexes:

```javascript
// docker/init-mongo.js (runs on first MongoDB container startup)

// === users_db ===
db = db.getSiblingDB('users_db');
db.createCollection('users');
db.createCollection('organizations');
db.createCollection('audit_logs');

db.users.createIndex({ email: 1 }, { unique: true });
db.users.createIndex({ organizationId: 1 });
db.organizations.createIndex({ registrationNumber: 1 }, { unique: true });
db.organizations.createIndex({ type: 1 });
db.audit_logs.createIndex({ userId: 1 });
db.audit_logs.createIndex({ action: 1 });
db.audit_logs.createIndex({ timestamp: 1 });

// === products_db ===
db = db.getSiblingDB('products_db');
db.createCollection('products');
db.createCollection('inventory');

db.products.createIndex({ sku: 1 }, { unique: true });
db.products.createIndex({ category: 1, active: 1 });
db.products.createIndex({ distributorId: 1 });
db.inventory.createIndex({ productId: 1, warehouseId: 1, batchNumber: 1 }, { unique: true });
db.inventory.createIndex({ expiryDate: 1 });
db.inventory.createIndex({ distributorId: 1 });

// === orders_db ===
db = db.getSiblingDB('orders_db');
db.createCollection('orders');

db.orders.createIndex({ orderNumber: 1 }, { unique: true });
db.orders.createIndex({ buyerOrgId: 1, status: 1 });
db.orders.createIndex({ distributorOrgId: 1, status: 1 });
db.orders.createIndex({ createdAt: 1 });
```

### 1.6 Get Connection Strings

1. On the **Database** screen, click **Connect**
2. Choose **Connect your application**
3. Copy the connection string (it looks like):
   ```
   mongodb+srv://admin:<password>@edublitz-cluster.xxxxx.mongodb.net/?retryWrites=true&w=majority
   ```

Each service needs a **unique** connection string with the correct database name:

```
# User Service
mongodb+srv://admin:YOUR_PASSWORD@edublitz-cluster.xxxxx.mongodb.net/users_db?retryWrites=true&w=majority

# Product Service
mongodb+srv://admin:YOUR_PASSWORD@edublitz-cluster.xxxxx.mongodb.net/products_db?retryWrites=true&w=majority

# Order Service
mongodb+srv://admin:YOUR_PASSWORD@edublitz-cluster.xxxxx.mongodb.net/orders_db?retryWrites=true&w=majority
```

> **Checkpoint:** You now have 3 MongoDB Atlas connection URIs ready for Kubernetes Secrets.

---

## Phase 2 - AWS EC2 Command Center

> **Goal:** Create a Linux server to use as your build and management machine.

### 2.1 Launch EC2 Instance

1. AWS Console > **EC2** > **Instances** > **Launch Instance**
2. **Name:** `Command-Center`
3. **OS Image:** Ubuntu Server 22.04 LTS (or 24.04 LTS)
4. **Instance Type:** `t3.medium` (4 GB RAM - required for building Java Docker images)
5. **Key Pair:** Create new > `project-key` > **Download** the `.pem` file (you only get one chance)
6. **Network Settings** > **Edit**:
   - **Security Group Name:** `command-center-sg`
   - Inbound Rules:
     - SSH (Port 22) from **My IP**
     - All Traffic (All ports) from `0.0.0.0/0`
7. Click **Launch Instance**

### 2.2 Connect to EC2

Open your local terminal:

```bash
chmod 400 project-key.pem
ssh -i project-key.pem ubuntu@<YOUR_EC2_PUBLIC_IP>
```

> **Checkpoint:** You should see `ubuntu@ip-xxxxxxxx` prompt. You are inside your Command Center.

---

## Phase 3 - EKS Cluster Setup (Detailed)

> **Goal:** Create a managed Kubernetes cluster on AWS. This is the most infrastructure-heavy step.

### 3.1 What is EKS?

**Amazon EKS** (Elastic Kubernetes Service) is a **managed Kubernetes control plane**. AWS handles the Kubernetes API server, etcd, and other control plane components. You only manage the **worker nodes** (EC2 instances that run your pods).

```
┌─────────────────────────────────────────────────────────┐
│                    EKS Cluster                           │
│                                                          │
│  ┌──────────────────────────┐   ┌─────────────────────┐ │
│  │    Control Plane (AWS)    │   │   Worker Nodes (You) │ │
│  │  - API Server             │   │                      │ │
│  │  - etcd                   │   │  ┌──────┐ ┌──────┐  │ │
│  │  - Scheduler              │   │  │ Node │ │ Node │  │ │
│  │  - Controller Manager     │   │  │  #1  │ │  #2  │  │ │
│  │  (Fully managed by AWS)   │   │  │      │ │      │  │ │
│  └──────────────────────────┘   │  │ Pods │ │ Pods │  │ │
│                                  │  └──────┘ └──────┘  │ │
│                                  └─────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### 3.2 Understanding VPC and Subnets

EKS requires a **VPC** (Virtual Private Cloud) with specific subnet configuration. Here's what you need to understand:

```
┌─────────────────────────────────────────────────┐
│                     VPC (10.20.0.0/16)             │
│                                                   │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐    │
│  │ Public    │  │ Public    │  │ Public    │    │
│  │ Subnet 1  │  │ Subnet 2  │  │ Subnet 3  │    │
│  │ (AZ-1)    │  │ (AZ-2)    │  │ (AZ-3)    │    │
│  │           │  │           │  │           │    │
│  │ ALB lives │  │           │  │           │    │
│  │ here      │  │           │  │           │    │
│  │ ☁ NAT GW  │  │ ☁ NAT GW  │  │ ☁ NAT GW  │    │
│  └───────────┘  └───────────┘  └───────────┘    │
│                                                   │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐    │
│  │ Private   │  │ Private   │  │ Private   │    │
│  │ Subnet 1  │  │ Subnet 2  │  │ Subnet 3  │    │
│  │ (AZ-1)    │  │ (AZ-2)    │  │ (AZ-3)    │    │
│  │           │  │           │  │           │    │
│  │ EKS Nodes │  │ EKS Nodes │  │ EKS Nodes │    │
│  │ live here │  │ live here │  │ live here │    │
│  └───────────┘  └───────────┘  └───────────┘    │
└─────────────────────────────────────────────────┘
```

**Public Subnets** host the ALB (Load Balancer), NAT Gateways, and bastion hosts. **Private Subnets** host the EKS worker nodes for security. Each subnet must have specific **Kubernetes tags** for EKS to discover them:

| Subnet Type | Required Tag | Value |
|---|---|---|
| Public | `kubernetes.io/role/elb` | `1` |
| Private | `kubernetes.io/role/internal-elb` | `1` |
| Both | `kubernetes.io/cluster/<cluster-name>` | `shared` |

### 3.3 Understanding IAM Roles

EKS needs **two separate IAM roles**:

| Role | Purpose | Trusted Entity | Required Policies |
|---|---|---|---|
| **EKS Cluster Role** | Lets AWS manage the Kubernetes control plane | `eks.amazonaws.com` | `AmazonEKSClusterPolicy` |
| **EKS Node Group Role** | Lets EC2 worker nodes join the cluster and pull images | `ec2.amazonaws.com` | `AmazonEKSWorkerNodePolicy`, `AmazonEKS_CNI_Policy`, `AmazonEC2ContainerRegistryReadOnly` |

> **If using eksctl** (recommended): eksctl can create these roles automatically. If you have an existing VPC, you may need to create them manually in the AWS Console > IAM > Roles.

### 3.4 Install eksctl (On Your Local Machine or EC2)

```bash
# Install eksctl
curl --silent --location "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_$(uname -m).tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin/
eksctl version
```

### 3.5 Option A - Simple EKS Cluster (eksctl creates everything)

If you don't have an existing VPC, let eksctl create the VPC, subnets, IAM roles, and everything automatically. Create a config file:

```bash
nano cluster-config.yaml
```

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: medfarm-prod-eks
  region: ap-southeast-2
  version: "1.30"

iam:
  withOIDC: true  # CRITICAL: Enables IRSA for service accounts

vpc:
  # Let eksctl create VPC, subnets, NAT gateways, IGW
  managed: true
  cidr: 10.20.0.0/16
  publicSubnetCIDRs:
    - 10.20.0.0/24
    - 10.20.1.0/24
  privateSubnetCIDRs:
    - 10.20.2.0/24
    - 10.20.3.0/24

addons:
  - name: vpc-k8s              # AWS VPC CNI (networking)
    version: latest
  - name: coredns             # DNS for services
    version: latest
  - name: kube-proxy           # Service routing
    version: latest

managedNodeGroups:
  - name: medfarm-ng
    instanceType: t3.medium    # 2 vCPU, 4 GB RAM
    desiredCapacity: 2
    minSize: 1
    maxSize: 5
    volumeSize: 30             # 30 GB EBS gp3 storage
    iam:
      withAddonPolicies:
        autoScaler: true       # Enable cluster autoscaler
    labels:
      role: application
    taints: {}                 # No taints (all pods can schedule)
    tags:
      Environment: production
      Project: med-erp

cloudWatch:
  clusterLogging:
    enableTypes:
      - "api"
      - "audit"
      - "authenticator"
      - "controllerManager"
      - "scheduler"
```

**Field-by-field explanation:**

| Field | What It Does | Why It Matters |
|---|---|---|
| `iam.withOIDC: true` | Creates an IAM OIDC identity provider for the cluster | Required for IRSA (service accounts to assume IAM roles) |
| `vpc.managed: true` | eksctl creates VPC, subnets, NAT, IGW automatically | Easiest option - no manual VPC setup |
| `addons` | Installs essential cluster add-ons | vpc-k8s (networking), coredns (DNS), kube-proxy (routing) |
| `volumeSize: 30` | EBS volume for each node | Needs space for container images |
| `autoScaler: true` | Attaches cluster autoscaler policy | Lets HPA scale the node group if needed |
| `cloudWatch` | Enables EKS control plane logging | Helps debug cluster issues |

**Create the cluster:**

```bash
eksctl create cluster -f cluster-config.yaml
```

> **This takes 15-20 minutes.** Go grab a coffee. eksctl will create: VPC, 4 subnets, 2 NAT gateways, Internet Gateway, route tables, IAM roles, EKS control plane, and 2 worker nodes.

### 3.6 Option B - Use Existing VPC

If you already have a VPC with subnets, create a config that references them:

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: medfarm-prod-eks
  region: ap-southeast-2
  version: "1.30"

iam:
  withOIDC: true

vpc:
  id: vpc-0abc123def456789     # Your existing VPC ID
  subnets:
    public:
      - subnet-public-1a        # 2+ public subnets (across AZs)
      - subnet-public-2b
    private:
      - subnet-private-1a       # 2+ private subnets (across AZs)
      - subnet-private-2b

managedNodeGroups:
  - name: medfarm-ng
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 1
    maxSize: 5
    volumeSize: 30
```

### 3.7 Verify Cluster Creation

```bash
# Configure kubectl to talk to your new cluster
aws eks update-kubeconfig --region ap-southeast-2 --name medfarm-prod-eks

# Check nodes (should show 2 Ready nodes)
kubectl get nodes

# Check namespaces
kubectl get ns

# Check cluster info
kubectl cluster-info

# Check the OIDC provider (needed for Load Balancer Controller IRSA)
aws eks describe-cluster \
  --name medfarm-prod-eks \
  --region ap-southeast-2 \
  --query "cluster.identity.oidc.issuer" \
  --output text
```

> **Expected output for OIDC:** `https://oidc.eks.ap-southeast-2.amazonaws.com/id/XXXXXXXXXXXX` - Save this URL, you'll need it in Phase 8.

### 3.8 Verify Node IAM Role

Confirm the node group role has the required policies:

```bash
# Get the node group role name
NODE_ROLE_ARN=$(aws eks describe-nodegroup \
  --cluster-name medfarm-prod-eks \
  --nodegroup-name medfarm-ng \
  --region ap-southeast-2 \
  --query "nodegroup.nodeRole" \
  --output text)

echo "Node Role ARN: $NODE_ROLE_ARN"

# List attached policies (should see at least 3 EKS policies)
aws iam list-attached-role-policies --role-name $(basename $NODE_ROLE_ARN)
```

You should see:
- `AmazonEKSWorkerNodePolicy`
- `AmazonEKS_CNI_Policy`
- `AmazonEC2ContainerRegistryReadOnly`

If any are missing, attach them:

```bash
aws iam attach-role-policy \
  --role-name $(basename $NODE_ROLE_ARN) \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy

aws iam attach-role-policy \
  --role-name $(basename $NODE_ROLE_ARN) \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy

aws iam attach-role-policy \
  --role-name $(basename $NODE_ROLE_ARN) \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
```

> **Checkpoint:** EKS cluster is running with 2 Ready nodes. OIDC provider is enabled. Node role has all required policies.

---

## Phase 4 - Install Prerequisites on EC2

> **Goal:** Install all required CLI tools on your EC2 command center. Ensure you're still SSH'd in.

### 4.1 Update System

```bash
sudo apt-get update && sudo apt-get upgrade -y
```

### 4.2 Install AWS CLI v2

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt-get install unzip -y
unzip awscliv2.zip
sudo ./aws/install

# Configure with your credentials
aws configure
# Enter: Access Key ID, Secret Access Key
# Region: ap-southeast-2
# Output format: json

# Verify
aws sts get-caller-identity
```

### 4.3 Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Verify
kubectl version --client
```

### 4.4 Install Docker

```bash
sudo apt-get install docker.io -y

# Add your user to docker group (avoids sudo for every docker command)
sudo usermod -aG docker $USER
newgrp docker

# Verify
docker --version
docker run hello-world
```

### 4.5 Install eksctl

```bash
curl --silent --location "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_$(uname -m).tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin/

# Verify
eksctl version
```

### 4.6 Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version
```

### 4.7 Install Java 17 & Maven (For Building Spring Boot)

```bash
sudo apt-get install -y openjdk-17-jdk-headless maven

# Verify
java -version
# Expected: openjdk version "17.x.x"
mvn -version
# Expected: Apache Maven 3.x.x
```

### 4.8 Install Node.js 20 (For Building React Frontend)

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# Verify
node --version
# Expected: v20.x.x
npm --version
```

> **Checkpoint:** All tools installed. Run `docker --version`, `kubectl version --client`, `eksctl version`, `helm version` to confirm.

---

## Phase 5 - Build & Push Docker Images to ECR

> **Goal:** Convert your source code into Docker images and store them in AWS ECR.

### 5.1 Clone Repository

```bash
git clone https://github.com/shubhamkalsait/edublitz-b2b-medical-erp.git
cd edublitz-b2b-medical-erp
git checkout project-b22
```

### 5.2 Login to ECR

```bash
aws ecr get-login-password --region ap-southeast-2 \
  | docker login --username AWS \
    --password-stdin <YOUR_ACCOUNT_ID>.dkr.ecr.ap-southeast-2.amazonaws.com
```

> Replace `<YOUR_ACCOUNT_ID>` with your 12-digit AWS account number.

### 5.3 Create ECR Repositories

> **Important:** The correct repository names are `user-service`, `product-service`, `order-service`, and `frontend`. Do NOT use `user-app` or similar - they won't match the K8s deployment YAMLs.

**Option A - Via CLI:**

```bash
aws ecr create-repository --repository-name user-service --region ap-southeast-2
aws ecr create-repository --repository-name product-service --region ap-southeast-2
aws ecr create-repository --repository-name order-service --region ap-southeast-2
aws ecr create-repository --repository-name frontend --region ap-southeast-2
```

**Option B - Via Console:**

AWS Console > ECR > Create Repository > type name > Create.

### 5.4 Understanding the Dockerfiles

Each backend service uses a **multi-stage Dockerfile**:

```
Stage 1 (Build):        eclipse-temurin:17-jdk-alpine
  └── Copy pom.xml → mvn dependency:go-offline
  └── Copy src → mvn clean package -DskipTests
  └── Output: target/*.jar

Stage 2 (Runtime):      eclipse-temurin:17-jre-alpine
  └── Create non-root user (appuser)
  └── Copy *.jar from Stage 1
  └── HEALTHCHECK on port 8081
  └── JVM flags: -XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0
  └── ENTRYPOINT: java -jar app.jar
```

**Why JDK 17 (not 21)?** The project originally used JDK 21 but encountered library compatibility issues with Spring Boot dependencies. Downgrading to JDK 17 (LTS) resolved all issues.

**The frontend Dockerfile:**

```
Stage 1 (Build):        node:20-alpine
  └── npm ci → npm run build → dist/

Stage 2 (Serve):        nginx:1.27-alpine
  └── Copy dist/ to /usr/share/nginx/html
  └── Copy nginx.conf (SPA routing, gzip, security headers)
  └── HEALTHCHECK on port 80
```

### 5.5 Set Your ECR URI Variable

```bash
# Set this once, use it everywhere
ECR_URI="<YOUR_ACCOUNT_ID>.dkr.ecr.ap-southeast-2.amazonaws.com"
```

### 5.6 Build & Push Backend Images

```bash
# ─── User Service ───
cd user-service
mvn clean package -DskipTests
docker build -t user-service:latest .
docker tag user-service:latest ${ECR_URI}/user-service:latest
docker push ${ECR_URI}/user-service:latest
cd ..

# ─── Product Service ───
cd product-service
mvn clean package -DskipTests
docker build -t product-service:latest .
docker tag product-service:latest ${ECR_URI}/product-service:latest
docker push ${ECR_URI}/product-service:latest
cd ..

# ─── Order Service ───
cd order-service
mvn clean package -DskipTests
docker build -t order-service:latest .
docker tag order-service:latest ${ECR_URI}/order-service:latest
docker push ${ECR_URI}/order-service:latest
cd ..
```

> **Troubleshooting:** If Maven build fails, check if `pom.xml` is in the correct directory. If Docker build fails, ensure you're in the correct service folder (where `Dockerfile` exists).

### 5.7 Build & Push Frontend Image

> **Important:** The frontend needs `VITE_*` environment variables baked in at **build time** (Vite's behavior). These tell the React app where the backend APIs are.

```bash
cd frontend

# Set backend API URL (the ALB/Ingress domain you'll configure in Route 53)
# These are BAKED IN at build time - they cannot be changed after docker build
VITE_USER_SERVICE_URL=https://api.yourdomain.com/api/v1 \
VITE_PRODUCT_SERVICE_URL=https://api.yourdomain.com/api/v1 \
VITE_ORDER_SERVICE_URL=https://api.yourdomain.com/api/v1 \
docker build -t frontend:latest .

docker tag frontend:latest ${ECR_URI}/frontend:latest
docker push ${ECR_URI}/frontend:latest
cd ..
```

**How it works in the code:**

```typescript
// frontend/src/api/axios.ts
export const userApi = createApiClient(
  import.meta.env.VITE_USER_SERVICE_URL ?? '/api/user'   // fallback for local dev
)

export const productApi = createApiClient(
  import.meta.env.VITE_PRODUCT_SERVICE_URL ?? '/api/product'
)

export const orderApi = createApiClient(
  import.meta.env.VITE_ORDER_SERVICE_URL ?? '/api/order'
)
```

> **Note:** If you need to change the API URL later, you must rebuild the frontend image with the new env vars and push it to ECR again.

> **Checkpoint:** All 4 images pushed to ECR. Verify: `aws ecr describe-images --repository-name user-service --region ap-southeast-2`

---

## Phase 6 - SSL Certificates (ACM)

> **Goal:** Enable HTTPS for both frontend and backend.

### 6.1 The Two-Certificate Rule (CRITICAL)

This is the **#1 source of deployment failures** for beginners. Read carefully:

```
┌───────────────────────────────────────────────────────┐
│                  SSL CERTIFICATE RULE                  │
│                                                        │
│  Frontend (S3 + CloudFront)                            │
│    └── Certificate MUST be in us-east-1                │
│    └── Reason: CloudFront is a global service,         │
│        it only trusts certs from us-east-1              │
│                                                        │
│  Backend (EKS + ALB Ingress)                            │
│    └── Certificate MUST be in SAME region as EKS       │
│    └── Example: ap-southeast-2 (Sydney)                │
└───────────────────────────────────────────────────────┘
```

### 6.2 Create Backend Certificate (EKS Region)

1. AWS Console > **Certificate Manager (ACM)**
2. **Switch region to** `ap-southeast-2` (or your EKS region)
3. Click **Request a certificate** > **Request a public certificate**
4. **Domain Name:** `api.yourdomain.com` (use your actual subdomain)
5. **Validation Method:** DNS Validation
6. Click **Request**
7. Expand the certificate > Click **Create record in Route 53** > **Create records**
8. Wait 2-5 minutes until status changes from **Pending validation** to **Issued**
9. **COPY the ARN** (looks like `arn:aws:acm:ap-southeast-2:123456789012:certificate/xxxx-xxxx-xxxx`)

### 6.3 Create Frontend Certificate (us-east-1)

1. **Switch region to** `us-east-1` (North Virginia)
2. AWS Console > **Certificate Manager (ACM)**
3. Click **Request a certificate** > **Request a public certificate**
4. **Domain Name:** `yourdomain.com` (your main domain)
5. **Validation Method:** DNS Validation
6. Click **Request**
7. **Create records in Route 53** > **Create records**
8. Wait for **Issued** status
9. **COPY the ARN**

> **Checkpoint:** You have 2 certificate ARNs: one in `ap-southeast-2` (backend), one in `us-east-1` (frontend).

---

## Phase 7 - S3 Bucket & CloudFront (Frontend)

> **Goal:** Host the React frontend and serve it via CloudFront CDN with HTTPS.

### 7.1 Create S3 Bucket

1. AWS Console > **S3** > **Create bucket**
2. **Bucket name:** `medpharm-frontend-<unique-id>` (must be globally unique)
3. **Block Public Access settings:** Uncheck **Block all public access** > Confirm
4. Click **Create bucket**

### 7.2 Enable Static Website Hosting

1. Select your bucket > **Properties** tab
2. Scroll down to **Static website hosting** > **Edit**
3. Select **Enable**
4. **Index document:** `index.html`
5. **Error document:** `index.html` (for SPA routing)
6. **Save changes**

### 7.3 Upload Frontend Build

```bash
# Option A: Upload the dist/ folder from your build
# (If you built locally or on EC2)
aws s3 cp frontend/dist/ s3://medpharm-frontend-<unique-id>/ --recursive

# Option B: Download from ECR, extract, and upload (more complex)
# Usually you just build locally or on EC2 and upload
```

### 7.4 Create CloudFront Distribution

1. AWS Console > **CloudFront** > **Create distribution**
2. **Origin Domain:** Select your S3 bucket from the dropdown
3. **Origin Protocol Policy:** HTTP only (S3 doesn't support HTTPS directly)
4. **Viewer Protocol Policy:** Redirect HTTP to HTTPS
5. **Alternate Domain Name (CNAME):** `yourdomain.com`
6. **Custom SSL Certificate:** Select the **us-east-1** certificate (from Phase 6.3)
7. **Default Root Object:** `index.html`
8. **Behaviors > Edit:**
   - Path Pattern: `/*`
   - Cache Policy: `CachingOptimized` (or custom for SPA)
   - If you need SPA routing: create a custom error page for 404 → return `/index.html` with 200 response
9. Click **Create distribution**

> **Wait 20-30 minutes** for the distribution status to change to **Deployed**.

### 7.5 Configure Custom Error Page for SPA

> **Important for React Router:** Without this, refreshing any page other than `/` returns a 404 from S3.

1. Select your CloudFront distribution > **Error pages** tab
2. Click **Create custom error response**
3. **HTTP Error Code:** 404
4. **Error Caching TTL:** 0
5. **Customize Error Response:** Yes
6. **Response Page Path:** `/index.html`
7. **HTTP Response Code:** 200
8. Click **Create**

> **Checkpoint:** CloudFront distribution is Deployed. Note the distribution domain name: `dXXXXXXXXXXXX.cloudfront.net`

---

## Phase 8 - IAM Setup for AWS Load Balancer Controller (IRSA)

> **Goal:** Create the IAM role and policy for the AWS Load Balancer Controller to create and manage Application Load Balancers.

### 8.1 Understanding IRSA

**IRSA** (IAM Roles for Service Accounts) is the modern, secure way to give Kubernetes pods AWS permissions. Instead of attaching IAM policies to all EC2 nodes (which gives every pod access), you create a specific IAM role that only the Load Balancer Controller's service account can assume.

```
Without IRSA (Old way - NOT recommended):
  EC2 Node Role ──► Has broad permissions ──► ALL pods on that node inherit those permissions

With IRSA (Modern way - Recommended):
  AWS LBC Service Account ──► Assumes specific IAM Role ──► Only LBC pods have ALB permissions
  All other pods ──► No AWS permissions (unless explicitly configured)
```

### 8.2 Download IAM Policy

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```

### 8.3 Create IAM Policy

```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json \
  --region ap-southeast-2
```

**Save the Policy ARN from the output:**

```
arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy
```

### 8.4 Create IAM Role with OIDC Trust

This is the key step that most beginners miss. You create an IAM role that **trusts your EKS cluster's OIDC provider**:

```bash
# Get your OIDC provider URL and AWS account ID
OIDC_URL=$(aws eks describe-cluster \
  --name medfarm-prod-eks \
  --region ap-southeast-2 \
  --query "cluster.identity.oidc.issuer" \
  --output text)

ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)

echo "OIDC URL: $OIDC_URL"
echo "Account ID: $ACCOUNT_ID"

# Extract the OIDC provider host (without https://)
OIDC_HOST=${OIDC_URL#https://}
echo "OIDC Host: $OIDC_HOST"
```

**Create the trust policy:**

```bash
cat > lb-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/${OIDC_HOST}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${OIDC_HOST}:sub": "system:serviceaccount:kube-system:aws-load-balancer-controller"
        }
      }
    }
  ]
}
EOF
```

**Create the role and attach the policy:**

```bash
# Create IAM role
aws iam create-role \
  --role-name AWSLoadBalancerControllerIAMRole \
  --assume-role-policy-document file://lb-trust-policy.json

# Attach the ALB controller policy
aws iam attach-role-policy \
  --role-name AWSLoadBalancerControllerIAMRole \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy

# Verify
aws iam get-role --role-name AWSLoadBalancerControllerIAMRole --query "Role.Arn" --output text
```

**Save the Role ARN:**

```
arn:aws:iam::<ACCOUNT_ID>:role/AWSLoadBalancerControllerIAMRole
```

### 8.5 Create Security Group for ALB

```bash
# Get your VPC ID
VPC_ID=$(aws eks describe-cluster \
  --name medfarm-prod-eks \
  --region ap-southeast-2 \
  --query "cluster.resourcesVpcConfig.vpcId" \
  --output text)

# Create Security Group
SG_ID=$(aws ec2 create-security-group \
  --group-name medfarm-alb-sg \
  --description "Security group for MedPharm ALB" \
  --vpc-id $VPC_ID \
  --query "GroupId" \
  --output text)

echo "Security Group ID: $SG_ID"

# Allow HTTPS inbound (443)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# Allow HTTP inbound (80) - for redirect
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# Allow all outbound
aws ec2 authorize-security-group-egress \
  --group-id $SG_ID \
  --protocol all \
  --port -1 \
  --cidr 0.0.0.0/0
```

> **Checkpoint:** You have: IAM Policy ARN, IAM Role ARN, and Security Group ID. All needed for the next phases.

---

## Phase 9 - Kubernetes Deployment

> **Goal:** Deploy all microservices to the EKS cluster using Kubernetes manifests.

### 9.1 Configure kubectl

```bash
aws eks update-kubeconfig --region ap-southeast-2 --name medfarm-prod-eks
kubectl get nodes   # Should show 2 Ready nodes
```

### 9.2 Namespace

The namespace isolates all project resources:

```bash
kubectl apply -f k8s/namespace/
# Creates: namespace "med-erp"
```

Verify: `kubectl get ns med-erp`

### 9.3 Secrets

> **CRITICAL FIX:** Many beginners forget the `JWT_SECRET` key. The repo requires **4 keys** in the secret, not 3. Without JWT_SECRET, all 3 services will fail to start.

```bash
kubectl create secret generic app-secrets \
  -n med-erp \
  --from-literal=MONGODB_URI_USER="mongodb+srv://admin:YOUR_PASSWORD@edublitz-cluster.xxxxx.mongodb.net/users_db?retryWrites=true&w=majority" \
  --from-literal=MONGODB_URI_PRODUCT="mongodb+srv://admin:YOUR_PASSWORD@edublitz-cluster.xxxxx.mongodb.net/products_db?retryWrites=true&w=majority" \
  --from-literal=MONGODB_URI_ORDER="mongodb+srv://admin:YOUR_PASSWORD@edublitz-cluster.xxxxx.mongodb.net/orders_db?retryWrites=true&w=majority" \
  --from-literal=JWT_SECRET="404E635266556A586E3272357538782F413F4428472B4B6250645367566B5970" \
  --dry-run=client -o yaml | kubectl apply -f -
```

**What each key does:**

| Key | Used By | Description |
|---|---|---|
| `MONGODB_URI_USER` | user-service | MongoDB connection string for `users_db` |
| `MONGODB_URI_PRODUCT` | product-service | MongoDB connection string for `products_db` |
| `MONGODB_URI_ORDER` | order-service | MongoDB connection string for `orders_db` |
| `JWT_SECRET` | ALL 3 services | 256-bit hex key for signing JWT tokens (must be identical across services) |

> **Note:** The Kubernetes deployment YAMLs reference these keys via `secretKeyRef`. The application.yml files use `${JWT_SECRET:default}` syntax with environment variable injection.

### 9.4 ConfigMap

```bash
kubectl apply -f k8s/configmaps/
```

**The ConfigMap contains:**

| Key | Value | Used By | Description |
|---|---|---|---|
| `JWT_EXPIRATION` | `86400000` | user-service | Token expires in 24 hours (ms) |
| `JWT_REFRESH_EXPIRATION` | `604800000` | user-service | Refresh token expires in 7 days (ms) |
| `CORS_ALLOWED_ORIGINS` | Your frontend domain | ALL 3 services | **Must match** your CloudFront domain |
| `LOW_STOCK_THRESHOLD` | `10` | product-service | Alert when stock falls below 10 |
| `PRODUCT_SERVICE_URL` | `http://product-service:8082/api/v1` | order-service | Internal URL for product lookups |
| `USER_SERVICE_URL` | `http://user-service:8081/api/v1` | order-service | Internal URL for user validation |

> **Important:** Update `CORS_ALLOWED_ORIGINS` in `k8s/configmaps/app-config.yaml` to your actual frontend domain (e.g., `https://yourdomain.com`). If this doesn't match, the frontend will get CORS errors.

### 9.5 Update Deployment YAMLs (Image References)

Replace the placeholder `YOUR_ECR_REGISTRY` with your actual ECR URI:

```bash
# Find and replace in all deployment files
ECR="<YOUR_ACCOUNT_ID>.dkr.ecr.ap-southeast-2.amazonaws.com"

sed -i "s|YOUR_ECR_REGISTRY|${ECR}|g" k8s/deployments/user-service-deployment.yaml
sed -i "s|YOUR_ECR_REGISTRY|${ECR}|g" k8s/deployments/product-service-deployment.yaml
sed -i "s|YOUR_ECR_REGISTRY|${ECR}|g" k8s/deployments/order-service-deployment.yaml
```

**Key features of each deployment (already configured in the YAMLs):**

| Feature | Value | Why It Matters |
|---|---|---|
| Replicas | 2 | High availability |
| Strategy | RollingUpdate (maxSurge:1, maxUnavailable:0) | Zero-downtime updates |
| CPU Request/Limit | 250m / 500m | Fair scheduling, prevent resource hogging |
| Memory Request/Limit | 512Mi / 1Gi | Fair scheduling, prevent OOM kills |
| Readiness Probe | `/api/v1/actuator/health/readiness` (30s delay) | Pod added to service only when ready |
| Liveness Probe | `/api/v1/actuator/health/liveness` (60s delay) | Restarts pod if it crashes |
| Pre-stop Hook | `sleep 10` | Graceful shutdown (finish in-flight requests) |
| Pod Anti-affinity | Spread pods across nodes | If one node dies, pods survive |
| Prometheus Annotations | `/api/v1/actuator/prometheus` | Metrics scraping for monitoring |

### 9.6 Services

```bash
kubectl apply -f k8s/services/
```

All services are **ClusterIP** (internal only):

| Service Name | Port | Target | Accessible By |
|---|---|---|---|
| `user-service` | 8081 | user-service pods | Ingress, other pods |
| `product-service` | 8082 | product-service pods | Ingress, other pods |
| `order-service` | 8083 | order-service pods | Ingress, other pods |

### 9.7 HPA (Horizontal Pod Autoscaler)

```bash
kubectl apply -f k8s/hpa/
```

| Service | Min Pods | Max Pods | CPU Threshold | Memory Threshold |
|---|---|---|---|---|
| user-service | 2 | 6 | 70% | 80% |
| product-service | 2 | 8 | 70% | 80% |
| order-service | 2 | 6 | 70% | 80% |

> **How HPA works:** When CPU exceeds 70% or Memory exceeds 80%, Kubernetes automatically creates more pods (up to max). When load drops, it removes extra pods (down to min).

### 9.8 Install AWS Load Balancer Controller

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=medfarm-prod-eks \
  --set serviceAccount.create=true \
  --set serviceAccount.annotations."eks\.amazonaws\.io/role-arn"=arn:aws:iam::<ACCOUNT_ID>:role/AWSLoadBalancerControllerIAMRole
```

> Replace `<ACCOUNT_ID>` with your actual AWS account ID and the role ARN from Phase 8.

**Verify installation:**

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
# Should show: Running
```

### 9.9 Configure Ingress

Before applying, update `k8s/ingress/ingress.yaml` with your values:

```bash
nano k8s/ingress/ingress.yaml
```

**Update these 3 fields:**

```yaml
# 1. Certificate ARN (from Phase 6.2 - backend cert in EKS region)
alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-southeast-2:ACCOUNT_ID:certificate/YOUR_CERT_ID

# 2. Security Group (from Phase 8.5)
alb.ingress.kubernetes.io/security-groups: sg-XXXXXXXXXX

# 3. Host (your API domain)
spec:
  rules:
    - host: api.yourdomain.com    # Your actual domain
```

**Path routing configuration (already in the YAML):**

| Path | Routes To | Service Port |
|---|---|---|
| `/api/v1/auth` | user-service | 8081 |
| `/api/v1/users` | user-service | 8081 |
| `/api/v1/organizations` | user-service | 8081 |
| `/api/v1/products` | product-service | 8082 |
| `/api/v1/orders` | order-service | 8083 |

### 9.10 Apply All Manifests

```bash
# Order matters - dependencies first
kubectl apply -f k8s/namespace/          # 1. Namespace (everything else needs this)
kubectl apply -f k8s/configmaps/         # 2. ConfigMaps (deployments reference these)
kubectl apply -f k8s/secrets/            # 3. Secrets (deployments reference these)
kubectl apply -f k8s/services/            # 4. Services (ingress references these)
kubectl apply -f k8s/deployments/         # 5. Deployments
kubectl apply -f k8s/hpa/                # 6. HPA (references deployments)
kubectl apply -f k8s/ingress/            # 7. Ingress (references services + SG + cert)
```

### 9.11 Verify Everything

```bash
# Check pods (all should be Running, 2 replicas each)
kubectl get pods -n med-erp

# Check services
kubectl get svc -n med-erp

# Check HPA
kubectl get hpa -n med-erp

# Check Ingress (ADDRESS field shows the ALB DNS name)
kubectl get ingress -n med-erp

# Watch rollout progress
kubectl rollout status deployment/user-service -n med-erp
kubectl rollout status deployment/product-service -n med-erp
kubectl rollout status deployment/order-service -n med-erp
```

> **Expected Ingress output:** The ADDRESS field should show something like `k8s-med-erp-med-erp-xxxx.ap-southeast-2.elb.amazonaws.com`. Save this URL for Route 53.

**If pods are not Running:**

```bash
# Describe a pod to see events
kubectl describe pod <pod-name> -n med-erp

# View logs
kubectl logs <pod-name> -n med-erp --previous
```

> **Checkpoint:** All pods Running. Ingress has an ADDRESS (ALB DNS name). HPA shows current replicas.

---

## Phase 10 - DNS Configuration (Route 53)

> **Goal:** Connect your custom domain names to CloudFront (frontend) and ALB (backend).

### Prerequisite

You need a **Hosted Zone** in Route 53 for your domain. If you don't have one:

1. Route 53 > **Hosted Zones** > **Create Hosted Zone**
2. Enter your domain name (e.g., `yourdomain.com`)
3. Click **Create**

### 10.1 Frontend DNS (A Record → CloudFront)

1. Route 53 > Select your **Hosted Zone**
2. Click **Create Record**
3. **Record Name:** `yourdomain.com` (or empty for root)
4. **Record Type:** **A** (Alias)
5. **Route traffic to:** **Alias to CloudFront distribution**
6. Select your CloudFront distribution from the dropdown
7. Click **Create Record**

### 10.2 Backend DNS (CNAME → ALB)

1. In the **same Hosted Zone**, click **Create Record**
2. **Record Name:** `api` (creates `api.yourdomain.com`)
3. **Record Type:** **CNAME**
4. **Value:** The ALB address from `kubectl get ingress -n med-erp` (the ADDRESS field)
5. Click **Create Record**

### 10.3 Verify DNS Propagation

```bash
# Test frontend
dig yourdomain.com
# Should return CloudFront IPs

# Test backend
dig api.yourdomain.com
# Should return ALB IP
```

> DNS propagation takes 5-10 minutes, sometimes up to 30 minutes globally.

> **Checkpoint:** `https://yourdomain.com` serves the React app. `https://api.yourdomain.com` is ready.

---

## Phase 11 - Final Verification & Testing

> **Goal:** Test the entire system end-to-end.

### 11.1 Backend Health Check

```bash
# Test via domain
curl https://api.yourdomain.com/api/v1/actuator/health

# Expected: {"status": "UP"}
```

### 11.2 Swagger UI (API Documentation)

Open in your browser:

| Service | URL |
|---|---|
| User Service | `https://api.yourdomain.com/api/v1/swagger-ui.html` |
| Product Service | `https://api.yourdomain.com/api/v1/swagger-ui.html` |
| Order Service | `https://api.yourdomain.com/api/v1/swagger-ui.html` |

### 11.3 Frontend

Open `https://yourdomain.com` in your browser. You should see the **Login page**.

### 11.4 End-to-End Test

1. **Register an Organization** (via Swagger or frontend)
2. **Register an Admin User** with email/password
3. **Log In** through the frontend
4. **Create a Product** (e.g., "Paracetamol 500mg")
5. **Place an Order** for the product
6. **Verify in MongoDB Atlas** that the order appears in `orders_db.orders` collection

### 11.5 Verify K8s Resources

```bash
# Full status check
echo "=== Pods ==="
kubectl get pods -n med-erp -o wide

echo "=== Services ==="
kubectl get svc -n med-erp

echo "=== HPA ==="
kubectl get hpa -n med-erp

echo "=== Ingress ==="
kubectl get ingress -n med-erp

echo "=== Resource Summary ==="
kubectl top pods -n med-erp
```

---

## Troubleshooting Guide

| Error | Cause | Fix |
|---|---|---|
| **ImagePullBackOff** | ECR image URL wrong, or image not pushed | Check `kubectl describe pod <pod>` - look at `image:` field. Verify image exists in ECR |
| **CrashLoopBackOff** | Missing env vars, wrong MongoDB URI, missing JWT_SECRET | Run `kubectl logs <pod> -n med-erp --previous`. Check secrets and configmaps |
| **Pending (pod stuck)** | Insufficient node resources, node not ready | Check `kubectl describe node`. Scale node group if needed |
| **502 Bad Gateway** | ALB target group not healthy (pods not ready yet) | Wait 2-3 minutes. Check target health in AWS Console > EC2 > Target Groups |
| **Invalid identity token** | IRSA not configured for LB controller | Fix Phase 8: verify IAM role ARN, OIDC provider, trust policy |
| **CORS errors in browser** | `CORS_ALLOWED_ORIGINS` in ConfigMap doesn't match frontend domain | Update ConfigMap: `kubectl edit cm app-config -n med-erp` |
| **Certificate error** | Wrong region for cert (frontend cert not in us-east-1) | Re-read Phase 6: Two-Certificate Rule |
| **DNS not resolving** | Route 53 records not propagated yet | Wait 10-15 minutes. Use `dig` or `nslookup` to check |
| **MongoDB connection refused** | Wrong URI, Atlas IP whitelist blocking, or wrong DB name | Check secret: `kubectl get secret app-secrets -n med-erp -o yaml`. Verify Atlas Network Access |
| **JWT validation fails across services** | `JWT_SECRET` differs between services | All 3 services MUST have the same JWT_SECRET |
| **Ingress has no ADDRESS** | AWS LB Controller not running or IRSA misconfigured | Check: `kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller` |
| **Frontend shows blank page or 404** | CloudFront custom error pages not configured, or S3 bucket empty | Re-check Phase 7.5 (custom error page for SPA routing) |

---

## Useful kubectl Commands

```bash
# ─── Viewing ───────────────────────────────
kubectl get all -n med-erp                    # See all resources
kubectl get pods -n med-erp -o wide          # Pods with node info
kubectl get events -n med-erp --sort-by=.metadata.creationTimestamp  # Recent events

# ─── Debugging ────────────────────────────
kubectl logs -f deployment/user-service -n med-erp --tail=100       # Stream logs
kubectl logs <pod> -n med-erp --previous                              # Crash logs
kubectl describe pod <pod> -n med-erp                                 # Detailed pod info
kubectl describe ingress med-erp-ingress -n med-erp                  # Ingress events (ALB status)

# ─── Exec Into Pod ────────────────────────
kubectl exec -it <pod-name> -n med-erp -- sh

# ─── Scale & Restart ──────────────────────
kubectl scale deployment/product-service --replicas=3 -n med-erp    # Manual scale
kubectl rollout restart deployment/order-service -n med-erp          # Restart all pods
kubectl rollout undo deployment/user-service -n med-erp              # Rollback to previous

# ─── Update Image ─────────────────────────
kubectl set image deployment/user-service \
  user-service=<ACCOUNT_ID>.dkr.ecr.ap-southeast-2.amazonaws.com/user-service:v1.1.0 \
  -n med-erp

# ─── Port Forward (local testing) ─────────
kubectl port-forward svc/user-service 8081:8081 -n med-erp
kubectl port-forward svc/product-service 8082:8082 -n med-erp

# ─── Resources ────────────────────────────
kubectl top pods -n med-erp                     # CPU/Memory usage
kubectl top nodes                               # Node resource usage
kubectl get hpa -n med-erp                      # Autoscaler status
```

---

## Appendix - Complete YAML Reference

### Namespace (`k8s/namespace/namespace.yaml`)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: med-erp
  labels:
    app.kubernetes.io/name: med-erp
    environment: production
```

### Secrets (`k8s/secrets/app-secrets.yaml`)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: med-erp
type: Opaque
data:
  # echo -n "your-mongodb-uri" | base64
  MONGODB_URI_USER: <base64-encoded>
  MONGODB_URI_PRODUCT: <base64-encoded>
  MONGODB_URI_ORDER: <base64-encoded>
  # echo -n "your-256-bit-hex-jwt-secret" | base64
  JWT_SECRET: <base64-encoded>
```

### ConfigMap (`k8s/configmaps/app-config.yaml`)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: med-erp
  labels:
    app.kubernetes.io/part-of: med-erp
data:
  JWT_EXPIRATION: "86400000"
  JWT_REFRESH_EXPIRATION: "604800000"
  CORS_ALLOWED_ORIGINS: "https://yourdomain.com"      # UPDATE THIS
  LOW_STOCK_THRESHOLD: "10"
  PRODUCT_SERVICE_URL: "http://product-service:8082/api/v1"
  USER_SERVICE_URL: "http://user-service:8081/api/v1"
```

### User Service Deployment (`k8s/deployments/user-service-deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: med-erp
  labels:
    app: user-service
    app.kubernetes.io/name: user-service
    app.kubernetes.io/part-of: med-erp
    version: "1.0.0"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: user-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: user-service
        version: "1.0.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/api/v1/actuator/prometheus"
        prometheus.io/port: "8081"
    spec:
      serviceAccountName: default
      terminationGracePeriodSeconds: 60
      containers:
        - name: user-service
          image: <ACCOUNT_ID>.dkr.ecr.ap-southeast-2.amazonaws.com/user-service:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8081
              protocol: TCP
          env:
            - name: MONGODB_URI
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MONGODB_URI_USER
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: JWT_SECRET
            - name: JWT_EXPIRATION
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: JWT_EXPIRATION
            - name: JWT_REFRESH_EXPIRATION
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: JWT_REFRESH_EXPIRATION
            - name: CORS_ALLOWED_ORIGINS
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: CORS_ALLOWED_ORIGINS
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "500m"
              memory: "1Gi"
          readinessProbe:
            httpGet:
              path: /api/v1/actuator/health/readiness
              port: 8081
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /api/v1/actuator/health/liveness
              port: 8081
            initialDelaySeconds: 60
            periodSeconds: 30
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: user-service
                topologyKey: kubernetes.io/hostname
```

> The `product-service-deployment.yaml` and `order-service-deployment.yaml` follow the same pattern. The key differences: port numbers (8082/8083), image names, and environment variables.

### Services (`k8s/services/services.yaml`)

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: med-erp
spec:
  type: ClusterIP
  selector:
    app: user-service
  ports:
    - name: http
      port: 8081
      targetPort: 8081
---
apiVersion: v1
kind: Service
metadata:
  name: product-service
  namespace: med-erp
spec:
  type: ClusterIP
  selector:
    app: product-service
  ports:
    - name: http
      port: 8082
      targetPort: 8082
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: med-erp
spec:
  type: ClusterIP
  selector:
    app: order-service
  ports:
    - name: http
      port: 8083
      targetPort: 8083
```

### HPA (`k8s/hpa/hpa.yaml`)

```yaml
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service-hpa
  namespace: med-erp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service
  minReplicas: 2
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: product-service-hpa
  namespace: med-erp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: product-service
  minReplicas: 2
  maxReplicas: 8
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
  namespace: med-erp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 2
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

### Ingress (`k8s/ingress/ingress.yaml`)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: med-erp-ingress
  namespace: med-erp
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/load-balancer-name: med-erp-alb
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/certificate-arn: <PASTE_YOUR_BACKEND_CERT_ARN>
    alb.ingress.kubernetes.io/healthcheck-path: /api/v1/actuator/health
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "30"
    alb.ingress.kubernetes.io/healthy-threshold-count: "2"
    alb.ingress.kubernetes.io/unhealthy-threshold-count: "3"
    alb.ingress.kubernetes.io/security-groups: <PASTE_YOUR_ALB_SG_ID>
    alb.ingress.kubernetes.io/tags: "Environment=production,Project=med-erp"
spec:
  rules:
    - host: api.yourdomain.com     # UPDATE THIS
      http:
        paths:
          - path: /api/v1/auth
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 8081
          - path: /api/v1/users
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 8081
          - path: /api/v1/organizations
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 8081
          - path: /api/v1/products
            pathType: Prefix
            backend:
              service:
                name: product-service
                port:
                  number: 8082
          - path: /api/v1/orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 8083
```

---

## Quick Reference - Environment Variables Flow

```
┌──────────────────────────────────────────────────────────┐
│                   Environment Variable Flow                 │
│                                                             │
│  K8s Secret (app-secrets)                                  │
│    ├── MONGODB_URI_USER   ──► user-service    → MONGODB_URI │
│    ├── MONGODB_URI_PRODUCT ─► product-service → MONGODB_URI │
│    ├── MONGODB_URI_ORDER   ─► order-service   → MONGODB_URI │
│    └── JWT_SECRET          ──► ALL services    → jwt.secret │
│                                                             │
│  K8s ConfigMap (app-config)                                 │
│    ├── JWT_EXPIRATION        ──► user-service               │
│    ├── JWT_REFRESH_EXPIRATION──► user-service               │
│    ├── CORS_ALLOWED_ORIGINS  ──► ALL services               │
│    ├── LOW_STOCK_THRESHOLD   ──► product-service            │
│    ├── PRODUCT_SERVICE_URL   ──► order-service              │
│    └── USER_SERVICE_URL      ──► order-service              │
│                                                             │
│  Frontend Build (VITE_*, baked in at build time)            │
│    ├── VITE_USER_SERVICE_URL   ──► React axios base URL     │
│    ├── VITE_PRODUCT_SERVICE_URL──► React axios base URL     │
│    └── VITE_ORDER_SERVICE_URL  ──► React axios base URL     │
└──────────────────────────────────────────────────────────┘
```

---

## Deployment Checklist

Use this checklist before going live:

- [ ] MongoDB Atlas: 3 databases created (`users_db`, `products_db`, `orders_db`)
- [ ] MongoDB Atlas: Database user created, network access configured
- [ ] EC2 Command Center: Ubuntu 22.04, t3.medium, tools installed
- [ ] EKS Cluster: 2+ nodes Running, OIDC enabled
- [ ] Node IAM Role: 3 required policies attached
- [ ] ECR: 4 repositories created (user-service, product-service, order-service, frontend)
- [ ] Docker Images: All 4 images built and pushed to ECR
- [ ] Backend SSL Certificate: Created in EKS region (ap-southeast-2), validated, ARN copied
- [ ] Frontend SSL Certificate: Created in us-east-1, validated, ARN copied
- [ ] S3 Bucket: Created, static hosting enabled, frontend dist uploaded
- [ ] CloudFront: Distribution created, us-east-1 cert attached, custom error page for SPA configured
- [ ] IAM: LB Controller IAM policy created, IAM role created with OIDC trust
- [ ] ALB Security Group: Created, ports 80/443 open
- [ ] K8s Namespace: `med-erp` created
- [ ] K8s Secrets: `app-secrets` with 4 keys (3 MongoDB URIs + JWT_SECRET)
- [ ] K8s ConfigMap: `app-config` with `CORS_ALLOWED_ORIGINS` matching frontend domain
- [ ] K8s Deployments: Image URIs updated to ECR, all 3 services applied
- [ ] K8s Services: ClusterIP services for user/product/order-service
- [ ] K8s HPA: Autoscalers for all 3 services
- [ ] AWS LB Controller: Installed via Helm with correct IRSA role ARN
- [ ] K8s Ingress: Certificate ARN, security group ID, and host domain updated
- [ ] Route 53: A record (frontend → CloudFront), CNAME (api → ALB)
- [ ] Final Test: Health check returns UP, frontend shows login page, E2E flow works

---

> **Repository:** [edublitz-b2b-medical-erp](https://github.com/shubhamkalsait/edublitz-b2b-medical-erp/tree/project-b22)
>
> **Kubernetes Guide:** [KUBERNETES_DEPLOYMENT.md](https://github.com/shubhamkalsait/edublitz-b2b-medical-erp/blob/project-b22/docs/KUBERNETES_DEPLOYMENT.md)
