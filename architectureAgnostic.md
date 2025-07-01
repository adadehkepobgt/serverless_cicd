## Architecture-Agnostic Jenkins Pipeline

Here's the refined pipeline that works for ANY Lambda-based architecture:

```groovy
pipeline {
    agent any
    
    environment {
        AWS_DEFAULT_REGION = 'us-east-1'
        DEV_ACCOUNT = 'dev-account'
        S3_FRONTEND_BUCKET_DEV = 'euc-lambda-poc-frontend-dev'
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
                echo "🔀 Merged to dev branch - Running pipeline"
                echo "Building branch: ${env.BRANCH_NAME}"
                echo "Commit: ${env.GIT_COMMIT_SHORT}"
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh '''
                    echo "📦 Installing dependencies..."
                    
                    # Create virtual environment for backend
                    python3 -m venv venv
                    source venv/bin/activate
                    
                    # Upgrade pip
                    pip install --upgrade pip
                    
                    # Install backend Lambda dependencies
                    for lambda_dir in backend/lambdas/*/; do
                        if [ -f "$lambda_dir/requirements.txt" ]; then
                            echo "Installing dependencies for $lambda_dir"
                            pip install -r "$lambda_dir/requirements.txt"
                        fi
                    done
                    
                    # Install testing dependencies
                    pip install pytest boto3 moto pytest-html
                    
                    # Install frontend dependencies if frontend exists
                    if [ -d "frontend" ] && [ -f "frontend/package.json" ]; then
                        echo "Installing frontend dependencies..."
                        cd frontend
                        npm install
                        cd ..
                    fi
                '''
            }
        }
        
        stage('Build Frontend') {
            when {
                expression { fileExists('frontend') }
            }
            steps {
                sh '''
                    echo "🏗️ Building frontend..."
                    
                    # Build frontend assets if build tools are used
                    if [ -f "frontend/package.json" ]; then
                        echo "Running frontend build process..."
                        cd frontend
                        npm run build
                        cd ..
                    fi
                    
                    # Create standardized frontend deployment package
                    if [ -d "frontend/dist" ]; then
                        echo "Using frontend/dist/ as build output"
                        cp -r frontend/dist frontend-build/
                    elif [ -d "frontend/build" ]; then
                        echo "Using frontend/build/ as build output"
                        cp -r frontend/build frontend-build/
                    else
                        echo "Using frontend/ directory directly (static files)"
                        mkdir -p frontend-build
                        cp -r frontend/* frontend-build/ 2>/dev/null || true
                    fi
                    
                    echo "✅ Frontend build completed"
                '''
            }
        }
        
        stage('Run Unit Tests') {
            steps {
                sh '''
                    echo "🧪 Running unit tests..."
                    
                    source venv/bin/activate
                    export PYTHONPATH="${WORKSPACE}/backend/lambdas:${PYTHONPATH}"
                    
                    # Run backend unit tests
                    python -m pytest tests/unit/ -v \
                        --junit-xml=unit-test-results.xml \
                        --html=unit-test-report.html \
                        --self-contained-html
                '''
            }
            post {
                always {
                    junit 'unit-test-results.xml'
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: '.',
                        reportFiles: 'unit-test-report.html',
                        reportName: 'Unit Test Report'
                    ])
                }
            }
        }
        
        stage('Run Integration Tests') {
            steps {
                echo "🔗 Running integration tests..."
                echo "ℹ️  Integration tests are architecture-specific and test actual AWS service integrations"
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding', 
                     credentialsId: 'aws-dev-account-credentials']
                ]) {
                    sh '''
                        source venv/bin/activate
                        export PYTHONPATH="${WORKSPACE}/backend/lambdas:${PYTHONPATH}"
                        
                        # Run integration tests against dev environment
                        # These tests should verify your specific architecture:
                        # - API Gateway endpoints (if using API Gateway)
                        # - S3 event processing (if using S3 triggers)
                        # - SQS message processing (if using queues)
                        # - DynamoDB operations (if using database)
                        # - Any other AWS service integrations
                        python -m pytest tests/integration/ -v \
                            --junit-xml=integration-test-results.xml \
                            --html=integration-test-report.html \
                            --self-contained-html
                    '''
                }
            }
            post {
                always {
                    junit 'integration-test-results.xml'
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: '.',
                        reportFiles: 'integration-test-report.html',
                        reportName: 'Integration Test Report'
                    ])
                }
            }
        }
        
        stage('Deploy Frontend Assets') {
            when {
                expression { fileExists('frontend-build') }
            }
            steps {
                echo "📁 Deploying frontend assets to S3..."
                echo "ℹ️  S3 is used as static file storage regardless of backend architecture"
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding', 
                     credentialsId: 'aws-dev-account-credentials']
                ]) {
                    sh '''
                        # Deploy frontend assets to S3 (universal static file storage)
                        # This works for any architecture:
                        # - Direct S3 website hosting
                        # - CloudFront distribution source
                        # - Assets served by API Gateway
                        # - Resources loaded by any frontend framework
                        aws s3 sync frontend-build/ s3://${S3_FRONTEND_BUCKET_DEV}/ \
                            --delete \
                            --region ${AWS_DEFAULT_REGION}
                        
                        echo "✅ Frontend assets deployed to S3"
                    '''
                }
            }
        }
        
        stage('Deploy Backend Functions') {
            steps {
                echo "⚡ Deploying backend Lambda functions..."
                echo "ℹ️  Lambda deployment is identical regardless of triggers or integrations"
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding', 
                     credentialsId: 'aws-dev-account-credentials']
                ]) {
                    sh '''
                        # Deploy all Lambda functions found in backend/lambdas/
                        # This works for any Lambda integration pattern:
                        # - API Gateway → Lambda
                        # - S3 Events → Lambda  
                        # - SQS/SNS → Lambda
                        # - EventBridge → Lambda
                        # - Direct invocation
                        # - Application Load Balancer → Lambda
                        # - Lambda → Lambda
                        
                        for lambda_dir in backend/lambdas/*/; do
                            if [ -d "$lambda_dir" ]; then
                                lambda_name=$(basename $lambda_dir)
                                echo "📦 Packaging and deploying $lambda_name"
                                
                                # Create clean deployment package
                                cd $lambda_dir
                                zip -r ../../${lambda_name}-dev.zip . \
                                    -x "*.pyc" "*__pycache__*" "tests/*" "*.md" ".git*" "*.DS_Store"
                                cd ../..
                                
                                # Update Lambda function code
                                aws lambda update-function-code \
                                    --function-name ${lambda_name}-dev \
                                    --zip-file fileb://${lambda_name}-dev.zip \
                                    --region ${AWS_DEFAULT_REGION}
                                
                                echo "✅ ${lambda_name}-dev updated successfully"
                            fi
                        done
                    '''
                }
            }
        }
        
        stage('Verify Deployment Success') {
            steps {
                echo "🔍 Verifying deployment success..."
                echo "ℹ️  Checking deployment status, not testing integrations"
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding', 
                     credentialsId: 'aws-dev-account-credentials']
                ]) {
                    sh '''
                        echo "📁 Frontend Assets Status:"
                        if aws s3 ls s3://${S3_FRONTEND_BUCKET_DEV}/ >/dev/null 2>&1; then
                            aws s3 ls s3://${S3_FRONTEND_BUCKET_DEV}/ --recursive --human-readable --summarize
                            echo "✅ Frontend assets successfully deployed"
                        else
                            echo "ℹ️  No frontend assets found (API-only architecture)"
                        fi
                        
                        echo ""
                        echo "⚡ Lambda Function Deployment Status:"
                        for lambda_dir in backend/lambdas/*/; do
                            if [ -d "$lambda_dir" ]; then
                                lambda_name=$(basename $lambda_dir)
                                echo "Checking ${lambda_name}-dev deployment status..."
                                
                                aws lambda get-function \
                                    --function-name ${lambda_name}-dev \
                                    --region ${AWS_DEFAULT_REGION} \
                                    --query 'Configuration.[FunctionName,LastModified,State,Runtime]' \
                                    --output table
                            fi
                        done
                        
                        echo ""
                        echo "✅ All deployments verified successfully"
                        echo "ℹ️  Integration testing validates that deployed components work together"
                    '''
                }
            }
        }
    }
    
    post {
        always {
            sh '''
                echo "🧹 Cleaning up build artifacts..."
                rm -rf venv || true
                rm -f *.zip || true
                rm -rf frontend-build || true
                rm -rf node_modules || true
            '''
        }
        success {
            script {
                def hasFrontend = fileExists('frontend')
                def frontendMessage = hasFrontend ? 
                    "✅ Frontend assets deployed to S3" : 
                    "ℹ️  No frontend (API-only architecture)"
                
                echo """
🎉 DEV DEPLOYMENT COMPLETED SUCCESSFULLY!

📋 Summary:
✅ Unit tests passed (Lambda logic verified)
✅ Integration tests passed (Architecture verified)
${frontendMessage}
✅ Backend Lambda functions updated
✅ All deployments verified

🌐 Dev Environment Ready:
${hasFrontend ? "Frontend: https://${env.S3_FRONTEND_BUCKET_DEV}.s3.${env.AWS_DEFAULT_REGION}.amazonaws.com" : ""}
Backend: Lambda functions active in dev account

🏗️ Architecture Support:
This pipeline works with any Lambda-based architecture:
- API Gateway + Lambda
- Event-driven (S3, SQS, SNS, EventBridge)
- Serverless web applications  
- Microservices
- Data processing pipelines

🚀 Next Steps:
1. Test your specific architecture in dev environment
2. Verify all integrations work as expected
3. Ready for CCB review when needed
                """
            }
        }
        failure {
            echo '''
❌ DEV DEPLOYMENT FAILED!

🔧 Troubleshooting Steps:
1. Check console output above for specific errors
2. Common issues:
   - Unit test failures: Fix Lambda function logic
   - Integration test failures: Check AWS service configurations
   - AWS permissions: Verify aws-dev-account-credentials
   - Lambda packaging: Check requirements.txt files
   - Missing AWS resources: Ensure Lambda functions and S3 bucket exist

🔄 To retry:
Push another commit to dev branch to trigger pipeline again

💡 Architecture-specific debugging:
- API Gateway: Check endpoint configurations and Lambda triggers
- Event-driven: Verify event source mappings and permissions
- S3: Check bucket policies and event configurations
- SQS/SNS: Verify queue/topic permissions and Lambda subscriptions
            '''
        }
        unstable {
            echo '''
⚠️ DEPLOYMENT COMPLETED WITH WARNINGS

Some tests may have failed but deployment proceeded.
Check test reports for details.

🔍 Common causes:
- Integration tests failed (architecture-specific issues)
- Flaky tests due to AWS service latency
- Partial functionality not yet implemented
            '''
        }
    }
}
```

---

## **Key Architecture-Agnostic Improvements**

### **1. Conditional Frontend Handling**
```groovy
when {
    expression { fileExists('frontend') }
}
```
- **API-only projects**: Skip frontend stages entirely
- **Web applications**: Process frontend normally

### **2. Universal Lambda Deployment**
```groovy
# Works for ANY Lambda trigger:
aws lambda update-function-code \
    --function-name ${lambda_name}-dev \
    --zip-file fileb://${lambda_name}-dev.zip
```
- **API Gateway**: Lambda functions serve HTTP requests
- **S3 Events**: Lambda functions process file uploads
- **SQS**: Lambda functions process queue messages
- **EventBridge**: Lambda functions handle custom events

### **3. Architecture-Aware Messaging**
```groovy
echo "ℹ️  Integration tests are architecture-specific and test actual AWS service integrations"
```
- **Clear documentation** of what each stage does
- **Explains responsibility** for integration tests

### **4. Flexible Verification**
```groovy
if aws s3 ls s3://${S3_FRONTEND_BUCKET_DEV}/ >/dev/null 2>&1; then
    # Frontend exists
else
    echo "ℹ️  No frontend assets found (API-only architecture)"
fi
```
- **Handles both**: Web apps and pure APIs
- **No failures**: If frontend doesn't exist

### **5. Smart Success Messages**
```groovy
def hasFrontend = fileExists('frontend')
def frontendMessage = hasFrontend ? 
    "✅ Frontend assets deployed to S3" : 
    "ℹ️  No frontend (API-only architecture)"
```
- **Dynamic messaging** based on project structure
- **Relevant information** for each architecture type

---

## **Supported Architectures (Same Pipeline)**

### **API Gateway + Lambda**
```
tests/integration/test_api_endpoints.py
→ Tests HTTP endpoints and responses
```

### **Event-Driven Processing**
```
tests/integration/test_s3_processing.py
→ Tests file upload triggers and processing
```

### **Microservices**
```
tests/integration/test_service_communication.py
→ Tests Lambda-to-Lambda communication
```

### **Data Pipelines**
```
tests/integration/test_data_flow.py
→ Tests data transformation and storage
```

**The pipeline code never changes - only the integration tests adapt to your specific architecture!**


## Repository Structure for Architecture-Agnostic Pipeline

### **Complete Repository Structure**

```
euc-lambda-poc/
├── Jenkinsfile                           # Jenkins pipeline configuration
├── README.md                             # Project documentation
├── .gitignore                            # Git ignore patterns
├── requirements-dev.txt                  # Development dependencies (optional)
├── 
├── frontend/                             # Frontend assets (conditional)
│   ├── index.html                        # Main HTML file
│   ├── styles.css                        # CSS styles
│   ├── app.js                           # JavaScript application
│   ├── assets/                          # Static assets
│   │   ├── images/
│   │   ├── fonts/
│   │   └── icons/
│   ├── package.json                     # Frontend build dependencies (optional)
│   ├── package-lock.json                # Dependency lock file (if using npm)
│   └── webpack.config.js                # Build configuration (if using webpack)
│   
├── backend/                             # Backend Lambda functions
│   └── lambdas/                         # All Lambda functions
│       ├── user-authentication/         # Individual Lambda function
│       │   ├── lambda_function.py       # Main handler
│       │   ├── requirements.txt         # Function dependencies
│       │   ├── helpers.py               # Supporting modules
│       │   └── config.py                # Configuration
│       ├── data-processor/              # Another Lambda function
│       │   ├── lambda_function.py
│       │   ├── requirements.txt
│       │   ├── data_utils.py
│       │   └── models.py
│       ├── api-gateway-handler/         # API Gateway Lambda
│       │   ├── lambda_function.py
│       │   ├── requirements.txt
│       │   ├── routes.py
│       │   └── middleware.py
│       └── notification-service/        # Event-driven Lambda
│           ├── lambda_function.py
│           ├── requirements.txt
│           └── email_templates.py
│
├── tests/                               # All test files
│   ├── unit/                           # Unit tests (architecture-agnostic)
│   │   ├── test_user_authentication.py  # Test Lambda logic
│   │   ├── test_data_processor.py       # Test data processing
│   │   ├── test_api_handler.py          # Test API logic
│   │   └── test_notification_service.py # Test notification logic
│   ├── integration/                    # Integration tests (architecture-specific)
│   │   ├── test_api_endpoints.py       # API Gateway integration
│   │   ├── test_s3_processing.py       # S3 event integration
│   │   ├── test_notification_flow.py   # SNS/SQS integration
│   │   └── test_end_to_end.py          # Full workflow tests
│   ├── fixtures/                       # Test data and fixtures
│   │   ├── sample_data.json
│   │   ├── test_files/
│   │   └── mock_responses.py
│   └── conftest.py                     # Pytest configuration
│
├── infrastructure/                     # Infrastructure as Code (optional)
│   ├── cloudformation/                 # CloudFormation templates
│   │   ├── lambda-functions.yaml
│   │   ├── api-gateway.yaml
│   │   └── s3-buckets.yaml
│   ├── terraform/                      # Terraform configurations
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── serverless.yml                  # Serverless Framework (alternative)
│
├── docs/                               # Documentation
│   ├── api/                           # API documentation
│   │   ├── endpoints.md
│   │   └── authentication.md
│   ├── architecture.md                # Architecture overview
│   ├── deployment.md                  # Deployment instructions
│   └── testing.md                     # Testing guidelines
│
└── scripts/                           # Utility scripts
    ├── deploy-local.sh                # Local deployment script
    ├── run-tests.sh                   # Test runner script
    └── setup-dev-env.sh               # Development setup
```

---

## **Architecture-Specific Examples**

### **Example 1: API Gateway Architecture**

```
euc-lambda-poc/
├── Jenkinsfile
├── frontend/                          # React/Vue web app
│   ├── src/
│   ├── public/
│   ├── package.json
│   └── build/                         # Generated by npm run build
├── backend/
│   └── lambdas/
│       ├── users-api/                 # GET/POST /users
│       │   ├── lambda_function.py
│       │   └── requirements.txt
│       ├── products-api/              # GET/POST /products  
│       │   ├── lambda_function.py
│       │   └── requirements.txt
│       ├── orders-api/                # GET/POST /orders
│       │   ├── lambda_function.py
│       │   └── requirements.txt
│       └── auth-api/                  # POST /login, /register
│           ├── lambda_function.py
│           └── requirements.txt
└── tests/
    ├── unit/                          # Test individual Lambda logic
    └── integration/
        └── test_api_endpoints.py      # Test actual API Gateway endpoints
```

**Integration Test Example:**
```python
# tests/integration/test_api_endpoints.py
import requests
import pytest

def test_users_api():
    response = requests.get('https://your-api.execute-api.us-east-1.amazonaws.com/dev/users')
    assert response.status_code == 200
    assert 'users' in response.json()

def test_auth_api():
    response = requests.post('https://your-api.execute-api.us-east-1.amazonaws.com/dev/login',
                           json={'username': 'test', 'password': 'test'})
    assert response.status_code in [200, 401]
```

### **Example 2: Event-Driven Architecture**

```
euc-lambda-poc/
├── Jenkinsfile
├── backend/
│   └── lambdas/
│       ├── s3-file-processor/         # Triggered by S3 uploads
│       │   ├── lambda_function.py
│       │   └── requirements.txt
│       ├── order-queue-processor/     # Triggered by SQS messages
│       │   ├── lambda_function.py
│       │   └── requirements.txt
│       ├── email-notification/        # Triggered by SNS events
│       │   ├── lambda_function.py
│       │   └── requirements.txt
│       └── data-sync/                 # Triggered by DynamoDB streams
│           ├── lambda_function.py
│           └── requirements.txt
└── tests/
    ├── unit/                          # Test Lambda logic
    └── integration/
        ├── test_s3_processing.py      # Test S3 event processing
        ├── test_queue_processing.py   # Test SQS message processing
        └── test_notification_flow.py  # Test SNS notifications
```

**Integration Test Example:**
```python
# tests/integration/test_s3_processing.py
import boto3
import time

def test_s3_file_processing():
    s3 = boto3.client('s3')
    
    # Upload test file to trigger Lambda
    s3.put_object(
        Bucket='input-bucket-dev',
        Key='test-file.csv',
        Body=b'name,age\nJohn,25\nJane,30'
    )
    
    # Wait for processing
    time.sleep(10)
    
    # Verify processed file exists
    response = s3.list_objects_v2(
        Bucket='output-bucket-dev',
        Prefix='processed-test-file'
    )
    assert response['KeyCount'] > 0
```

### **Example 3: Pure API Backend (No Frontend)**

```
euc-lambda-poc/
├── Jenkinsfile
├── backend/
│   └── lambdas/
│       └── graphql-api/               # Single GraphQL endpoint
│           ├── lambda_function.py
│           ├── requirements.txt
│           ├── schema.py
│           └── resolvers.py
└── tests/
    ├── unit/
    │   └── test_graphql_logic.py
    └── integration/
        └── test_graphql_api.py        # Test GraphQL queries/mutations
```

### **Example 4: Microservices Architecture**

```
euc-lambda-poc/
├── Jenkinsfile
├── frontend/                          # Admin dashboard
│   ├── index.html
│   └── app.js
├── backend/
│   └── lambdas/
│       ├── user-service/              # User management microservice
│       │   ├── lambda_function.py
│       │   └── requirements.txt
│       ├── inventory-service/         # Inventory microservice
│       │   ├── lambda_function.py
│       │   └── requirements.txt
│       ├── order-service/             # Order processing microservice
│       │   ├── lambda_function.py
│       │   └── requirements.txt
│       └── notification-service/      # Notification microservice
│           ├── lambda_function.py
│           └── requirements.txt
└── tests/
    ├── unit/                          # Test each service logic
    └── integration/
        └── test_service_communication.py  # Test inter-service communication
```

---

## **Key Files Explained**

### **Jenkinsfile (Required)**
```groovy
# The architecture-agnostic pipeline we created
# Works with any of the above structures
```

### **requirements.txt (Per Lambda)**
```txt
# backend/lambdas/user-authentication/requirements.txt
boto3==1.26.137
bcrypt==4.0.1
jwt==1.3.1
requests==2.31.0
```

### **Frontend package.json (Optional)**
```json
{
  "name": "euc-lambda-poc-frontend",
  "version": "1.0.0",
  "scripts": {
    "build": "webpack --mode production",
    "dev": "webpack serve --mode development"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "webpack": "^5.88.0",
    "webpack-cli": "^5.1.4"
  }
}
```

### **Sample Lambda Function**
```python
# backend/lambdas/user-authentication/lambda_function.py
import json
import boto3

def lambda_handler(event, context):
    """
    Architecture-agnostic Lambda function
    Can be triggered by:
    - API Gateway (HTTP requests)
    - S3 Events (file uploads)
    - SQS Messages (queue processing)
    - Direct invocation
    """
    
    # Your business logic here
    result = process_request(event)
    
    return {
        'statusCode': 200,
        'body': json.dumps(result)
    }

def process_request(event):
    # Business logic implementation
    return {'message': 'Success'}
```

### **.gitignore**
```
# Python
__pycache__/
*.pyc
*.pyo
venv/
.pytest_cache/

# Frontend
node_modules/
dist/
build/
*.log

# IDE
.vscode/
.idea/
*.swp

# AWS
.aws/

# Jenkins
*.zip
frontend-build/

# OS
.DS_Store
Thumbs.db
```

---

## **Pipeline Compatibility Matrix**

| Architecture | Frontend | Backend | Pipeline Changes |
|-------------|----------|---------|------------------|
| **API Gateway + React** | ✅ frontend/ | ✅ backend/lambdas/ | None |
| **Event-Driven Processing** | ❌ No frontend | ✅ backend/lambdas/ | None |
| **Microservices** | ✅ frontend/ | ✅ backend/lambdas/ | None |
| **Pure API Backend** | ❌ No frontend | ✅ backend/lambdas/ | None |
| **Serverless Web App** | ✅ frontend/ | ✅ backend/lambdas/ | None |

**The pipeline automatically adapts to whatever structure you have!**

---

## **Getting Started**

### **Step 1: Choose Your Architecture**
Pick one of the examples above that matches your use case

### **Step 2: Create Repository Structure**
```bash
mkdir euc-lambda-poc
cd euc-lambda-poc

# Create basic structure
mkdir -p backend/lambdas
mkdir -p tests/{unit,integration}
mkdir -p frontend  # If you have frontend

# Add Jenkinsfile
# (Copy the architecture-agnostic pipeline we created)
```

### **Step 3: Add Your Lambda Functions**
```bash
# For each Lambda function:
mkdir backend/lambdas/your-function-name
# Add lambda_function.py and requirements.txt
```

### **Step 4: Add Tests**
```bash
# Unit tests (test Lambda logic)
# Integration tests (test your specific architecture)
```

**The pipeline will automatically discover and deploy everything!**
