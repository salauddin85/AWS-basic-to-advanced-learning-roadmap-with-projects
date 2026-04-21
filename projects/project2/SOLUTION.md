# Project 2: Solution — Serverless API with CI/CD Pipeline

## Complete Implementation

---

### Directory Setup

```bash
mkdir -p project-2-serverless-api/{src/{handlers,utils,models},tests/{unit,integration},infrastructure}
cd project-2-serverless-api
```

---

## Application Code

### src/utils/response.py

```python
import json

def success(data, status_code=200):
    return {
        'statusCode': status_code,
        'headers': {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*',
            'Access-Control-Allow-Headers': 'Content-Type,X-Api-Key',
            'Access-Control-Allow-Methods': 'OPTIONS,GET,POST,PUT,DELETE'
        },
        'body': json.dumps(data, default=str)
    }

def error(message, status_code=400):
    return {
        'statusCode': status_code,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps({'error': message})
    }

def validation_error(errors):
    return {
        'statusCode': 422,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps({'error': 'Validation failed', 'details': errors})
    }
```

---

### src/utils/dynamodb.py

```python
import boto3
import os

_tables = {}

def get_table(table_name):
    """Get a DynamoDB table resource with connection reuse."""
    if table_name not in _tables:
        dynamodb = boto3.resource('dynamodb', region_name=os.environ.get('AWS_REGION', 'us-east-1'))
        _tables[table_name] = dynamodb.Table(table_name)
    return _tables[table_name]
```

---

### src/utils/validation.py

```python
import re

def validate_user(data):
    errors = []
    if not data.get('name') or len(data['name'].strip()) < 2:
        errors.append('name must be at least 2 characters')
    if not data.get('email') or not re.match(r'^[^@]+@[^@]+\.[^@]+$', data['email']):
        errors.append('email must be a valid email address')
    return errors

def validate_order(data):
    errors = []
    if not data.get('items') or not isinstance(data['items'], list) or len(data['items']) == 0:
        errors.append('items must be a non-empty list')
    else:
        for i, item in enumerate(data['items']):
            if not item.get('name'):
                errors.append(f'items[{i}].name is required')
            if not isinstance(item.get('price'), (int, float)) or item['price'] <= 0:
                errors.append(f'items[{i}].price must be a positive number')
            if not isinstance(item.get('quantity'), int) or item['quantity'] <= 0:
                errors.append(f'items[{i}].quantity must be a positive integer')
    return errors
```

---

### src/handlers/users.py

```python
import json
import os
import uuid
import logging
from datetime import datetime
from boto3.dynamodb.conditions import Key
from utils.dynamodb import get_table
from utils.response import success, error, validation_error
from utils.validation import validate_user

logger = logging.getLogger()
logger.setLevel(os.environ.get('LOG_LEVEL', 'INFO'))

users_table = get_table(os.environ.get('USERS_TABLE', 'users-staging'))


def create_user(event, context):
    try:
        body = json.loads(event.get('body') or '{}')
        errors = validate_user(body)
        if errors:
            return validation_error(errors)

        user = {
            'userId':    str(uuid.uuid4()),
            'name':      body['name'].strip(),
            'email':     body['email'].lower().strip(),
            'status':    'ACTIVE',
            'createdAt': datetime.utcnow().isoformat(),
            'updatedAt': datetime.utcnow().isoformat()
        }

        users_table.put_item(
            Item=user,
            ConditionExpression='attribute_not_exists(userId)'
        )

        logger.info(json.dumps({'event': 'user_created', 'userId': user['userId']}))
        return success(user, 201)

    except json.JSONDecodeError:
        return error('Invalid JSON body', 400)
    except Exception as e:
        logger.error(f'Error creating user: {e}', exc_info=True)
        return error('Internal server error', 500)


def get_user(event, context):
    try:
        user_id = event.get('pathParameters', {}).get('userId')
        if not user_id:
            return error('userId is required', 400)

        response = users_table.get_item(Key={'userId': user_id})
        item = response.get('Item')

        if not item:
            return error('User not found', 404)

        return success(item)

    except Exception as e:
        logger.error(f'Error getting user: {e}', exc_info=True)
        return error('Internal server error', 500)


def list_users(event, context):
    try:
        response = users_table.scan(Limit=50)
        return success({
            'users': response.get('Items', []),
            'count': response.get('Count', 0)
        })
    except Exception as e:
        logger.error(f'Error listing users: {e}', exc_info=True)
        return error('Internal server error', 500)


def delete_user(event, context):
    try:
        user_id = event.get('pathParameters', {}).get('userId')
        users_table.delete_item(Key={'userId': user_id})
        logger.info(json.dumps({'event': 'user_deleted', 'userId': user_id}))
        return success({'message': 'User deleted'})
    except Exception as e:
        logger.error(f'Error deleting user: {e}', exc_info=True)
        return error('Internal server error', 500)
```

---

### src/handlers/orders.py

```python
import json
import os
import uuid
import logging
from decimal import Decimal
from datetime import datetime
from boto3.dynamodb.conditions import Key
from utils.dynamodb import get_table
from utils.response import success, error, validation_error
from utils.validation import validate_order

logger = logging.getLogger()
logger.setLevel(os.environ.get('LOG_LEVEL', 'INFO'))

orders_table = get_table(os.environ.get('ORDERS_TABLE', 'orders-staging'))


def create_order(event, context):
    try:
        body = json.loads(event.get('body') or '{}')
        user_id = event.get('pathParameters', {}).get('userId') or body.get('userId')

        if not user_id:
            return error('userId is required', 400)

        errors = validate_order(body)
        if errors:
            return validation_error(errors)

        total = sum(
            Decimal(str(item['price'])) * item['quantity']
            for item in body['items']
        )

        order = {
            'userId':    user_id,
            'orderId':   str(uuid.uuid4()),
            'status':    'PENDING',
            'items':     body['items'],
            'total':     total,
            'createdAt': datetime.utcnow().isoformat(),
            'updatedAt': datetime.utcnow().isoformat()
        }

        orders_table.put_item(Item=order)
        logger.info(json.dumps({'event': 'order_created', 'orderId': order['orderId'], 'userId': user_id}))
        return success(order, 201)

    except json.JSONDecodeError:
        return error('Invalid JSON body', 400)
    except Exception as e:
        logger.error(f'Error creating order: {e}', exc_info=True)
        return error('Internal server error', 500)


def get_orders_by_user(event, context):
    try:
        user_id = event.get('pathParameters', {}).get('userId')
        if not user_id:
            return error('userId is required', 400)

        response = orders_table.query(
            KeyConditionExpression=Key('userId').eq(user_id)
        )
        return success({'orders': response.get('Items', []), 'count': response.get('Count', 0)})

    except Exception as e:
        logger.error(f'Error getting orders: {e}', exc_info=True)
        return error('Internal server error', 500)


def update_order_status(event, context):
    try:
        user_id  = event['pathParameters']['userId']
        order_id = event['pathParameters']['orderId']
        body     = json.loads(event.get('body') or '{}')
        new_status = body.get('status')

        allowed = ['PENDING', 'PROCESSING', 'SHIPPED', 'DELIVERED', 'CANCELLED']
        if new_status not in allowed:
            return error(f'status must be one of: {", ".join(allowed)}', 400)

        orders_table.update_item(
            Key={'userId': user_id, 'orderId': order_id},
            UpdateExpression='SET #s = :status, updatedAt = :ts',
            ExpressionAttributeNames={'#s': 'status'},
            ExpressionAttributeValues={
                ':status': new_status,
                ':ts':     datetime.utcnow().isoformat()
            }
        )
        return success({'message': 'Order status updated', 'status': new_status})

    except Exception as e:
        logger.error(f'Error updating order: {e}', exc_info=True)
        return error('Internal server error', 500)
```

---

### src/handlers/health.py

```python
import json
import os
import boto3
from utils.response import success, error

def health_check(event, context):
    checks = {'status': 'healthy', 'checks': {}}

    # DynamoDB reachability
    try:
        dynamodb = boto3.client('dynamodb')
        dynamodb.describe_table(TableName=os.environ.get('USERS_TABLE', 'users-staging'))
        checks['checks']['dynamodb'] = 'ok'
    except Exception as e:
        checks['checks']['dynamodb'] = f'error: {str(e)}'
        checks['status'] = 'degraded'

    status_code = 200 if checks['status'] == 'healthy' else 503
    return {
        'statusCode': status_code,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps(checks)
    }
```

---

## Tests

### tests/unit/test_users.py

```python
import json
import pytest
from unittest.mock import patch, MagicMock
import sys, os
sys.path.insert(0, os.path.join(os.path.dirname(__file__), '../../src'))

@pytest.fixture
def mock_table():
    with patch('handlers.users.users_table') as mock:
        yield mock

def test_create_user_success(mock_table):
    from handlers.users import create_user
    mock_table.put_item.return_value = {}

    event = {
        'body': json.dumps({'name': 'Alice', 'email': 'alice@example.com'})
    }
    response = create_user(event, {})

    assert response['statusCode'] == 201
    body = json.loads(response['body'])
    assert body['name'] == 'Alice'
    assert body['email'] == 'alice@example.com'
    assert 'userId' in body

def test_create_user_invalid_email(mock_table):
    from handlers.users import create_user
    event = {'body': json.dumps({'name': 'Alice', 'email': 'not-an-email'})}
    response = create_user(event, {})
    assert response['statusCode'] == 422

def test_create_user_missing_name(mock_table):
    from handlers.users import create_user
    event = {'body': json.dumps({'email': 'alice@example.com'})}
    response = create_user(event, {})
    assert response['statusCode'] == 422

def test_get_user_found(mock_table):
    from handlers.users import get_user
    mock_table.get_item.return_value = {
        'Item': {'userId': 'u-001', 'name': 'Alice', 'email': 'alice@example.com'}
    }
    event = {'pathParameters': {'userId': 'u-001'}}
    response = get_user(event, {})
    assert response['statusCode'] == 200

def test_get_user_not_found(mock_table):
    from handlers.users import get_user
    mock_table.get_item.return_value = {}
    event = {'pathParameters': {'userId': 'nonexistent'}}
    response = get_user(event, {})
    assert response['statusCode'] == 404

def test_create_user_invalid_json(mock_table):
    from handlers.users import create_user
    event = {'body': '{invalid json'}
    response = create_user(event, {})
    assert response['statusCode'] == 400
```

---

### tests/unit/test_orders.py

```python
import json
import pytest
from unittest.mock import patch, MagicMock
from decimal import Decimal
import sys, os
sys.path.insert(0, os.path.join(os.path.dirname(__file__), '../../src'))

@pytest.fixture
def mock_table():
    with patch('handlers.orders.orders_table') as mock:
        yield mock

def test_create_order_success(mock_table):
    from handlers.orders import create_order
    mock_table.put_item.return_value = {}

    event = {
        'pathParameters': {'userId': 'u-001'},
        'body': json.dumps({
            'items': [
                {'name': 'Widget', 'price': 9.99, 'quantity': 2},
                {'name': 'Gadget', 'price': 24.99, 'quantity': 1}
            ]
        })
    }
    response = create_order(event, {})
    assert response['statusCode'] == 201
    body = json.loads(response['body'])
    assert body['status'] == 'PENDING'
    assert 'orderId' in body

def test_create_order_empty_items(mock_table):
    from handlers.orders import create_order
    event = {
        'pathParameters': {'userId': 'u-001'},
        'body': json.dumps({'items': []})
    }
    response = create_order(event, {})
    assert response['statusCode'] == 422

def test_create_order_invalid_price(mock_table):
    from handlers.orders import create_order
    event = {
        'pathParameters': {'userId': 'u-001'},
        'body': json.dumps({'items': [{'name': 'Widget', 'price': -5, 'quantity': 1}]})
    }
    response = create_order(event, {})
    assert response['statusCode'] == 422
```

---

## SAM Template

### infrastructure/template.yaml

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Serverless API - intern portfolio project

Globals:
  Function:
    Runtime: python3.12
    Timeout: 30
    MemorySize: 256
    Tracing: Active
    Environment:
      Variables:
        USERS_TABLE:  !Ref UsersTable
        ORDERS_TABLE: !Ref OrdersTable
        LOG_LEVEL:    INFO

Parameters:
  Environment:
    Type: String
    AllowedValues: [staging, production]
    Default: staging

Resources:

  # ── API Gateway ──────────────────────────────────────────────────────
  InternApi:
    Type: AWS::Serverless::Api
    Properties:
      Name:         !Sub 'intern-api-${Environment}'
      StageName:    !Ref Environment
      TracingEnabled: true
      MethodSettings:
        - ResourcePath: '/*'
          HttpMethod:   '*'
          MetricsEnabled: true

  # ── Lambda Functions ─────────────────────────────────────────────────
  CreateUserFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: !Sub 'create-user-${Environment}'
      Handler:      handlers.users.create_user
      CodeUri:      ../src/
      Role:         !GetAtt LambdaRole.Arn
      Events:
        Api:
          Type: Api
          Properties:
            RestApiId: !Ref InternApi
            Path:      /users
            Method:    POST

  GetUserFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: !Sub 'get-user-${Environment}'
      Handler:      handlers.users.get_user
      CodeUri:      ../src/
      Role:         !GetAtt LambdaRole.Arn
      Events:
        Api:
          Type: Api
          Properties:
            RestApiId: !Ref InternApi
            Path:      /users/{userId}
            Method:    GET

  CreateOrderFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: !Sub 'create-order-${Environment}'
      Handler:      handlers.orders.create_order
      CodeUri:      ../src/
      Role:         !GetAtt LambdaRole.Arn
      ReservedConcurrentExecutions: 50
      Events:
        Api:
          Type: Api
          Properties:
            RestApiId: !Ref InternApi
            Path:      /users/{userId}/orders
            Method:    POST

  GetOrdersFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: !Sub 'get-orders-${Environment}'
      Handler:      handlers.orders.get_orders_by_user
      CodeUri:      ../src/
      Role:         !GetAtt LambdaRole.Arn
      Events:
        Api:
          Type: Api
          Properties:
            RestApiId: !Ref InternApi
            Path:      /users/{userId}/orders
            Method:    GET

  HealthFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: !Sub 'health-${Environment}'
      Handler:      handlers.health.health_check
      CodeUri:      ../src/
      Role:         !GetAtt LambdaRole.Arn
      Events:
        Api:
          Type: Api
          Properties:
            RestApiId: !Ref InternApi
            Path:      /health
            Method:    GET

  # ── DynamoDB ─────────────────────────────────────────────────────────
  UsersTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName:    !Sub 'users-${Environment}'
      BillingMode:  PAY_PER_REQUEST
      SSESpecification:
        SSEEnabled: true
      PointInTimeRecoverySpecification:
        PointInTimeRecoveryEnabled: true
      AttributeDefinitions:
        - { AttributeName: userId, AttributeType: S }
      KeySchema:
        - { AttributeName: userId, KeyType: HASH }
      Tags:
        - { Key: Environment, Value: !Ref Environment }

  OrdersTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName:    !Sub 'orders-${Environment}'
      BillingMode:  PAY_PER_REQUEST
      SSESpecification:
        SSEEnabled: true
      PointInTimeRecoverySpecification:
        PointInTimeRecoveryEnabled: true
      AttributeDefinitions:
        - { AttributeName: userId,  AttributeType: S }
        - { AttributeName: orderId, AttributeType: S }
        - { AttributeName: status,  AttributeType: S }
      KeySchema:
        - { AttributeName: userId,  KeyType: HASH }
        - { AttributeName: orderId, KeyType: RANGE }
      GlobalSecondaryIndexes:
        - IndexName: status-index
          KeySchema:
            - { AttributeName: status, KeyType: HASH }
          Projection:
            ProjectionType: ALL
      Tags:
        - { Key: Environment, Value: !Ref Environment }

  # ── IAM ──────────────────────────────────────────────────────────────
  LambdaRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub 'intern-lambda-role-${Environment}'
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal: { Service: lambda.amazonaws.com }
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

  # ── CloudWatch Alarms ────────────────────────────────────────────────
  LambdaErrorAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName:          !Sub 'lambda-errors-${Environment}'
      MetricName:         Errors
      Namespace:          AWS/Lambda
      Statistic:          Sum
      Period:             300
      EvaluationPeriods:  1
      Threshold:          5
      ComparisonOperator: GreaterThanThreshold

Outputs:
  ApiUrl:
    Description: API Gateway URL
    Value: !Sub 'https://${InternApi}.execute-api.${AWS::Region}.amazonaws.com/${Environment}'
  UsersTableName:
    Value: !Ref UsersTable
  OrdersTableName:
    Value: !Ref OrdersTable
```

---

## buildspec.yml

```yaml
version: 0.2

env:
  variables:
    SAM_BUCKET: your-sam-artifacts-bucket

phases:
  install:
    runtime-versions:
      python: 3.12
    commands:
      - pip install --upgrade pip
      - pip install aws-sam-cli
      - pip install -r requirements-dev.txt

  pre_build:
    commands:
      - echo "=== Unit Tests ==="
      - python -m pytest tests/unit/ -v --junitxml=reports/unit-tests.xml --cov=src --cov-report=xml:reports/coverage.xml
      - echo "=== Lint ==="
      - flake8 src/ --max-line-length=120 --ignore=E501
      - echo "=== Security scan ==="
      - bandit -r src/ -ll -f json -o reports/bandit.json || true

  build:
    commands:
      - echo "=== SAM Build ==="
      - sam build --template-file infrastructure/template.yaml
      - echo "=== SAM Package ==="
      - sam package
          --template-file .aws-sam/build/template.yaml
          --s3-bucket $SAM_BUCKET
          --s3-prefix $CODEBUILD_BUILD_ID
          --output-template-file packaged.yaml

  post_build:
    commands:
      - echo "Build completed — $(date)"

artifacts:
  files:
    - packaged.yaml
    - samconfig.toml
  discard-paths: yes

reports:
  UnitTestReport:
    files: [ reports/unit-tests.xml ]
    file-format: JUNITXML
  CoverageReport:
    files: [ reports/coverage.xml ]
    file-format: COBERTURAXML

cache:
  paths:
    - /root/.cache/pip/**/*
```

---

## samconfig.toml

```toml
version = 0.1

[default.global.parameters]
stack_name = "intern-serverless-api"

[default.deploy.parameters]
capabilities       = "CAPABILITY_IAM CAPABILITY_NAMED_IAM"
confirm_changeset  = true
resolve_s3         = true

[staging.deploy.parameters]
stack_name         = "intern-serverless-api-staging"
s3_bucket          = "your-sam-artifacts-bucket"
region             = "us-east-1"
parameter_overrides = "Environment=staging"
confirm_changeset  = false

[production.deploy.parameters]
stack_name         = "intern-serverless-api-production"
s3_bucket          = "your-sam-artifacts-bucket"
region             = "us-east-1"
parameter_overrides = "Environment=production"
confirm_changeset  = true
```

---

## requirements.txt

```
boto3>=1.34.0
aws-xray-sdk>=2.12.0
```

## requirements-dev.txt

```
pytest>=7.4.0
pytest-cov>=4.1.0
moto[dynamodb]>=4.2.0
flake8>=6.1.0
bandit>=1.7.5
black>=23.0.0
aws-sam-cli>=1.100.0
```

---

## Deployment & Verification

```bash
# 1. Create S3 bucket for SAM artifacts
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
aws s3 mb s3://sam-artifacts-${ACCOUNT_ID}

# 2. Build & deploy to staging
sam build --template-file infrastructure/template.yaml
sam deploy --config-env staging

# 3. Run tests against staging
API_URL=$(aws cloudformation describe-stacks \
  --stack-name intern-serverless-api-staging \
  --query 'Stacks[0].Outputs[?OutputKey==`ApiUrl`].OutputValue' \
  --output text)

# 4. Health check
curl "$API_URL/health"
# {"status": "healthy", "checks": {"dynamodb": "ok"}}

# 5. Create a user
curl -X POST "$API_URL/users" \
  -H "Content-Type: application/json" \
  -d '{"name": "Jane Smith", "email": "jane@example.com"}'

# 6. Get user
curl "$API_URL/users/USER_ID_FROM_ABOVE"

# 7. Create order
curl -X POST "$API_URL/users/USER_ID/orders" \
  -H "Content-Type: application/json" \
  -d '{"items": [{"name": "Book", "price": 14.99, "quantity": 1}]}'

# 8. Deploy to production after review
sam deploy --config-env production
```

---

## Verification Checklist

- [ ] All unit tests pass (6+ test cases)
- [ ] SAM build completes without errors
- [ ] Staging stack deployed successfully
- [ ] Health endpoint returns `{"status": "healthy"}`
- [ ] Create user returns HTTP 201 with userId
- [ ] Create order returns HTTP 201 with orderId
- [ ] DynamoDB tables have items visible in console
- [ ] CloudWatch logs show structured JSON logs
- [ ] X-Ray traces visible in console
- [ ] Lambda errors alarm is configured
- [ ] Production deploy requires confirmation

---

*Solution for Project 2 — Serverless API with CI/CD*
