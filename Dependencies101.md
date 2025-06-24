**Complete Guide: AWS Testing Dependencies for Serverless CI/CD**

## **Overview of Testing Dependencies**

When building serverless applications, you need to test AWS integrations without hitting real AWS services every time. Here's the ecosystem:

```
Testing Pyramid for AWS Serverless:

┌─────────────────────────────────────────────┐
│             Production AWS                  │ ← Real AWS (prod/qa)
├─────────────────────────────────────────────┤
│             LocalStack                      │ ← Full AWS simulation
├─────────────────────────────────────────────┤
│               Moto                          │ ← AWS service mocking
├─────────────────────────────────────────────┤
│            Boto3 + Pytest                  │ ← Unit testing
└─────────────────────────────────────────────┘
```

## **Boto3: AWS SDK for Python**

**What is Boto3?**
Boto3 is the official AWS SDK for Python. It's how your Lambda functions and tests communicate with AWS services.

**Core Boto3 Concepts:**

**1. Client vs Resource:**
```python
import boto3

# Low-level client (more control, verbose)
dynamodb_client = boto3.client('dynamodb')
response = dynamodb_client.get_item(
    TableName='my-table',
    Key={
        'id': {'S': 'item-123'}
    }
)

# High-level resource (easier to use)
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('my-table')
response = table.get_item(Key={'id': 'item-123'})
```

**2. Session Management:**
```python
# Default session (uses default AWS credentials)
s3 = boto3.client('s3')

# Custom session with specific credentials
session = boto3.Session(
    aws_access_key_id='YOUR_ACCESS_KEY',
    aws_secret_access_key='YOUR_SECRET_KEY',
    region_name='us-east-1'
)
s3 = session.client('s3')

# Session with profile
session = boto3.Session(profile_name='dev-profile')
s3 = session.client('s3')
```

**3. Configuration Hierarchy:**
```
Boto3 looks for credentials in this order:
1. Explicitly passed parameters
2. Environment variables (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY)
3. AWS credentials file (~/.aws/credentials)
4. IAM roles (in EC2/Lambda)
5. IAM roles for tasks (in ECS)
```

**Common Boto3 Patterns for Lambda:**

**DynamoDB Operations:**
```python
import boto3
from botocore.exceptions import ClientError

class DatabaseHandler:
    def __init__(self):
        self.dynamodb = boto3.resource('dynamodb')
        self.table = self.dynamodb.Table('my-table')
    
    def get_item(self, item_id):
        try:
            response = self.table.get_item(Key={'id': item_id})
            return response.get('Item')
        except ClientError as e:
            print(f"Error getting item: {e}")
            return None
    
    def put_item(self, item_data):
        try:
            response = self.table.put_item(Item=item_data)
            return response
        except ClientError as e:
            print(f"Error putting item: {e}")
            raise
    
    def query_items(self, partition_key, sort_key_condition=None):
        try:
            if sort_key_condition:
                response = self.table.query(
                    KeyConditionExpression=Key('pk').eq(partition_key) & sort_key_condition
                )
            else:
                response = self.table.query(
                    KeyConditionExpression=Key('pk').eq(partition_key)
                )
            return response['Items']
        except ClientError as e:
            print(f"Error querying items: {e}")
            return []
```

**S3 Operations:**
```python
import boto3
import json
from botocore.exceptions import ClientError

class S3Handler:
    def __init__(self):
        self.s3 = boto3.client('s3')
    
    def upload_json(self, bucket, key, data):
        try:
            self.s3.put_object(
                Bucket=bucket,
                Key=key,
                Body=json.dumps(data),
                ContentType='application/json'
            )
            return True
        except ClientError as e:
            print(f"Error uploading to S3: {e}")
            return False
    
    def download_json(self, bucket, key):
        try:
            response = self.s3.get_object(Bucket=bucket, Key=key)
            content = response['Body'].read()
            return json.loads(content)
        except ClientError as e:
            print(f"Error downloading from S3: {e}")
            return None
    
    def generate_presigned_url(self, bucket, key, expiration=3600):
        try:
            url = self.s3.generate_presigned_url(
                'get_object',
                Params={'Bucket': bucket, 'Key': key},
                ExpiresIn=expiration
            )
            return url
        except ClientError as e:
            print(f"Error generating presigned URL: {e}")
            return None
```

**Error Handling with Boto3:**
```python
from botocore.exceptions import ClientError, NoCredentialsError, PartialCredentialsError

def robust_aws_operation():
    try:
        # Your AWS operation
        dynamodb = boto3.resource('dynamodb')
        table = dynamodb.Table('my-table')
        response = table.get_item(Key={'id': 'test'})
        return response['Item']
        
    except NoCredentialsError:
        print("AWS credentials not found")
        raise
    except PartialCredentialsError:
        print("Incomplete AWS credentials")
        raise
    except ClientError as e:
        error_code = e.response['Error']['Code']
        error_message = e.response['Error']['Message']
        
        if error_code == 'ResourceNotFoundException':
            print("Table or item not found")
            return None
        elif error_code == 'ValidationException':
            print(f"Invalid request: {error_message}")
            raise
        elif error_code == 'ThrottlingException':
            print("Request was throttled, implement retry logic")
            raise
        else:
            print(f"Unexpected error: {error_code} - {error_message}")
            raise
```

## **Moto: AWS Service Mocking Library**

**What is Moto?**
Moto is a library that mocks AWS services for testing. It intercepts boto3 calls and returns fake responses, allowing you to test AWS interactions without real AWS resources.

**Moto Installation and Basic Usage:**
```bash
pip install moto[all]  # Install all service mocks
# or
pip install moto[dynamodb,s3,lambda]  # Install specific services
```

**Moto Decorators:**
```python
import boto3
import pytest
from moto import mock_dynamodb, mock_s3, mock_lambda

# Decorator approach (recommended)
@mock_dynamodb
def test_dynamodb_operations():
    # Create mock DynamoDB
    dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
    
    # Create table
    table = dynamodb.create_table(
        TableName='test-table',
        KeySchema=[{'AttributeName': 'id', 'KeyType': 'HASH'}],
        AttributeDefinitions=[{'AttributeName': 'id', 'AttributeType': 'S'}],
        BillingMode='PAY_PER_REQUEST'
    )
    
    # Test operations
    table.put_item(Item={'id': 'test-123', 'data': 'test-data'})
    response = table.get_item(Key={'id': 'test-123'})
    
    assert response['Item']['data'] == 'test-data'

# Context manager approach
def test_s3_operations():
    with mock_s3():
        s3 = boto3.client('s3', region_name='us-east-1')
        
        # Create bucket
        s3.create_bucket(Bucket='test-bucket')
        
        # Upload object
        s3.put_object(
            Bucket='test-bucket',
            Key='test-file.txt',
            Body=b'test content'
        )
        
        # Test download
        response = s3.get_object(Bucket='test-bucket', Key='test-file.txt')
        content = response['Body'].read()
        
        assert content == b'test content'
```

**Advanced Moto Usage:**

**1. Complex DynamoDB Testing:**
```python
import boto3
import pytest
from moto import mock_dynamodb
from decimal import Decimal

@mock_dynamodb
class TestDynamoDBHandler:
    def setup_method(self):
        """Setup run before each test method"""
        self.dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
        
        # Create table with GSI
        self.table = self.dynamodb.create_table(
            TableName='users',
            KeySchema=[
                {'AttributeName': 'user_id', 'KeyType': 'HASH'},
                {'AttributeName': 'created_at', 'KeyType': 'RANGE'}
            ],
            AttributeDefinitions=[
                {'AttributeName': 'user_id', 'AttributeType': 'S'},
                {'AttributeName': 'created_at', 'AttributeType': 'S'},
                {'AttributeName': 'email', 'AttributeType': 'S'}
            ],
            GlobalSecondaryIndexes=[
                {
                    'IndexName': 'email-index',
                    'KeySchema': [{'AttributeName': 'email', 'KeyType': 'HASH'}],
                    'Projection': {'ProjectionType': 'ALL'}
                }
            ],
            BillingMode='PAY_PER_REQUEST'
        )
    
    def test_user_crud_operations(self):
        # Create user
        user_data = {
            'user_id': 'user-123',
            'created_at': '2024-06-24T10:00:00Z',
            'email': 'test@example.com',
            'name': 'Test User',
            'score': Decimal('99.5')
        }
        
        self.table.put_item(Item=user_data)
        
        # Read user
        response = self.table.get_item(
            Key={'user_id': 'user-123', 'created_at': '2024-06-24T10:00:00Z'}
        )
        assert response['Item']['email'] == 'test@example.com'
        
        # Update user
        self.table.update_item(
            Key={'user_id': 'user-123', 'created_at': '2024-06-24T10:00:00Z'},
            UpdateExpression='SET score = score + :increment',
            ExpressionAttributeValues={':increment': Decimal('0.5')}
        )
        
        # Verify update
        updated_item = self.table.get_item(
            Key={'user_id': 'user-123', 'created_at': '2024-06-24T10:00:00Z'}
        )['Item']
        assert updated_item['score'] == Decimal('100.0')
    
    def test_query_gsi(self):
        # Add test data
        self.table.put_item(Item={
            'user_id': 'user-456',
            'created_at': '2024-06-24T11:00:00Z',
            'email': 'query@example.com',
            'name': 'Query User'
        })
        
        # Query GSI
        response = self.table.query(
            IndexName='email-index',
            KeyConditionExpression=Key('email').eq('query@example.com')
        )
        
        assert len(response['Items']) == 1
        assert response['Items'][0]['name'] == 'Query User'
```

**2. S3 with Complex Scenarios:**
```python
import boto3
import json
from moto import mock_s3
from botocore.exceptions import ClientError

@mock_s3
class TestS3Handler:
    def setup_method(self):
        self.s3 = boto3.client('s3', region_name='us-east-1')
        self.bucket_name = 'test-bucket'
        
        # Create bucket
        self.s3.create_bucket(Bucket=self.bucket_name)
        
        # Enable versioning
        self.s3.put_bucket_versioning(
            Bucket=self.bucket_name,
            VersioningConfiguration={'Status': 'Enabled'}
        )
    
    def test_file_upload_and_metadata(self):
        # Upload with metadata
        self.s3.put_object(
            Bucket=self.bucket_name,
            Key='data/test-file.json',
            Body=json.dumps({'test': 'data'}),
            ContentType='application/json',
            Metadata={
                'uploaded-by': 'test-user',
                'environment': 'test'
            }
        )
        
        # Verify metadata
        response = self.s3.head_object(
            Bucket=self.bucket_name,
            Key='data/test-file.json'
        )
        
        assert response['Metadata']['uploaded-by'] == 'test-user'
        assert response['ContentType'] == 'application/json'
    
    def test_list_objects_with_prefix(self):
        # Upload multiple files
        files = [
            'logs/2024/06/24/app.log',
            'logs/2024/06/24/error.log',
            'data/users.json',
            'data/orders.json'
        ]
        
        for file_key in files:
            self.s3.put_object(
                Bucket=self.bucket_name,
                Key=file_key,
                Body='test content'
            )
        
        # List logs only
        response = self.s3.list_objects_v2(
            Bucket=self.bucket_name,
            Prefix='logs/'
        )
        
        log_files = [obj['Key'] for obj in response['Contents']]
        assert len(log_files) == 2
        assert all('logs/' in key for key in log_files)
    
    def test_error_handling(self):
        # Test non-existent object
        with pytest.raises(ClientError) as exc_info:
            self.s3.get_object(
                Bucket=self.bucket_name,
                Key='non-existent-file.txt'
            )
        
        assert exc_info.value.response['Error']['Code'] == 'NoSuchKey'
```

**3. Lambda Function Testing with Moto:**
```python
import boto3
import json
import zipfile
import io
from moto import mock_lambda, mock_iam

@mock_lambda
@mock_iam
def test_lambda_function_creation():
    # Create IAM role for Lambda
    iam = boto3.client('iam', region_name='us-east-1')
    
    assume_role_policy = {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {"Service": "lambda.amazonaws.com"},
                "Action": "sts:AssumeRole"
            }
        ]
    }
    
    iam.create_role(
        RoleName='test-lambda-role',
        AssumeRolePolicyDocument=json.dumps(assume_role_policy)
    )
    
    # Create Lambda function code
    lambda_code = '''
def lambda_handler(event, context):
    return {
        'statusCode': 200,
        'body': json.dumps({'message': 'Hello from Lambda!'})
    }
'''
    
    # Create ZIP file
    zip_buffer = io.BytesIO()
    with zipfile.ZipFile(zip_buffer, 'w') as zip_file:
        zip_file.writestr('lambda_function.py', lambda_code)
    zip_buffer.seek(0)
    
    # Create Lambda function
    lambda_client = boto3.client('lambda', region_name='us-east-1')
    
    response = lambda_client.create_function(
        FunctionName='test-function',
        Runtime='python3.9',
        Role='arn:aws:iam::123456789012:role/test-lambda-role',
        Handler='lambda_function.lambda_handler',
        Code={'ZipFile': zip_buffer.read()},
        Description='Test Lambda function'
    )
    
    assert response['FunctionName'] == 'test-function'
    assert response['Runtime'] == 'python3.9'
    
    # Test function invocation
    invoke_response = lambda_client.invoke(
        FunctionName='test-function',
        Payload=json.dumps({'test': 'data'})
    )
    
    result = json.loads(invoke_response['Payload'].read())
    assert result['statusCode'] == 200
```

## **LocalStack: Full AWS Cloud Stack**

**What is LocalStack?**
LocalStack provides a fully functional local AWS cloud stack. Unlike Moto (which mocks individual services), LocalStack runs actual AWS-compatible services locally.

**LocalStack vs Moto Comparison:**
```
Moto:                          LocalStack:
┌─────────────────────┐       ┌─────────────────────┐
│ • Python library    │       │ • Standalone service│
│ • Function mocking  │       │ • Real API endpoints│
│ • Fast execution    │       │ • Cross-language    │
│ • Limited features  │       │ • Full AWS features │
│ • Unit testing     │       │ • Integration tests │
└─────────────────────┘       └─────────────────────┘
```

**LocalStack Installation:**
```bash
# Install LocalStack
pip install localstack

# Install with specific services
pip install localstack[full]

# Using Docker (recommended for CI/CD)
docker pull localstack/localstack

# Using Docker Compose
version: '3.8'
services:
  localstack:
    image: localstack/localstack
    ports:
      - "4566:4566"
    environment:
      - SERVICES=dynamodb,s3,lambda,sqs,sns
      - DEBUG=1
      - DATA_DIR=/tmp/localstack/data
    volumes:
      - "./tmp/localstack:/tmp/localstack"
      - "/var/run/docker.sock:/var/run/docker.sock"
```

**LocalStack Configuration:**
```bash
# Environment variables for LocalStack
export SERVICES=dynamodb,s3,lambda,sqs,sns,apigateway
export DEBUG=1
export DATA_DIR=/tmp/localstack/data
export PORT_WEB_UI=8080
export LAMBDA_EXECUTOR=docker  # or local
export LOCALSTACK_HOSTNAME=localhost
```

**Using LocalStack in Tests:**
```python
import boto3
import pytest
import subprocess
import time
import requests
import os

class LocalStackManager:
    """Manager class for LocalStack lifecycle"""
    
    @staticmethod
    def start_localstack():
        """Start LocalStack service"""
        # Set environment variables
        env = os.environ.copy()
        env.update({
            'SERVICES': 'dynamodb,s3,lambda,sqs,sns',
            'DEBUG': '1',
            'DATA_DIR': '/tmp/localstack/data'
        })
        
        # Start LocalStack
        process = subprocess.Popen(
            ['localstack', 'start'],
            env=env,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE
        )
        
        # Wait for LocalStack to be ready
        max_retries = 30
        for _ in range(max_retries):
            try:
                response = requests.get('http://localhost:4566/health')
                if response.status_code == 200:
                    health_data = response.json()
                    if health_data.get('services', {}).get('dynamodb') == 'running':
                        break
            except:
                pass
            time.sleep(2)
        else:
            raise Exception("LocalStack failed to start")
        
        return process
    
    @staticmethod
    def stop_localstack():
        """Stop LocalStack service"""
        subprocess.run(['localstack', 'stop'], capture_output=True)

@pytest.fixture(scope='session')
def localstack():
    """Session-scoped LocalStack fixture"""
    process = LocalStackManager.start_localstack()
    yield
    LocalStackManager.stop_localstack()

@pytest.fixture
def aws_credentials():
    """Configure boto3 for LocalStack"""
    os.environ['AWS_ACCESS_KEY_ID'] = 'test'
    os.environ['AWS_SECRET_ACCESS_KEY'] = 'test'
    os.environ['AWS_DEFAULT_REGION'] = 'us-east-1'
    os.environ['AWS_ENDPOINT_URL'] = 'http://localhost:4566'

def test_dynamodb_with_localstack(localstack, aws_credentials):
    """Test DynamoDB operations with LocalStack"""
    
    # Create DynamoDB client pointing to LocalStack
    dynamodb = boto3.resource(
        'dynamodb',
        endpoint_url='http://localhost:4566',
        region_name='us-east-1'
    )
    
    # Create table
    table = dynamodb.create_table(
        TableName='test-table',
        KeySchema=[{'AttributeName': 'id', 'KeyType': 'HASH'}],
        AttributeDefinitions=[{'AttributeName': 'id', 'AttributeType': 'S'}],
        BillingMode='PAY_PER_REQUEST'
    )
    
    # Wait for table to be active
    table.wait_until_exists()
    
    # Test operations
    table.put_item(Item={
        'id': 'test-123',
        'data': 'test-data',
        'timestamp': '2024-06-24T10:00:00Z'
    })
    
    response = table.get_item(Key={'id': 'test-123'})
    assert response['Item']['data'] == 'test-data'
    
    # Test query
    response = table.scan()
    assert len(response['Items']) == 1

def test_s3_with_localstack(localstack, aws_credentials):
    """Test S3 operations with LocalStack"""
    
    s3 = boto3.client(
        's3',
        endpoint_url='http://localhost:4566',
        region_name='us-east-1'
    )
    
    # Create bucket
    bucket_name = 'test-bucket'
    s3.create_bucket(Bucket=bucket_name)
    
    # Upload file
    s3.put_object(
        Bucket=bucket_name,
        Key='test-file.txt',
        Body=b'Hello LocalStack!'
    )
    
    # Download file
    response = s3.get_object(Bucket=bucket_name, Key='test-file.txt')
    content = response['Body'].read()
    
    assert content == b'Hello LocalStack!'
    
    # Test presigned URL
    url = s3.generate_presigned_url(
        'get_object',
        Params={'Bucket': bucket_name, 'Key': 'test-file.txt'},
        ExpiresIn=3600
    )
    
    assert 'localhost:4566' in url
```

**Advanced LocalStack Usage:**

**1. Lambda Function Testing:**
```python
import boto3
import json
import zipfile
import io

def test_lambda_with_localstack(localstack, aws_credentials):
    """Test Lambda function deployment and invocation"""
    
    # Create Lambda client
    lambda_client = boto3.client(
        'lambda',
        endpoint_url='http://localhost:4566',
        region_name='us-east-1'
    )
    
    # Create IAM role (LocalStack auto-creates if not exists)
    iam = boto3.client(
        'iam',
        endpoint_url='http://localhost:4566',
        region_name='us-east-1'
    )
    
    # Lambda function code
    lambda_code = '''
import json

def lambda_handler(event, context):
    name = event.get('name', 'World')
    return {
        'statusCode': 200,
        'body': json.dumps({
            'message': f'Hello {name}!',
            'event': event
        })
    }
'''
    
    # Create ZIP file
    zip_buffer = io.BytesIO()
    with zipfile.ZipFile(zip_buffer, 'w') as zip_file:
        zip_file.writestr('lambda_function.py', lambda_code)
    zip_buffer.seek(0)
    
    # Deploy Lambda function
    response = lambda_client.create_function(
        FunctionName='test-function',
        Runtime='python3.9',
        Role='arn:aws:iam::000000000000:role/lambda-role',  # LocalStack default
        Handler='lambda_function.lambda_handler',
        Code={'ZipFile': zip_buffer.read()}
    )
    
    # Test function invocation
    invoke_response = lambda_client.invoke(
        FunctionName='test-function',
        Payload=json.dumps({'name': 'LocalStack'})
    )
    
    result = json.loads(invoke_response['Payload'].read())
    response_body = json.loads(result['body'])
    
    assert response_body['message'] == 'Hello LocalStack!'
```

**2. SQS and SNS Integration:**
```python
import boto3
import json

def test_sqs_sns_integration(localstack, aws_credentials):
    """Test SQS and SNS integration"""
    
    # Create SQS client
    sqs = boto3.client(
        'sqs',
        endpoint_url='http://localhost:4566',
        region_name='us-east-1'
    )
    
    # Create SNS client
    sns = boto3.client(
        'sns',
        endpoint_url='http://localhost:4566',
        region_name='us-east-1'
    )
    
    # Create SQS queue
    queue_response = sqs.create_queue(QueueName='test-queue')
    queue_url = queue_response['QueueUrl']
    
    # Get queue attributes to get ARN
    queue_attrs = sqs.get_queue_attributes(
        QueueUrl=queue_url,
        AttributeNames=['QueueArn']
    )
    queue_arn = queue_attrs['Attributes']['QueueArn']
    
    # Create SNS topic
    topic_response = sns.create_topic(Name='test-topic')
    topic_arn = topic_response['TopicArn']
    
    # Subscribe queue to topic
    sns.subscribe(
        TopicArn=topic_arn,
        Protocol='sqs',
        Endpoint=queue_arn
    )
    
    # Publish message to topic
    message = {'test': 'message', 'timestamp': '2024-06-24T10:00:00Z'}
    sns.publish(
        TopicArn=topic_arn,
        Message=json.dumps(message),
        Subject='Test Message'
    )
    
    # Receive message from queue
    messages = sqs.receive_message(QueueUrl=queue_url)
    
    assert 'Messages' in messages
    received_message = json.loads(messages['Messages'][0]['Body'])
    assert 'test' in json.loads(received_message['Message'])
```

## **Testing Framework Integration**

**Pytest Configuration for AWS Testing:**

**conftest.py:**
```python
import pytest
import boto3
import os
from moto import mock_dynamodb, mock_s3, mock_lambda
from localstack_utils import LocalStackManager

# Global fixtures
@pytest.fixture(scope='session')
def aws_credentials():
    """Mock AWS Credentials for moto"""
    os.environ['AWS_ACCESS_KEY_ID'] = 'testing'
    os.environ['AWS_SECRET_ACCESS_KEY'] = 'testing'
    os.environ['AWS_SECURITY_TOKEN'] = 'testing'
    os.environ['AWS_SESSION_TOKEN'] = 'testing'
    os.environ['AWS_DEFAULT_REGION'] = 'us-east-1'

@pytest.fixture
def dynamodb_mock(aws_credentials):
    """Mock DynamoDB for unit tests"""
    with mock_dynamodb():
        yield boto3.resource('dynamodb', region_name='us-east-1')

@pytest.fixture
def s3_mock(aws_credentials):
    """Mock S3 for unit tests"""
    with mock_s3():
        yield boto3.client('s3', region_name='us-east-1')

@pytest.fixture(scope='session')
def localstack_session():
    """Session-scoped LocalStack for integration tests"""
    process = LocalStackManager.start_localstack()
    
    # Configure boto3 for LocalStack
    os.environ['AWS_ENDPOINT_URL'] = 'http://localhost:4566'
    
    yield
    
    LocalStackManager.stop_localstack()

@pytest.fixture
def localstack_aws():
    """LocalStack AWS clients"""
    return {
        'dynamodb': boto3.resource(
            'dynamodb',
            endpoint_url='http://localhost:4566',
            region_name='us-east-1'
        ),
        's3': boto3.client(
            's3',
            endpoint_url='http://localhost:4566',
            region_name='us-east-1'
        ),
        'lambda': boto3.client(
            'lambda',
            endpoint_url='http://localhost:4566',
            region_name='us-east-1'
        )
    }
```

**Test Organization:**
```
tests/
├── unit/                    # Fast tests with moto
│   ├── test_database.py
│   ├── test_s3_handler.py
│   └── test_lambda_function.py
├── integration/             # Slower tests with LocalStack
│   ├── test_full_workflow.py
│   ├── test_service_integration.py
│   └── test_lambda_deployment.py
├── fixtures/
│   ├── sample_events.py
│   └── test_data.py
└── conftest.py
```

**Example Integration Test:**
```python
# tests/integration/test_full_workflow.py
import pytest
import json
import boto3
from your_lambda_function import lambda_handler

class TestFullWorkflow:
    """Integration tests using LocalStack"""
    
    def setup_method(self, localstack_aws):
        """Setup test environment"""
        self.aws = localstack_aws
        
        # Create DynamoDB table
        self.table = self.aws['dynamodb'].create_table(
            TableName='users',
            KeySchema=[{'AttributeName': 'user_id', 'KeyType': 'HASH'}],
            AttributeDefinitions=[{'AttributeName': 'user_id', 'AttributeType': 'S'}],
            BillingMode='PAY_PER_REQUEST'
        )
        
        # Create S3 bucket
        self.aws['s3'].create_bucket(Bucket='user-data')
        
        # Wait for resources to be ready
        self.table.wait_until_exists()
    
    def test_user_registration_workflow(self, localstack_session, localstack_aws):
        """Test complete user registration workflow"""
        
        # Test event
        event = {
            'httpMethod': 'POST',
            'body': json.dumps({
                'user_id': 'user-123',
                'email': 'test@example.com',
                'name': 'Test User'
            }),
            'headers': {'Content-Type': 'application/json'}
        }
        
        # Execute Lambda function
        response = lambda_handler(event, {})
        
        # Verify response
        assert response['statusCode'] == 201
        response_body = json.loads(response['body'])
        assert response_body['user_id'] == 'user-123'
        
        # Verify data was stored in DynamoDB
        item = self.table.get_item(Key={'user_id': 'user-123'})
        assert 'Item' in item
        assert item['Item']['email'] == 'test@example.com'
        
        # Verify file was created in S3
        try:
            obj = self.aws['s3'].get_object(
                Bucket='user-data',
                Key=f"users/{response_body['user_id']}.json"
            )
            user_data = json.loads(obj['Body'].read())
            assert user_data['email'] == 'test@example.com'
        except Exception as e:
            pytest.fail(f"S3 object not found: {e}")
```

## **Performance and Best Practices**

**Test Performance Optimization:**

**1. Test Isolation:**
```python
import pytest
from moto import mock_dynamodb

class TestDatabaseOperations:
    """Isolated test class"""
    
    @pytest.fixture(autouse=True)
    def setup_test_environment(self, dynamodb_mock):
        """Auto-setup for each test"""
        self.dynamodb = dynamodb_mock
        
        # Create fresh table for each test
        self.table = self.dynamodb.create_table(
            TableName=f'test-table-{id(self)}',  # Unique table name
            KeySchema=[{'AttributeName': 'id', 'KeyType': 'HASH'}],
            AttributeDefinitions=[{'AttributeName': 'id', 'AttributeType': 'S'}],
            BillingMode='PAY_PER_REQUEST'
        )
    
    def test_operation_1(self):
        # Test uses fresh table
        pass
    
    def test_operation_2(self):
        # Test uses different fresh table
        pass
```

**2. Fixture Scope Optimization:**
```python
# Expensive fixtures - session scope
@pytest.fixture(scope='session')
def localstack_environment():
    """Start LocalStack once per test session"""
    pass

# Medium cost fixtures - class scope  
@pytest.fixture(scope='class')
def aws_resources():
    """Create AWS resources once per test class"""
    pass

# Cheap fixtures - function scope
@pytest.fixture
def test_data():
    """Generate test data for each test"""
    pass
```

**3. Parallel Test Execution:**
```bash
# Install pytest-xdist for parallel execution
pip install pytest-xdist

# Run tests in parallel
pytest -n auto tests/unit/          # Auto-detect CPU cores
pytest -n 4 tests/integration/      # Use 4 processes
```

**CI/CD Integration Best Practices:**

**Jenkins Pipeline Optimization:**
```groovy
stage('Unit Tests') {
    parallel {
        stage('Moto Tests') {
            steps {
                sh '''
                    export PYTHONPATH="${WORKSPACE}/functions/${FUNCTION_NAME}:${PYTHONPATH}"
                    cd tests/unit
                    python3 -m pytest -v --junitxml=moto-results.xml
                '''
            }
        }
        stage('Integration Tests') {
            steps {
                sh '''
                    # Start LocalStack
                    docker run -d --name localstack-test -p 4566:4566 \
                        -e SERVICES=dynamodb,s3,lambda \
                        localstack/localstack
                    
                    # Wait for LocalStack
                    sleep 30
                    
                    # Run integration tests
                    export AWS_ENDPOINT_URL=http://localhost:4566
                    cd tests/integration
                    python3 -m pytest -v --junitxml=integration-results.xml
                    
                    # Cleanup
                    docker stop localstack-test
                    docker rm localstack-test
                '''
            }
        }
    }
}
```

**Memory and Resource Management:**
```python
# Cleanup resources after tests
@pytest.fixture
def aws_cleanup():
    """Cleanup AWS resources after test"""
    created_resources = []
    
    yield created_resources
    
    # Cleanup logic
    for resource in created_resources:
        try:
            resource.delete()
        except:
            pass  # Ignore cleanup errors in tests
```

This comprehensive guide covers all the testing dependencies you'll need for your serverless CI/CD pipeline, from unit testing with Moto to integration testing with LocalStack, all integrated into your Jenkins pipeline for robust automated testing.
