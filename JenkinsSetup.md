**Step-by-Step Jenkins Setup Guide**

**Creating the Pipeline Project**

**Step 1: Create New Pipeline**
1. Go to Jenkins Dashboard
2. Click **"New Item"** (top left)
3. **Item name**: Enter `Lambda-Serverless-Pipeline` (or your preferred name)
4. Select **"Pipeline"** (scroll down to find it)
5. Click **"OK"**

**Step 2: General Configuration**
In the configuration page that opens:

**General Section:**
- ☑️ **Check "Discard old builds"**
  - **Days to keep builds**: `30`
  - **Max # of builds to keep**: `10`
- ☐ **Leave "GitHub project" unchecked** (we're using BitBucket)
- ☐ **Leave "This project is parameterized" unchecked** (for now)
- ☐ **Leave "Disable this project" unchecked**

**Step 3: Build Triggers**
**Build Triggers Section:**
- ☐ **Leave "Build after other projects are built" unchecked**
- ☑️ **Check "Build when a change is pushed to BitBucket"**
- ☐ **Leave "Build periodically" unchecked**
- ☐ **Leave "Poll SCM" unchecked**
- ☐ **Leave "Quiet period" unchecked**

**Step 4: Advanced Project Options**
**Advanced Project Options Section:**
- **Display Name**: Leave empty (will use project name)
- ☐ **Leave "Use custom workspace" unchecked**

**Step 5: Pipeline Configuration**
**Pipeline Section:** (This is the most important part)

**Definition**: Select **"Pipeline script from SCM"**

**SCM**: Select **"Git"**

**Repository URL**: Enter your BitBucket repository URL
- Format: `https://bitbucket.org/your-username/your-repo-name.git`
- Example: `https://bitbucket.org/johnsmith/lambda-project.git`

**Credentials**: Click **"Add"** dropdown → **"Jenkins"**
- **Kind**: Select **"Username with password"**
- **Scope**: Select **"Global (Jenkins, nodes, items, all child items, etc.)"**
- **Username**: Your BitBucket username
- **Password**: Your BitBucket App Password (NOT your regular password)
- **ID**: `bitbucket-credentials`
- **Description**: `BitBucket Repository Access`
- Click **"Add"**
- Then select the credential you just created from the dropdown

**Branches to build**:
- **Branch Specifier**: `*/main` (or `*/master` if that's your default branch)

**Repository browser**: Select **"bitbucketweb"**
- **URL**: `https://bitbucket.org/your-username/your-repo-name/`

**Additional Behaviours**: Leave empty for now

**Script Path**: `Jenkinsfile` (default - this tells Jenkins to look for Jenkinsfile in your repo root)

**Lightweight checkout**: ☑️ **Check this box** (makes checkout faster)

**Step 6: Save Configuration**
Click **"Save"** at the bottom

---

**Setting Up AWS Credentials**

**Step 7: Add AWS Credentials**
1. Go to **Manage Jenkins** → **Manage Credentials**
2. Click **"Global"** under **"Stores scoped to Jenkins"**
3. Click **"Add Credentials"** (left side)

**For Development Environment:**
- **Kind**: Select **"AWS Credentials"**
- **Scope**: **"Global (Jenkins, nodes, items, all child items, etc.)"**
- **ID**: `aws-dev-credentials`
- **Description**: `AWS Development Environment Access`
- **Access Key ID**: Enter the Access Key from your Jenkins-dev IAM user
- **Secret Access Key**: Enter the Secret Key from your Jenkins-dev IAM user
- Click **"OK"**

**Repeat for QA Environment:**
- **Kind**: **"AWS Credentials"**
- **Scope**: **"Global"**
- **ID**: `aws-qa-credentials`
- **Description**: `AWS QA Environment Access`
- **Access Key ID**: Enter QA Access Key
- **Secret Access Key**: Enter QA Secret Key

**Repeat for Production Environment:**
- **Kind**: **"AWS Credentials"**
- **Scope**: **"Global"**
- **ID**: `aws-prod-credentials`
- **Description**: `AWS Production Environment Access`
- **Access Key ID**: Enter Production Access Key
- **Secret Access Key**: Enter Production Secret Key

---

**Configuring Global Tools**

**Step 8: Configure Global Tools**
1. Go to **Manage Jenkins** → **Global Tool Configuration**

**Git Configuration:**
- **Name**: `Default`
- **Path to Git executable**: `/usr/bin/git` (leave default)

**Add Terraform:**
- Scroll to **"Terraform"** section
- Click **"Add Terraform"**
- **Name**: `Terraform`
- ☑️ **Check "Install automatically"**
- Click **"Add Installer"** → **"Install from hashicorp.com"**
- **Version**: Select latest version (e.g., `1.6.6`)

**Python (if not already configured):**
- Scroll to **"Python"** section
- If not present, Jenkins will use system Python
- **Name**: `Python3`
- **Path**: `/usr/bin/python3`

Click **"Save"**

---

**Installing Required Plugins**

**Step 9: Install Essential Plugins**
1. Go to **Manage Jenkins** → **Manage Plugins**
2. Click **"Available"** tab
3. Search and install these plugins (check the box, then click "Install without restart"):

**Required Plugins:**
- ☑️ **Bitbucket Plugin** (for BitBucket integration)
- ☑️ **Pipeline Plugin** (should already be installed)
- ☑️ **AWS Steps Plugin** (for withAWS functionality)
- ☑️ **Terraform Plugin** (for Terraform integration)
- ☑️ **Blue Ocean Plugin** (for better pipeline visualization)
- ☑️ **JUnit Plugin** (for test result reporting)
- ☑️ **Timestamper Plugin** (adds timestamps to console output)
- ☑️ **Workspace Cleanup Plugin** (for cleanWs() function)

**Recommended Plugins:**
- ☑️ **Build Timeout Plugin** (prevent stuck builds)
- ☑️ **Email Extension Plugin** (for email notifications)
- ☑️ **Slack Notification Plugin** (if using Slack)

4. After selecting plugins, click **"Install without restart"**
5. ☑️ **Check "Restart Jenkins when installation is complete and no jobs are running"**

---

**Configuring System Settings**

**Step 10: Configure System Settings**
1. Go to **Manage Jenkins** → **Configure System**

**Jenkins Location:**
- **Jenkins URL**: Enter your Jenkins URL (important for webhooks)
- **System Admin e-mail address**: Your email

**Global properties:**
- ☑️ **Check "Environment variables"**
- Click **"Add"**
- **Name**: `AWS_DEFAULT_REGION`
- **Value**: `us-east-1`

**E-mail Notification (Optional):**
- **SMTP server**: Your SMTP server (e.g., `smtp.gmail.com`)
- **Advanced**: Configure if needed

Click **"Save"**

---

**Testing the Setup**

**Step 11: Test Pipeline Creation**
1. Go to your pipeline project
2. Click **"Build Now"** (left side)
3. Check if it can connect to BitBucket and checkout code
4. If it fails, check the console output for errors

**Step 12: Test AWS Credentials**
Create a simple test job:
1. **New Item** → **"Freestyle project"** → Name: `Test-AWS-Connection`
2. **Build Steps** → **"Execute shell"**
3. Add command: `aws sts get-caller-identity`
4. **Build Environment** → ☑️ **"Use secret text(s) or file(s)"**
5. **Bindings** → **"AWS access key and secret key"**
6. Select your aws-dev-credentials
7. Save and build to test AWS connectivity

---

**BitBucket Webhook Setup**

**Step 13: Configure BitBucket Webhook**
1. In BitBucket, go to your repository
2. **Repository settings** → **Webhooks**
3. **Add webhook**
- **Title**: `Jenkins Pipeline Trigger`
- **URL**: `https://your-jenkins-url/bitbucket-hook/`
  - ⚠️ **Important**: Include the trailing slash `/`
  - Example: `https://jenkins.company.com/bitbucket-hook/`
- **Status**: ☑️ **Active**
- **Triggers**: 
  - ☑️ **Repository push**
  - ☑️ **Pull request created**
  - ☑️ **Pull request updated**
4. **Save**

---

**Security Configuration**

**Step 14: Configure Security (Important)**
1. **Manage Jenkins** → **Configure Global Security**

**Authorization:**
- Select **"Matrix-based security"** or **"Project-based Matrix Authorization Strategy"**
- Add your username with all permissions
- For other users, give appropriate permissions

**CSRF Protection:**
- ☑️ **Check "Prevent Cross Site Request Forgery exploits"**

---

**Final Checklist**

**Before running your first pipeline:**
- ✅ BitBucket repository has correct structure
- ✅ Jenkinsfile exists in repository root
- ✅ AWS credentials are configured in Jenkins
- ✅ All required plugins are installed
- ✅ BitBucket webhook is configured
- ✅ S3 buckets exist for Terraform state
- ✅ DynamoDB tables exist for Terraform locking
- ✅ IAM users have correct policies

**Test Pipeline:**
1. Make a small commit to your repository
2. Check if Jenkins automatically triggers a build
3. Monitor the console output for any issues
4. Fix any configuration problems before proceeding

This setup provides a solid foundation for your Jenkins serverless CI/CD pipeline with proper security, credentials management, and integration with BitBucket and AWS.
