## Next Steps After Writing the Jenkinsfile

### **Step 1: Set Up Jenkins Infrastructure**

### **1.1 Install Required Jenkins Plugins**
**Navigate to:** Manage Jenkins → Manage Plugins → Available

**Essential Plugins to Install:**
```
Core Plugins:
✅ Pipeline
✅ Pipeline: Stage View
✅ Git
✅ Blue Ocean (recommended for better UI)

AWS Integration:
✅ AWS Pipeline
✅ AWS Credentials Plugin
✅ AWS SDK

Build Tools:
✅ Python Plugin
✅ Terraform Plugin
✅ NodeJS Plugin (if using frontend builds)

Reporting:
✅ JUnit Plugin
✅ HTML Publisher Plugin
✅ Test Results Analyzer

Notifications (optional):
✅ Slack Notification Plugin
✅ Email Extension Plugin
```

**Installation Process:**
1. Check all required plugins
2. Click "Install without restart"
3. Wait for installation to complete
4. Restart Jenkins if prompted

### **1.2 Configure Global Tools**
**Navigate to:** Manage Jenkins → Global Tool Configuration

**Python Configuration:**
```
Name: Python3
☑️ Install automatically
Installer: Extract *.zip/*.tar.gz
Download URL: https://www.python.org/ftp/python/3.11.7/Python-3.11.7.tgz
Subdirectory: Python-3.11.7
```

**Terraform Configuration:**
```
Name: Terraform
☑️ Install automatically
Version: 1.6.6 (or latest stable)
```

**Git Configuration:**
```
Name: Default
Path to Git executable: git (or auto-detect)
```

---

### **Step 2: Set Up AWS Infrastructure Prerequisites**

### **2.1 Create S3 Bucket for Terraform State**
```bash
# Create Terraform state bucket
aws s3 mb s3://euc-lambda-poc-terraform-state-dev --region us-east-1

# Enable versioning
aws s3api put-bucket-versioning \
    --bucket euc-lambda-poc-terraform-state-dev \
    --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption \
    --bucket euc-lambda-poc-terraform-state-dev \
    --server-side-encryption-configuration '{
        "Rules": [
            {
                "ApplyServerSideEncryptionByDefault": {
                    "SSEAlgorithm": "AES256"
                }
            }
        ]
    }'
```

### **2.2 Create IAM Role for Jenkins**
**Create `jenkins-iam-policy.json`:**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "lambda:*",
                "s3:*",
                "dynamodb:*",
                "apigateway:*",
                "iam:GetRole",
                "iam:PassRole",
                "iam:CreateRole",
                "iam:DeleteRole",
                "iam:AttachRolePolicy",
                "iam:DetachRolePolicy",
                "iam:PutRolePolicy",
                "iam:DeleteRolePolicy",
                "logs:*",
                "kms:*",
                "ssm:*",
                "sqs:*",
                "sts:GetCallerIdentity"
            ],
            "Resource": "*"
        }
    ]
}
```

**Create IAM user and policy:**
```bash
# Create IAM user for Jenkins
aws iam create-user --user-name jenkins-euc-lambda-poc

# Create policy
aws iam create-policy \
    --policy-name JenkinsEucLambdaPocPolicy \
    --policy-document file://jenkins-iam-policy.json

# Attach policy to user
aws iam attach-user-policy \
    --user-name jenkins-euc-lambda-poc \
    --policy-arn arn:aws:iam::YOUR-ACCOUNT-ID:policy/JenkinsEucLambdaPocPolicy

# Create access keys
aws iam create-access-key --user-name jenkins-euc-lambda-poc
```

**Save the Access Key ID and Secret Access Key for Jenkins configuration**

---

### **Step 3: Configure Jenkins Credentials**

### **3.1 Add AWS Credentials**
**Navigate to:** Manage Jenkins → Manage Credentials → Global → Add Credentials

**AWS Credentials:**
```
Kind: AWS Credentials
ID: aws-dev-account-credentials
Description: AWS Dev Account for Lambda Deployment
Access Key ID: [From Step 2.2]
Secret Access Key: [From Step 2.2]
```

### **3.2 Add Bitbucket Credentials (if using Bitbucket)**
```
Kind: Username with password
ID: bitbucket-credentials
Username: [Your Bitbucket username]
Password: [Your Bitbucket app password]
Description: Bitbucket Repository Access
```

---

### **Step 4: Prepare Your Repository**

### **4.1 Create Repository Structure**
```bash
# Initialize repository
mkdir euc-lambda-poc
cd euc-lambda-poc
git init

# Create directory structure
mkdir -p terraform
mkdir -p user-auth
mkdir -p order-processor
mkdir -p api-gateway
mkdir -p tests/unit
mkdir -p tests/integration
mkdir -p scripts

# Add .gitignore
cat > .gitignore << 'EOF'
# Python
__pycache__/
*.pyc
*.pyo
venv/
.pytest_cache/
*.egg-info/

# Terraform
*.tfstate
*.tfstate.*
.terraform/
.terraform.lock.hcl
terraform.tfvars
*.tfplan

# AWS
.aws/

# IDE
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Build artifacts
*.zip
*-build/
node_modules/
dist/
build/

# Logs
*.log
EOF
```

### **4.2 Add Terraform Configuration**
**Create `terraform/variables.tf`:**
```hcl
variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
  default     = "dev"
}

variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "project_name" {
  description = "Project name for resource naming"
  type        = string
  default     = "euc-lambda-poc"
}

variable "lambda_functions" {
  description = "List of Lambda functions to create"
  type        = list(string)
  default     = ["user-auth", "order-processor", "api-gateway"]
}

variable "lambda_runtime" {
  description = "Lambda runtime version"
  type        = string
  default     = "python3.11"
}

variable "lambda_timeout" {
  description = "Lambda function timeout in seconds"
  type        = number
  default     = 30
}

variable "lambda_memory_size" {
  description = "Lambda function memory size in MB"
  type        = number
  default     = 128
}

variable "lambda_environment_variables" {
  description = "Environment variables for Lambda functions"
  type        = map(string)
  default     = {}
}

variable "lambda_reserved_concurrency" {
  description = "Reserved concurrency for Lambda functions"
  type        = number
  default     = -1
}

variable "cloudwatch_log_retention_days" {
  description = "CloudWatch log retention in days"
  type        = number
  default     = 14
}

variable "log_retention_days" {
  description = "S3 log retention in days"
  type        = number
  default     = 90
}

variable "create_terraform_state_bucket" {
  description = "Create Terraform state bucket"
  type        = bool
  default     = false
}

variable "enable_vpc" {
  description = "Enable VPC for Lambda functions"
  type        = bool
  default     = false
}

variable "vpc_id" {
  description = "VPC ID for Lambda functions"
  type        = string
  default     = ""
}

variable "vpc_subnet_ids" {
  description = "VPC subnet IDs for Lambda functions"
  type        = list(string)
  default     = []
}

variable "enable_dlq" {
  description = "Enable dead letter queue for Lambda functions"
  type        = bool
  default     = false
}

variable "enable_api_gateway" {
  description = "Enable API Gateway for integration testing"
  type        = bool
  default     = true
}

variable "enable_sqs_testing" {
  description = "Enable SQS queue for integration testing"
  type        = bool
  default     = false
}

variable "enable_s3_testing" {
  description = "Enable S3 event notifications for testing"
  type        = bool
  default     = false
}
```

### **4.3 Add Sample Lambda Functions**
**Create `user-auth/lambda_function.py`:**
```python
import json
import os

def lambda_handler(event, context):
    """
    User authentication Lambda function
    """
    print(f"Event: {json.dumps(event)}")
    
    environment = os.environ.get('ENVIRONMENT', 'unknown')
    
    return {
        'statusCode': 200,
        'headers': {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*'
        },
        'body': json.dumps({
            'message': 'User authentication service',
            'environment': environment,
            'service': 'user-auth'
        })
    }
```

**Create `user-auth/requirements.txt`:**
```
boto3==1.34.0
requests==2.31.0
```

**Create `order-processor/lambda_function.py`:**
```python
import json
import os

def lambda_handler(event, context):
    """
    Order processing Lambda function
    """
    print(f"Event: {json.dumps(event)}")
    
    environment = os.environ.get('ENVIRONMENT', 'unknown')
    
    return {
        'statusCode': 200,
        'headers': {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*'
        },
        'body': json.dumps({
            'message': 'Order processing service',
            'environment': environment,
            'service': 'order-processor'
        })
    }
```

**Create `order-processor/requirements.txt`:**
```
boto3==1.34.0
pandas==2.1.4
```

**Create `api-gateway/lambda_function.py`:**
```python
import json
import os

def lambda_handler(event, context):
    """
    API Gateway Lambda function
    """
    print(f"Event: {json.dumps(event)}")
    
    http_method = event.get('httpMethod', 'GET')
    path = event.get('path', '/')
    environment = os.environ.get('ENVIRONMENT', 'unknown')
    
    if path == '/health':
        return {
            'statusCode': 200,
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps({
                'status': 'healthy',
                'environment': environment,
                'service': 'api-gateway'
            })
        }
    
    return {
        'statusCode': 200,
        'headers': {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*'
        },
        'body': json.dumps({
            'message': 'API Gateway service',
            'method': http_method,
            'path': path,
            'environment': environment
        })
    }
```

**Create `api-gateway/requirements.txt`:**
```
boto3==1.34.0
```

### **4.4 Add Sample Tests**
**Create `tests/unit/test_user_auth.py`:**
```python
import pytest
import json
from unittest.mock import patch
import sys
import os

# Add Lambda function to Python path
sys.path.insert(0, os.path.join(os.path.dirname(__file__), '..', '..', 'user-auth'))

from lambda_function import lambda_handler

def test_user_auth_lambda_handler():
    """Test user auth Lambda function"""
    event = {
        'httpMethod': 'POST',
        'path': '/auth/login',
        'body': json.dumps({
            'username': 'testuser',
            'password': 'testpass'
        })
    }
    
    context = {}
    
    response = lambda_handler(event, context)
    
    assert response['statusCode'] == 200
    assert 'body' in response
    
    body = json.loads(response['body'])
    assert 'message' in body
    assert body['service'] == 'user-auth'

def test_user_auth_environment_variable():
    """Test environment variable handling"""
    with patch.dict(os.environ, {'ENVIRONMENT': 'test'}):
        event = {}
        context = {}
        
        response = lambda_handler(event, context)
        body = json.loads(response['body'])
        
        assert body['environment'] == 'test'
```

**Create `tests/integration/test_basic_integration.py`:**
```python
import pytest
import boto3
import os

def test_integration_aws_connection():
    """Test basic AWS connection"""
    try:
        sts = boto3.client('sts')
        response = sts.get_caller_identity()
        assert 'Account' in response
        assert 'UserId' in response
    except Exception as e:
        pytest.skip(f"AWS credentials not configured: {e}")

def test_integration_placeholder():
    """Placeholder integration test"""
    # This will be replaced with real integration tests
    # once Terraform infrastructure is deployed
    assert True
```

### **4.5 Commit Initial Code**
```bash
# Add all files
git add .

# Initial commit
git commit -m "Initial commit: Jenkins pipeline, Terraform config, sample Lambda functions"

# Add remote (replace with your repository URL)
git remote add origin https://bitbucket.org/your-org/euc-lambda-poc.git

# Push to repository
git push -u origin main

# Create dev branch
git checkout -b dev
git push -u origin dev
```

---

### **Step 5: Deploy Initial Infrastructure**

### **5.1 Deploy Terraform Infrastructure**
```bash
# Navigate to terraform directory
cd terraform

# Initialize Terraform
terraform init \
    -backend-config="bucket=euc-lambda-poc-terraform-state-dev" \
    -backend-config="key=lambda-poc/dev.tfstate" \
    -backend-config="region=us-east-1"

# Create dev workspace
terraform workspace new dev

# Plan deployment
terraform plan -var="environment=dev"

# Apply infrastructure
terraform apply -var="environment=dev"

# Verify outputs
terraform output
```

### **5.2 Verify Infrastructure**
```bash
# Check Lambda functions
aws lambda list-functions --query 'Functions[?contains(FunctionName, `euc-lambda-poc-dev`)].FunctionName'

# Check S3 buckets
aws s3 ls | grep euc-lambda-poc

# Check DynamoDB tables
aws dynamodb list-tables --query 'TableNames[?contains(@, `euc-lambda-poc-dev`)]'
```

---

### **Step 6: Create Jenkins Job**

### **6.1 Create Multibranch Pipeline**
**In Jenkins UI:**
1. **Click:** "New Item"
2. **Enter name:** `euc-lambda-poc-pipeline`
3. **Select:** "Multibranch Pipeline"
4. **Click:** "OK"

### **6.2 Configure Branch Sources**
**Branch Sources:**
```
Source: Git
Repository URL: https://bitbucket.org/your-org/euc-lambda-poc.git
Credentials: bitbucket-credentials
```

**Behaviors:**
```
☑️ Discover branches
    Strategy: Exclude branches that are also filed as PRs
☑️ Discover pull requests from origin
    Strategy: Merging the pull request with the current target branch revision
☑️ Filter by name (with regular expression)
    Regular expression: (dev|main|staging)
```

**Build Configuration:**
```
Mode: by Jenkinsfile
Script Path: Jenkinsfile
```

**Scan Multibranch Pipeline Triggers:**
```
☑️ Periodically if not otherwise run
Interval: 1 minute
```

### **6.3 Save and Scan**
1. **Click:** "Save"
2. **Click:** "Scan Multibranch Pipeline Now"
3. **Verify:** Jenkins discovers the `dev` branch
4. **Check:** Pipeline appears in branch list

---

### **Step 7: Test the Pipeline**

### **7.1 Trigger First Build**
```bash
# Make a small change to trigger build
echo "# EUC Lambda POC" > README.md
git add README.md
git commit -m "Add README to trigger pipeline"
git push origin dev
```

### **7.2 Monitor Build Progress**
**In Jenkins:**
1. **Navigate to:** euc-lambda-poc-pipeline → dev branch
2. **Click:** Latest build number
3. **Monitor:** Console Output for real-time progress
4. **Check:** Each stage completion

### **7.3 Verify Build Results**
**Expected Results:**
```
✅ Checkout
✅ Install Dependencies
✅ Build Assets (skipped - no build files)
✅ Run Unit Tests
⚠️ Run Integration Tests (may fail initially)
✅ Deploy Static Assets (skipped - no static files)
✅ Deploy Lambda Functions
✅ Verify Deployment
```

---

### **Step 8: Set Up Bitbucket Webhook (Optional)**

### **8.1 Configure Webhook in Bitbucket**
**In Bitbucket Repository:**
1. **Go to:** Repository Settings → Webhooks
2. **Click:** "Add webhook"
3. **Configure:**
   ```
   Title: Jenkins Pipeline Trigger
   URL: http://YOUR-JENKINS-URL:8080/bitbucket-hook/
   Triggers: 
     ☑️ Repository push
     ☑️ Pull request created
     ☑️ Pull request updated
   ```
4. **Save webhook**

---

### **Step 9: Validate and Troubleshoot**

### **9.1 Common Issues and Solutions**

**Issue: Terraform state conflicts**
```bash
# Solution: Check state bucket exists and permissions
aws s3 ls s3://euc-lambda-poc-terraform-state-dev/
```

**Issue: AWS credentials not working**
```bash
# Solution: Test credentials manually
aws sts get-caller-identity
```

**Issue: Lambda functions not found**
```bash
# Solution: Check if Terraform created functions
aws lambda list-functions --query 'Functions[?contains(FunctionName, `dev`)].FunctionName'
```

**Issue: Tests failing**
```bash
# Solution: Check test dependencies and AWS access
pip install pytest boto3 moto
pytest tests/unit/ -v
```

### **9.2 Success Verification Checklist**
```
✅ Jenkins pipeline runs without errors
✅ All Lambda functions updated with latest code  
✅ Unit tests pass
✅ Integration tests run (may have failures initially)
✅ S3 buckets accessible
✅ DynamoDB tables created
✅ AWS credentials working
✅ Terraform state managed properly
✅ Git webhooks triggering builds
```

---

### **Step 10: Next Steps**

**Once everything is working:**
1. **Add real frontend** (if needed)
2. **Implement actual business logic** in Lambda functions
3. **Add comprehensive integration tests**
4. **Set up staging environment**
5. **Configure monitoring and alerting**
6. **Add security scanning**
7. **Set up production deployment**

**You now have a fully functional CI/CD pipeline that:**
- ✅ Automatically deploys on code changes
- ✅ Runs tests to ensure quality
- ✅ Manages infrastructure as code
- ✅ Provides comprehensive feedback
- ✅ Supports multiple environments

The pipeline is ready for development and can be extended with additional features as needed!
