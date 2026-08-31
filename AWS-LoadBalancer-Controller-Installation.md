# Installing AWS Load Balancer Controller Without Helm Repository Access

## Overview

This guide covers the installation of the AWS Load Balancer Controller on Amazon EKS when the official Helm repository (`https://aws.github.io/eks-charts`) is unreachable due to network restrictions.

**Alternative methods covered:**
1. **Local Helm Chart Installation** (Recommended)
2. **Direct Chart Download**
3. **Git Clone with Sparse Checkout**

---

## Prerequisites

Before installation, ensure you have:

| Requirement | Verification Command |
|-------------|---------------------|
| EKS Cluster | `aws eks describe-cluster --name YOUR_CLUSTER_NAME --region YOUR_REGION` |
| AWS CLI | `aws --version` |
| kubectl | `kubectl version` |
| Helm v3 | `helm version` |
| IAM Permissions | Ability to create IAM roles/policies |

---

## Step 1: IAM Setup (IRSA)

### 1.1 Create IAM Policy

First, create the IAM policy that defines the permissions the controller needs.

```bash
# Download the policy document
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.13.3/docs/install/iam_policy.json

# Create the policy
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

**Expected Output:**
```json
{
    "Policy": {
        "Arn": "arn:aws:iam::YOUR_AWS_ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy"
    }
}
```

### 1.2 Associate IAM OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster YOUR_CLUSTER_NAME \
  --region YOUR_AWS_REGION \
  --approve
```

### 1.3 Create ServiceAccount with IAM Role

```bash
eksctl create iamserviceaccount \
  --cluster=YOUR_CLUSTER_NAME \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::YOUR_AWS_ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --region=YOUR_AWS_REGION \
  --approve
```

**Verification:**
```bash
kubectl get sa aws-load-balancer-controller -n kube-system -o yaml
```

Expected output:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::YOUR_AWS_ACCOUNT_ID:role/eksctl-YOUR_CLUSTER_NAME-addon-iamservicea-Role1-XXXXXXXX
```

---

## Step 2: Install the Controller

### Method 1: Download Chart Package (Recommended)

**Download the official chart package:**

```bash
# For version 1.14.0 (Latest stable)
wget https://github.com/aws/eks-charts/raw/master/stable/aws-load-balancer-controller/aws-load-balancer-controller-1.14.0.tgz

# Or use curl if wget is not available
curl -LO https://github.com/aws/eks-charts/raw/master/stable/aws-load-balancer-controller/aws-load-balancer-controller-1.14.0.tgz
```

**Install using the downloaded package:**

```bash
helm install aws-load-balancer-controller \
  aws-load-balancer-controller-1.14.0.tgz \
  -n kube-system \
  --set clusterName=YOUR_CLUSTER_NAME \
  --set region=YOUR_AWS_REGION \
  --set vpcId=YOUR_VPC_ID \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

**Example with actual values:**
```bash
helm install aws-load-balancer-controller \
  aws-load-balancer-controller-1.14.0.tgz \
  -n kube-system \
  --set clusterName=demo-ingress-cluster \
  --set region=us-east-1 \
  --set vpcId=vpc-02c882da4a348b133 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

### Method 2: Clone Chart Locally

**Clone only the chart directory:**

```bash
# Clone with sparse checkout
git clone --depth=1 --filter=blob:none --sparse https://github.com/aws/eks-charts.git
cd eks-charts
git sparse-checkout set stable/aws-load-balancer-controller

# Navigate to the chart
cd stable/aws-load-balancer-controller
```

**Install using the local chart:**
```bash
helm install aws-load-balancer-controller \
  . \
  -n kube-system \
  --set clusterName=YOUR_CLUSTER_NAME \
  --set region=YOUR_AWS_REGION \
  --set vpcId=YOUR_VPC_ID \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

### Method 3: Download Chart with curl

**Get the chart as a single file:**
```bash
# Download the entire chart as a tarball
curl -L https://github.com/aws/eks-charts/archive/refs/heads/master.tar.gz -o eks-charts.tar.gz

# Extract the specific chart
tar -xzf eks-charts.tar.gz eks-charts-master/stable/aws-load-balancer-controller

# Move to the chart directory
mv eks-charts-master/stable/aws-load-balancer-controller .
```

---

## Step 3: Critical Configuration Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `clusterName` | Your EKS cluster name | `demo-ingress-cluster` |
| `region` | AWS region | `us-east-1` |
| `vpcId` | VPC ID for your cluster | `vpc-02c882da4a348b133` |
| `serviceAccount.create` | Set to `false` to use existing SA | `false` |
| `serviceAccount.name` | Existing ServiceAccount name | `aws-load-balancer-controller` |

### Why These Parameters Matter

- **`region` & `vpcId`**: The controller needs these to know which AWS environment it's operating in. Without explicit values, it tries to discover them through EC2 Instance Metadata Service (IMDS), which may fail depending on your network configuration.

- **`serviceAccount.create=false`**: Prevents Helm from creating a new ServiceAccount, ensuring it uses the one we created with IRSA.

---

## Step 4: Verify Installation

### 4.1 Check Helm Release

```bash
helm list -n kube-system
```

Expected output:
```
NAME                          	NAMESPACE  	REVISION	UPDATED                             	STATUS  	CHART                                       	APP VERSION
aws-load-balancer-controller  	kube-system	1       	2026-08-30 23:49:12.123456 +0000 UTC 	deployed	aws-load-balancer-controller-1.14.0       	v2.14.1
```

### 4.2 Check Deployment

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

Expected output:
```
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   2/2     2            2           5m
```

### 4.3 Check Pods

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```

Expected output:
```
NAME                                            READY   STATUS    RESTARTS   AGE
aws-load-balancer-controller-778948cb66-5cxz9   1/1     Running   0          40s
aws-load-balancer-controller-778948cb66-lcj9w   1/1     Running   0          40s
```

### 4.4 Check Controller Logs

```bash
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller --tail=30
```

Look for successful initialization messages, not errors like:
- `failed to get VPC ID`
- `failed to fetch VPC ID from instance metadata`
- `context deadline exceeded`

---

## Step 5: Troubleshooting Common Issues

### Issue 1: Pods CrashLoopBackOff

**Symptom:**
```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```
Shows `CrashLoopBackOff` status.

**Solution:** Check logs for error messages:

```bash
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller --tail=50
```

**Common Causes:**

| Error Message | Solution |
|---------------|----------|
| `failed to get VPC ID` | Set `vpcId` explicitly using `--set vpcId=YOUR_VPC_ID` |
| `failed to get region` | Set `region` explicitly using `--set region=YOUR_AWS_REGION` |
| `AccessDenied` | Verify IAM role permissions and trust policy |
| `AssumeRoleWithWebIdentity` | Check OIDC provider association |

### Issue 2: Helm Installation Fails

**Symptom:** `helm install` returns errors.

**Solutions:**

1. **Clear Helm cache:**
```bash
rm -rf ~/.cache/helm/repository
```

2. **Remove existing release:**
```bash
helm uninstall aws-load-balancer-controller -n kube-system
```

3. **Verify chart integrity:**
```bash
helm lint aws-load-balancer-controller-1.14.0.tgz
```

### Issue 3: Ingress Not Creating ALB

**Verify:**
1. Subnet tags are correct:
```bash
aws ec2 describe-subnets --filters "Name=vpc-id,Values=YOUR_VPC_ID" --query 'Subnets[*].[SubnetId,Tags]'
```

2. IngressClass exists:
```bash
kubectl get ingressclass alb
```

3. Controller logs:
```bash
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller --tail=50
```

---

## Step 6: Tag Subnets for ALB Discovery

For the controller to automatically discover subnets, tag public subnets:

```bash
aws ec2 create-tags \
  --resources subnet-YOUR_PUBLIC_SUBNET_1_ID subnet-YOUR_PUBLIC_SUBNET_2_ID \
  --tags Key=kubernetes.io/role/elb,Value=1
```

**Verify tags:**
```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=YOUR_VPC_ID" \
  --query 'Subnets[*].[SubnetId,AvailabilityZone,CidrBlock,Tags[?Key==`Name`].Value|[0],Tags[?Key==`kubernetes.io/role/elb`].Value|[0]]' \
  --output table
```

---

## Step 7: Create an Ingress to Test

**Example Ingress YAML:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /app
            pathType: Prefix
            backend:
              service:
                name: your-app-service
                port:
                  number: 80
```

**Apply the Ingress:**
```bash
kubectl apply -f demo-ingress.yaml
```

**Check ALB Creation:**
```bash
kubectl get ingress
```

Expected output:
```
NAME           CLASS   HOSTS   ADDRESS                                                              PORTS   AGE
demo-ingress   alb     *       k8s-default-demoingr-xxxxxxxxxx-xxxxxxxxxx.elb.amazonaws.com         80      2m
```

---

## Architecture Summary

Your final architecture:

```text
                    INTERNET
                        │
                        ▼
              ┌─────────────────────┐
              │        ALB          │
              │  public subnets     │
              │  SG: allow TCP 80   │
              └──────────┬──────────┘
                        │
                        ▼
              Kubernetes Pods
              in PRIVATE subnets
                        │
                        ▼
              AWS Load Balancer Controller
              (running in EKS)
```

---

## Quick Reference: Essential Commands

```bash
# Get VPC ID
aws eks describe-cluster --name YOUR_CLUSTER_NAME --query 'cluster.resourcesVpcConfig.vpcId' --output text

# Get Region
aws configure get region

# Verify ServiceAccount
kubectl get sa aws-load-balancer-controller -n kube-system -o yaml

# Install Controller
helm install aws-load-balancer-controller \
  aws-load-balancer-controller-1.14.0.tgz \
  -n kube-system \
  --set clusterName=YOUR_CLUSTER_NAME \
  --set region=YOUR_AWS_REGION \
  --set vpcId=YOUR_VPC_ID \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

# Check Status
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
kubectl get deployment -n kube-system aws-load-balancer-controller

# View Logs
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller --tail=30
```

---
