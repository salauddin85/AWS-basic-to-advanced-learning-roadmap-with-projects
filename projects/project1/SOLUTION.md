# Project 1: Solution — Three-Tier Web Application

## Complete Terraform Implementation

### providers.tf

```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  backend "s3" {
    bucket         = "your-terraform-state-ACCOUNT_ID"
    key            = "project1/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}

provider "aws" {
  region = var.aws_region
  default_tags {
    tags = {
      Project     = "three-tier-app"
      ManagedBy   = "Terraform"
      Environment = var.environment
    }
  }
}
```

### variables.tf

```hcl
variable "aws_region"    { default = "us-east-1" }
variable "environment"   { default = "production" }
variable "vpc_cidr"      { default = "10.0.0.0/16" }
variable "db_password" {
  sensitive = true
  type      = string
}
```

### main.tf (VPC Module Call)

```hcl
data "aws_availability_zones" "available" { state = "available" }

# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
}

# Public Subnets
resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index + 1)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  tags = { Name = "public-${count.index + 1}" }
}

# Private App Subnets
resource "aws_subnet" "private_app" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 11)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = { Name = "private-app-${count.index + 1}" }
}

# Private DB Subnets
resource "aws_subnet" "private_db" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 21)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = { Name = "private-db-${count.index + 1}" }
}

# NAT Gateway
resource "aws_eip" "nat" {
  count  = 1
  domain = "vpc"
}

resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public[0].id
  depends_on    = [aws_internet_gateway.main]
}

# Route Tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
}

resource "aws_route_table_association" "public" {
  count          = 2
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main.id
  }
}

resource "aws_route_table_association" "private_app" {
  count          = 2
  subnet_id      = aws_subnet.private_app[count.index].id
  route_table_id = aws_route_table.private.id
}

# Security Groups
resource "aws_security_group" "alb" {
  name   = "alb-sg"
  vpc_id = aws_vpc.main.id
  ingress { from_port = 80  to_port = 80  protocol = "tcp" cidr_blocks = ["0.0.0.0/0"] }
  ingress { from_port = 443 to_port = 443 protocol = "tcp" cidr_blocks = ["0.0.0.0/0"] }
  egress  { from_port = 0   to_port = 0   protocol = "-1"  cidr_blocks = ["0.0.0.0/0"] }
}

resource "aws_security_group" "app" {
  name   = "app-sg"
  vpc_id = aws_vpc.main.id
  ingress {
    from_port       = 80
    to_port         = 80
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }
  egress { from_port = 0 to_port = 0 protocol = "-1" cidr_blocks = ["0.0.0.0/0"] }
}

resource "aws_security_group" "db" {
  name   = "db-sg"
  vpc_id = aws_vpc.main.id
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }
}

# ALB
resource "aws_lb" "main" {
  name               = "main-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id
  access_logs {
    bucket  = aws_s3_bucket.alb_logs.bucket
    enabled = true
  }
}

resource "aws_lb_target_group" "app" {
  name     = "app-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id
  health_check {
    path                = "/health"
    healthy_threshold   = 2
    unhealthy_threshold = 3
    interval            = 30
  }
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

# Launch Template
resource "aws_launch_template" "app" {
  name_prefix   = "app-"
  image_id      = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  iam_instance_profile { name = aws_iam_instance_profile.app.name }
  vpc_security_group_ids = [aws_security_group.app.id]
  metadata_options {
    http_tokens = "required"  # IMDSv2 required
  }
  user_data = base64encode(<<-EOF
    #!/bin/bash
    yum update -y
    yum install -y httpd
    systemctl start httpd
    systemctl enable httpd
    echo "<h1>Three-Tier App — $(hostname)</h1>" > /var/www/html/index.html
    echo "OK" > /var/www/html/health
    EOF
  )
}

# Auto Scaling Group
resource "aws_autoscaling_group" "app" {
  name                = "app-asg"
  min_size            = 2
  max_size            = 6
  desired_capacity    = 2
  vpc_zone_identifier = aws_subnet.private_app[*].id
  target_group_arns   = [aws_lb_target_group.app.arn]
  health_check_type   = "ELB"
  health_check_grace_period = 300

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }
}

# Auto Scaling Policy
resource "aws_autoscaling_policy" "cpu" {
  name                   = "cpu-target-tracking"
  autoscaling_group_name = aws_autoscaling_group.app.name
  policy_type            = "TargetTrackingScaling"
  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value = 70.0
  }
}

# RDS
resource "aws_db_subnet_group" "main" {
  name       = "main-db-subnet"
  subnet_ids = aws_subnet.private_db[*].id
}

resource "aws_db_instance" "postgres" {
  identifier             = "main-postgres"
  engine                 = "postgres"
  engine_version         = "15.4"
  instance_class         = "db.t3.micro"
  allocated_storage      = 20
  storage_encrypted      = true
  username               = "admin"
  password               = var.db_password
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.db.id]
  multi_az               = true
  backup_retention_period = 7
  deletion_protection    = true
  skip_final_snapshot    = false
  final_snapshot_identifier = "main-postgres-final"
}

# Secrets Manager for DB credentials
resource "aws_secretsmanager_secret" "db_creds" {
  name       = "/production/database/credentials"
  kms_key_id = aws_kms_key.main.arn
}

resource "aws_secretsmanager_secret_version" "db_creds" {
  secret_id = aws_secretsmanager_secret.db_creds.id
  secret_string = jsonencode({
    username = "admin"
    password = var.db_password
    host     = aws_db_instance.postgres.address
    port     = 5432
    dbname   = "production"
  })
}

# KMS Key
resource "aws_kms_key" "main" {
  description             = "Production encryption key"
  deletion_window_in_days = 7
  enable_key_rotation     = true
}

# S3 for ALB logs
resource "aws_s3_bucket" "alb_logs" {
  bucket        = "alb-logs-${data.aws_caller_identity.current.account_id}"
  force_destroy = true
}

# CloudWatch Alarms
resource "aws_cloudwatch_metric_alarm" "cpu_high" {
  alarm_name          = "asg-cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 300
  statistic           = "Average"
  threshold           = 75
  alarm_actions       = [aws_sns_topic.alerts.arn]
  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.app.name
  }
}

resource "aws_sns_topic" "alerts" {
  name = "production-alerts"
}

# Outputs
output "alb_dns" {
  value = aws_lb.main.dns_name
}

output "db_endpoint" {
  value     = aws_db_instance.postgres.address
  sensitive = true
}
```

---

## Application Code (app.py)

```python
# app/app.py
from flask import Flask, jsonify, request
import psycopg2
import redis
import os
import json
import logging
import boto3

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = Flask(__name__)

def get_db_credentials():
    """Fetch credentials from Secrets Manager"""
    client = boto3.client('secretsmanager')
    secret = client.get_secret_value(SecretId='/production/database/credentials')
    return json.loads(secret['SecretString'])

# Redis connection (lazy initialization)
redis_client = None

def get_redis():
    global redis_client
    if not redis_client:
        redis_client = redis.Redis(
            host=os.environ['REDIS_ENDPOINT'],
            port=6379,
            decode_responses=True
        )
    return redis_client

@app.route('/health')
def health():
    return 'OK', 200

@app.route('/')
def index():
    import socket
    return jsonify({
        'message': 'Three-Tier Application',
        'host': socket.gethostname(),
        'version': '1.0.0'
    })

@app.route('/users/<user_id>')
def get_user(user_id):
    cache = get_redis()
    cached = cache.get(f'user:{user_id}')
    if cached:
        logger.info(f'Cache hit for user: {user_id}')
        return jsonify(json.loads(cached))

    creds = get_db_credentials()
    conn = psycopg2.connect(
        host=creds['host'],
        database=creds['dbname'],
        user=creds['username'],
        password=creds['password']
    )
    cur = conn.cursor()
    cur.execute('SELECT id, name, email FROM users WHERE id = %s', (user_id,))
    row = cur.fetchone()
    conn.close()

    if not row:
        return jsonify({'error': 'User not found'}), 404

    user = {'id': row[0], 'name': row[1], 'email': row[2]}
    cache.setex(f'user:{user_id}', 300, json.dumps(user))

    return jsonify(user)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=80)
```

---

## User Data Bootstrap Script

```bash
#!/bin/bash
# scripts/user_data.sh
set -e

# Update system
yum update -y

# Install Python and dependencies
yum install -y python3 python3-pip gcc postgresql-devel python3-devel

# Install application
cd /opt
aws s3 cp s3://your-app-bucket/app.tar.gz .
tar -xzf app.tar.gz
cd app
pip3 install -r requirements.txt

# Create systemd service
cat > /etc/systemd/system/webapp.service << 'EOF'
[Unit]
Description=Web Application
After=network.target

[Service]
User=ec2-user
WorkingDirectory=/opt/app
ExecStart=/usr/bin/python3 app.py
Restart=always
RestartSec=3
Environment=REDIS_ENDPOINT=your-cache.abc123.cache.amazonaws.com

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start webapp
systemctl enable webapp

# Signal success to ASG
/opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource WebServerGroup --region ${AWS::Region}
```

---

## Deployment Steps

```bash
# 1. Set up state backend
aws s3 mb s3://terraform-state-$(aws sts get-caller-identity --query Account --output text)

# 2. Configure variables
export TF_VAR_db_password="$(openssl rand -base64 32)"
aws secretsmanager put-secret-value \
  --secret-id /terraform/db-password \
  --secret-string "$TF_VAR_db_password"

# 3. Initialize Terraform
terraform init

# 4. Plan
terraform plan -out=tfplan

# 5. Apply
terraform apply tfplan

# 6. Get ALB DNS
terraform output alb_dns

# 7. Verify
curl http://$(terraform output -raw alb_dns)/health
```

---

## Verification Checklist

- [ ] ALB DNS resolves and returns HTTP 200
- [ ] Both EC2 instances are healthy in target group
- [ ] ASG shows 2 running instances across 2 AZs
- [ ] RDS is Multi-AZ and not publicly accessible
- [ ] Security groups: DB only allows from app-sg
- [ ] CloudWatch dashboard shows all metrics
- [ ] SNS alarm notification email received
- [ ] Secrets Manager has DB credentials
- [ ] KMS key used for RDS encryption
- [ ] S3 bucket has ALB access logs

---

*Solution for Project 1 — Three-Tier Web Application*
