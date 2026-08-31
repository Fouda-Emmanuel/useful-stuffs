# Giving Kubernetes Pods Access to AWS Resources

Kubernetes workloads running on Amazon EKS often need permission to access AWS services.

Examples:

* AWS Load Balancer Controller → creates and manages ALBs/NLBs
* ExternalDNS → manages Route 53 records
* EBS CSI Driver → creates and attaches EBS volumes
* Application Pods → access S3, SQS, DynamoDB, Secrets Manager, etc.

A Kubernetes Pod cannot automatically call AWS APIs with permissions.

We need a secure way to answer:

> **Which AWS IAM Role should this Pod use?**

There are two main approaches:

1. **IRSA — IAM Roles for Service Accounts**
2. **EKS Pod Identity**

Both connect a Kubernetes identity to an AWS IAM Role, but they work differently.

---

# 1. The Core Concept

Before looking at the implementation, understand the architecture.

A Pod uses a Kubernetes ServiceAccount:

```text
Pod
 │
 │ uses
 ▼
Kubernetes ServiceAccount
```

The ServiceAccount itself is a Kubernetes identity.

But AWS does not understand Kubernetes ServiceAccounts directly.

We therefore need a mechanism that connects:

```text
Kubernetes Identity
        │
        ▼
AWS IAM Identity
```

Both IRSA and EKS Pod Identity solve this problem.

The final goal is always conceptually:

```text
Pod
 │
 ▼
Kubernetes ServiceAccount
 │
 ▼
IAM Role
 │
 ▼
IAM Policy
 │
 ▼
AWS Resources
```

For example:

```text
AWS Load Balancer Controller Pod
            │
            ▼
aws-load-balancer-controller
ServiceAccount
            │
            ▼
IAM Role
            │
            ▼
AWSLoadBalancerControllerIAMPolicy
            │
            ▼
AWS APIs
 ├── Elastic Load Balancing
 ├── EC2
 ├── Target Groups
 ├── Security Groups
 └── etc.
```

---

# Method 1: IRSA (IAM Roles for Service Accounts)

## What is IRSA?

IRSA allows a Kubernetes ServiceAccount to assume an AWS IAM Role using an OIDC identity provider.

The architecture looks like this:

```text
                    AWS ACCOUNT
                         │
                         │
                  IAM OIDC Provider
                         │
                         │ Trust
                         ▼
                     IAM Role
                         │
                         │ Permissions
                         ▼
                    IAM Policy
                         ▲
                         │
                 Role Annotation
                         │
                         │
                 Kubernetes Cluster
                         │
                         ▼
                 ServiceAccount
                         │
                         ▼
                       Pod
```

More specifically:

```text
Pod
 │
 │ uses
 ▼
ServiceAccount
kube-system/aws-load-balancer-controller
 │
 │ identified through OIDC token
 ▼
EKS OIDC Provider
 │
 │ assumes role using STS
 ▼
IAM Role
 │
 │ has attached
 ▼
IAM Policy
 │
 ▼
AWS API Permissions
```

---

# When to Use IRSA

IRSA is useful when:

* You already use IRSA in your EKS environment.
* You want direct integration between Kubernetes ServiceAccounts and IAM Roles.
* You need compatibility with existing EKS workloads.
* A tool's installation documentation uses IRSA.
* You want granular IAM permissions per workload.

---

# Prerequisites

Before configuring IRSA, you need:

```text
EKS Cluster                    Required
AWS CLI                        Required
kubectl                        Required
eksctl                         Recommended
IAM permissions                Required
```

Check your cluster:

```bash
aws eks describe-cluster \
  --name <CLUSTER_NAME> \
  --region <REGION>
```

Example:

```bash
aws eks describe-cluster \
  --name YOUR_CLUSTER_NAME \
  --region YOUR_AWS_REGION
```

---

# Step 1: Check the EKS OIDC Issuer

Every EKS cluster has an OIDC issuer URL.

Check it with:

```bash
aws eks describe-cluster \
  --name <CLUSTER_NAME> \
  --region <REGION> \
  --query 'cluster.identity.oidc.issuer' \
  --output text
```

Example:

```bash
aws eks describe-cluster \
  --name YOUR_CLUSTER_NAME \
  --region YOUR_AWS_REGION \
  --query 'cluster.identity.oidc.issuer' \
  --output text
```

Example output:

```text
https://oidc.eks.YOUR_AWS_REGION.amazonaws.com/id/YOUR_OIDC_ID
```

Important distinction:

> The EKS cluster having an OIDC issuer URL does NOT automatically mean an IAM OIDC Provider exists in your AWS account.

The cluster exposes an OIDC issuer:

```text
EKS Cluster
     │
     ▼
OIDC Issuer URL
```

But AWS IAM must also trust/register that issuer:

```text
AWS IAM
     │
     ▼
IAM OIDC Provider
```

---

# Step 2: Check Whether an IAM OIDC Provider Already Exists

Run:

```bash
aws iam list-open-id-connect-providers
```

If the result is empty:

```json
{
    "OpenIDConnectProviderList": []
}
```

then the IAM OIDC Provider has not yet been associated.

---

# Step 3: Associate the IAM OIDC Provider

Using `eksctl`:

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster <CLUSTER_NAME> \
  --region <REGION> \
  --approve
```

Example:

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster YOUR_CLUSTER_NAME \
  --region YOUR_AWS_REGION \
  --approve
```

This creates an IAM resource similar to:

```text
IAM OIDC Provider

arn:aws:iam::YOUR_AWS_ACCOUNT_ID:oidc-provider/
oidc.eks.YOUR_AWS_REGION.amazonaws.com/id/YOUR_OIDC_ID
```

Verify:

```bash
aws iam list-open-id-connect-providers
```

Example output:

```json
{
    "OpenIDConnectProviderList": [
        {
            "Arn": "arn:aws:iam::YOUR_AWS_ACCOUNT_ID:oidc-provider/oidc.eks.YOUR_AWS_REGION.amazonaws.com/id/YOUR_OIDC_ID"
        }
    ]
}
```

Now the relationship is:

```text
EKS Cluster
     │
     │ has
     ▼
OIDC Issuer
     │
     │ registered in IAM as
     ▼
IAM OIDC Provider
```

This is one of the fundamental components required for IRSA.

---

# Step 4: Create or Obtain the Required IAM Policy

Before creating the IAM Role, determine what AWS permissions the application needs.

For example, the AWS Load Balancer Controller requires permissions to manage:

* Application Load Balancers
* Target Groups
* Listeners
* Listener Rules
* Security Groups
* EC2 networking resources

The official controller installation provides an IAM policy document.

Download it:

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.13.3/docs/install/iam_policy.json
```

Create the IAM policy:

```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

The output contains the policy ARN:

```text
arn:aws:iam::YOUR_AWS_ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy
```

Conceptually:

```text
IAM Policy
      │
      ▼
Defines permissions

Example:
 ├── elasticloadbalancing:CreateLoadBalancer
 ├── elasticloadbalancing:CreateTargetGroup
 ├── ec2:DescribeSubnets
 └── etc.
```

Important:

> A policy does not give permissions by itself. It must be attached to an IAM identity such as a Role.

---

# Step 5: Create an IAM Role and Kubernetes ServiceAccount Using eksctl

The easiest way to configure IRSA is using `eksctl`.

General command:

```bash
eksctl create iamserviceaccount \
  --cluster=<CLUSTER_NAME> \
  --namespace=<NAMESPACE> \
  --name=<SERVICE_ACCOUNT_NAME> \
  --attach-policy-arn=<IAM_POLICY_ARN> \
  --override-existing-serviceaccounts \
  --region=<REGION> \
  --approve
```

Example for AWS Load Balancer Controller:

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

---

# What Does This Command Actually Create?

This command performs several tasks.

## 1. Creates an IAM Role

Example:

```text
eksctl-YOUR_CLUSTER_NAME-addon-iamservicea-Role1-XXXXXXXX
```

---

## 2. Attaches the IAM Policy to the Role

```text
IAM Role
    │
    └── AWSLoadBalancerControllerIAMPolicy
```

Therefore:

```text
IAM Role
      │
      ▼
IAM Policy
      │
      ▼
AWS Permissions
```

---

## 3. Configures the IAM Role Trust Policy

The role must trust the EKS OIDC Provider.

Conceptually:

```text
IAM Role
    │
    │ Trust Relationship
    ▼
IAM OIDC Provider
    │
    ▼
Specific Kubernetes ServiceAccount
```

The trust policy restricts who can assume the role.

Conceptually, it contains something like:

```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "IAM OIDC PROVIDER ARN"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "<OIDC_PROVIDER>:sub": "system:serviceaccount:<NAMESPACE>:<SERVICE_ACCOUNT>"
    }
  }
}
```

For example:

```text
system:serviceaccount:kube-system:aws-load-balancer-controller
```

This is extremely important.

The trust policy does not simply say:

> "Any Pod in the cluster can assume this role."

Instead, it can restrict access to:

```text
Cluster
  │
  └── Namespace: kube-system
         │
         └── ServiceAccount:
             aws-load-balancer-controller
```

This provides fine-grained security.

---

## 4. Creates the Kubernetes ServiceAccount

Example:

```text
kube-system/aws-load-balancer-controller
```

Verify:

```bash
kubectl get sa \
  aws-load-balancer-controller \
  -n kube-system
```

---

## 5. Annotates the ServiceAccount

Check:

```bash
kubectl get sa \
  aws-load-balancer-controller \
  -n kube-system \
  -o yaml
```

You should see:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: aws-load-balancer-controller
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::YOUR_AWS_ACCOUNT_ID:role/YOUR_ROLE_NAME
```

The annotation creates the connection:

```text
Kubernetes
────────────

ServiceAccount
aws-load-balancer-controller
        │
        │ annotation
        │
        ▼
IAM Role ARN


AWS
────────────

IAM Role
        │
        ▼
IAM Policy
        │
        ▼
AWS Permissions
```

---

# Step 6: Verify the IAM Role

List the roles:

```bash
aws iam list-roles \
  --query 'Roles[?contains(RoleName, `aws-load-balancer-controller`)].RoleName'
```

Or inspect the role in the AWS Console.

Check:

### Permissions

Verify that the required policy is attached.

```text
IAM Role
   │
   └── Permissions
         │
         └── AWSLoadBalancerControllerIAMPolicy
```

### Trust Relationships

Verify that the role trusts the EKS OIDC provider.

The trust relationship should restrict access to the intended ServiceAccount.

---

# Step 7: Deploy the Application Using the ServiceAccount

Creating the ServiceAccount alone is not enough.

The Pod must explicitly use it.

Example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: example-app
  template:
    metadata:
      labels:
        app: example-app
    spec:
      serviceAccountName: aws-load-balancer-controller
      containers:
        - name: app
          image: example-image
```

The important line is:

```yaml
serviceAccountName: aws-load-balancer-controller
```

The complete chain becomes:

```text
Pod
 │
 │ uses
 ▼
ServiceAccount
aws-load-balancer-controller
 │
 │ annotation contains IAM Role ARN
 ▼
IAM Role
 │
 │ permissions attached
 ▼
IAM Policy
 │
 ▼
AWS APIs
```

---

# IRSA Authentication Flow

When the Pod needs to call AWS:

```text
1. Pod starts
      │
      ▼
2. Pod uses a Kubernetes ServiceAccount
      │
      ▼
3. Kubernetes provides a ServiceAccount identity token
      │
      ▼
4. Pod requests temporary AWS credentials
      │
      ▼
5. AWS validates the token against the IAM OIDC Provider
      │
      ▼
6. IAM checks the Role Trust Policy
      │
      ▼
7. AWS verifies:
   "Is this ServiceAccount allowed to assume this Role?"
      │
      ▼
8. If allowed → STS provides temporary credentials
      │
      ▼
9. Pod calls AWS APIs using the IAM Role permissions
```

Conceptually:

```text
                 KUBERNETES

                      Pod
                       │
                       ▼
                ServiceAccount
                       │
                  OIDC Token
                       │
                       │
───────────────────────┼────────────────────────
                       │
                       ▼
                    AWS IAM

               IAM OIDC Provider
                       │
                       ▼
                 Trust Policy
                       │
                 Allowed?
                  /      \
                YES       NO
                 │         │
                 ▼         ▼
             IAM Role    Access Denied
                 │
                 ▼
             IAM Policy
                 │
                 ▼
          Temporary AWS Credentials
```

---

# Important IRSA Components Summary

| Component                 | Purpose                                          |
| ------------------------- | ------------------------------------------------ |
| EKS Cluster OIDC Issuer   | Provides Kubernetes workload identity            |
| IAM OIDC Provider         | Allows AWS IAM to trust the EKS OIDC issuer      |
| Kubernetes ServiceAccount | Kubernetes identity used by Pods                 |
| IAM Role                  | AWS identity that provides temporary permissions |
| IAM Policy                | Defines allowed AWS actions                      |
| Trust Policy              | Defines who can assume the IAM Role              |
| ServiceAccount Annotation | Connects the ServiceAccount to the IAM Role      |

---

# Method 2: EKS Pod Identity

## What is EKS Pod Identity?

EKS Pod Identity is another way to give Kubernetes workloads AWS IAM permissions.

Unlike IRSA, you do not need to create an IAM OIDC Provider.

The architecture is:

```text
Pod
 │
 ▼
Kubernetes ServiceAccount
 │
 ▼
EKS Pod Identity Association
 │
 ▼
IAM Role
 │
 ▼
IAM Policy
 │
 ▼
AWS Resources
```

EKS Pod Identity uses the EKS Pod Identity Agent running inside the cluster.

Check for it:

```bash
kubectl get pods -n kube-system \
  | grep eks-pod-identity-agent
```

Example:

```text
eks-pod-identity-agent-xxxxx   Running
eks-pod-identity-agent-yyyyy   Running
```

Usually, there is one Pod Identity Agent per node.

---

# Important: Pod Identity Agent vs Pod Identity Association

These are different things.

## Pod Identity Agent

The agent is installed inside the EKS cluster.

```text
EKS Cluster
    │
    └── Pod Identity Agent
```

Having the agent installed does NOT mean a workload already has an IAM Role.

---

## Pod Identity Association

An association connects:

```text
EKS Cluster
     │
     ├── Namespace
     │
     ├── ServiceAccount
     │
     └── IAM Role
```

Example:

```text
Cluster:
YOUR_CLUSTER_NAME

Namespace:
kube-system

ServiceAccount:
aws-load-balancer-controller

IAM Role:
AWSLoadBalancerControllerRole
```

This relationship is called a:

> **Pod Identity Association**

---

# Step 1: Ensure EKS Pod Identity Agent is Installed

Check:

```bash
kubectl get daemonset \
  -n kube-system
```

Or:

```bash
kubectl get pods \
  -n kube-system \
  | grep eks-pod-identity-agent
```

If the agent is running:

```text
eks-pod-identity-agent-xxxxx   Running
```

then the cluster has the required agent.

---

# Step 2: Create the Required IAM Policy

Just like IRSA, first determine what AWS permissions the workload needs.

Example:

```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

This creates:

```text
IAM Policy
      │
      ▼
Defines AWS permissions
```

---

# Step 3: Create an IAM Role for Pod Identity

With Pod Identity, the IAM Role trust policy is different from IRSA.

IRSA trusts:

```text
OIDC Provider
```

Pod Identity trusts:

```text
pods.eks.amazonaws.com
```

Conceptually:

```json
{
  "Effect": "Allow",
  "Principal": {
    "Service": "pods.eks.amazonaws.com"
  },
  "Action": [
    "sts:AssumeRole",
    "sts:TagSession"
  ]
}
```

Create the trust policy file:

```bash
cat <<EOF > trust-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "pods.eks.amazonaws.com"
            },
            "Action": [
                "sts:AssumeRole",
                "sts:TagSession"
            ]
        }
    ]
}
EOF
```

Create the role:

```bash
aws iam create-role \
  --role-name AWSLoadBalancerControllerPodIdentityRole \
  --assume-role-policy-document file://trust-policy.json
```

Attach the policy:

```bash
aws iam attach-role-policy \
  --role-name AWSLoadBalancerControllerPodIdentityRole \
  --policy-arn arn:aws:iam::YOUR_AWS_ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy
```

Now we have:

```text
IAM Role
     │
     └── IAM Policy
```

---

# Step 4: Create the Kubernetes ServiceAccount

Create the ServiceAccount:

```bash
kubectl create serviceaccount \
  aws-load-balancer-controller \
  -n kube-system
```

Or YAML:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: aws-load-balancer-controller
  namespace: kube-system
```

Important difference:

With Pod Identity, you do NOT need this annotation:

```yaml
eks.amazonaws.com/role-arn: ...
```

That annotation belongs to IRSA.

---

# Step 5: Create the Pod Identity Association

Now connect:

```text
EKS Cluster
     +
Namespace
     +
ServiceAccount
     +
IAM Role
```

Run:

```bash
aws eks create-pod-identity-association \
  --cluster-name <CLUSTER_NAME> \
  --namespace <NAMESPACE> \
  --service-account <SERVICE_ACCOUNT_NAME> \
  --role-arn <IAM_ROLE_ARN>
```

Example:

```bash
aws eks create-pod-identity-association \
  --cluster-name YOUR_CLUSTER_NAME \
  --namespace kube-system \
  --service-account aws-load-balancer-controller \
  --role-arn arn:aws:iam::YOUR_AWS_ACCOUNT_ID:role/AWSLoadBalancerControllerPodIdentityRole \
  --region YOUR_AWS_REGION
```

AWS creates an association similar to:

```text
EKS Cluster
YOUR_CLUSTER_NAME
       │
       ▼
Namespace
kube-system
       │
       ▼
ServiceAccount
aws-load-balancer-controller
       │
       ▼
Pod Identity Association
       │
       ▼
IAM Role
```

---

# Step 6: Verify the Pod Identity Association

Run:

```bash
aws eks list-pod-identity-associations \
  --cluster-name <CLUSTER_NAME> \
  --region <REGION>
```

Example:

```bash
aws eks list-pod-identity-associations \
  --cluster-name YOUR_CLUSTER_NAME \
  --region YOUR_AWS_REGION
```

To inspect association details:

```bash
aws eks describe-pod-identity-association \
  --cluster-name <CLUSTER_NAME> \
  --association-id <ASSOCIATION_ID> \
  --region <REGION>
```

---

# Pod Identity Authentication Flow

The flow is:

```text
Pod
 │
 ▼
ServiceAccount
 │
 ▼
EKS Pod Identity Agent
 │
 ▼
EKS Pod Identity Association
 │
 ▼
IAM Role
 │
 ▼
IAM Policy
 │
 ▼
AWS Credentials
```

More detailed:

```text
1. Pod starts
      │
      ▼
2. Pod uses Kubernetes ServiceAccount
      │
      ▼
3. EKS Pod Identity Agent participates in credential delivery
      │
      ▼
4. EKS checks Pod Identity Association
      │
      ▼
5. Association identifies IAM Role
      │
      ▼
6. AWS provides temporary credentials
      │
      ▼
7. Pod accesses AWS services
```

---

# IRSA vs EKS Pod Identity

## Architecture Comparison

### IRSA

```text
Pod
 │
 ▼
ServiceAccount
 │
 ▼
OIDC Token
 │
 ▼
IAM OIDC Provider
 │
 ▼
IAM Role
 │
 ▼
IAM Policy
```

### EKS Pod Identity

```text
Pod
 │
 ▼
ServiceAccount
 │
 ▼
EKS Pod Identity Agent
 │
 ▼
Pod Identity Association
 │
 ▼
IAM Role
 │
 ▼
IAM Policy
```

---

# Main Differences

| Feature                   | IRSA                             | EKS Pod Identity         |
| ------------------------- | -------------------------------- | ------------------------ |
| Kubernetes ServiceAccount | Yes                              | Yes                      |
| IAM Role                  | Yes                              | Yes                      |
| IAM Policy                | Yes                              | Yes                      |
| OIDC Provider Required    | Yes                              | No                       |
| ServiceAccount Annotation | Yes                              | No                       |
| Pod Identity Agent        | No                               | Yes                      |
| Pod Identity Association  | No                               | Yes                      |
| Trust Principal           | OIDC Provider                    | `pods.eks.amazonaws.com` |
| Works outside EKS         | Yes, OIDC-based approaches exist | No, EKS-specific         |
| Management complexity     | More components                  | Simpler                  |
| Legacy ecosystem support  | Very common                      | Newer                    |

---

# The Most Important Difference

The biggest difference is **how AWS knows which IAM Role belongs to a Kubernetes ServiceAccount**.

## IRSA

The connection is stored partly in the Kubernetes ServiceAccount:

```yaml
annotations:
  eks.amazonaws.com/role-arn: arn:aws:iam::YOUR_AWS_ACCOUNT_ID:role/YOUR_ROLE_NAME
```

And authorization relies on:

```text
OIDC Provider
+
IAM Role Trust Policy
```

---

## EKS Pod Identity

The connection is stored as an AWS resource:

```text
Pod Identity Association

Cluster
+
Namespace
+
ServiceAccount
+
IAM Role
```

There is no IAM role annotation required on the ServiceAccount.

---

# Which Method Should I Use?

There is no universal answer.

## Use IRSA when:

* Existing workloads already use IRSA.
* Official installation documentation specifically uses IRSA.
* You need compatibility with older tooling.
* Your organization already manages OIDC-based workload identity.

## Use EKS Pod Identity when:

* Starting a new EKS project.
* You want simpler IAM integration.
* You prefer AWS-managed associations.
* Your workload/tool supports Pod Identity.

---

# Important Rule: Do Not Mix Them Accidentally

For a workload, choose one identity mechanism intentionally.

Avoid creating unnecessary overlapping configurations like:

```text
ServiceAccount
   │
   ├── IRSA Role Annotation
   │
   └── Pod Identity Association
```

Instead, clearly choose:

```text
Option A → IRSA
```

or:

```text
Option B → EKS Pod Identity
```

---

# Production Mental Model

Regardless of which method is used, remember this:

```text
                    WHO?
                     │
                     ▼
          Kubernetes ServiceAccount
                     │
                     │
          ┌──────────┴──────────┐
          │                     │
          ▼                     ▼
        IRSA              Pod Identity
          │                     │
          ▼                     ▼
      IAM Role             IAM Role
          │                     │
          └──────────┬──────────┘
                     ▼
                IAM Policy
                     │
                     ▼
              AWS Permissions
                     │
                     ▼
               AWS Resources
```

The ServiceAccount answers:

> **Which Kubernetes workload is this?**

The IAM Role answers:

> **Which AWS identity should it become?**

The IAM Policy answers:

> **What is that identity allowed to do?**

And IRSA or Pod Identity answers:

> **How does the Kubernetes identity securely obtain the AWS identity?**

---

# Quick Implementation Checklist

## IRSA

```text
[ ] Create EKS Cluster
[ ] Get EKS OIDC Issuer URL
[ ] Associate IAM OIDC Provider
[ ] Create IAM Policy
[ ] Create IAM Role
[ ] Configure Role Trust Policy
[ ] Attach IAM Policy to Role
[ ] Create Kubernetes ServiceAccount
[ ] Annotate ServiceAccount with IAM Role ARN
[ ] Deploy Pod using ServiceAccount
```

Using `eksctl`, several steps are automated:

```bash
eksctl create iamserviceaccount ...
```

---

## EKS Pod Identity

```text
[ ] Create EKS Cluster
[ ] Install/Enable Pod Identity Agent
[ ] Create IAM Policy
[ ] Create IAM Role
[ ] Configure pods.eks.amazonaws.com Trust Policy
[ ] Attach IAM Policy to Role
[ ] Create Kubernetes ServiceAccount
[ ] Create Pod Identity Association
[ ] Deploy Pod using ServiceAccount
```

---

# Final Summary

Both mechanisms solve the same problem:

> **Giving a Kubernetes workload secure, temporary AWS credentials without putting long-term AWS access keys inside containers.**

### IRSA

```text
ServiceAccount
      │
      ▼
OIDC
      │
      ▼
IAM Role
      │
      ▼
IAM Policy
```

### EKS Pod Identity

```text
ServiceAccount
      │
      ▼
Pod Identity Association
      │
      ▼
IAM Role
      │
      ▼
IAM Policy
```

For production environments, always follow the principle of least privilege:

```text
One workload
      │
      ▼
Dedicated ServiceAccount
      │
      ▼
Dedicated IAM Role
      │
      ▼
Minimum required IAM permissions
```

Avoid giving application Pods broad permissions such as:

```text
AdministratorAccess
```

unless there is an exceptional and explicitly justified reason.

---
