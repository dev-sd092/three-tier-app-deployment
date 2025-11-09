# 🚀 Three-Tier Application Deployment on AWS EKS

## 🧠 Project Overview
This project demonstrates a **complete CI/CD-ready three-tier architecture** (Frontend, Backend, Database) deployed on **Amazon EKS (Elastic Kubernetes Service)** using **Docker, AWS ECR, and AWS Load Balancer Controller**.

The goal was to containerize each application layer, push Docker images to **Amazon ECR**, orchestrate deployment on **EKS**, and expose the running application securely using an **Application Load Balancer (ALB)**.

---

## 🏗️ Architecture Overview

### Tech Stack
| Layer | Technology | Description |
|--------|-------------|-------------|
| **Frontend** | React (Node.js) | User interface served on port `3000`. |
| **Backend** | Node.js + Express | API layer connecting to MongoDB. |
| **Database** | MongoDB | Persistent storage for task entries. |
| **Containerization** | Docker | Each tier containerized and pushed to ECR. |
| **Orchestration** | Amazon EKS | Manages pods, services, and deployments. |
| **Ingress** | AWS Load Balancer Controller | Provides external access to EKS workloads. |
| **CI/CD Ready** | AWS CLI + ECR + kubectl + eksctl + helm | Manual to automated provisioning flow. |

---

## ⚙️ Step-by-Step Implementation

### 1️⃣ EC2 Setup
- Created a **t2.micro** EC2 instance (Ubuntu) with **30 GB** storage for the deployment environment.
- Installed required tools:
  ```bash
  sudo apt update
  sudo apt install docker.io unzip -y
  ```
- Then give permission to your user to run docker commands:
  ```bash
  sudo grpadd docker
  sudo usermod -aG docker $USER
  newgrp docker
  ```

---

### 2️⃣ Frontend & Backend Dockerization
- Wrote Dockerfiles for frontend and backend applications.
- Built and verified containers locally:
```bash
  docker build -t frontend:latest .
  docker run -d -p 3000:3000 frontend
```
- Opened port 3000 in EC2 Security Group for browser access.
Repeated the same for the backend (different port, e.g., 8080).

---

### 3️⃣ Push Docker Images to ECR

- Installed AWS CLI:
  ```bash
  curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
  unzip awscliv2.zip
  sudo ./aws/install
  aws configure
  ```
- Created private ECR repositories (frontend & backend).
- Followed AWS push commands:
  ```bash
  aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <AWS_ACCOUNT_ID>.dkr.ecr.us-west-1.amazonaws.com
  docker tag frontend:latest <ECR_URI>/frontend:latest
  docker push <ECR_URI>/frontend:latest
  ```
- Repeated for backend.

---

### 4️⃣ Create and Configure EKS Cluster

- Installed kubectl and eksctl:
  ### kubectl
  ```bash
  curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
  chmod +x ./kubectl && sudo mv ./kubectl /usr/local/bin
  kubectl version --short --client
  ```
  
  ### eksctl
  ```bash
  curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
  sudo mv /tmp/eksctl /usr/local/bin
  eksctl version
  ```
- Then created the EKS cluster:
  ```bash
  eksctl create cluster --name three-tier-cluster --region us-east-1 --node-type t2.medium --nodes-min 2 --nodes-max 2
  aws eks update-kubeconfig --region us-east-1 --name three-tier-cluster
  kubectl get nodes
  ```
---

### 5️⃣ Deploy Application to EKS

- Created a dedicated namespace:
  ```bash
  kubectl create ns three-tier
  ```
  
- Applied manifests in order:
  database.yaml → MongoDB
  backend.yaml → API
  frontend.yaml → Web UI

- Verified pods and services:
  ```bash
  kubectl get all -n three-tier
  ```
  
---

### 6️⃣ Configure AWS Load Balancer Controller

- Installed IAM policies and Helm chart to enable ALB ingress for external access:
  ```bash
  curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
  aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json

  eksctl utils associate-iam-oidc-provider --region us-west-1 --cluster=three-tier-cluster --approve
  eksctl create iamserviceaccount \
    --cluster=three-tier-cluster \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --role-name AmazonEKSLoadBalancerControllerRole \
    --attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
    --approve --region=us-west-1
   ```

---

- Install Helm and deploy controller:
  ```bash
  sudo snap install helm --classic
  helm repo add eks https://aws.github.io/eks-charts
  helm repo update
  helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    -n kube-system \
    --set clusterName=three-tier-cluster \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller
  ```
---

### 7️⃣ Deploy Ingress
  kubectl apply -f ingress.yaml
  kubectl get ingress mainlb -n three-tier
- The output shows the ALB DNS name for accessing the app.

---

### 8️⃣ Validate Database Entries
- Checked live data inside MongoDB running in the cluster:
  ```bash
  kubectl exec -it mongodb-<pod> -n three-tier -- /bin/sh
  show dbs
  use todo
  db.tasks.find()
  ```
---

### 🧩 Common Issue and Fix

## Error:
- AccessDenied: elasticloadbalancing:DescribeListenerAttributes

## Root Cause:
- IAM role attached to the AWS Load Balancer Controller was missing one or more updated ELBv2 permissions.

## Fix:
  ```bash
 curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
 aws iam put-role-policy \
   --role-name AmazonEKSLoadBalancerControllerRole \
   --policy-name AWSLoadBalancerControllerAdditionalPolicy \
   --policy-document file://iam_policy.json
 kubectl -n kube-system rollout restart deployment aws-load-balancer-controller
```
## ✅ Result: ALB successfully created, application publicly accessible.

---

### 🧹 Cleanup
- eksctl delete cluster --name three-tier-cluster --region us-west-1
- # Then delete ALB, EC2, and Security Groups manually

---

### 📸 Screenshots
|Section|	ScreenShot |
|-------|------------|
|**🖥️ Application** |	<img width="1902" height="1028" alt="Screenshot 2025-11-09 041718" src="https://github.com/user-attachments/assets/5bb1c5c1-6f79-42d2-a6ba-64034772c71a" /> |
|**☁️ Load Balancer** | <img width="1616" height="217" alt="Screenshot 2025-11-09 041733" src="https://github.com/user-attachments/assets/a73dd65d-7c20-4783-bef5-33fe7d86df99" /> |	
|**🧩 EKS Cluster**	| <img width="1561" height="209" alt="Screenshot 2025-11-09 041801" src="https://github.com/user-attachments/assets/dd1c4187-3d19-4991-911c-c793c062b94b" /> |
|**🐳 ECR**	| <img width="1566" height="226" alt="Screenshot 2025-11-09 041831" src="https://github.com/user-attachments/assets/f8123dfb-86a8-4558-9bd6-3f8cd83d03c1" /> |
|**🧮 EC2**	| <img width="1608" height="250" alt="Screenshot 2025-11-09 041743" src="https://github.com/user-attachments/assets/4425aabc-6e31-494c-b4ae-9bdeab9c4aab" /> |
|**📂 Database** |	<img width="1667" height="262" alt="Screenshot 2025-11-09 042617" src="https://github.com/user-attachments/assets/ede178f0-11a5-4ac9-9670-701ca6ad91ae" /> |

---

###🧾 Key Learnings
- End-to-end Kubernetes deployment workflow on AWS.
- Integration of ECR → EKS → ALB for production-grade setup.
- Role-based IAM configuration for EKS controllers.
- Debugging permission-based issues in AWS IAM and Load Balancer.
- Practical hands-on exposure to container orchestration and cloud-native networking.

---

### 🔗 Connect With Me
- 📘 [LinkedIn Profile](https://www.linkedin.com/in/sanket-desai/)

---

### 🏁 Outcome
- ✅ Successfully deployed a three-tier application using AWS-managed Kubernetes infrastructure with complete CI/CD readiness and cloud-native scalability.

---
