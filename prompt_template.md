# 🤖 **Complete Test Suite Generator Prompts for GitHub Copilot**

Here are the master prompts that will generate complete `unit-tests.json` and `integration-tests.json` files:

---

## **🧪 Unit Tests Generator Prompt**

```
Generate a complete unit-tests.json file for our Lambda RDS testing pipeline:

**System Context:**
- Lambda Function Pattern: euc-lambda-poc
- Database: RDS MySQL with tables (test_tasks, test_jobs, test_audit)
- Testing Framework: Jenkins + boto3 + PyYAML
- AWS Region: ap-northeast-1
- Pipeline: Automated CI/CD testing

**Lambda Operations to Test:**
1. RetrieveTaskById - Fetch task by ID from test_tasks table
2. RetrieveJobList - Get all jobs from test_jobs table
3. RetrieveDistinctColumns - Get unique values from specified column
4. Submit - Insert new record into test_tasks table
5. Update - Modify existing record in test_tasks table
6. RetrieveAuditTrail - Get audit records
7. Health Check - Basic connectivity test

**Required Test Coverage:**
- Success scenarios (valid inputs, expected responses)
- Error scenarios (invalid inputs, missing parameters)
- Database connectivity validation
- Response format verification
- Edge cases and boundary conditions

**Template Structure:**
Create a JSON file with this exact structure:

```json
{
  "scenarios": [
    {
      "name": "Test-Name-Here",
      "event": {
        "operation": "OperationName",
        "queryStringParameters": {
          "param1": "value1",
          "param2": "value2"
        },
        "body": "JSON string if needed"
      },
      "expected": {
        "statusCode": 200,
        "bodyContains": ["expected", "text"],
        "requiresDbConnection": true
      },
      "description": "What this test validates",
      "category": "unit",
      "priority": "high"
    }
  ]
}
```

**Generate 20-25 comprehensive unit test scenarios covering:**
- Each operation with valid and invalid inputs
- Missing parameter scenarios
- Database error conditions
- Response validation tests
- Performance boundary tests

Create the complete unit-tests.json file now.
```

---

## **🔗 Integration Tests Generator Prompt**

```
Generate a complete integration-tests.json file for our Lambda RDS workflow testing:

**System Context:**
- Lambda Function Pattern: euc-lambda-poc
- Database: RDS MySQL with test data
- Testing Framework: Jenkins + boto3 + PyYAML
- AWS Region: ap-northeast-1
- Pipeline: End-to-end workflow validation

**Integration Workflows to Create:**
1. Complete Task Management Lifecycle (Submit → Retrieve → Update → Verify)
2. Database Query Operations Workflow (List → Filter → Distinct → Audit)
3. Error Recovery Workflow (Submit → Fail → Retry → Success)
4. Data Consistency Validation (Multi-operation data integrity)
5. Performance Workflow (Concurrent operations testing)

**Template Structure:**
Create a JSON file with this exact structure:

```json
{
  "workflows": [
    {
      "name": "Workflow-Name-Here",
      "description": "What this workflow tests",
      "category": "integration",
      "priority": "high",
      "cleanup_required": true,
      "steps": [
        {
          "name": "Step-Name",
          "type": "invoke_lambda",
          "payload": {
            "operation": "OperationName",
            "queryStringParameters": {
              "param1": "value1"
            },
            "body": "JSON if needed"
          },
          "expect": {
            "statusCode": 200,
            "bodyContains": ["expected", "content"],
            "requiresDbConnection": true
          },
          "description": "What this step does"
        },
        {
          "name": "Wait-Step",
          "type": "wait",
          "seconds": 2,
          "description": "Wait for database commit"
        }
      ]
    }
  ]
}
```

**Workflow Requirements:**
- Use dynamic test data with ${timestamp} and ${uuid} placeholders
- Include proper wait steps for database operations
- Add verification steps after data modifications
- Include cleanup considerations
- Cover both success and failure paths

**Generate 5-7 comprehensive integration workflows covering:**
- Full CRUD operations lifecycle
- Cross-operation data dependencies
- Error handling and recovery
- Data consistency validation
- Performance under load

Create the complete integration-tests.json file now.
```

---

## **🎯 Combined Test Suite Generator Prompt**

```
Generate both unit-tests.json AND integration-tests.json files for our complete Lambda RDS testing suite:

**Application Architecture:**
- Lambda Functions: euc-lambda-poc-* (dev/test/prod environments)
- Database: AWS RDS MySQL
- Tables: test_tasks, test_jobs, test_audit, test_queue
- Authentication: IAM role-based
- Region: ap-northeast-1
- Testing Pipeline: Jenkins CI/CD with boto3

**Database Schema Context:**
```sql
-- test_tasks table
task_id (VARCHAR, PK), status (VARCHAR), description (TEXT), 
created_by (VARCHAR), created_at (TIMESTAMP), updated_at (TIMESTAMP)

-- test_jobs table  
job_id (VARCHAR, PK), job_name (VARCHAR), status (VARCHAR),
priority (INT), created_at (TIMESTAMP)

-- test_audit table
audit_id (INT, PK), table_name (VARCHAR), operation (VARCHAR),
record_id (VARCHAR), timestamp (TIMESTAMP), user_id (VARCHAR)
```

**Lambda Operations Available:**
1. RetrieveTaskById(table_name, pkey_id) → Returns task data
2. RetrieveJobList(table_name, filters?) → Returns job list
3. RetrieveDistinctColumns(table_name, column_name) → Returns unique values
4. Submit(table_name, data) → Inserts new record
5. Update(data) → Updates existing record
6. RetrieveAuditTrail(table_name, record_id?) → Returns audit history
7. Health Check → Basic connectivity test

**TEST FILE 1: unit-tests.json**
Generate comprehensive unit tests with these categories:

**Success Scenarios (60%):**
- Valid data retrieval operations
- Successful data insertion
- Proper data updates
- Correct health checks
- Valid distinct column queries

**Error Scenarios (30%):**
- Missing required parameters
- Invalid table names
- Non-existent record IDs
- Malformed JSON bodies
- Database connection failures

**Edge Cases (10%):**
- Boundary value testing
- Large data payloads
- Special characters in data
- Concurrent access simulation

**TEST FILE 2: integration-tests.json**
Generate end-to-end workflow tests:

**Primary Workflows:**
1. Complete Task Lifecycle (Create→Read→Update→Audit)
2. Job Management Workflow (List→Filter→Process→Track)
3. Data Consistency Validation (Multi-table operations)
4. Error Recovery Testing (Failure→Retry→Success)
5. Audit Trail Verification (Operation→Log→Verify)

**Template Requirements:**
- Use timestamp placeholders: ${timestamp}, ${uuid}, ${build_id}
- Include database wait steps between operations
- Add proper success/failure validation
- Include cleanup steps where needed
- Test both individual operations and workflows

**Output Format:**
Generate TWO complete JSON files:

FILE 1: unit-tests.json (25-30 test scenarios)
FILE 2: integration-tests.json (6-8 comprehensive workflows)

Both files must be production-ready and compatible with our Jenkins pipeline framework.

Create both complete test files now.
```

---

## **⚡ Quick Generation Prompt (for Speed)**

```
Create complete unit-tests.json and integration-tests.json for Lambda RDS testing:

**Context:** Jenkins CI/CD, boto3 testing, RDS MySQL, Lambda function pattern: euc-lambda-poc

**Operations:** RetrieveTaskById, RetrieveJobList, Submit, Update, RetrieveDistinctColumns, Health Check

**Tables:** test_tasks(task_id, status, description), test_jobs(job_id, job_name, status)

**Requirements:**
unit-tests.json: 20+ scenarios (success/error/edge cases)
integration-tests.json: 5+ workflows (end-to-end testing)

**Format:** Standard JSON with scenarios[] and workflows[] arrays

Generate both complete files with realistic test data and proper validation.
```

---

## **🔧 Advanced Customization Prompt**

```
Generate enterprise-grade test suites (unit-tests.json + integration-tests.json) for Lambda RDS application:

**Enterprise Requirements:**
- Compliance testing (data validation, audit trails)
- Performance benchmarks (response time < 3000ms)
- Security validation (SQL injection prevention)
- Error resilience (graceful failure handling)
- Production readiness validation

**Database Context:**
```json
{
  "tables": {
    "test_tasks": {
      "primary_key": "task_id",
      "required_fields": ["task_id", "status", "created_by"],
      "optional_fields": ["description", "priority", "due_date"],
      "constraints": ["status IN ('pending','in_progress','completed','failed')"]
    },
    "test_jobs": {
      "primary_key": "job_id", 
      "required_fields": ["job_id", "job_name"],
      "business_rules": ["active jobs must have valid owner"]
    }
  }
}
```

**Test Categories:**
**unit-tests.json:**
- Functional tests (75%)
- Security tests (15%) 
- Performance tests (10%)

**integration-tests.json:**
- Business process workflows (60%)
- Data consistency workflows (25%)
- Error recovery workflows (15%)

**Quality Standards:**
- Each test must have clear description
- Include expected response times
- Add database validation steps
- Include security boundary testing
- Cover all business logic paths

**Advanced Features:**
- Dynamic test data generation
- Parameterized test scenarios
- Conditional test execution
- Test dependency management
- Automated cleanup procedures

Generate production-ready test suites with enterprise-level coverage and validation.
```

---

## **📋 Usage Instructions for Teams**

### **Step-by-Step Implementation:**

**1. Choose Your Prompt Level:**
- **Basic Team**: Use "Combined Test Suite Generator"
- **Advanced Team**: Use "Advanced Customization Prompt"  
- **Quick Start**: Use "Quick Generation Prompt"

**2. Customize for Your Application:**
```bash
# Replace these placeholders before using:
- euc-lambda-poc → Your actual function pattern
- test_tasks, test_jobs → Your actual table names
- ap-northeast-1 → Your AWS region
- Your specific operations and parameters
```

**3. Generate Files with Copilot:**
```bash
# In your IDE with GitHub Copilot:
1. Create: tests/config/unit-tests.json
2. Paste the unit test prompt as comment
3. Let Copilot generate the complete file
4. Create: tests/config/integration-tests.json  
5. Paste the integration test prompt as comment
6. Let Copilot generate the complete file
```

**4. Validate Generated Files:**
```bash
# Test JSON syntax
python3 -c "import json; print('✅ unit-tests.json valid') if json.load(open('tests/config/unit-tests.json')) else print('❌ Invalid')"
python3 -c "import json; print('✅ integration-tests.json valid') if json.load(open('tests/config/integration-tests.json')) else print('❌ Invalid')"

# Test with pipeline
./tests/run_tests.sh unit
./tests/run_tests.sh integration
```

---

## **📊 Expected Output Structure**

**unit-tests.json (25-30 scenarios):**
```json
{
  "scenarios": [
    {
      "name": "RetrieveTaskById-Valid-Request",
      "event": {...},
      "expected": {...},
      "description": "...",
      "category": "unit",
      "priority": "high"
    },
    // ... 24 more comprehensive test scenarios
  ]
}
```

**integration-tests.json (5-8 workflows):**
```json
{
  "workflows": [
    {
      "name": "Complete-Task-Management-Workflow", 
      "description": "...",
      "steps": [
        {"name": "Submit-Task", "type": "invoke_lambda", ...},
        {"name": "Wait-DB-Commit", "type": "wait", ...},
        {"name": "Retrieve-Task", "type": "invoke_lambda", ...},
        // ... more workflow steps
      ]
    },
    // ... 4-7 more comprehensive workflows
  ]
}
```

**These prompts will generate complete, production-ready test suites that integrate seamlessly with your Jenkins CI/CD pipeline!** 🚀
