## 🔧 **Jenkins Auto-Approve & Auto-Merge Configuration**

### **📋 Prerequisites Check**
```
✅ Bitbucket Push and Pull Request Plugin installed
✅ Jenkins has API access to Bitbucket
✅ Proper webhook configuration
✅ Service account with merge permissions
```

---

## **🔐 Step 1: Configure Bitbucket Credentials**

### **In Jenkins Dashboard:**
```
1. Manage Jenkins → Manage Credentials
2. Add Credentials → Username with password
   - Username: [Bitbucket service account]
   - Password: [App password or API token]
   - ID: bitbucket-service-account
```

### **Bitbucket App Password Setup:**
```
Bitbucket Settings → App passwords → Create app password
Required permissions:
✅ Repositories: Read, Write
✅ Pull requests: Read, Write
✅ Account: Read
```

---

## **🛠️ Step 2: Jenkins Pipeline Configuration**

### **Jenkinsfile Example:**
```groovy
pipeline {
    agent any
    
    environment {
        BITBUCKET_CREDENTIALS = credentials('bitbucket-service-account')
        REPO_SLUG = 'your-repo-name'
        WORKSPACE_ID = 'your-workspace'
    }
    
    stages {
        stage('Build & Test') {
            steps {
                script {
                    // Your build and test steps
                    sh 'npm install'
                    sh 'npm test'
                    sh 'npm run build'
                }
            }
        }
        
        stage('Auto Approve PR') {
            when {
                // Only for pull requests
                changeRequest()
            }
            steps {
                script {
                    def prId = env.CHANGE_ID
                    approveReviews(prId)
                }
            }
        }
        
        stage('Auto Merge PR') {
            when {
                allOf {
                    changeRequest()
                    // Add additional conditions as needed
                    expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
                }
            }
            steps {
                script {
                    def prId = env.CHANGE_ID
                    mergePullRequest(prId)
                }
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed - PR will not be auto-merged'
        }
    }
}

// Function to approve pull request
def approveReviews(prId) {
    def approveUrl = "https://api.bitbucket.org/2.0/repositories/${WORKSPACE_ID}/${REPO_SLUG}/pullrequests/${prId}/approve"
    
    sh """
        curl -X POST \\
        -H "Content-Type: application/json" \\
        -u "\$BITBUCKET_CREDENTIALS_USR:\$BITBUCKET_CREDENTIALS_PSW" \\
        "${approveUrl}"
    """
    
    echo "✅ Pull request ${prId} approved"
}

// Function to merge pull request
def mergePullRequest(prId) {
    def mergeUrl = "https://api.bitbucket.org/2.0/repositories/${WORKSPACE_ID}/${REPO_SLUG}/pullrequests/${prId}/merge"
    
    def mergePayload = """
    {
        "type": "pullrequest_merge",
        "message": "Auto-merged by Jenkins after successful CI/CD pipeline",
        "close_source_branch": true,
        "merge_strategy": "merge_commit"
    }
    """
    
    sh """
        curl -X POST \\
        -H "Content-Type: application/json" \\
        -u "\$BITBUCKET_CREDENTIALS_USR:\$BITBUCKET_CREDENTIALS_PSW" \\
        -d '${mergePayload}' \\
        "${mergeUrl}"
    """
    
    echo "🚀 Pull request ${prId} merged successfully"
}
```

---

## **⚙️ Step 3: Plugin Configuration**

### **Bitbucket Push and Pull Request Plugin Settings:**
```
Jenkins Dashboard → Manage Jenkins → Configure System

Bitbucket Server Integration:
1. Server Base URL: https://bitbucket.org
2. Credentials: Select your bitbucket-service-account
3. Test Connection ✅

Pull Request Triggers:
☑️ Build when a pull request is created
☑️ Build when a pull request is updated
☑️ Build when target branch changes
```

---

## **🔧 Step 4: Advanced Configuration Options**

### **Conditional Auto-Merge (Recommended):**
```groovy
stage('Conditional Auto Merge') {
    when {
        allOf {
            changeRequest()
            // Only auto-merge from specific branches
            anyOf {
                branch 'feature/*'
                branch 'bugfix/*'
            }
            // Ensure all checks pass
            expression { currentBuild.result == 'SUCCESS' }
            // Check for specific commit message patterns
            expression { 
                env.GIT_COMMIT_MESSAGE?.contains('[auto-merge]') 
            }
        }
    }
    steps {
        script {
            mergePullRequest(env.CHANGE_ID)
        }
    }
}
```

### **Branch Protection Rules Setup:**
```groovy
// Only auto-merge if specific conditions are met
def shouldAutoMerge() {
    def checks = [
        currentBuild.result == 'SUCCESS',
        env.CHANGE_TARGET == 'main' || env.CHANGE_TARGET == 'develop',
        !env.GIT_COMMIT_MESSAGE?.contains('[skip-auto-merge]')
    ]
    
    return checks.every { it == true }
}
```

---

## **🛡️ Step 5: Safety Measures**

### **Add Required Checks:**
```groovy
stage('Pre-Merge Validation') {
    steps {
        script {
            // Check test coverage
            def coverage = sh(
                script: 'npm run test:coverage | grep "Lines" | awk "{print \$4}" | sed "s/%//"',
                returnStdout: true
            ).trim().toInteger()
            
            if (coverage < 80) {
                error("Test coverage ${coverage}% is below required 80%")
            }
            
            // Check for breaking changes
            sh 'npm audit --audit-level high'
            
            // Validate branch naming convention
            if (!env.CHANGE_BRANCH?.matches(/(feature|bugfix|hotfix)\/.*/) && env.CHANGE_TARGET == 'main') {
                error("Invalid branch naming for main branch merge")
            }
        }
    }
}
```

---

## **📝 Step 6: Webhook Configuration**

### **Bitbucket Webhook Setup:**
```
Repository Settings → Webhooks → Add webhook

URL: https://your-jenkins-url/bitbucket-hook/
Events to trigger:
☑️ Repository push
☑️ Pull request created
☑️ Pull request updated
☑️ Pull request approved
☑️ Pull request merged
```

---

## **🔍 Step 7: Testing & Monitoring**

### **Test the Setup:**
```bash
# 1. Create a test branch
git checkout -b feature/test-auto-merge

# 2. Make a small change and commit
echo "// Test auto-merge" >> test.js
git add test.js
git commit -m "Test auto-merge functionality [auto-merge]"

# 3. Push and create PR
git push origin feature/test-auto-merge
# Create PR via Bitbucket UI or CLI

# 4. Monitor Jenkins pipeline execution
```

### **Monitoring Dashboard Addition:**
```groovy
// Add to your Jenkins pipeline for metrics
stage('Report Metrics') {
    steps {
        script {
            def metrics = [
                pr_id: env.CHANGE_ID,
                merge_time: new Date().format('yyyy-MM-dd HH:mm:ss'),
                pipeline_duration: currentBuild.duration,
                auto_merged: true
            ]
            
            writeJSON file: 'merge-metrics.json', json: metrics
            archiveArtifacts artifacts: 'merge-metrics.json'
        }
    }
}
```

---

## **⚠️ Important Considerations**

### **Security Best Practices:**
- Use service accounts, not personal credentials
- Limit auto-merge to specific branch patterns
- Require successful CI checks before merge
- Set up branch protection rules in Bitbucket
- Log all auto-merge activities for auditing

### **Team Guidelines:**
- Use `[skip-auto-merge]` in commit messages to override
- Establish clear branch naming conventions
- Set minimum test coverage requirements
- Define approval requirements for critical branches

This setup will automatically approve and merge PRs that pass your CI/CD pipeline while maintaining safety guardrails! 🚀
