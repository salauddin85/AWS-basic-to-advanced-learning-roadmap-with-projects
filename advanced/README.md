# 🔴 AWS Training — Advanced Level
### Intern DevOps Developer Roadmap | Guided by a 10-Year AWS DevOps Expert

---

> **Prerequisites:** Complete Basic + Intermediate levels. At this stage, you're learning
> enterprise-grade patterns: containers, CI/CD pipelines, multi-account governance,
> cost optimization, disaster recovery, and security hardening.

---

## 📋 Table of Contents

1. [Containers on AWS — ECS & EKS](#step-1-containers-on-aws--ecs--eks)
2. [AWS CI/CD Pipeline](#step-2-aws-cicd-pipeline)
3. [AWS Secrets & Parameter Store](#step-3-aws-secrets--parameter-store)
4. [AWS KMS — Key Management Service](#step-4-aws-kms--key-management-service)
5. [AWS WAF & Shield](#step-5-aws-waf--shield)
6. [VPC Peering & Transit Gateway](#step-6-vpc-peering--transit-gateway)
7. [AWS Organizations & Multi-Account Strategy](#step-7-aws-organizations--multi-account-strategy)
8. [Cost Optimization & FinOps](#step-8-cost-optimization--finops)
9. [Disaster Recovery & Backup Strategy](#step-9-disaster-recovery--backup-strategy)
10. [AWS Well-Architected Framework](#step-10-aws-well-architected-framework)
11. [Observability — X-Ray, OpenTelemetry](#step-11-observability--x-ray-opentelemetry)
12. [Production-Grade Terraform Patterns](#step-12-production-grade-terraform-patterns)

---

## Step 1: Containers on AWS — ECS & EKS

### 🎯 Why Containers?

Containers package your application + dependencies into a portable unit.
"Works on my machine" becomes "works everywhere."

```
Traditional Deployment:
  Dev Machine → Staging Server → Production Server
  "It works on my laptop!" → Fails in prod (different OS, library versions)

Container Deployment:
  Build once → Run anywhere (Docker image is identical in all environments)
```

### Docker Basics (Prerequisites)

```dockerfile
# Dockerfile for a Node.js app
FROM node:20-alpine

WORKDIR /app

# Copy package files first (caching layer)
COPY package*.json ./
RUN npm ci --only=production

# Copy application code
COPY src/ ./src/

# Non-root user for security
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodeuser -u 1001
USER nodeuser

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s \
  CMD curl -f http://localhost:3000/health || exit 1

CMD ["node", "src/index.js"]
```

### ECR — Elastic Container Registry

ECR is AWS's private Docker registry. Store your container images here.

```bash
# Create ECR repository
aws ecr create-repository \
  --repository-name intern-app \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=AES256

# Authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com

# Build, tag, and push image
docker build -t intern-app .
docker tag intern-app:latest ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/intern-app:latest
docker push ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/intern-app:latest
```

### ECS — Elastic Container Service

ECS is AWS's managed container orchestration service.

**Key Concepts:**
- **Cluster:** Logical grouping of tasks/services
- **Task Definition:** Blueprint (like docker-compose.yml)
- **Service:** Ensures N copies of a task run continuously
- **Fargate:** Serverless compute for containers (no EC2 to manage)

```json
// Task Definition
{
  "family": "intern-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::ACCOUNT:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::ACCOUNT:role/ecsTaskRole",
  "containerDefinitions": [{
    "name": "intern-app",
    "image": "ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/intern-app:latest",
    "portMappings": [{"containerPort": 3000, "protocol": "tcp"}],
    "environment": [
      {"name": "NODE_ENV", "value": "production"},
      {"name": "PORT", "value": "3000"}
    ],
    "secrets": [{
      "name": "DATABASE_URL",
      "valueFrom": "arn:aws:secretsmanager:us-east-1:ACCOUNT:secret:/prod/db/url"
    }],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/intern-app",
        "awslogs-region": "us-east-1",
        "awslogs-stream-prefix": "ecs"
      }
    },
    "healthCheck": {
      "command": ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"],
      "interval": 30,
      "timeout": 5,
      "retries": 3
    }
  }]
}
```

```bash
# Create ECS Cluster
aws ecs create-cluster \
  --cluster-name intern-cluster \
  --capacity-providers FARGATE \
  --default-capacity-provider-strategy capacityProvider=FARGATE,weight=1

# Register Task Definition
aws ecs register-task-definition --cli-input-json file://task-definition.json

# Create ECS Service
aws ecs create-service \
  --cluster intern-cluster \
  --service-name intern-app-service \
  --task-definition intern-app:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration 'awsvpcConfiguration={
    subnets=[subnet-private-1a,subnet-private-1b],
    securityGroups=[sg-ecs],
    assignPublicIp=DISABLED
  }' \
  --load-balancers 'targetGroupArn=arn:...,containerName=intern-app,containerPort=3000'
```

### EKS — Elastic Kubernetes Service

EKS is managed Kubernetes. Use it when you need:
- Advanced orchestration features
- Multi-cloud portability
- Large team familiar with Kubernetes
- Complex deployment strategies (Canary, Blue/Green)

```bash
# Create EKS cluster using eksctl (easiest method)
eksctl create cluster \
  --name intern-eks \
  --region us-east-1 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 4 \
  --with-oidc \
  --ssh-access \
  --ssh-public-key my-key

# Configure kubectl
aws eks update-kubeconfig --region us-east-1 --name intern-eks

# Deploy application
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: intern-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: intern-app
  template:
    metadata:
      labels:
        app: intern-app
    spec:
      containers:
      - name: intern-app
        image: ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/intern-app:v1.2.0
        ports:
        - containerPort: 3000
        resources:
          requests:
            cpu: "100m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        env:
        - name: NODE_ENV
          value: production
```

### ECS vs EKS Decision Guide

| Consideration | Choose ECS | Choose EKS |
|---------------|-----------|-----------|
| Team experience | New to containers | Kubernetes experience |
| Complexity | Simple microservices | Complex orchestration |
| AWS lock-in | Acceptable | Want portability |
| Management overhead | Minimal (Fargate) | Higher |
| Community | AWS-focused | Large open-source |

---

### 📝 Assignment 1 — Containers

**Task 1:** Containerize a simple Node.js "Hello World" API:
- Write a `Dockerfile` following best practices (non-root, health check, small image)
- Build and push to ECR
- Deploy to ECS Fargate with 2 replicas behind an ALB

**Task 2:** Configure ECS Service Auto Scaling:
- Scale out when CPU > 60%
- Scale in when CPU < 30%
- Min: 1, Max: 6 tasks

**Task 3:** Implement a rolling deployment in ECS:
- Update your task definition with a new image tag
- Configure deployment: minimum healthy percent 100%, max 200%
- Test that zero downtime occurs during deployment

**Deliverable:** Dockerfile + task definition JSON + deployment screenshots + `assignment-adv-1.md`.

---

## Step 2: AWS CI/CD Pipeline

### 🎯 What is CI/CD?

**CI (Continuous Integration):** Automatically build and test code on every push.

**CD (Continuous Delivery/Deployment):** Automatically deploy tested code to environments.

```
Developer Push
      │
   CodeCommit / GitHub
      │
   CodeBuild (CI)
   ├── Run unit tests
   ├── Run security scan
   ├── Build Docker image
   └── Push to ECR
      │
   CodePipeline
   ├── Deploy to Dev (automatic)
   ├── Deploy to Staging (automatic)
   ├── Manual Approval Gate ← Human review
   └── Deploy to Production (automatic after approval)
```

### AWS CI/CD Services

| Service | Purpose |
|---------|---------|
| **CodeCommit** | Private Git repository |
| **CodeBuild** | Build + test runner (like Jenkins/GitHub Actions) |
| **CodeDeploy** | Deploy to EC2, ECS, Lambda |
| **CodePipeline** | Orchestrate the full pipeline |

### CodeBuild — buildspec.yml

```yaml
# buildspec.yml — placed in your repo root
version: 0.2

env:
  variables:
    AWS_REGION: us-east-1
    ECR_REPO: ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/intern-app
  secrets-manager:
    SONAR_TOKEN: /cicd/sonar/token

phases:
  install:
    runtime-versions:
      nodejs: 20
    commands:
      - echo "Installing dependencies..."
      - npm ci

  pre_build:
    commands:
      - echo "Running tests..."
      - npm test
      - echo "Running security scan..."
      - npm audit --audit-level=high
      - echo "Logging in to ECR..."
      - aws ecr get-login-password --region $AWS_REGION |
          docker login --username AWS --password-stdin $ECR_REPO

  build:
    commands:
      - echo "Building Docker image..."
      - IMAGE_TAG=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c1-7)
      - docker build -t $ECR_REPO:$IMAGE_TAG -t $ECR_REPO:latest .
      - echo "Pushing to ECR..."
      - docker push $ECR_REPO:$IMAGE_TAG
      - docker push $ECR_REPO:latest
      - echo "Creating imagedefinitions.json..."
      - printf '[{"name":"intern-app","imageUri":"%s"}]' $ECR_REPO:$IMAGE_TAG > imagedefinitions.json

  post_build:
    commands:
      - echo "Build completed successfully"

artifacts:
  files:
    - imagedefinitions.json

cache:
  paths:
    - '/root/.npm/**/*'
    - 'node_modules/**/*'
```

### CodePipeline — Full Pipeline

```bash
# Create pipeline configuration
cat > pipeline.json << 'EOF'
{
  "pipeline": {
    "name": "intern-app-pipeline",
    "roleArn": "arn:aws:iam::ACCOUNT:role/CodePipelineRole",
    "artifactStore": {
      "type": "S3",
      "location": "my-codepipeline-artifacts"
    },
    "stages": [
      {
        "name": "Source",
        "actions": [{
          "name": "Source",
          "actionTypeId": {
            "category": "Source",
            "owner": "AWS",
            "provider": "CodeCommit",
            "version": "1"
          },
          "configuration": {
            "RepositoryName": "intern-app",
            "BranchName": "main"
          },
          "outputArtifacts": [{"name": "SourceOutput"}]
        }]
      },
      {
        "name": "Build",
        "actions": [{
          "name": "Build",
          "actionTypeId": {
            "category": "Build",
            "owner": "AWS",
            "provider": "CodeBuild",
            "version": "1"
          },
          "configuration": {
            "ProjectName": "intern-app-build"
          },
          "inputArtifacts": [{"name": "SourceOutput"}],
          "outputArtifacts": [{"name": "BuildOutput"}]
        }]
      },
      {
        "name": "DeployToStaging",
        "actions": [{
          "name": "DeployECS",
          "actionTypeId": {
            "category": "Deploy",
            "owner": "AWS",
            "provider": "ECS",
            "version": "1"
          },
          "configuration": {
            "ClusterName": "intern-cluster",
            "ServiceName": "intern-app-staging"
          },
          "inputArtifacts": [{"name": "BuildOutput"}]
        }]
      },
      {
        "name": "ManualApproval",
        "actions": [{
          "name": "ApproveProduction",
          "actionTypeId": {
            "category": "Approval",
            "owner": "AWS",
            "provider": "Manual",
            "version": "1"
          },
          "configuration": {
            "NotificationArn": "arn:aws:sns:us-east-1:ACCOUNT:approvals",
            "CustomData": "Review staging before approving production deployment"
          }
        }]
      },
      {
        "name": "DeployToProduction",
        "actions": [{
          "name": "DeployECS",
          "actionTypeId": {
            "category": "Deploy",
            "owner": "AWS",
            "provider": "ECS",
            "version": "1"
          },
          "configuration": {
            "ClusterName": "intern-cluster",
            "ServiceName": "intern-app-production"
          },
          "inputArtifacts": [{"name": "BuildOutput"}]
        }]
      }
    ]
  }
}
EOF

aws codepipeline create-pipeline --cli-input-json file://pipeline.json
```

---

### 📝 Assignment 2 — CI/CD Pipeline

**Task 1:** Build a complete CI/CD pipeline for your containerized app:
- Source: GitHub or CodeCommit
- Build: CodeBuild with unit tests + Docker build + ECR push
- Deploy: ECS service deployment

**Task 2:** Add a manual approval gate before production deployment.
Test by pushing code, approving in the console, and verifying production deployment.

**Task 3:** Add automated rollback:
- If ECS deployment fails health checks, automatically roll back
- Test by deploying a deliberately broken container image

**Deliverable:** `buildspec.yml` + pipeline screenshots + `assignment-adv-2.md`.

---

## Step 3: AWS Secrets & Parameter Store

### 🎯 Never Store Secrets in Code

```
❌ NEVER DO THIS:
DATABASE_URL = "postgres://admin:Password123@mydb.amazonaws.com/prod"
API_KEY = "sk-abc123def456..."

✅ ALWAYS DO THIS:
DATABASE_URL = get_secret("/production/database/url")
API_KEY = get_secret("/production/api/key")
```

### AWS Secrets Manager

Best for: **Credentials that need automatic rotation** (DB passwords, API keys).

```bash
# Store a secret
aws secretsmanager create-secret \
  --name /production/database/credentials \
  --description "Production PostgreSQL credentials" \
  --secret-string '{
    "username": "admin",
    "password": "SuperSecure123!",
    "host": "mydb.cluster.us-east-1.rds.amazonaws.com",
    "port": 5432,
    "dbname": "production"
  }' \
  --kms-key-id arn:aws:kms:us-east-1:ACCOUNT:key/KEY_ID

# Enable automatic rotation (for RDS)
aws secretsmanager rotate-secret \
  --secret-id /production/database/credentials \
  --rotation-lambda-arn arn:aws:lambda:...:SecretsManagerRDSPostgreSQLRotationSingleUser \
  --rotation-rules AutomaticallyAfterDays=30
```

```python
# Retrieve secret in Python
import boto3, json

def get_db_credentials():
    client = boto3.client('secretsmanager', region_name='us-east-1')
    secret = client.get_secret_value(SecretId='/production/database/credentials')
    return json.loads(secret['SecretString'])

creds = get_db_credentials()
connection = psycopg2.connect(
    host=creds['host'],
    database=creds['dbname'],
    user=creds['username'],
    password=creds['password']
)
```

### SSM Parameter Store

Best for: **Non-sensitive configuration** (feature flags, endpoints, config values).

```bash
# Store parameter (SecureString for sensitive, String for non-sensitive)
aws ssm put-parameter \
  --name /production/app/database-url \
  --value "postgres://mydb.cluster.us-east-1.rds.amazonaws.com:5432/prod" \
  --type SecureString \
  --key-id alias/aws/ssm

aws ssm put-parameter \
  --name /production/app/max-connections \
  --value "100" \
  --type String

# Get parameter
aws ssm get-parameter \
  --name /production/app/database-url \
  --with-decryption

# Get multiple parameters by path
aws ssm get-parameters-by-path \
  --path /production/app/ \
  --with-decryption \
  --recursive
```

### Comparison: Secrets Manager vs Parameter Store

| Feature | Secrets Manager | Parameter Store |
|---------|-----------------|-----------------|
| Cost | $0.40/secret/month | Free (Standard) |
| Rotation | ✅ Built-in automatic | Manual |
| Encryption | ✅ Always encrypted | Optional (SecureString) |
| Use case | DB passwords, API keys | Config, feature flags |
| Cross-account | ✅ Yes | Limited |

---

### 📝 Assignment 3 — Secrets Management

**Task 1:** Migrate your RDS credentials from being hardcoded to Secrets Manager.
Update your ECS Task Definition to pull from Secrets Manager.

**Task 2:** Enable automatic rotation for your RDS password in Secrets Manager.
Verify your application keeps working after rotation (no downtime).

**Task 3:** Create a Parameter Store hierarchy for your app:
```
/production/
  /database/url
  /cache/endpoint
  /app/max-connections
  /app/log-level
/staging/
  /database/url
  /cache/endpoint
```

**Deliverable:** Proof that app works with secrets from Secrets Manager + `assignment-adv-3.md`.

---

## Step 4: AWS KMS — Key Management Service

### 🎯 What is KMS?

**KMS (Key Management Service)** is AWS's managed service for creating and managing
**cryptographic keys** used to encrypt your data.

**Why KMS?**
- Encryption at rest for S3, EBS, RDS, DynamoDB
- Centralized key management
- Audit trail of all key usage via CloudTrail
- Fine-grained access control

### KMS Concepts

**CMK (Customer Managed Key):** Keys you create and control.

**AWS Managed Key:** Keys AWS creates for you (e.g., `aws/s3`, `aws/rds`).

**Key Policy:** IAM-like policy attached to the key.

**Envelope Encryption:**
```
Your Data → Encrypted with Data Key
Data Key  → Encrypted with CMK (stored in KMS)
```

### Creating and Using KMS Keys

```bash
# Create a CMK
aws kms create-key \
  --description "Production application encryption key" \
  --key-usage ENCRYPT_DECRYPT \
  --origin AWS_KMS \
  --tags TagKey=Environment,TagValue=Production

# Create alias for easy reference
aws kms create-alias \
  --alias-name alias/production-app-key \
  --target-key-id KEY_ID

# Encrypt data
aws kms encrypt \
  --key-id alias/production-app-key \
  --plaintext "sensitive-data" \
  --output text \
  --query CiphertextBlob | base64 -d > encrypted.bin

# Decrypt data
aws kms decrypt \
  --ciphertext-blob fileb://encrypted.bin \
  --output text \
  --query Plaintext | base64 -d
```

### Encryption in S3

```bash
# Create S3 bucket with KMS encryption
aws s3api create-bucket \
  --bucket my-encrypted-bucket \
  --region us-east-1

aws s3api put-bucket-encryption \
  --bucket my-encrypted-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:ACCOUNT:alias/production-app-key"
      },
      "BucketKeyEnabled": true
    }]
  }'
```

---

### 📝 Assignment 4 — KMS

**Task 1:** Create a customer managed key (CMK) for your application.
Apply it to:
- S3 bucket encryption
- EBS volume encryption
- RDS encryption (re-create with encryption enabled)

**Task 2:** Create a key policy that allows:
- Your IAM user to administer the key
- Your EC2 IAM role to use the key for encrypt/decrypt
- Nobody else

**Task 3:** Explain envelope encryption. Draw a diagram showing how data moves through
the encryption/decryption process using KMS.

**Deliverable:** KMS key policy JSON + `assignment-adv-4.md`.

---

## Step 5: AWS WAF & Shield

### 🎯 Application Security

**WAF (Web Application Firewall):** Filters HTTP/HTTPS traffic based on rules.

**Shield:** DDoS protection.

### AWS WAF

WAF protects against:
- **SQL Injection** — `' OR 1=1 --`
- **Cross-site scripting (XSS)** — `<script>alert(1)</script>`
- **Rate limiting** — Block IP after 100 requests/5 min
- **Geo-blocking** — Block traffic from certain countries
- **Known bad bots** — Block scrapers and scanners

```bash
# Create WAF Web ACL
aws wafv2 create-web-acl \
  --name intern-waf \
  --scope CLOUDFRONT \
  --region us-east-1 \
  --default-action Allow={} \
  --rules '[
    {
      "Name": "AWSManagedRulesCommonRuleSet",
      "Priority": 1,
      "OverrideAction": {"None": {}},
      "Statement": {
        "ManagedRuleGroupStatement": {
          "VendorName": "AWS",
          "Name": "AWSManagedRulesCommonRuleSet"
        }
      },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "CommonRuleSet"
      }
    },
    {
      "Name": "RateLimitRule",
      "Priority": 2,
      "Action": {"Block": {}},
      "Statement": {
        "RateBasedStatement": {
          "Limit": 2000,
          "AggregateKeyType": "IP"
        }
      },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "RateLimit"
      }
    }
  ]' \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=intern-waf
```

### AWS Shield

| Tier | Cost | Protection |
|------|------|-----------|
| **Shield Standard** | Free | Basic L3/L4 DDoS protection (everyone gets this) |
| **Shield Advanced** | $3,000/month | L7 DDoS, 24/7 DRT team, cost protection |

---

### 📝 Assignment 5 — WAF & Shield

**Task 1:** Attach WAF to your CloudFront distribution with:
- AWS Managed Rules (OWASP Top 10 protections)
- IP rate limiting: 1000 requests per 5 minutes per IP
- Block AWS IP Reputation list

**Task 2:** Test your WAF:
- Try a SQL injection request: `curl "https://yourdomain.com/?id=1' OR '1'='1"`
- Try XSS: `curl "https://yourdomain.com/?q=<script>alert(1)</script>"`
- Verify WAF blocked them (check WAF logs)

**Task 3:** Design a WAF rule to:
- Allow traffic from Bangladesh only (for a Bangladesh-specific app)
- Block all other countries except US and UK
Show the rule configuration.

**Deliverable:** WAF console screenshots + test results + `assignment-adv-5.md`.

---

## Step 6: VPC Peering & Transit Gateway

### 🎯 Connecting Multiple VPCs

In real companies, you have multiple VPCs:
- One per environment (dev, staging, prod)
- One per business unit
- One per region

These VPCs need to communicate securely **without going over the internet**.

### VPC Peering

One-to-one, private connection between two VPCs (same or different account/region).

```
VPC-A (10.0.0.0/16) ←──── Peering ────→ VPC-B (172.16.0.0/16)
```

**Limitations:**
- No transitive routing (A→B, B→C does NOT mean A→C)
- CIDRs must not overlap

```bash
# Create peering connection
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-A-id \
  --peer-vpc-id vpc-B-id \
  --peer-region us-east-1

# Accept the peering connection
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id pcx-0abc123

# Add routes in BOTH VPCs
# In VPC-A route table:
aws ec2 create-route \
  --route-table-id rtb-VPC-A \
  --destination-cidr-block 172.16.0.0/16 \
  --vpc-peering-connection-id pcx-0abc123

# In VPC-B route table:
aws ec2 create-route \
  --route-table-id rtb-VPC-B \
  --destination-cidr-block 10.0.0.0/16 \
  --vpc-peering-connection-id pcx-0abc123
```

### Transit Gateway

For many VPCs, peering becomes unmanageable (N*(N-1)/2 connections).
**Transit Gateway** is a central hub connecting many VPCs with transitive routing.

```
           Transit Gateway
           /      |      \
      VPC-A    VPC-B    VPC-C
              (hub)
  A can now reach C through Transit Gateway!
```

```bash
# Create Transit Gateway
aws ec2 create-transit-gateway \
  --description "Company Transit Gateway" \
  --options 'DefaultRouteTableAssociation=enable,DefaultRouteTablePropagation=enable'

# Attach VPC to Transit Gateway
aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id tgw-0abc123 \
  --vpc-id vpc-0abc123 \
  --subnet-ids subnet-private-1a subnet-private-1b

# Add route in VPC to Transit Gateway
aws ec2 create-route \
  --route-table-id rtb-private \
  --destination-cidr-block 10.0.0.0/8 \
  --transit-gateway-id tgw-0abc123
```

---

### 📝 Assignment 6 — VPC Connectivity

**Task 1:** Create a second VPC `intern-vpc-2` (CIDR: 10.20.0.0/16).
Set up VPC Peering between `intern-vpc` and `intern-vpc-2`.
Verify connectivity by pinging between EC2 instances in each VPC.

**Task 2:** Create a Transit Gateway and connect both VPCs to it.
Test transitive routing by deploying a third VPC and verifying all three can communicate.

**Task 3:** A company has 15 VPCs. How many VPC Peering connections would be needed for
full mesh connectivity? How does Transit Gateway solve this? Show the math.

**Deliverable:** Architecture diagram + ping proof + `assignment-adv-6.md`.

---

## Step 7: AWS Organizations & Multi-Account Strategy

### 🎯 Why Multiple AWS Accounts?

Professional companies use **separate AWS accounts** for:
- **Security isolation** — Compromise of dev doesn't affect prod
- **Billing separation** — Track costs per team/project
- **Blast radius reduction** — Mistakes in dev can't delete prod resources
- **Compliance** — Production data isolated from developer access

### AWS Organizations Structure

```
Root (Management Account)
├── Security OU
│   ├── Security-Tooling Account (GuardDuty, Security Hub, CloudTrail)
│   └── Log-Archive Account (centralized logs)
├── Infrastructure OU
│   └── Shared-Services Account (DNS, networking hub)
├── Workloads OU
│   ├── Development Account
│   ├── Staging Account
│   └── Production Account
└── Sandbox OU
    └── Developer Sandbox Accounts (free to experiment)
```

### Service Control Policies (SCPs)

SCPs restrict what AWS actions can be taken in member accounts,
**regardless of IAM permissions**.

```json
// Deny deleting CloudTrail in ANY account
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyCloudTrailDelete",
    "Effect": "Deny",
    "Action": [
      "cloudtrail:DeleteTrail",
      "cloudtrail:StopLogging",
      "cloudtrail:UpdateTrail"
    ],
    "Resource": "*"
  }]
}
```

```json
// Restrict to approved AWS regions only
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Action": "*",
    "Resource": "*",
    "Condition": {
      "StringNotEquals": {
        "aws:RequestedRegion": ["us-east-1", "ap-southeast-1"]
      }
    }
  }]
}
```

### AWS Control Tower

AWS Control Tower sets up a **Landing Zone** — pre-configured multi-account structure
following AWS best practices. Use it to provision new accounts at scale.

---

### 📝 Assignment 7 — Multi-Account

**Task 1:** Set up AWS Organizations with your account as the management account.
Create two member accounts: `intern-dev` and `intern-staging`.

**Task 2:** Create an SCP that denies:
- Deletion of any CloudTrail trail
- Creation of resources outside us-east-1

Apply it to both member accounts. Test that you cannot delete CloudTrail.

**Task 3:** Design a full multi-account strategy for a startup growing from 3 to 50 engineers.
Include: account structure, SCPs, billing strategy, and IAM federation approach.

**Deliverable:** Organizations console screenshots + SCP JSON + strategy document + `assignment-adv-7.md`.

---

## Step 8: Cost Optimization & FinOps

### 🎯 Cloud Costs Can Spiral Quickly

Without cost management, AWS bills can become shocking.
A senior DevOps engineer is responsible for cost optimization.

### Cost Visibility Tools

**AWS Cost Explorer:** Visualize spending patterns.

**AWS Budgets:** Set alerts when costs exceed thresholds.

**AWS Cost and Usage Report (CUR):** Detailed hourly billing data.

**AWS Compute Optimizer:** Recommends rightsizing for EC2, Lambda, ECS.

### Cost Optimization Strategies

**1. Rightsizing**
```bash
# Find underutilized EC2 instances
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0abc123 \
  --start-time 2024-01-01T00:00:00 \
  --end-time 2024-02-01T00:00:00 \
  --period 86400 \
  --statistics Average

# If average CPU < 5% → downsize the instance
```

**2. Reserved Instances & Savings Plans**
```
On-Demand t3.medium (24/7 for 1 year): $0.0416/hr × 8760 hr = $364/yr
1-Year Reserved Instance (no upfront): $0.0252/hr × 8760 hr = $221/yr
Savings: $143/yr per instance (39% savings)
```

**3. Auto Scaling (Don't pay for idle)**
```
Without ASG: Always running 10 EC2 instances = $X/month
With ASG: Scales 2-10 based on load → average 4 instances = $0.4X/month
```

**4. S3 Storage Class Optimization**
```bash
# Use S3 Intelligent-Tiering for uncertain access patterns
aws s3api put-bucket-intelligent-tiering-configuration \
  --bucket my-bucket \
  --id intelligent-tiering \
  --intelligent-tiering-configuration '{
    "Id": "intelligent-tiering",
    "Status": "Enabled",
    "Tierings": [
      {"Days": 90, "AccessTier": "ARCHIVE_ACCESS"},
      {"Days": 180, "AccessTier": "DEEP_ARCHIVE_ACCESS"}
    ]
  }'
```

**5. Spot Instances for Non-Critical Workloads**
```bash
# Use Spot Instances for batch processing (up to 90% savings)
aws ec2 request-spot-fleet \
  --spot-fleet-request-config '{
    "IamFleetRole": "arn:aws:iam::ACCOUNT:role/SpotFleetRole",
    "SpotPrice": "0.03",
    "TargetCapacity": 5,
    "LaunchSpecifications": [{
      "ImageId": "ami-0abc123",
      "InstanceType": "c5.xlarge",
      "SubnetId": "subnet-private-1a"
    }]
  }'
```

**6. Data Transfer Costs (Often Overlooked)**
```
Free: Data in, same-AZ traffic, S3 to CloudFront
$0.09/GB: Data out to internet
$0.01/GB: Cross-AZ data transfer (surprisingly expensive at scale!)

Cost tip: Keep your app servers and database in the same AZ
to avoid cross-AZ charges for high-throughput apps.
```

---

### 📝 Assignment 8 — Cost Optimization

**Task 1:** Set up AWS Budgets:
- Monthly budget alert at $10 and $20
- Service-level budget for EC2 at $5
- Anomaly detection alert

**Task 2:** Analyze your current spend in Cost Explorer:
- Which service costs the most?
- What's the hourly cost of your running resources?
- What would Reserved Instances save you?

**Task 3:** Write a cost optimization report for a fictional company spending $50,000/month:
- EC2: $25,000 (various instances running 24/7)
- RDS: $10,000 (Multi-AZ, too large)
- Data transfer: $8,000 (cross-region)
- S3: $7,000 (all in Standard)

Provide specific recommendations to cut costs by 40%.

**Deliverable:** Budget screenshots + cost analysis + `assignment-adv-8.md`.

---

## Step 9: Disaster Recovery & Backup Strategy

### 🎯 The Four DR Strategies

**RTO (Recovery Time Objective):** How long can you afford to be down?
**RPO (Recovery Point Objective):** How much data can you afford to lose?

```
                  Cost    RTO       RPO
Backup & Restore  Low     Hours     Hours
Pilot Light       Medium  Minutes   Minutes
Warm Standby      High    Minutes   Seconds
Multi-Site Active Very High Seconds  Zero

Pilot Light: Minimal running infra, scale up on DR
Warm Standby: Scaled-down version always running
Multi-Site: Full capacity in two regions simultaneously
```

### AWS Backup

Centralized backup service for EBS, RDS, DynamoDB, EFS, S3.

```bash
# Create backup vault
aws backup create-backup-vault \
  --backup-vault-name production-vault \
  --encryption-key-arn arn:aws:kms:us-east-1:ACCOUNT:alias/backup-key

# Create backup plan
cat > backup-plan.json << 'EOF'
{
  "BackupPlan": {
    "BackupPlanName": "production-backup",
    "Rules": [
      {
        "RuleName": "daily-backup",
        "TargetBackupVaultName": "production-vault",
        "ScheduleExpression": "cron(0 2 * * ? *)",
        "StartWindowMinutes": 60,
        "CompletionWindowMinutes": 180,
        "Lifecycle": {
          "MoveToColdStorageAfterDays": 30,
          "DeleteAfterDays": 365
        },
        "CopyActions": [{
          "DestinationBackupVaultArn": "arn:aws:backup:us-west-2:ACCOUNT:backup-vault:dr-vault",
          "Lifecycle": {"DeleteAfterDays": 90}
        }]
      }
    ]
  }
}
EOF

aws backup create-backup-plan --cli-input-json file://backup-plan.json

# Assign resources to backup plan
aws backup create-backup-selection \
  --backup-plan-id PLAN_ID \
  --backup-selection '{
    "SelectionName": "all-production",
    "IamRoleArn": "arn:aws:iam::ACCOUNT:role/AWSBackupDefaultServiceRole",
    "ListOfTags": [{"ConditionType": "STRINGEQUALS", "ConditionKey": "Environment", "ConditionValue": "production"}]
  }'
```

### Multi-Region DR with Route 53 Failover

```
Normal:          Route 53 → Primary ALB (us-east-1)
                 Route 53 health check: HEALTHY

DR Failover:     Route 53 health check detects PRIMARY is UNHEALTHY
                 Route 53 → Failover ALB (us-west-2)
                 RDS read replica in us-west-2 promoted to primary
```

---

### 📝 Assignment 9 — Disaster Recovery

**Task 1:** Set up AWS Backup for your:
- EC2 instances (daily, 30-day retention)
- RDS database (daily, 7-day retention, cross-region copy to us-west-2)
- S3 bucket (versioning + replication to us-west-2)

**Task 2:** Simulate a regional failure:
- Terminate all EC2 instances
- Restore from backup
- Document how long it took (this is your measured RTO)

**Task 3:** Design a DR strategy for a bank application:
- RTO requirement: 15 minutes
- RPO requirement: 1 minute
- Current region: us-east-1
Which DR strategy would you use? Justify your architecture decision.

**Deliverable:** Backup plan screenshots + recovery time measurements + DR strategy doc + `assignment-adv-9.md`.

---

## Step 10: AWS Well-Architected Framework

### 🎯 The Six Pillars

The **Well-Architected Framework** is AWS's guide to building secure, high-performing,
resilient, and efficient infrastructure.

```
┌─────────────────────────────────────────────────────────────────┐
│                  Well-Architected Framework                     │
├──────────────┬──────────────┬──────────────┬────────────────────┤
│ Operational  │   Security   │ Reliability  │   Performance      │
│ Excellence   │              │              │   Efficiency       │
├──────────────┴──────────────┴──────────────┴────────────────────┤
│         Cost Optimization          Sustainability               │
└─────────────────────────────────────────────────────────────────┘
```

### 1. Operational Excellence

**Key Practices:**
- Infrastructure as Code (Terraform, CloudFormation)
- CI/CD pipelines for all deployments
- Runbooks and playbooks for operational tasks
- Post-incident reviews (blameless)
- Game days — intentionally introduce failures to test resilience

### 2. Security

**Key Practices:**
- Least privilege IAM
- Enable CloudTrail, GuardDuty, Security Hub in all accounts
- Encrypt data in transit (TLS) and at rest (KMS)
- No hardcoded credentials — use Secrets Manager
- Regular penetration testing
- VPC flow logs enabled

### 3. Reliability

**Key Practices:**
- Multi-AZ for all production resources
- Auto Scaling for compute
- Database Multi-AZ + read replicas
- Regular backup tests (restore, not just backup)
- Circuit breakers in application code
- Chaos engineering

### 4. Performance Efficiency

**Key Practices:**
- Use managed services (don't run your own Kafka, use MSK)
- Cache aggressively (CloudFront, ElastiCache)
- Choose correct instance family (compute vs memory optimized)
- Use CloudFront for global users
- Monitor and improve application performance

### 5. Cost Optimization

**Key Practices:**
- Reserved Instances/Savings Plans for stable workloads
- Spot Instances for batch/fault-tolerant workloads
- S3 lifecycle policies
- Delete unused resources (snapshots, unattached EBS, old AMIs)
- Right-size instances using Compute Optimizer

### 6. Sustainability

**Key Practices:**
- Use Graviton (ARM) instances (better performance/watt)
- Choose regions with more renewable energy
- Auto-scale down during off-hours
- Use serverless (Lambda, Fargate) — no idle compute
- Optimize data transfer (CDN, compression)

---

### 📝 Assignment 10 — Well-Architected Review

**Task 1:** Run a Well-Architected Review on your intern VPC setup.
For each of the 6 pillars, identify:
- 2 things you're doing well
- 2 things you need to improve

**Task 2:** Use the AWS Well-Architected Tool in the console to formally assess your workload.
Implement at least 3 high-risk recommendations.

**Task 3:** Write a Production Readiness Checklist (minimum 30 items) that a team should
complete before going live with a new service.

**Deliverable:** WAR Tool screenshot + checklist + pillar analysis + `assignment-adv-10.md`.

---

## Step 11: Observability — X-Ray, OpenTelemetry

### 🎯 From Monitoring to Observability

**Monitoring:** Is my system up? Are metrics normal?

**Observability:** WHY is my system behaving this way? What is the root cause?

**Three pillars of observability:**
1. **Metrics** — Aggregated numbers (CloudWatch)
2. **Logs** — Detailed event records (CloudWatch Logs, OpenSearch)
3. **Traces** — Request journey across services (X-Ray)

### AWS X-Ray

X-Ray traces requests as they flow through your distributed application.

```
Request: GET /api/orders/123
  │
  ├─ ALB: 5ms
  ├─ Lambda (get-order): 245ms
  │   ├─ DynamoDB query: 12ms
  │   └─ External API call: 220ms ← BOTTLENECK IDENTIFIED!
  └─ Response: 250ms total
```

```python
# Instrument Lambda with X-Ray
from aws_xray_sdk.core import xray_recorder, patch_all

# Patch all supported libraries (boto3, requests, etc.)
patch_all()

@xray_recorder.capture('get_order')
def get_order(order_id):
    # This function is now traced
    with xray_recorder.capture('dynamodb_query'):
        order = dynamodb.get_item(Key={'orderId': order_id})

    with xray_recorder.capture('enrich_order'):
        # Add subsegment annotations
        xray_recorder.current_subsegment().put_annotation('order_id', order_id)
        enriched = enrich_with_user_data(order)

    return enriched
```

### CloudWatch Container Insights

For ECS/EKS, Container Insights provides:
- CPU and memory per container
- Network I/O per service
- Disk I/O per volume

```bash
# Enable Container Insights on ECS
aws ecs update-cluster-settings \
  --cluster intern-cluster \
  --settings name=containerInsights,value=enabled
```

### OpenTelemetry (Future-Proof Choice)

Instead of vendor-specific SDKs, use **OpenTelemetry** — the CNCF standard for
collecting telemetry data. Send to AWS X-Ray, Prometheus, Jaeger, Datadog, etc.

```python
# OpenTelemetry with AWS X-Ray exporter
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

provider = TracerProvider()
processor = BatchSpanProcessor(OTLPSpanExporter())
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)

tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("process_order") as span:
    span.set_attribute("order.id", "123")
    span.set_attribute("user.id", "user-001")
    # ... business logic
```

---

### 📝 Assignment 11 — Observability

**Task 1:** Add X-Ray tracing to your Lambda functions.
Create a trace that shows at least 3 subsegments (DB call, external API, etc.).

**Task 2:** Build a CloudWatch Dashboard that shows all three pillars:
- Metrics panel (CPU, request count, error rate)
- Log Insights widget (recent errors)
- X-Ray traces panel (slowest requests)

**Task 3:** Set up a CloudWatch alarm that triggers when:
- P99 latency (99th percentile) > 1 second
- Measured over a 5-minute window
- Sent to SNS → your email

**Deliverable:** X-Ray trace screenshots + dashboard screenshot + `assignment-adv-11.md`.

---

## Step 12: Production-Grade Terraform Patterns

### 🎯 Enterprise Terraform Practices

At this level, Terraform must be:
- **Collaborative** (multiple engineers working simultaneously)
- **Modular** (reusable across environments and projects)
- **Secure** (no secrets in code, state encrypted)
- **Auditable** (all changes tracked in Git)

### Module Architecture

```
infrastructure/
├── modules/
│   ├── vpc/             # Reusable VPC module
│   ├── ecs-service/     # Reusable ECS service module
│   ├── rds/             # Reusable RDS module
│   └── alb/             # Reusable ALB module
├── environments/
│   ├── dev/
│   │   ├── main.tf      # Uses modules
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   └── production/
└── global/
    ├── iam/
    └── route53/
```

### Production Module Example

```hcl
# modules/ecs-service/main.tf
resource "aws_ecs_service" "this" {
  name            = var.service_name
  cluster         = var.cluster_id
  task_definition = aws_ecs_task_definition.this.arn
  desired_count   = var.desired_count
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = var.private_subnet_ids
    security_groups  = [aws_security_group.ecs_tasks.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.this.arn
    container_name   = var.container_name
    container_port   = var.container_port
  }

  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }

  deployment_controller {
    type = "ECS"
  }

  lifecycle {
    ignore_changes = [desired_count]  # Let ASG manage this
  }
}

# Auto Scaling
resource "aws_appautoscaling_target" "this" {
  max_capacity       = var.max_capacity
  min_capacity       = var.min_capacity
  resource_id        = "service/${var.cluster_name}/${aws_ecs_service.this.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "cpu" {
  name               = "${var.service_name}-cpu-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.this.resource_id
  scalable_dimension = aws_appautoscaling_target.this.scalable_dimension
  service_namespace  = aws_appautoscaling_target.this.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value = 70.0
  }
}
```

### Atlantis — GitOps for Terraform

**Atlantis** automates Terraform via Pull Requests:

```
1. Engineer opens PR: "Add new RDS instance"
2. Atlantis automatically runs: terraform plan
3. Posts plan output as PR comment
4. Tech lead reviews plan (not code alone)
5. Tech lead comments: "atlantis apply"
6. Atlantis runs: terraform apply
7. PR is merged
```

This ensures:
- **All changes are reviewed** before being applied
- **Audit trail** in Git history
- **No manual terraform apply** from laptops

---

### 📝 Assignment 12 — Final Advanced Level Assignment

**Task 1 (Production Infrastructure):** Using your Terraform modules, deploy a complete
production-grade environment:
- Multi-AZ VPC
- ECS Fargate service with 3 replicas
- RDS PostgreSQL Multi-AZ
- ElastiCache Redis
- ALB with HTTPS (ACM certificate)
- WAF attached to CloudFront
- All secrets in Secrets Manager
- CloudWatch alarms and dashboard
- AWS Backup plan

**Task 2 (CI/CD):** Set up a full pipeline that on every `git push`:
1. Runs tests
2. Builds Docker image → ECR
3. Deploys to staging
4. Requires manual approval
5. Deploys to production
6. Sends Slack notification on success/failure

**Task 3 (Chaos Engineering):** Run the following chaos experiments and document results:
- Terminate all EC2/ECS tasks simultaneously
- Simulate AZ failure (stop all resources in one AZ)
- Introduce artificial latency to your database
- Exhaust memory on one container

**Task 4 (Capstone Presentation):** Prepare a 10-minute architecture presentation covering:
- Your complete architecture diagram
- Security model (how data is protected)
- Scaling behavior (how it handles 10x traffic)
- Cost model (monthly estimate)
- DR strategy (how you recover from regional failure)

**Deliverable:** All Terraform code + pipeline config + chaos test results + architecture diagram + `assignment-adv-12-capstone.md`.

---

## 🎓 Advanced Level Complete!

You now have advanced AWS knowledge covering enterprise-grade DevOps:

- ✅ Containers — ECS/EKS/ECR with Fargate
- ✅ CI/CD — CodePipeline end-to-end automation
- ✅ Secrets Management — Secrets Manager + Parameter Store
- ✅ Encryption — KMS with CMKs and envelope encryption
- ✅ WAF & Shield — Application security at the edge
- ✅ Network connectivity — VPC Peering and Transit Gateway
- ✅ Multi-account — AWS Organizations, SCPs, Control Tower
- ✅ Cost optimization — FinOps practices and tooling
- ✅ Disaster Recovery — Backup strategies and multi-region failover
- ✅ Well-Architected — All six pillars implemented
- ✅ Observability — X-Ray, OpenTelemetry, Container Insights
- ✅ Production Terraform — Modules, remote state, GitOps

**Next Step:** Proceed to the [Resume Projects](../projects/) section.

---

*Training designed by a 10-Year AWS DevOps Expert | © AWS DevOps Intern Training Program*
