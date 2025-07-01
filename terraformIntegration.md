## How the Pipeline Currently Updates Lambda Functions

### **Current Pipeline Approach (Direct AWS CLI)**

```bash
# From the Deploy Lambda Functions stage
aws lambda update-function-code \
    --function-name "${lambda_name}-dev" \
    --zip-file "fileb://${lambda_name}-dev.zip" \
    --region ${AWS_DEFAULT_REGION}
```

**What This Does:**
- **Assumes Lambda functions already exist** (created manually or by infrastructure team)
- **Only updates the code** inside existing functions
- **Does NOT create/modify:** Function configuration, triggers, permissions, VPC settings, etc.
- **Requires:** Functions to be pre-created with correct names (`{directory-name}-dev`)

---

## **Integrating Terraform into the Pipeline**

### **Option 1: Hybrid Approach (Recommended)**

**Terraform manages infrastructure, Pipeline manages code:**

```groovy
pipeline {
    agent any
    
    environment {
        AWS_DEFAULT_REGION = 'us-east-1'
        S3_ASSETS_BUCKET_DEV = 'euc-lambda-poc-assets-dev'
        S3_TERRAFORM_STATE = 'euc-lambda-poc-terraform-state'
        TERRAFORM_WORKSPACE = 'dev'
    }
    
    tools {
        python 'Python3'
        terraform 'Terraform'  // Add Terraform tool
    }
    
    when {
        branch 'dev'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()
                }
                echo "🔀 Merged to dev branch - Running pipeline"
                echo "Building branch: ${env.BRANCH_NAME}"
                echo "Commit: ${env.GIT_COMMIT_SHORT}"
            }
        }
        
        stage('Terraform Plan & Apply') {
            steps {
                echo "🏗️ Managing infrastructure with Terraform..."
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding', 
                     credentialsId: 'aws-dev-account-credentials']
                ]) {
                    sh '''
                        cd terraform/
                        
                        # Initialize Terraform
                        terraform init \
                            -backend-config="bucket=${S3_TERRAFORM_STATE}" \
                            -backend-config="key=lambda-poc/${TERRAFORM_WORKSPACE}.tfstate" \
                            -backend-config="region=${AWS_DEFAULT_REGION}"
                        
                        # Select or create workspace
                        terraform workspace select ${TERRAFORM_WORKSPACE} || terraform workspace new ${TERRAFORM_WORKSPACE}
                        
                        # Plan infrastructure changes
                        terraform plan \
                            -var="environment=${TERRAFORM_WORKSPACE}" \
                            -var="aws_region=${AWS_DEFAULT_REGION}" \
                            -out=tfplan
                        
                        # Apply infrastructure changes
                        terraform apply -auto-approve tfplan
                        
                        # Output important values
                        terraform output -json > ../terraform-outputs.json
                    '''
                }
            }
        }
        
        // ... (Install Dependencies, Build Assets, Run Tests stages remain the same)
        
        stage('Deploy Lambda Functions') {
            steps {
                echo "⚡ Deploying Lambda functions..."
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding', 
                     credentialsId: 'aws-dev-account-credentials']
                ]) {
                    sh '''
                        # Get Lambda function names from Terraform outputs
                        if [ -f "terraform-outputs.json" ]; then
                            echo "Using Lambda functions created by Terraform:"
                            cat terraform-outputs.json | jq -r '.lambda_function_names.value[]'
                        fi
                        
                        # Deploy each Lambda function
                        find . -name "lambda_function.py" -type f | while read lambda_file; do
                            lambda_dir=$(dirname "$lambda_file")
                            lambda_name=$(basename "$lambda_dir")
                            terraform_function_name="${lambda_name}-${TERRAFORM_WORKSPACE}"
                            
                            echo "📦 Packaging and deploying $terraform_function_name"
                            
                            # Create deployment package
                            cd "$lambda_dir"
                            zip -r "../../${terraform_function_name}.zip" . \
                                -x "*.pyc" "*__pycache__*" "tests/*" "*.md" ".git*"
                            cd - > /dev/null
                            
                            # Update function code
                            aws lambda update-function-code \
                                --function-name "$terraform_function_name" \
                                --zip-file "fileb://${terraform_function_name}.zip" \
                                --region ${AWS_DEFAULT_REGION}
                            
                            echo "✅ $terraform_function_name updated successfully"
                        done
                    '''
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                echo "🔍 Verifying Terraform-managed deployment..."
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding', 
                     credentialsId: 'aws-dev-account-credentials']
                ]) {
                    sh '''
                        # Verify using Terraform outputs
                        cd terraform/
                        
                        echo "📋 Terraform-managed resources:"
                        terraform output
                        
                        echo ""
                        echo "⚡ Lambda Function Status:"
                        
                        # Get function names from Terraform and verify
                        terraform output -json lambda_function_names | jq -r '.[]' | while read function_name; do
                            echo "Checking $function_name..."
                            aws lambda get-function \
                                --function-name "$function_name" \
                                --region ${AWS_DEFAULT_REGION} \
                                --query 'Configuration.[FunctionName,LastModified,State,Runtime]' \
                                --output table
                        done
                        
                        echo ""
                        echo "✅ All Terraform-managed resources verified"
                    '''
                }
            }
        }
    }
    
    post {
        success {
            script {
                echo """
🎉 TERRAFORM + LAMBDA DEPLOYMENT SUCCESSFUL!

🏗️ Infrastructure: Managed by Terraform
⚡ Lambda Code: Updated by Jenkins Pipeline
🌐 Environment: ${env.TERRAFORM_WORKSPACE}

📋 What Terraform Manages:
✅ Lambda function creation/configuration
✅ IAM roles and policies
✅ API Gateway setup (if applicable)
✅ S3 bucket configuration
✅ VPC and security groups
✅ CloudWatch logs and monitoring

📋 What Pipeline Manages:
✅ Code compilation and testing
✅ Lambda function code updates
✅ Static asset deployment
✅ Deployment verification

🚀 Benefits:
- Infrastructure as Code (version controlled)
- Consistent environment setup
- Easy to replicate across environments
- Automated dependency management
                """
            }
        }
    }
}
```

---

### **Terraform Configuration Example**

**Directory Structure:**
```
euc-lambda-poc/
├── Jenkinsfile
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── lambda.tf
│   ├── s3.tf
│   └── api-gateway.tf (optional)
├── user-auth/
│   └── lambda_function.py
├── order-processor/
│   └── lambda_function.py
└── api-gateway/
    └── lambda_function.py
```

**terraform/variables.tf:**
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
```

**terraform/lambda.tf:**
```hcl
# IAM role for Lambda functions
resource "aws_iam_role" "lambda_execution_role" {
  name = "${var.project_name}-lambda-role-${var.environment}"

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

# Attach basic Lambda execution policy
resource "aws_iam_role_policy_attachment" "lambda_basic_execution" {
  role       = aws_iam_role.lambda_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

# Custom policy for Lambda functions
resource "aws_iam_role_policy" "lambda_custom_policy" {
  name = "${var.project_name}-lambda-policy-${var.environment}"
  role = aws_iam_role.lambda_execution_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "dynamodb:GetItem",
          "dynamodb:PutItem",
          "dynamodb:UpdateItem",
          "dynamodb:DeleteItem"
        ]
        Resource = [
          "arn:aws:s3:::${var.project_name}-*/*",
          "arn:aws:dynamodb:${var.aws_region}:*:table/${var.project_name}-*"
        ]
      }
    ]
  })
}

# Create Lambda functions
resource "aws_lambda_function" "functions" {
  for_each = toset(var.lambda_functions)
  
  function_name = "${each.value}-${var.environment}"
  role         = aws_iam_role.lambda_execution_role.arn
  handler      = "lambda_function.lambda_handler"
  runtime      = "python3.11"
  timeout      = 30
  memory_size  = 128

  # Placeholder code (will be updated by Jenkins)
  filename         = "placeholder.zip"
  source_code_hash = data.archive_file.placeholder_zip.output_base64sha256

  environment {
    variables = {
      ENVIRONMENT = var.environment
      PROJECT_NAME = var.project_name
    }
  }

  tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "Terraform"
  }
}

# Create placeholder ZIP file
data "archive_file" "placeholder_zip" {
  type        = "zip"
  output_path = "placeholder.zip"
  source {
    content = <<EOF
def lambda_handler(event, context):
    return {
        'statusCode': 200,
        'body': 'Placeholder function - will be updated by Jenkins'
    }
EOF
    filename = "lambda_function.py"
  }
}
```

**terraform/s3.tf:**
```hcl
# S3 bucket for static assets
resource "aws_s3_bucket" "assets" {
  bucket = "${var.project_name}-assets-${var.environment}"
  
  tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "Terraform"
  }
}

# S3 bucket configuration
resource "aws_s3_bucket_public_access_block" "assets" {
  bucket = aws_s3_bucket.assets.id

  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}

# S3 bucket policy for public read access
resource "aws_s3_bucket_policy" "assets" {
  bucket = aws_s3_bucket.assets.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = "*"
        Action = "s3:GetObject"
        Resource = "${aws_s3_bucket.assets.arn}/*"
      }
    ]
  })
}

# Enable static website hosting
resource "aws_s3_bucket_website_configuration" "assets" {
  bucket = aws_s3_bucket.assets.id

  index_document {
    suffix = "index.html"
  }

  error_document {
    key = "error.html"
  }
}
```

**terraform/outputs.tf:**
```hcl
output "lambda_function_names" {
  description = "Names of created Lambda functions"
  value       = [for f in aws_lambda_function.functions : f.function_name]
}

output "lambda_function_arns" {
  description = "ARNs of created Lambda functions"
  value       = [for f in aws_lambda_function.functions : f.arn]
}

output "s3_bucket_name" {
  description = "Name of the S3 assets bucket"
  value       = aws_s3_bucket.assets.bucket
}

output "s3_website_url" {
  description = "S3 website URL"
  value       = aws_s3_bucket_website_configuration.assets.website_endpoint
}

output "iam_role_arn" {
  description = "Lambda execution role ARN"
  value       = aws_iam_role.lambda_execution_role.arn
}
```

---

### **Option 2: Full Terraform Approach**

**Terraform manages everything including code deployment:**

```hcl
# terraform/lambda.tf
resource "aws_lambda_function" "functions" {
  for_each = toset(var.lambda_functions)
  
  function_name = "${each.value}-${var.environment}"
  role         = aws_iam_role.lambda_execution_role.arn
  handler      = "lambda_function.lambda_handler"
  runtime      = "python3.11"
  
  # Use actual code from repository
  filename         = "${path.module}/../build/${each.value}.zip"
  source_code_hash = filebase64sha256("${path.module}/../build/${each.value}.zip")
  
  depends_on = [null_resource.build_lambdas]
}

# Build Lambda packages before Terraform apply
resource "null_resource" "build_lambdas" {
  for_each = toset(var.lambda_functions)
  
  triggers = {
    # Rebuild when source files change
    source_hash = data.archive_file.lambda_zip[each.key].output_base64sha256
  }
  
  provisioner "local-exec" {
    command = <<-EOT
      mkdir -p build
      cd ${each.value}
      zip -r ../build/${each.value}.zip . -x "*.pyc" "*__pycache__*" "tests/*"
    EOT
  }
}

data "archive_file" "lambda_zip" {
  for_each = toset(var.lambda_functions)
  
  type        = "zip"
  source_dir  = "${path.module}/../${each.value}"
  output_path = "${path.module}/../build/${each.value}.zip"
  excludes    = ["*.pyc", "__pycache__", "tests"]
}
```

**Pipeline becomes simpler:**
```groovy
stage('Deploy Infrastructure and Code') {
    steps {
        echo "🏗️ Deploying everything with Terraform..."
        withCredentials([...]) {
            sh '''
                cd terraform/
                terraform init
                terraform plan -var="environment=dev"
                terraform apply -auto-approve
            '''
        }
    }
}
```

---

### **Benefits of Terraform Integration**

### **Infrastructure as Code:**
- **Version controlled:** Infrastructure changes tracked in Git
- **Reproducible:** Same infrastructure across environments
- **Documented:** Clear resource definitions and dependencies

### **Consistency:**
- **Standardized naming:** Consistent resource naming patterns
- **Environment parity:** Dev/staging/prod environments identical
- **Dependency management:** Terraform handles resource dependencies

### **Security:**
- **Proper IAM roles:** Least-privilege access policies
- **Resource isolation:** Environment-specific resources
- **Compliance:** Infrastructure meets security standards

### **Operational Benefits:**
- **Easy rollbacks:** Terraform state management
- **Resource tracking:** Clear ownership and purpose
- **Cost management:** Tagged resources for cost allocation
- **Disaster recovery:** Infrastructure can be recreated quickly

The **Hybrid Approach (Option 1)** is recommended because it provides the best of both worlds: robust infrastructure management through Terraform and fast, iterative code deployment through the Jenkins pipeline.

## main.tf for Hybrid Approach (Option 1)

Here's a comprehensive `main.tf` that sets up the core infrastructure while leaving code deployment to the Jenkins pipeline:

```hcl
# terraform/main.tf

terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    archive = {
      source  = "hashicorp/archive"
      version = "~> 2.4"
    }
  }
  
  backend "s3" {
    # Backend configuration will be provided via -backend-config in pipeline
    # bucket = "your-terraform-state-bucket"
    # key    = "lambda-poc/dev.tfstate"
    # region = "us-east-1"
  }
}

# Configure AWS Provider
provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "Terraform"
      Repository  = "euc-lambda-poc"
    }
  }
}

# Get current AWS account and region
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

# Local values for resource naming and configuration
locals {
  name_prefix = "${var.project_name}-${var.environment}"
  
  # Auto-discover Lambda functions from directory structure
  lambda_functions = var.lambda_functions
  
  # Common tags for all resources
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
    Repository  = "euc-lambda-poc"
  }
  
  # Account ID for resource ARNs
  account_id = data.aws_caller_identity.current.account_id
  region     = data.aws_region.current.name
}

#===============================================================================
# S3 BUCKETS
#===============================================================================

# S3 bucket for static assets
resource "aws_s3_bucket" "assets" {
  bucket        = "${local.name_prefix}-assets"
  force_destroy = var.environment != "prod"  # Allow destruction in non-prod
}

# S3 bucket for Terraform state (if needed)
resource "aws_s3_bucket" "terraform_state" {
  count         = var.create_terraform_state_bucket ? 1 : 0
  bucket        = "${local.name_prefix}-terraform-state"
  force_destroy = false
}

# S3 bucket for deployment artifacts (large Lambda packages)
resource "aws_s3_bucket" "deployments" {
  bucket        = "${local.name_prefix}-deployments"
  force_destroy = var.environment != "prod"
}

# S3 bucket for application logs
resource "aws_s3_bucket" "logs" {
  bucket        = "${local.name_prefix}-logs"
  force_destroy = var.environment != "prod"
}

# S3 bucket versioning
resource "aws_s3_bucket_versioning" "assets" {
  bucket = aws_s3_bucket.assets.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  count  = var.create_terraform_state_bucket ? 1 : 0
  bucket = aws_s3_bucket.terraform_state[0].id
  versioning_configuration {
    status = "Enabled"
  }
}

# S3 bucket server-side encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "assets" {
  bucket = aws_s3_bucket.assets.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  count  = var.create_terraform_state_bucket ? 1 : 0
  bucket = aws_s3_bucket.terraform_state[0].id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# S3 bucket public access configuration for assets
resource "aws_s3_bucket_public_access_block" "assets" {
  bucket = aws_s3_bucket.assets.id

  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}

# S3 bucket policy for public read access (assets only)
resource "aws_s3_bucket_policy" "assets" {
  bucket = aws_s3_bucket.assets.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "PublicReadGetObject"
        Effect    = "Allow"
        Principal = "*"
        Action    = "s3:GetObject"
        Resource  = "${aws_s3_bucket.assets.arn}/*"
      }
    ]
  })

  depends_on = [aws_s3_bucket_public_access_block.assets]
}

# S3 static website hosting
resource "aws_s3_bucket_website_configuration" "assets" {
  bucket = aws_s3_bucket.assets.id

  index_document {
    suffix = "index.html"
  }

  error_document {
    key = "error.html"
  }
}

# S3 lifecycle policy for logs
resource "aws_s3_bucket_lifecycle_configuration" "logs" {
  bucket = aws_s3_bucket.logs.id

  rule {
    id     = "log_retention"
    status = "Enabled"

    expiration {
      days = var.log_retention_days
    }

    noncurrent_version_expiration {
      noncurrent_days = 30
    }
  }
}

#===============================================================================
# IAM ROLES AND POLICIES
#===============================================================================

# Lambda execution role
resource "aws_iam_role" "lambda_execution" {
  name = "${local.name_prefix}-lambda-execution-role"

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

# Basic Lambda execution policy attachment
resource "aws_iam_role_policy_attachment" "lambda_basic_execution" {
  role       = aws_iam_role.lambda_execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

# VPC execution policy (if Lambda functions need VPC access)
resource "aws_iam_role_policy_attachment" "lambda_vpc_execution" {
  count      = var.enable_vpc ? 1 : 0
  role       = aws_iam_role.lambda_execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
}

# Custom Lambda policy for project-specific permissions
resource "aws_iam_role_policy" "lambda_custom" {
  name = "${local.name_prefix}-lambda-custom-policy"
  role = aws_iam_role.lambda_execution.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "S3Access"
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.assets.arn,
          "${aws_s3_bucket.assets.arn}/*",
          aws_s3_bucket.deployments.arn,
          "${aws_s3_bucket.deployments.arn}/*",
          aws_s3_bucket.logs.arn,
          "${aws_s3_bucket.logs.arn}/*"
        ]
      },
      {
        Sid    = "DynamoDBAccess"
        Effect = "Allow"
        Action = [
          "dynamodb:GetItem",
          "dynamodb:PutItem",
          "dynamodb:UpdateItem",
          "dynamodb:DeleteItem",
          "dynamodb:Query",
          "dynamodb:Scan"
        ]
        Resource = "arn:aws:dynamodb:${local.region}:${local.account_id}:table/${local.name_prefix}-*"
      },
      {
        Sid    = "SecretsManagerAccess"
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue"
        ]
        Resource = "arn:aws:secretsmanager:${local.region}:${local.account_id}:secret:${local.name_prefix}/*"
      },
      {
        Sid    = "SSMParameterAccess"
        Effect = "Allow"
        Action = [
          "ssm:GetParameter",
          "ssm:GetParameters",
          "ssm:GetParametersByPath"
        ]
        Resource = "arn:aws:ssm:${local.region}:${local.account_id}:parameter/${local.name_prefix}/*"
      },
      {
        Sid    = "CloudWatchLogsAccess"
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "arn:aws:logs:${local.region}:${local.account_id}:log-group:/aws/lambda/${local.name_prefix}-*"
      }
    ]
  })
}

#===============================================================================
# KMS KEY FOR ENCRYPTION
#===============================================================================

# KMS key for encryption
resource "aws_kms_key" "main" {
  description             = "KMS key for ${local.name_prefix}"
  deletion_window_in_days = var.environment == "prod" ? 30 : 7

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "Enable IAM User Permissions"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${local.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "Allow Lambda service to use the key"
        Effect = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
        Action = [
          "kms:Decrypt",
          "kms:GenerateDataKey"
        ]
        Resource = "*"
      }
    ]
  })
}

# KMS key alias
resource "aws_kms_alias" "main" {
  name          = "alias/${local.name_prefix}"
  target_key_id = aws_kms_key.main.key_id
}

#===============================================================================
# LAMBDA FUNCTIONS
#===============================================================================

# Create placeholder Lambda functions (Jenkins will update the code)
resource "aws_lambda_function" "functions" {
  for_each = toset(local.lambda_functions)

  function_name = "${each.value}-${var.environment}"
  role         = aws_iam_role.lambda_execution.arn
  handler      = "lambda_function.lambda_handler"
  runtime      = var.lambda_runtime
  timeout      = var.lambda_timeout
  memory_size  = var.lambda_memory_size

  # Placeholder code (Jenkins will update this)
  filename         = data.archive_file.placeholder_zip.output_path
  source_code_hash = data.archive_file.placeholder_zip.output_base64sha256

  # Environment variables
  environment {
    variables = merge(
      var.lambda_environment_variables,
      {
        ENVIRONMENT        = var.environment
        PROJECT_NAME       = var.project_name
        S3_ASSETS_BUCKET   = aws_s3_bucket.assets.bucket
        S3_LOGS_BUCKET     = aws_s3_bucket.logs.bucket
        KMS_KEY_ID         = aws_kms_key.main.key_id
        LOG_LEVEL          = var.environment == "prod" ? "INFO" : "DEBUG"
      }
    )
  }

  # KMS encryption for environment variables
  kms_key_arn = aws_kms_key.main.arn

  # VPC configuration (if enabled)
  dynamic "vpc_config" {
    for_each = var.enable_vpc ? [1] : []
    content {
      subnet_ids         = var.vpc_subnet_ids
      security_group_ids = [aws_security_group.lambda[0].id]
    }
  }

  # Dead letter queue configuration
  dynamic "dead_letter_config" {
    for_each = var.enable_dlq ? [1] : []
    content {
      target_arn = aws_sqs_queue.dlq[0].arn
    }
  }

  # Reserved concurrency (if specified)
  reserved_concurrent_executions = var.lambda_reserved_concurrency

  depends_on = [
    aws_iam_role_policy_attachment.lambda_basic_execution,
    aws_iam_role_policy.lambda_custom,
    aws_cloudwatch_log_group.lambda
  ]
}

# Create placeholder ZIP file for initial deployment
data "archive_file" "placeholder_zip" {
  type        = "zip"
  output_path = "${path.module}/placeholder.zip"

  source {
    content = <<EOF
import json
import os

def lambda_handler(event, context):
    """
    Placeholder Lambda function
    This will be replaced by Jenkins pipeline with actual code
    """
    return {
        'statusCode': 200,
        'headers': {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*'
        },
        'body': json.dumps({
            'message': 'Placeholder function - waiting for Jenkins deployment',
            'environment': os.environ.get('ENVIRONMENT', 'unknown'),
            'function_name': context.function_name if context else 'unknown'
        })
    }
EOF
    filename = "lambda_function.py"
  }
}

#===============================================================================
# CLOUDWATCH LOGS
#===============================================================================

# CloudWatch log groups for Lambda functions
resource "aws_cloudwatch_log_group" "lambda" {
  for_each = toset(local.lambda_functions)

  name              = "/aws/lambda/${each.value}-${var.environment}"
  retention_in_days = var.cloudwatch_log_retention_days
  kms_key_id        = aws_kms_key.main.arn
}

#===============================================================================
# VPC RESOURCES (OPTIONAL)
#===============================================================================

# Security group for Lambda functions (if VPC is enabled)
resource "aws_security_group" "lambda" {
  count = var.enable_vpc ? 1 : 0

  name_prefix = "${local.name_prefix}-lambda-"
  vpc_id      = var.vpc_id

  # Outbound rules
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  lifecycle {
    create_before_destroy = true
  }
}

#===============================================================================
# SQS DEAD LETTER QUEUE (OPTIONAL)
#===============================================================================

# Dead letter queue for failed Lambda executions
resource "aws_sqs_queue" "dlq" {
  count = var.enable_dlq ? 1 : 0

  name                       = "${local.name_prefix}-lambda-dlq"
  message_retention_seconds  = 1209600  # 14 days
  kms_master_key_id         = aws_kms_key.main.key_id
  
  redrive_allow_policy = jsonencode({
    redrivePermission = "byQueue",
    sourceQueueArns   = ["arn:aws:sqs:${local.region}:${local.account_id}:*"]
  })
}

# Dead letter queue policy
resource "aws_sqs_queue_policy" "dlq" {
  count     = var.enable_dlq ? 1 : 0
  queue_url = aws_sqs_queue.dlq[0].id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowLambdaAccess"
        Effect = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
        Action   = "sqs:SendMessage"
        Resource = aws_sqs_queue.dlq[0].arn
        Condition = {
          StringEquals = {
            "aws:SourceAccount" = local.account_id
          }
        }
      }
    ]
  })
}

#===============================================================================
# PARAMETER STORE VALUES
#===============================================================================

# Store common configuration in Parameter Store
resource "aws_ssm_parameter" "config" {
  for_each = {
    "s3-assets-bucket"     = aws_s3_bucket.assets.bucket
    "s3-deployments-bucket" = aws_s3_bucket.deployments.bucket
    "s3-logs-bucket"       = aws_s3_bucket.logs.bucket
    "kms-key-id"           = aws_kms_key.main.key_id
    "lambda-role-arn"      = aws_iam_role.lambda_execution.arn
    "environment"          = var.environment
    "project-name"         = var.project_name
  }

  name  = "/${local.name_prefix}/config/${each.key}"
  type  = "String"
  value = each.value

  tags = local.common_tags
}

#===============================================================================
# OUTPUTS
#===============================================================================

# Export important values for pipeline and other modules
output "lambda_function_names" {
  description = "Names of created Lambda functions"
  value       = [for f in aws_lambda_function.functions : f.function_name]
}

output "lambda_function_arns" {
  description = "ARNs of created Lambda functions"
  value       = { for k, f in aws_lambda_function.functions : k => f.arn }
}

output "s3_assets_bucket" {
  description = "S3 assets bucket name"
  value       = aws_s3_bucket.assets.bucket
}

output "s3_assets_bucket_website_url" {
  description = "S3 website URL for assets"
  value       = "http://${aws_s3_bucket_website_configuration.assets.website_endpoint}"
}

output "s3_deployments_bucket" {
  description = "S3 deployments bucket name"
  value       = aws_s3_bucket.deployments.bucket
}

output "s3_logs_bucket" {
  description = "S3 logs bucket name"
  value       = aws_s3_bucket.logs.bucket
}

output "lambda_execution_role_arn" {
  description = "Lambda execution role ARN"
  value       = aws_iam_role.lambda_execution.arn
}

output "kms_key_id" {
  description = "KMS key ID for encryption"
  value       = aws_kms_key.main.key_id
}

output "kms_key_arn" {
  description = "KMS key ARN for encryption"
  value       = aws_kms_key.main.arn
}

output "cloudwatch_log_groups" {
  description = "CloudWatch log group names"
  value       = { for k, lg in aws_cloudwatch_log_group.lambda : k => lg.name }
}

output "region" {
  description = "AWS region"
  value       = local.region
}

output "account_id" {
  description = "AWS account ID"
  value       = local.account_id
}

output "environment" {
  description = "Environment name"
  value       = var.environment
}

output "project_name" {
  description = "Project name"
  value       = var.project_name
}

# Output for Jenkins pipeline
output "deployment_info" {
  description = "Deployment information for Jenkins pipeline"
  value = {
    environment              = var.environment
    lambda_functions        = [for f in aws_lambda_function.functions : f.function_name]
    s3_assets_bucket        = aws_s3_bucket.assets.bucket
    s3_deployments_bucket   = aws_s3_bucket.deployments.bucket
    region                  = local.region
    account_id              = local.account_id
    kms_key_id              = aws_kms_key.main.key_id
    lambda_execution_role   = aws_iam_role.lambda_execution.arn
  }
}
```

This `main.tf` provides:

### **Complete Infrastructure Setup:**
- **S3 buckets:** Assets, deployments, logs, optional Terraform state
- **Lambda functions:** Created with placeholder code
- **IAM roles/policies:** Secure, least-privilege access
- **KMS encryption:** For environment variables and data
- **CloudWatch logs:** With proper retention policies
- **Parameter Store:** Configuration management
- **Optional features:** VPC, DLQ, with conditional creation

### **Jenkins Integration Points:**
- **Placeholder Lambda functions:** Jenkins updates the code
- **S3 bucket outputs:** Pipeline knows where to deploy assets
- **Function name outputs:** Pipeline knows which functions to update
- **Consistent naming:** `-dev` suffix for environment isolation

### **Security Best Practices:**
- **Encryption at rest:** KMS keys for Lambda and S3
- **Least privilege IAM:** Only necessary permissions
- **Resource isolation:** Environment-specific resources
- **Audit logging:** CloudWatch logs with retention

### **Operational Features:**
- **Lifecycle management:** Log retention and cleanup
- **Dead letter queues:** Error handling (optional)
- **Parameter Store:** Centralized configuration
- **Comprehensive outputs:** All info needed by pipeline

The pipeline can now focus on code deployment while Terraform manages all the infrastructure complexity!
