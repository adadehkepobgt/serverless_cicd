---
mode: agent
model: gpt-4
tools: ['workspace']
description: Generate comprehensive unit-tests.json for AWS Lambda functions with modular trigger support
---

# Generate Lambda Unit Tests

## Instructions for User
1. **DELETE** any trigger sections below that don't apply to your Lambda
2. **MODIFY** file paths to match your project structure
3. **ADD** additional files if needed in the "Additional Project Files" section
4. Run this prompt to generate comprehensive unit tests
5. **MANUALLY CREATE** `tests/config/unit-tests.json` and paste the generated content

---

## 🚨 IMPORTANT: File Creation Instructions
**GitHub Copilot cannot create files directly. After generation:**

1. **Create the directory**: `mkdir -p tests/config`
2. **Create the file**: `touch tests/config/unit-tests.json`
3. **Copy-paste the generated JSON** into the file

---

## Project File Analysis

Analyze the Lambda function and related files in ${workspaceFolder}

### Core Project Files
**Lambda Function**: ${input:functionFile:lambda_handler/lambda_function.py}
**Requirements**: ${input:requirementsFile:lambda_handler/requirements.txt}

### Common Configuration Files (DELETE if not used)
**Config File (Python)**: ${input:configPyFile:config.py}
**Config File (YAML)**: ${input:configYamlFile:config.yaml}
**Database Module**: ${input:databaseFile:database.py}
**Environment File**: ${input:envFile:.env}
**Constants File**: ${input:constantsFile:constants.py}

### Additional Project Files (ADD your own files here)
**Utils Module**: ${input:utilsFile:utils.py}
**Models File**: ${input:modelsFile:models.py}
**Validators File**: ${input:validatorsFile:validators.py}
**Services File**: ${input:servicesFile:services.py}
**Helpers File**: ${input:helpersFile:helpers.py}
<!-- ADD MORE FILES AS NEEDED:
**Your Custom File**: ${input:customFile:path/to/your/file.py}
-->

### Lambda Function Details
**Function Name**: ${input:functionName:Enter your Lambda function name}
**Primary Operations**: ${input:operations:List main operations (e.g., ProcessFile, SendNotification)}
**Main Parameters**: ${input:parameters:List key parameters or event fields}
**External Services**: ${input:externalServices:List services used (S3, DynamoDB, SES, etc.)}
**Business Logic**: ${input:businessLogic:Describe what your Lambda does}

### Payload Size Configuration
**Maximum Allowed Payload Size**: ${input:maxPayloadSize:Enter max payload size (e.g., 6MB, 256KB, 10MB)}
**Payload Size Units**: ${input:payloadUnits:MB/KB/GB}
**Large Payload Handling**: ${input:largePayloadHandling:Describe how large payloads are handled (streaming, chunking, rejection)}

### Test Scope Configuration
**Include Error Cases**: ${input:includeErrorCases:YES/NO - Include test cases for validation errors, authentication failures, and business rule violations}
**Include Non-200 Status Codes**: ${input:includeNon200:YES/NO - Include test cases that return 400, 401, 403, 404, 422, 500, etc.}
**Error Scenarios to Test**: ${input:errorScenarios:If YES above, list specific error scenarios (e.g., invalid input, missing auth, resource not found, validation failures)}
**Expected Error Status Codes**: ${input:errorStatusCodes:If YES above, list status codes to test (e.g., 400, 401, 403, 404, 422, 500)}

---

## 🌐 API Gateway Lambda Configuration
**DELETE THIS ENTIRE SECTION IF NOT USING API GATEWAY**

**HTTP Methods**: ${input:httpMethods:Enter supported methods (GET, POST, PUT, DELETE)}
**API Endpoints**: ${input:apiEndpoints:List endpoints (e.g., /users, /orders/{id})}
**Authentication**: ${input:authType:API_KEY/JWT/COGNITO/NONE}
**Request Format**: ${input:requestFormat:JSON/FORM_DATA/QUERY_PARAMS}

### API Gateway Payload Limits
**API Gateway Payload Limit**: ${input:apiGatewayLimit:Default 10MB for REST API, 256KB for WebSocket}
**Request Body Handling**: ${input:requestBodyHandling:How you handle large request bodies}

### API Gateway Test Requirements
Generate tests for:
- HTTP method-specific scenarios (GET queryStringParameters, POST body)
- Authentication and authorization flows
- API Gateway event structure validation
- CORS and header handling
- Request/response format validation
- Payload size validation (within ${input:maxPayloadSize} ${input:payloadUnits} limit)

**IF includeErrorCases = YES:**
- Invalid authentication scenarios (401)
- Missing authorization scenarios (403)
- Invalid request format scenarios (400)
- Resource not found scenarios (404)
- Validation failure scenarios (422)
- Payload too large scenarios (413)

---

## 📁 S3 Lambda Configuration  
**DELETE THIS ENTIRE SECTION IF NOT USING S3 TRIGGERS**

**S3 Buckets**: ${input:s3Buckets:List buckets your Lambda processes}
**File Types**: ${input:fileTypes:List file types (jpg, pdf, csv, json)}
**Processing Operations**: ${input:s3Operations:List operations (resize, convert, validate)}
**Output Destinations**: ${input:s3Outputs:Where processed files go}

### S3 File Size Limits
**Maximum File Size**: ${input:s3MaxFileSize:Max file size your Lambda can process}
**File Size Handling**: ${input:s3FileSizeHandling:How you handle large files (streaming, chunking, parallel processing)}

### S3 Test Requirements
Generate tests for:
- S3 event structure validation (bucket, key, eventName)
- File processing scenarios (different file types, sizes up to ${input:s3MaxFileSize})
- S3 permission and access scenarios
- Multiple file processing workflows
- File validation and processing
- Large file handling within size limits

**IF includeErrorCases = YES:**
- Missing S3 object scenarios
- Invalid file format scenarios
- File size exceeded scenarios
- S3 permission denied scenarios
- Malformed S3 event scenarios

---

## 📊 DynamoDB Lambda Configuration
**DELETE THIS ENTIRE SECTION IF NOT USING DYNAMODB TRIGGERS**

**DynamoDB Tables**: ${input:dynamoTables:List tables with streams}
**Stream Events**: ${input:streamEvents:INSERT/MODIFY/REMOVE events you handle}
**Processing Logic**: ${input:dynamoProcessing:What you do with stream events}
**Output Actions**: ${input:dynamoOutputs:Where processed data goes}

### DynamoDB Record Size Limits
**DynamoDB Item Size Limit**: ${input:dynamoItemSize:Default 400KB per item}
**Batch Processing Size**: ${input:dynamoBatchSize:Number of records processed in batch}

### DynamoDB Test Requirements
Generate tests for:
- DynamoDB stream event processing
- INSERT/MODIFY/REMOVE event handling
- Batch processing scenarios (respecting ${input:dynamoBatchSize} records)
- Stream record validation (within ${input:dynamoItemSize} limit)
- Cross-table operations

**IF includeErrorCases = YES:**
- Invalid stream record format scenarios
- DynamoDB table access denied scenarios
- Item size exceeded scenarios
- Batch processing failure scenarios
- Stream event parsing error scenarios

---

## 📬 SQS/SNS Lambda Configuration
**DELETE THIS ENTIRE SECTION IF NOT USING MESSAGE QUEUES**

**Queue Names**: ${input:queueNames:List SQS queues or SNS topics}
**Message Types**: ${input:messageTypes:List message types you process}
**Message Format**: ${input:messageFormat:JSON/XML/STRING format}
**Batch Size**: ${input:batchSize:How many messages processed at once}
**Dead Letter Queues**: ${input:dlqHandling:How you handle failed messages}

### Message Size Limits
**SQS Message Size Limit**: ${input:sqsMessageSize:Default 256KB for standard, 2GB for extended}
**SNS Message Size Limit**: ${input:snsMessageSize:Default 256KB}
**Batch Message Total Size**: ${input:batchTotalSize:Total size for batch processing}

### Message Queue Test Requirements
Generate tests for:
- Message queue event processing
- Batch message handling (within ${input:batchTotalSize} total size)
- Message format validation
- Message ordering and deduplication
- Queue processing workflows
- Individual message size validation (within ${input:sqsMessageSize} or ${input:snsMessageSize})

**IF includeErrorCases = YES:**
- Invalid message format scenarios
- Message size exceeded scenarios
- Queue access denied scenarios
- Message parsing error scenarios
- Dead letter queue handling scenarios

---

## ⏰ EventBridge/CloudWatch Lambda Configuration
**DELETE THIS ENTIRE SECTION IF NOT USING SCHEDULED/EVENT TRIGGERS**

**Event Sources**: ${input:eventSources:List event sources (scheduled, custom events)}
**Event Patterns**: ${input:eventPatterns:Event patterns you match}
**Schedule Expression**: ${input:scheduleExpr:Cron or rate expressions}
**Event Processing**: ${input:eventProcessing:What you do with events}

### Event Size Limits
**EventBridge Event Size Limit**: ${input:eventBridgeSize:Default 256KB per event}
**Custom Event Size**: ${input:customEventSize:Size of your custom events}

### Event-Driven Test Requirements
Generate tests for:
- Scheduled event processing
- Custom event pattern matching
- Event source validation
- Timestamp and scheduling logic
- Event transformation and routing
- Event size validation (within ${input:eventBridgeSize} limit)

**IF includeErrorCases = YES:**
- Invalid event pattern scenarios
- Event source unavailable scenarios
- Event size exceeded scenarios
- Event parsing error scenarios
- Schedule execution failure scenarios

---

## 🔄 Generic/Custom Lambda Configuration
**DELETE THIS SECTION IF YOU USED ONE OF THE ABOVE SPECIFIC TRIGGERS**

**Trigger Type**: ${input:customTrigger:Describe your custom trigger}
**Event Structure**: ${input:eventStructure:Describe expected event format}
**Input Processing**: ${input:inputProcessing:How you process input}
**Output Format**: ${input:outputFormat:What your Lambda returns}

### Custom Payload Limits
**Custom Event Size Limit**: ${input:customEventLimit:Size limit for your custom events}
**Response Size Limit**: ${input:responseLimit:Size limit for Lambda response}

### Custom Trigger Test Requirements
Generate tests for:
- Custom event structure validation
- Input parameter processing
- Business logic scenarios
- Output format validation
- Workflow execution
- Payload size validation (within ${input:customEventLimit})

**IF includeErrorCases = YES:**
- Invalid custom event structure scenarios
- Event size exceeded scenarios
- Processing failure scenarios
- Invalid input parameter scenarios
- Business rule violation scenarios

---

## Comprehensive Analysis Required

Examine ALL specified files that exist to understand:

### From Lambda Function (`${workspaceFolder}/${input:functionFile}`):
- Main handler function and trigger event structure
- Core business operations and logic flow
- Input validation and parameter handling
- Response/output format and structure
- Error handling and exception management
- Processing patterns and workflows
- External service integrations

### From Configuration Files (analyze any that exist):
- Configuration constants and environment settings
- Service URLs and connection parameters
- Timeout, retry, and performance settings
- Feature flags and business rule configurations
- Logging and monitoring configurations

### From Additional Project Files (analyze any that exist):
- Utility functions and helper methods
- Data models and validation logic
- Service integration patterns
- Business rule implementations
- Error handling patterns
- Processing workflows

## Test Generation Requirements

Generate 15-25 comprehensive test scenarios covering:

### 🎯 Core Functionality (Always Include)
- Valid events with complete required data
- Boundary value testing for parameters
- Different valid event variations
- Optional parameter scenarios
- External service integrations

### 📊 Data Processing (Always Include)
- Different input data formats
- Various payload sizes (small, medium, near-maximum ${input:maxPayloadSize} ${input:payloadUnits})
- Multiple processing workflows
- Data validation scenarios
- Business rule execution

### 🔧 Integration Scenarios (Always Include)
- External service calls
- Database operations
- File processing operations
- Message processing
- API integrations

### ⚡ Performance & Reliability (Always Include)
- Standard processing loads
- Concurrent processing scenarios
- Resource utilization testing
- Timeout handling
- Retry mechanisms

### 🎪 Edge Cases & Payload Testing (Always Include)
- Boundary conditions
- Optional field variations
- Different configuration settings
- Multiple input sources
- Complex workflow scenarios
- **Payload size boundary testing (test with payloads approaching but not exceeding ${input:maxPayloadSize} ${input:payloadUnits})**
- **Small payload scenarios (minimal valid data)**
- **Medium payload scenarios (typical data volumes)**
- **Large payload scenarios (near maximum allowed size)**

### ❌ Error Cases (Include ONLY if includeErrorCases = YES)
- Missing required event fields
- Invalid parameter values and types
- Malformed event structures
- Business rule violations
- Validation failures
- Authentication failures (if applicable)
- Authorization/permission denials (if applicable)
- Invalid tokens or credentials (if applicable)
- Resource access restrictions
- External service failures
- Network timeouts and connection issues
- Resource limitations and throttling
- Configuration errors
- Database connection failures
- Payload size exceeded scenarios

## Output JSON Structure

Create `tests/config/unit-tests.json` with this structure:

```json
{
  "scenarios": [
    {
      "name": "Operation-Scenario-Description",
      "description": "Clear description of test purpose",
      "testType": "success|error", // Indicate if this is a success or error test case
      "payloadSize": "estimated_size_description", // e.g., "small", "medium", "large", "near-max", "exceeded"
      "event": {
        // Event structure appropriate for your trigger type
        // Will be customized based on your Lambda's trigger
        // Include payload size considerations
      },
      "expected": {
        "statusCode": 200, // Or error codes like 400, 401, 403, 404, 422, 500 if includeErrorCases = YES
        "response": "expected_output",
        "bodyContains": ["expected_data_elements"], // For success cases
        "bodyNotContains": ["error", "exception"], // For success cases, opposite for error cases
        "logContains": ["processing_messages"],
        "externalCalls": ["s3:GetObject", "dynamodb:PutItem"],
        "responseTime": 5000,
        "maxPayloadSize": "${input:maxPayloadSize} ${input:payloadUnits}"
      }
    }
  ]
}
```

## Final Requirements
- Generate tests appropriate for your specific trigger type (based on sections you kept)
- Use realistic event structures for your trigger source
- **Include payload size considerations in test scenarios (within ${input:maxPayloadSize} ${input:payloadUnits} limit)**
- Test actual business logic from all analyzed files
- Cover external service integration patterns
- Use actual configuration values where available
- Include performance and reliability scenarios
- **Generate test scenarios with various payload sizes: small, medium, and large (but within limits)**
- **IF includeErrorCases = YES: Generate error test cases with appropriate status codes from ${input:errorStatusCodes}**
- **IF includeErrorCases = NO: Generate only success test cases (status code 200)**
- **IF includeNon200 = YES: Include test cases that legitimately return non-200 status codes**
- **IF includeNon200 = NO: Focus only on 200 status code scenarios**
- Generate 15-25 comprehensive test scenarios (more if including error cases)
- Use proper naming conventions for test identification
- **Ensure test expectations match the test type (success vs error scenarios)**
- Focus on functional validation and operational testing
- **Validate that all test payloads respect the specified maximum size limit**
- **Include specific error scenarios from ${input:errorScenarios} if provided**

**🎯 COPY THE FOLLOWING JSON TO `tests/config/unit-tests.json`:**

Please analyze ALL the specified project files that exist and generate comprehensive unit tests appropriate for your Lambda trigger type and business requirements. 

**Test Generation Strategy:**
- **IF includeErrorCases = NO**: Focus on functional testing that validates proper Lambda operation and business logic execution with 200 status codes only
- **IF includeErrorCases = YES**: Include both success scenarios (200) AND error scenarios (${input:errorStatusCodes}) as specified in ${input:errorScenarios}
- **Always respect payload size constraints (maximum ${input:maxPayloadSize} ${input:payloadUnits})**
- **Include appropriate mix of success and error cases based on user preferences**
