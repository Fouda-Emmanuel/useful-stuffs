# 🌐 Complete DNS Management Guide: Spaceship to Route 53

A comprehensive step-by-step guide for setting up DNS when your domain is registered with **Spaceship** and you want to use **Amazon Route 53** as your DNS provider.

---

## 📋 Table of Contents

1. [Core Architecture](#core-architecture)
2. [Understanding the Components](#understanding-the-components)
3. [Step 1: Create Route 53 Hosted Zone](#step-1-create-route-53-hosted-zone)
4. [Step 2: Delegate DNS Authority (Nameserver Update)](#step-2-delegate-dns-authority-nameserver-update)
5. [Step 3: Validate Domain Ownership (ACM Certificate)](#step-3-validate-domain-ownership-acm-certificate)
6. [Step 4: Create Application DNS Records](#step-4-create-application-dns-records)
7. [Step 5: Verify DNS Propagation](#step-5-verify-dns-propagation)
8. [Demo 2: Path-Based Routing](#demo-2-path-based-routing)
9. [Demo 3: Host-Based (Subdomain) Routing](#demo-3-host-based-subdomain-routing)
10. [Troubleshooting](#troubleshooting)
11. [Quick Reference](#quick-reference)

---

## Core Architecture

```text
                    🌍 INTERNET
                         │
                         │
                  example.com
                         │
                         ▼
                 ┌──────────────┐
                 │  SPACESHIP   │
                 │              │
                 │  Registrar   │  ← Domain ownership & registration
                 └──────┬───────┘
                        │
                        │ NS Delegation
                        │ "Route 53 manages DNS"
                        ▼
                 ┌──────────────┐
                 │   ROUTE 53   │
                 │              │
                 │ Public Zone  │  ← DNS record management
                 │ example.com  │
                 └──────┬───────┘
                        │
                        │ DNS Records
                        ▼
                 ┌──────────────┐
                 │     ACM      │
                 │              │
                 │ Certificate  │  ← TLS/HTTPS certificates
                 │   ISSUED ✅  │
                 └──────────────┘
                        │
                        │ Associated to
                        ▼
                 ┌──────────────┐
                 │   AWS ALB    │
                 │              │
                 │ Application  │  ← Your application load balancer
                 │ Load Balancer│
                 └──────────────┘
```

---

## Understanding the Components

### Registrar vs DNS Provider

This is the most important concept to understand:

| Component | Role | Example |
|-----------|------|---------|
| **Domain Registrar** | Where you **buy** the domain | Spaceship, GoDaddy, Namecheap |
| **DNS Provider** | Where you **manage DNS records** | Route 53, Cloudflare, Spaceship DNS |

### Why Use Route 53?

- **Seamless integration** with AWS services (ALB, ACM, CloudFront)
- **Alias records** for AWS resources (points directly to AWS services)
- **Automatic ACM validation** with one-click record creation
- **High availability** and low latency
- **Supports both IPv4 and IPv6** (dual-stack)

---

## Step 1: Create Route 53 Hosted Zone

### 1.1 Create the Hosted Zone

**AWS Console Path:**
```
Route 53 → Hosted zones → Create hosted zone
```

**Configuration:**

| Field | Value |
|-------|-------|
| Domain name | `your-domain.com` (e.g., `example.com`) |
| Type | **Public hosted zone** |

### 1.2 What Route 53 Creates

Route 53 automatically creates two records:

**1. NS (Nameserver) Record:**
```text
ns-649.awsdns-17.net.
ns-115.awsdns-14.com.
ns-1906.awsdns-46.co.uk.
ns-1187.awsdns-20.org.
```

**2. SOA (Start of Authority) Record:**
```text
ns-649.awsdns-17.net. awsdns-hostmaster.amazon.com. 1 7200 900 1209600 86400
```

### 1.3 📝 Save the Nameservers

**⚠️ CRITICAL:** Copy the 4 nameserver addresses. You'll need them in Step 2.

```bash
# Example of what you'll see (YOUR values will be different):
ns-649.awsdns-17.net
ns-115.awsdns-14.com
ns-1906.awsdns-46.co.uk
ns-1187.awsdns-20.org
```

---

## Step 2: Delegate DNS Authority (Nameserver Update)

### 2.1 Navigate to Nameservers in Spaceship

**Spaceship Path:**
```
Spaceship Dashboard → Domain Manager → [YOUR DOMAIN] → Nameservers → Change
```

### 2.2 Select Custom Nameservers

Choose **Custom nameservers** (NOT `Spaceship nameservers`).

### 2.3 Enter Route 53 Nameservers

Replace the default nameservers with the **4 Route 53 nameservers**:

```text
⚠️ IMPORTANT: Remove the trailing dots (.)
❌ DO NOT use: ns-649.awsdns-17.net.  ← with trailing dot
✅ USE:         ns-649.awsdns-17.net    ← no trailing dot
```

**Example Configuration:**

```text
Custom nameservers
┌────────────────────────────────────────────────┐
│ 1. ns-649.awsdns-17.net                     │
│ 2. ns-115.awsdns-14.com                     │
│ 3. ns-1906.awsdns-46.co.uk                  │
│ 4. ns-1187.awsdns-20.org                    │
└────────────────────────────────────────────────┘
```

### 2.4 Save Changes

Click **Save** or **Update**.

### 2.5 Verify the Change

After saving, the Nameservers section should show:

```text
Nameservers: Custom nameservers
Managed with: Custom DNS
```

**Note:** DNS propagation can take **up to 48 hours**, but typically propagates within **15-30 minutes**.

---

## Step 3: Validate Domain Ownership (ACM Certificate)

### 3.1 Why ACM Validation is Needed

AWS Certificate Manager (ACM) needs to verify that you **actually own the domain** before issuing an SSL/TLS certificate. This prevents security issues.

### 3.2 Request an ACM Certificate

**AWS Console Path:**
```
ACM → Request certificate → Request a public certificate
```

#### Option A: Single Domain (Demo 1/2)

```text
Domain names:
- example.com

Validation method: DNS
Key algorithm: RSA 2048
```

#### Option B: Wildcard Certificate (Demo 3)

```text
Domain names:
- example.com
- *.example.com

Validation method: DNS
Key algorithm: RSA 2048
```

### 3.3 Understanding Wildcard Certificates

```text
*.example.com
    │
    ├── Covers: api.example.com ✅
    ├── Covers: app.example.com ✅
    ├── Covers: www.example.com ✅
    ├── Covers: any-subdomain.example.com ✅
    │
    └── Does NOT cover: example.com ❌ (must add explicitly)
```

**Why both?**

- `*.example.com` covers ALL subdomains
- `example.com` covers the root/apex domain
- **You need BOTH for full coverage**

### 3.4 Create Validation Records

#### Option 1: Automatic (Recommended)

In ACM, click **Create records in Route 53**. ACM will:
1. Detect your hosted zone
2. Automatically create the CNAME validation record
3. No manual typing required!

#### Option 2: Manual

**What ACM gives you:**
```text
CNAME name:  _0a09b94ee3002a3556a82985564e4c78.example.com.
CNAME value: _5dad3c6ddc7ac9b4330ccd54893e8277.jkddzztszm.acm-validations.aws.
```

**What you create in Route 53:**
```text
Record name: _0a09b94ee3002a3556a82985564e4c78.example.com
Record type: CNAME
Record value: _5dad3c6ddc7ac9b4330ccd54893e8277.jkddzztszm.acm-validations.aws.
```

### 3.5 Certificate Status Flow

```text
Pending validation
        │
        │ ACM periodically checks DNS
        ▼
     Issued ✅
```

**Important:** If you already have a validation record from a previous certificate, a new certificate may validate immediately.

---

## Step 4: Create Application DNS Records

### 4.1 Understanding Alias Records vs CNAME

This is a **critical distinction**:

| Record Type | Use Case | Limitations |
|-------------|----------|-------------|
| **CNAME** | Point a domain to another DNS name | ❌ Cannot be used at the zone apex (root domain) |
| **Alias A** (Route 53) | Point the root domain to AWS resources | ✅ Route 53-only feature |
| **Alias AAAA** (Route 53) | Point IPv6 traffic to AWS resources | ✅ Route 53-only feature |

### 4.2 Why CNAME Doesn't Work at the Root

```text
DNS Zone: example.com
Root: example.com

❌ CNAME at root is NOT allowed because:
┌─────────────────────────────────────────────────────────┐
│ example.com  IN  CNAME  alb-xxxx.elb.amazonaws.com    │
│                                                       │
│ Problem: The root also needs SOA and NS records.     │
│ CNAME cannot coexist with other record types.        │
└─────────────────────────────────────────────────────────┘

✅ Alias A at root IS allowed because:
┌─────────────────────────────────────────────────────────┐
│ example.com  IN  A  ALIAS  alb-xxxx.elb.amazonaws.com │
│                                                       │
│ Route 53 handles this specially.                      │
│ The A record resolves to the ALB's IP addresses.      │
└─────────────────────────────────────────────────────────┘
```

### 4.3 Creating Records for Single Domain (Demo 2)

**In Route 53 Hosted Zone:**

```text
Record name: [blank]  ← Creates record for root domain
Record type: A
Alias: Yes
Route traffic to: Alias to Application and Classic Load Balancers
Region: YOUR_AWS_REGION (e.g., us-east-1)
Load balancer: YOUR_ALB_NAME
Evaluate target health: Yes
```

**Result:**
```text
example.com → ALB
```

### 4.4 Creating Records for Subdomains (Demo 3)

**Create three Alias A records:**

```text
Record 1 (Root):
  Record name: [blank]
  Record type: A (Alias)
  Target: YOUR_ALB

Record 2 (iPhone):
  Record name: iphone
  Record type: A (Alias)
  Target: YOUR_ALB

Record 3 (Android):
  Record name: android
  Record type: A (Alias)
  Target: YOUR_ALB
```

**Result:**
```text
example.com → ALB
iphone.example.com → ALB
android.example.com → ALB
```

### 4.5 What's `dualstack` in the ALB DNS Name?

When you select an Application Load Balancer, Route 53 shows:

```text
dualstack.YOUR_ALB_NAME.elb.amazonaws.com
```

This is **not a different ALB**:
- `dualstack` indicates the ALB supports **both IPv4 and IPv6**
- It's the **same load balancer**, just with a dual-stack DNS name
- Route 53 automatically selects this for Alias records

```text
YOUR_ALB_NAME.elb.amazonaws.com
                │
                │ Same ALB
                ▼
dualstack.YOUR_ALB_NAME.elb.amazonaws.com
```

---

## Step 5: Verify DNS Propagation

### 5.1 Check Nameserver Delegation

```bash
dig NS example.com @8.8.8.8
```

**Expected output:**
```text
;; ANSWER SECTION:
example.com.  172800  IN  NS  ns-115.awsdns-14.com.
example.com.  172800  IN  NS  ns-1187.awsdns-20.org.
example.com.  172800  IN  NS  ns-1906.awsdns-46.co.uk.
example.com.  172800  IN  NS  ns-649.awsdns-17.net.
```

### 5.2 Check A Record Resolution

```bash
dig A example.com @8.8.8.8
```

**Expected output:**
```text
;; ANSWER SECTION:
example.com.  60  IN  A  54.152.25.199    # ALB IP address
example.com.  60  IN  A  3.223.119.71     # ALB IP address
```

### 5.3 Check Subdomain Resolution

```bash
dig A iphone.example.com @8.8.8.8
```

### 5.4 Test HTTP Redirect to HTTPS

```bash
curl -I http://example.com
```

**Expected output:**
```text
HTTP/1.1 301 Moved Permanently
Location: https://example.com:443/
Server: awselb/2.0
```

### 5.5 Test HTTPS End-to-End

```bash
curl -I https://example.com
curl -I https://iphone.example.com
curl -I https://android.example.com
```

---

## Demo 2: Path-Based Routing

### Architecture

```text
                    Internet
                       │
                       ▼
              ┌─────────────────┐
              │    Route 53     │
              │                 │
              │ example.com ────┼──────┐
              └─────────────────┘      │
                                       ▼
                              ┌─────────────────┐
                              │   AWS ALB       │
                              │ HTTPS :443      │
                              │ ACM Cert        │
                              └────────┬────────┘
                                       │
                              Path-based routing
                         ┌─────────────┼─────────────┐
                         │             │             │
                   /iphone       /android        /
                         │             │             │
                         ▼             ▼             ▼
                   iphone-svc    android-svc    desktop-svc
                         │             │             │
                         ▼             ▼             ▼
                       Pods          Pods          Pods
```

### Ingress Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo2-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:ACCOUNT_ID:certificate/CERTIFICATE_ID
spec:
  ingressClassName: alb
  rules:
    - host: example.com
      http:
        paths:
          - path: /iphone
            pathType: Prefix
            backend:
              service:
                name: iphone-svc
                port:
                  number: 80
          - path: /android
            pathType: Prefix
            backend:
              service:
                name: android-svc
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: desktop-svc
                port:
                  number: 80
```

### DNS Records (Demo 2)

| Record Name | Type | Target |
|-------------|------|--------|
| `example.com` | A (Alias) | ALB DNS name |
| `_0a09...example.com` | CNAME | ACM validation |

---

## Demo 3: Host-Based (Subdomain) Routing

### Architecture

```text
                    Internet
                       │
                       ▼
              ┌─────────────────┐
              │    Route 53     │
              │                 │
              │ example.com ────┼──────┐
              │ iphone... ──────┼──┐   │
              │ android... ─────┼┐ │   │
              └─────────────────┘│ │   │
                                 │ │   │
                                 ▼ ▼   ▼
                         ┌─────────────────┐
                         │   AWS ALB       │
                         │ HTTPS :443      │
                         │ Wildcard Cert   │
                         │ *.example.com   │
                         └────────┬────────┘
                                  │
                         Host-based routing
                    ┌─────────────┼─────────────┐
                    ▼             ▼             ▼
              example.com    iphone...    android...
                    │             │             │
                    ▼             ▼             ▼
              desktop-svc    iphone-svc    android-svc
                    │             │             │
                    ▼             ▼             ▼
                  Pods          Pods          Pods
```

### Wildcard Certificate

```text
Certificate contains:
┌─────────────────────────────────────────────┐
│ 1. example.com                             │
│ 2. *.example.com                           │
│    ├── Covers: iphone.example.com          │
│    ├── Covers: android.example.com         │
│    └── Covers: any-subdomain.example.com   │
└─────────────────────────────────────────────┘
```

### Ingress Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo3-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS": 443}, {"HTTP": 80}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:ACCOUNT_ID:certificate/WILDCARD_CERT_ID
spec:
  ingressClassName: alb
  rules:
    - host: iphone.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: iphone-svc
                port:
                  number: 80
    - host: android.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: android-svc
                port:
                  number: 80
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: desktop-svc
                port:
                  number: 80
```

### DNS Records (Demo 3)

| Record Name | Type | Target |
|-------------|------|--------|
| `example.com` | A (Alias) | ALB DNS name |
| `iphone.example.com` | A (Alias) | ALB DNS name |
| `android.example.com` | A (Alias) | ALB DNS name |
| `_0a09...example.com` | CNAME | ACM validation |

---

## DNS Records Summary

### Complete Route 53 Record Set (Demo 3)

| Record Name | Type | Purpose | Target |
|-------------|------|---------|--------|
| `example.com` | NS | Zone delegation | Route 53 nameservers |
| `example.com` | SOA | Zone administration | Route 53 SOA record |
| `_0a09...example.com` | CNAME | ACM validation | `...acm-validations.aws` |
| `example.com` | A (Alias) | Application root | ALB DNS name |
| `iphone.example.com` | A (Alias) | iPhone application | ALB DNS name |
| `android.example.com` | A (Alias) | Android application | ALB DNS name |

### Record Types Cheat Sheet

| Record Type | Purpose | Use Case |
|-------------|---------|----------|
| **NS** | Nameservers | Delegating DNS authority |
| **SOA** | Start of Authority | Zone administrative info |
| **A** | IPv4 Address | Point domain to an IP (or Alias) |
| **CNAME** | Canonical Name | Point domain to another domain |
| **TXT** | Text | Verification, SPF, DKIM |
| **MX** | Mail Exchange | Email routing |

---

## Troubleshooting

### Issue 1: `SERVFAIL` After Nameserver Change

**Symptom:**
```bash
dig NS example.com @8.8.8.8
;; status: SERVFAIL
```

**Cause:** DNS propagation delay
**Solution:** Wait 15-30 minutes and retry

### Issue 2: "Oops, something went wrong" at Spaceship

**Cause:** Trailing dot (`.`) at the end of nameservers
**Solution:** Remove trailing dots from all 4 nameservers

```text
❌ ns-649.awsdns-17.net.
✅ ns-649.awsdns-17.net
```

### Issue 3: Certificate Stuck on "Pending Validation"

**Check:**
```bash
dig CNAME _0a09...example.com @8.8.8.8
```

**If no response:** The validation record isn't in Route 53
**Solution:** Click **Create records in Route 53** in ACM

### Issue 4: Subdomain Not Resolving

**Check:**
```bash
dig A iphone.example.com @8.8.8.8
```

**If no response:**
1. Verify Route 53 has the subdomain record
2. Wait for DNS propagation
3. Check if you used `A Alias` or `CNAME`

### Issue 5: HTTP → HTTPS Redirect Not Working

**Check Ingress annotation:**
```yaml
alb.ingress.kubernetes.io/ssl-redirect: '443'
```

**Check ALB listeners:**
```text
HTTP :80 → Redirect → HTTPS :443 ✅
```

### Issue 6: ALB Not Creating

**Check:**
```bash
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```

**Common causes:**
- Missing VPC ID in controller configuration
- Missing region in controller configuration
- Subnets not tagged with `kubernetes.io/role/elb=1`

---

## Quick Reference

### Essential Commands

```bash
# Check nameservers
dig NS example.com @8.8.8.8

# Check A record
dig A example.com @8.8.8.8

# Check CNAME
dig CNAME _0a09...example.com @8.8.8.8

# Test HTTP redirect
curl -I http://example.com

# Test HTTPS
curl -I https://example.com

# Test subdomain
curl -I https://iphone.example.com
```

### AWS CLI Commands

```bash
# Get VPC ID
aws eks describe-cluster --name YOUR_CLUSTER_NAME --query 'cluster.resourcesVpcConfig.vpcId' --output text

# Get cluster ARN (includes region)
aws eks describe-cluster --name YOUR_CLUSTER_NAME --query 'cluster.arn' --output text

# Get ACM certificate ARNs
aws acm list-certificates --region us-east-1

# Verify subnet tags for ALB discovery
aws ec2 describe-subnets --filters "Name=vpc-id,Values=YOUR_VPC_ID" --query 'Subnets[*].[SubnetId,Tags]' --output table
```

### Architecture Decision Flowchart

```text
Do you want to manage DNS yourself?
         │
         ▼
    Use Route 53?
    │         │
   Yes        No
    │         │
    ▼         ▼
Use Alias   Use CNAME
Records     Records
(Allowed at  (Not allowed
root domain) at root domain)

Do you need subdomains?
    │         │
   Yes        No
    │         │
    ▼         ▼
Wildcard   Single
Certificate Domain
(example.com +  Certificate
 *.example.com)
```

---

## Key Takeaways

### DNS Concepts

1. **Registrar ≠ DNS Provider:** You can buy a domain from one company and manage DNS with another
2. **Alias A vs CNAME:** Use Alias A at the root, CNAME for subdomains (prefer Alias for AWS resources)
3. **ACM Validation:** Route 53 makes SSL certificate validation automatic with one click
4. **Wildcard Certificates:** Cover all subdomains but NOT the apex domain
5. **DNS Propagation:** Changes take 15 minutes to 48 hours to propagate
6. **`dualstack`:** Indicates IPv4/IPv6 support, not a different ALB

### Production Recommendations

1. ✅ **Always use Alias records** when pointing to AWS resources
2. ✅ **Request wildcard certificates** for applications with subdomains
3. ✅ **Keep ACM validation records** for automatic certificate renewal
4. ✅ **Monitor DNS propagation** using tools like `dig @8.8.8.8`
5. ✅ **Document all records** in your infrastructure-as-code
6. ✅ **Tag subnets** with `kubernetes.io/role/elb=1` for ALB discovery
7. ✅ **Use separate certificates** for different environments (dev/staging/prod)

---

## Complete Setup Checklist

### Phase 1: DNS Setup
- [ ] Create Route 53 Public Hosted Zone
- [ ] Copy 4 nameservers from Route 53
- [ ] Change nameservers in Spaceship to custom
- [ ] Verify nameserver delegation with `dig NS`
- [ ] Request ACM certificate (single or wildcard)
- [ ] Create validation records in Route 53
- [ ] Wait for certificate status: **Issued ✅**

### Phase 2: Application Routing
- [ ] Deploy applications (Deployments + Services)
- [ ] Install AWS Load Balancer Controller
- [ ] Create Ingress with appropriate routing rules
- [ ] Verify ALB is created
- [ ] Create Route 53 Alias records to ALB
- [ ] Verify DNS resolution
- [ ] Test HTTPS endpoints

### Phase 3: Verification
- [ ] Test HTTP → HTTPS redirect
- [ ] Test path-based routing (Demo 2)
- [ ] Test host-based routing (Demo 3)
- [ ] Verify wildcard certificate covers all subdomains

---
