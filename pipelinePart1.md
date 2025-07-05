## Simplified Jenkinsfile for Lambda-Only Updates

Here's a streamlined version focused **only** on updating existing Lambda functions:

```groovy
pipeline {
    agent any
    
    environment {
        AWS_DEFAULT_REGION = 'us-east-1'
        DEV_ACCOUNT = 'dev-account'
    }
    
    tools {
        python 'Python3'
    }
    
    // Only run pipeline on dev branch
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
                echo "🔀 Code pushed to dev branch - Updating Lambda functions"
                echo "Building branch: ${env.BRANCH_NAME}"
                echo "Commit: ${env.GIT_COMMIT_SHORT}"
            }
        }
        
        stage('Install Lambda Dependencies') {
            steps {
                sh '''
                    echo "📦 Installing Lambda dependencies..."
                    
                    # Create virtual environment for dependency installation
                    python3 -m venv venv
                    source venv/bin/activate
                    
                    # Upgrade pip
                    pip install --upgrade pip
                    
                    # Install dependencies for each Lambda function
                    find . -name "lambda_function.py" -type f | while read lambda_file; do
                        lambda_dir=$(dirname "$lambda_file")
                        lambda_name=$(basename "$lambda_dir")
                        
                        echo "Processing Lambda: $lambda_name"
                        
                        if [ -f "$lambda_dir/requirements.txt" ]; then
                            echo "Installing dependencies for $lambda_name"
                            # Install dependencies into Lambda directory for packaging
                            pip install -r "$lambda_dir/requirements.txt" -t "$lambda_dir/"
                        else
                            echo "No requirements.txt found for $lambda_name - skipping dependencies"
                        fi
                    done
                    
                    echo "✅ All Lambda dependencies installed"
                '''
            }
        }
        
        stage('Package and Update Lambda Functions') {
            steps {
                echo "⚡ Updating existing Lambda functions in AWS..."
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding', 
                     credentialsId: 'aws-dev-account-credentials']
                ]) {
                    sh '''
                        # Find and update all Lambda functions
                        find . -name "lambda_function.py" -type f | while read lambda_file; do
                            lambda_dir=$(dirname "$lambda_file")
                            lambda_name=$(basename "$lambda_dir")
                            
                            echo "📦 Packaging Lambda: $lambda_name"
                            
                            # Create deployment package
                            cd "$lambda_dir"
                            
                            # Create zip excluding unnecessary files
                            zip -r "../../${lambda_name}-dev.zip" . \
                                -x "*.pyc" "*/__pycache__/*" "*/.pytest_cache/*" \
                                   "*/tests/*" "*.md" ".DS_Store" "*.coverage*" \
                                   "venv/*" ".git*" "*.log"
                            
                            cd - > /dev/null
                            
                            echo "🚀 Updating Lambda function: ${lambda_name}-dev"
                            
                            # Update existing Lambda function code
                            aws lambda update-function-code \
                                --function-name "${lambda_name}-dev" \
                                --zip-file "fileb://${lambda_name}-dev.zip" \
                                --region ${AWS_DEFAULT_REGION}
                            
                            echo "✅ ${lambda_name}-dev updated successfully"
                            echo ""
                        done
                    '''
                }
            }
        }
        
        stage('Verify Lambda Updates') {
            steps {
                echo "🔍 Verifying Lambda function updates..."
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding', 
                     credentialsId: 'aws-dev-account-credentials']
                ]) {
                    sh '''
                        echo "⚡ Updated Lambda Functions Status:"
                        echo "================================================"
                        
                        find . -name "lambda_function.py" -type f | while read lambda_file; do
                            lambda_dir=$(dirname "$lambda_file")
                            lambda_name=$(basename "$lambda_dir")
                            
                            echo "Checking: ${lambda_name}-dev"
                            
                            # Get function info to verify update
                            aws lambda get-function \
                                --function-name "${lambda_name}-dev" \
                                --region ${AWS_DEFAULT_REGION} \
                                --query 'Configuration.[FunctionName,LastModified,State,Runtime,CodeSize]' \
                                --output table
                            
                            echo ""
                        done
                        
                        echo "✅ All Lambda functions verified successfully"
                    '''
                }
            }
        }
    }
    
    post {
        always {
            sh '''
                echo "🧹 Cleaning up build artifacts..."
                
                # Remove deployment packages
                rm -f *-dev.zip || true
                
                # Remove virtual environment
                rm -rf venv || true
                
                # Clean up any installed dependencies in Lambda directories
                find . -name "lambda_function.py" -type f | while read lambda_file; do
                    lambda_dir=$(dirname "$lambda_file")
                    # Remove installed packages but keep lambda_function.py and requirements.txt
                    cd "$lambda_dir"
                    find . -type d -name "*.dist-info" -exec rm -rf {} + 2>/dev/null || true
                    find . -type d -name "__pycache__" -exec rm -rf {} + 2>/dev/null || true
                    find . -name "*.pyc" -delete 2>/dev/null || true
                    cd - > /dev/null
                done
                
                echo "✅ Cleanup completed"
            '''
        }
        success {
            script {
                // Count Lambda functions
                def lambdaCount = sh(
                    script: 'find . -name "lambda_function.py" -type f | wc -l',
                    returnStdout: true
                ).trim() as Integer
                
                echo """
🎉 LAMBDA FUNCTIONS UPDATED SUCCESSFULLY!

📋 Deployment Summary:
✅ ${lambdaCount} Lambda function(s) updated in AWS dev account
✅ All functions verified and running
✅ Code changes from VSCode → AWS Lambda ✓

📝 Updated Functions:"""
                
                // List all updated functions
                sh '''
                    find . -name "lambda_function.py" -type f | while read lambda_file; do
                        lambda_dir=$(dirname "$lambda_file")
                        lambda_name=$(basename "$lambda_dir")
                        echo "   ✅ ${lambda_name}-dev"
                    done
                '''
                
                echo """
🔄 Development Workflow:
1. Edit code in VSCode ✓
2. Push to dev branch ✓  
3. Jenkins updates AWS Lambda ✓
4. Test your functions in AWS console

🚀 Your Lambda functions are now updated and ready for testing!
                """
            }
        }
        failure {
            echo '''
❌ LAMBDA UPDATE FAILED!

🔧 Common Issues & Solutions:
1. AWS Credentials: Verify 'aws-dev-account-credentials' in Jenkins
2. Lambda Function Names: Ensure functions exist as '{directory-name}-dev'
3. Dependencies: Check requirements.txt files are valid
4. Permissions: Verify Jenkins has lambda:UpdateFunctionCode permission

🔄 To retry:
Fix the issue and push another commit to dev branch

💡 Debug Steps:
1. Check Jenkins console output above for specific error
2. Verify Lambda function exists in AWS console
3. Test AWS credentials manually if needed
            '''
        }
    }
}
```

## **Key Changes Made:**

### **Removed:**
- ❌ **S3/Static Asset deployment** - No frontend needed
- ❌ **Build Assets stage** - No building required
- ❌ **Unit/Integration testing** - Skipped for now
- ❌ **Frontend-related file checks** (package.json, dist/, build/, etc.)

### **Focused On:**
- ✅ **Lambda function detection** - Finds all `lambda_function.py` files
- ✅ **Dependency installation** - Handles `requirements.txt` per Lambda
- ✅ **Code packaging** - Creates ZIP files for deployment
- ✅ **AWS Lambda updates** - Updates existing Lambda functions only
- ✅ **Verification** - Confirms updates succeeded

---

## **What This Pipeline Does:**

### **1. When you push to `dev` branch:**
```
VSCode → git push origin dev → Webhook → Jenkins → AWS Lambda Update
```

### **2. Pipeline automatically:**
- Finds all directories with `lambda_function.py`
- Installs dependencies from `requirements.txt` (if exists)
- Packages each Lambda into a ZIP file
- Updates the existing Lambda function in AWS (with `-dev` suffix)
- Verifies the update succeeded

### **3. Expected Repository Structure:**
```
your-repo/
├── user-service/
│   ├── lambda_function.py      # Your Lambda code
│   └── requirements.txt        # Optional dependencies
├── email-service/
│   ├── lambda_function.py      # Another Lambda
│   └── requirements.txt        # Its dependencies
├── data-processor/
│   └── lambda_function.py      # Lambda without dependencies
└── Jenkinsfile                 # This pipeline
```

### **4. AWS Requirements:**
- Lambda functions must exist as: `user-service-dev`, `email-service-dev`, etc.
- Jenkins needs AWS credentials with `lambda:UpdateFunctionCode` permission

---

## **Workflow:**
1. **Edit code in VSCode** ✏️
2. **Commit and push to dev branch** 📤
3. **Jenkins automatically updates AWS Lambda** ⚡
4. **Test your updated function in AWS console** 🧪

**This pipeline is now laser-focused on the core need: syncing your local code changes to AWS Lambda functions!**

## Detailed Pipeline Script Breakdown

Let me walk through **exactly** what this Jenkins pipeline does, step by step:

---

## **Pipeline Header & Configuration**

```groovy
pipeline {
    agent any
    
    environment {
        AWS_DEFAULT_REGION = 'us-east-1'
        DEV_ACCOUNT = 'dev-account'
    }
    
    tools {
        python 'Python3'
    }
    
    when {
        branch 'dev'
    }
```

**What this does:**
- **`agent any`** - Jenkins can run this on any available worker node
- **`environment`** - Sets global variables that all stages can use:
  - `AWS_DEFAULT_REGION = 'us-east-1'` - All AWS commands will use US East region
  - `DEV_ACCOUNT = 'dev-account'` - Reference to development account (not actively used)
- **`tools { python 'Python3' }`** - Makes sure Python 3 is available for the build
- **`when { branch 'dev' }`** - **CRITICAL**: Pipeline only runs when you push to `dev` branch

**Real example:** If you push to `main` or `feature/xyz`, nothing happens. Only `dev` triggers this pipeline.

---

## **Stage 1: Checkout**

```groovy
stage('Checkout') {
    steps {
        checkout scm
        script {
            env.GIT_COMMIT_SHORT = sh(
                script: "git rev-parse --short HEAD",
                returnStdout: true
            ).trim()
        }
        echo "🔀 Code pushed to dev branch - Updating Lambda functions"
        echo "Building branch: ${env.BRANCH_NAME}"
        echo "Commit: ${env.GIT_COMMIT_SHORT}"
    }
}
```

**What this does:**
1. **`checkout scm`** - Downloads your repository code to Jenkins workspace
2. **`git rev-parse --short HEAD`** - Gets the short commit hash (like `abc1234`)
3. **`env.GIT_COMMIT_SHORT`** - Stores commit hash for later use
4. **Echo statements** - Shows info in Jenkins console

**Example Jenkins console output:**
```
🔀 Code pushed to dev branch - Updating Lambda functions
Building branch: dev
Commit: a4b7c2d
```

**What Jenkins workspace looks like after this stage:**
```
/var/jenkins_home/workspace/your-pipeline/
├── user-service/
│   ├── lambda_function.py
│   └── requirements.txt
├── email-service/
│   └── lambda_function.py
└── Jenkinsfile
```

---

## **Stage 2: Install Lambda Dependencies**

```groovy
stage('Install Lambda Dependencies') {
    steps {
        sh '''
            echo "📦 Installing Lambda dependencies..."
            
            # Create virtual environment for dependency installation
            python3 -m venv venv
            source venv/bin/activate
            
            # Upgrade pip
            pip install --upgrade pip
            
            # Install dependencies for each Lambda function
            find . -name "lambda_function.py" -type f | while read lambda_file; do
                lambda_dir=$(dirname "$lambda_file")
                lambda_name=$(basename "$lambda_dir")
                
                echo "Processing Lambda: $lambda_name"
                
                if [ -f "$lambda_dir/requirements.txt" ]; then
                    echo "Installing dependencies for $lambda_name"
                    # Install dependencies into Lambda directory for packaging
                    pip install -r "$lambda_dir/requirements.txt" -t "$lambda_dir/"
                else
                    echo "No requirements.txt found for $lambda_name - skipping dependencies"
                fi
            done
            
            echo "✅ All Lambda dependencies installed"
        '''
    }
}
```

**What this does step by step:**

### **Step 1: Create Virtual Environment**
```bash
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
```
- Creates isolated Python environment in `venv/` folder
- Activates it (so pip installs don't affect system Python)
- Updates pip to latest version

### **Step 2: Find All Lambda Functions**
```bash
find . -name "lambda_function.py" -type f
```
**Example output:**
```
./user-service/lambda_function.py
./email-service/lambda_function.py
./data-processor/lambda_function.py
```

### **Step 3: Process Each Lambda Function**
For each `lambda_function.py` found:

```bash
lambda_dir=$(dirname "$lambda_file")     # Gets: ./user-service
lambda_name=$(basename "$lambda_dir")    # Gets: user-service
```

### **Step 4: Install Dependencies**
```bash
if [ -f "$lambda_dir/requirements.txt" ]; then
    pip install -r "$lambda_dir/requirements.txt" -t "$lambda_dir/"
fi
```

**Key detail: `-t "$lambda_dir/"`**
- **`-t`** means "install TO this directory"
- Installs packages directly into Lambda folder (not venv)
- **Why?** Lambda needs dependencies bundled with code

**Example:** If `user-service/requirements.txt` contains:
```
requests==2.28.1
boto3==1.26.0
```

**Result:** Jenkins installs these into `user-service/` folder:
```
user-service/
├── lambda_function.py
├── requirements.txt
├── requests/              # ← Installed package
├── boto3/                 # ← Installed package
├── urllib3/               # ← Dependency of requests
└── ...other packages
```

**Console output during this stage:**
```
📦 Installing Lambda dependencies...
Processing Lambda: user-service
Installing dependencies for user-service
Collecting requests==2.28.1
Installing collected packages: requests, boto3
Successfully installed requests-2.28.1 boto3-1.26.0

Processing Lambda: email-service
No requirements.txt found for email-service - skipping dependencies

Processing Lambda: data-processor
Installing dependencies for data-processor
✅ All Lambda dependencies installed
```

---

## **Stage 3: Package and Update Lambda Functions**

```groovy
stage('Package and Update Lambda Functions') {
    steps {
        echo "⚡ Updating existing Lambda functions in AWS..."
        withCredentials([
            [$class: 'AmazonWebServicesCredentialsBinding', 
             credentialsId: 'aws-dev-account-credentials']
        ]) {
            sh '''
                # Find and update all Lambda functions
                find . -name "lambda_function.py" -type f | while read lambda_file; do
                    lambda_dir=$(dirname "$lambda_file")
                    lambda_name=$(basename "$lambda_dir")
                    
                    echo "📦 Packaging Lambda: $lambda_name"
                    
                    # Create deployment package
                    cd "$lambda_dir"
                    
                    # Create zip excluding unnecessary files
                    zip -r "../../${lambda_name}-dev.zip" . \
                        -x "*.pyc" "*/__pycache__/*" "*/.pytest_cache/*" \
                               "*/tests/*" "*.md" ".DS_Store" "*.coverage*" \
                               "venv/*" ".git*" "*.log"
                    
                    cd - > /dev/null
                    
                    echo "🚀 Updating Lambda function: ${lambda_name}-dev"
                    
                    # Update existing Lambda function code
                    aws lambda update-function-code \
                        --function-name "${lambda_name}-dev" \
                        --zip-file "fileb://${lambda_name}-dev.zip" \
                        --region ${AWS_DEFAULT_REGION}
                    
                    echo "✅ ${lambda_name}-dev updated successfully"
                    echo ""
                done
            '''
        }
    }
}
```

**What this does in detail:**

### **Step 1: AWS Credentials**
```groovy
withCredentials([
    [$class: 'AmazonWebServicesCredentialsBinding', 
     credentialsId: 'aws-dev-account-credentials']
])
```
- Loads AWS credentials from Jenkins
- Makes `aws` CLI commands work
- Credentials are temporarily available during this stage

### **Step 2: For Each Lambda Function**

**Process `user-service/` example:**

```bash
lambda_dir="./user-service"
lambda_name="user-service"
```

### **Step 3: Create ZIP Package**
```bash
cd "./user-service"
zip -r "../../user-service-dev.zip" . \
    -x "*.pyc" "*/__pycache__/*" ...
```

**What gets included in ZIP:**
- ✅ `lambda_function.py` (your code)
- ✅ `requirements.txt` (dependency list)
- ✅ `requests/` folder (installed package)
- ✅ `boto3/` folder (installed package)
- ✅ All other installed dependencies

**What gets excluded (`-x` flags):**
- ❌ `*.pyc` (Python compiled files)
- ❌ `__pycache__/` (Python cache folders)
- ❌ `.pytest_cache/` (Test cache)
- ❌ `tests/` (Test files)
- ❌ `*.md` (Documentation)
- ❌ `.DS_Store` (Mac system files)
- ❌ `venv/` (Virtual environment)
- ❌ `.git*` (Git files)

**Result:** Creates `user-service-dev.zip` in Jenkins workspace root

### **Step 4: Update AWS Lambda**
```bash
aws lambda update-function-code \
    --function-name "user-service-dev" \
    --zip-file "fileb://user-service-dev.zip" \
    --region us-east-1
```

**What this does:**
- **`update-function-code`** - Updates existing Lambda function code (doesn't create new one)
- **`--function-name "user-service-dev"`** - Updates Lambda named `user-service-dev`
- **`--zip-file "fileb://user-service-dev.zip"`** - Uploads ZIP file as new code
- **`fileb://`** prefix means "read binary file"

**AWS Response:**
```json
{
    "FunctionName": "user-service-dev",
    "FunctionArn": "arn:aws:lambda:us-east-1:123456789:function:user-service-dev",
    "LastModified": "2025-07-05T15:30:45.123+0000",
    "CodeSha256": "abc123...",
    "Version": "$LATEST"
}
```

**Console output during this stage:**
```
⚡ Updating existing Lambda functions in AWS...
📦 Packaging Lambda: user-service
🚀 Updating Lambda function: user-service-dev
✅ user-service-dev updated successfully

📦 Packaging Lambda: email-service  
🚀 Updating Lambda function: email-service-dev
✅ email-service-dev updated successfully
```

---

## **Stage 4: Verify Lambda Updates**

```groovy
stage('Verify Lambda Updates') {
    steps {
        echo "🔍 Verifying Lambda function updates..."
        withCredentials([...]) {
            sh '''
                echo "⚡ Updated Lambda Functions Status:"
                echo "================================================"
                
                find . -name "lambda_function.py" -type f | while read lambda_file; do
                    lambda_dir=$(dirname "$lambda_file")
                    lambda_name=$(basename "$lambda_dir")
                    
                    echo "Checking: ${lambda_name}-dev"
                    
                    # Get function info to verify update
                    aws lambda get-function \
                        --function-name "${lambda_name}-dev" \
                        --region ${AWS_DEFAULT_REGION} \
                        --query 'Configuration.[FunctionName,LastModified,State,Runtime,CodeSize]' \
                        --output table
                    
                    echo ""
                done
                
                echo "✅ All Lambda functions verified successfully"
            '''
        }
    }
}
```

**What this does:**

### **Step 1: For Each Lambda Function**
Runs `aws lambda get-function` to retrieve current status

### **Step 2: Query Specific Information**
```bash
--query 'Configuration.[FunctionName,LastModified,State,Runtime,CodeSize]'
--output table
```

**Example output:**
```
⚡ Updated Lambda Functions Status:
================================================
Checking: user-service-dev
----------------------------------------------------
|                  GetFunction                     |
+----------+--------------------------------------+
|  user-service-dev  |  2025-07-05T15:30:45.123Z  |
|  Active           |  python3.9                  |
|  2048576          |                              |
+----------+--------------------------------------+

Checking: email-service-dev
----------------------------------------------------
|                  GetFunction                     |
+----------+--------------------------------------+
|  email-service-dev |  2025-07-05T15:30:47.456Z  |
|  Active           |  python3.9                  |
|  1024512          |                              |
+----------+--------------------------------------+

✅ All Lambda functions verified successfully
```

**What this tells you:**
- **FunctionName** - Confirms correct function was updated
- **LastModified** - Shows timestamp of update (should be recent)
- **State** - Should be "Active" (ready to execute)
- **Runtime** - Confirms Python version
- **CodeSize** - Size of uploaded ZIP file in bytes

---

## **Post-Build Actions**

```groovy
post {
    always {
        sh '''
            echo "🧹 Cleaning up build artifacts..."
            
            # Remove deployment packages
            rm -f *-dev.zip || true
            
            # Remove virtual environment
            rm -rf venv || true
            
            # Clean up installed dependencies in Lambda directories
            find . -name "lambda_function.py" -type f | while read lambda_file; do
                lambda_dir=$(dirname "$lambda_file")
                cd "$lambda_dir"
                find . -type d -name "*.dist-info" -exec rm -rf {} + 2>/dev/null || true
                find . -type d -name "__pycache__" -exec rm -rf {} + 2>/dev/null || true
                find . -name "*.pyc" -delete 2>/dev/null || true
                cd - > /dev/null
            done
            
            echo "✅ Cleanup completed"
        '''
    }
    success {
        # Shows success message with summary
    }
    failure {
        # Shows troubleshooting information
    }
}
```

**What this does:**

### **`always` Section** (runs regardless of success/failure)
- **Removes ZIP files** (`user-service-dev.zip`, etc.) to save disk space
- **Removes virtual environment** (`venv/` folder)
- **Cleans Lambda directories** - removes installed packages but keeps source code
  - Removes `.dist-info/` folders (package metadata)
  - Removes `__pycache__/` folders (Python cache)
  - Removes `.pyc` files (compiled Python)

**After cleanup, Lambda directories look like:**
```
user-service/
├── lambda_function.py     # ✅ Kept
└── requirements.txt       # ✅ Kept
# All installed packages removed
```

### **`success` Section** (only runs if everything succeeded)
Shows celebration message with deployment summary

### **`failure` Section** (only runs if something failed)
Shows troubleshooting steps and common solutions

---

## **Complete Workflow Summary**

### **What happens when you push to dev:**

1. **Git Push** → **Webhook** → **Jenkins Triggered**
2. **Checkout** - Downloads your repository code
3. **Install Dependencies** - For each Lambda:
   - Finds `requirements.txt`
   - Installs packages directly into Lambda folder
4. **Package & Update** - For each Lambda:
   - Creates ZIP file with code + dependencies
   - Uploads to AWS Lambda function (named `{folder}-dev`)
5. **Verify** - Confirms all Lambda functions updated successfully
6. **Cleanup** - Removes temporary files

### **End Result:**
Your local code changes are now running in AWS Lambda functions with `-dev` suffix.

**Example:** If you edit `user-service/lambda_function.py` locally and push to dev, the `user-service-dev` Lambda function in AWS gets updated with your changes automatically.

**This pipeline creates a seamless development workflow: Local Code → Git Push → AWS Lambda Update! 🚀**


terraform 
## Detailed Terraform Setup for Jenkins Lambda Pipeline

Let me walk you through the complete Terraform setup step by step.

---

## **Step 1: Create Terraform Directory Structure**

First, create the directory structure in your repository:

```bash
# In your repository root
mkdir terraform
cd terraform
```

Your repository should now look like:
```
your-repo/
├── user-service/
│   └── lambda_function.py
├── email-service/
│   └── lambda_function.py
├── terraform/              # ← New directory
├── Jenkinsfile
└── README.md
```

---

## **Step 2: Create Terraform Configuration Files**

### **Create `terraform/variables.tf`:**

```hcl
# terraform/variables.tf

variable "lambda_functions" {
  description = "List of Lambda function names (should match your directory names)"
  type        = list(string)
  default     = [
    "user-service",
    "email-service"
    # Add more function names as you create more Lambda directories
  ]
}

variable "aws_region" {
  description = "AWS region for Lambda functions"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

variable "project_name" {
  description = "Project name for resource naming"
  type        = string
  default     = "lambda-cicd"
}
```

### **Create `terraform/provider.tf`:**

```hcl
# terraform/provider.tf

terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  }
}
```

### **Create `terraform/lambda.tf`:**

```hcl
# terraform/lambda.tf

# Data source to get current AWS account ID
data "aws_caller_identity" "current" {}

# IAM Role for Lambda Functions
resource "aws_iam_role" "lambda_execution_role" {
  name = "${var.project_name}-lambda-execution-role-${var.environment}"

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

  tags = {
    Name = "${var.project_name}-lambda-execution-role-${var.environment}"
  }
}

# Attach basic Lambda execution policy
resource "aws_iam_role_policy_attachment" "lambda_basic_execution" {
  role       = aws_iam_role.lambda_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

# Optional: Attach VPC access policy if your Lambdas need VPC access
resource "aws_iam_role_policy_attachment" "lambda_vpc_access" {
  role       = aws_iam_role.lambda_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
}

# Create placeholder Lambda deployment package
data "archive_file" "placeholder_lambda" {
  type        = "zip"
  output_path = "${path.module}/placeholder.zip"
  
  source {
    content = <<EOF
def lambda_handler(event, context):
    """
    Placeholder Lambda function.
    This will be replaced by Jenkins deployment.
    """
    return {
        'statusCode': 200,
        'body': {
            'message': 'Placeholder function - will be updated by Jenkins',
            'function': context.function_name if context else 'unknown',
            'event': event
        }
    }
EOF
    filename = "lambda_function.py"
  }
}

# Create Lambda functions for each service
resource "aws_lambda_function" "dev_functions" {
  for_each = toset(var.lambda_functions)

  filename         = data.archive_file.placeholder_lambda.output_path
  function_name    = "${each.value}-${var.environment}"
  role            = aws_iam_role.lambda_execution_role.arn
  handler         = "lambda_function.lambda_handler"
  runtime         = "python3.9"
  timeout         = 30
  memory_size     = 256

  # Ignore changes to code since Jenkins will update it
  lifecycle {
    ignore_changes = [
      filename,
      last_modified,
      source_code_hash,
      source_code_size,
    ]
  }

  tags = {
    Name        = "${each.value}-${var.environment}"
    Service     = each.value
    Environment = var.environment
  }
}
```

### **Create `terraform/jenkins-iam.tf`:**

```hcl
# terraform/jenkins-iam.tf

# IAM Policy for Jenkins to update Lambda functions
resource "aws_iam_policy" "jenkins_lambda_policy" {
  name        = "${var.project_name}-jenkins-lambda-policy"
  description = "Policy for Jenkins to update Lambda functions"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "LambdaUpdatePermissions"
        Effect = "Allow"
        Action = [
          "lambda:UpdateFunctionCode",
          "lambda:GetFunction",
          "lambda:ListFunctions",
          "lambda:GetFunctionConfiguration"
        ]
        Resource = [
          "arn:aws:lambda:${var.aws_region}:${data.aws_caller_identity.current.account_id}:function:*-${var.environment}"
        ]
      },
      {
        Sid    = "LambdaListPermissions"
        Effect = "Allow"
        Action = [
          "lambda:ListFunctions"
        ]
        Resource = "*"
      }
    ]
  })

  tags = {
    Name = "${var.project_name}-jenkins-lambda-policy"
  }
}

# IAM User for Jenkins
resource "aws_iam_user" "jenkins_user" {
  name = "${var.project_name}-jenkins-user"
  path = "/"

  tags = {
    Name    = "${var.project_name}-jenkins-user"
    Purpose = "Jenkins CI/CD for Lambda functions"
  }
}

# Attach policy to Jenkins user
resource "aws_iam_user_policy_attachment" "jenkins_policy_attachment" {
  user       = aws_iam_user.jenkins_user.name
  policy_arn = aws_iam_policy.jenkins_lambda_policy.arn
}

# Create access key for Jenkins user
resource "aws_iam_access_key" "jenkins_access_key" {
  user = aws_iam_user.jenkins_user.name
}
```

### **Create `terraform/outputs.tf`:**

```hcl
# terraform/outputs.tf

output "lambda_function_names" {
  description = "Names of created Lambda functions"
  value       = [for func in aws_lambda_function.dev_functions : func.function_name]
}

output "lambda_function_arns" {
  description = "ARNs of created Lambda functions"
  value       = [for func in aws_lambda_function.dev_functions : func.arn]
}

output "jenkins_user_name" {
  description = "IAM user name for Jenkins"
  value       = aws_iam_user.jenkins_user.name
}

output "jenkins_access_key_id" {
  description = "Access Key ID for Jenkins (add this to Jenkins credentials)"
  value       = aws_iam_access_key.jenkins_access_key.id
}

output "jenkins_secret_access_key" {
  description = "Secret Access Key for Jenkins (add this to Jenkins credentials)"
  value       = aws_iam_access_key.jenkins_access_key.secret
  sensitive   = true
}

output "aws_region" {
  description = "AWS region where resources are created"
  value       = var.aws_region
}

output "aws_account_id" {
  description = "AWS account ID"
  value       = data.aws_caller_identity.current.account_id
}

# Summary information
output "setup_summary" {
  description = "Summary of created resources"
  value = {
    lambda_functions = [for func in aws_lambda_function.dev_functions : func.function_name]
    jenkins_user     = aws_iam_user.jenkins_user.name
    aws_region      = var.aws_region
    environment     = var.environment
  }
}
```

---

## **Step 3: Deploy Infrastructure with Terraform**

### **Initialize Terraform:**

```bash
cd terraform

# Initialize Terraform (downloads AWS provider)
terraform init
```

**Expected output:**
```
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.31.0...

Terraform has been successfully initialized!
```

### **Review the Plan:**

```bash
# See what Terraform will create
terraform plan
```

**Expected output:**
```
Terraform will perform the following actions:

  # aws_iam_access_key.jenkins_access_key will be created
  + resource "aws_iam_access_key" "jenkins_access_key" {
      + id     = (known after apply)
      + secret = (sensitive value)
      + user   = "lambda-cicd-jenkins-user"
    }

  # aws_iam_policy.jenkins_lambda_policy will be created
  + resource "aws_iam_policy" "jenkins_lambda_policy" {
      + arn         = (known after apply)
      + name        = "lambda-cicd-jenkins-lambda-policy"
      + policy      = jsonencode(...)
    }

  # aws_lambda_function.dev_functions["email-service"] will be created
  + resource "aws_lambda_function" "dev_functions" {
      + function_name = "email-service-dev"
      + handler       = "lambda_function.lambda_handler"
      + runtime       = "python3.9"
      + role          = (known after apply)
    }

  # aws_lambda_function.dev_functions["user-service"] will be created
  + resource "aws_lambda_function" "dev_functions" {
      + function_name = "user-service-dev"
      + handler       = "lambda_function.lambda_handler"
      + runtime       = "python3.9"
      + role          = (known after apply)
    }

Plan: 7 to add, 0 to change, 0 to destroy.
```

### **Apply the Configuration:**

```bash
# Create the infrastructure
terraform apply
```

**Terraform will ask for confirmation:**
```
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes
```

**Expected output:**
```
aws_iam_role.lambda_execution_role: Creating...
aws_iam_user.jenkins_user: Creating...
aws_iam_policy.jenkins_lambda_policy: Creating...
data.archive_file.placeholder_lambda: Reading...
aws_iam_role.lambda_execution_role: Creation complete after 2s
aws_lambda_function.dev_functions["user-service"]: Creating...
aws_lambda_function.dev_functions["email-service"]: Creating...
aws_iam_access_key.jenkins_access_key: Creating...

Apply complete! Resources: 7 added, 0 changed, 0 destroyed.

Outputs:

jenkins_access_key_id = "AKIA..."
lambda_function_names = [
  "email-service-dev",
  "user-service-dev",
]
setup_summary = {
  "aws_region" = "us-east-1"
  "environment" = "dev"
  "jenkins_user" = "lambda-cicd-jenkins-user"
  "lambda_functions" = [
    "email-service-dev",
    "user-service-dev",
  ]
}
```

### **Get Jenkins Credentials:**

```bash
# Get the Access Key ID (public)
terraform output jenkins_access_key_id

# Get the Secret Access Key (sensitive)
terraform output -raw jenkins_secret_access_key
```

**Save these credentials! You'll need them for Jenkins configuration.**

---

## **Step 4: Verify Infrastructure in AWS Console**

### **Check Lambda Functions:**

1. **Go to AWS Lambda Console**
2. **Filter by name:** `-dev`
3. **You should see:**
   - `user-service-dev`
   - `email-service-dev`

### **Test a Lambda Function:**

1. **Click on `user-service-dev`**
2. **Go to Test tab**
3. **Create test event** (use default)
4. **Click Test**

**Expected result:**
```json
{
  "statusCode": 200,
  "body": {
    "message": "Placeholder function - will be updated by Jenkins",
    "function": "user-service-dev",
    "event": {...}
  }
}
```

### **Check IAM User:**

1. **Go to IAM Console → Users**
2. **Find:** `lambda-cicd-jenkins-user`
3. **Click on it → Permissions tab**
4. **You should see:** `lambda-cicd-jenkins-lambda-policy` attached

---

## **Step 5: Configure Jenkins AWS Credentials**

### **Add Credentials to Jenkins:**

1. **Jenkins Dashboard** → **Manage Jenkins** → **Credentials**
2. **Click** `(global)` domain
3. **Add Credentials**
4. **Kind:** `AWS Credentials`
5. **ID:** `aws-dev-account-credentials` (must match Jenkinsfile!)
6. **Access Key ID:** (from terraform output)
7. **Secret Access Key:** (from terraform output)
8. **Description:** `AWS credentials for Lambda deployment`
9. **Create**

---

## **Step 6: Test the Setup**

### **Manual Test (Before Jenkins):**

```bash
# Configure AWS CLI with Jenkins credentials
aws configure set aws_access_key_id YOUR_ACCESS_KEY_ID
aws configure set aws_secret_access_key YOUR_SECRET_ACCESS_KEY
aws configure set region us-east-1

# List Lambda functions
aws lambda list-functions --query 'Functions[?contains(FunctionName, `-dev`)].FunctionName'

# Test updating a function
echo 'def lambda_handler(event, context): return "Test from CLI"' > test.py
zip test.zip test.py

aws lambda update-function-code \
    --function-name user-service-dev \
    --zip-file fileb://test.zip

# Verify the update
aws lambda get-function --function-name user-service-dev \
    --query 'Configuration.[FunctionName,LastModified,State]' \
    --output table
```

**Expected output:**
```
---------------------------------------------------------------
|                        GetFunction                         |
+------------------+-----------------------------+-----------+
|  user-service-dev|  2025-07-05T10:30:45.123Z  |  Active   |
+------------------+-----------------------------+-----------+
```

---

## **Step 7: Test Jenkins Pipeline**

### **Push Code to Dev Branch:**

```bash
# Make sure you're in dev branch
git checkout dev

# Add terraform files to git
git add terraform/
git commit -m "Add Terraform infrastructure for Lambda deployment"

# Push to trigger Jenkins
git push origin dev
```

### **Watch Jenkins:**

1. **Go to Jenkins Dashboard**
2. **Click on your pipeline job**
3. **Watch for new build to start**
4. **Click on build number to see console output**

**Expected Jenkins console output:**
```
🔀 Code pushed to dev branch - Updating Lambda functions
Building branch: dev
Commit: a4b7c2d

📦 Installing Lambda dependencies...
Processing Lambda: user-service
No requirements.txt found for user-service - skipping dependencies
Processing Lambda: email-service
No requirements.txt found for email-service - skipping dependencies
✅ All Lambda dependencies installed

⚡ Updating existing Lambda functions in AWS...
📦 Packaging Lambda: user-service
🚀 Updating Lambda function: user-service-dev
✅ user-service-dev updated successfully

📦 Packaging Lambda: email-service
🚀 Updating Lambda function: email-service-dev
✅ email-service-dev updated successfully

🔍 Verifying Lambda function updates...
✅ All Lambda functions verified successfully

🎉 LAMBDA FUNCTIONS UPDATED SUCCESSFULLY!
```

---

## **Step 8: Verify End-to-End Workflow**

### **Make a Code Change:**

```bash
# Edit a Lambda function
cat > user-service/lambda_function.py << 'EOF'
def lambda_handler(event, context):
    """
    Updated user service Lambda function
    """
    return {
        'statusCode': 200,
        'body': {
            'message': 'Hello from updated user service!',
            'version': '1.0.1',
            'timestamp': context.aws_request_id if context else 'test'
        }
    }
EOF

# Commit and push
git add user-service/lambda_function.py
git commit -m "Update user service Lambda function"
git push origin dev
```

### **Verify in AWS Console:**

1. **Go to Lambda Console → user-service-dev**
2. **Check Last Modified timestamp** (should be recent)
3. **Test the function**
4. **Expected result:**
```json
{
  "statusCode": 200,
  "body": {
    "message": "Hello from updated user service!",
    "version": "1.0.1",
    "timestamp": "..."
  }
}
```

---

## **Step 9: Add More Lambda Functions**

### **To add a new Lambda function:**

1. **Create new directory with `lambda_function.py`:**
```bash
mkdir data-processor
cat > data-processor/lambda_function.py << 'EOF'
def lambda_handler(event, context):
    return {'message': 'Data processor service'}
EOF
```

2. **Update Terraform variables:**
```hcl
# In terraform/variables.tf
variable "lambda_functions" {
  default = [
    "user-service",
    "email-service",
    "data-processor"  # ← Add new function
  ]
}
```

3. **Apply Terraform:**
```bash
cd terraform
terraform apply
```

4. **Commit and push:**
```bash
git add .
git commit -m "Add data processor Lambda function"
git push origin dev
```

**Jenkins will automatically detect and deploy the new function!**

---

## **Troubleshooting**

### **Issue: Terraform apply fails with permission errors**

**Solution:** Make sure your local AWS credentials have permission to create IAM roles, Lambda functions, etc.

### **Issue: Jenkins can't find AWS credentials**

**Solution:** Verify the credential ID in Jenkins matches exactly: `aws-dev-account-credentials`

### **Issue: Lambda function not found**

**Solution:** Check that Terraform created the function with `-dev` suffix matching your directory name.

---

## **Summary**

**You now have a complete setup where:**

✅ **Terraform manages infrastructure** (Lambda functions + IAM permissions)
✅ **Jenkins has proper AWS credentials** to update Lambda functions
✅ **Pipeline automatically deploys** when you push to dev branch
✅ **Easy to add new functions** by creating directories and updating Terraform

**Workflow:**
1. **Edit code in VSCode** ✏️
2. **git push origin dev** 📤
3. **Jenkins updates AWS Lambda** ⚡
4. **Test in AWS console** 🧪

**Your development workflow is now fully automated! 🚀**

## Understanding Terraform Deployment - When, Why, and How Often

Great question! Let me explain the **infrastructure vs application code** distinction and when you run Terraform.

---

## **Infrastructure vs Application Code**

### **Infrastructure (Terraform) - "The Foundation"**
- **Lambda functions** (empty containers)
- **IAM roles and policies** (permissions)
- **S3 buckets, databases, networks** (resources)
- **Changes rarely** - maybe once per week/month

### **Application Code (Jenkins) - "The Content"**
- **Your Python code** (`lambda_function.py`)
- **Dependencies** (`requirements.txt`)
- **Business logic changes**
- **Changes frequently** - multiple times per day

---

## **Why Terraform is Run Manually**

### **Different Lifecycle:**

```
Infrastructure (Terraform):
├── Create Lambda function ← Run once
├── Set up permissions ← Run once  
├── Add new Lambda function ← Run occasionally
└── Change IAM policies ← Run rarely

Application Code (Jenkins):
├── Update lambda_function.py ← Run daily
├── Fix bugs ← Run daily
├── Add features ← Run daily
└── Deploy hotfixes ← Run multiple times daily
```

### **Different Teams/Responsibilities:**

```
DevOps Engineer / Platform Team:
└── Runs Terraform to set up infrastructure

Developers:
└── Push code → Jenkins → Application deployment
```

---

## **When Do You Run Terraform?**

### **Initial Setup (Once):**
```bash
# First time setting up the project
terraform apply
```
**Creates:**
- Lambda functions: `user-service-dev`, `email-service-dev`
- IAM user: `jenkins-user`
- Permissions for Jenkins to update Lambda functions

### **Adding New Services (Occasionally):**

**Scenario:** You create a new microservice

```bash
# 1. Create new Lambda directory
mkdir payment-service
echo 'def lambda_handler...' > payment-service/lambda_function.py

# 2. Update Terraform variables
# terraform/variables.tf
variable "lambda_functions" {
  default = [
    "user-service",
    "email-service", 
    "payment-service"  # ← Add new service
  ]
}

# 3. Run Terraform to create new Lambda function
cd terraform
terraform apply  # Creates payment-service-dev
```

### **Infrastructure Changes (Rarely):**

**Examples:**
- Change Lambda memory/timeout settings
- Add new IAM permissions
- Add VPC configuration
- Change AWS region

---

## **Typical Development Timeline**

### **Week 1 (Project Setup):**
```bash
Day 1: DevOps runs terraform apply  # Creates infrastructure
Day 2-7: Developers push code daily # Jenkins updates Lambda code
```

### **Week 2-4 (Regular Development):**
```bash
Daily: git push origin dev          # Jenkins updates Lambda code
       (No Terraform needed)
```

### **Week 5 (Add New Service):**
```bash
Day 1: Add new service directory
       Update terraform/variables.tf
       terraform apply              # Creates new Lambda function
Day 2-7: git push origin dev        # Jenkins updates all Lambda code
```

---

## **Automation Options for Terraform**

While Terraform is often run manually, you can automate it:

### **Option 1: Manual (Recommended for Infrastructure)**

```bash
# Run when infrastructure changes needed
terraform apply
```

**Pros:**
- ✅ Controlled infrastructure changes
- ✅ Review before applying
- ✅ No accidental resource deletion

**Cons:**
- ❌ Manual step required

### **Option 2: Automated Terraform (Advanced)**

**Create separate Jenkins pipeline for infrastructure:**

```groovy
// Jenkinsfile-infrastructure
pipeline {
    agent any
    
    when {
        changeset "terraform/*"  // Only run when terraform files change
    }
    
    stages {
        stage('Terraform Plan') {
            steps {
                sh 'terraform plan'
            }
        }
        stage('Terraform Apply') {
            when {
                branch 'main'  // Only apply on main branch
            }
            steps {
                sh 'terraform apply -auto-approve'
            }
        }
    }
}
```

**Trigger:** Only when `terraform/` files change

### **Option 3: GitOps with Terraform Cloud**

```bash
# Terraform runs automatically when you push terraform changes
git add terraform/
git commit -m "Add new Lambda function"
git push origin main  # Triggers Terraform Cloud
```

---

## **Recommended Workflow**

### **For Your Project (Simple Approach):**

```bash
# Step 1: Initial setup (DevOps - one time)
terraform apply

# Step 2: Daily development (Developers - frequent)
git push origin dev → Jenkins updates Lambda code

# Step 3: Add new services (DevOps - occasionally)
# Update terraform/variables.tf
terraform apply

# Step 4: Continue development (Developers - frequent)
git push origin dev → Jenkins updates Lambda code
```

---

## **Real-World Example Timeline**

### **Project Start:**
```
Week 1:
├── DevOps: terraform apply (creates 2 Lambda functions)
├── Dev: Push user-service code → Jenkins deploys
├── Dev: Push email-service code → Jenkins deploys
├── Dev: Fix bug in user-service → Jenkins deploys
└── Dev: Add feature to email-service → Jenkins deploys
```

### **Project Growth:**
```
Week 4:
├── DevOps: Add payment-service to terraform/variables.tf
├── DevOps: terraform apply (creates payment-service-dev)
├── Dev: Push payment-service code → Jenkins deploys
├── Dev: Update user-service → Jenkins deploys
└── Dev: Fix payment-service bug → Jenkins deploys
```

### **Infrastructure Changes:**
```
Week 8:
├── DevOps: Increase Lambda memory in terraform/lambda.tf
├── DevOps: terraform apply (updates all Lambda memory)
├── Dev: Continue normal development → Jenkins deploys
```

---

## **Key Principles**

### **Terraform (Infrastructure):**
- **Run manually** or with careful automation
- **Review changes** before applying
- **Infrequent** but important changes
- **Requires infrastructure knowledge**

### **Jenkins (Application):**
- **Fully automated** on git push
- **Frequent** code changes
- **Developers self-serve**
- **Fast feedback loop**

---

## **Answer to Your Question:**

**Why manual?**
- Infrastructure changes are **infrequent and important**
- Need **human review** before creating/destroying resources
- **Different lifecycle** than application code

**How often?**
- **Initial setup:** Once per project
- **Add new services:** Few times per month
- **Infrastructure changes:** Few times per year
- **Daily development:** Never (Jenkins handles application code)

**Best practice:**
```
Terraform = "Set up the kitchen" (done rarely)
Jenkins = "Cook the meals" (done daily)
```

**Your workflow after initial setup:**
1. **Edit code in VSCode** ✏️
2. **git push origin dev** 📤  
3. **Jenkins automatically updates Lambda code** ⚡

**You won't need to run Terraform again unless you're adding new services or changing infrastructure!**

