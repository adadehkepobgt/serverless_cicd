**Complete Guide: Terraform for Serverless Lambda CI/CD**

## **What is Terraform?**

Terraform is an Infrastructure as Code (IaC) tool that allows you to define, provision, and manage cloud infrastructure using declarative configuration files. Think of it as "code that creates AWS resources" - instead of clicking through AWS Console, you write code that automatically creates your Lambda functions, IAM roles, DynamoDB tables, etc.

## **Core Terraform Concepts**

**Infrastructure as Code (IaC) Benefits:**
```
Traditional Approach          vs.         Terraform Approach
┌─────────────────┐                      ┌─────────────────┐
│ Manual AWS      │                      │  Write .tf      │
│ Console Clicks  │                      │  files once     │
│                 │                      │                 │
│ ❌ Error-prone   │                      │ ✅ Repeatable   │
│ ❌ Not tracked   │                      │ ✅ Version ctrl │
│ ❌ Hard to share │                      │ ✅ Collaborative│
│ ❌ No rollback   │                      │ ✅ Easy rollback│
└─────────────────┘                      └─────────────────┘
```

**Key Terraform Terminology:**

**Configuration Files (.tf):** Human-readable files that describe infrastructure
**State File:** Terraform's "memory" of what it has created
**Provider:** Plugin that knows how to talk to specific cloud (AWS, Azure, GCP)
**Resource:** Individual infrastructure component (Lambda function, S3 bucket)
**Module:** Reusable collection of resources
**Plan:** Preview of what Terraform will change
**Apply:** Execute the planned changes

## **How Terraform Works**

**Terraform Workflow:**
```
1. Write Config    2. Plan         3. Apply        4. State Update
┌─────────────┐   ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│   .tf files │──►│terraform    │─►│terraform    │─►│ terraform.  │
│             │   │plan         │ │apply        │ │ tfstate     │
│ main.tf     │   │             │ │             │ │             │
│ variables.tf│   │Shows changes│ │Makes changes│ │Tracks state │
└─────────────┘   └─────────────┘ └─────────────┘ └─────────────┘
```

**Detailed Execution Process:**

**1. Initialization (`terraform init`):**
```
┌─────────────────────────────────────────────────────────────┐
│ terraform init                                              │
├─────────────────────────────────────────────────────────────┤
│ ✓ Download required providers (AWS, archive, etc.)         │
│ ✓ Configure backend (S3 for state storage)                 │
│ ✓ Download modules from registry or local paths            │
│ ✓ Create .terraform/ directory with dependencies           │
│ ✓ Initialize state file if it doesn't exist                │
└─────────────────────────────────────────────────────────────┘
```

**2. Planning (`terraform plan`):**
```
Current State    Desired State    Generated Plan
┌───────────┐   ┌───────────┐    ┌─────────────────┐
│ Lambda    │   │ Lambda    │    │ + Create API    │
│ Function  │ + │ Function  │ =  │   Gateway       │
│           │   │ API GW    │    │ ~ Update Lambda │
│           │   │ DynamoDB  │    │   memory: 128→256│
└───────────┘   └───────────┘    │ + Create DynamoDB│
                                 └─────────────────┘
```

**3. Applying (`terraform apply`):**
```
Terraform Plan Execution:
├── Create resources in dependency order
├── Update existing resources if needed
├── Delete resources marked for removal
├── Update state file with current reality
└── Show summary of changes made
```

## **Terraform Configuration Language (HCL)**

**Basic Syntax Structure:**
```hcl
# This is a comment

# Resource Block
resource "aws_lambda_function" "my_function" {
  function_name = "my-lambda"
  runtime      = "python3.9"
  handler      = "lambda_function.lambda_handler"
  
  # Nested block
  environment {
    variables = {
      ENV = "production"
    }
  }
}

# Variable Block
variable "function_name" {
  description = "Name of the Lambda function"
  type        = string
  default     = "my-default-function"
}

# Output Block
output "function_arn" {
  value = aws_lambda_function.my_function.arn
}
```

**Data Types in Terraform:**
```hcl
# String
variable "name" {
  type = string
  default = "my-function"
}

# Number
variable "memory_size" {
  type = number
  default = 128
}

# Boolean
variable "enable_monitoring" {
  type = bool
  default = true
}

# List
variable "subnet_ids" {
  type = list(string)
  default = ["subnet-123", "subnet-456"]
}

# Map
variable "tags" {
  type = map(string)
  default = {
    Environment = "dev"
    Project     = "lambda-app"
  }
}

# Object (complex type)
variable "lambda_config" {
  type = object({
    name    = string
    memory  = number
    timeout = number
  })
  default = {
    name    = "my-function"
    memory  = 256
    timeout = 30
  }
}
```

## **Terraform State Management**

**What is State?**
State is Terraform's way of tracking what infrastructure it has created and managing the relationship between your configuration files and real-world resources.

**State File Contents:**
```json
{
  "version": 4,
  "terraform_version": "1.6.0",
  "resources": [
    {
      "type": "aws_lambda_function",
      "name": "my_function",
      "instances": [
        {
          "attributes": {
            "arn": "arn:aws:lambda:us-east-1:123456789:function:my-function",
            "function_name": "my-function",
            "memory_size": 256,
            "last_modified": "2024-06-24T10:30:00.000+0000"
          }
        }
      ]
    }
  ]
}
```

**Local vs Remote State:**

**Local State (Development):**
```
project/
├── main.tf
├── variables.tf
└── terraform.tfstate  ← Stored locally
```

**Remote State (Production):**
```
Local Machine           Remote Backend (S3)
┌─────────────┐       ┌─────────────────────┐
│  main.tf    │       │  S3 Bucket          │
│  variables.tf│  ────►│  terraform.tfstate  │
│             │       │                     │
│ (No local   │       │  + Version history  │
│  state file)│       │  + Locking with     │
│             │       │    DynamoDB         │
└─────────────┘       └─────────────────────┘
```

**State Locking:**
```
Developer A starts terraform apply
         ↓
DynamoDB lock table creates lock record
         ↓
Developer B tries terraform apply
         ↓
Terraform sees lock, waits or fails
         ↓
Developer A finishes, lock is released
         ↓
Developer B can now proceed
```

## **Backend Configuration for Your Use Case**

**S3 Backend Setup:**
```hcl
terraform {
  backend "s3" {
    bucket         = "your-company-terraform-state-dev"
    key            = "lambda-functions/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock-dev"
    encrypt        = true
  }
}
```

**What Each Setting Does:**
- **bucket**: S3 bucket where state file is stored
- **key**: Path within bucket (like a file path)
- **region**: AWS region for S3 bucket
- **dynamodb_table**: Table for state locking
- **encrypt**: Encrypts state file at rest

**Multiple Environments with Same Backend:**
```
Backend Configuration per Environment:

Dev:    bucket = "company-terraform-state-dev"
        key    = "lambda-functions/terraform.tfstate"

QA:     bucket = "company-terraform-state-qa"  
        key    = "lambda-functions/terraform.tfstate"

Prod:   bucket = "company-terraform-state-prod"
        key    = "lambda-functions/terraform.tfstate"
```

## **Terraform Modules for Lambda Functions**

**Why Use Modules?**
```
Without Modules (Repetitive):          With Modules (DRY):
┌─────────────────────────────┐       ┌─────────────────────────────┐
│ dev/main.tf                 │       │ modules/lambda/main.tf      │
│   - Lambda function config  │       │   - Lambda function config  │
│   - IAM role config         │       │   - IAM role config         │
│   - CloudWatch logs config  │       │   - CloudWatch logs config  │
├─────────────────────────────┤       ├─────────────────────────────┤
│ qa/main.tf                  │       │ dev/main.tf                 │
│   - Same Lambda config     │  ───► │   module "lambda" {         │
│   - Same IAM role config    │       │     source = "../modules/   │
│   - Same CloudWatch config  │       │              lambda"        │
├─────────────────────────────┤       │     name = "my-func-dev"    │
│ prod/main.tf                │       │   }                         │
│   - Same configs again...   │       └─────────────────────────────┘
└─────────────────────────────┘
```

**Module Structure for Lambda:**
```
modules/lambda-function/
├── main.tf        # Resource definitions
├── variables.tf   # Input variables
├── outputs.tf     # Output values
└── README.md      # Documentation
```

**Module Input Variables (`variables.tf`):**
```hcl
variable "function_name" {
  description = "Name of the Lambda function"
  type        = string
}

variable "source_dir" {
  description = "Directory containing Lambda source code"
  type        = string
}

variable "runtime" {
  description = "Lambda runtime"
  type        = string
  default     = "python3.9"
}

variable "memory_size" {
  description = "Memory allocation for Lambda"
  type        = number
  default     = 128
  
  validation {
    condition     = var.memory_size >= 128 && var.memory_size <= 10240
    error_message = "Memory size must be between 128 MB and 10,240 MB."
  }
}

variable "environment_variables" {
  description = "Environment variables for Lambda"
  type        = map(string)
  default     = {}
}

variable "tags" {
  description = "Tags to apply to all resources"
  type        = map(string)
  default     = {}
}
```

**Module Outputs (`outputs.tf`):**
```hcl
output "function_name" {
  description = "Name of the created Lambda function"
  value       = aws_lambda_function.main.function_name
}

output "function_arn" {
  description = "ARN of the Lambda function"
  value       = aws_lambda_function.main.arn
}

output "invoke_arn" {
  description = "ARN for invoking the Lambda function"
  value       = aws_lambda_function.main.invoke_arn
}

output "role_arn" {
  description = "ARN of the Lambda execution role"
  value       = aws_iam_role.lambda_role.arn
}
```

## **Lambda-Specific Terraform Resources**

**Core Lambda Resources:**

**1. Lambda Function:**
```hcl
resource "aws_lambda_function" "main" {
  filename         = "lambda_function.zip"
  function_name    = var.function_name
  role            = aws_iam_role.lambda_role.arn
  handler         = "lambda_function.lambda_handler"
  runtime         = "python3.9"
  memory_size     = var.memory_size
  timeout         = var.timeout
  
  # Source code hash for updates
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256
  
  # Environment variables
  environment {
    variables = var.environment_variables
  }
  
  # VPC configuration (if needed)
  vpc_config {
    subnet_ids         = var.subnet_ids
    security_group_ids = var.security_group_ids
  }
  
  # Dead letter queue (for error handling)
  dead_letter_config {
    target_arn = aws_sqs_queue.dlq.arn
  }
  
  tags = var.tags
}
```

**2. IAM Role and Policies:**
```hcl
# Lambda execution role
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

# Basic Lambda execution permissions
resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

# Custom policy for DynamoDB access
resource "aws_iam_role_policy" "dynamodb_access" {
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
        Resource = [
          "arn:aws:dynamodb:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:table/${var.dynamodb_table_name}",
          "arn:aws:dynamodb:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:table/${var.dynamodb_table_name}/index/*"
        ]
      }
    ]
  })
}
```

**3. Lambda Packaging:**
```hcl
# Package Lambda function code
data "archive_file" "lambda_zip" {
  type        = "zip"
  source_dir  = var.source_dir
  output_path = "${path.module}/${var.function_name}.zip"
  
  # Exclude unnecessary files
  excludes = [
    "__pycache__",
    "*.pyc",
    ".pytest_cache",
    "tests",
    ".git"
  ]
}
```

**4. CloudWatch Log Group:**
```hcl
resource "aws_cloudwatch_log_group" "lambda_logs" {
  name              = "/aws/lambda/${var.function_name}"
  retention_in_days = var.log_retention_days
  
  tags = var.tags
}
```

**5. Lambda Versioning and Aliases:**
```hcl
# Lambda version (immutable)
resource "aws_lambda_function" "main" {
  # ... function configuration
  publish = true  # Creates new version on each update
}

# Lambda alias for traffic management
resource "aws_lambda_alias" "live" {
  name             = "live"
  description      = "Live traffic alias"
  function_name    = aws_lambda_function.main.function_name
  function_version = aws_lambda_function.main.version
}

# Weighted alias for canary deployments
resource "aws_lambda_alias" "canary" {
  name             = "canary"
  description      = "Canary deployment alias"
  function_name    = aws_lambda_function.main.function_name
  function_version = aws_lambda_function.main.version
  
  routing_config {
    additional_version_weights = {
      (aws_lambda_function.main.version) = 0.1  # 10% traffic
    }
  }
}
```

## **Environment-Specific Configuration**

**Using .tfvars Files:**

**dev.tfvars:**
```hcl
# Development environment configuration
function_name    = "my-lambda-function-dev"
memory_size     = 256
timeout         = 60
log_level       = "DEBUG"

environment_variables = {
  ENVIRONMENT    = "development"
  LOG_LEVEL     = "DEBUG"
  DATABASE_URL  = "dev-database-endpoint"
  API_BASE_URL  = "https://api-dev.company.com"
}

tags = {
  Environment = "dev"
  Owner       = "development-team"
  Project     = "lambda-pipeline"
}
```

**qa.tfvars:**
```hcl
# QA environment configuration
function_name    = "my-lambda-function-qa"
memory_size     = 512
timeout         = 120
log_level       = "INFO"

environment_variables = {
  ENVIRONMENT    = "qa"
  LOG_LEVEL     = "INFO"
  DATABASE_URL  = "qa-database-endpoint"
  API_BASE_URL  = "https://api-qa.company.com"
}

tags = {
  Environment = "qa"
  Owner       = "qa-team"
  Project     = "lambda-pipeline"
}
```

**prod.tfvars:**
```hcl
# Production environment configuration
function_name    = "my-lambda-function-prod"
memory_size     = 1024
timeout         = 300
log_level       = "WARN"

environment_variables = {
  ENVIRONMENT    = "production"
  LOG_LEVEL     = "WARN"
  DATABASE_URL  = "prod-database-endpoint"
  API_BASE_URL  = "https://api.company.com"
}

tags = {
  Environment = "production"
  Owner       = "platform-team"
  Project     = "lambda-pipeline"
  CostCenter  = "engineering"
}
```

## **Terraform Commands for CI/CD Pipeline**

**Command Execution in Jenkins:**

**1. Initialization:**
```bash
cd infrastructure/dev
terraform init
```
What happens:
- Downloads AWS provider
- Configures S3 backend
- Creates .terraform/ directory
- Validates backend configuration

**2. Planning:**
```bash
terraform plan -var-file=dev.tfvars -out=tfplan
```
What happens:
- Reads current state from S3
- Compares with desired configuration
- Shows planned changes
- Saves plan to file for consistent apply

**3. Applying:**
```bash
terraform apply tfplan
```
What happens:
- Executes the saved plan
- Creates/updates/deletes resources
- Updates state file in S3
- Shows summary of changes

**4. Validation (optional):**
```bash
terraform validate
terraform fmt -check
```
What happens:
- Validates syntax and configuration
- Checks formatting consistency

**5. Cleanup (if needed):**
```bash
terraform destroy -var-file=dev.tfvars
```
What happens:
- Destroys all managed resources
- Updates state to reflect deletions

## **Terraform Best Practices for Lambda CI/CD**

**1. State Management:**
```hcl
# Always use remote state for team collaboration
terraform {
  backend "s3" {
    bucket         = "company-terraform-state-${var.environment}"
    key            = "lambda-functions/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock-${var.environment}"
    encrypt        = true
  }
}
```

**2. Resource Naming:**
```hcl
# Use consistent naming conventions
locals {
  name_prefix = "${var.project_name}-${var.environment}"
  
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "Terraform"
    CreatedBy   = "CI/CD Pipeline"
  }
}

resource "aws_lambda_function" "main" {
  function_name = "${local.name_prefix}-lambda"
  tags         = local.common_tags
}
```

**3. Variable Validation:**
```hcl
variable "memory_size" {
  description = "Lambda memory size"
  type        = number
  default     = 128
  
  validation {
    condition = var.memory_size >= 128 && var.memory_size <= 10240 && var.memory_size % 64 == 0
    error_message = "Memory size must be between 128-10240 MB and divisible by 64."
  }
}

variable "environment" {
  description = "Environment name"
  type        = string
  
  validation {
    condition     = contains(["dev", "qa", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, qa, staging, prod."
  }
}
```

**4. Data Sources for Dynamic Values:**
```hcl
# Get current AWS account ID
data "aws_caller_identity" "current" {}

# Get current AWS region
data "aws_region" "current" {}

# Get VPC information
data "aws_vpc" "main" {
  tags = {
    Name = "${var.environment}-vpc"
  }
}

# Use in resources
resource "aws_lambda_function" "main" {
  function_name = "${var.function_name}-${data.aws_region.current.name}"
  
  vpc_config {
    subnet_ids         = data.aws_subnets.private.ids
    security_group_ids = [data.aws_security_group.lambda.id]
  }
}
```

**5. Conditional Resources:**
```hcl
# Create API Gateway only if needed
resource "aws_api_gateway_rest_api" "main" {
  count = var.create_api_gateway ? 1 : 0
  name  = "${var.function_name}-api"
}

# Environment-specific configurations
resource "aws_lambda_function" "main" {
  memory_size = var.environment == "prod" ? 1024 : 256
  timeout     = var.environment == "prod" ? 300 : 60
  
  # Enable X-Ray tracing in production
  tracing_config {
    mode = var.environment == "prod" ? "Active" : "PassThrough"
  }
}
```

## **Troubleshooting Terraform Issues**

**Common Issues and Solutions:**

**1. State Lock Issues:**
```bash
Error: Error locking state: Error acquiring the state lock
```
Solution:
```bash
# Check who has the lock
aws dynamodb get-item --table-name terraform-state-lock-dev --key '{"LockID":{"S":"your-state-path"}}'

# Force unlock (use carefully!)
terraform force-unlock LOCK_ID
```

**2. State Drift:**
```bash
# Check for state drift
terraform plan -detailed-exitcode

# Refresh state to match reality
terraform apply -refresh-only
```

**3. Resource Dependencies:**
```hcl
# Explicit dependencies
resource "aws_lambda_function" "main" {
  # ...
  depends_on = [
    aws_iam_role_policy_attachment.lambda_basic,
    aws_cloudwatch_log_group.lambda_logs
  ]
}
```

**4. Provider Version Conflicts:**
```hcl
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"  # Allow 5.x but not 6.x
    }
  }
}
```

**5. Import Existing Resources:**
```bash
# Import existing Lambda function
terraform import aws_lambda_function.main my-existing-function

# Import existing IAM role
terraform import aws_iam_role.lambda_role my-existing-role
```

## **Terraform in Jenkins Pipeline Integration**

**Pipeline Stage Configuration:**
```groovy
stage('Deploy to Dev') {
    steps {
        withAWS(credentials: 'aws-dev-credentials', region: 'us-east-1') {
            sh '''
                cd infrastructure/dev
                
                # Initialize Terraform
                terraform init -backend-config="bucket=${TERRAFORM_STATE_BUCKET}"
                
                # Validate configuration
                terraform validate
                
                # Plan changes
                terraform plan -var-file=dev.tfvars -out=tfplan -detailed-exitcode
                
                # Apply if changes exist
                if [ $? -eq 2 ]; then
                    terraform apply tfplan
                else
                    echo "No changes to apply"
                fi
            '''
        }
    }
}
```

**Error Handling:**
```groovy
script {
    try {
        sh 'terraform apply tfplan'
    } catch (Exception e) {
        sh 'terraform show tfplan'  // Show what was planned
        throw e
    }
}
```

This comprehensive understanding of Terraform will enable you to effectively manage your Lambda infrastructure as code, automate deployments through Jenkins, and maintain consistent environments across your development lifecycle.
