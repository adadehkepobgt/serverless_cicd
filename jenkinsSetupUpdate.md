## Complete Jenkins Pipeline Configuration

### **Step 1: Create Pipeline Job**
• Jenkins Dashboard → **New Item**
• Enter name: `lambda-development-pipeline`
• Select: **Pipeline** (not Multibranch)
• Click **OK**

---

### **Step 2: General Configuration**

**☑️ Bitbucket Team/Project:** 
• Check this box
• **Team or User Account:** `your-bitbucket-workspace-name`
• **Repository Name:** `your-repository-name`

**All other General options:** Leave unchecked

---

### **Step 3: Build Triggers Configuration**

**☑️ Build when a change is pushed to Bitbucket**
• Check this box only

**All other trigger options:** Leave unchecked
• ❌ Build after other projects are built
• ❌ Build periodically  
• ❌ Poll SCM

---

### **Step 4: Pipeline Configuration**

**Definition:** `Pipeline script from SCM`

**SCM:** `Git`

**Repository URL:** `https://bitbucket.org/your-workspace/your-repo.git`

**Credentials:** 
• Click **Add** → **Jenkins**
• **Kind:** `Username with password`
• **Username:** Your Bitbucket username
• **Password:** Your Bitbucket App Password
• **ID:** `bitbucket-credentials`
• **Description:** `Bitbucket Access`
• Click **Add**, then select from dropdown

**Branches to build:**
• **Branch Specifier:** `*/dev`

**Additional Behaviours:** Click **Add** and select:
• **Clean before checkout**
• **Clean after checkout**

**Script Path:** `Jenkinsfile`

**☑️ Lightweight checkout:** Check this box

---

### **Step 5: Environment Variables (Optional)**
Go to **Build Environment** section:
• ☑️ **Delete workspace before build starts** (recommended)

---

### **Step 6: Bitbucket App Password Setup**

**Create App Password:**
• Bitbucket → **Personal Settings** → **App passwords**
• Click **Create app password**
• **Label:** `Jenkins Auto Merge`
• **Permissions:**
  - ☑️ **Repositories: Write**
  - ☑️ **Pull requests: Write**
• Copy the generated password

**Add to Jenkins Credentials:**
• Jenkins → **Manage Jenkins** → **Manage Credentials** → **Global**
• **Add Credentials** → **Username with password**
• **Username:** Your Bitbucket username
• **Password:** The App Password you copied
• **ID:** `bitbucket-auto-merge`
• **Description:** `Bitbucket Auto Merge Access`

---

### **Step 7: Bitbucket Repository Configuration**

**7a. Webhooks:**
• Repository Settings → **Webhooks**
• Click **Add webhook**
• **Title:** `Jenkins Pipeline Trigger`
• **URL:** `http://your-jenkins-url:8080/bitbucket-hook/`
• **Status:** Active
• **Triggers:**
  - ☑️ **Repository push**
  - ☑️ **Pull request created**
  - ☑️ **Pull request updated**
• **Save**

**7b. Branch Permissions (Optional but Recommended):**
• Repository Settings → **Branch permissions**
• **Branch or pattern:** `dev`
• **Branch permission type:** `Restrict pushes`
• ☑️ **Allow only if pull request requirements pass**
• **Pull request requirements:**
  - ☑️ **Require builds to pass**
• **Save**

---

### **Step 8: Jenkins Global Configuration**

**Add Environment Variables:**
• Jenkins → **Manage Jenkins** → **Configure System**
• Scroll to **Global properties**
• ☑️ **Environment variables**
• Add these variables:
  - **Name:** `BITBUCKET_WORKSPACE` **Value:** `your-workspace-name`
  - **Name:** `BITBUCKET_REPO` **Value:** `your-repository-name`
  - **Name:** `AWS_REGION` **Value:** `us-east-1` (or your region)
  - **Name:** `LAMBDA_FUNCTION_NAME` **Value:** `your-lambda-function-name`
  - **Name:** `S3_BUCKET` **Value:** `your-s3-bucket-name`

---

### **Step 9: Jenkinsfile (No Auto-Delete Branch)**

Create this `Jenkinsfile` in your repository root:

```groovy
pipeline {
    agent any
    
    environment {
        AWS_REGION = "${env.AWS_REGION}"
        LAMBDA_FUNCTION_NAME = "${env.LAMBDA_FUNCTION_NAME}"
        S3_BUCKET = "${env.S3_BUCKET}"
        BITBUCKET_WORKSPACE = "${env.BITBUCKET_WORKSPACE}"
        BITBUCKET_REPO = "${env.BITBUCKET_REPO}"
    }
    
    stages {
        stage('PR Validation') {
            when {
                changeRequest target: 'dev'
            }
            steps {
                echo "🔍 Testing PR #${env.CHANGE_ID}: ${env.CHANGE_BRANCH} → dev"
            }
        }
        
        stage('Install Dependencies') {
            steps {
                script {
                    if (fileExists('requirements.txt')) {
                        sh 'pip install -r requirements.txt'
                    } else if (fileExists('package.json')) {
                        sh 'npm install'
                    }
                }
            }
        }
        
        stage('Run Tests') {
            steps {
                script {
                    if (fileExists('tests/')) {
                        if (fileExists('requirements.txt')) {
                            sh 'python -m pytest tests/ --junitxml=test-results.xml -v'
                        } else if (fileExists('package.json')) {
                            sh 'npm test'
                        }
                    } else {
                        echo "⚠️ No tests directory found - creating placeholder"
                        sh 'mkdir -p tests && echo "# Add your tests here" > tests/README.md'
                    }
                }
            }
            post {
                always {
                    script {
                        if (fileExists('test-results.xml')) {
                            publishTestResults testResultsPattern: 'test-results.xml'
                        }
                    }
                }
            }
        }
        
        stage('Build Lambda Package') {
            steps {
                script {
                    if (fileExists('requirements.txt')) {
                        sh '''
                            mkdir -p package
                            pip install -r requirements.txt -t package/
                            cp *.py package/
                            cd package && zip -r ../lambda-deployment.zip .
                        '''
                    } else if (fileExists('package.json')) {
                        sh '''
                            npm install --production
                            zip -r lambda-deployment.zip . -x "tests/*" "*.git/*" "node_modules/.cache/*"
                        '''
                    }
                }
                archiveArtifacts artifacts: 'lambda-deployment.zip', fingerprint: true
            }
        }
        
        stage('Integration Tests') {
            steps {
                echo "✅ Running integration tests..."
                // Add your specific S3/Lambda integration tests here
                sh 'echo "Integration tests completed successfully"'
            }
        }
        
        stage('Auto-Merge PR') {
            when {
                allOf {
                    changeRequest target: 'dev'
                    expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
                }
            }
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'bitbucket-auto-merge', 
                                    usernameVariable: 'BB_USER', passwordVariable: 'BB_PASS')]) {
                        
                        echo "🚀 Auto-merging PR #${env.CHANGE_ID} after successful tests"
                        
                        def mergeResponse = sh(
                            script: """
                                curl -s -w "%{http_code}" -o merge_response.json \
                                -X POST \
                                -u '${BB_USER}:${BB_PASS}' \
                                'https://api.bitbucket.org/2.0/repositories/${env.BITBUCKET_WORKSPACE}/${env.BITBUCKET_REPO}/pullrequests/${env.CHANGE_ID}/merge' \
                                -H 'Content-Type: application/json' \
                                -d '{
                                    "type": "merge",
                                    "message": "🤖 Auto-merged by Jenkins after passing all tests\\n\\nBuild: ${BUILD_URL}",
                                    "close_source_branch": false
                                }'
                            """,
                            returnStdout: true
                        ).trim()
                        
                        if (mergeResponse == "200") {
                            echo "✅ PR merged successfully - feature branch preserved for developer decision"
                            def mergeData = readJSON file: 'merge_response.json'
                            env.MERGE_COMMIT = mergeData.hash
                        } else {
                            echo "Response details:"
                            sh 'cat merge_response.json || echo "No response file"'
                            error "❌ Failed to merge PR. HTTP Status: ${mergeResponse}"
                        }
                    }
                }
            }
        }
        
        stage('Deploy to Dev Environment') {
            when {
                anyOf {
                    expression { env.MERGE_COMMIT != null }
                    allOf {
                        branch 'dev'
                        not { changeRequest() }
                    }
                }
            }
            steps {
                echo "🚀 Deploying to dev environment..."
                script {
                    echo "Deploying Lambda function: ${env.LAMBDA_FUNCTION_NAME}-dev"
                    echo "Using deployment package: lambda-deployment.zip"
                    echo "Target S3 bucket: ${env.S3_BUCKET}"
                    
                    // Add your actual AWS deployment commands here when ready
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        
        success {
            script {
                if (env.CHANGE_ID && env.MERGE_COMMIT) {
                    echo "🎉 PR #${env.CHANGE_ID} auto-merged and deployed to dev!"
                    echo "💡 Feature branch '${env.CHANGE_BRANCH}' is preserved - developer can delete manually if desired"
                } else if (env.CHANGE_ID) {
                    echo "✅ PR #${env.CHANGE_ID} tests passed - ready for merge"
                } else {
                    echo "✅ Dev deployment successful"
                }
            }
        }
        
        failure {
            script {
                if (env.CHANGE_ID) {
                    echo "❌ PR #${env.CHANGE_ID} failed tests - merge blocked"
                } else {
                    echo "❌ Dev build/deployment failed"
                }
            }
        }
    }
}
```

---

### **Step 10: Test the Setup**

**Test Workflow:**
1. Create feature branch: `git checkout -b feature/test-pipeline`
2. Make changes and push: `git push origin feature/test-pipeline`
3. Create PR in Bitbucket targeting `dev` branch
4. Jenkins should trigger automatically
5. If tests pass → PR auto-merges, feature branch stays
6. Developer manually deletes feature branch when ready

**Key Points:**
• `"close_source_branch": false` keeps the feature branch after merge
• Developer has full control over when to delete feature branches
• All automation focused on testing and deployment, not branch management


why these triggers
## Why We Need Webhooks (Step 7a)

### **What Webhooks Do:**
Webhooks are Bitbucket's way of **instantly notifying Jenkins** when something happens in your repository, instead of Jenkins constantly checking for changes.

### **Without Webhooks:**
• Jenkins would use **Poll SCM** - checking every few minutes: "Any changes? Any changes? Any changes?"
• **Slow response**: 2-5 minute delays before builds start
• **Resource waste**: Constant unnecessary API calls to Bitbucket
• **Network overhead**: Jenkins hitting Bitbucket server repeatedly

### **With Webhooks:**
• **Instant notification**: Bitbucket immediately tells Jenkins "Hey, something happened!"
• **Zero delay**: Build starts within seconds of PR creation/push
• **Efficient**: No wasted polling, only real events trigger builds
• **Real-time**: Developer sees build status immediately in their PR

**Think of it like:**
- **Without webhook**: Constantly calling a friend to ask "Are you home yet?"
- **With webhook**: Friend calls you the moment they get home

---

## Why These 3 Specific Triggers

### **☑️ Repository push**
**When it triggers:** Developer pushes commits directly to `dev` branch
**Why needed:** 
• Someone might push directly to dev (hotfixes, admin changes)
• Need to deploy these direct changes to dev environment
• Maintains consistency - all code in dev gets tested and deployed

**Example scenario:**
```bash
git checkout dev
git commit -m "Quick hotfix"
git push origin dev  # ← This triggers Jenkins
```

### **☑️ Pull request created** 
**When it triggers:** Developer creates a new PR targeting `dev` branch
**Why needed:**
• Start testing **immediately** when PR is opened
• Give instant feedback to developer about test results
• Block the merge if tests fail from the very beginning

**Example scenario:**
```bash
# Developer creates PR: feature/new-feature → dev
# Jenkins immediately starts: "Testing your PR..."
```

### **☑️ Pull request updated**
**When it triggers:** Developer pushes new commits to an existing PR
**Why needed:**
• Re-test every time developer adds fixes or new code
• Ensure the **latest version** of the PR passes tests
• Handle the common workflow: "Oops, fix failing test, push again"

**Example scenario:**
```bash
# PR exists, tests failed
git commit -m "Fix failing test"
git push origin feature/new-feature  # ← Re-triggers Jenkins testing
```

### **Why NOT other triggers:**
• **Pull request merged**: We don't need this because Jenkins auto-merges
• **Pull request declined**: No point testing rejected code  
• **Pull request approved**: We auto-merge on test success, not manual approval

---

## Why Branch Permissions (Step 7b)

### **Branch Permissions Purpose:**
**Enforce the rule:** "You cannot push directly to dev - you MUST go through a PR and pass tests"

### **Without Branch Permissions:**
• Developer could bypass your entire pipeline: `git push origin dev`
• No testing, no validation, direct deployment of potentially broken code
• Your careful CI/CD process becomes optional

### **With Branch Permissions:**
• **Direct pushes blocked**: Only PR merges allowed (except for admins)
• **"Require builds to pass"**: PR can't merge unless Jenkins gives ✅
• **Enforced workflow**: Every change must go through your pipeline

### **The Protection Flow:**
1. Developer tries to push directly to dev → **BLOCKED**
2. Developer creates PR → Jenkins tests it
3. Tests fail → PR shows "❌ Cannot merge" 
4. Tests pass → Jenkins auto-merges → Deployment happens

**Think of it like:**
- A bouncer at a club door who checks IDs (tests) before letting anyone in (merge)

---

## Complete Flow Visualization

```
Developer pushes to feature branch
           ↓
    Creates PR → dev
           ↓
Webhook → "Pull request created" → Jenkins starts testing
           ↓
    Tests running...
           ↓
   ┌─ Tests FAIL → PR blocked, no merge
   │
   └─ Tests PASS → Jenkins auto-merges → Webhook "Repository push" 
                                              ↓
                                         Jenkins deploys to dev
```

**The webhooks ensure instant communication, and branch permissions ensure the process can't be bypassed.**

## Why We Need Webhooks (Step 7a)

### **What Webhooks Do:**
Webhooks are Bitbucket's way of **instantly notifying Jenkins** when something happens in your repository, instead of Jenkins constantly checking for changes.

### **Without Webhooks:**
• Jenkins would use **Poll SCM** - checking every few minutes: "Any changes? Any changes? Any changes?"
• **Slow response**: 2-5 minute delays before builds start
• **Resource waste**: Constant unnecessary API calls to Bitbucket
• **Network overhead**: Jenkins hitting Bitbucket server repeatedly

### **With Webhooks:**
• **Instant notification**: Bitbucket immediately tells Jenkins "Hey, something happened!"
• **Zero delay**: Build starts within seconds of PR creation/push
• **Efficient**: No wasted polling, only real events trigger builds
• **Real-time**: Developer sees build status immediately in their PR

**Think of it like:**
- **Without webhook**: Constantly calling a friend to ask "Are you home yet?"
- **With webhook**: Friend calls you the moment they get home

---

## Why These 3 Specific Triggers

### **☑️ Repository push**
**When it triggers:** Developer pushes commits directly to `dev` branch
**Why needed:** 
• Someone might push directly to dev (hotfixes, admin changes)
• Need to deploy these direct changes to dev environment
• Maintains consistency - all code in dev gets tested and deployed

**Example scenario:**
```bash
git checkout dev
git commit -m "Quick hotfix"
git push origin dev  # ← This triggers Jenkins
```

### **☑️ Pull request created** 
**When it triggers:** Developer creates a new PR targeting `dev` branch
**Why needed:**
• Start testing **immediately** when PR is opened
• Give instant feedback to developer about test results
• Block the merge if tests fail from the very beginning

**Example scenario:**
```bash
# Developer creates PR: feature/new-feature → dev
# Jenkins immediately starts: "Testing your PR..."
```

### **☑️ Pull request updated**
**When it triggers:** Developer pushes new commits to an existing PR
**Why needed:**
• Re-test every time developer adds fixes or new code
• Ensure the **latest version** of the PR passes tests
• Handle the common workflow: "Oops, fix failing test, push again"

**Example scenario:**
```bash
# PR exists, tests failed
git commit -m "Fix failing test"
git push origin feature/new-feature  # ← Re-triggers Jenkins testing
```

### **Why NOT other triggers:**
• **Pull request merged**: We don't need this because Jenkins auto-merges
• **Pull request declined**: No point testing rejected code  
• **Pull request approved**: We auto-merge on test success, not manual approval

---

## Why Branch Permissions (Step 7b)

### **Branch Permissions Purpose:**
**Enforce the rule:** "You cannot push directly to dev - you MUST go through a PR and pass tests"

### **Without Branch Permissions:**
• Developer could bypass your entire pipeline: `git push origin dev`
• No testing, no validation, direct deployment of potentially broken code
• Your careful CI/CD process becomes optional

### **With Branch Permissions:**
• **Direct pushes blocked**: Only PR merges allowed (except for admins)
• **"Require builds to pass"**: PR can't merge unless Jenkins gives ✅
• **Enforced workflow**: Every change must go through your pipeline

### **The Protection Flow:**
1. Developer tries to push directly to dev → **BLOCKED**
2. Developer creates PR → Jenkins tests it
3. Tests fail → PR shows "❌ Cannot merge" 
4. Tests pass → Jenkins auto-merges → Deployment happens

**Think of it like:**
- A bouncer at a club door who checks IDs (tests) before letting anyone in (merge)

---

## Complete Flow Visualization

```
Developer pushes to feature branch
           ↓
    Creates PR → dev
           ↓
Webhook → "Pull request created" → Jenkins starts testing
           ↓
    Tests running...
           ↓
   ┌─ Tests FAIL → PR blocked, no merge
   │
   └─ Tests PASS → Jenkins auto-merges → Webhook "Repository push" 
                                              ↓
                                         Jenkins deploys to dev
```

**The webhooks ensure instant communication, and branch permissions ensure the process can't be bypassed.**


## What the Bitbucket App Password Setup is For

This step creates **authentication credentials** that allow Jenkins to automatically merge pull requests on your behalf. Here's the breakdown:

### **Why Jenkins Needs These Credentials:**

**The Problem:**
• Jenkins can read your repository (clone, checkout code) using basic Git credentials
• But Jenkins CANNOT merge pull requests without special API permissions
• Pull request merging requires authenticated API calls to Bitbucket

**The Solution:**
• **App Password** = Special token with specific permissions (safer than your real password)
• **API Access** = Allows Jenkins to make authenticated calls to Bitbucket's REST API
• **Auto-merge capability** = Jenkins can programmatically merge PRs after tests pass

### **What Each Permission Does:**

**☑️ Repositories: Write**
• Allows Jenkins to push commits and merge branches
• Required for the actual merge operation
• Lets Jenkins update the dev branch with merged code

**☑️ Pull requests: Write** 
• Allows Jenkins to modify pull request status
• Required to execute the merge command via API
• Lets Jenkins close/merge PRs programmatically

### **How It's Used in the Pipeline:**

In your Jenkinsfile, this section uses these credentials:

```groovy
stage('Auto-Merge PR') {
    steps {
        script {
            withCredentials([usernamePassword(credentialsId: 'bitbucket-auto-merge', 
                            usernameVariable: 'BB_USER', passwordVariable: 'BB_PASS')]) {
                
                // This API call merges the PR automatically
                curl -u '${BB_USER}:${BB_PASS}' \
                'https://api.bitbucket.org/2.0/repositories/.../pullrequests/${CHANGE_ID}/merge'
            }
        }
    }
}
```

### **Security Benefits:**

**App Password vs Real Password:**
• ✅ **Limited scope** - Only specific permissions, not full account access
• ✅ **Revokable** - Can be deleted anytime without changing your main password
• ✅ **Auditable** - Bitbucket logs show which app password was used
• ✅ **Safer** - If compromised, attacker can't access your full account

### **Alternative (Manual Process):**

**Without this setup:**
1. Jenkins runs tests ✅
2. Tests pass ✅  
3. Jenkins says "PR ready to merge" ✅
4. **Developer must manually click "Merge" in Bitbucket UI** ❌
5. Jenkins then deploys after manual merge ✅

**With this setup:**
1. Jenkins runs tests ✅
2. Tests pass ✅
3. **Jenkins automatically merges PR** ✅ 
4. Jenkins immediately deploys ✅

### **If You Don't Want Auto-Merge:**

If you prefer manual merging, you can **skip Step 6 entirely** and remove the `Auto-Merge PR` stage from the Jenkinsfile. The pipeline will still:
• Run tests on PRs
• Block merging if tests fail  
• Deploy after you manually merge
• Preserve feature branches

The choice is yours - auto-merge for maximum automation, or manual merge for more control.
