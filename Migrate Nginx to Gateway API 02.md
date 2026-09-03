# Complete Kubernetes Ingress to Gateway API Migration Guide

## A Production-Ready Migration with Zero Downtime

---

## 📋 Table of Contents

1. [Overview](#1-overview)
2. [Why Migrate from Ingress to Gateway API?](#2-why-migrate-from-ingress-to-gateway-api)
3. [Architecture Comparison](#3-architecture-comparison)
4. [Important Concepts](#4-important-concepts)
5. [Prerequisites](#5-prerequisites)
6. [Environment Setup](#6-environment-setup)
7. [Migration Strategy](#7-migration-strategy)
8. [Phase 1: Existing NGINX Ingress Setup](#8-phase-1-existing-nginx-ingress-setup)
9. [Phase 2: TLS with cert-manager and Let's Encrypt](#9-phase-2-tls-with-cert-manager-and-lets-encrypt)
10. [Phase 3: Install Envoy Gateway](#10-phase-3-install-envoy-gateway)
11. [Phase 4: Create Gateway API Resources](#11-phase-4-create-gateway-api-resources)
12. [Phase 5: Test Envoy Before DNS Cutover](#12-phase-5-test-envoy-before-dns-cutover)
13. [Phase 6: DNS Cutover](#13-phase-6-dns-cutover)
14. [Phase 7: Verify Production Traffic](#14-phase-7-verify-production-traffic)
15. [Phase 8: Remove NGINX Ingress](#15-phase-8-remove-nginx-ingress)
16. [Phase 9: Weighted Routing and Canary Deployments](#16-phase-9-weighted-routing-and-canary-deployments)
17. [Rollback Strategy](#17-rollback-strategy)
18. [Troubleshooting Guide](#18-troubleshooting-guide)
19. [Useful Commands](#19-useful-commands)
20. [Production Checklist](#20-production-checklist)
21. [Final Architecture](#21-final-architecture)
22. [References](#22-references)

---

## 1. Overview

This guide walks you through a complete, production-ready migration from **NGINX Ingress Controller** to **Envoy Gateway** using the **Kubernetes Gateway API**. The migration is designed to be **zero-downtime**, allowing you to safely transition production traffic.

### What You'll Learn

- ✅ Deploy and configure NGINX Ingress Controller on EKS
- ✅ Configure TLS with cert-manager and Let's Encrypt using DNS-01
- ✅ Set up host-based routing with subdomains
- ✅ Install Envoy Gateway with Helm
- ✅ Create Gateway API resources (GatewayClass, Gateway, HTTPRoute)
- ✅ Perform zero-downtime DNS migration
- ✅ Implement weighted routing and canary deployments
- ✅ Troubleshoot common issues
- ✅ Rollback strategies

### Lab Environment

```text
Cloud Provider: Amazon Web Services (AWS)
Kubernetes Distribution: Amazon EKS
Domain: yourdomain.com (replace with your actual domain)
DNS Provider: AWS Route 53
Certificate Authority: Let's Encrypt
Ingress Controller: NGINX Ingress (to be migrated)
Gateway Implementation: Envoy Gateway
```

---

## 2. Why Migrate from Ingress to Gateway API?

### The Problem with Ingress

Kubernetes Ingress API is intentionally limited. As requirements grow, implementations rely on controller-specific annotations:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
```

**Problem**: These annotations are implementation-specific and not portable.

### The Solution: Gateway API

Gateway API moves routing concepts into standardized Kubernetes resources:

```yaml
backendRefs:
  - name: application-v1
    port: 80
    weight: 90
  - name: application-v2
    port: 80
    weight: 10
```

**Benefits**:
- ✅ Standardized across implementations
- ✅ Role-based separation (Platform vs Application teams)
- ✅ Native weighted routing and canary deployments
- ✅ More expressive than Ingress
- ✅ Future-proof

### Important Note About NGINX Ingress

The community Ingress-NGINX project entered retirement in March 2026. Existing deployments work but receive no further bug fixes or security updates. **Migration is recommended.**

---

## 3. Architecture Comparison

### Before: NGINX Ingress Architecture

```text
┌─────────────────────────────────────────────────────────────────┐
│                   NGINX Ingress Architecture                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Internet → Route53 → AWS CLB → NGINX Controller → Apps        │
│                                                                 │
│  Components:                                                    │
│  - Ingress (API resource)                                       │
│  - NGINX Ingress Controller                                     │
│  - NGINX-specific annotations                                   │
│  - AWS Classic Load Balancer (CLB)                             │
│  - Single point of routing                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### After: Envoy Gateway Architecture

```text
┌─────────────────────────────────────────────────────────────────┐
│                   Envoy Gateway Architecture                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Internet → Route53 → AWS NLB → Envoy Proxy → Apps              │
│                                                                 │
│  Components:                                                    │
│  - GatewayClass (defines the implementation)                    │
│  - Gateway (entry point)                                        │
│  - HTTPRoute (routing rules)                                    │
│  - Envoy Proxy (data plane)                                     │
│  - Native Gateway API (no annotations)                         │
│  - Separation of concerns (Platform vs Application)            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Feature Comparison

| Feature | NGINX Ingress | Envoy Gateway |
|---------|--------------|---------------|
| **API** | Ingress (v1) | Gateway API (v1) |
| **Routing** | Annotations-based | Native API fields |
| **Weighted Routing** | Limited via annotations | Native `weight` field |
| **Canary Deployments** | Complex annotations | Simple weight shifting |
| **Multi-Tenancy** | Limited | Built-in namespace isolation |
| **Observability** | Basic | Built-in Prometheus |
| **Future-Proof** | Legacy | Industry standard |
| **Role Separation** | No | Yes (Platform vs App) |

---

## 4. Important Concepts

### GatewayClass

Defines which Gateway controller should manage a Gateway.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

**Purpose**: "Which Gateway implementation is responsible?"

### Gateway

Defines the network entry point.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: application-gateway
  namespace: app1-ns
spec:
  gatewayClassName: eg
  listeners:
  - name: http
    protocol: HTTP
    port: 80
  - name: https
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: application-tls
```

**Purpose**: "Where does traffic enter the cluster?"

### HTTPRoute

Defines application routing rules.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: desktop-route
  namespace: app1-ns
spec:
  parentRefs:
  - name: application-gateway
  hostnames:
  - yourdomain.com
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: desktop-svc
      port: 80
```

**Purpose**: "Where should this request go?"

### The Relationship

```text
GatewayClass
    │
    │ "Which implementation?"
    ▼
Gateway
    │
    │ "Where does traffic enter?"
    ▼
HTTPRoute
    │
    │ "How should traffic be routed?"
    ▼
Service
    │
    │ "Which application?"
    ▼
Pod
```

---

## 5. Prerequisites

### Infrastructure Requirements

- AWS Account with appropriate permissions
- EKS Cluster (1.28+ recommended)
- kubectl configured
- Helm installed
- AWS CLI configured
- Route 53 hosted zone for your domain
- Existing Kubernetes application

### Verify Prerequisites

```bash
# Check kubectl
kubectl version --client

# Check Helm
helm version

# Check AWS CLI
aws --version

# Check EKS cluster access
aws eks update-kubeconfig --region <AWS_REGION> --name <CLUSTER_NAME>
kubectl get nodes
```

### Software Versions Used in This Guide

```text
Kubernetes: v1.36.3
NGINX Ingress Controller: v1.15.1
cert-manager: v1.17.1
Envoy Gateway: v1.6.1
Gateway API CRDs: v1.2.0
Helm: v3.x
AWS CLI: v2.x
```

---

## 6. Environment Setup

### Set Environment Variables

```bash
# --- REQUIRED: Replace these with your actual values ---
export CLUSTER_NAME="your-cluster-name"
export AWS_REGION="us-east-1"
export AWS_ACCOUNT_ID="123456789012"
export HOSTED_ZONE_ID="Z0123456789ABCDEF"
export DOMAIN="yourdomain.com"
export EMAIL="your-email@example.com"
export NAMESPACE="app1-ns"

# Optional: Set versions
export NGINX_INGRESS_VERSION="4.15.1"
export CERT_MANAGER_VERSION="v1.17.1"
export ENVOY_GATEWAY_VERSION="v1.6.1"
export GATEWAY_API_VERSION="v1.2.0"

# Verify
echo "Cluster: $CLUSTER_NAME"
echo "Region: $AWS_REGION"
echo "Domain: $DOMAIN"
echo "Namespace: $NAMESPACE"
```

### Create Working Directory

```bash
mkdir -p ~/migration-guide
cd ~/migration-guide
```

---

## 7. Migration Strategy

### The Golden Rule

> **DO NOT delete the old ingress controller before the new Gateway has been tested and verified.**

### Zero-Downtime Migration Flow

```text
Phase 1: Current State (100% NGINX)
├── Route53 → NGINX ELB → NGINX → Apps

Phase 2: Deploy Gateway Alongside (0% Traffic)
├── Route53 → NGINX ELB → NGINX → Apps
├── Envoy Gateway deployed but idle

Phase 3: Test Gateway Internally
├── Test Envoy directly (bypassing DNS)
├── Verify routing, TLS, and application functionality

Phase 4: DNS Cutover
├── Route53 → Envoy ELB → Envoy → Apps
├── NGINX still running (ready for rollback)

Phase 5: Remove NGINX
├── Route53 → Envoy ELB → Envoy → Apps
├── NGINX uninstalled
```

### Key Principle

```text
Build the new path BEFORE destroying the old one.
```

---

## 8. Phase 1: Existing NGINX Ingress Setup

### 8.1 Create EKS Cluster (If Not Already Done)

```bash
# Create a basic EKS cluster
eksctl create cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 4 \
  --managed

# Verify cluster
kubectl get nodes
kubectl get pods -A
```

### 8.2 Install NGINX Ingress Controller

```bash
# Add NGINX Ingress Helm repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install NGINX Ingress Controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --version $NGINX_INGRESS_VERSION \
  --set controller.service.type=LoadBalancer

# Verify installation
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx

# Get the LoadBalancer DNS
export NGINX_ELB=$(kubectl get svc -n ingress-nginx ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "NGINX ELB: $NGINX_ELB"
```

### 8.3 Deploy Sample Applications

Create the namespace and application YAML files:

```yaml
# ns.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: $NAMESPACE
```

```yaml
# desktop.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: desktop-deploy
  namespace: $NAMESPACE
spec:
  replicas: 2
  selector:
    matchLabels:
      app: desktop-page
  template:
    metadata:
      labels:
        app: desktop-page
    spec:
      containers:
      - name: python-http
        image: python:alpine
        command: ["python", "-m", "http.server", "5678"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: desktop-svc
  namespace: $NAMESPACE
spec:
  selector:
    app: desktop-page
  ports:
  - port: 80
    targetPort: 5678
```

```yaml
# iphone.yaml (same pattern)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iphone-deploy
  namespace: $NAMESPACE
...
```

```yaml
# android.yaml (same pattern)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: android-deploy
  namespace: $NAMESPACE
...
```

Apply the applications:

```bash
# Apply namespace
kubectl apply -f ns.yaml

# Apply applications
kubectl apply -f desktop.yaml
kubectl apply -f iphone.yaml
kubectl apply -f android.yaml

# Verify
kubectl get pods -n $NAMESPACE
kubectl get svc -n $NAMESPACE
```

### 8.4 Create Host-Based Ingress

```yaml
# ingress-host.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: application-ingress
  namespace: $NAMESPACE
spec:
  ingressClassName: nginx
  rules:
  - host: $DOMAIN
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: desktop-svc
            port:
              number: 80
  - host: iphone.$DOMAIN
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: iphone-svc
            port:
              number: 80
  - host: android.$DOMAIN
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: android-svc
            port:
              number: 80
```

Apply the Ingress:

```bash
# Replace domain placeholder
sed "s/\$DOMAIN/$DOMAIN/g" ingress-host.yaml | kubectl apply -f -

# Verify Ingress
kubectl get ingress -n $NAMESPACE
kubectl describe ingress -n $NAMESPACE application-ingress

# Test HTTP routing
curl http://$DOMAIN/
curl http://iphone.$DOMAIN/
curl http://android.$DOMAIN/
```

### 8.5 Create Route53 Records

```bash
# Create A records
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch '{
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "'$DOMAIN'.",
          "Type": "A",
          "AliasTarget": {
            "HostedZoneId": "Z35SXDOTRQ7X7K",
            "DNSName": "'$NGINX_ELB'",
            "EvaluateTargetHealth": false
          }
        }
      },
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "iphone.'$DOMAIN'.",
          "Type": "A",
          "AliasTarget": {
            "HostedZoneId": "Z35SXDOTRQ7X7K",
            "DNSName": "'$NGINX_ELB'",
            "EvaluateTargetHealth": false
          }
        }
      },
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "android.'$DOMAIN'.",
          "Type": "A",
          "AliasTarget": {
            "HostedZoneId": "Z35SXDOTRQ7X7K",
            "DNSName": "'$NGINX_ELB'",
            "EvaluateTargetHealth": false
          }
        }
      }
    ]
  }'

# Test DNS
dig $DOMAIN @1.1.1.1 +short
```

---

## 9. Phase 2: TLS with cert-manager and Let's Encrypt

### 9.1 Install cert-manager

```bash
# Add Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Install cert-manager
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version $CERT_MANAGER_VERSION \
  --set crds.enabled=true \
  --wait

# Verify
kubectl get pods -n cert-manager
kubectl get crd | grep cert-manager
```

### 9.2 Configure IAM for Route53 (EKS Pod Identity)

Create IAM policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets",
        "route53:ListResourceRecordSets"
      ],
      "Resource": "arn:aws:route53:::hostedzone/$HOSTED_ZONE_ID",
      "Condition": {
        "ForAllValues:StringEquals": {
          "route53:ChangeResourceRecordSetsRecordTypes": ["TXT"]
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": "route53:GetChange",
      "Resource": "arn:aws:route53:::change/*"
    }
  ]
}
```

Create the policy and role:

```bash
# Create the policy
aws iam create-policy \
  --policy-name CertManagerRoute53Policy \
  --policy-document file://cert-manager-policy.json

# Create IAM role with trust relationship
aws iam create-role \
  --role-name CertManagerRoute53Role \
  --assume-role-policy-document '{
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
  }'

# Attach policy to role
aws iam attach-role-policy \
  --role-name CertManagerRoute53Role \
  --policy-arn arn:aws:iam::${AWS_ACCOUNT_ID}:policy/CertManagerRoute53Policy

# Create EKS Pod Identity association
aws eks create-pod-identity-association \
  --cluster-name $CLUSTER_NAME \
  --namespace cert-manager \
  --service-account cert-manager \
  --role-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/CertManagerRoute53Role

# Restart cert-manager to pick up permissions
kubectl rollout restart deployment cert-manager -n cert-manager
```

### 9.3 Create ClusterIssuer

```yaml
# cluster-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: $EMAIL
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
    - dns01:
        route53:
          region: $AWS_REGION
          hostedZoneID: $HOSTED_ZONE_ID
```

Apply:

```bash
sed "s/\$EMAIL/$EMAIL/g; s/\$AWS_REGION/$AWS_REGION/g; s/\$HOSTED_ZONE_ID/$HOSTED_ZONE_ID/g" cluster-issuer.yaml | kubectl apply -f -

# Verify
kubectl get clusterissuer
kubectl describe clusterissuer letsencrypt-prod
```

### 9.4 Create Certificate

```yaml
# certificate.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: application-tls
  namespace: $NAMESPACE
spec:
  secretName: application-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - $DOMAIN
  - "*.$DOMAIN"
  duration: 2160h
  renewBefore: 360h
```

Apply and verify:

```bash
sed "s/\$DOMAIN/$DOMAIN/g; s/\$NAMESPACE/$NAMESPACE/g" certificate.yaml | kubectl apply -f -

# Watch the certificate issuance
kubectl get certificate -n $NAMESPACE -w
kubectl get certificaterequest,order,challenge -n $NAMESPACE

# Verify the secret
kubectl get secret -n $NAMESPACE application-tls
kubectl get secret application-tls -n $NAMESPACE -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```

### 9.5 Update Ingress with TLS

```yaml
# ingress-tls.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: application-ingress
  namespace: $NAMESPACE
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - $DOMAIN
    secretName: application-tls
  rules:
  - host: $DOMAIN
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: desktop-svc
            port:
              number: 80
  - host: iphone.$DOMAIN
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: iphone-svc
            port:
              number: 80
  - host: android.$DOMAIN
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: android-svc
            port:
              number: 80
```

Apply and test HTTPS:

```bash
sed "s/\$DOMAIN/$DOMAIN/g; s/\$NAMESPACE/$NAMESPACE/g" ingress-tls.yaml | kubectl apply -f -

# Test HTTPS
curl https://$DOMAIN/
curl https://iphone.$DOMAIN/
curl https://android.$DOMAIN/

# Check certificate
openssl s_client -connect $DOMAIN:443 -servername $DOMAIN </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```

---

## 10. Phase 3: Install Envoy Gateway

### 10.1 Install Gateway API CRDs

```bash
# Install Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v$GATEWAY_API_VERSION/standard-install.yaml

# Verify CRDs
kubectl get crd | grep gateway.networking.k8s.io
```

### 10.2 Install Envoy Gateway

```bash
# Add Envoy Gateway Helm repository
helm repo add envoy-gateway https://envoyproxy.github.io/gateway
helm repo update

# Install Envoy Gateway
helm install envoy-gateway envoy-gateway/gateway \
  --namespace envoy-gateway-system \
  --create-namespace \
  --version $ENVOY_GATEWAY_VERSION \
  --wait

# Verify installation
kubectl get pods -n envoy-gateway-system
kubectl get svc -n envoy-gateway-system
helm list -n envoy-gateway-system
```

**Note**: At this point, Envoy Gateway creates a ClusterIP service, NOT a LoadBalancer. This is expected.

---

## 11. Phase 4: Create Gateway API Resources

### 11.1 Create GatewayClass

```yaml
# gatewayclass.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

Apply and verify:

```bash
kubectl apply -f gatewayclass.yaml

# Verify GatewayClass is accepted
kubectl get gatewayclass
kubectl describe gatewayclass eg

# Check Envoy Gateway logs
kubectl logs -n envoy-gateway-system deployment/envoy-gateway --tail=20 | grep gatewayclass
```

Expected output:

```text
NAME   CONTROLLER                                      ACCEPTED   AGE
eg     gateway.envoyproxy.io/gatewayclass-controller   True       10s
```

### 11.2 Create Gateway

```yaml
# gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: application-gateway
  namespace: $NAMESPACE
spec:
  gatewayClassName: eg
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: Same
  - name: https
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: application-tls
    allowedRoutes:
      namespaces:
        from: Same
```

Apply and wait for the Gateway to become programmed:

```bash
sed "s/\$NAMESPACE/$NAMESPACE/g" gateway.yaml | kubectl apply -f -

# Watch the Gateway become ready
kubectl get gateway -n $NAMESPACE -w

# Describe the Gateway
kubectl describe gateway -n $NAMESPACE application-gateway
```

After a minute, you should see:

```text
NAME                 CLASS   ADDRESS                                                                  PROGRAMMED   AGE
application-gateway  eg      xxxxxxxx-xxxxxxxx.elb.amazonaws.com                                      True         30s
```

### 11.3 Get the Envoy LoadBalancer

```bash
# Get the Envoy LoadBalancer DNS
export ENVOY_ELB=$(kubectl get svc -n envoy-gateway-system \
  -o jsonpath='{.items[?(@.metadata.name=="envoy-app1-ns-application-gateway-*")].status.loadBalancer.ingress[0].hostname}')
echo "Envoy ELB: $ENVOY_ELB"

# Get the Envoy IP (for testing)
export ENVOY_IP=$(dig +short $ENVOY_ELB | head -1)
echo "Envoy IP: $ENVOY_IP"
```

### 11.4 Create HTTPRoutes

```yaml
# httproutes.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: desktop-route
  namespace: $NAMESPACE
spec:
  parentRefs:
  - name: application-gateway
  hostnames:
  - $DOMAIN
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: desktop-svc
      port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: iphone-route
  namespace: $NAMESPACE
spec:
  parentRefs:
  - name: application-gateway
  hostnames:
  - iphone.$DOMAIN
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: iphone-svc
      port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: android-route
  namespace: $NAMESPACE
spec:
  parentRefs:
  - name: application-gateway
  hostnames:
  - android.$DOMAIN
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: android-svc
      port: 80
```

Apply and verify:

```bash
sed "s/\$DOMAIN/$DOMAIN/g; s/\$NAMESPACE/$NAMESPACE/g" httproutes.yaml | kubectl apply -f -

# Verify HTTPRoutes are accepted
kubectl get httproute -n $NAMESPACE
kubectl describe httproute -n $NAMESPACE desktop-route
kubectl describe gateway -n $NAMESPACE application-gateway

# Check that routes are attached
kubectl get gateway -n $NAMESPACE -o yaml | grep -A 5 "Attached Routes"
```

Expected:

```text
NAME            HOSTNAMES                   AGE
desktop-route   ["$DOMAIN"]                 10s
iphone-route    ["iphone.$DOMAIN"]          10s
android-route   ["android.$DOMAIN"]         10s
```

---

## 12. Phase 5: Test Envoy Before DNS Cutover

**THIS IS THE MOST IMPORTANT STEP!** Do not skip this.

### 12.1 Test Routing

```bash
# Get the Envoy IP
export ENVOY_IP=$(dig +short $ENVOY_ELB | head -1)
echo "Envoy IP: $ENVOY_IP"

# Test desktop route
echo "Testing Desktop Route:"
curl -k --resolve $DOMAIN:443:$ENVOY_IP https://$DOMAIN/

# Test iPhone route
echo -e "\nTesting iPhone Route:"
curl -k --resolve iphone.$DOMAIN:443:$ENVOY_IP https://iphone.$DOMAIN/

# Test Android route
echo -e "\nTesting Android Route:"
curl -k --resolve android.$DOMAIN:443:$ENVOY_IP https://android.$DOMAIN/
```

Expected output:

```html
Desktop: <h1>Desktop Users</h1>
iPhone:  <h1>iPhone Users</h1>
Android: <h1>Android Users</h1>
```

### 12.2 Verify TLS Certificate

```bash
# Check the certificate served by Envoy
openssl s_client -connect $ENVOY_ELB:443 -servername $DOMAIN </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates -ext subjectAltName

# Should show:
# subject=CN = $DOMAIN
# issuer=C = US, O = Let's Encrypt
# DNS:*.$DOMAIN, DNS:$DOMAIN
```

### 12.3 Compare NGINX vs Envoy

```bash
# Compare responses from both
echo "=== NGINX (Current) ==="
curl -s -k --resolve $DOMAIN:443:$NGINX_IP https://$DOMAIN/

echo -e "\n=== Envoy (New) ==="
curl -s -k --resolve $DOMAIN:443:$ENVOY_IP https://$DOMAIN/
```

Both should return the same content.

---

## 13. Phase 6: DNS Cutover

### 13.1 Update Route53

```bash
# Update Route53 to point to Envoy ELB
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch '{
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "'$DOMAIN'.",
          "Type": "A",
          "AliasTarget": {
            "HostedZoneId": "Z35SXDOTRQ7X7K",
            "DNSName": "'$ENVOY_ELB'",
            "EvaluateTargetHealth": false
          }
        }
      },
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "iphone.'$DOMAIN'.",
          "Type": "A",
          "AliasTarget": {
            "HostedZoneId": "Z35SXDOTRQ7X7K",
            "DNSName": "'$ENVOY_ELB'",
            "EvaluateTargetHealth": false
          }
        }
      },
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "android.'$DOMAIN'.",
          "Type": "A",
          "AliasTarget": {
            "HostedZoneId": "Z35SXDOTRQ7X7K",
            "DNSName": "'$ENVOY_ELB'",
            "EvaluateTargetHealth": false
          }
        }
      }
    ]
  }'
```

### 13.2 Monitor DNS Propagation

```bash
# Wait for DNS propagation (usually 30-60 seconds)
echo "Waiting for DNS propagation..."
sleep 60

# Test DNS resolution
dig $DOMAIN @1.1.1.1 +short
dig iphone.$DOMAIN @1.1.1.1 +short
dig android.$DOMAIN @1.1.1.1 +short

# All should resolve to the Envoy IP
```

---

## 14. Phase 7: Verify Production Traffic

### 14.1 Test All Routes

```bash
# Test without --resolve (real DNS)
echo "Testing Desktop:"
curl https://$DOMAIN/

echo -e "\nTesting iPhone:"
curl https://iphone.$DOMAIN/

echo -e "\nTesting Android:"
curl https://android.$DOMAIN/
```

### 14.2 Verify TLS Certificate in Browser

Open these URLs in your browser:

```text
https://$DOMAIN/
https://iphone.$DOMAIN/
https://android.$DOMAIN/
```

Check for the padlock icon (🔒).

### 14.3 Check Envoy Logs

```bash
# Check Envoy Gateway logs
kubectl logs -n envoy-gateway-system deployment/envoy-gateway --tail=50

# Check Envoy proxy logs
kubectl logs -n envoy-gateway-system -l app.kubernetes.io/name=envoy-proxy --tail=50
```

---

## 15. Phase 8: Remove NGINX Ingress

### 15.1 Verify Everything Works

```bash
# Final verification
curl https://$DOMAIN/
curl https://iphone.$DOMAIN/
curl https://android.$DOMAIN/

# Check certificate
openssl s_client -connect $DOMAIN:443 -servername $DOMAIN </dev/null 2>/dev/null | openssl x509 -noout -issuer -dates
```

### 15.2 Uninstall NGINX

```bash
# Check current releases
helm list -n kube-system

# Uninstall NGINX Ingress
helm uninstall ingress-nginx -n kube-system

# Verify removal
kubectl get pods -n kube-system | grep ingress-nginx
kubectl get svc -n kube-system | grep ingress-nginx
```

### 15.3 Final Architecture Verification

```bash
# Check only Envoy LoadBalancer exists
kubectl get svc -A | grep LoadBalancer

# Should show only the Envoy LoadBalancer
```

---

## 16. Phase 9: Weighted Routing and Canary Deployments

### 16.1 Create Canary Version

```yaml
# canary-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: desktop-canary-deploy
  namespace: $NAMESPACE
  labels:
    app: desktop-canary-page
spec:
  replicas: 1
  selector:
    matchLabels:
      app: desktop-canary-page
  template:
    metadata:
      labels:
        app: desktop-canary-page
    spec:
      containers:
      - name: python-http
        image: python:alpine
        command: ["python", "-c", "from http.server import HTTPServer, BaseHTTPRequestHandler; class Handler(BaseHTTPRequestHandler): def do_GET(self): self.send_response(200); self.end_headers(); self.wfile.write(b'<html><body><h1>Desktop CANARY</h1></body></html>'); HTTPServer(('', 5678), Handler).serve_forever()"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: desktop-canary-svc
  namespace: $NAMESPACE
spec:
  selector:
    app: desktop-canary-page
  ports:
  - port: 80
    targetPort: 5678
```

Apply and verify:

```bash
sed "s/\$NAMESPACE/$NAMESPACE/g" canary-deployment.yaml | kubectl apply -f -

# Verify canary is running
kubectl get pods -n $NAMESPACE -l app=desktop-canary-page
kubectl get svc -n $NAMESPACE desktop-canary-svc
```

### 16.2 Implement Weighted Routing

```yaml
# weighted-routing.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: desktop-weighted-route
  namespace: $NAMESPACE
spec:
  parentRefs:
  - name: application-gateway
  hostnames:
  - $DOMAIN
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: desktop-svc
      port: 80
      weight: 90
    - name: desktop-canary-svc
      port: 80
      weight: 10
```

Apply initial weighted routing (90/10):

```bash
sed "s/\$DOMAIN/$DOMAIN/g; s/\$NAMESPACE/$NAMESPACE/g" weighted-routing.yaml | kubectl apply -f -
```

### 16.3 Test Weighted Distribution

```bash
# Test weighted distribution
for i in {1..100}; do
  curl -s https://$DOMAIN/ | grep -q "CANARY" && echo "canary" || echo "stable"
done | sort | uniq -c
```

Expected output (approximately):
```text
90 stable
10 canary
```

### 16.4 Gradual Canary Rollout

```yaml
# Phase 1: 90/10 (Initial canary)
backendRefs:
- name: desktop-svc
  port: 80
  weight: 90
- name: desktop-canary-svc
  port: 80
  weight: 10

# Phase 2: 70/30 (Increase confidence)
backendRefs:
- name: desktop-svc
  port: 80
  weight: 70
- name: desktop-canary-svc
  port: 80
  weight: 30

# Phase 3: 50/50 (Half and half)
backendRefs:
- name: desktop-svc
  port: 80
  weight: 50
- name: desktop-canary-svc
  port: 80
  weight: 50

# Phase 4: 20/80 (Gradual rollout)
backendRefs:
- name: desktop-svc
  port: 80
  weight: 20
- name: desktop-canary-svc
  port: 80
  weight: 80

# Phase 5: 0/100 (Full canary)
backendRefs:
- name: desktop-canary-svc
  port: 80
  weight: 100
```

Update weights for each phase:

```bash
# Example: Update to 70/30
cat <<EOF | sed "s/\$DOMAIN/$DOMAIN/g; s/\$NAMESPACE/$NAMESPACE/g" | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: desktop-weighted-route
  namespace: $NAMESPACE
spec:
  parentRefs:
  - name: application-gateway
  hostnames:
  - $DOMAIN
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: desktop-svc
      port: 80
      weight: 70
    - name: desktop-canary-svc
      port: 80
      weight: 30
EOF
```

### 16.5 Header-Based Routing (A/B Testing)

```yaml
# header-routing.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: desktop-header-route
  namespace: $NAMESPACE
spec:
  parentRefs:
  - name: application-gateway
  hostnames:
  - $DOMAIN
  rules:
  # Users with header "X-Canary: true" go to canary
  - matches:
    - path:
        type: PathPrefix
        value: /
      headers:
      - name: X-Canary
        value: "true"
    backendRefs:
    - name: desktop-canary-svc
      port: 80
  # All other users go to stable
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: desktop-svc
      port: 80
```

Apply and test:

```bash
sed "s/\$DOMAIN/$DOMAIN/g; s/\$NAMESPACE/$NAMESPACE/g" header-routing.yaml | kubectl apply -f -

# Test canary header
curl -H "X-Canary: true" https://$DOMAIN/

# Test regular user (stable)
curl https://$DOMAIN/
```

---

## 17. Rollback Strategy

### 17.1 Rollback to NGINX

If Envoy Gateway fails:

```bash
# 1. Point DNS back to NGINX ELB
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch '{
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "'$DOMAIN'.",
          "Type": "A",
          "AliasTarget": {
            "HostedZoneId": "Z35SXDOTRQ7X7K",
            "DNSName": "'$NGINX_ELB'",
            "EvaluateTargetHealth": false
          }
        }
      }
    ]
  }'

# 2. Wait for DNS propagation (30-60 seconds)
sleep 60

# 3. Verify traffic is back to NGINX
curl https://$DOMAIN/

# 4. Investigate Envoy issues
kubectl logs -n envoy-gateway-system deployment/envoy-gateway --tail=100
```

### 17.2 Rollback from Weighted Routing

If canary deployment has issues:

```bash
# 1. Quickly rollback to 100% stable
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: desktop-weighted-route
  namespace: $NAMESPACE
spec:
  parentRefs:
  - name: application-gateway
  hostnames:
  - $DOMAIN
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: desktop-svc
      port: 80
      weight: 100
EOF

# 2. Verify traffic is back to stable
curl https://$DOMAIN/
```

---

## 18. Troubleshooting Guide

### 18.1 GatewayClass Not Accepted

**Symptom**: `kubectl get gatewayclass` shows `ACCEPTED: False`

**Check**:
```bash
kubectl describe gatewayclass eg
kubectl logs -n envoy-gateway-system deployment/envoy-gateway --tail=50
```

**Solution**: Verify the controller name matches:
```yaml
controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

### 18.2 Gateway Not Programmed

**Symptom**: `PROGRAMMED: False`

**Check**:
```bash
kubectl describe gateway -n $NAMESPACE application-gateway
kubectl get pods -n envoy-gateway-system
```

**Common Causes**:
1. GatewayClass not accepted
2. TLS Secret missing
3. Envoy Gateway pods not running

### 18.3 HTTPRoute Not Accepted

**Symptom**: `Accepted: False`

**Check**:
```bash
kubectl describe httproute -n $NAMESPACE desktop-route
```

**Common Causes**:
1. Wrong Gateway name
2. Wrong namespace
3. Service doesn't exist
4. Wrong Service port

**Solution**: Verify the Service exists:
```bash
kubectl get svc -n $NAMESPACE desktop-svc
```

### 18.4 TLS Not Working

**Symptom**: "SSL certificate problem"

**Check**:
```bash
# Verify Secret exists
kubectl get secret -n $NAMESPACE application-tls

# Verify certificate
kubectl get secret application-tls -n $NAMESPACE -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text

# Check Gateway TLS configuration
kubectl describe gateway -n $NAMESPACE application-gateway | grep -A 10 "TLS"
```

### 18.5 DNS Still Points to NGINX

**Check**:
```bash
dig $DOMAIN @1.1.1.1 +short
```

**Solution**: Update Route53 record to point to Envoy ELB.

### 18.6 `curl --resolve` Fails

**Incorrect**:
```bash
curl --resolve $DOMAIN:443:$ENVOY_ELB https://$DOMAIN/
```

**Correct**:
```bash
# Use IP address, not DNS name
export ENVOY_IP=$(dig +short $ENVOY_ELB | head -1)
curl -k --resolve $DOMAIN:443:$ENVOY_IP https://$DOMAIN/
```

### 18.7 Envoy Not Routing

**Check**:
```bash
# Check HTTPRoute status
kubectl describe httproute -n $NAMESPACE desktop-route

# Check Envoy proxy logs
kubectl logs -n envoy-gateway-system -l app.kubernetes.io/name=envoy-proxy --tail=50

# Check Service endpoints
kubectl get endpoints -n $NAMESPACE desktop-svc
```

### 18.8 Certificate Not Issuing

**Check**:
```bash
# Check cert-manager logs
kubectl logs -n cert-manager deployment/cert-manager --tail=50

# Check challenge status
kubectl describe challenge -n $NAMESPACE

# Verify IAM permissions
kubectl logs -n cert-manager deployment/cert-manager | grep -i "route53"
```

### 18.9 Weighted Routing Not Working

**Check**:
```bash
# Verify HTTPRoute has weights
kubectl describe httproute -n $NAMESPACE desktop-weighted-route

# Test distribution
for i in {1..100}; do
  curl -s https://$DOMAIN/ | grep -q "CANARY" && echo "canary" || echo "stable"
done | sort | uniq -c
```

---

## 19. Useful Commands

### Kubernetes Resources

```bash
# Gateway API resources
kubectl get gatewayclass
kubectl get gateway -A
kubectl get httproute -A

# Pods and Services
kubectl get pods -A | grep -E "envoy|nginx|cert-manager"
kubectl get svc -A | grep LoadBalancer

# Certificates
kubectl get certificate -A
kubectl get certificaterequest -A
kubectl get challenge -A

# Ingress (old)
kubectl get ingress -A
```

### Envoy Gateway

```bash
# Controller logs
kubectl logs -n envoy-gateway-system deployment/envoy-gateway -f

# Envoy proxy logs
kubectl logs -n envoy-gateway-system -l app.kubernetes.io/name=envoy-proxy -f

# Envoy stats
kubectl port-forward -n envoy-gateway-system deployment/envoy-proxy 19000:19000
curl http://localhost:19000/stats
```

### cert-manager

```bash
# Cert-manager logs
kubectl logs -n cert-manager deployment/cert-manager -f

# Certificate status
kubectl describe certificate -n $NAMESPACE application-tls

# Challenge status
kubectl describe challenge -n $NAMESPACE
```

### DNS and Networking

```bash
# DNS resolution
dig $DOMAIN @1.1.1.1 +short
dig iphone.$DOMAIN @1.1.1.1 +short
dig android.$DOMAIN @1.1.1.1 +short

# DNS propagation
dig $DOMAIN @8.8.8.8 +short
dig $DOMAIN @ns-1990.awsdns-56.co.uk +short

# HTTPS testing
curl -vk https://$DOMAIN/
openssl s_client -connect $DOMAIN:443 -servername $DOMAIN </dev/null 2>/dev/null | openssl x509 -text
```

### Debugging

```bash
# Check all conditions
kubectl describe gatewayclass eg
kubectl describe gateway -n $NAMESPACE application-gateway
kubectl describe httproute -n $NAMESPACE desktop-route

# Get resource status in YAML
kubectl get gateway -n $NAMESPACE application-gateway -o yaml

# Watch resources
kubectl get gateway -n $NAMESPACE -w
kubectl get httproute -n $NAMESPACE -w
```

---

## 20. Production Checklist

### Before Migration

- [ ] Backup all manifests
- [ ] Document current NGINX configuration
- [ ] Document all annotations
- [ ] Document TLS configuration
- [ ] Document DNS records
- [ ] Document current LoadBalancer
- [ ] Confirm application health
- [ ] Confirm rollback procedure
- [ ] Verify monitoring
- [ ] Verify logging
- [ ] Verify alerting

### During Migration

- [ ] Install Gateway API implementation
- [ ] Keep NGINX running
- [ ] Create GatewayClass and verify acceptance
- [ ] Create Gateway and verify programmed
- [ ] Configure TLS
- [ ] Create HTTPRoutes and verify acceptance
- [ ] Verify `Accepted=True`
- [ ] Verify `ResolvedRefs=True`
- [ ] Verify `Programmed=True`
- [ ] Test Envoy LoadBalancer directly
- [ ] Test HTTPS
- [ ] Test every hostname
- [ ] Test every path
- [ ] Test application functionality
- [ ] Change DNS
- [ ] Monitor traffic

### After Migration

- [ ] Verify DNS resolution
- [ ] Verify HTTPS
- [ ] Verify certificates
- [ ] Verify application health
- [ ] Verify metrics
- [ ] Verify logs
- [ ] Verify error rates
- [ ] Keep NGINX temporarily for rollback
- [ ] Remove NGINX after stabilization
- [ ] Remove obsolete Ingress resources
- [ ] Update documentation
- [ ] Update monitoring/alerts
- [ ] Update runbooks

---

## 21. Final Architecture

```text
┌─────────────────────────────────────────────────────────────────────┐
│                         INTERNET                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│                         Route 53 DNS                               │
│                              │                                      │
│              ┌───────────────┼───────────────┐                      │
│              │               │               │                      │
│              ▼               ▼               ▼                      │
│          $DOMAIN       android.*       iphone.*                    │
│              │               │               │                      │
│              └───────────────┼───────────────┘                      │
│                              │                                      │
│                              ▼                                      │
│                       AWS LoadBalancer                              │
│                    (Envoy Gateway ELB)                              │
│                              │                                      │
│                              ▼                                      │
│                        Envoy Gateway                                │
│                     (HTTPS Termination)                             │
│                              │                                      │
│                              ▼                                      │
│                         Gateway API                                 │
│                              │                                      │
│                          HTTPRoutes                                 │
│                              │                                      │
│              ┌───────────────┼───────────────┐                      │
│              │               │               │                      │
│              ▼               ▼               ▼                      │
│        desktop-route    android-route    iphone-route               │
│              │               │               │                      │
│              ▼               ▼               ▼                      │
│         desktop-svc     android-svc     iphone-svc                  │
│              │               │               │                      │
│              ▼               ▼               ▼                      │
│         Desktop Pods     Android Pods    iPhone Pods                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 22. References

### Kubernetes Gateway API

- [Gateway API Overview](https://kubernetes.io/docs/concepts/services-networking/gateway/)
- [Gateway API Documentation](https://gateway-api.sigs.k8s.io/)
- [Ingress to Gateway API Migration](https://gateway-api.sigs.k8s.io/guides/migrating-from-ingress/)

### Envoy Gateway

- [Envoy Gateway Documentation](https://gateway.envoyproxy.io/)
- [Envoy Gateway Quickstart](https://gateway.envoyproxy.io/v1.6/tasks/quickstart/)
- [HTTP Traffic Splitting](https://gateway.envoyproxy.io/latest/tasks/traffic/http-traffic-splitting/)
- [HTTP Request Mirroring](https://gateway.envoyproxy.io/v1.6/tasks/traffic/http-request-mirroring/)
- [Secure Gateways (TLS)](https://gateway.envoyproxy.io/docs/tasks/security/secure-gateways/)

### cert-manager

- [cert-manager Documentation](https://cert-manager.io/docs/)
- [DNS-01 Challenge](https://cert-manager.io/docs/configuration/acme/dns01/)
- [Route 53 Configuration](https://cert-manager.io/docs/configuration/acme/dns01/route53/)

### NGINX Ingress

- [NGINX Ingress Documentation](https://kubernetes.github.io/ingress-nginx/)
- [Retirement Announcement](https://kubernetes.github.io/ingress-nginx/)

---

