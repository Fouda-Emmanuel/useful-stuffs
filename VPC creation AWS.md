
# VPC Creation AWS


Best practice order:

1. VPC
2. Subnets
3. IGW
4. Route tables
5. NAT Gateway
6. EC2 / RDS / ALB

---

## One more REQUIRED thing for public EC2 (IMPORTANT)

For an EC2 to be **reachable from the internet**, it needs **ALL THREE**:

1. Public subnet (route table → IGW)
2. Public IP (or Elastic IP)
3. Security Group allowing inbound traffic (e.g. port 80 / 22)

Route table alone is **not enough**.

---

# 🧠 FINAL CLEAN VERSION (MEMORIZE THIS)

Here is the **perfect mental model**:

```
1) Create VPC (10.0.0.0/16)

2) Create subnets:
   - Public:  10.0.1.0/24
   - Private: 10.0.2.0/24

3) Create Internet Gateway
   - Attach to VPC

4) Create Public Route Table
   - 10.0.0.0/16 → local
   - 0.0.0.0/0   → IGW
   - Associate with Public Subnet

5) Create NAT Gateway
   - Create In Public Subnet
   - With Elastic IP

6) Create Private Route Table
   - 10.0.0.0/16 → local
   - 0.0.0.0/0   → NAT Gateway
   - Associate with Private Subnet

7) Launch EC2 instances
   - Public EC2 → has public IP
   - Private EC2 → no public IP
```

---

# 🔐 Where NACL Fits (just placing it mentally)

NACL is:

* Attached to **subnets**
* Checked **before** Security Groups

Traffic path:

```
Internet
 → IGW
 → Route Table
 → NACL
 → Security Group
 → EC2
```

(Default NACL allows all — so no problem.)

---
