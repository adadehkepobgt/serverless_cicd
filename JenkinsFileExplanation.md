## Detailed Explanation of Each Part of the Groovy Script

### **Part 1: Pipeline Declaration and Global Configuration**

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
```

**Detailed Breakdown:**

**`pipeline {`**
- Declares this is a Jenkins Declarative Pipeline
- Provides structured syntax for CI/CD workflow

**`agent any`**
- Tells Jenkins this pipeline can run on ANY available worker node
- Jenkins will automatically assign an available executor
- Alternative: `agent { label 'linux' }` for specific node types

**`environment { }`**
- Creates global environment variables available to ALL pipeline stages
- **`AWS_DEFAULT_REGION = 'us-east-1'`**: Sets consistent AWS region for all AWS CLI commands
- **`DEV_ACCOUNT = 'dev-account'`**: Human-readable identifier for development account
- **`S3_FRONTEND_BUCKET_DEV`**: Pre-created S3 bucket name for frontend static files

**`tools { python 'Python3' }`**
- References Python installation configured in Jenkins Global Tool Configuration
- Jenkins automatically adds Python to PATH for all stages
- Must match the name configured in Manage Jenkins → Global Tool Configuration

**`when { branch 'dev' }`**
- **Critical filter**: Pipeline ONLY executes when branch name is exactly 'dev'
- Feature branches (feature/login, feature/api, etc.) are completely ignored
- Prevents unnecessary pipeline runs during development

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
- **SCM = Source Code Management** (Git repository)
- Downloads complete repository to Jenkins workspace
- Uses credentials configured in multibranch pipeline settings
- Automatically checks out the specific branch that triggered the build

**`script { }`**
- Allows imperative Groovy code within declarative pipeline
- Enables complex logic and variable manipulation

**`env.GIT_COMMIT_SHORT = sh(...)`**
- **`sh()`**: Executes shell command on Jenkins agent
- **`git rev-parse --short HEAD`**: Git command to get shortened commit hash
- **`returnStdout: true`**: Captures command output instead of displaying it
- **`.trim()`**: Removes trailing newline characters
- **Result**: "a1b2c3d" instead of full "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0"

**`echo` statements**
- Display build information in Jenkins console output
- **`${env.BRANCH_NAME}`**: Built-in Jenkins variable for current branch
- **`${env.GIT_COMMIT_SHORT}`**: Custom variable we just created

**Example Console Output:**
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
            
            # Install frontend dependencies (if using build tools)
            if [ -f "frontend/package.json" ]; then
                cd frontend
                npm install
                cd ..
            fi
        '''
    }
}
```

**Detailed Breakdown:**

**`sh ''' ... '''`**
- Triple quotes allow multi-line shell scripts
- Entire script executes in single shell session (variables persist)

**Python Virtual Environment:**
```bash
python3 -m venv venv          # Creates isolated Python environment in 'venv' folder
source venv/bin/activate      # Activates virtual environment for this session
pip install --upgrade pip     # Updates pip to latest version
```
- **Why virtual environment?** Prevents conflicts with system Python packages
- **Isolation**: Each build gets clean dependency environment

**Lambda Dependencies Loop:**
```bash
for lambda_dir in backend/lambdas/*/; do
    if [ -f "$lambda_dir/requirements.txt" ]; then
        pip install -r "$lambda_dir/requirements.txt"
    fi
done
```
- **`backend/lambdas/*/`**: Shell glob pattern matching all subdirectories
- **Finds**: `backend/lambdas/user-auth/`, `backend/lambdas/data-processor/`, etc.
- **`if [ -f ... ]`**: Checks if requirements.txt file exists
- **`pip install -r`**: Installs all packages listed in requirements.txt

**Example Directory Scan:**
```
backend/lambdas/user-authentication/requirements.txt → pip install boto3 bcrypt
backend/lambdas/data-processor/requirements.txt → pip install pandas numpy
backend/lambdas/api-handler/requirements.txt → pip install requests flask
```

**Testing Dependencies:**
```bash
pip install pytest boto3 moto pytest-html
```
- **pytest**: Python testing framework
- **boto3**: AWS SDK for integration tests
- **moto**: Mock AWS services for unit tests
- **pytest-html**: Generates HTML test reports

**Frontend Dependencies (Conditional):**
```bash
if [ -f "frontend/package.json" ]; then
    npm install
fi
```
- **Only runs if frontend uses build tools** (React, Vue, etc.)
- **Skips if frontend is pure HTML/CSS/JS**

---

### **Part 4: Build Frontend Stage**

```groovy
stage('Build Frontend') {
    steps {
        sh '''
            echo "🏗️ Building frontend..."
            
            # Build frontend assets if needed
            if [ -f "frontend/package.json" ]; then
                cd frontend
                npm run build
                cd ..
            fi
            
            # Create frontend deployment package
            if [ -d "frontend/dist" ]; then
                cp -r frontend/dist frontend-build/
            elif [ -d "frontend/build" ]; then
                cp -r frontend/build frontend-build/
            else
                # Direct static files
                cp -r frontend/ frontend-build/
            fi
            
            echo "✅ Frontend build completed"
        '''
    }
}
```

**Detailed Breakdown:**

**Build Process (Conditional):**
```bash
if [ -f "frontend/package.json" ]; then
    cd frontend
    npm run build
    cd ..
fi
```
- **Detects build tools**: Only runs if package.json exists
- **`npm run build`**: Executes build script defined in package.json
- **Examples**: Webpack bundling, Sass compilation, minification

**Standardized Output Creation:**
```bash
if [ -d "frontend/dist" ]; then
    cp -r frontend/dist frontend-build/
elif [ -d "frontend/build" ]; then
    cp -r frontend/build frontend-build/
else
    cp -r frontend/ frontend-build/
fi
```

**Logic Flow:**
1. **Check for `frontend/dist/`** (Vue.js, Vite builds)
2. **If not found, check `frontend/build/`** (React, Create React App)
3. **If neither, copy entire frontend/** (static HTML/CSS/JS)
4. **Result**: All scenarios produce `frontend-build/` folder

**Build Scenarios:**

**React Application:**
```
frontend/package.json exists → npm run build → frontend/build/ → frontend-build/
```

**Vue Application:**
```
frontend/package.json exists → npm run build → frontend/dist/ → frontend-build/
```

**Static Files:**
```
No package.json → frontend/ (direct copy) → frontend-build/
```

**Why Standardize?** Later deployment stage always looks for `frontend-build/` regardless of build tool used.

---

### **Part 5: Run Unit Tests Stage**

```groovy
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
```

**Detailed Breakdown:**

**Environment Activation:**
```bash
source venv/bin/activate
export PYTHONPATH="${WORKSPACE}/backend/lambdas:${PYTHONPATH}"
```
- **`source venv/bin/activate`**: Activates Python virtual environment with dependencies
- **`PYTHONPATH`**: Tells Python where to find Lambda function modules
- **`${WORKSPACE}`**: Jenkins built-in variable pointing to build directory
- **Result**: Tests can import Lambda functions: `from user_auth import lambda_handler`

**Pytest Execution:**
```bash
python -m pytest tests/unit/ -v \
    --junit-xml=unit-test-results.xml \
    --html=unit-test-report.html \
    --self-contained-html
```

**Pytest Options Explained:**
- **`tests/unit/`**: Directory containing unit test files
- **`-v`**: Verbose output showing each individual test
- **`--junit-xml=unit-test-results.xml`**: Creates XML report for Jenkins
- **`--html=unit-test-report.html`**: Creates human-readable HTML report
- **`--self-contained-html`**: Embeds CSS/JS in HTML file (no external dependencies)

**Example Test Execution:**
```
tests/unit/test_user_auth.py::test_valid_login PASSED
tests/unit/test_user_auth.py::test_invalid_password FAILED
tests/unit/test_data_processor.py::test_process_csv PASSED
========================= 2 passed, 1 failed =========================
```

**Post Section:**
```groovy
post {
    always {
        junit 'unit-test-results.xml'
        publishHTML([...])
    }
}
```

**`always` block**: Runs regardless of test success/failure

**`junit 'unit-test-results.xml'`**
- Jenkins reads XML file and displays test results in UI
- Shows pass/fail counts, test trends, individual test details
- Marks build as UNSTABLE if tests fail

**`publishHTML([...])`**
- Makes HTML report accessible via Jenkins UI
- **`allowMissing: false`**: Fail if HTML report not generated
- **`alwaysLinkToLastBuild: true`**: Show link on main build page
- **`keepAll: true`**: Archive reports from all builds
- **`reportName: 'Unit Test Report'`**: Display name in Jenkins UI

---

### **Part 6: Run Integration Tests Stage**

```groovy
stage('Run Integration Tests') {
    steps {
        echo "🔗 Running integration tests..."
        withCredentials([
            [$class: 'AmazonWebServicesCredentialsBinding', 
             credentialsId: 'aws-dev-account-credentials']
        ]) {
            sh '''
                source venv/bin/activate
                export PYTHONPATH="${WORKSPACE}/backend/lambdas:${PYTHONPATH}"
                
                # Run integration tests against dev environment
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
            publishHTML([...])
        }
    }
}
```

**Detailed Breakdown:**

**AWS Credentials Binding:**
```groovy
withCredentials([
    [$class: 'AmazonWebServicesCredentialsBinding', 
     credentialsId: 'aws-dev-account-credentials']
]) {
    // Shell commands here have AWS access
}
```
- **`withCredentials`**: Temporarily provides secure credentials
- **`AmazonWebServicesCredentialsBinding`**: Specific class for AWS credentials
- **`credentialsId`**: References credentials stored in Jenkins Credential Store
- **Automatic environment variables**: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`

**Integration Test Execution:**
- **Same pytest command as unit tests** but different directory
- **`tests/integration/`**: Tests that interact with real AWS services
- **Real AWS access**: Tests can connect to RDS, S3, DynamoDB, etc.

**Integration Test Examples:**
```python
def test_lambda_rds_connection():
    # Tests actual database connection in dev account
    
def test_s3_file_upload():
    # Tests uploading to real S3 bucket in dev account
    
def test_api_gateway_endpoint():
    # Tests actual API Gateway endpoints
```

**Why Separate Reports?**
- **Unit tests**: Fast, isolated, mock dependencies
- **Integration tests**: Slower, real AWS services, network dependent
- **Separate analysis**: Can identify if failures are code logic vs infrastructure

---

### **Part 7: Deploy Frontend to S3 Stage**

```groovy
stage('Deploy Frontend to S3') {
    steps {
        echo "📁 Deploying frontend to S3..."
        withCredentials([
            [$class: 'AmazonWebServicesCredentialsBinding', 
             credentialsId: 'aws-dev-account-credentials']
        ]) {
            sh '''
                # Deploy Frontend to S3
                aws s3 sync frontend-build/ s3://${S3_FRONTEND_BUCKET_DEV}/ \
                    --delete \
                    --region ${AWS_DEFAULT_REGION}
                
                echo "✅ Frontend deployed to S3"
            '''
        }
    }
}
```

**Detailed Breakdown:**

**AWS S3 Sync Command:**
```bash
aws s3 sync frontend-build/ s3://${S3_FRONTEND_BUCKET_DEV}/ \
    --delete \
    --region ${AWS_DEFAULT_REGION}
```

**Command Breakdown:**
- **`aws s3 sync`**: Synchronizes local directory with S3 bucket
- **`frontend-build/`**: Source directory (trailing slash = sync contents)
- **`s3://${S3_FRONTEND_BUCKET_DEV}/`**: Target S3 bucket
- **`--delete`**: Remove files in S3 that don't exist locally (clean deployment)
- **`--region`**: Specify AWS region for bucket operation

**Environment Variable Substitution:**
- **`${S3_FRONTEND_BUCKET_DEV}`**: Resolves to `euc-lambda-poc-frontend-dev`
- **`${AWS_DEFAULT_REGION}`**: Resolves to `us-east-1`
- **Final command**: `aws s3 sync frontend-build/ s3://euc-lambda-poc-frontend-dev/ --delete --region us-east-1`

**Sync Behavior:**
```
Local: frontend-build/index.html (modified)
Local: frontend-build/styles.css (unchanged)
Local: frontend-build/new-feature.js (new file)
S3: old-feature.js (no longer exists locally)

Result:
- Upload index.html (changed)
- Skip styles.css (unchanged)
- Upload new-feature.js (new)
- Delete old-feature.js (--delete flag)
```

**Why S3 for Frontend?**
- **Static hosting**: S3 can serve HTML/CSS/JS directly
- **CDN integration**: Can be fronted by CloudFront
- **Cost effective**: No servers needed for static files

---

### **Part 8: Deploy Backend Lambdas Stage**

```groovy
stage('Deploy Backend Lambdas') {
    steps {
        echo "⚡ Deploying backend Lambda functions..."
        withCredentials([
            [$class: 'AmazonWebServicesCredentialsBinding', 
             credentialsId: 'aws-dev-account-credentials']
        ]) {
            sh '''
                # Deploy Backend Lambda Functions
                for lambda_dir in backend/lambdas/*/; do
                    lambda_name=$(basename $lambda_dir)
                    echo "📦 Packaging and deploying $lambda_name"
                    
                    # Create deployment package
                    cd $lambda_dir
                    zip -r ../../${lambda_name}-dev.zip . \
                        -x "*.pyc" "*__pycache__*" "tests/*" "*.md" ".git*"
                    cd ../..
                    
                    # Update Lambda function
                    aws lambda update-function-code \
                        --function-name ${lambda_name}-dev \
                        --zip-file fileb://${lambda_name}-dev.zip \
                        --region ${AWS_DEFAULT_REGION}
                    
                    echo "✅ ${lambda_name}-dev updated successfully"
                done
            '''
        }
    }
}
```

**Detailed Breakdown:**

**Lambda Directory Loop:**
```bash
for lambda_dir in backend/lambdas/*/; do
    lambda_name=$(basename $lambda_dir)
    # Process each Lambda function
done
```
- **`backend/lambdas/*/`**: Shell glob matching all subdirectories
- **`$(basename $lambda_dir)`**: Extracts directory name without path
- **Example**: `backend/lambdas/user-auth/` → `lambda_name="user-auth"`

**Packaging Process:**
```bash
cd $lambda_dir
zip -r ../../${lambda_name}-dev.zip . \
    -x "*.pyc" "*__pycache__*" "tests/*" "*.md" ".git*"
cd ../..
```

**ZIP Command Breakdown:**
- **`cd $lambda_dir`**: Enter specific Lambda function directory
- **`zip -r`**: Create recursive ZIP archive
- **`../../${lambda_name}-dev.zip`**: Output file (go up 2 levels from lambda dir)
- **`.`**: Include all files in current directory
- **`-x`**: Exclude patterns

**Exclusion Patterns:**
- **`*.pyc`**: Python bytecode files (unnecessary for Lambda)
- **`*__pycache__*`**: Python cache directories
- **`tests/*`**: Test files (not needed in deployment)
- **`*.md`**: Documentation files
- **`.git*`**: Git metadata

**Example Packaging:**
```
backend/lambdas/user-auth/
├── lambda_function.py     ✅ Include
├── helpers.py             ✅ Include
├── requirements.txt       ✅ Include
├── __pycache__/          ❌ Exclude
├── tests/                ❌ Exclude
└── README.md             ❌ Exclude

Result: user-auth-dev.zip (clean, minimal package)
```

**Lambda Function Update:**
```bash
aws lambda update-function-code \
    --function-name ${lambda_name}-dev \
    --zip-file fileb://${lambda_name}-dev.zip \
    --region ${AWS_DEFAULT_REGION}
```

**AWS CLI Command Breakdown:**
- **`update-function-code`**: Updates existing Lambda function code
- **`--function-name ${lambda_name}-dev`**: Target function name
- **`--zip-file fileb://`**: Binary file prefix for ZIP upload
- **`--region`**: AWS region for Lambda function

**Example Updates:**
```
user-auth-dev.zip → aws lambda update-function-code --function-name user-auth-dev
data-processor-dev.zip → aws lambda update-function-code --function-name data-processor-dev
api-handler-dev.zip → aws lambda update-function-code --function-name api-handler-dev
```

**Prerequisites:**
- Lambda functions must already exist in AWS (created by infrastructure team)
- Function names must follow pattern: `{lambda-name}-dev`
- Deployment package must be under 50MB (or use S3 for larger packages)

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
                echo "📁 Frontend Files in S3:"
                aws s3 ls s3://${S3_FRONTEND_BUCKET_DEV}/ --recursive --human-readable --summarize
                
                echo ""
                echo "⚡ Lambda Function Status:"
                for lambda_dir in backend/lambdas/*/; do
                    lambda_name=$(basename $lambda_dir)
                    echo "Checking ${lambda_name}-dev..."
                    
                    aws lambda get-function \
                        --function-name ${lambda_name}-dev \
                        --region ${AWS_DEFAULT_REGION} \
                        --query 'Configuration.[FunctionName,LastModified,State,Runtime]' \
                        --output table
                done
                
                echo ""
                echo "🌐 Frontend URL: https://${S3_FRONTEND_BUCKET_DEV}.s3.${AWS_DEFAULT_REGION}.amazonaws.com"
                echo "✅ All deployments verified successfully"
            '''
        }
    }
}
```

**Detailed Breakdown:**

**S3 Verification:**
```bash
aws s3 ls s3://${S3_FRONTEND_BUCKET_DEV}/ --recursive --human-readable --summarize
```

**Command Options:**
- **`ls`**: List objects in S3 bucket
- **`--recursive`**: List all objects in all subdirectories
- **`--human-readable`**: Show file sizes in KB/MB instead of bytes
- **`--summarize`**: Show total count and size at the end

**Example S3 Output:**
```
2024-01-15 10:30:00    2.5 KiB index.html
2024-01-15 10:30:00    1.2 KiB styles.css
2024-01-15 10:30:00    5.7 KiB app.js
2024-01-15 10:30:00    856 Bytes favicon.ico

Total Objects: 4
   Total Size: 10.3 KiB
```

**Lambda Function Verification:**
```bash
aws lambda get-function \
    --function-name ${lambda_name}-dev \
    --region ${AWS_DEFAULT_REGION} \
    --query 'Configuration.[FunctionName,LastModified,State,Runtime]' \
    --output table
```

**Command Breakdown:**
- **`get-function`**: Retrieves Lambda function metadata
- **`--function-name`**: Specific function to check
- **`--query`**: JMESPath query to extract specific fields
- **`--output table`**: Format output as ASCII table

**Query Fields:**
- **`FunctionName`**: Confirms correct function name
- **`LastModified`**: Shows when function was last updated (should be recent)
- **`State`**: Function status (should be "Active")
- **`Runtime`**: Python version (should match your code)

**Example Lambda Output:**
```
|  user-auth-dev  |  2024-01-15T10:30:00.000Z  |  Active  |  python3.9  |
|  data-processor-dev  |  2024-01-15T10:30:15.000Z  |  Active  |  python3.9  |
|  api-handler-dev  |  2024-01-15T10:30:30.000Z  |  Active  |  python3.9  |
```

**Verification Success Indicators:**
- **S3**: All expected files present with recent timestamps
- **Lambda**: All functions in "Active" state with recent "LastModified"
- **URLs**: Frontend accessible via S3 static website URL

---

### **Part 10: Post-Pipeline Actions**

```groovy
post {
    always {
        // Clean up build artifacts
        sh '''
            echo "🧹 Cleaning up build artifacts..."
            rm -rf venv || true
            rm -f *.zip || true
            rm -rf frontend-build || true
            rm -rf node_modules || true
        '''
    }
    success {
        echo '''
🎉 DEV DEPLOYMENT COMPLETED SUCCESSFULLY!

📋 Summary:
✅ Unit tests passed
✅ Integration tests passed  
✅ Frontend deployed to S3
✅ Backend Lambda functions updated
✅ All components verified

🌐 Dev Environment Ready:
Frontend: https://${S3_FRONTEND_BUCKET_DEV}.s3.${AWS_DEFAULT_REGION}.amazonaws.com
Backend: Lambda functions active in dev account

🚀 Next Steps:
1. Test the application in dev environment
2. Verify all features work as expected
3. Ready for CCB review when needed
        '''
    }
    failure {
        echo '''
❌ DEV DEPLOYMENT FAILED!

🔧 Troubleshooting Steps:
1. Check the console output above for specific errors
2. Common issues:
   - Test failures: Fix failing tests and retry
   - AWS permissions: Verify aws-dev-account-credentials
   - Lambda packaging: Check requirements.txt files
   - S3 deployment: Verify bucket exists and permissions

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

**`post` Section:**
- Runs after all stages complete, regardless of success/failure
- Contains conditional blocks based on build result

**`always` Block:**
```bash
rm -rf venv || true
rm -f *.zip || true
rm -rf frontend-build || true
rm -rf node_modules || true
```
- **Cleanup**: Removes temporary files created during build
- **`|| true`**: Prevents command failure if files don't exist
- **Why cleanup?** Prevents workspace bloat and ensures clean subsequent builds

**Files Cleaned:**
- **`venv/`**: Python virtual environment (large, recreated each build)
- **`*.zip`**: Lambda deployment packages (no longer needed)
- **`frontend-build/`**: Temporary frontend build directory
- **`node_modules/`**: NPM dependencies (large, cached separately)

**`success` Block:**
- **Only runs if all stages passed**
- Provides comprehensive success summary
- Shows frontend URL for immediate testing
- Outlines next steps for team

**`failure` Block:**
- **Only runs if any stage failed**
- Provides troubleshooting guidance
- Lists common failure scenarios
- Explains how to retry deployment

**`unstable` Block:**
- **Runs if tests failed but build continued**
- Indicates partial success (deployment succeeded, some tests failed)

**Build Status Hierarchy:**
1. **SUCCESS**: All stages passed, all tests passed
2. **UNSTABLE**: Deployment succeeded, but some tests failed
3. **FAILURE**: One or more stages failed, deployment may not have completed

This post-pipeline section ensures proper cleanup and provides clear feedback to developers about build results and next steps.
