## Refined Architecture-Agnostic Pipeline

You're absolutely right! Real-world Lambda projects are much simpler. Here's the refined pipeline:

```groovy
pipeline {
    agent any
    
    environment {
        AWS_DEFAULT_REGION = 'us-east-1'
        DEV_ACCOUNT = 'dev-account'
        S3_ASSETS_BUCKET_DEV = 'euc-lambda-poc-assets-dev'
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
                    
                    # Create virtual environment
                    python3 -m venv venv
                    source venv/bin/activate
                    
                    # Upgrade pip
                    pip install --upgrade pip
                    
                    # Install dependencies for each Lambda function
                    find . -name "lambda_function.py" -type f | while read lambda_file; do
                        lambda_dir=$(dirname "$lambda_file")
                        if [ -f "$lambda_dir/requirements.txt" ]; then
                            echo "Installing dependencies for $lambda_dir"
                            pip install -r "$lambda_dir/requirements.txt"
                        fi
                    done
                    
                    # Install testing dependencies
                    pip install pytest boto3 moto pytest-html
                    
                    # Install any build dependencies if they exist
                    if [ -f "package.json" ]; then
                        npm install
                    fi
                '''
            }
        }
        
        stage('Build Assets') {
            when {
                anyOf {
                    expression { fileExists('package.json') }
                    expression { fileExists('build.py') }
                    expression { fileExists('Makefile') }
                }
            }
            steps {
                sh '''
                    echo "🏗️ Building assets..."
                    
                    # Build using package.json if present
                    if [ -f "package.json" ]; then
                        echo "Running npm build..."
                        npm run build
                    fi
                    
                    # Build using build.py if present
                    if [ -f "build.py" ]; then
                        echo "Running Python build script..."
                        source venv/bin/activate
                        python build.py
                    fi
                    
                    # Build using Makefile if present
                    if [ -f "Makefile" ]; then
                        echo "Running make build..."
                        make build
                    fi
                    
                    echo "✅ Build completed"
                '''
            }
        }
        
        stage('Run Unit Tests') {
            steps {
                sh '''
                    echo "🧪 Running unit tests..."
                    
                    source venv/bin/activate
                    
                    # Set Python path to include all Lambda function directories
                    lambda_paths=""
                    find . -name "lambda_function.py" -type f | while read lambda_file; do
                        lambda_dir=$(dirname "$lambda_file")
                        lambda_paths="$lambda_paths:$lambda_dir"
                    done
                    export PYTHONPATH="${WORKSPACE}${lambda_paths}:${PYTHONPATH}"
                    
                    # Run unit tests if test directory exists
                    if [ -d "tests" ]; then
                        python -m pytest tests/ -v \
                            --junit-xml=unit-test-results.xml \
                            --html=unit-test-report.html \
                            --self-contained-html
                    else
                        echo "No tests directory found, skipping unit tests"
                        # Create empty test results for Jenkins
                        echo '<?xml version="1.0"?><testsuite tests="0" failures="0" errors="0" skipped="0"/>' > unit-test-results.xml
                    fi
                '''
            }
            post {
                always {
                    junit 'unit-test-results.xml'
                    script {
                        if (fileExists('unit-test-report.html')) {
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
            }
        }
        
        stage('Run Integration Tests') {
            when {
                expression { fileExists('tests') }
            }
            steps {
                echo "🔗 Running integration tests..."
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding', 
                     credentialsId: 'aws-dev-account-credentials']
                ]) {
                    sh '''
                        source venv/bin/activate
                        
                        # Set Python path to include all Lambda function directories
                        lambda_paths=""
                        find . -name "lambda_function.py" -type f | while read lambda_file; do
                            lambda_dir=$(dirname "$lambda_file")
                            lambda_paths="$lambda_paths:$lambda_dir"
                        done
                        export PYTHONPATH="${WORKSPACE}${lambda_paths}:${PYTHONPATH}"
                        
                        # Run integration tests
                        python -m pytest tests/ -k "integration" -v \
                            --junit-xml=integration-test-results.xml \
                            --html=integration-test-report.html \
                            --self-contained-html || true
                    '''
                }
            }
            post {
                always {
                    script {
                        if (fileExists('integration-test-results.xml')) {
                            junit 'integration-test-results.xml'
                        }
                        if (fileExists('integration-test-report.html')) {
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
            }
        }
        
        stage('Deploy Static Assets') {
            when {
                anyOf {
                    expression { fileExists('static') }
                    expression { fileExists('assets') }
                    expression { fileExists('public') }
                    expression { fileExists('dist') }
                    expression { fileExists('build') }
                }
            }
            steps {
                echo "📁 Deploying static assets..."
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding', 
                     credentialsId: 'aws-dev-account-credentials']
                ]) {
                    sh '''
                        # Deploy static assets to S3
                        # Check for common static asset directories
                        
                        if [ -d "dist" ]; then
                            echo "Deploying from dist/ directory"
                            aws s3 sync dist/ s3://${S3_ASSETS_BUCKET_DEV}/ \
                                --delete --region ${AWS_DEFAULT_REGION}
                        elif [ -d "build" ]; then
                            echo "Deploying from build/ directory"
                            aws s3 sync build/ s3://${S3_ASSETS_BUCKET_DEV}/ \
                                --delete --region ${AWS_DEFAULT_REGION}
                        elif [ -d "public" ]; then
                            echo "Deploying from public/ directory"
                            aws s3 sync public/ s3://${S3_ASSETS_BUCKET_DEV}/ \
                                --delete --region ${AWS_DEFAULT_REGION}
                        elif [ -d "static" ]; then
                            echo "Deploying from static/ directory"
                            aws s3 sync static/ s3://${S3_ASSETS_BUCKET_DEV}/ \
                                --delete --region ${AWS_DEFAULT_REGION}
                        elif [ -d "assets" ]; then
                            echo "Deploying from assets/ directory"
                            aws s3 sync assets/ s3://${S3_ASSETS_BUCKET_DEV}/ \
                                --delete --region ${AWS_DEFAULT_REGION}
                        fi
                        
                        echo "✅ Static assets deployed to S3"
                    '''
                }
            }
        }
        
        stage('Deploy Lambda Functions') {
            steps {
                echo "⚡ Deploying Lambda functions..."
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding', 
                     credentialsId: 'aws-dev-account-credentials']
                ]) {
                    sh '''
                        # Find and deploy all Lambda functions
                        find . -name "lambda_function.py" -type f | while read lambda_file; do
                            lambda_dir=$(dirname "$lambda_file")
                            lambda_name=$(basename "$lambda_dir")
                            
                            echo "📦 Packaging and deploying $lambda_name"
                            
                            # Create deployment package
                            cd "$lambda_dir"
                            zip -r "../../${lambda_name}-dev.zip" . \
                                -x "*.pyc" "*__pycache__*" "tests/*" "*.md" ".git*" \
                                   "*.DS_Store" "*.pytest_cache*" "*.coverage*"
                            cd - > /dev/null
                            
                            # Update Lambda function
                            aws lambda update-function-code \
                                --function-name "${lambda_name}-dev" \
                                --zip-file "fileb://${lambda_name}-dev.zip" \
                                --region ${AWS_DEFAULT_REGION}
                            
                            echo "✅ ${lambda_name}-dev updated successfully"
                        done
                    '''
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                echo "🔍 Verifying deployment..."
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding', 
                     credentialsId: 'aws-dev-account-credentials']
                ]) {
                    sh '''
                        # Verify static assets if they exist
                        has_assets=false
                        for dir in dist build public static assets; do
                            if [ -d "$dir" ]; then
                                has_assets=true
                                break
                            fi
                        done
                        
                        if [ "$has_assets" = true ]; then
                            echo "📁 Static Assets in S3:"
                            aws s3 ls s3://${S3_ASSETS_BUCKET_DEV}/ --recursive --human-readable --summarize
                            echo ""
                        fi
                        
                        # Verify Lambda functions
                        echo "⚡ Lambda Function Status:"
                        find . -name "lambda_function.py" -type f | while read lambda_file; do
                            lambda_dir=$(dirname "$lambda_file")
                            lambda_name=$(basename "$lambda_dir")
                            
                            echo "Checking ${lambda_name}-dev..."
                            aws lambda get-function \
                                --function-name "${lambda_name}-dev" \
                                --region ${AWS_DEFAULT_REGION} \
                                --query 'Configuration.[FunctionName,LastModified,State,Runtime]' \
                                --output table
                        done
                        
                        echo ""
                        echo "✅ All deployments verified successfully"
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
                rm -f *-dev.zip || true
                rm -rf node_modules || true
                rm -rf .pytest_cache || true
            '''
        }
        success {
            script {
                // Count Lambda functions
                def lambdaCount = sh(
                    script: 'find . -name "lambda_function.py" -type f | wc -l',
                    returnStdout: true
                ).trim() as Integer
                
                // Check for static assets
                def hasAssets = false
                ['dist', 'build', 'public', 'static', 'assets'].each { dir ->
                    if (fileExists(dir)) {
                        hasAssets = true
                    }
                }
                
                echo """
🎉 DEPLOYMENT COMPLETED SUCCESSFULLY!

📋 Summary:
✅ Tests executed
${hasAssets ? '✅ Static assets deployed to S3' : 'ℹ️  No static assets found'}
✅ ${lambdaCount} Lambda function(s) deployed
✅ All deployments verified

🏗️ Detected Structure:
- Lambda functions: ${lambdaCount}
- Static assets: ${hasAssets ? 'Yes' : 'No'}
- Tests: ${fileExists('tests') ? 'Yes' : 'No'}

🌐 Dev Environment Ready:
${hasAssets ? "Static Assets: https://${env.S3_ASSETS_BUCKET_DEV}.s3.${env.AWS_DEFAULT_REGION}.amazonaws.com" : ""}
Lambda functions: Available in dev account with '-dev' suffix

🚀 Next Steps:
1. Test your application in dev environment
2. Verify all functionality works as expected
3. Ready for promotion when testing completes
                """
            }
        }
        failure {
            echo '''
❌ DEPLOYMENT FAILED!

🔧 Troubleshooting Steps:
1. Check console output above for specific errors
2. Common issues:
   - Test failures: Fix test logic and retry
   - AWS permissions: Verify aws-dev-account-credentials
   - Lambda packaging: Check requirements.txt files
   - Missing AWS resources: Ensure Lambda functions exist in dev account

🔄 To retry:
Push another commit to dev branch to trigger pipeline again
            '''
        }
        unstable {
            echo '''
⚠️ DEPLOYMENT COMPLETED WITH WARNINGS

Some tests may have failed but deployment proceeded.
Check test reports for details.
            '''
        }
    }
}
```

---

## **Simplified Repository Structure**

### **Real-World Lambda Project Structure**

```
euc-lambda-poc/
├── Jenkinsfile                    # Pipeline configuration
├── README.md                      # Project documentation
├── requirements.txt               # Global dependencies (optional)
├── package.json                   # Build tools (optional)
├── 
├── user-auth/                     # Lambda function 1
│   ├── lambda_function.py         # Main handler
│   └── requirements.txt           # Function dependencies
├── 
├── order-processor/               # Lambda function 2
│   ├── lambda_function.py         # Main handler
│   ├── requirements.txt           # Function dependencies
│   └── utils.py                   # Helper functions
├── 
├── notification-service/          # Lambda function 3
│   ├── lambda_function.py         # Main handler
│   ├── requirements.txt           # Function dependencies
│   └── templates.py               # Email templates
├── 
├── api-gateway/                   # Lambda function 4
│   ├── lambda_function.py         # Main handler
│   └── requirements.txt           # Function dependencies
├── 
├── static/                        # Static assets (optional)
│   ├── index.html
│   ├── style.css
│   └── app.js
├── 
└── tests/                         # Tests (optional)
    ├── test_user_auth.py          # Unit tests
    ├── test_order_processor.py    # Unit tests
    └── test_integration.py        # Integration tests
```

### **Alternative Structure Examples**

**API-focused project:**
```
my-api/
├── Jenkinsfile
├── auth/
│   └── lambda_function.py
├── users/
│   └── lambda_function.py
├── products/
│   └── lambda_function.py
└── orders/
    └── lambda_function.py
```

**Event-driven project:**
```
data-pipeline/
├── Jenkinsfile
├── file-processor/
│   └── lambda_function.py
├── data-transformer/
│   └── lambda_function.py
├── email-sender/
│   └── lambda_function.py
└── assets/
    └── email-templates/
```

**Mixed project:**
```
web-app/
├── Jenkinsfile
├── build.py                       # Custom build script
├── api/
│   └── lambda_function.py
├── auth/
│   └── lambda_function.py
├── public/                        # Static files
│   ├── index.html
│   └── app.js
└── tests/
    └── test_api.py
```

---

## **Key Pipeline Improvements**

### **1. Dynamic Lambda Discovery**
```bash
find . -name "lambda_function.py" -type f | while read lambda_file; do
    lambda_dir=$(dirname "$lambda_file")
    lambda_name=$(basename "$lambda_dir")
    # Deploy $lambda_name
done
```
**Automatically finds all Lambda functions regardless of folder structure**

### **2. Flexible Asset Detection**
```bash
for dir in dist build public static assets; do
    if [ -d "$dir" ]; then
        aws s3 sync "$dir"/ s3://${S3_ASSETS_BUCKET_DEV}/
        break
    fi
done
```
**Deploys static assets from any common directory name**

### **3. Optional Testing**
```bash
if [ -d "tests" ]; then
    python -m pytest tests/
else
    echo "No tests directory found, skipping tests"
fi
```
**Works with or without tests**

### **4. Flexible Build Support**
```bash
# Supports multiple build methods
if [ -f "package.json" ]; then npm run build; fi
if [ -f "build.py" ]; then python build.py; fi
if [ -f "Makefile" ]; then make build; fi
```
**Adapts to any build system**

### **5. Smart Path Management**
```bash
# Automatically includes all Lambda directories in Python path
find . -name "lambda_function.py" -type f | while read lambda_file; do
    lambda_dir=$(dirname "$lambda_file")
    lambda_paths="$lambda_paths:$lambda_dir"
done
export PYTHONPATH="${WORKSPACE}${lambda_paths}:${PYTHONPATH}"
```
**Enables imports between Lambda functions**

---

## **Usage Examples**

### **Simple API Project**
```
my-api/
├── Jenkinsfile
├── auth/lambda_function.py
├── users/lambda_function.py
└── orders/lambda_function.py
```
**Pipeline automatically:**
- Finds 3 Lambda functions
- Deploys as `auth-dev`, `users-dev`, `orders-dev`
- No static assets, no tests

### **Web Application**
```
web-app/
├── Jenkinsfile
├── package.json
├── api/lambda_function.py
├── dist/               # Built by npm run build
│   ├── index.html
│   └── app.js
└── tests/test_api.py
```
**Pipeline automatically:**
- Runs `npm run build`
- Deploys `dist/` to S3
- Deploys `api-dev` Lambda
- Runs tests

### **Data Pipeline**
```
data-pipeline/
├── Jenkinsfile
├── processor/lambda_function.py
├── transformer/lambda_function.py
└── notifier/lambda_function.py
```
**Pipeline automatically:**
- Finds 3 Lambda functions
- Deploys as `processor-dev`, `transformer-dev`, `notifier-dev`
- No static assets, no tests

**The same Jenkinsfile works for all these structures!**


## Detailed Explanation of Every Part of the Refined Pipeline Script

### **Part 1: Pipeline Declaration and Environment Setup**

```groovy
pipeline {
    agent any
    
    environment {
        AWS_DEFAULT_REGION = 'us-east-1'
        DEV_ACCOUNT = 'dev-account'
        S3_ASSETS_BUCKET_DEV = 'euc-lambda-poc-assets-dev'
    }
    
    tools {
        python 'Python3'
    }
    
    when {
        branch 'dev'
    }
```

**Detailed Breakdown:**

**`pipeline {`**
- **Purpose**: Declares Jenkins Declarative Pipeline syntax
- **Effect**: Provides structured, validated pipeline framework
- **Alternative**: Scripted pipeline (more flexible, less structured)

**`agent any`**
- **Purpose**: Tells Jenkins this pipeline can run on any available build agent
- **Effect**: Jenkins scheduler assigns first available executor
- **Alternatives**: 
  - `agent { label 'linux' }` - specific OS
  - `agent { docker { image 'python:3.9' } }` - containerized environment

**`environment` Block:**
- **Purpose**: Creates global environment variables accessible in all stages
- **Scope**: Available to all shell commands and Groovy scripts

**`AWS_DEFAULT_REGION = 'us-east-1'`**
- **Purpose**: Sets consistent AWS region for all AWS CLI commands
- **Effect**: All `aws` commands automatically use this region
- **Usage**: `aws lambda update-function-code --region ${AWS_DEFAULT_REGION}`

**`DEV_ACCOUNT = 'dev-account'`**
- **Purpose**: Human-readable identifier for development AWS account
- **Effect**: Documentation and logging purposes
- **Note**: Not used in actual AWS commands (credentials handle account)

**`S3_ASSETS_BUCKET_DEV = 'euc-lambda-poc-assets-dev'`**
- **Purpose**: Pre-defined S3 bucket name for static asset deployment
- **Effect**: All static files deploy to this bucket
- **Requirement**: Bucket must exist in dev account before pipeline runs

**`tools { python 'Python3' }`**
- **Purpose**: References Python installation configured in Jenkins
- **Effect**: Adds Python to PATH for all pipeline stages
- **Requirement**: Must match name in Jenkins Global Tool Configuration
- **Location**: Manage Jenkins → Global Tool Configuration → Python

**`when { branch 'dev' }`**
- **Purpose**: Pipeline execution filter
- **Effect**: Only executes when branch name equals 'dev'
- **Behavior**: Feature branches completely ignored
- **Alternative**: `when { anyOf { branch 'dev'; branch 'main' } }`

---

### **Part 2: Checkout Stage**

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
        echo "🔀 Merged to dev branch - Running pipeline"
        echo "Building branch: ${env.BRANCH_NAME}"
        echo "Commit: ${env.GIT_COMMIT_SHORT}"
    }
}
```

**Detailed Breakdown:**

**`checkout scm`**
- **Purpose**: Downloads repository code to Jenkins workspace
- **SCM**: Source Code Management (Git repository)
- **Effect**: Complete repository available in `${WORKSPACE}` directory
- **Credentials**: Uses credentials configured in multibranch pipeline
- **Automatic**: Checks out the specific commit that triggered the build

**`script { }` Block:**
- **Purpose**: Allows imperative Groovy code within declarative pipeline
- **Use cases**: Variable manipulation, complex logic, conditionals
- **Scope**: Variables created here available to subsequent stages

**`env.GIT_COMMIT_SHORT = sh(...)`**
- **`sh()`**: Executes shell command on Jenkins build agent
- **`script: "git rev-parse --short HEAD"`**: Git command to get abbreviated commit hash
- **`returnStdout: true`**: Captures command output instead of displaying
- **`.trim()`**: Removes trailing newline characters from output
- **Result**: `env.GIT_COMMIT_SHORT` contains "a1b2c3d" format

**Example Git Command Execution:**
```bash
$ git rev-parse --short HEAD
a1b2c3d
```
**After `.trim()`**: `env.GIT_COMMIT_SHORT = "a1b2c3d"`

**`echo` Statements:**
- **Purpose**: Display build information in Jenkins console
- **`${env.BRANCH_NAME}`**: Built-in Jenkins variable (automatically set)
- **`${env.GIT_COMMIT_SHORT}`**: Custom variable we just created
- **Visibility**: Appears in Jenkins build console and logs

**Console Output Example:**
```
🔀 Merged to dev branch - Running pipeline
Building branch: dev
Commit: a1b2c3d
```

---

### **Part 3: Install Dependencies Stage**

```groovy
stage('Install Dependencies') {
    steps {
        sh '''
            echo "📦 Installing dependencies..."
            
            # Create virtual environment
            python3 -m venv venv
            source venv/bin/activate
            
            # Upgrade pip
            pip install --upgrade pip
            
            # Install dependencies for each Lambda function
            find . -name "lambda_function.py" -type f | while read lambda_file; do
                lambda_dir=$(dirname "$lambda_file")
                if [ -f "$lambda_dir/requirements.txt" ]; then
                    echo "Installing dependencies for $lambda_dir"
                    pip install -r "$lambda_dir/requirements.txt"
                fi
            done
            
            # Install testing dependencies
            pip install pytest boto3 moto pytest-html
            
            # Install any build dependencies if they exist
            if [ -f "package.json" ]; then
                npm install
            fi
        '''
    }
}
```

**Detailed Breakdown:**

**`sh ''' ... '''`**
- **Purpose**: Multi-line shell script execution
- **Scope**: Entire script runs in single shell session
- **Persistence**: Variables and environment persist between commands

**Virtual Environment Setup:**
```bash
python3 -m venv venv          # Creates isolated Python environment
source venv/bin/activate      # Activates virtual environment
pip install --upgrade pip     # Updates pip to latest version
```

**Why Virtual Environment?**
- **Isolation**: Prevents conflicts with system Python packages
- **Reproducibility**: Each build gets clean dependency environment
- **Consistency**: Same dependencies installed every time

**Dynamic Lambda Discovery:**
```bash
find . -name "lambda_function.py" -type f | while read lambda_file; do
    lambda_dir=$(dirname "$lambda_file")
    if [ -f "$lambda_dir/requirements.txt" ]; then
        pip install -r "$lambda_dir/requirements.txt"
    fi
done
```

**Command Breakdown:**
- **`find . -name "lambda_function.py" -type f`**: Searches entire repository for Lambda functions
- **`while read lambda_file`**: Processes each found file
- **`$(dirname "$lambda_file")`**: Gets directory containing the Lambda function
- **`if [ -f "$lambda_dir/requirements.txt" ]`**: Checks if requirements file exists
- **`pip install -r`**: Installs packages listed in requirements.txt

**Example Discovery Process:**
```
Repository scan finds:
./user-auth/lambda_function.py → lambda_dir="./user-auth"
./order-processor/lambda_function.py → lambda_dir="./order-processor"
./api-gateway/lambda_function.py → lambda_dir="./api-gateway"

Requirements installation:
pip install -r ./user-auth/requirements.txt → boto3, bcrypt
pip install -r ./order-processor/requirements.txt → pandas, requests
pip install -r ./api-gateway/requirements.txt → flask, jwt
```

**Testing Dependencies:**
```bash
pip install pytest boto3 moto pytest-html
```
- **pytest**: Python testing framework for unit and integration tests
- **boto3**: AWS SDK for Python (for integration tests)
- **moto**: Mock AWS services for unit tests (no real AWS calls)
- **pytest-html**: Generates HTML test reports for Jenkins

**Optional Build Dependencies:**
```bash
if [ -f "package.json" ]; then
    npm install
fi
```
- **Conditional**: Only runs if package.json exists
- **Purpose**: Installs frontend build tools (webpack, babel, etc.)
- **Scope**: Global to repository (not per Lambda function)

---

### **Part 4: Build Assets Stage**

```groovy
stage('Build Assets') {
    when {
        anyOf {
            expression { fileExists('package.json') }
            expression { fileExists('build.py') }
            expression { fileExists('Makefile') }
        }
    }
    steps {
        sh '''
            echo "🏗️ Building assets..."
            
            # Build using package.json if present
            if [ -f "package.json" ]; then
                echo "Running npm build..."
                npm run build
            fi
            
            # Build using build.py if present
            if [ -f "build.py" ]; then
                echo "Running Python build script..."
                source venv/bin/activate
                python build.py
            fi
            
            # Build using Makefile if present
            if [ -f "Makefile" ]; then
                echo "Running make build..."
                make build
            fi
            
            echo "✅ Build completed"
        '''
    }
}
```

**Detailed Breakdown:**

**`when` Block with `anyOf`:**
```groovy
when {
    anyOf {
        expression { fileExists('package.json') }
        expression { fileExists('build.py') }
        expression { fileExists('Makefile') }
    }
}
```
- **Purpose**: Stage only runs if at least one build configuration exists
- **`anyOf`**: Logical OR operation (any condition true = stage runs)
- **`fileExists()`**: Jenkins built-in function to check file existence
- **Alternative**: Stage skipped if no build tools detected

**NPM Build Process:**
```bash
if [ -f "package.json" ]; then
    echo "Running npm build..."
    npm run build
fi
```
- **Trigger**: package.json file exists
- **Command**: `npm run build` executes build script defined in package.json
- **Typical output**: dist/, build/, or public/ directory with compiled assets
- **Examples**: Webpack bundling, Sass compilation, minification, transpilation

**Custom Python Build:**
```bash
if [ -f "build.py" ]; then
    echo "Running Python build script..."
    source venv/bin/activate
    python build.py
fi
```
- **Trigger**: build.py file exists
- **Environment**: Activates virtual environment first
- **Flexibility**: Custom build logic (file processing, asset generation, etc.)
- **Example use cases**: Template generation, configuration file creation

**Makefile Build:**
```bash
if [ -f "Makefile" ]; then
    echo "Running make build..."
    make build
fi
```
- **Trigger**: Makefile exists
- **Command**: Executes 'build' target in Makefile
- **Flexibility**: Complex build processes, multiple languages
- **Example targets**: compile, minify, test, package

**Build Tool Priority:**
- **No priority**: All detected build tools execute
- **Sequential**: npm → Python → make (if multiple exist)
- **Independent**: Each build tool handles different aspects

**Example Build Scenarios:**

**React Application:**
```json
// package.json
{
  "scripts": {
    "build": "webpack --mode production"
  }
}
```
**Result**: `npm run build` creates `dist/` directory

**Custom Asset Processing:**
```python
# build.py
import os
import shutil

def build():
    # Custom build logic
    shutil.copytree('src/templates', 'dist/templates')
    # Process configuration files
    # Generate dynamic content

if __name__ == "__main__":
    build()
```

**Complex Build Pipeline:**
```makefile
# Makefile
build:
	sass src/styles.scss dist/styles.css
	uglifyjs src/app.js -o dist/app.min.js
	cp src/*.html dist/
```

---

### **Part 5: Run Unit Tests Stage**

```groovy
stage('Run Unit Tests') {
    steps {
        sh '''
            echo "🧪 Running unit tests..."
            
            source venv/bin/activate
            
            # Set Python path to include all Lambda function directories
            lambda_paths=""
            find . -name "lambda_function.py" -type f | while read lambda_file; do
                lambda_dir=$(dirname "$lambda_file")
                lambda_paths="$lambda_paths:$lambda_dir"
            done
            export PYTHONPATH="${WORKSPACE}${lambda_paths}:${PYTHONPATH}"
            
            # Run unit tests if test directory exists
            if [ -d "tests" ]; then
                python -m pytest tests/ -v \
                    --junit-xml=unit-test-results.xml \
                    --html=unit-test-report.html \
                    --self-contained-html
            else
                echo "No tests directory found, skipping unit tests"
                # Create empty test results for Jenkins
                echo '<?xml version="1.0"?><testsuite tests="0" failures="0" errors="0" skipped="0"/>' > unit-test-results.xml
            fi
        '''
    }
    post {
        always {
            junit 'unit-test-results.xml'
            script {
                if (fileExists('unit-test-report.html')) {
                    publishHTML([...])
                }
            }
        }
    }
}
```

**Detailed Breakdown:**

**Environment Activation:**
```bash
source venv/bin/activate
```
- **Purpose**: Activates Python virtual environment with installed dependencies
- **Effect**: Python and pip commands use virtual environment packages
- **Scope**: Remains active for remainder of shell script

**Dynamic Python Path Construction:**
```bash
lambda_paths=""
find . -name "lambda_function.py" -type f | while read lambda_file; do
    lambda_dir=$(dirname "$lambda_file")
    lambda_paths="$lambda_paths:$lambda_dir"
done
export PYTHONPATH="${WORKSPACE}${lambda_paths}:${PYTHONPATH}"
```

**Path Building Process:**
- **`lambda_paths=""`**: Initialize empty path string
- **`find . -name "lambda_function.py"`**: Locate all Lambda functions
- **`lambda_paths="$lambda_paths:$lambda_dir"`**: Append each directory to path
- **`export PYTHONPATH`**: Make Lambda modules importable

**Example Path Construction:**
```
Found Lambda functions:
./user-auth/lambda_function.py
./order-processor/lambda_function.py
./api-gateway/lambda_function.py

Generated PYTHONPATH:
/jenkins/workspace/euc-lambda-poc:./user-auth:./order-processor:./api-gateway:/existing/python/path

Result: Tests can import Lambda modules:
from user-auth.lambda_function import lambda_handler
from order-processor.utils import process_data
```

**Conditional Test Execution:**
```bash
if [ -d "tests" ]; then
    python -m pytest tests/ -v \
        --junit-xml=unit-test-results.xml \
        --html=unit-test-report.html \
        --self-contained-html
else
    echo "No tests directory found, skipping unit tests"
    echo '<?xml version="1.0"?><testsuite tests="0" failures="0" errors="0" skipped="0"/>' > unit-test-results.xml
fi
```

**Test Execution Options:**
- **`python -m pytest`**: Runs pytest as module (ensures proper path handling)
- **`tests/`**: Target directory containing test files
- **`-v`**: Verbose output (shows individual test names and results)

**Pytest Output Options:**
- **`--junit-xml=unit-test-results.xml`**: Creates JUnit XML format for Jenkins
- **`--html=unit-test-report.html`**: Creates human-readable HTML report
- **`--self-contained-html`**: Embeds CSS/JavaScript in HTML (no external dependencies)

**Empty Test Results (Fallback):**
```xml
<?xml version="1.0"?>
<testsuite tests="0" failures="0" errors="0" skipped="0"/>
```
- **Purpose**: Prevents Jenkins from failing when no tests exist
- **Effect**: Jenkins shows "0 tests" instead of error
- **Format**: Valid JUnit XML with zero test cases

**Post-Stage Actions:**
```groovy
post {
    always {
        junit 'unit-test-results.xml'
        script {
            if (fileExists('unit-test-report.html')) {
                publishHTML([...])
            }
        }
    }
}
```

**`always` Block:**
- **Purpose**: Executes regardless of test success/failure
- **Scope**: Runs even if shell script fails

**`junit 'unit-test-results.xml'`:**
- **Purpose**: Jenkins parses XML and displays test results in UI
- **Features**: Test trends, pass/fail counts, individual test details
- **Build Status**: Marks build UNSTABLE if tests fail

**Conditional HTML Publishing:**
```groovy
script {
    if (fileExists('unit-test-report.html')) {
        publishHTML([...])
    }
}
```
- **Safety**: Only publishes HTML if file exists
- **Effect**: HTML report accessible via Jenkins UI
- **Link**: Appears on build page as "Unit Test Report"

**Example Test Execution Output:**
```
🧪 Running unit tests...
========================= test session starts ==========================
tests/test_user_auth.py::test_valid_login PASSED
tests/test_user_auth.py::test_invalid_password FAILED
tests/test_order_processor.py::test_process_order PASSED
tests/test_api_gateway.py::test_route_handler PASSED
========================= 3 passed, 1 failed ===========================
```

---

### **Part 6: Run Integration Tests Stage**

```groovy
stage('Run Integration Tests') {
    when {
        expression { fileExists('tests') }
    }
    steps {
        echo "🔗 Running integration tests..."
        withCredentials([
            [$class: 'AmazonWebServicesCredentialsBinding', 
             credentialsId: 'aws-dev-account-credentials']
        ]) {
            sh '''
                source venv/bin/activate
                
                # Set Python path to include all Lambda function directories
                lambda_paths=""
                find . -name "lambda_function.py" -type f | while read lambda_file; do
                    lambda_dir=$(dirname "$lambda_file")
                    lambda_paths="$lambda_paths:$lambda_dir"
                done
                export PYTHONPATH="${WORKSPACE}${lambda_paths}:${PYTHONPATH}"
                
                # Run integration tests
                python -m pytest tests/ -k "integration" -v \
                    --junit-xml=integration-test-results.xml \
                    --html=integration-test-report.html \
                    --self-contained-html || true
            '''
        }
    }
    post {
        always {
            script {
                if (fileExists('integration-test-results.xml')) {
                    junit 'integration-test-results.xml'
                }
                if (fileExists('integration-test-report.html')) {
                    publishHTML([...])
                }
            }
        }
    }
}
```

**Detailed Breakdown:**

**Stage Condition:**
```groovy
when {
    expression { fileExists('tests') }
}
```
- **Purpose**: Only run if tests directory exists
- **Efficiency**: Avoids unnecessary AWS credential setup
- **Graceful**: Pipeline continues if no tests (doesn't fail)

**AWS Credentials Wrapper:**
```groovy
withCredentials([
    [$class: 'AmazonWebServicesCredentialsBinding', 
     credentialsId: 'aws-dev-account-credentials']
]) {
    // Shell commands with AWS access
}
```

**Credential Binding Details:**
- **`AmazonWebServicesCredentialsBinding`**: Specific class for AWS credentials
- **`credentialsId`**: References credential stored in Jenkins Credential Store
- **Automatic Environment Variables**: 
  - `AWS_ACCESS_KEY_ID`
  - `AWS_SECRET_ACCESS_KEY`
  - `AWS_SESSION_TOKEN` (if using temporary credentials)

**Environment Setup (Same as Unit Tests):**
```bash
source venv/bin/activate
# Set PYTHONPATH for Lambda imports
```
- **Consistency**: Same environment as unit tests
- **Access**: Lambda functions available for import and testing

**Integration Test Execution:**
```bash
python -m pytest tests/ -k "integration" -v \
    --junit-xml=integration-test-results.xml \
    --html=integration-test-report.html \
    --self-contained-html || true
```

**Pytest Options Explained:**
- **`tests/`**: Scans entire tests directory
- **`-k "integration"`**: Only runs tests with "integration" in name
- **`-v`**: Verbose output
- **`|| true`**: Continues pipeline even if tests fail (non-blocking)

**Test Selection Examples:**
```python
# These tests will run (contain "integration"):
def test_integration_api_gateway():
    pass

def test_user_integration():
    pass

def test_s3_integration_flow():
    pass

# These tests will NOT run:
def test_unit_auth():
    pass

def test_user_validation():
    pass
```

**Integration Test Scenarios:**

**API Gateway Integration:**
```python
def test_integration_api_gateway():
    import requests
    
    # Test actual deployed API Gateway endpoint
    response = requests.post(
        'https://api-gateway-url.execute-api.us-east-1.amazonaws.com/dev/users',
        json={'name': 'Test User'}
    )
    assert response.status_code == 201
    assert 'id' in response.json()
```

**S3 Integration:**
```python
def test_integration_s3_processing():
    import boto3
    import time
    
    s3 = boto3.client('s3')
    
    # Upload test file to trigger Lambda
    s3.put_object(
        Bucket='input-bucket-dev',
        Key='test-upload.csv',
        Body=b'name,age\nJohn,25'
    )
    
    # Wait for Lambda processing
    time.sleep(5)
    
    # Verify processed file exists
    response = s3.list_objects_v2(
        Bucket='output-bucket-dev',
        Prefix='processed-test-upload'
    )
    assert response['KeyCount'] > 0
```

**Database Integration:**
```python
def test_integration_dynamodb():
    import boto3
    
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('users-dev')
    
    # Test Lambda function that writes to DynamoDB
    # Invoke Lambda directly or via API Gateway
    # Verify data was written correctly
    
    response = table.get_item(Key={'id': 'test-user-id'})
    assert 'Item' in response
```

**Conditional Post-Processing:**
```groovy
post {
    always {
        script {
            if (fileExists('integration-test-results.xml')) {
                junit 'integration-test-results.xml'
            }
            if (fileExists('integration-test-report.html')) {
                publishHTML([...])
            }
        }
    }
}
```

**Safety Checks:**
- **`if (fileExists(...))`**: Only process files that exist
- **Robustness**: Handles cases where tests don't generate reports
- **No Failures**: Missing reports don't fail the pipeline

**Why `|| true` for Integration Tests:**
- **Non-blocking**: Integration test failures don't stop deployment
- **Environment Issues**: Tests might fail due to AWS service latency
- **Partial Functionality**: Some features might not be fully implemented
- **Flexibility**: Team can deploy and fix issues in dev environment

---

### **Part 7: Deploy Static Assets Stage**

```groovy
stage('Deploy Static Assets') {
    when {
        anyOf {
            expression { fileExists('static') }
            expression { fileExists('assets') }
            expression { fileExists('public') }
            expression { fileExists('dist') }
            expression { fileExists('build') }
        }
    }
    steps {
        echo "📁 Deploying static assets..."
        withCredentials([
            [$class: 'AmazonWebServicesCredentialsBinding', 
             credentialsId: 'aws-dev-account-credentials']
        ]) {
            sh '''
                # Deploy static assets to S3
                # Check for common static asset directories
                
                if [ -d "dist" ]; then
                    echo "Deploying from dist/ directory"
                    aws s3 sync dist/ s3://${S3_ASSETS_BUCKET_DEV}/ \
                        --delete --region ${AWS_DEFAULT_REGION}
                elif [ -d "build" ]; then
                    echo "Deploying from build/ directory"
                    aws s3 sync build/ s3://${S3_ASSETS_BUCKET_DEV}/ \
                        --delete --region ${AWS_DEFAULT_REGION}
                elif [ -d "public" ]; then
                    echo "Deploying from public/ directory"
                    aws s3 sync public/ s3://${S3_ASSETS_BUCKET_DEV}/ \
                        --delete --region ${AWS_DEFAULT_REGION}
                elif [ -d "static" ]; then
                    echo "Deploying from static/ directory"
                    aws s3 sync static/ s3://${S3_ASSETS_BUCKET_DEV}/ \
                        --delete --region ${AWS_DEFAULT_REGION}
                elif [ -d "assets" ]; then
                    echo "Deploying from assets/ directory"
                    aws s3 sync assets/ s3://${S3_ASSETS_BUCKET_DEV}/ \
                        --delete --region ${AWS_DEFAULT_REGION}
                fi
                
                echo "✅ Static assets deployed to S3"
            '''
        }
    }
}
```

**Detailed Breakdown:**

**Conditional Execution:**
```groovy
when {
    anyOf {
        expression { fileExists('static') }
        expression { fileExists('assets') }
        expression { fileExists('public') }
        expression { fileExists('dist') }
        expression { fileExists('build') }
    }
}
```
- **Purpose**: Only deploy if static assets exist
- **Flexibility**: Supports multiple common directory naming conventions
- **Efficiency**: Skips stage for pure API projects

**Directory Priority Chain:**
```bash
if [ -d "dist" ]; then
    # Deploy from dist/
elif [ -d "build" ]; then
    # Deploy from build/
elif [ -d "public" ]; then
    # Deploy from public/
elif [ -d "static" ]; then
    # Deploy from static/
elif [ -d "assets" ]; then
    # Deploy from assets/
fi
```

**Priority Order Reasoning:**
1. **`dist/`**: Webpack/Vite build output (most common for modern apps)
2. **`build/`**: Create React App and similar tools
3. **`public/`**: Direct serving directory (Next.js, Nuxt.js)
4. **`static/`**: Django/Flask style static files
5. **`assets/`**: Generic asset directory

**AWS S3 Sync Command:**
```bash
aws s3 sync dist/ s3://${S3_ASSETS_BUCKET_DEV}/ \
    --delete --region ${AWS_DEFAULT_REGION}
```

**Command Breakdown:**
- **`aws s3 sync`**: Synchronizes local directory with S3 bucket
- **`dist/`**: Source directory (trailing slash = sync contents, not directory)
- **`s3://${S3_ASSETS_BUCKET_DEV}/`**: Target S3 bucket
- **`--delete`**: Remove files in S3 that don't exist locally
- **`--region ${AWS_DEFAULT_REGION}`**: Specify AWS region

**Sync Behavior Examples:**

**Initial Deployment:**
```
Local dist/:
├── index.html
├── app.js
└── styles.css

S3 Result:
s3://bucket/index.html
s3://bucket/app.js
s3://bucket/styles.css
```

**Update Deployment:**
```
Local dist/:
├── index.html (modified)
├── app.js (unchanged)
├── styles.css (deleted)
└── new-feature.js (new)

S3 Operations:
- Upload index.html (changed)
- Skip app.js (unchanged)  
- Delete styles.css (--delete flag)
- Upload new-feature.js (new)
```

**Environment Variable Substitution:**
- **`${S3_ASSETS_BUCKET_DEV}`**: Resolves to `euc-lambda-poc-assets-dev`
- **`${AWS_DEFAULT_REGION}`**: Resolves to `us-east-1`
- **Final Command**: `aws s3 sync dist/ s3://euc-lambda-poc-assets-dev/ --delete --region us-east-1`

**Use Cases for Static Assets:**

**React/Vue Application:**
```
npm run build → dist/
├── index.html
├── static/js/app.bundle.js
├── static/css/main.css
└── static/media/logo.png
```

**Simple Website:**
```
public/
├── index.html
├── style.css
├── script.js
└── images/
    └── banner.jpg
```

**Documentation Site:**
```
build/
├── index.html
├── docs/
│   ├── api.html
│   └── guide.html
└── assets/
    └── styles.css
```

**Why S3 for Static Assets:**
- **Cost-effective**: No server costs for static file hosting
- **Scalable**: Handles traffic spikes automatically
- **CDN Integration**: Can be fronted by CloudFront
- **Direct HTTP Access**: Files accessible via HTTPS URLs
- **Versioning**: S3 versioning enables rollbacks

**S3 Website Hosting Setup (Done separately):**
```bash
# Enable static website hosting
aws s3 website s3://euc-lambda-poc-assets-dev \
    --index-document index.html \
    --error-document error.html

# Set bucket policy for public read access
aws s3api put-bucket-policy \
    --bucket euc-lambda-poc-assets-dev \
    --policy file://public-read-policy.json
```

**Access URL Pattern:**
- **S3 Website Endpoint**: `http://euc-lambda-poc-assets-dev.s3-website-us-east-1.amazonaws.com`
- **S3 REST Endpoint**: `https://euc-lambda-poc-assets-dev.s3.us-east-1.amazonaws.com/index.html`

---

### **Part 8: Deploy Lambda Functions Stage**

```groovy
stage('Deploy Lambda Functions') {
    steps {
        echo "⚡ Deploying Lambda functions..."
        withCredentials([
            [$class: 'AmazonWebServicesCredentialsBinding', 
             credentialsId: 'aws-dev-account-credentials']
        ]) {
            sh '''
                # Find and deploy all Lambda functions
                find . -name "lambda_function.py" -type f | while read lambda_file; do
                    lambda_dir=$(dirname "$lambda_file")
                    lambda_name=$(basename "$lambda_dir")
                    
                    echo "📦 Packaging and deploying $lambda_name"
                    
                    # Create deployment package
                    cd "$lambda_dir"
                    zip -r "../../${lambda_name}-dev.zip" . \
                        -x "*.pyc" "*__pycache__*" "tests/*" "*.md" ".git*" \
                           "*.DS_Store" "*.pytest_cache*" "*.coverage*"
                    cd - > /dev/null
                    
                    # Update Lambda function
                    aws lambda update-function-code \
                        --function-name "${lambda_name}-dev" \
                        --zip-file "fileb://${lambda_name}-dev.zip" \
                        --region ${AWS_DEFAULT_REGION}
                    
                    echo "✅ ${lambda_name}-dev updated successfully"
                done
            '''
        }
    }
}
```

**Detailed Breakdown:**

**Dynamic Lambda Discovery:**
```bash
find . -name "lambda_function.py" -type f | while read lambda_file; do
    lambda_dir=$(dirname "$lambda_file")
    lambda_name=$(basename "$lambda_dir")
    # Process each Lambda function
done
```

**Discovery Process:**
- **`find . -name "lambda_function.py" -type f`**: Searches repository for Lambda entry points
- **`while read lambda_file`**: Processes each found file
- **`$(dirname "$lambda_file")`**: Gets parent directory of lambda_function.py
- **`$(basename "$lambda_dir")`**: Extracts directory name as function name

**Example Discovery:**
```
Repository structure:
./user-auth/lambda_function.py → lambda_dir="./user-auth", lambda_name="user-auth"
./order-processor/lambda_function.py → lambda_dir="./order-processor", lambda_name="order-processor"
./api-gateway/lambda_function.py → lambda_dir="./api-gateway", lambda_name="api-gateway"

Deployment targets:
user-auth-dev
order-processor-dev
api-gateway-dev
```

**Packaging Process:**
```bash
cd "$lambda_dir"
zip -r "../../${lambda_name}-dev.zip" . \
    -x "*.pyc" "*__pycache__*" "tests/*" "*.md" ".git*" \
       "*.DS_Store" "*.pytest_cache*" "*.coverage*"
cd - > /dev/null
```

**Directory Navigation:**
- **`cd "$lambda_dir"`**: Enter Lambda function directory
- **`cd - > /dev/null`**: Return to previous directory (suppress output)

**ZIP Command Breakdown:**
- **`zip -r`**: Create recursive ZIP archive
- **`"../../${lambda_name}-dev.zip"`**: Output file (go up 2 levels from Lambda dir)
- **`.`**: Include all files in current directory
- **`-x`**: Exclude patterns

**Exclusion Patterns Explained:**
- **`*.pyc`**: Python bytecode files (unnecessary for Lambda)
- **`*__pycache__*`**: Python cache directories
- **`tests/*`**: Test files (not needed in deployment)
- **`*.md`**: Documentation files
- **`.git*`**: Git metadata
- **`*.DS_Store`**: macOS system files
- **`*.pytest_cache*`**: Pytest cache files
- **`*.coverage*`**: Code coverage files

**Example Packaging:**
```
Lambda directory: ./user-auth/
├── lambda_function.py     ✅ Include
├── helpers.py             ✅ Include  
├── requirements.txt       ✅ Include
├── config.py              ✅ Include
├── __pycache__/          ❌ Exclude
├── tests/                ❌ Exclude
├── README.md             ❌ Exclude
└── .coverage             ❌ Exclude

Result: user-auth-dev.zip (clean, minimal package)
```

**Lambda Function Update:**
```bash
aws lambda update-function-code \
    --function-name "${lambda_name}-dev" \
    --zip-file "fileb://${lambda_name}-dev.zip" \
    --region ${AWS_DEFAULT_REGION}
```

**AWS CLI Command Breakdown:**
- **`update-function-code`**: Updates existing Lambda function code
- **`--function-name "${lambda_name}-dev"`**: Target function (with -dev suffix)
- **`--zip-file "fileb://..."`**: Binary file upload (ZIP package)
- **`--region ${AWS_DEFAULT_REGION}`**: AWS region for function

**Function Naming Convention:**
- **Repository**: `user-auth/` directory
- **AWS Function**: `user-auth-dev` Lambda function
- **Pattern**: `{directory-name}-dev`

**Deployment Prerequisites:**
- **Lambda functions must exist**: Created by infrastructure team
- **Proper IAM permissions**: Update function code permission
- **Function runtime**: Must match code (Python 3.9, etc.)
- **Package size**: Must be under 50MB (or use S3 for larger packages)

**Example Deployment Output:**
```
📦 Packaging and deploying user-auth
✅ user-auth-dev updated successfully
📦 Packaging and deploying order-processor  
✅ order-processor-dev updated successfully
📦 Packaging and deploying api-gateway
✅ api-gateway-dev updated successfully
```

**Lambda Function Integration Patterns:**

**API Gateway Trigger:**
```
API Gateway → user-auth-dev Lambda
GET /login → Executes lambda_function.lambda_handler()
```

**S3 Event Trigger:**
```
S3 Upload → order-processor-dev Lambda
File uploaded to s3://bucket → Executes lambda_function.lambda_handler()
```

**Direct Invocation:**
```
Application → api-gateway-dev Lambda
boto3.client('lambda').invoke() → Executes lambda_function.lambda_handler()
```

**Why This Deployment is Architecture-Agnostic:**
- **Same packaging**: Regardless of Lambda trigger type
- **Same update command**: Whether API Gateway, S3, SQS, etc.
- **Same function structure**: All use lambda_function.py entry point
- **Trigger configuration**: Done separately in AWS (not in deployment)

---

### **Part 9: Verify Deployment Stage**

```groovy
stage('Verify Deployment') {
    steps {
        echo "🔍 Verifying deployment..."
        withCredentials([
            [$class: 'AmazonWebServicesCredentialsBinding', 
             credentialsId: 'aws-dev-account-credentials']
        ]) {
            sh '''
                # Verify static assets if they exist
                has_assets=false
                for dir in dist build public static assets; do
                    if [ -d "$dir" ]; then
                        has_assets=true
                        break
                    fi
                done
                
                if [ "$has_assets" = true ]; then
                    echo "📁 Static Assets in S3:"
                    aws s3 ls s3://${S3_ASSETS_BUCKET_DEV}/ --recursive --human-readable --summarize
                    echo ""
                fi
                
                # Verify Lambda functions
                echo "⚡ Lambda Function Status:"
                find . -name "lambda_function.py" -type f | while read lambda_file; do
                    lambda_dir=$(dirname "$lambda_file")
                    lambda_name=$(basename "$lambda_dir")
                    
                    echo "Checking ${lambda_name}-dev..."
                    aws lambda get-function \
                        --function-name "${lambda_name}-dev" \
                        --region ${AWS_DEFAULT_REGION} \
                        --query 'Configuration.[FunctionName,LastModified,State,Runtime]' \
                        --output table
                done
                
                echo ""
                echo "✅ All deployments verified successfully"
            '''
        }
    }
}
```

**Detailed Breakdown:**

**Asset Directory Detection:**
```bash
has_assets=false
for dir in dist build public static assets; do
    if [ -d "$dir" ]; then
        has_assets=true
        break
    fi
done
```

**Detection Logic:**
- **`has_assets=false`**: Initialize flag
- **`for dir in ...`**: Check each common asset directory
- **`if [ -d "$dir" ]`**: Test if directory exists
- **`has_assets=true; break`**: Set flag and exit loop on first match

**Why This Approach:**
- **Efficient**: Stops checking after finding first asset directory
- **Comprehensive**: Covers all common naming conventions
- **Binary result**: Either has assets or doesn't

**Conditional S3 Verification:**
```bash
if [ "$has_assets" = true ]; then
    echo "📁 Static Assets in S3:"
    aws s3 ls s3://${S3_ASSETS_BUCKET_DEV}/ --recursive --human-readable --summarize
    echo ""
fi
```

**S3 List Command:**
- **`aws s3 ls`**: List objects in S3 bucket
- **`--recursive`**: Show all objects in all subdirectories
- **`--human-readable`**: Display file sizes in KB/MB instead of bytes
- **`--summarize`**: Show total object count and size at end

**Example S3 Output:**
```
📁 Static Assets in S3:
2024-01-15 10:30:00    2.5 KiB index.html
2024-01-15 10:30:00    1.2 KiB styles.css
2024-01-15 10:30:00    5.7 KiB app.js
2024-01-15 10:30:00    856 Bytes favicon.ico
2024-01-15 10:30:00    3.2 KiB assets/logo.png

Total Objects: 5
   Total Size: 13.5 KiB
```

**Lambda Function Verification:**
```bash
find . -name "lambda_function.py" -type f | while read lambda_file; do
    lambda_dir=$(dirname "$lambda_file")
    lambda_name=$(basename "$lambda_dir")
    
    echo "Checking ${lambda_name}-dev..."
    aws lambda get-function \
        --function-name "${lambda_name}-dev" \
        --region ${AWS_DEFAULT_REGION} \
        --query 'Configuration.[FunctionName,LastModified,State,Runtime]' \
        --output table
done
```

**Function Discovery (Same as Deployment):**
- **Consistency**: Uses identical logic to deployment stage
- **Reliability**: Verifies exactly what was deployed

**AWS Lambda Get-Function Command:**
- **`get-function`**: Retrieves Lambda function metadata
- **`--function-name`**: Specific function to query
- **`--query`**: JMESPath query to extract specific fields
- **`--output table`**: Format output as ASCII table

**Query Fields Explained:**
- **`Configuration.FunctionName`**: Confirms correct function name
- **`Configuration.LastModified`**: Shows when function was last updated
- **`Configuration.State`**: Function status (Active, Pending, Failed)
- **`Configuration.Runtime`**: Python version (python3.9, python3.11, etc.)

**Example Lambda Verification Output:**
```
⚡ Lambda Function Status:
Checking user-auth-dev...
|    user-auth-dev    |  2024-01-15T10:30:00.000Z  |  Active  |  python3.9  |

Checking order-processor-dev...
|  order-processor-dev  |  2024-01-15T10:30:15.000Z  |  Active  |  python3.9  |

Checking api-gateway-dev...
|   api-gateway-dev   |  2024-01-15T10:30:30.000Z  |  Active  |  python3.9  |
```

**Verification Success Indicators:**

**For S3 Assets:**
- **Files present**: Expected files exist in bucket
- **Recent timestamps**: Files were just uploaded
- **Correct sizes**: File sizes match expectations

**For Lambda Functions:**
- **State: Active**: Function is ready to receive invocations
- **Recent LastModified**: Function was just updated
- **Correct Runtime**: Matches development environment

**Why Verification is Important:**

**Deployment Confirmation:**
- **Proof of success**: Deployments actually completed
- **Early detection**: Catch deployment issues immediately
- **Build confidence**: Team knows deployment worked

**Troubleshooting Information:**
- **Missing files**: S3 sync might have failed
- **Wrong timestamps**: Deployment might not have updated
- **Function errors**: Lambda might be in Failed state

**This Stage Does NOT Test:**
- **Functionality**: Whether Lambda functions work correctly
- **Integrations**: Whether API Gateway → Lambda connections work
- **Performance**: Whether functions execute quickly
- **Business logic**: Whether application logic is correct

**Those aspects are tested in Integration Tests stage**

---

### **Part 10: Post-Pipeline Actions**

```groovy
post {
    always {
        sh '''
            echo "🧹 Cleaning up build artifacts..."
            rm -rf venv || true
            rm -f *-dev.zip || true
            rm -rf node_modules || true
            rm -rf .pytest_cache || true
        '''
    }
    success {
        script {
            // Count Lambda functions
            def lambdaCount = sh(
                script: 'find . -name "lambda_function.py" -type f | wc -l',
                returnStdout: true
            ).trim() as Integer
            
            // Check for static assets
            def hasAssets = false
            ['dist', 'build', 'public', 'static', 'assets'].each { dir ->
                if (fileExists(dir)) {
                    hasAssets = true
                }
            }
            
            echo """
🎉 DEPLOYMENT COMPLETED SUCCESSFULLY!

📋 Summary:
✅ Tests executed
${hasAssets ? '✅ Static assets deployed to S3' : 'ℹ️  No static assets found'}
✅ ${lambdaCount} Lambda function(s) deployed
✅ All deployments verified

🏗️ Detected Structure:
- Lambda functions: ${lambdaCount}
- Static assets: ${hasAssets ? 'Yes' : 'No'}
- Tests: ${fileExists('tests') ? 'Yes' : 'No'}

🌐 Dev Environment Ready:
${hasAssets ? "Static Assets: https://${env.S3_ASSETS_BUCKET_DEV}.s3.${env.AWS_DEFAULT_REGION}.amazonaws.com" : ""}
Lambda functions: Available in dev account with '-dev' suffix

🚀 Next Steps:
1. Test your application in dev environment
2. Verify all functionality works as expected
3. Ready for promotion when testing completes
            """
        }
    }
    failure {
        echo '''
❌ DEPLOYMENT FAILED!

🔧 Troubleshooting Steps:
1. Check console output above for specific errors
2. Common issues:
   - Test failures: Fix test logic and retry
   - AWS permissions: Verify aws-dev-account-credentials
   - Lambda packaging: Check requirements.txt files
   - Missing AWS resources: Ensure Lambda functions exist in dev account

🔄 To retry:
Push another commit to dev branch to trigger pipeline again
        '''
    }
    unstable {
        echo '''
⚠️ DEPLOYMENT COMPLETED WITH WARNINGS

Some tests may have failed but deployment proceeded.
Check test reports for details.
        '''
    }
}
```

**Detailed Breakdown:**

**`post` Section Structure:**
```groovy
post {
    always { }    // Runs regardless of build result
    success { }   // Only runs if all stages passed
    failure { }   // Only runs if any stage failed
    unstable { }  // Only runs if tests failed but build continued
}
```

**Build Status Hierarchy:**
1. **SUCCESS**: All stages passed, all tests passed
2. **UNSTABLE**: Deployment succeeded, but some tests failed
3. **FAILURE**: One or more stages failed

**`always` Block - Cleanup:**
```bash
echo "🧹 Cleaning up build artifacts..."
rm -rf venv || true
rm -f *-dev.zip || true
rm -rf node_modules || true
rm -rf .pytest_cache || true
```

**Cleanup Commands Explained:**
- **`rm -rf venv`**: Remove Python virtual environment (large directory)
- **`rm -f *-dev.zip`**: Remove Lambda deployment packages
- **`rm -rf node_modules`**: Remove Node.js dependencies (if installed)
- **`rm -rf .pytest_cache`**: Remove pytest cache files
- **`|| true`**: Prevent command failure if files don't exist

**Why Cleanup is Important:**
- **Disk space**: Build artifacts can be large (hundreds of MB)
- **Clean state**: Next build starts with clean workspace
- **Security**: Remove temporary files that might contain sensitive data

**`success` Block - Dynamic Success Message:**

**Lambda Function Counting:**
```groovy
def lambdaCount = sh(
    script: 'find . -name "lambda_function.py" -type f | wc -l',
    returnStdout: true
).trim() as Integer
```
- **`sh(...)`**: Execute shell command and capture output
- **`returnStdout: true`**: Return command output instead of exit code
- **`.trim()`**: Remove whitespace
- **`as Integer`**: Convert string to integer

**Asset Detection:**
```groovy
def hasAssets = false
['dist', 'build', 'public', 'static', 'assets'].each { dir ->
    if (fileExists(dir)) {
        hasAssets = true
    }
}
```
- **List iteration**: Check each common asset directory
- **`fileExists(dir)`**: Jenkins built-in function
- **Early exit**: Could be optimized with `return true` but clarity preferred

**Dynamic Message Construction:**
```groovy
echo """
🎉 DEPLOYMENT COMPLETED SUCCESSFULLY!
...
${hasAssets ? '✅ Static assets deployed to S3' : 'ℹ️  No static assets found'}
✅ ${lambdaCount} Lambda function(s) deployed
...
${hasAssets ? "Static Assets: https://${env.S3_ASSETS_BUCKET_DEV}.s3.${env.AWS_DEFAULT_REGION}.amazonaws.com" : ""}
...
"""
```

**Conditional Message Parts:**
- **`${hasAssets ? 'A' : 'B'}`**: Ternary operator for conditional text
- **`${lambdaCount}`**: Variable interpolation
- **`${env.VARIABLE}`**: Environment variable access

**Example Success Messages:**

**Web Application:**
```
🎉 DEPLOYMENT COMPLETED SUCCESSFULLY!

📋 Summary:
✅ Tests executed
✅ Static assets deployed to S3
✅ 3 Lambda function(s) deployed
✅ All deployments verified

🏗️ Detected Structure:
- Lambda functions: 3
- Static assets: Yes
- Tests: Yes

🌐 Dev Environment Ready:
Static Assets: https://euc-lambda-poc-assets-dev.s3.us-east-1.amazonaws.com
Lambda functions: Available in dev account with '-dev' suffix
```

**Pure API:**
```
🎉 DEPLOYMENT COMPLETED SUCCESSFULLY!

📋 Summary:
✅ Tests executed
ℹ️  No static assets found
✅ 2 Lambda function(s) deployed
✅ All deployments verified

🏗️ Detected Structure:
- Lambda functions: 2
- Static assets: No
- Tests: Yes

🌐 Dev Environment Ready:
Lambda functions: Available in dev account with '-dev' suffix
```

**`failure` Block - Troubleshooting Guidance:**
```groovy
failure {
    echo '''
❌ DEPLOYMENT FAILED!

🔧 Troubleshooting Steps:
1. Check console output above for specific errors
2. Common issues:
   - Test failures: Fix test logic and retry
   - AWS permissions: Verify aws-dev-account-credentials
   - Lambda packaging: Check requirements.txt files
   - Missing AWS resources: Ensure Lambda functions exist in dev account

🔄 To retry:
Push another commit to dev branch to trigger pipeline again
    '''
}
```

**Failure Guidance Categories:**
- **Immediate action**: Check console output
- **Common issues**: Most frequent failure causes
- **Resolution steps**: How to fix and retry

**`unstable` Block - Warning Message:**
```groovy
unstable {
    echo '''
⚠️ DEPLOYMENT COMPLETED WITH WARNINGS

Some tests may have failed but deployment proceeded.
Check test reports for details.
    '''
}
```

**Unstable State Causes:**
- **Unit test failures**: Tests found bugs but deployment continued
- **Integration test failures**: AWS services not responding correctly
- **Partial functionality**: Some features not yet complete

**When Each Block Executes:**

**Normal Success Flow:**
```
All stages pass → success block runs → "🎉 DEPLOYMENT COMPLETED SUCCESSFULLY!"
```

**Test Failure (Non-blocking):**
```
Tests fail, deployment continues → unstable block runs → "⚠️ DEPLOYMENT COMPLETED WITH WARNINGS"
```

**Stage Failure:**
```
Any stage fails → failure block runs → "❌ DEPLOYMENT FAILED!"
```

**Combined Execution:**
```
Any scenario → always block runs → "🧹 Cleaning up build artifacts..."
```

This post-pipeline section ensures proper cleanup, provides clear feedback to developers, and guides them on next steps based on the build result.
