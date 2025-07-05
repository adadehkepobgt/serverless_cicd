## Yes! Here's a Simple Test Pipeline

Let's create a minimal pipeline to test your Jenkins-Bitbucket connection and configurations.

### **Step 1: Create Simple Test Files**

**Create these files in your repository:**

#### **Jenkinsfile (in root)**
```groovy
pipeline {
    agent any
    
    stages {
        stage('Test Connection') {
            steps {
                echo "🔗 Testing Jenkins-Bitbucket connection..."
                echo "Repository: ${env.GIT_URL}"
                echo "Branch: ${env.GIT_BRANCH}"
                echo "Commit: ${env.GIT_COMMIT}"
            }
        }
        
        stage('Environment Check') {
            steps {
                sh '''
                    echo "🔍 Checking environment..."
                    echo "Current directory: $(pwd)"
                    echo "Files in repository:"
                    ls -la
                    echo "Git status:"
                    git status
                '''
            }
        }
        
        stage('Simple Test') {
            steps {
                sh '''
                    echo "🧪 Running simple test..."
                    echo "Testing basic shell commands..."
                    
                    # Test file creation
                    echo "Hello from Jenkins!" > test-output.txt
                    cat test-output.txt
                    
                    # Test Python availability
                    python3 --version || echo "Python3 not available"
                    
                    echo "✅ Simple test completed!"
                '''
            }
        }
        
        stage('Deploy to Dev') {
            when {
                branch 'dev'
            }
            steps {
                echo "🚀 This would deploy to dev environment..."
                echo "Deployment simulation successful!"
            }
        }
    }
    
    post {
        success {
            echo "✅ Test pipeline completed successfully!"
        }
        failure {
            echo "❌ Test pipeline failed!"
        }
        always {
            echo "🧹 Cleaning up test files..."
            sh 'rm -f test-output.txt || true'
        }
    }
}
```

#### **README.md (in root)**
```markdown
# Test Repository

This is a test repository for Jenkins CI/CD pipeline.

## Pipeline Status
- ✅ Jenkins connection
- ✅ Bitbucket webhooks
- ✅ SSH authentication
- ✅ Basic deployment

## Test Commands
```bash
# Push to dev branch to trigger deployment
git checkout dev
git push origin dev
```

**Last updated:** $(date)
```

### **Step 2: Push to Dev Branch**

**Commands to test:**
```bash
# 1. Create and add files
git add Jenkinsfile README.md
git commit -m "Add test pipeline configuration"

# 2. Switch to dev branch (or create if doesn't exist)
git checkout dev

# 3. Push to trigger Jenkins
git push origin dev
```

### **Step 3: What Should Happen**

**1. Webhook triggers Jenkins:**
- Bitbucket sends webhook to Jenkins
- Jenkins starts build within seconds

**2. Jenkins pipeline runs:**
- ✅ **Test Connection** - Shows repo info
- ✅ **Environment Check** - Lists files, shows git status  
- ✅ **Simple Test** - Tests basic commands
- ✅ **Deploy to Dev** - Shows deployment message (dev branch only)

**3. Expected Jenkins output:**
```
🔗 Testing Jenkins-Bitbucket connection...
Repository: ssh://git@bitbucket.corp.jefco.com:7999/~hcia/test.git
Branch: origin/dev
Commit: abc123...

🔍 Checking environment...
Current directory: /var/jenkins_home/workspace/test-pipeline
Files in repository:
-rw-r--r-- 1 jenkins jenkins  1234 Jul 05 10:30 Jenkinsfile
-rw-r--r-- 1 jenkins jenkins   456 Jul 05 10:30 README.md

🧪 Running simple test...
Testing basic shell commands...
Hello from Jenkins!
Python 3.9.2

✅ Simple test completed!
🚀 This would deploy to dev environment...
Deployment simulation successful!
✅ Test pipeline completed successfully!
```

### **Step 4: Test Different Scenarios**

#### **Test 1: Direct Push to Dev**
```bash
git checkout dev
echo "Test change $(date)" >> README.md
git add README.md
git commit -m "Test direct push to dev"
git push origin dev
```
**Expected:** Jenkins should trigger and run deployment stage

#### **Test 2: Feature Branch + PR** 
```bash
git checkout -b feature/test-pr
echo "PR test change" >> README.md
git add README.md
git commit -m "Test PR workflow"
git push origin feature/test-pr
```
**Then create PR in Bitbucket:** `feature/test-pr` → `dev`
**Expected:** Jenkins should trigger but skip deployment stage

### **Step 5: Verify What's Working**

**Check these in Jenkins:**
- ✅ **Build triggered automatically** (webhook working)
- ✅ **SSH connection successful** (credentials working)
- ✅ **Repository cloned** (access permissions working)
- ✅ **Environment variables available** (Jenkins setup working)
- ✅ **Deployment stage runs on dev branch** (branch conditions working)

**Check these in Bitbucket:**
- ✅ **Webhook shows successful delivery**
- ✅ **Build status appears in commits** (if configured)

### **If Something Fails:**

**Webhook not triggering:**
- Check webhook URL: `http://your-jenkins-url:8080/bitbucket-hook/`
- Verify webhook is active in Bitbucket

**SSH/Authentication errors:**
- Re-run SSH setup pipeline from earlier
- Check Jenkins credentials

**Pipeline fails:**
- Look at Jenkins console output
- Check specific stage that failed

### **Quick Success Test:**

**Minimum working test:**
1. **Push this simple Jenkinsfile**
2. **Check Jenkins shows build started** 
3. **Verify all 4 stages complete successfully**
4. **See "✅ Test pipeline completed successfully!" message**

**If this works, your basic setup is solid and you can add complexity (real tests, deployment scripts, auto-merge, etc.)!**
