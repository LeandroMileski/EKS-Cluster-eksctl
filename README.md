# Part 1: Create EKS Cluster with eksctl

What you'll create: 1 EKS Cluster, 3 Worker Nodes (EC2 node group), 1 Fargate Profile

Step 1 — Create the EKS Cluster with Node Group. (change Region and if you want to use config file check https://docs.aws.amazon.com/eks/latest/eksctl/creating-and-managing-clusters.html)
 
eksctl create cluster \
  --name demo-cluster \
  --version 1.29 \
  --region eu-north-1 \
  --nodegroup-name demo-nodes \
  --node-type t3.small \
  --nodes 3 \
  --nodes-min 1 \
  --nodes-max 3

⏳ This takes 15–20 minutes. eksctl creates the VPC, subnets, IAM roles, and CloudFormation stacks automatically.


Step 2 — Verify the Cluster & Nodes
bash# Check cluster is active
eksctl get cluster --name demo-cluster --region eu-north-1

Check all 3 nodes are Ready
kubectl get nodes

Step 3 — Create the Fargate Profile
Fargate profiles require a namespace to associate with. First create the namespace, then the profile:
bash# Create a namespace for Fargate workloads
kubectl create namespace fargate-app

Create the Fargate profile
eksctl create fargateprofile \
  --cluster demo-cluster \
  --name demo-fargate-profile \
  --namespace fargate-app \
  --region eu-north-1

⏳ Fargate profile creation takes 2–5 minutes.


Step 4 — Verify the Fargate Profile
eksctl get fargateprofile \
  --cluster demo-cluster \
  --region eu-north-1

Expected output:
NAME                  SELECTOR_NAMESPACE   SELECTOR_LABELS   AGE
demo-fargate-profile  fargate-app          <none>            2m

Step 5 — Verify Everything Together
bash# All nodes (EC2 + Fargate virtual nodes)
kubectl get nodes

Cluster info
kubectl cluster-info

# Part 2: Deploy MySQL and phpMyAdmin on EKS

What you'll deploy:
MySQL database on EC2 nodes via Helm
phpMyAdmin on EC2 nodes via Helm
Both using Bitnami charts with custom values


Step 1 — Install Helm (if not already installed)
bash# Windows (Git Bash)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
Or download directly from https://helm.sh/docs/intro/install/
bash# Verify
helm version

Step 2 — Add Bitnami Helm Repository
bashhelm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

Step 3 — Create a Namespace for the App
bashkubectl create namespace mysql-app

Step 4 — Create MySQL Values File
Create a file called mysql-values.yaml:
yamlglobal:
  security:
    allowInsecureImages: True

image:
  registry: docker.io
  tag: latest
  repository: bitnamilegacy/mysql

auth:
  rootPassword: "rootpassword123"
  database: "mydb"
  username: "myuser"
  password: "mypassword123"

primary:
  nodeSelector: {}   # deploys on EC2 nodes (not Fargate)
  persistence:
    enabled: false
secondary:
  persistence:
    enabled: false

Step 5 — Deploy MySQL via Helm
bash
helm install mysql bitnami/mysql \
  --namespace mysql-app \
  --values mysql-values.yaml

⏳ Wait for MySQL to be running before proceeding:

bashkubectl get pods -n mysql-app -w
Wait until you see:
mysql-0   1/1   Running   0   1m

Step 6 — Create phpMyAdmin Values File
Create a file called phpmyadmin-values.yaml:
yamlglobal:
  security:
    allowInsecureImages: True

image:
  registry: docker.io
  tag: latest
  repository: bitnamilegacy/phpmyadmin

db:
  host: mysql.mysql-app.svc.cluster.local
  port: 3306

service:
  type: LoadBalancer

Step 7 — Deploy phpMyAdmin via Helm
bash
helm install phpmyadmin bitnami/phpmyadmin \
  --namespace mysql-app \
  --values phpmyadmin-values.yaml

Step 8 — Verify Both Deployments
bash# Check all pods are Running
kubectl get pods -n mysql-app

Check services
kubectl get svc -n mysql-app
Expected output:
NAME          TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)
mysql         ClusterIP      10.100.x.x       <none>          3306/TCP
phpmyadmin    LoadBalancer   10.100.x.x       <EXTERNAL-IP>   80:xxxxx/TCP

Step 9 — Access phpMyAdmin
bash# Get the external IP/URL of phpMyAdmin
kubectl get svc phpmyadmin -n mysql-app
Copy the EXTERNAL-IP and open it in your browser:
http://<EXTERNAL-IP>/
Login with:
FieldValueServermysql.mysql-app.svc.cluster.localUsernamemyuserPasswordmypassword123

Step 10 — Verify Pods are on EC2 Nodes (not Fargate)
bash
kubectl get pods -n mysql-app -o wide
The NODE column should show EC2 node names (e.g. ip-192-168-x-x.eu-north-1.compute.internal), not fargate-* nodes.

Summary
ResourceDetailsNamespacemysql-appMySQLBitnami chart, bitnamilegacy/mysql imagephpMyAdminBitnami chart, bitnamilegacy/phpmyadmin imageAccessVia LoadBalancer external IP on port 80NodesEC2 (demo-nodes node group)

Troubleshooting
bash# If pods are stuck in Pending
kubectl describe pod <pod-name> -n mysql-app

# Part 3: Deploy Java Application on Fargate

What you'll deploy

Java application running on Fargate (not EC2 nodes)
3 replicas
Deployed into the fargate-app namespace (which is already linked to the Fargate profile from Exercise 1)


Step 1 — Verify your Fargate Profile is Ready
eksctl get fargateprofile --cluster demo-cluster --region eu-north-1
Make sure fargate-app namespace is listed. If not, create it:
eksctl create fargateprofile \
  --cluster demo-cluster \
  --name demo-fargate-profile \
  --namespace fargate-app \
  --region eu-north-1

Step 2 — Create the Namespace
kubectl create namespace fargate-app

Step 3 — Create the Deployment YAML
Create java-app-deployment.yaml:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app
  namespace: fargate-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: java-app
  template:
    metadata:
      labels:
        app: java-app
    spec:
      containers:
        - name: java-app
          image: <your-dockerhub-or-ecr-image>   # e.g. your-user/java-app:latest
          ports:
            - containerPort: 8080
          env:
            - name: DB_HOST
              value: "mysql.mysql-app.svc.cluster.local"
            - name: DB_PORT
              value: "3306"
            - name: DB_NAME
              value: "mydb"
            - name: DB_USER
              value: "myuser"
            - name: DB_PASSWORD
              value: "mypassword123"
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "500m"
              memory: "1Gi"

⚠️ Replace <your-dockerhub-or-ecr-image> with your actual image. Fargate requires a valid, accessible container image.


Step 4 — Create a Service YAML
Create java-app-service.yaml:
yamlapiVersion: v1a
kind: Service
metadata:
  name: java-app-service
  namespace: fargate-app
spec:
  selector:
    app: java-app
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080

Step 5 — Deploy the Application
kubectl apply -f java-app-deployment.yaml
kubectl apply -f java-app-service.yaml

Step 6 — Verify Pods are Running on Fargate
Watch pods come up (Fargate takes ~1-2 min per pod)
kubectl get pods -n fargate-app -w

Expected output:
NAME                        READY   STATUS    RESTARTS   AGE
java-app-xxxxxxxxx-xxxxx   1/1     Running   0          2m
java-app-xxxxxxxxx-xxxxx   1/1     Running   0          2m
java-app-xxxxxxxxx-xxxxx   1/1     Running   0          2m

Confirm they're on Fargate nodes:
kubectl get pods -n fargate-app -o wide
The NODE column should show fargate- prefixed node names.

Step 7 — Get the External URL
kubectl get svc java-app-service -n fargate-app
Copy the EXTERNAL-IP and open in your browser:
http://<EXTERNAL-IP>/

Step 8 — Verify Replica Count
kubectl get deployment java-app -n fargate-app
Expected:
NAME       READY   UP-TO-DATE   AVAILABLE
java-app   3/3     3            3