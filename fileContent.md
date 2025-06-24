**Content for Each NEW File**

**1. `functions/your-function-name/requirements.txt`**
```
boto3>=1.28.85
botocore>=1.31.85
# Add any other libraries your existing code imports
# Check your lambda_function.py, database.py, config.py, utils.py for import statements
# Common ones might be:
# requests>=2.31.0
# pandas>=2.0.0
# numpy>=1.24.0
# psycopg2-binary>=2.9.0  # if using PostgreSQL
# pymongo>=4.3.0  # if using MongoDB
```

**2. `Jenkinsfile` (in root directory)**
```groovy
pipeline {
    agent any
    
    environment {
        AWS_DEFAULT_REGION = 'us-east-1'
        FUNCTION_NAME = 'your-function-name'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Code checked out from BitBucket'
                // Change classification logic
                script {
                    def commitMessage = sh(script: 'git log -1 --pretty=%B', returnStdout: true).trim()
                    if (commitMessage.contains('hotfix:') || commitMessage.contains('config:') || commitMessage.contains('env:')) {
                        env.CHANGE_TYPE = 'operational'
                    } else {
                        env.CHANGE_TYPE = 'normal'
                    }
                    echo "Change classified as: ${env.CHANGE_TYPE}"
                }
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh '''
                    pip3 install --user pytest pytest-cov moto boto3 localstack
                    cd functions/${FUNCTION_NAME}
                    pip3 install --user -r requirements.txt
                '''
            }
        }
        
        stage('Unit Tests') {
            steps {
                sh '''
                    export PYTHONPATH="${WORKSPACE}/functions/${FUNCTION_NAME}:${PYTHONPATH}"
                    cd tests/unit
                    python3 -m pytest test_lambda_function.py -v --junitxml=results.xml
                '''
                junit 'tests/unit/results.xml'
            }
        }
        
        stage('Integration Tests') {
            steps {
                sh '''
                    export PYTHONPATH="${WORKSPACE}/functions/${FUNCTION_NAME}:${PYTHONPATH}"
                    cd tests/integration
                    python3 -m pytest test_integration.py -v
                '''
            }
        }
        
        stage('Approval - Operational') {
            when {
                environment name: 'CHANGE_TYPE', value: 'operational'
            }
            steps {
                script {
                    def approver = input(
                        message: 'Approve operational change?',
                        submitterParameter: 'APPROVER',
                        parameters: [
                            choice(choices: ['Approve', 'Reject'], description: 'Approval decision', name: 'DECISION')
                        ]
                    )
                    if (approver.DECISION != 'Approve') {
                        error("Deployment rejected by ${approver.APPROVER}")
                    }
                }
            }
        }
        
        stage('Approval - Normal') {
            when {
                environment name: 'CHANGE_TYPE', value: 'normal'
            }
            steps {
                script {
                    def approver = input(
                        message: 'Change Control Board: Approve normal change?',
                        submitterParameter: 'APPROVER',
                        parameters: [
                            choice(choices: ['Approve', 'Reject'], description: 'Board decision', name: 'DECISION'),
                            text(defaultValue: '', description: 'Approval comments', name: 'COMMENTS')
                        ]
                    )
                    if (approver.DECISION != 'Approve') {
                        error("Deployment rejected by change control board")
                    }
                }
            }
        }
        
        stage('Deploy to Dev') {
            steps {
                withAWS(credentials: 'aws-dev-credentials', region: 'us-east-1') {
                    sh '''
                        cd infrastructure/dev
                        terraform init
                        terraform plan -var-file=dev.tfvars -out=tfplan
                        terraform apply tfplan
                    '''
                }
            }
        }
        
        stage('QA Tests') {
            steps {
                withAWS(credentials: 'aws-qa-credentials', region: 'us-east-1') {
                    sh '''
                        cd infrastructure/qa
                        terraform init
                        terraform plan -var-file=qa.tfvars -out=tfplan
                        terraform apply tfplan
                        
                        # Run QA tests
                        cd ../../tests/qa
                        python3 -m pytest test_qa_environment.py -v
                    '''
                }
            }
        }
        
        stage('Deploy to Production') {
            steps {
                withAWS(credentials: 'aws-prod-credentials', region: 'us-east-1') {
                    sh '''
                        cd infrastructure/prod
                        terraform init
                        terraform plan -var-file=prod.tfvars -out=tfplan
                        terraform apply tfplan
                    '''
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
```

**3. `.gitignore` (in root directory)**
```
# Terraform
.terraform/
*.tfstate
*.tfstate.*
.terraform.lock.hcl
*.tfplan

# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
build/
develop-eggs/
dist/
downloads/
eggs/
.eggs/
lib/
lib64/
parts/
sdist/
var/
wheels/
*.egg-info/
.installed.cfg
*.egg
MANIFEST

# Testing
.pytest_cache/
.coverage
htmlcov/
.tox/
.cache
nosetests.xml
coverage.xml
*.cover
.hypothesis/

# Environment files
.env
.venv
env/
venv/
ENV/
env.bak/
venv.bak/

# IDE
.vscode/
.idea/
*.swp
*.swo
*~

# OS
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db

# Logs
*.log
logs/

# LocalStack
.localstack/

# AWS
.aws/

# Jenkins
.jenkins/
```

**4. `infrastructure/modules/lambda-function/main.tf`**
```hcl
# IAM Role for Lambda
resource "aws_iam_role" "lambda_role" {
  name = "${var.function_name}-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
      }
    ]
  })

  tags = var.tags
}

# Basic execution policy
resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

# DynamoDB access policy
resource "aws_iam_role_policy" "lambda_dynamodb" {
  count = var.enable_dynamodb_access ? 1 : 0
  name  = "${var.function_name}-dynamodb-policy"
  role  = aws_iam_role.lambda_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "dynamodb:GetItem",
          "dynamodb:PutItem",
          "dynamodb:UpdateItem",
          "dynamodb:DeleteItem",
          "dynamodb:Query",
          "dynamodb:Scan"
        ]
        Resource = var.dynamodb_table_arns
      }
    ]
  })
}

# S3 access policy
resource "aws_iam_role_policy" "lambda_s3" {
  count = var.enable_s3_access ? 1 : 0
  name  = "${var.function_name}-s3-policy"
  role  = aws_iam_role.lambda_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject"
        ]
        Resource = var.s3_bucket_arns
      }
    ]
  })
}

# Package Lambda function
data "archive_file" "lambda_zip" {
  type        = "zip"
  source_dir  = var.source_dir
  output_path = "${path.module}/${var.function_name}.zip"
  excludes    = ["__pycache__", "*.pyc", ".pytest_cache", "tests"]
}

# Lambda function
resource "aws_lambda_function" "main" {
  filename         = data.archive_file.lambda_zip.output_path
  function_name    = var.function_name
  role            = aws_iam_role.lambda_role.arn
  handler         = var.handler
  runtime         = var.runtime
  memory_size     = var.memory_size
  timeout         = var.timeout
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256

  environment {
    variables = var.environment_variables
  }

  vpc_config {
    subnet_ids         = var.subnet_ids
    security_group_ids = var.security_group_ids
  }

  tags = var.tags

  depends_on = [
    aws_iam_role_policy_attachment.lambda_basic,
    aws_cloudwatch_log_group.lambda_logs
  ]
}

# CloudWatch Log Group
resource "aws_cloudwatch_log_group" "lambda_logs" {
  name              = "/aws/lambda/${var.function_name}"
  retention_in_days = var.log_retention_days
  tags              = var.tags
}

# Lambda Alias for traffic management
resource "aws_lambda_alias" "main" {
  name             = var.alias_name
  description      = "Alias for ${var.function_name}"
  function_name    = aws_lambda_function.main.function_name
  function_version = aws_lambda_function.main.version
}
```

**5. `infrastructure/modules/lambda-function/variables.tf`**
```hcl
variable "function_name" {
  description = "Name of the Lambda function"
  type        = string
}

variable "source_dir" {
  description = "Directory containing Lambda function code"
  type        = string
}

variable "handler" {
  description = "Lambda function handler"
  type        = string
  default     = "lambda_function.lambda_handler"
}

variable "runtime" {
  description = "Lambda runtime"
  type        = string
  default     = "python3.9"
}

variable "memory_size" {
  description = "Lambda memory size in MB"
  type        = number
  default     = 128
}

variable "timeout" {
  description = "Lambda timeout in seconds"
  type        = number
  default     = 30
}

variable "environment_variables" {
  description = "Environment variables for Lambda"
  type        = map(string)
  default     = {}
}

variable "log_retention_days" {
  description = "CloudWatch log retention in days"
  type        = number
  default     = 14
}

variable "alias_name" {
  description = "Lambda alias name"
  type        = string
  default     = "live"
}

variable "enable_dynamodb_access" {
  description = "Enable DynamoDB access"
  type        = bool
  default     = true
}

variable "enable_s3_access" {
  description = "Enable S3 access"
  type        = bool
  default     = false
}

variable "dynamodb_table_arns" {
  description = "DynamoDB table ARNs for access"
  type        = list(string)
  default     = ["*"]
}

variable "s3_bucket_arns" {
  description = "S3 bucket ARNs for access"
  type        = list(string)
  default     = ["*"]
}

variable "subnet_ids" {
  description = "VPC subnet IDs"
  type        = list(string)
  default     = []
}

variable "security_group_ids" {
  description = "VPC security group IDs"
  type        = list(string)
  default     = []
}

variable "tags" {
  description = "Tags to apply to resources"
  type        = map(string)
  default = {
    ManagedBy = "Terraform"
  }
}
```

**6. `infrastructure/modules/lambda-function/outputs.tf`**
```hcl
output "function_name" {
  description = "Name of the Lambda function"
  value       = aws_lambda_function.main.function_name
}

output "function_arn" {
  description = "ARN of the Lambda function"
  value       = aws_lambda_function.main.arn
}

output "function_version" {
  description = "Version of the Lambda function"
  value       = aws_lambda_function.main.version
}

output "alias_arn" {
  description = "ARN of the Lambda alias"
  value       = aws_lambda_alias.main.arn
}

output "role_arn" {
  description = "ARN of the Lambda execution role"
  value       = aws_iam_role.lambda_role.arn
}

output "log_group_name" {
  description = "Name of the CloudWatch log group"
  value       = aws_cloudwatch_log_group.lambda_logs.name
}

output "invoke_arn" {
  description = "Invoke ARN of the Lambda function"
  value       = aws_lambda_function.main.invoke_arn
}
```

**7. `infrastructure/dev/main.tf`**
```hcl
terraform {
  required_version = ">= 1.0"
  
  backend "s3" {
    bucket         = "your-company-terraform-state-dev"
    key            = "lambda-functions/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock-dev"
    encrypt        = true
  }
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    archive = {
      source  = "hashicorp/archive"
      version = "~> 2.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Environment = "dev"
      Project     = "lambda-pipeline"
      ManagedBy   = "Terraform"
    }
  }
}

module "lambda_function" {
  source = "../modules/lambda-function"
  
  function_name    = var.function_name
  source_dir       = "${path.root}/../../functions/your-function-name"
  memory_size      = var.memory_size
  timeout          = var.timeout
  alias_name       = "dev"
  
  environment_variables = {
    ENVIRONMENT = "dev"
    LOG_LEVEL   = "DEBUG"
  }
  
  enable_dynamodb_access = true
  enable_s3_access       = var.enable_s3_access
  
  tags = {
    Environment = "dev"
  }
}

# API Gateway (if needed)
resource "aws_api_gateway_rest_api" "main" {
  count = var.create_api_gateway ? 1 : 0
  name  = "${var.function_name}-api"
  
  endpoint_configuration {
    types = ["REGIONAL"]
  }
}
```

**8. `infrastructure/dev/variables.tf`**
```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "function_name" {
  description = "Lambda function name"
  type        = string
}

variable "memory_size" {
  description = "Lambda memory size"
  type        = number
  default     = 128
}

variable "timeout" {
  description = "Lambda timeout"
  type        = number
  default     = 30
}

variable "enable_s3_access" {
  description = "Enable S3 access for Lambda"
  type        = bool
  default     = false
}

variable "create_api_gateway" {
  description = "Create API Gateway"
  type        = bool
  default     = false
}
```

**9. `infrastructure/dev/dev.tfvars`**
```hcl
function_name       = "my-lambda-function-dev"
memory_size         = 256
timeout             = 60
enable_s3_access    = true
create_api_gateway  = false
```

**10. `tests/unit/test_lambda_function.py`**
```python
import json
import pytest
import boto3
import sys
import os
from moto import mock_dynamodb, mock_s3

# Add function directory to Python path
sys.path.insert(0, os.path.join(os.path.dirname(__file__), '../../functions/your-function-name'))

# Import your modules
from lambda_function import lambda_handler
import database
import config
import utils

class TestLambdaFunction:
    """Unit tests for Lambda function"""
    
    @mock_dynamodb
    def test_lambda_handler_get_request(self):
        """Test GET request handling"""
        
        # Setup mock DynamoDB
        dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
        table = dynamodb.create_table(
            TableName='test-table',
            KeySchema=[{'AttributeName': 'id', 'KeyType': 'HASH'}],
            AttributeDefinitions=[{'AttributeName': 'id', 'AttributeType': 'S'}],
            BillingMode='PAY_PER_REQUEST'
        )
        
        # Add test data
        table.put_item(Item={'id': 'test-123', 'data': 'test value'})
        
        # Mock event
        event = {
            'httpMethod': 'GET',
            'pathParameters': {'id': 'test-123'},
            'headers': {'Content-Type': 'application/json'}
        }
        
        context = {}
        
        # Execute function
        response = lambda_handler(event, context)
        
        # Assertions
        assert response['statusCode'] == 200
        assert 'body' in response
        
        body = json.loads(response['body'])
        assert 'data' in body or 'message' in body

    @mock_dynamodb
    def test_lambda_handler_post_request(self):
        """Test POST request handling"""
        
        # Setup mock DynamoDB
        dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
        table = dynamodb.create_table(
            TableName='test-table',
            KeySchema=[{'AttributeName': 'id', 'KeyType': 'HASH'}],
            AttributeDefinitions=[{'AttributeName': 'id', 'AttributeType': 'S'}],
            BillingMode='PAY_PER_REQUEST'
        )
        
        # Mock event
        event = {
            'httpMethod': 'POST',
            'body': json.dumps({
                'id': 'new-item',
                'data': 'new data value'
            }),
            'headers': {'Content-Type': 'application/json'}
        }
        
        context = {}
        
        # Execute function
        response = lambda_handler(event, context)
        
        # Assertions
        assert response['statusCode'] in [200, 201]
        assert 'body' in response

    def test_lambda_handler_invalid_method(self):
        """Test invalid HTTP method handling"""
        
        event = {
            'httpMethod': 'DELETE',
            'pathParameters': {'id': 'test-123'}
        }
        
        context = {}
        response = lambda_handler(event, context)
        
        # Should return method not allowed
        assert response['statusCode'] in [400, 405, 500]

    def test_lambda_handler_missing_parameters(self):
        """Test missing required parameters"""
        
        event = {
            'httpMethod': 'GET'
            # Missing pathParameters
        }
        
        context = {}
        response = lambda_handler(event, context)
        
        # Should handle gracefully
        assert response['statusCode'] in [400, 500]
        assert 'body' in response

    def test_lambda_handler_invalid_json(self):
        """Test invalid JSON in body"""
        
        event = {
            'httpMethod': 'POST',
            'body': 'invalid json string',
            'headers': {'Content-Type': 'application/json'}
        }
        
        context = {}
        response = lambda_handler(event, context)
        
        # Should handle JSON parse error
        assert response['statusCode'] in [400, 500]
```

**11. `tests/unit/test_database.py`**
```python
import pytest
import boto3
import sys
import os
from moto import mock_dynamodb

# Add function directory to Python path
sys.path.insert(0, os.path.join(os.path.dirname(__file__), '../../functions/your-function-name'))

import database

class TestDatabase:
    """Unit tests for database module"""
    
    @mock_dynamodb
    def test_database_connection(self):
        """Test database connection functionality"""
        
        # Setup mock DynamoDB
        dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
        table = dynamodb.create_table(
            TableName='test-table',
            KeySchema=[{'AttributeName': 'id', 'KeyType': 'HASH'}],
            AttributeDefinitions=[{'AttributeName': 'id', 'AttributeType': 'S'}],
            BillingMode='PAY_PER_REQUEST'
        )
        
        # Test database operations (adjust based on your database.py functions)
        # Example:
        # result = database.get_item('test-id')
        # assert result is not None
        
        # Add specific tests based on functions in your database.py
        pass

    @mock_dynamodb  
    def test_database_insert(self):
        """Test database insert operations"""
        # Add tests for insert functions in your database.py
        pass
        
    @mock_dynamodb
    def test_database_update(self):
        """Test database update operations"""
        # Add tests for update functions in your database.py
        pass
        
    @mock_dynamodb
    def test_database_delete(self):
        """Test database delete operations"""
        # Add tests for delete functions in your database.py
        pass
```

**12. `tests/unit/test_config.py`**
```python
import pytest
import sys
import os

# Add function directory to Python path
sys.path.insert(0, os.path.join(os.path.dirname(__file__), '../../functions/your-function-name'))

import config

class TestConfig:
    """Unit tests for config module"""
    
    def test_config_values(self):
        """Test configuration values"""
        # Test that config values are properly set
        # Adjust based on what's in your config.py
        
        # Example tests:
        # assert hasattr(config, 'DATABASE_URL')
        # assert hasattr(config, 'API_KEY')
        # assert config.TIMEOUT > 0
        pass
    
    def test_environment_specific_config(self):
        """Test environment-specific configuration"""
        # Test that config behaves differently in different environments
        
        import os
        os.environ['ENVIRONMENT'] = 'test'
        
        # Reload config or test environment-specific behavior
        # Based on your config.py implementation
        pass
    
    def test_config_validation(self):
        """Test configuration validation"""
        # Test that config validates required values
        # Based on your config.py validation logic
        pass
```

**13. `tests/unit/test_utils.py`**
```python
import pytest
import sys
import os

# Add function directory to Python path
sys.path.insert(0, os.path.join(os.path.dirname(__file__), '../../functions/your-function-name'))

import utils

class TestUtils:
    """Unit tests for utils module"""
    
    def test_utility_functions(self):
        """Test utility functions"""
        # Add tests based on functions in your utils.py
        
        # Example tests:
        # result = utils.format_response(200, {'message': 'success'})
        # assert result['statusCode'] == 200
        # assert 'body' in result
        pass
    
    def test_data_validation(self):
        """Test data validation utilities"""
        # Test validation functions if you have them in utils.py
        pass
    
    def test_error_handling(self):
        """Test error handling utilities"""
        # Test error handling functions if you have them in utils.py
        pass
    
    def test_data_transformation(self):
        """Test data transformation utilities"""
        # Test any data transformation functions in utils.py
        pass
```

These files provide a complete foundation for your Jenkins serverless CI/CD pipeline. Adjust the specific content based on what your existing `lambda_function.py`, `database.py`, `config.py`, and `utils.py` files actually contain.
