# 🤖 **Fill-in-the-Blank Prompt Templates for GitHub Copilot**

Perfect! Here are reusable prompt templates that app teams can fill out and submit to GitHub Copilot.

---

## **📋 Unit Testing Prompt Template**

```
Generate a unit-tests.json file for my AWS Lambda function with comprehensive test scenarios.

MY LAMBDA FUNCTION DETAILS:
- Function Name: [FILL: e.g., task-management-lambda]
- Main Operations: [FILL: e.g., Submit, RetrieveTaskById, Update, Delete, HealthCheck]
- Table Names: [FILL: e.g., tasks, jobs, audit_logs]
- Primary Key Field: [FILL: e.g., task_id, job_id, id]
- Authentication: [FILL: API_KEY / JWT / NONE]
- Main Parameters: [FILL: e.g., table_name, pkey_id, status, user_id]

GENERATE TEST CASES FOR:
✅ Each operation with valid inputs (expect 200)
✅ Each operation with missing required parameters (expect 400)
✅ Invalid operations (expect 400)
✅ Authentication failures if applicable (expect 401)
✅ Edge cases specific to my business logic

USE THIS EXACT JSON STRUCTURE:
{
  "scenarios": [
    {
      "name": "descriptive-test-name",
      "event": {
        "queryStringParameters": {
          "operation": "operation_name",
          "table_name": "table_name",
          "pkey_id": "value"
        },
        "body": "json_string_when_needed"
      },
      "expected": {
        "statusCode": 200,
        "bodyContains": ["expected_content"]
      }
    }
  ]
}

NAMING CONVENTION: Use format "OperationName-Scenario-Type" 
(e.g., "Submit-Valid-Request", "RetrieveTaskById-Missing-ID")

GENERATE 10-15 comprehensive test scenarios covering all my operations and edge cases.
Save as: tests/config/unit-tests.json
```

---

## **🔄 Integration Testing Prompt Template**

```
Generate an integration-tests.json file for my AWS Lambda function with end-to-end workflow tests.

MY LAMBDA FUNCTION DETAILS:
- Function Name: [FILL: e.g., task-management-lambda]
- Main Operations: [FILL: e.g., Submit, RetrieveTaskById, Update, Delete, HealthCheck]
- Table Names: [FILL: e.g., tasks, jobs, audit_logs]
- Primary Key Field: [FILL: e.g., task_id, job_id, id]
- Business Workflows: [FILL: e.g., "Create task → Retrieve → Update status → Verify"]
- Data Dependencies: [FILL: e.g., "Tasks depend on Jobs", "Audit logs created after updates"]

CREATE THESE WORKFLOW TYPES:
1. [FILL: e.g., Complete-Task-Lifecycle-Workflow]
   - Description: [FILL: e.g., Test creating, retrieving, updating, and verifying a task]
   - Steps: [FILL: e.g., Submit → Wait → Retrieve → Update → Wait → Verify]

2. [FILL: e.g., Data-Retrieval-Workflow]
   - Description: [FILL: e.g., Test various data retrieval operations]
   - Steps: [FILL: e.g., Get list → Get by ID → Get distinct values]

3. [FILL: e.g., Error-Recovery-Workflow]
   - Description: [FILL: e.g., Test error handling and system recovery]
   - Steps: [FILL: e.g., Try invalid operation → Verify error → Test health check]

USE THIS EXACT JSON STRUCTURE:
{
  "workflows": [
    {
      "name": "workflow-name",
      "description": "business scenario description",
      "steps": [
        {
          "name": "step-name",
          "type": "invoke_lambda",
          "payload": {
            "queryStringParameters": {
              "operation": "operation_name",
              "table_name": "table_name"
            },
            "body": "json_string_when_needed"
          },
          "expect": {
            "statusCode": 200,
            "bodyContains": ["expected_content"]
          }
        },
        {
          "name": "wait-for-processing",
          "type": "wait",
          "seconds": 2
        }
      ]
    }
  ]
}

REQUIREMENTS:
- Create 3-5 realistic business workflows
- Each workflow should have 3-8 steps
- Add wait steps between data-modifying operations
- Use timestamp placeholders: ${timestamp}
- Include error scenarios and recovery steps
- Test end-to-end business processes

Save as: tests/config/integration-tests.json
```

---

## **🎯 Quick Reference Template Card**

**Print this out for your team:**

```
╔══════════════════════════════════════════════════════════════╗
║                    COPILOT TEST GENERATION                   ║
║                       QUICK REFERENCE                        ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  BEFORE USING PROMPTS, FILL OUT:                            ║
║                                                              ║
║  📋 Function Name: ________________________                 ║
║                                                              ║
║  🔧 Operations: ___________________________                 ║
║      (e.g., Submit, Retrieve, Update, Delete)               ║
║                                                              ║
║  🗃️  Tables: ______________________________                 ║
║      (e.g., tasks, jobs, users)                             ║
║                                                              ║
║  🔑 Primary Key: ___________________________                ║
║      (e.g., task_id, user_id, id)                          ║
║                                                              ║
║  🔐 Auth Type: _____________________________                ║
║      (API_KEY / JWT / NONE)                                 ║
║                                                              ║
║  📊 Main Params: ___________________________                ║
║      (e.g., table_name, status, user_id)                   ║
║                                                              ║
║  🔄 Business Flows: ________________________                ║
║      (e.g., "Create → Update → Verify")                    ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

---

## **📝 Example Filled Template**

**Here's how a developer would fill it out:**

### **Unit Testing Example:**
```
Generate a unit-tests.json file for my AWS Lambda function with comprehensive test scenarios.

MY LAMBDA FUNCTION DETAILS:
- Function Name: employee-management-lambda
- Main Operations: CreateEmployee, GetEmployeeById, UpdateEmployee, DeleteEmployee, GetAllEmployees, HealthCheck
- Table Names: employees, departments
- Primary Key Field: employee_id
- Authentication: API_KEY
- Main Parameters: table_name, employee_id, department_id, status

GENERATE TEST CASES FOR:
✅ Each operation with valid inputs (expect 200)
✅ Each operation with missing required parameters (expect 400)
✅ Invalid operations (expect 400)
✅ Authentication failures if applicable (expect 401)
✅ Edge cases specific to my business logic

[REST OF TEMPLATE REMAINS THE SAME]
```

### **Integration Testing Example:**
```
Generate an integration-tests.json file for my AWS Lambda function with end-to-end workflow tests.

MY LAMBDA FUNCTION DETAILS:
- Function Name: employee-management-lambda
- Main Operations: CreateEmployee, GetEmployeeById, UpdateEmployee, DeleteEmployee, GetAllEmployees
- Table Names: employees, departments
- Primary Key Field: employee_id
- Business Workflows: "Create employee → Retrieve → Update status → Verify update"
- Data Dependencies: "Employees belong to departments"

CREATE THESE WORKFLOW TYPES:
1. Complete-Employee-Lifecycle-Workflow
   - Description: Test creating, retrieving, updating, and verifying an employee record
   - Steps: CreateEmployee → Wait → GetEmployeeById → UpdateEmployee → Wait → GetEmployeeById

2. Employee-Department-Workflow
   - Description: Test employee and department relationship operations
   - Steps: GetAllEmployees → GetEmployeeById → Update department → Verify

3. Error-Recovery-Workflow
   - Description: Test error handling and system recovery
   - Steps: Try invalid operation → Verify error → Test health check → Verify recovery

[REST OF TEMPLATE REMAINS THE SAME]
```

---

## **📁 Template Files for Teams**

**Create these template files in your project:**

**`templates/unit-tests-prompt.md`:**
```markdown
# Unit Tests Prompt Template

Fill out the sections below, then copy the entire content to GitHub Copilot:

## MY LAMBDA FUNCTION DETAILS:
- Function Name: [YOUR_FUNCTION_NAME]
- Main Operations: [LIST_YOUR_OPERATIONS]
- Table Names: [LIST_YOUR_TABLES]
- Primary Key Field: [YOUR_PRIMARY_KEY]
- Authentication: [API_KEY/JWT/NONE]
- Main Parameters: [LIST_MAIN_PARAMETERS]

[Include the full unit testing prompt template here]
```

**`templates/integration-tests-prompt.md`:**
```markdown
# Integration Tests Prompt Template

Fill out the sections below, then copy the entire content to GitHub Copilot:

## MY LAMBDA FUNCTION DETAILS:
- Function Name: [YOUR_FUNCTION_NAME]
- Main Operations: [LIST_YOUR_OPERATIONS]
- Table Names: [LIST_YOUR_TABLES]
- Primary Key Field: [YOUR_PRIMARY_KEY]
- Business Workflows: [DESCRIBE_YOUR_WORKFLOWS]
- Data Dependencies: [DESCRIBE_DEPENDENCIES]

[Include the full integration testing prompt template here]
```

**This approach gives teams a structured, fill-in-the-blank template that generates consistent, comprehensive test files across all Lambda projects!** 🚀
