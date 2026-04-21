# Project 3: Containerized Microservices on ECS with GitOps
### Resume Project | AWS DevOps Portfolio

---

## 🏆 Project Overview

Deploy a **microservices application** on **Amazon ECS Fargate** using a GitOps-based
CI/CD workflow. This project demonstrates container orchestration, service mesh patterns,
and modern deployment strategies — skills that are highly valued at companies using
Docker and Kubernetes.

**What You'll Build:**
Three microservices (User Service, Order Service, Notification Service) deployed on
ECS Fargate with service discovery, centralized logging, distributed tracing, and
a fully automated deployment pipeline.

---

## 🛠️ Technologies & AWS Services Used

- **Containers:** Docker, Amazon ECR
- **Orchestration:** Amazon ECS Fargate
- **Load Balancing:** ALB with path-based routing
- **Service Discovery:** AWS Cloud Map
- **Networking:** VPC, Private Subnets, Security Groups
- **Database:** Amazon RDS PostgreSQL (per-service)
- **Cache:** Amazon ElastiCache Redis
- **Messaging:** Amazon SQS (async communication between services)
- **CI/CD:** GitHub Actions → ECR → ECS Rolling Deploy
- **Monitoring:** CloudWatch Container Insights, X-Ray
- **Security:** ECR image scanning, IAM task roles, Secrets Manager
- **IaC:** Terraform with modules

---

## 🏗️ Architecture

```
                    Internet
                       │
              [Application Load Balancer]
              (Public Subnets AZ-a, AZ-b)
                       │
         ┌─────────────┼──────────────┐
         │             │              │
   /users/*       /orders/*    /notifications/*
         │             │              │
  [User Service]  [Order Service] [Notification Service]
   ECS Fargate    ECS Fargate     ECS Fargate
   2 tasks         2 tasks         1 task
         │             │              │
  [RDS Users]   [RDS Orders]    [SQS Queue]
                      │              │
               ──── publishes ────►  │
               order.created event   │
                                     │
                             Sends email via SES

All services connect via Cloud Map:
  user-service.local:3001
  order-service.local:3002
  notification-service.local:3003
```

---

## 📁 Project Structure

```
project-3-microservices/
├── services/
│   ├── user-service/
│   │   ├── src/
│   │   │   ├── app.py
│   │   │   └── models/user.py
│   │   ├── Dockerfile
│   │   ├── requirements.txt
│   │   └── buildspec.yml
│   ├── order-service/
│   │   ├── src/
│   │   │   ├── app.py
│   │   │   └── models/order.py
│   │   ├── Dockerfile
│   │   ├── requirements.txt
│   │   └── buildspec.yml
│   └── notification-service/
│       ├── src/
│       │   ├── app.py
│       │   └── processors/
│       ├── Dockerfile
│       ├── requirements.txt
│       └── buildspec.yml
├── infrastructure/
│   ├── terraform/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── modules/
│   │       ├── ecs-cluster/
│   │       ├── ecs-service/
│   │       ├── rds/
│   │       └── sqs/
│   └── task-definitions/
│       ├── user-service.json
│       ├── order-service.json
│       └── notification-service.json
├── .github/
│   └── workflows/
│       ├── user-service.yml
│       ├── order-service.yml
│       └── notification-service.yml
├── docker-compose.yml       # Local development
└── README.md
```

---

## 🐳 Dockerfiles

### User Service Dockerfile

```dockerfile
# services/user-service/Dockerfile
FROM python:3.12-slim AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.12-slim AS runtime

# Security: Run as non-root
RUN groupadd --gid 1001 appgroup && \
    useradd --uid 1001 --gid appgroup --shell /bin/sh --no-create-home appuser

WORKDIR /app

# Copy dependencies from builder
COPY --from=builder /root/.local /home/appuser/.local

# Copy application
COPY --chown=appuser:appgroup src/ ./

USER appuser

ENV PATH=/home/appuser/.local/bin:$PATH
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

EXPOSE 3001

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:3001/health')"

CMD ["python", "-m", "gunicorn", "--bind", "0.0.0.0:3001", "--workers", "2", "app:app"]
```

---

## 🔧 ECS Task Definition

```json
{
  "family": "user-service",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::ACCOUNT:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::ACCOUNT:role/user-service-task-role",
  "containerDefinitions": [
    {
      "name": "user-service",
      "image": "ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/user-service:latest",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 3001,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {"name": "PORT", "value": "3001"},
        {"name": "SERVICE_NAME", "value": "user-service"},
        {"name": "LOG_LEVEL", "value": "INFO"}
      ],
      "secrets": [
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:ACCOUNT:secret:/production/user-service/database-url"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/user-service",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs",
          "awslogs-datetime-format": "%Y-%m-%dT%H:%M:%S"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:3001/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    },
    {
      "name": "xray-daemon",
      "image": "amazon/aws-xray-daemon",
      "essential": false,
      "portMappings": [
        {"containerPort": 2000, "protocol": "udp"}
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/xray",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "xray"
        }
      }
    }
  ]
}
```

---

## 🔁 GitHub Actions CI/CD

```yaml
# .github/workflows/user-service.yml
name: Deploy User Service

on:
  push:
    branches: [main]
    paths:
      - 'services/user-service/**'
      - '.github/workflows/user-service.yml'

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: user-service
  ECS_CLUSTER: production-cluster
  ECS_SERVICE: user-service
  CONTAINER_NAME: user-service

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.12'

      - name: Install dependencies
        run: |
          cd services/user-service
          pip install -r requirements.txt -r requirements-dev.txt

      - name: Run tests
        run: |
          cd services/user-service
          python -m pytest tests/ -v --cov=src --cov-report=xml

      - name: Upload coverage
        uses: codecov/codecov-action@v3

  security-scan:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: 'services/user-service'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

  build-and-push:
    runs-on: ubuntu-latest
    needs: [test, security-scan]
    outputs:
      image: ${{ steps.build-image.outputs.image }}

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, and push image
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          cd services/user-service
          docker build \
            --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
            --build-arg VCS_REF=${{ github.sha }} \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:latest \
            .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
          echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT

      - name: Scan ECR image for vulnerabilities
        uses: aws-actions/amazon-ecr-scan-image@v1
        with:
          repository: ${{ env.ECR_REPOSITORY }}
          tag: ${{ github.sha }}

  deploy:
    runs-on: ubuntu-latest
    needs: build-and-push
    environment: production

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Update ECS Task Definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: infrastructure/task-definitions/user-service.json
          container-name: ${{ env.CONTAINER_NAME }}
          image: ${{ needs.build-and-push.outputs.image }}

      - name: Deploy to ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
          codedeploy-appspec: appspec.yaml

      - name: Notify Slack on success
        if: success()
        uses: slackapi/slack-github-action@v1.24.0
        with:
          payload: |
            {"text": "✅ User Service deployed successfully to production (commit: ${{ github.sha }})"}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: Notify Slack on failure
        if: failure()
        uses: slackapi/slack-github-action@v1.24.0
        with:
          payload: |
            {"text": "❌ User Service deployment FAILED (commit: ${{ github.sha }})"}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## 📡 Service Communication (SQS Pattern)

```python
# Order service publishes event after creating order
import boto3
import json

sqs = boto3.client('sqs')
NOTIFICATION_QUEUE_URL = os.environ['NOTIFICATION_QUEUE_URL']

def create_order(data):
    # Save order to DB
    order = save_to_database(data)

    # Publish event to SQS (async, non-blocking)
    sqs.send_message(
        QueueUrl=NOTIFICATION_QUEUE_URL,
        MessageBody=json.dumps({
            'eventType': 'ORDER_CREATED',
            'orderId': order['orderId'],
            'userId': order['userId'],
            'total': str(order['total']),
            'timestamp': datetime.utcnow().isoformat()
        }),
        MessageAttributes={
            'eventType': {
                'StringValue': 'ORDER_CREATED',
                'DataType': 'String'
            }
        }
    )

    return order
```

```python
# Notification service consumes SQS messages
import boto3
import json

sqs = boto3.client('sqs')
ses = boto3.client('ses')

def process_notifications():
    while True:
        messages = sqs.receive_message(
            QueueUrl=NOTIFICATION_QUEUE_URL,
            MaxNumberOfMessages=10,
            WaitTimeSeconds=20  # Long polling
        )

        for message in messages.get('Messages', []):
            event = json.loads(message['Body'])

            if event['eventType'] == 'ORDER_CREATED':
                send_order_confirmation_email(event)

            # Delete processed message
            sqs.delete_message(
                QueueUrl=NOTIFICATION_QUEUE_URL,
                ReceiptHandle=message['ReceiptHandle']
            )
```

---

## 🔒 Security Architecture

```
Service-to-Service Authorization:
  - No inter-service calls allowed by default
  - Order Service → User Service: allowed via Cloud Map DNS
  - All calls within VPC only (no internet required)

IAM Task Roles (least privilege):
  user-service-task-role:
    - rds:Connect to user-service database only
    - secretsmanager:GetSecretValue for user-service secrets only

  order-service-task-role:
    - rds:Connect to order-service database only
    - sqs:SendMessage to notification-queue only

  notification-service-task-role:
    - sqs:ReceiveMessage, sqs:DeleteMessage
    - ses:SendEmail

ECR:
  - Image scanning on push enabled
  - Lifecycle policy: keep last 10 images, delete untagged after 1 day
  - Images signed with AWS Signer (optional)
```

---

## 🚀 Local Development

```bash
# docker-compose.yml for local development
docker-compose up

# Services available locally:
# User Service:         http://localhost:3001
# Order Service:        http://localhost:3002
# Notification Service: http://localhost:3003
```

---

## 📊 Monitoring & Observability

**Container Insights** provides per-service metrics:
- CPU and memory per task
- Network I/O per service
- Task count over time

**X-Ray Service Map** shows:
- Request flow between services
- Latency breakdown per service
- Error rates per service

**CloudWatch Log Insights** for cross-service queries:
```
# Find slow requests across all services
fields @timestamp, service, requestId, duration
| filter duration > 1000
| sort @timestamp desc
| limit 100
```

---

## 💰 Estimated Cost

| Resource | Config | Monthly Cost |
|----------|--------|-------------|
| ECS Fargate | 5 tasks × 0.25 vCPU × 0.5 GB | ~$18 |
| RDS (2 instances) | db.t3.micro each | ~$30 |
| ALB | Per LCU | ~$18 |
| SQS | 1M messages | ~$0.40 |
| ElastiCache | cache.t3.micro | ~$12 |
| NAT Gateway | Per GB | ~$32 |
| **Total** | | **~$110/month** |

---

## 🎯 What This Project Demonstrates (For Resume)

- Microservices architecture design and implementation
- Container best practices (multi-stage builds, non-root, health checks)
- ECS Fargate deployment and service management
- GitHub Actions CI/CD with automated testing, security scanning
- Asynchronous service communication via SQS
- Service discovery with Cloud Map
- Production observability (Container Insights, X-Ray, structured logging)
- Infrastructure as Code with Terraform modules

---

## 📄 License

MIT License — Free to use for learning and portfolio purposes.
