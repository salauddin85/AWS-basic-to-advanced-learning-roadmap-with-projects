# Project 3: Solution — Containerized Microservices on ECS

## Complete Implementation

---

## User Service

### services/user-service/Dockerfile

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.12-slim AS runtime

RUN groupadd --gid 1001 appgroup && \
    useradd --uid 1001 --gid appgroup --no-create-home appuser

WORKDIR /app
COPY --from=builder /root/.local /home/appuser/.local
COPY --chown=appuser:appgroup src/ .

USER appuser
ENV PATH=/home/appuser/.local/bin:$PATH \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

EXPOSE 3001

HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:3001/health')" || exit 1

CMD ["gunicorn", "--bind", "0.0.0.0:3001", "--workers", "2", "--timeout", "30", "app:app"]
```

### services/user-service/src/app.py

```python
import os
import json
import uuid
import logging
import boto3
import psycopg2
from flask import Flask, jsonify, request
from aws_xray_sdk.core import xray_recorder, patch_all

patch_all()

logging.basicConfig(
    format='%(asctime)s %(levelname)s %(name)s %(message)s',
    level=os.environ.get('LOG_LEVEL', 'INFO')
)
logger = logging.getLogger('user-service')

app = Flask(__name__)
app.config['SERVICE_NAME'] = 'user-service'


def get_db_connection():
    """Fetch credentials from Secrets Manager and connect."""
    sm = boto3.client('secretsmanager')
    secret = json.loads(
        sm.get_secret_value(SecretId=os.environ['DB_SECRET_ARN'])['SecretString']
    )
    return psycopg2.connect(
        host=secret['host'],
        database=secret['dbname'],
        user=secret['username'],
        password=secret['password'],
        connect_timeout=5
    )


def init_db():
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute("""
        CREATE TABLE IF NOT EXISTS users (
            user_id   VARCHAR(36) PRIMARY KEY,
            name      VARCHAR(100) NOT NULL,
            email     VARCHAR(255) NOT NULL UNIQUE,
            status    VARCHAR(20)  NOT NULL DEFAULT 'ACTIVE',
            created_at TIMESTAMP   DEFAULT NOW(),
            updated_at TIMESTAMP   DEFAULT NOW()
        )
    """)
    conn.commit()
    cur.close()
    conn.close()
    logger.info('Database initialized')


@app.route('/health')
def health():
    try:
        conn = get_db_connection()
        conn.close()
        db_status = 'ok'
    except Exception as e:
        db_status = f'error: {str(e)}'

    status = 'healthy' if db_status == 'ok' else 'degraded'
    return jsonify({'status': status, 'db': db_status}), 200 if status == 'healthy' else 503


@app.route('/users', methods=['POST'])
@xray_recorder.capture('create_user')
def create_user():
    data = request.get_json()
    if not data or not data.get('name') or not data.get('email'):
        return jsonify({'error': 'name and email are required'}), 400

    user_id = str(uuid.uuid4())
    conn = get_db_connection()
    try:
        cur = conn.cursor()
        cur.execute(
            "INSERT INTO users (user_id, name, email) VALUES (%s, %s, %s)",
            (user_id, data['name'], data['email'])
        )
        conn.commit()
        logger.info(json.dumps({'event': 'user_created', 'user_id': user_id}))
        return jsonify({'userId': user_id, 'name': data['name'], 'email': data['email']}), 201
    except psycopg2.IntegrityError:
        conn.rollback()
        return jsonify({'error': 'Email already exists'}), 409
    finally:
        cur.close()
        conn.close()


@app.route('/users/<user_id>', methods=['GET'])
@xray_recorder.capture('get_user')
def get_user(user_id):
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute("SELECT user_id, name, email, status FROM users WHERE user_id = %s", (user_id,))
    row = cur.fetchone()
    cur.close()
    conn.close()

    if not row:
        return jsonify({'error': 'User not found'}), 404

    return jsonify({'userId': row[0], 'name': row[1], 'email': row[2], 'status': row[3]})


if __name__ == '__main__':
    init_db()
    app.run(host='0.0.0.0', port=3001)
```

### services/user-service/requirements.txt

```
flask==3.0.0
gunicorn==21.2.0
psycopg2-binary==2.9.9
boto3==1.34.0
aws-xray-sdk==2.12.0
```

---

## Order Service

### services/order-service/src/app.py

```python
import os, json, uuid, logging, boto3
import psycopg2
from flask import Flask, jsonify, request
from aws_xray_sdk.core import xray_recorder, patch_all
from datetime import datetime

patch_all()
logging.basicConfig(level=os.environ.get('LOG_LEVEL', 'INFO'))
logger = logging.getLogger('order-service')

app = Flask(__name__)
sqs = boto3.client('sqs')
sm  = boto3.client('secretsmanager')


def get_db_connection():
    secret = json.loads(
        sm.get_secret_value(SecretId=os.environ['DB_SECRET_ARN'])['SecretString']
    )
    return psycopg2.connect(
        host=secret['host'], database=secret['dbname'],
        user=secret['username'], password=secret['password']
    )


@app.route('/health')
def health():
    return jsonify({'status': 'healthy', 'service': 'order-service'}), 200


@app.route('/orders', methods=['POST'])
@xray_recorder.capture('create_order')
def create_order():
    data = request.get_json()
    if not data or not data.get('userId') or not data.get('items'):
        return jsonify({'error': 'userId and items are required'}), 400

    order_id = str(uuid.uuid4())
    total    = sum(float(i['price']) * int(i['quantity']) for i in data['items'])

    conn = get_db_connection()
    cur  = conn.cursor()
    cur.execute(
        "INSERT INTO orders (order_id, user_id, status, items, total) VALUES (%s, %s, %s, %s, %s)",
        (order_id, data['userId'], 'PENDING', json.dumps(data['items']), total)
    )
    conn.commit()
    cur.close()
    conn.close()

    # Publish async event to notification service
    with xray_recorder.capture('sqs_publish'):
        sqs.send_message(
            QueueUrl=os.environ['NOTIFICATION_QUEUE_URL'],
            MessageBody=json.dumps({
                'eventType': 'ORDER_CREATED',
                'orderId':   order_id,
                'userId':    data['userId'],
                'total':     str(total),
                'timestamp': datetime.utcnow().isoformat()
            })
        )

    logger.info(json.dumps({'event': 'order_created', 'orderId': order_id}))
    return jsonify({'orderId': order_id, 'status': 'PENDING', 'total': total}), 201


@app.route('/orders/user/<user_id>', methods=['GET'])
@xray_recorder.capture('get_orders')
def get_user_orders(user_id):
    conn = get_db_connection()
    cur  = conn.cursor()
    cur.execute(
        "SELECT order_id, status, total, created_at FROM orders WHERE user_id = %s ORDER BY created_at DESC",
        (user_id,)
    )
    orders = [{'orderId': r[0], 'status': r[1], 'total': float(r[2]), 'createdAt': str(r[3])}
              for r in cur.fetchall()]
    cur.close()
    conn.close()
    return jsonify({'orders': orders, 'count': len(orders)})


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=3002)
```

---

## Notification Service

### services/notification-service/src/app.py

```python
import os, json, time, logging, boto3

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger('notification-service')

sqs = boto3.client('sqs')
ses = boto3.client('ses')

QUEUE_URL    = os.environ['NOTIFICATION_QUEUE_URL']
FROM_EMAIL   = os.environ.get('FROM_EMAIL', 'noreply@example.com')


def send_order_confirmation(event):
    """Send order confirmation email via SES."""
    try:
        ses.send_email(
            Source=FROM_EMAIL,
            Destination={'ToAddresses': [event.get('userEmail', FROM_EMAIL)]},
            Message={
                'Subject': {'Data': f"Order Confirmed #{event['orderId'][:8].upper()}"},
                'Body': {
                    'Text': {
                        'Data': (
                            f"Thank you for your order!\n\n"
                            f"Order ID : {event['orderId']}\n"
                            f"Total    : ${event['total']}\n"
                            f"Status   : PENDING\n\n"
                            f"We'll notify you when it ships."
                        )
                    }
                }
            }
        )
        logger.info(json.dumps({'event': 'email_sent', 'orderId': event['orderId']}))
    except Exception as e:
        logger.error(f'Failed to send email: {e}')


def process_message(message):
    event = json.loads(message['Body'])
    if event.get('eventType') == 'ORDER_CREATED':
        send_order_confirmation(event)
    else:
        logger.warning(f"Unknown eventType: {event.get('eventType')}")


def main():
    logger.info('Notification service started — polling SQS...')
    while True:
        try:
            resp = sqs.receive_message(
                QueueUrl=QUEUE_URL,
                MaxNumberOfMessages=10,
                WaitTimeSeconds=20
            )
            for msg in resp.get('Messages', []):
                process_message(msg)
                sqs.delete_message(
                    QueueUrl=QUEUE_URL,
                    ReceiptHandle=msg['ReceiptHandle']
                )
        except Exception as e:
            logger.error(f'SQS polling error: {e}')
            time.sleep(5)


if __name__ == '__main__':
    main()
```

---

## Terraform Infrastructure

### infrastructure/terraform/main.tf

```hcl
# ── Data ─────────────────────────────────────────────────────────────
data "aws_availability_zones" "available" { state = "available" }
data "aws_caller_identity"    "current"   {}

# ── VPC ──────────────────────────────────────────────────────────────
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = { Name = "${var.environment}-vpc" }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
}

resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet("10.0.0.0/16", 8, count.index + 1)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  tags = { Name = "public-${count.index + 1}" }
}

resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet("10.0.0.0/16", 8, count.index + 11)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = { Name = "private-${count.index + 1}" }
}

resource "aws_eip"         "nat" { domain = "vpc" }
resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public[0].id
  depends_on    = [aws_internet_gateway.main]
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route { cidr_block = "0.0.0.0/0"; gateway_id = aws_internet_gateway.main.id }
}
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  route { cidr_block = "0.0.0.0/0"; nat_gateway_id = aws_nat_gateway.main.id }
}
resource "aws_route_table_association" "public" {
  count          = 2
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}
resource "aws_route_table_association" "private" {
  count          = 2
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}

# ── Security Groups ───────────────────────────────────────────────────
resource "aws_security_group" "alb" {
  name   = "alb-sg"
  vpc_id = aws_vpc.main.id
  ingress { from_port = 80  to_port = 80  protocol = "tcp" cidr_blocks = ["0.0.0.0/0"] }
  ingress { from_port = 443 to_port = 443 protocol = "tcp" cidr_blocks = ["0.0.0.0/0"] }
  egress  { from_port = 0   to_port = 0   protocol = "-1"  cidr_blocks = ["0.0.0.0/0"] }
}

resource "aws_security_group" "ecs" {
  name   = "ecs-tasks-sg"
  vpc_id = aws_vpc.main.id
  ingress {
    from_port       = 0
    to_port         = 65535
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }
  egress { from_port = 0 to_port = 0 protocol = "-1" cidr_blocks = ["0.0.0.0/0"] }
}

resource "aws_security_group" "rds" {
  name   = "rds-sg"
  vpc_id = aws_vpc.main.id
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.ecs.id]
  }
}

# ── ECR Repositories ─────────────────────────────────────────────────
resource "aws_ecr_repository" "services" {
  for_each = toset(["user-service", "order-service", "notification-service"])
  name     = each.value
  image_scanning_configuration { scan_on_push = true }
  image_tag_mutability = "MUTABLE"
}

resource "aws_ecr_lifecycle_policy" "cleanup" {
  for_each   = aws_ecr_repository.services
  repository = each.value.name
  policy = jsonencode({
    rules = [
      { rulePriority = 1, description = "Keep last 10 tagged images",
        selection = { tagStatus = "tagged", tagPrefixList = ["v"], countType = "imageCountMoreThan", countNumber = 10 },
        action = { type = "expire" }
      },
      { rulePriority = 2, description = "Remove untagged after 1 day",
        selection = { tagStatus = "untagged", countType = "sinceImagePushed", countUnit = "days", countNumber = 1 },
        action = { type = "expire" }
      }
    ]
  })
}

# ── ECS Cluster ───────────────────────────────────────────────────────
resource "aws_ecs_cluster" "main" {
  name = "${var.environment}-cluster"
  setting { name = "containerInsights"; value = "enabled" }
}

resource "aws_ecs_cluster_capacity_providers" "main" {
  cluster_name       = aws_ecs_cluster.main.name
  capacity_providers = ["FARGATE", "FARGATE_SPOT"]
  default_capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = 1
  }
}

# ── ALB ───────────────────────────────────────────────────────────────
resource "aws_lb" "main" {
  name               = "${var.environment}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id
}

resource "aws_lb_target_group" "services" {
  for_each    = { user-service = 3001, order-service = 3002 }
  name        = "${each.key}-tg"
  port        = each.value
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"
  health_check {
    path                = "/health"
    healthy_threshold   = 2
    unhealthy_threshold = 3
    interval            = 30
    timeout             = 5
  }
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"
  default_action    { type = "fixed-response"
    fixed_response { content_type = "text/plain"; message_body = "Not Found"; status_code = "404" }
  }
}

resource "aws_lb_listener_rule" "user_service" {
  listener_arn = aws_lb_listener.http.arn
  priority     = 100
  action       { type = "forward"; target_group_arn = aws_lb_target_group.services["user-service"].arn }
  condition    { path_pattern { values = ["/users*"] } }
}

resource "aws_lb_listener_rule" "order_service" {
  listener_arn = aws_lb_listener.http.arn
  priority     = 110
  action       { type = "forward"; target_group_arn = aws_lb_target_group.services["order-service"].arn }
  condition    { path_pattern { values = ["/orders*"] } }
}

# ── SQS ───────────────────────────────────────────────────────────────
resource "aws_sqs_queue" "notification_dlq" {
  name                      = "notification-dlq"
  message_retention_seconds = 1209600  # 14 days
}

resource "aws_sqs_queue" "notification" {
  name                       = "notification-queue"
  visibility_timeout_seconds = 60
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.notification_dlq.arn
    maxReceiveCount     = 3
  })
}

# ── IAM Task Roles ────────────────────────────────────────────────────
resource "aws_iam_role" "ecs_execution" {
  name = "ecs-execution-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{ Effect = "Allow"; Principal = { Service = "ecs-tasks.amazonaws.com" }; Action = "sts:AssumeRole" }]
  })
}
resource "aws_iam_role_policy_attachment" "ecs_execution" {
  role       = aws_iam_role.ecs_execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}
resource "aws_iam_role_policy" "secrets_access" {
  name = "secrets-access"
  role = aws_iam_role.ecs_execution.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{ Effect = "Allow"; Action = ["secretsmanager:GetSecretValue"]; Resource = "*" }]
  })
}

resource "aws_iam_role" "order_service_task" {
  name = "order-service-task-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{ Effect = "Allow"; Principal = { Service = "ecs-tasks.amazonaws.com" }; Action = "sts:AssumeRole" }]
  })
}
resource "aws_iam_role_policy" "order_sqs" {
  name = "sqs-publish"
  role = aws_iam_role.order_service_task.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{ Effect = "Allow"; Action = ["sqs:SendMessage"]; Resource = aws_sqs_queue.notification.arn }]
  })
}

# ── CloudWatch Log Groups ─────────────────────────────────────────────
resource "aws_cloudwatch_log_group" "services" {
  for_each          = toset(["user-service", "order-service", "notification-service"])
  name              = "/ecs/${each.value}"
  retention_in_days = 30
}

# ── RDS ───────────────────────────────────────────────────────────────
resource "aws_db_subnet_group" "main" {
  name       = "main-db-subnet-group"
  subnet_ids = aws_subnet.private[*].id
}

resource "aws_db_instance" "postgres" {
  identifier             = "${var.environment}-postgres"
  engine                 = "postgres"
  engine_version         = "15.4"
  instance_class         = "db.t3.micro"
  allocated_storage      = 20
  storage_encrypted      = true
  db_name                = "microservices"
  username               = "dbadmin"
  password               = var.db_password
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  multi_az               = var.environment == "production"
  backup_retention_period = 7
  deletion_protection    = var.environment == "production"
  skip_final_snapshot    = var.environment != "production"
  publicly_accessible    = false
}

resource "aws_secretsmanager_secret"         "db" { name = "/${var.environment}/database/credentials" }
resource "aws_secretsmanager_secret_version" "db" {
  secret_id     = aws_secretsmanager_secret.db.id
  secret_string = jsonencode({
    host     = aws_db_instance.postgres.address
    port     = 5432
    dbname   = "microservices"
    username = "dbadmin"
    password = var.db_password
  })
}

# ── Outputs ───────────────────────────────────────────────────────────
output "alb_dns_name"   { value = aws_lb.main.dns_name }
output "cluster_name"   { value = aws_ecs_cluster.main.name }
output "ecr_urls"       { value = { for k, v in aws_ecr_repository.services : k => v.repository_url } }
output "queue_url"      { value = aws_sqs_queue.notification.url }
output "db_secret_arn"  { value = aws_secretsmanager_secret.db.arn; sensitive = true }
```

### infrastructure/terraform/variables.tf

```hcl
variable "environment" { default = "staging" }
variable "aws_region"  { default = "us-east-1" }
variable "db_password" { type = string; sensitive = true }
```

---

## GitHub Actions Workflow (user-service)

### .github/workflows/user-service.yml

```yaml
name: User Service — CI/CD

on:
  push:
    branches: [main]
    paths:
      - 'services/user-service/**'
      - '.github/workflows/user-service.yml'
  pull_request:
    paths:
      - 'services/user-service/**'

env:
  AWS_REGION:     us-east-1
  ECR_REPO:       user-service
  ECS_CLUSTER:    production-cluster
  ECS_SERVICE:    user-service
  CONTAINER_NAME: user-service

jobs:
  # ────────────────────────────────────────────────
  test:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: services/user-service
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v4
        with:
          python-version: '3.12'
          cache: pip
          cache-dependency-path: services/user-service/requirements*.txt

      - run: pip install -r requirements.txt -r requirements-dev.txt

      - name: Run tests with coverage
        run: python -m pytest tests/ -v --cov=src --cov-fail-under=70

      - name: Lint
        run: flake8 src/ --max-line-length=120

  # ────────────────────────────────────────────────
  security:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
      - name: Trivy filesystem scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type:  fs
          scan-ref:   services/user-service
          severity:   CRITICAL,HIGH
          exit-code:  '1'

  # ────────────────────────────────────────────────
  build:
    runs-on: ubuntu-latest
    needs: [test, security]
    if: github.ref == 'refs/heads/main'
    outputs:
      image: ${{ steps.push.outputs.image }}

    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id:     ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region:            ${{ env.AWS_REGION }}

      - id: login
        uses: aws-actions/amazon-ecr-login@v2

      - id: push
        env:
          REGISTRY: ${{ steps.login.outputs.registry }}
          TAG:      ${{ github.sha }}
        run: |
          cd services/user-service
          docker build \
            --cache-from $REGISTRY/$ECR_REPO:latest \
            --build-arg BUILDKIT_INLINE_CACHE=1 \
            -t $REGISTRY/$ECR_REPO:$TAG \
            -t $REGISTRY/$ECR_REPO:latest .
          docker push $REGISTRY/$ECR_REPO:$TAG
          docker push $REGISTRY/$ECR_REPO:latest
          echo "image=$REGISTRY/$ECR_REPO:$TAG" >> $GITHUB_OUTPUT

  # ────────────────────────────────────────────────
  deploy:
    runs-on: ubuntu-latest
    needs: build
    environment: production        # requires manual approval in GitHub

    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id:     ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region:            ${{ env.AWS_REGION }}

      - id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: infrastructure/task-definitions/user-service.json
          container-name:  ${{ env.CONTAINER_NAME }}
          image:           ${{ needs.build.outputs.image }}

      - name: Deploy to ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition:          ${{ steps.task-def.outputs.task-definition }}
          service:                  ${{ env.ECS_SERVICE }}
          cluster:                  ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true

      - name: Notify success
        if: success()
        run: |
          curl -X POST "${{ secrets.SLACK_WEBHOOK_URL }}" \
            -H 'Content-type: application/json' \
            --data '{"text":"✅ *user-service* deployed to production (`${{ github.sha }}`)"}'

      - name: Notify failure
        if: failure()
        run: |
          curl -X POST "${{ secrets.SLACK_WEBHOOK_URL }}" \
            -H 'Content-type: application/json' \
            --data '{"text":"❌ *user-service* deployment FAILED (`${{ github.sha }}`)"}'
```

---

## Docker Compose (Local Dev)

```yaml
# docker-compose.yml
version: '3.9'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB:       microservices
      POSTGRES_USER:     dbadmin
      POSTGRES_PASSWORD: localpassword
    ports:
      - "5432:5432"
    volumes:
      - pg_data:/var/lib/postgresql/data

  localstack:
    image: localstack/localstack:latest
    environment:
      SERVICES: sqs,ses,secretsmanager
    ports:
      - "4566:4566"

  user-service:
    build: ./services/user-service
    ports:
      - "3001:3001"
    environment:
      DB_SECRET_ARN: "local-secret"
      DB_HOST:       postgres
      DB_NAME:       microservices
      DB_USER:       dbadmin
      DB_PASSWORD:   localpassword
      LOG_LEVEL:     DEBUG
      AWS_ENDPOINT:  http://localstack:4566
    depends_on:
      - postgres
      - localstack

  order-service:
    build: ./services/order-service
    ports:
      - "3002:3002"
    environment:
      DB_SECRET_ARN:          "local-secret"
      NOTIFICATION_QUEUE_URL: http://localstack:4566/000000000000/notification-queue
      LOG_LEVEL:              DEBUG
    depends_on:
      - postgres
      - localstack

  notification-service:
    build: ./services/notification-service
    environment:
      NOTIFICATION_QUEUE_URL: http://localstack:4566/000000000000/notification-queue
      FROM_EMAIL:             noreply@example.com
      LOG_LEVEL:              DEBUG
    depends_on:
      - localstack

volumes:
  pg_data:
```

---

## Deployment Steps

```bash
# ── Step 1: Deploy infrastructure ────────────────────────────────────
cd infrastructure/terraform
terraform init
terraform apply -var="environment=production" -var="db_password=SuperSecure123!"

# ── Step 2: Capture outputs ───────────────────────────────────────────
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION=us-east-1

# ── Step 3: Build & push images ───────────────────────────────────────
aws ecr get-login-password --region $REGION | \
  docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com

for SVC in user-service order-service notification-service; do
  docker build -t ${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/${SVC}:latest \
    services/${SVC}/
  docker push ${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/${SVC}:latest
  echo "Pushed ${SVC}"
done

# ── Step 4: Create ECS services ───────────────────────────────────────
# (Use task definitions in infrastructure/task-definitions/)
aws ecs create-service \
  --cluster production-cluster \
  --service-name user-service \
  --task-definition user-service:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={
      subnets=[$(terraform output -raw private_subnet_ids)],
      securityGroups=[sg-ecs],
      assignPublicIp=DISABLED}" \
  --load-balancers "targetGroupArn=$(terraform output -raw user_service_tg_arn),containerName=user-service,containerPort=3001"

# ── Step 5: Test the deployment ───────────────────────────────────────
ALB=$(terraform output -raw alb_dns_name)

curl "http://${ALB}/users"                              # list users
curl -X POST "http://${ALB}/users" \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice","email":"alice@example.com"}'     # create user

curl -X POST "http://${ALB}/orders" \
  -H "Content-Type: application/json" \
  -d '{"userId":"USER_ID","items":[{"name":"Book","price":9.99,"quantity":1}]}'
```

---

## Verification Checklist

- [ ] ECR repositories created with image scanning enabled
- [ ] ECS cluster has Container Insights enabled
- [ ] All three services running with 2+ tasks
- [ ] ALB routes `/users*` → user-service, `/orders*` → order-service
- [ ] Services are in private subnets (no public IP)
- [ ] RDS not publicly accessible
- [ ] SQS DLQ configured for poison messages
- [ ] CloudWatch log groups with 30-day retention
- [ ] GitHub Actions pipeline passes all stages
- [ ] Manual approval gate blocks direct production push
- [ ] X-Ray traces visible across service calls
- [ ] Container Insights shows CPU/memory per service

---

*Solution for Project 3 — Containerized Microservices on ECS*
