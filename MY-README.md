# 🚀 Ultimate Beginner's Guide: MedPharm ERP Deployment

## 🗺️ Navigation Roadmap
This guide is divided into **5 Phases**.
1.  🟢 **Phase 1:** Infrastructure Setup (Cloud Resources).
2.  🟡 **Phase 2:** Installing Tools (On the Command Center).
3.  🟠 **Phase 3:** Building & Pushing Code (Docker & ECR).
4.  🟣 **Phase 4:** Security Setup (SSL Certificates).
5.  🔵 **Phase 5:** Kubernetes Deployment (The Final Step).

---

## 🟢 Phase 1: Infrastructure Setup

### 1. Create MongoDB Database (Atlas)

**Goal:** Create a cloud database to store your User, Product, and Order data.

### 🌐 1. Login to MongoDB Atlas
1.  Open your web browser and go to: [https://www.mongodb.com/atlas](https://www.mongodb.com/atlas)
2.  **Log in** with your email/password (or **Sign Up** for free if you don't have an account).

---

### ⚙️ 2. Build a Free Database
1.  On the dashboard, locate and click the big green button: **Build a Database**.
2.  Select the **M0 Sandbox** option.
    *   *Note:* This is the **Free** tier. It is perfect for learning and projects.
3.  **Cluster Name:** Type `edublitz-cluster`.
4.  **Region:** Click the dropdown menu.
    *   Select the region closest to you (e.g., **Singapore**, **Mumbai**, or **Sydney**).
    *   *Why?* This ensures your app runs fast by being close to your data.
5.  Click **Create**.
    *   *Wait:* It might take 2-3 minutes to initialize.

---

### 🔑 3. Create Database User (Security)
You need a username and password to let your application talk to the database.

1.  Look at the **Left Sidebar** (menu on the left) and click on **Database Access**.
2.  Click the green button: **Add New Database User**.
3.  **Authentication Method:**
    *   Select **Password**.
    *   **Username:** Type `admin`
    *   **Password:** Type `Redhat@123`
    *   **⚠️ IMPORTANT:** Save this password in a safe place (like your Notepad)! You will need it later.
4.  Click **Add User**.

---

### 🌍 4. Allow Network Access (Firewall)
By default, MongoDB blocks all connections. You must open the door for your AWS server.

1.  On the **Left Sidebar**, click on **Network Access**.
2.  Click the green button: **Add IP Address**.
3.  Select the option: **Allow Access from Anywhere**.
4.  You will see `0.0.0.0/0` appear in the box.
    *   *Note:* This means any computer with the password can connect. Keep this for learning purposes.
5.  Click **Confirm**.

---

### 💾 5. Create Databases
Now we will create the actual storage containers for your data.

1.  On the **Left Sidebar**, click on **Browse Collections**.
2.  You will see a button that says **Add My Own Data**. Click it.
3.  A popup window will appear. You need to create **3 separate databases**. Follow these steps for each one:

#### Database 1: Users
*   **Database Name:** Type `users-db`
*   **Collection Name:** Type `users`
*   Click **Create**.

#### Database 2: Products
*   Click **Add My Own Data** again.
*   **Database Name:** Type `products-db`
*   **Collection Name:** Type `products`
*   Click **Create**.

#### Database 3: Orders
*   Click **Add My Own Data** again.
*   **Database Name:** Type `orders-db`
*   **Collection Name:** Type `orders`
*   Click **Create**.

---

### 🔗 6. Get Connection String
This string is the "address" your code uses to find the database.

1.  On the **Database Deployment** screen (where you created the cluster), click the **Connect** button.
2.  Choose **Connect your application**.
3.  Under **Select your driver's version**, leave it as default (Node.js or whatever is selected).
4.  You will see a code box with a string that looks like this:
    `mongodb+srv://<username>:<password>@edublitz-cluster.xxxx.mongodb.net/?retryWrites=true&w=majority`
5.  **Copy this entire string**.
6.  **Paste it into your Notepad**.
    *   *Tip:* You will replace `<password>` with `Redhat@123` in your code later.

---


### 2. Create Command Center (EC2)
*Goal: Create a powerful Linux server to act as your main control panel.*

1.  Go to **AWS Console** > **EC2** > **Instances** > **Launch Instance**.
2.  **Name:** `Command-Center`.
3.  **OS Images:** Select **Ubuntu Server 22.04 LTS**.
4.  **Instance Type:** Select **t3.medium** (Crucial: Needs 4GB RAM to build Java/Docker images).
5.  **Key Pair:**
    *   Create a new Key Pair (e.g., `project-key.pem`).
    *   **Download** the `.pem` file immediately. You won't get a second chance.
6.  **Network Settings:**
    *   Click **Edit** on Network Settings.
    *   Create Security Group.
    *   Add Rule: **SSH** (Port 22) from **My IP**.
    *   Add Rule: **All Traffic** (All ports, All protocols) from `0.0.0.0/0` (Required for internet access).
7.  **Launch Instance**.

**✅ Checkpoint: Connect to EC2**
Open your local terminal/command prompt and run:
```bash
chmod 400 project-key.pem
ssh -i project-key.pem ubuntu@<YOUR-EC2-PUBLIC-IP>
```
*If you see `ubuntu@ip-...` you are successfully inside your Command Center.*

---

### 3. Create EKS Cluster
*Goal: Create the Kubernetes cluster to run your containers.*

1.  **On your Command Center (EC2)**, create a config file:
    ```bash
    nano cluster-config.yaml
    ```
2.  Paste this content:
    ```yaml
    apiVersion: eksctl.io/v1alpha5
    kind: ClusterConfig

    metadata:
      name: medfarm-prod-eks
      region: ap-southeast-2  # Sydney (Or your chosen region)

    managedNodeGroups:
      - name: medfarm-nodes
        instanceType: t3.medium
        desiredCapacity: 2
        minSize: 1
        maxSize: 3
    ```
    *(Press `Ctrl+X`, then `Y`, then `Enter` to save and exit)*
3.  Create Cluster:
    ```bash
    eksctl create cluster -f cluster-config.yaml
    ```
    *⏳ Wait 15-20 minutes. Go grab a coffee.*

---

### 4. S3 Setup (Frontend)
*Goal: Host the React frontend.*

1.  Go to **AWS Console** > **S3**.
2.  **Create Bucket:** Name it `medpharm-frontend-project` (must be globally unique).
3.  Uncheck **Block all public access**.
4.  **Properties** tab > Scroll down > **Static website hosting** > **Edit** > **Enable**.
5.  **Permissions** tab > **Block public access** > Turn OFF "Block all public access".
6.  **Objects** tab > Upload your React `dist` or `build` folder contents.

---

## 🟡 Phase 2: Install Prerequisites (On EC2 Command Center)

**Ensure you are still SSH'd into your EC2 instance.**

### A. Install AWS CLI & Kubectl
```bash
# AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install
aws configure  # Enter your Access Key and Secret Key here

# Kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

### B. Install Docker & Build Tools
```bash
# Docker
sudo apt-get update
sudo apt-get install docker.io -y
sudo usermod -aG docker $USER
newgrp docker

# Maven (Java Build Tool)
sudo apt install maven -y

# Node.js (Frontend Build Tool)
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs
```

### C. Install Eksctl & Helm
```bash
# Eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin/

# Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

---

## 🟠 Phase 3: Clone, Build & Push

### 5. Clone Repository
```bash
git clone https://github.com/shubhamkalsait/edublitz-b2b-medical-erp.git
cd edublitz-b2b-medical-erp/project-b22
```

### 6. Build & Push Docker Images
*Goal: Turn your Java code into Docker images and store them in AWS.*

**A. Login to ECR (Replace `<YOUR_ACCOUNT_ID>` with your actual 12-digit AWS ID)**
```bash
aws ecr get-login-password --region ap-southeast-2 | docker login --username AWS --password-stdin <YOUR_ACCOUNT_ID>.dkr.ecr.ap-southeast-2.amazonaws.com
```

**B. Create ECR Repositories (Do this once)**
*Go to AWS Console > ECR > Create Repository.* Create 3 repos: `user-app`, `product-app`, `order-app`.

**C. Build & Push (Repeat for all 3 services)**
```bash
# 1. User App
cd user-app
mvn clean package
docker build -t user-app .
docker tag user-app:latest <YOUR_ACCOUNT_ID>.dkr.ecr.ap-southeast-2.amazonaws.com/user-app:latest
docker push <YOUR_ACCOUNT_ID>.dkr.ecr.ap-southeast-2.amazonaws.com/user-app:latest
cd ..

# 2. Product App
cd product-app
mvn clean package
docker build -t product-app .
docker tag product-app:latest <YOUR_ACCOUNT_ID>.dkr.ecr.ap-southeast-2.amazonaws.com/product-app:latest
docker push <YOUR_ACCOUNT_ID>.dkr.ecr.ap-southeast-2.amazonaws.com/product-app:latest
cd ..

# 3. Order App
cd order-app
mvn clean package
docker build -t order-app .
docker tag order-app:latest <YOUR_ACCOUNT_ID>.dkr.ecr.ap-southeast-2.amazonaws.com/order-app:latest
docker push <YOUR_ACCOUNT_ID>.dkr.ecr.ap-southeast-2.amazonaws.com/order-app:latest
cd ..
```

---

## 🟣 Phase 4: SSL & Certificates

### 7. Create SSL Certificate (ACM)
*Goal: Enable HTTPS (Secure) for your backend API.*

1.  Go to **AWS Console** > **Certificate Manager (ACM)**.
2.  **CRITICAL:** Switch the region to **ap-southeast-2** (Must match your EKS region).
3.  Click **Request a certificate** > **Request a public certificate**.
4.  **Domain Name:** `api.edublitz-b2b-erp.online` (Use your own domain if you have one).
5.  **Validation Method:** **DNS validation**.
6.  Click **Request**.
7.  **Validate:** Click the domain > **Create record in Route 53** > **Create records**.
8.  Wait 2-5 minutes until status changes from **Pending validation** to **Issued**.
9.  **Copy the ARN** (It looks like `arn:aws:acm:...`).

---

## 🔵 Phase 5: Backend YAML Deployment

### 8. Configure Kubeconfig
```bash
aws eks update-kubeconfig --region ap-southeast-2 --name medfarm-prod-eks
kubectl get nodes
# You should see 2 nodes listed as Ready
```

### 9. Create Secrets (Database Credentials)
*We store the password securely so it's not visible in plain text.*
```bash
kubectl create secret generic app-secrets \
  -n med-erp \
  --from-literal=MONGODB_URI_USER="mongodb+srv://admin:Redhat@123@edublitz-cluster.xxxx.mongodb.net/users-tv?retryWrites=true&w=majority" \
  --from-literal=MONGODB_URI_PRODUCT="mongodb+srv://admin:Redhat@123@edublitz-cluster.xxxx.mongodb.net/products-tv?retryWrites=true&w=majority" \
  --from-literal=MONGODB_URI_ORDER="mongodb+srv://admin:Redhat@123@edublitz-cluster.xxxx.mongodb.net/orders-tv?retryWrites=true&w=majority" \
  --dry-run=client -o yaml | kubectl apply -f -
```

### 10. Update Deployment YAMLs
*Goal: Tell Kubernetes to use the images you just pushed to ECR.*

1.  Open the deployment files:
    ```bash
    nano k8s/deployments/user-deployment.yaml
    ```
2.  Find the line `image: user-app:latest`.
3.  Replace it with your ECR image:
    `image: <YOUR_ACCOUNT_ID>.dkr.ecr.ap-southeast-2.amazonaws.com/user-app:latest`
4.  Save and Exit (`Ctrl+X`, `Y`, `Enter`).
5.  Repeat for `product-deployment.yaml` and `order-deployment.yaml`.

### 11. Setup IAM Role for Ingress (MISSING STEP!)
*If you skip this, your Load Balancer will fail with "Invalid identity token".*

1.  Create a file `iam-policy.json`:
    ```bash
    curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
    ```
2.  Create the policy:
    ```bash
    aws iam create-policy \
        --policy-name AWSLoadBalancerControllerIAMPolicy \
        --policy-document file://iam-policy.json
    ```
3.  **Note:** In a real setup, you would create an IAM role and attach this policy. For this beginner guide, ensure your **EKS Node Role** has this policy attached (via AWS Console > IAM > Roles).

### 12. Install Ingress Controller
```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=medfarm-prod-eks \
  --set serviceAccount.create=true \
  --set serviceAccount.annotations."eks\.amazonaws\.io/role-arn"=arn:aws:iam::<YOUR_ACCOUNT_ID>:role/AWSLoadBalancerControllerIAMRole
```
*Note: If you don't have the IAM Role ARN created in Step 11, consult the AWS EKS docs for creating the service account with IRSA.*

### 13. Update Ingress YAML
1.  Open `k8s/ingress/ingress.yaml`.
2.  Find the line `alb.ingress.kubernetes.io/certificate-arn`.
3.  Paste the ARN you copied in **Phase 4, Step 9**.

### 14. Apply All Manifests
*The moment of truth.*
```bash
kubectl apply -f k8s/namespace/
kubectl apply -f k8s/configmaps/
kubectl apply -f k8s/secrets/
kubectl apply -f k8s/services/
kubectl apply -f k8s/deployments/
kubectl apply -f k8s/hpa/
kubectl apply -f k8s/ingress/
```

### 15. Final Verification
```bash
# Check Pods
kubectl get pods -n med-erp
# Expected: All pods should be 'Running'

# Check Ingress
kubectl get ingress -n med-erp
# Expected: You will see an ADDRESS (URL)

# Test the Backend
curl http://<LOAD-BALANCER-ADDRESS>/api/v1/users/health
```

---

## 🛠️ Common Troubleshooting

*   **Error:** `ImagePullBackOff` on pods.
    *   **Fix:** Check if the image URL in `deployment.yaml` matches the ECR repo exactly. Check if you ran `docker push`.
*   **Error:** `Connection Refused` connecting to Database.
    *   **Fix:** Check if the Secret `app-secrets` was created. Run `kubectl describe secret app-secrets -n med-erp` to see if the password matches MongoDB Atlas.
*   **Error:** `502 Bad Gateway` on Ingress.
    *   **Fix:** This is usually the Load Balancer waiting for the Target Groups. Wait 2-3 minutes. If it persists, check the **AWS Console > EC2 > Load Balancers > Target Groups** to see if the Instance Health is healthy.







===
___


# 🚀 Final Phase: Frontend to Backend Integration & SSL Setup

**Objective:** Connect the Frontend (React) and Backend (Spring Boot) using custom domains and secure connections (HTTPS), resolving the IAM permission issues from the previous session.

---

## 🛑️ Phase 1: Pre-Requisites Check
Before starting, ensure the following are ready:
1.  **Backend Check:**
    *   **EKS Cluster:** `medfarm-cluster` (Region: `ap-southeast-2`) is running.
    *   **Pods:** Run `kubectl get pods -n med-erp` to ensure User, Product, and Order pods are `Running`.
    *   **Database:** MongoDB Atlas cluster `edublitz-cluster` is active with databases: `users-db`, `products-db`, `orders-db`.
2.  **Docker Images:** Images (`user-app`, `product-app`, `order-app`) are pushed to ECR.
3.  **Route 53:** A Hosted Zone already exists for your domain.

---

## 🔒 Phase 2: Fixing the IAM Permission Issue (The "Missing Part")
**Problem:** In the previous session, the AWS Load Balancer Controller failed to create the Ingress. The error was `Invalid identity token` / `Failed to refresh cached credentials`.

**Root Cause:** The EKS Node Group did not have the necessary IAM policy to create AWS Load Balancers.

### Step 2.1: Update Node Role Policy
1.  Go to **AWS Console** > **EC2** > **Node Groups** > Select `medfarm-nodegroup`.
2.  Click the **IAM Role** link (this takes you to the IAM console).
3.  Click **Add permissions** > **Attach policies**.
4.  Search for: `AmazonEKSLoadBalancingPolicy`.
5.  Select it and click **Add permissions**.

*Result:* Now, when you apply the Ingress YAML, the AWS Load Balancer will successfully create because the nodes have permission to talk to AWS.

---

## 🌐 Phase 3: SSL Certificates & Region Rules (Crucial Step)
This is the most important concept to master. **CloudFront and EKS use SSL from different regions.**

### Rule #1: Frontend (S3 + CloudFront)
*   **SSL Requirement:** For CloudFront, the SSL Certificate **MUST** be created in the **North Virginia (`us-east-1`) region.
*   *Reason:* CloudFront is a global service, but it only trusts certificates from `us-east-1`.

### Rule #2: Backend (EKS + Ingress)
*   **SSL Requirement:** For the Load Balancer (Ingress) used by EKS, the SSL Certificate **MUST** be in the **Same Region as the Cluster** (`ap-sou-east-2`).

---

## 📂 Phase 4: Setup Backend SSL (Ingress Certificate)

We need a certificate for the API subdomain (e.g., `api.edublitz-b2b-erp.online`).

1.  **Switch Region:** In the AWS Console, change the top right region to **Asia Pacific (Sydney)** to match your EKS region.
2.  Go to **Certificate Manager (ACM)** > **Request a public certificate**.
3.  **Domain Name:** `api.edublitz-b2b-erp.online` (Use a subdomain if your main domain is locked or pending).
4.  **Validation Method:** **DNS Validation**.
5.  **Create Record:** Click **Create Record in Route 53**.
6.  **Update:** Wait 2-5 minutes for the status to change from `Pending validation` to **Issued**.
7.  **Copy the ARN:** Save the Amazon Resource Name (ARN). It looks like:
    `arn:aws:acm:ap-southeast-2:123456789:certificate/xxxx-xxxx-xxxx`

---

## ☁️ Phase 5: Setup CloudFront (Frontend Deployment)
We will host the React frontend on S3 and expose it via CloudFront using the SSL Certificate created in Phase 4.

### Step 5.1: Create CloudFront Distribution
1.  **Switch Region:** **Crucial:** Change region back to **North Virginia (`us-east-1`) because CloudFront requires this region for SSL integration.
2.  Go to **CloudFront** > **Distributions** > **Create distribution**.
3.  **Origin Settings:**
    *   **Origin Domain:** Select your **S3 Bucket** (e.g., `medpharm-frontend-b22`).
    *   **Viewer Protocol Policy:** Redirect HTTP to HTTPS.
4. **Settings:**
    *   **Alternate Domain Names:** Add your frontend domain (e.g., `edublitz-b2b-erp.online`).
    *   **Custom SSL Certificate:** Select the certificate you created in **Phase 4** (North Virginia region).
5.  **Create Distribution:**
    *   Wait 20-30 minutes for the status to become **Deployed**.

*Note: If your main domain (`ajublitz.com`) is already taken or has issues, use a subdomain like `erp.ajublitz.com` or `edublitz-b2b-erp.online`.*

---

## 🌍 Phase 6: Setup Backend Ingress (Load Balancer Integration)
We will expose the Backend APIs via the Load Balancer using the Backend SSL Certificate.

### Step 6.1: Update `ingress.yaml`
Open your `k8s/ingress/ingress.yaml`. You must update two things to connect the domain and SSL.

1.  **Add Domain:**
    ```yaml
    spec:
      rules:
        - host: api.edublitz-b2b-erp.online
    ```
2.  **Add Annotations for Security:**
    Find the `annotations` section and add these lines:
    ```yaml
    annotations:
      alb.ingress.kubernetes.io/certificate-arn: "arn:aws:acm:ap-southeast-2:123456789:certificate/xxxx-xxxx-xxxx" # PASTE THE ARN FROM PHASE 4 HERE
      alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
      alb.ingress.kubernetes.io/ssl-redirect: '443'  # Force HTTPS
    ```
3. **Update Security Group (Advanced):**
    *   In the AWS Console, create a new Security Group (e.g., `medfarm-sg`).
    *   Inbound rules: Allow **HTTPS (Port 443)** traffic from `0.0.0.0/0` (Anywhere) to access the API.

### Step 6.2: Update Security Group in Ingress
You can do this in the YAML, but for beginners, it's easier in the AWS Console.
1.  Go to **EC2** > **Load Balancers**.
2.  Find your **Application Load Balancer** (Auto-created by the Ingress).
3.  Click the **Security** tab.
4.  Click **Edit** > **Edit Security Groups**.
5. Add the `medfarm-sg` created in Step 6.1.
6.  Click **Save changes**.

---

## 🖥️ Phase 7: Apply Changes & Verify

### Step 7.1: Apply Ingress YAML
Run the command:
```bash
kubectl apply -f k8s/ingress/ingress.yaml -n med-erp
```

### Step 7.2: Verify the Load Balancer
1.  Check Ingress status:
    ```bash
    kubectl get ingress -n med-erp
    ```
2.  Look at the **ADDRESS** field. You will see a long URL like:
    `medfarm-ingress-xxxxxxxxx.elb.amazonaws.com`.

---

## 🌐 Phase 8: Route 53 (Connecting Domain to Services)
We need to map your custom domain to the Load Balancer/CloudFront.

### 8.1: Frontend Route (S3 + CloudFront)
1.  Go to **Route 53** > **Hosted Zones**.
2.  Select your Hosted Zone.
3.  **Create Record:**
    *   **Record Type:** **A Record**.
    *   **Record name:** `edublitz-b2b-erp.online`.
    *   **Value:** `s3-website-website-xxxxx.cloudfront.net` (Copy from your CloudFront Distribution Settings).
4.  **Result:** Opening `https://edublitz-b2b-erp.online` should now show your React application.

### 8.2: Backend Route (API)
1.  In the **Same Hosted Zone**, **Create Record**:
    *   **Record Type:** **CNAME** (Alias).
    *   **Record name:** `api` (for `api.edublitz-b2b-erp.online`).
    *   **Value:** The **ADDRESS** of the Load Balancer (copied in Phase 7.2).
2.  **Result:** Opening `https://api.edublitz-b2b-erp.online` should hit the Backend API.

---

## ✅ Phase 9: Final Verification

### 1. Frontend Test
1.  Open your browser.
2.  Go to: `https://edublitz-b2b-erp.online`.
3. **Expected:** You should see the Login Page.

### 2. Backend API Test
1.  Go to: `https://api.edublitz-b2b-api.online/actuator/health` (or `/actuator/health` depending on your context path).
2.  **Expected Output:** `{"status": "UP"}`.
3. **Expected Output:** `{"status": "UP"}`.

### 3. Login Functionality (Full Integration)
1.  Log in as an Admin (User: `admin` / Password: `admin`).
2.  Create an Organization.
3.  Create a Product (e.g., "Paracetamol").
4.  Place an Order.
5. **Check:** Verify if the order reflects in the MongoDB Atlas dashboard.

---

## 🛠️ Troubleshooting Common Errors

### Error 1: "You do not have permission to access this certificate"
*   **Cause:** You are trying to use a certificate created in `ap-southeast-2` for CloudFront, OR you don't have the **AWSLoadBalancingController** policy in the Node Role.
*   **Fix:** Ensure you added the policy as shown in **Phase 2**. If CloudFront is failing, ensure the certificate is in `us-east-1`.

### Error 2: "502 Bad Gateway"
*   **Cause:** The Security Group attached to the Load Balancer is not allowing port 443 traffic.
*   **Fix:** Follow **Phase 6.2** to ensure your Security Group allows traffic on port 443.

### Error 3: "Cross-Origin Read Blocking (CORS)"
*   **Cause:** The backend (Spring Boot) is blocking requests from your Frontend domain.
*   **Fix:** Developers must add `@CrossOrigin` annotations in the Java code or update the Ingress annotations to handle CORS.

### Error 4: Frontend Loads but API fails
*   **Check:** Did you update the `ingress.yaml` with the new ARN? Did you update the Security Group?
*   **Check:** Did you wait 10-15 minutes for Route 53 to propagate?

---

## 🎓 Summary of Steps
1.  **Fix IAM:** Add `AmazonEKSLoadBalancingPolicy` to the Node Role.
2.  **Frontend SSL:** Create Certificate in `us-east-1`, apply to CloudFront.
3. **Backend SSL:** Create Certificate in `ap-southeast-2`, update `ingress.yaml` with the Backend ARN.
4. **DNS Mapping:** Update Route 53 records to point to the CloudFront (Frontend) and Load Balancer (Backend).
5. **Final Check:** Test the full user flow from Frontend to Backend.