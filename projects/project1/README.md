# Project 1: Production-Grade AWS Three-Tier Web Application
### Resume Project | AWS DevOps Portfolio

---

## 🏆 Project Overview

Deploy a **highly available, secure, and scalable three-tier web application** on AWS using
industry-standard Infrastructure as Code. This project demonstrates core AWS competencies
that hiring managers look for in junior-to-mid DevOps engineers.

**What You'll Build:**
A production-ready web application stack including VPC networking, load balancing,
auto scaling, managed database, caching layer, CDN, and full monitoring — all automated
with Terraform.

---

## 🛠️ Technologies & AWS Services Used

- **Networking:** VPC, Subnets, Route Tables, Internet Gateway, NAT Gateway
- **Compute:** EC2 with Auto Scaling Group
- **Load Balancing:** Application Load Balancer (ALB) with SSL termination
- **Database:** Amazon RDS PostgreSQL (Multi-AZ)
- **Caching:** Amazon ElastiCache Redis
- **Storage:** Amazon S3 (static assets + ALB logs)
- **CDN:** Amazon CloudFront
- **DNS:** Amazon Route 53
- **Security:** IAM, Security Groups, AWS KMS, Secrets Manager, WAF
- **Monitoring:** CloudWatch (Dashboards, Alarms, Logs)
- **IaC:** Terraform

---

## 🏗️ Architecture

```
                        Internet
                           │
                    [Route 53 DNS]
                           │
                  [CloudFront + WAF]
                           │
              [Application Load Balancer]
              (Public Subnets - AZ-a, AZ-b)
                    /               \
           [EC2 Auto Scaling Group]
           (Private App Subnets)
           AZ-a: 10.0.11.0/24    AZ-b: 10.0.12.0/24
                    |                    |
         [ElastiCache Redis Cluster]
         (Private Cache Subnets)
                    |
         [RDS PostgreSQL Multi-AZ]
         Primary: AZ-a    Standby: AZ-b
         (Private DB Subnets)
         10.0.21.0/24    10.0.22.0/24
```

---

## 📁 Project Structure

```
project-1-three-tier-app/
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── providers.tf
│   ├── modules/
│   │   ├── vpc/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   └── outputs.tf
│   │   ├── security-groups/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   └── outputs.tf
│   │   ├── alb/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   └── outputs.tf
│   │   ├── asg/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   └── outputs.tf
│   │   ├── rds/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   └── outputs.tf
│   │   └── elasticache/
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       └── outputs.tf
│   └── environments/
│       ├── dev/
│       │   ├── main.tf
│       │   └── terraform.tfvars
│       └── production/
│           ├── main.tf
│           └── terraform.tfvars
├── app/
│   ├── app.py               # Python Flask application
│   ├── requirements.txt
│   └── templates/
│       └── index.html
├── scripts/
│   ├── user_data.sh         # EC2 bootstrap script
│   └── deploy.sh            # Deployment helper
├── monitoring/
│   └── cloudwatch_dashboard.json
└── README.md
```

---

## 🚀 Prerequisites

- AWS Account with appropriate IAM permissions
- Terraform >= 1.5.0 installed
- AWS CLI configured
- A registered domain name (optional, for Route 53)

---

## 📋 Step-by-Step Deployment Guide

### Step 1: Clone and Configure

```bash
git clone https://github.com/yourusername/aws-three-tier-app
cd aws-three-tier-app
```

### Step 2: Initialize Terraform Backend

```bash
# Create S3 bucket for Terraform state
aws s3 mb s3://your-terraform-state-$(aws sts get-caller-identity --query Account --output text)
aws s3api put-bucket-versioning \
  --bucket your-terraform-state-$(aws sts get-caller-identity --query Account --output text) \
  --versioning-configuration Status=Enabled

# Create DynamoDB table for state locking
aws dynamodb create-table \
  --table-name terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

### Step 3: Configure Variables

```bash
cd terraform/environments/production
cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars with your values
```

### Step 4: Deploy

```bash
terraform init
terraform plan
terraform apply
```

---

## 🔒 Security Highlights

- ✅ All data encrypted at rest with customer-managed KMS key
- ✅ All data encrypted in transit (TLS 1.2+)
- ✅ Database credentials stored in AWS Secrets Manager
- ✅ No public access to RDS or ElastiCache
- ✅ Security groups follow least-privilege principle
- ✅ WAF attached to CloudFront (OWASP Top 10 protection)
- ✅ VPC Flow Logs enabled
- ✅ CloudTrail enabled for audit logging
- ✅ IMDSv2 enforced on all EC2 instances

---

## 📊 Monitoring

The project includes a pre-built CloudWatch dashboard monitoring:

- EC2 CPU utilization and instance count (ASG)
- ALB request count, response times, 5XX errors
- RDS CPU, connections, and free storage
- ElastiCache cache hit ratio and memory usage
- CloudFront cache hit rate and error rate

**Alarms configured for:**

| Alarm | Threshold | Action |
|-------|-----------|--------|
| EC2 High CPU | > 75% for 5 min | Scale out + notify |
| ALB 5XX errors | > 10 in 5 min | Notify |
| RDS CPU | > 80% for 5 min | Notify |
| RDS Free Storage | < 2 GB | Critical notify |
| ASG Min Healthy | < 1 instance | Critical notify |

---

## 💰 Estimated Cost

| Resource | Config | Monthly Cost |
|----------|--------|-------------|
| EC2 (ASG) | 2× t3.micro | ~$17 |
| RDS | db.t3.micro Multi-AZ | ~$29 |
| ElastiCache | cache.t3.micro | ~$12 |
| ALB | Per LCU | ~$18 |
| NAT Gateway | Per GB | ~$32 |
| **Total** | | **~$108/month** |

> Use `terraform destroy` after testing to avoid charges.

---

## 🎯 What This Project Demonstrates (For Resume)

- Proficiency with core AWS networking (VPC, subnets, routing)
- Ability to design multi-tier, high-availability architectures
- Security-first approach to cloud infrastructure
- Infrastructure as Code with Terraform
- Production monitoring and alerting setup
- Cost awareness and resource optimization

---

## 📄 License

MIT License — Free to use for learning and portfolio purposes.
