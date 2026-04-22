# 🟡 AWS Training — Intermediate Level
### Intern DevOps Developer Roadmap 

---

> **Prerequisites:** You must have completed the Basic Level and all its assignments.
> At this level, we go deeper into core services, introduce managed services, load balancing,
> auto scaling, databases, monitoring, and Infrastructure as Code.

---

## 📋 Table of Contents

1. [Elastic Load Balancing (ELB)](#step-1-elastic-load-balancing-elb)
2. [Auto Scaling Groups (ASG)](#step-2-auto-scaling-groups-asg)
3. [RDS — Relational Database Service](#step-3-rds--relational-database-service)
4. [DynamoDB](#step-4-dynamodb)
5. [ElastiCache](#step-5-elasticache)
6. [CloudWatch — Monitoring & Alerts](#step-6-cloudwatch--monitoring--alerts)
7. [CloudTrail — Audit Logging](#step-7-cloudtrail--audit-logging)
8. [Route 53 — DNS Management](#step-8-route-53--dns-management)
9. [CloudFront — CDN](#step-9-cloudfront--cdn)
10. [Lambda — Serverless Computing](#step-10-lambda--serverless-computing)
11. [Infrastructure as Code with CloudFormation](#step-11-infrastructure-as-code-with-cloudformation)
12. [Terraform on AWS](#step-12-terraform-on-aws)

---

## Step 1: Elastic Load Balancing (ELB)

### 🎯 What is a Load Balancer?

A **Load Balancer** distributes incoming traffic across multiple EC2 instances
to ensure no single server is overwhelmed. It also performs health checks and
automatically routes traffic away from unhealthy instances.

```
Internet
   │
[Load Balancer]   ← Single entry point
   ├── EC2 Instance 1 (us-east-1a)
   ├── EC2 Instance 2 (us-east-1b)
   └── EC2 Instance 3 (us-east-1a)
```

### Types of AWS Load Balancers

| Type | Layer | Use Case | Protocol |
|------|-------|----------|----------|
| **ALB** (Application) | Layer 7 | HTTP/HTTPS, path-based routing | HTTP, HTTPS, gRPC |
| **NLB** (Network) | Layer 4 | Ultra-high performance, TCP/UDP | TCP, UDP, TLS |
| **GLB** (Gateway) | Layer 3+4 | Firewall appliances | IP |
| **CLB** (Classic) | Layer 4/7 | Legacy (avoid) | HTTP, HTTPS, TCP |

### Application Load Balancer (ALB) Deep Dive

ALB is the **most commonly used** load balancer for web applications.

**Key features:**
- **Path-based routing** — `/api/*` → API servers, `/static/*` → S3
- **Host-based routing** — `api.myapp.com` → API servers, `app.myapp.com` → Web servers
- **Target Groups** — Logical grouping of targets (EC2, Lambda, IP addresses)
- **Health Checks** — Automatically removes unhealthy targets

```
ALB Architecture:
┌─────────────────────────────────────────────────────┐
│  Application Load Balancer (intern-alb)             │
│                                                     │
│  Listener: HTTP:80  →  Redirect to HTTPS            │
│  Listener: HTTPS:443 →  Forward Rules:              │
│    /api/*    →  Target Group: api-tg                │
│    /admin/*  →  Target Group: admin-tg              │
│    default   →  Target Group: web-tg                │
└─────────────────────────────────────────────────────┘
         │                    │                    │
    [api-tg]             [admin-tg]            [web-tg]
  EC2 instances         EC2 instances        EC2 instances
```

### Creating an ALB

```bash
# Create ALB
aws elbv2 create-load-balancer \
  --name intern-alb \
  --subnets subnet-public-1a subnet-public-1b \
  --security-groups sg-alb \
  --scheme internet-facing \
  --type application

# Create Target Group
aws elbv2 create-target-group \
  --name web-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-0abc123 \
  --health-check-path /health \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3

# Register targets
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --targets Id=i-0abc123 Id=i-0def456

# Create Listener
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:... \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:...
```

### Health Check Best Practices

```
Health Check Configuration:
  Protocol: HTTP
  Path: /health   (dedicated endpoint, not /)
  Port: traffic-port
  Healthy threshold: 2 consecutive successes
  Unhealthy threshold: 3 consecutive failures
  Interval: 30 seconds
  Timeout: 5 seconds
```

Your application should have a `/health` endpoint that:
- Returns HTTP 200 OK
- Checks database connectivity
- Checks cache connectivity
- Returns response within 5 seconds

---

### 📝 Assignment 1 — Load Balancer

**Task 1:** Set up an ALB with:
- 2 EC2 instances running Apache in your private subnets
- Target group with health check on `/`
- ALB in public subnets
- Verify traffic is distributed by checking access logs on each instance

**Task 2:** Simulate a failure. Stop one EC2 instance. What happens to the ALB?
How long does it take for traffic to stop routing to the failed instance?

**Task 3:** Configure path-based routing:
- `/api` → Returns "API Server Response"
- Default → Returns "Web Server Response"
Use separate target groups for each.

**Deliverable:** Architecture diagram + screenshots + `assignment-int-1.md`.

---

## Step 2: Auto Scaling Groups (ASG)

### 🎯 What is Auto Scaling?

**Auto Scaling** automatically adjusts the number of EC2 instances based on demand.
Too much traffic? Add instances. Traffic drops? Remove instances. Pay only for what you use.

```
Normal Load:     [EC2][EC2]            → 2 instances
Traffic Spike:   [EC2][EC2][EC2][EC2]  → ASG scales out to 4
Traffic Drops:   [EC2][EC2]            → ASG scales in back to 2
```

### ASG Key Components

**Launch Template:** Defines WHAT to launch (AMI, instance type, security groups, user data).

**Auto Scaling Group:** Defines HOW MANY to maintain and WHERE.

**Scaling Policies:** Defines WHEN to add or remove instances.

### Launch Template

```bash
# Create Launch Template
aws ec2 create-launch-template \
  --launch-template-name web-server-lt \
  --version-description "v1" \
  --launch-template-data '{
    "ImageId": "ami-0c02fb55956c7d316",
    "InstanceType": "t3.micro",
    "KeyName": "my-key",
    "SecurityGroupIds": ["sg-app"],
    "IamInstanceProfile": {"Name": "EC2-S3-Role"},
    "UserData": "IyEvYmluL2Jhc2gKCnl1bSB1cGRhdGUgLXkK",
    "TagSpecifications": [{
      "ResourceType": "instance",
      "Tags": [{"Key": "Name", "Value": "web-server-asg"}]
    }]
  }'
```

### Auto Scaling Group

```bash
# Create ASG
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name web-asg \
  --launch-template LaunchTemplateName=web-server-lt,Version='$Latest' \
  --min-size 2 \
  --max-size 6 \
  --desired-capacity 2 \
  --vpc-zone-identifier "subnet-private-1a,subnet-private-1b" \
  --target-group-arns arn:aws:elasticloadbalancing:... \
  --health-check-type ELB \
  --health-check-grace-period 300 \
  --tags Key=Name,Value=web-asg,PropagateAtLaunch=true
```

### Scaling Policies

**1. Target Tracking Scaling (Recommended)**
```bash
# Scale to maintain 70% CPU utilization
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name web-asg \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "TargetValue": 70.0,
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 300
  }'
```

**2. Step Scaling**
```
CPU < 30%  → Remove 1 instance
CPU 30-70% → No change
CPU > 70%  → Add 1 instance
CPU > 90%  → Add 2 instances
```

**3. Scheduled Scaling**
```bash
# Scale up at 8am (business hours start)
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name web-asg \
  --scheduled-action-name scale-up-morning \
  --recurrence "0 8 * * MON-FRI" \
  --min-size 4 \
  --max-size 10 \
  --desired-capacity 4
```

### ASG Best Practices

1. ✅ Always use **Launch Templates** (not Launch Configurations — deprecated)
2. ✅ Deploy across **multiple AZs** for high availability
3. ✅ Attach to a **Load Balancer** with health checks
4. ✅ Set **cooldown periods** to prevent thrashing (rapid scale in/out)
5. ✅ Use **Lifecycle Hooks** for graceful startup/shutdown
6. ✅ Set min size ≥ 2 for production (never 0 in prod)

---

### 📝 Assignment 2 — Auto Scaling

**Task 1:** Create an ASG:
- Min: 1, Max: 4, Desired: 2
- Launch Template: Amazon Linux 2023, t3.micro, Apache installed
- Subnets: your private subnets
- Attached to your ALB target group

**Task 2:** Create a Target Tracking policy to scale when CPU > 50%.
Use the `stress` tool on an EC2 instance (`sudo yum install stress -y && stress --cpu 4`)
to trigger scale-out. Screenshot the ASG activity log.

**Task 3:** Set up a scheduled scaling action to scale to 4 instances at a specific
time 5 minutes from now. Verify it triggers correctly.

**Deliverable:** Activity log screenshots + `assignment-int-2.md`.

---

## Step 3: RDS — Relational Database Service

### 🎯 What is RDS?

**RDS (Relational Database Service)** is AWS's managed database service.
Instead of managing your own MySQL/PostgreSQL server, AWS handles:
- Hardware provisioning
- Database setup and patching
- Automated backups
- Multi-AZ failover
- Read replicas

### Supported Database Engines

| Engine | AWS Version | Use Case |
|--------|-------------|----------|
| **MySQL** | MySQL 8.0 | General purpose web apps |
| **PostgreSQL** | PostgreSQL 15 | Advanced features, JSON |
| **MariaDB** | MariaDB 10.6 | MySQL-compatible |
| **Oracle** | Oracle EE/SE | Enterprise legacy |
| **SQL Server** | SQL Server 2019 | Windows/.NET apps |
| **Aurora MySQL** | MySQL-compatible | Cloud-native, 5x faster |
| **Aurora PostgreSQL** | PostgreSQL-compatible | Cloud-native, 3x faster |

### RDS Key Concepts

**DB Subnet Group:** Tells RDS which subnets to use (must span ≥2 AZs).

**Multi-AZ Deployment:** Synchronous replication to a standby instance.
- Automatic failover (60-120 seconds)
- For high availability, NOT for performance

**Read Replicas:** Asynchronous copies for read-heavy workloads.
- Reduce load on primary
- Can promote to standalone DB
- Can be in different region (cross-region)

```
Production RDS Architecture:
┌──────────────────────────────────────────────────────┐
│                         RDS                          │
│                                                      │
│  Primary DB (us-east-1a)  ←sync→  Standby (us-east-1b) │
│         ↓ async                                      │
│  Read Replica (us-east-1c)                           │
└──────────────────────────────────────────────────────┘
      ↑ writes                  ↑ reads only
   App servers              Reporting queries
```

### Creating RDS PostgreSQL

```bash
# Create DB Subnet Group
aws rds create-db-subnet-group \
  --db-subnet-group-name intern-db-subnet-group \
  --db-subnet-group-description "DB subnets" \
  --subnet-ids subnet-db-1a subnet-db-1b

# Create RDS Instance
aws rds create-db-instance \
  --db-instance-identifier intern-postgres \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --engine-version 15.4 \
  --master-username admin \
  --master-user-password SecurePassword123! \
  --allocated-storage 20 \
  --storage-type gp3 \
  --db-subnet-group-name intern-db-subnet-group \
  --vpc-security-group-ids sg-db \
  --multi-az \
  --backup-retention-period 7 \
  --preferred-backup-window "02:00-03:00" \
  --preferred-maintenance-window "sun:04:00-sun:05:00" \
  --deletion-protection \
  --no-publicly-accessible \
  --tags Key=Name,Value=intern-postgres
```

### RDS Security Best Practices

1. ✅ **Never make RDS publicly accessible**
2. ✅ **Use DB Security Groups** — allow only from app security group
3. ✅ **Enable encryption at rest** — `--storage-encrypted`
4. ✅ **Enable deletion protection** — prevents accidental deletion
5. ✅ **Enable automated backups** — minimum 7 days retention
6. ✅ **Use IAM authentication** instead of username/password where possible
7. ✅ **Store credentials in AWS Secrets Manager** — never in code
8. ✅ **Enable Multi-AZ** for production

### Secrets Manager for DB Credentials

```bash
# Store DB password in Secrets Manager
aws secretsmanager create-secret \
  --name /production/postgres/password \
  --secret-string '{"username":"admin","password":"SecurePassword123!"}'

# Retrieve in application code (Python example)
import boto3, json
client = boto3.client('secretsmanager')
secret = json.loads(client.get_secret_value(SecretId='/production/postgres/password')['SecretString'])
password = secret['password']
```

---

### 📝 Assignment 3 — RDS

**Task 1:** Create a PostgreSQL RDS instance:
- Type: `db.t3.micro`
- Multi-AZ: Enabled
- No public access
- Security group: `db-sg` (allows port 5432 from `app-sg`)
- Store credentials in Secrets Manager

**Task 2:** Connect to the RDS instance from your EC2 app server (via SSH tunnel or direct).
Create a test database and table. Insert 5 records.

**Task 3:** Simulate Multi-AZ failover by rebooting with failover. Record how long it takes.
Check the endpoint — does it change?

**Task 4:** Enable a Read Replica. Explain when you would use it vs. the primary.

**Deliverable:** RDS console screenshots + connection proof + `assignment-int-3.md`.

---

## Step 4: DynamoDB

### 🎯 What is DynamoDB?

**DynamoDB** is AWS's fully managed **NoSQL** database service.
It's designed for **single-digit millisecond** performance at any scale.

### When to Use DynamoDB vs RDS?

| Scenario | Use DynamoDB | Use RDS |
|----------|-------------|---------|
| Schema | Flexible/dynamic | Fixed, relational |
| Scale | Massive (millions TPS) | Moderate |
| Queries | Simple key-based | Complex SQL joins |
| Consistency | Eventually/strongly | ACID transactions |
| Examples | User sessions, IoT, gaming | E-commerce orders, banking |

### DynamoDB Core Concepts

**Table:** Collection of items (like a database table).

**Item:** Single record (like a row). Up to 400 KB.

**Attributes:** Fields on an item (like columns). Flexible — not all items need the same attributes.

**Primary Key:**
- **Partition Key only** — Unique identifier (e.g., `userId`)
- **Partition + Sort Key** — Composite key (e.g., `userId + orderDate`)

**GSI (Global Secondary Index):** Query on non-primary-key attributes.

```
Table: Orders
┌──────────────┬────────────┬───────────┬──────────┐
│ userId (PK)  │ orderId(SK)│ status    │ total    │
├──────────────┼────────────┼───────────┼──────────┤
│ user-001     │ ord-2024-1 │ DELIVERED │ 59.99    │
│ user-001     │ ord-2024-2 │ PENDING   │ 120.00   │
│ user-002     │ ord-2024-3 │ SHIPPED   │ 35.00    │
└──────────────┴────────────┴───────────┴──────────┘
GSI: status-index (query by status)
```

### DynamoDB Operations

```python
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Orders')

# Create item
table.put_item(Item={
    'userId': 'user-001',
    'orderId': 'ord-2024-001',
    'status': 'PENDING',
    'total': Decimal('59.99'),
    'items': ['item1', 'item2']
})

# Get item (by primary key)
response = table.get_item(Key={'userId': 'user-001', 'orderId': 'ord-2024-001'})

# Query (all orders for user-001)
response = table.query(
    KeyConditionExpression=Key('userId').eq('user-001')
)

# Query by status using GSI
response = table.query(
    IndexName='status-index',
    KeyConditionExpression=Key('status').eq('PENDING')
)

# Update item
table.update_item(
    Key={'userId': 'user-001', 'orderId': 'ord-2024-001'},
    UpdateExpression='SET #s = :status',
    ExpressionAttributeNames={'#s': 'status'},
    ExpressionAttributeValues={':status': 'DELIVERED'}
)
```

### DynamoDB Pricing Models

| Mode | Use Case | Pricing |
|------|----------|---------|
| **On-Demand** | Unpredictable traffic | Per request |
| **Provisioned** | Predictable traffic | Per RCU/WCU/hr |

> **RCU** = Read Capacity Unit (1 strongly consistent read of ≤4KB/sec)
> **WCU** = Write Capacity Unit (1 write of ≤1KB/sec)

---

### 📝 Assignment 4 — DynamoDB

**Task 1:** Create a DynamoDB table `UserSessions`:
- Partition Key: `userId` (String)
- Sort Key: `sessionId` (String)
- Billing: On-Demand

**Task 2:** Write a Python script that:
- Inserts 10 user session records
- Queries all sessions for a specific user
- Updates a session status
- Deletes an expired session

**Task 3:** Add a GSI on `status` attribute. Run a query to find all `ACTIVE` sessions.

**Deliverable:** Python script + query output screenshots + `assignment-int-4.md`.

---

## Step 5: ElastiCache

### 🎯 What is ElastiCache?

**ElastiCache** is AWS's managed in-memory caching service.
It dramatically reduces database load by caching frequently accessed data in RAM.

```
Without Cache:                    With Cache:
User Request → App → Database     User Request → App → Cache (hit!) → Return
                                                        Cache (miss) → DB → Cache → Return
Response: ~100ms                  Response: ~1ms (cache hit)
```

### ElastiCache Engines

| Engine | Best For | Data Structures |
|--------|----------|-----------------|
| **Redis** | Sessions, leaderboards, pub/sub, complex data | Strings, Lists, Sets, Hashes, Sorted Sets |
| **Memcached** | Simple caching, high throughput | Strings only |

> **Recommendation:** Use **Redis** — it supports more data types, persistence,
> replication, and is more feature-rich.

### Common Caching Patterns

**Cache-Aside (Lazy Loading):**
```python
import redis, json

r = redis.Redis(host='my-cache.abc.0001.use1.cache.amazonaws.com', port=6379)

def get_user(user_id):
    # Check cache first
    cached = r.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)  # Cache hit!

    # Cache miss — get from database
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)

    # Store in cache for 1 hour
    r.setex(f"user:{user_id}", 3600, json.dumps(user))
    return user
```

**Write-Through:**
```python
def update_user(user_id, data):
    # Write to DB
    db.query("UPDATE users SET ... WHERE id = %s", user_id)
    # Immediately update cache
    r.setex(f"user:{user_id}", 3600, json.dumps(data))
```

### ElastiCache Architecture

```bash
# Create Redis Replication Group (cluster with replicas)
aws elasticache create-replication-group \
  --replication-group-id intern-redis \
  --description "Intern Redis Cache" \
  --engine redis \
  --cache-node-type cache.t3.micro \
  --num-cache-clusters 2 \
  --cache-subnet-group-name intern-cache-subnet \
  --security-group-ids sg-cache \
  --at-rest-encryption-enabled \
  --transit-encryption-enabled
```

---

### 📝 Assignment 5 — ElastiCache

**Task 1:** Create a Redis ElastiCache cluster in your private subnet.

**Task 2:** Write a Python script implementing cache-aside pattern for:
- Caching database user lookup results
- 5-minute TTL
- Handling cache miss (fallback to mock DB)

**Task 3:** Explain cache invalidation strategies. When should you invalidate a cache entry?
What problems can arise from stale cache data?

**Deliverable:** Python script + `assignment-int-5.md`.

---

## Step 6: CloudWatch — Monitoring & Alerts

### 🎯 Why Monitoring Matters

In production, you need to know:
- Is my application healthy?
- Are there performance issues?
- Did something fail?
- Am I about to run out of resources?

Without monitoring, you find out about problems from angry customers.

### CloudWatch Core Components

**Metrics:** Numerical time-series data (CPU usage, request count, error rate).

**Logs:** Text log data from your applications and AWS services.

**Alarms:** Trigger actions when metrics exceed thresholds.

**Dashboards:** Visual overview of your metrics.

**Events/EventBridge:** Respond to changes in your AWS environment.

### Key Metrics to Monitor

```
EC2:
  - CPUUtilization        > 80% → Alert
  - StatusCheckFailed     > 0  → Alert (instance unhealthy)
  - NetworkIn/Out         → Track traffic
  - DiskReadOps/WriteOps  → I/O patterns

ALB:
  - TargetResponseTime    > 1s  → Performance alert
  - HTTPCode_ELB_5XX      > 0  → Alert (server errors)
  - HealthyHostCount       < 1  → Critical alert
  - RequestCount           → Traffic monitoring

RDS:
  - CPUUtilization        > 80% → Scale up
  - FreeStorageSpace      < 10% → Alert
  - DatabaseConnections   → Connection pool monitoring
  - ReadLatency           > 20ms → Performance alert

Custom Metrics (Application):
  - ErrorRate             > 1%  → Alert
  - ResponseTime          > 500ms → Alert
  - QueueDepth            > 100 → Alert
```

### Creating Alarms

```bash
# CPU alarm for EC2
aws cloudwatch put-metric-alarm \
  --alarm-name high-cpu \
  --alarm-description "CPU > 80% for 5 minutes" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=AutoScalingGroupName,Value=web-asg \
  --alarm-actions arn:aws:sns:us-east-1:123456789:alerts \
  --ok-actions arn:aws:sns:us-east-1:123456789:alerts
```

### CloudWatch Logs

```bash
# Create log group
aws logs create-log-group --log-group-name /myapp/production

# Set retention (never let logs accumulate forever!)
aws logs put-retention-policy \
  --log-group-name /myapp/production \
  --retention-in-days 30

# Query logs with CloudWatch Insights
aws logs start-query \
  --log-group-name /myapp/production \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 20'
```

### Custom Metrics (Application-Level)

```python
import boto3

cw = boto3.client('cloudwatch')

# Push custom metric
cw.put_metric_data(
    Namespace='MyApp/Production',
    MetricData=[{
        'MetricName': 'OrdersProcessed',
        'Value': 42,
        'Unit': 'Count',
        'Dimensions': [{'Name': 'Environment', 'Value': 'production'}]
    }]
)
```

### SNS — Alert Notifications

SNS (Simple Notification Service) delivers CloudWatch alarm notifications.

```bash
# Create SNS topic
aws sns create-topic --name alerts

# Subscribe email
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789:alerts \
  --protocol email \
  --notification-endpoint devops@company.com
```

---

### 📝 Assignment 6 — CloudWatch

**Task 1:** Set up a CloudWatch Dashboard with:
- EC2 CPU utilization
- ALB request count and response time
- RDS CPU and connections

**Task 2:** Create alarms with SNS email notifications for:
- EC2 CPU > 75% for 5 minutes
- ALB 5XX errors > 5 in 1 minute
- RDS free storage < 2 GB

**Task 3:** Generate some load on your EC2 instance using `stress`.
Observe the metrics in the dashboard and verify the alarm triggers.

**Deliverable:** Dashboard screenshot + alarm screenshots + email notification proof + `assignment-int-6.md`.

---

## Step 7: CloudTrail — Audit Logging

### 🎯 What is CloudTrail?

**CloudTrail** records all **API calls** made in your AWS account.
Every time anyone (user, role, service) does anything in AWS, CloudTrail logs it.

**Why it matters:**
- **Security auditing** — Who deleted that S3 bucket?
- **Compliance** — Prove who did what and when
- **Incident response** — Trace unauthorized changes
- **Operations** — Debug "who changed this security group?"

### CloudTrail Event Types

| Type | What it Records |
|------|-----------------|
| **Management Events** | API calls on resources (create, delete, modify) |
| **Data Events** | S3 GetObject, DynamoDB GetItem (high volume, paid) |
| **Insights Events** | Unusual API activity detection |

### CloudTrail Setup

```bash
# Create S3 bucket for CloudTrail logs
aws s3 mb s3://my-cloudtrail-logs-ACCOUNT_ID

# Apply bucket policy
cat > cloudtrail-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSCloudTrailAclCheck",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::my-cloudtrail-logs-ACCOUNT_ID"
    },
    {
      "Sid": "AWSCloudTrailWrite",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-cloudtrail-logs-ACCOUNT_ID/AWSLogs/ACCOUNT_ID/*",
      "Condition": {"StringEquals": {"s3:x-amz-acl": "bucket-owner-full-control"}}
    }
  ]
}
EOF

# Create trail
aws cloudtrail create-trail \
  --name management-trail \
  --s3-bucket-name my-cloudtrail-logs-ACCOUNT_ID \
  --include-global-service-events \
  --is-multi-region-trail \
  --enable-log-file-validation

aws cloudtrail start-logging --name management-trail
```

### Querying CloudTrail with Athena

For large-scale analysis, use Amazon Athena:

```sql
-- Find all EC2 terminate actions in last 30 days
SELECT eventtime, useridentity.principalid, sourceipaddress, requestparameters
FROM cloudtrail_logs
WHERE eventsource = 'ec2.amazonaws.com'
  AND eventname = 'TerminateInstances'
  AND eventtime > date_add('day', -30, now())
ORDER BY eventtime DESC;
```

---

### 📝 Assignment 7 — CloudTrail

**Task 1:** Enable CloudTrail in your account with multi-region logging to S3.

**Task 2:** Perform these actions then search for them in CloudTrail Event History:
- Launch and terminate an EC2 instance
- Create and delete an S3 bucket
- Create a new IAM user

**Task 3:** CloudTrail shows someone ran `DeleteBucket` on your production S3 bucket.
Write an incident response procedure:
- How to investigate who did it
- How to recover the data (hint: versioning/backup)
- How to prevent it in future (hint: bucket policy, MFA delete)

**Deliverable:** CloudTrail event screenshots + `assignment-int-7.md`.

---

## Step 8: Route 53 — DNS Management

### 🎯 What is Route 53?

**Route 53** is AWS's highly available and scalable DNS service.
It translates domain names (`myapp.com`) to IP addresses.

**Why "Route 53"?** TCP/UDP port 53 is the DNS port.

### Route 53 Concepts

**Hosted Zone:** Container for DNS records for a domain.

**Record Types:**

| Type | Purpose | Example |
|------|---------|---------|
| **A** | Maps domain to IPv4 | `myapp.com → 1.2.3.4` |
| **AAAA** | Maps domain to IPv6 | `myapp.com → 2001:db8::1` |
| **CNAME** | Alias to another domain | `www → myapp.com` |
| **Alias** | AWS resource alias | `myapp.com → ALB DNS` |
| **MX** | Mail server | `myapp.com → mail.myapp.com` |
| **TXT** | Text records | SPF, DKIM, domain verification |
| **NS** | Name servers | Authoritative DNS servers |

> **Important:** Use **Alias records** (not CNAME) for AWS resources like ALB, CloudFront, S3.
> Alias records are free, faster, and support the apex domain (`myapp.com`, not just `www.myapp.com`).

### Routing Policies

| Policy | Use Case |
|--------|----------|
| **Simple** | Single resource, no health checks |
| **Weighted** | A/B testing, gradual deployments |
| **Latency** | Route users to lowest-latency region |
| **Failover** | Primary/secondary disaster recovery |
| **Geolocation** | Serve different content by country |
| **Multi-Value Answer** | Multiple healthy resources |

```bash
# Create Weighted routing (10% to v2, 90% to v1)
aws route53 change-resource-record-sets \
  --hosted-zone-id ZONE_ID \
  --change-batch '{
    "Changes": [
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "myapp.com",
          "Type": "A",
          "SetIdentifier": "v1",
          "Weight": 90,
          "AliasTarget": {
            "HostedZoneId": "ALB_ZONE_ID",
            "DNSName": "my-alb-v1.us-east-1.elb.amazonaws.com",
            "EvaluateTargetHealth": true
          }
        }
      }
    ]
  }'
```

---

### 📝 Assignment 8 — Route 53

**Task 1:** Create a hosted zone for a test domain (you can use a free domain from Freenom
or just document the configuration you would apply).

**Task 2:** Create DNS records pointing to your ALB:
- `myapp.example.com` → ALB (Alias record)
- `www.myapp.example.com` → CNAME to above

**Task 3:** Design a failover DNS configuration:
- Primary: ALB in us-east-1
- Secondary: ALB in us-west-2
- Explain how Route 53 health checks determine failover timing

**Deliverable:** Route 53 console screenshots + `assignment-int-8.md`.

---

## Step 9: CloudFront — CDN

### 🎯 What is CloudFront?

**CloudFront** is AWS's **Content Delivery Network (CDN)**. It caches your content
at **edge locations** worldwide, reducing latency for global users.

```
Without CDN:
User in Bangladesh → Server in us-east-1 → 200ms latency

With CloudFront:
User in Bangladesh → Edge Location in Singapore → 20ms latency (cached!)
```

### CloudFront Key Concepts

**Distribution:** A CloudFront configuration.

**Origin:** Where CloudFront fetches content from (S3, ALB, EC2, custom).

**Cache Behavior:** Rules for what to cache and for how long.

**Edge Location:** 400+ global CDN nodes.

### CloudFront Use Cases

1. **Static websites** (S3 → CloudFront) — serve globally at low cost
2. **API acceleration** (ALB → CloudFront) — cache API responses, WAF protection
3. **Video streaming** — adaptive bitrate streaming
4. **Security** — WAF, DDoS protection (AWS Shield) integrated

### CloudFront + S3 Static Website

```bash
# Create CloudFront distribution for S3 website
aws cloudfront create-distribution \
  --distribution-config '{
    "CallerReference": "my-site-2024",
    "Origins": {
      "Quantity": 1,
      "Items": [{
        "Id": "S3-intern-static-site",
        "DomainName": "intern-static-site.s3-website-us-east-1.amazonaws.com",
        "CustomOriginConfig": {
          "HTTPPort": 80,
          "OriginProtocolPolicy": "http-only"
        }
      }]
    },
    "DefaultCacheBehavior": {
      "ViewerProtocolPolicy": "redirect-to-https",
      "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
      "TargetOriginId": "S3-intern-static-site"
    },
    "Enabled": true,
    "DefaultRootObject": "index.html",
    "HttpVersion": "http2"
  }'
```

---

### 📝 Assignment 9 — CloudFront

**Task 1:** Create a CloudFront distribution in front of your S3 static website.
Test that the site loads via the CloudFront URL.

**Task 2:** Configure cache settings:
- HTML files: no cache (TTL = 0)
- CSS/JS files: cache for 1 year
- Images: cache for 30 days

Explain why different file types need different cache TTLs.

**Task 3:** Enable HTTPS on CloudFront using ACM (AWS Certificate Manager).
Request a free SSL certificate and attach it to CloudFront.

**Deliverable:** CloudFront distribution screenshots + HTTPS verification + `assignment-int-9.md`.

---

## Step 10: Lambda — Serverless Computing

### 🎯 What is Lambda?

**AWS Lambda** is a serverless compute service. You upload code, AWS runs it.
No servers to provision, patch, or manage.

**How it works:**
1. An event triggers your Lambda (HTTP request, S3 upload, DynamoDB change, etc.)
2. AWS spins up a container to run your code
3. You're billed per millisecond of execution
4. AWS handles scaling automatically (0 to millions of requests)

### Lambda Key Concepts

**Function:** Your code + runtime + configuration.

**Runtime:** Language environment (Python 3.12, Node.js 20, Java 21, etc.)

**Handler:** Entry point function (e.g., `lambda_function.lambda_handler`).

**Trigger:** Event source that invokes the function.

**Execution Role:** IAM role the Lambda assumes.

**Timeout:** Max 15 minutes (default 3 seconds).

**Memory:** 128 MB to 10,240 MB.

### Lambda Function Example

```python
# lambda_function.py
import json
import boto3

def lambda_handler(event, context):
    """
    Triggered by API Gateway GET /users/{id}
    Returns user details from DynamoDB
    """
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('Users')

    user_id = event['pathParameters']['id']

    response = table.get_item(Key={'userId': user_id})

    if 'Item' not in response:
        return {
            'statusCode': 404,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps({'error': 'User not found'})
        }

    return {
        'statusCode': 200,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps(response['Item'])
    }
```

### Deploying Lambda

```bash
# Zip code
zip function.zip lambda_function.py

# Create Lambda function
aws lambda create-function \
  --function-name get-user \
  --runtime python3.12 \
  --role arn:aws:iam::123456789:role/lambda-dynamodb-role \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://function.zip \
  --timeout 10 \
  --memory-size 256 \
  --environment Variables='{TABLE_NAME=Users}'

# Update function code
zip function.zip lambda_function.py
aws lambda update-function-code \
  --function-name get-user \
  --zip-file fileb://function.zip

# Test invocation
aws lambda invoke \
  --function-name get-user \
  --payload '{"pathParameters": {"id": "user-001"}}' \
  output.json && cat output.json
```

### Lambda + API Gateway (Serverless API)

```
Internet → API Gateway → Lambda → DynamoDB
                      ↓
                  Response
```

```bash
# Create REST API
aws apigateway create-rest-api --name intern-api

# The full API Gateway setup requires multiple steps.
# In practice, use AWS SAM or Serverless Framework.
```

### Lambda Best Practices

1. ✅ **Keep functions small** — single responsibility
2. ✅ **Use environment variables** for configuration
3. ✅ **Reuse connections** — initialize DB/SDK outside handler
4. ✅ **Set appropriate timeout** — don't use default 3 seconds for DB calls
5. ✅ **Use Lambda Layers** for shared dependencies
6. ✅ **Enable X-Ray tracing** for debugging
7. ✅ **Monitor cold starts** — use Provisioned Concurrency for latency-sensitive apps

---

### 📝 Assignment 10 — Lambda

**Task 1:** Create a Lambda function (Python) triggered by API Gateway:
- `GET /hello` → Returns `{"message": "Hello from Lambda!", "timestamp": "..."}`
- `POST /echo` → Returns the request body back

**Task 2:** Create a Lambda function triggered by S3:
- When a file is uploaded to your S3 bucket
- Lambda logs the filename, size, and uploader to CloudWatch
- Test by uploading a file

**Task 3:** Analyze Lambda costs vs EC2:
- Your app processes 10 million requests/month
- Each request takes 200ms, uses 256 MB memory
- Calculate Lambda cost vs a t3.small EC2 ($0.0208/hr)

**Deliverable:** Lambda code + API Gateway URL proof + `assignment-int-10.md`.

---

## Step 11: Infrastructure as Code with CloudFormation

### 🎯 What is CloudFormation?

**CloudFormation** is AWS's native **Infrastructure as Code (IaC)** service.
You define your infrastructure in YAML or JSON templates, and CloudFormation creates
and manages the resources.

**Why IaC?**
- **Reproducible** — Create identical environments (dev/staging/prod) every time
- **Version controlled** — Store in Git, track changes, rollback
- **Automated** — No manual clicking in console
- **Documented** — Your code IS the documentation

### CloudFormation Template Structure

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'My web application stack'

Parameters:
  Environment:
    Type: String
    Default: development
    AllowedValues: [development, staging, production]
  InstanceType:
    Type: String
    Default: t3.micro

Mappings:
  RegionAMI:
    us-east-1:
      AMI: ami-0c02fb55956c7d316
    us-west-2:
      AMI: ami-0ceecbb0f30a902a6

Resources:
  # VPC
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-vpc'

  # Public Subnet
  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      MapPublicIpOnLaunch: true

  # Internet Gateway
  IGW:
    Type: AWS::EC2::InternetGateway

  IGWAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref MyVPC
      InternetGatewayId: !Ref IGW

  # EC2 Instance
  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !FindInMap [RegionAMI, !Ref 'AWS::Region', AMI]
      InstanceType: !Ref InstanceType
      SubnetId: !Ref PublicSubnet
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-web-server'
      UserData:
        Fn::Base64: |
          #!/bin/bash
          yum update -y
          yum install -y httpd
          systemctl start httpd

Outputs:
  WebServerPublicIP:
    Description: Public IP of web server
    Value: !GetAtt WebServer.PublicIp
    Export:
      Name: !Sub '${AWS::StackName}-WebServerIP'
```

### Deploying CloudFormation

```bash
# Validate template
aws cloudformation validate-template \
  --template-body file://template.yaml

# Deploy stack
aws cloudformation deploy \
  --stack-name intern-web-stack \
  --template-file template.yaml \
  --parameter-overrides Environment=development InstanceType=t3.micro \
  --capabilities CAPABILITY_IAM

# List stacks
aws cloudformation list-stacks

# Describe stack
aws cloudformation describe-stacks --stack-name intern-web-stack

# View outputs
aws cloudformation describe-stacks \
  --stack-name intern-web-stack \
  --query 'Stacks[0].Outputs'

# Delete stack (deletes all resources!)
aws cloudformation delete-stack --stack-name intern-web-stack
```

---

### 📝 Assignment 11 — CloudFormation

**Task 1:** Write a CloudFormation template that creates:
- VPC with CIDR 10.0.0.0/16
- 2 public subnets in different AZs
- Internet Gateway + route table
- Security group allowing HTTP/HTTPS
- EC2 instance with Apache

Deploy it, verify Apache is accessible, then delete the stack.

**Task 2:** Modify the template to accept `Environment` and `InstanceType` as parameters.
Deploy two stacks: one for dev, one for staging.

**Task 3:** What happens if your CloudFormation update fails halfway through?
Explain rollback behavior and how to handle drift.

**Deliverable:** CloudFormation template files + `assignment-int-11.md`.

---

## Step 12: Terraform on AWS

### 🎯 Why Terraform?

While CloudFormation is AWS-native, **Terraform** is the industry-standard IaC tool
that works across AWS, Azure, GCP, and hundreds of other providers.

> **In most companies, Terraform is preferred over CloudFormation** because it's
> provider-agnostic, has a larger ecosystem, and the HCL language is cleaner.

### Terraform Core Concepts

**Provider:** Plugin to interact with an API (AWS, Azure, etc.)

**Resource:** Infrastructure component to manage.

**State:** Terraform's record of what it has created (stores in `.tfstate`).

**Plan:** Preview of changes before applying.

**Apply:** Execute the changes.

### Project Structure (Best Practice)

```
terraform-project/
├── main.tf           # Main resource definitions
├── variables.tf      # Input variable declarations
├── outputs.tf        # Output values
├── providers.tf      # Provider configuration
├── terraform.tfvars  # Variable values (DO NOT commit secrets!)
└── modules/
    ├── vpc/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── ec2/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

### Terraform AWS Example

```hcl
# providers.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "intern/terraform.tfstate"
    region = "us-east-1"
  }
}

provider "aws" {
  region = var.aws_region
}
```

```hcl
# variables.tf
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  default     = "10.0.0.0/16"
}
```

```hcl
# main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 1)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.environment}-public-${count.index + 1}"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${var.environment}-igw" }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = { Name = "${var.environment}-public-rt" }
}

resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}
```

```hcl
# outputs.tf
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "Public subnet IDs"
  value       = aws_subnet.public[*].id
}
```

### Terraform Workflow

```bash
# Install Terraform
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=...] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# Initialize
terraform init

# Preview changes
terraform plan -var="environment=development"

# Apply changes
terraform apply -var="environment=development"

# Destroy everything (careful!)
terraform destroy
```

### Remote State (Critical for Teams)

```bash
# Create S3 bucket for state
aws s3 mb s3://my-terraform-state --region us-east-1
aws s3api put-bucket-versioning \
  --bucket my-terraform-state \
  --versioning-configuration Status=Enabled

# Create DynamoDB for state locking
aws dynamodb create-table \
  --table-name terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

---

### 📝 Assignment 12 — Final Intermediate Level Assignment

**Task 1 (Terraform):** Recreate your `intern-vpc` using Terraform:
- VPC, 6 subnets (public/private/DB), IGW, NAT, Route Tables
- Use variables for environment name and CIDR
- Use remote state in S3

**Task 2 (Full Stack):** Deploy a complete web application:
- ALB + ASG (min 2 instances) running Apache
- RDS PostgreSQL in DB subnet (credentials in Secrets Manager)
- ElastiCache Redis in private subnet
- CloudWatch alarms for CPU, error rate, DB connections
- All infrastructure in Terraform

**Task 3 (Architecture Review):** Draw the complete architecture diagram for your
deployed stack. Label:
- All networking components
- Security group rules
- Data flows
- AZ distribution

**Task 4 (Reflection):** Write 500 words on:
- What does "Infrastructure as Code" enable that manual console work doesn't?
- How would you handle Terraform state in a team of 10 engineers?
- What's the difference between CloudFormation and Terraform in terms of state management?

**Deliverable:** All Terraform files + architecture diagram + `assignment-int-12-final.md`.

---

## 🎓 Intermediate Level Complete!

You now have solid intermediate-level AWS knowledge:

- ✅ Load Balancers — ALB/NLB with path-based routing
- ✅ Auto Scaling — Dynamic and scheduled scaling
- ✅ RDS — Managed databases with Multi-AZ and read replicas
- ✅ DynamoDB — NoSQL at scale
- ✅ ElastiCache — Caching strategies
- ✅ CloudWatch — Monitoring, alerting, dashboards
- ✅ CloudTrail — Audit logging and security
- ✅ Route 53 — DNS and routing policies
- ✅ CloudFront — Global CDN
- ✅ Lambda — Serverless computing
- ✅ CloudFormation — AWS-native IaC
- ✅ Terraform — Industry-standard IaC

**Next Step:** Proceed to [Advanced Level README](../advanced/README.md)

---

*Training designed by a 10-Year AWS DevOps Expert | © AWS DevOps Intern Training Program*
