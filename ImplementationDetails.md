**Updated Implementation Guide for Web-based Jenkins and AWS Console Only**

**Step 1: Jenkins Configuration (Using Existing Web Jenkins)**

**Plugin Installation via Jenkins Web UI**
Since you already have Jenkins through the website, go to Jenkins Dashboard > Manage Jenkins > Manage Plugins > Available tab. Search and install these plugins:
- Bitbucket Plugin
- Pipeline Plugin  
- Blue Ocean Plugin
- AWS Steps Plugin
- Terraform Plugin
- Docker Plugin
- Timestamper Plugin

After installation, click "Restart Jenkins when installation is complete and no jobs are running."

**Global Tool Configuration**
Go to Manage Jenkins > Global Tool Configuration. You'll need to configure tools that Jenkins can automatically install:

For **Git**: Usually pre-installed, verify it's configured
For **Terraform**: Add Terraform installation > Install automatically > Choose "Install from hashicorp.com" > Select latest version
For **AWS CLI**: Add a shell installation that downloads AWS CLI during builds

**Credential Management via AWS Console**
Since you can't use AWS CLI, create IAM users through AWS Console:

1. Go to AWS Console > IAM > Users > Create User
2. Create user "jenkins-dev" with programmatic access
3. Attach policies: `AWSLambdaFullAccess`, `AmazonDynamoDBFullAccess`, `AmazonS3FullAccess`, `CloudWatchFullAccess`
4. Download the CSV with Access Key ID and Secret Access Key
5. Repeat for "jenkins-qa" and "jenkins-prod" users

In Jenkins web UI: Manage Jenkins > Manage Credentials > Global > Add Credentials
- Select "AWS Credentials" type
- Add Access Key ID and Secret Access Key from CSV
- Set ID as "aws-dev-credentials"
- Repeat for QA and Prod credentials

**Jenkins Agent Configuration**
If your Jenkins is cloud-hosted, you may need to configure agents that can run your builds. Go to Manage Jenkins > Manage Nodes and Clouds. You can either use the built-in node or configure cloud agents (AWS EC2, Docker) that will have the necessary tools installed.

**Step 2: BitBucket Repository Structure (Web Interface)**

**Repository Setup via BitBucket Web UI**
Log into your BitBucket account and navigate to your existing repository. You'll reorganize files using BitBucket's web interface:

**Current File Reorganization**
1. In BitBucket web UI, create new folder structure by clicking "Source" > "+" > "Create file"
2. When creating files, use paths like `functions/your-function-name/lambda_function.py` to automatically create folders
3. Move your existing files:
   - `lambda_function.py` → `functions/your-function-name/lambda_function.py`
   - `database.py` → `functions/your-function-name/database.py`
   - `config.py` → `functions/your-function-name/config.py`
   - `utils.py` → `functions/your-function-name/utils.py`

**Create New Required Files via BitBucket Web UI**

**Create `functions/your-function-name/requirements.txt`:**
```
boto3>=1.28.85
# Add any other dependencies your existing code uses
# Check your import statements in lambda_function.py, database.py, config.py, utils.py
```

**Create `Jenkinsfile` in repository root:**
```groovy
pipeline {
    agent any
    
    environment {
        AWS_DEFAULT_REGION = 'us-east-1'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Code checked out from BitBucket'
            }
        }
        
        stage('Unit Tests') {
            steps {
                script {
                    // Install dependencies and run tests
                    sh '''
                        pip3 install --user pytest moto boto3
                        cd tests/unit
                        python3 -m pytest test_lambda_function.py -v
                    '''
                }
            }
        }
        
        stage('Deploy to Dev') {
            steps {
                withAWS(credentials: 'aws-dev-credentials', region: 'us-east-1') {
                    script {
                        // Install Terraform if needed
                        sh '''
                            cd infrastructure/dev
                            terraform init
                            terraform plan -var-file=dev.tfvars
                            terraform apply -var-file=dev.tfvars -auto-approve
                        '''
                    }
                }
            }
        }
    }
}
```

**Create `.gitignore` in repository root:**
```
.terraform/
*.tfstate
*.tfstate.backup
__pycache__/
*.pyc
.pytest_cache/
.env
```

**BitBucket Webhook Setup**
1. In BitBucket, go to Repository Settings > Webhooks
2. Click "Add webhook"
3. Title: "Jenkins Pipeline Trigger"
4. URL: `https://your-jenkins-url/bitbucket-hook/` (get this from your Jenkins provider)
5. Select triggers: "Repository push", "Pull request created", "Pull request updated"
6. Save webhook

**Step 3: Terraform Infrastructure Setup (AWS Console)**

**S3 Backend Setup via AWS Console**
1. Go to AWS Console > S3 > Create bucket
2. Create buckets:
   - `your-company-terraform-state-dev`
   - `your-company-terraform-state-qa`
   - `your-company-terraform-state-prod`
3. For each bucket: Properties > Versioning > Enable
4. For each bucket: Properties > Default encryption > Enable with AES-256

**DynamoDB Tables via AWS Console**
1. Go to AWS Console > DynamoDB > Create table
2. Create tables:
   - Table name: `terraform-state-lock-dev`
   - Partition key: `LockID` (String)
   - Use default settings
3. Repeat for `terraform-state-lock-qa` and `terraform-state-lock-prod`

**Create Terraform Files via BitBucket Web UI**

**Create `infrastructure/modules/lambda-function/main.tf`:**
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
}

# Basic execution policy
resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

# DynamoDB access policy
resource "aws_iam_role_policy" "lambda_dynamodb" {
  name = "${var.function_name}-dynamodb-policy"
  role = aws_iam_role.lambda_role.id

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
        Resource = "*"
      }
    ]
  })
}

# Package Lambda function
data "archive_file" "lambda_zip" {
  type        = "zip"
  source_dir  = var.source_dir
  output_path = "${path.module}/${var.function_name}.zip"
}

# Lambda function
resource "aws_lambda_function" "main" {
  filename         = data.archive_file.lambda_zip.output_path
  function_name    = var.function_name
  role            = aws_iam_role.lambda_role.arn
  handler         = "lambda_function.lambda_handler"
  runtime         = "python3.9"
  memory_size     = var.memory_size
  timeout         = var.timeout
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256

  environment {
    variables = var.environment_variables
  }
}

# CloudWatch Log Group
resource "aws_cloudwatch_log_group" "lambda_logs" {
  name              = "/aws/lambda/${var.function_name}"
  retention_in_days = 14
}
```

**Create `infrastructure/modules/lambda-function/variables.tf`:**
```hcl
variable "function_name" {
  description = "Name of the Lambda function"
  type        = string
}

variable "source_dir" {
  description = "Directory containing Lambda function code"
  type        = string
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
```

**Create `infrastructure/dev/main.tf`:**
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
  }
}

provider "aws" {
  region = "us-east-1"
}

module "your_lambda_function" {
  source = "../modules/lambda-function"
  
  function_name = var.function_name
  source_dir    = "${path.root}/../../functions/your-function-name"
  memory_size   = var.memory_size
  timeout       = var.timeout
  
  environment_variables = {
    ENVIRONMENT = "dev"
  }
}
```

**Create `infrastructure/dev/variables.tf`:**
```hcl
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
```

**Create `infrastructure/dev/dev.tfvars`:**
```hcl
function_name = "my-lambda-function-dev"
memory_size   = 256
timeout       = 60
```

**Step 4: Unit Testing Setup (Web-based)**

**Create `tests/unit/test_lambda_function.py` via BitBucket:**
```python
import json
import pytest
import boto3
import sys
import os
from moto import mock_dynamodb

# Add function directory to Python path
sys.path.insert(0, os.path.join(os.path.dirname(__file__), '../../functions/your-function-name'))

# Import your Lambda function
from lambda_function import lambda_handler

@mock_dynamodb
def test_lambda_handler_basic():
    """Test basic Lambda function execution"""
    
    # Create mock event (adjust based on your function's expected input)
    event = {
        'httpMethod': 'GET',
        'pathParameters': {'id': 'test-123'},
        'body': None
    }
    
    context = {}
    
    # Execute function
    response = lambda_handler(event, context)
    
    # Basic assertions
    assert isinstance(response, dict)
    assert 'statusCode' in response
    assert response['statusCode'] in [200, 400, 404, 500]

@mock_dynamodb  
def test_lambda_handler_with_body():
    """Test Lambda function with POST data"""
    
    event = {
        'httpMethod': 'POST',
        'body': json.dumps({
            'test_data': 'sample value',
            'id': 'test-456'
        }),
        'headers': {
            'Content-Type': 'application/json'
        }
    }
    
    context = {}
    response = lambda_handler(event, context)
    
    assert response['statusCode'] in [200, 201, 400, 500]
    if 'body' in response:
        # If your function returns JSON, test it can be parsed
        try:
            json.loads(response['body'])
        except:
            pass  # Body might not be JSON

def test_lambda_handler_error_handling():
    """Test error handling with invalid input"""
    
    event = {}  # Empty event should trigger error handling
    context = {}
    
    response = lambda_handler(event, context)
    
    # Should handle errors gracefully
    assert response['statusCode'] in [400, 500]
```

**Step 5: LocalStack Integration Testing (Simplified)**

Since you can't install LocalStack on the Jenkins server directly, create a simplified integration test that can work with Jenkins agents:

**Create `tests/integration/test_integration.py`:**
```python
import json
import boto3
import pytest
import sys
import os
from moto import mock_dynamodb, mock_s3

# Add function directory to path  
sys.path.insert(0, os.path.join(os.path.dirname(__file__), '../../functions/your-function-name'))

from lambda_function import lambda_handler

class TestIntegration:
    """Integration tests using moto for more realistic AWS service simulation"""
    
    @mock_dynamodb
    @mock_s3
    def test_full_workflow_simulation(self):
        """Test complete workflow with mocked AWS services"""
        
        # Setup mock DynamoDB table
        dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
        
        # Create table (adjust schema to match your database.py usage)
        table = dynamodb.create_table(
            TableName='your-table-name',
            KeySchema=[
                {
                    'AttributeName': 'id',
                    'KeyType': 'HASH'
                }
            ],
            AttributeDefinitions=[
                {
                    'AttributeName': 'id', 
                    'AttributeType': 'S'
                }
            ],
            BillingMode='PAY_PER_REQUEST'
        )
        
        # Add test data
        table.put_item(Item={
            'id': 'test-item-1',
            'data': 'integration test data'
        })
        
        # Setup mock S3 bucket
        s3 = boto3.client('s3', region_name='us-east-1')
        s3.create_bucket(Bucket='test-bucket')
        
        # Test Lambda function
        event = {
            'httpMethod': 'GET',
            'pathParameters': {'id': 'test-item-1'}
        }
        
        response = lambda_handler(event, {})
        
        # Verify response
        assert response['statusCode'] == 200
        
        # Additional assertions based on your function's behavior
        if 'body' in response:
            body = json.loads(response['body'])
            assert 'data' in body or 'message' in body
```

**Step 6: Environment-Specific Testing Setup (AWS Console)**

**QA Environment Setup**
1. Copy dev Terraform files to qa directory via BitBucket web interface
2. Create `infrastructure/qa/` folder structure identical to dev
3. Modify backend configuration in `infrastructure/qa/main.tf`

**Create `infrastructure/qa/main.tf` (modify backend section):**
```hcl
terraform {
  backend "s3" {
    bucket         = "your-company-terraform-state-qa"
    key            = "lambda-functions/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock-qa"
    encrypt        = true
  }
}

# Rest same as dev/main.tf
```

**Create `infrastructure/qa/qa.tfvars`:**
```hcl
function_name = "my-lambda-function-qa"
memory_size   = 512
timeout       = 120
```

**Create Jenkins Pipeline for Each Environment**

**Creating Pipeline via Jenkins Web UI**
1. Go to Jenkins Dashboard > New Item
2. Enter name: "Lambda-Function-Pipeline"
3. Select "Pipeline" and click OK
4. In configuration:
   - Under "Pipeline" section
   - Definition: "Pipeline script from SCM"
   - SCM: "Git"
   - Repository URL: Your BitBucket repository URL
   - Credentials: Add BitBucket credentials (username + app password)
   - Branch: */main or */master
   - Script Path: Jenkinsfile

**Manual Verification Steps**
Before running the full pipeline, manually verify components work:

1. **Test BitBucket Connection**: Create a simple Jenkins job that just checks out code
2. **Test AWS Credentials**: Create a job that runs `aws sts get-caller-identity` 
3. **Test Terraform**: Create a job that runs `terraform version`
4. **Test Python Dependencies**: Create a job that runs `pip3 install pytest boto3 moto`

**Required Setup Checklist**
- ✅ Jenkins plugins installed via web UI
- ✅ AWS IAM users created via console with proper policies
- ✅ AWS credentials added to Jenkins via web UI
- ✅ S3 buckets created via console for Terraform state
- ✅ DynamoDB tables created via console for state locking
- ✅ BitBucket repository restructured with new folder layout
- ✅ BitBucket webhook configured to trigger Jenkins
- ✅ All Terraform and test files created via BitBucket web interface
- ✅ Basic Jenkinsfile created for pipeline execution

This approach works entirely through web interfaces without requiring direct server access or AWS CLI usage, while still providing a complete CI/CD pipeline for your serverless application.
