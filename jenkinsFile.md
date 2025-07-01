## Simplified Pipeline - Only Run on Dev Branch

### **Updated Strategy**

**What happens:**
- **Feature branch pushes**: Pipeline does NOT run
- **Merge to dev branch**: Pipeline runs tests + deployment
- **Faster development**: No waiting for tests on every feature commit

### **Updated Jenkinsfile**

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
                    
                    # Install frontend dependencies (if using build tools)
                    if [ -f "frontend/package.json" ]; then
                        cd frontend
                        npm install
                        cd ..
                    fi
                '''
            }
        }
        
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
    }
    
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
}
```

---

## **What This Pipeline Does**

### **Trigger Condition**
```groovy
when {
    branch 'dev'
}
```
**Only runs when code is pushed to dev branch** - feature branches are completely ignored.

### **Pipeline Flow**
1. **Developer works on feature branch** → No pipeline runs
2. **Developer merges feature → dev** → Pipeline triggers immediately
3. **Pipeline runs full cycle**: Test → Deploy → Verify
4. **Dev environment updated** → Ready for testing

---

## **Developer Workflow**

### **Step 1: Feature Development**
```bash
# Developer creates and works on feature
git checkout -b feature/user-login
# ... make changes ...
git push origin feature/user-login
# ✅ No pipeline runs - fast development cycle
```

### **Step 2: Merge to Dev**
```bash
# Create PR and merge feature → dev
git checkout dev
git merge feature/user-login
git push origin dev
# 🚀 Pipeline triggers automatically
```

### **Step 3: Pipeline Execution**
```
🔀 Merged to dev branch - Running pipeline
📦 Installing dependencies...
🏗️ Building frontend...
🧪 Running unit tests...
🔗 Running integration tests...
📁 Deploying frontend to S3...
⚡ Deploying backend Lambda functions...
🔍 Verifying deployment...
🎉 DEV DEPLOYMENT COMPLETED SUCCESSFULLY!
```

### **Step 4: Cleanup**
```bash
# Delete feature branch after successful merge
git branch -d feature/user-login
git push origin --delete feature/user-login
```

---

## **Key Benefits**

**Faster Development:**
- No waiting for tests on every feature commit
- Only test when ready to deploy

**Focused Testing:**
- Tests run on integration-ready code
- Catches issues before dev deployment

**Immediate Deployment:**
- Dev environment always reflects latest merged code
- Quick feedback loop for integrated features

**Clean Workflow:**
- Feature branches for development
- Dev branch for testing and deployment
- No unnecessary pipeline runs

This approach gives you the fastest possible development cycle while ensuring that only tested, integrated code gets deployed to your dev environment.
