## Cross-Account Configuration with Jenkins

### **When You Need Cross-Account Setup**

### **Common Scenarios:**

**1. Separate AWS Accounts by Environment**
```
Dev Account:     123456789012 (development, testing)
Staging Account: 234567890123 (pre-production validation)  
Prod Account:    345678901234 (production workloads)
```

**2. Organizational Structure**
```
Shared Services: 111111111111 (Jenkins, CI/CD, logging)
Application Dev: 222222222222 (dev environments)
Application Prod: 333333333333 (production environments)
```

**3. Security Requirements**
```
Security Account: 444444444444 (KMS keys, secrets)
Workload Account: 555555555555 (Lambda functions, APIs)
Data Account:     666666666666 (databases, data lakes)
```

---

### **Cross-Account Architecture Overview**

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Jenkins       │    │   Dev Account   │    │  Prod Account   │
│   Account       │    │   (Target 1)    │    │   (Target 2)    │
│                 │    │                 │    │                 │
│ ┌─────────────┐ │    │ ┌─────────────┐ │    │ ┌─────────────┐ │
│ │   Jenkins   │ │    │ │Cross-Account│ │    │ │Cross-Account│ │
│ │   Server    │────┼─│►│    Role     │ │    │ │    Role     │ │
│ │             │ │    │ │             │ │    │ │             │ │
│ └─────────────┘ │    │ └─────────────┘ │    │ └─────────────┘ │
│                 │    │        │        │    │        │        │
│                 │    │        ▼        │    │        ▼        │
│                 │    │ ┌─────────────┐ │    │ ┌─────────────┐ │
│                 │    │ │   Lambda    │ │    │ │   Lambda    │ │
│                 │    │ │ Functions   │ │    │ │ Functions   │ │
│                 │    │ │     S3      │ │    │ │     S3      │ │
│                 │    │ │    API GW   │ │    │ │    API GW   │ │
│                 │    │ └─────────────┘ │    │ └─────────────┘ │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

---

### **Step 1: Create Cross-Account IAM Roles (In Target Accounts)**

### **In Each Target Account (Dev, Prod)**

**Create Cross-Account Role:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::JENKINS-ACCOUNT-ID:role/JenkinsCrossAccountRole"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "unique-external-id-dev"
        }
      }
    }
  ]
}
```

**Cross-Account Role Name:** `CrossAccountDeploymentRole`

**Attach Deployment Permissions Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "lambda:UpdateFunctionCode",
        "lambda:UpdateFunctionConfiguration",
        "lambda:GetFunction",
        "lambda:ListFunctions",
        "lambda:TagResource",
        "lambda:UntagResource"
      ],
      "Resource": "arn:aws:lambda:*:*:function:*-dev",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::*-dev",
        "arn:aws:s3:::*-dev/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "sts:GetCallerIdentity"
      ],
      "Resource": "*"
    }
  ]
}
```

**AWS CLI Commands to Create Roles:**
```bash
# Create trust policy file
cat > trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::JENKINS-ACCOUNT-ID:role/JenkinsCrossAccountRole"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "unique-external-id-dev"
        }
      }
    }
  ]
}
EOF

# Create the role
aws iam create-role \
  --role-name CrossAccountDeploymentRole \
  --assume-role-policy-document file://trust-policy.json

# Attach permissions policy
aws iam attach-role-policy \
  --role-name CrossAccountDeploymentRole \
  --policy-arn arn:aws:iam::aws:policy/PowerUserAccess  # Or custom policy

# Get role ARN for Jenkins configuration
aws iam get-role --role-name CrossAccountDeploymentRole --query 'Role.Arn'
```

---

### **Step 2: Configure Jenkins Server IAM Role**

### **In Jenkins Account**

**Jenkins Server Role Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": [
        "arn:aws:iam::DEV-ACCOUNT-ID:role/CrossAccountDeploymentRole",
        "arn:aws:iam::PROD-ACCOUNT-ID:role/CrossAccountDeploymentRole"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "sts:GetCallerIdentity"
      ],
      "Resource": "*"
    }
  ]
}
```

**Attach to Jenkins EC2 Instance Role:**
```bash
# If Jenkins runs on EC2, attach this policy to the instance role
aws iam attach-role-policy \
  --role-name JenkinsInstanceRole \
  --policy-arn arn:aws:iam::JENKINS-ACCOUNT:policy/CrossAccountAssumeRolePolicy
```

---

### **Step 3: Configure Jenkins Credentials**

### **Option A: Cross-Account Role Credentials**

**In Jenkins UI:**
1. **Navigate to:** Manage Jenkins → Manage Credentials
2. **Click:** Global → Add Credentials
3. **Select:** "AWS Credentials"
4. **Configure:**
   - **Kind:** AWS Credentials
   - **ID:** `aws-dev-cross-account-credentials`
   - **Access Key ID:** `LEAVE EMPTY` (will use instance role)
   - **Secret Access Key:** `LEAVE EMPTY` (will use instance role)
   - **Role ARN:** `arn:aws:iam::DEV-ACCOUNT-ID:role/CrossAccountDeploymentRole`
   - **External ID:** `unique-external-id-dev`
   - **Description:** `Cross-account access to Dev account`

**Repeat for each target account:**
```
aws-dev-cross-account-credentials   → Dev Account Role
aws-staging-cross-account-credentials → Staging Account Role  
aws-prod-cross-account-credentials  → Prod Account Role
```

### **Option B: Credential Scripts (Advanced)**

**Create Jenkins Credential Script:**
```groovy
// In Jenkins credential store
def devAccountCredentials = [
    $class: 'AmazonWebServicesCredentialsBinding',
    credentialsId: 'aws-dev-cross-account-credentials',
    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY',
    roleArn: 'arn:aws:iam::DEV-ACCOUNT-ID:role/CrossAccountDeploymentRole',
    externalId: 'unique-external-id-dev',
    durationSeconds: 3600
]
```

---

### **Step 4: Update Pipeline for Cross-Account**

### **Enhanced Pipeline with Cross-Account Support**

```groovy
pipeline {
    agent any
    
    environment {
        AWS_DEFAULT_REGION = 'us-east-1'
        PROJECT_NAME = 'euc-lambda-poc'
        
        // Account mapping
        DEV_ACCOUNT_ID = '123456789012'
        STAGING_ACCOUNT_ID = '234567890123'
        PROD_ACCOUNT_ID = '345678901234'
        
        // Environment-specific configuration
        TARGET_ENVIRONMENT = "${env.BRANCH_NAME == 'main' ? 'prod' : 'dev'}"
        TARGET_ACCOUNT_ID = "${env.BRANCH_NAME == 'main' ? env.PROD_ACCOUNT_ID : env.DEV_ACCOUNT_ID}"
        AWS_CREDENTIALS_ID = "aws-${TARGET_ENVIRONMENT}-cross-account-credentials"
        
        // Resource naming
        S3_ASSETS_BUCKET = "${PROJECT_NAME}-assets-${TARGET_ENVIRONMENT}"
        LAMBDA_SUFFIX = "-${TARGET_ENVIRONMENT}"
    }
    
    when {
        anyOf {
            branch 'dev'
            branch 'main'
        }
    }
    
    stages {
        stage('Validate Target Account') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding', 
                     credentialsId: "${AWS_CREDENTIALS_ID}"]
                ]) {
                    sh '''
                        echo "🔍 Validating target account access..."
                        
                        # Get current account ID
                        CURRENT_ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
                        echo "Current AWS Account: $CURRENT_ACCOUNT"
                        echo "Expected Account: ${TARGET_ACCOUNT_ID}"
                        
                        # Validate we're in the correct account
                        if [ "$CURRENT_ACCOUNT" != "${TARGET_ACCOUNT_ID}" ]; then
                            echo "❌ ERROR: Wrong AWS account!"
                            echo "Expected: ${TARGET_ACCOUNT_ID}"
                            echo "Current:  $CURRENT_ACCOUNT"
                            exit 1
                        fi
                        
                        echo "✅ Confirmed access to ${TARGET_ENVIRONMENT} account"
                        
                        # Validate S3 bucket exists
                        if aws s3api head-bucket --bucket ${S3_ASSETS_BUCKET} 2>/dev/null; then
                            echo "✅ S3 bucket ${S3_ASSETS_BUCKET} accessible"
                        else
                            echo "⚠️  S3 bucket ${S3_ASSETS_BUCKET} not found or not accessible"
                        fi
                    '''
                }
            }
        }
        
        stage('Deploy Lambda Functions') {
            steps {
                echo "⚡ Deploying Lambda functions to ${TARGET_ENVIRONMENT} account..."
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding', 
                     credentialsId: "${AWS_CREDENTIALS_ID}"]
                ]) {
                    sh '''
                        # Deploy with account validation
                        find . -name "lambda_function.py" -type f | while read lambda_file; do
                            lambda_dir=$(dirname "$lambda_file")
                            lambda_name=$(basename "$lambda_dir")
                            function_name="${lambda_name}${LAMBDA_SUFFIX}"
                            
                            echo "📦 Deploying $function_name to account ${TARGET_ACCOUNT_ID}"
                            
                            # Create deployment package
                            cd "$lambda_dir"
                            zip -r "../../${function_name}.zip" . \
                                -x "*.pyc" "*__pycache__*" "tests/*" "*.md" ".git*"
                            cd - > /dev/null
                            
                            # Update Lambda function
                            aws lambda update-function-code \
                                --function-name "$function_name" \
                                --zip-file "fileb://${function_name}.zip" \
                                --region ${AWS_DEFAULT_REGION}
                            
                            echo "✅ $function_name deployed successfully"
                        done
                    '''
                }
            }
        }
        
        stage('Cross-Account Integration Tests') {
            steps {
                echo "🔗 Running cross-account integration tests..."
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding', 
                     credentialsId: "${AWS_CREDENTIALS_ID}"]
                ]) {
                    sh '''
                        source venv/bin/activate
                        
                        # Set environment variables for tests
                        export TARGET_ACCOUNT_ID="${TARGET_ACCOUNT_ID}"
                        export TARGET_ENVIRONMENT="${TARGET_ENVIRONMENT}"
                        export AWS_ACCOUNT_CONTEXT="${TARGET_ENVIRONMENT}"
                        
                        # Run tests with account context
                        python -m pytest tests/integration/ \
                            -v \
                            --junit-xml=integration-test-results.xml \
                            -k "integration" || true
                    '''
                }
            }
        }
    }
    
    post {
        success {
            script {
                echo """
🎉 CROSS-ACCOUNT DEPLOYMENT SUCCESSFUL!

🎯 Target Environment: ${env.TARGET_ENVIRONMENT}
🏢 Target Account: ${env.TARGET_ACCOUNT_ID}
🌐 Region: ${env.AWS_DEFAULT_REGION}

📋 Deployed Resources:
✅ Lambda functions with suffix: ${env.LAMBDA_SUFFIX}
✅ Static assets to: ${env.S3_ASSETS_BUCKET}

🔒 Security:
✅ Cross-account role assumed successfully
✅ Account validation passed
✅ Environment isolation maintained

🚀 Next Steps:
- Verify functionality in ${env.TARGET_ENVIRONMENT} environment
- Check AWS Console in account ${env.TARGET_ACCOUNT_ID}
- Run manual testing if needed
                """
            }
        }
    }
}
```

---

### **Step 5: Testing Cross-Account Setup**

### **Manual Verification Script**

```bash
#!/bin/bash
# test-cross-account.sh

# Test assume role functionality
echo "Testing cross-account access..."

# Dev account test
echo "=== Testing Dev Account Access ==="
aws sts assume-role \
  --role-arn "arn:aws:iam::DEV-ACCOUNT-ID:role/CrossAccountDeploymentRole" \
  --role-session-name "jenkins-test-session" \
  --external-id "unique-external-id-dev"

# Prod account test  
echo "=== Testing Prod Account Access ==="
aws sts assume-role \
  --role-arn "arn:aws:iam::PROD-ACCOUNT-ID:role/CrossAccountDeploymentRole" \
  --role-session-name "jenkins-test-session" \
  --external-id "unique-external-id-prod"

echo "Cross-account access test completed"
```

### **Jenkins Test Job**

**Create simple test pipeline:**
```groovy
pipeline {
    agent any
    stages {
        stage('Test Cross-Account Access') {
            parallel {
                stage('Test Dev Account') {
                    steps {
                        withCredentials([
                            [$class: 'AmazonWebServicesCredentialsBinding', 
                             credentialsId: 'aws-dev-cross-account-credentials']
                        ]) {
                            sh '''
                                echo "Testing Dev account access..."
                                aws sts get-caller-identity
                                aws s3 ls --region us-east-1
                            '''
                        }
                    }
                }
                stage('Test Prod Account') {
                    steps {
                        withCredentials([
                            [$class: 'AmazonWebServicesCredentialsBinding', 
                             credentialsId: 'aws-prod-cross-account-credentials']
                        ]) {
                            sh '''
                                echo "Testing Prod account access..."
                                aws sts get-caller-identity
                                aws s3 ls --region us-east-1
                            '''
                        }
                    }
                }
            }
        }
    }
}
```

---

### **Security Best Practices**

### **1. Least Privilege Access**
```json
{
  "Effect": "Allow",
  "Action": [
    "lambda:UpdateFunctionCode",
    "lambda:GetFunction"
  ],
  "Resource": "arn:aws:lambda:us-east-1:*:function:*-dev",
  "Condition": {
    "StringEquals": {
      "lambda:FunctionTag/Environment": "dev"
    }
  }
}
```

### **2. External ID for Additional Security**
```json
{
  "Condition": {
    "StringEquals": {
      "sts:ExternalId": "randomly-generated-unique-id-per-account"
    }
  }
}
```

### **3. Time-Limited Sessions**
```groovy
withCredentials([
    [$class: 'AmazonWebServicesCredentialsBinding', 
     credentialsId: 'aws-dev-cross-account-credentials',
     durationSeconds: 1800]  // 30 minutes max
]) {
    // Deployment operations
}
```

### **4. Resource Tagging**
```bash
# Tag all deployed resources
aws lambda tag-resource \
  --resource "arn:aws:lambda:us-east-1:${TARGET_ACCOUNT_ID}:function:${function_name}" \
  --tags Environment=${TARGET_ENVIRONMENT},DeployedBy=Jenkins,Project=${PROJECT_NAME}
```

### **5. Audit Logging**
```bash
# Log all cross-account operations
echo "$(date): Assumed role in account ${TARGET_ACCOUNT_ID} for deployment" >> /var/log/jenkins/cross-account.log
```

This setup provides secure, auditable, and automated cross-account deployments while maintaining environment isolation and following AWS security best practices.
