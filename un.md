# 🔧 **Simplified Pipeline for unit-tests.json**

You're right! Let's keep it simple. Here's a streamlined version that just reads the JSON file and shows responses only when tests fail:

---

## **🚀 Simplified Pipeline Stage**

**Replace your current "Run Unit Tests" stage with this:**

```groovy
stage('Run Unit Tests') {
    steps {
        echo "🧪 Running Unit Tests from unit-tests.json..."
        
        script {
            // Check if unit-tests.json exists
            if (!fileExists('tests/config/unit-tests.json')) {
                error "❌ unit-tests.json file not found at tests/config/unit-tests.json"
            }
            
            // Read the unit tests configuration
            def unitTestsConfig = readJSON file: 'tests/config/unit-tests.json'
            def scenarios = unitTestsConfig.scenarios
            
            echo "📊 Found ${scenarios.size()} test scenarios"
            
            withAWS(region: "${AWS_DEFAULT_REGION}", role: 'arn:aws:iam::623341402407:role/jef-tfe-role') {
                scenarios.each { scenario ->
                    def payload = scenario.event
                    
                    echo "📝 Testing: ${scenario.name}"
                    
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
                    
                    // Check if test passed or failed
                    if (response.statusCode == scenario.expected.statusCode) {
                        echo "✅ ${scenario.name} - PASSED"
                    } else {
                        echo "❌ ${scenario.name} - FAILED"
                        echo "Expected status: ${scenario.expected.statusCode}, Got: ${response.statusCode}"
                        echo "Response: ${response}"
                    }
                }
            }
        }
    }
}
```

---

## **📄 Even Simpler Version (Minimal)**

If you want it even more minimal:

```groovy
stage('Run Unit Tests') {
    steps {
        echo "🧪 Running Unit Tests..."
        
        script {
            def scenarios = readJSON(file: 'tests/config/unit-tests.json').scenarios
            
            withAWS(region: "${AWS_DEFAULT_REGION}", role: 'arn:aws:iam::623341402407:role/jef-tfe-role') {
                scenarios.each { scenario ->
                    echo "📝 ${scenario.name}"
                    
                    def payload = scenario.event
                    
                    // Add timestamp if queryStringParameters exists
                    if (payload.queryStringParameters) {
                        payload.queryStringParameters.timestamp = new Date().format('yyyyMMdd_HHmmss')
                    }
                    
                    def response = invokeLambda(
                        functionName: env.LAMBDA_FUNCTION_NAME,
                        payload: payload,
                        region: env.AWS_DEFAULT_REGION
                    )
                    
                    if (response.statusCode == scenario.expected.statusCode) {
                        echo "✅ PASSED"
                    } else {
                        echo "❌ FAILED - Expected: ${scenario.expected.statusCode}, Got: ${response.statusCode}"
                        echo "Response: ${response}"
                    }
                }
            }
        }
    }
}
```

---

## **📋 Complete Example unit-tests.json**

Here's a clean example that matches your format:

```json
{
  "scenarios": [
    {
      "name": "RetrieveTaskById-Missing-Parameters",
      "event": {
        "queryStringParameters": {
          "operation": "RetrieveTaskById"
        }
      },
      "expected": {
        "statusCode": 400,
        "bodyContains": ["error"]
      }
    },
    {
      "name": "RetrieveDistinctColumns-Valid-Request",
      "event": {
        "queryStringParameters": {
          "operation": "RetrieveDistinctColumns",
          "table_name": "test_tasks",
          "column_name": "status"
        }
      },
      "expected": {
        "statusCode": 200
      }
    },
    {
      "name": "RetrieveJobList-Valid-Request",
      "event": {
        "queryStringParameters": {
          "operation": "RetrieveJobList",
          "table_name": "test_jobs"
        }
      },
      "expected": {
        "statusCode": 200
      }
    },
    {
      "name": "Submit-Valid-Task",
      "event": {
        "queryStringParameters": {
          "operation": "Submit",
          "table_name": "test_tasks"
        },
        "body": "{\"task_id\": \"test_${timestamp}\", \"status\": \"pending\", \"description\": \"Test task\"}"
      },
      "expected": {
        "statusCode": 200
      }
    },
    {
      "name": "Update-Valid-Task",
      "event": {
        "queryStringParameters": {
          "operation": "Update"
        },
        "body": "{\"task_id\": \"existing_task\", \"status\": \"completed\"}"
      },
      "expected": {
        "statusCode": 200
      }
    },
    {
      "name": "Health-Check",
      "event": {
        "queryStringParameters": {
          "operation": "HealthCheck"
        }
      },
      "expected": {
        "statusCode": 200
      }
    }
  ]
}
```

---

## **🎯 What This Does**

**✅ Simple Behavior:**
1. Reads `tests/config/unit-tests.json`
2. Loops through each scenario
3. Adds timestamp to payload
4. Invokes Lambda with the payload
5. Compares response status code
6. **Only shows response details when test fails**
7. Prints ✅ PASSED or ❌ FAILED for each test

**📊 Output Example:**
```
📝 RetrieveTaskById-Missing-Parameters
✅ PASSED

📝 RetrieveDistinctColumns-Valid-Request  
✅ PASSED

📝 Submit-Valid-Task
❌ FAILED - Expected: 200, Got: 500
Response: {statusCode=500, body={"error": "Database connection failed"}}

📝 Health-Check
✅ PASSED
```

**That's it! Clean, simple, and shows failures with response details only when needed.** 🚀
