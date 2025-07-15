# 🚀 **Complete Jenkins Pipeline Scripts - Both Approaches**

Here are the complete Jenkins pipeline scripts for both the boto3 and AWS CLI testing approaches:

---

## **📋 Option 1: Pipeline with Python boto3 Testing**

### **`Jenkinsfile` - boto3 Version**
```groovy
pipeline {
    agent any
    
    environment {
        AWS_DEFAULT_REGION = 'ap-northeast-1'
        BUILD_ID = "${BUILD_NUMBER}-${env.GIT_COMMIT?.take(7)}"
        TEST_RESULTS_DIR = 'tests/reports'
        LAMBDA_FUNCTION_PATTERN = 'euc-lambda-poc'
        
        // Test configuration
        TEST_SESSION_ID = "test_${BUILD_ID}_${env.BUILD_TIMESTAMP ?: new Date().format('yyyyMMdd-HHmmss')}"
        PYTHON_PATH = "/usr/bin/python3"
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        skipStagesAfterUnstable()
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo "🔄 Checking out source code..."
                checkout scm
                
                script {
                    // Set build description
                    currentBuild.description = "Lambda Testing Pipeline - Build ${BUILD_ID}"
                }
            }
        }
        
        stage('Setup Build Environment') {
            steps {
                echo "🔧 Setting up build environment..."
                
                script {
                    // Display environment info
                    sh '''
                        echo "=== Build Environment ==="
                        echo "Build ID: ${BUILD_ID}"
                        echo "Git Commit: ${GIT_COMMIT}"
                        echo "Git Branch: ${GIT_BRANCH}"
                        echo "AWS Region: ${AWS_DEFAULT_REGION}"
                        echo "Test Session: ${TEST_SESSION_ID}"
                        echo "=========================="
                    '''
                }
                
                // Create necessary directories
                sh '''
                    mkdir -p ${TEST_RESULTS_DIR}/{unit,integration}
                    mkdir -p tests/{config,events,utils}
                    mkdir -p artifacts
                '''
                
                // Display Python environment
                sh '''
                    echo "🐍 Python Environment:"
                    python3 --version || echo "Python3 not found"
                    pip3 --version || echo "pip3 not found"
                    which python3 || echo "python3 not in PATH"
                '''
            }
        }
        
        stage('Install Dependencies') {
            steps {
                echo "📦 Installing Python dependencies..."
                
                withAWS(region: "${AWS_DEFAULT_REGION}", role: 'arn:aws:iam::623341402407:role/jef-tfe-role') {
                    sh '''
                        # Install boto3 if not already available
                        echo "Checking boto3 installation..."
                        if python3 -c "import boto3" 2>/dev/null; then
                            echo "✅ boto3 already available: $(python3 -c "import boto3; print(boto3.__version__)")"
                        else
                            echo "📦 Installing boto3..."
                            pip3 install boto3 --user --no-warn-script-location
                            
                            # Verify installation
                            if python3 -c "import boto3" 2>/dev/null; then
                                echo "✅ boto3 installed successfully: $(python3 -c "import boto3; print(boto3.__version__)")"
                            else
                                echo "❌ Failed to install boto3"
                                exit 1
                            fi
                        fi
                        
                        # Test AWS connectivity
                        echo "🔐 Testing AWS connectivity..."
                        aws sts get-caller-identity
                        echo "✅ AWS credentials working"
                    '''
                }
            }
        }
        
        stage('Package Lambda Function') {
            steps {
                echo "📦 Packaging Lambda function..."
                
                script {
                    // Create Lambda package
                    sh '''
                        echo "Creating Lambda deployment package..."
                        
                        # Create temporary package directory
                        rm -rf lambda_package
                        mkdir -p lambda_package
                        
                        # Copy Lambda source code
                        if [ -f "lambda_function.py" ]; then
                            cp lambda_function.py lambda_package/
                            echo "✅ Copied lambda_function.py"
                        else
                            echo "⚠️ lambda_function.py not found, using existing deployed function"
                        fi
                        
                        # Copy any additional files (if they exist)
                        if [ -d "src" ]; then
                            cp -r src/* lambda_package/
                            echo "✅ Copied src directory"
                        fi
                        
                        # Create ZIP package
                        cd lambda_package
                        zip -r ../lambda-${BUILD_ID}.zip . -x "*.pyc" "__pycache__/*"
                        cd ..
                        
                        echo "📦 Lambda package created: lambda-${BUILD_ID}.zip"
                        ls -lh lambda-${BUILD_ID}.zip || echo "Package not created, using existing deployment"
                    '''
                }
            }
        }
        
        stage('Deploy to S3') {
            when {
                expression { fileExists('lambda-' + env.BUILD_ID + '.zip') }
            }
            steps {
                echo "☁️ Uploading Lambda package to S3..."
                
                withAWS(region: "${AWS_DEFAULT_REGION}", role: 'arn:aws:iam::623341402407:role/jef-tfe-role') {
                    script {
                        // Upload to S3
                        s3Upload(
                            bucket: 'jef-tfe-poc-lambda',
                            file: "lambda-${BUILD_ID}.zip",
                            path: "lambda-packages/lambda-${BUILD_ID}.zip"
                        )
                        
                        echo "✅ Lambda package uploaded to S3"
                        
                        // Store S3 path for later use
                        env.LAMBDA_S3_KEY = "lambda-packages/lambda-${BUILD_ID}.zip"
                    }
                }
            }
        }
        
        stage('Update Terraform Configuration') {
            when {
                expression { env.LAMBDA_S3_KEY != null }
            }
            steps {
                echo "🏗️ Updating Terraform configuration..."
                
                script {
                    // Update terraform.tfvars or main.tf with new S3 key
                    sh '''
                        if [ -f "terraform.tfvars" ]; then
                            # Update S3 key in terraform.tfvars
                            sed -i "s|lambda_s3_key.*=.*|lambda_s3_key = \\"${LAMBDA_S3_KEY}\\"|g" terraform.tfvars
                            echo "✅ Updated terraform.tfvars with new S3 key"
                            grep "lambda_s3_key" terraform.tfvars || echo "S3 key not found in terraform.tfvars"
                        else
                            echo "⚠️ terraform.tfvars not found, skipping update"
                        fi
                        
                        # Commit changes if needed (optional)
                        git add terraform.tfvars || true
                        git commit -m "Update Lambda S3 key for build ${BUILD_ID}" || echo "No changes to commit"
                    '''
                }
            }
        }
        
        stage('Wait for Terraform Deployment') {
            steps {
                echo "⏳ Waiting for Terraform Enterprise deployment..."
                
                script {
                    // Wait for TFE deployment (simulate or implement actual check)
                    sh '''
                        echo "🏗️ Terraform Enterprise should deploy the updated Lambda function"
                        echo "💡 In a real environment, you might:"
                        echo "   - Trigger TFE run via API"
                        echo "   - Poll TFE for deployment completion"
                        echo "   - Check Lambda function last modified time"
                        
                        # Simulate wait time for deployment
                        echo "⏳ Waiting 30 seconds for deployment to complete..."
                        sleep 30
                        
                        echo "✅ Deployment wait completed"
                    '''
                }
            }
        }
        
        stage('Setup Test Environment') {
            steps {
                echo "🧪 Setting up test environment..."
                
                withAWS(region: "${AWS_DEFAULT_REGION}", role: 'arn:aws:iam::623341402407:role/jef-tfe-role') {
                    sh '''
                        # Verify test framework files
                        echo "📋 Checking test framework files..."
                        
                        if [ ! -f "tests/lambda_tester.py" ]; then
                            echo "❌ tests/lambda_tester.py not found"
                            echo "Please ensure the boto3 testing framework files are committed to the repository"
                            exit 1
                        fi
                        
                        if [ ! -f "tests/test_runner.py" ]; then
                            echo "❌ tests/test_runner.py not found"
                            exit 1
                        fi
                        
                        echo "✅ Test framework files found"
                        
                        # Check test configurations
                        if [ ! -f "tests/config/unit-tests.json" ]; then
                            echo "⚠️ Unit test configuration not found, will create default"
                        fi
                        
                        if [ ! -f "tests/config/integration-tests.json" ]; then
                            echo "⚠️ Integration test configuration not found, will create default"
                        fi
                        
                        # Verify Lambda function exists
                        echo "🔍 Verifying Lambda function..."
                        LAMBDA_FUNCTION=$(aws lambda list-functions --query "Functions[?contains(FunctionName, '${LAMBDA_FUNCTION_PATTERN}')].FunctionName" --output text | head -n1)
                        
                        if [ -z "$LAMBDA_FUNCTION" ]; then
                            echo "❌ No Lambda function found matching '${LAMBDA_FUNCTION_PATTERN}'"
                            exit 1
                        fi
                        
                        echo "✅ Found Lambda function: $LAMBDA_FUNCTION"
                        echo "LAMBDA_FUNCTION_NAME=$LAMBDA_FUNCTION" > lambda_info.env
                    '''
                }
            }
        }
        
        stage('Unit Tests') {
            steps {
                echo "🧪 Running Lambda Unit Tests..."
                
                withAWS(region: "${AWS_DEFAULT_REGION}", role: 'arn:aws:iam::623341402407:role/jef-tfe-role') {
                    script {
                        try {
                            sh '''
                                # Load Lambda function info
                                source lambda_info.env
                                export LAMBDA_FUNCTION_NAME
                                
                                echo "🚀 Starting unit tests for: $LAMBDA_FUNCTION_NAME"
                                echo "Test Session: ${TEST_SESSION_ID}"
                                
                                # Run unit tests
                                python3 tests/test_runner.py unit
                                
                                echo "✅ Unit tests completed successfully"
                            '''
                        } catch (Exception e) {
                            echo "❌ Unit tests failed: ${e.getMessage()}"
                            currentBuild.result = 'UNSTABLE'
                        }
                    }
                }
            }
            post {
                always {
                    // Publish unit test results
                    script {
                        if (fileExists("${TEST_RESULTS_DIR}/unit/junit-results.xml")) {
                            publishTestResults(
                                testResultsPattern: "${TEST_RESULTS_DIR}/unit/junit-results.xml",
                                allowEmptyResults: false
                            )
                            echo "📊 Published unit test results"
                        } else {
                            echo "⚠️ Unit test results not found"
                        }
                    }
                    
                    // Archive unit test artifacts
                    archiveArtifacts(
                        artifacts: "${TEST_RESULTS_DIR}/unit/**/*",
                        allowEmptyArchive: true,
                        fingerprint: true
                    )
                }
                success {
                    echo "✅ Unit tests passed successfully"
                }
                failure {
                    echo "❌ Unit tests failed"
                }
            }
        }
        
        stage('Integration Tests') {
            steps {
                echo "🔗 Running Lambda Integration Tests..."
                
                withAWS(region: "${AWS_DEFAULT_REGION}", role: 'arn:aws:iam::623341402407:role/jef-tfe-role') {
                    script {
                        try {
                            sh '''
                                # Load Lambda function info
                                source lambda_info.env
                                export LAMBDA_FUNCTION_NAME
                                
                                echo "🚀 Starting integration tests for: $LAMBDA_FUNCTION_NAME"
                                
                                # Run integration tests
                                python3 tests/test_runner.py integration
                                
                                echo "✅ Integration tests completed successfully"
                            '''
                        } catch (Exception e) {
                            echo "❌ Integration tests failed: ${e.getMessage()}"
                            currentBuild.result = 'UNSTABLE'
                        }
                    }
                }
            }
            post {
                always {
                    // Publish integration test results
                    script {
                        if (fileExists("${TEST_RESULTS_DIR}/integration/junit-results.xml")) {
                            publishTestResults(
                                testResultsPattern: "${TEST_RESULTS_DIR}/integration/junit-results.xml",
                                allowEmptyResults: false
                            )
                            echo "📊 Published integration test results"
                        } else {
                            echo "⚠️ Integration test results not found"
                        }
                    }
                    
                    // Archive integration test artifacts
                    archiveArtifacts(
                        artifacts: "${TEST_RESULTS_DIR}/integration/**/*",
                        allowEmptyArchive: true,
                        fingerprint: true
                    )
                }
                success {
                    echo "✅ Integration tests passed successfully"
                }
                failure {
                    echo "❌ Integration tests failed"
                }
            }
        }
        
        stage('Generate Test Report') {
            steps {
                echo "📊 Generating comprehensive test report..."
                
                script {
                    sh '''
                        echo "=== RDS Lambda Test Summary ===" > test_summary.txt
                        echo "Build ID: ${BUILD_ID}" >> test_summary.txt
                        echo "Test Session: ${TEST_SESSION_ID}" >> test_summary.txt
                        echo "Timestamp: $(date)" >> test_summary.txt
                        echo "Lambda Function: $(cat lambda_info.env | grep LAMBDA_FUNCTION_NAME | cut -d= -f2)" >> test_summary.txt
                        echo "AWS Region: ${AWS_DEFAULT_REGION}" >> test_summary.txt
                        echo "" >> test_summary.txt
                        
                        # Unit test summary
                        if [ -f "${TEST_RESULTS_DIR}/unit/summary.json" ]; then
                            echo "Unit Tests:" >> test_summary.txt
                            python3 -c "
import json
try:
    with open('${TEST_RESULTS_DIR}/unit/summary.json', 'r') as f:
        data = json.load(f)
    print(f'  Total: {data.get(\"total\", 0)}')
    print(f'  Passed: {data.get(\"passed\", 0)}')
    print(f'  Failed: {data.get(\"failed\", 0)}')
    print(f'  DB Connections: {data.get(\"db_connectivity\", {}).get(\"successful_connections\", 0)}/{data.get(\"total\", 0)}')
    print(f'  Avg Duration: {data.get(\"average_duration_ms\", 0):.0f}ms')
except Exception as e:
    print(f'  Error reading unit test summary: {e}')
" >> test_summary.txt
                        else
                            echo "  Unit Tests: Not completed" >> test_summary.txt
                        fi
                        
                        echo "" >> test_summary.txt
                        
                        # Integration test summary
                        if [ -f "${TEST_RESULTS_DIR}/integration/summary.json" ]; then
                            echo "Integration Tests:" >> test_summary.txt
                            python3 -c "
import json
try:
    with open('${TEST_RESULTS_DIR}/integration/summary.json', 'r') as f:
        data = json.load(f)
    print(f'  Total: {data.get(\"total\", 0)}')
    print(f'  Passed: {data.get(\"passed\", 0)}')
    print(f'  Failed: {data.get(\"failed\", 0)}')
except Exception as e:
    print(f'  Error reading integration test summary: {e}')
" >> test_summary.txt
                        else
                            echo "  Integration Tests: Not completed" >> test_summary.txt
                        fi
                        
                        echo "" >> test_summary.txt
                        echo "=== End Test Summary ===" >> test_summary.txt
                        
                        # Display summary
                        cat test_summary.txt
                    '''
                }
            }
            post {
                always {
                    // Archive test summary
                    archiveArtifacts(
                        artifacts: "test_summary.txt",
                        allowEmptyArchive: true
                    )
                }
            }
        }
    }
    
    post {
        always {
            echo "🧹 Pipeline cleanup..."
            
            // Clean up temporary files
            sh '''
                rm -f lambda_info.env
                rm -rf lambda_package
                rm -f lambda-*.zip
            '''
            
            // Archive all test reports
            archiveArtifacts(
                artifacts: "${TEST_RESULTS_DIR}/**/*",
                allowEmptyArchive: true,
                fingerprint: true
            )
        }
        
        success {
            echo "🎉 Pipeline completed successfully!"
            
            script {
                // Update build description
                currentBuild.description = "✅ Lambda Testing Pipeline - All tests passed - Build ${BUILD_ID}"
                
                // Send success notification (if configured)
                // slackSend(channel: '#deployments', color: 'good', message: "✅ Lambda tests passed for build ${BUILD_ID}")
            }
        }
        
        failure {
            echo "❌ Pipeline failed!"
            
            script {
                currentBuild.description = "❌ Lambda Testing Pipeline - Failed - Build ${BUILD_ID}"
                
                // Send failure notification (if configured)
                // slackSend(channel: '#deployments', color: 'danger', message: "❌ Lambda tests failed for build ${BUILD_ID}")
            }
        }
        
        unstable {
            echo "⚠️ Pipeline unstable - some tests failed"
            
            script {
                currentBuild.description = "⚠️ Lambda Testing Pipeline - Unstable - Build ${BUILD_ID}"
                
                // Send unstable notification (if configured)
                // slackSend(channel: '#deployments', color: 'warning', message: "⚠️ Lambda tests unstable for build ${BUILD_ID}")
            }
        }
    }
}
```

---

## **📋 Option 2: Pipeline with AWS CLI Testing**

### **`Jenkinsfile` - AWS CLI Version**
```groovy
pipeline {
    agent any
    
    environment {
        AWS_DEFAULT_REGION = 'ap-northeast-1'
        BUILD_ID = "${BUILD_NUMBER}-${env.GIT_COMMIT?.take(7)}"
        TEST_RESULTS_DIR = 'tests/reports'
        LAMBDA_FUNCTION_PATTERN = 'euc-lambda-poc'
        
        // Test configuration
        TEST_SESSION_ID = "test_${BUILD_ID}_${env.BUILD_TIMESTAMP ?: new Date().format('yyyyMMdd-HHmmss')}"
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        skipStagesAfterUnstable()
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo "🔄 Checking out source code..."
                checkout scm
                
                script {
                    currentBuild.description = "Lambda Testing Pipeline (AWS CLI) - Build ${BUILD_ID}"
                }
            }
        }
        
        stage('Setup Build Environment') {
            steps {
                echo "🔧 Setting up build environment..."
                
                script {
                    sh '''
                        echo "=== Build Environment ==="
                        echo "Build ID: ${BUILD_ID}"
                        echo "Git Commit: ${GIT_COMMIT}"
                        echo "Git Branch: ${GIT_BRANCH}"
                        echo "AWS Region: ${AWS_DEFAULT_REGION}"
                        echo "Test Session: ${TEST_SESSION_ID}"
                        echo "=========================="
                    '''
                }
                
                // Create necessary directories
                sh '''
                    mkdir -p ${TEST_RESULTS_DIR}/{unit,integration}
                    mkdir -p tests/{config,utils}
                    mkdir -p artifacts
                '''
                
                // Check required tools
                sh '''
                    echo "🔧 Checking required tools..."
                    aws --version || echo "❌ AWS CLI not found"
                    jq --version || echo "❌ jq not found"
                    openssl version || echo "❌ openssl not found"
                '''
            }
        }
        
        stage('Install Dependencies') {
            steps {
                echo "📦 Installing dependencies..."
                
                withAWS(region: "${AWS_DEFAULT_REGION}", role: 'arn:aws:iam::623341402407:role/jef-tfe-role') {
                    sh '''
                        # Install jq if not available
                        if ! command -v jq &> /dev/null; then
                            echo "📦 Installing jq..."
                            if command -v yum &> /dev/null; then
                                sudo yum install -y jq
                            elif command -v apt-get &> /dev/null; then
                                sudo apt-get update && sudo apt-get install -y jq
                            else
                                echo "❌ Cannot install jq automatically"
                                exit 1
                            fi
                        fi
                        
                        echo "✅ jq available: $(jq --version)"
                        
                        # Test AWS connectivity
                        echo "🔐 Testing AWS connectivity..."
                        aws sts get-caller-identity
                        echo "✅ AWS credentials working"
                    '''
                }
            }
        }
        
        stage('Package Lambda Function') {
            steps {
                echo "📦 Packaging Lambda function..."
                
                script {
                    sh '''
                        echo "Creating Lambda deployment package..."
                        
                        # Create temporary package directory
                        rm -rf lambda_package
                        mkdir -p lambda_package
                        
                        # Copy Lambda source code
                        if [ -f "lambda_function.py" ]; then
                            cp lambda_function.py lambda_package/
                            echo "✅ Copied lambda_function.py"
                        else
                            echo "⚠️ lambda_function.py not found, using existing deployed function"
                        fi
                        
                        # Copy any additional files
                        if [ -d "src" ]; then
                            cp -r src/* lambda_package/
                            echo "✅ Copied src directory"
                        fi
                        
                        # Create ZIP package
                        if [ -d "lambda_package" ] && [ "$(ls -A lambda_package)" ]; then
                            cd lambda_package
                            zip -r ../lambda-${BUILD_ID}.zip . -x "*.pyc" "__pycache__/*"
                            cd ..
                            echo "📦 Lambda package created: lambda-${BUILD_ID}.zip"
                            ls -lh lambda-${BUILD_ID}.zip
                        else
                            echo "⚠️ No Lambda source found, using existing deployment"
                        fi
                    '''
                }
            }
        }
        
        stage('Deploy to S3') {
            when {
                expression { fileExists('lambda-' + env.BUILD_ID + '.zip') }
            }
            steps {
                echo "☁️ Uploading Lambda package to S3..."
                
                withAWS(region: "${AWS_DEFAULT_REGION}", role: 'arn:aws:iam::623341402407:role/jef-tfe-role') {
                    script {
                        s3Upload(
                            bucket: 'jef-tfe-poc-lambda',
                            file: "lambda-${BUILD_ID}.zip",
                            path: "lambda-packages/lambda-${BUILD_ID}.zip"
                        )
                        
                        echo "✅ Lambda package uploaded to S3"
                        env.LAMBDA_S3_KEY = "lambda-packages/lambda-${BUILD_ID}.zip"
                    }
                }
            }
        }
        
        stage('Wait for Terraform Deployment') {
            steps {
                echo "⏳ Waiting for Terraform Enterprise deployment..."
                
                script {
                    sh '''
                        echo "🏗️ Terraform Enterprise should deploy the updated Lambda function"
                        echo "⏳ Waiting 30 seconds for deployment to complete..."
                        sleep 30
                        echo "✅ Deployment wait completed"
                    '''
                }
            }
        }
        
        stage('Setup Test Environment') {
            steps {
                echo "🧪 Setting up test environment..."
                
                withAWS(region: "${AWS_DEFAULT_REGION}", role: 'arn:aws:iam::623341402407:role/jef-tfe-role') {
                    sh '''
                        # Make test scripts executable
                        chmod +x tests/run_tests.sh tests/setup_prerequisites.sh
                        
                        # Run prerequisites check
                        echo "🔧 Running prerequisites check..."
                        ./tests/setup_prerequisites.sh
                        
                        echo "✅ Test environment setup completed"
                    '''
                }
            }
        }
        
        stage('Unit Tests') {
            steps {
                echo "🧪 Running Lambda Unit Tests..."
                
                withAWS(region: "${AWS_DEFAULT_REGION}", role: 'arn:aws:iam::623341402407:role/jef-tfe-role') {
                    script {
                        try {
                            sh '''
                                echo "🚀 Starting unit tests..."
                                echo "Test Session: ${TEST_SESSION_ID}"
                                
                                # Run unit tests
                                ./tests/run_tests.sh unit
                                
                                echo "✅ Unit tests completed successfully"
                            '''
                        } catch (Exception e) {
                            echo "❌ Unit tests failed: ${e.getMessage()}"
                            currentBuild.result = 'UNSTABLE'
                        }
                    }
                }
            }
            post {
                always {
                    script {
                        if (fileExists("${TEST_RESULTS_DIR}/unit/junit-results.xml")) {
                            publishTestResults(
                                testResultsPattern: "${TEST_RESULTS_DIR}/unit/junit-results.xml",
                                allowEmptyResults: false
                            )
                            echo "📊 Published unit test results"
                        } else {
                            echo "⚠️ Unit test results not found"
                        }
                    }
                    
                    archiveArtifacts(
                        artifacts: "${TEST_RESULTS_DIR}/unit/**/*",
                        allowEmptyArchive: true,
                        fingerprint: true
                    )
                }
            }
        }
        
        stage('Integration Tests') {
            steps {
                echo "🔗 Running Lambda Integration Tests..."
                
                withAWS(region: "${AWS_DEFAULT_REGION}", role: 'arn:aws:iam::623341402407:role/jef-tfe-role') {
                    script {
                        try {
                            sh '''
                                echo "🚀 Starting integration tests..."
                                
                                # Run integration tests
                                ./tests/run_tests.sh integration
                                
                                echo "✅ Integration tests completed successfully"
                            '''
                        } catch (Exception e) {
                            echo "❌ Integration tests failed: ${e.getMessage()}"
                            currentBuild.result = 'UNSTABLE'
                        }
                    }
                }
            }
            post {
                always {
                    script {
                        if (fileExists("${TEST_RESULTS_DIR}/integration/junit-results.xml")) {
                            publishTestResults(
                                testResultsPattern: "${TEST_RESULTS_DIR}/integration/junit-results.xml",
                                allowEmptyResults: false
                            )
                            echo "📊 Published integration test results"
                        }
                    }
                    
                    archiveArtifacts(
                        artifacts: "${TEST_RESULTS_DIR}/integration/**/*",
                        allowEmptyArchive: true,
                        fingerprint: true
                    )
                }
            }
        }
        
        stage('Comprehensive Test Run') {
            when {
                expression { 
                    currentBuild.result == null || currentBuild.result == 'SUCCESS' 
                }
            }
            steps {
                echo "🎯 Running comprehensive test suite..."
                
                withAWS(region: "${AWS_DEFAULT_REGION}", role: 'arn:aws:iam::623341402407:role/jef-tfe-role') {
                    script {
                        try {
                            sh '''
                                echo "🚀 Running all tests together for final validation..."
                                
                                # Run all tests
                                ./tests/run_tests.sh all
                                
                                echo "✅ Comprehensive tests completed successfully"
                            '''
                        } catch (Exception e) {
                            echo "❌ Comprehensive tests failed: ${e.getMessage()}"
                            currentBuild.result = 'UNSTABLE'
                        }
                    }
                }
            }
        }
        
        stage('Generate Test Report') {
            steps {
                echo "📊 Generating test report..."
                
                script {
                    sh '''
                        echo "=== RDS Lambda Test Summary (AWS CLI) ===" > test_summary.txt
                        echo "Build ID: ${BUILD_ID}" >> test_summary.txt
                        echo "Test Session: ${TEST_SESSION_ID}" >> test_summary.txt
                        echo "Timestamp: $(date)" >> test_summary.txt
                        echo "AWS Region: ${AWS_DEFAULT_REGION}" >> test_summary.txt
                        echo "" >> test_summary.txt
                        
                        # Check for test results
                        if [ -f "${TEST_RESULTS_DIR}/unit/summary.json" ]; then
                            echo "Unit Tests:" >> test_summary.txt
                            jq -r '"  Total: " + (.total|tostring) + "\\n  Passed: " + (.passed|tostring) + "\\n  Failed: " + (.failed|tostring)' ${TEST_RESULTS_DIR}/unit/summary.json >> test_summary.txt 2>/dev/null || echo "  Unit test summary not available" >> test_summary.txt
                        else
                            echo "Unit Tests: Not completed" >> test_summary.txt
                        fi
                        
                        echo "" >> test_summary.txt
                        
                        if [ -f "${TEST_RESULTS_DIR}/integration/summary.json" ]; then
                            echo "Integration Tests:" >> test_summary.txt
                            jq -r '"  Total: " + (.total|tostring) + "\\n  Passed: " + (.passed|tostring) + "\\n  Failed: " + (.failed|tostring)' ${TEST_RESULTS_DIR}/integration/summary.json >> test_summary.txt 2>/dev/null || echo "  Integration test summary not available" >> test_summary.txt
                        else
                            echo "Integration Tests: Not completed" >> test_summary.txt
                        fi
                        
                        echo "" >> test_summary.txt
                        echo "=== End Test Summary ===" >> test_summary.txt
                        
                        # Display summary
                        cat test_summary.txt
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts(
                        artifacts: "test_summary.txt",
                        allowEmptyArchive: true
                    )
                }
            }
        }
    }
    
    post {
        always {
            echo "🧹 Pipeline cleanup..."
            
            sh '''
                rm -rf lambda_package
                rm -f lambda-*.zip
                # Clean up temporary test files
                rm -f tests/utils/lambda_* tests/utils/invoke_* || true
            '''
            
            archiveArtifacts(
                artifacts: "${TEST_RESULTS_DIR}/**/*",
                allowEmptyArchive: true,
                fingerprint: true
            )
        }
        
        success {
            echo "🎉 Pipeline completed successfully!"
            script {
                currentBuild.description = "✅ Lambda Testing Pipeline (AWS CLI) - All tests passed - Build ${BUILD_ID}"
            }
        }
        
        failure {
            echo "❌ Pipeline failed!"
            script {
                currentBuild.description = "❌ Lambda Testing Pipeline (AWS CLI) - Failed - Build ${BUILD_ID}"
            }
        }
        
        unstable {
            echo "⚠️ Pipeline unstable - some tests failed"
            script {
                currentBuild.description = "⚠️ Lambda Testing Pipeline (AWS CLI) - Unstable - Build ${BUILD_ID}"
            }
        }
    }
}
```

---

## **📋 Simplified Pipeline (Minimal Version)**

### **`Jenkinsfile-simple` - For Quick Implementation**
```groovy
pipeline {
    agent any
    
    environment {
        AWS_DEFAULT_REGION = 'ap-northeast-1'
        BUILD_ID = "${BUILD_NUMBER}-${env.GIT_COMMIT?.take(7)}"
        LAMBDA_FUNCTION_PATTERN = 'euc-lambda-poc'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Setup') {
            steps {
                sh 'mkdir -p tests/reports/{unit,integration}'
            }
        }
        
        stage('Test Lambda Function') {
            steps {
                withAWS(region: "${AWS_DEFAULT_REGION}", role: 'arn:aws:iam::623341402407:role/jef-tfe-role') {
                    script {
                        // Choose your testing approach:
                        
                        // Option 1: Python boto3
                        sh '''
                            pip3 install boto3 --user || echo "boto3 already installed"
                            python3 tests/test_runner.py unit
                            python3 tests/test_runner.py integration
                        '''
                        
                        // Option 2: AWS CLI (comment out option 1 and uncomment this)
                        /*
                        sh '''
                            chmod +x tests/run_tests.sh
                            ./tests/run_tests.sh unit
                            ./tests/run_tests.sh integration
                        '''
                        */
                    }
                }
            }
            post {
                always {
                    publishTestResults testResultsPattern: 'tests/reports/**/*.xml'
                    archiveArtifacts artifacts: 'tests/reports/**/*', allowEmptyArchive: true
                }
            }
        }
    }
    
    post {
        success {
            echo "✅ All Lambda tests passed!"
        }
        failure {
            echo "❌ Lambda tests failed!"
        }
    }
}
```

---

## **🔧 Pipeline Configuration Tips**

### **Environment Variables to Set:**
```groovy
environment {
    // Required
    AWS_DEFAULT_REGION = 'ap-northeast-1'
    BUILD_ID = "${BUILD_NUMBER}-${env.GIT_COMMIT?.take(7)}"
    
    // Testing specific
    LAMBDA_FUNCTION_PATTERN = 'euc-lambda-poc'  // Adjust to your function naming
    TEST_RESULTS_DIR = 'tests/reports'
    
    // Optional - Database testing
    TEST_DB_NAME = 'your_test_database'          // If different from prod
    RDS_ENDPOINT = 'your-dev-rds-endpoint'       // If need to override
}
```

### **Jenkins Plugins Required:**
- **AWS Steps Plugin** ✅ (for `withAWS`, `s3Upload`)
- **JUnit Plugin** ✅ (for `publishTestResults`)
- **Pipeline Plugin** ✅ (for pipeline syntax)
- **Build Timeout Plugin** (for `timeout`)

### **IAM Permissions Needed:**
Your `jef-tfe-role` needs:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "lambda:ListFunctions",
                "lambda:GetFunction",
                "lambda:InvokeFunction"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject"
            ],
            "Resource": "arn:aws:s3:::jef-tfe-poc-lambda/*"
        }
    ]
}
```

---

## **🚀 Pipeline Execution Flow**

### **Complete Flow:**
```
1. 🔄 Checkout Code
2. 🔧 Setup Environment
3. 📦 Install Dependencies (boto3 or jq)
4. 📦 Package Lambda Function
5. ☁️ Upload to S3
6. ⏳ Wait for Terraform Deployment
7. 🧪 Run Unit Tests
8. 🔗 Run Integration Tests
9. 📊 Generate Reports
10. 🧹 Cleanup
```

### **Quick Test-Only Flow:**
```
1. 🔄 Checkout Code
2. 🔧 Setup Test Environment
3. 🧪 Run Tests Against Existing Lambda
4. 📊 Publish Results
```

---

## **📋 How to Choose Which Pipeline to Use**

### **Use Full Pipeline (boto3 or AWS CLI) when:**
- ✅ You want comprehensive CI/CD with testing
- ✅ You deploy Lambda functions regularly
- ✅ You need detailed test reporting
- ✅ You want to catch issues before production

### **Use Simplified Pipeline when:**
- ✅ You just want to add testing to existing pipeline
- ✅ Lambda function is already deployed
- ✅ You want quick implementation
- ✅ Minimal changes to existing workflow

**Choose boto3 if you want robust error handling and extensibility, choose AWS CLI if you want simple setup with existing tools!** 🎯
