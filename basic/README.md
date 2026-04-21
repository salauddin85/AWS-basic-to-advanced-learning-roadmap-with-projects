# 🟢 AWS Training — Basic Level
### Intern DevOps Developer Roadmap 

---

> **Welcome, Intern!** This document is your complete guide to understanding AWS from the ground up.
> Each step builds on the previous one. Read carefully, take notes, and complete every assignment
> before moving to the next step. No shortcuts — fundamentals are everything in DevOps.

---

## 📋 Table of Contents

1. [What is Cloud Computing?](#step-1-what-is-cloud-computing)
2. [Introduction to AWS](#step-2-introduction-to-aws)
3. [AWS Global Infrastructure](#step-3-aws-global-infrastructure)
4. [VPC — Virtual Private Cloud](#step-4-vpc--virtual-private-cloud)
5. [Subnets](#step-5-subnets)
6. [Route Tables](#step-6-route-tables)
7. [Internet Gateway & NAT Gateway](#step-7-internet-gateway--nat-gateway)
8. [Security Groups & NACLs](#step-8-security-groups--nacls)
9. [EC2 — Elastic Compute Cloud](#step-9-ec2--elastic-compute-cloud)
10. [S3 — Simple Storage Service](#step-10-s3--simple-storage-service)
11. [IAM — Identity and Access Management](#step-11-iam--identity-and-access-management)
12. [AWS CLI Setup](#step-12-aws-cli-setup)

---

## Step 1: What is Cloud Computing?

### 🎯 Concept

Cloud computing means **renting IT resources** (servers, storage, networking, databases) over the internet
instead of owning physical hardware. You pay only for what you use — like paying for electricity
rather than owning a power plant.

### Three Service Models

| Model | Full Name | You Manage | Provider Manages | Example |
|-------|-----------|------------|-----------------|---------|
| **IaaS** | Infrastructure as a Service | OS, apps, data | Hardware, networking | AWS EC2 |
| **PaaS** | Platform as a Service | Apps, data | OS, runtime, hardware | AWS Elastic Beanstalk |
| **SaaS** | Software as a Service | Nothing (just use it) | Everything | Gmail, Salesforce |

### Three Deployment Models

- **Public Cloud** — Resources shared across many customers (AWS, Azure, GCP)
- **Private Cloud** — Dedicated infrastructure for one organization
- **Hybrid Cloud** — Mix of on-premises + public cloud

### Why Cloud? (Business Perspective)

- **No upfront capital cost** — Pay as you go
- **Scale instantly** — Add 1000 servers in minutes
- **Global reach** — Deploy worldwide in seconds
- **High availability** — Built-in redundancy
- **Focus on business** — Not on managing hardware

---

### 📝 Assignment 1 — Cloud Concepts

> **Objective:** Solidify your understanding of cloud fundamentals.

**Task 1:** In your own words (3–5 sentences), explain the difference between IaaS and PaaS.
Give a real-world analogy (e.g., renting a house vs. staying in a hotel).

**Task 2:** Your manager asks: "Why should we move our on-premises servers to AWS?"
Write a 1-page justification covering: cost, scalability, reliability, and security.

**Task 3:** Research and list 5 AWS competitors and one strength each has over AWS.

**Deliverable:** A written document (`assignment1.md`) with your answers.

---

## Step 2: Introduction to AWS

### 🎯 What is AWS?

**Amazon Web Services (AWS)** is the world's most comprehensive and broadly adopted cloud platform,
offering over **200 fully featured services** from data centers globally.

**AWS was launched in 2006.** Today it holds ~32% of the global cloud market.

### AWS Account Structure

```
AWS Organization (Root)
├── Management Account (billing, policies)
├── Dev Account
├── Staging Account
└── Production Account
```

**Best Practice:** Never use your root account for daily work. Always create IAM users.

### Core AWS Service Categories

| Category | Key Services |
|----------|-------------|
| **Compute** | EC2, Lambda, ECS, EKS |
| **Storage** | S3, EBS, EFS, Glacier |
| **Database** | RDS, DynamoDB, ElastiCache, Redshift |
| **Networking** | VPC, Route 53, CloudFront, ELB |
| **Security** | IAM, KMS, WAF, Shield, GuardDuty |
| **Monitoring** | CloudWatch, CloudTrail, X-Ray |
| **DevOps** | CodePipeline, CodeBuild, CodeDeploy |
| **Containers** | ECS, EKS, ECR, Fargate |

### AWS Free Tier

When you create an AWS account, you get **12 months of free tier** access:
- 750 hours/month of EC2 t2.micro
- 5 GB of S3 storage
- 25 GB of DynamoDB
- 1 million Lambda requests/month

> ⚠️ **WARNING:** Always set a billing alarm when learning AWS to avoid surprise charges!

### How to Create an AWS Account (Step-by-Step)

1. Go to [https://aws.amazon.com](https://aws.amazon.com)
2. Click **"Create an AWS Account"**
3. Enter email, password, account name
4. Provide payment information (required even for free tier)
5. Verify identity via phone
6. Select **Basic Support Plan** (free)
7. Sign in to the AWS Management Console

---

### 📝 Assignment 2 — AWS Account Setup

> **Objective:** Get hands-on with your first AWS account.

**Task 1:** Create a free-tier AWS account. Take a screenshot of your AWS Management Console home page.

**Task 2:** Navigate to **Billing → Budgets** and create a budget alert for $5.
This ensures you'll be notified before any accidental charges. Screenshot the budget.

**Task 3:** List 10 AWS services visible in the console. For each, write one sentence
describing what it does in your own words.

**Deliverable:** Screenshots + `assignment2.md` with service descriptions.

---

## Step 3: AWS Global Infrastructure

### 🎯 Why Global Infrastructure Matters

AWS infrastructure is designed for **high availability, low latency, and fault tolerance**.
Understanding this is critical before you deploy anything.

### Key Components

#### 🌍 Regions

A **Region** is a physical geographic area containing multiple data centers.

```
Examples:
- us-east-1      → N. Virginia, USA
- us-west-2      → Oregon, USA
- eu-west-1      → Ireland, Europe
- ap-southeast-1 → Singapore, Asia
- ap-south-1     → Mumbai, India
```

**How to choose a Region?**
1. **Latency** — Choose the region closest to your users
2. **Compliance** — Some data must stay in certain countries (GDPR in Europe)
3. **Cost** — Prices vary by region (us-east-1 is usually cheapest)
4. **Service availability** — Not all services are in every region

#### 🏢 Availability Zones (AZs)

An **Availability Zone** is one or more discrete data centers within a Region.
Each AZ has:
- Independent power
- Independent cooling
- Independent networking
- Physical separation from other AZs (miles apart)

```
us-east-1 Region
├── us-east-1a  (AZ 1 — Data Center A)
├── us-east-1b  (AZ 2 — Data Center B)
├── us-east-1c  (AZ 3 — Data Center C)
├── us-east-1d  (AZ 4 — Data Center D)
└── us-east-1f  (AZ 5 — Data Center F)
```

> **Real-world analogy:** A Region is a city. AZs are different neighborhoods in that city.
> If one neighborhood floods, others still work.

#### ⚡ Edge Locations

Edge locations are smaller AWS endpoints used by **CloudFront (CDN)** and **Route 53 (DNS)**
to cache content closer to users. There are 400+ edge locations worldwide.

### High Availability Design Principle

**Always deploy across multiple AZs.**

```
❌ BAD Design — Single AZ
   App Server → us-east-1a only

✅ GOOD Design — Multi-AZ
   Load Balancer
   ├── App Server → us-east-1a
   └── App Server → us-east-1b
```

---

### 📝 Assignment 3 — Global Infrastructure

**Task 1:** Go to the AWS console. List all available regions. Which region has the most
Availability Zones? (Hint: Check the EC2 console → AZ list)

**Task 2:** Your company is building an app for users in Bangladesh and India.
Which AWS region would you choose and why? Consider latency, compliance, and pricing.

**Task 3:** Draw a simple diagram (on paper or any tool) showing a Region with 3 AZs,
and how you'd deploy a web application for high availability.

**Deliverable:** `assignment3.md` with answers and a photo/screenshot of your diagram.

---

## Step 4: VPC — Virtual Private Cloud

### 🎯 What is a VPC?

A **VPC (Virtual Private Cloud)** is your own **isolated, private network** within AWS.

Think of AWS as a massive city. Your VPC is your **private building** inside that city.
You control who can enter, which rooms connect to which, and what goes in/out.

```
AWS Cloud (The City)
└── Your VPC (Your Private Building)
    ├── Public Subnet (Reception — accessible from outside)
    ├── Private Subnet (Office floor — internal only)
    └── Database Subnet (Vault — most restricted)
```

### Why Do We Need a VPC?

Without a VPC, all your resources would be exposed to the internet or completely uncontrolled.
A VPC gives you:

1. **Isolation** — Your resources are separated from other AWS customers
2. **Control** — You define IP ranges, subnets, routing, and firewalls
3. **Security** — Multi-layer network security
4. **Compliance** — Meet regulatory requirements for network isolation

### VPC Key Concepts

#### CIDR Block (IP Address Range)

When you create a VPC, you define an IP address range using **CIDR notation**.

```
Example: 10.0.0.0/16

This means:
- Network address: 10.0.0.0
- /16 = first 16 bits are fixed
- Available IPs: 10.0.0.0 → 10.0.255.255
- Total addresses: 65,536 IPs
```

**CIDR Quick Reference:**

| CIDR | Total IPs | Use Case |
|------|-----------|----------|
| /16 | 65,536 | Large VPC (recommended for VPCs) |
| /24 | 256 | Medium subnet |
| /28 | 16 | Small subnet |

**Best Practice:** Use RFC 1918 private IP ranges:
- `10.0.0.0/8`
- `172.16.0.0/12`
- `192.168.0.0/16`

### Default VPC vs Custom VPC

| Feature | Default VPC | Custom VPC |
|---------|------------|-----------|
| Created by | AWS automatically | You create it |
| CIDR | 172.31.0.0/16 | You choose |
| Subnets | Public only | You design |
| Use case | Learning/testing | Production |

> **Best Practice:** Never use the default VPC for production workloads. Always create
> a custom VPC with proper subnet design.

### Creating a VPC (Console Steps)

1. Go to **VPC Console → Your VPCs → Create VPC**
2. Enter:
   - **Name:** `my-production-vpc`
   - **IPv4 CIDR:** `10.0.0.0/16`
   - **Tenancy:** Default
3. Click **Create VPC**

### Creating a VPC (AWS CLI)

```bash
# Create VPC
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=my-production-vpc}]'

# Output will include VpcId like: vpc-0abc123def456789
```

---

### 📝 Assignment 4 — VPC Creation

**Task 1:** Create a custom VPC named `intern-vpc` with CIDR `10.10.0.0/16` in your
chosen region using the AWS Console. Screenshot the VPC details page.

**Task 2:** Answer these questions in writing:
- What is the difference between a VPC and a subnet?
- Why should you use `/16` for a VPC instead of `/24`?
- Can two VPCs in the same account have the same CIDR block?

**Task 3:** Using AWS CLI, create a second VPC with CIDR `10.20.0.0/16` named
`intern-vpc-2`. Show the CLI command and output.

**Deliverable:** Screenshots + `assignment4.md` with answers.

---

## Step 5: Subnets

### 🎯 What is a Subnet?

A **Subnet (Sub-Network)** is a **smaller division of your VPC**'s IP address range.

> **Real-world analogy:** Your VPC is a big apartment building. Subnets are individual floors.
> Some floors are public (lobby, reception) and some are private (residential floors).

### Why Do We Need Subnets?

1. **Separate resources by function** — Web servers, app servers, databases
2. **Control traffic** — Apply different security rules per subnet
3. **High availability** — One subnet per AZ for redundancy
4. **Network segmentation** — Isolate sensitive resources

### Public vs Private Subnets

```
VPC: 10.0.0.0/16
│
├── Public Subnet (10.0.1.0/24) — AZ: us-east-1a
│   └── Web Servers, Load Balancers, NAT Gateways
│   └── HAS route to Internet Gateway (directly reachable from internet)
│
├── Private Subnet (10.0.2.0/24) — AZ: us-east-1a
│   └── App Servers, APIs
│   └── NO direct internet access (can reach internet via NAT)
│
└── Database Subnet (10.0.3.0/24) — AZ: us-east-1a
    └── RDS, ElastiCache
    └── NO internet access at all
```

### Subnet CIDR Planning (Best Practice)

For a production VPC (`10.0.0.0/16`), always plan for multiple AZs:

```
Public Subnets:
  10.0.1.0/24  → AZ-a (254 IPs)
  10.0.2.0/24  → AZ-b (254 IPs)
  10.0.3.0/24  → AZ-c (254 IPs)

Private App Subnets:
  10.0.11.0/24 → AZ-a (254 IPs)
  10.0.12.0/24 → AZ-b (254 IPs)
  10.0.13.0/24 → AZ-c (254 IPs)

Private DB Subnets:
  10.0.21.0/24 → AZ-a (254 IPs)
  10.0.22.0/24 → AZ-b (254 IPs)
  10.0.23.0/24 → AZ-c (254 IPs)
```

> ⚠️ **Note:** AWS reserves **5 IPs** per subnet (first 4 + last 1).
> A `/24` subnet has 256 addresses, but only **251 usable**.

### Creating Subnets

**Console Steps:**
1. VPC Console → Subnets → Create Subnet
2. Select your VPC
3. Choose AZ, enter Name, enter CIDR
4. Create

**AWS CLI:**
```bash
# Create Public Subnet in AZ-a
aws ec2 create-subnet \
  --vpc-id vpc-0abc123def456789 \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet-1a}]'

# Create Private Subnet in AZ-a
aws ec2 create-subnet \
  --vpc-id vpc-0abc123def456789 \
  --cidr-block 10.0.11.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-subnet-1a}]'
```

### Auto-assign Public IPs

For public subnets, enable auto-assignment of public IPs:

```bash
aws ec2 modify-subnet-attribute \
  --subnet-id subnet-0abc123 \
  --map-public-ip-on-launch
```

---

### 📝 Assignment 5 — Subnet Design

**Task 1:** Inside your `intern-vpc` (10.10.0.0/16), create the following subnets:

| Name | CIDR | AZ | Type |
|------|------|----|------|
| public-subnet-1a | 10.10.1.0/24 | AZ-a | Public |
| public-subnet-1b | 10.10.2.0/24 | AZ-b | Public |
| private-subnet-1a | 10.10.11.0/24 | AZ-a | Private |
| private-subnet-1b | 10.10.12.0/24 | AZ-b | Private |
| db-subnet-1a | 10.10.21.0/24 | AZ-a | Database |
| db-subnet-1b | 10.10.22.0/24 | AZ-b | Database |

**Task 2:** Why is it important to have subnets in multiple AZs?
Give a concrete failure scenario where this matters.

**Task 3:** Calculate: If you have a `/24` subnet, how many IP addresses are available
for your EC2 instances? Show your math.

**Deliverable:** Console screenshots of all 6 subnets + `assignment5.md`.

---

## Step 6: Route Tables

### 🎯 What is a Route Table?

A **Route Table** is a set of **rules (routes)** that determine where network traffic goes.

Every subnet must be associated with a route table. The route table tells traffic:
**"If you're going to destination X, send it through gateway Y."**

> **Real-world analogy:** A route table is like a **GPS navigation system** for your network packets.
> When a packet needs to go somewhere, it checks the route table to find the best path.

### Why Do We Need Route Tables?

Without route tables, no traffic would flow anywhere. Route tables answer:
- **"How do I reach the internet?"** → Via Internet Gateway
- **"How do I reach other subnets?"** → Via local routing
- **"How do I reach on-premises?"** → Via VPN Gateway or Direct Connect

### How Route Tables Work

```
Route Table: public-rt

Destination        Target              Description
-----------        ------              -----------
10.0.0.0/16        local               All VPC-internal traffic stays local
0.0.0.0/0          igw-0abc123         All other traffic goes to Internet Gateway
```

When a packet arrives at a subnet:
1. The router checks the **most specific** matching route first
2. `10.0.0.0/16` matches all internal VPC traffic → sends to local
3. `0.0.0.0/0` matches everything else → sends to Internet Gateway

### Route Table Types

| Type | Description | Used For |
|------|-------------|----------|
| **Main Route Table** | Default for all subnets not explicitly associated | Rarely used in production |
| **Custom Route Table** | Explicitly associated with specific subnets | Always use this |

### Public vs Private Route Table

```
Public Route Table (for public subnets):
  10.0.0.0/16  → local
  0.0.0.0/0    → Internet Gateway

Private Route Table (for private subnets):
  10.0.0.0/16  → local
  0.0.0.0/0    → NAT Gateway (outbound only)

Database Route Table (for DB subnets):
  10.0.0.0/16  → local
  (NO 0.0.0.0/0 — no internet access)
```

### Creating Route Tables (Console)

1. VPC Console → Route Tables → Create Route Table
2. Name: `public-rt`, select your VPC
3. After creation, click **Edit Routes → Add Route**
4. Destination: `0.0.0.0/0`, Target: your Internet Gateway
5. Click **Subnet Associations → Edit → Associate** public subnets

### Creating Route Tables (CLI)

```bash
# Create route table
aws ec2 create-route-table \
  --vpc-id vpc-0abc123def456789 \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=public-rt}]'

# Add route to Internet Gateway
aws ec2 create-route \
  --route-table-id rtb-0abc123 \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-0abc123

# Associate subnet with route table
aws ec2 associate-route-table \
  --route-table-id rtb-0abc123 \
  --subnet-id subnet-0abc123
```

---

### 📝 Assignment 6 — Route Tables

**Task 1:** Create three route tables in your `intern-vpc`:
- `public-rt` — Will route to Internet Gateway
- `private-rt` — Will route to NAT Gateway (configure after Step 7)
- `db-rt` — No internet route

Associate the correct subnets with each route table.

**Task 2:** A developer asks: "Why can't my EC2 in the private subnet access the internet
even though I added a route to the Internet Gateway?"
Write a clear explanation of what's wrong and how to fix it.

**Task 3:** Draw the complete routing flow: A packet from your private subnet app server
wants to reach `google.com`. Trace every hop it takes.

**Deliverable:** Screenshots + `assignment6.md`.

---

## Step 7: Internet Gateway & NAT Gateway

### 🎯 Internet Gateway (IGW)

An **Internet Gateway** is a horizontally scaled, redundant AWS component that allows
communication between your VPC and the internet.

```
Internet ←→ Internet Gateway ←→ Public Subnet ←→ EC2 (with public IP)
```

**Key facts:**
- One IGW per VPC
- Free to attach
- Fully managed by AWS (no bandwidth limits)
- Enables BOTH inbound and outbound internet traffic

**Creating and Attaching an IGW:**

```bash
# Create Internet Gateway
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=my-igw}]'

# Attach to VPC
aws ec2 attach-internet-gateway \
  --internet-gateway-id igw-0abc123 \
  --vpc-id vpc-0abc123def456789
```

### 🎯 NAT Gateway (Network Address Translation)

A **NAT Gateway** allows instances in a **private subnet** to initiate outbound connections
to the internet, while **blocking inbound connections** from the internet.

```
Private Subnet EC2  →  NAT Gateway (in Public Subnet)  →  Internet Gateway  →  Internet
                            ↑
                     Has Elastic IP (public IP)
```

**Why NAT Gateway?**
- App servers need to download packages, make API calls, etc.
- But we don't want them directly reachable from the internet
- NAT Gateway = "one-way door" for outbound traffic only

**NAT Gateway vs NAT Instance:**

| Feature | NAT Gateway | NAT Instance |
|---------|-------------|--------------|
| Managed by | AWS | You |
| High Availability | Built-in | Manual |
| Bandwidth | Up to 100 Gbps | Limited by instance type |
| Cost | ~$0.045/hr + data | EC2 cost |
| Recommended | ✅ Yes | ❌ Legacy |

**Creating NAT Gateway:**

```bash
# First allocate an Elastic IP
aws ec2 allocate-address --domain vpc

# Create NAT Gateway in PUBLIC subnet
aws ec2 create-nat-gateway \
  --subnet-id subnet-public-1a \
  --allocation-id eipalloc-0abc123 \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=my-nat-gw}]'
```

### Complete VPC Architecture

```
                          Internet
                             │
                    [Internet Gateway]
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
   Public Subnet         Public Subnet        Public Subnet
    (AZ-a)                (AZ-b)               (AZ-c)
   10.0.1.0/24          10.0.2.0/24          10.0.3.0/24
  [NAT Gateway]        [NAT Gateway]        [NAT Gateway]
  [Load Balancer]                                 │
        │                    │                    │
   Private Subnet        Private Subnet       Private Subnet
    (AZ-a)                (AZ-b)               (AZ-c)
   10.0.11.0/24         10.0.12.0/24         10.0.13.0/24
  [App Servers]         [App Servers]        [App Servers]
        │                    │                    │
   DB Subnet             DB Subnet            DB Subnet
    (AZ-a)                (AZ-b)               (AZ-c)
   10.0.21.0/24         10.0.22.0/24         10.0.23.0/24
  [RDS Primary]         [RDS Replica]
```

---

### 📝 Assignment 7 — Internet & NAT Gateway

**Task 1:** In your `intern-vpc`:
1. Create and attach an Internet Gateway named `intern-igw`
2. Add route `0.0.0.0/0 → intern-igw` to your `public-rt`
3. Create a NAT Gateway in `public-subnet-1a` with a new Elastic IP
4. Add route `0.0.0.0/0 → NAT Gateway` to your `private-rt`

**Task 2:** A NAT Gateway is placed in a **public** subnet — not a private one. Explain why.

**Task 3:** NAT Gateway costs money (~$0.045/hr). For a cost-optimized development environment,
what alternative approach could you use? What are the tradeoffs?

**Deliverable:** VPC diagram screenshot showing complete setup + `assignment7.md`.

---

## Step 8: Security Groups & NACLs

### 🎯 Security Groups (SG)

A **Security Group** is a **virtual firewall** for your EC2 instances.
It controls inbound and outbound traffic at the **instance level**.

**Key characteristics:**
- **Stateful** — If you allow inbound traffic, the response is automatically allowed
- Applied to **instances** (not subnets)
- **Default: deny all inbound, allow all outbound**
- You can reference other security groups as sources

```
Security Group: web-server-sg
─────────────────────────────────
Inbound Rules:
  Type    | Protocol | Port | Source
  --------|----------|------|------------------
  HTTP    | TCP      | 80   | 0.0.0.0/0 (internet)
  HTTPS   | TCP      | 443  | 0.0.0.0/0 (internet)
  SSH     | TCP      | 22   | 10.0.0.0/16 (VPC only)

Outbound Rules:
  All traffic → 0.0.0.0/0 (allow all)
```

### Security Group Chaining (Best Practice)

Instead of using IP addresses, reference security groups:

```
Web SG allows: HTTP/HTTPS from internet
App SG allows: port 8080 from Web SG only
DB SG allows: port 5432 from App SG only
```

This means the database ONLY accepts connections from the app tier — not from anywhere else.

### 🎯 Network ACLs (NACLs)

A **Network ACL** is a firewall at the **subnet level** (not instance level).

| Feature | Security Group | Network ACL |
|---------|---------------|-------------|
| Level | Instance | Subnet |
| Stateful? | ✅ Yes | ❌ No (stateless) |
| Rules | Allow only | Allow and Deny |
| Evaluation | All rules | In order (lowest number first) |
| Default | Deny all in, allow all out | Allow all in, allow all out |

**When to use NACLs?**
- Block a specific malicious IP range at the subnet level
- Extra layer of defense alongside Security Groups
- Compliance requirements

```
NACL: public-nacl
──────────────────────────────────────────
Inbound Rules:
  Rule# | Type  | Protocol | Port Range | Source       | Action
  ------|-------|----------|------------|--------------|--------
  100   | HTTP  | TCP      | 80         | 0.0.0.0/0    | ALLOW
  110   | HTTPS | TCP      | 443        | 0.0.0.0/0    | ALLOW
  120   | SSH   | TCP      | 22         | YOUR_IP/32   | ALLOW
  *     | ALL   | ALL      | ALL        | 0.0.0.0/0    | DENY
```

> ⚠️ NACLs are **stateless** — you must explicitly allow return traffic.
> For HTTP (port 80 inbound), you must also allow **ephemeral ports 1024-65535 outbound**.

---

### 📝 Assignment 8 — Security Groups & NACLs

**Task 1:** Create the following Security Groups in your VPC:

- **`web-sg`**: Allow HTTP (80), HTTPS (443) from anywhere; SSH (22) from your IP only
- **`app-sg`**: Allow port 3000 from `web-sg` only; no direct internet
- **`db-sg`**: Allow port 5432 from `app-sg` only; no direct internet

**Task 2:** Explain with an example why Security Groups being **stateful** matters.
What would happen if they were stateless like NACLs?

**Task 3:** A security audit found that your database security group allows
`0.0.0.0/0` on port 5432. Write an incident response explaining:
- What the risk is
- How to fix it immediately
- How to prevent it in the future

**Deliverable:** Screenshots of all SGs + `assignment8.md`.

---

## Step 9: EC2 — Elastic Compute Cloud

### 🎯 What is EC2?

**EC2 (Elastic Compute Cloud)** is AWS's virtual server service. It lets you rent
virtual machines (called **instances**) on demand.

### Instance Types

EC2 instances are categorized by use case:

| Family | Use Case | Example Types |
|--------|----------|---------------|
| **t** (General) | Dev/test, small apps | t3.micro, t3.medium |
| **m** (General) | Production balanced | m5.large, m5.xlarge |
| **c** (Compute) | CPU-intensive | c5.xlarge, c5.2xlarge |
| **r** (Memory) | Memory-intensive | r5.large, r5.4xlarge |
| **p** (GPU) | ML, graphics | p3.2xlarge |
| **i** (Storage) | High I/O | i3.large |

**Instance size naming:**
```
t3.micro → t3 family, micro size
   micro < small < medium < large < xlarge < 2xlarge < 4xlarge...
```

### EC2 Pricing Models

| Model | Description | Savings | Use Case |
|-------|-------------|---------|----------|
| **On-Demand** | Pay per hour/second | Baseline | Dev/test, unpredictable |
| **Reserved** | 1-3 year commitment | Up to 75% | Stable production |
| **Spot** | Bid on spare capacity | Up to 90% | Batch jobs, fault-tolerant |
| **Savings Plans** | Flexible commitment | Up to 66% | Flexible production |

### Key Concepts

**AMI (Amazon Machine Image):** Template for your instance (OS + software).
```
Examples:
- Amazon Linux 2023 (Amazon's optimized Linux)
- Ubuntu 22.04 LTS
- Windows Server 2022
- Custom AMIs (your own OS + configured software)
```

**EBS (Elastic Block Store):** Persistent disk storage for EC2.
- Root volume: `/dev/xvda` (system disk)
- Data volume: `/dev/xvdb` (extra disk)
- Types: gp3 (default), io2 (high performance), st1 (throughput), sc1 (cold)

**Key Pairs:** SSH authentication keys.
```bash
# Generate key pair
aws ec2 create-key-pair \
  --key-name my-key \
  --query 'KeyMaterial' \
  --output text > my-key.pem

chmod 400 my-key.pem
```

### Launching an EC2 Instance

```bash
# Launch EC2 instance
aws ec2 run-instances \
  --image-id ami-0c02fb55956c7d316 \
  --instance-type t3.micro \
  --key-name my-key \
  --security-group-ids sg-0abc123 \
  --subnet-id subnet-0abc123 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=my-web-server}]'

# Connect via SSH
ssh -i my-key.pem ec2-user@<PUBLIC-IP>
```

### EC2 User Data (Bootstrap Script)

Run commands automatically when an instance first starts:

```bash
# Example User Data script
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Hello from EC2!</h1>" > /var/www/html/index.html
```

---

### 📝 Assignment 9 — EC2

**Task 1:** Launch a `t3.micro` EC2 instance:
- AMI: Amazon Linux 2023
- Subnet: your `public-subnet-1a`
- Security Group: your `web-sg`
- User Data: Install and start Apache web server
- Verify by visiting the public IP in browser

**Task 2:** Connect via SSH to your instance. Run these commands and screenshot output:
```bash
df -h          # disk usage
free -m        # memory usage
uptime         # system uptime
```

**Task 3:** Stop (not terminate) your instance. Start it again. What changed about
the public IP? How would you fix this if your app needs a static IP?

**Deliverable:** Screenshots + `assignment9.md` with observations.

---

## Step 10: S3 — Simple Storage Service

### 🎯 What is S3?

**S3 (Simple Storage Service)** is AWS's object storage service.
It stores files as **objects** in **buckets** — think of a bucket as a folder,
and objects as the files inside.

**S3 is:**
- Infinitely scalable (no storage limit)
- 11 nines durability (99.999999999%)
- Globally accessible via URL

### S3 Key Concepts

**Bucket:** Container for objects. Must have a globally unique name.

**Object:** Any file — images, videos, logs, backups, static websites. Max size: 5 TB.

**Object Key:** The "path" of the object in the bucket.
```
Bucket: my-company-bucket
Key: images/logo.png
URL: https://my-company-bucket.s3.amazonaws.com/images/logo.png
```

### S3 Storage Classes (Cost Optimization)

| Class | Use Case | Retrieval | Cost |
|-------|----------|-----------|------|
| **S3 Standard** | Frequently accessed | Immediate | Highest |
| **S3 Standard-IA** | Infrequent access | Immediate | Lower |
| **S3 One Zone-IA** | Non-critical, infrequent | Immediate | Lower |
| **S3 Glacier Instant** | Archives, rapid access | Milliseconds | Low |
| **S3 Glacier Flexible** | Archives, flexible | Minutes–hours | Very Low |
| **S3 Glacier Deep Archive** | Long-term archives | 12+ hours | Lowest |

### S3 Key Features

**Versioning:** Keep multiple versions of objects.
```bash
aws s3api put-bucket-versioning \
  --bucket my-bucket \
  --versioning-configuration Status=Enabled
```

**Lifecycle Policies:** Automatically transition or delete old objects.
```json
{
  "Rules": [{
    "ID": "Move to Glacier after 90 days",
    "Status": "Enabled",
    "Transitions": [{
      "Days": 90,
      "StorageClass": "GLACIER"
    }],
    "Expiration": { "Days": 365 }
  }]
}
```

**Bucket Policy:** Control access to bucket objects.
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicReadGetObject",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*"
  }]
}
```

### S3 CLI Commands

```bash
# Create bucket
aws s3 mb s3://my-unique-bucket-name --region us-east-1

# Upload file
aws s3 cp myfile.txt s3://my-bucket/folder/myfile.txt

# Download file
aws s3 cp s3://my-bucket/folder/myfile.txt ./myfile.txt

# List bucket contents
aws s3 ls s3://my-bucket/ --recursive

# Sync folder to S3
aws s3 sync ./local-folder s3://my-bucket/remote-folder

# Delete object
aws s3 rm s3://my-bucket/folder/myfile.txt
```

---

### 📝 Assignment 10 — S3

**Task 1:** Create an S3 bucket and host a static website:
1. Create bucket `intern-static-site-<your-name>`
2. Enable static website hosting
3. Upload an `index.html` with a simple webpage
4. Access it via the S3 website URL

**Task 2:** Create a lifecycle policy that:
- Moves objects to Standard-IA after 30 days
- Moves to Glacier after 90 days
- Deletes after 365 days

**Task 3:** What is the difference between S3 bucket policies and IAM policies?
When would you use each?

**Deliverable:** Screenshot of working website + `assignment10.md`.

---

## Step 11: IAM — Identity and Access Management

### 🎯 What is IAM?

**IAM (Identity and Access Management)** is how you control **who can access what** in AWS.

> **The Golden Rule of IAM: Principle of Least Privilege**
> Grant only the minimum permissions required to do the job. Nothing more.

### IAM Components

**Users:** Individual people or applications needing AWS access.

**Groups:** Collection of users. Assign permissions to groups, not users.

**Roles:** Permissions for AWS services or federated identities (not users).

**Policies:** JSON documents defining what actions are allowed/denied.

```
IAM Structure:
├── Users
│   ├── alice (developer)
│   └── bob (ops)
├── Groups
│   ├── Developers → DeveloperPolicy
│   └── Ops → OpsPolicy
└── Roles
    ├── EC2Role → S3ReadOnlyAccess
    └── LambdaRole → DynamoDBAccess
```

### IAM Policy Structure

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ReadAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ]
    }
  ]
}
```

**Key fields:**
- `Effect`: `Allow` or `Deny`
- `Action`: AWS API calls (e.g., `s3:GetObject`, `ec2:DescribeInstances`)
- `Resource`: ARN of what resource the rule applies to
- `Condition`: Optional — restrict by IP, MFA, time, etc.

### IAM Roles for EC2

Instead of putting AWS credentials on an EC2 instance, **use IAM Roles**:

```bash
# Create role policy document
cat > trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
EOF

# Create the role
aws iam create-role \
  --role-name EC2-S3-ReadOnly \
  --assume-role-policy-document file://trust-policy.json

# Attach policy to role
aws iam attach-role-policy \
  --role-name EC2-S3-ReadOnly \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

### IAM Security Best Practices

1. ✅ **Enable MFA** on root and all IAM users
2. ✅ **Never use root account** for day-to-day work
3. ✅ **Use IAM roles** for EC2, Lambda, and services
4. ✅ **Rotate access keys** every 90 days
5. ✅ **Use IAM groups** — assign policies to groups, not users
6. ✅ **Apply least privilege** — start with no access, grant only what's needed
7. ✅ **Enable CloudTrail** to audit all API calls
8. ❌ **Never commit credentials** to code repositories

---

### 📝 Assignment 11 — IAM

**Task 1:** Create the following IAM setup:
- Group: `Developers` with `AmazonEC2ReadOnlyAccess` + `AmazonS3ReadOnlyAccess`
- User: `intern-dev` (assign to Developers group)
- Generate access keys for `intern-dev` and configure AWS CLI with them

**Task 2:** Create a custom IAM policy that allows:
- List and read objects in only your `intern-static-site` S3 bucket
- NO other S3 permissions
- NO EC2 or other service permissions

**Task 3:** Create an IAM role that an EC2 instance can use to read from your S3 bucket.
Explain why using a role is more secure than storing access keys on the instance.

**Deliverable:** IAM console screenshots + your policy JSON + `assignment11.md`.

---

## Step 12: AWS CLI Setup

### 🎯 AWS CLI Installation and Configuration

The **AWS CLI** lets you manage AWS from your terminal — essential for DevOps automation.

### Installation

```bash
# Linux/macOS
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version

# macOS Homebrew
brew install awscli

# Windows
# Download installer from: https://awscli.amazonaws.com/AWSCLIV2.msi
```

### Configuration

```bash
# Configure default profile
aws configure
# AWS Access Key ID: [your key]
# AWS Secret Access Key: [your secret]
# Default region name: us-east-1
# Default output format: json

# View config
cat ~/.aws/credentials
cat ~/.aws/config

# Use named profiles (best practice for multiple accounts)
aws configure --profile production
aws configure --profile development

# Use a specific profile
aws s3 ls --profile production
```

### Useful CLI Commands Reference

```bash
# Identity check
aws sts get-caller-identity

# EC2
aws ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId,State.Name,PublicIpAddress]' --output table

# S3
aws s3 ls
aws s3 ls s3://my-bucket

# IAM
aws iam list-users
aws iam list-groups

# VPC
aws ec2 describe-vpcs
aws ec2 describe-subnets
```

---

### 📝 Assignment 12 — Final Basic Level Assignment

**Task 1 (CLI Setup):** Configure AWS CLI with your `intern-dev` credentials.
Run `aws sts get-caller-identity` and confirm you're authenticated.

**Task 2 (Full VPC Review):** In your `intern-vpc`, confirm you have:
- [ ] 6 subnets (2 public, 2 private, 2 DB) across 2 AZs
- [ ] Internet Gateway attached
- [ ] NAT Gateway in a public subnet
- [ ] 3 route tables with correct associations
- [ ] 3 security groups (web, app, db)

Draw the complete architecture diagram and label every component.

**Task 3 (Integration):** Launch an EC2 instance in your private subnet with:
- Security group: `app-sg`
- IAM role with S3 read access
- User data that installs AWS CLI and runs `aws s3 ls` on startup

Verify the instance can reach the internet (through NAT) by SSHing via a bastion host
in the public subnet.

**Task 4 (Reflection):** Write a 500-word reflection:
- What is the most important concept you learned at the Basic level?
- What surprised you most about AWS networking?
- What would you do differently if you were setting up a production VPC?

**Deliverable:** Complete architecture diagram + all screenshots + `assignment12-final.md`.

---

## 🎓 Basic Level Complete!

Congratulations on completing the Basic AWS level! You now understand:

- ✅ Cloud computing fundamentals and why AWS
- ✅ AWS global infrastructure (Regions, AZs, Edge Locations)
- ✅ VPC design — the foundation of all AWS networking
- ✅ Subnets — public, private, and database tiers
- ✅ Route tables — how traffic flows in your network
- ✅ Internet Gateway and NAT Gateway
- ✅ Security Groups and NACLs
- ✅ EC2 — launching and managing virtual servers
- ✅ S3 — object storage fundamentals
- ✅ IAM — identity, access, and security
- ✅ AWS CLI — automation foundation

**Next Step:** Proceed to [Intermediate Level README](../intermediate/README.md)

---

*Training designed by a 10-year AWS DevOps Expert | © AWS DevOps Intern Training Program*
