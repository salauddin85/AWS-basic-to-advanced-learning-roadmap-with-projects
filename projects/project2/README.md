# Project 2: Serverless API with CI/CD Pipeline
### Resume Project | AWS DevOps Portfolio

---

## 🏆 Project Overview

Build and deploy a **fully serverless REST API** with a complete **CI/CD pipeline** that
automatically tests, builds, and deploys on every code push. This project demonstrates
modern serverless architecture and DevOps automation — highly valued skills in today's market.

**What You'll Build:**
A production-ready serverless API using Lambda + API Gateway + DynamoDB, with automated
deployment via CodePipeline and comprehensive monitoring. Zero servers to manage.

---

## 🛠️ Technologies & AWS Services Used

- **Compute:** AWS Lambda (Python 3.12)
- **API:** Amazon API Gateway (REST API with usage plans)
- **Database:** Amazon DynamoDB (with GSIs)
- **Storage:** Amazon S3 (Lambda deployment packages)
- **Security:** IAM Roles, API Keys, AWS WAF
- **Secrets:** AWS Secrets Manager
- **CI/CD:** AWS CodePipeline, CodeBuild, CodeCommit
- **Monitoring:** CloudWatch Logs, X-Ray tracing, CloudWatch Alarms
- **Notifications:** Amazon SNS
- **IaC:** AWS SAM (Serverless Application Model) + Terraform

---

## 🏗️ Architecture

```
GitHub/CodeCommit
       │
  [CodePipeline]
  ├── Stage 1: Source
  ├── Stage 2: Test (CodeBuild → pytest)
  ├── Stage 3: Build (SAM package)
  ├── Stage 4: Deploy to Staging
  ├── Stage 5: Integration Tests
  ├── Stage 6: Manual Approval
  └── Stage 7: Deploy to Production
              │
    [API Gateway] ← HTTPS endpoint
    /users  /orders  /health
              │
        [AWS Lambda]
    ┌─────────────────────┐
    │  get_user           │  ← Reads from DynamoDB
    │  create_order       │  ← Writes to DynamoDB + sends SNS
    │  list_orders        │  ← Queries DynamoDB GSI
    │  health_check       │  ← Returns 200 OK
    └─────────────────────┘
              │
    [DynamoDB Tables]
    ┌─────────────┐  ┌──────────────┐
    │   Users     │  │   Orders     │
    │ userId (PK) │  │ userId (PK)  │
    │ email       │  │ orderId (SK) │
    │ name        │  │ status       │
    └─────────────┘  │ total        │
                     └──────────────┘
                     GSI: status-index
```

---

## 📁 Project Structure

```
project-2-serverless-api/
├── src/
│   ├── handlers/
│   │   ├── users.py         # User CRUD operations
│   │   ├── orders.py        # Order management
│   │   └── health.py        # Health check endpoint
│   ├── utils/
│   │   ├── dynamodb.py      # DynamoDB helper functions
│   │   ├── response.py      # Standardized API responses
│   │   └── validation.py    # Input validation
│   └── models/
│       ├── user.py
│       └── order.py
├── tests/
│   ├── unit/
│   │   ├── test_users.py
│   │   └── test_orders.py
│   └── integration/
│       └── test_api.py
├── infrastructure/
│   ├── template.yaml        # SAM template (Lambda + API Gateway + DynamoDB)
│   └── pipeline.tf          # Terraform for CodePipeline
├── buildspec.yml            # CodeBuild configuration
├── samconfig.toml           # SAM configuration
├── requirements.txt
├── requirements-dev.txt
└── README.md
```

---

## 🔧 Application Code

### SAM Template

```yaml
# infrastructure/template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Serverless API — intern portfolio project

Globals:
  Function:
    Runtime: python3.12
    Timeout: 30
    MemorySize: 256
    Tracing: Active  # X-Ray enabled
    Environment:
      Variables:
        USERS_TABLE: !Ref UsersTable
        ORDERS_TABLE: !Ref OrdersTable
        LOG_LEVEL: INFO
    Layers:
      - !Ref DependenciesLayer

Parameters:
  Environment:
    Type: String
    AllowedValues: [staging, production]

Resources:
  # API Gateway
  InternApi:
    Type: AWS::Serverless::Api
    Properties:
      Name: !Sub 'intern-api-${Environment}'
      StageName: !Ref Environment
      TracingEnabled: true
      Auth:
        ApiKeyRequired: true
        UsagePlan:
          CreateUsagePlan: PER_API
          Quota:
            Limit: 1000
            Period: DAY
          Throttle:
            BurstLimit: 100
            RateLimit: 50
      AccessLogDestination:
        DestinationArn: !GetAtt ApiAccessLogGroup.Arn
      MethodSettings:
        - ResourcePath: '/*'
          HttpMethod: '*'
          MetricsEnabled: true
          DataTraceEnabled: true

  # Lambda Functions
  GetUserFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: !Sub 'get-user-${Environment}'
      Handler: handlers.users.get_user
      CodeUri: src/
      Role: !GetAtt LambdaExecutionRole.Arn
      Events:
        GetUser:
          Type: Api
          Properties:
            RestApiId: !Ref InternApi
            Path: /users/{userId}
            Method: GET

  CreateOrderFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: !Sub 'create-order-${Environment}'
      Handler: handlers.orders.create_order
      CodeUri: src/
      Role: !GetAtt LambdaExecutionRole.Arn
      ReservedConcurrentExecutions: 50
      Events:
        CreateOrder:
          Type: Api
          Properties:
            RestApiId: !Ref InternApi
            Path: /orders
            Method: POST

  # DynamoDB Tables
  UsersTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: !Sub 'users-${Environment}'
      BillingMode: PAY_PER_REQUEST
      TableClass: STANDARD
      PointInTimeRecoverySpecification:
        PointInTimeRecoveryEnabled: true
      SSESpecification:
        SSEEnabled: true
      AttributeDefinitions:
        - AttributeName: userId
          AttributeType: S
      KeySchema:
        - AttributeName: userId
          KeyType: HASH
      Tags:
        - Key: Environment
          Value: !Ref Environment

  OrdersTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: !Sub 'orders-${Environment}'
      BillingMode: PAY_PER_REQUEST
      PointInTimeRecoverySpecification:
        PointInTimeRecoveryEnabled: true
      SSESpecification:
        SSEEnabled: true
      AttributeDefinitions:
        - AttributeName: userId
          AttributeType: S
        - AttributeName: orderId
          AttributeType: S
        - AttributeName: status
          AttributeType: S
      KeySchema:
        - AttributeName: userId
          KeyType: HASH
        - AttributeName: orderId
          KeyType: RANGE
      GlobalSecondaryIndexes:
        - IndexName: status-index
          KeySchema:
            - AttributeName: status
              KeyType: HASH
          Projection:
            ProjectionType: ALL

  # CloudWatch Log Groups
  ApiAccessLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub '/api/${Environment}/access-logs'
      RetentionInDays: 30

  # IAM Role
  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
        - arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess
      Policies:
        - PolicyName: DynamoDBAccess
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - dynamodb:GetItem
                  - dynamodb:PutItem
                  - dynamodb:UpdateItem
                  - dynamodb:DeleteItem
                  - dynamodb:Query
                  - dynamodb:Scan
                Resource:
                  - !GetAtt UsersTable.Arn
                  - !GetAtt OrdersTable.Arn
                  - !Sub '${OrdersTable.Arn}/index/*'

Outputs:
  ApiUrl:
    Description: API Gateway endpoint URL
    Value: !Sub 'https://${InternApi}.execute-api.${AWS::Region}.amazonaws.com/${Environment}'
  ApiKey:
    Description: API Key for authentication
    Value: !Ref InternApi
```

---

### Lambda Handler Example

```python
# src/handlers/orders.py
import json
import os
import uuid
import logging
from datetime import datetime
from utils.dynamodb import get_table
from utils.response import success, error, validation_error
from utils.validation import validate_order

logger = logging.getLogger()
logger.setLevel(os.environ.get('LOG_LEVEL', 'INFO'))

orders_table = get_table(os.environ['ORDERS_TABLE'])

def create_order(event, context):
    """
    POST /orders
    Creates a new order for a user.
    """
    try:
        # Parse body
        body = json.loads(event.get('body', '{}'))
        user_id = event.get('pathParameters', {}).get('userId') or body.get('userId')

        # Validate input
        errors = validate_order(body)
        if errors:
            return validation_error(errors)

        # Create order
        order = {
            'userId': user_id,
            'orderId': str(uuid.uuid4()),
            'status': 'PENDING',
            'items': body['items'],
            'total': sum(item['price'] * item['quantity'] for item in body['items']),
            'createdAt': datetime.utcnow().isoformat(),
            'updatedAt': datetime.utcnow().isoformat()
        }

        orders_table.put_item(Item=order)

        logger.info(f"Order created: {order['orderId']} for user: {user_id}")

        return success(order, status_code=201)

    except json.JSONDecodeError:
        return error("Invalid JSON in request body", 400)
    except Exception as e:
        logger.error(f"Error creating order: {str(e)}", exc_info=True)
        return error("Internal server error", 500)
```

---

## 🔁 CI/CD Pipeline

### buildspec.yml

```yaml
version: 0.2

env:
  variables:
    SAM_BUCKET: your-sam-deployment-bucket

phases:
  install:
    runtime-versions:
      python: 3.12
    commands:
      - pip install --upgrade pip
      - pip install -r requirements-dev.txt
      - pip install aws-sam-cli

  pre_build:
    commands:
      - echo "Running unit tests..."
      - python -m pytest tests/unit/ -v --junitxml=test-results.xml
      - echo "Running linting..."
      - flake8 src/ --max-line-length=120
      - echo "Security scanning..."
      - bandit -r src/ -ll

  build:
    commands:
      - echo "Packaging SAM application..."
      - sam package \
          --template-file infrastructure/template.yaml \
          --s3-bucket $SAM_BUCKET \
          --output-template-file packaged.yaml

  post_build:
    commands:
      - echo "Build completed"

artifacts:
  files:
    - packaged.yaml
    - samconfig.toml
  discard-paths: yes

reports:
  unit-tests:
    files:
      - test-results.xml
    file-format: JUNITXML
```

---

## 🚀 Deployment

```bash
# Install SAM CLI
pip install aws-sam-cli

# Build
sam build

# Deploy to staging
sam deploy \
  --config-env staging \
  --parameter-overrides Environment=staging

# Deploy to production
sam deploy \
  --config-env production \
  --parameter-overrides Environment=production
```

---

## 🧪 Testing the API

```bash
# Get your API URL and key from SAM outputs
API_URL=$(aws cloudformation describe-stacks \
  --stack-name intern-serverless-api-production \
  --query 'Stacks[0].Outputs[?OutputKey==`ApiUrl`].OutputValue' \
  --output text)

# Create a user
curl -X POST "$API_URL/users" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "John Doe", "email": "john@example.com"}'

# Get user
curl "$API_URL/users/USER_ID" \
  -H "x-api-key: YOUR_API_KEY"

# Create order
curl -X POST "$API_URL/orders" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"userId": "USER_ID", "items": [{"name": "Book", "price": 9.99, "quantity": 2}]}'
```

---

## 📊 Monitoring Setup

The project configures X-Ray tracing on all Lambda functions and API Gateway,
enabling you to trace requests end-to-end and identify bottlenecks.

CloudWatch alarms are set for:
- Lambda errors > 1% in any 5-minute window
- Lambda duration P99 > 3 seconds
- API Gateway 5XX rate > 1%
- DynamoDB throttled requests > 0

---

## 💰 Estimated Cost

| Resource | Usage | Monthly Cost |
|----------|-------|-------------|
| Lambda | 1M requests × 256MB × 200ms | ~$0.83 |
| API Gateway | 1M requests | ~$3.50 |
| DynamoDB | On-demand, 1M reads/writes | ~$1.50 |
| CloudWatch | Logs + metrics | ~$2.00 |
| **Total** | 1M requests/month | **~$8/month** |

Serverless architecture costs near-zero at low traffic and scales economically to millions of requests.

---

## 🎯 What This Project Demonstrates (For Resume)

- Serverless architecture design and implementation
- AWS SAM proficiency (industry-standard serverless IaC)
- Full CI/CD pipeline with automated testing and approval gates
- Observability with distributed tracing (X-Ray)
- API security (API keys, WAF, throttling)
- DynamoDB data modeling (partition key, sort key, GSI design)

---

## 📄 License

MIT License — Free to use for learning and portfolio purposes.
