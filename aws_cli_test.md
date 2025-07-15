# 🔧 **AWS CLI Version of RDS Lambda Testing Framework**

Here's the complete AWS CLI implementation as an alternative to the boto3 approach:

---

## **📋 Configuration Files (Same as boto3 version)**

### **`tests/config/unit-tests.json`**
```json
{
  "scenarios": [
    {
      "name": "RetrieveTaskById-Valid-Request",
      "event": {
        "operation": "RetrieveTaskById",
        "queryStringParameters": {
          "table_name": "test_tasks",
          "pkey_id": "task_123"
        }
      },
      "expected": {
        "statusCode": 200,
        "bodyContains": ["task_data"]
      }
    },
    {
      "name": "RetrieveTaskById-Missing-Parameters",
      "event": {
        "operation": "RetrieveTaskById",
        "queryStringParameters": {}
      },
      "expected": {
        "statusCode": 400,
        "bodyContains": ["error"]
      }
    },
    {
      "name": "RetrieveDistinctColumns-Valid-Request",
      "event": {
        "operation": "RetrieveDistinctColumns",
        "queryStringParameters": {
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
        "operation": "RetrieveJobList",
        "queryStringParameters": {
          "table_name": "test_jobs"
        }
      },
      "expected": {
        "statusCode": 200
      }
    },
    {
      "name": "Submit-Valid-Data",
      "event": {
        "operation": "Submit",
        "queryStringParameters": {
          "table_name": "test_tasks"
        },
        "body": "{\"task_id\": \"test_task_001\", \"status\": \"pending\", \"description\": \"Test task for CI/CD\"}"
      },
      "expected": {
        "statusCode": 200,
        "bodyContains": ["success", "Data submitted successfully"]
      }
    },
    {
      "name": "Update-Valid-Data",
      "event": {
        "operation": "Update",
        "body": "{\"task_id\": \"test_task_001\", \"status\": \"completed\"}"
      },
      "expected": {
        "statusCode": 200,
        "bodyContains": ["success", "Queue records updated successfully"]
      }
    },
    {
      "name": "Invalid-Operation",
      "event": {
        "operation": "InvalidOperation"
      },
      "expected": {
        "statusCode": 200,
        "bodyContains": ["error", "Invalid operation"]
      }
    },
    {
      "name": "Health-Check",
      "event": {
        "test": "health",
        "source": "jenkins"
      },
      "expected": {
        "statusCode": 200
      }
    }
  ]
}
```

### **`tests/config/integration-tests.json`**
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
            "operation": "Submit",
            "queryStringParameters": {
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
          "name": "Wait-for-Database-Commit",
          "type": "wait",
          "seconds": 2
        },
        {
          "name": "Retrieve-Task-by-ID",
          "type": "invoke_lambda",
          "payload": {
            "operation": "RetrieveTaskById",
            "queryStringParameters": {
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
            "operation": "Update",
            "body": "{\"task_id\": \"integration_test_${timestamp}\", \"status\": \"in_progress\"}"
          },
          "expect": {
            "statusCode": 200,
            "bodyContains": ["success"]
          }
        },
        {
          "name": "Verify-Updated-Task",
          "type": "invoke_lambda",
          "payload": {
            "operation": "RetrieveTaskById",
            "queryStringParameters": {
              "table_name": "test_tasks",
              "pkey_id": "integration_test_${timestamp}"
            }
          },
          "expect": {
            "statusCode": 200,
            "bodyContains": ["task_data", "in_progress"]
          }
        }
      ]
    },
    {
      "name": "Database-Query-Operations-Workflow",
      "description": "Test various database query operations",
      "steps": [
        {
          "name": "Get-Job-List",
          "type": "invoke_lambda",
          "payload": {
            "operation": "RetrieveJobList",
            "queryStringParameters": {
              "table_name": "test_jobs"
            }
          },
          "expect": {
            "statusCode": 200
          }
        },
        {
          "name": "Get-Distinct-Column-Values",
          "type": "invoke_lambda",
          "payload": {
            "operation": "RetrieveDistinctColumns",
            "queryStringParameters": {
              "table_name": "test_tasks",
              "column_name": "status"
            }
          },
          "expect": {
            "statusCode": 200
          }
        }
      ]
    }
  ]
}
```

---

## **🔧 Main Test Runner Script**

### **`tests/run_tests.sh` - AWS CLI Implementation**
```bash
#!/bin/bash

# RDS Lambda Testing Framework - AWS CLI Implementation
# Comprehensive testing for RDS-connected Lambda functions

set -e  # Exit on any error

# Configuration
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
CONFIG_DIR="${SCRIPT_DIR}/config"
REPORTS_DIR="${SCRIPT_DIR}/reports"
UTILS_DIR="${SCRIPT_DIR}/utils"

# AWS Configuration
AWS_REGION="${AWS_DEFAULT_REGION:-ap-northeast-1}"
LAMBDA_FUNCTION_PATTERN="${LAMBDA_FUNCTION_PATTERN:-euc-lambda-poc}"

# Test Session Configuration
BUILD_ID="${BUILD_ID:-$(date +%Y%m%d-%H%M%S)}"
TEST_SESSION_ID="test_${BUILD_ID}_$(openssl rand -hex 4)"

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
PURPLE='\033[0;35m'
NC='\033[0m' # No Color

# Ensure required tools are available
check_dependencies() {
    echo "🔧 Checking dependencies..."
    
    local missing_deps=()
    
    # Check for required commands
    for cmd in aws jq openssl; do
        if ! command -v "$cmd" &> /dev/null; then
            missing_deps+=("$cmd")
        fi
    done
    
    if [ ${#missing_deps[@]} -ne 0 ]; then
        echo -e "${RED}❌ Missing required dependencies: ${missing_deps[*]}${NC}"
        echo "Please install the missing dependencies and try again."
        exit 1
    fi
    
    echo -e "${GREEN}✅ All dependencies available${NC}"
}

# Create necessary directories
setup_directories() {
    mkdir -p "${REPORTS_DIR}"/{unit,integration}
    mkdir -p "${UTILS_DIR}"
    mkdir -p "${CONFIG_DIR}"
}

# Find Lambda function
find_lambda_function() {
    echo "🔍 Looking for Lambda function matching: ${LAMBDA_FUNCTION_PATTERN}"
    
    # List all Lambda functions and filter by pattern
    local functions
    functions=$(aws lambda list-functions \
        --region "${AWS_REGION}" \
        --query "Functions[?contains(FunctionName, '${LAMBDA_FUNCTION_PATTERN}')].FunctionName" \
        --output text)
    
    if [ -z "$functions" ]; then
        echo -e "${RED}❌ No Lambda function found matching '${LAMBDA_FUNCTION_PATTERN}'${NC}"
        exit 1
    fi
    
    # Take the first matching function
    LAMBDA_FUNCTION_NAME=$(echo "$functions" | head -n1)
    echo -e "${GREEN}✅ Found Lambda function: ${LAMBDA_FUNCTION_NAME}${NC}"
    
    # Save function info for reports
    aws lambda get-function \
        --function-name "${LAMBDA_FUNCTION_NAME}" \
        --region "${AWS_REGION}" \
        --query 'Configuration.{Name:FunctionName,Runtime:Runtime,Timeout:Timeout,Memory:MemorySize,LastModified:LastModified}' \
        > "${REPORTS_DIR}/lambda-function-info.json"
    
    export LAMBDA_FUNCTION_NAME
}

# Verify Lambda function access and get details
verify_lambda_access() {
    echo "🔍 Verifying Lambda function access..."
    
    local function_info
    function_info=$(aws lambda get-function \
        --function-name "${LAMBDA_FUNCTION_NAME}" \
        --region "${AWS_REGION}" \
        --query 'Configuration.{Runtime:Runtime,Timeout:Timeout,Memory:MemorySize}' \
        --output json 2>/dev/null)
    
    if [ $? -ne 0 ]; then
        echo -e "${RED}❌ Cannot access Lambda function '${LAMBDA_FUNCTION_NAME}'${NC}"
        exit 1
    fi
    
    local runtime=$(echo "$function_info" | jq -r '.Runtime')
    local timeout=$(echo "$function_info" | jq -r '.Timeout')
    local memory=$(echo "$function_info" | jq -r '.Memory')
    
    echo -e "${GREEN}✅ Lambda function '${LAMBDA_FUNCTION_NAME}' is accessible${NC}"
    echo "   Runtime: $runtime"
    echo "   Timeout: ${timeout}s"
    echo "   Memory: ${memory}MB"
    
    # Check for environment variables
    local env_count
    env_count=$(aws lambda get-function \
        --function-name "${LAMBDA_FUNCTION_NAME}" \
        --region "${AWS_REGION}" \
        --query 'Configuration.Environment.Variables | length(@)' \
        --output text 2>/dev/null || echo "0")
    
    if [ "$env_count" -gt 0 ]; then
        echo "   Environment variables: $env_count configured"
        
        # Check for database-related environment variables
        local db_vars
        db_vars=$(aws lambda get-function \
            --function-name "${LAMBDA_FUNCTION_NAME}" \
            --region "${AWS_REGION}" \
            --query 'Configuration.Environment.Variables | keys(@) | join(`", "`, @)' \
            --output text 2>/dev/null | grep -iE "(db|database|rds|host|endpoint)" || echo "")
        
        if [ -n "$db_vars" ]; then
            echo "   Database-related env vars detected"
        fi
    fi
}

# Replace placeholders in JSON payload
replace_placeholders() {
    local payload="$1"
    local timestamp=$(date +%Y%m%d_%H%M%S)
    local today=$(date +%Y-%m-%d)
    local uuid=$(openssl rand -hex 8)
    
    echo "$payload" | \
        sed "s/\${timestamp}/${timestamp}/g" | \
        sed "s/\${build_id}/${BUILD_ID}/g" | \
        sed "s/\${test_session}/${TEST_SESSION_ID}/g" | \
        sed "s/\${today}/${today}/g" | \
        sed "s/\${uuid}/${uuid}/g"
}

# Invoke Lambda function with enhanced error handling
invoke_lambda() {
    local payload="$1"
    local test_name="$2"
    
    echo "    🚀 Invoking Lambda: ${LAMBDA_FUNCTION_NAME}"
    
    # Extract operation for logging
    local operation
    operation=$(echo "$payload" | jq -r '.operation // "unknown"')
    echo "       Operation: $operation"
    
    # Create temporary files for response
    local response_file="${UTILS_DIR}/lambda_response_${test_name}.json"
    local payload_file="${UTILS_DIR}/lambda_payload_${test_name}.json"
    local timing_file="${UTILS_DIR}/lambda_timing_${test_name}.txt"
    
    # Save payload for debugging
    echo "$payload" > "$payload_file"
    
    # Invoke Lambda with timing
    local start_time=$(date +%s%3N)
    
    local invoke_result
    if aws lambda invoke \
        --function-name "${LAMBDA_FUNCTION_NAME}" \
        --region "${AWS_REGION}" \
        --payload "$payload" \
        --cli-binary-format raw-in-base64-out \
        "$response_file" > "${UTILS_DIR}/invoke_metadata_${test_name}.json" 2>&1; then
        
        local end_time=$(date +%s%3N)
        local duration=$((end_time - start_time))
        echo "$duration" > "$timing_file"
        
        # Return success
        echo "SUCCESS:$response_file:$duration:$operation"
    else
        local end_time=$(date +%s%3N)
        local duration=$((end_time - start_time))
        echo "$duration" > "$timing_file"
        
        # Return failure
        echo "FAILURE:$response_file:$duration:$operation:AWS_CLI_ERROR"
    fi
}

# Analyze RDS response for database connectivity indicators
analyze_rds_response() {
    local response_file="$1"
    local operation="$2"
    
    local analysis='{"db_connected":false,"data_returned":false,"record_count":0,"has_error":false,"response_type":"unknown"}'
    
    if [ ! -f "$response_file" ]; then
        echo "$analysis"
        return
    fi
    
    local response_content
    response_content=$(cat "$response_file" 2>/dev/null || echo '{}')
    
    local body_content
    body_content=$(echo "$response_content" | jq -r '.body // ""' 2>/dev/null || echo "")
    
    # Check for successful database connection indicators
    local db_connected=false
    local data_returned=false
    local has_error=false
    local response_type="unknown"
    local record_count=0
    
    # Success indicators
    if echo "$body_content" | grep -iq -E "(success|task_data|submitted successfully)"; then
        db_connected=true
        response_type="success"
    fi
    
    # Error indicators
    if echo "$body_content" | grep -iq -E "(error|failed|exception)"; then
        has_error=true
        response_type="error"
    fi
    
    # Data indicators
    if echo "$body_content" | jq -e 'type == "array"' >/dev/null 2>&1; then
        data_returned=true
        record_count=$(echo "$body_content" | jq 'length' 2>/dev/null || echo 0)
    elif echo "$body_content" | jq -e 'has("task_data") or has("results")' >/dev/null 2>&1; then
        data_returned=true
        record_count=1
    fi
    
    # Operation-specific analysis
    case "$operation" in
        "RetrieveTaskById"|"RetrieveJobList"|"RetrieveView"|"RetrieveAuditTrail")
            if [ "$has_error" = false ]; then
                db_connected=true
            fi
            ;;
        "Submit"|"Update")
            if echo "$body_content" | grep -iq "success"; then
                db_connected=true
            fi
            ;;
    esac
    
    # Build analysis JSON
    analysis=$(jq -n \
        --argjson db_connected "$db_connected" \
        --argjson data_returned "$data_returned" \
        --argjson record_count "$record_count" \
        --argjson has_error "$has_error" \
        --arg response_type "$response_type" \
        '{
            db_connected: $db_connected,
            data_returned: $data_returned,
            record_count: $record_count,
            has_error: $has_error,
            response_type: $response_type
        }')
    
    echo "$analysis"
}

# Validate Lambda response against expected criteria
validate_response() {
    local response_file="$1"
    local expected_json="$2"
    local analysis_json="$3"
    
    if [ ! -f "$response_file" ]; then
        echo "FAILED:Response file not found"
        return
    fi
    
    local response_content
    response_content=$(cat "$response_file" 2>/dev/null || echo '{}')
    
    # Validate status code
    local expected_status
    expected_status=$(echo "$expected_json" | jq -r '.statusCode // null')
    
    if [ "$expected_status" != "null" ]; then
        local actual_status
        actual_status=$(echo "$response_content" | jq -r '.statusCode // 200')
        
        if [ "$actual_status" != "$expected_status" ]; then
            echo "FAILED:Status code mismatch: expected $expected_status, got $actual_status"
            return
        fi
    fi
    
    # Validate body content
    local expected_body_contains
    expected_body_contains=$(echo "$expected_json" | jq -r '.bodyContains[]? // empty' 2>/dev/null)
    
    if [ -n "$expected_body_contains" ]; then
        local response_body
        response_body=$(echo "$response_content" | jq -r '.body // ""')
        
        while IFS= read -r required_text; do
            if [ -n "$required_text" ] && ! echo "$response_body" | grep -q "$required_text"; then
                echo "FAILED:Response missing required text: '$required_text'"
                return
            fi
        done <<< "$expected_body_contains"
    fi
    
    # RDS-specific validations
    local requires_db
    requires_db=$(echo "$expected_json" | jq -r '.requiresDbConnection // false')
    
    if [ "$requires_db" = "true" ]; then
        local db_connected
        db_connected=$(echo "$analysis_json" | jq -r '.db_connected // false')
        
        if [ "$db_connected" != "true" ]; then
            echo "FAILED:Database connection required but not detected"
            return
        fi
    fi
    
    echo "PASSED:Validation successful"
}

# Save detailed test response
save_detailed_response() {
    local response_file="$1"
    local test_name="$2"
    local duration="$3"
    local operation="$4"
    local analysis="$5"
    local result_status="$6"
    
    local detailed_file="${REPORTS_DIR}/unit/${test_name}-${BUILD_ID}.json"
    
    local response_content="{}"
    if [ -f "$response_file" ]; then
        response_content=$(cat "$response_file")
    fi
    
    jq -n \
        --arg timestamp "$(date -Iseconds)" \
        --arg build_id "$BUILD_ID" \
        --arg test_session "$TEST_SESSION_ID" \
        --argjson lambda_response "$response_content" \
        --argjson rds_analysis "$analysis" \
        --arg operation "$operation" \
        --argjson duration "$duration" \
        --arg result_status "$result_status" \
        '{
            timestamp: $timestamp,
            build_id: $build_id,
            test_session: $test_session,
            lambda_response: $lambda_response,
            rds_analysis: $rds_analysis,
            test_metadata: {
                operation: $operation,
                duration_ms: $duration,
                result_status: $result_status,
                db_connected: $rds_analysis.db_connected
            }
        }' > "$detailed_file"
}

# Generate JUnit XML report
generate_junit_report() {
    local results_file="$1"
    local test_type="$2"
    
    local junit_file="${REPORTS_DIR}/${test_type}/junit-results.xml"
    
    # Read results
    local total_tests passed_tests failed_tests total_time
    total_tests=$(jq 'length' "$results_file")
    passed_tests=$(jq '[.[] | select(.passed == true)] | length' "$results_file")
    failed_tests=$((total_tests - passed_tests))
    total_time=$(jq '[.[].duration] | add / 1000' "$results_file")
    
    # Start XML generation
    cat > "$junit_file" << EOF
<?xml version="1.0" encoding="UTF-8"?>
<testsuite name="${LAMBDA_FUNCTION_NAME} RDS ${test_type^} Tests" 
           tests="${total_tests}" 
           failures="${failed_tests}" 
           time="${total_time}" 
           timestamp="$(date -Iseconds)">
  <properties>
    <property name="lambda.function" value="${LAMBDA_FUNCTION_NAME}"/>
    <property name="aws.region" value="${AWS_REGION}"/>
    <property name="build.id" value="${BUILD_ID}"/>
    <property name="test.session" value="${TEST_SESSION_ID}"/>
  </properties>
EOF
    
    # Add test cases
    jq -r '.[] | 
        "  <testcase name=\"" + .name + "\" time=\"" + (.duration/1000|tostring) + "\" classname=\"RDS." + .operation + "\">" +
        (if .passed then "" else 
            "\n    <failure message=\"" + (.error // "Test failed") + "\">" +
            "Operation: " + .operation + "\nError: " + (.error // "Unknown error") +
            "</failure>" 
        end) +
        "\n  </testcase>"' "$results_file" >> "$junit_file"
    
    # Close XML
    echo "</testsuite>" >> "$junit_file"
}

# Generate summary JSON report
generate_summary_report() {
    local results_file="$1"
    local test_type="$2"
    
    local summary_file="${REPORTS_DIR}/${test_type}/summary.json"
    
    # Calculate statistics
    local total passed failed avg_duration db_connected
    total=$(jq 'length' "$results_file")
    passed=$(jq '[.[] | select(.passed == true)] | length' "$results_file")
    failed=$((total - passed))
    avg_duration=$(jq '[.[].duration] | if length > 0 then add / length else 0 end' "$results_file")
    db_connected=$(jq '[.[] | select(.db_connected == true)] | length' "$results_file")
    
    # Generate operations summary
    local operations_summary
    operations_summary=$(jq 'group_by(.operation) | map({
        operation: .[0].operation,
        total: length,
        passed: [.[] | select(.passed == true)] | length,
        db_connected: [.[] | select(.db_connected == true)] | length
    })' "$results_file")
    
    jq -n \
        --arg timestamp "$(date -Iseconds)" \
        --arg build_id "$BUILD_ID" \
        --arg test_session "$TEST_SESSION_ID" \
        --arg lambda_function "$LAMBDA_FUNCTION_NAME" \
        --arg test_type "$test_type" \
        --argjson total "$total" \
        --argjson passed "$passed" \
        --argjson failed "$failed" \
        --argjson avg_duration "$avg_duration" \
        --argjson db_connected "$db_connected" \
        --argjson operations_summary "$operations_summary" \
        --slurpfile results "$results_file" \
        '{
            timestamp: $timestamp,
            build_id: $build_id,
            test_session: $test_session,
            lambda_function: $lambda_function,
            test_type: $test_type,
            total: $total,
            passed: $passed,
            failed: $failed,
            average_duration_ms: $avg_duration,
            db_operations_summary: $operations_summary,
            db_connectivity: {
                successful_connections: $db_connected,
                total_db_operations: $total,
                connection_rate: (if $total > 0 then ($db_connected / $total) else 0 end)
            },
            results: $results[0]
        }' > "$summary_file"
}

# Run unit tests
run_unit_tests() {
    echo -e "${BLUE}🧪 Running RDS Lambda Unit Tests${NC}"
    echo "=" * 50
    
    local config_file="${CONFIG_DIR}/unit-tests.json"
    
    if [ ! -f "$config_file" ]; then
        echo -e "${YELLOW}⚠️ Unit test configuration not found: $config_file${NC}"
        echo "Creating default configuration..."
        create_default_unit_config
    fi
    
    local results_file="${REPORTS_DIR}/unit/results.json"
    echo "[]" > "$results_file"
    
    local scenario_count
    scenario_count=$(jq '.scenarios | length' "$config_file")
    
    echo "Testing Lambda: ${LAMBDA_FUNCTION_NAME}"
    echo "Region: ${AWS_REGION}"
    echo "Test Session: ${TEST_SESSION_ID}"
    echo "Scenarios: ${scenario_count}"
    echo
    
    local total_passed=0
    local total_failed=0
    local total_duration=0
    local db_operations_successful=0
    
    # Process each scenario
    jq -c '.scenarios[]' "$config_file" | while IFS= read -r scenario; do
        local scenario_name
        scenario_name=$(echo "$scenario" | jq -r '.name')
        local test_event
        test_event=$(echo "$scenario" | jq -c '.event')
        local expected
        expected=$(echo "$scenario" | jq -c '.expected')
        
        echo -e "  📝 Scenario: ${scenario_name}"
        
        # Replace placeholders
        local processed_payload
        processed_payload=$(replace_placeholders "$test_event")
        
        # Create safe test name for files
        local safe_test_name
        safe_test_name=$(echo "$scenario_name" | tr ' ' '-' | tr -d '[:punct:]' | tr '[:upper:]' '[:lower:]')
        
        # Invoke Lambda
        local invoke_result
        invoke_result=$(invoke_lambda "$processed_payload" "$safe_test_name")
        
        # Parse invoke result
        IFS=':' read -r status response_file duration operation error <<< "$invoke_result"
        
        local test_result='{"name":"'$scenario_name'","passed":false,"duration":0,"error":"Unknown","operation":"unknown","db_connected":false,"data_returned":false}'
        
        if [ "$status" = "SUCCESS" ]; then
            # Analyze RDS response
            local analysis
            analysis=$(analyze_rds_response "$response_file" "$operation")
            
            # Validate response
            local validation_result
            validation_result=$(validate_response "$response_file" "$expected" "$analysis")
            
            IFS=':' read -r validation_status validation_message <<< "$validation_result"
            
            if [ "$validation_status" = "PASSED" ]; then
                local db_status=""
                if [ "$(echo "$analysis" | jq -r '.db_connected')" = "true" ]; then
                    db_status=" 📊 DB Connected"
                    db_operations_successful=$((db_operations_successful + 1))
                else
                    db_status=" ⚠️ DB Issue"
                fi
                
                echo -e "    ${GREEN}✅ PASSED${NC}: ${scenario_name} (${duration}ms)${db_status}"
                
                test_result=$(jq -n \
                    --arg name "$scenario_name" \
                    --argjson passed true \
                    --argjson duration "$duration" \
                    --arg error "null" \
                    --arg operation "$operation" \
                    --argjson db_connected "$(echo "$analysis" | jq '.db_connected')" \
                    --argjson data_returned "$(echo "$analysis" | jq '.data_returned')" \
                    '{name: $name, passed: $passed, duration: $duration, error: $error, operation: $operation, db_connected: $db_connected, data_returned: $data_returned}')
                
                total_passed=$((total_passed + 1))
            else
                echo -e "    ${RED}❌ FAILED${NC}: ${scenario_name} - ${validation_message}"
                
                test_result=$(jq -n \
                    --arg name "$scenario_name" \
                    --argjson passed false \
                    --argjson duration "$duration" \
                    --arg error "$validation_message" \
                    --arg operation "$operation" \
                    --argjson db_connected false \
                    --argjson data_returned false \
                    '{name: $name, passed: $passed, duration: $duration, error: $error, operation: $operation, db_connected: $db_connected, data_returned: $data_returned}')
                
                total_failed=$((total_failed + 1))
            fi
            
            # Save detailed response
            save_detailed_response "$response_file" "$safe_test_name" "$duration" "$operation" "$analysis" "$validation_status"
        else
            local error_msg="$error"
            echo -e "    ${RED}❌ FAILED${NC}: ${scenario_name} - ${error_msg}"
            
            test_result=$(jq -n \
                --arg name "$scenario_name" \
                --argjson passed false \
                --argjson duration "$duration" \
                --arg error "$error_msg" \
                --arg operation "$operation" \
                --argjson db_connected false \
                --argjson data_returned false \
                '{name: $name, passed: $passed, duration: $duration, error: $error, operation: $operation, db_connected: $db_connected, data_returned: $data_returned}')
            
            total_failed=$((total_failed + 1))
        fi
        
        total_duration=$((total_duration + duration))
        
        # Append result to results file
        local temp_results
        temp_results=$(mktemp)
        jq --argjson new_result "$test_result" '. += [$new_result]' "$results_file" > "$temp_results"
        mv "$temp_results" "$results_file"
    done
    
    # Generate reports
    generate_junit_report "$results_file" "unit"
    generate_summary_report "$results_file" "unit"
    
    # Print summary
    local total_tests=$((total_passed + total_failed))
    local avg_duration=0
    if [ $total_tests -gt 0 ]; then
        avg_duration=$((total_duration / total_tests))
    fi
    
    echo
    echo -e "${BLUE}📊 RDS Lambda Unit Test Summary:${NC}"
    echo "=" * 40
    echo "Total: $total_tests"
    echo "Passed: $total_passed"
    echo "Failed: $total_failed"
    echo "Average Duration: ${avg_duration}ms"
    echo "DB Operations Successful: ${db_operations_successful}/${total_tests}"
    echo "Lambda Function: ${LAMBDA_FUNCTION_NAME}"
    
    # Return success if all tests passed
    [ $total_failed -eq 0 ]
}

# Create default unit test configuration
create_default_unit_config() {
    local config_file="${CONFIG_DIR}/unit-tests.json"
    
    cat > "$config_file" << 'EOF'
{
  "scenarios": [
    {
      "name": "Basic-Health-Check",
      "event": {
        "test": "health",
        "source": "jenkins"
      },
      "expected": {
        "statusCode": 200
      }
    },
    {
      "name": "RetrieveJobList-Test",
      "event": {
        "operation": "RetrieveJobList",
        "queryStringParameters": {
          "table_name": "test_jobs"
        }
      },
      "expected": {
        "statusCode": 200
      }
    }
  ]
}
EOF
    
    echo -e "${GREEN}📝 Created default unit test configuration: $config_file${NC}"
}

# Run integration tests (simplified version)
run_integration_tests() {
    echo -e "${BLUE}🔗 Running RDS Lambda Integration Tests${NC}"
    echo "=" * 50
    
    local config_file="${CONFIG_DIR}/integration-tests.json"
    
    if [ ! -f "$config_file" ]; then
        echo -e "${YELLOW}⚠️ Integration test configuration not found: $config_file${NC}"
        echo "Creating default configuration..."
        create_default_integration_config
    fi
    
    local workflow_count
    workflow_count=$(jq '.workflows | length' "$config_file")
    
    echo "Testing Lambda: ${LAMBDA_FUNCTION_NAME}"
    echo "Workflows: ${workflow_count}"
    echo
    
    local workflow_success=true
    
    # Process each workflow
    jq -c '.workflows[]' "$config_file" | while IFS= read -r workflow; do
        local workflow_name
        workflow_name=$(echo "$workflow" | jq -r '.name')
        local workflow_description
        workflow_description=$(echo "$workflow" | jq -r '.description // ""')
        
        echo -e "  🔗 Workflow: ${workflow_name}"
        if [ -n "$workflow_description" ]; then
            echo "      Description: $workflow_description"
        fi
        
        local step_success=true
        
        # Process each step
        echo "$workflow" | jq -c '.steps[]' | while IFS= read -r step; do
            local step_name
            step_name=$(echo "$step" | jq -r '.name')
            local step_type
            step_type=$(echo "$step" | jq -r '.type')
            
            echo "    📋 Step: $step_name"
            
            case "$step_type" in
                "invoke_lambda")
                    local payload
                    payload=$(echo "$step" | jq -c '.payload')
                    local expected
                    expected=$(echo "$step" | jq -c '.expect')
                    
                    # Process payload
                    local processed_payload
                    processed_payload=$(replace_placeholders "$payload")
                    
                    # Create safe step name
                    local safe_step_name
                    safe_step_name=$(echo "$step_name" | tr ' ' '-' | tr -d '[:punct:]' | tr '[:upper:]' '[:lower:]')
                    
                    # Invoke Lambda
                    local invoke_result
                    invoke_result=$(invoke_lambda "$processed_payload" "integration_${safe_step_name}")
                    
                    # Parse result
                    IFS=':' read -r status response_file duration operation error <<< "$invoke_result"
                    
                    if [ "$status" = "SUCCESS" ]; then
                        local analysis
                        analysis=$(analyze_rds_response "$response_file" "$operation")
                        
                        local validation_result
                        validation_result=$(validate_response "$response_file" "$expected" "$analysis")
                        
                        IFS=':' read -r validation_status validation_message <<< "$validation_result"
                        
                        if [ "$validation_status" = "PASSED" ]; then
                            echo -e "      ${GREEN}✅ Passed${NC} (${duration}ms)"
                        else
                            echo -e "      ${RED}❌ Failed${NC}: $validation_message"
                            step_success=false
                        fi
                    else
                        echo -e "      ${RED}❌ Failed${NC}: $error"
                        step_success=false
                    fi
                    ;;
                    
                "wait")
                    local seconds
                    seconds=$(echo "$step" | jq -r '.seconds')
                    echo "      ⏳ Waiting ${seconds} seconds..."
                    sleep "$seconds"
                    echo -e "      ${GREEN}✅ Wait completed${NC}"
                    ;;
                    
                *)
                    echo -e "      ${YELLOW}⚠️ Unknown step type: $step_type${NC}"
                    ;;
            esac
        done
        
        if [ "$step_success" = true ]; then
            echo -e "  ${GREEN}✅ Workflow completed successfully${NC}"
        else
            echo -e "  ${RED}❌ Workflow failed${NC}"
            workflow_success=false
        fi
        echo
    done
    
    if [ "$workflow_success" = true ]; then
        echo -e "${GREEN}🎉 All integration tests passed!${NC}"
        return 0
    else
        echo -e "${RED}❌ Some integration tests failed!${NC}"
        return 1
    fi
}

# Create default integration test configuration
create_default_integration_config() {
    local config_file="${CONFIG_DIR}/integration-tests.json"
    
    cat > "$config_file" << 'EOF'
{
  "workflows": [
    {
      "name": "Basic-RDS-Test",
      "description": "Test basic database connectivity",
      "steps": [
        {
          "name": "Test-Database-Connection",
          "type": "invoke_lambda",
          "payload": {
            "operation": "RetrieveJobList",
            "queryStringParameters": {
              "table_name": "test_jobs"
            }
          },
          "expect": {
            "statusCode": 200
          }
        }
      ]
    }
  ]
}
EOF
    
    echo -e "${GREEN}📝 Created default integration test configuration: $config_file${NC}"
}

# Cleanup temporary files
cleanup() {
    echo "🧹 Cleaning up temporary files..."
    rm -f "${UTILS_DIR}"/lambda_*
    rm -f "${UTILS_DIR}"/invoke_*
}

# Main function
main() {
    if [ $# -lt 1 ]; then
        echo "Usage: $0 [unit|integration|all]"
        echo
        echo "Commands:"
        echo "  unit        - Run unit tests for RDS operations"
        echo "  integration - Run integration workflow tests"
        echo "  all         - Run all RDS Lambda tests"
        exit 1
    fi
    
    local test_type="$1"
    
    echo -e "${PURPLE}🚀 Initializing RDS Lambda Tester (AWS CLI)${NC}"
    echo "Region: ${AWS_REGION}"
    echo "Test Type: ${test_type}"
    echo "=" * 50
    
    # Setup
    check_dependencies
    setup_directories
    find_lambda_function
    verify_lambda_access
    
    local success=true
    
    case "$test_type" in
        "unit")
            if ! run_unit_tests; then
                success=false
            fi
            ;;
            
        "integration")
            if ! run_integration_tests; then
                success=false
            fi
            ;;
            
        "all")
            echo -e "${BLUE}🧪 Running All RDS Lambda Tests${NC}"
            echo "=" * 50
            
            if ! run_unit_tests; then
                success=false
            fi
            
            echo
            echo "=" * 50
            echo
            
            if ! run_integration_tests; then
                success=false
            fi
            
            echo
            echo "=" * 50
            echo -e "${BLUE}📊 Overall RDS Lambda Test Summary:${NC}"
            echo "Unit Tests: $([ -f "${REPORTS_DIR}/unit/summary.json" ] && echo "✅ Completed" || echo "❌ Failed")"
            echo "Integration Tests: $([ -f "${REPORTS_DIR}/integration/summary.json" ] && echo "✅ Completed" || echo "❌ Failed")"
            echo "Overall: $([ "$success" = true ] && echo "✅ ALL TESTS PASSED" || echo "❌ SOME TESTS FAILED")"
            ;;
            
        *)
            echo -e "${RED}❌ Unknown test type: $test_type${NC}"
            echo "Valid options: unit, integration, all"
            exit 1
            ;;
    esac
    
    # Cleanup
    cleanup
    
    # Exit with appropriate code
    if [ "$success" = true ]; then
        echo -e "\n${GREEN}🎉 All $test_type tests passed! RDS Lambda is working correctly.${NC}"
        exit 0
    else
        echo -e "\n${RED}❌ Some $test_type tests failed! Check RDS connectivity and Lambda logic.${NC}"
        exit 1
    fi
}

# Run main function with all arguments
main "$@"
```

---

## **🔧 Helper Scripts**

### **`tests/utils/test_helpers.sh` - Common Functions**
```bash
#!/bin/bash

# Common helper functions for AWS CLI testing

# Check if jq is available and install if needed
ensure_jq() {
    if ! command -v jq &> /dev/null; then
        echo "Installing jq..."
        if command -v yum &> /dev/null; then
            sudo yum install -y jq
        elif command -v apt-get &> /dev/null; then
            sudo apt-get update && sudo apt-get install -y jq
        else
            echo "Please install jq manually"
            exit 1
        fi
    fi
}

# Validate JSON
validate_json() {
    local json_file="$1"
    if ! jq empty "$json_file" 2>/dev/null; then
        echo "Invalid JSON in $json_file"
        return 1
    fi
    return 0
}

# Pretty print test result
print_test_result() {
    local status="$1"
    local test_name="$2"
    local duration="$3"
    local message="$4"
    
    if [ "$status" = "PASSED" ]; then
        echo -e "    ${GREEN}✅ PASSED${NC}: $test_name (${duration}ms) $message"
    else
        echo -e "    ${RED}❌ FAILED${NC}: $test_name - $message"
    fi
}
```

### **`tests/setup_prerequisites.sh` - Prerequisites Checker**
```bash
#!/bin/bash

echo "🔧 Checking Prerequisites for RDS Lambda Testing (AWS CLI Version)"
echo "=================================================================="

# Check AWS CLI
echo "1. Checking AWS CLI..."
if aws --version >/dev/null 2>&1; then
    echo "   ✅ AWS CLI available: $(aws --version)"
else
    echo "   ❌ AWS CLI not found"
    exit 1
fi

# Check jq
echo "2. Checking jq..."
if jq --version >/dev/null 2>&1; then
    echo "   ✅ jq available: $(jq --version)"
else
    echo "   ❌ jq not found. Installing..."
    if command -v yum &> /dev/null; then
        sudo yum install -y jq
    elif command -v apt-get &> /dev/null; then
        sudo apt-get update && sudo apt-get install -y jq
    else
        echo "   ❌ Please install jq manually"
        exit 1
    fi
fi

# Check openssl
echo "3. Checking openssl..."
if openssl version >/dev/null 2>&1; then
    echo "   ✅ openssl available: $(openssl version)"
else
    echo "   ❌ openssl not found"
    exit 1
fi

# Check AWS credentials
echo "4. Checking AWS credentials..."
if aws sts get-caller-identity >/dev/null 2>&1; then
    echo "   ✅ AWS credentials configured"
    echo "   Identity: $(aws sts get-caller-identity --query 'Arn' --output text)"
else
    echo "   ❌ AWS credentials not configured or invalid"
    exit 1
fi

# Check directory structure
echo "5. Checking directory structure..."
for dir in "tests" "tests/config" "tests/reports" "tests/reports/unit" "tests/reports/integration" "tests/utils"; do
    if [ -d "$dir" ]; then
        echo "   ✅ Directory exists: $dir"
    else
        echo "   📁 Creating directory: $dir"
        mkdir -p "$dir"
    fi
done

# Check Lambda function access
echo "6. Checking Lambda function access..."
LAMBDA_FUNCTION=$(aws lambda list-functions --query 'Functions[?contains(FunctionName, `euc-lambda-poc`)].FunctionName' --output text --region ap-northeast-1 | head -n1)

if [ ! -z "$LAMBDA_FUNCTION" ]; then
    echo "   ✅ Lambda function found: $LAMBDA_FUNCTION"
    
    if aws lambda get-function --function-name "$LAMBDA_FUNCTION" --region ap-northeast-1 >/dev/null 2>&1; then
        echo "   ✅ Lambda function accessible"
    else
        echo "   ❌ Cannot access Lambda function"
        exit 1
    fi
else
    echo "   ❌ No Lambda function found matching 'euc-lambda-poc'"
    exit 1
fi

echo ""
echo "🎉 All prerequisites satisfied!"
echo "You're ready to run the RDS Lambda testing framework (AWS CLI version)."
echo ""
echo "Quick test commands:"
echo "  chmod +x tests/run_tests.sh"
echo "  ./tests/run_tests.sh unit"
echo "  ./tests/run_tests.sh integration"
echo "  ./tests/run_tests.sh all"
```

---

## **📊 Key Differences: AWS CLI vs boto3**

### **✅ AWS CLI Advantages:**
- **No dependencies** - Uses existing AWS CLI installation
- **Simple setup** - Just bash scripts, no Python packages
- **Direct AWS integration** - Same tool used for deployment
- **Familiar syntax** - Most teams already know AWS CLI

### **❌ AWS CLI Limitations:**
- **Complex JSON handling** - Requires `jq` for JSON processing
- **Limited error handling** - Basic error detection compared to Python
- **Harder to extend** - Adding features requires more bash complexity
- **Debugging challenges** - Less detailed error information

### **🔧 File Structure (AWS CLI Version):**
```
tests/
├── run_tests.sh                 # Main test runner (executable)
├── setup_prerequisites.sh       # Prerequisites checker
├── config/
│   ├── unit-tests.json         # Test scenarios
│   └── integration-tests.json  # Workflow definitions
├── utils/
│   └── test_helpers.sh         # Helper functions
└── reports/                    # Generated test reports
    ├── unit/
    └── integration/
```

### **📋 Usage Commands:**
```bash
# Make scripts executable
chmod +x tests/run_tests.sh tests/setup_prerequisites.sh

# Check prerequisites
./tests/setup_prerequisites.sh

# Run tests
./tests/run_tests.sh unit
./tests/run_tests.sh integration
./tests/run_tests.sh all
```

### **🔗 Jenkins Integration:**
```groovy
stage('Unit Tests') {
    steps {
        withAWS(region: "${AWS_DEFAULT_REGION}", role: 'arn:aws:iam::623341402407:role/jef-tfe-role') {
            sh 'chmod +x tests/run_tests.sh'
            sh './tests/run_tests.sh unit'
        }
    }
    post {
        always {
            publishTestResults testResultsPattern: 'tests/reports/**/*.xml'
        }
    }
}
```

**The AWS CLI version provides the same comprehensive RDS Lambda testing capabilities with simpler setup but less sophisticated error handling and extensibility compared to the boto3 approach.** 🎯


