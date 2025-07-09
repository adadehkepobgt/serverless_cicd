pipeline {
    agent any
    
    environment {
        AWS_DEFAULT_REGION = 'ap-northeast-1'
        S3_BUCKET = 'ss-dev-eucm-cicd-apnortheast1-bucket'
        LAMBDA_FUNCTION_NAME = 'ss-dev-eucm-cicd-lambda'
    }
    
    triggers {
        pollSCM('H/2 * * * *')
    }
    
    stages {
        stage('Checkout') {
            when {
                branch 'dev'
            }
            steps {
                echo "🔄 Checking out dev branch..."
                checkout scm
                
                script {
                    env.GIT_COMMIT_SHORT = sh(
                        returnStdout: true,
                        script: 'git rev-parse --short HEAD'
                    ).trim()
                    
                    echo "Building commit: ${env.GIT_COMMIT_SHORT}"
                }
            }
        }
        
        stage('Build Lambda Package') {
            when {
                branch 'dev'
            }
            steps {
                echo "📦 Creating Lambda deployment package..."
                
                sh '''
                    rm -f lambda_deployment.zip
                    
                    zip -r lambda_deployment.zip . \
                        -x "*.git*" \
                        -x "Jenkinsfile" \
                        -x "*.md" \
                        -x "terraform/*" \
                        -x "*.pyc" \
                        -x "__pycache__/*"
                    
                    if [ -f lambda_deployment.zip ]; then
                        echo "✅ Package created successfully"
                        ls -lh lambda_deployment.zip
                    else
                        echo "❌ Failed to create package"
                        exit 1
                    fi
                '''
            }
        }
        
        stage('Upload to S3') {
            when {
                branch 'dev'
            }
            steps {
                echo "☁️ Uploading to S3..."
                
                script {
                    def timestamp = sh(returnStdout: true, script: 'date "+%Y%m%d-%H%M%S"').trim()
                    env.S3_KEY = "lambda_functions/dev-${timestamp}-${env.GIT_COMMIT_SHORT}-lambda.zip"
                }
                
                sh '''
                    aws s3 cp lambda_deployment.zip "s3://${S3_BUCKET}/${S3_KEY}"
                    
                    if aws s3 ls "s3://${S3_BUCKET}/${S3_KEY}"; then
                        echo "✅ Successfully uploaded to S3"
                        echo "S3 Location: s3://${S3_BUCKET}/${S3_KEY}"
                    else
                        echo "❌ Failed to upload to S3"
                        exit 1
                    fi
                '''
            }
        }
        
        stage('Deployment Complete') {
            when {
                branch 'dev'
            }
            steps {
                echo "🎉 Deployment pipeline completed!"
                echo "Package uploaded to: s3://${env.S3_BUCKET}/${env.S3_KEY}"
                echo "Lambda function should be automatically updated via S3 trigger"
            }
        }
    }
    
    post {
        success {
            script {
                if (env.BRANCH_NAME == 'dev') {
                    echo "✅ DEV DEPLOYMENT SUCCESSFUL"
                    echo "📦 Package: s3://${env.S3_BUCKET}/${env.S3_KEY}"
                    echo "🔄 Lambda will be updated automatically via S3 trigger"
                }
            }
        }
        
        failure {
            echo "❌ Pipeline failed on dev branch"
        }
        
        always {
            sh 'rm -f lambda_deployment.zip'
        }
    }
}
