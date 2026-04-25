# Part 1: Create EKS Cluster with eksctl

**What you'll create:**  
- 1 EKS Cluster  
- 3 Worker Nodes (EC2 node group)  
- 1 Fargate Profile  

---

## Step 1 — Create the EKS Cluster with Node Group

(Change region if needed. If you want to use a config file, see: https://docs.aws.amazon.com/eks/latest/eksctl/creating-and-managing-clusters.html)

```bash
eksctl create cluster \
  --name demo-cluster \
  --version 1.29 \
  --region eu-north-1 \
  --nodegroup-name demo-nodes \
  --node-type t3.small \
  --nodes 3 \
  --nodes-min 1 \
  --nodes-max 3
```

⏳ This takes 15–20 minutes. `eksctl` creates the VPC, subnets, IAM roles, and CloudFormation stacks automatically.

---

## Step 2 — Verify the Cluster & Nodes

```bash
eksctl get cluster --name demo-cluster --region eu-north-1
kubectl get nodes
```

---

## Step 3 — Create the Fargate Profile

```bash
kubectl create namespace fargate-app

eksctl create fargateprofile \
  --cluster demo-cluster \
  --name demo-fargate-profile \
  --namespace fargate-app \
  --region eu-north-1
```

---

## Step 4 — Verify the Fargate Profile

```bash
eksctl get fargateprofile \
  --cluster demo-cluster \
  --region eu-north-1
```

---

## Step 5 — Verify Everything Together

```bash
kubectl get nodes
kubectl cluster-info
```

---

# Part 2: Deploy MySQL and phpMyAdmin on EKS

## Step 1 — Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

---

## Step 2 — Add Bitnami Repo

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

---

## Step 3 — Create Namespace

```bash
kubectl create namespace mysql-app
```

---

## Step 4 — Config (or Clone) MySQL Values File

---

## Step 5 — Deploy MySQL

```bash
helm install mysql bitnami/mysql \
  --namespace mysql-app \
  --values mysql-values.yaml
```

---

## Step 6 — Config (or clone) phpMyAdmin Values File

---

## Step 7 — Deploy phpMyAdmin

```bash
helm install phpmyadmin bitnami/phpmyadmin \
  --namespace mysql-app \
  --values phpmyadmin-values.yaml
```

---

# Part 3: Deploy Java Application on Fargate via YAML file

## Step 1 — Config java-app-deployment.yaml

⚠️ Replace <your-dockerhub-or-ecr-image> with your actual image. Fargate requires a valid, accessible container image.

