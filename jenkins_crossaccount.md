## Cross-Account Access: Technical Overview

### **The Architecture**

**Trust Relationship Setup:**
• Jenkins runs in AWS Account A with its own IAM service role
• Target AWS Account B creates a "deployment role" that trusts Account A's Jenkins role
• Jenkins uses AWS STS (Security Token Service) to assume the target account role
• Receives temporary credentials (15 minutes to 12 hours) for operations

### **How It Works**

**Authentication Flow:**
1. Jenkins authenticates with its home account using EC2 instance profile
2. Calls AWS STS AssumeRole API targeting the cross-account deployment role
3. AWS validates the trust relationship and issues temporary credentials
4. Jenkins uses these credentials to deploy/test in the target account
5. Credentials automatically expire - no cleanup needed

**Pipeline Integration:**
• AWS CLI plugin handles role assumption transparently
• Terraform can use assumed role credentials automatically
• Same Jenkins job can deploy to multiple accounts sequentially
• Each account access is isolated with specific permissions

### **Security Model**

**Permissions Structure:**
• Cross-account role has minimal permissions (only Lambda, S3, API Gateway operations)
• No permanent credentials stored in Jenkins
• All operations logged to CloudTrail in both accounts
• Role assumption requires both accounts to explicitly trust each other

**Advantages Over Alternatives:**
• **vs API Keys**: No credential rotation, no storage security risks
• **vs Separate Jenkins**: One system, consistent process, easier maintenance
• **vs Manual Process**: Automated, auditable, repeatable deployments

### **Operational Benefits**

**For Multiple Environments:**
• Same pipeline template works for dev, staging, production accounts
• Environment-specific configurations without code changes
• Centralized deployment monitoring and logging

**For Team Workflow:**
• Developers don't need access to production AWS accounts
• All deployments go through controlled pipeline process
• Consistent deployment process regardless of target environment

### **Implementation Scope**

**What Jenkins Needs:**
• AWS CLI plugin (already standard)
• Cross-account IAM roles configured in target accounts
• Pipeline scripts updated to use role assumption

**What We Avoid:**
• No credential management systems
• No VPN or network connectivity between accounts
• No duplicate Jenkins infrastructure per account

This approach leverages AWS's built-in cross-account security model - it's the standard enterprise pattern for multi-account deployments.
