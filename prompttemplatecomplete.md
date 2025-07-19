# 🤖 **Universal Prompt Templates for Lambda Test Generation**

Here are comprehensive, generalized prompt templates that work for any Lambda function and produce thorough test coverage.

---

## **📋 Unit Test Generation Prompt Template**

```
Generate a comprehensive unit-tests.json file for my AWS Lambda function. Analyze my code and create exhaustive test scenarios covering all possible cases.

ANALYZE MY LAMBDA FUNCTION FOR:
• All operations/functions defined in the code
• Input parameters and validation requirements  
• Expected response formats and status codes
• Error handling and edge cases
• Authentication/authorization if present

CREATE UNIT TEST SCENARIOS FOR:
✅ SUCCESS CASES (200 responses):
   - Valid requests with all required parameters
   - Valid requests with optional parameters
   - Boundary value testing (min/max values)
   - Different data types and formats

✅ CLIENT ERROR CASES (400 responses):
   - Missing required parameters
   - Invalid parameter values
   - Malformed request body
   - Invalid data types
   - Parameter validation failures

✅ AUTHENTICATION ERRORS (401/403):
   - Missing authentication headers
   - Invalid API keys/tokens
   - Expired credentials
   - Insufficient permissions

✅ SERVER ERROR CASES (500 responses):
   - Database connection failures
   - External service timeouts
   - Unexpected exceptions
   - Resource limitations

✅ EDGE CASES:
   - Empty request body
   - Null/undefined values
   - Special characters in inputs
   - Large payload testing
   - Concurrent request scenarios

USE THIS EXACT JSON STRUCTURE:
{
  "scenarios": [
    {
      "name": "descriptive-test-name",
      "description": "what this test validates",
      "event": {
        "httpMethod": "GET|POST|PUT|DELETE",
        "headers": {
          "Content-Type": "application/json",
          "Authorization": "Bearer token_if_needed"
        },
        "queryStringParameters": {
          "param1": "value1",
          "param2": "value2"
        },
        "pathParameters": {
          "id": "path_value_if_needed"
        },
        "body": "json_string_when_needed"
      },
      "expected": {
        "statusCode": 200,
        "bodyContains": ["expected_content"],
        "bodyNotContains": ["should_not_contain"],
        "headers": {
          "Content-Type": "application/json"
        }
      }
    }
  ]
}

NAMING CONVENTION:
- Format: "Operation-Scenario-Type"
- Examples: "GetUser-Valid-Request", "CreateTask-Missing-Title", "DeleteItem-Unauthorized"

REQUIREMENTS:
🎯 Generate 15-25 comprehensive test scenarios
🎯 Cover ALL operations found in my code
🎯 Include positive, negative, and edge cases
🎯 Use realistic test data and parameters
🎯 Add proper HTTP methods based on operations
🎯 Include authentication scenarios if applicable
🎯 Cover all possible error conditions
🎯 Use descriptive names and descriptions
🎯 Include boundary testing scenarios

Save as: tests/config/unit-tests.json
```

---

## **🔄 Integration Test Generation Prompt Template**

```
Generate a comprehensive integration-tests.json file for my AWS Lambda function. Create realistic end-to-end workflow tests that simulate real user scenarios and business processes.

ANALYZE MY LAMBDA FUNCTION FOR:
• Business workflows and user journeys
• Data dependencies between operations
• External service integrations
• State changes and data persistence
• Error recovery mechanisms

CREATE INTEGRATION WORKFLOWS FOR:

🔄 COMPLETE BUSINESS WORKFLOWS:
   - Full CRUD lifecycle (Create → Read → Update → Delete → Verify)
   - Multi-step business processes
   - Data validation workflows
   - User journey simulations

🔄 DATA FLOW WORKFLOWS:
   - Cross-operation data dependencies
   - State transition testing
   - Data consistency validation
   - Cascade operation testing

🔄 ERROR RECOVERY WORKFLOWS:
   - Failure scenario testing
   - System resilience validation
   - Rollback mechanism testing
   - Health check and recovery

🔄 INTEGRATION WORKFLOWS:
   - External service integration testing
   - Database operation workflows
   - API endpoint chain testing
   - Cross-system data flow

🔄 PERFORMANCE WORKFLOWS:
   - Load testing scenarios
   - Concurrent operation testing
   - Resource utilization validation
   - Timeout and retry testing

USE THIS EXACT JSON STRUCTURE:
{
  "workflows": [
    {
      "name": "workflow-name",
      "description": "detailed description of business scenario",
      "timeout": 300,
      "steps": [
        {
          "name": "descriptive-step-name",
          "description": "what this step accomplishes",
          "type": "invoke_lambda",
          "retry": 3,
          "payload": {
            "httpMethod": "POST",
            "headers": {
              "Content-Type": "application/json",
              "Authorization": "Bearer ${auth_token}"
            },
            "queryStringParameters": {
              "operation": "operation_name",
              "param1": "${dynamic_value}"
            },
            "pathParameters": {
              "id": "${generated_id}"
            },
            "body": "{\"key\": \"value\", \"timestamp\": \"${timestamp}\"}"
          },
          "expect": {
            "statusCode": 200,
            "bodyContains": ["success", "created"],
            "bodyNotContains": ["error", "failed"],
            "responseTime": 5000,
            "saveResponse": {
              "taskId": "$.body.data.id",
              "status": "$.body.data.status"
            }
          }
        },
        {
          "name": "wait-for-processing",
          "description": "allow system to process the request",
          "type": "wait",
          "seconds": 3
        },
        {
          "name": "verify-operation",
          "description": "verify the previous operation completed successfully",
          "type": "invoke_lambda",
          "payload": {
            "httpMethod": "GET",
            "queryStringParameters": {
              "operation": "retrieve",
              "id": "${taskId}"
            }
          },
          "expect": {
            "statusCode": 200,
            "bodyContains": ["${taskId}", "${status}"]
          }
        }
      ]
    }
  ]
}

WORKFLOW TYPES TO INCLUDE:

1. "Complete-Resource-Lifecycle"
   - Create resource → Verify creation → Update resource → Verify update → Delete resource → Verify deletion

2. "Data-Retrieval-And-Manipulation"
   - Get list → Get specific item → Filter/search → Sort → Paginate → Verify results

3. "Error-Handling-And-Recovery"
   - Trigger error condition → Verify error response → Attempt recovery → Verify system health

4. "Authentication-And-Authorization"
   - Test with valid credentials → Test with invalid credentials → Test permission levels → Verify access control

5. "Performance-And-Reliability"
   - Sequential operations → Concurrent operations → Load testing → Timeout scenarios

DYNAMIC VALUES TO USE:
• ${timestamp} - Current timestamp
• ${uuid} - Generated unique ID
• ${auth_token} - Authentication token
• ${random_string} - Random test data
• ${generated_id} - ID from previous step
• ${dynamic_value} - Value saved from previous response

REQUIREMENTS:
🎯 Generate 5-8 comprehensive workflows
🎯 Each workflow should have 4-10 realistic steps
🎯 Include wait steps between data-modifying operations
🎯 Use dynamic values and response chaining
🎯 Cover complete business scenarios end-to-end
🎯 Include error scenarios and recovery testing
🎯 Add proper timeouts and retry mechanisms
🎯 Test realistic user journeys and workflows
🎯 Include performance and reliability scenarios
🎯 Use descriptive names and detailed descriptions

Save as: tests/config/integration-tests.json
```

---

## **🎯 Quick Copilot Instructions File**

**Create `.github/copilot-instructions.md`:**

```markdown
# GitHub Copilot Test Generation Instructions

## Unit Test Generation
When generating unit-tests.json:
- Analyze ALL functions and operations in the Lambda code
- Create 15-25 comprehensive test scenarios
- Cover success (200), client error (400), auth error (401), and server error (500) cases
- Include edge cases and boundary testing
- Use descriptive naming: "Operation-Scenario-Type"
- Include proper HTTP methods and realistic test data
- Test parameter validation and error handling

## Integration Test Generation
When generating integration-tests.json:
- Create 5-8 end-to-end business workflows
- Each workflow should have 4-10 realistic steps
- Include complete CRUD lifecycles
- Use dynamic values and response chaining
- Add wait steps between data operations
- Test error recovery and system resilience
- Include performance and reliability scenarios

## JSON Structure Requirements
- Use exact JSON structures provided in prompt templates
- Include proper HTTP methods, headers, and status codes
- Use dynamic placeholders: ${timestamp}, ${uuid}, ${auth_token}
- Add comprehensive validation in expect sections
- Include timeout and retry configurations for integration tests

## Test Coverage Goals
- 100% operation coverage in unit tests
- Complete business workflow coverage in integration tests
- Error scenario and edge case coverage
- Authentication and authorization testing
- Performance and reliability validation
```

---

## **🚀 Usage Instructions for Teams**

### **Step 1: Copy the appropriate prompt**
- Use unit test prompt for unit-tests.json
- Use integration test prompt for integration-tests.json

### **Step 2: Open GitHub Copilot Chat**
- Paste the complete prompt
- Copilot will analyze your Lambda code
- It will generate comprehensive test scenarios

### **Step 3: Review and customize**
- Verify test scenarios match your business logic
- Add any specific edge cases for your domain
- Ensure test data is realistic and appropriate

### **Step 4: Save files**
- Save as `tests/config/unit-tests.json`
- Save as `tests/config/integration-tests.json`

**These templates are designed to work with ANY Lambda function and generate production-ready, comprehensive test suites automatically!** 🚀

# 📋 **Test Generation Template Files for App Teams**

Here are the actual template files that app teams can use. Create these files in your project repository:

---

## **📁 File Structure for Teams**

```
project-root/
├── templates/
│   ├── unit-tests-prompt-template.md
│   ├── integration-tests-prompt-template.md
│   └── README.md
├── .github/
│   └── copilot-instructions.md
└── tests/
    └── config/
        ├── unit-tests.json          (generated)
        └── integration-tests.json   (generated)
```

---

## **📄 templates/unit-tests-prompt-template.md**

```markdown
# Unit Tests Generation Template

## Instructions
1. Fill out the sections marked with [FILL THIS]
2. Copy the entire completed prompt below
3. Paste into GitHub Copilot Chat
4. Save the generated JSON as `tests/config/unit-tests.json`

---

## PROMPT TEMPLATE (Fill out and copy everything below this line)

Generate a comprehensive unit-tests.json file for my AWS Lambda function. Analyze my code and create exhaustive test scenarios covering all possible cases.

### MY LAMBDA FUNCTION DETAILS:
- **Function Name**: [FILL THIS: e.g., user-management-lambda]
- **Primary Operations**: [FILL THIS: e.g., CreateUser, GetUser, UpdateUser, DeleteUser, ListUsers]
- **HTTP Methods Used**: [FILL THIS: e.g., GET, POST, PUT, DELETE]
- **Authentication Type**: [FILL THIS: API_KEY / JWT / COGNITO / NONE]
- **Main Parameters**: [FILL THIS: e.g., user_id, email, role, department]
- **Database Tables**: [FILL THIS: e.g., users, profiles, audit_logs]
- **Expected Response Format**: [FILL THIS: JSON / XML / Plain Text]

### SPECIFIC TEST SCENARIOS TO INCLUDE:
[FILL THIS - Add any specific business rules or edge cases unique to your function]
- Example: "Test email format validation"
- Example: "Test role-based access control"
- Example: "Test duplicate email prevention"

ANALYZE MY LAMBDA FUNCTION FOR:
• All operations/functions defined in the code
• Input parameters and validation requirements  
• Expected response formats and status codes
• Error handling and edge cases
• Authentication/authorization if present

CREATE UNIT TEST SCENARIOS FOR:
✅ SUCCESS CASES (200 responses):
   - Valid requests with all required parameters
   - Valid requests with optional parameters
   - Boundary value testing (min/max values)
   - Different data types and formats

✅ CLIENT ERROR CASES (400 responses):
   - Missing required parameters
   - Invalid parameter values
   - Malformed request body
   - Invalid data types
   - Parameter validation failures

✅ AUTHENTICATION ERRORS (401/403):
   - Missing authentication headers
   - Invalid API keys/tokens
   - Expired credentials
   - Insufficient permissions

✅ SERVER ERROR CASES (500 responses):
   - Database connection failures
   - External service timeouts
   - Unexpected exceptions
   - Resource limitations

✅ EDGE CASES:
   - Empty request body
   - Null/undefined values
   - Special characters in inputs
   - Large payload testing
   - Concurrent request scenarios

USE THIS EXACT JSON STRUCTURE:
{
  "scenarios": [
    {
      "name": "descriptive-test-name",
      "description": "what this test validates",
      "event": {
        "httpMethod": "GET|POST|PUT|DELETE",
        "headers": {
          "Content-Type": "application/json",
          "Authorization": "Bearer token_if_needed"
        },
        "queryStringParameters": {
          "param1": "value1",
          "param2": "value2"
        },
        "pathParameters": {
          "id": "path_value_if_needed"
        },
        "body": "json_string_when_needed"
      },
      "expected": {
        "statusCode": 200,
        "bodyContains": ["expected_content"],
        "bodyNotContains": ["should_not_contain"],
        "headers": {
          "Content-Type": "application/json"
        }
      }
    }
  ]
}

NAMING CONVENTION:
- Format: "Operation-Scenario-Type"
- Examples: "GetUser-Valid-Request", "CreateUser-Missing-Email", "DeleteUser-Unauthorized"

REQUIREMENTS:
🎯 Generate 15-25 comprehensive test scenarios
🎯 Cover ALL operations found in my code
🎯 Include positive, negative, and edge cases
🎯 Use realistic test data and parameters
🎯 Add proper HTTP methods based on operations
🎯 Include authentication scenarios if applicable
🎯 Cover all possible error conditions
🎯 Use descriptive names and descriptions
🎯 Include boundary testing scenarios

Save as: tests/config/unit-tests.json
```

---

## **📄 templates/integration-tests-prompt-template.md**

```markdown
# Integration Tests Generation Template

## Instructions
1. Fill out the sections marked with [FILL THIS]
2. Copy the entire completed prompt below
3. Paste into GitHub Copilot Chat
4. Save the generated JSON as `tests/config/integration-tests.json`

---

## PROMPT TEMPLATE (Fill out and copy everything below this line)

Generate a comprehensive integration-tests.json file for my AWS Lambda function. Create realistic end-to-end workflow tests that simulate real user scenarios and business processes.

### MY LAMBDA FUNCTION DETAILS:
- **Function Name**: [FILL THIS: e.g., task-management-lambda]
- **Primary Operations**: [FILL THIS: e.g., CreateTask, GetTask, UpdateTask, DeleteTask, ListTasks]
- **Business Workflows**: [FILL THIS: e.g., "Create task → Assign → Update status → Complete → Archive"]
- **Data Dependencies**: [FILL THIS: e.g., "Tasks belong to Projects", "Users can have multiple roles"]
- **External Integrations**: [FILL THIS: e.g., Email service, S3 storage, Third-party APIs]
- **State Transitions**: [FILL THIS: e.g., "pending → in-progress → completed → archived"]

### SPECIFIC WORKFLOWS TO TEST:
[FILL THIS - Describe your main business scenarios]
1. [WORKFLOW NAME]: [FILL THIS: e.g., "Complete Task Lifecycle"]
   - Description: [FILL THIS: e.g., "Test creating, updating, and completing a task"]
   - Steps: [FILL THIS: e.g., "Create → Assign → Update → Complete → Verify"]

2. [WORKFLOW NAME]: [FILL THIS: e.g., "User Permission Testing"]
   - Description: [FILL THIS: e.g., "Test different user roles and permissions"]
   - Steps: [FILL THIS: e.g., "Login admin → Create → Login user → Try access → Verify permissions"]

3. [WORKFLOW NAME]: [FILL THIS: e.g., "Error Recovery Testing"]
   - Description: [FILL THIS: e.g., "Test system recovery from failures"]
   - Steps: [FILL THIS: e.g., "Trigger error → Verify error handling → Retry → Verify recovery"]

ANALYZE MY LAMBDA FUNCTION FOR:
• Business workflows and user journeys
• Data dependencies between operations
• External service integrations
• State changes and data persistence
• Error recovery mechanisms

CREATE INTEGRATION WORKFLOWS FOR:

🔄 COMPLETE BUSINESS WORKFLOWS:
   - Full CRUD lifecycle (Create → Read → Update → Delete → Verify)
   - Multi-step business processes
   - Data validation workflows
   - User journey simulations

🔄 DATA FLOW WORKFLOWS:
   - Cross-operation data dependencies
   - State transition testing
   - Data consistency validation
   - Cascade operation testing

🔄 ERROR RECOVERY WORKFLOWS:
   - Failure scenario testing
   - System resilience validation
   - Rollback mechanism testing
   - Health check and recovery

🔄 INTEGRATION WORKFLOWS:
   - External service integration testing
   - Database operation workflows
   - API endpoint chain testing
   - Cross-system data flow

🔄 PERFORMANCE WORKFLOWS:
   - Load testing scenarios
   - Concurrent operation testing
   - Resource utilization validation
   - Timeout and retry testing

USE THIS EXACT JSON STRUCTURE:
{
  "workflows": [
    {
      "name": "workflow-name",
      "description": "detailed description of business scenario",
      "timeout": 300,
      "steps": [
        {
          "name": "descriptive-step-name",
          "description": "what this step accomplishes",
          "type": "invoke_lambda",
          "retry": 3,
          "payload": {
            "httpMethod": "POST",
            "headers": {
              "Content-Type": "application/json",
              "Authorization": "Bearer ${auth_token}"
            },
            "queryStringParameters": {
              "operation": "operation_name",
              "param1": "${dynamic_value}"
            },
            "pathParameters": {
              "id": "${generated_id}"
            },
            "body": "{\"key\": \"value\", \"timestamp\": \"${timestamp}\"}"
          },
          "expect": {
            "statusCode": 200,
            "bodyContains": ["success", "created"],
            "bodyNotContains": ["error", "failed"],
            "responseTime": 5000,
            "saveResponse": {
              "taskId": "$.body.data.id",
              "status": "$.body.data.status"
            }
          }
        },
        {
          "name": "wait-for-processing",
          "description": "allow system to process the request",
          "type": "wait",
          "seconds": 3
        }
      ]
    }
  ]
}

DYNAMIC VALUES TO USE:
• ${timestamp} - Current timestamp
• ${uuid} - Generated unique ID
• ${auth_token} - Authentication token
• ${random_string} - Random test data
• ${generated_id} - ID from previous step
• ${dynamic_value} - Value saved from previous response

REQUIREMENTS:
🎯 Generate 5-8 comprehensive workflows
🎯 Each workflow should have 4-10 realistic steps
🎯 Include wait steps between data-modifying operations
🎯 Use dynamic values and response chaining
🎯 Cover complete business scenarios end-to-end
🎯 Include error scenarios and recovery testing
🎯 Add proper timeouts and retry mechanisms
🎯 Test realistic user journeys and workflows
🎯 Include performance and reliability scenarios
🎯 Use descriptive names and detailed descriptions

Save as: tests/config/integration-tests.json
```

---

## **📄 templates/README.md**

```markdown
# 🤖 Test Generation Templates

This directory contains prompt templates for generating comprehensive test files using GitHub Copilot.

## 📋 Quick Start

### Step 1: Choose Your Template
- **unit-tests-prompt-template.md** - For generating unit-tests.json
- **integration-tests-prompt-template.md** - For generating integration-tests.json

### Step 2: Fill Out the Template
Open the template file and fill in all sections marked with `[FILL THIS]`:
- Function details (name, operations, parameters)
- Business workflows and scenarios
- Specific test cases for your domain

### Step 3: Generate Tests
1. Copy the completed prompt from the template
2. Open GitHub Copilot Chat in VS Code (Ctrl+Shift+I)
3. Paste the prompt and let Copilot analyze your code
4. Review and save the generated JSON files

### Step 4: Save Files
- Save unit tests as: `tests/config/unit-tests.json`
- Save integration tests as: `tests/config/integration-tests.json`

## 📝 Example: Filled Template

### Before (Template):
```
- **Function Name**: [FILL THIS: e.g., user-management-lambda]
- **Primary Operations**: [FILL THIS: e.g., CreateUser, GetUser, UpdateUser]
```

### After (Filled):
```
- **Function Name**: employee-payroll-lambda
- **Primary Operations**: CreateEmployee, GetEmployee, UpdateSalary, DeleteEmployee, GeneratePayslip
```

## 🎯 Tips for Better Results

### Be Specific
- List ALL your function operations
- Include exact parameter names
- Describe your business logic

### Include Edge Cases
- Mention specific validation rules
- Add domain-specific scenarios
- Include error conditions unique to your function

### Test Data
- Use realistic test values
- Include boundary conditions
- Consider your actual data formats

## 🔧 Troubleshooting

**Q: Copilot didn't generate enough test cases**
A: Be more specific about your operations and add more detail to the [FILL THIS] sections

**Q: Tests don't match my business logic**
A: Add more specific business scenarios in the "SPECIFIC TEST SCENARIOS" section

**Q: Missing authentication tests**
A: Make sure to specify your authentication type and requirements

## 📞 Support
For questions about these templates, contact the DevOps team or check the project documentation.
```

---

## **📄 Example: Completed Template**

**Here's how a developer would fill out the unit test template:**

```markdown
### MY LAMBDA FUNCTION DETAILS:
- **Function Name**: employee-management-lambda
- **Primary Operations**: CreateEmployee, GetEmployeeById, UpdateEmployee, DeleteEmployee, ListEmployees, GetEmployeesByDepartment
- **HTTP Methods Used**: GET, POST, PUT, DELETE
- **Authentication Type**: JWT
- **Main Parameters**: employee_id, email, department_id, salary, hire_date
- **Database Tables**: employees, departments, salaries
- **Expected Response Format**: JSON

### SPECIFIC TEST SCENARIOS TO INCLUDE:
- Test email format validation (must be company domain)
- Test salary range validation (minimum wage compliance)
- Test department existence validation
- Test employee ID uniqueness
- Test hire date cannot be future date
- Test role-based access (only HR can see salaries)
```

**This approach gives teams ready-to-use templates that they just fill in the blanks and get comprehensive test files generated automatically!** 🚀
