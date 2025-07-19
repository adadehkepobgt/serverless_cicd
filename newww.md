# 🎯 **Proper `.prompt.md` Files for GitHub Copilot**

Perfect! Let's create proper prompt files using the official GitHub Copilot structure:

---

## **📄 `.github/prompts/generate-unit-tests.prompt.md`**

```markdown
---
mode: agent
model: gpt-4
tools: ['workspace']
description: Generate comprehensive unit-tests.json for AWS Lambda functions with proper HTTP method constraints
---

# Generate Lambda Unit Tests

## Instructions for User
1. Fill out the template sections below
2. Run this prompt with your Lambda function details
3. Generated file will be saved as `tests/config/unit-tests.json`

---

## Lambda Function Analysis

Analyze the Lambda function in ${workspaceFolder} and generate comprehensive unit test scenarios.

### Function Details Template
Please provide the following information about your Lambda function:

**Function Name**: ${input:functionName:Enter your Lambda function name}
**Primary Operations**: ${input:operations:List main operations (e.g., CreateUser, GetUser, UpdateUser)}
**HTTP Methods**: ${input:httpMethods:Enter ONLY supported HTTP methods (e.g., GET, POST)}
**Authentication**: ${input:authType:Enter auth type (API_KEY/JWT/COGNITO/NONE)}
**Parameters**: ${input:parameters:List main parameters (e.g., user_id, email, role)}
**Database Tables**: ${input:dbTables:List database tables used}
**Business Rules**: ${input:businessRules:Enter specific validation rules or constraints}

### Critical Constraints

⚠️ **HTTP Method Restriction**: Only generate tests for the HTTP methods specified above.
- If only GET/POST are supported, DO NOT create PUT/DELETE/PATCH tests
- GET operations: Use queryStringParameters
- POST operations: Use body with JSON

### Analysis Requirements

Examine the Lambda function code and identify:
- All operations/functions defined
- Input parameter validation patterns
- Expected response formats and status codes
- Error handling mechanisms
- Authentication/authorization patterns

### Test Generation Requirements

Generate 20-25 comprehensive test scenarios covering:

#### ✅ Success Cases (200 responses)
- Valid requests with all required parameters
- Valid requests with optional parameters
- Boundary value testing
- Different valid data formats

#### ❌ Client Errors (400 responses)
- Missing required parameters
- Invalid parameter values
- Malformed request body
- Data type validation failures
- Business rule violations

#### 🔒 Authentication Errors (401/403)
- Missing authentication headers
- Invalid tokens/API keys
- Expired credentials
- Insufficient permissions

#### 💥 Server Errors (500 responses)
- Database connection failures
- External service timeouts
- Unexpected exceptions
- Resource limitations

#### 🎯 Edge Cases
- Empty/null values
- Special characters
- Large payloads
- Boundary conditions

### Output JSON Structure

Create `tests/config/unit-tests.json` with this exact structure:

```json
{
  "scenarios": [
    {
      "name": "Operation-Scenario-Type",
      "description": "Clear description of what this test validates",
      "event": {
        "httpMethod": "GET|POST",
        "headers": {
          "Content-Type": "application/json",
          "Authorization": "Bearer test-token"
        },
        "queryStringParameters": {
          "operation": "operation_name",
          "param": "value"
        },
        "pathParameters": {
          "id": "value_if_needed"
        },
        "body": "json_string_for_post_requests"
      },
      "expected": {
        "statusCode": 200,
        "bodyContains": ["expected_content"],
        "bodyNotContains": ["error_content"],
        "headers": {
          "Content-Type": "application/json"
        }
      }
    }
  ]
}
```

### Naming Convention
- Format: "Operation-Scenario-Type"
- Examples: "GetUser-Valid-Request", "CreateUser-Missing-Email", "UpdateUser-Unauthorized"

### Final Requirements
🎯 Generate realistic test data based on actual code analysis
🎯 Cover ALL operations found in the Lambda function
🎯 Use ONLY the HTTP methods specified in function details
🎯 Include comprehensive error handling scenarios
🎯 Create descriptive test names and descriptions
🎯 Ensure valid JSON syntax
🎯 Save as tests/config/unit-tests.json

Please analyze my Lambda function and generate the comprehensive unit test file now.
```

---

## **📄 `.github/prompts/generate-integration-tests.prompt.md`**

```markdown
---
mode: agent
model: gpt-4
tools: ['workspace']
description: Generate comprehensive integration-tests.json for Lambda workflow testing
---

# Generate Lambda Integration Tests

## Instructions for User
1. Provide your Lambda function workflow details below
2. Run this prompt to generate end-to-end test scenarios
3. Generated file will be saved as `tests/config/integration-tests.json`

---

## Workflow Analysis

Analyze the Lambda function in ${workspaceFolder} and generate realistic end-to-end workflow tests.

### Function Workflow Details

**Function Name**: ${input:functionName:Enter your Lambda function name}
**Supported HTTP Methods**: ${input:httpMethods:Enter ONLY supported methods (e.g., GET, POST)}
**Primary Workflows**: ${input:workflows:Describe main business workflows}
**Data Dependencies**: ${input:dataDeps:Describe how data flows between operations}
**State Transitions**: ${input:stateTransitions:Describe status/state changes}
**External Services**: ${input:externalServices:List external integrations}
**Business Scenarios**: ${input:businessScenarios:Describe key user journeys}

### Critical Constraints

⚠️ **HTTP Method Restriction**: Only use the HTTP methods specified above in ALL workflow steps.
- Never generate workflow steps with unsupported methods
- Ensure request format matches your Lambda's expectations

### Workflow Requirements

Generate 6-8 comprehensive workflow scenarios covering:

#### 🔄 Complete Business Workflows
- Full CRUD lifecycle with verification steps
- Multi-step business processes
- Real user journey simulations
- Data validation workflows

#### 📊 Data Flow Workflows  
- Cross-operation data dependencies
- State transition testing
- Data consistency validation
- Cascade operation testing

#### 🔧 Error Recovery Workflows
- Failure scenario testing
- System resilience validation
- Rollback mechanism testing
- Health check and recovery

#### ⚡ Performance Workflows
- Load testing scenarios
- Concurrent operations
- Resource utilization
- Timeout and retry testing

### Output JSON Structure

Create `tests/config/integration-tests.json` with this structure:

```json
{
  "workflows": [
    {
      "name": "descriptive-workflow-name",
      "description": "Detailed business scenario description",
      "timeout": 300,
      "steps": [
        {
          "name": "step-name",
          "description": "What this step accomplishes",
          "type": "invoke_lambda",
          "retry": 3,
          "payload": {
            "httpMethod": "GET|POST",
            "headers": {
              "Content-Type": "application/json",
              "Authorization": "Bearer ${auth_token}"
            },
            "queryStringParameters": {
              "operation": "operation_name",
              "param": "${dynamic_value}"
            },
            "pathParameters": {
              "id": "${generated_id}"
            },
            "body": "{\"operation\": \"action\", \"data\": {...}}"
          },
          "expect": {
            "statusCode": 200,
            "bodyContains": ["success"],
            "bodyNotContains": ["error"],
            "responseTime": 5000,
            "saveResponse": {
              "variableName": "$.body.data.id"
            }
          }
        },
        {
          "name": "wait-step",
          "description": "Allow processing time",
          "type": "wait",
          "seconds": 3
        }
      ]
    }
  ]
}
```

### Dynamic Variables
- `${timestamp}` - Current timestamp
- `${uuid}` - Generated unique ID
- `${auth_token}` - Authentication token
- `${random_string}` - Random test data
- `${generated_id}` - Saved from previous response

### Workflow Requirements
🎯 4-10 realistic steps per workflow
🎯 Include wait steps between modifications
🎯 Use dynamic values and response chaining
🎯 Test complete business scenarios end-to-end
🎯 Include error and recovery scenarios
🎯 Use ONLY supported HTTP methods
🎯 Add proper timeouts and retry logic
🎯 Create descriptive names and descriptions

Please analyze my Lambda function workflows and generate the comprehensive integration test file now.
```

---

## **📄 `.github/prompts/analyze-lambda-function.prompt.md`**

```markdown
---
mode: agent
model: gpt-4
tools: ['workspace']
description: Comprehensive analysis of Lambda function for test preparation
---

# Analyze Lambda Function

## Analysis Target

Analyze the Lambda function in the current workspace: ${workspaceFolder}

**Target Function**: ${input:functionFile:Enter path to Lambda function file (e.g., src/lambda_function.py)}

---

## Comprehensive Analysis

Please provide a detailed analysis of the specified Lambda function:

### 1. Function Overview
- Function name and main purpose
- Programming language and runtime
- Entry point and handler function
- File structure and organization
- Dependencies and imports

### 2. Operations Inventory
For each operation/function found, provide:
- Operation name and purpose
- HTTP method used (GET, POST, PUT, DELETE, etc.)
- Input parameters (required vs optional)
- Parameter validation rules
- Expected input format
- Output format and structure
- Business logic summary

### 3. Technical Architecture
- Authentication/authorization mechanism
- Database tables and external services used
- Error handling patterns and strategies
- Response formats and status codes
- Configuration and environment variables
- Performance considerations

### 4. API Interface Analysis
- Request routing mechanism
- Parameter parsing (query string vs body vs path)
- Content-Type handling
- Header requirements
- Response structure consistency

### 5. Testing Recommendations

Based on the analysis, recommend:

#### Critical Test Scenarios
- High-priority operations to test
- Complex business logic scenarios
- Data validation edge cases
- Security and permission scenarios

#### HTTP Method Testing Strategy
- Confirmed supported HTTP methods
- Request format for each method
- Parameter passing conventions
- Response expectations

#### Edge Cases and Boundaries
- Input validation limits
- Error condition triggers
- Performance bottlenecks
- Security vulnerabilities

#### Integration Points
- External service dependencies
- Database interaction patterns
- Cross-function data flow
- State management scenarios

### 6. Test Generation Guidance

Provide specific recommendations for:
- Number of unit test scenarios needed
- Key integration workflows to test
- Realistic test data examples
- Authentication test requirements
- Error scenario priorities

### Output Format

Present the analysis in clear, structured sections with:
- Bullet points for easy scanning
- Code examples where helpful
- Specific recommendations for test generation
- Identified risks or concerns
- Actionable next steps

This analysis will be used to generate comprehensive unit and integration tests.
```

---

## **📄 `.github/prompts/validate-test-coverage.prompt.md`**

```markdown
---
mode: agent
model: gpt-4
tools: ['workspace']
description: Validate generated test files for completeness and accuracy
---

# Validate Test Coverage

## Validation Target

Review the generated test files in the current workspace: ${workspaceFolder}

**Unit Tests**: `tests/config/unit-tests.json`
**Integration Tests**: `tests/config/integration-tests.json`
**Source Function**: ${input:functionFile:Enter path to Lambda function file}

---

## Comprehensive Validation

### 1. Unit Test File Validation

Analyze `tests/config/unit-tests.json` and verify:

#### Coverage Completeness
- [ ] All Lambda operations have corresponding tests
- [ ] Success scenarios (200) for each operation
- [ ] Client error scenarios (400) for validation failures
- [ ] Authentication error scenarios (401/403) if applicable
- [ ] Server error scenarios (500) for system failures
- [ ] Edge cases and boundary value testing

#### Technical Accuracy
- [ ] Only supported HTTP methods used
- [ ] Correct request format (GET queryStringParameters vs POST body)
- [ ] Valid JSON syntax throughout
- [ ] Realistic and varied test data
- [ ] Proper error expectations and assertions
- [ ] Consistent naming convention followed

#### Test Quality Assessment
- [ ] Descriptive test names and descriptions
- [ ] Appropriate parameter combinations
- [ ] Comprehensive validation scenarios
- [ ] Proper authentication handling
- [ ] Boundary and edge case coverage

### 2. Integration Test File Validation

Analyze `tests/config/integration-tests.json` and verify:

#### Workflow Completeness
- [ ] Complete business workflows represented
- [ ] Data dependency flows covered
- [ ] Error recovery scenarios included
- [ ] Multi-step operations validated
- [ ] State transitions properly tested

#### Technical Implementation
- [ ] Dynamic variable usage correct
- [ ] Wait steps between data modifications
- [ ] Response chaining configured properly
- [ ] Timeout and retry settings appropriate
- [ ] Valid workflow JSON structure

#### Business Scenario Coverage
- [ ] Realistic user journeys tested
- [ ] Cross-operation data flow validated
- [ ] Performance scenarios included
- [ ] Failure and recovery workflows

### 3. Cross-File Analysis

Compare both files for:
- Consistency in operation names
- Matching HTTP methods and request formats
- Complementary test coverage
- No conflicting test scenarios
- Proper data flow between unit and integration tests

### 4. Function Code Alignment

Verify tests match the actual Lambda function:
- [ ] All function operations are tested
- [ ] Parameter names match function signature
- [ ] Validation rules reflected in tests
- [ ] Error handling scenarios match code
- [ ] Response formats align with function output

### 5. Gap Identification

Identify any missing test scenarios:

#### Unit Test Gaps
- Uncovered operations or functions
- Missing error conditions
- Inadequate boundary testing
- Insufficient validation scenarios
- Authentication edge cases

#### Integration Test Gaps
- Missing business workflows
- Incomplete data flow scenarios
- Lack of error recovery testing
- Missing performance scenarios
- Inadequate state transition testing

### 6. Quality Recommendations

Provide specific suggestions for:

#### Improvements Needed
- Additional test scenarios to add
- Test data quality enhancements
- Structural optimizations
- Coverage gap resolution

#### Best Practice Alignment
- Naming convention improvements
- Documentation enhancements
- Test organization suggestions
- Maintenance considerations

### 7. Validation Summary

Provide a comprehensive summary including:
- Overall test coverage percentage
- Critical gaps requiring immediate attention
- Quality score and recommendations
- Specific action items for improvement
- Readiness assessment for deployment

### Output Format

Present validation results as:
- Clear checklist with pass/fail status
- Specific recommendations for each gap
- Prioritized action items
- Code examples for improvements where helpful
- Summary assessment and next steps
```

---

## **🚀 How to Use These Prompt Files**

### **Method 1: Direct Execution**
```bash
# In VS Code Command Palette (Ctrl+Shift+P)
> GitHub Copilot: Run Prompt

# Select: .github/prompts/generate-unit-tests.prompt.md
# Fill in the prompted values
```

### **Method 2: Chat Reference**
```bash
# In GitHub Copilot Chat
@workspace Run the prompt file .github/prompts/generate-unit-tests.prompt.md for my Lambda function
```

### **Method 3: Automated Workflow**
```bash
# Sequential execution
@workspace Run these prompts in order:
1. .github/prompts/analyze-lambda-function.prompt.md
2. .github/prompts/generate-unit-tests.prompt.md  
3. .github/prompts/generate-integration-tests.prompt.md
4. .github/prompts/validate-test-coverage.prompt.md
```

---

## **💡 Benefits of This Structure**

1. **Interactive Prompts**: Input variables make prompts reusable
2. **Mode Control**: Agent mode ensures comprehensive analysis
3. **Tool Integration**: Workspace tool gives full code context
4. **Metadata**: Description and model settings for consistency
5. **Professional**: Enterprise-ready prompt management

**These prompt files provide a professional, standardized approach to test generation that your entire team can use consistently!** 🎯
