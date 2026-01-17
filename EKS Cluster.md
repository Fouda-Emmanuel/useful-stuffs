# Create Secured Kubernetes Cluster on AWS EKS Using AWS Console

In this guide, you’ll learn how to set up a **secure Kubernetes cluster** on AWS EKS using the AWS Management Console. We will configure a **bastion host** in a public subnet, deploy the **EKS cluster and node groups in private subnets**, and deploy a **sample application** to verify everything is working.

---

## **Prerequisites**

Before you start, ensure you have:

* An **AWS account** with sufficient permissions (EC2, VPC, IAM, EKS).
* **AWS CLI** installed (optional but useful).
* **kubectl** installed on your local machine.
* Basic understanding of networking and EC2 instances.

---

## **Step 1: Create a VPC with Public and Private Subnets**

A VPC is required to host your cluster. We will create **public and private subnets** for security and network isolation.

1. Open the **VPC Console** → **Create VPC**.
2. Configure the following:

   * **VPC CIDR Block**: `10.0.0.0/16`
   * **Subnets**:

     * **Public Subnets** (for bastion host): e.g., `10.0.1.0/24`, `10.0.2.0/24`
     * **Private Subnets** (for worker nodes): e.g., `10.0.3.0/24`, `10.0.4.0/24`
   * **Internet Gateway (IGW)**: Attach it to the VPC.
   * **Route Tables**:

     * Public subnet → route `0.0.0.0/0` via IGW
     * Private subnet → no direct internet route (optional: use NAT for outbound internet)

> **Expected outcome:** You now have a VPC with public and private subnets across multiple availability zones for redundancy.

> *Tip:* AWS provides EKS VPC templates that automate subnet creation.

---

## **Step 2: Create a Bastion Host in the Public Subnet**

A **bastion host** is a secure EC2 instance used to access private nodes. This ensures your cluster is not publicly accessible.

1. Open **EC2 Console** → **Launch Instance**.
2. Choose an instance type (e.g., **t2.micro** for free tier).
3. Select the **public subnet** in the **Network** section.
4. Create or use an existing **key pair** for SSH.
5. Configure **Security Group**:

   * Allow **SSH (port 22)** from your **local IP** only.
6. Launch the instance.
7. (Optional) Add **user data** to install AWS CLI and kubectl:

```bash
#!/bin/bash
apt-get update -y
apt-get install -y unzip curl
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
./aws/install
curl -LO "https://dl.k8s.io/release/v1.31.0/bin/linux/amd64/kubectl"
install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
aws --version
kubectl version --client
```

> **Expected outcome:** You now have a bastion host that can SSH into private nodes.

> *Tip:* Always restrict SSH access to your IP to improve security.

---

## **Step 3: Create IAM Roles for Cluster and Nodes**

### **3.1 Cluster IAM Role**

1. Open **IAM Console** → **Create Role** → **EKS** → **EKS Cluster**.
2. Attach policy: `AmazonEKSClusterPolicy`
3. Name it `eks-cluster-role` → Create role.

> **Purpose:** This role allows AWS to manage your EKS control plane.

### **3.2 Worker Node IAM Role**

1. Open **IAM Console** → **Create Role** → **EC2**.
2. Attach policies:

   * `AmazonEKSWorkerNodePolicy`
   * `AmazonEKS_CNI_Policy`
   * `AmazonEC2ContainerRegistryReadOnly`
3. Name it `eks-node-role` → Create role.

> **Purpose:** This role allows worker nodes to join the cluster and pull images from ECR.

---

## **Step 4: Create an EKS Cluster**

1. Open **Amazon EKS Console** → **Create Cluster**.
2. Configure:

   * **Cluster Name**: e.g., `my-secure-cluster`
   * **Kubernetes Version**: latest stable version
   * **Cluster Role**: select `eks-cluster-role`
   * **VPC and Subnets**: select your **private subnets**
   * **Security Group**: default or custom allowing node communication
3. Click **Create** and wait (~10–15 minutes).

> **Expected outcome:** EKS control plane is provisioned. The cluster will be in **private subnets** and **not accessible publicly**.

---

## **Step 5: Create a Node Group in Private Subnets**

1. In **EKS Console**, select your cluster → **Compute** → **Add Node Group**.
2. Configure:

   * **Node Group Name**: e.g., `private-node-group`
   * **IAM Role**: select `eks-node-role`
   * **Subnets**: select **private subnets**
   * **Instance Type**: `t3.medium`
   * **Scaling Configuration**: min=1, desired=2, max=3
3. Click **Create** and wait for nodes to register.

> **Expected outcome:** Nodes will join the cluster. You can verify later with `kubectl get nodes`.

---

## **Step 6: Configure Security Groups for EKS Cluster**

1. In your cluster → **Networking** → **Add Security Group**.
2. Add rules:

   * **Port 443** → allow access from **bastion host SG** (for API server)
   * **Custom ports** if needed (e.g., 33080)
3. Ensure **node security groups allow traffic from cluster security group**.

> **Purpose:** Proper SG configuration ensures secure communication between control plane, worker nodes, and bastion host.

---

## **Step 7: Configure kubectl to Access the Cluster**

### **7.1 From your local machine or bastion host**

1. Install `kubectl` (if not done yet):

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

2. Install AWS CLI (if not done yet):

```bash
sudo apt update -y
sudo apt install unzip curl -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

3. Configure AWS CLI:

```bash
aws configure
# Enter AWS Access Key, Secret, region, output format
```

4. Update kubeconfig to connect to EKS:

```bash
aws eks --region <your-region> update-kubeconfig --name <cluster-name>
```

5. Verify connection:

```bash
kubectl get nodes
kubectl cluster-info
```

> **Expected outcome:** You should see your worker nodes and cluster info.

---

## **Step 8: Create a Kubernetes Secret for ECR (Optional)**

If you plan to pull private Docker images from AWS ECR:

```bash
kubectl create secret docker-registry ecr-credentials \
  --docker-server=<ECR-URL> \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password) \
  --docker-email=<your-email>
kubectl get secret
```

> **Note:** Replace `<ECR-URL>` and `<your-email>` with actual values.

---

## **Step 9: Deploy a Sample Application**

1. Create a file named `sample-app.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: goapp-deployment
spec:
  selector:
    matchLabels:
      app: goapp
  replicas: 2
  template:
    metadata:
      labels:
        app: goapp
    spec:
      containers:
      - name: goapp
        image: owahed1/go-lang-app:0.0.2
        imagePullPolicy: Always
        ports:
        - containerPort: 8000
---
apiVersion: v1
kind: Service
metadata:
  name: goapp-service
spec:
  selector:
    app: goapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000
  type: LoadBalancer
```

2. Deploy the application:

```bash
kubectl apply -f sample-app.yaml
```

3. Verify:

```bash
kubectl get pods
kubectl get svc
```

> **Expected outcome:**

* 2 pods running for the deployment
* A LoadBalancer service created with an external IP to access your app.

---

## **Step 10: Access the Application**

* Use the **LoadBalancer external IP** from `kubectl get svc` to open the app in a browser.

> **Note:** Private nodes will only be accessible via the LoadBalancer or bastion host.

---

## **Step 11: Clean Up Resources**

To avoid unnecessary costs:

1. Delete **node groups** first.
2. Delete **EKS cluster**.
3. Delete **NAT Gateway** (if used).
4. Delete **VPC** and associated subnets.
5. Delete **Load Balancers** and **Elastic IPs**.

> **Tip:** Always check for leftover EC2, EBS, or ELB resources to avoid billing surprises.

---

✅ **Congratulations!** You now have a secure, private EKS cluster, accessed via a bastion host, with a deployed sample application.

---
