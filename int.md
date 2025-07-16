# 🔧 **Simplified Integration Tests Pipeline Stage**

Based on your integration tests JSON structure, here's a simple pipeline stage that reads the JSON file and shows responses only when steps fail:

---

## **🚀 Simplified Integration Tests Stage**

```groovy
stage('Run Integration Tests') {
    steps {
        echo "🔗 Running Integration Tests from integration-tests.json..."
        
        script {
            // Check if integration-tests.json exists
            if (!fileExists('tests/config/integration-tests.json')) {
                error "❌ integration-tests.json file not found at tests/config/integration-tests.json"
            }
            
            // Read the integration tests configuration
            def integrationTestsConfig = readJSON file: 'tests/config/integration-tests.json'
            def workflows = integrationTestsConfig.workflows
            
            echo "📊 Found ${workflows.size()} integration workflows"
            
            withAWS(region: "${AWS_DEFAULT_REGION}", role: 'arn:aws:iam::623341402407:role/jef-tfe-role') {
                workflows.each { workflow ->
                    echo "🔄 Workflow: ${workflow.name}"
                    
                    workflow.steps.each { step ->
                        echo "📝 Step: ${step.name}"
                        
                        if (step.type == "invoke_lambda") {
                            def payload = step.payload
                            
                            // Add dynamic values to payload
                            if (payload.queryStringParameters) {
                                payload.queryStringParameters.timestamp = new Date().format('yyyyMMdd_HHmmss')
                                payload.queryStringParameters.build_id = env.BUILD_ID
                                payload.queryStringParameters.test_session = env.TEST_SESSION_ID
                                payload.queryStringParameters.uuid = UUID.randomUUID().toString()
                            }
                            
                            // Replace placeholders in body if present
                            if (payload.body) {
                                payload.body = payload.body
                                    .replace('${timestamp}', new Date().format('yyyyMMdd_HHmmss'))
                                    .replace('${build_id}', env.BUILD_ID)
                                    .replace('${uuid}', UUID.randomUUID().toString())
                            }
                            
                            def response = invokeLambda(
                                functionName: env.LAMBDA_FUNCTION_NAME,
                                payload: payload,
                                region: env.AWS_DEFAULT_REGION
                            )
                            
                            // Check if step passed or failed
                            if (response.statusCode == step.expect.statusCode) {
                                echo "✅ ${step.name} - PASSED"
                            } else {
                                echo "❌ ${step.name} - FAILED"
                                echo "Expected status: ${step.expect.statusCode}, Got: ${response.statusCode}"
                                echo "Response: ${response}"
                            }
                            
                        } else if (step.type == "wait") {
                            echo "⏳ Waiting ${step.seconds} seconds..."
                            sleep(step.seconds)
                            echo "✅ Wait completed"
                        }
                    }
                    
                    echo "🎉 Workflow '${workflow.name}' completed"
                }
            }
        }
    }
}
```

---

## **📄 Even Simpler Version (Minimal)**

If you want it more minimal:

```groovy
stage('Run Integration Tests') {
    steps {
        echo "🔗 Running Integration Tests..."
        
        script {
            def workflows = readJSON(file: 'tests/config/integration-tests.json').workflows
            
            withAWS(region: "${AWS_DEFAULT_REGION}", role: 'arn:aws:iam::623341402407:role/jef-tfe-role') {
                workflows.each { workflow ->
                    echo "🔄 ${workflow.name}"
                    
                    workflow.steps.each { step ->
                        echo "📝 ${step.name}"
                        
                        if (step.type == "invoke_lambda") {
                            def payload = step.payload
                            
                            // Add timestamp if queryStringParameters exists
                            if (payload.queryStringParameters) {
                                payload.queryStringParameters.timestamp = new Date().format('yyyyMMdd_HHmmss')
                            }
                            
                            // Replace timestamp in body
                            if (payload.body) {
                                payload.body = payload.body.replace('${timestamp}', new Date().format('yyyyMMdd_HHmmss'))
                            }
                            
                            def response = invokeLambda(
                                functionName: env.LAMBDA_FUNCTION_NAME,
                                payload: payload,
                                region: env.AWS_DEFAULT_REGION
                            )
                            
                            if (response.statusCode == step.expect.statusCode) {
                                echo "✅ PASSED"
                            } else {
                                echo "❌ FAILED - Expected: ${step.expect.statusCode}, Got: ${response.statusCode}"
                                echo "Response: ${response}"
                            }
                            
                        } else if (step.type == "wait") {
                            sleep(step.seconds ?: 2)
                            echo "✅ Wait completed"
                        }
                    }
                }
            }
        }
    }
}
```

---

## **📋 Complete Example integration-tests.json**

Here's a clean example that matches your format:

```json
{
  "workflows": [
    {
      "name": "Complete-Task-Management-Workflow",
      "description": "Test full task lifecycle: create, retrieve, update, audit",
      "steps": [
        {
          "name": "Submit-New-Task",
          "type": "invoke_lambda",
          "payload": {
            "queryStringParameters": {
              "operation": "Submit",
              "table_name": "test_tasks"
            },
            "body": "{\"task_id\": \"integration_test_${timestamp}\", \"status\": \"pending\", \"description\": \"Integration test task\", \"created_by\": \"jenkins\"}"
          },
          "expect": {
            "statusCode": 200,
            "bodyContains": ["success"]
          }
        },
        {
          "name": "Wait-For-DB-Commit",
          "type": "wait",
          "seconds": 2
        },
        {
          "name": "Retrieve-Task-by-ID",
          "type": "invoke_lambda",
          "payload": {
            "queryStringParameters": {
              "operation": "RetrieveTaskById",
              "table_name": "test_tasks",
              "pkey_id": "integration_test_${timestamp}"
            }
          },
          "expect": {
            "statusCode": 200,
            "bodyContains": ["task_data"]
          }
        },
        {
          "name": "Update-Task-Status",
          "type": "invoke_lambda",
          "payload": {
            "queryStringParameters": {
              "operation": "Update"
            },
            "body": "{\"task_id\": \"integration_test_${timestamp}\", \"status\": \"in_progress\"}"
          },
          "expect": {
            "statusCode": 200,
            "bodyContains": ["success"]
          }
        },
        {
          "name": "Wait-For-Update",
          "type": "wait",
          "seconds": 1
        },
        {
          "name": "Verify-Task-Updated",
          "type": "invoke_lambda",
          "payload": {
            "queryStringParameters": {
              "operation": "RetrieveTaskById",
              "table_name": "test_tasks",
              "pkey_id": "integration_test_${timestamp}"
            }
          },
          "expect": {
            "statusCode": 200,
            "bodyContains": ["in_progress"]
          }
        }
      ]
    },
    {
      "name": "Job-Management-Workflow",
      "description": "Test job operations workflow",
      "steps": [
        {
          "name": "Retrieve-Job-List",
          "type": "invoke_lambda",
          "payload": {
            "queryStringParameters": {
              "operation": "RetrieveJobList",
              "table_name": "test_jobs"
            }
          },
          "expect": {
            "statusCode": 200,
            "bodyContains": ["jobs"]
          }
        },
        {
          "name": "Get-Distinct-Job-Status",
          "type": "invoke_lambda",
          "payload": {
            "queryStringParameters": {
              "operation": "RetrieveDistinctColumns",
              "table_name": "test_jobs",
              "column_name": "status"
            }
          },
          "expect": {
            "statusCode": 200,
            "bodyContains": ["distinct"]
          }
        }
      ]
    },
    {
      "name": "Error-Recovery-Workflow",
      "description": "Test error handling and recovery",
      "steps": [
        {
          "name": "Try-Invalid-Operation",
          "type": "invoke_lambda",
          "payload": {
            "queryStringParameters": {
              "operation": "InvalidOperation"
            }
          },
          "expect": {
            "statusCode": 400
          }
        },
        {
          "name": "Recovery-Health-Check",
          "type": "invoke_lambda",
          "payload": {
            "queryStringParameters": {
              "operation": "HealthCheck"
            }
          },
          "expect": {
            "statusCode": 200,
            "bodyContains": ["healthy"]
          }
        }
      ]
    }
  ]
}
```

---

## **🎯 What This Does**

**✅ Simple Behavior:**
1. Reads `tests/config/integration-tests.json`
2. Loops through each workflow
3. Executes each step in sequence
4. For `invoke_lambda` steps:
   - Adds timestamps to payload
   - Invokes Lambda
   - Compares response status code
   - **Only shows response details when step fails**
5. For `wait` steps:
   - Sleeps for specified seconds
6. Prints ✅ PASSED or ❌ FAILED for each step

**📊 Output Example:**
```
🔄 Complete-Task-Management-Workflow
📝 Submit-New-Task
✅ PASSED

📝 Wait-For-DB-Commit
✅ Wait completed

📝 Retrieve-Task-by-ID
✅ PASSED

📝 Update-Task-Status
❌ FAILED - Expected: 200, Got: 500
Response: {statusCode=500, body={"error": "Task not found"}}

📝 Wait-For-Update
✅ Wait completed

📝 Verify-Task-Updated
❌ FAILED - Expected: 200, Got: 404
Response: {statusCode=404, body={"error": "Task does not exist"}}

🎉 Workflow 'Complete-Task-Management-Workflow' completed
```

**Clean, simple integration testing that shows failures with response details only when needed!** 🚀
