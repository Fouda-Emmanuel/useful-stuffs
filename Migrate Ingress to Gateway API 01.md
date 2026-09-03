# Migrating from NGINX Ingress to Gateway API with Envoy Gateway on Amazon EKS

A practical, production-oriented guide for migrating an existing Kubernetes application from **NGINX Ingress** to **Gateway API + Envoy Gateway** with:

* HTTPS/TLS
* cert-manager
* Let's Encrypt
* AWS Route 53 DNS-01
* Amazon EKS Pod Identity
* AWS Load Balancer
* Host-based routing
* Zero/minimal-downtime migration
* Rollback strategy
* Gateway API traffic splitting
* Canary deployments
* Weighted routing

This guide is designed to work as both:

1. A learning guide for engineers who are new to Gateway API.
2. A reusable migration runbook for DevOps / Platform Engineering teams.

---

# Table of Contents

* [1. Overview](#1-overview)
* [2. Why migrate from Ingress to Gateway API?](#2-why-migrate-from-ingress-to-gateway-api)
* [3. Before and After Architecture](#3-before-and-after-architecture)
* [4. Important Concepts](#4-important-concepts)
* [5. Environment Variables and Placeholders](#5-environment-variables-and-placeholders)
* [6. Prerequisites](#6-prerequisites)
* [7. Migration Strategy](#7-migration-strategy)
* [8. Phase 1 — Existing NGINX Ingress](#8-phase-1--existing-nginx-ingress)
* [9. Phase 2 — TLS with cert-manager](#9-phase-2--tls-with-cert-manager)
* [10. Phase 3 — Install Envoy Gateway](#10-phase-3--install-envoy-gateway)
* [11. Phase 4 — Create GatewayClass](#11-phase-4--create-gatewayclass)
* [12. Phase 5 — Create Gateway](#12-phase-5--create-gateway)
* [13. Phase 6 — Create HTTPRoutes](#13-phase-6--create-httproutes)
* [14. Phase 7 — Test Envoy Before DNS Cutover](#14-phase-7--test-envoy-before-dns-cutover)
* [15. Phase 8 — DNS Cutover](#15-phase-8--dns-cutover)
* [16. Phase 9 — Verify Production Traffic](#16-phase-9--verify-production-traffic)
* [17. Phase 10 — Remove NGINX](#17-phase-10--remove-nginx)
* [18. Rollback Strategy](#18-rollback-strategy)
* [19. Understanding TLS Termination](#19-understanding-tls-termination)
* [20. Understanding Gateway API Objects](#20-understanding-gateway-api-objects)
* [21. Gateway API vs Ingress](#21-gateway-api-vs-ingress)
* [22. Weighted Traffic Splitting](#22-weighted-traffic-splitting)
* [23. Canary Deployment](#23-canary-deployment)
* [24. Progressive Canary Rollout](#24-progressive-canary-rollout)
* [25. Traffic Mirroring](#25-traffic-mirroring)
* [26. Production Checklist](#26-production-checklist)
* [27. Troubleshooting](#27-troubleshooting)
* [28. Useful Commands](#28-useful-commands)
* [29. Final Architecture](#29-final-architecture)
* [30. References](#30-references)

---

# 1. Overview

Kubernetes originally provided the `Ingress` API as the standard way to expose HTTP and HTTPS applications.

A typical architecture looked like:

```text
Internet
   |
   v
AWS Load Balancer
   |
   v
NGINX Ingress Controller
   |
   v
Ingress
   |
   v
Kubernetes Service
   |
   v
Pods
```

The Kubernetes Gateway API was introduced to provide a more expressive and extensible model for traffic management.

Gateway API separates responsibilities into different resources:

```text
GatewayClass
     |
     v
  Gateway
     |
     v
 HTTPRoute
     |
     v
 Service
     |
     v
 Pods
```

This separation makes it easier to distinguish:

* Platform configuration
* Network entry points
* Application routing
* Application ownership

Gateway API is the successor to the Kubernetes Ingress API, although migrating from Ingress requires converting the existing resources because Gateway API does not simply replace the `Ingress` kind.

---

# 2. Why migrate from Ingress to Gateway API?

The Kubernetes Ingress API is relatively simple and intentionally limited.

As requirements become more advanced, implementations frequently rely on controller-specific annotations and configuration.

For example:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
```

The problem is that these annotations are implementation-specific.

A different controller may use completely different annotations.

Gateway API moves many routing concepts into standardized Kubernetes resources.

For example:

```yaml
backendRefs:
  - name: application-v1
    port: 80
    weight: 90

  - name: application-v2
    port: 80
    weight: 10
```

This makes the intent much clearer.

## Important note about NGINX Ingress

The community Ingress-NGINX project entered retirement in March 2026. Existing deployments do not suddenly stop working, but the project documentation states that after retirement there are no further releases, bug fixes, or security updates.

Therefore, existing NGINX Ingress environments should be evaluated and migrated rather than assuming the controller will receive indefinite future maintenance.

---

# 3. Before and After Architecture

## Before

The original architecture:

```text
                         Internet
                            |
                            v
                     Route 53 DNS
                            |
                            v
                    AWS Load Balancer
                            |
                            v
                  NGINX Ingress Controller
                            |
                            v
                     Ingress Object
                            |
                            v
                     Kubernetes Service
                            |
                            v
                           Pods
```

For host-based routing:

```text
foudadev.example.com
        |
        v
      NGINX
        |
        +----> desktop-service
        |
        +----> android-service
        |
        +----> iphone-service
```

---

# 4. Important Concepts

Before performing the migration, understand these resources.

## GatewayClass

Defines which Gateway controller should manage a Gateway.

Example:

```yaml
kind: GatewayClass
```

Think of it as:

> "Which Gateway implementation is responsible for this Gateway?"

---

## Gateway

Defines the network entry point.

It controls things such as:

* HTTP listener
* HTTPS listener
* Port
* TLS configuration
* Which routes can attach

Example:

```yaml
kind: Gateway
```

Think:

> "Where does traffic enter the cluster?"

---

## HTTPRoute

Defines application routing.

It can match based on:

* Hostname
* Path
* Headers
* Query parameters
* HTTP method

and forward traffic to:

* Kubernetes Services

Example:

```yaml
kind: HTTPRoute
```

Think:

> "Where should this request go?"

---

## Service

Gateway API normally forwards traffic to Kubernetes Services.

Example:

```text
HTTPRoute
    |
    v
desktop-svc
    |
    v
Desktop Pods
```

---

# 5. Environment Variables and Placeholders

Never commit real infrastructure identifiers, credentials, or private information to a public repository.

Use placeholders such as:

```text
<CLUSTER_NAME>
<AWS_REGION>
<AWS_ACCOUNT_ID>
<HOSTED_ZONE_ID>
<DOMAIN>
<EMAIL>
<IAM_ROLE_ARN>
<ENVoy_ELB_DNS>
<APPLICATION_NAMESPACE>
```

Example:

```bash
export CLUSTER_NAME="<CLUSTER_NAME>"
export AWS_REGION="<AWS_REGION>"
export DOMAIN="<DOMAIN>"
export NAMESPACE="<APPLICATION_NAMESPACE>"
```

For example, instead of:

```text
123456789012
```

use:

```text
<AWS_ACCOUNT_ID>
```

Instead of:

```text
example.com
```

use:

```text
<DOMAIN>
```

---

# 6. Prerequisites

The examples assume:

* Amazon EKS
* `kubectl`
* `helm`
* AWS CLI
* Route 53
* An existing Kubernetes application
* Existing NGINX Ingress
* A DNS name pointing to the current ingress endpoint

Verify:

```bash
kubectl version --client
```

```bash
helm version
```

```bash
aws --version
```

Verify the cluster:

```bash
aws eks update-kubeconfig \
  --region <AWS_REGION> \
  --name <CLUSTER_NAME>
```

Then:

```bash
kubectl get nodes
```

All nodes should be `Ready`.

---

# 7. Migration Strategy

The most important principle of this migration is:

> **Do not replace the old ingress controller before the new Gateway has been tested.**

The safe migration looks like:

```text
                  OLD PATH
                  --------

Route 53
   |
   v
NGINX Load Balancer
   |
   v
NGINX
   |
   v
Applications


                  NEW PATH
                  --------

Route 53
   |
   v
Envoy Load Balancer
   |
   v
Envoy Gateway
   |
   v
HTTPRoutes
   |
   v
Applications
```

During migration:

```text
                 Applications
                 /           \
                /             \
             NGINX           Envoy
               |                |
           OLD PATH          NEW PATH
```

The old path remains available while the new path is tested.

Only after the new path is confirmed should DNS be changed.

---

# 8. Phase 1 — Existing NGINX Ingress

Assume the existing application looks like:

```text
desktop-svc
android-svc
iphone-svc
```

and the existing Ingress contains host-based routing.

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: application-ingress
  namespace: <APPLICATION_NAMESPACE>
spec:
  ingressClassName: nginx

  tls:
    - hosts:
        - <DOMAIN>
        - android.<DOMAIN>
        - iphone.<DOMAIN>
      secretName: <TLS_SECRET>

  rules:

    - host: <DOMAIN>
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: desktop-svc
                port:
                  number: 80

    - host: android.<DOMAIN>
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: android-svc
                port:
                  number: 80

    - host: iphone.<DOMAIN>
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: iphone-svc
                port:
                  number: 80
```

Verify:

```bash
kubectl get ingress -A
```

```bash
kubectl describe ingress application-ingress \
  -n <APPLICATION_NAMESPACE>
```

Before migrating, verify that the existing application works.

---

# 9. Phase 2 — TLS with cert-manager

Gateway API does not itself issue certificates.

A certificate management solution such as cert-manager can obtain certificates from Let's Encrypt and store them as Kubernetes TLS Secrets.

For AWS + Route 53, DNS-01 is a good option, especially when wildcard certificates are required.

Example certificate:

```text
<DOMAIN>
*.<DOMAIN>
```

The basic flow is:

```text
                    cert-manager
                         |
                         v
                  Let's Encrypt
                         |
                   DNS-01 challenge
                         |
                         v
                     Route 53
                         |
                         v
                  TXT record
                         |
                         v
                  Let's Encrypt
                   validates DNS
                         |
                         v
                Certificate issued
                         |
                         v
              Kubernetes TLS Secret
```

cert-manager's Route 53 documentation recommends using temporary IAM role credentials rather than long-lived access keys where possible. The Route 53 permissions can also be restricted to the required hosted zone and TXT records.

## Install cert-manager

Use the current cert-manager installation instructions for the version you choose.

Verify:

```bash
kubectl get pods -n cert-manager
```

You should see the cert-manager components running.

---

## AWS authentication for cert-manager

On EKS, avoid storing long-lived AWS access keys inside Kubernetes Secrets.

Use an AWS IAM role through an appropriate EKS workload identity mechanism.

For example:

```text
cert-manager Pod
       |
       v
EKS workload identity
       |
       v
IAM Role
       |
       v
Route 53 API
```

The IAM role should have only the permissions required to manage the DNS challenge.

---

## Example Route 53 policy

Use a restricted policy similar to:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "route53:GetChange",
      "Resource": "arn:aws:route53:::change/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets",
        "route53:ListResourceRecordSets"
      ],
      "Resource": "arn:aws:route53:::hostedzone/<HOSTED_ZONE_ID>",
      "Condition": {
        "ForAllValues:StringEquals": {
          "route53:ChangeResourceRecordSetsRecordTypes": [
            "TXT"
          ]
        }
      }
    }
  ]
}
```

Further restrict the policy according to your organization's security requirements.

---

## Create a production ClusterIssuer

Example:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: <EMAIL>
    server: https://acme-v02.api.letsencrypt.org/directory

    privateKeySecretRef:
      name: letsencrypt-prod-account-key

    solvers:
      - dns01:
          route53:
            region: <AWS_REGION>
            hostedZoneID: <HOSTED_ZONE_ID>
```

Apply:

```bash
kubectl apply -f clusterissuer.yaml
```

Verify:

```bash
kubectl get clusterissuer
```

You want:

```text
READY   True
```

---

## Create the Certificate

Example:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: application-tls
  namespace: <APPLICATION_NAMESPACE>

spec:
  secretName: application-tls

  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer

  dnsNames:
    - <DOMAIN>
    - "*.<DOMAIN>"
```

Apply:

```bash
kubectl apply -f certificate.yaml
```

Check:

```bash
kubectl get certificate -n <APPLICATION_NAMESPACE>
```

Then:

```bash
kubectl describe certificate application-tls \
  -n <APPLICATION_NAMESPACE>
```

The resulting Secret should be:

```text
application-tls
```

Verify:

```bash
kubectl get secret application-tls \
  -n <APPLICATION_NAMESPACE>
```

The Secret should have:

```text
type: kubernetes.io/tls
```

---

# 10. Phase 3 — Install Envoy Gateway

Install Envoy Gateway alongside NGINX.

Do **not** uninstall NGINX yet.

Example:

```bash
helm install envoy-gateway \
  oci://docker.io/envoyproxy/gateway-helm \
  --version <ENVOY_GATEWAY_VERSION> \
  -n envoy-gateway-system \
  --create-namespace
```

Verify:

```bash
helm list -n envoy-gateway-system
```

Then:

```bash
kubectl get pods -n envoy-gateway-system
```

You should see the Envoy Gateway controller.

Example:

```text
envoy-gateway-xxxxx   1/1   Running
```

At this point:

```text
NGINX                         Envoy Gateway
  |                                |
  v                                v
Existing traffic              No application traffic yet
```

Nothing has been migrated yet.

---

# 11. Phase 4 — Create GatewayClass

Create:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg

spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

Apply:

```bash
kubectl apply -f gatewayclass.yaml
```

Verify:

```bash
kubectl get gatewayclass
```

Expected:

```text
NAME   CONTROLLER                                      ACCEPTED
eg     gateway.envoyproxy.io/gatewayclass-controller   True
```

Inspect:

```bash
kubectl describe gatewayclass eg
```

You want:

```text
Accepted: True
```

Conceptually:

```text
GatewayClass
     |
     | selects
     v
Envoy Gateway controller
```

The Envoy Gateway controller uses:

```text
gateway.envoyproxy.io/gatewayclass-controller
```

as its controller name by default.

---

# 12. Phase 5 — Create Gateway

Create a Gateway in the namespace containing the application and TLS Secret.

Example:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway

metadata:
  name: application-gateway
  namespace: <APPLICATION_NAMESPACE>

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

Apply:

```bash
kubectl apply -f gateway.yaml
```

Check:

```bash
kubectl get gateway -n <APPLICATION_NAMESPACE>
```

Initially you may see:

```text
PROGRAMMED   False
```

while Envoy Gateway creates its data-plane infrastructure.

Eventually:

```text
PROGRAMMED   True
```

Check:

```bash
kubectl describe gateway application-gateway \
  -n <APPLICATION_NAMESPACE>
```

You should eventually see:

```text
Accepted: True
Programmed: True
```

and an external address such as:

```text
<ENVoy_ELB_DNS_NAME>
```

---

# 13. Phase 6 — Create HTTPRoutes

Gateway API separates the Gateway from application routing.

Instead of one large Ingress object, create HTTPRoutes describing application traffic.

For example:

## Desktop

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute

metadata:
  name: desktop-route
  namespace: <APPLICATION_NAMESPACE>

spec:
  parentRefs:
    - name: application-gateway

  hostnames:
    - <DOMAIN>

  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /

      backendRefs:
        - name: desktop-svc
          port: 80
```

---

## Android

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute

metadata:
  name: android-route
  namespace: <APPLICATION_NAMESPACE>

spec:
  parentRefs:
    - name: application-gateway

  hostnames:
    - android.<DOMAIN>

  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /

      backendRefs:
        - name: android-svc
          port: 80
```

---

## iPhone

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute

metadata:
  name: iphone-route
  namespace: <APPLICATION_NAMESPACE>

spec:
  parentRefs:
    - name: application-gateway

  hostnames:
    - iphone.<DOMAIN>

  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /

      backendRefs:
        - name: iphone-svc
          port: 80
```

Apply:

```bash
kubectl apply -f httproute.yaml
```

Verify:

```bash
kubectl get httproute \
  -n <APPLICATION_NAMESPACE>
```

Then:

```bash
kubectl describe httproute desktop-route \
  -n <APPLICATION_NAMESPACE>
```

You want:

```text
Accepted: True
ResolvedRefs: True
```

The HTTPRoute API uses `parentRefs`, `hostnames`, matching rules, and `backendRefs` to define how HTTP requests are routed to Services.

---

# 14. Phase 7 — Test Envoy Before DNS Cutover

This is one of the most important migration steps.

Do **not** immediately change Route 53.

First obtain the Envoy Load Balancer DNS name:

```bash
kubectl get gateway \
  -n <APPLICATION_NAMESPACE>
```

For example:

```text
application-gateway
<ENVOY_ELB_DNS_NAME>
```

Resolve it:

```bash
dig +short <ENVOY_ELB_DNS_NAME>
```

Example result:

```text
203.0.113.10
```

Use the returned IP to test the new endpoint while preserving the real hostname.

`curl --resolve` expects:

```text
hostname:port:IP
```

not another DNS hostname.

Correct:

```bash
curl --resolve <DOMAIN>:443:<ENVOY_IP> \
  https://<DOMAIN>/
```

Test Android:

```bash
curl --resolve android.<DOMAIN>:443:<ENVOY_IP> \
  https://android.<DOMAIN>/
```

Test iPhone:

```bash
curl --resolve iphone.<DOMAIN>:443:<ENVOY_IP> \
  https://iphone.<DOMAIN>/
```

This is useful because:

```text
URL hostname
     |
     +--> remains <DOMAIN>
     |
     +--> SNI remains <DOMAIN>
     |
     +--> Host header remains <DOMAIN>
     
Connection destination
     |
     +--> forced to Envoy IP
```

Therefore you can test Envoy without changing public DNS.

---

# 15. Phase 8 — DNS Cutover

Once Envoy works correctly, change DNS.

Before:

```text
<DOMAIN>
      |
      v
NGINX Load Balancer
```

After:

```text
<DOMAIN>
      |
      v
Envoy Load Balancer
```

For Route 53, an Alias A record is recommended when pointing the domain at an AWS load balancer.

Example:

```text
Name:
<DOMAIN>

Type:
A

Alias:
Yes

Target:
dualstack.<ENVOY_ELB_DNS_NAME>
```

For:

```text
android.<DOMAIN>
```

point it to the same Envoy Load Balancer.

For:

```text
iphone.<DOMAIN>
```

point it to the same Envoy Load Balancer.

The resulting DNS structure:

```text
                         Route 53
                            |
             +--------------+--------------+
             |              |              |
             v              v              v
         <DOMAIN>       android.*       iphone.*
             |              |              |
             +--------------+--------------+
                            |
                            v
                     Envoy Load Balancer
```

---

# 16. Phase 9 — Verify Production Traffic

First verify DNS:

```bash
dig +short <DOMAIN>
```

```bash
dig +short android.<DOMAIN>
```

```bash
dig +short iphone.<DOMAIN>
```

Then test normally:

```bash
curl https://<DOMAIN>/
```

```bash
curl https://android.<DOMAIN>/
```

```bash
curl https://iphone.<DOMAIN>/
```

Do **not** use `--resolve` for this test.

At this point:

```text
curl
  |
  v
DNS
  |
  v
Envoy Load Balancer
  |
  v
Envoy
  |
  v
HTTPRoute
  |
  v
Service
  |
  v
Pod
```

This is the real end-to-end test.

---

# 17. Phase 10 — Remove NGINX

Only remove NGINX after:

* Envoy is healthy
* Gateway is `Programmed=True`
* HTTPRoutes are `Accepted=True`
* Backend references are resolved
* TLS works
* DNS points to Envoy
* Application traffic works
* Monitoring shows no unexpected errors
* Rollback is still possible

Check NGINX:

```bash
helm list -n kube-system
```

Then:

```bash
kubectl get pods -n kube-system | grep ingress-nginx
```

and:

```bash
kubectl get svc -n kube-system | grep ingress-nginx
```

If the old release is:

```text
ingress-nginx
```

remove it:

```bash
helm uninstall ingress-nginx \
  -n kube-system
```

Verify:

```bash
kubectl get pods -n kube-system | grep ingress-nginx
```

No output should be returned.

Verify:

```bash
kubectl get svc -n kube-system | grep ingress-nginx
```

Again, no output should be returned.

---

# 18. Rollback Strategy

Never perform a production migration without a rollback plan.

During the migration:

```text
                 Route 53
                    |
                    v
                  Envoy
                    |
                 New path
```

Keep the old NGINX infrastructure alive temporarily.

If Envoy fails:

```text
Route 53
   |
   v
NGINX Load Balancer
   |
   v
Applications
```

To rollback:

1. Change Route 53 records back to the NGINX Load Balancer.
2. Wait for DNS propagation/caching.
3. Verify application traffic.
4. Investigate Envoy separately.
5. Do not delete the old path until the migration is proven stable.

This is much safer than:

```text
Delete NGINX
    |
    v
Install Envoy
    |
    v
Hope everything works
```

---

# 19. Understanding TLS Termination

TLS termination occurs at the Gateway/Envoy layer.

The flow is:

```text
Client
  |
  | HTTPS :443
  |
  v
AWS Load Balancer
  |
  v
Envoy
  |
  | TLS termination
  |
  v
HTTPRoute
  |
  v
Service
  |
  v
Pod
```

The Gateway contains:

```yaml
listeners:
  - name: https
    protocol: HTTPS
    port: 443

    tls:
      mode: Terminate

      certificateRefs:
        - kind: Secret
          name: application-tls
```

The Kubernetes Secret contains:

```text
tls.crt
tls.key
```

The client sees:

```text
HTTPS
```

while Envoy can forward HTTP traffic internally.

The certificate is selected using the requested hostname/SNI.

For example:

```text
SNI: android.<DOMAIN>
       |
       v
TLS certificate
       |
       v
application-tls
```

If the certificate contains:

```text
<DOMAIN>
*.<DOMAIN>
```

then it can cover both the root domain and subdomains.

---

# 20. Understanding Gateway API Objects

A useful mental model:

```text
GatewayClass
    |
    | "Which implementation?"
    |
    v
Gateway
    |
    | "Where does traffic enter?"
    |
    v
HTTPRoute
    |
    | "How should traffic be routed?"
    |
    v
Service
    |
    | "Which application?"
    |
    v
Pod
```

Another way to remember it:

| Resource     | Main question                            |
| ------------ | ---------------------------------------- |
| GatewayClass | Who manages the Gateway?                 |
| Gateway      | Where does traffic enter?                |
| HTTPRoute    | How is traffic routed?                   |
| Service      | Which workload receives traffic?         |
| Pod          | Where does the application actually run? |

---

# 21. Gateway API vs Ingress

## Ingress

```yaml
kind: Ingress
```

Often contains:

* hosts
* paths
* TLS
* backend services
* controller-specific annotations

Example:

```text
Ingress
 ├── TLS
 ├── Host
 ├── Path
 └── Backend
```

---

## Gateway API

Separates these responsibilities:

```text
GatewayClass
      |
      v
Gateway
      |
      v
HTTPRoute
      |
      v
Service
```

This makes it possible to establish clearer ownership.

For example:

```text
Platform Team
      |
      v
GatewayClass
      |
      v
Gateway
      |
      |
      +----------------------+
                             |
Application Team             |
      |                      |
      v                      |
HTTPRoute -------------------+
      |
      v
Service
```

This separation is one of the major architectural advantages of Gateway API.

---

# 22. Weighted Traffic Splitting

One of the most useful Gateway API features is traffic splitting.

Suppose we have two versions:

```text
desktop-v1
desktop-v2
```

and Services:

```text
desktop-v1-svc
desktop-v2-svc
```

We can send:

```text
90% → v1
10% → v2
```

using:

```yaml
backendRefs:

  - name: desktop-v1-svc
    port: 80
    weight: 90

  - name: desktop-v2-svc
    port: 80
    weight: 10
```

Complete example:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute

metadata:
  name: desktop-canary
  namespace: <APPLICATION_NAMESPACE>

spec:
  parentRefs:
    - name: application-gateway

  hostnames:
    - <DOMAIN>

  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /

      backendRefs:

        - name: desktop-v1-svc
          port: 80
          weight: 90

        - name: desktop-v2-svc
          port: 80
          weight: 10
```

The important point:

> `weight` is not required to add up to 100.

The percentage is calculated as:

```text
backend weight
----------------------------- × 100
sum of all backend weights
```

Therefore:

```text
90 + 10 = 100

v1 = 90 / 100 = 90%
v2 = 10 / 100 = 10%
```

But this is equally valid:

```yaml
weight: 9
```

and:

```yaml
weight: 1
```

because:

```text
9 / (9 + 1) = 90%
1 / (9 + 1) = 10%
```

Envoy Gateway documents weighted `backendRefs` specifically for this traffic-splitting use case.

---

# 23. Canary Deployment

Canary deployment means introducing a new version to only a portion of users.

Example:

```text
                 Incoming traffic
                       |
                       v
                  Envoy Gateway
                       |
                 HTTPRoute
                       |
              +--------+--------+
              |                 |
              v                 v
           v1 Service        v2 Service
              |                 |
              v                 v
          v1 Pods            v2 Pods

             90%                10%
```

This allows us to test the new version with limited traffic.

For example:

```text
v1 = stable
v2 = new version
```

Start:

```text
v1 = 100%
v2 = 0%
```

Then:

```text
v1 = 90%
v2 = 10%
```

Then:

```text
v1 = 75%
v2 = 25%
```

Then:

```text
v1 = 50%
v2 = 50%
```

Then:

```text
v1 = 25%
v2 = 75%
```

Finally:

```text
v1 = 0%
v2 = 100%
```

---

# 24. Progressive Canary Rollout

A production rollout should normally be gradual.

Example:

## Stage 1

```yaml
backendRefs:
  - name: application-v1
    port: 80
    weight: 99

  - name: application-v2
    port: 80
    weight: 1
```

Deploy:

```text
99% v1
1%  v2
```

Monitor:

* HTTP 5xx
* latency
* CPU
* memory
* application errors
* logs
* business metrics

---

## Stage 2

Change to:

```yaml
weight: 95
```

and:

```yaml
weight: 5
```

Result:

```text
95% v1
5%  v2
```

---

## Stage 3

```text
90% v1
10% v2
```

---

## Stage 4

```text
75% v1
25% v2
```

---

## Stage 5

```text
50% v1
50% v2
```

---

## Stage 6

```text
25% v1
75% v2
```

---

## Final

```text
0% v1
100% v2
```

At that point:

1. Scale down v1.
2. Verify v2.
3. Remove the old Deployment/Service when appropriate.

---

# 25. Traffic Mirroring

Weighted traffic splitting is not the only useful feature.

Gateway API can also mirror requests to another backend.

The important difference:

```text
Traffic splitting
```

means:

```text
90% → v1
10% → v2
```

The client receives the response from whichever backend handled the request.

With mirroring:

```text
100% → v1
         |
         +---- copy/mirror ----> v2
```

The response still comes from v1.

The mirrored backend's response is ignored.

This can be useful for testing a new application version with real traffic without exposing users to the new version. Envoy Gateway supports HTTP request mirroring through HTTPRoute filters.

---

# 26. Production Checklist

Before migration:

* [ ] Backup existing manifests
* [ ] Document current NGINX configuration
* [ ] Document all annotations
* [ ] Document TLS configuration
* [ ] Document DNS records
* [ ] Document current Load Balancer
* [ ] Confirm application health
* [ ] Confirm rollback procedure
* [ ] Verify monitoring
* [ ] Verify logging
* [ ] Verify DNS ownership

During migration:

* [ ] Install Gateway API implementation
* [ ] Keep NGINX running
* [ ] Create GatewayClass
* [ ] Create Gateway
* [ ] Configure TLS
* [ ] Create HTTPRoutes
* [ ] Verify `Accepted=True`
* [ ] Verify `ResolvedRefs=True`
* [ ] Verify `Programmed=True`
* [ ] Test Envoy Load Balancer directly
* [ ] Test HTTPS
* [ ] Test every hostname
* [ ] Test every path
* [ ] Test application functionality
* [ ] Change DNS
* [ ] Monitor traffic

After migration:

* [ ] Verify DNS
* [ ] Verify HTTPS
* [ ] Verify certificates
* [ ] Verify application health
* [ ] Verify metrics
* [ ] Verify logs
* [ ] Verify error rates
* [ ] Keep NGINX temporarily for rollback
* [ ] Remove NGINX after stabilization
* [ ] Remove obsolete Ingress resources
* [ ] Update documentation
* [ ] Update monitoring/alerts

---

# 27. Troubleshooting

## GatewayClass not accepted

Check:

```bash
kubectl describe gatewayclass <GATEWAYCLASS_NAME>
```

Look for:

```text
Accepted: True
```

If not accepted, verify the controller name:

```yaml
controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

---

## Gateway not programmed

Run:

```bash
kubectl describe gateway \
  -n <APPLICATION_NAMESPACE>
```

Look at:

```text
Accepted
Programmed
```

Also check:

```bash
kubectl get pods \
  -n envoy-gateway-system
```

and:

```bash
kubectl logs \
  -n envoy-gateway-system \
  deployment/envoy-gateway
```

---

## HTTPRoute not accepted

Run:

```bash
kubectl describe httproute \
  -n <APPLICATION_NAMESPACE> \
  <ROUTE_NAME>
```

Check:

```text
Accepted: True
ResolvedRefs: True
```

Common causes:

* Wrong Gateway name
* Wrong namespace
* Listener restrictions
* Incorrect hostname
* Service does not exist
* Wrong Service port
* Cross-namespace reference without permission

---

## TLS not working

Check the Secret:

```bash
kubectl get secret \
  -n <APPLICATION_NAMESPACE>
```

Then:

```bash
kubectl describe gateway \
  -n <APPLICATION_NAMESPACE>
```

Verify the HTTPS listener:

```yaml
protocol: HTTPS
port: 443

tls:
  mode: Terminate

  certificateRefs:
    - kind: Secret
      name: application-tls
```

Also check:

```bash
kubectl get certificate \
  -n <APPLICATION_NAMESPACE>
```

and:

```bash
kubectl describe certificate \
  -n <APPLICATION_NAMESPACE>
```

---

## Route works on one hostname but not another

Check:

```bash
kubectl get httproute \
  -n <APPLICATION_NAMESPACE>
```

Verify:

```yaml
hostnames:
  - <EXPECTED_HOSTNAME>
```

Remember that these are different:

```text
example.com
www.example.com
api.example.com
```

The TLS certificate and HTTPRoute hostname configuration must cover the intended hostnames.

---

## DNS still reaches NGINX

Check:

```bash
dig +short <DOMAIN>
```

Then inspect Route 53.

Make sure the Alias target is the new Envoy Load Balancer.

Also remember DNS caching can cause clients to continue using previous answers for some time.

---

## `curl --resolve` fails

Incorrect:

```bash
curl --resolve example.com:443:load-balancer.amazonaws.com \
  https://example.com
```

Correct:

```bash
curl --resolve example.com:443:203.0.113.10 \
  https://example.com
```

`--resolve` requires an IP address.

---

# 28. Useful Commands

## Kubernetes

```bash
kubectl get nodes
```

```bash
kubectl get pods -A
```

```bash
kubectl get svc -A
```

```bash
kubectl get gatewayclass
```

```bash
kubectl get gateway -A
```

```bash
kubectl get httproute -A
```

---

## Gateway details

```bash
kubectl describe gateway \
  -n <APPLICATION_NAMESPACE> \
  <GATEWAY_NAME>
```

```bash
kubectl describe gatewayclass \
  <GATEWAYCLASS_NAME>
```

```bash
kubectl describe httproute \
  -n <APPLICATION_NAMESPACE> \
  <ROUTE_NAME>
```

---

## Envoy

```bash
kubectl get pods \
  -n envoy-gateway-system
```

```bash
kubectl get svc \
  -n envoy-gateway-system
```

```bash
kubectl logs \
  -n envoy-gateway-system \
  deployment/envoy-gateway
```

---

## DNS

```bash
dig +short <DOMAIN>
```

```bash
dig +short <ENVOY_ELB_DNS_NAME>
```

```bash
dig NS <DOMAIN>
```

---

## HTTPS

```bash
curl https://<DOMAIN>/
```

```bash
curl https://android.<DOMAIN>/
```

```bash
curl https://iphone.<DOMAIN>/
```

Inspect the certificate:

```bash
openssl s_client \
  -connect <DOMAIN>:443 \
  -servername <DOMAIN>
```

---

# 29. Final Architecture

After migration and cleanup, the architecture becomes:

```text
                              INTERNET
                                  |
                                  |
                                  v
                           Route 53 DNS
                                  |
                                  |
                     +------------+------------+
                     |            |            |
                     v            v            v
                <DOMAIN>      android.*      iphone.*
                     |            |            |
                     +------------+------------+
                                  |
                                  v
                       AWS Load Balancer
                                  |
                                  v
                        Envoy Gateway
                                  |
                         HTTPS termination
                                  |
                                  v
                          Gateway API
                                  |
                           HTTPRoute
                                  |
                 +----------------+----------------+
                 |                |                |
                 v                v                v
          desktop-route    android-route    iphone-route
                 |                |                |
                 v                v                v
           desktop-svc      android-svc      iphone-svc
                 |                |                |
                 v                v                v
             Desktop Pods    Android Pods     iPhone Pods
```

The old NGINX path is gone:

```text
NGINX
   X
```

The application now uses:

```text
Route 53
   |
   v
AWS Load Balancer
   |
   v
Envoy Gateway
   |
   v
Gateway API
   |
   v
HTTPRoute
   |
   v
Service
   |
   v
Pod
```

---

# 30. References

## Kubernetes Gateway API

Kubernetes documentation:

* Gateway API overview
* Gateway API architecture
* Ingress migration

[Kubernetes Gateway API documentation](https://kubernetes.io/docs/concepts/services-networking/gateway/?utm_source=chatgpt.com)

Gateway API is the successor to the Kubernetes Ingress API, and Kubernetes documents a migration path from Ingress to Gateway API.

---

## Envoy Gateway

[Envoy Gateway documentation](https://gateway.envoyproxy.io/?utm_source=chatgpt.com)

Quickstart:

[Envoy Gateway Quickstart](https://gateway.envoyproxy.io/v1.6/tasks/quickstart/?utm_source=chatgpt.com)

Traffic splitting:

[Envoy Gateway HTTP traffic splitting](https://gateway.envoyproxy.io/latest/tasks/traffic/http-traffic-splitting/?utm_source=chatgpt.com)

TLS / secure gateways:

[Envoy Gateway Secure Gateways documentation](https://gateway.envoyproxy.io/docs/tasks/security/secure-gateways/?utm_source=chatgpt.com)

---

## cert-manager

[cert-manager documentation](https://cert-manager.io/docs/?utm_source=chatgpt.com)

DNS-01:

[cert-manager DNS-01 documentation](https://cert-manager.io/docs/configuration/acme/dns01/?utm_source=chatgpt.com)

Route 53:

[cert-manager Route 53 documentation](https://cert-manager.io/docs/configuration/acme/dns01/route53/?utm_source=chatgpt.com)

---

# Summary

The migration can be remembered as four major phases:

```text
1. PREPARE
   |
   +-- Keep NGINX
   +-- Install Envoy Gateway
   +-- Configure TLS
   +-- Create GatewayClass
   +-- Create Gateway
   +-- Create HTTPRoutes


2. TEST
   |
   +-- Test Envoy Load Balancer directly
   +-- Verify TLS
   +-- Verify routing
   +-- Verify applications


3. CUTOVER
   |
   +-- Change Route 53
   +-- DNS → Envoy Load Balancer
   +-- Monitor production traffic


4. CLEANUP
   |
   +-- Keep NGINX temporarily
   +-- Confirm stability
   +-- Remove NGINX
   +-- Remove obsolete Ingress resources
```

The key principle is:

> **Build the new traffic path before destroying the old one.**

And once Gateway API is established, advanced traffic-management patterns become much cleaner:

```text
                  Gateway
                     |
                  HTTPRoute
                     |
        +------------+-------------+
        |            |             |
        v            v             v
     Stable       Canary       Mirrored
       90%          10%         Traffic
```

Gateway API therefore provides a strong foundation for:

* Host-based routing
* Path-based routing
* TLS
* Traffic splitting
* Canary releases
* Request mirroring
* Header-based routing
* Advanced traffic management
* Platform/application separation

For production, always validate the exact capabilities, version compatibility, security model, and AWS load-balancer behavior of the Gateway implementation you choose before applying this runbook unchanged to a live environment.
