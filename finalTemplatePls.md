# 🎯 **Modular, Customizable Prompt Templates**

You're absolutely right! Let's make the templates modular so users can easily delete unused sections and add their own files. Here's the updated approach:

---

## **📄 `.github/prompts/generate-unit-tests.prompt.md`**

```markdown
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
5. Generated file will be saved as `tests/config/unit-tests.json`

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

---

## 🌐 API Gateway Lambda Configuration
**DELETE THIS ENTIRE SECTION IF NOT USING API GATEWAY**

**HTTP Methods**: ${input:httpMethods:Enter supported methods (GET, POST, PUT, DELETE)}
**API Endpoints**: ${input:apiEndpoints:List endpoints (e.g., /users, /orders/{id})}
**Authentication**: ${input:authType:API_KEY/JWT/COGNITO/NONE}
**Request Format**: ${input:requestFormat:JSON/FORM_DATA/QUERY_PARAMS}

### API Gateway Test Requirements
Generate tests for:
- HTTP method-specific scenarios (GET queryStringParameters, POST body)
- Authentication and authorization scenarios
- API Gateway event structure validation
- CORS and header handling
- Request/response format validation

---

## 📁 S3 Lambda Configuration  
**DELETE THIS ENTIRE SECTION IF NOT USING S3 TRIGGERS**

**S3 Buckets**: ${input:s3Buckets:List buckets your Lambda processes}
**File Types**: ${input:fileTypes:List file types (jpg, pdf, csv, json)}
**Processing Operations**: ${input:s3Operations:List operations (resize, convert, validate)}
**Output Destinations**: ${input:s3Outputs:Where processed files go}

### S3 Test Requirements
Generate tests for:
- S3 event structure validation (bucket, key, eventName)
- File processing scenarios (different file types, sizes)
- S3 permission and access scenarios
- Multiple file processing workflows
- File validation and error handling

---

## 📊 DynamoDB Lambda Configuration
**DELETE THIS ENTIRE SECTION IF NOT USING DYNAMODB TRIGGERS**

**DynamoDB Tables**: ${input:dynamoTables:List tables with streams}
**Stream Events**: ${input:streamEvents:INSERT/MODIFY/REMOVE events you handle}
**Processing Logic**: ${input:dynamoProcessing:What you do with stream events}
**Output Actions**: ${input:dynamoOutputs:Where processed data goes}

### DynamoDB Test Requirements
Generate tests for:
- DynamoDB stream event processing
- INSERT/MODIFY/REMOVE event handling
- Batch processing scenarios
- Stream record validation
- Cross-table operations

---

## 📬 SQS/SNS Lambda Configuration
**DELETE THIS ENTIRE SECTION IF NOT USING MESSAGE QUEUES**

**Queue Names**: ${input:queueNames:List SQS queues or SNS topics}
**Message Types**: ${input:messageTypes:List message types you process}
**Message Format**: ${input:messageFormat:JSON/XML/STRING format}
**Batch Size**: ${input:batchSize:How many messages processed at once}
**Dead Letter Queues**: ${input:dlqHandling:How you handle failed messages}

### Message Queue Test Requirements
Generate tests for:
- Message queue event processing
- Batch message handling
- Message format validation
- Dead letter queue scenarios
- Message ordering and deduplication

---

## ⏰ EventBridge/CloudWatch Lambda Configuration
**DELETE THIS ENTIRE SECTION IF NOT USING SCHEDULED/EVENT TRIGGERS**

**Event Sources**: ${input:eventSources:List event sources (scheduled, custom events)}
**Event Patterns**: ${input:eventPatterns:Event patterns you match}
**Schedule Expression**: ${input:scheduleExpr:Cron or rate expressions}
**Event Processing**: ${input:eventProcessing:What you do with events}

### Event-Driven Test Requirements
Generate tests for:
- Scheduled event processing
- Custom event pattern matching
- Event source validation
- Timestamp and scheduling logic
- Event transformation and routing

---

## 🔄 Generic/Custom Lambda Configuration
**DELETE THIS SECTION IF YOU USED ONE OF THE ABOVE SPECIFIC TRIGGERS**

**Trigger Type**: ${input:customTrigger:Describe your custom trigger}
**Event Structure**: ${input:eventStructure:Describe expected event format}
**Input Processing**: ${input:inputProcessing:How you process input}
**Output Format**: ${input:outputFormat:What your Lambda returns}

### Custom Trigger Test Requirements
Generate tests for:
- Custom event structure validation
- Input parameter processing
- Business logic scenarios
- Error handling patterns
- Output format validation

---

## Comprehensive Analysis Required

Examine ALL specified files that exist to understand:

### From Lambda Function (`${workspaceFolder}/${input:functionFile}`):
- Main handler function and trigger event structure
- Core business operations and logic flow
- Input validation and parameter handling
- Response/output format and structure
- Error handling and exception management
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
- Custom error handling

## Test Generation Requirements

Generate 20-25 comprehensive test scenarios covering:

### ✅ Success Cases
- Valid events with complete required data
- Boundary value testing for parameters
- Different valid event variations
- Optional parameter scenarios
- Successful external service calls

### ❌ Error Cases
- Missing required event fields
- Invalid parameter values and types
- Malformed event structures
- Business rule violations
- Validation failures

### 🔒 Security/Permission Cases (if applicable)
- Authentication failures
- Authorization/permission denials
- Invalid tokens or credentials
- Resource access restrictions

### 💥 System Error Cases
- External service failures
- Network timeouts and connection issues
- Resource limitations and throttling
- Configuration errors
- Database connection failures

### 🎯 Edge Cases
- Empty or null event data
- Large payload processing
- Concurrent processing scenarios
- Resource constraint testing
- Boundary condition validation

## Output JSON Structure

Create `tests/config/unit-tests.json` with this structure:

```json
{
  "scenarios": [
    {
      "name": "Operation-Scenario-Type",
      "description": "Clear description of test purpose",
      "event": {
        // Event structure appropriate for your trigger type
        // Will be customized based on your Lambda's trigger
      },
      "expected": {
        "statusCode": 200, // For HTTP triggers
        "response": "expected_output", // For other triggers
        "bodyContains": ["success_indicators"],
        "bodyNotContains": ["error_indicators"],
        "logContains": ["processing_messages"],
        "externalCalls": ["s3:GetObject", "dynamodb:PutItem"],
        "responseTime": 5000
      }
    }
  ]
}
```

## Final Requirements
🎯 Generate tests appropriate for your specific trigger type (based on sections you kept)
🎯 Use realistic event structures for your trigger source
🎯 Include trigger-specific error scenarios
🎯 Test actual business logic from all analyzed files
🎯 Cover external service integration patterns
🎯 Use actual configuration values where available
🎯 Include performance and reliability scenarios
🎯 Generate 20-25 comprehensive test scenarios
🎯 Use proper naming conventions for your trigger type
🎯 Save as tests/config/unit-tests.json

Please analyze ALL the specified project files that exist and generate comprehensive unit tests appropriate for your Lambda trigger type and business requirements.
```

---

## **📄 `.github/prompts/generate-integration-tests.prompt.md`**

```markdown
---
mode: agent
model: gpt-4
tools: ['workspace']
description: Generate comprehensive integration-tests.json with modular trigger support
---

# Generate Lambda Integration Tests

## Instructions for User
1. **DELETE** any trigger sections below that don't apply to your Lambda
2. **MODIFY** file paths to match your project structure  
3. **ADD** additional files if needed in the "Additional Project Files" section
4. Run this prompt to generate end-to-end workflow tests
5. Generated file will be saved as `tests/config/integration-tests.json`

---

## Project Architecture Analysis

Analyze the complete project structure in ${workspaceFolder}

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
**Another File**: ${input:anotherFile:path/to/another/file.py}
-->

### Lambda Architecture Details
**Function Name**: ${input:functionName:Enter your Lambda function name}
**Primary Workflows**: ${input:workflows:Describe main business processes}
**Data Sources**: ${input:dataSources:List data sources (S3 buckets, DynamoDB tables, etc.)}
**Output Destinations**: ${input:outputDestinations:Where does processed data go?}
**External Services**: ${input:externalServices:List external integrations}
**Business Scenarios**: ${input:businessScenarios:Describe key use cases}

---

## 🌐 API Gateway Integration Workflows
**DELETE THIS ENTIRE SECTION IF NOT USING API GATEWAY**

**API Endpoints**: ${input:apiEndpoints:List all endpoints}
**User Journeys**: ${input:userJourneys:Describe complete user flows}
**Data Dependencies**: ${input:apiDataDeps:How operations depend on each other}
**Authentication Flow**: ${input:authFlow:Login → Operations → Logout flow}

### API Gateway Workflow Types
Generate workflows for:
- **Complete CRUD Workflows**: Full resource lifecycle testing
- **Authentication Flows**: Login → Operations → Token refresh → Logout  
- **Business Process Flows**: Multi-step API operations with data dependencies
- **Error Recovery Flows**: API failure handling and retry mechanisms
- **Cross-Resource Workflows**: Operations that span multiple resources

---

## 📁 S3 Integration Workflows
**DELETE THIS ENTIRE SECTION IF NOT USING S3 TRIGGERS**

**S3 Buckets**: ${input:s3WorkflowBuckets:Input and output buckets}
**File Processing Pipeline**: ${input:s3Pipeline:Describe complete file processing flow}
**File Dependencies**: ${input:s3Dependencies:How files relate to each other}
**Notification Chain**: ${input:s3Notifications:What happens after processing}

### S3 Workflow Types
Generate workflows for:
- **File Processing Pipelines**: Upload → Process → Transform → Store → Notify
- **Batch Processing Workflows**: Multiple file processing with coordination
- **Error Handling Flows**: Failed uploads, corrupted files, processing failures
- **Data Lifecycle Workflows**: File creation → processing → archival → cleanup
- **Multi-Stage Processing**: File → Stage1 → Stage2 → Final output

---

## 📊 DynamoDB Integration Workflows
**DELETE THIS ENTIRE SECTION IF NOT USING DYNAMODB TRIGGERS**

**Table Relationships**: ${input:dynamoRelations:How tables relate to each other}
**Data Flow**: ${input:dynamoFlow:Stream → Process → Update flow}
**Aggregation Logic**: ${input:dynamoAggregation:How you calculate aggregates}
**Cross-Table Updates**: ${input:dynamoCrossTable:Multi-table operations}

### DynamoDB Workflow Types
Generate workflows for:
- **Data Synchronization Flows**: Stream processing → Transform → Replicate
- **Event-Driven Workflows**: Data change → Trigger → Process → Update
- **Aggregation Workflows**: Stream events → Calculate → Update aggregates
- **Cross-Table Workflows**: Multi-table updates and consistency validation
- **Data Pipeline Workflows**: Ingest → Transform → Aggregate → Export

---

## 📬 Message Queue Integration Workflows
**DELETE THIS ENTIRE SECTION IF NOT USING MESSAGE QUEUES**

**Queue Architecture**: ${input:queueArch:How queues connect to each other}
**Message Flow**: ${input:messageFlow:Message → Process → Forward flow}
**Batch Processing**: ${input:batchFlow:How you handle message batches}
**Error Recovery**: ${input:queueErrorFlow:Failed message handling}

### Message Queue Workflow Types
Generate workflows for:
- **Message Processing Pipelines**: Receive → Process → Transform → Forward
- **Batch Processing Workflows**: Queue → Batch → Process → Acknowledge
- **Error Handling Flows**: Failed messages → Dead letter → Retry → Escalate
- **Fan-out Workflows**: Single message → Multiple processing paths
- **Message Orchestration**: Complex multi-step message processing

---

## ⏰ Scheduled Integration Workflows
**DELETE THIS ENTIRE SECTION IF NOT USING SCHEDULED TRIGGERS**

**Schedule Types**: ${input:scheduleTypes:Cron, rate, or event-driven schedules}
**Batch Operations**: ${input:batchOps:What batch operations you perform}
**Data Processing**: ${input:scheduleDataProc:How you process scheduled data}
**Reporting Flow**: ${input:reportingFlow:Report generation and delivery}

### Scheduled Workflow Types
Generate workflows for:
- **Batch Job Workflows**: Trigger → Extract → Transform → Load → Report
- **Maintenance Workflows**: Cleanup → Optimize → Backup → Monitor
- **Monitoring Workflows**: Check → Alert → Escalate → Resolve
- **Data Pipeline Workflows**: Schedule → Extract → Process → Deliver
- **Periodic Processing**: Regular data updates and synchronization

---

## 🔄 Custom Integration Workflows
**DELETE THIS SECTION IF YOU USED ONE OF THE ABOVE SPECIFIC TRIGGERS**

**Custom Trigger Flow**: ${input:customFlow:Describe your end-to-end process}
**Integration Points**: ${input:customIntegrations:External systems you integrate with}
**Data Processing**: ${input:customProcessing:How you transform data}
**Output Handling**: ${input:customOutput:What you do with results}

### Custom Workflow Types
Generate workflows for:
- **End-to-End Business Processes**: Complete business scenario testing
- **Data Integration Flows**: Input → Transform → Output validation
- **Service Orchestration**: Multi-service coordination and validation
- **Error Recovery Scenarios**: Failure handling and system recovery
- **Performance Workflows**: Load testing and optimization validation

---

## Integration Requirements

Generate 6-8 comprehensive workflow scenarios:

### 🔄 Complete Business Workflows
- End-to-end business processes using your actual trigger
- Multi-step operations with realistic data flow
- Cross-service integration testing
- State management and persistence validation

### 📊 Data Flow Workflows  
- Data ingestion → processing → output workflows
- Cross-system data synchronization
- Data validation and transformation pipelines
- Error handling and data recovery scenarios

### 🔧 Resilience & Recovery Workflows
- Service failure and recovery testing
- Retry and circuit breaker validation
- Data consistency and rollback scenarios
- Health check and monitoring workflows

### ⚡ Performance & Scale Workflows
- Load testing with realistic event volumes
- Concurrent processing scenarios
- Resource utilization and optimization
- Timeout and performance boundary testing

## Output JSON Structure

Create trigger-appropriate integration workflows:

```json
{
  "workflows": [
    {
      "name": "realistic-business-workflow",
      "description": "End-to-end business scenario",
      "timeout": 300,
      "setup": {
        "createTestData": ["data_to_create"],
        "configureServices": ["services_to_configure"]
      },
      "steps": [
        {
          "name": "step-name",
          "description": "What this step accomplishes",
          "type": "invoke_lambda", // or "trigger_event" for non-HTTP
          "retry": 3,
          "payload": {
            // Trigger-appropriate event structure
          },
          "expect": {
            "executionSuccess": true,
            "responseTime": 5000,
            "logContains": ["processing_messages"],
            "externalCalls": ["expected_service_calls"],
            "outputValidation": {
              "checkType": "validation_criteria"
            }
          }
        },
        {
          "name": "wait-for-processing",
          "description": "Allow asynchronous processing",
          "type": "wait",
          "seconds": 5
        },
        {
          "name": "verify-results",
          "description": "Validate complete workflow",
          "type": "validation",
          "checks": [
            {
              "service": "service_name",
              "operation": "check_operation",
              "parameters": {"key": "value"}
            }
          ]
        }
      ],
      "cleanup": {
        "removeTestData": ["data_to_cleanup"],
        "resetServices": ["services_to_reset"]
      }
    }
  ]
}
```

## Dynamic Variables
- `${timestamp}` - Current timestamp
- `${uuid}` - Generated unique ID
- `${auth_token}` - Authentication token (for API Gateway)
- `${test_file_key}` - Generated test file key (for S3)
- `${test_record_id}` - Generated test record ID (for DynamoDB)
- `${test_message_id}` - Generated message ID (for SQS)
- `${config_value}` - Values from configuration files

## Final Requirements
🎯 Generate workflows appropriate for your specific trigger type (based on sections you kept)
🎯 Use realistic event structures and data volumes
🎯 Include complete business scenario validation
🎯 Test actual external service integrations from your files
🎯 Include error handling and recovery scenarios
🎯 Validate end-to-end data flow and transformations
🎯 Test performance under realistic load conditions
🎯 Include proper setup and cleanup procedures
🎯 Generate 6-8 comprehensive integration workflows
🎯 Use dynamic variables for realistic testing
🎯 Save as tests/config/integration-tests.json

Please analyze ALL the specified project files that exist and generate comprehensive integration workflows appropriate for your Lambda trigger type and business requirements.
```

---

## **📄 Updated `.github/prompts/README.md`**

```markdown
# 🤖 Modular Lambda Test Generation Prompts

This directory contains **modular, customizable** GitHub Copilot prompts for generating comprehensive test files for AWS Lambda functions. These prompts support any Lambda trigger type and can be easily customized by deleting unused sections and adding your own files.

## 🎯 Key Features

- ✂️ **Modular Design**: Delete sections for triggers you don't use
- 🔧 **Customizable**: Add your own project files easily
- 🌍 **Universal**: Works with any Lambda trigger type
- 📊 **Comprehensive**: Generates 20+ unit tests and 6-8 integration workflows
- 🚀 **Project-Aware**: Analyzes your actual code structure

## 📁 Available Prompts

### 🔬 Unit Tests
- **File**: `generate-unit-tests.prompt.md`
- **Purpose**: Generate comprehensive unit test scenarios
- **Output**: `tests/config/unit-tests.json`
- **Coverage**: Success, error, security, and edge cases

### 🔄 Integration Tests  
- **File**: `generate-integration-tests.prompt.md`
- **Purpose**: Generate end-to-end workflow tests
- **Output**: `tests/config/integration-tests.json`
- **Coverage**: Complete business workflows with external service validation

## 🚀 Quick Start Guide

### Step 1: Customize the Prompt Template

#### For Each Prompt File:
1. **Open** `.github/prompts/generate-unit-tests.prompt.md`
2. **Delete** trigger sections you don't use:
   ```markdown
   ## 📁 S3 Lambda Configuration  
   **DELETE THIS ENTIRE SECTION IF NOT USING S3 TRIGGERS**
   [... entire section ...]
   ```
3. **Add** your custom files:
   ```markdown
   ### Additional Project Files (ADD your own files here)
   **Your Custom File**: ${input:customFile:path/to/your/file.py}
   **Another File**: ${input:anotherFile:path/to/another/file.py}
   ```
4. **Save** your customized prompt

### Step 2: Run Your Customized Prompt

#### Method 1: Command Palette (Recommended)
1. `Ctrl+Shift+P` → `GitHub Copilot: Run Prompt`
2. Select your customized prompt file
3. Fill in only the relevant fields
4. Generate tests

#### Method 2: Chat Reference
```
@workspace Run .github/prompts/generate-unit-tests.prompt.md
```

## 📋 Customization Examples

### Example 1: API Gateway Lambda

#### What to Keep:
```markdown
## 🌐 API Gateway Lambda Configuration
**HTTP Methods**: ${input:httpMethods:GET, POST, PUT, DELETE}
**Authentication**: ${input:authType:JWT}
```

#### What to Delete:
```markdown
## 📁 S3 Lambda Configuration  
**DELETE THIS ENTIRE SECTION IF NOT USING S3 TRIGGERS**
[... delete entire section ...]

## 📊 DynamoDB Lambda Configuration
**DELETE THIS ENTIRE SECTION IF NOT USING DYNAMODB TRIGGERS** 
[... delete entire section ...]
```

#### What to Add:
```markdown
### Additional Project Files (ADD your own files here)
**Auth Utils**: ${input:authFile:auth/jwt_utils.py}
**API Models**: ${input:modelsFile:models/api_models.py}
**Business Logic**: ${input:businessFile:services/user_service.py}
```

### Example 2: S3 Processing Lambda

#### What to Keep:
```markdown
## 📁 S3 Lambda Configuration  
**S3 Buckets**: ${input:s3Buckets:input-bucket, output-bucket}
**File Types**: ${input:fileTypes:jpg, png, pdf}
```

#### What to Delete:
```markdown
## 🌐 API Gateway Lambda Configuration
**DELETE THIS ENTIRE SECTION IF NOT USING API GATEWAY**
[... delete entire section ...]
```

#### What to Add:
```markdown
### Additional Project Files (ADD your own files here)
**Image Processor**: ${input:imageFile:processors/image_processor.py}
**File Validator**: ${input:validatorFile:validators/file_validator.py}
**S3 Utils**: ${input:s3Utils:utils/s3_helper.py}
```

### Example 3: Multi-Trigger Lambda

#### Keep Multiple Sections:
```markdown
## 🌐 API Gateway Lambda Configuration
[... keep this section ...]

## 📬 SQS/SNS Lambda Configuration
[... keep this section ...]
```

#### Delete Others:
```markdown
## 📁 S3 Lambda Configuration  
**DELETE THIS ENTIRE SECTION IF NOT USING S3 TRIGGERS**
[... delete ...]
```

## 🛠️ Adding Custom Files

### Common File Types to Add:

#### Business Logic Files:
```markdown
**Business Rules**: ${input:businessRules:business/rules.py}
**Workflow Engine**: ${input:workflowFile:workflow/engine.py}
**State Machine**: ${input:stateMachine:state/machine.py}
```

#### Integration Files:
```markdown
**API Client**: ${input:apiClient:clients/external_api.py}
**Database Client**: ${input:dbClient:clients/database.py}
**Message Producer**: ${input:messageProducer:messaging/producer.py}
```

#### Utility Files:
```markdown
**Crypto Utils**: ${input:cryptoUtils:utils/crypto.py}
**Date Utils**: ${input:dateUtils:utils/date_helper.py}
**Logging Config**: ${input:loggingConfig:config/logging.py}
```

#### Domain-Specific Files:
```markdown
**Payment Processor**: ${input:paymentFile:payments/processor.py}
**Email Service**: ${input:emailFile:notifications/email.py}
**Report Generator**: ${input:reportFile:reports/generator.py}
```

## 📝 Usage Examples by Lambda Type

### 🌐 API Gateway Example

#### Customized Prompt Input:
```
Lambda Function: api/lambda_function.py
Auth Utils: auth/jwt_handler.py
API Models: models/request_models.py
Business Service: services/user_service.py

Function Name: user-management-api
HTTP Methods: GET, POST, PUT
Authentication: JWT
Operations: CreateUser, GetUser, UpdateUser, DeleteUser
```

### 📁 S3 Processing Example

#### Customized Prompt Input:
```
Lambda Function: processors/image_lambda.py
Image Processor: processors/image_utils.py
S3 Helper: utils/s3_operations.py
Validation Rules: validators/image_validator.py

Function Name: image-processing-service
S3 Buckets: raw-images, processed-images
File Types: jpg, png, gif
Processing Operations: resize, thumbnail, watermark
```

### 📬 SQS Processing Example

#### Customized Prompt Input:
```
Lambda Function: queue/message_processor.py
Message Parser: parsers/message_parser.py
Business Logic: services/order_service.py
Database Client: clients/dynamo_client.py

Function Name: order-processing-queue
Queue Names: order-queue, notification-queue
Message Types: OrderCreated, OrderUpdated, OrderCancelled
```

## 🎯 Best Practices

### 1. Start with Template Cleanup
- Always delete unused trigger sections first
- This reduces prompt complexity and improves accuracy
- Only keep sections relevant to your Lambda

### 2. Add All Relevant Files
- Include ALL files your Lambda imports or uses
- Don't forget utility files, models, and configuration
- More context = better test generation

### 3. Be Specific About Your Architecture
- Provide detailed business logic descriptions
- Include actual table names, API endpoints, and service names
- Specify exact file types, message formats, etc.

### 4. Use Realistic File Paths
- Match your actual project structure
- Use relative paths from project root
- Double-check file paths exist

### 5. Review Generated Tests
- Always review before using in production
- Customize test data for your specific use cases
- Add additional edge cases specific to your domain

## 🔧 Template Maintenance

### Version Control Your Customizations
```bash
# Create custom templates for your team
cp generate-unit-tests.prompt.md generate-unit-tests-api.prompt.md
cp generate-unit-tests.prompt.md generate-unit-tests-s3.prompt.md

# Customize each for specific use cases
# Commit to share with team
```

### Team-Specific Templates
Create specialized templates for common patterns:
- `generate-unit-tests-api-gateway.prompt.md`
- `generate-unit-tests-s3-processing.prompt.md`
- `generate-unit-tests-data-pipeline.prompt.md`

### Update and Improve
- Regularly update templates based on team feedback
- Add new trigger types as AWS releases them
- Improve prompts based on generated test quality

## 🚨 Troubleshooting

### Common Issues:

#### "Too many irrelevant tests generated"
- **Solution**: Delete unused trigger sections more aggressively
- Make sure you only kept relevant configuration sections

#### "Tests don't match my code structure"
- **Solution**: Add more of your actual project files to analysis
- Verify file paths are correct and files exist

#### "Missing edge cases for my domain"
- **Solution**: Add domain-specific files and be more detailed in business logic description
- Customize generated tests after generation

#### "Prompt is too long/complex"
- **Solution**: Delete ALL unused sections, even if you think they might be relevant
- Focus on only your actual trigger type and architecture

## 📞 Support

For questions about customizing these prompts:
1. **Review** the examples above for your Lambda type
2. **Start simple** - delete everything you don't use
3. **Add incrementally** - add files one by one and test
4. **Validate results** - ensure generated tests match your expectations

---

**These modular prompts adapt to any Lambda architecture. Customize once, use everywhere!** 🚀
```

---

## **📄 `.github/prompts/CUSTOMIZATION-GUIDE.md`**

```markdown
# ✂️ Prompt Customization Guide

## 🎯 Quick Customization Steps

### 1. Identify Your Lambda Type
- 🌐 **API Gateway**: REST/HTTP APIs
- 📁 **S3**: File processing triggers  
- 📊 **DynamoDB**: Stream processing
- 📬 **SQS/SNS**: Message queues
- ⏰ **Scheduled**: Cron/EventBridge
- 🔄 **Custom**: Other triggers

### 2. Delete Unused Sections

#### Find sections like this:
```markdown
## 📁 S3 Lambda Configuration  
**DELETE THIS ENTIRE SECTION IF NOT USING S3 TRIGGERS**
```

#### Delete the ENTIRE section:
```markdown
## 📁 S3 Lambda Configuration  
**DELETE THIS ENTIRE SECTION IF NOT USING S3 TRIGGERS**

**S3 Buckets**: ${input:s3Buckets:List buckets your Lambda processes}
**File Types**: ${input:fileTypes:List file types (jpg, pdf, csv, json)}
...
[Delete everything until the next ## section]
```

### 3. Add Your Files

#### Find this section:
```markdown
### Additional Project Files (ADD your own files here)
**Utils Module**: ${input:utilsFile:utils.py}
<!-- ADD MORE FILES AS NEEDED:
**Your Custom File**: ${input:customFile:path/to/your/file.py}
-->
```

#### Add your files:
```markdown
### Additional Project Files (ADD your own files here)
**Auth Handler**: ${input:authFile:auth/handler.py}
**Business Logic**: ${input:businessFile:services/business.py}
**Data Models**: ${input:modelsFile:models/data_models.py}
**API Client**: ${input:apiClient:clients/external_api.py}
```

## 🔧 File Input Format

### Pattern:
```markdown
**Display Name**: ${input:variableName:default/path/file.py}
```

### Examples:
```markdown
**Database Utils**: ${input:dbUtils:utils/database.py}
**Config Manager**: ${input:configMgr:config/manager.py}
**Email Service**: ${input:emailSvc:services/email.py}
```

## 🎨 Customization Templates

### API Gateway Lambda
```markdown
## 🌐 API Gateway Lambda Configuration
**HTTP Methods**: ${input:httpMethods:GET, POST, PUT, DELETE}
**Authentication**: ${input:authType:JWT}

### Additional Project Files
**Auth Middleware**: ${input:authMiddleware:middleware/auth.py}
**Request Validators**: ${input:validators:validators/request.py}
**Response Builders**: ${input:responseBuilder:builders/response.py}
**Business Services**: ${input:businessServices:services/business.py}
```

### S3 Processing Lambda
```markdown
## 📁 S3 Lambda Configuration  
**S3 Buckets**: ${input:s3Buckets:input-bucket, output-bucket}
**File Types**: ${input:fileTypes:jpg, png, pdf, csv}

### Additional Project Files
**File Processor**: ${input:fileProcessor:processors/file_handler.py}
**Image Utils**: ${input:imageUtils:utils/image_processing.py}
**S3 Operations**: ${input:s3Ops:clients/s3_client.py}
**File Validator**: ${input:fileValidator:validators/file_validator.py}
```

### Message Queue Lambda
```markdown
## 📬 SQS/SNS Lambda Configuration
**Queue Names**: ${input:queueNames:order-queue, notification-queue}
**Message Types**: ${input:messageTypes:OrderEvent, NotificationEvent}

### Additional Project Files
**Message Parser**: ${input:messageParser:parsers/message_parser.py}
**Queue Client**: ${input:queueClient:clients/sqs_client.py}
**Event Handlers**: ${input:eventHandlers:handlers/event_handler.py}
**Message Validators**: ${input:msgValidators:validators/message_validator.py}
```

## 🚀 Save and Use

### 1. Save Your Customized Template
```bash
# Save with descriptive name
.github/prompts/generate-unit-tests-api.prompt.md
.github/prompts/generate-unit-tests-s3.prompt.md
```

### 2. Use in VS Code
```
Ctrl+Shift+P → GitHub Copilot: Run Prompt
Select: generate-unit-tests-api.prompt.md
```

### 3. Fill Only Relevant Fields
Since you deleted unused sections, you'll only see inputs for:
- Your trigger type
- Your actual files
- Your specific configuration

**Quick, clean, and focused on YOUR Lambda!** ✨
```

This modular approach gives users maximum flexibility while maintaining the comprehensive analysis capabilities. Users can easily customize templates for their specific needs and reuse them across similar Lambda functions.
